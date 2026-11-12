---
id: "ea0c4abe-9366-4b26-964c-e4b8149e3fad"
title: "ChatBI 人工评审 / Accuracy Rate 交付记录：后端 + 前端 master/develop 三支 MR（ssg-sales-ops）"
description: "人工评审与 Accuracy Rate 的落地记录：三支 MR（后端 !524、前端 master !975 / develop !974）、口径实现细节与关键代码行、评审行为约定、建表 V1.1.119、入口放管理员页的判断理由、验证到类型检查一级及 TST 未实跑的缺口。"
status: "active"
created_at: "2026-11-12T11:13:36.272Z"
updated_at: "2026-11-12T11:13:36.272Z"
content_hash: "92bcf24e3650691fc2ab7aea7436bae3bfaeb39e08a2a9762868cc7258a27c33"
source_paths:
  - "ssg_sales_ops-wt-review-master/src/views/Setting/reports/user-logs/components/views/QueryView.vue"
  - "ssg_sales_ops-wt-review-master/src/views/Setting/reports/components/QuestionReviewDrawer.vue"
  - "ssg_sales_ops-wt-review-master/src/views/Setting/reports/usage-issue-feedback/index.vue"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiQuestionReviewService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiQuestionReviewMapper.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.119__Create_ssg_chatbi_question_review.sql"
session_ids:
  - "mnemon-run-v1-014d6b78-3f8f-47ae-9425-e1b75134bbf9"
memory_body_ids:
  []
---

# ChatBI 问数「人工评审 / Accuracy Rate」交付记录（ssg-sales-ops，2026-11-12）

设计阶段的方案、概念拆分、建表 SQL 草案与 6 项待拍板决策见既有文档 **38b9c4c0-0988-4206-9198-11f20587ae61**（"ChatBI 报表页 Success Rate → Accuracy Rate（人工评审口径）设计方案与数据模型"）。本文只记**实际落地结果**：三支 MR、口径实现细节、实现期做的判断、验证到哪一步、以及未完成项。

状态：**三支 MR 已创建但未合并，等用户点合并**；TST 未实跑 SQL。

## 1. 交付清单

| 侧 | 仓库 | 分支 | commit | 目标分支（基点） | MR |
|---|---|---|---|---|---|
| 后端 | dataagent/ssg-sales-ops | feature/question-review-admin | `5c0636c9a` "feat(chatbi): 问数记录支持人工评审，准确率按已评审口径统计" | admin-release（`5ea04b79d`） | !524 https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/524 |
| 前端 master | dataagent/ssg_sales_ops | feature/question-review-master | `9c735d02` "feat(reports): 新增人工评审，问数主指标改为 Accuracy Rate" | master（`64663081`） | !975 https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/975 |
| 前端 develop | dataagent/ssg_sales_ops | feature/question-review-develop | `ea0bf53b` | develop（`f82b841d`） | !974 https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/974 |

- 前端两支 `git patch-id --stable` **完全一致** = `8d21c0ec7e6cd873c01303b29d3b009a9ed01afa`，证明 master / develop 是同一次改动（两边各 9 files, +1171/-3）。
- 后端 9 files, +659/-5。远程推送时有 LF→CRLF 警告（新建 Java/SQL 文件），无影响。

## 2. 指标口径（实现结果）

- 卡片标题 `Accuracy Rate`，主值 = **Correct ÷ Reviewed**（未评审不进分母）；副行 `Correct 114 / Reviewed 120`；再一行 `Coverage 24% · Pending 380`。
- 无任何评审时主值显示 `—`（不显示 0%），coverage 文案降级为 `No reviews yet` ——否则上线首日全员看到 0% 会被当成故障。
- 旧口径（agent 自报成功，`(total-failed)/total`）**逻辑保留、卡片默认隐藏**：`QueryView.vue` 里 `const showAgentSuccessRate = false` 一行即可恢复，领导要看时不用改逻辑。
- `/queries/summary` 响应新增 `review` 块：`correct / wrong / reviewed / pending / accuracyRate / coverageRate / byTag`。前端对 `review` 块缺失做 null 兜底（后端未升级或该时间窗无评审时不炸）。

