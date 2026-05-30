# 01 - 起步：Sentinel 集成配置

> 本章是限流熔断功能实现的起点。完成本章后，你的微服务将具备 Sentinel 限流和熔断能力，并能与 OpenFeign 集成实现服务降级。

## 前置准备

在开始之前，请确保：

1. 已完成第四阶段组件 16（OpenFeign）的功能实现
2. 已阅读组件 18 的功能设计文档
3. 了解 Sentinel 的核心概念（资源、规则、熔断器状态机）

## 第一步：添加 Sentinel 依赖

在 `yudao-module-system/yudao-module-system-server/pom.xml` 中添加 Sentinel 依赖：

```xml
<!-- yudao-module-system/yudao-module-system-server/pom.xml -->
<dependencies>
    <!-- Sentinel 限流熔断 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>

    <!-- Sentinel 动态规则（Nacos 数据源） -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-alibaba-sentinel-datasource-nacos</artifactId>
    </dependency>
</dependencies>
```

**依赖说明：**

| 依赖 | 职责 |
|------|------|
| `spring-cloud-starter-alibaba-sentinel` | Sentinel 核心，提供限流、熔断、降级能力 |
| `sentinel-datasource-nacos` | 从 Nacos 动态拉取规则，支持实时更新 |

**版本管理：**

Sentinel 的版本由 `spring-cloud-alibaba-dependencies` BOM 统一管理，不需要手动指定版本号：

```xml
<!-- yudao-dependencies/pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>${spring.cloud.alibaba.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 第二步：基础配置

在 `application.yaml` 中添加 Sentinel 配置：

```yaml
# yudao-module-system/yudao-module-system-server/src/main/resources/application.yaml
spring:
  cloud:
    sentinel:
      # Sentinel Dashboard 配置
      transport:
        dashboard: localhost:8080  # Dashboard 地址
        port: 8719                 # 与 Dashboard 通信的端口

      # 从 Nacos 动态拉取规则
      datasource:
        # 流控规则
        flow:
          nacos:
            server-addr: ${spring.cloud.nacos.server-addr}
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: flow

        # 降级规则
        degrade:
          nacos:
            server-addr: ${spring.cloud.nacos.server-addr}
            data-id: ${spring.application.name}-degrade-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: degrade

      # Sentinel 日志目录
      log:
        dir: ${user.home}/logs/sentinel
```

**配置项说明：**

| 配置项 | 说明 |
|--------|------|
| `transport.dashboard` | Sentinel Dashboard 地址，用于实时监控和规则管理 |
| `transport.port` | 客户端与 Dashboard 通信的端口，默认 8719 |
| `datasource.flow` | 流控规则的数据源 |
| `datasource.degrade` | 降级规则的数据源 |
| `rule-type: flow` | 流控规则类型 |
| `rule-type: degrade` | 降级规则类型 |
| `log.dir` | Sentinel 日志目录 |

## 第三步：启用 OpenFeign + Sentinel 集成

在 `application.yaml` 中启用 OpenFeign 的 Sentinel 支持：

```yaml
spring:
  cloud:
    openfeign:
      sentinel:
        enabled: true  # 启用 OpenFeign + Sentinel 集成
```

**启用后的效果：**

```
未启用 Sentinel：
  trade-server → product-server → 调用失败 → 抛出异常

启用 Sentinel：
  trade-server → Sentinel 检查 → product-server
                     │
                     ├── 正常 → 调用成功
                     ├── 限流 → 调用 FallbackFactory，返回降级响应
                     └── 熔断 → 调用 FallbackFactory，返回降级响应
```

## 第四步：定义 FallbackFactory

为每个 Feign 客户端定义降级工厂，处理服务不可用的情况。

### 4.1 ProductSpuApi 降级工厂

```java
// yudao-module-trade/yudao-module-trade-server/src/main/java/cn/iocoder/yudao/module/trade/dal/rpc/product/ProductSpuApiFallbackFactory.java
package cn.iocoder.yudao.module.trade.dal.rpc.product;

import cn.iocoder.yudao.framework.common.pojo.CommonResult;
import cn.iocoder.yudao.module.product.api.spu.ProductSpuApi;
import cn.iocoder.yudao.module.product.api.spu.vo.ProductSpuRespVO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.openfeign.FallbackFactory;
import org.springframework.stereotype.Component;

import static cn.iocoder.yudao.framework.common.exception.enums.GlobalErrorCodeConstants.SERVICE_UNAVAILABLE;

