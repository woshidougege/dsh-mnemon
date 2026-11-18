---
id: "fd202c70-993b-4181-b119-1c8e983999a9"
title: "注解 SQL → Mapper XML 改造：MR !578 评审结论、!587 根因修正与迁移覆盖台账"
description: "修正文档 753fabcc 的 !586 结论（真根因是 selectCandidates 的 <script> 包裹住 RECOVERED_PLAIN_SQL 里的裸 <，!587 去掉 script 才解决）；评审 !578 的 permission 包 84 条 XML 化能否避免同类问题（能降级但非零风险，实测 950 行 0 CDATA/0 script），给出已迁/未迁台账（chatbi 188 条最大缺口）、三条补强建议（硬性禁止项、Mapper 全解析冒烟测试、优先迁 chatbi），并指出 !578 的 V1.1.130 与 !585 种子脚本撞号、需改 V1.1.131。"
status: "active"
created_at: "2026-11-18T11:39:08.135Z"
updated_at: "2026-11-18T11:39:08.135Z"
content_hash: "a2d847872805c98560678bc867d6c587aa17da450d41865c7ddd111f87a239f8"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiReviewJiraMapper.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.130__Optimize_admin_control_permission_query.sql"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.130__Seed_service_portfolio_kpi_cards.sql"
  - "smartop-service/admin/src/main/resources/mapper/permission/PermissionAuditDao.xml"
  - "smartop-service/admin/src/main/resources/mapper/permission/EffectivePermissionQueryDao.xml"
  - "smartop-service/admin/src/main/resources/mapper/permission/PermissionManagementDao.xml"
  - "smartop-service/admin/src/main/resources/mapper/permission/RoleManagementDao.xml"
  - "smartop-service/admin/src/main/resources/mapper/permission/UserGroupManagementDao.xml"
  - "smartop-service/admin/src/main/resources/mapper/permission/EffectivePermissionDao.xml"
session_ids:
  - "mnemon-review-v1-7605b2ec-586a-4932-8a60-e37da57766fd"
memory_body_ids:
  []
---

# 注解 SQL → Mapper XML 改造：MR !578 评审结论、!587 根因修正与覆盖台账

2026-11-18 晚。背景：用户正在系统性改造仓库里的 SQL（注解 SQL 迁 Mapper XML），!578 是 `permission` 包首例；!587 是对当天 Mapper 加载失败事故的**收口修复**。本文对已有文档 **753fabcc**（admin-release 启动两连撞事故复盘）做结论修正，并记录 !578 的评审判断与迁移台账。

## 1. 修正 753fabcc：!586 的修法不足，真根因是 `<script>` 包裹

753fabcc 记录 !586 的修法是「把 `CANDIDATE_SELECT` 里的 `event_time < e.event_time` 改成反向比较 `e.event_time > ssg_chatbi_event.event_time`」。**改完后错误一模一样**（列号仍是 3520），Class 换新也没用。

真正根因：同一段 SQL 还引用了自动评审的两个表达式常量
`ChatbiQuestionAutoReviewMapper.SESSION_RESULT_SQL` 与 `ChatbiQuestionAutoReviewMapper.FAILURE_CATEGORY_SQL`，后者内含 `RECOVERED_PLAIN_SQL`：

```sql
AND bad.started_at < ok.started_at
```

该常量是**专门给注解 SQL（非 script）准备的「原始运算符版」**，注释里写明。而 `selectCandidates` 把 `CANDIDATE_SELECT` 包进 `@Select("<script>" + ...)`，于是它跟着进了 XML 解析 → Mapper 仍加载失败。

定位手法（可复用）：把编译后的 class 按 UTF-8 解开搜 ` < `，全文件只有这一处命中。

修法（MR !587，1 文件，`fix/review-mapper-no-script` → `admin-release`）：
- `selectCandidates` **去掉 `<script>` 包裹**——条件全是静态拼接，没有 `<if>`/`<foreach>`，本就不该用 script
- `&gt;=` / `&lt;=` / `&gt;` 还原成原生运算符
- 加注释说明为什么不能再包 script（与 `findCandidateByTurn` 复用同一份 `CANDIDATE_SELECT`）

