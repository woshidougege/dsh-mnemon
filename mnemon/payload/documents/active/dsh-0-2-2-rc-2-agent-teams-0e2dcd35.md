---
id: "0e2dcd35-3cf4-476f-a240-8e710dd7a89d"
title: "DSH 0.2.2-rc.2 升级：安装失败根因、脱离进程树的更新手法与 Agent Teams 会话级开关迁移"
description: "本机 DSH 桌面版升级到 0.2.2-rc.2 的可复用记录：MoveFileExW error 32 导致的安装回滚与 WMI 外挂更新流程（含退出码 2 不代表失败的判据），Agent Teams 三包改名 / experimental 组合被移除 / 斜杠命令废弃 / 按会话开关与 defaultEnabled 配置口子（仅 maxTasks 有效），以及官方默认关闭的四条理由与随包文档索引。"
status: "active"
created_at: "2026-11-13T03:21:51.485Z"
updated_at: "2026-11-13T03:21:51.485Z"
content_hash: "9b2fe0dedc3a38ce8f8b71f61c33544f3ea3c22f9c28f6f95e04601ac56f26a1"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-68a47981-93eb-421b-9270-52ff40c47255"
memory_body_ids:
  []
---

# DSH 0.2.2-rc.2 升级落地与「Agent Teams 变会话级开关」迁移记录

面向本机 DSH 桌面版（Windows，profile = `desktop`）。记录两件事：一次安装失败的真实根因与脱离进程树的更新手法；rc.2 把 Agent Teams 从"全局强制装"改为"按会话开关"的事实、配置口子与官方文档索引。状态：已升到 0.2.2-rc.2；`defaultEnabled: true` 尚未写入 patch（等用户决定）。

## 1. 升级：失败根因是"文件被占用"，不是下载失败

- 安装日志：`installer-logs\directory-failure-20261112-171010.log`
  - `Operation: MoveFileExW` / `Windows error: 32 (另一个程序正在使用此文件，进程无法访问。)`
  - `Source: C:\Users\63588\software\DeepSeek Harness` → `Destination: ...\DeepSeek Harness.old-{6ACED7E9-...}`
  - `Attempts: backup: Windows error: 32`，重试 10 次全失败 → 回滚 → 重启后仍是旧版本。
- 包本身早已下载完成：16:55、299,659,528 字节、SHA512 `5rRAkVru…NcJA==` 与 feed 一致 —— 说明**校验/下载环节没问题，卡在安装期的目录改名**（宿主进程占着 exe/asar）。
- 另发现 `C:\Users\63588\software\DeepSeek Harness.new-{52ECA3C1-...}` 是 10 月 30 日那次更新失败的残骸，会干扰后续清理。

### 可复用的更新手法（自更新会杀掉自己，必须外挂）

用 WMI 创建脱离 DSH 进程树的静默更新器（父进程是系统服务）：`Win32_Process.Create` 返回 `returnValue: 0`。脚本流程：

1. 结束全部 `DeepSeek Harness` 进程（实测 8 个）
2. 清理 `.new-*` / `.old-*` 残骸
3. `setup.exe /S /D=C:\Users\63588\software\DeepSeek Harness` 静默安装
4. 校验 exe `FileVersion` 后重新启动应用

- 日志：`%LOCALAPPDATA%\Temp\dsh-update\update.log`
- 逃生阀：桌面 `取消DSH自动更新.cmd`（写 cancel 标记，脚本读到即不动作）、`立即更新DSH.cmd`（10 秒后执行）
- 经验：**倒计时先回复用户再动**，因为安装必然中断当前会话；会话本身持久化，重启后记录仍在。
- 本次结果：17:25 那次安装 `exit code 2` 是清理 10/30 遗留目录没删干净，但文件实际替换成功；exe FileVersion = `0.2.2-rc.2`，asar 213 MB → 208 MB。**退出码 2 在这里不代表安装失败**，要看 FileVersion 与 asar 体积。

## 2. Agent Teams 为什么"看起来没了"

旧形态：`@deepseek-ai/dsh-experimental-agent-team-profile` 组合包硬装进 profile，套用到**所有**会话，并顺带禁用普通 `tool-subagent` / `tool-subagent-fork` 委派工具。

rc.2 的做法：

| 旧包 | 新包 |
|---|---|
| `dsh-experimental-agent-team` | `@deepseek-ai/dsh-agent-team` |
| `dsh-experimental-tool-agent-team` | `@deepseek-ai/dsh-tool-agent-team` |
| `dsh-experimental-client-ui-agent-team` | `@deepseek-ai/dsh-client-ui-agent-team` |

