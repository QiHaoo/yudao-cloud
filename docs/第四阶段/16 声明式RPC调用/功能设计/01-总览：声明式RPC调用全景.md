# 01 - 总览：声明式 RPC 调用全景

## 场景引入

假设你的电商系统已经拆分为微服务架构，现在 trade-server（交易服务）需要在创建订单时调用 product-server（商品服务）查询商品信息、调用 system-server（系统服务）查询用户信息。

**传统方式：使用 RestTemplate**

```java
// trade-server 中调用 product-server
String url = "http://product-server/admin-api/product/spu/get?id=" + spuId;
ProductSpuRespVO spu = restTemplate.getForObject(url, ProductSpuRespVO.class);

// trade-server 中调用 system-server
String url2 = "http://system-server/admin-api/system/user/get?id=" + userId;
AdminUserRespDTO user = restTemplate.getForObject(url2, AdminUserRespDTO.class);
```

这种方式存在以下痛点：

```
痛点一：URL 硬编码
  服务名写死在代码中，如果服务名变更，需要修改所有调用方

痛点二：序列化/反序列化手动处理
  手动拼接 URL 参数，手动解析响应 JSON

痛点三：无类型检查
  编译器无法检查参数类型是否正确，运行时才发现错误

痛点四：错误处理分散
  每个调用点都需要单独处理异常

痛点五：无统一的拦截机制
  租户信息传递、日志记录等横切关注点需要在每个调用点重复编码
```

## OpenFeign 的声明式 HTTP 客户端

OpenFeign 的核心思想是：**用接口定义代替代码实现**。你只需要声明"我要调用什么方法"，而不需要关心"如何发起 HTTP 请求"。

```
传统方式（命令式）：
  开发者告诉程序"怎么做"：
  1. 拼接 URL
  2. 设置请求参数
  3. 发起 HTTP 请求
  4. 解析响应 JSON
  5. 转换为 Java 对象

OpenFeign 方式（声明式）：
  开发者告诉程序"做什么"：
  @FeignClient(name = "product-server")
  public interface ProductSpuApi {
      @GetMapping("/get")
      CommonResult<ProductSpuRespVO> getSpu(@RequestParam("id") Long id);
  }
  // 使用时直接调用接口方法
  productSpuApi.getSpu(spuId);
```

OpenFeign 的工作原理：

```
┌─────────────────────────────────────────────────────────────────┐
│                        调用方代码                                │
│                                                                 │
│  @Resource                                                      │
│  private ProductSpuApi productSpuApi;                           │
│                                                                 │
│  CommonResult<ProductSpuRespVO> result = productSpuApi          │
│      .getSpu(1L);                                               │
└────────────────────────────────┬────────────────────────────────┘
                                 │ 方法调用
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Feign 代理对象（JDK 动态代理）                 │
│                                                                 │
│  1. 解析方法上的 @GetMapping("/get") 注解                       │
│  2. 解析参数上的 @RequestParam("id") 注解                       │
│  3. 构建 HTTP 请求：GET /rpc-api/product/spu/get?id=1           │
│  4. 通过 LoadBalancer 选择目标实例                               │
│  5. 发起 HTTP 请求                                              │
│  6. 将响应 JSON 反序列化为 CommonResult<ProductSpuRespVO>        │
└────────────────────────────────┬────────────────────────────────┘
                                 │ HTTP 请求
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    product-server                                │
│                                                                 │
│  @GetMapping("/rpc-api/product/spu/get")                        │
│  public CommonResult<ProductSpuRespVO> getSpu(Long id) {        │
│      // 业务逻辑                                                │
│  }                                                              │
└─────────────────────────────────────────────────────────────────┘
```

## *-api 模块设计：接口定义 + DTO 共享

### 为什么需要 *-api 模块

在微服务架构中，服务间的调用需要遵循一个原则：**调用方和被调用方共享同一份接口契约**。

```
没有 *-api 模块的问题：

trade-server                          product-server
    │                                      │
    │  // 调用方自己定义 DTO               │
    │  class ProductSpuDTO {               │
    │      Long id;                        │
    │      String name;                    │
    │      Integer price;                  │
    │  }                                   │
    │                                      │
    │  // 被调用方也定义了 DTO             │  class ProductSpuRespVO {
    │                                      │      Long spuId;      ← 字段名不同！
    │                                      │      String spuName;  ← 字段名不同！
    │                                      │      Integer price;
    │                                      │  }
    │                                      │
    │  调用时字段映射可能出错！             │
```

*-api 模块的设计意图：

```
yudao-module-product-api（共享模块）
├── ProductSpuApi.java              ← Feign 接口定义
└── dto/
    ├── ProductSpuRespVO.java       ← 响应 DTO
    └── ProductSpuSaveReqVO.java    ← 请求 DTO

trade-server 依赖 yudao-module-product-api
product-server 依赖 yudao-module-product-api

两个服务共享同一份接口契约，编译时就能发现类型错误！
```

