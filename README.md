# docker-image-action

把公网的 Docker 镜像与 Helm Chart 批量「搬运」（镜像）到私有 registry —— 阿里云 ACR 与腾讯云 TCR，供内网 / 受限网络环境拉取使用。

- 镜像清单维护在 `images.properties`，Chart 清单维护在 `charts.properties`，一行一个，支持 `#` 注释。
- 由 GitHub Actions 执行：读清单 → `pull → tag → push → 清理`。
- 无需本地环境，改清单、推 master（或手动触发）即可跑。

## 工作原理

项目有**两条相互独立**的搬运流程，各自对应一个 workflow 和一份清单：

| 流程 | 清单文件 | Workflow | 目标 registry |
| --- | --- | --- | --- |
| Docker 镜像 | `images.properties` | `.github/workflows/docker.yaml` | 阿里云 ACR |
| Helm Chart | `charts.properties` | `.github/workflows/helm.yaml` | 镜像 → ACR，Chart 本身 → 腾讯云 TCR |

**为什么 Chart 推 TCR**：Helm Chart 是 OCI artifact，阿里云 ACR 个人版不支持存储 Chart，因此 Chart 本身推到 TCR；Chart 引用的镜像仍推到 ACR。

- **Docker 流程**：逐行读 `images.properties` → `docker pull` → 重打 tag 到 ACR → `docker push` → `docker rmi` 释放磁盘。
- **Helm 流程**：逐行读 `charts.properties` → 用 `helm images get` 解析 Chart 引用的镜像并逐个搬到 ACR → `helm pull` 拉取 Chart 本身并 `helm push` 到 TCR。

## 快速上手

**3 步搬运一个镜像 / Chart：**

