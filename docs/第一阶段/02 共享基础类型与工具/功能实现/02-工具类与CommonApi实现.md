# 工具类与 CommonApi 实现

本章实现 `yudao-common` 中的工具类和跨模块 RPC 契约接口。

> 对应功能设计：[03-分页与工具类设计](../功能设计/03-分页与工具类设计.md)、[04-跨模块RPC契约设计](../功能设计/04-跨模块RPC契约设计.md)

---

## 1. CollectionUtils 核心方法

`CollectionUtils` 是项目中使用频率最高的工具类。以下是核心方法的实现：

```java
// yudao-framework/yudao-common/src/.../util/collection/CollectionUtils.java
public class CollectionUtils {

    // 列表转换
    public static <T, R> List<R> convertList(Collection<T> from, Function<T, R> func) {
        if (CollUtil.isEmpty(from)) {
            return Collections.emptyList();
        }
        return from.stream().map(func).filter(Objects::nonNull).collect(Collectors.toList());
    }

    // 集合转换
    public static <T, R> Set<R> convertSet(Collection<T> from, Function<T, R> func) {
        if (CollUtil.isEmpty(from)) {
            return Collections.emptySet();
        }
        return from.stream().map(func).filter(Objects::nonNull).collect(Collectors.toSet());
    }

    // Map 转换
    public static <T, K, V> Map<K, V> convertMap(Collection<T> from,
                                                   Function<T, K> keyFunc,
                                                   Function<T, V> valueFunc) {
        if (CollUtil.isEmpty(from)) {
            return Collections.emptyMap();
        }
        return from.stream().collect(Collectors.toMap(keyFunc, valueFunc, (v1, v2) -> v1));
    }

    // 分页结果转换
    public static <T, R> PageResult<R> convertPage(PageResult<T> from, Function<T, R> func) {
        return new PageResult<>(convertList(from.getList(), func), from.getTotal());
    }

    // 新旧列表对比：返回 [创建列表, 更新列表, 删除列表]
    public static <T> List<List<T>> diffList(List<T> oldList, List<T> newList,
                                              BiFunction<T, T, Boolean> matcher) {
        List<T> createList = new ArrayList<>();
        List<T> updateList = new ArrayList<>();
        List<T> deleteList = new ArrayList<>();

        // 找新增和更新
        for (T newItem : newList) {
            T oldItem = oldList.stream()
                    .filter(o -> matcher.apply(o, newItem))
                    .findFirst().orElse(null);
            if (oldItem == null) {
                createList.add(newItem);
            } else {
                updateList.add(newItem);
            }
        }
        // 找删除
        for (T oldItem : oldList) {
            boolean exists = newList.stream()
                    .anyMatch(n -> matcher.apply(oldItem, n));
            if (!exists) {
                deleteList.add(oldItem);
            }
        }

        return Arrays.asList(createList, updateList, deleteList);
    }
}
```

`diffList` 在"批量保存"场景中非常有用：

```java
@Override
@Transactional
public void saveList(List<ProductSaveReqVO> newList) {
    List<ProductDO> oldList = productMapper.selectListBySpuId(spuId);
    List<List<ProductDO>> diff = CollectionUtils.diffList(oldList,
            BeanUtils.toBean(newList, ProductDO.class),
            (old, new_) -> old.getId().equals(new_.getId()));

    diff.get(0).forEach(productMapper::insert);    // 新增
    diff.get(1).forEach(productMapper::updateById); // 更新
    diff.get(2).forEach(productMapper::deleteById); // 删除
}
```

---

## 2. JsonUtils 定制

JsonUtils 的核心是定制 `ObjectMapper`：

```java
// yudao-framework/yudao-common/src/.../util/json/JsonUtils.java
public class JsonUtils {

    private static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    static {
        // 基本配置
        OBJECT_MAPPER.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        OBJECT_MAPPER.configure(SerializationFeature.FAIL_ON_EMPTY_BEANS, false);
        OBJECT_MAPPER.setSerializationInclusion(JsonInclude.Include.NON_NULL);

        // LocalDateTime 序列化为时间戳（毫秒）
        JavaTimeModule javaTimeModule = new JavaTimeModule();
        javaTimeModule.addSerializer(LocalDateTime.class, new TimestampLocalDateTimeSerializer());
        javaTimeModule.addDeserializer(LocalDateTime.class, new TimestampLocalDateTimeDeserializer());
        OBJECT_MAPPER.registerModule(javaTimeModule);

        // Long 超过 JS 安全整数范围时序列化为 String
        SimpleModule simpleModule = new SimpleModule();
        simpleModule.addSerializer(Long.class, NumberSerializer.instance);
        simpleModule.addSerializer(Long.TYPE, NumberSerializer.instance);
        OBJECT_MAPPER.registerModule(simpleModule);
    }

    public static String toJsonString(Object object) {
        return OBJECT_MAPPER.writeValueAsString(object);
    }

    public static <T> T parseObject(String text, Class<T> clazz) {
        return OBJECT_MAPPER.readValue(text, clazz);
    }

    // convertObject: 避免 serialize → deserialize 的开销
    public static <T> T convertObject(Object fromValue, Class<T> toValueType) {
        return OBJECT_MAPPER.convertValue(fromValue, toValueType);
    }
}
```

`NumberSerializer` 的实现——当 Long 值超过 JavaScript 的安全整数范围时序列化为字符串：

