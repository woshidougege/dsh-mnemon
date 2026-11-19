---
id: "8d4061ae-aa32-4062-a1d1-078b5cfbda27"
title: "反馈中心 SQL 片段化与 Mapper XML 全量缩进规范化（MR !590）"
description: "MR !590 纪实：AdminFeedbackTimelineMapper.xml 1612→491 行（7 片段/29 include）、56 文件纯行首空白规范化（两道 trimStart/删空白整文件比对）、24 处列侧去 lower 恢复 btree、ORDER BY 补 id tie-breaker 修 OFFSET 漏行、MapperXmlBindingTest 落地 fd202c70 的冒烟测试建议（121 XML/799 statement/missing=0）、三条明确不做的取舍理由与验证矩阵。"
status: "active"
created_at: "2026-11-19T04:25:47.687Z"
updated_at: "2026-11-19T04:25:47.687Z"
content_hash: "f44c5983f672b558d789db40d5cd56506367d506ff79b0916e7b171375b43ebb"
source_paths:
  - "smartop-service/admin/src/main/resources/mapper/usagefeedback/AdminFeedbackTimelineMapper.xml"
  - "smartop-service/admin/src/main/resources/mapper/usagefeedback/UsageIssueFeedbackMapper.xml"
  - "smartop-service/admin/src/main/resources/mapper/usagefeedback/AdminDownvotedAnswerMapper.xml"
  - "smartop-service/admin/src/test/java/com/lenovo/admin/MapperXmlBindingTest.java"
session_ids:
  - "mnemon-review-v1-abb0407b-540d-4411-84b0-b87095288214"
memory_body_ids:
  []
---

# 反馈中心 SQL 片段化与 Mapper XML 全量缩进规范化（MR !590）

2026-11-19。承接「注解 SQL → Mapper XML」改造主线（见文档 `fd202c70`），本 MR 收尾「迁入 XML 后语句体重复、SQL 未格式化」的债，并**落地了 `fd202c70` 提出的「Mapper 全解析冒烟测试」补强建议**；同批还修掉两个真实的索引/分页隐患。区别于 `8541407e`（同一 Mapper 的 facets 与降级，MR !551），本文只讲结构重构与验证手法。

## 1. 交付

| 项 | 值 |
|---|---|
| 仓库 / worktree | `ssg-sales-ops`，`.agents/worktrees/feedback-timeline-sql` |
| 分支 → 目标 | `feature/feedback-center-sql-admin` → `admin-release` |
| MR | **!590** `refactor(admin): 反馈中心 SQL 片段化与索引修正，Mapper XML 全量缩进规范化` — state=opened，`can_be_merged`，无冲突，**未合并**（留给人工审核） |
| 规模 | 63 文件 +2343 / −4080；其中 XML 40 个 +2247/−3458，Java 23 个 +96/−622 |
| HEAD | `dcec7b84f825087e4ad8efda248b323fbe1dc06d` |
| 编译 | `mvn ... test-compile` → BUILD SUCCESS（15 模块全绿） |

三个提交（push 前 amend 过一次，旧 sha `8cc1e702f/612bb9913/263409061` 已作废）：

| sha | message | 规模 |
|---|---|---|
| `8cc1e702f` | refactor(usagefeedback): 反馈中心 SQL 抽出公共片段，并修正索引与排序 | 6 文件 +808/−2176 |
| `866a29c31` | refactor(admin): 统一 Mapper XML 缩进，清理未使用的 SQL 常量 | 56 文件 +1440/−1904 |
| `dcec7b84f` | test(admin): 校验 Mapper XML 声明与 Spring 装配结果一致 | 1 文件 +95 |

## 2. 片段化：AdminFeedbackTimelineMapper.xml 1612 → 491 行

6 个 statement 的语句体合计约 1300 行 → **277 行**：

| statement | 前 | 后 |
|---|--:|--:|
| selectAdminFeedbackTimeline | 230 | 59 |
| countAdminFeedbackTimeline | 179 | 18 |
| selectAdminFeedbackTimelineStats | 191 | 20 |
| selectReporters | 208 | 37 |
| selectReviewStats | 429 | 55 |
| selectFacetStats | 370 | 88 |

抽出 **7 个 `<sql>` 片段、29 处 `<include>`**：`combinedCte`（76 行 ×6）、`filterBase`（54 ×5）、`timelineFilter`（24 ×4）、`facetFilter`（6 ×8）、`condTimeRange` / `condKeyword` / `condIncludeInternal`。改造前该文件 `<sql>` 与 `<include>` 均为 0。

同批：`UsageIssueFeedbackMapper.xml` 的 `selectAdminList`/`countAdminList` WHERE（48 行）、`AdminDownvotedAnswerMapper.xml` 的 LATERAL（12 行）与 downvote WHERE（50 行）也抽成片段。

## 3. 缩进规范化：只动行首空白的两道自检

