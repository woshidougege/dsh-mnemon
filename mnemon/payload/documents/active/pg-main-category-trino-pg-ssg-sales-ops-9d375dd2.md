---
id: "9d375dd2-6df1-40cd-9b51-fb85ed25333f"
title: "上游新增列落地到 PG：main_category 与 Trino→PG 三环境对齐（ssg-sales-ops）"
description: "bookings_simulation_ssg_kpi 新增 main_category 列的完整落地知识：加列必须「先改 source_sql 再 ALTER」的顺序规则与失败报错原文、三环境（生产 10.62.79.215 / uat .221 / tst .148）25 列对齐状态与备份文件、tst/uat 自 11-10 起两周未同步的教训、main_category 语义与「仅服务类 KPI 有值」限制、ssg_sync_task / ssg_sync_job 元数据表真实列名、psycopg 所在 python 解释器与本次速率实测。"
status: "active"
created_at: "2026-11-20T06:51:34.150Z"
updated_at: "2026-11-20T06:51:34.150Z"
content_hash: "74c2da72772cbdf97573acc7c81a05a5158aae06ccc59a617e0aa2dff07fb099"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/service/TrinoToPostgresSyncService.java"
  - "smartop-component/smartop-component-task-schedule/src/main/java/com/lenovo/component/schedule/service/ScheduledTaskService.java"
session_ids:
  - "mnemon-review-v1-c8a0859b-e321-439b-bad6-ca1e0126018c"
memory_body_ids:
  []
---

# 上游新增列落地到 PG：main_category（Trino→PG 三环境对齐）

ssg-sales-ops / bookings_simulation_ssg_kpi。记录一次"上游表加列 → PG 加列 → 重跑同步 → 三环境对齐"的完整落地过程，重点是可复用的**顺序规则**与排查中确认的元数据表真名。它是既有运维手册的增量补充，不替代它（见文末关联文档）。

## 1. 核心规则：加列必须"先改 source_sql，再 ALTER TABLE"，不能反

同步服务在写入前会校验「**目标表的每一列都必须能在源查询结果集中找到**」（`TrinoToPostgresSyncService.java:1231-1254`）。

- 先 `ALTER TABLE ... ADD COLUMN`、后改 SQL → 下一次调度立刻失败，报 `源查询缺少目标列: targetTable=bookings_simulation_ssg_kpi, column=main_category, sourceSql=SELECT * FROM prd_upsell.bookings_simulation_ssg_kpi`。
- 先改 SQL、后 ALTER → 不报错，多余列直接忽略，这一轮只是这列没写进去。
- 两者同一次窗口内一起改，是最省事的做法。

另一个方向的历史证据：上游**新增**列时，旧的 `SELECT *` 语句不会中断（源多出列无人校验）；**改列名/删列**才会中断——所以显式列清单是防"上游加列炸同步"的长期解，但防不住上游改名。

## 2. 三个环境的最终状态（本次已全部对齐）

| 环境 | 主机 | ssg_sync_task id | cron | 目标表列数 | 状态 |
|---|---|---|---|---|---|
| 生产 | 10.62.79.215 | 2 | `0 0 7 * * ?` | 24 → **25** | source_sql 25 列 + `main_category text` |
| pre/uat | 10.62.79.221 | 6 | `0 0 5 ? * WED` | 24 → **25** | 同上 |
| dev/tst | 10.62.79.148 | 2 | `0 0 10 ? * WED` | 24 → **25** | 同上 |

task id 跨环境重复（生产与 tst 都是 2），动手前必须用主机/库名确认环境。

生效的 source_sql（25 列，`main_category` 插在 `reporting_category` 之后，与源表列序一致）：

```sql
SELECT business_unit, quarter, week, geo, sub_geo, region AS market, country,
       customer_segment, customer_sub_segment_l1, customer_sub_segment_l2,
       customer_group, customer_group_l1, customer_group_l2, route_to_market,
       offering_bucket, oh_l1_hbb, oh_l2_offering, oh_l3_sub_offering, oh_l4,
       reporting_category, main_category, apos_flag, kpi, value, ingest_date
  FROM prd_upsell.bookings_simulation_ssg_kpi
```

DDL：`ALTER TABLE public.bookings_simulation_ssg_kpi ADD COLUMN main_category text;`（原表原地加列，秒级、无重写）。改前备份：`.tmp/sync_task_backup_prod_20261120.json`、`.tmp/sync_task_backup_10_62_79_221.json`、`.tmp/sync_task_backup_10_62_79_148.json`（含原 SQL 与表结构，回滚取用）。改完清 `retry_count=0`、`last_error=NULL`。

## 3. 顺带修掉的更严重问题：tst/uat 自 11-10 起完全没同步成功

