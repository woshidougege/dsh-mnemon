---
id: "4bc2253b-d4c9-43b4-a6d8-8c0c111caa3e"
title: "问题反馈页 Review source 取值修正（MANUAL→HUMAN）与评审四维只列有数据选项（MR !1039 / !1040 后续）"
description: "ssg_chatbi_question_review.review_source 实际只有 AUTO/HUMAN，原 MANUAL 写法导致永远 0 条（01f997a6 / d60eaaa9）；按用户要求把 Review status/tag/reviewer/source 四维对齐 user-logs onlyWithData，只列有数据项（cdac56e7 / ef428847）；本地 8082+8090 无头 Chrome 实测条数；Reviewer 姓名尾部粘连 itcode 数字为上游数据问题，未决。"
status: "active"
created_at: "2026-11-14T09:25:49.901Z"
updated_at: "2026-11-14T09:25:49.901Z"
content_hash: "5b0bb61cf361a0fba041abe7b524664eb61e84494bf8733693ffb87162f897ae"
source_paths:
  - "src/views/Setting/reports/usage-issue-feedback/index.vue"
session_ids:
  - "mnemon-run-v1-a44930c0-517d-4ae4-905d-3a9c3614a5df"
memory_body_ids:
  []
---

# 问题反馈页评审筛选：review_source 真实取值修正 + 四维「只列有数据的选项」

承接文档 `acae4a8d-59be-4ec8-b4ef-4a73d62d64af`（三页筛选下拉条数统一，MR !554 / !1039 / !1040）。本文只记录该分支后续两个提交的增量结论。

## 1. 真 bug：Review source 取值写成 `MANUAL`

- 位置：`src/views/Setting/reports/usage-issue-feedback/index.vue` 的 `REVIEW_SOURCE_FILTERS`（约 :119）。
- 原代码 `{ value: 'MANUAL', label: 'Manual' }`，但库里 `public.ssg_chatbi_question_review.review_source` **只有 `AUTO` / `HUMAN` 两个值**，因此选 "Manual" 永远筛出 0 条，且文案与问数记录页不一致。
- 修正为 `{ value: 'HUMAN', label: 'Human' }` + `{ value: 'AUTO', label: 'Auto rule' }`，并在上方留注释说明真实取值来源。
- 提交：master `01f997a6`（MR !1039）、develop `d60eaaa9`（MR !1040），各 4 insertions / 2 deletions，单文件。
- 实测库内分布：`AUTO` 734 条、`HUMAN` 5 条。所以多数时间范围下拉里只看到 `Auto rule`，不是 bug。

## 2. 用户决策：评审四维对齐 user logs 的 `onlyWithData`

用户原话（本轮）：`统一成user logs的逻辑哈， 然后第二个问题我没看懂。 第三我有个问题human啥意思，是别人的itcode.吗`

采取的口径：Issue & Request 页的 **Review status / Review tag / Reviewer / Review source** 四个下拉只列「当前筛选口径下真有数据」的选项，0 条不显示；做法等同 user-logs `src/views/Setting/reports/user-logs/components/FilterBar.vue:175-178` 的 `onlyWithData(list, counts)`。数据尚未返回时仍全列、只是不显示数字（count 函数返回空串 `''` 而非 0），避免下拉先闪一排 0。

- 实现：新增 `withReviewCounts(FILTERS, countFn)` 组合器，产出 `reviewStatusOptions` / `reviewTagOptions` / `reviewSourceOptions` 三个 computed（script 约 :158-160）；`reviewerOptions` 直接取 `reviewStats.byReviewer`（天然只有有数据的人）。
- 模板由 `v-for="option in REVIEW_STATUS_FILTERS"` 等改为 `reviewStatusOptions` / `reviewTagOptions` / `reviewerOptions` / `reviewSourceOptions`（约 :1051 / :1076 / :1102 / :1127）。
- 计数来源：`reviewStatusCount` 读 `reviewStats`（CORRECT/WRONG/PENDING），`reviewTagCount` 读 `reviewStats.byTag`（tag 大写匹配），`reviewSourceCount` 读 `reviewStats.bySource`（先 `map[value]` 再回退 `map[value.toLowerCase()]`）。非评审维度（Type / Band / L3-L5 / Team / Reason / Jira status）仍走 `facets` 接口的 `facetCount(dim, value)`，未返回时返回 `''`。
- 提交：master `cdac56e7`（MR !1039）、develop `ef428847`（MR !1040），1 file changed，+26 −7。

## 3. 本地联调验证（2026-11-14 实测）

- 后端 8082 跑的是含 !553 / !554 的后端代码，前端 vite 8090；无头 Chrome CDP 端口 9222 逐个点开下拉读取文案。
- 实测结果：Review status → `Wrong 6` / `Pending 36`；Review tag → `Definition mismatch 2` / `Wrong report 1` / `Display error 1` / `Other 2`；Reviewer → `Auto rule 6`；Review source → `Auto rule 6`（该时间范围 `HUMAN` 为 0，故上一版显示的 `Human 0` 现已不出现）。
- 截图在 `.tmp/shots/`：`D1-review-status.png`、`D2-review-tag.png`、`D3-reviewer.png`、`D4-review-source.png`，以及全景 `A1-data-inquiry-records.png` / `B1-feedback-tab.png` / `C1-issue-request.png`。
- 工作分支 `fix/feedback-facets-wire-dev` 的 upstream 已删除；本地 `.env.development` 与 `vite.config.ts` 保留联调改动未提交。

## 4. 概念澄清（避免再次误读）

| 筛选 | 含义 | 取值 |
|---|---|---|
| Reviewer | 具体是谁评的 | 真人 name+itcode，或 `Auto rule`（`system:auto` 系统账号） |
| Review source | 评审是怎么产生的 | 只有 `Human`（人工页面点评） / `Auto rule`（自动规则） |

`Human` 是评审来源类型，**不是** itcode。

## 5. 遗留未决

上游组织画像（LUDP 同步的人员数据）返回的 name 尾部粘连了工号数字，如 `{"itcode":"achen50","name":"Ashley Chen50"}`、`{"itcode":"bho5","name":"Bowie HO5"}`、`{"itcode":"chu10","name":"Cola Hu10"}`。不是本次改动引入，也不影响筛选（筛选用 itcode）。修法只能在展示层按「name 以 itcode 末尾数字结尾则截断」，改动的是别人的名字，用户尚未批准，暂不动。
