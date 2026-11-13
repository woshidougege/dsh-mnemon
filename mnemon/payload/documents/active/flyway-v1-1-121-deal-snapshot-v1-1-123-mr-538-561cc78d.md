---
id: "561cc78d-6747-48d6-9820-7e3e2e874b36"
title: "Flyway 迁移版本号重复排查与改号修复：V1.1.121 冲突 → deal snapshot 改号 V1.1.123（MR !538）"
description: "admin-release 上 V1.1.121 被 deal snapshot 与 review_auto 同时占用导致 Flyway 启动失败；记录 DUPLICATE_VERSIONED_MIGRATION 机制（CompositeMigrationResolver，不受 validate-on-migrate:false 保护）、三环境 flyway_schema_history 证据、改号到 V1.1.123 的落地（commit 0eea3629f / MR !538）、上线顺序与同类事故复用手册。"
status: "active"
created_at: "2026-11-13T06:02:08.080Z"
updated_at: "2026-11-13T06:02:08.080Z"
content_hash: "2ff5315fdcbc41f130e67c54ea6c8996ff22b53e6790edc187c1e2f01585e6d7"
source_paths:
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.123__Create_bookings_simu_deal_week.sql"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chat/service/AskDriverCustomerService.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/chat/service/LudpPivotTest.java"
  - "pom.xml"
session_ids:
  - "mnemon-run-v1-6960f982-3ba5-4928-b3ec-b5a9ed9ed6cc"
memory_body_ids:
  []
---

# Flyway 迁移版本号重复的排查与改号修复（V1.1.121 冲突 → deal snapshot 改号 V1.1.123，MR !538）

2026-11-13 在 `dataagent/ssg-sales-ops` 的 `admin-release` 分支上发现 `V1.1.121` 被两个迁移文件同时占用，**Flyway 启动阶段直接失败**（admin 服务起不来）。本文记录根因机制、三环境证据、改号方案与落地结果，可作为后续同类「版本号撞车」事故的复用手册。

## 1. 冲突事实

| 版本 | 文件 | 作者 / 提交 | 并入 admin-release |
|---|---|---|---|
| 1.1.121 | `V1.1.121__Create_bookings_simu_deal_week.sql` | zhangyong `b0734814d`（chore: renumber deal snapshot migration to V1.1.121） | `3b9e2e1a6`，2026-11-13 13:41:12 |
| 1.1.121 | `V1.1.121__Create_ssg_chatbi_question_review_auto.sql` | renxp4 `5e6bd7cfe`（feat(chatbi): configurable auto review rules） | `612a2af05`，2026-11-13 11:54:23 |

成因：zhangyong 11:14 改号时只看到 1.1.120 被「问题评审日志」占用（提交说明写着 "V1.1.121 is the deal snapshot"），而自动评审脚本 11:31 也取了 121；随后 `feature/ask-simu-deal-snapshot` 13:41 才合入，冲突才落到 admin-release 上。全仓其余 123 个版本号无重复。

## 2. 根因：重复版本号不受 `validate-on-migrate: false` 保护

`application.yml` 里配的 `validate-on-migrate: false` + `VALIDATE_NOT_PRESENT` + `ignore-missing-migrations: true` **完全挡不住**这一条。重复检查在「发现可用迁移」阶段，不在 validate 阶段：

- `org/flywaydb/core/internal/resolver/CompositeMigrationResolver.java`：`static void checkForIncompatibilities(List<ResolvedMigration> migrations)`（第 83 行），由 `doFindAvailableMigrations()` 在第 126 行调用；命中即抛 `CoreErrorCode.DUPLICATE_VERSIONED_MIGRATION`：

```
Found more than one migration with version 1.1.121
Offenders:
-> classpath:db/migration/V1.1.121__Create_bookings_simu_deal_week.sql (SQL)
-> classpath:db/migration/V1.1.121__Create_ssg_chatbi_question_review_auto.sql (SQL)
```

版本来源：Spring Boot 3.0.13 的 BOM 是 import 方式引入的，把 flyway-core 锁在 9.5.1；**import 进来的 BOM 内部属性无法在项目里覆盖**，所以在根 `pom.xml` 显式写 `flyway-core` 11.20.3。Flyway 10 起 PostgreSQL 支持拆到独立模块，必须同时引 `flyway-database-postgresql`，否则启动报 `No database found to handle jdbc:postgresql://...`。

**结论：一旦 admin-release 的包部署，tst/pre/prod 三套环境的 admin 都会在建连/迁移阶段起不来，与 validate 配置无关。**

## 3. 三环境 `flyway_schema_history` 证据（p_renxp4 只读查询）

| 环境 | 1.1.119 | 1.1.120 | 1.1.121 | 1.1.122 |
|---|---|---|---|---|
| tst 10.62.79.148 | — | 已跑 review log（12:00:40） | 已跑 **review_auto**（12:00:41） | 未跑 |
| pre 10.62.79.221 | — | 已跑（12:01:59） | 已跑 **review_auto**（12:01:59） | 未跑 |
| prod 10.62.79.215 | 未跑 | 未跑 | 未跑（最新 = 1.1.118，2026-11-12 14:35） | 未跑 |

这张表决定了「改哪一个 121」：tst / pre 已经把 1.1.121 记成 `review_auto`，所以**不能动 review_auto 那个文件**，只能把 deal snapshot 往后挪。

