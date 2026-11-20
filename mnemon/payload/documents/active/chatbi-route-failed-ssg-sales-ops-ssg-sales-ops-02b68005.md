---
id: "02b68005-05ee-4b33-b617-f9c3b01164f4"
title: "ChatBI 使用分析页三块指标口径释疑 + Route failed 列降级方案（ssg-sales-ops / ssg_sales_ops）"
description: "Usage Analytics（user-logs）页 Accuracy Rate / Success rate / Route failed 三处口径的代码级解释（含 SQL FILTER、routeState 七态、success 三选一判定），以及 Route failed 作为汇报指标无信息量的论证与 A/C 两种降级方案（未实施，待用户拍板）。"
status: "active"
created_at: "2026-11-20T02:24:08.376Z"
updated_at: "2026-11-20T02:24:08.376Z"
content_hash: "83584b12e837ade186504b4d2eea3e9815edc938eb4c8e5c0eb901b1900b96a2"
source_paths:
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/index.vue"
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue"
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/api.ts"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/controller/ChatbiEventReportController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "smartop-service/admin/src/main/resources/mapper/chatbi/ChatbiEventQueryMapper.xml"
session_ids:
  - "mnemon-review-v1-27f61da1-be5e-4a4f-9d7f-264b08a368a4"
memory_body_ids:
  []
---

# ChatBI 使用分析页三块指标口径释疑 + Route failed 列降级方案（ssg-sales-ops / ssg_sales_ops）

来源：用户拿一张 Users → Usage Analytics 页截图，圈出 Accuracy Rate 卡片、Success rate 卡片、来源表 Route failed 列，要求解释这三处代码是怎么算出来的；随后追问「Route failed 对领导汇报有没有意义」「改成 hover tooltip 是否更好」。以下为当轮查证结出的口径说明与 UI 处置建议。**代码未改，等用户选方案。**

## 1. 链路（前端在 ssg_sales_ops，后端在 ssg-sales-ops）

| 层 | 位置 |
|---|---|
| 页面入口 | `ssg_sales_ops/src/views/Setting/reports/user-logs/index.vue:36-57`（TABS 四项 query/feedback/pages/raw，`activeTab` 默认 `query`） |
| 三块卡片 + 来源表渲染 | `ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue:190-256`（卡片）、`:235-256`、`:147-165`（顶部 Route failed 汇总条）、`:365-391`（来源表 Route failed 列） |
| 卡片组件 | `src/views/Setting/reports/user-logs/components/ui/MetricCards.vue` |
| 接口 | `POST /api/admin/chatbi/events/report/queries/summary` → `api.ts:352` `queriesSummary` |
| 控制器 | `ChatbiEventReportController.java:725-741` `queriesSummaryPost`（同文件 `:703` 是明细 `/queries`） |
| 汇总组装 | `ChatbiEventReportService.java:1206-1340`（review 块、sourceStats）、`:684` `ratio()` |
| SQL | `smartop-service/admin/src/main/resources/mapper/chatbi/ChatbiEventQueryMapper.xml:154`（sessionSummary）、`:158`（sessionReviewStats）、`:166`（sessionSourceStats）、`:170`（TOP 问题） |
| 路由七态常量 | `ChatbiEventQueryMapper.java:555-588` |

页面注释写明它由旧后端模板 `smartop-service/admin/src/main/resources/templates/chatbi/event-report.html` 迁来，视觉与口径刻意保持与旧页一致。

## 2. 三个指标是三套口径，互不对应

| 指标 | 公式 | 截图实测 | 含义 |
|---|---|---|---|
| Accuracy Rate | `correct / reviewed`，`reviewed = CORRECT + WRONG`（N/A 不进分子分母）；`reviewed=0` 返回 `null` → 前端显示 `—`，不装 0% | 0 / 21 = **0.0%** | 人工/自动评审对人给的判断 |
| Correct / Reviewed · Coverage · N/A · Pending | `pending = total - reviewed - notApplicable`；`coverageRate = (reviewed + notApplicable)/total` | 0/21；`(21+11)/470 = 6.81%` → 前端 `Math.round` 得 **7%**（Accuracy 用 `toFixed(1)`，Coverage 用整百分比） | 样本量与覆盖率 |
| Success rate | `(total - failed)/total` | `(470-20)/470 = 95.745%` → **95.7%**，caption `Failed 20` | 机器「有没有拿到合规结果」 |

关键结论：**Accuracy 0.0% 与 Success 95.7% 并存不是 bug**，一个是人评判断，一个是机器出数判定。

评审数据在 `public.ssg_chatbi_question_review`，主键 `session_id + turn_key`；`turn_key = COALESCE(NULLIF(btrim(query_id),''), 'row:' || id)`，见 `ChatbiEventQueryMapper.java:636-651`（REVIEW_JOIN + Jira LATERAL）、`:657-667`（REVIEW_SELECT_SESSION）。`reviewed` **不区分来源**：`review_source=AUTO` 的规则评审与 `HUMAN` 一起进 correct/wrong —— 21 条全 WRONG 时，0.0% 很可能主要来自自动规则评审。

## 3. 成功判定是三选一（Success rate 的分子来源）

`sessionSummary` 里失败 = 下列三条都不满足：
1. 事件自身 `status` 或 `properties.answer_status` = `success`；
2. 同 session+turn 的 `agent.tool_audit_logs` 存在 `status='succeeded'` 且 `tool_name ∈ (execute_compiled_query, execute_sql, finalize_query_result)`，且 `output_summary` 里正则匹配到 `contract = smart_data_query_result/v1` 且 `status = success`；
3. 上述任一成立即算成功。

