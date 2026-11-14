---
id: "acae4a8d-59be-4ec8-b4ef-4a73d62d64af"
title: "ChatBI 报表三页筛选下拉条数统一（MR !554 / !1039 / !1040）"
description: "Data Inquiry Records / Feedback / Issue & Request 三页筛选下拉条数写法统一；后端补齐 listPagePathCounts / listPageViewPathCounts 与 listJiraStatusCounts（未建单归 __no_ticket__）；!553 与 !554 需同批合并；前端 master/develop 已分叉（29 vs 662 独有文件）故 →develop 必须重拉分支。"
status: "active"
created_at: "2026-11-14T04:50:40.762Z"
updated_at: "2026-11-14T04:50:40.762Z"
content_hash: "cec08a88b44e8859cff91ef46a69057b970d150f974b3e44d63561f1dad93549"
source_paths:
  - "src/views/Setting/reports/user-logs/components/ManagerCombo.vue"
  - "src/views/Setting/reports/user-logs/components/FilterBar.vue"
  - "src/views/Setting/reports/user-logs/styles.less"
  - "src/views/Setting/reports/usage-issue-feedback/index.vue"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiFeedbackRecordMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java"
session_ids:
  - "mnemon-run-v1-c9faa0fc-518c-4962-ae14-8c6316989d1b"
memory_body_ids:
  []
---

# ChatBI 报表三页筛选下拉条数统一（MR !554 / !1039 / !1040）

承接 **bb23dcc6-51fa-4326-866a-cd69c6d75e00**（条数联动首轮，其中把 Page / Jira status 列为「仍未补条数的 6 个维度」）与 **8541407e-42d7-4863-afaa-bf889252fa65**（反馈中心 facets 与降级）。本文只记录把这两个缺口补上、并把三页下拉的条数写法统一这一增量。

## 1. 问题

「Setting 报表」下三个页面 **Data Inquiry Records / Feedback / Issue & Request** 的筛选下拉，条数有三套写法：有的只显示数字、有的写成 `12 rows`、Page 与 Jira status 下拉干脆没有数字；同一类下拉小字的外边距、等宽数字、长文案截断也不一致。

## 2. 前端改动（4 文件，+40/−25，commit `0d22443e` / `862bc9a9`）

- `src/views/Setting/reports/user-logs/components/ManagerCombo.vue`：User / L3-L5 下拉改为「左侧主文案 `Bowie HOS · bho5`，右侧纯数字」，汇报线下拉去掉 ` rows` 单位。
- `src/views/Setting/reports/user-logs/styles.less`：`.combo-option-meta` 与 `.ulr-opt-count` 统一成同一组数值（`margin-left`、`tabular-nums`、长文案截断）。
- `src/views/Setting/reports/user-logs/components/FilterBar.vue`：Page 下拉接 `pageCounts`、Jira status 下拉接 `jiraStatusCounts`。
- `src/views/Setting/reports/usage-issue-feedback/index.vue`：User 下拉右侧只显示条数；评审四维与 facet 条数在数据未就绪时返回空串，不再先渲染一个假的 0。
- 后端未上时 `pageCounts` / `jiraStatusCounts` 缺失 → 这两项自动不显示数字，不报错（前后端可分别合并）。

## 3. 后端改动（3 文件，+81，commit `dc2d0944f`）

- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java`：新增 `listPagePathCounts`、`listPageViewPathCounts`，与既有候选来源 `listPagePaths` / `listPageViewPathsForOptions` **同口径**。
- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiFeedbackRecordMapper.java`：新增 `listJiraStatusCounts`；未建单的反馈归入 `__no_ticket__` 桶。
- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java`：options 组装处下发 `pageCounts`（:1896 默认、:1903 `scope=pages` 时）与 `jiraStatusCounts`（:1898）；`normalizeRatingFilter`（:2148-2151）保持「反馈类型多选只保留 1 / -1，其余丢弃，null = 不限」的旧语义。

## 4. 验证（tst 环境 10.62.79.148 真实库探针）

- `mvn compile` 通过；`npm run type-check` 通过。
- `jiraStatusCounts = {Done=32, In Progress=2, New=1, Pending=2, Testing=5, __no_ticket__=28}`，合计 70 与反馈总数吻合。
- `pageCounts` 12 个 key，与 Page 候选一一对应：`/spark/gpssus=922`、`/insightHub=319`、`/spark/workspace=245`、`/customizedReport=68`、`/toolkits=93`、`/usageIssueFeedback=168` 等。
- 探针的 `query` 段单独失败：`NullPointerException: Cannot invoke "ChatbiQuestionClusterService.profiles()" because "this.questionClusterService" is null`（裸 main 探针未注入该 bean，非本次改动引入）。

## 5. MR 与合并依赖

| 仓库 | 分支 → 目标 | MR | commit |
|---|---|---|---|
| ssg_sales_ops（前端） | `fix/filter-option-counts-unify` → `master` | !1039 | `0d22443e` |
| ssg_sales_ops（前端） | `fix/filter-option-counts-unify-dev` → `develop` | !1040 | `862bc9a9` |
| ssg-sales-ops（后端） | `fix/filter-option-counts` → `admin-release` | !554 | `dc2d0944f` |

- 最初基于 master 建的前端 `fix/filter-option-counts-unify → develop` MR !1038 报 `cannot_be_merged` / `has_conflicts=True`（`changes_count=23`），已关闭；develop 侧改为从 `origin/develop` 重新拉分支重做，形成 !1040。三个 MR 现均 `can_be_merged`。
- **Feedback 页的 User / Band / Team 条数来自上一条后端 MR !553（`fix/feedback-option-counts` → `admin-release`）**，要让 Feedback 页签完整，!553 与 !554 需一起合；二者都改 `ChatbiEventReportService` 但改不同区域，不冲突。
- Issue & Request 页的 Type / Band / L3-L5 / Team / Reason / Jira status 条数走已合并的 `facets`（`selectFacetStats`），本次只统一了样式，数据口径未动。

## 6. 环境/流程事实

- **前端 `master` 与 `develop` 已严重分叉**：`master` 独有文件 29 个（`components.d.ts`、`src/router/index.ts`、`src/services/api.ts`、`src/views/IC/*` 等），`develop` 独有 662 个。因此**不能**把基于 `master` 的分支直接作为 `→ develop` 的 MR 源分支，必须从 `origin/develop` 重新拉分支再 cherry-pick / 重放改动。
- 后端编译期曾报 `ChatbiEventReportService.java:[2204,28] 无法从静态上下文引用非静态方法 toLong(java.lang.Object)`——静态方法里调用实例工具方法会被 maven-compiler-plugin 直接拦死，修完才通过。
- 建 MR 仍沿用 `.tmp/create-mr.ps1`（GitLab project 需 URL 编码、409 时回查已存在的 opened MR），与 bb23dcc6 记录一致。