## 4. 落地方案

改 zhangyong 的 deal snapshot 为 **V1.1.123**：保留 121 = review_auto、122 = oh_l4 视图，与 zhangyong 团队前两次改号做法一致。

- 仓库 / 工作树：`dataagent/ssg-sales-ops`，独立 worktree `C:\Users\63588\lenovo\ssg-sales-ops-wt-mig121`（主工作区保持不动）
- 分支：`fix/renumber-deal-snapshot-121-admin`，基于 `origin/admin-release` `c21724862`（Merge branch 'feature/ask-drivers-oh-l4'）
- commit：`0eea3629f` fix(admin): 把 deal snapshot 迁移改号为 V1.1.123，修复 1.1.121 版本号重复（3 files changed, 2 insertions(+), 2 deletions(-)）
- 改动：
  1. `smartop-service/admin/src/main/resources/db/migration/V1.1.121__Create_bookings_simu_deal_week.sql` → `V1.1.123__Create_bookings_simu_deal_week.sql`，**纯改名，内容一字未动**（Git 识别 rename 100%）
  2. `smartop-service/admin/src/test/java/com/lenovo/admin/chat/service/LudpPivotTest.java:312` 的 classpath 资源名同步改为 `db/migration/V1.1.123__...` —— 该用例用 `getResourceAsStream` 按文件名读脚本，只改名不改引用会 NPE
  3. `smartop-service/admin/src/main/java/com/lenovo/admin/chat/service/AskDriverCustomerService.java:136` javadoc「见 V1.1.121」→「见 V1.1.123」
- MR：**!538** `fix/renumber-deal-snapshot-121-admin` → `admin-release`，https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/538 ，`state=opened`、`merge_status=can_be_merged`、`has_conflicts=False`、sha `0eea3629f786`。按项目规矩**只提不点合并**，等用户确认。

为什么不反向改 review_auto 的 121：tst/pre 历史里会变成 missing，新号码会重跑该脚本，只能靠脚本幂等兜底。

依赖核对：`V1.1.122` 建的是 `bookings_simulation_ssg_kpi` 上的视图和索引，**不依赖** `bookings_simu_deal_week`，122 先跑、123 后跑无依赖问题。

## 5. 验证

- 迁移目录 125 个文件，重号检查：**无重复版本号**；全仓 `V1.1.121` 引用清零（原本只有上述 2 处）
- `mvn -o -T 4 -pl smartop-service/admin -am test -Dtest=LudpPivotTest` → **BUILD SUCCESS，Tests run: 19, Failures: 0, Errors: 0**，证明改号后 classpath 能读到 1.1.123 并通过内容断言
- 坑：多模块 `-am` 且 `-Dtest=单类` 时，`smartop-component-jsqlparser5` / `smartop-common-core` 会以 `No tests matching pattern "LudpPivotTest" were executed!` 让整个构建 FAILURE；加 `-Dsurefire.failIfNoSpecifiedTests=false`（或只在 admin 模块内跑）即可

## 6. 上线影响

- 合并后：tst / pre 下次启动只多跑 1.1.122、1.1.123
- prod 停在 1.1.118，首次部署会一次跑 1.1.119 → 1.1.123 五条；其中 1.1.122 给约 1500 万行的 `bookings_simulation_ssg_kpi` 建索引且**非 CONCURRENTLY**，必须选同步空窗期
- 1.1.x 缺号 41/47/48/49/73/76/107/108 是历史改号遗留，Flyway 不敏感，不是问题

## 7. 同类事故复用手册

1. 全仓扫 `db/migration/V*__*.sql` 的版本号，找重复；同时算出当前最大版本
2. **先查三套环境的 `flyway_schema_history`**：已被历史登记的版本号不能改（否则变 missing + 重跑），优先改「尚未在任何环境登记」的那一侧
3. 改号只做 `git mv`，脚本内容不动；旧号码在 tst/pre 无记录时可安全跳过
4. 全仓 grep 旧版本号：脚本名常被测试（`getResourceAsStream`）和 javadoc 以字符串形式引用
5. 检查新号码与中间版本（如 1.1.122）之间有无表/视图依赖，决定是否需要调序
6. 验证：重号检查 + 定向跑相关测试；提交走 MR，不自行合并

## 8. 附带隐患：主工作区的未跟踪 SQL 会覆盖同事的 116

本地分支 `feature/admin-control-export-admin` 当时落后 `admin-release` **98 个提交**，工作区有两个**未跟踪**的迁移文件：

- `smartop-service/admin/src/main/resources/db/migration/V1.1.116__Optimize_hc_member_lookup.sql` —— 本地 48 行的**旧版**；远端同名文件 150 行，多了 2026-11-12 补充的 `ssg_hc_member_line_lookup(p_itcode, p_period)` 函数段（与视图 `ssg_hc_member_line` 逐行等价，按 itcode 点查）
- `V1.1.117__Add_chatbi_event_generated_columns.sql` —— 与远端内容一致

这两个文件一旦被 `git add` 提交，会把同事的 116 覆盖回去，**建议直接删掉本地副本**。

## 相关文档

- ChatBI 问数自动评审规则（V1.1.121，本冲突中的另一侧）：`750e10a9-15ec-4ab0-9295-1f45e6d90bfd`
- ChatBI 人工评审留痕（V1.1.120）：`6353a23e-5302-4b9c-9301-049e028565d3`
