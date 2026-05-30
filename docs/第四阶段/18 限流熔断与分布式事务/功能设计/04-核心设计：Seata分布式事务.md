# 04 - 核心设计：Seata 分布式事务

## Seata 的三种模式

### AT 模式（Automatic Transaction）

AT 模式是 Seata 最常用的模式，核心思想是**自动补偿**：

```
AT 模式的工作原理：

1. 拦截业务 SQL
   │
   ├── 解析 SQL，识别表和字段
   ├── 查询修改前的数据（before image）
   └── 生成 undo_log 记录

2. 执行业务 SQL
   │
   └── 正常执行 SQL

3. 生成 after image
   │
   └── 查询修改后的数据

4. 提交本地事务
   │
   ├── 业务 SQL + undo_log 一起提交
   └── 向 TC 注册分支事务

5. 全局提交
   │
   └── 删除 undo_log

6. 全局回滚
   │
   ├── 根据 undo_log 反向补偿
   ├── INSERT → DELETE
   ├── DELETE → INSERT
   ├── UPDATE → UPDATE（还原）
   └── 删除 undo_log
```

### AT 模式的 undo_log

```sql
-- undo_log 表结构
CREATE TABLE undo_log (
    id bigint(20) NOT NULL AUTO_INCREMENT,
    branch_id bigint(20) NOT NULL,
    xid varchar(100) NOT NULL,
    context varchar(128) NOT NULL,
    rollback_info longblob NOT NULL,
    log_status int(11) NOT NULL,
    log_created datetime NOT NULL,
    log_modified datetime NOT NULL,
    ext varchar(100) DEFAULT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY ux_undo_log (xid, branch_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- 示例：UPDATE 操作的 undo_log
-- 业务 SQL：UPDATE stock SET quantity = 9 WHERE product_id = 1
-- before image：quantity = 10
-- after image：quantity = 9
-- rollback_info：UPDATE stock SET quantity = 10 WHERE product_id = 1
```

### TCC 模式（Try-Confirm-Cancel）

TCC 模式需要手动编写三个方法：

```java
// TCC 模式的三个阶段
public interface StockService {

    // Try：预留资源
    @TwoPhaseBusinessAction(name = "deductStock",
        commitMethod = "confirm", rollbackMethod = "cancel")
    boolean tryDeductStock(
        @BusinessActionContextParameter(paramName = "productId") Long productId,
        @BusinessActionContextParameter(paramName = "quantity") Integer quantity);

    // Confirm：确认提交
    boolean confirm(BusinessActionContext context);

    // Cancel：取消回滚
    boolean cancel(BusinessActionContext context);
}

// 实现
@Service
public class StockServiceImpl implements StockService {

    @Override
    public boolean tryDeductStock(Long productId, Integer quantity) {
        // Try：冻结库存（不是真正扣减）
        // UPDATE stock SET frozen = frozen + 1 WHERE product_id = 1
        return true;
    }

    @Override
    public boolean confirm(BusinessActionContext context) {
        // Confirm：真正扣减库存
        // UPDATE stock SET quantity = quantity - 1, frozen = frozen - 1 WHERE product_id = 1
        return true;
    }

    @Override
    public boolean cancel(BusinessActionContext context) {
        // Cancel：释放冻结库存
        // UPDATE stock SET frozen = frozen - 1 WHERE product_id = 1
        return true;
    }
}
```

### Saga 模式

Saga 模式适合长事务，每个步骤都有对应的补偿操作：

```
Saga 模式的执行流程：

正向流程：
  步骤 1：创建订单 → 步骤 2：扣减库存 → 步骤 3：扣减积分 → 步骤 4：创建支付单

补偿流程（步骤 3 失败）：
  步骤 3 失败 → 补偿步骤 2（恢复库存） → 补偿步骤 1（取消订单）

特点：
  ├── 每个步骤都有对应的补偿操作
  ├── 适合业务流程长的场景
  ├── 补偿操作需要业务方实现
  └── 最终一致性
```

## AT 模式的工作原理

### 全局事务流程

```
trade-server（TM）                    Seata Server（TC）
    │                                     │
    │  1. 开启全局事务                     │
    │ ─────────────────────────────────→ │
    │                                     │  生成全局事务 ID（XID）
    │ <───────────────────────────────── │
    │                                     │
    │  2. 执行业务 SQL                     │
    │  ├── 生成 before image              │
    │  ├── 执行 SQL                       │
    │  ├── 生成 after image               │
    │  └── 生成 undo_log                  │
    │                                     │
    │  3. 注册分支事务                     │
    │ ─────────────────────────────────→ │
    │                                     │  记录分支事务状态
    │ <───────────────────────────────── │
    │                                     │
    │  4. 提交本地事务                     │
    │  ├── 业务 SQL + undo_log 一起提交   │
    │  └── 本地事务完成                   │
    │                                     │
    │  5. 全局提交（所有分支成功）         │
    │ ─────────────────────────────────→ │
    │                                     │  通知所有分支删除 undo_log
    │ <───────────────────────────────── │
    │                                     │
    │  6. 删除 undo_log                   │
    │                                     │
```

### 全局回滚流程

```
trade-server（TM）                    Seata Server（TC）
    │                                     │
    │  1. 开启全局事务                     │
    │ ─────────────────────────────────→ │
    │                                     │
    │  2. 分支事务 1 成功                  │
    │  3. 分支事务 2 成功                  │
    │  4. 分支事务 3 失败                  │
    │                                     │
    │  5. 全局回滚                         │
    │ ─────────────────────────────────→ │
    │                                     │  通知所有分支回滚
    │ <───────────────────────────────── │
    │                                     │
    │  6. 分支事务 2 回滚                  │
    │  ├── 读取 undo_log                  │
    │  ├── 执行反向 SQL                   │
    │  └── 删除 undo_log                  │
    │                                     │
    │  7. 分支事务 1 回滚                  │
    │  ├── 读取 undo_log                  │
    │  ├── 执行反向 SQL                   │
    │  └── 删除 undo_log                  │
    │                                     │
    │  8. 全局事务完成（回滚）             │
    │ <───────────────────────────────── │
```

