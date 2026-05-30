# 10-核心设计：异步场景上下文传递 TransmittableThreadLocal

## 场景切入

你写了一个接口，查询订单列表后需要异步记录操作日志：

```java
@GetMapping("/list")
@PreAuthorize("@ss.hasPermission('system:order:query')")
public CommonResult<List<OrderRespVO>> getOrderList() {
    List<OrderRespVO> list = orderService.list();
    // 异步记录操作日志
    asyncLogService.recordLog("查询订单列表");
    return success(list);
}

@Async
public void recordLog(String action) {
    Long userId = SecurityFrameworkUtils.getLoginUserId();  // ← 这里会返回 null！
    logMapper.insert(new OperateLogDO(userId, action));
}
```

你发现 `getLoginUserId()` 返回了 `null`——操作日志没有记录到用户信息。为什么？

因为 `@Async` 方法会在**另一个线程**中执行，而 Spring Security 的 `SecurityContextHolder` 默认使用 `ThreadLocal` 存储上下文。新线程中没有这个 ThreadLocal 的值，所以拿不到用户信息。

## ThreadLocal 的局限性

`ThreadLocal` 是 Java 中最常用的线程隔离机制：每个线程有自己的变量副本，互不干扰。Spring Security 用它存储当前请求的认证信息：

```
主线程                          异步线程
  │                              │
  │  SecurityContext             │  SecurityContext
  │  {userId=1, role=admin}      │  null ← 空的！
  │                              │
  ▼                              ▼
  @Async 方法                    getLoginUserId()
  recordLog()                    → null
```

问题的根源：`ThreadLocal` 只在线程内共享，跨线程就丢失了。

## 解决方案：TransmittableThreadLocal

