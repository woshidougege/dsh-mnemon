---
id: "c28e03c1-9f82-401e-aae7-bea1cd218808"
title: "WorkBuddy PANEL 8789 实跑与单号迁移踩坑：auth 文件名 glob、积分保底 503、端点实测"
description: "PANEL(1789/桌面 WorkBuddy-Panel) 实际部署事实：api_key 沿用原值、auth_dir 指向桌面 auths、auth 文件名必须 workbuddy*.json（否则 loaded 0）、单号迁移后 22 模型与 /v1/search 打通、该号积分 0 导致付费模型 503 credit floor、/health 与 /panel/api/accounts 均 404 而 reload 需 POST、源码侧 normalizeExpiresAt 毫秒归一与静态 Key 不刷新逻辑、以及只切搜索/全量迁号两条待选路线。"
status: "active"
created_at: "2026-11-19T09:12:31.170Z"
updated_at: "2026-11-19T09:12:31.170Z"
content_hash: "f88ca36d5b135b663778c1d37f0d2f8e98c0f05cd8fb45a31eb487923067d56e"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-11e23e26-31e9-40c1-a19b-da4b373a8acc"
memory_body_ids:
  []
---

# WorkBuddy PANEL 8789 实跑与单号迁移踩坑（2026-11-19）

背景分支：本次是既有「HUB(8788, Python 便携) vs PANEL(8789, Go 单二进制) 检索通道对比」的落地执行。对比结论与 A/B/C 决策依据见 `a33b2a00-83e9-4029-b9c7-ac52472c9708`；2.1.3 升级与回退流程见 `8ccc4628-b79d-4945-bfb3-d931035979c8`。本文只记**实际部署事实、失败点与可复用操作步骤**。

## 1. 目标与硬约束（用户同轮提出）

- api_key 切完**不能变**（DSH 侧要改的地方太多）。
- 部署放**桌面**。
- 先起 **8789**、**只迁一个号**试；8788 HUB 全程不动，作为回滚面。

## 2. 实际部署形态

- 目录 `C:\Users\63588\Desktop\WorkBuddy-Panel`，二进制 `wb2api.exe`，监听 `127.0.0.1:8789`，驻留约 39.7MB。
- `config.json` 的 `auth_dir` 指向 `C:/Users/63588/Desktop/WorkBuddy-Panel/auths`；启动日志：`workbuddy2api listening on 127.0.0.1:8789 (api_key=0f****…**44, 来源 配置文件 config.json)` —— api_key 沿用原值，来源是配置文件而非环境变量。
- 迁移手法：把 HUB 的 `accounts/<uid>.json` 转成扁平 auth 文件丢进 `auths/`。
- 首个迁移号 `03c25064-00b4-4b22-9a03-e6566bce8637`（昵称"元"，realm=cn，platform=CLI）。该号在 HUB 里本就是 `enabled=false`，因此不存在 refreshToken 互斥。

## 3. 踩坑 1（关键）：auth 文件名必须匹配 `workbuddy*.json`

`auth.LoadDirReport` 用 glob 只认 `workbuddy*.json`。第一次命名成 `<uid>.json` 时：

- 启动日志 `loaded 0 account(s)`，`/v1/models` 返回 `{"data":[]}`；
- `POST /panel/api/accounts/reload?force=1` 返回 `{"ok":true,"pool":0,"result":{"files":0,...}}`。

改名 `workbuddy-<uid>.json` 后立刻 `loaded 1 account(s)`，`/v1/models` 返回 16329 字节、22 个模型（`cn:fast-model`、`cn:balanced-model`、`cn:deep-model`、`cn:glm-5.3`、`cn:hy4flash-free` …，全带 `cn:` 前缀）。

补充：目录轮询每 10s 一次，删除需连续 2 次确认才生效；改配置后重启进程才读新的 `auth_dir`。

## 4. 踩坑 2：账号本身积分为 0，付费模型 503

