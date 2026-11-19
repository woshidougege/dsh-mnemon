---
id: "02486bfd-f172-41a6-afeb-9d2fc6fd39e1"
title: "DSH 搜索链路诊断：官方搜索靠账号 token（searchBilling）+ 余额 402，WorkBuddy 反代无法转发"
description: "dsh-web-deepseek-api 的搜索链路与 searchBilling(auto/account/api-key) 语义、x-dsh-auth-token 与 inferenceOrigin 约束、401/402 实测区分、localhost:8788 无 /v1/tools/search（上游为腾讯 CodeBuddy 非 DeepSeek）、相关 bundle entry 与源码位置，以及 A/B/C 三条路线与未决状态。"
status: "active"
created_at: "2026-11-19T08:56:34.265Z"
updated_at: "2026-11-19T08:56:34.265Z"
content_hash: "2d07d2dbf623e772c2b537cf253c85f89ae52c49c8b6dc34922279c754f87893"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-ab49a278-822b-486a-b344-1b0f08e2888a"
memory_body_ids:
  []
---

# DSH 搜索链路诊断：官方搜索要靠账号 token + 充值，WorkBuddy 反代无法转发

调查触发：用户问「我的 DeepSeek harness 现在搜索走的是啥，是官方的搜索吗？我是 workbuddy 反代出来的，支持官方搜索吗」。结论用于以后回答同类问题，不必重跑实验。

## 结论（可直接复用）

1. **官方搜索在 DSH 里是一条独立链路**：插件 `@deepseek-ai/dsh-web-deepseek-api` 直连 `https://api.deepseek.com/v1/tools/search`（fetch 为 `/v1/tools/fetch`），与模型走哪个 provider **无关**。
2. **走账号 token 技术上完全可行**，由 `searchBilling` 决定；但**本机实测被余额挡住**：账号 token 鉴权通过 → **HTTP 402 Insufficient Balance**，不是 401。
3. **WorkBuddy 反代（`localhost:8788`）转发官方搜索不可行**：它上游是**腾讯 CodeBuddy（`copilot.tencent.com`）**，不是 DeepSeek。实测 `http://localhost:8788/v1/tools/search` → **404**，端点表里没有 tools/search，也没有可转发的 DeepSeek 上游凭据。

## 关键配置与语义

`dsh-web-deepseek-api` 源码里的开关：

```ts
searchBilling: z.union(['auto', 'account', 'api-key']).default('auto')
```

- `auto`：跟随会话路由，**只有模型路由是 `deepseek-account` 时**才用账号 token。本机模型路由是 work-buddy，所以永远轮不到 → 这也是"装了搜索插件但到底怎么搜索"的直接答案。
- `account`：强制用已登录的 DeepSeek 账号 token，与模型路由无关。
- `api-key`：用 `searchApiKeyEnv` 指向的环境变量凭据。

两个必须记住的坑：

- 账号 token 通过 **`x-dsh-auth-token`** 头发送；`resolveToken()` 只把 token 交给 `inferenceOrigin`（默认 `https://api.deepseek.com`），所以该端点正好在允许范围内，不会被拒。
- 一旦同时设置 **`searchBaseURL` + `searchApiKeyEnv`**，就**固定成 API key 模式、永远不发账号 token**，等于把账号 token 这条路关掉。
- 切到 `account` 后若余额不足，**不会自动回退**到免费引擎池（free-search / ddg / exa 等），只会稳定 402。

## 实测证据（本机，2026-11-19）

| 场景 | 结果 |
|---|---|
| `DEEPSEEK_API_KEY`（尾号 `ea85`）直打 `/v1/tools/search` | **HTTP 401** Authentication Fails, your api key is invalid |
| 同一 key 打 `/v1/tools/fetch` | **HTTP 401**（同一个失效 key） |
| 从凭据 `deepseek-account-platform/default` 取账号 token，加 `x-dsh-auth-token` 打 `/v1/tools/search` | **HTTP 402** `{"error":{"message":"Insufficient Balance","type":"insufficient_balance_error"}}` —— 401→402 说明 token 本身有效、鉴权通过，只是账号余额为 0 |
| `http://localhost:8788/v1/models` | 200，24 KB，模型 owned_by=workbuddy（22 个模型） |
| `http://localhost:8788/v1/tools/search` | **404** |
| `http://localhost:8788/health` | 200，`{"ok":true,"realm":"cn","crypto_ok":true,...}` |

