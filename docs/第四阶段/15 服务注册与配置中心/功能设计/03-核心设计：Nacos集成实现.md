# 03 - 核心设计：Nacos 集成实现

## Spring Cloud Alibaba Nacos Discovery 集成

### 自动装配原理

Spring Cloud Alibaba Nacos Discovery 通过 Spring Boot 的自动装配机制，实现了服务注册的零代码接入。核心流程：

```
微服务启动
    │
    ▼
NacosServiceAutoConfiguration
    │
    ▼
NacosAutoServiceRegistration
    │
    ├─→ 读取 spring.cloud.nacos.discovery 配置
    ├─→ 构建注册信息（服务名、IP、端口、元数据）
    ├─→ 向 Nacos Server 发送注册请求
    └─→ 启动心跳线程，定期续约
```

### 关键配置项

```yaml
spring:
  cloud:
    nacos:
      server-addr: 127.0.0.1:8848  # Nacos 服务器地址
      username: nacos               # 账号
      password: nacos-admin         # 密码
      discovery:
        namespace: dev              # 命名空间（环境隔离）
        group: DEFAULT_GROUP        # 分组
        metadata:
          version: 1.0.0            # 服务实例的元数据
```

各配置项的作用：

| 配置项 | 作用 | 示例 |
|--------|------|------|
| `server-addr` | Nacos 服务器地址，支持集群（逗号分隔） | `127.0.0.1:8848` |
| `namespace` | 命名空间，用于环境隔离 | `dev` / `test` / `prod` |
| `group` | 分组，用于业务隔离 | `DEFAULT_GROUP` |
| `metadata` | 服务实例的元数据，可用于灰度发布 | `version: 1.0.0` |

### 服务元数据的应用

yudao-cloud 利用服务实例的 `metadata` 实现灰度发布：

```yaml
# system-server 实例 1
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          version: 1.0.0  # 当前版本

# system-server 实例 2（新版本）
spring:
  cloud:
    nacos:
      discovery:
        metadata:
          version: 2.0.0  # 新版本
```

网关根据请求头中的 `version` 字段，路由到对应版本的实例：

```
请求 Header: version=2.0.0
    │
    ▼
Gateway 的 GrayLoadBalancer
    │
    ├─→ 获取所有 system-server 实例
    ├─→ 过滤 metadata.version == 2.0.0 的实例
    └─→ 从过滤后的实例中选择一个（随机+权重）
```

## Spring Cloud Alibaba Nacos Config 集成

### 配置加载流程

Nacos Config 的配置加载发生在 Spring 应用上下文初始化之前：

```
Spring Boot 启动
    │
    ▼
BootstrapApplicationListener
    │
    ▼
NacosConfigBootstrapConfiguration
    │
    ├─→ 读取 spring.cloud.nacos.config 配置
    ├─→ 从 Nacos Server 拉取配置
    ├─→ 将配置注入到 Spring Environment
    └─→ 启动配置监听，等待变更推送
    │
    ▼
Spring Application Context 初始化
    │
    ▼
加载 application.yaml（本地配置）
    │
    ▼
加载 Nacos 配置（通过 spring.config.import）
    │
    ▼
Bean 创建和注入
```

### 两种配置加载方式

yudao-cloud 使用 `spring.config.import` 方式加载 Nacos 配置（Spring Boot 2.4+ 推荐）：

```yaml
# application.yaml
spring:
  config:
    import:
      # 1. 加载本地配置
      - optional:classpath:application-${spring.profiles.active}.yaml
      # 2. 加载 Nacos 配置
      - optional:nacos:${spring.application.name}-${spring.profiles.active}.yaml
```

配置加载优先级（从高到低）：

```
1. Nacos 上的 ${spring.application.name}-${profile}.yaml  ← 最高优先级
2. 本地 application-${profile}.yaml
3. 本地 application.yaml                                   ← 最低优先级
```

这意味着：Nacos 上的配置可以覆盖本地配置，实现了"本地配置做默认值，Nacos 配置做覆盖"的灵活策略。

## 共享配置 vs 应用配置

