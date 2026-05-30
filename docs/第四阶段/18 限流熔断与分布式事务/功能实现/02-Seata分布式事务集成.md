# 02 - Seata 分布式事务集成

> 本章实现 Seata 分布式事务的集成。完成本章后，你的微服务将具备跨服务事务的一致性保证能力，使用 `@GlobalTransactional` 注解即可开启全局事务。

## 前置准备

在开始之前，请确保：

1. 已完成第四阶段组件 16（OpenFeign）的功能实现
2. 已阅读组件 18 的功能设计文档，了解 Seata AT 模式的原理
3. 本地已安装或能启动 Seata Server

## 第一步：理解 Seata 的架构

```
Seata 的三个角色：

TM（Transaction Manager，事务管理器）：
  ├── 开启、提交、回滚全局事务
  └── 通常是发起方（如 trade-server）

RM（Resource Manager，资源管理器）：
  ├── 管理分支事务的资源
  ├── 向 TC 注册分支事务
  └── 执行提交或回滚

TC（Transaction Coordinator，事务协调器）：
  ├── 维护全局事务和分支事务的状态
  ├── 协调提交或回滚
  └── Seata Server 提供
```

```
AT 模式的工作流程：

1. TM 开启全局事务 → TC 生成 XID
2. TM 执行业务 SQL → RM 拦截 SQL，生成 undo_log
3. RM 注册分支事务 → TC 记录分支状态
4. RM 提交本地事务（业务 SQL + undo_log）
5. 全局提交 → TC 通知所有 RM 删除 undo_log
6. 全局回滚 → TC 通知所有 RM 根据 undo_log 反向补偿
```

## 第二步：准备 undo_log 表

Seata AT 模式需要在每个业务数据库中创建 `undo_log` 表：

```sql
-- 在每个业务数据库中执行（如 yudao_cloud_system、yudao_cloud_trade 等）
CREATE TABLE `undo_log` (
    `id` bigint(20) NOT NULL AUTO_INCREMENT,
    `branch_id` bigint(20) NOT NULL,
    `xid` varchar(100) NOT NULL,
    `context` varchar(128) NOT NULL,
    `rollback_info` longblob NOT NULL,
    `log_status` int(11) NOT NULL,
    `log_created` datetime NOT NULL,
    `log_modified` datetime NOT NULL,
    `ext` varchar(100) DEFAULT NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

**为什么每个数据库都需要 undo_log 表？**

AT 模式通过拦截 SQL 生成 undo_log，用于回滚时的反向补偿。每个数据库的事务是独立的，所以每个数据库都需要自己的 undo_log 表来记录该数据库的操作。

```
示例：

trade-server 操作 trade 数据库：
  INSERT INTO orders ... → 生成 undo_log（DELETE FROM orders WHERE id=xxx）
  → 存储在 trade 数据库的 undo_log 表

product-server 操作 product 数据库：
  UPDATE stock SET quantity = 9 ... → 生成 undo_log（UPDATE stock SET quantity=10 ...）
  → 存储在 product 数据库的 undo_log 表
```

## 第三步：添加 Seata 依赖

在业务服务的 `pom.xml` 中添加 Seata 依赖：

```xml
<!-- yudao-module-trade/yudao-module-trade-server/pom.xml -->
<dependencies>
    <!-- Seata 分布式事务 -->
    <dependency>
        <groupId>io.seata</groupId>
        <artifactId>seata-spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

**版本管理：**

Seata 的版本在 `yudao-dependencies/pom.xml` 中统一管理：

```xml
<!-- yudao-dependencies/pom.xml -->
<properties>
    <seata.version>1.7.1</seata.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.seata</groupId>
            <artifactId>seata-spring-boot-starter</artifactId>
            <version>${seata.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 第四步：配置 Seata

在 `application.yaml` 中添加 Seata 配置：

```yaml
# yudao-module-trade/yudao-module-trade-server/src/main/resources/application.yaml
seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: yudao-cloud-tx-group  # 事务分组

  service:
    vgroup-mapping:
      yudao-cloud-tx-group: default  # 事务分组映射到 Seata Server 集群

  registry:
    type: nacos  # 使用 Nacos 注册中心
    nacos:
      server-addr: ${spring.cloud.nacos.server-addr}
      namespace: ${spring.cloud.nacos.discovery.namespace}
      group: DEFAULT_GROUP
      application: seata-server  # Seata Server 在 Nacos 中的服务名

  config:
    type: nacos  # 使用 Nacos 配置中心
    nacos:
      server-addr: ${spring.cloud.nacos.server-addr}
      namespace: ${spring.cloud.nacos.config.namespace}
      group: DEFAULT_GROUP
      data-id: seataServer.properties
