---
id: "e087714a-341f-42af-9178-50ee7d507f2c"
title: "usage-issue-feedback 评审统计口径诊断：只覆盖点踩回答（42 vs 55）与 A/B 方案"
description: "用户问「逻辑没同步到 usage-issue-feedback 页面吗」的根因调查：评审表主键 (session_id,turn_key) 而 ssg_usage_issue_feedback 无会话列，后端 AdminFeedbackTimelineMapper:424 只统计 kind='downvote'，前端 FILTER_SQL 使 issue 行被静默过滤；含两个修法（文案提示 vs 让 issue 进评审）的权衡与未完成交接。"
status: "active"
created_at: "2026-11-13T23:34:05.065Z"
updated_at: "2026-11-13T23:34:05.065Z"
content_hash: "a16b6cc4d7f109a6b75b5b36107d8aa943038cedb8d2c5285ea62de6f911302d"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/dao/AdminFeedbackTimelineMapper.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.52__Create_feedback_notification_center.sql"
session_ids:
  - "mnemon-run-v1-8befe3ea-add2-4485-ab87-afbb89c0a372"
memory_body_ids:
  []
---

# usage-issue-feedback 评审统计口径诊断：为何评审只覆盖点踩回答（42 vs 55）

> 状态（2026-11-14）：**调查已完成，方案 A/B 待用户拍板**，尚未改代码。
> 相关文档：ChatBI 人工评审 JOIN 二义性 500 修复（4a6ba38c-451d-4a48-b625-05b6d75db712）、评审改动留痕 V1.1.120（6353a23e-5302-4b9c-9301-049e028565d3）、Success Rate → Accuracy Rate 设计（38b9c4c0-0988-4206-9198-11f20587ae61）。

## 1. 现象

用户看到 usage-issue-feedback 页面 Review status 计数为 **Correct 0 / Wrong 6 / Pending 36 = 42**，而表格 Total 是 55，Overview 的 Downvoted Answers 是 42。于是问「逻辑没同步到 usage-issue-feedback 页面吗」。

结论：不是同步问题，是**评审粒度只到「一次问数」，不覆盖提交的问题单**，且分母从未在 UI 上说明。

## 2. 根因（三层证据）

**数据模型层**
- `public.ssg_chatbi_question_review` 的主键是 `(session_id, turn_key)`：turn_key 对点踩取 `rf.turn_id`，对问数记录取 `query_id`。
- `public.ssg_usage_issue_feedback` 实测列（TST，information_schema 直查）：
  `id / request_id / reporter_user_id / page_path / page_name / feedback_type / impact / title / description / steps / expected_result / contact / browser_info / status / jira_key / jira_url / error_message / attachment_error / retry_count / created_at / updated_at / original_question / last_notified_status / done_notified_at / jira_assignee`
  —— **没有 session_id，也没有 turn_id**，所以 issue 行结构上无法关联评审。
- `session_id / turn_id` 只加在 `public.ssg_feedback_jira_issue` 上（`smartop-service/admin/src/main/resources/db/migration/V1.1.52__Create_feedback_notification_center.sql` :51-54），且历史数据大量缺失：总 124 行、仅 67 行有值。

**后端口径**
- `smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/dao/AdminFeedbackTimelineMapper.java:424` 统计评审时显式加 `AND c.kind = 'downvote'`，注释即「只统计点踩行——问题单没有会话，本来就没有可评审的轮次」。

**前端口径**
- `C:/Users/63588/lenovo/ssg_sales_ops-wt-review-polish-master/src/views/Setting/reports/usage-issue-feedback/index.vue:517-520`：`hasReviewLink()` 要求 `row.sessionId && (row.reviewTurnKey || row.turnId)`，issue 行一律 false → Review 按钮置灰，tooltip 文案「Only downvoted answers link to a question」（:1225-1239）。
- 同文件 `FILTER_SQL` :233-241 的评审条件只对 `kind='downvote'` 成立，所以**一旦勾选任一评审筛选（status / tag / reviewer / source），13 条 issue 行既不在已评审集合也不在 PENDING 分支，会被静默过滤掉**：Total 从 55 掉到 36 且无任何提示。这是真正的缺陷点。

**当次查到的库内计数（TST 快照，全量口径，非页面筛选口径）**
- 有效点踩（rating=-1 且未 retracted 且排除 probe run）：115
- `ssg_chatbi_question_review`：CORRECT 1 / WRONG 738
- `ssg_usage_issue_feedback`：20 行（original_question 全部非空）
- `ssg_feedback_jira_issue`：124 行，session/turn 有值 67

## 3. 两个方案与权衡（已向用户提出，未决策）

**A. 补文案与联动提示（推荐；不动数据口径，当天可上）**
- Review status / tag / reviewer / source 下拉里注明「Counts cover downvoted answers (42)」。
- 一旦选了任一评审条件，表格上方显示提示：`Review filters apply to downvoted answers only — 13 submitted issues are not included.`，让 Total 变化有解释。

**B. 让 issue 也进评审（要动提交链路，历史补不回来）**
- 前端提交反馈时带当前 `sessionId + turnId`；`ssg_usage_issue_feedback` 加两列；`AdminFeedbackTimelineMapper` 的 issue 分支补 `LEFT JOIN ssg_chatbi_question_review`。
- 代价：已提交的 20 条历史数据永远没有会话，只能长期标「不可评审」；且用户在 Data Workbench / P&L / No Access 等非问数页面提交的反馈本来就没有轮次。

建议 A；B 只在「要求所有反馈都能评审」时才值得做。

## 4. 未完成事项（交接）

- 方案 A/B 等用户选择后再动手。
- MR `!549`（`FROM FROM` 导致的 500）等待用户点合并（MR 合并只能用户操作）。
- commit `65ed85a54` 上次推送失败，未确认已推到远端。失败形态是 GitLab 侧 TLS 握手问题，不是认证：`gitlab.xpaas.lenovo.com` 解析到 10.99.205.6；`git ls-remote` 报 `schannel: failed to receive handshake, SSL/TLS connection failed` / `TLS connect error: error:0A000126:SSL routines::unexpected eof while reading` / `Send failure: Connection was aborted`；`curl` 首页与 api 均 000（exit 35），一次 exit 52 empty reply；改 `http1.1` 重试同样失败；同期 TST 直连返回 200。重试推送前先确认这条链路恢复。

## 5. 可复用经验

- 报表页出现「分子分母对不上」时，先分清两件事：**统计口径覆盖哪些 kind** 与 **筛选条件是否把不覆盖的 kind 静默丢弃**。前者是设计，后者是体验缺陷，修法完全不同。
- 评审类功能的粒度一旦绑定 `session_id + turn_key`，任何没有会话的实体（表单型反馈、非问数页面反馈）天然不可评审；这类实体应在提交时就把会话标识写进自己的表，事后无法回填。
