# API 文档与访问日志设计

## 两件"看不见但至关重要"的事

在电商后台的日常开发中，有两个基础设施虽然不直接产生业务价值，但对开发效率和运维能力至关重要：

1. **API 文档**：前后端分离开发的基础。没有文档，前端开发者不知道接口长什么样、参数是什么、返回什么。
2. **访问日志**：线上排查的基础。没有日志，出了问题你只能问用户"你点了什么"。

本章讲解框架如何自动生成 API 文档，以及如何自动采集访问日志。

## API 文档：OpenAPI 3 + Knife4j

### 技术选型

| 技术 | 角色 | 说明 |
|------|------|------|
| SpringDoc OpenAPI | 文档生成引擎 | 从 Spring MVC 的 `@RequestMapping` 等注解自动推断 API 结构 |
| OpenAPI 3 规范 | 文档格式标准 | Swagger 2.0 的升级版，更强大的 Schema 支持 |
| Knife4j | 增强 UI | 在 Swagger UI 基础上提供更好的文档浏览体验 |

### YudaoSwaggerAutoConfiguration 配置类

```java
// 文件：cn.iocoder.yudao.framework.swagger.config.YudaoSwaggerAutoConfiguration

@AutoConfiguration(before = Knife4jAutoConfiguration.class)
@ConditionalOnClass({OpenAPI.class})
@EnableConfigurationProperties(SwaggerProperties.class)
@ConditionalOnProperty(prefix = "springdoc.api-docs", name = "enabled",
                        havingValue = "true", matchIfMissing = true)
@Import(Knife4jOpenApiCustomizer.class)
public class YudaoSwaggerAutoConfiguration {

    @Bean
    public OpenAPI createApi(SwaggerProperties properties) {
        Map<String, SecurityScheme> securitySchemes = buildSecuritySchemes();
        OpenAPI openAPI = new OpenAPI()
                .info(buildInfo(properties))
                .components(new Components().securitySchemes(securitySchemes))
                .addSecurityItem(new SecurityRequirement().addList(HttpHeaders.AUTHORIZATION));
        securitySchemes.keySet().forEach(key ->
                openAPI.addSecurityItem(new SecurityRequirement().addList(key)));
        return openAPI;
    }

    @Bean
    public GroupedOpenApi allGroupedOpenApi() {
        return buildGroupedOpenApi("all", "");
    }

    public static GroupedOpenApi buildGroupedOpenApi(String group, String path) {
        return GroupedOpenApi.builder()
                .group(group)
                .pathsToMatch("/admin-api/" + path + "/**", "/app-api/" + path + "/**")
                .addOperationCustomizer((operation, handlerMethod) -> operation
                        .addParametersItem(buildTenantHeaderParameter())
                        .addParametersItem(buildSecurityHeaderParameter()))
                .addOperationCustomizer(buildOperationIdCustomizer())
                .build();
    }
}
```

### 自动注入的公共参数

每个 API 接口都会自动注入两个请求头参数，无需手动添加：

**1. tenant-id（租户编号）**

```java
private static Parameter buildTenantHeaderParameter() {
    return new Parameter()
            .name("tenant-id")
            .description("租户编号")
            .in("header")
            .schema(new IntegerSchema()._default(1L).name("tenant-id").description("租户编号"));
}
```

多租户是项目的核心特性之一。每个 API 请求都需要携带 `tenant-id` 请求头，告诉后端当前操作属于哪个租户。在文档中自动注入这个参数，前端开发者不会忘记传递。

**2. Authorization（认证 Token）**

```java
private static Parameter buildSecurityHeaderParameter() {
    return new Parameter()
            .name("Authorization")
            .description("认证 Token")
            .in("header")
            .schema(new StringSchema()._default("Bearer test1").name("tenant-id")
                    .description("认证 Token"));
}
```

默认值设为 `"Bearer test1"`，方便在开发环境中直接测试。生产环境中前端会替换为真实的 Token。

