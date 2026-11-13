---
id: "b117eb20-1364-4363-8be7-88c7bc0f5401"
title: "Admin 导出文件名改为小写下划线 + 下线 Inspection Report 导出（MR !531/!986/!987）"
description: "导出文件名规范从「菜单名-日期」改为 menu_slug_yyyyMMdd 全小写下划线（ExportFileNames 统一入口、slug 兜底、Content-Disposition header 封装、四个页面新旧名对照、复用旧 doc 9d4099c3 的 RFC 5987 结论）；Inspection Report 导出（后端 export.xlsx + ProbeReportExportService + 前端两个按钮）按下线删除且不应恢复；测试 19 通过、vue-tsc 双分支通过、MR !531/!986/!987 只开不合并。"
status: "active"
created_at: "2026-11-13T03:24:27.265Z"
updated_at: "2026-11-13T03:24:27.265Z"
content_hash: "143de3f89f699f021e6c87065c23e923ea4be71f3bb45706cda41f946b2bb5b4"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/util/ExportFileNames.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/util/ExportFileNamesTest.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/service/UsageIssueFeedbackExportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/datainquiryprobe/controller/ProbeBaselineReportController.java"
session_ids:
  - "mnemon-run-v1-e97c3e9b-9d0c-4577-a41a-56ff18b75d67"
memory_body_ids:
  []
---

# Admin 导出文件名规范改为「小写下划线」+ 下线 Inspection Report 导出（MR !531 / !986 / !987）

来源：分支 `fix-export-filename-style-admin`（worktree `.agents/worktrees/fix-export-filename-style-admin`）的完成记录。本文件记录两项独立决策：**(A) 导出文件名风格整体推翻重做**，**(B) 用户明确要求删除 Inspection Report 的导出功能**。

> 相关但已被本文件部分取代的旧记录：`9d4099c3-825a-404d-90b5-83fed32a6e4a`（MR !525，反馈导出文件名对齐菜单名 `Issue & Request-yyyyMMdd`）。该文件中的 **Content-Disposition 必须加引号 + RFC 5987** 的踩坑结论仍然成立，但那条命名风格本身已被本文件取代。

## A. 文件名风格：为什么推翻「菜单名-日期」

用户反馈（原话）：`User Logs-Data Inquiry Records-20261113.xlsx 这样真的好吗？怎么看起来Logs-Data怪怪的。而且这种名字 linux和mac不友好把`

两个问题：
1. **可读性**：菜单名首尾相接（`User Logs` + `Data Inquiry Records`）读起来变成 `Logs-Data`，看不出层级；
2. **shell / 文件系统友好性**：名字里有空格、`&`、大小写混排，在 bash/zsh 需要转义，在 macOS 默认大小写不敏感文件系统上还容易发生匹配歧义。

新规范（唯一入口 `smartop-service/admin/src/main/java/com/lenovo/admin/util/ExportFileNames.java`）：

- 形态：`menu_slug_yyyyMMdd.xlsx`，**以菜单名开头**，一眼看出是哪个页面导的；
- 字符集只允许 `[a-z0-9_]` + `.xlsx`：无空格、无 `&`、全小写；
- 长度 ≤ 64；
- `slug()` 兜底把页签名里的空格/引号/斜杠一律折成下划线（`How's this?` → `how_s_this`）；
- `header(fileName)` 统一产出 Content-Disposition（继承 !525 的引号 + RFC 5987 写法，不再各处手拼）。

| 页面 / 页签 | 新文件名 | 旧文件名 |
|---|---|---|
| User Logs · Data Inquiry Records | `user_logs_data_inquiry_records_20261113.xlsx` | `chatbi-query-20261113.xlsx` |
| User Logs · Raw Events | `user_logs_raw_events_20261113.xlsx` | `chatbi-raw-20261113.xlsx` |
| User Logs · Feedback | `user_logs_feedback_20261113.xlsx` | `chatbi-feedback-20261113.xlsx` |
| Issue & Request | `issue_and_request_20261113.xlsx` | `Issue & Request-20261113.xlsx` |

注意：Excel **工作表标签**仍是 `Issue & Request`——sheet tab 不落盘、不受 shell 影响，所以不改。

改动落地在 `smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportService.java`（`FILE_PREFIX` 常量与 `FILE_DAY` 日期格式化被 `ExportFileNames` 调用取代）与 `smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/service/UsageIssueFeedbackExportService.java`；测试断言分布在 `smartop-service/admin/src/test/java/com/lenovo/admin/util/ExportFileNamesTest.java`、`smartop-service/admin/src/test/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportServiceTest.java`、`smartop-service/admin/src/test/java/com/lenovo/admin/usagefeedback/controller/UsageIssueFeedbackControllerExportTest.java`（原来断言的 `Issue & Request-\d{8}\.xlsx` 正则同步改写）。

## B. Inspection Report 导出整体下线

用户一句明确指令：`Inspection Report 的导出功能删掉哈 这个不需要导出`。不是改名，是删除，**不要重新加回**。

- 后端删除 `GET /data-inquiry-probe/baseline/export.xlsx`（`smartop-service/admin/src/main/java/com/lenovo/admin/datainquiryprobe/controller/ProbeBaselineReportController.java`），连带删掉 `smartop-service/admin/src/main/java/com/lenovo/admin/probe/service/ProbeReportExportService.java` 及其单测 `smartop-service/admin/src/test/java/com/lenovo/admin/probe/service/ProbeReportExportServiceTest.java`；
- `ExportFileNames` 里不再保留 `inspectionReport(...)` 命名规则；
- 前端删掉 `downloadBaselineExport` / `baselineExportUrl`（`views/Setting/reports/inspection-report/api.ts`），以及 `views/Setting/reports/inspection-report/components/views/BaselineView.vue` 里的 `Export Excel`、`Export this batch` 两个按钮与 `exportRuns()` / `exportBatch()` 两个函数（净 -60 行）。
- 该 View 的 rest（批次、题池、SQL 预览等）不受影响。

## 验证与 MR

- 后端：`-Dtest=ExportFileNamesTest,UsageFeedbackExportServiceTest,UsageIssueFeedbackControllerExportTest,ChatbiEventExportServiceStreamTest,ChatbiEventExportServiceEstimateTest` → **Tests run: 19, Failures: 0, Errors: 0**，BUILD SUCCESS。
- 前端：master / develop 两侧 `vue-tsc --build` 均 exit=0；`exportRuns` / `exportBatch` / `downloadBaselineExport` 全仓无残留引用。
- MR（按既有约定**只开 MR、不合并**）：
  - `ssg-sales-ops` **!531** `fix-export-filename-style-admin` → `admin-release`，10 个文件，HEAD `d0c010bb7`，`can_be_merged`；
  - `ssg_sales_ops` **!986** → `master`，**!987** → `develop`，各 2 个文件（仅前端删除）。
- 提交信息：`d0c010bb7 refactor(admin): 导出文件名改全小写下划线，Linux/macOS 友好`、`42612153 / 66242112 feat(web): 去掉 Inspection Report 的导出入口`。

## 可复用结论

- 以后新增 admin 导出，一律走 `ExportFileNames`，不要在各 service/controller 里手拼文件名与 Content-Disposition；新名字必须是 `[a-z0-9_]` 全小写、以菜单名开头、带 `yyyyMMdd`。
- 「菜单名开头」的可读性目标不自动成立：多个词拼接后可能连读歧义（`Logs-Data`），所以用分隔符 `_` 并保持全小写，避免依赖大小写边界。
- 中文/带符号的标题只是 UI 文案，不能直接当文件名。
