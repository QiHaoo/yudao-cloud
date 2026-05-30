# 角色与菜单 CRUD

> 本文档对应源码：
> - `cn.iocoder.yudao.module.system.controller.admin.permission.RoleController`
> - `cn.iocoder.yudao.module.system.controller.admin.permission.MenuController`
> - `cn.iocoder.yudao.module.system.service.permission.RoleServiceImpl`
> - `cn.iocoder.yudao.module.system.service.permission.MenuServiceImpl`

## 一、API 接口总览

### 1.1 角色管理 API

| 方法 | 路径 | 权限标识 | 说明 |
|------|------|---------|------|
| POST | `/system/role/create` | `system:role:create` | 创建角色 |
| PUT | `/system/role/update` | `system:role:update` | 修改角色 |
| DELETE | `/system/role/delete` | `system:role:delete` | 删除角色 |
| DELETE | `/system/role/delete-list` | `system:role:delete` | 批量删除 |
| GET | `/system/role/get` | `system:role:query` | 获取单个角色 |
| GET | `/system/role/page` | `system:role:query` | 分页查询角色 |
| GET | `/system/role/list-all-simple` | **无需权限** | 获取角色下拉列表 |
| GET | `/system/role/export-excel` | `system:role:export` | 导出角色 Excel |

### 1.2 菜单管理 API

| 方法 | 路径 | 权限标识 | 说明 |
|------|------|---------|------|
| POST | `/system/menu/create` | `system:menu:create` | 创建菜单 |
| PUT | `/system/menu/update` | `system:menu:update` | 修改菜单 |
| DELETE | `/system/menu/delete` | `system:menu:delete` | 删除菜单 |
| DELETE | `/system/menu/delete-list` | `system:menu:delete` | 批量删除 |
| GET | `/system/menu/list` | `system:menu:query` | 获取菜单列表（管理界面） |
| GET | `/system/menu/list-all-simple` | **无需权限** | 获取菜单下拉列表 |
| GET | `/system/menu/get` | `system:menu:query` | 获取单个菜单 |

**注意**：`list-all-simple` 接口不需要权限校验，因为它用于前端下拉选择框的数据源（如角色分配菜单时的菜单树）。但它的数据会根据租户套餐过滤——多租户场景下，只返回当前租户已开通的菜单。

---

## 二、角色 CRUD 实现

### 2.1 创建角色

```java
// RoleServiceImpl.java
@Override
@Transactional(rollbackFor = Exception.class)
@LogRecord(type = SYSTEM_ROLE_TYPE, subType = SYSTEM_ROLE_CREATE_SUB_TYPE,
           bizNo = "{{#role.id}}", success = SYSTEM_ROLE_CREATE_SUCCESS)
public Long createRole(RoleSaveReqVO createReqVO, Integer type) {
    // 1. 校验角色名称和编码唯一
    validateRoleDuplicate(createReqVO.getName(), createReqVO.getCode(), null);

    // 2. 插入数据库
    RoleDO role = BeanUtils.toBean(createReqVO, RoleDO.class)
            .setType(ObjectUtil.defaultIfNull(type, RoleTypeEnum.CUSTOM.getType()))
            .setStatus(ObjUtil.defaultIfNull(createReqVO.getStatus(), CommonStatusEnum.ENABLE.getStatus()))
            .setDataScope(DataScopeEnum.ALL.getScope()); // 默认全部数据权限
    roleMapper.insert(role);

    // 3. 记录操作日志
    LogRecordContext.putVariable("role", role);
    return role.getId();
}
```

**设计要点**：
- 新角色默认 `dataScope = ALL`，因为有些小型项目不需要数据权限控制，如果默认限制范围反而会造成困扰
- `type` 参数允许外部指定角色类型，但不传时默认为 `CUSTOM`
- 使用 `@LogRecord` 注解自动记录操作日志，支持变更前后对比（`@DiffLogField`）

### 2.2 校验角色唯一性

```java
// RoleServiceImpl.java
@VisibleForTesting
void validateRoleDuplicate(String name, String code, Long id) {
    // 0. 超级管理员，不允许创建
    if (RoleCodeEnum.isSuperAdmin(code)) {
        throw exception(ROLE_ADMIN_CODE_ERROR, code);
    }
    // 1. 该 name 名字被其它角色所使用
    RoleDO role = roleMapper.selectByName(name);
    if (role != null && !role.getId().equals(id)) {
        throw exception(ROLE_NAME_DUPLICATE, name);
    }
    // 2. 是否存在相同编码的角色
    if (!StringUtils.hasText(code)) {
        return;
    }
    role = roleMapper.selectByCode(code);
    if (role != null && !role.getId().equals(id)) {
        throw exception(ROLE_CODE_DUPLICATE, code);
    }
}
```

