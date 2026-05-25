# LoRA 堆叠与权重搭配

> **场景**：同时使用多个 LoRA（角色 + 服装 + 风格），每个要精确控制影响程度。

## 一、LoRA 是什么？

**白话**：LoRA 是一个很小的模型补丁文件（30-150MB），作用是"微调"大模型的特定方面——比如让模型认识某个角色、学会某种画风。

**与 Checkpoint 的区别**：Checkpoint 是整个模型（2-50GB），LoRA 是一个小补丁（30-150MB）。LoRA 必须绑定一个基座模型才能工作。

**LoRA 文件格式**：`.safetensors`，存放在 `models/loras/` 目录。

## 二、核心原理：串联而非并联

```
Checkpoint → LoRA #1 (角色) → LoRA #2 (服装) → LoRA #3 (风格) → KSampler
```

每个 LoRA 依次修改模型权重，顺序影响结果。**先应用的角色 LoRA 权重内建进模型，后应用的风格 LoRA 在此基础上叠加。**

## 三、权重搭配策略

| 位置 | 类型 | strength_model | 说明 |
|:----:|:----:|:--------------:|:-----|
| 第 1 个 | 角色 LoRA | 0.7-1.0 | 最底层，建立特征基础 |
| 第 2 个 | 服装/配饰 LoRA | 0.5-0.7 | 附着于角色之上 |
| 第 3 个 | 风格 LoRA | 0.3-0.5 | 表面覆盖，权重最低 |

## 四、Prompt 写法

多 LoRA 场景下，prompt 中**必须**包含 LoRA 对应的触发词（trigger words）：

```
Prompt: (trigger_word_for_lora1:1.2), (trigger_word_for_lora2:1.0),
        detailed scene description, styling from lora3
```

触发词通常在 LoRA 下载页面的介绍中找到，必须使用否则 LoRA 无效。

## 五、规则与禁忌

| ✅ 允许 | ❌ 禁止 |
|:--------|:--------|
| 权重递减排列 | 总 strength_model > 2.0 |
| LoRA + IP-Adapter 共存 | 两个同类型 LoRA（两个角色 LoRA）|
| LoRA + ControlNet 共存 | 使用不兼容的基座模型 |
| 蒸馏 LoRA 搭配蒸馏 Checkpoint | 将 SD1.5 的 LoRA 用在 SDXL 上 |

## 六、模型下载

| 来源 | 说明 |
|:-----|:-----|
| Civitai | LoRA 最丰富，需代理 |
| 哩布哩布 | 搜索同名 LoRA，国内直连 |
| 存放 | `models/loras/`（30-150MB/个）|

## 七、常见问题

- **LoRA 没起作用** → 没写触发词，或基座模型不匹配
- **画面崩溃/变花** → 总权重太高（>2.0）
- **角色特征被覆盖** → 权重递减顺序错误，或风格 LoRA 权重设太高
