# 04-前置知识：OAuth2.0 与 Token 认证

## 从"第三方登录"说起

你在浏览器上打开一个新注册的电商网站，看到一个按钮："用微信登录"。你点了一下，微信弹出一个页面，告诉你"某某电商想获取你的头像和昵称"，你点了"同意"，然后就自动登录成功了。

整个过程你没有输入任何账号密码。但问题来了：

- 那个电商网站**并没有拿到你的微信密码**，它是怎么登录你的？
- 你点了"同意"之后，电商网站拿到了什么？它能用你的微信发朋友圈吗？
- 如果你不想让这个电商继续访问你的微信信息，怎么"收回授权"？

这些问题的答案，就是 **OAuth2.0 协议**。

## OAuth2.0 的诞生背景

### 没有 OAuth 的时代

在 OAuth 出现之前，如果你想让一个第三方应用访问你在另一个平台上的数据，唯一的办法是**把你的账号密码给它**。

比如，你想让一个"打印照片"的网站从你的相册网站下载照片，你得把相册网站的账号密码告诉它。这带来三个严重问题：

1. **密码泄露风险**：第三方网站保存了你的密码，它被攻破了你的密码就泄露了
2. **权限无法控制**：你给了密码，第三方网站能做任何事——删照片、改密码、发消息
3. **无法撤销**：你想取消授权，只能改密码，但这会影响你自己正常使用

### OAuth 的核心思想

OAuth 的核心思想是**用"令牌（Token）"代替"密码"**。

回到刚才的场景：你不需要把相册密码告诉打印网站，而是通过相册网站生成一个**专用的、有限权限的令牌**给打印网站。这个令牌只能读取照片，不能删除、不能改密码。你随时可以撤销这个令牌，而不影响你自己登录相册网站。

类比一下：你的家门密码是"账号密码"，你把密码告诉了快递员，快递员就能进你家做任何事。而 OAuth 相当于给快递员一个**临时门禁卡**，只能开大门、只能在今天用、只能取快递。快递员不需要知道你的家门密码。

### OAuth 2.0 vs OAuth 1.0

OAuth 2.0（RFC 6749，2012 年发布）是对 OAuth 1.0 的重大升级：

| 维度 | OAuth 1.0 | OAuth 2.0 |
|------|-----------|-----------|
| 签名机制 | 每个请求都需要复杂的签名（HMAC-SHA1） | 使用 HTTPS 保证安全，不再要求请求签名 |
| 授权流程 | 只有一种流程 | 定义了四种授权模式，适配不同场景 |
| Token 类型 | 只有一种 Token | 区分 Access Token 和 Refresh Token |
| 实现复杂度 | 非常复杂，容易出错 | 简化了很多，但也更依赖 HTTPS |
| 适用范围 | 主要用于 Web 应用 | Web、移动 App、桌面应用、IoT 设备都能用 |

今天我们说的"OAuth"，基本都是指 OAuth 2.0。

## OAuth2.0 的四种授权模式

OAuth 2.0 定义了四种获取令牌的方式，称为**授权模式（Grant Type）**。每种模式适用于不同的场景。下面逐一讲解。

### 角色定义

在理解四种模式之前，先认识 OAuth 2.0 中的四个角色：

| 角色 | 含义 | 电商例子 |
|------|------|---------|
| **资源所有者（Resource Owner）** | 数据的拥有者，通常是用户 | 你（电商系统的管理员） |
| **客户端（Client）** | 想要访问资源的第三方应用 | 一个数据分析工具 |
| **授权服务器（Authorization Server）** | 验证用户身份、颁发令牌的服务 | 电商系统的 OAuth2 服务 |
| **资源服务器（Resource Server）** | 存放用户数据的 API 服务 | 电商系统的商品/订单 API |

### 模式一：授权码模式（Authorization Code）

**这是最常用、最安全的模式**，适合有后端服务器的 Web 应用。

**电商场景**：你公司的数据分析工具需要访问电商后台的销售数据。数据分析工具是一个独立的 Web 应用，有自己的后端服务器。

**流程**：