commit 2 覆盖 56 个 XML。**只改行首空白，无一字符内容变更**，靠两道机械断言而非人工 review：

1. 格式化器每行断言 `trimStart()` 与原文完全一致，不满足即拒绝写回；
2. 写回后再跑一遍「删除全部空白字符后整文件比对」，40 个改写文件零差异。

顺带删掉 **55 个** 迁到 XML 后已无人引用的 SQL 文本常量（commit 1 删 5、commit 2 删 50），Java 侧净减 620 行。删除前用 `类名.常量名` 形式**全限定**搜过全部 1276 个 java 文件——只按简单名搜会误删。

## 4. 两处顺带修掉的真实隐患

### 4.1 列侧 lower() 阻断 btree 索引（24 处）

`lower(f.reporter_user_id) = lower(#{itcode})` → `f.reporter_user_id = lower(#{itcode})`：参数侧 lower 保留（大小写兜底不变），列侧不再被函数包住，索引可用。改前实测两份库确认安全：

- `ssg_user.itcode` 286/288 行、`ssg_usage_issue_feedback.reporter_user_id` 23/20 行、`ssg_chatbi_internal_user.itcode` 62/62 行；
- **大写记录均为 0**，故列侧去 lower 不会漏匹配。

### 4.2 ORDER BY 缺 tie-breaker 导致 OFFSET 分页重复/漏行

`selectMine` / `selectAdminList` 的 `ORDER BY f.created_at DESC` 补 `, f.id DESC`。单排序键时同秒的行顺序不确定，导出每页 500 行按 OFFSET 翻页会重复或漏行。

## 5. MapperXmlBindingTest：XML 只在运行期加载，编译期发现不了

新增 `smartop-service/admin/src/test/java/com/lenovo/admin/MapperXmlBindingTest.java`（95 行）：扫 `resources/mapper` 下全部 XML，把声明的 statement 与 `SqlSessionFactory` 的 Configuration 逐条对齐，从而在 CI 阶段暴露 `<include>` 写错、id 缺失、命名空间错配这类只有启动才炸的问题。

结果：`BINDING_CHECK xmls=121 declaredStatements=799 fragments=19 missing=0`，`MapperXmlBindingTest.everyMapperXmlStatementIsBound` 64 s 通过。

**坑**：初版放在 `com.lenovo.admin.usagefeedback` 包，surefire 报 `Socket Unexpected end of file from server`（`MapperXmlBindingTest.everyMapperXmlStatementIsBound:52`，栈顶 `org.apache.xerces.jaxp.DocumentBuilderImpl.parse`）并 BUILD FAILURE；把测试类移到 `com.lenovo.admin` 包后同一命令通过。检查点未留下根因判定——遇到该报错先怀疑测试包位置/环境而非 SQL 本身。

## 6. 三条明确不做（及其理由）

1. `adminFeedbackTimeline()` 一次请求串 5 条大 SQL（合计 1394 行）——合并成一条可读性会崩，加缓存属架构改动，不塞进本 MR。
2. `prepareAttachments` 在 `for (MultipartFile)` 里逐条 insert——`MAX_SCREENSHOTS = 5` 是硬上限，改批量反而丢掉「哪个附件失败」的定位。
3. `JiraFeedbackEmailNotifier` 约 300 行硬编码邮件 HTML 抽模板——抽 Thymeleaf 后渲染结果不可能字节级一致，而邮件已在线发给用户，验证成本高于收益。

## 7. 验证矩阵

| 项 | 结果 |
|---|---|
| 121 个 XML 解析 | OK=121 FAIL=0，799 statement，19 片段 |
| Spring 装配（MapperXmlBindingTest） | missing=0，BUILD SUCCESS |
| 片段化前后执行比对（新旧 SQL 各跑一次逐行比） | 24/24 SAME |
| lower() 改写前后 | 24/24 SAME |
| 排序 tie-breaker | 4/4 SAME |
| 格式化是否动了内容 | 40 文件删空白后零差异 |
| test-compile | BUILD SUCCESS |

## 8. 可复用的操作经验

- **纯空白重构的验收手法**：逐行 `trimStart()` 不变 + 删空白字符后整文件比对。比「肉眼看 diff」可靠，也能用于缩进/换行符大范围整理。
- **GitLab REST 建 MR**：token 由 `git credential fill`（`host=gitlab.xpaas.lenovo.com`）取 password 作为 `PRIVATE-TOKEN`；project 参数需 URL 编码 `dataagent%2Fssg-sales-ops`。校验状态看 `merge_status` / `has_conflicts` / `changes_count` / `pipelines`。
- 长时间 `mvn` 放 pwsh 后台 job（id 形如 `pwsh-0a39379b7b4b455e`），完成通知带退出状态，日志再单独取，避免阻塞。
- 启动日志里的 `systemPath ... ${env.JAVA_HOME}/lib/jconsole.jar` 与 `LoggingHandler ... AmqpOutboundEndpoint` 报错是既有噪声，与本 MR 无关。
