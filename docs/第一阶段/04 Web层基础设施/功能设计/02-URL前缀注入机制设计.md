# URL 前缀注入机制设计

## 从"复制粘贴"到"自动推断"

在[第 1 章](01-问题域与设计目标.md)中我们提到了 URL 前缀的问题：电商后台有管理端和用户端两套 API，前缀分别是 `/admin-api` 和 `/app-api`。如果在每个 Controller 上硬编码前缀，改前缀时需要批量修改所有文件，而且容易遗漏。

本章讲解框架如何通过"包路径约定 + WebMvcRegistrations 扩展"实现前缀的自动注入，让 Controller 完全不感知前缀的存在。

## 核心机制：WebMvcRegistrations + setPathPrefixes

Spring Boot 提供了 `WebMvcRegistrations` 接口，允许开发者自定义 Spring MVC 的核心组件。其中 `getRequestMappingHandlerMapping()` 方法可以替换默认的 `RequestMappingHandlerMapping`。

`RequestMappingHandlerMapping` 有一个关键方法 `setPathPrefixes(Map<String, Predicate<Class<?>>> pathPrefixes)`，它的作用是：**为满足条件的 Controller 类自动添加 URL 前缀**。

工作流程如下：

```
Spring 容器启动
    │
    ▼
YudaoWebAutoConfiguration.webMvcRegistrations()
    │
    ▼
创建自定义 RequestMappingHandlerMapping
    │
    ▼
调用 setPathPrefixes(buildPathPrefixes(webProperties))
    │
    ▼
构建 prefix → Predicate 映射：
  "/admin-api" → clazz 匹配 **.controller.admin.**
  "/app-api"   → clazz 匹配 **.controller.app.**
    │
    ▼
Spring MVC 注册 Controller 时，自动检测包路径并注入前缀
```

### 源码解读

核心代码在 `YudaoWebAutoConfiguration` 中：

```java
// 文件：cn.iocoder.yudao.framework.web.config.YudaoWebAutoConfiguration

@Bean
public WebMvcRegistrations webMvcRegistrations(WebProperties webProperties) {
    return new WebMvcRegistrations() {

        @Override
        public RequestMappingHandlerMapping getRequestMappingHandlerMapping() {
            RequestMappingHandlerMapping mapping = new RequestMappingHandlerMapping();
            mapping.setPathPrefixes(buildPathPrefixes(webProperties));
            return mapping;
        }

        private Map<String, Predicate<Class<?>>> buildPathPrefixes(WebProperties webProperties) {
            AntPathMatcher antPathMatcher = new AntPathMatcher(".");
            Map<String, Predicate<Class<?>>> pathPrefixes = Maps.newLinkedHashMapWithExpectedSize(2);
            putPathPrefix(pathPrefixes, webProperties.getAdminApi(), antPathMatcher);
            putPathPrefix(pathPrefixes, webProperties.getAppApi(), antPathMatcher);
            return pathPrefixes;
        }

        private void putPathPrefix(Map<String, Predicate<Class<?>>> pathPrefixes,
                                    WebProperties.Api api, AntPathMatcher matcher) {
            if (api == null || StrUtil.isEmpty(api.getPrefix())) {
                return;
            }
            pathPrefixes.put(api.getPrefix(),
                    clazz -> clazz.isAnnotationPresent(RestController.class)
                            && matcher.match(api.getController(), clazz.getPackage().getName()));
        }
    };
}
```

这段代码做了三件事：

1. **创建自定义的 `RequestMappingHandlerMapping`**，替代 Spring MVC 默认的
2. **构建前缀映射表**，从 `WebProperties` 读取配置
3. **定义匹配条件**，用 Ant 风格路径匹配器检查 Controller 的包名

## 包路径匹配的细节

### 为什么用 "." 作为分隔符？

注意 `AntPathMatcher` 的构造参数：

```java
AntPathMatcher antPathMatcher = new AntPathMatcher(".");
```

默认的 `AntPathMatcher` 用 `/` 作为路径分隔符，匹配的是 URL 路径（如 `/api/users`）。但这里要匹配的是 **Java 包名**，包名用 `.` 分隔（如 `cn.iocoder.yudao.module.product.controller.admin`）。

所以传入 `"."` 作为分隔符，让 `**.controller.admin.**` 能正确匹配包名。

### 匹配规则详解

默认配置中，两个 API 的匹配规则是：

| 前缀 | 包路径模式 | 匹配示例 |
|------|-----------|---------|
| `/admin-api` | `**.controller.admin.**` | `cn.iocoder.yudao.module.product.controller.admin.spu.AdminProductSpuController` |
| `/app-api` | `**.controller.app.**` | `cn.iocoder.yudao.module.product.controller.app.spu.AppProductSpuController` |

`**` 是 Ant 风格的通配符，匹配任意层级的包名。所以无论 Controller 在哪个模块（product、order、trade），只要它在 `controller.admin` 或 `controller.app` 包下，就会自动获得对应的前缀。

### 匹配条件的双重检查

```java
clazz.isAnnotationPresent(RestController.class)
    && matcher.match(api.getController(), clazz.getPackage().getName())
```

