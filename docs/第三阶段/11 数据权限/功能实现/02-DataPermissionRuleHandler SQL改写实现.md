# 02 - DataPermissionRuleHandler SQL改写实现

本章介绍数据权限组件的核心——`DataPermissionRuleHandler`，它是 MyBatis-Plus `MultiDataPermissionHandler` 接口的实现，负责在 SQL 执行前拦截并改写 SQL，自动追加数据权限过滤条件。

## MultiDataPermissionHandler 接口

MyBatis-Plus 提供了 `MultiDataPermissionHandler` 接口作为数据权限的扩展点：

```java
// MyBatis-Plus 提供的接口
public interface MultiDataPermissionHandler {
    /**
     * 获取数据权限 SQL 片段
     *
     * @param table              表信息
     * @param where              原始 WHERE 条件
     * @param mappedStatementId  Mapper 方法的 ID
     * @return 追加的 WHERE 条件表达式
     */
    Expression getSqlSegment(Table table, Expression where, String mappedStatementId);
}
```

yudao-cloud 的 `DataPermissionRuleHandler` 实现了这个接口，在 `getSqlSegment()` 中根据规则生成 WHERE 条件，MyBatis-Plus 会自动将其拼接到原始 SQL 中。

## DataPermissionRuleHandler 完整实现

```java
// 文件路径：yudao-framework/yudao-spring-boot-starter-biz-data-permission/src/main/java/cn/iocoder/yudao/framework/datapermission/core/db/DataPermissionRuleHandler.java
package cn.iocoder.yudao.framework.datapermission.core.db;

import cn.hutool.core.collection.CollUtil;
import cn.iocoder.yudao.framework.datapermission.core.rule.DataPermissionRule;
import cn.iocoder.yudao.framework.datapermission.core.rule.DataPermissionRuleFactory;
import cn.iocoder.yudao.framework.mybatis.core.util.MyBatisUtils;
import com.baomidou.mybatisplus.extension.plugins.handler.MultiDataPermissionHandler;
import lombok.RequiredArgsConstructor;
import net.sf.jsqlparser.expression.Expression;
import net.sf.jsqlparser.expression.operators.conditional.AndExpression;
import net.sf.jsqlparser.schema.Table;

import java.util.List;

import static cn.iocoder.yudao.framework.security.core.util.SecurityFrameworkUtils.skipPermissionCheck;

/**
 * 基于 {@link DataPermissionRule} 的数据权限处理器
 *
 * 它的底层，是基于 MyBatis Plus 的数据权限插件
 * 核心原理：它会在 SQL 执行前拦截 SQL 语句，并根据用户权限动态添加权限相关的 SQL 片段。
 * 这样，只有用户有权限访问的数据才会被查询出来。
 *
 * @author 芋道源码
 */
@RequiredArgsConstructor
public class DataPermissionRuleHandler implements MultiDataPermissionHandler {

    private final DataPermissionRuleFactory ruleFactory;

    @Override
    public Expression getSqlSegment(Table table, Expression where, String mappedStatementId) {
        // 特殊：跨租户访问
        if (skipPermissionCheck()) {
            return null;
        }

        // 获得 Mapper 对应的数据权限的规则
        List<DataPermissionRule> rules = ruleFactory.getDataPermissionRule(mappedStatementId);
        if (CollUtil.isEmpty(rules)) {
            return null;
        }

        // 生成条件
        Expression allExpression = null;
        for (DataPermissionRule rule : rules) {
            // 判断表名是否匹配
            String tableName = MyBatisUtils.getTableName(table);
            if (!rule.getTableNames().contains(tableName)) {
                continue;
            }

            // 单条规则的条件
            Expression oneExpress = rule.getExpression(tableName, table.getAlias());
            if (oneExpress == null) {
                continue;
            }
            // 拼接到 allExpression 中
            allExpression = allExpression == null ? oneExpress
                    : new AndExpression(allExpression, oneExpress);
        }
        return allExpression;
    }

}
```

## getSqlSegment() 完整流程

