# 02 - TokenAuthenticationFilter 实现

> 本章实现网关的核心过滤器——TokenAuthenticationFilter。完成本章后，网关将具备 Token 校验、用户信息注入、租户透传的能力。

## 设计回顾

在功能设计中，我们确定了 TokenAuthenticationFilter 的职责：

1. 从请求头提取 Token
2. 调用 system-server 校验 Token 有效性
3. 获取用户信息（userId、userType、tenantId）
4. 将用户信息注入到请求头 `login-user`
5. 转发到下游服务

```
请求到达 Gateway
    │
    ├── 移除伪造的 login-user 请求头
    │
    ├── 提取 Authorization: Bearer xxx
    │   ├── Token 为空 → 直接放行
    │   └── Token 非空 → 继续校验
    │
    ├── 从缓存获取用户信息
    │   ├── 缓存命中 → 直接使用
    │   └── 缓存未命中 → 调用 system-server 校验
    │
    ├── 校验结果处理
    │   ├── 有效 → 注入用户信息到请求头，转发
    │   └── 无效 → 放行（交给服务端判断是否需要登录）
    │
    └── 转发到下游服务
```

## 第一步：LoginUser 数据模型

首先定义网关侧的 LoginUser 类，用于承载用户信息：

```java
// yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/filter/security/LoginUser.java
package cn.iocoder.yudao.gateway.filter.security;

import lombok.Data;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Map;

/**
 * 登录用户信息
 *
 * copy from yudao-spring-boot-starter-security 的 LoginUser 类
 */
@Data
public class LoginUser {

    /**
     * 用户编号
     */
    private Long id;
    /**
     * 用户类型
     */
    private Integer userType;
    /**
     * 额外的用户信息
     */
    private Map<String, String> info;
    /**
     * 租户编号
     */
    private Long tenantId;
    /**
     * 授权范围
     */
    private List<String> scopes;
    /**
     * 过期时间
     */
    private LocalDateTime expiresTime;

}
```

**为什么网关要单独定义 LoginUser，而不是复用 framework 中的类？**

Gateway 基于 WebFlux，而 `yudao-spring-boot-starter-security` 中的 LoginUser 所在模块依赖了 WebMvc（Servlet）。如果直接复用，会导致 WebFlux 和 WebMvc 冲突。因此，Gateway 中的 LoginUser 是一个简化版本，只包含网关需要的字段。

## 第二步：SecurityFrameworkUtils 工具类

这个工具类封装了与安全相关的操作：提取 Token、设置用户信息、移除伪造请求头。

```java
// yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/util/SecurityFrameworkUtils.java
package cn.iocoder.yudao.gateway.util;

import cn.hutool.core.map.MapUtil;
import cn.iocoder.yudao.framework.common.util.json.JsonUtils;
import cn.iocoder.yudao.gateway.filter.security.LoginUser;
import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.util.StringUtils;
import org.springframework.web.server.ServerWebExchange;

import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;

/**
 * 安全服务工具类
 *
 * copy from yudao-spring-boot-starter-security 的 SecurityFrameworkUtils 类
 */
@Slf4j
public class SecurityFrameworkUtils {

    private static final String AUTHORIZATION_HEADER = "Authorization";
    private static final String AUTHORIZATION_BEARER = "Bearer";
    private static final String LOGIN_USER_HEADER = "login-user";
    private static final String LOGIN_USER_ID_ATTR = "login-user-id";
    private static final String LOGIN_USER_TYPE_ATTR = "login-user-type";

    private SecurityFrameworkUtils() {}

    /**
     * 从请求中提取认证 Token
     *
     * Token 格式：Bearer {token}
     * 示例：Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
     */
    public static String obtainAuthorization(ServerWebExchange exchange) {
        String authorization = exchange.getRequest().getHeaders().getFirst(AUTHORIZATION_HEADER);
        if (!StringUtils.hasText(authorization)) {
            return null;
        }
        int index = authorization.indexOf(AUTHORIZATION_BEARER + " ");
        if (index == -1) {
            return null;
        }
        return authorization.substring(index + 7).trim();
    }

    /**
     * 设置登录用户到 exchange 属性
     *
     * 使用 exchange.getAttributes() 存储，同一请求链路中都可以访问
     */
    public static void setLoginUser(ServerWebExchange exchange, LoginUser user) {
        exchange.getAttributes().put(LOGIN_USER_ID_ATTR, user.getId());
        exchange.getAttributes().put(LOGIN_USER_TYPE_ATTR, user.getUserType());
    }

    /**
     * 移除请求头中的 login-user
     *
     * 防止前端伪造 login-user 请求头，模拟其他用户身份
     */
    public static ServerWebExchange removeLoginUser(ServerWebExchange exchange) {
        if (!exchange.getRequest().getHeaders().containsKey(LOGIN_USER_HEADER)) {
            return exchange;
        }
        ServerHttpRequest request = exchange.getRequest().mutate()
                .headers(httpHeaders -> httpHeaders.remove(LOGIN_USER_HEADER)).build();
        return exchange.mutate().request(request).build();
    }

    /**
     * 获取登录用户编号
     */
    public static Long getLoginUserId(ServerWebExchange exchange) {
        return MapUtil.getLong(exchange.getAttributes(), LOGIN_USER_ID_ATTR);
    }

    /**
     * 获取登录用户类型
     */
    public static Integer getLoginUserType(ServerWebExchange exchange) {
        return MapUtil.getInt(exchange.getAttributes(), LOGIN_USER_TYPE_ATTR);
    }

    /**
     * 将用户信息设置到 login-user 请求头
     *
     * 使用 JSON 序列化，URL 编码避免中文乱码
     */
    @SneakyThrows
    public static void setLoginUserHeader(ServerHttpRequest.Builder builder, LoginUser user) {
        try {
            String userStr = JsonUtils.toJsonString(user);
            userStr = URLEncoder.encode(userStr, StandardCharsets.UTF_8.name());
            builder.header(LOGIN_USER_HEADER, userStr);
        } catch (Exception ex) {
            log.error("[setLoginUserHeader][序列化 user({}) 发生异常]", user, ex);
            throw ex;
        }
    }

}
```

