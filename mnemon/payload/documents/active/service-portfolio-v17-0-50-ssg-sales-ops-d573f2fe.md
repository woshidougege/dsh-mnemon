---
id: "d573f2fe-4bf1-4151-b287-c802c881c27b"
title: "Service Portfolio v17.0 交付盘点：原型可信边界、会议 50 条需求落地状态、待业务方输入（ssg-sales-ops）"
description: "对照 v17.0 原型、13 章会议录屏与 TST 线上页面：四模块状态、6 件必做、前端 3 条断链、原型静态 vs 脚本渲染判定与死代码、后端 44 接口与 7 张零种子配置表、50 条需求落地状态（22/12/14/2）、需业务方提供的 4 项硬输入、报告路径。缺口细节引用既有文档 936f3b21。"
status: "active"
created_at: "2026-11-17T15:37:43.301Z"
updated_at: "2026-11-17T15:37:43.301Z"
content_hash: "83eee007dc6157659f2b00d492c663ef5fb2e1ac4171768eee25eed321291be1"
source_paths:
  - "outputs/service-portfolio-gap-report.html"
session_ids:
  - "mnemon-run-v1-dab6f05e-9193-47a9-94ed-f75c9a09ada8"
memory_body_ids:
  []
---

# Service Portfolio v17.0 交付盘点：原型 / 会议 / TST 线上三方对照（ssg-sales-ops）

2026-11-17 的交付盘点：把 HBB Service Portfolio 原型 v17.0、需求评审会议录屏（13 章）与 TST 线上实跑页面三方逐功能对照，产出可交付分析报告。报告落盘 `outputs/service-portfolio-gap-report.html`（9 章 / 14 表 / 150 行明细 / 5 张 TST 实测截图，自包含 HTML，已在 Chrome 实际渲染校验：图片全部解码、无残留占位符）。

**与既有文档的分工**：接口级缺陷细节（HBB CR 的 4 个 P0、By Offering 少 16,724 PN、base64 不兼容、四配置页 localStorage）已由 `936f3b21-d21d-435b-acf3-41fc411f9eaf` 承载，本文只记录它没有的三个新维度：原型可信边界、会议 50 条需求的落地状态矩阵、需业务方提供的输入清单。

## 一句话结论

框架与主流程通，TST 能跑；卡住的不是技术，而是**业务方输入未到 + 前端有接口却没接**。

## 1. 四模块实际状态

| 模块 | 状态 | 关键缺口 |
|---|---|---|
| Big Events | 基本可用 | 8 张 KPI 只有 3 张可点（其余 `link_url` 为空）；项目卡带 "Sample data" 水印；季节模板未做 |
| Portfolio Management | 功能最完整 | 审批接口无权限校验；时间轴四节点恒 Not set；外链不能增删 |
| Concord Projects | 数据已通 | 113 行全量无分页；总数与会议所述 108 对不上 |
| Lifecycle | 图表与表都通 | By Offering、By Geo 未接 `getLifecycleOverview`；首屏 1.8~3.3s |

## 2. 必须尽快处理的六件事

1. **D1 审批接口无权限**：`POST /hbb/requests/{id}/approvals` 只校验登录态，`HbbRequestService.java:459` → 任何已登录用户可把单据改成 APPROVED。唯一的安全问题。
2. **D2 七张配置表种子为空**：PRE 与 PROD 实测 0 行，上线后 Project / Overview / EOL / Roadmap 四块全空。
3. **D5 五张 KPI 卡点不动**：PBI 链接未提供。
4. **D7 去掉 "Sample data" 水印**。
5. **D4 项目四阶段时间写不进去**：`V1.1.128` 建了列，`saveProject` 只写 name/lifecycle/sortNo/remark，四列恒 null。
6. **D10 / D11 配置页显示异常**：COMPUTED 卡显示 "—"；TAB 类型目标位置显示 `concord/concord` 这种原始值。

## 3. 前端三条"断链"（接口都给了，前端没接）

- 首页项目卡读 **localStorage**（`index.vue:90`），写库的 `saveProjectCardConfig`（`projectCardConfig.ts:128`）全仓库**零调用者** → 配置页改 Projects 对首页完全无效。
- EOL 面板**硬编码**（`EolReviewPanels.vue:16-51`），且字段名 `criteria` 与契约 `scope` 不一致。
- `/config/overview-modules` **只有 CRUD、无消费方**；Management 页仍读硬编码的 `changeManagementConfig.ts:53-104`。
- 另有 9 个悬空接口（定义但无人调用），其中 `getLifecycleOverview` 悬空即 By Offering 表无数据的直接原因。

## 4. 原型 v17.0 可信边界（静态 mock vs 脚本渲染）

判定方法：看 `<script>` 是否有选择器指向该模块 DOM。

- **纯静态 HTML**：KPI 8 卡、Strategic Pillars 左图（整张 base64 PNG，435,978 字符，不可结构化）、Portfolio 新版与 legacy、Concord 汇总 3 卡（环形图 `conic-gradient` 写死 73.1707%）、Concord 主表（23 列 × 10 行写死，JS 只做 `filterConcordRows()` 前端过滤）、Lifecycle Overview 2 表、By Offering 汇总表初值、参考幻灯片 8 张。
- **Chart.js 脚本渲染**：Lifecycle Overview 2 图（`renderHbInventoryCharts()`，内联数组 PCSD 21 点 / ISG 19 点 × Creation/Obsolete/Active 三系列）、By Offering 趋势图。
- **脚本里"无宿主"的沉默失效**：`renderHbEventsIsgChart()` 期望 `#hbEventsIsgChart`（DOM 不存在）→ 空转，实际图仍是手写 SVG；`renderHbProjectDetail()` 期望 `#hbProjectDetail` + `.hb-project-item`（均无 DOM）→ 49 条 `hbProjectDetails` 数据无法展示。

