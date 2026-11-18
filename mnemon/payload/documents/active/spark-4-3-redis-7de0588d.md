---
id: "7de0588d-2d49-4153-b601-b8fffe61d472"
title: "生产 SPARK 菜单开通执行记录：4 人 × 3 菜单写入、生产 Redis 生效判据与审计缺口"
description: "ssg_ops_agent 生产环境为 4 个 itcode 写入 12 行 ssg_user_menu（VISIBLE × 3 含入口页 workspace）、p_renxp4 只读需 a_appconnect、两套 Redis 中只有 reston 10.62.79.178:6379/db=1 持有 token 的定位判据与 DEL 踢下线、直改库绕过 ssg_permission_audit_log 的补录缺口。"
status: "active"
created_at: "2026-11-18T02:48:49.318Z"
updated_at: "2026-11-18T02:48:49.318Z"
content_hash: "942cfcac1f053528c849033cf25a738c939423397b741468f1e00ad46bc0e033"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/dao/PermissionAuditDao.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/service/EffectivePermissionService.java"
session_ids:
  - "mnemon-run-v1-b19a5285-b46d-4a45-8afd-80bf1a65d9a4"
memory_body_ids:
  []
---

# 生产 SPARK 菜单开通执行记录：4 人 × 3 菜单 + Redis 踢下线定位 + 审计缺口

时间：2026-11-18。仓库 `C:\\Users\\63588\\lenovo\\ssg-sales-ops`。生产库 `10.62.79.215:5432/ssg_ops_agent`（PostgreSQL）。

关联文档（本文只记录**执行结果**与它们没覆盖的判据，不复述其结论）：
- `48d981bf-63a2-4530-bc45-8225e87d99a6` —— 生产 GPS&SUS 6 人核查与**待执行** UPSERT 方案、6 个 managed 菜单清单、`EffectivePermissionService.visibleMenuKeys` 生效规则
- `1d44b2d4-bc96-45f3-8138-c552448766ea` —— auth 侧 `MenuPermissionResolver` 的 HIDDEN / 缺行语义
- `48f9fd58-5593-4cd7-93c6-8d3a8cc76e4e` —— 管理员降权/恢复 + `DEL token_<itcode>` 踢下线 Runbook

## 一、最关键的新结论：生产只有 reston 那套 Redis 生效

admin 的 `lmp-sy-prod` 与 `lmp-reston-prod` **指向同一个 PG 库**（`10.62.79.215:5432/ssg_ops_agent`），但 **Redis 是两套、只有一套有登录态**：

| 来源 | sentinel | master | db | 有 token_* ? |
| --- | --- | --- | --- | --- |
| reston（production application 里的 `spring.redis.sentinelConf`） | `10.62.79.178:26379` / `10.62.79.174:26379` / `10.62.79.134:26379`，`masterNode: sentinel10.62.79.178` | `10.62.79.178:6379` | 1 | ✅ 有 |
| sy | `10.122.198.105:26379`（`master=sentinel10.122.198.105`） | `10.122.198.105:6379` | 11 | ❌ 从来没有 |

判据做法：对每个 itcode 各查一次 `EXISTS token_<itcode>` + `TTL`，命中哪套就踢哪套。本次实测 reston 上 `token_mmazzantini` TTL 63914s、`token_mmikula1` TTL 35247s；sy 上 4 个 key 全部 `exists=false / ttlSec=-2`。**不要靠 profile 名或库地址推断 Redis**，同一份用户数据下两套 Redis 的登录态完全不同。

## 二、写权限：p_renxp4 只读，必须 a_appconnect

- `SELECT current_user, has_table_privilege(...)` 结果：`p_renxp4 | can_select=t | can_insert=f | can_update=f | can_delete=f`
- 直写报错原文：`ERROR: permission denied for table ssg_user_menu`
- 本次按用户逐次确认后改用服务账号 `a_appconnect`（其连接串即 admin `lmp-reston-prod` 的 `spring.datasource`）执行；`p_renxp4` 依然可以读回来验证。
- 注意 `ssg_permission_audit_log` 对 `p_renxp4` 同样无 INSERT 权限，补录审计也必须走 `a_appconnect`。

