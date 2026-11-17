---
id: "adc284dd-9ea9-4dc4-8ba4-82e6d2fa1648"
title: "上游 market→region 改名：Bookings 同步影响分析与最小改法（ssg-sales-ops）"
description: "上游 bookings_simulation_ssg 把 market 改名 region 后的影响判定与方案：纯改名证据（21 取值 100% 对应）、只改 source_sql 用 region AS market 的理由、TRUNCATE 无空结果保护导致「改对了反而清表」的风险与执行顺序、跟上游改名的完整代价清单、daas_flag/deal_name 与同批 Access Denied 待确认项。"
status: "active"
created_at: "2026-11-17T02:52:48.557Z"
updated_at: "2026-11-17T02:52:48.557Z"
content_hash: "90b4ad003a032563cc39708f3d2dd315bff6224dff0c7d8e7304bab2301700df"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/bookings/BookingsKpiCaseSql.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/bookings/constants/BookingsLowerPaneBreakdown.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/bookings/constants/BookingsFilterOptionSortOrder.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/bookings/service/BookingsService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/service/TrinoToPostgresSyncService.java"
session_ids:
  - "mnemon-run-v1-7c057332-98d4-42fa-976e-783746b6851d"
memory_body_ids:
  []
---

# 上游 `market` → `region` 改名：Bookings 同步链路影响分析与最小改法（ssg-sales-ops）

2026-11-17 排查。触发点：上游 KPI 明细表把 `market` 列改名为 `region`，导致 `ssg_sync_task` id=2 的 Bookings 同步每晚失败。本文记录「是不是必须跟着改名」的判定证据、选定方案与 rationale、以及在空结果场景下的真实风险，可复用于同类上游 schema 变更。

## 1. 结论先行

**是纯列改名，值域完全一样**，因此不必改我们的表结构和对外接口。选定方案：**只改同步任务的 `source_sql`**，用显式列清单 + `region AS market` 别名，把上游改名挡在同步层。

## 2. 证据：纯改名判定

明细表 `prd_upsell.bookings_simulation_ssg` **同时存在** `market` 与 `region` 两列（最新分区约 9.5 万行），两边各 21 个取值，`market × region` 配对 **100% 一一对应**：

| 值 | 行数 | 值 | 行数 |
|---|---|---|---|
| NA | 29,537 | CEE-CA | 1,516 |
| DACH | 15,884 | ANZ | 1,474 |
| JAPAN | 11,246 | NORDICS | 1,007 |
| CAP | 8,065 | MEA | 530 |
| UKI | 4,421 | KSA | 331 |
| INDIA / IBERIA / BENELUX / ITALY-ISRAEL / FRANCE | 2,428 / 2,249 / 1,938 / 1,811 / 1,527 | LAS / MX / TURKIYE / BRAZIL / NEC / 空 | 312 / 247 / 118 / 84 / 24 / 18 |

结论：改名不改变数据，只改变列名。

## 3. 目标侧现状（不可轻易动）

- 目标表 `bookings_simulation_ssg_kpi` 现有 24 列：
  `apos_flag, business_unit, country, customer_group, customer_group_l1, customer_group_l2, customer_segment, customer_sub_segment_l1, customer_sub_segment_l2, geo, ingest_date, kpi, market, offering_bucket, oh_l1_hbb, oh_l2_offering, oh_l3_sub_offering, oh_l4, quarter, reporting_category, route_to_market, sub_geo, value, week`；目标表约 **1,538 万行**。
- `mv_bookings_week_full` 直接引用 `market` 列：
  `SELECT quarter, week, geo, business_unit, customer_sub_segment_l1, customer_group_l1, oh_l1_hbb, route_to_market, market, oh_l3_sub_offering, kpi, sum(value) AS total_value FROM bookings_simulation_ssg_kpi WHERE TRIM(BOTH FROM COALESCE(week, '')) <> '' GROUP BY ...`
- `market` 在我们这边是**对外维度约定**，不只是一个列名：
  - `smartop-service/admin/src/main/java/com/lenovo/admin/bookings/BookingsKpiCaseSql.java:121` → `public static final String MKT = "market";`（第 209 行 `coalesce(market,'') <> 'NEC'` 用它做 NEC 过滤）
  - `smartop-service/admin/src/main/java/com/lenovo/admin/bookings/service/BookingsService.java:3765` → 同一 NEC 过滤
  - `smartop-service/admin/src/main/java/com/lenovo/admin/bookings/constants/BookingsLowerPaneBreakdown.java:9` → `MARKET("market")`，第 35 行 `case MARKET -> "market"`（下半区图表/明细表行维度）
  - `BookingsFilterOptionSortOrder.java:17/82` → 筛选项 market ↔ 列 `market`
  - `BookingsAgg.java:25`、`BookingsWeekAgg.java:26` 实体的 `private String market;`；`BookingsLowerPaneQuery`/`BookingsKpiCardsQuery` 等的 `@Schema(description = "Route to market")`
  - 前端 Tab 叫 Market；XML（Mapper）侧 0 处 market 引用。
  - 全仓 bookings 包内 market 相关命中 16 处。

## 4. 选定方案：只改 source_sql（不改 PG 结构、不改代码、不改前端）

