---
id: "d6ae03bd-2e22-43c2-8884-a4cda6b516f3"
title: "SSG 同步任务跨环境同步：LGFS 逻辑合并、cron 统一与 p49_g1189 列权限阻塞（ssg-sales-ops）"
description: "ssg_sync_task 任务的跨环境同步记录：LGFS billing/asset 生产落库配置与 APPEND 理由、测试 cron 统一周三 10:00、生产 cron 现状、p49_g1189 列级 SELECT 权限阻塞与 APPEND 去重建议。"
status: "active"
created_at: "2026-11-12T10:21:57.504Z"
updated_at: "2026-11-12T10:21:57.504Z"
content_hash: "44d9dce11e3e7fcaddc7c5466a22afe0ff440326c2b4de9819369b746957a153"
source_paths:
  - ".tmp/sync-groups/prod_lgfs_task5_backup.json"
  - ".tmp/sync-groups/tst_cron_backup_20261112.json"
session_ids:
  - "mnemon-run-v1-a12ce66b-f12a-4e69-92c7-e598be8e1691"
memory_body_ids:
  []
---

# SSG 同步任务（Sync Center / ssg_sync_task）跨环境同步：LGFS 逻辑与 cron 调整记录

相关文档：admin 启动耗时排查（`114173fc-0b85-4307-b84e-887ba2faaea3`）也涉及 `ssg_sync_task` 的补注册逻辑（`CocFlagSyncTaskRegistrar` 幂等按 `handler_key` 补行，cron 只在新建时写入）。

## 1. 任务配置的存放与访问路径

- 任务定义在生产/测试 PG 的 `ssg_sync_task` 表：`task_name`、`target_table`、`source_sql`、`write_mode`（APPEND / TRUNCATE）、`cron_expr`（Quartz 6 段式）、`enabled`、`next_trigger_time`、`created_by`、`created_time`、`updated_time`。
- 写权限：个人账号 `p_renxp4` 对 `ssg_sync_task` 只有只读（写时报 `ERR: permission denied for table ssg_sync_task`），改配置需走 `a_appconnect`。
- 调度执行用的 Trino/Presto 账号是 **`p49_g1189`**（`db_host` = `pbi-presto-row.ludp.lenovo.com`，生产 413 条查询、测试 323 条，是两边的主力账号）。按表申请列权限时要指名这个账号，不是个人账号。
- 从 Admin Control 页面建的任务会写 `created_by`；`created_by` 为空说明是直接改库或其它入口建的（11-11 新增的 5 个 simu/tcv 任务属此类）。

## 2. 前端分组规则与 cron 现状

- 分组匹配（Scheduler Center 前端逻辑）：`target_table` 以 `lgfs` 开头、含 `lilo`，或等于 `dim_svc_acnt_group` → LGFS 组；以 `gps` 开头 → GPS；`bkgs` 且不以 `gps` 开头 → Bookings。
- 生产 cron 约定：Bookings 三项（1/2/4）`0 0 7 * * ?`（07:00）；其余全部 `0 30 7 * * ?`（07:30），含 cfs、LGFS 组、GPS 组、I&C、Other、pnpl；simu/tcv 五项是凌晨日跑（`04:45`/`04:50`/`04:55`），history 两个周日 `03:30`/`03:45`。
- 测试环境 2026-11-12 已统一为「每周三」：16 个任务改成 `0 0 10 ? * WED`，`simu_customer_week_history`/`tcv_customer_history`（id 51/54）特意错开到 `0 30 10 ? * WED`，因为它们与 `simu_customer_week_recent`/`tcv_customer_recent`（id 50/53）写同一张目标表，同刻跑会冲突。原 cron 备份：`.tmp/sync-groups/tst_cron_backup_20261112.json`。
- 改 cron 后 `next_trigger_time` 需重算，否则仍按旧时间触发。

## 3. LGFS 组同步操作（2026-11-12 执行）

测试侧「新逻辑」是当天 18:01 / 18:07 改的（生产此前没有），同步到生产并**保留生产原 cron**：

| 环境 | id | 任务 | source_sql | write_mode | cron |
|---|---|---|---|---|---|
| 生产 | 5 | lgfs_lease_billing_summary | `select * from prd_upsell.lgfs_lease_billing_summary`（原为自写 CTE，join cml_account_dedup / country_geo_mapping / service_account_group） | TRUNCATE → **APPEND** | `0 30 7 * * ?`（未变） |
| 生产 | 39 | lgfs_lease_asset_summary（新插入，created_by renxp4，18:13） | `select * from prd_upsell.lgfs_lease_asset_summary` | APPEND | `0 30 7 * * ?` |
| 测试 | 5 | lgfs_lease_billing_summary | 同上 | APPEND | `0 0 10 ? * WED` |
| 测试 | 56 | lgfs_lease_asset_summary | 同上 | APPEND | `0 0 10 ? * WED` |

要点与理由：
- 改成 APPEND 是为了保留历史快照（生产 billing 20,824 行 / 3 个快照，asset 45,705 行 / 5 个快照，最新都停在 2026-10-19）。TRUNCATE 会把这些冲掉。
- 生产 LGFS 组同步后共 4 个任务、cron 全为 07:30：id 5（ON）、6 funder_report_lilo_and_lfs_detail（OFF）、28 dim_svc_acnt_group（OFF）、39（ON）。
- 改前配置备份：`.tmp/sync-groups/prod_lgfs_task5_backup.json`。
- `lgfs_lease_asset_summary` 早前是「有表没任务」（生产 45,705 行、测试 27,158 行，`ssg_sync_task` 里按 `target_table`/`source_sql` 都搜不到），靠外部途径灌数；本次才纳入调度。

## 4. 未解决阻塞：列级 SELECT 权限

`p49_g1189` 对两张新源表 **`count(*)` 能通、按列查被拒**（表在、能连，缺列权限），因此 2026-11-13 07:30 起两个任务都会 FAILED：

- `prd_upsell.lgfs_lease_billing_summary`：`Access Denied: Cannot select from columns [tranche_id, sold_to_party_id, customer_group, geo_cd, deal_type, segment, sub_segment, contract_start_date, invoiced_usd, remaining_usd, subscription_usd, snapshot_date]`
- `prd_upsell.lgfs_lease_asset_summary`：同型报错，17 列全拒（含 `snapshot_date`、`fv_rv_usd`、`asset_count` 等）
- 历史同类：`gps_pnpl_master_data`（7 列被拒，任务 id 32 最后一次 FAILED 11-12 10:50）、`cml_account_dedup`（id 5 从 2026-08-28 建任务起从未成功）

结论：权限申请单要同时cover 这几张表，且明确授予 `p49_g1189`；测试环境同一账号，同样会红。

## 5. 已知风险与建议（未采纳，待定）

- APPEND 模式无去重：同一天重试/补跑成功两次会插出重复快照。建议加 PRE hook 先删当天 `snapshot_date` 再写入。测试侧也未配 hook（`hooks: []`）。
- 生产 cron 是否统一到 07:00 或 07:30、凌晨跑的 simu/tcv 五项是否并入 07:30，用户尚未决定。
