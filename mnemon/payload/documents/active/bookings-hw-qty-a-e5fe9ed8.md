---
id: "e5fe9ed8-0b73-44be-b883-f39a4972b6a9"
title: "Bookings HW Qty 图表单位：单轴按量级压缩（方案 A）设计与效果验证"
description: "HW Qty 图表为何不能用“按 BU 各一套单位”：单轴 4 线（PCSD/ISG/MBG/Total）约束、生产数据量级、A/B/C/D 方案对比、方案 A 的 miniTable.vue compactValue 落地细节与已交付对比效果图（.tmp/hwqty-mockup/）。"
status: "active"
created_at: "2026-11-13T07:49:45.040Z"
updated_at: "2026-11-13T07:49:45.040Z"
content_hash: "7bcd9546f62737897aec1ecbc5a92bf4343059f73baec64103b6b734691d3b0a"
source_paths:
  - ".tmp/hwqty-mockup/mockup.html"
session_ids:
  - "mnemon-run-v1-78e81db1-565a-48f8-8c9e-9803b39700aa"
memory_body_ids:
  []
---

# Bookings HW Qty 图表单位：单轴“按量级压缩”方案（A）设计与效果验证

## 1. 背景与现状
- 后端把 HW Qty 统一除以 1000 返回 K：`smartop-service/admin/src/main/java/com/lenovo/admin/bookings/service/BookingsService.java:796-801`，接口 `GET /admin/bookings/hw-kpi-summary`（Controller 见 `BookingsController.java:128-131`，Service 入口 `BookingsService.java:553-561`）。
- 前端两处把单位写死成 `K`：`src/views/Bookings/otherKpis.vue:178-181`、`src/views/GPSSUS/index.vue:2296-2301`。
- 图表是 `src/views/Bookings/components/miniTable.vue:112-213`：`rows` 直接作为 series 画折线，**只有一根 Y 轴**，单位取 `valueDisplay.unit`（:207）。
- 关键约束：`rows` 就是 BUs —— `PCSD / ISG / MBG / Total`（`BookingsService.java:736-770`）；默认 BU 过滤器为空 → 三选 → 图上同时 4 条线。因此这不是“一个 BU 一张图”。

## 2. 核心结论：按 BU 各用一套单位不成立
同一根线性轴上若 PCSD 画 13.3（M）、ISG 画 52.6（K），视觉上 ISG 是 PCSD 的 4 倍，实际只有 1/250 —— 图形错误、跨 BU 对比失效。要“真按 BU 分单位”只能双 Y 轴或拆图，都要改布局/刻度且牺牲同轴对比。
另一反直觉点：单位不改变 ISG 线形 —— ISG 仅 PCSD 的约 1/250（占比 0.4%），M 或 K 都贴着 0 走；改 K 只救回“数字”，没救回“趋势”。

## 3. 生产数据量级（生产库 mv_bookings_agg，kpi='HW Qty'，已排除 NEC，FY2526Q3）
| 行 | 原始台件 | 现状（K） | 全改 M | 方案 A |
|---|---|---|---|---|
| PCSD | 13,253,873 | 13,253.9 | 13.3 | 13.3M |
| MBG | 15,609,229 | 15,609.2 | 15.6 | 15.6M |
| ISG | 52,575 | 52.6 | 0.1 | 52.6K |
| Total | 28,915,677 | 28,915.7 | 28.9 | 28.9M |

ISG 近 11 个季度全改 M 后为 `0.1 0.1 0.1 0.1 0.0 0.1 0.1 0.0 0.0 0.0 0.0`，近乎“0 线”—— 这正是当初 M→K 的原因。PCSD/MBG 用 K 则是 7 位数字、小数位全是噪声（0.1K = 100 台，对千万台件无意义）。矛盾本质是“位数太长”，解法是保留 3 位有效数字。

## 4. 四个方案
| 方案 | 做法 | 改动面 | 代价/风险 |
|---|---|---|---|
| **A（推荐）** | 数据仍按 K 返回；标签/tooltip 中 ≥1000K 显示 `x.xM`，否则 `xx.xK`；轴刻度同样压缩、轴标题动态 `(K)/(M)` | 只改 `miniTable.vue` + 两处 `valueDisplay` 配置 | 后端不动，已合的 3 个分支 MR 不必回滚；图上 M/K 后缀混排 |
| B | 双 Y 轴：ISG 挂右轴(K)，其余左轴(M) | `miniTable` 图表配置 + `Show Data` 表按行带单位 | 1~1.5 天；两轴易被误读成同量级；同列混单位 |
| C | 拆两张图：PCSD/MBG 一张(M)，ISG 单独一张(K) | `otherKpis.vue` + GPSSUS 模板 + 卡片栅格 | 1.5~2 天；打破 2×2 栅格；失去一眼对比 |
| D | 后端按 BU 返不同单位 | 后端 + 前端需加“单位字段” | 不推荐：同一列两种量纲，导出/问数/后续消费方都有 1000 倍坑；须与 C 绑定才有意义 |

## 5. 方案 A 落地细节
1. `valueDisplay` 增加仅 Qty 用的标记：`{ mode:'number', scale:1, unit:'K', compact:'K' }`，含义为“数据单位是 K，展示时按量级压缩”。
2. `miniTable.vue` 新增 `compactValue()`，供 `datalabels`/`tooltip` 使用：`≥1000 → (v/1000).toFixed(1)+'M'`，否则 `v.toFixed(1)+'K'`。
3. Y 轴：`max ≥ 1000` 时刻度也压缩、轴标题 `Qty (M)`；只选 ISG 时维持 `Qty (K)`。
4. `Show Data` 表格保留精确值（如 13,253.9）。
5. 项目已有同类先例：`src/utils/util.ts` 的 `formatMillionData`、`drivers.ts`、`data.ts` 均为 `≥1e6→M / ≥1e3→K`；只有这张图是死单位。

## 6. 效果验证（已交付）
- 静态验证页：`.tmp/hwqty-mockup/mockup.html`；对比图：`.tmp/hwqty-mockup/hwqty-units.png`（左 = 现状全 K，右 = 方案 A）。
- 左右使用同一组生产数据、同一版 Chart.js（`node_modules/chart.js/dist/chart.umd.js` + `chartjs-plugin-datalabels`），无头 Chrome 渲染截图，非手工拼图。
- 观察结果：标签由 8 字符降为 5 字符；Y 轴由 0~35,000.0 变为 0~35M，轴标题 Qty (M)；ISG 仍为 K（49.3K、52.6K）；两侧 Chart.js `display:'auto'` 都会隐藏压盖标签（各藏 MBG 3 个点），A 方案不改变这一点。

## 7. 顺带发现（未实施）
- 只选一个 BU 时 `Total` 行与该 BU 行数值完全相同，图上出现两条重合线（`hwWideBlock` 末尾无条件加 Total，`BookingsService.java:758-770`）。
- Bookings 页最多 13 季度 × 4 线 = 52 个标签，`display:'auto'` 已隐藏重叠者；若要求标签全可读，可只标最后一个季度，其余走 tooltip。

## 8. 未闭合
用户需确认：走方案 A（纯前端半天），还是“Y 轴保持在 K 水平（0~30,000）只压缩标签”；以及 M/K 混排是否可接受（不可接受则只能选 C 拆图）。确认后实施路径为改 `miniTable.vue` + 两处 `valueDisplay`，先提 develop MR，验证通过再提 master。相关分支已存在：后端 `origin/admin-release` 上的本地 `feec561d3 fix(admin): HW Qty 改按千（K）展示`；前端 `f4bd80e7` 已在 `origin/develop` 等多个分支上。
