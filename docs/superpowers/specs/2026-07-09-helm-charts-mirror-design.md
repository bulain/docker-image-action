# Helm Charts 镜像到阿里云 ACR — 设计文档

日期：2026-07-09
状态：已确认，待实现

## 背景

现有 `.github/workflows/docker.yaml` 通过 `images.txt` 清单把 Docker 镜像镜像到阿里云
ACR（登录 → pull → tag → push → 清理），并处理了重名命名空间和 `--platform`。

此前尝试把 Helm chart `bitnamicharts/redis:27.0.13` 加进 `images.txt`，但 chart 是 OCI
artifact，用 `docker pull` 处理并不合适，该行随后被回退。因此需要一套独立的 Helm chart
镜像流程。

## 目标

- 用一个独立的 GitHub Actions workflow，把 Helm chart（OCI 格式）镜像到阿里云 ACR。
- chart 清单风格与 `images.txt` 一致：每行一个，支持 `#` 注释与空行。
- 与镜像镜像流程解耦，保持各自脚本清晰。

## 非目标（YAGNI）

- 不做重名检测（helm 从 chart 元数据取名，跨命名空间不冲突）。
- 不处理 `--platform`（chart 与架构无关）。
- 不做磁盘扩容（chart 体积小，KB–MB 级）。
- 不支持传统 HTTP Helm 仓库（`helm repo add`）；只支持 OCI 来源。

## 新增文件

- `.github/workflows/helm.yaml` —— 工作流。
- `charts.txt` —— chart 清单。

## Secrets / 环境变量

复用现有 ACR secrets：

- `ACR_REGISTRY_ENDPOINT`
- `ACR_REGISTRY_USER`
- `ACR_REGISTRY_PASSWORD`

新增一个：

- `ACR_CHARTS` —— chart 推送到的 ACR 命名空间（如 `my-charts`），让 chart 与镜像分开存放。

> 注意：`ACR_REGISTRY_NAME`（镜像用的命名空间）不用于 chart；chart 使用独立的 `ACR_CHARTS`。

## 清单格式（`charts.txt`）

```
# 短写 → 默认从 docker.io 拉取
bitnamicharts/redis:27.0.13

# 完整写法 → 任意 OCI registry
oci://ghcr.io/some-org/some-chart:1.2.3
```

每行解析规则：

1. 去掉空行与注释行（以 `#` 开头，可含前导空白）——复用 `docker.yaml` 的判断逻辑。
2. 若行以 `oci://` 开头 → 原样作为来源引用；否则补前缀 `oci://registry-1.docker.io/`。
3. 拆出结尾的 `:tag`：`tag` 作为 `helm pull --version` 的值；其余部分作为 chart 引用（`ref`）。

## 工作流流程

触发与运行环境：

- 触发：`workflow_dispatch` + `push: [master]`（同 `docker.yaml`）。
- 运行环境：`ubuntu-latest`。
- 安装 helm：`azure/setup-helm@v4`。
- 拉取代码：`actions/checkout@v5`（与现有 workflow 版本一致）。

核心步骤（bash，`set -e`）：

```bash
helm registry login "$ACR_REGISTRY_ENDPOINT" -u "$ACR_REGISTRY_USER" -p "$ACR_REGISTRY_PASSWORD"

mkdir -p ./charts

while IFS= read -r line || [ -n "$line" ]; do
    # 忽略空行与注释
    [[ -z "$line" ]] && continue
    echo "$line" | grep -q '^\s*#' && continue

    # 拆出 tag（版本）与 chart 引用
    entry=$(echo "$line" | awk '{print $NF}')
    tag="${entry##*:}"
    ref="${entry%:*}"

    # 短写补全默认 registry
    case "$ref" in
        oci://*) ;;
        *) ref="oci://registry-1.docker.io/$ref" ;;
    esac

    echo "helm pull $ref --version $tag"
    helm pull "$ref" --version "$tag" -d ./charts

    # helm 生成 ./charts/<chartname>-<tag>.tgz
    tgz=$(ls ./charts/*.tgz)
    echo "helm push $tgz oci://$ACR_REGISTRY_ENDPOINT/$ACR_CHARTS"
    helm push "$tgz" "oci://$ACR_REGISTRY_ENDPOINT/$ACR_CHARTS"

    rm -f ./charts/*.tgz
done < charts.txt
```

结果：`oci://$ACR_REGISTRY_ENDPOINT/$ACR_CHARTS/redis:27.0.13`。chart 名与 tag 由
`helm push` 从 chart 元数据自动推导，无需手动改名或去重。

> 实现说明：`ref="${entry%:*}"` 依赖“最后一个冒号是 tag 分隔符”。因 `oci://` 中的冒号出现
> 在末段之前，对 `oci://host/path:tag` 用 `${entry##*:}` / `${entry%:*}` 仍能正确取到
> 结尾 tag；对短写 `repo/chart:tag` 同样成立。实现时需覆盖这两种输入验证。

## 错误处理

- `set -e`：任一 `helm pull` / `helm push` 失败即让整个 job 失败。
- 每行动作前 echo 来源引用与推送目标（沿用 `docker.yaml` 的详细日志风格）。
- 每次循环结束清理 `.tgz`，避免多 chart 时文件混淆。

## 测试与验证

- 提交前本地 `bash -n` 语法检查脚本片段。
- 首次运行用 `workflow_dispatch` 手动触发，`charts.txt` 只放一个已知 chart
  （`bitnamicharts/redis:27.0.13`）验证端到端。
- 检查 ACR 中出现 `$ACR_CHARTS/redis:27.0.13`，并可 `helm pull` 回来。
