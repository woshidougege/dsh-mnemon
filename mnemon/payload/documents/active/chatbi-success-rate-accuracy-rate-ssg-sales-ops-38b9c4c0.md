---
id: "38b9c4c0-0988-4206-9198-11f20587ae61"
title: "ChatBI 报表页 Success Rate → Accuracy Rate（人工评审口径）设计方案与数据模型（ssg-sales-ops）"
description: "把用户日志报表卡片的成功率改成人工评审口径的 Accuracy Rate：概念拆分、卡片与表格改法、ssg_chatbi_question_review 建表 SQL、3 个接口、抽样偏差等坑、P1–P3 分期与 6 项待拍板决策。"
status: "active"
created_at: "2026-11-12T10:27:28.174Z"
updated_at: "2026-11-12T10:27:28.174Z"
content_hash: "c7c840199b3337be483a26361d97a4db8b81e93b5118d15e0f8ca4bbba5ed998"
source_paths:
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue"
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/api.ts"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/controller/ChatbiEventReportController.java"
  - "smartop-service/admin/src/main/resources/db/migration"
session_ids:
  - "mnemon-run-v1-3ca8ecc1-2250-454f-9b05-5ad4515ad309"
memory_body_ids:
  []
---

# ChatBI 报表页：Success Rate → Accuracy Rate（人工评审口径）设计方案（ssg-sales-ops）

来源：用户要求把用户日志报表页卡片上的百分比从"系统自报成功"改成按人工 review 判定，并新增按 review 结果展示的第二块图/表；准确度 = 正确数量 ÷ 已 review 的问题数量。以下为该需求在设计阶段形成的方案（尚未开发，等待用户拍板）。

## 1. 需求本质

不是改标题，而是**判断依据更换**：现有百分比 `(total - failed)/total` 由后端按 `query_session` 自算，口径是"agent 有没有报错"，不等于答案是否正确。改为 Accuracy Rate 后，分子 = 人工标记 Correct 的数量，分母 = 已评审数量（未评审不进分母）。因此必须先建立评审能力，否则卡片无数据可算。

## 2. 现状证据（改造点）

- 卡片位置：`ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue:166-175`
- 表格：`ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue:314-400`，现有 `Final result`(Success/Failed) 与 `Failure category` 均为机器判定
- 前端接口：`ssg_sales_ops/src/views/Setting/reports/user-logs/api.ts:95` → `POST /api/admin/chatbi/events/report/queries/summary`
- 后端：`smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/controller/ChatbiEventReportController.java:68`
- 迁移目录：`smartop-service/admin/src/main/resources/db/migration`，现有最新编号 `V1.1.99__Add_usage_issue_feedback_attachment_jira.sql`
- 可复用能力：`ctx.openSessionDetail(sessionId, title, turnId)` 抽屉可在评审弹窗内展示会话详情

## 3. 概念拆分（设计地基）

| 概念 | 判定方 | 现状 | 落地 |
|---|---|---|---|
| Final result / Success·Failed | agent 自报 + 前端结果卡 | 已有 | 表格保留不动 |
| Review verdict / Correct·Wrong | 人工评审 | 无 | 新增 |
| Accuracy Rate | Correct ÷ 已评审 | 无 | 卡片主数字 |

并排后可派生高价值指标"**假成功率**"= agent 报 Success 但人工判 Wrong ÷ 已评审数；配合归因标签可回答"假成功是口径理解错还是路由错"。

## 4. 卡片改法

```
Accuracy Rate                          ← 主数字换人工口径
95.0%                                  = Correct ÷ Reviewed（未评审不入分母）
Correct 114 / Reviewed 120             ← 主说明，明确分母
Coverage 24% · Pending 380             ← 覆盖率，防"只挑好答的评"
Agent-reported success 95.0% · Failed 25   ← 旧口径降级为参考行
```

- 无任何评审时主数字显示 `—` + `No reviews yet`，不回落 0%/100%（与 Service Portfolio 空态约定一致）
- hover title 写公式：`Accuracy = questions marked Correct ÷ questions reviewed; Pending not counted.`
- **必须同时显示 Coverage**，否则早期 20 条样本会冒充全量结论

## 5. 表格新增（插在 Final result 右侧，即 QueryView.vue:389 之后，便于与机器判定横向对比）

| 新列 | 取值 |
|---|---|
| Review status | Pending(灰) / Correct(绿) / Wrong(红)；缺行即 Pending，不预生成数据 |
| Attribution tag | 口径理解错 / 路由错 / 数据缺失 / 权限不足 / 展示错误 / 其他；**仅 Wrong 必填**，Correct 置灰 |
| Reviewer | 名字 + 评审时间 tooltip |
| Action | Pending→评审；已评审→查看，可重新评审 |

