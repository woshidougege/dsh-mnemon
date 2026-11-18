---
id: "4ad84c6f-035c-4e07-9b52-19438e2f5f5b"
title: "Admin Control 注解 SQL 全量 XML 化（419 条 / 110 DAO）与真实接口比对交付记录"
description: "admin 模块注解 SQL 639→0 的完成记录：419 条 statement 迁入 121 XML、110 DAO，Java 净删 3454 行；三层验证（文本比对 419/419、起始动词扫描、18082/18083 双服务真实接口比对 243 端点 241 一致 + 2 处时间戳证伪）；真实接口比对抓到的三个缺陷（CTE 常量未展开、FILTER_SQL 同名覆盖、文本块 \\\\. 转义丢失使 bookingPerActivePns 变 0.00）与「迁移/校验共用解析器导致假阴性」教训；同 MR 移除 tk.mybatis、删 datacenter/dict、MP 3.5.12 与 Druid 1.2.27 升级、README 持久层约定；commit b5024d3af / MR !578 未合并与 xxl-job 停用等遗留。"
status: "active"
created_at: "2026-11-18T15:52:15.935Z"
updated_at: "2026-11-18T15:52:15.935Z"
content_hash: "ec0d05ebfe3fbb36493daa500db347bda44040a25ff169bb2ad881fd784be397"
source_paths:
  - "smartop-service/admin/src/main/resources/mapper/gpssus/GpsSusActivePnMapper.xml"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/dao/GpsSusActivePnMapper.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/pojo/dto/GpsSusActivePnRow.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/service/GpsSusService.java"
  - "README.md"
  - "pom.xml"
session_ids:
  - "mnemon-review-v1-26acad0a-aa2b-42e7-b54c-ea6905d17a2f"
memory_body_ids:
  []
---

# Admin Control 注解 SQL 全量 XML 化（419 条 / 110 DAO）与真实接口比对交付记录

2026-11-18 完成。分支 `feature/admin-control-permission-sql-admin`，commit `b5024d3af`，165 个文件改动（+1216 / −4670），MR **!578**（opened，**未合并**，等用户审核）。
https://gitlab.xpaas.lenovo.com/dataagent/ssg-sales-ops/-/merge_requests/578

本文件是「全模块完成」的交付与验证记录。分批评审与风险台账见 `fd202c70-993b-4181-b119-1c8e983999a9`；permission 包 86 条的实测与索引设计见 `5ed1a09b-4e8b-4374-bd69-26e2021bf97a`；发布期两个启动坑见 `753fabcc-76fb-4326-ada3-4e35c7994d30`。本文不重复它们的细节。

## 1. 结果

- `admin` 模块注解 SQL：**639 → 0**（`@Select/@Insert/@Update/@Delete` 残留归零）。
- **419 条 statement 迁入 121 个 XML**，覆盖 110 个 DAO/Mapper；Java 侧净删 **3454 行**。
- 迁移在 worktree 中进行：`C:\Users\63588\lenovo\ssg-sales-ops\.agents\worktrees\admin-control-sql-refactor`。

## 2. 三层验证（缺一层会漏 bug）

1. **文本比对**：改动前注解里的 SQL 与 XML statement 的 SQL，归一化后逐字相等 —— 419/419 一致。
2. **SQL 起始动词扫描**：每条 statement 去标签后必须以 SELECT/INSERT/UPDATE/DELETE/WITH 开头（此层查出过 CTE 头部被常量覆盖）。
3. **真实接口比对**：改动前 jar 起在 **18083**（`admin-release` 主工作区），改动后起在 **18082**（worktree），连同一个 tst 库、同一个 `user` 请求头，逐一调用 Admin Control 全部 GET 端点。

真实接口比对结果：**243 个 GET 端点，241 个响应字节级一致**。剩 2 个差异经「同一服务连发两次请求」证伪为端点自身时间戳，与 SQL 无关：

| 端点 | 差异 | 证据 |
|---|---|---|
| `/bi-admin/overview` | `generatedTime` 微秒级不同 | 同一服务两次请求同在 @68 位置出现差异 |
| `/scheduler/cache/overview` | `ttlSeconds` 597670 vs 597671 | Redis TTL 每秒递减，同服务两次亦不同 |

（首次对比时这三个端点表现为 len 相同、去时间戳后比对才判等；单服务双请求的对照实验是最终定论手段。）

