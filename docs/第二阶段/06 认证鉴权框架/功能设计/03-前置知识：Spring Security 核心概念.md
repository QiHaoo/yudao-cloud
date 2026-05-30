# 03-前置知识：Spring Security 核心概念

## 为什么需要了解 Spring Security？

在上一章中，我们分析了认证鉴权的问题域和设计目标。你可能已经跃跃欲试，想要直接动手写代码了。但在动手之前，我们需要先理解一个关键问题：

**Spring Security 到底是什么？它在我们的项目中扮演什么角色？**

很多人对 Spring Security 的印象是"加个依赖就能保护接口"，或者"用来做登录的"。这些理解不算错，但太浅了。如果你不理解它的核心机制，后续在看 yudao 的源码时，会反复遇到困惑：

- 为什么 `TokenAuthenticationFilter` 要放在 `UsernamePasswordAuthenticationFilter` 前面？
- 为什么认证信息要塞进 `SecurityContextHolder` 而不是直接传参？
- 为什么 `@PreAuthorize` 能在方法执行前拦截？

这些问题的答案都藏在 Spring Security 的核心设计里。

---

## Spring Security 是什么？

### 一句话定义

Spring Security 是 Spring 生态中的**安全框架**，提供认证（Authentication）和授权（Authorization）的完整解决方案。

### 用类比理解

把你的 Web 应用想象成一栋办公楼：

```
访客（HTTP 请求）
    │
    ▼
┌─────────────────────────────────────────┐
│            办公楼大门（Web 服务器）         │
│                                         │
│  ┌─────────────────────────────────┐    │
│  │      保安团队（SecurityFilterChain）│   │
│  │                                 │    │
│  │  保安A：检查工牌（认证过滤器）     │    │
│  │  保安B：检查门禁卡（授权过滤器）   │    │
│  │  保安C：记录访客日志（日志过滤器）  │    │
│  │  ...                            │    │
│  └─────────────────────────────────┘    │
│                 │                        │
│                 ▼                        │
│  ┌─────────────────────────────────┐    │
│  │      各个办公室（Controller）      │    │
│  │                                 │    │
│  │  商品管理部                       │    │
│  │  订单管理部                       │    │
│  │  用户管理部                       │    │
│  └─────────────────────────────────┘    │
└─────────────────────────────────────────┘
```

- **SecurityFilterChain** 就是保安团队，由多个保安（过滤器）组成，每个人负责不同的检查任务
- **Authentication** 就是工牌，上面写着"你是谁、什么职位、有哪些权限"
- **SecurityContextHolder** 就是你口袋里的工牌夹，随时可以掏出来证明身份
- **AuthenticationManager** 就是保安队长，协调各个保安完成身份核验
- **AccessDecisionManager** 就是门禁系统，根据你的身份决定能进哪些房间

### Spring Security 解决的核心问题

| 问题 | 没有 Spring Security 时 | 有 Spring Security 时 |
|------|------------------------|----------------------|
| 认证 | 每个接口手动检查 Token | 过滤器链自动拦截和验证 |
| 授权 | 每个方法写 if-else 判断权限 | `@PreAuthorize` 声明式鉴权 |
| 会话管理 | 自己实现 Session/Token 管理 | 框架内置支持 |
| 安全防护 | 自己防 CSRF、XSS 等攻击 | 框架内置防护机制 |
| 上下文传递 | 用户信息到处传参 | `SecurityContextHolder` 自动管理 |

---

## 核心概念逐一讲解

### 概念一：SecurityFilterChain（过滤器链）

#### 什么是过滤器链？

过滤器链是 Spring Security 的**核心执行机制**。每个 HTTP 请求到达你的 Controller 之前，都要经过这条链上的所有过滤器，就像进机场要经过安检、边检、登机口检查一样。

