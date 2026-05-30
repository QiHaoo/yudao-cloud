# 04 - AuthController 端点实现

> **功能设计参考**：[04-核心设计：登录流程与安全保障](../功能设计/04-核心设计：登录流程与安全保障.md)、[06-配置设计与边界](../功能设计/06-配置设计与边界.md)

本章讲解 `AuthController` 的端点设计——它是认证功能的 HTTP 入口，定义了所有认证相关的 REST API。读者完成后，能清楚地知道每个端点的 URL、请求方式、权限设置，以及为什么这样设计。

---

## 1. 控制器总览

> 源码路径：`yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/controller/admin/auth/AuthController.java`

```java
@Tag(name = "管理后台 - 认证")
@RestController
@RequestMapping("/system/auth")
@Validated
@Slf4j
public class AuthController {

    @Resource
    private AdminAuthService authService;
    @Resource
    private AdminUserService userService;
    @Resource
    private RoleService roleService;
    @Resource
    private MenuService menuService;
    @Resource
    private PermissionService permissionService;
    @Resource
    private SocialClientService socialClientService;
    @Resource
    private SecurityProperties securityProperties;
    // ...
}
```

**基础信息**：
- 路径前缀：`/system/auth`
- 通过 `YudaoWebAutoConfiguration` 的 URL 前缀注入，实际访问路径为 `/admin-api/system/auth`
- Swagger 标签：`管理后台 - 认证`

### 端点一览表

| 端点 | HTTP 方法 | 路径 | 权限 | 说明 |
|------|----------|------|------|------|
| login | POST | /login | @PermitAll | 账号密码登录 |
| logout | POST | /logout | @PermitAll | 登出 |
| refreshToken | POST | /refresh-token | @PermitAll | 刷新令牌 |
| getPermissionInfo | GET | /get-permission-info | 需认证 | 获取权限信息 |
| register | POST | /register | @PermitAll | 用户注册 |
| smsLogin | POST | /sms-login | @PermitAll | 短信登录 |
| sendLoginSmsCode | POST | /send-sms-code | @PermitAll | 发送短信验证码 |
| resetPassword | POST | /reset-password | @PermitAll | 重置密码 |
| socialLogin | GET | /social-auth-redirect | @PermitAll | 社交授权跳转 |
| socialQuickLogin | POST | /social-login | @PermitAll | 社交登录 |

---

## 2. @PermitAll：为什么认证端点不需要认证

观察上面的端点表，会发现一个看似矛盾的现象：**认证相关的端点几乎都标注了 `@PermitAll`，即不需要登录就能访问**。

这是合理的——认证端点本身就是为"未登录用户"服务的：

- `/login`：用户还没登录，怎么可能要求先登录才能调用？
- `/logout`：用户要退出登录，此时 token 可能已过期，不应该强制要求有效 token
- `/refresh-token`：access_token 已过期，用户正是拿着 refresh_token 来换新 token 的
- `/register`：新用户注册，更不可能要求已登录

**`@PermitAll` 的作用**：告诉 Spring Security 的过滤器链，这些端点不需要经过认证校验，直接放行。

### 与 Spring Security 的交互

```
请求进入 → Spring Security FilterChain
                │
                ▼
        ┌───────────────────┐
        │ SecurityFilter    │
        │ 检查 @PermitAll?  │
        └───────┬───────────┘
                │ 是 → 直接放行到 Controller
                │ 否 → 检查 token → 构建 SecurityContext → 放行或拒绝
                ▼
        AuthController 处理请求
```

---

## 3. 登录端点详解

### 3.1 账号密码登录

```java
@PostMapping("/login")
@PermitAll
@Operation(summary = "使用账号密码登录")
public CommonResult<AuthLoginRespVO> login(@RequestBody @Valid AuthLoginReqVO reqVO) {
    return success(authService.login(reqVO));
}
```

**前端调用示例**：

```javascript
// POST /admin-api/system/auth/login
{
    "username": "admin",
    "password": "admin123",
    "captchaVerification": "xxx"  // 验证码结果（可选，取决于配置）
}
```

**响应示例**：

```json
{
    "code": 0,
    "data": {
        "userId": 1,
        "accessToken": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
        "refreshToken": "f9e8d7c6-b5a4-3210-fedc-ba9876543210",
        "expiresTime": "2024-01-01T12:00:00"
    }
}
```

