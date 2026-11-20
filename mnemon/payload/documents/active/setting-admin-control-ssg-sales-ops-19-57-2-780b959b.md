---
id: "780b959b-d140-4162-9c67-a192e70d2b96"
title: "路由改名交付：/setting → /admin-control（ssg-sales-ops，前端 19 文件/57 处 + 后端 2 处）"
description: "把 Admin Control 路由 /setting 全量改为 /admin-control 的交付记录：路由表/侧边栏/深链/页面跳转共 57 处 19 文件、后端仅注释与测试桩 2 处、为何无需改库（configRoute 全是 hash 路由、url 限 http/https、ssg_menu 用 nav_item）、刻意保留 views/Setting 目录与不加 redirect 的代价、UU 未合并文件与无关脏改动的提交注意事项。"
status: "active"
created_at: "2026-11-20T03:47:04.170Z"
updated_at: "2026-11-20T03:47:04.170Z"
content_hash: "8f982746cbc4090fc3d07b67a2165347fe65a05ebb545425acc64fc40099098a"
source_paths:
  - "src/router/index.ts"
  - "src/App.vue"
  - "src/components/SideBar/index.vue"
  - "src/constants/feedbackMenuCatalog.ts"
  - "src/constants/adminMenu.ts"
  - "src/views/UsageIssueFeedback/index.vue"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/adminmenu/service/AdminMenuConfigService.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/scheduler/service/SchedulerCenterServiceTest.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.102__Create_ssg_admin_menu_item.sql"
session_ids:
  - "mnemon-review-v1-6160472a-f75d-4a4e-972d-1895a4dbfcf4"
memory_body_ids:
  []
---

# 路由改名交付：`/setting` → `/admin-control`（ssg-sales-ops 前端 19 文件 + 后端 2 处）

来源：2026-11-20 一轮任务。用户指令（checkpoint ev.815）：先把本地前端跑起来（端口挂了），然后「直接全改完 别遗留 就admin-control」——即 `/setting` 全量改名成 `/admin-control`、不保留兼容路径、范围只限 admin-control。工作区 `C:\Users\63588\lenovo\ssg-sales-ops`（Vue 3 + Vite 7.3.8 前端，`smartop-service/admin` Java 后端）。

## 1. 改法（逐文件人工替换，非全局 sed）

| 旧 | 新 |
|---|---|
| `/setting` | `/admin-control` |
| `/setting/settings?panel=xxx` | `/admin-control/settings?panel=xxx` |
| `/setting?view=xxx` / `/setting?tab=xxx` | `/admin-control?view=xxx` / `?tab=xxx` |
| route name `setting` / `setting-settings` | `adminControl` / `adminControlSettings` |

合计 **57 处 / 19 个前端文件**：
- `src/router/index.ts:193-204`：path 与 name
- `src/components/SideBar/index.vue:130`：底部菜单 `path: '/admin-control'`（`navItem: 'admin_control'`，`requiresAdmin`）
- `src/App.vue:15,24`：Admin settings 收起全局菜单的判断（`route.path === '/admin-control/settings'`、`startsWith('/admin-control/settings/')`，返回 `/admin-control` 时恢复）
- `src/constants/feedbackMenuCatalog.ts`：13 处（`'/admin-control/settings': 'Admin Control · Settings'` 映射、`/setting?view=` 深链构造函数、页签反查）
- `src/constants/adminMenu.ts`：深链注释与默认页签说明
- 页面跳转：`views/Setting/reports/scheduler-center/index.vue`（3 处 push）、`inspection-report`、`notification-settings`（router-link）、`permissions/AuditLogPage.vue`、`settings/panels/JiraPanel.vue`（panel 跳转）、`views/Setting/index.vue`（齿轮按钮 + 深链注释）、`settings/index.vue`、`settings/registry.ts`、`reports/people`、`reports/permissions`、`reports/user-logs/constants.ts`、`reports/usage-issue-feedback/styles.less`
- `src/views/AskPreview/components/AskNav.vue`、`src/views/UsageIssueFeedback/index.vue:431,437`（`startsWith('/setting')` 路径判断）

## 2. 关键结构决策（为什么这样挂）

- **Admin settings 挂在 `/admin-control/settings` 而不是独立路径**：侧边栏高亮按路径前缀匹配，独立路径会让 Admin Control 不高亮。该注释已写进 `router/index.ts:198-199`。
- Admin Control（`/admin-control`，看数据，顶部页签）与 Admin settings（`/admin-control/settings`，改配置，左栏 11 分区）是两个路由、两个组件，改名时必须分别处理，混讲最容易错（见相关文档）。

