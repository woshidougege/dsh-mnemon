---
id: "e7cdd3c7-ce26-4dc9-a8a3-8dcaee8b4c65"
title: "Service Portfolio 展示页只读接口与前端对接核对（MR !569，ssg-sales-ops）"
description: "2026-11-17 增量交接：MR !569 三个展示页只读 GET（project-cards/overview-modules/eol-panels）；前端 develop 逐接口核对表（哪些已接、哪些读 localStorage/硬编码/未调用）；前端待改 5 处（含 scope≠criteria）；遗留未验证 gps_pn_lifecycle_overview.business_unit 取值域；worktree/构建脚本与 %SystemDrive% 误建目录等环境坑。"
status: "active"
created_at: "2026-11-17T09:50:05.668Z"
updated_at: "2026-11-17T09:50:05.668Z"
content_hash: "ca2f6df22f6d3745e506847fecf64dd2773d0ef1c5552bfc25f7ebf7951ac451"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java"
session_ids:
  - "mnemon-run-v1-46f65fcd-4ee8-498c-8eea-2df0fbc263e1"
memory_body_ids:
  []
---

# Service Portfolio 展示页只读接口 + 前端对接核对（MR !569，ssg-sales-ops）

2026-11-17 交接记录。前置结论请看既有文档，本文只写增量：

- `35cd7d23-05ba-4fce-a08c-8c42e977e732` —— MyBatis foreach 索引绑定 bug 根因 + hbb/bu/ohPaths 参数落地 + MR !568 实测数字
- `a9793e48-cf89-4069-9871-075abf46c72f` —— OH 树 / 4 个配置页后端化 / HBB CR 修复（MR !564）
- `936f3b21-d21d-435b-acf3-41fc411f9eaf` —— 前后端全量缺口审计（P0、By Offering 范围、localStorage、12 张新表）

## 1. 交付：展示页独立只读接口（MR !569）

- 分支 `feature/sp-api-fix-admin`，提交 `482c9d788 feat(service-portfolio): 展示页独立只读接口（项目卡 / Overview 模块 / EOL 面板）`，1 个文件 26 行（`smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java`）。
- MR !569 → `admin-release`，opened / can_be_merged / has_conflicts=false：https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/569
- 三个 GET 直接委托配置页已在用的查询方法，无新增 SQL：
  - `GET /service-portfolio/project-cards` ← 替代前端 localStorage 项目卡
  - `GET /service-portfolio/overview-modules` ← 复用配置接口
  - `GET /service-portfolio/eol-panels` ← 替代前端硬编码面板
- 接口分层约定（本次确认落地）：`/service-portfolio/*` 是展示页只读；`/service-portfolio/config/*` 是配置页 GET+PUT，写操作仅 admin 可见。
- 编译 `BUILD SUCCESS`（smartop-api 13.35s + admin 2:12，总 2:26）。
- 状态更正：!568 已于 2026-11-17T17:10:00+08:00 merged 进 `admin-release`（sha `ff650a1b7`）；!564 的源分支早前已被删。不要再把 !564 / !568 当"仍开着"。

## 2. 前端 develop（a09da9b9）逐接口核对结论

逐条对照前端 `src/services/portfolio.ts`（475 行）与 `hbbChangeRequest.ts`（345 行）。**接口层基本对得上，不存在大量接口不匹配**；真正的缺口是展示数据前端没接后端。

| 页块 | 前端实际调用 | 状态 |
|---|---|---|
| 顶部 8 张 KPI | `GET /kpi-cards` | 已接 |
| Big Events 路线图 | `GET /roadmap` | 已接 |
| Portfolio Management 外链 | `GET /page-links` | 已接 |
| Concord 列表 | `GET /concord` + `/concord/options` | 已接 |
| Lifecycle 财年图 | `getLifecycleUtilization('PCSD')` + `('ISG')` 各一次 | 已接 |
| Lifecycle Offering Inventory | `GET /oh/tree` + `/offering-trend` | 已接 |
| Big Events 项目高亮卡 | `loadProjectCardConfig()` 读 localStorage | **没接后端** |
| Lifecycle EOL 面板 | `EolReviewPanels.vue` 内硬编码 2 个 | **没接后端** |
| By Offering / By Geo / 合计 | 无任何调用 | **没接后端** |