### 3.2 短信验证码登录

```java
@PostMapping("/sms-login")
@PermitAll
@Operation(summary = "使用短信验证码登录")
public CommonResult<AuthLoginRespVO> smsLogin(@RequestBody @Valid AuthSmsLoginReqVO reqVO) {
    return success(authService.smsLogin(reqVO));
}
```

源码注释中提到了一个可选的限流配置：

```java
// 可按需开启限流：https://github.com/YunaiV/ruoyi-vue-pro/issues/851
// @RateLimiter(time = 60, count = 6, keyResolver = ExpressionRateLimiterKeyResolver.class,
//              keyArg = "#reqVO.mobile")
```

这是基于手机号的限流：每 60 秒最多 6 次短信登录尝试。当前被注释掉了，生产环境建议按需开启，防止短信轰炸。

### 3.3 发送短信验证码

```java
@PostMapping("/send-sms-code")
@PermitAll
@Operation(summary = "发送手机验证码")
public CommonResult<Boolean> sendLoginSmsCode(@RequestBody @Valid AuthSmsSendReqVO reqVO) {
    authService.sendSmsCode(reqVO);
    return success(true);
}
```

**调用流程**：前端先调用 `/send-sms-code` 发送验证码，用户收到短信后输入验证码，再调用 `/sms-login` 登录。

### 3.4 社交登录

社交登录分为两步：

**第一步：获取授权跳转 URL**

```java
@GetMapping("/social-auth-redirect")
@PermitAll
@Operation(summary = "社交授权的跳转")
@Parameters({
        @Parameter(name = "type", description = "社交类型", required = true),
        @Parameter(name = "redirectUri", description = "回调路径")
})
public CommonResult<String> socialLogin(@RequestParam("type") Integer type,
                                        @RequestParam("redirectUri") String redirectUri) {
    return success(socialClientService.getAuthorizeUrl(
            type, UserTypeEnum.ADMIN.getValue(), redirectUri));
}
```

前端拿到 URL 后跳转到社交平台（如 GitHub、微信）的授权页面。

**第二步：授权回调后登录**

```java
@PostMapping("/social-login")
@PermitAll
@Operation(summary = "社交快捷登录，使用 code 授权码",
           description = "适合未登录的用户，但是社交账号已绑定用户")
public CommonResult<AuthLoginRespVO> socialQuickLogin(@RequestBody @Valid AuthSocialLoginReqVO reqVO) {
    return success(authService.socialLogin(reqVO));
}
```

**完整社交登录流程**：

```
前端页面
   │
   ├── 1. GET /social-auth-redirect?type=10&redirectUri=/callback
   │      返回 GitHub 授权 URL
   │
   ├── 2. 浏览器跳转到 GitHub 授权页
   │      用户点击"授权"
   │
   ├── 3. GitHub 回调 redirectUri?code=xxx&state=yyy
   │
   └── 4. POST /social-login { type: 10, code: "xxx", state: "yyy" }
          返回 AuthLoginRespVO
```

---

## 4. get-permission-info：@DataPermission(enable = false)

```java
@GetMapping("/get-permission-info")
@Operation(summary = "获取登录用户的权限信息")
@DataPermission(enable = false) // 忽略数据权限，避免因为过滤，导致无法查询用户
public CommonResult<AuthPermissionInfoRespVO> getPermissionInfo() {
    // 1.1 获得用户信息
    AdminUserDO user = userService.getUser(getLoginUserId());
    if (user == null) {
        return success(null);
    }

    // 1.2 获得角色列表
    Set<Long> roleIds = permissionService.getUserRoleIdListByUserId(getLoginUserId());
    if (CollUtil.isEmpty(roleIds)) {
        return success(AuthConvert.INSTANCE.convert(user, Collections.emptyList(), Collections.emptyList()));
    }
    List<RoleDO> roles = roleService.getRoleList(roleIds);
    roles.removeIf(role -> !CommonStatusEnum.ENABLE.getStatus().equals(role.getStatus()));

    // 1.3 获得菜单列表
    Set<Long> menuIds = permissionService.getRoleMenuListByRoleId(convertSet(roles, RoleDO::getId));
    List<MenuDO> menuList = menuService.getMenuList(menuIds);
    menuList = menuService.filterDisableMenus(menuList);

    // 2. 拼接结果返回
    return success(AuthConvert.INSTANCE.convert(user, roles, menuList));
}
```