这两个参数的自动注入意味着：**Controller 的方法签名中不需要声明这两个参数**，它们由框架层（认证 Filter、租户 Filter）从请求头中提取并设置到上下文中。

### operationId 命名规则

OpenAPI 规范中，每个接口有一个 `operationId`，用于唯一标识一个操作。默认情况下，SpringDoc 使用方法名作为 operationId，但不同 Controller 中可能有同名方法（如 `list`、`get`、`delete`），导致冲突。

框架自定义了 operationId 的生成规则：

```java
private static OperationCustomizer buildOperationIdCustomizer() {
    return (operation, handlerMethod) -> {
        String className = handlerMethod.getBeanType().getSimpleName();
        String classPrefix = className.replaceAll("Controller$", "");
        String methodName = handlerMethod.getMethod().getName();
        String operationId = classPrefix + "_" + methodName;
        operation.setOperationId(operationId);
        return operation;
    };
}
```

生成规则：`{类名去掉Controller后缀}_{方法名}`

| Controller 类 | 方法名 | operationId |
|--------------|--------|-------------|
| `AdminProductSpuController` | `getSpuPage` | `AdminProductSpu_getSpuPage` |
| `AppProductSpuController` | `getSpuDetail` | `AppProductSpu_getSpuDetail` |
| `AdminOrderController` | `list` | `AdminOrder_list` |

这样即使管理端和用户端都有 `list` 方法，operationId 也不会冲突。

### 分组机制

`buildGroupedOpenApi` 方法创建了分组的 API 文档。默认创建了一个 `all` 分组，匹配所有 `/admin-api/**` 和 `/app-api/**` 的路径。

业务模块可以创建自己的分组：

```java
// 在业务模块的配置类中
@Bean
public GroupedOpenApi productGroupedOpenApi() {
    return YudaoSwaggerAutoConfiguration.buildGroupedOpenApi("product");
}
```

这会创建一个 `product` 分组，只显示 `/admin-api/product/**` 和 `/app-api/product/**` 的接口。

### Knife4jOpenApiCustomizer：增强文档属性

```java
// 文件：cn.iocoder.yudao.framework.swagger.config.Knife4jOpenApiCustomizer

@Primary
@Configuration
public class Knife4jOpenApiCustomizer
        extends com.github.xiaoymin.knife4j.spring.extension.Knife4jOpenApiCustomizer
        implements GlobalOpenApiCustomizer {

    @Override
    public void customise(OpenAPI openApi) {
        if (knife4jProperties.isEnable()) {
            // 注入 Knife4j 的增强配置（设置、文档等）
            OpenApiExtensionResolver resolver = new OpenApiExtensionResolver(
                    setting, knife4jProperties.getDocuments());
            resolver.start();
            Map<String, Object> objectMap = new HashMap<>();
            objectMap.put(GlobalConstants.EXTENSION_OPEN_SETTING_NAME, setting);
            objectMap.put(GlobalConstants.EXTENSION_OPEN_MARKDOWN_NAME,
                    resolver.getMarkdownFiles());
            openApi.addExtension(GlobalConstants.EXTENSION_OPEN_API_NAME, objectMap);
            // 注入 Tag 的排序属性
            addOrderExtension(openApi);
        }
    }
}
```

这个自定义类做了两件事：
1. 注入 Knife4j 的扩展属性（文档设置、Markdown 文件等）
2. 扫描 `@ApiSupport(order = N)` 注解，为 Tag 添加排序属性，让文档中的接口分组可以按指定顺序展示

### SwaggerProperties 配置

```java
// 文件：cn.iocoder.yudao.framework.swagger.config.SwaggerProperties

@ConfigurationProperties("yudao.swagger")
@Data
public class SwaggerProperties {
    private String title;        // 文档标题
    private String description;  // 文档描述
    private String author;       // 作者
    private String version;      // 版本
    private String url;          // 项目 URL
    private String email;        // 联系邮箱
    private String license;      // 许可证
    private String licenseUrl;   // 许可证 URL
}
```

