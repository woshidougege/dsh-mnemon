---
id: "4766279b-2ba2-4d67-b468-c3de9e967690"
title: "Jira 反馈邮件改用 FreeMarker 模板渲染（推翻 !590 的“不做”判断，MR !591 → !592）"
description: "!591 只是把拼串搬进文件（控制流仍在 Java）被用户指出后，改用 spring-boot-starter-freemarker：选 FreeMarker 而非 Thymeleaf 的理由（无 DOM 重序列化，可字节级一致）、.ftlh 自动转义、328→249 行、8 个 html 片段合并为单模板、反射影子双渲染 2916 组矩阵 ALL IDENTICAL、FreeMarkerViewResolver 对 3 个 @Controller 的无副作用排查，以及 !592 交付状态与三条 SQL 遗留。"
status: "active"
created_at: "2026-11-19T06:05:53.705Z"
updated_at: "2026-11-19T06:05:53.705Z"
content_hash: "8fd06ad61256f90b904a5574207347c99f519f5993fb69fa59194fa07d39ecc7"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/notification/service/JiraFeedbackEmailNotifier.java"
  - "smartop-service/admin/src/main/resources/templates/email/feedback/jira-ticket-update.ftlh"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/notification/JiraFeedbackEmailTemplateTest.java"
  - "smartop-service/admin/pom.xml"
session_ids:
  - "mnemon-review-v1-a83f0fea-3700-4189-8c62-4eb0daff006a"
memory_body_ids:
  []
---

# Jira 反馈邮件改用 FreeMarker 模板渲染（!591 被推翻 → !592）

状态（2026-11-19 14:05）：MR **!592** 已创建（`feature/feedback-mail-freemarker-admin` → `admin-release`，state=opened，未合并）。!591 已由用户自行合并进 `admin-release`。

关联文档：
- **8d4061ae-aa32-4062-a1d1-078b5cfbda27**（MR !590，反馈中心 SQL 片段化）：其"三条明确不做"的第 3 条写"抽 Thymeleaf 后渲染结果不可能字节级一致，验证成本高于收益"——**本文推翻该判断**，原因见第 1 节。
- **c883d9b9-641a-4c06-a251-55f80717c97b**（SSGAI 455 张单）也涉及同一 `JiraFeedbackEmailNotifier` 的链路上下文。

## 1. 为什么第一版不算"用了模板引擎"（!591 的问题）

用户对 !591 的评价原意（m04196 附近）："我看还是拼接的呀，没用 Spring 的模板引擎。"这个判断是对的。!591 只是把字符串从 Java 搬进了文件：

```java
// !591：Java 里算好，再 replace
sb.append(ticketCard(...));
if (statusText != null) sb.append(statusPill(statusText));
if (resolutionText != null) sb.append(resolutionBlock(resolutionText));
String html = loadTemplate("jiraTicketUpdate.html")
    .replace("{{card}}", sb.toString())
    .replace("{{status}}", statusText);
```

`{{占位符}}` + `String.replace` 与 `+` 拼串等价：条件分支、片段拼装、转义全留在 Java。8 个 `*.html` 片段文件（jiraActionButtons / jiraNextStep / jiraPrimaryButton / jiraResolutionBlock / jiraSecondaryButton / jiraStatusPill / jiraTicketCard / jiraTicketUpdate）也是这个模式的产物。

## 2. 决策：选 FreeMarker 而不是 Thymeleaf（关键理由）

上一版之所以被判定"抽模板不划算"，隐含前提是只能用 Thymeleaf。**Thymeleaf 会把 HTML 解析成 DOM 再重新序列化，属性顺序、自闭合标签、空白都不可控**，所以字节级一致确实做不到。FreeMarker 是纯文本模板引擎，只在指令处替换，输出可预测——字节级一致因此可达。

落点：
- `smartop-service/admin/pom.xml` 新增 `spring-boot-starter-freemarker`（Spring Boot 管理版本，无显式版本号）
- 模板合并为单文件 `smartop-service/admin/src/main/resources/templates/email/feedback/jira-ticket-update.ftlh`（109 行），8 个 `*.html` 片段全部删除
- `smartop-service/admin/src/main/java/com/lenovo/admin/notification/service/JiraFeedbackEmailNotifier.java` 328 → 249 行（另一分支口径 191 行净变化，因基线不同）

模板内条件化：

```freemarker
<#assign accent = completed?then("#1f9d68", "#5b4bd6")>
<#assign badge = completed?then("Completed", "Created")>
<#if statusText?has_content>
  <span style="background:${badgeBackground};color:${badgeColor};">${statusText}</span>
</#if>
<#if jiraUrlSafe?has_content || homeUrlSafe?has_content>
  <#if jiraUrlSafe?has_content><a href="${jiraUrlSafe}">View in Jira</a></#if>
  <#if homeUrlSafe?has_content><a href="${homeUrlSafe}">Open SPARK</a></#if>
</#if>
```

Java 侧只剩两件事：组装 model（业务文案判定 + `safeUrl()` 协议白名单）和渲染。

```java
@Autowired private freemarker.template.Configuration freemarkerConfiguration;
private String render(Map<String, Object> model) {
    try (StringWriter writer = new StringWriter()) {
        freemarkerConfiguration.getTemplate(TEMPLATE, "UTF-8").process(model, writer);
        return writer.toString();
    } catch (IOException | TemplateException ex) {
        throw new IllegalStateException("Failed to render email template " + TEMPLATE, ex);
    }
}
```

