# 安装 ComfyUI —— 每条命令都是做什么的

> 📌 **如果你已经安装了 ComfyUI**：打开终端执行 `ls ~/workspace/ComfyUI/main.py`，如果能看到文件，说明已安装。请跳到下一章[下载第一个模型](00-03-下载第一个模型并首次生成.md)。
>
> 📌 **检查是否跑得起来**：`cd ~/workspace/ComfyUI && python main.py`，能出现 `Prompt server running on: http://0.0.0.0:8188` 就行。

这是本书最关键的步骤。跟着我一步一步来，**每完成一步，停下来看看终端里发生了什么**。

---

## 方案 A：手动安装（推荐，可定制性最强）

### 第 1 步：打开终端

**macOS**：Command + 空格 → 输入"终端" → 回车
**Windows**：Win + R → 输入 `cmd` → 回车 → 或搜索"PowerShell"
**Linux**：Ctrl + Alt + T

> 📌 **你的操作系统**：macOS。以下命令以 macOS 为例，Windows 用户注意括号中的替代方案。

### 第 2 步：决定 ComfyUI 装在哪里

建议放在用户目录下（这样权限问题最少）。我们把它放在桌面或工作目录下：

```bash
cd ~/workspace
```

> 💡 这条命令的意思是"进入 workspace 目录"。`~` 代表你的用户主目录（即 `/Users/bruce.ding`）。
> 如果 `workspace` 不存在，先创建：`mkdir -p ~/workspace`

### 第 3 步：从 GitHub 下载 ComfyUI 代码

```bash
git clone https://github.com/comfyanonymous/ComfyUI.git
```

**这条命令在做什么？**
- `git clone` = 把 GitHub 上的整个项目复制到你的电脑
- 后面的 URL = ComfyUI 的官方仓库地址
- 执行完成后，你的 `workspace` 目录下会多一个 `ComfyUI` 文件夹

**如果你在国内，GitHub 访问慢：**
使用镜像地址替代：
```bash
git clone https://gitclone.com/github.com/comfyanonymous/ComfyUI.git
```

> ✅ **验证**：执行完后，输入 `ls` 看看有没有出现 `ComfyUI` 文件夹。

### 第 4 步：进入 ComfyUI 目录

```bash
cd ComfyUI
```

> ✅ 验证：终端提示符前面应该变成类似 `~/workspace/ComfyUI %` 或 `(base) bruce.ding@Mac ComfyUI %`

### 第 5 步：创建 Python 虚拟环境（非常重要！）

```bash
python3 -m venv venv
```

**为什么需要虚拟环境？**
虚拟环境是一个"隔离的 Python 小房间"。每个 Python 项目都有自己的虚拟环境，这样：
- 项目 A 需要 1.0 版本的包、项目 B 需要 2.0 版本 → 互不干扰
- 不会污染你系统自带的 Python
- 删掉整个项目文件夹就卸载干净了

**这条命令在做什么？**
- `python3` = 调用 Python 3 解释器
- `-m venv` = 使用 Python 内置的 `venv` 模块
- `venv` = 虚拟环境文件夹的名字（叫 `venv` 是惯例）

执行完后，`ComfyUI/` 目录下会出现一个 `venv/` 文件夹，里面装了一个独立的 Python 副本。

> ⚠️ **macOS 你可能遇到的情况**：如果你用 Homebrew 装的 Python，这步会正常工作。如果你用的是 Conda（`/opt/anaconda3`），虚拟环境会基于 Conda 的 Python 版本创建。
>
> 💡 **Conda 用户专属提示**：如果你习惯用 conda 环境，可以跳过 venv，直接用：
> ```bash
> # 创建并激活 conda 环境
> conda create -n comfyui python=3.11
> conda activate comfyui
> ```
> 后续所有 `pip install` 和 `python main.py` 在 conda 环境下执行即可。用 `which python` 确认当前使用的是 conda 路径而不是系统 Python。

### 第 6 步：激活虚拟环境

```bash
source venv/bin/activate
```

**这条命令在做什么？**
- 告诉终端：从现在开始，所有 Python 相关的操作都在 `venv` 这个小房间里进行
- 激活成功后，终端提示符前面会出现 `(venv)`

