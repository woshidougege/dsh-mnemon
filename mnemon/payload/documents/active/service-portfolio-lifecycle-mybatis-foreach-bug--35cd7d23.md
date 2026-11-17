---
id: "35cd7d23-05ba-4fce-a08c-8c42e977e732"
title: "Service Portfolio Lifecycle 接口修复：MyBatis foreach 索引绑定 bug 与 ohPaths/hbb/bu 参数落地（ssg-sales-ops admin, MR!568）"
description: "前端反馈的两条问题已修：ohPaths 500 的根因为 MyBatis foreach 不改写 #{ohp[0]}；三接口统一 hbb+bu+ohPaths，含本地实测数字、ohPaths 参数契约、分支/MR!568 与本地启动验证注意点。"
status: "active"
created_at: "2026-11-17T08:24:11.705Z"
updated_at: "2026-11-17T08:24:11.705Z"
content_hash: "999367460b76f2ba76cfe266e4165debc134baeab4dddf5a7d6a74629af1b2c3"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/dao/ServicePortfolioReadMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java"
session_ids:
  - "mnemon-run-v1-7419ff1d-a780-4fb0-b752-13ab424e73d7"
memory_body_ids:
  []
---

# Service Portfolio Lifecycle 接口修复：OH 路径筛选 + hbb/bu 参数（ssg-sales-ops admin）

实施记录与前端交接说明。前置分析（OH 树、财年口径、原型对齐）见文档 `c7140f7c-24ae-4a3e-89ec-282af1331ae8`，本文只记录该分析落地后的实现与实测结论。

## 1. 前端反馈两条 + 结论

| 反馈 | 结论 |
|---|---|
| Lifecycle 接口缺 `bu` 参数 | 已补；overview / utilization / offering-trend 三接口参数统一为 hbb + bu + ohPaths |
| 一传 `ohPaths` 就 500 | **后端真实 bug**，不是前端传错参数；前端参数格式无需改动 |

## 2. 根因：MyBatis foreach 不改写 `#{ohp[0]}` 索引写法

Mapper 里用 `#{ohp[0]}` 取路径层级时，MyBatis 的 `<foreach>` 只把 `#{ohp}` / `#{ohp.xxx}` 改写成 `__frch_ohp_0` 形式，**索引下标写法被原样保留**为参数名，运行期从参数对象取不到值，直接抛 `ReflectionException`，接口返回 500。

最小复现（MyBatis 3.5.17，直接解析 SQL 看参数映射）：

- `#{ohp[0]}` → 参数名仍是 `ohp[0]`，取值抛 `ReflectionException`
- 改成 Map 元素 + `#{ohp.l1}` → 参数名变成 `__frch_ohp_0.l1`，取值正常

修复做法即后者：`ohPaths` 的每一层用 Map 键（`l1`/`l2`/`l3`…）承载，SQL 用 `#{ohp.l1}` 这类属性路径访问。涉及文件：

- `smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/dao/ServicePortfolioReadMapper.java`
- `smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java`
- `smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java`

## 3. 对外参数契约

- `/lifecycle/overview`：hbb、bu、ohPaths
- `/lifecycle/utilization`：hbb、bu、ohPaths
- `/lifecycle/offering-trend`：hbb、bu、ohPaths、fyFrom、fyTo
- `ohPaths` 格式：`active-pn:<节点 value 原样回传>`，多条路径逗号分隔，支持 1~4 层；非法 base64 直接忽略筛选、返回全量而不报错。
- By Offering 表仍取 `/lifecycle/overview` 的 `byOffering`；`offering-trend` 与 `utilization` 返回同一财年序列，仅支持 fyFrom/fyTo 裁剪，不要用它们渲染 By Offering 表。

## 4. 本地真实库实测（2026-11-17）

| 请求 | 结果 |
|---|---|
| overview 无参 | 332,114 PN / 277,916 released / 15 行 byOffering |
| `ohPaths=GPS` | 309,429 PN，3 行 byOffering |
| `ohPaths=GPS + Digital Workplace Solutions` | 320,721 PN（多路径并集生效） |
| `ohPaths=GPS/Support Services` | 299,526 PN（精确命中） |
| utilization `ohPaths=GPS` | 6 个财年 |
| offering-trend `hbb=GPS&ohPaths=…&fyFrom=FY2324&fyTo=FY2526` | 3 行 |
| 故意传坏 base64 | 忽略筛选返回全量，不报错 |

## 5. 交付与验证环境注意

- 分支：`feature/sp-api-fix-admin`（基线 `origin/admin-release`），提交 `ff650a1b7` "fix(service-portfolio): OH 路径筛选改用 Map 绑定，offering-trend 补 hbb/bu"（3 files, +49/-27）。
- MR：https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/568 → `admin-release`，`can_be_merged`、无冲突、3 个文件。
- 上一轮 MR !564 已 merged 且源分支被删，因此本轮重新推分支、新建 !568；不要拿 !564 当"仍开着"。
- 本地起服务验证时，Admin 能启动但 RabbitMQ 连不上会持续刷 `org.springframework.amqp.AmqpConnectException: java.net.ConnectException: Connection refused`（`AmqpOutboundEndpoint` 定时重试），属本地缺 MQ，不影响 HTTP 接口验证；日志约 6,000 行会淹没输出，需要先按关键字过滤再读。
- 本地调试用 `sp-run.args`（classpath file，约 490 条）启动，端口 18085；验证完成后要确认端口已释放、进程已 kill。
