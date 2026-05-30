# 01 - 起步：Feign 客户端与 API 模块

> 本章是声明式 RPC 调用的起点。完成本章后，你将理解 *-api 模块的设计，并能定义第一个 Feign 客户端接口。

## 前置准备

在开始之前，请确保：

1. 已完成第四阶段组件 15（服务注册与配置中心）的配置
2. Nacos Server 已启动，服务注册功能正常
3. 已完成第一阶段的功能实现（项目骨架、共享基础类型）

## 为什么需要 *-api 模块

### 问题场景

假设 trade-server（交易服务）需要调用 product-server（商品服务）查询商品信息：

**没有 *-api 模块的情况：**

```java
// trade-server 中定义 DTO
public class ProductSpuDTO {
    private Long id;
    private String name;
    private Integer price;
}

// product-server 中定义 VO
public class ProductSpuRespVO {
    private Long spuId;      // 字段名不同！
    private String spuName;  // 字段名不同！
    private Integer price;
}

// 调用时需要手动映射，容易出错
ProductSpuDTO dto = new ProductSpuDTO();
dto.setId(vo.getSpuId());
dto.setName(vo.getSpuName());
dto.setPrice(vo.getPrice());
```

**有 *-api 模块的情况：**

```java
// yudao-module-product-api 中定义 DTO（共享）
public class ProductSpuRespDTO {
    private Long id;
    private String name;
    private Integer price;
}

// trade-server 和 product-server 共享同一个 DTO
// 编译时就能发现类型错误，无需手动映射
```

### *-api 模块的设计价值

```
yudao-module-product-api（共享模块）
├── ProductSpuApi.java              ← Feign 接口定义
└── dto/
    ├── ProductSpuRespDTO.java       ← 响应 DTO
    └── ProductSkuRespDTO.java       ← 响应 DTO

trade-server 依赖 yudao-module-product-api
product-server 依赖 yudao-module-product-api

两个服务共享同一份接口契约，编译时就能发现类型错误！
```

## *-api 模块的目录结构

### 标准目录结构

以 product 模块为例：

```
yudao-module-product-api/
└── src/main/java/
    └── cn/iocoder/yudao/module/product/
        ├── api/                        ← Feign 接口
        │   ├── spu/
        │   │   ├── ProductSpuApi.java  ← SPU 接口
        │   │   └── dto/
        │   │       └── ProductSpuRespDTO.java
        │   ├── sku/
        │   │   ├── ProductSkuApi.java
        │   │   └── dto/
        │   │       └── ProductSkuRespDTO.java
        │   └── category/
        │       ├── ProductCategoryApi.java
        │       └── dto/
        │           └── ProductCategoryRespDTO.java
        └── enums/                      ← 枚举常量
            ├── ApiConstants.java       ← 服务名和前缀
            ├── ErrorCodeConstants.java ← 错误码
            └── DictTypeConstants.java  ← 字典类型
```

### 命名规范

| 元素 | 命名规范 | 示例 |
|------|----------|------|
| API 接口 | `{业务}Api` | `ProductSpuApi` |
| 响应 DTO | `{业务}RespDTO` | `ProductSpuRespDTO` |
| 请求 DTO | `{业务}SaveReqDTO` / `{业务}PageReqDTO` | `ProductSpuSaveReqDTO` |
| API 常量 | `ApiConstants` | `ApiConstants.NAME` |
| 错误码 | `ErrorCodeConstants` | `ErrorCodeConstants.SPU_NOT_EXISTS` |

## 第一步：创建 *-api 模块

### 1.1 创建模块目录

以 product 模块为例，创建以下目录结构：

```
yudao-module-mall/yudao-module-product-api/
├── pom.xml
└── src/main/java/cn/iocoder/yudao/module/product/
    ├── api/
    │   ├── spu/
    │   │   ├── ProductSpuApi.java
    │   │   └── dto/
    │   │       └── ProductSpuRespDTO.java
    │   └── sku/
    │       ├── ProductSkuApi.java
    │       └── dto/
    │           └── ProductSkuRespDTO.java
    └── enums/
        └── ApiConstants.java
```

### 1.2 配置 pom.xml

