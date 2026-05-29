# LambdaQueryWrapperX 实现

---

本章实现查询条件增强类 `LambdaQueryWrapperX`。它继承 MyBatis-Plus 的 `LambdaQueryWrapper`，增加了 `*IfPresent` 系列方法，让条件拼接不再需要手动判断 null。

> 设计原理请回顾：[04-BaseMapperX与查询增强设计](../功能设计/04-BaseMapperX与查询增强设计.md)

## 1. 先看原生 LambdaQueryWrapper 的痛点

在没有增强之前，查询代码是这样的：

```java
// 原生写法：每个条件都要手动判断 null
LambdaQueryWrapper<AdminUserDO> wrapper = new LambdaQueryWrapper<>();
if (StringUtils.hasText(reqVO.getUsername())) {
    wrapper.like(AdminUserDO::getUsername, reqVO.getUsername());
}
if (reqVO.getStatus() != null) {
    wrapper.eq(AdminUserDO::getStatus, reqVO.getStatus());
}
if (reqVO.getDeptId() != null) {
    wrapper.eq(AdminUserDO::getDeptId, reqVO.getDeptId());
}
```

`LambdaQueryWrapperX` 的目标是消除这些 `if` 判断。

## 2. LambdaQueryWrapperX 完整实现

> 源文件：`cn.iocoder.yudao.framework.mybatis.core.query.LambdaQueryWrapperX`

```java
public class LambdaQueryWrapperX<T> extends LambdaQueryWrapper<T> {

    public LambdaQueryWrapperX<T> likeIfPresent(SFunction<T, ?> column, String val) {
        if (StringUtils.hasText(val)) {
            return (LambdaQueryWrapperX<T>) super.like(column, val);
        }
        return this;
    }
    public LambdaQueryWrapperX<T> likeRightIfPresent(SFunction<T, ?> column, String val) {
        if (StringUtils.hasText(val)) {
            return (LambdaQueryWrapperX<T>) super.likeRight(column, val);
        }
        return this;
    }

    public LambdaQueryWrapperX<T> inIfPresent(SFunction<T, ?> column, Collection<?> values) {
        if (ObjectUtil.isAllNotEmpty(values) && !ArrayUtil.isEmpty(values)) {
            return (LambdaQueryWrapperX<T>) super.in(column, values);
        }
        return this;
    }

    public LambdaQueryWrapperX<T> inIfPresent(SFunction<T, ?> column, Object... values) {
        if (ObjectUtil.isAllNotEmpty(values) && !ArrayUtil.isEmpty(values)) {
            return (LambdaQueryWrapperX<T>) super.in(column, values);
        }
        return this;
    }

    public LambdaQueryWrapperX<T> eqIfPresent(SFunction<T, ?> column, Object val) {
        if (ObjectUtil.isNotEmpty(val)) {
            return (LambdaQueryWrapperX<T>) super.eq(column, val);
        }
        return this;
    }

    public LambdaQueryWrapperX<T> neIfPresent(SFunction<T, ?> column, Object val) {
        if (ObjectUtil.isNotEmpty(val)) {
            return (LambdaQueryWrapperX<T>) super.ne(column, val);
        }
        return this;
    }

    public LambdaQueryWrapperX<T> gtIfPresent(SFunction<T, ?> column, Object val) {
        if (val != null) {
            return (LambdaQueryWrapperX<T>) super.gt(column, val);
        }
        return this;
    }

    public LambdaQueryWrapperX<T> geIfPresent(SFunction<T, ?> column, Object val) {
        if (val != null) {
            return (LambdaQueryWrapperX<T>) super.ge(column, val);
        }
        return this;
    }

    public LambdaQueryWrapperX<T> ltIfPresent(SFunction<T, ?> column, Object val) {
        if (val != null) {
            return (LambdaQueryWrapperX<T>) super.lt(column, val);
        }
        return this;
    }

    public LambdaQueryWrapperX<T> leIfPresent(SFunction<T, ?> column, Object val) {
        if (val != null) {
            return (LambdaQueryWrapperX<T>) super.le(column, val);
        }
        return this;
    }

    public LambdaQueryWrapperX<T> betweenIfPresent(SFunction<T, ?> column, Object val1, Object val2) {
        if (val1 != null && val2 != null) {
            return (LambdaQueryWrapperX<T>) super.between(column, val1, val2);
        }
        if (val1 != null) {
            return (LambdaQueryWrapperX<T>) ge(column, val1);
        }
        if (val2 != null) {
            return (LambdaQueryWrapperX<T>) le(column, val2);
        }
        return this;
    }

    public LambdaQueryWrapperX<T> betweenIfPresent(SFunction<T, ?> column, Object[] values) {
        Object val1 = ArrayUtils.get(values, 0);
        Object val2 = ArrayUtils.get(values, 1);
        return betweenIfPresent(column, val1, val2);
    }

    // ========== 重写父类方法，方便链式调用 ==========

    @Override
    public LambdaQueryWrapperX<T> eq(boolean condition, SFunction<T, ?> column, Object val) {
        super.eq(condition, column, val);
        return this;
    }

    @Override
    public LambdaQueryWrapperX<T> eq(SFunction<T, ?> column, Object val) {
        super.eq(column, val);
        return this;
    }

    @Override
    public LambdaQueryWrapperX<T> orderByDesc(SFunction<T, ?> column) {
        super.orderByDesc(true, column);
        return this;
    }

    @Override
    public LambdaQueryWrapperX<T> last(String lastSql) {
        super.last(lastSql);
        return this;
    }

    @Override
    public LambdaQueryWrapperX<T> in(SFunction<T, ?> column, Collection<?> coll) {
        super.in(column, coll);
        return this;
    }

}
```

