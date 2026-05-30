# 06-SecurityFilterChain 配置与 @PreAuthorize 启用

## 目标

配置 Spring Security 的 `SecurityFilterChain`：禁用 CSRF/Session、启用 @PreAuthorize、注册 Token 过滤器、配置免认证 URL 规则。

## YudaoWebSecurityConfigurerAdapter 实现

```java
// 文件路径：config/YudaoWebSecurityConfigurerAdapter.java
@AutoConfiguration
@AutoConfigureOrder(-1)
@EnableMethodSecurity(securedEnabled = true)  // 启用 @PreAuthorize
public class YudaoWebSecurityConfigurerAdapter {

    @Resource private WebProperties webProperties;
    @Resource private SecurityProperties securityProperties;
    @Resource private AuthenticationEntryPoint authenticationEntryPoint;
    @Resource private AccessDeniedHandler accessDeniedHandler;
    @Resource private TokenAuthenticationFilter authenticationTokenFilter;
    @Resource private List<AuthorizeRequestsCustomizer> authorizeRequestsCustomizers;
    @Resource private ApplicationContext applicationContext;

    @Bean
    public AuthenticationManager authenticationManagerBean(
            AuthenticationConfiguration authenticationConfiguration) throws Exception {
        return authenticationConfiguration.getAuthenticationManager();
    }

    @Bean
    protected SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
        // 1. 基础配置
        httpSecurity
            .cors(Customizer.withDefaults())           // 开启跨域
            .csrf(AbstractHttpConfigurer::disable)      // 禁用 CSRF（无状态 Token）
            .sessionManagement(c -> c
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))  // 禁用 Session
            .headers(c -> c.frameOptions(
                HeadersConfigurer.FrameOptionsConfig::disable))  // 允许 iframe
            .exceptionHandling(c -> c
                .authenticationEntryPoint(authenticationEntryPoint)
                .accessDeniedHandler(accessDeniedHandler));

        // 2. 免认证 URL 规则
        Multimap<HttpMethod, String> permitAllUrls = getPermitAllUrlsFromAnnotations();
        httpSecurity
            .authorizeHttpRequests(c -> c
                // 静态资源
                .requestMatchers(HttpMethod.GET, "/*.html", "/*.css", "/*.js").permitAll()
                // @PermitAll 注解
                .requestMatchers(HttpMethod.GET, permitAllUrls.get(HttpMethod.GET).toArray(new String[0])).permitAll()
                .requestMatchers(HttpMethod.POST, permitAllUrls.get(HttpMethod.POST).toArray(new String[0])).permitAll()
                // ... 其他 HTTP 方法
                // 配置文件中的免认证 URL
                .requestMatchers(securityProperties.getPermitAllUrls().toArray(new String[0])).permitAll()
            )
            // 模块自定义规则
            .authorizeHttpRequests(c -> authorizeRequestsCustomizers
                .forEach(customizer -> customizer.customize(c)))
            // 兜底：必须认证
            .authorizeHttpRequests(c -> c
                .dispatcherTypeMatchers(DispatcherType.ASYNC).permitAll()
                .anyRequest().authenticated());

        // 3. 注册 Token 过滤器
        httpSecurity.addFilterBefore(authenticationTokenFilter,
            UsernamePasswordAuthenticationFilter.class);

        return httpSecurity.build();
    }
}
```

## @EnableMethodSecurity 的作用

```java
@EnableMethodSecurity(securedEnabled = true)
```

这行代码启用了 Spring Security 的方法级安全：
- `@PreAuthorize`：方法执行前检查权限（默认启用）
- `@PostAuthorize`：方法执行后检查权限（默认启用）
- `@Secured`：简单角色检查（需要 `securedEnabled = true`）

启用后，Spring AOP 会拦截带这些注解的方法，在方法执行前/后进行权限校验。

## @PermitAll 自动扫描

`getPermitAllUrlsFromAnnotations()` 在启动时扫描所有 Controller，收集带 `@PermitAll` 的 URL：

```java
private Multimap<HttpMethod, String> getPermitAllUrlsFromAnnotations() {
    Multimap<HttpMethod, String> result = HashMultimap.create();
    RequestMappingHandlerMapping handlerMapping =
        (RequestMappingHandlerMapping) applicationContext.getBean("requestMappingHandlerMapping");

    for (Map.Entry<RequestMappingInfo, HandlerMethod> entry
            : handlerMapping.getHandlerMethods().entrySet()) {
        HandlerMethod handlerMethod = entry.getValue();
        if (!handlerMethod.hasMethodAnnotation(PermitAll.class)
                && !handlerMethod.getBeanType().isAnnotationPresent(PermitAll.class)) {
            continue;
        }
        // 收集 URL 和 HTTP 方法...
    }
    return result;
}
```

## 小结

本章配置了 SecurityFilterChain 的核心行为：
- **无状态**：禁用 CSRF 和 Session
- **方法级鉴权**：`@EnableMethodSecurity` 启用 `@PreAuthorize`
- **三层免认证规则**：@PermitAll → 配置文件 → 模块自定义
- **兜底规则**：所有未匹配的请求必须认证
