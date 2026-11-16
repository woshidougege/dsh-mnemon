---
id: "d20c058a-95b1-42c4-889f-e777a91088ab"
title: "ssg_sales_ops 分支模型定稿：dev/<itcode> 个人分支 + 双 MR + patch-id 等价校验（落地记录）"
description: "用户拍板推翻旧「一功能一分支」流程：每人一条 dev/<itcode>，每次交付提 →develop 与 →master 两个 MR，用 git patch-id 校验两边 diff 必须相等（单分支下不 cherry-pick，改为一对三点 diff）；四条硬约束（两 MR 建好才合、不勾删源分支、develop→master 顺序、禁止 merge origin/develop）、三个风险对策；AGENTS.md 20→129 行、docs 降为存根，工作区未提交，遗留通知当事人/清理旧分支/README 断链。"
status: "active"
created_at: "2026-11-16T02:16:25.461Z"
updated_at: "2026-11-16T02:16:25.461Z"
content_hash: "d10c28fc6a4b8c070c39bd78e38837e68de06e205057901beb24b9d7bfb774a2"
source_paths:
  - "AGENTS.md"
  - "docs/gitlab-branch-workflow.md"
  - "CLAUDE.md"
  - "README.md"
session_ids:
  - "mnemon-run-v1-a7e35b24-3471-4939-86cd-48b4ae2600fe"
memory_body_ids:
  []
---

# 分支模型定稿：每人一条个人分支 + 双 MR + patch-id 等价校验（ssg_sales_ops 前端）

状态：**规则已定稿并已写入仓库工作区，尚未提交、未提 MR**（2026-11-16）。
本文记录的是用户拍板后的**最终方案与落地结果**。前序调研与冲突分析见 `3061db0b-a4fe-469f-a461-91792b307add`（当时结论为「双分支 vs 单分支待拍板、未落地」，本文件取代其待决状态），多 harness 覆盖分析见 `e09863d9-d9a9-4086-8c3c-75ef110487a7`。

## 1. 用户拍板的模型（推翻仓库旧文档）

用户原话要点：同事「只提交一个 develop 的 MR，还提交一堆 MR 创建一堆分支」，要求「让他们所有的开发在一个分支上，然后只提交 2 个 MR（develop 和 master），每个人一个开发分支」。确认口径为：**开发时一条分支，提 MR 时两个（一个 →develop，一个 →master）**。

规则只落在两个文件：`AGENTS.md` 为唯一正文，`CLAUDE.md` 保持一行 `@AGENTS.md`。用户明确「只管 CLAUDE.md 和 agents.md」。其他 harness 一律先不建薄壳（仓库里本来一个都没有，未确认同事在用什么工具前创建等于制造第二份正文、必然漂移）。

## 2. 关键设计推理：单分支下 patch-id 校验怎么改

用户最初的要求是「校验 cherry-pick + `git patch-id --stable` 必须相等」，这与「一条分支提两个 MR」**逻辑互斥**——同一源分支只有一份代码，不存在第二份可供 cherry-pick。处理方式不是放弃校验，而是改变校验对象：

```powershell
git fetch origin --prune
# 两个 MR 各自真实的 diff（合并基不同，必须分别算）
git diff origin/develop...dev/<itcode> | git patch-id --stable
git diff origin/master...dev/<itcode>  | git patch-id --stable
```

- 两行输出的**第一列必须相同**，第二列是占位 0 不参与比较。
- 它证明的是「→develop 和 →master 两个 MR 交付内容一模一样，不多不少」。
- **该等式只在个人分支基于最新 `origin/master` 时成立**，因此「每轮先重置到 origin/master」与「patch-id 校验」是绑定的。分支合并基过旧时 →develop 侧会把别人历史算进 diff，patch-id 必然不等——这恰好自动暴露「分支没重置」或「被 develop 污染」。

## 3. 四条硬约束（不写就会真出事）