```
HTTP 请求
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SecurityFilterChain                          │
│                                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │HeaderWrite│→│CsrfFilter│→│AuthFilter│→│ExceptionT│→ ...   │
│  │Filter    │  │          │  │          │  │ranslation│        │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Controller
```

#### Spring Security 内置的主要过滤器

Spring Security 默认注册了很多过滤器，每个都有特定职责：

| 过滤器 | 职责 | 执行顺序 |
|--------|------|---------|
| `SecurityContextPersistenceFilter` | 在请求开始时加载 SecurityContext，结束时清空 | 100 |
| `HeaderWriterFilter` | 写入安全相关的 HTTP 头（如 X-Content-Type-Options） | 200 |
| `CorsFilter` | 处理跨域请求 | 300 |
| `CsrfFilter` | 防止 CSRF 攻击 | 400 |
| `LogoutFilter` | 处理登出请求 | 500 |
| `UsernamePasswordAuthenticationFilter` | 处理表单登录 | 600 |
| `DefaultLoginPageGeneratingFilter` | 生成默认登录页面 | 700 |
| `BasicAuthenticationFilter` | 处理 HTTP Basic 认证 | 800 |
| `RequestCacheAwareFilter` | 缓存请求，登录成功后重定向 | 900 |
| `SecurityContextHolderAwareRequestFilter` | 包装请求，提供安全相关的方法 | 1000 |
| `AnonymousAuthenticationFilter` | 为未认证用户创建匿名身份 | 1100 |
| `SessionManagementFilter` | 管理会话 | 1200 |
| `ExceptionTranslationFilter` | 捕获安全异常，返回合适的错误响应 | 1300 |
| `FilterSecurityInterceptor` | 最终的授权决策 | 1400 |

**注意**：这些数字是排序值，不是实际顺序。Spring Security 使用 `Integer.MIN_VALUE` 到 `Integer.MAX_VALUE` 的范围来控制顺序。

#### 过滤器链的执行模型

关键点：**过滤器链是"洋葱模型"**——请求从外到内穿过所有过滤器，响应从内到外再穿过一遍。

```
请求进入
    │
    ▼
Filter 1 ─── doFilter() ───┐
    │                       │
    ▼                       │
Filter 2 ─── doFilter() ───┤
    │                       │
    ▼                       │
Filter 3 ─── doFilter() ───┤
    │                       │
    ▼                       │
Controller 执行 ◄───────────┘
    │
    ▼
Filter 3 ◄── 响应返回
    │
    ▼
Filter 2 ◄── 响应返回
    │
    ▼
Filter 1 ◄── 响应返回
    │
    ▼
响应发送给客户端
```

#### 本项目中的自定义过滤器位置

在 yudao 中，我们添加了一个自定义的 `TokenAuthenticationFilter`，它被放在 `UsernamePasswordAuthenticationFilter` 之前：

```java
// 来源：YudaoWebSecurityConfigurerAdapter.java
httpSecurity.addFilterBefore(authenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);
```

这意味着执行顺序是：

```
... → 其他过滤器 → TokenAuthenticationFilter → UsernamePasswordAuthenticationFilter → ...
```

为什么要放在前面？因为 yudao 使用 Token 认证，不使用表单登录。`TokenAuthenticationFilter` 必须先于 `UsernamePasswordAuthenticationFilter` 执行，否则请求会被后者拦截（因为它会尝试处理登录请求）。

---

### 概念二：Authentication（认证信息）

#### 什么是 Authentication？

`Authentication` 是 Spring Security 中表示"用户已认证信息"的接口。它就像一张工牌，上面记录着：

```java
public interface Authentication extends Principal, Serializable {
    // 用户的身份信息（通常是 UserDetails 或自定义对象）
    Object getPrincipal();
    // 用户的凭证（如密码）
    Object getCredentials();
    // 用户的权限列表
    Collection<? extends GrantedAuthority> getAuthorities();
    // 认证详情（如 IP 地址）
    Object getDetails();
    // 是否已认证
    boolean isAuthenticated();
}
```

