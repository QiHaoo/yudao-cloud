# 03-Stream 消息监听与消费

> 本章对应功能设计：[03-核心设计：Redis MQ 实现](../功能设计/03-核心设计：Redis%20MQ实现.md) 中的"消费者：两种监听器"和"Redis Stream 的消费者组机制"部分。

## 本章目标

实现 Redis Stream 的消息监听与消费机制。完成本章后，你将理解：

1. `AbstractRedisStreamMessageListener` 如何消费 Stream 消息并自动 ACK
2. 消费者组的创建与注册过程
3. `StreamMessageListenerContainer` 如何驱动 `XREADGROUP` 拉取消息
4. Pending 消息重发和 Stream 清理两个后台任务的实现

## 第一步：广播消费——AbstractRedisChannelMessageListener

在深入 Stream 消费之前，先看更简单的 Pub/Sub 广播消费。两种监听器的结构几乎一样，理解了一个，另一个就自然明白了。

```java
// 源码位置：yudao-framework/yudao-spring-boot-starter-mq/src/main/java/cn/iocoder/yudao/framework/mq/redis/core/pubsub/AbstractRedisChannelMessageListener.java
public abstract class AbstractRedisChannelMessageListener<T extends AbstractRedisChannelMessage>
        implements MessageListener {

    private final Class<T> messageType;
    private final String channel;
    @Setter
    private RedisMQTemplate redisMQTemplate;

    @SneakyThrows
    protected AbstractRedisChannelMessageListener() {
        this.messageType = getMessageClass();
        this.channel = messageType.getDeclaredConstructor().newInstance().getChannel();
    }

    @Override
    public final void onMessage(Message message, byte[] bytes) {
        // 1. 反序列化
        T messageObj = JsonUtils.parseObject(message.getBody(), messageType);
        try {
            // 2. 拦截器 before
            consumeMessageBefore(messageObj);
            // 3. 业务消费
            this.onMessage(messageObj);
        } finally {
            // 4. 拦截器 after
            consumeMessageAfter(messageObj);
        }
    }

    public abstract void onMessage(T message);
}
```

### 泛型推断：自动获取消息类型

构造函数中的 `getMessageClass()` 是整个设计的关键——它通过反射自动获取泛型参数 `T` 的实际类型：

```java
@SuppressWarnings("unchecked")
private Class<T> getMessageClass() {
    Type type = TypeUtil.getTypeArgument(getClass(), 0);
    if (type == null) {
        throw new IllegalStateException(String.format("类型(%s) 需要设置消息类型", getClass().getName()));
    }
    return (Class<T>) type;
}
```

`TypeUtil.getTypeArgument(getClass(), 0)` 获取当前类继承时指定的第一个泛型参数。比如：

```java
public class RedisWebSocketMessageConsumer
        extends AbstractRedisChannelMessageListener<RedisWebSocketMessage> { ... }
```

`getClass()` 返回 `RedisWebSocketMessageConsumer.class`，`getTypeArgument(..., 0)` 返回 `RedisWebSocketMessage.class`。

这样做的好处是消费者子类不需要手动传 Class 对象，代码更简洁：

```java
// 不需要这样写：
public RedisWebSocketMessageConsumer() {
    super(RedisWebSocketMessage.class); // 冗余
}

// 直接写泛型参数就够了：
public class RedisWebSocketMessageConsumer
        extends AbstractRedisChannelMessageListener<RedisWebSocketMessage> { ... }
```

### Channel 自动推导

构造函数通过反射创建消息实例来获取 channel 名：

```java
this.channel = messageType.getDeclaredConstructor().newInstance().getChannel();
```

这行代码做了三件事：
1. `getDeclaredConstructor()` 获取无参构造器
2. `newInstance()` 创建一个临时消息实例
3. `getChannel()` 调用默认实现，返回类名

**为什么不用硬编码？** 如果用 `this.channel = "RedisWebSocketMessage"`，消息类改名时消费者也得跟着改，容易遗漏。通过反射自动推导，channel 名始终与消息类名一致。

### 模板方法模式

`onMessage(Message, byte[])` 被声明为 `final`，子类不能覆盖。它定义了固定的处理流程：

```
反序列化 → 拦截器 before → 业务消费 → 拦截器 after
```