## @GlobalTransactional 注解

### 基本用法

在发起方的方法上添加 `@GlobalTransactional` 注解：

```java
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

### 注解属性

```java
@GlobalTransactional(
    timeoutMills = 60000,           // 全局事务超时时间（毫秒）
    name = "createOrder",           // 全局事务名称
    rollbackFor = Exception.class,  // 回滚异常类型
    noRollbackFor = BusinessException.class  // 不回滚的异常类型
)
```

### XID 传播

Seata 通过 XID（全局事务 ID）来关联分支事务：

```
trade-server 开启全局事务
    │
    ├── 生成 XID = 192.168.1.10:8091:12345
    │
    ├── 调用 product-server
    │   │
    │   ├── 请求头携带 XID
    │   │   Header: TX_XID = 192.168.1.10:8091:12345
    │   │
    │   └── product-server 从请求头获取 XID
    │       └── 注册分支事务，关联到 XID
    │
    └── 调用 member-server
        │
        ├── 请求头携带 XID
        │   Header: TX_XID = 192.168.1.10:8091:12345
        │
        └── member-server 从请求头获取 XID
            └── 注册分支事务，关联到 XID
```

## 与 Spring Boot 的集成

### 依赖引入

```xml
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-spring-boot-starter</artifactId>
    <version>${seata.version}</version>
</dependency>
```

### 配置

```yaml
seata:
  enabled: true
  application-id: ${spring.application.name}
  tx-service-group: yudao-cloud-tx-group  # 事务分组
  service:
    vgroup-mapping:
      yudao-cloud-tx-group: default       # 映射到 Seata Server 集群
  registry:
    type: nacos                            # 使用 Nacos 注册中心
    nacos:
      server-addr: 127.0.0.1:8848
      namespace: dev
```

### 数据源代理

Seata AT 模式需要代理数据源，自动拦截 SQL：

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource dataSource(DataSourceProperties properties) {
        // 创建原始数据源
        DataSource dataSource = properties.initializeDataSourceBuilder().build();

        // 使用 Seata 代理数据源
        return new DataSourceProxy(dataSource);
    }
}
```

## yudao-cloud 中的使用场景

### 典型场景：下单流程

```java
@Override
@GlobalTransactional
public Long createOrder(TradeOrderCreateReqVO reqVO) {
    // 1. 校验商品信息（远程调用）
    ProductSpuRespVO spu = productSpuApi.getSpu(reqVO.getSpuId()).getCheckedData();
    if (spu == null) {
        throw new ServiceException(PRODUCT_NOT_EXISTS);
    }

    // 2. 校验用户信息（远程调用）
    MemberUserRespVO user = memberUserApi.getUser(reqVO.getUserId()).getCheckedData();
    if (user == null) {
        throw new ServiceException(USER_NOT_EXISTS);
    }

    // 3. 扣减库存（远程调用，分支事务）
    productSpuApi.deductStock(reqVO.getSpuId(), reqVO.getQuantity());

    // 4. 扣减积分（远程调用，分支事务）
    memberPointApi.deductPoints(reqVO.getUserId(), 100);

    // 5. 创建订单（本地事务）
    TradeOrderDO order = new TradeOrderDO();
    order.setSpuId(reqVO.getSpuId());
    order.setUserId(reqVO.getUserId());
    order.setTotalPrice(spu.getPrice() * reqVO.getQuantity());
    tradeOrderMapper.insert(order);

    // 6. 创建支付单（远程调用，分支事务）
    payOrderApi.createPayment(order.getId(), order.getTotalPrice());

    return order.getId();
}
```

### 事务边界设计

```
好的设计：
  @GlobalTransactional
  public void createOrder() {
      // 1. 创建订单
      // 2. 扣减库存
      // 3. 扣减积分
      // 所有操作在一个全局事务中
  }

不好的设计：
  public void createOrder() {
      // 1. 创建订单（独立事务）
      createOrderRecord();

      // 2. 扣减库存（独立事务）
      deductStock();

      // 3. 扣减积分（独立事务）
      deductPoints();
      // 如果步骤 3 失败，步骤 1 和 2 无法回滚！
  }
```

### 注意事项

```
1. @GlobalTransactional 只能在发起方使用
   ├── 正确：trade-server 的 createOrder 方法
   └── 错误：product-server 的 deductStock 方法

2. 远程调用需要传播 XID
   ├── OpenFeign 自动传播（通过请求头）
   └── RestTemplate 需要手动传播

3. 避免长事务
   ├── 全局事务时间不宜过长
   ├── 避免在事务中做耗时操作
   └── 可以将非关键操作移到事务外

4. 幂等性设计
   ├── 分支事务需要支持重试
   └── 使用唯一键避免重复操作
```

## 思考题

1. **Seata AT 模式要求每个数据库都创建 undo_log 表。如果一个服务连接多个数据源，每个数据源都需要创建 undo_log 表吗？**

2. **在高并发场景下，undo_log 表可能成为性能瓶颈。如何优化 undo_log 的性能？**

3. **如果全局事务超时了，Seata 会如何处理？已提交的分支事务会回滚吗？**

4. **Seata 的 XID 通过请求头传播。如果服务间调用使用消息队列（异步），XID 如何传播？**
