# 02 - 核心设计：JimuReport 集成

## 场景引入

上一章我们看到了报表模块的全景——JimuReport 和 GoView 两大工具。这一章深入几个核心设计问题：

1. **JimuReport 集成方式**：怎么在 Spring Boot 中集成 JimuReport？
2. **数据集设计**：SQL 数据集、API 数据集、Java 数据集各自怎么用？
3. **报表导出**：Excel、PDF、Word 导出怎么实现？
4. **GoView 数据接口**：大屏的数据接口怎么设计？

---

## JimuReport 集成方式

### 依赖引入

```xml
<dependency>
    <groupId>org.jeecgframework.jimureport</groupId>
    <artifactId>jimureport-spring-boot-starter</artifactId>
    <version>${jimureport.version}</version>
</dependency>
```

### 集成模式

JimuReport 支持两种集成模式：

```
模式 1：内嵌模式（推荐）
  │
  ├── JimuReport 作为 JAR 依赖嵌入业务系统
  ├── 共用业务系统的数据源
  ├── 通过 /jmreport/list 访问报表列表
  ├── 通过 /jmreport/design/{id} 访问设计器
  └── 通过 /jmreport/view/{id} 访问报表页面

模式 2：独立部署
  │
  ├── JimuReport 独立部署为一个 Spring Boot 应用
  ├── 配置独立的数据源
  └── 通过 API 获取报表数据
```

### 内嵌模式的优势

- **共享数据源**：报表直接查询业务数据库，不需要数据同步
- **统一权限**：复用业务系统的用户和权限体系
- **部署简单**：不需要额外部署服务

### 内嵌模式的劣势

- **性能影响**：复杂报表查询可能影响业务系统性能
- **耦合度高**：报表和业务系统在同一个进程中

---

## 数据集设计

### 数据集类型

```
数据集类型
  │
  ├── SQL 数据集
  │     → 直接执行 SQL 查询
  │     → 例：SELECT * FROM orders WHERE date >= #{startDate}
  │
  ├── API 数据集
  │     → 调用 REST 接口获取数据
  │     → 例：GET /api/sales/trend?startDate=2024-06-01
  │
  └── Java 数据集
        → 调用 Java 方法获取数据
        → 例：SalesService.getSalesTrend(startDate)
```

### SQL 数据集

最常用的数据集类型。直接在报表设计器中编写 SQL：

```sql
-- 销售日报数据集
SELECT
    DATE(o.create_time) AS date,
    COUNT(*) AS order_count,
    SUM(o.pay_price) AS total_amount
FROM trade_order o
WHERE o.create_time >= #{startDate}
  AND o.create_time <= #{endDate}
  AND o.status = 30  -- 已支付
GROUP BY DATE(o.create_time)
ORDER BY date
```

**参数化查询**：`#{startDate}` 和 `#{endDate}` 是报表参数，用户在查询界面输入。

### API 数据集

适合数据来自其他微服务的场景：

```
API 数据集配置
  │
  ├── URL：/api/report/sales/trend
  ├── Method：GET
  ├── 参数：
  │     ├── startDate: #{startDate}
  │     └── endDate: #{endDate}
  │
  └── 返回格式：
        {
          "data": [
            {"date": "2024-06-01", "amount": 45000},
            {"date": "2024-06-02", "amount": 52000}
          ]
        }
```

### Java 数据集

适合复杂计算逻辑的场景：

```java
// Java 数据集示例
public class SalesReportService {

    /**
     * 获取销售趋势数据
     * 被 JimuReport 调用
     */
    public List<Map<String, Object>> getSalesTrend(String startDate, String endDate) {
        // 复杂的业务逻辑
        // ...
        return result;
    }
}
```

---

## 报表模板设计

### 报表区域

```
报表模板结构
  │
  ├── 标题区
  │     └── 报表标题、日期范围
  │
  ├── 参数区
  │     └── 查询条件输入框
  │
  ├── 表头区
  │     └── 列标题
  │
  ├── 表体区
  │     └── 数据行（绑定数据集字段）
  │
  └── 表尾区
        └── 汇总行（SUM、AVG、COUNT）
```

### 字段绑定

```
表头区：
  ┌──────────┬──────────┬──────────┐
  │ 日期      │ 订单数    │ 销售额    │
  └──────────┴──────────┴──────────┘

表体区（绑定数据集字段）：
  ┌──────────┬──────────┬──────────┐
  │ ${date}  │ ${count} │ ${amount}│
  └──────────┴──────────┴──────────┘

表尾区（汇总公式）：
  ┌──────────┬──────────┬──────────┐
  │ 合计      │ SUM(订单) │ SUM(销售) │
  └──────────┴──────────┴──────────┘
```

