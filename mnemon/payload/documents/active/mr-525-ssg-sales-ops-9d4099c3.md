---
id: "9d4099c3-825a-404d-90b5-83fed32a6e4a"
title: "反馈导出文件名对齐菜单 + 状态列配色（MR !525，ssg-sales-ops）"
description: "MR !525 实现交接：导出文件名改 Issue & Request-yyyyMMdd 且 Sheet 改名；Content-Disposition 必须加引号并补 RFC 5987 的原因；状态列复用 JiraStatusTag 色板与具体色值；去掉点踩整行红的取舍（115 红盖住 82 条 FAILED）；5 个测试断言、LibreOffice 缺失的验证缺口；前端兜底名与 TST jira 开关两个未决项。"
status: "active"
created_at: "2026-11-12T11:54:46.012Z"
updated_at: "2026-11-12T11:54:46.012Z"
content_hash: "b98ab84bf724a3092e5f0bbdf9ded818c9b9e4a4b10feb50b1bd0448d707bd54"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/controller/UsageIssueFeedbackController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiPrettyExcelWriter.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportServiceTest.java"
session_ids:
  - "mnemon-run-v1-243c4d6c-0022-4135-9beb-f070cbde1a8a"
memory_body_ids:
  []
---

# 反馈导出文件名对齐菜单 + 状态列配色（MR !525，ssg-sales-ops）

来源：分支 `fix-export-filename-style-admin` 的完成记录。用户诉求是「导出文件名跟页面菜单一致、Excel 里状态列配色跟页面标签一致」。提交 `2655d2853`「fix(admin): 反馈导出文件名对齐菜单并补日期，状态列改用页面标签配色」，已 push（origin 同名分支），MR **!525** → `admin-release`，创建后回查 `state=opened`、`merge_status=can_be_merged`、`has_conflicts=False`、4 个文件 / 232 插入 / 18 删除。**按既有约定只开 MR，未合并**（合并与部署由用户负责）。

## 1. 文件名与 Sheet 名

- 导出文件名：`feedback-20261112.xlsx` → **`Issue & Request-20261112.xlsx`**（菜单/页面标题同名 + `yyyyMMdd` 日期）。
- Excel 内 Sheet 名：`Feedback` → `Issue & Request`。
- 实现点在 `smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/controller/UsageIssueFeedbackController.java` 导出入口：xlsx 用 `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet`，csv 用 `text/csv;charset=UTF-8`。

### 必须带引号 + RFC 5987 的原因（踩坑）

旧响应头是裸拼 `attachment;filename=` + 文件名。新名字含**空格**和 **&**，未加引号的 Content-Disposition 会被浏览器在空格处截断，可能只存成 `Issue`。改为：

```java
response.setHeader(HttpHeaders.CONTENT_DISPOSITION,
        "attachment;filename=\"" + exportFileName + "\";filename*=UTF-8''"
                + encodeFileName(exportFileName));
```

`encodeFileName`（同文件 `UsageIssueFeedbackController.java:248-255`）用 `URLEncoder.encode(name, UTF-8)` 再把 `+` 换成 `%20`，异常时回落 `attachment`。附件代理接口早就用同一套 RFC 5987 写法（该文件 202/236 行，注释 247 行）。

前端解析要能吃两种形式，见 `C:\Users\63588\lenovo\ssg_sales_ops\src\services\usageIssueFeedback.ts:786-799`：先 `/filename\*=UTF-8''([^;]+)/i` → `decodeURIComponent`，再退 `/filename="?([^";]+)"?/i`。只补一种，下载名就静默回落到兜底名。

## 2. 状态列配色（与页面标签同一板）

色值来源：前端 `JiraStatusTag.vue` 的 palette；后端 tone 由 `JiraIssueViewDecorator.tone()` 产出，所以两边颜色天然一致。改动落在 `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiPrettyExcelWriter.java`（+73 行）。

| 单元格 | 底色 / 字色 |
|---|---|
| Status = 已完成(CREATED) | `#ECFDF5` / `#047857` |
| Status = 建单中(PENDING) | `#FFFBEB` / `#B45309` |
| Status = 建单失败(FAILED) | `#FEF2F2` / `#B91C1C` |
| Jira status 已完成 | `#ECFDF5` |
| Jira status 进行中 | `#FFFBEB` |
| Jira status 未开始(New) | `#EFF6FF` |
| Type = 提交问题 | `#EFF3FF` / `#1D4ED8` |
| Type = 点踩 | `#F5F3FF` / `#6D28D9` |

其它列保留原斑马纹，避免整张表花掉；另外**冻结前两列**（Time / User），长表横滚时仍知道是谁、什么时候提的。

### 取舍：去掉「点踩整行铺红」

原实现给点踩记录整行染红。测试数据一对比就露馅：115 条点踩整片红，把真正建单失败的 82 条盖住了。现在只保留 FAILED / 报错行红。若用户仍希望点踩行有底色，备选是浅灰或极浅紫。

## 3. 验证

`mvn -o -T 4 -pl smartop-service/admin -am test -Dtest=UsageFeedbackExportServiceTest` → `Tests run: 5, Failures: 0, Errors: 0` / BUILD SUCCESS（admin 是 15 模块反应的最后一环）。

新增用例（`smartop-service/admin/src/test/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportServiceTest.java`，+76 行）直接断言：文件名匹配 `Issue & Request-\d{8}\.xlsx`、Sheet 名为 `Issue & Request`、Type / Status / Jira status 三列的实际填充色（`F5F3FF` / `EFF3FF` / `EFF6FF` / `FFFBEB` / `FEF2F2`），普通列仍是斑马纹。

**未验证的缺口**：本机没装 LibreOffice（`C:\Program Files\LibreOffice\program\soffice.exe` 与 x86 路径均不存在），无法把 xlsx 渲成图看效果，只能等部署 TST 后实际下一份核对。

## 4. 未决事项

1. 前端兜底文件名仍是 `feedback-${exportTimestamp()}.${format}`（`C:\Users\63588\lenovo\ssg_sales_ops\src\services\usageIssueFeedback.ts:839`），只在响应头解析不出来时才用。是否改成 `Issue & Request-…` 待用户定；要改需单独走前端 master / develop 双 MR。
2. TST 的 `jira.flow.usage-feedback.enabled` / `jira.flow.dislike.enabled` 仍为 false，新提交不会自动建单；是否打开待用户决定（同一未决项在 TST 数据同步记录 `4568440b-b6c8-47bd-a416-bce10601a830` 里也挂着）。

## 5. 本仓库相关约定

同一需求链路上反复出现「agent 只创建 MR，不执行合并」：合并与部署由用户负责。文件名/表头类改动要同时顾到后端响应头与前端 `exportFileNameFromResponse` 两侧，缺一侧会静默回落到兜底名。