### 为什么需要 @DataPermission(enable = false)

源码注释写得很清楚：**忽略数据权限，避免因为过滤，导致无法查询用户**。

数据权限拦截器会根据当前登录用户的角色，自动在 SQL 中追加过滤条件（如 `WHERE dept_id IN (1, 2, 3)`）。但在 `get-permission-info` 这个场景下：

1. 当前用户刚刚登录，正在获取自己的权限信息
2. 如果数据权限拦截器生效，可能会过滤掉用户自己的数据
3. 这会导致"明明已登录，却查不到自己的信息"的诡异问题

参考链接：https://t.zsxq.com/LHnrp

### 获取权限信息的完整流程

```
GET /admin-api/system/auth/get-permission-info
        │
        ▼
┌───────────────────────────────────────────┐
│ 1. 获取当前登录用户 ID                     │
│ SecurityFrameworkUtils.getLoginUserId()   │
│ → 从 SecurityContext 中获取               │
└───────────┬───────────────────────────────┘
            ▼
┌───────────────────────────────────────────┐
│ 2. 查询用户信息                            │
│ userService.getUser(userId)               │
│ → 从 system_users 表查询                  │
└───────────┬───────────────────────────────┘
            │ 用户不存在 → 返回 null
            ▼
┌───────────────────────────────────────────┐
│ 3. 查询用户角色                            │
│ permissionService.getUserRoleIdListByUserId│
│ → 从 system_user_role 表查询              │
│ roleService.getRoleList(roleIds)          │
│ → 从 system_role 表查询                   │
│ → 过滤掉禁用的角色                         │
└───────────┬───────────────────────────────┘
            │ 无角色 → 返回空的权限信息
            ▼
┌───────────────────────────────────────────┐
│ 4. 查询角色菜单                            │
│ permissionService.getRoleMenuListByRoleId │
│ → 从 system_role_menu 表查询              │
│ menuService.getMenuList(menuIds)          │
│ → 从 system_menu 表查询                   │
│ → 过滤掉禁用的菜单                         │
└───────────┬───────────────────────────────┘
            ▼
┌───────────────────────────────────────────┐
│ 5. 构建响应                                │
│ AuthConvert.INSTANCE.convert(user, roles, menuList)│
│ → 提取角色标识（如 "admin"）               │
│ → 提取权限标识（如 "system:user:create"）  │
│ → 构建菜单树（父子关系）                   │
└───────────┬───────────────────────────────┘
            ▼
    AuthPermissionInfoRespVO
```

**设计要点**：

1. **这个端点需要认证**（没有 `@PermitAll`），因为它的目的是获取"当前登录用户"的权限信息
2. **关闭数据权限**：`@DataPermission(enable = false)` 确保查询不受数据权限拦截器干扰
3. **三层查询**：用户 → 角色 → 菜单，形成权限信息的完整链路
4. **过滤禁用项**：角色和菜单都有启用/禁用状态，只返回启用的
5. **菜单树在后端构建**：`AuthConvert.buildMenuTree()` 将扁平的菜单列表组装成树形结构，前端直接渲染

**电商场景**：管理员"张三"登录电商后台后，前端调用此接口获取权限信息。系统返回：
- 用户信息：张三、运营部
- 角色：`["shop_manager", "order_viewer"]`
- 权限：`["goods:create", "goods:update", "order:read", "order:export"]`
- 菜单树：商品管理 > 商品列表、商品分类；订单管理 > 订单列表、订单统计

前端根据这些信息动态渲染侧边栏菜单和操作按钮。

---

## 5. 登出端点

```java
@PostMapping("/logout")
@PermitAll
@Operation(summary = "登出系统")
public CommonResult<Boolean> logout(HttpServletRequest request) {
    String token = SecurityFrameworkUtils.obtainAuthorization(request,
            securityProperties.getTokenHeader(), securityProperties.getTokenParameter());
    if (StrUtil.isNotBlank(token)) {
        authService.logout(token, LoginLogTypeEnum.LOGOUT_SELF.getType());
    }
    return success(true);
}
```

**设计要点**：

