---
id: "88550f5d-ceb8-4ddc-8d91-bafeec243c3f"
title: "ChatBI 人工评审「Not applicable 保存 400」排查记录：接口校验契约与线上构建核验法（ssg-sales-ops）"
description: "评审 N/A 必填 attributionTag 的 400 实测契约；用线上 chunk 特征串核验生产前端是否为新版三态的方法与压缩后 canSave 逻辑；未验证假设（详情重拉导致显示回 Pending）与待确认项。增量于交付文档 ea0c4abe。"
status: "active"
created_at: "2026-11-17T02:01:05.582Z"
updated_at: "2026-11-17T02:01:05.582Z"
content_hash: "eb16d5773d04f28102fb8b4308d6ac1a2e53a58b6c0c0fc211102332ed89c7dc"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiQuestionReviewService.java"
  - "src/views/Setting/reports/components/QuestionReviewDrawer.vue"
session_ids:
  - "mnemon-run-v1-19661b5e-0274-418a-93f2-9e03185f8a4b"
memory_body_ids:
  []
---

# ChatBI 人工评审「Not applicable 保存 400」排查记录（ssg-sales-ops，2026-11-17）

增量排查记录。评审功能的交付背景（三支 MR、Accuracy Rate 口径、建表 V1.1.119）见既有文档 **ea0c4abe-9366-4b26-964c-e4b8149e3fad**，本文不重复。
状态：现象已被两条独立证据收敛到「后端校验 + 线上构建实况」，但**根因未最终定论**——还差用户侧三点确认（见 §3）。

## 1. 评审接口实测契约
- 端点：`GET / PUT / DELETE /queries/review?sessionId=…&turnKey=…`；PUT 仅 admin，评审人取当前登录身份（`resolveItcode`）。
- `ChatbiQuestionReviewService.save`（约 86-140 行）校验：
  - `status` 必须在 CORRECT / WRONG / NOT_APPLICABLE。
  - CORRECT：服务端**强制清空** attributionTag（否则列表出现「正确 + 数据缺失」矛盾行）。
  - WRONG：attributionTag 必填且必须在 WRONG_TAGS 内，否则提示 "use NOT_APPLICABLE when the question cannot be judged"。
  - NOT_APPLICABLE：原因必填且必须在 NA_REASONS 内（`NO_PERMISSION` / `OUT_OF_SCOPE`），空串直接 400：`attributionTag is required when status is NOT_APPLICABLE`。
- 语义细节：只有结论真的变化才写 history 流水（避免重复点保存刷流水）；人工保存会 `autoReviewMapper.undismiss` 清掉「别自动评」抑制标记——人工结论优先于规则，规则变更后可重新自动评。
- GET 详情：无评审行返回 null → 前端渲染 Pending，后端不预生成数据。
- 实测：`{"status":"NOT_APPLICABLE","attributionTag":""}` → HTTP 400 上述文案；补 `NO_PERMISSION` → HTTP 200，reviewSource=HUMAN，history 记 `SAVE: "" -> NOT_APPLICABLE / NO_PERMISSION`。

## 2. 线上前端构建核验法（本次最可复用的一条）
**不要用 GitLab 分支/master 状态推断线上构建新旧**——本次据此误判过一次。
做法（只读、可复现）：
1. 拉生产首页，取入口 chunk（当时 `/assets/index-Cv3mH68Y.js`，约 1.55MB）。入口里没有评审代码，属懒加载。
2. 把入口引用的全部 `index-*.js` 子 chunk（当时 45 个）逐个下载。
3. 对每个 chunk grep 特征串：`Select a reason`、`Why is it not applicable`、`qr-attribution`、`NOT_APPLICABLE`。唯一命中 `index-5f4Ja0tf.js`（约 366KB），说明评审抽屉在该 chunk。
4. 读压缩后逻辑比对语义，而不是比对源码文本。

生产该 chunk 当时的等价逻辑（属新版三态）：
- `canSave = linkAvailable && !saving && Boolean(status) && (status === 'CORRECT' || Boolean(attributionTag))`
- 切状态时若已选 tag 不在新集合内则清空：`if (!(WRONG?WRONG_TAGS:NOT_APPLICABLE?NA_REASONS:[]).some(o=>o.value===tag)) tag=''`
- 保存必须先过 canSave：`async save(){ if (canSave) await reviewSave({sessionId,turnKey,status,attributionTag,remark}) }`
推论：走这份 UI 不可能发出 `attributionTag: ""` 的 N/A 请求 ⇒ 那个 400 只能来自旧构建页面、其他调用方、或直接调接口。

源码侧对照（master）：`QuestionReviewDrawer.vue` 的 `<select id="qr-attribution">` 含 `<option value="" disabled>Select a reason</option>`，`canSave` 同上述判断；NA 时标签文案切到 "Why is it not applicable?"。若页面缺这个占位项，浏览器会把第一个真实选项（Out of scope）显示成「已选中」，而绑定值仍是空串——这正是「看着选好了、一点就报错」的观感来源。

## 3. 未验证假设与下一步（继续排查时从这里接）
- **假设 A（优先）**：保存成功后组件 `await le()` 重拉详情；若 detail 匹配不到该 turn 的评审行，三态会被重置、显示回 Pending，用户观感等同「保存不了」，但后端其实已入库。线索：该 turn 回读的 `question` / `userId` / `queryId` 全为空，疑事件表回填时缺 `query_id`。需对生产库做只读查询确认。
- 待用户提供：① 点 Save 时按钮是否 disabled；② 400 Response 原文；③ TST 前端地址（拆同一 chunk 比对是否与生产同一份构建）。三点齐了才能判断「测试+生产同时复现」是代码问题还是两端都跑了旧构建。

## 4. 教训
- 判定线上运行的是哪份代码，唯一可信证据是线上产物本身（chunk 特征串 + 压缩逻辑），不是仓库分支或 commit 时间。
- 该 400 文案只反映「N/A 缺原因」这一层；先把契约和线上产物钉死，再谈谁该改。
