# 02 - AdminAuthServiceImpl 登录实现

> **功能设计参考**：[04-核心设计：登录流程与安全保障](../功能设计/04-核心设计：登录流程与安全保障.md)

本章聚焦 `AdminAuthServiceImpl` 的核心实现——它是整个认证功能的编排中心，负责账号密码登录、短信登录、社交登录、登出和令牌刷新。读者完成后，能清楚地知道每种登录方式的完整调用链路。

---

## 1. 整体结构

`AdminAuthServiceImpl` 实现了 `AdminAuthService` 接口，依赖以下核心服务：

```
AdminAuthServiceImpl
├── AdminUserService        ← 用户查询、密码校验
├── LoginLogService         ← 登录日志记录
├── OAuth2TokenService      ← 令牌创建/刷新/删除
├── SocialUserService       ← 社交用户绑定关系
├── MemberService           ← C 端会员服务（登出日志中获取会员手机号）
├── SmsCodeApi              ← 短信验证码发送/校验
├── CaptchaService          ← 图形验证码校验（AJ-Captcha）
└── Validator               ← JSR-303 手动校验
```

> 源码路径：`yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/service/auth/AdminAuthServiceImpl.java`

---

## 2. authenticate()：三层身份校验

这是账号密码登录的底层校验方法，被 `login()` 调用。它执行三层递进式校验：

```java
@Override
public AdminUserDO authenticate(String username, String password) {
    final LoginLogTypeEnum logTypeEnum = LoginLogTypeEnum.LOGIN_USERNAME;
    // 校验账号是否存在
    AdminUserDO user = userService.getUserByUsername(username);
    if (user == null) {
        createLoginLog(null, username, logTypeEnum, LoginResultEnum.BAD_CREDENTIALS);
        throw exception(AUTH_LOGIN_BAD_CREDENTIALS);
    }
    if (!userService.isPasswordMatch(password, user.getPassword())) {
        createLoginLog(user.getId(), username, logTypeEnum, LoginResultEnum.BAD_CREDENTIALS);
        throw exception(AUTH_LOGIN_BAD_CREDENTIALS);
    }
    // 校验是否禁用
    if (CommonStatusEnum.isDisable(user.getStatus())) {
        createLoginLog(user.getId(), username, logTypeEnum, LoginResultEnum.USER_DISABLED);
        throw exception(AUTH_LOGIN_USER_DISABLED);
    }
    return user;
}
```

### 三层校验流程

```
请求进入 authenticate(username, password)
        │
        ▼
┌───────────────────────────────┐
│ 第一层：用户是否存在？         │
│ userService.getUserByUsername  │
└───────────┬───────────────────┘
            │ 不存在 → 记录日志(BAD_CREDENTIALS)，抛异常
            │ 存在 ↓
┌───────────▼───────────────────┐
│ 第二层：密码是否匹配？         │
│ userService.isPasswordMatch   │
│ (BCrypt 比对)                 │
└───────────┬───────────────────┘
            │ 不匹配 → 记录日志(BAD_CREDENTIALS)，抛异常
            │ 匹配 ↓
┌───────────▼───────────────────┐
│ 第三层：账号是否启用？         │
│ CommonStatusEnum.isDisable    │
└───────────┬───────────────────┘
            │ 禁用 → 记录日志(USER_DISABLED)，抛异常
            │ 启用 ↓
            ▼
        返回 AdminUserDO
```

**设计要点**：

1. **用户不存在和密码错误返回相同错误码** `AUTH_LOGIN_BAD_CREDENTIALS`。这是安全设计的通用做法——如果分别提示"用户不存在"和"密码错误"，攻击者可以枚举出哪些用户名是有效的
2. **每一层校验失败都会记录登录日志**，包含失败原因（`LoginResultEnum`）。安全审计时可以追溯"谁在什么时候尝试登录失败了多少次"
3. **用户不存在时 `userId` 为 null**，日志中只有 username，这是合理的——没有用户 ID 可用

**电商场景**：电商后台管理员尝试登录，如果输错了密码，系统记录一条日志："IP 为 192.168.1.100 的用户 admin 于 2024-01-01 10:30 登录失败（密码错误）"。连续多次失败后，运维人员可以发现暴力破解行为。

---

## 3. login()：账号密码登录完整流程

