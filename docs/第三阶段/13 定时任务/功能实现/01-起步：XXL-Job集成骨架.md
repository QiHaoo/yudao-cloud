# 01-起步：XXL-Job 集成骨架

## 回顾：功能设计的核心结论

在功能设计部分，我们明确了定时任务组件的核心目标：

1. **集成 XXL-Job**：将 XXL-Job 的执行器以 Spring Boot Starter 方式集成，实现自动配置
2. **配置属性化**：通过 `XxlJobProperties` 将 YAML 配置映射为 Java 对象，支持参数校验
3. **条件装配**：通过 `@ConditionalOnClass` 和 `@ConditionalOnProperty` 实现灵活的开关控制
4. **执行器自动注册**：业务模块只需引入依赖和配置，无需手动创建执行器 Bean

本章我们从零开始，一步步实现这个集成骨架。

## 第一步：创建 Maven 模块

首先创建 `yudao-spring-boot-starter-job` 模块，作为定时任务的 Spring Boot Starter。

### 模块目录结构

```
yudao-framework/yudao-spring-boot-starter-job/
├── pom.xml
└── src/main/java/cn/iocoder/yudao/framework/quartz/config/
    ├── XxlJobProperties.java
    ├── YudaoXxlJobAutoConfiguration.java
    └── YudaoAsyncAutoConfiguration.java
└── src/main/resources/META-INF/spring/
    └── org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

### pom.xml 配置

> 源文件路径：`yudao-framework/yudao-spring-boot-starter-job/pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <groupId>cn.iocoder.cloud</groupId>
        <artifactId>yudao-framework</artifactId>
        <version>${revision}</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>
    <artifactId>yudao-spring-boot-starter-job</artifactId>
    <packaging>jar</packaging>

    <name>${project.artifactId}</name>
    <description>任务拓展，基于 XXL-Job 实现</description>
    <url>https://github.com/YunaiV/ruoyi-vue-pro</url>

    <dependencies>
        <!-- 基础依赖 -->
        <dependency>
            <groupId>cn.iocoder.cloud</groupId>
            <artifactId>yudao-common</artifactId>
        </dependency>

        <!-- Spring 核心 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-configuration-processor</artifactId>
            <optional>true</optional>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- XXL-Job 核心 -->
        <dependency>
            <groupId>com.xuxueli</groupId>
            <artifactId>xxl-job-core</artifactId>
        </dependency>

        <!-- 参数校验 -->
        <dependency>
            <groupId>jakarta.validation</groupId>
            <artifactId>jakarta.validation-api</artifactId>
        </dependency>
    </dependencies>

</project>
```

**关键依赖说明**：

| 依赖 | 作用 |
|------|------|
| `yudao-common` | 框架基础模块，提供通用工具类 |
| `spring-boot-configuration-processor` | 生成配置元数据，IDE 可以自动提示 YAML 配置 |
| `spring-boot-starter` | Spring Boot 基础能力 |
| `xxl-job-core` | XXL-Job 客户端核心库 |
| `jakarta.validation-api` | 参数校验注解 |

## 第二步：实现配置属性类

配置属性类负责将 YAML 配置映射为结构化的 Java 对象，并支持参数校验。

### XxlJobProperties

> 源文件路径：`yudao-framework/yudao-spring-boot-starter-job/src/main/java/cn/iocoder/yudao/framework/quartz/config/XxlJobProperties.java`

```java
package cn.iocoder.yudao.framework.quartz.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

import javax.validation.Valid;
import javax.validation.constraints.NotEmpty;
import javax.validation.constraints.NotNull;

/**
 * XXL-Job 配置类
 */
@ConfigurationProperties("xxl.job")
@Validated
@Data
public class XxlJobProperties {

    /**
     * 是否开启，默认为 true
     */
    private Boolean enabled = true;

    /**
     * 访问令牌
     */
    private String accessToken;

    /**
     * 调度器（Admin）配置
     */
    @NotNull(message = "调度器配置不能为空")
    private AdminProperties admin;

    /**
     * 执行器配置
     */
    @NotNull(message = "执行器配置不能为空")
    private ExecutorProperties executor;

    /**
     * XXL-Job 调度器配置类
     */
    @Data
    @Valid
    public static class AdminProperties {

        /**
         * 调度器地址
         */
        @NotEmpty(message = "调度器地址不能为空")
        private String addresses;

    }

    /**
     * XXL-Job 执行器配置类
     */
    @Data
    @Valid
    public static class ExecutorProperties {

        /**
         * 默认端口，-1 表示随机端口
         */
        private static final Integer PORT_DEFAULT = -1;

        /**
         * 默认日志保留天数，-1 表示永久保留
         */
        private static final Integer LOG_RETENTION_DAYS_DEFAULT = 30;

        /**
         * 应用名（执行器名称）
         */
        @NotEmpty(message = "应用名不能为空")
        private String appName;

        /**
         * 执行器 IP（可选，默认自动获取）
         */
        private String ip;

        /**
         * 执行器端口，默认 -1 随机
         */
        private Integer port = PORT_DEFAULT;

