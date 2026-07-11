# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 语言

**必须始终使用中文回答。**

## 用途

将公网 Docker 镜像与 Helm Chart 搬运（镜像）到私有 registry（阿里云 ACR / 腾讯云 TCR）。没有可构建的应用代码——核心逻辑全部是两个 GitHub Actions workflow 里的 bash 脚本。

## 架构

- 清单文件命名规则：清单名 = 读取它的 workflow 名（根目录清单与 `.github/workflows/` 下同名 workflow 一一对应，靠目录区分）。
- `docker.yaml`（根目录）— 待搬运的 Docker 镜像清单，YAML 顶层 `images` 数组，每条一行字符串。注释掉某行（保持 `#`）即禁用。短写（`mysql:9.7`）默认取自 docker.io；可用 `--platform linux/amd64 xxx` 指定架构。
- `helm.yaml`（根目录）— 待搬运的 Helm Chart 清单，YAML 顶层 `charts` 数组，格式 `oci://.../name:version`（`:` 后为 chart 版本）。短写补全为 `oci://registry-1.docker.io/`。注释掉某行即禁用。
- `git-charts.yaml` — git 源码 chart 清单（只存源码、未发布为 OCI 的 chart），`charts` 数组每条含 `repo/path/ref/namespace`。`namespace` 为目标前缀，helm push 自动在其后追加 chart 名。
- `config.yaml`（根目录）— 搬运行为开关统一配置，扁平小写下划线 key（`push_images`/`push_charts`/`keep_image_namespace`/`keep_chart_namespace`）。三个 workflow 运行时用 `yq '.key // "默认"' config.yaml` 读取（带 fallback，注释掉某行即回退默认）。docker 流程只读 `push_images`/`keep_image_namespace`。
- `.github/workflows/docker.yaml` — 用 `yq` 读根目录 `docker.yaml` 清单，`pull → tag → push → rmi`（逐个清理磁盘）。
- `.github/workflows/helm.yaml` — 用 `yq` 读根目录 `helm.yaml` 清单：用 `helm template` 渲染后提取 `image:`/`initImage:` 字段解析 chart 引用的镜像并搬运（推到 ACR，能识别 CR 里的镜像），再 `helm pull` chart 本身推到 TCR。
- `.github/workflows/git-charts.yaml` — 用 `yq` 读 `git-charts.yaml`：`git clone` + `helm package` 推 chart 到 TCR，`helm template` 提取镜像推到 ACR。
- `docs/superpowers/specs/` — 两份设计文档，改动搬运逻辑前应先读。

## 关键约定

- 两个 workflow 都由 `workflow_dispatch`（手动，无 inputs）或 `push: master` 触发。搬运行为开关不再走 workflow input，而是运行时从根目录 `config.yaml` 用 `yq` 读取（带 `// "默认"` fallback）——改默认行为就改 config.yaml 或脚本里的 fallback。
- 镜像目标路径由 `KEEP_IMAGE_NAMESPACE`（是否保留 `bitnami/` 段）控制；镜像 tag 固定保留原始 tag，无 tag 时回退到 chart 版本号。
- Chart 目标路径由 `KEEP_CHART_NAMESPACE` 控制；`TCR_PLAIN_HTTP`（来自 Secrets，可选，未配置默认 `true`）决定 `helm push` 走 `--plain-http` 还是 HTTPS（ACR 个人版不支持 OCI chart，故 chart 推 TCR）。
- `ACR_PLAIN_HTTP`（来自 Secrets，可选，未配置默认 `true`）决定 `docker push` 到 ACR 走明文 HTTP 还是 HTTPS。因 `docker push` 无 `--plain-http` flag，为 `true` 时在 login 前把 `$ACR_REGISTRY_ENDPOINT` 写进 `/etc/docker/daemon.json` 的 `insecure-registries` 并重启 docker（docker.yaml 合并进扩容后的那次 restart，helm/git-charts 在 login 前单独重启）。用于 ACR endpoint 指向内网 HTTP registry 的场景。
- 所有 registry 凭据来自 GitHub Secrets（`ACR_*` / `TCR_*`），脚本中不硬编码。
- 磁盘空间紧张：docker.yaml 用 `easimon/maximize-build-space` 扩容，两个脚本都在每次 push 后 `docker rmi` 释放空间。

## 日常操作

改搬运清单 = 编辑 `.yaml` 清单（注释/取消注释 `- xxx` 行）。改搬运开关（推不推、保不保留命名空间/tag）= 编辑根目录 `config.yaml`。改搬运逻辑 = 编辑对应 workflow 的 bash 脚本。本仓库无本地构建、lint 或测试命令；验证靠在 GitHub Actions 上实跑 workflow。
