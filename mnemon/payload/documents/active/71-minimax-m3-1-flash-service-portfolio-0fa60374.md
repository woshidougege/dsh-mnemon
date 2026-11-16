---
id: "0fa60374-eb4e-4f65-b830-6fdac81c3244"
title: "71 分钟录屏原生视频理解：minimax-m3.1-flash 分段转写流程与 Service Portfolio 关键发现"
description: "把 71.5 分钟会议录屏整段送进 minimax-m3.1-flash 的可复用流程（ffmpeg 切片 + base64 data URI + 3 并发，16MB body 上限与 crf 体积实测，imageio-ffmpeg 替代本机缺失 ffmpeg），以及 Service Portfolio 业务发现：季度轮播配置表四项、Change Request 状态/优先级/字段与原型差异、8 张 KPI 口径核对。关联 4105dfd5 与 121d6b28。"
status: "active"
created_at: "2026-11-16T09:05:40.146Z"
updated_at: "2026-11-16T09:05:40.146Z"
content_hash: "0a8e017a0b0d9fc1e27451348dade73b233a8392a09b6b731bcd848f6c3a721c"
source_paths:
  - "outputs/service-portfolio-video-analysis-20261028.md"
session_ids:
  - "mnemon-run-v1-f31d7535-5a83-4406-b031-e8138c49b6b4"
memory_body_ids:
  []
---

# 71 分钟会议录屏「原生视频」理解：minimax-m3.1-flash 分段转写流程与 Service Portfolio 关键发现

工作区 `C:\Users\63588\lenovo\ssg-sales-ops`。源视频 `SPARK_Internal__Service_Portfolio_Design-20261028_143257-会议记录.mp4`（229.45 MB / 1920x1080 / 16 fps / **71 分 32 秒**，Teams 录制 2026-10-28 06:32 UTC）。
产物：`outputs/service-portfolio-video-analysis-20261028.md`（8076 行 / 约 145K 字符：目录 + 10 项关键发现 + 15 段逐段转写）。
本文只记录「怎么把长录屏整段送进多模态模型」的可复用流程，以及与既有文档 `4105dfd5-d53c-49c4-b972-0c4138f5c8a0`（OpenCV 抽帧路线）、`121d6b28-8c37-4caf-b9f6-8db4047c94a8`（By Offering 复用 GPS/SUS）的业务交集。

## 1. 方法：切片 + 原生视频 base64 送模型（不走抽帧）

- 用户指定模型 **minimax-m3.1-flash**，并要求「整段视频都送进去」而非抽帧描述。
- 15 段 × 5 分钟切片，`ffmpeg -c:v libx264 -crf 24` 重编码后每段 4–7.6 MB。
- 传输方式是 `video_url` + **base64 data URI**，模型直接解码视频画面 —— 这是与旧方案（OpenCV 抽帧 + 人工描述，见 `4105dfd5-d53c-49c4-b972-0c4138f5c8a0`）的本质差别。
- 3 并发跑完，总耗时约 8 分钟；单段消耗 **104,138 prompt tokens**（5 分钟），末段 32,170；全程约 **149 万 tokens**。
- 模型自述局限：1080p 共享屏小字读不全，输出中已用「疑似/约」标注；**视频无字幕、未做音轨转写**，故口头讨论内容不可得，报告只描述画面。

## 2. 环境约束与踩坑（会直接决定方案可行性）

- **网关单请求 body 上限 16 MB**（`16777216` bytes）。原始切片 19.5 MB 直接被 413 拒绝，原文 `payload too large (20443600 bytes > 16777216 limit)`。必须重编码压到 4–7 MB。
- crf 档位与体积（同一段 5 分钟素材实测）：crf 26 → 3.09 MB（b64 4.12 MB）；crf 30 → 2.26 MB（b64 3.01 MB）；crf 34 → 1.67 MB（b64 2.23 MB）。注意 **base64 会放大约 1.33 倍**，要按 b64 后体积对 16 MB 上限做预算。
- 本机当时仍无 ffmpeg / ffprobe（与 `4105dfd5` 记录的约束一致），解法是 `pip install imageio-ffmpeg` 取到约 **105 MB** 的静态二进制替代。
- 编码进程输出形如 `enc segments: 13` / `s09 ok 80.6s pt=104138` / `ALL DONE`，可用 `pt=` 直接核对每段 token 消耗。
- 逐段调用若拼 code 字符串出现 `Unexpected character '→'`（异常信息取自 run_code 失败回执），是代码里混入中文箭头导致解析失败，不是数据问题。

