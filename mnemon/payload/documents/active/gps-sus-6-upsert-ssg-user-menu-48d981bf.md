---
id: "48d981bf-63a2-4530-bc45-8225e87d99a6"
title: "生产 GPS&SUS 菜单权限开通：6 人核查结果与待执行 UPSERT 方案（ssg_user_menu）"
description: "生产 ssg_ops_agent 菜单权限现状（6 个 managed 菜单、EffectivePermissionService 生效规则、缺行即 HIDDEN）；6 个 itcode 的 ssg_user_menu 核查结果（3 人已有、3 人 0 行需补 18 行）；p_renxp4 只读导致需 a_appconnect 的写权限卡点；18 行 UPSERT 与回滚 SQL、HIDDEN 口径说明、SQL 直写不触发 refreshToken 的生效条件。"
status: "active"
created_at: "2026-11-17T03:06:04.633Z"
updated_at: "2026-11-17T03:06:04.633Z"
content_hash: "a56fb52a77b7c7135991449fbcc9527edc20a7e58cfafd6b3e4e5c0f336fb4d4"
source_paths:
  - "src/stores/user.ts"
  - "src/services/api.ts"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/permission/service/EffectivePermissionService.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/controller/GpsSusController.java"
session_ids:
  - "mnemon-run-v1-6feba903-8a1a-457a-bf53-bac3010d4809"
memory_body_ids:
  []
---

# 生产 GPS&SUS 菜单开通：6 人核查结果与待执行 UPSERT 方案（ssg_user_menu）

时间：2026-11-17。仓库 `C:\Users\63588\lenovo\ssg-sales-ops`。生产库 `10.62.79.215:5432/ssg_ops_agent`（PostgreSQL）。

关联文档（本问不重复其结论）：
- `1d44b2d4-bc96-45f3-8138-c552448766ea` —— auth 侧 `MenuPermissionResolver` 的 HIDDEN / 缺行语义
- `4a8215fb-68f5-4aa8-a53e-14b230c96d67` —— menu_key 单一真源改造、测试环境 renxp4 权限对齐

## 一、生产菜单清单（共 6 个，全部 managed=true）

| menu_key | 名称 | 路径 | is_entry |
| --- | --- | --- | --- |
| businessInsight | Business Insights | /spark | false |
| businessInsight:workspace | My Workspace | /spark/workspace | **true**（入口页） |
| businessInsight:bookings | Bookings | /spark/bookings | false |
| businessInsight:lgfs | LGFS | /spark/lgfs | false |
| businessInsight:gpssus | GPS & SUS | /spark/gpssus | false |
| businessInsight:tcv | TCV | /spark/tcv | false |

admin 侧生效规则（`EffectivePermissionService.visibleMenuKeys`，`smartop-service/admin/src/main/java/com/lenovo/admin/permission/service/EffectivePermissionService.java:37-61`）：非管理员只认自己那条 `ssg_user_menu.access_state='VISIBLE'` 且 `enabled=true` 的记录，**缺行即 HIDDEN**；可见子菜单自动提升父菜单，父菜单可见时把入口页 workspace 带出。前端 `hasMenuPermission` 直接比对后端下发的 `menuKeys`（`src/stores/user.ts:68-72`）。

## 二、核查结果：6 人里只需开通 3 人

6 个 itcode 均存在于 `ssg_user`，邮箱前缀与 display_name 对得上。

| itcode | ssg_user_menu 现有行 | 能否进 /spark/gpssus |
| --- | --- | --- |
| kwierzbicki | 6 行（含 businessInsight:gpssus=VISIBLE） | 可以 |
| kmurphy2 | 6 行 | 可以 |
| mkohut | 6 行 | 可以 |
| mliptakova（user_id 353） | 0 行 | 不行 |
| jjakubcova（user_id 354） | 0 行 | 不行 |
| pmacdonald（user_id 185） | 0 行 | 不行 |

后 3 人目前连 businessInsight 父菜单和 My Workspace 入口都看不到。合计需写 18 行（3 人 × 6 菜单）。前 3 人的 6 行是全量行，bookings/lgfs/tcv 也是 VISIBLE。

