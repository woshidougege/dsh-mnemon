---
id: "6103c10c-0cbb-4290-bb14-56e938ec3b25"
title: "WorkBuddy2API PANEL 临期积分优先规则：选号加权 expiringUrgency + FEFO 扣减 + 会话粘性"
description: "Go 版 workbuddy2api-panel 的临期积分机制全貌：pool.prefer_expiring/expiring_soon 配置、expiringUrgency 加权公式与 Min1.5/Max20/Scale0.2/虚拟槽3 常量及 issue #140 根因、deductExpiryLocked 的 FEFO 分桶扣减、session_sticky 与前缀缓存按账号隔离的实测命中数据、embeddings/3D 开关判读，以及「跨域不共享权重」导致 intl 临期号无法被 cn 请求消耗的坑。"
status: "active"
created_at: "2026-11-19T10:14:07.721Z"
updated_at: "2026-11-19T10:14:07.721Z"
content_hash: "61f64d68a6246227d5a19004e5cbaf262472322895619112d5b33e2a59bb2153"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-df52d849-8cbe-4c2d-8332-4ebc85ae853c"
memory_body_ids:
  []
---

# WorkBuddy2API PANEL：临期积分优先使用的两套规则 + 会话粘性/功能开关判读

来源：2026-11-19 一次代码级答疑（用户问「有没有临期积分优先使用的规则」→ 展开读源码；随后问「这些是什么」→ 解读面板功能开关页）。仓库为 Go 版 `misakano7545/workbuddy2api-panel`。以下机制均由源码位置与常量佐证，不看源码无法重建。

## 1. 配置状态

`config.json` 的 `pool` 段：

```json
"pool": { "prefer_expiring": true, "expiring_soon": "168h" }
```

`prefer_expiring` 打开「最早到期优先路由」，`expiring_soon` 是快过期窗口 = 7 天。**两套规则同时生效，不是一套**：一套在选号时加权，一套在消费时扣减。

## 2. 规则一：选号时给临期账号加权（expiringUrgency）

`internal/pool/pick.go:584` 附近：

```go
frac := math.Sqrt(float64(remain) / float64(p.expiringWindow))
return expiringUrgencyMin + (expiringUrgencyMax-expiringUrgencyMin)*(1-frac)
```

常量在 `internal/pool/realm.go:26-37`：`expiringUrgencyMin = 1.5`、`expiringUrgencyMax = 20.0`、`expiringVirtualSlots = 3`（未注入窗口时的兼容固定倍率）、`expiringCreditScale = 0.2`。

语义：刚进 7 天窗口 → ×1.5；越逼近到期越趋向 ×20。函数形态取 `math.Sqrt` 而非线性，是为了让最后几小时才陡升——线性/√5 在 flip 用例里都压不过余额基数差（`internal/pool/expiring_urgency_test.go` 固化了倍率上下界与单调性）。

同时把窗口内账号的 **credits 项系数从 10 压到 2**（`expiringCreditScale = 0.2`），把「紧迫度」升为主轴、「余额多少」降为次轴。源码注释给出根因（issue #140 第 2 点）：余额基数恒主导临期决策——余量 500 分号的 credits 项是 10、50 分号只有 1，而紧迫度对「1 天 vs 6 天」能达到的比上限只有 ~8.4 倍，压不过 10:1 的基数差。

最终取值（`internal/pool/pick.go:621`）：

```go
return math.Max(normal, p.weightOfScaled(e, maxCredits, now, expiringCreditScale)*u)
```

**只加不减**：开开关不会让任何号比原来更少被选中。无窗口信息时 `expiringUrgency` 退回 `expiringVirtualSlots`（保持旧口径，`pick.go:589`）。

设计边界（`realm.go:15-25` 注释）：`Max` 必须 ≥ 基础权重天花板（1 + credits 项 10 + 闲置项上限 5 = 16），否则低余额临期号翻不了盘。已知极端情形仍可能压不过——高余额号同时闲置满载（权重 16）而低余额号刚用过（权重 2，无闲置补偿）。要连这种也压过需把 Max 提到 35，代价是批次临终几小时几乎独占流量，故默认不加。

## 3. 规则二：消费时按 FEFO 扣减（deductExpiryLocked）

`internal/pool/state.go:337-385`：`// deductExpiryLocked 按 FEFO 近似扣减到期快照：先从最早的自然日桶扣，扣穿再顺延。`

- 有 `expiryBuckets` 时逐桶扣减，扣穿的桶原地过滤掉（保持升序），随后用下一桶的 `ExpireAt`/`Remain` 重写 `creditsEarliestExpiry`/`creditsEarliestRemaining`，使两次余额刷新之间的首页快照仍随用量收敛。
- 无 buckets（旧状态/测试只写聚合字段）时退回只扣最早一批，`d >= creditsEarliestRemaining` 则归零并清空时间。
- `d <= 0` 直接返回；调用方必须已持有 `p.mu`。

