---
id: "bbcceb8a-b681-4b82-8fc7-3a3d867cba28"
title: "admincontrol 归并修订版：settings 页面并入、968 文件引用矩阵与 @MapperScan 两层通配符硬约束"
description: "admin control 页面后端归并的后续修订：settings 11 个 section 与页签平级→admincontrol 扩为约 10 包；permission/role/notification/scheduler/adminmenu/jira 六包 111 文件零外部引用；@MapperScan(\"com.lenovo.admin.*.dao\") 只匹配一层，搬两层包后 Mapper 全扫不到；/hc/* 一拆两半、5 个僵尸 view 待决策。"
status: "active"
created_at: "2026-11-17T06:25:25.780Z"
updated_at: "2026-11-17T06:25:25.780Z"
content_hash: "a1cbaa93b77fd2bb03c5f82d3b8caa91461250efb044006669a63e0fe7749858"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/AdminApplication.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/bookings"
session_ids:
  - "mnemon-run-v1-36949d21-febb-44d8-ac28-8019307357a5"
memory_body_ids:
  []
---

# admincontrol 归并（修订版）：settings 页面并入、引用矩阵与 @MapperScan 硬约束

日期：2026-11-17。范围：`smartop-service/admin`（服务名 `admin`，包根 `com.lenovo.admin`）的后端包结构归并论证。
本片是文档 `3ebbfe47-3740-42a5-9292-e7808dfa2d60`（第一版：注解 SQL 基线、方法级引用矩阵、`admincontrol.{userlogs,feedback,inspection,tabs}` 四包迁移矩阵、6 步搬移顺序与 SqlTest 安全网）的**后续修订**：用户随后把范围扩到 settings 页面，包数由 4 变为约 10–11 个，并新增两条硬约束。

## 1. 用户最终定性的归属判据（本次新增，最关键）

- 命名：`admin` 是**服务名**，`admincontrol` 是**页面名**；`com.lenovo.admin.admincontrol` 不构成冗余。
- 拆包原则（用户原话，paraphrase）：「他可以拆，但是他首先得先在一个大的包里吧 按照前端页面的菜单 实际去拆包」——先有页面级大包，再按前端菜单在包内下钻，禁止一上来碎成小包。
- **settings 页面与 admin control 页签是平级关系**（用户 2026-11-17 明确）：「setting/settings，其实也是里面的页面 只不过 怕用户误操作 放到了设置里，他其实是平级的 相当与 也是 admin control 的页签」。
- 进一步：「调度中心 权限中心等 他其实也是 admin control 里面的……在后端代码这里 可以放一个包呢」。→ 即 `/setting/settings` 下全部 section 的后端都归属 `admincontrol`，不再论证「permission/scheduler 是领域层不能搬」。

## 2. 前端页面 → 后端接口的映射（按 view 取证，不按 URL 前缀）

页签结构（前端 `@/views/Setting/` 下）：

| 页面 | 内部页签/分区 | 入口组件 |
|---|---|---|
| User Logs | **仅开 4 个**：Data Inquiry Records / Feedback / Page Analytics / Raw Events | `views/user-logs/index.vue` 的 `TABS`；`VIEWS` 里另有 5 个**保留但页签已注释掉**：overview / user / org / behavior / question |
| Issue & Request | 单页约 1521 行 | `usage-issue-feedback/index.vue` |
| Inspection Report | 2 顶级 tab：Smart Data Query（Health Check / Baseline / LUDP Performance）、IntelliData（Engine Inspection）；另有 `?mode=settings` 的 5 个配置 section | `inspection-report/index.vue` |
| People | 3 子页签：Team & Band / HC list / Internal users | `people/index.vue` |
| Admin Control tabs | 单页 | `settings/panels/AdminTabsPanel.vue` |

view/组件 → 实际调用的接口：

