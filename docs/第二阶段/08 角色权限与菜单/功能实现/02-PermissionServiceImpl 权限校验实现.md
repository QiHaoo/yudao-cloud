# PermissionServiceImpl 权限校验实现

> 本文档对应源码：`cn.iocoder.yudao.module.system.service.permission.PermissionServiceImpl`

## 一、PermissionService 接口全景

`PermissionService` 是整个权限体系的核心枢纽，承担三大职责：

```
PermissionService
├── 权限校验
│   ├── hasAnyPermissions(userId, permissions)  → 判断用户是否有指定权限
│   └── hasAnyRoles(userId, roles)              → 判断用户是否有指定角色
├── 权限分配
│   ├── assignRoleMenu(roleId, menuIds)         → 给角色分配菜单权限
│   ├── assignUserRole(userId, roleIds)         → 给用户分配角色
│   └── assignRoleDataScope(roleId, dataScope, deptIds) → 设置角色数据范围
└── 权限查询
    ├── getRoleMenuListByRoleId(roleIds)         → 查询角色拥有的菜单
    ├── getUserRoleIdListByUserId(userId)         → 查询用户拥有的角色
    └── getDeptDataPermission(userId)             → 计算用户的部门数据权限
```

---

## 二、hasAnyPermissions —— 权限校验核心

这是整个权限体系中调用频率最高的方法。每次用户访问受保护的 API 接口时，Spring Security 的 `@PreAuthorize` 注解最终会调用到这个方法。

### 2.1 整体流程

```
请求进入 Controller
        │
        ▼
@PreAuthorize("@ss.hasPermission('system:order:create')")
        │
        ▼
PermissionServiceImpl.hasAnyPermissions(userId, "system:order:create")
        │
        ├─ 1. permissions 为空？ → true（有权限）
        │
        ├─ 2. 获取用户已启用的角色列表（带缓存）
        │      getEnableUserRoleListByUserIdFromCache(userId)
        │      └─ 角色列表为空？ → false（无权限）
        │
        ├─ 3. 遍历每个 permission，调用 hasAnyPermission(roles, permission)
        │      │
        │      ├─ 3a. 通过 permission 查找对应的 menuId 列表（带缓存）
        │      │      menuService.getMenuIdListByPermissionFromCache(permission)
        │      │      └─ 为空？ → false（权限标识不存在，严格模式拒绝）
        │      │
        │      └─ 3b. 对每个 menuId，获取拥有该菜单的角色 ID 集合（带缓存）
        │             getSelf().getMenuRoleIdListByMenuIdFromCache(menuId)
        │             └─ 与用户角色有交集？ → true（有权限）
        │
        └─ 4. 以上都不满足，检查是否为超管
               roleService.hasAnySuperAdmin(roleIds)
               └─ 是超管？ → true（拥有所有权限）
```

### 2.2 源码解析

```java
// PermissionServiceImpl.java
@Override
public boolean hasAnyPermissions(Long userId, String... permissions) {
    // 1. 如果为空，说明已经有权限
    if (ArrayUtil.isEmpty(permissions)) {
        return true;
    }

    // 2. 获得当前登录的角色。如果为空，说明没有权限
    List<RoleDO> roles = getEnableUserRoleListByUserIdFromCache(userId);
    if (CollUtil.isEmpty(roles)) {
        return false;
    }

    // 3. 遍历判断每个权限，如果有一满足，说明有权限
    for (String permission : permissions) {
        if (hasAnyPermission(roles, permission)) {
            return true;
        }
    }

    // 4. 如果是超管，也说明有权限
    return roleService.hasAnySuperAdmin(convertSet(roles, RoleDO::getId));
}
```

**关键设计决策**：

- **超管兜底**：即使角色没有被显式分配某个菜单权限，超管（`super_admin`）也能通过所有校验。这避免了每次新增菜单都要手动给超管分配的麻烦。
- **严格模式**：如果一个 permission 字符串在数据库中找不到对应的 Menu，`hasAnyPermission()` 会返回 `false`。这意味着如果菜单被删除或者 permission 写错了，访问会被拒绝，而不是错误地放行。
- **任一满足**：`hasAnyPermissions` 中的 `Any` 表示只要传入的多个权限中有任意一个匹配即可通过。

### 2.3 getEnableUserRoleListByUserIdFromCache —— 获取用户的有效角色

```java
@VisibleForTesting
List<RoleDO> getEnableUserRoleListByUserIdFromCache(Long userId) {
    // 1. 获得用户拥有的角色编号（带 Redis 缓存）
    Set<Long> roleIds = getSelf().getUserRoleIdListByUserIdFromCache(userId);
    // 2. 获得角色数组，并移除被禁用的
    List<RoleDO> roles = roleService.getRoleListFromCache(roleIds);
    roles.removeIf(role -> !CommonStatusEnum.ENABLE.getStatus().equals(role.getStatus()));
    return roles;
}
```

