---
id: "c7140f7c-24ae-4a3e-89ec-282af1331ae8"
title: "Service Portfolio：OH 选择器树形筛选与财年绑定问题分析（前端反馈，ssg-sales-ops）"
description: "前端反馈的 OH 树形多选与财年绑定问题根因：/oh 接口扁平化+全局 LIMIT 50 导致仅 GPS 有子节点、lifecycle 无 ohCodes 参数；含层级基数/脏数据实测、原型对齐目标、三项接口改法与 3 个待拍板决策。"
status: "active"
created_at: "2026-11-13T02:39:49.332Z"
updated_at: "2026-11-13T02:39:49.332Z"
content_hash: "28cee5fbc3b0e6b72a48c735874550e296383833561079b8bc42c6427d0d98a7"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/dao/ServicePortfolioReadMapper.java"
  - "docs/service-portfolio-api.md"
session_ids:
  - "mnemon-run-v1-9722a195-c4c8-418b-8630-e739414d187a"
memory_body_ids:
  []
---

# Service Portfolio：OH 选择器树形筛选与财年绑定问题分析（ssg-sales-ops）

仓库 `dataagent/ssg-sales-ops`，工作树 `.agents/worktrees/service-portfolio-admin`（基于 `admin-release`）。
来源：前端反馈（2026-11-13）——"这个筛选我看图是树形多选的，应该全量的，你这个写的是模糊搜索最多 50 个吗？这个页面接口感觉没跟财年 OH 筛选绑定，查出来数据看起来没法渲染成效果图"。
结论：**两个问题都成立，根因都在后端接口**，不是前端实现错误。本轮只做分析，未改代码。

## 1. 现状接口（问题①：只有扁平列表 + 全局 50 上限）

- 唯一入口 `GET /service-portfolio/oh?keyword=xxx` → `[{code,hbb,l2,l3,l4,l5,l6}]`。
- `smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java:340-342` 把 `limit` 写死为 50。
- `smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/dao/ServicePortfolioReadMapper.java:157-175`：
  ```sql
  SELECT offering_hierarchy_code AS code, oh_l1_hbb AS hbb, oh_l2_offering AS l2, ...
  GROUP BY 1..7 ORDER BY 1 LIMIT 50
  ```
- 缺陷三条：① 无父子节点对象（中间层 L1/L2/L3 无 name/path/children/计数），前端无法懒加载或折叠；② `ORDER BY code` 字典序 + 全局 50 条，实测前 50 条分布：Global Product Services 38、Digital Workplace Solutions 10、Sustainability Services 2，Hybrid Cloud / Vertical / AI / Vehicle Computing **0**（首个码分别为 `H000000001KV` / `H00000000056` / `H00000000GQA` / `H00000000PV8`）——正是截图里只有 GPS 展开有子节点、其余展开为空的原因；③ 无 HBB / 财年 / 状态维度，keyword 是唯一入口。
- 前端"Offerings load on demand, 50 records per request"是照后端文档补的，原型里没有这句。

## 2. 问题②：Lifecycle 接口没有 OH 参数

- `smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java:68-90`：
  `@GetMapping("/lifecycle/overview") lifecycleOverview(hbb)` 与 `@GetMapping("/lifecycle/utilization") lifecycleUtilization(hbb, bu)`——只有 hbb/bu。
- 原型中 Select OH 属于 Lifecycle 页 **By Offering** 视图（Overview / By Offering 两个按钮），勾选后 `updateHbOfferingInventory()` 重算右侧 **Fiscal Year / Qty of Active PN / Booking per Active PNs** 表与 **PN Inventory Trend** 图，标题由 `· All Offerings` 变为 `· N selected OHs`。因此树画出来也没有参数可传。

## 3. 原型实际效果（对齐目标）

- By Offering 视图里 `aside.hb-offering-tree`（title "Select OH"）+ 右侧 `.hb-offering-summary` 表 + `.hb-offering-trend` canvas。
- 静态节点、checkbox 多选 + caret 折叠，**无计数、无懒加载**；DOM 中可见 14 个 `data-offering-filter` 节点，L2 通过 `.indent` 缩进（如 GPS 下 Configuration / Deployment / Support Services）。注意 "Managed & Professional Services" 标签在两个不同父节点下重复出现（Digital Workplace 与 Hybrid Cloud 的 Managed Cloud），前端若按 label 做 key 会冲突。

## 4. 数据侧能力与限制（已实测 gps_pn_lifecycle_overview）

| 项 | 结论 |
|---|---|
| 层级基数 | L1 8 / L1→L2 15 / →L3 65 / →L4 259 / OH 码 1617（全路径 1629 行），distinct L2 = 12 |
| 全量树体量 | 约 1.6K 节点，JSON 约 150KB，可一次下发，无须 50 条上限 |
| 脏数据 | 293 个 OH 码 L1/L2/L3 全为 NULL（如 `H0000000022E`）；1 个空串 L1；1 个 `Vehicle Computing Service`（25 个码，不在标准 HBB 枚举内） |
| OH 主数据表 | **不存在**，层级只能从 `gps_pn_lifecycle_overview` 反推 |
| 财年 | 无 OH 版本/生效财年表，只有 PN 级 `creation_fiscal_year` / `withdrawal_fiscal_year`（空值 1,418 / 277,786），只能近似 |

近似口径（`creation_fiscal_year <= FY` 且 `withdrawal_fiscal_year` 为空或不早于 FY）筛"该财年有效 OH"：FY2122 715、FY2425 1,400、FY2627 1,548（全量 1,617），结果合理。

## 5. 建议改法（待用户拍板后实施）

1. `GET /oh/tree?hbb=` → 全量层级（L1→L3），每节点带 `path / name / ohCodeCount / pnCount / children[]`，**不设上限**。
2. `GET /oh/codes?path=&keyword=&page=&size=` → 叶子 OH 码真分页，返回 `{rows,total}`，保留关键字搜索。
3. Lifecycle 加 OH 维度：`/lifecycle/utilization?hbb=&bu=&ohCodes=` 与新增 `/lifecycle/offering-trend?ohCodes=&fyFrom=&fyTo=`，让点击节点后能重算右侧表与图。
4. 落地范围：`ServicePortfolioReadMapper` / `ServicePortfolioReadService` / `ServicePortfolioController`，并同步 `docs/service-portfolio-api.md`（注意该 docs 目录被 `.gitignore` 忽略，见文档 `5b52aa68-ce8b-4e5c-9d25-575abd544793`），最后提 MR 由用户合并。

## 6. 待用户决定的 3 个问题（当前阻塞实施）

1. 树的层级做到哪层？原型只到 L2，数据能到 L6；建议做到 L3，叶子 OH 码放右侧列表分页。
2. 财年是否真按财年过滤？只能近似式（无 OH 版本表），或本版只按 OH 过滤、财年仅作右表展示维度。
3. 293 个无 HBB 的 OH 码隐藏还是归入 "Unmapped" 节点？（隐藏则 1,617 → 1,324）
