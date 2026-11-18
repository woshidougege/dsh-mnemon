---
id: "cf1cbe31-a3d4-41fc-9c71-4c27164ed795"
title: "HW KPI 图标签“全都不显示”复盘：漏解构 datasetIndex 与真实卡片尺寸下压线策略失效"
description: "HW Bookings/Qty 主图一个标签都不画的两个叠加原因：自绘插件取数据项漏 datasetIndex 致 afterDatasetsDraw 抛 ReferenceError；以及真实卡片 490 CSS px 宽、绘图区约 430×110 px 下“不压线”策略只能标出首季与峰值（宽幅 mockup 高估可放数量）。含修复档位、MAX_LABELS_PER_SERIES=5、提交 b050a3df/7d693f7b 与 MR !1090/!1070/!1088 状态、本地 5175+8082 实跑数值与 TST 未验证边界、bsk 会话与点击坑。"
status: "active"
created_at: "2026-11-18T07:48:16.694Z"
updated_at: "2026-11-18T07:48:16.694Z"
content_hash: "4c3a0e58aeab7f24e24a47fb507c8749877ddcf6747b53bff8399f3b977029fa"
source_paths:
  - "src/views/Bookings/components/miniTable.vue"
  - ".tmp/local-hw-final.png"
  - "vite.config.ts"
  - "src/utils/request.ts"
session_ids:
  - "mnemon-run-v1-5e739a02-263d-4aa3-a4b9-7e8246abab6f"
memory_body_ids:
  []
---

# HW KPI 图“全都不显示”复盘：漏解构 datasetIndex + 真实卡片尺寸下“不压线”策略失效

> 同一张卡片的相邻记录：带单位/自绘双字号标签落地见 `0200947d-78c0-4b83-905e-cacea7c7ad11`；inset 小图读不全与 Bookings 轴 \$M 口径见 `f5b15acb-8bb5-49fa-9f28-4958f4928c5f`。本文只记录本轮“标签整体消失”的两个叠加原因、修复形态与 MR 状态。

## 1. 现象与定位

用户反馈「现在好严重啊 全都不显示了」——HW Bookings / HW Qty 主图上连一个数值标签都没有；TST 上此前也只剩两三个数。

## 2. 根因一：自绘标签插件抛错，整块画不出来

`src/views/Bookings/components/miniTable.vue` 自绘标签插件取数据项时漏字段，下面某一档又引用了它：

```js
// 第 197 行
const { element, dataIndex, text, unit, priority } = item   // 少了 datasetIndex
// 第 235 行
return !touchesLine(box, datasetIndex)                      // ReferenceError: datasetIndex is not defined
```

只要某条线首次匹配到「只压自己的线」这一档，异常即抛出；`afterDatasetsDraw` 钩子抛错会让整个绘制中断，结果是一张标签都画不出来。该缺陷由上一笔 `9184ec1d`（标签分三段退让）引入，浏览器控制台可复现两条 `ReferenceError: datasetIndex is not defined`。

## 3. 根因二：策略在真实卡片尺寸下根本行不通

真实卡片在 DPR=2 下仅 **490 CSS px 宽**，绘图区约 **430×110 px**；11 个季度、4~6 条线，线距不到 10px。硬要求“一根线都不压”，实际只有首季与峰值能标出来，其余全部落空——这正是 TST 上只剩两三个数字的直接原因。

此前的“效果图”都是按 1560~1980px 宽手写的 HTML 画的，同一策略下能放下 21~25 个标签，因此长期没暴露差异。**教训：标签排布类改动必须在真实卡片尺寸（含 DPR）下验证，宽幅 mockup 会系统性高估可放数量。**

## 4. 修复形态

- 补上 `datasetIndex`，插件不再抛错；
- 按优先级分流：关键点（首季、峰值、最近三季）放不下时**允许压线**，更早的季度只在不压线时补位；
- 每条线最多 5 个标签：`MAX_LABELS_PER_SERIES = 5`（`miniTable.vue:196`，判定在 203 行、计数在 261 行），避免把整张卡填满。想更疏只改这个常量。

## 5. 提交与 MR

| 分支 / 目标 | 提交 | MR | 状态 |
|---|---|---|---|
| `dev/renxp4-hw-kpi-inset-modes` → `develop` | `b050a3df`（基于 `3fc466c8`） | **!1090** | opened |
| `dev/renxp4-hw-kpi-unit` → `master` | `7d693f7b`（cherry-pick；父 `9184ec1d`） | **!1070** | 本轮更新，opened / can_be_merged |
| develop 侧旧 MR | `3fc466c8` | !1088 | 已被合并 |

远端为 `gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops`。旧 `!1088` 合并后 develop 侧另开 `!1090`。

## 6. 验证与边界

- `npm run type-check`（`vue-tsc --build`）通过；
- 本地实跑：自起 vite dev（`localhost:5175`，另一个 worktree 占着 80 端口），后端 `127.0.0.1:8082`，用 bsk 连本机 Chrome 实例 `62af6ee9`，从 `/spark/bookings` 进 Operational KPIs → BU 维度 → 滚到 HW Bookings 截图（`.tmp/local-hw-final.png`）；
- 页面实取数值：HW Bookings Total `$10.5B / $13.0B / $15.0B`、PCSD `$7.5B / $7.0B`、ISG `$1.1B / $0.8B / $0.6B`；HW Qty Total `25.4M / 28.1M / 25.6M`、PCSD `15.6M / 15.1M / 15.8M`、MBG `10.5M / 11.9M / 11.6M`、ISG 按自身 K 显示 `52.6K / 63.6K / 44.5K`，右下角「ISG · K」三态角标正常；
- 另用 evaluate 直接取组件源码确认服务端下发的是新版本：`{hasCap:true, capValue:"5", hasDatasetIndexFix:true}`，排除浏览器缓存旧模块；
- **边界**：以上是本地工作树（未部署）验证，TST 需等 !1070、!1090 合并并 CI 部署后复核。

## 7. 操作坑（可复用）

- bsk 会话有时限（默认 idle 5m），超时报 `session not registered or already stopped`，需 `bsk session start` 重开；重开的浏览器窗口不带登录态，页面被踢到 `/noAccess`（cookie 里 `Authorization` 仍在、`userInfo` 为空），此时无法再截取需登录的图。
- 严格观察模式下脚本执行容易报 “strict observation changed before this action”，改成分步 `read`/`extract` 或把事实写成 expect 检查更稳；`bsk click` 常报 “pointer target is covered”，用 `evaluate` 内 `element.click()` 更可靠。
- 选择器宽泛时会报 “selector matched N elements”，需换成 VOM ref 或更窄的选择器。
