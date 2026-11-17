---
id: "2a95b6da-13f6-48f7-ace0-52cef76ec38f"
title: "评审 WRONG 自动建单上线后生产批量建单现状 + Jira 设置页 Assignee/Routing 组织方式"
description: "2026-11-17 开关打开后生产 450 张单（SSGAI-1424~1428）的证据与剩余范围；JiraPanel.vue 中 Automation 只读 Assignee 与 Routing 可编辑 assignee 的分布原因、page focal 优先级，以及 save() 校验/文案 bug。"
status: "active"
created_at: "2026-11-17T03:09:36.607Z"
updated_at: "2026-11-17T03:09:36.607Z"
content_hash: "2c36cbb1767f0f7e7386923e5b8e5b6db2ba2a3606129be27020696520ad18ce"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiReviewJiraMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/jira/ChatbiReviewJiraCreator.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/jira/ChatbiReviewJiraScheduler.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/jira/ChatbiReviewJiraService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/probe/service/ProbeSettingService.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.127__Create_chatbi_review_jira.sql"
session_ids:
  - "mnemon-run-v1-2bfd5159-2644-4d9d-8435-79d5c28d898b"
memory_body_ids:
  []
---

# 评审自动建单上线后生产批量建单现状 + Jira 设置页 Assignee 分布（2026-11-17）

状态：功能已合并到 master 并在生产运行中；止血方案与前端小改（校验/文案、Assignee 是否并入 Automation）仍待用户选择。

关联文档：
- **cc5d103b-628e-4216-a1a6-08ef73e07f46** —— 该功能的最终设计口径、三支 MR（!559 / !1054 / !1055）、新表 V1.1.127、三级查重与 advisory lock 的完整记录。本文只补充「上线后的生产实际行为」与「设置页 Assignee 组织方式」两块增量。

## 一、建单条件（复核代码后的确认口径）

唯一过滤条件在 `ChatbiReviewJiraMapper.java` 的候选扫描 SQL（类注释第 17 行原文「自动评审与人工评审都扫：review_source 只记录、不作为过滤条件」，过滤条件约在第 83 行）：

```sql
WHERE r.status = 'WRONG'
  AND r.reviewed_at <= now() - make_interval(mins => 2)   -- SETTLE_MINUTES，避开与评审行写入抢同一行
  AND r.reviewed_at >= now() - 72 小时                     -- DEFAULT_LOOKBACK_HOURS
  AND NOT EXISTS (本链路台账 / 失败链路台账已有单)
```

判定顺序（`ChatbiReviewJiraCreator.java`）：本链路台账有单 → 跳过；失败链路台账同 turn 有单 → 复用；Jira 按 turn 反查到单 → 只 `addComment`；都没有才新建。标题 `[Review wrong][TAG] question`；指派人 `jira.review-assignee`，空则 `longyu3`，**不回退** `jira.assignee`。

调度（`ChatbiReviewJiraScheduler.java`）：`initialDelayString = ${ssg.chatbi.review-jira.initial-delay-ms:240000}`、`fixedDelayString = ${ssg.chatbi.review-jira.fixed-delay-ms:900000}`，即启动后 4 分钟首跑、之后每 15 分钟；`ChatbiReviewJiraService.java` 常量：`DEFAULT_LOOKBACK_HOURS = 72`、`SETTLE_MINUTES = 2`、`MAX_ATTEMPTS = 3`、`MAX_PER_RUN = 50`、`DRY_RUN_PREVIEW = 20`。

三个边界：**不是即时**（最坏 15 分钟）、**只补 72 小时内**的评审（更早的历史不追）、**每轮上限 50 条**。

## 二、上线后的生产实际行为（证据）

开关在 2026-11-17 09:43 打开，调度器 4 分钟后开始扫。生产库实测：

