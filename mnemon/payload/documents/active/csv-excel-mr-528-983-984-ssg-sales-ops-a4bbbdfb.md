---
id: "a4bbbdfb-a8cf-4aca-987c-0e21370df51f"
title: "三个导出取消 CSV、统一只产 Excel（MR !528 / !983 / !984，ssg-sales-ops）"
description: "MR 交接：CSV 从前端弹窗到后端服务完整下线，三个导出只产 xlsx；超单 sheet 行数改自动分片；format 参数保留但忽略以兼容旧页面；17 个测试与两侧 vue-tsc 验证；文件名统一与 TST jira 开关两个未决项；关联 MR !525 文档。"
status: "active"
created_at: "2026-11-13T02:36:41.083Z"
updated_at: "2026-11-13T02:36:41.083Z"
content_hash: "c194dd2b9a789cc97a859bf1c6933cbda60f90548dab32f8cdcbd9dbcd3718a9"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/controller/ChatbiEventReportController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/controller/UsageIssueFeedbackController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportService.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/chatbi/service/ChatbiEventExportServiceEstimateTest.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/usagefeedback/controller/UsageIssueFeedbackControllerExportTest.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportServiceTest.java"
session_ids:
  - "mnemon-run-v1-ea6748ba-5e40-4c3f-b1eb-273e41413bc2"
memory_body_ids:
  []
---

# 三个导出取消 CSV、统一只产 Excel（MR !528 / !983 / !984）

来源：分支 `fix-export-filename-style-admin`（worktree `C:\Users\63588\lenovo\ssg-sales-ops\.agents\worktrees\fix-export-filename-style-admin`）的完成记录。

用户诉求：三个导出（User Logs / Issue & Request 反馈 / Inspection Report 巡检报告）都只出 Excel，CSV 选项从前端弹窗和后端一起下线。

**只开 MR、不合并**（合并与部署由用户负责），三个 MR 交付时均为 `opened` 且 `merge_status=can_be_merged`、无冲突。

## 1. 后端：ssg-sales-ops → `admin-release`，MR !528

提交：
- `095b026a9` 「fix(admin): 三个导出取消 CSV，统一只产出 Excel」，7 files，+67 / -256（首个提交误把临时文件 `.git-msg-xlsx.txt` 一起 add，amend 后清掉，文件数 8→7）
- `6962ae6ea` 「docs(admin): 导出注释同步 CSV 下线」，2 files，+2 / -2

改动点：
- `smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportService.java`：删掉 `writeCsv`、`csvRow`、`csvCell` 三个方法；删掉 `XLSX_MAX_ROWS` 限行与 `xlsxAllowed()` 判定；`ExportPlan` 不再有 `xlsx`、`downgraded` 字段；文件名固定为 `Issue & Request-<yyyyMMdd>.xlsx`。
- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java`：删掉 CSV 分支，以及超行拒导文案 `The detail view has N rows, over the Excel sheet limit`；明细超出单 sheet 上限改由 `ChatbiPrettyExcelWriter` 的 `SheetPlan` 自动分片（sheet 名 `Sheet 1/2 …`）。
- `smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/controller/UsageIssueFeedbackController.java` 与 `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/controller/ChatbiEventReportController.java`：Content-Type 固定为 xlsx（`application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`）；`/export/count` 响应不再返回 `xlsxLimit`、`xlsxAllowed`。
- **兼容取舍**：请求参数 `format` 保留但忽略——旧页面或缓存页面即便传 `format=csv` 也照样拿到 xlsx，不会 400。这是「不破坏已发布前端」的代价，后续可择机彻底删除该参数。

验证（admin 模块，离线）：
`-Dtest=UsageFeedbackExportServiceTest,UsageIssueFeedbackControllerExportTest,ChatbiEventExportServiceEstimateTest,ChatbiEventExportServiceStreamTest,ProbeReportExportServiceTest` → **Tests run: 17, Failures: 0, Errors: 0**
相关测试文件：`smartop-service/admin/src/test/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportServiceTest.java`、`smartop-service/admin/src/test/java/com/lenovo/admin/usagefeedback/controller/UsageIssueFeedbackControllerExportTest.java`、`smartop-service/admin/src/test/java/com/lenovo/admin/chatbi/service/ChatbiEventExportServiceEstimateTest.java`（后两个 + `ProbeReportExportServiceTest` 在 MR 的 8 个改动文件里）。

## 2. 前端：ssg_sales_ops（`C:\Users\63588\lenovo\ssg_sales_ops`）两个分支、两个 MR

同一改动分别落到 master 与 develop（各自单独提交、单独 MR，未跨分支合并）：

| 分支 | 提交 | MR | 目标 |
|---|---|---|---|
| `fix/export-xlsx-only-master` | `57a0f1c9` | !983 | master |
| `fix/export-xlsx-only-develop` | `ba99b4d0` | !984 | develop |

两者均为 5 files，+17 / -125，涉及：
- `src/views/Setting/reports/components/ReportExportDialog.vue`：删 Format 卡片、CSV 选项、「超上限改 CSV」提示；行数只显示 `Estimated rows N`，不再与 1,048,576 对比。
- `src/views/Setting/reports/components/reportExport.ts`：类型收敛为 `ReportExportFormat = 'xlsx'`，去掉 `xlsxLimit`、`xlsxAllowed`。
- `src/services/usageIssueFeedback.ts`：`UsageIssueFeedbackExportCount` 不再有 `xlsxLimit` / `xlsxAllowed`；`downloadUsageIssueFeedbackExport` 仍走 fetch + Authorization + blob + `a[download]`。
- `src/views/Setting/reports/user-logs/components/modals/ExportModal.vue` 与 `src/views/Setting/reports/usage-issue-feedback/UsageIssueFeedbackExportDialog.vue`：不再传 `formats`、`xlsx-limit`、`default-format`。

验证：master、develop 两侧 `vue-tsc --build` 均 exit 0。

## 3. MR 交付表（均为 opened，等用户点合并）

| 仓库 | MR | 目标分支 | 链接 |
|---|---|---|---|
| ssg-sales-ops | !528 | admin-release | https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/528 |
| ssg_sales_ops | !983 | master | https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/983 |
| ssg_sales_ops | !984 | develop | https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/984 |

## 4. 未决项（留给用户拍板）

1. **文件名是否统一成菜单名**：Issue & Request 已是 `Issue & Request-<yyyyMMdd>.xlsx`；User Logs 仍是 `chatbi-<tab>-<日期>`；Inspection Report 是 `smart-data-query-probe-3d.xlsx`（无日期）。要不要一起对齐。
2. **TST 开关**：`jira.flow.usage-feedback.enabled`、`jira.flow.dislike.enabled` 仍为 false，TST 上新反馈不会自动建 Jira 单。

## 5. 关联与教训

- 文件名/状态列配色的前序交接见文档 `9d4099c3-825a-404d-90b5-83fed32a6e4a`（MR !525）。那份文档里 CSV 还是活的能力（Controller 里 xlsx/csv 两套 Content-Type），本次改动把它全部下线，读旧文档时以本文为准。
- Content-Disposition 必须带引号 + `filename*=UTF-8''` 的结论仍适用于 xlsx 分支，不要因为「只剩 xlsx」就退化回裸拼。
- 教训：用文件写 commit message（`git commit -F .git-msg-xlsx.txt`）时别把该临时文件放在仓库根并被 `git add -A` 扫进去；本次首个提交就多带了一个 `.git-msg-xlsx.txt`，靠 amend 修正。
