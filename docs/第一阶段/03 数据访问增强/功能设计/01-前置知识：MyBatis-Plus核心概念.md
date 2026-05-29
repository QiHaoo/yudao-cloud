# 前置知识：MyBatis-Plus 核心概念

---

## 1. 为什么需要了解 MyBatis-Plus

在 yudao-cloud 的数据访问层，所有数据库操作都基于 MyBatis-Plus。框架在 MyBatis-Plus 之上做了进一步封装（`BaseMapperX`、`LambdaQueryWrapperX` 等），如果不理解 MyBatis-Plus 的核心概念，就无法理解这些封装"为什么要这么做"。

本章不打算全面介绍 MyBatis-Plus 的所有功能——那是官方文档的工作。我们只聚焦 yudao-cloud 实际用到的部分，让你具备足够的知识去理解后续的设计。

---

## 2. MyBatis-Plus 是什么

MyBatis 是一个半自动的 ORM 框架：你需要手写 SQL（或 XML 映射文件），MyBatis 帮你把结果映射成 Java 对象。它灵活，但每个表的单表 CRUD 都要写一遍 XML，非常重复。

MyBatis-Plus 在 MyBatis 之上做了一层增强：**单表 CRUD 零 SQL**。你只需要定义实体类和 Mapper 接口，就能直接调用 `selectById`、`insert`、`updateById`、`deleteById` 等方法，不需要写任何 XML。

```
传统 MyBatis 的工作流：

  实体类 + Mapper 接口 + XML 映射文件 → MyBatis → 数据库
      ↑
      手写 SQL，每张表都要写

MyBatis-Plus 的工作流：

  实体类 + Mapper 接口 → MyBatis-Plus → 数据库
      ↑
      单表 CRUD 自动生成，多表查询仍可手写
```

### 为什么选 MyBatis-Plus 而不是 JPA

Java 世界里另一个流行的 ORM 是 Spring Data JPA（基于 Hibernate）。两者的核心区别：

| 维度 | MyBatis-Plus | Spring Data JPA |
|------|-------------|----------------|
| SQL 控制 | 显式（你写 SQL，或由框架生成） | 隐式（框架自动生成） |
| 复杂查询 | 手写 SQL，完全可控 | JPQL 或 Criteria API，学习成本高 |
| 性能调优 | 直接看 SQL，容易优化 | 需要理解 Hibernate 的缓存、懒加载机制 |
| 学习曲线 | 会 SQL 就能上手 | 需要理解 ORM 映射、实体状态、关联关系 |
| 国内生态 | 极其丰富，中文文档完善 | 文档以英文为主 |

yudao-cloud 选择 MyBatis-Plus 的核心原因：**SQL 可控**。电商系统中大量复杂查询（多表关联、动态条件、跨数据库兼容），手写 SQL 比让框架猜更可靠。而 MyBatis-Plus 解决了 MyBatis 的"单表 CRUD 重复"痛点，同时保留了手写 SQL 的灵活性。

---

## 3. 实体类注解

MyBatis-Plus 通过注解把 Java 类和数据库表关联起来。

### @TableName — 表名映射

```java
@TableName("system_users")
public class AdminUserDO extends TenantBaseDO {
    // ...
}
```

如果不写 `@TableName`，默认用类名驼峰转下划线作为表名。但 yudao-cloud 中所有 DO 都显式标注，避免歧义。

### @TableId — 主键

```java
@TableId
private Long id;
```

主键字段用 `@TableId` 标注。MyBatis-Plus 支持多种主键策略：

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `AUTO` | 数据库自增 | MySQL |
| `INPUT` | 用户自己设置 ID | Oracle、PostgreSQL（用序列生成） |
| `ASSIGN_ID` | 雪花算法生成 Long ID | 通用，不依赖数据库 |
| `NONE` | 无策略，由 `IdTypeEnvironmentPostProcessor` 自动决定 | yudao-cloud 的默认配置 |

yudao-cloud 配置 `id-type: NONE`，然后由 `IdTypeEnvironmentPostProcessor` 在启动时根据实际数据库类型自动切换（详见第 5 章）。

