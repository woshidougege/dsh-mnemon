---
id: "71393374-411d-4d4c-8f7d-8570f050506b"
title: "手机端远程配对后聊天图片无法预览：三层根因与 setAttribute 补丁方案（dsh-remote-web-ui 0.4.19）"
description: "远程配对设备打开官方 UI 后，聊天 Markdown 图片显示“无法预览”的多层根因（/api/file 只认 owner、Host 围栏、插件只补了 img src setter 而 React 走 setAttribute），含实测 Host/cookie 对照、/remote 通道可用的证据、三条可行路径与待实施的 setAttribute 补丁方案。"
status: "active"
created_at: "2026-11-19T03:13:44.838Z"
updated_at: "2026-11-19T03:13:44.838Z"
content_hash: "44a438f9ce8c73c9e2e4e41293eb2d2ea7c327c5bc0fe4eb879246ea43cd970f"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-cb603de3-9195-4322-b0d0-142fef2ab6a2"
memory_body_ids:
  []
---

# 手机端远程配对后聊天图片无法预览：三层根因与 `setAttribute` 补丁方案

环境：DSH 桌面端 Web GUI `http://127.0.0.1:57921`，手机通过插件 `@linxin666/dsh-remote-web-ui` 0.4.19（`C:\Users\63588\.dsh\profiles\desktop\node_modules\@linxin666\dsh-remote-web-ui`）扫码配对后进入同一套官方 UI。

结论先说：**没有可调参数能解决**（`remoteAccess` 无论 `read-only` 还是放开都无关），这是插件补丁覆盖面的代码缺口。根因已定位，补丁方案已定，**截至记录时未实施**，等用户确认。

## 1. 现象与范围

- 聊天里 `![](C:\\...png)` 在手机端显示「图片无法预览」；同一会话内文字聊天、附件上传都正常。
- 本地（回环浏览器）一切正常，只有经 `/remote` 通道进来的配对设备失败。

## 2. 三层叠加的根因

### 第一层：`/api/file` 只认 owner 身份（硬编码）

`dsh-client-connection/lib/index.js` 中：

```js
if (peer.principal.kind !== "owner")
  return new Response("forbidden", { status: 403 });
```

远程浏览器配对后仍不是 owner，因此裸 `/api/file` 永远 403。

### 第二层：公网/局域网 Host 过不了 Host 围栏

`isTrustedApiRequest(request, trustedHosts)` / `isTrustedHost(request, trustedHosts)`（`lib/index.js:2257`、`lib/index.js:2290`，类型侧同源 `lib/types/routes.js:64`、`lib/types/routes.js:115`）只接受 loopback、bind IP 或显式 `trustedHosts`；`trustedHosts` 属于 harness web server 配置，不在插件里。

实测（回环端口 57921，同一 `/api/file` 路径只换 Host 头）：

| Host 头 | 结果 |
|---|---|
| `127.0.0.1:57921` | 401（进入认证，未过 owner） |
| `localhost:57921` | 401 |
| `39.96.21.173:57921` | **403** |
| `example.com` | **403** |

即便加上 `trustedHosts`，手机也没有 harness 的浏览器认证 cookie（cookie 与访问过的 host 绑定），图片请求照样过不去。

### 第三层：插件自己的设计边界

插件 README（"运行时现实：局域网浏览器走 `/remote` 网关，不走原生 `/api`"）明确写：本插件从不重绑 web server，宿主停在本机回环，harness 的 `/api` Host 围栏对局域网权威返回 403；局域网权威上只有插件自己的面在应答（配对入口页、状态视图、门控 `/remote` 通道）。要在局域网权威恢复原生 `/api` 需要 harness web server 的 `trustedHosts`，不在本包范围内。

## 3. 为什么聊天能用、图片不能用

插件把 `fetch`、`EventSource`、`WebSocket` 和上传改写到自己的门控通道（`lib/client.js` 的 `installRemoteChannel(window, options)`，:4681；`REMOTE_CHANNEL_RULES` :4561、`shouldRewriteFetchPath` :4589、`shouldRewriteWsPath` :4603、`rewriteRawUrl` :4614），这些请求不碰裸 `/api`。

但图片只包了 **`src` 的 setter**：

```js
patchSrcAccessor(window.HTMLImageElement, rewrite)   // lib/client.js:4749
```

`patchSrcAccessor(ctor, rewrite)`（:4655）通过 `Object.getOwnPropertyDescriptor(ctor.prototype, "src")` + `Object.defineProperty` 只替换 `set(value)`：

