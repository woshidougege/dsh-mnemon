---
id: "e4ce57a0-5fd7-4a73-ad84-5aa57bf3ce0f"
title: "ChatBI 问数自动建 Jira：双时机事件驱动 + 每小时兜底的落地方案（2026-11-17）"
description: "用户要求的两处事件驱动（人工评审 WRONG 立即建单、回答结束触发自动评审）+ 每小时兜底的落地方案：数据可行性核对、四个落地点与行号、五个坑（afterCommit/不走 scan/独立线程池/审计延迟/人工优先）、两个待拍板参数，附同批 Jira 设置页 Assignee 内联与校验改动的 commit 记录。"
status: "active"
created_at: "2026-11-17T03:22:41.001Z"
updated_at: "2026-11-17T03:22:41.001Z"
content_hash: "b09ba9d39fc11d34097bd49a0b81238738b6e0881e8533155060876e4dc51015"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiTrackingService.java"
session_ids:
  - "mnemon-run-v1-ccbc6348-58e4-49f5-967a-2df2ae9b7aa7"
memory_body_ids:
  []
---

# ChatBI 问数自动建 Jira：双时机事件驱动 + 每小时兜底（落地方案）

状态：2026-11-17 与用户对齐方向，**尚未实施**；两个参数问题待用户拍板。关联文档 `750e10a9-15ec-4ab0-9295-1f45e6d90bfd`（自动评审规则 V1.1.121）与 `f60c9e73-60f7-4ded-875a-34001eb9d258`（生产运行时核对）。

## 1. 用户要求的目标形态

用户原话（2026-11-17）："他有2个时机。1、用户手动评审，评审到了wrong 就得立马创建jira 2、用户问问题，回答结束后，要立刻调用自动评审review. 如果按照规则 评审到了wrong ,就要立马创建jira"；"然后每小时有个兜底的定时任务 对极少数失败的去补漏"。

→ 两处事件驱动（人工评审保存后、埋点入库后触发自动评审）+ 每小时兜底，取代现在纯轮询的单层结构。

## 2. 数据可行性核对（生产近 7 天 `query_session`）

- 595 行 / 595 轮，一轮一条（重复轮次 `multi=0`）。
- 595/595 带 `status`；`answer_status` 属性 0 条 —— 结束信号就是 `status`。
- 分布：success 560 / failed 33 / needs_input 2；峰值 20 条/小时，单条评审 SQL 约 200–700ms，扛得住。
- 埋点本身幂等：`mapper.insertIgnore` 按 event_id 去重，只有 `rows == 1` 才走后续逻辑，所以"入库成功即触发"不会重复评审。

## 3. 现状 vs 目标

| 环节 | 现在 | 目标 |
|---|---|---|
| 人工判 Wrong → 建单 | 15 分钟轮询 | 保存后立即建 |
| 回答结束 → 自动评审 | 不存在（只有 Admin 页手动 Run backfill） | 新做，事件驱动 |
| 自动评审判 Wrong → 建单 | 靠轮询 | 新做，立即建 |
| 兜底 | 15 分钟一轮、每轮 50 条 | 每小时 |

## 4. 四个落地点

1. **手动评审立即建单**：`ChatbiQuestionReviewService.save()`（`smartop-service/admin/.../service/ChatbiQuestionReviewService.java:86-140`）upsert 成功且 `status == WRONG` 时，异步调 `ChatbiReviewJiraCreator.createOne(...)`。
2. **回答结束触发自动评审**：`ChatbiTrackingService.collect()` 第 199-209 行 —— `insertIgnore == 1` 且 `eventName == "query_session"` 且 `status` 非空时，异步调新的 `autoReviewService.reviewTurn(sessionId, queryId)`。
3. **自动评审单条入口**：`ChatbiQuestionAutoReviewService` 目前只有 `runBackfill(from, to)`（按时间窗扫），需加单 turn 版本，复用 `ChatbiQuestionAutoReviewMapper` 的判定条件。
4. **兜底降频**：`ChatbiReviewJiraScheduler` 的 `fixedDelay` 由 `900000`（15 分钟）改为 `3600000`（1 小时），scan 逻辑不动，继续管失败重试与漏网。