- 加载时自动移除已停用的 `experimental-agent-team-profile` 组合选项 —— 用户 `~/.dsh/profiles/desktop/package.json` 里那行是被加载器删掉的，不是手滑。
- `agent-team` + `tool-agent-team` 已在 **base 组合默认加载**；Web 组合带 `ui-agent-team`。**能力没减少，只是默认关**。
- 开关位置：**根会话**的「模型与推理等级」菜单 →「启用智能体团队协作」（说明文案："通过共享任务板协调多个智能体的分工与进度，复杂任务表现更佳，token 开销也相应更高。"）。开启后从会话头部 Subagent 入口进同一面板：上方成员、下方共享任务板；子会话只保留导航。
- **`/team on` / `/team off` 斜杠命令已移除**（`v0.2.1-rc.3/team-mode-menu`）。命令实现方改为 `ctx.agentTeams.setEnabled(rootAgent, enabled)` / 浏览器端 `remote.agentTeams.setEnabled(rootSessionId, enabled)`。用户"开关消失"的体感主要来自这里。
- 模型侧多出四个工具：`createTeamTask` / `listTeamTasks` / `getTeamTask` / `updateTeamTask`。
- 行为语义：开关按会话保存、重启仍生效，根会话的选择控制整棵子代理树的工具与提示词；关闭**不丢任务、不丢子代理**（已进入执行的操作继续完成，排队中的在开始时检查模式）；只有根会话能改；没保存过选择的旧会话任务板保持开启。必读事件 `team/mode`，旧读取器会拒绝该未知事件。

## 3. 配置口子与坑

```yaml
# C:\Users\63588\.dsh\profiles\desktop\cordis.patch.yml
- id: agent-team
  config:
    defaultEnabled: true
```

- `@deepseek-ai/dsh-agent-team` **只认两个键**：`maxTasks`（默认 256）、`defaultEnabled`（默认 `false`）。`maxMembers`、`maxMessageBytes`、`disposalTimeoutMs` 写了不生效，dsh 启动时会告警（用户旧配置里正好有这三个键）。
- `defaultEnabled: true` 会在首次模型步骤之前启用新根会话（含普通分叉）并把选择存为 `team/mode`；**已有选择优先，恢复会话时永不应用默认值**（包括从未保存选择的会话）。保存失败会拒绝 Agent 创建。
- Team 激活前要求 `sessionPersistence` 可用；任务修改在向在线根 Session 追加 `team/task` 前响应取消，随后等刷盘。刷盘不确定时报 `TEAM_TASK_PERSISTENCE_UNCONFIRMED`（任务已可见但持久性未知），重试前先读任务。
- 并发：任务修改必须带 `expectedRevision`，过期报 `TEAM_TASK_STALE_REVISION`；`writeScopes` 重叠只警告不阻止；删除任务留墓碑、不占 `maxTasks`。
- Client 插件迁移：`TeamActionProps` → `TeamBoardProps`，任务内容注册到 `conversation.session.header.actions.subagent.footer`（不再是页头操作插槽）。
- 改 patch 前按用户既有习惯备份：同目录 `cordis.patch.yml.bak-*`。

## 4. 官方为何默认关（文档证据，非专门说明文）

用户追问"官方有啥说明"，随包文档给出的事实与可归纳的理由：

1. **token/性能**：开关自带说明即写"token 开销也相应更高"；包文档「模型体验 → Token 影响」写明每个任务结果会追加一份紧凑任务视图。
2. **适用面窄**：原文"委派 agent 需要划分和跟踪共享工作时选择此包；无需共享任务板时，普通 subagent 已足够"。边界明确：不创建隔离工作目录、不跨进程协调、不自动释放任务所有者。
3. **影响整棵树**：根会话选择控制整棵树的工具集合与提示词，属行为级开关，故交给用户按会话决定。
4. **依赖会话持久化**：模式要落盘成 `team/mode`，保存失败会拒绝 Agent 创建；默认开等于把"写盘失败"升级成"新建会话失败"。

（`defaultEnabled` 配置项本身的存在即证明官方意图是"默认关但可强制开"，而非删除能力。）

## 5. 文档索引（均在 `app.asar` 内 `dsh/` 下）

- `node_modules/@deepseek-ai/dsh/docs/upgrade-guide/v0.2.1-rc.2/session-team-mode/guide.zh.md` —— 会话级模式与迁移步骤
- `node_modules/@deepseek-ai/dsh/docs/upgrade-guide/v0.2.1-rc.3/team-mode-menu/guide.zh.md` —— 斜杠命令移除
- `node_modules/@deepseek-ai/dsh/docs/upgrade-guide/v0.2.1-rc.3/team-projection/guide.zh.md` —— 投影检查点重建
- `node_modules/@deepseek-ai/dsh/docs/upgrade-guide/v0.2.1-rc.2/agent-team-subagents/guide.zh.md` —— 已移除的 Team 工具/服务方法/配置键替代
- `node_modules/@deepseek-ai/dsh-agent-team/README.zh.md` —— 配置字段真源、`defaultEnabled`、最小工作配置
- `node_modules/@deepseek-ai/dsh/docs/subsystems/agent-team.zh.md#web-projection`、`docs/user/guide/python-sdk.zh.md#enable-team-mode`（含可直接用的 SDK patch）
- 源码地图：`src/index.ts`（配置校验）、`src/task-board.ts`（CAS/DAG）、`src/journal.ts`（Lead 日志事务）、`src/projection.ts`（回放 `team/mode` 与任务、发布 `agentTeam` 视图）、`src/task-view.ts`（就绪性与写重叠推导）

## 6. 未决

- 是否把 `defaultEnabled: true` 写进 `C:\Users\63588\.dsh\profiles\desktop\cordis.patch.yml`（让新会话默认就是团队模式）—— 已向用户提议，未答复。替代路径是先在当前会话手动打开开关试效果。
