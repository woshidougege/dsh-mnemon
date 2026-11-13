---
id: "38cfa2fb-9dd3-4b6d-bf04-a86a0e7f0d3d"
title: "ssg_sales_ops：评审抽屉改两栏工作台布局（QuestionReviewDrawer，MR !1028）"
description: "QuestionReviewDrawer.vue 从 680px 单列改为 min(1080px,96vw) 两栏工作台：Element Plus drawer teleport 导致必须用非 scoped 外壳样式块 + 唯一前缀、el-drawer__header 默认 flex 裁切右边界的坑、断点与样式变量、MR !1028 提交验证信息及未完成的三处评审筛选前端与待决策事项。"
status: "active"
created_at: "2026-11-13T21:05:56.984Z"
updated_at: "2026-11-13T21:05:56.984Z"
content_hash: "cb779e97823fc481229786cde7dbeb33cee73f3ab169229110a75a94d5942d84"
source_paths:
  - "src/views/Setting/reports/components/QuestionReviewDrawer.vue"
session_ids:
  - "mnemon-run-v1-bfa9e41a-1aff-41e3-88a5-807f8177aa11"
memory_body_ids:
  []
---

# ssg_sales_ops：评审抽屉改两栏工作台布局（QuestionReviewDrawer，MR !1028）

## 背景与目标

- 目标文件：`src/views/Setting/reports/components/QuestionReviewDrawer.vue`（改造前为 680px 单列抽屉）。
- 用户给出了一版设计稿，要求从单列改为两栏工作台：左栏「Question + Context」，右栏「Your verdict + Discussion/History」。
- 硬约束：**逻辑零改动**——保存 / 清空 / 评论 / 历史记录、Open session detail 跳转与 turnId 定位闪烁全部保持原样，本次只动布局与样式。

## 实现要点

- 抽屉尺寸改为 `size="min(1080px, 96vw)"`，`direction="rtl"`，`:show-close="false"` + `append-to-body`，关闭按钮自绘在 `.qr-head-side` 内。
- 头部 Current verdict 由原来的 168px 固定卡片改成紧凑 chip（`.qr-verdict-chip`）回到正常流，关闭按钮排在 chip 右侧，不再贴右边界绝对定位。
- 主体拆 `.qr-grid` → `.qr-main` + `.qr-side`，右栏 `.qr-side-inner` 用 sticky 跟随滚动；Context 卡片从侧栏移到主栏（按设计稿）。
- 响应式：≤1040px 折单栏（`.qr-ledger` 与 `.qr-verdict` 都改单列、`.qr-ledger-row.is-wide` 占满 `1 / -1`），≤760px 头部竖排（`.qr-head` flex-direction: column，右侧留 60px 给关闭按钮）、抽屉宽度 100%。
- 无障碍/动效：给 `.qr-verdict-btn / .qr-tab / .qr-post / .qr-save / .qr-open / .qr-clear / .qr-text-btn` 加 `:focus-visible` outline；`@media (prefers-reduced-motion: reduce)` 关闭过渡。

## 两个 Element Plus 坑（可复用）

1. **el-drawer 外壳被 teleport 到 body，scoped 样式覆盖不到**。header / body / footer 均由 Element Plus 渲染，因此必须单独写一个非 scoped 的 `<style lang="less">` 块，并用组件独有的 `.question-review-drawer.el-drawer` 前缀限定作用域，否则会污染全局。
2. **`.el-drawer__header` 默认 display:flex 会裁掉右边界**。`.qr-head` 作为唯一 flex 子项按内容宽度撑出容器，右边界连同关闭按钮一起被裁；解法是把 header 改 `display: block`，宽度交给 `.qr-head` 自己控制。实测诊断数值：`innerWidth=500 scrollWidth=499`，此时 `HTML w=485 right=485 sw=499 cw=485` —— 容器宽仍然大于可见宽度。

## 样式分层

- 组件 CSS 变量在 scoped 块与非 scoped 外壳块里**各声明一份**（teleport 后的节点不在 scoped 作用域内）：`--qr-canvas #fbfaff`、`--qr-surface #ffffff`、`--qr-soft #f4f3fc`、`--qr-line rgba(93,84,150,.08)`、`--qr-ink #33304a`、`--qr-txt-2 #5f5a78`、`--qr-muted #8d879b`、`--qr-muted-2 #9a95b4`、`--qr-brand #948ae8`、`--qr-brand-deep #6f63c9`、`--qr-correct #15803d`、`--qr-wrong #c0344f`、`--qr-amber #d9a95e`。
- 主要结构类：`.qr-head / .qr-head-main / .qr-head-question / .qr-head-side / .qr-verdict-chip / .qr-body / .qr-grid / .qr-main / .qr-side / .qr-side-inner / .qr-block / .qr-ledger / .qr-verdict / .qr-activity / .qr-footer`。

## 提交与验证

- 分支 `feature/review-drawer-redesign`（从 origin/develop 切出），提交 `8323a80b feat(review): 评审抽屉改两栏工作台布局`，1 file changed，331 insertions(+) / 258 deletions(-)。
- MR **!1028** → develop：https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1028
- 验证：`npm run type-check` 通过；eslint 0 error，保留 4 条既有 `@typescript-eslint/no-explicit-any` warning（170:15、261:15、276:15、294:15）；`vite build` 通过（59.72s）。
- 本地按组件样式渲染三档宽度截图核对（2160x1800 / 1800x1800 / 840x1800），产物在 `ssg_sales_ops-wt-review-polish-master/.tmp/shot-drawer-v2-*.png`。

## 交接与待决策

- 设计稿主栏含 Answer / 图表 / 工具调用区块，但列表行只有问题文本、没有回答全文；要让主栏真正展示回答，需按 `sessionId + turnId` 再拉一次会话详情，会让本 MR 变大——是否接入待用户确认。
- 是否另出一个 master 版 MR（沿用既往双 MR 做法）待用户定。
- 三处评审筛选：后端已改完并编译通过（Feedback 接口已返回评审统计、Issue & Request 的四个接口已接参数），前端尚未动——`FeedbackView` 计数回填、Issue & Request 页四个多选筛选、导出参数。

## 相关文档

- `6f91417f-7e38-40d3-8d7e-e91d9a9ee8d2`：同一文件的 master→develop 假冲突诊断与「从 origin/develop 重切 + cherry-pick」规程（!997→!1001）。该文档记录 develop 已有一次抽屉重做提交 `b1589356 feat(review): redesign the review drawer as a case file`，改同一文件前应先确认 develop 侧最新版本。
