---
id: "f5b15acb-8bb5-49fa-9f28-4958f4928c5f"
title: "HW KPI 图卡后续反馈：inset 小图读不全与 Bookings 轴回到 $M（根因、方案、待定）"
description: "TST 目视验收后新暴露的两条反馈：inset 小图 134×68 框/11 个点导致标签被切与被 auto 隐藏（成因与 A 弹窗/B 加大/C zoom 插件三方案对比）；Bookings 轴 $B 与全页 M 口径打架（三选项，推荐后端 B 门槛 1e9→1e10 + 标签抽稀）。代码未动，待用户确认。"
status: "active"
created_at: "2026-11-17T09:49:50.894Z"
updated_at: "2026-11-17T09:49:50.894Z"
content_hash: "f3bbbd8db9833d215c1d0a5f2e96950c6132ce28065e57348762444049459175"
source_paths:
  - "src/views/Bookings/components/miniTable.vue"
  - "src/components/SparkChart/index.vue"
session_ids:
  - "mnemon-run-v1-0a55bf93-7630-4dcf-a6fd-9f9784275034"
memory_body_ids:
  []
---

# HW KPI 图卡后续反馈：inset 小图读不全 与 Bookings 轴回到 $M（根因、方案、待定）

> 上游文档：`98248233-94f5-434d-8021-39a475145154`（动态单位实施落地与三 MR 交付）、`b881d9ec-57e3-4636-bf0a-5c10604822a9`（单位契约与阈值）、`9b904808-5759-4d8a-af76-2e6d691ae8f4`（Bookings M→B 起源）。本文只记录**上线后在 TST 做目视验收时新暴露的两条用户反馈**及其量化根因，不改写上述结论。

## 1. 验收确认（TST 实测，2026/11/17）

生产/UAT/TST 三域名：`https://ssgspark.lenovo.com` / `ssgspark-uat.lenovo.com` / `ssgspark-tst.lenovo.com`（TST 页 `/spark/gpssus`）。BU 维度实测：

- HW Bookings 轴为 `$B`（刻度 16.0/14.0/…/0.0），标签 `15.0B / 12.9B / 12.1B / 9.60B`，量级小的 ISG 自动降档为 `557.9M / 593.1M / 969.5M` —— 块级选轴 + 值级降档按设计生效。
- HW Qty 轴保持 `M`（块内最大 29.3M，选 M 正确），主图 ISG 标签被 `hideInMain` 置空，右下角出现 `ISG · K` 内嵌小图（43.5K、52.6K…44.9K）。
- **Geo 维度（6 条线 AP/EUROPE/META/NA/LA/ISO）不带 inset**，Qty 落 M、Bookings 落 $B —— 与"inset 只在 `hw_qty` 且维度为 BU 时下发"一致。用户曾以为"没生效"，实为维度不同（Geo 下六条线同量级 25M~29M，无需拆小图）。
- 最右季 FY2627Q3 断崖（Bookings 15.0B→2.20B、Qty 24.8M→9.95M）是当季未跑完，非缺陷。

## 2. 小图"显示不全"的量化根因（用户问：能不能让用户自己拉图）

`src/views/Bookings/components/miniTable.vue` 的 inset 是个 **134×68px 悬浮框**（`:389 .bk-opkpi-inset`，`bottom:26px; width:134px; height:68px`），内部 canvas 只有 **126×50**（`:411 .bk-opkpi-inset-chart{height:50px}`），要画 11 个季度 → **每点约 11px**，而一个 `52.6K` 标签本身约 25px。三个真实成因：

1. **右侧被切**：inset 的 `layout.padding.right` 仅 8px（`:176-178`），最后一个点的标签顶到边界外。
2. **中间断档不是丢数据**：`datalabels.display:'auto'`（`:197`）按重叠自动隐藏标签，所以看起来"隔几个点才有一个数"。
3. **框体 `pointer-events: none`**（`:401`），点击无反应。

方案对比（均未实施，等用户拍板）：

| 方案 | 做法 | 评价 | 成本 |
|---|---|---|---|
| **A 点击放大（推荐）** | `cursor:pointer` + 放大角标，点击弹 `el-dialog` 画 900×360 大图（含 11 季全量标签、tooltip、Show Data） | 点间距变 80px，标签完全放得开；不动主图布局 | 约 60 行，无新依赖 |
| B 直接把框加大 | 240×100，主图右侧让位 | 治标：11 个点仍挤，且遮挡主图更多 | 小 |
| C 引入 `chartjs-plugin-zoom` | 滚轮缩放 + 拖拽平移 | 真"自由拉图"，但新增依赖、与页面滚动冲突、移动端难用 | 中 |

**判断依据**：用户诉求是"能看清楚 + 自己控制"，A 用 1/10 的成本覆盖 90% 场景；未来确实要平移缩放再叠 C 不白做。

## 3. Bookings 轴显示 `$B` 与全页口径不一致（用户质疑）

理由不是审美，而是三处硬伤：

1. **与本页其它 KPI 打架**：同屏顶部 `Mx $1,237.8M`、`MT $1,396.4M` 都是 M，只有这张卡跳 B，读者要心算换算。
2. **B 太粗**：`0.1B = 1 亿美元`，刻度只剩 10.0/12.0/14.0 三格，看不出季度间几十 M 的波动。
3. **当初换 B 只是为了让标签变短**（`15,063.2` → `15.1B`）；实测该卡 11 季、图区约 950px，**每点 86px，8 字符只占 48px，M 完全放得下** —— 真正拥挤来自同一点附近 4~6 条线的标签互相压，与单位无关。

| 方案 | 轴 | 标签示例 | 评价 |
|---|---|---|---|
| **① 抬高 B 的门槛（推荐）** | `$M` | `15,063.2 / 9,162.8 / 2,251.9` | 金额需 ≥ **100 亿**才用 B（后端常量 `1e9` → `1e10`，Qty 的 K/M 档不受影响）；当前 91~150 亿落 M 档，将来上 100 亿自动升 B |
| ② 保持现状 | `$B` | `15.1B / 9.16B` | 标签最短但口径不一致、精度粗 |
| ③ 轴 M + 标签抽稀 | `$M` | 隔一季显示一个 | 更彻底解决多线拥挤，少看到的数值可从 tooltip 补 |

**倾向 ①+③ 组合**：后端把 B 门槛抬到 `1e10`，前端在"季度点 ≥9 且线数 ≥4"时标签隔点显示。

## 4. 附带发现的既有问题（非本次改动引入）

Geo 视图下 Bookings 卡左下的 `0.16B`、`3.39B` 与最左 x 轴刻度挤在一起 —— 6 条线标签过密时 Chart.js datalabels 自动避让的老毛病；若要处理，可对 Geo 这类多线场景隔季显示。

## 5. 状态

**TASK AS OF THIS BLOCK：两处改动都未动代码**，等用户确认（①小图是否做 A 弹窗、是否顺带支持拖拽缩放；②Bookings 是否接受 ①（含是否同时做 ③））。两处都在同一个 `miniTable.vue` + 后端一个常量，可一次改完合提 MR；按项目惯例合并动作留给用户。

## 6. 路径注意

工作目录是 `C:\Users\63588\lenovo\ssg-sales-ops`（**连字符**），但 grep/glob 结果里同一批源码出现在 `C:\Users\63588\lenovo\ssg_sales_ops`（**下划线**）。找文件时以相对路径 `src/views/Bookings/components/miniTable.vue`、`src/components/SparkChart/index.vue` 为准，不要在两种绝对路径间互相推断。
