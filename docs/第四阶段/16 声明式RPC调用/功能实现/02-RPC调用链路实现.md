# 02 - RPC 调用链路实现

> 本章介绍 RPC 调用链路中的高级特性：请求拦截器、错误解码器、URL 前缀约定、CommonResult 封装。

## RPC 调用链路全景

在深入实现之前，先理解完整的 RPC 调用链路：

```
调用方代码
    │
    ▼
Feign 代理（JDK 动态代理）
    │
    ├── 1. 解析方法注解，构建 HTTP 请求
    ├── 2. 执行请求拦截器链
    │       ├── TenantRequestInterceptor（传递租户 ID）
    │       ├── DataPermissionRequestInterceptor（传递数据权限）
    │       └── 其他自定义拦截器
    ├── 3. 通过 LoadBalancer 选择实例
    ├── 4. 发起 HTTP 请求
    ├── 5. 处理响应
    │       ├── 成功：反序列化为 CommonResult<T>
    │       └── 失败：FeignErrorDecoder 处理
    └── 6. 返回结果
    │
    ▼
返回 CommonResult<T>
```

## 请求拦截器

### 为什么需要请求拦截器

在微服务架构中，服务间调用需要传递一些上下文信息：

```
场景一：多租户系统
  trade-server 发起请求时，需要携带租户 ID
  product-server 接收请求后，根据租户 ID 过滤数据

场景二：数据权限
  trade-server 发起请求时，需要携带数据权限信息
  product-server 接收请求后，根据数据权限过滤数据

场景三：链路追踪
  每次请求都需要携带 trace-id，便于日志追踪
```

**没有拦截器的问题：**

```java
// 每次调用都要手动添加请求头
@GetMapping("/rpc-api/product/spu/get")
public CommonResult<ProductSpuRespDTO> getSpu(@RequestParam("id") Long id) {
    // 手动添加租户 ID
    HttpHeaders headers = new HttpHeaders();
    headers.add("tenant-id", TenantContextHolder.getTenantId().toString());

    // 手动构建请求
    HttpEntity<?> request = new HttpEntity<>(headers);
    return restTemplate.exchange(url, HttpMethod.GET, request, ...);
}
```

**有拦截器后：**

```java
// 直接调用，拦截器自动处理
CommonResult<ProductSpuRespDTO> result = productSpuApi.getSpu(spuId);
```

### TenantRequestInterceptor 实现

```java
// yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/web/core/handler/TenantRequestInterceptor.java
package cn.iocoder.yudao.framework.web.core.handler;

import cn.iocoder.yudao.framework.tenant.core.context.TenantContextHolder;
import feign.RequestInterceptor;
import feign.RequestTemplate;

/**
 * 租户请求拦截器
 *
 * 在 RPC 调用时，自动传递租户 ID
 */
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

**工作流程：**

```
trade-server 发起 RPC 调用
    │
    ▼
TenantRequestInterceptor 拦截
    │
    ├── 从 TenantContextHolder 获取当前租户 ID
    ├── 添加到请求头 tenant-id
    │
    ▼
product-server 接收请求
    │
    ├── 从请求头读取 tenant-id
    ├── 设置到 TenantContextHolder
    └── 业务代码可以正常使用租户 ID
```

### DataPermissionRequestInterceptor 实现

```java
// yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/web/core/handler/DataPermissionRequestInterceptor.java
package cn.iocoder.yudao.framework.web.core.handler;

import cn.iocoder.yudao.framework.datapermission.core.annotation.DataPermission;
import cn.iocoder.yudao.framework.datapermission.core.context.DataPermissionContextHolder;
import cn.iocoder.yudao.framework.common.util.json.JsonUtils;
import feign.RequestInterceptor;
import feign.RequestTemplate;

/**
 * 数据权限请求拦截器
 *
 * 在 RPC 调用时，自动传递数据权限信息
 */
public class DataPermissionRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        // 从当前上下文获取数据权限
        DataPermission dataPermission = DataPermissionContextHolder.get();
        if (dataPermission != null) {
            // 添加到请求头（JSON 格式）
            template.header("data-permission", JsonUtils.toJsonString(dataPermission));
        }
    }

}
```

### 注册拦截器

拦截器需要注册到 Feign 配置中：

```java
// 方式一：全局配置（推荐）
@Configuration
public class FeignConfiguration {

    @Bean
    public TenantRequestInterceptor tenantRequestInterceptor() {
        return new TenantRequestInterceptor();
    }