## 3. 逐方法分析

### 3.1 字符串类型：likeIfPresent / likeRightIfPresent

```java
public LambdaQueryWrapperX<T> likeIfPresent(SFunction<T, ?> column, String val) {
    if (StringUtils.hasText(val)) {
        return (LambdaQueryWrapperX<T>) super.like(column, val);
    }
    return this;
}
```

使用 `StringUtils.hasText(val)` 判断，覆盖三种"空"情况：
- `null`
- 空串 `""`
- 纯空白 `"   "`

这三种情况都会跳过条件拼接。

### 3.2 数值/对象类型：eqIfPresent / neIfPresent

```java
public LambdaQueryWrapperX<T> eqIfPresent(SFunction<T, ?> column, Object val) {
    if (ObjectUtil.isNotEmpty(val)) {
        return (LambdaQueryWrapperX<T>) super.eq(column, val);
    }
    return this;
}
```

使用 `ObjectUtil.isNotEmpty(val)` 判断。对于字符串，`isNotEmpty` 会同时检查 null 和空串；对于其他类型，只检查 null。

### 3.3 范围比较：gtIfPresent / geIfPresent / ltIfPresent / leIfPresent

```java
public LambdaQueryWrapperX<T> gtIfPresent(SFunction<T, ?> column, Object val) {
    if (val != null) {
        return (LambdaQueryWrapperX<T>) super.gt(column, val);
    }
    return this;
}
```

只用 `val != null` 判断。注意这里不用 `ObjectUtil.isNotEmpty`，因为数值 `0` 是合法值，不能跳过。

### 3.4 集合类型：inIfPresent

```java
public LambdaQueryWrapperX<T> inIfPresent(SFunction<T, ?> column, Collection<?> values) {
    if (ObjectUtil.isAllNotEmpty(values) && !ArrayUtil.isEmpty(values)) {
        return (LambdaQueryWrapperX<T>) super.in(column, values);
    }
    return this;
}
```

双重检查：`ObjectUtil.isAllNotEmpty` 检查集合中每个元素都不为空，`!ArrayUtil.isEmpty` 检查集合本身不为空。空集合表示"不限制"，跳过 IN 条件。

### 3.5 betweenIfPresent -- 智能降级

```java
public LambdaQueryWrapperX<T> betweenIfPresent(SFunction<T, ?> column, Object val1, Object val2) {
    if (val1 != null && val2 != null) {
        return (LambdaQueryWrapperX<T>) super.between(column, val1, val2);
    }
    if (val1 != null) {
        return (LambdaQueryWrapperX<T>) ge(column, val1);
    }
    if (val2 != null) {
        return (LambdaQueryWrapperX<T>) le(column, val2);
    }
    return this;
}
```

