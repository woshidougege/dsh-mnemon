---
id: "81078dc5-14c3-46cd-af43-9737596876b4"
title: "Claude Code / Codex 子智能体插件：调用方式、模型来源与三份配置收敛"
description: "dsh-subagent-claude-code 与 dsh-subagent-codex 两插件的能力与调用方式（AI 自动/可点名）、模型不随调用参数传递、scrubbedParentEnv 造成的配置不继承机制与代码行号证据、本机 DSH/Claude Code/Codex 三处 base_url-model-key 对照与 8080 不通问题、方案 A（塞插件 env）与 B（单一源+同步脚本）的取舍。调研完成，待用户拍板。"
status: "active"
created_at: "2026-11-19T06:52:21.272Z"
updated_at: "2026-11-19T06:52:21.272Z"
content_hash: "3a5c792b4775f16cd78ac0e6fbc6f838ca1a6a040de44dcd7edcff6b0f5e5da1"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-7e189d5d-5e59-49b8-9c1b-da3fe2b0f4f8"
memory_body_ids:
  []
---

# Claude Code / Codex 子智能体插件：调用方式、模型来源与「三份配置」收敛

状态：**调研完成，收敛方案待用户拍板**（2026-11-19）。本文记录结论与代码证据；用户尚未同意执行任何方案，也未修改任何配置文件。

## 1. 两个插件是什么

| 插件包 | 注册的 provider / 工具 | 实现要点 |
|---|---|---|
| `@deepseek-ai/dsh-subagent-claude-code` | provider 名 `claude-code`（`DEFAULT_PROVIDER_NAME`），调用工具 `subagent_claude_code` | 注册进 `subagents` 服务；每次 accepted run 在工作区里拉起官方 **Agent SDK**，SDK 再 spawn 真实 CLI，挂在共享 `subprocess` owner 下 |
| `@deepseek-ai/dsh-subagent-codex` | 调用工具 `subagent_codex` | 底层是 npm 包 `@openai/codex`（`bin/codex.js` 按平台定位 vendor 二进制，Windows x64 → `@openai/codex-win32-x64`） |

两者都是「Profile-named one-shot 子智能体 provider」：`inheritsParentContext = false`（不继承父会话上下文）、`capabilities = NO_START_CAPABILITIES`（模块 `@deepseek-ai/dsh-subagent-claude-code` 头部注释 + `lib/index.js` 中的 `ClaudeCodeProvider` 类字段）。

调用入参只有：`prompt`（任务文本）+ 可选 `cwd` + `name`（显示名）。返回子任务名，完成后给完成通知与最终答案；子智能体中间过程、工具调用、stderr 不进当前会话。

## 2. 怎么调用：AI 自动为主，用户可点名

- 工具注册给**主智能体**，不是 UI 按钮。自然语言下指令即可（「用 Claude Code 把这段审一下」）。
- 可点名强制（「用 subagent_codex 跑这个任务」）。
- 插件管理页只负责启用/配置，**没有手动触发入口**。

## 3. 模型不在调用入参里

`Config`（`@deepseek-ai/dsh-subagent-claude-code/lib/index.js:495-501`）：

```js
providerName: z.string().min(1).default("claude-code"),
model: z.string().min(1),
env: z.dict(z.string()).default({}),
permissionMode: z.union([...CLAUDE_CODE_PERMISSION_MODES]).default(DEFAULT_CLAUDE_CODE_PERMISSION_MODE),
disposeGraceMs: z.number().default(DEFAULT_DISPOSE_GRACE_MS)
```

start() 里只在配置存在 model 时才下发：`...this.config.model === void 0 ? {} : { model: this.config.model }`（`:525`）。默认 `permissionMode` 实际解析为 `"dontAsk"`（`:546`）。

Codex 侧配置 schema 有 `env: z.dict(z.string()).default({})`（`@deepseek-ai/dsh-subagent-codex/lib/index.js:819`），start 时 `env: spec.env`（`:655`），apply 时 `env: this.config.env`（`:846`、`:864`）。

结论：DSH 主会话**只传任务文本和工作目录**。模型要么写插件配置 `model`，要么落回产品原生设置。

## 4. 为什么不继承 DSH 的 base_url / api key / model（架构限制，不是缺开关）

