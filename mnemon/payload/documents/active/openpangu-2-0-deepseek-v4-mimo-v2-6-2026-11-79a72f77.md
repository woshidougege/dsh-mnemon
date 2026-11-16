---
id: "79a72f77-cb0f-4f81-b498-d29134a93d2c"
title: "三方旗舰模型性能对比：openPangu-2.0 / DeepSeek-V4 / MiMo-V2.6 选型参考（2026-11）"
description: "盘古 2.0 Pro/Flash、DeepSeek-V4-Pro/Flash、MiMo-V2.6-Pro 的规格与官方自报基准对比，含可比性警告、各场景选型分配、接入 CodeArts 的真实收益，以及未执行的盲测提议。"
status: "active"
created_at: "2026-11-16T08:37:47.453Z"
updated_at: "2026-11-16T08:37:47.453Z"
content_hash: "04a242314ad5947d0f7fa6ea907e8b4c325f97795c093d8c84c5e5827d88f271"
source_paths:
  []
session_ids:
  - "mnemon-run-v1-14acce28-9462-43bb-9bde-44978b8ce6c0"
memory_body_ids:
  []
---

# 三方旗舰模型性能横向对比：openPangu-2.0 / DeepSeek-V4 / MiMo-V2.6（2026-11）

来源：2026-11-16 会话中对 openPangu-2.0-Pro、DeepSeek-V4、MiMo-V2.6 官方模型卡与官网的多源抓取。所有分数均为**厂商自报**，非本机实测；本机实测能力矩阵见相关文档 `bb5c6bd1-4e0b-4dbd-a48e-d9d78d28cb2a`（视频解码名单）。

## 1. 规格对照

| | openPangu-2.0-Pro | DeepSeek-V4-Pro | MiMo-V2.6-Pro |
|---|---|---|---|
| 参数 | 505B 总 / 18B 激活 | 1.6T 总 / 49B 激活（V4-Flash 284B/13B） | 未公开（384 routed experts / 8 激活） |
| 上下文 | 512K | 1M | 1M（最大输出 128K） |
| 模态 | 纯文本 | 纯文本（另有 V4-Flash-Vision-Exp） | 全模态输入：文本/图像/视频/音频，输出文本 |
| 架构 | MLA + DSA/SWA 分层（1:2）、4 支流 mHC 残差、3 头 MTP | FP4+FP8 混合（MoE expert FP4） | 70 层(60 SWA/10 GA)、hidden 6144、SWA window 128、MTP 5 SWA 层 window 1024 |
| 训练 | 昇腾 NPU 原生，34T tokens，Muon 优化器 | — | — |
| 推理档位 | Thinking / Non-Thinking 双版本 | Non-think / Think High / Think Max | 可关 thinking（`extra_body.thinking.type=disabled`） |

MiMo-V2.6-Pro 商务面：RPM 100 / TPM 10M；人民币 ¥0.025（缓存命中）/ ¥3（未命中）/ ¥6（输出）每百万 tokens；官网标注 Artificial Analysis 综合智能指数 46.32。**MiMo-V2.5-Pro / V2.5 已于 2026.10.21 10:00 (UTC+8) 下线，旧名失效。**

## 2. 口径基本一致的可比项（盘古 Pro-Thinking vs DS-V4-Pro-Max）

| Benchmark | 盘古 2.0-Pro | DS-V4-Pro Max |
|---|---|---|
| GPQA-Diamond | 87.9 | **90.1** |
| HLE | 27.1 | **37.7** |
| SimpleQA-Verified | 34.9 | **57.9** |
| Chinese-SimpleQA | 74.2 | **84.4** |
| SWE-bench Verified | 68.5 | **80.6** |
| BrowseComp | 65.7 | **83.4** |
| LiveCodeBench | 85.7 (V6) | **93.5**（版本口径存疑） |
| HMMT Feb 2026 | 86.2 | **95.2** |
| AIME 2026 | **95.4**（+Python 97.9） | 未报 |

盘古 Flash-Thinking 相对 Pro 降级很小：AIME 93.3、GPQA 83.7、IFEval 95.9（严格）、BrowseComp 57.0 —— 省额度场景可直接用 Flash。盘古 Non-Thinking 档掉档明显（HLE 9.0、Apex 2.1）。

