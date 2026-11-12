---
id: "4568440b-b6c8-47bd-a416-bce10601a830"
title: "生产→TST 数据同步：三个导出按钮依赖的 8 张表与踩坑（ssg-sales-ops）"
description: "把反馈中心/点踩/User Logs/巡检报告导出所需的生产数据灌入 TST：依赖表清单与行数基线、replace vs 按 id 合并策略、生成列/序列/时间列等 6 个坑、回滚备份、附件文件缺失与 jira 开关未开的未决事项。"
status: "active"
created_at: "2026-11-12T10:53:23.644Z"
updated_at: "2026-11-12T10:53:23.644Z"
content_hash: "8bebf207094d8951fc1370436fb852f9134b4744b631f33c27ee97ebe1954f6d"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-bd4ab953-fec1-4b9a-80bf-4cc277a76735"
memory_body_ids:
  []
---

# 生产 → TST 数据同步：三个导出按钮依赖的表（ssg-sales-ops）

来源：为验证「反馈中心 / User Logs / 巡检报告」三个导出按钮的修复（Jira ticket 列可点、导出带 Feedback details），把生产相关表灌进测试库 TST。2026-11-12 执行完成，行数与生产逐表对齐。本文件记录**可复用的同步口径、踩坑与未决事项**，不含任何库口令。

## 1. 结论

- 只搬「3 个导出按钮」各自依赖的 8 张业务表 + 2 张 agent 会话表，**不动 TST 其它数据**。
- 业务表用「整表替换」（DELETE + COPY IN），会话/消息用「只增不删」的按 id 合并 —— 以免清掉测试环境同事自己的会话。
- 同步后 TST 8 张业务表行数与生产完全一致，数据面已具备验证条件；功能开关与 MR 合并仍需用户拍板。

## 2. 依赖表清单（同步后 2026-11-12 基线）

| 表 | 用途 | 同步方式 | TST = PROD |
|---|---|---|---|
| `public.ssg_usage_issue_feedback` | 反馈中心提交的问题 | replace | 20 |
| `public.ssg_usage_issue_feedback_attachment` | 问题附件 | replace | 18 |
| `agent.result_feedback` | 点踩反馈（"结果反馈"） | replace | 220 |
| `public.ssg_feedback_jira_issue` | 点踩 ↔ Jira 单映射 | replace | 122 |
| `public.ssg_jira_issue` | Jira 单状态缓存 | replace | 134 |
| `public.ssg_jira_comment` | Jira 评论 | replace | 251 |
| `public.ssg_ai_probe_run` | 巡检报告 | replace | 4063 |
| `public.ssg_chatbi_event` | User Logs 事件 | replace | 78 829 |

会话/消息按 `agent.result_feedback.session_id` 反查生产范围（159 个会话 / 8321 条消息），按 id `WHERE NOT EXISTS` 合并进 TST；因此 `agent.sessions` TST 1 884 vs 生产 6 856、`agent.messages` TST 32 080 vs 生产 163 944 的差异是**刻意保留**的，不是漏同步。

## 3. 同步脚本与策略

脚本：本地 worktree `C:\Users\63588\lenovo\ssg-sales-ops-wt-jiralink` 下 `.tmp/SyncData.java`（配套 `.tmp/ExportSanity.java` / `ExportSanity2.java` 做 TST vs PROD 计数比对）。**脚本内硬编码了生产/测试库连接串与口令，仅存活于本地 .tmp，禁止提交**。

- 阶段参数：`backup` / 全量（默认）/ `load`（跳过 dump 只灌数，dump 产物已在 `.tmp/syncdump`）。
- `dump()`：`COPY (SELECT <cols> FROM t) TO STDOUT WITH (FORMAT csv)`；列名先由 `cols(conn, table)` 从 `information_schema` 取，避免手写列清单漂移。
- `replace()`（`SyncData.java:60-62`）：单事务内 `DELETE FROM` 目标表 → `copyIn()`（`SyncData.java:48`）→ `seqFix()`。
- `mergeById()`（`SyncData.java:80-84`）：建 stage 表灌 CSV，再 `INSERT INTO t (...) SELECT ... FROM stage s WHERE NOT EXISTS (SELECT 1 FROM t WHERE t.id = s.id)`，最后 `DROP TABLE stage`。
- `seqFix()`（`SyncData.java:93-108`）：`select pg_get_serial_sequence('<table>','id')` → `setval(seq, GREATEST(COALESCE((SELECT max(id) FROM t),1),1))`，整体包在 try/catch 里，失败只打印 `seq skip`。

## 4. 踩坑清单（复用价值最高）

