---
id: "68e95ff1-b38b-4461-a184-870bff5c7262"
title: "Admin Control permission 包 XML 化（MR !578）修订：listUsers 拆分回退、实测持平、src/test 编译失败根因"
description: "推翻旧文档的 listUsers 拆分收益（5 次中位数 0.99–1.05x，三次往返无收益）与回退理由、排序 collation 与 NULL 转空串语义缺陷、1152 值/219 行审计结果零差异、chatbi 两个测试 18 个 any() vs 主代码 25 参数根因、MapperXmlSmokeTest 3 用例通过、permission 仅占 admin 548 条注解 SQL 的 15% 与剩余范围优先级。"
status: "active"
created_at: "2026-11-18T08:50:15.354Z"
updated_at: "2026-11-18T08:50:15.354Z"
content_hash: "82254cec98b08d7e3057889517ecb679b53ee136ab67b8644a13253b78bd7ef0"
source_paths:
  - "smartop-service/admin/src/main/resources/mapper/permission/PermissionManagementDao.xml"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/service/PermissionManagementService.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/permission/MapperXmlSmokeTest.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/chatbi/service/ChatbiEventExportServiceStreamTest.java"
session_ids:
  - "mnemon-run-v1-d22e3ce2-081e-4c95-995d-b0fc94290172"
memory_body_ids:
  []
---

# Admin Control「Access control」MR3 修订记录：listUsers 拆分回退、结果一致性验证、src/test 编译失败根因

> 本文是文档 5ed1a09b-4e8b-4374-bd69-26e2021bf97a（permission 包注解 SQL 全面 XML 化，记录于 commit bffa7d503、18 文件 +1235/−911）的**修订与纠错**。那篇文档里两条关键结论在后续复核中被推翻：**listUsers 拆成 3 条 SQL 已被回退**，「11.19ms → 3.83ms、2.9 倍提升」**不成立**；smoke test 也已从「待启用」变为「已执行通过」。方案论证见文档 5e8a14c4-809f-47fa-bc84-63669530e951，admin 模块整体注解 SQL 基线见文档 3ebbfe47-3740-42a5-9292-e7808dfa2d60。后续 agent 若依据旧文档继续拆分 listUsers，请先读第 2 节。

## 1. 最终状态（worktree C:/Users/63588/lenovo/ssg-sales-ops/.agents/worktrees/admin-control-sql-refactor）

分支 feature/admin-control-permission-sql-admin → admin-release，3 个 commit，**16 个文件 +1137 / −911**：

    a34ab95a5 chore(admin): 删除回退拆分后遗留的两个未引用 DTO
    3d15920b5 refactor(admin): 回退 listUsers 拆分，只保留 XML 化与缺陷修复
    bffa7d503 refactor(admin): 权限模块注解 SQL 迁入 XML 并优化 Access control 查询

MR !578 state=opened、merge_status=can_be_merged、无冲突、**未合并**（合并留给人工审核）。描述已重写：删掉 2.9 倍那段，换成「实测持平」表并补上「本次只覆盖 permission 包」的范围说明（desc_len 由 4850 增至 5202）。编译 mvn -o -T 4 -pl smartop-service/admin compile → BUILD SUCCESS（约 2m23s）。

## 2. 被推翻的性能结论（最重要的一条）

**错在哪**：旧文档的 11.19ms / 3.83ms 来自单次 EXPLAIN ANALYZE。11.185ms 里含首次冷规划开销（14.271ms 量级），单次测量把规划噪声当成收益。

**正确测法**：同一查询跑 5 次取中位数，对比「单条 SQL（规划+执行）」与「拆成 3 条」：

| pageSize | 单条 SQL | 拆 3 条 | 比值 |
|---|---|---|---|
| 10 | 2.670 ms | 2.697 ms | 0.99x |
| 20 | 4.057 ms | 4.159 ms | 0.98x |
| 50 | 8.779 ms | 8.388 ms | 1.05x |
| 100 | 19.985 ms | 20.389 ms | 0.98x |
| 200 | 43.614 ms | 41.660 ms | 1.05x |

**结论：性能基本持平，而 3 条 SQL 就是 3 次网络往返。收益不存在、成本真实 → 回退拆分。** 因此 listUsers 保持原 SQL 形状，只把注解 SQL 搬进 XML；删除为拆分新增的 PermissionUserRoleLink、PermissionUserGroupLink 两个 DTO、两个 DAO 方法与 PermissionManagementService 里的 groupingBy 拼接逻辑（commit 3d15920b5 回退，a34ab95a5 收尾删除死代码）。PermissionManagementDao.xml 中 listUsers 仍保留 4 个 string_agg 相关子查询（role_ids 按 ur.role_id、group_ids 按 gm.group_id、group_names 按 g.group_name、role_names 按 r.role_name 排序）。

### 2.1 顺带发现的语义缺陷（回退的第二个理由）

拆分方案在 Java 侧重新拼接字符串，无法复现 PostgreSQL 行为：

- role_ids 按 role_id 数值排序，role_names 按 role_name 排序——**两条 SQL 的排序规则本来就不同**，Java 侧按同一顺序拼接会错位；
- 空集合在 SQL 里是 NULL，Java 拼接会变成空串 ""。