错误串里的官方提示（值得记住的措辞）：搜索端点**由所用 DeepSeek 模型 provider 的 base URL 推导**；若不对，引导用户去 Models 页检查该 provider 的 base URL，或改 `web-deepseek-api.searchBaseURL` / `searchApiKeyEnv`。**只有用户能选择或更改端点。**

## 网关侧调查（为什么这条路走不通）

- 目录 `C:\Users\63588\Desktop\workbuddy2api-panel-go\`：只有 `wb2api.exe`、`wb2api.official.exe`、`config.json`、`README.md`、`auths\`、`data\` —— **0 个 .go 源文件**，无源码可改。
- 另一个网关目录：`C:\Users\63588\Desktop\WorkBuddy-Portable\`（Python 版 `wb-proxy\`，见其他文档）。
- README 明确：把**腾讯 CodeBuddy（`copilot.tencent.com`）账号**包装为 OpenAI 兼容反向代理；入站只支持 `chat_completions`（`POST /v1/chat/completions`）、`codex_re…`、`/v1/images/generations`、`/v1/images/edits`、`/v1/videos`、`/v1/videos/{task_id}`、`/v1/videos/{task_id}/content`。
- 因此即便在网关里补一个 `/v1/tools/*` 路由，**也没有 DeepSeek 上游 endpoint 和凭据可转发**；除非自己塞一个真的 DeepSeek 官方 key，那就等于自费 + 多一层跳转，不如直连。

## 源码与 bundle 位置（复查时直接跳）

- 插件源码：`@deepseek-ai/dsh-web-deepseek-api` 包内 `src/transport.ts`、`src/index.ts`（asar 内路径 `.../app.asar/dsh/node_modules/@deepseek-ai/dsh-web-deepseek-api/src/`；本次调查在 `C:\Users\63588\public\.tmp\asar\dsh__node_modules__@deepseek-ai__dsh-web-deepseek-api__src__transport.ts` 与 `..._src__index.ts` 有提取副本）。注意该包**没有 `src/types.ts`**。
- bundle 注册点 `@deepseek-ai/dsh-base/cordis.patch.yml`：`- id: web-search-deepseek` / `name: '@deepseek-ai/dsh-web-search-deepseek'`（约 493–494 行）；`- id: web-deepseek-api` / `name: '@deepseek-ai/dsh-web-deepseek-api'`（约 498–499 行）；`- id: deepseek-account`（120–121 行）；`- id: llm-deepseek`（`dsh-llm-deepseek-api-key`，549–550 行）与 `- id: llm-deepseek-account`（552–553 行）。
- 账号服务 `@deepseek-ai/dsh-deepseek-account`：凭据键 `credentialKey('deepseek-account-platform','default')`（`src/index.ts:34`）与 `'device'`（:35）。HTTP 401 或顶层码 `40003` 时清除本地凭据并发 `deepseek-account/session-expired`；其他 HTTP 错误**保留**凭据。
- 官方 Search API **按次计费**，插件描述原话 "Incurs charges"，无免费额度。

## 当时给用户的三条路（尚未选定）

| 方案 | 成本 | 效果 | 动作 |
|---|---|---|---|
| A. 账号充值 + `searchBilling: account` | 充几元钱 | 真·官方搜索（账号 token） | 设置页 → Plugins → Built-in plugin settings → Web search and fetch → 计费方式选「账号」；或 patch 里加 `searchBilling: account` |
| B. 继续用 free-search 免费池 | 0 | 非官方，但实测 bing/exa/ddg 可用 | 把 `searchProvider` 改成 `ddg` |
| C. 买官方 API key 填 `DEEPSEEK_API_KEY` | 自费 | 真·官方搜索 | 换掉失效的那个 key |

建议顺序：**先 B 恢复可用，再视余额考虑 A**；DSH 同一时刻只能挂一个 provider，切换需改 `cordis.patch.yml`（改前备份）。当时助手已提出可代改配置并实测，**用户尚未答复选哪条**，也未执行任何配置修改。

## 复用提示

- 别再重复"拿无效 key 直接试"这一步：`ea85` 那个 key 已知失效，重试只会再拿 401。
- 判断账号 token 是否可用，看状态码：401 = token/凭据问题；**402 = token 有效但欠费**，需要充值而不是换 token。
