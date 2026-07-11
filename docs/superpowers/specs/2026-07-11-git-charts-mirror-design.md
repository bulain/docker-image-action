# Git 源 Helm Chart 搬运设计

日期：2026-07-11

## 背景

现有 `helm.yaml` 只能搬运**已发布为 OCI 制品**的 Helm Chart（`helm pull oci://...`）。
但部分 chart（如 Percona 的 operator chart）只以**源码**形式存在于 git 仓库中，
未发布为 OCI，无法用现有流程处理，必须先 `git clone` + `helm package` 才能得到 `.tgz` 产物。

首个用例：`percona/percona-helm-charts` 仓库的两个 chart：

- `charts/ps-operator`（tag `ps-operator-1.2.0`，chart version 1.2.0）：templates 只部署 operator 自身，
  `helm images get` 解析出的镜像仅 operator 主镜像 `percona/percona-server-mysql-operator:1.2.0` 一个。
- `charts/ps-db`（tag `ps-db-1.2.0`，chart version 1.2.0）：部署数据库实例，
  引用 percona-server、haproxy、mysql-router、orchestrator、pmm-client、xtrabackup、toolkit 等多个组件镜像，
  以 `helm images get` 实际解析结果为准。

## 目标与产物

新增一个**独立** workflow，处理"git 源码 chart"。对清单中每个 chart：

- **Chart 本身** → `git clone` + `helm package` 打包成 `.tgz` → 以 OCI 格式 `helm push` 到 **TCR**。
- **Chart 引用的镜像** → `helm images get` 解析全部镜像 → `docker pull → tag → push` 到 **ACR** → 逐个 `docker rmi` 释放磁盘。

分工：chart 传 TCR，images 传 ACR（与 `helm.yaml` 一致）。

## 清单文件 `git-charts.yaml`（仓库根目录）

结构化 YAML，顶层 `charts` 数组，每条四字段：

```yaml
# Git 源 Helm Chart 清单：仓库源码 chart，需 git clone + helm package 后搬运
# repo: git 仓库地址  path: chart 子路径  ref: git 分支/tag/commit  namespace: chart 目标前缀
charts:
  - repo: https://github.com/percona/percona-helm-charts.git
    path: charts/ps-operator
    ref: ps-operator-1.2.0
    namespace: percona/helm-charts
  - repo: https://github.com/percona/percona-helm-charts.git
    path: charts/ps-db
    ref: ps-db-1.2.0
    namespace: percona/helm-charts
```

- `ref` 用于 `git checkout`（支持分支/tag/commit）。
- `namespace` 为 chart 推到 TCR 时的目标前缀（可多段，如 `percona/helm-charts` → `.../percona/helm-charts/ps-operator`），受 `KEEP_CHART_NAMESPACE` 控制。
- chart 版本号由 `helm package` 自动从 `Chart.yaml` 读取，**不在清单里重复写版本**。
- 将来加 pg-operator、psmdb-operator 等只需追加数组项。
- workflow 里用 `yq`（GitHub runner 预装）解析。

## Workflow `.github/workflows/git-charts.yaml`

### 触发

`workflow_dispatch`（带布尔 inputs）+ `push: master`。
push 触发时 inputs 为空，靠 env 里的 `|| 'true'` / `|| 'false'` fallback 决定默认行为，
与现有两个 workflow 约定一致。

### 开关（env，复用 helm.yaml 语义）

| env | 默认 | 作用 |
|-----|------|------|
| `PUSH_CHARTS` | true | 是否推 chart 到 TCR |
| `PUSH_IMAGES` | true | 是否推镜像到 ACR |
| `KEEP_IMAGE_NAMESPACE` | false | 推镜像是否保留命名空间段（如 `percona/xxx`） |
| `KEEP_IMAGE_ORIGINAL_TAG` | true | 推镜像是否保留原始 tag（关则用 chart 版本号） |
| `KEEP_CHART_NAMESPACE` | true | 推 chart 是否加子路径 |
| `CHART_PLAIN_HTTP` | true | helm push 走 `--plain-http`（关则 HTTPS） |

### Secrets

- `ACR_*`（`ACR_REGISTRY_ENDPOINT` / `_AK` / `_SK` / `_IMAGES`）— 镜像目标。
- `TCR_*`（`TCR_REGISTRY_ENDPOINT` / `_AK` / `_SK` / `_NS`）— chart 目标。

脚本中不硬编码凭据。

### 步骤

1. **磁盘扩容**：percona operator 依赖镜像多且大，比照 `docker.yaml` 用
   `easimon/maximize-build-space` 扩容并 restart docker。
2. Checkout 本仓库 → `azure/setup-helm` → 装 `helm-images` 插件（同 helm.yaml）。
3. 登录：`PUSH_IMAGES=true` 时 `docker login` ACR；`PUSH_CHARTS=true` 时 `helm registry login` TCR。
4. 主脚本：`yq` 读 `git-charts.yaml`，遍历每条 `{repo, path, ref}`：
   - `git clone --depth 1 --branch <ref> <repo>` → 进入 `<path>` 目录。
   - `helm dependency build`（若 chart 声明了子依赖）。
   - **镜像**（PUSH_IMAGES）：`helm images get . | awk '!seen[$0]++'` 去重后逐个
     `pull → tag → push` 到 `$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_IMAGES/...`，逐个 `rmi`；
     沿用 helm.yaml 的 `KEEP_IMAGE_NAMESPACE` / `KEEP_IMAGE_ORIGINAL_TAG`（含 tag 回退到 chart 版本号）逻辑。
   - **chart**（PUSH_CHARTS）：`helm package .` 得 `.tgz` →
     `helm push <tgz> oci://$TCR_REGISTRY_ENDPOINT/$TCR_REGISTRY_NS[/子路径] $HELM_HTTP_FLAG`。

### chart 子路径规则

`KEEP_CHART_NAMESPACE=true` 时用清单里的 `namespace` 字段作子路径，
chart 推到 `oci://$TCR/$TCR_NS/<namespace>/<chart名>`（如 `oci://$TCR/$TCR_NS/percona/helm-charts/ps-operator`，
chart 名由 `helm push` 从包名自动决定）；
关闭则直推 `oci://$TCR/$TCR_NS`（丢前缀）。

## 与现有文件的关系

- 不修改 `docker.yaml` / `helm.yaml` / `images.properties` / `charts.properties`。
- 新增 `git-charts.yaml`（清单）与 `.github/workflows/git-charts.yaml`（workflow）。

## 验证

本仓库无本地构建/测试命令，验证靠在 GitHub Actions 上实跑该 workflow。
