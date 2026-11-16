---
id: "e09863d9-d9a9-4086-8c3c-75ef110487a7"
title: "多 Harness 指令文件覆盖矩阵与 AGENTS.md 单一事实源方案（ssg_sales_ops）"
description: "ssg_sales_ops 仓库 master/develop 下 4 个 MD 文件的 blob hash 与作用、DSH 源码确认的 AGENTS.md/CLAUDE.md 读取规则、19+ harness 指令文件矩阵（是否直读 AGENTS.md、需不需要薄壳）、A/B 分档结论、单一事实源骨架与「复制全文 vs 一行指路」取舍、README 断链与 @import 空指针两个坑。调研完成但落地未执行。"
status: "active"
created_at: "2026-11-16T01:49:20.091Z"
updated_at: "2026-11-16T01:49:20.091Z"
content_hash: "85c038e1037cec8200ac1b27eea3f5e624249f4270d0a4f0a447317b4402a35b"
source_paths:
  - "AGENTS.md"
  - "CLAUDE.md"
  - "README.md"
  - "docs/gitlab-branch-workflow.md"
session_ids:
  - "mnemon-run-v1-b312fe64-8c1b-4be8-ab34-71e07bda5ed0"
memory_body_ids:
  []
---

# 多 Harness 指令文件覆盖矩阵与「AGENTS.md 单一事实源」方案（ssg_sales_ops）

状态：**调研完成，落地未执行**（2026-11-16）。本文只是调查结论与待拍板的方案；仓库当时尚未新增任何薄壳规则文件。

## 1. 仓库现状（已 fetch origin 后逐文件比对 blob hash）

`master` 与 `develop` 的 MD 文件**完全同源**（`git diff` 为空，逐个 blob hash 相同）：

| 文件 | blob hash | 作用 |
|---|---|---|
| `AGENTS.md` | `2866c42ff57c3215bc47c0d4d8e1fee0f194b4d7` | 唯一真实规则源（20 行，全是 Git/MR 流程约束） |
| `CLAUDE.md` | `eef4bd20cf972fc2f3c7405fdae3fdcff80e59d4` | 只有一行 `@AGENTS.md`（Claude Code 的 import 语法） |
| `README.md` | `0b2dcd4d17a326b1cd47faf73533a260b3141249` | 模板介绍 + 发布流程 + 组件写法规范 |
| `docs/gitlab-branch-workflow.md` | `786e45b21b4d91fb59c603defac9d2d12fc014fe` | 分支工作流细则 |

仓库内**没有任何其他 harness 的配置文件**：无 `.cursor/rules`、`.github/copilot-instructions.md`、`GEMINI.md`、`.trae`、`.codebuddy`、`.qoder`、`QWEN.md`、`.windsurf`、`.clinerules`；`docs/` 下只有上面那一个文件。工作区的 `.agents/worktrees/`、`.tmp/worktrees/` 是 worktree 副本，不是规则文件。

## 2. DeepSeek Harness 自身读哪些文件（直接读源码确认，非文档推断）

