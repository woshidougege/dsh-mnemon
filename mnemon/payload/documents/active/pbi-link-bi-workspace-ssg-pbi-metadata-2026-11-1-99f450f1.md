---
id: "99f450f1-6195-4a65-99a1-2499785b6331"
title: "PBI 页面链接 link_bi 改为 workspace 入口：跨表同步范围与三张必须不动的表（ssg-pbi-metadata，2026-11-19）"
description: "生产+测试双库把 catalog id=65 的页级 link_bi 从个人 app 入口换成 workspace 入口：必须同改 catalog/thumbnail/sheet_search 三表；kpi_map/rag_notes/km_sync_records 存报告级共享链接不能动；含个人账号无写权限、sheet_search 为重建式派生索引、审计不留痕等坑。"
status: "active"
created_at: "2026-11-19T10:57:41.210Z"
updated_at: "2026-11-19T10:57:41.210Z"
content_hash: "b533acd0da78efca9bdae1774c7ed925b24e8a536a020e72d56e31ea27bbedf0"
source_paths:
  - "app/services/pbi/metadata_write.py"
  - "app/services/pbi/metadata_identity.py"
  - "app/services/pbi/search_index.py"
  - "app/tasks/workflow_search_index.py"
  - "app/services/pbi/audit.py"
session_ids:
  - "mnemon-review-v1-d35b7b9b-f614-4234-b369-e9e8094e4821"
memory_body_ids:
  []
---

# PBI 页面链接（link_bi）从个人 app 入口改为 workspace 入口：跨表同步范围与三张"必须不动"的表

日期 2026-11-19。项目 ssg-pbi-metadata（Power BI 元数据 + 知识库服务）。本文记录一次生产/测试双库的 link_bi 数据修正的操作口径与发现的表耦合关系，供今后同类"链接发版"任务复用。

## 1. 需求与用户裁定范围

Metadatas Center 卡片上的「Open Power BI」原本指向**个人 app 入口**：

```
https://app.powerbi.com/groups/me/apps/0597ca37-8cc8-48ca-9dea-3d9199feda29/reports/3440ca2d-d7ab-4764-b483-fae8797be1f0/1aa0ff4c1a8e305a9506?experience=power-bi
```

用户拍板两项范围（原文选项）：**"生产和测试都改（推荐）"**、**"只改 Welcome Page（id=65）"**。

目标值为 workspace 入口（workspace id `0a3580b3-a39c-4ea9-9167-72949069e901`，tenant `5c7d0b28-bdf8-410c-aa93-4df372b16203`），即把 `groups/me/apps/<appId>` 段换成 `groups/<workspaceId>` 并补 `ctid`。section 段 `1aa0ff4c1a8e305a9506` 与 `?experience=power-bi` 保持不变。

环境：生产 `10.62.79.215`、测试 `10.62.79.148`（SSG GA Scorecard 报告各 34 页 / 1726 行 catalog）。

## 2. 关键发现：link_bi 在六张表里的分布，决定"改哪几张"

| 表 | 该页命中 | link_bi 形态 | 处置 |
|---|---|---|---|
| `agent.pbi_meta_sheet_catalog`（id=65） | 1 | 带 section 段的**页级**链接 | **改**（权威来源） |
| `agent.pbi_meta_thumbnail`（id=458） | 1 | 页级；注意列名是 `pbi_link`，**没有 `link_bi` 列** | **改** |
| `agent.pbi_sheet_search`（按 sheet_name） | 1 | 页级 | **改**（派生索引，见 §4） |
| `agent.pbi_meta_sheet_kpi_map` | 6 | **报告级** app 链接（无 section 段），Welcome Page / SSE Coverage / SSG Pipeline 三页共用同一值 | **不改** |
| `agent.pbi_meta_rag_notes` | 1（sheet_name='Welcome Page - Welcome Page'，name_bi='SSG GA Scorecard'；`source_url` 全空） | 报告级，三页共用 | **不改** |
| `agent.pbi_meta_km_sync_records` | 3 | 报告级，三页共用 | **不改** |

结论：child 表（kpi_map / rag_notes / km_sync_records）存的是**报告级**链接，改它们会同时改到另外两个页面的跳转与 KM 文档键，因此"只改 Welcome Page"就只能停在上述三表。这三表只影响 KM 同步页的跳转和文档键，不影响元数据卡片与「Open Power BI」。

## 3. 执行与校验

每套库各 3 行 UPDATE（catalog 按 id=65、thumbnail 按 id=458、sheet_search 按 `sheet_name='Welcome Page - Welcome Page'`）。改前先确认新 URL 未被任何行占用（唯一约束冲突）。

校验口径（两套库结果一致）：`catalog_updated=1 / thumbnail_updated=1 / sheet_search_updated=1`，新链接命中 1 行，旧链接残留 0 行，报告页数仍 34。改前旧值备份于 `.tmp/link_change_backup.json`，回滚即照抄旧值。

## 4. 四个必须记住的坑

1. **权限**：个人账号 `p_renxp4` 对 `pbi_meta_sheet_catalog` 只有 SELECT，首次 UPDATE 报 `psycopg2.errors.InsufficientPrivilege: permission denied for table pbi_meta_sheet_catalog`。写入必须用应用自己配置的 `a_appconnect`（读 `.env` / `.env.prd`），直接改数据不算改表结构。
2. **`pbi_sheet_search` 是派生索引，不是权威**：`app/tasks/workflow_search_index.py:132` 会**drop 后整表重建**，`app/services/pbi/search_index.py:372` 建向量索引。直接 UPDATE 只保证当下正确，长期正确的是 `pbi_meta_sheet_catalog`；若日后重建索引仍出现旧链接，说明生成索引的取数源（catalog/child）没同步。
3. **审计不留痕**：审计写入是应用层 SQLAlchemy 监听器（`app/services/pbi/audit.py`，由 `set_session_audit_context` 注入 actor），直连 SQL 修正不会写 `pbi_meta_audit_change_sets` / `_entity_versions`。需要留痕就得从「Edit Metadata」界面重存一次。
4. **应用层的关联维护已有人做，但只覆盖自然键**：`app/services/pbi/metadata_identity.py` 的 `sync_child_metadata_scope()` 在 catalog 的 `link_bi/name_bi/sheet_name` 变化时批量同步子表；`metadata_write.py:37` 的 `insert_only` 分支按 `link_bi` 去重返回 `{"inserted": False, "reason": "link_exists"}`。也就是说「改链接」应优先走接口，本次直连 SQL 是绕过这些钩子的一次性数据修正。

## 5. 结构参考

- `pbi_meta_km_sync_records`(`id, link_bi, rag_note_id, km_doc_id, km_task_id, sync_status, content_hash, metadata_hash, km_file_updated_at, km_characters, last_synced_at, last_error, created_at, updated_at`)
- `pbi_meta_thumbnail`(`id, pbi_name, sheet_name, pbi_link, thumbnail, content_type, created_at, updated_at, owner_system`)
- `pbi_sheet_search`(`link_bi, name_bi, sheet_name, category_1/2, aliases, keywords, metrics, dimensions, business_questions, example_questions, description, rag_notes, cjk_ngrams, search_vector, source_hash, indexed_at`)
