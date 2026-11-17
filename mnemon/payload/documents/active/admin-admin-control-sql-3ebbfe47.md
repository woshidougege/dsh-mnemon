---
id: "3ebbfe47-3740-42a5-9292-e7808dfa2d60"
title: "admin 模块（admin control 页面）接口与包结构重组分析：注解 SQL 基线、方法级引用矩阵与迁移顺序"
description: "admin control 页面 61 个接口与包分散度分析：89 dao/686 语句/6306 行注解 SQL 基线；类级引用判据误判被方法级推翻（ChatbiEventReportService 28 方法仅 isAdmin() 外泄）→ admincontrol.{userlogs,feedback,inspection,tabs} 迁移矩阵、6 步纯搬移顺序与 SqlTest 安全网。"
status: "active"
created_at: "2026-11-17T06:17:48.582Z"
updated_at: "2026-11-17T06:17:48.582Z"
content_hash: "dc51f6b28760a5c456340f5ffb55862b73f7a9c8b251d62def46187ec40f7313"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/controller/ChatbiEventReportController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/dao/UsageIssueFeedbackMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/service/UsageIssueFeedbackService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/controller/UsageIssueFeedbackController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/adminmenu/service/AdminMenuConfigService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/adminmenu/controller/AdminMenuConfigController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/probe/service/ProbeBaselineService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/bookings"
session_ids:
  - "mnemon-run-v1-d2c83131-2623-4086-a620-04f6d14783f7"
memory_body_ids:
  []
---

# admin 模块（admin control 页面）接口与包结构重组分析

日期：2026-11-17。范围：`smartop-service/admin`（服务名 `admin`，包根 `com.lenovo.admin`；`admincontrol` 是**页面**概念，不是服务名）。

## 1. 用户提出的问题（原始意图）

- **注解 SQL**：admin control 页面所有接口的 SQL 都写在 MyBatis 注解里，问「是不是不太好」「写 xml 文件是不是比较好」，要求分析。
- **包分散**：admin control 的接口没有按业务模块归到一处，而是很分散；对照 `smartop-service/admin/src/main/java/com/lenovo/admin/bookings`——用户原话（paraphrase，2026-11-17）：「他的所有代码全在这个里面。这才对」，要求分析是否真的分散。
- **拆包原则**：用户明确「他可以拆，但是他首先得先在一个大的包里吧 按照前端页面的菜单 实际去拆包」。即：先有一个页面级大包，再按前端菜单在包内下钻，而不是一上来就碎成很多小包。
- **命名澄清（关键纠正）**：用户指出「admin 是这个服务的名字，里面的 admincontrol 是没问题的」「admin control 里面 哪有 chatbi 呢 只有 userlogs feedback 等」。→ 包路径 `com.lenovo.admin.*` 里的 `admin` 是服务，不是页面；`admincontrol` 作为页面大包成立，`admin.admincontrol` 不构成命名冗余。

## 2. 注解 SQL 现状基线（逐 dao 目录统计）

```
dir                  files  stmts  sqlLines  scripts
chatbi\dao              25    241     2039      108
usagefeedback\dao        4     38      521       14
aiprobe\dao              1      1        6        0
datainquiryprobe\dao     2      8       43        0
adminmenu\dao            1      3       19        1
permission\dao           6     84      874       27
menu\dao                 3     56      671       21
user\dao                 5     41      257       18
notification\dao         4     22      195        4
scheduler\dao            0      0        0        0
biadmin\dao              1      1        8        1
insightshub\dao          8     71      762        9
serviceportfolio\dao    21     77      285       12
datacenter\dao           8     43      626       17
TOTAL                   89    686     6306      232
```

（`scripts` 列为注解 SQL 中出现 `<script>` 包裹的计数。）

- 规模集中在 4 处：`chatbi/dao` 2039 行（占 32%）、`permission/dao` 874、`insightshub/dao` 762、`menu/dao` 671、`datacenter/dao` 626、`usagefeedback/dao` 521。
- 232 处 `<script>` 说明动态 SQL 早已在注解内用 script 标签拼装，注解 SQL 并不是「纯静态小查询」，改 XML 的收益与风险都要按这个规模估。
- 注解 SQL 复用共享常量有已验证的坑：`ChatbiEventQueryMapper.STATUS_CATEGORY_SQL` / `RECOVERED_SQL` 在 `<script>` 里写作 `&lt;`/`&gt;`，注解 SQL 不经 XML 解析，必须 `.replace("&lt;","<").replace("&gt;",">")`，否则 PostgreSQL 直接语法错（详见文档 `64ae3942-bdb8-4694-bb7e-9ef24b218c19`）。
- 「是否整体迁移到 XML」的最终分析结论未随本次检查点保留 → **视为开放议题**，需重新评估（可迁移性判据：SQL 里是否复用常量片段、是否需要 `<if>/<foreach>` 动态分支、是否有对应 `*SqlTest` 可直接做金标准）。

