---
id: "df3142b1-3c89-4e17-82c2-c764cf6084bb"
title: "ssg_sales_ops：master 缺漏盘查与回灌规程（反向打补丁法 + cherry-pick MR !1005）"
description: "master 从不合并 develop、靠双 MR 发版的仓库事实；用 git apply -R/正向检查+行命中率判定 master 是否已有某改动；他十几个 MR 的盘查结论（仅 !980/!988/!997/!998 真缺）、!825/!824 存疑；tmp/zhangyong-master-backfill 四次 cherry-pick 建 !1005（8 files +168/-10，vue-tsc 通过、vitest 缺失）。"
status: "active"
created_at: "2026-11-13T06:28:43.955Z"
updated_at: "2026-11-13T06:28:43.955Z"
content_hash: "392ca481d7aa5bfa13f89688dd8fde3254ddada9da823a6101995a7bf38df417"
source_paths:
  - "src/components/PbiRecommend/sparkDrivers.ts"
  - "src/views/AskPreview/ask-preview.css"
  - "src/views/AskPreview/components/DriversPanel.vue"
  - "src/views/Setting/reports/trino-sync/components/TaskGroups.vue"
session_ids:
  - "mnemon-run-v1-6daae815-c225-4f1d-ad40-7f71fe8f6828"
memory_body_ids:
  []
---

# ssg_sales_ops：master 缺漏盘查与回灌规程（反向打补丁法 + 一次 cherry-pick MR !1005）

前端仓库 `dataagent/ssg_sales_ops`（Vue3 + Element Plus，GitLab `gitlab.xpaas.lenovo.com`）。记录一次「同事的改动只进了 develop、master 缺了哪些」的盘查方法、结论，以及替他补的 master MR。

状态：**MR !1005 已建、`can_be_merged`、未合并**（2026-11-13）。!1001 / !1004 同样未合并。

相关：分支从 master 切出导致 develop MR 假冲突的诊断（!997→!1001、!998→!1004）已单独记在文档 `6f91417f-7e38-40d3-8d7e-e91d9a9ee8d2`，此处不重复，只引用其结论。

## 1. 仓库约定：双 MR，master 从不合并 develop

- 盘查时：`origin/master` = `709ba729`（2026-11-13 14:17），`origin/develop` = `058eb4e1`（14:18:54）。
- `develop` 领先 `master` **626** 个提交，`master` 领先 `develop` **20** 个。
- master 最近 8 个合并（`22952f1e`…`709ba729`）**全部**是 `*-master` 分支（`fix/export-xlsx-only-master`、`feature/question-review-*-master`、`feature/settings-nav-collapse-master` 等）；查不到任何 develop→master 的合并记录。
- 结论：这个项目**没有「定期把 develop 合回 master」的流程**，同一改动靠两条分支/两条 MR（`-master` / `-develop`）分别发。所以「develop 上有、master 上没有」是常态，不是事故；但不能假设 develop 的内容 master 都有——需要逐条盘。

## 2. 判定「master 是否已有某次改动」的方法

在 master 的临时 worktree 里对每个候选提交跑三件事，产出一张表：

1. **反向应用**：`git apply -R --check <patch>`（patch = 该功能提交的 diff）。成功 = master **已经有**这段改动。
2. **正向应用**：`git apply --check <patch>`。成功、反向失败 = master **确实缺**这段改动。
3. **行命中率**：把提交新增的行逐行在 master 的文件里 grep，数 n/10。用于区分「已被后续重构覆盖」与「真的缺」——两边都失败时（如新增行 100% 命中）就以命中率判定。

注意事项：
- 优先挑**纯功能提交**，不要挑 merge commit（例如 !988 挑 `ce7efbc0` 而非 merge `68a0ccba`），这样回灌后 master 历史保持线性，diff 也好核对。
- `git log` 显示的部分提交 parents 数量异常（`f4f23f64`、`3b3d3111` 的 parents 为 `d9d8c21f` = master 侧提交）——正是它们从 master 切出的证据，也解释了为什么它们在 master 上应用零摩擦、在 develop 上却冲突。

## 3. 盘查结论（他最近十几个 MR）

