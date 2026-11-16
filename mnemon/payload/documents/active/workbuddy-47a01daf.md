---
id: "47a01daf-6ba2-4019-a367-fe7295840f3a"
title: "WorkBuddy 网关「视频→帧」降级补丁：隔离端口验证、参数、抽帧坑与音频边界"
description: "把 video_url 抽成 JPEG 帧改写为 image_url 的网关补丁实现：扩wbg_proxy.py:16535/28768 扩展点、wb_video_frames.py 与 apply-video-patch.py 两个新文件、WB_VIDEO_* 环境变量、三组实测 token 数与时间点、WinError 206 data URI 传参坑、fail-soft 行为，以及音频零支持的三条可选路径与待决项。"
status: "active"
created_at: "2026-11-16T08:02:29.791Z"
updated_at: "2026-11-16T08:02:29.791Z"
content_hash: "796eaccb212287ac1777d94a7ba38bb5668b93d79bafcad4edaa4ee36d9b7dbd"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-47a68f66-37c8-47ee-84cb-95f137b137cc"
memory_body_ids:
  []
---

# WorkBuddy 网关「视频 → 帧」降级补丁：隔离端口验证过关的实现、参数与边界

承接 `4d643612-ad53-4b3c-bdab-6fb324eb31f2`（mimo-v2.6-pro 仅 text+image 的实测与源码核对，当时只提出"抽帧 + 逐帧读图"设想）与 `425f80f3-74da-428d-a55a-7c777d812224`（重启 8788 会切断自身工具调用的操作纪律）。本文记录该设想**已落地为可安装补丁**：在网关出口把 `video_url` 片段抽成 JPEG 帧后改写为 `image_url`，全程只在副本 / 8789 测试实例上验证，线上 8788 未改动。状态：补丁实现与验证完成，是否上 8788、是否加音频链路仍待用户决定。

## 0. 隔离纪律（用户明确要求）

用户原话（checkpoint 事件 11919）：「你如果改网关 最好另起一个端口啊 不然容易给自己搞蹦 你也是从网关出来的。」

做法：复制 93 个文件 / 5.5 MB / 44 个 .py 到 `C:\Users\63588\Desktop\WorkBuddy-Portable\wb-proxy-test\`，在 **8789** 端口跑补丁版验证。完成后核对：8788 的 `wb_proxy.py` 内 `expand_video_content` 命中 **0** 处、大小仍是 **1,397,351 字节**；8789 监听 pid 39448，8788 监听 pid 26748。测试实例的 `accounts/` 是复制快照，与 8788 互不干扰，过期自行刷新、不写回原目录。

## 1. 补丁点与新增文件

扩展点在请求出口，即 `build_upstream_body()` 之前：

- `wb_proxy.py:16535` `def expand_video_content(body):`
- `wb_proxy.py:28768` `forwarded = expand_video_content(forwarded)`（位于 `_dispatch_chat_post`）

新增两个文件（均在 `wb-proxy-test\`）：

- `wb_video_frames.py`（4,432 B）—— 用**系统 python**（带 cv2，`C:\Users\63588\AppData\Local\Programs\Python\Python312\python.exe`）做子进程抽帧；输入支持 `data:` / http(s) / 本地路径，输出 data URI 帧。
- `apply-video-patch.py`（3,489 B）—— 一键把补丁装到任意 wb-proxy 目录，自带备份（`*.bak-videopatch-20261116`）、`py_compile`、幂等检查（二次运行输出 `already patched`）。

安装到线上（尚未执行，需用户点头后重启才生效）：

```powershell
cd C:\Users\63588\Desktop\WorkBuddy-Portable\wb-proxy-test
C:\Users\63588\AppData\Local\Programs\Python\Python312\python.exe apply-video-patch.py --target ..\wb-proxy
```

## 2. 环境变量（都可不设）

| 变量 | 默认 | 含义 |
|---|---|---|
| `WB_VIDEO_MAX_FRAMES` | 8 | 单段视频最多抽多少帧 |
| `WB_VIDEO_FPS` | 1 | 抽帧帧率上限 |
| `WB_VIDEO_PYTHON` | PATH 上的 python | 抽帧子进程解释器，需带 cv2 |
| `WB_VIDEO_ENABLE` | 1 | 置 0 整段关闭 |

## 3. 三组实测（均在 8789 跑通）

| 场景 | 结果 |
|---|---|
| 3 秒真实会议录屏 `clip.mp4`（192,202 B） | 抽 3 帧，`prompt_tokens` 65 → **1614**，模型准确描述"显示名为 **SSG PN Inventory Overview** 的仪表板…能看到 Shuwen、Shinee5 等人的头像" |
| 60 秒合成视频（1,400,646 B，帧内写 `T=xx.xs / SEGMENT n`） | 抽 8 帧，时间点 **3.7 / 11.3 / 18.7 / 26.3 / 33.7 / 41.3 / 48.7 / 56.3s**，8 个 `T=` 与 SEGMENT 编号全部读对，`prompt_tokens` 1909 |
| 客户端自带帧（`video_url.frames` 传 data URI 数组） | 走零解码路径，3 帧 / 13.4 KB 每帧直出，模型读出 `T=010.0s / 030.0s / 050.0s` |
| 坏 URL `http://127.0.0.1:1/missing.mp4` | fail-soft：HTTP 200、原样转发，日志 `extractor exited 1`，请求不炸 |

