# 03 - OAuth2TokenServiceImpl 令牌管理

> **功能设计参考**：[03-核心设计：OAuth2 令牌体系](../功能设计/03-核心设计：OAuth2 令牌体系.md)

本章深入 `OAuth2TokenServiceImpl` 的实现——它是整个令牌体系的核心，负责令牌的创建、刷新、查询、校验和清理。读者完成后，能清楚地理解"双写 MySQL + Redis"的存储策略、令牌轮换机制，以及 UUID 令牌生成方案。

---

## 1. 整体结构

> 源码路径：`yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/service/oauth2/OAuth2TokenServiceImpl.java`

```
OAuth2TokenServiceImpl
├── OAuth2AccessTokenMapper    ← 访问令牌的 MySQL DAO
├── OAuth2RefreshTokenMapper   ← 刷新令牌的 MySQL DAO
├── OAuth2AccessTokenRedisDAO  ← 访问令牌的 Redis DAO
├── OAuth2ClientService        ← OAuth2 客户端配置（从缓存读取）
└── AdminUserService           ← 用户信息（懒加载，避免循环依赖）
```

**核心设计原则**：令牌同时写入 MySQL（持久化）和 Redis（高速缓存），查询时优先读 Redis，Redis 没有再查 MySQL 并回写 Redis。

---

## 2. createAccessToken()：创建访问令牌

### 2.1 入口方法

```java
@Override
@Transactional(rollbackFor = Exception.class)
public OAuth2AccessTokenDO createAccessToken(Long userId, Integer userType, String clientId, List<String> scopes) {
    OAuth2ClientDO clientDO = oauth2ClientService.validOAuthClientFromCache(clientId);
    // 创建刷新令牌
    OAuth2RefreshTokenDO refreshTokenDO = createOAuth2RefreshToken(userId, userType, clientDO, scopes);
    // 创建访问令牌
    return createOAuth2AccessToken(refreshTokenDO, clientDO);
}
```

**关键点**：

- `@Transactional` 保证刷新令牌和访问令牌的创建在同一个事务中——要么都成功，要么都回滚
- 先创建刷新令牌，再创建访问令牌。因为访问令牌需要关联刷新令牌的 `refreshToken` 值
- `validOAuthClientFromCache()` 从缓存中获取客户端配置，包含访问令牌和刷新令牌的有效期

### 2.2 创建刷新令牌

```java
private OAuth2RefreshTokenDO createOAuth2RefreshToken(Long userId, Integer userType,
                                                       OAuth2ClientDO clientDO, List<String> scopes) {
    OAuth2RefreshTokenDO refreshToken = new OAuth2RefreshTokenDO()
            .setRefreshToken(generateRefreshToken())
            .setUserId(userId).setUserType(userType)
            .setClientId(clientDO.getClientId()).setScopes(scopes)
            .setExpiresTime(LocalDateTime.now().plusSeconds(clientDO.getRefreshTokenValiditySeconds()));
    oauth2RefreshTokenMapper.insert(refreshToken);
    return refreshToken;
}
```

**刷新令牌特点**：
- 有效期较长（通常 7-30 天），由客户端配置 `refreshTokenValiditySeconds` 控制
- 只写 MySQL，不写 Redis。因为刷新令牌的查询频率很低（只在 access_token 过期时才用）
- 使用 UUID 作为令牌值（`generateRefreshToken()` → `IdUtil.fastSimpleUUID()`）

### 2.3 创建访问令牌

