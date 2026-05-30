# 05-SecurityFrameworkService 权限校验实现

## 目标

实现 `@ss` Bean——`SecurityFrameworkServiceImpl`，它是 `@PreAuthorize("@ss.hasPermission('...')")` 的执行者。

## 接口定义

```java
// 文件路径：core/service/SecurityFrameworkService.java
public interface SecurityFrameworkService {
    boolean hasPermission(String permission);
    boolean hasAnyPermissions(String... permissions);
    boolean hasRole(String role);
    boolean hasAnyRoles(String... roles);
    boolean hasScope(String scope);
    boolean hasAnyScopes(String... scope);
}
```

## 实现类

```java
// 文件路径：core/service/SecurityFrameworkServiceImpl.java
@AllArgsConstructor
public class SecurityFrameworkServiceImpl implements SecurityFrameworkService {

    private final PermissionCommonApi permissionApi;

    // 角色校验缓存（1 分钟过期）
    private final LoadingCache<KeyValue<Long, List<String>>, Boolean> hasAnyRolesCache =
        buildCache(Duration.ofMinutes(1L), new CacheLoader<>() {
            @Override
            public Boolean load(KeyValue<Long, List<String>> key) {
                return permissionApi.hasAnyRoles(key.getKey(),
                    key.getValue().toArray(new String[0])).getCheckedData();
            }
        });

    // 权限校验缓存（1 分钟过期）
    private final LoadingCache<KeyValue<Long, List<String>>, Boolean> hasAnyPermissionsCache =
        buildCache(Duration.ofMinutes(1L), new CacheLoader<>() {
            @Override
            public Boolean load(KeyValue<Long, List<String>> key) {
                return permissionApi.hasAnyPermissions(key.getKey(),
                    key.getValue().toArray(new String[0])).getCheckedData();
            }
        });

    @Override
    @SneakyThrows
    public boolean hasAnyPermissions(String... permissions) {
        // 跨租户跳过
        if (skipPermissionCheck()) {
            return true;
        }
        Long userId = getLoginUserId();
        if (userId == null) {
            return false;
        }
        // 查缓存 → 缓存未命中 → RPC 调用
        return hasAnyPermissionsCache.get(
            new KeyValue<>(userId, Arrays.asList(permissions)));
    }

    @Override
    @SneakyThrows
    public boolean hasAnyRoles(String... roles) {
        if (skipPermissionCheck()) {
            return true;
        }
        Long userId = getLoginUserId();
        if (userId == null) {
            return false;
        }
        return hasAnyRolesCache.get(
            new KeyValue<>(userId, Arrays.asList(roles)));
    }

    @Override
    public boolean hasAnyScopes(String... scope) {
        if (skipPermissionCheck()) {
            return true;
        }
        LoginUser user = SecurityFrameworkUtils.getLoginUser();
        if (user == null) {
            return false;
        }
        return CollUtil.containsAny(user.getScopes(), Arrays.asList(scope));
    }

    // hasPermission/hasRole/hasScope 委托给 hasAny* 方法
    @Override
    public boolean hasPermission(String permission) { return hasAnyPermissions(permission); }
    @Override
    public boolean hasRole(String role) { return hasAnyRoles(role); }
    @Override
    public boolean hasScope(String scope) { return hasAnyScopes(scope); }
}
```

关键设计：
- **Guava Cache**：1 分钟缓存，key 是 `(userId, permissions)` 组合，避免每次 RPC
- **跨租户跳过**：`skipPermissionCheck()` 返回 true 时直接放行
- **Feign RPC**：缓存未命中时调用 `PermissionCommonApi` 校验

## 注册到自动配置

```java
@Bean("ss") // Bean 名称 "ss"，方便 @PreAuthorize("@ss.xxx") 使用
public SecurityFrameworkService securityFrameworkService(PermissionCommonApi permissionApi) {
    return new SecurityFrameworkServiceImpl(permissionApi);
}
```

## 小结

`SecurityFrameworkServiceImpl` 实现了基于 RPC + 缓存的权限校验。业务代码通过 `@PreAuthorize("@ss.hasPermission('xxx')")` 即可实现方法级鉴权。