子类只需实现 `onMessage(T message)` 处理业务逻辑，不需要关心拦截器链的调用。

## 第二步：集群消费——AbstractRedisStreamMessageListener

Stream 消费者与 Channel 消费者的结构非常相似，但增加了三个关键能力：消费者组、ACK 机制、Pending 队列。

```java
// 源码位置：yudao-framework/yudao-spring-boot-starter-mq/src/main/java/cn/iocoder/yudao/framework/mq/redis/core/stream/AbstractRedisStreamMessageListener.java
public abstract class AbstractRedisStreamMessageListener<T extends AbstractRedisStreamMessage>
        implements StreamListener<String, ObjectRecord<String, String>> {

    private final Class<T> messageType;
    @Getter
    private final String streamKey;
    @Value("${spring.application.name}")
    @Getter
    private String group;
    @Setter
    private RedisMQTemplate redisMQTemplate;

    @SneakyThrows
    protected AbstractRedisStreamMessageListener() {
        this.messageType = getMessageClass();
        this.streamKey = messageType.getDeclaredConstructor().newInstance().getStreamKey();
    }

    @Override
    public void onMessage(ObjectRecord<String, String> message) {
        T messageObj = JsonUtils.parseObject(message.getValue(), messageType);
        try {
            consumeMessageBefore(messageObj);
            // 消费消息
            this.onMessage(messageObj);
            // 关键：消费成功后 ACK
            redisMQTemplate.getRedisTemplate().opsForStream().acknowledge(group, message);
        } finally {
            consumeMessageAfter(messageObj);
        }
    }

    public abstract void onMessage(T message);
}
```

### 与 Channel 消费者的三个关键区别

| 维度 | Channel 消费者 | Stream 消费者 |
|------|---------------|--------------|
| 接口 | `MessageListener` | `StreamListener<String, ObjectRecord<String, String>>` |
| 消费者组 | 无（广播） | `@Value("${spring.application.name}")` 自动使用应用名 |
| ACK | 无（Pub/Sub 无 ACK 概念） | 消费成功后调用 `acknowledge()` |
| 反序列化来源 | `message.getBody()` | `message.getValue()` |

### 消费者组的设计

`group` 字段通过 `@Value("${spring.application.name}")` 注入，默认使用 Spring 应用名。这意味着：

- 同一个应用的多个实例属于同一个消费者组
- 组内的消息只会被一个实例消费（集群消费）
- 不同应用各自独立消费所有消息

```
Stream: order-created
  │
  ├─ Consumer Group: order-service      ← spring.application.name=order-service
  │    ├─ Consumer A (192.168.1.1@1234)
  │    └─ Consumer B (192.168.1.2@5678)
  │
  └─ Consumer Group: analytics-service  ← spring.application.name=analytics-service
       └─ Consumer C (192.168.1.3@9012)
```

### ACK 机制

```java
redisMQTemplate.getRedisTemplate().opsForStream().acknowledge(group, message);
```

`acknowledge()` 执行 Redis 的 `XACK` 命令，告知 Redis 该消息已被成功处理。ACK 后消息不会从 Stream 中删除（Stream 设计如此），但会从消费者的 Pending 队列中移除。

**为什么 ACK 放在 `try` 块内而不是 `finally` 中？**

如果 `onMessage(messageObj)` 抛出异常，说明消费失败，不应该 ACK。消息会留在 Pending 队列中，等待 `RedisPendingMessageResendJob` 重新投递。如果放在 `finally` 中，消费失败也会 ACK，消息就丢了。

## 第三步：消费者组的创建与注册

消费者组的创建和监听器注册由 `YudaoRedisMQConsumerAutoConfiguration` 自动完成。

### 自动配置类

```java
// 源码位置：yudao-framework/yudao-spring-boot-starter-mq/src/main/java/cn/iocoder/yudao/framework/mq/redis/config/YudaoRedisMQConsumerAutoConfiguration.java
@Slf44
@EnableScheduling
@AutoConfiguration(after = YudaoRedisAutoConfiguration.class)
public class YudaoRedisMQConsumerAutoConfiguration {

    @Bean(initMethod = "start", destroyMethod = "stop")
    @ConditionalOnBean(AbstractRedisStreamMessageListener.class)
    public StreamMessageListenerContainer<String, ObjectRecord<String, String>> redisStreamMessageListenerContainer(
            RedisMQTemplate redisMQTemplate, List<AbstractRedisStreamMessageListener<?>> listeners) {
        // ...
    }
}
```

