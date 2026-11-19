---
id: "00c8b297-ee92-4caf-b3cb-01943ef6b4b5"
title: "回放翻译：接口层方案否决理由与「原文对照 / 反问未翻 / 大屏末轮」三个反馈的根因"
description: "ssg_sales_ops 回放翻译追加查证（只读、未拍板）：为何翻译不能下沉到接口层（SSE 逐事件 + 正文由前端 pbiFold 折叠 + 产品形态退步）；领导反馈的三个缺陷各自的根因行号、改动范围与风险，以及先 2.3+2.1 后 2.2 的落地顺序和「图表/SQL/内置文案仍英文」的能力边界。"
status: "active"
created_at: "2026-11-19T10:59:41.065Z"
updated_at: "2026-11-19T10:59:41.065Z"
content_hash: "f68db882234ec1edf0599e8a865bd10851da1d5aac7ed78c05a9470d84659f39"
source_paths:
  - "src/views/Setting/reports/history-replay/HistoryChatPanel.vue"
  - "src/components/PbiRecommend/PbiRecommendation.vue"
  - "src/components/Agent/AgentChatPanel.vue"
  - "src/components/PbiRecommend/pbiParse.ts"
  - "src/components/PbiRecommend/pbiFold.ts"
session_ids:
  - "mnemon-review-v1-f92def5e-b993-4f91-9578-14d47a9fe710"
memory_body_ids:
  []
---

# 回放翻译：接口层方案否决理由与三个领导反馈的根因

承接已有文档：
- f8c73c4a-a6ef-491e-a80c-96d0df436d17：回放与 Chat BI 的渲染内核归属（外壳独立、内核同一份）、正文替换 A/B/C 三方案、!1125 从 master 摘掉翻译。
- b9e951c9-33cb-412e-9af8-853d163a3886：控件 Teleport 进 DetailModal 标题栏、只翻当前 turn（!1114 / !1115）。
- 62323dec-a57b-4d04-8fe1-21736166b548：全手动化、视口 1/3 锚定当前轮（!1121 / !1122）。

本文记录 2026-11-19 的追加查证。用户要求「先给结论、先别改代码」，故本轮纯只读分析，方案尚未拍板；下面所有改动建议都不是已落地事实。

## 1. 翻译不能下沉到接口层——三条独立理由

1. **实时会话根本没有「一段完整正文」**。发问后是 pbiApi.getSessionRuntime(sessionId) 取运行时快照，再 startRuntimeStream(...) 订阅 SSE（services/pbiRuntimeStream.ts:417 的 fetchEventSource；src/components/Agent/AgentChatPanel.vue:1162 封装、:1810 直连）。事件逐条到达（如 tool_call_finished），前端累积进 rawEvents 后才由 pbiFold 折叠成 renderModel。接口层要翻只能逐事件翻，每段都要等模型返回：实测 519 token 中文约 1.x 秒、658 token 日语约 3.3 秒 → 流式输出变成一顿一顿，且用户可能只想看英文，成本白花。
2. **回放接口给的是原始事件，不是排版好的文本**。/session-detail 虽然整包返回，但内容是 rawEvents 数组，正文由 buildPbiRenderModel → pbiFold 在前端算出（finalAssistantText、profileTaskAnswers[].content）。后端若要翻，必须先复刻这套折叠逻辑（src/components/PbiRecommend/pbiParse.ts 1800+ 行、pbiFold.ts 700 行）→ 等于用 Java 重写前端渲染内核，之后 pbiParse 一改就要双份同步维护。
3. **产品形态会退步**：全篇译文、无原文对照、切语言要重新拉取整个会话（几 MB rawEvents），与已定策略「只翻当前问题、手动点才翻、随时切回 Original」直接冲突。

结论：翻译接口（POST /chatbi/events/report/translate，单次可翻一批文本）留在服务端，能力做进共用组件，触发点留在前端。实时会话可以省一步：**只翻答案、不翻问题**——问题由用户自己输入；回放里的问题来自客服/产线的英文留言，才需要翻。

## 2. 三个反馈的根因与改动面

### 2.1 「没有原文对照」

src/views/Setting/reports/history-replay/HistoryChatPanel.vue 的 userBubbleText()（:350-354）是**替换式**：

