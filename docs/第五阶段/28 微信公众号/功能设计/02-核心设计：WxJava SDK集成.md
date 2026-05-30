# 02 - 核心设计：WxJava SDK 集成

## 场景引入

上一章我们看到了微信公众号模块的全景。这一章深入几个核心设计问题：

1. **WxJava SDK 集成**：怎么在 Spring Boot 中集成 WxJava？怎么配置多公众号？
2. **消息路由**：微信推送的消息怎么分发到不同的处理器？
3. **自动回复**：关键词匹配和消息类型匹配怎么实现？
4. **模板消息**：怎么给用户推送订单状态通知？

---

## WxJava SDK 集成

### 依赖引入

在 `pom.xml` 中引入 WxJava 的公众号组件：

```xml
<dependency>
    <groupId>com.github.binarywang</groupId>
    <artifactId>weixin-java-mp</artifactId>
    <version>${weixin-java.version}</version>
</dependency>
```

### 多公众号配置

一个系统可能管理多个公众号。WxJava 支持通过 `WxMpService` 的不同实例来区分：

```
多公众号架构
  │
  ├── 公众号 A (appId: wx_aaa)
  │     └── WxMpService 实例 A
  │
  ├── 公众号 B (appId: wx_bbb)
  │     └── WxMpService 实例 B
  │
  └── 公众号 C (appId: wx_ccc)
        └── WxMpService 实例 C
```

### WxMpService 的创建

每个公众号对应一个 `WxMpService` 实例，配置信息来自 `MpAccountDO`：

```
MpAccountDO (数据库)
  │
  ├── appId
  ├── appSecret
  ├── token
  └── aesKey
  │
  ▼
WxMpConfigStorage (配置存储)
  │
  ▼
WxMpService (服务实例)
```

### WxMpService 核心能力

```
WxMpService
  │
  ├── getAccessToken()
  │     → 获取访问令牌（自动刷新）
  │
  ├── getUserService()
  │     ├── userInfo()        - 获取粉丝信息
  │     ├── userUpdate()      - 更新粉丝备注
  │     └── userList()        - 获取粉丝列表
  │
  ├── getMenuService()
  │     ├── menuCreate()      - 创建菜单
  │     ├── menuGet()         - 查询菜单
  │     └── menuDelete()      - 删除菜单
  │
  ├── getMaterialService()
  │     ├── materialFileUpload()  - 上传临时素材
  │     ├── materialNewsUpload()  - 上传永久图文素材
  │     └── materialDelete()      - 删除永久素材
  │
  └── getTemplateMsgService()
        └── sendTemplateMsg() - 发送模板消息
```

---

## 消息路由

### 消息接收流程

```
用户发送消息到公众号
  │
  ▼
微信服务器
  │
  │  HTTP POST (XML 格式)
  │  URL: /mp/message/{appId}
  ▼
MpMessageController (消息回调接口)
  │
  ├── 1. 验证签名（用 token 验证消息来自微信）
  ├── 2. 解密消息（如果开启了安全模式）
  ├── 3. 解析 XML 消息
  │
  ▼
WxMpMessageRouter (消息路由器)
  │
  ├── 规则 1：消息类型 = text + 内容包含"查订单"
  │     → 处理器：OrderQueryHandler
  │
  ├── 规则 2：消息类型 = text + 内容包含"客服"
  │     → 处理器：CustomerServiceHandler
  │
  ├── 规则 3：消息类型 = event + 事件 = subscribe
  │     → 处理器：SubscribeHandler（关注回复）
  │
  └── 默认：未匹配任何规则
        → 返回空（不回复）
```

### 消息路由规则

WxJava 的 `WxMpMessageRouter` 支持链式配置规则：

```java
// 伪代码示例
new WxMpMessageRouter(wxMpService)
    .rule()
        .msgType(WxConsts.XmlMsgType.TEXT)
        .content("查订单")
        .handler(orderQueryHandler)
        .end()
    .rule()
        .msgType(WxConsts.XmlMsgType.TEXT)
        .content("客服")
        .handler(customerServiceHandler)
        .end()
    .rule()
        .event(WxConsts.EventType.SUBSCRIBE)
        .handler(subscribeHandler)
        .end();
```

### 消息处理器

每个处理器实现 `WxMpMessageHandler` 接口：

```java
public interface WxMpMessageHandler {
    WxMpXmlOutMessage handle(WxMpXmlMessage wxMessage,
                              Map<String, Object> context,
                              WxMpService wxMpService,
                              WxMpSessionManager sessionManager) throws WxErrorException;
}
```

---

## 自动回复设计

### 自动回复实体

```java
// MpAutoReplyDO - 自动回复配置
public class MpAutoReplyDO extends BaseDO {
    private Long id;            // 编号
    private Long accountId;     // 公众号账号编号
    private String appId;       // App ID

    // 回复类型
    private Integer type;       // 关注回复/关键词回复/消息回复

    // 请求条件
    private String requestKeyword;      // 关键词（关键词回复时）
    private Integer requestMatch;       // 匹配方式（全匹配/半匹配）
    private String requestMessageType;  // 消息类型（消息回复时）

    // 回复内容
    private String responseMessageType; // 回复消息类型
    private String responseContent;     // 回复文本内容
    private String responseMediaId;     // 回复媒体 ID
    private List<Article> responseArticles; // 回复图文消息
    // ...
}
```