```java
@Override
@DataPermission(enable = false)
public AuthLoginRespVO login(AuthLoginReqVO reqVO) {
    // 校验验证码
    validateCaptcha(reqVO);

    // 使用账号密码，进行登录
    AdminUserDO user = authenticate(reqVO.getUsername(), reqVO.getPassword());

    // 如果 socialType 非空，说明需要绑定社交用户
    if (reqVO.getSocialType() != null) {
        socialUserService.bindSocialUser(new SocialUserBindReqDTO(user.getId(), getUserType().getValue(),
                reqVO.getSocialType(), reqVO.getSocialCode(), reqVO.getSocialState()));
    }
    // 创建 Token 令牌，记录登录日志
    return createTokenAfterLoginSuccess(user.getId(), reqVO.getUsername(), LoginLogTypeEnum.LOGIN_USERNAME);
}
```

### 完整流程图

```
POST /system/auth/login
        │
        ▼
┌───────────────────────────────┐
│ 1. @DataPermission(enable=false)│  ← 关闭数据权限，避免租户过滤干扰登录
└───────────┬───────────────────┘
            ▼
┌───────────────────────────────┐
│ 2. validateCaptcha(reqVO)     │  ← 校验图形验证码（可配置开关）
└───────────┬───────────────────┘
            ▼
┌───────────────────────────────┐
│ 3. authenticate(username, pwd)│  ← 三层身份校验（见上文）
└───────────┬───────────────────┘
            ▼
┌───────────────────────────────┐
│ 4. 绑定社交用户（可选）        │  ← 如果 socialType 非空
│ socialUserService.bindSocialUser│
└───────────┬───────────────────┘
            ▼
┌───────────────────────────────┐
│ 5. createTokenAfterLoginSuccess│  ← 创建令牌 + 记录日志
└───────────┬───────────────────┘
            ▼
        AuthLoginRespVO
```

**设计要点**：

- **`@DataPermission(enable = false)`**：登录方法上关闭数据权限。因为登录过程中需要查询用户信息，如果此时数据权限拦截器生效，可能会因为租户或部门过滤导致查不到用户，造成"明明用户存在却登录失败"的诡异问题
- **社交绑定是可选的**：只有当 `socialType` 不为空时才执行绑定。这是"登录并绑定"的场景——用户已经有一个账号，想关联微信/GitHub，下次可以直接社交登录
- **所有登录方式最终都走到 `createTokenAfterLoginSuccess()`**：这是一个模板方法，统一处理令牌创建和日志记录

### validateCaptcha() 实现

```java
@VisibleForTesting
void validateCaptcha(AuthLoginReqVO reqVO) {
    ResponseModel response = doValidateCaptcha(reqVO);
    if (!response.isSuccess()) {
        createLoginLog(null, reqVO.getUsername(), LoginLogTypeEnum.LOGIN_USERNAME,
                LoginResultEnum.CAPTCHA_CODE_ERROR);
        throw exception(AUTH_LOGIN_CAPTCHA_CODE_ERROR, response.getRepMsg());
    }
}

private ResponseModel doValidateCaptcha(CaptchaVerificationReqVO reqVO) {
    // 如果验证码关闭，则不进行校验
    if (!captchaEnable) {
        return ResponseModel.success();
    }
    ValidationUtils.validate(validator, reqVO, CaptchaVerificationReqVO.CodeEnableGroup.class);
    CaptchaVO captchaVO = new CaptchaVO();
    captchaVO.setCaptchaVerification(reqVO.getCaptchaVerification());
    return captchaService.verification(captchaVO);
}
```

验证码通过配置项 `yudao.captcha.enable`（默认 `true`）控制开关。关闭后 `doValidateCaptcha()` 直接返回成功，跳过校验。

---

## 4. createTokenAfterLoginSuccess()：模板方法

这是所有登录方式的汇聚点——账号密码登录、短信登录、社交登录、注册，最终都调用这个方法：

```java
private AuthLoginRespVO createTokenAfterLoginSuccess(Long userId, String username, LoginLogTypeEnum logType) {
    // 插入登陆日志
    createLoginLog(userId, username, logType, LoginResultEnum.SUCCESS);
    // 创建访问令牌
    OAuth2AccessTokenDO accessTokenDO = oauth2TokenService.createAccessToken(userId, getUserType().getValue(),
            OAuth2ClientConstants.CLIENT_ID_DEFAULT, null);
    // 构建返回结果
    return BeanUtils.toBean(accessTokenDO, AuthLoginRespVO.class);
}
```

**执行两件事**：

1. **记录登录成功日志**：调用 `createLoginLog()`，写入 `system_login_log` 表，包含用户 ID、登录类型、IP、User-Agent 等
2. **创建 OAuth2 令牌**：调用 `OAuth2TokenService.createAccessToken()`，创建访问令牌和刷新令牌（详见下一章）

