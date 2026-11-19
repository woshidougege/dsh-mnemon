---
id: "62fa30a6-cbdc-4661-a99e-68d47604441f"
title: "历史回放弹窗多语言翻译方案（问题气泡 + 答案译文块，复用 gpt-4o 通道）"
description: "user-logs 历史回放加翻译的可行性结论：复用后端 openai/gpt-4o 通道与 SchedulerCronAiService 调用范式，分层范围（问题气泡必做、答案用可折叠译文块绕开 PbiRecommendation、图表不翻）、七项配套（批量、sha256 缓存表、四语言+custom、术语锁词、markdown 结构、默认关、合规）与 2-2.5 天估算，待用户确认开工。"
status: "active"
created_at: "2026-11-19T07:36:37.017Z"
updated_at: "2026-11-19T07:36:37.017Z"
content_hash: "77c3a26fb220e91428ddb78d2ac9017dd2107bef0bd5d65eb19617d4b4e7c8ec"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/scheduler/service/SchedulerCronAiService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chat/model/nostream/NoStreamOpenAI.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chat/config/OpenAiConfig.java"
  - "smartop-service/admin/src/main/resources/application-lmp-reston-tst.yml"
session_ids:
  - "mnemon-review-v1-f0c14e08-62f6-4630-8473-70a385e39129"
memory_body_ids:
  []
---

# 历史回放弹窗多语言翻译（问题气泡 + 答案译文块）可行性方案

状态（2026-11-19）：**结论已给出，等用户确认是否开工**，未改任何代码。

## 1. 结论

可以加，而且是低成本加法：**后端复用现成的 OpenAI 通道，前端只加两处 UI**，不改线上聊天页。范围建议只覆盖「回放弹窗」：① 用户问题气泡翻译 ② AI 答案追加可折叠译文块；图表/表格/列名/SQL 第一版不翻。预估 2～2.5 天。

## 2. 已核实的基础设施事实（为什么"好加"）

- 后端已有 `openai` 配置块：`application-lmp-reston-{reston-pre,reston-dev,reston-tst,reston-prod}.yml:83-90` → `model-name: gpt-4o`，base-url 指向 Lenovo aiverse 内网网关（`/ics-apps/projects/115/salesops/aiverse/endpoint/v1`）。**不需要接入新模型能力**。
- 现成的一次性（非流式）调用范例：`smartop-service/admin/src/main/java/com/lenovo/admin/scheduler/service/SchedulerCronAiService.java:67-73` —— `noStreamOpenAI.getNoStreamOpenAIModel(modelName, new HashMap<>())` + `model.chat(ChatRequest.builder().messages(...).temperature(0.0).build()).aiMessage().text()`，prompt 走 `classpath:prompts/scheduler/cron-expression.txt` 模板 + `{{var}}` 替换。
- 相关装配点：`smartop-service/admin/src/main/java/com/lenovo/admin/chat/model/nostream/NoStreamOpenAI.java`、`smartop-service/admin/src/main/java/com/lenovo/admin/chat/config/OpenAiConfig.java`（`@ConfigurationProperties("openai")`）。
- 翻译接口的落点：问数记录（user-logs）报告的既有前缀 `API_PREFIX = '/api/admin/chatbi/events/report'`（`ssg_sales_ops/src/views/Setting/reports/user-logs/constants.ts:11`），新增 `POST /api/admin/chatbi/events/report/translate`，入参 `texts[] + targetLang`。

## 3. 分层翻译范围与 rationale

| 层 | 结论 | 具体位置 | 风险 |
|---|---|---|---|
| ① 用户问题气泡 | 必做 | `C:\\Users\\63588\\lenovo\\ssg_sales_ops\\src\\views\\Setting\\reports\\history-replay\\HistoryChatPanel.vue:252-260`，就是单个 `message.content` | 零 |
| ② AI 答案正文 | 要做，但**不原地替换** | `finalAssistantText`、`profileTaskAnswers[].content`（markdown），字段定义见 `PbiRecommend/pbiParse.ts:611-637` 的 `PbiRenderModel` | 中 |
| ③ 图表/表格/列名/SQL/季度标签 | 不做 | `smartDataNode` 与 echarts 配置内部 | 高 |

第 ② 层的核心判断：AI 答案不是回放组件自己渲染的，而是交给公用组件 `C:\\Users\\63588\\lenovo\\ssg_sales_ops\\src\\components\\PbiRecommend\\PbiRecommendation.vue`（真实聊天页也在用）。**在组件内改文本会污染线上聊天页，回归面大。**
因此采用：回放气泡下追加一个可折叠「译文」块，用纯 markdown 渲染译文，完全绕开 `PbiRecommendation`。翻译只影响阅读，不动图表卡片。这样日文/其他语种答案看不懂的问题即解决，且聊天页零回归。

回放弹窗挂载点（同一父页面）：`ssg_sales_ops/src/views/Setting/reports/user-logs/index.vue:573`（`sessionDetailRef`）、`:612`（`openSession(sessionId, title, turnId)`）、`:1495`（`<DetailModal ref="sessionDetailRef" @review-saved="onReviewSaved" />`）。

## 4. 必须一起做的配套（不做会踩坑）

1. **批量发送**：一次请求合并整条会话的多条文本，不要每条消息一个请求。
2. **服务端缓存（硬性）**：键 = `sha256(原文) + 目标语言 + model`，落表；重复打开同一会话直接返回译文，否则每次点开都产生模型费用。
3. **语言定义**：固定 `zh / en / ja / es` + `custom` 自定义输入（限 40 字符、字符白名单，防止把 prompt 注入当"语言名"传进来）。
4. **术语锁词**：DaaS、TCV、Fiscal Qtr、Smart Data Query、PBI 等不翻译，否则中文译文里看不出对应指标。
5. **markdown 结构**：prompt 明确要求保留列表/表格/代码块结构；结构损坏时退回纯文本，不要渲染坏结构。
6. **默认关、按需翻**：顶部语言下拉（Original / English / 中文 / 日本語 / Español / Custom…），选择后才翻译。不做"打开即全翻"，否则每次等 2-3 秒且易触发限流。
7. **合规确认**：会话正文会发送到 aiverse 内网网关（该网关目前已在跑 scheduler cron 生成），需业务侧确认可接受。

## 5. 工作量

- 后端（translate 接口 + prompt 模板 + 缓存表 + 单测）：约 1 天
- 前端（语言下拉 + 气泡译文 + markdown 渲染 + 缓存调用 + 原文/译文切换）：约 1 天
- 联调自测：0.5 天
- 合计 2～2.5 天；若还要翻图表与表格，再加约 2 天且必须改动公用组件，第一版不建议。

## 6. 未决 / 下一步

- 等用户确认是否按上述范围实施（只在回放弹窗内生效，不碰聊天页与 `PbiRecommendation`）。
- 未定：aiverse 网关合规确认、缓存表命名与迁移版本号、custom 语言的校验细则。
