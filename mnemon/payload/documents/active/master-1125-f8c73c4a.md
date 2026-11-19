---
id: "f8c73c4a-a6ef-491e-a80c-96d0df436d17"
title: "回放翻译渲染内核归属、正文替换三方案与 master 撤下翻译（!1125）"
description: "ssg_sales_ops 回放与 Chat BI 共用 PbiRecommendation.vue / pbiParse.ts / pbiFold.ts 的查证结论（外壳独立、内核同一份，附两侧 props 差异）；正文替换的 A/B/C 三方案与代价；!1125 从 master 摘掉翻译的范围、刻意保留的骨架优化，以及 master/develop 在 HistoryChatPanel.vue 分叉后的两条复原路径。"
status: "active"
created_at: "2026-11-19T10:52:42.398Z"
updated_at: "2026-11-19T10:52:42.398Z"
content_hash: "21b4b779d7cc32e601eb58fd8fa6b9f667507e3fb4538bbc3e73c9a2125cb3f3"
source_paths:
  - "src/components/PbiRecommend/PbiRecommendation.vue"
  - "src/components/Agent/AgentChatPanel.vue"
  - "src/views/Setting/reports/history-replay/HistoryChatPanel.vue"
  - "src/views/Setting/reports/history-replay/translateApi.ts"
  - "src/views/Setting/reports/user-logs/components/modals/DetailModal.vue"
  - "src/views/Setting/reports/user-logs/components/ui/LoadingRows.vue"
session_ids:
  - "mnemon-review-v1-681d63ff-26c9-4d27-b15a-4fdc1e5a3eef"
memory_body_ids:
  []
---

# 回放翻译的渲染内核归属、正文替换三方案与 master 撤下翻译（!1125）

承接三份已有文档：
- f3474585-e700-4c3c-9a68-df5abd42e039：问题替换 / 答案叠加的呈现策略与 !1123 / !1124 的动效落地。
- b9e951c9-33cb-412e-9af8-853d163a3886：控件 Teleport 进 DetailModal 标题栏 + 只翻当前 turn（!1114 / !1115）。
- 62fa30a6-cbdc-4661-a99e-68d47604441f：最初的翻译可行性方案（答案用可折叠译文块绕开 PbiRecommendation）。

本文补两块此前未落纸的内容：回放与 Chat BI 是否为同一份渲染代码（用户 2026-11-19 追问，明确要求"先给我结论 先别改"，故本轮只查证未改动），以及把翻译从 master 摘掉后的分支状态。

## 1. 结论：外壳独立，渲染内核同一份

用户原话（历史记录）："真正替换正文呢， 我们不是复制过来的代码吗 没跟 chat BI 用同一份代码吧？ 先给我结论 先别改"

查证结果：
- src/components/PbiRecommend/PbiRecommendation.vue 全仓库只有这一份，没有副本。
- Chat BI 正式会话：src/components/Agent/AgentChatPanel.vue:27 import，:2145 用于 msg.role === 'pbiAssistant' 的分支。
- 回放弹窗：src/views/Setting/reports/history-replay/HistoryChatPanel.vue:3 import，:267 使用，同一路径同一个组件。
- 解析层 src/components/PbiRecommend/pbiParse.ts、pbiFold.ts 同样共用，不是回放侧另写的。

"复制过来的代码"这个印象只在壳层成立：
- HistoryChatPanel.vue 确实是从 AgentChatPanel.vue 的会话流派生出的回放专用版（气泡、工具栏、滚动都在这里），改它零风险。
- 正文不归它画，归 <PbiRecommendation>；回放对它是"只读包装"：外层套 <div class="history-pbi-readonly">，只传 pbiMsg 与 sendInput（指向空函数），不传任何交互态。

props 差异（这也是 A 方案回归面的具体范围）：

| 使用方 | 传入 props |
|---|---|
| Chat BI（AgentChatPanel.vue:2145） | pbiMsg、profileTaskAnswerStates、turnRunning(=loading)、isLatestRound(=msg.id === lastPbiMessageId)、priorSmartData、expandable、@sendInput |
| 回放（HistoryChatPanel.vue:267） | pbiMsg、@sendInput（忽略入参） |

## 2. "真正替换正文"的三条路（用户尚未拍板）

- A（推荐） 共用组件加可选 prop：textOverride?: string 或 renderText?: (node) => string；回放传、Chat BI 不传则走原分支。属"加能力"而非"改行为"，但上线前必须对 Chat BI 做一次回归，回归面按上表 props 逐项核对。
- B（现状） 不碰组件，只在回放侧叠加译文块。技术上做不到替换：正文文本是组件内部从 renderModel 现算的，外层遮不住。
- C 复制 PbiRecommendation 到 history-replay 目录：真正隔离、零回归，但 pbiParse.ts（1800+ 行）、pbiFold.ts（700 行）要跟着派生，Chat BI 升级解析逻辑时两份会分叉，维护成本高。

无论 A 还是 C，能替换的只有自然语言段落：finalAssistantText 与 profileTaskAnswers[].content。图表标题、表格单元格、SQL 代码块由结构化节点渲染，译文塞不进去，最多翻表头。

## 3. 把翻译从 master 摘掉（!1125）

用户要求（历史记录）："把master合并进去的翻译功能 摘出来把 先不发生产了"

- 分支 feature/remove-replay-translate-master → master，提交 b18fd372，MR !1125，state=opened、mergeable、无冲突。develop / tst 完全不动。
- 摘除范围（净 -889 / +2）：
  - 删除 src/views/Setting/reports/history-replay/translateApi.ts（108 行）。
  - HistoryChatPanel.vue 回退到翻译引入前 6e33bbd3^，逐行一致。
  - src/views/Setting/reports/user-logs/components/modals/DetailModal.vue 回退到 0c7f3108^，移除仅为翻译控件加的 #replay-toolbar-anchor 与 .dm-head-actions 样式。
- 刻意保留 f8cce319 中与翻译无关的部分：user-logs/components/ui/LoadingRows.vue 的指标卡骨架、user-logs/styles.less 的柔和微光、5 个 view 的 variant="metric"。理由：这些是通用视觉优化，撤掉会让 Core metrics 回到被用户吐槽的灰条。
- 校验：全仓已无 translateApi / replay-toolbar-anchor / translateTexts 引用；vue-tsc --build exit=0；两个文件与对应历史版本的 git diff 均为空（确认干净回退）。

### 后续必须知道的分叉

master 与 develop 现在在 HistoryChatPanel.vue 上已分叉；将来把翻译发到生产时直接合并该文件会冲突。干净做法有两个：把 master 上的翻译提交序列 6e33bbd3 → 56b8c8ea → 0c7f3108 → 8bcd1dd7 → f8cce319 重新 cherry-pick 到 master，或单独提一个"重新接入翻译"的 MR。

另注：!1121 / !1122（patch-id 9e1b1f40）与 !1123 / !1124 均已被合并且源分支被删，说明这批工作走"合并后即删源分支"的流程；下次续做要先 fetch，再基于最新基线重开分支（本轮 !1123 / !1124 就是这样重建的：a93d5b2b / f8cce319，patch-id 5a95306f，merge-tree 预检无冲突）。
