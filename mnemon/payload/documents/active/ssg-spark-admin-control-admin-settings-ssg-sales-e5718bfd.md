---
id: "e5718bfd-fbfd-4213-aa85-72f8bcf99b19"
title: "SSG Spark Admin Control / Admin settings 功能全景清单（ssg-sales-ops）"
description: "后台两层功能的完整清单：Admin Control 四页签 + Admin settings 11 分区、跨模块口径（导出仅 xlsx/时间双轨）、12–13 条有入口但不可用的已知限制、核对中发现的权限校验技术债，以及单文件 HTML 交付做法。"
status: "active"
created_at: "2026-11-16T10:26:10.750Z"
updated_at: "2026-11-16T10:26:10.750Z"
content_hash: "fab51afaf109a1099caf330a0ea812b1226170df62b83e78818d11a8bd756eb4"
source_paths:
  - "outputs/admin-control-feature-list.html"
  - "outputs/admin-control-feature-list.pdf"
  - "outputs/admin-control-feature-list.md"
session_ids:
  - "mnemon-run-v1-767a8f5a-3ee5-466f-a742-ec6d3e718769"
memory_body_ids:
  []
---

# SSG Spark Admin Control / Admin settings 功能全景清单（ssg-sales-ops）

来源：2026-11-16 的一轮「从前后端代码层面整理 admin 功能清单」任务。做法是逐个后台页面读前端组件、再用后端 controller 反查接口与校验，只写「页面上真有这件事」的功能；先后产出 Markdown（596 行）、单文件 HTML（963 行 / 87,883 bytes）与 25 页 PDF。

## 1. 交付物

- `outputs/admin-control-feature-list.md`（596 行，约 50 KB，第 1 版）
- `outputs/admin-control-feature-list.html`（单文件、无外部依赖、双击可开；左导航 + 搜索 + 深色模式 + 打印；22 个 section，标签闭合校验 div 767/767 / section 22/22 / table 10/10 / tr 80/80）
- `outputs/admin-control-feature-list.pdf`（25 页，935×1210pt，Chrome headless 打印）

切 HTML 的原因（用户原话，checkpoint ev.386）：「看的不直观呀 是不是做成html的对用户比较友好呢」——长文档面向业务同事时，网页的目录/搜索/折叠比 Markdown 可读。

## 2. 两层结构（入口都在侧边栏底部，非管理员不可见）

- **Admin Control（`/setting`）**：看数据，顶部一排页签。
- **Admin settings（`/setting/settings`）**：改配置，左栏 11 个分区。
- 两层是不同路由、不同组件，混讲最容易出错。

## 3. Admin Control 的四个区块

| 页签 | 要点 |
| --- | --- |
| **User Logs**（默认） | 4 个子页签：Data Inquiry Records（4 指标卡 + 报表来源分布 + 15 列明细）、Feedback（5 卡 + 4 宫格比率 + 24 列，可一键开 Jira）、Page Analytics（5 卡 + 日趋势 + Top10 + 三级树表）、Raw Events（原始埋点 7 列）。共用一条筛选栏（全局搜索 Ctrl+K、6 档快捷时间、20 多个维度并按页签自动显隐）、会话回放弹窗、人工评审抽屉（Correct/Wrong + 归因）、Excel 导出（全局互斥锁 + 阶段进度） |
| **Issue & Request** | 用户问题单 + 被点踩回答合成一张总表：时间窗、5 指标卡（Total feedback 副行拆「提交的问题单 · 点踩回答」/ Submitted Issues / Downvoted Answers / Jira failed / Jira tickets）、**13 个多选下拉**（User 候选只列反馈中出现过的账号并带条数、Type、Band、L3/L4/L5 三条汇报线、Team、Reason、Jira status、Review status、Review tag、Reviewer、Review source）+ Include internal users 勾选（默认不勾，内部账号默认排除）、12 列列表、工单详情抽屉（Discussion 可直接发 Jira 评论、附件图片预览/其他下载、Retry Jira / Sync Jira status / Open Jira）、Jira 预览建单。评论限流：同工单 60 秒 1 条、每人每天 20 条 |
| **Inspection Report** | Smart Data Query 下三子页签：Health Check（Run now + 1.2 秒实时进度）、Baseline（基准值回归 + 判例库 + 回归指标 + 提 Jira）、LUDP Performance（直连耗时、110 秒超时率）。右上齿轮进配置：题库、基准 SQL、巡检调度、Smart DQ 30 个参数、IntelliData 连接 |
| **自定义快捷链接** | 任意 iframe / 新标签页入口；沉浸模式（隐藏侧边栏、顶部热区唤出工具条、Esc 退出） |

分页默认 10 条/页，页大小偏好会被记住。

## 4. Admin settings 的 11 个分区

