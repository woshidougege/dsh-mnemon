---
id: "936f3b21-d21d-435b-acf3-41fc411f9eaf"
title: "Service Portfolio 前后端全量缺口审计：HBB 变更请求 P0 缺陷、By Offering 数据范围、配置页 localStorage（ssg-sales-ops）"
description: "2026-11-17 对照 origin/develop 前端 45 文件与后端 18 条 HBB CR 接口：4 个 P0（草稿 400、Impact 被清空、附件无权限、只读接口无鉴权）、active-pn 硬编码 HBB 白名单漏 16,724 PN、base64 编码不兼容、4 配置页全 localStorage、空表/缺表/无数据源三类区分与 12 张新表设计、tst 实测基数、3 个待拍板项。"
status: "active"
created_at: "2026-11-17T01:58:49.335Z"
updated_at: "2026-11-17T01:58:49.335Z"
content_hash: "5ff6ccea79ca1b5b43bd5d7923656f9f6a836aef595e28154fed3aa7dd1a701a"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/dao/GpsSusActivePnMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/pojo/query/GpsSusActivePnQuery.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java"
  - "outputs/service-portfolio-backend-gap-20261117.md"
session_ids:
  - "mnemon-run-v1-94f812c3-364f-445b-88a6-4106b2d38ab5"
memory_body_ids:
  []
---

# Service Portfolio 前后端全量缺口审计（ssg-sales-ops，2026-11-17）

对 `origin/develop` 上前端 Portfolio 模块（45 个文件、近 400KB，最后一次提交 11-16：EOL 审查面板 + roadmap 财季配置）与后端 18 条 HBB 变更请求接口做逐项对照，并实测 tst 库数据。结论：HBB CR 接口外形全部对齐，但存在 4 个直接阻塞联调的 P0；Lifecycle By Offering 必然偏小；4 个配置页完全没有后端。完整报告落盘于 `outputs/service-portfolio-backend-gap-20261117.md`（9.4K）。

## 1. HBB 变更请求：18/18 URL+method 对齐，4 个 P0

前端 18 个 HTTP 调用与后端 18 个路由 URL、method 一一对应；状态/优先级/变更类型/BU/HBB L1 五张枚举表一致。缺陷如下：

| 严重度 | 问题 | 证据 |
|---|---|---|
| P0 | 草稿保存必然 400 | 前端 Save as a Draft 只校验归属，后端 `create` 无条件要求 `changePoints`、`justification` 非空，且库列是 NOT NULL（`HbbRequestService.java:470-475`） |
| P0 | 每次编辑清空 Impact Assessment | `update` 无条件 `setImpactAssessment(...)`，前端 payload 从不传该字段（`HbbRequestService.java:225`） |
| P0 | 附件删除无权限校验 | 连请求归属都不查，任意登录用户可删他人附件（`HbbAttachmentService.java:110-121`） |
| P0 | 4 个只读接口不校验登录 | 详情/评论/审批/附件都不过 `itcode()`（`HbbRequestController.java:83/122/139/156`） |

前端已画、后端未实现：删除请求（前端提示文案直接写 "backend has no delete endpoint"）、评论编辑、批量写接口（现为逐条 PUT）、项目时间线（只能显示 Not set）、一键 PPT。

## 2. Lifecycle By Offering 数字必然偏小 + base64 不兼容

前端复用 GPSSUS 页的 `/gpssus/active-pn`，其 SQL 写死：

```sql
AND trim(lifecycle.oh_l1_hbb) IN ('Global Product Services', 'Sustainability Services')
```

tst 实测：GPS+SUS 合计 315,390 个 PN，而 `gps_pn_lifecycle_overview` 中 PCSD+ISG 合计 332,114 → **漏 16,724**（DWS 11,292 / Vertical 2,987 / HCS 1,464 / AI 189 / Vehicle 25 / HBB 为空 767）。用户勾选 DWS 或 AI 的 offering 时后端 `decodeActivePnPath` 判定越界，整页返回空（与既有文档 `121d6b28-8c37-4caf-b9f6-8db4047c94a8` 讨论的"复用 active-pn"路径直接相关——复用可行，但必须先放开 hbbs 范围）。

第二处编码不兼容：前端 `ohTree.ts` 用 `btoa()` 生成**标准 base64**，后端用 `Base64.getUrlDecoder()`（URL-safe、无 padding）解，路径含 `+` 或 `/` 即解不出，同样整页空。

## 3. 四个配置页全部 localStorage，零后端写接口

KPI 卡、项目卡、Overview Modules、EOL Review 全部只写 localStorage，且都**配了不生效**：

