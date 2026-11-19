---
id: "07982661-e4cb-4dd5-8487-4781c2ee6f9f"
title: "SPARK 邮件通知体系盘点与可预览标准下的 Thymeleaf 方案（订正 !592 的 FreeMarker 选型前提）"
description: "盘点 4 种邮件散在 3 套机制（FreeMarker 模板 / Java 拼串 ×2 / Word 导出 HTML + String.replace）、3 个 tableApply 孤儿模板；用户撤销「字节保真」约束并改为「可读 + 双击可预览」，据此把选型从 FreeMarker 反转为 Thymeleaf natural template，含目标目录结构与改造清单；状态为提案待批。"
status: "active"
created_at: "2026-11-19T06:13:17.916Z"
updated_at: "2026-11-19T06:13:17.916Z"
content_hash: "cb0de689f7b701efeceebb5fb5ff2a5412faa5abd700b72200af405f86220b05"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/notification/service/NotificationEmailNotifier.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/notification/service/JiraFeedbackEmailNotifier.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/menu/service/impl/MenusServiceImpl.java"
  - "smartop-service/admin/src/main/java/com/lenovo/admin/biadmin/service/BiFreshnessService.java"
  - "smartop-service/admin/src/main/resources/templates/email"
  - "smartop-component/smartop-component-mail/src/main/java/com/lenovo/component/util/MailTemplateUtil.java"
session_ids:
  - "mnemon-review-v1-7fa75bc2-1ee4-467e-94d2-a15f150fd38f"
memory_body_ids:
  []
---

# SPARK 邮件通知体系盘点与「可预览」标准下的 Thymeleaf 方案

状态（2026-11-19 14:12）：**调研与提案阶段，未开工**。checkpoint 结尾是向用户征询「要我动手吗？」，暂无答复。
前序交付：MR **!592** 已在 14:09 由用户合并进 `admin-release`（sha `ae561d92a6954c8b58c2abd0b8dd3ec585f4cd8b`，changes=12，title `refactor(notification): Jira 反馈邮件改用 FreeMarker 模板渲染`），即 FreeMarker 版已在发布分支上。

关联文档：
- **4766279b-2ba2-4d67-b468-c3de9e967690**（!592 FreeMarker 改造）——本文**订正其选型前提**：该文以「Thymeleaf 会 DOM 重序列化、做不到字节级一致」为由选 FreeMarker，而用户后续明确表示不要求字节保真（checkpoint event 11577：「我没说一字不差。只要大致差不多就行 甚至你可以优化 美化」）。字节保真这个约束是 assistant 自己加的。
- **8d4061ae-aa32-4062-a1d1-078b5cfbda27**（!590 反馈中心 SQL）——其「不做」第 3 条同样建立在 Thymeleaf 不可字节一致的前提上，受同一订正影响。

## 1. 用户真正提的验收标准

checkpoint event 11617（用户原话意图）：本地代码可读性要高，能通过代码看出「一共有多少种邮件样式」，最好本地直接双击 html 就能在浏览器看到样式。

这条标准替换掉被撤销的「逐字节一致」，直接改变了技术结论。

## 2. 盘点结论：4 种邮件，散在 3 套机制里

| # | 邮件 | 实现方式 | 位置 |
|---|---|---|---|
| 1 | Jira 反馈进度 | FreeMarker 模板 | `smartop-service/admin/src/main/resources/templates/email/feedback/jira-ticket-update.ftlh`（8228B） |
| 2 | 通知中心通知 | Java 内 26 行 `+` 拼串 | `smartop-service/admin/src/main/java/com/lenovo/admin/notification/service/NotificationEmailNotifier.java:80-105`（`render`），发送点 `:73` `mailService.sendMailFrom` |
| 3 | 报表分享 | `{{}}` 占位符 + `String.replace`，Word 导出的老 HTML（`class=WordSection1`、`MsoNormal`、等线字体） | 模板 `templates/email/menu/ReportMenuShare.html`（1095B，28 行），Java `smartop-service/admin/src/main/java/com/lenovo/admin/menu/service/impl/MenusServiceImpl.java:794` 取模板、`:800` `mailService.sendMail` |
| 4 | 数据新鲜度告警 | Java 内 2 行拼串 + 手写 `escape()` | `smartop-service/admin/src/main/java/com/lenovo/admin/biadmin/service/BiFreshnessService.java:451-452`（`sendMail` :445-459），发送点 `:453` `mailService.sendMailIter`；触发点 `:390`（data not refreshed）、`:419`（linked tables out of sync） |

