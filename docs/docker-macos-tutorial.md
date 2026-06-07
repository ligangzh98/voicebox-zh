# Voicebox Docker 镜像在 macOS 12 上的使用教程

## 前提条件

### 1. 安装 Docker Desktop

macOS 12 (Monterey) 需要安装 **Docker Desktop for Mac**。

- 下载地址：https://www.docker.com/products/docker-desktop/
- 最低要求：Docker Desktop 4.x（支持 macOS 12）
- 安装后确保 Docker Desktop 正在运行（菜单栏显示 Docker 图标）

验证安装：

```bash
docker --version
# 期望输出类似：Docker version 24.x.x 或更高

docker compose version
# 期望输出类似：Docker Compose version v2.x.x
```

### 2. 关于芯片架构的说明

| Mac 类型 | 运行方式 | 性能 |
|---------|---------|------|
| Intel Mac (x86_64) | 原生运行 | ✅ 最佳性能 |
| Apple Silicon (M1/M2/M3/M4) | QEMU 模拟 amd64 | ⚠️ 较慢，ML 推理会明显变慢 |

> **Apple Silicon 用户注意**：Docker 镜像基于 `linux/amd64` 构建，在 Apple Silicon 上通过 QEMU 模拟运行。TTS 模型推理可能会很慢。如果你主要在 Apple Silicon Mac 上使用，建议优先考虑桌面应用版本。

---

## 步骤一：下载 Docker 镜像文件

前往 GitHub Releases 页面下载 Docker 镜像分卷文件：

```
https://github.com/ligangzh98/voicebox-zh/releases
```

找到对应版本的以下文件（以 v0.5.0 为例）：

```
voicebox-v0.5.0-docker.tar.gz.part-aa
voicebox-v0.5.0-docker.tar.gz.part-ab
```

> 文件数量取决于镜像大小，每个分卷约 1 GB。将所有分卷文件下载到同一个目录。

---

## 步骤二：合并分卷文件

打开终端，进入下载文件所在目录，执行合并命令：

```bash
cd ~/Downloads

# 合并所有分卷为一个完整的 tar.gz 文件
cat voicebox-v*-docker.tar.gz.part-* > voicebox-docker.tar.gz
```

> **注意**：`cat` 命令会按文件名顺序（part-aa, part-ab, ...）拼接，确保分卷文件名正确排序。

验证合并后的文件大小（通常 2-4 GB）：

```bash
ls -lh voicebox-docker.tar.gz
```

---

## 步骤三：加载 Docker 镜像

```bash
docker load < voicebox-docker.tar.gz
```

加载过程可能需要几分钟，完成后会显示：

```
Loaded image: voicebox:latest
```

验证镜像已加载：

```bash
docker images | grep voicebox
# 应该看到 voicebox  latest  xxx  xxx MB/GB  xxx
```

---

## 步骤四：准备部署配置文件

你需要一个 `docker-compose.deploy.yml` 文件。有两种方式获取：

### 方式 A：从仓库下载

```bash
curl -O https://raw.githubusercontent.com/ligangzh98/voicebox-zh/main/docker-compose.deploy.yml
```

### 方式 B：手动创建

在你希望运行 Voicebox 的目录下创建 `docker-compose.deploy.yml`：

```yaml
services:
  voicebox:
    image: voicebox:latest
    container_name: voicebox
    restart: unless-stopped

    ports:
      # 仅绑定本机，安全性更好
      - "127.0.0.1:17493:17493"

    volumes:
      # 生成的音频文件（宿主机可直接访问）
      - ./output:/app/data/generations

      # 持久化数据（配置、数据库等）
      - voicebox-data:/app/data

      # HuggingFace 模型缓存（避免重复下载）
      - huggingface-cache:/home/voicebox/.cache/huggingface

    environment:
      - LOG_LEVEL=info
      - NUMBA_CACHE_DIR=/tmp/numba_cache

    networks:
      - voicebox-net

    deploy:
      resources:
        limits:
          cpus: '4'
          memory: 8G

networks:
  voicebox-net:
    driver: bridge

volumes:
  voicebox-data:
  huggingface-cache:
```

