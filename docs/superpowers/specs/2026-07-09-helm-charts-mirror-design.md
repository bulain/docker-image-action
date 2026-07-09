# Helm Charts 镜像到阿里云 ACR — 设计文档

日期：2026-07-09
状态：已确认，待实现

## 背景

现有 `.github/workflows/docker.yaml` 通过 `images.txt` 清单把 Docker 镜像镜像到阿里云
ACR（登录 → pull → tag → push → 清理），并处理了重名命名空间和 `--platform`。

此前尝试把 Helm chart `bitnamicharts/redis:27.0.13` 加进 `images.txt`，但 chart 是 OCI
artifact，用 `docker pull` 处理并不合适，该行随后被回退。因此需要一套独立的 Helm chart
镜像流程。

此外，chart 部署时会引用容器镜像（如 `docker.io/bitnami/redis:8.2.1`）。为让 chart 在
内网 ACR 环境可用，需要在镜像 chart 的同时，解析出 chart 引用的镜像并一并镜像到 ACR。

## 目标

- 用一个独立的 GitHub Actions workflow，把 Helm chart（OCI 格式）镜像到阿里云 ACR。
- 同一流程中解析出每个 chart 引用的容器镜像，并镜像到 ACR。
- chart 清单风格与 `images.txt` 一致：每行一个，支持 `#` 注释与空行。
- 与镜像镜像流程解耦，保持各自脚本清晰。

## 非目标（YAGNI）

- 不处理 `--platform`（chart 与其镜像的架构处理沿用默认）。
- 不做磁盘扩容（靠逐个镜像 pull-push-rmi 控制磁盘占用）。
- 不支持传统 HTTP Helm 仓库（`helm repo add`）；只支持 OCI 来源。
- 提取镜像时不做跨 chart 的重名检测（见「已知权衡」）。

## 新增文件

- `.github/workflows/helm.yaml` —— 工作流。
- `charts.txt` —— chart 清单。

## Secrets / 环境变量

复用现有 ACR 凭据：

- `ACR_REGISTRY_ENDPOINT` —— ACR 地址。
- `ACR_REGISTRY_USER` —— 登录用户。
- `ACR_REGISTRY_SECRET` —— 登录密码；`docker login` 与 `helm registry login` 共用同一个。

新增两个命名空间 secret：

- `ACR_REGISTRY_IMAGES` —— chart 引用的镜像推送到的 ACR 命名空间。
- `ACR_REGISTRY_CHARTS` —— chart 本身推送到的 ACR 命名空间。

> 说明：本流程不使用 docker.yaml 的 `ACR_REGISTRY_NAME`；镜像与 chart 各用一个专门的
> 新 secret，命名对称。

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

## 工具准备

- `azure/setup-helm@v4` —— 安装 helm。
- `helm-images` 插件 —— `helm plugin install https://github.com/nikhilsbhat/helm-images`，
  提供 `helm images get <chart-ref>` 列出 chart 引用的镜像。
- runner 自带 docker，用于镜像的 pull/tag/push；无需额外安装 skopeo。

## 目标路径

- chart → `oci://$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_CHARTS`。
- 镜像 → `$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_IMAGES/<镜像名>:<tag>`（只取镜像名与 tag，
  丢弃来源 registry 与命名空间前缀）。

## 工作流流程

触发与运行环境：

- 触发：`workflow_dispatch` + `push: [master]`（同 `docker.yaml`）。
- 运行环境：`ubuntu-latest`。
- 拉取代码：`actions/checkout@v5`（与现有 workflow 版本一致）。

核心步骤（bash，`set -e`）：

```bash
# 两处登录，凭据复用同一 ACR
docker login -u "$ACR_REGISTRY_USER" -p "$ACR_REGISTRY_SECRET" "$ACR_REGISTRY_ENDPOINT"
helm registry login "$ACR_REGISTRY_ENDPOINT" -u "$ACR_REGISTRY_USER" -p "$ACR_REGISTRY_SECRET"

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

    # ---- 1. 解析 chart 引用的镜像并逐个搬运 ----
    echo "helm images get $ref --version $tag"
    helm images get "$ref" --version "$tag" | while read -r img; do
        [[ -z "$img" ]] && continue
        echo "docker pull $img"
        docker pull "$img"
        # 去掉 @sha256 摘要，只取「镜像名:tag」
        name_tag=$(echo "${img%%@*}" | awk -F'/' '{print $NF}')
        new_img="$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_IMAGES/$name_tag"
        echo "docker tag $img $new_img"
        docker tag "$img" "$new_img"
        echo "docker push $new_img"
        docker push "$new_img"
        # 逐个清理，控制磁盘
        docker rmi "$img" "$new_img"
    done

    # ---- 2. push chart 本身 ----
    echo "helm pull $ref --version $tag"
    helm pull "$ref" --version "$tag" -d ./charts
    tgz=$(ls ./charts/*.tgz)
    echo "helm push $tgz oci://$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_CHARTS"
    helm push "$tgz" "oci://$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_CHARTS"
    rm -f ./charts/*.tgz
done < charts.txt
```

结果：

- chart：`oci://$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_CHARTS/redis:27.0.13`。chart 名与
  tag 由 `helm push` 从 chart 元数据自动推导。
- 镜像：`$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_IMAGES/redis:8.2.1` 等。

> 实现说明：`helm images get` 通过 `helm template` 渲染并提取镜像，`helm-images` 需能访问
> chart。若某些 chart 的镜像在 values 中以非标准字段声明，可能提取不全；实现时以 bitnami
> 系列 chart 为主验证。

## 错误处理

- `set -e`：任一 `docker`/`helm` 命令失败即让整个 job 失败。
- 每个动作前 echo 来源与目标（沿用 `docker.yaml` 的详细日志风格）。
- 每次镜像循环结束 `docker rmi` 清理；每个 chart 结束清理 `.tgz`。
- 注意：`| while` 子 shell 中 `set -e` 的失败传播有限，实现时对镜像循环显式检查
  `docker pull`/`push` 返回码（或改用进程替换避免子 shell）。

## 已知权衡

1. **同名镜像覆盖**：只取「镜像名:tag」，来源不同但同名的镜像（如 `bitnami/redis` 与
   `library/redis`）在 ACR 会撞名互相覆盖。当前 bitnami 单一生态场景通常无此问题；如需区分，
   后续可改为保留命名空间前缀。
2. **私有源镜像**：`docker login` 只登录了 ACR。若 chart 引用的镜像来自需要认证的私有源，
   `docker pull` 会失败；公共镜像不受影响。

## 测试与验证

- 提交前对脚本片段做 `bash -n` 语法检查。
- 首次运行用 `workflow_dispatch` 手动触发，`charts.txt` 只放一个已知 chart
  （`bitnamicharts/redis:27.0.13`）验证端到端。
- 检查 ACR 中出现 `$ACR_REGISTRY_CHARTS/redis:27.0.13`，以及该 chart 引用的镜像出现在
  `$ACR_REGISTRY_IMAGES/` 下，并可分别 `helm pull` / `docker pull` 回来。
