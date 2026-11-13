---
id: "e473a76e-9ef6-443f-b7e6-fcbf4f544574"
title: "ssg_sales_ops：master↔develop 三块一致性核对结论与 I&C 例外（!1018）"
description: "通知中心/反馈中心/Admin Control 三块在 master 与 develop 的逐文件核对结论：回放评审链路 5 个文件缺失及已开 !1018 对齐、I&C 菜单项属故意差异（master 无 /spark/ic 路由）、两支未合并的 case-file 抽屉重构会冲突、建议 CI 加 diff 守护。"
status: "active"
created_at: "2026-11-13T10:44:03.159Z"
updated_at: "2026-11-13T10:44:03.159Z"
content_hash: "7517ec65f984c2b137d4572e52ad7a33870ee8d8c13dd1a0ec60cb01250bf303"
source_paths:
  - "src/views/Setting/reports/history-replay/HistoryResultFeedback.vue"
  - "src/views/Setting/reports/history-replay/HistoryChatPanel.vue"
  - "src/views/Setting/reports/user-logs/components/modals/DetailModal.vue"
  - "src/views/Setting/reports/user-logs/index.vue"
  - "src/constants/feedbackMenuCatalog.ts"
  - "src/views/BiAdmin/moduleTargets.ts"
  - "src/router/index.ts"
session_ids:
  - "mnemon-run-v1-ed2e71a3-d79d-43b6-bcd8-279b93b9946f"
memory_body_ids:
  []
---

# ssg_sales_ops：master ↔ develop 三块一致性核对（通知 / 反馈 / Admin Control）

日期：2026-11-13。基线：origin/develop `92e92bc9`、origin/master `231a31fd`。

## 核对范围（"三块必须一致"）

1. 通知中心：`NotificationCenter`、`services/notification*.ts`、`reports/notification-settings`
2. 反馈中心：`views/UsageIssueFeedback`、`reports/usage-issue-feedback`、`services/usageIssueFeedback.ts`、`answerFeedback.ts`、`constants/usageIssueFeedback.ts`
3. Admin Control：`reports/**`、`settings/panels`、`registry.ts`、`adminMenu`、`adminScope`、`BiAdmin`、`AskPreview`

## 结论

- 通知中心：完全一致。
- 反馈中心：组件与 services 一致，仅 `src/constants/feedbackMenuCatalog.ts` 差 I&C 菜单项（该文件 master 侧 +10/-8 即此项）。
- Admin Control：**会话回放评审链路缺失**。develop 的 `d709076c feat(review): add review action to history answers` 整条链路 master 完全没有；另 `src/views/BiAdmin/moduleTargets.ts` 差 `ic: '/spark/ic'` 一行。

缺口文件（develop 有、master 无，均来自 `d709076c`）：

- `src/views/Setting/reports/history-replay/HistoryResultFeedback.vue` —— 每条回答旁 review 图标，`emit('review', { sessionId, turnId, question })`
- `src/views/Setting/reports/history-replay/HistoryChatPanel.vue` —— `review` 事件 + `questionAt(index)`（向前找最近一条 role==='user' 的消息作为问题原文）
- `src/views/Setting/reports/user-logs/components/modals/DetailModal.vue` —— 会话详情内嵌 `QuestionReviewDrawer`，`@saved` 向上 emit `review-saved`
- `src/views/Setting/reports/user-logs/index.vue` —— `review-saved` 后刷新列表与 Accuracy 卡片
- `src/assets/images/svg/review.svg` —— 图标资源

影响面差异不只是样式：develop 上能在会话回放里直接评审，master 只能从列表进抽屉。

## 已处理

分支 `feature/admin-review-consistency`（从 origin/master `231a31fd` 切出），commit `e7185a60`，把上述 9 个文件**直接取 develop 版逐字节对齐**（不是重新实现，避免再次分叉），MR !1018 → master。

验证：`git diff origin/develop -- <三块全部路径>` 除 I&C 外为空；`vue-tsc` 通过；eslint 0 error（61 warnings，多为历史 `no-explicit-any`）；`vite build` 通过。

## 已知的故意差异（对齐时不要带过去）

`constants/feedbackMenuCatalog.ts` 与 `BiAdmin/moduleTargets.ts` 中的 I&C 项：master 没有 `/spark/ic` 路由与页面（`src/router/index.ts` 里 `portfolio`、`portfolio/config`、`ic` 均只在 develop）。对齐后反馈页会出现打不开的选项；master 原代码注释写明「I&C 上生产后打开注释即可」。若要放开，须连同 I&C 页面一起带（约 23 个文件 + 1 条路由）。

## 未决事项

1. I&C 是否本期在 master 放开（等 I&C 上生产 or 现在把页面一起搬）。
2. `feature/question-review-drawer-master`（`08e1e292`）与 `feature/question-review-drawer-develop-latest`（`4c8ebfd5`）仍未合并，内容是把 `QuestionReviewDrawer` 重构成 case file（作者 renxp4，14:10），与 !1018 对齐后的抽屉版本会冲突，需先决定合并还是废弃其中一支。
3. 建议在 CI 加守护：三块路径下 master 与 develop 的 `git diff` 非空即报警，替代人工核对。

## 操作教训

MR 被合并后 GitLab 会删掉源分支，本地同名分支再 push 会得到 `! [rejected] ... (stale info)` / `fatal: couldn't find remote ref`；此时应从最新目标分支重切分支再接续提交，不要试图强行推送旧分支。
