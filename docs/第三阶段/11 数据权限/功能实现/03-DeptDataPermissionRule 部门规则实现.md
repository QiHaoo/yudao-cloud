# 03 - DeptDataPermissionRule 部门规则实现

本章介绍数据权限组件的内置规则实现——`DeptDataPermissionRule`，它是基于部门（dept_id）和用户（user_id）的数据权限规则，支持五级数据范围的 SQL 过滤。

## DeptDataPermissionRule 类结构

```java
// 文件路径：yudao-framework/yudao-spring-boot-starter-biz-data-permission/src/main/java/cn/iocoder/yudao/framework/datapermission/core/rule/dept/DeptDataPermissionRule.java
package cn.iocoder.yudao.framework.datapermission.core.rule.dept;

import cn.hutool.core.collection.CollUtil;
import cn.hutool.core.util.ObjectUtil;
import cn.hutool.core.util.StrUtil;
import cn.iocoder.yudao.framework.common.biz.system.permission.PermissionCommonApi;
import cn.iocoder.yudao.framework.common.enums.UserTypeEnum;
import cn.iocoder.yudao.framework.common.util.collection.CollectionUtils;
import cn.iocoder.yudao.framework.common.util.json.JsonUtils;
import cn.iocoder.yudao.framework.datapermission.core.rule.DataPermissionRule;
import cn.iocoder.yudao.framework.mybatis.core.dataobject.BaseDO;
import cn.iocoder.yudao.framework.mybatis.core.util.MyBatisUtils;
import cn.iocoder.yudao.framework.security.core.LoginUser;
import cn.iocoder.yudao.framework.security.core.util.SecurityFrameworkUtils;
import cn.iocoder.yudao.framework.common.biz.system.permission.dto.DeptDataPermissionRespDTO;
import com.baomidou.mybatisplus.core.metadata.TableInfoHelper;
import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import net.sf.jsqlparser.expression.*;
import net.sf.jsqlparser.expression.operators.conditional.OrExpression;
import net.sf.jsqlparser.expression.operators.relational.EqualsTo;
import net.sf.jsqlparser.expression.operators.relational.ExpressionList;
import net.sf.jsqlparser.expression.operators.relational.InExpression;
import net.sf.jsqlparser.expression.operators.relational.ParenthesedExpressionList;

import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;

/**
 * 基于部门的 {@link DataPermissionRule} 数据权限规则实现
 *
 * 注意，使用 DeptDataPermissionRule 时，需要保证表中有 dept_id 部门编号的字段，可自定义。
 *
 * 实际业务场景下，会存在一个经典的问题？当用户修改部门时，冗余的 dept_id 是否需要修改？
 * 1. 一般情况下，dept_id 不进行修改，则会导致用户看不到之前的数据。【yudao-server 采用该方案】
 * 2. 部分情况下，希望该用户还是能看到之前的数据，则有两种方式解决：
 *  1）编写洗数据的脚本，将 dept_id 修改成新部门的编号；【建议】
 *      最终过滤条件是 WHERE dept_id = ?
 *  2）洗数据的话，可能涉及的数据量较大，也可以采用 user_id 进行过滤的方式，此时需要获取到 dept_id 对应的所有 user_id 用户编号；
 *      最终过滤条件是 WHERE user_id IN (?, ?, ? ...)
 *  3）想要保证原 dept_id 和 user_id 都可以看的到，此时使用 dept_id 和 user_id 一起过滤；
 *      最终过滤条件是 WHERE dept_id = ? OR user_id IN (?, ?, ? ...)
 *
 * @author 芋道源码
 */
@AllArgsConstructor
@Slf4j
public class DeptDataPermissionRule implements DataPermissionRule {

    /**
     * LoginUser 的 Context 缓存 Key
     */
    protected static final String CONTEXT_KEY = DeptDataPermissionRule.class.getSimpleName();

    private static final String DEPT_COLUMN_NAME = "dept_id";
    private static final String USER_COLUMN_NAME = "user_id";

    private final PermissionCommonApi permissionApi;

    /**
     * 基于部门的表字段配置
     * 一般情况下，每个表的部门编号字段是 dept_id，通过该配置自定义。
     *
     * key：表名
     * value：字段名
     */
    private final Map<String, String> deptColumns = new HashMap<>();
    /**
     * 基于用户的表字段配置
     * 一般情况下，每个表的部门编号字段是 dept_id，通过该配置自定义。
     *
     * key：表名
     * value：字段名
     */
    private final Map<String, String> userColumns = new HashMap<>();
    /**
     * 所有表名，是 {@link #deptColumns} 和 {@link #userColumns} 的合集
     */
    private final Set<String> TABLE_NAMES = new HashSet<>();

    // ... 后续方法
}
```

