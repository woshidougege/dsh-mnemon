---
id: "a4f0e3cb-9da0-401b-bb0c-0cdcc838effb"
title: "HW KPI 单位呈现四方案对比与 inset 卡片内展开（回退 1e9、MR !573/!1070/!1071）"
description: "推翻上一版 B 门槛 1e11 与 inset 弹窗：金额门槛恢复 1e9、标签一律跟轴同单位（0.56B 而非 557.9M）、数量仍 K/M 自适应；inset 改为收起 168×88 不画标签→点击卡片内 320×196 显示标签并可切 4/8/全部，同时修 overflow 裁切。含 A/B/C/D 四方案对比表（推荐 B）与三个 MR（!573 admin-release、!1070 master、!1071 develop）及验证、踩坑、待用户选方案的遗留。"
status: "active"
created_at: "2026-11-17T11:48:18.926Z"
updated_at: "2026-11-17T11:48:18.926Z"
content_hash: "c9831cb1e1c6e28649c6637cd74b107dd073a937ccfdece10ef4eaf277022306"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/bookings/service/BookingsService.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/bookings/service/BookingsServiceHwKpiTest.java"
  - "src/views/Bookings/components/miniTable.vue"
  - ".tmp/hwqty-mockup/inset-inline-v3.html"
session_ids:
  - "mnemon-run-v1-77dfe07b-1b63-4d39-a33a-f0ff3befe75d"
memory_body_ids:
  []
---

# HW KPI 单位呈现方案对比与 inset 卡片内展开（回退 1e9、三 MR !573/!1070/!1071）

> 本文件**取代** `937ea76a-91fb-4584-95e9-ec7972b3f2da`（金额 B 门槛 1e11 + inset 920px 弹窗、MR !1057/!1069/!572）中关于门槛取值与 inset 交互的结论。接口契约与口径仍以 `b881d9ec-57e3-4636-bf0a-5c10604822a9` 为准；inset 密度根因见 `f5b15acb-8bb5-49fa-9f28-4958f4928c5f`（134×68 框 / 11 个点 / 每点约 11px）。

## 1. 用户反馈触发的两处回退

- 用户 (checkpoint 事件 4471)："感觉还是图形里带单位好看呢"；随后 (事件 4484)："你给我几个动态单位的案例 我看看哪个合适"。
- **金额门槛回退**：上一轮把 B 档门槛从 1e10 抬到 1e11，理由是"$B 下 0.1B 精度太粗"，但 Bookings 常态 91~150 亿因此掉回 `$M`，标签反而变成 `15,015.9` / `9,162.8` / `2,251.9`（8 字符），比 `15.0B / 9.16B / 2.25B` 更难看。结论：**恢复 `≥1e9 → B`**（回到改动前的判断标准），放弃 1e11。
- **取消值级降档**：以前 `$B` 轴上遇到小值会降成 `557.9M`，造成"轴写 B、标签写 M"的单位不一致，更乱。现在**标签一律跟轴同单位**（`0.56B`）。
- **数量仍 K/M 自适应**：PCSD 1300 万台 `13.0M`、ISG 5 万 `52.6K`。不把 PCSD/MBG 压成 M 是刻意的——它们量级不小，数字长是数据本身造成的。

## 2. inset 小图：弹窗 → 卡片内原地展开

- 弹窗被否：一点就遮住整页、还要关闭，与"只想看清楚一点"的意图不匹配。
- **收起态** 168×88 角标，**不画任何数值标签**——"糊成一团"的根因就是 11 个季度的点挤在 134px 里，8px 字体必然重叠，`datalabels: 'auto'` 救不了（该版本 datalabels 只有 `display`，无 `auto`）。
- **点击后**原地放大到 320×196，此时才显示季度与数值标签，并带 `Last 4 / Last 8 / All` 切换；再点或点 ✕ 收起。
- 顺带修裁切：inset 原先放在 `overflow-x:auto` 的滚动容器里，右侧与底部被裁掉，现移到卡片层（`.card` 内 `position:absolute`，`right:14px; bottom:44px`；展开态 `z-index:6`）。

## 3. 四种单位呈现方案（同一份真实 Bookings 数据 FY2425Q1–FY2627Q3）

