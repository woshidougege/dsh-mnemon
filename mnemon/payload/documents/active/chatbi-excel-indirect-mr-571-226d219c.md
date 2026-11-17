---
id: "226d219c-6c8f-42b4-b913-3c37c3dc92e9"
title: "ChatBI 评审模板 Excel 级联下拉（命名区域 + INDIRECT）与 MR !571 交付"
description: "评审模板 Review status → Attribution tag 级联下拉的实现机制（隐藏 sheet `Review options` + Tag_* 命名区域 + INDIRECT 公式）、$H$5 绝对行导致级联失效的坑与 $H5 修法、文案与导入归一口径必须一致、POI/SXSSF 探针验证结果与「未在真 Excel 点击」的边界、MR !571 状态（接续 !570 已 merged）。"
status: "active"
created_at: "2026-11-17T10:19:48.569Z"
updated_at: "2026-11-17T10:19:48.569Z"
content_hash: "de5660bc855e1c4afd6e5163719abd7c54035150bfdc3dd29c3bc8981efb8039"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-7a991c9b-9c5b-4e40-ba62-017ad3e30042"
memory_body_ids:
  []
---

# ChatBI 评审模板：Excel 级联下拉实现（MR !571，ssg-sales-ops）

来源：!570 交付后用户追问——Review status 与 Attribution tag 两列能否**关联**、Excel 是否支持、Not applicable 是否只有 2 个选项、不填 Review status 是否应禁止填 Attribution tag。结论：能，Excel 原生支持，但**不能靠两个独立内联列表实现**，列表源只能指向固定区域，级联必须走「隐藏 sheet + 命名区域 + INDIRECT 公式」。本批已改为级联并提 MR。

worktree：`C:\Users\63588\lenovo\ssg-sales-ops-wt-reviewio`，分支 `feature/review-import-export`，仓库 `https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops.git`

相关文档：
- `fe2a04fc-7723-479a-a750-552ab8ab1ab6` — !570：导出四层筛选透传、POI 5.5.1 `showDropDown` 语义反转坑、SXSSF 下拉探针法、当时「不做级联」的取舍（本批已推翻该取舍）
- `559c3ec9-b092-48e8-bb50-e45a998e967a` — 功能本体（评审模板导出 + 批量导入）与导入侧校验

## 1. 级联行为矩阵（最终交付）

| Review status（H 列） | Attribution tag（I 列）下拉实际内容 |
|---|---|
| 留空 | 仅一个空白项 —— 等于禁止填写，手输被 STOP 级拦停 |
| Correct | 同上（Correct 不带归因） |
| Wrong | Wrong report / Definition mismatch / Data missing / Display error / Other（5 个） |
| Not applicable | No permission / Out of scope（2 个） |

## 2. 实现机制（三件套）

落点：`smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java`（`reviewTemplateTable` / `streamReviewTemplate` / `tagCascadeFormula` / `optionLists`）与 `.../service/ChatbiPrettyExcelWriter.java`（新增 `addHiddenListSheet` / `addFormulaListValidation`，SXSSF 流式下可用）。

1. **隐藏 sheet** `Review options`：`setSheetVisibility(SheetVisibility.HIDDEN)` → OOXML `<sheet ... state="hidden"/>`。三列分别放：A = Wrong 的 5 个 tag，B = N/A 的 2 个，C = 一个空格占位。
2. **命名区域**（workbook definedName）：`Tag_Wrong` → `'Review options'!$A$1:$A$5`、`Tag_Not_applicable` → `'Review options'!$B$1:$B$2`、`Tag_none` → `'Review options'!$C$1`。
3. **归因列列表源写成公式**（`addFormulaListValidation`，H 为结论列）：

```
INDIRECT("Tag_"&IF(OR($H5="",$H5="Correct"),"none",SUBSTITUTE($H5," ","_")))
```

`SUBSTITUTE` 用来把中文/空格型标签转成合法名称：`Not applicable` → `Tag_Not_applicable`；`none` 分支承接留空与 Correct。

## 3. 关键坑：行引用必须相对（`$H5` 而非 `$H$5`）

第一版写成 `$H$5`（列和行都绝对），整片区域 H5:H2006 全都会读第一行的结论 —— 第二行以后选什么结论都不影响归因列，级联等于没做。必须写 **`$H5`（列绝对、行相对）**，Excel 才会按行平移成 H6、H7…… 实现里用 `CellReference.convertNumToColString(REVIEW_TEMPLATE_STATUS_COLUMN)` 动态取列字母 + 字面数据首行号 5，**不要**拼 `$` 到行号前。

## 4. 文案一致性硬约束

下拉候选文案必须与 `REVIEW_TAG_LABELS` 及导入侧归一表逐字一致，否则用户从下拉挑到的字符串导回时无法识别成同一枚举，整行被判非法。表头说明行同步补两条：`pick from the dropdown (leave empty to keep as is)` 与 `Wrong needs an attribution tag, Not applicable needs No permission or Out of scope`。

## 5. 验证方式与边界

- 编译：`mvn -pl smartop-service/admin -am -DskipTests compile` → BUILD SUCCESS（本批两次，02:04 / 03:25 min）
- 用**生产类本身**生成 xlsx 再用 POI 读回，实测：2 个 sheet（`sheet[1]=Review options visibility=HIDDEN`）、3 个 definedName 指向正确、2 个 dataValidation —— `H5:H2006` 内联列表 + `I5:I2006` INDIRECT 公式、`showDropDown="false"`（箭头可见，印证反转语义）、`errorStyle="stop"`、`allowBlank="true"`；裸 OOXML 里 `<dataValidations count="2">` 与 `state="hidden"` 均在
- **未在真 Excel 里点过**（该机器开不了 Excel）。标准级联写法，但需人工确认：合并后导一份模板，把结论分别选 Wrong / Not applicable / 留空，各点一次归因列
- 兜底：即便 Excel 端被绕过，导入侧原有多重校验仍在（Wrong 必须 5 个之一、N/A 必须 2 个之一、Correct 带的 tag 会被清空）

## 6. 交付状态

- `66a87da54 fix(report): 评审模板归因列改级联下拉`（2 文件 +132/−13：`ChatbiEventExportService.java` +68/−?、`ChatbiPrettyExcelWriter.java` +77）→ **MR !571** `opened / can_be_merged / 无冲突`，https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/571
- 上一批 `0aff6e691 fix(report): 导出跟随人工评审筛选，评审模板两列改下拉`（5 文件 +97/−12）→ **MR !570 已 merged**，!571 只含级联增量
- 无新迁移；合并后需重新部署才生效（生产 flyway 仍 1.1.127）
- 演示/调试用探针脚本在 worktree 的 `.tmp\poi-probe\Probe2.java`（生成 xlsx + 读回 definedName/dataValidation，「同样的 SXSSF 调用能否被流式写出吞掉」的验证法）