1. **生成列不能出现在 COPY 里**：`public.ssg_chatbi_event` 有 7 个生成列 —— `question`、`session_title`、`query_id`、`model`、`status`、`execution_mode`、`profile_name`。带上就报 `ERROR: column "question" is a generated column / Generated columns cannot be used in COPY.`。做法：从列清单里过滤生成列（`is_generated = 'ALWAYS'`），实际只 COPY 13 列。
2. **没有 `id` 列的表调 `pg_get_serial_sequence` 直接抛错**：`ssg_feedback_jira_issue`、`ssg_jira_issue`、`ssg_jira_comment` 均无 `id`，报 `ERROR: column "id" of relation "..." does not exist`。`seqFix` 必须容错跳过（首版没 catch，事务回滚导致半途中断）。
3. **序列默认值 vs identity**：`ssg_ai_probe_run.id` 与 `ssg_chatbi_event.id` 是 `default nextval(...)`（`ssg_ai_probe_run_id_seq`、`chatbi_event_id_seq`），`identity=NO` —— 灌完必须显式 `setval`，否则应用侧插入撞主键。
4. **`ssg_ai_probe_run` 的时间列叫 `started_at`**，不是 `created_at`；该表另有 `created_time`/`updated_time`（`result_hash`、`expected_sql`、`sql_verdict`、`sql_similarity`、`compare_note`、`jira_key`、`batch_id`、`agent_steps` 等列）。按 `created_at` 写 sanity 查询会报 `column "created_at" does not exist`。
5. **数据量导致超时**：整表 dump 出 `messages.csv` 296 MB、`probe_run.csv` 688 MB、`chatbi_event.csv` 36 MB。前台 `run_code` 120 s 必然超时，需作为后台 job + `-Xmx2g` 跑；把会话/消息按生产反馈涉及的 session_id 收窄后降到 102 KB / 8 KB 量级。
6. **临时 Java 工具编译**：需 `-encoding UTF-8`（源文件含中文）且 classpath 带 PG 驱动 `C:\Users\63588\.m2\repository\org\postgresql\postgresql\42.5.4\postgresql-42.5.4.jar`；`JAVA_HOME` 用 `C:\Program Files\Java\jdk-17.0.19`。缺驱动时的报错是 `ClassNotFoundException: SyncData`/`CopyOut 无法转换为 InputStream`，容易误判成代码问题。

## 5. 回滚保险

清表前把 TST 原数据整表 dump 到 `C:\Users\63588\lenovo\ssg-sales-ops-wt-jiralink\.tmp\syncdump\tst-backup\`：

`public_ssg_usage_issue_feedback.csv`、`..._attachment.csv`、`public_ssg_feedback_jira_issue.csv`（7 字节即空表）、`public_ssg_jira_issue.csv`、`public_ssg_jira_comment.csv`、`agent_result_feedback.csv`、`public_ssg_ai_probe_run.csv`、`public_ssg_chatbi_event.csv`。需要复原直接 `COPY ... FROM` 回对应表即可。

## 6. 同步后校验基线（2026-11-12）

- 反馈中心：20 条提交问题**全部**带 Jira 单号（`has_jira=t rows=20`）；点踩 220 条中 down=115，全部带 session_id；115 条点踩里 114 条能对上 Jira 单。
- Jira 状态缓存分布：Done 107 / Testing 9 / In Progress 7 / New 6 / Pending 3 / Ready for Testing 1 / Grooming 1（不再是清一色 "No ticket yet"）。
- User Logs：`ssg_chatbi_event` 78 829 行，覆盖 2026-09-11 17:38 ~ 2026-11-12 18:43，7 915 个会话；导出即带 `Feedback details` 表。
- 巡检报告：`ssg_ai_probe_run` 4 063 次运行，2026-09-24 ~ 2026-11-12。

## 7. 已知限制与未决事项

- **附件只有行、没有文件**：`ssg_usage_issue_feedback_attachment` 搬的是记录，文件仍在生产对象存储，TST 点预览会 404。
- **TST 功能开关仍关**：`ssg_probe_setting` 的 `jira.flow.usage-feedback.enabled`、`jira.flow.dislike.enabled` 在测试环境仍是 `false`（生产 `true`），所以 TST 上新提交反馈/点踩**不会自动建单**。是否改成 true 需用户拍板（属手动改库，不走 flyway）。
- **MR !521 仍 opened**：修的是「Jira ticket 列可点 + Feedback details 默认带上」。数据已就绪，合并后可直接在 TST 验证这两条。
- TST 应用同步期间在跑，`ssg_chatbi_event` 会被测试流量继续追加（本次清掉 92 456 行老数据后灌入生产 78 829 行，之后新增属正常）。

相关：同仓库跨环境 `ssg_sync_task` 任务配置同步（cron/APPEND/列权限）见文档 `d6ae03bd-2e22-43c2-8884-a4cda6b516f3`。
