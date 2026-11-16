---
id: "e6544a1f-d30c-42ee-8f77-dead8deb331c"
title: "ChatBI 评审三态（Not applicable）落地交付与两库数据核查（2026-11-16）"
description: "三态改造已完成的实现细节与定稿统计口径、AUTO+No permission 由 DELETE 改为改判 N/A 的条件与调用点、Backfill 按钮执行顺序、测试/生产 flyway 版本与 NO_PERMISSION 命中 0 行的核查证据、三条待合并 MR 与 tmp 分支、待改文案与手工 SQL 的前置条件。"
status: "active"
created_at: "2026-11-16T08:08:12.743Z"
updated_at: "2026-11-16T08:08:12.743Z"
content_hash: "28d1f750cdd3555db87b2e8c2101394b1958357c5139afc9c06ef7cb2eb72c40"
source_paths:
  - "src/views/Setting/settings/panels/ReviewRulesPanel.vue"
  - "src/views/Setting/reports/user-logs/api.ts"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiQuestionReviewService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiQuestionAutoReviewService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiQuestionAutoReviewMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/controller/ChatbiEventReportController.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.126__Add_question_review_not_applicable.sql"
session_ids:
  - "mnemon-run-v1-cf8770e3-8d5f-4009-a3a3-b96c7398833c"
memory_body_ids:
  []
---

# ChatBI 评审三态（Not applicable）落地交付与两库数据核查（2026-11-16）

来源：三态改造完成并部署测试环境后，用户问「是不是需要手动处理数据，还是点一下处理历史数据的那个按钮」。据此连测试库与生产库做只读核查，并整理交付状态。改造前的方案与取舍见 **c9e39da9-9d7e-4222-9082-b1cb80ef8447**（本文不重复设计理由）。

## 1. 已落地改动

结论由两态扩为三态：Correct / Wrong / **Not applicable（NOT_APPLICABLE，chip 显示蓝灰 N/A）**。

- 评审抽屉第三颗结论按钮为 `–`；切换结论时下拉候选按分组替换（Wrong → 「错在哪」，N/A → 「为什么」），旧标签不属于新分组即清空。
- Wrong 的归因候选里移除了 `No permission`；N/A 的原因候选为 `No permission` / `Out of scope`。
- 三个页面（User Logs 的 Query / Feedback 页签、Issue & Request 点踩列表）的筛选候选、行内标签、准确率卡片均支持第三态。
- 后端 `ChatbiQuestionReviewService` 做三态校验，报错文案用英文（英文平台约定）。
- 迁移 `V1.1.126__Add_question_review_not_applicable.sql` 只改表结构：放开主表 `status` / `attribution_tag`（新增 `NOT_APPLICABLE`、`OUT_OF_SCOPE`）与流水表 `from_status` / `to_status` 的 CHECK，不动数据。
- 导出（会话导出、点踩导出）的单元格文案与配色补第三态。

### 统计口径（定稿）

| 指标 | 公式 |
|---|---|
| Accuracy Rate | `Correct ÷ (Correct + Wrong)`，N/A 不进分母 |
| 覆盖率 | `(reviewed + N/A) ÷ total`，勾 N/A 也算有人处理过 |
| 待评审数 | `total − reviewed − N/A` |

归因图与原因图分离：`byTag` 只统计 Wrong 的归因，`byReason` 只统计 N/A 的原因，避免 `No permission` 在两张图里重复出现。

## 2. 自动评审：从 DELETE 改为改判 N/A

- 旧逻辑 `deleteLegacyAutoNoPermission()`（`ChatbiQuestionAutoReviewMapper.java:113-116`）直接 DELETE，这些轮次会退回 Pending、丢失结论。
- 现改为 `convertLegacyAutoNoPermission()`（`ChatbiQuestionAutoReviewMapper.java:130`）：`WHERE review_source='AUTO' AND status='WRONG' AND attribution_tag='NO_PERMISSION'` → 改判 `NOT_APPLICABLE`，备注补 `Auto: no permission, converted to not applicable.`
- 调用点：`ChatbiQuestionAutoReviewService.java:71`（saveSetting）、`:102`（runBackfill）。**该改判全表执行，与回填选的日期范围无关**；日期范围只影响后续补漏扫描。
- 前置条件：`auto_review_enabled` 必须为开，否则接口直接抛 `Turn on Auto review before running a backfill`（`ChatbiQuestionAutoReviewService.java:93`）。
- 人工结论（`review_source='HUMAN'`）不被这条逻辑触碰——这是刻意的，人工判断不应被自动改判。

## 3. Backfill 按钮的真实语义

入口：Admin settings → Operations → Review rules → Backfill（前端 `src/views/Setting/settings/panels/ReviewRulesPanel.vue`，注意目录是 `Setting/settings/panels`，旧的 `Setting/panels` 路径已不存在）。

