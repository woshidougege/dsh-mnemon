---
id: "06e3e0aa-21f8-4d59-bff8-bba8abb2f44b"
title: "ChatBI 使用分析页指标区改造：三列漏斗 + Accuracy Rate 改 success-only 口径（ssg-sales-ops 实施记录）"
description: "已实施：user-logs 页 4 张平铺指标卡改为三列漏斗、Accuracy Rate 口径改为仅在成功轮次内已评级（Graded 替换 Reviewed）、Route failed 列改名 Routing failed 并补定义 tooltip；含后端新增 success* 字段、mapper finalSuccess 列、JRebel 热更限制与未提交/UU 冲突遗留。"
status: "active"
created_at: "2026-11-20T02:42:24.806Z"
updated_at: "2026-11-20T02:42:24.806Z"
content_hash: "ccda45279622ecf05c444f33b0c5dbac8275ae2852b79c555ca4904abe487d8c"
source_paths:
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue"
  - "ssg_sales_ops/src/views/Setting/reports/user-logs/styles.less"
  - "ssg_sales_ops/src/constants/feedbackMenuCatalog.ts"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java"
  - "smartop-service/admin/src/main/resources/mapper/chatbi/ChatbiEventQueryMapper.xml"
session_ids:
  - "mnemon-review-v1-2c2b387e-c3aa-4dca-a423-70d29fd451f4"
memory_body_ids:
  []
---

# ChatBI 使用分析页指标区改造：三列漏斗 + 准确率改为 success-only 口径（ssg-sales-ops）

记录 2026-11-20 完成的一次已落地改动（代码已写入、未提交）。前置的背景文档是 `02b68005-05ee-4b33-b617-f9c3b01164f4`（口径释疑 + 当时未实施的 Route failed 降级 A/C 方案）；本文档记录最终采取的实施路径，**与那份文档的"待拍板/未实施"状态不同，已按本次方案落地**。

## 1. 触发问题（决策由来）

原页面 4 张平铺指标卡并排，Accuracy Rate 显示 `0.0%`，副文案 `Correct 0 / Reviewed 21`。领导把 0.0% 读成"整体准确率 0%"，真实分母只是 21 条已评审轮次（总问题数 470，成功 450）。根因是**排版没有体现层级** + **"Reviewed" 这个词让人以为是全量评审**。

由此确立两条不可回退的口径原则：
- Accuracy Rate 的分母只能是"成功轮次里已判对错的"，失败轮次没有答案可判，不得进分母；页面上必须显式写出分母是什么（`Graded 21 of 450 successful turns`）。
- 彻底移除 "Reviewed" 字样，改用 **Graded**，并在卡片标题加 `success only` 徽标。

## 2. 布局：4 张平铺卡 → 三列漏斗

三列用箭头串联，让"先成功、再判准确"成为可见层级：`470 问 → 450 成功 / 20 失败 → 成功里 21 条判过错 → 准确率 0%`。

| 列 | 卡片 | 关键点 |
|---|---|---|
| 1 | Question count（`.metric.hero`）+ Users involved（`.metric.amber`） | 口径不变，仅重新编组 |
| 2 | Final success（`.metric.blue`）+ Failed（`.metric.rose`） | Failed 副文案直接写 `4.3% · not graded for accuracy`，当场说清这 20 条不进准确率；百分比口径未变，仍是 `(total-failed)/total`，含非 SDQ 轮次 |
| 3 | Accuracy Rate（`.metric.violet`）+ Review coverage（`.metric.ghost`） | Accuracy 加 `success only` 徽标；副文案两行：`Correct 0 · Wrong 21` / `Graded 21 of 450 successful turns`；标题挂 `accuracyTitle` tooltip |

卡片下方一行 `mf-note` 说明准确率只在成功轮次上计算；仅当库里确实存在"失败但带评审"的轮次时才追加渲染：`N failed turns also carry a review; they stay out of the accuracy denominator`。

