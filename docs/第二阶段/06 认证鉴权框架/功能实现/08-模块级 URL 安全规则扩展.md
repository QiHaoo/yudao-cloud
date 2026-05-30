# 08-模块级 URL 安全规则扩展

## 目标

实现 `AuthorizeRequestsCustomizer` 抽象类，让每个 Maven 模块能自定义 URL 安全规则。

## AuthorizeRequestsCustomizer 实现

```java
// 文件路径：config/AuthorizeRequestsCustomizer.java
public abstract class AuthorizeRequestsCustomizer
        implements Customizer<AuthorizeHttpRequestsConfigurer<HttpSecurity>
            .AuthorizationManagerRequestMatcherRegistry>, Ordered {

    @Resource
    private WebProperties webProperties;

    protected String buildAdminApi(String url) {
        return webProperties.getAdminApi().getPrefix() + url;
    }

    protected String buildAppApi(String url) {
        return webProperties.getAppApi().getPrefix() + url;
    }

    @Override
    public int getOrder() {
        return 0;
    }
}
```

`buildAdminApi("/user/**")` 会返回 `/admin-api/user/**`，自动拼接 URL 前缀。

## 使用示例：System 模块的自定义规则

```java
// yudao-module-system 中的实现
@Component
public class SecurityConfiguration extends AuthorizeRequestsCustomizer {

    @Override
    public void customize(AuthorizeHttpRequestsConfigurer<HttpSecurity>
            .AuthorizationManagerRequestMatcherRegistry registry) {
        // 放行 Swagger
        registry.requestMatchers(buildAdminApi("/v3/api-docs/**")).permitAll();
        // 放行 Druid 监控
        registry.requestMatchers("/druid/**").permitAll();
        // 放行 RPC 服务路径
        registry.requestMatchers(ApiConstants.PREFIX + "/**").permitAll();
    }
}
```

## 小结

`AuthorizeRequestsCustomizer` 提供了模块级的 URL 安全规则扩展点，让每个模块能独立声明自己的免认证规则，无需修改框架层代码。
