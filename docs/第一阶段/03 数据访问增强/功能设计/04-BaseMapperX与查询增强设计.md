# BaseMapperX 与查询增强设计

---

## 1. 从一个分页查询的"样板代码"说起

回顾上一章的问题：一个分页查询方法中，空值判断占了代码量的一半。但 `*IfPresent` 只解决了查询条件的问题——分页查询还有更多的"样板代码"。

看看如果每个 Mapper 都手写分页逻辑，会是什么样：

```java
// 第一个 Mapper：用户
default PageResult<AdminUserDO> selectPage(UserPageReqVO reqVO) {
    Page<AdminUserDO> page = new Page<>(reqVO.getPageNo(), reqVO.getPageSize());
    LambdaQueryWrapper<AdminUserDO> wrapper = new LambdaQueryWrapper<AdminUserDO>()
        .likeIfPresent(AdminUserDO::getUsername, reqVO.getUsername())
        .eqIfPresent(AdminUserDO::getStatus, reqVO.getStatus())
        .orderByDesc(AdminUserDO::getId);
    Page<AdminUserDO> result = selectPage(page, wrapper);
    return new PageResult<>(result.getRecords(), result.getTotal()); // 重复的转换
}

// 第二个 Mapper：商品
default PageResult<ProductDO> selectPage(ProductPageReqVO reqVO) {
    Page<ProductDO> page = new Page<>(reqVO.getPageNo(), reqVO.getPageSize());
    LambdaQueryWrapper<ProductDO> wrapper = new LambdaQueryWrapper<ProductDO>()
        .likeIfPresent(ProductDO::getName, reqVO.getName())
        .eqIfPresent(ProductDO::getStatus, reqVO.getStatus())
        .orderByDesc(ProductDO::getId);
    Page<ProductDO> result = selectPage(page, wrapper);
    return new PageResult<>(result.getRecords(), result.getTotal()); // 同样的转换
}
```

`new Page<>(...)` 和 `new PageResult<>(...)` 这两行在每个分页方法中都出现。50 个 Mapper 就是 100 行重复代码。

更关键的是"不分页"场景：导出 Excel 时需要查询全部数据（`pageSize = -1`），但原生的 `selectPage` 不支持这种用法，需要单独写一个 `selectList` 调用。

---

## 2. BaseMapperX 的设计

`BaseMapperX` 是对 MyBatis-Plus `BaseMapper` 的扩展，所有 yudao-cloud 的 Mapper 都继承它。

```
BaseMapper（MyBatis-Plus 提供）
  └── MPJBaseMapper（mybatis-plus-join 提供，增加连表能力）
        └── BaseMapperX（yudao-cloud 提供，增加分页/快捷查询/批量操作）
```

### 2.1 分页查询：selectPage

```java
// 源码位置：cn.iocoder.yudao.framework.mybatis.core.mapper.BaseMapperX
default PageResult<T> selectPage(PageParam pageParam, @Param("ew") Wrapper<T> queryWrapper) {
    return selectPage(pageParam, null, queryWrapper);
}

default PageResult<T> selectPage(PageParam pageParam, Collection<SortingField> sortingFields,
                                  @Param("ew") Wrapper<T> queryWrapper) {
    // 特殊：不分页，直接查询全部
    if (PageParam.PAGE_SIZE_NONE.equals(pageParam.getPageSize())) {
        MyBatisUtils.addOrder(queryWrapper, sortingFields);
        List<T> list = selectList(queryWrapper);
        return new PageResult<>(list, (long) list.size());
    }

    // MyBatis Plus 查询
    IPage<T> mpPage = MyBatisUtils.buildPage(pageParam, sortingFields);
    selectPage(mpPage, queryWrapper);
    return new PageResult<>(mpPage.getRecords(), mpPage.getTotal());
}
```

设计要点：

1. **直接返回 `PageResult<T>`**。调用方不需要再做 `Page` 到 `PageResult` 的转换。

2. **自动处理 `PAGE_SIZE_NONE`**。当 `pageSize = -1` 时，跳过分页插件，直接查询全部数据。这统一了"分页"和"不分页"两种场景的调用方式。

3. **支持动态排序**。通过 `SortablePageParam` 传入排序字段，在构建 `Page` 对象时自动添加 `ORDER BY`。`MyBatisUtils.buildPage` 内部会将驼峰字段名转为下划线，并做安全校验防止 SQL 注入。

有了 `BaseMapperX`，分页查询简化为：

