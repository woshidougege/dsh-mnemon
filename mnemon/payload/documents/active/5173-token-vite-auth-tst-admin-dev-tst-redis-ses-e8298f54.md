---
id: "e8298f54-6834-4b0f-99aa-aeab603ea881"
title: "本地 5173 前端 token 为空排查：vite 代理分工（auth→tst / admin→本地）、dev 与 tst 共用 Redis、session cookie 生命周期"
description: "5173 前端 token 为空的实测排查：vite 代理确实按「auth 走 tst、admin 走本地 8082」工作（含 3 条对照响应）；dev 与 tst 的 spring.redis 完全一致（database=1）故 tst token 本地可校验；前端 token 为无 expires 的 session cookie（src/utils/cookie.ts:6-7），无 token 不设 Authorization（src/services/request.ts:33-39），401 会 removeUserInfo 并刷新（:89-97）；登录页从 res.data.Authorization 取值（src/views/Login/index.vue:63-72）；待用户确认 token 为空的具体位置。"
status: "active"
created_at: "2026-11-20T09:50:22.935Z"
updated_at: "2026-11-20T09:50:22.935Z"
content_hash: "2d64eddcd6d3678a5ae4f67dcea8648f3c93326455231e78987812edd67f7de8"
source_paths:
  - "src/utils/cookie.ts"
  - "src/services/request.ts"
  - "src/views/Login/index.vue"
session_ids:
  - "mnemon-review-v1-cfff2bf4-886e-4261-90a9-26cbc62a55ed"
memory_body_ids:
  []
---

# 本地 5173 前端「token 为空」排查：vite 代理分工、dev/tst 共用 Redis 与 session cookie

2026-11-20 排查记录（结论部分待用户确认后收口）。工作目录 `C:\\Users\\63588\\lenovo\\ssg-sales-ops`。

## 1. 用户给定的环境约定

用户明确要求：**认证（auth）走测试环境，admin 服务走本地**。本次实测确认 vite 代理已按此分工，无需改动。

## 2. 代理分工实测（全部通过，问题不在链路）

运行时 `import.meta.env`：`MODE=development`、`VITE_API_BASE_URL=""`（空 → 前端走相对路径，由 vite 代理转发）、`VITE_API_REMOTE_TARGET=https://ssgspark-tst.lenovo.com`、`VITE_LOCAL_API_TARGET=http://127.0.0.1:8082`、`VITE_ENV=development`。

| 验证 | 结果 | 结论 |
|---|---|---|
| `http://127.0.0.1:5173/` | 200 | 前端可访问 |
| `POST 5173/api/auth/user/login.do`（假账号） | `{"status":300,"msg":"INTERNAL_ERROR_PLEASE_RETRY"}` | 与直连 tst 的 `/api/auth/user/login.do` 响应**逐字一致** → auth 确实转发到 tst |
| `5173/api/admin/*`（无 token） | 与 `127.0.0.1:8082` 直连一致的业务错误 | admin 确实打到本地，未走测试环境 |
| 直连 `tst/auth/user/login.do`（去掉 `/api`） | nginx `405 Not Allowed` | 带 `/api` 前缀的路径才是正确入口 |

## 3. 本地 admin 能校验 tst 签发的 token

`application-lmp-reston-dev.yml` 与 `application-lmp-reston-tst.yml` 的 `spring.redis` 段（各自约 43–49 行）**host、port、database 完全相同**（database=1），即本地 admin（profile=dev）与 tst 共用同一 Redis 库。

推论（重要）：tst 登录签发的 token 在本地 admin 的 `tokenService.checkToken` 上应能通过，**不存在「本地不认 tst token」**。排查时不必再怀疑这一层。

## 4. 前端 token 存取与失效路径（代码事实）

- 写入：`src/utils/cookie.ts:6-7` — `setToken` 调用 `Cookies.set(Token, token)`，**没有设置 `expires`/max-age**，因此是 session cookie，关闭浏览器即消失，重开必须重新登录。
- 请求头：`src/services/request.ts:33-39` — 仅当 `getToken()` 有值且不是登录接口时才 `config.headers.Authorization = token`；无 token 时静默不带头。
- 失效清理：`src/services/request.ts:89-97` — 已有 token 且后端返回 401 时提示 `Login expired. Please sign in again.`，执行 `removeUserInfo()` 并 `window.location.href = '/'"` 刷新回登录，视觉上等同「token 一直是空的」。
- 登录取值：`src/views/Login/index.vue:63-72` — token 取自 `res.data?.Authorization`；为空时提示英文 `Login failed: Token not obtained`。
- cookie 按 origin 隔离：`5175/5176` 等曾启动过的端口、以及 `localhost` 与 `127.0.0.1`，token 都不共享。

## 5. 尚未闭环

用户尚未说明「token 为空」的**具体出现位置**，需要其一才能定位：

1. 登录页英文提示 `Login failed: Token not obtained` → 问题在 tst 登录接口返回值；
2. 页面红色 toast → 某个业务接口报错；
3. 控制台某请求无 `Authorization` 头 → cookie 缺失，优先查 session cookie 与 origin 差异。

若确认只是「每次重开浏览器都要重新登录」，修法是给 `src/utils/cookie.ts` 的 `Cookies.set` 增加 `expires`（如 12 小时/7 天）；属代码改动，须用户确认后再动。

## 6. 相关文档

- `38622843-363f-4bc2-ad74-dbdaba9e027d` — Admin 本地 dev vs tst 跨环境 488 端点对照：记录了同一套本地 8082 / 远端 `/api/admin` 对照方法，以及 dev profile 缺 `kmverse.ask.admins` 等配置差异（本文的 token 链路排查与其互补）。
