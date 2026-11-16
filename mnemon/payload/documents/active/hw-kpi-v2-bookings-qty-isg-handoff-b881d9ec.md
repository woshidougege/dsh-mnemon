---
id: "b881d9ec-57e3-4636-bf0a-5c10604822a9"
title: "HW KPI 单位方案 v2：Bookings 动态单位 + Qty 主图内嵌 ISG 小图（已定决策与实施 handoff）"
description: "用户已拍定 Bookings 按数量级动态单位、Qty 走 ⑨（主图不变 + ISG 右下角内嵌小图）后的 v2 方案：接口 {unit, unitScale, format:AUTO, inset} 契约、块级选档 vs 值级降级的区分与阈值、轴不混排与跳档代价、toMillions/toThousands 仅留 1 位小数的传输层精度隐患、前后端改动清单（含行号）、两个 MR 实施顺序。"
status: "active"
created_at: "2026-11-16T09:59:48.430Z"
updated_at: "2026-11-16T09:59:48.430Z"
content_hash: "d71d82dcb47eb80bd580bc5d924e5d8a5d4d1ec2eab32e197d2b1629885c2479"
source_paths:
  - ".tmp/hwqty-mockup/unit-design.md"
session_ids:
  - "mnemon-run-v1-40335ba3-a048-4c35-aafd-216b910a0465"
memory_body_ids:
  []
---

# HW KPI 单位方案 v2：Bookings 动态单位 + Qty 主图内嵌 ISG 小图（实施 handoff）

> 关联文档：`9b904808-5759-4d8a-af76-2e6d691ae8f4`（Bookings 单位 M→B 与 K/M/B 自适应，**待决策版**，本文件取代其"待选"结论）、`32e1bb91-5509-483c-b11d-ff85d7b016bd`（ISG 贴底的量化根因与五个候选方案）、`e5fe9ed8-0b73-44be-b883-f39a4972b6a9`（Qty 单轴按量级压缩 = 方案 A）。口径与数据事实不在此重复。

## 1. 用户已拍定的两条（取代前文的待定项）

- **HW Bookings：按数量级动态单位**，不是锁死 `$B`。用户原话（checkpoint 事件 1400）："Bookings 是按数量级动态单位"。
- **HW Qty：按业务已选的 ⑨ 方案**——主图不变，**ISG 挪到右下角内嵌小图**。用户原话（checkpoint 事件 1400）："QTY那个用户不是选择 右下角小图显示吗 ？"

助手此前把"Qty 主图轴用 M 还是 K"误列为待定项，已更正：那只是主图刻度标注方式，**内嵌小图照加、不受影响**。

| 指标 | 主图单位 | 标签 | 内嵌小图 |
|---|---|---|---|
| HW Bookings | 按数量级动态（$B/$M/$K） | 每值再降级 | 无（ISG 占 Total 6%~15%，主图看得见） |
| HW Qty | 按数量级动态（常规即 Qty(M)） | 每值再降级 | **ISG（K）右下角** |
| HW AUR | 固定 `$` | 固定 2 位 | 无 |

## 2. 接口契约：后端返回"块级单位 + 数值"，不返回格式化字符串

用户先问"后端返回单位是不是比较好"，结论：**对，但只返回块级单位与原始数值，不要每格拼成 `"15.1B"`**。理由：Chart.js 要数字才能画，字符串进入图表就得反解；否则标签、tooltip、Show Data、导出各写一套，迟早对不上。分工 = **后端定单位类别与块级基准刻度，前端只负责把数字缩写成 B/M/K**。

```json
{
  "id": "hw_bookings",
  "unit": "$B",              // 轴单位，块级固定，不随筛选变
  "unitScale": 1000000000,   // 1 个单位 = 多少原始美元
  "format": "AUTO",          // AUTO = 标签按量级降级
  "inset": null,
  "rows": [{ "bu": "ISG", "FY2627Q2": "2.2519" }]
}
```

Qty 块：主图 `unit: "Qty (M)"`，另加 `inset: { series: "ISG", unit: "Qty (K)" }`。

## 3. "动态单位"分两级（关键区分：动态 ≠ 混排）

