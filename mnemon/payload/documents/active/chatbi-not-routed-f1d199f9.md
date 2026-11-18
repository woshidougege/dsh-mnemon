---
id: "f1d199f9-9322-4c39-b7f1-ca8160e6fc52"
title: "ChatBI Not routed 成因分类与\"沿用上轮路由\"可行性裁决（生产实测）"
description: "Not routed 只表示本轮未再调 route_report：30 天 4912 轮中 1201 未路由（追问 1026/全会话未路由 175/关联假阴性 219），其中 686 轮实际出数；revise_query 经 planned_state.py:112/114/161/665 锁定同报表，与上次路由一致率 731/732=99.86%，故读本轮工具输出 report_id（87.3% 自带）优于跨轮回退；含五层分层规则与三处改动位置。"
status: "active"
created_at: "2026-11-18T01:59:51.405Z"
updated_at: "2026-11-18T01:59:51.405Z"
content_hash: "945b766e288a86b6ca4361b64a48ef3b59b68387c48151133fcd035d7900694e"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java"
session_ids:
  - "mnemon-run-v1-c037a933-4b43-46bc-8ebd-07cc9ad0eb20"
memory_body_ids:
  []
---

# ChatBI 问数记录 "Not routed" 成因分类与"沿用上轮路由"可行性裁决

来源：2026-11-18 对 `ssg-sales-ops` 生产库（个人只读账号 `p_renxp4`）的实测 + `ssg-spark`（同级目录）源码核对。关联文档：`25fb29f1-f532-44dc-b110-618cf4796fdc`（轮次级改造与错标规模）、`d7382448-aaf9-4003-971f-58f1f56da61b`（三列取值链与 TST 实测）。

## 1. 结论先行

- **`Not routed` 不是路由故障**，它只表示"本轮没有再调用 `route_report`"。
- 追问轮（换维度、加筛选、拆分、回答澄清）由 `revise_query → execute_compiled_query` 继续沿用同一报表，因此天然没有 `route_report` 审计行。
- 同事提出的"revise_query 用上个路由结果"**机制上成立**，但**不该实现成跨轮回退**；应改为直接读取本轮工具输出里本来就带的 `report_id`（更准、更简、无需时间对齐）。
- **不能把 `Not routed` 当作"这轮没出数"**：1201 个 Not routed 轮里有 686 轮实际执行 SQL 且 `status=success`。

## 2. `Not routed` 的判定链（源码）

`smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java`

- `:564-574`、`:595-599`：把 `agent.tool_audit_logs` 中 `tool_name='route_report'` **且 `turn_id` = 本轮 `query_id`** 的记录 LEFT JOIN 进来。
- 有记录 → 取 `report_id`，再查 `agent.sdq_reports` 得到报表名；没有 → `Not routed`。
- `:556-560` 注释：当前实现**刻意不跨轮**取值。
- `route_report` 不是模型每轮都调用的普通工具，而是 runtime 完成报表路由后的后续动作（`ssg-spark/app/ssg/smart_data_query/tools/search_knowledge.py:1400-1470`，`when_to_use: Only as a trusted runtime completion action after report routing`）。只在"发生路由的那一轮"留审计。

## 3. 生产实测（近 30 天，Smart Data Query 模式，4912 轮）

| 状态 | 轮次 | 占比 |
|---|---|---|
| routed | 3697 | 75.3% |
| Not routed | 1201 | 24.4% |
| No audit data | 15 | 0.3% |

`route_report` 自身健康：30 天调用 4192 次，成功 4019 / 失败 172（成功率 95.9%），**report_id 解析失败 0 例**——不存在"路由了却查不到报表名"。

### 1201 个 Not routed 的构成

| 实际情形 | 轮次 | 说明 |
|---|---|---|
| 同一会话前面某轮已路由成功，本轮是追问 | 1026 | 含改维度/加筛选/拆分/答澄清；其中 **686 轮执行了 SQL 且 success** |
| 整个会话一次都没路由 | 175 | 多为澄清续跑轮（文案 `User has answered your questions: …`）、纯对话反馈轮、知识问答轮（如 `what's the PCSD TCV definition?`） |

交叉维度：其中 219 轮的 `query_id` 与审计 `turn_id` 关联失败，且 **124 轮的同一会话在 ±5 分钟内确有 `route_report` 记录 = 假 Not routed**（占总轮次 2.5%）。分桶实测：b 最近轮已路由≤5min 124 / c 5-15min 12 / d >15min 4 / e 最近轮也无路由 79。

## 4. 可行性核查：revise_query 是否锁定同一报表

成立。`ssg-spark/app/ssg/smart_data_query/runtime/planned_state.py` 中 `report_id` 是 plan state 必填字段（`:87`），并有强制一致校验，任一不符直接抛错：

- `:112` `self.evidence_bundle.report_id != self.report_id`
- `:114` `prepared.snapshot.report.report_id != self.report_id`
- `:161` `execution.report_id != self.report_id`
- `:665` `current.report_id != pending.report_id`