**模板方法模式**：这里用到了设计模式中的模板方法模式。`createTokenAfterLoginSuccess()` 定义了"登录成功后的标准动作"，各个登录方法只负责自己的认证逻辑，认证成功后都委托给这个方法处理后续工作。

### createLoginLog() 实现

```java
private void createLoginLog(Long userId, String username,
                            LoginLogTypeEnum logTypeEnum, LoginResultEnum loginResult) {
    // 插入登录日志
    LoginLogCreateReqDTO reqDTO = new LoginLogCreateReqDTO();
    reqDTO.setLogType(logTypeEnum.getType());
    reqDTO.setTraceId(TracerUtils.getTraceId());
    reqDTO.setUserId(userId);
    reqDTO.setUserType(getUserType().getValue());
    reqDTO.setUsername(username);
    reqDTO.setUserAgent(ServletUtils.getUserAgent());
    reqDTO.setUserIp(ServletUtils.getClientIP());
    reqDTO.setResult(loginResult.getResult());
    loginLogService.createLoginLog(reqDTO);
    // 更新最后登录时间
    if (userId != null && Objects.equals(LoginResultEnum.SUCCESS.getResult(), loginResult.getResult())) {
        userService.updateUserLogin(userId, ServletUtils.getClientIP());
    }
}
```

注意：只有登录成功时（`loginResult == SUCCESS` 且 `userId != null`）才更新用户的最后登录时间和 IP。

---

## 5. smsLogin()：短信验证码登录

```java
@Override
public AuthLoginRespVO smsLogin(AuthSmsLoginReqVO reqVO) {
    // 校验验证码
    smsCodeApi.useSmsCode(AuthConvert.INSTANCE.convert(reqVO,
            SmsSceneEnum.ADMIN_MEMBER_LOGIN.getScene(), getClientIP())).checkError();

    // 获得用户信息
    AdminUserDO user = userService.getUserByMobile(reqVO.getMobile());
    if (user == null) {
        throw exception(USER_NOT_EXISTS);
    }

    // 创建 Token 令牌，记录登录日志
    return createTokenAfterLoginSuccess(user.getId(), reqVO.getMobile(), LoginLogTypeEnum.LOGIN_MOBILE);
}
```

### 流程图

```
POST /system/auth/sms-login
        │
        ▼
┌───────────────────────────────┐
│ 1. 校验短信验证码              │
│ smsCodeApi.useSmsCode()       │  ← 验证码一次性消费（用后即废）
└───────────┬───────────────────┘
            ▼
┌───────────────────────────────┐
│ 2. 按手机号查询用户            │
│ userService.getUserByMobile() │
└───────────┬───────────────────┘
            │ 不存在 → 抛 USER_NOT_EXISTS
            ▼
┌───────────────────────────────┐
│ 3. createTokenAfterLoginSuccess│
└───────────┬───────────────────┘
            ▼
        AuthLoginRespVO
```

**与账号密码登录的区别**：

| 对比项 | 账号密码登录 | 短信登录 |
|--------|------------|---------|
| 身份凭证 | username + password | mobile + 验证码 |
| 前置校验 | 图形验证码 | 无（短信本身就是验证码） |
| 用户查询 | 按用户名查 | 按手机号查 |
| 日志类型 | LOGIN_USERNAME | LOGIN_MOBILE |
| 错误处理 | BAD_CREDENTIALS / USER_DISABLED | USER_NOT_EXISTS |

注意短信登录不校验用户是否禁用——这是一个设计选择。如果需要，可以在 `if (user == null)` 之后增加禁用校验。

---

## 6. socialLogin()：社交登录

```java
@Override
public AuthLoginRespVO socialLogin(AuthSocialLoginReqVO reqVO) {
    // 使用 code 授权码，进行登录。然后，获得到绑定的用户编号
    SocialUserRespDTO socialUser = socialUserService.getSocialUserByCode(
            UserTypeEnum.ADMIN.getValue(), reqVO.getType(),
            reqVO.getCode(), reqVO.getState());
    if (socialUser == null || socialUser.getUserId() == null) {
        throw exception(AUTH_THIRD_LOGIN_NOT_BIND);
    }

    // 获得用户
    AdminUserDO user = userService.getUser(socialUser.getUserId());
    if (user == null) {
        throw exception(USER_NOT_EXISTS);
    }

    // 创建 Token 令牌，记录登录日志
    return createTokenAfterLoginSuccess(user.getId(), user.getUsername(), LoginLogTypeEnum.LOGIN_SOCIAL);
}
```