## deptColumns 和 userColumns 配置

这两个 Map 是 `DeptDataPermissionRule` 的核心配置，建立了**表名**到**字段名**的映射关系：

```
deptColumns:
  ┌─────────────────┬───────────┐
  │ 表名             │ 字段名    │
  ├─────────────────┼───────────┤
  │ trade_order     │ dept_id   │  ← 订单表的部门字段是 dept_id
  │ trade_customer  │ dept_id   │  ← 客户表的部门字段是 dept_id
  │ system_dept     │ id        │  ← 部门表的主键是 id（特殊处理）
  └─────────────────┴───────────┘

userColumns:
  ┌─────────────────┬───────────┐
  │ 表名             │ 字段名    │
  ├─────────────────┼───────────┤
  │ trade_order     │ user_id   │  ← 订单表的用户字段是 user_id
  │ trade_customer  │ creator   │  ← 客户表的用户字段是 creator
  └─────────────────┴───────────┘
```

### 配置方法

```java
// ==================== 添加配置 ====================

public void addDeptColumn(Class<? extends BaseDO> entityClass) {
    addDeptColumn(entityClass, DEPT_COLUMN_NAME);
}

public void addDeptColumn(Class<? extends BaseDO> entityClass, String columnName) {
    String tableName = TableInfoHelper.getTableInfo(entityClass).getTableName();
    addDeptColumn(tableName, columnName);
}

public void addDeptColumn(String tableName, String columnName) {
    deptColumns.put(tableName, columnName);
    TABLE_NAMES.add(tableName);
}

public void addUserColumn(Class<? extends BaseDO> entityClass) {
    addUserColumn(entityClass, USER_COLUMN_NAME);
}

public void addUserColumn(Class<? extends BaseDO> entityClass, String columnName) {
    String tableName = TableInfoHelper.getTableInfo(entityClass).getTableName();
    addUserColumn(tableName, columnName);
}

public void addUserColumn(String tableName, String columnName) {
    userColumns.put(tableName, columnName);
    TABLE_NAMES.add(tableName);
}
```

### 配置方式说明

| 方法 | 作用 | 示例 |
|------|------|------|
| `addDeptColumn(Class<?> entityClass)` | 注册部门字段（默认 `dept_id`） | `rule.addDeptColumn(OrderDO.class)` |
| `addDeptColumn(Class<?> entityClass, String column)` | 注册部门字段（自定义列名） | `rule.addDeptColumn(DeptDO.class, "id")` |
| `addUserColumn(Class<?> entityClass)` | 注册用户字段（默认 `user_id`） | `rule.addUserColumn(OrderDO.class)` |
| `addUserColumn(Class<?> entityClass, String column)` | 注册用户字段（自定义列名） | `rule.addUserColumn(OrderDO.class, "creator")` |

## getTableNames() 方法

```java
@Override
public Set<String> getTableNames() {
    return TABLE_NAMES;
}
```

