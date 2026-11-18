---
id: "063dbd84-ca76-4692-b6a9-0d2ca4c0ffca"
title: "Bookings 内嵌 ISG 小图三态化与数值标签贪心填充（实现 handoff）"
description: "miniTable.vue 内嵌 ISG 小图的实现 handoff：三态（收起/小图/大图）循环与标题行入口、bottom 26→48px 避开 40° 斜排 X 轴标签、数值标签自绘 plugin（单位与数值同为 10px、unitGap 4、offset 13）、贪心填充取代硬性过滤的覆盖率与取舍、master/develop 双线 MR 状态与 TST 复核清单。"
status: "active"
created_at: "2026-11-18T04:17:07.367Z"
updated_at: "2026-11-18T04:17:07.367Z"
content_hash: "829cd2d431d1fa4eae8ad130bc94a838d8395e75507a8a45f42a0d1e1f0b9bf2"
source_paths:
  - "src/views/Bookings/components/miniTable.vue"
session_ids:
  - "mnemon-run-v1-443f0ee5-9576-4108-8a9d-945d24f3e203"
memory_body_ids:
  []
---

# Bookings 内嵌 ISG 小图三态化 + 数值标签贪心填充（实现 handoff）

主题文件：`src/views/Bookings/components/miniTable.vue`（GPSSUS 页复用同一组件，改动会一起生效）。
上位文档：`b881d9ec-57e3-4636-bf0a-5c10604822a9`（HW KPI 单位方案 v2 / inset 契约）、`f5b15acb-8bb5-49fa-9f28-4958f4928c5f`（inset 读不全与 Bookings 轴口径反馈）。本文只记录这两篇之后新增的三态、定位、标签排版与 MR 落地状态。

## 1. 小图三态（用户要求：小图/大图之外再加"收起成图标"）

- 状态机：`INSET_MODES` 数组 + `const insetMode = ref('preview')`；`cycleInset()` 取 `(index + 1) % INSET_MODES.length` 逐级循环 preview → expanded → collapsed。
- 收起态：卡片内不渲染任何图表，只在标题行留一个 `〰 ISG` 胶囊按钮（`class="bk-inset-toggle"`，11px 折线图标，高 17px），渲染条件 `v-if="insetChart && insetMode === 'collapsed'"`。
- 小图态（默认）：168×88 角标，只画趋势不写数值。
- 大图态：320×196，带 K 轴、标签与 `Last 4 / 8 / All` 区间切换（`.bk-opkpi-inset-ranges`，`@click.stop` 防止触发循环），右上角 `✕` 收起、`⤢` 展开；点击与 `keydown.enter` 都可循环。
- 设计理由：收起态入口放在**标题行**而不是图内——图内任何位置都会盖住曲线或坐标轴，标题行是唯一零遮挡的位置。

## 2. 小图压在 X 轴标签上：根因与改法

- 根因：小图原 `bottom: 26px`，而 X 轴季度名 40° 斜排约占 46px 高，小图底边正好切在标签带上，`FY2627Q1/Q2/Q3` 上半截被小图白底盖掉。
- 改法：`bottom: 48px`（现网代码即此值）。底边收在斜标签上方约 2px，小图坐在绘图区右下角。
- 残留代价：仍会压住 Total 曲线尾部约 2 个季度，属"角标小图"固有代价，可接受。
- 48 是按"9 字符 × 40° 斜排"估的；Chart.js 轴高随标签长度动态计算，TST 若仍差几像素需微调该值。

## 3. 数值标签自绘：单位与数值同字号

- 起因：`ChartDataLabels` 不支持一段文字两套字号，主图改为自绘 plugin `id: 'hw-compact-labels'`（`afterDatasetsDraw`）。
- 用户明确要求：单位字号必须与数值一致（8px 单位在 11 季的图上几乎看不清），且标签离数据点更远。
- 现行参数：
  - `valueFont = '600 10px Consolas, Menlo, monospace'`
  - `unitFont = '500 10px Consolas, Menlo, monospace'`（原 8px）
  - `unitGap = 4`（原 1px，数值与单位不再粘连）
  - `labelOffsetY = 13`（原 9，标签离数据点/曲线更远）
  - 包围盒 `y1 = midY - 7 / y2 = midY + 7`（原 ±6，随标签变宽同步调整）
- 数值与单位的区分方式改为**只靠字重（600 vs 500）+ `globalAlpha = 0.8`**，不再缩字号。
- 保留规则：`inset.hideInMain && dataset.label === inset.series` 时主图不标该序列数值（否则一串小数字贴着 X 轴）。

## 4. 标签密度：贪心填充取代硬性过滤

- 走过的弯路（已被用户否决）：先做成硬性过滤——只保留首季、最近 3 季与峰值，其余直接丢弃；结果 44 个点只剩 4~5 个标签，用户反馈"都没几个数字了 不太好吧"。
- 现方案：
  1. 全部候选参与排位，`priorityOf(dataIndex, value === seriesPeak)` 升序：最近 3 季最优 → 首季/峰值 → 其余按从新到旧。
  2. 每个点先试**下方 13px**，有冲突再试**上方 13px**，上下都撞才放弃（双向排布提高覆盖率）。
  3. 冲突判定用矩形包围盒相交，保证不叠字；越界 `chartArea` 直接跳过。
- 实测覆盖率（示意图数据）：Bookings 44 候选 → 39 画出；Qty 33 → 30。对照"全部强制标注"：Bookings 3 处重叠、Qty 8 处重叠。
- 为什么不追求 100%：重叠集中在最右最新季度——多条线趋近 0，4 个标签叠成一团 `$0.7B $0.1B`。同一季度相邻线常只差 2~3M（屏上十余像素），30px 宽标签上下排 13px 仍会撞。取舍是牺牲中段某条线，保住最新 3 季与峰值。
- 维度差异：Bookings 标签短（`$9.2B` 5 字符）所以与全标差别小；Qty 标签更宽（`25.4 M`）。切到 Geo（6 条线）或更长的百万级数字，贪心与全标差距会显著放大。

## 5. 分支与 MR 落地状态（截至本次 checkpoint）

- master 线：分支 `dev/renxp4-hw-kpi-unit` → MR **!1070**，包含三态与标签调整，sha `d9ae63db`，`can_be_merged`。
- develop 线：分支 `dev/renxp4-hw-kpi-inset-modes` → MR **!1081**（`7b88aabe` Y 轴负刻度/单位可读性/标签密度 + `0e877d5f` 三态与避开 X 轴）**已合并**；MR **!1082**（`3427d905` 单位字号+间距 → `a7b2a222` 贪心填充）open。
- 同步策略：同一修复要同时进 master 与 develop，按 commit 在两条分支分别提交/ cherry-pick，保持两边内容一致；master 线缺失 develop 的 v5 改动时，develop 分支需带上两笔。
- 验证：`npm run type-check`（`vue-tsc --build`）通过。
- 待办：合并后在 TST 一次性复核——单位字号是否与数值一致、标签是否还压 X 轴、三态点击循环是否正常、Y 轴是否还有负刻度。

## 6. 可复用教训

- 标签密度不要用"白名单过滤"解决，用贪心占位 + 优先级排序；过滤会让图看上去像没数据。
- 视觉可读性优先于"淡化处理"：单位靠缩字号淡化在本场景失败，靠字重与透明度才成立。
- 小图/角标类覆盖层要按 X 轴斜排标签的实际高度留底边距，不能按绘图区边距估。
- 一次 UI 改动往往要同步两条发布线（master/develop），落地时先确认哪条线已有前置改动，避免 develop 只拿到半套。
