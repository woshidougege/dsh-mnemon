---
id: "114173fc-0b85-4307-b84e-887ba2faaea3"
title: "admin 启动耗时排查：65 秒空窗定位与启动打点方案（ssg-sales-ops）"
description: "ssg-sales-ops admin 服务启动 66.3 秒零日志空窗的排查证据、分段耗时、StartupWatchdog/StartupEventLogger 打点实现（MR !522/!523）与部署后读日志判断树。"
status: "active"
created_at: "2026-11-12T10:12:20.402Z"
updated_at: "2026-11-12T10:12:20.402Z"
content_hash: "f6c67e17fc98e852dffbf6cb31b4967a2258f16fa68f2a07e381f05eee757756"
source_paths:
  - ".tmp/worktrees/startup-timing/smartop-service/admin/src/main/java/com/lenovo/admin/AdminApplication.java"
  - ".tmp/worktrees/startup-timing/smartop-service/admin/src/main/java/com/lenovo/admin/conf/StartupWatchdog.java"
  - ".tmp/worktrees/startup-timing/smartop-service/admin/src/main/java/com/lenovo/admin/conf/StartupEventLogger.java"
  - ".tmp/worktrees/startup-timing/smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/TrinoToPostgresSyncJobRecovery.java"
  - ".tmp/worktrees/startup-timing/smartop-service/admin/src/main/java/com/lenovo/admin/coc/schedule/CocFlagSyncTaskRegistrar.java"
session_ids:
  - "mnemon-run-v1-aa5bc996-82a1-4100-98cc-5f9f1f231548"
memory_body_ids:
  []
---

# admin 启动耗时排查：65 秒空窗的定位过程与启动打点方案（ssg-sales-ops）

仓库：`dataagent/ssg-sales-ops`（https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops），admin 服务（smartop-service/admin），Spring Boot 3.0 + Jetty + Druid + Flyway + Eureka + jasypt。
排查分支：`fix/startup-timing-admin`（基于 `origin/admin-release`），工作树 `.tmp/worktrees/startup-timing`。

## 排查约束（决定方案形态）

- **用户没有 kubectl 权限，只能看到平台上的 pod 日志**，因此不能用 jstack / 进容器抓栈；所有证据必须由打进日志的代码产出，靠重启一次再读日志定位。这直接否掉了"exec 进容器看线程"的路线，改成进程内看门狗自打栈。

## 问题与两次实测

现象：admin 启动末尾有一大段"全进程零日志"的空窗（最初怀疑 65 秒），整个启动约 158–193 秒。

| 指标 | 14:11 那次 | 16:45 那次 |
|---|---|---|
| Agent + JVM + 类加载 | 19.482s | 9.075s |
| `SpringApplication.run` 总耗时 | 158.093s | 192.791s |
| Runner 本体耗时 | 未知（原怀疑 65s） | **0.276s** |
| 数据源探测（取连接 + `SELECT 1`） | 未测 | **0.002s**（SQL 216ms，网络 RTT 占大头） |
| 中断任务恢复（select 0 条 → 逐条标记 0 条） | 未测 | 0.266s |
| 自动批次刷新（0 个 RUNNING） | 未测 | 0.005s |
| COC 注册（已有 id=55） | 未测 | 0.004s |

**被自己探针否掉的假设**：上一轮判断"可能卡在 Druid 拿连接满 60s（`testOnBorrow: false` + `maxWait: 60000` + 无 `socketTimeout`，连接被服务端/防火墙静默断开后仍当作空闲连接借出）"——实测取连接 2ms，假设不成立。

## 空窗的精确位置（16:45 那次）

```
16:48:12.498  Started ServerConnector{HTTP/1.1}{0.0.0.0:80}      ← 端口已开
16:48:12.502~12.590  jasypt RefreshScope 刷新（最后一条 refresh 日志）
16:48:12.773  Eureka registration status: 204                    ← 异步线程打的
        ┌──────────── 66.3 秒，全进程零日志 ────────────┐
16:49:19.106  [startup] Runner-1 同步任务恢复开始                 ← Runner 第一行
```