这里注意 `getSelf()` 的用法——通过 `SpringUtil.getBean(getClass())` 获取当前类的代理对象，确保 `@Cacheable` 等 AOP 注解能生效。这是一个经典的 Spring AOP 自调用问题的解决方案。

---

## 三、hasAnyRoles —— 角色校验

```java
@Override
public boolean hasAnyRoles(Long userId, String... roles) {
    if (ArrayUtil.isEmpty(roles)) {
        return true;
    }
    // 获取用户已启用的角色
    List<RoleDO> roleList = getEnableUserRoleListByUserIdFromCache(userId);
    if (CollUtil.isEmpty(roleList)) {
        return false;
    }
    // 判断角色标识是否有交集
    Set<String> userRoles = convertSet(roleList, RoleDO::getCode);
    return CollUtil.containsAny(userRoles, Sets.newHashSet(roles));
}
```

角色校验比权限校验简单，只需比较角色标识字符串。使用场景是 `@PreAuthorize("@ss.hasRole('admin')")`。

---

## 四、assignRoleMenu —— 角色菜单分配（差异对比模式）

这是权限分配的核心方法，采用了 **diff-based（差异对比）** 模式：

```java
@Override
@DSTransactional
@Caching(evict = {
    @CacheEvict(value = RedisKeyConstants.MENU_ROLE_ID_LIST, allEntries = true),
    @CacheEvict(value = RedisKeyConstants.PERMISSION_MENU_ID_LIST, allEntries = true)
})
public void assignRoleMenu(Long roleId, Set<Long> menuIds) {
    // 1. 查询数据库中当前角色已拥有的菜单
    Set<Long> dbMenuIds = convertSet(
        roleMenuMapper.selectListByRoleId(roleId), RoleMenuDO::getMenuId);

    // 2. 计算差异
    Set<Long> menuIdList = CollUtil.emptyIfNull(menuIds);
    Collection<Long> createMenuIds = CollUtil.subtract(menuIdList, dbMenuIds);  // 新增
    Collection<Long> deleteMenuIds = CollUtil.subtract(dbMenuIds, menuIdList);  // 删除

    // 3. 只执行增量操作
    if (CollUtil.isNotEmpty(createMenuIds)) {
        roleMenuMapper.insertBatch(/* ... */);
    }
    if (CollUtil.isNotEmpty(deleteMenuIds)) {
        roleMenuMapper.deleteListByRoleIdAndMenuIds(roleId, deleteMenuIds);
    }
}
```

**diff-based 模式的优势**：

```
传统做法：先全部删除，再全部插入
├── DELETE FROM system_role_menu WHERE role_id = 1
├── INSERT INTO system_role_menu VALUES (1, 101), (1, 102), ...
└── 问题：产生大量无效的 DELETE + INSERT，且在高并发下可能丢失数据

diff-based 做法：计算差异，只执行必要的操作
├── 当前数据库：{101, 102, 103}
├── 前端提交：  {102, 103, 104}
├── 差异：新增 {104}，删除 {101}
└── 只执行 INSERT (1, 104) 和 DELETE (1, 101)
```

**缓存失效策略**：

- `@CacheEvict(value = MENU_ROLE_ID_LIST, allEntries = true)`：清除所有"菜单→角色"的缓存，因为一次分配可能涉及多个菜单
- `@CacheEvict(value = PERMISSION_MENU_ID_LIST, allEntries = true)`：清除所有"权限→菜单"的缓存
- 使用 `allEntries = true` 是因为无法精确知道哪些缓存条目受影响，全量清除虽然粗暴但简单可靠

---

## 五、assignUserRole —— 用户角色分配

逻辑与 `assignRoleMenu` 完全一致，也是 diff-based 模式：

```java
@Override
@DSTransactional
@CacheEvict(value = RedisKeyConstants.USER_ROLE_ID_LIST, key = "#userId")
public void assignUserRole(Long userId, Set<Long> roleIds) {
    Set<Long> dbRoleIds = convertSet(
        userRoleMapper.selectListByUserId(userId), UserRoleDO::getRoleId);

    Set<Long> roleIdList = CollUtil.emptyIfNull(roleIds);
    Collection<Long> createRoleIds = CollUtil.subtract(roleIdList, dbRoleIds);
    Collection<Long> deleteRoleIds = CollUtil.subtract(dbRoleIds, roleIdList);

    if (!CollectionUtil.isEmpty(createRoleIds)) {
        userRoleMapper.insertBatch(/* ... */);
    }
    if (!CollectionUtil.isEmpty(deleteRoleIds)) {
        userRoleMapper.deleteListByUserIdAndRoleIdIds(userId, deleteRoleIds);
    }
}
```

---

## 六、getDeptDataPermission —— 数据权限计算

当数据权限框架需要判断用户能看哪些部门的数据时，会调用此方法：

