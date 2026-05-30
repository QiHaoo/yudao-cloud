# 06-核心设计：LoginUser 统一身份模型

## 场景切入

认证通过后，系统知道了"这个请求是张三发的"。但"张三"这个身份信息该怎么表示？

最简单的做法：存一个 `userId = 1L`。但你很快发现这远远不够——业务代码需要知道：
- 张三是管理员还是普通会员？（`userType`）
- 张三属于哪个租户？（`tenantId`）—— 如果是 SaaS 系统
- 张三的 OAuth2 授权范围是什么？（`scopes`）—— 如果有第三方接入
- 张三的 Token 什么时候过期？（`expiresTime`）
- 张三的昵称和部门是什么？（`info`）—— 用于日志记录和权限判断

你需要一个**统一的、可扩展的**登录用户信息载体。

## LoginUser 的设计

```java
public class LoginUser {
    private Long id;                    // 用户编号
    private Integer userType;           // 用户类型（1-会员 2-管理员）
    private Map<String, String> info;   // 额外信息（昵称、部门ID等）
    private Long tenantId;              // 租户编号
    private List<String> scopes;        // OAuth2 授权范围
    private LocalDateTime expiresTime;  // 过期时间

    // 上下文字段（不序列化）
    @JsonIgnore
    private Map<String, Object> context;
    private Long visitTenantId;         // 访问的租户编号
}
```

### 字段逐一解析

#### `id` + `userType`：身份的唯一标识

单靠 `id` 无法唯一确定一个用户，因为同一个 `id` 在不同用户类型下可能代表不同的人。比如：
- 会员 ID=1 是"张三"
- 管理员 ID=1 也是"张三"（但可能是不同的人，或者同一个人的不同身份）

`UserTypeEnum` 定义了两种用户类型：
```java
public enum UserTypeEnum {
    MEMBER(1, "会员"),   // C 端普通用户
    ADMIN(2, "管理员");  // B 端管理后台
}
```

这种设计支持了**多端登录**：同一个系统既可以有管理后台（管理员登录），也可以有用户端（会员登录）。

#### `info`：可扩展的额外信息

为什么用 `Map<String, String>` 而不是固定字段？因为不同场景需要的信息不同：

- 管理员需要：昵称（`nickname`）、部门ID（`deptId`）
- 会员需要：昵称（`nickname`）、手机号（`mobile`）
- 未来可能需要：头像、VIP 等级、积分……

用 Map 可以避免 LoginUser 随业务需求不断膨胀。框架预定义了两个常用 key：
```java
public static final String INFO_KEY_NICKNAME = "nickname";
public static final String INFO_KEY_DEPT_ID = "deptId";
```

#### `tenantId`：多租户标识

在 SaaS 系统中，多个租户共享同一个服务实例。`tenantId` 标识当前用户属于哪个租户，后续的多租户 SQL 拦截器会用它自动追加 `WHERE tenant_id = ?` 条件。

#### `scopes`：OAuth2 授权范围

当系统接入第三方应用时，`scopes` 控制该 Token 能访问哪些资源。比如：
- `user:read` —— 只能读取用户信息
- `user:write` —— 可以修改用户信息
- `order:*` —— 可以访问所有订单接口

对于内部管理后台，`scopes` 通常为空。

#### `expiresTime`：过期时间

记录 Token 的过期时间，用于：
- 前端判断是否需要刷新 Token
- 业务代码判断 Token 是否即将过期

#### `context`：请求级临时缓存

`context` 是一个 `@JsonIgnore` 的 Map，不会被序列化到 login-user Header 中。它的用途是在一次请求内缓存临时数据，避免重复计算。

例如：某个权限校验需要查询当前用户的部门树，第一次查询后存入 `context`，后续直接从缓存读取。

#### `visitTenantId`：跨租户访问

管理员有时需要"切换"到其他租户的视角查看数据。此时：
- `tenantId` —— 管理员自己的租户（如平台租户）
- `visitTenantId` —— 管理员要访问的目标租户

当 `visitTenantId != tenantId` 时，框架会跳过权限校验（因为跨租户的权限体系不同）。

## LoginUser 与 Spring Security 的集成

LoginUser 不是一个孤立的数据类，它深度集成在 Spring Security 的体系中：

### 存储：SecurityContextHolder

认证成功后，LoginUser 被包装成 `UsernamePasswordAuthenticationToken` 放入 SecurityContextHolder：

