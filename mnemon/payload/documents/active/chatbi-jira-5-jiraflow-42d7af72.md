---
id: "42d7af72-3dd9-4757-9a98-2475a21d93a8"
title: "ChatBI 评审判错自动建 Jira 单：第 5 条 JiraFlow 链路方案分析（待确认）"
description: "复用现有 JiraFlow 自动建单机制接入「评审 WRONG」链路：定时扫描而非同步建单、独立台账表 ssg_chatbi_review_jira、开关放 JiraPanel 的技术理由（page-save 全量提交互相覆盖）、指派人默认 longyu3 且不回退 jira.assignee、三项待确认默认（仅人工单 / 已有单加评论 / 7 天回看）与改动面清单。"
status: "active"
created_at: "2026-11-16T09:42:11.972Z"
updated_at: "2026-11-16T09:42:11.972Z"
content_hash: "bc764b8098cd39dec8c8509432ab29dc2b8864b325df97d09f6f9ce994b7b572"
source_paths:
  - "src/views/Setting/reports/components/QuestionReviewDrawer.vue"
  - "src/views/Setting/reports/user-logs/api.ts"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiQuestionReviewMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiQuestionAutoReviewService.java"
session_ids:
  - "mnemon-run-v1-1afdc191-ebe7-44ad-9e30-715fb9c8f927"
memory_body_ids:
  []
---

# ChatBI「评审判错 → 自动建 Jira 单」方案分析（第 5 条 JiraFlow 链路）

状态：**仅完成分析，用户尚未确认，未开始实现。** 用户要求（检查点 event 2186）：「然后不是有个jira建单功能吗 review错误的要自动建单 并且要在合适的地方加个设置开关。 你分析下」；补充（event 2214）：「这个的指派人 也得能设置 默认是longyu3」。

关联文档：自动评审判定口径与 backfill 修复见 **2b13c2e2-62ad-49c9-b0f2-5bb15637a132**；评审三态/归因表立项见 **750e10a9-15ec-4ab0-9295-1f45e6d90bfd**、留痕见 **6353a23e-5302-4b9c-9301-049e028565d3**。

## 1. 结论：不需要新建机制，接第 5 条链路

现有 `probe/service/JiraFlow.java:11` 已定义 4 条链路（Inspection / Dislike feedback / Issue & Request / Failed query），开关、指派人、幂等台账、限流、重试全部现成。把「评审判错」作为第 5 条链路 `REVIEW_WRONG` 接入即可。

| 能力 | 现成实现位置 |
|---|---|
| 总开关 `jira.auto-create.enabled`（缺省开） | `ProbeSettingService.java:527` `jiraAutoCreateEnabled()` |
| 链路开关 `jira.flow.<flow>.enabled`（缺省关） | `ProbeSettingService.java:532` `jiraFlowEnabled(flow)`、`:541` `jiraFlowKey()` |
| 放行判定 = 总开关 && 链路开关 | `ProbeSettingService.java:537` `jiraAutoCreateAllowed()` |
| 每链路独立指派人 | `ProbeSettingService.java:551` `usageFeedbackAssignee()` / `:556` `dislikeAssignee()` / `:561` `probeAssignee()` / `:571` `failedQueryAssignee()` |
| 缺省指派人常量 | `ProbeSettingService.java:220` `DEFAULT_FAILED_QUERY_ASSIGNEE = "longyu3"` |
| 开关拦截点在建单内部（被拦返回 null，不算失败） | `ProbeJiraClient.java:109-119` → `settingService.jiraFlowEnabled(effectiveFlow)` |
| 幂等台账 + `pg_advisory_xact_lock` | `ChatbiFailedQueryJiraMapper.java:39` |
| 定时扫描 + settle + 限流 50 + 重试 3 | `ChatbiFailedQueryJiraService.java:38-46`、`ChatbiFailedQueryJiraScheduler.java:28` |
| 台账表模板 | `V1.1.109__Create_chatbi_failed_query_jira.sql` |
| 评审表（主键同为 `(session_id, turn_key)`） | `V1.1.119` 的 `ssg_chatbi_question_review`；`ChatbiQuestionReviewMapper.java:15` 注释明写主键与失败台账完全一致 |

前端 `JiraPanel.vue:42-143` 是声明式配置：`FLOWS` 数组 + `ASSIGNEE_FIELDS` 数组，加一条链路只需各加一个对象。

## 2. 建议做法

**触发用定时扫描，不用保存时同步建单。** 理由：`ChatbiQuestionReviewService.java:86` 的 `save()` 同步建单会让保存卡 1~3 秒；自动评审 backfill 一次可插几百行（`ChatbiQuestionAutoReviewMapper.java:134` 批量 insert），同步建单会请求超时或触发 Jira 限流。流程：

```
评审 WRONG 落库 → reviewed_at 过后 N 分钟稳定期 → 扫描候选 → 逐条建单（一条一个事务）
```

建议扫描口径：