```
  浏览器                   数据分析工具              授权服务器              资源服务器
  (用户)                  (Client 后端)            (电商 OAuth2)           (电商 API)
    │                          │                       │                      │
    │  1. 点击"用电商账号登录"   │                       │                      │
    │─────────────────────────>│                       │                      │
    │                          │                       │                      │
    │  2. 重定向到授权页面       │                       │                      │
    │<─────────────────────────│                       │                      │
    │                          │                       │                      │
    │  3. 跳转到授权页面         │                       │                      │
    │──────────────────────────────────────────────────>│                      │
    │                          │                       │                      │
    │  4. 用户登录 + 同意授权    │                       │                      │
    │──────────────────────────────────────────────────>│                      │
    │                          │                       │                      │
    │  5. 返回授权码(code)       │                       │                      │
    │<──────────────────────────────────────────────────│                      │
    │  (302 重定向到 redirect_uri?code=abc123)          │                      │
    │                          │                       │                      │
    │  6. 浏览器带到 code        │                       │                      │
    │─────────────────────────>│                       │                      │
    │                          │                       │                      │
    │                          │  7. 用 code 换 Token   │                      │
    │                          │──────────────────────>│                      │
    │                          │  (POST /token         │                      │
    │                          │   grant_type=         │                      │
    │                          │   authorization_code  │                      │
    │                          │   code=abc123         │                      │
    │                          │   client_id=xxx       │                      │
    │                          │   client_secret=xxx)  │                      │
    │                          │                       │                      │
    │                          │  8. 返回 Token         │                      │
    │                          │<──────────────────────│                      │
    │                          │  {access_token,       │                      │
    │                          │   refresh_token,      │                      │
    │                          │   expires_in}         │                      │
    │                          │                       │                      │
    │                          │  9. 用 Token 访问 API  │                      │
    │                          │─────────────────────────────────────────────>│
    │                          │                       │                      │
    │                          │  10. 返回数据           │                      │
    │                          │<─────────────────────────────────────────────│
```

**关键安全点**：

- 授权码（code）是**一次性**的，用过即废
- 授权码通过浏览器传递（步骤 5），但 Token 通过后端传递（步骤 7-8），Token 永远不经过浏览器
- `client_secret` 只在后端使用，不暴露给前端

**为什么安全？** 因为即使有人截获了授权码（步骤 5），他也无法用它换到 Token，因为他不知道 `client_secret`。

### 模式二：隐式模式（Implicit）

**已不推荐使用**，仅用于理解历史背景。适合纯前端应用（没有后端服务器），但已被 PKCE 方案取代。

**电商场景**：一个纯 JavaScript 的数据看板，没有自己的后端服务器，直接在浏览器中调用电商 API。

**流程**：

```
  浏览器                   数据看板(JS)             授权服务器
  (用户)                  (Client 前端)            (电商 OAuth2)
    │                          │                       │
    │  1. 点击"用电商账号登录"   │                       │
    │─────────────────────────>│                       │
    │                          │                       │
    │  2. 重定向到授权页面       │                       │
    │<─────────────────────────│                       │
    │                          │                       │
    │  3. 跳转到授权页面         │                       │
    │──────────────────────────────────────────────────>│
    │                          │                       │
    │  4. 用户登录 + 同意授权    │                       │
    │──────────────────────────────────────────────────>│
    │                          │                       │
    │  5. 直接返回 Token         │                       │
    │<──────────────────────────────────────────────────│
    │  (302 重定向到 redirect_uri#access_token=xxx)     │
    │                          │                       │
    │  6. 浏览器从 URL 片段提取   │                       │
    │     Token               │                       │
    │─────────────────────────>│                       │
    │                          │                       │
    │                          │  7. 直接用 Token 访问   │
    │                          │──────────────────────>│
```

**为什么不再推荐？**

- Token 直接暴露在 URL 中（步骤 5），容易被中间人攻击或日志记录
- 没有 `client_secret`，任何人都可以伪造客户端
- 没有 Refresh Token，Token 过期后必须重新授权

### 模式三：密码模式（Password）

**这是本项目主要使用的模式**，适合高度信任的客户端（比如自家的管理后台前端）。

**电商场景**：你的电商管理后台前端（Vue 应用）需要登录。因为前端和后端是同一个团队开发的，前端可以被信任直接收集用户名和密码。

