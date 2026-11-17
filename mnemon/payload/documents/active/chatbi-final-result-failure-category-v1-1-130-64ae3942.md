---
id: "64ae3942-bdb8-4694-bb7e-9ef24b218c19"
title: "ChatBI 自动评审规则改为 (Final result × Failure category) 可配 + V1.1.130 重建：交付记录"
description: "规则从写死 8 条件改为按真实字段二元组配置、候选值取近 180 天去重(16 组)；取消空表回退(空表=不评审)与 Add suggested(14) 必点；V1.1.130 须重启 flyway、tst 旧表 0 行；注解 SQL 复用 XML 转义常量的 &lt;/&gt; 还原坑；后端 f153a7b3c、前端 ed274278，MR 待指令。"
status: "active"
created_at: "2026-11-17T04:44:01.234Z"
updated_at: "2026-11-17T04:44:01.234Z"
content_hash: "c7a986a0fc710380567ccd903d7b798ff6716a2e915b09ebe968940e89a61ad6"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/controller/ChatbiEventReportController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiAutoReviewRuleMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiQuestionAutoReviewMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiAutoReviewRuleService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiQuestionAutoReviewService.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.130__Rework_chatbi_auto_review_rule.sql"
session_ids:
  - "mnemon-run-v1-3a5503d4-18b9-444d-9971-0036e7ee7e43"
memory_body_ids:
  []
---

# ChatBI 自动评审规则：按真实字段值配置（V1.1.130 交付记录，2026-11-17）

状态：前后端均已实现，后端 `mvn -o -T 4` 15 个模块 BUILD SUCCESS，本地前端 200 / 后端 200；**MR 未提交，等用户指令**。本记录**取代** `3105b760-9de6-4d10-8806-8ec777d33b8e`（V1.1.129 的「8 个写死条件 + 空表回退代码默认值」方案）；判定常量的生产口径与行号速查见 `2b13c2e2-62ad-49c9-b0f2-5bb15637a132`。

## 1. 规则模型：从「8 个写死条件」改为「(Final result, Failure category) 二元组」

- 旧：条件（`timeout` / `failed_report` / `failed_definition` / `failed_display` / `failed_other` / `no_result` / `needs_input` / `permission_denied`）由代码预先定义，取值口径与界面筛选用的一套不一致。
- 新：**一行规则 = 一个 Final result + 一个 Failure category**。取值不预先定义，而是从 DB 里取近 180 天真实出现过的组合去重（当时共 **16 组**）。判不判、怎么判完全由规则表决定，Java 侧按 `(sessionResult, failureCategory)` 查表。

## 2. 两个必须知道的行为变更

1. **取消「表为空时用代码默认值兜底」**：规则表为空 = 不自动评审（这样才叫规则可配）。代价：**重启后端后第一件事必须去规则页点 `Add suggested (14)`**，否则新问答全部停在 Pending。这是相对 V1.1.129 方案的反转，旧方案"空表与改造前等价"已作废。
2. `Add suggested (14)` 是按现有数据分布整理的整套推荐规则（超时、业务校验失败判 Wrong，无权限判 Not applicable…）。14 条 ≠ 16 组：只收录建议保留的组合。

## 3. 迁移与运维前置（不照做页面直接报错）

- 新增 `smartop-service/admin/src/main/resources/db/migration/V1.1.130__Rework_chatbi_auto_review_rule.sql`，重建规则表，新列含 `session_result` / `failure_category`。
- flyway 只在应用启动时执行：**不重启后端，规则页报 `column session_result does not exist`**（tst 当前仍是旧的 8 条件结构）。tst 上旧表 0 行，重建无数据损失。
- 迁移号教训（沿用 V1.1.129 那次）：同目录加迁移前先 `git fetch` 核对 `admin-release` 上 `V1.1.12x/13x` 的占用，避免重复版本号。

## 4. 关键实现点与坑

