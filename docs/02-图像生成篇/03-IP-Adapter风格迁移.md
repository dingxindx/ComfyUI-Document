# IP-Adapter 单图风格迁移

**场景**：有一张参考图（如某位画家的风格），用它的风格生成新内容，无需训练 LoRA。

## 一、前置依赖

安装自定义节点 `ComfyUI-IPAdapter-Plus`（ComfyUI 0.2.0+ 已内置部分支持）：

```bash
cd ComfyUI/custom_nodes/
git clone https://github.com/cubiq/ComfyUI_IPAdapter_plus.git
```

### 所需模型

| 模型 | 存放路径 |
|------|---------|
| `ip-adapter_sd15.safetensors` | `ComfyUI/models/ipadapter/` |
| `ip-adapter_sdxl_vit-h.safetensors` | `ComfyUI/models/ipadapter/` |
| CLIP ViT-H 图像编码器 | `ComfyUI/models/clip_vision/` |

## 二、核心架构

```
Load Image → CLIPVisionLoader → IPAdapter Encoder → IPAdapter Apply → KSampler
              ↑                                      ↑                  ↑
           参考图                                weight=0.6-0.8    model (已注入风格)
```

## 三、关键参数

| 参数 | 推荐值 | 说明 |
|------|:------:|------|
| `weight` | 0.6-0.8 | 风格迁移强度；0=无影响，1=完全复制 |
| `weight_type` | linear | 权重衰减模式 |
| `start_at` | 0.0 | 从去噪开始就应用 |
| `end_at` | 1.0 | 全程应用 |

## 四、两种场景的权重策略

| 场景 | weight | 说明 |
|------|:------:|------|
| **保留内容改变风格**（如照片→水彩画） | 0.8-1.0 | 搭配描述原始内容的 prompt |
| **完全改变内容保留风格**（如用梵高画风画猫） | 0.5-0.7 | prompt 写新内容 |

## 五、进阶技巧

- **IP-Adapter 可串联使用**：多个 IP-Adapter Apply 节点串联，每个控制不同的参考图，第二个 weight 设低（~0.3）混合多种风格
- **IP-Adapter 与 LoRA 可共存**：IP-Adapter 应用节点的 MODEL 输出再进入 LoRA 链