**流程**：

```
  浏览器                   前端(Vue)                授权服务器
  (管理员)                (Client 前端)            (电商 OAuth2)
    │                          │                       │
    │  1. 输入账号密码           │                       │
    │─────────────────────────>│                       │
    │                          │                       │
    │                          │  2. 直接发送账号密码     │
    │                          │──────────────────────>│
    │                          │  (POST /token         │
    │                          │   grant_type=password  │
    │                          │   username=admin       │
    │                          │   password=123456      │
    │                          │   client_id=default    │
    │                          │   client_secret=xxx)   │
    │                          │                       │
    │                          │  3. 验证账号密码        │
    │                          │                       │
    │                          │  4. 返回 Token         │
    │                          │<──────────────────────│
    │                          │  {access_token,       │
    │                          │   refresh_token,      │
    │                          │   expires_in}         │
    │                          │                       │
    │  5. 登录成功               │                       │
    │<─────────────────────────│                       │
```

**为什么本项目用密码模式？**

在 `AdminAuthServiceImpl.createTokenAfterLoginSuccess()` 中可以看到，后台管理的登录直接使用密码模式：

```java
// 源码位置：AdminAuthServiceImpl.java
private AuthLoginRespVO createTokenAfterLoginSuccess(Long userId, String username, LoginLogTypeEnum logType) {
    // 插入登陆日志
    createLoginLog(userId, username, logType, LoginResultEnum.SUCCESS);
    // 创建访问令牌
    OAuth2AccessTokenDO accessTokenDO = oauth2TokenService.createAccessToken(
        userId, getUserType().getValue(),
        OAuth2ClientConstants.CLIENT_ID_DEFAULT, null);  // 默认客户端
    // 构建返回结果
    return BeanUtils.toBean(accessTokenDO, AuthLoginRespVO.class);
}
```

管理后台的前端和后端属于同一个系统，前端是"高度信任"的客户端。让用户先在前端输入账号密码，后端验证后直接返回 Token——这就是密码模式的典型场景。

### 模式四：客户端模式（Client Credentials）

**适合服务间调用**，没有"用户"的概念，只有"客户端"。

**电商场景**：你的库存服务需要定时调用电商后台的商品 API，获取最新的商品列表。这是机器对机器的调用，没有用户参与。

**流程**：

```
  库存服务                   授权服务器              资源服务器
  (Client)                  (电商 OAuth2)           (电商 API)
    │                          │                      │
    │  1. 用自己的凭证请求 Token │                      │
    │─────────────────────────>│                      │
    │  (POST /token            │                      │
    │   grant_type=            │                      │
    │   client_credentials     │                      │
    │   client_id=inventory    │                      │
    │   client_secret=xxx)     │                      │
    │                          │                      │
    │  2. 返回 Token            │                      │
    │<─────────────────────────│                      │
    │  {access_token,          │                      │
    │   expires_in}            │                      │
    │                          │                      │
    │  3. 用 Token 调用 API     │                      │
    │────────────────────────────────────────────────>│
    │                          │                      │
    │  4. 返回数据               │                      │
    │<────────────────────────────────────────────────│
```

**特点**：
- 没有 Refresh Token（因为客户端自己就能重新获取）
- 没有"用户"参与，Token 代表的是"客户端"本身
- 适用于微服务间的内部调用

本项目中，`OAuth2GrantServiceImpl.grantClientCredentials()` 的实现可以看到，客户端模式下用户 ID 被设为 0（无用户）：

```java
// 源码位置：OAuth2GrantServiceImpl.java
@Override
public OAuth2AccessTokenDO grantClientCredentials(String clientId, List<String> scopes) {
    // 特殊：客户端模式下没有用户，userId 设为 0
    return oauth2TokenService.createAccessToken(0L, UserTypeEnum.ADMIN.getValue(), clientId, scopes);
}
```

### 四种模式对比