**原型自身数据矛盾（不可作为口径依据）**：ISG 同一指标 Events 表 225,208 / 223,155 与 Lifecycle 表 224,479 / 223,552 两套数；Offering 汇总 270,789 vs PCSD 46,332 + ISG 224,479 = 270,811，尾差 22。

**其他实现风险**：14 个 OH 复选框只有 7 个有数据（`filter(Boolean)` 丢值，只选无数据的会退回 All Offerings）；HBB 维度下拉值 `GPS`/`SUS` 与列值 "Global Product Services"/"Sustainability Service" 匹配不上 → 选 GPS/SUS 得空表；41 条记录只渲染 10 行（无服务端分页）；两个 FY select 均无 `selected` → 默认区间 FY06/07–FY06/07，执行过滤后表格全隐、图表仅剩 1 点；`Project`/`CR` 两个类型 tab 首屏双开且缺"全部"语义；10 个资源链接为 `href="#"`；Copilot 4 套问答全硬编码。

## 5. 后端能力盘点（静态分析）

- 共 **44 个接口**：`ServicePortfolioController` 展示只读 14 + `ServicePortfolioConfigController` 配置读写 8 + `HbbRequestController` 22。
- 取数仅两套来源：GPS&SUS 同步表 3 张（`gps_pn_lifecycle_overview` / `gps_spark_concord_projectcr_detail` / `gps_concord_project_pn_bkgs`）+ 自有 `sp_*` 24 张（`V1.1.113`、`V1.1.128`）。
- 24 张自有表**全部无业务种子**，仅 `sp_sp_kpi_card` 有 8 张卡种子（`V1.1.129__Seed_service_portfolio_kpi_cards.sql:8-43`）；`FAST_PASS`/`CROSS_TEAM`/`SLA` 为 MANUAL 死数（94 / 9 / 1.5）。
- 全部 SQL 写在 Mapper 接口的 `@Select` 注解里，`resources/mapper` 下无任何 XML。
- 权限只在 Controller/Service 内手写判断，无 Spring Security、无拦截器兜底（`CiopInterceptorConfig.java:25-27` 中 `roleapi` 未配置直接 return）。
- 5 个 COMPUTED 指标里 2 个做千分位取整（`ServicePortfolioReadService.java:369-371`）→ PN 数 < 500 时卡片显示 0。
- 源同步表未导入时是 **SQL 异常**（`relation "public.gps_pn_lifecycle_overview" does not exist`）而非空结果 → 整块 500。
- `/roadmap` 只保留当前财年+当前季度，`roadmap_quarter` 必须匹配 `FY26/27 Q1` 形式（`Q[1-4]` 正则），格式错即静默丢弃。
- `sp.service-portfolio.attachment-dir` 所有 profile 都未配置 → 回落 `${java.io.tmpdir}/ssg-service-portfolio/attachments`，重启/多副本丢附件；`/export` 无行数上限。

**明确未实现**：Smart Fill（后端零命中，前端实现必 404）；一键 PPT（无 Controller/Service）；xlsx 导出（仅 CSV，`HbbRequestService.java:510-533` 固定 12 列）；Roadmap 与 Page Link 维护（只读，靠 SQL）；审批证据自动抽取（仅 `EVIDENCE` 枚举预留）。

## 6. 会议 50 条需求落地状态

已实现 22 / 部分实现 12 / 未实现 14 / 决定项 2。未实现的 14 条里有 6 条责任不在我方（业务资料未到）。

## 7. 需业务方提供（真正的卡点）

| 优先级 | 需要什么 | 谁给 |
|---|---|---|
| 最硬 | 5 张 KPI 卡的 PBI / 资料库链接 | 业务方 + PBI 管理员 |
| 最硬 | GPS、SUS 取数逻辑文档（会上承诺"发群里"，至今未到） | 业务方 |
| 最硬 | 3–4 套展示模板的要素清单 | 业务方 → 设计 |
| 最硬 | 每张 KPI 卡的数据来源与增强口径 | 业务方 |

这四项不给，Concord / Lifecycle 的数字能否对外、Big Events 是否继续逐项目手写，都无法定案。

## 8. 建议收尾顺序

P0（D1 鉴权、D2 种子数据）→ 前端断链（项目卡、EOL / overview-modules 接契约）→ 数据口径与分页 → 业务方输入到位后再做模板化版式与 PPT / Smart Fill。

## 源文件与证据

- 报告（自包含）：`outputs/service-portfolio-gap-report.html`
- 会议需求清单：`.tmp/sp-analysis/meeting-requirements.md`（13 章逐章解读 + 50 条需求条目表 + 11 项专项议题 + 悬空问题清单）
- 原型结构解析：`.tmp/sp-analysis/proto-structure.md`（按 `[P]:行号` 定位原型单文件 HTML 各模块）
- 后端盘点：`.tmp/sp-analysis/be-inventory.md`（接口清单 / 表清单 / COMPUTED 口径 / 风险点）
- 关键代码：`smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/{controller,service,mapper}/`、`HbbRequestService.java:459,470-475,510-533`、`HbbAttachmentService.java:110-121,137-143`、`V1.1.128`、`V1.1.129__Seed_service_portfolio_kpi_cards.sql`

> 静态 mock 判定与接口清单来自静态分析；"表为空""页面报错"这类结论来自 TST/PRE 实跑，未在 PROD 复测。
