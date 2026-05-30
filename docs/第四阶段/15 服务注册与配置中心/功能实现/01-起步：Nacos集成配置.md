# 01 - 起步：Nacos 集成配置

> 本章是功能实现的起点。完成本章后，你的微服务将能够连接 Nacos Server，实现服务注册与配置拉取。

## 前置准备

在开始之前，请确保：

1. 已完成第一阶段的项目骨架搭建（Maven 多模块结构）
2. 已完成第四阶段组件 15 功能设计的阅读
3. 本地已安装 Nacos Server（开发环境推荐使用 Docker 或单机模式启动）

启动 Nacos Server（单机模式）：

```bash
# Docker 方式
docker run -d --name nacos -p 8848:8848 -p 9848:9848 -e MODE=standalone nacos/nacos-server:v2.2.3

# 或者下载解压后，进入 bin 目录
startup.cmd -m standalone  # Windows
startup.sh -m standalone   # Linux/Mac
```

启动后访问 `http://127.0.0.1:8848/nacos`，默认账号密码为 `nacos/nacos`。

## 第一步：BOM 版本管理

在 `yudao-dependencies/pom.xml` 中，Spring Cloud Alibaba 的版本已经统一管理。确认以下依赖存在：

```xml
<!-- yudao-dependencies/pom.xml -->
<properties>
    <spring.cloud.alibaba.version>2021.0.6.2</spring.cloud.alibaba.version>
</properties>

<dependencyManagement>
    <dependencies>
        <!-- Spring Cloud Alibaba BOM -->
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

**为什么在 BOM 中管理版本？**

这是 Maven 多模块项目的标准做法。所有模块共用同一个版本号，避免版本不一致导致的依赖冲突。当需要升级 Nacos 客户端版本时，只需修改一处。

## 第二步：微服务引入 Nacos 依赖

以 `system-server` 为例，在其 `pom.xml` 中引入 Nacos 相关依赖：

```xml
<!-- yudao-module-system/yudao-module-system-server/pom.xml -->
<dependencies>
    <!-- Nacos 服务发现 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!-- Nacos 配置中心 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>

    <!-- Spring Cloud Bootstrap（Nacos 配置中心需要） -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bootstrap</artifactId>
    </dependency>
</dependencies>
```

**三个依赖的职责：**

| 依赖 | 职责 | 核心类 |
|------|------|--------|
| `nacos-discovery` | 服务注册与发现 | `NacosDiscoveryProperties`、`NacosServiceDiscovery` |
| `nacos-config` | 配置中心客户端 | `NacosConfigProperties`、`NacosPropertySourceLocator` |
| `spring-cloud-starter-bootstrap` | 支持 bootstrap 上下文 | `BootstrapApplicationListener` |

## 第三步：主配置文件 application.yaml

配置 `spring.config.import`，这是 Spring Boot 2.4+ 推荐的配置加载方式：

```yaml
# yudao-module-system/yudao-module-system-server/src/main/resources/application.yaml
spring:
  application:
    name: system-server

  profiles:
    active: local

  config:
    import:
      - optional:classpath:application-${spring.profiles.active}.yaml  # 加载本地配置
      - optional:nacos:${spring.application.name}-${spring.profiles.active}.yaml  # 加载 Nacos 配置
```

**配置解析：**

- `spring.application.name`：服务名称，Nacos 注册和配置拉取都依赖此名称
- `spring.profiles.active`：激活的 profile，决定加载哪个环境的配置
- `spring.config.import`：
  - `optional:classpath:application-${spring.profiles.active}.yaml`：加载本地的环境配置文件
  - `optional:nacos:${spring.application.name}-${spring.profiles.active}.yaml`：从 Nacos 加载远程配置

**`optional:` 前缀的作用：**

如果 Nacos Server 不可用，使用 `optional:` 前缀后，服务不会启动失败，而是降级到本地配置。这在本地开发时非常有用——你不需要强制启动 Nacos。

## 第四步：环境配置文件 application-local.yaml

创建本地环境的 Nacos 连接配置：

```yaml
# yudao-module-system/yudao-module-system-server/src/main/resources/application-local.yaml
--- #################### 注册中心 + 配置中心相关配置 ####################

spring:
  cloud:
    nacos:
      server-addr: 127.0.0.1:8848  # Nacos 服务器地址
      username: nacos               # Nacos 账号（默认）
      password: nacos-admin         # Nacos 密码（默认）
      discovery:                    # 【服务发现】配置项
        namespace: dev              # 命名空间，用于环境隔离
        group: DEFAULT_GROUP        # 分组
        metadata:
          version: 1.0.0            # 服务实例版本号，可用于灰度发布
      config:                       # 【配置中心】配置项
        namespace: dev              # 命名空间
        group: DEFAULT_GROUP        # 分组
