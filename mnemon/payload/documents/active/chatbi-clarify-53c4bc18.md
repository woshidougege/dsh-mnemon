---
id: "53c4bc18-6900-4cfa-9b75-d36a82197291"
title: "ChatBI 问数记录「问题轮」口径契约与 clarify/补录轮处置决策（生产复核）"
description: "问数记录页数字自洽契约：单一口径问题集（SESSION_WHERE 排除 clarify_answer/no_answer/巡检/内部员工）、8 条验收恒等式（3/10=30%）、补录轮无 question 根因（ChatbiSessionBackfillService.java:157/178）、生产复核实测 518=493+25（95.2%）、四态来源桶求和 518、实施顺序与悬置项。"
status: "active"
created_at: "2026-11-13T02:50:47.661Z"
updated_at: "2026-11-13T02:50:47.661Z"
content_hash: "31e10307e0dc9610c40aeef42b0eb663c0aeddf748366e9e5c0866d10d851e50"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiSessionBackfillService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
session_ids:
  - "mnemon-run-v1-df00d10a-f41f-4826-8e13-a73b7a914d74"
memory_body_ids:
  []
---

# ChatBI 问数记录「问题轮」口径契约与 clarify / 补录轮处置决策

## 0. 背景与硬约束

- 页面：SSG ChatBI「问数记录 / 会话记录」（生产库 `10.62.79.215/ssg_ops_agent`）。
- 用户原始约束（2026-11-13）：
  - 「测试环境数据可能是坏的，你如果分析数据，得看生产的」——**所有口径结论必须由生产库复核，TST 只用于看 schema/结构**。
  - 「不能让领导一眼看出来数据对不上……统计的时候 3/10，成功率不是 30% 吗，别显示成 20%、40%」——页面任何分子/分母必须同源，可手算对上。
- 前置方案见文档 `25fb29f1-f532-44dc-b110-618cf4796fdc`（Source report 轮次级改造、四态 routed/route_failed/not_routed/no_audit、轮级 LATERAL、按报表名归并）。本文只补充**它标注"待定"的 clarify 轮口径**、页面数字自洽契约、补录轮根因与生产复核数字。

## 1. 判定：什么算「一个问题」（最终口径）

一条 `query_session`  событие只有同时满足才算一行问题，才进入任何分母：

```
event_name = 'query_session'
AND  question 非空（SESSION_QUESTION_SQL 的 COALESCE 链）
AND  不是 clarify 答复轮（question 以 "User has answered your questions:" 开头）
AND  不是未回答轮（"The user did not answer the pending ask_user_question…"）
AND  不是巡检（properties->>'trigger' != 'probe' 且 NOT EXISTS ssg_ai_probe_run）
AND  按需排除内部员工（ssg_chatbi_internal_user.enabled）
AND  SESSION_QUESTION_SQL != ''
```

- 实现方式：新增 `TURN_TYPE_SQL`（question / clarify_answer / no_answer）与 `QUESTION_ONLY_FILTER`，**挂在 `ChatbiEventQueryMapper.SESSION_WHERE`（约 :778-800）**，明细、总数、汇总、分组、来源报表、导出、反馈页自动共用同一 WHERE。
- **决策：不做"置灰仍显示"**。曾经推荐过 B 方案（打标 `turnType=clarify_answer`、明细显示但置灰、统计排除），已放弃——表格 10 行、卡片 7 条就是用户说的"一眼看出对不上"。改为 A 方案：这三类系统轮**既不出现在明细，也不出现在总数**。
- 前提纠正（用户口头）：`simu` 是后来改名成 `Services_Bookings`，同一个报表而非两个同名报表；`report_id` 一律不上屏。

## 2. 数字自洽契约（验收恒等式）

| 页面数字 | 算法 | 必须成立 |
|---|---|---|
| Question details 的 N records | `count(*)`，与明细同一 WHERE | 翻页前后不变 |
| 会话 `n/N` 编号 | 同一结果集上的窗口函数 | 不会出现 `1/4` 与 `4/4` 跨口径 |
| Final result 的 Success / Failed | 同一次 `count(*) filter` | Success + Failed = total |
| 成功率 | Success / total | 3/10 必然显示 30.0% |
| 报表来源卡片 | 每个问题行只落一个互斥桶（四态枚举） | Σ各报表 + Not routed + No audit data = total |
| 每个来源的成功率 | 该来源 Success / 该来源 total | 与该行自身数字自洽 |
| 导出 Excel | 与页面同一 SELECT | 屏幕数字 = 文件数字 |

三条硬性禁止：① 不用 `failed/(total-failed)` 之类换分母；② 不用 `statusCounts`（步骤分类）反推成功率（代码注释已有提醒）；③ 不把已排除的系统轮算进任何分母。

## 3. 根因：为什么会出现"假问题轮"

生产 `ssg_chatbi_event` 共 8605 条 `query_session`，只有一部分是用户真打字：