| 维度 | 授权码模式 | 隐式模式 | 密码模式 | 客户端模式 |
|------|-----------|---------|---------|-----------|
| **适用场景** | 第三方 Web 应用 | 纯前端应用（已废弃） | 高度信任的自家应用 | 服务间调用 |
| **是否有用户参与** | 是 | 是 | 是 | 否 |
| **是否需要 client_secret** | 是（后端使用） | 否 | 是 | 是 |
| **是否有 Refresh Token** | 是 | 否 | 是 | 否 |
| **Token 经过浏览器** | 否（只经过后端） | 是（URL 片段） | 否（只经过后端） | 不适用 |
| **安全等级** | 最高 | 最低 | 中等 | 中等 |
| **本项目支持** | 是 | 是（不推荐） | 是（主要使用） | 是 |
| **典型电商场景** | 第三方数据分析工具接入 | - | 管理后台登录 | 库存服务调用商品 API |

### 还有一种：刷新模式（Refresh Token）

严格来说这不是"获取授权"的模式，而是"续期"的模式。当 Access Token 过期时，客户端可以用 Refresh Token 获取一个新的 Access Token，而不需要用户重新输入密码。

```
  前端                      授权服务器
    │                          │
    │  1. Access Token 过期了    │
    │                          │
    │  2. 用 Refresh Token 换   │
    │     新的 Access Token     │
    │─────────────────────────>│
    │  (POST /token            │
    │   grant_type=refresh_token│
    │   refresh_token=xxx      │
    │   client_id=default)     │
    │                          │
    │  3. 返回新的 Token 对      │
    │<─────────────────────────│
    │  {access_token: 新的,    │
    │   refresh_token: 新的,   │
    │   expires_in: 7200}      │
```

## Token 是什么？

### 从"通行证"到"令牌"

前面反复提到 Token，现在来正式定义它。

**Token（令牌）是一串随机字符串，它代表着某个用户对某个客户端的授权。** 当客户端携带 Token 请求 API 时，资源服务器通过查找 Token 对应的用户信息来识别"请求者是谁"。

在本项目中，Token 使用 `IdUtil.fastSimpleUUID()` 生成，是一个 32 位的十六进制字符串，例如：`a1b2c3d4e5f6789012345678abcdef01`。

### Access Token vs Refresh Token

OAuth 2.0 引入了两种 Token，各有分工：

| 维度 | Access Token（访问令牌） | Refresh Token（刷新令牌） |
|------|------------------------|--------------------------|
| **用途** | 携带它访问 API | 用来获取新的 Access Token |
| **有效期** | 短（本项目默认 7200 秒 = 2 小时） | 长（本项目默认 2592000 秒 = 30 天） |
| **发送频率** | 每次 API 请求都发送 | 只在 Access Token 过期时发送 |
| **泄露风险** | 高（传输频繁） | 低（使用频率低） |
| **安全策略** | 即使泄露，影响范围有限（短期有效） | 必须严格保护 |

**为什么要分两种 Token？** 这是一个安全与体验的平衡：

- 如果只有一个长期 Token：用户体验好（不用频繁登录），但一旦泄露，攻击者可以长期使用
- 如果只有一个短期 Token：安全性好，但用户每 2 小时就要重新登录一次
- Access Token + Refresh Token：Access Token 短期有效，泄露了影响有限；过期后用 Refresh Token 静默续期，用户无感知

### Token 里存了什么？

查看本项目的数据模型 `OAuth2AccessTokenDO`，一个 Token 包含以下信息：

```
OAuth2AccessTokenDO
├── id              : Long          // 数据库自增 ID
├── accessToken     : String        // 访问令牌（32 位 UUID）
├── refreshToken    : String        // 关联的刷新令牌
├── userId          : Long          // 用户编号
├── userType        : Integer       // 用户类型（管理员/会员）
├── userInfo        : Map           // 用户附加信息（昵称、部门等）
├── clientId        : String        // 客户端编号
├── scopes          : List<String>  // 授权范围
├── expiresTime     : LocalDateTime // 过期时间
└── tenantId        : Long          // 租户编号（多租户支持）
```

Token 本身不包含用户信息（它只是一个随机字符串），但服务端保存了 Token 与用户信息的映射关系。当请求携带 Token 到达时，服务端通过查找这个映射来识别用户。

## Token 的存储策略

Token 生成之后，存在哪里？这是一个关键的架构决策。

### 方案对比

