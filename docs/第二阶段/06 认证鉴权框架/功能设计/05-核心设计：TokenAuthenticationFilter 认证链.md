# 05-核心设计：TokenAuthenticationFilter 认证链

## 场景切入

一个请求到达服务器，第一个要回答的问题就是："这个请求是谁发的？"

想象一个电商后台：张三登录后想查看订单列表。前端在请求头里带上了 `Authorization: Bearer abc123xyz`。这个 Token 到底该怎么校验？校验通过后，怎么让后续的业务代码知道"当前用户是张三"？

`TokenAuthenticationFilter` 就是这个"门卫"——它拦截每一个请求，识别用户身份，然后把身份信息传递给后续所有环节。

## 设计目标

1. **统一入口**：所有请求的身份识别都经过同一个过滤器
2. **多策略识别**：支持从 Header 透传、Token 校验、Mock 模拟三种方式获取用户
3. **无侵入**：业务代码不需要关心认证细节，直接通过 `SecurityFrameworkUtils.getLoginUser()` 获取当前用户
4. **容错处理**：Token 无效时不中断请求链路（对于免认证接口），而是让后续的权限检查决定是否放行

## 在过滤器链中的位置

```
请求进入
  │
  ▼
┌─────────────────────────────────────────┐
│  Spring Security 过滤器链                │
│                                         │
│  ① SecurityContextPersistenceFilter     │
│  ② ...                                 │
│  ③ TokenAuthenticationFilter  ← 我们在这里（在 UsernamePasswordAuthenticationFilter 之前）
│  ④ UsernamePasswordAuthenticationFilter │
│  ⑤ ...                                 │
│  ⑥ FilterSecurityInterceptor            │
│                                         │
└─────────────────────────────────────────┘
  │
  ▼
Controller / Service
```

通过 `httpSecurity.addFilterBefore(authenticationTokenFilter, UsernamePasswordAuthenticationFilter.class)` 配置，`TokenAuthenticationFilter` 被插入到 `UsernamePasswordAuthenticationFilter` 之前。这意味着：传统的用户名密码登录流程被完全跳过，我们用自研的 Token 认证替代。

## 认证策略的优先级

`TokenAuthenticationFilter.doFilterInternal()` 按以下优先级尝试识别用户：

```
请求到达
  │
  ▼
┌──────────────────────────────────────┐
│ 策略一：从 Header 透传解析            │
│ 检查 login-user Header 是否存在       │
│ （微服务间 RPC 场景）                  │
└──────────────┬───────────────────────┘
               │ 未找到
               ▼
┌──────────────────────────────────────┐
│ 策略二：从 Token 校验                  │
│ 从 Authorization Header 或 token      │
│ 参数提取 Token，调用 RPC 校验          │
│ （前端直连或 Nginx 转发场景）           │
└──────────────┬───────────────────────┘
               │ Token 校验失败
               ▼
┌──────────────────────────────────────┐
│ 策略三：Mock 模拟登录                  │
│ Token 以 mockSecret 开头？            │
│ 直接构建模拟用户                       │
│ （仅开发环境启用）                      │
└──────────────┬───────────────────────┘
               │ 未匹配
               ▼
         loginUser = null
         （后续由权限检查决定是否放行）
```

### 策略一：Header 透传解析

**适用场景**：微服务间通过 Feign 调用时，上游服务已经认证了用户，需要把用户信息传递给下游服务。

**工作原理**：
```java
private LoginUser buildLoginUserByHeader(HttpServletRequest request) {
    String loginUserStr = request.getHeader("login-user");  // 从 Header 获取
    if (StrUtil.isEmpty(loginUserStr)) {
        return null;
    }
    loginUserStr = URLDecoder.decode(loginUserStr, StandardCharsets.UTF_8.name()); // URL 解码
    LoginUser loginUser = JsonUtils.parseObject(loginUserStr, LoginUser.class);     // JSON 反序列化
    return loginUser;
}
```

这里有一个关键细节：`login-user` Header 的值是 URL 编码的 JSON 字符串。为什么要 URL 编码？因为 HTTP Header 中直接传 JSON 可能有编码问题（特别是包含中文时），URL 编码可以确保安全传输。

### 策略二：Token 校验

