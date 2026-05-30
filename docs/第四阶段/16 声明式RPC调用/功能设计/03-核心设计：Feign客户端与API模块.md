# 03 - 核心设计：Feign 客户端与 API 模块

## @FeignClient 注解的使用

### 基本用法

yudao-cloud 中的 Feign 客户端定义遵循统一规范：

```java
@FeignClient(name = ApiConstants.NAME)  // 指定服务名
@Tag(name = "RPC 服务 - 管理员用户")       // Swagger 文档标签
public interface AdminUserApi {

    String PREFIX = ApiConstants.PREFIX + "/user";  // URL 前缀

    @GetMapping(PREFIX + "/get")
    @Operation(summary = "通过用户 ID 查询用户")
    CommonResult<AdminUserRespDTO> getUser(@RequestParam("id") Long id);

    @GetMapping(PREFIX + "/list")
    @Operation(summary = "通过用户 ID 查询用户们")
    CommonResult<List<AdminUserRespDTO>> getUserList(@RequestParam("ids") Collection<Long> ids);
}
```

### @FeignClient 的核心属性

| 属性 | 作用 | 示例 |
|------|------|------|
| `name` | 目标服务名，对应 `spring.application.name` | `system-server` |
| `path` | URL 前缀（yudao-cloud 未使用，采用常量方式） | - |
| `fallbackFactory` | 降级工厂（yudao-cloud 暂未启用） | - |
| `primary` | 是否为主 Bean | `false`（某些接口） |

### 服务名的定义规范

yudao-cloud 中，服务名通过 `ApiConstants` 统一定义：

```java
// yudao-module-system-api 中的 ApiConstants
public class ApiConstants {
    public static final String NAME = "system-server";  // 服务名
    public static final String PREFIX = RpcConstants.RPC_API_PREFIX + "/system";  // URL 前缀
}

// yudao-framework/yudao-common 中的 RpcConstants
public interface RpcConstants {
    String RPC_API_PREFIX = "/rpc-api";  // RPC API 统一前缀
    String SYSTEM_NAME = "system-server";
    String SYSTEM_PREFIX = RPC_API_PREFIX + "/system";
    String INFRA_NAME = "infra-server";
    String INFRA_PREFIX = RPC_API_PREFIX + "/infra";
}
```

## *-api 模块的目录结构和命名规范

### 目录结构

```
yudao-module-product-api/
└── src/main/java/
    └── cn/iocoder/yudao/module/product/
        ├── api/                        ← Feign 接口
        │   ├── spu/
        │   │   ├── ProductSpuApi.java  ← SPU 接口
        │   │   └── dto/
        │   │       └── ProductSpuRespVO.java
        │   ├── sku/
        │   │   ├── ProductSkuApi.java
        │   │   └── dto/
        │   │       └── ProductSkuRespVO.java
        │   └── category/
        │       ├── ProductCategoryApi.java
        │       └── dto/
        │           └── ProductCategoryRespVO.java
        └── enums/                      ← 枚举常量
            ├── ApiConstants.java       ← 服务名和前缀
            ├── ErrorCodeConstants.java ← 错误码
            └── DictTypeConstants.java  ← 字典类型
```

### 命名规范

| 元素 | 命名规范 | 示例 |
|------|----------|------|
| API 接口 | `{业务}Api` | `ProductSpuApi` |
| 响应 DTO | `{业务}RespVO` | `ProductSpuRespVO` |
| 请求 DTO | `{业务}SaveReqVO` / `{业务}PageReqVO` | `ProductSpuSaveReqVO` |
| API 常量 | `ApiConstants` | `ApiConstants.NAME` |
| 错误码 | `ErrorCodeConstants` | `ErrorCodeConstants.SPU_NOT_EXISTS` |

### API 接口的定义规范

