---
id: "e3c69c40-4850-43d7-9fa8-28a30bed9f40"
title: "WorkBuddy-Panel 本机运维：启动/重启脚本、开机自启现状与事件页为空的真因"
description: "Windows 单机部署 workbuddy2api-panel 的运维手册：wb2api.exe 必须在 WorkBuddy-Panel 目录以相对 -config 启动、桌面一键启动/重启 bat 与 start-panel.ps1 的设计约束（幂等、隐藏启动、验活必须带 api_key）、三处开机自启均无配置的实测结论、事件页恒空源于 logging.request_archive_enabled=false 且需重启生效、面板各端点实测响应与从 app.js 枚举 /panel/api/ 的方法、以及旧 HUB 抢 8788 端口与 update.proxy 7897 的排查要点。"
status: "active"
created_at: "2026-11-19T10:26:22.167Z"
updated_at: "2026-11-19T10:26:22.167Z"
content_hash: "1a6ddc0e1ba237014de5f8bf194a4548a47f05b6bcdfec8cf26ac9d086e819c8"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-b55abbcb-eeb2-45f5-b525-0643a10bbf64"
memory_body_ids:
  []
---

# WorkBuddy-Panel 本机运维：启动/重启脚本、开机自启现状、事件页为空的真因

来源：2026-11-19 的一次实操（用户问「电脑重启后怎么启动程序」→ 要求桌面一键启动 → 追问「面板数据是空的」）。部署在 Windows 桌面目录，非源码仓库内的开发环境，因此与源码级机制文档分开记录：临期积分/会话粘性的代码级机制见 `6103c10c-0cbb-4290-bb14-56e938ec3b25`（WorkBuddy2API PANEL 临期积分优先规则）。

## 1. 部署布局与启动前提

- 可执行文件：`C:\Users\63588\Desktop\WorkBuddy-Panel\wb2api.exe`
- 配置：同目录 `config.json`（另有 `config.example.json` 8006B，`config.json` 8318B），改动前备份为 `config.before-archive.json`
- 监听：`listen = 127.0.0.1:8788`
- 实际命令行：`"C:\Users\63588\Desktop\WorkBuddy-Panel\wb2api.exe" -config config.json`
- 关键约束：`-config config.json` 是**相对路径**，且 `auth_dir` 虽为绝对路径、`state_file` 为 `./data/state.json`，因此**必须在 `WorkBuddy-Panel` 目录下启动**；换目录启动会找不到配置并退出。

同一目录还有：`auths/`（账号目录，每 10s 轮询热加载）、`data/`（状态文件）、`panel.out.log`/`panel.err.log`（重定向日志，排查启动失败看 `panel.err.log`）、`switch-to-8788.ps1`。

## 2. 一键启动/重启脚本（本次交付物）

| 文件 | 位置 | 作用 |
|---|---|---|
| `启动 WorkBuddy 网关.bat` | 桌面 | 双击启动；已在运行则只提示，不重复起进程抢端口 |
| `重启 WorkBuddy 网关.bat` | 桌面 | 停进程 → 等端口释放 → 重新拉起 → 验活 |
| `start-panel.ps1` | `WorkBuddy-Panel\` | 启动逻辑本体（bat 是薄包装，bat 仅 174B / ps1 2247B） |
| `restart-panel.ps1` | `WorkBuddy-Panel\` | 重启逻辑本体 |

设计要点（改脚本时不要退化）：

1. **后台隐藏启动**：`Start-Process -FilePath <exe> -ArgumentList "-config","config.json" -WorkingDirectory <panel dir> -WindowStyle Hidden`，所以关掉黑窗口不影响网关。
2. **幂等**：先用进程/端口探测，若已在运行直接打印 pid 并跳过启动，避免第二个实例争 8788。
3. **验活必须带 key**：`/v1/models` 要鉴权，**不带 key 直接回 401**（曾因此误报「服务没起来」）。脚本改为从 `config.json` 的 `api_key` 取值再请求。
4. 成功输出形如：API `http://127.0.0.1:8788/v1`、面板 `http://127.0.0.1:8788/panel/`、模型 `49 个`；失败给三条排查建议（端口占用 / 看 `panel.err.log` / 手动跑 exe 看报错）。

