# images/charts 清单改为 YAML 格式设计

日期：2026-07-11

## 背景

`images.properties`、`charts.properties` 目前是纯文本清单，逐行一个条目，
用 `#` 注释与空行控制。两个 workflow（`docker.yaml` / `helm.yaml`）逐行 read 文件，
再手动跳过空行与 `#` 注释行。

为与新增的 `git-charts.yaml`（已是结构化 YAML）风格统一、便于用 `yq` 解析，
把两个清单改为 YAML 格式。

## 目标与范围

- `images.properties` → `images.yaml`
- `charts.properties` → `charts.yaml`
- 同步改 `.github/workflows/docker.yaml`、`helm.yaml` 的读取逻辑：
  从"逐行 read 文件 + 手动跳过空行/注释"改为 `yq` 取数组元素。

**不改**：搬运逻辑本身（`--platform` 解析、tag/namespace 开关、pull/tag/push/rmi 清理等一律保留）；
不改 `git-charts.yaml`。

## YAML 结构

顶层单一 list，每条目是**一行字符串**，内容等于原 `.properties` 的一行。
`#` 注释保留两种用途：分组说明、以及注释掉条目=禁用（复刻现有"注释=禁用"习惯）。

### `images.yaml`

```yaml
# Docker images list. 注释掉某行即禁用该条目。
# 短写默认取自 docker.io；可用 --platform linux/amd64 指定架构。
images:
  # https://github.com/rabbitmq/cluster-operator
  # - ghcr.io/rabbitmq/cluster-operator:2.22.1
  # - mysql:9.7
  # - valkey/valkey:9.1-alpine
```

### `charts.yaml`

```yaml
# Helm charts list, oci://.../name:version。注释掉某行即禁用该条目。
charts:
  # https://github.com/VictoriaMetrics/helm-charts
  # - oci://ghcr.io/victoriametrics/helm-charts/victoria-metrics-cluster:0.46.0
```

- 现有条目几乎全是注释状态，迁移后维持原样（该注释的仍注释）。
- 字符串内容原样保留 `--platform ...` 前缀、`oci://...:version`，workflow 解析逻辑不变。
- 分组用的 `# https://github.com/...` 注释行照迁。

## Workflow 读取逻辑改动

把"逐行 read 文件 + 手动跳过空行/注释"换成 `yq` 取数组，其余搬运逻辑完全不动。

### docker.yaml（末尾循环）

```bash
count=$(yq '.images | length' images.yaml)
for i in $(seq 0 $((count - 1))); do
    line=$(yq ".images[$i]" images.yaml)
    # 以下沿用现有逻辑：--platform 解析、image 取名、KEEP_IMAGE_NAMESPACE、push、rmi
done
```

### helm.yaml（主循环）

数据源从 `< charts.properties` 换成 `yq`：

```bash
count=$(yq '.charts | length' charts.yaml)
for i in $(seq 0 $((count - 1))); do
    entry=$(yq ".charts[$i]" charts.yaml)
    # 以下沿用现有逻辑：拆 tag/ref、helm images get、KEEP_* 开关、helm pull/push
done
```

### 要点

- `yq`（mikefarah/Go 版，GitHub runner 预装）取数组元素输出无引号原始字符串，与原 read 到的行内容一致。
- 注释掉的 `- xxx` 行 `yq` 读不到，天然实现"注释=禁用"。
- 数组为空/全注释时 `length` 为 0，循环不执行——等价于现在全注释时不搬任何东西。
- 原先手动的"空行 / `#` 注释跳过"逻辑删除（YAML 层已处理）。

## 与现有文件的关系

- 删除 `images.properties`、`charts.properties`，新增 `images.yaml`、`charts.yaml`。
- 改 `docker.yaml`、`helm.yaml` 的文件名引用与读取循环。
- CLAUDE.md 中提到 `.properties` 的描述后续可一并更新（非本次搬运逻辑范畴，可选）。

## 验证

本仓库无本地运行手段，验证靠在 GitHub Actions 上实跑 workflow。
本地可用 `yq`/python 校验 YAML 语法与数组解析。
