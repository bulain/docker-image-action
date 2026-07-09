# Helm Charts 镜像到阿里云 ACR / TCR — 设计文档

日期：2026-07-09
状态：已实现

## 背景

现有 `.github/workflows/docker.yaml` 通过 `images.properties` 清单把 Docker 镜像镜像到
阿里云 ACR（登录 → pull → tag → push → 清理），并处理了 `--platform` 与命名空间。

此前尝试把 Helm chart `bitnamicharts/redis:27.0.13` 加进 `images.txt`，但 chart 是 OCI
artifact，用 `docker pull` 处理并不合适，该行随后被回退。因此需要一套独立的 Helm chart
镜像流程。

此外，chart 部署时会引用容器镜像（如 `docker.io/bitnami/redis:8.2.1`）。为让 chart 在
内网环境可用，需要在镜像 chart 的同时，解析出 chart 引用的镜像并一并镜像。

**关键约束（实现中发现）**：阿里云 ACR 个人版不支持通过 `helm push` 推送 OCI Helm chart
（服务端返回 `403 unknown manifest class for application/vnd.cncf.helm.config.v1+json`）。
因此最终方案把 **镜像** 推到 ACR，把 **chart** 推到一个支持 OCI chart 的独立 registry
（下称 TCR，通过 `TCR_REGISTRY_*` 配置，可指向腾讯云 TCR 或任意支持 OCI chart 的仓库）。

## 目标

- 用一个独立的 GitHub Actions workflow（`helm.yaml`），镜像 Helm chart（OCI 格式）。
- 同一流程中解析出每个 chart 引用的容器镜像，并镜像到 ACR。
- chart 清单风格与 `images.properties` 一致：每行一个，支持 `#` 注释与空行。
- 与镜像镜像流程解耦，保持各自脚本清晰。

## 非目标（YAGNI）

- 不处理 `--platform`（chart 与其镜像的架构处理沿用默认）。
- 不做磁盘扩容（靠逐个镜像 pull-push-rmi 控制磁盘占用）。
- 不支持传统 HTTP Helm 仓库（`helm repo add`）；只支持 OCI 来源。

## 涉及文件

- `.github/workflows/helm.yaml` —— 工作流。
- `charts.properties` —— chart 清单。

## 触发方式与开关

- 触发：`workflow_dispatch`（手动，可设开关）+ `push: [master]`（自动）。
- push 触发时 inputs 为空，各开关 fallback 到默认值。

| input | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `keep_image_namespace` | boolean | `false` | 推送镜像时是否保留原命名空间段（如 `bitnami/redis`）。|
| `keep_chart_namespace` | boolean | `true` | 推送 chart 时是否把源命名空间作为子路径（如 `bitnamicharts`）。|
| `chart_plain_http` | boolean | `true` | chart 操作是否走明文 HTTP（`--plain-http`），关闭则走 HTTPS。|

对应 env fallback：

```yaml
KEEP_IMAGE_NAMESPACE: "${{ github.event.inputs.keep_image_namespace || 'false' }}"
KEEP_CHART_NAMESPACE: "${{ github.event.inputs.keep_chart_namespace || 'true' }}"
CHART_PLAIN_HTTP:     "${{ github.event.inputs.chart_plain_http || 'true' }}"
```

## Secrets / 环境变量

镜像推送到 ACR：

- `ACR_REGISTRY_ENDPOINT` —— ACR 地址。
- `ACR_REGISTRY_AK` —— 登录用户名（AccessKey）。
- `ACR_REGISTRY_SK` —— 登录密码（Secret）。
- `ACR_REGISTRY_IMAGES` —— chart 引用的镜像推送到的 ACR 命名空间。

chart 推送到 TCR（支持 OCI chart 的独立 registry）：

- `TCR_REGISTRY_ENDPOINT` —— chart registry 地址。
- `TCR_REGISTRY_AK` —— 登录用户名。
- `TCR_REGISTRY_SK` —— 登录密码。
- `TCR_REGISTRY_NS` —— chart 推送到的命名空间。

## 清单格式（`charts.properties`）

```
# 短写 → 默认从 docker.io 拉取
bitnamicharts/redis:27.0.13

# 完整写法 → 任意 OCI registry
oci://ghcr.io/some-org/some-chart:1.2.3
```

每行解析规则：

1. 去掉空行与注释行（以 `#` 开头，可含前导空白）。
2. 若行以 `oci://` 开头 → 原样作为来源引用；否则补前缀 `oci://registry-1.docker.io/`。
3. 拆出结尾的 `:tag`：`tag` 作为 `helm pull --version` 的值；其余部分作为 chart 引用（`ref`）。

## 工具准备

- `azure/setup-helm@v4` —— 安装 helm。
- `helm-images` 插件 —— `helm plugin install https://github.com/nikhilsbhat/helm-images --verify=false`，
  提供 `helm images get <chart-ref>` 列出 chart 引用的镜像。（新版 helm 默认要求验证插件源，
  helm-images 不支持，故加 `--verify=false`。）
- runner 自带 docker，用于镜像的 pull/tag/push；无需额外安装 skopeo。

## 目标路径与命名规则

### 镜像 → ACR

```
$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_IMAGES/<name_path>:<chart 版本号>
```

- **tag 用 chart 版本号**（`$tag`），而非镜像自身的原 tag。同一 chart 引用的所有镜像
  都带相同的 chart 版本 tag。
