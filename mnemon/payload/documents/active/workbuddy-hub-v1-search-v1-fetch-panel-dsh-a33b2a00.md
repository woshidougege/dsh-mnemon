---
id: "a33b2a00-83e9-4029-b9c7-ac52472c9708"
title: "WorkBuddy 网关联网检索通道对比：HUB 无 /v1/search 与 /v1/fetch，PANEL 有且可对接 DSH"
description: "HUB（Python 便携版）实测 /v1/search 与 /v1/fetch 均 404；PANEL（Go 单二进制 fork）内置 agenttool/v1/search 与 webfetch 直通并对 DSH UA 做 DeepSeek Search API 兼容；DSH 侧 fetch 解码只要 {content:string} 的源码证据与 searchBaseURL/fetchBaseURL 配置字段、账号格式迁移成本（auths/ vs accounts/*.json）、上游删库与停维护风险，以及 A/B/C 三条待决策路线。"
status: "active"
created_at: "2026-11-19T09:02:01.109Z"
updated_at: "2026-11-19T09:02:01.109Z"
content_hash: "cba3b764b5293eddbaa5d5112d798786a309332379c092df8efc645e38807714"
source_paths:
  - ".tmp/asar/dsh__node_modules__@deepseek-ai__dsh-web-deepseek-api__src__fetch.ts"
session_ids:
  - "mnemon-review-v1-b9a07ddf-50b7-4567-ad6b-9dcc03048f4b"
memory_body_ids:
  []
---

# WorkBuddy 反代网关的联网检索通道：哪条分支真有 /v1/search 与 /v1/fetch

调查时间 2026-11-19。目标：让 DSH 的核心 `web_search` / `web_fetch` 跑在本机 WorkBuddy（腾讯 CodeBuddy）账号池上，不额外给 DeepSeek 充值。

与既有文档的关系：`02486bfd-f172-41a6-afeb-9d2fc6fd39e1`（DSH 搜索链路诊断）当时的结论是「WorkBuddy 反代无法转发，因为没有 DeepSeek 上游 endpoint 与凭据」。**该结论只对当时看到的那两个网关成立**：Python 便携版 HUB，以及一个只有 `wb2api.exe` 无源码的 Go 目录。本次发现 PANEL 分支已内置官方 `agenttool` 直通，故结论需按分支限定。分支沿革与升级历史见 `8ccc4628-b79d-4945-bfb3-d931035979c8`。

## 1. 实测对照

| 端点 | HUB（当前在跑） | PANEL |
|---|---|---|
| `POST /v1/search` | **404** | 存在，查询免费 |
| `POST /v1/fetch` | **404** | 存在，返 `{"content":"<markdown>"}` |
| 对话协议 | `/v1/chat/completions` | 同上 + `/v1/responses` + `/v1/messages` |

当前 HUB 进程：`PID 26748`，`C:\\Users\\63588\\Desktop\\WorkBuddy-Portable\\wb-proxy\\python\\python.exe wb_proxy.py --port 8788 --lan`。健康快照：16 个账号（cn 14 / intl 2），就绪 3（cn 3 / intl 0），`deepseek-v4.1-flash` cn 2，`api_key_required: true`，media 任务 200 条（completed 195 / failed 5）。

## 2. PANEL 是什么

- 仓库 `misakano7545` 的 fork（本次核到 1329★，2026-11-19 当天有更新，release `v2026.11.19.901`）；Go 单二进制 `wb2api.exe`，内嵌单页面板。
- 链路：`Sliverkiss/workbuddy2api`（已删库）→ `linguo2625469/workbuddy2api-panel`（2026-10-21 宣布停维护）→ 本仓。**中间每一环都无官方支持与后续保障**，本仓自称自用增强分支。
- 维护披露：issue/PR 处置、审查、测试由 AI 编码 agent 辅助完成；维护者是学生，更新随缘、无 SLA。
- 本地已有一份 PANEL 目录：`C:\\Users\\63588\\Desktop\\workbuddy2api-panel-go\\`（仅 `wb2api.exe` / `wb2api.official.exe` / `config.json` / `README.md` / `auths\\` / `data\\`，无 Go 源码）。

## 3. PANEL 的检索端点语义

- `POST /v1/search`：透传官方 `{base}/agenttool/v1/search`，body `{query, max_results}` → `{results[].title/url/snippet/date, total_results}`。对 UA 含 `deepseek-harness` 的客户端启用 **DeepSeek Search API 兼容**：接受 `count` 作为条数，`date` 缺失补 `null`、`title`/`snippet` 缺失补空串；其余客户端按上游原文逐字节透传。
- `POST /v1/fetch`：`{"url":"https://…"}` → `{"content":"<markdown>"}`，透传官方 `agenttool/v1/webfetch`。
- body 加 `"deep": true` 改走官方综述式检索 `agentic_search`（服务端多轮检索 + 抓页 + 带引用综述），返回 `synthesis.content` + 整页正文 + `statuses` + `partial` 降级标记。**实测单次 18~66s、prompt ~48K tokens**，故默认走 snippet 快路径（~1s）。
- 另有服务端联网工具：`/v1/messages` 与 `/v1/responses` 接受 `{"type":"web_search_20250305","name":"web_search"}`（Responses 侧为托管 `{"type":"web_search"}`），网关先让模型自写检索词、执行检索、注入上下文再走既有流式管道；Anthropic 侧按协议回 `server_tool_use` + `web_search_tool_result`（不回退成客户端可执行 `tool_use`）。

