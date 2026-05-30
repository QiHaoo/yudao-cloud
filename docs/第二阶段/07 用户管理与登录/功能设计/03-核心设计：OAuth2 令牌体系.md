# 核心设计：OAuth2 令牌体系

## 从一个电商场景说起

你的电商平台后台管理系统上线了，运营人员小李每天早上登录系统处理订单。某天早上，小李登录后发现一个问题——她昨天晚上在公司电脑上登录的会话还在，但她现在用的是家里的笔记本。

这引出了令牌管理的核心问题：

- 一个用户可以同时在多个设备登录吗？（可以，本项目允许）
- 令牌过期后怎么办？（用 refreshToken 换新令牌）
- 用户主动退出后，令牌怎么处理？（立刻删除）
- 管理员怎么强制踢人？（删除该用户的所有令牌）

这些问题的答案，都藏在 OAuth2 令牌体系的设计中。

## OAuth2 授权模式

OAuth2 协议定义了多种授权模式（Grant Type），适用于不同的登录场景。本项目中 `OAuth2GrantTypeEnum` 定义了以下模式：

```
  OAuth2 授权模式
  ================

  1. password（密码模式）
     适用：管理后台登录
     流程：用户直接提供账号密码
     场景：运营人员在后台管理系统登录

  2. authorization_code（授权码模式）
     适用：第三方登录
     流程：跳转授权页 -> 获取授权码 -> 换取令牌
     场景：允许第三方应用接入你的电商 API

  3. implicit（简化模式）
     适用：纯前端应用
     流程：跳转授权页 -> 直接返回令牌
     场景：前端 SPA 应用（安全性较低，不推荐）

  4. client_credentials（客户端模式）
     适用：服务间调用
     流程：客户端直接用 clientId + secret 换取令牌
     场景：订单服务调用商品服务的内部 API

  5. refresh_token（刷新模式）
     适用：令牌续期
     流程：用 refreshToken 换取新的 accessToken
     场景：accessToken 过期后，无需重新登录
```

在实际的电商后台场景中，最常用的是 **password 模式**（管理员登录）和 **refresh_token 模式**（令牌续期）。客户端模式用于微服务间调用。

## 客户端配置：OAuth2ClientDO

在 OAuth2 体系中，"客户端"不是指浏览器或手机，而是指**接入认证系统的应用**。比如你的电商后台管理系统是一个客户端，移动端 App 是另一个客户端。

`OAuth2ClientDO` 的核心字段：

```java
public class OAuth2ClientDO extends BaseDO {
    private String clientId;                    // 客户端标识，如 "default"
    private String secret;                      // 客户端密钥
    private Integer accessTokenValiditySeconds;  // 访问令牌有效期（秒）
    private Integer refreshTokenValiditySeconds; // 刷新令牌有效期（秒）
    private List<String> authorizedGrantTypes;   // 允许的授权模式
    private List<String> scopes;                 // 授权范围
    private List<String> redirectUris;           // 重定向地址（授权码模式用）
    // ...
}
```

为什么需要客户端配置？因为不同应用的令牌策略不同：

| 客户端 | accessToken 有效期 | refreshToken 有效期 | 授权模式 |
|--------|-------------------|--------------------|---------|
| 管理后台（default） | 较短（如 2 小时） | 较长（如 30 天） | password, refresh_token |
| 移动端 App | 较长（如 7 天） | 更长（如 90 天） | password, sms, social |
| 第三方接入 | 中等（如 1 小时） | 中等（如 7 天） | authorization_code |
| 内部服务 | 短（如 30 分钟） | 无 | client_credentials |

本项目默认使用 `clientId = "default"`，对应管理后台场景：

```java
// OAuth2ClientConstants.java
public interface OAuth2ClientConstants {
    String CLIENT_ID_DEFAULT = "default";
}
```

## 令牌数据结构

### 访问令牌：OAuth2AccessTokenDO

访问令牌（Access Token）是用户每次请求时携带的凭证。对应数据库表 `system_oauth2_access_token`：

```java
public class OAuth2AccessTokenDO extends TenantBaseDO {
    private Long id;                    // 数据库自增 ID
    private String accessToken;         // 访问令牌（UUID 字符串）
    private String refreshToken;        // 关联的刷新令牌
    private Long userId;                // 用户编号
    private Integer userType;           // 用户类型（管理员/会员）
    private Map<String, String> userInfo; // 用户快照信息（昵称、部门ID等）
    private String clientId;            // 客户端标识
    private List<String> scopes;        // 授权范围
    private LocalDateTime expiresTime;  // 过期时间
}
```

几个设计要点：

