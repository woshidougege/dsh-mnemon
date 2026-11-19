---
id: "ef8bc24f-cb4e-4c20-8bfe-375427240ddd"
title: "SSG Sales OPS「问数没反应」排障手册：埋点判据、分层排除与 token 续期生产验证"
description: "用户报问数没反应时的定位方法：用 ssg_chatbi_event 的 send_click vs query_session vs answer_status 先切成前端/后端两半；Redis token、服务存活、Trino、AI 网关的分层排除查法与结果；AgentChatPanel.vue:812 loading 静默吞点击、:857 getBotlandSign 前置两个嫌疑点；以及 2026-11-19 token 续期生产生效的三条验证判据。"
status: "active"
created_at: "2026-11-19T08:13:08.765Z"
updated_at: "2026-11-19T08:13:08.765Z"
content_hash: "84b80450a71ca6b609eeb64b64320371f6e78c98414f4e34ff2e1a5bdcd3b833"
source_paths:
  - "src/components/Agent/AgentChatPanel.vue"
  - "src/views/Home/index.vue"
  - "src/analytics/events.ts"
session_ids:
  - "mnemon-review-v1-05ee4d52-a8a0-4db8-8c1b-092fec485e55"
memory_body_ids:
  []
---

# SSG Sales OPS「问数没反应」排障手册：埋点判据 + 分层排除（2026-11-19）

关联：token 续期链路的诊断与实现见 `3e165d1b-e482-47fc-ae8a-3aa629f85665`（根因）与 `3ea6d59b-4306-4a90-a819-3cd162fc0732`（实现/MR）。本文档记录另一条独立链路：**用户报"问数没反应"时怎么在 10 分钟内定位到层**。

## 1. 事件

2026-11-19 用户报 zhaojm5@lenovo.com "问数没反应"。同时段 longyu3 也连续 3 次点发送同样无任何业务记录 → **系统性问题，不是单账号**。

## 2. 核心判据：用埋点把故障切成两半（先看这个）

表 `ssg_chatbi_event`（字段 t / event_name / user_id / status / mode / q）。一次问数周期里三类事件的产生时机：

- `send_click` — 前端点击发送**立即**上报（`src/views/Home/index.vue:829` → `track(AnalyticsEvent.SendClick, …)`，枚举定义在 `src/analytics/events.ts:6`）
- `query_session` — 前端**拿到响应后**才上报（status=success）
- `answer_status` — 流式答案结束（status=completed）

判读规则：

| send_click | query_session | 结论 |
|---|---|---|
| 有 | **全无** | 请求根本没发出/没到后端 → 查前端发送环节，别在 Trino / LLM 侧浪费时间 |
| 有 | 有 | 后端处理了 → 看 `answer_status` 与 `ssg_chatbi_failed_query_jira` 的 failed_step / failed_reason |
| 无 | 无 | 连点击都没到埋点 → 前端页面/埋点本身挂了 |

本次证据：16 点 send_click 5 次（zhaojm5 两次：16:03:57、16:04:42；longyu3 三次：16:05:26 / 16:07:05 / 16:08:05），**query_session=0、answer_status=0**；全站最后一次 query_session 停在 15:33:50。之前的小时是正常的（14 点 17 次 send_click / 10 次 query_session，15 点 6 / 3），说明 15:37 之后进入故障态。

## 3. 分层排除（本次已排除的层，可复用查法）

| 层 | 查法 | 本次结果 |
|---|---|---|
| 登录凭证 | Redis `token_<itcode>` 存在性 + TTL + idle，另看 `login_degraded` / `profile_incomplete` 字段 | zhaojm5 TTL=83266s(23.13h)、idle=6s、degraded=0、profile 完整；52 分钟前刚被自动续期过一次 |
| 服务存活 | 看 `page_view` 埋点是否仍在持续上报（它不走业务链路，是独立存活信号） | 到 16:08 仍在正常写 |
| Trino | `select 1` 与 `select count(*) from hive.prd_upsell.bookings_simulation_ssg_kpi`（15,500,975 行） | 2790ms / 1496ms，正常 |
| AI 网关 | GET `{aiverse base-url}/models` | 200 / 974ms，正常 |