`betweenIfPresent` 不是简单的"两个都有才拼接"，而是智能降级：

| val1 | val2 | 结果 |
|------|------|------|
| 有值 | 有值 | `BETWEEN val1 AND val2` |
| 有值 | null | `>= val1` |
| null | 有值 | `<= val2` |
| null | null | 跳过 |

还有一个数组版本 `betweenIfPresent(SFunction<T, ?> column, Object[] values)`，从数组中取出第一个和第二个元素，委托给上面的方法。这在前端传入时间范围数组时很方便。

## 4. 为什么要重写父类方法

```java
@Override
public LambdaQueryWrapperX<T> eq(SFunction<T, ?> column, Object val) {
    super.eq(column, val);
    return this;
}
```

父类 `LambdaQueryWrapper` 的 `eq` 方法返回的是 `LambdaQueryWrapper` 类型。如果不重写，链式调用时会丢失 `LambdaQueryWrapperX` 的类型信息：

```java
// 不重写的后果：
new LambdaQueryWrapperX<DeptDO>()
    .eq(DeptDO::getStatus, 0)           // 返回 LambdaQueryWrapper
    .likeIfPresent(DeptDO::getName, name) // 编译错误！LambdaQueryWrapper 没有 likeIfPresent 方法
```

重写后，所有方法都返回 `LambdaQueryWrapperX`，链式调用不会断。

## 5. 空值判断策略汇总

| 方法 | 判断方式 | 适用场景 | 为什么不用 != null |
|------|---------|---------|------------------|
| `likeIfPresent` | `hasText` | 字符串模糊搜索 | 空串 `""` 也不应该匹配 |
| `likeRightIfPresent` | `hasText` | 字符串前缀搜索 | 同上 |
| `eqIfPresent` | `ObjectUtil.isNotEmpty` | 通用等值判断 | 字符串空串也要跳过 |
| `neIfPresent` | `ObjectUtil.isNotEmpty` | 排除某值 | 同上 |
| `gtIfPresent` 等 | `!= null` | 数值范围判断 | `0` 是合法值 |
| `inIfPresent` | `isAllNotEmpty && !isEmpty` | 集合 IN 查询 | 空集合表示不限制 |
| `betweenIfPresent` | `!= null` + 智能降级 | 时间/数值范围 | 允许只传一端 |

## 6. 使用效果对比

### 之前（原生 LambdaQueryWrapper）

```java
LambdaQueryWrapper<AdminUserDO> wrapper = new LambdaQueryWrapper<>();
if (StringUtils.hasText(reqVO.getUsername())) {
    wrapper.like(AdminUserDO::getUsername, reqVO.getUsername());
}
if (reqVO.getStatus() != null) {
    wrapper.eq(AdminUserDO::getStatus, reqVO.getStatus());
}
if (reqVO.getCreateTime() != null && reqVO.getCreateTime().length == 2) {
    wrapper.between(AdminUserDO::getCreateTime, reqVO.getCreateTime()[0], reqVO.getCreateTime()[1]);
}
wrapper.orderByDesc(AdminUserDO::getId);
```

### 之后（LambdaQueryWrapperX）

```java
LambdaQueryWrapperX<AdminUserDO> wrapper = new LambdaQueryWrapperX<AdminUserDO>()
    .likeIfPresent(AdminUserDO::getUsername, reqVO.getUsername())
    .eqIfPresent(AdminUserDO::getStatus, reqVO.getStatus())
    .betweenIfPresent(AdminUserDO::getCreateTime, reqVO.getCreateTime())
    .orderByDesc(AdminUserDO::getId);
```

代码行数从 8 行减少到 4 行，且语义更清晰。

---

> 下一章：[04-BaseMapperX实现](./04-BaseMapperX实现.md)
>
> 设计参考：[04-BaseMapperX与查询增强设计](../功能设计/04-BaseMapperX与查询增强设计.md)
