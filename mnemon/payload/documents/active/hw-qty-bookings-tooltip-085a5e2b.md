---
id: "085a5e2b-9be3-4cdf-aa9e-59c676943cc2"
title: "HW Qty/Bookings 最底线条零标签：候选位全死根因、只往上退让修复与 tooltip 单序列单位"
description: "miniTable.vue 标签排布为何让最底一条线零标签（下方出界/上条线下置标签重叠/上方压线），修复三条（关键点优先、先试上方、只往上退让+白描边）与标签数变化；以及动态块单位为何不能按线显示 K、ISG tooltip 改为按自身单位 52.6K；含 develop !1086 / master !1070 提交号与 TST 待验证边界。"
status: "active"
created_at: "2026-11-18T06:30:23.722Z"
updated_at: "2026-11-18T06:30:23.722Z"
content_hash: "4e5300fbcbf4ac52ca5f916c5cd854947ac1e91f0edf4bb27cfd78cf57e421fa"
source_paths:
  - "src/views/Bookings/components/miniTable.vue"
session_ids:
  - "mnemon-run-v1-9e2e1fa9-461c-4679-b6ca-51a0f78f0c03"
memory_body_ids:
  []
---

# HW Qty / Bookings 最底线条零标签：根因、退让策略与 tooltip 单序列单位

主题文件：`src/views/Bookings/components/miniTable.vue`（Bookings 与 GPSSUS 共用）。
相邻文档：`1307beef-abaa-4421-b44a-5a18bc9a2cde`（标签样式终态 + 数字避线四方案调研）、`0200947d-78c0-4b83-905e-cacea7c7ad11`（图上带单位落地 + MR 状态）、`e5fe9ed8-0b73-44be-b883-f39a4972b6a9`（单轴按量级压缩方案 A）。本文只记录这两条用户问题的排查结论与修复。

## 1. 用户提问（2026-11-18）
1. 左图（HW Bookings）最下面那条线怎么一个数值都没显示；
2. 右图（HW Qty）最下面那条线是不是该显示 K —— 质疑「不是动态单位吗」。

## 2. 现象与根因：不是没数据，是三个候选位全被判死
按 TST 真实卡片比例本地复现：Bookings 的 ISG 线 **0/11 个点有标签**（仅首季 `$0.60B`），11 个季度全被压掉：

| 候选位置 | 失败原因 |
|---|---|
| 点下方 13px | ISG 离绘图区底边只有 3~5px，**包围盒出界** |
| 点上方 13px | 上一条线（MBG）的标签默认放**它自己点下方** 13px，正好压住 ISG 头顶 → 判重拦截 |
| 点上方 22px | 压到 MBG 折线 → 被 `hitsLine` 避线规则拒掉 |

即「上一条线占下方 + 本条线无下方空间 + 上方压线」三者叠加，最底一条线必然零标签；越靠下的线越受影响。

## 3. 修复（仅 `miniTable.vue`，+23/−9）
1. **排位优先级**：首季 / 峰值 / 最近一季提前占位 → 每条线先保住 3 个关键值，不被上面几条线吃光候选位。
2. **候选位先试上方**：标签贴自己点的上方，下方留给下面那条线（原来是上一条线占下方，底下那条永远无位）。
3. **退让只往上**：四个位置全占死时，只在上方允许压线（`fits("strict") || fits("any", spots.slice(0,2))` 的降级链只保留上面两档），**禁止往下让**。往下让会出现 `$2.21B` 落到 ISG 线旁边被读成 ISG 数值的错误读数，比不显示更糟；压线由 2.5px 白色描边保底可读。

效果（本地同比例渲染）：Bookings 标签 13 → 18 个、ISG 0 → 3；Qty 18 → 21。

## 4. 「动态单位」为什么不能按线给 K
动态单位是**按整张图（块单位）**算的，不是按每条线：整图最大值 29.3M → 全图固定走 M 档，所以 44K~63K 只能写成 `0.04M~0.06M`。
若单独让 ISG 标 `52.6K`，同一根线性轴上会同时出现 `52.6` 与 `25.4`（M 档）两个数，视觉上 ISG 变成最粗的一条线，而其实际量级只有 PCSD 的约 1/250 —— 属图形错误，比缺数字更严重。真要按线分单位只有双 Y 轴或拆图（前文档已论证不划算）。
该线保留两个出口：
- 右下角 **ISG·K inset 小图**（`insetSeries` / `buildInsetDataset`，44.9K / 52.6K）；
- **hover tooltip 改口径**：`formatChartTooltipValue` 原来对 ISG 输出 `ISG: 0.1M`（52,575 与 44,486 都归到 0.1M，信息量为零），现改为按该序列自身单位显示 `ISG: 52.6K`，与 inset 小图口径一致。

## 5. MR 与验证状态
| 目标 | MR | 分支 / 提交 |
|---|---|---|
| develop | **!1086**（新建） | `dev/renxp4-hw-kpi-inset-modes` @ `3aa0fca9`（已 rebase 到 develop 后 force push） |
| master | **!1070**（追加提交） | `dev/renxp4-hw-kpi-unit` @ `053d796c` |

- `npm run type-check`（`vue-tsc --build`）通过。
- 注意：本次曾误把提交推到一次性分支 `dev/renxp4-hw-kpi-inset-modes` 再 cherry-pick 到 unit 分支，产生重复提交后靠 rebase 收敛；同一 patch 落两个目标分支时优先直接在同一工作区切分支提交，避免 cherry-pick 叠加。
- **页面实测未做**：结论基于 `.tmp/hwqty-mockup/` 按卡片比例还原的本地渲染（如 `labels-fix-v3.png`、`bookings-label-fix6.png`、`tst-bookings-bottom-zoom.png`），真实图区略高、标签数应更多，需合并部署 TST 后复核。
