# 02-RedisMQTemplate 发送实现

> 本章对应功能设计：[03-核心设计：Redis MQ 实现](../功能设计/03-核心设计：Redis%20MQ实现.md) 中的"RedisMQTemplate：统一发送 API"部分。

## 本章目标

实现 `RedisMQTemplate`——整个 Redis MQ 框架的核心发送入口。完成本章后，你将理解：

1. 如何通过方法重载实现"一个 `send` 方法，两种发送模式"
2. 拦截器链的执行顺序和 try-finally 保障
3. 自动配置如何将拦截器注入到模板中

## 第一步：RedisMQTemplate 核心实现

`RedisMQTemplate` 是整个 Redis MQ 的核心入口，它提供了两个 `send` 方法，分别对应 Pub/Sub 和 Stream 两种消息模式。

```java
// 源码位置：yudao-framework/yudao-spring-boot-starter-mq/src/main/java/cn/iocoder/yudao/framework/mq/redis/core/RedisMQTemplate.java
@AllArgsConstructor
public class RedisMQTemplate {

    @Getter
    private final RedisTemplate<String, ?> redisTemplate;
    /**
     * 拦截器数组
     */
    @Getter
    private final List<RedisMessageInterceptor> interceptors = new ArrayList<>();

    /**
     * 发送 Redis 消息，基于 Redis pub/sub 实现
     *
     * @param message 消息
     */
    public <T extends AbstractRedisChannelMessage> void send(T message) {
        try {
            sendMessageBefore(message);
            // 发送消息
            redisTemplate.convertAndSend(message.getChannel(), JsonUtils.toJsonString(message));
        } finally {
            sendMessageAfter(message);
        }
    }

    /**
     * 发送 Redis 消息，基于 Redis Stream 实现
     *
     * @param message 消息
     * @return 消息记录的编号对象
     */
    public <T extends AbstractRedisStreamMessage> RecordId send(T message) {
        try {
            sendMessageBefore(message);
            // 发送消息
            return redisTemplate.opsForStream().add(StreamRecords.newRecord()
                    .ofObject(JsonUtils.toJsonString(message)) // 设置内容
                    .withStreamKey(message.getStreamKey())); // 设置 stream key
        } finally {
            sendMessageAfter(message);
        }
    }

    /**
     * 添加拦截器
     *
     * @param interceptor 拦截器
     */
    public void addInterceptor(RedisMessageInterceptor interceptor) {
        interceptors.add(interceptor);
    }

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

}
```

## 第二步：理解两个 send 方法

两个 `send` 方法通过 Java 的方法重载（Overload）实现多态。编译器根据参数类型自动选择正确的重载版本：

```java
// 发送广播消息 → 调用 send(AbstractRedisChannelMessage) → PUBLISH 命令
redisMQTemplate.send(new RedisWebSocketMessage().setUserId(1L));

// 发送可靠消息 → 调用 send(AbstractRedisStreamMessage) → XADD 命令
redisMQTemplate.send(new OrderCreatedMessage().setOrderId(100L));
```

### 两个 send 方法的对比

| 维度 | send(ChannelMessage) | send(StreamMessage) |
|------|---------------------|---------------------|
| 底层命令 | `PUBLISH` | `XADD` |
| 返回值 | void | RecordId（消息 ID） |
| 持久化 | 不持久化 | 持久化到 AOF/RDB |
| 消费模式 | 广播（所有订阅者收到） | 集群（组内一个消费者收到） |
| 适用场景 | 实时通知、WebSocket 广播 | 订单处理、库存扣减 |

### Pub/Sub 发送细节

```java
redisTemplate.convertAndSend(message.getChannel(), JsonUtils.toJsonString(message));
```

- `message.getChannel()` 获取频道名（默认是类名）
- `JsonUtils.toJsonString(message)` 将消息序列化为 JSON 字符串
- `convertAndSend` 底层执行 Redis 的 `PUBLISH` 命令

### Stream 发送细节

```java
return redisTemplate.opsForStream().add(StreamRecords.newRecord()
        .ofObject(JsonUtils.toJsonString(message))
        .withStreamKey(message.getStreamKey()));
```

- `StreamRecords.newRecord()` 创建一条 Stream 记录
- `.ofObject(...)` 设置记录内容（JSON 字符串）
- `.withStreamKey(...)` 设置 Stream 的 key（默认是类名）
- 返回 `RecordId`，包含时间戳和序列号（如 `1685184000000-0`）

## 第三步：拦截器链的执行顺序

发送和消费都遵循"洋葱模型"——Before 正序、After 倒序：

```
假设拦截器列表为 [拦截器A, 拦截器B, 拦截器C]

发送流程：
  拦截器A.sendMessageBefore()   ← 最先执行
    拦截器B.sendMessageBefore()
      拦截器C.sendMessageBefore()
        ↓
      实际发送消息（PUBLISH 或 XADD）
        ↓
      拦截器C.sendMessageAfter()  ← 最先执行（倒序）
    拦截器B.sendMessageAfter()
  拦截器A.sendMessageAfter()     ← 最后执行
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

**为什么 Before 正序、After 倒序？**

这保证了"谁先设置的，谁最后清理"。比如拦截器 A 设置了租户上下文，拦截器 B 设置了链路追踪上下文，那么清理时 B 先清理、A 后清理，不会互相干扰。

### try-finally 保障

```java
public <T extends AbstractRedisChannelMessage> void send(T message) {
    try {
        sendMessageBefore(message);
        redisTemplate.convertAndSend(message.getChannel(), JsonUtils.toJsonString(message));
    } finally {
        sendMessageAfter(message); // 即使发送失败也会执行
    }
}
```

`try-finally` 确保 `sendMessageAfter` 一定执行，即使发送过程中抛异常。这对于清理上下文（如租户信息）非常重要——如果 `sendMessageBefore` 设置了某些上下文但发送失败了，`sendMessageAfter` 必须清理这些上下文，否则会"泄漏"到后续请求。

## 第四步：自动配置——Producer 端

`RedisMQTemplate` 的创建和拦截器注入由自动配置类完成：

```java
// 源码位置：yudao-framework/yudao-spring-boot-starter-mq/src/main/java/cn/iocoder/yudao/framework/mq/redis/config/YudaoRedisMQProducerAutoConfiguration.java
@Slf4j
@AutoConfiguration(after = YudaoRedisAutoConfiguration.class)
public class YudaoRedisMQProducerAutoConfiguration {

