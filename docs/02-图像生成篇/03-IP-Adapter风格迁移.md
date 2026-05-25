# IP-Adapter 风格迁移

> **场景**：你有一张参考图（如梵高的星空），想把它的"风格"迁移到新图片上，但不需要训练 LoRA。

## 一、IP-Adapter 是什么？

**白话**：给 AI 一张风格参考图，AI 提取这张图的"风格特征"（笔触、色彩、光影），应用到新内容上。

和 ControlNet 的区别：ControlNet 控制"形状/姿态"，IP-Adapter 控制"风格/质感"。

## 二、前置准备

### 安装节点

```bash
cd ~/workspace/ComfyUI/custom_nodes/
git clone https://gitclone.com/github.com/cubiq/ComfyUI_IPAdapter_plus.git
```

### 下载模型

| 文件 | 存放路径 |
|:-----|:---------|
| `ip-adapter_sd15.safetensors` | `models/ipadapter/` |
| `ip-adapter_sdxl_vit-h.safetensors` | `models/ipadapter/` |
| CLIP Vision ViT-H 模型 | `models/clip_vision/` |

镜像下载：`hf-mirror.com/h94/IP-Adapter`

## 三、工作流结构

```
Load Image (风格参考图) ──→ CLIPVisionLoader ──→ IPAdapterEncoder
                                                  ↓
CheckpointLoader → IPAdapterApply → KSampler → VAE Decode → Save Image
```

## 四、关键参数

### IPAdapterApply 参数

| 参数 | 推荐值 | 说明 |
|:-----|:------:|:-----|
| `weight` | 0.6-0.8 | 风格迁移强度。0=无影响，1.0=完全复制风格 |
| `weight_type` | linear | 权重衰减模式 |
| `start_at` | 0.0 | 从去噪开始就应用 |
| `end_at` | 1.0 | 全程应用 |

### 两种场景的参数策略

| 场景 | weight | Prompt 写法 |
|:-----|:------:|:------------|
| 保留内容改风格（照片→水彩画） | 0.8-1.0 | 描述原始内容，如 "a photo of a cat" |
| 改变内容保留风格（梵高画猫） | 0.5-0.7 | 描述新内容，如 "a cat, van gogh style" |

## 五、进阶技巧

- **串联多个 IP-Adapter**：第二个 weight 设低（~0.3），用于混合多种风格
- **IP-Adapter + LoRA 共存**：IP-Adapter Apply 的 MODEL 输出再进 LoRA 加载器
- **IP-Adapter + ControlNet 共存**：ControlNet 改 conditioning，IP-Adapter 改 model，互不冲突
