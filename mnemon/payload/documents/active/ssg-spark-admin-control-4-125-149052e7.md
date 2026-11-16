---
id: "149052e7-b222-43f1-80c1-3b72da3a4732"
title: "SSG Spark Admin Control 文字版交付：4 页汇报稿与 125 项功能清单定稿"
description: "承接 HTML 版功能清单，记录 Word/PDF 文字版的两次形态转折（领导汇报版 → 纯清单版）、125 项清单的 14 章骨架与项数、docx→PDF 生成与逐页渲染校验流程、Windows/PowerShell 相关环境坑。"
status: "active"
created_at: "2026-11-16T10:39:36.960Z"
updated_at: "2026-11-16T10:39:36.960Z"
content_hash: "442609fc34d762d29398f5f21bd85028abec89049a7072adedc6fca43b41eda8"
source_paths:
  - "outputs/admin-control-brief-leadership.docx"
  - "outputs/admin-control-feature-checklist.docx"
  - "outputs/admin-control-feature-list.html"
session_ids:
  - "mnemon-run-v1-54d3fa5a-52b2-441c-b620-8ada07144187"
memory_body_ids:
  []
---

# SSG Spark Admin Control「文字版功能清单」定稿：4 页汇报稿与 125 项清单

承接 2026-11-16 同日的代码级梳理任务。页面级功能细节、两层结构、口径与技术债见文档 `e5718bfd-fbfd-4213-aa85-72f8bcf99b19`（SSG Spark Admin Control / Admin settings 功能全景清单）；本文只记录该文档之后新增的**文字版（Word/PDF）交付**与**清单型定稿**，避免重复。

## 1. 需求演进（决定交付形态的三次转折）

1. 用户原话：「给我个文字版的呢 给领导看的 要清晰哈」→ 从网页版转为 Word（同时出 PDF）。
2. 那一版按管理层习惯写：一 结论先行 / 二 它解决的三个问题 / 三 能力全景 / 四 治理与可控性 / 五 当前边界 / 六 建议下一步 + 附一模块入口对照 + 附二管理动作速查。4 页。
3. 用户随后：「你直接发我呗」→ 交付不能只给文件链接，要把**全文直接贴在对话里**（复制即用）。
4. 用户纠正：「要的是功能清单 你这个看起来像是总结呀」→ 改成 14 章 125 项纯清单，去掉结论、背景、建议。

**可复用判断**：同一个后台，两种交付物目的不同——「给领导汇报」用结论先行 + 问题/价值/边界/建议；「功能清单」只按模块逐项列「功能点：能做什么」，不写结论与建议。用户说「清单」时不要塞总结章节。

## 2. 交付物（本轮新增，均在 `outputs/`）

- `outputs/admin-control-brief-leadership.docx`（43,792 bytes，4 页）+ 同名 `.pdf`（232,162 bytes）——领导汇报版，结论先行。
- `outputs/admin-control-feature-checklist.docx`（47,310 bytes，6 页）+ 同名 `.pdf`（224,043 bytes）——功能清单版，14 章 125 项。
- 渲染校验图：`.tmp/brief-p1..p4.png`（869×1228）、checklist 首页截图。

## 3. 125 项清单的章节骨架（章 / 项数）

入口与页签 6 · 使用分析 User Logs 12 · 反馈闭环 Issue & Request 12 · 质量巡检 Inspection Report 10 · 人员与名单 People 10 · 权限 Access control 16 · 审计日志 Audit log 6 · 通知 Notification settings 6 · Jira 集成 8 · 派单与分组 Page focal / Mode groups 6 · 调度中心 Scheduler Center 14 · 自动评审 Review rules 5 · 页签与显示 Admin Control tabs / Display 6 · 通用口径与已知限制 8 = **125 项**。

写法约定：每项一行「编号 + 功能点名 + 冒号 + 能做什么」，只写页面可操作事实；限制单独归到第十四章（五个扩展视图入口未显示、四个评审维度筛选未接线、HC 导入不可回滚、导出仅 xlsx 且单并发、缓存每作用域最多 40 key 等）。对外表述建议统一说成「规划中」。

## 4. 生成与校验流程（可复用）

1. 用 runtime python 的 `python-docx` 写正文与表格，`openpyxl`/`Pillow` 可用；该 site-packages 里**没有 pymupdf**，校验前需 `pip install pymupdf`（`fitz` 已弃用并告警，用 `import pymupdf`）。
2. docx → PDF 后，用 PyMuPDF 读 `page_count` 并把每页渲染成 PNG 逐页目检（869×1228 对应 A4 96dpi 观感），比只看页数可靠。
3. 排版要点：章节标题与首段必须同页（避免页底孤标题）；表格用固定列宽，否则长入口地址把表撑变形；打印/深色不适用于 docx，但 HTML 版那边用 `@media print` 隐藏侧栏与顶栏。
4. 校验结论以「实际页数 + 逐页截图」为准，不以脚本成功与否为准。

## 5. 环境坑（Windows）

- PowerShell 中 `python - <<'PY'` heredoc 会报 `ParserError: 缺少字符串终止符`，应改为写入临时 `.py` 文件再执行。
- 早期 `import fitz` 在该 Python 里曾缺包（`ModuleNotFoundError: No module named 'fitz'`），装包后 `fitz` 只作为兼容层存在。
- 输出中文 PDF 时终端打印会乱码（GBK 控制台），页数与像素尺寸是可信信号，正文以渲染图核对。