即一次查询生命周期内报表被锁定，`revise_query`（`ssg-spark/app/ssg/smart_data_query/tools/revise_query.py:59` `class ReviseSmartDataQueryTool`）只能在同报表前提下改问题/查询。另见 `continuation.py:696,713`（续跑要求 `execution["report_id"]` 非空）。

### 生产验证：与"本会话最近一次路由"的一致性

在 1201 个 Not routed 轮中，取"本轮执行审计里的 report_id"对比"本会话此前最近一次 route_report 的路由结果"：

| 情形 | 轮次 |
|---|---|
| 本轮带 report_id，且与上一次路由**完全一致** | **731** |
| 本轮带 report_id，但与上次路由**不同** | 1 |
| 本轮带 report_id，但之前无任何路由 | 3 |
| 本轮完全没有 report_id | 480 |

**一致率 731/732 = 99.86%**，业务上回退几乎不会出错。但那 1 个例外即风险来源：30 天内 **126 个会话发生换报表、共 265 次路由**，"无脑回退"会把这类轮次标成错误报表。（注：口径与文档 `25fb29f1` 的 174 会话/5339 会话不同，后者为 2026-09~11 全窗。）

## 5. 推荐落法：读本轮 report_id 而非跨轮回退

`execute_compiled_query` / `execute_sql` / `resolve_plan` 的输出体本来就带 `report_id`：

```json
{"contract":"smart_data_query_result/v1","status":"success","message":"OK","report_id":"tcv", ...}
{"status":"ready","contract":"smart_data_query_composition_compilation/v1","run_id":"...","report_id":"simu", ...}
```

30 天实测：这三类工具共 14875 次调用，**12985 次（87.3%）输出带 report_id**；按状态分：succeeded 带 12984 / 不带 1622，failed 不带 267 / 带 1，started 未知 1。解析正则可复用现有 `ROUTE_REPORT_ID_EXPR`（`ChatbiEventQueryMapper.java:550-553`）。

| | 回退上一轮（同事方案） | 直接读本轮 report_id |
|---|---|---|
| 覆盖那 731 轮 | 覆盖 | 覆盖 |
| 会话换报表时准确性 | 会错（实测已有 1 例） | 天然正确 |
| 依赖跨轮/跨会话时间对齐 | 是（还要处理 upstream_session_id、事件时间早于审计时间） | 否 |
| 改动量 | 需窗口函数回退 | 加一个正则子查询 |

### 剩下的 480 轮，回退也补不上

| 构成 | 轮次 | 说明 |
|---|---|---|
| 无任何工具调用 | 234 | 纯对话、闲聊、知识问答 |
| `run_data_query` | 77 | 输出是 `Report scope requires routing` + 候选报表 options 列表，**绝不能当本轮报表** |
| 只 `finalize_query_result` | 44 | 仅可视化收尾，沿用上轮结果 |
| 只 `revise_query` / 只 `execute_sql` | 27 | 修订未执行 / 裸 SQL，本不属于任何报表 |
| 其余 | ~98 | ask_user_question、query_sdq_catalog、read_file、Skill、search_memory、tool_search、delegate_task |

补充：**执行失败轮基本不带 report_id**（268 个 failed 里仅 1 个带），失败轮要显示报表确实只能靠回退。

## 6. 建议的分层规则

```
1) 本轮 route_report 的 report_id                          → 报表名（现状）
2) 本轮 execute_compiled_query / execute_sql 的 report_id  → 报表名（新增，覆盖 731 轮）
3) 本轮 resolve_plan 的 report_id                          → 报表名
4) 本轮有执行/修订动作但以上都取不到 → 本会话此前最近一次 route_report，
   标 "Carried over"（↺ 或副标题），不冒充 routed
5) 其余 → "No report this turn"（不再混入 Not routed）
```

预期：Not routed 从 1201（24.4%）降到约 480（9.8%），其中 234 轮纯对话直接变 `No report`。列值形态变为 `SSG Bookings` / `SSG Bookings ↺` / `No report`。

若坚持只做回退，两个必须的前提：① 回退值必须与本轮执行审计的 report_id 交叉校验，不一致以本轮为准；② 回退只在本轮存在执行/修订动作时生效，纯闲聊轮不贴报表名。

## 7. 改动位置

- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java:564-622`：`SOURCE_LATERAL_SQL` 增本轮 report_id 解析；`ROUTE_STATE_SQL` 增 `carried` / `no_report` 两态；`SOURCE_LABEL_SQL`、`SOURCE_KEY_SQL`（下拉分组）同步。
- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java:1945-1951`：导出只认 `routed` / `route_failed`，需加新状态，否则 Excel 里新口径又被抹成 `—`。
- 前端来源下拉：`carried` / `no_report` 的英文文案（SPARK 为英文平台）。

## 8. 待确认

- 是否按第 6 节完整五层实现，还是先做 1→2 两步（改动最小，可消掉约 61% 的 Not routed）。
- 124~140 轮 `query_id` 与 agent `turn_id` 不一致的假阴性，是用时间窗兜底，还是让前端把 agent turn_id 一并上报。