## 4. DSH 侧对接契约（源码证据）

提取副本：`.tmp/asar/dsh__node_modules__@deepseek-ai__dsh-web-deepseek-api__src__fetch.ts`（源码在 `app.asar/dsh/node_modules/@deepseek-ai/dsh-web-deepseek-api/src/fetch.ts`）。

- `DeepSeekFetchApiProvider.fetch()` 用 `dispatchDeepSeekWeb(options.auth, { path: '/fetch', body: { url: request.url }, decode: … })`；decode 判定为

  ```ts
  if (!isRecord(payload) || typeof payload.content !== 'string') throw invalidResponse()
  return { url: request.url, statusCode: 200, body: { kind: 'text', content: payload.content }, truncated: false }
  ```

  即 **只要 `{content: string}`**，与 PANEL 的 `/v1/fetch` 完全对齐；搜索侧只需 `results[].url/title/snippet/date`，PANEL 对 DSH UA 自动补齐。
- 配置项（`src/index.ts` 附近，共 389 行）：`searchBilling`（`auto|account|api-key`，默认 `auto`）、`searchBaseURL`、`searchApiKeyEnv`、`fetchBaseURL`、`fetchApiKeyEnv`。base 后面追加 `/search` 与 `/fetch`；只设 `searchApiKeyEnv` 而缺 base 时用 `DEEPSEEK_API_KEY`。
- 现有 `cordis.patch.yml`（460 行）注册情况：`- id: web-search-free`（行 177）、`- id: web`（258）带 `searchProvider: deepseek-search-api` / `fetchProvider: deepseek-fetch-api`（260–261），并有注释说明「让核心 web_search 走 dsh-free-search 的引擎池」需同时保留 `fetchProvider: http`，因为 patch 会整条目替换 config；另见 `fetchProviderTakeover: true`（236）。

对接后要配的两组（**未执行**，等用户确认）：

```yaml
- id: web-deepseek-api
  config:
    searchBaseURL: http://127.0.0.1:8788/v1
    searchApiKeyEnv: WORK_BUDDY_API_KEY
    fetchBaseURL: http://127.0.0.1:8788/v1
    fetchApiKeyEnv: WORK_BUDDY_API_KEY
```

base 用 `/v1`，追加后正好是 `/v1/search` 与 `/v1/fetch`；key 用 PANEL 自己 `config.json` 生成的，不是 HUB 的（HUB 上这两个端点本来就是 404）。

## 5. 迁移成本与风险

- **账号格式不通**：PANEL 用 `auths/` + 浏览器 OAuth 设备授权；HUB 是 `accounts/*.json`（UUID 命名），字段含 `uid, nickname, domain, realm, platform, product, enterpriseId, accessToken, refreshToken, expiresAt, addedAt, source, apiKey, proxySlot, proxy, enabled, tasksEnabled, weight, note, group, disabledModels, poolScope, accountBanned, bannedAt, restrictedUntil, credits, tier, lastCheckin, lastDailyChat, lastActivityReport, modelCaps`。16 个号需重新授权或写转换脚本。
- HUB 侧 `accounts/` 里目前 8 个 UUID 文件可直接读，凭据字段齐全（值不落文档）。
- **风险**：上游原始仓库已删库并停更，PANEL 上游亦于 2026-10-21 停维护；两家都是非官方反代，账号风控/封禁风险由使用者承担；PANEL 作者声明「上游随时可能彻底收紧导致通道不可用」。

## 6. 待决策（用户尚未答复）

- **A（省事）**：不换网关，`searchProvider` 改 `ddg`、`fetchProvider` 改 `fs-quality-fetch`。0 成本、立刻恢复搜索，质量一般。
- **B（彻底）**：迁到 PANEL，DSH 搜索走账号池官方检索，质量最好，但要挪账号 + 改配置。
- **C（折中，已提出）**：先把 PANEL 拉下来在别的端口试跑再决定。

## 复用提示

- 判断某个 WorkBuddy 分支是否支持 DSH 联网，**先打 `POST /v1/search` 与 `POST /v1/fetch` 看是否 404**，比读 README 快；HUB 与 PANEL 在同一端口号（8788）上表现相反，别把端口当判据。
- 别再把「WorkBuddy 反代没有检索能力」当通论——按分支看。