## 3. 业务发现（相对需求 PPT / 原型的增量）

### 3.1 季度轮播是「配置表驱动」，会议室里当场把表敲完

| Projects | Display From | Display To | Display Quarter |
|---|---|---|---|
| HBB New Version | 4/1 | 6/30 | Q1 |
| PCSD PN Lifecycle | 7/1 | 9/30 | Q2 |
| ISG PN Lifecycle | 10/1 | 12/31 | Q3 |
| ISG Lifecycle Request | 10/1 | 12/31 | Q4 |

要点：第 4 行项目名是 **ISG Lifecycle Request**，不是 PPT 里写的「HBB Change Request」；它和 ISG PN Lifecycle **日期完全相同**，只靠 Quarter 区分先后 —— 说明排序语义是季度而非日期区间。表格字段即 `projects / display_from / display_to / display_quarter`（PowerPoint 里逐字录入，录屏 00:00–02:38）。

### 3.2 Change Request 的状态、优先级、表单字段都与原型不一致

| 维度 | 原型（PPT） | 录屏里的真实系统 |
|---|---|---|
| 状态 | Draft / Under Review / Approved / Rejected（4） | Draft 12 / Under Review 42 / **Pending 38** / Approved 46 / **Not Approved 8**（5） |
| 优先级 | Medium / High / Low（3 档） | **Low / Normal / High / Critical**（4 档；柱顶数字见 42 / 98 / 30） |
| 流程 | — | Submission → Pre Assessment → Leadership Review → Decision & Release；另一版页面为 Request → Pre Assessment → Customer Review → Outcome Review |
| 表单字段 | 10 个 | **12 个**，多出 HBR Title / HBR Type / **Impact Level（Medium / Cross HBR）** / Impact Date / Target Date |

`Impact Level = Cross HBR` 是跨 HBB 变更，走另一条审批路径，当前表结构里没有这个概念。

### 3.3 8 张 KPI 的口径核对通过

录屏看板是 4 组 × 2 张：`My Events 1.6K / 288K`、`Portfolio Management 30+ / 12K`、`General Inquiry 94% / 9+`、`Lifecycle 1'860 / 12K`，与实现侧的 1.6K / 278K / 108 / 12K+ / 94% / 9+ / 1.5WD / 12K 可逐一对上。

其他可确认细节：`Requests by Change Type` 环形图中心 **428 Total**（另一页面为 328 Total，属于不同版本/口径）；`The R2 Transition Project has been completed.`；`Showing 1-10 of 23 requests`；页面体系含 My Assets / Portfolio Management / Demand Projects / Lifecycle 四个主 Tab，以及 `SSG Offering Hierarchy & Service Portfolio`、`ISG Service PN Reduction Plan & Execution Tracking`、`HBR Change Request Management` 三套看板。

### 3.4 与既有文档可能冲突，需核对

`4105dfd5-d53c-49c4-b972-0c4138f5c8a0` 记录的是「HBB V3.3 Change Request 原型全屏（54:26 帧）共 128 条（Pending 28）」，本次视频理解读到的是 `Pending 38` 且总数 428 —— 两者不是同一页面/时间点，引用计数时应注明来源与采样位置，不要混用。

## 4. 待拍板 / 待办（录屏提出但未执行）

- 优先级改 4 档（Low / Normal / High / Critical）；状态补 `Pending` 与 `Not Approved`（或改为 Not Approved 语义的 Rejected）。
- 季度轮播做成配置表（`projects / display_from / display_to / display_quarter`），而非写死日期。
- Change Request 表单补 HBR Title / HBR Type / Impact Level / Impact Date / Target Date 五个字段，并区分 Cross HBR 审批路径。
- 会议本身**没有**出现明确责任人、行动项截止时间或最终签字结论；业务口径仍需业务方提供。
