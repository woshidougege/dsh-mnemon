---
id: "bb5c6bd1-4e0b-4dbd-a48e-d9d78d28cb2a"
title: "WorkBuddy 原生视频能力名单：3 个模型真解码、其余 18 个被替换成固定提示语（干净对照实测）"
description: "关掉抽帧补丁后的 21 模型干净对照矩阵：仅 minimax-m3 / minimax-m3.1-flash / space-bunny 原生解码（60s +18.5K tokens，平台约 2fps×154 tok/帧），mimo 等 18 个恒 +29；含请求侧四种 video 标记与 9 个别名全无效、产品配置无 supportsVideo、/v1/models 能力位与抽帧白名单修正、两条路径成本对比。"
status: "active"
created_at: "2026-11-16T08:34:50.403Z"
updated_at: "2026-11-16T08:34:50.403Z"
content_hash: "556ac91401553d63abe0fd1cc2b70aa6713a75b906f34678b471d9f8cbdc37fa"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-bddfe54c-9a56-46bf-94ee-dfb300c0b589"
memory_body_ids:
  []
---

# WorkBuddy 平台原生视频能力名单：只有 3 个模型真解码，其余 18 个被替换成固定提示语

承接 `47a01daf-6ba2-4019-a367-fe7295840f3a`（视频→帧降级补丁的实现与验证）与 `4d643612-ad53-4b3c-bdab-6fb324eb31f2`（mimo-v2.6-pro 仅 text+image 的实测与源码核对）。本文修正前两者的一个关键前提：**平台本身支持视频输入，只是按 model id 灰度开放**；此前"平台/网关不支持视频"的判断是在补丁污染下得出的错误结论。

## 0. 污染来源与干净对照

上一轮 21 模型矩阵跑在 **8789**（补丁开启，`WB_VIDEO_ENABLE=1`）上，每个 `video_url` 都被抽帧代码改写成 `image_url`，所以全体模型看起来都"读到了视频"。改为 `WB_VIDEO_ENABLE=0` 关掉补丁重跑，才得到干净对照组。

## 1. 结论：原生支持视频的模型只有 3 个

判据是「1 秒视频 vs 60 秒视频」的 `prompt_tokens` 增量：真解码的模型增量随时长线性增长，不支持的模型无论多长都只多出固定的 +28~+29。

| 模型 | text-only | +1s 视频 | +60s 视频 | 判定 |
|---|---|---|---|---|
| **minimax-m3** | 181 | +150（331） | **+18,540**（18,721） | ✅ 原生解码 |
| **minimax-m3.1-flash** | 207/208 | +150（357） | **+18,548**（18,756） | ✅ 原生解码 |
| **space-bunny** | 161 | +150（311） | **+18,549**（18,710） | ✅ 原生解码 |
| mimo-v2.6-pro | 24 | +29（53） | **+29** | ❌ 视频被丢弃 |
| hy4-preview-f / hy3 / hy4std / hy4flash / hy4flash-free | 58 | +28（86） | — | ❌ |
| deepseek-v4.1-flash / deepseek-v4-pro | 41 / 94 | +29 | — | ❌ |
| glm-5.1 / 5.2 / 5.3 / glm-5v-turbo | 18~24 | +29 | — | ❌ |
| kimi-k3-1 / k2.8-preview / k2.7 / k2.6 / k3.1-pro | 108 / 108 / 22 / 22 / 108 | +29 | — | ❌ |
| step-5-preview / mimo-v2.6-pro | 27 / 24 | +29 | — | ❌ |

那 +29 就是服务端塞进去的固定提示语 *"this model does not support videos and the content has been filtered"* 本身。矩阵结果保存在 `.tmp\sp-video\matrix.json`（21 行，字段 model/text/video/delta/verdict）。

## 2. 内容层验证：三个模型确实在看画面

60 秒合成视频（1,400,646 B，帧内烧 `T=xx.xs / SEGMENT n`，共 5 个 segment）逐帧嵌入：

- **minimax-m3** 按时间顺序复述出 `T=000.0s — SEGMENT 0`、`000.5s`、`001.0s`、`001.5s`…（间隔 0.5s → 平台内部约 **2 fps** 抽帧，60 s ≈ 120 帧，18,540 ÷ 120 ≈ **154 token/帧**）。
- **space-bunny** 与 **minimax-m3.1-flash** 被问"最后出现的 SEGMENT 编号"，均答 `5`（正确）。

## 3. 为什么"给 mimo 加个视频标记"行不通

请求侧四种标记全部无效，结果完全一致（全为 +29）：

| 做法 | prompt_tokens |
|---|---|
| 只传 `video_url` | 53 |
| 顶层 `modalities:["text","video"]` | 53 |
| 顶层 `input_modalities:["text","video"]` | 53 |
| `supports_video: true` | 53 |
| `video_url.detail:"high"` | 53 |

