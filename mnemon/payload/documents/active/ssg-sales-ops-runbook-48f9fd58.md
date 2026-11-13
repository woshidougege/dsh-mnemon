---
id: "48f9fd58-5593-4cd7-93c6-8d3a8cc76e4e"
title: "ssg_sales_ops：管理员降权/恢复与登录态踢下线 Runbook（测试 & 生产）"
description: "ssg_ops_agent 上把指定 itcode 降权为普通用户/恢复管理员的 SQL 模板与验证判据；测试库需 a_appconnect（p_renxp4 无写权限）且需用户同意；Redis token_{itcode} 结构、测试/生产 Redis 位置与 DEL 踢下线步骤；含 ssg_user_role 无 enabled 列、jedis 3.1.0 编译等坑。"
status: "active"
created_at: "2026-11-13T17:57:57.239Z"
updated_at: "2026-11-13T17:57:57.239Z"
content_hash: "a435e0974ae83c5c3a6d45415dd7b1ea20a88a14f136fc3edb9b34344a96370c"
source_paths:
  - "src/components/TopNav/index.vue"
  - "src/router/index.ts"
  - "src/stores/user.ts"
  - "smartop-service/auth/src/main/resources/application-lmp-reston-tst.yml"
  - "smartop-service/admin/src/main/resources/application-lmp-reston-tst.yml"
session_ids:
  - "mnemon-run-v1-34ad7d0c-52d2-4f2a-b212-85ed82966fa8"
memory_body_ids:
  []
---

# ssg_sales_ops：管理员降权/恢复与登录态踢下线 Runbook（测试 & 生产）

**用途**：把某个 itcode 临时降为普通用户（验证非管理员菜单形态）或恢复管理员，并让新权限在浏览器里立刻生效。本文记录的是 2026-11-14 在 `ssg_ops_agent` 上实测过的完整链路，含环境差异与踩过的坑。

相关文档：`1d44b2d4-bc96-45f3-8138-c552448766ea`（SSG SPARK 登录后跳 /noAccess 的菜单权限语义排查，解释了 `ssg_user_menu` 在 auth 侧"只有 VISIBLE 算可见、缺行=隐藏"的语义）。

## 1. 权限由三处共同决定，改一处就会不一致

| 位置 | 含义 |
| --- | --- |
| `public.ssg_user.is_admin` | 管理员标记，管理员走 isAdmin 短路 |
| `public.ssg_user_role` | 是否存在 `SUPER_ADMIN`（实测 `role_id = 1`）关联行 |
| `public.ssg_user_menu` | `access_state = 'VISIBLE'` 的菜单行；auth 侧语义下非 VISIBLE 一律当禁用，**缺行才是隐藏** |

判定函数：`SELECT public.ssg_is_active_admin('<itcode>')`。有效管理员总数可用于交叉验证，实测变化：测试库 57 → 降权后 56；生产 55 → 降权后 54 → 恢复后 55。

## 2. 环境与执行账号

- **测试库**：`10.62.79.148:5432`，库 `ssg_ops_agent`
  - `p_renxp4` 对 `ssg_user` / `ssg_user_menu` / `ssg_user_role` **三张表均无写权限**（权限查询结果全为 `f`）
  - 写操作只能用 `a_appconnect`；这属于跨账号提权，**执行前必须取得用户明确同意**（本次用户回的是"同意你 去执行"）
- **生产库**：`10.62.79.215:5432`，库 `ssg_ops_agent`
  - 降权/恢复同样用 `a_appconnect` 执行；`p_renxp4` 只用于只读回读
- 数据库/Redis 口令见各环境 `application-*.yml` 或 ENC 密文，**不落文档**。

## 3. 降权 SQL 模板（单事务，改完立刻回读）

```sql
-- 执行账号：a_appconnect（目标用户在 ssg_user / ssg_user_menu / ssg_user_role 上无写权限）
BEGIN;
UPDATE public.ssg_user SET is_admin = false, update_time = now() WHERE id = <userId>;
DELETE FROM public.ssg_user_role WHERE user_id = <userId>
  AND role_id IN (SELECT id FROM public.ssg_role WHERE role_code = 'SUPER_ADMIN');
DELETE FROM public.ssg_user_menu WHERE user_id = <userId>;
INSERT INTO public.ssg_user_menu (id, user_id, menu_id, enabled, create_time, update_time, operator, access_state)
SELECT nextval('public.ssg_user_menu_id_seq'), <userId>, m.id, true, now(), now(), '<itcode>', 'VISIBLE'
FROM public.ssg_menu m
WHERE m.menu_key IN ('businessInsight', 'businessInsight:workspace', 'businessInsight:bookings');
COMMIT;
```