```xml
<!-- yudao-module-mall/yudao-module-product-api/pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <groupId>cn.iocoder.cloud</groupId>
        <artifactId>yudao-module-mall</artifactId>
        <version>${revision}</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>yudao-module-product-api</artifactId>
    <packaging>jar</packaging>

    <name>${project.artifactId}</name>
    <description>
        product 模块 API，暴露给其它模块调用
    </description>

    <dependencies>
        <!-- 公共模块 -->
        <dependency>
            <groupId>cn.iocoder.cloud</groupId>
            <artifactId>yudao-common</artifactId>
        </dependency>

        <!-- Web 相关 -->
        <dependency>
            <groupId>org.springdoc</groupId>
            <artifactId>springdoc-openapi-ui</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- 参数校验 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- RPC 远程调用相关 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
            <optional>true</optional>
        </dependency>
    </dependencies>
</project>
```

**依赖说明：**

| 依赖 | 作用 | scope |
|------|------|-------|
| `yudao-common` | 公共基础类（CommonResult 等） | compile |
| `springdoc-openapi-ui` | Swagger 注解支持 | provided |
| `spring-boot-starter-validation` | 参数校验注解 | optional |
| `spring-cloud-starter-openfeign` | Feign 注解支持 | optional |

**为什么 OpenFeign 依赖是 optional？**

因为 *-api 模块可能被非 Spring Cloud 环境引用（如单体模式），使用 `optional` 可以避免强制依赖。

## 第二步：定义 ApiConstants

### 2.1 创建 ApiConstants

```java
// yudao-module-mall/yudao-module-product-api/src/main/java/cn/iocoder/yudao/module/product/enums/ApiConstants.java
package cn.iocoder.yudao.module.product.enums;

import cn.iocoder.yudao.framework.common.enums.RpcConstants;

/**
 * API 相关的枚举
 */
public class ApiConstants {

    /**
     * 服务名
     *
     * 注意，需要保证和 spring.application.name 保持一致
     */
    public static final String NAME = "product-server";

    /**
     * URL 前缀
     */
    public static final String PREFIX = RpcConstants.RPC_API_PREFIX + "/product";

    /**
     * 版本号
     */
    public static final String VERSION = "1.0.0";

}
```

### 2.2 RpcConstants 定义

`RpcConstants` 定义在 `yudao-common` 中，是所有 RPC 接口的基础常量：

```java
// yudao-framework/yudao-common/src/main/java/cn/iocoder/yudao/framework/common/enums/RpcConstants.java
package cn.iocoder.yudao.framework.common.enums;

/**
 * RPC 相关的枚举
 */
public interface RpcConstants {

    /**
     * RPC API 的前缀
     */
    String RPC_API_PREFIX = "/rpc-api";

    /**
     * system 服务名
     */
    String SYSTEM_NAME = "system-server";

    /**
     * system 服务的前缀
     */
    String SYSTEM_PREFIX = RPC_API_PREFIX + "/system";

    /**
     * infra 服务名
     */
    String INFRA_NAME = "infra-server";

    /**
     * infra 服务的前缀
     */
    String INFRA_PREFIX = RPC_API_PREFIX + "/infra";

}
```

**URL 前缀的设计：**

```
业务 API 前缀：
  管理后台：/admin-api/{module}/...
  用户端：  /app-api/{module}/...

RPC API 前缀：
  服务间调用：/rpc-api/{module}/...
```

使用统一的 `/rpc-api` 前缀，避免与业务 API 路径冲突。

## 第三步：定义 DTO

### 3.1 响应 DTO

```java
// yudao-module-mall/yudao-module-product-api/src/main/java/cn/iocoder/yudao/module/product/api/spu/dto/ProductSpuRespDTO.java
package cn.iocoder.yudao.module.product.api.spu.dto;

import io.swagger.v3.oas.annotations.media.Schema;
import lombok.Data;

import java.util.List;

@Schema(description = "RPC 服务 - 商品 SPU Response DTO")
@Data
public class ProductSpuRespDTO {

    @Schema(description = "SPU 编号", requiredMode = Schema.RequiredMode.REQUIRED, example = "1024")
    private Long id;

    @Schema(description = "SPU 名称", requiredMode = Schema.RequiredMode.REQUIRED, example = "iPhone 15")
    private String name;

    @Schema(description = "商品分类编号", requiredMode = Schema.RequiredMode.REQUIRED, example = "1")
    private Long categoryId;

    @Schema(description = "商品图片", requiredMode = Schema.RequiredMode.REQUIRED, example = "https://www.iocoder.cn/1.png")
    private String picUrl;

    @Schema(description = "销量", requiredMode = Schema.RequiredMode.REQUIRED, example = "100")
    private Integer salesCount;

    @Schema(description = "价格，单位：分", requiredMode = Schema.RequiredMode.REQUIRED, example = "100")
    private Integer price;

    @Schema(description = "库存", requiredMode = Schema.RequiredMode.REQUIRED, example = "100")
    private Integer stock;

}
```