```java
private OAuth2AccessTokenDO createOAuth2AccessToken(OAuth2RefreshTokenDO refreshTokenDO,
                                                     OAuth2ClientDO clientDO) {
    OAuth2AccessTokenDO accessTokenDO = new OAuth2AccessTokenDO()
            .setAccessToken(generateAccessToken())
            .setUserId(refreshTokenDO.getUserId()).setUserType(refreshTokenDO.getUserType())
            .setUserInfo(buildUserInfo(refreshTokenDO.getUserId(), refreshTokenDO.getUserType()))
            .setClientId(clientDO.getClientId()).setScopes(refreshTokenDO.getScopes())
            .setRefreshToken(refreshTokenDO.getRefreshToken())
            .setExpiresTime(LocalDateTime.now().plusSeconds(clientDO.getAccessTokenValiditySeconds()));
    // 优先从 refreshToken 获取租户编号，避免 ThreadLocal 被污染时导致 tenantId 为 null
    Long tenantId = refreshTokenDO.getTenantId();
    if (tenantId == null) {
        tenantId = TenantContextHolder.getTenantId();
    }
    accessTokenDO.setTenantId(tenantId);
    oauth2AccessTokenMapper.insert(accessTokenDO);
    // 记录到 Redis 中
    oauth2AccessTokenRedisDAO.set(accessTokenDO);
    return accessTokenDO;
}
```

**访问令牌特点**：
- 有效期较短（通常 2 小时），由客户端配置 `accessTokenValiditySeconds` 控制
- **双写**：先写 MySQL（`oauth2AccessTokenMapper.insert`），再写 Redis（`oauth2AccessTokenRedisDAO.set`）
- 包含 `userInfo`（用户昵称、部门 ID），方便后续构建 `LoginUser` 对象时不需要再查数据库
- `refreshToken` 字段存储的是关联的刷新令牌值，用于令牌刷新时找到对应的刷新令牌

### 2.4 双写流程图

```
createAccessToken(userId, userType, clientId, scopes)
        │
        ▼
┌───────────────────────────────────┐
│ 1. 校验客户端配置                  │
│ oauth2ClientService               │
│   .validOAuthClientFromCache()    │
└───────────┬───────────────────────┘
            ▼
┌───────────────────────────────────┐
│ 2. 创建刷新令牌 (只写 MySQL)       │
│ generateRefreshToken() → UUID     │
│ oauth2RefreshTokenMapper.insert() │
└───────────┬───────────────────────┘
            ▼
┌───────────────────────────────────┐
│ 3. 创建访问令牌 (双写)             │
│ generateAccessToken() → UUID      │
│                                   │
│ ┌─────────────┐  ┌──────────────┐│
│ │   MySQL     │  │    Redis     ││
│ │ insert()    │→ │    set()     ││
│ └─────────────┘  └──────────────┘│
│                                   │
│ Redis key: oauth2_access_token:{} │
│ Redis TTL: accessToken有效期(秒)  │
└───────────────────────────────────┘
```

---

## 3. buildUserInfo()：构建用户信息快照

```java
private Map<String, String> buildUserInfo(Long userId, Integer userType) {
    if (userId == null || userId <= 0) {
        return Collections.emptyMap();
    }
    if (userType.equals(UserTypeEnum.ADMIN.getValue())) {
        AdminUserDO user = adminUserService.getUser(userId);
        return MapUtil.builder(LoginUser.INFO_KEY_NICKNAME, user.getNickname())
                .put(LoginUser.INFO_KEY_DEPT_ID, StrUtil.toStringOrNull(user.getDeptId())).build();
    } else if (userType.equals(UserTypeEnum.MEMBER.getValue())) {
        // 注意：目前 Member 暂时不读取，可以按需实现
        return Collections.emptyMap();
    }
    throw new IllegalArgumentException("未知用户类型：" + userType);
}
```

**设计要点**：

- 将用户的昵称和部门 ID 存入令牌的 `userInfo` 字段（JSON 格式）
- 这样在后续请求中，从令牌解析出 `LoginUser` 时可以直接获取这些信息，**避免每次都查数据库**
- 管理员类型存储 nickname 和 deptId，C 端会员类型暂不存储

**电商场景**：管理员登录后，系统把"张三"（昵称）和"运营部"（部门 ID）写入令牌。后续每次请求，网关从令牌中直接读取这些信息，不需要再查用户表，大幅减少数据库压力。

---

## 4. refreshAccessToken()：令牌轮换

