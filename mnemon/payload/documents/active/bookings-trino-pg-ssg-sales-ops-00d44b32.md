---
id: "00d44b32-34b4-480f-8812-e51ebf823d93"
title: "Bookings 同步（Trino→PG）运维手册：环境任务矩阵、免登录触发、清表前数据闸门（ssg-sales-ops）"
description: "生产 ssg_sync_task 改 SQL 后落地的操作层知识：环境→任务 id 矩阵、已生效的 24 列 region AS market 语句与备份、TRUNCATE 无空结果保护的风险与触发前四道数据闸门、用 next_trigger_time 免登录触发调度管线的内部实现与行号、个人账号对 ssg_sync_task 只有 SELECT 的权限坑、tst/uat 未改与 daas_flag/deal_name 未决项。"
status: "active"
created_at: "2026-11-17T03:24:01.787Z"
updated_at: "2026-11-17T03:24:01.787Z"
content_hash: "65b8d95d4cfe8177a0545e10d72e73491ed0602d6b317e1399fc7158c195ad0e"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/service/TrinoToPostgresSyncService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/service/TrinoToPostgresScheduleService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/controller/TrinoToPostgresScheduleController.java"
  - "smartop-component/smartop-component-task-schedule/src/main/java/com/lenovo/component/schedule/ScheduledTaskScheduler.java"
  - "smartop-component/smartop-component-task-schedule/src/main/java/com/lenovo/component/schedule/service/ScheduledTaskService.java"
session_ids:
  - "mnemon-run-v1-369d15f1-b224-4e77-bc1e-02a4d502fe3e"
memory_body_ids:
  []
---

# Bookings 同步（LUDP Trino → PG）运维手册：环境矩阵、免登录触发、清表前数据闸门

2026-11-17 生产事故处理沉淀（仓库 `ssg-sales-ops`）。本文只记**可复用的操作层知识**：任务怎么找、怎么改、怎么在无登录态下触发、什么条件下绝不能触发。改名判定的证据与方案取舍见 `adc284dd-9ea9-4dc4-8ba4-82e6d2fa1648`（不要重复维护）。

## 1. 环境 → 调度任务 id 矩阵（以 2026-11-17 记录为准）

| 环境 | 库主机 | `ssg_sync_task` id | cron | source_sql 状态 |
|---|---|---|---|---|
| 生产 | （a_appconnect 可写） | 2 | `0 0 7 * * ?` | **已改为显式 24 列 + `region AS market`**，write_mode=TRUNCATE |
| dev/tst | 10.62.79.148 | 2 | 每周三 10:00 | 仍是 `SELECT *`，未改 |
| pre/uat | 10.62.79.221 | 6 | 每周三 05:00 | 仍是 `SELECT *`，未改 |

task id 跨环境会重复（生产与 tst 都是 2），改库前必须先用主机/库名确认环境，不能只看 id。

## 2. 生产已生效的改动（id=2）

```sql
-- 旧：SELECT * FROM prd_upsell.bookings_simulation_ssg_kpi
SELECT business_unit, quarter, week, geo, sub_geo, region AS market, country,
       customer_segment, customer_sub_segment_l1, customer_sub_segment_l2,
       customer_group, customer_group_l1, customer_group_l2, route_to_market,
       offering_bucket, oh_l1_hbb, oh_l2_offering, oh_l3_sub_offering, oh_l4,
       reporting_category, apos_flag, kpi, value, ingest_date
  FROM prd_upsell.bookings_simulation_ssg_kpi
```

- 回读确认：24 列，列名与目标表 24 列逐一对应，`region AS market` 生效；已用 Trino JDBC 直连实跑验证过。
- 同时清 `retry_count=0`、`last_error=NULL`，否则控制台会长期显示历史 3 次失败。
- 改前原 SQL 备份：`.tmp/sync_task_backup_prod_20261117.json`（回滚直接取该文件的值）。

## 3. 清表风险：改对 SQL 反而更危险

数据阶段顺序是「前置 hook → TRUNCATE → COPY」，且在**同一个事务**里：

- `smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/service/TrinoToPostgresSyncService.java:491-493`（`TRUNCATE` 分支，`WriteMode.TRUNCATE` 判定见 :179）
- `TrinoToPostgresSyncService.java:1197` → `st.execute("TRUNCATE TABLE " + quoteQualifiedName(targetTable))`
- `TrinoToPostgresSyncService.java:489-490` 注释说明 hook 与数据阶段同事务：TRUNCATE 之前执行，失败回滚后目标表原样保留

