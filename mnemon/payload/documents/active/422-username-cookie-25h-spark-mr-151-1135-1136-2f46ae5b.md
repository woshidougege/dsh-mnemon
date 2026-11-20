---
id: "2f46ae5b-9d1b-461d-8ea1-90ad5b6e96f4"
title: "建会话 422 修复实施交接：username cookie 25h 有效期 + spark 会话归属以登录身份为准（MR !151 / !1135 / !1136）"
description: "承接诊断文档 3fcc7eea 的执行版方案：前端 setUsername 加 25h 有效期并在 fetchUserInfo 后用服务端 itcode 回填；spark 侧 CreateSessionRequest.itcode 改为可选、create_session 改用 _resolve_user_scope 以登录身份为真源。含精确行号、三条 MR 与 commit、验证状况、发版后是否需刷新页面的结论，以及 worktree/PowerShell 编码两个坑。"
status: "active"
created_at: "2026-11-20T07:34:43.150Z"
updated_at: "2026-11-20T07:34:43.150Z"
content_hash: "634f2d2d654639d3753e4d326e28626b563736b81d7697ea30bc70ab07f11050"
source_paths:
  - "src/utils/cookie.ts"
  - "src/stores/user.ts"
  - "app/api/routes/sessions.py"
session_ids:
  - "mnemon-review-v1-9538179b-0559-4202-90ab-2338a823893c"
memory_body_ids:
  []
---

# 建会话 422 修复实施交接：username cookie 有效期 + spark 会话归属以登录身份为准

**状态（2026-11-20）**：修复已实施并推送，**3 个 MR 待用户合并**；spark 侧需部署才生效。
**前情**：诊断与三条建议见文档 `3fcc7eea-71ef-4ff0-8296-3e5f9ac945ae`（当时标注"未执行"）。本文只记录被执行的那一版方案，不覆盖该文档。

## 1. 根因（承接诊断结论）

登录身份写在两处：cookie `username`（供前端 `createSession` 传 `itcode`）与 SSO/受信头解析出的服务端登录态。前者是无 expires 的 session cookie，浏览器关闭即丢；后者由 25h 续期窗口维持。两者一旦脱钩，`createSession` 发出 `itcode=undefined` → spark `CreateSessionRequest` 的 `Field(min_length=1)` 校验失败 422，而 422 不触发 401 清理分支，用户被留在原页面反复失败。

## 2. 采用的方案：两端都改（客户端兜底 + 服务端真源）

用户明确两边都要改、且 spark 同属自家仓库，因此不做单边兜底。

### 前端 ssg_sales_ops（2 files changed, +11/−2）

| 文件 | 改动 |
|---|---|
| `src/utils/cookie.ts:20-24` | `const USERNAME_EXPIRES_DAYS = 25 / 24`，`setUsername` 改为 `Cookies.set(UserName, userName, { expires: USERNAME_EXPIRES_DAYS })` —— 25 小时有效期，对齐后端 token 续期窗口，不再随浏览器关闭丢失 |
| `src/stores/user.ts:42` | `fetchUserInfo` 成功后新增 `if (userInfo.itcode) setUsername(userInfo.itcode)` —— 每次拿到服务端身份就回填 cookie，形成自愈路径（即使 cookie 已丢，一次成功请求即可恢复） |

依据：`src/stores/user.ts:25` 的 `fetchUserInfo(force = false)` 已经能拿到 `itcode`，且 `/api/auth/user/info` 在 token 有效时恒可调用；`src/views/Login/index.vue:66` 也已有 `setUsername(form.username)`，两条写入路径共用同一 cookie key `username`（`cookie.ts:4`）。

### spark 后端（1 file changed, +12/−8）