```
getSqlSegment(table, where, mappedStatementId)
    │
    ▼
skipPermissionCheck()？ ──是──▶ return null（跳过，不改写 SQL）
    │否
    ▼
ruleFactory.getDataPermissionRule(mappedStatementId)
    │
    ▼
规则列表为空？ ──是──▶ return null（无规则，不改写 SQL）
    │否
    ▼
allExpression = null
    │
    ▼
┌─── 遍历每条规则 ───────────────────────────────────┐
│   │                                                 │
│   ▼                                                 │
│   rule.getTableNames() 包含当前表名？                │
│   │否                                              │
│   │  └──▶ continue（跳过这条规则）                  │
│   │是                                              │
│   ▼                                                 │
│   rule.getExpression(tableName, tableAlias)         │
│   │                                                 │
│   ▼                                                 │
│   表达式为 null？ ──是──▶ continue（跳过）           │
│   │否                                              │
│   ▼                                                 │
│   allExpression == null？                           │
│   │是 → allExpression = oneExpression               │
│   │否 → allExpression = allExpression AND one       │
│   │                                                 │
│   └─── 继续下一条规则 ─────────────────────────────┘
    │
    ▼
return allExpression
```

### 步骤一：skipPermissionCheck() 跳过检查

```java
// 文件路径：yudao-framework/yudao-spring-boot-starter-security/src/main/java/cn/iocoder/yudao/framework/security/core/util/SecurityFrameworkUtils.java
public static boolean skipPermissionCheck() {
    LoginUser loginUser = getLoginUser();
    if (loginUser == null) {
        return false;
    }
    // visitTenantId 为空，说明是跨租户访问（如系统级别的操作）
    if (loginUser.getVisitTenantId() == null) {
        return true;
    }
    return false;
}
```

`skipPermissionCheck()` 的作用是判断是否需要跳过数据权限检查。当 `visitTenantId` 为空时，说明是跨租户访问（如系统级别的操作），此时跳过数据权限检查。

### 步骤二：获取规则列表

```java
List<DataPermissionRule> rules = ruleFactory.getDataPermissionRule(mappedStatementId);
```

通过 `DataPermissionRuleFactory` 获取当前 Mapper 方法应该使用的规则列表。这里会根据 `@DataPermission` 注解上下文进行过滤。

### 步骤三：遍历规则，生成条件

```java
Expression allExpression = null;
for (DataPermissionRule rule : rules) {
    // 判断表名是否匹配
    String tableName = MyBatisUtils.getTableName(table);
    if (!rule.getTableNames().contains(tableName)) {
        continue;
    }

    // 单条规则的条件
    Expression oneExpress = rule.getExpression(tableName, table.getAlias());
    if (oneExpress == null) {
        continue;
    }
    // 拼接到 allExpression 中
    allExpression = allExpression == null ? oneExpress
            : new AndExpression(allExpression, oneExpress);
}
```

关键点：
1. **表名匹配**：只有规则声明的表名包含当前表名时，才会生成条件
2. **条件拼接**：多个规则的条件用 `AND` 连接
3. **空值处理**：如果规则返回 `null`，表示不加条件

## JSQLParser 表达式构建

JSQLParser 是一个 SQL 语句解析器，yudao-cloud 使用它来构建 SQL 表达式。以下是常用的表达式构建方式：

### EqualsTo：等于条件

```java
// 生成：user_id = 42
new EqualsTo(
    MyBatisUtils.buildColumn(tableName, tableAlias, "user_id"),
    new LongValue(42)
);
```

### InExpression：IN 条件

```java
// 生成：dept_id IN (100, 101, 102)
new InExpression(
    MyBatisUtils.buildColumn(tableName, tableAlias, "dept_id"),
    new ParenthesedExpressionList(
        new ExpressionList<LongValue>(
            Arrays.asList(new LongValue(100), new LongValue(101), new LongValue(102))
        )
    )
);
```

### OrExpression：OR 条件

```java
// 生成：dept_id IN (100, 101) OR user_id = 42
new OrExpression(deptExpression, userExpression);
```

### AndExpression：AND 条件

```java
// 生成：条件1 AND 条件2
new AndExpression(expression1, expression2);
```

### ParenthesedExpressionList：带括号的表达式

