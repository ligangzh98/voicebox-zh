# Docker Build Pipeline — Design Spec

**Date**: 2026-05-31
**Status**: Approved

## Overview

新增 GitHub Actions workflow，在 tag 推送 (`v*`) 时自动构建 Docker 镜像并导出为 `.tar.gz` 文件，上传到 GitHub Release。用户下载后可离线传输到局域网机器，通过 `docker compose up` 直接运行。

## Motivation

当前项目有完整的 Dockerfile 和 docker-compose.yml，但没有自动构建流程。局域网部署场景需要一个预构建的镜像文件，避免在目标机器上拉代码 + 安装依赖 + 构建。

## Design

### 新增文件

| 文件 | 用途 |
|---|---|
| `.github/workflows/docker-build.yml` | GitHub Actions 构建流水线 |
| `docker-compose.deploy.yml` | 部署用 compose 文件（使用预构建镜像） |

### Workflow 流程

```
push tag (v*)
    ↓
Checkout 代码
    ↓
docker build -t voicebox:latest .
    ↓
docker save voicebox:latest | gzip → voicebox-v{version}-docker.tar.gz
    ↓
上传到 GitHub Release（追加到已有 draft release）
```

### docker-compose.deploy.yml

基于现有 `docker-compose.yml`，唯一差异：用 `image: voicebox:latest` 替换 `build: .`。其余配置（ports、volumes、environment、networks、resources）完全保持一致。

### 局域网部署流程

```bash
# 1. 下载 voicebox-v0.5.0-docker.tar.gz（从 GitHub Release）
# 2. 拷贝到局域网机器
# 3. 导入镜像
docker load < voicebox-v0.5.0-docker.tar.gz
# 4. 启动服务
docker compose -f docker-compose.deploy.yml up -d
```

### 关键决策

- **触发时机**: tag `v*`，与现有 release.yml 对齐
- **分发方式**: tar.gz 导出，不上传 registry（用户只需要离线传输）
- **目标平台**: linux/amd64
- **压缩**: gzip 减小传输体积（原始 tar 可能数 GB）
- **Release 关联**: 追加到 release.yml 创建的 draft release，共用 version 和 changelog

### 权限

- `contents: write` — 上传 release asset 所需
- 无需 secrets（不上传 registry）

### 边界情况

- **Docker 构建失败**: workflow 失败，不影响 release.yml（独立 workflow）
- **Release 尚未创建**: `softprops/action-gh-release@v2` 在 release 不存在时会自动创建；如果 release.yml 先创建了 draft，则追加文件
- **重复运行**: 同一 tag 多次触发时，同名 asset 会被覆盖