位置：`C:\Users\63588\software\DeepSeek Harness\resources\app.asar\dsh\` 内 `@deepseek-ai/dsh-agent-instructions` 包：

- `src/config.ts`：`DEFAULT_INSTRUCTION_FILE_CANDIDATES = ['AGENTS.md', 'CLAUDE.md']`、`DEFAULT_LOCAL_INSTRUCTION_FILE_CANDIDATES = ['AGENTS.local.md', 'CLAUDE.local.md']`、`DEFAULT_PROJECT_ROOT_MARKERS = ['.git']`
- `src/render.ts`：`USER_GLOBAL_FILE = 'AGENTS.md'`、`AGENTS_GLOBAL_DIRECTORY = 'agents-global'`、`USER_GLOBAL_DIRECTORIES = [USER_GLOBAL_DIRECTORY, AGENTS_GLOBAL_DIRECTORY]`

结论：DSH 同时读 `AGENTS.md` 与 `CLAUDE.md`，支持 `@import`，全局兜底为 `~/.dsh/AGENTS.md` 与 `~/.agents/AGENTS.md`，项目边界由 `.git` 标记。

## 3. 主流 harness 指令文件矩阵（2026-11 口径）

| Harness | 主文件 | 其他可被读取 | 是否自己读 AGENTS.md |
|---|---|---|---|
| Codex | `AGENTS.md` | `AGENTS.override.md`；从 `~/.codex/` 到项目根再到 cwd 逐层，每层一个，默认合计上限 32 KiB | 原生 |
| Claude Code | `CLAUDE.md` | `CLAUDE.local.md`、`.claude/rules/*.md`（可按 path 生效） | 仅当目录里**没有** `CLAUDE.md` 时（v2.1.277+） |
| Cursor | `.cursor/rules/*.mdc`（description/globs/alwaysApply frontmatter） | 根与子目录 `AGENTS.md` 合并、就近优先；`.cursorrules` 仅用于迁移 | 原生 |
| GitHub Copilot | `.github/copilot-instructions.md`、`.github/instructions/*.instructions.md`（`applyTo` glob） | 仓库内任意位置 `AGENTS.md`（云端 agent / code review） | 云端读；IDE 插件侧靠 copilot-instructions |
| Gemini CLI | `GEMINI.md` | 可在 `settings.json` 配 `context.fileName` 改成 AGENTS.md | 默认不读（需配置） |
| **DeepSeek Harness** | `AGENTS.md` + `CLAUDE.md` | `AGENTS.local.md`、`CLAUDE.local.md`；全局 `~/.dsh/AGENTS.md`、`~/.agents/AGENTS.md` | 两个都读，支持 `@import` |
| ZCode（Z.ai） | `AGENTS.md` | 全局 `~/.zcode/AGENTS.md` | 原生 |
| Qoder（阿里） | `.qoder/rules/*.md` | — | 不读 |
| Qwen Code（阿里） | `QWEN.md` | `.qwen/rules/*.md`（`paths:` 条件加载）；v0.26.2 起两者都加载，`QWEN.md` 优先 | 读，但优先级更低 |
| Trae（字节） | `.trae/rules/project_rules.md` | 全局规则在 IDE 设置里 | 不读 |
| CodeBuddy（腾讯） | `.codebuddy/rules/*.md` | `CODEBUDDY.md`（CLI 启动加载）、`.codebuddy/` 下 skills/commands | 走自家记忆体系 |
| WorkBuddy（腾讯办公） | `.workbuddy/rules/`（用户级 `~/.workbuddy/rules/`） | 部分版本认 `AGENTS.md` | 存疑，需实机验证 |
| Zed | `.rules` | fallback 链：`.cursorrules` → `.windsurfrules` → `.clinerules` → `.github/copilot-instructions.md` → `AGENT.md` → `AGENTS.md` | 排最后兜底 |
| Cline / Roo / Kilo | `.clinerules/` | `AGENTS.md`、`~/.agents/AGENTS.md` 在 Rules 面板可勾选 | 原生 |
| Windsurf（现 Devin Desktop） | `.windsurf/rules/*.md`（always_on/glob/model_decision/manual） | `.windsurfrules`（旧）、`AGENTS.md`、`AGENTS.local.md` | 原生 |
| Aider | `CONVENTIONS.md` | 任何文件，但需 `/read`、`--read` 或写进 `.aider.conf.yml` 的 `read:` | 不自动 |
| Continue | `.continue/rules/*.md` | `AGENTS.md` | 原生 |
| JetBrains Junie | `.junie/guidelines.md` | `AGENTS.md`（Junie CLI 直读） | CLI 读 |
| Amazon Q Developer | `.amazonq/rules/*.md` | — | 不读 |
| Augment Code | `.augment/rules/` | `AGENTS.md` | 原生 |
| Warp | `WARP.md` | `AGENTS.md`（官方推荐直接用 AGENTS.md） | 原生 |

口径来源：[rulefiles.dev 指令文件矩阵](https://rulefiles.dev/matrix/)（CC BY 4.0，`lastReviewed: 2026-11-04`）＋各官方文档；DSH 一行来自上面的源码阅读。

## 4. 分档结论

**A. 无需新增文件（AGENTS.md 通道已通）**：Codex、Cursor、Cline/Roo/Kilo、Windsurf、Zed（兜底）、Continue、Augment、Warp、ZCode、DeepSeek Harness，以及 Claude Code（靠 `CLAUDE.md` 的 `@AGENTS.md`）。

**B. 想要覆盖就必须各加一个薄壳文件**：`GEMINI.md`、`QWEN.md`、`.trae/rules/project_rules.md`、`.qoder/rules/*.md`、`.codebuddy/rules/*.md` + `CODEBUDDY.md`、`.workbuddy/rules/`（待实机确认）、`.github/copilot-instructions.md`（只影响 IDE 侧 Copilot Chat）、`.cursor/rules/*.mdc`（仅当需要 Cursor 专属 glob 规则）、`.junie/guidelines.md`、`.amazonq/rules/`、`.augment/rules/`、`.windsurf/rules/`、`.clinerules/`、`CONVENTIONS.md`（Aider）、`WARP.md`。

## 5. 建议骨架与取舍

推荐「**AGENTS.md 单一事实源 + 各 harness 薄壳**」：

```
AGENTS.md                       # 唯一真实内容
CLAUDE.md                       # @AGENTS.md（已有）
GEMINI.md                       # Gemini 不支持 import
QWEN.md                         # 同上
.github/copilot-instructions.md # Copilot 不支持 import
.trae/rules/project_rules.md
.qoder/rules/00-agents.md
.codebuddy/rules/00-agents.md
```

其余（Cursor `.mdc`、Junie、Amazon Q、Augment、Windsurf、Cline、Aider）先不加——AGENTS.md 通道已通，加了只是多份副本要同步。

**薄壳内容两种写法及其取舍（未决）**：
- 复制全文：模型效果最好（不必二次读文件），代价是 `AGENTS.md` 改一次要同步 6 个文件；
- 写一行指路（"完整规则见 AGENTS.md"）：几乎零维护，但弱模型可能懒得去读。

倾向：**Gemini / Qwen 用复制全文**（import 能力最弱），其他用指路。

## 6. 顺带发现的坑

1. `README.md` 第 55 行链接 `docs/RELEASE_PROCESS.md`，但 `docs/` 下只有 `gitlab-branch-workflow.md` —— **断链**。
2. `CLAUDE.md` 的 `@AGENTS.md` 对 Claude Code 是正解（有 `CLAUDE.md` 就不读 `AGENTS.md`），但**任何不支持 `@import` 的 harness 复制这个文件都是空指针** —— 要么给它们真内容，要么别复制。

## 7. 当时待用户确认的两点

1. "qcode" 按 **Qoder**（阿里，`.qoder/rules`）理解，"zcode" 按 Z.ai 的 **ZCode**（`AGENTS.md`）理解；若是内部自研工具需重新查其规则文件名。
2. 薄壳用「复制全文」还是「一行指路」（见第 5 节）。
