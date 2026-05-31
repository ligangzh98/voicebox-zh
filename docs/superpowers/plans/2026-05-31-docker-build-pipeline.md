# Docker Build Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 新增 GitHub Actions workflow，在 tag push (`v*`) 时自动构建 Docker 镜像并导出 tar.gz，上传到 GitHub Release；同时提供部署用 docker-compose 文件。

**Architecture:** 新建独立 workflow 文件 (`docker-build.yml`)，与现有 `release.yml` 并行为独立的 tag-triggered workflow。新建 `docker-compose.deploy.yml` 基于现有 compose 文件，仅将 `build: .` 替换为 `image: voicebox:latest`。

**Tech Stack:** GitHub Actions, Docker, softprops/action-gh-release@v2

---

### Task 1: Create docker-compose.deploy.yml

**Files:**
- Create: `docker-compose.deploy.yml`

- [ ] **Step 1: Create docker-compose.deploy.yml**

基于现有 `docker-compose.yml`，将 `build: .` 替换为 `image: voicebox:latest`，其余配置完全不变：

```yaml
services:
  voicebox:
    image: voicebox:latest
    container_name: voicebox
    restart: unless-stopped

    ports:
      # Bind to localhost only for security
      - "127.0.0.1:17493:17493"

    volumes:
      # Bind-mount for generated audio (customize the host path as needed)
      # Host side:      ./output/
      # Container side: /app/data/generations/
      - ./output:/app/data/generations

      # Named volume for profiles, DB, cache (persists across container restarts)
      - voicebox-data:/app/data

      # HuggingFace model cache (so models aren't re-downloaded on rebuild)
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

- [ ] **Step 2: Commit**

```bash
git add docker-compose.deploy.yml
git commit -m "feat: add docker-compose.deploy.yml for pre-built image deployment"
```

---

### Task 2: Create docker-build.yml workflow

**Files:**
- Create: `.github/workflows/docker-build.yml`

- [ ] **Step 1: Create the workflow file**

```yaml
name: Docker Build

on:
  push:
    tags:
      - "v*"

jobs:
  build-docker:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build Docker image
        run: docker build -t voicebox:latest .

      - name: Export Docker image as tar.gz
        run: |
          VERSION="${GITHUB_REF_NAME#v}"
          docker save voicebox:latest | gzip > "voicebox-v${VERSION}-docker.tar.gz"
          echo "ARTIFACT_NAME=voicebox-v${VERSION}-docker.tar.gz" >> "$GITHUB_ENV"

      - name: Upload to GitHub Release
        uses: softprops/action-gh-release@v2
        with:
          files: ${{ env.ARTIFACT_NAME }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 2: Verify YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/docker-build.yml'))" && echo "OK: valid YAML"
```

Expected: `OK: valid YAML`

- [ ] **Step 3: Commit**

```bash
git add .github/workflows/docker-build.yml
git commit -m "feat: add Docker build workflow for tag-triggered image export"
```

---

### Task 3: Verification

- [ ] **Step 1: Confirm all new files exist**

```bash
ls -la .github/workflows/docker-build.yml docker-compose.deploy.yml
```

Expected: both files listed

- [ ] **Step 2: Confirm git log**

```bash
git log --oneline -3
```

Expected: two new commits visible

- [ ] **Step 3: Manual verification notes**

本次改动仅为新增文件，不修改任何现有代码。发布时验证步骤：
1. Push 一个 tag (如 `v0.5.1-test`) 触发 workflow
2. 在 Actions 面板确认 `Docker Build` workflow 成功
3. 在 Release 页面确认 `.tar.gz` asset 存在
4. 下载 tar.gz → `docker load` → `docker compose -f docker-compose.deploy.yml up` 验证
