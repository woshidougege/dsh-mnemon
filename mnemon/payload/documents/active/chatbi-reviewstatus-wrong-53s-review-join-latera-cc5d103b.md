---
id: "cc5d103b-628e-4216-a1a6-08ef73e07f46"
title: "ChatBI 明细 reviewStatus=WRONG 53s 超时定位（REVIEW_JOIN→LATERAL）+ 评审自动建单落地 MR !559/!1054/!1055"
description: "派生表 LEFT JOIN 因 ON 里两个 COALESCE 无法走索引、规划器反转为评审表驱动（736 loops / 35 万行重扫）的排查证据与 58,743ms→752ms 结果；评审 WRONG 自动建 Jira 单最终口径（含自动评审、72h 回看、三级查重、advisory lock、新表 V1.1.127、开关默认关）与三支 MR 交付记录。"
status: "active"
created_at: "2026-11-16T10:18:46.206Z"
updated_at: "2026-11-16T10:18:46.206Z"
content_hash: "d1afd26cb1e15604422f04efd1f3558405fe50acfdb7dbaa194213925588243c"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/controller/ChatbiEventReportController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiReviewJiraMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/jira/ChatbiReviewJiraCreator.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/jira/ChatbiReviewJiraScheduler.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/jira/ChatbiReviewJiraService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/probe/service/JiraFlow.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/probe/service/ProbeSettingService.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.127__Create_chatbi_review_jira.sql"
session_ids:
  - "mnemon-run-v1-641a978f-2607-4c92-9047-21256162c492"
memory_body_ids:
  []
---

# ChatBI 问数明细 reviewStatus=WRONG 53s 超时定位 + 评审自动建单落地（MR !559 / !1054 / !1055）

状态：**已实现并推送，三支 MR 处于 opened 状态，等待用户决定合并。** 合并、部署仍需单独授权。

关联文档：
- **42d7af72-3dd9-4757-9a98-2475a21d93a8** —— 同一需求的第 5 条 JiraFlow 链路方案分析（当时仅分析、三项待确认）；本文记录最终落地口径与差异。
- 自动评审判定口径与 backfill 修复：**2b13c2e2-62ad-49c9-b0f2-5bb15637a132**
- 评审三态/归因表立项：**750e10a9-15ec-4ab0-9295-1f45e6d90bfd**、留痕：**6353a23e-5302-4b9c-9301-049e028565d3**

## 一、超时定位：只有 reviewStatus=WRONG 会 53s 超时

现象：问数明细页 reviewStatus 筛选在 WRONG 取值下稳定 ~53s 超时；其他取值 1.6~6.4s；size=1 也照样 53s，说明与返回行数无关。

根因（性能证据来自生产库 EXPLAIN (ANALYZE, BUFFERS)）：
- `ChatbiEventQueryMapper.REVIEW_JOIN` 原先写成派生表 + LEFT JOIN，ON 条件里含两个 COALESCE 表达式，索引无法生效。
- 规划器只能反向以评审表为驱动表，对命中窗口的评审行逐行重扫事件侧：`Rows Removed by Join Filter: 351787`、`loops=736`。
- 完整 SQL 58,743ms，shared buffers 命中约 1480 万。

修复：把 `REVIEW_JOIN` 改为 `LEFT JOIN LATERAL (… LIMIT 1)`，走主键。同一数据库实测 **58,743ms → 752ms**（Execution 244ms），buffers 降到约 2.7 万，返回行数与窗口值不变。

注意：`REVIEW_JOIN` 只有一处定义，明细、导出、汇总、reviewStats 共用，改一处即全生效。修复单独成一个 commit（`03f75a0bf` perf(feedback)），与功能 commit 分开，便于回滚。

上线后候选扫描 SQL 复验：`Limit → Nested Loop Left Join → Index Scan Backward using idx_ssg_chatbi_question_review_reviewed`，Execution 35.4ms，planlines=18。

## 二、评审 WRONG 自动建单：最终口径（与方案分析文档的差异）

