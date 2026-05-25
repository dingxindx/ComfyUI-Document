# TTS 语音合成与克隆

基于输入的文本生成自然语音，或使用 5 秒参考音频进行语音克隆。

## 一、前置依赖

自定义节点 `ComfyUI-ChatTTS-OpenVoice`：

```bash
cd ComfyUI/custom_nodes/
git clone https://github.com/AIFSH/ComfyUI-ChatTTS-OpenVoice.git
pip install -r requirements.txt
```

## 二、文本转语音（基础版）

### 核心节点

| 节点 | 说明 |
|:----:|------|
| **ChatTTSNode** | 右键 → 搜索 "ChatTTS" |
| **Save Audio** | 右键 → 搜索 "Save Audio" |

### ChatTTSNode 参数

| 参数 | 推荐值 | 说明 |
|:----:|:------:|------|
| `text` | 输入文本 | 中文语音合成效果优秀，支持中英混合 |
| `audio_seed` | 42 | 控制音色随机种子，固定种子获得一致音色 |
| `temperature` | 0.3 | 采样温度，推荐 0.1-0.5 |
| `top_P` | 0.7 | 核采样参数 |
| `top_K` | 20 | Top-K 采样参数 |

## 三、语音克隆（5 秒参考音频）

### 新增节点

在基础版基础上增加 `Load Audio` 节点：

```
Load Audio (参考音频) → ChatTTSNode → Save Audio
```

### 连接

- `ref_audio`（或 `audio_conditioning`）← Load Audio 的 `AUDIO` 输出
- 参考音频要求：3-10 秒、清晰无噪、单一说话人

### 语音克隆参数

| 参数 | 推荐值 | 说明 |
|:----:|:------:|------|
| `emotion_strength` | 0.3-0.6 | 情感强度 0-1 |
| `audio_seed` | 42 | 固定音色 |
| `temperature` | 0.3 | 稳定性控制 |

### emotion_strength 参考

| 值 | 效果 | 场景 |
|:--:|:----:|:----:|
| 0.0 | 完全中性 | 新闻播报 |
| **0.3-0.5** | **自然有感情** | **日常对话（推荐）** |
| 0.7-0.9 | 情绪明显 | 朗读情感强烈段落（诗歌、独白） |
| 1.0 | 最大情感表达 | 可能过于夸张，谨慎使用 |
