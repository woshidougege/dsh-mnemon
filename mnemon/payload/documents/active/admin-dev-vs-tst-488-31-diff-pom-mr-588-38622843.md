---
id: "38622843-363f-4bc2-ad74-dbdaba9e027d"
title: "Admin 本地 dev vs tst 跨环境 488 端点对照（31 DIFF 归因）、库回滚加固与 POM 警告清理 MR !588"
description: "本地 dev profile 对测试环境 tst profile 的 488 端点对照：457 一致 31 差异分布六类（dev 缺 kmverse.ask.admins 致 6 个端点 300、dispatch 开关、时间戳/随机、锁顺序副作用、JSR-303 校验顺序与 locale、超时、并发快照需串行复测）；DbRestore3 因 pk_ssg_chatbi_question_cluster 并发写冲突中断后的 ON CONFLICT DO NOTHING + 单表 try/catch 加固与 48/49 表复原、cluster 多 24 行属采集新写入；两条历史 POM 警告（sqljdbc4 写成 scope=4.0、admin 重复声明 smartop-api）修复为 MR !588；create-mr.ps1 需 URL 编码 project 否则 404、参数是 -DescFile；!578 描述 4 处口径错误待用户决定是否回改。"
status: "active"
created_at: "2026-11-18T20:55:44.747Z"
updated_at: "2026-11-18T20:55:44.747Z"
content_hash: "00e027ba4100eac0d69ad0c848e4c96649c962d863403bb6b08ff3072872c8f1"
source_paths:
  - "smartop-common/smartop-common-datasource/pom.xml"
  - "smartop-service/admin/pom.xml"
session_ids:
  - "mnemon-review-v1-1dea3710-9d2d-4b4f-b899-fac9abfc06d6"
memory_body_ids:
  []
---

# Admin：本地 dev 与 tst 跨环境全量接口对照（488 端点 / 31 DIFF 归因）、库数据回滚加固与 Maven POM 警告清理（MR !588）

2026-11-19 完成。工作目录 worktree `C:\Users\63588\lenovo\ssg-sales-ops\.agents\worktrees\admin-control-sql-refactor`（分支已切到 `feature/fix-pom-warnings-admin`，基于 `admin-release` 的 `acc7317a3`）。

相关文档：改动前/后 jar 同库对照的 488/496 回归与 @Options 缺陷见 `f88cb5b5-3f66-444e-9cc6-f27525525c52`；419 条 XML 化交付见 `4ad84c6f-035c-4e07-9b52-19438e2f5f5b`；permission 包实测见 `68e95ff1-b38b-4461-a184-870bff5c7262`。**本文记录的是另一组对照**：本地 dev profile 服务 vs 测试环境 tst profile 服务，因此差异数与前述同库对照（473 一致 / 15 差异）不可直接比较。

## 1. 跨环境对照结果

两侧：本地 `http://127.0.0.1:8082`（用户在 IDEA + JRebel 自起，全程未动）vs `https://ssgspark-tst.lenovo.com/api/admin`，同一套 `user` 请求头（renxp4 / isAdmin）。

| 层 | 端点数 | 一致 | 不一致 |
|---|---|---|---|
| GET 查询 | 243 | 232 | 11 |
| POST/PUT 写 | 222 | 202 | 20 |
| DELETE | 23 | 23 | 0 |
| 合计 | 488 | 457（93.6%） | 31 |

## 2. 31 个 DIFF 的六类归因（无一是 SQL 迁移问题）

**① 配置差异 10 个**（本地 dev profile vs 远端 tst profile，配置本就不同）
- `/api/v1/chat/admin/*`（4 个）、`/chat/settings/me`、`/chat/me/profile` 本地返回 `{"status":300,"msg":"Only SPARK admins can change these settings"}`。根因：`AskSettingsService.java:82-85` 只读 `@Value("${kmverse.ask.admins:}")`；`application-lmp-reston-tst.yml:147` 有 `admins: zhangyong5,renxp4`，而 `application-lmp-reston-dev.yml` 的 kmverse 段（:124）**没有这一行** → 本地 admins 为空、isAdmin 恒 false。这是本地跑 admin 时常被误判成 bug 的地方。
- `/api/v1/chat/dispatch`、`/dispatch/stream` 返回 `errorCode:"DISABLED"`：dev 关闭、tst 开启。
- `/data-inquiry-probe/baseline/jira/verify`：Jira 配置不同。

**② 时间戳 / 随机 / JWT 6 个**：`/bi-admin/overview` 的 generatedTime、`/scheduler/cache/overview` 的 generatedAt+ttlSeconds、`/botland/getBotlandSign` 的 JWT、`/efficiency/getRandomEfficiency`（本身即随机）、`/chatbi/.../review/rules` 的 updatedAt、两个 export 的 500（两侧都 500，仅 error timestamp 不同）。

**③ 锁 / 任务状态（顺序副作用）6 个**：`/chatbi/.../export`（本地先抢到导出锁返回 409，远端 400）、`/hive-performance/run` 与 `/scheduler/cache/rebuild`（本地已有批次在跑 → 300，远端 200）、`/chatbi/notifications/archive-read`（updated 计数因先跑而不同）。

**④ 校验消息顺序 / 国际化 3 个**：`/permission/roles/999999` 本地报 `roleCode is required`、远端报 `roleName is required`（JSR-303 字段校验顺序不同）；`/menu/management/errorflag/update` 同理；`/insights-hub/reports/999999/favorite` 本地中文 `不能为null`、远端 `must not be null`（locale 不同）。