**关键设计决策：**

1. **`removeLoginUser` 为什么在最前面执行？**

   前端可能伪造 `login-user` 请求头，冒充其他用户。网关必须在处理请求之前先移除这个头，然后由网关自己设置正确的用户信息。

2. **`setLoginUserHeader` 为什么要 URL 编码？**

   用户信息中可能包含中文（如昵称），直接放在 HTTP Header 中会导致乱码。URL 编码后可以安全传输。

3. **`exchange.getAttributes()` 是什么？**

   这是 WebFlux 的 `ServerWebExchange` 提供的请求级属性存储。在同一请求的过滤器链中，所有过滤器都可以读写这些属性。它不是 HTTP Header，不会被转发到下游服务。

## 第三步：WebFrameworkUtils 工具类

这个工具类封装了 Web 相关的操作：获取租户 ID、获取客户端 IP、写 JSON 响应。

```java
// yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/util/WebFrameworkUtils.java
package cn.iocoder.yudao.gateway.util;

import cn.hutool.core.net.NetUtil;
import cn.hutool.core.util.ArrayUtil;
import cn.hutool.core.util.NumberUtil;
import cn.iocoder.yudao.framework.common.util.json.JsonUtils;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.route.Route;
import org.springframework.cloud.gateway.support.ServerWebExchangeUtils;
import org.springframework.core.io.buffer.DataBufferFactory;
import org.springframework.http.HttpHeaders;
import org.springframework.http.MediaType;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.http.server.reactive.ServerHttpResponse;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

/**
 * Web 工具类
 *
 * copy from yudao-spring-boot-starter-web 的 WebFrameworkUtils 类
 */
@Slf4j
public class WebFrameworkUtils {

    private static final String HEADER_TENANT_ID = "tenant-id";

    private WebFrameworkUtils() {}

    /**
     * 设置租户 ID 到 WebClient 请求头
     */
    public static void setTenantIdHeader(Long tenantId, HttpHeaders httpHeaders) {
        if (tenantId == null) {
            return;
        }
        httpHeaders.set(HEADER_TENANT_ID, String.valueOf(tenantId));
    }

    /**
     * 从请求头获取租户 ID
     */
    public static Long getTenantId(ServerWebExchange exchange) {
        String tenantId = exchange.getRequest().getHeaders().getFirst(HEADER_TENANT_ID);
        return NumberUtil.isNumber(tenantId) ? Long.valueOf(tenantId) : null;
    }

    /**
     * 写 JSON 响应
     */
    @SuppressWarnings("deprecation")
    public static Mono<Void> writeJSON(ServerWebExchange exchange, Object object) {
        ServerHttpResponse response = exchange.getResponse();
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON_UTF8);
        return response.writeWith(Mono.fromSupplier(() -> {
            DataBufferFactory bufferFactory = response.bufferFactory();
            try {
                return bufferFactory.wrap(JsonUtils.toJsonByte(object));
            } catch (Exception ex) {
                ServerHttpRequest request = exchange.getRequest();
                log.error("[writeJSON][uri({}/{}) 发生异常]", request.getURI(), request.getMethod(), ex);
                return bufferFactory.wrap(new byte[0]);
            }
        }));
    }

    /**
     * 获取客户端 IP
     */
    public static String getClientIP(ServerWebExchange exchange, String... otherHeaderNames) {
        String[] headers = { "X-Forwarded-For", "X-Real-IP", "Proxy-Client-IP",
                "WL-Proxy-Client-IP", "HTTP_CLIENT_IP", "HTTP_X_FORWARDED_FOR" };
        if (ArrayUtil.isNotEmpty(otherHeaderNames)) {
            headers = ArrayUtil.addAll(headers, otherHeaderNames);
        }
        String ip;
        for (String header : headers) {
            ip = exchange.getRequest().getHeaders().getFirst(header);
            if (!NetUtil.isUnknown(ip)) {
                return NetUtil.getMultistageReverseProxyIp(ip);
            }
        }
        if (exchange.getRequest().getRemoteAddress() == null) {
            return null;
        }
        ip = exchange.getRequest().getRemoteAddress().getHostString();
        return NetUtil.getMultistageReverseProxyIp(ip);
    }

    /**
     * 获取请求匹配的路由
     */
    public static Route getGatewayRoute(ServerWebExchange exchange) {
        return exchange.getAttribute(ServerWebExchangeUtils.GATEWAY_ROUTE_ATTR);
    }

}
```