这样两个调用点都不经过 XML 解析，与 `RECOVERED_PLAIN_SQL` 的设计口径一致，也无需再造一份转义常量。

验证：`mvn -o -T 4 -pl smartop-service/admin -am -DskipTests compile` BUILD SUCCESS；IDEA 启 `AdminApplication` 通过（日志出现 `AdminApplication starting.....`，此前在这之前就抛异常退出）；8082 LISTEN（pid 17168），`/menu/all` 200；测试库 `EXPLAIN (ANALYZE, BUFFERS)` Planning 91.7 ms / Execution 136.4 ms，计划中可见 `Join Filter: (bad.started_at < ok.started_at)`，证明裸 `<` 在 PG 侧正常。

环境噪音（勿误判）：`/actuator/health` 返回 503、`/actuator/info` 404、控制台刷 `Connection refused`，原因是本地没起 RabbitMQ（配置指向 tst 的 MQ），与评审无关。

**教训**：同一 SQL 常量被 script / 非 script 两种上下文复用时，常量写法只有一个合法口径；修 bug 要沿常量引用链搜全图，不能只改报错那一行。

## 2. !578 能否避免同类问题：能，但不是零风险

`refactor(admin): Access control 权限 SQL XML 化、移除 tk.mybatis 与 datacenter/dict 僵尸模块`，`feature/admin-control-permission-sql-admin` → `admin-release`，118 文件，author=renxp4。

本次 bug 成立的三个条件在 XML 化后都不成立：

| 条件 | 注解 SQL 现状 | 迁 XML 后 |
|---|---|---|
| 手工拼 Java 字符串 | 有，无编译期检查 | SQL 在 XML，IDE 立刻校验 |
| 同一份 SQL 被两种上下文复用 | 有（`CANDIDATE_SELECT`） | 只有一个上下文，`<sql>` 多处引用也无害 |
| 常量混入只对一种上下文合法的写法（裸 `<`） | 有 | 全项目只有一套转义规则 |

实测该分支 6 个 XML 共 950 行：`&lt;` 5 处 / `&gt;` 4 处，**0 处 CDATA、0 处 `<script>`**。典型：`PermissionAuditDao.xml` 的 `WHERE p.id &lt; a.id` 与 `WHERE operator &lt;&gt; ''`；`EffectivePermissionQueryDao.xml` 的 `WHERE m.id &lt;&gt; #{menuId}`。不用 CDATA 是对的——避免嵌套限制与两种转义风格混用。

但要说清：XML 里写 `a.x < b.y` 一样会解析失败、服务起不来。区别是把「运行期偶发的字符串拼接陷阱」降级为「写时 IDE 红 + 启动时统一暴露」，而不是彻底消失。

## 3. 覆盖面台账（决定「以后不再发生」的边界）

已迁（`smartop-service/admin/src/main/resources/mapper/permission/`，namespace 与 DAO 全限定名一致，共 84 条）：

| XML | 语句数 | `<sql>` 片段 |
|---|---|---|
| PermissionManagementDao.xml | 24 | 0 |
| RoleManagementDao.xml | 26 | 1 |
| UserGroupManagementDao.xml | 18 | 2 |
| PermissionAuditDao.xml | 8 | 1 |
| EffectivePermissionQueryDao.xml | 6 | 0 |
| EffectivePermissionDao.xml | 2 | 0 |

未迁（!578 描述自带的分批表）：chatbi 188 `@Select` + 58 写操作；usagefeedback 26+12；datainquiryprobe/aiprobe 8+1；notification 11+11；jira 9+9；scheduler/dataintegration 12；access control 老接口 menu 56 / user 41 / role 19 / adminmenu 3。

**结论**：出事的 `ChatbiReviewJiraMapper` 在 chatbi 包，是 188 条里的一条，所以「以后不再发生」要等 chatbi 也迁完；这一次仍只能靠 !587 修。

## 4. 三条低成本补强建议（已向用户提出）

