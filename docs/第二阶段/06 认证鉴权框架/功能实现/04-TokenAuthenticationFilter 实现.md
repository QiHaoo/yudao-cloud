# 04-TokenAuthenticationFilter 实现

## 目标

实现认证过滤器 `TokenAuthenticationFilter`，它是整个认证链的核心——拦截每个请求，识别用户身份。

## 实现

```java
// 文件路径：core/filter/TokenAuthenticationFilter.java
@RequiredArgsConstructor
@Slf4j
public class TokenAuthenticationFilter extends OncePerRequestFilter {

    private final SecurityProperties securityProperties;
    private final GlobalExceptionHandler globalExceptionHandler;
    private final OAuth2TokenCommonApi oauth2TokenApi;

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        // 策略一：从 Header 透传解析
        LoginUser loginUser = buildLoginUserByHeader(request);

        // 策略二：从 Token 校验
        if (loginUser == null) {
            String token = SecurityFrameworkUtils.obtainAuthorization(request,
                    securityProperties.getTokenHeader(), securityProperties.getTokenParameter());
            if (StrUtil.isNotEmpty(token)) {
                Integer userType = WebFrameworkUtils.getLoginUserType(request);
                try {
                    loginUser = buildLoginUserByToken(token, userType);
                    // 策略三：Mock 模拟
                    if (loginUser == null) {
                        loginUser = mockLoginUser(request, token, userType);
                    }
                } catch (Throwable ex) {
                    // 认证异常，返回错误响应
                    CommonResult<?> result = globalExceptionHandler.allExceptionHandler(request, ex);
                    ServletUtils.writeJSON(response, result);
                    return;
                }
            }
        }

        // 设置用户上下文
        if (loginUser != null) {
            SecurityFrameworkUtils.setLoginUser(loginUser, request);
        }
        chain.doFilter(request, response);
    }
}
```

### 策略一：Header 透传解析

```java
private LoginUser buildLoginUserByHeader(HttpServletRequest request) {
    String loginUserStr = request.getHeader(SecurityFrameworkUtils.LOGIN_USER_HEADER);
    if (StrUtil.isEmpty(loginUserStr)) {
        return null;
    }
    loginUserStr = URLDecoder.decode(loginUserStr, StandardCharsets.UTF_8.name());
    LoginUser loginUser = JsonUtils.parseObject(loginUserStr, LoginUser.class);
    // 校验用户类型
    Integer userType = WebFrameworkUtils.getLoginUserType(request);
    if (userType != null && loginUser != null
            && ObjectUtil.notEqual(loginUser.getUserType(), userType)) {
        throw new AccessDeniedException("错误的用户类型");
    }
    return loginUser;
}
```

### 策略二：Token 校验

```java
private LoginUser buildLoginUserByToken(String token, Integer userType) {
    try {
        OAuth2AccessTokenCheckRespDTO accessToken =
            oauth2TokenApi.checkAccessToken(token).getCheckedData();
        if (accessToken == null) {
            return null;
        }
        // 校验用户类型
        if (userType != null && ObjectUtil.notEqual(accessToken.getUserType(), userType)) {
            throw new AccessDeniedException("错误的用户类型");
        }
        return new LoginUser().setId(accessToken.getUserId())
                .setUserType(accessToken.getUserType())
                .setInfo(accessToken.getUserInfo())
                .setTenantId(accessToken.getTenantId())
                .setScopes(accessToken.getScopes())
                .setExpiresTime(accessToken.getExpiresTime());
    } catch (ServiceException serviceException) {
        return null; // Token 无效，返回 null（可能是免认证接口）
    }
}
```

### 策略三：Mock 模拟

```java
private LoginUser mockLoginUser(HttpServletRequest request, String token, Integer userType) {
    if (!securityProperties.getMockEnable()) {
        return null;
    }
    if (!token.startsWith(securityProperties.getMockSecret())) {
        return null;
    }
    Long userId = Long.valueOf(token.substring(securityProperties.getMockSecret().length()));
    return new LoginUser().setId(userId).setUserType(userType)
            .setTenantId(WebFrameworkUtils.getTenantId(request));
}
```

## 注册到自动配置

```java
// 在 YudaoSecurityAutoConfiguration 中
@Bean
public TokenAuthenticationFilter authenticationTokenFilter(
        GlobalExceptionHandler globalExceptionHandler,
        OAuth2TokenCommonApi oauth2TokenApi) {
    return new TokenAuthenticationFilter(securityProperties, globalExceptionHandler, oauth2TokenApi);
}
```

## 小结

`TokenAuthenticationFilter` 实现了三级认证策略：
1. **Header 透传**：微服务间 RPC 场景
2. **Token 校验**：前端直连场景
3. **Mock 模拟**：开发调试场景

认证成功后，通过 `SecurityFrameworkUtils.setLoginUser()` 设置用户上下文，后续代码即可获取当前用户。
