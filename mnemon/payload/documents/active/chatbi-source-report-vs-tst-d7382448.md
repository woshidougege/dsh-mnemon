---
id: "d7382448-aaf9-4003-971f-58f1f56da61b"
title: "ChatBI 问数记录三列取值来源与粒度缺陷排查（Source report 会话级 vs 轮次级，TST 实测）"
description: "问数记录页 Question / Session title / Source report 三列的取值链、四类根因（粒度错位、失败路由误用、sdq_reports 重名、口径扩散）、TST 实测数据缺口（23/6030 会话可对齐）、按 turn 取值的修复 SQL 与改法顺序。"
status: "active"
created_at: "2026-11-13T02:19:16.079Z"
updated_at: "2026-11-13T02:19:16.079Z"
content_hash: "afff8f6063cbca0bffc34ba6ab1c8d849b29f0204c81101f7eba57deea5fba10"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiAgentSessionMapper.java"
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue"
session_ids:
  - "mnemon-run-v1-f405baa7-94ae-46a8-8ec8-f4be6f4519ed"
memory_body_ids:
  []
---

# ChatBI 问数记录三列（Question / Session title / Source report）取值来源与粒度缺陷排查（TST 实测）

结论：领导反馈的「Source report」列"不对"成立，成因是三层叠加——**统计粒度错位（会话级 vs 轮次级）、失败路由被当作来源、报表名重名**。同一套口径还扩散到卡片、筛选、导出、反馈页。相关既有交付记录见 **ea0c4abe-9366-4b26-964c-e4b8149e3fad**（人工评审/Accuracy Rate，同为 `decorateSessionRows` 调用方，改动需同步）。

状态：排查完成并给出改法，尚未动代码（父会话末尾在问是否先只改后端 + 单测）。

## 1. 三列取值链

| 列 | 取值 | 位置 |
|---|---|---|
| Question | `question` → `properties.question_content` → `original_question` → **兜底 `session_title`** | `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java:657-661` |
| Session title | `ssg_chatbi_event.session_title`，前端截断 42 字符，角标 `n/N` 仅 N>1 时显示 | `ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue:352-370` |
| Source report | **整个 session 最早一条 `route_report`** 的 `report_id` → `sdq_reports.report_name`，否则三档占位符 | `ChatbiEventQueryMapper.java:539-578`（`ROUTE_REPORT_ID_EXPR` + `LEFT JOIN LATERAL … WHERE t.tool_name='route_report' ORDER BY t.started_at LIMIT 1`）、`ChatbiEventReportService.java:1144-1205` |

关键两行决定了一切：SQL 侧 `ORDER BY t.started_at LIMIT 1`，Java 侧 `reportIdBySession.putIfAbsent(sid, rid)`——**表格按"问题·轮次"逐行，Source report 却按会话取值且固定取第一条**。

## 2. 四个具体缺陷（含实测）

**① 粒度错位**：会话 `5c5c8799`（11-12 16:09 / 16:17 两轮）——第 1 轮 turn `191fb6d3` 的 `route_report` 选中 `report_id=simu` 且**该步为 failed**；第 2 轮 turn `eec634c7` 工具链为 `Skill / read_file / tool_search / revise_query×3 / search_memory`，**未调用 route_report**。页面两行都显示 "Services Bookings"（`simu` 在 `sdq_reports` 中的名字）。
真·一会话两报表的也存在：`0e9dcd73` 第 1 轮 `upsell_svc_offering`（PN Master）、第 2 轮 `offering_hier`（Offering Hierarchy），相隔 1 分钟，两行都显示 "PN Master"。
全库 11 个会话有多条 route_report，其中只有 `0e9dcd73` 换了报表，其余多为同报表重复（如 `1fcd36da`/`21cac058` 三次 simu、`d6c896e7` 两次 upsell_svc_offering 后一次 failed）。

**② 失败路由被当成来源**：413 条 `route_report` 中 339 succeeded / 74 failed；72 个会话有失败记录，**70 个会话一次都没路由成功但审计里仍带 report_id**，页面照样显示报表名——与同一行的 Failure category `Business validation failed / route_report: tool_business_error` 自相矛盾。

**③ 报表重名**：`sdq_reports` 中 `report_id='simu'`（Services Bookings）与 `report_id='Services_Bookings'`（Services Bookings hw_sub_series_description）的 `report_name` **同为 "Services Bookings"**，列里不带 id 则无法区分。