**1. accessToken 是 UUID，不是自增 ID**

```java
private static String generateAccessToken() {
    return IdUtil.fastSimpleUUID();
    // 生成类似 "a1b2c3d4e5f67890" 的随机字符串
}
```

为什么不用自增 ID？因为令牌会被外部使用（传给前端、传给其他服务），如果用自增 ID，攻击者可以轻易猜测相邻的令牌值。UUID 的随机性让猜测变得不可行。

**2. userInfo 是快照，不是关联查询**

```java
private Map<String, String> userInfo;
// 存储 {"nickname": "小李", "deptId": "100"}
```

为什么不直接关联查询用户表？因为令牌校验是高频操作（每个请求都要执行），如果每次都关联查询用户表，数据库压力会很大。把用户的基本信息快照到令牌中，校验时不需要额外查询。

当然，这也意味着用户修改昵称后，已发放的令牌中存储的还是旧昵称。这是性能和实时性之间的权衡——昵称这种展示信息，短暂的不一致是可以接受的。

**3. 继承 TenantBaseDO 支持多租户**

令牌表继承了 `TenantBaseDO`，自动带有 `tenantId` 字段。这意味着不同租户的令牌是隔离的——租户 A 的管理员看不到租户 B 的会话。

### 刷新令牌：OAuth2RefreshTokenDO

刷新令牌（Refresh Token）用于在访问令牌过期后获取新的访问令牌，而不需要用户重新登录。对应数据库表 `system_oauth2_refresh_token`：

```java
public class OAuth2RefreshTokenDO extends TenantBaseDO {
    private Long id;                    // 数据库自增 ID
    private String refreshToken;        // 刷新令牌（UUID 字符串）
    private Long userId;                // 用户编号
    private Integer userType;           // 用户类型
    private String clientId;            // 客户端标识
    private List<String> scopes;        // 授权范围
    private LocalDateTime expiresTime;  // 过期时间
}
```

为什么刷新令牌和访问令牌是两个独立的实体？因为它们的生命周期不同：

```
  时间线：
  --------+------------------+------------------+---------->
          |                  |                  |
       登录时            accessToken          refreshToken
       同时创建          过期（如2小时）       过期（如30天）
          |                  |                  |
          |                  v                  |
          |            用 refreshToken          |
          |            换取新 accessToken        |
          |            (refreshToken 不变)      |
          |                                     |
          |                                     v
          |                               需要重新登录
```

访问令牌有效期短（如 2 小时），频繁更换；刷新令牌有效期长（如 30 天），相对稳定。这种"双令牌"设计的好处是：

- 访证令牌短命，即使泄露，影响窗口有限
- 刷新令牌长命，用户不需要频繁重新登录
- 刷新令牌泄露后，可以通过更换 refreshToken 来止损

## Redis 存储设计

### Key 的命名规则

```java
// RedisKeyConstants.java
String OAUTH2_ACCESS_TOKEN = "oauth2_access_token:%s";
```

实际的 Redis Key 格式为：`oauth2_access_token:{accessToken}`

例如：`oauth2_access_token:a1b2c3d4-e5f6-7890-abcd-ef1234567890`

为什么用 accessToken 本身作为 Key 的一部分？因为最频繁的操作是"根据 accessToken 查令牌信息"——每个请求都需要执行这个查询。用 accessToken 作 Key，可以直接命中，不需要额外的映射。

### Value 的结构

Redis 中存储的是 `OAuth2AccessTokenDO` 的 JSON 序列化：

```java
public void set(OAuth2AccessTokenDO accessTokenDO) {
    String redisKey = formatKey(accessTokenDO.getAccessToken());
    // 清理多余字段，避免缓存
    accessTokenDO.setUpdater(null).setUpdateTime(null)
                 .setCreateTime(null).setCreator(null).setDeleted(null);
    long time = LocalDateTimeUtil.between(
        LocalDateTime.now(), accessTokenDO.getExpiresTime(), ChronoUnit.SECONDS);
    if (time > 0) {
        stringRedisTemplate.opsForValue()
            .set(redisKey, JsonUtils.toJsonString(accessTokenDO), time, TimeUnit.SECONDS);
    }
}
```

注意两个细节：

1. **清理冗余字段**：`updater`、`updateTime`、`createTime`、`creator`、`deleted` 这些字段在令牌校验时不需要，清理掉可以减少 Redis 内存占用
2. **TTL 精确到秒**：Redis 的过期时间与令牌的 `expiresTime` 精确对齐，令牌过期后 Redis 自动清理，不需要额外的定时任务

### 查询策略：Redis 优先，MySQL 兜底

