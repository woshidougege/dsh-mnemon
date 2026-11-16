---
id: "4105dfd5-d53c-49c4-b972-0c4138f5c8a0"
title: "录屏转写 → 单文件 HTML 会议纪要：OpenCV 抽帧 + base64 内嵌 + CDP 校验流程"
description: "把 71 分钟会议录屏做成单文件 HTML 纪要的可复用流程：本机无 ffmpeg 只能用 cv2；差分抽帧 th=0.1 + 最小间隔 25s + 章节补帧共 60 张；1000px webp(≈28KB) base64 内嵌得 2.4MB 单文件；lightbox/画面开关/搜索联动与 CDP 断言校验。关联文档 121d6b28-8c37-4caf-b9f6-8db4047c94a8。"
status: "active"
created_at: "2026-11-16T07:29:39.681Z"
updated_at: "2026-11-16T07:29:39.681Z"
content_hash: "ebd078d735306091f2d3489eacfb47b51d5a869387d19e1e1046e6dc89e2f6f4"
source_paths:
  - "outputs/service-portfolio-meeting-20261028.html"
  - ".tmp/sp-video/verify.mjs"
session_ids:
  - "mnemon-run-v1-c7380177-3e76-4754-a761-b781210e6eb4"
memory_body_ids:
  []
---

# 录屏转写 → 单文件 HTML 会议纪要：抽帧、内嵌与校验流程

工作区 `C:\Users\63588\lenovo\ssg-sales-ops`。产物：`outputs/service-portfolio-meeting-20261028.html`（单文件，2.4MB，含 60 张会议画面）。相关会议结论见文档 `121d6b28-8c37-4caf-b9f6-8db4047c94a8`（Lifecycle By Offering 复用 GPS/SUS），本文只记录「把录屏做成可交互页面」的流程与环境约束。

## 1. 环境约束（先确认这些，否则方案会跑不通）

- 本机**没有 ffmpeg / ffprobe**（PowerShell 报「术语 'ffmpeg' 不会被识别」），因此不能转 webm、不能用 ffprobe 读时长。
- Python **cv2 5.0.0 可用**；**没有 imageio_ffmpeg** 模块。视频处理只能走 OpenCV。
- 源视频：1920x1080、约 16.0 fps、4292.16 s（71 分 32 秒）、229 MB。转写文本另有一份（1,552 段识别片段 → 归并 372 条发言）。
- 模型本身看不到视频画面，只能读转写；画面里的表格/原型必须靠抽帧+人工描述补。用户明确要求「结合画面才能理解」。

## 2. 按画面变化抽帧（比等间隔有效得多）

- 以每 2 秒采样一次做相邻帧差分，共 2147 个样本；差分分位数 0.5/0.75/0.9/0.95/0.99 = 0.008 / 0.016 / 0.04 / 0.11 / 0.308 —— 大部分时间画面不动，长尾才是翻页/切窗口。
- 阈值扫描：th=0.1 → 124 候选，th=0.15 → 65，th=0.2 → 39，th=0.25 → 27，th=0.3 → 25。
- 最终参数：th=0.1 + **强制相邻帧最小间隔 25 s** → 48 帧；再在 13 个章节起点各补 1 帧（补入 s3/s5/s6/s10/s12/s13 等共 6 张），合计 **60 帧**。
- 单帧转 1000px webp 约 28.1 KB，60 张共 1.65 MB；base64 注入同一 HTML 后整文件 2.28–2.4 MB，仍是单文件，可直接外发。
- 生成物落在 `.tmp/sp-video/`：`scan.py`（抽帧）、`sheets/sheet_01..05.jpg`（每张 12 帧的预览拼图）、`verify.mjs`（CDP 校验）、`shot1/shot2.png`、`v_top.png`、`v_mid.png`。

## 3. 页面结构与交互（已验证）

- 三栏：左为逐句发言流（`#log`，按 13 个章节插入 `.divider`），右为章节总结卡（`#sum`），滚左联动右侧卡片高亮；卡片点击反向滚动左侧（用 lockUntil 约 900ms 防止回环）。
- 画面按时间戳插在相邻两条发言之间，图注统一格式：`38:22 会议画面 | Lifecycle 页（EOL 判定卡片）+ 幻灯片 SSG PN Inventory by HBB · PCSD`。
- 点图全屏 lightbox（`#lightbox.on`，`inset:0`、`rgba(18,10,32,.9)`、Esc/点击关闭）；右上「会议画面：开/关」按钮切纯文字模式；搜索命中时强制隐藏所有 figure，只留命中的发言行。
- 搜索高亮用正则转义 + `<mark>`，注意用 `p.dataset.raw` 保存原文再渲染，避免二次转义。

## 4. 自动校验（无浏览器自动化框架时的做法）

- 用 Chrome `--remote-debugging-port` 起无头实例，Node 里走 CDP WebSocket：`Page.navigate` → `Page.captureScreenshot` 存 PNG → `Runtime.evaluate` 注入断言脚本，返回 `{figs, btn, first, offInfo, lbOn, lbOff, searchInfo, align}`。
- 通过标准：`figs=60`、`btn=「会议画面：开」`、关掉后 `visible=0` 且 btn 变「关」、searchInfo 中 `rows=372` 而 `rowsVisible=2`（搜索 GPS）、lightbox 点击后 `lbOn` 为 true、`frames=60`。
- 坑：`verify.mjs` 里 `shot()` 用了未声明的 `dir`（ReferenceError: dir is not defined，exit 1）——脚本内需显式定义目录常量；`rg` 搜不存在的 worktree 路径会 `exit 2`（os error 2），不要据此判断文件缺失。

## 5. 尚未实现（用户需拍板）

- 动图：可行但体积大，10 s 1080p 抽 8fps 约 2–4 MB/段，需用户指定时间段。
- 原视频片段内嵌：不可行（无 ffmpeg，229 MB 也塞不进单文件）。
- 替代方案：给每张图加「▶ 原视频 38:22」链接跳本机 mp4 指定时刻，轻量但仅本机有效、外发失效。

## 6. 顺带从画面读出的业务数据（可作后端建表参考）

- HBB V3.3 Change Request 原型全屏（54:26 帧）：共 **128 条**（Draft 12 / Under Review 42 / Pending 28 / Approved 46 / Not Approved 8）。
- 4 步时间轴：Submission(Apr 1–May 31) → Pre-Assessment(Jun–Jul) → Leadership Review(Aug) → Decision & Release(Sep)。
- Change Type 分布：Add 29 / Rename 27 / Move across HBB 19 / Move within HBB 19 / Inactivate 17 / Change Offering Bucket 15 / Remapping PNs 11；列表列头 Request ID / Title / Change Type / HBB L1 / Requester / Team / Submitted / Status / Actions。

## 7. 参与人账号名（按会议显示的五个账号，用户要求统一用会议名）

- Shuwen SW8 Liu（业务方）· Devin CZ1 Zhang（原型）· Xiangpeng XP4 Ren（后端）· Xu Yang（前端）· Shinee Shan5 Gao。左侧逐句记录保留原口头称呼（舒文/相鹏/杨旭）不动，因为原话如此。
