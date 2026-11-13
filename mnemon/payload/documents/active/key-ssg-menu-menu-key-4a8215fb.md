---
id: "4a8215fb-68f5-4aa8-a53e-14b230c96d67"
title: "菜单权限 key 统一为后端原值（ssg_menu.menu_key 单一真源）"
description: "ssg-sales-ops 前端 hasMenuPermission 双拼 businessInsight: 前缀导致反馈页 NO ACCESS 的根因，以及把路由守卫/TopNav 全部改用后端 ssg_menu.menu_key 原值的三文件改法与 MR（!1019 13e47bfb master / !1020 761cbe80 develop）；附测试环境 renxp4 权限对齐生产的执行记录。"
status: "active"
created_at: "2026-11-13T17:45:39.896Z"
updated_at: "2026-11-13T17:45:39.896Z"
content_hash: "234c21590eb17eeaf6297c5fdbafdd8abf4e1c0d885a40387e7eb44a69bcd5ca"
source_paths:
  - "src/stores/user.ts"
  - "src/router/index.ts"
  - "src/components/TopNav/index.vue"
  - ".tmp/apply-tst-perm.sql"
session_ids:
  - "mnemon-run-v1-47f58ef5-e59a-45cf-9b6a-0680cded7c03"
memory_body_ids:
  []
---

# 菜单权限 key 统一为后端原值（ssg_menu.menu_key 单一真源）

日期：2026-11-14。仓库：C:\Users\63588\lenovo\ssg-sales-ops（Vue3 + vue-tsc）。
关联文档：1d44b2d4-bc96-45f3-8138-c552448766ea（auth 侧 MenuPermissionResolver 的 HIDDEN/缺行语义排查，本文不重复）。

## 问题：前端自己拼前缀，一处调用链拼出双前缀

`src/stores/user.ts` 的 `hasMenuPermission(menuKey)` 之前对**任何**入参都补 `businessInsight:` 前缀，再用带冒号的短 key 去比对：

- 路由守卫 `src/router/index.ts` 传的是短 key（`workspace`/`bookings`），被补成 `businessInsight:workspace`，正好匹配后端 `menuKeys`。
- 反馈页 Issue & Request 的 Affected page 走 `feedbackMenuCatalog.ts`，它本来就是完整 key `businessInsight:workspace`，补完变成 `businessInsight:businessInsight:workspace`，永远匹配不上。

表现：非管理员在 Issue & Request 页面看到 My Workspace / Bookings / GPS & SUS / LGFS 全是 NO ACCESS（P&L / Pipeline / TCV 显示 SOON 是功能未上线，属正常）。

## 决策（用户要求）

「2 边的行为改成一致，都由后端控制 key，这样好修改」—— 即前端不再造 key：权限判定只认后端下发的 `ssg_menu.menu_key` 原样字符串。

## 改法（3 个文件）

| 位置 | 之前 | 现在 |
|---|---|---|
| `src/stores/user.ts` | 短 key 补 `businessInsight:`，带冒号原样比 | `menuKeys.includes(menuKey)`，一个字不改 |
| `src/router/index.ts` | `meta.menuKey: 'workspace'` 等 7–9 处短 key | 全部改成完整 key，如 `businessInsight:workspace`、`businessInsight:gpssus`、`businessInsight:service-portfolio` |
| `src/components/TopNav/index.vue` | `hasMenuPermission(item.key)` 传短 key | 每个 navItem 新增 `menuKey` 字段（完整 key），`key` 只留作高亮、图标映射、`menuEmailConfig` 联系邮箱 |

TopNav 调用点最终形态：`const allowed = userStore.hasMenuPermission(targetItem.menuKey ?? targetItem.key)`。
反馈页 `feedbackMenuCatalog.ts` 无需改动 —— 它一直是完整 key。

## 为什么后端不用动

auth 侧 `MenuPermissionResolver` 本来就是把 `ssg_menu.menu_key` 原样塞进 `menuKeys` / `disabledMenuKeys`。bug 纯在前端拼接层。

**以后新增菜单的流程**：库里配好 `menu_key` → 前端 router meta 与 TopNav navItem `menuKey` 照抄同一字符串，`hasMenuPermission` 那行不用碰。

## 验证与交付

- 两条分支均 `npm run type-check`（vue-tsc --build）通过；改后 checks：router 侧 7 处 menuKey、navItems 13 项（含 development-only footprint）全部为完整 key。
- MR !1019（master）→ 提交 `13e47bfb`，can_be_merged；MR !1020（develop）→ 提交 `761cbe80`，can_be_merged。标题：fix(menu)+feat(reports): 修非管理员菜单权限误判，补问数评审筛选器。
- develop 先提交后 cherry-pick 到 master 时，`src/router/index.ts` 与 `src/components/TopNav/index.vue` 有冲突，按各分支现状手工解冲突（`patch-id` 与源提交不同，属正常）。
- **分支差异已知并保留**：master 的 `disabledNavItems` 仍含 `service-portfolio` 且没有 I&C 项，develop 有 I&C。未顺手抹平。

## 附：测试环境 renxp4 权限对齐生产（已执行）

环境：`10.62.79.148:5432/ssg_ops_agent`（测试）。执行账号 `a_appconnect` —— `p_renxp4` 在这三张表上全是 `f`（只读），没有写权限，故此步需用户显式点头。

SQL 保存在 `.tmp/apply-tst-perm.sql`，与生产同一套三步：
1. `UPDATE public.ssg_user SET is_admin=false, update_time=now() WHERE id=79`
2. `DELETE FROM public.ssg_user_role WHERE user_id=79 AND role_id IN (SELECT id FROM public.ssg_role WHERE role_code='SUPER_ADMIN')`
3. `DELETE FROM public.ssg_user_menu WHERE user_id=79` + 按 `menu_key IN ('businessInsight','businessInsight:workspace','businessInsight:bookings')` 插 3 行 `access_state='VISIBLE', enabled=true`

结果：`is_admin=f`（原 t）、SUPER_ADMIN 关联 0 行、`ssg_user_menu` 3 行 VISIBLE、`ssg_is_active_admin('renxp4')=f`、有效管理员 57 → 56。

两点环境差异：测试库菜单共 17 个（生产只有 6 个 `businessInsight*`），多出的 `dialogue`/`reportHub`/`workbench`/`toolkits`/`knowledge`/`efficiency`/`usageFeedback`/`admin` 因无 `ssg_user_menu` 行而隐藏，效果与生产一致；测试库 `ssg_user_role` **没有 `enabled` 列**（生产有），照抄生产的回读语句会报 `column "enabled" does not exist`。

改库后前端 token 里的 `menuKeys` 不会自动更新，需刷新页面或删除 `token_renxp4` 重登。