阿里巴巴开源的 [TransmittableThreadLocal](https://github.com/alibaba/transmittable-thread-local)（简称 TTL）解决了这个问题。它在 `ThreadLocal` 的基础上增加了**跨线程传递**的能力：

```
主线程                          异步线程
  │                              │
  │  SecurityContext             │  SecurityContext
  │  {userId=1, role=admin}      │  {userId=1, role=admin} ← 自动传递！
  │                              │
  ▼                              ▼
  @Async 方法                    getLoginUserId()
  recordLog()                    → 1 ✓
```

### 实现原理

TTL 的核心思想是：在线程提交任务时，自动捕获当前线程的 ThreadLocal 值，在新线程执行任务前自动设置，执行完自动恢复。

```
主线程提交异步任务时：
  1. 捕获主线程的所有 TTL 值 → Map<TransmittableThreadLocal, Object>
  2. 将值打包到任务对象中

异步线程执行任务时：
  3. 从任务对象中取出 TTL 值
  4. 设置到异步线程的 TTL 中
  5. 执行任务
  6. 恢复异步线程原来的 TTL 值
```

## 本项目的实现

### TransmittableThreadLocalSecurityContextHolderStrategy

框架用 TTL 替换了 Spring Security 默认的 `ThreadLocal`：

```java
public class TransmittableThreadLocalSecurityContextHolderStrategy
        implements SecurityContextHolderStrategy {

    private static final ThreadLocal<SecurityContext> CONTEXT_HOLDER =
        new TransmittableThreadLocal<>();

    @Override
    public void clearContext() {
        CONTEXT_HOLDER.remove();
    }

    @Override
    public SecurityContext getContext() {
        SecurityContext ctx = CONTEXT_HOLDER.get();
        if (ctx == null) {
            ctx = createEmptyContext();
            CONTEXT_HOLDER.set(ctx);
        }
        return ctx;
    }

    @Override
    public void setContext(SecurityContext context) {
        Assert.notNull(context, "Only non-null SecurityContext instances are permitted");
        CONTEXT_HOLDER.set(context);
    }

    @Override
    public SecurityContext createEmptyContext() {
        return new SecurityContextImpl();
    }
}
```

### 注册方式

在 `YudaoSecurityAutoConfiguration` 中，通过 `MethodInvokingFactoryBean` 调用 `SecurityContextHolder.setStrategyName()`：

```java
@Bean
public MethodInvokingFactoryBean securityContextHolderMethodInvokingFactoryBean() {
    MethodInvokingFactoryBean bean = new MethodInvokingFactoryBean();
    bean.setTargetClass(SecurityContextHolder.class);
    bean.setTargetMethod("setStrategyName");
    bean.setArguments(TransmittableThreadLocalSecurityContextHolderStrategy.class.getName());
    return bean;
}
```

为什么用 `MethodInvokingFactoryBean` 而不是直接调用？因为需要在 Spring 容器初始化时执行，`MethodInvokingFactoryBean` 提供了这个能力。

## 效果对比

### 使用前（默认 ThreadLocal）

```java
@Async
public void asyncTask() {
    // ❌ 返回 null，因为异步线程没有 SecurityContext
    Long userId = SecurityFrameworkUtils.getLoginUserId();
}
```

### 使用后（TransmittableThreadLocal）

```java
@Async
public void asyncTask() {
    // ✅ 返回正确的用户 ID，TTL 自动传递了 SecurityContext
    Long userId = SecurityFrameworkUtils.getLoginUserId();
}
```

## 适用场景

| 场景 | 是否需要 TTL | 说明 |
|------|-------------|------|
| `@Async` 异步方法 | ✅ 需要 | 新线程执行，ThreadLocal 丢失 |
| `CompletableFuture.supplyAsync()` | ✅ 需要 | ForkJoinPool 或自定义线程池 |
| `Thread pool` 提交任务 | ✅ 需要 | 线程池复用线程，上下文可能错乱 |
| 同步方法调用 | ❌ 不需要 | 同一线程，ThreadLocal 正常工作 |
| Servlet 请求处理 | ❌ 不需要 | 每个请求在独立线程中处理 |

## 注意事项

### 1. 线程池需要配合 TTL 使用

如果使用 `ThreadPoolExecutor`，需要包装为 `TtlExecutors.getTtlExecutorService()`，否则 TTL 的自动传递不生效：

```java
// ❌ 普通线程池，TTL 不生效
ExecutorService executor = Executors.newFixedThreadPool(10);

// ✅ TTL 包装后的线程池
ExecutorService executor = TtlExecutors.getTtlExecutorService(
    Executors.newFixedThreadPool(10));
```

Spring 的 `@Async` 默认使用 `SimpleAsyncTaskExecutor`（每次创建新线程），TTL 能正常工作。但如果配置了自定义线程池，需要确保它是 TTL 兼容的。

### 2. SecurityContext 的清理

TTL 会自动传递上下文，但**不会自动清理**。在异步任务执行完后，应该清理 SecurityContext，避免线程池复用线程时上下文泄漏：

```java
@Async
public void asyncTask() {
    try {
        // 业务逻辑
        Long userId = SecurityFrameworkUtils.getLoginUserId();
        // ...
    } finally {
        SecurityContextHolder.clearContext();  // 清理
    }
}
```

### 3. 性能开销

TTL 的每次跨线程传递都需要捕获和恢复 ThreadLocal 值，有一定的性能开销。对于高频调用的异步任务，需要评估这个开销是否可接受。

## 与 InheritableThreadLocal 的对比

Java 原生的 `InheritableThreadLocal` 也能在父子线程间传递值，但它有两个致命缺陷：

| 维度 | InheritableThreadLocal | TransmittableThreadLocal |
|------|----------------------|------------------------|
| 父子线程传递 | ✅ 支持 | ✅ 支持 |
| 线程池复用 | ❌ 不支持（线程复用时值不更新） | ✅ 支持 |
| 提交时捕获 | ❌ 创建时捕获 | ✅ 提交时捕获 |

线程池中的线程会被复用，`InheritableThreadLocal` 只在线程**创建**时从父线程继承值，后续复用时不会更新。而 TTL 在任务**提交**时捕获值，确保每次任务都能拿到最新的上下文。

## 思考题

1. **如果同时有多个异步任务并发执行，它们的 SecurityContext 会互相干扰吗？** 提示：TTL 是线程隔离的。

2. **在微服务架构中，TTL 能跨服务传递 SecurityContext 吗？** 如果不能，需要什么机制？提示：考虑 Feign 调用的 `LoginUserRequestInterceptor`。

3. **Spring Security 5.x 默认的 `SecurityContextHolder` 策略是什么？为什么官方不用 TTL？** 提示：考虑 Spring Security 的设计原则和外部依赖的取舍。

4. **如果项目不引入 TTL，还有什么方式解决异步场景的上下文传递问题？** 比如手动传递参数、使用 `RequestContextHolder` 等。
