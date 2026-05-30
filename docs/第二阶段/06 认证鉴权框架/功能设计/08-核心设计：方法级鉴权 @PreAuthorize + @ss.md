# 08-核心设计：方法级鉴权 @PreAuthorize + @ss

## 场景切入

认证解决了"你是谁"，但接下来更关键的问题是"你能做什么"。

一个电商后台有几百个接口，不同角色能访问的接口不同：
- 普通客服：只能查看订单，不能修改价格
- 仓库管理员：可以管理库存，不能查看财务报表
- 超级管理员：什么都能做

怎么在代码层面实现这种权限控制？最直观的做法是每个方法开头写 `if` 判断：

```java
public CommonResult<Long> deleteProduct(Long id) {
    // ❌ 反面示例：手动检查权限
    if (!permissionService.hasPermission("system:product:delete")) {
        return CommonResult.error(403, "没有权限");
    }
    productService.delete(id);
    return CommonResult.success();
}
```

这有两个问题：
1. **重复代码**：每个需要权限控制的方法都要写一遍
2. **容易遗漏**：新开发者可能忘记加权限检查

框架用 `@PreAuthorize` 注解 + `@ss` Bean 解决了这个问题。

## @PreAuthorize 注解

`@PreAuthorize` 是 Spring Security 提供的注解，在方法执行**之前**检查权限。它支持 SpEL（Spring Expression Language）表达式：

```java
@PreAuthorize("@ss.hasPermission('system:product:delete')")
public CommonResult<Long> deleteProduct(Long id) {
    productService.delete(id);
    return CommonResult.success();
}
```

含义：调用 `@ss` Bean 的 `hasPermission` 方法，传入权限标识 `system:product:delete`。如果返回 `true`，执行方法；如果返回 `false`，返回 403 错误。

### 为什么用 @ss 而不是 Spring Security 内置的 hasAuthority？

Spring Security 内置了 `hasAuthority('xxx')` 和 `hasRole('xxx')` 表达式，但它们依赖 `UserDetails.getAuthorities()` 返回的权限列表。本项目没有使用 `UserDetails` 体系，所以需要自定义鉴权逻辑。

`@ss` 是一个自定义的 Spring Bean，名称取自 Spring Security 的缩写，方便记忆和书写。

### 常见的鉴权表达式

```java
// 校验单个权限
@PreAuthorize("@ss.hasPermission('system:user:create')")

// 校验多个权限（任一即可）
@PreAuthorize("@ss.hasAnyPermissions('system:user:create', 'system:user:update')")

// 校验角色
@PreAuthorize("@ss.hasRole('admin')")

// 校验多个角色（任一即可）
@PreAuthorize("@ss.hasAnyRoles('admin', 'super_admin')")

// 校验 OAuth2 Scope
@PreAuthorize("@ss.hasScope('user:read')")

// 组合条件（AND）
@PreAuthorize("@ss.hasPermission('system:user:delete') and @ss.hasRole('admin')")
```

## @ss Bean 的实现：SecurityFrameworkServiceImpl

`@ss` 对应的 Bean 是 `SecurityFrameworkServiceImpl`，注册在 `YudaoSecurityAutoConfiguration` 中：

```java
@Bean("ss") // 使用 Spring Security 的缩写，方便使用
public SecurityFrameworkService securityFrameworkService(PermissionCommonApi permissionApi) {
    return new SecurityFrameworkServiceImpl(permissionApi);
}
```

### 核心方法：hasAnyPermissions

```java
@Override
@SneakyThrows
public boolean hasAnyPermissions(String... permissions) {
    // 1. 跨租户访问时，跳过权限校验
    if (skipPermissionCheck()) {
        return true;
    }

    // 2. 获取当前用户 ID
    Long userId = getLoginUserId();
    if (userId == null) {
        return false;
    }

    // 3. 查缓存 → 缓存未命中 → RPC 调用
    return hasAnyPermissionsCache.get(new KeyValue<>(userId, Arrays.asList(permissions)));
}
```

这段代码有三个关键设计：

#### 设计一：跨租户跳过权限校验

当管理员跨租户访问时（`visitTenantId != tenantId`），直接返回 `true`。为什么？因为跨租户的权限体系不同——A 租户的角色和权限在 B 租户中可能不存在。此时只能信任管理员的身份，不做细粒度权限校验。

