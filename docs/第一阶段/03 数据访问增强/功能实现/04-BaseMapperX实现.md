# BaseMapperX 实现

---

本章实现 Mapper 增强接口 `BaseMapperX`。它继承 MyBatis-Plus 的 `MPJBaseMapper`，增加了分页查询返回 `PageResult`、单字段/多字段快捷查询、批量操作等能力。

> 设计原理请回顾：[04-BaseMapperX与查询增强设计](../功能设计/04-BaseMapperX与查询增强设计.md)

## 1. BaseMapperX 完整实现

> 源文件：`cn.iocoder.yudao.framework.mybatis.core.mapper.BaseMapperX`

```java
public interface BaseMapperX<T> extends MPJBaseMapper<T> {

    // ========== 分页查询 ==========

    default PageResult<T> selectPage(SortablePageParam pageParam, @Param("ew") Wrapper<T> queryWrapper) {
        return selectPage(pageParam, pageParam.getSortingFields(), queryWrapper);
    }

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
        // 转换返回
        return new PageResult<>(mpPage.getRecords(), mpPage.getTotal());
    }

    // ========== 联表分页查询 ==========

    default <D> PageResult<D> selectJoinPage(PageParam pageParam, Class<D> clazz,
                                              MPJLambdaWrapper<T> lambdaWrapper) {
        if (PageParam.PAGE_SIZE_NONE.equals(pageParam.getPageSize())) {
            List<D> list = selectJoinList(clazz, lambdaWrapper);
            return new PageResult<>(list, (long) list.size());
        }
        IPage<D> mpPage = MyBatisUtils.buildPage(pageParam);
        mpPage = selectJoinPage(mpPage, clazz, lambdaWrapper);
        return new PageResult<>(mpPage.getRecords(), mpPage.getTotal());
    }

    default <D> PageResult<D> selectJoinPage(SortablePageParam pageParam, Class<D> clazz,
                                              MPJLambdaWrapper<T> lambdaWrapper) {
        if (PageParam.PAGE_SIZE_NONE.equals(pageParam.getPageSize())) {
            List<D> list = selectJoinList(clazz, lambdaWrapper);
            return new PageResult<>(list, (long) list.size());
        }
        IPage<D> mpPage = MyBatisUtils.buildPage(pageParam, pageParam.getSortingFields());
        mpPage = selectJoinPage(mpPage, clazz, lambdaWrapper);
        return new PageResult<>(mpPage.getRecords(), mpPage.getTotal());
    }

    default <DTO> PageResult<DTO> selectJoinPage(PageParam pageParam, Class<DTO> resultTypeClass,
                                                  MPJBaseJoin<T> joinQueryWrapper) {
        IPage<DTO> mpPage = MyBatisUtils.buildPage(pageParam);
        selectJoinPage(mpPage, resultTypeClass, joinQueryWrapper);
        return new PageResult<>(mpPage.getRecords(), mpPage.getTotal());
    }

    // ========== 单字段 / 多字段快捷查询 ==========

    default T selectOne(String field, Object value) {
        return selectOne(new QueryWrapper<T>().eq(field, value));
    }

    default T selectOne(SFunction<T, ?> field, Object value) {
        return selectOne(new LambdaQueryWrapper<T>().eq(field, value));
    }

    default T selectOne(String field1, Object value1, String field2, Object value2) {
        return selectOne(new QueryWrapper<T>().eq(field1, value1).eq(field2, value2));
    }

    default T selectOne(SFunction<T, ?> field1, Object value1, SFunction<T, ?> field2, Object value2) {
        return selectOne(new LambdaQueryWrapper<T>().eq(field1, value1).eq(field2, value2));
    }

    default T selectOne(SFunction<T, ?> field1, Object value1, SFunction<T, ?> field2, Object value2,
                        SFunction<T, ?> field3, Object value3) {
        return selectOne(new LambdaQueryWrapper<T>().eq(field1, value1).eq(field2, value2).eq(field3, value3));
    }

    // ========== 悲观锁查询 ==========

    /**
     * 获得满足条件的一条记录，并使用 FOR UPDATE 锁定。
     * 注意：需要在事务中调用，否则锁会立即释放。
     */
    default T selectOneForUpdate(LambdaQueryWrapper<T> queryWrapper) {
        return selectOne(queryWrapper.last("FOR UPDATE"));
    }

    default T selectOneForUpdate(SFunction<T, ?> field, Object value) {
        return selectOneForUpdate(new LambdaQueryWrapper<T>().eq(field, value));
    }

    default T selectOneForUpdate(SFunction<T, ?> field1, Object value1, SFunction<T, ?> field2, Object value2) {
        return selectOneForUpdate(new LambdaQueryWrapper<T>().eq(field1, value1).eq(field2, value2));
    }

    default T selectOneForUpdate(SFunction<T, ?> field1, Object value1, SFunction<T, ?> field2, Object value2,
                                 SFunction<T, ?> field3, Object value3) {
        return selectOneForUpdate(new LambdaQueryWrapper<T>().eq(field1, value1).eq(field2, value2).eq(field3, value3));
    }

    // ========== 并发安全查询 ==========

    /**
     * 获取满足条件的第 1 条记录
     * 目的：解决并发场景下，插入多条记录后，使用 selectOne 会报错的问题
     */
    default T selectFirstOne(SFunction<T, ?> field, Object value) {
        List<T> list = selectList(new LambdaQueryWrapper<T>().eq(field, value));
        return CollUtil.getFirst(list);
    }

    default T selectFirstOne(SFunction<T, ?> field1, Object value1, SFunction<T, ?> field2, Object value2) {
        List<T> list = selectList(new LambdaQueryWrapper<T>().eq(field1, value1).eq(field2, value2));
        return CollUtil.getFirst(list);
    }

    default T selectFirstOne(SFunction<T, ?> field1, Object value1, SFunction<T, ?> field2, Object value2,
                             SFunction<T, ?> field3, Object value3) {
        List<T> list = selectList(new LambdaQueryWrapper<T>().eq(field1, value1).eq(field2, value2).eq(field3, value3));
        return CollUtil.getFirst(list);
    }

    // ========== 计数 ==========

    default Long selectCount() {
        return selectCount(new QueryWrapper<>());
    }

    default Long selectCount(String field, Object value) {
        return selectCount(new QueryWrapper<T>().eq(field, value));
    }

    default Long selectCount(SFunction<T, ?> field, Object value) {
        return selectCount(new LambdaQueryWrapper<T>().eq(field, value));
    }

    // ========== 列表查询 ==========

    default List<T> selectList() {
        return selectList(new QueryWrapper<>());
    }

    default List<T> selectList(String field, Object value) {
        return selectList(new QueryWrapper<T>().eq(field, value));
    }

    default List<T> selectList(SFunction<T, ?> field, Object value) {
        return selectList(new LambdaQueryWrapper<T>().eq(field, value));
    }

    default List<T> selectList(String field, Collection<?> values) {
        if (CollUtil.isEmpty(values)) {
            return CollUtil.newArrayList();
        }
        return selectList(new QueryWrapper<T>().in(field, values));
    }

    default List<T> selectList(SFunction<T, ?> field, Collection<?> values) {
        if (CollUtil.isEmpty(values)) {
            return CollUtil.newArrayList();
        }
        return selectList(new LambdaQueryWrapper<T>().in(field, values));
    }

    default List<T> selectList(SFunction<T, ?> field1, Object value1, SFunction<T, ?> field2, Object value2) {
        return selectList(new LambdaQueryWrapper<T>().eq(field1, value1).eq(field2, value2));
    }

    // ========== 批量操作 ==========

    /**
     * 批量插入，适合大量数据插入
     */
    default Boolean insertBatch(Collection<T> entities) {
        // 特殊：SQL Server 批量插入后，获取 id 会报错，因此通过循环处理
        DbType dbType = JdbcUtils.getDbType();
        if (JdbcUtils.isSQLServer(dbType)) {
            entities.forEach(this::insert);
            return CollUtil.isNotEmpty(entities);
        }
        return Db.saveBatch(entities);
    }

    /**
     * 批量插入，适合大量数据插入
     */
    default Boolean insertBatch(Collection<T> entities, int size) {
        DbType dbType = JdbcUtils.getDbType();
        if (JdbcUtils.isSQLServer(dbType)) {
            entities.forEach(this::insert);
            return CollUtil.isNotEmpty(entities);
        }
        return Db.saveBatch(entities, size);
    }

    default int updateBatch(T update) {
        return update(update, new QueryWrapper<>());
    }

    default Boolean updateBatch(Collection<T> entities) {
        return Db.updateBatchById(entities);
    }

    default Boolean updateBatch(Collection<T> entities, int size) {
        return Db.updateBatchById(entities, size);
    }

    // ========== 删除 ==========

    default int delete(String field, String value) {
        return delete(new QueryWrapper<T>().eq(field, value));
    }

    default int delete(SFunction<T, ?> field, Object value) {
        return delete(new LambdaQueryWrapper<T>().eq(field, value));
    }

    default int deleteBatch(SFunction<T, ?> field, Collection<?> values) {
        if (CollUtil.isEmpty(values)) {
            return 0;
        }
        return delete(new LambdaQueryWrapper<T>().in(field, values));
    }

}
```