## 5. 五个必须提前定的坑（含理由）

1. **事务提交后再触发**：`save()` 里 upsert 之后立刻异步调用时，异步线程可能读不到未提交行 —— 用 `afterCommit` 回调触发（自动评审同理，埋点 insert 之后）。
2. **建单不要走 scan**：`scan()` 有 `reviewed_at <= now() - 2 分钟` 的 settle，立即建单会白等 2 分钟 —— 直接调 `ChatbiReviewJiraCreator.createOne`，它自带 `pg_advisory_xact_lock` + 台账幂等，不会重复建。
3. **线程池不能复用**：现有 `chatbiTaskExecutor` 是 core=1 / max=2 / queue=1000 / `DiscardPolicy`；Jira 建单是 HTTP 慢调用（1–3 秒/条），450 条会把提问聚类堵死还静默丢弃 —— 单开 `chatbi-jira-executor`（2–4 线程、有界队列）。
4. **自动评审依赖的审计数据可能还没落库**：判 Wrong 要读 `tool_audit_logs`（LDUP 权限）、HC、`route_report`，可能比埋点晚几十秒；立即评会"没看到权限失败"而误判 Wrong —— 明显缺数据（如 failed 但 audit 无记录）就不下结论，留给兜底。
5. **人工优先**：自动评审 SQL 已有 `NOT EXISTS (已有评审行)`，单条版必须保留，否则人工判完又被自动改掉。

## 6. 待用户拍板

- **无权限要不要自动判 N/A**：`ChatbiQuestionAutoReviewMapper` 第 92 行写死 `AND NOT LUDP_PERMISSION_SQL`，即自动评审永不判 N/A，无权限一律留人工。
- **兜底失败的处置**：`ChatbiReviewJiraService` 现在是失败重试 3 次、间隔 30 分钟、每轮上限 50 条；改每小时跑之后这两个参数是否跟着调。

## 7. 同批已提交的 Jira 设置页改动（未 push、未提 MR）

分支 `dev/renxp4`（worktree `C:\\Users\\63588\\lenovo\\ssg_sales_ops-wt-nareview`），文件 `src/views/Setting/settings/panels/JiraPanel.vue`：

- `689d1c71` feat：Assignee 列由只读文本改为内联 `el-input`（`size="small"`，宽 200px），五行对应 `jira.probe-assignee` / `jira.dislike-assignee` / `jira.usage-feedback-assignee` / `jira.failed-query-assignee` / `jira.review-assignee`，后两个 placeholder `Defaults to longyu3`、其余 `Jira username`；页签 4→3（Automation / Connection / Ticket content），Routing 删除，page focal 说明（精确路径 → 最长前缀 → `*` 兜底）移到流程表下，标题 `How the Issue & Request assignee is picked`，保留 `Open the Page focal table` 入口。
- `a1afa0f1` fix：`FieldDef` 新增 `optional?: boolean`；`save()` 改为 `!field.optional && !String(form[field.key] || '').trim()`，只拦三个必填（Issue & Request / Dislike / Inspection），文案 `These assignees are required: <labels>` —— 与"Failed query / Review wrong 留空回退 longyu3"的口径一致。
- 动因（用户原话）："在Automation 能改 Assignee就行了 你设计的太复杂了 用户反而不会用"。
- 验证：`npm run type-check` 两次通过；本地 `http://localhost:80/setting/settings?panel=jira` 热更新可见；`vite.config.ts` 代理在同工作树 reset 时被冲掉后已补回（未提交），`/api/admin/*` 实测 200。
- 交付纪律：每个功能一次提交，MR 等用户命令再提。
