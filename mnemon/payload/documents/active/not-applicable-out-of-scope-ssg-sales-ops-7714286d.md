---
id: "7714286d-1a8d-4dd3-9770-39139a751aae"
title: "评审结论三态落地：Not applicable 语义、Out of scope 文案与抽屉结论区布局（ssg_sales_ops）"
description: "评审结论从两态扩为 CORRECT/WRONG/NOT_APPLICABLE 的落地细节：准确率分母排除 N/A、归因标签必须分层（WRONG_TAG_OPTIONS vs NA_REASON_OPTIONS）、Out of scope 优于 Not in scope 的四条理由、368px 右栏三列导致文字挤压的宽度验算与两列改法、Backfill 是转换而非清除旧 AUTO No-permission 判定、MR !1049/!1050/!558 与已合并旧 MR 导致 tst 仍是旧样式的坑。"
status: "active"
created_at: "2026-11-16T08:17:26.368Z"
updated_at: "2026-11-16T08:17:26.368Z"
content_hash: "9f8a241725c8e326665f3da3fcca9a69c0e6c38a22162b6d3308e9e962a5be21"
source_paths:
  - "ssg_sales_ops/src/views/Setting/reports/components/QuestionReviewDrawer.vue"
  - "ssg_sales_ops/src/views/Setting/reports/components/questionReview.ts"
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/components/FilterBar.vue"
  - "ssg_sales_ops/src/views/Setting/settings/panels/ReviewRulesPanel.vue"
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventExportService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/usagefeedback/service/UsageFeedbackExportService.java"
session_ids:
  - "mnemon-run-v1-c692817b-a995-4785-a2e0-85f14f884970"
memory_body_ids:
  []
---

# 评审结论三态落地：Not applicable 语义、Out of scope 文案与抽屉结论区布局（ssg_sales_ops）

三态化的落地记录（前端 + 后端 + 文案 + 布局）。概念与数据模型的原始设计见 38b9c4c0-0988-4206-9198-11f20587ae61（该篇仍是两态 CORRECT/WRONG 版本）；抽屉外壳两栏工作台改造见 38cfa2fb-9dd3-4b6d-bf04-a86a0e7f0d3d。本文记录此后新增的第三个结论 NOT_APPLICABLE 及其连锁改动。

## 1. 为什么要有第三态

原两态把「权限不足」「问题不在系统范围内」都塞进 Wrong，导致准确率被系统性低估：这类问题根本不该按对错评判。改为三态后：

- `CORRECT` / `WRONG` / `NOT_APPLICABLE`
- **准确率分母排除 NOT_APPLICABLE**（分母 = Correct + Wrong），N/A 行既不进分子也不进分母
- `NO_PERMISSION` 从「Wrong 的归因」改为「N/A 的原因」——语义从「答错了」纠正为「不适用」

## 2. 归因标签必须分层，不能共用一套下拉

`ssg_sales_ops/src/views/Setting/reports/components/questionReview.ts` 拆成三组常量：

- `ATTRIBUTION_TAG_OPTIONS`：全量标签字典，**只用于「值 → 英文展示」和筛选候选**
- `WRONG_TAG_OPTIONS`：DEFINITION_MISMATCH / WRONG_REPORT / DATA_MISSING / DISPLAY_ERROR / OTHER（Answer wrong 时的「错在哪」，**不含 NO_PERMISSION**）
- `NA_REASON_OPTIONS`：NO_PERMISSION / OUT_OF_SCOPE（不判对错的两种原因）

理由：若共用下拉，用户会在 Wrong 下选到 No permission，语义立刻矛盾，且后端 CHECK 约束会拒。

## 3. 文案决策：Out of scope（不用 Not in scope）

四条理由（用户当面问「推荐哪个」，结论是改掉原来的 Not in scope）：

1. 更短（12 字符 vs 14），在 368px 右栏里宽度更安全
2. 更地道——英语里 `in scope / out of scope` 是固定搭配，`not in scope` 是否定式变体，正式文案少用
3. 与数据库枚举 `OUT_OF_SCOPE` 同词根，读日志/查库不用换脑子
4. 与并列项 `No permission` 的否定式 + 名词短语结构更齐

已全站统一：筛选项、下拉、表格列、两个导出文件（`ChatbiEventExportService.java`、`UsageFeedbackExportService.java` 各改 1 个字符串常量）。

## 4. 结论卡片布局：从三列改两列 + N/A 独占一行

用户反馈「Not applicable 那张卡像竖排文字」。根因不是配色也不是字号，是**宽度**：

- 抽屉右栏固定 **368px**（`grid-template-columns: minmax(0, 1fr) 368px`，`QuestionReviewDrawer.vue:811`）
- 三列等分 → 每卡约 **109px**；扣 24px 圆点 + 10px 间距 + 左右 24px 内边距 → 文字区只剩约 **51px**
- `Not applicable` 在 12px/700 下约需 **78px** → 必然断行；说明文案 `No permission / out of scope` 约需 **150px** → 折四行，把第三张卡撑到约 120px，grid 又让另两张等高，内容悬空显得松散

改法（`QuestionReviewDrawer.vue`）：`grid-template-columns: repeat(2, minmax(0, 1fr))`，N/A 卡加 `grid-column: 1 / -1` 独占整行；说明文案缩为 `Usable as is` / `Needs fixing` / `No permission or out of scope`；字号 9.5px → 10px 并加 `white-space: nowrap`。

验算：Correct/Wrong 各 168px（可用 110px，文案约 62px ✓）；N/A 344px（可用 286px，文案约 151px ✓）；总高 58+8+58 = 124px，与原被撑高的约 120px 基本持平，**不会把下方下拉与备注框往下推**。

语义理由也支持这个排布：N/A 本来就不判对错，单列一类比与 Correct/Wrong 三等分更准确。

## 5. Backfill 语义：转换而非清除

`ssg_sales_ops/src/views/Setting/settings/panels/ReviewRulesPanel.vue` 的说明文案曾与实现不符（写「只碰没有结论的行」，实际会先改判已有结论的 AUTO 行）。修正后的口径：

- 没有后台任务、没有夜间扫描，**纯手动**运行，选时间窗后自己点
- 已有结论保持不变，**唯一例外**是历史遗留的 AUTO 判定 No-permission 行，被 **converted to Not applicable**（不是 cleared）——因为那条自动规则已不存在
- 重复运行安全（幂等）
- 权限失败不再产生自动 Wrong：未评审时保持 Pending，评审后由人判 `Not applicable · No permission`，从而不进准确率分母

## 6. 落地面（2026-11-16）

前端 3+1 文件：`QuestionReviewDrawer.vue`、`components/questionReview.ts`、`user-logs/components/FilterBar.vue`，随后追加 `ReviewRulesPanel.vue` 文案修正；后端 2 个导出服务各 1 行。

MR：前端 `!1049`（dev/renxp4 → master）、`!1050`（tmp/review-na-develop → develop）；后端 `!558`（feature/review-not-applicable-admin → admin-release）。

踩坑：用户在 15:56 已合并上一批 !1047/!1048/!557，因此部署到 tst 看到的仍是旧样式——样式/文案类改动**必须再开新 MR 并合并**才在环境上生效，只看自己分支的 commit 容易误判「已生效」。

## 7. 尚未处理

右侧「Current verdict」区块信息层级偏弱：Pending 时整块白底灰字与抽屉背景同色，反而成了最不显眼的元素。建议加 3px 左侧色条（Correct 绿 / Wrong 红 / N/A 蓝 / Pending 灰虚线框）与列表状态 chip 呼应——已提出，用户未确认是否执行。