#### UsernamePasswordAuthenticationToken

这是 Spring Security 中最常用的 `Authentication` 实现。虽然名字里有"UsernamePassword"，但它实际上可以存储任何类型的用户信息。

```
UsernamePasswordAuthenticationToken
    │
    ├── principal: Object        ← 存储用户信息（可以是任意对象）
    ├── credentials: Object      ← 存储凭证（通常为 null）
    ├── authorities: List        ← 存储权限列表
    ├── details: Object          ← 存储请求详情
    └── authenticated: boolean   ← 是否已认证
```

#### 本项目如何使用 Authentication

在 yudao 中，我们把 `LoginUser` 对象塞进 `Authentication.principal` 字段：

```java
// 来源：SecurityFrameworkUtils.java
private static Authentication buildAuthentication(LoginUser loginUser, HttpServletRequest request) {
    // 创建 UsernamePasswordAuthenticationToken 对象
    UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(
            loginUser, null, Collections.emptyList());
    authenticationToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
    return authenticationToken;
}
```

这里有几个值得注意的点：

1. **`principal` 是 `LoginUser` 对象**——不是默认的 `UserDetails`，而是我们自定义的用户信息载体
2. **`credentials` 是 `null`**——因为 Token 认证不需要密码
3. **`authorities` 是空列表**——权限校验走的是 `@ss` Bean，不是 Spring Security 内置的权限机制
4. **`authenticated` 设为 `true`**——Token 验证通过后才创建这个对象

#### 用类比理解

想象你去银行办业务：

- **Authentication** 就是你的身份证 + 银行卡 + 业务凭条
- **principal** 是身份证上的信息（姓名、身份证号）
- **credentials** 是你的密码（这里为 null，因为已经验证过了）
- **authorities** 是你被授权办理的业务类型
- **details** 是你在哪个柜台、几点来的

---

### 概念三：SecurityContextHolder（上下文持有者）

#### 什么是 SecurityContextHolder？

`SecurityContextHolder` 是 Spring Security 中最简单的组件，但也是最容易被误解的。它就做一件事：**存储当前请求的 `SecurityContext`，而 `SecurityContext` 里存着 `Authentication`**。

```
SecurityContextHolder（静态工具类）
    │
    └── ThreadLocal<SecurityContext>
            │
            └── SecurityContext
                    │
                    └── Authentication
                            │
                            ├── principal: LoginUser
                            ├── credentials: null
                            └── authorities: []
```

#### 为什么用 ThreadLocal？

核心原因：**一个请求对应一个线程，一个线程对应一个用户**。

```
请求A（用户张三）──→ 线程1 ──→ SecurityContextHolder 存储张三的信息
请求B（用户李四）──→ 线程2 ──→ SecurityContextHolder 存储李四的信息
请求C（用户王五）──→ 线程3 ──→ SecurityContextHolder 存储王五的信息
```

每个线程的 `SecurityContextHolder` 是独立的，互不干扰。这就是为什么你可以在任何地方调用 `SecurityContextHolder.getContext()` 获取当前用户——它自动返回当前线程对应的用户信息。

#### 用类比理解

把 `SecurityContextHolder` 想象成每个人口袋里的工牌夹：

- 你（线程）进入公司（处理请求），把工牌（Authentication）放进工牌夹（SecurityContextHolder）
- 任何时候需要证明身份，从工牌夹里掏出来就行
- 离开公司（请求处理完成），工牌夹自动清空

#### 本项目的定制：TransmittableThreadLocal

Spring Security 默认使用 `ThreadLocal` 存储上下文，但有一个问题：**当使用 `@Async` 异步执行时，会创建新线程，原线程的用户信息会丢失**。

yudao 解决了这个问题，用 `TransmittableThreadLocal` 替换了默认的 `ThreadLocal`：

