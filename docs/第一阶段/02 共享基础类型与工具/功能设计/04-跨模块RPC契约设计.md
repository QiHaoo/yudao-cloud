# 跨模块 RPC 契约设计

---

## 1. 问题：框架层怎么调用业务模块？

在"三层模块体系"中，我们确立了一条铁律：**框架 Starter 不能依赖业务模块**。但框架层确实需要调用业务模块的功能：

- `yudao-spring-boot-starter-web` 的 `GlobalExceptionHandler` 需要把错误日志写入数据库 → 需要调用 infra 模块的 `ApiErrorLogService`
- `yudao-spring-boot-starter-security` 的 `SecurityFrameworkServiceImpl` 需要校验用户权限 → 需要调用 system 模块的 `PermissionService`
- `yudao-spring-boot-starter-web` 的 `ApiAccessLogFilter` 需要记录访问日志 → 需要调用 infra 模块的 `ApiAccessLogService`

如果框架 Starter 直接依赖 `yudao-module-system-server`，就会形成循环依赖。怎么办？

---

## 2. 解法：CommonApi 接口放在 yudao-common 中

yudao-cloud 的解法是：**把框架层需要调用的业务接口定义在 `yudao-common` 中**。

```
yudao-common
  ├── CommonResult.java
  ├── PageParam.java
  ├── ...
  └── biz/
      ├── system/
      │   ├── oauth2/OAuth2TokenCommonApi.java    ← Token 操作
      │   ├── permission/PermissionCommonApi.java  ← 权限校验
      │   ├── tenant/TenantCommonApi.java          ← 租户查询
      │   └── dict/DictDataCommonApi.java          ← 字典查询
      └── infra/
          └── logger/
              ├── ApiAccessLogCommonApi.java        ← 访问日志
              └── ApiErrorLogCommonApi.java         ← 错误日志
```

这些接口使用 `@FeignClient` 注解，指向对应的微服务：

```java
@FeignClient(name = "system-server")
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

### 依赖关系

```
yudao-common (定义 CommonApi 接口)
  ↑
  ├── yudao-spring-boot-starter-security (使用 PermissionCommonApi)
  ├── yudao-spring-boot-starter-web (使用 ApiAccessLogCommonApi, ApiErrorLogCommonApi)
  ↑
  ├── yudao-module-system-server (实现 PermissionCommonApi)
  └── yudao-module-infra-server (实现 ApiAccessLogCommonApi, ApiErrorLogCommonApi)
```

框架 Starter 依赖 `yudao-common`（最底层模块），通过接口调用业务功能。业务模块实现这些接口。依赖方向始终是单向的，没有循环。

---

## 3. 两级 API 体系

yudao-cloud 有两级 API 体系：

| 级别 | 位置 | 用途 | 示例 |
|------|------|------|------|
| **CommonApi** | `yudao-common` | 框架级跨模块调用 | `PermissionCommonApi`, `OAuth2TokenCommonApi` |
| **业务 Api** | `yudao-module-xxx-api` | 业务级跨模块调用 | `DeptApi`, `AdminUserApi`, `PayOrderApi` |

### CommonApi vs 业务 Api 的区别

| 维度 | CommonApi | 业务 Api |
|------|----------|----------|
| 定义位置 | `yudao-common` | `yudao-module-xxx-api` |
| 调用者 | 框架 Starter | 其他业务模块 |
| 稳定性 | 极高（框架级，很少变更） | 较高（业务级，随需求变更） |
| 接口数量 | 很少（每个模块 1-3 个） | 较多（每个模块 5-15 个） |
| 设计目的 | 打破框架→业务的反向依赖 | 模块间业务协作 |

### 典型的业务 Api

```java
// yudao-module-system-api/src/.../DeptApi.java
@FeignClient(name = "system-server")
@Tag(name = "RPC 服务 - 部门")
public interface DeptApi {
    String PREFIX = RpcConstants.RPC_API_PREFIX + "/system/dept";

    @GetMapping(PREFIX + "/get")
    CommonResult<DeptRespDTO> getDept(@RequestParam("id") Long id);

    @GetMapping(PREFIX + "/list-all-simple")
    CommonResult<List<DeptRespDTO>> getSimpleDeptList();

    @GetMapping(PREFIX + "/valid")
    CommonResult<Boolean> validDepts(@RequestParam("ids") Collection<Long> ids);
}
```

服务名 `system-server` 必须与目标服务的 `spring.application.name` 一致。

---

## 4. API 模块的轻量化设计

`yudao-module-system-api` 的 pom.xml 只依赖 `yudao-common`，且大部分依赖是 `optional`：

```xml
<dependencies>
    <dependency>
        <groupId>cn.iocoder.cloud</groupId>
        <artifactId>yudao-common</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
        <optional>true</optional>  <!-- 不传递给消费者 -->
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

`optional=true` 确保当其他模块依赖 `yudao-module-system-api` 时，不会自动拉入 OpenFeign 和 Validation 的传递依赖。这是因为：

1. 在单体模式下，OpenFeign 被排除了，不需要它
2. 只有实现 `@FeignClient` 接口的模块才真正需要 OpenFeign

---

## 5. RPC 调用的 URL 前缀

所有 RPC 接口统一使用 `/rpc-api/{module}/...` 前缀：

```java
// yudao-common/src/.../RpcConstants.java
public interface RpcConstants {
    String SYSTEM_NAME = "system-server";
    String INFRA_NAME = "infra-server";

    String RPC_API_PREFIX = "/rpc-api";
}
```

| 前缀 | 用途 | 来源 |
|------|------|------|
| `/admin-api/` | 管理端接口 | Controller 在 `*.controller.admin.*` 包下 |
| `/app-api/` | 用户端接口 | Controller 在 `*.controller.app.*` 包下 |
| `/rpc-api/` | 模块间 RPC | Feign 接口中定义 |

三种前缀由不同的机制注入：
- `/admin-api/` 和 `/app-api/` 由 `YudaoWebAutoConfiguration` 根据 Controller 包路径自动注入
- `/rpc-api/` 在 Feign 接口中显式写在 URL 路径里

---

## 思考题

1. 为什么 `CommonApi` 的数量要控制在"每个模块 1-3 个"？如果每个业务方法都定义一个 CommonApi 会怎样？
2. `@FeignClient(name = "system-server")` 中的 `name` 有什么作用？在单体模式和微服务模式下分别怎么解析？
3. 业务 Api 模块的依赖为什么用 `optional` 而不是 `provided`？两者有什么区别？