MiMo-V2.6-Pro 自报（题集与其他家不同）：DeepSWE v1.1 71.9、Terminal Bench 2.1 89.9、Toolathlon-Verified 76.9、OSWorld-Verified 82.0、GDPval-AA 1673 Elo、CyberGym 94.0、Agents' Last Exam 31.6、MiMo VisualCoding 72.3。

**口径警告（重要）**：Terminal Bench 2.0 与 2.1、Toolathlon 与 Toolathlon-Verified、LiveCodeBench 各代版本互不可比。真正能横向排的只有 GPQA、HLE、SimpleQA-Verified、BrowseComp、SWE-Verified 这几项。

## 3. 判定结论

- **盘古 2.0 属第一梯队边缘**：数学（AIME 95.4）与指令跟随（IFEval 94.5）已接近顶级，18B 激活打出 505B 等效，参数效率是其技术亮点。
- **三项明显短板**：① 世界知识/事实可靠性（SimpleQA 34.9 vs 57.9、HLE 27.1 vs 37.7）最突出，冷门/时效问题易含糊或编造；② 真实仓库代码工程（SWE 68.5 vs 80.6）；③ 检索型 Agent（BrowseComp 65.7 vs 83.4）。上下文 512K 亦小于另两家 1M。
- **MiMo-V2.6-Pro 强在长程 Agent/电脑操控/安全与全模态**，代码工程 DeepSWE 仍低于 DeepSeek-V4-Pro 之外的第一梯队。
- 外部评价参考：知乎「5050亿参数！余承东开源盘古新模型，"世界第一"还有些距离」——与上述短板判断一致。

## 4. 场景分配建议（针对本机多模型网关）

| 场景 | 选择 |
|---|---|
| 写代码 / 修 bug / 大仓库改造 | `deepseek-v4-pro`（SWE 80.6） |
| 长程 Agent / 电脑操控 / 安全分析 | `mimo-v2.6-pro`（OSWorld 82、CyberGym 94） |
| 视频/音频/图像理解 | `mimo-v2.6-pro`（网关内其余多为纯文本+图；真正原生视频解码另见 `bb5c6bd1-4e0b-4dbd-a48e-d9d78d28cb2a`） |
| 数学硬题 | `deepseek-v4-pro` 或盘古 Pro |
| 冷门知识问答 | 不用盘古 |

**接入 CodeArts 的正当理由不是"盘古更强"，而是三点**：① 多一个独立免费额度池（华为云积分，不消耗网关现有 16 个号的余额）；② 国产合规/昇腾原生场景；③ CodeArts 福利名单还含 `deepseek-v4-pro-0813`、`deepseek-v4-flash-0731`、`glm-5.3-flash`，等于多一个额度池。

## 5. 待定（未执行）

已向用户提议同题盲测（5 题：数学 / 真实 bug 修复 / 中文写作 / 长上下文检索 / 常识陷阱；候选 `deepseek-v4-pro` vs `mimo-v2.6-pro` vs `kimi-k3.1-pro` 可即时跑，盘古需先开 CodeArts 授权接入），**用户尚未回应，未执行**。

## 6. 来源

- https://huggingface.co/openpangu/openPangu-2.0-Pro/blob/main/README_EN.md （含完整 Pro/Flash 评测表与架构说明）
- https://www.modelscope.ai/models/openpangu/openPangu-2.0-Pro
- https://support.huaweicloud.com/bestpractice-maas/bestpractice_maas_0021_02.html （华为云 MaaS 快速体验）
- https://www.huawei.com/cn/news/2026/7/openpangu （官方开源公告）
- https://huggingface.co/deepseek-ai/DeepSeek-V4-Pro （含 Base/Instruct 评测与三档推理模式、Chat Template 说明：本版无 Jinja 模板，改用 `encoding` 目录 Python 脚本）
- https://www.xiaomimimo.com 官网模型页 + https://huggingface.co/XiaomiMiMo/MiMo-V2.6-Pro-RL
- https://zhuanlan.zhihu.com/p/2066568286255462250 （第三方评测评论）
