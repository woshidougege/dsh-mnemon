---
id: "65bfad27-16bf-4c14-a718-f9c529e63d8f"
title: "腾讯网关搜索 vs DSH 免费搜索插件：6 组中英查询质量对照与选型结论"
description: "同一时点三条搜索链路状态（核心 web_search 仍 401 / 免费插件引擎池通过 / PANEL 腾讯 5 条 1.2-1.5s），6 组中英文 query 的胜负矩阵（腾讯强中文长尾与站内帖子，exa 强英文技术文档与 CHANGELOG），腾讯的空 URL、栏目页、时效偏差、脏链接缺陷谱，插件中文站内搜索弱的缺陷谱，以及 A/B/C 修复方案与 deepseekToolsBaseUrl 待验证项。"
status: "active"
created_at: "2026-11-20T03:45:37.748Z"
updated_at: "2026-11-20T03:45:37.748Z"
content_hash: "6e60008960342f4497351d235f9e3309aca7f1822a633366b6cb1256d0e0c7da"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-d2e8190c-b223-4715-a10a-3001c477a521"
memory_body_ids:
  []
---

# 腾讯网关搜索 vs DSH 免费搜索插件：质量对照（本机 2026-11-20 实测）

触发：用户问「这个免费插件好用，还是 8788 的腾讯搜索好用呢？」。本文是选型基准，下次再问「搜索该走哪条」可直接复用，不必重跑对照。

关联文档：
- `a33b2a00-83e9-4029-b9c7-ac52472c9708`（网关各分支是否有 /v1/search、/v1/fetch，PANEL 的 agenttool 透传语义与 DSH 对接契约）
- `02486bfd-f172-41a6-afeb-9d2fc6fd39e1`（DSH 官方搜索靠账号 token + searchBilling，8788 上游是腾讯不是 DeepSeek）

## 0. 三条链路现状（同一时点）

| 链路 | 走什么 | 状态 |
|---|---|---|
| DSH 核心 `web_search` | `web.searchProvider: deepseek-search-api` → `https://api.deepseek.com/v1/tools/search` | 仍 401（api key 失效）；`fetchProvider` 已改为 `fs-quality-fetch` |
| DSH 免费搜索插件 `web-search-free`（`dsh-free-search` 0.9.73，`provider: auto`） | 免 key 引擎池 | 体检通过：bing / exa / ddg / anysearch / tavily 各 results=2；仅 `deepseek-official` 401 |
| 网关 `POST /v1/search`（PANEL，`127.0.0.1:8788`） | 腾讯 WorkBuddy `agenttool/v1/search` | 正常，5 条/次，1200–1500ms 稳定 |

网关当前状态：重启后新进程 pid=28348，49 个模型，`request_archive_enabled = true`，事件日志 15 条（缓存命中均值 94.7%，粘性路由生效）。归档只记开关生效之后的请求，历史 135 条不会补录。

## 1. 六组对照（每 query 各取前 5 条）

| 查询 | 腾讯（8788） | 插件（auto） | 赢家 |
|---|---|---|---|
| `DeepSeek V4 发布` | 命中官网 `deepseek.com/news/v4-preview/` + 模型卡 ×2 | 全是官网首页 / 登录页 / 平台页 / 镜像站 | 腾讯 |
| `2026年11月 国内经济政策 最新` | 发改委、今日头条、中国政策研究网 | 华尔街日报中文、世界银行、国家统计局、IMF、VOA | 腾讯 |
| `v2ex deepseek 使用体验` | 精准到帖子 `v2ex.com/t/1257339`、`t/1250093`、`t/1213287` | 全是 V2EX 首页 / 节点页 / 维基百科 / 第三方导航 | 腾讯 |
| `今天 A股 收盘 涨跌` | 5 条里 2 条 **url 为空**，其余是「午评」而非收盘 | 5 条精确文章，带发布日期（11-19） | 插件（exa） |
| `Go 1.26 release notes` | `go.dev/doc/go1.26` 排第 1 | 同样第 1 | 平手 |
| `kubernetes 1.35 changelog` | 官方博客 + 中/意文档 + 一个 netlify 预览站噪音 | 直接给 `CHANGELOG/CHANGELOG-1.35.md` 源文件与 release tag | 插件（exa） |

速度：腾讯 1200–1500ms 稳；插件 1180–3400ms 波动（exa 最慢到 3.4s）。

## 2. 双方缺陷谱（选型时按此权衡）

腾讯 / WorkBuddy `agenttool/v1/search`：
1. 会返回**空 url**（A股那次 2/5 条），等于废数据；
2. 偏给**栏目页而非文章页**（如 `m.nbd.com.cn/web_app/column/2272?max_pos=...`）；
3. 有时效偏差，问「收盘」给「午评」；
4. 偶有脏链接（`deepseek.com/?q=Analyze%20and%20summarize...` 这类把 prompt 塞进 URL 的条目）。

免费插件（`provider: auto`）：
1. **中文站内搜索弱**：问 V2EX 帖子给首页，问产品发布给官网首页；
2. auto 路由对中文倾向 bing / anysearch，长尾能力一般；英文技术更常落到 exa；
3. exa 延迟高时 3–4s。

## 3. 结论

- **中文为主**（国内新闻、政策、论坛/公众号讨论）→ 腾讯更好，中文索引深度插件比不上。
- **写代码查文档**（Go、K8s、SDK、release 分支）→ exa 更好，能直击仓库里的 CHANGELOG 与 tag。
- **最优是两条都留、按 query 语言分流**，而不是二选一。

## 4. 修复 DSH 搜索的三个方案（待用户选定）

- **A（最稳）**：把 `web.searchProvider` 从 `deepseek-search-api` 改成因免费插件接管 → 立刻可用，但中文质量等于上表插件列。
- **B（需验证）**：插件的 `deepseekToolsBaseUrl`（`z.string().default("")`，注释为「自定义 DeepSeek 兼容 tools 端点」，对应 issue #124）指向 `http://127.0.0.1:8788/v1`，让 DSH 搜索吃腾讯结果。未知项两项：路径拼接是否正好落成 `/v1/search`；网关是否认插件发过去的请求体。相关设置键：`deepseekToolsBaseUrl` / `deepseekToolsApiKey` / `deepseekFetchBaseUrl` / `deepseekFetchApiKey`（后者留空即不启用该层，key 留空时依次复用工具 key、搜索 key）。
- **C**：换一个有效的 DeepSeek 官方 key。

推荐：先花几分钟验 B，通了按「中文走腾讯、英文走 exa」配双路由；不通退回 A。