## 3. ChatbiEventReportController：URL 已经天然分组（61 个接口）

`@RequestMapping("/chatbi/events/report")`，1823 行、61 个接口；全模块 63 个 Controller 全部挂在 `/api/admin/**`。

| 子域 | 数量 | URL |
|---|---|---|
| hc | 14 | `/hc/import/preview`、`/hc/import`、`/hc/imports`、`/hc/managers`、`/hc/bands`、`/hc/overrides`(GET/POST/DELETE)、`/hc/consultant-managers`(+`/{itcode}/line`、`/sync`) |
| queries | 13 | `/queries`、`/queries/summary`、`/queries/review`(GET/PUT/DELETE)、`/queries/review/settings`、`/queries/review/rules`、`/queries/review/backfill`、`/queries/review/comments`、`/queries/sources`、`/queries/failed` |
| export | 4 | `/export/lock`、`/export`、`/export/progress`、`/export/count` |
| feedback | 3 | `/feedback/{feedbackId}/jira`(+`/preview`、`/preview/discard`) |
| questions | 3 | `/questions`、`/questions/members`、`/questions/rebuild` |
| org | 3 | `/org/analysis`、`/org/options`、`/org/sync` |
| admin-feedback | 2 | `PUT/DELETE /admin-feedback/{sessionId}/turns/{turnId}` |
| profile-groups | 2 | GET/POST `/profile-groups` |
| internal-users | 2 | GET/POST `/internal-users` |
| 其余 15 个单接口 | 15 | `/data`、`/feedback-records`、`/overview`、`/breakdown`、`/users`、`/visits`、`/pages/summary`、`/session-detail`、`/backfill`、`/failed-jira/scan`、`/review-jira/scan`、`/menu`、`/dict`、`/options`、`/analytics/deep` |

拆类方案（URL 一个不改，前端零改动，Service 同步拆）：`UserLogsHcController`(14) / `UserLogsQueryController`(13，含当前分支在改的评审规则) / `UserLogsFeedbackController`(6) / `UserLogsAnalyticsController`(~18) / `UserLogsExportController`(4) / `UserLogsOpsController`(3)。

## 4. 决定成败的方法：类级引用 → 方法级引用（结论反转）

- 第一轮按「类是否被其他包 import」判断，得到 `ChatbiEventReportService` 被 6 个包引用 ⇒ 不能搬 ⇒ 不建 `admincontrol`。
- 用户纠正后按**方法级**重算：`ChatbiEventReportService` 2725 行、**28 个 public 方法，外部只调用 1 个** `isAdmin()`（调用方：aiprobe、datainquiryprobe、hiveperf、scheduler，共 5 处）。⇒ 结论反转：它是纯页面服务，可搬；唯一前置是把 `isAdmin()` 抽成 `AdminAccessChecker` 放进 `permission` 包（该包已有 `PermissionManagementDao.isAdminByitcode`）。
- **可复用教训**：判断包搬迁边界只能看「方法级外部调用」，不能看「类级引用」。一个 2700 行、28 方法的类被 5 个包共享，很可能只共享了 1 个方法；用类级引用判据会把整页代码误判为领域内核。
- 另一条同样重要的教训：先用「页面里显示哪个接口」命名，会撞上「这个类还服务别处」；正确顺序是**先量方法级耦合，再决定这个类属于页面还是领域**。

## 5. 共享件的真实耦合面（方法级，方向：调用方 → 被调类.方法）