```java
// 生成：(dept_id IN (100, 101) OR user_id = 42)
new ParenthesedExpressionList(new OrExpression(deptExpression, userExpression));
```

### EqualsTo(null, null)：空结果条件

```java
// 生成：WHERE null = null
// 作用：保证返回的数据为空
new EqualsTo(null, null);
```

## MyBatisUtils.buildColumn() 工具方法

`MyBatisUtils.buildColumn()` 用于构建带表名/别名的列引用：

```java
// 文件路径：yudao-framework/yudao-spring-boot-starter-mybatis/src/main/java/cn/iocoder/yudao/framework/mybatis/core/util/MyBatisUtils.java
public static Column buildColumn(String tableName, Alias tableAlias, String columnName) {
    // 如果有别名，使用别名.列名
    if (tableAlias != null) {
        return new Column(tableAlias.getName(), columnName);
    }
    // 否则使用表名.列名
    return new Column(tableName, columnName);
}
```

生成的 SQL 效果：
```sql
-- 无别名时
dept_id IN (100, 101, 102)

-- 有别名时（如 SELECT * FROM trade_order o）
o.dept_id IN (100, 101, 102)
```

## SQL 改写示例

### 示例一：单规则，单表

```
原始 SQL：
  SELECT * FROM trade_order WHERE status = 1

规则配置：
  DeptDataPermissionRule:
    - trade_order 表的 dept_id 字段
    - 当前用户可见部门：[100, 101, 102]

改写过程：
  1. DataPermissionRuleHandler.getSqlSegment() 被调用
  2. 获取规则列表：[DeptDataPermissionRule]
  3. 遍历规则：
     - DeptDataPermissionRule.getTableNames() 包含 "trade_order"？→ 是
     - DeptDataPermissionRule.getExpression("trade_order", null)
       → 返回：dept_id IN (100, 101, 102)
  4. allExpression = dept_id IN (100, 101, 102)

改写后的 SQL：
  SELECT * FROM trade_order
  WHERE status = 1
    AND dept_id IN (100, 101, 102)
```

### 示例二：多规则，单表

```
原始 SQL：
  SELECT * FROM trade_order WHERE status = 1

规则配置：
  TenantRule:
    - trade_order 表的 tenant_id 字段
    - 当前租户 ID：1
  DeptDataPermissionRule:
    - trade_order 表的 dept_id 字段
    - 当前用户可见部门：[100, 101, 102]

改写过程：
  1. 获取规则列表：[TenantRule, DeptDataPermissionRule]
  2. 遍历规则：
     - TenantRule.getExpression("trade_order", null)
       → 返回：tenant_id = 1
     - DeptDataPermissionRule.getExpression("trade_order", null)
       → 返回：dept_id IN (100, 101, 102)
  3. allExpression = tenant_id = 1 AND dept_id IN (100, 101, 102)

改写后的 SQL：
  SELECT * FROM trade_order
  WHERE status = 1
    AND tenant_id = 1
    AND dept_id IN (100, 101, 102)
```

### 示例三：带表别名的 SQL

```
原始 SQL：
  SELECT o.* FROM trade_order o
  LEFT JOIN trade_customer c ON o.customer_id = c.id
  WHERE o.status = 1

规则配置：
  DeptDataPermissionRule:
    - trade_order 表的 dept_id 字段
    - 当前用户可见部门：[100, 101]

改写后的 SQL：
  SELECT o.* FROM trade_order o
  LEFT JOIN trade_customer c ON o.customer_id = c.id
  WHERE o.status = 1
    AND o.dept_id IN (100, 101)

注意：customer 表如果没有配置 deptColumns/userColumns，不会被过滤
```

### 示例四：禁用数据权限

```
配置：
  @DataPermission(enable = false)

原始 SQL：
  SELECT * FROM trade_order WHERE status = 1

改写后的 SQL：
  （不改写，保持原样）
  SELECT * FROM trade_order WHERE status = 1
```

### 示例五：无权限时返回空结果

```
配置：
  用户既没有部门权限也没有个人权限

原始 SQL：
  SELECT * FROM trade_order WHERE status = 1

改写后的 SQL：
  SELECT * FROM trade_order
  WHERE status = 1
    AND null = null

结果：返回空数据集
```