#### 设计二：Guava Cache 缓存

权限校验结果被缓存 1 分钟。为什么？

- **性能**：每次 `@PreAuthorize` 都触发一次 RPC 调用，性能开销大
- **一致性**：1 分钟的缓存窗口意味着权限变更最多延迟 1 分钟生效，对于管理后台来说可接受
- **降级**：如果 system 服务暂时不可用，缓存中的旧结果仍可使用

```java
private final LoadingCache<KeyValue<Long, List<String>>, Boolean> hasAnyPermissionsCache = buildCache(
    Duration.ofMinutes(1L),  // 1 分钟过期
    new CacheLoader<KeyValue<Long, List<String>>, Boolean>() {
        @Override
        public Boolean load(KeyValue<Long, List<String>> key) {
            return permissionApi.hasAnyPermissions(
                key.getKey(),
                key.getValue().toArray(new String[0])
            ).getCheckedData();
        }
    });
```

缓存的 key 是 `(userId, permissions列表)` 的组合。这意味着：
- 用户 1 校验 `system:user:create` → 缓存
- 用户 1 校验 `system:user:delete` → 新的缓存条目
- 用户 2 校验 `system:user:create` → 新的缓存条目

#### 设计三：Feign RPC 调用

缓存未命中时，通过 `PermissionCommonApi`（Feign 客户端）调用 system 服务：

```java
// yudao-common 中定义的 RPC 接口
@FeignClient(name = "system-server")
public interface PermissionCommonApi {

    @GetMapping("/rpc-api/system/permission/has-any-permissions")
    CommonResult<Boolean> hasAnyPermissions(
        @RequestParam("userId") Long userId,
        @RequestParam("permissions") String... permissions);
}
```

system 服务收到请求后，执行真正的权限校验逻辑（查询用户角色 → 查询角色菜单 → 匹配权限标识）。

## 完整的鉴权流程

以 `@PreAuthorize("@ss.hasPermission('system:product:delete')")` 为例：

```
1. 用户请求 DELETE /admin-api/product/123
   │
   ▼
2. TokenAuthenticationFilter 认证通过
   LoginUser {id: 1, userType: 2, tenantId: 1}
   │
   ▼
3. Spring Security URL 级检查
   /admin-api/product/123 需要认证 → 已认证 ✓
   │
   ▼
4. Controller 方法被调用前，@PreAuthorize 触发
   │
   ▼
5. SpEL 表达式求值：@ss.hasPermission('system:product:delete')
   │
   ▼
6. SecurityFrameworkServiceImpl.hasAnyPermissions("system:product:delete")
   │
   ├── 6a. skipPermissionCheck() → false（未跨租户）
   │
   ├── 6b. getLoginUserId() → 1
   │
   ├── 6c. 查缓存：Key(1, ["system:product:delete"])
   │       ├── 缓存命中 → 返回缓存结果
   │       └── 缓存未命中 → 继续
   │
   └── 6d. permissionApi.hasAnyPermissions(1, "system:product:delete")
           │
           ▼
    7. system 服务处理 RPC 请求
       │
       ├── 7a. 查询用户 1 的角色列表 → [role_admin, role_warehouse]
       │
       ├── 7b. 遍历每个角色，查询其菜单权限
       │       role_admin → [system:product:*, system:order:*]
       │       role_warehouse → [system:stock:*, system:product:read]
       │
       ├── 7c. 匹配权限标识
       │       system:product:delete ∈ system:product:* → ✓ 匹配
       │
       └── 7d. 返回 true
   │
   ▼
8. @PreAuthorize 校验通过 → 执行方法体
   │
   ▼
9. 返回 CommonResult.success()
```

## 权限标识的设计规范

权限标识采用 `模块:资源:操作` 的三级格式：

```
system:user:create    → 系统模块-用户-创建
system:user:delete    → 系统模块-用户-删除
system:order:query    → 系统模块-订单-查询
pay:refund:create     → 支付模块-退款-创建
```

这种格式的优点：
- **层次清晰**：一眼看出属于哪个模块、什么资源、什么操作
- **支持通配符**：`system:product:*` 表示商品的所有操作
- **便于批量管理**：前端可以按模块分组展示权限树