- `runBackfill`(:207) → `executeBackfill`(:216) → `reportApi.reviewBackfill`(:225) → `POST ${API_PREFIX}/queries/review/backfill`（`src/views/Setting/reports/user-logs/api.ts:235-236`）。
- 后端 `ChatbiEventReportController.java:849-862` → `ChatbiQuestionAutoReviewService.runBackfill`。
- 执行顺序：**先**改判历史 AUTO+NO_PERMISSION，**再**扫描补漏。面板文案明确写「Manual only / no background job, no nightly scan」。
- 面板上 `Save` 保存一次规则同样会触发改判（走 saveSetting）。

## 4. 两库只读核查结论：无需处理任何数据

| 项 | 测试（10.62.79.148 / `application-lmp-*-tst.yml`） | 生产（10.62.79.215 / `application-lmp-reston-prod.yml`） |
|---|---|---|
| flyway 最新版本 | **1.1.126**，success=t，installed_on 2026-11-16 16:01:11 | **1.1.124**（三态迁移尚未发布） |
| CHECK 是否含三态 | 是（`status`、`attribution_tag`、流水 `from_status`/`to_status` 均含 `NOT_APPLICABLE` + `OUT_OF_SCOPE`） | 否 |
| AUTO + Wrong + No permission | **0 行** | **0 行** |
| HUMAN + Wrong + No permission | **0 行** | **0 行** |

测试库状态分布：`AUTO/WRONG` 共 734（WRONG_REPORT 336、OTHER 314、DEFINITION_MISMATCH 77、DISPLAY_ERROR 7），`HUMAN` 侧 WRONG 5（DISPLAY_ERROR 2、WRONG_REPORT 1、OTHER 1）与 CORRECT 1。
生产库：`AUTO/WRONG` 735 行，`HUMAN/WRONG` 1 行，`NO_PERMISSION` 在任何 source/status 组合下均为 0。

原因：自动规则 SQL 本来就排除 LUDP 权限失败，`NO_PERMISSION` 从未被写进表里。因此**按钮点下去是空操作，手工 SQL 也没有可改的行**。

## 5. 交付状态与待办

- 三条 MR 均 `can_be_merged`、无冲突，等用户点名后合并：
  - 前端 !1047 `dev/renxp4` → `master`：https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1047
  - 前端 !1048 `tmp/review-na-develop` → `develop`：https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1048
  - 后端 !557 `feature/review-not-applicable-admin` → `admin-release`：https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/557
- `develop` 那条走一次性承接分支 `tmp/review-na-develop`：`develop` 上另有「页面选择器改为按菜单快照选页」的提交与 master 分叉，直接用 `dev/renxp4` 合会带入 master-only 噪音；cherry-pick 过去是干净的，**合并后需删除该 tmp 分支**。
- 验证记录：后端 `mvn -o -T 4 -pl smartop-service/admin -am -DskipTests compile` BUILD SUCCESS；前端 `npm run type-check` 通过（两个目标分支各跑一次）；两目标分支 diff 补丁 patch-id 一致（`2d4a8f0e`）。按项目约定未开浏览器做页面验证，也未连库执行迁移（迁移由部署自动跑）。
- 待改文案（已提出，等用户批准）：`ReviewRulesPanel.vue:358-360` 仍写 `only legacy automatic No-permission verdicts are cleared`，而实现是改判而非清除，应改为 `converted to Not applicable`。
- 手工 SQL 的未来用法（当前两库命中 0 行，暂不需要）：仅针对人工判过的 `HUMAN + WRONG + NO_PERMISSION`，且**目标环境 flyway 必须先到 1.1.126**，否则 `status` 的 CHECK 会拒绝 `NOT_APPLICABLE`：

```sql
UPDATE public.ssg_chatbi_question_review
SET status = 'NOT_APPLICABLE', remark = COALESCE(NULLIF(btrim(remark),''), 'Migrated from wrong/no-permission')
WHERE status = 'WRONG' AND attribution_tag = 'NO_PERMISSION';
```

## 6. 相关文档

- 三态方案与实施清单（改造前决策）：`c9e39da9-9d7e-4222-9082-b1cb80ef8447`
- 生产排查（筛选返回空 + 653 条 backfill 漏评审）：`acb35915-0667-44bf-a5e9-8dac79c502d8`
- backfill 纳入修复 + 筛选候选联动（MR !543/!542/!1021/!1022）：`2b13c2e2-62ad-49c9-b0f2-5bb15637a132`
- Accuracy Rate 口径与数据模型：`38b9c4c0-0988-4206-9198-11f20587ae61`；评审改动留痕（V1.1.120）：`6353a23e-5302-4b9c-9301-049e028565d3`。