```java
@Override
@Transactional(noRollbackFor = ServiceException.class)
public OAuth2AccessTokenDO refreshAccessToken(String refreshToken, String clientId) {
    // 查询刷新令牌
    OAuth2RefreshTokenDO refreshTokenDO = oauth2RefreshTokenMapper.selectByRefreshToken(refreshToken);
    if (refreshTokenDO == null) {
        throw exception0(GlobalErrorCodeConstants.BAD_REQUEST.getCode(), "无效的刷新令牌");
    }

    // 校验 Client 匹配
    OAuth2ClientDO clientDO = oauth2ClientService.validOAuthClientFromCache(clientId);
    if (ObjectUtil.notEqual(clientId, refreshTokenDO.getClientId())) {
        throw exception0(GlobalErrorCodeConstants.BAD_REQUEST.getCode(), "刷新令牌的客户端编号不正确");
    }

    // 移除相关的访问令牌
    List<OAuth2AccessTokenDO> accessTokenDOs = oauth2AccessTokenMapper.selectListByRefreshToken(refreshToken);
    if (CollUtil.isNotEmpty(accessTokenDOs)) {
        oauth2AccessTokenMapper.deleteByIds(convertSet(accessTokenDOs, OAuth2AccessTokenDO::getId));
        oauth2AccessTokenRedisDAO.deleteList(convertSet(accessTokenDOs, OAuth2AccessTokenDO::getAccessToken));
    }

    // 已过期的情况下，删除刷新令牌
    if (DateUtils.isExpired(refreshTokenDO.getExpiresTime())) {
        oauth2RefreshTokenMapper.deleteById(refreshTokenDO.getId());
        throw exception0(GlobalErrorCodeConstants.UNAUTHORIZED.getCode(), "刷新令牌已过期");
    }

    // 创建访问令牌
    return createOAuth2AccessToken(refreshTokenDO, clientDO);
}
```

### 流程图

```
refreshAccessToken(refreshToken, clientId)
        │
        ▼
┌───────────────────────────────────┐
│ 1. 查询刷新令牌                    │
│ oauth2RefreshTokenMapper          │
│   .selectByRefreshToken()         │
└───────────┬───────────────────────┘
            │ 不存在 → 抛 400 "无效的刷新令牌"
            ▼
┌───────────────────────────────────┐
│ 2. 校验 clientId 匹配             │
│ clientDO.clientId ==              │
│   refreshTokenDO.clientId ?       │
└───────────┬───────────────────────┘
            │ 不匹配 → 抛 400 "客户端编号不正确"
            ▼
┌───────────────────────────────────┐
│ 3. 移除旧的访问令牌 (MySQL + Redis)│
│ oauth2AccessTokenMapper            │
│   .selectListByRefreshToken()      │
│ oauth2AccessTokenMapper            │
│   .deleteByIds()                   │
│ oauth2AccessTokenRedisDAO          │
│   .deleteList()                    │
└───────────┬───────────────────────┘
            ▼
┌───────────────────────────────────┐
│ 4. 检查刷新令牌是否过期            │
│ DateUtils.isExpired()             │
└───────────┬───────────────────────┘
            │ 已过期 → 删除刷新令牌，抛 401 "刷新令牌已过期"
            ▼
┌───────────────────────────────────┐
│ 5. 创建新的访问令牌               │
│ createOAuth2AccessToken()         │
│ (MySQL + Redis 双写)              │
└───────────┬───────────────────────┘
            ▼
    新的 OAuth2AccessTokenDO
```

### 设计要点

1. **令牌轮换（Token Rotation）**：每次刷新都会**删除旧的访问令牌，创建新的访问令牌**。这是安全最佳实践——如果旧令牌被泄露，刷新后就失效了
2. **一个刷新令牌可以对应多个访问令牌**：`selectListByRefreshToken()` 返回的是列表。在正常场景下通常只有一个，但在网络重试等异常场景下可能有多个
3. **先删旧令牌，再检查过期**：即使刷新令牌已过期，也先把旧的访问令牌清理掉，避免残留脏数据
4. **`@Transactional(noRollbackFor = ServiceException.class)`**：即使抛出 `ServiceException`（如"刷新令牌已过期"），也不回滚前面的删除操作。因为删除旧令牌是有意义的，不应该回滚

