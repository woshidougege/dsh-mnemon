---
id: "03324a5b-e25e-4f46-9002-5f8cbddf3bb5"
title: "ChatBI Source report 五步归因规则：准确率实测与两处补丁（30 天生产数据）"
description: "五步归因规则在 4903 轮生产数据上的逐步命中量与可验证准确率：route 3683/exec 516/resolve 146/carried 423/none 135，可验证集 3048→3047=99.97%；补丁 1（carried 要求本轮有查询动作，423→100）与补丁 2（exec 优先于 route，消除唯一错例）后确定 4345(88.6%)+推断 100(2.0%)+无报表 458(9.3%)，Not routed 1201→458，残余 704 轮无客观真值需抽检。"
status: "active"
created_at: "2026-11-18T02:33:42.929Z"
updated_at: "2026-11-18T02:33:42.929Z"
content_hash: "ddca2c1a297a1750b40f9acb945062adb820e161fe463a08c826ffb914419078"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java"
session_ids:
  - "mnemon-run-v1-b00be74b-fea9-430e-9f88-2ef421df1759"
memory_body_ids:
  []
---

# ChatBI Source report 五步归因规则：准确率实测与两处补丁

来源：2026-11-18 针对 `ssg-sales-ops` 生产库（Smart Data Query 模式、近 30 天）对五步归因规则的量化验证；源码核对在同级目录 `ssg-spark`。

**关联文档**：`f1d199f9-9322-4c39-b7f1-ca8160e6fc52`（Not routed 成因分类与"沿用上轮路由"可行性裁决）、`25fb29f1-f532-44dc-b110-618cf4796fdc`（轮次级改造实施方案）、`d7382448-aaf9-4003-971f-58f1f56da61b`（三列取值链）。本文只补前者未覆盖的部分：五步规则的**逐步命中量、可验证准确率、两处必要补丁、残余风险**，不重复成因分类。

## 1. 被验证的五步规则（原始版）

1. 本轮 `route_report` 的 report_id → 报表名
2. 本轮 `execute_compiled_query` / `execute_sql` 的 report_id → 报表名
3. 本轮 `resolve_plan` 的 report_id → 报表名
4. 以上都取不到，但本会话此前有路由 → 沿用并标 "↺ Carried over"
5. 以上都取不到 → "No report this turn"

## 2. 逐步实测（4903 轮）

| 步 | 命中轮次 | 占比 | 可验证轮次 | 与真实执行一致 |
|---|---|---|---|---|
| 1 route | 3683 | 75.1% | 2532 | 2531（99.96%） |
| 2 exec | 516 | 10.5% | 516 | 516（100%，同源） |
| 3 resolve | 146 | 3.0% | 0 | 无法验证（编译产物未执行） |
| 4 carried | 423 | 8.6% | 0 | 无法验证 |
| 5 none | 135 | 2.8% | 0 | — |

真值判定方法：以**本轮执行输出里的 `report_id`** 为客观真值（系统实际跑的那份报表）。可验证集 3048 轮命中 3047 轮 = **99.97%**，唯一错例为 2026-11-03 `longyu3` 的 ship report 轮：路由到 `simu`、实际执行 `Services_Bookings rdl_isg_gaap_revenue_ca`。

该"异常"报告并非脏数据：它在 `agent.sdq_reports` 中真实存在、`status=active`、`report_name = ISG PR`（同表还有 inactive 的 `Services_Bookings`、`Services Bookings hw_sub_series_description` 三个同名残留）。即命名不规范，不是解析 bug。

分桶复核（同口径重算，合计 4905）：A 路由且本轮执行与路由一致 2998 / B 路由但与执行不同 1 / C 路由无执行报表 685 / D 无路由无执行报表 558 / E 无路由且执行 = 上一轮路由 659 / F 无路由但执行不同 1 / G 无路由且无历史路由 3。E/F/G 合计 663 中 659 命中"沿用上一个"，命中率 99.4%。

## 3. 第 4 步（carried）的隐藏缺陷

423 个 carried 轮按本轮实际动作拆解：

| 本轮动作 | 轮次 | 影响 |
|---|---|---|
| 有查询动作（execute / revise / resolve） | 100 | 机制上必然正确（plan state 锁定同报表） |
| 只有其他工具（澄清 / 查目录 / 记忆） | 123 | 勉强合理 |
| **完全没有工具调用** | **200** | 纯对话、知识问答却被贴上报社名（如 `SSG Bookings`），纯误导 |

独立交叉检验（看 carried 轮之后的走向）：后续无路由 285 / 后续仍是同一报表 90 / **10 分钟内就换报表 33** / 10–60 分钟换 8 / 更晚换 7。33/423 = 7.8% 有误标嫌疑（最可能是"用户已换话题但本轮未走路由"）。

## 4. 两处补丁与最终口径

- **补丁 1：第 4 步加前提——本轮必须存在查询动作**（`execute_compiled_query` / `execute_sql` / `revise_query` / `resolve_plan` 任一），纯对话轮直接落到第 5 步。效果：carried 423 → **100（2.0%）**，none 135 → **458（9.3%）**。
- **补丁 2：第 1、2 步换序，exec 优先于 route**。`exec` 的 report_id 是"实际执行的那份"，`route` 只是"打算用的"；换序后唯一错例被消除，其余 2998 轮两者本来就相等，无副作用。

补丁后指标：

| 口径 | 轮次 | 占比 | 准确率 |
|---|---|---|---|
| 确定（exec → route → resolve） | 4345 | 88.6% | 可验证集 3048/3048 = 100% |
| 推断（carried，本轮确有查询动作） | 100 | 2.0% | 机制确定，估计误标 ≈8 轮（0.16%） |
| 无报表（No report this turn） | 458 | 9.3% | 其中 234 轮确实没调任何工具，判定正确 |

`Not routed` 从 1201 轮（24.4%）降到 458 轮（9.3%），Source report 可解释覆盖率 75.3% → 88.6%。

## 5. 残余风险与验收建议

- 146 轮（resolve）+ 100 轮（carried）+ 458 轮（none）= **704 轮（14.4%）没有客观真实值**，只能靠机制论证。上线后建议抽 20 条 carried 人工回看，比继续追 SQL 覆盖率更稳妥。
- report_id → report_name 映射：第 1–4 步命中值全部存在于 `agent.sdq_reports`（`rid_not_in_dict = 0`），不会出现"取到报表却显示 Unknown report"。
- 第 5 步 135 轮（旧口径）全部本来就没有 report_id，属正常。

## 6. 落地改动位置（沿用关联文档，补充优先级要求）

- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java`
  - `:550-553` `ROUTE_REPORT_ID_EXPR`：exec / resolve 侧复用同一正则解析输出体 `"report_id":"…"`
  - `:564-622` `SOURCE_LATERAL_SQL` 增本轮 report_id 解析并按 **exec → route → resolve** 排序；`ROUTE_STATE_SQL` 增 `carried` / `no_report` 两态；`SOURCE_LABEL_SQL`、`SOURCE_KEY_SQL`（下拉分组）同步
- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java:1945-1951`：导出只认 `routed` / `route_failed`，必须加新状态，否则 Excel 里新口径被抹成 `—`
- 前端来源下拉：`carried` / `no_report` 的英文文案（SPARK 为英文平台）
- 机制依据（同级 `ssg-spark` 仓库）：`app/ssg/smart_data_query/runtime/planned_state.py:87/112/114/161/665` 强制 report_id 一致；`app/ssg/smart_data_query/continuation.py:696,713` 续跑要求 `execution["report_id"]` 非空。因此 carried 在"本轮确有查询动作"时机制上是确定的。
