---
id: "5e8a14c4-809f-47fa-bc84-63669530e951"
title: "Admin Control「Access control」试点：SQL 体检（EXPLAIN 实测）与 MR1–MR3 改造方案"
description: "Access control 子试点切法与证据：/setting 与 /setting/settings 两层结构、permission 6 个 DAO 映射；lower(itcode) 索引失效与 listUsers 81 次子查询的实测执行计划；审计 previous_detail 建模缺陷与 O(用户×菜单) 内存推演；MR1 索引/函数 → MR2 SQL 形状 2+Java → MR3 permission 包 XML 化（含 mapper-locations 缺失与不做清单）。"
status: "active"
created_at: "2026-11-18T07:32:08.368Z"
updated_at: "2026-11-18T07:32:08.368Z"
content_hash: "f0d43d766303effc2333a7dab8e7b43bd6f56cf76e209bb23f95bca575ced3cb"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/dao/PermissionManagementDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/dao/RoleManagementDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/dao/UserGroupManagementDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/dao/PermissionAuditDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/dao/EffectivePermissionQueryDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/dao/EffectivePermissionDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/service/PermissionManagementService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/service/PermissionEffectiveQueryService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/adminmenu/dao/AdminMenuConfigDao.java"
session_ids:
  - "mnemon-run-v1-088354a5-cba7-4f38-8581-7422f6564efb"
memory_body_ids:
  []
---

# Admin Control 「Access control」试点：SQL 体检与三层改造方案

> 结论先行：试点切 **Access control（含 Audit log）**，不分整页。理由是它唯一闭环、不碰别人的表、数据量极小、可枚举（前端 4 页签 + 1 审计页、后端 26 接口），且同时暴露了索引失效、SQL 形状、注解 vs XML 三类问题 —— 一次试点验证三件事。
> 收益定位要诚实：生产数据量小（285 用户 / 6 受管菜单 / 944 行覆盖 / 224 条审计），**本轮收益在可维护性与索引正确性，不在响应时间**。

## 1. 范围对齐：Admin Control 已经不是「一个页面」

前端 `src/views/Setting/` 下有两个独立路由：

| 路由 | 是什么 | 内容 |
|---|---|---|
| `/setting` | Admin Control 主页面 | 顶部页签 User Logs / Inspection Report / Usage Issue Feedback + 后端可配的快捷页签（表 `ssg_admin_menu_item`） |
| `/setting/settings` | Admin settings（右上角齿轮） | 11 个面板：People、Access control、Audit log、Notification settings、Jira integration、Page focal、Mode groups、Scheduler Center、Admin tabs、Display、Review rules |

关键事实：原先挂在 Admin Control 顶部页签的 `permissions / people / notifications / scheduler / menu-settings` 已整体迁到 `/setting/settings`，旧深链在 `src/views/Setting/index.vue`（约 L88）里 302 过去。因此「Admin Control 的 SQL」现在分散在 **5 个后端包**里，直接整页试点体量过大 —— 这是选择只切 Access control 的直接原因。

## 2. 前端 → 后端映射（Access control 子树）

页面结构：`src/views/Setting/reports/permissions/index.vue` 定义 4 个页签

| 页签 | 组件 | 行数 |
|---|---|---|
| Users | `UserManagement.vue`（pageSize=10） | 658 |
| User Groups | `groups/index.vue` | 464 |
| Roles | `RoleManagement.vue` | 2157 |
| Menu Permissions | `index.vue` 内联 3 模式：Default Access / By Menu / By User | 2566 |
| （独立面板）Audit log | `AuditLogPage.vue` | 1327 |

接口映射：