```ts
if (isShowingOriginal(message.turnId || '')) return original
return translatedText('user', index) || original   // 翻了就把英文换掉
```

模板里用户气泡只有一行文本（:661-669），**没有任何原文/译文切换入口**；Original 开关只做在 assistant 译文块上（:685-691 的 translate-panel-toggle）。标题栏按钮其实能切回（translateActiveTurn() :457-474，isTurnTranslated 命中时走 toggleOriginal），但它的文案会随状态翻转成 Original，读者只会理解成「又一个翻译按钮」。

建议改法：双语气泡——原文常驻，译文作为下方小字段落，气泡右上角加很轻的 `中文 / EN` 可点标签。改动只在 HistoryChatPanel 单文件，零风险。

### 2.2 「反问没翻译 = 翻译不全」

assistantPlainText()（:373-385）只抽两处：

```ts
if (model.finalAssistantText) parts.push(model.finalAssistantText)
;(model.profileTaskAnswers || []).forEach((answer) => {
  if (answer?.content) parts.push(answer.content)
})
```

而一轮回答里这些文本**一条都没进翻译队列**：

| 字段 | 是什么 |
|---|---|
| profileTaskInputs[].inputs.questions[] | 子 Agent 反问卡的问题 + 选项 |
| inputNode.inputs.questions[] | 主 Agent 反问 |
| escalation.reason | 反问里「为什么需要你补充」的说明 |
| recommendationNodes[].recommendations[] | What drove this? 建议卡 |
| profileTasks[].requestSummary | 子任务进度摘要 |

反问卡恰恰是带按钮的显眼卡片，整块英文挂着，第一眼就是「翻译不全」。

关键约束：反问卡由 PbiRecommendation 内部渲染（HistoryChatPanel.vue:676-680 只传 pbiMsg 与指向空函数的 @sendInput），**外层遮不住、塞不进译文**。所以这块必须走 f8c73c4a 里的方案 A：给共用组件加只读映射（如 translatedTexts，按问题原文查译文），渲染时译文优先、原文留在下面。会连带影响 Chat BI 实时会话，回归面按 f8c73c4a 里的 props 差异表逐项核对。

易漏点：反问卡的 **options 也要翻**（This quarter / Last quarter 这类），不翻就是半中半英，比不翻更难看。

### 2.3 「大屏下最后一个问题翻不了」

syncActiveTurn()（:525-541）只认「顶部落在视口 32% 线以上」的 user 行：

```ts
const anchor = viewTop + viewHeight * 0.32
rows.forEach((row) => { if (row.getBoundingClientRect().top <= anchor) next = row.dataset.turnId || next })
```

大屏容器很高，一屏能装下好几轮甚至全部内容 → 最后几轮的问题行**永远在 32% 线下方**，滚到底也推不上去 → activeTurnId 一直停在前面的轮次，按钮永远对着前面的问题，最后一轮点不到。

建议补丁：滚动到底时直接取最后一轮；内容不足一屏（scrollHeight <= clientHeight）时直接取最后一轮；锚点由 32% 调到 45~50% 也合理。同样只在 HistoryChatPanel 单文件。

## 3. 建议落地顺序与能力边界

| 问题 | 改动范围 | 风险 | 工作量 |
|---|---|---|---|
| 2.3 大屏末轮 | HistoryChatPanel 单文件 | 低 | 小 |
| 2.1 原文不可见 | HistoryChatPanel 单文件 | 低 | 小 |
| 2.2 反问未翻 | + PbiRecommendation 共用组件 | 中（需回归 Chat BI 实时会话） | 中 |

先做 2.3 与 2.1（当天可出、领导立刻能看到变化），2.2 单独做并连 Chat BI 实时会话一起验证。

预期边界必须提前对齐：2.2 做完也不是 100% 中文——图表标题、表头、SQL 代码块、组件内置按钮文案（Export 之类）仍是英文，这些属于结构化渲染或组件内置文案，要全覆盖得另开一轮。

## 4. 校验

本轮所有结论来自对上述文件的只读检索（HistoryChatPanel.vue 全文 1017 行、pbiParse.ts 关键接口段），未改任何代码；type-check 基线 exit=0（残留引用：无）。
