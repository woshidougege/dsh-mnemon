---
id: "8bbfce97-6332-4232-a383-d7d135260067"
title: "admin 持久层依赖体检（Spring Boot 3）：MyBatis / MyBatis-Plus / tk.mybatis / druid 谁真的在生效"
description: "ssg-sales-ops admin 与 smartop-common 的持久层依赖拓扑实测：按 jar 内 spring.factories vs AutoConfiguration.imports 判定 MP 3.5.5 生效、tk.mybatis 1.2.4 与 druid 1.1.23 完全不加载；mybatis-spring 被降到 2.1.2 的 pom 顺序根因、10 个 tk DAO 与 4 个必抛 BindingException 的调用点、@DS(magellanPRC) 未定义导致回退 admin 库、PageHelper 单轨分页、5 步整改顺序与影响面。"
status: "active"
created_at: "2026-11-18T09:23:57.932Z"
updated_at: "2026-11-18T09:23:57.932Z"
content_hash: "2926c6b1d2e0691bde3813e7f63908cd25bbfd1d078f63154eb57e39e7cc82fe"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/AdminApplication.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/menu/dao/MenusDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/dao/CiopDataUploadLogDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/service/impl/DataUploadServiceImpl.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/service/impl/DataApproveServiceImpl.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/service/impl/ManagementServiceImpl.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/service/impl/SyncTableServiceImpl.java"
  - "smartop-service/admin/src/main/resources/application.yml"
session_ids:
  - "mnemon-run-v1-de84d497-ddc6-4e94-aec7-b59a41d9b1b3"
memory_body_ids:
  []
---

# admin 持久层依赖体检（Spring Boot 3）：MyBatis / MyBatis-Plus / tk.mybatis / druid 谁真的在生效

日期：2026-11-18。范围：`ssg-sales-ops` 的 `smartop-service/admin`（包根 `com.lenovo.admin`，启动类 `AdminApplication`）及其共享模块 `smartop-common-core` / `smartop-common-datasource`。
触发：在动手把整个 admin control 的注解 SQL 迁到 XML 之前，用户要求先确认「mybatis 和 mybatis plus 用的合理吗，是正确的搭配吗」。
结论一句话：**不合理——三套持久层框架叠在同一个 classpath 上，实际只有 MyBatis-Plus 在真正工作；tk.mybatis 是死依赖但代码仍在调它的通用方法，mybatis-spring 被降到 Spring 5 时代的 2.1.2；分页只有 PageHelper 单轨。地基不摆正就开始批量搬 SQL 会踩坑。**

## 1. 判据：怎么判断哪个 starter 在 Boot 3 下真的生效

不能只看 pom。要看 jar 内注册的是 Boot 2 的 `META-INF/spring.factories` 还是 Boot 3 的 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`，逐个拆包结果：

| 依赖 | 注册方式 | Boot 3 是否加载 | 实际状态 |
|---|---|---|---|
| mybatis-plus-spring-boot-autoconfigure 3.5.5 | 两个都有 | 是 | **生效**，启动日志有 MP 3.5.5 banner |
| mybatis-spring-boot-autoconfigure 3.0.4 | 两个都有 | 是 | 加载了，但 `SqlSessionFactory` 被 MP 先创建 → 空转，只贡献 jar |
| mapper-spring-boot-autoconfigure 1.2.4（tk.mybatis） | **只有 spring.factories** | 否 | 完全不加载 |
| druid-spring-boot-starter 1.1.23 | **只有 spring.factories** | 否 | 完全不加载（连接池参数另有兜底，故不报错） |
| pagehelper-spring-boot-autoconfigure 2.0.0 | 两个都有 | 是 | **生效**，23 处 `PageHelper.startPage` |
| dynamic-datasource-spring-boot3-starter 4.3.0 | imports | 是 | **生效**，日志 `initial loaded [1] datasource` |

## 2. mybatis-spring 被降到 2.1.2（P0）

依赖树实际形态：

```
+- org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.4
|  \- org.mybatis:mybatis:3.5.17
\- com.baomidou:mybatis-plus-boot-starter:3.5.5
   \- org.mybatis:mybatis-spring:2.1.2      <-- 真正生效的是这个