**适用场景**：前端直接请求服务（通过 Nginx 转发或直接访问），需要从请求中提取 Token 并校验。

**工作原理**：
```java
private LoginUser buildLoginUserByToken(String token, Integer userType) {
    // 1. 调用 system 服务校验 Token
    OAuth2AccessTokenCheckRespDTO accessToken = oauth2TokenApi.checkAccessToken(token).getCheckedData();
    if (accessToken == null) {
        return null;  // Token 无效
    }
    // 2. 校验用户类型是否匹配
    if (userType != null && ObjectUtil.notEqual(accessToken.getUserType(), userType)) {
        throw new AccessDeniedException("错误的用户类型");
    }
    // 3. 构建 LoginUser
    return new LoginUser().setId(accessToken.getUserId())
            .setUserType(accessToken.getUserType())
            .setInfo(accessToken.getUserInfo())
            .setTenantId(accessToken.getTenantId())
            .setScopes(accessToken.getScopes())
            .setExpiresTime(accessToken.getExpiresTime());
}
```

注意这里有一个**用户类型校验**的细节：`/admin-api/*` 的请求只能由管理员类型用户访问，`/app-api/*` 只能由会员类型用户访问。这个类型信息是从 URL 前缀中提取的（`WebFrameworkUtils.getLoginUserType(request)`）。

**为什么 Token 校验失败不直接抛异常？** 因为有些接口是免认证的（标记了 `@PermitAll`）。如果 Token 无效就直接报错，那未登录用户就无法访问这些接口了。所以这里返回 `null`，让后续的权限检查来决定是否放行。

但有一个例外：如果 Token 校验过程中抛出了业务异常（`ServiceException`，比如 Token 已过期），也会返回 `null` 而不是中断请求。这是因为过期 Token 访问免认证接口是合理的场景。

### 策略三：Mock 模拟登录

**适用场景**：开发环境下，不想每次都走完整的登录流程，直接用"密钥 + 用户ID"模拟登录。

**工作原理**：
```java
private LoginUser mockLoginUser(HttpServletRequest request, String token, Integer userType) {
    if (!securityProperties.getMockEnable()) {
        return null;  // 未启用 Mock
    }
    if (!token.startsWith(securityProperties.getMockSecret())) {
        return null;  // Token 不以 mockSecret 开头
    }
    Long userId = Long.valueOf(token.substring(securityProperties.getMockSecret().length()));
    return new LoginUser().setId(userId).setUserType(userType)
            .setTenantId(WebFrameworkUtils.getTenantId(request));
}
```

使用方式：配置 `yudao.security.mock-enable=true`，然后请求时带 `Authorization: test1`（假设 `mockSecret` 为 `test`），即可模拟用户 ID 为 1 的用户登录。

**安全警告**：线上环境**必须关闭**此功能（`mockEnable=false`），否则任何人都能模拟任意用户！

## 认证成功后：设置用户上下文

无论通过哪种策略识别到用户，最终都会调用：

```java
if (loginUser != null) {
    SecurityFrameworkUtils.setLoginUser(loginUser, request);
}
```

`setLoginUser` 做了两件事：

1. **放入 SecurityContextHolder**：创建 `UsernamePasswordAuthenticationToken`，设置到 Spring Security 上下文
2. **放入 Request Attribute**：设置 `loginUserId` 和 `loginUserType` 到 Request 中，供 `ApiAccessLogFilter` 等组件使用

```java
public static void setLoginUser(LoginUser loginUser, HttpServletRequest request) {
    // 1. 放入 SecurityContextHolder
    Authentication authentication = buildAuthentication(loginUser, request);
    SecurityContextHolder.getContext().setAuthentication(authentication);

    // 2. 放入 Request Attribute
    if (request != null) {
        WebFrameworkUtils.setLoginUserId(request, loginUser.getId());
        WebFrameworkUtils.setLoginUserType(request, loginUser.getUserType());
    }
}
```

为什么要同时放入两个地方？因为 Spring Security 的 Filter 在 `ApiAccessLogFilter` 之后执行，当访问日志过滤器需要记录用户信息时，SecurityContext 中还没有用户。所以额外放入 Request Attribute，确保访问日志能获取到用户 ID。

## 异常处理