关键教训：列对不上时（旧语句 + 上游改名）异常发生在写入前，事务回滚**救了 1538 万行**；把 SQL 改对之后这道保险就没了——源表为 0 行时，TRUNCATE + COPY 0 行 + commit 会得到一个**状态仍为 SUCCESS 的空表**，两个物化视图也会被刷空。

因此触发前的强制闸门（本次实操）：
1. 源表 `COUNT(*)` 明显大于 0 且 **60 秒后复查稳定**（本次 15,501,163 → 仍 15,501,163）；
2. `ingest_date` 是当天（本次 2026-11-17 09:57:45）；
3. 维度基数与空值对得上历史（KPI 数 37；region 空值 10,500，与前一天 PG 里 market 空值完全一致）；
4. 确认无 running/pending 任务（本次触发前 running/pending = 0）。

另一个坑：源表 `COUNT=0` 不一定是上游坏了。本次在 09:57 后到约 11:19 之间是**上游重灌窗口**，表结构已是新 26 列、`ingest_date` 全空、COUNT=0；11:19 前后补写完立刻回到 1550 万行。遇到 0 行先等/轮询，别急着改代码。

体量参考：1550 万行按历史 10.5k~13k 行/秒约 **24~30 分钟**，之后刷 `mv_bookings_agg` + `mv_bookings_week_full` 约 5 分钟，再刷 redis 几秒。

## 4. 无登录态如何触发一次完整同步

`POST /data-integration/trino-to-postgres/task/{id}/run` 需要登录态（控制台「立即执行」走它，`TrinoToPostgresScheduleController.java:66-69`），只拿到 DB 权限时用调度器绕行：

- 把 `ssg_sync_task.next_trigger_time` 设为当前时间，等调度器下一轮扫描即可，走的是同一条管线（triggerType=AUTO）。
- 轮询器：`smartop-component/smartop-component-task-schedule/src/main/java/com/lenovo/component/schedule/ScheduledTaskScheduler.java:20-31`，`task-schedule.poll-interval-ms` 默认 **30000**、`initial-delay-ms` 默认 15000。
- 取任务：`.../service/ScheduledTaskService.java:200-209` `selectDueTasks`：`enabled=true AND (next_trigger_time <= now OR retry_at <= now)`；`triggerDueTasks` 循环见 :183-185。
- 执行完调度器会把 `next_trigger_time` 重算回下一个 cron 点；`backfillNextTriggerTime()`（:211-231）在启动后 20 秒补算 next_trigger_time 为空的任务。
- 大任务仍在跑时只顺延重检、不动 nextTriggerTime/retryAt（:313 注释，修复原死循环根因）。

改任务本身走 `PUT /data-integration/trino-to-postgres/task/{id}`（`TrinoToPostgresScheduleController.java:46-49`），可改字段：sourceSql / targetTable / writeMode / cronExpr / taskName / displayGroup。服务层 `updateTask` 见 `TrinoToPostgresScheduleService.java:65-83`，业务列用 `LambdaUpdateWrapper` 显式 set（`updateById` 会忽略 null 的 displayGroup）；hooks 传 null 表示保留原配置、传空数组才是清空（:113 `syncHooks`）。

## 5. 账号权限（踩过的坑）

个人账号对 `public.ssg_sync_task` **只有 SELECT**，更新会直接报：

```
psycopg.errors.InsufficientPrivilege: permission denied for table ssg_sync_task
```

写库必须用服务账号 `a_appconnect`；排查/取数用个人账号即可。

## 6. 未决项

- dev/tst（10.62.79.148 任务 2）与 pre/uat（10.62.79.221 任务 6）仍是 `SELECT *`：下次自动运行会以「缺少目标列 market」失败——安全失败、不丢数据，但补数时要先改 SQL。用户当时明确「先同步生产，其他的先不弄」。
- 源表新增 `daas_flag`、`deal_name` 约 5,880,718 行非空（38%）。本次为快速恢复只同步原 24 列，两列未落库；现有物化视图/视图/Java 代码均未引用，页面无影响。若业务要按 DaaS 口径或 deal 下钻，需加列后重跑一次（再约 25 分钟）。

## 7. 记录时状态（历史，勿当现状）

2026-11-17 11:22 触发后 job id=674 从 COUNTING 进入数据阶段；此前 662/666/670 均 FAILED（10:04 那批报「数据同步失败」）。