除了包路径匹配，还检查了 `@RestController` 注解。这确保了：
- 只有 RESTful Controller 会被注入前缀
- 普通的 `@Controller`（如页面控制器）不会被影响
- `@Service`、`@Component` 等其他 Bean 不会被误匹配

## 配置体系

### WebProperties 配置类

```java
// 文件：cn.iocoder.yudao.framework.web.config.WebProperties

@ConfigurationProperties(prefix = "yudao.web")
@Validated
@Data
public class WebProperties {

    @NotNull(message = "APP API 不能为空")
    private Api appApi = new Api("/app-api", "**.controller.app.**");

    @NotNull(message = "Admin API 不能为空")
    private Api adminApi = new Api("/admin-api", "**.controller.admin.**");

    @NotNull(message = "Admin UI 不能为空")
    private Ui adminUi;

    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    @Valid
    public static class Api {
        @NotEmpty(message = "API 前缀不能为空")
        private String prefix;

        @NotEmpty(message = "Controller 所在包不能为空")
        private String controller;
    }

    @Data
    @Valid
    public static class Ui {
        private String url;
    }
}
```

### 配置示例

在 `application.yml` 中：

```yaml
yudao:
  web:
    admin-api:
      prefix: /admin-api          # 管理端 API 前缀
      controller: "**.controller.admin.**"  # 管理端 Controller 包路径
    app-api:
      prefix: /app-api            # 用户端 API 前缀
      controller: "**.controller.app.**"    # 用户端 Controller 包路径
    admin-ui:
      url: http://localhost:80    # 管理后台前端地址（用于文档等场景）
```

如果需要改前缀（比如从 `/admin-api` 改成 `/manage-api`），只需修改配置文件，所有 Controller 自动生效。

## 前缀注入的实际效果

### 改造前（硬编码前缀）

```java
@RestController
@RequestMapping("/admin-api/system/dept")
public class AdminSystemDeptController {

    @GetMapping("/list")  // 实际路径：/admin-api/system/dept/list
    public CommonResult<List<SystemDeptRespVO>> getDeptList(...) { ... }
}
```

### 改造后（自动注入前缀）

```java
@RestController
@RequestMapping("/system/dept")  // 只写业务路径
public class AdminSystemDeptController {

    @GetMapping("/list")  // 实际路径：/admin-api/system/dept/list（框架自动加前缀）
    public CommonResult<List<SystemDeptRespVO>> getDeptList(...) { ... }
}
```

Controller 完全不感知 `/admin-api` 前缀。框架在 Spring MVC 注册 Controller 时，检测到它在 `controller.admin` 包下，自动注入 `/admin-api` 前缀。

## 前缀注入的安全意义

前缀注入不仅仅是代码整洁的问题，它还有重要的安全意义。

### Nginx 反向代理的便利

生产环境中，Nginx 通常只转发特定前缀的请求到后端：

```nginx
location /admin-api/ {
    proxy_pass http://backend:8080;
}

location /app-api/ {
    proxy_pass http://backend:8080;
}
```

这意味着 Swagger 文档（`/v3/api-docs`）、Spring Boot Actuator（`/actuator/health`）等内部端点不会被 Nginx 转发到外部，自然也不会暴露给公网用户。

### URL 前缀与用户类型的映射

在 `WebFrameworkUtils.getLoginUserType()` 中，URL 前缀被用来推断用户类型：

```java
public static Integer getLoginUserType(HttpServletRequest request) {
    // 1. 优先，从 Attribute 中获取
    Integer userType = (Integer) request.getAttribute(REQUEST_ATTRIBUTE_LOGIN_USER_TYPE);
    if (userType != null) {
        return userType;
    }
    // 2. 其次，基于 URL 前缀的约定
    if (request.getServletPath().startsWith(properties.getAdminApi().getPrefix())) {
        return UserTypeEnum.ADMIN.getValue();
    }
    if (request.getServletPath().startsWith(properties.getAppApi().getPrefix())) {
        return UserTypeEnum.MEMBER.getValue();
    }
    return null;
}
```

这是一个"约定优于配置"的典型应用：`/admin-api` 开头的请求默认是管理员操作，`/app-api` 开头的请求默认是会员操作。这个推断在日志记录、权限判断等场景中非常有用。

## 为什么不用其他方案？

### 方案对比

| 方案 | 实现方式 | 优点 | 缺点 |
|------|---------|------|------|
| **硬编码前缀** | 每个 Controller 的 `@RequestMapping` 中写前缀 | 简单直观 | 改前缀需批量修改，容易遗漏 |
| **注解标记** | 自定义注解 `@AdminApi`、`@AppApi`，配合 AOP 注入 | 语义清晰 | 需要自定义注解 + AOP，复杂度高 |
| **配置文件映射** | 在配置文件中定义"包名 → 前缀"的映射 | 灵活 | 配置繁琐，容易出错 |
| **包路径约定**（本项目） | 基于包名自动推断前缀 | 零侵入，改前缀只需改配置 | 依赖包名规范 |