**`@ConditionalOnBean(AbstractRedisStreamMessageListener.class)`**：只有存在 Stream 监听器时才创建容器。如果项目只用 Pub/Sub，不会创建 Stream 相关的 Bean，节省资源。

### 完整的容器创建流程

```java
@Bean(initMethod = "start", destroyMethod = "stop")
@ConditionalOnBean(AbstractRedisStreamMessageListener.class)
public StreamMessageListenerContainer<String, ObjectRecord<String, String>> redisStreamMessageListenerContainer(
        RedisMQTemplate redisMQTemplate, List<AbstractRedisStreamMessageListener<?>> listeners) {
    RedisTemplate<String, ?> redisTemplate = redisMQTemplate.getRedisTemplate();
    checkRedisVersion(redisTemplate);

    // 第一步，创建 StreamMessageListenerContainer 容器
    StreamMessageListenerContainer.StreamMessageListenerContainerOptions<String, ObjectRecord<String, String>> containerOptions =
            StreamMessageListenerContainer.StreamMessageListenerContainerOptions.builder()
                    .batchSize(10)          // 一次性最多拉取 10 条
                    .targetType(String.class)
                    .build();
    StreamMessageListenerContainer<String, ObjectRecord<String, String>> container =
            StreamMessageListenerContainer.create(redisMQTemplate.getRedisTemplate().getRequiredConnectionFactory(), containerOptions);

    // 第二步，注册监听器
    String consumerName = buildConsumerName();
    listeners.parallelStream().forEach(listener -> {
        // 创建消费者组（如果不存在）
        try {
            redisTemplate.opsForStream().createGroup(listener.getStreamKey(), listener.getGroup());
        } catch (Exception ignore) {
            // 如果 group 已存在会抛异常，忽略即可
        }
        // 设置 redisMQTemplate（用于 ACK）
        listener.setRedisMQTemplate(redisMQTemplate);
        // 创建 Consumer 对象
        Consumer consumer = Consumer.from(listener.getGroup(), consumerName);
        // 设置消费进度
        StreamOffset<String> streamOffset = StreamOffset.create(listener.getStreamKey(), ReadOffset.lastConsumed());
        // 注册到容器
        StreamMessageListenerContainer.StreamReadRequestBuilder<String> builder =
                StreamMessageListenerContainer.StreamReadRequest
                        .builder(streamOffset).consumer(consumer)
                        .autoAcknowledge(false)    // 不自动 ACK，由监听器手动 ACK
                        .cancelOnError(throwable -> false); // 发生异常不取消消费
        container.register(builder.build(), listener);
    });
    return container;
}
```

逐步拆解：

**1. 创建容器**

```java
StreamMessageListenerContainer.StreamMessageListenerContainerOptions<String, ObjectRecord<String, String>> containerOptions =
        StreamMessageListenerContainer.StreamMessageListenerContainerOptions.builder()
                .batchSize(10)
                .targetType(String.class)
                .build();
```

`batchSize(10)` 对应 Redis `XREADGROUP` 的 `COUNT` 参数，每次最多拉取 10 条消息。设置太大会增加单次处理延迟，设置太小会增加 Redis 请求频率。

**2. 创建消费者组**

```java
redisTemplate.opsForStream().createGroup(listener.getStreamKey(), listener.getGroup());
```

执行 Redis 的 `XGROUP CREATE` 命令。如果 Stream 不存在，Redis 会自动创建；如果 Group 已存在会抛异常，代码中用 `try-catch` 忽略。

**3. 消费者名**

```java
public static String buildConsumerName() {
    return String.format("%s@%d", SystemUtil.getHostInfo().getAddress(), SystemUtil.getCurrentPID());
}
```

格式为 `IP@PID`，如 `192.168.1.100@1234`。参考了 RocketMQ 的 clientId 设计，保证同一台机器上不同进程的消费者名不同。

**4. 消费进度**

```java
StreamOffset<String> streamOffset = StreamOffset.create(listener.getStreamKey(), ReadOffset.lastConsumed());
```