**转义来源**：`.ftlh` 后缀命中 FreeMarker 的 HTML output format，`${...}` 自动 HTML 转义，因此旧代码里手写的 `escape()` / `HtmlUtils.htmlEscape` 全部可以删掉。注意逻辑：删掉手写转义是安全的**前提**是后缀必须是 `.ftlh`（改成 `.ftl` 会静默失去自动转义）。

## 3. 等价性验证：反射双渲染 2916 组矩阵

邮件已在线发出，改动必须先证明输出字节级不变。做法：
1. 把改造前 346 行原版类保留并反射加载进同一进程（`Snap2.java`，7061 字节的对照实现），与新版同输入双渲染；
2. 跑 2916 组参数矩阵，覆盖 null、空白串、`<script>`、引号、`javascript:`、`ftp://` 等；

```
TOTAL=2916 SAME=2916 DIFF=0 FAIL=0
RESULT: ALL IDENTICAL
```

结论：FreeMarker 自动转义与 `HtmlUtils.htmlEscape` 输出逐字节一致。这套"旧实现留在同进程做影子渲染 + 维度矩阵全覆盖"的手法，适用于任何"已上线输出不能变"的模板/序列化改造。

新增 `smartop-service/admin/src/test/java/com/lenovo/admin/notification/JiraFeedbackEmailTemplateTest.java`（115 行，4 用例，跑在真实 Spring 上下文），4/4 通过；其中 `escapesUserSuppliedValues` 断言 `non-http url must not reach href`。

## 4. 副作用排查：FreeMarkerViewResolver 自动注册会不会拦既有 Controller

引入 `spring-boot-starter-freemarker` 会注册 `FreeMarkerViewResolver`，理论上可能改变既有无视图返回值的 Handler 行为。实际排查：
- admin 模块 `@RestController` 57 个（不受影响）；
- 仅 3 个 `@Controller`：`aiprobe/controller/AiProbeReportController.java`、`ChatbiEventReportController.java`、`ProbeBaselineReportController.java`；
- 三个逐个体检——**返回视图名的方法数均为 0**，故新 resolver 不会截获现有请求。

顺带发现：后两者的实际路径不在 `smartop-service/admin/src/main/java/com/lenovo/admin/controller/`，首次 grep 报 `os error 3 系统找不到指定的路径`；真实位置以 `aiprobe/controller/` 这类子包为准，排查时不要按旧路径假设。另清理了 12 个 stale 文件（含 `deleted=12` 的一次性清理）。

## 5. 构建与测试

- `mvn` 全量 reactor 15 模块 BUILD SUCCESS（长期命令放 pwsh 后台 job，如 `pwsh-452216cb8fa04eac` / `pwsh-b1887d83e2cc4039`，完成通知带状态，避免 120s 工具超时）
- `JiraFeedbackEmailTemplateTest` 4/4，耗时约 57–58 s（启动真实 Spring 上下文，正常偏慢）
- 首次 `mvn` 曾出现无关噪声：`AmqpOutboundEndpoint` `MessageHandlingException`（既有环境噪声，与本改动无关）
- 一次失败用例 `JiraFeedbackEmailTemplateTest.escapesUserSuppliedValues:76 expected: <false> but was: <true>`，说明测试断言先于实现补齐——修正测试后通过
- 一次 `javac .encoding=UTF-8` 写成 `-encoding` 之外的形式导致 `ClassNotFoundException: /encoding=UTF-8`（Windows 下参数被当成类名）；正确写法是 `javac -encoding UTF-8 ...`

## 6. 交付状态

| 分支 | commit | 内容 |
|---|---|---|
| `feature/feedback-center-sql-admin` | `b7a2acd45` | 第二次提交：FreeMarker 版（4 文件 +271/−169）；该分支先前已有 `2e1bb59bf` 等 4 个提交 |
| `feature/feedback-mail-freemarker-admin` | `ae561d92a` | 基于最新 `admin-release` 重开分支（12 文件 +266/−235，含 8 个 html 删除） |

- **!591**（`feature/feedback-center-sql-admin` → `admin-release`，sha `2e1bb59bf`）用户已于 12:36 自行合并；其 8 个 `*.html` 片段现已落在 `admin-release` 上。
- **!592**（`feature/feedback-mail-freemarker-admin` → `admin-release`，sha `ae561d92a`）state=opened，未合并，描述 2518 字符。review 时看净效果：删 8 个 html + 加 1 个 `.ftlh`。
- 说明：用户长期口径是"只提 MR 不合并"（m00421），但 !591 被自行合并，说明该口径按单次指示执行，不可当默认。

## 7. 遗留（未做，等用户发话）

1. `adminFeedbackTimeline()` 一次请求串 5 条大 SQL（合计约 1394 行）。
2. `prepareAttachments` 在 `for (MultipartFile)` 里逐条 insert（`MAX_SCREENSHOTS = 5` 硬上限）。
3. 导出用 `OFFSET` 分页。

## 8. 可复用的经验

- 判断"是否真用了模板引擎"的检验标准：**控制流（if/loop）是否在模板里**。占位符替换只是拼串换皮。
- 换模板引擎先看它是否做 DOM 重序列化：Thymeleaf 会，FreeMarker/Velocity 不会。想字节级一致就选纯文本引擎。
- 已上线输出不可回归的改造，验收手段是影子实现 + 全维度矩阵双渲染，而不是靠 code review。
- 引入任何 `spring-boot-starter-*` 前，先数清受影响的技术栈：这里就是 `@Controller` 数量 + 有无视图名返回值。
- Windows 上 `.ftlh` 落盘会有 `LF will be replaced by CRLF` 警告（git autocrlf），不影响渲染，但 review diff 时注意别误判为改动；
