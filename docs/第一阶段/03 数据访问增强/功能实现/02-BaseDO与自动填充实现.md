# BaseDO 与自动填充实现

---

本章实现数据访问层的基石：`BaseDO` 实体基类和 `DefaultDBFieldHandler` 自动填充处理器。所有业务实体都继承 `BaseDO`，自动获得审计字段和软删除能力。

> 设计原理请回顾：[03-实体基类与自动填充设计](../功能设计/03-实体基类与自动填充设计.md)

## 1. BaseDO -- 实体基类

> 源文件：`cn.iocoder.yudao.framework.mybatis.core.dataobject.BaseDO`

```java
@Data
@JsonIgnoreProperties(value = "transMap") // 由于 Easy-Trans 会添加 transMap 属性，避免 Jackson 在 Spring Cache 反序列化报错
public abstract class BaseDO implements Serializable, TransPojo {

    /**
     * 创建时间
     */
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;
    /**
     * 最后更新时间
     */
    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
    /**
     * 创建者，目前使用 SysUser 的 id 编号
     *
     * 使用 String 类型的原因是，未来可能会存在非数值的情况，留好拓展性。
     */
    @TableField(fill = FieldFill.INSERT, jdbcType = JdbcType.VARCHAR)
    private String creator;
    /**
     * 更新者，目前使用 SysUser 的 id 编号
     *
     * 使用 String 类型的原因是，未来可能会存在非数值的情况，留好拓展性。
     */
    @TableField(fill = FieldFill.INSERT_UPDATE, jdbcType = JdbcType.VARCHAR)
    private String updater;
    /**
     * 是否删除
     */
    @TableLogic
    private Boolean deleted;

    /**
     * 把 creator、createTime、updateTime、updater 都清空，避免前端直接传递 creator 之类的字段，直接就被更新了
     */
    public void clean(){
        this.creator = null;
        this.createTime = null;
        this.updater = null;
        this.updateTime = null;
    }

}
```

### 5 个审计字段详解

| 字段 | 类型 | 填充时机 | 说明 |
|------|------|---------|------|
| `createTime` | `LocalDateTime` | INSERT | 记录创建时间 |
| `updateTime` | `LocalDateTime` | INSERT + UPDATE | 记录最后更新时间 |
| `creator` | `String` | INSERT | 创建人 ID（用 String 而非 Long，预留扩展性） |
| `updater` | `String` | INSERT + UPDATE | 更新人 ID |
| `deleted` | `Boolean` | -- | 逻辑删除标记，`@TableLogic` 使 MyBatis-Plus 自动处理 |

### 关键设计点

**`@TableField(fill = ...)` 注解**：告诉 MyBatis-Plus 在插入/更新时自动填充这些字段。但注解本身只声明"需要填充"，具体填什么值由 `MetaObjectHandler` 实现决定。

**`@TableLogic` 注解**：MyBatis-Plus 会自动将 `DELETE` 语句转为 `UPDATE SET deleted = true`，所有 `SELECT` 语句自动追加 `WHERE deleted = false`。业务代码完全不需要关心软删除逻辑。

**`creator`/`updater` 用 String 类型**：虽然当前存储的是用户 ID（数值），但未来可能有 "SYSTEM"、"WECHAT_SYNC" 等非数值来源。String 类型提供了更好的扩展性，代价是多占一点存储空间。

**`clean()` 方法**：在更新操作前调用，清空审计字段，防止前端传入的值覆盖自动填充结果。用法示例：

```java
// Controller 接收到前端传来的更新请求
DeptDO updateObj = BeanUtils.toBean(updateReqVO, DeptDO.class);
updateObj.clean(); // 清空审计字段，让自动填充来设置
deptMapper.updateById(updateObj);
```

**`@JsonIgnoreProperties(value = "transMap")`**：Easy-Trans 框架会在实体中动态添加 `transMap` 属性用于翻译。如果不忽略它，Spring Cache 序列化时会报错。

## 2. DefaultDBFieldHandler -- 自动填充

> 源文件：`cn.iocoder.yudao.framework.mybatis.core.handler.DefaultDBFieldHandler`

```java
public class DefaultDBFieldHandler implements MetaObjectHandler {

    @Override
    @SuppressWarnings("PatternVariableCanBeUsed")
    public void insertFill(MetaObject metaObject) {
        if (Objects.nonNull(metaObject) && metaObject.getOriginalObject() instanceof BaseDO) {
            BaseDO baseDO = (BaseDO) metaObject.getOriginalObject();

            LocalDateTime current = LocalDateTime.now();
            // 创建时间为空，则以当前时间为插入时间
            if (Objects.isNull(baseDO.getCreateTime())) {
                baseDO.setCreateTime(current);
            }
            // 更新时间为空，则以当前时间为更新时间
            if (Objects.isNull(baseDO.getUpdateTime())) {
                baseDO.setUpdateTime(current);
            }

            Long userId = SecurityFrameworkUtils.getLoginUserId();
            // 当前登录用户不为空，创建人为空，则当前登录用户为创建人
            if (Objects.nonNull(userId) && Objects.isNull(baseDO.getCreator())) {
                baseDO.setCreator(userId.toString());
            }
            // 当前登录用户不为空，更新人为空，则当前登录用户为更新人
            if (Objects.nonNull(userId) && Objects.isNull(baseDO.getUpdater())) {
                baseDO.setUpdater(userId.toString());
            }
        }
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        // 更新时间为空，则以当前时间为更新时间
        Object modifyTime = getFieldValByName("updateTime", metaObject);
        if (Objects.isNull(modifyTime)) {
            setFieldValByName("updateTime", LocalDateTime.now(), metaObject);
        }

        // 当前登录用户不为空，更新人为空，则当前登录用户为更新人
        Object modifier = getFieldValByName("updater", metaObject);
        Long userId = SecurityFrameworkUtils.getLoginUserId();
        if (Objects.nonNull(userId) && Objects.isNull(modifier)) {
            setFieldValByName("updater", userId.toString(), metaObject);
        }
    }
}
```