```java
// 来源：TransmittableThreadLocalSecurityContextHolderStrategy.java
public class TransmittableThreadLocalSecurityContextHolderStrategy implements SecurityContextHolderStrategy {

    private static final ThreadLocal<SecurityContext> CONTEXT_HOLDER = new TransmittableThreadLocal<>();

    @Override
    public SecurityContext getContext() {
        SecurityContext ctx = CONTEXT_HOLDER.get();
        if (ctx == null) {
            ctx = createEmptyContext();
            CONTEXT_HOLDER.set(ctx);
        }
        return ctx;
    }

    @Override
    public void setContext(SecurityContext context) {
        Assert.notNull(context, "Only non-null SecurityContext instances are permitted");
        CONTEXT_HOLDER.set(context);
    }
}
```

`TransmittableThreadLocal` 是阿里巴巴开源的库，它能在父子线程之间自动传递数据，解决异步场景下的上下文丢失问题。

---

### 概念四：AuthenticationManager（认证管理器）

#### 什么是 AuthenticationManager？

`AuthenticationManager` 是认证的**协调者**。它不亲自验证用户，而是把任务分派给下面的 `AuthenticationProvider`。

```
AuthenticationManager（认证管理器）
    │
    ├── AuthenticationProvider 1：处理用户名密码认证
    │       │
    │       └── UserDetailsService 加载用户 → 比对密码 → 返回 Authentication
    │
    ├── AuthenticationProvider 2：处理 Token 认证
    │       │
    │       └── 验证 Token → 解析用户信息 → 返回 Authentication
    │
    └── AuthenticationProvider 3：处理第三方登录
            │
            └── 调用第三方 API → 获取用户信息 → 返回 Authentication
```

#### 认证流程

```java
// 伪代码：AuthenticationManager 的工作原理
public Authentication authenticate(Authentication authentication) {
    // 遍历所有 AuthenticationProvider
    for (AuthenticationProvider provider : providers) {
        // 找到支持当前认证类型的 Provider
        if (provider.supports(authentication.getClass())) {
            // 委托给 Provider 处理
            return provider.authenticate(authentication);
        }
    }
    throw new AuthenticationException("不支持的认证类型");
}
```

#### 用类比理解

`AuthenticationManager` 就像银行的**大堂经理**：

- 你来办业务（认证请求），大堂经理接待你
- 大堂经理不亲自办业务，而是根据你的需求，把你引导到对应的窗口：
  - 开户业务 → 柜台A（UsernamePasswordAuthenticationProvider）
  - 信用卡申请 → 柜台B（TokenAuthenticationProvider）
  - 理财业务 → 柜台C（第三方登录Provider）
- 每个柜台（Provider）处理完后，把结果（Authentication）返回给你

#### 本项目的配置

yudao 通过 `@Bean` 注册 `AuthenticationManager`，让它可以被注入使用：

```java
// 来源：YudaoWebSecurityConfigurerAdapter.java
@Bean
public AuthenticationManager authenticationManagerBean(AuthenticationConfiguration authenticationConfiguration) throws Exception {
    return authenticationConfiguration.getAuthenticationManager();
}
```

**注意**：在 yudao 的实际流程中，`AuthenticationManager` 并不是核心角色。因为 `TokenAuthenticationFilter` 直接调用 `OAuth2TokenCommonApi` 验证 Token，没有走 `AuthenticationManager` 的标准流程。这是设计上的简化——Token 认证的逻辑比用户名密码简单，不需要多层 Provider 的协调。

---

### 概念五：AccessDecisionManager（决策管理器）

#### 什么是 AccessDecisionManager？

如果说 `AuthenticationManager` 解决"你是谁"，那么 `AccessDecisionManager` 解决"你能做什么"。它根据用户的身份和权限，决定是否放行当前请求。

