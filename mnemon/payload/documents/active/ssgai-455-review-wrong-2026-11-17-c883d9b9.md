---
id: "c883d9b9-641a-4c06-a251-55f80717c97b"
title: "SSGAI 455 张 Review-wrong 单关单执行方案与试点验证（2026-11-17）"
description: "把 455 张 [Review wrong] 单关到 Done 且不发平台通知的可执行方案：平台通知取证（JiraIssueMapper 只取两张反馈表，评审单 0 命中）、试点 SSGAI-978 实走 Pending→In Progress→Rejected(Out of Scope)→Done 得 Not a defect、动态 transition id 与 Bug Category/Component 必填坑、剩 454 张的全量方案与待确认项。"
status: "active"
created_at: "2026-11-17T08:43:25.578Z"
updated_at: "2026-11-17T08:43:25.578Z"
content_hash: "7f135ac4b7337433d95ce927bb2ba864862d3d260071beb552d0b6057b4c7232"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/jira/dao/JiraIssueMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/jira/service/JiraIssueNotifyService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/jira/service/JiraIssueSyncService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/jira/schedule/SyncScheduleHandler.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/notification/service/JiraFeedbackEmailNotifier.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/jira/FeedbackNotificationService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/jira/dao/JiraIssueStatusCache.java"
session_ids:
  - "mnemon-run-v1-9e154ef1-358b-4e18-95cb-74ccdb1d0712"
memory_body_ids:
  []
---

# SSGAI 455 张 Review-wrong 单关单执行方案（试点已验证，全量待确认）

状态（2026-11-17 16:43）：试点单 SSGAI-978 已关到 Done 并验证；剩 **454 张** 待用户点头后全量执行。用户的硬约束是「绝对不能发邮件和站内信」（后放宽为「Jira 发邮件没事，怕的是平台发」）。

关联文档：
- **bb58fc3f-4d3b-48b3-8679-da5341fd6b89** —— 本链路的前置调研（Jira 11.3.12 doTransition 已移除 notifyUsers、账号权限、完整工作流图、Outgoing Mail / Bulk change 两条路线）。本文是它的**执行验证与修正**，多处结论被实测推翻。
- **2a95b6da-13f6-48f7-ace0-52cef76ec38f** —— 这 455 张单是怎么被自动建出来的（建单条件、台账、assignee=longyu3）。

## 一、平台侧通知取证：这 455 张单不在任何通知链路上（这是"不发邮件"担忧的真正解答）

平台的通知机制（定时任务 `jira-issue-status-sync`，cron 10 分钟）链路：

```
ScheduledTask(jira-issue-status-sync)
  → JiraIssueSyncService 刷新 ssg_jira_issue 中的状态
  → JiraIssueNotifyService.notifyChanges(keys)   // 比对 last_notified_status，变了才发
       ├─ 问题上报：UsageIssueFeedbackNotificationService.notifyStatusChanged → 站内信 + 邮件
       ├─ 点踩反馈：FeedbackNotificationService.notifyJiraStatus / notifyJiraDone → 站内信 + 邮件
       └─ 自定义规则：dispatchCustomRules（读 ssg_notification_custom_rule）
```

决定性证据在**取数 SQL**，`smartop-service/admin/src/main/java/com/lenovo/admin/jira/dao/JiraIssueMapper.java:43-48` 只联合两张表：

```sql
FROM (SELECT jira_key FROM public.ssg_feedback_jira_issue
      UNION
      SELECT jira_key FROM public.ssg_usage_issue_feedback)
WHERE NOT EXISTS (SELECT 1 FROM public.ssg_jira_issue j WHERE j.jira_key = t.jira_key)
```

**没有 `ssg_chatbi_review_jira`**。生产库实测（SSGAI-978..1433 区间内）：

| 检查项 | 结果 |
|---|---|
| ssg_chatbi_review_jira（评审台账） | 455 行 |
| ssg_feedback_jira_issue（点踩，有通知） | 129 行有单号，落在区间内 **0** |
| ssg_usage_issue_feedback（问题上报，有通知） | 22 行有单号，落在区间内 **0** |
| ssg_jira_issue（平台共享状态表） | 143 行，落在区间内 **0**（平台连状态都不缓存） |
| ssg_notification_custom_rule（自定义规则） | **0 条** |

另：`smartop-service/admin/src/main/java/com/lenovo/admin/notification/service/JiraFeedbackEmailNotifier.java:60 (jiraCreated)` / `:72 (jiraDone)` 在整个 admin 模块 **0 调用点**，是死代码。评审侧的 `JiraIssueStatusCache`（2 分钟缓存）、`enrichJiraStatuses`（`JiraIssueViewDecorator.decorate`，只塞展示字段）全为只读。