```java
@Override
@DataPermission(enable = false) // 关闭数据权限，避免递归
public DeptDataPermissionRespDTO getDeptDataPermission(Long userId) {
    List<RoleDO> roles = getEnableUserRoleListByUserIdFromCache(userId);

    DeptDataPermissionRespDTO result = new DeptDataPermissionRespDTO();
    if (CollUtil.isEmpty(roles)) {
        result.setSelf(true);  // 无角色 → 只能看自己的数据
        return result;
    }

    // 惰性获取用户部门 ID（只查一次数据库）
    Supplier<Long> userDeptId = Suppliers.memoize(
        () -> userService.getUser(userId).getDeptId());

    for (RoleDO role : roles) {
        if (role.getDataScope() == null) continue;

        switch (role.getDataScope()) {
            case ALL:             // 全部数据
                result.setAll(true);
                break;
            case DEPT_CUSTOM:     // 指定部门
                CollUtil.addAll(result.getDeptIds(), role.getDataScopeDeptIds());
                CollectionUtils.addIfNotNull(result.getDeptIds(), userDeptId.get());
                break;
            case DEPT_ONLY:       // 仅本部门
                CollectionUtils.addIfNotNull(result.getDeptIds(), userDeptId.get());
                break;
            case DEPT_AND_CHILD:  // 本部门及子部门
                Long deptId = userDeptId.get();
                if (deptId != null) {
                    CollUtil.addAll(result.getDeptIds(),
                        deptService.getChildDeptIdListFromCache(deptId));
                    result.getDeptIds().add(deptId);
                }
                break;
            case SELF:            // 仅本人
                result.setSelf(true);
                break;
        }
    }
    return result;
}
```

**多角色合并逻辑**：一个用户可能有多个角色，每个角色的数据范围不同。最终结果是 **取并集**——用户能看到所有角色允许的数据的合集。例如：

```
用户张三有两个角色：
├── 角色A：DEPT_ONLY（本部门 = 部门100）
└── 角色B：DEPT_CUSTOM（指定部门 200, 300）

最终张三能看到：部门 100 + 部门 200 + 部门 300 的数据
```

**`@DataPermission(enable = false)` 的必要性**：这个方法本身在计算数据权限，如果数据权限框架在计算过程中再次调用这个方法，会导致无限递归。通过 `enable = false` 临时关闭数据权限注解。

**`Suppliers.memoize()` 的性能考量**：`userDeptId` 是一个惰性计算的 Supplier，只有在第一次调用 `get()` 时才会查询数据库，后续调用直接返回缓存值。因为多个角色可能都需要用到用户部门 ID，这样避免了重复查询。

---

## 七、缓存策略总览

```
Redis 缓存 Key 设计：

user_role_ids:{userId}         → Set<Long>  用户拥有的角色 ID 集合
role:{roleId}                  → RoleDO     角色信息
menu_role_ids:{menuId}         → Set<Long>  拥有指定菜单的角色 ID 集合
permission_menu_ids:{permission} → List<Long> 权限标识对应的菜单 ID 列表

缓存写入时机：
├── @Cacheable 注解自动缓存（首次查询时）
└── 分配操作后通过 @CacheEvict 清除相关缓存

缓存失效时机：
├── assignRoleMenu   → 清除 MENU_ROLE_ID_LIST + PERMISSION_MENU_ID_LIST
├── assignUserRole   → 清除 USER_ROLE_ID_LIST
├── processRoleDeleted → 清除 MENU_ROLE_ID_LIST + USER_ROLE_ID_LIST
└── processMenuDeleted → 清除 MENU_ROLE_ID_LIST（按 menuId 精确清除）
```

---

## 八、getSelf() 模式详解

```java
private PermissionServiceImpl getSelf() {
    return SpringUtil.getBean(getClass());
}
```

**为什么需要这个方法？**

Spring AOP 基于代理实现。当一个 Bean 的方法 A 调用同一个 Bean 的方法 B 时，B 上的 `@Cacheable`、`@Transactional` 等 AOP 注解不会生效，因为调用绕过了代理对象。

```
直接调用（AOP 不生效）：
this.getMenuRoleIdListByMenuIdFromCache(menuId)
└── 直接执行目标方法，跳过缓存代理

通过 getSelf() 调用（AOP 生效）：
getSelf().getMenuRoleIdListByMenuIdFromCache(menuId)
└── 从 Spring 容器获取代理对象 → 走缓存拦截器 → 命中缓存则返回
```

---

## 九、思考题

1. `hasAnyPermissions` 中超管的判断放在循环之后，而不是循环之前。如果改成循环之前先判断超管，对性能有什么影响？

2. `assignRoleMenu` 使用 `allEntries = true` 清除所有缓存条目，能否设计一个更精确的缓存失效策略？

3. `getDeptDataPermission` 中，当一个用户同时有 `ALL` 和 `DEPT_ONLY` 两个角色时，最终结果是什么？代码中是否存在冗余计算？
