---
id: "1d44b2d4-bc96-45f3-8138-c552448766ea"
title: "SSG SPARK 登录后全部路由跳 /noAccess：auth 侧菜单权限语义排查（ssg_user_menu）"
description: "生产 SSG SPARK 用户 menuKeys 为空导致所有路由跳 /noAccess 的完整排查：前端取的是 auth 服务 permissions，auth-release 的 MenuPermissionResolver 把非 VISIBLE（含 HIDDEN）一律当 DISABLED 并向下整棵子树传播，与 admin 侧 EffectivePermissionService 语义相反；修复为 ssg_user_menu 三行改 VISIBLE + 删三行（缺行=隐藏），并记录 ProviderAuthRestInterceptor 信任 USERINFO 不查 Redis 导致踢下线不即时生效。"
status: "active"
created_at: "2026-11-13T17:01:13.848Z"
updated_at: "2026-11-13T17:01:13.848Z"
content_hash: "45162472fdcf883c0377bcfeb6d0a81479703457d6a100e8544f74ad7f5978f2"
source_paths:
  - "src/router/index.ts"
  - "src/api/request.ts"
  - "src/utils/cookie.ts"
  - "smartop-common/smartop-common-lmp/src/main/java/com/lenovo/common/lmp/provider/ProviderAuthRestInterceptor.java"
session_ids:
  - "mnemon-run-v1-28803c8c-3bcf-4e42-9470-3cbcb5eb7c95"
memory_body_ids:
  []
---

# SSG SPARK 登录后被踢到 /noAccess：auth 侧菜单权限语义排查

日期：2026-11-13/14。环境：生产 https://ssgspark.lenovo.com，用户 renxp4（user id=79）。

## 现象

用户登录后点任何带 menuKey 的路由都落到空白的 `/noAccess` 页；F12 看 `GET /api/auth/user/info` 返回 `menuKeys: []`、`disabledMenuKeys: []`，即"一条菜单都没有"。

前端落点：`src/router/index.ts:204-206` 定义 `/noAccess` 路由，`src/router/index.ts:305` 在守卫里 `next({ path: '/noAccess', replace: true })`。守卫用 `hasMenuPermission('workspace')` 之类判断，menuKeys 为空时永远匹配不上。

## 根因：两套服务、两套 HIDDEN 语义

前端菜单权限**不是** admin 服务算的，而是 **auth 服务**算的。生产 auth 服务跑的是 `origin/auth-release`，对应提交 `ca980de34 feat(auth): return disabled menu permissions`（11 文件 +471/-28），新引入：

- `com/lenovo/auth/user/util/MenuPermissionResolver.java`（187 行，纯函数式解析器）
- `com/lenovo/auth/user/dao/MenuPermissionDao.java`：`getActiveMenuNodes()` 用递归 CTE 取从启用根菜单可达的启用节点；`getUserMenuGrants(userId)` 查 `ssg_user_menu` 里 `enabled = TRUE` 的直授，**缺行即隐藏**
- POJO `MenuPermissionNode` / `MenuPermissionGrant` / `MenuPermissionResult`，`UserInfoVo` 与 `UserPassportImpl` 增加 `disabledMenuKeys`

auth 侧语义（关键，与 admin 侧完全不同）：

- `parseAccessState`：只有 `VISIBLE`（忽略大小写）算可见，**其余任何值（包括 `HIDDEN`）一律当 DISABLED**
- `hasDisabledSelfOrAncestor`：自身或任一祖先为 DISABLED → 该节点进 `disabledMenuKeys`
- `collectRelevantIds`：相关集合 = 所有授权节点的自身+祖先链，加上每个 DISABLED 授权的**整个子树**
- 排序 `MENU_ORDER`：sortOrder 空值最后，再按 menuKey

对比 admin 服务（`admin-release` 的 `EffectivePermissionService`）：父菜单即使 HIDDEN 也不影响子菜单可见性。**同一个 `HIDDEN` 在 admin 侧等于"只是不给你看"，在 auth 侧等于"整棵子树全部禁用"**。误按 admin 语义把父菜单设成 HIDDEN，结果 visibleNodes 为空 → `menuKeys: []`。这不是权限没配上，是语义搞错了。

## 修复（库 ssg_ops_agent，表 ssg_user_menu，user_id=79）

```
businessInsight            HIDDEN → VISIBLE
businessInsight:workspace  HIDDEN → VISIBLE
businessInsight:bookings   VISIBLE（不变）
businessInsight:lgfs / :gpssus / :tcv  → 删除行
```

删那三行是有意的：auth 侧 `HIDDEN` 会进 `disabledMenuKeys`，前端当"禁用"而非"没开"；**缺行才是真正的隐藏**（`collectRelevantIds` 从 grants 出发，没行就不会出现在结果里）。现在名下 3 行、全 VISIBLE。

验证：`/api/auth/user/info` 返回 `menuKeys: ["businessInsight","businessInsight:workspace","businessInsight:bookings"]`、`disabledMenuKeys: []`，刷新后 Business Insights → Bookings 可进，右侧只剩一个子菜单。

## 附带发现：Redis 里没 token 也能通过鉴权

用户那把 Authorization 在 Redis `token_renxp4` 已不存在（`ttl=-2 exists=false`，`DEL` 返回 0，故不是我们删的），但同样请求头 curl 仍 200。原因：`smartop-common/smartop-common-lmp/src/main/java/com/lenovo/common/lmp/provider/ProviderAuthRestInterceptor.java:77-82` 只要请求头带 `USERINFO` 就直接信任网关注入的结果，根本不查 Redis 里的 `checkToken`。

后果：**"踢下线"对已经加载完的页面未必即时生效**，真正拦住请求的是 token 过期或前端自己清 cookie。前端 `src/api/request.ts:88-107` 只在 `getToken() && status===401` 时 `removeUserInfo()` 并跳 `/`；`src/utils/cookie.ts:12-14` 的 `removeToken` 带 `path: '/', domain: '.' + hostname`，清 cookie 时必须对齐 domain/path，否则删不掉。

## 操作教训

1. 改菜单可见性前先确认**是哪套服务在算**（生产 auth=`origin/auth-release` 的 `MenuPermissionResolver`；admin 的 `EffectivePermissionService` 是另一套），同一份 `ssg_user_menu` 数据在两边语义不同。
2. 在 auth 侧表达"不想给看"要用**删行**，不要用 `HIDDEN`；用 `HIDDEN` 会连带整棵子树变禁用，还会出现在 disabledMenuKeys 里被前端标成"禁用"。
3. 排查时先打 `/api/auth/user/info` 看 menuKeys/disabledMenuKeys，比翻前端守卫代码快得多。