```
views/OverviewView.vue        /overview                /analytics/deep
views/UserView.vue            /users                   /analytics/deep
views/OrgView.vue             /org/analysis
views/BehaviorView.vue        /breakdown /analytics/deep /menu /visits
views/QuestionView.vue        /questions /questions/members /analytics/deep
views/QueryView.vue (已开)     /queries /queries/summary
views/PageAnalyticsView.vue   /pages/summary
views/FeedbackView.vue        /feedback-records /feedback/{id}/jira[/preview]
views/RawView.vue             /data
components/FilterBar.vue      /org/options /queries/sources /options
components/modals/DetailModal.vue    /session-detail
components/modals/ExportModal.vue    /export/*
user-logs/index.vue           /dict /queries/review* /queries/failed /profile-groups /internal-users /admin-feedback/*
people/TeamOverridePanel.vue  /hc/overrides* /hc/consultant-managers*
people/HcListPanel.vue        /hc/managers /hc/imports /hc/import/**
people/InternalUsersPanel.vue /internal-users
```

**已修正的误判**：`/hc/*` 的 14 个接口曾被当成 User Logs 的一组，实际一拆两半——`/hc/managers`、`/hc/bands` 是 User Logs 筛选器的下拉数据源；`/hc/overrides*`、`/hc/consultant-managers*`、`/hc/import*` 属 People 页面三个面板。

settings registry（`@/views/Setting/settings/registry.ts` 的 `SETTINGS_SECTIONS`，5 组 11 分区）：
`access-people`: people(page) / access(page) / audit-log(page, read-only)；`notifications`: notifications(page-save)；`jira-intake`: jira(panel) / page-focal(panel) / mode-groups(panel)；`operations`: scheduler(page, immediate) / review-rules(panel)；`workspace`: admin-tabs(panel) / display(panel)。
各分区后端归属：JiraPanel→`jiraSettingsApi.get/save`+`baselineApi.jiraVerify`；ModeGroupsPanel→`reportApi.profileGroups/saveProfileGroups`（chatbi）；ReviewRulesPanel→`reportApi.reviewSettings/saveReviewSettings/reviewBackfill`（chatbi）；AdminTabsPanel→`services/adminMenu` 的 `fetchAdminMenuItemsForAdmin / saveAdminMenuItems / resetAdminMenuItems`（adminmenu）；DisplayPanel 纯前端 localStorage（`user-logs/jiraVisibility`、`kmVisibility`），无后端；PageFocalPanel 只是包了 `usage-issue-feedback/PageFocalEditor.vue`。

## 3. 引用计数证据（扫 968 个 java 文件的 import，统计每个包被其他包引用的类）

| 包 | 文件 | Controllers | 被外部包引用 |
|---|---|---|---|
| permission | 56 | 3（PermissionManagement / RoleManagement / UserGroupManagement） | **0** |
| role | 9 | 1（RoleController） | **0** |
| notification | 20 | 2（NotificationSetting / UserNotification） | **0** |
| scheduler | 10 | 2（SchedulerCenter / CacheAdmin） | **0** |
| adminmenu | 5 | 1（AdminMenuConfig） | **0** |
| jira | 11 | 1（JiraIssue） | **0** |
| user | 23 | 4（Bu / Group / Url / User） | 2 包 4 类：`task`←UserDao,User,UserQueryParam；`datacenter.service`←UserDao |
| menu | 34 | 1（MenuController） | 3 包 5 类：`task`←MenuPbiLinkDao,MenuPbiLinkBean,MenuService；`menurole.pojo`←Menu；`util`←Menu |
| usagefeedback | 12 | 1（UsageIssueFeedback） | 1 包 1 类：`chat.service`←JiraCommentService |
| chatbi | 77 | 3（ChatbiEventReport / ChatbiTracking / FeedbackJiraOpen） | 6 包 17 类（`chat.service` 等） |

