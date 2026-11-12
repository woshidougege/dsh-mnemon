---
id: "5b52aa68-ce8b-4e5c-9d25-575abd544793"
title: "Service Portfolio（admin）身份接口清理与 KPI 卡片文案来源排查（ssg-sales-ops）"
description: "删除 /service-portfolio/me 改用 GET /api/auth/user/info（MR !514）；KPI 卡片配置表 sp_sp_kpi_card/roadmap/page_link 在 TST 均为 0 行，中文文案来自 defaultsOfKpi() 的 Java 字面量兜底；docs/ 被 .gitignore 忽略。"
status: "active"
created_at: "2026-11-12T11:51:08.108Z"
updated_at: "2026-11-12T11:51:08.108Z"
content_hash: "b66e824e08441672ba740cd453063a628c1b6058367caa128b7e63d4b48d3e99"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java"
  - "docs/service-portfolio-api.md"
session_ids:
  - "mnemon-run-v1-3dceeade-dc4a-4b1d-870e-cdbca577f4fc"
memory_body_ids:
  []
---

# Service Portfolio（admin）身份接口清理与 KPI 卡片文案来源排查（ssg-sales-ops）

仓库：`dataagent/ssg-sales-ops`（https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops），模块 `smartop-service/admin`，包 `com.lenovo.admin.serviceportfolio`。
本轮工作树：`.agents/worktrees/service-portfolio-admin`，分支 `fix/sp-remove-me-admin`（基于 `origin/admin-release` 的 `31d819478`）。

## 1. 删掉 Service Portfolio 自带的 `GET /service-portfolio/me`（MR !514）

背景：TST 部署后用户质疑"已有 `user info` 接口，为什么还要一个 `/me`"。

结论与做法：
- 身份统一用既有的 **`GET /api/auth/user/info`** → `{itcode, name, displayName, isAdmin, menuKeys, ...}`，读 Redis 会话快照。只带 `user` 请求头而无会话时 `name`/`isAdmin` 为 `null` 属正常。
- 原 `/service-portfolio/me`（返回 `{itcode, name, admin}`，走 `DataRefreshController.resolveItcode(request)` + `HbbRequestService.displayName/isAdmin` 直查库、不走会话）被标记为**自测期遗留的临时接口**，从前端文档中标注"勿用、待清理"，随后在代码中删除。

代码变更（`smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java`）：
- 删除 `@GetMapping("/me")` 方法（13 行）。
- 连带清理两个只为它存在的依赖：`private HbbRequestService requestService` 字段（所有 HBB 接口都在 `HbbRequestController`，此字段无其他调用点）、import `DataRefreshController` 与 `jakarta.servlet.http.HttpServletRequest`。
- 结果：控制器只剩 `readService` 一个依赖，13 个只读接口签名/返回结构不变，`/hbb/*` 写接口未动。

验证与交付：
- `mvn -pl smartop-service/admin -am compile -DskipTests` → BUILD SUCCESS（16 模块，2:36）。注意：全量编译是后台 job + 长超时，前台 120s 会 timeout。
- MR **!514** `fix/sp-remove-me-admin → admin-release`，commit `722f0751d2685c8e021d30d10cb14dbe83213dde`，1 个文件 +36，`state=opened`、`merge_status=can_be_merged`、`has_conflicts=false`。**未合并，等用户确认。**

## 2. 前端接口文档

路径：`docs/service-portfolio-api.md`（196 → 206 行）。改了两处：
1. 第 14 行「身份」行：当前登录人信息统一用 `GET /api/auth/user/info`，Service Portfolio 不再自带身份接口；`user: {...}` 请求头仅网关/本地调试用，生产由 SSO 注入。
2. 第 1.1 节标题由 `GET /me` 改为「当前用户（用既有接口，不要用 /service-portfolio/me）」，写明会话依赖、`isAdmin` 只控制按钮显示（安全边界在后端 `HbbRequestService` 校验 `ssg_is_active_admin`）、`/service-portfolio/me` 已移除。

