这份《PAIUI Vision 核心开发与部署规范文档》是你接下来所有开发工作的准则。请务必保存在你的项目根目录中。

------

# 📖 PAIUI Vision - 核心开发与部署规范 (V1.0)

## 一、 系统架构定位

**PAIUI Vision** 是一款专为电商卖家设计的 AI 视觉生成引擎。系统采用“计算与业务彻底分离”**的设计哲学，支持**“本地算力 (Local GPU)”**与**“云端算力 (Cloud SaaS)”双模平滑切换。全链路采用 Docker 容器化编排，以满足未来“一键私有化交付”的商业需求。

------

## 二、 核心环境与版本要求

为了杜绝“在我的电脑上能跑，客户那边跑不起来”的经典悲剧，所有开发与运行环境必须严格锁定以下大版本：

### 1. 宿主机要求 (Host Environment)

- **操作系统**：Windows 11 (22H2及以上) 或 Ubuntu 22.04 LTS。
- **虚拟化底座 (Windows端)**：必须启用 **WSL2** (Windows Subsystem for Linux)。
- **容器引擎**：Docker Engine v24.0+ 或 Docker Desktop v4.20+。
- **硬件驱动**：NVIDIA GPU Driver (最低支持 CUDA 11.8，推荐 12.1+)。
- **GPU 穿透组件**：必须安装 `NVIDIA Container Toolkit`。

### 2. 容器服务栈 (Container Stack)

- **中台业务 (PAIUI-app)**：**PHP 8.3 (FPM)** + **ThinkPHP 8.x** (基于 Alpine Linux 镜像以减小体积)。
- **核心数据库 (PAIUI-db)**：**MySQL 8.0.35+** (禁用旧版认证协议，使用 `caching_sha2_password`)。
- **队列与缓存 (PAIUI-redis)**：**Redis 7.0+** (AOF 持久化模式开启)。
- **AI 推理引擎 (PAIUI-ai-engine)**：**Python 3.10.x** + **PyTorch 2.1.2** + **CUDA 12.1** (运行 ComfyUI)。

------

## 三、 标准化项目目录结构

请在你的 Windows `D:\` 或非系统盘创建项目，严格遵守以下结构，确保“代码、配置、模型、数据”彻底分离：

Plaintext`/PAIUI-Vision ├── docker-compose.yml         # 全局容器编排配置文件 (核心) ├── .env                       # 全局环境变量 (数据库密码、算力模式切换等) ├── /PAIUI-app                # ThinkPHP 业务中台 │   ├── /src                   # PHP 核心代码 │   ├── /docker                # 存放 PHP 相关的 Dockerfile 和 Nginx 配置 │   └── /public                # Web 根目录 ├── /PAIUI-ai                 # AI 引擎侧 (如果采用本地模式) │   ├── /models                # 挂载的 AI 模型 (大文件，绝对不要打包进镜像) │   ├── /output                # AI 生成的高清图缓存区 │   └── /custom_nodes          # ComfyUI 的自定义插件目录 └── /storage                   # 持久化数据     ├── /mysql_data            # MySQL 数据库挂载卷     └── /redis_data            # Redis 缓存挂载卷 `

------

## 四、 全局环境变量规范 (`.env`)

必须使用 `.env` 文件控制算力流向。在 ThinkPHP 中编写 `AiService` 时，通过读取以下变量来决定请求发往何处：

------

## 五、 开发与部署的“红线”注意事项 (Precautions)

这是无数开发者踩过坑总结出来的血泪教训，请在开发前仔细核对：

### 🚨 1. 换行符的“隐形炸弹” (CRLF vs LF)

- **问题**：Windows 系统默认使用 `CRLF`（回车换行），而 Linux 容器只认 `LF`。如果你的 `.sh` 启动脚本或 PHP 配置文件带有 `CRLF`，Docker 启动时会直接报错 `standard_init_linux.go: exec user process caused: no such file or directory`。
- **解决方案**：在你的代码编辑器（VS Code / PhpStorm）右下角，强制将整个项目的换行符设置为 **LF**。并建议在根目录添加 `.gitattributes` 文件，强制 Git 拉取时使用 `LF`。

### 🚨 2. 容器内网通信盲区 (Internal DNS)

- **问题**：在 ThinkPHP 代码中，如果用 `127.0.0.1` 尝试连接数据库或 Redis，一定会失败。因为在 `PAIUI-app` 容器内部，`127.0.0.1` 指的是该容器自己，而不是宿主机或其他容器。
- **解决方案**：永远使用 `docker-compose.yml` 中定义的服务名（Service Name）进行通信。例如，连接数据库的主机名填 `PAIUI-db`。

