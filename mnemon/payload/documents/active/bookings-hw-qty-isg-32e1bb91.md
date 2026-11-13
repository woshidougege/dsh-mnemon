---
id: "32e1bb91-5509-483c-b11d-ff85d7b016bd"
title: "Bookings HW Qty 图表：ISG 曲线贴底的成因与可见性方案（双轴/拆图/图例聚焦）"
description: "在方案 A（按量级压缩标签）之上，ISG 趋势仍贴底：250 倍量级差、跌幅仅约 0.6 像素的量化根因；B 双轴（右轴量程人为、需配轴色防误读）、C 拆图、图例点击聚焦、卡内嵌小图、指数化五个候选的代价与推荐；已交付五方案对比板/评审图、待业务定方向的实施待办。"
status: "active"
created_at: "2026-11-13T08:01:52.258Z"
updated_at: "2026-11-13T08:01:52.258Z"
content_hash: "f7ac9a2c3e560e4ee7146ace8886cc35521e42a0e62409e9bb59d94c13a4117d"
source_paths:
  - ".tmp/hwqty-mockup/mockup.html"
session_ids:
  - "mnemon-run-v1-51abbfb3-3465-4af0-85e0-3280b3cab788"
memory_body_ids:
  []
---

# Bookings HW Qty 图表：ISG 曲线“贴底”的成因与可见性方案

> 关联文档：`e5fe9ed8-0b73-44be-b883-f39a4972b6a9`（Bookings HW Qty 图表单位：单轴按量级压缩（方案 A）设计与效果验证）。那篇记录“数值长度”问题（全 K / 全 M / 方案 A / 双轴 / 拆图 的对比与 A 的落地细节）；本文只记录后续子问题：**在方案 A 之上，ISG 趋势仍看不清，怎么办**。两篇结论可叠加。

## 1. 根因（量化，非样式问题）

- 生产库 `mv_bookings_agg`、`kpi='HW Qty'`、排 NEC：ISG 单季约 5 万台件，PCSD / MBG 各约 1,300 万台件，**相差约 250 倍**（ISG 占比约 0.4%）。
- ISG 自身从 54.4K 跌到 44.5K（跌 18%），在约 300px 高的绘图区里只值 **约 0.6 个像素**。
- 因此：换 K、换 M、加后缀、压缩标签都改变不了这条线；只有**换坐标轴或换图形**才能解决。ISG 与 PCSD/MBG/Total 共用一根线性轴时，它必然贴着底部画成一条“平线”。

## 2. 候选方案与代价

| 方案 | 做法 | 效果 | 代价 / 风险 |
|---|---|---|---|
| **B 双 Y 轴** | ISG 挂右轴（0~70K 量程），PCSD/MBG/Total 左轴（M） | ISG 波形立刻清晰 | 约半天；**右轴量程是人为设定**：设 0~70K 时 ISG 会“爬”到 PCSD 上方，设 0~200K 又缩回底部 → 高度具有误导性，必须用轴配色 +“ISG 右轴”图例说明，仍承担误读风险 |
| **C 拆两张图** | 上：PCSD / MBG / Total（M）；下：ISG 单独（K，0 起） | 各自成轴、不误读，ISG 的 Q2 高点 / Q4 低点一眼可见（波动约占 1/6 高度） | 约 1 天；Qty 卡片要变高，或在 2×2 栅格里多占一格；失去跨 BU 一眼比量级 |
| **图例点击聚焦** | 默认仍是当前图；点图例“ISG”时坐标轴按 ISG 范围重算、其他线变灰，再点恢复 | 不动布局、不误读 | 约半天，是三种改图方案里改动最小的（`miniTable.vue` 加 `legend.onClick`，该图目前没有 legend 点击定制）；缺点是用户不点就发现不了，图例旁需加提示 |
| **卡内嵌小图** | Qty 卡片右下角放 160×90 的 ISG 迷你图，自带坐标轴 | 布局不动、不误读 | 空间紧，字号会缩到 8px 左右；响应式上下浮动时定位不稳 |
| **指数化（首季=100）** | 四条线一起比涨跌 | 能同时比较趋势 | 绝对量消失，业务通常不接受 |

推荐顺序：**能改布局 → 方案 A + C**；**不能改布局 → 方案 A + B（配轴配色）**；只想最小改动 → 图例点击聚焦。A 解决“数字太长”，B/C 解决“ISG 看不清”，两者不冲突。

## 3. 已交付的评审材料（仅作参考，非正式代码）

- `.tmp/hwqty-mockup/hwqty-all-options.png`：五方案对比板（① 全 K 现状、② 全 M、③ 方案 A、④ 双 Y 轴、⑤ 拆开画，右下“怎么选”），2 倍图 2960×2360，可直接贴 PPT / 发领导。五格点位比例配色完全一致，差别只在坐标轴与数字格式。
- `.tmp/hwqty-mockup/isg-fix.png`：B（双轴）vs C（拆图）对比。
- `.tmp/hwqty-mockup/hwqty-units.png`：现状全 K vs 方案 A（左 13,253.9，右 13.3M / 15.6M / 52.6K / 28.9M，轴 0~35M、标题 Qty (M)）。
- 源文件 `.tmp/hwqty-mockup/mockup.html`：改数据、配色、时间区间直接改它；用本地 Chrome 无头渲染 + 项目 `node_modules/chart.js` / `chartjs-plugin-datalabels` 出图，非手工拼图。

## 4. 当前状态与待办

- **待业务/领导决策**：要“数字别太长”（选 A 即可），还是要“ISG 趋势也得看清”（需 A + B 或 A + C）；图上 M、K 后缀混排能否接受（不能接受就只能拆图 C）。
- 定方向后实施：A 只动 `miniTable.vue` + 两处 `valueDisplay` 配置（`otherKpis.vue`、`GPSSUS/index.vue`），后端不动、已合分支不必回滚；先提 develop MR 验证，再提 master。
- 建议实施前先做可点击原型（含 `Show Data` 表格里的单位展示）在真实页面验证，再改正式代码。
- 已知遗留（本子问题之外）：`display:'auto'` 会隐藏互相压盖的标签（13 季度 × 4 线最多 52 个标签，现在本就读不全）；单 BU 过滤时 Total 行与该 BU 数值重合会出现两条重叠线（`hwWideBlock` 无条件追加 Total）。

相关代码位置（复用关联文档中的路径）：后端 `BookingsService.java:796-801`（HW Qty 统一 /1000）、`BookingsService.java:736-770`（rows = BUs）；前端 `src/views/Bookings/components/miniTable.vue:112-213`（单 Y 轴，单位取 `valueDisplay.unit`，:207）、`src/views/Bookings/otherKpis.vue:178-181`、`src/views/GPSSUS/index.vue:2296-2301`（写死 K）。
