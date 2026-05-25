# LTX 2.3 ComfyUI 完全精通指南

## 文档说明

本文档是一份**自包含**的 LTX 2.3 视频生成模型学习教程，严格遵循节点级操作精度，每个步骤均可复现。读者只需依照本文档顺序操作，即可从零基础达到精通级别。

**前置条件**：ComfyUI 已安装并能正常运行（版本要求 ≥ v0.3.0）

**目标模型版本**：LTX 2.3（Lightricks 开源视频生成模型，22B 参数）

---

## 目录

1. [LTX 2.3 核心概念与设计哲学](#一ltx-23-核心概念与设计哲学)
2. [环境准备与模型部署](#二环境准备与模型部署)
3. [工作流模板系统使用](#三工作流模板系统使用)
4. [工作流 A：文生视频基础工作流](#四工作流a文生视频基础工作流)
5. [工作流 B：图生视频工作流](#五工作流b图生视频工作流)
6. [工作流 C：两阶段高质量工作流（核心）](#六工作流c两阶段高质量工作流核心)
7. [工作流 D：GGUF 低显存方案](#七工作流d-gguf-低显存方案)
8. [工作流 E：首末帧控制工作流](#八工作流e首末帧控制工作流)
9. [高级优化技巧](#九高级优化技巧)
10. [故障排除指南](#十故障排除指南)

---

## 一、LTX 2.3 核心概念与设计哲学

### 1.1 模型概述

LTX 2.3 是 Lightricks 开源的视频生成模型，参数量 22B，相比 LTX 2（19B）有以下核心改进：

| 特性     | LTX 2    | LTX 2.3          |
| -------- | -------- | ---------------- |
| 参数量   | 19B      | 22B              |
| 竖屏支持 | 一般     | 大幅优化（9:16） |
| 音频质量 | 有杂音   | 更清晰，噪声降低 |
| 图生视频 | 偶有卡顿 | 运动更一致       |
| 文字渲染 | 模糊     | 更清晰           |

### 1.2 LTX 2.3 的设计哲学

理解以下三个核心概念是精通的基础：

**1. 两阶段生成哲学**

LTX 2.3 官方推荐**不在单次采样中追求最终质量**，而是：
- 第一阶段：半分辨率生成，确定构图和运动轨迹（成本低）
- 第二阶段：潜空间放大，补充细节（质量高）

这种设计的本质是**空间解耦**——将"做什么"和"做多细"分离。

**2. 潜空间放大的优势**

传统的做法是：生成低分辨率视频 → 解码为像素 → 用 ESRGAN 等模型放大。这会丢失时序一致性。

LTX 2.3 的做法是：在潜空间中直接放大（`LTXVLatentUpsampler`），保持运动连贯性的同时增加空间细节。

**3. 文本编码器的选择逻辑**

Gemma 3 12B 是 LTX 2.3 指定的文本编码器，不可替换为 CLIP 或其他模型。这是因为 LTX 2.3 的训练 pipeline 与 Gemma 3 的 embedding space 深度绑定。

### 1.3 Checkpoint 类型选择指南

LTX 2.3 提供三种 checkpoint 变体，选择场景如下：

| Checkpoint 类型 | 显存占用 | 质量       | 推荐场景                           |
| --------------- | -------- | ---------- | ---------------------------------- |
| **dev (BF16)**  | ~44GB    | 最高       | 24GB+ 显卡 + 探索性创作            |
| **fp8**         | ~23-29GB | 接近 dev   | 16-24GB 显卡，日常使用首选         |
| **distilled**   | ~23GB    | 略低于 fp8 | 需要快速迭代 + 搭配 distilled LoRA |

> 💡 **提示**：distilled checkpoint 必须配合 `ltx-2.3-22b-distilled-lora-384` 使用，且 CFG 需设为 1.0，步数 4-8 步。不使用 LoRA 时，效果反而不如 fp8。

---

## 二、环境准备与模型部署

### 2.1 必需的自定义节点

在开始之前，确认以下节点已安装：

**【节点 1】：ComfyUI-LTXVideo（官方核心节点）**

【安装来源】：https://github.com/Lightricks/ComfyUI-LTXVideo

【安装方式】：ComfyUI Manager → 搜索 "LTXVideo" → Install → 重启 ComfyUI

【网络处理】：
> ⚠️ **网络提示**：通过 ComfyUI Manager 安装此节点需要从 GitHub 拉取代码。若无法访问，可使用以下替代方案：
> - 方案 A：手动下载 ZIP → 解压到 `custom_nodes/ComfyUI-LTXVideo-master` → 重命名去掉 `-master`
> - 方案 B：使用代理环境

**【节点 2】：ComfyUI-GGUF（如需使用 GGUF 量化模型）**

【安装来源】：https://github.com/city96/ComfyUI-GGUF

【安装方式】：ComfyUI Manager → 搜索 "GGUF" → Install → 重启

【网络处理】：同上

**【节点 3】：ComfyUI-RTX-Video-Super-Resolution（可选，用于 4K 升频）**

【安装来源】：ComfyUI Manager → 搜索 "RTX Video" → Install

【网络处理】：同上

### 2.2 模型下载清单

**【模型 1】：LTX 2.3 Checkpoint（三选一）**

> ⚠️ **网络提示**：从 HuggingFace 下载模型需要特殊网络环境。如无法访问，可使用：
> - 国内镜像站：hf-mirror.com（搜索 "Lightricks/LTX-2.3-22B"）
> - 哩布哩布 AI：搜索 "LTX 2.3" 查找搬运资源

**选项 A - fp8 版本（推荐 16-24GB 显卡）**：
- 下载地址：https://huggingface.co/Lightricks/LTX-2.3-22B-fp8
- 存放路径：`ComfyUI/models/checkpoints/ltx-2.3-22b-dev-fp8.safetensors`

**选项 B - dev BF16 版本（24GB+ 显卡）**：
- 下载地址：https://huggingface.co/Lightricks/LTX-2.3-22B
- 存放路径：`ComfyUI/models/checkpoints/ltx-2.3-22b-dev.safetensors`

**选项 C - distilled 版本（搭配 LoRA 使用）**：
- 下载地址：https://huggingface.co/Lightricks/LTX-2.3-22B-distilled
- 存放路径：`ComfyUI/models/checkpoints/ltx-2.3-22b-distilled.safetensors`

**【模型 2】：Gemma 3 12B 文本编码器（必需）**

> ⚠️ **网络提示**：此模型约 8GB，从 HuggingFace 下载需要代理或镜像。

- 下载地址：https://huggingface.co/Lightricks/gemma_3_12B_it_fp4_mixed
- 存放路径：`ComfyUI/models/text_encoders/gemma_3_12B_it_fp4_mixed.safetensors`

**【模型 3】：LTX 2.3 潜空间放大模型（两阶段工作流必需）**

> ⚠️ **网络提示**：同上的网络处理方案。

- 下载地址：https://huggingface.co/Lightricks/LTX-2.3-spatial-upscaler-x2
- 存放路径：`ComfyUI/models/latent_upscale_models/ltx-2.3-spatial-upscaler-x2-1.0.safetensors`

**【模型 4】：Distilled LoRA（使用 distilled checkpoint 时必需）**

- 下载地址：https://huggingface.co/Lightricks/LTX-2.3-22B-distilled-lora-384
- 存放路径：`ComfyUI/models/loras/ltx-2.3-22b-distilled-lora-384.safetensors`

**【模型 5】：LTX Audio VAE（需要音频时使用）**

- 下载地址：https://huggingface.co/Lightricks/LTX-2.3-audio-vae
- 存放路径：`ComfyUI/models/vae/ltx-audio-vae.safetensors`

### 2.3 文件结构验证

完成下载后，目录结构应为：

```
ComfyUI/
├── models/
│   ├── checkpoints/
│   │   └── ltx-2.3-22b-dev-fp8.safetensors  (或你选择的版本)
│   ├── text_encoders/
│   │   └── gemma_3_12B_it_fp4_mixed.safetensors
│   ├── latent_upscale_models/
│   │   └── ltx-2.3-spatial-upscaler-x2-1.0.safetensors
│   ├── loras/
│   │   └── ltx-2.3-22b-distilled-lora-384.safetensors
│   └── vae/
│       └── ltx-audio-vae.safetensors (可选)
└── custom_nodes/
    ├── ComfyUI-LTXVideo/
    └── ComfyUI-GGUF/ (可选)
```

---

## 三、工作流模板系统使用

> 💡 **提示**：这是官方提供的最快捷方式，应优先于手动搭建。

### 3.1 打开工作流模板

【操作步骤】：
1. 顶部菜单栏 → **Workflow** → **Browse Workflow Templates**
2. 在搜索框输入 "LTX-2.3"
3. 选择目标模板：
   - **Text to Video (LTX-2.3)** - 文生视频
   - **Image to Video (LTX-2.3)** - 图生视频
   - **FirstFrame/LastFrame (LTX-2.3)** - 首末帧控制

### 3.2 自动模型检测与下载

加载模板后，ComfyUI 会自动：
1. 扫描工作流所需的所有模型
2. 弹出对话框列出缺失模型
3. 提供 HuggingFace 直链（需要代理访问）

> ⚠️ **网络提示**：自动下载功能需要访问 HuggingFace API。如无法使用，可根据弹窗提示的模型名称，手动从镜像站下载后放到对应目录，然后点击 "Refresh"。

---

## 四、工作流 A：文生视频基础工作流

这是理解 LTX 2.3 工作流结构的最小单元。后续所有工作流都在此基础上扩展。

### 4.1 整体结构预览

```
[Load Checkpoint] ─┬─→ [LTXVTextToVideoSampler] → [VAEDecodeTiled] → [SaveVideo]
                   │
[Gemma Text Encoder] ┘
                   │
[EmptyLTXVLatentVideo] ─→ (潜空间初始化)
```

### 4.2 逐节点搭建

**【节点 1】：Checkpoint 加载器（CheckpointLoaderSimple）**

【添加方式】：鼠标右键 → 搜索 "checkpoint" → 选择 "CheckpointLoaderSimple"

【添加位置】：画布左上角

【输入连接】：无

【关键参数及设置理由】：
- `ckpt_name`: 选择 `ltx-2.3-22b-dev-fp8.safetensors`（根据你的显卡选择）
- 设置理由：fp8 版本在 16GB 显卡上可稳定运行，质量损失约 5-10%

【输出连接】：
- `MODEL` → 连接到 LTXVTextToVideoSampler 的 `model` 输入端口
- `VAE` → 连接到 VAEDecodeTiled 的 `vae` 输入端口

> ⚠️ **注意**：LTX 2.3 的 VAE 已集成在 checkpoint 中，无需单独加载 VAE 文件。

**【节点 2】：Gemma 文本编码器加载器（LTXTextEncoderLoader）**

【添加方式】：鼠标右键 → 搜索 "LTXText" → 选择 "LTXTextEncoderLoader"

【添加位置】：放在 Load Checkpoint 节点右侧

【输入连接】：无

【关键参数及设置理由】：
- `text_encoder_name`: 选择 `gemma_3_12B_it_fp4_mixed.safetensors`
- 设置理由：LTX 2.3 的文本编码器已固定为 Gemma 3 12B，不可替换

【输出连接】：
- `text_encoder` → 连接到 LTXVTextToVideoSampler 的 `text_encoder` 输入端口

**【节点 3】：空视频潜空间（EmptyLTXVLatentVideo）**

【添加方式】：鼠标右键 → 搜索 "EmptyLTXVLatent" → 选择 "EmptyLTXVLatentVideo"

【添加位置】：放在 Checkpoint Loader 下方

【输入连接】：无

【关键参数及设置理由】：
- `width`: 832（推荐值，需为 32 的倍数）
- `height`: 480（推荐值，需为 32 的倍数）
- `length`: 33（帧数，公式为 8×N+1，推荐 33、41、49）
- `batch_size`: 1

设置理由：
- LTX 模型有步幅约束，宽高必须能被 32 整除，帧数必须满足 8×N+1
- 832×480 是 16:9 的入门分辨率，显存友好
- length=33 约 1.1 秒视频（33帧 / 30fps）

【输出连接】：
- `latent` → 连接到 LTXVTextToVideoSampler 的 `samples` 输入端口

**【节点 4】：正面提示词（CLIP Text Encode / Positive）**

【添加方式】：鼠标右键 → 搜索 "CLIP Text" → 选择 "CLIP Text Encode (Positive)"

【添加位置】：放在 Gemma Encoder 右侧

【关键参数及设置理由】：
- `text`: 填写你的正向提示词
- 推荐格式："[场景描述]，[运动描述]，[风格描述]，[画质描述]"

示例：
```
Cinematic shot of a woman walking through a rainy Tokyo street at night, holding a transparent umbrella, camera slowly pans left, neon signs reflecting on wet ground, 4K, high quality, detailed face
```

【输出连接】：
- `CONDITIONING` → 连接到 LTXVTextToVideoSampler 的 `positive` 输入端口

**【节点 5】：负面提示词（CLIP Text Encode / Negative）**

【添加方式】：鼠标右键 → 搜索 "CLIP Text" → 选择 "CLIP Text Encode (Negative)"

【关键参数及设置理由】：
- `text`: 填写负面提示词

推荐模板：
```
worst quality, low quality, blurry, distorted, ugly, bad anatomy, bad hands, deformed, disfigured, watermarked, text, logo, cropped, jpeg artifacts, missing fingers, extra digits, poorly drawn hands, mutated hands, mutation, deformed, bad anatomy
```

【输出连接】：
- `CONDITIONING` → 连接到 LTXVTextToVideoSampler 的 `negative` 输入端口

**【节点 6】：LTX 文生视频采样器（LTXVTextToVideoSampler）**

【添加方式】：鼠标右键 → 搜索 "LTXVTextToVideo" → 选择 "LTXVTextToVideoSampler"

【添加位置】：放在所有输入节点的交汇处

【输入连接】：
- `model` ← 来自 CheckpointLoaderSimple 的 `MODEL`
- `text_encoder` ← 来自 LTXTextEncoderLoader 的 `text_encoder`
- `samples` ← 来自 EmptyLTXVLatentVideo 的 `latent`
- `positive` ← 来自 CLIP Text Encode (Positive) 的 `CONDITIONING`
- `negative` ← 来自 CLIP Text Encode (Negative) 的 `CONDITIONING`

【关键参数及设置理由】：

| 参数           | 推荐值         | 设置理由                                 |
| -------------- | -------------- | ---------------------------------------- |
| `seed`         | 随机数或固定值 | 固定种子可复现结果；-1 为随机            |
| `steps`        | 30-50          | 低于 20 步运动不连贯，高于 50 步收益递减 |
| `cfg`          | 3.0-4.0        | 视频生成推荐 2.0-5.0，过高会导致运动僵硬 |
| `sampler_name` | euler          | LTX 模型原生测试的采样器                 |
| `scheduler`    | normal         | 配合 euler 使用                          |

【输出连接】：
- `latent` → 连接到 VAEDecodeTiled 的 `samples` 输入端口

**【节点 7】：分块 VAE 解码（VAEDecodeTiled）**

【添加方式】：鼠标右键 → 搜索 "VAEDecodeTiled" → 选择

【添加位置】：放在采样器右侧

【输入连接】：
- `samples` ← 来自 LTXVTextToVideoSampler 的 `latent`
- `vae` ← 来自 CheckpointLoaderSimple 的 `VAE`

【关键参数及设置理由】：
- `tile_size`: 512（默认值）
- 设置理由：分块解码可减少显存峰值，12GB 显卡建议保持默认

【输出连接】：
- `IMAGE` → 连接到 SaveVideo 或 VideoCombine 的 `images`

**【节点 8】：保存视频（SaveVideo）**

【添加方式】：鼠标右键 → 搜索 "SaveVideo" → 选择

【输入连接】：
- `images` ← 来自 VAEDecodeTiled 的 `IMAGE`

【关键参数及设置理由】：
- `filename_prefix`: 自定义输出文件名前缀
- `fps`: 24 或 30（推荐 24，与训练数据一致）

### 4.3 工作流 A 检查清单

- [ ] 所有节点已连接，无红线
- [ ] 提示词已填写
- [ ] 分辨率宽高为 32 的倍数
- [ ] length 满足 8×N+1
- [ ] 点击 Queue Prompt 能正常执行

---

## 五、工作流 B：图生视频工作流

从一张静态图片生成延续运动的视频。

### 5.1 与工作流 A 的差异点

图生视频相比文生视频：
- 用 `LTXVImageToVideoSampler` 替代 `LTXVTextToVideoSampler`
- 增加 `LoadImage` 节点作为首帧输入
- 运动由"提示词"驱动，而非"从零生成"

### 5.2 逐节点搭建

**【节点 1-5】**：同工作流 A（Checkpoint、Gemma Encoder、EmptyLatent、Positive、Negative）

**【节点 6】：加载图像（LoadImage）**

【添加方式】：鼠标右键 → 搜索 "LoadImage" → 选择

【添加位置】：放在采样器左侧

【关键参数及设置理由】：
- 选择你的首帧图片（推荐 16:9 比例，如 832×480）
- 设置理由：首帧分辨率应与目标视频分辨率一致或成比例

【输出连接】：
- `IMAGE` → 连接到 ImageResize 节点

**【节点 7】：图像缩放（ImageResize）**

【添加方式】：鼠标右键 → 搜索 "ImageResize" → 选择

【输入连接】：
- `image` ← 来自 LoadImage 的 `IMAGE`

【关键参数及设置理由】：
- `width`: 832（与 EmptyLatent 的 width 一致）
- `height`: 480（与 EmptyLatent 的 height 一致）
- `resampling`: lanczos

【输出连接】：
- `image` → 连接到 LTXVImageToVideoSampler 的 `image` 输入端口

**【节点 8】：LTX 图生视频采样器（LTXVImageToVideoSampler）**

【添加方式】：鼠标右键 → 搜索 "LTXVImageToVideo" → 选择 "LTXVImageToVideoSampler"

【输入连接】：
- `model` ← 来自 CheckpointLoaderSimple 的 `MODEL`
- `text_encoder` ← 来自 LTXTextEncoderLoader 的 `text_encoder`
- `samples` ← 来自 EmptyLTXVLatentVideo 的 `latent`
- `image` ← 来自 ImageResize 的 `image`（经过缩放）
- `positive` ← 来自 CLIP Text Encode (Positive) 的 `CONDITIONING`
- `negative` ← 来自 CLIP Text Encode (Negative) 的 `CONDITIONING`

【关键参数及设置理由】：

| 参数                | 推荐值     | 设置理由                         |
| ------------------- | ---------- | -------------------------------- |
| `seed`              | 固定或随机 | -                                |
| `steps`             | 40-60      | 图生视频需更多步数以平滑首帧过渡 |
| `cfg`               | 3.0-3.5    | 图生视频 cfg 过高会扭曲原图      |
| `image_noise_scale` | 0.1-0.3    | 控制首帧保留程度；越低越接近原图 |

【输出连接】：
- `latent` → 连接到 VAEDecodeTiled 的 `samples`

**【节点 9-10】**：VAEDecodeTiled + SaveVideo（同工作流 A）

### 5.3 图生视频的运动提示词技巧

> 💡 **提示**：图生视频的提示词应描述**运动**而非场景。

- **错误示例**："a beautiful mountain landscape"
- **正确示例**："camera slowly pans left, clouds drift across the sky, light rays through trees"

---

## 六、工作流 C：两阶段高质量工作流（核心）

这是 LTX 2.3 的**精髓工作流**，所有追求质量的用户都应掌握。

### 6.1 两阶段流程图解

```
阶段一（半分辨率）：
[Load Checkpoint] → [LTXVTextToVideoSampler] (640×384, 25-35步) → [Latent Temp Storage]
                              ↓
阶段二（潜空间放大 + 精炼）：
[Load Distilled LoRA] → [LTXVLatentUpsampler] (2×放大) → [LTXVTextToVideoSampler] (精炼, 15-20步) → [VAEDecode] → [SaveVideo]
```

### 6.2 阶段一：半分辨率基础生成

**【节点 1-5】**：同工作流 A，但分辨率参数不同

【EmptyLTXVLatentVideo 参数调整】：
- `width`: 640（半分辨率）
- `height`: 384（半分辨率）
- `length`: 33

> 💡 **理由**：阶段一只需要确定构图和运动轨迹，低分辨率可节省显存和计算时间。

**【LTXVTextToVideoSampler 参数调整】**：
- `steps`: 25-35（阶段一步数可略低）
- `cfg`: 4.0-5.0（略高以确定运动方向）

**【新增：潜空间中转存储】**

由于阶段二的采样器需要阶段一的输出，需将 latent 保存到变量。

【添加方式】：直接连线（不经过额外节点）

【连接】：
- `latent`（阶段一采样器输出）→ 连接到阶段二 Upsampler 的 `latent` 输入

### 6.3 阶段二：加载 Distilled LoRA（如使用 distilled checkpoint）

**【节点 6】：LoRA 加载器（LoraLoaderModelOnly）**

【添加方式】：鼠标右键 → 搜索 "LoraLoaderModelOnly" → 选择

【添加位置】：放在阶段一采样器和阶段二 Upsampler 之间

【输入连接】：
- `model` ← 来自 CheckpointLoaderSimple 的 `MODEL`（同阶段一）

【关键参数及设置理由】：
- `lora_name`: 选择 `ltx-2.3-22b-distilled-lora-384.safetensors`
- `strength_model`: 0.6-0.8（推荐 0.7）

> 💡 **理由**：Distilled LoRA 在放大阶段可提升纹理细节，但强度过高会导致过饱和。

【输出连接】：
- `MODEL` → 连接到阶段二采样器的 `model` 输入

### 6.4 阶段二：潜空间放大

**【节点 7】：潜空间放大模型加载器（LatentUpscaleModelLoader）**

【添加方式】：鼠标右键 → 搜索 "LatentUpscaleModelLoader" → 选择

【关键参数及设置理由】：
- `model_name`: 选择 `ltx-2.3-spatial-upscaler-x2-1.0.safetensors`

【输出连接】：
- `UPSCALE_MODEL` → 连接到 LTXVLatentUpsampler 的 `upscale_model` 输入

**【节点 8】：LTX 潜空间放大（LTXVLatentUpsampler）**

【添加方式】：鼠标右键 → 搜索 "LTXVLatentUpsampler" → 选择

【输入连接】：
- `latent` ← 来自阶段一 LTXVTextToVideoSampler 的 `latent` 输出
- `upscale_model` ← 来自 LatentUpscaleModelLoader 的 `UPSCALE_MODEL`

【关键参数及设置理由】：
- `width`: 1280（目标宽，为阶段一的 2 倍）
- `height`: 768（目标高，为阶段一的 2 倍）
- 设置理由：放大因子固定为 2×，不可自定义

【输出连接】：
- `latent` → 连接到阶段二采样器的 `samples` 输入

### 6.5 阶段二：精炼采样

**【节点 9】：LTXVTextToVideoSampler（阶段二）**

【添加方式】：第二个实例，与阶段一相同节点类型

【输入连接】：
- `model` ← 来自 LoraLoaderModelOnly 的 `MODEL`（或直接来自 Checkpoint）
- `text_encoder` ← 来自 LTXTextEncoderLoader（可复用阶段一）
- `samples` ← 来自 LTXVLatentUpsampler 的 `latent`
- `positive` ← 来自正面提示词节点（可与阶段一相同或更细化）
- `negative` ← 来自负面提示词节点

【关键参数及设置理由】：

| 参数      | 推荐值  | 设置理由                             |
| --------- | ------- | ------------------------------------ |
| `steps`   | 15-20   | 精炼阶段不需太多步，主要补充细节     |
| `cfg`     | 3.0-4.0 | 与阶段一保持一致或略低               |
| `denoise` | 0.4-0.6 | 降噪强度，保留原有结构的同时增加细节 |

> 💡 **关键理解**：阶段二不是"重新生成"，而是在已有潜空间结构上的"细节补充"。`denoise < 1.0` 正是这个含义。

**【节点 10-11】**：VAEDecodeTiled + SaveVideo（同工作流 A）

### 6.6 两阶段工作流的参数调优速查

| 场景                    | 阶段一步数 | 阶段二步数 | denoise | LoRA 强度 |
| ----------------------- | ---------- | ---------- | ------- | --------- |
| 人像特写（需细腻皮肤）  | 30         | 20         | 0.5     | 0.7       |
| 风景/场景（需锐利边缘） | 25         | 15         | 0.6     | 0.5       |
| 动作/运动（需流畅）     | 35         | 18         | 0.4     | 0.6       |
| 风格化/艺术             | 25         | 15         | 0.5     | 0.8       |

---

## 七、工作流 D：GGUF 低显存方案

对于 12-16GB 显存的显卡，可通过 GGUF 量化将显存需求从 ~23GB 降至 ~18GB。

### 7.1 准备工作

> ⚠️ **网络提示**：GGUF 量化模型需要从 HuggingFace 下载，同样需特殊网络环境。国内镜像站 hf-mirror.com 搜索 "LTX-2.3-GGUF" 可找到搬运。

**必需文件**：
- GGUF 量化模型：`sulphur_dev-Q4_K_M.gguf`（或其他量化级别）
- 存放路径：`ComfyUI/models/unet/`（注意不是 checkpoints 目录！）

### 7.2 使用官方模板改造

官方 LTX 模板默认使用 `CheckpointLoaderSimple` 加载模型。但 GGUF 模型需要用专门的 `Unet Loader (GGUF)` 节点。

**【推荐操作流程】**：

1. 从模板库加载 **Image to Video (LTX-2.3)** 模板
2. 点击外层节点右上角的**展开按钮**，进入内部工作流
3. 找到模型加载区域（通常有 `CheckpointLoaderSimple` 和 `LoraLoader`）
4. 按以下步骤改造

### 7.3 逐节点改造

**【原连接】**（需断开）：
```
CheckpointLoaderSimple 的 MODEL 输出 → LoRA加载器 的 model 输入
```

**【新增节点】：Unet Loader (GGUF)**

【添加方式】：鼠标右键 → 搜索 "Unet Loader GGUF" → 选择

【添加位置】：放在原 CheckpointLoaderSimple 旁边

【关键参数及设置理由】：
- `unet_name`: 选择你的 GGUF 文件（如 `sulphur_dev-Q4_K_M.gguf`）
- 设置理由：GGUF 量化模型需通过此节点加载，不能使用普通 Checkpoint Loader

> ⚠️ **重要**：不要删除原来的 CheckpointLoaderSimple！官方模板内部可能还通过它维持工作流完整性。只改主模型流向。

**【修改后的连接】**：
```
Unet Loader (GGUF) 的 model 输出 → LoRA加载器（仅模型） 的 model 输入
```

**【保留原样】**：
- CLIP 和 VAE 的加载路径**不要动**
- 如果原 CheckpointLoaderSimple 输出的 VAE 仍在使用，保持连接

### 7.4 GGUF 量化级别选择

| 量化级别   | 显存占用 | 质量损失 | 推荐场景          |
| ---------- | -------- | -------- | ----------------- |
| Q2_K       | ~11GB    | 明显     | 12GB 显卡勉强运行 |
| Q3_K_M     | ~14GB    | 中等     | 12GB 显卡稳定运行 |
| **Q4_K_M** | ~18GB    | 轻微     | 16GB 显卡首选     |
| Q5_K_M     | ~22GB    | 极小     | 24GB 显卡可选     |
| Q8_0       | ~30GB    | 几乎无损 | 24GB+ 显卡        |

> 💡 **提示**：Q4_K_M 是显存和质量的最佳平衡点，绝大多数 16GB 显卡用户应选择此级别。

### 7.5 GGUF 工作流检查清单

- [ ] GGUF 文件放在 `models/unet/` 目录
- [ ] ComfyUI-GGUF 自定义节点已安装
- [ ] 使用 Unet Loader (GGUF) 而非 CheckpointLoaderSimple
- [ ] 原 CheckpointLoaderSimple 保留（不删除）
- [ ] VAE 和 CLIP 连接未被切断

---

## 八、工作流 E：首末帧控制工作流

此工作流可指定视频的开始帧和结束帧，让模型自动生成中间过渡动画。

### 8.1 使用模板（最简单）

【操作步骤】：
1. Workflow → Browse Workflow Templates
2. 搜索 "LTX-2.3 FirstFrame/LastFrame"
3. 加载模板

### 8.2 首末帧连接的要点

此工作流的核心节点是 `LTXVImageToVideoSampler` 的特殊使用方式：

【关键连接】：需要**两个独立的图像加载节点**分别连接到采样器的 `first_frame` 和 `last_frame` 端口

【提示词撰写方式】：
提示词应描述首帧到末帧之间的**过渡动作**，而非描述场景。

**示例**：
```
Camera slowly pushes forward through the cockpit, moving past the pilots toward the front window. The space station remains stationary in the background.
```

### 8.3 参数调整

| 参数                | 推荐值  | 理由                                 |
| ------------------- | ------- | ------------------------------------ |
| `steps`             | 50-80   | 需要更多步数保证首末帧之间的平滑过渡 |
| `cfg`               | 3.0-3.5 | 与普通图生视频相同                   |
| `frame_consistency` | 较高    | 确保首末帧特征保持                   |

---

## 九、高级优化技巧

### 9.1 显存优化策略矩阵

根据你的显卡选择方案：

| 显存  | 推荐方案              | 关键设置                     |
| ----- | --------------------- | ---------------------------- |
| 12GB  | GGUF Q3 + VAE Offload | 启用 `--novram` 启动参数     |
| 16GB  | GGUF Q4 或 fp8        | 两阶段工作流，阶段一半分辨率 |
| 24GB  | fp8 或 BF16           | 两阶段工作流 + 全分辨率      |
| 32GB+ | BF16                  | 批量生成，高分辨率           |

### 9.2 VAE Offload 设置（减少显存峰值）

当显存不足时，可将 VAE 解码过程卸载到 CPU：

【方法】：在 ComfyUI 启动命令中添加：
```bash
python main.py --novram
```

或在节点层面：使用 `VAEDecodeTiled` 而非 `VAEDecode`，并保持 `tile_size=512`。

### 9.3 RTX 视频超分辨率升频（可选）

此节点可将生成的低分辨率视频硬件升频至 4K，仅限 RTX 显卡。

**【安装】**：ComfyUI Manager → 搜索 "RTX Video Super Resolution" → 安装

**【节点位置】**：放在 VAEDecode 之后、SaveVideo 之前

【连接】：
- `images` ← 来自 VAEDecode 的 `IMAGE`
- `upscale_images` → 连接到 SaveVideo 的 `images`

【参数】：
- `scale`: 2（1080p→4K）或 3（720p→4K）
- `quality`: ULTRA

> 💡 **提示**：此节点使用 TensorRT 加速，速度极快，几秒内完成 4K 升频。

### 9.4 首帧过烤问题修复（LatentCut + LatentConcat）

使用图生视频时，有时首帧会被过度修改（称为"过烤"）。解决方案：

**【技巧】**：使用 `LatentCut` 和 `LatentConcat` 节点将首帧潜空间与生成潜空间拼接

具体设置较复杂，核心思路：
1. 从 EmptyLatent 获取首帧潜空间
2. 使用 LatentCut 提取首帧
3. 使用 LatentConcat 将首帧与后续生成帧拼接
4. 确保首帧在采样过程中保持固定

---

## 十、故障排除指南

### 10.1 常见错误及解决方案

| 错误现象                             | 原因                           | 解决方案                                   |
| ------------------------------------ | ------------------------------ | ------------------------------------------ |
| `Value not in list`                  | 模板中的模型下拉框未找到模型   | 检查模型是否放在正确目录，刷新节点列表     |
| 节点显示红色，报 `Missing Node Type` | 自定义节点未安装               | 按第二章安装 ComfyUI-LTXVideo              |
| OOM during decode                    | VAE 解码时显存不足             | 使用 VAE Decode Tiled 并降低 tile_size     |
| Gemma 编码器崩溃                     | 模型文件存放路径错误或文件损坏 | 确认文件在 `text_encoders/` 目录，重新下载 |
| 生成的视频全黑或全绿                 | VAE 连接错误                   | 确保 VAE 端口正确连接                      |
| 运动不连贯/抖动                      | steps 太少或 cfg 过高          | steps 增加到 40+，cfg 降低到 3.0-3.5       |
| 首帧被过度修改                       | image_noise_scale 过高         | 降低 image_noise_scale 到 0.1-0.2          |

### 10.2 质量优化检查清单

如果视频质量不理想，按顺序检查：

- [ ] 分辨率宽高是 32 的倍数吗？
- [ ] length 满足 8×N+1 吗？
- [ ] steps 是否在推荐范围内？（文生视频 30-50，图生视频 40-60）
- [ ] cfg 是否在 2.0-5.0 之间？
- [ ] 提示词是否具体描述了运动/场景？
- [ ] 是否使用了两阶段工作流？（高质量要求时）
- [ ] 显存是否充足？（检查是否触发了显存交换）

### 10.3 获取帮助的渠道

- ComfyUI-LTXVideo GitHub Issues
- LTX 官方 Discord
- 哩布哩布 AI 社区工作流分享

---

## 📋 附：全文档速查表

### 一、节点速查（中英文对照）

| 中文名称                  | 英文节点名                 | 所在菜单路径             | 所属工作流 |
| ------------------------- | -------------------------- | ------------------------ | ---------- |
| Checkpoint 加载器（简易） | CheckpointLoaderSimple     | 右键→checkpoint          | A/B/C/D    |
| Gemma 文本编码器加载器    | LTXTextEncoderLoader       | 右键→LTXText             | A/B/C/D/E  |
| 空视频潜空间              | EmptyLTXVLatentVideo       | 右键→EmptyLTXVLatent     | A/B/C      |
| LTX 文生视频采样器        | LTXVTextToVideoSampler     | 右键→LTXVTextToVideo     | A/C        |
| LTX 图生视频采样器        | LTXVImageToVideoSampler    | 右键→LTXVImageToVideo    | B/E        |
| LTX 潜空间放大            | LTXVLatentUpsampler        | 右键→LTXVLatentUpsampler | C          |
| 潜空间放大模型加载器      | LatentUpscaleModelLoader   | 右键→LatentUpscale       | C          |
| LoRA 加载器（仅模型）     | LoraLoaderModelOnly        | 右键→LoraLoader          | C          |
| 分块 VAE 解码             | VAEDecodeTiled             | 右键→VAEDecodeTiled      | A/B/C      |
| Unet 加载器（GGUF）       | Unet Loader (GGUF)         | 右键→GGUF→Unet Loader    | D          |
| 保存视频                  | SaveVideo                  | 右键→SaveVideo           | A/B/C/E    |
| RTX 视频超分辨率          | RTX Video Super Resolution | 右键→RTX Video           | 优化       |

### 二、自定义节点清单

| 节点名称             | GitHub 地址                                    | 安装方式            | 是否需要代理 | 所属工作流 |
| -------------------- | ---------------------------------------------- | ------------------- | ------------ | ---------- |
| ComfyUI-LTXVideo     | https://github.com/Lightricks/ComfyUI-LTXVideo | Manager / git clone | 是（有镜像） | 全部       |
| ComfyUI-GGUF         | https://github.com/city96/ComfyUI-GGUF         | Manager / git clone | 是（有镜像） | D          |
| ComfyUI-RTX-Video-SR | 内置 Manager 源                                | Manager             | 否           | 优化       |

### 三、模型下载清单

| 模型名称                                    | 来源平台    | 存放路径               | 是否需要代理 | 所属工作流 |
| ------------------------------------------- | ----------- | ---------------------- | ------------ | ---------- |
| ltx-2.3-22b-dev-fp8.safetensors             | HuggingFace | checkpoints/           | 是（有镜像） | 全部       |
| gemma_3_12B_it_fp4_mixed.safetensors        | HuggingFace | text_encoders/         | 是（有镜像） | 全部       |
| ltx-2.3-spatial-upscaler-x2-1.0.safetensors | HuggingFace | latent_upscale_models/ | 是（有镜像） | C          |
| ltx-2.3-22b-distilled-lora-384.safetensors  | HuggingFace | loras/                 | 是（有镜像） | C          |
| sulphur_dev-Q4_K_M.gguf                     | HuggingFace | unet/                  | 是（有镜像） | D          |

### 四、网络环境处理清单

| 操作/资源                      | 问题类型          | 处理方案（含替代方案）                                |
| ------------------------------ | ----------------- | ----------------------------------------------------- |
| 从 HuggingFace 下载模型        | 下载失败/速度极慢 | 使用 hf-mirror.com 镜像站；或哩布哩布 AI 搜索搬运资源 |
| 安装 ComfyUI-LTXVideo          | GitHub 访问失败   | 手动下载 ZIP 解压到 custom_nodes/；或使用代理         |
| ComfyUI Manager 安装插件       | 默认源在国外      | 配置 ComfyUI Manager 镜像源设置；或手动 git clone     |
| 访问 raw.githubusercontent.com | 无法访问          | 使用代理；或通过 GitHub 页面手动下载后放置            |

### 五、关键参数速查

| 工作流场景   | 节点                    | 参数              | 推荐值    | 理由                     |
| ------------ | ----------------------- | ----------------- | --------- | ------------------------ |
| 文生视频     | LTXVTextToVideoSampler  | steps             | 30-50     | 保证运动连贯性           |
| 文生视频     | LTXVTextToVideoSampler  | cfg               | 3.0-4.0   | 避免运动僵硬             |
| 图生视频     | LTXVImageToVideoSampler | steps             | 40-60     | 需要更多步数平滑首帧过渡 |
| 图生视频     | LTXVImageToVideoSampler | image_noise_scale | 0.1-0.3   | 控制首帧保留程度         |
| 两阶段阶段一 | EmptyLTXVLatentVideo    | width/height      | 640×384   | 半分辨率节省显存         |
| 两阶段阶段一 | LTXVTextToVideoSampler  | steps             | 25-35     | 只需确定构图和运动       |
| 两阶段阶段二 | LTXVLatentUpsampler     | 放大因子          | 2×        | 固定值，不可自定义       |
| 两阶段阶段二 | LTXVTextToVideoSampler  | denoise           | 0.4-0.6   | 细节补充而非重新生成     |
| 蒸馏模式     | LoraLoaderModelOnly     | strength_model    | 0.6-0.8   | 平衡细节和过饱和         |
| 首末帧       | LTXVImageToVideoSampler | steps             | 50-80     | 保证首末帧平滑过渡       |
| 潜空间配置   | EmptyLTXVLatentVideo    | length            | 8×N+1     | 模型步幅约束             |
| 潜空间配置   | EmptyLTXVLatentVideo    | width/height      | 32 的倍数 | 模型步幅约束             |

---

*本文档基于 LTX 2.3（2026年3月发布版本）编写。随着版本更新，部分细节可能变化，请以官方最新文档为准。*