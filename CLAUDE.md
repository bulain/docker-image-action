# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 语言

**必须始终使用中文回答。**

## 用途

将公网 Docker 镜像与 Helm Chart 搬运（镜像）到私有 registry（阿里云 ACR / 腾讯云 TCR）。没有可构建的应用代码——核心逻辑全部是两个 GitHub Actions workflow 里的 bash 脚本。

## 架构

- `images.properties` — 待搬运的 Docker 镜像清单，一行一个。支持 `#` 注释与空行。短写（`mysql:9.7`）默认取自 docker.io；可用 `--platform linux/amd64 xxx` 指定架构。
- `charts.properties` — 待搬运的 Helm Chart 清单，格式 `oci://.../name:version`（`:` 后为 chart 版本）。短写补全为 `oci://registry-1.docker.io/`。
- `.github/workflows/docker.yaml` — 逐行读 `images.properties`，`pull → tag → push → rmi`（逐个清理磁盘）。
- `.github/workflows/helm.yaml` — 逐行读 `charts.properties`：用 `helm images get` 解析 chart 引用的镜像并搬运（推到 ACR），再 `helm pull` chart 本身推到 TCR。
- `docs/superpowers/specs/` — 两份设计文档，改动搬运逻辑前应先读。

## 关键约定

- 两个 workflow 都由 `workflow_dispatch`（手动，带布尔 inputs）或 `push: master` 触发。**push 触发时 inputs 为空**，靠 env 里的 `|| 'true'` / `|| 'false'` fallback 决定行为——改默认行为就改这些 fallback。
- 镜像目标路径 / tag 由开关控制：`KEEP_IMAGE_NAMESPACE`（是否保留 `bitnami/` 段）、`KEEP_IMAGE_ORIGINAL_TAG`（关则用 chart 版本号当 tag）。
- Chart 目标路径由 `KEEP_CHART_NAMESPACE` 控制；`CHART_PLAIN_HTTP` 决定 `helm push` 走 `--plain-http` 还是 HTTPS（ACR 个人版不支持 OCI chart，故 chart 推 TCR）。
- 所有 registry 凭据来自 GitHub Secrets（`ACR_*` / `TCR_*`），脚本中不硬编码。
- 磁盘空间紧张：docker.yaml 用 `easimon/maximize-build-space` 扩容，两个脚本都在每次 push 后 `docker rmi` 释放空间。

## 日常操作

改搬运清单 = 编辑 `.properties`（注释/取消注释行）。改搬运逻辑 = 编辑对应 workflow 的 bash 脚本。本仓库无本地构建、lint 或测试命令；验证靠在 GitHub Actions 上实跑 workflow。