/**
 * ProductSpuApi 降级工厂
 */
@Slf4j
@Component
public class ProductSpuApiFallbackFactory implements FallbackFactory<ProductSpuApi> {

    @Override
    public ProductSpuApi create(Throwable cause) {
        return new ProductSpuApi() {
            @Override
            public CommonResult<ProductSpuRespVO> getSpu(Long id) {
                log.error("[getSpu][商品服务调用失败，商品ID：{}]", id, cause);
                return CommonResult.error(SERVICE_UNAVAILABLE.getCode(),
                        "商品服务暂时不可用，请稍后重试");
            }

            @Override
            public CommonResult<Void> deductStock(Long id, Integer count) {
                log.error("[deductStock][商品服务调用失败，商品ID：{}，数量：{}]", id, count, cause);
                return CommonResult.error(SERVICE_UNAVAILABLE.getCode(),
                        "商品服务暂时不可用，请稍后重试");
            }
        };
    }

}
```

### 4.2 Feign 客户端指定降级工厂

在 `@FeignClient` 注解中指定 `fallbackFactory`：

```java
// yudao-module-trade/yudao-module-trade-server/src/main/java/cn/iocoder/yudao/module/trade/dal/rpc/product/ProductSpuApi.java
package cn.iocoder.yudao.module.trade.dal.rpc.product;

import cn.iocoder.yudao.framework.common.pojo.CommonResult;
import cn.iocoder.yudao.module.product.api.spu.ProductSpuApi;
import cn.iocoder.yudao.module.product.api.spu.vo.ProductSpuRespVO;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.*;

@FeignClient(name = "product-server",
        fallbackFactory = ProductSpuApiFallbackFactory.class)
public interface ProductSpuApi extends cn.iocoder.yudao.module.product.api.spu.ProductSpuApi {

    // 继承 product-api 中定义的接口方法

}
```

**为什么使用 `fallbackFactory` 而非 `fallback`？**

| 方式 | 优点 | 缺点 |
|------|------|------|
| `fallback` | 简单 | 无法获取异常信息 |
| `fallbackFactory` | 可以获取异常信息，便于日志记录 | 稍微复杂 |

生产环境推荐使用 `fallbackFactory`，因为它可以记录异常信息，便于排查问题。

### 4.3 降级工厂的典型模式

```java
@Slf4j
@Component
public class XxxApiFallbackFactory implements FallbackFactory<XxxApi> {

    @Override
    public XxxApi create(Throwable cause) {
        return new XxxApi() {
            @Override
            public CommonResult<XxxRespVO> getXxx(Long id) {
                // 1. 记录日志（包含异常信息）
                log.error("[getXxx][服务调用失败，ID：{}]", id, cause);

                // 2. 返回降级响应
                return CommonResult.error(SERVICE_UNAVAILABLE.getCode(),
                        "服务暂时不可用，请稍后重试");

                // 或者返回默认值
                // return CommonResult.success(getDefaultXxx(id));

                // 或者返回缓存数据
                // return CommonResult.success(getFromCache(id));
            }
        };
    }

}
```

## 第五步：Sentinel Dashboard

### 5.1 启动 Dashboard

```bash
# 下载 sentinel-dashboard.jar
# https://github.com/alibaba/Sentinel/releases

# 启动
java -jar sentinel-dashboard.jar

# 访问 http://localhost:8080
# 默认账号密码：sentinel/sentinel
```

### 5.2 配置流控规则

在 Dashboard 中为 system-server 配置流控规则：

```
资源名：getProduct（对应 @SentinelResource 的 value）
阈值类型：QPS
单机阈值：1000
流控模式：直接
流控效果：快速失败
```

### 5.3 配置降级规则

在 Dashboard 中配置降级规则：

```
资源名：getProduct
熔断策略：慢调用比例
比例阈值：0.5（50%）
慢调用阈值：1000ms
熔断时长：10秒
最小请求数：5
统计时长：10000ms
```

### 5.4 从 Nacos 动态推送规则

在 Nacos 中创建流控规则配置：

**Data ID**: `system-server-flow-rules`
**Group**: `SENTINEL_GROUP`
**格式**: JSON

```json
[
  {
    "resource": "getProduct",
    "grade": 1,
    "count": 1000,
    "strategy": 0,
    "controlBehavior": 0
  },
  {
    "resource": "createOrder",
    "grade": 1,
    "count": 500,
    "strategy": 0,
    "controlBehavior": 0
  }
]
```

**Data ID**: `system-server-degrade-rules`
**Group**: `SENTINEL_GROUP`
**格式**: JSON

```json
[
  {
    "resource": "getProduct",
    "grade": 0,
    "count": 0.5,
    "slowRatioThreshold": 1000,
    "timeWindow": 10,
    "minRequestAmount": 5,
    "statIntervalMs": 10000
  }
]
```

## 第六步：使用 @SentinelResource 注解

对于需要精细控制的资源，可以使用 `@SentinelResource` 注解：

```java
@Service
public class TradeOrderServiceImpl implements TradeOrderService {

