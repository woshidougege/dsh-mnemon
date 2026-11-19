---
id: "dd868132-79ae-412d-a3ed-0535528b966c"
title: "Service Portfolio 配置页保存交互现状与单条提交改造评估（2026-11-19）"
description: "四类配置模块（KPI Cards / Projects / Overview Modules / EOL Review）保存交互矩阵：仅 Projects 弹窗一步提交，其余为「弹窗改内存 + 页面级 Save」两段式；后端 4 个接口均为整份数组 PUT，排序靠下标换算 sort_no、删除靠数组缺席；代价是漏点、无 version 的 last-write-wins 覆盖、metricKey 未校验唯一；给出 A（2–3 小时，砍掉第二步）与 B（加单条端点 + 乐观锁，后端 1–2 天 + 前端 1 天）两方案及推荐。"
status: "active"
created_at: "2026-11-19T10:00:02.656Z"
updated_at: "2026-11-19T10:00:02.656Z"
content_hash: "6f68e5eb7a10351c6c3fcd04adf6b0eeedd499a63909cddc5d3236eb8371196c"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioConfigService.java"
session_ids:
  - "mnemon-review-v1-af74d393-8f03-4a57-b8ef-41fac72fd680"
memory_body_ids:
  []
---

# Service Portfolio 配置页保存交互现状与「单条提交」改造评估（ssg-sales-ops，2026-11-19）

范围：Portfolio 配置页下 4 个配置模块的保存交互与后端接口契约。前端仓库 `ssg_sales_ops`（目标分支 develop），后端仓库 `ssg-sales-ops`（目标分支 admin-release）。
关联文档：`a538574e-ee32-4c2d-b97c-85a1f34396bc`（KPI 取数口径与 valueType 降级缺陷）、`b3126a3d-15e3-424e-b74c-2f52f0880a37`（valueType 统一为 COMPUTED 的双端修复与 MR 落地）。

## 0. 结论

用户提问「配置里那些数据是不是都不支持单条提交，都要分两步整体提交」。答案：**不统一，四类配置分两种交互方式**。

- 只有 **Projects 是弹窗内一步保存**（弹窗 Save 直接发请求）。
- **KPI Cards / Overview Modules / EOL Review Panels 是两段式**：弹窗里的 Save 只写前端内存变量，必须再点页面顶部 Save 才发请求。
- 后端 4 个接口**全部是整份数组 PUT**，没有提供任何单条新增、修改、删除接口。

## 1. 交互矩阵（行号为前端 ssg_sales_ops 实测）

| 模块 | 弹窗按钮 | 弹窗是否调接口 | 页面级按钮 | 后端接口 |
| --- | --- | --- | --- | --- |
| KPI Cards | Save card（`src/views/Portfolio/config/KpiCardsConfig.vue:244`） | 否，只写 `cards.value`（:85-130） | Save KPI cards（:170） | `PUT /config/kpi-cards` 全量 |
| Projects | Save project（`ProjectCardsConfig.vue:99`） | 是，:154 `persistProjects` → `saveProjectCardList` | 无（删除即时生效，:80） | `PUT /config/project-cards` 全量 |
| Overview Modules | Save module（`OverviewModulesConfig.vue:87`） | 否，只写 `modules.value`（:119） | Save modules（:149） | `PUT /config/overview-modules` 全量 |
| EOL Review Panels | Save panel（`EolReviewConfig.vue:58`） | 否，只写 `panels.value`（:83） | Save panels（:154） | `PUT /config/eol-panels` 全量 |

页面级保存统一走「整份数组 PUT」，例如 KPI：`saveConfiguration()` 调用 `savePortfolioKpiConfig(cards.value.map((item, index) => ({ ...item, sortNo: index + 1 })))`（`KpiCardsConfig.vue:142-147`）。

## 2. 为什么设计成整份提交（架构理由）

- **排序**：顺序由数组下标换算成 `sort_no`（`ServicePortfolioConfigService.java:127`），前端保存时重编号。
- **删除**：表达为「提交的数组里没有这一行」，后端按数组 upsert 并对缺席行做删除。
- **接口经济性**：一个整份 PUT 同时覆盖增、删、改、排序四种操作，避免为 4 类记录 × 4 种操作维护 16 个端点。

## 3. 代价与已知风险

1. **容易漏点**：弹窗 Save 后以为已生效，只要不再点页面 Save，刷新即丢失。
2. **last-write-wins 静默覆盖**：表无 version 字段、无 etag，两人同时编辑时后保存者覆盖先保存者，页面不提示。
3. **Projects 同样有覆盖问题**：它虽一步提交，提交的仍是整份列表。
4. **KPI metricKey 未校验唯一**：新增卡片允许重复 `metricKey`，后端 upsert 会落到同一行，两行互相覆盖。

## 4. 改造选项

| 方案 | 改动 | 工时 | 解决什么 |
| --- | --- | --- | --- |
| A 最小改动（建议先做） | 三个页面的弹窗 Save 直接调用现有整份 PUT，砍掉页面级第二步；不动后端 | 2–3 小时 | 只解决「两段式让人怀疑是否生效」的体验问题，不解决并发覆盖 |
| B 真正的单条接口 | 后端加 4 组 `POST` / `PUT /{id}` / `DELETE /{id}`，前端三页改即时提交 | 后端 1–2 天 + 前端 1 天 | 需同时引入 version 字段与乐观锁，才能把并发覆盖一起解决 |

建议：先用 A 上生产（当天可交付），并发覆盖等业务真出现「改了但没保存」的投诉再上 B。

## 5. 未决事项（截至 2026-11-19 17:59）

- 用户尚未拍板现在改 A 方案还是先由业务试用一轮；若动手，改动落在分支 `feature/sp-display-wiring-develop`，走新的 develop MR。
- 前端主仓库 `ssg_sales_ops` 当时停在 develop 且有他人遗留的未解决冲突（`UU src/constants/feedbackMenuCatalog.ts`，落后 origin/develop 85 commits），属他人待办；该状态为临时事实，仅作动手前的前置检查项。

## 6. 操作提示

- `sp_sp_kpi_card` 的 owner 为 `a_appconnect`，日常账号只有 SELECT；改这 8 行需临时借服务账号，且必须逐次获得用户授权（本次已按「授权你1次」单次执行）。
- 配置表的 `data_source` 列是纯备注，不参与取数；真正决定是否实时取数的是 `value_type` 与 `resolveMetric` 里的 5 个 metric key 白名单（见关联文档 a538574e）。