`TABLE_NAMES` 是 `deptColumns` 和 `userColumns` 的 key 集合的合集。当调用 `addDeptColumn()` 或 `addUserColumn()` 时，会自动将表名添加到 `TABLE_NAMES` 中。

## getExpression() 核心逻辑

`getExpression()` 是 `DeptDataPermissionRule` 最核心的方法，它根据当前用户的数据权限生成 SQL 过滤条件：

```java
@Override
public Expression getExpression(String tableName, Alias tableAlias) {
    // 只有有登陆用户的情况下，才进行数据权限的处理
    LoginUser loginUser = SecurityFrameworkUtils.getLoginUser();
    if (loginUser == null) {
        return null;
    }
    // 只有管理员类型的用户，才进行数据权限的处理
    if (ObjectUtil.notEqual(loginUser.getUserType(), UserTypeEnum.ADMIN.getValue())) {
        return null;
    }

    // 获得数据权限
    DeptDataPermissionRespDTO deptDataPermission = loginUser.getContext(CONTEXT_KEY, DeptDataPermissionRespDTO.class);
    // 从上下文中拿不到，则调用逻辑进行获取
    if (deptDataPermission == null) {
        deptDataPermission = permissionApi.getDeptDataPermission(loginUser.getId()).getCheckedData();
        if (deptDataPermission == null) {
            log.error("[getExpression][LoginUser({}) 获取数据权限为 null]", JsonUtils.toJsonString(loginUser));
            throw new NullPointerException(String.format("LoginUser(%d) Table(%s/%s) 未返回数据权限",
                    loginUser.getId(), tableName, tableAlias.getName()));
        }
        // 添加到上下文中，避免重复计算
        loginUser.setContext(CONTEXT_KEY, deptDataPermission);
    }

    // 情况一，如果是 ALL 可查看全部，则无需拼接条件
    if (deptDataPermission.getAll()) {
        return null;
    }

    // 情况二，即不能查看部门，又不能查看自己，则说明 100% 无权限
    if (CollUtil.isEmpty(deptDataPermission.getDeptIds())
        && Boolean.FALSE.equals(deptDataPermission.getSelf())) {
        return new EqualsTo(null, null); // WHERE null = null，可以保证返回的数据为空
    }

    // 情况三，拼接 Dept 和 User 的条件，最后组合
    Expression deptExpression = buildDeptExpression(tableName,tableAlias, deptDataPermission.getDeptIds());
    Expression userExpression = buildUserExpression(tableName, tableAlias, deptDataPermission.getSelf(), loginUser.getId());
    if (deptExpression == null && userExpression == null) {
        // TODO 芋艿：获得不到条件的时候，暂时不抛出异常，而是不返回数据
        log.warn("[getExpression][LoginUser({}) Table({}/{}) DeptDataPermission({}) 构建的条件为空]",
                JsonUtils.toJsonString(loginUser), tableName, tableAlias, JsonUtils.toJsonString(deptDataPermission));
        return new EqualsTo(null, null); // WHERE null = null，可以保证返回的数据为空
    }
    if (deptExpression == null) {
        return userExpression;
    }
    if (userExpression == null) {
        return deptExpression;
    }
    // 目前，如果有指定部门 + 可查看自己，采用 OR 条件。即，WHERE (dept_id IN ? OR user_id = ?)
    return new ParenthesedExpressionList(new OrExpression(deptExpression, userExpression));
}
```

### getExpression() 流程图

```
getExpression(tableName, tableAlias)
    │
    ▼
有登录用户？ ──否──▶ return null（不加条件）
    │是
    ▼
是管理员类型？ ──否──▶ return null（非管理员不处理）
    │是
    ▼
获取 DeptDataPermissionRespDTO
    │
    ▼
┌─── 判断数据范围 ──────────────────────────────────────┐
│                                                       │
│   all = true？ ──是──▶ return null（查看全部）          │
│   │否                                                │
│   ▼                                                   │
│   deptIds 为空 且 self = false？                       │
│   │是 → return null = null（无权限，返回空）            │
│   │否                                                │
│   ▼                                                   │
│   构建 deptExpression（部门条件）                       │
│   构建 userExpression（用户条件）                       │
│   │                                                   │
│   ▼                                                   │
│   两者都为 null？ → return null = null                  │
│   只有 dept？     → return deptExpression              │
│   只有 user？     → return userExpression              │
│   两者都有？      → return (dept OR user)              │
│                                                       │
└───────────────────────────────────────────────────────┘
```