## 2. 核心方法逐一分析

### 2.1 selectPage -- 分页查询返回 PageResult

```java
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
    // 转换返回
    return new PageResult<>(mpPage.getRecords(), mpPage.getTotal());
}
```

这个方法做了三件事：

1. **处理不分页场景**：当 `pageSize = -1`（`PAGE_SIZE_NONE`）时，跳过分页插件，直接查询全部数据。这统一了"分页"和"不分页"的调用方式。
2. **构建 MyBatis-Plus 的 Page 对象**：通过 `MyBatisUtils.buildPage` 将 `PageParam` 转换为 `Page`，同时处理排序字段。
3. **转换返回值**：将 MyBatis-Plus 的 `IPage` 转换为项目统一的 `PageResult`。

### 2.2 selectOne -- 快捷单条查询

```java
// 单字段查询
default T selectOne(SFunction<T, ?> field, Object value) {
    return selectOne(new LambdaQueryWrapper<T>().eq(field, value));
}

// 双字段查询
default T selectOne(SFunction<T, ?> field1, Object value1, SFunction<T, ?> field2, Object value2) {
    return selectOne(new LambdaQueryWrapper<T>().eq(field1, value1).eq(field2, value2));
}
```

业务代码中，按唯一字段查询的场景非常多（按用户名查、按编码查等）。这些快捷方法省去了每次手动构建 `LambdaQueryWrapper` 的步骤。

