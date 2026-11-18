---
id: "39c41e27-2c4b-43b1-bcf7-aad75e6372e5"
title: "Service Portfolio 前端接线 TST 验收（MR !1075：项目卡 / EOL 面板 / By Offering BU 筛选）"
description: "2026-11-18 TST 验收增量：project-cards / eol-panels / lifecycle/offering-trend 三个接口被真实页面调用（Sample data 徽标已删）；三处 UI 位置（Big Events 页底项目卡、Lifecycle 的 EOL 两栏、By Offering 的 Select BU）；BU 合并闭环 272,623 = 46,746 + 225,877，取消 IDG 后 225,877/1,770；遗留项目卡只显示第一条、EOL 文案仍是初稿；MR !1075 opened 未合入。"
status: "active"
created_at: "2026-11-18T03:30:38.288Z"
updated_at: "2026-11-18T03:30:38.288Z"
content_hash: "4507a795d0becd3857a9ec65c2dcea6d28fd2578a6a0eb03afaf6d2714864dbe"
source_paths:
  - "src/views/Portfolio/components/lifecycle/Lifecycle.vue"
  - "src/views/Portfolio/components/lifecycle/OfferingInventory.vue"
  - "src/views/GPSSUS/index.vue"
session_ids:
  - "mnemon-run-v1-69aa5b59-8bbe-4f9f-bcb0-1804b600892f"
memory_body_ids:
  []
---

# Service Portfolio 前端接线 TST 验收（MR !1075）

2026-11-18 增量验收记录。前端把展示页三处原本 localStorage / 硬编码的数据源改为接后端接口，部署 TST 后用真实浏览器会话核对。前置背景见：

- `1cc5f22b-e6ea-4ae2-80ed-7009516b6317` —— 2026-11-17 TST 部署验收（/spark 路由前缀、接口 401 需借登录 cookie、roadmap 取项目卡、KPI 只渲染 1/8、Sample data 硬编码）
- `e7cdd3c7-ce26-4dc9-a8a3-8dcaee8b4c65` —— MR !569 三个只读 GET 与前端逐接口核对表

## 1. 结论

用户部署 TST 后问「生效了吗、在页面哪」，**三处新接线全部生效**，判据是页面实际发出了后端请求（域名 `ssgspark-tst.lenovo.com`，页面路由 `/spark/portfolio`）：

```
/api/admin/service-portfolio/project-cards
/api/admin/service-portfolio/eol-panels
/api/admin/service-portfolio/lifecycle/offering-trend?bu=PCSD&fyFrom=FY2122&fyTo=FY2627
/api/admin/service-portfolio/lifecycle/offering-trend?bu=ISG&fyFrom=FY2122&fyTo=FY2627
```

`offering-trend` 的 `bu=` 参数确实被带上，说明前端与后端 !568 那版同时在线上。

## 2. 三处分别在页面的哪里

1. **项目卡** —— Big Events 页签，页面最下方左侧。卡片内容（ISG Service PN Lifecycle Review / In progress / 65% / 4 个里程碑 / 1 个 MEDIUM 风险）来自 `/project-cards` 第一条。**最硬证据是灰底「Sample data」徽标已消失**（该徽标是前端硬编码，本次删掉），即 2026-11-17 记录里的遗留问题已闭环。
2. **EOL 审查面板** —— Lifecycle 页签，滚到「SSG PN Inventory Overview」之后。左粉右紫的配色**不是写死**，按后端返回的 `tone`（pink / purple）加 class；判据条目、三步流程、Outcome 全部来自 `sp_sp_eol_review_panel` 表。
3. **BU 筛选** —— Lifecycle 页签 →「SSG PN Inventory Overview」→ 点「By Offering」子页签 → 左侧 Select BU。

## 3. BU 合并逻辑的闭环验证（可直接复用的判据）

- IDG + ISG 都勾：FY26/27 YTD = **272,623**（= PCSD 46,746 + ISG 225,877）
- 取消 IDG 后立即重发 `bu=ISG` 请求，数字变成 **225,877 / 1,770**（Qty of Active PNs / Booking per Active PNs），与上方「Utilization of Active PNs - ISG」卡片一字不差
- FY 序列（8 个 OH 全选）：FY26/27 YTD 272,623；FY25/26 266,264；FY24/25 281,825；FY23/24 246,020

## 4. 前端代码落点

- `src/views/Portfolio/components/lifecycle/Lifecycle.vue:120-167` —— 「SSG PN Inventory Overview」标题 + `role="tablist"` 的 Overview / By Offering 两个 tab（:122-123，`selectView('overview'|'offering')`）；:135 查询按钮 `refreshInventory`；:157-165 `<OfferingInventory>` 用 `v-if="offeringVisited" v-show="view === 'offering'"`；:166 `<EolReviewPanels />`
- `src/views/Portfolio/components/lifecycle/Lifecycle.vue:6` —— `import OfferingInventory from './OfferingInventory.vue'`
- `src/views/GPSSUS/index.vue:1977` `pnOfferingInventoryRows`、:2817 `pnOfferingInventoryChartOption`、:5260 `.pn-offering-chart` —— GPSSUS 侧同源的 PN inventory 图表实现

## 5. 遗留 / 风险

1. **项目卡只渲染第一条**（表里 15 行、有效 3 条），所以当前固定显示「ISG Service PN Lifecycle Review」。要轮播或展示其他项目，得改 sortNo 或补数据。
2. **EOL 面板显示的是配置表里的初稿文案**（"Created by WW PN Team"、"NOT embedded in any MTMs"、"Been created over 365 days"）。数据链路是通的，但业务方需在 Portfolio configuration 里填正式口径，否则评审时会被当成 demo 数据。
3. 验证过程去掉了 IDG 勾选，会话随后卡死（`session already has an unfinished command`），只能 stop 会话 `ubmn`，未能替用户勾回 —— 需人工在页面上点回 IDG 恢复默认两 BU 状态。
4. MR !1075 `feature/sp-display-wiring-develop` → `develop`，6 个文件变更，sha `e1da4355`，**opened / can_be_merged / has_conflicts=false**，尚未合入：https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1075

## 6. 构建与截图

- `vue-tsc --build` 通过，`EXITCODE=0`
- 截图（工作区相对路径，TST 实测）：`.tmp/sp-verify/tst-bigevents-projectcard.png`（项目卡）、`.tmp/sp-verify/tst-lifecycle-eolpanels.png`（EOL 两栏）、`.tmp/sp-verify/tst-lifecycle-byoffering.png`（By Offering BU 筛选），均为 2505×1253

## 7. 浏览器验证的坑

- `bsk` 会话要求先 `cooperate sync` 拿到未消费的 strict observation，再显式传顶层的 `observation_id`；否则报 `This action requires an unused strict observation`。
- 用户手动操作页面会触发 `User activity revoked this action`；此时不要重放提交，先 sync 再看 `structure_changed`。
- 长脚本执行中再次 evaluate 会报 `session already has an unfinished command`，只能等或 stop 会话。
