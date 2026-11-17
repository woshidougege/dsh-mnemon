---
id: "0200947d-78c0-4b83-905e-cacea7c7ad11"
title: "HW KPI 图上带单位落地记录：自绘双字号标签、块单位不降档与三个 MR 状态"
description: "HW Bookings/Qty 图表标签改成“数值带单位、单位弱化”后的最终实现与验证边界：chartjs-plugin-datalabels 不支持双字号故改为自绘 buildCompactLabelsPlugin、display:'auto' 消失后改用按季度倒序占位+包围盒判重、取消值级降档（后端撤销 1e11 金额门槛回到 ≥1e9）、tooltip 与 Show Data 同口径、MR !1070(master)/!1071(develop)/!573(admin-release) 提交号与 patch-id、type-check 与单测通过但页面实测待 TST、FY2627Q3 未跑完导致标签拥挤的已知取舍。"
status: "active"
created_at: "2026-11-17T11:57:49.910Z"
updated_at: "2026-11-17T11:57:49.910Z"
content_hash: "a06805842131c5559db2ce571da8453581028c5065b4b63b9e0435b93599d344"
source_paths:
  - "src/views/Bookings/components/miniTable.vue"
  - "src/utils/metricNumberFormat.ts"
  - "src/utils/hwMetricDisplay.ts"
  - ".tmp/mr-master-desc.md"
  - ".tmp/hwqty-mockup/final-labels.png"
session_ids:
  - "mnemon-run-v1-38c5dc49-f256-4d22-ac93-cbfbb11c68f2"
memory_body_ids:
  []
---

# HW KPI 图上带单位：落地收尾记录（自绘双字号标签 + 块单位不降档）

> 前置方案见 `b881d9ec-57e3-4636-bf0a-5c10604822a9`（v2 契约与改动清单），本文只记录**实际落地后的最终形态、与方案的差异、MR 与验证状态**。

## 1. 用户拍板与最终表现

用户在 B 方案（动态单位）上定稿：「B 比较好吧 他能动态显示单位」，代价是多一个字符，用**单位缩小、弱化**来吸收。最终效果（`.tmp/hwqty-mockup/final-labels.png`）：

- 块单位卡片（`format=AUTO`）标签自带单位：`15.0B` / `25.4M` / `52.6K`，金额带 `$` 前缀（`$15.0B`）；
- Y 轴标题不再写单位（标签已重复），但保留 `i` 提示说明「单位随数据量级自动切换」以防截图间单位不同被误判为错误；
- tooltip 与 Show Data 表格同口径带单位，消除「图里 15.0B、表里 15,015.9」。

## 2. 关键实现（与 v2 方案的差异）

1. **自绘标签替代 chartjs-plugin-datalabels**：该插件不支持一段文字用两套字号，因此主图标签改为自写 plugin `buildCompactLabelsPlugin`（`miniTable.vue`）：数值 9.5px 加粗深灰，单位 7px、62% 透明度、基线略下移 1px，让单位视觉上从属于数字。
2. **重叠让位不再依赖 `display:'auto'`**：自绘后失去该能力，改为按季度**倒序**占位 + 包围盒判重 → 最右边（最新）数据优先看清，比原先随机隐藏合理。
3. **取消值级降档**：块单位卡片统一走 `formatBlockUnitValue`，标签与轴共用块单位，不再出现「$B 轴里蹦出 557.9M」的混搭（v2 里允许的「值级降档」被撤回）。
4. **后端同步撤销金额独立门槛**：删掉 `HW_UNIT_BILLION_AMOUNT`(1e11)，`hwUnitScale(BigDecimal maxAbs)` 还原，金额与数量共用 `≥1e9` B 档 —— 因为 `$M` 下 15,015.9 这种 8 字符长数字反而不如 15.1B 直观。
5. **未改动**：非块单位卡（percent、固定 `$M`、AUR `format=PLAIN`）继续用 `ChartDataLabels` + 轴标题；GPSSUS 其它卡不受影响；ISG 内嵌小图逻辑保持（角标态不画数值、点击卡片内原地展开 320×196、`Last 4 / Last 8 / All` 切换）。

## 3. 改动文件

- `src/views/Bookings/components/miniTable.vue`（+106/−6）
- `src/utils/metricNumberFormat.ts`（`formatCompactMetricValue` 保留 B/M/K 压缩；`formatBlockUnitValue` 不加后缀直接展示块单位值）
- `src/utils/hwMetricDisplay.ts`（由 `hw-kpi-summary` 的 `unit/unitScale/format/inset` 生成展示配置；缺字段的接口退回按标题猜单位）
- `src/views/Bookings/otherKpis.vue`、`src/views/GPSSUS/index.vue`（删掉 `HW Bookings=$M` / `HW Qty=K` 硬编码）

## 4. MR 与验证状态

| 仓库 | MR | 目标 | 提交 |
|---|---|---|---|
| ssg_sales_ops（前端） | !1070 | `master` | `bb23d301` |
| ssg_sales_ops（前端） | !1071 | `develop` | `7db57a17`（cherry-pick，patch-id `25281f62` 与 dev 一致） |
| ssg-sales-ops（后端） | !573 | `admin-release` | `5314c409` |

三个均 `opened / can_be_merged / 无冲突`；标题与描述已重写为本轮最终形态（前端两条描述一致，后端写明前后端配套关系）。

验证边界：
- 两个目标分支各跑一次 `npm run type-check`（`vue-tsc --build`）：通过；
- 后端 `BookingsServiceHwKpiTest` 5 个用例：通过（本轮未再改后端）；
- 用生产数据按组件同一套算法渲染预览图核对 `$` 前缀、单位淡化、首点右对齐、边界裁剪；
- **页面实测未做**，待合到 TST 部署后验收三点：Bookings 标签形如 `$15.0B`、Qty 角标小图干净、点开后 `Last 4/8/All` 切换正常。

## 5. 已知取舍与分支说明

- 最右 FY2627Q3 数据未跑完，四条线几乎落在同一点，判重后只显示一两个标签；属数据问题，季度跑完自然散开。
- 本轮源分支是一次性分支 `dev/renxp4-hw-kpi-unit`（从最新 `origin/master` 拉出，只含本提交），原因是 `dev/renxp4` 上压着未交付的 Jira 设置页提交，避免相互牵连。