- **触发**：`ssg_chatbi_question_review.status='WRONG'`，**不区分人工评审与自动评审**（方案阶段待确认项，最终决定都建单）。`NOT_APPLICABLE`（No permission / Out of scope）不算缺陷，永不建单。
- **已有单**：三级查重 —— 本链路台账 → 失败链路台账中同一轮的 Jira 单号 → Jira 按 turn 反查；命中只 `addComment` 追加本轮评审结论，台账记 `mode=created/commented`。
- **防重复**：新表 `ssg_chatbi_review_jira`（`V1.1.127__Create_chatbi_review_jira.sql`，只建表不写数据），主键 `session_id + turn_key`，与评审表口径一致；建单事务内再取 `pg_advisory_xact_lock`，多实例并发扫描也只建一张单。
- **配置**：`jira.review-assignee` 默认 `longyu3`（不回退 `jira.assignee`，沿用失败链路约定）；开关 `jira.flow.review.enabled`，**默认关闭**。
- **开关位置**：Admin → Settings → Integrations → Jira 面板（不放 Review rules，因为两个页面都整页保存全部可写键，互相覆盖）。
- **扫描参数**：每 15 分钟一轮，回看 72 小时（方案阶段提的 7 天最终改为 72h），单条最多 3 次，单轮最多 50 条。
- **手工触发**：`POST /chatbi/events/report/review-jira/scan`，`dryRun` 默认 true。

## 三、代码与交付面

后端（ssg-sales-ops，admin 服务，基线 origin/admin-release 049d2df49）：
- 改：`ChatbiEventReportController.java`（+30）、`ChatbiEventQueryMapper.java`（+23/-9）、`probe/service/JiraFlow.java`（+15）、`probe/service/ProbeSettingService.java`（+30）
- 增：`chatbi/dao/ChatbiReviewJiraMapper.java`（143 行）、`chatbi/jira/ChatbiReviewJiraCreator.java`（291）、`chatbi/jira/ChatbiReviewJiraScheduler.java`（43）、`chatbi/jira/ChatbiReviewJiraService.java`（165）、`db/migration/V1.1.127__Create_chatbi_review_jira.sql`（86）
- 分支 `feature/review-wrong-jira-admin`，功能 commit `aa1716ebc`，性能 commit `03f75a0bf`，共 9 文件 +813/-13 → **MR !559 → admin-release**

前端（ssg_sales_ops）：
- 唯一改动文件 `src/views/Setting/settings/panels/JiraPanel.vue`（+12），加开关与指派人
- 生产分支 `dev/renxp4` `318a5fea` → **MR !1054 → master**；TST 分支 `tmp/review-jira-develop` `b654af06` → **MR !1055 → develop**
- 两条分支 `git patch-id --stable` 一致 `597b65f205afa5eb5a32b20eedd343fbaecdc15a`，两侧 `npm run type-check` 均通过

验证结果：后端 `mvn -o -T 4 -pl smartop-service/admin -am -DskipTests compile` BUILD SUCCESS（在 049d2df49 上重跑）；三支 MR 回查均 `state=opened merge_status=can_be_merged conflicts=False`（!559 files=9、!1054 files=1、!1055 files=1）。

## 四、上线注意

1. 开关默认关闭，部署后行为不变，需在 Jira 面板手动打开 Review wrong 才真正建单（与失败链路一致）。
2. 打开后只处理打开之后 72 小时内的新 WRONG 评审，不会把历史那 700 多条一次性刷成单。
3. 建议先发 tst，用 `dryRun=true` 触发一次查看候选，再打开开关。

## 五、相关既有口径（同一模块）

明细的提问取值：`SESSION_QUESTION_SQL = COALESCE(NULLIF(question,''), NULLIF(properties->>'question_content',''), NULLIF(properties->>'original_question',''), NULLIF(session_title,''), '')`；`TURN_TYPE_SQL` 再把以 `User has answered your questions` 开头的轮判为 `clarify_answer`、以 `The user did not answer` 开头的轮排除，保证表格行数与卡片上的 N questions 一致、成功率不被这些非提问行冲淡。
