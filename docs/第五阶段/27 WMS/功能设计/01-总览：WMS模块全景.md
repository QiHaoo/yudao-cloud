# 01 - 总览：WMS 模块全景

## 场景引入

你做了一个电商系统，仓库里堆满了货。问题来了：

- 仓库管理员问"这个 SKU 在哪个货架？还有多少库存？"
- 发货员问"今天要发多少单？哪些货要出库？"
- 采购问"上批货入库了吗？入了几个仓？"
- 盘点时发现"系统说有 100 个，实际只有 95 个，差了 5 个去哪了？"

**WMS（Warehouse Management System，仓库管理系统）** 就是为了解决这些问题而生的。它管理仓库中货物的"进、出、存、盘"全流程。

### 为什么电商系统需要 WMS？

很多人以为 ERP 的库存管理就够了。但 WMS 和 ERP 的库存管理关注点不同：

```
ERP 库存管理                    WMS 库存管理
├── 关注"有多少"               ├── 关注"在哪里"
├── 产品维度（SKU 级别）        ├── 库位维度（货架级别）
├── 财务对账用                 ├── 仓库作业用
└── 粗粒度                    └── 细粒度
```

如果你的电商业务仓库不大（几百个 SKU），ERP 的库存管理够用。但如果仓库大（几千个 SKU、多个仓库），就需要 WMS。

---

## 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           WMS 模块能力全景                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                       基础数据 (md)                                │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │  │
│  │  │ 商品管理  │  │ 商品 SKU │  │ 品牌     │  │ 分类     │         │  │
│  │  │(Item)    │  │(ItemSku) │  │(Brand)   │  │(Category)│         │  │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │  │
│  │  ┌──────────┐  ┌──────────┐                                     │  │
│  │  │ 仓库管理  │  │ 供应商/  │                                     │  │
│  │  │(Warehse) │  │ 客户     │                                     │  │
│  │  └──────────┘  └──────────┘                                     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────┐    ┌──────────────────────────┐         │
│  │        入库管理            │    │        出库管理            │         │
│  │                          │    │                          │         │
│  │  ┌──────────────────┐   │    │  ┌──────────────────┐   │         │
│  │  │ 入库单             │   │    │  │ 出库单             │   │         │
│  │  │(ReceiptOrder)    │   │    │  │(ShipmentOrder)   │   │         │
│  │  └──────────────────┘   │    │  └──────────────────┘   │         │
│  │  ┌──────────────────┐   │    │  ┌──────────────────┐   │         │
│  │  │ 入库明细           │   │    │  │ 出库明细           │   │         │
│  │  │(ReceiptDetail)   │   │    │  │(ShipmentDetail)  │   │         │
│  │  └──────────────────┘   │    │  └──────────────────┘   │         │
│  └──────────────────────────┘    └──────────────────────────┘         │
│                                                                         │
│  ┌──────────────────────────┐    ┌──────────────────────────┐         │
│  │        库存管理            │    │        盘点管理            │         │
│  │                          │    │                          │         │
│  │  ┌──────────────────┐   │    │  ┌──────────────────┐   │         │
│  │  │ 库存查询           │   │    │  │ 盘点单             │   │         │
│  │  │(Inventory)       │   │    │  │(CheckOrder)      │   │         │
│  │  └──────────────────┘   │    │  └──────────────────┘   │         │
│  │  ┌──────────────────┐   │    │  ┌──────────────────┐   │         │
│  │  │ 库存变动历史       │   │    │  │ 调拨单             │   │         │
│  │  │(InventoryHistory)│   │    │  │(MovementOrder)   │   │         │
│  │  └──────────────────┘   │    │  └──────────────────┘   │         │
│  └──────────────────────────┘    └──────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 核心能力

### 1. 入库管理

```
入库类型
  │
  ├── 采购入库
  │     → 供应商送货，验收入库
  │     → 关联 ERP 采购订单
  │
  ├── 销售退货入库
  │     → 客户退货，退回仓库
  │     → 关联 ERP 销售退货单
  │
  ├── 生产入库
  │     → 生产完工，成品入库
  │     → 关联 MES 生产工单
  │
  └── 其他入库
        → 调拨入库、盘盈入库等
```

### 2. 出库管理