> ✅ 验证：你应该看到类似 `(venv) bruce.ding@Mac ComfyUI %` 的字样。
>
> ⚠️ 如果你用的是 Conda，且不想用 `venv`，可以直接：`conda activate comfyui`（假设你创建了同名的 conda 环境）

### 第 7 步：安装 PyTorch（AI 计算的底层框架）

**对于 macOS（你）：**
```bash
# macOS 默认没有 NVIDIA 显卡，所以装 CPU 版本或 MPS 版本
pip install torch torchvision torchaudio
```

**对于 Windows/Linux 有 NVIDIA 显卡的用户：**
```bash
# CUDA 12.1（推荐）
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu121
```

**为什么这一步单独做？**
PyTorch 是 ComfyUI 的"引擎"（做 AI 计算的核心库）。它有不同版本（CPU 版 / CUDA 版 / MPS 版），需要根据你的硬件选择。如果把这步放到后面的 `requirements.txt` 一起装，可能会装错版本。

### 第 8 步：安装 ComfyUI 的其他依赖

```bash
pip install -r requirements.txt
```

**这条命令在做什么？**
- `pip install` = Python 的包管理工具，用来安装第三方库
- `-r requirements.txt` = 读取 `requirements.txt` 文件中的依赖列表，挨个安装

**如果你在国内，PyPI 下载慢：**
```bash
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

**这个过程会安装什么？**
依赖清单里包含几十个 Python 库，比如：
- `numpy` = 数学计算库
- `Pillow` = 图片处理库
- `opencv-python` = 计算机视觉库
- `safetensors` = 模型文件格式支持
- 等等...

安装过程可能需要 2-10 分钟（取决于网速）。你会看到很多输出信息，这是正常的。

> ✅ 验证：命令执行完毕后，如果没有任何错误信息（红色文字），说明安装成功。

### 第 9 步：确认安装成功

```bash
python main.py
```

**这条命令在做什么？**
- `python` = 用当前激活的虚拟环境中的 Python 运行
- `main.py` = ComfyUI 的启动文件

**你应该看到的输出：**
```
Starting ComfyUI Server...
Total VRAM ... GB, total RAM ... GB
Using device: mps (macOS) 或 cuda (NVIDIA)
Prompt server running on: http://0.0.0.0:8188
```

看到 `Prompt server running on: http://0.0.0.0:8188` 说明启动成功。

> ⚠️ **千万不要关闭这个终端窗口**！只要 ComfyUI 在运行，这个窗口就必须开着。关掉它就停止服务了。

### 第 10 步：打开浏览器访问

1. 打开浏览器（Chrome / Safari / Edge 都可以）
2. 在地址栏输入：`http://127.0.0.1:8188`（或 `http://localhost:8188`）
3. 回车 → 你就看到了 ComfyUI 的画布界面！

---

## 方案 B：ComfyUI Manager 一键安装（偷懒方案）

如果你的目的是"最快速度用上"，用 ComfyUI Manager 可以跳过很多手动步骤。

但是注意：**本教程基于方案 A 的手动安装**。方案 B 虽然省事，但出了问题你很难知道哪里出错了。

**如果你坚持用方案 B**，去 哩布哩布（liblibai.com）搜索"ComfyUI 整合包"，下载后解压即用。

---

## 🚑 安装常见问题排查

### 问题 1：`pip` 命令找不到
**表现**：`zsh: command not found: pip`
**解决**：改用 `pip3` 试试；或者先激活虚拟环境再试。

### 问题 2：安装过程红色错误信息
**表现**：安装到中间报红
**解决**：看错误信息最后几行。常见原因是：
- 网络超时 → 使用国内镜像重试
- 缺少编译器 → macOS 需要安装 Xcode Command Line Tools：`xcode-select --install`
- 版本冲突 → 创建一个全新的虚拟环境重新开始

### 问题 3：启动时端口被占用
**表现**：`Address already in use` 或端口 8188 被占用
**解决**：
```bash
# 换个端口启动
python main.py --port 8189
```
然后浏览器访问 `http://127.0.0.1:8189`

### 问题 4：ComfyUI 能打开但节点全是红色
**表现**：界面出现了，但所有节点模块都标红
**解决**：这是正常的——你还没有安装任何模型。继续下一章下载模型就好了。