    @Bean
    public DataPermissionRequestInterceptor dataPermissionRequestInterceptor() {
        return new DataPermissionRequestInterceptor();
    }
}
```

```yaml
# 方式二：通过配置文件
spring:
  cloud:
    openfeign:
      client:
        config:
          default:
            requestInterceptors:
              - cn.iocoder.yudao.framework.web.core.handler.TenantRequestInterceptor
              - cn.iocoder.yudao.framework.web.core.handler.DataPermissionRequestInterceptor
```

## 错误解码器

### 默认错误处理的问题

当远程服务返回非 200 状态码时，Feign 默认会抛出 `FeignException`，但这个异常信息不够友好：

```java
// 默认行为
try {
    CommonResult<ProductSpuRespDTO> result = productSpuApi.getSpu(spuId);
} catch (FeignException e) {
    // 异常信息不友好，无法获取业务错误码
    log.error("RPC 调用失败", e);
}
```

### 自定义 FeignErrorDecoder

```java
// yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/web/core/handler/FeignErrorDecoder.java
package cn.iocoder.yudao.framework.web.core.handler;

import cn.iocoder.yudao.framework.common.exception.ServiceException;
import cn.iocoder.yudao.framework.common.pojo.CommonResult;
import cn.iocoder.yudao.framework.common.util.json.JsonUtils;
import feign.Response;
import feign.codec.ErrorDecoder;
import lombok.extern.slf4j.Slf4j;

import java.io.IOException;
import java.nio.charset.StandardCharsets;

import static cn.iocoder.yudao.framework.common.exception.enums.GlobalErrorCodeConstants.INTERNAL_SERVER_ERROR;

/**
 * Feign 错误解码器
 *
 * 将远程异常翻译为本地异常
 */
@Slf4j
public class FeignErrorDecoder implements ErrorDecoder {

    @Override
    public Exception decode(String methodKey, Response response) {
        // 读取响应体
        String body;
        try {
            body = Util.toString(response.body().asReader(StandardCharsets.UTF_8));
        } catch (IOException e) {
            log.error("[decode][读取响应体失败，方法：{}]", methodKey, e);
            return new ServiceException(INTERNAL_SERVER_ERROR.getCode(), "远程调用失败");
        }

        // 解析为 CommonResult
        CommonResult<?> result;
        try {
            result = JsonUtils.parseObject(body, CommonResult.class);
        } catch (Exception e) {
            log.error("[decode][解析响应体失败，方法：{}，响应体：{}]", methodKey, body, e);
            return new ServiceException(INTERNAL_SERVER_ERROR.getCode(), "远程调用失败");
        }

        // 将远程错误码翻译为本地异常
        if (result != null && result.isError()) {
            return new ServiceException(result.getCode(), result.getMsg());
        }

        // 兜底处理
        return new ServiceException(INTERNAL_SERVER_ERROR.getCode(), "远程调用失败");
    }

}
```

### 错误处理流程

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

### 注册错误解码器

```java
@Configuration
public class FeignConfiguration {

    @Bean
    public ErrorDecoder errorDecoder() {
        return new FeignErrorDecoder();
    }
}
```

## URL 前缀约定

### 三套 API 前缀

yudao-cloud 定义了三套 API 前缀，避免路径冲突：

```
管理后台 API：/admin-api/{module}/...
  示例：/admin-api/product/spu/get
  用途：前端管理后台调用

用户端 API：/app-api/{module}/...
  示例：/app-api/product/spu/get
  用途：用户端 App 调用

RPC 调用 API：/rpc-api/{module}/...
  示例：/rpc-api/product/spu/get
  用途：服务间调用
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

## CommonResult 封装

### 为什么返回 CommonResult

**直接返回 DTO 的问题：**

```java
// 直接返回 DTO
ProductSpuRespDTO getSpu(Long id);

// 调用方无法知道是否成功
ProductSpuRespDTO spu = productSpuApi.getSpu(spuId);
// 如果 spu 是 null，是查询不到还是调用失败？
```

**返回 CommonResult 的优势：**

```java
// 返回 CommonResult
CommonResult<ProductSpuRespDTO> getSpu(Long id);

// 调用方可以明确处理
CommonResult<ProductSpuRespDTO> result = productSpuApi.getSpu(spuId);
if (result.isError()) {
    // 明确知道是业务错误
    throw new ServiceException(result.getCode(), result.getMsg());
}
ProductSpuRespDTO spu = result.getCheckedData();
```

