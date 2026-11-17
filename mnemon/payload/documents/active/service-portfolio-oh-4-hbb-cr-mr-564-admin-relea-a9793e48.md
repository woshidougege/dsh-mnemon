---
id: "a9793e48-cf89-4069-9871-075abf46c72f"
title: "Service Portfolio 后端实现交付：OH 树 + 4 个配置页后端化 + HBB CR 修复（MR !564, admin-release）"
description: "2026-11-17 落地记录：分支 feature/sp-api-fix-admin 单提交 f324e907e、MR !564（45 文件 +2890/-127，mergeable，未合并）。内容含 HBB CR 4 个 P0 修法与补端点、GET /oh/tree 与 /oh/codes 层级+分页设计、(Unmapped) 293 码、ohPaths 筛选、V1.1.128 建 12 张配置表与 /service-portfolio/config/* 接口、不改 gpssus 而自建 decodeOhBase64 的决策，以及前端待跟改 7 处。"
status: "active"
created_at: "2026-11-17T02:58:18.277Z"
updated_at: "2026-11-17T02:58:18.277Z"
content_hash: "408ff89b0b73ad2436a60b6103759625aeaca0b0bb51ad0fc9caf3bddf96b12f"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioConfigController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioConfigService.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.128__Extend_service_portfolio_config.sql"
session_ids:
  - "mnemon-run-v1-867a23c8-ce2b-48ff-a54c-187fc32e1b8b"
memory_body_ids:
  []
---

# Service Portfolio 后端实现交付：OH 树 + 4 个配置页后端化 + HBB CR 修复

2026-11-17 在 ssg-sales-ops 后端 admin 服务落地的实现交接。需求侧缺口与证据见既有文档 `936f3b21-d21d-435b-acf3-41fc411f9eaf`（HBB 变更请求 P0、By Offering 数据范围、配置页 localStorage）——本文只记录"已怎么实现/怎么交付"，不重复审计结论。

## 交付物

| 项 | 值 |
|---|---|
| 分支 | `feature/sp-api-fix-admin`，基线 `origin/admin-release` = `9327b8f1c` |
| 提交 | `f324e907e feat(service-portfolio): 接口修复 + OH 树 + 四个配置页后端化`（单提交，amend 后） |
| MR | !564 → `admin-release`，`opened` / `mergeable` / `has_conflicts=False` |
| 规模 | 45 个文件，+2890 / -127，全部在 `com.lenovo.admin.serviceportfolio` + 1 个迁移 |
| 链接 | https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/564 |

开发在 worktree `C:\Users\63588\lenovo\ssg-sales-ops\.agents\worktrees\sp-api-fix` 完成，主工作区 `C:\Users\63588\lenovo\ssg-sales-ops` 保持 `admin-release` 且 0 改动。MR 未合并（合并需单独授权）。

编译验证（离线、跳过测试）：`mvn -o -T 4 -pl smartop-service/admin -am compile -DskipTests -q` → exit 0（约 320s，需后台任务跑，同步 run_code 120s 会超时）。

## 1. HBB 变更请求缺陷修复

- 草稿可存：`changePoints` / `justification` 改为草稿非必填、提交时才校验；迁移同步放开对应 NOT NULL 列。
- 编辑不再清空 Impact：`update` 由整字段覆盖改为 merge 语义，前端不传的 `impactAssessment` 保持原值。
- 鉴权补齐：附件删除先校验请求归属；详情/评论/审批/附件 4 个只读接口补 `itcode()` 登录校验。
- 补齐缺失端点：`DELETE /requests/{id}`、`PUT /requests/{id}/comments/{commentId}`、`PUT /requests/batch`（替代逐条 PUT）、`GET /requests/{id}/audit`。
- 新增 `stage` 审批四阶段、项目四步流程日期；`requestNo` 并发撞键改为重试；码值三套（状态/优先级/变更类型等）统一并做旧值归一。

## 2. OH 树（替代扁平 + LIMIT 50）

- `GET /oh/tree`：L1(HBB) → L2 → L3 一次返回全量层级，节点带 OH 码数 / PN 数 / value。
- `GET /oh/codes`：叶子码真分页，替代原来的截断列表。
- 293 个未挂 HBB 的码归入 `(Unmapped)` 节点，不再被过滤掉。
- Lifecycle 三个接口都支持 `ohPaths` 筛选（传节点的 value 即可），不再依赖 `/gpssus/active-pn` 的 GPS+SUS 白名单（该白名单会漏 16,724 PN）。

编码不兼容的收口方式：前端 `btoa()` 产出标准 base64（含 `+ / =`），后端原用 `getUrlDecoder` 解导致整页空。现在两侧都做宽松解码；后端实现为 `ServicePortfolioReadService.decodeOhBase64()`，标准与 URL-safe 两种都能解。

## 3. 四个配置页后端化

- `/service-portfolio/config/{kpi-cards, project-cards, overview-modules, eol-panels}` 整份读写，仅 admin 可写；取代原先只存 localStorage、换机器即丢的做法。
- 迁移 `V1.1.128__Extend_service_portfolio_config.sql` 新建 12 张表，只建结构、不塞业务数据（业务填即可）。
- 新增实体/Mapper：`SpKpiCard`、`SpProjectCard` + `_Milestone` + `_Risk`、`SpOverviewModule` + `_Link`/`_Version`/`_Phase`/`_Month`/`_Tag`、`SpEolReviewPanel` + `_Scope` + `_Step`；响应行 `OhCodeRow`、`OhTreeRow`；新增 `ServicePortfolioConfigController` / `ServicePortfolioConfigService`。

## 关键决策与约束

- **不改别人模块的代码**（用户 2026-11-17 明确要求："最好不要动人家的代码。我们自己的业务模块里可以复制他们的逻辑 不要改人家的哈"）。因此 `GpsSusService.java` 的改动整体回滚到 `origin/admin-release` 基线（`git diff` 为空），兼容解码逻辑复制到自己的 `ServicePortfolioReadService.decodeOhBase64()`，不引用对方代码。
- **一个任务只留一个 MR**：amend + force push 更新的是同一个 !564（`changes` 46→45，sha `a9793a2ae`→`f324e907e`），GitLab 不会因 force push 新建 MR；无需关单。
- 完整实现清单（接口签名、编码规则、自测步骤）落在本地 `outputs/service-portfolio-api-fix-20261117.md`；该目录已被 `.gitignore` 的 `/outputs/` 忽略，不入库。

## 前端待跟改 7 处

1. 四个配置页改调 `/service-portfolio/config/*` 接口，弃用 localStorage。
2. Lifecycle By Offering 改调 `/service-portfolio/lifecycle/overview`，不要复用 `/gpssus/active-pn`。
3. OH 筛选器改用 `/oh/tree` + `/oh/codes`。
4. 草稿保存去掉前端兜底（后端已放开非必填）。
5. 码值改用统一后的新值。
6. 错误提示按后端英文文案重写。
7. 读 `stage` 字段展示四步审批流程。
