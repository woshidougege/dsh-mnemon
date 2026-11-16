---
id: "425f80f3-74da-428d-a55a-7c777d812224"
title: "WorkBuddy2API 2.2.0 升级落地：更新源永久切至 M-strange 与「检查更新 403」根因（GitHub 匿名配额）"
description: "便携版网关从 2.1.3 升到 2.2.0 的实测结果与保留项（账号/Key/代理槽/媒体任务/uTLS 桥）、2.2.0 新增来源 IP 与归属地功能及 client_geo_enabled 开关、检查更新 403 的真实根因（GitHub 匿名 API 60 次/小时/出口 IP 耗尽，网页兜底对已删库失效）、WB_UPDATE_REPO 环境变量永久指向 M-strange 的代码链路与验证，以及「重启 8788 网关会切断自身工具调用」的操作纪律与重启脚本细节。"
status: "active"
created_at: "2026-11-16T03:05:03.593Z"
updated_at: "2026-11-16T03:05:03.593Z"
content_hash: "4b8909b6c931e75bf00582764e0d0bb2ca2b334da1d5d453527d630ded5a8e42"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-8ac6c466-2e51-45bb-94f2-986c60445a08"
memory_body_ids:
  []
---

# WorkBuddy2API 便携版 2.2.0 升级落地与更新源永久切换（含 GitHub 403 根因）

承接 `8ccc4628-b79d-4945-bfb3-d931035979c8`（2.1.3 升级记录）中遗留的两个未决项：**更新源三选一**与**是否拉 M-strange v2.2.0**。本文记录该决策的落地结果、一次「升级后检查更新仍 403」的根因定位，以及一条操作纪律（重启网关会切断自身工具调用）。状态：已升到 2.2.0 并稳定运行，更新源已永久指向 M-strange。

## 1. 升级执行结果（2.1.3 → 2.2.0）

从提交到验证完毕约 40 秒。升级后核对项：

| 检查项 | 结果 |
|---|---|
| 版本 | `Server: wb-proxy/2.2.0 Python/3.12.15`，`status.json` 为 `{"state":"succeeded","version":"2.2.0"}` |
| 进程 | 升级前 PID 31528，升级后 31656（后续再重启为 26748），命令行 `wb_proxy.py --port 8788 --lan` |
| 监听 | `0.0.0.0:8788` 不变 |
| 账号 | 16 个 / 14 就绪（cn 14、intl 2），与升级前一致 |
| API Key / 代理槽 | 3 个 key、slot-1（Clash Verge HTTP `127.0.0.1:7897`）及探测记录均保留 |
| 媒体任务 | `wb-proxy\\media\\tasks.json` 200 条，173,597 字节，升级未触碰 |
| uTLS 桥 | `wb-tls-bridge-windows-amd64.exe --listen 127.0.0.1:0 --parent-watch`，`bridge_generation: 2`，状态 ready |
| settings.json | 只读不重写，升级未改动任何键 |

2.2.0 新增 `wb_client_geo.py`（13,049 字节），用于请求来源 IP 与归属地；面板 HTML 中新增「来源 IP」「来源 IP 归属地」「账号 → 来源 IP」三处界面。功能默认**开启**（读配置时判定 `is not False`），未写入 `client_geo_enabled` 即视为开；要关闭需在 `wb-proxy\\accounts\\settings.json` 顶层加 `"client_geo_enabled": false` 并重载页面。usage 记录字段从原有集合扩展为 28 个，新增列含 `client_ip`（实测已写入 `127.0.0.1`）。本机与内网来源不发外部查询，无额外成本。

2.2.0 **未改动** `wb_accounts.py`、`wb_media_tasks.py`、`wb_pool_scope.py`、`wb_codearts.py`，故账号池 text/media 分区与 CodeArts 出口行为与 2.1.3 一致。

备份目录（`wb-proxy\\updates\\`）：`1.7.30`、`2.0.4`、`backup-2.1.3`、`backup-2.2.0`（本次自动创建）。另有 `.tmp\\wb-snapshot-2.1.1` 手工快照兜底。回滚命令与 2.1.3 记录中的写法相同，仅 `--version 2.1.3 --pid <当前PID>`。

