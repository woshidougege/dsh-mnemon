---
id: "121d6b28-8c37-4caf-b9f6-8db4047c94a8"
title: "Service Portfolio Lifecycle「By Offering」：复用 GPS/SUS active-pn 而非重做 /oh（会议录屏结论）"
description: "会议录屏结论：By Offering 应复用已上线的 GPS/SUS /active-pn（四层树+BU×财年 inventory），加 hbbs[] 参数放开范围而非自建 /oh；含录屏原话时间戳、硬编码位置、business_unit/树规模实测、0.5–2 人日实施路径与 3 个待拍板项。"
status: "active"
created_at: "2026-11-16T03:17:07.909Z"
updated_at: "2026-11-16T03:17:07.909Z"
content_hash: "65127485a0918874d8826845cb32f06757fe8e1a467975a8b0661adfab3f52b1"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/controller/GpsSusController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/service/GpsSusService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/dao/GpsSusActivePnMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java"
session_ids:
  - "mnemon-run-v1-76517029-d0b3-4b30-ae23-d3c98dc582d4"
memory_body_ids:
  []
---

# Service Portfolio Lifecycle「By Offering」：复用 GPS/SUS active-pn，不重做 /oh

仓库 `dataagent/ssg-sales-ops`，工作树 `.agents/worktrees/service-portfolio-admin`（基于 `admin-release`）；前端在 `C:\Users\63588\lenovo\ssg_sales_ops`。
起因：用户对 Service Portfolio 新需求（Lifecycle 页 By Offering 视图）不熟悉，要求"看看会议录屏文件"再给建议。约 70 分钟录屏已转写为 `part1_small_plain.txt`（745 行，32:56）与 `part2_medium_plain.txt`（807 行，38:32）。
产出：**推翻了此前自建 `/service-portfolio/oh` 的思路**，改为把已上线的 GPS/SUS By Offering 参数化复用。

## 1. 录屏原话（关键证据）

| 时间 | 原话 | 含义 |
|---|---|---|
| part1 31:48–32:31 | 「life cycle 这块其实我们也基本上是跟那边是一致的」「首先这两张大的图，然后包括这个 by offering 这块的一个展示，就完全复用 GPS SUS 里面的这个展示」 | 不新建，复制并放开范围 |
| part1 29:41–30:27 | 「concord project 这块其实我们就完全复用了 SUS GPS 的那个展示…只是我们不限于 GPS 和 SUS 这两个 HBB」 | 同样做法 |
| part2 03:37–04:07 | 「也是展示 overview 和 by offering 两个…这块我们也是想复用这一部分，只是说我们是把所有的 HBB 都放开」 | 确认两视图，范围放开到全 HBB |
| part2 04:12–05:27 | 「下面需要增加两个我们可以自己配置的…筛选逻辑…还有流程」「尽量提供可以长期通用的模板，不然以后改就得改代码」 | 底部两个可配置模板块，需配置化 |
| part2 11:01–17:45 | 「老板要对比 GPS 和 DWS」「那就做两张大图：一个 filter 了 HBB 看各个 GBL，一个 filter 了 GBL 看各个 HBB」 | 对比用两张图，不堆长页面 |
| part2 35:47–36:10 | 「按讲的顺序做」 | 优先级：Big Events → Portfolio → Concord → Lifecycle |
| part2 18:30–19:00 | 取数逻辑业务方整理好发群里 | 口径由业务方给，不猜 |

## 2. 可复用的现成实现（已上线、正在运行）

**后端** `GET /gps-sus/active-pn`：`smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/controller/GpsSusController.java:141`（`activePn(GpsSusActivePnQuery)`）。一次返回三块，正好等于原型需要的模型：
- `metrics`：GPS / SUS 两张卡（`GpsSusService.java:1831-1833`）
- `hierarchy`：**四层树** HBB → Offering → Sub Offering → L4，节点带 `quantity`，`value` 为编码路径（`GpsSusService.java:2034-2071`，`levelIndex > 3` 即止，根层按 `GpsSusScope.SCOPE_HBB` 排序）
- `inventory`：按 **businessUnit × 财年** 的行，含 `qtyOfPnCreation / qtyOfPnObsolete / qtyOfActivePns / bookingPerActivePns`（`GpsSusService.java:1887-1950`）；财年由每行 creation/withdrawal fiscal year 自动派生（`resolveActivePnFiscalYears`，:1839-1848）
- 筛选已支持 `bu` + 编码路径 `hierarchy`（`filterActivePnRows`，:1990-2007；路径前缀匹配 `activePnPathMatches`）

限制只在两处硬编码：
- `smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/dao/GpsSusActivePnMapper.java:43-45`：
  ```sql
  WHERE trim(lifecycle.business_unit) IN ('PCSD','ISG')
    AND trim(lifecycle.oh_l1_hbb) IN ('Global Product Services','Sustainability Services')
    AND trim(lifecycle.in_scope) = 'Y'
  ```