- **第一级 · 块级（决定轴）**：取该块所有值中绝对值最大者 → `≥1e9 → $B`、`≥1e6 → $M`、`≥1e3 → $K`。Bookings 全量最大 15.06B，轴为 `$B`；筛到某小 market（最大约 8 亿）时轴自动变 `$M`。
- **第二级 · 值级（只改标签）**：每个值按自身量级降档。Bookings 同图可出现 `15.1B / 9.87B / 2.25B / 558M`；Qty 里是 `13.3M / 4.95M / 52.6K`。

**轴刻度只用块级单位，绝不混排**（Bookings 全是 B、Qty 全是 M），否则读者判断不了斜率和比例，这是"动态"与"混排"的本质差别。

格式化规则（只在前端 utils 实现一份）：还原到原始量级 → 固定阈值选 B/M/K → **≥10 保留 1 位小数，<10 保留 2 位**。示例：Bookings `15.0632→15.1B`、`9.8746→9.87B`、`0.5579→558M`（降一级）；Qty `13.2539→13.3M`、`0.052575→52.6K`。小数位是硬约束：1 位会把 ISG 的 557.9/582.5/593.1 全压成 `0.6B`。

**必须接受的代价**：块级单位会随筛选跳档（15B→800M）。缓解＝档位只设 K/M/B 三档、轴标题始终带单位（`$M`/`$B`）、同页同时刻只有一种单位。不想跳就只能锁死每卡单位（Bookings 永远 $B、Qty 永远 M），二者不可兼得。

## 4. 必须一起修的隐患：传输层精度已丢

后端 `BookingsDisplayFormat.toMillions/toThousands` **只留 1 位小数**：Qty 在 M 口径下 ISG 是 `0.1`，前端再格式化也变不出 `52.6K`。因此本次必须顺手把返回精度放到 **6 位有效数字**（或直接给原值），否则"按量级自动判断"根本跑不起来。

## 5. 改动清单

**后端**（`smartop-service/admin/src/main/java/com/lenovo/admin/bookings/service/BookingsService.java`）
1. `:776-802` `hwWideCell` —— 不再拼字符串，返回全精度数值。
2. `:731-773` `hwWideBlock` —— 加"算块内 max → 选档 → 写回 unit/unitScale"，并注入 `unit / unitScale / format / inset`。
3. `BookingsOperationalKpiSummaryResp.MetricBlock` —— 新增这 4 个字段（builder，向后兼容）。接口 `GET /admin/bookings/hw-kpi-summary`（`BookingsController.java:128`）。

**前端**
1. 新增 `utils/formatNumber.ts` 的 `formatCompact`（K/M/B 三级 + 负数 + 千分位），标签/tooltip/表格/导出共用。
2. `src/views/Bookings/components/miniTable.vue`：`:79-99` 三个 format 函数换成 `formatCompact`；`:207` 轴标题改读 `block.unit`；新增 `inset` 渲染（第二个 canvas + 独立 K 轴），并按 `hideInMain` 把主图里 ISG 的标签置空，否则贴底的 49.3K 会挤在 X 轴上。
3. `src/views/Bookings/otherKpis.vue:165-193`：删掉按标题匹配 `$M`/`K` 的硬编码，改读 `metric.unit`。
4. `src/views/GPSSUS/index.vue:2296-2301`：同样删掉三元表达式。

**不要跟着变**：Show Data 表格与导出仍用 `9,874.6` 这类 M 值，图与表两套格式是本项目常态。AUR 卡（$/台）不受影响，但分子是 Bookings，口径别一起改。另有 Y 轴 `beginAtZero` 老问题（grace 造成底部 -2,000，换 B 后 -2B），顺手修掉，别让领导以为有负 bookings。

## 6. 状态与下一步

- **代码尚未改动**（checkpoint 截止时用户先答"那就改这 2 个吧"、随即改口"先别改哈"追问自动判断，之后才定 v2）。
- 方案文档已产出并更新至 v2：`.tmp/hwqty-mockup/unit-design.md`。
- 计划顺序：先改后端精度与 4 个字段（admin-release MR）→ 再改前端格式化与 `inset` 渲染（dev 分支 MR），共两个 MR。等待用户确认剩余细项（ISG 降级显示 `558M` 还是 `0.56B`；块级跳档是否接受）后开工。
