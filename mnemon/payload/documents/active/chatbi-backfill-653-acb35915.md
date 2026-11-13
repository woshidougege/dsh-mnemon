---
id: "acb35915-0667-44bf-a5e9-8dac79c502d8"
title: "ChatBI 报表评审：筛选返回空 + backfill 漏评审 653 条（生产排查与三项待定改法）"
description: "281 条评审全为 system:auto 导致 Review source=Human 恒空、Reviewer 下拉由 userOptions 映射无法筛 Auto rule；漏评审实为 653 条 backfill 被 Mapper:83 排除，live 37 条逐类核查非真漏；含判定常量行号与三项待拍板改法。"
status: "active"
created_at: "2026-11-13T18:25:33.176Z"
updated_at: "2026-11-13T18:25:33.176Z"
content_hash: "e7d6d3419a9b22754f21b58576ec555a73b99b8f9891ad25009aab569b1c22cb"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-77c3905f-c85b-4123-acc9-151d35878c0f"
memory_body_ids:
  []
---

# ChatBI 报表：评审筛选返回空 + backfill 漏评审 653 条（生产数据排查）

来源：用户在用户日志报表页提出两点疑问——(1) Review status / Review source / Reviewer / Attribution 等筛选似乎不起过滤作用（选 Human 直接 0 条）；(2) 某条失败轮次（user=kmurphy2、id=97377、2026-11-09 23:16:58、failed_step=revise_query）一直显示 Pending，疑似漏评审。以下为连生产库 `ssg_ops_agent` 与代码检索得到的结论。

## 1. 结论一：筛选链路没坏，是数据里根本没有 HUMAN 评审

生产 `ssg_chatbi_question_review` 全量只有一个评审主体：

```
reviewer_itcode | reviewer_name | review_source | cnt
system:auto     | Auto rule     | AUTO          | 281
```

281 条评审 100% 由自动规则生成，**HUMAN 记录 0 条** → Review source=Human 必然查询为空；前端传参、后端校验、SQL 条件三步均正常。

- `ChatbiEventReportService.java:851-864` `normalizeReviewSource`：只保留 HUMAN / AUTO，空串与非法值一律丢成 null（全部非法 = 不筛选，不会误伤数据）。
- `ChatbiEventReportService.java:835-848` `normalizeReviewStatus`：只保留 CORRECT / WRONG / PENDING；**PENDING 不是库里的枚举值**，表示"评审表没有这一行"，由 SQL 侧 `qr.rv_status IS NULL` 判定。
- 真正的设计缺陷在 **Reviewer 下拉**：`FilterBar.vue:160-167` 的候选由 `userOptions`（字典真实用户清单，取 itcode）映射，而库里评审人只有 `system:auto`。选任何真实用户都查不到数据，且**无法筛出 Auto rule 那 281 条**。

## 2. 结论二：漏评审不是 1 条，是 653 条 backfill 数据

事件总量与失败分布：

| 类别 | 失败轮次 | 已评审 | 未评审 |
|---|---|---|---|
| backfill 回填（`properties.source='backfill'`） | 653 | 0 | **653** |
| 实时上报 | 316 | 279 | 37 |

- 全量 `ssg_chatbi_event` 8635 行，其中 backfill 5556 行；失败 969 = 653 backfill + 316 live。
- backfill 653 条永久 Pending 的根因：自动评审候选 SQL 在 `ChatbiQuestionAutoReviewMapper.java:83` 显式排除——
  `AND COALESCE(ssg_chatbi_event.properties ->> 'source', '') <> 'backfill'`
- 实时上报那 37 条经逐类核查**不是真漏**：33 条在 agent 审计里已有成功的 `smart_data_query_result` 结果卡（页面按 Final result 判定为 Success，本就不该判错）、1 条命中巡检、0 条 LUDP 权限不足。
- 复核口径（同一条 SQL 上过滤后得 total=37 / probe=1 / smart=33 / ldup=0）：`FINAL_SUCCESS_SQL`（`ChatbiEventQueryMapper.java:458-461`，status 或 answer_status=success，或命中智能数据展示）、`SMART_DATA_DISPLAY_SQL`（:428-435，查 `agent.tool_audit_logs` 且 `output_summary ? contract=smart_data_query_result/v1 + status=success`）、`LUDP_PERMISSION_SQL`（:444-449，execute 工具 succeeded 但业务返回 ludp_permission_denied）、`PROBE_RUN_EXCLUDED_SQL`（:471-473，以 `public.ssg_ai_probe_run.session_id` 为准，2026-10 后的巡检不再写 trigger 标记）、`RECOVERED_SQL`（:482+，"同轮同工具失败后重试成功"，必须按审计行时间先后判断，不能只看 steps 聚合，否则"先成功两次再失败"会被误判）、`AUDIT_SESSION_ID_SQL`（:407-409，优先 `properties.upstream_session_id`，回退 session_id）。
- 排查经验：手写 join 查询时裸 `status` 会报 `ERROR: column reference "status" is ambiguous`，必须带表别名。

## 3. 待用户拍板的三项改法（截至该次对话未决定、未改代码）

1. **backfill 是否纳入自动评审**：去掉 Mapper 里 `source <> 'backfill'` 一行即可一次补上 653 条；抽查的样例 `failed_step`、`failed_reason` 字段齐全，展示上与实时数据无差别。当初排除的原因推测是担心回填数据字段不全。
2. **Reviewer 下拉改数据驱动**：从 `ssg_chatbi_question_review` 取 distinct `reviewer_itcode/reviewer_name`，把 `system:auto` 显示为 Auto rule，使其可筛选。
3. **Review source / Review status 选项加计数，0 条置灰**：让用户在选之前就知道没有数据。

计划：三项在同一 MR 内同时改前后端。

## 4. 相关既有文档

- 750e10a9-15ec-4ab0-9295-1f45e6d90bfd：ChatBI 问数自动评审规则（V1.1.121）实施记录——"只处理最近 90 天真实问数，排除 backfill 与巡检 probe"的设计出处，以及 AUTO→HUMAN 覆盖与抑制名单语义。
- 38b9c4c0-0988-4206-9198-11f20587ae61：报表页 Success Rate → Accuracy Rate 的人工评审口径设计。
- d7382448-aaf9-4003-971f-58f1f56da61b：问数记录 Source report / Session title 粒度缺陷排查。
