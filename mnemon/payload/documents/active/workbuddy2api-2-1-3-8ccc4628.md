---
id: "8ccc4628-b79d-4945-bfb3-d931035979c8"
title: "WorkBuddy2API 便携版 2.1.3 升级记录：校验流程、回退路径与活跃分支调研"
description: "便携版 Python 网关升级到 2.1.3 的实测部署事实、SHA256/文件对比前置校验流程、两条回退路径与 rollback 命令、2.1.x 行为变化（/panel 404、默认 Go TLS 桥）、更新源指向已删库的未决问题，以及 GitHub 活跃分支调研与迁移成本判断（M-strange v2.2.0 为同源续命分支）。"
status: "active"
created_at: "2026-11-16T02:53:09.990Z"
updated_at: "2026-11-16T02:53:09.990Z"
content_hash: "f0425cd5b5e809394850f3c61377fda0aaafb3640487adade3060cfd7b362b86"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-27d27280-5c22-4c50-8a4e-c32dea41bca4"
memory_body_ids:
  []
---

# WorkBuddy2API 便携版：2.1.3 升级记录、回退路径与活跃分支调研

对象：腾讯 WorkBuddy 账号池 → OpenAI 兼容网关的**便携版（Python 实现）**，跑在用户本机。

## 1. 部署事实（升级后实测，2026-11-16 10:46）

- 根目录：`C:\Users\63588\Desktop\WorkBuddy-Portable`，网关代码在 `wb-proxy\`，解释器 `wb-proxy\python\python.exe`（Python 3.12.15）。
- 启动方式：`python.exe wb_proxy.py --port 8788 --lan`，监听 `0.0.0.0:8788`（**局域网可达，升级后保持不变**）。
- 升级后 PID 31528（10:46:27 启动），常驻内存约 165 MB。
- 账号：磁盘上 21 个账号文件，健康接口报 16 个（cn 14 就绪 / intl 2 但 0 就绪），模型 `deepseek-v4.1-flash` 可用；`/v1/models` 返回 20 个模型。`api_key_required: true`。
- 媒体：`media.enabled`，tasks.json 200 条任务、约 173 KB，升级后**全部保留**；`video_concurrency: 3`。
- `update_check_enabled: true`（见 §5 的问题）。

## 2. 升级到 2.1.3 的做法与验证

不走「解压覆盖」，走程序自带的更新器 `wb-proxy\wb_update_apply.py`（detached helper，源码见同目录 `wb_update_portable.py`）：下载 → 校验 SHA256 → 备份 → 停网关 → 替换 → 重启 → 健康检查，**任一步失败自动回滚**。参数：`--root`（wb-proxy 目录）、`--package-root`（便携版根）、`--version`、`--port`、`--lan`、`--pid`、`--repo`、`--keep-backups`（-1 表示读面板设置），`--action install|rollback`。整个过程 28 秒，状态写入 `wb-proxy\updates\status.json`（本次 `{"state":"succeeded","version":"2.1.3"}`），网关实际中断约 10 秒（10:46:17→10:46:27）。

动文件前做的独立校验（可复用为「来历不明仓库的升级前置检查」）：
1. 从镜像下载 `workbuddy2api-hub-v2.1.3.zip`（23.18 MB）+ `SHA256SUMS`；
2. 本地算哈希 `0dced47e6b24e6c3004a41dcf4ffa50b8dfa18c14aa9df99ec24d2359890f131`，与官方清单一致；
3. 解压逐字节对比 **53 个程序文件**：42 个相同，10 个改动（`dashboard.html` 1303754→1397986、`wb_accounts.py`、`wb_activity.py`、`wb_codearts.py`、`wb_logstore.py`、`wb_proxy.py` 1324769→1369784、`wb_settings.py`、`wb_tasks.py` 64769→100224、`wb_updates.py`、`wb_window_probe.py`），**新增 1 个 `wb_time.py`**（时区辅助，读过，干净）；
4. 扫描全部 Python 源码外部域名：只新增 `open.workbuddy.cn`（腾讯自家），无远程 eval / 可疑上报。

## 3. 2.1.x 的行为变化（踩坑点）

- **面板入口变了**：`/` 返回 200 直接就是面板；`/panel` 与 `/panel/` 现在 **404**；`/console` 返回 401。老书签会以为面板没了。
- `/accounts` 等管理接口需要**面板密码**，无凭据时返回 `401 \{"error":{"message":"panel password required"}}`。
- 新版默认启用内置 **Go/uTLS TLS 指纹桥**：`wb-proxy\bin\wb-tls-bridge-windows-amd64.exe --listen 127.0.0.1:0 --parent-watch`（PID 29056，约 22 MB），健康接口 `outbound_tls.bridge_state: ready / last_route: bridge`，`bin/` 下另有 darwin/linux × amd64/arm64 五个二进制。它只负责伪造 TLS 指纹，**不是网关本体**，所以此前「Go 版对性能没提升、瓶颈在上游」的实测结论依然成立。

## 4. 回退路径（两条，都已就位）

1. 更新器自留：`wb-proxy\updates\backup-2.1.3\`（内容为 2.1.1 的文件）；另有 `updates\2.0.4`、`updates\1.7.30` 历史目录。
2. 手工快照：`C:\Users\63588\public\.tmp\wb-snapshot-2.1.1\`（44 个文件）。

回退命令：

```powershell
& "C:\Users\63588\Desktop\WorkBuddy-Portable\wb-proxy\python\python.exe" `
  "C:\Users\63588\Desktop\WorkBuddy-Portable\wb-proxy\wb_update_apply.py" `
  --action rollback --root "C:\Users\63588\Desktop\WorkBuddy-Portable\wb-proxy" `
  --package-root "C:\Users\63588\Desktop\WorkBuddy-Portable" `
  --version 2.1.1 --port 8788 --lan 1 --pid <当前PID>