### 🚨 3. AI 引擎接口的非阻塞设计

- **问题**：由于生成一张 4K 电商主图可能需要 10~30 秒。如果前端发请求给 PHP，PHP 再发给 ComfyUI，并一直 `sleep()` 等待结果，会导致 PHP-FPM 进程被迅速耗尽，网站直接卡死。
- **解决方案**：必须强制使用 **“投递-轮询/回调”机制**。
  1. PHP 生成唯一 `TaskID`，推入 Redis 队列，立即返回前端 `200 状态码：任务排队中`。
  2. 后台守护进程消费队列，将任务发给 ComfyUI。
  3. 前端通过定时器轮询 PHP 接口（或 WebSocket）查询该 `TaskID` 的生成进度。

### 🚨 4. GPU 穿透的权限陷阱 (Windows WSL2)

- **问题**：在 Windows Docker Desktop 中，即便容器启动成功，也可能没有真正调用显卡，导致生图极慢（退化为 CPU 计算）。

- **解决方案**：确保在 `docker-compose.yml` 中针对 AI 容器明确声明了 `nvidia` 运行时：

  YAML`deploy:   resources:     reservations:       devices:         - driver: nvidia           count: 1           capabilities: [gpu] `

  启动后，在终端执行 `docker exec -it PAIUI-ai-engine nvidia-smi`，能看到显卡信息才算成功。

### 🚨 5. 模型资产与代码剥离

- **问题**：AI 模型（如 Flux, SDXL, ControlNet 权重）动辄数十 GB。绝对不能把它们写进 Dockerfile 打包进 Image 里，否则你的镜像将无法分发。
- **解决方案**：使用 Volume（卷挂载）。将宿主机的 `./PAIUI-ai/models` 挂载到容器内的 `/workspace/models`。未来打包交付时，代码镜像只有几十 MB，模型文件作为一个单独的压缩包提供给客户。


最新的 PAIUI 目录结构
Plaintext
/PAIUI
├── docker-compose.yml         # 核心编排文件
├── .env                       # 环境变量
├── /paiui-app                 # TP8 代码目录 (新建空文件夹即可)
├── /paiui-ai                  # AI 挂载目录 (新建空文件夹即可)
│   ├── /models
│   └── /output
└── /storage                   # 数据挂载目录 (新建空文件夹即可)
二、 核心文件：docker-compose.yml
请将以下代码直接复制并保存为 docker-compose.yml，注意层级缩进：

YAML
version: '3.8'

services:
  # 1. 业务中台 (PHP + Nginx) - 这里我们先用一个官方带Nginx的PHP镜像测试连通性
  paiui-app:
    image: webdevops/php-nginx:8.3-alpine
    container_name: paiui-app
    ports:
      - "80:80"
    volumes:
      - ./paiui-app:/app   # 代码挂载到容器内的 /app
    depends_on:
      - paiui-db
      - paiui-redis
    environment:
      - WEB_DOCUMENT_ROOT=/app/public  # TP8 的默认运行目录
      - DB_HOST=paiui-db
      - REDIS_HOST=paiui-redis

  # 2. 数据库
  paiui-db:
    image: mysql:8.0
    container_name: paiui-db
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASS}
      MYSQL_DATABASE: ${DB_NAME}
    ports:
      - "3306:3306"
    volumes:
      - ./storage/mysql_data:/var/lib/mysql

  # 3. 缓存与任务队列
  paiui-redis:
    image: redis:7.0-alpine
    container_name: paiui-redis
    ports:
      - "6379:6379"
    volumes:
      - ./storage/redis_data:/data

  # 4. AI 推理引擎 (ComfyUI)
  paiui-ai-engine:
    image: yanwk/comfyui-boot:latest-cuda12.1
    container_name: paiui-ai-engine
    ports:
      - "8188:8188"
    volumes:
      - ./paiui-ai/models:/home/runner/ComfyUI/models
      - ./paiui-ai/output:/home/runner/ComfyUI/output
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
    restart: unless-stopped
三、 环境变量配置：.env
在同级目录创建一个名为 .env 的文件，写入以下内容：

Ini, TOML
# PAIUI Database Config
DB_NAME=paiui_db
DB_PASS=paiui_secret_2026

# PAIUI AI Node Config
AI_COMPUTE_MODE=local
AI_ENDPOINT=http://paiui-ai-engine:8188