```
WARN: [pool] credit floor: sticky acct=元(03c25064) model=fast-model credits=0 < floor=100, unbind (paid model held out)
WARN: [pool] no selectable account realm="cn" model="fast-model" pool{total=1 healthy=1 ...} model_blocked=false —— 该域本请求快速失败（503）
```

不是配置问题，是号没额度。把 `pool.credit_floor` 调到 0 可去掉保底拦截，但真正的解法是换有积分的号。免费模型可用：`/v1/chat/completions` 用 `hy4-flash-free` 回 `ok`（models 列表里的 id 是 `cn:hy4flash-free`）。

## 5. 端点与路径事实（实测）

| 端点 | 结果 |
|---|---|
| `GET /v1/models` | 200，22 个模型 |
| `POST /v1/search` | 200，`{query,type,provider,results,total_results,response_time_ms}`；实测 5 条 / 1021ms，`results[].title/url/snippet` 正是 DSH 解析要的形状 |
| `POST /v1/chat/completions` | 可用（免费模型） |
| `/health`、`/v1/health` | **404，没有健康端点**，别拿它判断存活 |
| `GET /panel/api/accounts` | 404（该路径不存在） |
| `POST /panel/api/accounts/reload?force=1` | 200，`result{added,updated,removed,pending,parse_errors,files,skipped}` —— 热重载与"文件是否被识别"的诊断入口 |

面板：`http://127.0.0.1:8789/panel/`（key 沿用原值）。

## 6. 源码层面支撑"迁移无损"的几点

- **静态 API Key 永不刷新**：`a.isAPIKeyLocked()` 时 `NeedsRefresh` 恒 false。refresh 端点只认 OAuth 设备流；若放行会先 refresh、失败后把好号当坏号冷却换号 —— Key 本身是好的。
- **`Parse` 双形态**：嵌套 `{"auth":{...},"account":{...}}`（插件 OAuth 输出）与扁平 `{accessToken,refreshToken,expiresAt,domain,realm,uid,enterpriseId,nickname,device_token}`（手写/旧版）都吃；新版桌面端的 `$wbEncrypted` 信封由 `decryptWBEncrypted` 先解再按原两形态解析。
- **`normalizeExpiresAt` 毫秒归一**：官方 CodeBuddy CLI 凭据（`%LOCALAPPDATA%\CodeBuddyExtension\Data\Public\auth\*.info`）写毫秒（实测 1795254516410），本仓写秒；`expiresAtMillisThreshold = 100_000_000_000` 作分界，超出则 `/1000`。不归一会把到期推到公元 58800 年 → `NeedsRefresh` 恒 false → token 永不刷新 → 到期后整片静默 401。
- **realm 逃生门**：`SetGlobalEnabled(false)`（config `global.enabled=false`）时 `Realm()` 恒判 `cn`，用于锁死纯 CN 部署；但 `ResolveRealm` 不受其影响，避免把 global 账号写死成 cn 污染凭证。
- **出站指纹**：启动日志 `TLS 指纹复刻已启用（JA3 71dc8c533dd919ae9f4963224a4ba8fd / JA4 t13d0a1200_5d04281c6031_b76c5ee2507d）`，与 HUB 同为 uTLS 复刻思路，CLI 身份（`CLI/2.168.2 CodeBuddy/2.168.2` + `x-stainless-*`）近似同构。

## 7. 未决 / 下一步（截至本文）

1. **付费模型链路未验证** —— 需从其余 disabled 号里挑有积分的迁过去再测 chat。
2. **只切搜索**（低风险路线）：DSH 的 `searchBaseURL` 指到 8789、llm 仍留 8788，回滚只改两行。
3. **全量迁号**：逐个「HUB 侧 `enabled=false` → PANEL 侧启用」，最后再整体切 DSH。
4. 迁号顺序必须先去 HUB 禁用，否则 refreshToken 互斥。
