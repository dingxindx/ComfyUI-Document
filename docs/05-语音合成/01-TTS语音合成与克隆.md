# TTS 语音合成与克隆

## 一、什么是 ComfyUI 中的 TTS？

通过自定义节点在 ComfyUI 中将文本转为语音，甚至可以只用 5 秒参考音频克隆声音。

## 二、前置准备

```bash
cd ~/workspace/ComfyUI/custom_nodes/
git clone https://gitclone.com/github.com/AIFSH/ComfyUI-ChatTTS-OpenVoice.git
pip install -r requirements.txt
```

首次运行时自动下载 ChatTTS 和 OpenVoice 模型（需网络）。

## 三、基础 TTS 工作流

| 节点 | 说明 |
|:-----|:-----|
| ChatTTSNode | 右键 → 搜索 "ChatTTS" |
| Save Audio | 右键 → 搜索 "Save Audio" |

### 参数

| 参数 | 推荐值 | 说明 |
|:-----|:------:|:-----|
| `text` | 要合成的文本 | 中文效果优秀 |
| `audio_seed` | 42 | 固定种子获得一致音色 |
| `temperature` | 0.3 | 越低越稳定（0.1-0.5）|
| `top_P` | 0.7 | 核采样 |
| `top_K` | 20 | Top-K 采样 |

## 四、语音克隆工作流

额外需要 **Load Audio** 节点加载参考音频。

| 参数 | 推荐值 | 说明 |
|:-----|:------:|:-----|
| `emotion_strength` | 0.3-0.6 | 情感强度 0-1 |
| 参考音频 | 3-10 秒，清晰无噪，单一说话人 | 决定克隆质量 |

### emotion_strength 参考

| 值 | 效果 | 场景 |
|:--:|:----:|:-----|
| 0.0 | 完全中性 | 新闻播报 |
| 0.3-0.5 | 自然有感情 | **日常对话（推荐）** |
| 0.7-1.0 | 情绪明显 | 诗歌、独白、情感强烈 |

### 连线

```
Load Audio (参考) → ChatTTSNode.ref_audio
ChatTTSNode → Save Audio
```
