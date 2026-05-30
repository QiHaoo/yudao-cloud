# 03 - 核心设计：Sentinel 集成

## Sentinel 的核心概念

### 资源（Resource）

资源是 Sentinel 保护的对象，可以是：
- 一个 Java 方法
- 一个 API 接口
- 一段代码块

```java
// 方式一：通过注解定义资源
@SentinelResource(value = "getProduct", fallback = "getProductFallback")
public ProductSpuRespVO getProduct(Long id) {
    return productSpuApi.getSpu(id).getCheckedData();
}

// 方式二：通过 SphU.entry 定义资源
try (Entry entry = SphU.entry("getProduct")) {
    // 业务逻辑
    return productSpuApi.getSpu(id).getCheckedData();
} catch (BlockException ex) {
    // 被限流或熔断
    return getDefaultProduct();
}

// 方式三：自动适配（OpenFeign、Web 等）
// Sentinel 会自动将 @FeignClient 方法和 @RequestMapping 方法识别为资源
```

### 规则（Rule）

Sentinel 支持以下几种规则：

```
1. 流量控制规则（FlowRule）
   ├── 针对资源的 QPS 或并发线程数进行限制
   └── 超过阈值时，执行流控行为

2. 熔断降级规则（DegradeRule）
   ├── 针对资源的异常比例、异常数、慢调用比例进行熔断
   └── 熔断后，拒绝请求，直到恢复

3. 系统保护规则（SystemRule）
   ├── 针对系统的 CPU、负载、入口 QPS 进行保护
   └── 防止系统过载

4. 授权规则（AuthorityRule）
   ├── 针对调用方进行黑白名单控制
   └── 限制哪些调用方可以访问资源

5. 热点参数规则（ParamFlowRule）
   ├── 针对资源的热点参数进行限流
   └── 限制某个参数值的访问频率
```

### 流控效果

```
快速失败（默认）：
  ├── 超过阈值直接拒绝
  ├── 返回 BlockException
  └── 适合：大多数场景

Warm Up：
  ├── 从低阈值开始，逐步增加到阈值
  ├── 避免冷启动时的流量冲击
  └── 适合：秒杀活动开始时

排队等待：
  ├── 超过阈值的请求排队等待
  ├── 匀速通过
  └── 适合：对响应时间不敏感的场景
```

## 与 OpenFeign 的集成：fallback 降级

### 集成方式

在 `pom.xml` 中引入 Sentinel 依赖后，OpenFeign 自动与 Sentinel 集成：

```yaml
spring:
  cloud:
    openfeign:
      sentinel:
        enabled: true
```

### Fallback 降级

```java
// 定义 Feign 接口
@FeignClient(name = "product-server", fallbackFactory = ProductSpuApiFallbackFactory.class)
public interface ProductSpuApi {

    @GetMapping("/rpc-api/product/spu/get")
    CommonResult<ProductSpuRespVO> getSpu(@RequestParam("id") Long id);
}

// 定义降级工厂
@Component
public class ProductSpuApiFallbackFactory implements FallbackFactory<ProductSpuApi> {

    @Override
    public ProductSpuApi create(Throwable cause) {
        return new ProductSpuApi() {
            @Override
            public CommonResult<ProductSpuRespVO> getSpu(Long id) {
                // 记录日志
                log.error("商品服务调用失败，商品ID：{}", id, cause);

                // 返回降级响应
                return CommonResult.error(SERVICE_UNAVAILABLE.getCode(),
                    "商品服务暂时不可用，请稍后重试");
            }
        };
    }
}
```

### 降级流程

```
正常流程：
  trade-server → product-server → 返回商品信息

降级流程（product-server 不可用）：
  trade-server → Sentinel 检测到异常
      │
      ├── 触发熔断规则
      │   ├── 异常比例 > 50%
      │   └── 慢调用比例 > 80%
      │
      ├── 熔断器打开
      │   └── 拒绝所有请求
      │
      └── 调用 FallbackFactory
          └── 返回降级响应
```

