---
id: "ae67f8c5-491c-492b-97a7-a4fbe170ca95"
title: "ChatBI 评审 N/A 保存 400 定论：api.ts 两态遗留与双 toast 修复（ssg_sales_ops，2026-11-17）"
description: "收敛 88550f5d 的「根因未定论」：真因是 user-logs/api.ts:223 两态写法把 N/A 原因清空（UI canSave 通过、请求体被抹），叠加 request.ts 拦截器与抽屉 catch 双重弹窗吞掉后端 msg；含修复 diff、type-check、commit da79427d、master/develop 双 MR !1060/!1061 的 tmp 分支 patch-id 承接法、前后端工作区与 dev server 约定。"
status: "active"
created_at: "2026-11-17T02:40:03.816Z"
updated_at: "2026-11-17T02:40:03.816Z"
content_hash: "275814f1404d25d23fce64f6ad0b0d1f657d6743fa8c0c63022c072ae4df6fe1"
source_paths:
  - "src/views/Setting/reports/user-logs/api.ts"
  - "src/services/request.ts"
  - "src/views/Setting/reports/components/QuestionReviewDrawer.vue"
session_ids:
  - "mnemon-run-v1-9e4c123c-e322-430f-b544-197b8bd414a3"
memory_body_ids:
  []
---

# ChatBI 评审 N/A 保存 400 定论：api.ts 两态遗留抹空 attributionTag + 双 toast（2026-11-17）

本文收敛既有排查文档 **88550f5d-ceb8-4ddc-8d91-bafeec243c3f** 中「根因未最终定论」的状态，并修正其两条推论。三态语义与布局背景见 **e6544a1f-d30c-42ee-8f77-dead8deb331c**、**7714286d-1a8d-4dd3-9770-39139a751aae**，本文不重复。

## 1. 定论

- 现象：抽屉选 Not applicable + 原因 Out of scope → Save 返回 400；生产与本地同时复现。
- 现场证据（10:22:45 那次保存）：
  - 请求确实到达后端，231ms 返回，但 `ChatbiQuestionReviewService` 里 `log.info("问数评审已保存")` 一次都没打印。
  - tst 库中该轮次（sessionId `1f96e4c2…` / turnKey `5209bfbb…`）在 `ssg_chatbi_question_review` 与改动流水表**都是 0 行**。
  - ⇒ 请求进了 controller，卡在 `save()` 的校验被 400 挡回，未写库。
- **真因**：`src/views/Setting/reports/user-logs/api.ts:223` 仍是两态时代写法：

  ```ts
  attributionTag: payload.status === 'WRONG' ? payload.attributionTag || '' : '',
  ```

  三态改造只改了抽屉与 `questionReview.ts`，这一行漏改。选 N/A 时原因值在组件本地 ref 里（`canSave` 检查的就是它，能通过），但 `reviewSave` 组装请求体时把它抹成 `''`，后端抛 `attributionTag is required when status is NOT_APPLICABLE` → 400。
- 因此 88550f5d 中「走这份 UI 不可能发出 attributionTag 为空的 N/A 请求 → 只可能来自旧构建 / 其他调用方」的推论不成立：同一份源码在 UI 层校验通过、在中间 api 层丢字段。
- 该 bug 在 `master` / `develop` 上同样存在，与「本地也不行」一致。

## 2. 修复

- `api.ts:223` → `attributionTag: payload.status === 'CORRECT' ? '' : payload.attributionTag || ''`（Correct 强制清空；Wrong 与 N/A 各带自己的归因 / 原因）。
- 双 toast：`src/services/request.ts:98-104` 对所有非 401 错误先弹 `error.message`（axios 通用文案 `Request failed with status code 400`），`QuestionReviewDrawer.vue` 的 `save()` / `clearReview()` / `postComment()` 三个 catch 又各弹一次 → 同一次失败出现两条相同 toast，后端真实 msg 反被吞掉。
  - 拦截器：`message: error.response?.data?.msg || error.message`，条件同步改为 `error.message || error.response?.data?.msg`。
  - 抽屉：三处 catch 里的 `ElMessage.error(...)` 全部删除（成功提示保留），统一交给拦截器；该文件 `ElMessage.error` 出现次数 3 → 0。
- 验证：`npm run type-check`（`vue-tsc --build`）通过。

## 3. 交付（未合并，等用户点名）

- commit `da79427d` `fix(review): 修复 Not applicable 保存丢失原因与错误重复提示`，3 文件 11+/10−，分支 `dev/renxp4` 已推 origin。
- MR !1060 `dev/renxp4 → master`；MR !1061 `tmp/review-na-save-fix-develop → develop`；两条均 opened / can_be_merged / 3 文件，URL 前缀 `https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/`。
- `develop` 与 `master` 已分叉：`git merge-tree` 在 `src/constants/feedbackMenuCatalog.ts` 报 conflict（3 个 stage 条目）。故从 `origin/develop` 新建一次性分支 `tmp/review-na-save-fix-develop` 承接 cherry-pick，patch-id `742f30ebd13161c8bbc4aae1093192d6210e6907` 与 master 侧完全一致，证明两边内容等价。
- MR 创建脚本 `.tmp/create-mr.ps1`：`git credential fill` 取 `PRIVATE-TOKEN`，POST `/api/v4/projects/<URL 编码后的 path>/merge_requests`（`/` → `%2F`）。首次 POST 返回 404 `{"message":"404 Project Not Found"}`，修正 Repo 参数后成功；404 时先核对 Repo 与编码，不要先怀疑 token 权限。

## 4. 工作区布局（用户指定）

- 后端改在主工作区 `C:\Users\63588\lenovo\ssg-sales-ops`。
- 前端改在 worktree `C:\Users\63588\lenovo\ssg_sales_ops-wt-nareview`，**本地前端 dev server 也从该 worktree 启动**（VITE 7.2.4，`http://localhost:80/`；后端 8082 跑在 IDEA 里），这样改完能立刻在本地验证。
- 该 worktree 的 `vite.config.ts` 有未提交的本地改动（如端口 / proxy），切分支时用 `git stash` 保护、收尾再恢复，不要提交进 MR。

## 5. 教训

- 同一字段有三层「清理」逻辑时（组件 `canSave` 校验 / `api.ts` 组装请求体 / 后端校验），只改组件与后端仍会失败——中间层会用旧条件把值抹空。两态改三态这类扩展，必须 grep 全部 `=== 'WRONG'` 型分支，而不只改可见 UI。
- 重复弹窗来自「拦截器已统一提示 + 业务 catch 再提示」；让拦截器优先展示 `response.data.msg` 比在每个 catch 里解析更可靠，也避免后端真实原因被 axios 通用文案覆盖。
- 一次失败对应两条不同请求的错误假设很常见；先用后端日志有无业务 info 行 + 目标表是否 0 行来区分「请求没到」和「到了被校验挡回」，比读前端 toast 便宜得多。
- 88550f5d 的「线上 chunk 特征串核验构建新旧」方法仍有效，但当时据此把问题归到构建差异，忽略了同一仓库内 api 层的独立逻辑。