查表时的两个坑（本次踩到）：
- `ssg_admin_assistant_session` 的时间列是 **`created_time`**，写 `create_time` 直接报列不存在；该表另一组列是 `created_at` / `updated_at`，两套并存别混用。
- `ssg_chatbi_failed_query_jira` 没有 `itcode` 列（只与 user_id / jira_key 相关）。

Admin 库（db `ssg_ops_agent`）中与本链路相关的表：`ssg_chatbi_event`、`ssg_chatbi_question_review*`（6 张）、`ssg_chatbi_failed_query_jira`、`ssg_chatbi_review_jira`、`ssg_chatbi_profile_group`、`ssg_chatbi_internal_user`、`ssg_chatbi_event_probe_trigger_bak_20261103`、`ssg_admin_assistant_session`。

## 4. 前端两个高嫌疑点（截至本记录仍未结案）

- `src/components/Agent/AgentChatPanel.vue:812` — `if (loading.value) return`：上一轮请求的 loading 未复位时，后续点击被**静默吞掉**（埋点已记、请求不发），体感与"没反应"完全一致。
- `src/components/Agent/AgentChatPanel.vue:857` — 发送链第一步 `await api.getBotlandSign({...})` 拿签名；该接口不返回时整条发送链停在此，页面同样表现为没反应。签名接口失败会卡在 `getBotlandSign`，流式接口根本不会被创建。
- 相邻干扰项：`AgentChatPanel.vue:829` 处，当 Runtime 停在 `waiting_input` 时普通消息会进 `queued_turns` 排队，前端只弹一句 `ElMessage.warning('The previous turn is still waiting for your input.')`；用户忽略提示就会认为"问了没反应"。

## 5. 下一步验证动作（对应用户执行）

1. Ctrl+F5 强刷后重试 —— 若只是 `loading` 残留，刷新即恢复。
2. 仍无反应则 F12：Console 看红字；Network 看是否有一直 pending 的 `getBotlandSign` 或 chat 流式请求。
3. 拿到截图即可区分"签名接口卡住"vs"流式连接被拦"；若签名接口稳定卡住，再压 bot-land 侧（`/chat/getAccessToken`、`/chat/ext/getSign`）。

## 6. 旁证：token 续期已在生产生效（2026-11-19 验证）

沿用 `3ea6d59b` 的实现，生产验证的三条判据：

- **脚本在跑**：实时命令流抓到 `EVALSHA 97d5ab6f… 1 token_<itcode> <token> 82800 86400`，sha 与已合入脚本一致，参数正是 23h 阈值 / 新 TTL。
- **旧命令消失**：55 秒窗口内 EVALSHA +22、HGETALL +22、脚本内 HGET +44（=22×2，比对 token + 判降级），而客户端 `PEXPIRE` 与 `HGET token` 均为 **+0** —— 旧版 checkToken + renewToken 路径确实不再执行。
- **TTL 分布符合"按阈值续期"**：活跃用户 23 个，其中 13 个 TTL 在 82800~86400；刚访问过的仍是 85461 / 84263 / 83388（idle 只有几十秒）—— 旧逻辑下这里应是清一色 TTL+idle≈86400。
- 客户端侧仍需浏览器确认：Response Headers 出现 `Set-Cookie: Authorization=…; Max-Age=90000`，Cookie Expires 为 25 小时后。**昨天以前就已掉线的人必须重新登录一次**，因为续期只对有有效 token 的请求生效。

附带一条实例健康判据：用 Redis `CLIENT LIST` 的 age 分布看某应用实例是否刚重启 —— 10.62.222.146 maxAge≈8.8h、10.62.222.187 maxAge≈19h（说明部署过了），而 10.62.79.134/.174/.178 的 age 是数千小时（Redis/sentinel 侧长期未重启）。