所以这个卡片衡量「这一问有没有拿到合规 SDQ 结果」，与答得对不对无关。另外 `total=470` 是全量问题（含非 Smart Data Query 轮次），来源表 `Smart Data Query Question` 合计 381，差的 89 条不进来源分布——**卡片与来源表的 % 不能互相校验**。

## 4. Route failed 的真实定义（不是 failed 的子集）

`sessionSourceStats`（`ChatbiEventQueryMapper.xml:166`）：

```sql
count(*) FILTER (WHERE CASE
  WHEN sp.route_rid IS NOT NULL AND sp.route_ok THEN 'routed'
  WHEN sp.route_rid IS NOT NULL THEN 'route_failed'
  ...
END = 'route_failed')
```

- `route_rid IS NOT NULL` = 本 turn 有 `route_report` 审计记录；`route_ok` = 这些记录里按 `status='succeeded'` 优先、时间倒序取第一条是否成功（同轮重试成功一次即算 routed）。
- 七态：`routed / route_failed / exec / resolve / carried / no_audit / none`（`ChatbiEventQueryMapper.java:581-588`）。`exec`（直接执行）、`resolve`、`carried`（沿用上轮报表）**都不算 route_failed** —— TCV 79、No report 63 这列是 `—`，是走了这些路径，不是路由挂了。
- 与 `failed` 是**两个独立 `count(*) FILTER`**，既不互斥也不是子集，四列不能相加。唯一自洽的等式是 `success + failed = total`（181+13=194 ✓、78+1=79 ✓、59+4=63 ✓）。
- 前端 0 值渲染成 `—`：`QueryView.vue:390` `num(item.routeFailed) ? … : '—'`。
- 顶部汇总条 `Route failed 3 (0.8%)` = 各来源行求和，只在 `>0` 时整条渲染（`:147-165`）；`0.8% = 3/381`，分母是 SDQ 问题数，不是 470。

## 5. 判断：作汇报指标无信息量，作诊断指标不可替代

不适合放第一屏的三条理由（都有代码依据）：
1. **命名诱导误读**：`Failed 13` 与 `Route failed 2` 并排，天然被读成「2 是 13 的子集」，而实际是两个独立 FILTER。
2. **衡量中间环节**：`exec/carried/resolve` 全不进该列，且页面不告诉读者分母（只对调过 route_report 的轮次有意义），可比性弱。
3. **数量级不够支撑决策**：0.8% 属噪声区间，看不出趋势、不触发动作。

不可替代性：它是区分「报表没认出来」（route_failed）与「认出来了但查不出数」（failed 且 routeState=exec）的**唯一聚合出口**，属二级诊断，性质同 `Failure category`（Result quality below threshold / Parameter validation error…）与评审 `byTag`（Wrong report / Data missing / Definition mismatch）。结论：**降级而非删除**。

## 6. 已讨论的处置方案（未实施，等用户拍板）

- **A 最小改动**：表格只留 `total / success / failed / success rate`，Route failed 收进行内 tooltip 或 TOP5 下钻。落法有三件事——(1) 表头加 ⓘ 并与列名一起改 `Routing failed`，文案示例 *"Turns that called route_report and failed; not a subset of Failed. — means this source didn't re-route this turn."*；(2) 数字 tooltip 只在 `>0` 时给「2 of 194 turns (1.0%)」；(3) 顶部汇总条维持 `v-if` 现状。理由：误解发生在读表头时，只给数字挂 tooltip 无效，且 99% 单元格是 `—`，悬停 `—` 不会有人认为有内容。
- **C 收成健康提示**：`3 turns failed to route (0.8%) — view details`，表格彻底去列；对截图/投屏场景最友好（hover 等于不存在）。
- **推荐 A 主体 + C 兜底**：表头 tooltip 讲口径，列保留但 0 显示 `—`，顶部汇总条作为截图里唯一能读到的解释。
- 改动范围（若动手）：只动 `QueryView.vue:369`（表头）与 `:390`（数据格）+ 少量 tooltip 样式；**不动后端、不动 SQL**，`routeFailed` 字段照旧返回。表格是 `div + grid` 的 `source-row`／`<span>`，不是 `el-table`，用 `el-tooltip` 包表头 span 即可，不影响布局。

## 7. 未决 / 待确认

1. 用户尚未在 A / C / 「A+C」之间选定，故本轮**未改任何代码**。
2. 0.0% 的成因未落库核验：可通过 `p_renxp4` 拉这 21 条按 `status`、`review_source` 分组，确认是否全是 AUTO 规则评审；以及打印 2 条 `route_failed` turn 的明细。用户未答复是否需要连库。
3. 与 Accuracy Rate 恒 0% 的更大口径问题相连：见文档 `2b13c2e2-62ad-49c9-b0f2-5bb15637a132` §8.1（自动评审只对失败轮次出结论 → 生产 281 条评审全 WRONG）。本次截图 21 条属同类现象的不同数据窗口。
4. 若改 UI，需确认这页主要靠人登录看还是靠截图汇报——决定 A 是否足够。

## 8. 相关文档

- `38b9c4c0-0988-4206-9198-11f20587ae61`：Accuracy Rate 设计（概念拆分、卡片改法、建表与接口）
- `ea0c4abe-9366-4b26-964c-e4b8149e3fad`：人工评审 + Accuracy Rate 落地记录（三支 MR、`showAgentSuccessRate=false` 降级旧口径）
- `2b13c2e2-62ad-49c9-b0f2-5bb15637a132`：自动评审 backfill 修复（含 Accuracy Rate 口径未决项）