| 方案 | 单位位置 | 标签样例 | 好处 | 代价 |
|---|---|---|---|---|
| **A**（当时线上） | Y 轴外侧竖排 `$B` | `15.0` `6.40` `0.56` | 标签最短、图最干净 | 标签本身无单位，须看轴才知单位 |
| **B** | 每个数值后缀 | `15.0B` `6.40B` `0.56B` | 一眼可读；顺带消除轴 B / 标签 M 的不一致 | 多 1 字符，11 季度时更易重叠（实线末端最明显） |
| **C** | Y 轴刻度带上单位 | `16B` `12B` `8B` + 纯数字标签 | 标签短且单位在图内 | 刻度只有整数，ISG 的 `0.56B` 落在 `0B``1B` 之间看不出量级 |
| **D** | 图内右上角胶囊 `$B` | `15.0` `6.40` | 单位醒目、不占标签宽度 | 右上角正是 Total 峰值区（`15.0`），胶囊会压线；须挪右下或留白 |

**建议 B**：直接对应"图形里带单位"的诉求，且不改变轴。若选 B 需同时处理：① 标签字号 8.5px → 8px 并保留 auto 隐藏重叠；② tooltip / 图例 / 小图统一用 `15.0B` 一种写法，避免"标签 B、tooltip M"。

## 4. MR 清单（三个均 opened、`can_be_merged`、无冲突）

| 仓库 | MR | source → target | 内容 |
|---|---|---|---|
| ssg-sales-ops（后端 admin） | [!573](https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/573) | `fix/hw-kpi-unit-billion-admin` → `admin-release` | 提交 `5314c409e` "fix(admin): HW KPI 金额回到共用 10 亿 B 档"，2 文件 +11/−26，撤销上轮 1e11 |
| ssg_sales_ops（前端） | [!1070](https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1070) | → `master` | 标签不降档 + inset 卡片内展开 + 裁切修复 |
| ssg_sales_ops（前端） | [!1071](https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1071) | → `develop` | 同 !1070，双提交 patch-id 与 master 一致 |

## 5. 验证与口径

- 后端 `BookingsServiceHwKpiTest` 5/5 通过；Maven `BUILD SUCCESS`（22.7s）。前端 `vue-tsc` 两分支通过。
- 数据口径：生产库 `mv_bookings_agg`；PCSD/MBG 取 `kpi='HW Bookings'`，ISG 取 `kpi='PR Denominator'`（不排 NEC，与接口口径一致）。`$B` 粒度：≥10B 保留 1 位、<10B 保留 2 位（0.01B = 10M）。
- 预览产物（示意图，均 2026-11-17）：`.tmp/hwqty-mockup/inset-inline-v3.html` 与 `inset-inline-v3.png`(3374×1243)、`unit-variants-2.png`、`unit-variants-4.png`(3075×1363)、`probe-old.png`；依赖本地 `lib/chart.umd.js`(208518B)、`lib/datalabels.js`(12937B)。

## 6. 中途踩坑（环境/工具，非本次逻辑缺陷）

- 切分支后测试编译红：`ChatbiEventExportServiceEstimateTest.java:[75,27]`/`[107,27]` 调 `countSessionsForExport`、`ChatbiEventExportServiceStreamTest.java:[127,27]`/`[162,27]`/`[214,27]`/`[236,27]`/`[322,27]` 调 `querySessionsForExport`、`[351,22]` 调 `export` 参数不匹配，Maven 报"实际参数列表和形式参数列表长度不同"。属分支同步遗留的调用点，与 HW KPI 改动无关。
- headless Chrome 截图（`chrome.exe --screenshot`）若目标 png 已存在会继续沿用旧文件；判断是否真的生成要看文件 mtime + size（曾出现 `354214` → `305227` → `350797` 的字节数变化）。Chrome 路径：`C:\Program Files\Google\Chrome\Application\chrome.exe`。

## 7. 状态（TASK AS OF THIS BLOCK）

- **等用户从 A / B / C / D 中选一个**：选 B/C/D → 改前端并更新 !1070、!1071；选 A → 前端不动，直接合并三个 MR。
- 合并后需在 TST 目视确认两处：Bookings 标签是否为 `15.0B` 短格式；Qty 角标小图是否已无重叠数字。
- 页面回归仍未做；GPSSUS 页的 HW 卡未单独验证。合并/部署由用户执行，本文件不代表已上线。
