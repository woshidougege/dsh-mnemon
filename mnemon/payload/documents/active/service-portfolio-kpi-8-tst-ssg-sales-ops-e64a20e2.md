---
id: "e64a20e2-b44a-4c2e-a9ba-dfd6aaf9ee20"
title: "Service Portfolio KPI 卡配置落地：会议口径裁定、8 行目标配置与 TST 写权限阻塞（ssg-sales-ops）"
description: "KPI 只有 1 张卡的根因是 sp_sp_kpi_card 只配了 1 行可见卡 + 2 行联调残留；据 2026-10-28 需求评审录屏裁定走配置方案 A（链接必须后台可维护、否决 defaultsOfKpi 兜底）；给出 8 行目标配置与 OH_CODES 误配 1617K 的修正；TST 个人账号仅有 SELECT、表 owner 是服务账号 a_appconnect，写入被拒，待授权。"
status: "active"
created_at: "2026-11-17T14:14:09.384Z"
updated_at: "2026-11-17T14:14:09.384Z"
content_hash: "f040fffc716cb3d8932b27b2402031e7013d7ef212582f8e7bc2022b1635a779"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java"
  - ".tmp/pqprobe/kpi-fix.sql"
session_ids:
  - "mnemon-run-v1-4738018a-9b7f-438b-8f2f-09891fdaa8e9"
memory_body_ids:
  []
---

# Service Portfolio KPI 卡配置落地：会议口径裁定、8 行目标配置与 TST 写权限阻塞

日期：2026-11-17。页面：`https://ssgspark-tst.lenovo.com/spark/portfolio`（Business Insights / Big Events 顶部 KPI 卡）。

## 1. 问题现象

用户反馈「KPI 只有 1 张卡」。TST 实测 `public.sp_sp_kpi_card`：

```
总行数 = 3，visible = 1
id=1 | OH_CODES      | Offering Hierarchy Codes | MANUAL  | manual_value=1617 | unit=K | visible=t | sort_no=1
id=2 | sp-probe-kpi  | PROBE-DELETE-ME          | MANUAL  | 空                | 无     | visible=f | sort_no=2
id=3 | PROBE_KPI_NEW | PROBE-DELETE-ME          | MANUAL  | 空                | 无     | visible=f | sort_no=3
```

结论：不是接口 bug，是**配置没配全**；另有 2 行联调残留（`PROBE-DELETE-ME`）。同时 `OH_CODES` 被误配成 `MANUAL/1617/unit=K`，页面上因此显示成 `1617K`——`countOhCodes()` 算出的是 1,617 条 distinct OH code，本身不需除以 1000、不该带 K 单位。

## 2. 关键决策：A 而非 B（会议口径裁定）

初次给出的方案 B「`defaultsOfKpi()` 默认 8 张打底 + 配置覆盖」被用户质疑（「为啥你推荐改代码呢」）。回查需求评审录屏转写《SPARK · Service Portfolio 需求评审会（2026-10-28）》第 02 章「KPI 卡 · Business Highlights」，会议结论与方案 B 相反：

- 顶部一排 KPI 卡，**数字随后台数据刷新而变化**；
- 点击卡片跳对应 dashboard（PBI 或资料库）；个别卡片不是外链，如 Concord 的「30+」跳本页 Tab；
- **跳转链接必须能在后台维护，不能写死**——理由：PBI 发版、改版、做排序都会让链接变化（历史上有过链接整体变更）；
- 每张卡的数据来源由业务方提供，DAC 里会有说明，业务方后续还会给增强口径。

原话（时间码取自录屏转写）：

| 时间 | 原话 |
|---|---|
| 20:46 | 「这个需要我们后期自己再去配置吗？还是说这个其实就是固定的？比如说他上面这个 1.6k 这个 offering hierarchy code 的…」 |
| 22:23 | 「这一项他其实是会一直来自于同样的这个 PBI 对吧？他不需要我们再去给用户一个比如说你自己再去配置说他来自于哪儿的」 |
| 23:33 | 「因为咱们那个 Power BI 他以后…链接的给我一个我们可以维护的地方」 |
| 39:94 | 「他点击去哪儿，这个我们是希望自己可以…后边去修改，对吧？对对，就是也就是**用户可以配置**，就不用我们再额外的去开发」 |

**裁定**：走 **A**——把 8 张卡配齐，让配置表成为唯一事实来源。方案 B 等于把「配了几张就显示几张」变成「永远显示 8 张」，正是会议排除的「固定死」；且 `defaultsOfKpi()` 本身是兜底，与用户此前定的「算不出来的不要兜底造假，接口直接返回空结构」相冲。配置为空时接口应返回空数组、前端显示空态（前提是配置必须先配齐）。

用户明确拍板：「你帮我改库吧 走 1」。

## 3. 目标配置：8 行（替代现有 3 行）