**`getTenantId` 的数据来源：**

租户 ID 从请求头 `tenant-id` 中获取。前端在请求时携带此头，网关将其透传到下游服务。如果用户已登录，Token 校验结果中的 tenantId 优先级更高。

## 第四步：TokenAuthenticationFilter 核心实现

这是网关最核心的过滤器，负责 Token 校验和用户信息注入。

```java
// yudao-gateway/src/main/java/cn/iocoder/yudao/gateway/filter/security/TokenAuthenticationFilter.java
package cn.iocoder.yudao.gateway.filter.security;

import cn.hutool.core.util.StrUtil;
import cn.iocoder.yudao.framework.common.core.KeyValue;
import cn.iocoder.yudao.framework.common.pojo.CommonResult;
import cn.iocoder.yudao.framework.common.util.date.LocalDateTimeUtils;
import cn.iocoder.yudao.framework.common.util.json.JsonUtils;
import cn.iocoder.yudao.gateway.util.SecurityFrameworkUtils;
import cn.iocoder.yudao.gateway.util.WebFrameworkUtils;
import cn.iocoder.yudao.framework.common.biz.system.oauth2.OAuth2TokenCommonApi;
import cn.iocoder.yudao.framework.common.biz.system.oauth2.dto.OAuth2AccessTokenCheckRespDTO;
import com.fasterxml.jackson.core.type.TypeReference;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;
import org.springframework.cloud.client.loadbalancer.reactive.ReactorLoadBalancerExchangeFilterFunction;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.time.Duration;
import java.util.Objects;
import java.util.function.Function;

import static cn.iocoder.yudao.framework.common.util.cache.CacheUtils.buildAsyncReloadingCache;

/**
 * Token 过滤器，验证 token 的有效性
 *
 * 1. 验证通过时，将 userId、userType、tenantId 通过 Header 转发给服务
 * 2. 验证不通过，还是会转发给服务。因为，接口是否需要登录的校验，还是交给服务自身处理
 */
@Component
public class TokenAuthenticationFilter implements GlobalFilter, Ordered {

    /**
     * CommonResult<OAuth2AccessTokenCheckRespDTO> 对应的 TypeReference
     */
    private static final TypeReference<CommonResult<OAuth2AccessTokenCheckRespDTO>> CHECK_RESULT_TYPE_REFERENCE
            = new TypeReference<CommonResult<OAuth2AccessTokenCheckRespDTO>>() {};

    /**
     * 空的 LoginUser 的结果
     *
     * 用于解决：
     * 1. getLoginUser 返回 Mono.empty() 时，flatMap 无法处理的问题
     * 2. Token 过期时返回 LOGIN_USER_EMPTY，避免缓存无法刷新
     */
    private static final LoginUser LOGIN_USER_EMPTY = new LoginUser();

    private final WebClient webClient;

    /**
     * 登录用户的本地缓存
     *
     * key1：多租户的编号
     * key2：访问令牌
     * 过期时间：1 分钟
     */
    private final LoadingCache<KeyValue<Long, String>, LoginUser> loginUserCache = buildAsyncReloadingCache(
            Duration.ofMinutes(1),
            new CacheLoader<KeyValue<Long, String>, LoginUser>() {
                @Override
                public LoginUser load(KeyValue<Long, String> token) {
                    String body = checkAccessToken(token.getKey(), token.getValue()).block();
                    return buildUser(body);
                }
            });

    public TokenAuthenticationFilter(ReactorLoadBalancerExchangeFilterFunction lbFunction) {
        // 使用 WebClient 调用 system-server 校验 Token
        // 为什么不使用 OpenFeign？因为 Gateway 基于 WebFlux，OpenFeign 不支持响应式
        this.webClient = WebClient.builder().filter(lbFunction).build();
    }

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 1. 移除可能伪造的 login-user 请求头
        exchange = SecurityFrameworkUtils.removeLoginUser(exchange);

        // 2. 提取 Token
        String token = SecurityFrameworkUtils.obtainAuthorization(exchange);
        if (StrUtil.isEmpty(token)) {
            // 无 Token，直接放行（交给服务端判断是否需要登录）
            return chain.filter(exchange);
        }

        // 3. 校验 Token 并获取用户信息
        ServerWebExchange finalExchange = exchange;
        return getLoginUser(exchange, token).defaultIfEmpty(LOGIN_USER_EMPTY).flatMap(user -> {
            // 3.1 无用户或 Token 已过期，直接放行
            if (user == LOGIN_USER_EMPTY
                    || user.getExpiresTime() == null
                    || LocalDateTimeUtils.beforeNow(user.getExpiresTime())) {
                return chain.filter(finalExchange);
            }

            // 3.2 有用户，设置登录用户到 exchange 属性
            SecurityFrameworkUtils.setLoginUser(finalExchange, user);

            // 3.3 将用户信息注入到 login-user 请求头
            ServerWebExchange newExchange = finalExchange.mutate()
                    .request(builder -> SecurityFrameworkUtils.setLoginUserHeader(builder, user))
                    .build();
            return chain.filter(newExchange);
        });
    }

    /**
     * 获取登录用户信息
     *
     * 优先从缓存获取，缓存未命中则调用远程服务
     */
    private Mono<LoginUser> getLoginUser(ServerWebExchange exchange, String token) {
        // 从缓存中获取
        Long tenantId = WebFrameworkUtils.getTenantId(exchange);
        KeyValue<Long, String> cacheKey = new KeyValue<Long, String>().setKey(tenantId).setValue(token);
        LoginUser localUser = loginUserCache.getIfPresent(cacheKey);
        if (localUser != null) {
            return Mono.just(localUser);
        }

        // 缓存不存在，请求远程服务
        return checkAccessToken(tenantId, token).flatMap((Function<String, Mono<LoginUser>>) body -> {
            LoginUser remoteUser = buildUser(body);
            if (remoteUser != null) {
                loginUserCache.put(cacheKey, remoteUser);
                return Mono.just(remoteUser);
            }
            return Mono.empty();
        });
    }

    /**
     * 调用 system-server 校验 Token
     *
     * 使用 WebClient 而非 OpenFeign，因为 Gateway 基于 WebFlux
     */
    private Mono<String> checkAccessToken(Long tenantId, String token) {
        return webClient.get()
                .uri(OAuth2TokenCommonApi.URL_CHECK,
                        uriBuilder -> uriBuilder.queryParam("accessToken", token).build())
                .headers(httpHeaders -> WebFrameworkUtils.setTenantIdHeader(tenantId, httpHeaders))
                .retrieve()
                .bodyToMono(String.class);
    }

    /**
     * 解析 Token 校验结果，构建 LoginUser
     */
    private LoginUser buildUser(String body) {
        CommonResult<OAuth2AccessTokenCheckRespDTO> result = JsonUtils.parseObject(body, CHECK_RESULT_TYPE_REFERENCE);
        if (result == null) {
            return null;
        }
        if (result.isError()) {
            // 令牌已过期（code = 401），返回 LOGIN_USER_EMPTY 避免缓存
            if (Objects.equals(result.getCode(), HttpStatus.UNAUTHORIZED.value())) {
                return LOGIN_USER_EMPTY;
            }
            return null;
        }

        OAuth2AccessTokenCheckRespDTO tokenInfo = result.getData();
        return new LoginUser()
                .setId(tokenInfo.getUserId())
                .setUserType(tokenInfo.getUserType())
                .setInfo(tokenInfo.getUserInfo())
                .setTenantId(tokenInfo.getTenantId())
                .setScopes(tokenInfo.getScopes())
                .setExpiresTime(tokenInfo.getExpiresTime());
    }

    @Override
    public int getOrder() {
        return -100; // 高优先级，与 Spring Security Filter 的顺序对齐
    }

}
```

