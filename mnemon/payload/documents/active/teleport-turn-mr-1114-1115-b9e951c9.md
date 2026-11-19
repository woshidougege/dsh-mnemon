---
id: "b9e951c9-33cb-412e-9af8-853d163a3886"
title: "会话回放翻译落地：控件 Teleport 进弹窗标题栏 + 只翻当前 turn（MR !1114 / !1115）"
description: "ssg_sales_ops 前端把回放翻译从正文工具栏移入 DetailModal 标题栏（锚点 + Vue 3.5 defer Teleport）、翻译范围收敛到 targetTurnId 的当前问题与其答案、el-select 换成自绘胶囊+浮层并内联状态；双分支 develop/master 同步提交与 !1114/!1115 的验证方式（patch-id、merge-tree、vue-tsc）。"
status: "active"
created_at: "2026-11-19T08:59:03.476Z"
updated_at: "2026-11-19T08:59:03.476Z"
content_hash: "5432aa00aaa7577f34e9319a24d98bb1507fc61851d723636ac86a0f4233a308"
source_paths:
  - "src/views/Setting/reports/history-replay/HistoryChatPanel.vue"
  - "src/views/Setting/reports/user-logs/components/modals/DetailModal.vue"
session_ids:
  - "mnemon-review-v1-2ae2b115-ac06-4bcc-8427-72813501f34e"
memory_body_ids:
  []
---

# 会话回放翻译落地：控件移入弹窗标题栏 + 只翻当前 turn

日期：2026-11-19。仓库：GitLab `dataagent/ssg_sales_ops`（前端 Vue 3.5.24 + Element Plus）。
本地工作树：`C:/Users/63588/lenovo/ssg_sales_ops-wt-replay-i18n`（与主工作目录 `ssg_sales_ops` 分开的 worktree）。
前置调研与范围判断见文档 `62fa30a6-cbdc-4661-a99e-68d47604441f`（历史回放弹窗多语言翻译方案）；本文只记最终实现与交付。

## 1. 三个改动点与 rationale

| 改动 | 做法 | 为什么 |
|---|---|---|
| 位置 | 翻译工具栏不再占正文区域，改为挂到弹窗标题栏右侧 | 正文顶部被工具栏吃掉两行，用户反馈"像多了一条状态栏" |
| 范围 | 只提交 `targetTurnId` 对应的那条 user 消息（找不到则取最后一条 user）+ 紧随其后的 assistant 答案 | 原实现整条会话（100+ 条）一起提交，等 2-3 秒且费用高；回放场景只需要看懂"当前这一问一答" |
| 外观 | 去掉 `el-select` 方框，换成 26px 高胶囊触发按钮 + 自绘浮层 | el-select 在这个窄标题栏里视觉太重，且无法内联 spinner/错误态 |

### 1.1 Teleport 挂载（位置解耦的关键）

`DetailModal.vue` 的 `#header` slot 与内嵌的 `HistoryChatPanel` 是两棵子树，工具栏逻辑却在面板里。做法是：

- `src/views/Setting/reports/user-logs/components/modals/DetailModal.vue:171` 在标题行右侧留锚点：
  `<div id="replay-toolbar-anchor" class="dm-head-actions"></div>`（同 `.dm-head` 内、title 之后）
- `src/views/Setting/reports/history-replay/HistoryChatPanel.vue:462` 用 `<Teleport defer to="#replay-toolbar-anchor">` 把工具栏投送过去

`defer` 是必须的：Teleport 目标由父组件在同一轮渲染里创建，不加 defer 会出现目标节点尚未存在的时序问题。**该写法依赖 Vue 3.5+**（本项目 `vue ^3.5.22`，实装 3.5.24），移植到 3.4 需改为 `onMounted` 后再挂载或把锚点提到父级 template 之前。

配套样式：
- `DetailModal.vue:404-410` `.el-dialog.ulr-detail-dialog .dm-head-actions { display:flex; flex:none; align-items:center; gap:8px; margin-left:auto; }` —— `margin-left:auto` 把它推到标题行右端。
- `DetailModal.vue:346-374` 自定义 dialog 外观：`--el-dialog-padding-primary: 0`、header `padding: 18px 54px 14px 20px`（右侧留 54px 给关闭按钮）、`__headerbtn` 34×34 圆角 10px。
- `DetailModal.vue:395-402` `.dm-head` 保持 `padding-right: 38px` 并 `justify-content: flex-start`，title 单行 ellipsis，避免锚点被长标题挤掉。