### 3.2 DTO 设计原则

| 原则 | 说明 | 示例 |
|------|------|------|
| 只包含必要字段 | 不暴露内部实现细节 | 不包含 `create_time`、`update_time` 等 |
| 使用 DTO 后缀 | 区分 VO 和 DTO | `ProductSpuRespDTO` |
| 添加 Swagger 注解 | 便于文档生成和调试 | `@Schema(description = "...")` |
| 实现序列化接口 | 支持网络传输 | `implements Serializable`（可选） |

## 第四步：定义 Feign 接口

### 4.1 标准 Feign 接口定义

```java
// yudao-module-mall/yudao-module-product-api/src/main/java/cn/iocoder/yudao/module/product/api/spu/ProductSpuApi.java
package cn.iocoder.yudao.module.product.api.spu;

import cn.iocoder.yudao.framework.common.pojo.CommonResult;
import cn.iocoder.yudao.module.product.api.spu.dto.ProductSpuRespDTO;
import cn.iocoder.yudao.module.product.enums.ApiConstants;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

import java.util.Collection;
import java.util.List;
import java.util.Map;

import static cn.iocoder.yudao.framework.common.util.collection.CollectionUtils.convertMap;

@FeignClient(name = ApiConstants.NAME)
@Tag(name = "RPC 服务 - 商品 SPU")
public interface ProductSpuApi {

    String PREFIX = ApiConstants.PREFIX + "/spu";

    @GetMapping(PREFIX + "/list")
    @Schema(description = "批量查询 SPU 数组")
    @Parameter(name = "ids", description = "SPU 编号列表", required = true, example = "1,3,5")
    CommonResult<List<ProductSpuRespDTO>> getSpuList(@RequestParam("ids") Collection<Long> ids);

    /**
     * 批量查询 SPU MAP
     *
     * @param ids SPU 编号列表
     * @return SPU MAP
     */
    default Map<Long, ProductSpuRespDTO> getSpuMap(Collection<Long> ids) {
        return convertMap(getSpuList(ids).getCheckedData(), ProductSpuRespDTO::getId);
    }

    @GetMapping(PREFIX + "/valid")
    @Schema(description = "批量查询 SPU 数组，并且校验是否 SPU 是否有效")
    @Parameter(name = "ids", description = "SPU 编号列表", required = true, example = "1,3,5")
    CommonResult<List<ProductSpuRespDTO>> validateSpuList(@RequestParam("ids") Collection<Long> ids);

    @GetMapping(PREFIX + "/get")
    @Schema(description = "获得 SPU")
    @Parameter(name = "id", description = "SPU 编号", required = true, example = "1")
    CommonResult<ProductSpuRespDTO> getSpu(@RequestParam("id") Long id);

}
```

### 4.2 Feign 接口解析

**@FeignClient 注解：**

```java
@FeignClient(name = ApiConstants.NAME)
```

- `name`：目标服务名，对应 `spring.application.name`
- Feign 会根据服务名从 Nacos 获取实例列表

**URL 路径定义：**

```java
String PREFIX = ApiConstants.PREFIX + "/spu";
// 最终路径：/rpc-api/product/spu/get
```

**返回值规范：**

| 场景 | 返回类型 | 说明 |
|------|----------|------|
| 查询单个 | `CommonResult<T>` | 包装为统一响应 |
| 查询列表 | `CommonResult<List<T>>` | 包装为统一响应 |
| 校验操作 | `CommonResult<Boolean>` | 返回校验结果 |
| 操作成功 | `CommonResult<Void>` | 无返回数据 |

**default 方法：**

```java
default Map<Long, ProductSpuRespDTO> getSpuMap(Collection<Long> ids) {
    return convertMap(getSpuList(ids).getCheckedData(), ProductSpuRespDTO::getId);
}
```

- 接口中可以定义 `default` 方法，提供便捷的调用方式
- 这些方法内部调用 Feign 方法，不需要远程实现

## 第五步：在调用方使用 Feign 客户端

### 5.1 添加依赖

在 trade-server 的 `pom.xml` 中添加对 product-api 的依赖：

```xml
<!-- yudao-module-mall/yudao-module-trade-server/pom.xml -->
<dependencies>
    <!-- 依赖 product-api，获取 Feign 接口和 DTO -->
    <dependency>
        <groupId>cn.iocoder.cloud</groupId>
        <artifactId>yudao-module-product-api</artifactId>
    </dependency>
</dependencies>
```

### 5.2 注入并使用