```java
@FeignClient(name = ApiConstants.NAME)
@Tag(name = "RPC 服务 - 商品 SPU")
public interface ProductSpuApi {

    String PREFIX = ApiConstants.PREFIX + "/spu";

    // 查询单个
    @GetMapping(PREFIX + "/get")
    @Operation(summary = "获得商品 SPU")
    CommonResult<ProductSpuRespVO> getSpu(@RequestParam("id") Long id);

    // 查询列表
    @GetMapping(PREFIX + "/list")
    @Operation(summary = "获得商品 SPU 列表")
    CommonResult<List<ProductSpuRespVO>> getSpuList(@RequestParam("ids") Collection<Long> ids);

    // 校验有效性
    @GetMapping(PREFIX + "/valid")
    @Operation(summary = "校验商品 SPU 是否有效")
    CommonResult<Boolean> validateSpu(@RequestParam("id") Long id);
}
```

## RPC URL 前缀约定

### 前缀规则

yudao-cloud 使用统一的 RPC URL 前缀，避免与业务 API 路径冲突：

```
业务 API 前缀：
  管理后台：/admin-api/{module}/...
  用户端：  /app-api/{module}/...

RPC API 前缀：
  服务间调用：/rpc-api/{module}/...
```

### URL 路径对照

```
AdminUserApi 的完整 URL：
  /rpc-api/system/user/get

ProductSpuApi 的完整 URL：
  /rpc-api/product/spu/get

TradeOrderApi 的完整 URL：
  /rpc-api/trade/order/get
```

### 前缀定义方式

```java
// 方式一：在 RpcConstants 中定义（框架级 API）
public interface RpcConstants {
    String RPC_API_PREFIX = "/rpc-api";
    String SYSTEM_PREFIX = RPC_API_PREFIX + "/system";
}

// 方式二：在 ApiConstants 中定义（业务级 API）
public class ApiConstants {
    public static final String PREFIX = RpcConstants.RPC_API_PREFIX + "/product";
}
```

### 为什么需要统一前缀

```
没有统一前缀的问题：

ProductSpuApi:  @GetMapping("/product/spu/get")
ProductSpuController: @GetMapping("/product/spu/get")  ← 路径冲突！

有统一前缀：

ProductSpuApi:  @GetMapping("/rpc-api/product/spu/get")     ← RPC 调用
ProductSpuController: @GetMapping("/admin-api/product/spu/get")  ← 管理后台
ProductSpuController: @GetMapping("/app-api/product/spu/get")    ← 用户端
```

## Feign 请求拦截器

### TenantRequestInterceptor

用于在 RPC 调用时传递租户 ID：

```java
public class TenantRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // 从当前上下文获取租户 ID
        Long tenantId = TenantContextHolder.getTenantId();
        if (tenantId != null) {
            // 添加到请求头
            template.header("tenant-id", String.valueOf(tenantId));
        }
    }
}
```

工作流程：

```
trade-server 发起 RPC 调用
    │
    ├─→ TenantRequestInterceptor 拦截
    │   ├── 从 TenantContextHolder 获取当前租户 ID
    │   └── 添加到请求头 tenant-id
    │
    ▼
product-server 接收请求
    │
    ├─→ 从请求头读取 tenant-id
    ├─→ 设置到 TenantContextHolder
    └─→ 业务代码可以正常使用租户 ID
```

### DataPermissionRequestInterceptor

用于在 RPC 调用时传递数据权限信息：

```java
public class DataPermissionRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // 从当前上下文获取数据权限
        DataPermission dataPermission = DataPermissionContextHolder.get();
        if (dataPermission != null) {
            template.header("data-permission", JsonUtils.toJsonString(dataPermission));
        }
    }
}
```

### 拦截器的作用

