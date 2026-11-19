---
id: "00bd0694-cd83-4184-924d-e1517a8a6097"
title: "WorkBuddy PANEL 8789 媒体能力实测与「积分 0」误判根因：生图/视频/MCP 端点与余额刷新开关"
description: "修正前一份 PANEL 部署记录的归因错误：付费 503 不是账号没额度，而是 schedule.balance_refresh_enabled 关闭导致 credits 从未采样（开启后读到 2354 即通）；并给出本机 8789 媒体端点的真机实测——文生图 cn:hunyuan-image-alpha 10s/6.46 积分、视频 cn:minimax-video-h3 4s 有声 113s/64.57 积分（均高于 README 宣称值约 13%）、/mcp 5 个工具而 HUB /mcp 404、/v1/3d 403 默认关、去水印开关默认 false、签名直链时效与改图 URL 回投坑、DSH 接 /mcp 的配置形状。"
status: "active"
created_at: "2026-11-19T09:23:28.139Z"
updated_at: "2026-11-19T09:23:28.139Z"
content_hash: "56252d78886d4fbc1e2fb6973eabbe2d90b8fc0ea3a2aef0fba1225b6ac8d020"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-ddb0ce5d-7add-44c7-917c-509f1d12fe77"
memory_body_ids:
  []
---

# WorkBuddy PANEL 8789：媒体能力实测与「积分 0」误判的根因

对象：桌面部署的 PANEL（Go 单二进制 `wb2api.exe`，`C:\\Users\\63588\\Desktop\\WorkBuddy-Panel`，监听 `127.0.0.1:8789`）。
关联文档：部署形态、auth 文件名 glob 等踩坑见 `c28e03c1-9f82-401e-aae7-bea1cd218808`；HUB（Python 便携版，8788）的检索通道对比见 `a33b2a00-83e9-4029-b9c7-ac52472c9708`；HUB 2.1.3 升级见 `8ccc4628-b79d-4945-bfb3-d931035979c8`。
本文只记两件新事实：**（A）一次错误归因的更正**、**（B）媒体生成能力的真机实测结果**。

## A. 更正：不是「这个号积分是 0」，是余额刷新被关掉了

前一份部署记录把付费模型 503 归因为「该号本身没额度（`credit floor: sticky acct=元 credits=0 < floor=100`）」。**这个结论是错的。**

真实链路：

1. 初次配置时为了「别乱动账号」，把定时任务整批关掉，包含 `schedule.balance_refresh_enabled`（余额后台刷新，默认 5 分钟一次）。
2. 从没查过余额 → 池里该账号的 `credits` 一直停在初始值 0。
3. `pool.credit_floor`（默认 100）拿 0 去比 → 0 < 100 → 所有付费模型被池路由挡掉，返回 503。
4. 打开 `schedule.balance_refresh_enabled=true` 并重启后，启动日志立刻出现 `balance-sample 03c25064-…: credits=2354`，付费模型 `cn:glm-5.3-flash` 直接回 `ok`。

**可复用的排查规则**：PANEL 上出现「付费模型 503 / 池里 credits=0」时，先确认 `schedule.balance_refresh_enabled` 与 `pool.credit_floor`，再看账号本身；`credits=0` 很可能是**未采样**而不是**真没钱**。相关配置项：`pool.credit_floor=100`（低于此值挡单，0=关闭）、`pool.reserve_credits=10`（保留积分，防余额被用到 0）。

## B. 媒体生成：端点、实测耗时与真实扣费

PANEL 把上游媒体模型包成 OpenAI 形态接口，并额外提供 `POST /mcp`（Streamable HTTP 的纯 POST JSON-RPC，无 SSE、无 session；`GET /mcp` 回 405 + `Allow: POST`）。

### B1. 端点存在性（本机 8789 逐个探测）

