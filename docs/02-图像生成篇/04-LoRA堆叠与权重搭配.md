# LoRA 堆叠与权重搭配

**场景**：同时使用多个 LoRA（如角色 LoRA + 服装 LoRA + 风格 LoRA），需要精确控制每个的影响权重。

## 一、核心原理

ComfyUI 中加载多个 LoRA 的正确方式是**串联**（Chain）而非并联——每个 LoRA 依次修改模型权重。顺序影响结果：先应用的角色 LoRA 权重「内置」进模型，后应用的风格 LoRA 在此基础上叠加。

### 架构

```
Load Checkpoint → LoRA #1 (角色) → LoRA #2 (服装) → LoRA #3 (风格) → KSampler
```

## 二、权重经验法则

| LoRA 位置 | 类型 | 推荐 strength_model | 说明 |
|:---------:|:----:|:-------------------:|------|
| 第1个 | 角色 LoRA | 0.7-1.0 | 最底层，建立特征基础，权重最高 |
| 第2个 | 服装/配饰 LoRA | 0.5-0.7 | 附着于角色之上，权重中等 |
| 第3个 | 风格 LoRA | 0.3-0.5 | 表面覆盖，权重最低 |

## 三、Prompt 写法关键

在多 LoRA 场景下，prompt 中**必须**包含 LoRA 对应的触发词（trigger words）：

```
Prompt: (trigger_word_for_lora1:1.2), (trigger_word_for_lora2:1.0),
        detailed scene description, styling from lora3
```

## 四、规则与禁忌

| 规则 | 说明 |
|------|------|
| ✅ **权重递减** | 最底层（角色）权重最高，越上层越低 |
| ✅ **总权重不超过 2.0** | 多个 LoRA 的 strength_model 之和不宜超过 2.0，否则模型崩溃 |
| ❌ **同类 LoRA 不堆叠** | 不要同时使用两个角色 LoRA，风格会打架 |
| ✅ **LoRA 与 IP-Adapter 共存** | IP-Adapter 的 MODEL 输出再进入 LoRA 链可行 |

## 五、模型下载

| 来源 | 说明 |
|------|------|
| **Civitai** | LoRA 最丰富的平台 |
| **哩布哩布** | 搜索同名 LoRA，下载更快 |
| **存放路径** | `ComfyUI/models/loras/` |
| **文件大小** | 通常 30-150MB |