改动前两环境都是 `SELECT *` + `retry_count=3` + "执行失败已重试 3 次仍未成功，放弃本轮等待下个 cron 周期"，数据冻结在 15,381,792 行、`ingest_date = 2026-11-10 03:09:51`。也就是说它们已经连续错过 11-18 那轮，只因缺 `market` 列。生产改 SQL 时（11-17）漏改这两个环境，代价是两周的测试/预发数据陈旧。

教训：改这类同步 SQL，**同一批次内把三环境一起改**，不要只改生产留尾巴。

## 4. main_category 是什么，以及它的硬限制

上游定义 `varchar(255)`（同事口述的 "main_catogry / text" 是笔误）；PG 按业务要求用 `text`，与其余 23 个 `varchar(255)` 邻列不一致，属已知取舍。

- 它是 `reporting_category` 的**父级归并**：MA Upsell→Upsell；TruScale / Deployment Services / Professional Services / Software Cloud Services→Support / Deployment；MA Post Warranty→Post Warranty；Configuration Services→Configuration Services；Sustainability / Managed Services / 部分 MA Upsell→MA Support；ISG Leasing→ISG Leasing；AI Solutions→AI Professional Services。
- 取值非空 875,777 行（约 5.6%），其余 14,674,867 行是空串。**只有服务类 KPI（Svcs Bookings 一套）带值，HW 类 KPI（HW Qty / HW Bookings / Premium Seats…）全空**——若业务用它筛 HW booking 会得到空结果，这一点必须提前讲清。
- 纯新增，无连带改动：`mv_bookings_agg`、`mv_bookings_week_full`、`v_bookings_week_full_l4`、`BookingsAgg` / `BookingsWeekAgg` 实体、`BookingsKpiCaseSql` 维度常量、接口与前端均不引用它。要让它出现在页面筛选里是另一个量级（两个 mv + Java + 前端）。

## 5. 排查时确认的元数据表真名（避免重复踩坑）

用 psycopg 查询调度元数据时反复撞列名错误，实测真名如下：

- `ssg_sync_task`：`cron_expr`（**不是** `cron_expression`）、`next_trigger_time`、`last_trigger_time`、`retry_count`、`retry_at`、`last_error`、`source_sql`、`target_table`、`write_mode`、`enabled`、`handler_key`、`payload`、`business_module`、`category`、`display_group`、`source_profile`。
- `ssg_sync_job`：`started_time` / `finished_time` / `heartbeat_time`（**不是** `start_time` / `end_time`）；行数字段是 `total_count` / `processed_count` / `copied_count`（**没有** `rows_copied`）；进度 `progress_percent`、`rows_per_second`；阶段 `count_phase_status`、`data_phase_status`、`materialized_refresh_status`、`redis_refresh_status` 及各自 `_error` 字段。
- 文本列无法直接 `max()` 比较时先 `\d` 或查 `information_schema.columns` 再写 SQL，比反复试错快。

## 6. 运行环境前提

本机 `psycopg` 只装在 `C:\Users\63588\AppData\Local\Programs\Python\Python312\python.exe`（psycopg 3.3.4）；DSH 内置 python（`...\dsh-primary-runtime\dependencies\python\python.exe`，3.12.14）与 `Python311` 都没有，直接调用会 `ModuleNotFoundError: No module named 'psycopg'`。JDBC 备用：`C:\Users\63588\.m2\repository\org\postgresql\postgresql\42.5.4\postgresql-42.5.4.jar`。

## 7. 本次实测数据（供下次估算）

- 源表 15,550,644 行，`ingest_date = 2026-11-20 10:52:07`；PG count 15,500,975 → 同步后应追平源。
- 数据阶段速率 10.4k–11.4k 行/秒，count 阶段完成后 15,550,644 行约 23 分钟；之后刷 mv + redis 约 5 分钟。
- 历史参考：11-17 那次 15,501,163 行耗时 26.5 分钟（~9,700 行/秒）；job 670 因缺 `market` 列失败（0 行写入、事务回滚保住了表）；job 706 失败原因是 `Job interrupted by application restart`，与字段无关。
- 三个环境共读同一张 Hive 表，全表扫描会互相抢 Trino 资源：**生产先跑完，再依次触发 tst、uat**，不要并发。

## 关联文档

- `00d44b32-34b4-480f-8812-e51ebf823d93`「Bookings 同步（Trino→PG）运维手册」：环境任务矩阵、TRUNCATE 无空结果保护的风险与触发前四道数据闸门、用 `next_trigger_time` 免登录触发、个人账号对 `ssg_sync_task` 只有 SELECT 的权限坑。本文第 2 节把该手册中"tst/uat 未改"的悬置项闭环了。
