# 03-核心设计：Redis MQ 实现

## 消息基类体系

yudao-cloud 的 Redis MQ 建立在一个清晰的类继承体系之上：

```
AbstractRedisMessage                  ← 消息基类，持有 headers Map
    ├── AbstractRedisChannelMessage   ← Pub/Sub 消息，getChannel() 指定频道
    └── AbstractRedisStreamMessage    ← Stream 消息，getStreamKey() 指定流
```

### AbstractRedisMessage：消息的根基

所有消息都继承这个基类，它提供了一个 `headers` Map 用于携带元数据（如租户 ID）。

```java
// 源码位置：cn/iocoder/yudao/framework/mq/redis/core/message/AbstractRedisMessage.java
@Data
public abstract class AbstractRedisMessage {

    private Map<String, String> headers = new HashMap<>();

    public String getHeader(String key) {
        return headers.get(key);
    }

    public void addHeader(String key, String value) {
        headers.put(key, value);
    }
}
```

为什么用 `Map<String, String>` 而不是强类型字段？因为 headers 是扩展点——拦截器可以往里面塞任何东西（租户 ID、链路追踪 ID、时间戳等），用 Map 提供了最大的灵活性。

### AbstractRedisChannelMessage：广播消息

```java
// 源码位置：cn/iocoder/yudao/framework/mq/redis/core/pubsub/AbstractRedisChannelMessage.java
public abstract class AbstractRedisChannelMessage extends AbstractRedisMessage {

    @JsonIgnore // 避免序列化到消息体中，因为发送时已经指定了 channel
    public String getChannel() {
        return getClass().getSimpleName(); // 默认用类名作为 channel
    }
}
```

`getChannel()` 默认返回类的简单名。这意味着每个消息类自动拥有一个唯一的频道名，不需要额外配置。比如 `RedisWebSocketMessage` 类的 channel 就是 `"RedisWebSocketMessage"`。

### AbstractRedisStreamMessage：可靠消息

```java
// 源码位置：cn/iocoder/yudao/framework/mq/redis/core/stream/AbstractRedisStreamMessage.java
public abstract class AbstractRedisStreamMessage extends AbstractRedisMessage {

    @JsonIgnore
    public String getStreamKey() {
        return getClass().getSimpleName(); // 默认用类名作为 stream key
    }
}
```

与 ChannelMessage 几乎一样，区别在于语义：Channel 对应 Pub/Sub 的频道，StreamKey 对应 Stream 的键。

## RedisMQTemplate：统一发送 API

`RedisMQTemplate` 是整个 Redis MQ 的核心入口，它提供了两个 `send` 方法，分别对应两种消息模式。

```java
// 源码位置：cn/iocoder/yudao/framework/mq/redis/core/RedisMQTemplate.java
@AllArgsConstructor
public class RedisMQTemplate {

    @Getter
    private final RedisTemplate<String, ?> redisTemplate;
    @Getter
    private final List<RedisMessageInterceptor> interceptors = new ArrayList<>();

    /**
     * 发送 Channel 消息（Pub/Sub 广播）
     */
    public <T extends AbstractRedisChannelMessage> void send(T message) {
        try {
            sendMessageBefore(message);
            redisTemplate.convertAndSend(message.getChannel(), JsonUtils.toJsonString(message));
        } finally {
            sendMessageAfter(message);
        }
    }

    /**
     * 发送 Stream 消息（可靠投递）
     */
    public <T extends AbstractRedisStreamMessage> RecordId send(T message) {
        try {
            sendMessageBefore(message);
            return redisTemplate.opsForStream().add(StreamRecords.newRecord()
                    .ofObject(JsonUtils.toJsonString(message))
                    .withStreamKey(message.getStreamKey()));
        } finally {
            sendMessageAfter(message);
        }
    }
}
```

### 两个 send 方法的对比