配置示例：

```yaml
yudao:
  swagger:
    title: 电商后台管理系统 API
    description: 提供商品、订单、库存等管理接口
    author: 芋道源码
    version: 1.0.0
    url: https://www.iocoder.cn
    email: yudao@iocoder.cn
    license: MIT
    license-url: https://opensource.org/licenses/MIT
```

### 文档禁用

可以通过配置禁用 Swagger 文档（生产环境通常禁用）：

```yaml
springdoc:
  api-docs:
    enabled: false
```

或者只禁用 Knife4j：

```yaml
knife4j:
  enable: false
```

## 访问日志：ApiAccessLogFilter + ApiAccessLogInterceptor

### 设计动机

线上排查问题时，最常问的三个问题是：
1. 用户调了什么接口？传了什么参数？
2. 接口返回了什么？耗时多久？
3. 如果出错了，错误原因是什么？

访问日志就是要回答这三个问题。

### 双层协作：Filter + Interceptor

访问日志的采集由两个组件协作完成：

```
请求进入
    │
    ▼
ApiAccessLogFilter（Filter 层）
    │  记录请求开始时间
    │  提前获取 Query 参数和 Body（避免 XSS Filter 修改后丢失原始数据）
    │
    ▼
    ├─ 继续过滤器链 ─→ Controller 执行 ─→ 返回
    │
    ▼
ApiAccessLogFilter 的 finally 块
    │  获取 HandlerMethod（由 Interceptor 存入 Request 属性）
    │  读取 @ApiAccessLog 注解
    │  构建访问日志
    │  异步写入数据库
    │
    ▼
返回响应
```

为什么需要 Interceptor？因为 Filter 层无法直接获取 `HandlerMethod`（Controller 方法的元信息）。`HandlerMethod` 包含了方法名、参数、注解等信息，只有 Spring MVC 的 `HandlerInterceptor` 才能拿到。

### ApiAccessLogInterceptor

```java
// 文件：cn.iocoder.yudao.framework.apilog.core.interceptor.ApiAccessLogInterceptor

public class ApiAccessLogInterceptor implements HandlerInterceptor {

    public static final String ATTRIBUTE_HANDLER_METHOD = "HANDLER_METHOD";

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response, Object handler) {
        // 记录 HandlerMethod，提供给 ApiAccessLogFilter 使用
        HandlerMethod handlerMethod = handler instanceof HandlerMethod
                ? (HandlerMethod) handler : null;
        if (handlerMethod != null) {
            request.setAttribute(ATTRIBUTE_HANDLER_METHOD, handlerMethod);
        }

        // 非生产环境：打印请求日志
        if (!SpringUtils.isProd()) {
            Map<String, String> queryString = ServletUtils.getParamMap(request);
            String requestBody = ServletUtils.getBody(request);
            if (CollUtil.isEmpty(queryString) && StrUtil.isEmpty(requestBody)) {
                log.info("[preHandle][开始请求 URL({}) 无参数]", request.getRequestURI());
            } else {
                log.info("[preHandle][开始请求 URL({}) 参数({})]",
                        request.getRequestURI(),
                        StrUtil.blankToDefault(requestBody, queryString.toString()));
            }
            // 计时
            StopWatch stopWatch = new StopWatch();
            stopWatch.start();
            request.setAttribute(ATTRIBUTE_STOP_WATCH, stopWatch);
            // 打印 Controller 方法的源码位置
            printHandlerMethodPosition(handlerMethod);
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                 HttpServletResponse response,
                                 Object handler, Exception ex) {
        if (!SpringUtils.isProd()) {
            StopWatch stopWatch = (StopWatch) request.getAttribute(ATTRIBUTE_STOP_WATCH);
            stopWatch.stop();
            log.info("[afterCompletion][完成请求 URL({}) 耗时({} ms)]",
                    request.getRequestURI(), stopWatch.getTotalTimeMillis());
        }
    }
}
```

