# Swagger 与访问日志实现

> 本章是功能实现的第五章，对应功能设计中的 [API文档与访问日志设计](../功能设计/05-API文档与访问日志设计.md)。设计章节讲了"为什么需要 Swagger 文档和访问日志"。本章讲"怎么实现"。

## 1. Swagger/OpenAPI 3 配置

### 1.1 技术选型

项目使用：
- **SpringDoc OpenAPI 3**：Spring Boot 生态中 OpenAPI 3.0 规范的实现
- **Knife4j**：增强的 OpenAPI UI，提供更好的文档浏览体验

### 1.2 YudaoSwaggerAutoConfiguration

`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/swagger/config/YudaoSwaggerAutoConfiguration.java`:

```java
package cn.iocoder.yudao.framework.swagger.config;

import com.github.xiaoymin.knife4j.spring.configuration.Knife4jAutoConfiguration;
import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import io.swagger.v3.oas.models.media.IntegerSchema;
import io.swagger.v3.oas.models.media.StringSchema;
import io.swagger.v3.oas.models.parameters.Parameter;
import io.swagger.v3.oas.models.security.SecurityRequirement;
import io.swagger.v3.oas.models.security.SecurityScheme;
import org.springdoc.core.*;
import org.springdoc.core.customizers.OpenApiBuilderCustomizer;
import org.springdoc.core.customizers.OperationCustomizer;
import org.springdoc.core.customizers.ServerBaseUrlCustomizer;
import org.springdoc.core.providers.JavadocProvider;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.context.annotation.Primary;
import org.springframework.http.HttpHeaders;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import static cn.iocoder.yudao.framework.web.core.util.WebFrameworkUtils.HEADER_TENANT_ID;

@AutoConfiguration(before = Knife4jAutoConfiguration.class)
@ConditionalOnClass({OpenAPI.class})
@EnableConfigurationProperties(SwaggerProperties.class)
@ConditionalOnProperty(prefix = "springdoc.api-docs", name = "enabled",
                        havingValue = "true", matchIfMissing = true)
@Import(Knife4jOpenApiCustomizer.class)
public class YudaoSwaggerAutoConfiguration {

    // ========== 全局 OpenAPI 配置 ==========

    @Bean
    public OpenAPI createApi(SwaggerProperties properties) {
        Map<String, SecurityScheme> securitySchemas = buildSecuritySchemes();
        OpenAPI openAPI = new OpenAPI()
                .info(buildInfo(properties))
                .components(new Components().securitySchemes(securitySchemas))
                .addSecurityItem(new SecurityRequirement().addList(HttpHeaders.AUTHORIZATION));
        securitySchemas.keySet().forEach(key ->
                openAPI.addSecurityItem(new SecurityRequirement().addList(key)));
        return openAPI;
    }

    private Info buildInfo(SwaggerProperties properties) {
        return new Info()
                .title(properties.getTitle())
                .description(properties.getDescription())
                .version(properties.getVersion())
                .contact(new Contact()
                        .name(properties.getAuthor())
                        .url(properties.getUrl())
                        .email(properties.getEmail()))
                .license(new License()
                        .name(properties.getLicense())
                        .url(properties.getLicenseUrl()));
    }

    private Map<String, SecurityScheme> buildSecuritySchemes() {
        Map<String, SecurityScheme> securitySchemes = new HashMap<>();
        SecurityScheme securityScheme = new SecurityScheme()
                .type(SecurityScheme.Type.APIKEY)
                .name(HttpHeaders.AUTHORIZATION)
                .in(SecurityScheme.In.HEADER);
        securitySchemes.put(HttpHeaders.AUTHORIZATION, securityScheme);
        return securitySchemes;
    }

    @Bean
    @Primary
    public OpenAPIService openApiBuilder(Optional<OpenAPI> openAPI,
                                         SecurityService securityParser,
                                         SpringDocConfigProperties springDocConfigProperties,
                                         PropertyResolverUtils propertyResolverUtils,
                                         Optional<List<OpenApiBuilderCustomizer>> openApiBuilderCustomizers,
                                         Optional<List<ServerBaseUrlCustomizer>> serverBaseUrlCustomizers,
                                         Optional<JavadocProvider> javadocProvider) {
        return new OpenAPIService(openAPI, securityParser, springDocConfigProperties,
                propertyResolverUtils, openApiBuilderCustomizers,
                serverBaseUrlCustomizers, javadocProvider);
    }

    // ========== 分组 OpenAPI 配置 ==========

    @Bean
    public GroupedOpenApi allGroupedOpenApi() {
        return buildGroupedOpenApi("all", "");
    }

    public static GroupedOpenApi buildGroupedOpenApi(String group) {
        return buildGroupedOpenApi(group, group);
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

    private static Parameter buildTenantHeaderParameter() {
        return new Parameter()
                .name(HEADER_TENANT_ID)
                .description("租户编号")
                .in(String.valueOf(SecurityScheme.In.HEADER))
                .schema(new IntegerSchema()
                        ._default(1L)
                        .name(HEADER_TENANT_ID)
                        .description("租户编号"));
    }

    private static Parameter buildSecurityHeaderParameter() {
        return new Parameter()
                .name(HttpHeaders.AUTHORIZATION)
                .description("认证 Token")
                .in(String.valueOf(SecurityScheme.In.HEADER))
                .schema(new StringSchema()
                        ._default("Bearer test1")
                        .name(HEADER_TENANT_ID)
                        .description("认证 Token"));
    }

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
}
```