**保护机制**：
- 禁止创建 `super_admin` 编码的角色，防止越权
- 名称和编码都做唯一校验
- 更新时传入 `id`，排除自身

### 2.3 删除角色

```java
// RoleServiceImpl.java
@Override
@Transactional(rollbackFor = Exception.class)
@CacheEvict(value = RedisKeyConstants.ROLE, key = "#id")
public void deleteRole(Long id) {
    // 1. 校验是否可以删除
    RoleDO role = validateRoleForUpdate(id);

    // 2.1 标记删除
    roleMapper.deleteById(id);
    // 2.2 删除关联数据（用户-角色、角色-菜单）
    permissionService.processRoleDeleted(id);

    // 3. 记录操作日志
    LogRecordContext.putVariable("role", role);
}
```

**级联清理**：删除角色时，必须同步清理 `system_user_role` 和 `system_role_menu` 中的关联记录，否则会产生"幽灵关联"——角色已经不存在了，但用户还保留着对它的引用。

### 2.4 角色 VO 结构

**RoleSaveReqVO**（创建/修改请求）：

```java
public class RoleSaveReqVO {
    private Long id;           // 角色编号（修改时必填）
    private String name;       // 角色名称，最长 30 字符
    private String code;       // 角色标识，最长 100 字符
    private Integer sort;      // 显示顺序
    private Integer status;    // 状态（CommonStatusEnum）
    private String remark;     // 备注，最长 500 字符
}
```

**RoleRespVO**（响应）：

```java
public class RoleRespVO {
    private Long id;
    private String name;
    private String code;
    private Integer sort;
    private Integer status;
    private Integer type;           // 角色类型
    private String remark;
    private Integer dataScope;      // 数据范围
    private Set<Long> dataScopeDeptIds; // 自定义部门
    private LocalDateTime createTime;
}
```

---

## 三、菜单 CRUD 实现

### 3.1 创建菜单

```java
// MenuServiceImpl.java
@Override
@CacheEvict(value = RedisKeyConstants.PERMISSION_MENU_ID_LIST,
        key = "#createReqVO.permission",
        condition = "#createReqVO.permission != null")
public Long createMenu(MenuSaveVO createReqVO) {
    // 1. 校验父菜单存在
    validateParentMenu(createReqVO.getParentId(), null);
    // 2. 校验菜单名称唯一
    validateMenuName(createReqVO.getParentId(), createReqVO.getName(), null);
    // 3. 校验组件名唯一
    validateMenuComponentName(createReqVO.getComponentName(), null);

    // 4. 插入数据库
    MenuDO menu = BeanUtils.toBean(createReqVO, MenuDO.class);
    initMenuProperty(menu); // 按钮类型清空 path/icon/component
    menuMapper.insert(menu);
    return menu.getId();
}
```

### 3.2 菜单属性初始化

```java
// MenuServiceImpl.java
private void initMenuProperty(MenuDO menu) {
    if (MenuTypeEnum.BUTTON.getType().equals(menu.getType())) {
        menu.setComponent("");
        menu.setComponentName("");
        menu.setIcon("");
        menu.setPath("");
    }
}
```

按钮（BUTTON）类型只需要 `name`、`permission`、`parentId`、`sort`、`status`，其他字段自动清空。

### 3.3 父菜单校验

```java
// MenuServiceImpl.java
void validateParentMenu(Long parentId, Long childId) {
    if (parentId == null || ID_ROOT.equals(parentId)) {
        return; // 根节点无需校验
    }
    // 不能设置自己为父菜单
    if (parentId.equals(childId)) {
        throw exception(MENU_PARENT_ERROR);
    }
    MenuDO menu = menuMapper.selectById(parentId);
    // 父菜单不存在
    if (menu == null) {
        throw exception(MENU_PARENT_NOT_EXISTS);
    }
    // 父菜单必须是目录或菜单类型（按钮不能作为父节点）
    if (!MenuTypeEnum.DIR.getType().equals(menu.getType())
            && !MenuTypeEnum.MENU.getType().equals(menu.getType())) {
        throw exception(MENU_PARENT_NOT_DIR_OR_MENU);
    }
}
```

### 3.4 禁用菜单过滤

`filterDisableMenus()` 方法递归检查菜单树，过滤掉自身或父级被禁用的菜单：

