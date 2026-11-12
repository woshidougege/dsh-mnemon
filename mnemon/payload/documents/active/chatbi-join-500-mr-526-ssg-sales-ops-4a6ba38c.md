---
id: "4a6ba38c-451d-4a48-b625-05b6d75db712"
title: "ChatBI 人工评审 JOIN 二义性 500 修复（MR !526，ssg-sales-ops）"
description: "TST 部署后 /queries、/queries/summary、/feedback-records、/admin/downvoted-answers 全部 300：评审表 JOIN 进公共 FROM 导致 Postgres session_id/status 二义性；修法为派生表 + rv_ 前缀别名，MR !526（e4c15c4f9），含 TST 直连验证与回归缺口。"
status: "active"
created_at: "2026-11-12T13:17:07.256Z"
updated_at: "2026-11-12T13:17:07.256Z"
content_hash: "3cf0bdee514ce74479933d026f6d95e411d864144fdf51981f978655d6c50b99"
source_paths:
  - "ssg-sales-ops-wt-review-admin/smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "ssg-sales-ops-wt-review-admin/smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiFeedbackRecordMapper.java"
  - "ssg-sales-ops-wt-review-admin/smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/dao/AdminDownvotedAnswerMapper.java"
session_ids:
  - "mnemon-run-v1-51951d40-1585-4d5a-a002-7e9482103042"
memory_body_ids:
  []
---

# ChatBI 人工评审 JOIN 二义性 500 修复（MR !526，ssg-sales-ops）

承接文档 **ea0c4abe-9366-4b26-964c-e4b8149e3fad**（人工评审 / Accuracy Rate 交付记录）。那份文档的 §6 只做到"编译通过 + 结构核对"，TST 实跑是缺口；本次部署后 TST 立刻暴露出一个由第一版实现引入的 SQL bug，本记录是该 bug 的定位、修法与验证。

## 1. 现象

TST 部署后，以下四个接口全部返回 HTTP 200 但 body 为 `{"status":300,"msg":"INTERNAL_ERROR_PLEASE_RETRY","data":null,"success":false}`：

- `POST /queries`
- `POST /queries/summary`
- `/feedback-records`
- `GET /admin/downvoted-answers`

接口参数无关，四条都挂在同一段 SQL 上。对照：`/admin/feedback` 正常返回 rows（含 `l5Mgr` / `jiraStatusCategory` / `reasonCodes` 等列），说明服务本身与数据源都活着。

排查中顺带确认的两个非问题：
- `/api/admin/usage-issue-feedback/timeline`、`.../records` 返回 404 —— 这两个路径在当前版本里根本不存在（前端 `usageIssueFeedback.ts` 用的是 `API_PREFIX + '/admin/list'`、`'/admin/feedback'`、`'/admin/downvoted-answers'`），不是本次 bug。
- `psql` 客户端在 PATH 里为空，但 `psycopg` / `psycopg2` / `sqlalchemy` 均可用（`pg8000` 不可用），TST 直连验证走 Python 而非 psql。

## 2. 根因（Postgres 列名二义性）

第一版把评审表直接 JOIN 进查询的公共 FROM：

```sql
LEFT JOIN public.ssg_chatbi_question_review qr
       ON qr.session_id = ... AND qr.turn_key = ...
```

`public.ssg_chatbi_question_review` 的列（`session_id` / `turn_key` / `query_id` / `user_id` / `question` / `status` / `attribution_tag` / `remark` / `reviewer_itcode` / `reviewer_name` / `reviewed_at` / `created_at` / `updated_at`）与 `public.ssg_chatbi_event`、`agent.result_feedback` 大量重名，而会话查询里很多列是**不带表名**的引用。一旦两张同名列的表同时在 FROM 里，Postgres 直接报错：

```
AmbiguousColumn column reference "session_id" is ambiguous
AmbiguousColumn column reference "status" is ambiguous
```

在 TST 库（10.62.79.148 / ssg_ops_agent，个人账号 p_renxp4）上直跑该 JOIN 可原样复现；带完整限定名的 join_only 版本返回 1 行（78867），确认是限定名问题而非数据或权限问题。

## 3. 修法：派生表 + rv_ 前缀别名

三处评审 JOIN 统一改成先套一层派生表，把所有评审列重命名成 `rv_` 前缀，使业务列与评审列在结构上不可能重名：

