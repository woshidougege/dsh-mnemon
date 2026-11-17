---
id: "bb58fc3f-4d3b-48b3-8679-da5341fd6b89"
title: "Jira 455 张 SSGAI 单静默关闭可行性调研（XPaaS JIRA 11.3.12）"
description: "SSGAI 455 张 New 单关到 Done 且不发邮件的证据链：Jira 11 doTransition 已移除 notifyUsers、账号权限实测、4 步工作流与 transition id、Outgoing Mail 停机与 Bulk change 两条可行路线、Automation 残余风险。"
status: "active"
created_at: "2026-11-17T08:32:02.329Z"
updated_at: "2026-11-17T08:32:02.329Z"
content_hash: "1187e44d3967c30b9a6ef49ae4eb85bb30a7a59e8a7bab4888666e7fcb4d8bd7"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-b5140059-c39f-4ba5-b7b3-5695275e4f8f"
memory_body_ids:
  []
---

# Jira 455 张 Review-wrong 单静默关闭可行性调研

结论先行：在 **XPaaS JIRA 11.3.12（Server 版）** 上，用 REST 把 455 张 New 状态的 SSGAI 单批量关到 Done **无法保证不发通知**——Jira 11 已从 doTransition 移除 `notifyUsers` 参数，而唯一保留该参数的接口需要站点/项目管理员权限，本账号没有。可行路径只剩两条：管理员临时关闭 Outgoing Mail（100% 兜底），或用户自己在 UI 走 Bulk change 并取消勾选 `Send mail for this update`。

## 一、背景与硬约束

- 生产已由自动化建出 450~455 张 `[Review wrong]` 类工单（SSGAI），均为 **New** 状态、reporter=renxp4、assignee=longyu3、watchCnt=1、**with comments = 0**；labels 为 `Bookings_Raw`、`DataMind_AI`、`sdq-probe`。
- `GET /rest/api/2/issue/{key}/watchers` 实测 watchers 里只有 renxp4 一人，且通知方案 probe（`/rest/api/2/notificationscheme` 或对应探测接口）返回 `total=0, values=[]` —— 建单通知只落在 renxp4 与 longyu3。
- 用户硬约束（两次重复强调）：**绝对不能发邮件，也不能发站内信**。没有删除权限的说法与实际不符（实测 DELETE_ISSUES=true），但删除同样会触发 "Issue deleted" 通知，且关闭通知仍需管理员，因此删除不是出路。

## 二、工作流事实（实测 /issue/{key}/transitions）

SSGAI 项目（id=35100，"SSG Sales Ops BAU"）状态集合：Analyzed / In Progress / Rejected / Ready for Testing / Testing / Reopen / Pending / New / Done。

各状态可去往（括号内为 transition id）：

| 当前状态 | 可去 |
|---|---|
| **New**（455 张所在） | Pending(321) / Analyzed(11) / **In Progress(61)** |
| In Progress | Pending(321) / Analyzed(41) / New(51) / Rejected(71) / **Ready for Testing(81)** |
| Ready for Testing | Pending(321) / **Testing(91)** / In Progress(301) |
| Testing | Pending(321) / **Done(101)** / Reopen(111) / Ready for Testing(291) |
| Rejected | Pending(321) / Done(131) / New(161) |
| Done | Pending(321) / Testing(171) / Analyzed(191) / Rejected(181) |
| Pending / Reopen / Analyzed | 无通往 Done 的路 |

因此 New → Done 的最短合理路径是 **4 步**：`In Progress --81--> Ready for Testing --91--> Testing --101--> Done`。

另一种 3 步走法 `In Progress --71 Reject--> Rejected --131--> Done` 被否掉，因为会在单据历史里留下"被拒绝"的错误记录。

New / In Progress / Ready for Testing 三步的 `transitions?expand=transitions.fields` 均返回 `fields:{}`，不要求填额外字段；Testing→Done 是否强制 resolution 未知，需执行时观察（预案：自动补 Fixed）。

## 三、为什么 REST 静默不可行（本次推翻了早先的方案）