| 端点 | 探测结果 | 判读 |
|---|---|---|
| `POST /v1/images/generations` | 400 `invalid_request: prompt is required` | 端点可用，仅缺参数 |
| `POST /v1/images/edits` | 400 `图生图需要 image（data URL / 公网 URL / multipart）` | 端点可用 |
| `POST /v1/videos`、`POST /v1/videos/generations` | 400 `prompt is required` | 端点可用，互等价 |
| `POST /v1/3d/generations` | 403 `three_d_disabled` | 端点存在，`features.three_d_enabled` 默认关 |
| `POST /mcp` | 5 个工具可见 | HUB **没有**这个端点（`/mcp` → 404） |

`/v1/models` 共 22 个模型，其中 4 个带媒体模态：`cn:hunyuan-image-alpha`（out=image）、`cn:hunyuan-image-alpha-edit`（out=image）、`cn:kling-video-v3`（out=video）、`cn:minimax-video-h3`（out=video）。MCP `tools/list`（3D 与去水印关着时）返回：`list_generation_models`、`generate_image`、`edit_image`、`generate_video`、`get_video_task`。

### B2. 真机跑通的结果

| 能力 | 模型 | 结果 | 耗时 | 实际扣费 | README 宣称 |
|---|---|---|---|---|---|
| 文生图 | `cn:hunyuan-image-alpha` | 1024×1024 PNG，落盘 `C:\\Users\\63588\\Desktop\\WorkBuddy-Panel\\test-output\\cat.png`（206,689 B） | 10 s | **6.46** 积分 | 5.71 |
| 出视频 | `cn:minimax-video-h3` | 4 s / 16:9 / 有声，`status=completed`，落盘 `cat.mp4`（1.17 MB） | 113 s | **64.57** 积分 | 57.14 |

结论：**扣费按上游实际值走，比 README 示例高约 13%**，预算估算要用实测值而非文档值。视频接口缺省 `wait=true` 会阻塞到出片；kling 同规格要 8 分钟量级，客户端超时需给到 15 分钟以上，或传 `wait=false` 取 `task_id` 轮询。

### B3. 落地形态与坑

- 回执里的图/视频都是**云存储签名直链**，网关不转存字节：图约 24 h、视频约 12 h 有效。要长期保存必须自己下载落盘（本次已落盘，故截图/播放可用）。
- 上游默认带「AI生成 WORKBUDDY」水印（视频 `watermark: true`）。可选关闭，或走 `POST /v1/images/strip-watermark` 后处理——该功能由 `features.watermark_strip_enabled` 控制，**默认 false**（要把成品图回读、解码、再 base64 回传，有内存代价）。
- 改图的输入图必须是上游取得到的地址：拿「刚生成那张图的签名直链」直接回投会得到 `invoke model error`，用 `data:` URL 或自己可控的公网地址最稳。
- 任务台账在**进程内存**里，重启后查询退化为有界探测（命中即收养）；记账只用一把锁记一次，重复/并发查询不会重复扣费。
- 单租户前提：全网关共享一个 api_key，台账只记「提交账号」不记「调用方」，持有 key 即可查任意 `task_id`，多租户场景不可直接暴露。

### B4. 与 HUB 8788 的能力差

HUB：`/v1/mcp` → 404；`/v1/images/generations` 与 `/v1/videos` 回 400（仅参数错），`/v1/videos/generations` → 404；无 3D、无去水印。PANEL 多出：`/v1/videos/generations`、任务查询与成品下载、`/mcp` 工具集、3D（默认关）、去水印（默认关）。

### B5. 把媒体能力接进 DSH 的方式（未执行，留作选项）

PANEL 的 `/mcp` 可直接配给支持 HTTP MCP 的宿主，配好即可由 agent 直接调 `generate_image` / `generate_video`：

```json
{"mcpServers":{"wb2api":{"type":"http","url":"http://127.0.0.1:8789/mcp","headers":{"Authorization":"Bearer <api_key>"}}}}
```

## 待办 / 未执行项

- 尚未决定是否把 PANEL 的 `/v1/search` 或整体 baseURL 切给 DSH（现有两条路线：只切搜索、或全量迁号后整体切）。
- 剩余 14 个号仍在 HUB 侧 `enabled=false`，未评估余额与积分有效期。
- 水印关闭版未出（用户问过「要不要关水印再出一版」，未答）。