```java
default PageResult<AdminUserDO> selectPage(UserPageReqVO reqVO) {
    return selectPage(reqVO, new LambdaQueryWrapperX<AdminUserDO>()
        .likeIfPresent(AdminUserDO::getUsername, reqVO.getUsername())
        .eqIfPresent(AdminUserDO::getStatus, reqVO.getStatus())
        .orderByDesc(AdminUserDO::getId));
}
```

`reqVO` 同时作为分页参数（包含 pageNo/pageSize）和查询条件的载体。

### 2.2 快捷查询方法

BaseMapperX 提供了多种快捷查询方法，覆盖最常见的查询模式：

**单字段查询**：

```java
// 原来要写：
selectOne(new LambdaQueryWrapper<AdminUserDO>().eq(AdminUserDO::getUsername, "admin"));

// 现在一行：
selectOne(AdminUserDO::getUsername, "admin");
```

**双字段查询**：

```java
// 按租户 ID + 用户名查询（确保租户内唯一）
selectOne(AdminUserDO::getTenantId, 1L, AdminUserDO::getUsername, "admin");
```

**三字段查询**：

```java
// 按租户 + 部门 + 状态查询
selectOne(AdminUserDO::getTenantId, 1L, AdminUserDO::getDeptId, 100L,
          AdminUserDO::getStatus, 0);
```

**IN 查询**：

```java
// 查询指定部门的用户
selectList(AdminUserDO::getDeptId, Arrays.asList(1L, 2L, 3L));
```

这些方法在 `AdminUserMapper` 中的实际使用：

```java
@Mapper
public interface AdminUserMapper extends BaseMapperX<AdminUserDO> {

    default AdminUserDO selectByUsername(String username) {
        return selectOne(AdminUserDO::getUsername, username);
    }

    default AdminUserDO selectByEmail(String email) {
        return selectOne(AdminUserDO::getEmail, email);
    }

    default AdminUserDO selectByMobile(String mobile) {
        return selectOne(AdminUserDO::getMobile, mobile);
    }

    default List<AdminUserDO> selectListByStatus(Integer status) {
        return selectList(AdminUserDO::getStatus, status);
    }

    default List<AdminUserDO> selectListByDeptIds(Collection<Long> deptIds) {
        return selectList(AdminUserDO::getDeptId, deptIds);
    }
}
```

### 2.3 selectOneForUpdate：悲观锁查询

在并发场景下（如扣减库存、修改订单状态），需要使用悲观锁确保数据一致性：

```java
default T selectOneForUpdate(SFunction<T, ?> field, Object value) {
    return selectOneForUpdate(new LambdaQueryWrapper<T>().eq(field, value));
}

default T selectOneForUpdate(LambdaQueryWrapper<T> queryWrapper) {
    return selectOne(queryWrapper.last("FOR UPDATE"));
}
```

它在查询 SQL 末尾追加 `FOR UPDATE`，锁定匹配的行，直到当前事务提交后其他事务才能修改。

使用示例：

```java
@Transactional
public void deductStock(Long skuId, Integer quantity) {
    // SELECT * FROM product_sku WHERE id = 123 FOR UPDATE
    ProductSkuDO sku = productSkuMapper.selectOneForUpdate(ProductSkuDO::getId, skuId);
    if (sku.getStock() < quantity) {
        throw new ServiceException(PRODUCT_SKU_STOCK_NOT_ENOUGH);
    }
    sku.setStock(sku.getStock() - quantity);
    productSkuMapper.updateById(sku);
}
```

注意：`FOR UPDATE` 必须在事务中使用（`@Transactional`），否则锁会立即释放。

### 2.4 selectFirstOne：并发安全的查询

```java
default T selectFirstOne(SFunction<T, ?> field, Object value) {
    List<T> list = selectList(new LambdaQueryWrapper<T>().eq(field, value));
    return CollUtil.getFirst(list);
}
```

为什么需要 `selectFirstOne`？考虑这个并发场景：

1. 线程 A 和线程 B 同时注册同一个用户名
2. 线程 A 调用 `selectByUsername`，发现用户不存在
3. 线程 B 也调用 `selectByUsername`，也发现用户不存在
4. 线程 A 插入用户
5. 线程 B 也插入用户（此时有两条同名记录）
6. 线程 A 再调用 `selectByUsername`，`selectOne` 报错：期望 1 条记录，实际查到 2 条

`selectFirstOne` 使用 `selectList` + `CollUtil.getFirst` 代替 `selectOne`，即使有多条记录也不会抛异常。在并发场景下，这种"安全第一"的策略更可靠。

