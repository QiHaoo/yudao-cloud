# URL 前缀注入实现

> 本章是功能实现的第二章，对应功能设计中的 [URL 前缀注入机制设计](../功能设计/02-URL前缀注入机制设计.md)。设计章节讲了"为什么需要 URL 前缀注入"——避免在每个 Controller 上硬编码 `/admin-api` 或 `/app-api`。本章讲"怎么实现"。

## 1. 问题回顾

在没有前缀注入的项目中，每个 Controller 都要手动写前缀：

```java
@RestController
@RequestMapping("/admin-api/product/spu")  // 前缀硬编码
public class AdminProductSpuController { ... }

@RestController
@RequestMapping("/app-api/product/spu")   // 前缀硬编码
public class AppProductSpuController { ... }
```

我们希望的效果是：Controller 只写业务路径，前缀根据包路径自动注入。

## 2. 核心机制：WebMvcRegistrations

Spring Boot 提供了 `WebMvcRegistrations` 接口，允许我们自定义 Spring MVC 的核心组件。其中 `getRequestMappingHandlerMapping()` 方法可以替换默认的 `RequestMappingHandlerMapping`。

`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/web/config/YudaoWebAutoConfiguration.java`（URL 前缀相关部分）:

```java
@Bean
public WebMvcRegistrations webMvcRegistrations(WebProperties webProperties) {
    return new WebMvcRegistrations() {

        @Override
        public RequestMappingHandlerMapping getRequestMappingHandlerMapping() {
            RequestMappingHandlerMapping mapping = new RequestMappingHandlerMapping();
            // 实例化时就带上前缀
            mapping.setPathPrefixes(buildPathPrefixes(webProperties));
            return mapping;
        }

        /**
         * 构建 prefix → 匹配条件的映射
         */
        private Map<String, Predicate<Class<?>>> buildPathPrefixes(WebProperties webProperties) {
            AntPathMatcher antPathMatcher = new AntPathMatcher(".");
            Map<String, Predicate<Class<?>>> pathPrefixes = Maps.newLinkedHashMapWithExpectedSize(2);
            putPathPrefix(pathPrefixes, webProperties.getAdminApi(), antPathMatcher);
            putPathPrefix(pathPrefixes, webProperties.getAppApi(), antPathMatcher);
            return pathPrefixes;
        }

        /**
         * 设置 API 前缀，仅仅匹配 controller 包下的
         */
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

### 2.1 工作原理

这段代码的核心是 `RequestMappingHandlerMapping.setPathPrefixes()` 方法。这是 Spring MVC 5.3+ 引入的特性，允许你为满足特定条件的 Controller 类自动添加 URL 前缀。

让我们逐步拆解 `buildPathPrefixes` 方法的执行过程：

```
输入：WebProperties 配置
  adminApi.prefix = "/admin-api"
  adminApi.controller = "**.controller.admin.**"
  appApi.prefix = "/app-api"
  appApi.controller = "**.controller.app.**"

输出：Map<String, Predicate<Class<?>>>
  "/admin-api" → (clazz) -> clazz 有 @RestController 注解
                        && clazz 的包名匹配 **.controller.admin.**
  "/app-api"   → (clazz) -> clazz 有 @RestController 注解
                        && clazz 的包名匹配 **.controller.app.**
```

当 Spring MVC 处理请求时，`RequestMappingHandlerMapping` 会遍历这个 Map，对每个 Controller 类执行 `Predicate` 测试。如果匹配成功，该 Controller 的所有接口都会自动加上对应的前缀。

### 2.2 AntPathMatcher 的分隔符

注意 `new AntPathMatcher(".")` 这一行。默认的 `AntPathMatcher` 用 `/` 作为路径分隔符，但这里改为 `.`，因为我们要匹配的是 Java 包名（用 `.` 分隔），不是文件路径。

```
包名：cn.iocoder.yudao.module.product.controller.admin
匹配：**.controller.admin.**
      ^                ^
      |                |
      任意前缀         任意后缀
```

如果用默认的 `/` 分隔符，`**.controller.admin.**` 就无法正确匹配包名了。

### 2.3 Predicate 的两个条件

```java
clazz -> clazz.isAnnotationPresent(RestController.class)
        && matcher.match(api.getController(), clazz.getPackage().getName())