- **token 为空时静默成功**：即使请求中没有 token（例如 token 已丢失），也返回 `true`。这是因为登出是"尽力而为"的操作，不应该因为 token 问题而报错
- **token 提取方式**：`SecurityFrameworkUtils.obtainAuthorization()` 从请求 Header 或 Parameter 中提取 token，具体位置由 `SecurityProperties` 配置（默认从 `Authorization` Header 提取）
- **日志类型**：`LoginLogTypeEnum.LOGOUT_SELF` 表示用户主动登出。还有 `LOGOUT_DELETE`（管理员踢人）、`LOGOUT_TIMEOUT`（超时登出）等类型

---

## 6. 刷新令牌端点

```java
@PostMapping("/refresh-token")
@PermitAll
@Operation(summary = "刷新令牌")
@Parameter(name = "refreshToken", description = "刷新令牌", required = true)
public CommonResult<AuthLoginRespVO> refreshToken(@RequestParam("refreshToken") String refreshToken) {
    return success(authService.refreshToken(refreshToken));
}
```

**设计要点**：

- 刷新令牌通过 `@RequestParam` 传递（query 参数），而非请求体。这是 REST 惯例——GET 类操作用 query 参数，POST 类操作用请求体
- 前端通常在 HTTP 响应拦截器中统一处理 401 错误：检测到 401 时，自动用 refresh_token 调用此端点获取新 token

---

## 7. 注册端点

```java
@PostMapping("/register")
@PermitAll
@Operation(summary = "注册用户")
public CommonResult<AuthLoginRespVO> register(@RequestBody @Valid AuthRegisterReqVO registerReqVO) {
    return success(authService.register(registerReqVO));
}
```

注册成功后直接返回登录令牌，实现"注册即登录"。

---

## 8. 重置密码端点

```java
@PostMapping("/reset-password")
@PermitAll
@Operation(summary = "重置密码")
public CommonResult<Boolean> resetPassword(@RequestBody @Valid AuthResetPasswordReqVO reqVO) {
    authService.resetPassword(reqVO);
    return success(true);
}
```

**前端调用流程**：
1. 用户输入手机号，点击"发送验证码"（调用 `/send-sms-code`，scene=重置密码）
2. 用户输入验证码和新密码
3. 调用 `/reset-password` 提交

---

## 9. 端点设计总结

### 权限矩阵

```
                ┌──────────────────────────────────────────┐
                │           Spring Security 过滤器          │
                └────────────┬─────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
        @PermitAll                     需要认证
        (无需 token)                  (需要有效 token)
              │                             │
    ┌─────────┼─────────┐                   │
    │         │         │                   │
  /login  /logout  /refresh-token    /get-permission-info
  /sms-login /register                (获取当前用户权限)
  /send-sms-code
  /reset-password
  /social-auth-redirect
  /social-login
```

### 前端调用时序

```
首次登录：
  /login 或 /sms-login 或 /social-login
      ↓
  存储 accessToken + refreshToken 到 localStorage
      ↓
  /get-permission-info
      ↓
  渲染菜单和按钮

后续请求：
  每个 API 请求 Header 携带 Authorization: Bearer {accessToken}
      ↓
  401? → /refresh-token
      ↓
  更新 accessToken
      ↓
  重试原请求

登出：
  /logout
      ↓
  清除 localStorage 中的 token
      ↓
  跳转到登录页
```

---

## 小结

| 端点 | 权限 | 核心行为 | 特殊处理 |
|------|------|---------|---------|
| `/login` | @PermitAll | 账号密码登录 | 验证码校验（可选） |
| `/sms-login` | @PermitAll | 短信登录 | 可选限流 |
| `/social-login` | @PermitAll | 社交登录 | 需要先绑定 |
| `/logout` | @PermitAll | 登出 | token 为空静默成功 |
| `/refresh-token` | @PermitAll | 刷新令牌 | query 参数传递 |
| `/register` | @PermitAll | 注册 | 注册即登录 |
| `/send-sms-code` | @PermitAll | 发送验证码 | 重置密码需图形验证码 |
| `/reset-password` | @PermitAll | 重置密码 | 短信验证码确认身份 |
| `/social-auth-redirect` | @PermitAll | 获取授权 URL | 返回跳转地址 |
| `/get-permission-info` | 需认证 | 获取权限信息 | @DataPermission(enable=false) |
