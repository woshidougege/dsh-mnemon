---
id: "25fb29f1-f532-44dc-b110-618cf4796fdc"
title: "ChatBI 问数记录 Source report 轮次级改造：生产数据复核、错标规模与实施方案"
description: "生产库（10.62.79.215/ssg_ops_agent，2026-09~11）复核：7446 条问题行中 1628 行报表错标（21.9%）、782 会话受影响；轮级 LATERAL 取值、四态 routed/route_failed/not_routed/no_audit、按名归并重名报表、统计/筛选/导出/反馈统一、验收样本与工作量，待定 clarify 轮口径。"
status: "active"
created_at: "2026-11-13T02:40:28.126Z"
updated_at: "2026-11-13T02:40:28.126Z"
content_hash: "d99da0b2d5f319581417f02f8760c01521745a9034249e1e0dffc40139f6a269"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java"
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue"
session_ids:
  - "mnemon-run-v1-eddc4c8e-09e4-45fa-9332-dd0dd54f3733"
memory_body_ids:
  []
---

# ChatBI 问数记录 Source report：会话级→轮次级改造（生产数据复核与实施方案）

状态：方案待用户拍板（clarify 轮口径 A/B/C 未定）。**本文基于生产库复核，取代 d7382448 中基于 TST 的结论**（该文 TST 结论"99.9% 审计缺失 / 需时间窗兜底"已被生产数据否证，原文未改动，仅作历史）。

## 1. 结论（一句话）

现状把"会话第一轮用了哪个报表"当成"每一行问题用的报表"：生产 7446 条 Smart Data Query 问题行中 **1628 行（21.9%）报表标错**，最常用报表被高估约四分之一，未路由被低估一个数量级。改造方向是**按轮（query_id）取路由结果**，并把"报表名"与"路由状态"拆开表达，界面不出现 report_id。

## 2. 生产数据证据（10.62.79.215 / ssg_ops_agent，2026-09-02 ~ 11-13）

规模：query_session 事件 8597 / 会话 6110；agent.tool_audit_logs 61295 行 / 6862 会话；sdq_reports 21 行。

- **关联健康**：按会话 98.2%（8441/8597 事件落在有审计的会话）、按轮 95.8%（8240/8597）；`Historical audit missing` 仅 0.2%（15 行）。月度 tracking→audit 覆盖：2026-09 102/108、2026-10 4445/4466、2026-11 1497/1537。
- **键可用**：可关联会话里 `turn_id = query_id` 命中 **28/28**；时间窗兜底 **0 命中** → 不做兜底，省掉这条复杂度。
- **错标规模（7446 行逐行比对）**：
  - 显示报表本轮根本没用过：**1628 行（21.9%）**
    - 本轮完全没路由 1540 行（其中 456 是 clarify 答复轮，1099 是真实问题但走了非路由路径）
    - 本轮用的是另一个报表：88 行
  - 路由失败仍照常显示报表名、无任何标记：434 行
  - 有 ≥1 行被错标的会话：**782 / 5135 已路由会话 = 14.8%**
- **口径位移**（会话级 today → 轮级 proposed）：
  | Report | today | turn 级 | 变化 |
  |---|---|---|---|
  | Services Bookings | 5261 (70.7%) | 4021 (54.0%) | −23.6% |
  | TCV | 1018 | 752 | −26% |
  | Daas Bookings | 413 | 265 | −36% |
  | Not routed | 144 (1.9%) | 1540 (20.7%) | ×10.7 |
  另有 `Services Bookings [route failed] 343`、`ISG PR [route failed] 53`、`TCV [route failed] 29` 等失败态。
- **一个会话命中多报表在生产是真事**：174 个会话（162 个 2 报表、12 个 3 报表）= 已路由会话 3.3%。
  - `9a4e5d1f-b262-495e-8b6d-87647b37f043`：12 轮先后 Daas Bookings / Services Bookings / TCV，页面 12 行**全部**显示 Daas Bookings。
  - `f0340d5e-a0c6-4f67-8252-dd4134ce09bd`：20 轮其中 12 轮未路由，页面 20 行全显示 Services Bookings。