| 前端调用 | API | Controller | Service | DAO | 表 |
|---|---|---|---|---|---|
| Users 列表 | `GET /permission/users` | PermissionManagementController | listUsers（PageHelper） | PermissionManagementDao#listUsers | ssg_user + 3 张关联表 |
| Users 增删改/踢人 | `POST/PUT/DELETE /permission/users*` | 同上 | create/update/delete/kickUser | insertUser/updateUser/deleteUser + PermissionAuditDao | ssg_user、审计表 |
| User Groups | `/permission/groups/*` | UserGroupManagementController | UserGroupManagementService | UserGroupManagementDao | ssg_user_group、_member、_role |
| Roles | `/permission/roles/*` | RoleManagementController | RoleManagementService | RoleManagementDao | ssg_role、ssg_role_menu、ssg_user_role |
| Menu › Default Access | `GET/PUT /permission/menus`、`/menus/defaults` | PermissionManagementController | listManagedMenuTree / saveMenuDefaults | listManagedMenus / batchUpdateMenuDefaults | ssg_menu |
| Menu › By Menu | `/permission/menus/{id}/users`、`/effective-users`、`/effective-stats` | 同上 | listMenuUsers / listMenuEffectiveUsers / **listEffectiveMenuStats** | PermissionManagementDao + EffectivePermissionQueryDao | ssg_menu、ssg_user_menu |
| Menu › By User | `/permission/users/{itcode}/menus`、`/effective-menus` | 同上 | getUserMenus / saveUserMenus / getUserEffectiveMenus | 同两位 DAO | 同上 |
| Audit log | `/permission/audit/page`、`/audit/filters` | 同上 | listAuditPage | PermissionAuditDao | ssg_permission_audit_log |
| Admin Control 顶部页签 | `/menu-config/items` | AdminMenuConfigController | — | AdminMenuConfigDao（全删全插） | ssg_admin_menu_item |
| 登录后算 menuKeys | `GET /user/get` | UserController | **EffectivePermissionService** | EffectivePermissionDao | ssg_menu、ssg_user_menu |

DAO 规模（`permission/dao` 下 6 个，共 1224 行、59 个 `@Select`、0 个 `extends BaseMapper`）：

```
PermissionManagementDao.java     340 行  15 select  11 <script>
RoleManagementDao.java           320 行  18 select   7 <script>
UserGroupManagementDao.java      206 行  11 select   6 <script>   (@Mapper=true)
PermissionAuditDao.java          190 行   7 select   2 <script>   (@Mapper=true)
EffectivePermissionQueryDao.java 126 行   6 select   1 <script>   (@Mapper=true)
EffectivePermissionDao.java       42 行   2 select   0          (@Mapper=true)
```

SQL 组织方式是**纯注解 + Java 15 文本块 `"""`**，不是 MP Wrapper；全仓库 63 个 `BaseMapper` 一个都不在这里。即同一份「权限配置」业务，老接口（`/menu/*`、`/user/*`）走 MP/Wrapper + `${}`，新接口（`/permission/*`）走纯注解 SQL —— 两套写法隔着一堵墙。生产数据抽查：`itcode` 全小写 285/285（`lower(itcode) <> itcode` 为 0 行），无空值痕迹。

## 3. 体检发现（按严重度）

### P0｜索引写了但用不上（实测 EXPLAIN）

`/permission/*` 每条查询都带 `lower(itcode) = lower(?)`，命中位置：`EffectivePermissionQueryDao:26`（findEffectiveUser）、`PermissionManagementDao:114`/`:122`（findUserByItcode / lockUserByItcode）、`ssg_is_active_admin()` 函数体内部（**每行调一次**）、`PermissionAuditDao:76` 等。

生产 EXPLAIN 实测：

```
Seq Scan on ssg_user u (actual rows=1 loops=1)
  Filter: (lower((itcode)::text) = 'renxp4'::text)
  Rows Removed by Filter: 284
```

`index_itcode`（UNIQUE btree (itcode)）完全未被使用。因数据已全小写，下面改法语义等价且一行见效：

```sql
-- 现状：索引失效
WHERE lower(u.itcode) = lower(#{itcode})
-- 改为：命中 index_itcode
WHERE u.itcode = lower(#{itcode})
```

### P0｜`listUsers` 是「一条 SQL，81 次子查询」

`PermissionManagementDao.java:29-40` 在 SELECT 列表里塞 4 个 `string_agg` 相关子查询 + 1 个逐行函数：`role_ids`、`group_ids`、`group_names`、`role_names`，以及 `public.ssg_is_active_admin(u.itcode) AS is_admin`。

EXPLAIN（pageSize=10 的页，取 20 行）：`Limit (actual rows=20) Execution Time: 9.445 ms`，SubPlan 1–4 各 `loops=20` → **20 行 × 4 = 80 次子查询 + 20 次 `ssg_is_active_admin`**，而后者内部又是全表扫 285 行 → 实际约 20 × 285 ≈ 5700 行额外扫描。`RoleManagementDao:34-35`（`listRoles`）同款，2 个 count 子查询。

### P1｜PageHelper 的 count 把子查询重跑一遍

`PermissionManagementService.java:118` 用 `PageHelper.startPage(n, size, true)`，会拼 `select count(*) from (原SQL) t`，4 个 `string_agg` 子查询在 count 阶段被再算一遍（包在派生表里不一定能裁掉）。