```java
public OAuth2AccessTokenDO getAccessToken(String accessToken) {
    // 优先从 Redis 中获取
    OAuth2AccessTokenDO accessTokenDO = oauth2AccessTokenRedisDAO.get(accessToken);
    if (accessTokenDO != null) {
        return accessTokenDO;
    }

    // 获取不到，从 MySQL 中获取访问令牌
    accessTokenDO = oauth2AccessTokenMapper.selectByAccessToken(accessToken);
    if (accessTokenDO == null) {
        // 特殊：从 MySQL 中获取刷新令牌
        // 原因：解决部分场景不方便刷新访问令牌场景
        // 例如说，积木报表只允许传递 token，不允许传递 refresh_token
        // 再例如说，前端 WebSocket 的 token 直接跟在 url 上
        OAuth2RefreshTokenDO refreshTokenDO =
            oauth2RefreshTokenMapper.selectByRefreshToken(accessToken);
        if (refreshTokenDO != null && !DateUtils.isExpired(refreshTokenDO.getExpiresTime())) {
            accessTokenDO = convertToAccessToken(refreshTokenDO);
        }
    }

    // 如果在 MySQL 存在，则往 Redis 中写入（回补缓存）
    if (accessTokenDO != null && !DateUtils.isExpired(accessTokenDO.getExpiresTime())) {
        oauth2AccessTokenRedisDAO.set(accessTokenDO);
    }
    return accessTokenDO;
}
```

这个方法体现了"三级查询"策略：

```
  查询令牌的流程：

  +--------+     命中     +--------+
  | Redis  | ----------> | 返回   |
  +--------+             +--------+
       |
       | 未命中
       v
  +--------+     命中     +--------+     回写 Redis
  | MySQL  | ----------> | 返回   | ---------> |
  | (access|             +--------+            |
  | token) |                                   |
  +--------+                                   v
       |                                   +--------+
       | 未命中                            | Redis  |
       v                                   +--------+
  +--------+     命中     +--------+
  | MySQL  | ----------> | 返回   |
  | (refresh             +--------+
  | token) |
  +--------+
       |
       | 未命中
       v
  +--------+
  | 返回   |
  | null   |
  +--------+
```

为什么要查 refreshToken？这是为了解决一些特殊场景——有些第三方组件（如积木报表）只允许传递一个 token 参数，无法同时传递 accessToken 和 refreshToken。在这种情况下，用户可以把 refreshToken 当作 accessToken 传入，系统会自动识别并处理。

## 令牌生命周期详解

### 创建令牌（登录时）

当用户登录成功后，`OAuth2TokenServiceImpl.createAccessToken()` 被调用：

```java
public OAuth2AccessTokenDO createAccessToken(Long userId, Integer userType,
                                               String clientId, List<String> scopes) {
    // 1. 校验客户端配置
    OAuth2ClientDO clientDO = oauth2ClientService.validOAuthClientFromCache(clientId);

    // 2. 创建刷新令牌（先创建，因为访问令牌需要关联它）
    OAuth2RefreshTokenDO refreshTokenDO =
        createOAuth2RefreshToken(userId, userType, clientDO, scopes);

    // 3. 创建访问令牌
    return createOAuth2AccessToken(refreshTokenDO, clientDO);
}
```

整个过程在一个事务中完成（`@Transactional`），确保刷新令牌和访问令牌要么同时创建成功，要么同时失败。

创建访问令牌的细节：

```java
private OAuth2AccessTokenDO createOAuth2AccessToken(
        OAuth2RefreshTokenDO refreshTokenDO, OAuth2ClientDO clientDO) {
    OAuth2AccessTokenDO accessTokenDO = new OAuth2AccessTokenDO()
        .setAccessToken(generateAccessToken())           // UUID
        .setUserId(refreshTokenDO.getUserId())
        .setUserType(refreshTokenDO.getUserType())
        .setUserInfo(buildUserInfo(...))                  // 用户快照
        .setClientId(clientDO.getClientId())
        .setScopes(refreshTokenDO.getScopes())
        .setRefreshToken(refreshTokenDO.getRefreshToken()) // 关联刷新令牌
        .setExpiresTime(LocalDateTime.now()
            .plusSeconds(clientDO.getAccessTokenValiditySeconds())); // 过期时间

    // 写入 MySQL
    oauth2AccessTokenMapper.insert(accessTokenDO);
    // 写入 Redis
    oauth2AccessTokenRedisDAO.set(accessTokenDO);
    return accessTokenDO;
}
```

创建后的数据流向：