### 1.3 关键设计解读

**`@AutoConfiguration(before = Knife4jAutoConfiguration.class)`**

确保本配置在 Knife4j 之前加载。这样我们自定义的 `Knife4jOpenApiCustomizer` 能覆盖 Knife4j 默认的。

**`@ConditionalOnClass({OpenAPI.class})`**

只有引入了 OpenAPI 依赖时才生效。如果项目不需要 Swagger（比如纯后端服务），可以通过排除依赖来禁用。

**`@ConditionalOnProperty(prefix = "springdoc.api-docs", name = "enabled", havingValue = "true", matchIfMissing = true)`**

通过配置 `springdoc.api-docs.enabled=false` 可以禁用 API 文档。默认开启。

### 1.4 全局 OpenAPI 配置

`createApi` 方法创建全局的 `OpenAPI` 对象，配置了：

1. **API 信息**：标题、描述、版本、作者、许可证
2. **安全方案**：通过 `Authorization` 请求头传递 Token
3. **全局安全要求**：所有接口默认需要 `Authorization` 头

### 1.5 分组配置

`buildGroupedOpenApi` 方法为每个模块创建独立的 API 分组。

```java
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
```

业务模块可以这样使用：

```java
// 在某个业务模块的配置类中
@Bean
public GroupedOpenApi productGroupedOpenApi() {
    return YudaoSwaggerAutoConfiguration.buildGroupedOpenApi("product");
}
```

这样在 Swagger UI 中，就会出现一个 "product" 分组，只显示 `/admin-api/product/**` 和 `/app-api/product/**` 的接口。

### 1.6 全局参数注入

每个分组都会自动添加两个全局参数：

1. **`tenant-id`**：租户编号（默认值 1）
2. **`Authorization`**：认证 Token（默认值 `Bearer test1`）

这样在 Swagger UI 中调试接口时，不需要手动添加这两个参数。

### 1.7 OperationId 自定义

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

默认的 OperationId 可能在不同模块中重复（比如两个模块都有 `list` 方法）。通过组合 `类名前缀_方法名`（如 `ProductSpu_list`），确保全局唯一。

### 1.8 配置属性

`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/swagger/config/SwaggerProperties.java`:

```java
@ConfigurationProperties("yudao.swagger")
@Data
public class SwaggerProperties {
    @NotEmpty(message = "标题不能为空")
    private String title;
    @NotEmpty(message = "描述不能为空")
    private String description;
    @NotEmpty(message = "作者不能为空")
    private String author;
    @NotEmpty(message = "版本不能为空")
    private String version;
    @NotEmpty(message = "扫描的 package 不能为空")
    private String url;
    @NotEmpty(message = "扫描的 email 不能为空")
    private String email;
    @NotEmpty(message = "扫描的 license 不能为空")
    private String license;
    @NotEmpty(message = "扫描的 license-url 不能为空")
    private String licenseUrl;
}
```

配置示例：
```yaml
yudao:
  swagger:
    title: 电商后台管理系统
    description: 提供商品、订单、库存等管理接口
    author: 芋道源码
    version: 1.0.0
    url: https://github.com/YunaiV/ruoyi-vue-pro
    email: 123456789@qq.com
    license: MIT
    license-url: https://opensource.org/licenses/MIT
```