关键实现位置（`ssg_sales_ops` 前端）：`src/views/Setting/reports/user-logs/components/views/QueryView.vue:170-227` —— `showAgentSuccessRate`(:173)、`reviewSummary`(:176)、`reviewedCount/correctCount/wrongCount/pendingCount`(:180-183)、`accuracyRate`(:185)、`coverageRate`(:188)、`accuracyText`(:192)、`coverageCaption`(:199)、`metrics`(:207)。

## 3. 评审闭环（行为约定）

- 抽屉内先给判定依据：问题原文、路由到的报表、最终结果、用户/时间；判定为 `Correct` / `Wrong` 二选一。
- 选 `Wrong` **必须给归因标签**：Definition mismatch / Wrong report / Data missing / No permission / Display error / Other。
- 评审人取自登录态；所有管理员都能评审、都能**改判**（PUT 是 upsert）。
- 撤销评审回到 Pending，但**讨论区留言不删**——改口径结论时理由留在原处。
- 前端 API：`reportApi.reviewGet / reviewSave / reviewClear / reviewCommentAdd`（见 `QuestionReviewDrawer.vue:128,142,171,193,208`）。
- 新组件：`src/views/Setting/reports/components/QuestionReviewDrawer.vue`、`ReviewStatusChip.vue`、`questionReview.ts`。

三处入口：
1. 问数记录（QueryView）
2. 反馈页签 `FeedbackView.vue`：Final result 右侧新增 `Review status / Attribution tag / Reviewer / Review` 四列
3. Issue & Request 点踩行（`src/views/Setting/reports/usage-issue-feedback/index.vue`，复用 `DislikeDetailDrawer`，:1123）

## 4. 数据模型

- `ssg_chatbi_question_review`：唯一键 `(session_id, turn_key)`；`turn_key` = `query_id`，缺失时回退 `row:<event id>`——与既有 `ssg_chatbi_failed_query_jira` 同口径。
- `ssg_chatbi_question_review_comment`：讨论区留言。
- 迁移 `V1.1.119__Create_ssg_chatbi_question_review.sql`，**只建表、不预置数据**："没评审"就是表里没那一行。
- 三个页面共用同一套 join。

## 5. 实现期做的判断（后续可被推翻，但理由要留）

- **评审按钮不放普通用户反馈页**：`/usageIssueFeedback` 是普通用户自己的反馈页，放"评审"会让非管理员点出 403。因此挪到管理员页 `Setting → Issue & Request`（点踩行才带 sessionId+turnId；问题单行按钮置灰）。若产品要的是普通用户页**只读展示**评审结果，需另加一版只读列。
- **旧成功率卡片用开关隐藏而非删除**，见 §2。

## 6. 验证状态与缺口

已验证：
- `npm run type-check`（`vue-tsc --build`）exit 0；
- `mvn -o -q -T 4 -pl smartop-service/admin -am -DskipTests compile` exit 0。

未做 / 缺口：
- **TST 未实跑 SQL**：本轮会话没有 `p_renxp4` 的库密码（上次用完已清掉），只做了结构核对 + 编译通过。上 TST 前应先跑 `V1.1.119` 迁移，再打开列表页确认 UNION 那 5 个新列不报错。
- 未开浏览器做页面渲染验证（按约定验证只到类型检查一级）。
- 经验：`vue-tsc --build` 在本轮出现过一次 120s 前台超时，最终 exit 0；后续这类全量类型检查直接作为后台 job 跑更稳。
- 若需要 TST 里有评审样本，先按文档 **4568440b-b6c8-47bd-a416-bce10601a830**（生产→TST 数据同步）灌 `ssg_chatbi_event` 数据，否则 Accuracy Rate 无数据可算。

## 7. 未闭合事项

1. 第三处入口的位置（管理员页 vs 普通用户只读页）需用户确认。
2. TST 迁移 + 页面实际渲染验证待做。
3. 三支 MR 均未合并，等用户操作。
