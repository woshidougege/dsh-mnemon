---
id: "4dbebf1a-8fc7-4f8d-ad3c-e0eb923823f4"
title: "导出状态列共享色板 SemanticTones 与两处导出补齐配色（MR !540，ssg-sales-ops）"
description: "MR !540 交接：把「值→标签色」抽成 SemanticTones 并由 Confluence… 两处导出（User Logs / Issue & Request）共用；完整 tone 规则表、按表头名取列下标、去掉点踩整行红的理由；11 个用例读回 XSSFColor 断言；基线 ChatbiPrettyExcelWriterTest 4 个 NUMERIC/STRING 报错的已知失败；GitLab 项目 id 与 office_render 局部渲染两个操作提示。"
status: "active"
created_at: "2026-11-13T08:31:49.785Z"
updated_at: "2026-11-13T08:31:49.785Z"
content_hash: "419fe519ff3c84092fa458937a344211fd092a525bb5c019c9c781487a8039b7"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/support/SemanticTones.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportService.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/chatbi/service/ChatbiEventExportServiceStreamTest.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportServiceTest.java"
session_ids:
  - "mnemon-run-v1-5667ec01-0cda-4ff4-8eb3-15c6fd35154e"
memory_body_ids:
  []
---

# 导出状态列共享色板 SemanticTones（MR !540，ssg-sales-ops）

承接文档 `9d4099c3-825a-404d-90b5-83fed32a6e4a`（MR !525：文件名/Sheet 改名 + 首版状态列配色）。那一轮只给 Usage Feedback 导出的 Type / Status / Jira status 上色；本轮把「值 → 标签色」抽成共享组件，并补齐两条导出链路上其余状态列。第 3 处「Inspection Report 导出」在上一轮已按用户要求删除，本轮不涉及。

## 1. 新增共享组件

`smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/support/SemanticTones.java`（188 行，新文件）——把「字段取值 → tone」映射集中一处，两个导出服务共用，避免两套色板漂移。

接入点：
- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java`（+139/−25）：Feedback、Feedback details、Sessions、Failed sessions、Processing steps 五个 Sheet 的着色；`reviewStatusCell` / `reviewTagCell` 在 `ChatbiEventExportService.java:409` / `421`，调用点 `393` / `394`，tone 判定 `535` / `539`。
- `smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportService.java`（+17）：Issue & Request 的 Review status / Attribution tag 也改用同一组件。

## 2. 规则表（绿 / 琥珀 / 蓝 / 红 / 灰紫）

| 列 | 绿 | 琥珀 | 蓝 | 红 | 灰 / 紫 |
|---|---|---|---|---|---|
| Jira status | Done | In Progress、Testing、Creating | New、To Do | Creation failed、Cancelled | No ticket yet |
| Final result | Success | — | — | Failed | — |
| Failure category | — | 未最终失败 | — | 最终失败 | Recovered（紫） |
| KM status | Synced | Pending sync | — | Sync failed | Deleted |
| Type / Source | Helpful、User | — | Admin 代操作 | Not helpful | — |
| Review status | Correct | Pending | — | Wrong | — |
| Attribution tag | — | — | — | — | 判错时紫 |

## 3. 关键取舍

- **反馈明细不再整行铺红**：旧实现给点踩行整行铺 `FEF1F2`，红色语义标签压上去几乎看不出区别。现在只有**真正带错误信息**（如 `jiraError` 非空）的行铺红，其余交给彩色标签——与 Issue & Request 早就采用的规则统一（同一取舍在 `9d4099c3` 里已针对 115 条点踩盖住 82 条 FAILED 记录论证过）。
- **列下标按表头名取，不写死数字**：以后调整列顺序不会串色；断言侧同样用 `columnIndex(header, "Type")` 之类按表头定位。
- 非状态列仍走斑马纹（普通行 `F6F7FB`），避免整张表花掉。

## 4. 验证

- 新增用例 2 个，做法是真实写出 xlsx 后读回 `XSSFWorkbook` + `XSSFCellStyle` 的 `XSSFColor` RGB 逐格断言：Feedback 页签 11 个格、Sessions 的 Final result / Failure category；Issue & Request 的 `paintsStatusCellsLikePageTags` 补了评审三态 + 归因标签断言（`F5F3FF` / `EFF3FF` / `EFF6FF` / `FFFBEB` / `FEF2F2`）。
- `mvn -o -T 4 -pl smartop-service/admin -am test -Dtest=ChatbiEventExportServiceStreamTest,UsageFeedbackExportServiceTest` → `Tests run: 11, Failures: 0`，BUILD SUCCESS。
- 顺手修掉一个早已失效的造数：`1e0d17f26` 之后 Source report 要按 `routeState` 判断，用例没给该字段，那条断言此前一直是红的。

## 5. 已知基线失败（与本次改动无关）

`ChatbiPrettyExcelWriterTest` 在 admin-release 基线上就有 4 个用例报错：`IllegalState Cannot get a NUMERIC value from a STRING cell`（`preservesTextColumnAndFormatsKnownNumericColumns`、`keepsRowsBeyondStreamingWindowAndAppliesOutlineLevels:77`、`streamsRowsIncrementallyAndFinishesSheet:142`、`dynamicallySplitsStreamingSheetAndRestoresBusinessOrder:199`）。已用 `git stash` 在干净基线上复现，说明写入器对数字列的识别被人改过。**在修复前，admin 模块整包 `mvn test` 必然 BUILD FAILURE**，不要误判为本分支引入。

## 6. 交付状态

- 提交 `f953c839e feat(admin): 导出文件的状态列补全配色，和页面标签用同一套颜色`（5 文件，+463/−25），分支 `fix/export-cell-tones-admin` 已推送。
- **MR !540** https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/540 → `admin-release`，`merge_status=can_be_merged`、无冲突，**未合并**（沿用本仓库「agent 只创建 MR、合并与部署由用户负责」的约定）。
- 视觉核对图：`.tmp/export-tones-feedback.png`、`.tmp/export-tones-sessions.png`（`office_render` 逐区域渲染，非 PDF 导出）。

## 7. 操作提示

直接按路径拼 `GET /projects/dataagent%2Fssg-sales-ops` 会返回 `404 Project Not Found`（该 GitLab 实例需先按名查项目拿数字 id：`dataagent/ssg-sales-ops` → id=75300），之后 `POST /projects/75300/merge_requests` 正常。`office_render` 对 xlsx 不接受 `pages` 参数，要用 `sheet` + `range`（例如 `H1:Z6`）局部渲染才看得清色块。