结构和样式落点：
- 模板：`ssg_sales_ops/src/views/Setting/reports/user-logs/components/views/QueryView.vue` 的 `<SectionCard title="Core metrics">` 内改用 `.metric-funnel / .mf-lane / .mf-arrow`（含 `.mf-arrow-review`）。该文件里 `MetricCards`、`showAgentSuccessRate` 的引用数已归零（grep 计数 0），指标卡改为页面内联实现，文件约 1067 行。
- 样式：`ssg_sales_ops/src/views/Setting/reports/user-logs/styles.less` 指标卡区块（约 :365-399）——`.metrics` 四列栅格 + 1200/880/560 三档断点、`.metric` 悬浮位移与 `::after` 装饰圆、`.metric.hero` 渐变（含 `&[data-theme="dark"]` 深色覆盖）、`.delta.up/.down/.flat`。

## 3. Route failed 列 → Routing failed

不只是改名，还补了定义，因为它与 Failed 列**互相独立**（同一轮次可同时落入两列，也可都不入）：

- 列名改为 **Routing failed**，表头带 ⓘ，tooltip 文案：*Turns that called route_report this turn and failed. Independent from Failed — a turn can sit in both columns, or in neither.*
- 数字仅 `>0` 时再挂二级 tooltip：`{routeFailed} of {total} turns tried to route a report and failed`；为 0 时显示 `—`（不再显示 0）。
- 顶部汇总条同步改名。
- 表格状态映射处（`QueryView.vue` 约 :283）`route_failed` 仍拼 `· route failed` 后缀，未改内部状态名——只改面向人的文案，避免动后端枚举。

## 4. 后端改动（未提交）

- `smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java`：review 汇总新增 `successCorrect / successWrong / successNotApplicable / failedReviewed / successGraded / successAccuracyRate / successCoverageRate`。**旧字段（`accuracyRate` 等）全部保留**，理由：FeedbackView 复用同一响应，删字段会连带破坏它。
- `smartop-service/admin/src/main/resources/mapper/chatbi/ChatbiEventQueryMapper.xml:158`（`sessionReviewStats`）：带上与 `sessionSummary` 完全一致的成功判定，多返回一列 `finalSuccess`；`GROUP BY` 补到 6 列。理由：只有 SQL 里的成功判定与汇总口径同源，`successGraded` 的分母才站得住。
- 编译验证：`mvn -o -q -T 4 -pl smartop-service/admin -DskipTests compile` → EXIT=0。

## 5. 联调环境事实（踩过的坑）

- 前端不能用 `pnpm dev` 起：会先跑 `pnpm install` 并因 esbuild 的 ignored-builds 报错退出。可用替代：`node node_modules\vite\bin\vite.js --port 5173 --strictPort`。页面地址 `http://localhost:5173/Setting/reports/user-logs`（默认 tab=query）。
- 代理分流：`/api/auth` → 测试环境 `ssgspark-tst.lenovo.com`；`/api/admin` → 本地 `127.0.0.1:8082`。
- 本地后端起来后 `/actuator/health` 返回 503、`/jrebel/status` 返回 404，属"进程在但未就绪"，不代表前端联调失败。
- **JRebel 限制**：Java class 可热更；MyBatis mapper XML 一般是启动期解析，改了 `ChatbiEventQueryMapper.xml` 通常要 reload/重启。快速判据：页面上 Accuracy 显示 `Graded 21 of 450` 即 XML 已生效；显示 `—` 或 `Graded 0 of 450` 说明新代码读不到 `finalSuccess` 列，需要重启。

## 6. 交付状态与遗留

- 三处改动（前端 QueryView.vue、后端 service、mapper XML）**均未提交**，等用户统一处理。
- 前端仓库 `src/constants/feedbackMenuCatalog.ts` 处于 `UU`（未解决合并冲突）状态，文件内已无冲突标记（0 处），疑为解决完未 `git add`；同时 `develop...origin/develop [behind 112]`。**提交前必须先清掉这个 UU，否则 commit 会被卡**。
- 观察（未核实，非本次改动）：后端各 `application-lmp-*.yml` 连接池配置写法不统一，`application-lmp-sy-prod.yml` 用 `max-active: 20`，其余 profile 用 `maxActive: 20`（camelCase）；Spring Boot 绑定不区分这两种写法，但混用值得在下次改配置时统一。