`ReadOffset.lastConsumed()` 表示从上次消费到的位置继续消费。如果是新创建的 Group，从 Stream 的最新消息开始。

**5. 关键配置**

```java
.autoAcknowledge(false)           // 关闭自动 ACK
.cancelOnError(throwable -> false) // 异常时不取消消费
```

`autoAcknowledge(false)` 非常重要——如果开启自动 ACK，消息被拉取后立即标记为已消费，不管业务处理是否成功。关闭后由监听器在 `onMessage` 中手动 ACK。

`cancelOnError(throwable -> false)` 防止消费异常导致消费者被移除。默认行为是发生异常就取消消费，这对生产环境不可接受。

### Redis 版本检查

```java
public static void checkRedisVersion(RedisTemplate<String, ?> redisTemplate) {
    Properties info = redisTemplate.execute((RedisCallback<Properties>) RedisServerCommands::info);
    String version = MapUtil.getStr(info, "redis_version");
    int majorVersion = Integer.parseInt(StrUtil.subBefore(version, '.', false));
    if (majorVersion < 5) {
        throw new IllegalStateException(StrUtil.format(
            "您当前的 Redis 版本为 {}，小于最低要求的 5.0.0 版本！", version));
    }
}
```

Redis Stream 是 5.0 引入的特性，如果 Redis 版本低于 5.0，直接抛出明确的错误信息，避免运行时出现难以理解的异常。

## 第四步：Pub/Sub 容器的创建

Channel 消息的容器创建更简单，因为不需要消费者组和 ACK 机制：

```java
// 源码位置：yudao-framework/yudao-spring-boot-starter-mq/src/main/java/cn/iocoder/yudao/framework/mq/redis/config/YudaoRedisMQConsumerAutoConfiguration.java
@Bean
@ConditionalOnBean(AbstractRedisChannelMessageListener.class)
public RedisMessageListenerContainer redisMessageListenerContainer(
        RedisMQTemplate redisMQTemplate, List<AbstractRedisChannelMessageListener<?>> listeners) {
    RedisMessageListenerContainer container = new RedisMessageListenerContainer();
    container.setConnectionFactory(redisMQTemplate.getRedisTemplate().getRequiredConnectionFactory());
    listeners.forEach(listener -> {
        listener.setRedisMQTemplate(redisMQTemplate);
        container.addMessageListener(listener, new ChannelTopic(listener.getChannel()));
        log.info("[redisMessageListenerContainer][注册 Channel({}) 对应的监听器({})]",
                listener.getChannel(), listener.getClass().getName());
    });
    return container;
}
```

`RedisMessageListenerContainer` 是 Spring Data Redis 提供的 Pub/Sub 容器，内部维护一个长连接订阅所有注册的 Channel。收到消息后根据 `ChannelTopic` 路由到对应的 `MessageListener`。

## 第五步：后台任务——Pending 消息重发

当消费者崩溃或处理失败时，消息会卡在 Pending 队列。`RedisPendingMessageResendJob` 负责将超时的消息重新投递。

```java
// 源码位置：yudao-framework/yudao-spring-boot-starter-mq/src/main/java/cn/iocoder/yudao/framework/mq/redis/core/job/RedisPendingMessageResendJob.java
@Slf44
@AllArgsConstructor
public class RedisPendingMessageResendJob {

    private static final String LOCK_KEY = "redis:stream:pending-message-resend:lock";
    private static final int EXPIRE_TIME = 5 * 60; // 5 分钟超时

    private final List<AbstractRedisStreamMessageListener<?>> listeners;
    private final RedisMQTemplate redisTemplate;
    private final RedissonClient redissonClient;

    @Scheduled(cron = "35 * * * * ?") // 每分钟的第 35 秒执行
    public void messageResend() {
        RLock lock = redissonClient.getLock(LOCK_KEY);
        if (lock.tryLock()) {
            try {
                execute();
            } catch (Exception ex) {
                log.error("[messageResend][执行异常]", ex);
            } finally {
                lock.unlock();
            }
        }
    }
}
```

### 执行逻辑

