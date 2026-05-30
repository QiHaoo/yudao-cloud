# 07-核心设计：免认证 URL 管理

## 场景切入

不是所有接口都需要登录才能访问。典型的免认证接口包括：

- **登录接口**：`POST /system/auth/login` —— 用户还没登录，怎么要求他先登录？
- **注册接口**：`POST /system/auth/register` —— 新用户还没有账号
- **验证码接口**：`GET /system/captcha/get` —— 登录前就需要获取验证码
- **Swagger 文档**：`GET /v3/api-docs/**` —— 开发调试用
- **第三方回调**：`POST /pay/notify/**` —— 支付平台回调通知

问题是：怎么告诉 Spring Security"这些 URL 不需要认证"？

## 三种免认证机制

框架提供了三种互补的免认证机制，覆盖不同场景：

```
免认证 URL 的三个来源
  │
  ├── 1. @PermitAll 注解          ← 接口级别，标记在 Controller 方法/类上
  │
  ├── 2. yudao.security.permit-all-urls  ← 配置级别，在 application.yml 中配置
  │
  └── 3. AuthorizeRequestsCustomizer    ← 模块级别，每个 Maven 模块自定义规则
```

### 机制一：@PermitAll 注解

`@PermitAll` 是 JDK 标准注解（`javax.annotation.security.PermitAll`），可以标记在 Controller 的方法或类上。

**方法级标记**：只有该方法免认证
```java
@PostMapping("/login")
@PermitAll  // 只有登录接口免认证
public CommonResult<AuthLoginRespVO> login(@Valid @RequestBody AuthLoginReqVO reqVO) {
    return success(authService.login(reqVO));
}
```

**类级标记**：该 Controller 的所有方法都免认证
```java
@RestController
@PermitAll  // 整个 Controller 免认证
public class CaptchaController {
    // 所有接口都不需要登录
}
```

#### 自动扫描机制

框架在启动时自动扫描所有 Controller，收集带 `@PermitAll` 注解的 URL：

```java
private Multimap<HttpMethod, String> getPermitAllUrlsFromAnnotations() {
    Multimap<HttpMethod, String> result = HashMultimap.create();

    // 获取所有 Controller 的映射关系
    RequestMappingHandlerMapping handlerMapping = applicationContext.getBean("requestMappingHandlerMapping");
    Map<RequestMappingInfo, HandlerMethod> handlerMethodMap = handlerMapping.getHandlerMethods();

    // 遍历，找出有 @PermitAll 的接口
    for (Map.Entry<RequestMappingInfo, HandlerMethod> entry : handlerMethodMap.entrySet()) {
        HandlerMethod handlerMethod = entry.getValue();
        // 检查方法级或类级注解
        if (!handlerMethod.hasMethodAnnotation(PermitAll.class)
                && !handlerMethod.getBeanType().isAnnotationPresent(PermitAll.class)) {
            continue;
        }
        // 收集 URL 和对应的 HTTP 方法
        // ...
    }
    return result;
}
```

这里有一个重要的细节：`@PermitAll` 的扫描是**按 HTTP 方法**分组的。`GET /api/users` 和 `POST /api/users` 是两个不同的权限规则。

如果一个方法使用了 `@RequestMapping`（没有指定 method），则认为**所有 HTTP 方法**都需要免认证：
```java
@RequestMapping("/health")  // 没有指定 method
@PermitAll
public CommonResult<String> health() {
    return success("ok");
}
// 结果：GET/POST/PUT/DELETE /health 都免认证
```

### 机制二：配置化免认证 URL

有些 URL 不在业务代码中（如 Druid 监控、Actuator 健康检查），或者需要批量配置。这时可以用 `yudao.security.permit-all-urls`：

```yaml
yudao:
  security:
    permit-all-urls:
      - /druid/**          # Druid 监控
      - /actuator/**       # Actuator 健康检查
      - /error             # Spring 错误页面
```

这些 URL 会被直接加入 `SecurityFilterChain` 的 `permitAll()` 规则中。

### 机制三：AuthorizeRequestsCustomizer

每个 Maven 模块可能有自己的特殊需求。框架提供了 `AuthorizeRequestsCustomizer` 抽象类，每个模块可以实现它来自定义规则：

```java
// system 模块的自定义规则
@Component
public class SecurityConfiguration extends AuthorizeRequestsCustomizer {

    @Override
    public void customize(AuthorizeHttpRequestsConfigurer<HttpSecurity>
            .AuthorizationManagerRequestMatcherRegistry registry) {
        // 放行 Swagger
        registry.requestMatchers(buildAdminApi("/v3/api-docs/**")).permitAll();
        // 放行 Druid
        registry.requestMatchers("/druid/**").permitAll();
        // 放行 RPC 服务路径
        registry.requestMatchers(ApiConstants.PREFIX + "/**").permitAll();
    }
}
```