```java
// trade-server 中的 OrderService
@Service
public class OrderService {

    @Resource
    private ProductSpuApi productSpuApi;  // 注入 Feign 客户端

    public void createOrder(Long spuId, Integer quantity) {
        // 调用远程服务，像调用本地方法一样
        CommonResult<ProductSpuRespDTO> result = productSpuApi.getSpu(spuId);
        ProductSpuRespDTO spu = result.getCheckedData();

        // 使用商品信息创建订单
        // ...
    }
}
```

### 5.3 调用流程

```
trade-server 调用 productSpuApi.getSpu(1L)
        │
        ▼
Feign 代理（JDK 动态代理）
        │
        ├── 解析 @GetMapping("/rpc-api/product/spu/get")
        ├── 构建 HTTP 请求：GET /rpc-api/product/spu/get?id=1
        ├── 通过 LoadBalancer 选择 product-server 实例
        ├── 发起 HTTP 请求
        └── 将响应 JSON 反序列化为 CommonResult<ProductSpuRespDTO>
        │
        ▼
product-server 处理请求
        │
        ├── @GetMapping("/rpc-api/product/spu/get")
        ├── 查询商品信息
        └── 返回 CommonResult<ProductSpuRespDTO>
        │
        ▼
trade-server 获取结果
```

## 第六步：在实现方提供接口

### 6.1 添加依赖

在 product-server 的 `pom.xml` 中添加对 product-api 的依赖：

```xml
<!-- yudao-module-mall/yudao-module-product-server/pom.xml -->
<dependencies>
    <!-- 依赖 product-api，实现 Feign 接口 -->
    <dependency>
        <groupId>cn.iocoder.cloud</groupId>
        <artifactId>yudao-module-product-api</artifactId>
    </dependency>
</dependencies>
```

### 6.2 实现 Feign 接口

```java
// yudao-module-mall/yudao-module-product-server/src/main/java/cn/iocoder/yudao/module/product/controller/admin/spu/ProductSpuController.java
@RestController
@RequestMapping("/rpc-api/product/spu")
public class ProductSpuController implements ProductSpuApi {

    @Resource
    private ProductService productService;

    @Override
    @GetMapping("/get")
    public CommonResult<ProductSpuRespDTO> getSpu(@RequestParam("id") Long id) {
        ProductSpuRespDTO spu = productService.getSpu(id);
        return CommonResult.success(spu);
    }

    @Override
    @GetMapping("/list")
    public CommonResult<List<ProductSpuRespDTO>> getSpuList(@RequestParam("ids") Collection<Long> ids) {
        List<ProductSpuRespDTO> spus = productService.getSpuList(ids);
        return CommonResult.success(spus);
    }

    @Override
    @GetMapping("/valid")
    public CommonResult<List<ProductSpuRespDTO>> validateSpuList(@RequestParam("ids") Collection<Long> ids) {
        List<ProductSpuRespDTO> spus = productService.validateSpuList(ids);
        return CommonResult.success(spus);
    }
}
```

**实现要点：**

1. Controller 实现 Feign 接口，确保方法签名一致
2. URL 路径与接口定义一致：`/rpc-api/product/spu/get`
3. 返回 `CommonResult` 包装，保持统一响应格式

## 常见问题排查

### 问题一：Feign 调用 404

**现象**：调用远程服务返回 404

**排查步骤**：
1. 检查 URL 路径是否正确
2. 检查 Controller 是否实现了 Feign 接口
3. 检查 `@RequestMapping` 路径是否匹配

### 问题二：序列化错误

**现象**：返回值反序列化失败

**排查步骤**：
1. 检查 DTO 字段类型是否匹配
2. 检查 *-api 模块版本是否一致
3. 检查 Jackson 配置是否正确

### 问题三：找不到服务

**现象**：Feign 调用报错找不到服务

**排查步骤**：
1. 检查 Nacos 控制台，服务是否注册
2. 检查 `ApiConstants.NAME` 是否与 `spring.application.name` 一致
3. 检查服务是否启动

## 本章小结

本章完成了 Feign 客户端和 API 模块的基础配置：

1. **-api 模块设计**：理解了为什么要独立 API 模块，以及目录结构和命名规范
2. **ApiConstants 定义**：服务名和 URL 前缀的统一管理
3. **DTO 设计**：响应 DTO 的设计原则和规范
4. **Feign 接口定义**：标准的 Feign 接口写法和返回值规范
5. **调用方和实现方**：如何注入 Feign 客户端，如何实现 Feign 接口

下一章将介绍 RPC 调用链路的实现，包括请求拦截器、错误解码器等高级特性。
