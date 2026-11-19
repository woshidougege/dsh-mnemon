---
id: "a46010e5-2a40-4544-ae85-751d4058b950"
title: "ChatBI 状态口径分歧取证（234bc98b vs 0af54804）+ 注解 SQL 迁 XML 防复发审查（MR !578）"
description: "2026-11-19：两轮都 LUDP 无权限却一轮 Success 一轮 Failed 的生产取证（根因 revise_query 失败使 step_failed=1，LUDP 分支压过它；FINAL_SUCCESS_SQL 不看 LUDP，A/B/C 三方案待定）；MR !578 注解 SQL→XML 的防复发结论、覆盖缺口（chatbi 188+58 未迁）、Mapper 全解析冒烟测试建议，以及 V1.1.130 三方撞号须改 1.1.131。"
status: "active"
created_at: "2026-11-19T01:51:55.383Z"
updated_at: "2026-11-19T01:51:55.383Z"
content_hash: "e9eb9b986e237fac325c823a4dc5f350e7b5a56f09079591cf17d8d2f5a242a2"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/jira/ChatbiReviewJiraMapper.java"
session_ids:
  - "mnemon-review-v1-6246a6f9-4e72-4487-9b87-93ae15f16c1f"
memory_body_ids:
  []
---

# ChatBI 状态口径分歧取证 + 注解 SQL 迁 XML 防复发审查

日期：2026-11-19（生产数据 11-18 23:42 与 11-19 00:19 两轮）
相关既有文档（语义细节不重复，见各自条目）：`f60c9e73-60f7-4ded-875a-34001eb9d258`（无权限不进候选 + 37 条 Success·No LUDP permission）、`64ae3942-bdb8-4694-bb7e-9ef24b218c19`（Final result × Failure category 规则化 + V1.1.130）、`2b13c2e2-62ad-49c9-b0f2-5bb15637a132`（判定常量行号速查）。

## 一、个案：两轮都 "没权限"，结果一个 Success 一个 Failed

用户疑问：两个 session 都是 LUDP 无权限，为什么最终状态不同。生产库事实：

| 项 | `234bc98b-c22c-41b8-8f3a-8f1ff06663df` | `0af54804-ce80-4870-bb1f-06decbdb5302` |
|---|---|---|
| 时间 | 11-18 23:42:25 | 11-19 00:19:37 |
| 问题 | Lenovo base warranty. | Premier Support |
| 事件 status | **failed** | **success** |
| failed_step / failed_reason | `revise_query` / `tool_business_error` | 空 |
| step 统计 | 6 步：5 成功 **1 失败** | 5 步：5 成功 0 失败 |
| `execute_compiled_query` | succeeded，输出 `smart_data_query_failure/v1` + `reason=ludp_permission_denied` | 同左（LUDP 拒绝） |
| `FINAL_SUCCESS_SQL` 复算 | Failed (`f`) | Success (`t`) |
| Failure category | No LUDP permission | No LUDP permission |

失败那一轮的唯一失败工具是 `revise_query`（23:42:32），输出：
`{"status":"error","error_code":"query_revision_failed","message":"The query context was restored, but preparing the revision failed. No revised query was executed and no result is available.","retryable":false,"failure":{"tool_name":"revise_query","error_type":"tool_business_error"}}`
→ 这条 failed 使 `step_failed=1`，agent 上报整轮 `status=failed`。与权限无关。

成功那轮全程无失败工具：LUDP 拒绝发生在 `execute_compiled_query` 的**业务结果**里（工具调用本身 `succeeded`），agent 把"你没有该 LUDP 项目权限"当正常回答收尾并上报 `status=success`。

## 二、为什么两轮分类都是 No LUDP permission

`ChatbiEventQueryMapper.STATUS_CATEGORY_SQL`（checkpoint 中出现于 :462-466）分支顺序为 LUDP 优先：

```
CASE WHEN <LUDP_PERMISSION_SQL> THEN 'No LUDP permission'
     WHEN <NUM_STEP_FAILED> > 0 THEN ...
```

所以 LUDP 判定**压过** `revise_query` 失败。这也是规则表里能同时存在 `Success · No LUDP permission` 与 `Failed · No LUDP permission` 两行的原因——两者 "Final result" 键不同。

## 三、口径问题与三个选项（未改代码，待用户定方向）

`FINAL_SUCCESS_SQL`（checkpoint :416-419、:1349-1353 附近引用）只判两件事：事件自报 status/answer_status=success，或前端拿到结果卡（`SMART_DATA_DISPLAY_SQL`）。**不看 LUDP**，于是"解释一句没权限、一个数据没给"被算作 Success。

| 方案 | 改动 | 影响 |
|---|---|---|
| A 源头修 | Spark 侧 LUDP 拒绝时上报 `status=failed` / `answer_status=no_result` | 最干净，跨仓库，需等 Spark 发版 |
| B 平台口径 | `FINAL_SUCCESS_SQL` 加 `AND NOT LUDP_PERMISSION_SQL` | 两轮都变 Failed；历史 `Success · No LUDP permission`（近 180 天 37 条）统计口径整体变化，成功率下降，需业务确认 |
| C 仅展示 | 列表对 LUDP 行把 Final result 显示为 `No access` / `Blocked`，不改数据 | 零数据风险，但两个字段语义仍分离 |