`buildAdminApi()` 和 `buildAppApi()` 是父类提供的工具方法，自动拼接 URL 前缀（如 `/admin-api`）。

#### 为什么需要这个机制？

`@PermitAll` 只能标记在业务 Controller 上，但有些免认证 URL 来自框架组件（如 Druid 监控页面、Spring Boot Actuator），这些组件的 Controller 不在我们的代码中。`AuthorizeRequestsCustomizer` 提供了一个统一的扩展点，让每个模块都能声明自己的免认证规则。

## 三层规则的执行顺序

```
请求到达 SecurityFilterChain
  │
  ▼
① 全局共享规则（YudaoWebSecurityConfigurerAdapter.filterChain）
  ├── 静态资源：*.html, *.css, *.js → permitAll
  ├── @PermitAll 注解扫描结果 → permitAll
  └── yudao.security.permit-all-urls → permitAll
  │
  ▼
② 模块自定义规则（AuthorizeRequestsCustomizer）
  ├── Swagger → permitAll
  ├── Druid → permitAll
  └── RPC 路径 → permitAll
  │
  ▼
③ 兜底规则
  ├── ASYNC DispatcherType → permitAll（SSE 场景）
  └── anyRequest().authenticated()（必须认证）
```

**关键：规则按声明顺序匹配，先匹配到的先执行。** 所以：
- `@PermitAll` 和配置化 URL 的优先级最高
- 模块自定义规则次之
- 兜底规则要求"其他所有请求必须认证"

## 实际使用示例

### 场景一：新接口免认证

```java
// 方式一：使用 @PermitAll 注解
@GetMapping("/public/announcement")
@PermitAll
public CommonResult<AnnouncementRespVO> getAnnouncement() {
    return success(announcementService.getLatest());
}

// 方式二：在配置文件中添加
// yudao.security.permit-all-urls:
//   - /admin-api/public/**
```

推荐使用方式一，因为注解与代码紧耦合，更直观。

### 场景二：整个 Controller 免认证

```java
@RestController
@RequestMapping("/system/captcha")
@PermitAll  // 整个验证码 Controller 免认证
public class CaptchaController {
    @GetMapping("/get")
    public CommonResult<CaptchaRespVO> getCaptcha() { ... }

    @PostMapping("/check")
    public CommonResult<Boolean> checkCaptcha(@Valid @RequestBody CaptchaCheckReqVO reqVO) { ... }
}
```

### 场景三：模块级别的免认证

```java
// infra 模块需要放行代码生成器的预览页面
@Component
public class CodegenSecurityConfiguration extends AuthorizeRequestsCustomizer {
    @Override
    public void customize(AuthorizeHttpRequestsConfigurer<HttpSecurity>
            .AuthorizationManagerRequestMatcherRegistry registry) {
        registry.requestMatchers(buildAdminApi("/infra/codegen/preview/**")).permitAll();
    }
}
```

## 安全注意事项

### 1. 免认证 ≠ 无风险

即使接口不需要登录，也要注意：
- **输入校验**：免认证接口同样可能被注入攻击
- **限流保护**：防止恶意调用（如暴力破解验证码）
- **敏感信息**：免认证接口不应返回敏感数据

### 2. 避免过度使用 @PermitAll

最常见的安全漏洞之一就是"忘记给接口加认证"。框架的兜底规则（`anyRequest().authenticated()`）就是为了防止这种情况——默认所有接口都需要认证，只有显式标记的才免认证。

### 3. 模块化带来的安全边界

当项目拆分为多个微服务时，每个服务的 `AuthorizeRequestsCustomizer` 独立配置。要确保：
- 网关层有统一的 Token 校验
- 各服务的免认证规则不会互相冲突
- RPC 接口（`/rpc-api/**`）只在内网可访问

## 思考题

1. **@PermitAll 注解的扫描是在启动时完成的，如果运行时动态添加了一个 Controller（如通过热部署），@PermitAll 还能生效吗？** 提示：考虑 `RequestMappingHandlerMapping` 的刷新时机。

2. **为什么框架用 Guava 的 `Multimap<HttpMethod, String>` 而不是普通的 `Map<HttpMethod, List<String>>` 来存储免认证 URL？** 提示：考虑去重和多值场景。

3. **如果一个 URL 同时被 @PermitAll 和 @PreAuthorize 标记，会发生什么？** 这种情况是设计缺陷还是合理的？提示：考虑"公开接口但限制操作频率"的场景。

4. **在微服务架构中，免认证 URL 的配置应该放在网关层还是服务层？** 各有什么优劣？
