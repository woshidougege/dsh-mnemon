---
id: "753fabcc-76fb-4326-ada3-4e35c7994d30"
title: "admin-release 发布启动两连撞：V1.1.129 重号改 1.1.130（MR !585）+ MyBatis 注解 SQL 裸 < 破坏 XML 解析（MR !586）"
description: "2026-11-18 发布包启动失败的两个卡点：① admin-release 上 Create_chatbi_auto_review_rule 与 Seed_service_portfolio_kpi_cards 同为 V1.1.129，Flyway 抛 duplicate version 中止迁移，改种子脚本名为 V1.1.130（MR !585，仅改名）；② ChatbiReviewJiraMapper 常量 CANDIDATE_SELECT 中裸 `<` 被 @Select(\"<script>\"+常量) 当 XML 解析，SAXParseException columnNumber 3520 导致 Mapper 注册失败，因该常量同时被普通 @Select 复用故不能写 &lt;，改为反向比较 e.event_time > ssg_chatbi_event.event_time（MR !586）。含复用手册与 1.1.130 跨分支重号遗留风险。"
status: "active"
created_at: "2026-11-18T10:43:20.082Z"
updated_at: "2026-11-18T10:43:20.082Z"
content_hash: "be21502676217a4ad909457707bde8b8f5f1656da26f5a16d0eca32e977b4be9"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiReviewJiraMapper.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.130__Seed_service_portfolio_kpi_cards.sql"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.129__Create_chatbi_auto_review_rule.sql"
session_ids:
  - "mnemon-review-v1-071f7bca-bcbf-46d8-a796-bad504d9e848"
memory_body_ids:
  []
---

# admin-release 发布启动两连撞：V1.1.129 重号收尾（MR !585）+ MyBatis 注解 SQL 裸 `<` 破坏 XML 解析（MR !586）

2026-11-18 18:2x–18:42，为「今天要审发布生产」打出的 `admin-release` 包连挂在两道**启动期**关卡上。本文记录两处根因、改法理由、可复用判据与遗留风险。Flyway 重号的机制细节不在这里重复，见文档 `561cc78d-6747-48d6-9820-7e3e2e874b36`。

## 0. 为什么两个坑排队出现

`!581`（评审链路改造）11-18 17:41 合进 `admin-release` 之后**从未真正启动成功过**：Flyway 校验在启动最前面就终止进程，Mapper 加载这一步根本走不到。所以第一个坑修好后，第二个坑才第一次暴露——「编译过 ≠ 能启动」。

## 1. 坑一：`admin-release` 上两条 `V1.1.129`（不需要改脚本内容）

| 文件 | 提交 | 并入时间 |
|---|---|---|
| `V1.1.129__Create_chatbi_auto_review_rule.sql` | `1cd2818d7` refactor(review): 规则表结构直接并入 V1.1.129，去掉建了又删的 1.1.130 | 2026-11-17 13:52:06 |
| `V1.1.129__Seed_service_portfolio_kpi_cards.sql` | `c6d8dfc91` refactor(service-portfolio): KPI 卡去掉代码硬编码兜底，改由配置表驱动 | 2026-11-17 22:47:28 |

报错核心串：`Found more than one migration with version 1.1.129` → 整轮迁移中止 → Bean 创建失败 → 服务起不来（堆栈停在 `DbMigrate.migrate` + `FlywayMigrationInitializer.afterPropertiesSet`）。

**测试环境同样报错，不是环境差异**：tst 上一次成功部署是 11-17 15:11，那时包内只有一个 129；22:47 的 Service Portfolio 提交合入 `admin-release` 后，任何新包都同时带两个 129。核对两侧 `flyway_schema_history`：`version = 1.1.129` 各只有一行，`description = Create chatbi auto review rule`，`success = true`。

**改号选择**：不动 1.1.129（两环境记录都已是评审规则建表脚本，改它会让建表脚本换号重跑一遍），把后合入的种子脚本改名：

```
smartop-service/admin/src/main/resources/db/migration/
  V1.1.129__Seed_service_portfolio_kpi_cards.sql
    → V1.1.130__Seed_service_portfolio_kpi_cards.sql
```

只改文件名 + 加 5 行说明注释，SQL 内容一字未动（脚本本身以 `metric_key` 为幂等键，`sp_sp_kpi_card` 由 1.1.128 建好，重复执行安全）。提交 `faaeb7f8e`，分支 `fix/dup-migration-129` → `admin-release`，**MR !585**（1 文件，`can_be_merged`）。部署后预期：`1.1.129` 已应用且 checksum 不变 → 跳过；`1.1.130` 执行 8 条 KPI 卡种子。

