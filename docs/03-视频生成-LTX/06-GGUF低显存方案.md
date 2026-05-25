# GGUF 低显存方案（LTX 2.3）

对于 12-16GB 显存的显卡，通过 GGUF 量化将显存需求从 ~23GB 降至 ~18GB。

## 一、必需文件

| 模型 | 存放路径 |
|------|---------|
| GGUF 量化模型（如 `sulphur_dev-Q4_K_M.gguf`） | `ComfyUI/models/unet/`（注意：不是 checkpoints 目录！） |

需要安装 `ComfyUI-GGUF` 自定义节点。

## 二、改造官方模板

官方 LTX 模板默认使用 `CheckpointLoaderSimple`，GGUF 模型需用专门的 **Unet Loader (GGUF)** 节点。

### 操作流程

1. 从模板库加载 **Image to Video (LTX-2.3)** 模板
2. 找到模型加载区域（`CheckpointLoaderSimple` 和 `LoraLoader`）
3. 断开原连接：
   ```
   CheckpointLoaderSimple 的 MODEL → LoRA加载器
   ```
4. 添加 **Unet Loader (GGUF)** 节点，选择 GGUF 文件
5. 新连接：
   ```
   Unet Loader (GGUF) 的 model → LoRA加载器（仅模型）
   ```

> ⚠️ 不要删除原来的 CheckpointLoaderSimple！它仍提供 VAE 和 CLIP 路径。
> CLIP 和 VAE 的加载路径**不要动**。

## 三、GGUF 量化级别选择

| 量化级别 | 显存占用 | 质量损失 | 推荐场景 |
|:-------:|:--------:|:--------:|---------|
| Q2_K | ~11GB | 明显 | 12GB 显卡勉强运行 |
| Q3_K_M | ~14GB | 中等 | 12GB 显卡稳定运行 |
| **Q4_K_M** | **~18GB** | **轻微** | **16GB 显卡首选** |
| Q5_K_M | ~22GB | 极小 | 24GB 显卡可选 |
| Q8_0 | ~30GB | 几乎无损 | 24GB+ 显卡 |

> 💡 Q4_K_M 是显存和质量的最佳平衡点，绝大多数 16GB 显卡用户应选择此级别。

## 四、检查清单

- [ ] GGUF 文件放在 `models/unet/` 目录
- [ ] ComfyUI-GGUF 自定义节点已安装
- [ ] 使用 Unet Loader (GGUF) 而非 CheckpointLoaderSimple
- [ ] 原 CheckpointLoaderSimple 保留（不删除）
- [ ] VAE 和 CLIP 连接未被切断
