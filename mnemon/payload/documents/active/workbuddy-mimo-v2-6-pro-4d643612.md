---
id: "4d643612-ad53-4b3c-bdab-6fb324eb31f2"
title: "WorkBuddy 网关多模态能力边界：mimo-v2.6-pro 不支持视频/音频输入（实测 + 源码核对）"
description: "实测 6 组字段形态确认 localhost:8788 的 mimo-v2.6-pro 仅 text+image：video_url 对象被上游过滤、字符串与非标准字段直接 400，prompt_tokens 911→59 是关键判据；含 wb-proxy 源码核对（无看视频端点、reference_videos 属生成素材）与 1920 webp q82 修糊方案。"
status: "active"
created_at: "2026-11-16T07:46:13.396Z"
updated_at: "2026-11-16T07:46:13.396Z"
content_hash: "e4c3737d505c446f0722e144375edf39eeeb19a5f7a157836bcde802243facbd"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-a302fe6a-4dbf-4710-8230-c9f427f57dce"
memory_body_ids:
  []
---

# WorkBuddy 网关多模态能力边界：mimo-v2.6-pro 不支持视频/音频输入（实测 + 源码核对）

## 结论（可直接复用）

`http://localhost:8788/`（WorkBuddy 反代网关，源码在工作区之外的 `Desktop\\WorkBuddy-Portable\\wb-proxy`）的 `mimo-v2.6-pro`（Xiaomi MiMo-V2.6-Pro，1M ctx）**只支持 text + image**，没有开放视频/音频输入。网关具备的视频能力是**生视频**，不是**理解视频**：

- `/v1/models` 22 个模型全部 `input_modalities = [text, image]`，无任何模型声明 video 或 audio；`caps = {vision: True, tool_calls: True, reasoning: True}`，无 video 相关字段。
- 聊天路径对 `video_url`（对象形态）**收得下但模型看不到**，由上游返回"不支持视频"的兜底话术；其它字段形态直接被上游 400 拒。

## 六组实测结果（模型 mimo-v2.6-pro）

| 传法 | 结果 |
|---|---|
| `image_url`（data URI 图片） | HTTP 200，`prompt_tokens=911`，回复准确描述了画面（基线：图片通路正常） |
| `video_url`（对象 + base64 mp4） | HTTP 200 但 `prompt_tokens=59`，模型回复泛化（"I'm unable to view the screen's content"），视频未被解码 |
| `video_url`（对象 + 公网 mp4） | HTTP 200，模型原话 "**this model does not support videos and the content has been filtered. To process videos, please switch to a multimodal model**"；换第二个公网链接回 "I can't view videos—please switch to a multimodal model or share a key screenshot instead." |
| `video_url` 传字符串 | HTTP 400：`json: cannot unmarshal string into Go value of type v2.VideoContent`（上游 Go 结构体只接受对象） |
| `input_video` | HTTP 400：`unsupported content type at index 0: input_video` |
| `input_audio` / `audio_url` | HTTP 400：`unsupported content type at index 0: …` |

关键判据：**`prompt_tokens` 从 911 掉到 59** —— 视频字节完全没进上下文（不是抽帧太少，是被整段丢弃）。用公网真实 mp4（w3schools mov_bbb.mp4、MDN flower.mp4）复测同样被过滤，排除 base64/本地素材自身的嫌疑。

## 源码核对（为什么不是网关中途拦截）

- 网关源码里搜不到 "does not support videos" / "has been filtered" 等字样 → 那句话是**上游**说的，chat 路径把 content 原样透传；`v2.VideoContent` 类型在上游确实存在，只是该模型路由把视频过滤掉。
- 端点表只有：`/v1/chat/completions`、`/v1/responses`、`/v1/models`、`/v1/images/generations`、`/v1/videos`（**生**视频，异步任务）、`/v1/videos/<id>`、`/v1/videos/<id>/content`、`/health`。**没有"看视频"端点**，源码中也没有抽帧/ffmpeg 逻辑。
- `reference_videos` / `reference_audios` 属于**生图/生视频任务的参考素材**，与"让模型看视频"无关：`wb_proxy.py:849 resolve_reference_uploads()`（调用点 `wb_proxy.py:24608`，位于建 body 之后、排队之前）把 data URL 预览签名上传到 COS 换成 `download_url`，因为上游对 `video_url` 是真去下载的，data URL 会报 `2013: content[4].video_url: media url unreachable`（`wb_proxy.py:24574` 注释记录了 2026-11-02 的排查结论）。相关代码：`wb_media.py:597`（kinds 表）、`wb_media.py:1020`（mime→扩展名）、`wb_media_client.py:309`。

## 可行替代路径（若仍要"让画面被理解"）

1. **抽帧 + 逐帧读图**：图片通路正常（911 tokens/帧），60 帧约 7 万输入 tokens，可让模型为每帧生成中文图注替代手写标注。成本走外部模型计费，需用户点头。
2. **动图**：关键片段用 cv2 抽 8fps 做 6–8 秒 webp，纯本地零成本，约 2–4MB/段。
3. **真视频**：等网关上开 video 路由，或换成声明了 video 的模型/端点，调用方代码无需改动。

## 附：同期修正——HTML 内嵌帧"模糊"的根因与修复

此前按文档 `4105dfd5-d53c-49c4-b972-0c4138f5c8a0`（OpenCV 抽帧 + base64 内嵌流程）内嵌的是 1000px、q66 的 webp（≈28KB/张），在 2K 屏上还被拉伸一次，文字必然糊。修正为**原生 1920×1080 帧 + webp q82**：平均 55KB/张，60 张共 3.2MB，单文件 HTML 4.4MB；灯箱内文字可读，交互复测通过（60 帧齐全、画面开关 visible=0、灯箱可开可关、搜索时画面自动隐藏）。除抽帧分辨率与内嵌质量外，其余流程仍以上述文档为准。
