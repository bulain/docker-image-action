# 合并 ACR_REGISTRY_IMAGES 到 ACR_REGISTRY_NS 设计

日期：2026-07-11

## 目标

删除 `ACR_REGISTRY_IMAGES` 变量，其所有用途改用 `ACR_REGISTRY_NS`。改动后 Docker 镜像与 Chart 引用的镜像推送到**同一个 ACR 命名空间**，减少一个需要维护的 Secret。

## 背景

当前两个变量分工：

- `ACR_REGISTRY_NS` —— Docker 流程（`docker.yaml`）镜像目标命名空间。
- `ACR_REGISTRY_IMAGES` —— Helm/git 流程（`helm.yaml`、`git-charts.yaml`）中 Chart 引用镜像的目标命名空间。

两者本质都是「镜像推到 ACR 的哪个命名空间」，无需拆成两个。合并后统一由 `ACR_REGISTRY_NS` 承担。

## 改动清单

### 1. `.github/workflows/helm.yaml`
- env 段（约 L38）：删除 `ACR_REGISTRY_IMAGES: "${{ secrets.ACR_REGISTRY_IMAGES }}"`，改为 `ACR_REGISTRY_NS: "${{ secrets.ACR_REGISTRY_NS }}"`。
- 镜像目标路径（约 L144）：`$ACR_REGISTRY_IMAGES` → `$ACR_REGISTRY_NS`。

### 2. `.github/workflows/git-charts.yaml`
- env 段（约 L38）：同上（删 IMAGES，加 NS）。
- 镜像目标路径（约 L152）：`$ACR_REGISTRY_IMAGES` → `$ACR_REGISTRY_NS`。

### 3. `.github/workflows/docker.yaml`
- 已在用 `ACR_REGISTRY_NS`，无需改动。

### 4. `README.md`
- Secrets 表（L52-53）：删除 `ACR_REGISTRY_IMAGES` 行；`ACR_REGISTRY_NS` 说明改为「Docker 镜像 + Helm/git Chart 引用镜像的统一命名空间」。
- 示例说明（L148）：`ACR_REGISTRY_IMAGES` → `ACR_REGISTRY_NS`。

### 5. Spec 文档同步
- `docs/superpowers/specs/2026-07-09-helm-charts-mirror-design.md` 中的 `ACR_REGISTRY_IMAGES` 引用 → `ACR_REGISTRY_NS`。
- `docs/superpowers/specs/2026-07-11-git-charts-mirror-design.md` 中的 `ACR_REGISTRY_IMAGES` 引用 → `ACR_REGISTRY_NS`。

## GitHub 侧操作（脚本外，仅提醒）

- 删除 `ACR_REGISTRY_IMAGES` Secret。
- 确认 `ACR_REGISTRY_NS` 对应命名空间能同时容纳 Docker 镜像与 Chart 引用镜像。

## 验证

- 本仓库无本地测试，靠 GitHub Actions 实跑 workflow。
- 改完后全仓库 grep 确认无 `ACR_REGISTRY_IMAGES` 残留（本 spec 文件除外）。