| 共享类 | 外部调用方 | 被调方法 |
|---|---|---|
| `chatbi/dao/ChatbiEventMapper` | chat（AnswerFeedbackService / AskAdminService / AskMeService / AskSettingsService） | `insertIgnore()`；另 `findIdByEventId()`、`selectProbeSessionIds()` 由 chatbi 内部调用 |
| `chatbi/dao/ChatbiOrgProfileMapper` | aiprobe:AiProbeReportController | `listKnownItcodes()`（其余方法 chatbi 自用） |
| `chatbi/service/ChatbiOrgProfileService` | user:UserController | `upsertBasic()`（`analysis/options/sync` 仍由 chatbi 内部 Controller 调） |
| `chatbi/service/ChatbiExportLockService` | usagefeedback:UsageIssueFeedbackController | `snapshot()/tryAcquire()/release()`；`updateProgress()` 由 chatbi:ChatbiEventExportService 调 |
| `chatbi/service/ChatbiExportProgressService` | usagefeedback:UsageIssueFeedbackController | `start()/finish()/fail()/snapshot()`；`update()` 由两个导出 Service 调 |
| `chatbi/dao/ChatbiFeedbackMapper` | chat:AnswerFeedbackService | `selectMine()/selectOpenCases()/countDislikes()` |
| `chatbi/support/ChatbiTurnTrace` | chat:ChatController / BaseChat | `getStartedAt()/stepSuccess()/stepFailed()/setLlmAttempts()` |
| `chatbi/dao/FeedbackJiraTicketMapper` | 反向：chatbi:FeedbackJiraCreateGate / FeedbackJiraTicketService 调它 | `lockSessionTurn()/selectSyncState()/selectLatestJiraBySessionTurn()/attachExistingJira()/markSuperseded()/markCreated()/selectPreview()/savePreview()` |
| `usagefeedback/dao/UsageIssueFeedbackMapper` | 反向：jira:JiraIssueNotifyService 调 `selectNotifyCandidates()/markDoneNotified()/markStatusNotified()`；usagefeedback 内部调其余 5 个 | `selectIsAdmin()/selectUserProfile()/countAdminList()/selectAdminList()/selectForComment()` |
| `usagefeedback/notification/UsageIssueFeedbackNotificationService` | jira:JiraIssueNotifyService | `notifyStatusChanged()`（`notifyJiraFailed()/notifyJiraCreated()` 由 usagefeedback 自用） |
| `usagefeedback/service/JiraCommentService` | chat:DislikeJiraCommentService；usagefeedback:UsageIssueFeedbackController | `listByJiraKey()/openImageForJiraKey()/postByJiraKey()`；`list()/post()/openImage()` |

**结论：真正的跨页共享只有 3 类**——导出锁/进度、Jira 通知与评论、采集侧写入（`insertIgnore`、`ChatbiTurnTrace`）。其余「被其他包引用」都是 DTO/常量/配置，跟着搬即可。

## 6. chatbi 包的三类内容（77 个文件：53 仅包内、24 被其他包引用）

| 类别 | 代表 | 归属 |
|---|---|---|
| User Logs 页面（约 45 个） | `ChatbiEventQueryMapper`(2034 行)、`ChatbiEventReportService`(2725)、`ChatbiEventExportService`(1912)、`ChatbiHcService`(916)、`ChatbiQuestionReviewService`、`ChatbiReviewJiraCreator`、`ChatbiExport*` 以外的大部分 service/jira 类 | `admincontrol.userlogs` |
| 数据采集管道（约 8 个） | `ChatbiTrackingController/Service`、`ChatbiSessionBackfillService`、`ChatbiSessionBackfillScheduleHandler`、`SparkHistoryReplayException`、`SparkSessionHistoryClient`、`ChatbiAgentBackfillMapper` | 留 `chatbi`（用户在聊天时写事件、回填历史会话，与管理页面无关） |
| 页面间共享基础设施（约 12 个） | 导出锁/进度、`ChatbiEventMapper.insertIgnore`、`ChatbiOrgProfileService.upsertBasic`、`ChatbiFeedbackMapper`、`ChatbiTurnTrace`、`ChatbiAuthCookieFilter` | 留 `chatbi` 或另立 `shared` |

其余包的包内/包外分布：`usagefeedback` 9/3、`aiprobe` 9/15、`datainquiryprobe` 4/11、`adminmenu` 5/0、`probe` 11/14。

## 7. 迁移矩阵（实测可得性）

| 目标子包 | 来源 | 能搬 | 卡点 |
|---|---|---|---|
| `admincontrol.tabs` | `adminmenu`(5) | **5/5** | 无：0 外部引用，有测试，最佳第一个样板 |
| `admincontrol.feedback` | `usagefeedback`(12) | **9/12** | `UsageIssueFeedbackMapper`（jira 用 3 个通知方法）、`JiraCommentService`（chat 用）、`UsageIssueFeedbackNotificationService`（jira 用）留原位 |
| `admincontrol.userlogs` | `chatbi`(77) | **~45/77** | 先抽 `isAdmin()`；其余是采集管道 + 共享设施 |
| `admincontrol.inspection` | `aiprobe`(24)+`datainquiryprobe`(15) | **13/39** | 两包约 60% 是 probe 体系共享件（`SmartDqProbeClient`、`ProbeSettingService`、`DataInquiryProbeQuestionService`、`ProbeBaselineService`…被 chat/hiveperf 反调），只能搬 controller + 页面专用 service |

