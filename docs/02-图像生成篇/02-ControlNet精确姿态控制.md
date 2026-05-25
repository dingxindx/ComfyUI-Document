# ControlNet 精确姿态控制

> **前置**：你已成功跑通文生图工作流。ControlNet 是在那基础上的"精准控制"能力。

## 一、ControlNet 是什么？

**白话**：你给 AI 一张"参考图"（骨架图、边缘图、深度图），告诉它"按这个姿势/轮廓来画"，但细节让 AI 自由发挥。

**官方定义**：ControlNet 是一种神经网络结构，通过复制 UNET 的 encoder 层来注入额外条件控制。

**三种最常见的 ControlNet 类型**：

| 类型 | 输入 | 控制什么 | 典型场景 |
|:----:|:----|:---------|:---------|
| **OpenPose** | 人体骨骼关键点图 | 人物姿态（四肢、躯干位置） | 保持特定姿势 |
| **Canny Edge** | 边缘检测图（黑白线稿） | 物体轮廓和构图 | 保持整体构图 |
| **Depth（深度图）** | 深度灰度图 | 物体的空间纵深关系 | 保持场景结构 |

---

## 二、前置准备

### 安装所需的节点

```bash
cd ~/workspace/ComfyUI/custom_nodes/
git clone https://gitclone.com/github.com/Fannovel16/comfyui_controlnet_aux.git
pip install -r requirements.txt
```

### 下载 ControlNet 模型

| 模型文件 | 存放路径 | 镜像下载 |
|:---------|:---------|:---------|
| `control_v11p_sd15_openpose.pth` | `models/controlnet/` | hf-mirror.com/lllyasviel/ControlNet-v1-1 |
| `control_v11p_sd15_canny.pth` | `models/controlnet/` | 同上 |

完成后刷新 ComfyUI（F5 或重启）。

---

## 三、工作流结构

```
Load Image ─┬─→ OpenPose Preprocessor ──→ Preview Image (可选)
            │                                ↓
            │                        Apply ControlNet (OpenPose)
            │                                ↓
            └─→ Canny Preprocessor ──────→ Preview Image (可选)
                                                ↓
                                        Apply ControlNet (Canny)
                                                ↓
CheckpointLoaderSimple → CLIP Text → → KSampler (positive)
Empty Latent Image → → → → → → → KSampler (latent)
                                     ↓
                                   VAE Decode → Save Image
```

**关键区别**：ControlNet 不修改 KSampler 的结构，而是"修改 positive conditioning"——在 CLIP 的输出基础上，叠加 ControlNet 的控制信号。

---

## 四、节点详解

### 节点：Load Image（加载图片）

| 参数 | 说明 |
|:-----|:-----|
| 操作 | 右键 → 搜索 "Load Image" |
| 作用 | 加载你的参考姿态图（一张有人物的照片） |
| 输出 | IMAGE（绿色）→ 分叉到两个预处理器 |

### 节点：OpenPose Preprocessor（姿态预处理器）

| 参数 | 推荐值 | 说明 |
|:-----|:------:|:-----|
| `detect_hand` | enable | 检测手指关键点 |
| `detect_body` | enable | 检测躯干 |
| `detect_face` | disable | 面部不需要（除非要控制表情）|
| `resolution` | 512 | 预处理分辨率 |

**输出**是一张"火柴人"骨架图（白线+黑底），预览一下确认姿态提取正确。

### 节点：Canny Preprocessor（边缘预处理器）

| 参数 | 推荐值 | 说明 |
|:-----|:------:|:-----|
| `low_threshold` | 100 | 边缘检测低阈值，越低边缘越多 |
| `high_threshold` | 200 | 高阈值，越高边缘越少 |

**输出**是一张白线黑底的边缘图。

### 节点：ControlNetLoader（加载 ControlNet 模型）

| 参数 | 说明 |
|:-----|:-----|
| 操作 | 右键 → 搜索 "ControlNetLoader" |
| 参数 | `control_net_name` → 下拉选择已下载的 .pth 文件 |

**注意**：OpenPose 和 Canny 各需要一个 ControlNetLoader。

### 节点：Apply ControlNet（应用 ControlNet）

| 参数 | OpenPose 推荐值 | Canny 推荐值 | 说明 |
|:-----|:---------------:|:------------:|:-----|
| `strength` | 0.7-1.0 | 0.5-0.9 | 控制强度。OpenPose 要高以保证姿态准确，Canny 要低以免限制创造力 |
| `start_percent` | 0.0 | 0.0 | 从去噪开始就应用 |
| `end_percent` | 1.0 | 1.0 | 持续到结束 |

**串联连接**（关键！）：

```
CLIP Text (正面) → Apply ControlNet (OpenPose) → Apply ControlNet (Canny) → KSampler.positive
```

即：第一个 Apply ControlNet 的 CONDITIONING 输出接到第二个的 CONDITIONING 输入。

---

## 五、常见问题

| 问题 | 原因 | 解决 |
|:-----|:-----|:-----|
| 姿态不对 | strength 太低或模型不匹配 | 提高 strength 到 0.9+ |
| 画面僵硬变形 | strength 太高 | 降低到 0.7 |
| 预处理器报错 | 缺少模型文件 | 检查 models/controlnet/ 和 models/aux_preprocessors/ |
| 预览全黑 | 预处理器路径不对 | 确认 comfyui_controlnet_aux 已正确安装 |
