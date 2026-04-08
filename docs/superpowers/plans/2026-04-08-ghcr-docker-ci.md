# GHCR Docker CI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a GitHub Actions workflow that validates Docker builds on PRs and publishes the image to GHCR on `main` pushes.

**Architecture:** Keep Docker publishing in an isolated workflow file and use Docker official GitHub actions for metadata, registry login, build, push, and cache management. Document the resulting image and trigger behavior in the existing Docker section of the README.

**Tech Stack:** GitHub Actions YAML, Docker Buildx, GHCR, repository Markdown docs

---

### Task 1: Add Docker publish workflow

**Files:**
- Create: `.github/workflows/docker-build.yml`

- [ ] **Step 1: Write the workflow file**

```yaml
name: docker-build

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  workflow_dispatch:

concurrency:
  group: docker-build-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read
  packages: write

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      - name: Log in to GitHub Container Registry
        if: github.event_name != 'pull_request'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix=sha-
      - name: Build and push Docker image
        uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

- [ ] **Step 2: Verify the workflow file exists and reads correctly**

Run: `sed -n '1,220p' .github/workflows/docker-build.yml`
Expected: workflow contains `push`, `pull_request`, `workflow_dispatch`, GHCR login, metadata, and build/push steps

### Task 2: Document the workflow behavior

**Files:**
- Modify: `README.md`

- [ ] **Step 1: Add a short GHCR automation section under Docker deployment**

```md
### GitHub Actions 自动构建与推送

- `push` 到 `main` 时自动构建并推送镜像到 `ghcr.io/<owner>/<repo>`
- `pull_request` 到 `main` 时只校验 Docker 构建，不推送镜像
- 支持在 Actions 页面手动触发 `workflow_dispatch`

默认标签包括 `latest`、`main` 和 `sha-<commit>`。
```

- [ ] **Step 2: Verify the README section is present**

Run: `rg -n "GitHub Actions 自动构建与推送|ghcr.io" README.md`
Expected: README contains the new GHCR workflow notes

### Task 3: Run configuration verification

**Files:**
- Verify: `.github/workflows/docker-build.yml`
- Verify: `README.md`

- [ ] **Step 1: Check for whitespace and patch issues**

Run: `git diff --check`
Expected: no output

- [ ] **Step 2: Inspect the final diff**

Run: `git diff -- .github/workflows/docker-build.yml README.md`
Expected: only the new workflow file and README documentation changes appear
