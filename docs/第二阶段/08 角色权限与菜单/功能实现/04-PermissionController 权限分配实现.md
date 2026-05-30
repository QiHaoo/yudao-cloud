# PermissionController 权限分配实现

> 本文档对应源码：`cn.iocoder.yudao.module.system.controller.admin.permission.PermissionController`

## 一、权限分配 API 总览

`PermissionController` 提供了五个接口，用于管理员在后台配置角色与菜单、用户与角色之间的关联关系：

| 方法 | 路径 | 权限标识 | 说明 |
|------|------|---------|------|
| GET | `/system/permission/list-role-menus` | `system:permission:assign-role-menu` | 查询角色拥有的菜单 ID 列表 |
| POST | `/system/permission/assign-role-menu` | `system:permission:assign-role-menu` | 给角色分配菜单权限 |
| POST | `/system/permission/assign-role-data-scope` | `system:permission:assign-role-data-scope` | 设置角色的数据范围 |
| GET | `/system/permission/list-user-roles` | `system:permission:assign-user-role` | 查询用户拥有的角色 ID 列表 |
| POST | `/system/permission/assign-user-role` | `system:permission:assign-user-role` | 给用户分配角色 |

---

## 二、角色菜单分配

### 2.1 前端交互流程

```
管理员进入"角色管理"页面
    │
    ├─ 1. 点击某角色的"分配菜单"按钮
    │     GET /system/permission/list-role-menus?roleId=1
    │     → 返回该角色已拥有的菜单 ID 集合: [101, 102, 103]
    │
    ├─ 2. 前端展示菜单树，已勾选的菜单 ID 高亮
    │     菜单数据来源: GET /system/menu/list-all-simple
    │     → 返回所有可用菜单（已过滤禁用和租户未开通的）
    │
    ├─ 3. 管理员勾选/取消勾选菜单，点击"确认"
    │     POST /system/permission/assign-role-menu
    │     Body: { roleId: 1, menuIds: [102, 103, 104] }
    │
    └─ 4. 后端执行 diff-based 分配，清除相关缓存
```

### 2.2 后端实现

```java
// PermissionController.java
@PostMapping("/assign-role-menu")
@PreAuthorize("@ss.hasPermission('system:permission:assign-role-menu')")
public CommonResult<Boolean> assignRoleMenu(
        @Validated @RequestBody PermissionAssignRoleMenuReqVO reqVO) {
    // 开启多租户时，过滤掉租户套餐未开通的菜单
    tenantService.handleTenantMenu(menuIds ->
        reqVO.getMenuIds().removeIf(menuId -> !CollUtil.contains(menuIds, menuId)));
    // 执行分配
    permissionService.assignRoleMenu(reqVO.getRoleId(), reqVO.getMenuIds());
    return success(true);
}
```

**租户安全**：在多租户场景下，`tenantService.handleTenantMenu()` 会先获取当前租户套餐允许的菜单列表，然后从请求中移除不在列表中的菜单 ID。这防止了恶意租户通过接口给自己分配超出套餐范围的菜单权限。

### 2.3 请求 VO

```java
public class PermissionAssignRoleMenuReqVO {
    @NotNull(message = "角色编号不能为空")
    private Long roleId;

    private Set<Long> menuIds = Collections.emptySet(); // 兜底空集合
}
```

注意 `menuIds` 默认值是空集合——如果管理员取消了所有菜单的勾选，传入空集合是合法的，表示该角色不再拥有任何菜单权限。

---

## 三、角色数据权限分配

### 3.1 交互流程

```
管理员进入"角色管理"页面
    │
    ├─ 1. 点击某角色的"分配数据权限"按钮
    │     → 弹窗显示当前角色的数据范围设置
    │
    ├─ 2. 管理员选择数据范围类型（全部/自定义/本部门/本部门及下级/仅本人）
    │     如果选择"自定义"，展示部门树供勾选
    │
    └─ 3. 点击"确认"
          POST /system/permission/assign-role-data-scope
          Body: { roleId: 1, dataScope: 2, dataScopeDeptIds: [10, 20, 30] }
```

