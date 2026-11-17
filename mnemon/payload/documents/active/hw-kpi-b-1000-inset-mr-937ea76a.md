---
id: "937ea76a-91fb-4584-95e9-ec7972b3f2da"
title: "HW KPI 单位自适应实施落地：金额 B 档 1000 亿、标签抽稀、inset 放大（三 MR 交接）"
description: "b881d9ec/f5b15acb 之后的实施结果：金额 B 门槛由推荐 1e10 修正为 1e11（因 Bookings 常态 90~150 亿）、数量档仍 1e9；后端 hwUnitScale(idx,maxAbs)/hwUnitLabel>= 改造；前端取消值级降档、≥9 季度或 ≥4 序列隔点显示标签、i 提示、ISG inset 点击放大到 920px 可切 4/8/全部；三个 MR(!1057/!1069/!572) 与验证、遗留项。"
status: "active"
created_at: "2026-11-17T10:21:56.236Z"
updated_at: "2026-11-17T10:21:56.236Z"
content_hash: "d00bd387b0fe25a7fead2950839917f87ca60b12f8ea09cbaca74b13b916211d"
source_paths:
  - "src/utils/metricNumberFormat.ts"
  - "src/utils/hwMetricDisplay.ts"
  - "src/views/Bookings/components/miniTable.vue"
  - "src/views/Bookings/otherKpis.vue"
  - "src/views/GPSSUS/index.vue"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/bookings/service/BookingsService.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/bookings/service/BookingsServiceHwKpiTest.java"
session_ids:
  - "mnemon-run-v1-b0a7c7a8-9a3a-4f8b-83b8-5a696f8281bc"
memory_body_ids:
  []
---

# HW KPI 单位自适应落地：金额 B 档 1000 亿 + 标签抽稀 + inset 放大（三 MR 交接）

状态（2026-11-17 18:20）：代码已改完并推送，三个 MR 均 `can_be_merged`，等用户合并。本文件是 `b881d9ec`（v2 方案）与 `f5b15acb`（TST 反馈：轴回 $M + inset 读不全）之后的**实施结果**，二者"代码未动/待确认"的描述已被本次覆盖。

## 1. 关键决策修正：金额 B 门槛 = 1000 亿（1e11），不是 1e10

`f5b15acb` 当时推荐"后端 B 门槛 1e9 → 1e10（100 亿）"。实测不成立：Bookings Total 常态 **90~150 亿**（FY2627Q2 为 15,063.2M = 150.6 亿），1e10 门槛下 150.6 亿 > 100 亿，仍会判成 `$B`（15.1B），与同页顶部 `$1,396.4M` 口径照样打架。

最终取 **1e11（1000 亿）**：
- 金额（idx=0）升 B 需 >1000 亿，日常稳定停在 `$M` 显示 15,063.2；
- 数量（台/件）的 B 档仍是 **1e9（10 亿）**，与金额解耦；
- 另有一条精度理由：0.1B = 1 亿，B 档对 Bookings 粒度太粗。

## 2. 后端改动

`smartop-service/admin/src/main/java/com/lenovo/admin/bookings/service/BookingsService.java`

- 新增 `private static final BigDecimal HW_UNIT_BILLION_AMOUNT = new BigDecimal("10000000000")` → 最终值为 1e11（1000 亿），与 `HW_UNIT_BILLION = 1e9`（数量档）、`HW_UNIT_MILLION = 1e6`、`HW_UNIT_THOUSAND = 1e3` 并列。
- 签名变更：`static BigDecimal hwUnitScale(BigDecimal maxAbs)` → **`static BigDecimal hwUnitScale(int idx, BigDecimal maxAbs)`**，内部 `BigDecimal billionScale = idx == 0 ? HW_UNIT_BILLION_AMOUNT : HW_UNIT_BILLION;`。
- 调用点 `hwFinishBlock` 内：`BigDecimal scale = idx == 2 ? BigDecimal.ONE : hwUnitScale(idx, maxAbs);`（idx=2 是 AUR 比率，不换算）。
- `static String hwUnitLabel(int idx, BigDecimal scale)` 的 B 判断由 `== HW_UNIT_BILLION` 改为 **`>= HW_UNIT_BILLION`**，否则 1e11 档取不到 "B" 后缀；金额带 `$`、数量不带。

前端前提（沿用上一轮已合入 !560 的改动）：后端 `hwQtyKeepsRawCountSoFrontendCanCompactLabels` 保证 Qty 返回原始台件数（如 5,044,536），不再预压成 1 位小数——否则 ISG 在 M 刻度下先变 0.1，前端再也变不回 52.6K。

## 3. 前端改动

分支 `dev/renxp4-hw-kpi-unit`（commit `cf940ab8`），及等价分支 `feature/hw-kpi-unit-develop`（`7790124b`，基于 origin/develop）。两处 commit 的 `patch-id` 相同（`c9396319a4587128df08c566223b5567b0eade8f`），即 master 与 develop 内容一致。