1. **父进程凭据被清理**：`claudeQueryOptions` 里 `env: { ...scrubbedParentEnv(), ...spec.env }`（`@deepseek-ai/dsh-subagent-claude-code/lib/index.js:302`）；`sdkEnvironmentOverlay` 还会把被清理的键在子环境里显式置 `undefined`，防止「父环境原本存在的键」被 SDK 侧重新补回（`lib/index.js:21-27`，注释：「Encode the SDK's complete child environment as a subprocess overlay」）。子进程能拿到的只有插件配置里的 `env` 字典。
2. **子进程是独立产品**：Claude Code / Codex 只读自己的 `~/.claude/settings.json`、`~/.codex/config.toml`，不认识 `cordis.patch.yml` 里的 `work-buddy` provider。
3. **协议不对等**：DSH 主模型走 `openai-responses`；Claude Code 走 Anthropic `/v1/messages`；Codex 自定义 provider 走 `wire_api = "responses"`。没有一一映射的配置项。

## 5. 本机三处配置对照（2026-11-19 实测）

| 项 | DSH 主会话 | Claude Code | Codex |
|---|---|---|---|
| 配置文件 | `C:\Users\63588\.dsh\profiles\desktop\cordis.patch.yml`（`llm-pi-ai` → provider `work-buddy`，`agent-default-model`） | `C:\Users\63588\.claude\settings.json` | `C:\Users\63588\.codex\config.toml` |
| base_url | `http://localhost:8788` | `ANTHROPIC_BASE_URL=http://localhost:8788` | `http://39.96.21.173:8080/v1` ⚠️ **实测不通** |
| model | `deepseek-v4.1-flash`（`reasoningEffort: max`） | 全族映射为 `deepseek-v4.1-flash[1M]`（Haiku/Opus/Sonnet/Fable 同一模型） | `deepseek-v4.1-flash`（`wire_api = "responses"`） |
| key | `.credentials.yaml` 的 `WORK_BUDDY_API_KEY` | `ANTHROPIC_AUTH_TOKEN` | `experimental_bearer_token` |

连通性实测：`localhost:8788`（WorkBuddy 网关）在线 → Claude Code 链路通；`39.96.21.173:8080` 无响应 → Codex 子智能体虽模型名正确，首次委派会在 initialize 阶段失败。

所以用户现有配置其实**已经指向 DSH 主模型**（不是没配），痛点只是「同一套 base_url/model/key 维护三份」。

## 6. 两条收敛路径

**A. 全部塞进插件 `env` 字典**
- Claude Code：可行且彻底——`ANTHROPIC_BASE_URL`、`ANTHROPIC_AUTH_TOKEN`、`ANTHROPIC_MODEL` 写进 profile 的 `cordis.patch.yml`，清空 `settings.json` 的 env。
- Codex：不彻底——自定义 provider 的 `base_url`、`model` 只能写 `config.toml`（`env_key` 仅能解决 key）。
- 副作用：token 明文进 `cordis.patch.yml`。

**B. 单一源 + 同步脚本（当时推荐）**
- 唯一源：`cordis.patch.yml` 的 `work-buddy`（baseURL、model）+ `.credentials.yaml` 的 `WORK_BUDDY_API_KEY`。
- 脚本生成 `~/.claude/settings.json` 的 env 与 `~/.codex/config.toml` 的 model/base_url/key。
- 维护点从「三份文件」降为「一份文件 + 一条命令」，token 不散落到 DSH 配置。
- 建议落点：`C:\Users\63588\.dsh\sync-subagents.mjs`，加 `pnpm sync-agents` 入口。

## 7. 未决 / 待办

- 用户尚未确认执行 A 还是 B；上一轮以「要我现在做 B 吗？」收尾，等待回答。
- **不论选哪条路径，Codex 的 `base_url` 都必须从 `http://39.96.21.173:8080/v1` 改成 `http://127.0.0.1:8788/v1`**（备份 `config.toml` 后只改这一行）。
- Claude Code 走 Anthropic `/v1/messages` 能否真正跑通**尚未实测**（只验证了网关 8788 在线）。
- 修改插件 `model` 后需重启 DSH 生效；耗时/额度会走各自产品的账号体系。
