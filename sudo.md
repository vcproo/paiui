这份汇总将是你未来维护 PAIUI 底层算力引擎的“终极控制台操作手册”。

作为架构师，我帮你把刚才咱们一步步打下的“江山”，按照功能模块进行了逻辑重组和精简。建议你将这篇回复直接收藏或截图保存。

🚀 1. 日常高频使用：引擎启动命令
这是你以后每次重启云服务器后，唯一需要执行的一行命令（必须在 ComfyUI 根目录下执行）：

Bash
# 1. 先进入项目目录
cd /root/autodl-tmp/ComfyUI

# 2. 跨域穿透启动引擎 (暴露给外网访问)
python main.py --listen 0.0.0.0 --port 6006 --enable-cors-header
停止引擎的快捷键：在终端运行状态下，按下 Ctrl + C。

🛠️ 2. 底层环境修复 (PyTorch 重装降级防冲突)
如果未来你因为乱装其他环境导致引擎报错崩溃（比如提示找不到 CUDA），执行这套“核弹级”修复指令，强制锁死正确的 CUDA 12.1 版本底座：

Bash
# 1. 卸载所有错乱的视觉库
pip uninstall -y torch torchvision torchaudio

# 2. 强制拉取官方指定版本 (禁缓存)
pip install torch==2.5.1+cu121 torchvision==0.20.1+cu121 torchaudio==2.5.1+cu121 --index-url https://download.pytorch.org/whl/cu121 --no-cache-dir
📥 3. 极速拉取大模型 (跳过 UI 界面防墙)
当 UI 面板因为网络问题无法下载大模型时，通过国内镜像源直接拉取：

Bash
# 1. 进入大模型专属存放目录
cd /root/autodl-tmp/ComfyUI/models/checkpoints

# 2. 拉取 Juggernaut XL V9 (电商摄影级保真模型)
wget https://hf-mirror.com/RunDiffusion/Juggernaut-XL-v9/resolve/main/Juggernaut-XL_v9_RunDiffusionPhoto_v2.safetensors
🧩 4. 强制拉取 Github 核心插件 (带国内镜像加速)
当 Github 连不上，UI 面板搜不到插件时，使用这组命令强制把源码克隆到机器里（每次克隆前需确保进入了 custom_nodes 目录）：

Bash
# 1. 统一进入插件目录
cd /root/autodl-tmp/ComfyUI/custom_nodes

# 2. 安装 ComfyUI Manager (应用商店)
git clone https://github.com/ltdrdata/ComfyUI-Manager.git

# 3. 安装 "抠图神器" (BRIA RMBG)
git clone https://mirror.ghproxy.com/https://github.com/ZHO-ZHO-ZHO/ComfyUI-BRIA_RMBG.git

# 4. 安装 "变形锁死器" (Advanced ControlNet)
git clone https://mirror.ghproxy.com/https://github.com/Kosinkadink/ComfyUI-Advanced-ControlNet.git

# 5. 安装 "光影重绘器" (IC-Light)
git clone https://mirror.ghproxy.com/https://github.com/kijai/ComfyUI-IC-Light.git
🗑️ 5. 其他实用系统指令 (AutoDL 常用)
Bash
# 查看当前显卡显存占用情况
nvidia-smi

# 查看当前硬盘存储空间 (重点看 /root/autodl-tmp)
df -h
手册收好！ 现在你可以回到控制台，去执行咱们上一步最后交代的任务了：拉取那三个核心插件。一旦你那边准备就绪，我们就进入 API 工作流联调的最爽环节！