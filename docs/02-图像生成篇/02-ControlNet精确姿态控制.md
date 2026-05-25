# ControlNet 精确姿态控制（OpenPose + Canny）

**场景**：生成的人物姿态与参考图完全一致，同时保持画面细节的创意自由度。

## 一、前置依赖

安装自定义节点 `ComfyUI-Advanced-ControlNet`（或 `ComfyUI-ControlNet-Aux`）：

```bash
cd ComfyUI/custom_nodes/
git clone https://github.com/Fannovel16/comfyui_controlnet_aux.git
pip install -r requirements.txt
```

国内镜像：`git clone https://gitclone.com/github.com/Fannovel16/comfyui_controlnet_aux.git`。

### 所需模型下载

| 模型 | 存放路径 | 
|------|---------|
| `control_v11p_sd15_openpose.pth` | `ComfyUI/models/controlnet/` |
| `control_v11p_sd15_canny.pth` | `ComfyUI/models/controlnet/` |

使用 `export HF_ENDPOINT=https://hf-mirror.com` 镜像下载。

## 二、架构概览

```
Load Image ─┬→ OpenPose Preprocessor → Apply ControlNet (OpenPose) ─┐
             │                                                       ↓
             └→ Canny Preprocessor → Apply ControlNet (Canny) → KSampler (positive)
                                                                         ↑
CLIP Text Encode (Prompt) ──────────────────────────────────────────────┘
```

ControlNet OpenPose + Canny 的双通道配置需要**两个 Apply ControlNet 节点串联**：

```
CLIP Text Encode → Apply ControlNet(OpenPose) → Apply ControlNet(Canny) → KSampler(positive)
```

## 三、核心节点

### 节点：OpenPose 预处理器

| 项目 | 说明 |
|------|------|
| **安装来源** | `comfyui_controlnet_aux` |
| **关键参数** | `detect_hand`: enable, `detect_body`: enable, `detect_face`: disable, `resolution`: 512 |

### 节点：ControlNet 应用

| 参数 | 推荐值 | 说明 |
|------|:------:|------|
| `strength` | OpenPose: 0.7-1.0 / Canny: 0.5-0.9 | OpenPose 控制姿态需强信号，Canny 控制轮廓需适中 |
| `start_percent` | 0.0 | 从去噪开始就应用 |
| `end_percent` | 1.0 | 全程应用 |

## 四、工作原理

- **OpenPose** 控制「姿态」（骨骼结构），确保人物姿势正确
- **Canny** 控制「轮廓」（边缘构图），确保物体位置和形状准确
- OpenPose 的 `strength` 通常设高（~0.9），Canny 设适中（~0.6），否则画面过于僵硬
