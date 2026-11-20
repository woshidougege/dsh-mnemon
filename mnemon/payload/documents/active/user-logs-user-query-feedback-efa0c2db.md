---
id: "efa0c2db-c3ce-4263-b7d1-6c2d706c7bc4"
title: "User Logs 报表 user 维度统计下线：Query/Feedback 改动与未决范围"
description: "领导要求「按 user 统计的维度都删掉」后的实现记录：Query 页删 Users involved 卡并把指标区定为嵌卡 B 方案、Feedback 页删 Users 卡、Routing failed 列整体移除；后端字段故意保留的理由；以及 Overview/User view/Behavior/Org/Page Analytics 五个仍按人统计的页签待确认范围。"
status: "active"
created_at: "2026-11-20T08:50:31.499Z"
updated_at: "2026-11-20T08:50:31.499Z"
content_hash: "501cad768afd8f03f32fb670407f0936fe49f0b2ca8d9c0744921cb5d4f1baa4"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-e1425d0d-9e19-41e4-9484-4696dc24b389"
memory_body_ids:
  []
---

# User Logs 报表「user 维度统计下线」：Query / Feedback 两页改动与未决范围

> 状态（2026-11-20）：**Query、Feedback 两页已完成并前端验证通过；剩余 5 个页签与后端字段待用户确认范围。**
> 相关文档：Success Rate → Accuracy Rate 设计（38b9c4c0-0988-4206-9198-11f20587ae61）——本次的「指标区 B 方案」是该设计的延续。

## 1. 需求来源与口径

用户转述领导要求（原话）：「还有这个按照user统计的维度要删掉」。追问范围后用户明确：「关于user维度的统计我们都删掉」。

已落地的约定：**Query / Feedback 两个页签上所有「按人」的统计一律下线，指标一律按「问题 / 条数」计数**。该口径已写进代码注释，例如 FeedbackView.vue：「FeedbackCoverage / 点踩率：只按 question 维度统计；user 维度按业务方要求下线（2026-11-20）」。

## 2. 已完成改动

### 2.1 Query 页（components/views/QueryView.vue）

- 指标卡 `Users involved / distinct users`（取自 summary.users）整张删除。
- 指标区栅格 `.metric-nested` 从 `minmax(0,1.5fr) repeat(2, minmax(0,1fr))` 三列改为两列（1.5fr + 1fr）。
- 指标区定稿为 **B 方案（嵌卡）**：左边 `Succeeded` 卡内嵌 `Accuracy Rate` 子卡（左侧紫色竖条 #6f63c9 + `.mn-sub`），右边 `Question count` 卡。准确率视觉上从属于成功率，不作为并列的第四个指标。
- 同时删除了 A 方案残留：TEMP 版式开关、整棵纯树模板（约 45 行）、`layoutMode` 变量、`.layout-switch` / `.metric-tree` / `.mt-row` 全套样式；折叠的「Show how the accuracy number is built」保留为普通按钮。另清掉因删版式失效的 5 个变量（`accuracyTitle`、`coverageText`、`coverageRate`、`failedReviewedCount` 及 accuracyTitle 顺带用到的 3 个反问计数）。
- `Routing failed` 整列移除：表头单元格、行内数字格、hover 说明、顶部汇总条的 `Routing failed 3 0.8%`、前端 `sourceDistribution` 里的 `routeFailed` 汇总与百分比；表格栅格由 6 列改 5 列。现表头为 `Route Report / Status`｜`Question count`｜`Success`｜`Failed`｜`Success rate`。

### 2.2 Feedback 页（components/views/FeedbackView.vue）

- 顶部指标卡 `Users`（s.users）删除；现为四张卡：`Total feedback` / `Helpful` / `Not helpful` / `With comments`。
- 上一轮已删的 `User feedback rate`、`User dislike rate` 两个比率格不再回来，比率区只剩 question 维度。

### 2.3 验证方式（本次沿用的做法）

`@vue/compiler-sfc` 静态编译 script + template（输出即 script ok / bindings N / template ok），再请求 vite 实时编译该组件确认 HTTP 200（本次 QF=200、FB=200）。改动后两文件内已无 `users` 引用。

## 3. 关键决策：后端字段故意保留

`feedbackUserRatio`、`dislikeUserRatio`、`askUsers`、`dislikeUsers`，以及 Query summary 的 `users` **接口仍在返回，前端不再读取**；Query 的 `routeFailed` 也同理（SQL 未动）。

理由：删后端要动 SQL 和 DTO，且导出、看板等处可能仍在消费这些列，贸然删除会断。取舍是**先只摘前端展示层**，等确认无消费方再清后端。

## 4. 未决事项（下一步需要用户/领导明确）

1. **后端字段是否清**：上面那批 user 比率/计数与 `routeFailed` 是否连同 SQL、DTO 一起删除。
2. **范围是否扩到整个 Usage Analytics**：仍重度按人统计的页签有
   - `Overview`：Active users、New/Returning/Frequent users、D1/D7/D30 留存、按人次的日活曲线
   - `User view`：整页都是按人统计（人均操作数、留存、用户明细表）
   - `Behavior`：Feature users、Baseline users、留存、Frequent users
   - `Org`：Executive / Active users、按人的使用明细
   - `Page Analytics`：Unique Users 列、Sessions
   若「都删掉」指整站去人化，则 `User view` 整页需下架、`Overview` 留存区块需拆掉，属于另一个量级的改动。
3. **术语一致性**（已提出未决）：表格 `Source report` 列在某问路由失败时显示 `TCV · routing failed`，与刚删除的汇总列不是同一件事，是否改成更易读的 `TCV · lookup failed` / `TCV · 未取到报表`。