### 2.5 批量操作

```java
default Boolean insertBatch(Collection<T> entities) {
    // 特殊：SQL Server 批量插入后，获取 id 会报错，因此通过循环处理
    DbType dbType = JdbcUtils.getDbType();
    if (JdbcUtils.isSQLServer(dbType)) {
        entities.forEach(this::insert);
        return CollUtil.isNotEmpty(entities);
    }
    return Db.saveBatch(entities);
}
```

批量操作的设计要点：

1. **SQL Server 兼容**。SQL Server 批量插入后获取自增 ID 会报错，所以降级为逐条插入。这是"多数据库兼容"在代码层面的体现。

2. **支持自定义批次大小**。`insertBatch(entities, size)` 允许控制每批插入的记录数（默认 1000），避免一次性插入过多数据导致内存溢出。

3. **使用 `Db.saveBatch`**。这是 MyBatis-Plus 提供的批量操作工具，在内部使用 JDBC Batch 模式，比逐条插入高效得多。

其他批量方法：

```java
// 全表批量更新（无 WHERE 条件）
default int updateBatch(T update) {
    return update(update, new QueryWrapper<>());
}

// 按 ID 批量更新
default Boolean updateBatch(Collection<T> entities) {
    return Db.updateBatchById(entities);
}

// 按字段 IN 批量删除
default int deleteBatch(SFunction<T, ?> field, Collection<?> values) {
    if (CollUtil.isEmpty(values)) {
        return 0;
    }
    return delete(new LambdaQueryWrapper<T>().in(field, values));
}
```

---

## 3. LambdaQueryWrapperX：*IfPresent 模式

`LambdaQueryWrapperX` 扩展了 MyBatis-Plus 的 `LambdaQueryWrapper`，核心增加是 **`*IfPresent` 系列方法**。

### 3.1 问题：空值条件的噪音

没有 `*IfPresent` 时：

```java
new LambdaQueryWrapper<AdminUserDO>()
    .eq(reqVO.getStatus() != null, AdminUserDO::getStatus, reqVO.getStatus())
    .like(StringUtils.hasText(reqVO.getUsername()), AdminUserDO::getUsername, reqVO.getUsername())
    .between(reqVO.getBeginCreateTime() != null && reqVO.getEndCreateTime() != null,
             AdminUserDO::getCreateTime, reqVO.getBeginCreateTime(), reqVO.getEndCreateTime());
```

MyBatis-Plus 原生支持 `eq(boolean condition, ...)` 的条件判断，但写起来仍然啰嗦——每个方法都要传一个 `boolean` 参数。

### 3.2 *IfPresent 的解决方案

```java
new LambdaQueryWrapperX<AdminUserDO>()
    .eqIfPresent(AdminUserDO::getStatus, reqVO.getStatus())
    .likeIfPresent(AdminUserDO::getUsername, reqVO.getUsername())
    .betweenIfPresent(AdminUserDO::getCreateTime, reqVO.getBeginCreateTime(), reqVO.getEndCreateTime());
```

`*IfPresent` 方法内部自动判断值是否为空，为空则跳过该条件。各方法的判断逻辑：

| 方法 | 判断条件 | 含义 |
|------|---------|------|
| `eqIfPresent` | `ObjectUtil.isNotEmpty(val)` | 值不为 null 时才加等值条件 |
| `neIfPresent` | `ObjectUtil.isNotEmpty(val)` | 值不为 null 时才加不等条件 |
| `likeIfPresent` | `StringUtils.hasText(val)` | 字符串非空非空白时才加 LIKE 条件 |
| `likeRightIfPresent` | `StringUtils.hasText(val)` | 右模糊匹配 |
| `inIfPresent` | `ObjectUtil.isAllNotEmpty(values)` 且非空集合 | 集合非空时才加 IN 条件 |
| `gtIfPresent` / `geIfPresent` | `val != null` | 大于 / 大于等于 |
| `ltIfPresent` / `leIfPresent` | `val != null` | 小于 / 小于等于 |
| `betweenIfPresent` | 智能处理（见下文） | 范围查询 |

### 3.3 betweenIfPresent 的智能处理

`betweenIfPresent` 特别值得注意：

```java
public LambdaQueryWrapperX<T> betweenIfPresent(SFunction<T, ?> column, Object val1, Object val2) {
    if (val1 != null && val2 != null) {
        return (LambdaQueryWrapperX<T>) super.between(column, val1, val2);
    }
    if (val1 != null) {
        return (LambdaQueryWrapperX<T>) ge(column, val1);  // 只有开始值 → >=
    }
    if (val2 != null) {
        return (LambdaQueryWrapperX<T>) le(column, val2);  // 只有结束值 → <=
    }
    return this;  // 都为空 → 不加条件
}
```