- 候选 SQL 复用列表页口径并多带两列（`smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiQuestionAutoReviewMapper.java` `CANDIDATE_COLUMNS`）：`SESSION_RESULT_SQL` = `ChatbiEventQueryMapper.FINAL_SUCCESS_SQL` ? 'Success' : 'Failed'；`FAILURE_CATEGORY_SQL` = `STATUS_CATEGORY_SQL`，别名 `sessionResult` / `failureCategory`。
- **XML 转义坑**：`ChatbiEventQueryMapper.STATUS_CATEGORY_SQL` / `RECOVERED_SQL` 在 MyBatis `<script>` 里使用，尖括号写作 `&lt;`/`&gt;`（如 `bad.started_at &lt; ok.started_at`）；注解 SQL 不经过 XML 解析，必须 `.replace("&lt;","<").replace("&gt;",">")`，否则 PG 收到 `&lt;` 直接语法错。
- 候选范围不再写死（`CANDIDATE_WHERE`）：`event_name='query_session'` + 非空 `session_id` + `PROBE_RUN_EXCLUDED_SQL`（以 `public.ssg_ai_probe_run.session_id` 为准）+ `COALESCE(rs.auto_review_enabled, TRUE)` + `NOT EXISTS` 已有评审行 + `NOT EXISTS` dismissal。多扫几行无结论轮次是可接受的；写死范围会让管理员新加的成功类规则永远不生效。
- 批量与实时必须同源：`selectCandidates(from,to)` 与 `selectCandidateForTurn(sessionId, turnKey)`(LIMIT 1) 共用 `CANDIDATE_COLUMNS + CANDIDATE_WHERE`，否则会出现「页面点补漏有结论、实时触发却没结论」两套口径。
- 轮次主键 `TURN_KEY_SQL` = `query_id` → 回退 `row:<event id>`；`STATUS_CATEGORY_SQL` 里 LUDP 权限不足优先判定，必须回查 `agent.tool_audit_logs`（execute 工具 succeeded 但业务返回 `ludp_permission_denied`，单看 `failed_step` 永远识别不出来）。
- `NUM_*` 系列数字属性统一先做 `~ '^[0-9]+$'` 正则再 `::numeric`，脏数据按 0 处理（否则整个看板 500）。
- 旧「无权限判错」自动结论的清理：`UPDATE ... SET status='NOT_APPLICABLE' WHERE review_source='AUTO' AND status='WRONG'`。必须限定 `status='WRONG'`：NO_PERMISSION 现在也是合法 N/A 原因，不限定会把已改判好的行反复重写。改判而非删除，避免把轮次打回待评审。
- 状态分类只读事件自身 `properties.failed_reason/failed_step`，不再取 agent 审计里该会话「最早一条失败」（session 级口径会把上一轮失败串到当前行）。
- 接口 `/queries/review/rules` 返回动态候选值与建议规则；改动集中在 `chatbi/dao/ChatbiAutoReviewRuleMapper.java`、`chatbi/service/ChatbiAutoReviewRuleService.java`、`chatbi/service/ChatbiQuestionAutoReviewService.java`、`chatbi/controller/ChatbiEventReportController.java`。

## 5. 提交与本地状态

- 后端分支 `feature/review-rules-event-admin`：`f153a7b3c` feat(review): 自动评审规则改成按 Final result 与 Failure category 配置。
- 前端分支 `dev/renxp4`：`ed274278` feat(feedback): 评审规则页支持按真实字段值增删规则（另有未提交的 `vite.config.ts`）。
- 本地前端 http://localhost/ 、后端 8082 均在运行；预览图 `C:\\Users\\63588\\lenovo\\ssg_sales_ops-wt-nareview\\.tmp\\rr-rules-preview2.png`。

## 6. 页面形态

- `Add rule` 弹窗：先选 Final result 再选 Failure category，**只列出还没配过、且近 180 天确实出现过的组合**，保证一个组合只出现一次。
- 每条规则可改结论、开关、✕ 删除；卡片上「9 in 180 days」是该组合最近命中量，配置时能直接看出轻重。

## 7. 待办

- 用户确认后提 MR（后端 + 前端两笔）。
- 重启后端让 flyway 建 V1.1.130，随后立刻 `Add suggested (14)`，验证新问答不再停在 Pending。
