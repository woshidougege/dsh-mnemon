---
id: "9b904808-5759-4d8a-af76-2e6d691ae8f4"
title: "Bookings HW Bookings 卡片单位 M→B 与 K/M/B 自适应格式化（含口径与实施 handoff）"
description: "HW Bookings 卡片的单位改 B：生产口径（ISG=PR Denominator、ISG 占 6%~15%、末日季度未跑完）、小数位自适应阈值、前后端改动点（hwWideCell/otherKpis.vue/GPSSUS/miniTable）、与 Qty 卡的差异（无需内嵌小图）、K/M/B 三级自适应的三条边界与“格式化收口到前端”的架构建议；代码未动，待用户决策。"
status: "active"
created_at: "2026-11-16T09:36:17.929Z"
updated_at: "2026-11-16T09:36:17.929Z"
content_hash: "2cb634cb1158cf78b9c57c46042495fcb3284d7e81389058f6f56b940432522e"
source_paths:
  - ".tmp/hwqty-mockup/hwqty-bookings-b.png"
  - ".tmp/hwqty-mockup/mockup.html"
session_ids:
  - "mnemon-run-v1-808e6500-f128-4816-82a5-0a9cea96dfc0"
memory_body_ids:
  []
---

# Bookings HW Bookings 卡片单位 M → B 与 K/M/B 自适应格式化（handoff）

> 关联文档：`e5fe9ed8-0b73-44be-b883-f39a4972b6a9`（HW **Qty** 单位：单轴按量级压缩方案 A 的落地与对比图）、`32e1bb91-5509-483c-b11d-ff85d7b016bd`（HW Qty 的 ISG 贴底与双轴/拆图/聚焦候选）。**那两篇只覆盖 Qty 卡片；本文记录同族但独立的 HW Bookings 卡片单位问题，以及由它引出的 K/M/B 三级自适应讨论。**三篇结论可叠加，勿互相替代。

## 1. 数据口径（生产库已核实，与用户截图一致）

- PCSD / MBG 的 Bookings = `kpi='HW Bookings'`；**ISG 的 Bookings = `kpi='PR Denominator'`**。
  落在代码里就是 `BookingsService.java:480-485` 的 `hwBookingsPredicate`：`"ISG".equals(bu) ? BookingsKpiNames.PR_DENOMINATOR : BookingsKpiNames.HW_BOOKINGS`。
- 不排除 NEC；季度窗口 11 个（FY2425Q1 ~ FY2627Q3）。
- **最后一季 FY2627Q3 未跑完**，因此曲线末端下坠是正常的，不是数据错误（曾被当成异常追问过）。
- 生产量级（HW Bookings，单位 M）：FY2425Q1 PCSD 6,398.7 / ISG 557.9 / MBG 2,206.2 / Total 9,162.8；FY2627Q2 高点 Total 15,063.2（PCSD 9,874.6 / ISG 2,251.9 / MBG 2,936.7）。**ISG 占 Total 约 6%~15%**。

## 2. 结论：Bookings 只改单位，不需要内嵌小图

与 Qty 的关键差异：Qty 里 ISG 只有 PCSD 的约 1/250，主图必然贴底，才需要双轴/拆图/卡内嵌小图；Bookings 里 ISG 与 PCSD 只差约 16 倍，主图上就能看见走势，加内嵌小图属于"能加但没必要"。

推荐格式：单位统一 `$B`，**小数位自适应**——`≥10B 用 1 位小数，<10B 用 2 位小数`。

| 指标 | 现状 $M | 改 $B |
|---|---|---|
| Total | 15,063.2 | 15.1B |
| PCSD | 9,874.6 | 9.87B |
| MBG | 2,936.7 | 2.94B |
| ISG | 2,251.9 | 2.25B |
| ISG 低点 | 557.9 | 0.56B |

小数位是硬约束：只用 1 位会把 ISG 的 557.9 / 582.5 / 593.1 全压成 `0.6B`，2 位才能分出 0.56 / 0.58 / 0.59。

## 3. 实施要点（只改前端不够）