评审弹窗需展示"问了什么、路由到哪张报表、出来什么数、agent 走了哪几步"，复用 `ctx.openSessionDetail`，内部放 Mark Correct / Mark Wrong 并可直接跳下一条。建议 P1 即带**评审队列模式**：筛选 `Review status = Pending`，一屏一条，键盘 `1=正确 2=错误 3=跳过`，标完自动下一条（表格式评审吞吐不足，覆盖率上不去）。

## 6. 数据模型（迁移只建表，不塞数据）

新表 `public.ssg_chatbi_question_review`，一行 = 一个问题的最新结论；迁移编号接 `V1.1.100__Create_ssg_chatbi_question_review.sql`：

```sql
CREATE TABLE IF NOT EXISTS public.ssg_chatbi_question_review (
    question_key   VARCHAR(160) PRIMARY KEY,   -- query_id；缺失时 session_id#turn_id
    query_id       VARCHAR(64),
    session_id     VARCHAR(64) NOT NULL,
    turn_id        VARCHAR(64),
    status         VARCHAR(16) NOT NULL,       -- CORRECT | WRONG
    attribution_tag VARCHAR(32),               -- Wrong 时必填
    remark         VARCHAR(512),
    reviewer_itcode VARCHAR(64) NOT NULL,      -- 服务端从登录态取，不信前端
    reviewer_name  VARCHAR(128),
    reviewed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ssg_chatbi_qreview_status ON public.ssg_chatbi_question_review (status, reviewed_at DESC);
CREATE INDEX idx_ssg_chatbi_qreview_reviewer ON public.ssg_chatbi_question_review (reviewer_itcode, reviewed_at DESC);
```

刻意选择：
- **不建 PENDING 行**：无评审记录即 Pending；提问量每日增长，预插行既违反"迁移不塞数据"也无必要
- 归因标签用 code + 前端常量（参考 `constants.ts`），加标签不改表；需集中运维再上字典表
- 重新评审 = upsert 覆盖 status/attribution_tag/remark/reviewed_at；要留痕再加 `ssg_chatbi_question_review_log`（Phase 3）

## 7. 接口

1. `POST /queries`（列表）→ LEFT JOIN 评审表，多返回 `reviewStatus / attributionTag / remark / reviewerName / reviewedAt`，默认 Pending；不要在循环里逐条查
2. `POST /queries/summary` → 多返回 `review: { reviewed, correct, wrong, pending, accuracyRate, coverageRate, byTag: [...] }`；**WHERE 条件必须与列表完全一致**（现有 summary 已如此），否则卡片与表格对不上必被挑错
3. `PUT /queries/{questionKey}/review` body `{ status, attributionTag, remark }` → upsert，返回更新后行 + 新 review 汇总，前端就地刷新
4. 筛选栏加 `Review status`（多选，默认空）与 `Reviewed by = me`

权限：评审为写操作，`reviewer_itcode` 必须服务端取；按钮按权限组显示（参考 `kmverse.feedback.triagers` 的做法，这里设 `chatbi.reviewers`），不放开给所有可读表的人。

## 8. 已知坑

1. **抽样偏差**：只有被点踩/可疑的问题会被评，Accuracy 天生偏低 → 写清"已评审样本上的准确率"，或固定抽样（每天随机 N 条 + 全部 Failed 必评），卡片标 `sampled`
2. **分母漂移**：今天 120 条 95.0%，明天加评 30 条变 91.2%，属口径性质非 bug；趋势图需标注"累计已评审"线
3. **同一问题反复问**：可借 `ssg_chatbi_question_cluster` 按簇评审（一次覆盖同簇所有提问，Phase 2），Accuracy 仍按提问数加权
4. **语言**：该页现为英文（`Final result`/`Success`），默认英文文案 `Accuracy Rate / Review status / Attribution tag / Reviewer / Pending·Correct·Wrong`，中文对照放 tooltip

## 9. 分期

- **P1（可上线）**：迁移 + 评审表 + 3 接口 + 卡片换口径（含 Coverage/Pending）+ 4 列 + 评审弹窗 + Pending 筛选
- **P2**：评审队列（键盘流）+ 抽样策略 + 归因标签分布图 + 假成功率 + 按 review 状态导出
- **P3**：重新评审留痕/审计、双人盲评一致性（κ）、按簇评审

## 10. 待用户拍板（决定 P1 范围）

1. 粒度按"一次提问(query_id)"一行，还是同一 session 多问合并为一条评？
2. 评审权：固定 itcode 名单建权限组，还是管理员都能评？
3. 归因标签除「口径理解错」还要哪些？Wrong 且选"其他"时 remark 是否必填？
4. 分母是否加覆盖率门槛（如 reviewed < 30% 时数字变灰 + 样本不足提示）？
5. 旧的成功率留卡片第二行还是收进 tooltip？
6. 本页走英文文案还是切中文？