| 维度 | send(ChannelMessage) | send(StreamMessage) |
|------|---------------------|---------------------|
| 底层命令 | `PUBLISH` | `XADD` |
| 返回值 | void | RecordId（消息 ID） |
| 持久化 | 不持久化 | 持久化到 AOF/RDB |
| 消费模式 | 广播（所有订阅者收到） | 集群（组内一个消费者收到） |
| 适用场景 | 实时通知、WebSocket 广播 | 订单处理、库存扣减 |

### 拦截器链的执行顺序

发送和消费都遵循"洋葱模型"：

```
发送流程：
  sendMessageBefore [拦截器1, 拦截器2, 拦截器3]  ← 正序
      ↓
  实际发送消息
      ↓
  sendMessageAfter  [拦截器3, 拦截器2, 拦截器1]  ← 倒序

消费流程：
  consumeMessageBefore [拦截器1, 拦截器2, 拦截器3]  ← 正序
      ↓
  实际消费消息
      ↓
  consumeMessageAfter  [拦截器3, 拦截器2, 拦截器1]  ← 倒序
```

源码实现：

```java
private void sendMessageBefore(AbstractRedisMessage message) {
    // 正序
    interceptors.forEach(interceptor -> interceptor.sendMessageBefore(message));
}

private void sendMessageAfter(AbstractRedisMessage message) {
    // 倒序
    for (int i = interceptors.size() - 1; i >= 0; i--) {
        interceptors.get(i).sendMessageAfter(message);
    }
}
```

为什么用 `try-finally` 包裹？确保 `sendMessageAfter` 一定执行，即使发送过程中抛异常。这对于清理上下文（如租户信息）非常重要。

## 消费者：两种监听器

### AbstractRedisChannelMessageListener：广播消费

