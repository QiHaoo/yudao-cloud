# 06-核心设计：API访问日志

## 场景引入

系统上线后，运维团队提出了以下监控需求：

- **性能监控**：哪些接口响应时间超过 1 秒？
- **错误追踪**：哪些接口经常返回 500 错误？
- **访问统计**：每天的 API 调用量是多少？高峰时段是什么时候？
- **异常排查**：用户反馈操作失败，如何查看详细的请求和异常信息？

这些问题都需要通过 API 访问日志来解决。

## API 访问日志 vs 操作日志

在 yudao-cloud 中，有两种日志：

| 维度 | API 访问日志 | 操作日志 |
|------|-------------|---------|
| **记录范围** | 所有 API 请求 | 只记录标注了 `@ApiAccessLog` 的方法 |
| **实现方式** | Servlet Filter | Servlet Filter + 注解 |
| **记录内容** | 请求URL、参数、响应、耗时 | 操作人、操作类型、业务含义 |
| **主要用途** | 性能监控、错误追踪 | 业务审计、操作追溯 |
| **存储位置** | `infra_api_access_log` 表 | `system_operate_log` 表 |

**设计思路**：

```
┌─────────────────────────────────────────────────────────────────┐
│                    两种日志的分工                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  API 访问日志（技术层面）：                                       │
│  ├── 记录所有 HTTP 请求                                          │
│  ├── 关注：请求URL、参数、响应码、耗时                           │
│  └── 用途：性能监控、错误追踪、接口统计                          │
│                                                                 │
│  操作日志（业务层面）：                                           │
│  ├── 只记录关键业务操作                                          │
│  ├── 关注：谁在什么时间做了什么操作                              │
│  └── 用途：业务审计、操作追溯、合规检查                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## ApiAccessLogDO：访问日志实体

```java
// 源码位置：yudao-module-infra-server/.../dal/dataobject/logger/ApiAccessLogDO.java
@TableName("infra_api_access_log")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ApiAccessLogDO extends BaseDO {

    /**
     * 请求参数的最大长度
     */
    public static final Integer REQUEST_PARAMS_MAX_LENGTH = 8000;

    /**
     * 响应结果的最大长度
     */
    public static final Integer RESULT_MSG_MAX_LENGTH = 512;

    /**
     * 编号
     */
    @TableId
    private Long id;

    /**
     * 链路追踪编号
     * 通过链路追踪编号，可以将访问日志、错误日志、链路追踪日志、logger 打印日志等结合在一起
     */
    private String traceId;

    /**
     * 用户编号
     */
    private Long userId;

    /**
     * 用户类型
     * 枚举 UserTypeEnum
     */
    private Integer userType;

    /**
     * 应用名
     * 读取 spring.application.name 配置项
     */
    private String applicationName;

    // ========== 请求相关字段 ==========

    /**
     * 请求方法名
     * GET, POST, PUT, DELETE
     */
    private String requestMethod;

    /**
     * 访问地址
     */
    private String requestUrl;

    /**
     * 请求参数
     * query: Query String
     * body: Request Body
     */
    private String requestParams;

    /**
     * 响应结果
     */
    private String responseBody;

    /**
     * 用户 IP
     */
    private String userIp;

    /**
     * 浏览器 UA
     */
    private String userAgent;

    // ========== 执行相关字段 ==========

    /**
     * 操作模块
     */
    private String operateModule;

    /**
     * 操作名
     */
    private String operateName;

    /**
     * 操作分类
     * 枚举 OperateTypeEnum
     */
    private Integer operateType;

    /**
     * 开始请求时间
     */
    private LocalDateTime beginTime;

    /**
     * 结束请求时间
     */
    private LocalDateTime endTime;

    /**
     * 执行时长，单位：毫秒
     */
    private Integer duration;

    /**
     * 结果码
     * 使用 CommonResult#getCode() 属性
     */
    private Integer resultCode;

    /**
     * 结果提示
     * 使用 CommonResult#getMsg() 属性
     */
    private String resultMsg;
}
```

## ApiErrorLogDO：错误日志实体

当 API 请求发生异常时，会记录详细的错误信息：

```java
// 源码位置：yudao-module-infra-server/.../dal/dataobject/logger/ApiErrorLogDO.java
@TableName("infra_api_error_log")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ApiErrorLogDO extends BaseDO {

    /**
     * 请求参数的最大长度
     */
    public static final Integer REQUEST_PARAMS_MAX_LENGTH = 8000;

    /**
     * 编号
     */
    @TableId
    private Long id;

    /**
     * 用户编号
     */
    private Long userId;

    /**
     * 链路追踪编号
     */
    private String traceId;

    /**
     * 用户类型
     * 枚举 UserTypeEnum
     */
    private Integer userType;

    /**
     * 应用名
     */
    private String applicationName;

    // ========== 请求相关字段 ==========

    /**
     * 请求方法名
     */
    private String requestMethod;

    /**
     * 访问地址
     */
    private String requestUrl;

    /**
     * 请求参数
     */
    private String requestParams;

    /**
     * 用户 IP
     */
    private String userIp;

    /**
     * 浏览器 UA
     */
    private String userAgent;

    // ========== 异常相关字段 ==========

    /**
     * 异常发生时间
     */
    private LocalDateTime exceptionTime;

    /**
     * 异常名
     * Throwable#getClass() 的类全名
     */
    private String exceptionName;

    /**
     * 异常导致的消息
     */
    private String exceptionMessage;

    /**
     * 异常导致的根消息
     */
    private String exceptionRootCauseMessage;

    /**
     * 异常的栈轨迹
     */
    private String exceptionStackTrace;

    /**
     * 异常发生的类全名
     */
    private String exceptionClassName;

    /**
     * 异常发生的类文件
     */
    private String exceptionFileName;

    /**
     * 异常发生的方法名
     */
    private String exceptionMethodName;

    /**
     * 异常发生的方法所在行
     */
    private Integer exceptionLineNumber;

    // ========== 处理相关字段 ==========

    /**
     * 处理状态
     * 枚举 ApiErrorLogProcessStatusEnum
     */
    private Integer processStatus;

    /**
     * 处理时间
     */
    private LocalDateTime processTime;

    /**
     * 处理用户编号
     */
    private Long processUserId;
}
```

**错误日志 vs 访问日志**：

| 维度 | 访问日志 | 错误日志 |
|------|---------|---------|
| **记录时机** | 每次请求都记录 | 只在异常时记录 |
| **记录内容** | 请求和响应的基本信息 | 详细的异常堆栈信息 |
| **处理流程** | 无 | 支持标记为"已处理" |
| **存储表** | `infra_api_access_log` | `infra_api_error_log` |

## 日志处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    日志处理完整流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTP 请求进入                                                   │
│       │                                                         │
│       ▼                                                         │
│  ApiAccessLogFilter.doFilterInternal()                          │
│       │                                                         │
│       ├── 记录开始时间                                           │
│       ├── 提前获取请求参数                                       │
│       │                                                         │
│       ▼                                                         │
│  继续 Filter 链 ──► Controller 执行                             │
│       │                                                         │
│       ├── 正常返回                                               │
│       │   │                                                     │
│       │   ▼                                                     │
│       │   构建访问日志 ──► 异步写入 infra_api_access_log         │
│       │                                                         │
│       └── 异常抛出                                               │
│           │                                                     │
│           ▼                                                     │
│           构建访问日志 ──► 异步写入 infra_api_access_log         │
│           │                                                     │
│           ▼                                                     │
│           构建错误日志 ──► 异步写入 infra_api_error_log          │
│           │                                                     │
│           ▼                                                     │
│           继续抛出异常（由 GlobalExceptionHandler 处理）         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 敏感参数脱敏

### 内置脱敏字段

```java
// 源码位置：yudao-framework/.../apilog/core/filter/ApiAccessLogFilter.java
private static final String[] SANITIZE_KEYS = new String[]{
    "password",      // 密码
    "token",         // Token
    "accessToken",   // 访问令牌
    "refreshToken"   // 刷新令牌
};
```

### 脱敏效果示例

**请求参数脱敏**：

```json
// 原始参数
{
    "username": "admin",
    "password": "123456",
    "captcha": "abcd"
}