## 2. API 访问日志

### 2.1 架构设计

访问日志分为两层：

1. **Filter 层**（`ApiAccessLogFilter`）：记录请求的开始时间、参数、响应结果等
2. **Interceptor 层**（`ApiAccessLogInterceptor`）：在非生产环境打印 request/response 日志到控制台

```
请求到达
  │
  ▼
ApiAccessLogFilter（order = -103）
  │
  ├─ 记录开始时间
  ├─ 提前读取查询参数和请求体
  │
  ▼
Spring MVC DispatcherServlet
  │
  ▼
ApiAccessLogInterceptor.preHandle()
  │
  ├─ 记录 HandlerMethod 到 Request Attribute
  ├─ 非生产环境：打印请求日志 + 开始计时
  │
  ▼
Controller 执行
  │
  ▼
ApiAccessLogInterceptor.afterCompletion()
  │
  ├─ 非生产环境：打印耗时日志
  │
  ▼
回到 ApiAccessLogFilter
  │
  ├─ 记录结束时间
  ├─ 构建访问日志 DTO
  └─ 异步写入数据库
```

### 2.2 ApiAccessLogFilter

`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/apilog/core/filter/ApiAccessLogFilter.java`:

```java
package cn.iocoder.yudao.framework.apilog.core.filter;

import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.date.LocalDateTimeUtil;
import cn.hutool.core.exceptions.ExceptionUtil;
import cn.hutool.core.map.MapUtil;
import cn.hutool.core.util.ArrayUtil;
import cn.hutool.core.util.BooleanUtil;
import cn.hutool.core.util.StrUtil;
import cn.iocoder.yudao.framework.apilog.core.annotation.ApiAccessLog;
import cn.iocoder.yudao.framework.apilog.core.enums.OperateTypeEnum;
import cn.iocoder.yudao.framework.common.biz.infra.logger.ApiAccessLogCommonApi;
import cn.iocoder.yudao.framework.common.biz.infra.logger.dto.ApiAccessLogCreateReqDTO;
import cn.iocoder.yudao.framework.common.exception.enums.GlobalErrorCodeConstants;
import cn.iocoder.yudao.framework.common.pojo.CommonResult;
import cn.iocoder.yudao.framework.common.util.json.JsonUtils;
import cn.iocoder.yudao.framework.common.util.monitor.TracerUtils;
import cn.iocoder.yudao.framework.common.util.servlet.ServletUtils;
import cn.iocoder.yudao.framework.web.config.WebProperties;
import cn.iocoder.yudao.framework.web.core.filter.ApiRequestFilter;
import cn.iocoder.yudao.framework.web.core.util.WebFrameworkUtils;
import com.fasterxml.jackson.databind.JsonNode;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.method.HandlerMethod;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.Iterator;
import java.util.Map;

@Slf4j
public class ApiAccessLogFilter extends ApiRequestFilter {

    private static final String[] SANITIZE_KEYS =
            new String[]{"password", "token", "accessToken", "refreshToken"};

    private final String applicationName;
    private final ApiAccessLogCommonApi apiAccessLogApi;

    public ApiAccessLogFilter(WebProperties webProperties, String applicationName,
                               ApiAccessLogCommonApi apiAccessLogApi) {
        super(webProperties);
        this.applicationName = applicationName;
        this.apiAccessLogApi = apiAccessLogApi;
    }

    @Override
    @SuppressWarnings("NullableProblems")
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain)
            throws ServletException, IOException {
        // 获得开始时间
        LocalDateTime beginTime = LocalDateTime.now();
        // 提前获得参数，避免 XssFilter 过滤处理
        Map<String, String> queryString = ServletUtils.getParamMap(request);
        String requestBody = ServletUtils.getBody(request);

        try {
            // 继续过滤器
            filterChain.doFilter(request, response);
            // 正常执行，记录日志
            createApiAccessLog(request, beginTime, queryString, requestBody, null);
        } catch (Exception ex) {
            // 异常执行，记录日志
            createApiAccessLog(request, beginTime, queryString, requestBody, ex);
            throw ex;
        }
    }
```

### 2.3 关键设计细节

**提前读取参数**

```java
// 提前获得参数，避免 XssFilter 过滤处理
Map<String, String> queryString = ServletUtils.getParamMap(request);
String requestBody = ServletUtils.getBody(request);
```