| 类型 | 条数 | 页面可见 | 实际是什么 |
|---|---|---|---|
| clarify 答复轮 | 452 | 是 | agent 用 `ask_user_question` 追问，用户点选项被上报成一条新事件，question 是 `User has answered your questions: - Which result …: PCSD Pr…` |
| 未回答轮 | 90 | 是 | `The user did not answer the pending ask_user_question…`（超时/跳过） |
| 补录轮（无 question） | 5556 | 3341（另 2215 为巡检 probe，页面已过滤） | `ChatbiSessionBackfillService.buildEvent` 只写 `session_title`，**不写 question**，页面回退显示会话标题 |

- 补录根因位置：`smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiSessionBackfillService.java:157`（`props.put("session_title", …)`）与 `:178`（`props.put("source", "backfill")`）——事件里根本没有 question 字段。其中 105 条连标题都没有，Question 列直接空白。
- 多轮会话里每行都显示首问那句，造成"数据看起来重复"。
- 影响：问题数被夸大（Question count、成功率分母、会话的 `1/4 → 4/4` 都按埋点行数）；"未路由"虚高（1540 条里 456 条是 clarify 轮，本不该路由）。

## 4. 生产复核（最终口径，2026-11-13）

默认视图（排除巡检 + 内部员工）：

```
问题行 518 = 成功 493 + 失败 25             ✔ 成功率 95.2%
轮型分布 question 518 / clarify_answer 59 / no_answer 14
来源桶求和 518 = 518                        ✔
Not routed 291(273 会话) | Services Bookings 117 | TCV 64 | No audit data 14
Daas Bookings 12 | Opportunity 7 | ISG PR 5 | Offering Hierarchy 3 | PN Master 2
route_failed: Services Bookings 2(0 成功) | Daas Bookings 1(0 成功)
```

- 同时解决的另一处错配：默认视图 **518 行中 174 行（33.6%）显示的报表本轮根本没用到**；含内部员工时 5503 行中 1150 行（20.9%）。
- 内部员工口径差异是我不用 TST 的又一理由：`ssg_chatbi_internal_user` 生产 enabled=62 人 / 命中 8014 条 query_session，TST 仅 4 人 / 4202 条。

## 5. 关键 SQL 语义（复用而非重写）

- `FINAL_SUCCESS_SQL`（`ChatbiEventQueryMapper`）: `status='success'` **或** 前端确实渲染了 Smart Data Query 结果卡（`SMART_DATA_DISPLAY_SQL` 按 `contract=smart_data_query_result/v1` + `status=success` 判定 `execute_compiled_query`/`execute_sql`/`finalize_query_result`）。所以 status=success 无卡（知识问答/PBI 推荐）仍算成功，status=failed 但有卡（finalize 失败）也算成功。
- `LUDP_PERMISSION_SQL`：`reason=ludp_permission_denied` 的轮次在事件里常记 success，必须回查审计，否则识别不出权限失败。
- `PROBE_RUN_EXCLUDED_SQL`（约 :462）：以 `ssg_ai_probe_run.session_id` 为准；2026-10 后新巡检链路不再写 `trigger`，只靠属性会把巡检导出成业务数据，保留 trigger 判断兼容历史。
- `RECOVERED_SQL`：不能只看 steps 数组 `failed>0 && success>0`（按工具聚合、无顺序，"先成功两次最后失败"生产已有反例），必须存在失败行之后同一 (session_id, turn_id, tool_name) 的 succeeded 行。
- `AUDIT_SESSION_ID_SQL`：`COALESCE(upstream_session_id, session_id)`，AskPreview 另有 Spark 真实会话 ID。

## 6. 实施顺序（用户已同意按此推进）

1. **mapper**：新增 `TURN_TYPE_SQL` + `QUESTION_ONLY_FILTER` 挂进 `SESSION_WHERE`；`SOURCE_LATERAL_SQL` 由"会话最早一条 route_report"改为"按 `query_id` 取本轮最后一次成功路由"；`SOURCE_LABEL_SQL`/`SOURCE_KEY_SQL` 按报表名归并；`started_at` 需 `CAST(... AS timestamptz)`（TST 是 text、生产是 timestamptz）。
2. **service**：删掉 `enrichSessionDiagnostics` 里 Java 侧重复的会话级映射（保留状态分类/successText）。
3. **前端**：Report 列按四态渲染（正常 / route failed 灰红点 / Not routed 灰 / No audit data 浅灰），`n/N` 常显，筛选下拉按名字去重；`includeInternal` 已贯通（`components/FilterBar.vue:346,576,849-850`、`index.vue:87,148,276`、`context.ts:26`、`components/modals/ExportModal.vue:124`）。
4. **backfill**：`ChatbiSessionBackfillService` 按 `query_id` 再取本轮第一条 user 消息写入 `question`，3341 行 Question 列恢复原文；与页面改造分开提交。
5. **验收**：生产只读库跑三条恒等式 + 两个真实会话（`9a4e5d1f-b262-495e-8b6d-87647b37f043` 三报表、`f0340d5e-a0c6-4f67-8252-dd4134ce09bd` 20 轮），再动前端。

## 7. 悬而未决

- backfill 补写 question 是否回填历史 5556 行（或只对新数据生效）——未定，属独立提交。
- 报表重名（`simu`/`Services_Bookings` 改名残留）按名归并后，`Services Bookings hw_sub_series_description` 是否也并入同一桶——按现有四态先按 name 归并，细节待前端联调确认。