- **name_path** 受 `keep_image_namespace` 控制：
  - `false`（默认）：只留镜像名，如 `docker.io/bitnami/redis:8.2.1` → `redis`。
  - `true`：保留命名空间段（去 registry 域名），如 → `bitnami/redis`。
- 镜像列表经 `awk '!seen[$0]++'` 按原始字符串去重（保留原顺序），避免同一 chart 内
  重复镜像被多次搬运。

### chart → TCR

```
oci://$TCR_REGISTRY_ENDPOINT/$TCR_REGISTRY_NS[/<源命名空间>]
```

- **源命名空间**受 `keep_chart_namespace` 控制：
  - `true`（默认）：取源 ref 的命名空间段作子路径，如 `bitnamicharts`，最终推到
    `.../$TCR_REGISTRY_NS/bitnamicharts`（helm 再自动追加 chart 名 `redis`）。
  - `false`：直接推到 `.../$TCR_REGISTRY_NS`。
- chart 名与 tag 由 `helm push` 从 chart 元数据自动推导。

## 工作流流程

运行环境 `ubuntu-latest`，`actions/checkout@v5`。核心步骤（bash，`set -e`）：

```bash
set -e
# 镜像推送到 ACR
docker login -u "$ACR_REGISTRY_AK" -p "$ACR_REGISTRY_SK" "$ACR_REGISTRY_ENDPOINT"
# chart 走 HTTP 还是 HTTPS
if [ "$CHART_PLAIN_HTTP" = "true" ]; then
    HELM_HTTP_FLAG="--plain-http"
else
    HELM_HTTP_FLAG=""
fi
# chart 推送到支持 OCI chart 的 registry（ACR 个人版不支持 helm chart）
helm registry login "$TCR_REGISTRY_ENDPOINT" -u "$TCR_REGISTRY_AK" -p "$TCR_REGISTRY_SK" $HELM_HTTP_FLAG

mkdir -p ./charts

while IFS= read -r line || [ -n "$line" ]; do
    [[ -z "$line" ]] && continue
    echo "$line" | grep -q '^\s*#' && continue

    entry=$(echo "$line" | awk '{print $NF}')
    tag="${entry##*:}"
    ref="${entry%:*}"
    case "$ref" in
        oci://*) ;;
        *) ref="oci://registry-1.docker.io/$ref" ;;
    esac

    # ---- 1. 解析 chart 引用的镜像并逐个搬运 ----
    while read -r img; do
        [[ -z "$img" ]] && continue
        docker pull "$img"
        path="${img%%@*}"; path="${path%:*}"
        if [ "$KEEP_IMAGE_NAMESPACE" = "true" ]; then
            image_name=$(echo "$path" | awk -F'/' '{if (NF>=2) print $(NF-1)"/"$NF; else print $NF}')
        else
            image_name=$(echo "$path" | awk -F'/' '{print $NF}')
        fi
        new_img="$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_IMAGES/$image_name:$tag"
        docker tag "$img" "$new_img"
        docker push "$new_img"
        docker rmi "$img" "$new_img" || true
    done < <(helm images get "$ref" --version "$tag" | awk '!seen[$0]++')

    # ---- 2. push chart 本身 ----
    helm pull "$ref" --version "$tag" -d ./charts
    tgz=$(ls ./charts/*.tgz)
    if [ "$KEEP_CHART_NAMESPACE" = "true" ]; then
        chart_ns=$(echo "$ref" | sed 's#^oci://##' | awk -F'/' '{if (NF>=2) print $(NF-1)}')
        chart_target="oci://$TCR_REGISTRY_ENDPOINT/$TCR_REGISTRY_NS/$chart_ns"
    else
        chart_target="oci://$TCR_REGISTRY_ENDPOINT/$TCR_REGISTRY_NS"
    fi
    helm push "$tgz" "$chart_target" $HELM_HTTP_FLAG
    rm -f ./charts/*.tgz
done < charts.properties
```

## 错误处理

- `set -e`：任一 `docker`/`helm` 命令失败即让整个 job 失败。
- 每个动作前 echo 来源与目标（沿用 `docker.yaml` 的详细日志风格）。
- 镜像循环用进程替换 `done < <(...)` 而非管道 `|`，让 `set -e` 在循环体内正确生效。
- 每次镜像循环结束 `docker rmi ... || true` 清理；每个 chart 结束清理 `.tgz`。

## 已知权衡

1. **镜像 tag 统一为 chart 版本号**：便于按 chart 版本追溯，但丢失了镜像自身的原 tag。
2. **同名镜像覆盖**：`keep_image_namespace=false` 时只取镜像名，来源不同但同名的镜像会在
   ACR 撞名互相覆盖。bitnami 单一生态场景通常无此问题；需区分时开启 `keep_image_namespace`。
3. **私有源镜像**：`docker login` 只登录了 ACR。若 chart 引用的镜像来自需要认证的私有源，
   `docker pull` 会失败；公共镜像不受影响。
4. **ACR 个人版不支持 chart**：故 chart 单独推 TCR；若升级 ACR 企业版可考虑合并回 ACR。

## 测试与验证

- 提交前对脚本片段做 `bash -n` 语法检查，并验证命名空间/tag 提取逻辑。
- 首次运行用 `workflow_dispatch` 手动触发，`charts.properties` 只放一个已知 chart
  （`bitnamicharts/redis:27.0.13`）验证端到端。
- 检查镜像出现在 ACR 的 `$ACR_REGISTRY_IMAGES/` 下、chart 出现在 TCR 的
  `$TCR_REGISTRY_NS/` 下，并可分别 `docker pull` / `helm pull` 回来。
