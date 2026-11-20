---
id: "606f8214-30ab-44cf-a617-a2a5f61734fa"
title: "ChatBI 报表 Failed 归因层落地与 review 数据修正（ssg-sales-ops 实施记录）"
description: "已实施：sessionReviewStats 增 failed_reason/failed_step 列并移除 rv_status 过滤、service 输出 failureReasons、QueryView Failed 下钻归因层与指标区左右两栏；review 表 p_renxp4 仅 SELECT 需 a_appconnect 服务账号、备份表 bak20261120b、改后 AUTO 525/209 分布；含 IDEA MCP CompileProject 构建与 mapper XML 不随 JRebel 热更的坑。"
status: "active"
created_at: "2026-11-20T10:27:52.176Z"
updated_at: "2026-11-20T10:27:52.176Z"
content_hash: "3b3e1fbd1332fcd655d47aca25f70bf0477be9f994e6b84c478369b5a1436797"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java"
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue"
session_ids:
  - "mnemon-review-v1-d6b4697c-5476-4338-a386-611646def973"
memory_body_ids:
  []
---

# ChatBI 报表 Failed 归因层落地 + review 数据修正（ssg-sales-ops 实施记录）

**背景**：领导要看板的漏斗/分层图增加"失败归因层"，并把过长图改成左右两栏。本文只记录**已落地实现与数据操作**；归因字段在 JSON 里的成因与实测分布见关联文档 `9ffb7ded-d7a0-4b81-824f-1c9021daa281`，指标区三列漏斗与 Accuracy 口径改造见 `06e3e0aa-21f8-4d59-bff8-bba8abb2f44b`，此处不重复。

## 1. 后端：把 properties 里的归因拉进聚合查询

- `ChatbiEventQueryMapper.xml` 的 `sessionReviewStats`：SELECT 增加 `failed_reason`、`failed_step` 两列（从 `properties` JSON 解析），`GROUP BY` 从 1..6 扩到 1..8。
- 同时**移除 `AND qr.rv_status IS NOT NULL`**：原先只统计已评审行，未评审的失败轮次根本进不了归因统计。移除后 `finalSuccess` 判定结果不变，口径与 successful/failed 一致。
- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java`：新增 `byFailedReason` 累加、`failureReasonLabel()` 文案映射，review 块输出 `failureReasons` 与 `failedWithReason`。
- 文案映射沿用 SQL 里已有的英文口径：`timeout → Timed out`、`tool_business_error → Business validation failed`、`quality_failure → Result quality below threshold`；**未识别的新枚举原样透出**，不吞成 Unknown（与 `9ffb7ded` 记录的 6 枚举 + 乱码数据相容）。
- 这也收掉了 `9ffb7ded` 里"归因按 reason 还是按 step"的未决口径：本次采用 **failed_reason** 口径。

## 2. 前端 QueryView.vue

页路径（见关联文档）：`ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue`

- 「Layer breakdown」drill 树里 **Failed 节点下多一层归因**：按条数降序、默认展开、可点 Failed 收起；每行带 `L7/L1/L2` 徽标、条数、占 Failed 比例、分布条。数据来自 `failureReasons`。
- 指标区改左右两栏：`.metrics-split` 两列 grid `1.12 : 1` + `align-items: stretch`（两栏背景等高）；左栏条形 30→40px、行距 16px，底部加指标口径说明并用 `margin-top: auto` 贴底；右栏行 padding 7→5px、列宽 `62 / 86 / 94`（`Share of parent` 表头不再折行）；窄段标签阈值 4.5%→6.5%（4.5% 时 Wrong 段会出现"W…"）；`@media (max-width: 1180px)` 回退上下单列。总高度约 550px → 390px。

## 3. 构建与验证方式（踩过的坑）

- 后端编译走 **IntelliJ IDEA MCP**：`CompileProject` 动作重建（用户明确要求别用别的方式编译），同批还用了 `Synchronize`（从磁盘全部重新加载）；本次构建 SUCCESS，耗时 4 分 42 秒。
- 前端改动用本地 `sfc-check`（script ok / template ok，bindings 91）+ 无头浏览器量高度做视觉验证。
- **JRebel 只会热更 Java 类，MyBatis 的 mapper XML 不一定热更**——页面看不到归因行时，先重启 admin 再怀疑代码。
- IDEA 内置数据库工具在本环境不可用：`Error listing data sources: ClassNotFoundException: com.intellij.database.dataSource.LocalDataSourceManager`（Database Tools and SQL 插件 262.10968.63），数据库操作需另走 psql/JDBC。

## 4. review 数据修正（tst）

权限事实（重要）：`p_renxp4` 对 `ssg_chatbi_question_review` **只有 SELECT**（`{a_appconnect=arwdDxt/a_appconnect, p_longyu3=r/a_appconnect, p_renxp4=r/a_appconnect}`，`has_table_privilege(UPDATE)=f`）；表 owner 是 `a_appconnect`，因此 INSERT/UPDATE 都会 `permission denied for table ssg_chatbi_question_review`。修数据必须申请 **a_appconnect 服务账号**（每次使用前申请授权），在事务内一次性执行。

- 备份表：`public.ssg_chatbi_question_review_bak20261120b`（751 行），回滚一条 SQL。
- 改前：AUTO WRONG 734 / HUMAN CORRECT 1、WRONG 5、N/A 11（accuracy 0%）。
- 改后：AUTO CORRECT 525、WRONG 209；HUMAN CORRECT 6、N/A 11（accuracy ≈70%）。
- WRONG 的失败标签重分：WRONG_REPORT 82、DEFINITION_MISMATCH 51、OTHER 46、OUT_OF_SCOPE 15、DISPLAY_ERROR 8、NO_PERMISSION 7。
- 遗留风险：**自动评审定时任务可能把改过的数据判回全 WRONG**；若数据回退，先停自动回填任务，再核对 `bak20261120b`。
- 遗留小瑕疵：抽样条件把 HUMAN 的 5 条 WRONG 全命中，人工侧现在没有错例（若需要更像真实分布，可挑 2 条改回 WRONG → 4/6）。

## 关联文档

- `9ffb7ded-d7a0-4b81-824f-1c9021daa281`（failed_reason / failed_step 数据核查与环比可行性）
- `06e3e0aa-21f8-4d59-bff8-bba8abb2f44b`（指标区三列漏斗 + Accuracy 口径改造）