```

**配置项详解：**

| 配置项 | 作用 | 常用值 |
|--------|------|--------|
| `server-addr` | Nacos 服务器地址，集群用逗号分隔 | `127.0.0.1:8848` |
| `username` / `password` | Nacos 认证信息 | 生产环境必须修改 |
| `discovery.namespace` | 服务发现的命名空间 | `dev` / `test` / `prod` |
| `discovery.group` | 服务发现的分组 | `DEFAULT_GROUP` |
| `discovery.metadata` | 服务实例的元数据 | 可自定义，用于灰度发布 |
| `config.namespace` | 配置中心的命名空间 | 与 discovery 保持一致 |
| `config.group` | 配置中心的分组 | `DEFAULT_GROUP` |

**命名空间（namespace）的使用方式：**

在 Nacos 控制台创建命名空间时，需要填写的"命名空间ID"可以是：
- 自定义字符串：如 `dev`、`test`、`prod`
- UUID 格式：如 `a1b2c3d4-e5f6-7890-abcd-ef1234567890`

yudao-cloud 使用自定义字符串，更易读。

## 第五步：单体模式的配置（对照理解）

为了对照理解，看一下单体模式下是如何禁用 Nacos 的：

```yaml
# yudao-server/src/main/resources/application.yaml
spring:
  cloud:
    nacos:
      discovery:
        enabled: false  # 禁用服务发现
      config:
        enabled: false  # 禁用配置中心
```

**设计精妙之处：**

同一套业务代码（如 system 模块），既可以在单体模式下运行（禁用 Nacos，本地配置），也可以在微服务模式下运行（启用 Nacos，远程配置）。切换方式只需修改 `spring.profiles.active` 和 Nacos 相关配置。

## 第六步：验证配置

### 6.1 启动 Nacos Server

确保 Nacos Server 已启动，访问 `http://127.0.0.1:8848/nacos` 能看到控制台。

### 6.2 启动 system-server

```bash
# 在 system-server 模块目录下
mvn spring-boot:run
```

### 6.3 验证服务注册

在 Nacos 控制台的"服务管理 -> 服务列表"中，应该能看到 `system-server` 已经注册。

### 6.4 验证配置拉取

在 Nacos 控制台的"配置管理 -> 配置列表"中，可以创建一个测试配置：

- Data ID: `system-server-local.yaml`
- Group: `DEFAULT_GROUP`
- 格式: YAML
- 内容: `test.config: hello-nacos`

然后在代码中验证配置是否生效：

```java
@RestController
@RequestMapping("/test")
public class TestController {

    @Value("${test.config:default}")
    private String testConfig;

    @GetMapping("/config")
    public String testConfig() {
        return testConfig;  // 应返回 "hello-nacos"
    }
}
```

## 配置加载优先级

理解配置加载的优先级，有助于排查"配置不生效"的问题：

```
优先级从高到低：

1. Nacos 上的 ${service-name}-${profile}.yaml    ← 最高优先级，可覆盖本地
2. Nacos 上的 shared-configs（共享配置）
3. 本地 application-${profile}.yaml
4. 本地 application.yaml
5. 代码中的 @Value 默认值                        ← 最低优先级
```

**实际应用场景：**

- 本地 `application-local.yaml` 配置了数据库连接为 `localhost:3306`
- Nacos 上的 `system-server-local.yaml` 配置了数据库连接为 `10.0.0.10:3306`
- 最终生效的是 Nacos 上的配置（优先级更高）

这意味着：本地配置做默认值，Nacos 配置做覆盖。开发时用本地配置，部署时用 Nacos 配置。

## 常见问题排查

### 问题一：服务注册失败

**现象**：Nacos 控制台看不到服务

**排查步骤**：
1. 检查 `server-addr` 是否正确
2. 检查 Nacos Server 是否启动
3. 检查网络连通性：`telnet 127.0.0.1 8848`
4. 检查 `spring.application.name` 是否配置

### 问题二：配置未拉取

**现象**：`@Value` 注入的是默认值，而不是 Nacos 上的配置

**排查步骤**：
1. 检查 Data ID 是否正确：`${spring.application.name}-${spring.profiles.active}.yaml`
2. 检查 Group 是否匹配
3. 检查 namespace 是否匹配
4. 检查配置格式是否为 YAML

### 问题三：启动报错 Nacos 连接失败

**现象**：启动时抛出连接异常

**解决方案**：
- 确保使用了 `optional:nacos:` 前缀（而不是 `nacos:`）
- 如果不需要 Nacos，可以在 `application.yaml` 中禁用：
  ```yaml
  spring:
    cloud:
      nacos:
        discovery:
          enabled: false
        config:
          enabled: false
  ```

## 本章小结

本章完成了 Nacos 集成的基础配置：

1. **BOM 版本管理**：在 `yudao-dependencies` 中统一管理 Spring Cloud Alibaba 版本
2. **依赖引入**：在微服务中引入 `nacos-discovery`、`nacos-config`、`spring-cloud-starter-bootstrap`
3. **配置加载**：使用 `spring.config.import` 方式加载本地配置和 Nacos 配置
4. **环境隔离**：通过 `application-{profile}.yaml` 和 Nacos namespace 实现多环境隔离

下一章将介绍如何设计共享配置和多环境策略，让配置管理更加高效。