Interceptor 的职责：
1. **存储 HandlerMethod**：存入 Request 属性，供 Filter 层读取
2. **开发环境日志**：在非生产环境打印请求参数、响应耗时、Controller 源码位置

`printHandlerMethodPosition` 方法会尝试读取 Controller 方法的源码文件，定位到具体的行号，方便开发者在 IDE 中快速跳转。

### ApiAccessLogFilter

```java
// 文件：cn.iocoder.yudao.framework.apilog.core.filter.ApiAccessLogFilter

public class ApiAccessLogFilter extends ApiRequestFilter {

    private static final String[] SANITIZE_KEYS = {
        "password", "token", "accessToken", "refreshToken"
    };

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {
        LocalDateTime beginTime = LocalDateTime.now();
        // 提前获取参数，避免 XssFilter 修改后丢失原始数据
        Map<String, String> queryString = ServletUtils.getParamMap(request);
        String requestBody = ServletUtils.getBody(request);

        try {
            filterChain.doFilter(request, response);
            createApiAccessLog(request, beginTime, queryString, requestBody, null);
        } catch (Exception ex) {
            createApiAccessLog(request, beginTime, queryString, requestBody, ex);
            throw ex;
        }
    }
}
```

注意一个细节：Filter 在调用 `filterChain.doFilter()` **之前**就获取了 Query 参数和 Body。这是因为后续的 XssFilter 可能会修改参数值，访问日志应该记录原始的请求数据。

### @ApiAccessLog 注解

Controller 方法可以通过 `@ApiAccessLog` 注解控制日志行为：

```java
// 文件：cn.iocoder.yudao.framework.apilog.core.annotation.ApiAccessLog

@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ApiAccessLog {
    // 开关字段
    boolean enable() default true;           // 是否记录日志
    boolean requestEnable() default true;    // 是否记录请求参数
    boolean responseEnable() default false;  // 是否记录响应结果
    String[] sanitizeKeys() default {};      // 额外的脱敏字段

    // 模块字段
    String operateModule() default "";       // 操作模块
    String operateName() default "";         // 操作名
    OperateTypeEnum[] operateType() default {};  // 操作类型
}
```

使用示例：

```java
// 导出商品数据——记录日志，标记为 EXPORT 操作
@GetMapping("/export")
@ApiAccessLog(operateType = OperateTypeEnum.EXPORT)
public void export(HttpServletResponse response, ProductExportReqVO reqVO) { ... }

// 查询商品列表——不记录日志（高频查询，日志量太大）
@GetMapping("/list")
@ApiAccessLog(enable = false)
public CommonResult<List<ProductRespVO>> list(@Valid ProductPageReqVO reqVO) { ... }

// 修改密码——记录日志，但脱敏 password 字段
@PutMapping("/update-password")
@ApiAccessLog(sanitizeKeys = "newPassword")
public CommonResult<Boolean> updatePassword(@Valid @RequestBody PasswordUpdateReqVO reqVO) { ... }

// 查询订单详情——记录日志，同时记录响应结果（方便排查）
@GetMapping("/get")
@ApiAccessLog(responseEnable = true)
public CommonResult<OrderRespVO> getOrder(@RequestParam("id") Long id) { ... }
```

### 日志内容的构建

`buildApiAccessLog` 方法构建完整的访问日志：

