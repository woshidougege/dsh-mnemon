---
id: "98248233-94f5-434d-8021-39a475145154"
title: "HW KPI 动态单位：实施落地与双 MR 交付（含 develop 冲突处置）"
description: "HW Bookings/Qty 动态单位从方案落成代码并交付后的实况：hwFinishBlock 选档与 MathContext(6) 精度、MetricBlock 四字段、前端 metricNumberFormat/hwMetricDisplay 与 miniTable inset、单测与 type-check 结果、三 MR 拓扑（!560 admin-release / !1058 develop / !1057 master）、develop 落后 master 22 提交致 feedbackMenuCatalog.ts 恒冲突的一次性分支解法、I&C 上生产两处开关、TST 联合验收待办与 inset 遮挡取舍。"
status: "active"
created_at: "2026-11-16T10:35:15.382Z"
updated_at: "2026-11-16T10:35:15.382Z"
content_hash: "3565577c564ffff256b9cd4a2dc7b309ae1e8969717e47765e7225bcefe7d8f7"
source_paths:
  - ".tmp/hwqty-mockup/unit-design.md"
  - ".tmp/hwqty-mockup/qty-inset-preview.png"
  - "src/utils/metricNumberFormat.ts"
  - "src/utils/hwMetricDisplay.ts"
  - "src/views/Bookings/components/miniTable.vue"
  - "src/constants/feedbackMenuCatalog.ts"
  - "src/views/BiAdmin/moduleTargets.ts"
session_ids:
  - "mnemon-run-v1-8824dc63-d48b-4ab4-b878-2d591f1d682b"
memory_body_ids:
  []
---

## 状态

方案（见 `b881d9ec`：契约、阈值、块级 vs 值级）已**全部落地并推送**，三个 MR 全部 mergeable、无冲突。合并动作按项目惯例留给用户，助手不点合并。

## 落地实现（与方案文档的差异点）

后端 `ssg-sales-ops`，分支 `feature/hw-kpi-dynamic-unit-admin`（从 `origin/admin-release` 建）：
- `smartop-service/admin/src/main/java/com/lenovo/admin/bookings/service/BookingsService.java`：不再固定 `toMillions`，改为先返回原始高精度值，再由 `hwFinishBlock` 按块内最大绝对值选档（≥1e9 → `$B`、≥1e6 → `$M`、≥1e3 → `$K`），选出的单位写入 `unit / unitScale`；数值用 `new MathContext(6, HALF_UP)` 保留 6 位有效数字（前端降档才有的可降）。方案里该步骤所在的 `hwWideBlock` 实际对应 `hwFinishBlock`。
- `hw_qty` 且 BU 含 ISG 时额外下发 `inset{series:ISG, unit:K, unitScale:1000, hideInMain:true}`；按维度展开的视图不带 inset。
- `BookingsOperationalKpiSummaryResp.MetricBlock` 新增 `unit / unitScale / format / inset` 四个字段。
- 单测 `BookingsServiceHwKpiTest` 重写为 5 个用例（缺值不渲染 0、0 值、原始精度、选档阈值、AUR 不换算），`BUILD SUCCESS`。

前端 `ssg_sales_ops`：
- 新增 `src/utils/metricNumberFormat.ts`（还原量级 + B/M/K 压缩，≥10 保留 1 位、<10 保留 2 位）与 `src/utils/hwMetricDisplay.ts`（统一从 `hw-kpi-summary` 的 `unit/unitScale/format/inset` 生成展示配置；缺这些字段的老接口退回按标题猜单位，行为不变）。注意文件名与方案草稿里的 `formatNumber.ts / formatCompact` 不同。
- `src/views/Bookings/components/miniTable.vue`：轴标题读块单位，数据标签/坐标轴/Show Data 统一压缩格式；新增 inset 内嵌小图（主图隐藏该序列标签，小图按 K 单独画右下角）。
- `src/views/Bookings/otherKpis.vue`、`src/views/GPSSUS/index.vue`：删掉 `HW Bookings=$M`、`HW Qty=K` 硬编码，改透传后端字段。
- `npm run type-check`（`vue-tsc --build`）通过。

## 分支与 MR 拓扑

| MR | 源分支 | 目标 | 内容 |
|---|---|---|---|
| `!560` | `feature/hw-kpi-dynamic-unit-admin` | admin-release | 后端 3 files（`c411cdccd`） |
| `!1058` | `tmp/hw-kpi-unit-develop` | develop | 前端 6 files（`3a960a1e`） |
| `!1057` | `dev/renxp4-hw-kpi-unit` | master | 前端 5 files（`ccabdd4f`，develop 版 rebase 后源） |

合并顺序：先 `!1058`（develop）再 `!1057`（master），都不勾 Delete source；`!1058` 合完后手动删 `tmp/hw-kpi-unit-develop`。

**为什么要一次性分支而不是直接 `dev/renxp4`**：
1. `dev/renxp4` 上还压着一条未交付的 Jira 设置页提交（`318a5fea`），直接用它交付会互相牵连，故另起 `dev/renxp4-hw-kpi-unit`（从最新 `origin/master` 拉出，只含本轮提交）。
2. `develop` 落后 `master` 22 个提交，`dev/renxp4-hw-kpi-unit → develop` 必然在 `src/constants/feedbackMenuCatalog.ts` 冲突（master 用注释关掉 I&C 菜单项、develop 开着），故又起 `tmp/hw-kpi-unit-develop`（= 最新 `origin/develop` + 本轮改动）解冲突。首版冲突的 MR `!1056` 已关闭。
3. 该冲突的解法：**保留 develop 的 I&C 开启项**，同时保留 master 那条「I&C 上生产要同步打开 `moduleTargets.ts` 的 `PAGE_PATHS.ic`」注释。该文件相对 develop 只多 2 行注释，行为不变。副作用：`!1058` 的 diff 会带上 master 已合入而 develop 未有的提交。

**I&C 上生产需同时改两处**（缺一处则标签能点但跳不过去）：
- `src/constants/feedbackMenuCatalog.ts` 里 I&C 段取消注释；
- `src/views/BiAdmin/moduleTargets.ts` 的 `PAGE_PATHS` 补 `ic: '/spark/ic'`。

## 验证与遗留

- 已用真实量级渲染核对：Qty 主图 `26.2M / 13.3M`、标签 5 字符；Bookings `15.6B / 9.82B / 2.25B`，ISG 降档 `558M`；内嵌小图 ISG 按 K 显示 `44.5K~58.2K`（预览图 `.tmp/hwqty-mockup/qty-inset-preview.png`）。
- 页面实测未做，待后端 `!560` 与前端一起部署到 TST 后验收；**前后端必须一起上**，只上前端会退回旧的按标题猜单位（显示同现状，不白屏）。
- `hw-kpi-summary` 被 Bookings 页与 GPSSUS 页共用，两页的 HW Bookings / HW Qty 卡会一起变；HW AUR 不变（`format=PLAIN`）。
- 已知固有取舍：内嵌小图会盖住 MBG 最近 2~3 季的数字标签，数值改从 tooltip / Show Data 看。