另外失败的两类尝试：

- `type: "input_video"` / `type: "video"` → HTTP 400，`{"code":11101,"msg":"Parse message failed: unsupported content type at index 1: ..."}`（只认 `video_url`）。
- 9 个隐藏别名全部不存在：`mimo-v2.6-pro-video`、`mimo-v2.6-omni`、`mimo-v2.6`、`mimo-vl`、`mimo-omni`、`mimo-v2.6-pro-vl`、`minimax-m3-video`、`minimax-m3.1`、`space-bunny-video` → `{"code":11102,"msg":"model [x] service info not found"}`。

## 4. 阻断位置分层定位

1. **协议层：通过。** 上游 Go 服务端有 `v2.VideoContent` 结构，`video_url` 对象能正常解析。
2. **网关层：不拦。** `wb_proxy.py` 对 chat 的 content 原样透传；`_flatten_content` 只在 `/v1/responses` 翻译链路丢 video，不影响 chat 判断。
3. **平台路由层：在这里拦。** 服务端按 model id 决定解不解码视频——名单内走视觉编码器，其余把 video part 剥掉换成那句提示语。请求标记、网关声明都改不了。
4. **配置层：查不到。** 上游 `catalog/product-config-cn.json` 的 58 个模型只有 `supportsImages` / `disabledMultimodal` / `supportsReasoning` / `supportsToolCall` / `videoGeneration`（后者是**生成**视频，仅 kling-video-v3、minimax-video-h3 有），**没有 supportsVideo 字段**。本地缓存 `accounts\model.json` 2,595 条记录里 field names 提及 video 的为 0。→ 视频能力是服务端灰度，产品配置与文档都不体现。

## 5. 已在 8789 落地的修正

- `wb_proxy.py` 的 `/v1/models` 能力位（`wb_proxy.py:13541-13597`，vision 推导在 `13555`：`vision = bool(meta.get("supportsImages")) and not meta.get("disabledMultimodal")`，`13562`：`inputs = ["text"] + (["image"] if vision else [])`）为 3 个模型补上 video：
  ```
  minimax-m3          inputs=text+image+video  supports_video=True
  minimax-m3.1-flash  inputs=text+image+video  supports_video=True
  space-bunny         inputs=text+image+video  supports_video=True
  mimo-v2.6-pro       inputs=text+image         supports_video=False
  ```
  `architecture.modality` 输出 `text+image+video->text`、`capabilities.video=true`，使 Cherry Studio / LobeChat / OpenWebUI 这类读 OpenRouter 能力位的客户端开出视频上传入口。声明 22 个模型、其中 3 个带 video。
- 抽帧补丁加**白名单**：这 3 个模型直接走原生，其余 19 个（含 mimo）才走抽帧兜底。修掉了"补丁把 minimax-m3 的 60 秒视频压成 8 帧"的自身 bug。两条路并存互不干扰。

| 路径 | 1 秒视频 | 60 秒视频 |
|---|---|---|
| mimo（抽帧兜底） | +113（3 帧） | +1,864（8 帧，tokens 24→1,888） |
| minimax-m3（原生） | +150 | +18,540（≈120 帧） |

抽帧侧数值以补丁开启实测为准：1 s / 10 帧 320×240 = 15,543 B → 137 tokens；60 s / 900 帧 640×360 = 1,400,646 B → 1,888 tokens。

## 6. 选型建议

不必改网关，**换模型**即可，而且更便宜：`space-bunny` 与 `minimax-m3.1-flash` 都是 `x0.08` credits，mimo-v2.6-pro 是 `x0.13`（deepseek-v4-pro x0.51、kimi-k3.1-pro x0.97）。要让 mimo 看视频只有两条路：(a) 抽帧兜底——60 秒仅 1.9K tokens，但每秒只能看 8 个瞬间；(b) 找腾讯开白（平台侧开关，客户端无解）。

## 7. 当前实例状态（截至该轮结束）

- **8789**（测试副本 `C:\Users\63588\Desktop\WorkBuddy-Portable\wb-proxy-test\`）：pid 18176，patch markers 5，以 `WB_VIDEO_ENABLE=0` 启动（补丁关着，便于做本次对照）；去掉该环境变量重启即恢复抽帧。
- **8788**（线上）：untouched，0 处改动。
- 安装命令（需用户在场，重启会切断自身工具调用）：
  ```powershell
  cd C:\Users\63588\Desktop\WorkBuddy-Portable\wb-proxy-test
  C:\Users\63588\AppData\Local\Programs\Python\Python312\python.exe apply-video-patch.py --target ..\wb-proxy
  ```
  该脚本已同时包含视频能力标记与抽帧白名单。