```sql
FROM ssg_chatbi_question_review r
WHERE r.status = 'WRONG'
  AND r.reviewed_at BETWEEN now() - interval '<lookback>' AND now() - interval '2 min'
  AND NOT EXISTS (台账已有 jira_key / attempts>=3 / next_attempt_at>now())
```

**台账单独建表 `ssg_chatbi_review_jira`（主键同 `(session_id, turn_key)`），不在评审表加列。** 理由：评审行会被 `ChatbiQuestionReviewService.java:149` 的 `clear()` 物理删除，单号挂在评审行上一撤销就丢，再评 WRONG 会重复开单。

**提单内容**：Summary `[Wrong review][归因] 问题原文`；Body 含用户、时间、环境、评审人、来源（HUMAN/AUTO）、归因、**评审备注**（triage 核心信息）、问题原文、session/turn、复现路径。指派人 `reviewWrongAssignee()`，缺省 `longyu3`，**不回退 `jira.assignee`**（照抄 `ProbeSettingService.java:565-574` 的逻辑：这是「谁跟进错误单」的独立约定，页面留空回到缺省值而不是被通用经办人顶掉）。

## 3. 开关放 Settings → Jira integration，不放 Review rules

```ts
// FLOWS
{ key: 'jira.flow.review-wrong.enabled', label: 'Wrong review',
  trigger: 'A reviewer marks a question Wrong', assigneeKey: 'jira.review-wrong-assignee' }
// ASSIGNEE_FIELDS
{ key: 'jira.review-wrong-assignee', label: 'Wrong review assignee',
  placeholder: 'Defaults to longyu3', hint: '...' }
```

理由（load-bearing）：
1. `JiraFlow.java:6-9` 类注释已记录过历史上「三个开关散落在不同页面、语义也不一致」的问题，收敛到一处是既定设计。
2. **技术原因**：`JiraPanel` 是 `saveMode: 'page-save'`（设置中心 registry 中 `key: 'jira'` 那条，registry.ts:107-116）全量保存，提交整个 `WRITABLE_KEYS`。若 Review rules 也写同一个 key，两个页面各自全量提交会互相覆盖 → 开关只能有唯一保存入口。
3. 指派人字段需要与其它 4 条链路并排看才一致。

折中：在 `ReviewRulesPanel.vue` 加一行只读说明「Wrong verdicts open a Jira ticket. Configure the switch and assignee in Settings → Jira integration.」

## 4. 三个待确认问题与建议默认

1. **自动评审（timeout/failed 规则判定）产生的 WRONG 要不要建单？** 建议默认只对 `review_source='HUMAN'` 建单，另加子开关 `Include automatically reviewed verdicts`（缺省关）。理由：backfill 一次几百条会把 Jira 灌满，人工确认过的错误才需要跟进。
2. **同一轮已有 failed-query 单怎么办？** 建议不建第二张，改为在那张单下加评论（附归因 + 评审备注），台账记 `adopted=SSGAI-xxx`。参照 `ChatbiFailedQueryJiraMapper.java:72` 用 `SEARCH_JIRA_KEY_SQL` 排除已建单轮次的做法。
3. **首次打开开关的回看窗口？** 建议 7 天（评审常滞后于提问），窗口外历史 WRONG 不自动建单。对照：failed-query 用 24h 防「发布当天给过去三个月批量建单」。

## 5. 改动面（估算）

| 层 | 改动 | 量 |
|---|---|---|
| 后端 | `JiraFlow` +1 枚举；`ProbeSettingService` 常量 / WRITABLE_KEYS / forPage / 链路开关 / 指派人 共 6 处 | ~40 行 |
| 后端 | 迁移 `V1.1.127__Create_chatbi_review_jira.sql`（只建表，不塞数据） | ~70 行 |
| 后端 | 新增 `ChatbiReviewJiraMapper` / `Creator` / `Service` / `Scheduler`（照 failed-query 复制改造） | ~500 行 |
| 后端 | `reviewGet` detail LEFT JOIN 台账返回 `jiraKey`/`jiraUrl`，抽屉顶部显示「已提单 SSGAI-xxx」 | ~20 行 |
| 前端 | `JiraPanel.vue` FLOWS + ASSIGNEE_FIELDS 各一项；`ReviewRulesPanel.vue` 加指引说明；`QuestionReviewDrawer.vue:183` 附近显示单号 | ~40 行 |

无新路由、无新页面、不改评审三态逻辑。分支约定：后端 `origin/admin-release` → MR `admin-release`，前端双 MR。

## 6. 同轮附带产物（与本主题无关，仅记录结论）

设置页关闭按钮「×」视觉偏左上：根因是字符 `×`（U+00D7）字形在 em 框内偏基线以上，`align-items: center` 只能居中排版框。改为 CSS 两条 45° 线画图标（`.qr-close-icon::before/::after`，13×13、1.5px、`translateY(-50%) rotate(±45deg)`），同时删掉 `.qr-close` 的 `font-size: 22px; line-height: 1`。commit `96e1b61b`；前端两条分支已推（`!1051` / `!1052` 自动带上），后端 `!558` 未动。
