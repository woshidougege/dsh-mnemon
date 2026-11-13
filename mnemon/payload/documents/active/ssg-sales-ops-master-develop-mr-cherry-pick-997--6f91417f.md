---
id: "6f91417f-7e38-40d3-8d7e-e91d9a9ee8d2"
title: "ssg_sales_ops：分支从 master 切出导致 develop MR 假冲突——诊断与重切 cherry-pick 规程（!997→!1001）"
description: "双 merge-base 导致的假冲突诊断（QuestionReviewDrawer.vue 8 块冲突与 TaskGroups.vue 无关）、已实测的「从 origin/develop 重切 + cherry-pick」规程、就地解法与禁止手挑冲突块的注意事项、GitLab merge_status 缓存陷阱。"
status: "active"
created_at: "2026-11-13T06:13:21.952Z"
updated_at: "2026-11-13T06:13:21.952Z"
content_hash: "5bf88f8df496af04abe01af34a4609667306d5e0879ff3c834165c551ef26c90"
source_paths:
  - "src/views/Setting/reports/trino-sync/components/TaskGroups.vue"
session_ids:
  - "mnemon-run-v1-5c66eebf-e95c-4a72-8169-8aca14afea12"
memory_body_ids:
  []
---

# ssg_sales_ops：分支从 master 切出导致 develop MR「假冲突」——诊断与重切规程

前端仓库 `dataagent/ssg_sales_ops`（Vue3 + Element Plus）。记录一次实际诊断：MR 只改了 1 个文件却报冲突，根因是分支历史里夹带了 master，而不是他的代码有问题。状态：冲突已用重切方式绕开，新 MR !1001 已建、**未合并**。

## 1. 触发与现象

- MR !997：`tmp/scheduler-group-show-task-name-develop` → `develop`，内容只有 `src/views/Setting/reports/trino-sync/components/TaskGroups.vue` **+11/−1**（任务组行主链接改为 `taskName || targetTable`，不同时下方补灰色小字 `→ targetTable`，新增 `.group-row__table` 样式）。
- 13:41 创建时 `can_be_merged`；13:57 查仍是 `can_be_merged`；14:07 重算变成 `cannot_be_merged, has_conflicts=True`。
- 本地实测 merge 只有一个冲突文件：`UU src/views/Setting/reports/components/QuestionReviewDrawer.vue`（**8 个冲突块**）；`TaskGroups.vue` 完全不冲突。

## 2. 根因：双 merge-base

- 13:43:45 develop 合入 `b1589356 feat(review): redesign the review drawer as a case file`（就是重做 QuestionReviewDrawer.vue），13:49:33 经 `02cef543` 进入 develop。
- MR 分支从 **master** 拉出（head `894aa339` "Merge feature/scheduler-group-show-task-name into tmp (develop): keep develop's question-review files"），与 develop 是双头历史。
- `git merge-base --all HEAD origin/develop` 返回**两个**基：`b394a55bd`（develop 侧）与 `d9d8c21ff`（master 侧）。于是 git 把分支里那份**旧版抽屉**当成一次修改，去对撞 develop 刚做完的新版 → 假冲突。
- 反证：`git diff b394a55bd..HEAD` 只有 `TaskGroups.vue`；`git diff b394a55bd..origin/develop` 是 7 个文件（PbiRecommend、AskPreview drivers、QuestionReviewDrawer 等），两边文件集不相交。

## 3. 解法 A（推荐，已实测）

从最新 develop 重切，只 cherry-pick 纯功能提交：

```powershell
git fetch origin
git checkout -B tmp/scheduler-group-show-task-name-develop-v2 origin/develop   # e59221a2
git cherry-pick f4f23f64                                                       # feat(scheduler): show the task name in Task Groups rows
git push -u origin tmp/scheduler-group-show-task-name-develop-v2
```

结果：新 commit `a34f7659`，仍是 **1 file +11/−1**，无冲突；作者仍是 `zhangyong <zhangyong5@lenovo.com>`（cherry-pick 保留原作者）。新 MR **!1001** = https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1001 ，`opened / can_be_merged / changes=1 / conflicts=False`。原 !997 未动。

## 4. 解法 B（要保留原 MR 讨论时）

原分支本来就不该动那个抽屉文件，所以整文件取 develop 版本即可：

```powershell
git merge origin/develop
git checkout --theirs -- src/views/Setting/reports/components/QuestionReviewDrawer.vue
git add src/views/Setting/reports/components/QuestionReviewDrawer.vue
git commit
git push
```

或直接把 A 的提交覆盖推到原分支（原 MR 自动刷新、冲突消失）：
`git push --force-with-lease origin a34f7659:tmp/scheduler-group-show-task-name-develop`

**禁止**手工挑那 8 个冲突块——它们是 develop 新版「case file」抽屉的新增代码，手挑极易把抽屉改回旧版。

## 5. 操作要点

- GitLab 的 `merge_status` 有缓存：目标分支变了不会立刻重算，别信旧状态；用 `git merge-base --all` + 本地一次 merge 复现最可靠。
- 直接调 `PUT /projects/:id/merge_requests/:iid/merge` 在冲突/不可合并时返回 `400 {"message":"Merge request is not mergeable"}`，不是权限问题。
- 该仓库长期 master / develop 双支同改（同一改动在两条分支各留一份，如人工评审 !975 / !974 patch-id 一致，见文档 `ea0c4abe-9366-4b26-964c-e4b8149e3fad`）。**目标分支是 develop 的新分支必须从 `origin/develop` 切**，从 master 切会反复出现这种幽灵冲突。

## 6. 遗留与未做

- !1001 与 !997 都未合并，合并由用户点；未在 !997 上贴评论（用户未确认）。
- 未改作者代码。上一轮 review 的三条非阻塞建议均未采纳、留给作者：主行 `.target-link` 仍是 `ui-monospace` + `font-weight:650`（语义已变成人写的任务名，建议 sans，等宽留给下方表名）；缺 `max-width`/`text-overflow: ellipsis`/`title`（`taskName` 上限 120 字符，见 `src/views/Setting/reports/trino-sync/index.vue:1098`）；`v-if` 用 `!==` 严格比较，历史数据带尾空格时可能多渲一行（保存侧有 `trim()`，见 `index.vue:497`）。
- 该工程没配 CI，MR 上出现「no pipelines」是正常现象，不是失败。