```
AccessDecisionManager（决策管理器）
    │
    ├── AccessDecisionVoter 1：基于角色的投票
    │       │
    │       └── 检查用户是否有要求的角色 → 投票：赞成/反对/弃权
    │
    ├── AccessDecisionVoter 2：基于权限的投票
    │       │
    │       └── 检查用户是否有要求的权限 → 投票：赞成/反对/弃权
    │
    └── AccessDecisionVoter 3：基于 IP 的投票
            │
            └── 检查请求 IP 是否在白名单 → 投票：赞成/反对/弃权
```

#### 决策策略

Spring Security 提供三种决策策略：

| 策略 | 含义 | 适用场景 |
|------|------|---------|
| `AffirmativeBased` | 有一个赞成就放行 | 宽松策略，适用于多条件满足其一即可 |
| `ConsensusBased` | 赞成票多于反对票就放行 | 多数决，适用于民主决策 |
| `UnanimousBased` | 全部赞成才放行 | 严格策略，适用于安全要求高的场景 |

默认策略是 `AffirmativeBased`——只要有一个人投赞成票就放行。

#### 用类比理解

`AccessDecisionManager` 就像公司的**审批流程**：

- 你提交一个采购申请（访问请求）
- 审批系统把申请发给各个审批人（Voter）：
  - 部门经理检查预算 → 赞成/反对
  - 财务检查合规性 → 赞成/反对
  - 法务检查风险 → 赞成/反对
- 审批系统根据投票结果做最终决定

#### 本项目的鉴权方式

yudao 没有大量使用 Spring Security 内置的 `AccessDecisionManager`，而是采用更灵活的 `@PreAuthorize` + `@ss` Bean 方式：

```java
// Controller 方法上的鉴权注解
@PreAuthorize("@ss.hasPermission('system:user:create')")
public CommonResult<Long> createUser(@Valid @RequestBody UserCreateReqVO reqVO) {
    // ...
}
```

这种方式的好处是：
1. **更直观**：权限标识直接写在方法上，一目了然
2. **更灵活**：`@ss` Bean 内部可以实现任意复杂的鉴权逻辑
3. **与业务解耦**：权限校验逻辑集中在 `SecurityFrameworkServiceImpl`，不在 Controller 里

---

### 概念六：UserDetailsService（用户详情服务）

#### 什么是 UserDetailsService？

`UserDetailsService` 是 Spring Security 提供的**用户加载接口**。它的唯一职责是：给定一个用户名，返回这个用户的详细信息。

```java
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

#### 标准流程中的角色

在传统的用户名密码认证流程中，`UserDetailsService` 的位置是：

```
用户提交用户名密码
    │
    ▼
AuthenticationManager
    │
    ▼
UsernamePasswordAuthenticationProvider
    │
    ├── 1. 调用 UserDetailsService.loadUserByUsername()
    │       │
    │       └── 返回 UserDetails（包含密码、权限等）
    │
    ├── 2. 比对密码
    │       │
    │       └── 用户提交的密码 vs 数据库中的密码
    │
    └── 3. 密码正确 → 返回 Authentication 对象