### 3.2 后端实现

```java
// PermissionController.java
@PostMapping("/assign-role-data-scope")
@PreAuthorize("@ss.hasPermission('system:permission:assign-role-data-scope')")
public CommonResult<Boolean> assignRoleDataScope(
        @Valid @RequestBody PermissionAssignRoleDataScopeReqVO reqVO) {
    permissionService.assignRoleDataScope(
        reqVO.getRoleId(), reqVO.getDataScope(), reqVO.getDataScopeDeptIds());
    return success(true);
}
```

最终调用 `RoleServiceImpl.updateRoleDataScope()`：

```java
// RoleServiceImpl.java
@Override
@CacheEvict(value = RedisKeyConstants.ROLE, key = "#id")
public void updateRoleDataScope(Long id, Integer dataScope, Set<Long> dataScopeDeptIds) {
    validateRoleForUpdate(id);

    RoleDO updateObject = new RoleDO();
    updateObject.setId(id);
    updateObject.setDataScope(dataScope);
    updateObject.setDataScopeDeptIds(dataScopeDeptIds);
    roleMapper.updateById(updateObject);
}
```

### 3.3 请求 VO

```java
public class PermissionAssignRoleDataScopeReqVO {
    @NotNull(message = "角色编号不能为空")
    private Long roleId;

    @NotNull(message = "数据范围不能为空")
    @InEnum(value = DataScopeEnum.class)
    private Integer dataScope;

    // 只有 dataScope = DEPT_CUSTOM 时才需要
    private Set<Long> dataScopeDeptIds = Collections.emptySet();
}
```

**枚举校验**：`@InEnum(DataScopeEnum.class)` 确保传入的 `dataScope` 值必须是 1-5 之间的合法值。

---

## 四、用户角色分配

### 4.1 交互流程

```
管理员进入"用户管理"页面
    │
    ├─ 1. 点击某用户的"分配角色"按钮
    │     GET /system/permission/list-user-roles?userId=100
    │     → 返回该用户已拥有的角色 ID 集合: [1, 3]
    │
    ├─ 2. 前端展示角色列表（多选框），已选角色高亮
    │     角色数据来源: GET /system/role/list-all-simple
    │     → 返回所有启用的角色
    │
    ├─ 3. 管理员勾选角色，点击"确认"
    │     POST /system/permission/assign-user-role
    │     Body: { userId: 100, roleIds: [1, 2, 5] }
    │
    └─ 4. 后端执行 diff-based 分配，清除用户角色缓存
```

### 4.2 后端实现

```java
// PermissionController.java
@PostMapping("/assign-user-role")
@PreAuthorize("@ss.hasPermission('system:permission:assign-user-role')")
public CommonResult<Boolean> assignUserRole(
        @Validated @RequestBody PermissionAssignUserRoleReqVO reqVO) {
    permissionService.assignUserRole(reqVO.getUserId(), reqVO.getRoleIds());
    return success(true);
}
```

请求 VO：

```java
public class PermissionAssignUserRoleReqVO {
    @NotNull(message = "用户编号不能为空")
    private Long userId;

    private Set<Long> roleIds = Collections.emptySet();
}
```

---

## 五、Diff-Based 分配模式详解

所有分配操作都采用了相同的 diff-based 模式，这里用"角色菜单分配"做完整说明：

```
步骤 1：查询数据库当前状态
    SELECT menu_id FROM system_role_menu WHERE role_id = 1
    → dbMenuIds = {101, 102, 103}

步骤 2：前端提交的目标状态
    menuIds = {102, 103, 104}

步骤 3：计算差异
    createMenuIds = menuIds - dbMenuIds = {104}       // 需要新增
    deleteMenuIds = dbMenuIds - menuIds = {101}       // 需要删除

步骤 4：执行增量操作
    INSERT INTO system_role_menu (role_id, menu_id) VALUES (1, 104)
    DELETE FROM system_role_menu WHERE role_id = 1 AND menu_id IN (101)
```