真正 master 缺的只有 4 处：

| develop MR | 提交 | 改动 | master | 回灌 |
|---|---|---|---|---|
| !980 | `4e248e6c` | spark-drivers 首问答案面板（scope 取自问题、与答案数字核对） | 缺 | 是 |
| !988 | `ce7efbc0`（功能提交） | What drove this? 支持 OH L4 (Hardware Install) 过滤 | 缺 | 是 |
| !997 | `f4f23f64` | Task Groups 行显示任务名、下方小字给目标表 | 缺 | 是 |
| !998 | `3b3d3111` | All breakdowns 长客户名换行（`ask-preview.css` +4） | 缺 | 是 |

已确认**不用回灌**（master 已有）：
- 反向应用即成功：!967 `4bc2d36e`、!969 `0dca6dc8`、!924、!919、!903、!880、!874。
- 新增行命中 master：!937 10/10、!904 10/10、!862 10/10、!858 10/10、!856 10/10、!832 10/10、!835 5/5、!857 9/10、!929 7/10。

**悬而未决**：!825 `a6581c82`（ask-drivers target extras）命中 0/2、!824 `97ed37f5`（ask-drivers tcv）命中 3/5，都是 **2026-11-03** 的老改动，动 drivers 核心逻辑。当时判断可能已被后续提交覆盖、也可能真要补，**故意没塞进 !1005**，要带需另开分支试应用后再给人看。

## 4. 回灌 MR !1005

`https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1005`

```powershell
git checkout -B tmp/zhangyong-master-backfill 709ba729          # origin/master
git cherry-pick 4e248e6c ce7efbc0 f4f23f64 3b3d3111             # 顺序：!980 !988 !997 !998
git push -u origin tmp/zhangyong-master-backfill
```

- 结果提交：`132d16fa`（!980）→ `a13f2eb3`（!988）→ `2b53bc07`（!997）→ `5df43ccf`（!998），**4 个 cherry-pick 全部无冲突**。
- 规模 8 files **+168/−10**；涉及文件：
  `src/components/PbiRecommend/__tests__/sparkDrivers.test.ts`、
  `src/components/PbiRecommend/PbiRecommendation.vue`、
  `src/components/PbiRecommend/sparkDrivers.ts`、
  `src/views/AskPreview/__tests__/drivers.test.ts`、
  `src/views/AskPreview/ask-preview.css`、
  `src/views/AskPreview/components/DriversPanel.vue`、
  `src/views/AskPreview/lib/drivers.ts`、
  `src/views/Setting/reports/trino-sync/components/TaskGroups.vue`。
- 作者保持原作者 `zhangyong <zhangyong5@lenovo.com>`（cherry-pick 默认保留）；目标分支 `master`，`opened / can_be_merged / conflicts=False / changes=8`，diff_refs `709ba729..5df43ccf`。

## 5. 验证到哪一步（含缺口）

- `vue-tsc --build` **exit=0**。master 与 develop 的 `package.json` devDependencies 完全一致，所以在临时 worktree `C:\Users\63588\lenovo\fe-wt-master-backport` 里用目录联接复用 develop 的依赖：
  `New-Item -ItemType Junction ...\fe-wt-master-backport\node_modules -> C:\Users\63588\lenovo\ssg_sales_ops\node_modules`。
- **单测跑不了**：仓库 `devDependencies` 里没有 vitest，`node .../node_modules/vitest/vitest.mjs` 报 `Error: Cannot find module ...vitest.mjs`（`VITEST_EXIT=1`）。`__tests__` 目录虽在仓库里，本地无法执行——与 develop 现状一致，不是本次回灌引入的问题。
- 相关脚本：`type-check: vue-tsc --build`，`build: npm run type-check && vite build --mode development`。

## 6. 待人拍板的动作

- !1005（master 回灌）等点合并。
- !825 / !824 是否也要回灌 master，需另开分支试应用。
- 尚未在 !998 上贴评论（原本建议他别再「从 master 切分支再手工搬 develop 文件」，改用 `git checkout -b x origin/develop`，必须从 master 切时最后 `git rebase origin/develop` 收口）；用户未点头前未贴。