// 脱敏后
{
    "username": "admin"
}
```

**响应结果脱敏**：

```json
// 原始响应
{
    "code": 0,
    "msg": "success",
    "data": {
        "userId": 1,
        "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
        "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    }
}

// 脱敏后
{
    "code": 0,
    "msg": "success",
    "data": {
        "userId": 1
    }
}
```

## 日志查询 API

```java
// 源码位置：yudao-module-infra-server/.../controller/admin/logger/ApiAccessLogController.java
@Tag(name = "管理后台 - API 访问日志")
@RestController
@RequestMapping("/infra/api-access-log")
public class ApiAccessLogController {

    @Resource
    private ApiAccessLogService apiAccessLogService;

    @GetMapping("/page")
    @Operation(summary = "获得 API 访问日志分页")
    @PreAuthorize("@ss.hasPermission('infra:api-access-log:query')")
    public CommonResult<PageResult<ApiAccessLogRespVO>> getApiAccessLogPage(
            @Valid ApiAccessLogPageReqVO pageReqVO) {
        PageResult<ApiAccessLogDO> pageResult = apiAccessLogService.getApiAccessLogPage(pageReqVO);
        return success(BeanUtils.toBean(pageResult, ApiAccessLogRespVO.class));
    }

    @GetMapping("/export")
    @Operation(summary = "导出 API 访问日志")
    @PreAuthorize("@ss.hasPermission('infra:api-access-log:export')")
    public void exportApiAccessLog(@Valid ApiAccessLogPageReqVO pageReqVO,
                                   HttpServletResponse response) throws IOException {
        List<ApiAccessLogDO> list = apiAccessLogService.getApiAccessLogList(pageReqVO);
        // 导出 Excel
        ExcelUtils.write(response, "API访问日志.xls", "数据",
                         ApiAccessLogRespVO.class, BeanUtils.toBean(list, ApiAccessLogRespVO.class));
    }
}
```

```java
// 源码位置：yudao-module-infra-server/.../controller/admin/logger/ApiErrorLogController.java
@Tag(name = "管理后台 - API 错误日志")
@RestController
@RequestMapping("/infra/api-error-log")
public class ApiErrorLogController {

    @Resource
    private ApiErrorLogService apiErrorLogService;

    @GetMapping("/page")
    @Operation(summary = "获得 API 错误日志分页")
    @PreAuthorize("@ss.hasPermission('infra:api-error-log:query')")
    public CommonResult<PageResult<ApiErrorLogRespVO>> getApiErrorLogPage(
            @Valid ApiErrorLogPageReqVO pageReqVO) {
        PageResult<ApiErrorLogDO> pageResult = apiErrorLogService.getApiErrorLogPage(pageReqVO);
        return success(BeanUtils.toBean(pageResult, ApiErrorLogRespVO.class));
    }

    @PutMapping("/update-process")
    @Operation(summary = "更新 API 错误日志的处理状态")
    @PreAuthorize("@ss.hasPermission('infra:api-error-log:update')")
    public CommonResult<Boolean> updateApiErrorLogProcess(
            @RequestParam("id") Long id, @RequestParam("processStatus") Integer processStatus) {
        apiErrorLogService.updateApiErrorLogProcess(id, processStatus);
        return success(true);
    }
}
```

## 日志清理

为了避免日志数据无限增长，需要定期清理历史日志：

```java
// 源码位置：yudao-module-infra-server/.../job/logger/AccessLogCleanJob.java
@Component
public class AccessLogCleanJob {

    @Resource
    private ApiAccessLogService apiAccessLogService;

    /**
     * 清理访问日志
     * 默认保留 7 天
     */
    @Scheduled(cron = "0 0 2 * * ?")  // 每天凌晨 2 点执行
    public void cleanAccessLog() {
        apiAccessLogService.cleanAccessLog(7);  // 保留 7 天
    }
}
```

```java
// 源码位置：yudao-module-infra-server/.../job/logger/ErrorLogCleanJob.java
@Component
public class ErrorLogCleanJob {

    @Resource
    private ApiErrorLogService apiErrorLogService;

    /**
     * 清理错误日志
     * 默认保留 30 天
     */
    @Scheduled(cron = "0 0 3 * * ?")  // 每天凌晨 3 点执行
    public void cleanErrorLog() {
        apiErrorLogService.cleanErrorLog(30);  // 保留 30 天
    }
}
```

## 监控场景示例

### 场景一：性能监控

```sql
-- 查询响应时间超过 1 秒的接口
SELECT request_url, request_method, duration, begin_time
FROM infra_api_access_log
WHERE duration > 1000
ORDER BY duration DESC
LIMIT 100;

-- 统计每个接口的平均响应时间
SELECT request_url, request_method,
       COUNT(*) as request_count,
       AVG(duration) as avg_duration,
       MAX(duration) as max_duration
FROM infra_api_access_log
WHERE begin_time >= DATE_SUB(NOW(), INTERVAL 1 DAY)
GROUP BY request_url, request_method
ORDER BY avg_duration DESC;
```

### 场景二：错误追踪

```sql
-- 查询最近的错误日志
SELECT request_url, exception_name, exception_message, exception_time
FROM infra_api_error_log
WHERE process_status = 0  -- 未处理
ORDER BY exception_time DESC
LIMIT 50;

-- 统计错误率最高的接口
SELECT request_url, COUNT(*) as error_count
FROM infra_api_error_log
WHERE exception_time >= DATE_SUB(NOW(), INTERVAL 1 DAY)
GROUP BY request_url
ORDER BY error_count DESC;
```

### 场景三：访问统计

```sql
-- 统计每小时的访问量
SELECT DATE_FORMAT(begin_time, '%Y-%m-%d %H:00:00') as hour,
       COUNT(*) as request_count
FROM infra_api_access_log
WHERE begin_time >= DATE_SUB(NOW(), INTERVAL 7 DAY)
GROUP BY hour
ORDER BY hour;

-- 统计访问量最高的接口
SELECT request_url, request_method, COUNT(*) as request_count
FROM infra_api_access_log
WHERE begin_time >= DATE_SUB(NOW(), INTERVAL 1 DAY)
GROUP BY request_url, request_method
ORDER BY request_count DESC
LIMIT 20;
```

## 链路追踪集成

日志中的 `traceId` 字段用于链路追踪，可以将多个系统的日志串联起来：

```
┌─────────────────────────────────────────────────────────────────┐
│                    链路追踪示例                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  用户请求                                                        │
│       │                                                         │
│       ▼                                                         │
│  Gateway (traceId=abc123)                                       │
│       │                                                         │
│       ▼                                                         │
│  System Service (traceId=abc123)                                │
│       │                                                         │
│       ├── 访问日志: infra_api_access_log (traceId=abc123)       │
│       │                                                         │
│       ├── 日志打印: [traceId=abc123] 用户登录成功               │
│       │                                                         │
│       └── 错误日志: infra_api_error_log (traceId=abc123)        │
│                                                                 │
│  通过 traceId=abc123 可以追踪完整的请求链路                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 本章小结

API 访问日志模块通过以下设计实现了全面的 API 监控：

1. **全量记录**：通过 Servlet Filter 自动记录所有 API 请求
2. **错误分离**：错误日志单独存储，便于追踪和处理
3. **敏感脱敏**：自动脱敏密码、Token 等敏感信息
4. **异步写入**：日志异步写入数据库，不影响请求性能
5. **定期清理**：通过定时任务自动清理历史日志
6. **链路追踪**：通过 traceId 串联多个系统的日志

下一章我们将讨论配置设计与边界，了解各子功能的配置项以及与其他模块的分工。