成功日志形态：`[wb-proxy] converted 1 video part(s) into 3 frames`。

## 4. 关键坑（踩过才修好的）

- **`[WinError 206] 文件名或扩展名太长`**：把 256,294 字符的 data URI 当命令行参数传给抽帧子进程会直接爆；必须改走 stdin 或临时文件传参。此前日志表现为 `frames=0 meta={}` + "extractor exited 1"，容易被误读成"模型不支持"。
- 抽帧是**同步阻塞**的：60 秒视频约 1 秒解码完；1,400 KB 视频整链 6.7–8.8 秒往返。
- 便携版**自动更新会覆盖 `wb_proxy.py`**，补丁会丢，升级后需重跑一次 apply 脚本。
- 相关既有清洗逻辑（背景知识，勿重复改造）：`wb_proxy.py:14254 normalize_roles`（`developer` → `system`，否则 11128）、`wb_proxy.py:14602 sanitize_content` / `14685 sanitize_messages`（身份句与计费头清洗，`x-anthropic-billing-header` 出现即 400/11128）、`15117 guard_empty_text_parts`。

## 5. 这是帧近似，不是原生视频理解

- ✅ 画面内容问答（页面是什么、数字多少、谁在共享屏）、帧内文字 OCR 级读取很准。
- ⚠️ 稀疏采样：60 秒 = 8 个瞬间（每 7.5 秒一张）；**1 小时视频也仍是 8 帧**（每 7.5 分钟一张），中间大段全丢。加帧代价近似线性：8 帧 ≈ 1.9K prompt tokens，24 帧 ≈ 5–6K，64 帧 ≈ 15K（1M 上下文塞得下，但抽帧同步阻塞）。
- ❌ 动作连续性、转场、动画、"先点这里再拖那里"的因果：帧间全黑，推不出来。
- 真正的原生视频理解需要上游 `input_modalities` 含 video，属模型服务端能力，网关改不出来。

## 6. 音频：零支持，三条路待选

- `/v1/models` 22 个模型全是 `input_modalities=["text","image"]`，无 audio；实测 `input_audio` / `audio_url` → 上游 400 `unsupported content type`。
- 网关源码**没有任何 ASR / transcription 路由**：`reference_audios` 只用于生成视频的参考素材（`wb_proxy.py:860`），唯一的多媒体 POST 是 `/v1/videos`（生成任务，`wb_proxy.py:329 MEDIA_POST_PATHS`、`23148/23199`）。
- 本机也没有现成转录引擎：`faster_whisper` / `whisper` / `funasr` / `vosk` / `torch` / `soundfile` / `librosa` / `av` / `moviepy` 全缺，PATH 无 ffmpeg，网关自带 python 里也没有。
- 可选路径：① 本地 `faster-whisper`（CPU 推理，依赖约 200 MB + base 150 MB / small 500 MB 模型；用 PyAV 直接解音轨绕开 ffmpeg；1 分钟音频在 small 上约 10–20 秒；SSG/PN/HBB 等专有名词需加热词表）；② 云端 ASR（豆包/讯飞/Azure，效果最好但要 key 且音频出网）；③ 不做，只抽画面。

## 7. 未决项（等用户决定）

1. 是否安装 `faster-whisper`（约 200 MB 依赖 + small 模型 500 MB）并在 8789 上做「8 帧 + 音轨全文转录」链路。
2. 补丁是否上 8788 —— 需要 `restart-wb-proxy.bat`，**那一步会掐断正在跑的推理**（本会话即从该网关出网）。备选：先在 8789 观察几天。