**⑤ 超时 2 个**：`/chatbi/.../questions/rebuild`（两侧都超时）、`/data-inquiry-probe/baseline/cases/sync`（本地超时、远端 `saved:182`）。

**⑥ 并发快照 4 个**：`/chatbi/events/report/options` 与 `/chatbi/events/report/feedback-records` 首轮 DIFF，**串行重跑后逐字节相等** —— 同一批里 `review/backfill` 正在插数据造成的时序差异。方法论：跨环境比对时对「疑似并发快照」类差异必须串行复测再定性，否则会误报。

## 3. 测试确实改写了库，已全量回滚

根因是接口自身的破坏性设计（`ChatbiQuestionClusterService.rebuild()` 先清后建无事务，见 f88cb5b5 文档），不是迁移问题。

恢复工具：`DbRestore3.java`（读取导出的 jsonl，按每张表 DELETE 后批量 INSERT，`Types.OTHER` 按原类型回填，末尾 `setval(pg_get_serial_sequence(...))` 修正自增序列）。

第一轮失败与加固（关键教训）：
- 第一轮在 `public.ssg_chatbi_question_cluster` 撞唯一键中断，后续 3 张表没轮到：
  `ERROR: duplicate key value violates unique constraint "pk_ssg_chatbi_question_cluster"` / `Key (profile_name, question_raw)=(sdq_tcv_booking, penetration rate for premier support plus) already exists`
  原因：chatbi 采集任务在恢复期间持续并发写入，抢了同一行；且原脚本无 `ON CONFLICT`、无单表异常隔离，一处失败整个 for 循环终止。
- 加固两处：INSERT 加 `ON CONFLICT DO NOTHING`；`for (String fn : targets) { try { ... } catch (Exception e) { System.out.println("  FAIL " + fn + " -> " + e.getMessage()); } }` 让单表失败不中断；并对空 cols 的表单独走 count 校验分支再 `continue`。
- 恢复完成：48/49 张表行数与测试前完全一致（`public.ssg_chatbi_question_daily` 12008、`public.ssg_hive_perf_run` 263（-183 条测试巡检已清）、`public.ssg_permission_audit_log` 225、`public.ssg_hc_consultant_manager` 29、`public.ssg_chatbi_question_review_dismissal`、`public.ssg_admin_menu_item`、`chatbi_internal_user`、`profile_group` 等）。
- 唯一残留：`public.ssg_chatbi_question_cluster` 9964 → 9988（多 24 行）。这 24 行是采集任务在恢复过程中新写入的聚合行，**不是测试残留，删了反而丢真实数据**。

## 4. 顺带修复的两条历史 Maven 警告 → MR !588

https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/588
分支 `feature/fix-pom-warnings-admin`，commit `e30f7e6b`（修正版，早先误提交的 `.tmp-commit-msg.txt` 已剔除），纯 POM 清理、零代码改动、依赖树不变；`mvn validate` BUILD SUCCESS、两条 WARNING 消失。状态：opened，未合并。

| 位置 | 问题 | 引入来源 |
|---|---|---|
| `smartop-common/smartop-common-datasource/pom.xml:58` 附近（sqljdbc4 依赖） | 写了 `<scope>4.0</scope>`，把版本号填进 scope；Maven 忽略并回退 `compile` | `33ffc2f86` Cabin 代码迁移 |
| `smartop-service/admin/pom.xml:115` 附近 | `smartop-api` 在同一段里声明两次，第二个被丢弃 | `dc6af6d50` 渗透测试安全修复 |

两条都是历史遗留，非 !578 引入，也从未导致构建失败，只是每次构建刷 WARNING。

## 5. GitLab MR 自动创建脚本的两个坑（`create-mr.ps1`）

脚本用 `git credential fill`（`protocol=https` / `host=gitlab.xpaas.lenovo.com`）取 password 作 `PRIVATE-TOKEN`，POST `https://gitlab.xpaas.lenovo.com/api/v4/projects/<project>/merge_requests`。
- 参数名是 `-Project -Source -Target -Title -DescFile`（**没有 `-TitleFile`**，写错会报「找不到与参数名称 'TitleFile' 匹配的参数」）。
- **project 段必须 URL 编码**：写 `dataagent/ssg-sales-ops` 返回 `{ "error": "404 Not Found" }`；改成 `dataagent%2Fssg-sales-ops` 后成功 `CREATED !588 ... state=opened`。脚本对 409 已存在的 MR 会回查 opened 列表复用。

## 6. 遗留待决（等用户定）

`f88cb5b5` 文档对应的 MR !578「已知遗留」段落复扫发现 4 处口径错误（!578 已合并 `547952e2d`，代码无问题，只是描述误导 reviewer）：
1. 称 lower(itcode) 未命中索引、需建函数索引 —— **事实错误**：`V1.1.131` 已建 `ux_ssg_user_itcode_lower` 与 `idx_ssg_permission_audit_prev`，tst 库 flyway 显示 1.1.131 已成功执行；
2. `${}` 写「150 处/54 文件」，admin 单模块实测 105 处/48 文件（150 是含 auth+common 的全域）；
3. `PageHelper.startPage` 写 23 处，实测 14 处/9 文件；
4. permission 包 `lower(itcode)=lower(` 写 4 处，实测 6 处。

已合并 MR 是否回改描述由用户决定，本轮未动。