```

**配置项说明：**

| 配置项 | 说明 |
|--------|------|
| `enabled` | 是否启用 Seata |
| `application-id` | 应用标识，通常是服务名 |
| `tx-service-group` | 事务分组，用于隔离不同环境的事务 |
| `service.vgroup-mapping` | 事务分组到 Seata Server 集群的映射 |
| `registry.type` | 注册中心类型，使用 Nacos |
| `config.type` | 配置中心类型，使用 Nacos |

## 第五步：配置数据源代理

Seata AT 模式需要代理数据源，自动拦截 SQL 生成 undo_log。

### 5.1 为什么需要数据源代理？

```
正常数据源：
  应用 → DataSource → 数据库

Seata 代理数据源：
  应用 → DataSourceProxy → DataSource → 数据库
                │
                ├── 拦截 SQL
                ├── 生成 before image
                ├── 执行业务 SQL
                ├── 生成 after image
                ├── 生成 undo_log
                └── 一起提交（业务 SQL + undo_log）
```

### 5.2 实现方式

在 Spring Boot 中，Seata 会自动代理数据源，无需手动配置。但需要确保数据源配置正确：

```java
// 方式一：Spring Boot 自动配置（推荐）
// Seata 会自动检测 DataSource Bean 并代理

// 方式二：手动配置（特殊场景）
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource dataSource(DataSourceProperties properties) {
        DataSource dataSource = properties.initializeDataSourceBuilder().build();
        return new DataSourceProxy(dataSource);
    }

}
```

**Spring Boot 自动配置的原理：**

`seata-spring-boot-starter` 中包含 `SeataAutoConfiguration`，它会：
1. 检测到 `DataSource` Bean
2. 自动包装为 `DataSourceProxy`
3. 注册到 Seata 的 RM（资源管理器）

## 第六步：使用 @GlobalTransactional

### 6.1 基本用法

在发起方的方法上添加 `@GlobalTransactional` 注解：

```java
// yudao-module-trade/yudao-module-trade-server/src/main/java/cn/iocoder/yudao/module/trade/service/order/TradeOrderServiceImpl.java
@Service
public class TradeOrderServiceImpl implements TradeOrderService {

    @Resource
    private ProductSpuApi productSpuApi;

    @Resource
    private MemberPointApi memberPointApi;

    @Override
    @GlobalTransactional  // 开启全局事务
    public Long createOrder(TradeOrderCreateReqVO reqVO) {
        // 1. 创建订单（本地事务）
        TradeOrderDO order = new TradeOrderDO();
        order.setSpuId(reqVO.getSpuId());
        order.setUserId(reqVO.getUserId());
        tradeOrderMapper.insert(order);

        // 2. 扣减库存（远程调用，分支事务）
        productSpuApi.deductStock(reqVO.getSpuId(), 1);

        // 3. 扣减积分（远程调用，分支事务）
        memberPointApi.deductPoints(reqVO.getUserId(), 100);

        // 如果步骤 3 失败，步骤 1 和 2 会自动回滚
        return order.getId();
    }

}
```

**执行流程：**

```
trade-server (TM)                     Seata Server (TC)
    │                                      │
    │  1. @GlobalTransactional             │
    │  ─────────────────────────────────→ │
    │                                      │  生成 XID = 192.168.1.10:8091:12345
    │ <───────────────────────────────── │
    │                                      │
    │  2. INSERT INTO orders ...           │
    │     ├── 生成 before image            │
    │     ├── 执行 SQL                     │
    │     ├── 生成 undo_log                │
    │     └── 提交本地事务                 │
    │                                      │
    │  3. 调用 product-server              │
    │     ├── 请求头携带 XID               │
    │     └── 扣减库存（分支事务）          │
    │                                      │
    │  4. 调用 member-server               │
    │     ├── 请求头携带 XID               │
    │     └── 扣减积分（分支事务）          │
    │                                      │
    │  5. 全部成功 → 全局提交              │
    │  ─────────────────────────────────→ │
    │                                      │  通知所有分支删除 undo_log