在电商后台中，"创建时间范围"是一个常见查询条件。用户可能：
- 只选开始时间（查询"从某天之后"的数据）
- 只选结束时间（查询"到某天之前"的数据）
- 两个都选（查询某个时间段的数据）
- 两个都不选（不按时间筛选）

`betweenIfPresent` 自动处理了这四种情况，不需要业务代码写 if-else。

### 3.4 链式调用的类型安全

`LambdaQueryWrapperX` 重写了父类的 `eq`、`in`、`orderByDesc` 等方法，返回类型从 `LambdaQueryWrapper` 改为 `LambdaQueryWrapperX`：

```java
@Override
public LambdaQueryWrapperX<T> eq(SFunction<T, ?> column, Object val) {
    super.eq(column, val);
    return this;
}
```

这确保了链式调用时始终返回 `LambdaQueryWrapperX` 类型，可以继续调用 `*IfPresent` 方法。如果不重写，链式调用到 `eq` 后就失去了 `*IfPresent` 能力。

---

## 4. QueryWrapperX：字符串方式的增强

除了 `LambdaQueryWrapperX`，还有一个 `QueryWrapperX`——基于字符串字段名的增强版本：

```java
public class QueryWrapperX<T> extends QueryWrapper<T> {
    public QueryWrapperX<T> likeIfPresent(String column, String val) { ... }
    public QueryWrapperX<T> eqIfPresent(String column, Object val) { ... }
    // ... 同样的 *IfPresent 方法
}
```

它额外提供了一个跨数据库的 `limitN` 方法：

```java
public QueryWrapperX<T> limitN(int n) {
    DbType dbType = JdbcUtils.getDbType();
    switch (dbType) {
        case ORACLE:
        case ORACLE_12C:
            super.le("ROWNUM", n);
            break;
        case SQL_SERVER:
        case SQL_SERVER2005:
            super.select("TOP " + n + " *");
            break;
        default:
            super.last("LIMIT " + n);
    }
    return this;
}
```

不同数据库"取前 N 条"的语法不同，`limitN` 封装了这个差异。

---

## 5. 查询增强的整体关系

```
LambdaQueryWrapperX（lambda 方式，推荐）
├── 继承 LambdaQueryWrapper（MyBatis-Plus）
├── 新增 *IfPresent 方法
├── 重写 eq/in/orderByDesc 等方法返回 LambdaQueryWrapperX（支持链式）
│
QueryWrapperX（字符串方式）
├── 继承 QueryWrapper（MyBatis-Plus）
├── 新增 *IfPresent 方法
├── 新增 limitN 方法（跨数据库分页）
│
MPJLambdaWrapperX（连表方式）
├── 继承 MPJLambdaWrapper（mybatis-plus-join）
└── 新增 *IfPresent 方法

BaseMapperX
├── 继承 MPJBaseMapper（mybatis-plus-join）
├── selectPage → 返回 PageResult，支持 PAGE_SIZE_NONE
├── selectOne → 快捷查询（1/2/3 字段）
├── selectOneForUpdate → 悲观锁查询
├── selectFirstOne → 并发安全查询
├── insertBatch / updateBatch / deleteBatch → 批量操作
└── selectList / selectCount → 快捷方法
```

实际使用中，绝大多数场景使用 `LambdaQueryWrapperX`（类型安全），`QueryWrapperX` 用于需要动态列名的少数场景。

---

## 思考题

1. `selectOne(field, value)` 在查到多条记录时会抛异常（MyBatis-Plus 的默认行为），而 `selectFirstOne` 会返回第一条。你在设计 Mapper 方法时，如何决定使用哪一个？
2. `betweenIfPresent` 在只有一个值非 null 时，自动降级为 `ge` 或 `le`。如果业务上要求"两个值必须同时传或同时不传"，你会怎么实现？
3. `insertBatch` 对 SQL Server 降级为逐条插入。如果数据量很大（比如 10000 条），这种降级会带来什么性能问题？你有什么改进思路？
4. `LambdaQueryWrapperX` 重写了父类方法以保持返回类型。如果 MyBatis-Plus 升级后新增了方法但 `LambdaQueryWrapperX` 没有重写，会发生什么？这种继承方案的风险是什么？