```

#### 用类比理解

`UserDetailsService` 就像银行的**客户信息查询系统**：

- 你告诉银行你的姓名（用户名）
- 银行查询系统（UserDetailsService）从数据库加载你的信息：
  - 身份证号、联系方式、账户余额、信用等级...
- 银行用这些信息验证你的身份（比对密码）和权限（信用等级）

#### 本项目为什么不使用 UserDetailsService？

yudao **没有使用** `UserDetailsService`，原因是：

1. **Token 认证不需要它**：Token 里已经包含了用户信息，不需要从数据库重新加载
2. **多用户体系**：yudao 支持管理员和会员两种用户类型，`UserDetailsService` 只能处理一种
3. **微服务架构**：用户信息在 system 服务，security 框架在其他服务，无法直接调用

替代方案是 `OAuth2TokenCommonApi`——通过 Feign RPC 调用 system 服务验证 Token：

```java
// 来源：TokenAuthenticationFilter.java
private LoginUser buildLoginUserByToken(String token, Integer userType) {
    // 校验访问令牌
    OAuth2AccessTokenCheckRespDTO accessToken = oauth2TokenApi.checkAccessToken(token).getCheckedData();
    if (accessToken == null) {
        return null;
    }
    // 构建登录用户
    return new LoginUser().setId(accessToken.getUserId()).setUserType(accessToken.getUserType())
            .setInfo(accessToken.getUserInfo())
            .setTenantId(accessToken.getTenantId()).setScopes(accessToken.getScopes())
            .setExpiresTime(accessToken.getExpiresTime());
}
```

---

## Spring Security 的请求处理流程

现在我们已经了解了所有核心概念，把它们串起来看一个请求的完整处理流程。

### 流程全景图

```
客户端发送请求
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        SecurityFilterChain                              │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  TokenAuthenticationFilter（yudao 自定义）                       │   │
│  │                                                                 │   │
│  │  1. 从 Header/Parameter 提取 Token                              │   │
│  │  2. 调用 OAuth2TokenCommonApi.checkAccessToken() 验证 Token     │   │
│  │  3. 验证通过 → 创建 LoginUser 对象                               │   │
│  │  4. 构建 UsernamePasswordAuthenticationToken                    │   │
│  │  5. 放入 SecurityContextHolder                                   │   │
│  │  6. 继续过滤链                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  其他内置过滤器...                                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ExceptionTranslationFilter                                     │   │
│  │                                                                 │   │
│  │  捕获后续过滤器抛出的安全异常，返回合适的 HTTP 响应                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  FilterSecurityInterceptor / AuthorizationFilter                 │   │
│  │                                                                 │   │
│  │  1. 从 SecurityContextHolder 获取当前用户                         │   │
│  │  2. 检查 URL 是否需要认证                                        │   │
│  │  3. 需要认证 → 检查用户是否已认证                                  │   │
│  │  4. 未认证 → 抛出 AuthenticationException                        │   │
│  │  5. 已认证 → 检查是否有权限访问该 URL                              │   │
│  │  6. 无权限 → 抛出 AccessDeniedException                          │   │
│  │  7. 有权限 → 放行                                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
└──────────────────────────────┼──────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                           Controller                                    │
│                                                                         │
│  @PreAuthorize("@ss.hasPermission('system:user:create')")              │
│  public CommonResult<Long> createUser(...) {                            │
│      // 方法级鉴权由 Spring Security 的 AOP 机制处理                      │
│      // 调用 @ss.hasPermission() 进行权限校验                            │
│      // 校验通过 → 执行方法体                                             │
│      // 校验失败 → 抛出 AccessDeniedException                            │
│  }                                                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 简化版流程（重点记忆版）

```
请求进入
    │
    ▼
TokenAuthenticationFilter ──→ 提取 Token ──→ 验证 Token ──→ 构建 LoginUser ──→ 存入 SecurityContextHolder
    │
    ▼
URL 权限检查 ──→ 是否需要认证？──→ 用户是否已认证？──→ 是否有权限？
    │
    ▼
Controller ──→ @PreAuthorize ──→ @ss.hasPermission() ──→ 权限校验
    │
    ▼
执行业务逻辑 ──→ 返回响应
```

---

## Spring Boot 自动配置 Spring Security 的原理

### 自动配置是什么？

Spring Boot 的核心特性之一是**自动配置**：引入依赖后，框架自动帮你配置好大部分东西，你只需要做少量定制。

对于 Spring Security，引入 `spring-boot-starter-security` 后，Spring Boot 会自动：

1. 创建 `SecurityFilterChain`
2. 注册所有内置过滤器
3. 启用表单登录和 HTTP Basic 认证
4. 要求所有请求都需要认证

这就是为什么你引入依赖后，访问任何页面都会跳到登录页——因为默认配置要求所有请求都必须认证。

