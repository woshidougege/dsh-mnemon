---
id: "6c0e9cea-ef5e-4642-bf38-1aba552f7258"
title: "ChatBI 问数记录 Source report 七态口径落地与三 MR 交付（含生产分布与性能代价）"
description: "四态→七态（routed/exec/resolve/carried/route_failed/none/no_audit）的最终判定与显示规则、文案须与后端 SOURCE_LABEL_SQL 逐字一致、生产 4909 轮分布（Not routed 24%→8.9%）、全量统计 +343ms 与弃用的 4 路 UNION、前后端 5 文件改动、!577/!1076/!1077 三个 MR 与分支提交、遗留的 tool_audit_logs 索引与 stash/中文标题工程经验。"
status: "active"
created_at: "2026-11-18T03:10:39.768Z"
updated_at: "2026-11-18T03:10:39.768Z"
content_hash: "b8361a2f509ce91a50aefb1491c56d8eb979b6531174b124b6a674bd7ec2787a"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java"
session_ids:
  - "mnemon-run-v1-e186830b-cb52-4772-bc9d-a5c4c6cc71c5"
memory_body_ids:
  []
---

# ChatBI 问数记录 Source report 七态落地与三 MR 交付（2026-11-18）

取代 / 演进自 `25fb29f1-f532-44dc-b110-618cf4796fdc`（轮次级改造实施方案，四态 routed/route_failed/not_routed/no_audit）与 `d7382448-aaf9-4003-971f-58f1f56da61b`（三列取值缺陷排查）。原方案的 **not_routed 二态被细分为 none / exec / resolve / carried**，四态 → 七态。

## 1. 最终口径：routeState 七态（互斥）

| 状态 | 判定 | 界面显示 |
|---|---|---|
| routed | 本轮调 `route_report` 且成功 | 报表名 |
| exec | 未重新路由，但本轮真的执行了查询 | 报表名 |
| resolve | 只到编译、未执行 | 报表名 |
| carried | 本轮无路由记录、确有查询动作，沿用本会话此前报表 | `↺ 报表名`（U+21BA；无名字时 `Carried over`） |
| route_failed | 调了路由但失败 | `报表名 · route failed` / `Route failed` |
| none | 有审计但本轮完全无报表痕迹 | `No report this turn` |
| no_audit | 会话无 agent 审计，无从判断 | `No audit data` |

设计要点：

- **carried 是推断值**，必须用 `↺` 前缀与"本轮真的路由到这份报表"区分，否则会被误读。前端常量 `CARRIED_PREFIX = '↺ '`。
- **文案必须与后端 `SOURCE_LABEL_SQL` 逐字一致**：前端 `sourceDistribution` 用 `bySource('No report this turn')` 这类字面量回算占比，改一个字（如 `Not routed` → `No report this turn` 漏改一侧）就会全部算成 0。已在此处踩过坑，改动文案时必须前后端同改。
- 非 Smart Data Query 轮没有报表概念，直接显示 `—`。

## 2. 生产实测（近 30 天，Smart Data Query，4909 轮）

routed 3522 / 71.7%；exec 518 / 10.6%；route_failed 171 / 3.5%；resolve 146 / 3.0%；carried 100 / 2.0%；none 437 / 8.9%；no_audit 15 / 0.3%。

可指出报表名的比例由约 **76% → 90.8%**；`Not routed` 桶从约 24% 降到 **8.9%**，且这 8.9% 是真无报表可指（纯对话、问定义、点赞点踩），不是解析失败。

数据来源可信度高：用反射把 mapper 里的 SQL 常量原样 dump，先按 MyBatis XML 实体还原（`&lt;` → `<`），再在生产库跑 `EXPLAIN ANALYZE` 与分布统计 —— 跑的就是上线后的同一条 SQL。

## 3. 性能

- 全量统计查询 **1.56s → 1.90s（+343ms）**；明细页只对当页几十行算 LATERAL，影响更小。
- 曾试 4 路 UNION 写法：1.6s → 3.2s，**已弃用**。
- 现结构：一次 `(session_id, turn_id)` 索引扫描 + 聚合，`carried` 用 `CASE` 短路（只有前三路都空才回溯上一轮）。

## 4. 代码改动

后端（`C:\Users\63588\lenovo\ssg-sales-ops`，+110/−35）：

- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java` — 来源判定、标签、筛选键、`routeState` 全部重写；删掉多余的 `ROUTE_ONLY_LATERAL_SQL`；注释在 648-649 行说明了 not_routed 拆成 exec/resolve/carried 的原因。
- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java` — `reportName` 只在真正落到报表时给值。
- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java` — 导出同口径，`carried` 带 `↺`。

前端（`C:\Users\63588\lenovo\ssg_sales_ops`，+25/−17）：

- `src/views/Setting/reports/user-logs/components/views/QueryView.vue` — 分布卡片 `Not routed` → `No report`（154 行 `noReport` 桶、301 行 `sourceText`、374 行列表兜底文案）；`sourceText` 按七态渲染，`isSourcePlaceholder` 改为 none/no_audit。
- `src/views/Setting/reports/user-logs/components/views/FeedbackView.vue` — 点赞点踩页同口径（routed/exec/resolve/carried/route_failed 才落到报表，其余 `—`）。

校验：后端 `mvn compile` BUILD SUCCESS；前端 `vue-tsc --build` 通过、eslint 0 error（48 个既存 `any` warning，非本次引入）。

## 5. 交付（三个 MR，均已 opened / can_be_merged / 无冲突，未合并）

- **!577** `fix/chatbi-source-report-admin → admin-release`（repo ssg-sales-ops），commit `6731c8b1d`，基线 `origin/admin-release` `638a8f7b8`
  https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/577
- **!1076** `fix/chatbi-source-report-master → master`（repo ssg_sales_ops），`a86ab5e6`
  https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1076
- **!1077** `fix/chatbi-source-report-develop → develop`，`d20b180d`（从 master 分支 cherry-pick；`git patch-id --stable` 两侧第一列均 `c2465914b9f27d2ed0a7fa3c52dd416b6b0ffe6f`）
  https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1077

合并顺序按项目惯例：develop 先、master 后，合并第一个时不勾 Delete source branch；后端 !577 独立合。三个 MR 无表结构变更，不需要 flyway。

## 6. 遗留

- `agent.tool_audit_logs` 仍只有 `(session_id, turn_id)` 索引。若该页面要扛更大时间范围，加 `(session_id, started_at)` 索引可再压 `carried` 回溯那一步，需单独出迁移。
- 前端工作区有未跟踪目录 `ssg_sales_ops/.tmp-fe-sp/`（portfolio 相关 ts/vue 快照），非本次产生，未动。

## 7. 可复用的工程经验

- **未提交改动换基线用 stash 迁移，不要复制文件**：`git stash push` 指定文件 → 切到目标分支 → `git stash pop`；两个仓库都 auto-merge 干净落地，改动行数前后一致（BE 110/35、FE 25/17）。切基线前 admin-release 已有新提交，也没冲突。
- **PowerShell 传中文参数易编码翻车**：MR 标题/描述先写 UTF-8 无 BOM 文件（`.tmp/mr-title-*.txt`），脚本用 `Get-Content -Encoding UTF8` 读出再传；回查 title_len 50/56/56 属中文按字符计的正常长度，未变成 `æéå...`。
- 描述里 `carried` 前缀曾误写成 `↗`（U+2197），正确是 `↺`（U+21BA），建单前用字符码位校验修正。