Spring Boot 3.0 `run()` 顺序：`refreshContext()` → 打印 Started → `listeners.started()` → `callRunners()`。端口起来后只剩三段可能：1) `publishEvent(WebServerInitializedEvent)` 的其它监听器；2) `ApplicationStartedEvent` 广播；3) `callRunners()` 里 `getBeansOfType(ApplicationRunner/CommandLineRunner)` 会把条件装配/懒加载的 Runner Bean 一并**实例化**。

对 `smartop-service` + `smartop-common` + `smartop-component` 全量 grep 的结论：admin 侧这三处**一个监听器都没有**（全仓只有 auth 模块 `smartop-service/auth/src/main/java/com/lenovo/auth/adfs/task/ADFSRunnerImpl.java` `implements CommandLineRunner`，不在本进程）。所以 66 秒来自依赖 jar（spring-cloud-bus/RabbitMQ 监听容器、xxl-job executor、SkyWalking agent 钩子），或 `finishRefresh` 尾部某个排在 Web 容器之后的 `SmartLifecycle`。

## 分段耗时（相对 run 开始 16:46:07.112，总 192.79s）

| 段落 | 耗时 |
|---|---|
| 配置中心首轮 + 主容器前置 | 52.9s（上次 42.4s，波动大） |
| Jetty 创建 → Started | 10.9s |
| Druid / Flyway / Redis / MyBatis | 15.4s |
| 黑盒 1：MyBatis 完 → KPI 载入 | 17.0s |
| KafkaInit + Hibernate Validator | 10.5s |
| 黑盒 2：HV → Eureka 初始化 | 18.4s |
| 端口监听 + Eureka 注册 | 1.2s |
| **黑盒 3：端口就绪 → 第一个 Runner** | **66.3s ← 元凶** |
| Runner 全部 + run 返回 | 0.3s |

黑盒合计 101.7s，占 53%。本次总耗时比上次更长：配置中心段 42.4s→52.9s，且叠加两个黑盒（17.0 + 18.4 = 35.4s）。

## 已落地的打点实现（纯日志，不动业务逻辑）

第一轮：提交 `61a12a6b6`「诊断(admin): 启动耗时打点，定位 Runner 阶段 65 秒空窗」，7 个文件 / 94 行 → MR **!522**，已于 2026-11-12 16:30:10 合并进 `admin-release`，merge commit `4cf1849798e3f167433767e4db60389f56d81e5e`。

第二轮（针对"没有 kubectl 只能看日志"）：提交 `766f3640c`「诊断(admin): 启动看门狗 + 启动阶段边界打点」，5 files / +190 −4 → MR **!523**（https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/523，`fix/startup-timing-admin` → `admin-release`，opened / can_be_merged / 无冲突 / sha `766f3640cb32`）。构建：mvn BUILD SUCCESS（总 3:58 min，admin 模块 03:14 min）；druid 1.1.23 的 pom invalid 与 `systemPath` jconsole/tools 报错是既有噪声，非本次引入。

打点清单：