| 存储方案 | 优点 | 缺点 | 适用场景 |
|---------|------|------|---------|
| **纯 MySQL** | 数据持久化，不会丢失 | 每次校验都要查数据库，性能差 | 小规模系统 |
| **纯 Redis** | 性能极高，自动过期 | 重启可能丢失数据 | 对持久性要求不高 |
| **Redis + MySQL 双写** | 兼顾性能和持久性 | 实现复杂，需要维护一致性 | 生产环境（本项目采用） |

### 本项目的双写策略

本项目采用 **Redis + MySQL 双写** 策略，具体规则如下：

```
创建 Token 时：
  1. 写入 MySQL（持久化）
  2. 写入 Redis（设置 TTL = Token 剩余有效期）
     ┌──────────────┐      ┌──────────────┐
     │    MySQL     │      │    Redis     │
     │  (持久化)     │      │  (高速缓存)   │
     │              │      │              │
     │  完整数据     │      │  完整数据     │
     │  永久保存     │      │  TTL 过期自动  │
     │              │      │  删除         │
     └──────────────┘      └──────────────┘

查询 Token 时：
  1. 优先查 Redis → 命中则直接返回
  2. Redis 未命中 → 查 MySQL
  3. MySQL 命中 → 回写 Redis（下次就能命中）→ 返回
  4. MySQL 也未命中 → Token 不存在

删除 Token 时：
  1. 同时删除 MySQL 和 Redis 中的数据
```

源码中可以清楚地看到这个策略（`OAuth2TokenServiceImpl.getAccessToken()`）：

```java
// 源码位置：OAuth2TokenServiceImpl.java
@Override
public OAuth2AccessTokenDO getAccessToken(String accessToken) {
    // 优先从 Redis 中获取
    OAuth2AccessTokenDO accessTokenDO = oauth2AccessTokenRedisDAO.get(accessToken);
    if (accessTokenDO != null) {
        return accessTokenDO;
    }

    // 获取不到，从 MySQL 中获取访问令牌
    accessTokenDO = oauth2AccessTokenMapper.selectByAccessToken(accessToken);
    // ... 省略特殊逻辑 ...

    // 如果在 MySQL 存在，则往 Redis 中写入
    if (accessTokenDO != null && !DateUtils.isExpired(accessTokenDO.getExpiresTime())) {
        oauth2AccessTokenRedisDAO.set(accessTokenDO);
    }
    return accessTokenDO;
}
```

Redis 中的存储也设置了自动过期（`OAuth2AccessTokenRedisDAO.set()`）：

```java
// 源码位置：OAuth2AccessTokenRedisDAO.java
public void set(OAuth2AccessTokenDO accessTokenDO) {
    String redisKey = formatKey(accessTokenDO.getAccessToken());
    long time = LocalDateTimeUtil.between(
        LocalDateTime.now(), accessTokenDO.getExpiresTime(), ChronoUnit.SECONDS);
    if (time > 0) {
        stringRedisTemplate.opsForValue().set(
            redisKey, JsonUtils.toJsonString(accessTokenDO), time, TimeUnit.SECONDS);
    }
}
```

### 为什么这样设计？

- **Redis 解决性能问题**：每次 API 请求都要校验 Token，如果每次都查 MySQL，数据库压力会很大。Redis 的读取性能是 MySQL 的 10-100 倍。
- **MySQL 解决持久性问题**：Redis 重启可能丢失数据。如果 Token 只存在 Redis 中，Redis 重启后所有用户都会被"踢下线"。有了 MySQL，即使 Redis 重启，Token 也能从 MySQL 恢复。
- **Redis 自动过期解决清理问题**：Token 过期后，Redis 自动删除，不需要额外的清理逻辑。MySQL 中的过期数据则通过定时任务清理。

## Token 的生命周期

一个 Token 从诞生到消亡，经历以下阶段：

```
  创建                  使用                   刷新                  过期                  清理
   │                    │                     │                    │                    │
   ▼                    ▼                     ▼                    ▼                    ▼
┌──────┐  携带Token  ┌──────┐  Access过期  ┌──────┐  超过有效期  ┌──────┐  定时任务  ┌──────┐
│ 生成 │──────────>│ 校验 │──────────>│ 续期 │──────────>│ 失效 │──────────>│ 删除 │
│ Token│  请求API   │ Token│  用Refresh  │ Token│  Token不可用 │ Token│  物理清理  │ 数据 │
└──────┘           └──────┘  Token换新   └──────┘           └──────┘           └──────┘
```