```java
private boolean buildApiAccessLog(ApiAccessLogCreateReqDTO accessLog,
                                   HttpServletRequest request, LocalDateTime beginTime,
                                   Map<String, String> queryString,
                                   String requestBody, Exception ex) {
    // 1. 检查 @ApiAccessLog 注解，决定是否记录
    HandlerMethod handlerMethod = (HandlerMethod) request.getAttribute(ATTRIBUTE_HANDLER_METHOD);
    ApiAccessLog accessLogAnnotation = null;
    if (handlerMethod != null) {
        accessLogAnnotation = handlerMethod.getMethodAnnotation(ApiAccessLog.class);
        if (accessLogAnnotation != null && BooleanUtil.isFalse(accessLogAnnotation.enable())) {
            return false;  // 注解明确禁用
        }
    }

    // 2. 用户信息
    accessLog.setUserId(WebFrameworkUtils.getLoginUserId(request))
            .setUserType(WebFrameworkUtils.getLoginUserType(request));

    // 3. 访问结果（从 GlobalResponseBodyHandler 存储的 CommonResult 中获取）
    CommonResult<?> result = WebFrameworkUtils.getCommonResult(request);
    if (result != null) {
        accessLog.setResultCode(result.getCode()).setResultMsg(result.getMsg());
    } else if (ex != null) {
        accessLog.setResultCode(INTERNAL_SERVER_ERROR.getCode())
                .setResultMsg(ExceptionUtil.getRootCauseMessage(ex));
    }

    // 4. 请求信息
    accessLog.setTraceId(TracerUtils.getTraceId())
            .setApplicationName(applicationName)
            .setRequestUrl(request.getRequestURI())
            .setRequestMethod(request.getMethod())
            .setUserAgent(ServletUtils.getUserAgent(request))
            .setUserIp(ServletUtils.getClientIP(request));

    // 5. 请求参数（脱敏后）
    accessLog.setRequestParams(toJsonString(MapUtil.<String, Object>builder()
            .put("query", sanitizeMap(queryString, sanitizeKeys))
            .put("body", sanitizeJson(requestBody, sanitizeKeys)).build()));

    // 6. 持续时间
    accessLog.setBeginTime(beginTime)
            .setEndTime(LocalDateTime.now())
            .setDuration((int) LocalDateTimeUtil.between(beginTime,
                    LocalDateTime.now(), ChronoUnit.MILLIS));

    // 7. 操作信息（从注解或 Swagger 注解中获取）
    if (handlerMethod != null) {
        Tag tagAnnotation = handlerMethod.getBeanType().getAnnotation(Tag.class);
        Operation operationAnnotation = handlerMethod.getMethodAnnotation(Operation.class);
        // 优先使用 @ApiAccessLog 的配置，其次使用 Swagger 注解
        String operateModule = ...;  // @ApiAccessLog.operateModule() > @Tag.name()
        String operateName = ...;    // @ApiAccessLog.operateName() > @Operation.summary()
        OperateTypeEnum operateType = ...;  // @ApiAccessLog.operateType() > HTTP 方法推断
    }
    return true;
}
```

### 自动脱敏

访问日志中的敏感字段会被自动移除，防止密码、Token 等信息泄露到日志系统。

**默认脱敏字段**：

```java
private static final String[] SANITIZE_KEYS = {
    "password", "token", "accessToken", "refreshToken"
};
```

**自定义脱敏字段**：通过 `@ApiAccessLog(sanitizeKeys = {"newPassword", "oldPassword"})` 指定。

**脱敏逻辑**：

```java
private static String sanitizeMap(Map<String, ?> map, String[] sanitizeKeys) {
    if (CollUtil.isEmpty(map)) return null;
    if (sanitizeKeys != null) {
        MapUtil.removeAny(map, sanitizeKeys);  // 移除自定义脱敏字段
    }
    MapUtil.removeAny(map, SANITIZE_KEYS);     // 移除默认脱敏字段
    return JsonUtils.toJsonString(map);
}
```

对于 JSON 格式的请求参数和响应结果，脱敏会递归处理嵌套的 JSON 对象和数组：

```java
private static void sanitizeJson(JsonNode node, String[] sanitizeKeys) {
    if (node.isArray()) {
        for (JsonNode childNode : node) {
            sanitizeJson(childNode, sanitizeKeys);  // 递归处理数组元素
        }
        return;
    }
    if (!node.isObject()) return;
    Iterator<Map.Entry<String, JsonNode>> iterator = node.fields();
    while (iterator.hasNext()) {
        Map.Entry<String, JsonNode> entry = iterator.next();
        if (ArrayUtil.contains(sanitizeKeys, entry.getKey())
                || ArrayUtil.contains(SANITIZE_KEYS, entry.getKey())) {
            iterator.remove();  // 移除敏感字段
            continue;
        }
        sanitizeJson(entry.getValue(), sanitizeKeys);  // 递归处理嵌套对象
    }
}
```

