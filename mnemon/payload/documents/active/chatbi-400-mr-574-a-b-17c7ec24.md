---
id: "17c7ec24-408c-407c-9984-699cacb1be0c"
title: "ChatBI 评审模板导入 400 根因与修复（MR !574）+ 导入页改版 A/B 方案"
description: "评审模板四层表头导致导入 400 的根因、findHeaderRow 修复与 Excel 真实行号改造；空 Session ID 行必然 invalid 的原因与导出过滤建议；导入页整页工作区（A）vs 放大弹窗（B）方案对比、估时与待确认项。"
status: "active"
created_at: "2026-11-17T13:25:59.419Z"
updated_at: "2026-11-17T13:25:59.419Z"
content_hash: "dc4e0fb73e05b1150d1b0f42486eff9cdd10a9cc976a73f7e5b17fd458aa6c5e"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiReviewImportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiPrettyExcelWriter.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java"
session_ids:
  - "mnemon-run-v1-7e8173d9-c797-4164-96a1-b760fecb332c"
memory_body_ids:
  []
---

# ChatBI 评审模板「导入 400」根因修复（MR !574）与导入页改版方案（2026-11-17）

状态：后端修复已提交并开 MR（!574，`can_be_merged`，1 个文件，前端无需改动）；**导入页改版与「导出跳过空 Session ID 行」两项仍在等用户拍板，未动代码**。

接续：
- `3a6372e3-9537-40ce-8d30-142935b895ae` —— 导入/导出评审模板的最初方案（模板列、校验、待拍板决策）
- `226d219c-6c8f-42b4-b913-3c37c3dc92e9` —— 评审模板 Review status → Attribution tag 级联下拉（MR !571，本次已合入）

## 1. 报错现象与根因

用户导出模板、改完再导入，前端只弹 `Request failed with status code 400`（axios 默认文案，真实信息在顶部 toast）；后端 msg：

```
The file is not a Review template: column session id is missing.
```

导出的评审模板是**四层表头结构**（由 `ChatbiPrettyExcelWriter` 统一生成）：

| Excel 行 | 内容 |
|---|---|
| 1 | 标题带 `ChatBI Usage · Review template` |
| 2 | 说明行（`… every question turn, one row each; Session ID + Turn key identify the row …`） |
| 3 | 6pt 高的空行 |
| **4** | **真正的表头（11 列）** |
| 5+ | 数据 |

而 `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiReviewImportService.java` 用逐行读取模式（空行被丢弃），`importTemplate` 直接把**第一条非空行**当表头（`headerIndex(raw.get(0))`，原 line 90-105 附近），于是拿标题带去查 `session id`，必然 400。

**结论：导出的模板从未被自己的导入真正读过——上一轮只做了编译级验证，这是设计缺口，不是用户操作问题。**

## 2. 修复内容（commit `2fffa79e`，分支 `feature/review-import-export`）

- 新增 `findHeaderRow`：在前 20 条非空行里找**同时出现 `Session ID` 与 `Turn key`** 的一行作为表头，数据从它下一行开始；找不到时给出明确报错。
- 行记录由裸 `List<String>` 改为 `Row(excelRow, cells)`，导入报错里的行号改为 **Excel 真实行号**（原先是「跳过空行后的序号 +1」，与用户看到的行号对不上）。
- 兼容手工删掉标题带的情况：表头正好是第一条非空行时同样命中。

验证方法（可复用）：用**生产类** `ChatbiPrettyExcelWriter` 造同构模板（标题带 + 说明 + 空行 + 11 列表头 + 数据），再反射调用修复后的解析，打印 headerPos 与解析出的列索引映射，确认 11 列全部命中：

```
row[0] excelRow=1 -> |ChatBI Usage - Review template
row[1] excelRow=2 -> |note line ... part 1
row[2] excelRow=4 -> |Session ID|Turn key|Time     ← 表头在这里
headerPos=2
header={session id=0, turn key=1, ..., review source=10}
```

`mvn -pl smartop-service/admin -am compile` → BUILD SUCCESS。

相关常量位置（`ChatbiPrettyExcelWriter.java`）：`REVIEW_TEMPLATE_HEADERS` line 271、`reviewTemplateTable` line 305、`attachReviewValidations` line 367-383、`tagCascadeFormula` line 390-397、`reviewTemplateRow` line 400。

## 3. 遗留缺陷：空 Session ID 的行永远导不进去

修复后 470 行全部解析成功，但其中 10 行 `invalid`——它们的 `Session ID` 是空串。原因链：

- 评审主键是 `(session_id, turn_key)`；
- 导出侧 `SESSION_EXPORT_SELECT` 写的是 `COALESCE(session_id, '') AS "sessionId"`（`ChatbiEventExportService.java` line 1490 起），库里 `session_id` 为 NULL 的轮次被导出成空串；
- 这些 turn 在页面上也评审不了——抽屉会提示 “This record has no session link, so it cannot be reviewed”。

**建议（未实施）**：只在导出**评审模板**时加 WHERE 条件跳过 `session_id IS NULL` 的轮次，其他页签导出保持原样，invalid 自然归零。改动一处 WHERE。

## 4. 导入页改版：两个方案与推荐

用户反馈：900px 弹窗里塞 7 列，「信息量太大」，问是否整页显示更好。诊断出的结构问题（不只是信息量）：

- Question 被截成 `this isnt what im looking for, im looki…`，After 折三行；
- 变化表与跳过表**上下堆叠**，要滚 3 屏才能看完两组信息；
- 统计用两行抽象文案 `Will overwrite 0 automatic · 0 human verdicts`，Invalid 这类聚合信息没有出口；
- 术语歧义：`Skipped` 在统计里指“未改动”，在表格标题里又指“导不进去的行”。

| 方案 | 内容 | 改动量 | 估时 |
|---|---|---|---|
| **A 整页工作区**（推荐） | 新建 `ReviewImportView.vue`（约 350 行）+ `index.vue` 加 view 切换（不跳路由，可 Back 回列表）；指标条横排 + 一句黄色提示；Changes / Skipped 改 tab（默认 Changes）；表格加 Type 列（Overwrite auto / Overwrite human / Newly marked）；底部固定操作栏常驻 | 2–3 文件，一个 MR | 半天内 |
| B 放大弹窗 + tab | 只改 `ReviewImportModal.vue`：92vw × 88vh、两表改 tab、列宽重排 | 1 文件 | 半小时 |

推荐 A 的理由：批量评审会反复使用；两种注意力（逐条确认变化 vs 排查跳过/异常）不该挤在同一屏。文案统一为 **Changed / Unchanged / Invalid**。方案草图截图：`C:\Users\63588\lenovo\ssg_sales_ops-wt-nareview\.tmp\import-preview.png`（图中的 8 条数据是照用户截图造的示例）。

## 5. 待用户确认的两件事（截至本记录）

1. 导入页走 A（整页工作区）还是 B（放大弹窗 + tab）；
2. 10 条空 Session ID 的行是否在导出时直接跳过。

另有一个已提出、未回答的小问题：是否把弹窗顶部红条也改成显示后端真实 msg，而不是 axios 的 `Request failed with status code 400`。

## 6. 教训

- 导出模板必须用自家导入器跑一遍**往返测试**（export → import 空改），编译通过不等于格式自洽。
- 解析 Excel 时不要在“跳过空行后的序号”上做行号算术；行号应直接来自 POI 的 `row.getRowNum()`，否则报错信息与用户看到的 Excel 行永远差几行。
- 表头定位不要依赖“第一条非空行”，要按**特征列名组合**（`Session ID` + `Turn key`）定位。
