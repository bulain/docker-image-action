# docker-image-action

把公网的 Docker 镜像与 Helm Chart 批量「搬运」（镜像）到私有 registry —— 阿里云 ACR 与腾讯云 TCR，供内网 / 受限网络环境拉取使用。

- 镜像清单维护在 `docker.yaml`，Chart 清单维护在 `helm.yaml`，YAML 数组每行一个条目，注释掉某行（保持 `#`）即禁用。
- 由 GitHub Actions 执行：读清单 → `pull → tag → push → 清理`。
- 无需本地环境，改清单、推 master（或手动触发）即可跑。

## 工作原理

项目有**三条相互独立**的搬运流程，各自对应一个 workflow 和一份清单：

| 流程 | 清单文件 | Workflow | 目标 registry |
| --- | --- | --- | --- |
| Docker 镜像 | `docker.yaml` | `.github/workflows/docker.yaml` | 阿里云 ACR |
| Helm Chart（OCI） | `helm.yaml` | `.github/workflows/helm.yaml` | 镜像 → ACR，Chart 本身 → 腾讯云 TCR |
| Git 源码 Chart | `git-charts.yaml` | `.github/workflows/git-charts.yaml` | 镜像 → ACR，Chart 本身 → 腾讯云 TCR |

**为什么 Chart 推 TCR**：Helm Chart 是 OCI artifact，阿里云 ACR 个人版不支持存储 Chart，因此 Chart 本身推到 TCR；Chart 引用的镜像仍推到 ACR。

- **Docker 流程**：逐条读 `docker.yaml` → `docker pull` → 重打 tag 到 ACR → `docker push` → `docker rmi` 释放磁盘。
- **Helm 流程**：逐条读 `helm.yaml` → 用 `helm template` 渲染后提取 `image:`/`initImage:` 字段解析 Chart 引用的镜像并逐个搬到 ACR（能识别 CR 里的镜像）→ `helm pull` 拉取 Chart 本身并 `helm push` 到 TCR。
- **Git 源码 Chart 流程**：逐条读 `git-charts.yaml` → `git clone` 指定 repo/ref → 定位 chart 目录 → `helm template` 提取镜像搬到 ACR → `helm package` 打包 Chart 并 `helm push` 到 TCR。用于只有源码、未发布为 OCI 的 Chart。

## 快速上手

**3 步搬运一个镜像 / Chart：**