使用示例：

```java
// 之前
AdminUserDO user = adminUserMapper.selectOne(
    new LambdaQueryWrapper<AdminUserDO>().eq(AdminUserDO::getUsername, "admin"));

// 之后
AdminUserDO user = adminUserMapper.selectOne(AdminUserDO::getUsername, "admin");
```

### 2.3 selectOneForUpdate -- 悲观锁查询

```java
default T selectOneForUpdate(LambdaQueryWrapper<T> queryWrapper) {
    return selectOne(queryWrapper.last("FOR UPDATE"));
}
```

通过 `.last("FOR UPDATE")` 在 SQL 末尾追加 `FOR UPDATE`，实现悲观锁。典型应用场景：电商下单时扣减库存。

```java
@Transactional
public void deductStock(Long skuId, Integer quantity) {
    // SELECT * FROM product_sku WHERE id = ? FOR UPDATE
    ProductSkuDO sku = skuMapper.selectOneForUpdate(ProductSkuDO::getId, skuId);
    if (sku.getStock() < quantity) {
        throw exception(STOCK_NOT_ENOUGH);
    }
    sku.setStock(sku.getStock() - quantity);
    skuMapper.updateById(sku);
}
```

### 2.4 selectFirstOne -- 并发安全查询

```java
default T selectFirstOne(SFunction<T, ?> field, Object value) {
    List<T> list = selectList(new LambdaQueryWrapper<T>().eq(field, value));
    return CollUtil.getFirst(list);
}
```

`selectOne` 在查到多条时会抛异常。但在并发场景下（比如两个请求同时插入了相同编码的数据），后续按编码查询就会报错。`selectFirstOne` 改为查询列表后取第一条，避免异常。

### 2.5 insertBatch -- 批量插入

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

`Db.saveBatch` 是 MyBatis-Plus 提供的批量插入工具，内部会分批执行（默认 1000 条一批），避免单条 SQL 过大。

SQL Server 的特殊处理是因为：批量插入后尝试获取自增 ID 时会报错，只能降级为逐条插入。

### 2.6 deleteBatch -- 批量删除