### 配置分层设计

在微服务架构中，很多配置是所有服务共享的，比如数据库连接池配置、Redis 配置、日志配置等。如果每个服务都维护一份，会造成大量重复。

```
配置分层：
┌─────────────────────────────────────────────────────────┐
│                    应用专属配置                          │
│  ┌──────────────────┐  ┌──────────────────┐            │
│  │system-server.yaml│  │product-server.yaml│  ...       │
│  │                  │  │                  │            │
│  │ 业务开关         │  │ 商品相关配置     │            │
│  │ 特殊参数         │  │ 分页默认值       │            │
│  └──────────────────┘  └──────────────────┘            │
├─────────────────────────────────────────────────────────┤
│                    共享配置                              │
│  ┌──────────────────────────────────────────────┐      │
│  │ application-common.yaml                       │      │
│  │                                              │      │
│  │  数据库连接池配置                              │      │
│  │  Redis 配置                                   │      │
│  │  日志级别配置                                  │      │
│  │  Jackson 序列化配置                            │      │
│  └──────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

### Nacos 中的配置文件组织

```
Nacos 配置列表（namespace: dev）：

Data ID                                    Group
─────────────────────────────────────────  ──────────────
system-server-dev.yaml                     DEFAULT_GROUP    ← system 专属
product-server-dev.yaml                    DEFAULT_GROUP    ← product 专属
trade-server-dev.yaml                      DEFAULT_GROUP    ← trade 专属
application-common.yaml                    SHARED_GROUP     ← 共享配置
```

### 实际配置示例

以 system-server 为例，其 Nacos 配置内容：

```yaml
# Nacos 上的 system-server-dev.yaml

# 数据库配置（可覆盖本地默认值）
spring:
  datasource:
    dynamic:
      datasource:
        master:
          url: jdbc:mysql://nacos-mysql:3306/ruoyi-vue-pro
          username: root
          password: ${MYSQL_PASSWORD:123456}

# 业务配置
yudao:
  tenant:
    enable: true
  security:
    mock-enable: false  # 生产环境关闭 Mock
```

## 命名空间（Namespace）用于环境隔离

### 命名空间的设计

命名空间是 Nacos 中最高级别的隔离单元，用于实现环境级别的隔离：

```
Nacos Server
├── namespace: dev (开发环境)
│   ├── group: DEFAULT_GROUP
│   │   ├── system-server-dev.yaml
│   │   ├── product-server-dev.yaml
│   │   └── trade-server-dev.yaml
│   └── group: SHARED_GROUP
│       └── application-common.yaml
│
├── namespace: test (测试环境)
│   ├── group: DEFAULT_GROUP
│   │   ├── system-server-test.yaml
│   │   ├── product-server-test.yaml
│   │   └── trade-server-test.yaml
│   └── group: SHARED_GROUP
│       └── application-common.yaml
│
└── namespace: prod (生产环境)
    ├── group: DEFAULT_GROUP
    │   ├── system-server-prod.yaml
    │   ├── product-server-prod.yaml
    │   └── trade-server-prod.yaml
    └── group: SHARED_GROUP
        └── application-common.yaml
```

### 命名空间的隔离效果

```
开发环境的 system-server 启动：
  spring.cloud.nacos.config.namespace = dev
  │
  └─→ 只能读取 dev 命名空间下的配置
  └─→ 看不到 test 和 prod 的配置
  └─→ 注册到 dev 命名空间的服务列表

生产环境的 system-server 启动：
  spring.cloud.nacos.config.namespace = prod
  │
  └─→ 只能读取 prod 命名空间下的配置
  └─→ 看不到 dev 和 test 的配置
  └─→ 注册到 prod 命名空间的服务列表