```sql
LEFT JOIN (SELECT session_id AS rv_session_id, turn_key AS rv_turn_key,
                  status AS rv_status, attribution_tag AS rv_tag, remark AS rv_remark,
                  reviewer_itcode AS rv_reviewer_itcode, reviewer_name AS rv_reviewer_name,
                  reviewed_at AS rv_reviewed_at
             FROM public.ssg_chatbi_question_review) qr
       ON qr.rv_session_id = COALESCE(ssg_chatbi_event.session_id, '')
      AND qr.rv_turn_key  = COALESCE(NULLIF(btrim(ssg_chatbi_event.query_id), ''), 'row:' || ssg_chatbi_event.id)
```

`REVIEW_SELECT` 与 `sessionReviewStats` 同步改为读 `rv_*` 列；对外 JSON 列名不变（`reviewStatus` / `reviewTag` / `reviewerName` / `reviewedAt` / `reviewRemark` / `reviewTurnKey`），所以前端三个页面（问数记录、Like/Dislike、用户反馈点踩列表）不用改一行。

受影响文件：
- `ssg-sales-ops-wt-review-admin/smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java`（`REVIEW_SELECT_SESSION` :635-640、summary 里的 `sessionReviewStats` :1606-1610 附近）
- `ssg-sales-ops-wt-review-admin/smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiFeedbackRecordMapper.java`（`REVIEW_JOIN` / `REVIEW_SELECT`）
- `ssg-sales-ops-wt-review-admin/smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/dao/AdminDownvotedAnswerMapper.java`（点踩列表）

## 4. 验证

- `mvn -o -T 4 -pl smartop-service/admin -am -DskipTests compile` → exit 0
- 改后的三条 SQL（列表 / summary / 评审汇总）在 TST 库直接执行：全部正常返回，不再 ambiguous
- 点踩那条 JOIN 单独在 TST 库跑（`rf.rating = -1`，含 probe 会话与 `_feedback_retracted` 排除条件）：正常出 3 行（如 `b3d4a371-025c-408b-8686-bac352a6646e`）
- 未做：TST 重新部署后的接口回归（见 §6）

## 5. MR 与分支

- MR **!524**（第一版人工评审）已合入 `admin-release`，merge commit `5a43022ae`，含 `593b4154e feat(admin): 两条导出链路补人工评审列`、`7ed00fb22 feat(chatbi): 问数记录支持人工评审，准确率按已评审口径统计`。
- 因此本次修复单独走 MR **!526**（`feature/question-review-admin` → `admin-release`），只含一个 commit `e4c15c4f9 fix(admin): 评审 JOIN 改用 rv_ 前缀别名，修会话列表/汇总 500`。
- 提交时 Git 提示 `ChatbiEventQueryMapper.java` / `ChatbiFeedbackRecordMapper.java` / `AdminDownvotedAnswerMapper.java` 的 LF 将被 CRLF 替换——仓库既有换行策略，未处理。
- MR 由 `gitlab.xpaas.lenovo.com` API（`PRIVATE-TOKEN` 取自 git credential helper）创建：!524 `.../merge_requests/524`，!526 `.../merge_requests/526`；409 时按 source/target branch 复用已有 MR。

## 6. 未闭合

1. **TST 需重新部署** 才能在页面上看到修复；部署后要回归 `/queries`、`/queries/summary`、`/feedback-records`、`/admin/downvoted-answers` 四条接口。
2. MR !526 待合并。
3. 上次交付文档里的"TST 迁移 + 页面渲染验证"仍待做（本条修复不替代页面级验证）。

## 7. 可复用经验

- 往已经存在大量**不带表名**列引用的动态 SQL 里加 JOIN 时，必须先确认新表列名不与既有表重名；重名就用在 JOIN 处重命名的派生表（`rv_` 前缀），而不是逐个去给老列补限定名——后者改动面大且容易漏。
- 症状形态很迷惑：HTTP 200 + `status:300 INTERNAL_ERROR_PLEASE_RETRY`，看起来像环境/网关问题，实际是 SQL 报错被统一异常处理包了一层。批量 500 时优先抓服务端 SQL，别先怀疑环境。
- `agent.result_feedback.turn_id` 就是问数轮次 `query_id`，与 `ssg_chatbi_question_review.turn_key` 同口径——三处页面共用这套 join 的前提。