```java
default int deleteBatch(SFunction<T, ?> field, Collection<?> values) {
    if (CollUtil.isEmpty(values)) {
        return 0;
    }
    return delete(new LambdaQueryWrapper<T>().in(field, values));
}
```

先检查集合是否为空，避免生成 `WHERE id IN ()` 这种无效 SQL。

## 3. 为什么继承 MPJBaseMapper 而非 BaseMapper

```java
public interface BaseMapperX<T> extends MPJBaseMapper<T> {
```

`MPJBaseMapper` 是 `mybatis-plus-join` 库提供的接口，它已经继承了 MyBatis-Plus 的 `BaseMapper`。选择继承 `MPJBaseMapper` 的好处是：`BaseMapperX` 的实现类同时具备单表 CRUD 和联表查询的能力。

联表查询通过 `selectJoinPage`、`selectJoinList` 等方法实现，配合 `MPJLambdaWrapper` 使用。这部分能力在后续的业务模块中会用到。

## 4. MyBatisUtils.buildPage -- 分页构建工具

`selectPage` 内部调用了 `MyBatisUtils.buildPage` 来构建分页对象：

> 源文件：`cn.iocoder.yudao.framework.mybatis.core.util.MyBatisUtils`

```java
public static <T> Page<T> buildPage(PageParam pageParam, Collection<SortingField> sortingFields) {
    // 页码 + 数量
    Page<T> page = new Page<>(pageParam.getPageNo(), pageParam.getPageSize());
    page.setOptimizeJoinOfCountSql(false); // 关联 issue：https://gitee.com/zhijiantianya/yudao-cloud/issues/ID2QLL
    // 排序字段
    if (CollUtil.isNotEmpty(sortingFields)) {
        for (SortingField sortingField : sortingFields) {
            String columnName = buildSafeOrderColumn(sortingField.getField());
            if (columnName == null) {
                continue;
            }
            page.addOrder(new OrderItem().setAsc(isAscOrder(sortingField.getOrder())).setColumn(columnName));
        }
    }
    return page;
}
```

排序字段名会经过 `buildSafeOrderColumn` 安全检查：

```java
private static final Pattern SAFE_COLUMN_NAME_PATTERN = Pattern.compile("^[a-zA-Z0-9_]+(\\.[a-zA-Z0-9_]+)*$");

private static String buildSafeOrderColumn(String field) {
    String columnName = StrUtil.toUnderlineCase(field);
    if (StrUtil.isEmpty(columnName) || !SAFE_COLUMN_NAME_PATTERN.matcher(columnName).matches()) {
        return null;
    }
    return columnName;
}
```

这个正则检查防止 SQL 注入：如果前端传入的排序字段名包含特殊字符（如 `; DROP TABLE`），直接丢弃。

## 5. 业务使用模式总结

`BaseMapperX` 的使用模式在所有业务模块中保持一致：

```java
@Mapper
public interface OrderMapper extends BaseMapperX<OrderDO> {

    // 分页查询：selectPage + LambdaQueryWrapperX
    default PageResult<OrderDO> selectPage(OrderPageReqVO reqVO) {
        return selectPage(reqVO, new LambdaQueryWrapperX<OrderDO>()
                .likeIfPresent(OrderDO::getOrderNo, reqVO.getOrderNo())
                .eqIfPresent(OrderDO::getStatus, reqVO.getStatus())
                .betweenIfPresent(OrderDO::getCreateTime, reqVO.getCreateTime())
                .orderByDesc(OrderDO::getId));
    }

    // 单字段查询
    default OrderDO selectByOrderNo(String orderNo) {
        return selectOne(OrderDO::getOrderNo, orderNo);
    }

    // 多字段查询
    default OrderDO selectByUserIdAndOrderNo(Long userId, String orderNo) {
        return selectOne(OrderDO::getUserId, userId, OrderDO::getOrderNo, orderNo);
    }

    // IN 查询
    default List<OrderDO> selectListByUserId(Collection<Long> userIds) {
        return selectList(OrderDO::getUserId, userIds);
    }
}
```

---

> 下一章：[05-数据库兼容与多数据源](./05-数据库兼容与多数据源.md)
>
> 设计参考：[05-多数据源与多数据库设计](../功能设计/05-多数据源与多数据库设计.md)
