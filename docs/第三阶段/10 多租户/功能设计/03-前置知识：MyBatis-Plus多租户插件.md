# 03-前置知识：MyBatis-Plus 多租户插件

## 为什么需要 SQL 改写

在共享 Schema 行级隔离方案下，所有租户的数据在同一张表中，通过 `tenant_id` 列区分。如果让业务开发人员每次写 SQL 都手动加上 `WHERE tenant_id = ?`，不仅繁琐，而且极易遗漏——遗漏就意味着数据泄露。

**理想方案**：业务代码写的是"纯净"的 SQL，框架在 SQL 执行前自动追加租户条件。这就是 MyBatis-Plus 多租户插件做的事。

## TenantLineHandler 接口

MyBatis-Plus 提供了 `TenantLineHandler` 接口，开发者通过实现它来告诉插件"如何获取租户 ID"以及"哪些表不需要租户过滤"。

```java
public interface TenantLineHandler {

    /**
     * 获取租户 ID 的值
     * 插件会用这个值来改写 SQL
     *
     * @return 租户 ID 的表达式
     */
    Expression getTenantId();

    /**
     * 获取租户字段名
     * 默认是 "tenant_id"
     *
     * @return 租户字段名
     */
    default String getTenantIdColumn() {
        return "tenant_id";
    }

    /**
     * 是否忽略指定表的租户过滤
     *
     * @param tableName 表名
     * @return true 表示忽略（不追加租户条件），false 表示需要追加
     */
    default boolean ignoreTable(String tableName) {
        return false;
    }
}
```

### 核心方法解读

**`getTenantId()`**：返回当前租户 ID 的表达式。插件会把这个值嵌入到 SQL 中。通常实现为：

```java
@Override
public Expression getTenantId() {
    return new LongValue(TenantContextHolder.getRequiredTenantId());
}
```

**`getTenantIdColumn()`**：返回租户字段名，默认是 `tenant_id`。大多数情况下不需要重写。

**`ignoreTable(String tableName)`**：决定哪些表需要租户过滤。这是最灵活的扩展点，yudao-cloud 在这里实现了三级判断机制（详见后续章节）。

## TenantLineInnerInterceptor 工作原理

`TenantLineInnerInterceptor` 是 MyBatis-Plus 提供的多租户拦截器，它在 SQL 执行前对 SQL 进行改写。工作流程如下：

```
原始 SQL
    │
    ▼
┌─────────────────────────────────────────┐
│ JSQLParser 解析 SQL 为 AST              │
│ (Abstract Syntax Tree，抽象语法树)       │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 判断 SQL 类型                           │
│                                         │
│ SELECT → 追加 WHERE tenant_id = ?       │
│ INSERT → 追加 tenant_id 字段和值        │
│ UPDATE → 追加 WHERE tenant_id = ?       │
│ DELETE → 追加 WHERE tenant_id = ?       │
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│ 调用 ignoreTable() 判断是否跳过         │
│ 如果跳过 → 返回原始 SQL                 │
│ 如果不跳过 → 返回改写后的 SQL           │
└─────────────────────────────────────────┘
    │
    ▼
改写后的 SQL → 执行
```

### SQL 改写示例

#### SELECT 改写

```sql
-- 原始 SQL
SELECT id, name, price FROM product WHERE status = 1;

-- 改写后
SELECT id, name, price FROM product WHERE status = 1 AND tenant_id = 1;
```

如果原始 SQL 已经有 `WHERE` 条件，插件会用 `AND` 追加；如果没有 `WHERE`，会自动添加 `WHERE`。

#### INSERT 改写

```sql
-- 原始 SQL
INSERT INTO product (name, price) VALUES ('iPhone', 5999);

-- 改写后
INSERT INTO product (name, price, tenant_id) VALUES ('iPhone', 5999, 1);
```

插件会自动在字段列表和值列表中追加 `tenant_id`。

#### UPDATE 改写

```sql
-- 原始 SQL
UPDATE product SET price = 4999 WHERE id = 1;

-- 改写后
UPDATE product SET price = 4999 WHERE id = 1 AND tenant_id = 1;
```

#### DELETE 改写

```sql
-- 原始 SQL
DELETE FROM product WHERE id = 1;

-- 改写后
DELETE FROM product WHERE id = 1 AND tenant_id = 1;
```

### 子查询的处理

