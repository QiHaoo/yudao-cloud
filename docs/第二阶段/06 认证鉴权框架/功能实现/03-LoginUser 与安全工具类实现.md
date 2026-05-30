# 03-LoginUser 与安全工具类实现

## 目标

实现登录用户信息载体 `LoginUser` 和安全工具类 `SecurityFrameworkUtils`，为后续的 Token 认证和权限校验提供数据基础。

## LoginUser 实现

```java
// 文件路径：core/LoginUser.java
@Data
public class LoginUser {

    public static final String INFO_KEY_NICKNAME = "nickname";
    public static final String INFO_KEY_DEPT_ID = "deptId";

    private Long id;                    // 用户编号
    private Integer userType;           // 用户类型（1-会员 2-管理员）
    private Map<String, String> info;   // 额外信息
    private Long tenantId;              // 租户编号
    private List<String> scopes;        // OAuth2 授权范围
    private LocalDateTime expiresTime;  // 过期时间

    @JsonIgnore  // 不序列化到 login-user Header
    private Map<String, Object> context; // 请求级临时缓存

    private Long visitTenantId;         // 访问的租户编号

    // context 的便捷操作方法
    public void setContext(String key, Object value) {
        if (context == null) {
            context = new HashMap<>();
        }
        context.put(key, value);
    }

    public <T> T getContext(String key, Class<T> type) {
        return MapUtil.get(context, key, type);
    }
}
```

设计要点：
- `info` 用 `Map<String, String>` 而非固定字段，可自由扩展
- `context` 用 `@JsonIgnore` 标注，避免被序列化到 login-user Header（只在当前请求内有效）
- `visitTenantId` 支持跨租户访问场景

## SecurityFrameworkUtils 实现

```java
// 文件路径：core/util/SecurityFrameworkUtils.java
public class SecurityFrameworkUtils {

    public static final String AUTHORIZATION_BEARER = "Bearer";
    public static final String LOGIN_USER_HEADER = "login-user";

    private SecurityFrameworkUtils() {}

    /**
     * 从请求中提取 Token
     * 优先级：Header > Parameter，支持 Bearer 前缀
     */
    public static String obtainAuthorization(HttpServletRequest request,
                                             String headerName, String parameterName) {
        String token = request.getHeader(headerName);
        if (StrUtil.isEmpty(token)) {
            token = request.getParameter(parameterName);
        }
        if (!StringUtils.hasText(token)) {
            return null;
        }
        // 去除 Bearer 前缀
        int index = token.indexOf(AUTHORIZATION_BEARER + " ");
        return index >= 0 ? token.substring(index + 7).trim() : token;
    }

    /**
     * 获取当前登录用户
     */
    @Nullable
    public static LoginUser getLoginUser() {
        Authentication authentication = getAuthentication();
        if (authentication == null) {
            return null;
        }
        return authentication.getPrincipal() instanceof LoginUser
                ? (LoginUser) authentication.getPrincipal() : null;
    }

    /**
     * 获取当前用户 ID（便捷方法）
     */
    @Nullable
    public static Long getLoginUserId() {
        LoginUser loginUser = getLoginUser();
        return loginUser != null ? loginUser.getId() : null;
    }

    /**
     * 设置当前用户到 SecurityContextHolder
     */
    public static void setLoginUser(LoginUser loginUser, HttpServletRequest request) {
        // 1. 放入 SecurityContextHolder
        UsernamePasswordAuthenticationToken authentication =
            new UsernamePasswordAuthenticationToken(loginUser, null, Collections.emptyList());
        authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
        SecurityContextHolder.getContext().setAuthentication(authentication);

        // 2. 放入 Request Attribute（供 ApiAccessLogFilter 使用）
        if (request != null) {
            WebFrameworkUtils.setLoginUserId(request, loginUser.getId());
            WebFrameworkUtils.setLoginUserType(request, loginUser.getUserType());
        }
    }

    /**
     * 是否跳过权限校验（跨租户场景）
     */
    public static boolean skipPermissionCheck() {
        LoginUser loginUser = getLoginUser();
        if (loginUser == null || loginUser.getVisitTenantId() == null) {
            return false;
        }
        return ObjUtil.notEqual(loginUser.getVisitTenantId(), loginUser.getTenantId());
    }
}
```

关键设计：
- `obtainAuthorization()`：支持 `Authorization: Bearer xxx` 和 `?token=xxx` 两种方式
- `setLoginUser()`：同时写入 SecurityContextHolder 和 Request Attribute，解决访问日志获取不到用户的问题
- `skipPermissionCheck()`：跨租户访问时跳过权限校验

## 小结

本章实现了两个核心类：
- `LoginUser`：登录用户信息载体，支持多用户类型、多租户、可扩展信息
- `SecurityFrameworkUtils`：安全工具类，提供 Token 提取、用户读写、权限跳过等静态方法

它们是整个认证鉴权框架的数据基础。