    @Resource
    private ProductSpuApi productSpuApi;

    @Override
    @SentinelResource(value = "createOrder",
            fallback = "createOrderFallback",
            fallbackClass = TradeOrderServiceFallback.class)
    public Long createOrder(TradeOrderCreateReqVO reqVO) {
        // 业务逻辑
        ProductSpuRespVO spu = productSpuApi.getSpu(reqVO.getSpuId()).getCheckedData();
        // ...
    }

}

// 降级方法类
public class TradeOrderServiceFallback {

    public static Long createOrderFallback(TradeOrderCreateReqVO reqVO, Throwable cause) {
        log.error("[createOrder][订单创建降级，reqVO：{}]", reqVO, cause);
        throw new ServiceException(SERVICE_UNAVAILABLE, "订单服务暂时不可用");
    }

}
```

**`@SentinelResource` 属性说明：**

| 属性 | 说明 |
|------|------|
| `value` | 资源名，用于匹配规则 |
| `fallback` | 降级方法名（参数和返回值必须一致） |
| `fallbackClass` | 降级方法所在的类（静态方法） |
| `blockHandler` | 限流/熔断处理方法（参数必须包含 BlockException） |
| `exceptionsToIgnore` | 忽略的异常类型 |

## 第七步：验证集成

### 7.1 启动服务

确保以下服务已启动：
1. Nacos Server
2. Sentinel Dashboard（可选）
3. system-server
4. trade-server

### 7.2 测试正常调用

```bash
# trade-server 调用 product-server
curl http://localhost:48083/admin-api/trade/order/create \
  -H "Content-Type: application/json" \
  -d '{"spuId":1,"quantity":1}'

# 预期：正常创建订单
```

### 7.3 测试降级

```bash
# 停止 product-server，模拟服务不可用

# 再次调用
curl http://localhost:48083/admin-api/trade/order/create \
  -H "Content-Type: application/json" \
  -d '{"spuId":1,"quantity":1}'

# 预期：返回降级响应 "商品服务暂时不可用，请稍后重试"
```

### 7.4 测试限流

```bash
# 使用压测工具发送大量请求
# 超过阈值后，请求会被拒绝

# 预期：返回 BlockException 相关的错误信息
```

## 常见问题排查

### 问题一：Sentinel Dashboard 看不到资源

**现象**：Dashboard 中没有显示应用的资源

**排查步骤**：
1. 检查 `spring.cloud.sentinel.transport.dashboard` 配置是否正确
2. 检查 Dashboard 是否启动
3. 检查网络连通性
4. 检查应用是否正确启动（Sentinel 是懒加载，需要有请求才会注册资源）

### 问题二：限流不生效

**现象**：发送大量请求，但没有被限流

**排查步骤**：
1. 检查资源名是否匹配（`@SentinelResource` 的 value 与规则的 resource）
2. 检查规则是否正确配置
3. 检查规则是否已推送到应用（Dashboard 或 Nacos）

### 问题三：降级方法不被调用

**现象**：服务调用失败，但没有调用 FallbackFactory

**排查步骤**：
1. 检查 `spring.cloud.openfeign.sentinel.enabled` 是否为 true
2. 检查 `@FeignClient` 是否指定了 `fallbackFactory`
3. 检查 FallbackFactory 类是否被 Spring 扫描到（`@Component`）

## 本章小结

本章完成了 Sentinel 的基础集成：

1. **依赖引入**：添加 Sentinel 和 Nacos 数据源依赖
2. **基础配置**：配置 Dashboard 地址、数据源
3. **OpenFeign 集成**：启用 Sentinel 支持，定义 FallbackFactory
4. **Dashboard 使用**：配置流控规则和降级规则
5. **注解使用**：通过 `@SentinelResource` 精细控制资源

下一章将介绍 Seata 分布式事务的集成。