- **重试普遍**：452 个会话有多条 route_report（6155 行 / 5339 会话；5710 succeeded、445 失败、428 会话带失败）；**20 个会话的第一次路由就是失败的**——现在取"最早一条"正好取到失败那次。
- **假问题轮**：647 条（7.5%）= 452 条 `User has answered your questions: -` + 90 条 `The user did not answer the pending ask_user_question` + 105 条空问题；轮型分布 normal 7953 / answered_clarify 452 / empty 105 / no_answer 90。456 条落在未路由桶，进一步放大错配。
- **sdq_reports 重名（生产同样存在）**：3 个 report_id 都叫 "Services Bookings" —— `simu`（active，primary_schema_table_id=20，knowledge_layout=single_v1）、`Services_Bookings`（inactive，2026-10-13 建档，改名残留）、`Services Bookings hw_sub_series_description`。不展示 id 就必须**按名字归并**，否则筛选下拉出现三个同名项各命中一部分数据。
- 会话问数条数分布：1 条 5108、2-4 条 806、5-9 条 172、10+ 条 29。

## 3. 目标模型

四种路由状态，id 不上屏：

| 状态 | 显示 | 判定 |
|---|---|---|
| `routed` | `Services Bookings` | 本轮 route_report 有 report_id 且 status=succeeded |
| `route_failed` | 名字 + 灰红点 `route failed` | 本轮调用了路由但失败（仍有 report_id） |
| `not_routed` | `Not routed`（灰） | 本轮没有 route_report（会话有审计） |
| `no_audit` | `No audit data`（浅灰） | 会话无 agent 审计 |

`Report ID missing` 从界面删除（本身是内部实现细节），改为后端日志 + ops 计数。report_name 缺失时 `sourceLabel = 'Unknown report'`，不再回落到 id。

## 4. 后端改动

**只保留 SQL 一套**，删掉 Java 侧的会话级映射（`ChatbiEventReportService.java:1144-1152` 的 `reportIdBySession.putIfAbsent`；`statusCategory`/`successText` 保留）。

**4.1 turn 级 LATERAL**（索引现成：`idx_tool_audit_session_turn (session_id, turn_id)`）：

```sql
LEFT JOIN LATERAL (
  SELECT t.report_id, (t.status = 'succeeded') AS route_ok
  FROM agent.tool_audit_logs t
  WHERE t.tool_name = 'route_report'
    AND t.session_id = <audit_sid>                  -- 现有 AUDIT_SESSION_ID_SQL（优先 upstream_session_id）
    AND t.turn_id = COALESCE(ssg_chatbi_event.query_id, '')
    AND COALESCE(<report_id 正则>, '') <> ''
  ORDER BY (t.status = 'succeeded') DESC,           -- 同轮重试：成功优先
           CAST(t.started_at AS timestamptz) DESC   -- 再取最后一次
  LIMIT 1
) tr ON TRUE
LEFT JOIN agent.sdq_reports sr ON sr.report_id = tr.report_id
```

- `started_at` 在 TST 是 text、生产是 timestamptz，必须 `CAST(... AS timestamptz)`（mapper 已有同坑注释）。
- 同轮多次路由取"最后一次成功的"，避免"首次失败也当来源"。

**4.2 结构**：LATERAL 现在只在带 `sourceReport` 筛选时挂（`ChatbiEventQueryMapper.java:558-559` `SESSION_FROM_WITH_SOURCE`、`:630-636` `FROM_EVENT_FOR_SOURCE`）。明细要出这一列，须挂进 `SESSION_FROM`，删两个 conditional 分支，`SESSION_SOURCE_FILTER` 只留 CASE 条件。

**4.3 输出字段**（`SESSION_SELECT` 1295-1328、`SESSION_EXPORT_SELECT` 1361-1388，抽 `ROUTE_SELECT` 公用）：

```
sourceKey    -- 'report_name:services bookings' | '__not_routed__' | '__no_audit__'
sourceLabel  -- 显示文本
routeState   -- routed | route_failed | not_routed | no_audit
```

