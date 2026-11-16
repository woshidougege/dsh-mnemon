---
id: "c9e39da9-9d7e-4222-9082-b1cb80ef8447"
title: "ChatBI 评审第三状态（Not applicable / N/A）方案建议与实施清单（ssg-sales-ops）"
description: "把「无权限」「不在 scope」从 Wrong 的归因提升为独立第三状态的调研与方案：现状证据（枚举/DDL 两态约束）、推荐 NOT_APPLICABLE + reason 两级结构、5 处改动清单，以及 deleteLegacyAutoNoPermission 会误删新数据的风险。方案待用户拍板。"
status: "active"
created_at: "2026-11-16T07:25:31.241Z"
updated_at: "2026-11-16T07:25:31.241Z"
content_hash: "391439451af197f50420976c1549aa7bc6b1c745835404dfa4ba0f6ffb7775c0"
source_paths:
  - "ssg_sales_ops/src/views/Setting/reports/components/questionReview.ts"
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/api.ts"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiQuestionReviewService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiQuestionAutoReviewService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiQuestionAutoReviewMapper.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.119__Create_ssg_chatbi_question_review.sql"
session_ids:
  - "mnemon-run-v1-cab3a049-754f-4971-968c-bb5326eec273"
memory_body_ids:
  []
---

# ChatBI 评审第三状态（Not applicable / N/A）方案建议与实施清单（ssg-sales-ops）

来源：admin control 的问数人工评审（Question review）目前只有 Correct / Wrong 两态。用户指出实际还有第三种情况，且应把「无权限」从 Wrong 里移出来——原话（checkpoint ev.87）：「Correct / Wrong ，其实还有第三种状态， 分为无权限和 不在spoce里 ， 无权限现在不是在错误里吗 需要转过来。你推荐第三个叫啥呢」。

截至记录时，本方案**仅完成调研与建议，尚未经用户拍板、未改动任何代码**。

## 1. 现状证据（为什么"无权限"确实被算进 Wrong）

- 归因标签中 `NO_PERMISSION`（No permission）只是 Wrong 的一个子标签，不是独立结论：
  - 前端 `ssg_sales_ops/src/views/Setting/reports/components/questionReview.ts:18-25`（ATTRIBUTION_TAG_OPTIONS 六项：DEFINITION_MISMATCH / WRONG_REPORT / DATA_MISSING / NO_PERMISSION / DISPLAY_ERROR / OTHER）
  - 后端 `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiQuestionReviewService.java:30-32`（ATTRIBUTION_TAGS Set 同为六项）
- 结论枚举只有两态：`ChatbiQuestionReviewService.java:71-73` 校验 `status must be CORRECT or WRONG`；`ChatbiQuestionReviewService.java:62` 注释「判对时强制清空归因标签」，即标签只服务 Wrong。
- DDL 双重约束：`smartop-service/admin/src/main/resources/db/migration/V1.1.119__Create_ssg_chatbi_question_review.sql:29` 的 `CHECK (status IN ('CORRECT','WRONG'))`，以及 :30-33 的 attribution_tag CHECK（六个值）。
- 历史遗留：旧版自动规则把 access denied 判成 WRONG；规则已下线，现在靠 `ChatbiQuestionAutoReviewMapper.java:113-116` 的 `deleteLegacyAutoNoPermission()`（`DELETE ... WHERE review_source='AUTO' AND attribution_tag='NO_PERMISSION'`）在保存规则与补漏时清理，调用点见 `ChatbiQuestionAutoReviewService.java:69`（saveSetting）与 `:98`（runBackfill）。

即用户说的"转过来" = 把 NO_PERMISSION 从 Wrong 的归因升级为**独立结论**，并让"不在 scope"也归入该结论。

## 2. 推荐：第三状态 NOT_APPLICABLE，界面显示 Not applicable（chip 可缩写 N/A）

理由（decision + rationale）：
1. Correct / Wrong / N/A 是 AI/QA 评测量表的标准三态，管理员无需文档即可理解，不会与 Pending（未评审）或 Wrong 混淆。
2. 语义中立，同时容纳两类原因：无权限不是"答错"，不在 scope 也不是"答错"，共同点是「这一轮不该按对错评判」——正是 N/A 的定义。若硬叫 Out of scope，把"无权限"塞进去属语义妥协（问题在范围内，只是该用户没权限看数据）。
3. 准确率口径天然正确：Accuracy Rate = Correct / (Correct + Wrong)，N/A 不进分母；再补一个可评率 (Correct+Wrong)/Total，避免分母缩小却看不出来。

备选取舍：
- `Out of scope`：次选，最通俗，但"无权限"严格说不算 out of scope。
- `Excluded` / `Skipped` / `Not rated`：不推荐，读起来像"没评/跳过"，与 Pending 混淆。

## 3. 建议结构：状态 + 子原因两级

- 第三状态 `NOT_APPLICABLE` 必带 reason：`NO_PERMISSION`（No permission）/ `OUT_OF_SCOPE`（Not in scope）。
- 抽屉交互：Correct 不用选；Wrong 必须选错误归因；N/A 必须选 reason。
- 筛选条结论多一档；Wrong 的标签与 N/A 的 reason 共用一组候选（或分两列）；评审来源 HUMAN/AUTO 不变。

## 4. 实施会影响的位置（含一个必须处理的风险）

1. **DDL**：新迁移把 status CHECK 扩到含 `'NOT_APPLICABLE'`，attribution_tag CHECK 加 `'OUT_OF_SCOPE'`。存量 WRONG+NO_PERMISSION 老数据按项目既有规矩**手工改库**，不写进 SQL 迁移。
2. ⚠️ **自动清理会误删新数据**：`deleteLegacyAutoNoPermission` 语义是"删所有 AUTO + NO_PERMISSION"，一旦 N/A 复用该 reason 且自动评审会写入，则每次 saveSetting / runBackfill 都会清空这些行。需同时加时间边界（如 `reviewed_at < 上线时刻`）或改用独立字段区分旧版遗留与新语义。
3. **前端判定白名单**：`questionReview.ts:36-39` 的 `reviewStatusOf()` 只认 CORRECT|WRONG，未知值一律当 Pending；:42-44 `isReviewed()` 依赖它。不改则新状态显示为待评审、行内按钮回退成 "Review"。
4. **口径与计数**：`ChatbiQuestionReviewService.java:20` 的 Accuracy = CORRECT / 已评审数注释与汇总计算；QueryView 里 PENDING 由 `total - reviewed` 反推的逻辑；三个页面（问数记录 / 反馈记录 / 点踩列表）的筛选候选计数都要跟着改。
5. **自动规则**：现有 Review rules 只有 timeout / failed / no_result / needs_input 四个开关（标签归一在 `ChatbiQuestionAutoReviewService.java:138-143` normalizeTag），可新增 "No permission → N/A" 把 access denied 自动归档到第三状态，而不是像旧版那样判错。

## 5. 其他约束与后续

- 分支约定（沿用前序上下文，未在本次复核）：前端 `dev/renxp4`；后端 auth / admin 各自 release 分支，可先只给方案。
- 下一步待用户确认命名（Not applicable 或改 OUT_OF_SCOPE / EXCLUDED），确认后再动代码。
- 相关既有文档：Accuracy Rate 口径与数据模型 `38b9c4c0-0988-4206-9198-11f20587ae61`；评审改动留痕（V1.1.120 / MR !529/!981/!982）`6353a23e-5302-4b9c-9301-049e028565d3`。
