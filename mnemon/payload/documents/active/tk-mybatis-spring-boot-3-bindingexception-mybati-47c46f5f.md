---
id: "47c46f5f-fd19-41b1-a6e6-4598dbfe407d"
title: "tk.mybatis 在 Spring Boot 3 下实测可用（推翻「必抛 BindingException」）+ mybatis-spring 3.0.4 锁定 + Flyway 版本号重号陷阱"
description: "admin-control-sql-refactor 第 1 步的实测更正：tk mapper 3.5.3 接口自带 @SelectProvider，被 MP MybatisMapperAnnotationBuilder 的 type.getMethods() 一并解析，故通用方法可用（探针 16 断言全 true、statement 2306 不变），删 tk 的理由降级为清技术债；mybatis-spring 2.1.2 的真正根因是 smartop-service/admin/pom.xml:149 重复声明 mybatis-plus-boot-starter 把链路深度压到 2；Flyway 版本号不可字符串排序导致 V1.1.100 撞号→改 V1.1.130；admin-release 上 V1.1.129 重号待决策。更正文档 8bbfce97。"
status: "active"
created_at: "2026-11-18T09:57:08.307Z"
updated_at: "2026-11-18T09:57:08.307Z"
content_hash: "ef3952006b31fa024539d3a52cfba88e36a22ad4ae2eccfae729a65f47623c4e"
source_paths:
  - "pom.xml"
  - "smartop-service/admin/pom.xml"
  - "smartop-service/admin/src/main/resources/application.yml"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/dict/dao/DictDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/dict/dao/DictDetailDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/dao/DataUploadTempDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/dao/DataCenterDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/datacenter/dao/DataUploadDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/menu/dao/EmailLogDao.java"
session_ids:
  - "mnemon-run-v1-cec1ea18-bfa6-4fa5-8a32-b61b0bd33e07"
memory_body_ids:
  []
---

# tk.mybatis 在 Spring Boot 3 下实测可用：推翻旧结论的更正，外加 mybatis-spring 3.0.4 锁定与 Flyway 版本号陷阱

日期：2026-11-18。范围：`ssg-sales-ops` 分支 `admin-control-sql-refactor`（worktree `.agents/worktrees/admin-control-sql-refactor`），模块 `smartop-service/admin`。
触发：按既定 5 步整改顺序执行第 1 步（锁定 `mybatis-spring` 版本）时，先写探针验证旧假设，结果推翻了旧假设。

**本文更正文档 `8bbfce97-6332-4232-a383-d7d135260067`**（admin 持久层依赖体检）中的两条结论：①「tk.mybatis 一行都没在工作，4 个调用点必抛 `BindingException`」——错的；② mybatis-spring 被降到 2.1.2 的根因是「`smartop-common-core` 与 `smartop-common-datasource` 的 pom 声明顺序」——不够准确，真正触发点是 admin 模块的重复声明。该文档的依赖拓扑、jar 注册方式判据、druid 不加载等结论仍然有效。

## 1. 第 1 步已落地（提交 `b719fadea`，标题：锁定 mybatis-spring 3.0.4 并修正迁移脚本版本号）

只改两处：

1. **根 `pom.xml` 显式锁定 `mybatis-spring` 3.0.4**。此前全仓库无任何一处声明该版本，完全靠 Maven 最近优先仲裁。
2. **迁移脚本 `V1.1.100` 改名为 `V1.1.130`**。只改文件名，SQL 内容一字未动。原因见 §4。

## 2. 更正一：tk 的通用方法在 Boot 3 下**确实能工作**

旧结论的推理链是「`mapper-spring-boot-autoconfigure` 1.2.4 只注册 `spring.factories` → Boot 3 不加载 → `MapperHelper` 未初始化 → 调用 tk 通用方法必抛 `BindingException`」。实测不成立。

机制（`javap` 实测）：`tk.mybatis:mapper:3.5.3` 的接口方法**自带 `@SelectProvider`**——`SelectAllMapper.selectAll()` 标注了 `@SelectProvider(type=BaseSelectProvider)`。MP 的 `MybatisMapperAnnotationBuilder` 用 `type.getMethods()` 遍历（含从父接口继承的方法）时，把父接口方法上的 Provider 注解一并解析注册。所以它既不依赖 tk 的 `MapperAutoConfiguration`（Boot 3 下确实不加载），也不依赖 `MapperHelper`。

探针实测（`com.lenovo.admin.tmp.TkProbeTest`，临时目录，未纳入版本控制）：

```
PROBE CONFIG_CLASS=com.baomidou.mybatisplus.core.MybatisConfiguration
PROBE TOTAL_STATEMENTS=2306
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 83.10 s
```

16 项断言全部 `true`，覆盖 4 个 tk 通用方法调用点及旁证：