### 阶段一：创建（Create）

当用户登录成功（密码模式、授权码模式等），服务端创建 Token 对：

```java
// 源码位置：OAuth2TokenServiceImpl.java
@Override
public OAuth2AccessTokenDO createAccessToken(Long userId, Integer userType,
                                              String clientId, List<String> scopes) {
    OAuth2ClientDO clientDO = oauth2ClientService.validOAuthClientFromCache(clientId);
    // 创建刷新令牌
    OAuth2RefreshTokenDO refreshTokenDO = createOAuth2RefreshToken(userId, userType, clientDO, scopes);
    // 创建访问令牌
    return createOAuth2AccessToken(refreshTokenDO, clientDO);
}
```

创建过程：
1. 先创建 Refresh Token（写入 MySQL）
2. 再创建 Access Token（写入 MySQL + Redis）
3. Access Token 关联到 Refresh Token

### 阶段二：使用（Use）

客户端每次请求 API 时，在 Header 中携带 Access Token：

```
GET /admin-api/product/list
Authorization: Bearer a1b2c3d4e5f6789012345678abcdef01
```

服务端通过 `checkAccessToken()` 校验 Token 的有效性：

```java
// 源码位置：OAuth2TokenServiceImpl.java
@Override
public OAuth2AccessTokenDO checkAccessToken(String accessToken) {
    OAuth2AccessTokenDO accessTokenDO = getAccessToken(accessToken);
    if (accessTokenDO == null) {
        throw exception0(GlobalErrorCodeConstants.UNAUTHORIZED.getCode(), "访问令牌不存在");
    }
    if (DateUtils.isExpired(accessTokenDO.getExpiresTime())) {
        throw exception0(GlobalErrorCodeConstants.UNAUTHORIZED.getCode(), "访问令牌已过期");
    }
    return accessTokenDO;
}
```

### 阶段三：刷新（Refresh）

当 Access Token 过期，客户端用 Refresh Token 获取新的 Access Token：

```java
// 源码位置：OAuth2TokenServiceImpl.java
@Override
public OAuth2AccessTokenDO refreshAccessToken(String refreshToken, String clientId) {
    // 查询刷新令牌
    OAuth2RefreshTokenDO refreshTokenDO = oauth2RefreshTokenMapper.selectByRefreshToken(refreshToken);
    // 校验 Client 匹配
    // 移除旧的访问令牌
    // 检查刷新令牌是否过期
    // 创建新的访问令牌
    return createOAuth2AccessToken(refreshTokenDO, clientDO);
}
```

刷新过程的关键细节：
1. 旧的 Access Token 会被删除（MySQL + Redis）
2. 新的 Access Token 关联到同一个 Refresh Token
3. Refresh Token 本身不会被替换（除非它也过期了）

### 阶段四：过期（Expire）

Token 有明确的过期时间，由客户端（`OAuth2ClientDO`）配置决定：

```
OAuth2ClientDO
├── accessTokenValiditySeconds  : 7200    // Access Token 有效期：2 小时
└── refreshTokenValiditySeconds : 2592000 // Refresh Token 有效期：30 天
```

过期后的 Token 不能使用，但数据仍保留在数据库中（等待定时任务清理）。

### 阶段五：清理（Clean）

过期的 Token 数据需要定期清理，否则数据库会无限膨胀。本项目通过定时任务实现：

```java
// 源码位置：TokenCleanJob.java
@Component
public class TokenCleanJob {

    private static final Integer JOB_CLEAN_RETAIN_DAY = 14;  // 过期 14 天后清理
    private static final Integer DELETE_LIMIT = 100;         // 每批删除 100 条

    @XxlJob("tokenCleanJob")
    @TenantIgnore
    public void execute() {
        Integer refreshCount = oauth2TokenService.cleanRefreshToken(JOB_CLEAN_RETAIN_DAY, DELETE_LIMIT);
        Integer accessCount = oauth2TokenService.cleanAccessToken(JOB_CLEAN_RETAIN_DAY, DELETE_LIMIT);
    }
}
```