**电商场景**：管理员正在操作电商后台，访问令牌过期了。前端自动用 refresh_token 调用刷新接口，获取新的 access_token。如果 refresh_token 也过期了（超过 7 天），则需要重新登录。

---

## 5. getAccessToken()：Redis 优先、MySQL 兜底

```java
@Override
public OAuth2AccessTokenDO getAccessToken(String accessToken) {
    // 优先从 Redis 中获取
    OAuth2AccessTokenDO accessTokenDO = oauth2AccessTokenRedisDAO.get(accessToken);
    if (accessTokenDO != null) {
        return accessTokenDO;
    }

    // 获取不到，从 MySQL 中获取访问令牌
    accessTokenDO = oauth2AccessTokenMapper.selectByAccessToken(accessToken);
    if (accessTokenDO == null) {
        // 特殊：从 MySQL 中获取刷新令牌。原因：解决部分场景不方便刷新访问令牌场景
        // 例如说，积木报表只允许传递 token，不允许传递 refresh_token，导致无法刷新访问令牌
        // 再例如说，前端 WebSocket 的 token 直接跟在 url 上，无法传递 refresh_token
        OAuth2RefreshTokenDO refreshTokenDO = oauth2RefreshTokenMapper.selectByRefreshToken(accessToken);
        if (refreshTokenDO != null && !DateUtils.isExpired(refreshTokenDO.getExpiresTime())) {
            accessTokenDO = convertToAccessToken(refreshTokenDO);
        }
    }

    // 如果在 MySQL 存在，则往 Redis 中写入
    if (accessTokenDO != null && !DateUtils.isExpired(accessTokenDO.getExpiresTime())) {
        oauth2AccessTokenRedisDAO.set(accessTokenDO);
    }
    return accessTokenDO;
}
```

### 三级查询流程

```
getAccessToken(accessToken)
        │
        ▼
┌───────────────────────────────┐
│ 第一级：Redis 查询             │
│ oauth2AccessTokenRedisDAO.get │
└───────────┬───────────────────┘
            │ 命中 → 直接返回（最快路径）
            │ 未命中 ↓
┌───────────▼───────────────────┐
│ 第二级：MySQL 访问令牌表       │
│ oauth2AccessTokenMapper       │
│   .selectByAccessToken()      │
└───────────┬───────────────────┘
            │ 命中 → 回写 Redis → 返回
            │ 未命中 ↓
┌───────────▼───────────────────┐
│ 第三级：MySQL 刷新令牌表       │
│ oauth2RefreshTokenMapper      │
│   .selectByRefreshToken()     │
│ （特殊兼容场景）               │
└───────────┬───────────────────┘
            │ 命中且未过期 → 转换为访问令牌格式 → 回写 Redis → 返回
            │ 未命中 → 返回 null
```

**设计要点**：

1. **Redis 优先**：Redis 的读取延迟在毫秒级，MySQL 在几毫秒到几十毫秒。对于每个请求都要校验 token 的场景，Redis 优先能显著降低延迟
2. **MySQL 兜底**：Redis 可能因重启、内存淘汰等原因丢失数据。MySQL 作为持久化层，保证令牌不会因 Redis 故障而丢失
3. **回写 Redis（Cache-Aside 模式）**：从 MySQL 查到后，回写到 Redis，下次请求就能命中 Redis。这是一种典型的缓存预热策略
4. **第三级查询：刷新令牌兼容**：注释中明确说了，某些场景（如积木报表、WebSocket）只能传递 token，无法传递 refresh_token。此时允许用 refresh_token 的值当作 access_token 来校验，提高兼容性

**Redis 存储结构**：

```java
// OAuth2AccessTokenRedisDAO.set()
public void set(OAuth2AccessTokenDO accessTokenDO) {
    String redisKey = formatKey(accessTokenDO.getAccessToken());
    // 清理多余字段，避免缓存
    accessTokenDO.setUpdater(null).setUpdateTime(null)
            .setCreateTime(null).setCreator(null).setDeleted(null);
    long time = LocalDateTimeUtil.between(LocalDateTime.now(),
            accessTokenDO.getExpiresTime(), ChronoUnit.SECONDS);
    if (time > 0) {
        stringRedisTemplate.opsForValue().set(redisKey,
                JsonUtils.toJsonString(accessTokenDO), time, TimeUnit.SECONDS);
    }
}
```