- `com.lenovo.admin.dict.dao.DictDao.selectAll` / `.selectOne`
- `com.lenovo.admin.dict.dao.DictDetailDao.select`
- `com.lenovo.admin.menu.dao.EmailLogDao.insert`
- `com.lenovo.admin.datacenter.dao.DataUploadTempDao.insertSelective` / `.updateByPrimaryKeySelective` / `.selectOne`
- `com.lenovo.admin.datacenter.dao.DataApproveLogDao.insertSelective`
- `com.lenovo.admin.datacenter.dao.DataTableApplyDao.updateByPrimaryKeySelective` / `.selectOne`
- `com.lenovo.admin.datacenter.dao.DataCenterDao.insert` / `.selectOne` / `.selectCount`
- `com.lenovo.admin.datacenter.dao.DataUploadDao.insert` / `.delete`
- `CONTROL custom-annotation = true`

结论修正：**删 tk 的理由从「修线上 bug」降级为「清理技术债」**——两套通用 Mapper 并存；tk 1.2.4 依赖 `javax.persistence` 的 `@Id`/`@Table`，而项目主用 MP 的 `@TableId`；靠别人的解析器被动工作、隐式耦合。排查 tk DAO 全集用的是 `grep "extends BaseMapper"`，命中 64 个文件，其中真正 import `tk.mybatis.mapper.common.BaseMapper` 的只有 13 个文件（含 `AuthApplication` 里的 `@MapperScan` 与社会化注释），其余是 MP 的 `BaseMapper`。

## 3. 更正二：mybatis-spring 2.1.2 的真正触发点是 admin 模块的重复声明

真正的根因在 `smartop-service/admin/pom.xml:149`：那里**重复声明了一次 `mybatis-plus-boot-starter`**（`smartop-common-core` 已经引入过）。重复声明把 MP → mybatis-spring 这条链的**深度压到 2**，而 `datasource` → `mybatis-spring-boot-starter` 的链路深度是 3；Maven 最近优先取浅者，于是 2.1.2 胜出。

锁定 3.0.4 后用 `mvn dependency:tree` 复核：

```
+- org.mybatis:mybatis-spring:jar:3.0.4:compile (version managed from 2.1.2)
+- org.mybatis:mybatis:jar:3.5.17:compile
```

锁定后 tk 行为**完全没变**（statement 总数仍是 2306），这直接印证 §2 的第 3 条：`mapper-3.5.3.jar` 里一个 `org/mybatis/spring` 引用都没有，tk 与 mybatis-spring 版本无耦合。

## 4. 坑：Flyway 迁移脚本版本号不能用字符串排序

判断「现有最大版本」时若用字符串排序，`V1.1.99` 会排在 `V1.1.129` **之后**（`'9' > '1'`），得出「最大是 V1.1.99」的错误结论，于是新脚本取 `V1.1.100`——与基线里已有的 `V1.1.100__Add_ssg_menu_entry_flag.sql` 撞号，flyway 在 **resolve 阶段**就抛：

```
Found more than one migration with version 1.1.100
```

规避：改名 `V1.1.130`。注意 `smartop-service/admin/src/main/resources/application.yml` 里的 `validation-mode: VALIDATE_NOT_PRESENT`、`ignore-missing-migrations: true`、`validate-on-migrate: false` **都绕不过** resolve 阶段的版本号唯一性校验——这类配置只管「已应用但缺失」与「校验和不符」，不管重号。

## 5. 未解决：admin-release 自身有一处重号（非本分支引入）

`origin/admin-release` 上并存：

```
V1.1.129__Create_chatbi_auto_review_rule.sql
V1.1.129__Seed_service_portfolio_kpi_cards.sql   ← 由 c6d8dfc91 引入
```

按 §4 的配置分析，admin-release **现在启动就会报错**（resolve 阶段）。本分支已避开，但按「不改别人模块」的约定未动它。提议是把 `Seed_service_portfolio_kpi_cards` 顺延成 V1.1.130，在 MR 里带说明给作者——**但要注意**：本分支的迁移脚本已经占用了 V1.1.130，两边同时落主干会再次撞号，需协调成不同号段。

**待决策（下一步走哪条）**：
- 继续 S1：删 tk 通用 Mapper（10 个 DAO 去掉 `extends`、约 15 个调用点改写成 MP 写法、实体补 MP 注解、pom 去掉 tk）。
- 先修 129 重号。

## 6. 可复用的等价性验证手法

- **statement 总数基线**：`PROBE TOTAL_STATEMENTS=2306`，改动前后必须一模一样（本次 2306 → 2306，无增无减）。这比「能启动」强得多，能捕捉「方法名解析到不同 provider」这类静默漂移。
- **配置类基线**：`PROBE CONFIG_CLASS=com.baomidou.mybatisplus.core.MybatisConfiguration`——确认跑的是 MP 的配置实现而非原生 `org.apache.ibatis.session.Configuration`。
- `mvn -o -pl smartop-service/admin test-compile` → BUILD SUCCESS（166 个测试类）。
- 旧的一批 pre-existing 失败（chatbi 两个测试）已由新拉的 4 个提交修掉，本次 `mvn test-compile` 通过，无需再动。

## 7. 环境噪音（勿误判为本次改动引起）

启动日志里持续出现 RabbitMQ 发送失败：`java.net.ConnectException: Connection refused` / `MessageHandlingException: error occurred in message handler [org.springframework.integration.amqp.outbound.AmqpOutboundEndpoint]`，来自 `scheduling-1` 线程的定时消息外发。本地无 broker，与持久层改动无关。