## 五种数据范围的 SQL 生成

### 情况一：ALL（全部数据）

```java
if (deptDataPermission.getAll()) {
    return null;
}
```

当用户的数据范围是 ALL 时，`getExpression()` 返回 `null`，表示不添加任何过滤条件。

SQL 效果：
```sql
-- 原始 SQL
SELECT * FROM trade_order WHERE status = 1

-- 改写后（无变化）
SELECT * FROM trade_order WHERE status = 1
```

### 情况二：无权限（返回空结果）

```java
if (CollUtil.isEmpty(deptDataPermission.getDeptIds())
    && Boolean.FALSE.equals(deptDataPermission.getSelf())) {
    return new EqualsTo(null, null); // WHERE null = null
}
```

当用户既没有部门权限也没有个人权限时，返回 `WHERE null = null`，保证返回空结果集。

SQL 效果：
```sql
-- 原始 SQL
SELECT * FROM trade_order WHERE status = 1

-- 改写后
SELECT * FROM trade_order WHERE status = 1 AND null = null
-- 结果：返回空数据集
```

### 情况三：部门过滤（DEPT_ONLY、DEPT_AND_CHILD、DEPT_CUSTOM）

```java
private Expression buildDeptExpression(String tableName, Alias tableAlias, Set<Long> deptIds) {
    // 如果不存在配置，则无需作为条件
    String columnName = deptColumns.get(tableName);
    if (StrUtil.isEmpty(columnName)) {
        return null;
    }
    // 如果为空，则无条件
    if (CollUtil.isEmpty(deptIds)) {
        return null;
    }
    // 拼接条件
    return new InExpression(MyBatisUtils.buildColumn(tableName, tableAlias, columnName),
            // Parenthesis 的目的，是提供 (1,2,3) 的 () 左右括号
            new ParenthesedExpressionList(new ExpressionList<LongValue>(CollectionUtils.convertList(deptIds, LongValue::new))));
}
```

SQL 效果：
```sql
-- 用户可见部门：[100, 101, 102]
-- trade_order 表配置了 deptColumns: {"trade_order": "dept_id"}

-- 原始 SQL
SELECT * FROM trade_order WHERE status = 1

-- 改写后
SELECT * FROM trade_order WHERE status = 1 AND dept_id IN (100, 101, 102)
```

### 情况四：用户过滤（SELF）

```java
private Expression buildUserExpression(String tableName, Alias tableAlias, Boolean self, Long userId) {
    // 如果不查看自己，则无需作为条件
    if (Boolean.FALSE.equals(self)) {
        return null;
    }
    String columnName = userColumns.get(tableName);
    if (StrUtil.isEmpty(columnName)) {
        return null;
    }
    // 拼接条件
    return new EqualsTo(MyBatisUtils.buildColumn(tableName, tableAlias, columnName), new LongValue(userId));
}
```

SQL 效果：
```sql
-- 用户 ID：42
-- trade_order 表配置了 userColumns: {"trade_order": "user_id"}

-- 原始 SQL
SELECT * FROM trade_order WHERE status = 1

-- 改写后
SELECT * FROM trade_order WHERE status = 1 AND user_id = 42
```

### 情况五：部门 + 用户组合过滤

当用户同时拥有"部门"和"个人"的数据权限时，最终的 SQL 条件是：