```java
public static void setLoginUser(LoginUser loginUser, HttpServletRequest request) {
    UsernamePasswordAuthenticationToken authenticationToken =
        new UsernamePasswordAuthenticationToken(loginUser, null, Collections.emptyList());
    SecurityContextHolder.getContext().setAuthentication(authenticationToken);
}
```

注意 `principal` 参数传的是 `loginUser` 对象（而不是用户名字符串），`credentials` 传 `null`（Token 认证不需要密码），`authorities` 传空列表（权限通过 `@ss` Bean 校验，不走 Spring Security 的 Authority 机制）。

### 读取：SecurityFrameworkUtils

业务代码通过 `SecurityFrameworkUtils.getLoginUser()` 获取当前用户：

```java
public static LoginUser getLoginUser() {
    Authentication authentication = getAuthentication();
    if (authentication == null) {
        return null;
    }
    return authentication.getPrincipal() instanceof LoginUser
        ? (LoginUser) authentication.getPrincipal()
        : null;
}
```

为什么要 `instanceof` 检查？因为在某些场景下（如 Spring Security 的测试支持），`principal` 可能是其他类型。

### 快捷方法

框架提供了常用的快捷方法，避免业务代码每次都手动从 `info` Map 中取值：

```java
// 获取当前用户 ID
Long userId = SecurityFrameworkUtils.getLoginUserId();

// 获取当前用户昵称
String nickname = SecurityFrameworkUtils.getLoginUserNickname();

// 获取当前用户部门 ID
Long deptId = SecurityFrameworkUtils.getLoginUserDeptId();
```

## 为什么不直接用 Spring Security 的 UserDetailsService？

Spring Security 提供了 `UserDetailsService` 接口，用于加载用户详情。为什么本项目不用它？

| 对比维度 | UserDetailsService | 自研 LoginUser |
|---------|-------------------|---------------|
| 加载时机 | 认证时加载（每次登录） | Token 校验时从 Redis/MySQL 获取 |
| 数据来源 | 通常是数据库查询 | Token 携带的信息（已缓存在 Redis） |
| 扩展性 | 固定返回 `UserDetails` 接口 | 自由设计字段，支持 `context` 临时缓存 |
| 多用户类型 | 需要自定义实现 | `userType` 字段天然支持 |
| 多租户 | 需要额外实现 | `tenantId` 字段天然支持 |

核心原因：**本项目不使用 Spring Security 的认证流程**。传统的 `UsernamePasswordAuthenticationFilter` → `AuthenticationManager` → `UserDetailsService` 流程被完全跳过，取而代之的是自研的 `TokenAuthenticationFilter` → `OAuth2TokenCommonApi` 流程。

这是一个有意的设计选择：Spring Security 的认证流程太重了（支持 Remember-Me、Session 并发控制等），而本项目只需要 Token 认证。

## LoginUser 的生命周期

```
1. Token 校验成功
   OAuth2TokenCommonApi.checkAccessToken(token) 返回用户信息
   │
   ▼
2. 构建 LoginUser
   new LoginUser()
       .setId(accessToken.getUserId())
       .setUserType(accessToken.getUserType())
       .setInfo(accessToken.getUserInfo())
       .setTenantId(accessToken.getTenantId())
       .setScopes(accessToken.getScopes())
       .setExpiresTime(accessToken.getExpiresTime())
   │
   ▼
3. 放入 SecurityContextHolder
   SecurityFrameworkUtils.setLoginUser(loginUser, request)
   │
   ▼
4. 业务代码使用
   SecurityFrameworkUtils.getLoginUserId()
   @PreAuthorize("@ss.hasPermission('...')")
   │
   ▼
5. 请求结束
   SecurityContextHolder 自动清除（OncePerRequestFilter 保证）
```

## 思考题

1. **LoginUser 的 `info` 字段用 `Map<String, String>` 存储，而不是用强类型对象。这种设计的权衡是什么？** 在什么场景下你会改用强类型？

2. **为什么 LoginUser 的 `context` 字段用 `@JsonIgnore` 标注？** 如果不标注会有什么问题？提示：考虑 login-user Header 透传场景。

3. **如果要支持"一个用户同时在多个终端登录"（如 PC + 手机），LoginUser 需要增加什么字段？** 现有的设计能否支持？

4. **`visitTenantId` 和 `tenantId` 的分离设计，除了跨租户访问，还能用在什么场景？** 比如：租户管理员查看平台数据。
