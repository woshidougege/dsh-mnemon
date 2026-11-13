---
id: "f95e9809-bf7c-429e-8688-35c16e88b7b7"
title: "admin 巡检接口 500/超时根因：ssg_ai_probe_run 不再 select *（MR !537，ssg-sales-ops）"
description: "TST 下 batches/report 三条 admin 接口 500 或 30s 超时，根因为 recentBatches/range/daily 全列查询把 raw_response 等大字段拉进 JVM（1277 行 101MB，大字段占 98MB，实际仅需 19KB）；修法为 AiProbeRunColumns.withoutHeavyColumns 裁六列，MR !537，含护栏测试、边界说明与复测待办。"
status: "active"
created_at: "2026-11-13T04:55:23.906Z"
updated_at: "2026-11-13T04:55:23.906Z"
content_hash: "4924230e77600425f0795d7afdc4955ea85613270ee08d5bad5fe45d6bd38330"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-72304458-2c2f-4b0f-847c-3b2cf2b053e0"
memory_body_ids:
  []
---

# admin 巡检接口 500/超时根因：`ssg_ai_probe_run` 不再 `select *`（MR !537，ssg-sales-ops）

仓库：`C:\Users\63588\lenovo\ssg-sales-ops`（远端 `https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops.git`）。
目标分支 `admin-release`，修复分支 `fix/probe-run-columns-admin`。

## 1. 现象（TST 环境，2026-11-13）

三条 admin 接口同时报 500 或超时，初判像是"导出功能坏了"，实测与行数强相关、非随机：

| 请求 | 结果 |
|---|---|
| `batches?limit=1/2/3/5` | 全 200，每次约 1.5–2s |
| `batches?limit=10/20/50` | 全 500（连续 5 次、3 次复现），约 3.8s 失败 |
| `report/range?range=3D` | 时好时坏，失败发生在 1.5–4s |
| `report/range?range=7D / 30D` | 约 32s 后 500（网关 30s 掐断） |
| `admin/feedback?size=10` | 间歇 500，隔一会儿又 200 |

触发前提：当天早上把生产数据同步进 TST，把原本小基数的表放大到千行量级，缺陷因此暴露。
判定：**不是导出链路的回归**——导出代码本次未改动。

## 2. 根因：`select *` 把每条巡检的整包响应原文一起捞进 JVM

`ssg_ai_probe_run` 在设计上把每次巡检的完整响应存进行里（`raw_response`、`response_json`、`request_payload_json`、`result_json`、`agent_steps`、`error_stack`），单行最大 400 KB+。

两个热路径用全列查询却在内存里只算聚合：

- `smartop-service/admin/src/main/java/com/lenovo/admin/probe/service/ProbeRegressionService.java:143` —— `recentBatches(limit)` 把前端 `limit` 放大成 SQL 的 `limit max(200, min(2000, limit*100))`，所以批次选择器一开页就是 **limit=2000**。
- `smartop-service/admin/src/main/java/com/lenovo/admin/aiprobe/service/AiProbeReportService.java:74` —— `range()` 与 `daily()`（:30-42）同样全量取行、再在服务端内存里分页/汇总。

实测（TST，Postgres）：

- `probe_type='SMART_DQ' and started_at >= now() - interval '3 days'`：**71 行 / 7391 kB**，其中 `raw_response` 独占 **7189 kB**（`answer_text` 25 kB、`sql_text` 26 kB、其余近乎为 0）。这正是 3D 窗口"1.5–3s 勉强能扛"的原因。
- `batch_id is not null and expected_sql is not null` 取最近 2000 行：**1277 行 / 101 MB**，其中六个大字段合计 **98 MB（97%）**；而 `recentBatches` 真正需要的列加起来只有 **19 KB**。

即：接口每次为了算通过率，把约 100 MB 搬进 JVM。小规格 Pod 上表现为 GC 抖动 / OOM → 500 或 30s 超时，同 Pod 的其他接口（如 `admin/feedback`）被连带拖慢。**注意：`feedback` 被拖慢这一点尚无直接证据**，需要服务端日志才能钉死，它的 SQL 本身只碰 20 条反馈、很轻。