## 3. 后端只动 2 处（并说明为何无需改数据）

- `smartop-service/admin/src/main/java/com/lenovo/admin/adminmenu/service/AdminMenuConfigService.java:20`：类注释 `Admin Control（/setting）` → 新路径
- `smartop-service/admin/src/test/java/com/lenovo/admin/scheduler/service/SchedulerCenterServiceTest.java:36`：测试桩 `when(handler.configRoute()).thenReturn("/admin-control?view=trino")`。已核对第 28-50 行断言只覆盖 key/displayName/businessModule/labels/supportsManualRun，**不断言 configRoute**，改动安全。
- **不动数据**的依据：各 handler 的 `configRoute()` 实际返回全是 hash 路由——`#scheduler/business-monitoring`、`#scheduler/business-chatbi`、`#scheduler/business-data-sync`、`#inspection/settings/schedule`、`#inspection/settings/intellidata`、`#inspection/settings/jira`、`#inspection/results/ludp`；`ssg_admin_menu_item.url` 只接受 http/https 绝对地址（见 `V1.1.102__Create_ssg_admin_menu_item.sql` 的类型/约束注释），表里不会存 `/setting` 相对路径；`ssg_menu` 用 `nav_item` 作权限 key，与 path 无关。
- 历史迁移 SQL（`V1.1.102__Create_ssg_admin_menu_item.sql:2,51` 的 `COMMENT` 里仍写 `Admin Control（/setting）`）**刻意不改**：历史文件应保持原样；`AdminMenuConfigService.java:20` 的注释是活代码注释，所以改了。

## 4. 刻意保留的两件事（代价已告知用户）

1. **目录 `src/views/Setting/` 不改名**：不对应 URL、不影响界面；改它要 `git mv` 60+ 文件 + 改 30 处 `@/views/Setting/...` import，MR diff 会变成整片 rename。另 `@/assets/images/svg/setting.svg` 是图标资源名，也保留。
2. **不加 `/setting → /admin-control` redirect**（遵「别遗留」）。代价：外部 Jira 单、IM 收藏、旧文档里的链接会落到 404 空白页；要救回只需两行。

## 5. 验证与坑

- 终检口径：前端原始 `/setting` 行 37 条、真实遗留 0；`/Setting` 大写 50 处、URL-like 可疑 0；后端 java `/setting` 仅注释 2 处（已改）+ 测试桩 1 处（已改）；svg 路径含 setting 0（仅剩 `setting.svg` 资源 import）。
- **HTTP 200 不能证明路由还在/已删**：dev server 有 SPA fallback，`/setting` 与 `/admin-control` 都返回同一份 418 字节 shell（`<title>SSG SPARK`）。路由是否生效只能在浏览器里看，不能靠状态码。
- Vite 冷启 14.3s；改代码后首次加载 `/src/App.vue` 的依赖链要编译约 40 秒（`Invoke-WebRequest` 会先超时，43.8s 才 OK），之后毫秒级——别误判为服务挂了。
- ripgrep 不支持 lookahead：`(?:/setting(?![A-Za-z0-9_.]))` 报 `look-around ... not supported`，需 `--pcre2`，或先粗 grep 再在 JS 里过滤。

## 6. 提交前必须处理的现场状态

- `src/constants/feedbackMenuCatalog.ts` 仍是 **UU 未合并状态**（stages :1/:2/:3 = 601/599/603 行，worktree 601 行，内容是此前手工合好且无冲突标记，本轮又在其上改了 13 处路径）。**提交前需 `git add` 一次标记已解决**，否则 git 阻止提交。
- 工作区还混有上一轮的无关改动：前端 `views/Setting/reports/user-logs/components/views/QueryView.vue` (+276/−128，指标改动)、后端 `ChatbiEventReportService.java`、`mapper/chatbi/ChatbiEventQueryMapper.xml`。建议分 commit 提交，否则路由改名 MR 会被无关 diff 污染。

## 7. 相关文档

- `e5718bfd-fbfd-4213-aa85-72f8bcf99b19`：Admin Control / Admin settings 功能全景清单——该文档按 `/setting`、`/setting/settings` 描述两层路由，**本次改名后其路由写法已过期**，功能内容仍有效。
- `3ebbfe47-3740-42a5-9292-e7808dfa2d60`：admin 模块包结构重组分析（涉及 `adminmenu` → `admincontrol.tabs` 搬迁，与本次 `AdminMenuConfigService` 注释改动同一文件）。