| 项 | prod | tst |
|---|---|---|
| 台账 `ssg_chatbi_review_jira` | 450 行，全部 `mode=created`、`assignee=longyu3`、0 失败 | 0 行 |
| 时间跨度 | 09:45:15 → 11:00:26 | — |
| 最新单号 | SSGAI-1428（1424/1425/1426/1427 同一批） | — |
| 节奏 | 09 点 88 张、10 点 354 张、11 点 8 张 | — |
| WRONG 总数 | 736（AUTO 735 / HUMAN 1） | 739（AUTO 734 / HUMAN 5） |
| 72 小时内 WRONG | 455 | — |
| 未入台账 | 286（其中 72h 内仅 5 条） | 739 |

WRONG 按日分布：11-14 共 281 条、11-15 01:00 共 454 条、11-16 09:00 共 1 条。结论：**454 + 1 条落在 72h 窗口内被批量建单，剩 5 条一两轮内建完即停**；11-14 那 281 条在窗口外不会补建。因为 15 分钟一轮、每轮 50 条，10 点那一小时集中出单。

给用户的三条止血选项（都未执行，等选择）：
1. 关掉 Automation 里的 Review wrong 开关（停止后续扫描，已建单与台账保留）；
2. 把回看窗口从 72h 缩到 2h（改 `ChatbiReviewJiraService.DEFAULT_LOOKBACK_HOURS`，发版生效）；
3. 不动（只剩 5 条，一两轮自然结束）。

注意事项：这一批 450 张单的指派人全是 `longyu3`，需要评估对方是否被淹没。

## 三、Jira 设置页：Assignee 为什么单独放在 Routing tab

页面 `JiraPanel.vue`（`/setting/settings?panel=jira`）现有结构（行号基于当前版本）：

- Automation tab（428-493）：flow 开关矩阵表格，列为 Flow / Auto-create / Triggered by / **Assignee（只读文本）** / Status；只读单元格在 442-453 行 `{{ form[flow.assigneeKey] || 'Not set' }}`。另有 Inspection scope 与 Labels & content filters 两块。
- Connection tab（495-538）：auth type、base-url、username、API token（带 Validate key）。
- Ticket content tab（540-595）：标题/正文模板等。
- Routing tab（597-626）：5 个可编辑 `ASSIGNEE_FIELDS` 输入框（605-613），外加「How the Issue & Request assignee is picked」说明（619-623）。

`ASSIGNEE_FIELDS`（85-101）：`jira.usage-feedback-assignee`（Issue & Request）、`jira.dislike-assignee`（Dislike feedback）、`jira.probe-assignee`（Inspection）、`jira.failed-query-assignee`（默认 longyu3）、`jira.review-assignee`（默认 longyu3）。

分开的真实原因只是信息组织，不是技术限制：
- Automation 是「开关矩阵」视角，一行一个 flow；Routing 是「人」视角，5 个来源各一个 owner，且 Issue & Request 还有 **page focal 表**——按页面路径匹配（精确路径 → 最长前缀 → `*` 兜底行 → 最后才用上面那个 assignee），一条输入框表达不了，必须独立成块。
- 保存逻辑同一份：`form` 是同一对象，保存是全量 page-save（`WRITABLE_KEYS` 全提交），**拆 tab 不影响落库**。
- 若要合并：只改 `JiraPanel.vue`，把 Automation 表格 Assignee 列换成约 140px 输入框、Routing 只留 page focal 说明；代价是表格变宽，1440 屏大概率横向滚动。

## 四、已知小 bug（未修）

`JiraPanel.vue` 的 `save()`（302-320 行）用 303-305 行校验 **5 个 assignee 全部非空**才提交：

```ts
const missing = ASSIGNEE_FIELDS.filter(({ key }) => !String(form[key] || '').trim())
if (missing.length) {
  ElMessage.error('All three assignees are required: ' + missing.map((item) => item.label).join(', '))
  return
}
```

与实际语义冲突：提示文案还写「All three assignees」，而 `jira.failed-query-assignee` / `jira.review-assignee` 的 hint 明说「留空回退 longyu3」（后端 `ProbeSettingService.java` 中 `DEFAULT_REVIEW_ASSIGNEE = "longyu3"`、`resolveFlowAssignee` 亦有回退）。当前生产两个值都填了 longyu3，所以未暴露。可选修法：只对前 3 个必填做校验（或允许空并由后端回退），同时改文案。