```
出库类型
  │
  ├── 销售出库
  │     → 电商订单发货
  │     → 关联 ERP 销售订单
  │
  ├── 采购退货出库
  │     → 退回供应商
  │     → 关联 ERP 采购退货单
  │
  ├── 生产领料出库
  │     → 车间领料
  │     → 关联 MES 领料单
  │
  └── 其他出库
        → 调拨出库、盘亏出库等
```

### 3. 库存管理

```
库存管理
  │
  ├── 库存查询
  │     → 按 SKU + 仓库查询当前库存
  │
  ├── 库存变动历史
  │     → 每次入库/出库都记录变动明细
  │
  └── 库存预警
        → 库存低于安全库存时告警
```

### 4. 盘点管理

```
盘点流程
  │
  ├── 创建盘点单
  │     → 选择仓库、盘点范围
  │
  ├── 实盘录入
  │     → 录入每个 SKU 的实际数量
  │
  ├── 差异分析
  │     → 系统库存 vs 实际库存
  │     → 盈亏数量 = 实盘数量 - 账面数量
  │
  └── 盘点调整
        → 根据盘点结果调整库存
```

---

## 核心实体关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          WMS 核心实体关系                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  基础数据：                                                              │
│  WmsItemDO (商品) ── 1:N ── WmsItemSkuDO (SKU)                        │
│  WmsItemBrandDO (品牌)  WmsItemCategoryDO (分类)                       │
│  WmsWarehouseDO (仓库)  WmsMerchantDO (供应商/客户)                    │
│                                                                         │
│  入库线：                                                                │
│  WmsReceiptOrderDO (入库单)                                             │
│      └── 1:N ── WmsReceiptOrderDetailDO (入库明细)                     │
│                                                                         │
│  出库线：                                                                │
│  WmsShipmentOrderDO (出库单)                                            │
│      └── 1:N ── WmsShipmentOrderDetailDO (出库明细)                    │
│                                                                         │
│  库存线：                                                                │
│  WmsInventoryDO (库存) ──── skuId + warehouseId 联合唯一               │
│  WmsInventoryHistoryDO (库存变动历史)                                   │
│                                                                         │
│  盘点线：                                                                │
│  WmsCheckOrderDO (盘点单)                                               │
│      └── 1:N ── WmsCheckOrderDetailDO (盘点明细)                       │
│                                                                         │
│  调拨线：                                                                │
│  WmsMovementOrderDO (调拨单)                                            │
│      └── 1:N ── WmsMovementOrderDetailDO (调拨明细)                    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 库存实体

```java
// WmsInventoryDO - 库存
public class WmsInventoryDO extends BaseDO {
    private Long id;            // 库存编号
    private Long skuId;         // 商品 SKU 编号
    private Long warehouseId;   // 仓库编号
    private BigDecimal quantity; // 库存数量
    private String remark;      // 备注
}
```

**关键设计：`skuId + warehouseId` 联合唯一。**

同一个 SKU 在不同仓库有独立的库存记录。比如"蓝牙耳机-黑色"在 A 仓库有 100 个，在 B 仓库有 200 个。

### 库存与 ERP 的区别

| 维度 | WMS 库存 (WmsInventoryDO) | ERP 库存 (ErpStockDO) |
|------|--------------------------|----------------------|
| 粒度 | SKU 级别 | 产品级别 |
| 维度 | skuId + warehouseId | productId + warehouseId |
| 用途 | 仓库作业 | 财务对账 |
| 场景 | 发货时查库存 | 采购计划时查库存 |

---

## 入库单实体

```java
// WmsReceiptOrderDO - 入库单
public class WmsReceiptOrderDO extends BaseDO {
    private Long id;            // 入库单编号
    private String no;          // 入库单号
    private Integer type;       // 入库类型（采购入库/退货入库/其他）
    private LocalDateTime orderTime; // 单据日期
    private Integer status;     // 入库状态
    private String bizOrderNo;  // 业务订单号
    private Long merchantId;    // 供应商编号
    private Long warehouseId;   // 仓库编号
    private BigDecimal totalQuantity; // 总数量
    private BigDecimal totalPrice;    // 总金额
    private String remark;      // 备注
}
```

### 入库状态流转

```
入库状态
  │
  ├── 待入库 (0)
  │     → 入库单已创建
  │
  ├── 部分入库 (10)
  │     → 部分货物已入库
  │
  └── 已入库 (20)
        → 全部货物入库完成
```