结论：每花一分优先烧掉最早过期的那批，不是平均扣。

## 4. 规则三（同属选号域）：会话粘性路由

`config.json`: `session_sticky: {"enabled": true, "ttl": "30m", "gc_interval": "5m"}`。同一会话 30 分钟内固定绑同一个号，每 5 分钟清理过期绑定。

这条实际是省钱项，因为上游前缀缓存**按账号隔离**（cache_key 掺 uid）。`internal/server/sticky_derived_bind_test.go:1-4` 记录的实测：同一份 3.6k body 连发两次，只带 `user_id` 时落两个不同账号、**miss 3634/3634**；带 `conversation_id` 或什么都不带时第二轮**命中 3456/3634（95%）**。故 `session.ExtractKey` 对「只发 user_id」的客户端仍返回空（anti-monopoly 闸契约不变），但绑定改落到内容派生键 `session.DeriveCacheKey(body)`。

绑定键解析顺序见 `internal/session/ids.go`：`conversation_id`/`conversationId`（顶层或 `metadata` 内，metadata 优先、snake 优先于 camel），**只认 conversationId，绝不回落 user_id**（否则污染后台按对话聚合，issue #35）；缺失就不伪造、不发送。扫描走 `jsonscan.Lookup` 单遍零分配，走不通才回落整树解析。

绑定默认存本机（内存 + 状态文件），重启即失效；要跨机共享才需 Upstash Redis。

## 5. 面板功能开关三项的判读（`features` 段）

| 开关 | 当前 | 作用 | 判读 |
|---|---|---|---|
| `embeddings_enabled` | false | `/v1/embeddings`；上游嵌入模型与对话模型是两套独立注册表，实测算只有 `bge-m3`（1024 维）可用；关着回 `403 embeddings_disabled`（端点不消失，明确是未开而非挂了） | 保持关闭，对话场景用不到 |
| `three_d_enabled` | false | `/v1/3d/*` + MCP 工具 `generate_3d`/`get_3d_task`；关着回 `403 three_d_disabled`。另需 `three_d_auto_token`（默认 true）用账号 access token 调 `/v2/api-keys` 换 `ck_t_` 临时云凭证，签发失败且无手工凭证时回 `503 three_d_credential_missing` | 保持关闭 |
| `session_sticky` | true | 见第 4 节 | **唯一该开的**，cache 命中 81%~99% 主要由它撑起 |

## 6. 实测号池状态与一条跨域坑

当时 16 个号中 5 个在 7 天窗口内（`credits_expiring > 0`），其余最早 8.6 天后才过期：

| 域 | 余额 | 窗口内临期 | 最早到期 | 距今天 | 权重倍率 |
|---|---|---|---|---|---|
| intl | 340 | 150 | 11-20 03:59 | 0.4 天 | ×15.6 |
| intl | 340 | 180 | 11-20 04:04 | 0.4 天 | ×15.6 |
| cn | 4423 | 683 | 11-21 12:48 | 1.8 天 | ×10.7 |
| cn | 4614 | 164 | 11-26 10:17 | 6.7 天 | ×1.9 |
| cn | 5950 | 1802 | 11-26 11:31 | 6.7 天 | ×1.9 |

坑：倍率最高的两个号是 **intl 域**，而日常用的是 cn 域模型（如 `deepseek-v4.1-flash`）。**跨域不共享权重**——cn 请求选不到 intl 号，那批 330 分左右大概率原样过期（且 `credit_floor` 保底会再削掉可用余额）。要救只能当晚发 global 域请求（如 `global:deepseek-v4.1-flash-sg`、`global:glm-5.3-flash`）。这也说明「临期优先」的正确性受分池选号域约束：`AvailableUIDsForRealm` 先按 `e.a.Realm()` 过滤再计权（`internal/pool/realm.go:41`）。

## 7. 相关文档

- `00bd0694-cd83-4184-924d-e1517a8a6097`：PANEL 8789 媒体能力实测与「积分 0」误判根因（`schedule.balance_refresh_enabled` 关着导致 credits 从未采样；同处记录 `/v1/3d` 403、去水印默认 false）。
- `8ccc4628-b79d-4945-bfb3-d931035979c8`：WorkBuddy2API 便携版 2.1.3 升级记录与 Go 系分支调研（本仓库即 `misakano7545/workbuddy2api-panel`，1168★，MCP/embeddings/会话粘性/分级熔断）。