回退后该缺陷自然消失。教训：**跨 SQL/Java 的重建型重构必须逐字段比对，并注意 NULL 与排序 collation。**

## 3. 结果一致性验证（已完成，全部 0 差异）

| 对象 | 规模 | 差异 |
|---|---|---|
| listUsers 的 role_ids / group_ids / group_names / role_names | 288 用户 × 4 字段 = 1152 值 | 0 |
| 审计 previous_detail（coalesce 改为 IS NOT DISTINCT FROM） | 219 行，其中 44 行有值 | 0（PREFIX_DIFF=0，RESULT: IDENTICAL） |

## 4. mvn test 跑不了的根因（不属于本次改动）

admin-release 的**测试源集本身编译不过**，mvn test-compile 失败点全部在 chatbi 的两个导出测试：

    ChatbiEventExportServiceEstimateTest.java:75  无法将 countSessionsForExport 应用到给定类型
    ChatbiEventExportServiceStreamTest.java:127   无法将 querySessionsForExport 应用到给定类型
    要求: String,String,...,String,boolean,Integer,Integer   ← 25 个参数
    找到: Object,Object,...                                  ← 18 个 any()

根因：smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java:931（querySessionsForExport）与 :971（countSessionsForExport）现在有 25 个参数，而 ChatbiEventExportServiceStreamTest.java:127/162/214、ChatbiEventExportServiceEstimateTest.java:75 的 Mockito 打桩仍写 18–20 个 any()。提交历史佐证两条线各走各的：测试类最后一次改动 f953c839e，主代码最后一次改签名 6731c8b1d。

**对任何在该仓库跑 admin 单测的人**：这不是 permission 改动引入的，是主代码加参数未同步测试。修法是给这两个 mock 补齐 25 个参数（改动很小），由 chatbi 负责人修最合适。

## 5. MapperXmlSmokeTest：已执行通过

smartop-service/admin/src/test/java/com/lenovo/admin/permission/MapperXmlSmokeTest.java 的 3 个用例：6 个 XML 全部可被 XMLMapperBuilder 解析、每个 DAO 方法都有对应 statement、无孤儿 statement（6 个 DAO 接口共 84 条 statement + 4 个 sql 片段）。为使测试源集可编译，**临时把上述两个坏测试文件移出 → 跑测 → git checkout 还原**，工作区最终干净：

    [INFO] Running com.lenovo.admin.permission.MapperXmlSmokeTest
    [INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.810 s
    [INFO] BUILD SUCCESS

这才是「XML 迁移动了 84 条语句却不连库」的构建期安全网：语法错、include 引用错、方法名与 id 不一致都不会在编译期暴露，只有启动或首次调用才炸。chatbi 测试修好后 mvn test 会自动带出它。

## 6. 本次实际交付价值与未覆盖范围

本 MR 的真实价值（不含已被推翻的性能故事）：84 条注解 SQL 变成可维护 XML；审计多选筛选缺 AND 导致的必现 500 修复；审计过滤条件不再复制 40 行；两条索引（含 lower(itcode) 表达式唯一索引）；分页参数泄漏修复；mybatis-plus.mapper-locations 配置修复（否则 XML 根本不加载）。

**覆盖范围只有 permission 包**：整个 admin 模块约 548 条注解 SQL 中改了 84 条，约 15%。同页面其余部分未动：

| 页面/页签 | 包 | 未改注解 SQL |
|---|---|---|
| User Logs、People | chatbi | 188 @Select + 58 写（最大头，也是 User Logs/People 数据源） |
| Usage Issue Feedback | usagefeedback | 26 + 12 |
| Inspection Report | datainquiryprobe / aiprobe | 8 + 1 |
| Notifications | notification | 11 + 11 |
| Jira | jira | 9 + 9 |
| Scheduler / Trino Sync | dataintegration | 12 |
| Admin tabs | adminmenu | 1 + 2 |
| Access control 老接口 /menu /user /role | menu / user / role | 56 + 41 + 19（$ {} 直接拼接、SELECT *、orderBy 插值，注入风险最高） |

优先级建议：menu/user/role 安全风险最高宜优先；chatbi 体量最大宜其次。

## 7. 父对话中的协作约束

- 用户强调**单元测试必须实际跑通**，不接受用独立程序或静态验证替代（本次即因此把 smoke test 真正跑了一遍）。
- 助手遵守「不改别人模块的代码」：chatbi 的两个坏测试只做临时移出与还原，未修改其内容。

## 8. 复用清单（source paths）

- smartop-service/admin/src/main/resources/mapper/permission/PermissionManagementDao.xml
- smartop-service/admin/src/main/java/com/lenovo/admin/permission/service/PermissionManagementService.java
- smartop-service/admin/src/test/java/com/lenovo/admin/permission/MapperXmlSmokeTest.java
- smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java（:931、:971 的 25 参数签名）
- smartop-service/admin/src/test/java/com/lenovo/admin/chatbi/service/ChatbiEventExportServiceStreamTest.java（:127/:162/:214 过时 mock）