```

- **`isAnnotationPresent(RestController.class)`**：只对 `@RestController` 生效，普通的 `@Controller`（返回页面的）不受影响
- **`matcher.match(...)`**：包路径必须匹配配置的 Ant 规则

两个条件同时满足，才会注入前缀。

## 3. 配置属性类

URL 前缀的配置来自 `WebProperties`：

`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/web/config/WebProperties.java`:

```java
@ConfigurationProperties(prefix = "yudao.web")
@Validated
@Data
public class WebProperties {

    @NotNull(message = "APP API 不能为空")
    private Api appApi = new Api("/app-api", "**.controller.app.**");

    @NotNull(message = "Admin API 不能为空")
    private Api adminApi = new Api("/admin-api", "**.controller.admin.**");

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
}
```

默认值已经配好，开箱即用。如果需要自定义，在 `application.yml` 中覆盖即可：

```yaml
yudao:
  web:
    admin-api:
      prefix: /manage-api              # 改前缀
      controller: "**.controller.admin.**"
    app-api:
      prefix: /client-api
      controller: "**.controller.app.**"
```

## 4. 完整流程图

```
应用启动
  │
  ▼
YudaoWebAutoConfiguration 加载
  │
  ├─ 读取 WebProperties（yudao.web.*）
  │
  ▼
创建 WebMvcRegistrations Bean
  │
  ├─ buildPathPrefixes()
  │    ├─ 读取 adminApi → "/admin-api" + "**.controller.admin.**"
  │    └─ 读取 appApi   → "/app-api"   + "**.controller.app.**"
  │
  ▼
Spring MVC 注册 RequestMappingHandlerMapping
  │
  ├─ setPathPrefixes(pathPrefixes)
  │
  ▼
请求到达（如 GET /admin-api/product/spu/list）
  │
  ▼
RequestMappingHandlerMapping 查找匹配的 Controller
  │
  ├─ 遍历所有 @RestController 类
  ├─ 对 AdminProductSpuController 执行 Predicate
  │    ├─ @RestController? ✓
  │    └─ 包名匹配 **.controller.admin.**? ✓
  ├─ 匹配成功，注入前缀 "/admin-api"
  │
  ▼
@RequestMapping("/product/spu") + 前缀 "/admin-api"
  = 最终路径 "/admin-api/product/spu"
  │
  ▼
匹配成功，执行 Controller 方法
```

## 5. 实际效果验证

有了 URL 前缀注入，Controller 只需要写业务路径：

```java
package cn.iocoder.yudao.module.product.controller.admin;

@RestController
@RequestMapping("/product/spu")  // 只写业务路径，不需要 /admin-api 前缀
public class AdminProductSpuController {

    @GetMapping("/list")
    public CommonResult<PageResult<ProductSpuRespVO>> getSpuPage(@Valid ProductSpuPageReqVO reqVO) {
        // ...
    }
}
```

实际访问路径：`GET /admin-api/product/spu/list`

前缀 `/admin-api` 是由 `WebMvcRegistrations` 根据包路径 `cn.iocoder.yudao.module.product.controller.admin` 自动注入的。

## 6. WebFrameworkUtils 中的类型推断

URL 前缀还有一个妙用——根据前缀推断用户类型。

`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/web/core/util/WebFrameworkUtils.java`（相关部分）:

```java
public static Integer getLoginUserType(HttpServletRequest request) {
    if (request == null) {
        return null;
    }
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

这是一个"约定大于配置"的设计：`/admin-api` 开头的请求自动识别为管理员，`/app-api` 开头的自动识别为会员。不需要额外的配置或注解。

## 7. 小结

URL 前缀注入的实现原理：

1. 通过 `WebMvcRegistrations` 接口自定义 `RequestMappingHandlerMapping`
2. 使用 `setPathPrefixes()` 方法设置"前缀 → 匹配条件"的映射
3. 匹配条件基于 `@RestController` 注解 + Ant 包路径规则
4. `AntPathMatcher` 使用 `.` 作为分隔符以匹配 Java 包名
5. 配置通过 `WebProperties` 管理，支持运行时自定义

这个机制的核心价值是**解耦**：Controller 不知道前缀的存在，前缀的变化不影响任何业务代码。

---

**导航**

- 上一章：[01-起步：骨架搭建](01-起步：骨架搭建.md)
- 下一章：[03-全局异常处理实现](03-全局异常处理实现.md)
- 相关设计文档：[02-URL前缀注入机制设计](../功能设计/02-URL前缀注入机制设计.md)
