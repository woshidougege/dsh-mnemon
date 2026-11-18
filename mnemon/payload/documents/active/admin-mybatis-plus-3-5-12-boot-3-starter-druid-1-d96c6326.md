---
id: "d96c6326-eddf-44cc-87db-28b5db6ab024"
title: "admin 持久层升级落地：MyBatis-Plus 3.5.12（Boot 3 starter）+ druid 1.2.27 与内网 Maven/离线构建排查"
description: "ssg-sales-ops admin 把 MyBatis-Plus 3.5.5 换成 mybatis-plus-spring-boot3-starter 3.5.12、druid starter 换成核心 jar 1.2.27 的落地过程：内网 nexus 拉包约束、_remote.repositories 导致 offline 构建 present-but-unavailable 的根治、MP 3.5.7 insert(Collection) 重载导致 insert(any()) 编译失败、探针与依赖树验证证据、MR !578 状态与 chatbi 待决策项。"
status: "active"
created_at: "2026-11-18T11:52:02.654Z"
updated_at: "2026-11-18T11:52:02.654Z"
content_hash: "0476f879437873ebc7c60d8afa34fe840cc0519327c1f53f33f91e255715adf2"
source_paths:
  - "README.md"
  - "pom.xml"
  - "smartop-common/smartop-common-core/pom.xml"
  - "smartop-common/smartop-common-datasource/pom.xml"
  - "smartop-service/admin/pom.xml"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/dataintegration/service/DataRefreshBatchTrackerTest.java"
session_ids:
  - "mnemon-review-v1-ee5ae6c7-92fc-4fbb-a6ca-335279ad22e7"
memory_body_ids:
  []
---

# admin 持久层升级落地：MyBatis-Plus 3.5.12（Boot 3 starter）+ druid 1.2.27，以及内网 Maven / 离线构建排查

日期：2026-11-18。仓库 `dataagent/ssg-sales-ops`，分支 `feature/admin-control-permission-sql-admin`，worktree `.agents/worktrees/admin-control-sql-refactor`。
触发：用户要求「你去拉包吧，我内网有 maven，一定要从内网拉，不然自动打包他找不到的」——即在把 MP 升级落地前，先确认新坐标能从内网 nexus 取到，且 CI 自动打包可用。
推进关系：本文是文档 `8bbfce97-6332-4232-a383-d7d135260067`（admin 持久层依赖体检）的落地续篇，并**推翻其中一条限制**——该文当时判定「`mybatis-plus-spring-boot3-starter` 本地仓库无缓存、离线环境拿不到，只能记录不能马上换」；实测内网 nexus 有该构件，可以换。`47c46f5f-fd19-41b1-a6e6-4598dbfe407d`（tk.mybatis 实测更正 + mybatis-spring 3.0.4 锁定）中的锁定结论仍成立。

## 1. 内网 Maven 约束（硬要求，后续拉新依赖都适用）

- 内网仓库地址即 pom 里已声明的 `https://maven.xpaas.lenovo.com/nexus/repository/public/`（Nexus 聚合了 maven 中央代理与内部仓库）。
- 正确做法：单独写一份只镜像内网的 settings（本次为 `.tmp/settings-lenovo.xml`，`<mirrorOf>*</mirrorOf>` 指向内网 public），用 `-s` 指定后拉包，确保所有请求走内网、不碰公网；拉完核对构件版本确实存在于该 nexus（HTTP 200）。
- 为什么要这么严：公司的自动打包（CI）只能访问内网 nexus，公网拉到的东西同步不到 CI，构建会找不到依赖。**判断「能不能升」的标准是内网有没有，不是公网有没有。**

## 2. 实际改动（提交 `dedfef9bf`，chore(build): 升级 mybatis-plus 3.5.12（Boot 3 starter）与 druid 1.2.27）

| 项 | 从 | 到 |
| --- | --- | --- |
| MyBatis-Plus | `mybatis-plus-boot-starter` 3.5.5（Boot 2 坐标） | `mybatis-plus-spring-boot3-starter` 3.5.12 |
| druid | `druid-spring-boot-starter` 1.1.23 | 只引核心 jar `com.alibaba:druid:1.2.27` |
| `smartop-service/admin/pom.xml` | 重复声明 MP starter | 删除（core 已引入） |

理由（保留，评审时要用）：
- 3.5.5 的 `mybatis-plus-boot-starter` 面向 Boot 2，携带 `mybatis-spring 2.1.x`；`spring-boot3-starter` 原生声明 `mybatis-spring 3.0.4`，与根 pom 的 `dependencyManagement` 锁定一致。
- druid starter 的自动配置注册在 `spring.factories`，Boot 3 不加载；实际连接池一直由 dynamic-datasource 按 `spring.datasource.dynamic.druid` 创建，留着只多带 Boot 2 的 `jaxb-runtime`。换成纯核心 jar 后探针确认 `DATASOURCE_CLASS=com.baomidou.dynamic.datasource.DynamicRoutingDataSource`，行为不变。
- MP 3.5.9 起把 jsqlparser 相关能力（含 `PaginationInnerInterceptor`）拆到独立模块。本项目**不注册 MP 分页插件、分页只用 PageHelper**，故不引；项目里的 `jsqlparser 4.6` 与 `smartop-component-jsqlparser5` 与 MP 无关。
- 内网可取版本：druid 内网共 125 个版本（最新 1.2.28）；MP 3.5.7 ~ 3.5.18.1 均在内网。