### 流程图

```
POST /system/auth/social-login
        │
        ▼
┌───────────────────────────────────────────┐
│ 1. socialUserService.getSocialUserByCode  │
│    用授权码换取社交平台用户信息            │
│    同时查找本地绑定关系                    │
└───────────┬───────────────────────────────┘
            │ 未绑定 → 抛 AUTH_THIRD_LOGIN_NOT_BIND
            ▼
┌───────────────────────────────────────────┐
│ 2. userService.getUser(socialUser.userId) │
└───────────┬───────────────────────────────┘
            │ 用户不存在 → 抛 USER_NOT_EXISTS
            ▼
┌───────────────────────────────────────────┐
│ 3. createTokenAfterLoginSuccess           │
└───────────┬───────────────────────────────┘
            ▼
        AuthLoginRespVO
```

**设计要点**：

- 社交登录的前提是用户已经完成了"绑定"操作（通过账号密码登录时附带 socialType，或单独的绑定接口）
- `getSocialUserByCode()` 内部做了两件事：(1) 用授权码向社交平台换取用户唯一标识；(2) 在本地 `system_social_user` 表中查找绑定关系
- 如果没有绑定关系（`socialUser.getUserId() == null`），返回 `AUTH_THIRD_LOGIN_NOT_BIND`，前端可以引导用户先绑定

**电商场景**：电商管理员之前在"个人设置"中绑定了自己的 GitHub 账号。现在打开后台登录页，点击"GitHub 登录"，浏览器跳转到 GitHub 授权页面，授权后带着 code 回调回来，后端用 code 换取 GitHub 用户 ID，查到绑定的本地用户，直接登录。

---

## 7. logout()：登出

```java
@Override
public void logout(String token, Integer logType) {
    // 删除访问令牌
    OAuth2AccessTokenDO accessTokenDO = oauth2TokenService.removeAccessToken(token);
    if (accessTokenDO == null) {
        return;
    }
    // 删除成功，则记录登出日志
    createLogoutLog(accessTokenDO.getUserId(), accessTokenDO.getUserType(), logType);
}
```

### 流程图

```
POST /system/auth/logout
        │
        ▼
┌───────────────────────────────────────┐
│ 1. 从请求中提取 token                 │
│ SecurityFrameworkUtils.obtainAuthorization│
└───────────┬───────────────────────────┘
            │ token 为空 → 直接返回 true
            ▼
┌───────────────────────────────────────┐
│ 2. authService.logout(token, logType) │
│    oauth2TokenService.removeAccessToken│
│    删除 MySQL + Redis 中的令牌        │
└───────────┬───────────────────────────┘
            │ token 不存在 → 静默返回
            ▼
┌───────────────────────────────────────┐
│ 3. createLogoutLog()                  │
│    记录登出日志                        │
└───────────────────────────────────────┘
```

**设计要点**：

- 登出的核心动作是**删除令牌**——同时删除 MySQL 中的记录和 Redis 中的缓存
- 如果 token 已经不存在（例如重复登出），不会报错，静默返回
- 登出后前端应该清除本地存储的 token

---

## 8. refreshToken()：令牌刷新

```java
@Override
public AuthLoginRespVO refreshToken(String refreshToken) {
    OAuth2AccessTokenDO accessTokenDO = oauth2TokenService.refreshAccessToken(
            refreshToken, OAuth2ClientConstants.CLIENT_ID_DEFAULT);
    return BeanUtils.toBean(accessTokenDO, AuthLoginRespVO.class);
}
```

这个方法非常简洁，直接委托给 `OAuth2TokenService.refreshAccessToken()`。详细的令牌轮换机制在下一章讲解。

**调用时机**：前端检测到 `accessToken` 过期（通过 HTTP 401 响应或本地 `expiresTime` 判断），携带 `refreshToken` 调用此接口获取新的 `accessToken`。

---

## 9. sendSmsCode()：发送短信验证码

```java
@Override
public void sendSmsCode(AuthSmsSendReqVO reqVO) {
    // 如果是重置密码场景，需要校验图形验证码是否正确
    if (Objects.equals(SmsSceneEnum.ADMIN_MEMBER_RESET_PASSWORD.getScene(), reqVO.getScene())) {
        ResponseModel response = doValidateCaptcha(reqVO);
        if (!response.isSuccess()) {
            throw exception(AUTH_REGISTER_CAPTCHA_CODE_ERROR, response.getRepMsg());
        }
    }

    // 登录场景，验证是否存在
    if (userService.getUserByMobile(reqVO.getMobile()) == null) {
        throw exception(AUTH_MOBILE_NOT_EXISTS);
    }
    // 发送验证码
    smsCodeApi.sendSmsCode(AuthConvert.INSTANCE.convert(reqVO).setCreateIp(getClientIP()));
}
```

