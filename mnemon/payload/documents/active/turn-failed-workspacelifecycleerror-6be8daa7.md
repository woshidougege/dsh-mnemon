---
id: "6be8daa7-27e7-4ca1-93d9-4e26142f2a82"
title: "生产会话每轮 Turn failed（WorkspaceLifecycleError）复现与前端静默假死根因"
description: "ssg-sales-ops 某会话 workspace 丢失导致每轮 Turn 启动即 WorkspaceLifecycleError（复现对照数据）；附带 runtime_reset_required 全量重放与 pbiRuntimeStream 停滞 stop() 不复位 loading 导致点击被吞的链路、文件行号与两条待修项。"
status: "active"
created_at: "2026-11-19T12:24:28.121Z"
updated_at: "2026-11-19T12:24:28.121Z"
content_hash: "00c00370cc56e6855da1b14b727c4045bee110a523ccd2b9acb2790d8c2856cf"
source_paths:
  - "src/services/pbiRuntimeStream.ts"
  - "src/components/Agent/AgentChatPanel.vue"
session_ids:
  - "mnemon-review-v1-9db24969-20f5-41be-bc1a-e881c74abe02"
memory_body_ids:
  []
---

# 生产会话每轮 Turn 失败（WorkspaceLifecycleError）复现与双端根因

## 0. 现象

业务用户反馈：在 ssg-sales-ops 的 SSG 推荐会话里点发送没反应，重新登录也不行。

后端证据：该会话 `bba8fdb4-…`（profile `sdq_tcv_booking`，2026-11-18 17:17 创建，181 条消息）的 `updated_at` 一直停在 **18:15:53**，即 18:15 之后的 7 次点击（19:46 四次、20:07–20:08 三次）后端一次都没收到 → 不是请求被拒，而是前端根本没发出请求。

## 1. 复现（生产 spark-api，实测）

以该用户身份 `POST /spark-api/chat/{session_id}/turns`，问句 "Show me ISG FY2526Q4 total TCV"：

| 对象 | 结果 |
|---|---|
| 坏会话 `bba8fdb4-…` | HTTP 200，但 **1.66s 后 turn 变 failed**，`first_event_seq=0 / last_event_seq=0`，一个事件都没跑 |
| 评审者自己账号新建会话 | `completed`，81 个事件 |
| 该用户账号新建会话 `b1cf39d4-…` | `completed`，103 个事件，seq `[1,103]` |

失败 Turn（`f30b0db6-…`，20:19:46）的 runtime 返回：

```json
"last_error": {
  "type": "WorkspaceLifecycleError",
  "message": "Local Workspace is missing before its latest changes were persisted."
}
```

对照组说明：同一时刻、同一句问句，新会话正常；**只有这条 11-18 创建的老会话必失败**。与 token、权限、模型、数据源（Trino）无关。

## 2. 根因

spark 服务端为该会话分配的沙箱目录 `/home/app/workspaces/<session_id>` 已丢失（新建会话的 `workspace_remote_state` 为 `empty`、`workspace.is_temporary=true`）。每一轮 Turn 一启动即抛 `WorkspaceLifecycleError`，连一次模型调用都到不了（事件数 0），且服务端没有自愈重建逻辑。

## 3. 次生问题：reset 帧 + 全量重放

订阅坏会话事件流时，服务端先返回：

```
event: runtime_reset_required
```

随后是全量历史重放（实测 166626 字符 / 395 行 / 首个 `id: 605`，900+ 事件）；正常会话无此行为。

- `src/services/pbiRuntimeStream.ts:279` 声明了 `onResetRequired?: (meta: { cursor: number; latestSeq: number; truncated: boolean }) => void`，在 `:465-471` 触发（带 `truncated: true`），并让 `:473-482` 以 `source='reset'` 走 `onSnapshot` / `onRuntimeFrame`。
- 但 `AgentChatPanel` **从未消费 `onResetRequired`**，所以 reset 只表现为"整段历史被灌回来"。

## 4. 前端静默假死链路（点击被吞的真正原因）

1. `src/components/Agent/AgentChatPanel.vue:812` 与 `:1462` 都有 `if (loading.value) return` —— 发送入口的闸门。
2. `loading.value = false` 只有两处：`:1220`（onTerminal）、`:1228`（onFatalError）。
3. `src/services/pbiRuntimeStream.ts:590` 在连续 `MAX_STALLED_RECONNECTS = 3`（`:56`）次「连上即被关闭且无新业务帧」后直接 `this.stop()`，**既不调 `onTerminal` 也不调 `onFatalError`**。
4. 坏会话正好命中"reset + 重放帧、没有正常终态推进"的模式 → loading 永远停在 true → 后续所有点击被 `:812`/`:1462` 静默丢弃。刷新/重登都换不掉服务端那条会话，故症状持续。

相关实现细节（排查时有用）：
- `TERMINAL_STATUSES = new Set(['completed','idle','failed'])`（`:44`）；`isPbiRuntimeTerminal`（`:63-67`）要求状态属终态 **且** `queued_turn_count === 0`。
- `:53` `TERMINAL_GRACE_MS = 1200`：终态快照后不立刻断流，等 1200ms 消费重放帧再 `stop()`（`:535-541`）。
- `:50` `RECONNECT_DELAYS = [1000,2000,4000,8000,10000]`；`:47` `MAX_TRACKED_EVENT_IDS = 4000`。
- `:30-41` 控制帧 `runtime_snapshot` / `runtime_reset_required` 不推进游标、不进事件回调；出现子 Agent 事件时 `:494-503` 自动 `switchToRootSession()`。

## 5. 立即解法（已验证）

让用户**新建会话**重新提问，放弃 `bba8fdb4-…`。同一问句在新会话（`b1cf39d4-…`）已跑到 `completed`。若要减少历史损失，可把仍需要的消息复制到新会话。

## 6. 待修项（尚未动手）

1. **服务端**：会话 workspace 丢失后应自愈重建，而不是每轮抛 `WorkspaceLifecycleError`；至少把该错误翻译成用户可理解、可操作的提示。
2. **前端**：消费 `onResetRequired` 给用户明确错误气泡；在 `stop()`（`:590` 的停滞收流路径）里补一次 loading 兜底复位，避免整页静默假死。

## 7. 复现注意

- 复现会在目标会话里留下一条失败 Turn（`f30b0db6-…`），需向用户说明；自建的探测会话用完应删除。
- 判断"会话整体收尾"不要只看 status，还要看 `queued_turn_count`，否则有排队 Turn 的会话会被误判为终态。
