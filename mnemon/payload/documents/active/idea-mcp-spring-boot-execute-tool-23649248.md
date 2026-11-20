---
id: "23649248-42bc-420e-8949-46b687fda13f"
title: "用 IDEA MCP 启动 Spring Boot 服务：execute_tool 调用程序与失败模式"
description: "IntelliJ IDEA MCP 连接（srv-e3b0c44298fc1c14，唯一工具 execute_tool）的 streamable-http 调用程序、--paramName value 参数格式与三条错误对照、start_run_configuration 实测结果，以及 IDEA 索引假报错（LinkedHashMap 无法解析、差 19 行）的证据链与修复步骤；附 MR 待决事项。"
status: "active"
created_at: "2026-11-20T20:52:52.859Z"
updated_at: "2026-11-20T20:52:52.859Z"
content_hash: "2d79c4889620a35f2e211553815dd1a5b008d0128e54ca46c613073235d35958"
source_paths:
  - "smartop-service/admin/src/main/java/com/lenovo/admin/chatbi/service/ChatbiEventReportService.java"
session_ids:
  - "mnemon-review-v1-d44d87a8-dc61-4b5f-927f-fbf20a9e434a"
memory_body_ids:
  []
---

# 通过 IDEA MCP 在 IntelliJ 里启动/管理 Spring Boot 服务（操作程序 + 排障）

状态：**已验证可用**（2026-11-21 在 `ssg-sales-ops` 工作区实测，成功从 IDEA 内启动 `AdminApplication` 并跑通接口）。
用户明确要求服务在 **IDEA 里启动**（而不是命令行 `java -jar`），因此这条路径是后续默认做法。
性能改动本身与实测数字见关联文档 `f4ea3686-cfb6-4cd9-9e97-913d85d292bc`；SQL 归因见 `664308d6-d2cc-47b5-a77e-c544492e5504`。

## 1. 连接事实

| 项 | 值 |
|---|---|
| serverKey / 名称 | `srv-e3b0c44298fc1c14` / `idea`（custom 连接器） |
| serverInfo | `IntelliJ IDEA MCP Server 2026.2.3`，protocolVersion `2025-03-26` |
| transport | `streamable-http`，url `http://127.0.0.1:<port>/stream`，authMode `none` |
| 暴露的工具 | **只有 1 个**：`execute_tool`（Universal tool executor，动态转发到具体 IDE 工具） |
| 状态 | `state=healthy`（`/helpers/.../health` 返回 ok，tools_list 1 个工具） |

注意点：
- `execute_tool` **不会**注册进会话工具表（session 注入模式未开启，`/tools` 查询返回"找到 0 个可按会话激活的精确工具"）。所以不能当普通 MCP 工具直接调用，要走 HTTP 端点。
- `<port>` 是 IDEA 每次启动分配的，会变；从连接器配置 `/config`（安全可编辑配置接口）读取当前值，不要硬编码。
- 会话模式若强行启用会报错：`连接 "idea" 未设为"按会话启用"`。

## 2. 调用程序（已验证）

MCP streamable-http 三步：`initialize` → `notifications/initialized` → `tools/call`。
`initialize` 返回 `Mcp-Session-Id`，后续请求必须带回；**跨进程/重连复用旧 session id 会报 `Streamable HTTP session not found`**，要重新 initialize 拿新 id。

`execute_tool` 入参：`{ command, projectPath }`（`projectPath` 强烈建议传，减少歧义调用）。

`command` 是**字符串**，参数格式必须用 `--paramName value`（kebab/驼峰均可，与工具 schema 一致）：

```
start_run_configuration --configurationName AdminApplication --mode run
list_run_configurations --projectPath C:/Users/63588/lenovo/ssg-sales-ops
get_console_output
```

## 3. 失败模式对照（改命令时先看这张表）

