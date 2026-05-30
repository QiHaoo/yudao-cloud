# 04-核心设计：RBAC 权限模型

## 场景引入

电商后台有多种角色：
- **超级管理员**：可以做任何事
- **商品管理员**：管理商品上下架，但不能改价格
- **订单客服**：查看订单、处理退款，但不能修改商品
- **仓库管理员**：管理库存和发货，但不能查看财务数据

如果每个用户的权限都单独配置，100 个用户就需要配置 100 次。RBAC 的核心思想是：**把权限赋给角色，再把角色赋给用户**。

## RBAC 模型全景

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│  用户 A   │     │  用户 B   │     │  用户 C   │
│ (运营)    │     │ (客服)    │     │ (仓库)    │
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
     └────────┬───────┘────────┬───────┘
              │                │
        ┌─────▼─────┐    ┌────▼──────┐
        │  角色：运营  │    │  角色：客服  │
        └─────┬─────┘    └────┬──────┘
              │                │
    ┌─────────┼─────────┐     │
    │         │         │     │
┌───▼──┐ ┌───▼──┐ ┌───▼──┐ ┌▼─────┐
│菜单：  │ │菜单：  │ │菜单：  │ │菜单：  │
│商品    │ │订单    │ │活动    │ │订单    │
│管理    │ │管理    │ │管理    │ │查看    │
└───┬──┘ └───┬──┘ └───┬──┘ └┬─────┘
    │        │        │     │
┌───▼──┐ ┌───▼──┐ ┌───▼──┐ ┌▼─────┐
│权限：  │ │权限：  │ │权限：  │ │权限：  │
│商品    │ │订单    │ │活动    │ │订单    │
│创建    │ │查询    │ │创建    │ │查询    │
│商品    │ │订单    │ │活动    │ │订单    │
│编辑    │ │退款    │ │编辑    │ │退款    │
└──────┘ └──────┘ └──────┘ └──────┘
```

## 核心实体关系

```
AdminUserDO ←── UserRoleDO ──→ RoleDO ←── RoleMenuDO ──→ MenuDO
  (用户)        (用户-角色)      (角色)     (角色-菜单)      (菜单)
                                    │
                                    └── dataScope + dataScopeDeptIds
                                         (数据权限)
```

### 为什么用中间表而不是直接关联？

如果在 `AdminUserDO` 中用 `roleIds` JSON 字段存储角色，就像 `postIds` 一样：
- ✅ 简单，不需要额外的表
- ❌ 无法高效查询"某个角色下有哪些用户"
- ❌ 无法高效查询"某个用户是否拥有某个角色"
- ❌ 无法加数据库约束

中间表 `UserRoleDO` 和 `RoleMenuDO` 虽然多了表，但支持双向高效查询。

## 菜单：权限的载体

在本项目中，"菜单"不仅仅是导航栏的菜单项，它承载了三种角色：

```
MenuTypeEnum:
├── DIR(1)    目录：导航栏的文件夹图标
├── MENU(2)   菜单：导航栏的可点击项
└── BUTTON(3) 按钮：页面内的操作权限
```

### 菜单树

```
系统管理（DIR）
├── 用户管理（MENU）→ /system/user
│   ├── 用户新增（BUTTON）→ permission: "system:user:create"
│   ├── 用户修改（BUTTON）→ permission: "system:user:update"
│   ├── 用户删除（BUTTON）→ permission: "system:user:delete"
│   └── 用户导出（BUTTON）→ permission: "system:user:export"
├── 角色管理（MENU）→ /system/role
│   ├── 角色新增（BUTTON）→ permission: "system:role:create"
│   └── ...
└── 菜单管理（MENU）→ /system/menu
```

### 权限标识的命名规范

```
${模块}:${实体}:${操作}

示例：
system:user:create    系统模块 - 用户 - 创建
system:role:delete    系统模块 - 角色 - 删除
infra:config:query    基础设施 - 配置 - 查询
```

这个命名规范不是强制的，但遵循它可以让权限管理更清晰。

## 两级权限控制

本项目的权限控制分为两层：

### 第一层：URL 级别（SecurityFilterChain）

```java
// YudaoWebSecurityConfigurerAdapter
.anyRequest().authenticated()  // 其他所有请求需要认证
```

所有请求都必须经过 `TokenAuthenticationFilter` 认证。未认证的请求返回 401。

### 第二层：方法级别（@PreAuthorize）

```java
@PreAuthorize("@ss.hasPermission('system:user:create')")
@PostMapping("/create")
public CommonResult<Long> createUser(...) { ... }
```

即使用户已认证，也需要有对应的权限标识才能访问具体接口。没有权限返回 403。

### 两层的关系

```
请求到达
    │
    ▼
┌─────────────────────────┐
│ 第一层：Token 认证        │
│ TokenAuthenticationFilter│
│ ├─ Token 无效 → 401      │
│ └─ Token 有效 → 继续      │
└────────┬────────────────┘
         │
         ▼
┌─────────────────────────┐
│ 第二层：权限校验           │
│ @PreAuthorize            │
│ ├─ 无权限 → 403           │
│ └─ 有权限 → 执行方法      │
└─────────────────────────┘
```

## 超级管理员的特殊处理

```java
// PermissionServiceImpl
public boolean hasAnyPermissions(Long userId, String... permissions) {
    // ...
    // 如果是超管，也说明有权限
    return roleService.hasAnySuperAdmin(convertSet(roles, RoleDO::getId));
}
```

超级管理员（`RoleCodeEnum.TENANT_ADMIN`）跳过所有权限检查。这是一个设计决策：

| 方案 | 优点 | 缺点 |
|------|------|------|
| 超管跳过检查 | 简单，不会因配置错误锁死超管 | 超管可以做任何事，没有审计 |
| 超管也需配置权限 | 完整的审计链 | 配置复杂，容易遗漏 |

本项目选择前者，因为超管本身就是系统管理员，不需要自我限制。

## 角色类型

```java
public enum RoleTypeEnum {
    SYSTEM(1),  // 内置角色，不可删除
    CUSTOM(2)   // 自定义角色，用户创建
}
```

### 为什么区分系统角色和自定义角色？

- **系统角色**（如 `TENANT_ADMIN`）：框架运行依赖，不能被删除或修改
- **自定义角色**：用户根据业务需要创建，可以自由修改和删除

删除角色时需要检查是否为系统角色，防止误删导致系统异常。

## 思考题

1. 本项目将"菜单"和"权限"合并在一个实体 `MenuDO` 中（通过 `type` 区分）。如果拆成两个独立实体 `MenuDO` 和 `PermissionDO`，有什么优劣？
2. 如果电商平台要支持"临时权限"（如双十一大促期间临时开放价格修改权限），RBAC 模型需要怎么扩展？
3. 本项目中，一个用户可以有多个角色，权限取并集。如果需要"权限互斥"（如"审批人"和"申请人"不能是同一个人），该怎么设计？