```

这种隔离是物理级别的，不同命名空间的数据完全独立，从根本上避免了环境串扰的问题。

## 配置分组（Group）

### 分组的作用

分组是命名空间之下的二级隔离，用于将配置按业务维度分组：

```
namespace: dev
├── group: DEFAULT_GROUP        ← 默认分组，存放应用配置
│   ├── system-server-dev.yaml
│   ├── product-server-dev.yaml
│   └── trade-server-dev.yaml
│
├── group: SHARED_GROUP         ← 共享分组，存放公共配置
│   └── application-common.yaml
│
└── group: BUSINESS_GROUP       ← 业务分组，存放业务配置
    └── promotion-config.yaml
```

### 共享配置的加载

在 Nacos 配置中指定共享配置：

```yaml
spring:
  cloud:
    nacos:
      config:
        # 共享配置
        shared-configs:
          - data-id: application-common.yaml
            group: SHARED_GROUP
            refresh: true  # 支持热更新
```

共享配置的加载优先级低于应用专属配置，这意味着应用配置可以覆盖共享配置中的同名属性。

## bootstrap.yaml 配置结构

### 传统方式 vs 新方式

**传统方式（Spring Boot 2.4 之前）**：使用 `bootstrap.yaml` 加载 Nacos 配置

```yaml
# bootstrap.yaml（传统方式）
spring:
  application:
    name: system-server
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: dev
        file-extension: yaml
```

**新方式（Spring Boot 2.4+）**：使用 `spring.config.import` 加载

```yaml
# application.yaml（新方式）
spring:
  application:
    name: system-server
  config:
    import:
      - optional:nacos:system-server-dev.yaml
```

yudao-cloud 采用新方式，因为：
1. 不需要额外的 `bootstrap.yaml` 文件，配置更集中
2. `optional:` 前缀表示 Nacos 不可用时不会报错，支持降级到本地配置
3. 符合 Spring Boot 官方推荐

## 多环境配置策略

### 环境切换机制

yudao-cloud 通过 `spring.profiles.active` 切换环境：

```yaml
# application.yaml
spring:
  profiles:
    active: local  # 默认使用本地环境
```

各环境的配置文件：

```
resources/
├── application.yaml            ← 主配置（所有环境共享）
├── application-local.yaml      ← 本地开发配置
├── application-dev.yaml        ← 开发环境配置
├── application-test.yaml       ← 测试环境配置（如果需要）
└── application-prod.yaml       ← 生产环境配置（如果需要）
```

### 本地配置 vs Nacos 配置的分工

```
本地配置（application-local.yaml）：
  ├── Nacos 连接信息（server-addr、namespace）
  ├── 数据库连接（本地开发数据库）
  ├── Redis 连接（本地 Redis）
  └── 开发调试开关（captcha.enable: false）

Nacos 配置（system-server-dev.yaml）：
  ├── 数据库连接（开发环境数据库，覆盖本地）
  ├── Redis 连接（开发环境 Redis，覆盖本地）
  └── 业务配置（统一管理）
```

设计原则：
- **本地配置做默认值**：确保服务在没有 Nacos 的情况下也能启动
- **Nacos 配置做覆盖**：实现配置的集中管理和动态更新
- **敏感信息上 Nacos**：密码、密钥等不写入代码仓库

### 环境切换示例

```bash
# 本地开发
java -jar system-server.jar --spring.profiles.active=local

# 开发环境部署
java -jar system-server.jar --spring.profiles.active=dev

# 生产环境部署
java -jar system-server.jar --spring.profiles.active=prod
```

切换环境时，只需要改变 `spring.profiles.active` 的值，服务会自动加载对应环境的配置。

## 思考题

1. **yudao-cloud 使用 `spring.config.import` 方式加载 Nacos 配置，而不是传统的 `bootstrap.yaml`。这两种方式在配置加载顺序上有什么区别？在什么场景下你必须使用 `bootstrap.yaml`？**

2. **共享配置（shared-configs）和应用配置的优先级关系是什么？如果共享配置和应用配置中都定义了 `spring.datasource.url`，最终生效的是哪个？**

3. **命名空间（namespace）和分组（group）都能实现配置隔离，为什么需要两级隔离？只用命名空间够不够？只用分组够不够？**

4. **在生产环境中，如果 Nacos Server 宕机了，已启动的微服务还能正常运行吗？新启动的微服务呢？**