```java
// 目前，如果有指定部门 + 可查看自己，采用 OR 条件
return new ParenthesedExpressionList(new OrExpression(deptExpression, userExpression));
```

SQL 效果：
```sql
-- 用户可见部门：[100, 101]，用户 ID：42
-- trade_order 表同时配置了 deptColumns 和 userColumns

-- 原始 SQL
SELECT * FROM trade_order WHERE status = 1

-- 改写后
SELECT * FROM trade_order WHERE status = 1 AND (dept_id IN (100, 101) OR user_id = 42)
```

这个 OR 条件的含义是：**用户可以看到自己所在部门的数据，也可以看到自己创建的数据**。

## DeptDataPermissionRespDTO：数据权限传输对象

`DeptDataPermissionRespDTO` 是数据权限计算结果的载体：

```java
// 文件路径：yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/biz/system/permission/dto/DeptDataPermissionRespDTO.java
@Data
public class DeptDataPermissionRespDTO {

    /**
     * 是否可查看全部数据
     */
    private Boolean all;

    /**
     * 是否可查看自己的数据
     */
    private Boolean self;

    /**
     * 可查看的部门编号数组
     */
    private Set<Long> deptIds;

    public DeptDataPermissionRespDTO() {
        this.all = false;
        this.self = false;
        this.deptIds = new HashSet<>();
    }
}
```

三个字段与 `DataScopeEnum` 的对应关系：

| DataScopeEnum | all | self | deptIds |
|--------------|-----|------|---------|
| ALL | true | false | 空 |
| DEPT_CUSTOM | false | false | [指定的部门编号] |
| DEPT_ONLY | false | false | [用户所在部门] |
| DEPT_AND_CHILD | false | false | [用户所在部门 + 所有子部门] |
| SELF | false | true | 空 |

注意：一个用户可能拥有多个角色，每个角色有不同的数据范围。最终结果是所有角色的**并集**。

## 用户数据权限缓存

`getExpression()` 方法中，数据权限的获取有缓存机制：

```java
// 获得数据权限
DeptDataPermissionRespDTO deptDataPermission = loginUser.getContext(CONTEXT_KEY, DeptDataPermissionRespDTO.class);
// 从上下文中拿不到，则调用逻辑进行获取
if (deptDataPermission == null) {
    deptDataPermission = permissionApi.getDeptDataPermission(loginUser.getId()).getCheckedData();
    if (deptDataPermission == null) {
        log.error("[getExpression][LoginUser({}) 获取数据权限为 null]", JsonUtils.toJsonString(loginUser));
        throw new NullPointerException(String.format("LoginUser(%d) Table(%s/%s) 未返回数据权限",
                loginUser.getId(), tableName, tableAlias.getName()));
    }
    // 添加到上下文中，避免重复计算
    loginUser.setContext(CONTEXT_KEY, deptDataPermission);
}
```

缓存策略：
- 数据权限存储在 `LoginUser` 的上下文中，通过 `CONTEXT_KEY` 作为 key
- 首次获取时调用 `permissionApi.getDeptDataPermission()` 进行 RPC 调用
- 后续获取时直接从缓存中读取，避免重复 RPC 调用

## 自动配置

`DeptDataPermissionRule` 通过 `YudaoDeptDataPermissionAutoConfiguration` 自动配置：

