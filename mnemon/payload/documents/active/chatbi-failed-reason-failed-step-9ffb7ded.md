---
id: "9ffb7ded-d7a0-4b81-824f-1c9021daa281"
title: "ChatBI 埋点失败归因数据核查与环比可行性（failed_reason / failed_step 实测分布）"
description: "统计报表 Failed 节点「归因层 / 环比」的数据可行性核查：失败归因写在 query_session 的 properties JSON（failed_step / failed_reason / failed_duration_ms，见 ChatbiChatReporter.java:139-149、211-219），tst 979 条失败实测 reason 6 枚举与 step 分布；环比受 queriesSummary 单时间窗与仅一周数据限制；含「别只看列名、JSON 属性里才有数据」的误判教训；关联 5173 token 文档 e8298f54。"
status: "active"
created_at: "2026-11-20T09:56:21.523Z"
updated_at: "2026-11-20T09:56:21.523Z"
content_hash: "125515c74cb593bbe370f890ee64ec19bbcc516733f470d0ff7098d020abec1e"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiChatReporter.java"
session_ids:
  - "mnemon-review-v1-193d145a-39fd-47c8-9632-3e972d600c00"
memory_body_ids:
  []
---

# ChatBI 埋点失败归因数据核查与环比可行性

**背景**：ssg-sales-ops 统计报表（领导看板的漏斗/分层图）需要给 Failed 节点补两层——"归因层"（为什么失败）与"环比"（上一个等长周期对比）。这次排查先确认数据到底有没有，而不是先改代码。

## 结论速览

- **归因层：数据齐全，可以做，不用造分类**。失败原因不是独立列，而是存在 `query_session` 事件的 `properties` JSON 里。
- **环比：确实没有数据支撑**。接口只吃单时间窗，且库里只有一周数据，上一周期基本是空窗口。

## 归因字段位置与写入逻辑

代码：`smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiChatReporter.java`

- `query_session` 事件写入三类归因属性：`failed_step`（哪一步失败）、`failed_reason`（原因枚举）、`failed_duration_ms`（失败前耗时），见 :139-144。
- 没有可用失败步骤时补默认值：`failed_step = "答案生成"`、`failed_reason = "no_answer"`，见 :147-148。
- 原因解析 `resolveFailedReason()` :211-219：answer 为空 → `empty_response`；有 `answer.error` → 截断该 error 文本；否则 → `empty_answer`。
- 调用链抛错时会显式传入 `failureReason`（如"召回为空"），优先级高于从 answer 推断，见 :95-96。
- 埋点是旁路能力：`reportKnowledgeQuiz` 整体 try/catch，异常只记日志不影响问答返回（:106-109）；用户身份优先取 `UserInfo.getItcode()`，取不到再解析网关 `USERINFO` 头，都拿不到就跳过上报，绝不用请求体里的 userName（:187-205）。

## tst 实测分布（2026-11-20 快照，979 条失败）

| failed_reason | 条数 |
|---|---|
| tool_business_error | 331 |
| timeout | 326 |
| unknown_error | 226 |
| (null) | 55 |
| empty_response | 9 |
| quality_failure | 7 |

去重后 reason 只有 **6 个值**，是现成的受控枚举，不是自由文本。

failed_step 分布：`route_report` 437、`execute_compiled_query` 325、`revise_query` 85、(null) 33、`finalize_query_result` 22、`search_knowledge` 21、`resolve_plan` 20、`execute_sql` 11、`query_sdq_catalog` 9、乱码 9、`run_data_query` 4、`query_pbi_catalog` 3。那 9 条乱码 step 名是历史数据编码问题，非新 bug。

## 一个必须记住的误判教训

第一轮排查是**按列名找**（`error_code`、`failure_type` 之类），没找到就下了"归因层没数据"的结论——**这是错的**。原因存在 `properties` JSON 里，必须按 `properties ->> 'failed_reason'` 这类 JSON 取值方式来查。以后再遇到"某维度缺数据"的判断，先确认埋点是不是把字段塞进了 JSON 属性，别只看表列。

## 环比为什么难做

- 接口 `queriesSummary` 只接一个时间窗（start/end），没有"上一周期"入参。
- 要做必须由后端用**完全相同的一套筛选条件**（user、team、mode、band、source、report、review status、tag，共 8 个）再跑一遍上一等长窗口，返回两组数给前端算 delta。
- 成本与数据双重限制：这个聚合查询本身不轻，首屏多跑一遍很贵；而且库里只覆盖 2026-11-12 ~ 11-18 一周，上一周期基本为空，环比会显示 N/A 或从 0 跳到 N。
- 建议：环比只做 Question count、Success rate、Accuracy 三个核心指标，不做全表；等数据攒够两周以上再上更有意义。

## 未决口径（等用户拍板）

归因层按什么切分还没定，两个方案：

1. 按 **failed_reason 的 6 类**（timeout、tool_business_error、unknown_error…）——回答"为什么失败"；
2. 按 **failed_step 的业务步骤**（route_report、execute_compiled_query…）——回答"失败在哪一环"。

领导样图里那一层看起来更像第 2 种（步骤视角）。

## 关联文档

- 5173 前端 token 为空 / vite 代理（auth→tst、admin→本地 8082）/ dev 与 tst 共用 Redis 的完整结论见 `e8298f54-6834-4b0f-99aa-aeab603ea881`，此处不重复。