- `GET /rest/serverInfo`：version **11.3.12**，deploymentType **Server**，baseUrl `https://jira.xpaas.lenovo.com`。
- 查官方 v11000 REST 文档（`developer.atlassian.com/server/jira/platform/rest/v11000/api-group-issues/`）：
  - `POST /api/2/issue/{issueIdOrKey}/transitions`（operationId `doTransition`）的参数表**只有 issueIdOrKey**，`notifyUsers` 已被移除。
  - 仍保留 `notifyUsers` 的只有 `DELETE /issue`、`POST /issue/archive`、`PUT /issue`，描述明确写 "Admin or project admin permissions are required to disable the notification"。
- 账号权限实测（`/mypermissions` 与全局权限）：ADMINISTER=**false**、ADMINISTER_PROJECTS=**false**、BULK_CHANGE=**true**、TRANSITION_ISSUES=true、RESOLVE_ISSUES=true、CLOSE_ISSUES=true、EDIT_ISSUES=true、DELETE_ISSUES=true、CREATE_ISSUES=true、SCHEDULE_ISSUES=true、WORK_ON_ISSUES=true。
- 推论：既无参数可传、也无管理员权限，REST 改状态必然按通知方案发信；455 × 4 步 = 1820 次状态事件。

## 四、可行路径（按保险程度排序）

### 路线 A（100% 兜底，需管理员）：临时关闭 Outgoing Mail
管理员进 Settings → System → Mail Servers，取消邮件服务器 Enabled 并保存。此窗口内 Jira 任何通知都发不出，连 Automation 规则产生的邮件也会被拦截。之后用 REST 脚本把 455 张单走完 4 步（估算 1820 次请求，串行 15~25 分钟），跑完立刻恢复邮件服务器。该路线不依赖 `notifyUsers`，因此不受 Jira 11 移除参数的影响。

### 路线 B（无需管理员，需用户在界面操作）：Bulk change 取消勾选
Bulk Change 向导最后一步的 **`Send mail for this update`** 复选框（默认勾选）走的是 Jira 内部批量机制，不是 REST 的 `notifyUsers`，因此不受该限制影响；BULK_CHANGE=true 足够。455 张需 4 轮，每轮一条 transition：

| 轮次 | JQL | 动作 |
|---|---|---|
| 1 | `status = New` + labels/summary 过滤 | Transition → In Progress |
| 2 | `status = "In Progress"` 同上 | Transition → Ready for Testing |
| 3 | `status = "Ready for Testing"` | Transition → Testing |
| 4 | `status = Testing` | Transition → Done（若强校验 resolution，统一选 Fixed） |

每轮确认页都要取消勾选 `Send mail for this update`；JQL 单次上限 1000 张，455 张在限内。

### 站内信
Jira Server/DC 没有云版的 in-app 通知中心，状态变更只写入单据 Activity Stream / History，别人主动点开才可见，不推送。因此只要控制住邮件，就不存在"站内信"打扰。

## 五、残余风险（未验证，需人工确认）

1. **Automation for Jira / ScriptRunner 监听器**：若 SSGAI 项目配了"状态变为 Done 就通知 reporter"之类规则，那是 Jira 内部自发邮件，路线 B 的 Send mail 复选框**管不到**，只有路线 A（关邮件服务器）能兜住。需到 Project settings → Automation 查看。
2. 已发出的建单通知无法回收（已确认只落在 renxp4、longyu3 两个账号，范围可控）。
3. `notifyUsers=false` 的抑制效果无法先"试运行"验证，因为验证本身即写操作；因此建议先试点单张（例如最早的 SSGAI-978），确认无邮件后再全量。

## 六、未执行事项（交接）

本次仅做调研，**没有任何写操作**：未试点、未批量流转、未删除、未加评论。用户在对话中明确纠正过"我是让你分析 没让你改代码"。等待用户在路线 A 与路线 B 之间选择，以及是否先试点 SSGAI-978。

相关既有文档：生产批量建单现状与 Jira 设置页结构见 `2a95b6da-13f6-48f7-ace0-52cef76ec38f`。