## 3. 真实接口比对抓到的三个真实缺陷（文本层发现不了）

| 缺陷 | 症状 | 根因 |
|---|---|---|
| 常量未展开 | `/chatbi/events/report/questions` 直接 INTERNAL_ERROR | SQL 变成 `WITH fb AS ( AND user_id = ?`，中间整段消失 |
| 同名常量覆盖 | `/api/v1/chat/feedback/mine/dislikes`、`/usage-issue-feedback/mine/summary` 500，`Parameter 'kinds' not found` | `FILTER_SQL` 在两个类各定义一次，引用解析成了另一个类的值，带出并不存在的 `kinds` 参数 |
| 文本块转义丢失 | `/gpssus/active-pn` 的 `bookingPerActivePns` 从 27049.20 变 0.00 | Java 文本块里 `\\.` 未还原成 `\.`，PG 正则由「小数点」变成「反斜杠+任意字符」，所有数字不匹配 |

第三例的排查路径可复用：`GpsSusActivePnMapper.xml` 里 `WHEN trim(lifecycle.bookings_fy2324) ~ '^[+-]?[0-9]+(\\.[0-9]+)?$'` 的双反斜杠字节数（应为单反斜杠）是判据；Java 侧对应 `GpsSusActivePnRow.bookingFy2627Ytd` → `GpsSusService.java:1912` `addActivePnBooking(..., "FY2627", row.getBookingFy2627Ytd())` → `calculateActivePnBooking`。修复后 `bookingPerActivePns new=27049.20 old=27049.20`。

**方法论教训（本任务最重要的一条）**：迁移脚本与校验脚本共用同一套 SQL 解析器，解析器的缺陷在两侧同时表现，形成**假阴性**；文本比对与起始动词扫描都通过，只有真实接口对比暴露问题。因此「同源自校验」不能作为迁移正确性的唯一证据。

## 4. 三类特殊写法及正确处理

| 形态 | 举例 | 处理 |
|---|---|---|
| 常量拼接（含跨类、常量套常量） | `@Select(Q + " ORDER BY ...")` | 迭代求值解析全模块 **2340 个常量定义**，展开为字面量，并按类做作用域（`CONSTS.__byClass`）避免同名覆盖 |
| Java 文本块 | `@Select(""" ... """)` | 还原转义（`\\` → `\`），否则正则类 SQL 语义改变 |
| 数组形式 | `@Select({"<script>", "行1", "行2"})` | MyBatis 以空格 join 元素，迁移时保留元素间空格，不合并成连写 |

## 5. 同一 MR 里的其他改动

- **移除 tk.mybatis**：11 个 DAO 父接口换 MyBatis-Plus `BaseMapper`，15 处调用点改写，10 个实体 `javax.persistence` → MP 注解；根 `pom.xml` 锁 `mybatis-spring 3.0.4`（原来靠 Maven 仲裁降到 2.1.2）。
- **删僵尸模块**：`datacenter`（76 类 / 7483 行）与 `dict`（6 类 / 214 行）——它们访问的 `ciop_*` 表在两个库都不存在，`@DS` 数据源也未定义。
- **升级**：MP 3.5.5 → 3.5.12（换 Boot 3 专用坐标），Druid 1.1.23 → 1.2.27。
- **README 新增「持久层约定」章节**：禁止注册 MP 分页插件、禁止 `selectPage`（会静默全量）、给出 SQL 落点判定表。

## 6. 交付与遗留

审核重点：`ChatbiEventQueryMapper`（51 条）、`MenusDao`（46 条）、`UserDao`（32 条）——常量展开与动态标签集中在此。

用户需要执行：
1. 审核 MR !578（只提交不合并，符合既定要求）。
2. **xxl-job 调度中心**停用 `DataCenterAlert`（`@XxlJob DataCenterAlert`）与 `dueDateCalculateTask` —— 所属类已随 `datacenter` 删除。
3. 合入后本地 `mvn install` 一次 common 模块，否则 `.m2` 里仍是旧的 `smartop-common-datasource`。
4. 18082 / 18083 两个服务仍在跑，可停。

MR 描述「已知遗留」：`$\{}` 直接插值 150 处、`List<Map>` 619 处、PageHelper count 重复算子查询、`permission` 包 4 处 `lower(itcode)` 未命中索引（后者已在 `5ed1a09b-4e8b-4374-bd69-26e2021bf97a` 中用小写表达式唯一索引方案覆盖）。