```

Spring Boot 3 = Spring 6，官方配套是 `mybatis-spring 3.0.x`；2.1.2 是 Spring 5 时代产物。现在能跑是因为 2.1.x 与 3.0.x API 差异不大，属于**侥幸兼容**，不是正确搭配。
根因是 pom 声明顺序：`smartop-common-core`（带 MP starter）声明在 `smartop-common-datasource`（带 mybatis-spring-boot-starter）之前，Maven 就近优先把 3.0.4 裁掉。
另：`mybatis-plus-boot-starter` 是 Boot 2 坐标，Boot 3 对应 `mybatis-plus-spring-boot3-starter`；MP 3.5.5 的 autoconfigure jar 恰好同时塞了 imports 文件所以生效，属偶然。该 Boot3 artifact **本地仓库无缓存（离线环境拿不到）**，只能记录不能马上换。

## 3. tk.mybatis：一行都没在工作，但代码还在调它（P0）

10 个 DAO 真实 `extends tk.mybatis.mapper.common.BaseMapper`：

`CiopDataUploadLogDao`、`DataApproveLogDao`、`DataCenterDao`、`DataTableApplyColumnsDao`、`DataTableApplyDao`、`DataUploadDao`、`DataUploadTempDao`、`DictDao`、`DictDetailDao`、`EmailLogDao`
（另有 `smartop-service/admin/src/main/java/com/lenovo/admin/menu/dao/MenusDao.java:14` 仅 import 了 tk 的 BaseMapper 而并不继承。）

但 `smartop-service/admin/src/main/java/com/lenovo/admin/AdminApplication.java:10` 用的是原生 `org.mybatis.spring.annotation.MapperScan({"com.lenovo.admin.*.dao"})`，不是 tk 的扫描器；全仓库搜 `MapperHelper` / `MapperScannerConfigurer` 均为 0 命中。**tk 的通用方法从来没被注册成 MappedStatement。**

代码里仍有 4 个真实调用点在调 tk 方法：

| 位置 | 调用 | 业务 |
|---|---|---|
| `smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/service/impl/DataUploadServiceImpl.java:524` | `dataUploadTempDao.insertSelective(...)` | Excel 上传写临时表 |
| `smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/service/impl/DataApproveServiceImpl.java:218` | `dataApproveLogDao.insertSelective(...)` | 数据审批写日志 |
| `smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/service/impl/ManagementServiceImpl.java:785` | `dataUploadTempDao.updateByPrimaryKeySelective(...)` | 表申请回填关联 id |
| `smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/service/impl/ManagementServiceImpl.java:900` | `tableApplyDao.updateByPrimaryKeySelective(...)` | 审批结果更新 |

按装配链路推，这 4 行执行时必然抛 `BindingException: Invalid bound statement (not found)`——**这是静态推断，未做运行时实测**（要起应用调一次上传/审批接口才能确认）。要么这条路很久没人走，要么它一直在报错但没人追。
注意 `smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/service/impl/SyncTableServiceImpl.java:139` 里 `dataUploadLogDao.insertSelective` 已被注释掉，说明历史上已经有人在绕这个坑。

## 4. @DS("magellanPRC") 与 ciop_* 表缺失的因果假设

`@DS("magellanPRC")` 共 6 处：`SyncTableDao.java:40/45/58/62/67` 与 `CiopDataUploadLogDao.java:14`。但 6 个 profile 的 application-lmp-*.yml 里**都没有定义 magellanPRC 数据源**，而 datasource.dynamic 配了 `strict: false`，于是这些注解静默回退到 primary（`admin` 库），注解形同虚设。
配套证据：直连 primary 库枚举表，`ciop_*` 为 0 张，`ciop_data_upload_temp` / `ciop_data_approve_log` / `ciop_data_table_apply` / `ciop_data_upload` / `ciop_data_upload_log` 全部 NOT FOUND。→ 合理解释是这些表本应在 magellan 库，数据源没定义所以查到了 admin 库（**待验证假设**，也可能是该环境本来就没建这些表）。
另：`magellan` 在仓库里只以表名前缀形式出现在迁移脚本（`V1.1.111__Create_daas_tcv_customer_quarter.sql:56`、`V1.1.22__Register_ic_sync_tasks.sql:258`、`V1.1.25__Update_ic_source_queries.sql:371`，均为 `prd_upsell.magellan_final_bkgs_details_combine*`），不存在名为 magellan 的数据源定义。

## 5. 分页双轨 + 配置陷阱（P2）

- `PageHelper.startPage` 23 处 / 12 文件——唯一在用的分页；MP 的 `PaginationInnerInterceptor` 从未注册，`IPage` / `selectPage` 0 处使用。
- 危险点：**谁按 MP 习惯写 `selectPage`，会静默查全量**（无拦截器就不拼 LIMIT），只会在数据量变大后才炸。
- 使用面基线：`@TableName` 53 处 / 53 文件、`@TableId` 43 处 / 43 文件、`Wrappers.` 83 处 / 6 文件、`selectPage|selectList(` 44 处 / 24 文件、`javax.persistence` 14 处 / 9 文件（多为 POJO 上的 `@Transient` 排除字段）、`jakarta.persistence` 1 处。
- 配置陷阱（`smartop-service/admin/src/main/resources/application.yml` 88-91 行附近）：`mybatis-plus.mapper-locations: classpath*:/mapper/**/*.xml` 有效；但 `mybatis.*` 前缀对 MP **不生效**（MP 只读 `mybatis-plus.*`），历史上写的 `mybatis.configuration.map-underscore-to-camel-case` 一直是死配置。

## 6. 正确搭配目标态

```
Spring Boot 3.0.x
+- mybatis-plus-spring-boot3-starter 3.5.5   （换 Boot3 坐标；本地无缓存，暂不可换）
|  \- mybatis-spring 3.0.x                  （显式 dependencyManagement 锁定）
+- dynamic-datasource-spring-boot3-starter 4.3.0
+- druid 1.2.20+                             （1.1.23 是 2019 年的，有已知 CVE）
+- PageHelper 2.0.0 或 MP 分页插件（二选一，禁止双轨）
# tk.mybatis 整个删掉——它一行代码都没在工作
```

## 7. 整改顺序（按风险低→高，作为搬 SQL 的前置门）

| 步骤 | 动作 | 风险 | 影响面 |
|---|---|---|---|
| 1 | `dependencyManagement` 把 `mybatis-spring` 锁到 3.0.4（本地已有） | 低 | admin + auth 共用 common |
| 2 | 4 个 tk 方法调用点改 MP 写法（`insert` / `updateById`），再删 tk 依赖与 10 个 DAO 的 tk import | 中，须验证那 4 条业务路径 | admin |
| 3 | 删掉空转的 `mybatis-spring-boot-starter 3.0.4` | 低 | common-datasource |
| 4 | druid 升 1.2.x | 中，连接池参数要复核 | 全部服务 |
| 5 | 分页二选一：注册 MP 分页插件，或明确「只用 PageHelper」并写进规范 | 低 | admin |

第 1、3 步改的是 `smartop-common`，**auth 服务同样受影响，必须一起验证**。

## 8. 对「改完整个 admin control」的前置结论

- 10 个 tk DAO 里的**注解 SQL 本身是好的**，可以正常搬进 XML；但 3 个空的 tk 接口和那 4 个 tk 通用方法调用点是坏的，**搬 XML 救不了**，必须先改调用点。
- 分页双轨不定，后续凡涉及分页的语句不知道按哪套写，口径必须先定。
- 迁移规模基线以 `3ebbfe47-3740-42a5-9292-e7808dfa2d60`（89 dao / 686 条注解 SQL / 6306 行）为准；本文只负责持久层地基判断，具体迁移顺序见该文档。
- 辅助验证手法：直连 PostgreSQL 用 `.tmp/ResultDiff.java`（`SELECT id FROM ssg_user` 遍历 + 新旧 SQL 并排取 role_ids/group_ids/group_names/role_names 做逐行对比）与 `.tmp/Bench.java` / `.tmp/Bench2.java` 做耗时对比，凭 PG_URL/PG_USERNAME/PG_PASSWORD 环境变量连库，可作 SQL 迁移的等价性金标准。

## 9. 未验证项（不要当成结论用）

1. 那 4 个 tk 调用点是否真抛 `BindingException`——静态推断，需起应用实测。
2. `ciop_*` 表缺失是「数据源指错库」还是「该环境本来没建表」——未定。
3. rx 层结论：本仓库 spanner/jdbc 外的其他服务（auth）是否也吃同一个 mybatis-spring 2.1.2——只确认了模块依赖关系，未逐个服务实测。