把 `ssg_sync_task` id=2 的 `source_sql` 从 `SELECT * FROM prd_upsell.bookings_simulation_ssg_kpi` 换成显式列清单 + 别名，24 个表达式正好对应目标表 24 列：

```sql
SELECT business_unit, quarter, week, geo, sub_geo,
       region AS market,
       country, customer_segment, customer_sub_segment_l1, customer_sub_segment_l2,
       customer_group, customer_group_l1, customer_group_l2, route_to_market,
       offering_bucket, oh_l1_hbb, oh_l2_offering, oh_l3_sub_offering, oh_l4,
       reporting_category, apos_flag, kpi, value, ingest_date
FROM prd_upsell.bookings_simulation_ssg_kpi
```

**理由**：改表结构会连前端接口一起动（接口参数/返回字段都是 `market`），收益只是「和上游同名」；而显式列清单顺手解决真正的根因——`select *` 只要上游加列或改名就立刻失败，显式清单后上游再新增 `daas_flag`、`deal_name` 之类列都不影响我们。

**如果坚持跟上游同名，代价**（当时判断不值）：

| 改动 | 量 |
|---|---|
| `ALTER TABLE bookings_simulation_ssg_kpi RENAME COLUMN market TO region` + 加 `daas_flag`/`deal_name` | 1 条 DDL |
| `mv_bookings_agg`、`mv_bookings_week_full`、`v_bookings_week_full_l4` | 4 个对象要重建/改写；前两个物化视图改完要重刷（74 万行 / 412 万行，需跑几分钟），且**物化视图输出列名不会跟着基表改名走，必须显式对齐** |
| 后端 | `Cols.MKT`、`BookingsLowerPaneBreakdown.MARKET.sqlColumn`、`BookingsAgg.market`、`BookingsWeekAgg.market`、`BookingsService.java:3765` 的 NEC 过滤，共 5 处 |
| 前端 | 接口字段要保持 `market` 就得在实体上做映射，否则要改前端 |

## 5. 关键坑：TRUNCATE 模式没有空结果保护（改 SQL 前必须确认源表有数）

`smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/service/TrinoToPostgresSyncService.java`：
- `:488` `targetConn.setAutoCommit(false)`；`:490` 在数据事务内跑 PRE hook；`:491-493` TRUNCATE 目标表。
- `:497-506` 用 `ResultSetCsvReader` + `PGConnection.getCopyAPI().copyIn(...)` 写入；`:500` 在数据阶段才校验列映射 `buildSourceColumnIndexes(...)`。
- `:508` commit，`:510-514` 异常分支 `rollbackQuietly` + `failDataPhase` 后再抛。

推论（本次最重要的风险认知）：
- **当前失败反而是安全的**：列映射校验失败发生在 commit 之前，TRUNCATE 被 `:510-514` 回滚，1538 万行原样还在。这就是今天报错没有把表清空的原因。
- **一旦 `source_sql` 改对而源表仍为 0 行**：TRUNCATE + COPY 0 行会一起提交，任务标记 SUCCESS，目标表被清空，`mv_bookings_agg` / `mv_bookings_week_full` 跟着刷成空表，页面直接没有数据。
- 因此执行顺序必须是：**先找上游确认 KPI 表何时有数 → 有数后再改 `source_sql`**；未确认前把该任务 `enabled` 关掉，避免每天 07:00 触发 3 次重试。
- 建议的加固项（待办）：`totalCount = 0 && TRUNCATE` 时跳过写入并告警。参考已有的 PRE hook 机制（`runPreHooks` `::520-529`，ABORT 失败即回滚整个事务含 hook 改动）。

## 6. 待上游确认（与本次同批次的连带问题）

1. `daas_flag`、`deal_name` 两列**要不要落库**？显式列清单会跳过它们；若业务要用（DaaS 口径拆分、按 deal 下钻），需 ADD COLUMN 并补进清单——先定再落库。
2. 同日另外 3 个同步任务失败均为 `Access Denied: Cannot select from columns [...]`（`lgfs_lease_asset_summary`、`lgfs_lease_billing_summary`、`gps_pnpl_master_data`），疑似同一批权限或结构调整；那批里也有 `segment`、`sub_segment`、`deal_type` 等列，可能是同一次变更的连带后果，建议合并成一个窗口一起问。
3. 取数账号是 `p49_g1189`（Trino/Presto），按表申请列权限要指名该账号，不是个人账号——见关联文档。

## 7. 相关环境事实

- `ssg_sync_task` 的 PRE hook 均为「reload 前删除当前/上一季度」，例如 task 26 → `delete from gps_sus_kpi_ttl where fiscal_quarter>='FY2627Q3'`；simu/tcv/bookings 系列 hook 目标是 `bookings_simu_customer_week`、`tcv_customer_quarter`、`bookings_simu_deal_week`，`timeout_seconds` 300、`on_error=ABORT`。
- Bookings 三项生产 cron 为 `0 0 7 * * ?`（07:00），其余 `0 30 7 * * ?`（07:30）。

## 8. 关联文档

- `d6ae03bd-2e22-43c2-8884-a4cda6b516f3`（SSG 同步任务跨环境同步：LGFS 逻辑合并、cron 统一与 p49_g1189 列权限阻塞）—— 权限申请与分组/cron 约定以此为准。
