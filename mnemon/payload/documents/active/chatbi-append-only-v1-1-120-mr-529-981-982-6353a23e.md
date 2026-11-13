---
id: "6353a23e-5302-4b9c-9301-049e028565d3"
title: "ChatBI 人工评审改动留痕（append-only 审计流水）实施记录：V1.1.120 + MR !529/!981/!982"
description: "评审被其他管理员改判/撤销时原先不留痕（upsert 覆盖 reviewer、DELETE 删行）；本次新增 append-only 流水表 ssg_chatbi_question_review_log（迁移 V1.1.120）、仅在结论/归因/备注真变化时写 SAVE、撤销写 CLEAR 并记录操作人，GET /queries/review 多返回 history 并在抽屉加 History 区块，三支 MR !529/!981/!982，含编译与 tst 回滚验证、部署注意事项。"
status: "active"
created_at: "2026-11-13T02:23:29.336Z"
updated_at: "2026-11-13T02:23:29.336Z"
content_hash: "b44ad1193d52951ccf7a446b1759bf42c8d6c2180202cfc3c0c27492c8f45ad8"
source_paths:
  - "ssg-sales-ops-wt-review-admin/smartop-service/admin/src/main/resources/db/migration/V1.1.120__Create_ssg_chatbi_question_review_log.sql"
  - "ssg-sales-ops-wt-review-admin/smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/controller/ChatbiEventReportController.java"
  - "ssg-sales-ops-wt-review-admin/smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiQuestionReviewMapper.java"
  - "ssg-sales-ops-wt-review-admin/smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiQuestionReviewService.java"
  - "ssg_sales_ops-wt-review-master/src/views/Setting/reports/components/QuestionReviewDrawer.vue"
  - "ssg_sales_ops-wt-review-master/src/views/Setting/reports/user-logs/api.ts"
  - "ssg_sales_ops-wt-review-master/src/views/Setting/reports/components/questionReview.ts"
session_ids:
  - "mnemon-run-v1-dd111372-1b8e-46cc-9b7d-4d10103887c2"
memory_body_ids:
  []
---

# ChatBI 人工评审改动留痕（append-only 审计流水）实施记录

关联既有文档：
- 设计方案 **38b9c4c0-0988-4206-9198-11f20587ae61**（Success Rate → Accuracy Rate 方案与数据模型；其中把 `ssg_chatbi_question_review_log` 列为 Phase 3 待做项）
- 首版交付 **ea0c4abe-9366-4b26-964c-e4b8149e3fad**（MR !524 / !975 / !974）
- TST 500 修复 **4a6ba38c-451d-4a48-b625-05b6d75db712**（MR !526，rv_ 前缀别名）

## 1. 起因与结论

用户问：「其他管理员不是能修改 review 的结果吗，修改的所有记录保存了吗」。

**结论：没有。** 首版 `ssg_chatbi_question_review` 一行 = 一个问题的最新结论：
- 再保存一次就是 upsert 覆盖，`reviewer_itcode` / `reviewed_at` 一起被换成最后一个人 —— 「A 判 Correct，B 改成 Wrong」事后只剩 B 和 Wrong，A 那版查不到；
- 撤销接口直接 `DELETE` 删行，连「谁撤的」都不记（`clear` 未传登录人）；
- 讨论区是手写留言，语义上不能当审计日志。

本次补上 **append-only 改动流水**，不修改评审主表语义。

## 2. 数据层

新迁移 `smartop-service/admin/src/main/resources/db/migration/V1.1.120__Create_ssg_chatbi_question_review_log.sql`（只建表，不塞数据；编号接 `V1.1.119__Create_ssg_chatbi_question_review.sql`）：

| 列 | 说明 |
|---|---|
| `action` | `SAVE` 保存 / `CLEAR` 撤销 |
| `from_status` / `from_tag` / `from_remark` | 改前；首次评审为 NULL |
| `to_status` / `to_tag` / `to_remark` | 改后；撤销为 NULL |
| `operator_itcode` / `operator_name` / `created_at` | 谁、什么时候 |

索引 `(session_id, turn_key, created_at DESC, id DESC)`：按评审主键取历史即走该索引。

## 3. 写入时机（关键判断）

- **SAVE**：先读旧结论 → upsert → **只有结论/归因/备注真的变了**才写一条流水。重复点保存不产生噪音，避免历史被刷屏。
- **CLEAR**：先读旧结论 → 删行 → 写一条 CLEAR，保留「谁把哪条结论撤了、撤之前是什么」；撤销接口现在把当前登录人传下去（补上首版缺口）。
- 写流水是评审保存/撤销的附带动作，不改接口语义；追加而非覆盖，因此历史不可被后续操作篡改。