```java
// 源码位置：cn/iocoder/yudao/framework/mq/redis/core/pubsub/AbstractRedisChannelMessageListener.java
public abstract class AbstractRedisChannelMessageListener<T extends AbstractRedisChannelMessage>
        implements MessageListener {

    private final Class<T> messageType;
    private final String channel;
    @Setter
    private RedisMQTemplate redisMQTemplate;

    @SneakyThrows
    protected AbstractRedisChannelMessageListener() {
        this.messageType = getMessageClass();
        // 通过反射创建消息实例，调用 getChannel() 获取频道名
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

关键设计点：

1. **泛型推断**：构造函数通过 `TypeUtil.getTypeArgument(getClass(), 0)` 自动获取泛型参数类型，不需要手动传 Class 对象
2. **模板方法模式**：`onMessage(Message, byte[])` 是 final 的，定义了固定的处理流程；子类只需实现 `onMessage(T)` 处理业务逻辑
3. **channel 自动推导**：通过反射创建消息实例并调用 `getChannel()`，避免硬编码

### AbstractRedisStreamMessageListener：集群消费

```java
// 源码位置：cn/iocoder/yudao/framework/mq/redis/core/stream/AbstractRedisStreamMessageListener.java
public abstract class AbstractRedisStreamMessageListener<T extends AbstractRedisStreamMessage>
        implements StreamListener<String, ObjectRecord<String, String>> {

    private final Class<T> messageType;
    @Getter
    private final String streamKey;
    @Value("${spring.application.name}")
    @Getter
    private String group; // 消费者组名，默认用应用名
    @Setter
    private RedisMQTemplate redisMQTemplate;

    @Override
    public void onMessage(ObjectRecord<String, String> message) {
        T messageObj = JsonUtils.parseObject(message.getValue(), messageType);
        try {
            consumeMessageBefore(messageObj);
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

与 ChannelMessageListener 的关键区别：

1. **消费者组**：通过 `@Value("${spring.application.name}")` 自动使用应用名作为组名，同组内只有一个消费者收到消息
2. **ACK 机制**：消费成功后调用 `acknowledge()`，告知 Redis 该消息已处理完毕
3. **Pending 队列**：未 ACK 的消息会留在 Pending 队列中，由 `RedisPendingMessageResendJob` 定期重发

### 消费者组机制详解

```
Stream: order-created
  │
  ├─ Consumer Group: order-service
  │    ├─ Consumer A (IP: 192.168.1.1, PID: 1234)
  │    │    └─ 处理消息 1, 3, 5
  │    └─ Consumer B (IP: 192.168.1.2, PID: 5678)
  │         └─ 处理消息 2, 4, 6
  │
  └─ Consumer Group: analytics-service
       └─ Consumer C (IP: 192.168.1.3, PID: 9012)
            └─ 处理消息 1, 2, 3, 4, 5, 6（全部）
```

- 同一个 Group 内的消息只被一个 Consumer 处理（集群消费）
- 不同 Group 各自独立消费所有消息（广播消费）
- 这就是为什么 `group` 默认用 `spring.application.name`——同一个应用的多个实例属于同一个组

## Redis Stream 的消费者组机制

### 集群消费的实现

自动配置类中，消费者组的注册过程如下：

```java
// 源码位置：YudaoRedisMQConsumerAutoConfiguration.java
@Bean(initMethod = "start", destroyMethod = "stop")
public StreamMessageListenerContainer<String, ObjectRecord<String, String>> redisStreamMessageListenerContainer(
        RedisMQTemplate redisMQTemplate, List<AbstractRedisStreamMessageListener<?>> listeners) {
    // 1. 创建容器
    StreamMessageListenerContainer.StreamMessageListenerContainerOptions<String, ObjectRecord<String, String>> containerOptions =
            StreamMessageListenerContainer.StreamMessageListenerContainerOptions.builder()
                    .batchSize(10) // 一次性最多拉取 10 条
                    .targetType(String.class)
                    .build();
    StreamMessageListenerContainer<String, ObjectRecord<String, String>> container =
            StreamMessageListenerContainer.create(redisMQTemplate.getRedisTemplate().getRequiredConnectionFactory(), containerOptions);

    // 2. 注册监听器
    String consumerName = buildConsumerName(); // IP@PID 格式
    listeners.parallelStream().forEach(listener -> {
        // 创建消费者组（如果不存在）
        redisTemplate.opsForStream().createGroup(listener.getStreamKey(), listener.getGroup());
        // 创建消费者
        Consumer consumer = Consumer.from(listener.getGroup(), consumerName);
        // 注册到容器
        container.register(builder.build(), listener);
    });
    return container;
}

// 消费者名：IP@PID，参考 RocketMQ 的 clientId 设计
public static String buildConsumerName() {
    return String.format("%s@%d", SystemUtil.getHostInfo().getAddress(), SystemUtil.getCurrentPID());
}
```

### ACK 确认流程

```
Producer ──XADD──→ Stream
                      │
                      ↓
Consumer ──XREADGROUP──→ 读取消息（消息进入 Pending 队列）
                      │
                      ↓
              处理消息成功？
              ├─ 是 ──→ XACK ──→ 消息从 Pending 队列移除
              └─ 否 ──→ 不 ACK ──→ 消息留在 Pending 队列
                                    │
                                    ↓
                          RedisPendingMessageResendJob
                          （每分钟扫描，超时 5 分钟重发）
```

## 后台任务：保障消息可靠性

### RedisPendingMessageResendJob：待处理消息重发

当消费者崩溃或处理失败时，消息会卡在 Pending 队列。这个定时任务负责将超时的消息重新投递。

```java
// 源码位置：cn/iocoder/yudao/framework/mq/redis/core/job/RedisPendingMessageResendJob.java
@Slf4j
@AllArgsConstructor
public class RedisPendingMessageResendJob {

    private static final String LOCK_KEY = "redis:stream:pending-message-resend:lock";
    private static final int EXPIRE_TIME = 5 * 60; // 5 分钟超时

    @Scheduled(cron = "35 * * * * ?") // 每分钟的第 35 秒执行
    public void messageResend() {
        RLock lock = redissonClient.getLock(LOCK_KEY);
        if (lock.tryLock()) { // 分布式锁，确保只有一个节点执行
            try {
                execute();
            } finally {
                lock.unlock();
            }
        }
    }

    private void execute() {
        listeners.forEach(listener -> {
            // 获取 Pending 摘要
            PendingMessagesSummary summary = ops.pending(listener.getStreamKey(), listener.getGroup());
            // 遍历每个消费者的 Pending 消息
            summary.getPendingMessagesPerConsumer().forEach((consumerName, count) -> {
                PendingMessages pendingMessages = ops.pending(...);
                pendingMessages.forEach(pendingMessage -> {
                    // 超过 5 分钟才重发
                    if (pendingMessage.getElapsedTimeSinceLastDelivery().getSeconds() < EXPIRE_TIME) {
                        return;
                    }
                    // 重新投递 + ACK 原消息
                    ops.add(...); // 新消息
                    ops.acknowledge(...); // ACK 原消息
                });
            });
        });
    }
}
```

为什么用分布式锁？因为多实例部署时，只需要一个实例执行重发任务，避免重复投递。

### RedisStreamMessageCleanupJob：清理过期消息

Redis Stream 的消息会一直保留，如果不清理，内存会持续增长。这个任务定期清理旧消息。

```java
// 源码位置：cn/iocoder/yudao/framework/mq/redis/core/job/RedisStreamMessageCleanupJob.java
@Slf4j
@AllArgsConstructor
public class RedisStreamMessageCleanupJob {

    private static final String LOCK_KEY = "redis:stream:message-cleanup:lock";
    private static final long MAX_COUNT = 10000; // 保留最近 10000 条

    @Scheduled(cron = "0 0 * * * ?") // 每小时执行
    public void cleanup() {
        RLock lock = redissonClient.getLock(LOCK_KEY);
        if (lock.tryLock()) {
            try {
                execute();
            } finally {
                lock.unlock();
            }
        }
    }

    private void execute() {
        listeners.forEach(listener -> {
            // XTRIM 命令：只保留最近 MAX_COUNT 条消息
            Long trimCount = ops.trim(listener.getStreamKey(), MAX_COUNT, true);
        });
    }
}
```

`XTRIM` 命令配合 `MAXLEN` 参数，高效地删除旧消息。`true` 参数表示使用近似裁剪（`~`），Redis 会优化内部节点结构，性能更好。

## 实际使用示例：WebSocket 消息广播

来看一个完整的使用案例——通过 Redis Pub/Sub 实现 WebSocket 消息的跨节点广播。

### 消息定义

```java
// 源码位置：cn/iocoder/yudao/framework/websocket/core/sender/redis/RedisWebSocketMessage.java
@Data
public class RedisWebSocketMessage extends AbstractRedisChannelMessage {
    private String sessionId;
    private Integer userType;
    private Long userId;
    private String messageType;
    private String messageContent;
}
```

继承 `AbstractRedisChannelMessage`，使用广播模式——每个节点都需要收到消息，因为 WebSocket 连接分散在各个节点上。

### 消费者

```java
// 源码位置：cn/iocoder/yudao/framework/websocket/core/sender/redis/RedisWebSocketMessageConsumer.java
@RequiredArgsConstructor
public class RedisWebSocketMessageConsumer extends AbstractRedisChannelMessageListener<RedisWebSocketMessage> {

    private final RedisWebSocketMessageSender redisWebSocketMessageSender;

    @Override
    public void onMessage(RedisWebSocketMessage message) {
        redisWebSocketMessageSender.send(message.getSessionId(),
                message.getUserType(), message.getUserId(),
                message.getMessageType(), message.getMessageContent());
    }
}
```

泛型参数 `<RedisWebSocketMessage>` 让框架自动推断 channel 名和消息类型，消费者只需关注业务逻辑。

### 流程图

```
节点 A（用户发起聊天）
  │
  ├─ redisMQTemplate.send(RedisWebSocketMessage)
  │    └─ PUBLISH RedisWebSocketMessage "{json}"
  │
  ↓
Redis Server
  │
  ├─→ 节点 A 的 RedisWebSocketMessageConsumer
  │    └─ 检查 sessionId 是否在本节点 → 是 → 发送 WebSocket 消息
  │
  ├─→ 节点 B 的 RedisWebSocketMessageConsumer
  │    └─ 检查 sessionId 是否在本节点 → 是 → 发送 WebSocket 消息
  │
  └─→ 节点 C 的 RedisWebSocketMessageConsumer
       └─ 检查 sessionId 是否在本节点 → 否 → 忽略
```

## 消息拦截器：RedisMessageInterceptor

```java
// 源码位置：cn/iocoder/yudao/framework/mq/redis/core/interceptor/RedisMessageInterceptor.java
public interface RedisMessageInterceptor {

    default void sendMessageBefore(AbstractRedisMessage message) {}
    default void sendMessageAfter(AbstractRedisMessage message) {}
    default void consumeMessageBefore(AbstractRedisMessage message) {}
    default void consumeMessageAfter(AbstractRedisMessage message) {}
}
```

四个方法都是 `default` 空实现，拦截器只需覆盖关心的方法。典型的使用场景：

1. **多租户**：`sendMessageBefore` 写入租户 ID，`consumeMessageBefore` 读取并设置上下文
2. **日志记录**：`sendMessageBefore` 记录发送日志，`consumeMessageAfter` 记录消费日志
3. **链路追踪**：注入 traceId 到消息 headers

## 自动配置：如何与 Spring Boot 集成

### Producer 配置

```java
// 源码位置：cn/iocoder/yudao/framework/mq/redis/config/YudaoRedisMQProducerAutoConfiguration.java
@AutoConfiguration(after = YudaoRedisAutoConfiguration.class)
public class YudaoRedisMQProducerAutoConfiguration {

    @Bean
    public RedisMQTemplate redisMQTemplate(StringRedisTemplate redisTemplate,
                                           List<RedisMessageInterceptor> interceptors) {
        RedisMQTemplate redisMQTemplate = new RedisMQTemplate(redisTemplate);
        interceptors.forEach(redisMQTemplate::addInterceptor);
        return redisMQTemplate;
    }
}
```

Spring Boot 会自动收集所有 `RedisMessageInterceptor` Bean，注入到 `RedisMQTemplate` 中。

### Consumer 配置

```java
// 源码位置：cn/iocoder/yudao/framework/mq/redis/config/YudaoRedisMQConsumerAutoConfiguration.java
@AutoConfiguration(after = YudaoRedisAutoConfiguration.class)
@EnableScheduling // 启用定时任务
public class YudaoRedisMQConsumerAutoConfiguration {

    @Bean
    @ConditionalOnBean(AbstractRedisChannelMessageListener.class)
    public RedisMessageListenerContainer redisMessageListenerContainer(...) {
        // 只有存在 Channel 监听器时才创建 Pub/Sub 容器
    }

    @Bean
    @ConditionalOnBean(AbstractRedisStreamMessageListener.class)
    public StreamMessageListenerContainer redisStreamMessageListenerContainer(...) {
        // 只有存在 Stream 监听器时才创建 Stream 容器
    }

    @Bean
    @ConditionalOnBean(AbstractRedisStreamMessageListener.class)
    public RedisPendingMessageResendJob redisPendingMessageResendJob(...) {
        // 只有存在 Stream 监听器时才启动重发任务
    }

    @Bean
    @ConditionalOnBean(AbstractRedisStreamMessageListener.class)
    public RedisStreamMessageCleanupJob redisStreamMessageCleanupJob(...) {
        // 只有存在 Stream 监听器时才启动清理任务
    }
}
```

`@ConditionalOnBean` 确保按需加载——如果项目没有定义任何 Stream 监听器，就不会创建 Stream 容器和后台任务，节省资源。

## 思考题

1. **为什么 `AbstractRedisChannelMessage.getChannel()` 用 `@JsonIgnore`？** 提示：考虑消息序列化后发送到 Redis 的场景。

2. **`RedisPendingMessageResendJob` 的重发策略是"重新投递 + ACK 原消息"，这会导致消息顺序变化吗？有什么影响？**

3. **如果要实现消息的幂等消费（同一条消息不重复处理），应该在哪里实现？消费者基类还是业务层？**
