---
id: "0496ed04-ae74-4b0c-b316-fc1f5a739e3a"
title: "SSG Ops Agent 生产环境月活与 token 消耗统计口径（含可复用 SQL 与排除规则）"
description: "生产库 ssg_ops_agent 的月度活跃用户/提问次数口径（query_session 真实提问、ssg_chatbi_internal_user 62 账号与巡检三层排除、server_env 与重复行坑），以及 agent.model_token_ledger 的 token 统计入口、UTC 时间与 has_usage 规则、10 月 6.58 亿归属拆分（巡检 38.1%、外部 4.2%）。"
status: "active"
created_at: "2026-11-18T03:35:45.637Z"
updated_at: "2026-11-18T03:35:45.637Z"
content_hash: "13d2a347ab7dab994502e9b4a1330904089ce7ab73bd2e660406b3fbe732e8e8"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java"
session_ids:
  - "mnemon-run-v1-8ea56bdf-dd33-40e4-8366-bd8936316858"
memory_body_ids:
  []
---

# SSG Ops Agent 生产环境用量统计口径：月活（人员+提问次数）与 token 消耗

面向两个反复出现的取数需求：月度活跃用户与提问量、全平台 token 消耗。数据源统一为生产库 `10.62.79.215:5432/ssg_ops_agent`（只读账号 `p_renxp4`，全部查询仅 SELECT）。同一 mapper 的 source 判定口径见 `6c0e9cea-ef5e-4642-bf38-1aba552f7258`（Source report 七态）与 `25fb29f1-f532-44dc-b110-618cf4796fdc`（轮次级改造）。

## 第一部分：月活与提问次数

### 口径定义

- 事件表：`public.ssg_chatbi_event`
- **1 次提问** = 一条 `query_session` 事件且必须是"真实问题"：剔除 clarify 答复轮（content 形如 `User has answered your questions…`）和未回答轮
- **活跃人员** = 当期有提问的去重 `user_id`

业务口径的三层排除（对应看板"含内部员工"勾选框默认不勾）：

| 排除项 | 具体对象 | 依据 |
|---|---|---|
| 内部团队账号 | `ssg_chatbi_internal_user` 中 62 个 `enabled` 的 itcode | 看板默认排除 |
| 巡检流量 | `ssg_ai_probe_run` 内会话、`properties.trigger='probe'` | AI 问数自动化巡检 |
| 非提问轮 | clarify 答复轮、未回答轮 | 避免"点选项即一轮" |

代码出处：`smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiEventQueryMapper.java:879-881` —— `includeInternal` 为空或 false 时追加
`NOT EXISTS (SELECT 1 FROM ssg_chatbi_internal_user iu WHERE iu.enabled AND lower(iu.itcode) = lower(user_id))`。

**"内部人员"特指内部分名单，不是"联想员工"泛指。** 62 个账号按组分布：未匹配 HC 22（开发团队 `renxp4`/`zhangwqp5`/`panglei2` 等）、Data governance & enablement 10、DaaS & Channel & Data Foundation 10、Upsell Ops 9（含 EMEA Ops）、TSD 4 / IT 3 / IDG&BMS 2 / LILO 2。名单手工维护，两个方向的偏差都要留意：新人未入库会被计入业务口径；Upsell Ops 9 人若本身是统计对象则被误排除。

### 结果（截至 2026-11-18）

| 时间窗口 | 活跃人员（全量） | 提问（全量） | 活跃人员（业务） | 提问（业务） |
|---|---|---|---|---|
| 近 30 天 10-20~11-18 | 123 | 5,157 | 72 | 518 |
| 11 月至今 11-01~11-18 | 92 | 2,199 | 53 | 312 |
| 10 月整月 | 131 | 7,904 | 80 | 556 |

业务口径人均 7.2 次/人/月；11 月按当前节奏折算整月约 53 人 / 530 次，与 10 月基本持平。

近 30 天访问漏斗（`ssg_user` → 登录 → 浏览 → 问数）：注册 285 → `login_result` 149 → `page_view` 146 → `query_session` 124 → 有效真实提问 123；会话数 3,927。提问集中在 `sdq_tcv_booking`（409 次/65 人）、`ssg_pbi_recommend`（68 次/37 人）、`knowledge-quiz`（41 次/10 人）。

业务口径按日：11 月工作日每天 5~12 人 / 13~49 次，峰值 11-05（12 人/49 次）与 11-03（12 人/48 次），周末几乎归零；自然周提问量 101 / 105 / 171 / 103，整体平稳。外部最活跃 `mhruska1` 48 次、`szajickova` 45、`achen50` 34、`bho5` 25。

### 两个必须提醒的坑

1. **全量数字被内部团队用量严重抬高**：近 30 天 5,157 次真实提问中 4,639 次（90%）来自那 62 个内部账号，`longyu3` 一人 2,284 次——分散在 2,231 个会话却只用 110 个不同问题，是自动化跑批/回归而非真人提问。对外汇报应给业务口径（72 人 / 518 次），并说明差异只来自"含内部员工/含巡检"两个开关。
2. **不要按 `server_env` 过滤**：较早埋点没有该字段，5,351 条 `query_session` 是 `source=backfill` 回填、`server_env` 为空，但 `event_time` 保留原始时间、按天分布正常且同属这套生产库，必须计入否则少算一半。另外同会话同问题有 97 组 210 行重复（约 4%），按看板"会话记录"口径未去重，若内部另有去重规则数字会更小。