- Redis key 格式：`oauth2_access_token:{accessToken}`
- TTL：等于访问令牌的剩余有效期（秒）
- 写入前清理审计字段（`updater`、`updateTime` 等），减少 Redis 存储空间

---

## 6. checkAccessToken()：令牌校验

```java
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

这个方法在 `getAccessToken()` 的基础上增加了过期校验。被 Security 过滤器调用，用于每个请求的令牌校验。

---

## 7. removeAccessToken()：删除令牌

### 7.1 按令牌值删除（登出场景）

```java
@Override
@Transactional(rollbackFor = Exception.class)
public OAuth2AccessTokenDO removeAccessToken(String accessToken) {
    // 删除访问令牌
    OAuth2AccessTokenDO accessTokenDO = oauth2AccessTokenMapper.selectByAccessToken(accessToken);
    if (accessTokenDO == null) {
        return null;
    }
    oauth2AccessTokenMapper.deleteById(accessTokenDO.getId());
    oauth2AccessTokenRedisDAO.delete(accessToken);
    // 删除刷新令牌
    oauth2RefreshTokenMapper.deleteByRefreshToken(accessTokenDO.getRefreshToken());
    oauth2AccessTokenRedisDAO.delete(accessTokenDO.getRefreshToken());
    return accessTokenDO;
}
```

**删除范围**：同时删除访问令牌和关联的刷新令牌（MySQL + Redis），确保彻底清理。

### 7.2 按用户删除（管理员禁用用户场景）

```java
@Override
public void removeAccessToken(Long userId, Integer userType) {
    List<OAuth2AccessTokenDO> accessTokens = oauth2AccessTokenMapper
            .selectListByUserIdAndUserType(userId, userType);
    if (CollUtil.isEmpty(accessTokens)) {
        return;
    }
    accessTokens.forEach(accessToken -> {
        // 删除访问令牌
        oauth2AccessTokenMapper.deleteById(accessToken.getId());
        oauth2AccessTokenRedisDAO.delete(accessToken.getAccessToken());
        // 删除刷新令牌
        oauth2RefreshTokenMapper.deleteByRefreshToken(accessToken.getRefreshToken());
        oauth2AccessTokenRedisDAO.delete(accessToken.getRefreshToken());
    });
}
```

**使用场景**：管理员在后台禁用某个用户时，需要强制踢掉该用户的所有在线会话。此时按 userId 删除所有令牌。

---

## 8. UUID 令牌生成

```java
private static String generateAccessToken() {
    return IdUtil.fastSimpleUUID();
}

private static String generateRefreshToken() {
    return IdUtil.fastSimpleUUID();
}
```

**方案选择**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| **UUID（项目采用）** | 简单、无依赖、无碰撞风险 | 无序、不可携带信息、长度固定 32 位 |
| JWT | 自包含、可携带用户信息、无需服务端存储 | 无法主动失效、体积大、需要签名密钥管理 |
| 随机字符串 | 可控长度 | 需要校验唯一性 |

项目选择 UUID 的核心原因：**令牌需要服务端可控**。管理员禁用用户时，可以立即删除令牌使其失效。JWT 是无状态的，一旦签发就无法在过期前强制失效（除非引入黑名单，那还不如直接用有状态令牌）。

`IdUtil.fastSimpleUUID()` 来自 Hutool 工具库，生成的是去掉横线的 UUID（32 位十六进制字符串），如 `a1b2c3d4e5f6789012345678abcdef01`。

---

## 9. 清理过期令牌

```java
@Override
public Integer cleanRefreshToken(Integer exceedDay, Integer deleteLimit) {
    int count = 0;
    LocalDateTime expireDate = LocalDateTime.now().minusDays(exceedDay);
    for (int i = 0; i < Short.MAX_VALUE; i++) {
        int deleteCount = oauth2RefreshTokenMapper.deleteByExpiresTimeLt(expireDate, deleteLimit);
        count += deleteCount;
        if (deleteCount < deleteLimit) {
            break;
        }
    }
    return count;
}