`sourceKey` 用**名字**做 key（不是 id），改名后 simu / Services_Bookings 自动归并，筛选命中同一批数据。

**4.4 口径同步**：`sessionSourceStats`（1574-1587）按 `sourceKey` 分组并加 `routeFailed` 计数；`sessionTopQuestionsBySource`（1611-1640）随换；导出 `ChatbiEventExportService.java:845-878` `sourceStatsTable` 改用 `routeState`；反馈页走 `querySessionsPageBySessionIds`（`ChatbiEventQueryMapper.java:1519`）复用 `SESSION_SELECT`，只需 `ChatbiEventReportService.java:342-345` `FEEDBACK_TURN_FIELDS` 加 `routeState`。

## 5. 前端改动（`QueryView.vue`）

- **Report 列**（379-383）：只显示名字；四态用不同字重/颜色（routed 正常、route_failed 名字正常+灰红后缀、not_routed 灰、no_audit 更浅）；`:title` 只放"本轮路由状态 + 报表名"，不放 id。
- **Session title 列**（352-370）：`n/N` 角标**永远显示**（含 1/1，现在 count=1 不显示，导致"另一个会话"看起来像同一会话第三行）；同 session 行加左侧色条/浅底，组内仅首行显示标题，其余 `↳ same session`；标题 42 字符截断改为"组头完整显示 + 行内短标题"。
- **Question 列**：空值回退 `session_title` 时加灰标 `from title`；Question 点击开详情、Session title 点击过滤会话，两者都做成 hover 链接色以区分。

## 6. 验收（生产金样本）

| 会话 | 现状 | 改后 |
|---|---|---|
| `5c5c8799`（2 轮） | 两行 Services Bookings | 第 1 行 `Services Bookings · route failed`，第 2 行 `Not routed` |
| `9a4e5d1f-…`（12 轮 3 报表） | 全 Daas Bookings | 三个报表各出现在对应行 |
| `f0340d5e-…`（20 轮 12 未路由） | 全 Services Bookings | 12 行 Not routed |
| 20 个首轮失败会话挑一个 | 取到失败那次 | 取成功那次 |

单测（沿现有 SQL 测试范式）：同轮多次路由取成功；本轮无 route_report；会话无审计；report_name 缺失 → Unknown report；同名多 id 归并；改名前后归并。

## 7. 性能与工作量

- 明细页每行一次 `(session_id, turn_id)` 索引探测，挂上 `SESSION_FROM` 后列表/导出/反馈都带，需在**生产样本**跑 `EXPLAIN ANALYZE`（TST 8452 行不具代表性）；预期增量 < 50ms，超了退回"只明细页挂、统计页单算"。
- P0-a 后端（LATERAL + 字段 + 统计/筛选/导出/反馈统一 + 单测）1.5 人日；P0-b 前端状态化 + 角标常显 + 下拉按名去重 0.5~1 人日；P0-c `EXPLAIN ANALYZE` 与灰度（先只看不改筛选）0.5 人日；P2 上报侧在轮次结束时把 `report_id`/`route_state` 写进 `query_session.properties`（跨仓库，第二阶段）。
- 不做：不展示 report_id、不改 agent schema（只读）、不加表不加迁移、历史数据不重算、不做时间窗兜底。

## 8. 待决策 / 遗留

- **clarify 轮口径 A/B/C**：A 过滤掉（表干净，但 n/N 变化、历史导出对不上）；B 保留并标记 `turnType = question|clarify_answer|system`，默认显示但置灰且不计入 Question count/报表统计/成功率（推荐）；C 只修报表归属。
- TST 环境埋点↔审计几乎不重合（10 月 4466 vs 0）是**环境问题**，与代码无关，单独提单给环境 owner，不阻塞本次开发；所有数据分析以生产为准。
- 生产踩坑记录：探查脚本里 `ev_question` 列名不存在（应为 `ssg_chatbi_event.question` 一族）导致 `UndefinedColumn: column "ev_question" does not exist`；psycopg3 传 `%` 需转义（`only '%s', '%b', '%t' are allowed as placeholders`）。