更稳健的替代方案（如需区分"自动化"与"真人"）：不靠人事名单，改按行为筛——同一 itcode 单日提问超过 N 且去重问题数远小于提问数。

## 第二部分：token 消耗统计

### 统计入口：`agent.model_token_ledger`

agent 服务每次调用大模型写一行（约 5.5 万行，自 2026-09-23 起）。字段：`call_id / receipt_token / tenant_id / owner_user_id / session_id / executing_session_id / turn_id / scene / model / provider / started_at / finished_at / status / has_usage / usage_source / input_tokens / output_tokens / total_tokens / cached_input_tokens / cache_miss_input_tokens / cache_creation_input_tokens / provider_request_id / legacy`。token 数来自 Azure 回传 usage（`usage_source=provider_usage`），非估算。

可复用 SQL：

```sql
-- 10 月全平台
SELECT sum(total_tokens) tt, sum(input_tokens) it, sum(output_tokens) ot,
       sum(cached_input_tokens) hit, count(*) calls,
       count(DISTINCT owner_user_id) users
FROM agent.model_token_ledger
WHERE started_at::timestamptz >= '2026-10-01 00:00:00+08'
  AND started_at::timestamptz <  '2026-11-01 00:00:00+08';
```

三个必守规则：

- `started_at` 是 **UTC 文本**，必须先 `::timestamptz` 再按 `Asia/Shanghai` 取月界，否则错 8 小时
- `total_tokens = input_tokens + output_tokens`；`cached_input_tokens` 是 input 的子集，**不可再加总**
- 只统计 `has_usage=1` 行（10 月无 token 的仅 100 多条 cancelled/failed）

### 10 月单月结果

| 指标 | 数值 |
|---|---|
| 总 token | **658,057,754（6.58 亿）** |
| 输入 / 输出 | 639,142,493（97.2%）/ 18,915,261（2.8%） |
| 缓存命中 | 255,706,931（占输入 40.0%）；按未命中 383,405,876 计费 |
| 调用次数 / 账号 | 36,578 / 101 |
| provider | 全部 `azure_openai` |

归属拆分：

| 类别 | token | 占比 | 调用 |
|---|---|---|---|
| 自动化巡检（`ssg_ai_probe_run` 会话） | 250,789,935 | 38.1% | 14,982 |
| 真人会话 | 407,267,819 | 61.9% | 21,596 |
| 内部团队 50 人（含巡检） | 630,568,045 | 95.8% | 35,214 |
| 外部业务用户 51 人 | **27,489,709** | **4.2%** | 1,364 |

消耗 top：`longyu3` 3.25 亿（占全月 49%，巡检跑批载体）、`zhangqp5` 5,300 万、`panglei2` 2,725 万、`zhangyong5` 2,239 万（均为内部账号）。外部最大 `szajickova` 401 万、`mhruska1` 316 万、`bho5` 306 万。

按模型：`gpt-5.4` 536,606,082（81.5%，问数主链路 scene=main）、`gpt-5.6-luna` 87,670,745（13.3%，**全部是 `sdq_knowledge_admission` 知识库准入批处理**，3,993 次，不是用户问答）、`gpt-5.4-mini` 28,900,140（4.4%，澄清抽取/标题生成）、`gpt-5.5` 4,880,787（0.7%）。

按场景：`main` 5.60 亿（85%）、`sdq_knowledge_admission` 8,767 万（13.3%）、`final_response` 548 万、`sdq_clarification_retrieval_selection` 285 万、`sdq_clarification_extraction` 42 万、`critic` 105 万、`dream_extraction` 103 万、`compaction` 2.4 万、`generate_session_title` 1.4 万。

时间分布：10-01~10-11 每天 470~640 万且几乎全是巡检（真人仅几千到 100 万）；10-12 起真人上台阶（2,827 万），10-13 全月峰值 5,888 万；此后工作日 2,000~4,800 万，周末 1,200~2,000 万且真人几乎归零。日均 2,123 万。

### 三个提醒

1. **9 月不可比**：表 9-23 才建，9 月仅 8 天（1,648 次 / 2,569 万），别做月度环比。11-01~11-18 为 17,543 次 / 3.60 亿，日均 2,000 万，与 10 月基本持平。
2. **巡检占 38%**：用于成本分摊或对客汇报时按"真人 4.07 亿 / 外部客户 2,749 万"两档给，别直接甩 6.58 亿。巡检判定仅按 `ssg_ai_probe_run.session_id` 命中，有 687 个巡检会话在 ledger 中无调用记录（可能未调模型），实际巡检占比只多不少。
3. **`ssg_chatbi_event.properties.token_total` 不能做总量**：那是会话级汇总、覆盖不全（10 月仅 2,168 轮 / 5.85 亿，约 27% 轮次写入），只能交叉核对；与 ledger 差 2,500 万，正是后台 scene 与部分缺报之和。