对于 Oracle、PostgreSQL 等使用序列的数据库，还需要配合 `@KeySequence` 注解：

```java
@TableName("system_users")
@KeySequence("system_users_seq")  // Oracle/PostgreSQL 的序列名
public class AdminUserDO extends TenantBaseDO {
    @TableId
    private Long id;
}
```

### @TableField — 字段映射

大多数情况下，字段名驼峰转下划线后就是列名，不需要额外标注。但有些场景需要 `@TableField`：

```java
// 1. 使用 TypeHandler 存储复杂类型（JSON 序列化）
@TableField(typeHandler = JacksonTypeHandler.class)
private Set<Long> postIds;

// 2. 指定 JDBC 类型（避免类型推断错误）
@TableField(fill = FieldFill.INSERT, jdbcType = JdbcType.VARCHAR)
private String creator;

// 3. 标记非数据库字段
@TableField(exist = false)
private String tempField;
```

### @TableLogic — 逻辑删除

```java
@TableLogic
private Boolean deleted;
```

标注后，MyBatis-Plus 的 `deleteById` 不再执行 `DELETE FROM table WHERE id = ?`，而是执行 `UPDATE table SET deleted = true WHERE id = ?`。所有查询也会自动追加 `WHERE deleted = false` 条件。

yudao-cloud 中所有 DO 都继承自 `BaseDO`，其中已经定义了 `deleted` 字段并标注了 `@TableLogic`，所以业务 DO 不需要重复处理。

---

## 4. BaseMapper — 单表 CRUD 接口

MyBatis-Plus 的核心接口是 `BaseMapper<T>`，它为每张表提供了开箱即用的 CRUD 方法：

```java
public interface BaseMapper<T> {
    int insert(T entity);
    int deleteById(Serializable id);
    int updateById(T entity);
    T selectById(Serializable id);
    List<T> selectList(Wrapper<T> queryWrapper);
    <P extends IPage<T>> P selectPage(P page, Wrapper<T> queryWrapper);
    // ... 更多方法
}
```

你只需要写一个空接口继承它：

```java
@Mapper
public interface AdminUserMapper extends BaseMapper<AdminUserDO> {
    // 继承了一整套 CRUD 方法，零代码
}
```

在 yudao-cloud 中，Mapper 继承的不是 `BaseMapper`，而是 `BaseMapperX`（它又继承了 `MPJBaseMapper`）。`BaseMapperX` 在 `BaseMapper` 基础上增加了分页、批量操作等能力，详见第 4 章。

---

## 5. LambdaQueryWrapper — 类型安全的查询条件

查询时需要构建 WHERE 条件。MyBatis-Plus 提供了两种方式：

### 字符串方式（QueryWrapper）

```java
QueryWrapper<User> wrapper = new QueryWrapper<>();
wrapper.eq("username", "admin")
       .like("nickname", "张")
       .gt("create_time", startTime);
```

问题：字段名是字符串，拼错了编译不报错，运行时才报错。

### Lambda 方式（LambdaQueryWrapper）

```java
LambdaQueryWrapper<AdminUserDO> wrapper = new LambdaQueryWrapper<>();
wrapper.eq(AdminUserDO::getUsername, "admin")
       .like(AdminUserDO::getNickname, "张")
       .gt(AdminUserDO::getCreateTime, startTime);
```

字段通过方法引用（`AdminUserDO::getUsername`）传递，拼写错误编译期就能发现。这就是"类型安全"的含义。

yudao-cloud 几乎全部使用 Lambda 方式，只有少数遗留代码用字符串方式。

### 链式调用

LambdaQueryWrapper 的所有方法都返回 `this`，支持链式调用：

```java
new LambdaQueryWrapper<AdminUserDO>()
    .eq(AdminUserDO::getStatus, 0)
    .like(AdminUserDO::getNickname, "张")
    .orderByDesc(AdminUserDO::getId);
```

---

## 6. 分页插件