### CommonResult 的使用规范

```java
// Feign 接口定义
@FeignClient(name = ApiConstants.NAME)
public interface ProductSpuApi {

    @GetMapping(PREFIX + "/get")
    CommonResult<ProductSpuRespDTO> getSpu(@RequestParam("id") Long id);

    @GetMapping(PREFIX + "/list")
    CommonResult<List<ProductSpuRespDTO>> getSpuList(@RequestParam("ids") Collection<Long> ids);

    @GetMapping(PREFIX + "/valid")
    CommonResult<Boolean> validateSpuList(@RequestParam("ids") Collection<Long> ids);
}

// Controller 实现
@RestController
@RequestMapping("/rpc-api/product/spu")
public class ProductSpuController implements ProductSpuApi {

    @Override
    @GetMapping("/get")
    public CommonResult<ProductSpuRespDTO> getSpu(@RequestParam("id") Long id) {
        ProductSpuRespDTO spu = productService.getSpu(id);
        return CommonResult.success(spu);  // 成功时包装
    }

    @Override
    @GetMapping("/valid")
    public CommonResult<Boolean> validateSpuList(@RequestParam("ids") Collection<Long> ids) {
        productService.validateSpuList(ids);
        return CommonResult.success(true);  // 校验成功
    }
}
```

### getCheckedData() 方法

`getCheckedData()` 是一个便捷方法，用于获取数据并在失败时抛出异常：

```java
// CommonResult 中的 getCheckedData()
public T getCheckedData() {
    if (isError()) {
        throw new ServiceException(code, msg);
    }
    return data;
}

// 使用示例
CommonResult<ProductSpuRespDTO> result = productSpuApi.getSpu(spuId);
ProductSpuRespDTO spu = result.getCheckedData();
// 如果 result 是错误，会自动抛出 ServiceException
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
    cn.iocoder.yudao.module.product.api: DEBUG  # 开启 Feign 日志
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

**注意事项：**

- 生产环境不要使用 `FULL` 级别，会输出请求体和响应体，可能包含敏感信息
- 建议在开发环境使用 `BASIC` 或 `HEADERS`，生产环境使用 `NONE`

## 与 Sentinel 集成：熔断降级

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
    CommonResult<ProductSpuRespDTO> getSpu(@RequestParam("id") Long id);
}

// 降级工厂
@Component
public class ProductSpuApiFallbackFactory implements FallbackFactory<ProductSpuApi> {

    @Override
    public ProductSpuApi create(Throwable cause) {
        return new ProductSpuApi() {
            @Override
            public CommonResult<ProductSpuRespDTO> getSpu(Long id) {
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

## 完整的 Feign 配置类

```java
// yudao-framework/yudao-spring-boot-starter-web/src/main/java/cn/iocoder/yudao/framework/web/config/YudaoFeignAutoConfiguration.java
package cn.iocoder.yudao.framework.web.config;

import cn.iocoder.yudao.framework.web.core.handler.DataPermissionRequestInterceptor;
import cn.iocoder.yudao.framework.web.core.handler.FeignErrorDecoder;
import cn.iocoder.yudao.framework.web.core.handler.TenantRequestInterceptor;
import feign.codec.ErrorDecoder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * Feign 自动配置类
 */
@Configuration
public class YudaoFeignAutoConfiguration {

    @Bean
    public TenantRequestInterceptor tenantRequestInterceptor() {
        return new TenantRequestInterceptor();
    }

    @Bean
    public DataPermissionRequestInterceptor dataPermissionRequestInterceptor() {
        return new DataPermissionRequestInterceptor();
    }

    @Bean
    public ErrorDecoder errorDecoder() {
        return new FeignErrorDecoder();
    }

}
```

## 本章小结

本章介绍了 RPC 调用链路的实现：

1. **请求拦截器**：自动传递租户 ID、数据权限等上下文信息
2. **错误解码器**：将远程异常翻译为本地异常，统一错误处理
3. **URL 前缀约定**：使用 `/rpc-api` 前缀避免路径冲突
4. **CommonResult 封装**：统一响应格式，便于错误处理
5. **日志配置**：不同环境使用不同的日志级别
6. **Sentinel 集成**：熔断降级，提高系统可用性

下一章将介绍测试支持和源码索引，帮助你在本地开发时高效调试 Feign 调用。