        /**
         * 日志存储路径
         */
        @NotEmpty(message = "日志路径不能为空")
        private String logPath;

        /**
         * 日志保留天数，默认 30 天
         */
        private Integer logRetentionDays = LOG_RETENTION_DAYS_DEFAULT;

    }

}
```

**设计要点**：

1. **`@ConfigurationProperties("xxl.job")`**：绑定 `xxl.job` 前缀的 YAML 配置
2. **`@Validated` + JSR-303 注解**：启动时自动校验配置，缺失必填项会立即报错
3. **嵌套静态类**：`AdminProperties` 和 `ExecutorProperties` 分别管理调度器和执行器的配置
4. **默认值设计**：`enabled` 默认 `true`、`port` 默认 `-1`（随机）、`logRetentionDays` 默认 `30` 天

**对应的 YAML 配置**：

```yaml
xxl:
  job:
    enabled: true
    access-token: default_token
    admin:
      addresses: http://127.0.0.1:9090/xxl-job-admin
    executor:
      appname: ${spring.application.name}
      ip: ""  # 可选，留空自动获取
      port: -1  # 随机端口
      log-path: ${user.home}/logs/xxl-job/${spring.application.name}
      log-retention-days: 30
```

## 第三步：实现自动配置类

自动配置类是整个 Starter 的核心，负责读取配置并创建 XXL-Job 执行器 Bean。

### YudaoXxlJobAutoConfiguration

> 源文件路径：`yudao-framework/yudao-spring-boot-starter-job/src/main/java/cn/iocoder/yudao/framework/quartz/config/YudaoXxlJobAutoConfiguration.java`

```java
package cn.iocoder.yudao.framework.quartz.config;

import com.xxl.job.core.executor.XxlJobExecutor;
import com.xxl.job.core.executor.impl.XxlJobSpringExecutor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.annotation.EnableScheduling;

/**
 * XXL-Job 自动配置类
 *
 * @author 芋道源码
 */
@AutoConfiguration
@ConditionalOnClass(XxlJobSpringExecutor.class)
@ConditionalOnProperty(prefix = "xxl.job", name = "enabled",
                       havingValue = "true", matchIfMissing = true)
@EnableConfigurationProperties({XxlJobProperties.class})
@EnableScheduling
@Slf4j
public class YudaoXxlJobAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public XxlJobExecutor xxlJobExecutor(XxlJobProperties properties) {
        log.info("[xxlJobExecutor][初始化 XXL-Job 执行器的配置]");
        XxlJobProperties.AdminProperties admin = properties.getAdmin();
        XxlJobProperties.ExecutorProperties executor = properties.getExecutor();

        // 创建 XXL-Job Spring 执行器
        XxlJobExecutor xxlJobExecutor = new XxlJobSpringExecutor();
        xxlJobExecutor.setIp(executor.getIp());
        xxlJobExecutor.setPort(executor.getPort());
        xxlJobExecutor.setAppname(executor.getAppName());
        xxlJobExecutor.setLogPath(executor.getLogPath());
        xxlJobExecutor.setLogRetentionDays(executor.getLogRetentionDays());
        xxlJobExecutor.setAdminAddresses(admin.getAddresses());
        xxlJobExecutor.setAccessToken(properties.getAccessToken());
        return xxlJobExecutor;
    }

}
```

**注解详解**：

| 注解 | 作用 | 设计意图 |
|------|------|---------|
| `@AutoConfiguration` | 标记为 Spring Boot 自动配置类 | 2.7+ 替代 `@Configuration` |
| `@ConditionalOnClass(XxlJobSpringExecutor.class)` | classpath 存在 XXL-Job 才生效 | 未引入依赖时自动跳过 |
| `@ConditionalOnProperty(...)` | 配置 `xxl.job.enabled=true` 才生效 | 允许通过配置禁用 |
| `@EnableConfigurationProperties` | 注册配置属性类 | 自动绑定 YAML 配置 |
| `@EnableScheduling` | 开启 Spring 定时任务 | 让 `@Scheduled` 和 `@XxlJob` 共存 |
| `@ConditionalOnMissingBean` | 容器中无同类型 Bean 才创建 | 允许用户自定义覆盖 |

**执行器初始化流程**：

```
应用启动
  │
  ├─ 1. Spring Boot 扫描 AutoConfiguration.imports
  │     └─ 发现 YudaoXxlJobAutoConfiguration
  │
  ├─ 2. 检查条件注解
  │     ├─ @ConditionalOnClass：XxlJobSpringExecutor 在 classpath 中？ ✓
  │     └─ @ConditionalOnProperty：xxl.job.enabled=true？ ✓
  │
  ├─ 3. 绑定配置
  │     └─ 读取 YAML → 映射到 XxlJobProperties 对象
  │
  ├─ 4. 创建执行器 Bean
  │     └─ xxlJobExecutor() 方法执行
  │           ├─ new XxlJobSpringExecutor()
  │           ├─ 设置 Admin 地址、AccessToken
  │           └─ 设置 Executor 配置（appName、port、logPath 等）
  │
  └─ 5. XxlJobSpringExecutor 初始化
        ├─ 启动内嵌 Server（Netty）
        ├─ 向 Admin 注册执行器
        └─ 开始接收调度请求