| sort | metric_key | title | value_type | unit | 点击跳转 |
|---|---|---|---|---|---|
| 1 | OH_CODES | Offering Hierarchy Codes | COMPUTED | 无 | LIBRARY（资料库） |
| 2 | ACTIVE_PN | Active Part Numbers | COMPUTED | K | PBI |
| 3 | ODT_PROJECTS | ODT Projects | COMPUTED | 无 | TAB·concord |
| 4 | REQUESTS | Requests | COMPUTED | 无 | TAB·portfolio |
| 5 | FAST_PASS | Fast Pass Rate | MANUAL 94 | % | PBI |
| 6 | CROSS_TEAM | Cross-Team Engagement | MANUAL 9 | + | LIBRARY |
| 7 | SLA | SLA | MANUAL 1.5 | WD | PBI |
| 8 | PN_TERMINATED | Service PN Terminated | COMPUTED | K | TAB·lifecycle |

- COMPUTED 由后端 `resolveMetric` 实时算（`OH_CODES`/`ACTIVE_PN`/`PN_TERMINATED`/`ODT_PROJECTS`/`REQUESTS` 五个 SQL）；`FAST_PASS`/`CROSS_TEAM`/`SLA` 无数据源，取业务 PPT 值（94% / 9+ / 1.5WD）。
- 按「迁移只建表、不塞数据」的既有规矩，这批数据**不写进 flyway**，只作为一次性 DBA/配置操作。

表结构（`V1.1.113` 建表，`smartop-service` 迁移 204-227 行）：`id, metric_key, title, subtitle, unit, value_type(默认 MANUAL), manual_value, link_type(默认 NONE), link_url, permission_key, sort_no, visible, updated_by, updated_time`；唯一索引 `uk_sp_sp_kpi_card_key (metric_key)`。表注释：`'Big Events 顶部 KPI 卡配置：链接可维护，不能写死'`；`link_type` 取值 `PBI / LIBRARY / TAB / NONE`。

## 4. 阻塞：TST 写权限（未解决）

待执行 SQL 已落盘：`kpi-fix.sql`（仓库 `.tmp/pqprobe/kpi-fix.sql`，24 行 = 1 条 DELETE 清残留 + 1 条带 `ON CONFLICT` 的 UPSERT）。执行失败：

```
ERROR: permission denied for table sp_sp_kpi_card
  at PqFix.main(PqFix.java:28)
```

权限实测（`current_user = p_renxp4`，PostgreSQL 16.4）：

```
has_table_privilege → sel=t | ins=f | upd=f | del=f
tableowner (pg_tables) → a_appconnect
information_schema.role_table_grants → grantee=p_renxp4, privs=SELECT
pg_roles → p_renxp4 / p_longyu3 均非 superuser
```

即：个人账号在 TST 对该表**只读**，表 owner 是服务账号 `a_appconnect`。按 dev-requirements 规定，服务账号须逐次申请授权。给用户的三条出路（等待选择）：

1. 批准本次以 `a_appconnect` 执行该 DELETE + UPSERT（只改 `sp_sp_kpi_card` 这 8 行数据，不动表结构）；
2. 走后端配置接口 `PUT /service-portfolio/config/kpi-cards`（服务端用服务账号连库，`requireAdmin` 校验，需浏览器取当前 token）；
3. 用户自行执行 SQL，仅做后续验证。

## 5. 未闭合事项

1. 8 行配置尚未写入 TST（卡在权限），2 行 `PROBE-DELETE-ME` 残留仍在；
2. `OH_CODES` 仍为 `MANUAL/1617/unit=K`，页面仍显示 `1617K`；
3. `defaultsOfKpi()`（`ServicePortfolioReadService` 第 99-111 行，配置空时兜底 8 行）是否删除待定——与会议口径和不兜底原则冲突，但删掉前必须先配齐配置，否则页面全空；
4. `FAST_PASS`/`CROSS_TEAM`/`SLA` 的最终口径仍在等业务方给数据源。

## 6. 相关文档

- `5b52aa68-ce8b-4e5c-9d25-575abd544793`：Service Portfolio（admin）身份接口清理与 KPI 卡片文案来源排查——记录了当时「配置表 0 行、全部走 `defaultsOfKpi()` 兜底」的旧状态，现已变为 3 行且由此引发本次问题。
- `936f3b21-d21d-435b-acf3-41fc411f9eaf`：Service Portfolio 前后端全量缺口审计（含「空表 / 缺表 / 无数据源」三类区分）。

## 7. 参考路径

- `smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java`：`kpiCards()`（第 63-83 行）、`resolveMetric()`（第 85-97 行）、`defaultsOfKpi()`（第 99-111 行）。
- `.tmp/pqprobe/kpi-fix.sql`、`.tmp/pqprobe/PqKpi.java`（只读探测）。
