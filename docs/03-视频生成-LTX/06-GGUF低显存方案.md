# LTX 2.3 GGUF 低显存方案

## 一、什么时候需要？

| 显存 | 情况 |
|:----:|:-----|
| 12-16GB | LTX fp8 可能跑不动，需 GGUF 量化 |
| 16-24GB | 可选 GGUF Q4 省显存做两阶段 |

## 二、所需文件

| 文件 | 存放位置 |
|:-----|:---------|
| GGUF 量化模型（如 `sulphur_dev-Q4_K_M.gguf`） | `models/unet/`（注意不是 checkpoints！）|

需要安装 `ComfyUI-GGUF` 自定义节点。

## 三、改造官方模板

1. Workflow → Browse Workflow Templates → 加载 **Image to Video (LTX-2.3)**
2. 改造连接：

```
原连接：
CheckpointLoaderSimple 的 MODEL → LoRA加载器

改造后：
Unet Loader (GGUF) 的 model → LoRA加载器（仅模型）
```

> ⚠️ **不要删除原来的 CheckpointLoaderSimple！** 它仍提供 VA E 和 CLIP 路径。

## 四、GGUF 量化级别

| 级别 | 显存 | 质量 | 推荐 |
|:----:|:----:|:----:|:----:|
| Q2_K | ~11GB | 明显下降 | 12GB 勉强运行 |
| Q3_K_M | ~14GB | 中等 | 12GB 稳定运行 |
| **Q4_K_M** | **~18GB** | **轻微下降** | **16GB 首选** |
| Q5_K_M | ~22GB | 极小下降 | 24GB 可选 |
| Q8_0 | ~30GB | 几乎无损 | 24GB+ |

## 五、检查清单

- [ ] GGUF 文件放在 `models/unet/` 目录
- [ ] ComfyUI-GGUF 已安装
- [ ] 使用 Unet Loader (GGUF) 加载模型
- [ ] 原 CheckpointLoaderSimple 保留
- [ ] VAE 和 CLIP 连接未被切断
