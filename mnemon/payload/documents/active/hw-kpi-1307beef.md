---
id: "1307beef-abaa-4421-b44a-5a18bc9a2cde"
title: "HW KPI 数值标签终态（单位同字紧贴）与数字避线四方案调研"
description: "miniTable.vue 数值标签自绘 plugin 的终态参数（数值/单位同为 600 10px、unitGap 2、offset 13、取消单位淡化）、master !1070 / develop !1084 提交状态与 cherry-pick 冲突教训，以及\"数字避线\"调研：PBI 不做避线、白色描边的业界做法、描边与真避线的实现成本，和 ①现状/②描边/③避所有线/④只避其他线 四方案实测标签数与压线数对比与推荐。"
status: "active"
created_at: "2026-11-18T06:01:11.074Z"
updated_at: "2026-11-18T06:01:11.074Z"
content_hash: "10c7c3c9339fb54f96372d0f2c6e27536560317dbc77922b40bd710f12510a69"
source_paths:
  - "src/views/Bookings/components/miniTable.vue"
session_ids:
  - "mnemon-run-v1-8852168e-3518-45ed-b43d-8454978bc7ae"
memory_body_ids:
  []
---

# HW KPI 数值标签终态样式 + 数字避线四方案调研（含实测对比）

主题文件：`src/views/Bookings/components/miniTable.vue`（GPSSUS 页复用同一组件）。
相关文档：`063dbd84-ca76-4692-b6a9-0d2ca4c0ffca`（三态 + 贪心填充 handoff，unitGap 当时为 4）、`00abc05a-db17-4302-9eeb-6c47f99d9dc1`（inset 三态改造与 v5 修正）、`b881d9ec-57e3-4636-bf0a-5c10604822a9`（HW KPI 单位方案 v2）。本文只记录这两篇之后的新增内容：单位与数值"同一套字 + 紧贴"，以及"数字避开线条"的调研结论。

## 1. 标签样式终态（用户两条明确要求：单位紧跟数字、单位字号与数字一致）

自绘 plugin `buildCompactLabelsPlugin()`（id `hw-compact-labels`，`afterDatasetsDraw`）现行参数：

- `valueFont = '600 10px Consolas, Menlo, monospace'`
- `unitFont = '600 10px Consolas, Menlo, monospace'`（字重由 500 提到 600，与数值同字号同字重）
- `unitGap = 2`（由 4px 收窄；再大就"像被拆成两段"）
- `labelOffsetY = 13`（标签离数据点/曲线，原 9）
- 包围盒 `y1 = midY - 7 / y2 = midY + 7`
- **取消单位淡化**：原先 `globalAlpha = 0.75~0.8` 的做法已移除。理由：单位与数字同字号同字重时再降透明度，视觉上显得零碎，用户要的是 `25.4M` 这样一个整体。
- 副作用（正向）：标签宽度比 4px 间距时窄 2px，贪心排布可多塞 1~2 个标签。

## 2. 分支与 MR 状态（截至本次 checkpoint）

- master 线：分支 `dev/renxp4-hw-kpi-unit` → MR **!1070**，sha `999c6d8b`（含三态 + 贪心填充 + 单位紧贴），`can_be_merged`。
- develop 线：分支 `dev/renxp4-hw-kpi-inset-modes` → 新开 MR **!1084**，sha `5902513d`（仅这一笔单位样式）。**!1082 已合并**（`a7b2a222` 贪心填充），故拆分新 MR 接续。此前 !1081 已合并（三态 + Y 轴负刻度 + 标签密度）。
- 操作教训：从 `dev/renxp4-hw-kpi-inset-modes` cherry-pick 到 master 分支时，因同一函数连续被改导致 `UU src/views/Bookings/components/miniTable.vue` 冲突；实际处理是在干净的 `dev/renxp4-hw-kpi-unit` 上只 pick 单笔（`999c6d8b`，1 file changed, 4 insertions, 6 deletions）。需要 `git cherry-pick --abort` 的那次残留了未跟踪目录 `.tmp-fe-sp/`。
- `npm run type-check`（vue-tsc --build）每笔均通过。

## 3. "数字避开线条"调研（用户提出的优化点 3）

### 3.1 前提纠正：PBI 并不做这件事

Power BI 折线图 data label 的 Auto/Above/Below 只处理**标签与标签**的重叠，不感知曲线几何位置，也不检测标签与线的碰撞——Excel/PBI 中标签压在折线身上是常态。业界解决"文字压在线上仍可读"的通行做法是**白色描边光晕**（`strokeText` 白边 + fill），不是让文字躲开线。

### 3.2 两个可选方案

- **方案 A：白色描边（约 3 行，零风险）**：`ctx.strokeStyle = '#fff'; ctx.lineWidth = 2.5; ctx.strokeText(...)` 后再 fill。线条经过标签处被白边截断，文字保持清晰；不减少标签数、不增加碰撞判断。
- **方案 B：真正避让线条（约 40 行，技术可行）**：标签由自绘 plugin 定位，坐标完全可控，因此无障碍。做法：①把每条可见数据集相邻点连成线段集合（tension 0.4 的平滑曲线用直线采样近似，留 2px 余量吸收误差）；②每个候选标签框沿线段采样 10~12 个点判是否落入框内；③碰撞时换候选位，位置从"下方/上方 13px"扩到"下方/上方 13px + 下方/上方 22px"共 4 个。代价是标签数下降（Bookings 39 → 约 32，Qty 30 → 25），曲线密集处（ISG 低位线、FY2627Q3 崩盘区）标签无处可躲只能放弃。性能无压力：每次重绘约 6 万次坐标比较，<1ms。

### 3.3 四方案实测对比（卡片宽度按真实两列布局还原，同一批数据）

| 方案 | Qty 标签 | Qty 压线 | Bookings 标签 | Bookings 压线 |
|---|---|---|---|---|
| ① 现状（无描边） | 28 | 14 | 36 | 14 |
| ② + 白色描边 | 28 | 14 | 36 | 14 |
| ③ + 描边 + 避所有线 | 25 | 0 | 28 | 0 |
| ④ + 描边 + 只避其他线 | 28 | 11 | 32 | 6 |

- ② 只改善"压在线上是否可读"，布局不变，三行代码。
- ③ 完全不压线，Qty 少 3 个、Bookings 少 8 个标签；被去掉的多是原本无处安放的（ISG 贴底那条线附近）。
- ④ 标签基本不掉，但自己的折线仍会穿过自己的标签（多数"压线"是自家线从数据点穿出）。

### 3.4 结论与推荐

推荐 **③（描边 + 避所有线）**，理由：25 个标签已是旧硬过滤版（约 10 个）的 2.5 倍，数量够；"零压线"是业务看图第一眼的可读性信号；被舍弃的 3 个原本也是噪声。若更看重量，则 ④ 折中；② 建议无论如何叠加，与 ③/④ 不冲突。

## 4. 待决 / 未落地

- 方案 ③ 与 ④ 由用户二选一（checkpoint 结束时**代码尚未改动**，仅出过对比图）。
- 定案后改 `buildCompactLabelsPlugin` 单函数即可，并补进 !1070（master）与 !1084（develop）两笔 MR。
- 合并后 TST 复核清单：单位是否 `15.0B` 且与数字同字号紧贴、标签是否压 X 轴、三态点击循环、Y 轴是否出现负刻度、实际数据下的标签密度与压线情况。
