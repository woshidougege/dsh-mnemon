---
id: "3fcc7eea-71ef-4ff0-8296-3e5f9ac945ae"
title: "SSG Sales OPS 续期副作用：username session cookie 丢失导致建会话 422"
description: "11-19 上线 token 续期回写（Authorization 25h 且不断续期）后，前端 session cookie username 一旦随浏览器关闭丢失就没有恢复路径：createSession 的 itcode=undefined → spark 422，且不报 401、不跳登录页。含排查路径、排除项（前端与 spark 均未改）、zhangll70 时间线、三条修复建议（未执行）。"
status: "active"
created_at: "2026-11-20T06:52:26.370Z"
updated_at: "2026-11-20T06:52:26.370Z"
content_hash: "ecb94ef52d3573ce4fe333c5d55043487a0f3121e1606146c6b6cb09853cd273"
source_paths:
  - "src/utils/cookie.ts"
  - "src/views/Login/index.vue"
  - "src/components/Agent/AgentChatPanel.vue"
  - "src/services/request.ts"
  - "src/stores/user.ts"
  - ".agents/worktrees/auth-token-slide/smartop-common/smartop-common-lmp/src/main/java/com/lenovo/common/lmp/provider/ProviderAuthRestInterceptor.java"
  - ".agents/worktrees/auth-token-slide/smartop-common/smartop-common-core/src/main/java/com/lenovo/common/cors/CorsConfig.java"
session_ids:
  - "mnemon-review-v1-d090365b-09b1-4e2e-b13a-dbcc7a32acc8"
memory_body_ids:
  []
---

# SSG Sales OPS 续期修复的副作用：session cookie username 丢失 → 建会话 422（itcode undefined）

**关联文档**：3ea6d59b-4306-4a90-a819-3cd162fc0732（续期修复实现与四个 MR）、3e165d1b-e482-47fc-ae8a-3aa629f85665（原始根因：Redis 滑动 24h vs 浏览器 cookie 固定 24h）。本文是那轮修复上线后的**下游副作用调查**。

## 0. 触发与问题

2026-11-20 14 点前后，多名用户（zhangll70、sunpy2、zhangqp5、luaq1、niuna2…）在 chat_bi 点发送 / 新建会话无反应。用户原话：「为啥最近才出现这个问题呢 你看看 git 提交是前端改坏了吗？」

## 1. 结论

**不是前端代码改动引起的，但确实是「最近才出现」**——11-19 上线的 token 续期回写把「每 24 小时被踢回登录页」这个**自愈回路掐断了**，长期潜伏的 username session cookie 缺陷因此第一次有机会暴露。

## 2. 机制：两个 cookie 的寿命完全不对等

| cookie | 谁写 | 有效期 |
|---|---|---|
| Authorization | 登录 / 续期时服务端 Set-Cookie | **25 小时**，且每次真续期都回写 Set-Cookie（11-19 上线）→ 只要还在用就永不过期 |
| username | **只有前端账号密码登录页**（src/views/Login/index.vue:66 → src/utils/cookie.ts:17） | **session cookie，无 expires** → 浏览器重启 / 标签全关即丢 |

getUsername() 读的就是 username（src/utils/cookie.ts:19-21），两处建会话直接拿它当 itcode：

- src/components/Agent/AgentChatPanel.vue:1488 — itcode: getUsername() as string
- src/components/Agent/AgentChatPanel.vue:1705 — 同上

取到 undefined → axios 序列化时直接丢掉该字段 → spark 校验 itcode 必填 → 422。

**修复前的自愈回路**：登录下发 24h cookie 且此后永不刷新 → 满 24 小时浏览器必丢凭证 → 401 → 被踢回登录页 → 重新输账号密码 → username 被重新写入。隐患最多存活 24 小时。

**修复后的死锁**：Authorization 被续到 25h 且不断续期 → 用户再也不回登录页 → username 一旦因关浏览器丢失就**没有任何恢复路径**，而且不报 401、不跳转，用户只表现为「点了没反应」。

## 3. 关键代码位置