**核心设计解析：**

### 4.1 为什么使用 WebClient 而非 OpenFeign？

Gateway 基于 WebFlux（响应式），而 OpenFeign 基于 Servlet（阻塞式）。两者不兼容：

```
OpenFeign（阻塞式）：
  调用 → 线程阻塞等待响应 → 返回结果
  问题：在 WebFlux 中阻塞线程会导致性能问题

WebClient（响应式）：
  调用 → 非阻塞等待 → 回调处理
  适合：WebFlux 环境
```

`ReactorLoadBalancerExchangeFilterFunction` 为 WebClient 提供了负载均衡能力，使其可以通过服务名（如 `system-server`）调用，而不需要硬编码实例地址。

### 4.2 缓存机制

```
Guava LoadingCache 缓存策略：

Key：KeyValue<tenantId, token>
  ├── tenantId：区分不同租户的同名 Token
  └── token：访问令牌本身

Value：LoginUser 对象

过期时间：1 分钟
  ├── 优点：减少远程调用，提高性能
  └── 缺点：Token 被吊销后最多 1 分钟才生效

缓存未命中时：
  1. 调用 system-server 校验 Token
  2. 校验成功 → 缓存结果
  3. 校验失败 → 不缓存，下次请求重新校验
```

### 4.3 LOGIN_USER_EMPTY 的作用