### insertFill 的执行流程

```
调用 mapper.insert(entity)
    │
    ├── MyBatis-Plus 检测到 @TableField(fill = FieldFill.INSERT) 注解
    │
    ├── 调用 DefaultDBFieldHandler.insertFill(metaObject)
    │
    ├── 检查 metaObject.getOriginalObject() 是否为 BaseDO 实例
    │   └── 不是 BaseDO → 直接返回，不影响其他实体
    │
    ├── 判断 createTime 是否为空
    │   └── 为空 → 设置为 LocalDateTime.now()
    │
    ├── 判断 updateTime 是否为空
    │   └── 为空 → 设置为 LocalDateTime.now()
    │
    ├── 获取当前登录用户 ID
    │   └── 通过 SecurityFrameworkUtils.getLoginUserId() 从 SecurityContext 获取
    │
    ├── 判断 creator 是否为空
    │   └── 为空且用户 ID 存在 → 设置为 userId.toString()
    │
    └── 判断 updater 是否为空
        └── 为空且用户 ID 存在 → 设置为 userId.toString()
```

### "为空才填充"的设计原则

自动填充遵循一个核心原则：**只在字段为空时才填充，不覆盖业务代码已显式设置的值**。

这保留了灵活性。比如数据迁移场景，需要保留原始的创建时间和创建人：

```java
DeptDO dept = new DeptDO();
dept.setName("技术部");
dept.setCreateTime(LocalDateTime.of(2020, 1, 1, 0, 0)); // 保留原始创建时间
dept.setCreator("MIGRATION");                              // 保留原始创建人
deptMapper.insert(dept);
// 自动填充不会覆盖 createTime 和 creator，只会填充 updateTime 和 updater
```

### insertFill 与 updateFill 的区别

| 维度 | insertFill | updateFill |
|------|-----------|-----------|
| 触发时机 | INSERT 语句 | UPDATE 语句 |
| 填充字段 | createTime、updateTime、creator、updater | updateTime、updater |
| 获取值的方式 | 直接操作 BaseDO 对象 | 通过 `getFieldValByName` / `setFieldValByName` |
| 类型检查 | 通过 `instanceof BaseDO` 判断 | 不需要，只按字段名操作 |

`updateFill` 使用 `getFieldValByName` / `setFieldValByName` 而非直接操作对象，这是因为 MyBatis-Plus 的更新操作不一定传入完整的实体对象（可能是 `UpdateWrapper`），通过 MetaObject API 可以统一处理。

### 用户 ID 来源

`SecurityFrameworkUtils.getLoginUserId()` 从 Spring Security 的 `SecurityContext` 中获取当前登录用户 ID。这意味着：

- **Web 请求**：正常获取，自动填充 creator/updater
- **定时任务**：没有登录上下文，creator/updater 保持 null
- **消息队列消费者**：需要手动设置 SecurityContext，否则 creator/updater 保持 null

## 3. 注册自动填充处理器

在 `YudaoMybatisAutoConfiguration` 中注册 `DefaultDBFieldHandler`：

> 源文件：`cn.iocoder.yudao.framework.mybatis.config.YudaoMybatisAutoConfiguration`

```java
@Bean
public MetaObjectHandler defaultMetaObjectHandler() {
    return new DefaultDBFieldHandler(); // 自动填充参数类
}
```

Spring Boot 自动配置时会扫描到这个 Bean，MyBatis-Plus 在执行 INSERT/UPDATE 操作时会自动调用它。

## 4. 业务实体如何使用

以电商系统的商品分类为例，业务实体只需继承 `BaseDO`：

```java
@Data
@EqualsAndHashCode(callSuper = true)
@TableName("product_category")
public class ProductCategoryDO extends BaseDO {

    @TableId
    private Long id;
    /**
     * 分类名称
     */
    private String name;
    /**
     * 父分类 ID
     */
    private Long parentId;
    /**
     * 排序
     */
    private Integer sort;
    /**
     * 状态
     */
    private Integer status;
}
```

继承 `BaseDO` 后，`ProductCategoryDO` 自动拥有 5 个审计字段。调用 `mapper.insert(entity)` 时，MyBatis-Plus 会自动填充 `createTime`、`updateTime`、`creator`、`updater`。调用 `mapper.deleteById(id)` 时，MyBatis-Plus 会自动转为 `UPDATE SET deleted = true`。

## 5. 验证自动填充

可以用一个简单的测试验证自动填充是否生效：

```java
@SpringBootTest
class ProductCategoryMapperTest {

    @Resource
    private ProductCategoryMapper productCategoryMapper;

    @Test
    void testAutoFill() {
        ProductCategoryDO category = new ProductCategoryDO();
        category.setName("数码产品");
        category.setParentId(0L);
        category.setSort(1);
        category.setStatus(0);
        productCategoryMapper.insert(category);

        // 验证自动填充
        assertNotNull(category.getId());           // 主键自动生成
        assertNotNull(category.getCreateTime());   // 自动填充当前时间
        assertNotNull(category.getUpdateTime());   // 自动填充当前时间
        // creator/updater 取决于是否有登录用户，测试环境可能为 null
    }
}
```

---

> 下一章：[03-LambdaQueryWrapperX实现](./03-LambdaQueryWrapperX实现.md)
>
> 设计参考：[04-BaseMapperX与查询增强设计](../功能设计/04-BaseMapperX与查询增强设计.md)