### 自动配置的源码

Spring Security 的自动配置类是 `WebSecurityConfiguration`，核心逻辑：

```java
@Configuration
public class WebSecurityConfiguration {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests() // 配置 URL 权限
                .anyRequest().authenticated() // 所有请求都需要认证
            .and()
            .formLogin() // 启用表单登录
            .and()
            .httpBasic(); // 启用 HTTP Basic 认证
        return http.build();
    }
}
```

### 本项目如何覆盖自动配置

yudao 通过自定义 `SecurityFilterChain` 来覆盖默认配置：

```java
// 来源：YudaoWebSecurityConfigurerAdapter.java
@AutoConfiguration
@AutoConfigureOrder(-1) // 先于 Spring Security 自动配置
@EnableMethodSecurity(securedEnabled = true)
public class YudaoWebSecurityConfigurerAdapter {

    @Bean
    protected SecurityFilterChain filterChain(HttpSecurity httpSecurity) throws Exception {
        httpSecurity
            // 禁用 CSRF（因为不使用 Session）
            .csrf(AbstractHttpConfigurer::disable)
            // 设置为无状态（不创建 Session）
            .sessionManagement(c -> c.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            // 配置异常处理
            .exceptionHandling(c -> c.authenticationEntryPoint(authenticationEntryPoint)
                    .accessDeniedHandler(accessDeniedHandler));

        // 配置 URL 权限
        httpSecurity.authorizeHttpRequests(c -> c
            // 静态资源可匿名访问
            .requestMatchers(HttpMethod.GET, "/*.html", "/*.css", "/*.js").permitAll()
            // @PermitAll 标记的接口可匿名访问
            .requestMatchers(HttpMethod.GET, permitAllUrls.get(HttpMethod.GET).toArray(new String[0])).permitAll()
            // ...
        );

        // 添加自定义 Token 过滤器
        httpSecurity.addFilterBefore(authenticationTokenFilter, UsernamePasswordAuthenticationFilter.class);

        return httpSecurity.build();
    }
}
```

关键配置解读：

| 配置 | 默认值 | yudao 配置 | 原因 |
|------|--------|-----------|------|
| CSRF | 启用 | 禁用 | Token 认证不需要 CSRF 防护 |
| Session | 有状态 | 无状态 | 使用 Token 而非 Session |
| 登录方式 | 表单登录 | 自定义 Token | 前后端分离架构 |
| URL 权限 | 全部需要认证 | 按需配置 | 有些接口不需要认证 |

---

## 本项目对 Spring Security 的定制化

### 定制全景

yudao 对 Spring Security 进行了大幅"裁剪"，只保留了最核心的骨架，其他都替换成自己的实现：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Spring Security 标准组件                          │
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐         │
│  │UserDetailsServi │  │AuthenticationMa │  │AccessDecisionMa │         │
│  │ce               │  │nager            │  │nager            │         │
│  │（用户加载）      │  │（认证协调）      │  │（授权决策）      │         │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘         │
│           │                    │                    │                  │
│           │ 未使用             │ 辅助使用            │ 未使用            │
│           │                    │                    │                  │
│           ▼                    ▼                    ▼                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     yudao 自定义实现                              │   │
│  │                                                                 │   │
│  │  ┌──────────────────┐  ┌──────────────────┐                    │   │
│  │  │TokenAuthentication│  │SecurityFramework │                    │   │
│  │  │Filter            │  │ServiceImpl       │                    │   │
│  │  │（Token 认证过滤器）│  │（权限校验服务）   │                    │   │
│  │  └──────────────────┘  └──────────────────┘                    │   │
│  │                                                                 │   │
│  │  ┌──────────────────┐  ┌──────────────────┐                    │   │
│  │  │LoginUser         │  │OAuth2TokenCommon │                    │   │
│  │  │                  │  │Api               │                    │   │
│  │  │（用户信息载体）    │  │（Token 验证 RPC）│                    │   │
│  │  └──────────────────┘  └──────────────────┘                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