@Override
public Integer cleanAccessToken(Integer exceedDay, Integer deleteLimit) {
    int count = 0;
    LocalDateTime expireDate = LocalDateTime.now().minusDays(exceedDay);
    for (int i = 0; i < Short.MAX_VALUE; i++) {
        int deleteCount = oauth2AccessTokenMapper.deleteByExpiresTimeLt(expireDate, deleteLimit);
        count += deleteCount;
        if (deleteCount < deleteLimit) {
            break;
        }
    }
    return count;
}
```

**设计要点**：

- **分批删除**：每次删除 `deleteLimit` 条（如 1000 条），避免一次性删除大量数据导致数据库长时间锁表
- **循环直到删完**：当某次删除的条数小于 `deleteLimit`，说明没有更多数据了，退出循环
- 这两个方法由定时任务调用，定期清理过期的令牌数据

---

## 10. 数据模型对照

### OAuth2AccessTokenDO

> 源码路径：`yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/oauth2/OAuth2AccessTokenDO.java`

```java
@TableName(value = "system_oauth2_access_token", autoResultMap = true)
@Data
@EqualsAndHashCode(callSuper = true)
public class OAuth2AccessTokenDO extends TenantBaseDO {

    @TableId
    private Long id;
    private String accessToken;        // 访问令牌（UUID）
    private String refreshToken;       // 刷新令牌（UUID）
    private Long userId;               // 用户编号
    private Integer userType;          // 用户类型（ADMIN/MEMBER）
    @TableField(typeHandler = JacksonTypeHandler.class)
    private Map<String, String> userInfo;  // 用户信息快照
    private String clientId;           // 客户端编号
    @TableField(typeHandler = JacksonTypeHandler.class)
    private List<String> scopes;       // 授权范围
    private LocalDateTime expiresTime; // 过期时间
}
```

### OAuth2RefreshTokenDO

> 源码路径：`yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/dal/dataobject/oauth2/OAuth2RefreshTokenDO.java`

```java
@TableName(value = "system_oauth2_refresh_token", autoResultMap = true)
@Data
public class OAuth2RefreshTokenDO extends TenantBaseDO {

    private Long id;
    private String refreshToken;       // 刷新令牌（UUID）
    private Long userId;               // 用户编号
    private Integer userType;          // 用户类型
    private String clientId;           // 客户端编号
    @TableField(typeHandler = JacksonTypeHandler.class)
    private List<String> scopes;       // 授权范围
    private LocalDateTime expiresTime; // 过期时间
}
```

**两者的区别**：

| 对比项 | OAuth2AccessTokenDO | OAuth2RefreshTokenDO |
|--------|--------------------|--------------------|
| 表名 | system_oauth2_access_token | system_oauth2_refresh_token |
| 有效期 | 短（如 2 小时） | 长（如 7 天） |
| 存储位置 | MySQL + Redis | 仅 MySQL |
| userInfo | 有（用户信息快照） | 无 |
| refreshToken 字段 | 有（关联刷新令牌） | 无 |
| 用途 | 每次请求携带，用于身份校验 | 仅在 access_token 过期时用于刷新 |

---

## 11. OAuth2AccessTokenRedisDAO 实现

> 源码路径：`yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/dal/redis/oauth2/OAuth2AccessTokenRedisDAO.java`

```java
@Repository
public class OAuth2AccessTokenRedisDAO {

    @Resource
    private StringRedisTemplate stringRedisTemplate;

    public OAuth2AccessTokenDO get(String accessToken) {
        String redisKey = formatKey(accessToken);
        return JsonUtils.parseObject(
                stringRedisTemplate.opsForValue().get(redisKey), OAuth2AccessTokenDO.class);
    }