这段代码在 `filterChain.doFilter()` 之前执行。为什么要提前读取？因为后续的 `XssFilter` 会修改参数内容（清理 XSS）。访问日志应该记录原始参数，而不是被清理后的参数。

**异常处理**

```java
try {
    filterChain.doFilter(request, response);
    createApiAccessLog(request, beginTime, queryString, requestBody, null);
} catch (Exception ex) {
    createApiAccessLog(request, beginTime, queryString, requestBody, ex);
    throw ex;
}
```

无论请求成功还是失败，都要记录日志。注意 `catch` 块中 `throw ex` 会继续抛出异常，让 `GlobalExceptionHandler` 处理。

### 2.4 日志构建

```java
private boolean buildApiAccessLog(ApiAccessLogCreateReqDTO accessLog,
                                   HttpServletRequest request,
                                   LocalDateTime beginTime,
                                   Map<String, String> queryString,
                                   String requestBody,
                                   Exception ex) {
    // 判断：是否要记录操作日志
    HandlerMethod handlerMethod = (HandlerMethod) request.getAttribute(ATTRIBUTE_HANDLER_METHOD);
    ApiAccessLog accessLogAnnotation = null;
    if (handlerMethod != null) {
        accessLogAnnotation = handlerMethod.getMethodAnnotation(ApiAccessLog.class);
        if (accessLogAnnotation != null && BooleanUtil.isFalse(accessLogAnnotation.enable())) {
            return false;
        }
    }

    // 处理用户信息
    accessLog.setUserId(WebFrameworkUtils.getLoginUserId(request))
            .setUserType(WebFrameworkUtils.getLoginUserType(request));
    // 设置访问结果
    CommonResult<?> result = WebFrameworkUtils.getCommonResult(request);
    if (result != null) {
        accessLog.setResultCode(result.getCode()).setResultMsg(result.getMsg());
    } else if (ex != null) {
        accessLog.setResultCode(GlobalErrorCodeConstants.INTERNAL_SERVER_ERROR.getCode())
                .setResultMsg(ExceptionUtil.getRootCauseMessage(ex));
    } else {
        accessLog.setResultCode(GlobalErrorCodeConstants.SUCCESS.getCode()).setResultMsg("");
    }
    // 设置请求字段
    accessLog.setTraceId(TracerUtils.getTraceId())
            .setApplicationName(applicationName)
            .setRequestUrl(request.getRequestURI())
            .setRequestMethod(request.getMethod())
            .setUserAgent(ServletUtils.getUserAgent(request))
            .setUserIp(ServletUtils.getClientIP(request));
    // 请求参数（脱敏）
    String[] sanitizeKeys = accessLogAnnotation != null ? accessLogAnnotation.sanitizeKeys() : null;
    Boolean requestEnable = accessLogAnnotation != null ? accessLogAnnotation.requestEnable() : Boolean.TRUE;
    if (!BooleanUtil.isFalse(requestEnable)) {
        Map<String, Object> requestParams = MapUtil.<String, Object>builder()
                .put("query", sanitizeMap(queryString, sanitizeKeys))
                .put("body", sanitizeJson(requestBody, sanitizeKeys)).build();
        accessLog.setRequestParams(JsonUtils.toJsonString(requestParams));
    }
    // 响应结果
    Boolean responseEnable = accessLogAnnotation != null ? accessLogAnnotation.responseEnable() : Boolean.FALSE;
    if (BooleanUtil.isTrue(responseEnable)) {
        accessLog.setResponseBody(sanitizeJson(result, sanitizeKeys));
    }
    // 持续时间
    accessLog.setBeginTime(beginTime).setEndTime(LocalDateTime.now())
            .setDuration((int) LocalDateTimeUtil.between(
                    accessLog.getBeginTime(), accessLog.getEndTime(), ChronoUnit.MILLIS));

    // 操作模块
    if (handlerMethod != null) {
        Tag tagAnnotation = handlerMethod.getBeanType().getAnnotation(Tag.class);
        Operation operationAnnotation = handlerMethod.getMethodAnnotation(Operation.class);
        String operateModule = accessLogAnnotation != null
                && StrUtil.isNotBlank(accessLogAnnotation.operateModule()) ?
                accessLogAnnotation.operateModule() :
                tagAnnotation != null ?
                        StrUtil.nullToDefault(tagAnnotation.name(), tagAnnotation.description()) : null;
        String operateName = accessLogAnnotation != null
                && StrUtil.isNotBlank(accessLogAnnotation.operateName()) ?
                accessLogAnnotation.operateName() :
                operationAnnotation != null ? operationAnnotation.summary() : null;
        OperateTypeEnum operateType = accessLogAnnotation != null
                && accessLogAnnotation.operateType().length > 0 ?
                accessLogAnnotation.operateType()[0] : parseOperateLogType(request);
        accessLog.setOperateModule(operateModule)
                .setOperateName(operateName)
                .setOperateType(operateType.getType());
    }
    return true;
}
```

