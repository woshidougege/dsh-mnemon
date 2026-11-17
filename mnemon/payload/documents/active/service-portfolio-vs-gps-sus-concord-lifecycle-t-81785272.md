---
id: "81785272-03ae-4941-bd30-ef5a6ea06067"
title: "Service Portfolio vs GPS/SUS 取数口径差异与 Concord/Lifecycle 真实数据核对（TST 实测）"
description: "TST 实库核对结论：SP 全量 332,138 PN vs GPS 口径 312,061（+6.4%）、Concord 113 vs 46 行；两个 Tab 主干数据逐项对上、无重复计算；in_scope / Big Events KPI / All HBB 三个待拍板口径；SUS 单复数、1,419 空财年、FY25/26 批量作废 35,340 等数据质量隐患；前端仍静态的四处。"
status: "active"
created_at: "2026-11-17T16:15:47.650Z"
updated_at: "2026-11-17T16:15:47.650Z"
content_hash: "8efcb5637e9ef11297b5c5fc83778c25461c98ce65fffb219e1bd5332626952e"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/dao/ServicePortfolioReadMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/constants/ServicePortfolioConstants.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/dao/GpsSusNewProductMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/dao/GpsSusActivePnMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/service/GpsSusService.java"
session_ids:
  - "mnemon-run-v1-a78f2b6a-1a9c-4f0e-b3c4-f6653c15e84c"
memory_body_ids:
  []
---

# Service Portfolio vs GPS/SUS 取数口径差异 + Concord/Lifecycle 真实数据核对（TST 实测）

调查时间 2026-11-18。仓库 `ssg-sales-ops`，分支工作树 `.agents/worktrees/sp-api-fix`。前置文档：
- `121d6b28-8c37-4caf-b9f6-8db4047c94a8` —— Lifecycle By Offering 复用 GPS/SUS active-pn 的会议结论
- `e7cdd3c7-ce26-4dc9-a8a3-8dcaee8b4c65` —— MR !569 接口与前端对接核对

本文只记增量：**"全 HBB 不筛选"在代码里落到哪几个参数、TST 实库差多少、两个 Tab 的真实数据是否对得上，以及仍待拍板的口径。**

## 1. 会议依据（录屏 07/08 章）

| 时间码 | 原话/决定 |
|---|---|
| 29:46 | 舒文：「我们这块**不限于 GPS 和 SUS 这两个 HBB**，我们是把所有的都需要展示。」 |
| 29:29 / 30:04 | 完全复用 GPS/SUS 现有展示，唯一区别是去掉 HBB 限制 |
| 31:24 | 「**108 是我们全部的**，但 GPS/SUS 的话肯定没那么多。」 |
| 34:05–34:15 | 现有 "All HBB" 其实只含 GPS+SUS，复用时必须改语义，否则老板误读 |
| 36:40–37:04 | Lifecycle 的 Overview 与 By Offering 同理，放开所有 HBB |

结论：这条需求对应 **Concord Projects 与 Lifecycle 两个 Tab**，不是 Big Events。

## 2. 代码层：筛选条件被拿掉的位置

GPS 页面（原页面，被"筛选过"）：

- `smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/dao/GpsSusNewProductMapper.java:31` —— Concord 查询写入 `WHERE hbb_l1 IN (hbbL1Scope)`（第 41 行声明参数 `@Param("hbbL1Scope")`）
- `smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/service/GpsSusService.java:167-168` —— `NEW_PRODUCT_HBB_L1_SCOPE = ["Global Product Services", "Sustainability Service"]`
- `smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/dao/GpsSusActivePnMapper.java:43-45` —— Lifecycle 三个硬条件：
  ```sql
  WHERE trim(lifecycle.business_unit) IN ('PCSD','ISG')
    AND trim(lifecycle.oh_l1_hbb) IN ('Global Product Services','Sustainability Services')
    AND trim(lifecycle.in_scope) = 'Y'
  ```

Service Portfolio（新页面）：

- `smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/dao/ServicePortfolioReadMapper.java:46-51` —— Lifecycle 只保留 `pn 非空 + in_scope='Y'`，`hbb`/`bu` 改为可选参数
- 同文件 `:66-67` —— Concord 使用 `WHERE 1=1` + 可选 `hbb`；不传即全量

→ HBB 与 BU 两层筛选确实已移除，实现方式与会议"复用同一套代码、只改取数"一致。

## 3. TST 实库（10.62.79.148）数量差

| 口径 | 数量 |
|---|---|
| GPS 页面口径（GPS+SUS & PCSD/ISG & in_scope=Y） | **312,061** PN |
| Service Portfolio 全量（仅 in_scope=Y） | **332,138** PN |
| 差值 | **+20,077（+6.4%）** |

按 HBB 拆（in_scope=Y）：GPS 309,434 · DWS 11,301 · SUS 5,961 · VS 2,987 · HCS 1,464 · Unmapped 767 · AI 199 · VC 25。

Concord 差得更多：GPS 口径 46 行（GPS 34 + Sustainability Service 12），**SP 113 行 = 2.4 倍**。所以 TST 上 113 是对的，不是 bug。

## 4. Concord Projects 数据核对（全部通过）

