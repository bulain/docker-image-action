# Docker 镜像镜像到阿里云 ACR — 设计文档

日期：2026-07-09
状态：已实现

## 背景

需要把公共 Docker 镜像批量镜像到阿里云 ACR，供内网/受限环境使用。镜像清单维护在
`images.properties`，工作流 `.github/workflows/docker.yaml` 按清单逐个拉取并推送
（登录 → pull → tag → push → 清理）。

Helm chart 的镜像流程是独立的另一套（见 `2026-07-09-helm-charts-mirror-design.md`）：
chart 是 OCI artifact，阿里云 ACR 个人版不支持，改用其他 registry。两套流程解耦，各自脚本清晰。

## 目标

- 用一个 GitHub Actions workflow，把 Docker 镜像批量镜像到阿里云 ACR。
- 清单风格简单：每行一个镜像，支持 `#` 注释与空行，可选 `--platform` 指定架构。
- 通过开关控制是否保留镜像原命名空间。

## 非目标（YAGNI）

- 不做跨镜像的重名检测（早期实现有，现已移除，改由命名空间开关统一处理）。
- 不支持 Helm chart（另有独立流程）。

## 涉及文件

- `.github/workflows/docker.yaml` —— 工作流。
- `images.properties` —— 镜像清单。

## 触发方式与开关

- 触发：`workflow_dispatch`（手动，可设开关）+ `push: [master]`（自动）。
- push 触发时 inputs 为空，开关 fallback 到默认值。

| input | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `keep_image_namespace` | boolean | `false` | 推送时是否保留镜像的原命名空间段（如 `bitnami/redis`）。|

对应 env fallback：

```yaml
KEEP_IMAGE_NAMESPACE: "${{ github.event.inputs.keep_image_namespace || 'false' }}"
ACR_PLAIN_HTTP:       "${{ github.event.inputs.acr_plain_http || 'true' }}"
```

`acr_plain_http`（默认 `true`）：推送镜像到 ACR 走明文 HTTP。因 `docker push` 无 `--plain-http` flag，为 `true` 时在扩容后重启 docker 前把 `$ACR_REGISTRY_ENDPOINT` 写进 `/etc/docker/daemon.json` 的 `insecure-registries`，用于 ACR endpoint 指向内网 HTTP registry 的场景。

## Secrets / 环境变量

镜像推送到 ACR：

- `ACR_REGISTRY_ENDPOINT` —— ACR 地址，如 `registry.cn-hangzhou.aliyuncs.com`。
- `ACR_REGISTRY_NS` —— 目标命名空间。
- `ACR_REGISTRY_AK` —— 登录用户名（AccessKey）。
- `ACR_REGISTRY_SK` —— 登录密码（Secret）。

## 清单格式（`images.properties`）

```
# 短写 → 默认从 docker.io 拉取
mysql:9.7

# 完整写法 → 任意 registry
ghcr.io/valhalla/valhalla-scripted:3.6.2

# 可选 --platform 指定架构
--platform linux/amd64 rancher/mirrored-pause:3.6
```

每行解析规则：

1. 去掉空行与注释行（以 `#` 开头，可含前导空白）。
2. 镜像名取行内最后一个字段（`--platform xxx` 之后的部分）。
3. `@sha256:...` 摘要会被去除。

## 工具准备

- `actions/checkout@v5` —— 拉取仓库（读取 `images.properties`）。
- `docker/setup-buildx-action@v4` —— 配置 buildx。
- `easimon/maximize-build-space` —— 扩容并重启 docker，避免大镜像拉取时磁盘不足。
- runner 自带 docker，用于镜像的 pull/tag/push。

## 目标路径与命名规则

```
$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_NS/[platform_prefix][name_path]
```

- **platform_prefix**：若行内有 `--platform`，把架构拼到名称前，`/` 替换为 `_`。
  例如 `--platform linux/arm64` → 前缀 `linux_arm64_`。
- **name_path**：受 `keep_image_namespace` 控制：
  - `false`（默认）：只留 `镜像名:tag`。如 `bitnami/redis:8.2.1` → `redis:8.2.1`。
  - `true`：保留命名空间段（去掉 registry 域名）。如 `docker.io/bitnami/redis:8.2.1`
    → `bitnami/redis:8.2.1`。

示例（`ENDPOINT=reg.example.com`、`NS=myns`）：

| 清单行 | keep=false | keep=true |
| --- | --- | --- |
| `mysql:9.7` | `reg.example.com/myns/mysql:9.7` | `reg.example.com/myns/mysql:9.7` |
| `valkey/valkey:8.1-alpine` | `.../myns/valkey:8.1-alpine` | `.../myns/valkey/valkey:8.1-alpine` |
| `--platform linux/arm64 rancher/klipper-lb:v0.4.17` | `.../myns/linux_arm64_klipper-lb:v0.4.17` | `.../myns/linux_arm64_rancher/klipper-lb:v0.4.17` |

## 工作流流程

运行环境 `ubuntu-latest`。核心步骤：

1. **释放磁盘空间**：`easimon/maximize-build-space` 扩容并重启 docker。
2. `actions/checkout@v5` + `docker/setup-buildx-action@v4`。
3. `docker login` 登录 ACR。
4. 逐行处理 `images.properties`。

镜像循环核心逻辑（bash）：

```bash
docker login -u $ACR_REGISTRY_AK -p $ACR_REGISTRY_SK $ACR_REGISTRY_ENDPOINT

while IFS= read -r line || [ -n "$line" ]; do
    [[ -z "$line" ]] && continue
    echo "$line" | grep -q '^\s*#' && continue

    docker pull $line
    platform=$(echo "$line" | awk -F'--platform[ =]' '{if (NF>1) print $2}' | awk '{print $1}')
    if [ -z "$platform" ]; then
        platform_prefix=""
    else
        platform_prefix="${platform//\//_}_"
    fi

    image=$(echo "$line" | awk '{print $NF}')
    image_no_digest="${image%%@*}"
    image_name_tag=$(echo "$image_no_digest" | awk -F'/' '{print $NF}')

    if [ "$KEEP_IMAGE_NAMESPACE" = "true" ]; then
        name_path=$(echo "$image_no_digest" | awk -F'/' '{if (NF>=2) print $(NF-1)"/"$NF; else print $NF}')
    else
        name_path="$image_name_tag"
    fi

    new_image="$ACR_REGISTRY_ENDPOINT/$ACR_REGISTRY_NS/$platform_prefix$name_path"
    docker tag $image $new_image
    docker push $new_image
    docker rmi $image
    docker rmi $new_image
done < images.properties
```

## 错误处理

- 每个动作前 echo 来源与目标（详细日志风格）。
- 每处理完一个镜像即 `docker rmi` 删除源镜像与新镜像，并打印 `df -hT`，控制 runner
  磁盘占用。

## 已知权衡

1. **同名镜像覆盖**：`keep_image_namespace=false` 时只取镜像名，来源不同但同名的镜像会在
   ACR 撞名互相覆盖。需区分时开启 `keep_image_namespace`。
2. **移除了重名检测**：早期实现会在清单内检测重名并自动加命名空间前缀；现已简化为由开关
   统一控制，行为更可预测。

## 测试与验证

- 提交前对脚本片段做 `bash -n` 语法检查，并验证命名规则（含 `--platform` 前缀）。
- 首次运行用 `workflow_dispatch` 手动触发，`images.properties` 只放一个已知镜像验证端到端。
- 检查 ACR 中出现对应镜像，并可 `docker pull` 回来。