### 2.5 数据脱敏

访问日志中包含请求参数和响应结果，其中可能有敏感信息（如密码、Token）。项目内置了脱敏机制。

**内置脱敏字段**：

```java
private static final String[] SANITIZE_KEYS =
        new String[]{"password", "token", "accessToken", "refreshToken"};
```

**自定义脱敏**：通过 `@ApiAccessLog` 注解的 `sanitizeKeys` 属性：

```java
@ApiAccessLog(sanitizeKeys = {"secretKey", "apiKey"})
@PostMapping("/create")
public CommonResult<Long> createProduct(@Valid @RequestBody ProductSaveReqVO reqVO) {
    // ...
}
```

脱敏逻辑使用 Jackson 的 `JsonNode` API 递归遍历 JSON 树，移除匹配的字段。

### 2.6 @ApiAccessLog 注解

`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/apilog/core/annotation/ApiAccessLog.java`:

```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface ApiAccessLog {

    // ========== 开关字段 ==========

    boolean enable() default true;
    boolean requestEnable() default true;
    boolean responseEnable() default false;
    String[] sanitizeKeys() default {};

    // ========== 模块字段 ==========

    String operateModule() default "";
    String operateName() default "";
    OperateTypeEnum[] operateType() default {};
}
```

使用示例：

```java
@ApiAccessLog(
    operateModule = "商品管理",
    operateName = "创建商品",
    operateType = OperateTypeEnum.CREATE,
    responseEnable = true
)
@PostMapping("/create")
public CommonResult<Long> createProduct(@Valid @RequestBody ProductSaveReqVO reqVO) {
    return CommonResult.success(productService.createProduct(reqVO));
}
```

### 2.7 ApiAccessLogInterceptor

`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/apilog/core/interceptor/ApiAccessLogInterceptor.java`:

```java
@Slf4j
public class ApiAccessLogInterceptor implements HandlerInterceptor {

    public static final String ATTRIBUTE_HANDLER_METHOD = "HANDLER_METHOD";

    private static final String ATTRIBUTE_STOP_WATCH = "ApiAccessLogInterceptor.StopWatch";

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response, Object handler) {
        // 记录 HandlerMethod，提供给 ApiAccessLogFilter 使用
        HandlerMethod handlerMethod = handler instanceof HandlerMethod
                ? (HandlerMethod) handler : null;
        if (handlerMethod != null) {
            request.setAttribute(ATTRIBUTE_HANDLER_METHOD, handlerMethod);
        }

        // 打印 request 日志
        if (!SpringUtils.isProd()) {
            Map<String, String> queryString = ServletUtils.getParamMap(request);
            String requestBody = ServletUtils.getBody(request);
            if (CollUtil.isEmpty(queryString) && StrUtil.isEmpty(requestBody)) {
                log.info("[preHandle][开始请求 URL({}) 无参数]",
                        request.getRequestURI());
            } else {
                log.info("[preHandle][开始请求 URL({}) 参数({})]",
                        request.getRequestURI(),
                        StrUtil.blankToDefault(requestBody, queryString.toString()));
            }
            // 计时
            StopWatch stopWatch = new StopWatch();
            stopWatch.start();
            request.setAttribute(ATTRIBUTE_STOP_WATCH, stopWatch);
            // 打印 Controller 路径
            printHandlerMethodPosition(handlerMethod);
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                 HttpServletResponse response,
                                 Object handler, Exception ex) {
        // 打印 response 日志
        if (!SpringUtils.isProd()) {
            StopWatch stopWatch = (StopWatch) request.getAttribute(ATTRIBUTE_STOP_WATCH);
            stopWatch.stop();
            log.info("[afterCompletion][完成请求 URL({}) 耗时({} ms)]",
                    request.getRequestURI(), stopWatch.getTotalTimeMillis());
        }
    }
}
```