```java
private static final LoginUser LOGIN_USER_EMPTY = new LoginUser();
```

这个空对象解决了两个问题：

1. **`Mono.empty()` 导致 flatMap 不执行**：如果 `getLoginUser` 返回 `Mono.empty()`，后续的 `flatMap` 不会执行，导致请求无法继续。使用 `defaultIfEmpty(LOGIN_USER_EMPTY)` 保证 flatMap 总能执行。

2. **Token 过期时的缓存刷新**：当 Token 过期时，返回 `LOGIN_USER_EMPTY` 而非 `null`，这样缓存中会存储这个空对象。下次请求时，缓存命中但发现是空对象，会触发重新校验。

### 4.4 为什么不直接拒绝无效 Token？

```java
// 当前实现：Token 无效时放行，交给服务端判断
if (user == LOGIN_USER_EMPTY || ...) {
    return chain.filter(finalExchange);  // 放行
}

// 另一种方案：Token 无效时直接拒绝
if (user == LOGIN_USER_EMPTY || ...) {
    return WebFrameworkUtils.writeJSON(exchange,
        CommonResult.error(401, "Token 无效"));  // 拒绝
}
```

yudao-cloud 选择放行，原因是：

- 不是所有接口都需要登录（如公开 API）
- 是否需要登录的判断逻辑在各服务中，由 `@PreAuthorize` 等注解控制
- 网关只负责"尽力校验"，不负责"强制拒绝"

## 第五步：OAuth2TokenCommonApi 接口

Token 校验依赖 `system-server` 提供的 RPC 接口。这个接口定义在 `yudao-module-system-api` 中：

```java
// yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/biz/system/oauth2/OAuth2TokenCommonApi.java
@FeignClient(name = RpcConstants.SYSTEM_NAME)
public interface OAuth2TokenCommonApi {

    String PREFIX = RpcConstants.SYSTEM_PREFIX + "/oauth2/token";

    /**
     * 校验 Token 的 URL 地址，提供给 Gateway 使用
     */
    String URL_CHECK = "http://" + RpcConstants.SYSTEM_NAME + PREFIX + "/check";

    @GetMapping(PREFIX + "/check")
    CommonResult<OAuth2AccessTokenCheckRespDTO> checkAccessToken(
            @RequestParam("accessToken") String accessToken);

    // ... 其他方法
}
```

**`URL_CHECK` 常量：**

```
http://system-server/rpc-api/system/oauth2/token/check?accessToken=xxx
```

- `system-server`：服务名，由 LoadBalancer 解析为实际地址
- `/rpc-api/system/oauth2/token/check`：RPC 接口路径
- `accessToken`：要校验的 Token