MyBatis-Plus 的分页功能需要通过插件启用。在 `YudaoMybatisAutoConfiguration` 中注册：

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor()); // 分页插件
    return interceptor;
}
```

启用后，可以这样使用分页：

```java
Page<AdminUserDO> page = new Page<>(1, 10); // 第 1 页，每页 10 条
Page<AdminUserDO> result = mapper.selectPage(page, wrapper);
// result.getRecords() → 当前页数据
// result.getTotal()    → 总记录数
```

`MybatisPlusInterceptor` 是一个拦截器链，除了分页插件，还可以添加其他插件：

- `PaginationInnerInterceptor` — 分页
- `TenantLineInnerInterceptor` — 多租户（第 10 章会讲）
- `DataPermissionInterceptor` — 数据权限（第 11 章会讲）
- `BlockAttackInnerInterceptor` — 防全表更新/删除

yudao-cloud 默认只开启了分页插件，其他按需开启。

---

## 7. MetaObjectHandler — 自动填充

数据库表中有一些通用字段：创建时间、更新时间、创建人、更新人。这些字段不应该由业务代码手动赋值，而应该在插入/更新时自动填充。

MyBatis-Plus 提供了 `MetaObjectHandler` 接口：

```java
public interface MetaObjectHandler {
    void insertFill(MetaObject metaObject);  // 插入时自动填充
    void updateFill(MetaObject metaObject);  // 更新时自动填充
}
```

在实体字段上标注 `@TableField(fill = ...)` 来声明哪些字段需要自动填充：

```java
@TableField(fill = FieldFill.INSERT)          // 仅插入时填充
private LocalDateTime createTime;

@TableField(fill = FieldFill.INSERT_UPDATE)   // 插入和更新时都填充
private LocalDateTime updateTime;
```

yudao-cloud 的 `DefaultDBFieldHandler` 实现了这个接口，具体实现详见第 3 章。

---

## 8. MyBatis-Plus Join — 连表查询

原生 MyBatis-Plus 只支持单表查询，连表查询需要手写 SQL 或 XML。`mybatis-plus-join` 是一个社区扩展，为 MyBatis-Plus 增加了连表查询能力：

```java
MPJLambdaWrapper<AdminUserDO> wrapper = new MPJLambdaWrapper<AdminUserDO>()
    .selectAll(AdminUserDO.class)
    .select(DeptDO::getName)
    .leftJoin(DeptDO.class, DeptDO::getId, AdminUserDO::getDeptId)
    .eq(AdminUserDO::getStatus, 0);
```

yudao-cloud 的 `BaseMapperX` 继承了 `MPJBaseMapper`，因此所有 Mapper 都具备连表查询能力。

---

## 9. 小结

| 概念 | 作用 | yudao-cloud 中的使用 |
|------|------|---------------------|
| `@TableName` | 表名映射 | 所有 DO 显式标注 |
| `@TableId` | 主键策略 | 配合 `IdTypeEnvironmentPostProcessor` 自动选择 |
| `@TableField` | 字段映射、自动填充 | 用于 JSON 字段、审计字段 |
| `@TableLogic` | 逻辑删除 | `BaseDO.deleted` 统一处理 |
| `BaseMapper` | 单表 CRUD | 被 `BaseMapperX` 继承扩展 |
| `LambdaQueryWrapper` | 类型安全查询 | 被 `LambdaQueryWrapperX` 扩展 |
| `MybatisPlusInterceptor` | 插件链 | 默认注册分页插件 |
| `MetaObjectHandler` | 自动填充 | `DefaultDBFieldHandler` 实现 |

---

## 思考题

1. MyBatis-Plus 的 `BaseMapper` 已经提供了 `selectById`、`selectList` 等方法，为什么 yudao-cloud 还要在 `BaseMapperX` 中扩展 `selectOne(field, value)` 这样的快捷方法？
2. `LambdaQueryWrapper` 相比 `QueryWrapper` 的核心优势是什么？如果你的团队成员 Java 基础较弱，你会选择哪种方式？
3. `MetaObjectHandler` 的自动填充是在 MyBatis 层面完成的，如果业务代码直接通过 JDBC 操作数据库（绕过 MyBatis），审计字段会怎样？