### 1.2 只翻当前 turn

- 任务收集函数（`HistoryChatPanel.vue` 内，`collectTranslationJobs()` 一带，约 :396-453）：从 `targetTurnId` 定位 user 消息 → 取该条 `content` 与 `finalAssistantText`（回放模式下即紧随其后那条 assistant 答案），其他历史消息保持原文。
- 状态位：`HistoryChatPanel.vue:73-74` `translateStatus` / `translateError`，切换会话时（约 :316-317）与重新发起时（:396）清空。
- 空输入/空内容有两类错误文案：`'Type a language name, then press Translate.'`（custom 未填）、`'No natural-language text in the current question.'`（该 turn 没有可翻正文）。
- 进度：`total > 1` 时显示 `Translating ${done}/${total}…`，单条只显示 `Translating…`；失败统一 `Translation failed: ${message}`。

### 1.3 胶囊 + 浮层

- `HistoryChatPanel.vue:608-629` `.replay-toolbar` 改为 `display:flex; align-items:center; gap:8px`（不再 `justify-content: space-between`，因为已无"左侧 hint + 右侧动作"两栏，状态并入按钮）。
- `.replay-spin`（:614-628）：11px 圆环，`border-top-color: #6d57d8`，`animation: replay-spin .7s linear infinite`。
- `.replay-original-toggle`（:630 起）：眼睛图标 + Original/Translated 文案，`.is-on` 表示当前显示原文。
- `.lang-picker/.lang-trigger`（:671-688）：`height:26px`、`border-radius:999px`、渐变底 `linear-gradient(180deg,#fff,#faf8ff)`、字号 12px、地球图标 + 语言名 + chevron；浮层 `.lang-menu` 圆角 12px、当前语言右侧打勾、`custom` 展开为内联输入框 + Go。
- 错误态内联到按钮：`is-error` 时按钮与边框转红（`#c22a48`，#d03050 用于正文错误行），旁边一行 `text-overflow: ellipsis` 的原因，完整文案放按钮 `title`。

## 2. 交付方式（双分支同改）

同一份改动要同时进 `develop` 与 `master`，做法是从各自目标分支切 feature 分支、提交信息完全一致，再用 patch-id 比对确认等价：

| 分支 | 目标 | 最终提交 | MR |
|---|---|---|---|
| `feature/replay-translate-develop` | develop | `bd15567d` | !1114 |
| `feature/replay-translate-master` | master | `0c7f3108` | !1115 |

- 提交信息：`refactor(reports): 回放翻译收敛到当前问题并把控件放进弹窗标题栏`
- 两个 MR 均为 `state=opened`、`merge_status=checking`、`conflicts=False`。
- 注意：改动后重新提交并对两条分支做了**强制推送**（`+ 2b78a660...bd15567d`、`+ 5a8574f2...0c7f3108`），所以早先的 `2b78a660` / `5a8574f2` 引用已失效，以 MR 上的 sha 为准。

### 验证组合（可复用）

1. `npm run type-check`（`vue-tsc --build`）两条分支都 exit 0。
2. `git merge-tree` 预检目标分支，确认无冲突再开 MR。
3. `git patch-id --stable` 对比两分支的提交，确认除目标分支基线差异外内容等价。
4. 合并后在 tst 刷新页面目视确认（本次把改动落到 `DetailModal` 后不涉及路由/权限，无需额外配置）。

## 3. 需要注意

- 工具栏现在跨两层组件：`HistoryChatPanel` 里的事件/状态，`DetailModal` 里的容器与样式。给标题栏加东西时要同时看 `.dm-head` 的 `padding-right: 38px` 与 header 的 `54px`，否则会和关闭按钮或锚点重叠。
- 该面板同时被真实聊天页使用（`PbiRecommendation` 那条渲染链），回放专属能力一律靠 props 开关，不要在组件内按路由/全局状态分支。
- 之前 master 缺过回放评审链路（见 `e473a76e-9ef6-443f-b7e6-fcbf4f544574`），此类"只在 develop 做过"的功能在新开 master 分支前要先核对基线是否含依赖文件。
