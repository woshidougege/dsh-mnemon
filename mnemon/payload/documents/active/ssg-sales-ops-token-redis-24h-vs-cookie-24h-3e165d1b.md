---
id: "3e165d1b-e482-47fc-ae8a-3aa629f85665"
title: "SSG Sales OPS token 续期只做了一半：Redis 滑动 24h vs 浏览器 cookie 固定 24h（生产掉线根因）"
description: "生产用户“用着用着 token 到期”的根因：auth 服务 Redis token_<itcode> 每请求滑动续期 24h，但 AdfsController 登录回调写的 Authorization cookie Max-Age 固定 86400 且此后从不重写，浏览器到点删 cookie→401；含代码行号、PG 登录间隔 23-25h 尖峰证据、Redis TTL 核对结果、A/B/C 修复方向与 deleteToken 传参等顺带问题。"
status: "active"
created_at: "2026-11-18T08:09:39.987Z"
updated_at: "2026-11-18T08:09:39.987Z"
content_hash: "69030fd09fce5df12befdb29211a9b39e6790cb76c449587e82058bd87e22ef3"
source_paths:
  - ".agents/worktrees/auth-token-analyze/smartop-service/auth/src/main/java/com/lenovo/auth/adfs/controller/AdfsController.java"
  - ".agents/worktrees/auth-token-analyze/smartop-service/auth/src/main/java/com/lenovo/auth/adfs/constants/AdfsConstants.java"
  - ".agents/worktrees/auth-token-analyze/smartop-common/smartop-common-redis/src/main/java/com/lenovo/common/redis/token/impl/TokenServiceRedisImpl.java"
  - ".agents/worktrees/auth-token-analyze/smartop-common/smartop-common-lmp/src/main/java/com/lenovo/common/lmp/provider/ProviderAuthRestInterceptor.java"
  - ".agents/worktrees/auth-token-analyze/smartop-service/auth/src/main/java/com/lenovo/auth/user/service/impl/UserPassportImpl.java"
  - "src/services/request.ts"
  - "src/utils/cookie.ts"
session_ids:
  - "mnemon-run-v1-deb42488-b9d4-4e75-95c8-a0650388e14d"
memory_body_ids:
  []
---

# SSG Sales OPS：token 续期“只做了一半”——服务端 Redis 滑动、浏览器 cookie 固定 24h

用户提问（2026-11-18）：做了 token 续期，为什么生产用户还是“用着用着 token 到期”？要求去 worktree 看 auth 模块（auth-release 分支）代码并查生产数据。

## 结论

续期只做了一半：

- **服务端做了**：Redis `token_<itcode>` 每次校验通过后 `expire(..., 24, HOURS)`，是滑动 24h。
- **客户端没做**：ADFS 回调只在登录那一刻 `Set-Cookie: Authorization`，`Max-Age=86400` 从此不再刷新；cookie 到点被浏览器删除 → 请求不带凭据 → 401 → 前端清登录态。

即 cookie 生命周期是“固定 24h”，Redis 是“空闲超时 24h”，两者从第一天起就脱钩，所以“一直在用”也会在登录后第 24 小时准点掉线。

## 代码链路（worktree `.agents/worktrees/auth-token-analyze`，分支 auth-release）

做了的那一半（commit `4549e9ab2 feat(token): 登录 token 有效期改为 24 小时并支持滑动续期`，3 files +58/-4）：

- `smartop-common/smartop-common-redis/src/main/java/com/lenovo/common/redis/token/impl/TokenServiceRedisImpl.java#L92-L110`：校验通过后 `stringRedisTemplate.expire(tokenKey, 24, HOURS)`
- `smartop-common/smartop-common-lmp/src/main/java/com/lenovo/common/lmp/provider/ProviderAuthRestInterceptor.java#L93-L95`：`checkToken` 通过后调 `renewToken`，覆盖 `/**`（admin、auth 两服务都 `@CiopApplication` 生效）

漏掉的那一半：

- `smartop-service/auth/src/main/java/com/lenovo/auth/adfs/controller/AdfsController.java#L179-L186`：`ResponseCookie.from("Authorization", token).secure(true).path("/").domain(domain).sameSite("None").maxAge(86400)`，只在登录回调写一次，之后任何响应都不再 `Set-Cookie`
- `smartop-service/auth/src/main/java/com/lenovo/auth/adfs/constants/AdfsConstants.java#L14-L15`：`COOKIE_MAX_AGE_SECONDS = 86400L`，注释写着“与 Redis token 默认有效期保持一致”——但 Redis 侧会顺延，cookie 不会
- 同款写 cookie 逻辑在 `AdfsMobileBiController.java#L177-L195` 重复一份
- 密码登录路径更短：前端 `src/utils/cookie.ts#L7` 的 `Cookies.set(Token, token)` 无 expires，是 **session cookie**，关浏览器即失效（生产 30 天仅 10 次密码登录，影响面小）

