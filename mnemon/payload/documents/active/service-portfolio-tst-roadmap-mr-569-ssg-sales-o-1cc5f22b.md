---
id: "1cc5f22b-e6ea-4ae2-80ed-7009516b6317"
title: "Service Portfolio 展示页 TST 部署验收（roadmap 接项目卡，MR !569，ssg-sales-ops）"
description: "2026-11-17 TST 验收：前端路由前缀是 /spark/portfolio（/portfolio、/service-portfolio 均 404）；匿名访问接口 401 需借已登录会话 cookie Authorization；/roadmap 四季度内容与 banner 均取项目卡，project-cards/eol-panels/overview-modules/lifecycle 全部 200；遗留 KPI 只渲染 1/8 与项目卡 Sample data 硬编码标签。"
status: "active"
created_at: "2026-11-17T13:33:25.648Z"
updated_at: "2026-11-17T13:33:25.648Z"
content_hash: "9a69a3230182b6e52d635b47aba12f4a9ae6b3e5c025800f64015301bcafc559"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java"
session_ids:
  - "mnemon-run-v1-ec362648-cea1-4a6c-90ca-1362f4e9baa0"
memory_body_ids:
  []
---

# Service Portfolio 展示页 TST 部署验收（roadmap 已接项目卡，MR !569，ssg-sales-ops）

2026-11-17 验收记录。前置背景见：
- `e7cdd3c7-ce26-4dc9-a8a3-8dcaee8b4c65` —— MR !569 三个只读接口 + 前端逐接口核对表
- `33820175-1b4b-4478-8db3-98ad05e5339c` —— roadmap 以项目卡为排期主数据的决策与实现
- `5b52aa68-ce8b-4e5c-9d25-575abd544793` —— KPI 卡片文案来源（defaultsOfKpi 字面量兜底）

## 1. 结论

用户部署到 TST 后要求确认"有没有数据"。**后端与前端都已生效**：`/roadmap` 返回的季度内容全部来自 `sp_sp_project_card`，页面时间轴与紫色横幅也渲染出项目卡数据，与库中 3 条 demo 项目一致。

## 2. 验证踩坑：TST 前端路由前缀是 `/spark`

- `https://ssgspark-tst.lenovo.com/portfolio` → **404「页面不存在」**
- `https://ssgspark-tst.lenovo.com/service-portfolio` → **404「页面不存在」**
- `https://ssgspark-tst.lenovo.com/spark/portfolio` → **正常渲染 Service Portfolio 页**

不要用 `src/router/index.ts` 里的相对路径（`portfolio`、`portfolio/config`）直接拼域名；部署在 `/spark` 基路径下。左侧导航项为「Service Portfolio」，路由页含 Big Events / Portfolio Management / Concord Projects / Lifecycle 四个区域按钮。

## 3. 验证踩坑：TST 接口匿名访问 401

- 直接 curl 无凭证：`{"http":401,"msg":"token不存在或者为空"}`。
- 可行做法：在用户已登录的浏览器会话里取 cookie `Authorization`（`lsKeys` 中只有 `pbi_auth_token`，token 不在 localStorage），作为请求头打接口，返回 `status:200`。即网关 SSO 注入 token，手工联调必须借已登录会话。

## 4. `/roadmap` 实测返回（FY26/27，currentQuarter=Q3）

| 季度 | 内容 |
|---|---|
| Q1 Apr–Jun | 空 |
| Q2 Jul–Sep | MGB Service PN Cleanup（Completed） |
| **Q3 Oct–Dec（current）** | ISG Service PN Lifecycle Review（In progress） |
| Q4 Jan–Mar | PCSD Offering Hierarchy V3.2 Transition（Planned） |

`banner` = Q3 的 `demo-isg-lifecycle-review`，与原型中紫色横幅一致。页面上时间轴文案实测为 `Q1 Apr–Jun / Q2 Jul–Sep MGB Service PN Cleanup NOW / Q3 Oct–Dec ISG Service PN Lifecycle Review / Q4 Jan–Mar PCSD Offering Hierarchy V3.2 Transition`，右上角为 `FY26/27 Q3 (Oct-Dec) · ISG Service PN Lifecycle Review active`。

## 5. 其余接口 TST 实测（均 http 200 / biz 200）

- `GET /service-portfolio/project-cards` → 3 条，含 `roadmapQuarter`、`progress:65`、`owner`、`startDate/endDate`
- `GET /service-portfolio/eol-panels` → 2 条，字段名是 `scope`（前端硬编码版本里叫 `criteria`，需改名）
- `GET /service-portfolio/overview-modules` → 8 条，含 `layout:banner`、`group:top-banner`、`order`、`enabled`
- `GET /lifecycle/utilization?bu=ISG` / `?bu=PCSD` → 各 6 个财年（ISG FY2122 activeCount 144,539；PCSD FY2122 24,684），BU 参数可用
- `GET /lifecycle/overview` → totals pnCount 332,138 / released 277,940 / obsolete 54,195 / booking 11,014,815,261 + byOffering

据此后端部分可视为验收通过；前端仍未接的是 By Offering / By Geo / 合计（应接 `/lifecycle/overview`）。

## 6. 验收时发现的两个遗留问题（前端侧）

1. **顶部 8 张 KPI 只渲染出 1 张**（仅 `1617 K Offering Hierarchy Codes`）。`/kpi-cards` 后端返回 8 条（OH_CODES、ACTIVE_PN、ODT_PROJECTS、REQUESTS、FAST_PASS、CROSS_TEAM、SLA、PN_TERMINATED），因此要么配置表 `visible` 只开了 1 条，要么前端只渲染第一张——未定论，待查。
2. **项目卡右上角 `Sample data` 蓝标是前端硬编码文案**，不是后端字段；联调完成/上线前应去掉。

## 7. 截图

`.tmp/sp-verify/tst-portfolio-big-events.png`（1667×835）—— TST Big Events 页，含时间轴与横幅。注意：`bsk screenshot` 返回的临时 png 在会话 stop 后可能被清理，复制到工作区要尽早。