## 与 Spring Cloud Gateway 的集成：网关限流

### 集成方式

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-alibaba-sentinel-gateway</artifactId>
</dependency>
```

### 网关限流规则

```yaml
# Sentinel Dashboard 中配置的网关限流规则
gateway:
  flows:
    - resource: system-admin-api          # 路由 ID
      grade: 1                            # QPS 模式
      count: 1000                         # 限流阈值
      strategy: 0                         # 直接拒绝
      controlBehavior: 0                  # 快速失败

    - resource: product-admin-api
      grade: 1
      count: 500
      strategy: 0
      controlBehavior: 0
```

### 网关限流 vs 服务限流

```
网关限流（Gateway）：
  ├── 在请求进入微服务之前就限流
  ├── 保护所有后端服务
  ├── 基于路由 ID、请求参数限流
  └── 适合：全局限流、API 级别限流

服务限流（Sentinel）：
  ├── 在服务内部限流
  ├── 保护单个服务
  ├── 基于方法、资源限流
  └── 适合：服务级别限流、热点参数限流

两层限流的配合：
  Gateway 限流（第一层）：保护整个系统
      │
      └── 服务限流（第二层）：保护单个服务
```

## 限流规则配置：QPS、线程数、热点参数

### QPS 限流

限制资源每秒的请求数量：

```java
// 配置 QPS 限流规则
FlowRule rule = new FlowRule();
rule.setResource("getProduct");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);  // QPS 模式
rule.setCount(1000);                          // 阈值：1000 QPS
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT);  // 快速失败
FlowRuleManager.loadRules(Collections.singletonList(rule));
```

```
QPS 限流效果：

  请求数：1200 QPS
  阈值：1000 QPS
  │
  ├── 1000 个请求 → 正常处理
  └── 200 个请求 → 被拒绝，返回 BlockException
```

### 线程数限流

限制资源的并发线程数：

```java
FlowRule rule = new FlowRule();
rule.setResource("getProduct");
rule.setGrade(RuleConstant.FLOW_GRADE_THREAD);  // 线程数模式
rule.setCount(200);                               // 阈值：200 个线程
```

```
线程数限流效果：

  并发线程数：250
  阈值：200
  │
  ├── 200 个线程 → 正常处理
  └── 50 个请求 → 等待或被拒绝
```

### 热点参数限流

针对资源的热点参数进行限流：

```java
ParamFlowRule rule = new ParamFlowRule("getProduct")
    .setParamIdx(0)                    // 参数索引（第一个参数）
    .setGrade(RuleConstant.FLOW_GRADE_QPS)
    .setCount(100);                    // 总体阈值：100 QPS

// 特殊值配置：商品 ID=1 的阈值是 10
ParamFlowItem item = new ParamFlowItem()
    .setObject(String.valueOf(1L))
    .setClassType(long.class.getName())
    .setCount(10);
rule.setParamFlowItemList(Collections.singletonList(item));
```

```
热点参数限流效果：

  请求：getProduct(1) → 50 QPS
  请求：getProduct(2) → 30 QPS
  请求：getProduct(3) → 20 QPS

  总体阈值：100 QPS
  商品 ID=1 的阈值：10 QPS

  结果：
    getProduct(1)：只允许 10 QPS，其余被拒绝
    getProduct(2)：正常通过
    getProduct(3)：正常通过
```

## 熔断规则配置：慢调用比例、异常比例、异常数

### 慢调用比例

当慢调用比例超过阈值时，触发熔断：

```java
DegradeRule rule = new DegradeRule("getProduct")
    .setGrade(CircuitBreakerStrategy.SLOW_REQUEST_RATIO.getType())
    .setCount(0.5)           // 慢调用比例阈值：50%
    .setSlowRatioThreshold(1000)  // 慢调用阈值：1000ms
    .setTimeWindow(10)       // 熔断时长：10 秒
    .setMinRequestAmount(5)  // 最小请求数：5
    .setStatIntervalMs(10000);  // 统计时长：10 秒