    public void set(OAuth2AccessTokenDO accessTokenDO) {
        String redisKey = formatKey(accessTokenDO.getAccessToken());
        // 清理多余字段，避免缓存
        accessTokenDO.setUpdater(null).setUpdateTime(null)
                .setCreateTime(null).setCreator(null).setDeleted(null);
        long time = LocalDateTimeUtil.between(LocalDateTime.now(),
                accessTokenDO.getExpiresTime(), ChronoUnit.SECONDS);
        if (time > 0) {
            stringRedisTemplate.opsForValue().set(redisKey,
                    JsonUtils.toJsonString(accessTokenDO), time, TimeUnit.SECONDS);
        }
    }

    public void delete(String accessToken) {
        String redisKey = formatKey(accessToken);
        stringRedisTemplate.delete(redisKey);
    }

    public void deleteList(Collection<String> accessTokens) {
        List<String> redisKeys = CollectionUtils.convertList(
                accessTokens, OAuth2AccessTokenRedisDAO::formatKey);
        stringRedisTemplate.delete(redisKeys);
    }

    private static String formatKey(String accessToken) {
        return String.format(OAUTH2_ACCESS_TOKEN, accessToken);
    }
}
```

**实现要点**：

- 使用 `StringRedisTemplate` 存储 JSON 字符串，而非 `RedisTemplate` 存储对象。这样做的好处是：存储格式透明，便于调试；不受 Java 序列化框架限制
- `set()` 方法在写入前清理审计字段（`updater`、`updateTime`、`createTime`、`creator`、`deleted`），减少 JSON 体积
- TTL 使用 `ChronoUnit.SECONDS` 计算剩余秒数，确保 Redis 中的令牌与 MySQL 中的过期时间一致

---

## 12. 完整请求链路

以管理员登录后访问一个需要认证的接口为例：

```
1. 登录：POST /system/auth/login
   AdminAuthServiceImpl.login()
     → createTokenAfterLoginSuccess()
       → OAuth2TokenServiceImpl.createAccessToken()
         → createOAuth2RefreshToken()     [写 MySQL]
         → createOAuth2AccessToken()      [写 MySQL + 写 Redis]
       ← 返回 OAuth2AccessTokenDO
     ← 返回 AuthLoginRespVO (含 accessToken)

2. 访问接口：GET /system/user/profile
   Security 过滤器拦截请求
     → 从 Header 提取 accessToken
     → OAuth2TokenServiceImpl.checkAccessToken()
       → getAccessToken()
         → Redis 查询 → 命中 → 返回
       → 检查过期 → 未过期 → 返回 OAuth2AccessTokenDO
     → 构建 LoginUser 对象，放入 SecurityContext
   Controller 处理请求

3. 令牌过期：前端检测到 401
   POST /system/auth/refresh-token?refreshToken=xxx
   AdminAuthServiceImpl.refreshToken()
     → OAuth2TokenServiceImpl.refreshAccessToken()
       → 删除旧的访问令牌 [MySQL + Redis]
       → 创建新的访问令牌 [MySQL + Redis]
     ← 返回新的 AuthLoginRespVO

4. 登出：POST /system/auth/logout
   AdminAuthServiceImpl.logout()
     → OAuth2TokenServiceImpl.removeAccessToken()
       → 删除访问令牌 [MySQL + Redis]
       → 删除刷新令牌 [MySQL]
```

---

## 小结

| 方法 | 核心职责 | 存储操作 |
|------|---------|---------|
| `createAccessToken()` | 创建访问令牌 + 刷新令牌 | 刷新令牌→MySQL，访问令牌→MySQL+Redis |
| `refreshAccessToken()` | 令牌轮换：删旧建新 | 删旧访问令牌(MySQL+Redis)，建新访问令牌(MySQL+Redis) |
| `getAccessToken()` | 三级查询：Redis→MySQL访问令牌→MySQL刷新令牌 | 命中MySQL时回写Redis |
| `checkAccessToken()` | 校验令牌存在且未过期 | 只读 |
| `removeAccessToken(token)` | 登出：删除访问+刷新令牌 | 删除(MySQL+Redis) |
| `removeAccessToken(userId)` | 踢人：删除用户所有令牌 | 删除(MySQL+Redis) |
| `cleanRefreshToken/cleanAccessToken()` | 定时清理过期令牌 | 批量删除(MySQL) |