对比之下 `PermissionAuditDao` 做对了：list 与 count 分开两条 SQL + 手写 LIMIT/OFFSET（`:50` 与 `:117`，注释写明「与 listAuditPage 完全相同的筛选条件，只换成 count」）。代价是**约 40 行 where 条件复制粘贴两遍**，任何筛选字段变更都要改两处 —— 这正是 XML `<sql id>` + `<include refid>` 的唯一存在理由：不是审美，是消除复制。

### P1｜审计 `previous_detail` 会随时间恶化

`PermissionAuditDao.java:63-72` 用「按 id 回查同 target 的上一条」子查询取 `previous_detail`。线上索引是 `idx_ssg_permission_audit_target (target_type, target_itcode)`：`coalesce()` 包裹列使复合索引后两列失效；`target_id` 不在索引里，必须回表过滤；审计表 append-only，**224 行现在无感，20 万行即全表扫**。

根因不在 SQL 而在建模：审计表只存 `detail`（「改成了什么」），没存 before/after，页面只能逐行回查。正解是给审计表加 `before_detail` / `after_detail` 两列，写入时一次落库 —— 纯加列、不塞数据，符合迁移规矩。

### P1｜`listEffectiveMenuStats` 是 O(用户 × 菜单) 内存推演

`PermissionEffectiveQueryService.java:233-291`：外层遍历全部 active 用户、内层遍历全部受管菜单，`resolveState` + HashMap 查找，再 `collectAccessibleMenuIds` 向上遍历祖先。`/permission/menus/effective-stats` 每次打开 Menu Permissions 页都跑，**无缓存**。

现在 285 用户 × 6 菜单 = 1710 次无感，但线性放大：菜单涨到 30 → 8550 次；用户 3000 → 1.8 万次。公平地说这段**不适合下推 SQL**（祖先提升 + 入口页规则用递归 CTE 能写但可读性会崩），放 Java 是对的；可优化点是加 60s 本地缓存（菜单/用户变更时失效），或把该接口从「页面加载必调」改为「点开统计才调」。

### P2｜结构性小毛病

| 项 | 现状 | 说明 |
|---|---|---|
| `@Mapper` 标注不一致 | 6 个 DAO 中 4 个标了、2 个没标（Permission/Role 没标） | `AdminApplication:10` 已有 `@MapperScan("com.lenovo.admin.*.dao")`，标注冗余，建议统一去掉靠扫描 |
| 重复 SELECT 列表 | `RoleManagementDao:24-48`（listRoles）与 `:50-65`（findRoleItem）逐字重复 8 列 | XML `<sql>` 直接解决 |
| 全删全插 | `saveUserMenus` = `deleteManagedUserMenus` + `batchUpsertUserMenus`（有 `@Transactional`，OK） | 6 个菜单下无问题；菜单多了会变成几百行 upsert |
| 审计只存文本 | `detail = menuStates.toString()`（`PermissionManagementService.java:285`） | 前端只能靠 previous_detail 反推，见上 |

### 值得继承的好实践（这块底子高于全局平均）

1. **100% DTO 化**：`permission/dto` 下 44 个类，`List<Map>` **0 处**（对比 chatbi 的 265 处裸 Map，未传染）。
2. **文本块写 SQL**：`@Select("""...""")` 而非字符串拼接，可读性接近 XML。
3. **写操作有悲观锁**：`lockUserByItcode` / `lockRole` 用 `FOR UPDATE`。
4. **批量 upsert 用 `ON CONFLICT DO UPDATE`**，批量改默认值用 `CASE WHEN` —— PG 正确写法，没有 for 循环单条写。
5. **审计独立成 DAO + append-only**，不混进业务 SQL。

因此试点不是「救火」，是把 70 分模块推到 90 分，作为全项目样板。

## 4. 试点方案（3 个 MR，按依赖顺序）

### MR 1｜DB 层：索引与函数（不改业务代码）

```sql
-- 1) itcode 全小写已验证（285/285），二选一，推荐 B、A 兜底
-- A. 表达式唯一索引（代码不动）
CREATE UNIQUE INDEX CONCURRENTLY ux_ssg_user_itcode_lower ON public.ssg_user (lower(itcode));
-- B. 改函数体，让 exists 走 index_itcode
CREATE OR REPLACE FUNCTION public.ssg_is_active_admin(p_itcode text) ...
  WHERE u.itcode = lower(p_itcode)   -- 原为 lower(u.itcode) = lower(p_itcode)

-- 2) 审计回查索引（配合去掉 coalesce）
CREATE INDEX CONCURRENTLY idx_audit_prev
  ON public.ssg_permission_audit_log (target_type, target_id, target_itcode, id DESC);
```