| 页面 | 库中重算 | 结果 |
|---|---|---|
| 113 行 | `count(*)`=113 | ✓ |
| Volume Summary GPS 34 / DWS 31 / HCS 22 / AI 12 / SUS 12 / VS 2 | 按 `hbb_l1` 分组同值 | ✓ |
| On Track 83 / Completed 9 / Not Started 11 / Delayed 4 / At Risk 6 | 83 / 9 / 11 / 4 / 6 | ✓ |
| Booking $244,046,410.5 | `sum(svcs_bookings)` 关联后 244,046,410.5009 | ✓（末位为 numeric 精度） |
| CR 72 行 / Project 41 行 | `cr_type` CR 72 / Project 41 | ✓ |

## 5. Lifecycle 数据核对（全部通过）

| 页面 | 库中重算 | 结果 |
|---|---|---|
| PCSD 46,746 | FY26/27 期末在册 46,746 | ✓ |
| ISG 225,877 | FY26/27 期末在册 225,877 | ✓ |
| 趋势表 FY26/27 YTD 277,949 | 滚动累计（created − obsolete）277,949 | ✓ |
| 趋势表 5,654 | 1,571,405,057 ÷ 277,916 = 5,653.6 → 5,654 | ✓ |
| By Offering 首行 GPS/Support Services 299,530（Released 253,248 / Obsolete 46,282） | 同值 | ✓ |

关键校验：`byOffering` 各行合计 332,138 **等于** `count(distinct pn)` 332,138 → 一张 PN 只挂一个 OH，合计无重复计算。

实现要点（`ServicePortfolioReadService`）：`lifecycleOverview` 的 totals 由分组行累加（`:288-295`）；Utilization 的在册数是复算近似——`runningActive += created − obsolete`（`:345-353`），源表只有创建/作废两个财年字段，不引入快照表；`FIRST_UTIL_FISCAL_YEAR = "FY2122"`（`:42`），`FY2627YTD` 与 `FY2627` 合并成一行（`:334-336`），避免多出一根柱；`parseRoadmapSlot` 解析 `'FY26/27 Q1'`（`:201-217`）。

## 6. 三个还没拍板的口径

1. **`in_scope='Y'` 仍保留。** 代码注释（`ServicePortfolioReadMapper.java:23-26`）称源表 2026-11-10 起同步全量 PN、`in_scope=N` 占三分之一、不过滤 KPI 会虚高 3 倍；但 TST 实测 **332,138 行全为 Y，N 为 0 行**。要么维度没同步（保护是空转），要么"不筛选"还要再涨。需韦强/相鹏确认。
2. **Big Events KPI 口径未定。** ACTIVE_PN 278K + PN_TERMINATED 54K = 332K 即全量；若要跟 GPS 页面对齐应是 312K 量级。会议未说 KPI 按哪个口径。
3. **"All HBB" 语义陷阱。** GPS 页的 "All HBB" 只含 GPS+SUS，SP 的 all 才是真全量；两页并排看，必然被问"为什么数不一样"。

## 7. 数据质量隐患

- **SUS 命名不一致**：concord 表叫 `Sustainability Service`（单数），lifecycle 表叫 `Sustainability Services`（复数）。GPS 页因此两种写法并存（concord 用单数常量、lifecycle 用复数硬编码），目前能对上；但 SP 的校验常量 `ServicePortfolioConstants.java:24-30` 用复数，若有人拿它去筛 concord 的 SUS，12 行会静默漏掉。
- **1,419 张 PN 的 `creation_fiscal_year` 为空**，被 `coalesce(nullif(trim(...),''),'FY0607')` 并入 FY0607 并持续滚动进累计在册，使基数偏大 1,419（页面从 FY21/22 起显示，肉眼看不出）。
- **FY25/26 作废 35,340 张**（其余年份仅 1–3 千），是真实批量 EOL 清理，非 bug，但图上柱子会异常高，业务方会问。
- **totals 用 sum 而非 distinct**：目前无重复；一旦一张 PN 挂两个 offering，合计即虚高。

## 8. 两个 Tab 里仍是静态/未接的部分

| 位置 | 状态 |
|---|---|
| EOL Review 两张判据卡 | 硬编码在 `EolReviewPanels.vue:16-51`，未接 `/eol-panels` |
| Reference Slides 8 张 | 原型 base64 静态图 |
| By Geo 表 | 接口 `byGeo` 有 7 行（WW 178,861 / AP 91,018 / NA 18,495 / EMEA 17,725 / LAS 16,821 / Others 7,576 / Unknown 1,642），但 `getLifecycleOverview` 悬空、页面未渲染 → 建议优先级 P1 |
| Concord 12 个筛选器 | 其中 8 个没有 `v-model`，是死控件 |
| Concord 主表 / 分布图 / Booking | 全真实 |

## 9. 一句话判读

主干数据（表格、分布、Booking、双卡、趋势图）**全真且逐项对得上**；周边装饰性模块（EOL 判据、幻灯片、By Geo、部分筛选器）仍为静态。**"数据对"≠"口径对"**——口径三项（in_scope / KPI 基数 / All HBB 语义）仍需业务方拍板后才能写进交付报告。
