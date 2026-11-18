---
id: "5ed1a09b-4e8b-4374-bd69-26e2021bf97a"
title: "Admin Control Access control MR3 落地记录：permission 包注解 SQL 全面 XML 化（MR !578 实测与限制）"
description: "permission 包 6 个 DAO 的 911 行注解 SQL 迁入 6 个 XML（86 statement + 4 片段）、listUsers 拆成主查询+2 条批量 IN 的前后实测（11.19ms/448 buffers → 3.83ms/~263）、表达式唯一索引 lower(itcode) 的理由、审计多选筛选缺 AND 的必现 500 bug、mybatis-plus.mapper-locations 修复、smoke test 因 src/test 无法编译而待启用、MR !578 未合并状态。"
status: "active"
created_at: "2026-11-18T08:20:58.980Z"
updated_at: "2026-11-18T08:20:58.980Z"
content_hash: "31f4b77ee0fa12c5ded5f69047c85b057d21b646695cc449c9e3bb3c02ff87cf"
source_paths:
  - "smartop-service/admin/src/main/resources/mapper/permission/PermissionManagementDao.xml"
  - "smartop-service/admin/src/main/resources/mapper/permission/PermissionAuditDao.xml"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/service/PermissionManagementService.java"
  - "smartop-service/admin/src/main/resources/db/migration/V1.1.100__Optimize_admin_control_permission_query.sql"
  - "smartop-service/admin/src/main/resources/application.yml"
session_ids:
  - "mnemon-run-v1-ce4d8f5a-c538-4dab-a3d5-d94b007ff70b"
memory_body_ids:
  []
---

# Admin Control「Access control」MR3 落地记录：permission 包注解 SQL 全面 XML 化（MR !578）

> 前序方案与 SQL 体检证据见文档 `5e8a14c4-809f-47fa-bc84-63669530e951`（Admin Control「Access control」试点：SQL 体检与 MR1–MR3 改造方案）。本文只记录 **MR3 实际落地后的代码形态、实测数据、已知限制**，避免重复方案论证。

## 1. 状态

| 项 | 值 |
|---|---|
| 分支 | `feature/admin-control-permission-sql-admin` → 目标 `admin-release` |
| 提交 | `bffa7d503` |
| 变更规模 | 18 个文件，+1235 / −911 |
| MR | !578 `refactor(admin): 权限模块注解 SQL 迁入 XML 并优化 Access control 查询`（state=opened，mergeable，无冲突） |
| 合并/部署 | **未合并、未部署**，合并按钮留给人工审核 |
| 编译 | `mvn -o -T 4 -pl smartop-service/admin compile` → BUILD SUCCESS（972 文件，约 1m33s） |

工作目录为 worktree `C:\Users\63588\lenovo\ssg-sales-ops\.agents\worktrees\admin-control-sql-refactor`（worktree 自动生成的分支名 `admin-control-sql-refactor` 不符合规范，push 前重命名为上面的 `feature/...-admin`）。

## 2. 落地内容

1. **6 个 XML 承接全部注解 SQL**：`smartop-service/admin/src/main/resources/mapper/permission/` 下 EffectivePermissionDao.xml、EffectivePermissionQueryDao.xml、PermissionAuditDao.xml、PermissionManagementDao.xml、RoleManagementDao.xml、UserGroupManagementDao.xml，共 **86 条 statement + 4 个 `<sql>` 片段**。
2. **6 个 DAO 接口瘦身**：只留方法签名 + `@Param`，注解里的 SQL 字符串全删（Java 侧净减约 900 行）。
3. **新增两个批量查询载体 DTO**：`PermissionUserRoleLink`、`PermissionUserGroupLink`。
4. **迁移脚本** `V1.1.100__Optimize_admin_control_permission_query.sql`：只建索引，不写任何数据（符合该项目「migration 只管表结构」的约定）。
5. **配置修复** `application.yml`（约 L86-88）：原 `mybatis:` 段是死配置，改为 `mybatis-plus.mapper-locations`，否则 XML 根本不会被加载。