---

## 出库单实体

```java
// WmsShipmentOrderDO - 出库单
public class WmsShipmentOrderDO extends BaseDO {
    private Long id;            // 出库单编号
    private String no;          // 出库单号
    private Integer type;       // 出库类型（销售出库/退货出库/其他）
    private LocalDateTime orderTime; // 单据日期
    private Integer status;     // 出库状态
    private String bizOrderNo;  // 业务订单号
    private Long merchantId;    // 客户编号
    private Long warehouseId;   // 仓库编号
    private BigDecimal totalQuantity; // 总数量
    private BigDecimal totalPrice;    // 总金额
    private String remark;      // 备注
}
```

---

## 盘点管理

### 盘点单实体

```java
// WmsCheckOrderDO - 盘点单
public class WmsCheckOrderDO extends BaseDO {
    private Long id;            // 盘点单编号
    private String no;          // 盘点单号
    private LocalDateTime orderTime; // 单据日期
    private Integer status;     // 盘点状态
    private Long warehouseId;   // 仓库编号
    private BigDecimal totalQuantity; // 盈亏数量
    private BigDecimal totalPrice;    // 账面金额
    private BigDecimal actualPrice;   // 实际金额
    private String remark;      // 备注
}
```

### 盘点流程

```
盘点流程
  │
  ├── 1. 创建盘点单
  │     └── 选择仓库、盘点范围（全盘/抽盘）
  │
  ├── 2. 生成盘点清单
  │     └── 系统自动列出该仓库的所有 SKU 及账面数量
  │
  ├── 3. 实盘录入
  │     └── 仓库人员逐一录入实际数量
  │
  ├── 4. 差异分析
  │     ├── 盘盈：实盘 > 账面
  │     └── 盘亏：实盘 < 账面
  │
  └── 5. 盘点审核
        ├── 通过 → 调整库存
        └── 驳回 → 重新盘点
```

---

## 调拨管理

### 调拨的业务场景

```
仓库 A（北京仓）                仓库 B（上海仓）
  库存：500 个                   库存：100 个
  │                              │
  │    调拨 200 个               │
  │ ─────────────────────────────>│
  │                              │
  库存：300 个                   库存：300 个
```

### 调拨单实体

```java
// WmsMovementOrderDO - 调拨单
public class WmsMovementOrderDO extends BaseDO {
    private Long id;                // 调拨单编号
    private String no;              // 调拨单号
    private Integer status;         // 调拨状态
    private Long fromWarehouseId;   // 源仓库
    private Long toWarehouseId;     // 目标仓库
    private LocalDateTime orderTime; // 单据日期
    private String remark;          // 备注
}
```

---

## 模块结构

```
yudao-module-wms/
├── yudao-module-wms-api/                    # API 模块
└── yudao-module-wms-server/                 # 服务实现
    └── src/main/java/.../wms/
        ├── controller/admin/
        │   ├── md/          # 基础数据（商品、SKU、仓库、供应商）
        │   ├── order/
        │   │   ├── receipt/  # 入库管理
        │   │   ├── shipment/ # 出库管理
        │   │   ├── check/    # 盘点管理
        │   │   └── movement/ # 调拨管理
        │   ├── inventory/   # 库存管理
        │   └── home/        # 首页统计
        ├── service/
        ├── dal/
        └── convert/
```

---

## 与其他模块的集成

```
WMS
  │
  ├── ◄── ERP（采购入库来源、销售出库来源）
  │
  ├── ◄── MES（生产领料、生产入库）
  │
  ├── ◄── System（用户、部门）
  │
  ├── ◄── Trade（电商订单 → 销售出库）
  │
  └── ──> 消息通知（库存预警、盘点结果）
```

---

## 思考题

1. WMS 的库存用 `skuId + warehouseId` 联合唯一，ERP 的库存用 `productId + warehouseId` 联合唯一。两套库存数据怎么同步？
2. 盘点时发现盘亏 5 个，这个差异应该算谁的责任？系统应该怎么记录？
3. 调拨单需要源仓库和目标仓库都确认吗？如果源仓库已出库但目标仓库还没入库，库存怎么算？
4. 入库单支持"部分入库"（分批入库）。出库单是否也需要支持"部分出库"？