```java
// MenuServiceImpl.java
public List<MenuDO> filterDisableMenus(List<MenuDO> menuList) {
    Map<Long, MenuDO> menuMap = convertMap(menuList, MenuDO::getId);
    List<MenuDO> enabledMenus = new ArrayList<>();
    Set<Long> disabledMenuCache = new HashSet<>(); // 缓存已判定的禁用节点

    for (MenuDO menu : menuList) {
        if (isMenuDisabled(menu, menuMap, disabledMenuCache)) {
            continue;
        }
        enabledMenus.add(menu);
    }
    return enabledMenus;
}
```

**递归判断逻辑**：
- 自身被禁用 → 该节点及所有子孙节点都不可见
- 父节点被禁用 → 该节点不可见
- 使用 `disabledMenuCache` Set 缓存已判定的禁用节点，避免重复遍历

### 3.5 菜单 VO 结构

**MenuSaveVO**（创建/修改请求）：

```java
public class MenuSaveVO {
    private Long id;
    private String name;         // 菜单名称
    private String permission;   // 权限标识（仅按钮需要）
    private Integer type;        // 类型：1=目录，2=菜单，3=按钮
    private Integer sort;        // 显示顺序
    private Long parentId;       // 父菜单 ID
    private String path;         // 路由地址（目录/菜单需要）
    private String icon;         // 菜单图标（目录/菜单需要）
    private String component;    // 组件路径（菜单需要）
    private String componentName;// 组件名
    private Integer status;      // 状态
    private Boolean visible;     // 是否可见
    private Boolean keepAlive;   // 是否缓存
    private Boolean alwaysShow;  // 是否总是显示
}
```

---

## 四、RoleServiceImpl 的缓存与日志

### 4.1 角色缓存

```java
// 从缓存获取单个角色
@Cacheable(value = RedisKeyConstants.ROLE, key = "#id", unless = "#result == null")
public RoleDO getRoleFromCache(Long id) { ... }

// 批量获取（循环调用单个缓存方法）
public List<RoleDO> getRoleListFromCache(Collection<Long> ids) {
    RoleServiceImpl self = getSelf(); // 获取代理对象，确保 AOP 生效
    return CollectionUtils.convertList(ids, self::getRoleFromCache);
}
```

**批量缓存的取舍**：Spring CacheManager 不支持原生的批量操作，所以这里用 for 循环逐个从缓存读取。如果角色数量很多（如 100+），可以考虑使用 `RedisTemplate` 的 Pipeline 批量操作来优化。

### 4.2 操作日志

角色的增删改都使用了 `@LogRecord` 注解：

```java
@LogRecord(
    type = SYSTEM_ROLE_TYPE,              // 日志类型
    subType = SYSTEM_ROLE_CREATE_SUB_TYPE,// 子类型
    bizNo = "{{#role.id}}",               // 业务编号（SpEL 表达式）
    success = SYSTEM_ROLE_CREATE_SUCCESS  // 成功提示
)
```

更新操作还使用了 `DiffParseFunction` 自动对比变更前后的字段差异：

```java
// 更新前保存旧值
LogRecordContext.putVariable(DiffParseFunction.OLD_OBJECT,
    BeanUtils.toBean(role, RoleSaveReqVO.class));
LogRecordContext.putVariable("role", role);
```

---

## 五、Controller 层的权限控制模式

所有 CRUD 接口都使用 `@PreAuthorize` + `@ss.hasPermission` 进行权限校验：

```java
@PostMapping("/create")
@PreAuthorize("@ss.hasPermission('system:role:create')")
public CommonResult<Long> createRole(@Valid @RequestBody RoleSaveReqVO createReqVO) {
    return success(roleService.createRole(createReqVO, null));
}
```

`@ss` 是一个注册到 Spring 容器中的 SpEL Bean，内部调用 `PermissionService.hasAnyPermissions()`。

**权限标识命名规范**：`system:role:create`、`system:role:update`、`system:menu:delete` 等，与 MenuDO 的 `permission` 字段一一对应。

---

## 六、思考题

1. `list-all-simple` 接口不需要 `@PreAuthorize` 权限校验，这是出于什么考虑？如果加上权限校验会有什么问题？

2. 删除角色时需要级联清理 `user_role` 和 `role_menu` 表，但代码中没有在一个事务里同时删除角色本身和关联数据。如果删除角色成功但清理关联数据失败，会产生什么后果？

3. `filterDisableMenus()` 使用了递归 + 缓存的设计，如果菜单层级很深（如 10 层），这种实现的性能表现如何？有没有更好的替代方案？