**坑：`docs/` 被 `.gitignore:49` 忽略**，因此这份文档只在本地工作区，不在 MR 里；要纳入版本管理需挪到可跟踪位置（如 `smartop-service/admin/docs/`）。

## 3. 排查：接口返回里的中文到底来自代码还是数据库

用户看到 `/kpi-cards` 返回 `"subtitle": "Released 状态去重 PN"`、`"点击跳到 Concord Projects"` 等中文，追问来源。

读链路（`smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java`）：
- `:55-58` `kpiCards()`：先 `configMapper.selectKpiCards()` 读 `sp_sp_kpi_card`；**表为空即 `return defaultsOfKpi()`**。
- `:92-103` `defaultsOfKpi()` 的 8 行 `kpi(...)` 全是 Java 字面量（title/subtitle/unit/link），注释写明"配置表还没数据时的兜底，保证页面不空转"。
- `:77-89` `resolveMetric`：仅 `value_type=COMPUTED` 时实时算（`OH_CODES`/`ACTIVE_PN`/`PN_TERMINATED`/`ODT_PROJECTS`/`REQUESTS` 五个 SQL），其余返回 `manual_value`；`FAST_PASS`/`CROSS_TEAM`/`SLA` 值为 `null`（暂无数据源）。

TST 实测（DEV/TST 同库）：
```
select count(*) from public.sp_sp_kpi_card     -> 0
select count(*) from public.sp_sp_roadmap_item -> 0
select count(*) from public.sp_sp_page_link    -> 0
```
迁移 `V1.1.113` 只 `CREATE TABLE`、不 `INSERT`（遵循"迁移只建表、不塞数据"的既有规矩），所以**当前整页 KPI 卡标题/副标题/路线图/外链全部来自代码兜底文案**，不是库里的数据。

表结构保留 `title VARCHAR(120)` / `subtitle VARCHAR(300)` / `permission_key` / `value_type` / `link_type` 等列，目的是让业务日后在 Admin 里维护文案而不改代码；一旦 `sp_sp_kpi_card` 插了行，接口就会返回库里的值（可能又是中文）。表注释：`'Big Events 顶部 KPI 卡配置：链接可维护，不能写死'`。

业务数据本身是英文/编码：HBB L1（`Global Product Services`）、BU（`PCSD`）、PN 编码、日期、项目名；中文只出现在**展示文案字段**与错误 `msg`（如"请求不存在"，全项目统一中文）。

### 让接口"不再有中文"的三种选项（本轮未实施，等用户拍板）
1. 只把 `defaultsOfKpi()` 8 行改英文——风险最小，但库里有配置后仍可能出中文。
2. 后端不再下发 `subtitle`，文案交给前端写死——需同步改前端 `app.js` 的 `kpiHtml` 8 个副标题 + 文档。
3. 保持现状，把中文当业务可配置文案。
附带观察：`"（暂无数据源）"` 与前端已渲染的"— 待接入"标签语义重复，改英文时可顺手去掉。

## 4. 关于"代码里中文注释"的仓库惯例

`ServicePortfolioController.java` 内 10 行含中文，全部是类 Javadoc 与 `@Operation(summary=...)`，无中文标识符/日志。blame 指向 `4e17f10beb`（renxp4，2026-11-09）。admin 模块 62 个 Controller 中 **28 个** 使用中文 `@Operation(summary)`（如 `AdminMenuConfigController`、`BookingsController`、`CostPriceController`）——**仓库没有强制英文的约定**，中文注释是主流写法。若要统一改英文，需先界定范围（仅 serviceportfolio 包 / 含前端展示文案 / 含迁移脚本注释，后者只能新增 `COMMENT ON` 迁移）。

## 5. 未闭合事项
1. MR !514 未合并（用户自行操作）；合并后建议 TST 复测：`/service-portfolio/me` 应 404、其余 13 个接口不受影响。
2. 接口中文文案的处置方案（三选一）待用户决定，尚未建 MR。
3. `docs/service-portfolio-api.md` 仍在 `.gitignore` 覆盖范围内，是否迁入仓库待确认。