```

### 6.2 注解属性

```java
@GlobalTransactional(
    timeoutMills = 60000,           // 全局事务超时时间（毫秒），默认 60000
    name = "createOrder",           // 全局事务名称，用于监控
    rollbackFor = Exception.class,  // 回滚异常类型，默认所有异常
    noRollbackFor = BusinessException.class  // 不回滚的异常类型
)
```

| 属性 | 说明 | 默认值 |
|------|------|--------|
| `timeoutMills` | 全局事务超时时间 | 60000ms（1 分钟） |
| `name` | 全局事务名称 | 方法名 |
| `rollbackFor` | 触发回滚的异常类型 | Exception.class |
| `noRollbackFor` | 不触发回滚的异常类型 | 空 |

### 6.3 XID 传播机制

Seata 通过 XID（全局事务 ID）关联分支事务。XID 通过请求头自动传播：

```
trade-server 开启全局事务
    │
    ├── 生成 XID = 192.168.1.10:8091:12345
    │
    ├── 调用 product-server（OpenFeign）
    │   │
    │   ├── Seata 拦截器自动添加请求头
    │   │   Header: TX_XID = 192.168.1.10:8091:12345
    │   │
    │   └── product-server 的 Seata 拦截器自动解析 XID
    │       └── 注册分支事务，关联到 XID
    │
    └── 调用 member-server
        │
        ├── 请求头携带 XID
        │   Header: TX_XID = 192.168.1.10:8091:12345
        │
        └── member-server 解析 XID
            └── 注册分支事务，关联到 XID
```

**XID 传播的实现：**

Seata 提供了 `SeataFeignRequestInterceptor`，自动在 OpenFeign 请求中添加 XID 头。如果使用 RestTemplate，需要手动传播：

```java
// 手动传播 XID（RestTemplate 场景）
String xid = RootContext.getXID();
if (xid != null) {
    headers.set(RootContext.KEY_XID, xid);
}
```

## 第七步：Seata Server 部署

### 7.1 下载 Seata Server

```bash
# 下载地址
# https://github.com/seata/seata/releases

# 解压
tar -xzf seata-server-1.7.1.tar.gz
cd seata-server
```

### 7.2 配置 Seata Server

编辑 `conf/registry.conf`：

```conf
registry {
  type = "nacos"
  nacos {
    application = "seata-server"
    serverAddr = "127.0.0.1:8848"
    namespace = "dev"
    group = "DEFAULT_GROUP"
  }
}

config {
  type = "nacos"
  nacos {
    serverAddr = "127.0.0.1:8848"
    namespace = "dev"
    group = "DEFAULT_GROUP"
    dataId = "seataServer.properties"
  }
}
```

### 7.3 启动 Seata Server

```bash
# 单机模式启动
bin/seata-server.sh -p 8091

# 或者 Windows
bin/seata-server.bat -p 8091
```

### 7.4 在 Nacos 中配置 Seata Server

在 Nacos 中创建配置：

**Data ID**: `seataServer.properties`
**Group**: `DEFAULT_GROUP`

```properties
# 事务日志存储模式
store.mode=db
store.db.datasource=druid
store.db.dbType=mysql
store.db.driverClassName=com.mysql.cj.jdbc.Driver
store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true&rewriteBatchedStatements=true
store.db.user=root
store.db.password=root
```

## 第八步：验证分布式事务

### 8.1 启动服务

确保以下服务已启动：
1. Nacos Server
2. Seata Server
3. product-server
4. member-server
5. trade-server

### 8.2 测试正常下单

```bash
# 创建订单
curl -X POST http://localhost:48080/admin-api/trade/order/create \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {token}" \
  -d '{"spuId":1,"quantity":1}'

# 预期：订单创建成功，库存扣减，积分扣减
```

### 8.3 测试事务回滚

```bash
# 模拟扣减积分失败（如积分不足）