1. `smartop-service/admin/src/main/java/com/lenovo/admin/AdminApplication.java`：`main()` 里 `long bootStart = System.currentTimeMillis()`，`run()` 前后各打一行，输出总耗时。
2. **新增** `smartop-service/admin/src/main/java/com/lenovo/admin/conf/StartupWatchdog.java`（关键）：`run()` 超过 45s 未返回时，每 10s 打印一次 `main` 线程完整栈（40 帧）+ 各状态线程概览，最多 8 次；daemon 线程，`run()` 返回即退出。为什么这样做：不需要容器权限，日志里直接给出阻塞帧。
3. **新增** `smartop-service/admin/src/main/java/com/lenovo/admin/conf/StartupEventLogger.java`：监听 `ContextRefreshedEvent` / `WebServerInitializedEvent` / `ApplicationStartedEvent` / `ApplicationReadyEvent`，各打一行带相对毫秒；在 `ApplicationStartedEvent` 时用 `allowEagerInit=false` 列出待执行 Runner 名字（不提前实例化，避免污染耗时归属），顺带验证部署 jar 里有没有未知的第三个 Runner。
4. `smartop-service/admin/src/main/java/com/lenovo/admin/dataintegration/TrinoToPostgresSyncJobRecovery.java`（注意实际包路径是 `dataintegration` 根，不在 `service` 子包）：新增"从 DataSource 取连接 + `SELECT 1`"探测，两步分别计时，异常打完整栈；加 `@Order(0)`。
5. `TrinoToPostgresSyncService.markInterruptedJobsAsFailed`：`selectList` 与逐条 update 分开计时。
6. `DataRefreshBatchTracker.refreshRunningAutoBatches`：批次数量、单批耗时，异常打完整栈。
7. `smartop-service/admin/src/main/java/com/lenovo/admin/coc/schedule/CocFlagSyncTaskRegistrar.java`：开始/结束/查询耗时；加 `@Order(1)`。该类本身是幂等补注册（按 `handler_key` 往 `ssg_sync_task` 补行，已有则不动，cron 只在新建时写入，之后管理员改库/调度中心调整不被覆盖——因为项目约定 flyway 迁移只改表结构）。
8. `StartupRunnerTailLogger`（`@Component` + `@Order(Ordered.LOWEST_PRECEDENCE)`，`implements ApplicationRunner`）：所有 Runner 之后、`ApplicationReadyEvent` 之前打一行，把"Runner 阶段"和"ApplicationReadyEvent 阶段"分开。
   - 踩过的坑：上一次实测它打印在 COC 之前——`LOWEST_PRECEDENCE` 平局时并不保证排最后，会误导结论。修正方式：给两个真 Runner 显式 `@Order(0/1)`，而不是继续调收尾类的 order。
9. `AiProbeService.cleanStaleRunsOnStartup`：清理耗时。

## 部署后如何读日志（判断树）

```bash
grep "\[startup\]" admin.log
```

期望顺序：`[startup] ===== SpringApplication.run 开始 =====` → `ContextRefreshedEvent` → `WebServerInitializedEvent` → `ApplicationStartedEvent` → `待执行的 ApplicationRunner = [...]` → `Runner-1 同步任务恢复开始` → … → `全部 ApplicationRunner 执行完毕（ApplicationReadyEvent 之前）` → `ApplicationReadyEvent` → `run 返回，共耗时 N ms`。

| 现象 | 结论 |
|---|---|
| `WebServerInitializedEvent` → `ApplicationStartedEvent` 差 66s | 卡在 refresh 收尾：仍有 phase 排在 Web 容器之后的 `Lifecycle`/`SmartLifecycle` bean（MQ 监听容器这类），或 `WebServerInitializedEvent` 的其它监听器 |
| `ApplicationStartedEvent` → `Runner-1` 差 66s | 卡在 `callRunners` 取 Bean 时的实例化：某个 Runner 的依赖在构造/初始化阶段阻塞 |
| 两段都很短但总耗时仍 190s+ | 空窗挪到了别的阶段，找相邻两行间隔最大的位置 |
| 看门狗输出 `main 线程状态 = RUNNABLE` + `socketRead0` 之类帧 | 直接按栈顶业务帧定位（如 RabbitMQ `SocketUtils`、HTTP 客户端、加解密） |

## 流程约定与其他未决项

- **约定：agent 只创建 MR，不执行合并**；合并与部署由用户负责（第一轮按此执行，用户随后合并了 !522）。
- MR 是通过 GitLab API v4 创建的：PowerShell 脚本先用 `git credential fill`（`host=gitlab.xpaas.lenovo.com`）取 token，再 `PRIVATE-TOKEN` 调 `/projects/dataagent%2Fssg-sales-ops/merge_requests`；`create-mr.ps1` 支持 `-TitleFile`/`-DescFile`（UTF-8 读文件，避免中文标题乱码），对 409 回退为查询同 source/target 的已存在 MR。
- 未决：主工作区 `feature/admin-control-export-admin` 上还留着第一轮 7 个打点文件的未提交改动（`AdminApplication.java`、`CocFlagSyncTaskRegistrar.java`、`TrinoToPostgresSyncJobRecovery.java` 已修改；`conf/StartupEventLogger.java`、`conf/StartupWatchdog.java` 未跟踪），等复测完再一并清理还是立刻恢复，用户尚未表态。
- 未决：用户尚未回贴部署后的新日志（!523 待合并/部署）。