## 超级管理员的特殊处理

在 `PermissionServiceImpl.hasAnyPermissions()` 中，超级管理员角色会被特殊处理：

```java
// 伪代码
public boolean hasAnyPermissions(Long userId, String... permissions) {
    // 超级管理员直接放行
    if (hasAnySuperAdmin(userId)) {
        return true;
    }
    // 正常权限校验...
}
```

超级管理员（角色 code 为 `super_admin`）不需要逐个检查权限，直接返回 `true`。这避免了超管需要被授予每一个新权限的维护成本。

## @PreAuthorize 的启用

`@PreAuthorize` 默认不生效，需要显式启用。框架通过 `@EnableMethodSecurity(securedEnabled = true)` 启用：

```java
@AutoConfiguration
@EnableMethodSecurity(securedEnabled = true)
public class YudaoWebSecurityConfigurerAdapter {
    // ...
}
```

`securedEnabled = true` 启用了 `@Secured` 注解的支持（虽然本项目主要用 `@PreAuthorize`）。`@PreAuthorize` 在 `@EnableMethodSecurity` 中默认启用。

## 与 URL 级权限的对比

| 维度 | URL 级权限 | 方法级权限（@PreAuthorize） |
|------|-----------|--------------------------|
| 粒度 | 按 URL 路径 | 按 Controller/Service 方法 |
| 灵活性 | 低（同一 URL 无法区分操作） | 高（可以精确到方法） |
| 代码耦合 | 低（配置在 SecurityFilterChain 中） | 高（注解在代码上） |
| 可读性 | 需要对照配置文件 | 权限声明就在方法上面，一目了然 |
| 运行时开销 | 低（URL 匹配很快） | 中等（SpEL 求值 + 缓存查询） |
| 适用场景 | 简单的 URL 放行/拦截 | 复杂的业务权限控制 |

**本项目的设计选择**：URL 级只用于免认证（`@PermitAll`），业务权限全部用方法级（`@PreAuthorize`）。

## 常见使用模式

### 模式一：Controller 层权限控制

```java
@PutMapping("/update")
@PreAuthorize("@ss.hasPermission('system:user:update')")
public CommonResult<Boolean> updateUser(@Valid @RequestBody UserUpdateReqVO reqVO) {
    userService.updateUser(reqVO);
    return success(true);
}
```

### 模式二：Service 层权限控制

有时候权限判断需要基于业务数据，不适合放在 Controller 层：

```java
// Controller
@PutMapping("/update")
public CommonResult<Boolean> updateOrder(@Valid @RequestBody OrderUpdateReqVO reqVO) {
    orderService.updateOrder(reqVO);
    return success(true);
}

// Service
public void updateOrder(OrderUpdateReqVO reqVO) {
    OrderDO order = orderMapper.selectById(reqVO.getId());
    // 基于订单状态的权限判断
    if (order.getStatus() == OrderStatusEnum.PAID.getStatus()) {
        // 已支付订单需要特殊权限
        if (!securityFrameworkService.hasPermission("system:order:update-paid")) {
            throw new ServiceException(ErrorCodeConstants.NO_PERMISSION);
        }
    }
    // 更新订单...
}
```

### 模式三：数据权限与功能权限结合

```java
@PreAuthorize("@ss.hasPermission('system:order:query')")
public CommonResult<List<OrderRespVO>> getOrderList(OrderPageReqVO reqVO) {
    // 功能权限通过 @PreAuthorize 控制
    // 数据权限通过 DataPermissionInterceptor 自动追加 WHERE 条件
    return success(orderService.getOrderList(reqVO));
}
```

## 思考题

1. **Guava Cache 的 1 分钟缓存过期时间是怎么确定的？** 如果改成 5 分钟会有什么问题？如果改成 10 秒呢？提示：考虑缓存命中率和权限变更的实时性。

2. **如果 system 服务宕机了，权限校验会怎样？** 缓存能兜底多久？有没有更好的降级策略？

3. **为什么权限校验通过 Feign RPC 调用 system 服务，而不是在本地数据库查询？** 在单体应用和微服务应用中，这个选择各有什么影响？

4. **`@PreAuthorize` 注解可以标记在 Service 方法上，但通常建议标记在 Controller 上。为什么？** 什么场景下应该标记在 Service 上？