## 3. 三个坑（都会重复遇到）

1. **`_remote.repositories` 导致离线构建 `present, but unavailable`（本次最大的坑）**
   用 mirror settings 拉包后，新构件该文件里记录的仓库 id 是 `lenovo-nexus`，与 pom 声明的仓库列表对不上，`mvn -o` 报 `org.springframework.boot:spring-boot-dependencies:pom:3.4.1 (present, but unavailable): Cannot access maven-ali ... in offline mode`，换成 jackson-bom 又是同一个错，逐个删文件治不完。
   根治：删除 `%USERPROFILE%\.m2\repository` 下**全部** `_remote.repositories`（本次 3214 个，扫描 10.5s、删除 14.0s）。该文件只记录「构件来自哪个仓库 id」，删除后退化为 Maven 3.8 及以前的行为（视为本地已安装），在线构建会重新生成，对其他项目无副作用。
   注意：此操作改的是用户本机全局 `.m2`，需明确告知用户；想恢复只需再在线构建一次。

2. **MP 3.5.7 起 `BaseMapper` 新增 `insert(Collection<T>)`，测试里 `insert(any())` 编译不过**
   `smartop-service/admin/src/test/java/com/lenovo/admin/dataintegration/service/DataRefreshBatchTrackerTest.java` 两处（119/120 行）及 312 行的 `verify(...).insert(any())` 因重载而推断不出类型，必须写显式类型：`any(DataRefreshBatch.class)` / `any(DataRefreshBatchItem.class)`。全仓同类隐患扫描出 7 处 mock 调用歧义候选（`selectList(any())`、`selectCount(any())`、`selectById(any())` 等），本次只改了编译失败的两处。

3. **druid 1.2.27 的 pom 带 `systemPath` 依赖告警**
   `com.alibaba:jconsole` / `com.alibaba:tools` 的 `systemPath` 指向 `${env.JAVA_HOME}/lib/jconsole.jar`、`tools.jar`，JDK 9+ 已无这些文件，构建时打两条 WARNING，不影响结果，不必处理。

## 4. 验证方法与实测（可复用这套证据链）

| 验证 | 命令 / 手段 | 结果 |
| --- | --- | --- |
| 编译 | `mvn -o -T 4 -pl smartop-service/admin -am test-compile` | 15 个模块 BUILD SUCCESS，58.8s |
| 依赖树 | admin 段 dependency:tree | `mybatis-plus-*:3.5.12`、`mybatis-spring:3.0.4`、`mybatis:3.5.17`、`druid:1.2.27`，**无 tk.mybatis** |
| 启动探针（临时 `UpgradeProbeTest`，用后移出到 `.tmp/UpgradeProbeTest.java.bak`） | 打印 MP 配置类 / 数据源类 / statement 数 | `CONFIG_CLASS=com.baomidou.mybatisplus.core.MybatisConfiguration`、`DATASOURCE_CLASS=DynamicRoutingDataSource`、`TOTAL_STATEMENTS=2085` |
| XML 迁移核对 | 探针断言 | `MAPPER ChatbiEventMapper/BookingsAggMapper/SpKpiCardMapper=true`；`XML_PERMISSION=true`、`XML_AUDIT=true` |
| 冒烟 | `MapperXmlSmokeTest` | 3/3 通过 |

`DATASOURCE_CLASS=DynamicRoutingDataSource` 是砍掉 druid starter、换纯核心 jar 之后的关键回归证据。探针运行时容器会尝试连 RabbitMQ 报 `AmqpConnectException: Connection refused`，属环境噪音，不是失败。

## 5. README 持久层约定（本次同步更新，评审基准）

`README.md` 的「持久层约定（MyBatis-Plus）」小节为团队硬约定：MP 为唯一通用 Mapper / Wrapper；PageHelper 2.0.0 为唯一分页方案，必须 `startPage` + `try/finally clearPage`；**禁止注册 `PaginationInnerInterceptor`**（与 PageHelper 双重拦截）；**禁止 `selectPage` / `IPage`**（无 MP 分页插件时会静默返回全量）；`tk.mybatis` 已移除不得重新引入；单表 CRUD 用 BaseMapper+Wrapper，多表/聚合/窗口函数与动态条件 ≥3 个走 Mapper XML。本次把技术栈表改成 3.5.12 + Boot 3 starter 并补了 jsqlparser 模块说明。

## 6. 交付状态

- 分支 `feature/admin-control-permission-sql-admin` 已推送（`550f53c79..dedfef9bf`），连续 9 个 commit：XML 化、移除 tk.mybatis、删僵尸模块 datacenter/dict 与 CIOP 导出模板、README 约定、MP/druid 升级。
- MR `!578` 已更新：title「refactor(admin): Access control SQL XML 化、移除 tk.mybatis 与僵尸模块、升级 MP/druid」，state `opened`、`merge_status=can_be_merged`、`conflicts=False`，描述约 10434 字符（含第五节依赖升级）。

## 7. 下一步（待用户拍板，不要自行开始）

`chatbi` 模块迁移是下一个大头：188 个 `@Select` + 58 个写操作；`ChatbiEventQueryMapper` 单文件 2104 行、47 个 select。两个待决策点：① 它同时是 User Logs 与 People 两个页签的数据源，改动面远大于 permission；② 是拆分成多个 Mapper 还是整体搬 XML，节奏要用户定。