```

## 5. 未决问题：更新源指向已删仓库

`wb_updates.py:35` 仍是 `DEFAULT_REPOSITORY = "ardeyouxipianyi/workbuddy2api-hub"`（原作者的库已删），可用环境变量 `WB_UPDATE_REPO` 覆盖（须匹配 `REPOSITORY_RE`，否则回落默认值）。因此更新检查会静默失败。三个候选处置（用户当时未选定）：关掉更新检查 / 在 `Start-WorkBuddy.bat` 里 `set WB_UPDATE_REPO=niyouqianma/workbuddy2api-hub` 指向镜像 / 不管它。

## 6. 活跃分支调研（2026-11-16，GitHub 共 161 个相关仓库）

**同源续命分支——唯一在同一 Python 代码基线上继续开发的：**

- `M-strange/workbuddy2api-hub`（Python，66★，v2.2.0 发 11-13）：**合并上游 v2.0.4–v2.1.3（110 个文件，+15394/−1612）**，并保留自有增强：请求来源 IP 全链路、面板「来源 IP」聚合视图与按地区搜索、来源 IP 归属地（`wb_client_geo.py`）、账号池排队等待、池子超半数熔断自动探活、上游超时设置、listen backlog 1024。目录结构与账号/面板/池子分区可沿用，迁移成本≈0。
- `niyouqianma/workbuddy2api-hub`（Python，100★）：纯镜像搬运上游最后版本，无自研提交，只作存档。

**Go 系（活跃但迁移成本高）：**

- `misakano7545/workbuddy2api-panel`（1168★，3 人协作，最后提交 11-16 01:35，release v2026.11.15.2246）：单文件二进制，MCP/embeddings/会话粘性/分级熔断。
- `yreidev/workbuddy2api-panel`（97★，AGPL-3.0，测试覆盖 98.8%，请求头对齐官方桌面端 5.9.3，HTTPDNS 主备切换）。
- `xu20060420fan-cell/workbuddy2api-trae-qoder-minimax-panel`（78★）：多平台合集（WorkBuddy/CodeBuddy/Trae/Qoder/MiniMax/豆包/华为云 CodeArts），签到与成长任务一键完成 17/18。
- `jin66god/workbuddy2api-panel`（33★）：单人高频更新，无 release。
- **反例**：`linguo2625469/workbuddy2api-panel` 星最多（4871★）但停更于 10-21——星数不能当活跃度。

**结论与理由**：留在 2.1.3（用户已实测 Go 版「差距不大」）；要新功能优先试 M-strange v2.2.0（同源，两条命令可切）；Go 版不划算，因为要改 16 个号的名字、迁设备指纹、池子会分裂。所有这些都是被封项目的分叉，随时可能再消失，本地代码是唯一保底。

## 7. 待办（用户未决）

- 更新源三选一（§5）。
- 是否拉 M-strange v2.2.0 与本地 2.1.3 做完整 diff（文件改动、新增网络请求、可疑代码）——用户尚未答复。