```java
// 文件路径：yudao-framework/yudao-spring-boot-starter-biz-data-permission/src/main/java/cn/iocoder/yudao/framework/datapermission/config/YudaoDeptDataPermissionAutoConfiguration.java
package cn.iocoder.yudao.framework.datapermission.config;

import cn.hutool.extra.spring.SpringUtil;
import cn.iocoder.yudao.framework.common.biz.system.permission.PermissionCommonApi;
import cn.iocoder.yudao.framework.datapermission.core.rule.dept.DeptDataPermissionRule;
import cn.iocoder.yudao.framework.datapermission.core.rule.dept.DeptDataPermissionRuleCustomizer;
import cn.iocoder.yudao.framework.security.core.LoginUser;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.context.annotation.Bean;

import java.util.List;

/**
 * 基于部门的数据权限 AutoConfiguration
 *
 * @author 芋道源码
 */
@AutoConfiguration
@ConditionalOnClass(LoginUser.class)
@ConditionalOnBean(value = {DeptDataPermissionRuleCustomizer.class})
public class YudaoDeptDataPermissionAutoConfiguration {

    @Bean
    public DeptDataPermissionRule deptDataPermissionRule(PermissionCommonApi permissionApi,
                                                         List<DeptDataPermissionRuleCustomizer> customizers) {
        // Cloud 专属逻辑：优先使用本地的 PermissionApi 实现类，而不是 Feign 调用
        // 原因：在创建租户时，租户还没创建好，导致 Feign 调用获取数据权限时，报"租户不存在"的错误
        try {
            PermissionCommonApi permissionApiImpl = SpringUtil.getBean("permissionApiImpl", PermissionCommonApi.class);
            if (permissionApiImpl != null) {
                permissionApi = permissionApiImpl;
            }
        } catch (Exception ignored) {}

        // 创建 DeptDataPermissionRule 对象
        DeptDataPermissionRule rule = new DeptDataPermissionRule(permissionApi);
        // 补全表配置
        customizers.forEach(customizer -> customizer.customize(rule));
        return rule;
    }

}
```

自动配置的条件：
- `@ConditionalOnClass(LoginUser.class)`：必须有安全框架
- `@ConditionalOnBean(DeptDataPermissionRuleCustomizer.class)`：必须有业务层的 Customizer 配置

这意味着：**如果业务层没有配置 `DeptDataPermissionRuleCustomizer`，数据权限规则不会被激活**。

## 业务层配置示例

在 system 模块中，通过 `DataPermissionConfiguration` 配置部门规则：

```java
// 文件路径：yudao-module-system/yudao-module-system-server/src/main/java/cn/iocoder/yudao/module/system/framework/datapermission/config/DataPermissionConfiguration.java
package cn.iocoder.yudao.module.system.framework.datapermission.config;

import cn.iocoder.yudao.module.system.dal.dataobject.dept.DeptDO;
import cn.iocoder.yudao.module.system.dal.dataobject.user.AdminUserDO;
import cn.iocoder.yudao.framework.datapermission.core.rule.dept.DeptDataPermissionRuleCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * system 模块的数据权限 Configuration
 *
 * @author 芋道源码
 */
@Configuration(proxyBeanMethods = false)
public class DataPermissionConfiguration {

    @Bean
    public DeptDataPermissionRuleCustomizer sysDeptDataPermissionRuleCustomizer() {
        return rule -> {
            // dept
            rule.addDeptColumn(AdminUserDO.class);
            rule.addDeptColumn(DeptDO.class, "id");
            // user
            rule.addUserColumn(AdminUserDO.class, "id");
        };
    }

}
```

配置说明：
- `AdminUserDO.class`：用户表，部门字段默认为 `dept_id`，用户字段为 `id`（特殊：用户表的主键就是用户编号）
- `DeptDO.class`：部门表，部门字段为 `id`（特殊：部门表的主键就是部门编号）

## 思考题

1. `DeptDataPermissionRule` 的 `getExpression()` 方法中，如果用户既没有部门权限也没有个人权限，返回 `new EqualsTo(null, null)` 而不是 `return null`。这两种返回值有什么区别？为什么选择前者？

2. `buildDeptExpression()` 和 `buildUserExpression()` 方法中，如果表没有配置对应的列，会返回 `null`。这种设计有什么好处？如果抛出异常会怎样？

3. 当用户拥有多个角色时，数据权限是如何合并的？如果一个用户同时拥有 `DEPT_ONLY` 和 `SELF` 两个角色，最终的 SQL 条件是什么？