## 三、实际写入（12 行，已完成）

4 人 × 3 菜单，全部 `access_state='VISIBLE'`、`enabled=true`、`operator='renxp4'`，`UPDATE_COUNT: 12`：

| itcode | businessInsight | :workspace | :gpssus |
| --- | --- | --- | --- |
| mmazzantini（Matt Mazzantini） | VISIBLE | VISIBLE | VISIBLE |
| mmikula1（Matt Mikula） | VISIBLE | VISIBLE | VISIBLE |
| bconner（Byron Conner） | VISIBLE | VISIBLE | VISIBLE |
| pgreen（Phil Cutland Green） | VISIBLE | VISIBLE | VISIBLE |

- `Bookings` / `LGFS` / `TCV` **按用户明确要求不开**，保持无行。在 admin 侧缺行即 HIDDEN，与后台权限页"未勾选"落成 HIDDEN 等价。
- `:workspace` 必须一并置 VISIBLE：它是 `is_entry=true` 的模块入口页，父菜单可见而入口页不可见时，点 SPARK 会被前端路由守卫挡到"无访问权限"页，表现为"有权限却进不去"。
- 复核 SQL（按 itcode 聚合可见菜单）：4 人均为 `businessInsight, businessInsight:workspace, businessInsight:gpssus`。

## 四、踢下线（已完成）

`DEL token_mmazzantini` → 1，`DEL token_mmikula1` → 1，事后 4 个 key 全部 `existsAfter=false`。`bconner`、`pgreen` 原本不在线，`DEL` 返回 0，无需处理。
生效方式：被踢者下一个请求 401 回登录页，重登后 `/user/get` 重算 `menuKeys`；另外两人下次登录直接生效。注意 `ProviderAuthRestInterceptor` 信任网关 `USERINFO` 头、不查 Redis，已加载完的页面未必立刻被拦住（见 `1d44b2d4-bc96...`）。

## 五、遗留缺口：直改库绕过了权限审计

- 表结构与索引：迁移 `V1.1.77__Create_super_admin_role_and_permission_audit.sql:89-112`（`ssg_permission_audit_log`：`operator, action, target_type, target_id, target_itcode, role_id, role_code, reason, detail, create_time`）
- 写入方：`smartop-service/admin/src/main/java/com/lenovo/admin/permission/dao/PermissionAuditDao.java`（`insert` + `listRecent` + 分页 `prev` 相关子查询）
- 走后台"权限管理"页保存会由服务端补审计；本次直接 SQL 没有记录。若审计页需要留痕，按 `action=GRANT`、`target_type=USER_MENUS` 补 4 条（用 `a_appconnect`，`p_renxp4` 无 INSERT 权限）。

## 六、附带工具坑

- 本地 `~/.m2` 里 jedis 只有 `4.3.2`（含 sources）与 `2.7.0/2.11.1` 等旧版，**没有 5.2.2 目录**；高版本 jedis 的 `DefaultJedisClientConfig.Builder` 不再有 `socketTimeout(int)`（编译报"找不到符号"），且 `CommandObjects` 静态块依赖 `com.google.gson.Gson`，否则运行期 `NoClassDefFoundError: com/google/gson/Gson`。
- 池化依赖：`commons-pool2` 本地只有 `2.11.1`。

## 操作顺序建议（复用时）

1. 先 `has_table_privilege` 确认当前账号可写，否则向用户逐次确认后切 `a_appconnect`
2. 写 `ssg_user_menu`（含入口页 `:workspace`），读回按 itcode 聚合复核
3. 对每个 itcode 在两套 Redis 上各查 `EXISTS token_<itcode>`，只 `DEL` 命中那套的 key，再复核
4. 需要审计留痕则补 `ssg_permission_audit_log`