占位符与工具类：
- ReportMenuShare.html 的 5 个占位符：`{{date}}` `{{pageLink}}` `{{receiver}}` `{{reportName}}` `{{sender}}`
- 取模板工具：`smartop-component/smartop-component-mail/src/main/java/com/lenovo/component/util/MailTemplateUtil.java:26` `public static String getHtmlTemplate(String htmlTemplate)`

孤儿模板（来自已删除的 datacenter 模块，无人引用，构成本盘点最大噪声）：
- `templates/email/tableApply/applyNotify.html`（1380B）
- `templates/email/tableApply/applyPass.html`（1058B）
- `templates/email/tableApply/applyReject.html`（896B）

即：用户「翻目录数不清有几种」是真实的——要同时看模板目录、Java 里的字符串、再判断哪个是死文件。

## 3. 为什么 FreeMarker 做不到双击预览

`.ftlh` 里 `<#assign>`、`<#if>`、`\${...}` 是 FreeMarker 指令，浏览器不识别：会看到 `= completed?then(...)` 当正文显示，且 `Completed`/`Created` 两个分支同时出现。

Thymeleaf 的 natural template 反之——`th:text` 是标签属性，浏览器忽略未知属性并显示标签内静态文字，服务端渲染时才替换：

```html
<!-- Thymeleaf：双击显示 Completed，渲染显示真实值 -->
<span class="badge" th:text="\${badge}">Completed</span>
<!-- FreeMarker：双击显示 \${statusText} -->
<span class="badge">\${statusText}</span>
```

结论：同一份文件既是预览又是模板，正好命中「可读 + 可预览」标准。

## 4. 提案的目标结构

```
templates/email/
├── README.md                          # 4 种邮件、哪个 Service 发、什么触发
├── feedback/jira-ticket-update.html
├── notification/notify.html
├── menu/report-share.html
└── biadmin/freshness-alert.html
```

动作：`spring-boot-starter-freemarker` → `spring-boot-starter-thymeleaf`；4 种邮件全部改成 Thymeleaf 模板（含把 `NotificationEmailNotifier`、`BiFreshnessService` 两处拼串搬进模板）；模板内保留样例文案（既是预览内容也是缺值兜底）；删除 3 个 tableApply 孤儿。验证方式：4 个模板各渲染一次 + 浏览器截图。预计 1–2 小时，从 `admin-release` 拉新分支，不动已合并代码。

## 5. 顺带记录的模板既有缺陷（非本次改造引入）

已在合并后向用户列出，仅 1–4 影响显示正确性，且都是纯模板改动：
1. `max-width:640px` 在 Outlook（Word 渲染引擎）无效，只认 `width`，需 `<!--[if mso]>` 包一层 640px 表格。
2. 外层 `<table>` 的 `padding:34px 12px`（模板第 30 行）Outlook 忽略，只认 `<td>` padding。
3. 无深色模式处理：卡片 `#ffffff` 会被反色，而内联 `color:#20243a` 不跟随。
4. 无 media query：480px 以下 25px 标题 + 双按钮并排会挤坏。
5. 可接受降级：`border-radius`、`box-shadow` 在 Outlook 全丢。
审美项（非必须）：三个紫色 `#4b3faf`/`#5b4bd6`/`#6d5ce7` 冗余、`Completed` 徽标与票卡 `statusText` 信息重复、640px 偏窄（680 更常见）。建议只做 1–4。

## 6. 教训

- assistant 不要把「已经发给用户的内容不能变」自行升格为「一个字节都不能差」的硬约束；该约束后来成了选 FreeMarker 的唯一理由，标准一撤，选型即反转。
- 判断「项目里有几种 X」时，必须把「模板文件 + Java 内拼串 + 未引用死文件」三类一起数，只看目录会得到错误答案。