1. **配置 Secrets 与 Variables**（仅首次）：在仓库 `Settings → Secrets and variables → Actions` 配好 ACR / TCR 凭据与配置，详见 [前置配置：GitHub Secrets 与 Variables](#前置配置github-secrets-与-variables)。
2. **编辑清单**：在 `docker.yaml`（镜像）或 `helm.yaml`（Chart）里，取消注释你要搬运的行，或新增一行。

   ```diff
   - #mysql:9.7
   + mysql:9.7
   ```

3. **触发搬运**：
   - **自动**：提交并推送到 `master` 分支，对应 workflow 自动运行。
   - **手动**：在 GitHub `Actions` 页选择 workflow → `Run workflow`，行为由 `config.yaml` 决定（无参数勾选）。

运行结束后，镜像 / Chart 就出现在你的私有 registry 里。最终路径规则见 [走查示例](#走查示例)。

## 前置配置：GitHub Secrets 与 Variables

在仓库 `Settings → Secrets and variables → Actions` 配置。**凭据（AK/SK）放 Secrets 标签页；域名、命名空间、明文 HTTP 开关放 Variables 标签页。**

### Secrets（凭据，敏感）

| Secret | 用途 | 示例 |
| --- | --- | --- |
| `ACR_REGISTRY_AK` | ACR 用户名 / AccessKey | `your-username` |
| `ACR_REGISTRY_SK` | ACR 密码 / Secret | `your-password` |
| `TCR_REGISTRY_AK` | TCR 用户名 | `your-username` |
| `TCR_REGISTRY_SK` | TCR 密码 | `your-password` |

### Variables（非敏感配置）

**阿里云 ACR（三条流程都用到）**

| Variable | 用途 | 示例 |
| --- | --- | --- |
| `ACR_REGISTRY_ENDPOINT` | ACR 域名 | `registry.cn-hangzhou.aliyuncs.com` |
| `ACR_REGISTRY_NS` | 镜像统一命名空间（Docker 流程 + Helm/git Chart 引用镜像） | `my-namespace` |
| `ACR_PLAIN_HTTP` | 可选。推送镜像到 ACR 是否走明文 HTTP（endpoint 为内网 HTTP registry 时用），未配置默认 `true` | `false` |

**腾讯云 TCR（Helm 与 Git 源码 Chart 流程推 Chart 用到）**

| Variable | 用途 | 示例 |
| --- | --- | --- |
| `TCR_REGISTRY_ENDPOINT` | TCR 域名 | `my-ns.tencentcloudcr.com` |
| `TCR_REGISTRY_NS` | Chart 推送到的命名空间 | `charts` |
| `TCR_PLAIN_HTTP` | 可选。`helm push` 到 TCR 是否走明文 HTTP（`--plain-http`），未配置默认 `true` | `false` |

## 清单格式

清单都是 YAML。`docker.yaml`（顶层 `images:`）与 `helm.yaml`（顶层 `charts:`）**一行一个条目**，注释掉某行（保持 `#`）即禁用，空行忽略；`git-charts.yaml`（顶层 `charts:`）每条是含 `repo/path/ref/namespace` 的对象。

### `docker.yaml`（Docker 镜像）

- **短写**：默认从 docker.io 拉取，如 `mysql:9.7`、`redis:7.4-alpine`。
- **完整写法**：任意 registry，如 `ghcr.io/valhalla/valhalla-scripted:3.6.2`、`quay.io/prometheus/prometheus:v3.13.0`。
- **指定架构**（可选）：行内加 `--platform`，如 `--platform linux/amd64 rancher/mirrored-pause:3.6`。搬运后架构信息会作为前缀拼进镜像名（如 `linux_amd64_pause:3.6`）。

```yaml
images:
  - mysql:9.7
  - ghcr.io/valhalla/valhalla-scripted:3.6.2
  - --platform linux/amd64 rancher/mirrored-pause:3.6
```

### `helm.yaml`（Helm Chart）

- 格式 `oci://<域名>/<路径>/<chart名>:<版本>`，`:` 后为 Chart 版本号。
- **短写**：省略 `oci://` 域名时默认补全为 `oci://registry-1.docker.io/`，如 `bitnamicharts/redis:27.0.13`。
- **完整写法**：`oci://ghcr.io/prometheus-community/charts/prometheus:29.14.0`。

```yaml
charts:
  - oci://ghcr.io/prometheus-community/charts/prometheus:29.14.0
```

### `git-charts.yaml`（Git 源码 Chart）

只存源码、未发布为 OCI 的 Chart。顶层 `charts:` 数组，每条含四个字段：

- `repo`：git 仓库地址。
- `path`：Chart 在仓库中的相对路径。
- `ref`：git 分支 / tag（`git clone --branch` 用）。
- `namespace`：推到 TCR 时的目标前缀，`helm push` 自动在其后追加 Chart 名。

```yaml
charts:
  - repo: https://github.com/minio/operator.git
    path: helm/operator
    ref: v7.1.1
    namespace: minio/helm-charts
```

### `config.yaml`（搬运行为开关）

根目录 `config.yaml` 集中放搬运行为开关，三个 workflow 运行时用 `yq` 读取。详见下方「触发方式与开关参考」。

## 触发方式与开关参考

**触发方式**（三条流程相同）：

- **`push: master`**：自动运行。
- **`workflow_dispatch`**：手动运行（无参数勾选，行为由 `config.yaml` 决定）。

**搬运行为开关**统一放在根目录 `config.yaml`，所有触发方式运行时用 `yq` 读取。改开关 = 编辑该文件（注释掉某行会回退到默认值）：

| 开关 | 默认 | 用于 | 说明 |
| --- | --- | --- | --- |
| `push_images` | `true` | 全部 | 是否推送镜像。关闭则跳过登录与搬运循环。 |
| `push_charts` | `true` | Helm / git | 是否推送 Chart 本身到 TCR。 |
| `keep_image_namespace` | `true` | 全部 | 镜像是否保留原命名空间段。`true` → `bitnami/redis`；`false` → `redis`。 |
| `keep_chart_namespace` | `true` | Helm / git | Chart 是否保留命名空间子路径（如 `grafana-community/helm-charts`）。 |

> Docker workflow 只读 `push_images` / `keep_image_namespace`，其余 chart 相关开关忽略。

> 明文 HTTP 开关（`TCR_PLAIN_HTTP` / `ACR_PLAIN_HTTP`）由 Variables 控制（不是 config.yaml），见上方 Variables 表。

## 走查示例

### 示例 A：搬运一个 Docker 镜像

清单 `docker.yaml` 中有一行：

```yaml
  - bitnami/redis:7.4
```

假设 `ACR_REGISTRY_ENDPOINT=registry.cn-hangzhou.aliyuncs.com`、`ACR_REGISTRY_NS=my-ns`，触发后：

| `keep_image_namespace` | 推送到的最终路径 |
| --- | --- |
| `false` | `registry.cn-hangzhou.aliyuncs.com/my-ns/redis:7.4` |
| `true`（默认） | `registry.cn-hangzhou.aliyuncs.com/my-ns/bitnami/redis:7.4` |

若行首带 `--platform linux/amd64`，镜像名再加前缀，如 `.../my-ns/linux_amd64_redis:7.4`。

### 示例 B：搬运一个 Helm Chart

清单 `helm.yaml` 中有一行：

```yaml
  - oci://ghcr.io/prometheus-community/charts/prometheus:29.14.0
```

假设 `TCR_REGISTRY_ENDPOINT=my.tencentcloudcr.com`、`TCR_REGISTRY_NS=charts`，触发后分两部分：

**① Chart 引用的镜像** → 推到 ACR（`ACR_REGISTRY_NS` 命名空间），tag 规则：保留镜像原始 tag，如 `prometheus:v3.13.0`；镜像无 tag 时回退用 Chart 版本号，如 `prometheus:29.14.0`。

**② Chart 本身** → 推到 TCR：

| `keep_chart_namespace` | 推送到的最终路径 |
| --- | --- |
| `true`（默认） | `oci://my.tencentcloudcr.com/charts/prometheus-community/charts/prometheus:29.14.0` |
| `false` | `oci://my.tencentcloudcr.com/charts/prometheus:29.14.0` |

## 常见问题 / 注意事项

- **磁盘空间**：GitHub Actions runner 磁盘有限。Docker 流程用 `easimon/maximize-build-space` 扩容，且三条流程都在每次 push 后 `docker rmi` 逐个清理镜像。搬运超大镜像仍可能失败。
- **某些 Chart `helm template` 渲染失败**：部分 Chart（如 Loki）模板渲染需要必填 values（如 `loki.storage.bucketNames.chunks`），未提供时解析镜像会报错（脚本已用 `2>/dev/null` 吞掉错误，仅表现为提取不到镜像、不中断）。此类 Chart 需在脚本中补 `--set` 传值，或单独处理。
- **ACR 个人版不支持 Helm Chart**：这是 Chart 本身改推 TCR 的原因，镜像仍在 ACR。
- **`push: master` 会自动触发**：只要清单或 workflow 变更推到 master，就会按默认开关跑一遍。不想自动跑时，注意提交内容或临时调整触发条件。
- **凭据安全**：registry 凭据（AK/SK）走 GitHub Secrets，域名/命名空间/开关走 GitHub Variables，脚本中均不硬编码。