涉及文件（master 侧两轮累计 5 个）：
- `src/utils/metricNumberFormat.ts`（+70）
- `src/utils/hwMetricDisplay.ts`（+51）
- `src/views/Bookings/components/miniTable.vue`（+358/-…，主体）
- `src/views/Bookings/otherKpis.vue`
- `src/views/GPSSUS/index.vue`

三条行为规则：
1. **取消值级降档**：标签一律与 Y 轴同单位。不再出现"轴写 `$B`、标签写 557.9M"。`formatBlockUnitValue` 小数位随量级变：≥10 保留 1 位、≥0.1 保留 2 位、其余保留 3 位。
2. **标签抽稀**：季度序列 ≥9 个、或序列 ≥4 条时，每隔一个数据点才显示标签，**首尾始终显示**。13 个季度下 8 字符长数字不再互相遮挡。
3. **单位自适应提示**：卡片标题旁加小 `i`，hover 显示英文说明（轴单位随量级切换、标签与轴同单位），**仅在自动单位模式下出现**——目的是避免用户换筛选条件后截图单位不同被当 bug 上报。
4. **ISG inset 可放大**：小图（带 ⤢ 角标）点击弹 920px 弹窗，可切最近 4 / 8 / 全部季度；**小图本体尺寸不变**（仍是 `miniTable.vue` 里 134×68 的 `.bk-opkpi-inset`），即用弹窗替代"把 inset 撑大"来解 `f5b15acb` 记录的 11 点 / 每点约 11px 的密度问题。

## 4. MR 清单

| 仓库 | MR | source → target | 状态 |
|---|---|---|---|
| ssg_sales_ops | [!1057](https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1057) | dev/renxp4-hw-kpi-unit → master | opened，可合并，无冲突（两轮累计 5 文件） |
| ssg_sales_ops | [!1069](https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1069) | feature/hw-kpi-unit-develop → develop | opened，可合并，无冲突（本轮 2 文件） |
| ssg-sales-ops | [!572](https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/572) | feature/hw-kpi-dynamic-unit-admin → admin-release | opened，可合并，无冲突（2 文件） |

后端上一轮 MR !560（→ admin-release）已 merged，故本轮重开 !572，commit `92233773e`「feat(admin): 金额 B 档门槛抬到 1000 亿，与数量分档解耦」。前端 master 的 MR 累积两轮（上一轮未合）。

## 5. 验证

- 前端两分支各跑 `npm run type-check`（即 `vue-tsc --build`）通过。
- 后端 `BookingsServiceHwKpiTest` **5/5 通过**（`Tests run: 5, Failures: 0, Errors: 0`，2026-11-17 18:18:58）。相关用例：`blockUnitFollowsLargestValueInBlock`（断言 150.6 亿仍在 M 档、200 亿升档、数量仍 1e9）、`hwQtyKeepsRawCountSoFrontendCanCompactLabels`、`actualZeroMetricValueStillRendersZero`、`aurRequiresBothMetricValues`。
- 中途踩坑两条（非缺陷）：
  1. 后端首轮编译失败于 `CocFlagSyncScheduleHandler.java:71/127`：`找不到符号 cocFlagSync(String) / cocFlagSyncStatus()`，因为本地 `smartop-api` 未先 install；先 `-o -T 4 -pl smartop-api -am install -DskipTests` 才通。
  2. 首版测试红：`blockUnitFollowsLargestValueInBlock:49 expected: 1000000 but was: 10000000000` —— 即 1e10 门槛版本的行为不符预期，推动最终改到 1e11。
- 环境：本机 PATH 无 `mvn`/`mvnd`，用 IDEA 自带 Maven（`C:\Program Files\JetBrains\IntelliJ IDEA 2026.1.3\plugins\maven-plugin\lib\maven3\bin\mvn.cmd`），必须 `JAVA_HOME=C:\Program Files\Java\jdk-17.0.19`（JBR 25 会让 lombok 失效，报找不到 log/getter）。

## 6. 遗留

- **页面回归未做**（按约定非必需）。合入 TST 后需目视确认：Bookings 卡 `$M` + `15,063.2` 类标签；Qty 卡 `M` 轴 + 隔点标签 + 右下角 `ISG · K` 小图；点小图能放大切 4/8/全部。
- 门槛 1e11 属经验值，若未来 Bookings 体量整体上到千亿级需重估（当前 0.1B=1 亿的粗粒度问题仍会在 >1000 亿时复现）。
- 合并/部署由用户执行，本文件不代表已上线。

## 相关文档
- `b881d9ec-57e3-4636-bf0a-5c10604822a9` HW KPI 单位方案 v2（接口契约 `{unit, unitScale, format:AUTO, inset}`、块级 vs 值级区分）
- `f5b15acb-8bb5-49fa-9f28-4958f4928c5f` TST 反馈：inset 读不全与 Bookings 轴回 $M（量化根因 134×68/每点 11px）
- `9b904808-5759-4d8a-af76-2e6d691ae8f4` 更早的 M→B 与 K/M/B 自适应 handoff
