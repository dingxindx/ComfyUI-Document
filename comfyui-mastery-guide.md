# ComfyUI 实战精通指南：从入门到大师

> 本文档面向已具备基础 ComfyUI 操作能力的用户（了解安装、节点连接、能跑通简单文生图），目标是达到"精通"级别——自主从 HuggingFace、Civitai、哩布哩布 AI 三大社区获取资源，构建并优化任意工作流。

---

## 📖 目录

- [一、入门篇：核心设计哲学](#一入门篇核心设计哲学)
- [二、工作流模板系统（Workflow Templates）](#二工作流模板系统workflow-templates)
- [三、进阶篇：社区资源驱动的可控生成](#三进阶篇社区资源驱动的可控生成)
- [四、高级篇：视频生成与复杂逻辑](#四高级篇视频生成与复杂逻辑)
- [📋 附：全文档速查表](#-附全文档速查表)

---

## 一、入门篇：核心设计哲学

### 1.1 环境安装与启动（快速回顾）

**前提**：已安装 Python 3.10+ 和 Git。

```bash
# 克隆仓库
git clone https://github.com/comfyanonymous/ComfyUI.git
cd ComfyUI

# 创建虚拟环境（推荐）
python -m venv venv
source venv/bin/activate  # macOS/Linux
# venv\Scripts\activate   # Windows

# 安装依赖
pip install -r requirements.txt

# 启动
python main.py
```

> ⚠️ **网络提示**：`git clone https://github.com/...` 和 `pip install -r requirements.txt` 都需要访问 GitHub 和 PyPI。国内用户建议：
> - **GitHub 镜像**：将 `github.com` 替换为 `gitclone.com` 或 `mirror.ghproxy.com/https://github.com/`
> - **PyPI 镜像**：`pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple`
> - **ComfyUI 国内社区打包版**：访问 哩布哩布 (liblibai.com) 搜索"ComfyUI整合包"，直接下载免配置版本

启动后浏览器访问 `http://127.0.0.1:8188` 即可看到画布界面。

### 1.2 核心设计哲学（理解才能精通）

ComfyUI 与 WebUI (AUTOMATIC1111) 的本质区别在于三个设计决策：

#### ① 潜空间（Latent Space）—— 一切生成的中枢

Stable Diffusion 不在像素空间（RGB像素）中做计算，而是在**压缩了 48 倍的潜空间**中工作。流程如下：

```
像素图 → VAE编码器 → 潜空间张量 → UNet 去噪 → VAE解码器 → 像素图
```

> 💡 **提示**：理解潜空间意味着你明白为什么 VAE 节点（VAELoader / VAEEncode / VAEDecode）是任何工作流的起点和终点。VAE 是连接像素世界和潜空间的桥梁。

#### ② 节点数据流 —— 有向无环图（DAG）

ComfyUI 的每个节点是一个纯函数：接收输入张量 → 处理 → 输出张量。数据沿着连线单向流动，**不允许循环**（这就是为什么它叫"无环"图）。

关键推论：
- **执行顺序由依赖关系决定**，而非画布上的位置
- 一个输出可以分叉给多个节点（数据复用）
- 同一个输入端口只能接收一根连线

#### ③ 队列机制 —— 批处理与中断

- **Queue Prompt（添加到队列）**: 将当前工作流加入执行队列
- **Queue Prompt (Batch)**: 批量生成 N 张，自动随机种子
- **Interrupt（中断）**: 立即停止当前生成，结果保留已完成的步数

> 💡 **提示**：SSD/HDD 空间是隐性瓶颈。每个工作流会在 `output/` 目录生成文件，长时间运行视频生成注意磁盘空间。

---

## 二、工作流模板系统（Workflow Templates）

在进入具体工作流之前，有必要先介绍 ComfyUI 内置的**工作流模板系统**——这是快速上手新模型（尤其是 LTX、Wan 系列）的最短路径。

### 2.1 打开方式

**顶部菜单栏 → `Workflow` → `Browse Workflow Templates`**

### 2.2 核心功能

| 功能 | 说明 |
|------|------|
| **一键加载官方工作流** | ComfyUI 官方为每个主流模型（SDXL、Flux、LTX Video、Wan2.2 等）提供了预构建的工作流 |
| **自动检测缺失模型** | 加载模板后，若缺少模型文件，系统弹窗提示下载（含 HuggingFace 直链） |
| **自动分类展示** | 已安装的自定义节点的示例工作流会自动归类展示 |

### 2.3 使用策略

> ⚠️ **网络提示**：Workflow Templates 中的模型下载链接直指 HuggingFace。国内用户：
> - **推荐做法**：使用模板的节点结构，但从 哩布哩布 或 HuggingFace 镜像站（`hf-mirror.com`）下载模型文件后手动放入 `models/` 目录
> - **替代做法**：浏览器打开模板中提示的下载 URL，用 IDM/aria2 等工具下载，再放到对应目录

**千万不要**直接点击弹窗中的下载按钮如果网络不通畅——模型会下载到一半断掉，导致文件损坏。正确流程：

1. 打开 Workflow Template → 观察弹窗提示的模型文件名和路径
2. 手动从国内镜像下载相同文件
3. 放入正确目录后重启 ComfyUI
4. 再次加载模板 → 不再弹窗提示

---

## 三、进阶篇：社区资源驱动的可控生成

### 3.1 模型社区速览

| 平台 | 特点 | 获取方式 | 典型用途 |
|------|------|---------|---------|
| **HuggingFace** | 最全的基座模型 + 官方权重 | `git lfs` / 浏览器下载 | 基础模型、ControlNet、IP-Adapter |
| **Civitai** | 社区微调模型 + LoRA 最丰富 | 浏览器下载 / API下载 | LoRA、微调 Checkpoint、风格模型 |
| **哩布哩布AI** | 国内平台，高速下载 | 浏览器直接下载 | 国内用户首选，大部分 Civitai 模型有搬运 |

### 3.2 网络环境处理总则

> ⚠️ **网络提示**：以下操作在全文各个工作流中反复出现，遵循同一套策略：
>
> | 操作 | 问题 | 解决方案 |
> |------|------|---------|
> | 从 HuggingFace 下载模型 | 连接超时/速度极慢 | 使用 `hf-mirror.com` 镜像站：`export HF_ENDPOINT=https://hf-mirror.com` 或浏览器直接访问 `hf-mirror.com` |
> | 从 Civitai 下载模型 | 访问不稳定 | 通过 哩布哩布 搜索同名模型；或使用浏览器 + 下载工具（IDM/aria2） |
> | `git clone` GitHub 节点仓库 | raw.githubusercontent.com 域名被限 | 使用 `gitclone.com` 镜像：`git clone https://gitclone.com/github.com/...` |
> | ComfyUI Manager 在线安装 | 默认源在国外 | 方法1：使用 Manager 设置修改镜像源；方法2：手动 git clone 后重启 |
> | 远程 API 调用 | 某些 API 不可访问 | 具体问题具体分析，见各工作流标注 |

---

### 3.3 工作流 A：从 Civitai 下载例图 → 还原文生图工作流

**场景**：你在 Civitai 看到一张惊艳的图，想把它的参数"扒"下来在自己的 ComfyUI 中复现。

#### 步骤 1：从 Civitai 获取参数信息

> ⚠️ **网络提示**：访问 Civitai（civitai.com）需要代理环境。国内备选：在 哩布哩布 搜索图片 ID 或模型名称。
> - 如果不具备代理环境，此步骤需要代理环境，请使用 哩布哩布 替代。

打开 Civitai 上的目标图片 → 点击图片下方的"Show Metadata"或点击右上角"..." → "Download JSON" / "Download Image with Metadata"。

Civitai 图片元数据中包含以下关键信息（以 JSON 形式嵌入在 PNG 的 tEXt 块中）：
- `prompt` / `negative_prompt`
- `steps`, `sampler_name`, `cfg`
- `seed`
- `model`（使用的 checkpoint 名称 + 下载链接）
- 所有 LoRA 及其权重

#### 步骤 2：在 ComfyUI 中搭建对应工作流

> 💡 **提示**：最简单的方法：下载图片（含元数据）→ 将 PNG 文件**直接拖入 ComfyUI 画布**。ComfyUI 会自动读取嵌入的工作流 JSON 并还原节点图。

若图片无法自动还原（或需要手动搭建），按以下模板：

---

【节点名称】：加载 Checkpoint（Load Checkpoint）
【添加方式】：鼠标右键 → 搜索 "Load Checkpoint" → 或从菜单 `add_node → loaders → Load Checkpoint`
【添加位置】：画布最左侧
【输入连接】：无（根节点）
【关键参数及设置理由】：
  - `ckpt_name`: 选择你下载的 checkpoint — 必须与 Civitai 图片使用的模型一致，否则效果天差地别
【输出连接】：
  - `MODEL` → 连接到 KSampler 的 `model` 输入端口
  - `CLIP` → 连接到 CLIP Text Encode (Prompt) 的 `clip` 输入端口
  - `VAE` → 连接到 VAE Decode 的 `vae` 输入端口

---

【节点名称】：CLIP文本编码器（CLIP Text Encode (Prompt)）
【添加方式】：鼠标右键 → 搜索 "CLIP Text Encode"
【添加位置】：Load Checkpoint 节点右侧偏上
【输入连接】：
  - `clip` ← 来自 Load Checkpoint 的 `CLIP` 输出端口
【关键参数及设置理由】：
  - `text`: 从 Civitai 图片元数据中复制的 positive prompt — 这是生成的核心指令
【输出连接】：
  - `CONDITIONING` → 连接到 KSampler 的 `positive` 输入端口

---

【节点名称】：CLIP文本编码器（CLIP Text Encode (Negative Prompt)）
【添加方式】：鼠标右键 → 搜索 "CLIP Text Encode"
【添加位置】：Load Checkpoint 节点右侧偏下
【输入连接】：
  - `clip` ← 来自 Load Checkpoint 的 `CLIP` 输出端口
【关键参数及设置理由】：
  - `text`: 从 Civitai 元数据复制的 negative prompt
【输出连接】：
  - `CONDITIONING` → 连接到 KSampler 的 `negative` 输入端口

---

【节点名称】：潜空间图像（Empty Latent Image）
【添加方式】：鼠标右键 → 搜索 "Empty Latent Image"
【添加位置】：KSampler 节点左侧（或 KSampler 上方亦可，位置不影响执行）
【输入连接】：无（根节点）
【关键参数及设置理由】：
  - `width`: Civitai 图片的宽度（通常是 512/768/1024）
  - `height`: Civitai 图片的高度
  - `batch_size`: 1 — 单次生成一张
【输出连接】：
  - `LATENT` → 连接到 KSampler 的 `latent_image` 输入端口

---

【节点名称】：KSampler（KSampler）
【添加方式】：鼠标右键 → 搜索 "KSampler"
【添加位置】：所有编码器节点的右侧中心位置
【输入连接】：
  - `model` ← 来自 Load Checkpoint 的 `MODEL` 输出端口
  - `positive` ← 来自 CLIP Text Encode (Prompt) 的 `CONDITIONING` 输出端口
  - `negative` ← 来自 CLIP Text Encode (Negative Prompt) 的 `CONDITIONING` 输出端口
  - `latent_image` ← 来自 Empty Latent Image 的 `LATENT` 输出端口
【关键参数及设置理由】：
  - `seed`: 从 Civitai 元数据复制（设为 -1 则随机种子）
  - `steps`: 从 Civitai 元数据复制 — step 数越高细节越多，但回报递减
  - `cfg`: 从 Civitai 元数据复制 — 典型值 7.0；太高会导致色彩过饱和
  - `sampler_name`: 从 Civitai 元数据复制 — 如 `euler` / `dpmpp_2m` 等
  - `scheduler`: 从 Civitai 元数据复制 — 如 `karras` / `normal` / `simple`
  - `denoise`: 1.0 — 文生图不做降噪调整
【输出连接】：
  - `LATENT` → 连接到 VAE Decode 的 `samples` 输入端口

---

【节点名称】：VAE解码（VAE Decode）
【添加方式】：鼠标右键 → 搜索 "VAE Decode"
【添加位置】：KSampler 右侧
【输入连接】：
  - `samples` ← 来自 KSampler 的 `LATENT` 输出端口
  - `vae` ← 来自 Load Checkpoint 的 `VAE` 输出端口
【关键参数及设置理由】：无参数需要调整
【输出连接】：
  - `IMAGE` → 连接到 Save Image / Preview Image 的 `images` 输入端口

---

【节点名称】：保存图像（Save Image）
【添加方式】：鼠标右键 → 搜索 "Save Image"
【添加位置】：VAE Decode 右侧
【输入连接】：
  - `images` ← 来自 VAE Decode 的 `IMAGE` 输出端口
【关键参数及设置理由】：
  - `filename_prefix`: "ComfyUI" — 可自定义前缀识别图片

---

> 💡 **提示**：Civitai 元数据中的 `sampler_name` 可能在 ComfyUI 中名称不同。例如 Civitai 显示 "DPM++ 2M Karras"，对应 ComfyUI 中 `sampler_name = dpmpp_2m` + `scheduler = karras`。Civitai 的 "Euler a" 对应 `sampler_name = euler` + `scheduler = ancestral`。

---

### 3.4 工作流 B：ControlNet（OpenPose + Canny）精确姿态控制

**场景**：你想生成一个人物，其姿态与某张参考图完全一致，同时保持画面细节的创意自由度。

**前置条件**：需要安装自定义节点 `ComfyUI-Advanced-ControlNet`（或使用内置的 ControlNetLoader 节点）。

#### 前置依赖安装

> ⚠️ **网络提示**：以下模型需从 HuggingFace 下载。使用国内镜像：
> ```bash
> # 设置 HuggingFace 镜像环境变量
> export HF_ENDPOINT=https://hf-mirror.com
> # 或直接浏览器访问 hf-mirror.com 搜索下载
> ```
> 
> 所需模型文件及存放路径：
> | 模型 | 存放路径 | 镜像下载链接 |
> |------|---------|-------------|
> | `control_v11p_sd15_openpose.pth` | `ComfyUI/models/controlnet/` | [hf-mirror.com/lllyasviel/ControlNet-v1-1](https://hf-mirror.com/lllyasviel/ControlNet-v1-1) |
> | `control_v11p_sd15_canny.pth` | `ComfyUI/models/controlnet/` | 同上 |
> | `sd_xl_base_1.0.safetensors` | `ComfyUI/models/checkpoints/` | [hf-mirror.com/stabilityai/stable-diffusion-xl-base-1.0](https://hf-mirror.com/stabilityai/stable-diffusion-xl-base-1.0) |
>
> 如果使用哩布哩布：搜索 "ControlNet" 和 "SDXL" 可找到大量搬运模型。

---

#### 工作流节点详细说明

【节点名称】：加载图像（Load Image）
【添加方式】：鼠标右键 → 搜索 "Load Image"
【添加位置】：画布最左侧
【输入连接】：无（根节点）
【关键参数及设置理由】：
  - 选择参考姿态图片 — 这张图将用于提取 OpenPose 骨骼和 Canny 边缘
【输出连接】：
  - `IMAGE` → 连接到 OpenPose 预处理器 的 `image` 输入端口
  - `IMAGE` → 连接到 Canny 预处理器 的 `image` 输入端口
  - `IMAGE` → 连接到 ControlNet 应用节点（可选，作为 IP-Adapter 参考）

---

【节点名称】：OpenPose 预处理器（OpenPose Preprocessor）
> 此节点来自自定义节点 ComfyUI-Advanced-ControlNet 或 ComfyUI-ControlNet-Aux
【安装来源】：GitHub 仓库 `https://github.com/Fannovel16/comfyui_controlnet_aux`
【安装方式】：通过 ComfyUI Manager 搜索 "controlnet_aux" 安装 / 或 git clone：
```bash
cd ComfyUI/custom_nodes/
git clone https://github.com/Fannovel16/comfyui_controlnet_aux.git
pip install -r requirements.txt
```
> ⚠️ **网络提示**：git clone GitHub 仓库需要代理环境。国内备选：
> - 使用 `git clone https://gitclone.com/github.com/Fannovel16/comfyui_controlnet_aux.git`
> - 或从 哩布哩布 搜索 "controlnet_aux" 下载压缩包解压到 `custom_nodes/`
> - 或使用 ComfyUI Manager → Install via Git URL 填入 `https://gitclone.com/github.com/Fannovel16/comfyui_controlnet_aux.git`

【添加方式】：鼠标右键 → 搜索 "OpenPose Preprocessor"
【添加位置】：Load Image 节点右侧
【输入连接】：
  - `image` ← 来自 Load Image 的 `IMAGE` 输出端口
【关键参数及设置理由】：
  - `detect_hand`: "enable" — 检测手部关键点（对精细姿态控制重要）
  - `detect_body`: "enable" — 检测躯干关键点
  - `detect_face`: "disable" — 面部关键点在姿态控制中通常不需要
  - `resolution`: 512 — 预处理分辨率，越高越精细但更慢
【输出连接】：
  - `IMAGE` → 连接到 Preview Image（可选，用于预览骨骼图）

【输出连接】（继续 OpenPose Preprocessor）：
  - `POSE_KEYPOINT` / `IMAGE` → 连接到 ControlNet 应用节点的 `controlnet_conditioning` 输入端口

---

【节点名称】：Canny 预处理器（Canny Preprocessor）
> 内置节点，无需额外安装
【添加方式】：鼠标右键 → 搜索 "Canny" → 选择 "Canny Edge" 或从菜单 `add_node → image → preprocessing → Canny Edge`
【添加位置】：Load Image 节点右侧，OpenPose 预处理器下方
【输入连接】：
  - `image` ← 来自 Load Image 的 `IMAGE` 输出端口
【关键参数及设置理由】：
  - `low_threshold`: 100 — 边缘检测的低阈值，越低边缘越密集
  - `high_threshold`: 200 — 边缘检测的高阈值，越高检测到的边缘越少
  - 推荐值 (100, 200)：在保留主体轮廓的同时过滤噪声
【输出连接】：
  - `IMAGE` → 连接到 Preview Image（可选）

---

【节点名称】：ControlNet 加载器（ControlNetLoader）
【添加方式】：鼠标右键 → 搜索 "ControlNetLoader"
【添加位置】：左侧，Load Checkpoint 附近
【输入连接】：无（根节点）
【关键参数及设置理由】：
  - `control_net_name`: 选择已下载的 ControlNet 模型文件 — OpenPose 用 `control_v11p_sd15_openpose.pth`，Canny 用 `control_v11p_sd15_canny.pth`
【输出连接】：
  - `CONTROL_NET` → 连接到 ControlNet 应用节点的 `control_net` 输入端口

---

【节点名称】：ControlNet 应用（Apply ControlNet）
> 内置节点
【添加方式】：鼠标右键 → 搜索 "Apply ControlNet"
【添加位置】：KSampler 节点左侧，ControlNetLoader 和预处理器节点的中间
【输入连接】：
  - `conditioning` ← 来自 CLIP Text Encode (Prompt) 的 `CONDITIONING` 输出端口（正向提示词条件）
  - `control_net` ← 来自 ControlNetLoader 的 `CONTROL_NET` 输出端口
  - `image` ← 来自 预处理器（OpenPose 或 Canny）的 `IMAGE` 输出端口
【关键参数及设置理由】：
  - `strength`: 0.8 — ControlNet 的影响力。0.8 意味着模型 80% 遵从控制信号，20% 自由发挥；OpenPose 推荐 0.7-1.0，Canny 推荐 0.5-0.9
  - `start_percent`: 0.0 — 从去噪最开始就应用控制
  - `end_percent`: 1.0 — 在去噪全部过程中持续应用控制
【输出连接】：
  - `CONDITIONING` → 连接到 KSampler 的 `positive` 输入端口

---

**关键架构说明**：ControlNet OpenPose + Canny 的双通道配置需要**两个 Apply ControlNet 节点串联**：

```
CLIP Text Encode (Prompt) → Apply ControlNet (OpenPose) → Apply ControlNet (Canny) → KSampler (positive)
                                                        ↑                              ↑
                                              ControlNetLoader(openpose)     ControlNetLoader(canny)
                                              OpenPose Preprocessor          Canny Preprocessor
                                              ↑                              ↑
                                              Load Image (same image)
```

具体操作：
1. 搭建第一个 Apply ControlNet：连接 OpenPose 的处理链
2. 将第一个 Apply ControlNet 的 `CONDITIONING` 输出连接到第二个 Apply ControlNet 的 `conditioning` 输入
3. 第二个 Apply ControlNet 连接 Canny 的处理链
4. 第二个 Apply ControlNet 的 `CONDITIONING` 输出连接到 KSampler 的 `positive`

---

> 💡 **提示**：OpenPose 控制"姿态"（骨骼结构），Canny 控制"轮廓"（边缘构图）。两者互补：OpenPose 确保人物的姿势正确，Canny 确保物体的位置和形状准确。OpenPose 的 `strength` 通常设高（~0.9），Canny 设适中（~0.6），否则画面会过于僵硬。

---

### 3.5 工作流 C：IP-Adapter 单图风格迁移

**场景**：你有一张参考图（如某位画家的风格），想用它的风格生成新内容，但不需要训练 LoRA。

**前置依赖**：需要安装自定义节点 `ComfyUI-IPAdapter-Plus` 或使用内置版本（ComfyUI 0.2.0+ 已内置 IP-Adapter 支持）。

#### 前置依赖安装

> ⚠️ **网络提示**：IP-Adapter 模型需从 HuggingFace 下载。国内镜像：
> ```bash
> export HF_ENDPOINT=https://hf-mirror.com
> ```
> 
> 所需模型：
> | 模型 | 存放路径 |
> |------|---------|
> | `ip-adapter_sd15.safetensors` | `ComfyUI/models/ipadapter/` |
> | `ip-adapter_sdxl_vit-h.safetensors` | `ComfyUI/models/ipadapter/` |
> | `image_encoder_model.safetensors` (CLIP ViT-H) | `ComfyUI/models/clip_vision/` |
>
> 镜像下载链接：
> - IP-Adapter: [hf-mirror.com/h94/IP-Adapter](https://hf-mirror.com/h94/IP-Adapter)
> - CLIP Vision: [hf-mirror.com/openai/clip-vit-large-patch14](https://hf-mirror.com/openai/clip-vit-large-patch14)
>
> 哩布哩布替代：搜索"IP-Adapter"找到搬运模型。

---

【节点名称】：加载图像（Load Image）
【添加方式】：鼠标右键 → 搜索 "Load Image"
【添加位置】：画布最左侧
【输入连接】：无（根节点）
【关键参数及设置理由】：
  - 选择风格参考图 — 这是要迁移的"风格来源"
【输出连接】：
  - `IMAGE` → 连接到 CLIP Vision 编码器的 `image` 输入端口

---

【节点名称】：CLIP Vision 加载器（CLIPVisionLoader）
【添加方式】：鼠标右键 → 搜索 "CLIPVisionLoader"
【添加位置】：Load Image 右侧
【输入连接】：无（根节点）
【关键参数及设置理由】：
  - `clip_name`: 选择 `clip-vit-large-patch14.safetensors` 或 `CLIP-ViT-H-14-laion2B-s32B-b79K.safetensors` — IP-Adapter 依赖特定的 CLIP Vision 模型来编码参考图像
【输出连接】：
  - `CLIP_VISION` → 连接到 IP-Adapter 编码器的 `clip_vision` 输入端口

---

【节点名称】：IP-Adapter 编码器（IPAdapterEncoder 或 IPAdapterApply 中的编码部分）
> 如果使用 `ComfyUI-IPAdapter-Plus` 节点包
【安装来源】：GitHub 仓库 `https://github.com/cubiq/ComfyUI_IPAdapter_plus`
【安装方式】：ComfyUI Manager 搜索 "IPAdapter plus" 安装
> ⚠️ **网络提示**：同上，使用 git clone 镜像或 Manager 手动设置镜像源

【添加方式】：鼠标右键 → 搜索 "IPAdapter Encoder"
【输入连接】：
  - `image` ← 来自 Load Image 的 `IMAGE` 输出端口
  - `clip_vision` ← 来自 CLIPVisionLoader 的 `CLIP_VISION` 输出端口
【关键参数及设置理由】：
  - `num_tokens`: 4 — IP-Adapter 使用的 token 数量。4 是原始论文推荐值；16 捕获更精细细节但更占显存
【输出连接】：
  - `ipadapter_embeds` → 连接到 IP-Adapter Apply 节点的 `ipadapter_embeds` 输入端口

---

【节点名称】：IP-Adapter 应用（IPAdapterApply 或 IPAdapterUnifiedLoader）
> 根据节点包不同，可能是 IPAdapterApply 或 IPAdapterUnifiedLoader + IPAdapterApply 的组合
【添加方式】：鼠标右键 → 搜索 "IPAdapter Apply"
【添加位置】：KSampler 左侧，与 Apply ControlNet 同级别
【输入连接】：
  - `model` ← 来自 Load Checkpoint 的 `MODEL` 输出端口（先过 IP-Adapter，再过 KSampler）
  - `ipadapter_embeds` ← 来自 IP-Adapter Encoder 的 `ipadapter_embeds` 输出端口
【关键参数及设置理由】：
  - `weight`: 0.7 — 风格迁移强度。0 为无风格影响，1 为完全复制风格。推荐 0.6-0.8
  - `weight_type`: "linear" — 权重衰减模式。linear、ease in、ease out 等
  - `start_at`: 0.0 — 从去噪开始就应用
  - `end_at`: 1.0 — 全程应用
  - `noise`: 0.0 — 不添加额外噪声
【输出连接】：
  - `MODEL` → 连接到 KSampler 的 `model` 输入端口（替代原来 Load Checkpoint 的直接连接）

---

> 💡 **提示**：IP-Adapter 的关键参数是 `weight`。对于"保留内容但改变风格"的场景（如将照片转为水彩画），`weight` 可设 0.8-1.0 并搭配原始内容的 prompt（如 "a photo of a cat"）。对于"完全改变内容但保留风格"的场景（如用梵高风格画一只猫），`weight` 降低到 0.5-0.7，prompt 写新内容即可。

> 💡 **提示**：IP-Adapter 可以**串联使用**（多个 IP-Adapter Apply 节点串联），每个控制不同的参考图。第二个 IP-Adapter 的 `weight` 设低（~0.3），用于混合多种风格。

---

### 3.6 工作流 D：LoRA 堆叠工作流（多 LoRA 串联与权重搭配）

**场景**：你想同时使用多个 LoRA（例如：一个角色 LoRA + 一个服装 LoRA + 一个风格 LoRA），需要精确控制每个的影响权重。

**核心原理**：ComfyUI 中加载多个 LoRA 的正确方式是**串联**（Chain）而非并联——每个 LoRA 依次修改模型权重。顺序影响结果，先应用的角色 LoRA 权重"内置"进模型，后应用的风格 LoRA 在此基础上叠加。

---

【节点名称】：加载 Checkpoint（Load Checkpoint）
【添加方式】：鼠标右键 → 搜索 "Load Checkpoint"
【添加位置】：画布最左侧
【输出连接】：
  - `MODEL` → 连接到第一个 LoRA 加载器的 `model` 输入端口
  - `CLIP` → 连接到第一个 LoRA 加载器的 `clip` 输入端口

---

【节点名称】：LoRA 加载器 #1（Load LoRA）— 角色 LoRA
【添加方式】：鼠标右键 → 搜索 "Load LoRA"
【添加位置】：Load Checkpoint 右侧
【输入连接】：
  - `model` ← 来自 Load Checkpoint 的 `MODEL` 输出端口
  - `clip` ← 来自 Load Checkpoint 的 `CLIP` 输出端口
【关键参数及设置理由】：
  - `lora_name`: 选择已下载的角色 LoRA 文件（`.safetensors`）
  - `strength_model`: 0.8 — 模型权重强度。角色 LoRA 推荐 0.7-1.0，取决于 LoRA 训练质量
  - `strength_clip`: 0.8 — CLIP 文本编码器权重强度。通常与 `strength_model` 保持一致
  - 理由：角色 LoRA 是最"底层"的，放在第一个以建立特征基础
【输出连接】：
  - `MODEL` → 连接到第二个 LoRA 加载器的 `model` 输入端口
  - `CLIP` → 连接到第二个 LoRA 加载器的 `clip` 输入端口

---

【节点名称】：LoRA 加载器 #2（Load LoRA）— 服装/配饰 LoRA
【添加方式】：鼠标右键 → 搜索 "Load LoRA"
【添加位置】：LoRA #1 右侧
【输入连接】：
  - `model` ← 来自 LoRA #1 的 `MODEL` 输出端口
  - `clip` ← 来自 LoRA #1 的 `CLIP` 输出端口
【关键参数及设置理由】：
  - `lora_name`: 选择服装/配饰 LoRA
  - `strength_model`: 0.6 — 服装 LoRA 权重略低于角色 LoRA，以免遮盖角色特征
  - `strength_clip`: 0.6 — 保持与模型权重一致
  - 理由：服装是附着于角色之上的，权重不宜过高
【输出连接】：
  - `MODEL` → 连接到第三个 LoRA 加载器的 `model` 输入端口
  - `CLIP` → 连接到第三个 LoRA 加载器的 `clip` 输入端口

---

【节点名称】：LoRA 加载器 #3（Load LoRA）— 风格 LoRA
【添加方式】：鼠标右键 → 搜索 "Load LoRA"
【添加位置】：LoRA #2 右侧
【输入连接】：
  - `model` ← 来自 LoRA #2 的 `MODEL` 输出端口
  - `clip` ← 来自 LoRA #2 的 `CLIP` 输出端口
【关键参数及设置理由】：
  - `lora_name`: 选择风格 LoRA（如水彩、像素、赛博朋克风格）
  - `strength_model`: 0.4 — 风格 LoRA 放在最后，权重最低（0.3-0.5），仅做表面风格覆盖
  - `strength_clip`: 0.4 — 保持与模型权重一致
  - 理由：风格 LoRA 影响面最广，设置过高会破坏角色和服装的细节
【输出连接】：
  - `MODEL` → 连接到 KSampler 的 `model` 输入端口
  - `CLIP` → 连接到 CLIP Text Encode (Prompt) 的 `clip` 输入端口

---

**Prompt 写法关键**：在多 LoRA 场景下，prompt 中**必须**包含 LoRA 对应的触发词（trigger words）。LoRA 作者通常会在模型介绍页提供触发词。

```
Prompt: (trigger_word_for_lora1:1.2), (trigger_word_for_lora2:1.0), 
        detailed scene description, styling from lora3
```

> 💡 **提示**：LoRA 堆叠的经验法则：
> 1. **权重递减**：最底层（角色）权重最高，越上层权重越低
> 2. **总权重不超过 2.0**：多个 LoRA 的 `strength_model` 加起来不宜超过 2.0，否则模型崩溃
> 3. **同类 LoRA 不要堆叠**：不要同时使用两个角色 LoRA，风格会打架
> 4. **LoRA 与 IP-Adapter 可共存**：IP-Adapter 应用节点的 MODEL 输出再进入 LoRA 链是可行的

> ⚠️ **网络提示**：LoRA 模型下载：
> - **Civitai**：LoRA 最丰富的平台，搜索你需要的 LoRA → 下载 `.safetensors` 文件
> - **哩布哩布**：直接搜索同名 LoRA，下载速度更快
> - 存放路径：`ComfyUI/models/loras/`
> - LoRA 文件通常 30-150MB，下载耗时短

---

## 四、高级篇：视频生成与复杂逻辑

### 4.1 工作流模板系统的再运用

> 💡 **提示**：对于 LTX Video 和 Wan2.2 这类新模型，**最快捷的入门方式是使用 Workflow Templates**：`Workflow → Browse Workflow Templates` → 搜索 "LTX Video" 或 "Wan2.2" → 加载模板 → 手动下载模型后放对应目录。模板已经搭好了完整的节点链，只需替换模型文件路径即可。

---

### 4.2 工作流 E：LTX Video 2.3 视频生成

**场景**：用 LTX Video 2.3 模型生成高质量视频。LTX 2.3 是 Lightricks 推出的高效视频生成模型，支持文本到视频（T2V）和图像到视频（I2V）。

**核心概念**：LTX 2.3 有三类 checkpoint：`dev`、`fp8`、`distilled`，在不同场景下选择。

| Checkpoint 类型 | 用途 | 显存需求 | 推荐场景 |
|:---:|------|:--------:|---------|
| `dev` | 全精度基础模型 | 24GB+ | 追求最高质量 |
| `fp8` | 8-bit 量化版本 | 12-16GB | 大多数用户首选 |
| `distilled` | 蒸馏版本，4-8 步即可 | 12-16GB | 快速出图/低显存 |

#### 前置条件

> ⚠️ **网络提示**：
> 
> **LTX Video 模型下载**（从 HuggingFace）：
> ```bash
> export HF_ENDPOINT=https://hf-mirror.com
> ```
> - 模型仓库：[hf-mirror.com/Lightricks/LTX-Video](https://hf-mirror.com/Lightricks/LTX-Video)
> - 模型文件：`ltx-video-2b-0.9.1.safetensors`（dev）、`ltx-video-2b-0.9.1-fp8.safetensors`（fp8）
> - 存放路径：`ComfyUI/models/checkpoints/`
>
> **Gemma 3 12B 文本编码器**
> - 模型仓库：[hf-mirror.com/google/gemma-3-12b-it](https://hf-mirror.com/google/gemma-3-12b-it)
> - 注意：Gemma 3 12B 很大（~24GB），需要大量硬盘空间
> - 轻量替代：使用内置的 `t5-v1_1-xxl` 作为文本编码器（质量略低但兼容性好）
> - 存放路径：`ComfyUI/models/text_encoders/`
> - ⚠️ **此步骤需要代理环境**：如果镜像站没有 Gemma 3 的搬运，可能需要直接从 HuggingFace 下载。替代方案是使用 `t5-v1_1-xxl` 编码器（已预设在大多数 ComfyUI 安装中）。

---

#### 两阶段生成工作流

> ⚠️ **网络提示**：如果使用 Workflow Templates 加载 LTX Video 模板，自动检测缺失模型时会指向 HuggingFace。国内用户需手动从 hf-mirror.com 下载后放对应目录。

#### 第一阶段：半分辨率粗生成

【节点名称】：加载 LTX Video Checkpoint（Load Checkpoint）
【添加方式】：鼠标右键 → 搜索 "Load Checkpoint"
【添加位置】：画布最左侧
【关键参数及设置理由】：
  - `ckpt_name`: `ltx-video-2b-0.9.1-fp8.safetensors` — fp8 版本最省显存，适合大多数用户
【输出连接】：
  - `MODEL` → 连接到 ModelSamplingLTXV 的 `model` 输入端口
  - `CLIP` → 连接到 CLIP Text Encode 的 `clip` 输入端口（如果使用 CLIP）
  - `VAE` → 连接到 VAE Decode（LTX Video 内置 VAE，无需单独加载）

---

【节点名称】：LTX 模型采样调度器（ModelSamplingLTXV）
> 用于设置 LTX 视频生成的时间步调度
【添加方式】：鼠标右键 → 搜索 "ModelSamplingLTXV"
【添加位置】：Load Checkpoint 右侧
【输入连接】：
  - `model` ← 来自 Load Checkpoint 的 `MODEL` 输出端口
【关键参数及设置理由】：
  - `shift`: 4.0 — LTX 视频的调制移位参数，推荐 3.0-5.0，影响视频的动态幅度
【输出连接】：
  - `MODEL` → 连接到 KSampler 的 `model` 输入端口

---

【节点名称】：文本编码器 — 使用 Gemma 3 12B 或 T5（Text Encode）
> ⚡ **显存提示**：Gemma 3 12B 编码器占 ~12GB VRAM。如果显存不足（<24GB），强烈建议使用 fp8 或 GGUF Q4 量化版本，或使用内置 T5 编码器替代。
>
> 使用 GGUF Q4 版本的 Gemma 3 需要安装自定义节点 `ComfyUI-GGUF` 并通过它加载量化文本编码器。

```bash
# 安装 ComfyUI-GGUF（如果需要 GGUF 量化支持）
cd ComfyUI/custom_nodes/
git clone https://github.com/city96/ComfyUI-GGUF.git
```

> ⚠️ **网络提示**：`git clone https://github.com/city96/ComfyUI-GGUF.git` 需要代理环境。国内备选：使用 `git clone https://gitclone.com/github.com/city96/ComfyUI-GGUF.git`

【添加方式】：鼠标右键 → 搜索 "T5 Text Encode" 或 "CLIP Text Encode"
【添加位置】：Load Checkpoint 右侧下方
【关键参数及设置理由】：
  - `text`: 详细的视频描述 prompt — 描述动作、场景、运镜方式（如 "A cat walks gracefully across a wooden floor, camera slowly dollying in"）
  - 如果使用 Gemma 3：选择 `text_encoders/gemma-3-12b-it.safetensors`（需要自定义节点支持）
【输出连接】：
  - `CONDITIONING` → 连接到 KSampler 的 `positive` 输入端口

---

【节点名称】：空潜空间视频（Empty Latent Video）
> 这是视频生成专用的潜空间初始化节点
【添加方式】：鼠标右键 → 搜索 "Empty Latent Video"
【添加位置】：KSampler 左侧（或上方）
【输入连接】：无（根节点）
【关键参数及设置理由】：
  - `width`: 384 — 半分辨率宽度（LTX 推荐的低分辨率初稿尺寸）
  - `height`: 256 — 半分辨率高度
  - `length`: 49 — 生成帧数（约 2 秒 24fps 视频）
  - `batch_size`: 1
  - 理由：第一阶段用低分辨率生成粗糙视频内容，节省显存并减少伪影
【输出连接】：
  - `LATENT` → 连接到 KSampler 的 `latent_image` 输入端口

---

【节点名称】：KSampler（KSampler）
【添加方式】：鼠标右键 → 搜索 "KSampler"
【添加位置】：中心位置
【输入连接】：
  - `model` ← 来自 ModelSamplingLTXV 的 `MODEL` 输出端口
  - `positive` ← 来自 文本编码器 的 `CONDITIONING` 输出端口
  - `latent_image` ← 来自 Empty Latent Video 的 `LATENT` 输出端口
【关键参数及设置理由】：
  - `steps`: 30 — LTX 视频的推荐步数；distilled 版本可用 4-8 步
  - `cfg`: 5.0 — LTX Video 推荐 CFG 范围 4.0-7.0
  - `sampler_name`: "euler" — LTX 推荐使用 Euler 采样器
  - `scheduler`: "normal" — 标准调度器
  - `denoise`: 1.0 — 文生视频全部降噪
【输出连接】：
  - `LATENT` → 连接到 LTXV Latent Upsampler 的 `samples` 输入端口

---

#### 第二阶段：潜空间放大

【节点名称】：LTXV 潜空间上采样器（LTXVLatentUpsampler）
> 仅在 ComfyUI 原生支持或通过 Workflow Templates 加载时可用；如果找不到，需确保 ComfyUI 已更新到最新版本
【添加方式】：鼠标右键 → 搜索 "LTXV Latent Upsampler"
【添加位置】：KSampler 右侧
【输入连接】：
  - `samples` ← 来自 KSampler 的 `LATENT` 输出端口
【关键参数及设置理由】：
  - `width`: 768 — 目标全分辨率宽度（2 倍上采样）
  - `height`: 512 — 目标全分辨率高度
  - `length`: 49 — 与第一阶段帧数一致
  - `scale_factor`: 2.0 — 潜空间放大倍数，不经过 VAE 解码/再编码，避免质量损失
【输出连接】：
  - `LATENT` → 连接到 VAE Decode 的 `samples` 输入端口

---

【节点名称】：VAE 解码（VAE Decode）
【添加方式】：鼠标右键 → 搜索 "VAE Decode"
【添加位置】：LTXVLatentUpsampler 右侧
【输入连接】：
  - `samples` ← 来自 LTXVLatentUpsampler 的 `LATENT` 输出端口
  - `vae` ← 来自 Load Checkpoint 的 `VAE` 输出端口
【输出连接】：
  - `IMAGE` → 连接到 Video Combine 的 `images` 输入端口

---

【节点名称】：视频合并（Video Combine）
> 如果安装了 ComfyUI-VideoHelperSuite
【安装来源】：GitHub 仓库 `https://github.com/Kosinkadink/ComfyUI-VideoHelperSuite`
【安装方式】：ComfyUI Manager 搜索 "VideoHelperSuite" 安装
> ⚠️ **网络提示**：同上，使用镜像或手动下载

【添加方式】：鼠标右键 → 搜索 "Video Combine"
【添加位置】：最右侧
【输入连接】：
  - `images` ← 来自 VAE Decode 的 `IMAGE` 输出端口
【关键参数及设置理由】：
  - `frame_rate`: 24 — 输出视频帧率
  - `loop_count`: 0 — 不循环
  - `filename_prefix`: "LTX_Video_"
  - `format`: "video/mp4" — 输出 MP4
【输出连接】：无（输出节点，自动保存文件）

---

#### VAE Offload 显存优化策略

> ⚡ **显存优化**：如果 VRAM 不足（<16GB），启用 VAE Offload：
> 1. 安装节点 `ComfyUI-VideoHelperSuite`（上面已安装）
> 2. 在 KSampler 之前插入 `VAEDecode` 的 offload 模式（或使用系统级选项）
> 3. 设置：`Settings → Memory → VAE offload: enabled`
> 4. 另一种方法：使用 `ComfyUI-Manager` → `GPU memory management` 中的 `Model offload` 选项

---

### 4.3 工作流 F：TTS 语音合成与克隆工作流

**场景**：基于输入的文本生成自然语音，或使用 5 秒参考音频进行语音克隆。

**前置依赖**：自定义节点 `ComfyUI-ChatTTS-OpenVoice`

> ⚠️ **网络提示**：`ComfyUI-ChatTTS-OpenVoice` 插件安装：
> - GitHub 仓库：`https://github.com/AIFSH/ComfyUI-ChatTTS-OpenVoice`
> - **安装方式1**：ComfyUI Manager → "Install via Git URL" → 填入仓库地址
> - **安装方式2**：手动 git clone 到 `custom_nodes/` 目录
> ```bash
> cd ComfyUI/custom_nodes/
> git clone https://github.com/AIFSH/ComfyUI-ChatTTS-OpenVoice.git
> pip install -r requirements.txt
> ```
> - **国内备选**：从 哩布哩布 搜索 "ComfyUI-ChatTTS-OpenVoice" 下载压缩包解压到 `custom_nodes/`
> - ⚠️ 此插件安装 `requirements.txt` 时的 pip 下载可能需要代理或使用国内 PyPI 镜像

---

#### 工作流 F1：文本转语音（基础版）

【节点名称】：ChatTTS 节点（ChatTTSNode）
> 注意：实际节点名可能为 `ChatTTS`、`ChatTTSLoader` 或 `ChatTTSGenerate`
【添加方式】：鼠标右键 → 搜索 "ChatTTS"
【添加位置】：画布中央
【输入连接】：无（或者可能有模型加载端口）
【关键参数及设置理由】：
  - `text`: 输入要合成的文本 — 中文语音合成效果优秀，支持中英混合
  - `audio_seed`: 42 — 控制音色随机种子，固定种子获得一致的音色
  - `temperature`: 0.3 — 采样温度，控制输出多样性；越低越稳定，推荐 0.1-0.5
  - `top_P`: 0.7 — 核采样参数
  - `top_K`: 20 — Top-K 采样参数
【输出连接】：
  - `AUDIO` → 连接到 Save Audio 节点的 `audio` 输入端口

---

【节点名称】：保存音频（Save Audio）
【添加方式】：鼠标右键 → 搜索 "Save Audio"
【添加位置】：ChatTTS 节点右侧
【输入连接】：
  - `audio` ← 来自 ChatTTS 节点的 `AUDIO` 输出端口
【关键参数及设置理由】：
  - `filename_prefix`: "TTS_Output_"
【输出连接】：无

---

#### 工作流 F2：语音克隆（5 秒参考音频）

> 💡 **提示**：语音克隆需要一段 3-10 秒的目标说话人音频样本。克隆质量受参考音频质量影响很大——背景噪声越少、说话越清晰，克隆效果越好。

在 F1 的基础上添加以下节点：

【节点名称】：加载音频（Load Audio）
【添加方式】：鼠标右键 → 搜索 "Load Audio"
【添加位置】：ChatTTS 节点左侧
【输入连接】：无（根节点）
【关键参数及设置理由】：
  - `audio_file`: 选择参考音频文件 — 5 秒左右、清晰无噪、单一说话人
【输出连接】：
  - `AUDIO` → 连接到 ChatTTS 节点的 `ref_audio` 或 `audio_conditioning` 输入端口

---

【节点名称】：ChatTTS 节点（带语音克隆配置）
【添加方式】：同上
【输入连接】：
  - `ref_audio` ← 来自 Load Audio 的 `AUDIO` 输出端口
  - `text`: 目标文本
【关键参数及设置理由】：
  - `emotion_strength`: 0.5 — **情感强度参数（0-1）**：0 为中性/平静，0.3-0.5 为略带感情的自然朗读，0.7-1.0 为带有明显情绪。推荐 0.3-0.6 以保持自然
  - `audio_seed`: 42 — 固定音色种子
  - `temperature`: 0.3 — 稳定性控制
  - `top_P`: 0.7
  - `top_K`: 20
【输出连接】：
  - `AUDIO` → 连接到 Save Audio 节点的 `audio` 输入端口

---

> 💡 **提示**：`emotion_strength` 参数实验：
> - `0.0`：完全中性，适合新闻播报
> - `0.3-0.5`：自然有感情，适合日常对话
> - `0.7-0.9`：情绪明显，适合朗读情感强烈的段落（诗歌、独白）
> - `1.0`：最大情感表达，可能过于夸张，根据场景谨慎使用

---

### 4.4 工作流 G：Wan 2.2 视频生成

**场景**：使用 Wan 2.2 系列模型生成高质量视频，区分 T2V（文本到视频）、I2V（图像到视频）和 S2V（声音驱动视频）三个方向。

**模型概述**：

| 模型 | 参数量 | 功能 | 显存需求 |
|------|:------:|------|:--------:|
| Wan 2.2 5B T2V | 5B | 文本 → 视频 | 12-16GB fp8 |
| Wan 2.2 5B I2V | 5B | 图像 + 文本 → 视频 | 12-16GB fp8 |
| Wan 2.2 14B S2V | 14B | 声音/音频 → 驱动视频 | 32GB+ fp8 |

#### 前置依赖

> ⚠️ **网络提示**：
> 
> **Wan 2.2 模型下载**（从 HuggingFace）：
> - 模型仓库：[hf-mirror.com/Wan-AI/Wan2.1](https://hf-mirror.com/Wan-AI/Wan2.1)
> - 所需文件：
>   - `Wan2_2-T2V-5B.safetensors` 或 `Wan2_2-I2V-5B.safetensors`
>   - `Wan2_2-S2V-14B.safetensors`（仅 S2V 需要）
> - 存放路径：`ComfyUI/models/checkpoints/`
> 
> **自定义节点**：Wan 2.2 有官方 ComfyUI 集成（ComfyUI 0.3.0+），或使用第三方节点 `ComfyUI-Wan`：
> ```bash
> cd ComfyUI/custom_nodes/
> git clone https://github.com/kijai/ComfyUI-Wan-Superresolution.git
> ```
> ⚠️ 此步骤需要代理环境。国内备选：搜索 哩布哩布 上的 "Wan2.2 ComfyUI 节点"

**Workflow Templates 推荐路径**：
> 💡 **提示**：`Workflow → Browse Workflow Templates` → 搜索 "Wan" → 选择 Wan 2.2 Video Generation 模板 → **这是最省力的方式**，官方模板已包含所有节点连接。

---

#### 标准加载链：UNETLoader + CLIPLoader + VAELoader

Wan 2.2 使用的不是传统的 Load Checkpoint 节点，而是**分体加载**架构：

【节点名称】：UNET 加载器（UNETLoader）
> Wan 2.2 的 UNET 与 CLIP、VAE 是分开的文件，需要分别加载
【添加方式】：鼠标右键 → 搜索 "UNETLoader"
【添加位置】：画布最左侧
【关键参数及设置理由】：
  - `unet_name`: `Wan2_2-T2V-5B.safetensors` — 根据任务选择 T2V/I2V/S2V
【输出连接】：
  - `UNET` → 连接到 ModelSamplingSD3 的 `model` 输入端口

---

【节点名称】：CLIP 加载器（CLIPLoader）
【添加方式】：鼠标右键 → 搜索 "CLIPLoader"
【添加位置】：UNETLoader 下方
【关键参数及设置理由】：
  - `clip_name`: `umt5_xxl.safetensors` — Wan 2.2 使用 UMT5 作为文本编码器
  - `type`: "umt5" — 选择 UMT5 类型
【输出连接】：
  - `CLIP` → 连接到 CLIP Text Encode (Prompt) 的 `clip` 输入端口

---

【节点名称】：VAE 加载器（VAELoader）
【添加方式】：鼠标右键 → 搜索 "VAELoader"
【添加位置】：UNETLoader 右侧
【关键参数及设置理由】：
  - `vae_name`: `Wan2_2-VAE.safetensors` — Wan 2.2 专用 VAE
【输出连接】：
  - `VAE` → 连接到 VAE Decode 的 `vae` 输入端口

---

#### ModelSamplingSD3 调度器配置

【节点名称】：SD3 模型采样调度器（ModelSamplingSD3）
> Wan 2.2 使用与 SD3 相同的采样调度方式
【添加方式】：鼠标右键 → 搜索 "ModelSamplingSD3"
【添加位置】：UNETLoader 右侧
【输入连接】：
  - `model` ← 来自 UNETLoader 的 `UNET` 输出端口
【关键参数及设置理由】：
  - `shift`: 4.0 — 调度移位参数，Wan 2.2 推荐使用 3.0-5.0，影响视频的动态幅度和稳定性
【输出连接】：
  - `MODEL` → 连接到 KSampler 的 `model` 输入端口

---

#### 首帧过烤问题修复（LatentCut + LatentConcat 技巧）

> ⚠️ **重要**：Wan 2.2 I2V 模式在将参考图像作为首帧时，常出现**首帧过烤（overcooking）**现象——首帧与参考图像不符或过度平滑。以下技巧可修复此问题。

对于 I2V（图像到视频）工作流：

【节点名称】：加载图像（Load Image）
【添加方式】：鼠标右键 → 搜索 "Load Image"
【添加位置】：左侧
【输入连接】：无
【关键参数及设置理由】：
  - 选择作为首帧的参考图像
【输出连接】：
  - `IMAGE` → 连接到 VAE Encode 的 `pixels` 输入端口（用于编码为潜空间）
  - `IMAGE` → 连接到 Preview Image（可选，确认图像正确）

---

【节点名称】：VAE 编码（VAE Encode）
【添加方式】：鼠标右键 → 搜索 "VAE Encode"
【添加位置】：Load Image 右侧
【输入连接】：
  - `pixels` ← 来自 Load Image 的 `IMAGE` 输出端口
  - `vae` ← 来自 VAELoader 的 `VAE` 输出端口
【输出连接】：
  - `LATENT` → 连接到 LatentCut 的 `samples` 输入端口

---

【节点名称】：潜空间裁剪（LatentCut）
> 需要自定义节点 `ComfyUI-KJNodes` 或类似节点
【安装来源】：需要安装 `ComfyUI-KJNodes` 或 `ComfyUI-VideoHelperSuite`
> ⚠️ **网络提示**：GitHub 仓库 `https://github.com/kijai/ComfyUI-KJNodes` — 使用方式同上

【添加方式】：鼠标右键 → 搜索 "Latent Cut"
【输入连接】：
  - `samples` ← 来自 VAE Encode 的 `LATENT` 输出端口
【关键参数及设置理由】：
  - 将潜空间裁剪到与 Empty Latent Video 匹配的空间尺寸
【输出连接】：
  - `LATENT` → 连接到 LatentConcat 的 `samples1` 输入端口

---

【节点名称】：潜空间拼接（LatentConcat）
【添加方式】：鼠标右键 → 搜索 "Latent Concat"
【输入连接】：
  - `samples1` ← 来自 LatentCut 的 `LATENT` 输出端口（首帧潜空间）
  - `samples2` ← 来自 Empty Latent Video 的 `LATENT` 输出端口（生成的视频潜空间）
【关键参数及设置理由】：
  - 将编码的首帧与生成的视频帧在潜空间维度拼接，确保首帧与原图一致，避免"过烤"
【输出连接】：
  - `LATENT` → 连接到 KSampler 的 `latent_image` 输入端口

---

> 💡 **提示**：LatentCut + LatentConcat 技巧的核心原理是：在 I2V 模式下，Wan 模型的潜空间条件中首帧被重复注入多次，导致首帧区域被"过度渲染"（过烤）。通过显式地将参考图像的潜空间编码结果与生成视频的潜空间拼接，模型获得了更精确的首帧信号，从而消除过烤。

---

#### 轻量路径：Lightning LoRA（4 步采样 + CFG ≈ 1.0）

> ⚡ **性能优化**：对于低显存或需要快速迭代的场景，使用 Lightning LoRA 可大幅加速：
> - 优点：4 步即可生成较高质量视频，显存降低约 30%
> - 缺点：质量和多样性略低于全步数版本
> - 适用场景：快速原型、低显存环境（8-12GB）、批量实验

> ⚠️ **网络提示**：Lightning LoRA 文件需要从 HuggingFace 下载：
> - 模型仓库：[hf-mirror.com/Wan-AI/Wan2.1-Lightning](https://hf-mirror.com/Wan-AI/Wan2.1-Lightning)
> - 下载 `wan2.2-5b-lightning-lora.safetensors`
> - 存放路径：`ComfyUI/models/loras/`
> - 如果镜像站没有，请直接从 HuggingFace 下载（需代理环境）

配置变更（只需修改 KSampler 参数并添加 LoRA 加载器）：

在 UNETLoader → ModelSamplingSD3 → KSampler 的链路中插入 LoRA：

【节点名称】：LoRA 加载器（Load LoRA）— Lightning LoRA
【添加方式】：鼠标右键 → 搜索 "Load LoRA"
【添加位置】：ModelSamplingSD3 和 KSampler 之间
【输入连接】：
  - `model` ← 来自 ModelSamplingSD3 的 `MODEL` 输出端口
【关键参数及设置理由】：
  - `lora_name`: `wan2.2-5b-lightning-lora.safetensors`
  - `strength_model`: 1.0 — 全强度应用
【输出连接】：
  - `MODEL` → 连接到 KSampler 的 `model` 输入端口

KSampler 参数调整（Lightning 模式）：
  - `steps`: 4 — Lightning LoRA 仅需 4 步
  - `cfg`: 1.0 — Lightning 推荐的 CFG 值（范围 1.0-2.0）
  - `sampler_name`: "euler" — Lightning 使用 Euler
  - `scheduler`: "normal"

---

## 📋 附：全文档速查表

### 一、节点速查（中英文对照）

| 中文名称 | 英文节点名 | 所在菜单路径 | 所属工作流 |
|---------|-----------|-------------|-----------|
| 加载 Checkpoint | Load Checkpoint | `add_node → loaders → Load Checkpoint` | A/B/C/D/E |
| CLIP 文本编码器 | CLIP Text Encode (Prompt) | `add_node → conditioning → CLIP Text Encode (Prompt)` | A/B/C/D |
| 潜空间图像 | Empty Latent Image | `add_node → latent → Empty Latent Image` | A/B/C/D |
| KSampler | KSampler | `add_node → sampling → KSampler` | A/B/C/D/E/G |
| VAE 解码 | VAE Decode | `add_node → latent → VAE Decode` | A/B/C/D/E/G |
| 保存图像 | Save Image | `add_node → image → Save Image` | A/B/C/D |
| 加载图像 | Load Image | `add_node → image → Load Image` | B/C/G |
| OpenPose 预处理器 | OpenPose Preprocessor | 自定义节点 `controlnet_aux` | B |
| Canny 边缘 | Canny Edge | `add_node → image → preprocessing → Canny Edge` | B |
| ControlNet 加载器 | ControlNetLoader | `add_node → loaders → ControlNetLoader` | B |
| 应用 ControlNet | Apply ControlNet | `add_node → conditioning → Apply ControlNet` | B |
| CLIP Vision 加载器 | CLIPVisionLoader | `add_node → loaders → CLIPVisionLoader` | C |
| IP-Adapter 编码器 | IPAdapterEncoder | 自定义节点 `ComfyUI_IPAdapter_plus` | C |
| IP-Adapter 应用 | IPAdapterApply | 自定义节点 `ComfyUI_IPAdapter_plus` | C |
| LoRA 加载器 | Load LoRA | `add_node → loaders → Load LoRA` | D/G |
| 空潜空间视频 | Empty Latent Video | 内置节点 | E/G |
| LTX 模型采样调度器 | ModelSamplingLTXV | 内置节点 | E |
| LTXV 潜空间上采样器 | LTXVLatentUpsampler | 内置节点（需更新 ComfyUI） | E |
| 视频合并 | Video Combine | 自定义节点 `VideoHelperSuite` | E |
| ChatTTS 节点 | ChatTTSNode | 自定义节点 `ComfyUI-ChatTTS-OpenVoice` | F |
| 保存音频 | Save Audio | `add_node → audio → Save Audio` | F |
| 加载音频 | Load Audio | `add_node → audio → Load Audio` | F |
| UNET 加载器 | UNETLoader | 内置节点（Wan 专用） | G |
| VAE 加载器 | VAELoader | `add_node → loaders → VAELoader` | G |
| SD3 模型采样调度器 | ModelSamplingSD3 | 内置节点 | G |
| VAE 编码 | VAE Encode | `add_node → latent → VAE Encode` | G |
| 潜空间裁剪 | LatentCut | 自定义节点 `ComfyUI-KJNodes` | G |
| 潜空间拼接 | LatentConcat | 内置节点 | G |

### 二、自定义节点清单

| 节点名称 | GitHub 地址 | 安装方式 | 是否需要代理 | 所属工作流 |
|---------|-----------|---------|-------------|-----------|
| ComfyUI-ControlNet-Aux (controlnet_aux) | `github.com/Fannovel16/comfyui_controlnet_aux` | Manager / git clone | 是（有镜像） | B |
| ComfyUI_IPAdapter_plus | `github.com/cubiq/ComfyUI_IPAdapter_plus` | Manager / git clone | 是（有镜像） | C |
| ComfyUI-VideoHelperSuite | `github.com/Kosinkadink/ComfyUI-VideoHelperSuite` | Manager / git clone | 是（有镜像） | E |
| ComfyUI-ChatTTS-OpenVoice | `github.com/AIFSH/ComfyUI-ChatTTS-OpenVoice` | git clone / 压缩包 | 是（有镜像） | F |
| ComfyUI-KJNodes | `github.com/kijai/ComfyUI-KJNodes` | Manager / git clone | 是（有镜像） | G |
| ComfyUI-GGUF | `github.com/city96/ComfyUI-GGUF` | Manager / git clone | 是（有镜像） | E（可选） |
| ComfyUI-Wan-Superresolution | `github.com/kijai/ComfyUI-Wan-Superresolution` | Manager / git clone | 是（有镜像） | G（可选） |

> ⚠️ **网络提示**：以上所有 GitHub 仓库，在国内均可使用 `git clone https://gitclone.com/github.com/用户名/仓库名.git` 镜像。或在 哩布哩布 搜索同名节点获取压缩包。

### 三、模型下载清单

| 模型名称 | 来源平台 | 存放路径 | 是否需要代理 | 所属工作流 |
|---------|---------|---------|-------------|-----------|
| `sd_xl_base_1.0.safetensors` | HuggingFace | `models/checkpoints/` | 是（有镜像 hf-mirror.com） | B/C/D |
| `control_v11p_sd15_openpose.pth` | HuggingFace | `models/controlnet/` | 是（有镜像） | B |
| `control_v11p_sd15_canny.pth` | HuggingFace | `models/controlnet/` | 是（有镜像） | B |
| `ip-adapter_sd15.safetensors` | HuggingFace | `models/ipadapter/` | 是（有镜像） | C |
| `ip-adapter_sdxl_vit-h.safetensors` | HuggingFace | `models/ipadapter/` | 是（有镜像） | C |
| CLIP ViT-H 图像编码器 | HuggingFace | `models/clip_vision/` | 是（有镜像） | C |
| 各种 LoRA (.safetensors) | Civitai / 哩布哩布 | `models/loras/` | Civitai 需代理 / 哩布免代理 | D |
| `ltx-video-2b-0.9.1-fp8.safetensors` | HuggingFace | `models/checkpoints/` | 是（有镜像） | E |
| LTX VAE（内置） | 随模型一起 | 内置 | — | E |
| Gemma 3 12B IT | HuggingFace | `models/text_encoders/` | **此步骤需要代理环境** | E（可选） |
| `t5-v1_1-xxl` | HuggingFace | `models/text_encoders/` | 是（有镜像） | E（替代方案） |
| ChatTTS 模型 | 自动下载 | 自动 | 是（有镜像） | F |
| OpenVoice 模型 | 自动下载 | 自动 | 是（有镜像） | F |
| `Wan2_2-T2V-5B.safetensors` | HuggingFace | `models/checkpoints/` | 是（有镜像） | G |
| `Wan2_2-I2V-5B.safetensors` | HuggingFace | `models/checkpoints/` | 是（有镜像） | G |
| `Wan2_2-S2V-14B.safetensors` | HuggingFace | `models/checkpoints/` | 是（有镜像） | G（S2V 专用） |
| `Wan2_2-VAE.safetensors` | HuggingFace | `models/vae/` | 是（有镜像） | G |
| `umt5_xxl.safetensors` | HuggingFace | `models/clip/` | 是（有镜像） | G |
| `wan2.2-5b-lightning-lora.safetensors` | HuggingFace | `models/loras/` | 是（有镜像） | G（轻量路径） |

### 四、网络环境处理清单

| 操作/资源 | 问题类型 | 处理方案（含替代方案） |
|-----------|---------|----------------------|
| `git clone` GitHub 仓库 | raw.githubusercontent.com 被限 | 使用 `gitclone.com` 镜像：`git clone https://gitclone.com/github.com/用户名/仓库.git`；或在 哩布哩布 搜索同名节点下载压缩包 |
| 从 HuggingFace 下载模型 | 连接超时/速度慢 | 设置 `export HF_ENDPOINT=https://hf-mirror.com`，或直接访问 `hf-mirror.com` 搜索下载 |
| 从 HuggingFace 下载大型模型（>10GB） | 下载中断 | 使用 IDM/aria2 多线程下载；使用 `hf-mirror.com` 镜像站；或从 哩布哩布 搜索同名模型 |
| 从 Civitai 下载模型/LoRA | 访问不稳定/需代理 | 在 哩布哩布 搜索同名模型（通常有搬运）；或使用浏览器 + 下载工具（IDM/aria2） |
| 访问 Civitai 图片元数据 | 需要代理环境 | **此步骤需要代理环境**，备选：使用 哩布哩布 图片（部分包含元数据） |
| ComfyUI Manager 在线安装插件 | 默认源在国外 | 方法1：Manager 设置中配置镜像源；方法2：手动 git clone 镜像地址；方法3：下载压缩包解压到 `custom_nodes/` |
| `pip install -r requirements.txt` | 外网源慢 | 使用 PyPI 镜像：`pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple` |
| Gemma 3 12B 文本编码器下载 | 大文件下载 | **此步骤需要代理环境**；替代方案：使用内置 T5 编码器 |
| 自动下载的模型（ChatTTS 等） | 首次运行自动下载 | 如果卡住，手动从镜像站下载后放入对应目录，跳过自动下载 |
| Wan 2.2 模型下载（5B/14B） | 大文件下载 | 优先使用 hf-mirror.com 镜像；14B 需 30GB+ 硬盘空间 |
| YouTube / 国外 API 调用 | 不可访问 | 此步骤需要代理环境；部分替代方案使用国内同类服务 |

### 五、关键参数速查

| 场景 | 节点 | 参数 | 推荐值 | 理由 |
|-----|-----|-----|-------|-----|
| 文生图基础 | KSampler | `steps` | 20-30 | SDXL 推荐 20-30 步，超过 40 步回报递减 |
| 文生图基础 | KSampler | `cfg` | 7.0 | SD1.5/SDXL 的标准值，太高过饱和 |
| ControlNet OpenPose | Apply ControlNet | `strength` | 0.7-1.0 | 姿态控制需要较强的信号以保证结构准确 |
| ControlNet Canny | Apply ControlNet | `strength` | 0.5-0.9 | 边缘控制影响面广，过高会限制创造力 |
| ControlNet 双通道 | Apply ControlNet (串联) | 顺序 | OpenPose → Canny | OpenPose 定结构，Canny 定轮廓，串联叠加 |
| IP-Adapter 风格迁移 | IPAdapterApply | `weight` | 0.6-0.8 | 太高复制原图，太低效果不明显 |
| IP-Adapter + 内容变更 | IPAdapterApply | `weight` | 0.5-0.7 | 需要保留风格但改变内容时降低权重 |
| LoRA 堆叠 - 角色 | Load LoRA #1 | `strength_model` | 0.7-1.0 | 角色是基础，权重最高 |
| LoRA 堆叠 - 服装 | Load LoRA #2 | `strength_model` | 0.5-0.7 | 附着于角色之上，权重中等 |
| LoRA 堆叠 - 风格 | Load LoRA #3 | `strength_model` | 0.3-0.5 | 表面覆盖，权重最低 |
| LTX 视频 - 半分辨率 | Empty Latent Video | `width`/`height` | 384×256 | 低分辨率初稿节省显存 |
| LTX 视频 - 潜空间放大 | LTXVLatentUpsampler | `scale_factor` | 2.0 | 绕过 VAE 直接放大，减少质量损失 |
| LTX 视频 - f8 优化 | Load Checkpoint | `ckpt_name` | `ltx-video-...-fp8` | fp8 版显存减半，大多数用户首选 |
| LTX 视频 - distilled | KSampler | `steps` | 4-8 | 蒸馏版只需 4-8 步，适合快速出图 |
| TTS 语音 - 情感强度 | ChatTTSNode | `emotion_strength` | 0.3-0.6 | 0=中性，0.3-0.6=自然，0.7-1.0=强烈 |
| TTS 语音 - 克隆 | ChatTTSNode | 参考音频 | 3-10秒，清晰无噪 | 质量决定克隆效果 |
| Wan 2.2 - I2V 首帧过烤 | LatentCut + LatentConcat | 拼接首帧 | 必须使用 | 消除首帧过度渲染 |
| Wan 2.2 - Lightning | KSampler | `steps` | 4 | Lightning LoRA 仅需 4 步 |
| Wan 2.2 - Lightning | KSampler | `cfg` | 1.0 | Lightning 推荐 CFG≈1.0 |
| Wan 2.2 - 标准生成 | ModelSamplingSD3 | `shift` | 4.0 | 控制视频动态幅度和稳定性 |
| LTX 视频 - 标准生成 | ModelSamplingLTXV | `shift` | 4.0 | 推荐 3.0-5.0，影响视频动态幅度 |

---

> **文档版本**：v1.0 | 最后更新：2025年 | 作者：AI 工作流架构师
>
> **版权声明**：本文档为 ComfyUI 实战学习指南，所有提及的模型和节点的版权归原作者所有。文中提供的国内镜像方案仅用于学习研究，请遵守各平台的服务条款。
>
> **反馈与贡献**：如发现文档错误或需要补充内容，欢迎提交修正。实践是掌握的最好方式——每搭建一个工作流，都尝试理解每个参数为什么那样设置，而不仅仅是"这样能跑"。祝你学习愉快！

---

*"ComfyUI 的精髓不在于记住了多少节点，而在于理解了数据如何在潜空间中流动。"*