- KPI 卡：前台读 `/kpi-cards`，配置写的是 `portfolio.kpi-card-config.v1`
- 项目卡：前台只读单卡 legacy 键
- Overview Modules：`Management.vue` 用硬编码 `changeManagementConfig.ts`
- EOL：`EolReviewPanels.vue` 内是写死数组，连 "SSG Offering Hierarchy V3.2 Announcement" 标题与 Details 按钮都是硬编码，按钮为 disabled

## 4. 三类"没有数据"性质不同（决定要不要开发）

**(a) 表在、数据空 —— 业务填即可，不用开发**：`sp_sp_kpi_card` 0 行（8 张 KPI 卡）、`sp_sp_roadmap_item` 0 行（不填永远 4 个空季度）、`sp_sp_page_link` 0 行（CATALOG / PORTFOLIO 外链）。

**(b) 表不存在 —— 需新建 12 张**：KPI 卡不新建（`sp_sp_kpi_card` 补 4 列）；项目卡 `sp_sp_project_card` + `_milestone` + `_risk`；Overview `sp_sp_overview_module` + `_link` / `_version` / `_phase` / `_month` / `_tag`；EOL `sp_sp_eol_review_panel` + `_scope` + `_step`。

**(c) 数据源根本不存在 —— 加表也解决不了，需业务给口径**：Fast Pass Rate（写死 94）、Cross-Team Engagement（写死 9）、SLA（写死 1.5WD）三个 KPI 无 SQL，是空态兜底常量；审批 4 阶段流转、审批证据 OCR、EOL 判定规则（现为一段自然语言，代码从不解析）同样缺源。

## 5. "前端之前反馈的问题"溯源

GitLab 上 5 个 portfolio MR（!971 / !1014 / !1046 / !1053 merged，!970 closed 因与 develop 冲突改从 `feature/portfolio-mr-develop` 重提即 !971）讨论区只有合并记录、无反馈留痕，故改为从前端代码里挖显式标注：

1. `portfolio.ts:11`「后端当前未下发 permissionKey」——实际后端**已下发**，前端未使用
2. `hbbChangeRequest.ts:65`「字段名按 tst 环境实测返回声明」——当时字段名是猜的，现存三套兜底
3. Concord 页画了 8 个筛选器但标为 `unsupportedFilters`：HBB L2、HBB L3、PN OH Enablement status、Project/CR ID、Project/CR Name、Project Manager、Product Manager、SGR Stage
4. `/oh` 限 50 条，导致 OH 树只能懒加载（对应既有文档 `c7140f7c-24ae-4a3e-89ec-282af1331ae8` 的根因分析）
5. `lifecycle/utilization` 不返回刷新时间，前端退回本地时间

## 6. 数据实测（tst / postgres）

- `gps_pn_lifecycle_overview`：total 332,114，`in_scope='Y'` 332,114（全量）；BU = ISG 266,131 / PCSD 58,865 / ISU 3,552 / MBG 3,461 / VC 68 / Others 37；HBB = Global Product Services 309,429 / Digital Workplace Solutions 11,292 / Sustainability Services 5,961 / Vertical Solutions 2,987 / Hybrid Cloud Services 1,464 / (null) 767 / AI Solutions 189 / Vehicle Computing Service 25；字段非空率：oh_code 332,059、region 332,114、creation_fy 330,695、withdrawal_fy 54,189
- `gps_spark_concord_projectcr_detail`：113 行（PROJECT 41 / CR 72），concept_plan_start_date 全非空
- `gps_pnpl_master_data`：business_group = PCSD 82 / ISG 21 / MBG 13
- `GpsSusActivePnMapper.java` 的 booking 列以正则 `^[+-]?[0-9]+(\.[0-9]+)?$` 过滤后 cast numeric，非法值按 0

## 7. 待业务/用户拍板

1. Fast Pass / Cross Team / SLA 三个 KPI 是否有数据源——没有就必须由业务给口径，否则 KPI 卡永远显示 "—"
2. 审批 4 阶段是否落库（现仅一个 status 字段，流程条纯前端绘制）
3. By Offering 是否放开 `active-pn` 的 HBB 白名单（改成 hbbs[] 入参）

## 环境备注

- 本机 `javac/java` 实际指向 Oracle javapath（`C:\\Program Files\\Common Files\\Oracle\\Java\\javapath`），JDK 目录有 `jdk-17.0.19` 与 `latest`；checkpoint 中调用 `C:\\Program Files\\Java\\jdk-21\\bin\\javac.exe` 报"不会被识别"即因此
- 读取 `target/classes/**/*.class` 会以 binary file 报错，需改读源码
- GitLab MR 查询走 `git credential fill` 取 `gitlab.xpaas.lenovo.com` token（48 字符）
