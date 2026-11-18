---
id: "3ea6d59b-4306-4a90-a819-3cd162fc0732"
title: "SSG Sales OPS token 续期修复：checkAndRenew + Set-Cookie 回写与四个 MR 交付"
description: "承接诊断文档 3e165d1b 的实现交接：checkAndRenew Lua 脚本（<23h 才续期、每人每小时最多写一次）、ProviderAuthRestInterceptor 在真续期请求回写 25h Set-Cookie、前端 401 一律清登录态并双删 cookie；含目标行为矩阵、dev/renxp4 与 develop 分叉（749/43）的分支策略与 why、MR !579 !580 !1092 !1093、验证与遗留 UserPassportImpl deleteToken 传参 bug。"
status: "active"
created_at: "2026-11-18T08:33:15.479Z"
updated_at: "2026-11-18T08:33:15.479Z"
content_hash: "e9ea0bee83474f9e9b4bb9c56b72ca50a8f7151e4558270906b927b0891bcd77"
source_paths:
  - "src/services/request.ts"
  - "src/utils/cookie.ts"
  - "smartop-common/smartop-common-redis/src/main/java/com/lenovo/common/redis/token/TokenService.java"
  - "smartop-common/smartop-common-redis/src/main/java/com/lenovo/common/redis/token/impl/TokenServiceRedisImpl.java"
  - "smartop-common/smartop-common-lmp/src/main/java/com/lenovo/common/lmp/provider/ProviderAuthRestInterceptor.java"
  - "smartop-service/auth/src/main/java/com/lenovo/auth/adfs/controller/AdfsController.java"
  - "smartop-service/auth/src/main/java/com/lenovo/auth/adfs/constants/AdfsConstants.java"
session_ids:
  - "mnemon-run-v1-934f37df-5cbe-4692-8981-8f64df87e41b"
memory_body_ids:
  []
---

# SSG Sales OPS token 续期修复：实现交接与四个 MR（2026-11-18）

**关联诊断文档**：`3e165d1b-e482-47fc-ae8a-3aa629f85665`（记录了根因：Redis 滑动 24h vs 浏览器 cookie 固定 24h、生产 PG/Redis 证据）。本文档记录该诊断之后的**实现与交付**：改了什么、为什么这么改、MR 落在哪些分支、怎么验收。

## 1. 结论一句话

服务端 Redis `token_<itcode>` 每请求续满 24h 一直有效；问题在浏览器 `Authorization` cookie 只在 ADFS 回调登录时写一次 `maxAge(86400)` 此后永不刷新（密码登录路径写的还是 session cookie）。修法是把「续期」这件事做成端到端：服务端条件续期 + 在真续期的那次响应回写 `Set-Cookie`。

**目标行为（用户确认验收口径）**：
- 一直有操作 → 永不掉线
- 停 23 小时内回来 → 无感继续
- 停满 24 小时 → 正常掉线重登

## 2. 三处改动

### 后端（共享模块，auth / admin 两个服务各一份，patch 内容同一份）

1. `TokenService` / `TokenServiceRedisImpl`：新增 `checkAndRenew`。用**一段 Lua 脚本**在一次 Redis 往返里完成「token 比对 → 降级判断 → TTL 判断 → 条件 `EXPIRE`」。
   - 阈值：剩余有效期**低于 23h 才续期** → 单个用户**最多每小时写一次 Redis**。
   - 收益：原来是每个请求 4 次往返 + 1 次写，现在正常情况下零写。
2. `ProviderAuthRestInterceptor`：改用 `checkAndRenew`，并且只在「本次真的续期了」的那次请求上回写 `Set-Cookie`。
   - cookie 参数（maxAge **25h**、`Domain`、`SameSite`、`Path`）与登录时写入的参数保持一致，避免客户端凭证与服务端 TTL 再次脱钩。
   - `Secure` 与登录侧 `AdfsConstants.SAME_SITE = ";SameSite=None; Secure=true"` 对齐。

### 前端（两个文件，2 files changed +8/−2）

- `src/services/request.ts`（diff 起点 L86）：401 判断去掉 `getToken() &&` 前提 —— **只要后端明确返回 401 就清登录态并跳登录**。原逻辑在 cookie 已被浏览器删掉时（本地无 token）只弹一句裸 `Request failed with status code 401`，把用户留在原页面反复失败，体感像「用着用着就掉线」。
- `src/utils/cookie.ts`（L12-24）：`removeToken` / `removeUsername` 同时删**两种写法**的 cookie —— 带 `{ path:'/', domain:'.'+hostname }` 的 Domain cookie（ADFS 回调下发）和 `{ path:'/' }` 的 host-only cookie。只删一种会留下另一个，导致 401 之后清不干净、刷新后继续 401。
- `src/utils/cookie.ts#L25-L38` 的 `removeUserInfo` 已有行为：登出时顺带清 localStorage 里所有 `ulr:jira-fields:` 前缀的 Jira 字段显示偏好（try/catch 忽略隐私模式异常）。

### cookie domain 配置：不用改任何 yml