保留的 3 个 menu_key 是"只看得见 Business Insights → Bookings"的最小集合；其它模块因为没有对应 `ssg_user_menu` 可见行而对目标用户隐藏。

## 4. 恢复管理员 SQL 模板

```sql
UPDATE public.ssg_user SET is_admin = true, update_time = now() WHERE id = <userId>;
INSERT INTO public.ssg_user_role (user_id, role_id, operator)
SELECT <userId>, r.id, '<itcode>' FROM public.ssg_role r
WHERE r.role_code = 'SUPER_ADMIN'
  AND NOT EXISTS (SELECT 1 FROM public.ssg_user_role ur WHERE ur.user_id = <userId> AND ur.role_id = r.id);
```

`ssg_user_menu` 的历史 VISIBLE 行**不删**：管理员走 isAdmin 短路，留着不影响，且以后复测非管理员形态可直接复用。

## 5. 验证回读

```sql
SELECT id, itcode, state, is_admin FROM public.ssg_user WHERE id = <userId>;
SELECT user_id, role_id, role_code, operator FROM public.ssg_user_role WHERE user_id = <userId>;
SELECT m.menu_key, um.access_state, um.enabled
FROM public.ssg_user_menu um JOIN public.ssg_menu m ON m.id = um.menu_id WHERE um.user_id = <userId>;
SELECT public.ssg_is_active_admin('<itcode>');
```

**坑**：测试库 `ssg_user_role` **没有 `enabled` 列**（生产有），回读语句里带 `enabled` 会报 `ERROR: column "enabled" does not exist`，与业务数据无关。

## 6. 踢下线（让新权限立刻生效）

登录态是 Redis hash，key 形如 `token_{itcode}`：

- 字段：`login_degraded` / `user` / `profile_incomplete` / `thumbnail_photo` / `token`
- TTL ≈ 86400s 滑动窗口（实测 86303 / 86339 / 86396）
- `user` 字段里缓存了 `isAdmin` 与 `menuKeys` —— **只改库不会更新它**，这是"改完权限但页面行为没变"的主因

Redis 位置：

- **测试**：`10.62.79.203:6379` db=1（`application-lmp-reston-tst.yml` 里 auth/admin 都是单机，sentinel 配置被注释）
- **生产**：sentinel master `sentinel10.62.79.178`，sentinels `10.62.79.178/174/134:26379`，db=1

操作与判据：

```
before: exists=true ttl=86396
DEL token_renxp4  → 1
after: exists=false ttl=-2
```

注意：`/auth/user/info` 是**实时查库**的，所以接口返回值会立刻变；但 token 里的缓存副本不会。**已知限制**：`ProviderAuthRestInterceptor` 只要请求头带 `USERINFO` 就直接信任、不查 Redis（见相关文档），所以一直开着没刷新的标签页可能还能撑一会儿，关掉重开才是干净状态。

## 7. 前端菜单 key 约定（本次已合入）

菜单 key 统一使用后端原值，前端不再拼前缀：

- `src/router/index.ts` 替换 7 处，`src/components/TopNav/index.vue` navItems 13 项，`src/stores/user.ts`
- 分支 `feature/admin-review-filters-master`，commit `13e47bfb`（`refactor(permission): 菜单 key 统一用后端原值，前端不再拼前缀`，3 files changed, +35/-10）
- MR：`!1019`（sha 13e47bfb）、`!1020`（sha 761cbe80），均 `merge=can_be_merged`，标题 `fix(menu)+feat(reports): 修非管理员菜单权限误判，补问数评审筛选器`
- `vue-tsc --build` 类型检查通过

前端 `menuKeys` 不会自己变，权限调整后需要刷新页面或重新登录。

## 8. 本地探测 Redis 的工具坑

- 本机 `.m2` 同时存在 jedis **3.1.0** 与 **4.3.2**；按 4.x 写代码在 3.1.0 上编译会失败：`ScanParams`/`ScanResult` 在 `redis.clients.jedis` 包下（没有 `params`/`resps` 子包），`Jedis` 也没有 `(HostAndPort, int)` 构造器
- Windows 下 `javac` 不要写 `-encoding=UTF-8`（会报 `java.lang.ClassNotFoundException: /encoding=UTF-8`），必须写成 `-encoding UTF-8`
- sentinel 场景用 `JedisSentinelPool(masterName, sentinels, cfg, timeout, password, db)`，db 直接传 1