1. **配置 Secrets**（仅首次）：在仓库 `Settings → Secrets and variables → Actions` 配好 ACR / TCR 凭据，详见 [前置配置：GitHub Secrets](#前置配置github-secrets)。
2. **编辑清单**：在 `images.properties`（镜像）或 `charts.properties`（Chart）里，取消注释你要搬运的行，或新增一行。

   ```diff
   - #mysql:9.7
   + mysql:9.7
   ```

3. **触发搬运**：
   - **自动**：提交并推送到 `master` 分支，对应 workflow 自动运行。
   - **手动**：在 GitHub `Actions` 页选择 workflow → `Run workflow`，可临时调整开关。

运行结束后，镜像 / Chart 就出现在你的私有 registry 里。最终路径规则见 [走查示例](#走查示例)。

## 前置配置：GitHub Secrets

在仓库 `Settings → Secrets and variables → Actions` 添加以下 Secrets。

**阿里云 ACR（两条流程都用到）**

| Secret | 用途 | 示例 |
| --- | --- | --- |
| `ACR_REGISTRY_ENDPOINT` | ACR 域名 | `registry.cn-hangzhou.aliyuncs.com` |
| `ACR_REGISTRY_AK` | ACR 用户名 / AccessKey | `your-username` |
| `ACR_REGISTRY_SK` | ACR 密码 / Secret | `your-password` |
| `ACR_REGISTRY_NS` | **Docker 流程**镜像所在命名空间 | `my-namespace` |
| `ACR_REGISTRY_IMAGES` | **Helm 流程**中 Chart 镜像推送到的命名空间 | `my-images` |

**腾讯云 TCR（仅 Helm 流程推 Chart 用到）**

| Secret | 用途 | 示例 |
| --- | --- | --- |
| `TCR_REGISTRY_ENDPOINT` | TCR 域名 | `my-ns.tencentcloudcr.com` |
| `TCR_REGISTRY_AK` | TCR 用户名 | `your-username` |
| `TCR_REGISTRY_SK` | TCR 密码 | `your-password` |
| `TCR_REGISTRY_NS` | Chart 推送到的命名空间 | `charts` |

## 清单格式

两份清单都是纯文本，**一行一个条目**，`#` 开头为注释，空行忽略。

### `images.properties`（Docker 镜像）

- **短写**：默认从 docker.io 拉取，如 `mysql:9.7`、`redis:7.4-alpine`。
- **完整写法**：任意 registry，如 `ghcr.io/valhalla/valhalla-scripted:3.6.2`、`quay.io/prometheus/prometheus:v3.13.0`。
- **指定架构**（可选）：行首加 `--platform`，如 `--platform linux/amd64 rancher/mirrored-pause:3.6`。搬运后架构信息会作为前缀拼进镜像名（如 `linux_amd64_pause:3.6`）。

```properties
mysql:9.7
ghcr.io/valhalla/valhalla-scripted:3.6.2
--platform linux/amd64 rancher/mirrored-pause:3.6
```

### `charts.properties`（Helm Chart）

- 格式 `oci://<域名>/<路径>/<chart名>:<版本>`，`:` 后为 Chart 版本号。
- **短写**：省略 `oci://` 域名时默认补全为 `oci://registry-1.docker.io/`，如 `bitnamicharts/redis:27.0.13`。
- **完整写法**：`oci://ghcr.io/prometheus-community/charts/prometheus:29.14.0`。

```properties
oci://ghcr.io/prometheus-community/charts/prometheus:29.14.0
```

## 触发方式与开关参考

**触发方式**（两条流程相同）：

- **`push: master`**：自动运行。此时 inputs 为空，所有开关**回退到 env 里的默认值**（见下表 fallback 列）。
- **`workflow_dispatch`**：手动运行，可在 UI 逐项调整开关。

### Docker workflow 开关

| 开关 | 默认 (fallback) | 说明 |
| --- | --- | --- |
| `push_images` | `true` | 是否推送镜像。关闭则跳过登录与整个搬运循环。 |
| `keep_image_namespace` | `false` | 是否保留镜像原命名空间。`true` → `bitnami/redis`；`false` → `redis`。 |

### Helm workflow 开关

| 开关 | 默认 (fallback) | 说明 |
| --- | --- | --- |
| `push_charts` | `true` | 是否推送 Chart 本身到 TCR。 |
| `push_images` | `true` | 是否推送 Chart 引用的镜像到 ACR。 |
| `keep_image_namespace` | `false` | 镜像是否保留原命名空间段。 |
| `keep_image_original_tag` | `true` | `true` 用镜像原始 tag；`false` 用 Chart 版本号当 tag。 |
| `keep_chart_namespace` | `true` | Chart 是否保留域名之后的完整子路径（如 `grafana-community/helm-charts`）。 |
| `chart_plain_http` | `true` | `helm push` 用明文 HTTP（`--plain-http`）；`false` 用 HTTPS。 |

> **注意**：`push: master` 触发时全部按 fallback 值运行。要改自动触发的默认行为，需修改 workflow env 里的 `|| '...'` 回退值。

## 走查示例

### 示例 A：搬运一个 Docker 镜像

清单 `images.properties` 中有一行：

```properties
bitnami/redis:7.4
```

假设 `ACR_REGISTRY_ENDPOINT=registry.cn-hangzhou.aliyuncs.com`、`ACR_REGISTRY_NS=my-ns`，触发后：

| `keep_image_namespace` | 推送到的最终路径 |
| --- | --- |
| `false`（默认） | `registry.cn-hangzhou.aliyuncs.com/my-ns/redis:7.4` |
| `true` | `registry.cn-hangzhou.aliyuncs.com/my-ns/bitnami/redis:7.4` |

若行首带 `--platform linux/amd64`，镜像名再加前缀，如 `.../my-ns/linux_amd64_redis:7.4`。

### 示例 B：搬运一个 Helm Chart

清单 `charts.properties` 中有一行：

```properties
oci://ghcr.io/prometheus-community/charts/prometheus:29.14.0
```

假设 `TCR_REGISTRY_ENDPOINT=my.tencentcloudcr.com`、`TCR_REGISTRY_NS=charts`，触发后分两部分：

**① Chart 引用的镜像** → 推到 ACR（`ACR_REGISTRY_IMAGES` 命名空间），tag 规则：

- `keep_image_original_tag=true`（默认）：用镜像原始 tag，如 `prometheus:v3.13.0`。
- `keep_image_original_tag=false`：用 Chart 版本号，如 `prometheus:29.14.0`。

**② Chart 本身** → 推到 TCR：

| `keep_chart_namespace` | 推送到的最终路径 |
| --- | --- |
| `true`（默认） | `oci://my.tencentcloudcr.com/charts/prometheus-community/charts/prometheus:29.14.0` |
| `false` | `oci://my.tencentcloudcr.com/charts/prometheus:29.14.0` |

## 常见问题 / 注意事项

- **磁盘空间**：GitHub Actions runner 磁盘有限。Docker 流程用 `easimon/maximize-build-space` 扩容，且两条流程都在每次 push 后 `docker rmi` 逐个清理镜像。搬运超大镜像仍可能失败。
- **某些 Chart `helm images get` 失败**：部分 Chart（如 Loki）模板渲染需要必填 values（如 `loki.storage.bucketNames.chunks`），未提供时解析镜像会报错。此类 Chart 需在脚本中补 `--set` 传值，或单独处理。
- **ACR 个人版不支持 Helm Chart**：这是 Chart 本身改推 TCR 的原因，镜像仍在 ACR。
- **`push: master` 会自动触发**：只要清单或 workflow 变更推到 master，就会按默认开关跑一遍。不想自动跑时，注意提交内容或临时调整触发条件。
- **凭据安全**：所有 registry 凭据走 GitHub Secrets，脚本中不硬编码。