```js
set(value) { originalSet.call(this, rewrite(String(value))); }
```

而 React 渲染 `<img src="/api/file?...">` 走的是 `setAttribute('src', ...)`，**不经过属性 setter**，于是仍发出裸 `/api/file` 请求，命中第一、二层的 403，前端显示「图片无法预览」。验证：在 `lib/client.js` 中 grep `setAttribute`、grep `img` 均无命中，确认该路径没有任何改写。

## 4. 通道本身是可用的（关键证据）

- 通道路由：`REMOTE_PREFIX = "/remote"`、`REMOTE_API_PREFIX = "/remote/api"`（`lib/index.js:2049`、:2051）；`makeRemoteApiRoutes(deps)`（:3771）先 `pairedDeviceIdOf(req, service)` 校验配对设备，再用 `innerPathOf(url.pathname)`（:3736，拒 `/remote` 本身、要求每段 `isSafeSegment`）映射回内层路径，`loopbackOnlyDenial(inner)` 拦 LOCAL_ONLY 前缀，最后 `proxyLoopbackHttp(req, res, port, inner+search, deps.auth)` 带进程内部凭据回环转发。
- 配对 Cookie 名为 `dsh_pair`（schema `cookieName` 默认值在 `lib/index.js:19547`，实际取值 :19615）；`pairedDeviceIdOf` 先读 cookie 再读 `x-dsh-remote-device` 头，未知/撤销 id 视为无效，不会因过期 id 重新激活设备。
- 实测对照（同一 `path=C:\Users\63588\public\outputs\dsh-window.png`）：
  - `GET /remote/api/file?...` + `Cookie: dsh_pair=<设备 id>` → **302**（`cache-control: private, no-store`），说明通道认了凭据并把请求转给内层文件路由；
  - 同路径不带 cookie → **403 application/json**，body 为 `{"type":"server-response","rpcId":"invalid-request","result":{"ok":false,"error":{"code":"unpaired",...}}}`；
  - 用错的 cookie 名（`dsh-remote-device` / `remote-device`）→ 仍 403；
  - `GET /api/file?...` + `dsh_pair` → **401**（被 harness 认证层挡住）；
  - 通道上不存在的 `session/list` → 404 text/plain（进一步说明通道按路径白名单转发）。
- 设备表当前 4 条记录（字段 `createdAt,lastSeenAt,userAgent`），最新一条 UA 为 `Mozilla/5.0 (Phone; OpenHarmony 7.0; Android 10)`、`lastSeenAt: 1795057507441`。设备 id 是活凭据，不记录具体值。
- 安全细节：外层 WebSocket 握手的 `?device=<id>` 在转发内层时会被丢掉（`lib/index.js:3813` 附近注释），因为门控已完成认证，且内层回环上任何本地观察者都能看到该凭据。

## 5. 三条可行路径

1. **立刻绕过（已验证）**：图片改用附件上传（手机端可正常显示）；或把图放公网可访问 URL，聊天里写 `![](https://...)`，这类链接不走 `/api/file`。
2. **修插件（正解，待用户拍板）**：在 `installRemoteChannel` 的补丁集里再包一层 `HTMLImageElement.prototype.setAttribute`，对同源 `/api/file`（及其他被围栏挡住的同源 `/api/...`）改写到 `/remote/api/...`——通道已持有 owner 内部凭据，能读文件。改动范围仅 `C:\Users\63588\.dsh\profiles\desktop\node_modules\@linxin666\dsh-remote-web-ui\lib\client.js` 一个文件，需重启桌面端生效，回滚就是还原该文件；注意 npm 覆盖安装会丢失补丁。
3. **等上游**：属插件 `img` 补丁覆盖不到 React 渲染的缺口，可向作者提 issue（0.4.19，engines `dsh >=0.2.2-rc.3`、`node >=24.9.0`）。

## 6. 排查时踩过的坑（省下次的时间）

- `remoteAccess: read-only` 管的是 DSH 工具读写授权（能否远程改文件/跑命令），**与图片预览无关**，改它无效。
- 在通道上试 `/remote/api/session/list` 得 404 是正常现象（该路径不在转发白名单），不代表通道坏。
- 用错 cookie 名会得到与"未配对"相同的 403，容易误判为设备失效；务必用 `dsh_pair`。
- 压缩上下文时若摘要不比原区间短会被拒（"no net shrink"），同一 spec 重复提交必然同样失败——换更大区间或直接继续任务。