清理策略：
- 过期超过 **14 天**的 Token 才会被物理删除
- 每次删除 **100 条**，循环执行直到没有满足条件的数据
- 避免一次性删除大量数据造成数据库压力
- `@TenantIgnore` 注解确保跨租户清理

## 本项目的 OAuth2.0 实现与标准协议的关系

### 保留了什么？

本项目完整实现了 OAuth 2.0 的核心机制：

| 标准特性 | 本项目实现 | 对应源码 |
|---------|-----------|---------|
| 四种授权模式 | 全部支持 | `OAuth2GrantTypeEnum` 枚举定义了 5 种类型 |
| Access Token + Refresh Token | 完整实现 | `OAuth2AccessTokenDO` + `OAuth2RefreshTokenDO` |
| 客户端管理（client_id + client_secret） | 完整实现 | `OAuth2ClientDO`，支持配置有效期、授权范围等 |
| 授权范围（Scope） | 支持 | `OAuth2ClientDO.scopes` + `OAuth2ClientDO.autoApproveScopes` |
| 授权码模式的完整流程 | 支持 | `OAuth2CodeService` 管理授权码 |
| Token 校验接口 | 支持 | `OAuth2OpenController.checkToken()` |
| Token 撤销接口 | 支持 | `OAuth2OpenController.revokeToken()` |
| 授权页面 | 支持 | `OAuth2OpenController.authorize()` + `approveOrDeny()` |

### 简化了什么？

本项目没有使用 Spring Security OAuth2 框架，而是自研了 Token 管理层。主要简化点：

| 简化点 | 标准做法 | 本项目做法 | 原因 |
|-------|---------|-----------|------|
| **Token 格式** | JWT（自包含令牌） | 随机 UUID + 服务端存储 | JWT 无法主动撤销，服务端存储更灵活 |
| **授权服务器** | 独立的授权服务器（如 Keycloak） | 集成在 system 模块中 | 项目规模不需要独立部署 |
| **Token 校验** | 资源服务器本地校验 JWT 签名 | 通过 RPC 调用授权服务器校验 | 简化实现，统一管理 |
| **Scope 控制** | 严格的 Scope 隔离 | 宽松的 Scope 控制 | 管理后台场景不需要细粒度隔离 |
| **Spring Security OAuth2 依赖** | 使用框架提供的完整实现 | 自研 `OAuth2TokenService` 等 | 框架过重，学习成本高 |

### 为什么不用 Spring Security OAuth2？

项目源码注释中给出了直接说明：

> 从功能上，和 Spring Security OAuth 的 DefaultTokenServices + JdbcTokenStore 的功能，提供访问令牌、刷新令牌的操作

项目选择了**参考 Spring Security OAuth 的设计，但自研实现**。这样做的好处是：

1. **代码量少**：核心 Token 管理只有几个类，而 Spring Security OAuth2 有上百个类
2. **易于理解**：开发者可以直接阅读源码理解 Token 的完整生命周期
3. **灵活可控**：比如 Redis + MySQL 双写策略、Token 中携带用户信息等，都是自研才方便实现的
4. **降低依赖**：不引入庞大的 Spring Security OAuth2 依赖，减少版本冲突风险

## 思考题

1. **模式选择**：如果你要开发一个电商小程序（前端是微信小程序，后端是你的 Java 服务），应该选择哪种授权模式？为什么？如果小程序需要访问用户的微信头像和昵称，流程会有什么不同？

2. **安全权衡**：密码模式要求客户端直接接触用户密码，这在什么条件下是可接受的？如果你的管理后台要开放给第三方公司使用（而不是自用），密码模式还合适吗？应该怎么改？

3. **Token 存储**：本项目采用 Redis + MySQL 双写策略。如果 Redis 宕机了（不可用），系统还能正常工作吗？会有什么影响？你能想到的改进方案是什么？

4. **Token 过期策略**：本项目中 Access Token 有效期 2 小时，Refresh Token 有效期 30 天。如果一个管理员每天登录 8 小时，他多久会"被踢下线"需要重新输入密码？这个体验你能接受吗？有没有更好的方案？