## 2. 坑二：`@Select("<script>" + 常量)` 里的裸 `<`（必须改代码）

启动时 mapper 注册阶段失败：

```
18:34:30.250 [main] ERROR org.mybatis.spring.mapper.MapperFactoryBean - Error while adding the mapper 'interface com.lenovo.admin.chatbi.dao.ChatbiReviewJiraMapper' to configuration.
org.apache.ibatis.builder.BuilderException: Error creating document instance.  Cause: org.xml.sax.SAXParseException; lineNumber: 1; columnNumber: 3520; The content of elements must consist of well-formed character data or markup.
	at org.apache.ibatis.parsing.XPathParser.createDocument(XPathParser.java:262)
	at com.baomidou.mybatisplus.core.MybatisMapperAnnotationBuilder.buildSqlSourceFromStrings(MybatisMapperAnnotationBuilder.java:633)
```

定位：`smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/dao/ChatbiReviewJiraMapper.java`，常量 `CANDIDATE_SELECT` 中第 87 行

```java
+ "     AND ssg_chatbi_event.event_time < e.event_time"
```

被 `@Select("<script>" + CANDIDATE_SELECT ...)`（`selectCandidates`，第 139–150 行）当作 XML 解析，裸 `<` 非法，解析在第 3520 字符处中断 → Mapper 加不进配置 → 服务起不来。该行是 11-18 给建单正文加「同一会话上一轮提问」时写的。

### 为什么不能改成 `&lt;`

同一个常量 `CANDIDATE_SELECT` 还被 `findCandidateByTurn` 复用，而那一处是**普通 `@Select`，不做 XML 解析**：

```java
@Select(CANDIDATE_SELECT + " WHERE r.status = 'WRONG'" + " AND r.session_id = #{sessionId}" ...)
```

写成 `&lt;` 会原样发给 PG 报语法错。**一个常量两处用，转义只能救一半。**

### 改法：比较方向反过来（提交 `6b05b81a`，MR !586）

```diff
- + "     AND ssg_chatbi_event.event_time < e.event_time"
+ + "     AND e.event_time > ssg_chatbi_event.event_time"
```

语义完全等价，`>` 在 XML 里本就合法，普通 SQL 也照常认。同时在常量上加注释写明「这里既不能用裸 `<` 也不能用 `&lt;`」。分支 `fix/review-mapper-xml` → `admin-release`，1 文件（+5/-1），`can_be_merged`。`mvn -o -T 4 -pl smartop-service/admin -am -DskipTests compile` 已 BUILD SUCCESS——但编译不检查注解 SQL 的 XML 合法性，**只有启动才能真正验证**。

## 3. 复用手册（本项目）

1. **加迁移前先扫最大版本号**，别只看自己分支：
   `git ls-tree origin/admin-release smartop-service/admin/src/main/resources/db/migration | Select-String 'V1.1.1'`
   注意版本号不可按字符串排序（`V1.1.100` 会「小于」`V1.1.99`），见文档 `47c46f5f-fd19-41b1-a6e6-4598dbfe407d`。
2. **只有被 `<script>` 包裹的注解 SQL 才做 XML 解析**。判断一个 `<` 危险与否，先看该常量是否出现在 `@Select("<script>" ...)` / `@Insert("<script>" ...)` 里；被普通 `@Select(常量)` 复用的常量，既不能写裸 `<` 也不能写 `&lt;` → 用反向比较或 `!=` 改写。
3. `<>`（不等于）同样含裸 `<`，例如 `AND btrim(x) <> ''` 被塞进 `<script>` 会同样炸；改用 `!=`。
4. 本批同源改动里 `insertAutoReviews`、`listCommentsForSessions` 的 `<script>` 块只有 `<foreach>` 标签、无裸运算符，已确认无第三个同类错误。
5. 发版前先在 tst 起一次，确认过启动阶段（Flyway + Mapper 加载）再冲生产。

## 4. 遗留风险

- 文档 `47c46f5f` 记录：分支 `admin-control-sql-refactor` 的提交 `b719fadea` 已把 `V1.1.100` 改名为 **`V1.1.130`**。本次又占用了 `1.1.130`（`V1.1.130__Seed_service_portfolio_kpi_cards.sql`）。**该分支日后合入 `admin-release` 时会再次撞号**，需在合并前给其中一条改号（沿用「改后合入的那条」原则）。
- `admin-release` / `master` / `develop` 当时均无其他 1.1.130，故本次改号不会与主干二次撞车。