结论：**permission、role、notification、scheduler、adminmenu、jira 六个包共 111 个文件零外部引用，闭环**——它们挂在 `com.lenovo.admin` 根下不是技术约束，只是无人过问归属；整套搬入 `admincontrol` 不会产生任何连带修改。
配套验证：全仓检索字面量全限定类名（如 `"com.lenovo.admin.scheduler"`）= **0 处**，即搬包不会打破配置字符串。
方法级反例（第一版已记录）：`ChatbiEventReportService` 28 个 public 方法中仅 `isAdmin()` 被外部调用（aiprobe、datainquiryprobe、hiveperf、scheduler 共 5 处），所以类级引用会高估耦合。

## 4. 目标包结构（约 250 文件，占 968 的 26%）

```
com.lenovo.admin.admincontrol/
├── userlogs/       User Logs（4 已开页签 + 5 保留 view）  chatbi 45 + 评审/导出
├── feedback/       Issue & Request                        usagefeedback 12
├── inspection/     Inspection Report（2 tab × 5 section）  aiprobe 9 + datainquiryprobe 4 + probe 内核
├── people/         People（Team/HC/Internal）             chatbi hc/* + internal-users + profile-groups
├── access/         Access control + Audit log             permission 56 + role 9 + user 20 + menu 30
├── notifications/  Notification settings                  notification 20
├── jira/           Jira integration                       jira 11
├── scheduler/      Scheduler Center                       scheduler 10
├── tabs/           Admin Control tabs                     adminmenu 5
└── display/        Display                                0（纯前端 localStorage）
```
验收标准只有一条：打开某页面包，controller 列表能一一对应前端该页面的页签，且每个接口在前端 view 里点得到。

## 5. 必踩的坑（不处理会启动失败）

1. **`@MapperScan` 只有一层通配符 —— 硬伤**：`AdminApplication.java` 现为 `@MapperScan({ "com.lenovo.admin.*.dao" })`，`*` 只匹配一层；搬到 `com.lenovo.admin.admincontrol.scheduler.dao`（两层）后所有 Mapper 都扫不到，服务起不来。必须改为 `@MapperScan({ "com.lenovo.admin.*.dao", "com.lenovo.admin.admincontrol.*.dao" })` 或 `"com.lenovo.admin.**.dao"`。
2. `@ComponentScan(basePackages = {"com.lenovo.admin", "com.lenovo.*.*"})` 是前缀匹配，子包自动覆盖，**不用改**。
3. 外溢类需抽 `shared/`：`user` 的 `UserDao / User / UserQueryParam`；`menu` 的 `Menu` 实体与 `MenuService`、`MenuPbiLinkDao`。
4. 三个 settings 面板的后端不独立成包（page-focal 落在 feedback、mode-groups 落在 chatbi/people、review-rules 落在 chatbi/userlogs）——「一个分区一个包」不是机械对应，一个接口被两个页面复用是正常的。

## 6. 搬迁顺序与开放议题

- 最小可验证第一步：整体搬 `adminmenu`(5 文件) → `admincontrol.tabs`、`scheduler`(10 文件) → `admincontrol.scheduler`，同步改 `@MapperScan`，编译验证。这两块零外部引用、零字面量，风险最低。
- **僵尸接口待决策**：5 个页签已关闭的 view（overview / user / org / behavior / question）对应后端约 10 个接口（`/overview` `/users` `/org/*` `/breakdown` `/analytics/deep` `/visits` `/questions*`），前端代码保留但页签按钮被注释、用户点不到，仅测试与无头调用在用。搬包前应先决定「恢复还是删除」，否则搬进去的半年后还要清第二次。
- 交接物建议（用户提议、尚未产出）：`docs/admincontrol-map.md`，即「页面 view → Controller → 接口 URL」对照表，确认映射无误后再动代码，避免再次靠 URL 猜测。

## 7. 相关保留路径

- `smartop-service/admin/src/main/java/com/lenovo/admin/bookings` —— 用户指定的「正确形态」参照：一个页面/业务域的全部代码都在同一个包里。
- `smartop-service/admin/src/main/java/com/lenovo/admin/AdminApplication.java` —— `@MapperScan` / `@ComponentScan` 所在处，搬包必改点。
