---
id: "23a6f590-4cd8-474b-8318-1d44ecbff890"
title: "DSH 本地项目档案导入 OpenViking 记忆空间：迁移流程、持久性分层与三类记忆职责"
description: "131 篇本地 Documents 全量写入 viking://user/default/memories/projects 的过程与 http404 经验；OpenViking 文件/向量/账号三层持久性判断（S3 + disable_batch_delete + reindex 派生索引 + 账号级联删除唯一入口）；保留本地的三条理由（DSH 档案层只读本地、可 diff 可 git、成本近零）；真正风险是双份分叉与 content_hash 增量同步设想；运行时记忆/项目档案/记忆空间三层职责对照表。"
status: "active"
created_at: "2026-11-19T03:23:56.824Z"
updated_at: "2026-11-19T03:23:56.824Z"
content_hash: "3695b1fa4dc59d81e57bcfaea515d7fb41115c4d556cca9ca71a319e4eedeccf"
source_paths:
  []
session_ids:
  - "mnemon-review-v1-59fae499-0006-4c6e-9cf5-5855945829d9"
memory_body_ids:
  []
---

# DSH 本地档案与 OpenViking 记忆空间：批量迁移、持久性与分层职责

2026-11-19 完成。用户诉求链条：把本地项目档案（Documents）全量搬进 OpenViking（明确「不需要脱敏」）→ 追问「本地还建议存吗」→ 要求解释 dsh-mnemon 三类记忆的区别 → 纠正「OpenViking 那套是测试环境 + 共享 S3，不会被清空」。

## 1. 迁移结果（已完成）

- 来源：`C:\Users\63588\.mnemon\documents\active`，131 篇 Markdown，合计 936.7 KB。
- 目标：`viking://user/default/memories/projects/`，文件名与 frontmatter 原样保留。
- 结果：`ok=131 fail=0`，耗时 64.7 秒；`fs/ls projects` 计数 131，首尾为 `0-mr-1039-1040-5bd335b0.md` … `wrong-jira-assignee-routing-2a95b6da.md`；抽查 `hw-kpi-datasetindex-cf1cbe31.md` 回读 3644 字符，与本地一致。
- 一次性脚本：`.tmp\import_all.py`（位于工作目录 `C:\Users\63588\lenovo\ssg-spark-openviking`，改目标目录即可复用）。
- 端到端召回已验证：`mnemon_recall` 命中 `openviking-dsh-account-user-admin-366ff5ad.md` 与 `hw-kpi-v2-bookings-qty-isg-handoff-b881d9ec.md`，返回 `memoryBodyName=default`、`memoryProviderId=openviking`。
- 首轮批量写入曾出现 `total 131, ok=3 fail=128`（大量 http404 NOT_FOUND）；改为逐条写入后 131/131 全绿。经验：批量并发写 `content/write` 不可靠，逐文件写 + 计数校验更稳。

## 2. 持久性判断（修正过一次口径）

最初答复里说「测试环境 + 共享 S3，随时可能被清空」，用户当场纠正。核对 `ov.conf.test` 后确认原判断失准，正确的分层结论是：

| 层 | 会不会丢 | 依据 |
|---|---|---|
| 文件本身 | 不会 | `storage.agfs.backend = "s3"`，bucket `ssg-spark-openviking`，prefix `openviking`，且 `disable_batch_delete = true`；服务重启/发版/容器重建都不影响 |
| 向量索引 | 可能暂时不可用，但可重建 | 无独立向量库后端，`workspace = ./data/openviking`；`run.py` 启动时执行 `reindex(vectors_only)`，说明索引是从文件派生的可重建数据；重建依赖 aiverse 的 `text-embedding-3-large`（dim 3072）网关，网关/配额故障时检索降级，但文件仍可 browse/read |
| 账号 | 仅主动删除才没 | `DELETE /api/v1/admin/accounts/{id}` 是唯一会级联清理 S3 对象与向量记录的入口，不可逆 |

## 3. 为什么仍然建议保留本地（结论不变，理由换了）

1. **DSH 的项目档案层只读本地目录**：`mnemon_document_search` 检索的是 `C:\Users\63588\.mnemon\documents\active` 下文件的 frontmatter 与正文，本地删空则该层直接为空，只剩记忆空间那套检索，返回形态与元数据完全不同。
2. **本地明文可编辑、可 diff、能进 git**：改单篇、看历史差异、批量重构都在本地做；在 OpenViking 侧得走 `content/write` API。
3. **936 KB 成本约等于零**，删除无收益、只有风险。

定位：**本地是事实源（source of truth），OpenViking 是副本 + 语义召回层。**

## 4. 真正的风险是分叉，不是丢失

两份内容不会自动同步：本地更新后 OpenViking 仍是旧版，两边静默分叉。可选的缓解方案（尚未实施）：给 `.tmp\import_all.py` 增加增量模式，比对 frontmatter 的 `content_hash`，只推送新增与改动文件，手动触发而非定时自动跑。自动同步的顾虑是把测试环境当长期存储用，需用户取舍。

## 5. dsh-mnemon 三类记忆的职责分工

| | 运行时记忆 | 项目档案 Documents | 记忆空间 Memory Space |
|---|---|---|---|
| 内容形态 | 一句话的偏好/红线/约定 | 几百到几千行文档 | 一条条独立事实、决策、结论 |
| 进上下文方式 | 每轮完整注入（USER.md / MEMORY.md） | 先检索标题描述，命中才读正文 | 语义检索，命中几条给几条 |
| 存放位置 | `~/.dsh` 侧 USER.md、MEMORY.md | `C:\Users\63588\.mnemon\documents\active` | Provider（当前为 OpenViking `default` 空间） |
| 写入时机 | 用户说「以后都这样做」「不要 X」 | 一篇完整的调查/设计/流程 | 一次对话得出的可复用结论 |
| 当前规模 | 十几条常驻 | 131 篇按需读 | 131 篇（本次导入的 projects 副本） |

三个问题对应三层：运行时记忆答「你是谁、有哪些硬要求」；项目档案答「这个项目某件事具体怎么做」；记忆空间答「我们以前得出过什么结论」。

插件侧配置可见 `cordis.patch.yml`：`mnemon` 策略含 `include:mnemon-strategy-auto-capture`（本机自动捕获源为 `source:include:mnemon-source-memory-spaces`）、`include:mnemon-strategy-light-context`、`include:mnemon-strategy-default-three-tier`，源为 `mnemon-source-runtime` / `-documents` / `-memory-spaces` / `-skills`。界面入口：「插件 → 可组合记忆」看策略开关，「记忆系统」看状态/运行时记忆/项目档案/记忆空间四块。

## 6. 遗留

- 131 篇里有 23 篇命中凭据字样（`password|secret|jasypt|a_appconnect|p_renxp4`），例如 `ssg-sales-ops-runbook-48f9fd58.md`、`bookings-trino-pg-ssg-sales-ops-00d44b32.md`、`spark-4-3-redis-7de0588d.md`、`admin-spring-boot-3-mybatis-mybatis-plus-tk-myba-8bbfce97.md`。用户明确不脱敏，已原样写入，内容同时存在于本地磁盘、S3（团队共享桶）与向量库片段中。
- 增量同步脚本尚未实现，等用户决定是否要做。
- 相关文档：`366ff5ad-c3f7-48f4-82ba-92f9bb9eb959`（OpenViking 多租户隔离与 DSH 记忆后端接入结论）。