### 操作类型的自动推断

如果 `@ApiAccessLog` 注解没有指定 `operateType`，框架会根据 HTTP 方法自动推断：

| HTTP 方法 | 推断的操作类型 |
|-----------|--------------|
| GET | 查询 (GET) |
| POST | 新增 (CREATE) |
| PUT | 修改 (UPDATE) |
| DELETE | 删除 (DELETE) |
| 其他 | 其他 (OTHER) |

这种推断在大多数情况下是准确的，因为项目的接口遵循 RESTful 风格。如果不准确，可以通过注解显式指定。

### 操作模块和操作名的自动获取

日志中的"操作模块"和"操作名"有三个来源，按优先级排序：

1. `@ApiAccessLog(operateModule = "商品管理", operateName = "修改商品价格")` — 注解显式指定
2. `@Tag(name = "商品管理")` + `@Operation(summary = "修改商品价格")` — Swagger 注解
3. HTTP 方法推断 — 兜底方案

这种设计让 Swagger 注解同时服务于 API 文档和访问日志，避免了重复定义。

## 访问日志与错误日志的协作

访问日志和错误日志（见[第 3 章](03-全局异常处理设计.md)）是互补的：

| 维度 | 访问日志 | 错误日志 |
|------|---------|---------|
| 记录时机 | 每个 API 请求 | 只在系统异常时 |
| 记录内容 | 请求参数、响应结果、耗时 | 异常堆栈、完整上下文 |
| 脱敏处理 | 自动脱敏敏感字段 | 不脱敏（内部排查用） |
| 用途 | 审计、统计、排查 | 排查 Bug |

```
请求进入
    │
    ▼
ApiAccessLogFilter 开始计时
    │
    ▼
Controller 执行
    ├── 成功 → 访问日志（正常结果）
    │
    └── 抛出 ServiceException → 访问日志（业务错误）
         │
         └── 抛出 Exception → 访问日志（系统错误）
                              + 错误日志（持久化到数据库）
```

两者的记录是独立的——即使访问日志记录失败，错误日志仍然会尝试记录；反之亦然。

## 本章小结

API 文档和访问日志的设计体现了"自动化"和"零侵入"的理念：

**API 文档**：
1. 从代码自动推断 API 结构，无需手动维护文档
2. 自动注入公共参数（tenant-id、Authorization），前端不会遗漏
3. operationId 自动去重，避免多模块同名方法冲突
4. 与 Knife4j 集成，提供增强的文档浏览体验

**访问日志**：
1. Filter + Interceptor 双层协作，Filter 负责采集，Interceptor 负责传递 HandlerMethod
2. 提前获取请求参数，避免后续 Filter 修改导致日志不准确
3. 自动脱敏，密码、Token 等敏感字段不会出现在日志中
4. 操作信息复用 Swagger 注解，避免重复定义
5. 异步持久化，不阻塞正常请求

---

**思考题**

1. ApiAccessLogFilter 在调用 `filterChain.doFilter()` 之前就获取了请求参数。如果请求是文件上传（multipart），`ServletUtils.getBody(request)` 能正常工作吗？框架是否需要处理这种情况？
2. 访问日志的"操作模块"优先使用 `@ApiAccessLog` 的配置，其次使用 `@Tag` 注解。如果两个注解都没有，日志中的操作模块字段会是什么值？这对日志分析有什么影响？
3. 框架默认记录请求参数但不记录响应结果（`responseEnable = false`）。如果要排查"接口返回了错误数据"的问题，你会怎么做？这个默认设置合理吗？
4. Swagger 文档在生产环境中通常会被禁用。除了安全考虑，还有什么原因？如果生产环境需要 API 文档，有哪些替代方案？