### 定制清单

| 组件 | 标准 Spring Security | yudao 定制 | 定制原因 |
|------|---------------------|-----------|---------|
| 认证方式 | `UserDetailsService` + 密码比对 | `OAuth2TokenCommonApi` + Token 验证 | 支持 Token 认证和微服务架构 |
| 用户信息载体 | `UserDetails` 接口 | `LoginUser` 自定义类 | 需要存储租户、用户类型等扩展信息 |
| 权限校验 | `AccessDecisionManager` + Voter | `@PreAuthorize` + `@ss` Bean | 更灵活，支持远程权限校验 |
| 上下文存储 | `ThreadLocal` | `TransmittableThreadLocal` | 支持 `@Async` 异步场景 |
| CSRF 防护 | 启用 | 禁用 | Token 认证不需要 CSRF |
| Session 管理 | 有状态 | 无状态 | 前后端分离，使用 Token |
| 登录方式 | 表单登录 | 自定义 `/login` 接口 | 支持多种登录方式（账号、手机、社交） |

### 保留了什么？

yudao 保留了 Spring Security 的以下核心能力：

1. **SecurityFilterChain**：过滤器链机制，用于拦截和处理请求
2. **SecurityContextHolder**：上下文存储机制，用于保存当前用户信息
3. **@PreAuthorize**：方法级鉴权注解，用于声明权限要求
4. **AuthenticationEntryPoint**：认证失败处理器，用于返回 401 响应
5. **AccessDeniedHandler**：权限不足处理器，用于返回 403 响应

这些是 Spring Security 的"骨架"，其他都是可替换的"血肉"。

---

## 思考题

### 思考题 1：过滤器顺序为什么重要？

`TokenAuthenticationFilter` 必须放在 `UsernamePasswordAuthenticationFilter` 之前。如果反过来会怎样？提示：想想 `UsernamePasswordAuthenticationFilter` 在什么情况下会拦截请求。

### 思考题 2：ThreadLocal 的局限性

Spring Security 默认使用 `ThreadLocal` 存储用户上下文，yudao 替换成了 `TransmittableThreadLocal`。除了 `@Async` 场景，还有哪些场景会导致 `ThreadLocal` 数据丢失？（提示：想想线程池、响应式编程）

### 思考题 3：为什么不直接使用 UserDetailsService？

yudao 选择自研 Token 验证机制，而不是使用 Spring Security 内置的 `UserDetailsService`。除了文中提到的原因，你能想到其他好处吗？（提示：想想缓存、性能、微服务间调用）

### 思考题 4：安全与便利的平衡

yudao 禁用了 CSRF 防护，这在 Token 认证场景下是合理的。但如果项目同时支持 Session 和 Token 两种认证方式，应该如何设计 CSRF 策略？

---

## 本章小结

本章我们从底向上讲解了 Spring Security 的核心概念：

1. **SecurityFilterChain**：过滤器链是请求处理的核心机制，yudao 添加了自定义的 `TokenAuthenticationFilter`
2. **Authentication**：认证信息的载体，yudao 使用 `UsernamePasswordAuthenticationToken` 存储 `LoginUser`
3. **SecurityContextHolder**：上下文存储，yudao 用 `TransmittableThreadLocal` 替换默认实现
4. **AuthenticationManager**：认证协调者，yudao 中不是核心角色，因为 Token 认证逻辑更简单
5. **AccessDecisionManager**：授权决策者，yudao 使用 `@PreAuthorize` + `@ss` Bean 替代
6. **UserDetailsService**：用户加载服务，yudao 使用 `OAuth2TokenCommonApi` 替代

下一章我们将讲解 OAuth2.0 协议和 Token 认证原理，这是 yudao 认证体系的另一个重要基础。