## 2. 「检查更新 403」的真实根因

现象：升级到 2.2.0 后点「检查更新」仍报 403。直觉会归因于「老仓库被删」，实测否证——同一出口 IP 对**新旧仓库**、直连与走 Clash 代理，四种组合全部 403，响应体统一为：

```
{"message":"API rate limit exceeded for 31.59.135.188. ..."}
```

即 **GitHub 匿名 API 配额（60 次/小时/出口 IP）被耗光**，而该出口是共享代理节点，配额全节点共用。代码本体 `wb_updates.py` 已内置兜底（`fetch_releases`，函数说明在 `wb_updates.py:184` 附近交代设计意图）：API 那一跳失败（403 或连接被掐）就退回网页端点 `releases/latest`，读 urlopen 跟随重定向后的最终 URL 取 tag，不解析 HTML。判据差异：

- **对 M-strange**：API 403 → 网页兜底成功 → 得到 `v2.2.0`（两条网络路径均验证：直连、Clash 7897）。
- **对已删的老仓库**：API 403，网页兜底 404 → 代码 `raise api_error`，把 **403** 抛出。

所以 403 的含义是「配额耗尽 + 目标仓库不存在，兜底也救不回」，不是单纯的仓库删除信号。排查时注意：把 403 误读为「仓库不存在」会走弯路。

## 3. 更新源永久指向 M-strange

关键链路：`wb_updates.py:44` `REPOSITORY = configured_repository()`（读环境变量 `WB_UPDATE_REPO`，不匹配 `REPOSITORY_RE` 则回落 `DEFAULT_REPOSITORY = "ardeyouxipianyi/workbuddy2api-hub"`（`wb_updates.py:35`））；升级入口 `wb_proxy.py:26953` 传入 `repo=wb_updates.configured_repository()`。因此只要环境变量在，面板上「检查更新」与「立即升级」都会走新源，无需手动带 `--repo`。

已执行（用户级，写入注册表，升级替换文件不会丢失）：

```powershell
setx WB_UPDATE_REPO "M-strange/workbuddy2api-hub"
```

验证：重启网关后 `latest_version = 2.2.0`，`last_success` 更新为重启后的启动检查时间，说明启动检查也走新源。

遗留提示：匿名配额随时可能被邻居打光，但兜底会走网页端点，不影响可用性；若想彻底摆脱配额限制，加 `setx GITHUB_TOKEN <token>` 可把 60/小时提到 5000/小时（`wb_updates.py` 中 `TOKEN_ENV = "GITHUB_TOKEN"`，带 token 时走 `Authorization: Bearer`）。

## 4. 操作纪律：重启 8788 等于切断自身

代理网关跑在本机 `0.0.0.0:8788`，而**当前会话自身的出网请求也经过它**。执行「停进程 → 起新进程」期间，正在进行的工具调用会被掐断（实测出现一次 `Error: tool call aborted`，时点与重启吻合）。用户明确指出并要求注意。

可复用做法：

- 改动网关前，先把要跑的验证命令一次性备好，压短断连窗口；不要把「停服务」和「后续探索」放在同一串交互里。
- 重启确认尽量用只读请求（`health`、`status.json`、进程列表），一次取全。
- 若必须重启，先告知用户那一两秒面板会断。

重启脚本本身：`restart-wb-proxy.bat` 先杀掉所有 `wb_proxy.py` 实例再 `call start-wb-proxy-lan.bat`；`start-wb-proxy-lan-visible.bat` 只是设 `WB_LAUNCH_MODE=console` 后转调同一个 LAN 启动器（当时进程父级即 `cmd /c start-wb-proxy-lan-visible.bat 8788`）。注意 bat 必须纯 ASCII，且 `if(...)` 块内不能出现 `%VAR%`（含括号的路径会破坏 cmd 解析）。

## 5. 现状

版本 2.2.0，PID 26748，`0.0.0.0:8788` LAN 模式，健康检查 200，更新源 M-strange，客户端 IP 归属地功能开启。唯一还需人工决定的事项：是否加 `GITHUB_TOKEN` 以彻底规避匿名配额。