建议：**C 先止损 + A 排期**；B 会整体改动历史报表口径，不应单方面执行。

## 四、MR !578：注解 SQL 迁 XML 能否避免这类 mapper 解析失败

结论：**能消除本次根因的三要素，但不是零风险，且覆盖面不全。**

本次 bug 同时成立的三个条件，XML 化后都不再成立：

| 条件 | 注解 SQL 现状 | 迁 XML 后 |
|---|---|---|
| 手工拼 Java 字符串 | 有，编译期无检查 | SQL 在 XML，IDE 即时校验 |
| 同一份 SQL 被两种上下文复用（`<script>` / 非 script） | 有（`CANDIDATE_SELECT` 同时用于 selectCandidates 与 findCandidateByTurn） | 只有一套解析上下文 |
| 常量里混进只对一种上下文合法的写法（裸 `<`） | 有（`RECOVERED_PLAIN_SQL` 拼进 script） | 全项目一套转义规则 |

已落地写法：permission 包 6 个 XML 共约 950 行，统一 `&lt;`/`&gt;`（`&lt;` 5 处、`&gt;` 4 处），**0 处 CDATA、0 处 `<script>`**。不用 CDATA 是刻意的——避免"CDATA 内不能嵌套 CDATA"与两种转义风格混用。

反例必须说清：XML 里写裸 `a.x < b.y` 一样会导致 mapper 解析失败、服务起不来。差别只是把"运行期偶发的字符串拼接陷阱"降级为"写时 IDE 报错、启动时统一暴露"。

覆盖面（MR 描述自带表格）：permission 84 条已迁；**chatbi 188 + 58 写操作未迁**；usagefeedback/notification/jira 等 26+12、11+11、9+9 未迁。本次出事的 `ChatbiReviewJiraMapper` 就在 chatbi 包，仍需 !587（`fix/review-mapper-no-script -> admin-release`，1 文件，merge_status=can_be_merged）修复。"以后不再发生"要等 chatbi 那 188 条迁完。

## 五、三条补强建议（低成本，尚未落地）

1. 写进"持久层约定"作为硬性禁止项：DAO 接口禁写 `@Select/@Insert/@Update/@Delete` 的 SQL（极简单表查询除外）；禁 `<script>`，动态 SQL 一律进 XML；XML 内比较运算符统一 `&lt;`/`&gt;`，禁 CDATA。
2. **Mapper 全解析冒烟测试**——唯一能进 CI 拦住这类错误的办法。注解 SQL 与 XML 错误都只在启动时暴露，现在生产/测试都是部署完才发现服务起不来（上周 1.1.129、本次 3520 皆如此）。不连库即可：

```java
Configuration cfg = new Configuration();
for (String xml : mapperLocations) new XMLMapperBuilder(...).parse();
for (Class<?> dao : daoClasses) new MybatisMapperAnnotationBuilder(cfg, dao).parse();
```

3. 下一批优先迁 chatbi，尤其 `ChatbiEventQueryMapper`（1600 多行、几十个 SQL 常量互相拼接，同类问题风险最高）。

## 六、⚠️ MR !578 当前不可合：V1.1.130 三方撞号

- `admin-release` 已有 `V1.1.130__Seed_service_portfolio_kpi_cards.sql`（当日 18:xx 由 !585 合入）
- !578 分支上是 `V1.1.130__Optimize_admin_control_permission_query.sql`

两者并存会重演 1.1.129：Flyway resolve 阶段抛 `Found more than one migration with version 1.1.130`，服务起不来。MR 描述里"确认当前最大版本为 1.1.129，故顺延为 V1.1.130"写于 !585 改名之前，已过期。

处理建议：把该文件改名为 **`V1.1.131__Optimize_admin_control_permission_query.sql`**（SQL 内容不动），同步更新描述里的版本号说明，并复核是否存在第三处占用（`git ls-tree`）。

注：既有文档 `64ae3942-bdb8-4694-bb7e-9ef24b218c19` 记录的迁移号同为 `V1.1.130__Rework_chatbi_auto_review_rule.sql`，与上述占用存在冲突风险，落地前必须重新核对当前 `admin-release` 的 V1.1.12x/13x 占用表。

## 七、未决

- 状态口径方向 A / B / C 未定（用户尚未选择）。
- !578 版本号改名与描述更新未执行。
- 迁移约定三条、Mapper 全解析冒烟测试均未落地。

## 关键路径

- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java`（NUM_* / SMART_DATA_DISPLAY_SQL / LUDP_PERMISSION_SQL / FINAL_SUCCESS_SQL / RECOVERED_SQL / STATUS_CATEGORY_SQL）
- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/jira/ChatbiReviewJiraMapper.java`（本次裸 `<` 事故点，IMR !587 修复对象）