```java
// yudao-framework/yudao-common/src/.../util/json/databind/NumberSerializer.java
public class NumberSerializer extends com.fasterxml.jackson.databind.ser.std.NumberSerializer {
    public static final NumberSerializer instance = new NumberSerializer();

    @Override
    public void serialize(Number value, JsonGenerator gen, SerializerProvider provider) throws IOException {
        // 超过 JS 安全整数范围：序列化为 String
        if (value.longValue() > 9007199254740991L || value.longValue() < -9007199254740991L) {
            gen.writeString(value.toString());
        } else {
            super.serialize(value, gen, provider);
        }
    }
}
```

---

## 3. CommonApi 接口实现

以 `PermissionCommonApi` 为例，展示 CommonApi 的完整实现链。

### 接口定义（yudao-common）

```java
// yudao-framework/yudao-common/src/.../biz/system/permission/PermissionCommonApi.java
@FeignClient(name = "system-server", primary = false)
public interface PermissionCommonApi {

    @GetMapping("/rpc-api/system/permission/has-any-permissions")
    CommonResult<Boolean> hasAnyPermissions(
            @RequestParam("userId") Long userId,
            @RequestParam("permissions") String... permissions);

    @GetMapping("/rpc-api/system/permission/has-any-roles")
    CommonResult<Boolean> hasAnyRoles(
            @RequestParam("userId") Long userId,
            @RequestParam("roles") String... roles);
}
```

### 服务端实现（yudao-module-system-server）

```java
// yudao-module-system-server/src/.../api/permission/PermissionApiImpl.java
@RestController
@Validated
public class PermissionApiImpl implements PermissionCommonApi {

    @Resource
    private PermissionService permissionService;

    @Override
    public CommonResult<Boolean> hasAnyPermissions(Long userId, String... permissions) {
        boolean result = permissionService.hasAnyPermissions(userId, Arrays.asList(permissions));
        return CommonResult.success(result);
    }

    @Override
    public CommonResult<Boolean> hasAnyRoles(Long userId, String... roles) {
        boolean result = permissionService.hasAnyRoles(userId, Arrays.asList(roles));
        return CommonResult.success(result);
    }
}
```

### 框架层调用（yudao-spring-boot-starter-security）

```java
// yudao-framework/yudao-spring-boot-starter-security/src/.../SecurityFrameworkServiceImpl.java
public class SecurityFrameworkServiceImpl implements SecurityFrameworkService {

    private final PermissionCommonApi permissionApi;
    private final LoadingCache<KeyValue<Long, List<String>>, Boolean> hasAnyPermissionsCache;

    public SecurityFrameworkServiceImpl(PermissionCommonApi permissionApi) {
        this.permissionApi = permissionApi;
        // Guava 缓存：1 分钟 TTL
        this.hasAnyPermissionsCache = CacheUtils.buildCache(Duration.ofMinutes(1),
                key -> permissionApi.hasAnyPermissions(key.getKey(),
                        key.getValue().toArray(new String[0])).getCheckedData());
    }

    @Override
    public boolean hasPermission(String permission) {
        return hasAnyPermissions(permission);
    }

    @Override
    public boolean hasAnyPermissions(String... permissions) {
        Long userId = SecurityFrameworkUtils.getLoginUserId();
        if (userId == null) return false;
        if (skipPermissionCheck()) return true;  // 跨租户跳过校验

        return hasAnyPermissionsCache.getUnchecked(
                new KeyValue<>(userId, Arrays.asList(permissions)));
    }
}
```

调用链路：

```
Controller 上的 @PreAuthorize("@ss.hasPermission('system:dept:query')")
  → SecurityFrameworkServiceImpl.hasPermission("system:dept:query")
    → hasAnyPermissionsCache.getUnchecked(...)
      → [缓存未命中] permissionApi.hasAnyPermissions(userId, ["system:dept:query"])
        → [单体模式] PermissionApiImpl.hasAnyPermissions()
          → PermissionService.hasAnyPermissions()
            → 从 Redis 查用户权限列表
```

---

## 4. 自定义校验注解 @InEnum 实现

```java
// yudao-framework/yudao-common/src/.../validation/InEnum.java
@Target({ElementType.FIELD, ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Constraint(validatedBy = {InEnumValidator.class, InEnumCollectionValidator.class})
public @interface InEnum {
    Class<? extends ArrayValuable<?>> value();
    String message() default "传入的值不在允许的范围内";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// yudao-framework/yudao-common/src/.../validation/InEnumValidator.java
public class InEnumValidator implements ConstraintValidator<InEnum, Object> {
    private Object[] allowedValues;

    @Override
    public void initialize(InEnum annotation) {
        ArrayValuable<?>[] enumConstants = annotation.value().getEnumConstants();
        if (enumConstants == null) {
            // 不是枚举，尝试调用 array() 方法
            try {
                allowedValues = (Object[]) annotation.value()
                        .getMethod("array").invoke(null);
            } catch (Exception e) {
                allowedValues = new Object[0];
            }
        } else {
            allowedValues = enumConstants[0].array();
        }
    }

    @Override
    public boolean isValid(Object value, ConstraintValidatorContext context) {
        if (value == null) return true;  // null 由 @NotNull 校验
        for (Object allowed : allowedValues) {
            if (allowed.equals(value)) return true;
        }
        return false;
    }
}
```

使用示例：

```java
// VO 中使用
public class DeptSaveReqVO {
    @InEnum(CommonStatusEnum.class)  // 只能传 0（启用）或 1（禁用）
    private Integer status;
}
```