1. **后端**：`BookingsService.java:776-802` 的 `hwWideCell` 里 idx=0（HW Bookings）目前写死 `toMillionsUsd`，需增 `toBillionsUsd`（或按量级选单位）。接口 `GET /admin/bookings/hw-kpi-summary`（`BookingsController.java:128`）JSON 里不带单位，单位是前端配置的 → 必须前后端一起改，否则会出现"图上 15.1B、后端仍是 M"两套事实。
2. **前端两处**：`src/views/Bookings/otherKpis.vue:178-181` 的 `resolveMetricValueDisplay`、`src/views/GPSSUS/index.vue:2296-2301` 的 `hardwareMetricValueDisplay`，把 HW Bookings 的 unit 从 `$M` 改成 `$B`；`src/views/Bookings/components/miniTable.vue` 的 compact 逻辑要能按 "B" 截断小数位。
3. **Show Data 表格不要跟着变**：导出与核对仍要 9,874.6 这类 M 值。表格与图形用两套格式是项目常态。
4. **Y 轴负刻度**：现有 M 图底部出现 -2,000（grace 造成），换 B 会变成 -2B，需顺手处理 `beginAtZero`，否则会被误读成存在负 bookings。
5. **AUR 卡不受影响**（$/台），但它的分子是 Bookings，口径别跟着一起改。

## 4. K/M/B 三级自适应：能自动判断，但只有"值"能自适应

规则按**每个值**执行、阈值固定：`≥1e9 → x.xx B`、`≥1e6 → x.xx M`、`≥1e3 → x.x K`、更小原数。同图 PCSD 显示 13.3M、ISG 显示 52.6K 是成立的；Qty 那张已在做 K→M，扩成 K/M/B 只是多一级。

三条必须提前接受的边界：

1. **Y 轴刻度不能混单位**。轴按该图的"基准单位"走（Bookings=B、Qty=M），只有数据标签按值自适应。轴也自适应会让读者无法判断斜率和比例。
2. **同一指标的单位必须稳定，不随筛选跳**。不能按"当前最大值决定单位"——选 GEO=AP 出 M、切回全球变 B，用户会以为数据变了。
3. **它解决"数字太长"，不解决"量级差太多"**。Qty 里 ISG 标签写成 52.6K 了，线仍贴 X 轴，那是几何问题（见 `32e1bb91`）；要么拆图、要么双轴、要么点击聚焦。

**架构建议（趁改的时候收口）**：现状是"后端定小数位（`toMillions`/`toThousands`）+ 前端定单位字符串"，做三级自适应会变成前后端各写一套规则、迟早打架。建议改为**后端只返回原始数值，格式化统一放前端**（或接口直接返回 `{value, unit}`），让图表标签、Show Data、tooltip 共用一个函数，避免"图上 15.1B、表格里 15,063.2"这类互相对不上。以后加 B 或调阈值只改一处。

若目标真是"不同 BU 量级差太多也能一眼比"，自动单位帮不上，得换图形：**每个 BU 一行的小倍数表（BU 名 + 数值 + 独立缩放的 sparkline）**，量级差被隔离在行内，行间只比涨跌形态。

## 5. 当前状态（未动代码）

- 用户先答"那就改这 2 个吧，按照推荐的改"，**随即改口"先别改哈"，转去问能否自动判断 B/M/K**。因此**代码尚未改动**，第 3 节只是未经实施的方案。
- 助手最后给出的岔路待用户选择：(a) 先写一个 K/M/B 三级格式化函数（含负数与千分位）供团队评审，或 (b) 先出一版小倍数表格样式对比。
- 已交付评审图：`.tmp/hwqty-mockup/hwqty-bookings-b.png`（M vs B 对比，最终 5400×2100，源 mockup 在 `.tmp/hwqty-mockup/mockup.html`）。用户明确要求**图要超高清、放大能看细节**，后期版本从 3600×1400 一路提到 5400×2100。
- 面向群里的答复模板：Bookings 的 ISG 只差 16 倍、主图看得见，内嵌小图"能加但没必要"；Qty 差 250 倍才必须内嵌或拆图。
