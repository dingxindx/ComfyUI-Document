# IP-Adapter 文档修正总结

## 📋 基于实际工作流验证的修正

**验证时间**: 2026-05-28  
**验证来源**: 用户提供的可运行工作流 JSON (`2026-05-28_16_53_02.json`)

---

## 🔴 发现的关键错误

### 错误 1：核心节点名称错误（最严重）

| 项目 | 旧文档 | 修正后 |
|:-----|:-------|:-------|
| **核心节点** | `IPAdapterApply` | `IPAdapterEmbeds` |
| **节点数量** | 未明确 | 5 个节点（LoadImage + CLIPVisionLoader + IPAdapterModelLoader + IPAdapterEncoder + IPAdapterEmbeds） |

**影响**: 用户按照旧文档搜索 `IPAdapterApply` 找不到节点，无法搭建工作流。

---

### 错误 2：缺少 IPAdapterEncoder 节点

| 项目 | 旧文档 | 修正后 |
|:-----|:-------|:-------|
| **编码器节点** | 未提及 | `IPAdapterEncoder`（必需） |
| **输出** | 无 | `pos_embed` 和 `neg_embed` 两个输出 |

**影响**: 用户不知道需要将参考图编码为嵌入向量。

---

### 错误 3：连接方式错误

| 连接 | 旧文档 | 修正后 |
|:-----|:-------|:-------|
| IPAdapterEncoder → IPAdapterEmbeds | 无 | `pos_embed` → `pos_embed`、`neg_embed` → `neg_embed` |
| CLIPVisionLoader | 直接连 Apply | 连到 IPAdapterEncoder 的 `clip_vision` |
| LoadImage | 直接连 Apply | 连到 IPAdapterEncoder 的 `image` |

**影响**: 即使用户找到了正确的节点，也不知道如何连接。

---

### 错误 4：参数缺失

| 参数 | 旧文档 | 修正后 |
|:-----|:-------|:-------|
| `embeds_scaling` | 未提及 | 必需，推荐 `V only` |
| `pos_embed`/`neg_embed` | 未提及 | IPAdapterEncoder 的两个必需输出 |

**影响**: 用户即使连接正确，也可能因为参数设置错误导致效果不佳。

---

### 错误 5：工作流图错误

旧文档的 Mermaid 图显示的是 `IPAdapterApply` 节点，实际应该使用 `IPAdapterEmbeds`。

**修正**: 重绘了完整的工作流图，包含所有 5 个 IP-Adapter 相关节点。

---

## ✅ 修正内容总览

### 1. 节点详解部分

**新增**:
- `IPAdapterModelLoader` 详细说明
- `CLIPVisionLoader` 详细说明
- `IPAdapterEncoder` 详细说明（包含输入输出）
- `IPAdapterEmbeds` 详细说明（核心节点）
  - 输入端口表格
  - `embeds_scaling` 参数详解
  - `pos_embed`/`neg_embed` 连接说明

**删除**:
- `IPAdapter Unified Loader`（虽然存在但不是标准工作流）
- `IPAdapter Advanced`（重命名为 `IPAdapterEmbeds`）

---

### 2. 工作流图部分

**新增**:
- 完整的 Mermaid 流程图（包含 IPAdapterEncoder）
- 完整的连接表格（13 条连接）

**修正**:
- 数据流方向：MODEL → IPAdapterEmbeds → KSampler
- 参考图处理流程：LoadImage → IPAdapterEncoder → IPAdapterEmbeds

---

### 3. 手把手操作部分

**修正前**: 3 步（LoadImage → Unified Loader → KSampler）  
**修正后**: 6 步（包含 5 个节点的完整流程）

**新增**:
- 详细的 13 条连接表格
- IPAdapterEmbeds 参数设置表
- 关键提示：IPAdapterEncoder 的两个输出都要连接

---

### 4. 场景参数速查表

**新增列**: `embeds_scaling`（全部设置为 `V only`）

---

### 5. 常见问题排查

**新增问题**:
- IPAdapterEncoder 报错
- IPAdapterEmbeds 报错
- IPAdapterEmbeds 没有 IMAGE 输入（正常现象说明）
- embeds_scaling 选错

---

### 6. 检查清单

**从 11 项扩展到 16 项**，新增:
- LoadImage → IPAdapterEncoder 连接检查
- CLIPVisionLoader → IPAdapterEncoder 连接检查
- IPAdapterModelLoader → IPAdapterEncoder 连接检查
- IPAdapterEncoder → IPAdapterEmbeds 双连接检查
- embeds_scaling 设置检查

---

## 🎯 验证结果

**用户工作流使用的节点**:
```
✅ CheckpointLoaderSimple
✅ CLIPTextEncode (2 个)
✅ EmptyLatentImage
✅ KSampler
✅ VAE Decode
✅ PreviewImage
✅ LoadImage
✅ CLIPVisionLoader
✅ IPAdapterModelLoader
✅ IPAdapterEncoder
✅ IPAdapterEmbeds
```

**修正后文档覆盖**: 100% ✅

---

## 📝 给读者的建议

1. **严格按照连接表格接线**：特别是 IPAdapterEncoder 的 `pos_embed` 和 `neg_embed` 两个输出都要连接到 IPAdapterEmbeds
2. **embeds_scaling 选择 V only**：这是新版推荐设置，效果更好
3. **CLIP Vision 文件名必须准确**：`CLIP-ViT-H-14-laion2B-s32B-b79K.safetensors`
4. **首次使用 weight=0.7**：然后根据效果微调

---

## 📌 文档版本

- **修正前**: 基于旧版 IPAdapterApply 节点
- **修正后**: 基于新版 IPAdapterEncoder + IPAdapterEmbeds 节点（2025 最新版）
- **验证方式**: 用户实际运行的工作流 JSON 文件

---

**修正完成时间**: 2026-05-28  
**修正者**: AI Assistant  
**验证者**: 用户提供的可运行工作流