认证过程中如果发生异常（比如 Token 校验时 RPC 调用失败），`TokenAuthenticationFilter` 不会让异常直接抛出，而是交给 `GlobalExceptionHandler` 处理：

```java
try {
    loginUser = buildLoginUserByToken(token, userType);
    if (loginUser == null) {
        loginUser = mockLoginUser(request, token, userType);
    }
} catch (Throwable ex) {
    CommonResult<?> result = globalExceptionHandler.allExceptionHandler(request, ex);
    ServletUtils.writeJSON(response, result);
    return;  // 直接返回错误响应，不继续过滤链
}
```

这是一个重要的设计选择：认证异常（如 Token 无效）应该返回明确的错误信息（如 401），而不是让请求继续走到业务层再报错。

## 完整流程图

```
HTTP 请求进入
  │
  ▼
TokenAuthenticationFilter.doFilterInternal()
  │
  ├── 1. buildLoginUserByHeader(request)
  │     ├── 有 login-user Header？
  │     │   ├── 是 → JSON 反序列化 → 校验用户类型 → 返回 LoginUser
  │     │   └── 否 → 返回 null
  │     └── 异常 → 记录日志 → 抛出异常
  │
  ├── 如果 loginUser == null：
  │   ├── 2. SecurityFrameworkUtils.obtainAuthorization()
  │   │     ├── 从 Authorization Header 提取（支持 Bearer 前缀）
  │   │     └── 或从 token 参数提取
  │   │
  │   ├── 如果 token 非空：
  │   │   ├── 2a. buildLoginUserByToken(token, userType)
  │   │   │     ├── RPC 调用 checkAccessToken()
  │   │   │     ├── 校验用户类型
  │   │   │     ├── 构建 LoginUser → 返回
  │   │   │     └── ServiceException → 返回 null（Token 无效，可能是免认证接口）
  │   │   │
  │   │   └── 如果 loginUser == null：
  │   │       └── 2b. mockLoginUser(request, token, userType)
  │   │             ├── mockEnable == false → 返回 null
  │   │             ├── token 不以 mockSecret 开头 → 返回 null
  │   │             └── 构建模拟用户 → 返回 LoginUser
  │   │
  │   └── 如果过程中抛异常：
  │       └── GlobalExceptionHandler 处理 → 写入 JSON 响应 → return
  │
  ├── 如果 loginUser != null：
  │   └── SecurityFrameworkUtils.setLoginUser(loginUser, request)
  │         ├── SecurityContextHolder.getContext().setAuthentication(...)
  │         └── Request.setAttribute(loginUserId, loginUserType)
  │
  └── chain.doFilter(request, response)  // 继续过滤链
```

## 与 GlobalExceptionHandler 的协作

这里有一个容易被忽略的细节：`TokenAuthenticationFilter` 注入了 `GlobalExceptionHandler`。

为什么认证过滤器需要全局异常处理器？因为当认证失败时，需要返回与业务接口一致的错误格式（`CommonResult`），而不是 Spring Security 默认的 HTML 错误页面。

```java
// 认证失败时的处理
catch (Throwable ex) {
    CommonResult<?> result = globalExceptionHandler.allExceptionHandler(request, ex);
    ServletUtils.writeJSON(response, result);
    return;
}
```

这样，前端收到的错误响应格式始终是：
```json
{
    "code": 401,
    "msg": "未登录",
    "data": null
}
```

而不是 Spring Security 默认的 HTML 403 页面。

## 思考题

1. **为什么 TokenAuthenticationFilter 继承 OncePerRequestFilter 而不是直接实现 Filter？** 提示：考虑异步转发（forward）场景下过滤器会被执行几次。

2. **如果同时存在 login-user Header 和 Authorization Header，会发生什么？** 这种情况在什么场景下可能出现？是 Bug 还是预期行为？

3. **Mock 登录功能为什么要求 Token 以 mockSecret 开头，而不是用一个独立的 Header 或参数？** 提示：考虑安全性——如果用独立参数，任何人都能猜到；用密钥前缀，相当于一个"暗号"。

4. **Token 校验失败时返回 null 而不是抛异常，这会不会导致某些应该报错的场景被静默忽略？** 比如一个需要认证的接口，用户带了一个过期的 Token，会发生什么？