### yudao-cloud 中的 *-api 模块

yudao-cloud 中每个业务模块都拆分为 `*-api` 和 `*-server` 两个子模块：

```
yudao-module-system/
├── yudao-module-system-api/        ← API 模块（接口 + DTO）
│   └── src/main/java/
│       └── cn/iocoder/yudao/module/system/
│           ├── api/                ← Feign 接口
│           │   ├── user/AdminUserApi.java
│           │   ├── dept/DeptApi.java
│           │   ├── permission/PermissionApi.java
│           │   └── ...
│           └── enums/              ← 枚举常量
│               ├── ApiConstants.java
│               └── ErrorCodeConstants.java
│
└── yudao-module-system-server/     ← 实现模块（业务逻辑）
    └── src/main/java/
        └── cn/iocoder/yudao/module/system/
            ├── controller/admin/   ← Controller 实现
            ├── service/            ← Service 层
            └── dal/                ← 数据访问层
```

## 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        trade-server                             │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Service 层                             │  │
│  │                                                          │  │
│  │  @Resource                                               │  │
│  │  private ProductSpuApi productSpuApi;                    │  │
│  │                                                          │  │
│  │  public void createOrder(Long spuId) {                   │  │
│  │      CommonResult<ProductSpuRespVO> result               │  │
│  │          = productSpuApi.getSpu(spuId);                  │  │
│  │  }                                                       │  │
│  └──────────────────────────┬───────────────────────────────┘  │
│                              │                                  │
└──────────────────────────────┼──────────────────────────────────┘
                               │ 方法调用
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Feign 代理（JDK 动态代理）                     │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ 请求构建器   │  │ 请求拦截器   │  │ 错误解码器   │          │
│  │              │  │              │  │              │          │
│  │ 解析注解     │  │ 添加租户ID   │  │ 翻译异常     │          │
│  │ 拼接参数     │  │ 添加 Token   │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└────────────────────────────────┬────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                   LoadBalancer 负载均衡                          │
│                                                                 │
│  从 Nacos 获取 product-server 实例列表                          │
│  选择一个实例（轮询/随机/权重）                                  │
│                                                                 │
│  选中：192.168.1.20:48082                                       │
└────────────────────────────────┬────────────────────────────────┘
                                 │ HTTP 请求
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                   product-server                                │
│                                                                 │
│  @GetMapping("/rpc-api/product/spu/get")                        │
│  public CommonResult<ProductSpuRespVO> getSpu(Long id) {        │
│      ProductSpuRespVO spu = productService.getSpu(id);          │
│      return CommonResult.success(spu);                          │
│  }                                                              │
└─────────────────────────────────────────────────────────────────┘
```

## 与 Dubbo/gRPC 的对比

| 维度 | OpenFeign | Dubbo | gRPC |
|------|-----------|-------|------|
| **协议** | HTTP/REST | 自定义 TCP 协议 | HTTP/2 + Protobuf |
| **序列化** | JSON | Hessian2/JSON | Protobuf |
| **通信模型** | 请求-响应 | 请求-响应/异步/流式 | 请求-响应/流式 |
| **服务治理** | 依赖 Spring Cloud | 内置丰富 | 需要额外集成 |
| **学习成本** | 低（基于 HTTP） | 中 | 中高（需要学习 Protobuf） |
| **性能** | 中等 | 高 | 高 |
| **跨语言** | 好（基于 HTTP） | 差（Java 生态） | 好（多语言支持） |
| **调试便利性** | 好（可直接 curl） | 差（需要专用工具） | 差（需要专用工具） |
| **Spring Cloud 集成** | 原生支持 | 需要额外适配 | 需要额外适配 |

yudao-cloud 选择 OpenFeign 的理由：

1. **与 Spring Cloud 生态完美契合**：OpenFeign 是 Spring Cloud 的一等公民
2. **HTTP 协议通用性好**：前端、移动端、第三方系统都能直接调用
3. **调试方便**：可以用 curl、Postman 等工具直接测试
4. **学习成本低**：基于注解的方式，开发者熟悉 Spring MVC 就能上手
5. **与 Gateway 无缝集成**：网关可以直接转发 HTTP 请求

## 思考题

1. **OpenFeign 的核心是"声明式"编程。除了 Feign，你能举出 Java 生态中其他"声明式"编程的例子吗？它们有什么共通之处？**

2. **为什么 yudao-cloud 要把 API 模块和实现模块分开？如果不分开，会有什么问题？**

3. **OpenFeign 基于 HTTP/JSON 通信，性能不如 Dubbo 的二进制协议。在什么场景下这个性能差距是不可接受的？yudao-cloud 的场景是否需要考虑这个问题？**

4. **Feign 代理是通过 JDK 动态代理实现的。如果一个接口有多个方法，Feign 是为每个方法创建一个代理，还是为整个接口创建一个代理？**