auth 复用 `adfs.ciop.domain`，admin 复用 `app.env.domain`，本地 `localhost` 自动退回 host-only。

## 3. 分支与 MR 策略（含 why）

### 前端为什么不走 `dev/renxp4`

`dev/renxp4` 上还压着两个未合提交（`37cf7d25 style(feedback): 评审模板导入页重排视觉`、`eec619fe feat(feedback): 评审模板导入改为整页工作区`），且已开 `!1072`。按 AGENTS.md「每人一条 dev/<itcode>、每轮 reset 到最新 master」会把那个 MR 的内容冲掉。因此从 `origin/master` 另拉 `fix/token-slide-renew-fe`，只提这一笔，不与其他功能混。

### develop 与 master 已严重分叉

- develop 独有 **749** 个提交，master 独有 **43** 个；`master_is_ancestor=False`。
- 整分支 merge-tree vs develop 冲突 4 处：`src/constants/feedbackMenuCatalog.ts`、`src/views/Bookings/components/miniTable.vue`、`src/views/Setting/reports/user-logs/api.ts`、`src/views/Setting/reports/user-logs/index.vue`。
- 但本次改的 `src/services/request.ts`、`src/utils/cookie.ts` 两边内容**完全一致**，`package.json` 也无差异 → 从 `origin/develop` 拉临时分支 `tmp/token-slide-develop` cherry-pick 这一笔（干净通过），单独对 develop 提 MR。
- **遗留动作：合完之后 `tmp/token-slide-develop` 需要删除。**

### 四个 MR（均 opened / can_be_merged / conflicts=false，均未点合并）

| MR | 分支 → 目标 | 内容 | sha |
|---|---|---|---|
| !579 | `fix/token-slide-renew-auth` → `auth-release` | 3 files changed | `6ff4fcc43` |
| !580 | `fix/token-slide-renew-admin` → `admin-release` | 3 files changed | `ee2b7d0b0` |
| !1092 | `fix/token-slide-renew-fe` → `master` | 2 files +8/−2 | `c282d903` |
| !1093 | `tmp/token-slide-develop` → `develop` | 2 files（cherry-pick） | `eb2fc8ae` |

- 后端仓库 `dataagent/ssg-sales-ops`，前端仓库 `dataagent/ssg_sales_ops`（注意下划线）。
- 后端两个 MR 的 patch 已逐行核对为同一份：`TokenService.java` 26 行差异 0、`ProviderAuthRestInterceptor.java` 118 行差异 0、`TokenServiceRedisImpl.java` auth 52 / admin 53 行，唯一差异是 admin 多一行 `import java.util.Collections;`。
- commit message：后端 `fix(token): 续期时同步刷新登录 cookie，避免活跃用户 24 小时后掉线`（3 files, +197/−19）；前端 `fix(auth): 401 一律清理登录态，并同时删除两种写法的 token cookie`。

## 4. 验证

- 后端：auth worktree 与 admin worktree 各 `BUILD SUCCESS`（exit 0）。
- 前端：master 分支与 develop cherry-pick 分支各跑 `npm run type-check`（`vue-tsc --build`）exit 0。
- 部署后 TST 验收：登录 → 等 1 小时以上再操作 → DevTools 里看 `Authorization` cookie 的 Expires 是否被推后，即响应头出现 `Set-Cookie: Authorization=...`。最省事的验证是把工作台挂一整天，第二天同一时段仍在线即通过。

## 5. 遗留与已知问题

- **未修（既有 bug）**：`UserPassportImpl` 踢人时把 token 值当 itcode 传给 `deleteToken`，删的是一个不存在的 key —— 与本次无关，未改动，可单独一轮修。
- **GitLab REST 工具坑**：用 `dataagent%2Fssg-sales-ops` 这种 `%2F` 编码项目路径查 MR 会返回 `{"message": "404 Project Not Found"}`；创建 MR 的 POST 通道正常。查 MR 状态需换可用的调用方式。
- worktree 位置：`C:\\Users\\63588\\lenovo\\ssg-sales-ops\\.agents\\worktrees\\auth-token-slide`（auth）、`.agents\\worktrees\\admin-token-slide`（admin）。

## 6. 相关文件

- `src/services/request.ts`（401 处理，L86 起）
- `src/utils/cookie.ts`
- `smartop-common/smartop-common-redis/src/main/java/com/lenovo/common/redis/token/TokenService.java`
- `smartop-common/smartop-common-redis/src/main/java/com/lenovo/common/redis/token/impl/TokenServiceRedisImpl.java`
- `smartop-common/smartop-common-lmp/src/main/java/com/lenovo/common/lmp/provider/ProviderAuthRestInterceptor.java`
- `smartop-service/auth/src/main/java/com/lenovo/auth/adfs/controller/AdfsController.java`（L173-192 写 cookie）
- `smartop-service/auth/src/main/java/com/lenovo/auth/adfs/constants/AdfsConstants.java`（`COOKIE_MAX_AGE_SECONDS = 86400L`）