## 生产数据证据（只读，PG `ssg_ops_agent` / Redis 侧人为只读）

Redis（生产 sentinel `10.62.79.178`，db1）：

| 项 | 值 | 说明 |
|---|---|---|
| `token_*` key | 49 个，TTL 为 -1/-2 的 0 个 | 无 key 泄漏为永不过期 |
| TTL + idle | 逐个 ≈ 86400 | **服务端滑动续期确实在工作** |
| evicted_keys | 0（maxmemory 7.69G，used 496M） | 排除内存淘汰 |
| login_degraded_users / 事件 | 0 / 0 | 排除“LDAP 降级 1h 会话”这条可能原因 |

PG `ssg_chatbi_event` 登录审计（30 天）：

| 指标 | 值 |
|---|---|
| 登录方式 | adfs 1168 次 / password 10 次（同一人测试）→ 生产≈100% ADFS |
| 日登录量 | 24~56 次 / 20~50 人 ≈ **1.1~1.4 次/人/天** |
| 相邻登录间隔分布 | <5min 41 / 5-60min 35 / 1-3h 21 / 3-8h 30 / 8-23h 81 / **23-25h 72** / 25-48h 119 / >48h 183 |

`23-25h` 这个尖峰是“固定 24 小时”的指纹：若客户端也滑动，第二天照常使用的用户不会恰好落在 24h 整点重登。登录高峰在 9-11 点，掉线高峰自然出现在次日同一时段——用户体感就是“我正在干活，怎么就过期了”。

## 用户为什么没看到“请重新登录”

`src/services/request.ts#L89-L97` 只在 `getToken()` 非空且 401 时才清状态跳登录；cookie 已被浏览器删掉时 `getToken()` 为空，于是既不跳登录也不提示，用户只看到裸的 `Request failed with status code 401`，刷新后才被路由守卫带到登录页，ADFS 自动重登（所以表现为“偶尔抽风”而非“彻底下线”）。

## 顺带发现（都会放大现象，但不是主因）

1. **`deleteToken` 传参错了**：`UserPassportImpl.java#L167` 与 `#L502` 传的是 token 值，而 `deleteToken(String username)` 拼的是 `token_<itcode>` → 实际删 `token_<加密串>` 这个不存在的 key。目前是无害死代码，但语义错误。
2. **白名单与 feign 请求不续期**：`security.ignore.whites` 里的 `/menu/all`、`/output/*`、`/api/v1/chat/completions` 直接放行不续期；服务间 feign 带 `USERINFO` 头直接 `return true`（对前端影响小，但“踢下线不即时生效”同源）。
3. **降级会话更短**：LDAP 不可用时 token 只有 1 小时且明确不续期（`UserPassportImpl.java#L106`）。本次生产 0 次降级，非当前原因。
4. **单 key 多端共享**：`token_<itcode>` 一用户一条，ADFS 重登复用旧 token 只重写 TTL；Admin 里编辑用户/批量改角色触发 `userService.refreshToken(...)`（`UserController.java#L122/#L239/#L254`）也只是重置 TTL 不换 token。

## 修复方向（待用户决定，未实施）

- **A（推荐）**：拦截器校验通过后顺带刷新 cookie（`Set-Cookie: Authorization=...; Max-Age=86400; Secure; SameSite=None`），让客户端凭证跟服务端一起滑动；可按剩余 TTL 低于阈值才写，避免每请求都发 Set-Cookie。
- **B**：cookie 有效期直接拉长到 7 天，Redis 仍 24h 滑动——改动最小，代价是“登出”必须依赖服务端删 token 才彻底。
- **C**：前端每个成功响应后 `Cookies.set('Authorization', token, { expires: 1 })`。
- 配套小修：前端 401 分支在 `getToken()` 为空时也跳登录；修正 `deleteToken` 传参。
- 落地路径：auth 服务改动走 **auth-release** 分支，前端走 master/develop 双 MR。

## 教训

1. 登录态是双份状态（服务端 Redis TTL + 浏览器 cookie 过期时间），只改一边等于没改；两者的一致性必须显式维护。
2. 判断“续期是否生效”要看**客户端凭证的 Max-Age 是否被重写**，而不是只看 Redis TTL。
3. 登录审计日志的“相邻登录间隔直方图”是定位此类问题的低成本杀手锏：在 TTL 整点处出现尖峰 = 固定过期而非滑动。

相关文档：SSG SPARK 登录后全部路由跳 /noAccess（`1d44b2d4-bc96-45f3-8138-c552448766ea`，其中记录 ProviderAuthRestInterceptor 信任 USERINFO、不查 Redis）；管理员降权/踢下线 Runbook（`48f9fd58-5593-4cd7-93c6-8d3a8cc76e4e`，其中记录 `token_{itcode}` hash 字段与 Redis 位置）。