```
  createAccessToken()
        |
        v
  +------------------+          +------------------+
  | OAuth2RefreshTokenDO|        | OAuth2AccessTokenDO|
  |                  |          |                  |
  | id: 1001         |          | id: 2001         |
  | refreshToken:    |<---------| refreshToken:    |
  |   "d4e5f6..."    |  关联    |   "d4e5f6..."    |
  | userId: 1        |          | accessToken:     |
  | userType: ADMIN  |          |   "a1b2c3..."    |
  | clientId: default|          | userId: 1        |
  | expiresTime:     |          | expiresTime:     |
  |   30天后         |          |   2小时后        |
  +------------------+          +------------------+
        |                              |
        v                              v
     MySQL only                  MySQL + Redis
  (刷新令牌不缓存)            (访问令牌双写)
```

为什么刷新令牌不写入 Redis？因为刷新令牌的使用频率很低（只在 accessToken 过期时使用），没必要占用 Redis 内存。

### 刷新令牌（续期时）

当 accessToken 过期但 refreshToken 还有效时，前端调用 `/system/auth/refresh-token` 接口：

```java
public OAuth2AccessTokenDO refreshAccessToken(String refreshToken, String clientId) {
    // 1. 查询刷新令牌
    OAuth2RefreshTokenDO refreshTokenDO =
        oauth2RefreshTokenMapper.selectByRefreshToken(refreshToken);
    if (refreshTokenDO == null) {
        throw exception("无效的刷新令牌");
    }

    // 2. 校验客户端匹配
    OAuth2ClientDO clientDO = oauth2ClientService.validOAuthClientFromCache(clientId);
    if (!clientId.equals(refreshTokenDO.getClientId())) {
        throw exception("刷新令牌的客户端编号不正确");
    }

    // 3. 删除旧的访问令牌（MySQL + Redis）
    List<OAuth2AccessTokenDO> accessTokenDOs =
        oauth2AccessTokenMapper.selectListByRefreshToken(refreshToken);
    if (CollUtil.isNotEmpty(accessTokenDOs)) {
        oauth2AccessTokenMapper.deleteByIds(...);
        oauth2AccessTokenRedisDAO.deleteList(...);
    }

    // 4. 检查刷新令牌是否过期
    if (DateUtils.isExpired(refreshTokenDO.getExpiresTime())) {
        oauth2RefreshTokenMapper.deleteById(refreshTokenDO.getId());
        throw exception("刷新令牌已过期");
    }

    // 5. 创建新的访问令牌（刷新令牌不变）
    return createOAuth2AccessToken(refreshTokenDO, clientDO);
}
```

刷新过程的要点：

```
  刷新前：
  +------------------+     +------------------+
  | RefreshToken     |     | AccessToken      |
  | "d4e5f6..."      |---->| "a1b2c3..."      |  (已过期)
  | 有效期：30天      |     | 有效期：2小时     |
  +------------------+     +------------------+

  刷新后：
  +------------------+     +------------------+
  | RefreshToken     |     | AccessToken      |
  | "d4e5f6..."      |---->| "x9y8z7..."      |  (新的)
  | 有效期：30天(不变)|     | 有效期：2小时     |
  +------------------+     +------------------+
                            旧的 "a1b2c3..." 已删除
```

刷新令牌保持不变，只有访问令牌被替换。这样前端只需要在 accessToken 过期时调用一次 refresh 接口，后续继续使用同一个 refreshToken。

### 撤回令牌（登出时）

用户点击"退出登录"，调用 `/system/auth/logout`：

```java
// AdminAuthServiceImpl
public void logout(String token, Integer logType) {
    OAuth2AccessTokenDO accessTokenDO = oauth2TokenService.removeAccessToken(token);
    if (accessTokenDO == null) {
        return; // 令牌不存在，可能已经过期
    }
    // 记录登出日志
    createLogoutLog(accessTokenDO.getUserId(), accessTokenDO.getUserType(), logType);
}

// OAuth2TokenServiceImpl
public OAuth2AccessTokenDO removeAccessToken(String accessToken) {
    // 删除访问令牌（MySQL + Redis）
    OAuth2AccessTokenDO accessTokenDO =
        oauth2AccessTokenMapper.selectByAccessToken(accessToken);
    oauth2AccessTokenMapper.deleteById(accessTokenDO.getId());
    oauth2AccessTokenRedisDAO.delete(accessToken);

    // 删除关联的刷新令牌（MySQL + Redis）
    oauth2RefreshTokenMapper.deleteByRefreshToken(accessTokenDO.getRefreshToken());
    oauth2AccessTokenRedisDAO.delete(accessTokenDO.getRefreshToken());

    return accessTokenDO;
}
```