## 4. 展示

- `GET /queries/review` 响应多返回 `history`（与 `review`、`comments` 平级）。
- 抽屉 `QuestionReviewDrawer.vue` 在 Review 与 Discussion 之间新增 History 区块，按时间正序，文案规则：
  - 首次评审：`Reviewed as Wrong · Data missing`
  - 改判：`Wrong · Data missing → Correct`
  - 撤销：`Review cleared (was Correct)`
  - 只改备注、结论没动：`Remark updated`
- 后端 `reviewed_at` 等时间字段仍按本地时间格式化为 `'YYYY-MM-DD HH:MM:SS'`，前端不二次转换（沿用 `questionReview.ts` 的 `reviewedAtText`）。

## 5. 交付清单

| 侧 | 仓库 / 工作树 | 分支 | commit | 目标 | MR |
|---|---|---|---|---|---|
| 后端 | dataagent/ssg-sales-ops（`ssg-sales-ops-wt-review-admin`） | `feature/question-review-audit`（跟踪 origin/admin-release，基点 `6e783428b`） | `89de0cf2c` feat(admin): record question review change history（4 files, +143/-6） | admin-release | !529 https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/529 |
| 前端 develop | dataagent/ssg_sales_ops | `feature/question-review-develop` | `0dc036fd` feat(reports): show question review change history（2 files, +111/-3） | develop | !981 https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/981 |
| 前端 master | dataagent/ssg_sales_ops（`ssg_sales_ops-wt-review-master`） | `feature/question-review-master` | `16124a83` 同标题 | master | !982 https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/982 |

- 前端两支 `git patch-id --stable` 一致 = `70c066d077b8e9c54cdccae47cd88f0798a16c14`，证明 master / develop 为同一次改动（做法：在 master 工作树里改完，再 `cp` 两个文件到 develop 工作树，避免手抄漂移）。
- 前端改动范围仅 2 文件：`src/views/Setting/reports/user-logs/api.ts`（`QuestionReviewDetail` 加 `history` 字段）与 `src/views/Setting/reports/components/QuestionReviewDrawer.vue`。
- 相关接口锚点：`smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/controller/ChatbiEventReportController.java:716`（GET review）、`:739`（PUT save）、`:761`（DELETE clear）、`:785`（POST comments）；前端 `user-logs/api.ts`：`QuestionReview`(:92)、`QuestionReviewComment`(:106)、`QuestionReviewDetail`(:115)、`QuestionReviewSavePayload`(:121)、`reviewGet`(:158)、`reviewSave`(:161)、`reviewClear`(:170)、`reviewCommentAdd`(:173)。
- 三页共用的常量/文案仍在 `src/views/Setting/reports/components/questionReview.ts`（`ATTRIBUTION_TAG_OPTIONS`、`buildReviewTarget`），本次未改口径。

## 6. 验证到哪一步（以及缺口）

已验证：
- `mvn -o -T 4 -pl smartop-service/admin -am -DskipTests compile` 通过（在最新 `origin/admin-release` = `6e783428b` 上执行，含 revert 启动诊断两次提交）。
- V1.1.120 在 **tst 库事务内**实跑：建表、约束、索引、列注释正常，插入一条 SAVE 流水并回读（`1 SAVE null->WRONG Ren 2026-11-13 10:12:14`，评审主表 4 行不受影响），随后 rollback，库里不留数据。
- 前端 `vue-tsc --build` 通过；两支 patch-id 一致。

缺口 / 未做：
- 未在真实服务里端到端点过「改判 → History 出现两行」；未跑后端单测（编译级验证）。
- History 区块依赖后端新字段：**前端先上线不报错**，只是显示 `No changes yet`；后端先上则前端不展示。两侧无强耦合，可分别合并。

## 7. 部署注意

- 部署后 Flyway 自动建表，**不需要手工执行 DDL**。
- 流水表按追加增长，没有清理策略；按问数量级估算量级很小，暂不做归档。

## 8. 环境/工具教训

- 本机 `psql` 不在 PATH，且当前 runtime 的 Python **没有** `psycopg` / `psycopg2`（`ModuleNotFoundError`）——想直连 tst 校验 DDL 时，走 Python 会在 import 处就失败（与文档 4a6ba38c 记录的「psycopg 可用」不同环境，勿照抄结论）。
- 可用替代：本机有 JDK（`javac` 在 `C:\Program Files\Common Files\Oracle\Java\javapath`）与 PostgreSQL JDBC 驱动 `42.2.11` / `42.5.4`，写一个临时 Java 类连库 + `setAutoCommit(false)` + 回读 + `rollback()` 是最省事的 DDL 验证路径。