```java
private void execute() {
    StreamOperations<String, Object, Object> ops = redisTemplate.getRedisTemplate().opsForStream();
    listeners.forEach(listener -> {
        // 1. 获取 Pending 摘要
        PendingMessagesSummary pendingMessagesSummary = Objects.requireNonNull(
                ops.pending(listener.getStreamKey(), listener.getGroup()));
        // 2. 遍历每个消费者的 Pending 消息
        Map<String, Long> pendingMessagesPerConsumer = pendingMessagesSummary.getPendingMessagesPerConsumer();
        pendingMessagesPerConsumer.forEach((consumerName, pendingMessageCount) -> {
            // 3. 获取 Pending 消息详情
            PendingMessages pendingMessages = ops.pending(
                    listener.getStreamKey(),
                    Consumer.from(listener.getGroup(), consumerName),
                    Range.unbounded(), pendingMessageCount);
            if (pendingMessages.isEmpty()) {
                return;
            }
            pendingMessages.forEach(pendingMessage -> {
                // 4. 检查是否超时（5 分钟）
                long lastDelivery = pendingMessage.getElapsedTimeSinceLastDelivery().getSeconds();
                if (lastDelivery < EXPIRE_TIME) {
                    return; // 未超时，跳过
                }
                // 5. 获取原始消息内容
                List<MapRecord<String, Object, Object>> records = ops.range(listener.getStreamKey(),
                        Range.of(Range.Bound.inclusive(pendingMessage.getIdAsString()),
                                 Range.Bound.inclusive(pendingMessage.getIdAsString())));
                if (CollUtil.isEmpty(records)) {
                    return;
                }
                // 6. 重新投递 + ACK 原消息
                redisTemplate.getRedisTemplate().opsForStream().add(StreamRecords.newRecord()
                        .ofObject(records.get(0).getValue())
                        .withStreamKey(listener.getStreamKey()));
                redisTemplate.getRedisTemplate().opsForStream().acknowledge(listener.getGroup(), records.get(0));
                log.info("[processPendingMessage][消息({})重新投递成功]", records.get(0).getId());
            });
        });
    });
}
```

### 重发流程图

```
Pending 队列中的消息
  │
  ├─ elapsed < 5 分钟 → 跳过（可能正在处理中）
  │
  └─ elapsed >= 5 分钟 → 认为消费失败
       │
       ├─ XADD：创建新消息（新 ID）
       └─ XACK：ACK 原消息（从 Pending 移除）
```

**为什么用"重新投递 + ACK 原消息"而不是"直接转给其他消费者"？**

Redis Stream 的 `XAUTOCLAIM` 命令可以实现直接转给其他消费者，但 yudao-cloud 选择了更简单的方式：创建一条新消息，ACK 掉原消息。这样做的好处是逻辑简单，新消息会进入正常消费流程，由 Group 内的任意消费者处理。

**为什么用分布式锁？**

多实例部署时，只需要一个实例执行重发任务。如果每个实例都执行，同一条 Pending 消息会被重复投递，导致消息被消费多次。分布式锁确保同一时刻只有一个实例在执行重发。

## 第六步：后台任务——Stream 消息清理

Redis Stream 的消息会一直保留，如果不清理，内存会持续增长。`RedisStreamMessageCleanupJob` 定期清理旧消息。

```java
// 源码位置：yudao-framework/yudao-spring-boot-starter-mq/src/main/java/cn/iocoder/yudao/framework/mq/redis/core/job/RedisStreamMessageCleanupJob.java
@Slf44
@AllArgsConstructor
public class RedisStreamMessageCleanupJob {

    private static final String LOCK_KEY = "redis:stream:message-cleanup:lock";
    private static final long MAX_COUNT = 10000; // 保留最近 10000 条

    private final List<AbstractRedisStreamMessageListener<?>> listeners;
    private final RedisMQTemplate redisTemplate;
    private final RedissonClient redissonClient;

    @Scheduled(cron = "0 0 * * * ?") // 每小时执行
    public void cleanup() {
        RLock lock = redissonClient.getLock(LOCK_KEY);
        if (lock.tryLock()) {
            try {
                execute();
            } catch (Exception ex) {
                log.error("[cleanup][执行异常]", ex);
            } finally {
                lock.unlock();
            }
        }
    }

    private void execute() {
        StreamOperations<String, Object, Object> ops = redisTemplate.getRedisTemplate().opsForStream();
        listeners.forEach(listener -> {
            try {
                // XTRIM 命令：只保留最近 MAX_COUNT 条消息
                Long trimCount = ops.trim(listener.getStreamKey(), MAX_COUNT, true);
                if (trimCount != null && trimCount > 0) {
                    log.info("[execute][Stream({}) 清理消息数量({})]", listener.getStreamKey(), trimCount);
                }
            } catch (Exception ex) {
                log.error("[execute][Stream({}) 清理异常]", listener.getStreamKey(), ex);
            }
        });
    }
}
```