- `buildActivePnInventory` 里 BU 分组写死 `PCSD`/`ISG`（:1890-1891，其他 BU 直接被 `continue` 丢弃）

**前端** `C:\Users\63588\lenovo\ssg_sales_ops\src\views\GPSSUS\index.vue:529-706`：`pnInventoryView('overview'|'offering')`、`pnOfferingSelectedHierarchy` / `pnOfferingSelectedBusinessGroups`（默认 `['PCSD','ISG']`）/ `pnOfferingExpandedRoots`、`initializePnOfferingFilters()`、`loadPnOfferingData()`（请求 token 防竞态）。四级勾选级联重算与财年表/趋势图联动已完整——这就是需求截图里的界面。

## 3. 推荐方案（替代自建 /oh）

1. **后端先参数化，再共享（0.5–1 人日）**
   - `/gps-sus/active-pn` 增 `hbbs[]` 参数：不传 → 保持 GPS+SUS 白名单（现有页面零影响）；传值 → 按传入 HBB 过滤；`all` → 全放开。
   - `GpsSusScope.SCOPE_HBB` 补全 6 个标准 HBB 排序，否则 `indexOf` 返回 -1 导致根节点乱序。
   - `buildActivePnInventory` 的 BU 分组从写死 PCSD/ISG 扩成实际出现的值，否则全 HBB 场景丢失 ISU/MBG/VC/Others 共约 7,118 行。
   - Service Portfolio 侧暴露 `GET /service-portfolio/lifecycle/active-pn` 转调同一套 service；既有 `/oh` 废弃。
2. **前端搬界面（1.5–2 人日）**：把 `index.vue` 的 `pnOffering*` 状态 + 勾选级联 + 树 + 财年表 + trend 图整体迁到 Service Portfolio Lifecycle → By Offering，接口换新路径，样式按原型调。
3. **底部两个可配置模块**（筛选逻辑模板、EOL 流程）用配置表驱动，遵守录屏"预留模板、不改代码"，可放下一轮。

**结论与理由**：比原方案省约 2 天，且数据口径与 GPS/SUS 天然一致，避免同一指标两页两数。此前 `/oh` 方向只做了"取数"，没做"勾选 → 重算"这一层，所以前端无论怎样都渲染不出效果图（详见 `c7140f7c-24ae-4a3e-89ec-282af1331ae8` 的根因分析：扁平列表 + 全局 `LIMIT 50` + 无父子结构）。

## 4. 数据口径实测（`gps_pn_lifecycle_overview` / `gps_pnpl_master_data` 侧）

| 项 | 实测 | 需决定 |
|---|---|---|
| HBB | 6 个标准（Global Product Services / Sustainability Services / Digital Workplace Solutions / Hybrid Cloud Services / AI Solutions / Vertical Solutions）+ `Vehicle Computing Service`（仅 1 个码）+ `(blank)` 293 个脏码 | 建议白名单 6 个标准 HBB；脏码被 `IN (...)` 天然排除 |
| business_unit | ISG 266,131 / PCSD 58,865 / **ISU 3,552 / MBG 3,461 / VC 68 / Others 37** | 现页面只有 PCSD+ISG 两个勾选框；全 HBB 后必须扩 BU 分组 |
| HBB × BU 组合 | 每个 HBB 实际横跨多 BU（如 Global Product Services = ISG+MBG+PCSD，Vertical Solutions = ISU+Others+PCSD） | 不能按 HBB 反推 BU |
| 树规模 | `(oh_l1_hbb, oh_l2_offering, oh_l3_sub_offering, oh_l4)` distinct = **257** 个 L4 叶子 | 一次下发即可，不需要 50 条上限 |
| 财年 | 每行带 creation/withdrawal fiscal year，inventory 自动生成 4 行（FY2324–FY2627） | 沿用，不需要"按财年筛 OH" |

## 5. 待用户确认（阻塞开工的三项）

1. **HBB 是否 6 个全放开**（含 AI Solutions、Vehicle Computing Service）？
2. **BU 维度用哪几个**：全 6 种，还是 PCSD(IDG)/ISG/MBG 三种？
3. **同意"复用 active-pn"而不是重做 /oh** 吗？

确认后的实施顺序：先加 `hbbs` 参数 → 跑 GPS/SUS 原页面回归确认默认行为不变 → 再提 MR；前端随后迁移。

## 6. 踩坑备注

- `.agents/worktrees/service-portfolio-admin/.tmp/` 目录在部分执行上下文里不存在，直接 `writeFileSync` 写探针文件会报 `ENOENT`，需先建目录。
- 前端 `GPSSUS/index.vue` 中 `byOffering` 关键字命中为 0，相关实现全部以 `pnOffering*` / `pnInventoryView` 命名，按 `byOffering` 搜索会误判"未实现"。
