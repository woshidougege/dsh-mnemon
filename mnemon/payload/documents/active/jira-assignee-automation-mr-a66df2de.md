---
id: "a66df2de-9353-416f-bc6b-777157ca3ac1"
title: "Jira 设置页 Assignee 内联进 Automation 表格 + 校验对齐（实现落地，待提 MR）"
description: "用户拍板「在 Automation 能改 Assignee 就行」后的前端实现：JiraPanel.vue 流程表内联 assignee 输入框、删除 Routing 页签并把 page focal 说明下移、FieldDef 增加 optional 使 save() 只校验 3 个必填并修正文案；含 commit 689d1c71 / a1afa0f1、type-check 与本地预览结果、未 push 未提 MR 的待办状态。"
status: "active"
created_at: "2026-11-17T03:16:49.220Z"
updated_at: "2026-11-17T03:16:49.220Z"
content_hash: "9ef697f87a6526736e0495c2ded25765192a92cf1ac5f7143b8aa760a8d7bdb6"
source_paths:
  - "src/views/Setting/settings/panels/JiraPanel.vue"
  - "vite.config.ts"
session_ids:
  - "mnemon-run-v1-7207b7fa-8900-4d6e-b8c1-e85400df4553"
memory_body_ids:
  []
---

# Jira 设置页：Assignee 内联进 Automation 表格 + 校验对齐（实现落地，待提 MR）

日期：2026-11-17。状态：**代码已提交但未 push、未提 MR，等用户下达提 MR 指令**。

关联文档：
- **2a95b6da-13f6-48f7-ace0-52cef76ec38f** —— 上线后生产批量建单证据 + 改动前的 JiraPanel 结构（Automation 只读 Assignee / Routing 可编辑），该文把「Assignee 是否并入 Automation」「校验与文案 bug」列为待用户选择项；本文记录这两项的**决策与实现结果**。
- **cc5d103b-628e-4216-a1a6-08ef73e07f46** —— 评审 WRONG 自动建单的最终设计口径、三支 MR（!559 / !1054 / !1055）、新表 V1.1.127。

## 一、用户决策（原话）

> 「在Automation 能改 Assignee就行了 你设计的太复杂了 用户反而不会用」
> 「每一个功能提交一次 最后一块提MR 哈 MR等我命令再提」

即：不再维护独立的 Routing 页签，Assignee 直接在 Automation 流程表里就地编辑；提交粒度按「一个功能一次提交」；MR 必须等用户明确下令才推送/创建。

## 二、改动一：Assignee 内联进 Automation（commit `689d1c71`，+31/-46）

分支 `dev/renxp4`（worktree `C:\\Users\\63588\\lenovo\\ssg_sales_ops-wt-nareview`），唯一文件 `src/views/Setting/settings/panels/JiraPanel.vue`。

- Automation 流程表原只读单元格 `{{ form[flow.assigneeKey] || 'Not set' }}`（旧行 442-453）替换为内联输入框：

```html
<td class="jira-flow-table__assignee">
  <el-input v-model="form[flow.assigneeKey]" size="small"
            :placeholder="flow.assigneePlaceholder || 'Jira username'" />
</td>
```

- `FlowDef` 增加可选 `assigneePlaceholder`；五行映射不变：Inspection→`jira.probe-assignee`、Dislike→`jira.dislike-assignee`、Issue & Request→`jira.usage-feedback-assignee`、Failed query→`jira.failed-query-assignee`、Review wrong→`jira.review-assignee`。后两个占位文字 `Defaults to longyu3`，其余三个 `Jira username`。输入框宽 200px（用户现场看效果后可再调）。
- **Routing 页签整块删除**（原 608-637 行），页签从 4 个变为 3 个：Automation / Connection / Ticket content。
- Routing 里的 page focal 三步说明（精确路径 → 最长前缀 → `*` 兜底，以及「Open the Page focal table」按钮）迁到流程表下方，标题改为 `How the Issue & Request assignee is picked`，能力不丢。
- 保留 `.jira-flow-table__wrap { overflow-x: auto } / min-width: 620px`，表格不会撑破布局（1440 屏可能仍需横向滚动）。

## 三、改动二：保存校验与「留空回退」口径对齐（commit `a1afa0f1`，+8/-2）

- `FieldDef` 新增 `optional?: boolean`（注释：允许留空，后端会回退默认值，保存时不该拦）。
- `jira.failed-query-assignee`、`jira.review-assignee` 标 `optional: true`（两者 hint 本就写着留空回退 `longyu3`）。
- `save()` 校验由「5 个 assignee 全部非空」改为只拦必填：

```ts
const missing = ASSIGNEE_FIELDS.filter(
  (field) => !field.optional && !String(form[field.key] || '').trim(),
)
if (missing.length) {
  ElMessage.error('These assignees are required: ' + missing.map((field) => field.label).join(', '))
  return
}
```

原文案 `All three assignees are required` 与 5 个字段的实际校验数量不符，一并修正。

## 四、验证与本地预览

- `npm run type-check`（`vue-tsc --build`）在两次提交前均通过。
- 本地 dev server 已切换到 `dev/`renxp4 并热更新：`http://localhost:80/setting/settings?panel=jira` → Automation 页签可直接改 Assignee。
- `vite.config.ts` 的 `/api/admin/*` 代理在一次 `git reset` 中被冲掉，已手工补回但**未提交**（不属于这两个功能提交）；实测 root 200 / api 200。

## 五、未决事项（后续动作）

1. **未 push、未提 MR**：用户要求「MR 等我命令再提」，届时按现有分支提交建 MR（此前同类前端改动走 MR !1054 → master / !1055 → develop）。
2. 生产侧止血选择仍未拍板（关 Review wrong 开关 / 缩 `DEFAULT_LOOKBACK_HOURS` / 不动），见 2a95b6da。
3. 表格宽度与输入框宽度是否调整，等用户看过本地页面再定。