`ops.trim(listener.getStreamKey(), MAX_COUNT, true)` 执行 Redis 的 `XTRIM key MAXLEN ~ 10000` 命令。第三个参数 `true` 表示使用近似裁剪（`~`），Redis 会优化内部节点结构，性能比精确裁剪更好。

## 完整的消费流程

将所有组件串起来，一条 Stream 消息从发送到消费的完整流程如下：

```
1. Producer 发送
   redisMQTemplate.send(new OrderCreatedMessage().setOrderId(100L))
     │
     ├─ sendMessageBefore：拦截器正序执行
     ├─ XADD：写入 Stream
     └─ sendMessageAfter：拦截器倒序执行

2. Consumer 拉取
   StreamMessageListenerContainer 内部循环
     │
     └─ XREADGROUP：从 Stream 拉取消息
        └─ 消息进入 AbstractRedisStreamMessageListener.onMessage()

3. Consumer 处理
   onMessage(ObjectRecord message)
     │
     ├─ 反序列化：JsonUtils.parseObject(message.getValue(), messageType)
     ├─ consumeMessageBefore：拦截器正序执行
     ├─ this.onMessage(messageObj)：业务逻辑
     ├─ XACK：确认消费成功
     └─ consumeMessageAfter：拦截器倒序执行

4. 异常处理（消费失败时）
   │
   ├─ 不执行 XACK，消息留在 Pending 队列
   └─ RedisPendingMessageResendJob 每分钟扫描
      └─ 超过 5 分钟的消息重新投递
```

## 实际使用示例

来看一个业务模块如何使用 Stream 消费者。以工作流模块的消息事件为例：

```java
// 消息定义
@Data
public class BpmProcessInstanceResultMessage extends AbstractRedisStreamMessage {
    private Long processInstanceId;
    private Integer status; // 2 通过 3 驳回
    private Long userId;
    // getStreamKey() 使用默认值 "BpmProcessInstanceResultMessage"
}

// 消费者
@RequiredArgsConstructor
public class BpmProcessInstanceResultConsumer
        extends AbstractRedisStreamMessageListener<BpmProcessInstanceResultMessage> {

    private final BpmProcessInstanceApi processInstanceApi;

    @Override
    public void onMessage(BpmProcessInstanceResultMessage message) {
        // 处理审批结果：更新订单状态、发送通知等
        processInstanceApi.updateProcessInstanceResult(
                message.getProcessInstanceId(), message.getStatus());
    }
}
```

消费流程：

```
1. 工作流服务调用 redisMQTemplate.send(message)
   → XADD BpmProcessInstanceResultMessage "{processInstanceId:1, status:2, userId:100}"

2. 订单服务的 BpmProcessInstanceResultConsumer 收到消息
   → 反序列化为 BpmProcessInstanceResultMessage 对象
   → 调用 onMessage() 处理业务逻辑
   → XACK 确认消费

3. 如果订单服务有 3 个实例，只有一个实例会收到这条消息（集群消费）
```

## 本章小结

| 组件 | 职责 | 关键机制 |
|------|------|---------|
| `AbstractRedisChannelMessageListener` | 广播消费 | 泛型推断消息类型、模板方法模式 |
| `AbstractRedisStreamMessageListener` | 集群消费 | 消费者组（应用名）、手动 ACK |
| `StreamMessageListenerContainer` | Stream 容器 | XREADGROUP 拉取、消费者组创建 |
| `RedisPendingMessageResendJob` | Pending 重发 | 5 分钟超时、分布式锁 |
| `RedisStreamMessageCleanupJob` | 消息清理 | XTRIM 近似裁剪、每小时执行 |