结论：**关这 455 张单，平台侧邮件 0、站内信 0，平台数据一行不用改；需要挡的只有 Jira 自己的状态变更通知。**

## 二、关单路径：实测走 Rejected 中转（推翻了前置文档的两处结论）

试点号 `SSGAI-978`（起始状态 **Pending**，非 New）最终落点：**Done / Resolution = Not a defect / Reject Reason = Out of Scope / assignee = longyu3**，changelog 4 条：
`New -> Pending -> In Progress -> Rejected -> Done`

实际路径：

```
Pending --221--> In Progress --71(Reject Reason=Out of Scope)--> Rejected --131--> Done
```

**修正 1：transition id 依赖当前状态，必须动态查询。** 前置文档记的 `New --61--> In Progress` 在 Pending 上直接 400：
`It seems that you have tried to perform a workflow operation (In Progress) that is not valid for the current state of this issue`；而 Pending 下 `In Progress=221`。方案里因此**绝不写死 id**，每单先 `GET /rest/api/2/issue/{key}/transitions` 取可用流转。

**修正 2：Ready for Testing 并非无必填字段。** 前置文档称 `fields:{}`，实测 `In Progress --81--> Ready for Testing` 返回：
`400 {"errors":{"customfield_11850":"Bug Category is required.","components":"Component/s is required."}}`
Bug Category（customfield_11850）有 16 个选项、Component/s 有 6 个（BDS AI Agent(BDSAI)、CDM-FI(CDMFI)、I&C ROW(INCROW)、Magellan DI Hub(DIPORTAL)、Service Portfolio Enablement Platform(SPEP)、SSG portal）。而生产上 `jira.bug-category` / `jira.components` 两项配置都是空的 —— **硬填假值会污染 SSGAI 的组件与缺陷分类统计**，这是改道 Rejected 的主因。

**修正 3：走 Rejected 不再"留下错误历史"，反而更准确。** 前置文档以"会在历史留下被拒绝记录"为由否掉 3 步路径，实测 `Rejected --131 Done` 带 post-function **自动把 Resolution 设成 `Not a defect`** —— 对"不该建的单"语义比 `Fixed` 准确得多。Reject 唯一必填是 Reject Reason（customfield_18200，11 选项：ID Request / Change Request / Out of Scope / Test Script Issue / Wrong Test Data / Question / Cannot Reproduce / Incomplete Description / Wrong User Operation / Duplicated / Won't Fix），选 **Out of Scope**，不碰任何业务字段。

## 三、全量方案（待用户确认后执行）

| 项 | 方案 |
|---|---|
| 范围 | 台账 455 张，SSGAI-978 已完成，剩 **454** |
| 起点分流 | 按当前状态：多数 New（transition 61）、少量 Pending（221） |
| 每单步骤 | 动态查可用流转 → In Progress → Rejected(Out of Scope) → Done |
| 幂等/容错 | 每步先回读状态再决定动作；失败重试 1 次；单张失败不影响其他 |
| 并发 | 建议 4 并发（约 4~6 分钟）；保守可串行（约 12~18 分钟） |
| 产物 | CSV：单号 / 起始状态 / 每步耗时 / 最终状态 / 失败原因 |
| 不动 | 平台数据一行不改，`ssg_chatbi_review_jira` 台账不改（只记建单） |

**回滚**：状态可反向走（Done → Pending → …），但每反向一步都会再产生 changelog 和通知，实质是"再跑一遍"，所以试点确认后上量比事后回滚划算。

待用户回答的两件事：① 统一用 `Out of scope` 是否可接受；② 试点单 3 次状态变更实际收到几封邮件（聚合成 1 封还是 3 封），这决定全量邮件量级 30 倍差异（3 封/单 ≈ 1362 封 vs 几十封）。

## 四、可复用命令与字段速查

- 查流转：`GET /rest/api/2/issue/{key}/transitions?expand=transitions.fields`
- 执行流转：`POST /rest/api/2/issue/{key}/transitions`，body `{"transition":{"id":"..."},"fields":{...}}`（无 notifyUsers 参数，见 bb58fc3f）
- 回读：`GET /rest/api/2/issue/{key}?fields=status,resolution,assignee,summary,customfield_11850,customfield_18200`
- Resolution 列表：`GET /rest/api/2/resolution`（Fixed / Duplicate / …）
- 本账号权限（已实测）：`CLOSE_ISSUES=true, DELETE_ISSUES=true, TRANSITION_ISSUES=true, BULK_CHANGE=true`；`ADMINISTER=false, ADMINISTER_PROJECTS=false`（所以无法用 notifyUsers 抑制通知）

注：`ssg_notification_custom_rule` 实际列名与代码假设不符，按 `name` 查询报 `column "name" does not exist` —— 需要该表数据时先确认真实列名。