验收：EXPLAIN 从 Seq Scan 变 Index Scan；`/permission/users` Execution Time 目标 < 3ms（现 9.4ms）。

### MR 2｜SQL 形状：干掉 4 个子查询

`listUsers` 改 **2 条 SQL + Java 组装**（比 LEFT JOIN + `array_agg` 更稳，避免 `user_role × user_group` 行放大）：

```java
List<PermissionUserItem> rows = dao.listUsers(keyword, offset, limit);   // 只查 ssg_user
List<Integer> ids = rows.map(PermissionUserItem::getId);
Map<Integer, List<Integer>> roles  = dao.listRoleIdsByUserIds(ids);      // WHERE user_id = ANY(#{ids})
Map<Integer, List<String>>  groups = dao.listGroupNamesByUserIds(ids);
// Java 里塞回去
```

同时把 `is_admin` 从「逐行调函数」改为 LEFT JOIN 计算，或给 `ssg_user` 加冗余列 `is_super_admin` 由角色变更时维护。预期 20 行从 81 次子查询降到 3 条 SQL；`listRoles` 同款处理。

### MR 3｜组织方式：只把 `permission` 包 XML 化（试点核心）

```yaml
# application.yml（现在完全没有这行，写了 XML 也加载不进来）
mybatis-plus:
  mapper-locations: classpath*:/mapper/**/*.xml
  configuration:
    map-underscore-to-camel-case: true
```

搬迁顺序（按收益/风险比）：`PermissionAuditDao`（190 行 → ~40 行接口 + 1 XML，list/count 两份 where 合成一个 `<sql id="auditFilter">`）→ `PermissionManagementDao`（340 行）→ `RoleManagementDao` + `UserGroupManagementDao`（共享 `<sql>`）→ `EffectivePermissionQueryDao`（126 行，含递归 CTE，XML 里比文本块可读）。

**原则：纯搬迁，SQL 文本逐字搬，方法签名不变，Service 零改动** → 回归面收敛在 DAO 内部，前端零改动。

### 明确不做（防止试点变重构）

- 不动 `menu/`、`user/`、`role/` 老包（`${}`、`SELECT *`、`MenusDao.orderBy` 那些）—— 它们是 Admin Control 旧接口，与本次页签无调用关系。
- 不动 User Logs（chatbi）。虽然 XML 化收益最大（2104 行、47 select），但它是最近 commit 的活跃区（见 `fix(admin): 会话记录按本轮实际报表归集来源`、`fix(report): 评审模板导出跳过无会话行`），回归成本高，等 Access control 跑通再复制经验。
- 不在删 `@Mapper` 注解的同一个 commit 里改别的东西，避免 review 噪音。

## 5. 验收指标

| 指标 | 现状 | 目标 |
|---|---|---|
| `/permission/users` EXPLAIN | Seq Scan 或 4×20 SubPlan，9.4ms | 无 Seq Scan，< 3ms |
| `lower(itcode)` 查询计划 | Seq Scan 285 行 | Index Scan |
| listUsers 子查询数（20 行） | 81 | 3 条 SQL |
| 审计 list/count where 重复 | 2 份约 40 行 | 1 个 `<sql>` |
| permission 包 DAO 行数 | 1224 | ~500（接口）+ 4 个 XML |
| `List<Map>` | 0（已达标） | 保持 0 |
| 前端改动 | — | **0** |

## 6. 后续可选路径

1. 先出 MR1（索引/函数）+ EXPLAIN 前后对比 —— 最小、最快见效。
2. 或直接做 MR3 的 Audit DAO XML 化，用一个 DAO 验证 `<sql>/<include>` 能省多少。
3. 或把本报告归档成 `docs/admin-control-sql-audit.md` 再约时间动手。

---

参考：本仓已有 `admin 模块（admin control 页面）接口与包结构重组分析` 文档（id `3ebbfe47-3740-42a5-9292-e7808dfa2d60`），覆盖 89 个 dao / 686 条语句的注解 SQL 基线、方法级引用矩阵与 admincontrol.{userlogs,feedback,inspection,tabs} 迁移顺序；本文档补充其未覆盖的部分：Access control 子树的实测执行计划证据、审计建模缺陷、以及 MR1–MR3 的三层落地顺序。
