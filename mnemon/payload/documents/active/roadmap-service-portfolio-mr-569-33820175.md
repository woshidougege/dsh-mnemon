---
id: "33820175-1b4b-4478-8db3-98ad05e5339c"
title: "Roadmap 时间轴改以项目卡为排期主数据（Service Portfolio，MR !569）"
description: "2026-11-17 决策与实现：/roadmap 每季度 = 项目卡（roadmap_quarter/roadmap_index 排序）+ 手工运营节点，sp_sp_roadmap_item 仅留非项目固定流程；含否决方案理由、开发库 sp_sp_project_card 3 条有效数据与 BU 值域核对、构建已过但本地未实跑（RabbitMQ 不通）、git add -A 误提交 %SystemDrive% 的处理。"
status: "active"
created_at: "2026-11-17T10:20:33.535Z"
updated_at: "2026-11-17T10:20:33.535Z"
content_hash: "cd861c6d04cbea8e5ce0fb7714af951f50ed852dbc73bd70891634683de85e5d"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/controller/ServicePortfolioController.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/pojo/entity/SpProjectCard.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/dao/SpProjectCardMapper.java"
session_ids:
  - "mnemon-run-v1-7faae889-594d-460a-86cb-89a6246ed7b1"
memory_body_ids:
  []
---

# Service Portfolio 主页 Roadmap 改为以项目卡为排期主数据（MR !569 增量，ssg-sales-ops）

2026-11-17。承接前端诉求："先把主页的 roadmap 跟配置里的项目关联上吧，不然主页啥都没有"。

## 1. 问题：同一份"时间轴内容"有两个数据源

- `sp_sp_project_card` 上有 `roadmap_quarter`（如 `FY26/27 Q3`）+ `roadmap_index`（1-4），业务在配置页填的时候语义就是"放时间轴哪个格子的第几位"。
- 后端另有 `sp_sp_roadmap_item` 表专门存 roadmap 条目，但**没有任何界面能维护它**（此前审计确认该表 0 行）。
- 两者都声称自己决定"时间轴上显示什么"，必然漂移：配置页改排期时间轴不变；有人手工改库补 timeline 配置页又看不到。

## 2. 决策（用户拍板"改一下吧"）

**项目卡作为时间轴的唯一真相源**；`sp_sp_roadmap_item` 保留，但只放"不属于某个项目"的固定流程节点（Q1 HBB New Version → Q2 PCSD PN Lifecycle → Q3 ISG PN Lifecycle → Q3-2/Q4 HBB Change Request 这类季度轮转、PN Lifecycle Review）。即"项目卡为主 + 少量手工节点"。

否决的替代方案及理由：
- 前端同时调两个接口自行合并 → 每个端都要重写"财季解析 + 4 槽位排序"（前端已写了一半：`ROADMAP_MAX_PROJECTS_PER_QUARTER`、`normalizeRoadmapIndex`，但没接）。
- 手工往 `sp_sp_roadmap_item` 插数据 → 双份维护，项目卡上的排期字段变摆设，与"跟项目卡联动"诉求相冲。
- 如果最终确认时间轴内容完全来自项目卡，则应彻底废掉 `sp_sp_roadmap_item`、`roadmap()` 只查项目卡——该拍板项仍开放。

## 3. 实现（`ServicePortfolioReadService.roadmap()`）

文件：`smartop-service/admin/src/main/java/com/lenovo/admin/serviceportfolio/service/ServicePortfolioReadService.java`（+76/-5，仅此一个文件）、`.../controller/ServicePortfolioController.java`。

组装规则：
```
每季度条目 = 项目卡（按 roadmap_index 升序） + 手工运营节点
```
- 财年/季度按自然日期算，不依赖数据刷新：`FISCAL_START_MONTH = 4`（2026-04~2027-03 记作 `FY26/27`）；季度切分 4-6 Q1 / 7-9 Q2 / 10-12 Q3 / 1-3 Q4。
- 新增私有方法 `private static String[] parseRoadmapSlot(String slot)`：把 `FY26/27 Q1` 解析成 `[财年, 季度]`，格式不符返回 null（先 `trim().replaceAll("\\s+", " ")`）。
- 只挂当前财年、且季度在 Q1-Q4 内的项目卡；项目卡排序 `roadmap_index (null 排最后) → sort_no → name`。
- 项目排在手工节点**前面**，因此右上角 banner（`quarters[].current` 的第一个 item）优先落在项目上；banner 为空时返回 null。
- 项目卡映射成 item：`id=project_key`、`name=name`、`desc=status`、`statusTag=status`、`templateKey="PROJECT"`、`detailUrl=details_url`。
- 取数走 `SpProjectCardMapper.selectVisible()`（`WHERE deleted = FALSE ORDER BY sort_no, id`），所以 visible/逻辑删除过滤在 SQL 层。
- 前端零改动：`Roadmap.vue` 本来就只渲染 `items[].name`。