| 位置（`app/api/routes/sessions.py`） | 改动 |
|---|---|
| `:197-205` | `CreateSessionRequest.itcode` 由 `str = Field(min_length=1, max_length=128)` 改为 `str \| None = Field(default=None, max_length=128)`，docstring 写明"兼容旧前端的可选字段；会话归属以当前登录身份为准，仅在本地开发或无 token 的内部调用时作为兜底" |
| `:449` | `create_session` 由 `resolve_current_user(...) + current_user_for_itcode(request.itcode, ...)` 改为 `current_user = _resolve_user_scope(request.itcode, current_user)`，与列表/统计接口（`:537`、`:557`、`:604`、`:654` 等）统一 |
| `:2273-2282` | `_resolve_user_scope(itcode: str \| None, current_user: CurrentUser) -> CurrentUser`：先 `resolve_current_user`，**非 `local` 直接返回登录身份**，仅 `LOCAL_DEV_ITCODE = "local"`（`app/auth/context.py:13`）时才用 `current_user_for_itcode(itcode, tenant_id=current_user.tenant_id)` 兜底 |

净效果：客户端传的 itcode 不再能决定归属，身份真源回到登录态；旧前端不升级也不会再 422。

## 3. MR 与提交

| 仓库 | MR | 分支 → 目标 | commit | 状态 |
|---|---|---|---|---|
| ssg-spark | [!151](https://gitlab.xpaas.lenovo.com/ssg-sales/ssg-spark/-/merge_requests/151) | `fix/session-itcode-sso-identity` → `main` | `ea3b49f4` | opened / can_be_merged / conflicts=False |
| ssg_sales_ops | [!1135](https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1135) | `fix/itcode-cookie-master` → `master` | `2ce797ca` | 同上 |
| ssg_sales_ops | [!1136](https://gitlab.xpaas.lenovo.com/dataagent/ssg_sales_ops/-/merge_requests/1136) | `fix/itcode-cookie-develop` → `develop` | `f514c6eb`（cherry-pick） | 同上 |

两条前端分支 `git patch-id` 同为 `4566ef6233d1fb4044d04bb2e9140fe4e8d609b7`，确认是同一份补丁。

## 4. 验证

- spark：`python -m compileall` 通过；**pytest 未跑**（worktree 内未起容器），已在 MR 描述中如实声明为已知缺口。
- 前端两分支：`npm run type-check` 的报错全部是基线已有（`MarkdownRender/TableRender.vue`、`Toolkits/ToolEditorDialog.vue`、`Portfolio/config/OverviewModulesConfig.vue`、`OverviewModuleLinksFields.vue`），本次改动的 `cookie.ts` / `user.ts` 零错误。

## 5. 发版后的用户侧行为（用户原问题的答案）

- **spark 先合先部署** → 旧前端立刻恢复，用户**不需要刷新**（422 不再发生）。
- **前端两条都合再发版** → 前端属于「刷新一次才拿到新 bundle」的常规行为，刷新后 cookie 自愈逻辑生效；此后即使关浏览器也不会再丢 username。
- 顺序不敏感：任一侧先上都能消除症状，两侧都上才彻底移除复发路径。

## 6. 操作备注与踩过的坑

- 全程在独立 worktree 操作（spark 在 `C:/Users/63588/.codex/worktrees/spark-itcode/ssg-spark`，前端 master 分支 worktree 在 `.../worktrees/itcode-cookie-master/ssg_sales_ops`），**主工作区未动**：spark 主工作区仍留着 `fix/feedback-jira-allow-empty-details` 的 4 个未提交改动，前端 `develop` 主工作区仍留着 `feedbackMenuCatalog.ts` 的 `UU` 冲突。
- `git worktree remove` 在 Windows 上删除已有 worktree 可能报 `Directory not empty` / `.git/worktrees/... Invalid argument`，需手动清理目录后重建，不能只重试同一条命令。
- 用 PowerShell 脚本把中文 MR 标题写进文件会因编码转成乱码，触发 `TerminatorExpectedAtEndOfString` 解析错误；改为以 UTF-8 直接写 `.md` 文件再读取即可。