```
┌─────────────────────────────────────────────────────────────────┐
│                    Feign 请求拦截器链                            │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  TenantRequestInterceptor                                │  │
│  │  ├── 读取 TenantContextHolder                            │  │
│  │  └── 添加 tenant-id 到请求头                             │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  DataPermissionRequestInterceptor                        │  │
│  │  ├── 读取 DataPermissionContextHolder                    │  │
│  │  └── 添加 data-permission 到请求头                       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │  其他自定义拦截器                                         │  │
│  │  ├── 记录调用日志                                        │  │
│  │  └── 添加追踪 ID                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Feign 错误解码器

### 默认错误处理的问题

当远程服务返回非 200 状态码时，Feign 默认会抛出 `FeignException`，但这个异常信息不够友好。

### 自定义错误解码器

yudao-cloud 可以自定义 `ErrorDecoder` 来翻译远程异常：

```java
public class FeignErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        // 读取响应体
        String body = Util.toString(response.body().asReader(StandardCharsets.UTF_8));

        // 解析为 CommonResult
        CommonResult<?> result = JsonUtils.parseObject(body, CommonResult.class);
        if (result != null) {
            // 将远程错误码翻译为本地异常
            return new ServiceException(result.getCode(), result.getMsg());
        }

        // 兜底处理
        return new ServiceException(INTERNAL_SERVER_ERROR.getCode(), "远程调用失败");
    }
}
```

错误处理流程：

```
trade-server 调用 product-server
    │
    ▼
product-server 返回 500 + CommonResult
    │
    ▼
FeignErrorDecoder 拦截
    │
    ├── 解析响应体为 CommonResult
    ├── 提取错误码和错误信息
    └── 抛出 ServiceException
    │
    ▼
trade-server 的 GlobalExceptionHandler 捕获
    │
    └── 返回统一的错误响应
```

## Feign 日志级别配置

### 日志级别

| 级别 | 输出内容 |
|------|----------|
| `NONE` | 不输出任何日志（默认） |
| `BASIC` | 请求方法、URL、响应状态码、执行时间 |
| `HEADERS` | BASIC + 请求头和响应头 |
| `FULL` | HEADERS + 请求体和响应体 |

### 配置方式

```yaml
# application.yaml
logging:
  level:
    cn.iocoder.yudao.module.system.api: DEBUG  # 开启 Feign 日志
```

```java
// 配置类
@Configuration
public class FeignConfiguration {

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.FULL;  // 开启完整日志
    }
}
```

## 与 Sentinel 的集成：熔断降级

### 集成方式

在 `pom.xml` 中引入 Sentinel 依赖后，Feign 自动与 Sentinel 集成：

```yaml
# application.yaml
spring:
  cloud:
    openfeign:
      sentinel:
        enabled: true  # 开启 Sentinel 熔断降级
```

### Fallback 降级

```java
@FeignClient(name = ApiConstants.NAME, fallbackFactory = ProductSpuApiFallbackFactory.class)
public interface ProductSpuApi {

    @GetMapping(PREFIX + "/get")
    CommonResult<ProductSpuRespVO> getSpu(@RequestParam("id") Long id);
}

// 降级工厂
@Component
public class ProductSpuApiFallbackFactory implements FallbackFactory<ProductSpuApi> {

    @Override
    public ProductSpuApi create(Throwable cause) {
        return new ProductSpuApi() {
            @Override
            public CommonResult<ProductSpuRespVO> getSpu(Long id) {
                // 降级逻辑：返回默认值或缓存数据
                return CommonResult.error(SERVICE_UNAVAILABLE.getCode(), "商品服务暂时不可用");
            }
        };
    }
}
```

### 降级策略

```
正常流程：
  trade-server → product-server → 返回商品信息

降级流程（product-server 不可用）：
  trade-server → Sentinel 检测到异常
      │
      ├── 快速失败：直接返回错误
      ├── 返回默认值：返回缓存的商品信息
      └── 调用备用服务：调用其他数据源
```

## 思考题

1. **yudao-cloud 中的 Feign 接口都定义在 *-api 模块中，而不是在调用方模块中。如果调用方只需要调用远程服务的一个方法，但它需要依赖整个 *-api 模块，这会造成什么问题？如何优化？**

2. **Feign 请求拦截器的执行顺序是什么？如果 TenantRequestInterceptor 和 DataPermissionRequestInterceptor 都需要执行，它们的顺序如何控制？**

3. **为什么 yudao-cloud 暂未启用 Sentinel fallback？在什么场景下必须启用熔断降级？**

4. **Feign 的日志级别设置为 `FULL` 会输出请求体和响应体。在生产环境中，这可能会有什么风险？**