## 第六步：过滤器执行顺序

Gateway 中有多个过滤器，它们的执行顺序由 `getOrder()` 决定：

```
过滤器执行顺序（order 越小，优先级越高）：

1. AccessLogFilter           (order = HIGHEST_PRECEDENCE = Integer.MIN_VALUE)
   └── 记录访问日志

2. CorsFilter                (WebFilter，最外层)
   └── 处理跨域

3. TokenAuthenticationFilter  (order = -100)
   └── Token 校验 + 用户信息注入

4. GrayLoadBalancer           (order = ReactiveLoadBalancerClientFilter.ORDER - 50)
   └── 灰度负载均衡选择实例

5. 路由过滤器                 (GatewayFilter，如 RewritePath)
   └── 路径重写等
```

**为什么 TokenAuthenticationFilter 的 order 是 -100？**

- 需要在路由转发之前执行（order < 10000，因为 `ReactiveLoadBalancerClientFilter` 的默认 order 是 10150）
- 需要在跨域处理之后执行（CorsFilter 是 WebFilter，在 GlobalFilter 之前）
- -100 与 Spring Security Filter 的顺序对齐，保持一致性

## 第七步：验证 Token 校验

### 7.1 启动服务

确保以下服务已启动：
1. Nacos Server
2. system-server（提供 Token 校验接口）
3. gateway-server

### 7.2 测试无 Token 请求

```bash
# 无 Token 请求，应该直接放行
curl http://127.0.0.1:48080/admin-api/system/auth/login

# 预期：请求被转发到 system-server，由服务端判断是否需要登录
```

### 7.3 测试有效 Token

```bash
# 先获取 Token
TOKEN=$(curl -s -X POST http://127.0.0.1:48080/admin-api/system/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}' | jq -r '.data.accessToken')

# 使用 Token 访问需要登录的接口
curl http://127.0.0.1:48080/admin-api/system/user/profile \
  -H "Authorization: Bearer $TOKEN"

# 预期：请求被转发，且下游服务能从 login-user 请求头获取用户信息
```

### 7.4 测试无效 Token

```bash
# 使用无效 Token
curl http://127.0.0.1:48080/admin-api/system/user/profile \
  -H "Authorization: Bearer invalid-token"

# 预期：请求被放行（网关不拒绝），但下游服务会返回 401
```

## 租户 ID 透传机制

### 透传流程

```
前端请求
    │
    ▼
Header: tenant-id: 1
    │
    ▼
Gateway 的 TokenAuthenticationFilter
    │
    ├── 从请求头提取 tenant-id
    ├── 从 Token 校验结果中获取 tenantId
    └── 两者取优先级高的（Token 中的 tenantId）
    │
    ▼
注入到请求头 login-user 中（JSON 格式）
    │
    ▼
下游服务从 login-user 中解析 tenantId
    │
    └── TenantContextHolder.setTenantId(tenantId)
```

### 下游服务如何获取用户信息

下游服务（如 system-server）通过 `yudao-spring-boot-starter-security` 中的 `TokenAuthenticationFilter` 解析 `login-user` 请求头：

```java
// yudao-framework/yudao-spring-boot-starter-security 中的 TokenAuthenticationFilter
String loginUserStr = request.getHeader("login-user");
if (StringUtils.hasText(loginUserStr)) {
    loginUserStr = URLDecoder.decode(loginUserStr, StandardCharsets.UTF_8.name());
    LoginUser loginUser = JsonUtils.parseObject(loginUserStr, LoginUser.class);
    // 设置到 SecurityContext 中
    SecurityFrameworkUtils.setLoginUser(loginUser);
    // 设置租户 ID
    TenantContextHolder.setTenantId(loginUser.getTenantId());
}
```

## 本章小结

本章实现了网关的核心过滤器 TokenAuthenticationFilter：

1. **LoginUser 数据模型**：网关侧的用户信息载体
2. **SecurityFrameworkUtils**：安全相关工具类（Token 提取、用户信息注入）
3. **WebFrameworkUtils**：Web 相关工具类（租户 ID、客户端 IP）
4. **TokenAuthenticationFilter**：核心过滤器，实现 Token 校验和用户信息转发
5. **缓存机制**：使用 Guava LoadingCache 缓存 Token 校验结果，1 分钟过期
6. **租户透传**：从 Token 校验结果中获取 tenantId，注入到 login-user 请求头

下一章将实现跨域处理和限流配置。