    @Bean
    public RedisMQTemplate redisMQTemplate(StringRedisTemplate redisTemplate,
                                           List<RedisMessageInterceptor> interceptors) {
        RedisMQTemplate redisMQTemplate = new RedisMQTemplate(redisTemplate);
        // 添加拦截器
        interceptors.forEach(redisMQTemplate::addInterceptor);
        return redisMQTemplate;
    }

}
```

**关键设计点：**

1. **`@AutoConfiguration(after = YudaoRedisAutoConfiguration.class)`**：确保 Redis 基础配置先加载，`StringRedisTemplate` Bean 已经存在。

2. **`List<RedisMessageInterceptor> interceptors`**：Spring 会自动收集容器中所有 `RedisMessageInterceptor` 类型的 Bean，注入到这个列表中。要新增一个拦截器，只需要实现接口并加上 `@Component`，无需修改任何配置。

3. **`StringRedisTemplate` 而不是 `RedisTemplate<String, Object>`**：MQ 消息统一使用 JSON 字符串序列化，`StringRedisTemplate` 更合适。

### 自动配置注册

在 Spring Boot 2.7+ 中，自动配置通过 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件注册：

```
# 文件位置：yudao-spring-boot-starter-mq/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
cn.iocoder.yudao.framework.mq.redis.config.YudaoRedisMQProducerAutoConfiguration
cn.iocoder.yudao.framework.mq.redis.config.YudaoRedisMQConsumerAutoConfiguration
cn.iocoder.yudao.framework.mq.rabbitmq.config.YudaoRabbitMQAutoConfiguration
```

引入 `yudao-spring-boot-starter-mq` 依赖后，Spring Boot 自动加载这些配置类，无需手动 `@Import`。

## 实际使用示例：WebSocket 消息广播

来看一个完整的使用案例。`RedisWebSocketMessageSender` 使用 `RedisMQTemplate` 发送 WebSocket 广播消息：

```java
// 源码位置：yudao-framework/yudao-spring-boot-starter-websocket/src/main/java/cn/iocoder/yudao/framework/websocket/core/sender/redis/RedisWebSocketMessageSender.java
@Slf4j
public class RedisWebSocketMessageSender extends AbstractWebSocketMessageSender {

    private final RedisMQTemplate redisMQTemplate;

    public RedisWebSocketMessageSender(WebSocketSessionManager sessionManager,
                                       RedisMQTemplate redisMQTemplate) {
        super(sessionManager);
        this.redisMQTemplate = redisMQTemplate;
    }

    @Override
    public void send(Integer userType, Long userId, String messageType, String messageContent) {
        sendRedisMessage(null, userId, userType, messageType, messageContent);
    }

    @Override
    public void send(Integer userType, String messageType, String messageContent) {
        sendRedisMessage(null, null, userType, messageType, messageContent);
    }

    @Override
    public void send(String sessionId, String messageType, String messageContent) {
        sendRedisMessage(sessionId, null, null, messageType, messageContent);
    }

    /**
     * 通过 Redis 广播消息
     */
    private void sendRedisMessage(String sessionId, Long userId, Integer userType,
                                  String messageType, String messageContent) {
        RedisWebSocketMessage mqMessage = new RedisWebSocketMessage()
                .setSessionId(sessionId).setUserId(userId).setUserType(userType)
                .setMessageType(messageType).setMessageContent(messageContent);
        redisMQTemplate.send(mqMessage);
    }

}
```

消息定义：

```java
// 源码位置：yudao-framework/yudao-spring-boot-starter-websocket/src/main/java/cn/iocoder/yudao/framework/websocket/core/sender/redis/RedisWebSocketMessage.java
@Data
public class RedisWebSocketMessage extends AbstractRedisChannelMessage {
    private String sessionId;
    private Integer userType;
    private Long userId;
    private String messageType;
    private String messageContent;
}
```

**为什么 WebSocket 用 Channel 消息（广播）而不是 Stream 消息（集群）？**

WebSocket 连接分散在各个节点上。用户 A 的连接在节点 1，用户 B 的连接在节点 2。当需要给用户 A 发送消息时，所有节点都需要收到这条消息，然后各自检查"这个 session 是否在本节点上"。这正是广播模式的典型场景。

## 本章小结

| 要点 | 说明 |
|------|------|
| 两个 send 方法 | 通过方法重载，根据参数类型自动选择 Pub/Sub 或 Stream |
| 拦截器链 | Before 正序、After 倒序，洋葱模型 |
| try-finally | 确保 After 一定执行，防止上下文泄漏 |
| 自动配置 | Spring 自动收集所有拦截器 Bean，注入到 RedisMQTemplate |
| JSON 序列化 | 统一使用 `JsonUtils.toJsonString()` 序列化消息 |