## 三、写权限卡点（未解决，等待批准）

`p_renxp4` 的 `has_table_privilege` 结果全为 false（只有 SELECT）：`ssg_user_menu` INSERT/UPDATE、`ssg_menu` INSERT、`ssg_user` INSERT、`ssg_permission_audit_log` INSERT。
实测写库报错原文：`psycopg.errors.InsufficientPrivilege: permission denied for table ssg_user_menu`。
团队规则要求服务账号逐次申请，故必须先取得用户明确同意，再改用 `a_appconnect` 执行。

## 四、待执行 SQL（18 行）

```sql
INSERT INTO public.ssg_user_menu (user_id, menu_id, enabled, access_state, operator, create_time, update_time)
SELECT u.id, m.id, true, s.state, 'renxp4', now(), now()
FROM public.ssg_user u
JOIN (VALUES
    ('businessInsight','VISIBLE'), ('businessInsight:workspace','VISIBLE'),
    ('businessInsight:bookings','HIDDEN'), ('businessInsight:lgfs','HIDDEN'),
    ('businessInsight:gpssus','VISIBLE'), ('businessInsight:tcv','HIDDEN')
) AS s(menu_key,state) ON true
JOIN public.ssg_menu m ON m.menu_key = s.menu_key
WHERE lower(u.itcode) IN ('jjakubcova','mliptakova','pmacdonald')
ON CONFLICT (user_id, menu_id) DO UPDATE
SET access_state = EXCLUDED.access_state, enabled = true, update_time = now(), operator = EXCLUDED.operator;
```

另需补一条 `ssg_permission_audit_log` 审计记录。

口径说明：父菜单 VISIBLE + workspace VISIBLE + gpssus VISIBLE，其余三个显式 HIDDEN，与管理员在 Access 页点保存后的落库结果一致。bookings/lgfs/tcv 都是叶子节点（无子树），写 HIDDEN 不会像父菜单 HIDDEN 那样把整棵子树禁用（auth 侧传播语义见 `1d44b2d4`），但会进 `disabledMenuKeys` 被前端标成「禁用」。若要求 6 个页面全开，把这三项改成 VISIBLE。

## 五、生效与回滚

- 菜单权限随 token 缓存。Access 页保存时后端会自动调 `refreshToken`；SQL 直写不会触发，因此这 3 人需重新登录，或另调一次 `/auth/refreshToken`（`user/controller/UserController.java:278-279`，`@GetMapping(value = "refreshToken")`，入参 `itcodes: List<String>`）。
- 回滚（删掉这 18 行即回到 0 行 = 全 HIDDEN 原状）：

```sql
DELETE FROM public.ssg_user_menu
WHERE user_id IN (353,354,185)
  AND menu_id IN (1,2,5,4,3,6);
```

## 六、两条可选路径

1. 申请 `a_appconnect` 执行上面 SQL（需自行补审计行），执行后拉 6 人的最终权限矩阵核对。
2. 由用户在 Admin Control → Access 页手工给这 3 人勾 GPS & SUS：后端自动补审计并 refreshToken，审计里体现为一次正常管理操作。

## 关键文件

- `smartop-service/admin/src/main/java/com/lenovo/admin/permission/service/EffectivePermissionService.java:37-61` —— visibleMenuKeys 生效规则
- `smartop-service/admin/src/main/java/com/lenovo/admin/gpssus/controller/GpsSusController.java` —— `@RequestMapping("/gpssus")`，GPS&SUS 接口面
- `src/stores/user.ts:68-72` —— hasMenuPermission，直比 menuKeys
- `src/services/api.ts:247-257` —— getUserInfo → `/auth/user/info`，返回 `menuKeys: string[]`
- `user/controller/UserController.java:189-194`（info，返回 UserInfoVo）、`:278-279`（refreshToken）
- `user/service/impl/UserServiceImpl.java:120,126` → `user/dao/UserDao.java:150`（`List<Menu> getUserAllMenu(@Param("itcode") String itcode)`）