## 3. 关键改造与理由

### 3.1 listUsers：1 条重 SQL 拆成 3 条

原 SQL 在 SELECT 列表里塞 4 个 `string_agg` 相关子查询，并对每行调用 `ssg_is_active_admin(u.itcode)` —— 20 行数据要跑 81 次子查询，仅规划就要 14 ms。

改造后：主查询只取用户自身字段；角色、分组各用一条 `WHERE user_id = ANY(...)` 批量取回；`PermissionManagementService.fillUserRoleAndGroupLabels` 用 `groupingBy` 在 Java 里拼回原来的逗号分隔字符串（保持接口返回结构不变，前端零改动）。

实测（tst 库，`p_renxp4`@`ssg_ops_agent`）：

| 形态 | 执行 | 规划 | Buffers |
|---|---|---|---|
| 改造前（单条） | 11.19 ms | 14.27 ms | 448 |
| 改造后（3 条合计） | 3.83 ms | 0.18 ms | ~263 |

约定：**保留 listRoles 里的 2 个 count 子查询**（角色表仅 9 行，收益 <0.1 ms），**不动 listEffectiveMenuStats 的 O(users×menus) 双层循环**（属缓存问题，改 SQL 无解）。

### 3.2 索引：选表达式唯一索引而非改 SQL

`lower(itcode) = lower(?)` 让 `ssg_user` 全表扫描、`index_itcode` 失效。方案不是改写 4 处 Java 调用，而是加一条表达式唯一索引：

```sql
CREATE UNIQUE INDEX ux_ssg_user_itcode_lower ON public.ssg_user (lower(itcode));
```

理由：一处 DDL 同时覆盖 Java 侧 4 处调用与数据库函数 `ssg_is_active_admin` 内部的同一谓词，零语义风险。实测未加索引时单次函数调用 1.265 ms，预期加索引后降到 0.05 ms 量级。

### 3.3 顺带修掉一个必然 500 的真实 bug

原 `PermissionAuditDao` 筛选条件的前三个 `<if>` **没有写 `AND`**，靠 `<where>` 剥离首词兜底：只传一个条件时正常，但同时选两个及以上条件会拼成 `WHERE lower(a.operator) = ? upper(a.action) = ?` → SQL 语法错误，审计页面多选筛选必然报错。现已让四个条件都显式带 `AND`，并由 `<sql id="auditFilter">` 片段供列表与 count 共用。`previous_detail` 子查询用 `IS NOT DISTINCT FROM` 比较 target_id/target_itcode（NULL 安全）。

## 4. 验证方式

- `MapperXmlSmokeTest`（`src/test/java/com/lenovo/admin/permission/`）逐条渲染 86 条 statement 做静态校验；另用等价脚本在真库跑过 EXPLAIN。
- **已知限制**：该 smoke test 目前跑不起来 —— `admin-release` 上 `src/test` 本身编译不过（`ChatbiEventExportServiceStreamTest` 与主代码签名不一致，属既有问题）。测试目录修好后可直接启用；当前 XML 正确性靠等价脚本验证。
- 人工验收路径：起服务 → flyway 自动建 2 个索引（表仅 285/224 行，锁为毫秒级）→ 打开 Access control 的 Users 与 Audit log 页签 → 同时选两个筛选条件（旧代码会报错）。

## 5. 范围外 / 后续

- `menu`、`user` 老包（`$\{}` 拼接、`SELECT *`）按既定范围未触碰，是下一个试点。
- 审核顺序建议：PermissionAuditDao.xml（auditFilter + previous_detail）→ PermissionManagementDao.xml（listUsers 主查询 + 两条批量 ANY）→ PermissionManagementService.fillUserRoleAndGroupLabels → 其余 4 个 XML 为纯搬迁。
