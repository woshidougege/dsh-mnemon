---
id: "d2da0cf8-a949-4789-851c-81718ed06ac3"
title: "Claude Code / Codex 子智能体插件：permissionMode 默认值、可用取值与实测验证"
description: "两个 DSH 子智能体插件的权限层记录：Claude Code 默认 dontAsk（写入被拒）/acceptEdits/bypassPermissions 三档实测，Codex 默认 never 实为 workspace-write 且 config.toml 的 sandbox_mode 不生效，dangerously-bypass-approvals-and-sandbox 的越级写入验证法，patch yml 热生效，与无人值守无审批的风险控制要点。"
status: "active"
created_at: "2026-11-19T07:00:33.981Z"
updated_at: "2026-11-19T07:00:33.981Z"
content_hash: "1b65dd23c1c697d630b23a43320b6d6e7c5f72ca51f0905162e757866c0da1f1"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-cf9d88e3-7804-4739-9599-426f12f621b8"
memory_body_ids:
  []
---

# Claude Code / Codex 子智能体插件的权限模式：默认值、取值与实测验证

本机（DSH desktop profile，`@deepseek-ai/dsh-subagent-claude-code` / `@deepseek-ai/dsh-subagent-codex` 均为 `0.2.2-rc.5`）2026-11-19 实测记录。补充并引用既有调研文档 `81078dc5-14c3-46cd-af43-9737596876b4`（调用方式、模型来源、配置不继承机制、三处 base_url/model/key 对照、方案 A/B 取舍），本文只覆盖**权限**这一层。

## 1. 核心结论

- 两个插件各自有独立的 `permissionMode` 配置项，**与子产品原生设置互不影响**。
- 关键易错点：`~/.codex/config.toml` 的 `sandbox_mode = "danger-full-access"` 只是 Codex CLI 的默认值，**不控制插件行为**。插件在 thread 参数里下发自己的 `permissionMode`（`@deepseek-ai/dsh-subagent-codex/lib/index.js` 中 `THREAD_PERMISSION_PARAMS[this.permissionMode]`，约 `:216`），由此决定沙箱与审批策略。
- 无人值守子智能体不会弹任何审批；所有需要审批的操作直接被拒。要让它真正改文件/跑命令，只能在配置里选更宽的 permissionMode。
- 配置写在 `C:\Users\63588\.dsh\profiles\desktop\cordis.patch.yml` 即**热生效，无需重启 DSH**（实测：改完立刻派任务即成功）。

## 2. Claude Code 侧

`Config`（`@deepseek-ai/dsh-subagent-claude-code/lib/index.js:495-501`）含 `permissionMode`，默认解析为 `"dontAsk"`（`:546`，`DEFAULT_CLAUDE_CODE_PERMISSION_MODE`）。

| permissionMode | 行为（实测） |
|---|---|
| `dontAsk`（默认） | 只读操作可用；写文件、执行命令一律被拒，且不弹审批 |
| `acceptEdits` | 允许文件编辑，需审批的操作仍拒 |
| `bypassPermissions` | 全部放开；插件的 `canUseTool` 分支被关闭（`lib/index.js:309` 改为下发 `allowDangerouslySkipPermissions: true`） |

`dontAsk` 下的实测失败姿势（同一任务三种写法全拒）：`PowerShell Set-Content` → 权限被拒；Claude Code `Write` 工具 → 权限被拒；`Bash printf` → 权限被拒。

需要的配置：

```yaml
- id: subagent-claude-code
  config:
    permissionMode: bypassPermissions
```

复验：成功创建 `C:\Users\63588\public\claude-subagent-smoke2.txt`（44 字节，内容 `claude-subagent-ok 2026-11-19T14:55:55+08:00`）。

## 3. Codex 侧

`Config`（`@deepseek-ai/dsh-subagent-codex/lib/index.js:816-822`）：`permissionMode: z.union([...CODEX_PERMISSION_MODES]).default(DEFAULT_CODEX_PERMISSION_MODE)`，默认 `"never"`（`:546`、`:865` 回落 `"never"`）。

三个可用取值（`lib/index.js:20-28` 附近的映射表）：

| permissionMode | approvalPolicy | sandbox |
|---|---|---|
| `never`（默认） | `never` | `workspace-write` |
| （中间档） | — | `workspace-write` |
| `dangerously-bypass-approvals-and-sandbox` | `never` | `danger-full-access` |

注意默认档 `never` 名字像"最宽松"，实际语义是 **approvalPolicy=never + workspace-write**，即"工作区外写不了、也永远等不到授权"。要全放开必须写 `dangerously-bypass-approvals-and-sandbox`。

```yaml
- id: subagent-codex
  config:
    permissionMode: dangerously-bypass-approvals-and-sandbox
```

复验（越级验证法）：让子智能体往工作区 `C:\Users\63588\public` 的上一级写 `C:\Users\63588\codex-bypass-test.txt` —— 成功（`codex-bypass-ok 2026-11-19T14:59:12+08:00`），证明已脱离 workspace-write。同权限档下工作区内写入另测通过：`C:\Users\63588\public\codex-subagent-smoke.txt`（53 字节）。

另注：`C:\Users\63588\.codex\config.toml` 的自定义 provider 的 `base_url` 已从不通的 `39.96.21.173:8080/v1` 修正为 `http://127.0.0.1:8788`（`wire_api = "responses"`），这是 Codex 子智能体能跑通的前置条件。

## 4. 操作要点与风险

- 改 permissionMode 后**先在具体项目目录派一个"写文件 + 读回"的最小任务复验**，再用它干正事；越级写入是区分 workspace-write 与 danger-full-access 最快的手段。
- `bypassPermissions` / `dangerously-bypass-approvals-and-sandbox` 等于把传入 `cwd` 的整个子树交出去，且无人值守不弹审批：派任务时把 `cwd` 限定到具体项目目录，并在任务描述里写明"不要动其他文件"。
- 两个子智能体回报的模型名都是 `unknown` 属正常 —— 产品不向自身暴露模型标识；网关侧实际仍是 `deepseek-v4.1-flash[1M]`。
- 工具注册名：`subagent_claude_code`、`subagent_codex`（另有 `list_subagent_models`）。在 `run_code` 里不能直接当函数调（`tools.subagent_codex is not a function`），必须由主智能体按工具名发起委派。
- 本机生效配置位于 `C:\Users\63588\.dsh\profiles\desktop\cordis.patch.yml` 尾部（两个 `- id:` 覆盖块，插在 `# <<< dsh-windows-mcp <<<` 之前），并以注释标明来源日期。
- 后续若做"一份源 + 同步脚本"收敛（既有文档方案 B），记得把这两个 permissionMode 覆盖块纳入同步脚本的幂等检查，别被重写冲掉。

## 5. 未决项

- 验证用的临时文件（`claude-subagent-smoke2.txt`、`codex-subagent-smoke.txt`、`codex-bypass-test.txt`）是否保留为记录，用户尚未答复。