**设计要点**：

- **重置密码场景需要图形验证码**：防止攻击者恶意发送短信轰炸目标手机
- **登录场景只校验手机号是否注册**：未注册的手机号不允许发送验证码
- `AuthConvert.INSTANCE.convert(reqVO)` 将 VO 转换为 `SmsCodeSendReqDTO`，同时设置 `createIp` 用于频率限制

---

## 10. register()：用户注册

```java
@Override
public AuthLoginRespVO register(AuthRegisterReqVO registerReqVO) {
    // 1. 校验验证码
    validateCaptcha(registerReqVO);

    // 2. 校验用户名是否已存在
    Long userId = userService.registerUser(registerReqVO);

    // 3. 创建 Token 令牌，记录登录日志
    return createTokenAfterLoginSuccess(userId, registerReqVO.getUsername(), LoginLogTypeEnum.LOGIN_USERNAME);
}
```

注册成功后自动登录——这是一种常见的 UX 设计，避免用户注册后还要再登录一次。

---

## 11. resetPassword()：重置密码

```java
@Override
@Transactional(rollbackFor = Exception.class)
public void resetPassword(AuthResetPasswordReqVO reqVO) {
    AdminUserDO userByMobile = userService.getUserByMobile(reqVO.getMobile());
    if (userByMobile == null) {
        throw exception(USER_MOBILE_NOT_EXISTS);
    }

    smsCodeApi.useSmsCode(new SmsCodeUseReqDTO()
            .setCode(reqVO.getCode())
            .setMobile(reqVO.getMobile())
            .setScene(SmsSceneEnum.ADMIN_MEMBER_RESET_PASSWORD.getScene())
            .setUsedIp(getClientIP())
    ).checkError();

    userService.updateUserPassword(userByMobile.getId(), reqVO.getPassword());
}
```

流程：校验手机号存在 → 校验短信验证码 → 更新密码。注意 `@Transactional` 保证操作的原子性。

---

## 12. 核心调用关系总结

```
Controller 层
├── login()              → authService.login()
├── smsLogin()           → authService.smsLogin()
├── socialQuickLogin()   → authService.socialLogin()
├── refreshToken()       → authService.refreshToken()
├── logout()             → authService.logout()
├── register()           → authService.register()
├── sendLoginSmsCode()   → authService.sendSmsCode()
└── resetPassword()      → authService.resetPassword()

AdminAuthServiceImpl
├── login()
│   ├── validateCaptcha()
│   ├── authenticate()
│   │   ├── userService.getUserByUsername()
│   │   ├── userService.isPasswordMatch()
│   │   └── CommonStatusEnum.isDisable()
│   ├── socialUserService.bindSocialUser()  (可选)
│   └── createTokenAfterLoginSuccess()
│       ├── createLoginLog()
│       └── oauth2TokenService.createAccessToken()
│
├── smsLogin()
│   ├── smsCodeApi.useSmsCode()
│   ├── userService.getUserByMobile()
│   └── createTokenAfterLoginSuccess()
│
├── socialLogin()
│   ├── socialUserService.getSocialUserByCode()
│   ├── userService.getUser()
│   └── createTokenAfterLoginSuccess()
│
├── logout()
│   ├── oauth2TokenService.removeAccessToken()
│   └── createLogoutLog()
│
└── refreshToken()
    └── oauth2TokenService.refreshAccessToken()
```

---

## 小结

| 方法 | 核心职责 | 关键依赖 |
|------|---------|---------|
| `authenticate()` | 三层身份校验（存在性、密码、状态） | AdminUserService |
| `login()` | 账号密码登录（含验证码、社交绑定） | CaptchaService, SocialUserService |
| `smsLogin()` | 短信验证码登录 | SmsCodeApi |
| `socialLogin()` | 社交授权码登录 | SocialUserService |
| `logout()` | 删除令牌 + 记录日志 | OAuth2TokenService |
| `refreshToken()` | 委托令牌服务刷新令牌 | OAuth2TokenService |
| `createTokenAfterLoginSuccess()` | 模板方法：创建令牌 + 记录日志 | OAuth2TokenService, LoginLogService |

下一章将深入 `OAuth2TokenServiceImpl`，讲解令牌的创建、刷新和存储机制。