### 数据格式

```
数据格式设置
  │
  ├── 数字格式
  │     ├── #,##0        → 1,234
  │     ├── #,##0.00     → 1,234.56
  │     └── ¥#,##0.00   → ¥1,234.56
  │
  ├── 日期格式
  │     ├── yyyy-MM-dd   → 2024-06-01
  │     └── yyyy年MM月dd日 → 2024年06月01日
  │
  └── 百分比格式
        ├── 0.00%        → 12.34%
        └── 0.0%         → 12.3%
```

---

## 报表导出

### 导出格式

```
导出格式
  │
  ├── Excel (.xlsx)
  │     → 适合数据分析、二次加工
  │     → 支持多 Sheet
  │
  ├── PDF (.pdf)
  │     → 适合打印、存档
  │     → 格式固定，不可编辑
  │
  ├── Word (.docx)
  │     → 适合报告文档
  │     → 可编辑
  │
  └── 图片 (.png)
        → 适合嵌入邮件、PPT
```

### 导出流程

```
导出流程
  │
  ├── 1. 用户点击"导出 Excel"
  │
  ├── 2. 后端接收导出请求
  │     ├── 参数：报表 ID、查询条件、导出格式
  │
  ├── 3. 执行数据查询
  │     ├── 根据数据集查询数据
  │     └── 应用查询条件
  │
  ├── 4. 渲染报表
  │     ├── 填充模板
  │     ├── 应用数据格式
  │     └── 计算汇总行
  │
  └── 5. 生成文件并返回
        ├── 设置 Content-Type
        ├── 设置 Content-Disposition（文件名）
        └── 输出文件流
```

---

## GoView 数据接口设计

### 数据接口规范

GoView 的数据接口需要返回特定格式的 JSON：

```json
{
  "code": 0,
  "data": [
    {"date": "2024-06-01", "amount": 45000},
    {"date": "2024-06-02", "amount": 52000}
  ]
}
```

### GoView 数据接口示例

```java
@RestController
@RequestMapping("/api/report")
public class GoViewDataController {

    /**
     * 销售趋势数据
     * 供 GoView 折线图组件调用
     */
    @GetMapping("/sales/trend")
    public CommonResult<List<Map<String, Object>>> getSalesTrend(
            @RequestParam String startDate,
            @RequestParam String endDate) {
        // 查询销售趋势数据
        List<Map<String, Object>> data = salesService.getTrend(startDate, endDate);
        return CommonResult.success(data);
    }

    /**
     * 商品销量排行
     * 供 GoView 柱状图组件调用
     */
    @GetMapping("/sales/ranking")
    public CommonResult<List<Map<String, Object>>> getSalesRanking(
            @RequestParam(defaultValue = "10") int limit) {
        // 查询商品销量排行
        List<Map<String, Object>> data = salesService.getRanking(limit);
        return CommonResult.success(data);
    }
}
```

### GoView 数据刷新

GoView 支持定时刷新数据：

```
数据刷新配置
  │
  ├── 刷新间隔：5 秒 / 10 秒 / 30 秒 / 1 分钟
  │
  └── 刷新方式：
        ├── 全量刷新：重新请求所有数据接口
        └── 增量刷新：只请求变化的数据
```

---

## 报表权限控制

### 权限模型

```
报表权限
  │
  ├── 菜单权限
  │     ├── report:jimureport:view    查看报表
  │     ├── report:jimureport:design  设计报表
  │     ├── report:goview:view        查看大屏
  │     └── report:goview:design      设计大屏
  │
  └── 数据权限
        ├── 某些报表只允许特定角色查看
        └── 某些报表的数据按部门过滤
```

---

## 思考题

1. JimuReport 的 SQL 数据集直接查询业务数据库。如果报表 SQL 很复杂（多表 JOIN、子查询），会不会拖慢业务系统？
2. GoView 的数据接口返回的 JSON 格式是固定的。如果需要返回嵌套数据（如树形结构），怎么处理？
3. 报表导出 PDF 时，如果数据量很大（几万行），生成 PDF 的时间会不会很长？怎么优化？
4. JimuReport 和 GoView 的数据源配置是独立的还是共享的？如果共享，怎么保证数据一致性？