```

```
慢调用比例熔断效果：

  统计时长：10 秒
  最小请求数：5
  慢调用阈值：1000ms
  慢调用比例阈值：50%

  请求情况：
    请求 1：500ms（正常）
    请求 2：1200ms（慢调用）
    请求 3：1500ms（慢调用）
    请求 4：800ms（正常）
    请求 5：2000ms（慢调用）

  慢调用比例：3/5 = 60% > 50%
  结果：触发熔断，熔断时长 10 秒
```

### 异常比例

当异常比例超过阈值时，触发熔断：

```java
DegradeRule rule = new DegradeRule("getProduct")
    .setGrade(CircuitBreakerStrategy.ERROR_RATIO.getType())
    .setCount(0.5)           // 异常比例阈值：50%
    .setTimeWindow(10)       // 熔断时长：10 秒
    .setMinRequestAmount(5)  // 最小请求数：5
    .setStatIntervalMs(10000);  // 统计时长：10 秒
```

### 异常数

当异常数超过阈值时，触发熔断：

```java
DegradeRule rule = new DegradeRule("getProduct")
    .setGrade(CircuitBreakerStrategy.ERROR_COUNT.getType())
    .setCount(10)            // 异常数阈值：10 个
    .setTimeWindow(10)       // 熔断时长：10 秒
    .setMinRequestAmount(5)  // 最小请求数：5
    .setStatIntervalMs(10000);  // 统计时长：10 秒
```

### 熔断器状态机

```
         ┌─────────────────────────────────────────┐
         │                                         │
         ▼                                         │
┌───────────────┐    达到熔断条件    ┌───────────────┐
│   CLOSED      │ ───────────────→ │    OPEN       │
│  （关闭状态）  │                  │  （打开状态）  │
│  正常放行      │                  │  拒绝所有请求  │
└───────────────┘                  └───────┬───────┘
         ▲                                 │
         │                                 │ 熔断时长结束
         │                                 ▼
         │                         ┌───────────────┐
         │  探测成功                │  HALF-OPEN    │
         └──────────────────────── │ （半开状态）   │
                                   │  放行部分请求  │
                                   └───────────────┘
                                         │
                                         │ 探测失败
                                         ▼
                                   回到 OPEN 状态
```

## Dashboard 集成

### Sentinel Dashboard

Sentinel 提供了一个可视化管理界面，用于：
- 实时监控资源的 QPS、响应时间、异常比例
- 动态配置限流规则、熔断规则
- 查看集群拓扑

### 集成方式

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080  # Dashboard 地址
        port: 8719                 # 与 Dashboard 通信的端口
```

### Dashboard 功能

```
实时监控：
  ├── 资源的 QPS 曲线
  ├── 响应时间分布
  ├── 异常比例
  └── 并发线程数

规则管理：
  ├── 流控规则
  ├── 熔断规则
  ├── 系统规则
  ├── 授权规则
  └── 热点规则

集群拓扑：
  ├── 调用链路
  ├── 节点状态
  └── 实时流量
```

## 思考题

1. **Sentinel 的三种熔断策略（慢调用比例、异常比例、异常数）分别适用于什么场景？在电商系统中，你会如何选择？**

2. **Sentinel 的熔断器有三种状态（CLOSED、OPEN、HALF-OPEN）。如果半开状态下的探测请求一直失败，熔断器会如何处理？**

3. **热点参数限流可以针对特定参数值设置不同阈值。在电商场景中，有哪些"热点"需要特别保护？**

4. **Sentinel Dashboard 支持动态修改规则，这很方便，但也带来安全风险。如何保护 Dashboard 不被未授权访问？**
