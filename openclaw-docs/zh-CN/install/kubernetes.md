---
read_when:
    - 你希望在 Kubernetes 集群上运行 OpenClaw
    - 你想在 Kubernetes 环境中测试 OpenClaw
summary: 使用 Kustomize 将 OpenClaw Gateway 网关部署到 Kubernetes 集群
title: Kubernetes
x-i18n:
    generated_at: "2026-07-26T06:12:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c05eb0eb923fa1f515aca1f6dcb6073aba69af0bdf30233243027edfedd45a39
    source_path: install/kubernetes.md
    workflow: 16
---

这是在 Kubernetes 上运行 OpenClaw 的一个最小起点，并非生产就绪的部署。它涵盖核心资源，旨在根据你的环境进行调整。

## 为什么不使用 Helm

OpenClaw 是一个包含若干配置文件的单容器应用。值得定制的是智能体内容（Markdown 文件、Skills、配置覆盖），而不是基础设施模板。Kustomize 可以处理覆盖层，且没有 Helm chart 的额外开销。如果部署变得更加复杂，可以在这些清单之上再添加 Helm chart。

## 所需条件

- 一个正在运行的 Kubernetes 集群（AKS、EKS、GKE、k3s、kind、OpenShift 等）
- `kubectl` 已连接到你的集群
- 至少一个模型提供商的 API 密钥

## 快速开始

```bash
# 替换为你的提供商：ANTHROPIC、GEMINI、OPENAI 或 OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh

kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

`deploy.sh` 默认创建令牌身份验证。获取生成的 Gateway 网关令牌以用于 Control UI：

```bash
kubectl get secret openclaw-secrets -n openclaw -o jsonpath='{.data.OPENCLAW_GATEWAY_TOKEN}' | base64 -d
```

在本地调试时，`./scripts/k8s/deploy.sh --show-token` 会在部署后输出令牌。

## 使用 Kind 进行本地测试

如果没有集群，可以使用 [Kind](https://kind.sigs.k8s.io/) 在本地创建一个：

```bash
./scripts/k8s/create-kind.sh           # 自动检测 docker 或 podman
./scripts/k8s/create-kind.sh --delete  # 拆除
```

然后照常使用 `./scripts/k8s/deploy.sh` 进行部署。

## 分步操作

### 1) 部署

**选项 A：通过环境变量设置 API 密钥（一步完成）**

```bash
# 替换为你的提供商：ANTHROPIC、GEMINI、OPENAI 或 OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh
```

该脚本会创建一个包含 API 密钥和自动生成的 Gateway 网关令牌的 Kubernetes Secret，然后执行部署。如果 Secret 已存在，它会保留当前的 Gateway 网关令牌以及所有未更改的提供商密钥。

**选项 B：单独创建 Secret**

```bash
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

在任一命令中添加 `--show-token`，即可将令牌输出到 stdout 以供本地测试。

### 2) 访问 Gateway 网关

```bash
kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

## 部署的内容

```text
命名空间：openclaw（可通过 OPENCLAW_NAMESPACE 配置）
├── Deployment/openclaw        # 单个 Pod，初始化容器 + Gateway 网关
├── Service/openclaw           # 端口 18789 上的 ClusterIP
├── PersistentVolumeClaim      # 用于智能体状态和配置的 10Gi 空间
├── ConfigMap/openclaw-config  # openclaw.json + AGENTS.md
└── Secret/openclaw-secrets    # Gateway 网关令牌 + API 密钥
```

## 自定义

### 智能体指令

编辑 `scripts/k8s/manifests/configmap.yaml` 中的 `AGENTS.md`，然后重新部署：

```bash
./scripts/k8s/deploy.sh
```

### Gateway 配置

编辑 `scripts/k8s/manifests/configmap.yaml` 中的 `openclaw.json`。完整参考请参阅 [Gateway 配置](/zh-CN/gateway/configuration)。

### 添加提供商

导出其他密钥后重新运行：

```bash
export ANTHROPIC_API_KEY="..."
export OPENAI_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

除非覆盖，否则现有提供商密钥会保留在 Secret 中。

也可以直接修补 Secret：

```bash
kubectl patch secret openclaw-secrets -n openclaw \
  -p '{"stringData":{"<PROVIDER>_API_KEY":"..."}}'
kubectl rollout restart deployment/openclaw -n openclaw
```

### 自定义命名空间

```bash
OPENCLAW_NAMESPACE=my-namespace ./scripts/k8s/deploy.sh
```

### 自定义镜像

编辑 `scripts/k8s/manifests/deployment.yaml` 中的 `image` 字段：

```yaml
image: ghcr.io/openclaw/openclaw:slim # 主要镜像；Docker Hub 官方镜像：openclaw/openclaw
```

### 公开到端口转发之外

默认清单将 Pod 内的 Gateway 网关绑定到回环地址。这适用于 `kubectl port-forward`，但不适用于需要直接访问 Pod IP 的 Kubernetes `Service` 或 Ingress 路径。

要通过 Ingress 或负载均衡器公开 Gateway 网关：

- 将 `scripts/k8s/manifests/configmap.yaml` 中的 Gateway 网关绑定从 `loopback` 更改为与你的部署模型匹配的非回环绑定。
- 保持启用 Gateway 网关身份验证，并使用正确终止 TLS 的入口点。
- 使用受支持的 Web 安全模型配置 Control UI 以供远程访问（例如 HTTPS/Tailscale Serve，并在需要时明确配置允许的来源）。

## 重新部署

```bash
./scripts/k8s/deploy.sh
```

这会应用所有清单并重启 Pod，以加载所有配置或 Secret 更改。

## 拆除

```bash
./scripts/k8s/deploy.sh --delete
```

这会删除命名空间及其中的所有资源，包括 PVC。

## 架构说明

- 默认情况下，Gateway 网关绑定到 Pod 内的回环地址，因此随附设置用于 `kubectl port-forward`。
- 没有集群范围的资源；所有内容都位于单个命名空间中。
- 安全强化：`readOnlyRootFilesystem`、`drop: ALL` capabilities、非 root 用户（UID 1000）。
- 默认配置让 Control UI 使用更安全的本地访问路径：回环绑定以及从 `kubectl port-forward` 到 `http://127.0.0.1:18789`。
- 如果不再局限于 localhost 访问，请使用受支持的远程模型：HTTPS/Tailscale，以及适当的 Gateway 网关绑定和 Control UI 来源设置。
- Secret 在临时目录中生成并直接应用到集群；不会向仓库检出目录写入任何 Secret 材料。

## 文件结构

```text
scripts/k8s/
├── deploy.sh                   # 创建命名空间和 Secret，通过 kustomize 部署
├── create-kind.sh              # 本地 Kind 集群（自动检测 docker/podman）
└── manifests/
    ├── kustomization.yaml      # Kustomize 基础配置
    ├── configmap.yaml          # openclaw.json + AGENTS.md
    ├── deployment.yaml         # 包含安全强化的 Pod 规范
    ├── pvc.yaml                # 10Gi 持久化存储
    └── service.yaml            # 端口 18789 上的 ClusterIP
```

## 相关内容

- [Docker](/zh-CN/install/docker)
- [Docker VM 运行时](/zh-CN/install/docker-vm-runtime)
- [安装概览](/zh-CN/install)