| 分组 | 分区 | 管什么 |
| --- | --- | --- |
| Access & people | People | 团队/Band 人工补充（优先级高于 HC 清单）、月度 HC 清单导入（先预检后覆盖，不可回滚）、内部员工名单 |
| | Access control | 用户、用户组、角色、菜单权限；用户级三态 VISIBLE / DISABLED / HIDDEN（角色菜单已停用） |
| | Audit log | 只读审计：谁改了什么、before→after、理由 |
| Notifications | Notification settings | 站内/邮件两个总闸、4 组 10 个事件、收件人模式、自定义 Jira 状态规则（上限 50 条） |
| Jira & intake | Jira integration | 自动建单总闸 + 4 条链路开关、连接与 Validate、9 项工单默认值、模板、custom field、4 个经办人 |
| | Page focal | 页面 → Jira 经办人映射，带 `*` 兜底行 |
| | Mode groups | User Logs 里 Mode 筛选的分组名 |
| Operations | Scheduler Center | 任务 / 执行历史 / handler 目录、Run/Pause、cron 中文助手；内含 Trino→PG 数据同步与 Redis 缓存重建 |
| | Review rules | 4 条自动判错规则 + 手动补漏（按 7/30/90 天或全部） |
| Admin workspace | Admin Control tabs | 顶部页签的排序/改名/隐藏 + 新增快捷链接 |
| | Display | 个人显示的 Jira/KM 列开关（只存本地，不影响他人） |

## 5. 跨模块口径与四个易错点

1. **两个同名「Issue & Request」**：侧边栏那个是普通用户提单页；Admin Control 里那个是管理员看全量。不同组件、不同视角。
2. **导出只有 Excel**：CSV 已于 2026-11-13 下线；重型导出全平台互斥，别人在导时返回 409。
3. **时间口径双轨**：筛选/统计按北京时间切天，列表时间按浏览器本地时区显示——非北京同事会看到两行时间。
4. **Review rules 默认值前后端要一起看**：后端 `ChatbiQuestionAutoReviewService.java:43-44,58-61`（Record ReviewSetting + 四个布尔默认）中 `wrongTimeout=true`、`wrongFailed=true`、`wrongNoResult=false`、`wrongNeedsInput=false`，upsert 在 `:73-74`；前端用 `!!== false` 与 `=== true` 两种写法解析（对应行 74-77），保存时把四个开关折算成计数（行 65-68）。改动默认值必须两侧同步，否则「未配置」与「配置为关」不可区分。

## 6. 已知限制（有入口但点不到，不要写进对用户的承诺）

两级计数略有出入：Markdown 第 5 节与 HTML 导航标签都是 **12 条**，最终答复文字表述为 13 条。典型条目：

- Issue & Request 的 **Review status / Review tag / Reviewer / Review source 四个筛选后端未接**：筛选不生效，且对应三列恒为空。
- Inspection 的「手动跑」与三个设置弹窗没有入口。
- LUDP 超时阈值只读，不可改。

另外 HTML 附录把这一节做成独立 section（`id="gaps"`），便于对外时刻意跳过。

## 7. 技术债（开发备注，勿给最终用户）

核对中发现、需单列处理的内部问题：

- Trino 的两个 controller 与 Baseline 的三个只读接口**缺管理员校验**（越权读数据的风险面）。
- 若干接口无调用方（前端已不再走）。
- 残留文案（改版遗留措辞）。

这批内容在 HTML 里是折叠的「不要给最终用户」区块（`id="dev"`），需要时可拆成独立技术债清单。

## 8. 单文件 HTML 的复用要点（下次做同类交付）

- 页面单元统一为 `.row / tbody tr / .card / .callout / .pill`，搜索脚本按这些单元过滤，并额外隐藏「tbody 无可见行」的表格与空标题，避免留下孤零零的标题。
- 导航用 `scroll spy`（IntersectionObserver）联动高亮；`nav a.sub` 承接三级锚点（本例 7 个 `ul-*` 子锚点在 section 内）。
- 深色用 `[data-theme="dark"]` 变量组 + localStorage 记忆；打印用 `@media print` 隐藏 `.side/.top`、`section{break-inside:avoid}`。
- 验证链：Chrome/Edge headless 截图（本机路径 `C:\Program Files\Google\Chrome\Application\chrome.exe`、`C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe`）→ 用 PyMuPDF 读页数；打印时用固定 `@page` 与 935pt 宽度可压到 25 页，否则会出 31 页并出现 "OVERVIE W" 式断词。

## 9. 相关文档

- `c9e39da9-9d7e-4222-9082-b1cb80ef8447`：ChatBI 评审第三状态（Not applicable / N/A）方案——本清单「Review rules / 人工评审」的后续演进，含 `ChatbiQuestionAutoReviewService.java` 等完整源码路径。
- `00783d80-454e-422d-b457-52c4b3f14778`：设置页左侧导航自动折叠修复（MR !1042）——本清单「Admin settings 骨架」相关的前端缺陷记录。
