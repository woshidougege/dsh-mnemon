---
id: "233b7bf8-f9d0-48c5-8511-15934e162fa7"
title: "设置中心统一表单皮肤 settingsControls.less（Page focal 口径）与 Review rules 表格修正"
description: "在 .settings-page 作用域内统一 Element Plus 输入框/下拉/文本域/开关皮肤（32px 高、8px 圆角、浅底内描边、focus 紫色光晕、品牌紫主色），包含设计 token 取值、作用域边界理由、Review rules 三处修正与 commit 35e93885 的验证状态。"
status: "active"
created_at: "2026-11-17T04:04:52.180Z"
updated_at: "2026-11-17T04:04:52.180Z"
content_hash: "710e1d0ffd0b857f59b30b8ee915d3518813b79aabafc518ac100de7d9315c2a"
source_paths:
  - "src/styles/settingsControls.less"
  - "src/main.ts"
  - "src/views/Setting/reports/usage-issue-feedback/PageFocalEditor.vue"
  - "src/views/Setting/settings/panels/ModeGroupsPanel.vue"
  - "src/views/Setting/settings/panels/JiraPanel.vue"
session_ids:
  - "mnemon-run-v1-56980dca-7ff1-48b9-aedd-491d6e15b64e"
memory_body_ids:
  []
---

# 设置中心统一表单皮肤（Page focal 口径）与 Review rules 表格修正

仓库 `ssg_sales_ops`，工作树 `ssg_sales_ops-wt-nareview`，分支 `dev/renxp4`。

## 1. 需求

用户要求优化设置页整体观感，重点是 `settings?panel=jira` 的输入框、以及 Mode groups 的输入框"太难看"；并指定以 Page focal 页面的输入框为好看基准，全站统一输入框。原话（paraphrased）：优化页面，settings?panel=jira 也优化，主要输入框太难看，Mode groups 输入框也难看，整个 settings 都看看，Page focal 输入框还行，统一一下。

## 2. 根因判断

- 设置页里只有 Page focal 的 Assignee 输入框被单独美化过（`src/views/Setting/reports/usage-issue-feedback/PageFocalEditor.vue`，类 `.pf-assignee-input`，第 307 行模板、597-624 行样式：浅底、1px 内描边、hover 转白、focus 紫色描边加光晕）。
- 其余面板（Mode groups、Jira、Admin tabs 等）全是 Element Plus 未覆盖的默认样式：4px 圆角、白底、灰描边。
- "碎"的直接原因：面板里大量 `size="small"` 输入框默认只有 24px 高，与同一行的按钮、下拉（默认更高）对不齐。Mode groups 的 ID/Name 列、Jira 流程表的 assignee 列都是这种情况。

## 3. 实现：新建全局皮肤文件

新建 `src/styles/settingsControls.less`，全部规则限定在 `.settings-page` 作用域内，统一 Element Plus 表单控件：

```
--st-surface:      #ffffff;                  /* hover/focus 转白的底色 */
--st-surface-2:    #fdfcff;                  /* 常态浅底 */
--st-line:         rgba(93, 84, 150, 0.15);  /* 1px 内描边的常态色 */
--st-line-hover:   rgba(93, 84, 150, 0.28);
--st-txt:          #33304a;
--st-txt-3:        #9a95b4;                  /* placeholder */
--st-brand:        #948ae8;
--st-brand-soft:   rgba(148, 138, 232, 0.1); /* focus 3px 光晕 */
--st-radius:       8px;
```

- 高度：`.el-input / .el-select / .el-textarea` 一律 32px（同时置 `--el-input-height` 与 `--el-component-size`），消除 24px 与按钮不齐的问题。
- 描边：用 `box-shadow: 0 0 0 1px <line> inset` 而非 border，避免占位跳动。
- 状态：常态 `--st-surface-2` → hover 转白 + 描边加深 → focus 白底 + 品牌紫 1px 描边 + 3px 光晕。
- 禁用态保留灰底 `#f5f4fa`、不跟随转白，避免看起来仍可输入。
- 字号 12.5px，placeholder `#9a95b4`；下拉箭头与清除图标改 `#b3aecb`，不再用 EP 深灰。
- 文本域同一套取值，`padding: 8px 10px`、`line-height: 1.6`。
- 开关：EP 把配色写在组件自身，祖先变量覆盖不了，只能直接命中 `.el-switch`，置 `--el-switch-on-color: #6f63c9` / `--el-switch-off-color: #cfcbe0`。
- 另把 EP 主色在设置页内换成品牌紫（含 light-3/5/7/8/9 与 dark-2），否则按钮和链接仍是默认蓝，与紫灰界面冲突。

### 作用域为什么只限 `.settings-page`

Admin Control 的表格、评审抽屉有自己的密度和皮肤，不能被这套 32px 行高带偏。皮肤文件顶部注释里写明了这条边界，避免以后有人把规则提到全局。

引入位置：`src/main.ts` 在 `element-plus/dist/index.css` 与 `src/assets/element.css` 之后、`src/styles/adminScope.less` 之前。

## 4. Review rules 面板附带三处修正

1. 开关列宽 44px → 72px 并右对齐，原来表头 `AUTO REVIEW` 被折成三行。
2. 行内左右各补 12px 内距，hover 背景不再贴着文字。
3. 该面板单独写死的 30px 下拉高度去掉，与全局 32px 对齐。

## 5. 验证与状态（截至本次记录）

- 提交 `35e93885` `feat(feedback): 设置中心统一表单皮肤，输入框跟 Page focal 一个样子`，3 files changed，155 insertions(+)、5 deletions(-)，其中 `src/styles/settingsControls.less` 为新增。
- 同分支前一个提交 `041238d9` `feat(feedback): Review rules 改成按条件配置归类与开关`。
- `pnpm run type-check`（`vue-tsc --build`）通过。
- dev server 在跑，HMR 已应用；预览图 `settings-skin-preview.png`（1800x1520）是拿 Element Plus 真实 CSS 渲染的左右对照。
- **未 push、未提 MR**，等用户确认后才提。

## 6. 相关既有记录

- `a66df2de-9353-416f-bc6b-777157ca3ac1`：Jira 设置页 Assignee 内联进 Automation 表格的先前实现，本次 Jira 流程表输入框皮肤即作用于那张表。
- `424bade2-7360-4eb2-b7b0-b80aa79702de`：设置页导航默认收起与前端筛选条数统一，同一设置页区域的另一组改动。