## 3. 开机自启现状（2026-11-19 实测）

三处均无相关项，故重启后**不会**自动启动：启动文件夹为空（仅 `desktop.ini`）、`HKCU/HKLM Software\Microsoft\Windows\CurrentVersion\Run` 无 workbuddy 项、计划任务无匹配关键词。若要自启，可选启动文件夹放 bat、计划任务（可设延迟+失败重启+完全隐藏）、或 Run 键；尚未落地，用户当时只要求桌面可双击。

## 4. 事件页恒为空：`logging.request_archive_enabled`

根因不是采集失败，而是开关关着：

```json
"logging": { "request_archive_enabled": false, "request_client_info": true,
             "request_retention_days": 7, "request_archive_max_mb": 100,
             "panel_log_enabled": true, "panel_log_retention_days": 7, "panel_log_max_mb": 50 }
```

- `request_archive_enabled` 控制「是否把每次请求存成结构化事件」——关着时事件表恒空。
- 「运行日志」页那份带 `#001 … #135` 的文本流走的是另一个开关 `panel_log_enabled`（开着），所以两页对不上，这就是用户看到「两个页面对不上」的现象。
- `config.example.json` 该项默认 **`true`**，本机是 `false`（手动关过或旧版本默认值带过来）。
- 修复路径：`POST /panel/api/config` 提交整份 config（含 `config` 包装层），响应 `{"ok":true,"restart_required":["logging.request_archive_enabled"]}` —— **PANEL 自己标记该项非热生效，必须重启进程**。且只记录重启之后的请求，历史 135 条不补。
- 写操作的接口形态值得记住：读 `/panel/api/config` 返回 3653B，顶层键为 `api_key_env_managed, config, environment_managed_fields, ok, path`；回写要原样改 `config` 内的字段再整体 POST，并读回确认（`before: false` → `after: true`）。

## 5. 探测端点时的实测响应（避免把 404 当故障）

- `/v1/models`：无 key → **401**，带 key → **200**。这是验活正确姿势。
- `/panel/`：无鉴权即 **200**（面板页面免鉴权）。
- `/health`：**404**（该部署没有这个探针端点，别拿它当就绪判据）。
- `/`：404。
- `/panel/api/config`、`/panel/api/logs` 可用；当时按 guess 打的 `/panel/api/status`、`/panel/api/requests`、`/panel/api/events` 等路径返回 40x/404 —— 面板前端 `app.js` 里真实引用的是 `/panel/api/overview`、`/panel/api/models`、`/panel/api/model_probes`、`/panel/api/error_summary`、`/panel/api/tasks/live`、`/panel/api/logs/export`、`/panel/api/import/cockpit`。**要枚举面板接口，应从 `app.js` 里 grep `/panel/api/` 而不是猜路径。**

## 6. 端口冲突（启动失败的头号原因）

同机还留着旧 HUB 部署：`C:\Users\63588\Desktop\WorkBuddy-Portable\wb-proxy`，启动方式 `python wb_proxy.py --port 8788`。它和 PANEL 抢同一个 8788 端口，后启动的一方起不来。HUB 以前靠手动启动，只要不主动运行就不会冲突；排查启动失败时先查这一点。

## 7. 环境旁证（与网络出站相关）

- 进程环境变量 `HTTP_PROXY/HTTPS_PROXY/NO_PROXY` 均为空。
- 系统 IE 代理 `ProxyEnable = 0`，但 `ProxyServer` 值仍残留 `127.0.0.1:7897`，`ProxyOverride` 含 `localhost;127.*;...;<local>`。
- `config.json` 的 `update.proxy = http://127.0.0.1:7897`、`update.auto_enabled = true`、`auto_time = 04:00`：即自动更新走 7897 出口，而本地 127.0.0.1 调用走直连（ProxyOverride 生效）。排查「更新下载失败但 API 正常」时要区分这两条路径。