### 关键词匹配方式

```
关键词匹配 (MpAutoReplyMatchEnum)
  │
  ├── 全匹配 (FULL)
  │     → 用户消息 = 关键词
  │     → 例：用户发送"查订单" → 匹配关键词"查订单"
  │
  └── 半匹配 (PARTIAL)
        → 用户消息包含关键词
        → 例：用户发送"我想查订单" → 匹配关键词"查订单"
```

### 自动回复匹配流程

```
用户发送消息
  │
  ▼
匹配自动回复规则
  │
  ├── 1. 查找该公众号的所有自动回复配置
  │
  ├── 2. 按类型匹配
  │     ├── type = FOLLOW（关注回复）
  │     │     → 只在用户关注时触发
  │     │
  │     ├── type = KEYWORD（关键词回复）
  │     │     ├── requestMatch = FULL
  │     │     │     → 用户消息 == requestKeyword？
  │     │     └── requestMatch = PARTIAL
  │     │           → 用户消息包含 requestKeyword？
  │     │
  │     └── type = MESSAGE（消息回复）
  │           → 用户消息类型 == requestMessageType？
  │
  ├── 3. 匹配成功 → 返回对应的回复内容
  │
  └── 4. 未匹配 → 使用默认回复（如果有）
```

---

## 模板消息

### 模板消息的含义

模板消息是公众号向粉丝推送通知的唯一途径。微信对模板消息有严格限制：

- 只能推送与用户相关的通知（订单状态、物流信息等）
- 不能推送营销内容
- 每个用户每天有推送次数限制

### 模板消息实体

```java
// MpMessageTemplateDO - 消息模板
public class MpMessageTemplateDO extends BaseDO {
    private Long id;            // 模板编号
    private Long accountId;     // 公众号账号编号
    private String appId;       // App ID
    private String templateId;  // 微信模板 ID
    private String name;        // 模板名称
    private String content;     // 模板内容
    private String example;     // 模板示例
}
```

### 模板消息发送流程

```
业务系统（电商）              公众号模块                微信服务器
  │                           │                        │
  │  发送模板消息请求          │                        │
  │  (openId, templateId,    │                        │
  │   data: {订单号, 状态})   │                        │
  │──────────────────────────>│                        │
  │                           │                        │
  │                           │  sendTemplateMsg()     │
  │                           │───────────────────────>│
  │                           │                        │
  │                           │  发送成功              │
  │                           │<───────────────────────│
  │  成功                     │                        │
  │<──────────────────────────│                        │
```

### 模板消息内容示例

```
模板：订单状态通知
  │
  ├── 标题：您的订单状态已更新
  │
  ├── 订单编号：{{order_no.DATA}}
  ├── 订单状态：{{status.DATA}}
  ├── 下单时间：{{time.DATA}}
  └── 温馨提示：{{remark.DATA}}
```

---

## 菜单同步

### 菜单结构

微信公众号的自定义菜单最多 3 个一级菜单，每个一级菜单最多 5 个子菜单：

```
自定义菜单
  │
  ├── 一级菜单：商城入口
  │     ├── 子菜单：热门商品 → 跳转小程序
  │     ├── 子菜单：新品上架 → 跳转小程序
  │     └── 子菜单：优惠活动 → 跳转网页
  │
  ├── 一级菜单：我的
  │     ├── 子菜单：我的订单 → 跳转小程序
  │     ├── 子菜单：收货地址 → 跳转小程序
  │     └── 子菜单：会员中心 → 跳转小程序
  │
  └── 一级菜单：客服
        ├── 子菜单：在线客服 → 转发到客服
        └── 子菜单：常见问题 → 跳转网页
```

### 菜单同步流程

```
后台管理菜单
  │
  ▼
保存到数据库 (MpMenuDO)
  │
  ▼
同步到微信服务器
  │
  ├── 1. 查询当前微信服务器上的菜单
  ├── 2. 对比数据库和微信服务器的差异
  ├── 3. 删除微信服务器上的旧菜单
  └── 4. 创建数据库中的新菜单
```

---

## 素材管理

### 素材类型

```
素材类型
  │
  ├── 临时素材
  │     → 有效期 3 天
  │     → 用于消息回复中的图片、语音、视频
  │
  └── 永久素材
        → 永久有效
        → 用于图文消息、自定义菜单
```

### 素材上传流程

```
上传素材
  │
  ├── 1. 上传文件到本地服务器
  │
  ├── 2. 调用微信 API 上传到微信服务器
  │     └── 获得 mediaId
  │
  ├── 3. 保存素材记录到数据库 (MpMaterialDO)
  │     └── 存储 mediaId、文件 URL 等
  │
  └── 4. 后续使用时直接用 mediaId
```

---

## 思考题

1. WxJava 的 `WxMpService` 是线程安全的吗？多个请求同时调用会不会有问题？
2. 消息路由用的是"规则匹配"模式。如果规则很多（几百条），匹配效率会不会有问题？
3. 模板消息的推送频率被微信限制了。如果电商大促时需要给 10 万粉丝推送，怎么处理？
4. 自动回复的关键词匹配支持"半匹配"。如果用户发送的消息同时匹配多个关键词，应该回复哪个？