## 自动配置：注册到 MyBatis-Plus

`DataPermissionRuleHandler` 通过 `YudaoDataPermissionAutoConfiguration` 自动配置注册到 MyBatis-Plus：

```java
// 文件路径：yudao-framework/yudao-spring-boot-starter-biz-data-permission/src/main/java/cn/iocoder/yudao/framework/datapermission/config/YudaoDataPermissionAutoConfiguration.java
package cn.iocoder.yudao.framework.datapermission.config;

import cn.iocoder.yudao.framework.datapermission.core.aop.DataPermissionAnnotationAdvisor;
import cn.iocoder.yudao.framework.datapermission.core.db.DataPermissionRuleHandler;
import cn.iocoder.yudao.framework.datapermission.core.rule.DataPermissionRule;
import cn.iocoder.yudao.framework.datapermission.core.rule.DataPermissionRuleFactory;
import cn.iocoder.yudao.framework.datapermission.core.rule.DataPermissionRuleFactoryImpl;
import cn.iocoder.yudao.framework.mybatis.core.util.MyBatisUtils;
import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.DataPermissionInterceptor;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.context.annotation.Bean;

import java.util.List;

/**
 * 数据权限的自动配置类
 *
 * @author 芋道源码
 */
@AutoConfiguration
public class YudaoDataPermissionAutoConfiguration {

    @Bean
    public DataPermissionRuleFactory dataPermissionRuleFactory(List<DataPermissionRule> rules) {
        return new DataPermissionRuleFactoryImpl(rules);
    }

    @Bean
    public DataPermissionRuleHandler dataPermissionRuleHandler(MybatisPlusInterceptor interceptor,
                                                               DataPermissionRuleFactory ruleFactory) {
        // 创建 DataPermissionInterceptor 拦截器
        DataPermissionRuleHandler handler = new DataPermissionRuleHandler(ruleFactory);
        DataPermissionInterceptor inner = new DataPermissionInterceptor(handler);
        // 添加到 interceptor 中
        // 需要加在首个，主要是为了在分页插件前面。这个是 MyBatis Plus 的规定
        MyBatisUtils.addInterceptor(interceptor, inner, 0);
        return handler;
    }

    @Bean
    public DataPermissionAnnotationAdvisor dataPermissionAnnotationAdvisor() {
        return new DataPermissionAnnotationAdvisor();
    }

}
```

关键点：
1. **`DataPermissionRuleFactory`**：收集所有 `DataPermissionRule` Bean，创建工厂实例
2. **`DataPermissionRuleHandler`**：创建处理器，并注册到 MyBatis-Plus 的拦截器链中
3. **`DataPermissionAnnotationAdvisor`**：创建 AOP 切面，拦截 `@DataPermission` 注解
4. **拦截器顺序**：`MyBatisUtils.addInterceptor(interceptor, inner, 0)` 将数据权限拦截器添加到首位，确保在分页插件之前执行

## 与多租户的协作

在实际运行中，数据权限和多租户是叠加生效的。一个 SQL 最终可能同时被两个组件改写：

```
原始 SQL：SELECT * FROM trade_order WHERE status = 1

多租户改写后：SELECT * FROM trade_order WHERE status = 1 AND tenant_id = 1

数据权限改写后：SELECT * FROM trade_order WHERE status = 1 AND tenant_id = 1 AND dept_id IN (100, 101, 102)
```

两者的执行顺序是：**先多租户，后数据权限**。这在自动配置中通过 `MyBatisUtils.addInterceptor()` 的顺序来保证。

## 思考题

1. `getSqlSegment()` 方法中，如果多个规则都声明了同一个表，它们的条件会如何组合？如果其中一个规则返回 `null`，会发生什么？

2. `skipPermissionCheck()` 方法通过检查 `visitTenantId` 是否为空来判断是否跳过数据权限。这种设计有什么潜在问题？如果业务需要在非跨租户场景下跳过数据权限，应该如何扩展？

3. `DataPermissionRuleHandler` 的 `getSqlSegment()` 方法返回 `null` 和返回 `new EqualsTo(null, null)` 有什么区别？各适用什么场景？
