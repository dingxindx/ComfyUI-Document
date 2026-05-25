# ComfyUI Manager——你的"应用商店"

ComfyUI Manager 是一个自定义节点管理器，可以把它理解为 ComfyUI 的"应用商店"。通过它，你可以在界面上直接搜索、安装、更新各种节点，不需要手动 git clone。

**建议在你的第一个工作流跑通之后再安装 Manager**，这样你至少知道基础操作（节点、画布、参数），不会额外增加学习负担。

---

## 一、安装 ComfyUI Manager

### 方法 1：终端安装（推荐）

确保 ComfyUI 没有在运行（如果正在运行，去终端按 Ctrl+C 停止）。

```bash
# 进入 ComfyUI 的自定义节点目录
cd ~/workspace/ComfyUI/custom_nodes/

# 从 GitHub 克隆 Manager
git clone https://github.com/ltdrdata/ComfyUI-Manager.git

# 国内用户用镜像
# git clone https://gitclone.com/github.com/ltdrdata/ComfyUI-Manager.git
```

**装完后重启 ComfyUI：**
```bash
cd ~/workspace/ComfyUI
source venv/bin/activate   # 激活虚拟环境（如果还没激活）
python main.py             # 重启
```

### 方法 2：下载压缩包（国内用户备选）

1. 浏览器访问 `https://gitclone.com/github.com/ltdrdata/ComfyUI-Manager`
2. 点击 "Code" → "Download ZIP"
3. 解压到 `ComfyUI/custom_nodes/` 目录下
4. 确保文件夹名是 `ComfyUI-Manager`（不是 `ComfyUI-Manager-master`）
5. 重启 ComfyUI

---

## 二、如何使用 ComfyUI Manager

安装成功后，在 ComfyUI 界面右上角（或菜单栏）会出现一个 **Manager** 按钮。

### 打开 Manager

点击 **Manager** 按钮，你会看到：

```
┌───────────────────────────────────────────────┐
│  ComfyUI Manager                              │
│                                               │
│  [Install Custom Nodes]  ← 安装自定义节点      │
│  [Install Models]        ← 安装模型           │
│  [Update All]            ← 更新所有节点       │
│  [Update ComfyUI]        ← 更新 ComfyUI 本体  │
│  [Settings]              ← Manager 设置       │
└───────────────────────────────────────────────┘
```

### 安装自定义节点

1. 点击 **Manager** → **Install Custom Nodes**
2. 在搜索框输入节点名称（如 "IPAdapter"、"ControlNet"、"VideoHelperSuite"）
3. 在搜索结果中找到目标节点
4. 点击右侧的 **Install** 按钮
5. 等待安装完成
6. 弹窗提示 "Restart" → 重启 ComfyUI

> ⚠️ **国内用户**：如果 Manager 的在线搜索加载不出来，需要在 Manager Settings 中配置镜像源。

### 配置 Manager 镜像

点击 **Manager** → **Settings（设置图标）**：

| 设置项 | 建议值（国内） |
|:-------|:---------------|
| Git Clone Prefix | `https://gitclone.com/` |
| Pip Mirror | `https://pypi.tuna.tsinghua.edu.cn/simple` |

> ✅ 配置好后，Manager 的安装和更新都会走国内镜像，速度快很多。

---

## 三、Mananger 安装常见问题

### Q：Manager 按钮没出现？
**原因**：安装后没有重启 ComfyUI
**解决**：重启 ComfyUI

### Q：点 Manager 后一片空白？
**原因**：Manager 在加载 GitHub 的节点列表，网络访问慢
**解决**：配置镜像源（如上所述）

### Q：安装节点时出现红色错误？
**原因**：通常是 pip 安装依赖超时
**解决**：
```bash
# 手动进入节点目录安装依赖
cd ~/workspace/ComfyUI/custom_nodes/你刚安装的节点
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

---

## 四、什么时候需要安装什么节点？

这里是本书会用到的主要自定义节点，建议用到时再装，不用提前全部安装：

| 章节 | 需要安装的节点 | 备注 |
|:-----|:---------------|:-----|
| 入门篇 | 暂无 | 内置节点就够 |
| ControlNet | `comfyui_controlnet_aux` | 姿态控制必须 |
| IP-Adapter | `ComfyUI_IPAdapter_plus` | 风格迁移 |
| LTX 视频 | `ComfyUI-LTXVideo` | 视频生成 |
| Wan 视频 | `ComfyUI-Wan-Superresolution` | Wan 视频 |
| TTS 语音 | `ComfyUI-ChatTTS-OpenVoice` | 语音合成 |
| GGUF 量化 | `ComfyUI-GGUF` | 低显存优化 |