1. 把约定写成硬性禁止项，补进 !578 新增的 `README.md`「持久层约定」一节：DAO 禁止写 `@Select/@Insert/@Update/@Delete` 的 SQL（极简单表查询除外）；禁止 `<script>`，动态 SQL 一律进 XML；XML 比较运算符统一 `&lt;`/`&gt;`，禁止 CDATA。
2. **加「Mapper 全解析」冒烟测试**——唯一能进 CI 拦住这类错误的手段（现在生产/测试都是部署完才发现服务起不来：上周 1.1.129、今天 3520）。不连库、不启 Spring，只构建 MyBatis Configuration 并解析全部 mapper：

```java
Configuration cfg = new Configuration();
for (String xml : mapperLocations) new XMLMapperBuilder(...).parse();
for (Class<?> dao : daoClasses) new MybatisMapperAnnotationBuilder(cfg, dao).parse();
```

注解 SQL 拼写错误、XML 转义错误都能在 `mvn test` 阶段失败。
3. 下一批优先迁 chatbi，尤其 `ChatbiEventQueryMapper`（1600+ 行、几十个 SQL 常量互相拼接，同类问题风险最高）。

## 5. 阻塞项：!578 的迁移版本号又撞了，需改 V1.1.131

- `admin-release` 已有 `V1.1.130__Seed_service_portfolio_kpi_cards.sql`（!585 合入）
- !578 分支上是 `V1.1.130__Optimize_admin_control_permission_query.sql`

两者并存即重演 `Found more than one migration with version 1.1.130`（Flyway 在 resolve 阶段抛，`validate-on-migrate: false` 等配置绕不过，机制见文档 561cc78d）。!578 描述里「按数字序确认当前最大版本为 1.1.129，顺延为 V1.1.130」那段写于 !585 改名之前，已过期。

建议：改名为 **`V1.1.131__Optimize_admin_control_permission_query.sql`**（SQL 内容不动），同步更新 MR 描述的版本号说明，再用 `git ls-tree` 复核有无第三处撞号。

## 6. !578 其余值得留存的实现要点

- `PermissionAuditDao` 的 `operator`/`action`/`targetType` 三个 `<if>` 原先没写 `AND`、只靠 `<where>` 去头，**同时传任意两个条件就拼出 `WHERE lower(a.operator) = ? upper(a.action) = ?`**，Audit log 多选筛选必然失败；现每条件显式带 `AND`，列表与 count 的 where 抽成 `<sql id="auditFilter">` 共用。
- `previous_detail` 子查询由 `coalesce(p.target_id,'') = coalesce(a.target_id,'')` 改为 `IS NOT DISTINCT FROM`，语义等价但列不再被函数包裹，`idx_ssg_permission_audit_target` 后两列可用。
- 索引脚本（V1.1.130）：`ux_ssg_user_itcode_lower ON ssg_user (lower(itcode))` 与 `idx_ssg_permission_audit_prev (target_type, target_id, target_itcode, id DESC)`。用表达式索引而非改写 SQL，是因为一处 DDL 能同时覆盖 Java 侧 4 处调用与 PL/pgSQL 函数 `ssg_is_active_admin`；已确认 `ssg_user` 现有 288 行 itcode 全小写、无空白。
- `listUsers` 补 `clearPage()`（原只有 `listMenuUsers` 有），两者改 `try/finally` + `PageHelper.clearPage()`。
- `application.yml` 的 `mybatis.configuration.map-underscore-to-camel-case` 对 MP 无效（MP 只读 `mybatis-plus.*`），是死配置；改为 `mybatis-plus.mapper-locations: classpath*:/mapper/**/*.xml`。
- 被实测否决的改动：`listUsers` 的 4 个 `string_agg` 子查询拆成批量查询后，tst 库 EXPLAIN ANALYZE 5 次中位数在 pageSize 10/20/50/100/200 上分别 0.99x / 0.98x / 1.05x / 0.98x / 1.05x，基本持平，而三条 SQL 增加维护成本，故回退。
- tk.mybatis 移除与 mybatis-spring 3.0.4 锁定的技术细节见文档 47c46f5f，不在此重复。