本项目选择"包路径约定"方案，核心原因是：**它是最接近"约定优于配置"理念的方案**。项目中已经通过包结构（`controller.admin`、`controller.app`）区分了管理端和用户端，前缀注入只是把这个已有的约定"显式化"了。

### 为什么不选注解方案？

自定义注解（如 `@AdminApi("/system/dept")`）看起来语义更清晰，但有两个问题：

1. **侵入性**：每个 Controller 都需要加注解，和硬编码前缀一样容易遗漏
2. **冗余信息**：Controller 在 `controller.admin` 包下，已经隐含了"这是管理端接口"的信息，再加注解是重复表达

包路径方案的核心优势是：**信息只存在于一处（包路径），前缀自动推断，没有重复**。

## 与项目分层的关系

URL 前缀注入机制与项目的三层模块设计紧密配合：

```
yudao-module-product/
├── yudao-module-product-api/          # Feign 客户端 + DTO
│   └── src/main/java/.../
│       └── api/                       # 无 @RestController，不触发前缀注入
│
└── yudao-module-product-server/       # 业务实现
    └── src/main/java/.../
        ├── controller/
        │   ├── admin/                 # ← 匹配 **.controller.admin.**
        │   │   └── spu/
        │   │       └── AdminProductSpuController.java  → /admin-api/product/spu
        │   └── app/                   # ← 匹配 **.controller.app.**
        │       └── spu/
        │           └── AppProductSpuController.java   → /app-api/product/spu
        ├── service/
        └── dal/
```

- `controller.admin` 包下的 Controller 自动获得 `/admin-api` 前缀
- `controller.app` 包下的 Controller 自动获得 `/app-api` 前缀
- `service`、`dal` 等包下的类没有 `@RestController`，不受影响

## 前缀注入的时序

```
应用启动
    │
    ▼
Spring Boot 扫描到 @AutoConfiguration
    │
    ▼
YudaoWebAutoConfiguration.webMvcRegistrations() 被调用
    │
    ▼
创建 RequestMappingHandlerMapping，设置 pathPrefixes
    │
    ▼
Spring MVC 扫描所有 @RestController Bean
    │
    ▼
对每个 Controller：
  ├── 获取包名：cn.iocoder.yudao.module.product.controller.admin.spu
  ├── 检查 **.controller.admin.** → 匹配！
  ├── 注入前缀：/admin-api
  └── 注册最终路径：/admin-api/product/spu
    │
    ▼
Controller 可以接收请求了
```

关键点：前缀注入发生在 **Spring MVC 注册 Controller 时**，不是运行时动态计算的。所以对请求处理性能没有影响。

## WebFrameworkUtils 工具类

`WebFrameworkUtils` 是与 URL 前缀紧密配合的工具类，提供了一系列 Web 请求相关的辅助方法：

| 方法 | 作用 | 使用场景 |
|------|------|---------|
| `getTenantId(request)` | 从 `tenant-id` 请求头获取租户编号 | 多租户隔离 |
| `getVisitTenantId(request)` | 从 `visit-tenant-id` 请求头获取访问的租户编号 | 租户管理 |
| `getLoginUserId(request)` | 从请求属性获取当前登录用户 ID | 日志记录、权限判断 |
| `getLoginUserType(request)` | 获取用户类型（基于 URL 前缀推断） | 日志记录、权限判断 |
| `setLoginUserId(request, userId)` | 设置登录用户 ID（由认证 Filter 调用） | 认证流程 |
| `setCommonResult(request, result)` | 记录 Controller 返回结果 | 访问日志 |
| `getCommonResult(request)` | 获取 Controller 返回结果 | 访问日志 |
| `isRpcRequest(request)` | 判断是否为 RPC 请求（URL 以 `/rpc-api/` 开头） | 过滤器跳过判断 |

其中 `getLoginUserType()` 的实现特别值得注意——它体现了 URL 前缀的"一词多义"：前缀不仅标识了 API 版本，还隐含了用户类型信息。

## 本章小结

URL 前缀注入机制的设计可以用一句话概括：**让已有的包结构约定同时承担 URL 路由的职责**。

核心要点：
1. 通过 `WebMvcRegistrations` 替换默认的 `RequestMappingHandlerMapping`
2. 用 `setPathPrefixes()` 注入"前缀 → 匹配条件"映射
3. 用 `AntPathMatcher(".")` 匹配 Java 包名（而非 URL 路径）
4. 匹配条件是"有 `@RestController` 注解 + 包名匹配模式"
5. 前缀可通过 `yudao.web.*` 配置修改，零代码变更

---

**思考题**

1. 如果项目新增一个"开放平台"端（前缀 `/open-api`），需要改动哪些地方？对比硬编码方案，包路径方案的优势在哪里？
2. `AntPathMatcher` 的 `.` 分隔符参数如果不传（使用默认的 `/`），会发生什么？为什么这里必须用 `.`？
3. `WebFrameworkUtils.getLoginUserType()` 通过 URL 前缀推断用户类型。如果一个请求是 RPC 调用（URL 以 `/rpc-api/` 开头），这个方法会返回什么？为什么这样设计？