# 预期：
# 1. trade-server 的订单创建操作回滚
# 2. product-server 的库存扣减操作回滚
# 3. 返回错误信息
```

### 8.4 验证 undo_log

```sql
-- 在事务执行过程中，可以查看 undo_log
SELECT * FROM undo_log ORDER BY log_created DESC LIMIT 10;

-- 事务提交后，undo_log 应该被删除
```

## 第九步：注意事项

### 9.1 @GlobalTransactional 只能在发起方使用

```java
// 正确：trade-server 的 createOrder 方法（发起方）
@Override
@GlobalTransactional
public Long createOrder(TradeOrderCreateReqVO reqVO) {
    // ...
}

// 错误：product-server 的 deductStock 方法（参与方）
// 不需要加 @GlobalTransactional，Seata 会自动处理
@Override
public void deductStock(Long spuId, Integer quantity) {
    // ...
}
```

### 9.2 避免长事务

```java
// 不好的设计：事务中包含耗时操作
@GlobalTransactional
public void createOrder(TradeOrderCreateReqVO reqVO) {
    // 1. 创建订单
    createOrderRecord(reqVO);

    // 2. 发送短信通知（耗时操作）
    sendSmsNotification(reqVO.getUserId());  // 不应该放在事务中

    // 3. 扣减库存
    productSpuApi.deductStock(reqVO.getSpuId(), 1);
}

// 好的设计：将非关键操作移到事务外
public void createOrder(TradeOrderCreateReqVO reqVO) {
    // 1. 创建订单（在全局事务中）
    Long orderId = doCreateOrder(reqVO);

    // 2. 发送短信通知（在事务外）
    sendSmsNotification(reqVO.getUserId());
}

@GlobalTransactional
public Long doCreateOrder(TradeOrderCreateReqVO reqVO) {
    // 创建订单 + 扣减库存 + 扣减积分
}
```

### 9.3 幂等性设计

分支事务可能被重试，需要保证幂等性：

```java
// 使用唯一键避免重复操作
@Override
public void deductStock(Long spuId, Integer quantity) {
    // 使用乐观锁或唯一键保证幂等
    int updated = stockMapper.update(
        "UPDATE stock SET quantity = quantity - #{quantity} " +
        "WHERE spu_id = #{spuId} AND quantity >= #{quantity}",
        spuId, quantity
    );
    if (updated == 0) {
        throw new ServiceException(STOCK_NOT_ENOUGH);
    }
}
```

### 9.4 不支持 NoSQL

Seata AT 模式基于 SQL 拦截，不支持 NoSQL 数据库（如 MongoDB、Redis）。如果需要在 NoSQL 场景下使用分布式事务，需要选择 TCC 或 Saga 模式。

## 常见问题排查

### 问题一：事务未回滚

**现象**：某个分支事务失败，但其他分支没有回滚

**排查步骤**：
1. 检查 `@GlobalTransactional` 是否在发起方
2. 检查 XID 是否正确传播（查看日志中的 XID）
3. 检查 Seata Server 是否正常运行
4. 检查 undo_log 表是否存在

### 问题二：undo_log 冲突

**现象**：`Duplicate entry 'xxx' for key 'ux_undo_log'`

**原因**：并发事务冲突

**解决**：
1. 增加重试机制
2. 优化事务粒度，减少事务持有时间

### 问题三：Seata Server 不可达

**现象**：`io.seata.common.exception.FrameworkException: can not connect to services-server`

**排查步骤**：
1. 检查 Seata Server 是否启动
2. 检查 Nacos 中是否有 Seata Server 的注册信息
3. 检查网络连通性

### 问题四：全局事务超时

**现象**：事务执行时间超过 `timeoutMills`，自动回滚

**解决**：
1. 增加超时时间：`@GlobalTransactional(timeoutMills = 120000)`
2. 优化业务逻辑，减少事务执行时间

## 本章小结

本章完成了 Seata 分布式事务的集成：

1. **undo_log 表**：在每个业务数据库中创建
2. **依赖引入**：添加 `seata-spring-boot-starter`
3. **配置**：配置 Seata Server 地址、事务分组
4. **数据源代理**：Seata 自动代理，无需手动配置
5. **@GlobalTransactional**：在发起方使用，自动管理全局事务
6. **XID 传播**：OpenFeign 自动传播，RestTemplate 需手动处理

下一章将介绍测试策略和源码索引。