Interceptor 的作用有两个：

1. **记录 HandlerMethod**：将 `HandlerMethod` 存入 Request Attribute，供 `ApiAccessLogFilter` 后续读取（用于获取 `@ApiAccessLog` 注解和 `@Tag`、`@Operation` 注解）
2. **非生产环境日志**：在开发和测试环境打印请求/响应日志到控制台，方便调试

### 2.8 自动配置注册

`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/apilog/config/YudaoApiLogAutoConfiguration.java`:

```java
@AutoConfiguration(after = YudaoWebAutoConfiguration.class)
public class YudaoApiLogAutoConfiguration implements WebMvcConfigurer {

    @Bean
    @ConditionalOnProperty(prefix = "yudao.access-log", value = "enable",
                            matchIfMissing = true)
    public FilterRegistrationBean<ApiAccessLogFilter> apiAccessLogFilter(
            WebProperties webProperties,
            @Value("${spring.application.name}") String applicationName,
            ApiAccessLogCommonApi apiAccessLogApi) {
        ApiAccessLogFilter filter = new ApiAccessLogFilter(
                webProperties, applicationName, apiAccessLogApi);
        return createFilterBean(filter, WebFilterOrderEnum.API_ACCESS_LOG_FILTER);
    }

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new ApiAccessLogInterceptor());
    }
}
```

通过 `yudao.access-log.enable=false` 可以禁用访问日志。默认开启。

## 3. API 加解密（可选功能）

### 3.1 功能说明

API 加解密是一个可选功能，通过 `yudao.api-encrypt.enable=true` 开启。它可以在请求/响应层面自动加解密数据，适用于对数据安全要求较高的场景。

### 3.2 自动配置

`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/encrypt/config/YudaoApiEncryptAutoConfiguration.java`:

```java
@AutoConfiguration
@Slf4j
@EnableConfigurationProperties(ApiEncryptProperties.class)
@ConditionalOnProperty(prefix = "yudao.api-encrypt", name = "enable", havingValue = "true")
public class YudaoApiEncryptAutoConfiguration {

    @Bean
    public FilterRegistrationBean<ApiEncryptFilter> apiEncryptFilter(
            WebProperties webProperties,
            ApiEncryptProperties apiEncryptProperties,
            RequestMappingHandlerMapping requestMappingHandlerMapping,
            GlobalExceptionHandler globalExceptionHandler) {
        ApiEncryptFilter filter = new ApiEncryptFilter(webProperties,
                apiEncryptProperties, requestMappingHandlerMapping,
                globalExceptionHandler);
        return createFilterBean(filter, WebFilterOrderEnum.API_ENCRYPT_FILTER);
    }
}
```

### 3.3 配置

`yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/encrypt/config/ApiEncryptProperties.java`:

```java
@ConfigurationProperties(prefix = "yudao.api-encrypt")
@Validated
@Data
public class ApiEncryptProperties {

    @NotNull(message = "是否开启不能为空")
    private Boolean enable;

    @NotEmpty(message = "请求头（响应头）名称不能为空")
    private String header = "X-Api-Encrypt";

    @NotEmpty(message = "对称加密算法不能为空")
    private String algorithm;

    @NotEmpty(message = "请求的解密密钥不能为空")
    private String requestKey;

    @NotEmpty(message = "响应的加密密钥不能为空")
    private String responseKey;
}
```

支持 AES（对称加密）和 RSA（非对称加密）两种算法。

## 4. 小结

本章实现了三个功能：

1. **Swagger/OpenAPI 3 配置**：全局 API 信息、安全方案、分组、自动注入 tenant-id 和 Authorization 参数
2. **API 访问日志**：Filter + Interceptor 两层架构，异步写入数据库，支持脱敏和 `@ApiAccessLog` 注解控制
3. **API 加解密**（可选）：支持 AES/RSA 算法，通过 `@ApiEncrypt` 注解标记需要加解密的接口

---

**导航**

- 上一章：[04-过滤器与安全防护实现](04-过滤器与安全防护实现.md)
- 下一章：[06-完整请求处理流程](06-完整请求处理流程.md)
- 相关设计文档：[05-API文档与访问日志设计](../功能设计/05-API文档与访问日志设计.md)