- .agents/worktrees/auth-token-slide/smartop-common/smartop-common-lmp/src/main/java/com/lenovo/common/lmp/provider/ProviderAuthRestInterceptor.java:188-206 — refreshTokenCookie：ResponseCookie.from(HttpHeader.AUTHORIZATION, token) + .maxAge(Duration.ofSeconds(tokenCookieMaxAgeSeconds))；:73 字段声明；:181 注释写明「服务端 token 是滑动续期的，但浏览器 cookie 的 Max-Age 只在登录（ADFS 回调）时下发过一次」
- 同文件 :284 — Access-Control-Max-Age: 3600；smartop-common-core/.../cors/CorsConfig.java:28 — config.setMaxAge(2592000L)
- src/utils/cookie.ts:7 — Cookies.set(Token, token)；:16-18 — setUsername；:4 — const UserName = 'username'；:17 — Cookies.set(UserName, userName)
- src/services/request.ts:88-97 — 401 分支：if (getToken() && error.response?.status === 401) → removeUserInfo() + window.location.href = '/'
- src/stores/user.ts:7 — itcode: string；:25-51 fetchUserInfo 从 /api/auth/user/info 取（**已有 itcode 可用**，是修复方案的依据）
- spark 侧（另一仓库 ssg-spark）app/api/routes/sessions.py:196-201 — class CreateSessionRequest，itcode: str = Field(min_length=1, max_length=128, ...)

## 4. 排除项（证据）

1. **前端没改过这块**：git log --since=2026-11-14 -- src/services/pbiApi.ts src/components/Agent/AgentChatPanel.vue src/utils/cookie.ts src/views/Login/index.vue → 空。itcode 写法自 7 月联调 3aedfae3（07-20）起就是这样；近三个月只动过 searchSessions 的可选参数（10-15），未碰 createSession。
2. **spark 侧也不是今天改的**：git log -S 'itcode: str' → 只有 07-15 feat(api): bind sessions to itcode。
3. **不是 401 清理导致的跳转**：zhangll70 页面 session_id = 43164cd5 从 09:31 到 11:37 一直没变（request.ts:97 的跳转会重载页面进而换 session_id）；且 document.cookie 里读不到 username（已丢）。
4. **不是全站故障**：agent.sessions 11-20 新建 85 个（11-17 为 132、11-18 为 125）；query_session 埋点 61 次里 59 次成功。只有「重启过浏览器 / 关过全部标签」的那批人陆续中招，会随时间推移变多。
5. 其他排除方向：psql 不在 PATH（改用 JDBC 探针 + .tmp/usage-feedback-workspace-runtime/base/BOOT-INF/lib/postgresql-42.5.4.jar 连 ssg_ops_agent）；/spark-api/health、/version、/ready、/status 全返回 {"detail":"SSO authentication required."}（无 cookie 时的正常行为，非故障），/spark-api/healthz 404，/health 返回前端 index.html。

## 5. 时间线（zhangll70，2026-11-20，最关键证据）

| 时间 | 事件 |
|---|---|
| 09:31 | 进站（页面 session 43164cd5） |
| 09:33:13 | send_click → **query_session 成功**，建出会话 87e88f80 |
| 09:54:05 | 最后一轮 answer_status completed |
| **10:19:43 → 11:17:29** | **埋点空白约 1 小时**（人离开 / 关浏览器） |
| 11:17:29、11:17:38 | 两条 page_view（回来了） |
| 11:37:25 → 11:37:27 | 点新建会话 + send_click → **没有任何 query_session** |
| 14:04 / 14:09 / 14:12 | 又试 4 次，全无 query_session |

agent.sessions：他 11-20 只建了 09:33 那一个；11-19 建了 4 个（10:16、11:06、14:23、14:33）—— 说明 11-19 他还有 username。

## 6. 数据背景（供后续查询复用）

- 事件表 ssg_chatbi_event（字段：event_id、event_time、event_name、user_id、session_id、page_name、page_path、page_url、source_system、server_env、properties jsonb、created_at、question、session_title、query_id、model、status、execution_mode、profile_name），DB ssg_ops_agent
- 会话表 agent.sessions（判「真实新建会话数」用它，而非 send_click 埋点）；另有 agent.session_runtime_events、agent.turn_runs、agent.pbi_user_sessions、public.ssg_admin_assistant_session
- 判据：send_click 埋点数 ≠ 真实会话数；只有 query_session 事件才代表会话真的建成功

## 7. 修复建议（**尚未执行**，等用户确认）

1. **前端兜底（最小改动，建议先做）**：createSession 的 itcode 改成 userStore.user?.itcode || getUsername()（/api/auth/user/info 里本来就有 itcode）；取不到就不发请求，直接提示重新登录。
2. **cookie 对齐**：setUsername 加上与 Authorization 一致的 25h + domain，不要再用 session cookie。
3. **后端治本**：/spark-api/sessions 从 Authorization 解析 itcode，不要让客户端传（需 spark 侧改）。

**给已中招用户的自救**：F12 执行 document.cookie='username=<你的itcode>; path=/' 然后刷新；或手动清掉 Authorization cookie 让自己被踢回登录页重登一次。

## 8. 遗留

父会话最后一轮在问用户「要我先按第 1 条出前端补丁吗？」，用户未答复 → 补丁未实施。