---

## 步骤五：启动 Voicebox

在 `docker-compose.deploy.yml` 所在目录执行：

```bash
docker compose -f docker-compose.deploy.yml up -d
```

首次启动时，容器需要下载 TTS 模型文件（可能需要几分钟到十几分钟，取决于网络速度）。

查看启动日志：

```bash
docker compose -f docker-compose.deploy.yml logs -f
```

看到类似以下日志表示启动成功：

```
INFO:     Uvicorn running on http://0.0.0.0:17493
```

按 `Ctrl+C` 退出日志查看（容器仍在后台运行）。

---

## 步骤六：访问 Voicebox

打开浏览器，访问：

```
http://localhost:17493
```

即可看到 Voicebox 的 Web 界面。

---

## 常用操作

### 查看容器状态

```bash
docker compose -f docker-compose.deploy.yml ps
```

### 停止服务

```bash
docker compose -f docker-compose.deploy.yml down
```

### 重启服务

```bash
docker compose -f docker-compose.deploy.yml restart
```

### 查看实时日志

```bash
docker compose -f docker-compose.deploy.yml logs -f
```

### 更新镜像（下载新版本后）

```bash
# 1. 停止旧容器
docker compose -f docker-compose.deploy.yml down

# 2. 加载新镜像（重复步骤二、三）
cat voicebox-v*-docker.tar.gz.part-* > voicebox-docker.tar.gz
docker load < voicebox-docker.tar.gz

# 3. 启动新容器（数据卷会自动保留）
docker compose -f docker-compose.deploy.yml up -d
```

### 释放网络绑定（允许局域网访问）

默认只允许本机访问。如需让局域网内其他设备访问，修改端口绑定：

```yaml
ports:
  - "0.0.0.0:17493:17493"
```

> ⚠️ **安全警告**：API 没有内置认证机制，仅在可信网络中开放。

### 调整资源限制

根据你的 Mac 配置调整 CPU 和内存限制：

```yaml
deploy:
  resources:
    limits:
      cpus: '8'    # 根据你的 CPU 核心数调整
      memory: 16G  # 建议至少 8GB，16GB+ 更佳
```

---

## 获取生成的音频文件

生成的音频文件会保存在宿主机的 `./output` 目录中（与 `docker-compose.deploy.yml` 同级）。

```bash
ls ./output/
```

---

## 故障排查

### 容器启动后看到 JSON 而不是界面

如果访问 `http://localhost:17493` 看到 `{"message": "voicebox API", ...}`，说明前端构建可能有问题。尝试重新构建（需要源码）：

```bash
docker compose build --no-cache
```

### 模型每次重启都要重新下载

确保配置了 `huggingface-cache` 卷：

```yaml
volumes:
  - huggingface-cache:/home/voicebox/.cache/huggingface
```

### 内存不足（OOM Killed）

增加内存限制：

```yaml
deploy:
  resources:
    limits:
      memory: 16G
```

### 端口被占用

```bash
# 查看谁在使用 17493 端口
lsof -i :17493

# 或者换一个端口
ports:
  - "127.0.0.1:8080:17493"
```

### Apple Silicon Mac 运行缓慢

由于 Docker 镜像基于 `linux/amd64`，在 Apple Silicon 上需要通过 QEMU 模拟。ML 推理会明显变慢。建议：

1. 优先使用桌面应用版本（原生 Apple Silicon 支持）
2. 或者在 Intel Linux 服务器上部署 Docker 版本

---

## 附录：清理资源

如果不再需要 Voicebox，可以彻底清理：

```bash
# 停止并删除容器
docker compose -f docker-compose.deploy.yml down

# 删除镜像
docker rmi voicebox:latest

# 删除数据卷（⚠️ 这会删除所有生成的音频、配置和缓存的模型）
docker volume rm voicebox-data huggingface-cache

# 删除合并后的 tar.gz 文件
rm voicebox-docker.tar.gz
```