插件会递归处理子查询中的表引用：

```sql
-- 原始 SQL
SELECT * FROM product WHERE category_id IN (SELECT id FROM category WHERE status = 1);

-- 改写后（product 和 category 都会追加租户条件）
SELECT * FROM product WHERE category_id IN (SELECT id FROM category WHERE status = 1 AND tenant_id = 1)
AND tenant_id = 1;
```

### 多表 JOIN 的处理

```sql
-- 原始 SQL
SELECT p.*, c.name AS category_name
FROM product p
LEFT JOIN category c ON p.category_id = c.id
WHERE p.status = 1;

-- 改写后（两张表都追加租户条件）
SELECT p.*, c.name AS category_name
FROM product p
LEFT JOIN category c ON p.category_id = c.id AND c.tenant_id = 1
WHERE p.status = 1 AND p.tenant_id = 1;
```

## JSQLParser 的作用

JSQLParser 是一个开源的 SQL 解析库，它能将 SQL 字符串解析为抽象语法树（AST），然后对 AST 进行修改，最后再生成 SQL 字符串。

```
SQL 字符串                    AST (抽象语法树)
"SELECT * FROM product"  →   Select
  WHERE status = 1           ├── Table: product
"                           ├── Where: status = 1
                             └── (可修改、可追加)
```

**为什么用 AST 而不是正则表达式？**

SQL 语法复杂，用正则表达式改写 SQL 极易出错。例如：

```sql
-- 正则难以正确处理的情况
SELECT * FROM product WHERE name = 'WHERE' AND status = 1;
-- 如果用正则找 "WHERE"，会错误匹配到字符串值中的 'WHERE'
```

JSQLParser 通过语法分析，能准确识别 SQL 的各个组成部分，避免这类问题。

**性能考虑**：SQL 解析是有成本的。MyBatis-Plus 内部对解析结果做了缓存，相同的 SQL 模板只解析一次。

## 在 yudao-cloud 中的注册

在 `YudaoTenantAutoConfiguration` 中，多租户拦截器被注册到 MyBatis-Plus 的拦截器链中：

```java
// 源码位置：yudao-framework/yudao-spring-boot-starter-biz-tenant/.../YudaoTenantAutoConfiguration.java

@Bean
public TenantLineInnerInterceptor tenantLineInnerInterceptor(TenantProperties properties,
                                                             MybatisPlusInterceptor interceptor) {
    TenantLineInnerInterceptor inner = new TenantLineInnerInterceptor(new TenantDatabaseInterceptor(properties));
    // 需要加在首个，主要是为了在分页插件前面。这个是 MyBatis Plus 的规定
    MyBatisUtils.addInterceptor(interceptor, inner, 0);
    return inner;
}
```

**关键点**：多租户拦截器必须添加到拦截器链的**第一个位置**，在分页插件之前。这是 MyBatis-Plus 的规定——分页插件需要在 SQL 改写之后才能正确计算总数。

## 与 Spring AOP 的对比

有些框架使用 Spring AOP 来实现多租户，例如在 Service 方法上加注解，手动拼接查询条件。这两种方式的对比：

| 维度 | MyBatis-Plus 拦截器 | Spring AOP |
|------|-------------------|-----------|
| **透明度** | 完全透明，业务无感知 | 需要配合特定的查询方式 |
| **覆盖范围** | 所有 SQL 操作 | 只能拦截 Service 方法 |
| **SQL 改写精度** | 基于 AST，精确到表和字段 | 通常只能在查询条件上做文章 |
| **动态 SQL** | 支持（MyBatis 的 XML/注解） | 难以处理复杂的动态 SQL |
| **性能** | JSQLParser 解析 + 缓存 | 反射调用 |

yudao-cloud 选择 MyBatis-Plus 拦截器方案，正是因为它的**透明度最高、覆盖范围最广**。

## 小结

MyBatis-Plus 的多租户插件通过 `TenantLineHandler` 接口和 `TenantLineInnerInterceptor` 拦截器，在 SQL 执行前自动改写 SQL，追加租户条件。JSQLParser 负责将 SQL 解析为 AST 进行精确修改。这为 yudao-cloud 的多租户方案提供了**数据层自动隔离**的基础能力。

下一章我们将看到，yudao-cloud 是如何实现 `TenantLineHandler` 接口的，以及它的"三级判断"排除机制是如何工作的。
