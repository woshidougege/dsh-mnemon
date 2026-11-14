---
id: "459c1dd6-22d5-433d-baa8-b9e2bc6b1ad1"
title: "ChatBI 反馈列表 buildFeedbackReview NPE 根因与修复（MR !552）"
description: "推翻「TST 镜像落后」结论：feedback-records 整页 300 的真实根因是 buildFeedbackReview 对 rv_source 空值做 toUpperCase 的 NPE（在 runOr 之外故整页崩）；含真实堆栈与行号、修复 diff、真实 mapper 探针前后对比、回归测试、!552/!1036/!1037 交付。"
status: "active"
created_at: "2026-11-14T02:02:12.349Z"
updated_at: "2026-11-14T02:02:12.349Z"
content_hash: "b9841b52fa374afd88b6267fa1d6f77d58bcebdd95d82e8160eb8d654e2ae364"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/chatbi/service/ChatbiEventReportServiceFallbackTest.java"
session_ids:
  - "mnemon-run-v1-9e7e7f93-b336-4de9-8953-f0727000332d"
memory_body_ids:
  []
---

# ChatBI 反馈列表 buildFeedbackReview NPE 根因与修复（MR !552）

承接 **8541407e-42d7-4863-afaa-bf889252fa65**（Feedback 页 300 的「镜像落后」判定 + 逐段降级改造 + 八维度 facets）。本文记录的是**推翻那份文档部署结论**之后的真实根因：不是镜像落后，是代码里一直存在的 NPE，只是被前面的 SQL 语法错误挡住了。

## 1. 反证：不是部署问题

- `POST /api/admin/chatbi/events/report/feedback-records` → HTTP 200 + `{"status":300,"msg":"INTERNAL_ERROR_PLEASE_RETRY","data":null,"success":false}`
- 同一个 jar 里的 `/admin/usage-issue-feedback/admin/feedback` → 200

**同一份产物、一个接口通一个挂 → 排除「镜像落后于某个 MR」的假设。** 这个判据比「对比 MR 序列与镜像时间」更硬，应优先使用。

## 2. 真实堆栈（本地用真实 mapper 跑完整 Service 链路拿到）

```
java.lang.NullPointerException: Cannot invoke "String.toUpperCase(java.util.Locale)" because "value" is null
  at ChatbiEventReportService.buildFeedbackReview(ChatbiEventReportService.java:422)
  at ChatbiEventReportService.feedbackRecords(ChatbiEventReportService.java:385)
```

- 方法签名：`private Map<String, Object> buildFeedbackReview(List<Map<String, Object>> rows)`（`ChatbiEventReportService.java:394`）
- 调用链：`feedbackRecords`（`:295` / `:343`）→ `runSafely`（`:112`）→ `enrichFeedbackTurnOutcome`（`:343` → `:586`）→ `decorateSessionRows`（`:1368`）

## 3. 机制（两级放大）

1. **数据形态**：`selectReviewStats` 的 source 段是 `COALESCE(NULLIF(btrim(rv.rv_source), ''), '')`。没走过人工评审的行 `rv_source` 为 NULL → 这里返回**空串**；服务层 `text()` 再把空串归一成 **null**，于是 `bySource.put(value.toUpperCase(Locale.ROOT), count)` 抛 NPE。
2. **爆炸半径**：`buildFeedbackReview(...)` 在 **`runOr` 之外**——此前那条「附加统计逐段降级」的改造没覆盖这一段，异常直接冒到 Controller，整张列表变成 `INTERNAL_ERROR_PLEASE_RETRY`。

**潜伏原因**：该 NPE 一直存在，但被 `selectReviewStats` 里 `FROM FROM` 的语法错误挡在前面（先报 SQL 错，走不到 Java）。!550 修好 SQL 后它才成为第一现场。→ **修完一层必看下一层，SQL 通了不等于请求通了。**

## 4. 修复

文件：`smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java`

- source 段判空，空值不入 `bySource`；
- reviewer 段同样 `if (value == null) continue;`，否则会产出一个没有 itcode 的**假评审人**选项。

```java
} else if ("source".equals(dim)) {
    if (value != null) { bySource.put(value.toUpperCase(Locale.ROOT), count); }
} else if ("reviewer".equals(dim)) {
    if (value == null) { continue; }
```

## 5. 验证

同一份真实 mapper 探针，修复前后：

| | 结果 |
|---|---|
| 修复前 | `NullPointerException` at `buildFeedbackReview:422` |
| 修复后 | `OK total=70`、`rows=10`，review `{correct=0, wrong=17, reviewed=17, pending=201, bySource={AUTO=17}, byReviewer=[{itcode=system:auto, name=Auto rule, count=17}]}` |

回归测试 `smartop-service/admin/src/test/java/com/lenovo/admin/chatbi/service/ChatbiEventReportServiceFallbackTest.java`：`feedbackListSurvivesWhenAuxiliaryQueriesFail`（评审统计 / 汇总 / Jira 全抛异常，列表仍返回 1 行）+ `blankReviewSourceDoesNotBreakList`，2 例全绿；`mvn` 15 模块 `BUILD SUCCESS`。

**教训（上一轮为何漏）**：上一轮单元测试只灌了空列表，没覆盖「source 为空但 count>0」这种真实数据形态；只验证了「SQL 能跑」，没验证「Java 层能跑」。以后接口 300 而 SQL 单跑正常时，直接用真实 mapper 跑完整 Service 链路取堆栈，不靠推理。

## 6. 环境 quirk（本地跑探针时踩到）

- 直接 `javac -cp <超长 classpath>` 在 Windows 上会因命令行过长失败：`An error occurred trying to start process C:\Program Files\Common Files\Oracle\Java\javapath\javac.exe ... 文件名或扩展名太长`。
- 只编译部分源码会得到 `java.lang.NoClassDefFoundError: com/lenovo/admin/chatbi/ChatbiEventDictionary`（同包但未进 classes）。用 `mvn` 全量编译（产出 1323 个 class）后再跑探针才正常。

## 7. 交付

- **!552** → `admin-release`：https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/552（sha `7f17a229e`，2 files，+69/−1，`can_be_merged` 无冲突）
- 前端配套：**!1036** → `master`、**!1037** → `develop`（筛选下拉条数透传）
- 分支：`fix/feedback-review-source-npe`
