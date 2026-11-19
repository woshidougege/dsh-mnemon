---
id: "dfebe689-1537-49bd-a261-bb249822d57b"
title: "回放翻译：结果卡片随译文整体重建（setup 固化 props 的坑，MR !1128 / !1129）"
description: "ssg_sales_ops 回放翻译增量修复：切语言后图表标题/视图说明/Data tab 表头不刷新，根因是 SmartData.vue 与各图表壳子在 setup 顶层把 props 固化成普通常量；修复为 computed + PbiRecommendation 的 translationVersion 拼入 v-for key 强制整卡重建，含文件行号、type-check 报错与 MR 归属。"
status: "active"
created_at: "2026-11-19T12:12:53.327Z"
updated_at: "2026-11-19T12:12:53.327Z"
content_hash: "d88be6106190d4ff504adfbf8348ace67755e305770f6fc8565be0b0d839eecf"
source_paths:
  - "src/components/PbiRecommend/Nodes/SmartData.vue"
  - "src/components/PbiRecommend/PbiRecommendation.vue"
session_ids:
  - "mnemon-review-v1-95b6499f-1492-46ed-878b-792661acc9d0"
memory_body_ids:
  []
---

# 回放翻译：结果卡片必须随译文整体重建（MR !1128 / !1129）

前序文档：`3ac2a628-0a6a-42ed-a2a3-56755ff27ed0`（整轮替换 + 反问/建议/图表文案覆盖）、`62323dec-a57b-4d04-8fe1-21736166b548`（只翻当前视口那一轮）。本文是其后的增量修复：译文收集与回填链路本身没问题，问题在渲染层固化了旧节点，导致切语言时只有一部分文案跟着变。

## 现象与一次误判

用户反馈：卡片标题、气泡正文都随语言切换了，但**图表标题**、**视图说明句**（如 `Fiscal Qtr and Region form unique row pairs...`）、**Data tab 表头**仍是英文。第一反应是“部署的版本不对”（怀疑 !1128 没部署），确认部署无误后回到代码，问题在 `SmartData` 组件的响应式。

## 根因：setup 顶层把 props 固化成常量

`src/components/PbiRecommend/Nodes/SmartData.vue:44` 起：

`ts
// 注释原文：结果卡片一旦产出即固定，node 不会在生命周期内变化，故用普通常量而非 computed
const resultData = (props.node?.data ?? {}) as SmartDataRenderNode['data']
const warningNotices = pickSmartDataWarningNotices(resultData.notices) // :47
const viewGuard = resolveSmartDataViews(resultData)                     // :53
const validViews = viewGuard.views                                      // :54
`

换语言时父组件把 `node` 换成译文版：模板里 `props.node?.title` 这类**直接读取**的会更新，但 `resultData` / `viewGuard` / `validViews` 仍是旧对象。而图表标题（`:title="item.title"`）、视图说明（`view.reason`）、表头（`displayColumns`，依赖 `resultData.columns`）全部源自这几个常量——所以现象是“卡片标题翻了、图表内部没翻”。

子组件同样固化 props：`SmartDataTable.vue:36` 的 `const columns = props.columns`；各图表壳子的 `const enc/encoding/raw = props.view?...`（`SmartDataGeoMap.vue:54`、`SmartDataDistributionChart.vue:42` 与 `:51`、`SmartDataAdvancedChart.vue:55` 与 `:64`、`SmartDataChart.vue:69`、`SmartDataKpi.vue:36`、`SmartDataPanelChart.vue:43`）。

## 修复：两条缺一不可

1. **组件内改 computed**：`SmartData.vue` 的 `resultData`、`warningNotices`、`viewGuard`、`validViews` 改为 computed（模板自动解包，脚本内补 `.value`）。
2. **父组件按版本号重建**：`PbiRecommendation.vue:298` 加 `const translationVersion = ref(0)` 与 `watch(() => props.translations, () => { translationVersion.value += 1 })`，并把它拼进 `<SmartData>` 的 key（`:key="item.key + ':' + translationVersion"` 与 `:key="'smart:' + translationVersion"`）。

第 2 条的理由：子组件内部也把 props 固化成常量，只改自身 computed 无法穿透到 `columns` / `encoding`，所以让译文表一变就整卡重建（v-for key 变化即销毁重建），一次把子组件里的旧值全部作废。

验证：`npm run type-check`（`vue-tsc --build`）exit=0；中途曾因模板 key 表达式写错报 `PbiRecommendation.vue(427,7): error TS1005: ')' expected` 与 `TS1003: Identifier expected`，修正模板后通过。

## 交付与后续注意

- !1128（`feature/replay-translate-summary` → develop，commit `e8d84a27`）：只含“结果卡片下方总结文案送翻”（`summaryText`），不含本文的重建修复。
- !1129（`feature/replay-translate-card-refresh`，commit `9b6d8350`，从含 !1128 的 develop 重切）：带本文的重建修复。
- 新增渲染子组件时**不要在 setup 顶层把 props 固化成常量**，否则同类“切语言后局部不刷新”的问题会重新出现。
