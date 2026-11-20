---
id: "49ad133b-c29d-4a95-b655-cf830cbdb363"
title: "上游 KPI 表 region 整列 NULL：bookings market 塌缩的识别、影响与复核（ssg-sales-ops）"
description: "bookings_simulation_ssg_kpi 的 region 整列未赋值事故：同步 SUCCESS + 行数全绿但 market 全 NULL 的校验缺口、明细表对照证据、mv_bookings_agg 745,611→233,161 与 mv_bookings_week_full 4,152,837→1,554,313 的塌缩量化、单一快照无法回退与 (geo,sub_geo,country) 反推不可行（197 组合中 19 个一对多）、原地补值的复核判据、修复后 market 21→38 值域变化（ITALY-ISRAEL→ITALY/ISRAEL、大小写并存）及无硬编码枚举的确认、三环境串行节奏与任务阶段字段。"
status: "active"
created_at: "2026-11-20T08:16:46.841Z"
updated_at: "2026-11-20T08:16:46.841Z"
content_hash: "669317bc14d01115850f0903c00d6ef70436e62f33a753dd5ada3d5abf0bc450"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/service/TrinoToPostgresSyncService.java"
session_ids:
  - "mnemon-review-v1-6e97a1c6-c864-4e66-b6de-2573f16b65b6"
memory_body_ids:
  []
---

# 上游 KPI 表 region 整列 NULL：bookings market 塌缩的识别、影响与复核

ssg-sales-ops / `bookings_simulation_ssg_kpi`。记录一次"**同步任务全程 SUCCESS、行数校验全绿，但数据实际是坏的**"的事故：上游 KPI 聚合表 `region` 整列未赋值，导致 PG 侧 `market` 全 NULL、物化视图塌缩。重点是可复用的**校验缺口**、影响面量化方法与值域变化。落地流程与列规则见既有文档：`9d375dd2-6df1-40cd-9b51-fb85ed25333f`（main_category 加列落地）、`00d44b32-34b4-480f-8812-e51ebf823d93`（Trino→PG 同步运维手册）。

## 1. 核心教训：行数校验通过 ≠ 数据正确

该轮同步（生产 job 748）在 `status=SUCCESS`、`total = copied = 15,550,644`、`ingest_date` 与源表一致、`main_category` 分布与源表逐项一致的情况下，`market` 仍可能整列为空。只有在**同步后按业务维度列做 top-N 分布比对**（如 `GROUP BY market` 的固定几条）才暴露：分布只剩 `('<null>', 15550644)`。

→ 校验清单里"行数 + ingest_date + 新增列分布"不足够，必须再加一条**既有维度列的分布 top-N 抽查**。

## 2. 归因证据链（证明不在同步侧）

| 对象 | 观测值 |
|---|---|
| `prd_upsell.bookings_simulation_ssg_kpi` 总行数 | 15,550,644 |
| 其中 `region IS NULL` | **15,550,644（100%）** |
| `ingest_date` 快照数 | 仅 1 个（`2026-11-20 10:52:07.61`，无历史版本） |
| 明细表 `prd_upsell.bookings_simulation_ssg` 的 `region` | 正常：NA 39,102,514｜DACH 10,120,109｜NORDICS 5,463,294｜BRAZIL 4,925,910｜JAPAN 4,744,360… |

结论：同一时刻明细表 region 完好，只有 KPI 聚合表整列为空 → 上游在生成 KPI 表这一步漏赋值，与本侧 `region AS market` 的 SQL 无关。

## 3. 影响面量化（必须用物化视图行数对比，而不是只看主表）

| 对象 | 坏数据 | 正常数据 | 变化 |
|---|---|---|---|
| `mv_bookings_agg`（market 维度聚合） | **233,161** | 745,611 → 修复后 745,595 | -69% |
| `mv_bookings_week_full` | **1,554,313** | 4,152,837 → 修复后 4,156,390 | -63% |
| `v_bookings_week_full_l4`（明细粒度） | 15,266,964 | 15,217,483 | 基本不变 |

即：**明细粒度视图不受影响，凡是按 market 聚合/筛选的展示（Market Tab、market 拆分 KPI 卡片）全部错空**。物化视图行数是这类"维度塌缩"最快的判据——若 MV 行数骤降 60%+ 而主表行数正常，先查维度列而不是查同步。

## 4. 无法回退，也无法反推

- 源表只有单一 `ingest_date` 快照，PG 侧无历史备份 → 坏数据覆盖后**不能回滚**，只能等上游原地补值后重跑（32–35 分钟一轮）。
- 手动触发只是把问题提前半天暴露：即使不手动跑，次日 07:00 cron 也会拉到同一批坏数据。
- **不能用`(geo, sub_geo, country)` 反推 region**：197 个 distinct 组合中有 19 个组合对应多个 region（KR 同时属 KOREA/CAP，CZ、RO、TW、HK 亦有一对多；空 country / null country 组合各自对应多达 11 个 region）。硬映射会引入静默错误，不采用。

## 5. 修复后的复核方法

上游是**原地补值**而非重灌——判据：行数 15,550,644 与 `ingest_date` 均未变。复核结果：

- `region IS NULL`：15,550,644 → **8,214**；空串 `''`：**2,286**；合计 10,500，与 11-17 历史版本同一量级，属本来就有的正常缺失。
- 完整重跑 job 749（15:35:36 → 16:10:34 SUCCESS）后：行数一致、`market` 38 个值与源表 `region` **逐项一致**、`mv_bookings_agg` 745,595、`mv_bookings_week_full` 4,156,390。

## 6. 修复引入的值域变化（需向上游反馈）

`market` 取值个数 **21 → 38**：

- 新增：`EUROPE HQ` 42,876｜`AP HQ` 39,081｜`META HQ` 33,922｜`EAST EUROPE` 25,954｜`GTAPT` 25,922，以及零碎 `LI HQ` 306｜`FCCL` 270｜`LA HQ` 142｜`CZ/SK` 32｜`SEE` 12｜`BG HQ` 10｜`LAS HQ` 8｜`CENTRAL`。
- **值本体改名**：`ITALY-ISRAEL` → `ITALY/ISRAEL`（连字符变斜杠）。
- **大小写并存**：`EAST EUROPE`(25,954) vs `East Europe`(65)；`CENTRAL`(53) vs `Central`(33) —— 汇总时会被拆成两行，建议要求上游统一。

代码侧已确认**没有硬编码 market 枚举**（`Cols.MKT="market"`、`BookingsLowerPaneBreakdown.MARKET` 只是字段名；前端无枚举、按数据动态渲染），因此新增 17 个取值不会导致页面报错，只是 Market 维度多出细分项。

物化视图行数与"上一个好版本"相差几十至几千行（745,611 → 745,595）属正常：region 映射细化改变了聚合组合数，不是异常。

## 7. 并发与节奏约束

三环境读同一张 Hive 表（1555 万行），全表扫描会互相抢占 Trino。实测单轮约 9,500–10,700 行/秒，数据阶段约 32 分钟，含物化视图 + redis 一轮约 35 分钟。故串行推进：生产优先，跑完再按 tst → uat 各触发一次。

任务进度字段速查：`count_phase_status` → `data_phase_status`(cps/dps) → `materialized_refresh_status`(mv) → `redis_refresh_status`；`status` 依次为 `COUNTING / RUNNING / REFRESHING / SUCCESS`。