**与"先删后插"模式的对比**：

| 维度 | 先删后插 | Diff-based |
|------|---------|------------|
| SQL 执行次数 | 2 次（DELETE ALL + INSERT ALL） | 0~2 次（仅差异部分） |
| 数据库压力 | 每次都全量操作 | 只操作变更部分 |
| 并发安全 | 删除后、插入前如果有查询，会短暂丢失数据 | 不影响已有数据 |
| 实现复杂度 | 简单 | 需要计算集合差集 |

---

## 六、缓存失效策略

每次分配操作都会清除相关的 Redis 缓存，确保下次权限校验能读到最新数据：

```
assignRoleMenu(roleId, menuIds)
├── @CacheEvict(MENU_ROLE_ID_LIST, allEntries = true)
│   原因：角色的菜单变了，所有"菜单→角色"的映射都需要更新
│
└── @CacheEvict(PERMISSION_MENU_ID_LIST, allEntries = true)
    原因：菜单权限变了，"权限标识→菜单ID"的缓存可能受影响

assignUserRole(userId, roleIds)
└── @CacheEvict(USER_ROLE_ID_LIST, key = "#userId")
    原因：用户的角色变了，只需清除该用户的缓存（精确到 key）

assignRoleDataScope → 通过 RoleService.updateRoleDataScope()
└── @CacheEvict(ROLE, key = "#id")
    原因：角色的数据范围变了，只需清除该角色的缓存
```

**为什么 `assignRoleMenu` 用 `allEntries = true` 而 `assignUserRole` 用精确的 key？**

- `assignRoleMenu` 涉及多个 menuId，无法用单个 key 表达，只能全量清除
- `assignUserRole` 只涉及一个 userId，可以精确清除

---

## 七、权限分配的整体架构

```
┌──────────────────────────────────────────────────────────────┐
│                        前端管理界面                            │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ 角色管理页面  │  │ 用户管理页面  │  │ 菜单管理页面        │  │
│  └──────┬──────┘  └──────┬───────┘  └────────┬───────────┘  │
└─────────┼────────────────┼────────────────────┼──────────────┘
          │                │                    │
          ▼                ▼                    ▼
┌──────────────────────────────────────────────────────────────┐
│                  PermissionController                         │
│  ┌─────────────────┐  ┌──────────────────┐                   │
│  │ assignRoleMenu  │  │ assignUserRole   │                   │
│  │ assignRoleData  │  │ listUserRoles    │                   │
│  └────────┬────────┘  └────────┬─────────┘                   │
└───────────┼────────────────────┼──────────────────────────────┘
            │                    │
            ▼                    ▼
┌──────────────────────────────────────────────────────────────┐
│                  PermissionService                            │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────┐│
│  │ Diff 计算     │  │ DB 操作       │  │ 缓存清除            ││
│  │ CollUtil.     │  │ insertBatch  │  │ @CacheEvict         ││
│  │ subtract()    │  │ deleteList   │  │ allEntries / key    ││
│  └──────────────┘  └──────────────┘  └─────────────────────┘│
└──────────────────────────────────────────────────────────────┘
            │
            ▼
┌──────────────────────────────────────────────────────────────┐
│  MySQL                              Redis                     │
│  system_role_menu                   user_role_ids:*           │
│  system_user_role                   menu_role_ids:*           │
│  system_role (dataScope)            permission_menu_ids:*     │
└──────────────────────────────────────────────────────────────┘
```

---

## 八、思考题

1. `assignRoleMenu` 中先调用 `tenantService.handleTenantMenu()` 过滤菜单，如果某个菜单 ID 不在租户套餐中但前端传了，后端是静默忽略还是应该报错？两种策略各有什么优劣？

2. 如果一个管理员同时打开了两个浏览器标签页，分别给同一个角色分配不同的菜单，最后保存的结果会是什么？Diff-based 模式在这种并发场景下是否安全？

3. 当前的权限分配接口没有记录操作日志（对比角色 CRUD 使用了 `@LogRecord`），是否应该补充？如果补充，日志内容应该怎么设计？
