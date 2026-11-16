---
id: "3061db0b-a4fe-469f-a461-91792b307add"
title: "ssg_sales_ops：项目分支文档与本地 skill 的流程冲突——MR 实证分析与 AGENTS.md 重写方案"
description: "docs/gitlab-branch-workflow.md 与本地 skill 在分支模型/一致性校验/冲突处理上的 4 点相互冲突；约 100 个已合并 MR 的实证（develop 领先 master 667、master 超前 37、两位同事漏提/同分支双 MR、!1041/!1043 顺序反、backfill MR !1005）；根因 4 条与「AGENTS.md 承载概览+双分支规则、docs 降为存根」方案；双分支 vs 单分支待用户拍板，未落地。"
status: "active"
created_at: "2026-11-16T02:03:49.199Z"
updated_at: "2026-11-16T02:03:49.199Z"
content_hash: "bce23472ebd67693b4431a595d4dfaedb0fd12eaea4d9133a7bedffd6fbefb9c"
source_paths:
  - "AGENTS.md"
  - "CLAUDE.md"
  - "docs/gitlab-branch-workflow.md"
  - "README.md"
session_ids:
  - "mnemon-run-v1-f8edffe4-79af-43ec-872e-5a3068897627"
memory_body_ids:
  []
---

# ssg_sales_ops：项目分支文档与本地 skill 的流程冲突——MR 实证分析与 AGENTS.md 重写方案

状态：**诊断完成，改造未执行**（2026-11-16）。用户尚未拍板分支模型（双分支 vs 单分支），仓库内代码与 MD 均未改动。

## 0. 触发与范围

- 用户决定：本仓库只维护 `AGENTS.md` + `CLAUDE.md` 两个规则文件，"能覆盖大部分 harness 就行"（覆盖矩阵与文件盘点见文档 `e09863d9-d9a9-4086-8c3c-75ef110487a7`）。这条范围决策排除了新增 `GEMINI.md`、`.github/copilot-instructions.md` 等薄壳的路线。
- 用户怀疑"同事提交的 MR 有问题"的根源在项目 MD，要求把**本地 skill** 与项目文档逐条比对。
- 本地 skill：`ssg-sales-ops-branch-workflow`（156 行；仅适用前端仓库 `dataagent/ssg_sales_ops.git`，后端 `dataagent/ssg-sales-ops.git` 不适用，且 skill 明确后端是 `auth-release` + `admin-release` 双分支）。

## 1. 三份规则的核心冲突（行号均指 `docs/gitlab-branch-workflow.md`）

| 规则点 | 项目 `AGENTS.md` + `docs/gitlab-branch-workflow.md` | 本地 skill |
|---|---|---|
| 分支数量 | 一个变更一条分支，:35「不要拆成多个分支、多个 MR」 | 必须**两个独立分支**，禁止只建一个共用分支 |
| MR 怎么提 | :88「两个阶段是两个独立 MR，共用同一个源分支」 | 两个分支各提一个 MR，分别对应 master、develop |
| 命名 | `feature/<topic>`、`fix/<topic>`（:49） | `feature/<topic>-master`、`feature/<topic>-develop` |
| 两边一致性 | 不校验（同一分支天然一致） | cherry-pick + `git patch-id --stable` 必须相等 |
| 冲突处理 | 从目标分支拉**临时分支**，把功能分支合进去（:118-127），MR 合并后删除临时分支 | 在当前 MR 分支上 `git merge origin/<自己的目标分支>` |

加重问题的两点：

- `AGENTS.md`（第 5 行附近）**强制要求**执行任何分支/MR/冲突操作前"必须先完整读取"那份错文档，等于把错误流程制度化，而 skill 完全没被引用。
- `docs/gitlab-branch-workflow.md:12` 有过时信息：说后端"使用单分支 `admin-release`"，实际是 `auth-release` + `admin-release` 两个。

## 2. 实证：团队实际在跑双分支，且已漂移

只读 GitLab API 拉最近约 100 个已合并 MR（!969–!1043）按作者/target 统计：

| 作者 | 提 master | 提 develop | 行为特征 |
|---|---|---|---|
| renxp4（用户本人） | 39 | 40 | 双分支成对，最规范 |
| yangxu27 | 9 | 5 | 偏 master，且**同源分支提双 MR** |
| zhangyong5 | **0** | 6 | **只进 develop，从不提 master** |
| wangjj63 | 0 | 1 | 替别人提（`feature/gps-latest-fix` 的 develop 侧） |