```

## 第四步：注册自动配置

创建 `AutoConfiguration.imports` 文件，让 Spring Boot 能发现我们的自动配置类。

### AutoConfiguration.imports

> 源文件路径：`yudao-framework/yudao-spring-boot-starter-job/src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

```
cn.iocoder.yudao.framework.quartz.config.YudaoXxlJobAutoConfiguration
cn.iocoder.yudao.framework.quartz.config.YudaoAsyncAutoConfiguration
```

**说明**：这里同时注册了异步任务配置 `YudaoAsyncAutoConfiguration`，它负责为 Spring 的线程池添加 TTL（TransmittableThreadLocal）支持，确保异步任务能正确传递租户上下文。

## 第五步：业务模块使用

当业务模块（如订单服务）需要使用定时任务时，只需两步：

### 1. 引入依赖

```xml
<dependency>
    <groupId>cn.iocoder.cloud</groupId>
    <artifactId>yudao-spring-boot-starter-job</artifactId>
</dependency>
```

### 2. 配置 YAML

```yaml
# application.yaml
xxl:
  job:
    admin:
      addresses: http://127.0.0.1:9090/xxl-job-admin
    executor:
      appname: trade-service
      log-path: /home/work/logs/xxl-job/trade-service
    access-token: default_token
```

### 3. 编写 JobHandler

```java
@Component
public class TradeOrderAutoCancelJob {

    @Resource
    private TradeOrderUpdateService tradeOrderUpdateService;

    /**
     * 自动取消超时未支付的订单
     * 每小时执行一次：0 0 * * * ?
     */
    @XxlJob("tradeOrderAutoCancelJob")
    public void execute() {
        int count = tradeOrderUpdateService.cancelTimeoutOrders();
        XxlJobHelper.handleSuccess(String.format("取消超时订单 %d 个", count));
    }
}
```

就这么简单。不需要手动创建执行器 Bean，不需要手动注册 Handler，框架会自动完成所有初始化工作。

## 异步任务配置（补充）

在同一个 Starter 中，还包含异步任务的配置，用于确保异步执行时能正确传递上下文（如租户信息）。

### YudaoAsyncAutoConfiguration

> 源文件路径：`yudao-framework/yudao-spring-boot-starter-job/src/main/java/cn/iocoder/yudao/framework/quartz/config/YudaoAsyncAutoConfiguration.java`

```java
package cn.iocoder.yudao.framework.quartz.config;

import com.alibaba.ttl.TtlRunnable;
import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.core.task.SimpleAsyncTaskExecutor;

/**
 * 异步任务 Configuration
 */
@AutoConfiguration
@EnableAsync
public class YudaoAsyncAutoConfiguration {

    @Bean
    public BeanPostProcessor threadPoolTaskExecutorBeanPostProcessor() {
        return new BeanPostProcessor() {

            @Override
            public Object postProcessBeforeInitialization(Object bean, String beanName)
                    throws BeansException {
                // 处理 ThreadPoolTaskExecutor
                if (bean instanceof ThreadPoolTaskExecutor) {
                    ThreadPoolTaskExecutor executor = (ThreadPoolTaskExecutor) bean;
                    executor.setTaskDecorator(TtlRunnable::get);
                    return executor;
                }
                // 处理 SimpleAsyncTaskExecutor
                if (bean instanceof SimpleAsyncTaskExecutor) {
                    SimpleAsyncTaskExecutor executor = (SimpleAsyncTaskExecutor) bean;
                    executor.setTaskDecorator(TtlRunnable::get);
                    return executor;
                }
                return bean;
            }

        };
    }

}
```

**设计意图**：

- 通过 `BeanPostProcessor` 拦截所有线程池 Bean 的创建
- 为每个线程池设置 `TtlRunnable` 装饰器
- 确保异步任务执行时，`TransmittableThreadLocal` 中的租户上下文能正确传递

## 总结

本章我们完成了 XXL-Job 集成的骨架搭建：

| 步骤 | 文件 | 作用 |
|------|------|------|
| 1 | `pom.xml` | 引入 `xxl-job-core` 依赖 |
| 2 | `XxlJobProperties` | YAML 配置映射 + 参数校验 |
| 3 | `YudaoXxlJobAutoConfiguration` | 自动创建执行器 Bean |
| 4 | `AutoConfiguration.imports` | 注册自动配置类 |
| 5 | `YudaoAsyncAutoConfiguration` | 异步任务 TTL 支持 |

**核心设计思想**：

1. **约定优于配置**：默认开启、默认端口随机、默认日志保留 30 天
2. **条件装配**：通过 `@ConditionalOnClass` 和 `@ConditionalOnProperty` 实现灵活控制
3. **开闭原则**：`@ConditionalOnMissingBean` 允许用户自定义覆盖
4. **配置属性化**：所有配置都有类型安全的 Java 对象，IDE 可以自动提示

下一章我们将实现多租户任务支持，让定时任务能在多租户环境下正确执行。
