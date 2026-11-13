---
id: "57ee466d-cf65-4938-a2d0-b78e04c7afaa"
title: "ChatBI 用户日志报表：Include internal users 勾选框联动计数（MR !542 / !1021 / !1022）"
description: "内部员工勾选框从纯布尔开关改为带计数联动：汇总接口新增 internalCount（复用 SESSION_FILTER 但强制 includeInternal=true 再叠加内部名单，否则 NOT EXISTS 使结果恒 0）；实测 8020 条内部员工会话 / 62 个 enabled 账号，Human 由 0→5；commit 2fbdf2fa8 / 69fa2f71 / b626d823；含 Accuracy Rate 恒 0% 的未决口径。"
status: "active"
created_at: "2026-11-13T19:12:45.375Z"
updated_at: "2026-11-13T19:12:45.375Z"
content_hash: "cc6115fa3d8f30fc903ada462d217b25462a8fcf154d49160c982eefc05af21d"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "src/views/Setting/reports/user-logs/components/FilterBar.vue"
  - "src/views/Setting/reports/user-logs/index.vue"
session_ids:
  - "mnemon-run-v1-181092be-d659-454a-b52d-2db42ace2f58"
memory_body_ids:
  []
---

# ChatBI 用户日志报表：Include internal users 勾选框联动计数（!542 / !1021 / !1022）

前置：**2b13c2e2-62ad-49c9-b0f2-5bb15637a132**（backfill 纳入自动评审 + 评审筛选候选值联动）、**acb35915-0667-44bf-a5e9-8dac79c502d8**（筛选返回空的排查）。本文只记录 2026-11-14 这一轮**新增**的「内部员工勾选框联动」，不重复前两篇的结论。

## 1. 用户指出的问题（原话）

上一轮交付后用户否定了「候选值未联动」的收尾说法："说明你 筛选没联动啊 内部员工的勾选框也是筛选项 你怎么不联动"。
即：`Include internal users`（`FilterBar.vue:938-946`，`v-model="filters.includeInternal"` + `@change="onFilterPicked"`）只是布尔开关，标签上没有任何数字，没有参与「筛选项与数据联动」的一致性要求。
背景事实：测试环境仅有的 5 条人工评审，对象全是内部员工会话，所以不勾此框时 `Review source = Human` 恒为 0。

## 2. 后端实现（关键非显然点）

汇总接口新增 `internalCount` = 「其它筛选条件不变时，勾上此框会让列表多出多少行」。

- 口径复用同一条 `SESSION_FILTER`，但**必须强制按 `includeInternal=true` 重新生成 SQL**：`ChatbiEventQueryMapper.java:872-874` 在 `includeInternal` 为空/false 时会拼上
  `AND NOT EXISTS (SELECT 1 FROM public.ssg_chatbi_internal_user iu WHERE iu.enabled = true AND lower(iu.itcode) = lower(user_id))`
  若直接拿带 `NOT EXISTS` 的 WHERE 再判 `EXISTS`，结果恒为 0。
- 在上一步基础上额外要求命中内部员工名单（`ChatbiEventReviewMapper.sessionInternalCount`，以 `lower(itcode) = lower(user_id)` + `iu.enabled = true` 关联）。

实测（测试库）：**内部员工会话 8020 条，enabled 内部账号 62 个**。

## 3. 前端

勾选框标签改为 `Include internal users (8020)`；勾上/取消时该行数字、列表、四个评审下拉（status / tag / reviewer / source）的候选值按同一套口径整体重算。`FilterBar.vue:403` 仍只在 `filters.includeInternal` 为真时带 `includeInternal=true` 参数；`ExportModal.vue:90` 的导出条件标签同步带上数字。

联动的可验证效果：勾上后 `Review source` 的 `Human` 由 0 → 5，`Reviewer` 下拉出现 `Shinee Shan5 Gao · 4` 与 `Xiangpeng XP4 Ren · 1`（就是用户看到的那 5 条）。

## 4. 交付状态

| 仓库/分支 | commit | MR |
|---|---|---|
| ssg-sales-ops `feature/admin-review-filters-admin` | `2fbdf2fa8` | !542 |
| ssg_sales_ops `feature/admin-review-filters-master` | `69fa2f71` | !1021 |
| ssg_sales_ops `feature/admin-review-filters-develop` | `b626d823` | !1022 |

三支均 `can_be_merged`；后端 `build_project` 通过、前端 `npm run type-check`（vue-tsc --build）exit 0。前端 4 files changed +16/−1，后端 2 files changed +42。

## 5. 仍待用户决定的统计口径

自动评审只对**失败轮次**产出结论，成功轮次一条都不评，因此生产 281 条评审全部是 WRONG，`Accuracy Rate` 恒等于 0%、无信息量。两条候选出路（用户尚未选择）：
1. 对成功轮次抽样判 CORRECT；
2. 把 Accuracy Rate 改为只展示「已评审中人工复核的准确率」。

另有一项历史未决：是否直接用 SQL 把 454 条应判 Wrong 的回填记录写入（需 `a_appconnect` 授权），以避免等发版。