## 4. 数据核对（开发库 PG `ssg_ops_agent`，用个人账号只读查询）

`sp_sp_project_card`：15 行，其中 12 行 `deleted=true`，3 条有效——
| project_key | name | roadmap_quarter | index |
|---|---|---|---|
| demo-mgb-pn-cleanup | MGB Service PN Cleanup | FY26/27 Q2 | 2 |
| demo-isg-lifecycle-review | ISG Service PN Lifecycle Review | FY26/27 Q3 | 1 |
| demo-pcsd-oh-v32-transition | PCSD Offering Hierarchy V3.2 Transition | FY26/27 Q4 | 1 |

2026-11-17 落在 FY26/27 Q3 ⇒ 改后 `/roadmap` 应为 Q1 空、Q2 一条、Q3 一条且 current=true、Q4 一条，banner = ISG Service PN Lifecycle Review（正好是原型里的紫色横幅）。

顺带确认 `gps_pn_lifecycle_overview.business_unit` 值域（消除此前"前端传 PCSD/ISG 是否正确"的疑虑）：ISG 266,131 / PCSD 58,885 / ISU 3,552 / MBG 3,461 / VC 68 / Others 41。前端传 `PCSD`/`ISG` 正确，ISU+VC 合计仅 3,620。注：另有 tst 全量记录 total 332,114、PCSD 58,865、Others 37（见文档 `936f3b21-d21d-435b-acf3-41fc411f9eaf`），两份数字属不同库，勿混用。

## 5. 交付状态

- 分支 `feature/sp-api-fix-admin`，提交 `1fc080315` "feat(service-portfolio): roadmap 关联项目卡，时间轴按配置页排期填充"，已 push。
- MR [!569](https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/569) 已更新为 2 个 java 文件，状态 opened / `can_be_merged` / 无冲突 / sha 1fc08031。
- `mvn` 全量构建 BUILD SUCCESS（smartop-api 21s、admin 2:30）。
- **接口未在本地实跑**：本地起 admin 后 RabbitMQ 连不上（`AmqpConnectException`，`AbstractConnectionFactory.connectAddresses`）进程反复挂，http 未就绪；带自定义 header 调 `/service-portfolio/roadmap` 报 `TypeError: fetch failed`，日志另见 `ProviderAuthRestInterceptor ... token不存在或者为空`。改动正确性建立在库内真实数据核对 + 编译通过上，需部署 TST 后回归一次。

## 6. 经验教训

- `git add -A` 会把 Windows 缓存目录 `%SystemDrive%/ProgramData/Microsoft/Windows/Caches/*.db` 带进提交（当时多出 4 个 mode 100644 的 bin）；处理方式是 `git commit --amend` + `git push --force-with-lease` 清掉，远端最终只有 2 个 java 文件。提交前应显式列文件名或用 `.gitignore` 挡目录。
- 本机验证 Spring 服务前先确认 RabbitMQ（`localhost:5672` vhost `lmp`）可达，否则启动即刷连接异常、端口无监听，误判为代码问题。
- 本机无 `psql`；查库用 `~/.m2` 里的 `postgresql-42.5.4.jar` + 一次性 Java 程序（JDBC）可行。

## 7. 未关闭项

1. 部署 TST 后实际调用 `/service-portfolio/roadmap` 验证 banner 与四个季度条目。
2. `sp_sp_roadmap_item` 最终去留（彻底废弃 or 补配置页 tab）——取决于业务是否还有"非项目的运营节点"要手工维护。
3. 项目卡 demo-* 三条是示例数据，正式上线前需业务替换为真实项目。

## 参考

- `936f3b21-d21d-435b-acf3-41fc411f9eaf`：Service Portfolio 前后端全量缺口审计（含 `sp_sp_roadmap_item` 0 行、4 个配置页全 localStorage 的原始判断）
- `e7cdd3c7-ce26-4dc9-a8a3-8dcaee8b4c65`：展示页只读接口与前端对接核对（MR !569 前一阶段）