三条硬证据：

1. **`origin/develop` 领先 `origin/master` 667 个提交，同时 `origin/master` 又有 37 个 develop 没有的提交。** 两边已完全分叉，master 不是 develop 的子集，"同一份改动两边都有"无法自动保证。这也解释了 6f91417f 文档记录的"从 master 切出分支却报 develop 冲突"假冲突机理（双 merge-base）。
2. **漏提 master 靠事后补**：MR !1005 `tmp/zhangyong-master-backfill -> master`，标题「chore(master): backfill drivers & scheduler fixes that only …」，该分支实际只含 1 个提交 `ddb39af6 docs(settings): note legacy permission verdict cleanup`——是漏提 master 的直接后果。
3. **2026-11-16 刚发生一例顺序颠倒**：!1041 `tmp/toolkits-groups-icons-develop -> develop`（zhangyong5，09:49 合）与 !1043 `feature/toolkits-groups-icons -> master`（yangxu27，09:50 合）。同一功能、**两个人、两个不同分支名、master 先于 develop 合入**——文档 :86 说"合入 master 这个动作本身就是发布生产"，顺序不能反。
4. 用户自己最近一对 `fix/review-filter-candidates -> master`（!1023）与 `fix/review-filter-candidates-develop -> develop`（!1024）也是双分支，说明 `AGENTS.md`「同一变更共用一个功能分支」这条**连作者本人都没在执行**。

同一源分支提多个 MR 的典型（文档 :88 教出来的行为）：`feature/assistant`（!1013→master、!1012→develop）、`fix/smart-query`（!951→master、!952→develop）、`feature/spark-old-drivers`（!950→master、!937→develop、!930→master）、`feature/question-review-master`（!992/!982/!977/!975 全指 master）。

命名混乱：仓库同时混用 6 种形态 `feature/x`、`feature/x-master`、`feature/x-develop`、`feature/x-dev`、`tmp/x-develop`、`fix/x-master`；`tmp/` 从不清理（远端仍挂 `tmp/toolkits-groups-icons-develop`）。

## 3. 根因（文档哪里害人）

1. **文档教的正是同事现在干的事**：:88「共用同一个源分支」→ yangxu27 真的一个分支提两个 MR。这种模式没有任何机制保证两边变更一致，且一旦 develop 侧合入后源分支被删/改写，master 那份就不可追溯。
2. **没写"什么时候才可以提 master"**：只说顺序 develop → master，于是同事两边同时提、谁先合看心情（!1041/!1043 即反例），develop 上未验证的内容先进了 master。
3. **没有一致性验收标准**：skill 用 cherry-pick + `git patch-id --stable` 卡死，文档一个字没有，所以提完 develop MR 就以为完事。
4. **命名无规范 + `AGENTS.md` 把细则外包给错文档**：20 行的 `AGENTS.md` 只写"以工作流文档为准"，读者必然读到错误模型。

## 4. 建议改造（限定两个文件）

```
AGENTS.md
  ├─ 项目概览：技术栈、目录结构、npm 命令、代码规范（把 README 里的组件写法规范收进来；
  │   README 的 docs/RELEASE_PROCESS.md 是断链，一并修）
  └─ 分支与 MR：按【双独立分支 + cherry-pick + patch-id 一致】重写，
     明确 禁止同源分支提双 MR、命名后缀 -master/-develop、master MR 必须在 tst 验证通过后才提
CLAUDE.md                      →  @AGENTS.md 不动
docs/gitlab-branch-workflow.md →  改成一行"已并入 AGENTS.md"，或直接删除
```

注意 `CLAUDE.md` 只有一行 `@AGENTS.md`，该 import 语法只有 Claude Code 与 DeepSeek Harness 会展开；Cursor/Windsurf/Copilot 读到的是无意义文本，它们实际都依赖 `AGENTS.md` 本身（详见 e09863d9 文档）。

## 5. 未决事项

- **`AGENTS.md` 规定双分支还是单分支**：双分支是团队实际做法、也是 skill 的做法，倾向双分支；但等于认定 yangxu27 那类"同源分支双 MR"违规，需业务侧沟通。
- 备选下策：文档写成"两种都允许，但必须保证两边 patch 一致"——会继续留下可漂移的口子。
- 用户尚未回答该问题，改造（重写 `AGENTS.md`、处理 `docs/gitlab-branch-workflow.md`、修 README 断链）均未开始。