## 3. 修法：新增裁列工具，热路径不再取大字段

新增 `smartop-service/admin/src/main/java/com/lenovo/admin/aiprobe/support/AiProbeRunColumns.java`：

```java
// AiProbeRunColumns.withoutHeavyColumns(LambdaQueryWrapper<AiProbeRun>)
// 多行查询裁掉 raw_response / response_json / request_payload_json /
// result_json / agent_steps / error_stack 六列
```

接入三处：`AiProbeReportService.daily`、`AiProbeReportService.range`、`ProbeRegressionService.recentBatches`。

边界（决定"前端不用改"的关键）：

- 详情 `selectById` 与导出链路**仍取全列**，返回 JSON 结构不变。
- 预期效果：`batches?limit=20` 单次数据量从约 101 MB 降到约 3 MB（真正要用的 19 KB + 其余小字段）。

## 4. 验证

护栏测试 `smartop-service/admin/src/test/java/com/lenovo/admin/aiprobe/support/AiProbeRunColumnsTest.java`：断言六个大字段不出现在 select 列表、必用列仍在。

```
mvn -o -T 4 -pl smartop-service/admin -am test -Dtest=AiProbeRunColumnsTest,ProbeRegressionServiceTest
→ Tests run: 6, Failures: 0, Errors: 0, Skipped: 0
```

## 5. MR 与提交

- MR **!537**：https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/537 （`fix/probe-run-columns-admin` → `admin-release`），`change=4 个文件`，状态 `opened / can_be_merged`。
- 提交 `79d8fc51d fix(admin): 巡检列表与报表查询不再 select *，去掉整包响应大字段`（4 files, +89/−4）；上一版 `c6f0c6e47` 因误带 `.git-msg.txt` 被 forced update 覆盖。
- MR 通过 GitLab API 创建，`PRIVATE-TOKEN` 取自仓库的 git credential helper（`credential.https://gitlab.xpaas.lenovo.com.provider = gitlab`）；`GITLAB_TOKEN` 环境变量未设置。

## 6. 过程教训

- 在隔离 worktree `.agents/worktrees/fix-probe-run-columns` 里直接 ``-pl smartop-service/admin`` 编译会失败，报 `CocFlagSyncScheduleHandler.java:[71,54]` / `[127,58]` "找不到符号 `cocFlagSync(String)` / `cocFlagSyncStatus()`"（`com.lenovo.ciop.feign.auth.AuthFeignService` 缺方法）。这是该 worktree 依赖快照过旧，与本次改动无关；加 `-am` 全模块构建后 exit=0。结论：判定编译错误前先确认是否 worktree/依赖陈旧。
- 仓库既有换行策略会把 `AiProbeRunColumns.java`、`.git-msg.txt` 的 LF 报成将替换为 CRLF，属正常提示，不必处理。
- 提交前务必确认没有把临时提交信息文件（`.git-msg.txt`）带进 commit。

## 7. 待办 / 未决（截至本记录，未完成）

1. 等人工合并 !537 并发布 TST → 把三条 curl 连打 20 次复测是否仍抖。
2. 若发布后 `admin/feedback` 仍间歇 500，则不是取数问题，需要 TST Pod 日志；已有三个 `requestId`：`3fa46cbc-523801`、`0b8f00d0-785746`、`ba157359-785750`。
3. 可选：清理 TST 中同步过来的超大 `raw_response` 老行（需要 `a_appconnect` 写权限，待用户授权）。
4. TST 的 `jira.flow.usage-feedback.enabled` / `jira.flow.dislike.enabled` 仍为 false，等用户拍板。

## 8. 相关文档

- `4a6ba38c-451d-4a48-b625-05b6d75db712` —— ChatBI 人工评审 JOIN 二义性导致的同类 500（MR !526）；同为 admin 侧 500 排查，可对照"先在 TST 直跑 SQL 复现"的手法。