1. **两个 MR 都建好之后才允许合并第一个**：只建 →develop 就合，之后补 →master 时源分支已删或已脏。
2. **合并时不要勾「Delete source branch」**：GitLab 合第一个 MR 会删源分支，第二个 MR 的 source 直接失效；两个都合完再手工删。
3. **顺序固定 develop → master**，master 一律等 tst 验证通过（现场反例：!1041→develop 与 !1043→master 顺序反了，等于跳过 tst 就上生产）。
4. **个人分支绝不能被 develop 污染**：禁止 `git merge origin/develop`。否则 develop 上未上线的内容会跟着 →master 的 MR 一起上生产。与 master 冲突用 `git rebase origin/master`；仅 →develop 冲突才允许一次性 `tmp/<topic>-develop`（从 origin/develop 拉出、合入个人分支、用它提 →develop、合完立即删）。

## 4. 三个真实风险与对策

| 风险 | 对策 |
|---|---|
| 个人分支越用越老、合并基烂掉 | 每轮交付完成后 `git reset --hard origin/master` + `git push --force-with-lease`；force 只允许个人分支，`develop`/`master`/`release` 禁止 |
| 一次交付绑太多功能，review 看不懂、回滚变粗 | 约定「一轮交付 = 一个可独立上线的功能」，多轮交付各提各合 |
| 多人改动在 develop 撞车（现状 develop 领先 master 667、master 又有 37 个 develop 没有的） | 推送前 `git merge-tree --write-tree origin/develop HEAD`，返回码非 0 先本地解决 |

## 5. 落地结果（工作区，未提交）

```
AGENTS.md                      | 20 → 129 行
docs/gitlab-branch-workflow.md | 156 → 25 行（废弃存根）
2 files changed, 133 insertions(+), 162 deletions(-)
```

- **AGENTS.md** 分五块：项目概览（技术栈、`src/` 9 个目录职责、5 条 npm 命令、`components.d.ts`/`auto-imports.d.ts` 自动生成勿手改）、分支模型、提交与提 MR、合并顺序、交付前校验、冲突处理，另加禁止事项/代码规范/验证。SFC 内部顺序与 README 的组件规范一并收进来。
- **docs/gitlab-branch-workflow.md** 降为存根：一段废弃说明 + 新旧对照表（旧「一个功能一条分支、同一源分支提双 MR、无一致性校验」→ 新「每人 `dev/<itcode>`、两个 MR、patch-id 校验」）+ 废弃原因（develop/master 长期分叉、需 backfill MR !1005 事后补）。
- 旧的错误流程不再作为正文：旧文档教「:88 共用同一个源分支」，正是同事同分支提双 MR、漏提一侧的源头；且旧 `:12` 关于后端「单分支 admin-release」已过时（实为 auth-release + admin-release）。
- 校验结果：AGENTS.md 129 行、patch-id 命令两条、`--force-with-lease` 仅限个人分支、禁止勾删源分支、禁止 `merge origin/develop`、代码块成对、表格分隔行数量正确。

## 6. 遗留事项（handoff）

1. 改动**只在工作区，未提交**；当时当前分支仍是旧的 `fix/review-filter-candidates`（behind origin/master 21 个提交）。按新规则迁移需：`git switch -c dev/renxp4 origin/master` → 提交这两份文件 → push → 按新规则提两个 MR → 推前跑 patch-id 双校验。
2. **合并未获授权**：需用户点名具体 MR iid 才可合并。
3. 规则生效不等于同事知道：zhangyong5（只提 develop）、yangxu27（同源分支开多个 MR）是主要当事人，需当面/群里同步，或在 MR 描述里写明变更点供转发。
4. README 仍指向不存在的 `docs/RELEASE_PROCESS.md`（断链），超出「只管两个文件」范围未改。
5. 现存 40+ 个 `tmp/`、`-master`、`-develop` 旧分支未清理，建议先列「可删 / 有 open MR 先别删」清单，不自动删。

## 7. 可复用的教训

- 对团队规范而言，**「复制正文到多个 harness 文件」是必然漂移的根源**；一份正文 + 一行指路（`@AGENTS.md`）才可持续。
- 规则文档只要写错一处流程，就会稳定地培训出错误行为——分析同事的 MR 偏差时，先怀疑规则文档而非个人习惯。
- 用 `patch-id` 这类可机械校验的等式替代「大家都小心一点」的口头约定，才能让「两边一致」变成可验证事实。