| 错误原文 | 原因 | 正确写法 |
|---|---|---|
| `Invalid argument format: '{name:AdminApplication,mode:run}'. Expected '--paramName value' format.` | 用了 JSON/花括号风格传参 | `--configurationName AdminApplication --mode run` |
| `Missing required parameters: configurationName` | 参数名写成 `name` | 必须叫 `configurationName` |
| `Streamable HTTP session not found` | 复用了过期/别处的 session id | 重新 `initialize` |

`--mode` 只接受工具真正支持的值（`run` / `debug` / `jrebel` / `jrebel-debug`）；`list_run_configurations` 返回的 `modes` 字段是权威依据，不支持的 mode 会显式失败而不会静默降级。

## 4. 实测结果（本工作区）

- 启动前 `list_run_configurations` 显示 4 个 Spring Boot 配置全部 `running:false`：`AdminApplication (assistant verify)`、`AdminApplication Local Final`、`AuthApplication`、`AdminApplication`（均支持 run/debug/jrebel/jrebel-debug）。
- `start_run_configuration --configurationName AdminApplication --mode run` 返回 `Run configuration 'AdminApplication' started in run mode.`
- 进程：`java.exe` **pid 14540**，启动时间 04:44:58，JVM 为 `C:\Program Files\Java\jdk-17.0.19`；8082 LISTEN。
- 接口实测：冷请求 4750ms / 200 / 8850 字节；缓存命中 502 / 250 / 243ms；`/menu/all` 基线 485ms；经 vite 代理 625ms / 200。
- 结论：缓存改动在 IDEA 启动的实例里同样生效（命中耗时已贴平框架基线）。

## 5. 排障：IDEA 里"无法解析 `LinkedHashMap`"是假报错

现象：IDEA 编辑器中 JDK 自带的 `java.util.LinkedHashMap` 报红"无法解析"，但 `mvn` 全量编译 BUILD SUCCESS。

证据链：
- 磁盘文件 157354 字节、2972 行、BOM `70-61-63`、CRLF 2972 行；`import java.util.LinkedHashMap;` 在 **第 25 行**（未改动）。
- 磁盘上 `statusCounts = joinOr(statusTask, new LinkedHashMap<>(...` 在 **1253 行**；IDEA 截图里同一行显示 **1234 行**——差 **19 行**，正好是本次新增的缓存代码块 ⇒ **IDEA 打开的是改动前的旧快照，未从磁盘重新加载**。
- `build.log` 里有 `ClosedStorageException: storage is already closed ... compile-server\...\timestamps\data_i` ⇒ 编译缓存损坏。
- IDEA 能编译、能起 JVM、能跑通接口 ⇒ 索引显示问题，不影响 build，**不需要改任何代码**。

修复顺序：`File → Reload All from Disk`（弹窗选 "Load File System Version"）→ `File → Invalidate Caches`，勾选 "Clear file system cache and Local History" → `Invalidate and Restart`。

## 6. 待决策（未完成，需用户拍板）

性能改动仍留在工作区未提交（`ChatbiEventReportService.java`，+48/−2）。当前分支 `fix/chatbi-review-success-accuracy-admin`，其上 MR **!599** 有 4 个文件。

- 方案 A（推荐）：从 `origin/admin-release` 拉新分支 `fix/chatbi-report-perf-admin`，提交信息 `perf(chatbi-report): 汇总查询并行度提到 18，并给同一筛选条件加 60 秒结果缓存`。
  理由：!599 是"准确率口径修正"（SQL/业务逻辑），本次是"性能"（Java 层），验证方式、回滚风险、reviewer 关注点都不同，混在一个 MR 里难以分开评估。
- 方案 A + 顺带修两个既有 bug（`Parameter 'chatbiMode' not found`、`missing FROM-clause entry for table "sn_profile"`）。
- 方案 B：追加到 !599（只审一次，但性能改动难单独回滚）。

用户此前定过规矩：**提交与合并类操作必须先得到用户确认**再动手。