- 更正一处早期误答：By Offering 表、By Geo、合计三者都来自 `/lifecycle/overview`，不是 `offering-trend`（后者只有财年序列、没有 offering 维度）。前端定义了 `getLifecycleOverview` 但没有任何组件使用，所以该表当前无数据。
- HBB Change Request：前端 24 个方法（projects / requests / submit / status / batch / audit / comments / approvals / attachments / dashboard）路径与字段全部对齐，含 `stage`、`attachmentCount`、`lastCommentTime`、`evidenceFile`、`recordedBy`。
- 悬空（只定义、无组件使用，确认不用可删）：`getInventory`（gpssus/active-pn）、`getOhList`、`getOhCodes`。

## 3. 前端待改 5 处（handoff）

1. By Offering / By Geo / 合计 → 改接 `GET /lifecycle/overview`（带 hbb + bu + ohPaths）。
2. `offering-trend` 调用补 `bu`（连同 hbb），否则 BU 勾选不生效。
3. Big Events 项目卡 → 改调 `/service-portfolio/project-cards`，别读 localStorage。
4. EOL 面板 → 改调 `/service-portfolio/eol-panels`；注意后端字段名是 `scope`，不是前端硬编码里的 `criteria`。
5. 悬空方法 `getInventory` / `getOhList` / `getOhCodes` → 确认不用就删。

## 4. 未验证 / 遗留

- `gps_pn_lifecycle_overview.business_unit` 的实际取值域**未确认**（早期 lifecycle 侧资料出现过 PCSD / ISU / VC 这套词汇）。`bu` 本次只打通了参数链路 + 编译通过，没跑真实数据；若前端 ISG 那张卡为空，大概率是 `bu=ISG` 匹配不到行。下一步：先查该列 distinct 值域，再决定前端换值还是后端加映射。
- 数据来源单一性（迁移注释已定）：`gps_pn_lifecycle_overview` 为唯一来源，不另建目录表，避免两份数据打架（V1.1.113 注释）。

## 5. 环境与工具注意

- worktree 与主工作区并存，易混淆：`C:\Users\63588\lenovo\ssg-sales-ops`（主）vs `.agents\worktrees\sp-api-fix`（本次）/ `service-portfolio-admin` / `fix-coc-flag-sync-scheduler`（其他）。曾误在 `service-portfolio-admin` 里找 controller 找不到、读到旧编译产物。
- 构建脚本：`C:\Users\63588\lenovo\ssg-sales-ops\.tmp\build-sp.cmd`（`chcp 65001` + `cd /d` 到 worktree + IntelliJ 自带 maven `C:\Program Files\JetBrains\IntelliJ IDEA 2026.1.3\plugins\maven-plugin\lib\maven3\bin\mvn.cmd`），命令 `mvn -o -T 4 -pl smartop-api,smartop-service/admin compile -DskipTests`。
- run_code 内 `spawnSync('cmd.exe', ...)` / `spawnSync('C:\\Windows\\System32\\cmd.exe', ...)` 会报 `ENOENT`；改成写 `.cmd` 脚本再调用可用。`C:\Program Files\PowerShell\7\pwsh.exe` 不存在（只有 WindowsPowerShell 1.0 与 cmd）。
- 误建产物待清理：worktree 下出现了 `%SystemDrive%\ProgramData\Microsoft\Windows\Caches\{...}` 共 8 层深目录（脚本变量未展开），`git status` 显示 `?? %SystemDrive%/`，未被提交。
- 本地起服务的端口/参数细节（`sp-run.args`、18085、RabbitMQ 断连刷日志）见 `35cd7d23-05ba-4fce-a08c-8c42e977e732`，此处不重复。