`/setting/settings` 的 11 个分区**不进** `admincontrol`：`permission`/`menu`/`user`/`notification`/`scheduler` 是领域入口，登录鉴权、菜单渲染、通知发送等业务侧持续在用，不是页面独占。

## 8. 目标结构

```
com.lenovo.admin.admincontrol/
├── userlogs/      45 个：controller/（拆成 6 个）service/ dao/ pojo/ jira/
├── feedback/       9 个：controller/ service/ dao/
├── inspection/    13 个：controller/ service/
└── tabs/           5 个：controller/ service/ dao/ dto/

com.lenovo.admin.chatbi/    留：采集管道 8 个 + 共享基础设施 12 个
com.lenovo.admin.probe/     巡检内核不动
```

## 9. 落地顺序（每步可单独上线，全部是纯搬移）

1. `adminmenu` → `admincontrol.tabs`（5 文件，约半小时）：验证包形态与 import 成本，风险为零。
2. 抽 `isAdmin()` → `AdminAccessChecker`（放 `permission` 包，复用 `PermissionManagementDao.isAdminByitcode`）：后续所有搬迁的前置，单独一个 MR 便于回滚。
3. 拆 `ChatbiEventReportController` 1823 行 → 6 个 Controller（URL 不变，Service 同步拆）：先在 `chatbi/controller` 内拆开，不搬包，与手上的 review rules 分支冲突最小。
4. `usagefeedback` → `admincontrol.feedback`（9 文件）：整包搬，留 3 个共享件。
5. `chatbi` 的 45 个页面文件 → `admincontrol.userlogs`（最大一步，独立 MR，纯 import 变更）。
6. `aiprobe`/`datainquiryprobe` 的 Controller 搬进 `admincontrol.inspection`，内核留 `probe`。

## 10. 约束与安全网

- 每一步**只做搬移**：不改 URL、不改 SQL、不改逻辑，回归范围才可控。
- 已有 13 个 `*SqlTest` 与 `MapperSqlRender` 作为搬迁后的金标准——搬完跑一遍，SQL 渲染结果必须一字不差。
- 拆 Controller 时 `@RequestMapping("/chatbi/events/report")` 保留在每个类上，前端零改动（当前分支正在改 `/queries/review*`，拆类恰好趁手）。
- 拆包 MR 与功能 MR（评审规则）必须分开，否则 review 与回滚都会纠缠。
- 若最终仍要「按菜单建包」，只在**纯配置**子域成立（tabs / reviewrules / jiraconfig / display）；access control、people、notification、scheduler 四个不进页面包。
- 曾考虑用 `docs/admin-control-api.md`（页签 → URL 前缀 → 包 → Controller）做「索引」替代搬包；在方法级数据出来、确认可搬之后，该方案降级为搬迁完成后的补充文档，而非替代方案。

## 11. 维护归属（提交历史信号）

| 文件 | commits | 作者分布 |
|---|---|---|
| `ChatbiEventQueryMapper.java` | 72 | 70× renxp4 |
| `ChatbiEventReportService.java` | 70 | 69× renxp4 |
| `ChatbiEventExportService.java` | 46 | 43× renxp4 |
| `UsageIssueFeedbackController.java` | 23 | 22× renxp4 |
| `UsageIssueFeedbackService.java` | 22 | 21× renxp4 |
| `AdminMenuConfigService.java` | 2 | 2× renxp4 |

→ 这批页面文件基本单人维护（renxp4，显示名 Xiangpeng XP4 Ren），包结构重组的冲突成本低，但必须先与之对齐再动第 5 步（45 文件大搬）。

## 12. 相关文档

- `64ae3942-bdb8-4694-bb7e-9ef24b218c19`：ChatBI 自动评审规则可配化交付记录。含本分析第 3 步涉及的主要改动文件（`ChatbiEventReportController`、`ChatbiAutoReviewRuleMapper`、`ChatbiQuestionAutoReviewMapper`）与注解 SQL 转义坑，拆包时需以它为冲突基线。

## 13. 未决事项

- 注解 SQL 是否/如何迁移到 XML：只有基线统计与已知转义坑，缺少逐文件收益评估与结论。
- 是否真要做第 5 步（45 文件搬进 `admincontrol.userlogs`）：先做第 3 步拆类，之后重新判断——拆完类后若「同类留在 `chatbi` 里也不难受」，可不搬包。
- 用户尚未下达开工指令；此前询问「要我从第 1 步开始动手吗」，回答前不要擅自改动代码。