**④ 口径扩散**：顶部 "Smart Data Query Report Source" 卡片按**会话级** `sourceKey` 分组计数（`ChatbiEventQueryMapper.java:1574-1587`）；`sourceReport` 筛选同样按会话过滤（勾报表 B 命中"首轮是 B 的会话"）；导出 `ChatbiEventExportService.java:845-878` 与反馈页（`FEEDBACK_TURN_FIELDS` 复用同一批字段）同源。任何一处单独改会让卡片与明细互相矛盾。

## 3. 环境级数据缺口（TST，决定这一列当前可观测性）

- 埋点侧 6030 个 query_session 会话，session_id 能对上 `agent.tool_audit_logs` 的只有 **23** 个；agent 库 1663 个会话中有 1640 个在埋点表里找不到。
- 因此 7323 条 Smart Data Query 问题里，来源标签分布为 **Historical audit missing 7320 / Services Bookings 2 / Not routed 1**：这一列在 TST 基本只在显示"审计缺失"。
- 可用性正面证据：tracking 的 8470 条 query_session 全部带 `query_id`，audit 侧 413 条 route_report 全部带 `turn_id`，且 `turn_id = query_id` 能对上（`5c5c8799` 的 7 条审计行全部按 turn 对齐）——改造有键可用，不依赖时间窗猜测。

## 4. Session title 与 Question 的设计问题

- 非会话标识：6031 个会话中 **5871（97.3%）的 session_title 等于首问原文**，与 Question 首行完全重复。
- 跨会话同名：**5037（83.5%）**会话标题在别的会话里也出现过（424 个重复标题）；截图那行 "TuringCode Frontend" 其实来自另一个会话 `ad023302`（单问题、4 条工具记录、无 route_report → "Not routed to a report"），三行看似一个会话实为两个。
- 分组线索只有 `1/4…4/4` 角标且 count=1 时不显示；行序是全局 `event_time DESC`（`4/4` 在上、`1/4` 在下，阅读顺序反）；session id 仅在 hover tooltip。
- Question 空值静默回退成 session_title，老数据两列同一句话；截断 72 字符；与 Session title 都是文字按钮但点击行为不同（Question→打开会话详情并定位该轮，Session title→过滤到该会话），视觉无区分。

## 5. 建议改法与顺序（性价比降序）

1. **计算规则改按轮次**（多轮会话在 TST 占 990 个 / 16.4%，不改则永久错）：
   ```sql
   SELECT DISTINCT ON (turn_id) turn_id, report_id, status
   FROM agent.tool_audit_logs
   WHERE tool_name = 'route_report' AND session_id = :auditSid
   ORDER BY turn_id, (status = 'succeeded') DESC, started_at DESC
   ```
   关联 `turn_id = query_id`；对不上的历史数据用时间窗兜底：`route_report.started_at ∈ [本轮 event_time, 下一轮 event_time)`。卡片 / 筛选 / 导出 / 反馈页必须同步，否则口径互斥。
2. **拆列而非塞占位符**：`Report`（真名 + 小字 report_id）与 `Route status`（Routed / Route failed / Not routed in this question / No audit data）。现在业务值与数据质量占位符混在一列同一灰色，是最大可读性损失。
3. **按会话分组展示**：组头放 标题 + 短 session id + "n 个问题"，行内只留 `n/N` 与 Question，不再逐行重复标题。
4. **治理 `sdq_reports` 重名**，或列里始终带 id。
5. 若先落地，倾向"只改后端 + 单测，页面先不动"，待看过明细数据结构再定 UI。

## 6. 相关代码位置速查

- 会话级报表映射：`ChatbiEventQueryMapper.java:536-578`（注释明示"取每个会话最早一次 route_report 选中的 report_id"）；`ChatbiAgentSessionMapper.java:125,134,152`；`ChatbiAgentBackfillMapper.java:32`。
- 装饰/补全：`ChatbiEventReportService.java:1008` `decorateSessionRows`、`:1023`/`:1125` `enrichSessionDiagnostics`、`:820-830` 概览表、`:833-843` `dedupeSessionsByQueryId`、`:845-878` `sourceStatsTable`。
- 超时/失败文案：`ChatbiEventQueryMapper.java:503`、`ChatbiEventReportService.java:1294`（`case "route_report" -> "Timeout · Report routing"`）。