登出操作会同时删除访问令牌和刷新令牌，彻底终止这个会话。

还有一个按用户删除的方法，用于"强制踢人"场景：

```java
public void removeAccessToken(Long userId, Integer userType) {
    List<OAuth2AccessTokenDO> accessTokens =
        oauth2AccessTokenMapper.selectListByUserIdAndUserType(userId, userType);
    accessTokens.forEach(accessToken -> {
        oauth2AccessTokenMapper.deleteById(accessToken.getId());
        oauth2AccessTokenRedisDAO.delete(accessToken.getAccessToken());
        oauth2RefreshTokenMapper.deleteByRefreshToken(accessToken.getRefreshToken());
        oauth2AccessTokenRedisDAO.delete(accessToken.getRefreshToken());
    });
}
```

这个方法会删除指定用户的所有令牌——无论他在多少个设备上登录，全部强制下线。

## 完整的令牌生命周期图

```
                        用户认证成功
                             |
                             v
              +------------------------------+
              | 创建 RefreshToken (MySQL)     |
              | 创建 AccessToken  (MySQL+Redis)|
              +------------------------------+
                             |
                    返回 accessToken
                    + refreshToken
                             |
         +-------------------+-------------------+
         |                                       |
         v                                       v
    携带 accessToken                        携带 refreshToken
    访问业务接口                            调用 refresh 接口
         |                                       |
         v                                       v
    +----------+                          +----------+
    | Redis 查询|                          | MySQL 查询|
    | 命中？    |                          | 刷新令牌  |
    +----+-----+                          +----+-----+
         |yes                                 |
         v                                    v
    +----------+                          +----------+
    | 检查过期  |                          | 删除旧    |
    +----+-----+                          | AccessToken|
         |                                | 创建新    |
         |                                | AccessToken|
         v                                +----+-----+
    +----------+                               |
    | 返回用户  |                               v
    | 信息     |                          返回新的
    +----------+                          accessToken
         |
         v
    用户登出 / 管理员踢人
         |
         v
    +------------------------------+
    | 删除 AccessToken (MySQL+Redis)|
    | 删除 RefreshToken (MySQL+Redis)|
    | 记录登出日志                   |
    +------------------------------+
```

## 过期清理机制

Redis 中的令牌会自动过期（TTL 与 expiresTime 对齐），但 MySQL 中的过期记录需要手动清理。

项目提供了定时清理方法：

```java
public Integer cleanAccessToken(Integer exceedDay, Integer deleteLimit) {
    LocalDateTime expireDate = LocalDateTime.now().minusDays(exceedDay);
    int count = 0;
    for (int i = 0; i < Short.MAX_VALUE; i++) {
        int deleteCount = oauth2AccessTokenMapper
            .deleteByExpiresTimeLt(expireDate, deleteLimit);
        count += deleteCount;
        if (deleteCount < deleteLimit) {
            break; // 没有更多需要删除的记录
        }
    }
    return count;
}
```

为什么用循环分批删除？因为一次性删除大量记录会导致数据库长时间锁表，影响正常业务。分批删除（每次 deleteLimit 条）可以避免这个问题。

## 与认证鉴权框架的对接

令牌创建完成后，第六组件的 `TokenAuthenticationFilter` 会在每个请求中验证令牌：

```
  请求进入
      |
      v
  TokenAuthenticationFilter
      |
      v
  从 Header 中提取 token
      |
      v
  OAuth2TokenService.checkAccessToken(token)
      |
      v
  +----------+     +----------+
  | Redis 查询| --> | MySQL 查询|
  +----------+     +----------+
      |
      v
  构建 LoginUser 对象
      |
      v
  注入 SecurityContext
      |
      v
  继续处理业务请求
```

这就是整个令牌体系的完整闭环：第七组件创建令牌，第六组件验证令牌，两者通过 `OAuth2TokenService` 接口协作。

## 思考题

1. 本项目中刷新令牌不存 Redis，只存 MySQL。如果刷新操作很频繁（比如多个用户同时刷新），会不会成为性能瓶颈？
2. `getAccessToken()` 方法中有一个特殊逻辑：如果 accessToken 在 MySQL 中找不到，会尝试用它去查 refreshToken。这种设计有什么潜在的安全风险？
3. 如果要实现"同一账号只允许在一个设备登录"（新登录踢掉旧会话），需要修改哪些方法？
4. 令牌清理的 `cleanAccessToken()` 方法为什么用 `for` 循环分批删除，而不是直接 `DELETE WHERE expires_time < ? LIMIT n`？
