---
read_when:
    - 你想在 Kubernetes 叢集上執行 OpenClaw
    - 你想在 Kubernetes 環境中測試 OpenClaw
summary: 使用 Kustomize 將 OpenClaw 閘道部署至 Kubernetes 叢集
title: Kubernetes
x-i18n:
    generated_at: "2026-07-26T07:45:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c05eb0eb923fa1f515aca1f6dcb6073aba69af0bdf30233243027edfedd45a39
    source_path: install/kubernetes.md
    workflow: 16
---

這是在 Kubernetes 上執行 OpenClaw 的最小起點，並非可直接用於正式環境的部署。內容涵蓋核心資源，並預期依你的環境進行調整。

## 為何不使用 Helm

OpenClaw 是包含一些設定檔的單一容器。值得自訂的部分是代理程式內容（Markdown 檔案、Skills、設定覆寫），而非基礎架構範本。Kustomize 可處理覆疊，而不會產生 Helm chart 的額外負擔。如果部署變得更複雜，可在這些資訊清單之上疊加 Helm chart。

## 需要準備的項目

- 執行中的 Kubernetes 叢集（AKS、EKS、GKE、k3s、kind、OpenShift 等）
- `kubectl` 已連線至你的叢集
- 至少一個模型供應商的 API 金鑰

## 快速開始

```bash
# 替換為你的供應商：ANTHROPIC、GEMINI、OPENAI 或 OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh

kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

`deploy.sh` 預設會建立權杖驗證。擷取產生的閘道權杖，以供 Control UI 使用：

```bash
kubectl get secret openclaw-secrets -n openclaw -o jsonpath='{.data.OPENCLAW_GATEWAY_TOKEN}' | base64 -d
```

進行本機偵錯時，`./scripts/k8s/deploy.sh --show-token` 會在部署後印出權杖。

## 使用 Kind 進行本機測試

如果你沒有叢集，可使用 [Kind](https://kind.sigs.k8s.io/) 在本機建立一個：

```bash
./scripts/k8s/create-kind.sh           # 自動偵測 docker 或 podman
./scripts/k8s/create-kind.sh --delete  # 拆除
```

接著照常使用 `./scripts/k8s/deploy.sh` 進行部署。

## 逐步操作

### 1) 部署

**選項 A：環境中的 API 金鑰（單一步驟）**

```bash
# 替換為你的供應商：ANTHROPIC、GEMINI、OPENAI 或 OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh
```

此指令碼會建立包含 API 金鑰與自動產生之閘道權杖的 Kubernetes Secret，然後進行部署。如果 Secret 已存在，則會保留目前的閘道權杖，以及任何未變更的供應商金鑰。

**選項 B：另外建立 Secret**

```bash
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

在任一命令中加入 `--show-token`，即可在本機測試時將權杖印至標準輸出。

### 2) 存取閘道

```bash
kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

## 部署的內容

```text
Namespace: openclaw（可透過 OPENCLAW_NAMESPACE 設定）
├── Deployment/openclaw        # 單一 Pod、初始化容器 + 閘道
├── Service/openclaw           # 連接埠 18789 上的 ClusterIP
├── PersistentVolumeClaim      # 用於代理程式狀態與設定的 10Gi
├── ConfigMap/openclaw-config  # openclaw.json + AGENTS.md
└── Secret/openclaw-secrets    # 閘道權杖 + API 金鑰
```

## 自訂

### 代理程式指示

編輯 `scripts/k8s/manifests/configmap.yaml` 中的 `AGENTS.md`，然後重新部署：

```bash
./scripts/k8s/deploy.sh
```

### 閘道設定

編輯 `scripts/k8s/manifests/configmap.yaml` 中的 `openclaw.json`。完整參考資料請參閱[閘道設定](/zh-TW/gateway/configuration)。

### 新增供應商

匯出其他金鑰後重新執行：

```bash
export ANTHROPIC_API_KEY="..."
export OPENAI_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

除非你覆寫現有的供應商金鑰，否則這些金鑰會保留在 Secret 中。

或者直接修補 Secret：

```bash
kubectl patch secret openclaw-secrets -n openclaw \
  -p '{"stringData":{"<PROVIDER>_API_KEY":"..."}}'
kubectl rollout restart deployment/openclaw -n openclaw
```

### 自訂命名空間

```bash
OPENCLAW_NAMESPACE=my-namespace ./scripts/k8s/deploy.sh
```

### 自訂映像檔

編輯 `scripts/k8s/manifests/deployment.yaml` 中的 `image` 欄位：

```yaml
image: ghcr.io/openclaw/openclaw:slim # 主要映像檔；官方 Docker Hub 鏡像：openclaw/openclaw
```

### 公開至 port-forward 以外的範圍

預設資訊清單會將 Pod 內的閘道繫結至迴路位址。這適用於 `kubectl port-forward`，但不適用於需要直接連線至 Pod IP 的 Kubernetes `Service` 或 Ingress 路徑。

若要透過 Ingress 或負載平衡器公開閘道：

- 將 `scripts/k8s/manifests/configmap.yaml` 中的閘道繫結從 `loopback` 變更為符合你部署模型的非迴路繫結。
- 保持啟用閘道驗證，並使用正確終止 TLS 的進入點。
- 使用支援的網頁安全性模型，設定 Control UI 以供遠端存取（例如 HTTPS/Tailscale Serve，並視需要明確設定允許的來源）。

## 重新部署

```bash
./scripts/k8s/deploy.sh
```

這會套用所有資訊清單並重新啟動 Pod，以載入任何設定或 Secret 變更。

## 拆除

```bash
./scripts/k8s/deploy.sh --delete
```

這會刪除命名空間及其中所有資源，包括 PVC。

## 架構附註

- 閘道預設會繫結至 Pod 內的迴路位址，因此隨附的設定適用於 `kubectl port-forward`。
- 沒有叢集範圍的資源；所有項目都位於單一命名空間內。
- 安全性強化：`readOnlyRootFilesystem`、`drop: ALL` capabilities、非 root 使用者（UID 1000）。
- 預設設定會讓 Control UI 使用較安全的本機存取路徑：迴路繫結加上將 `kubectl port-forward` 設為 `http://127.0.0.1:18789`。
- 如果要從 localhost 存取擴展至其他位置，請使用支援的遠端模型：HTTPS/Tailscale，加上適當的閘道繫結與 Control UI 來源設定。
- Secret 會在暫存目錄中產生並直接套用至叢集；不會將任何機密資料寫入存放庫簽出目錄。

## 檔案結構

```text
scripts/k8s/
├── deploy.sh                   # 建立命名空間 + Secret，透過 kustomize 部署
├── create-kind.sh              # 本機 Kind 叢集（自動偵測 docker/podman）
└── manifests/
    ├── kustomization.yaml      # Kustomize 基礎
    ├── configmap.yaml          # openclaw.json + AGENTS.md
    ├── deployment.yaml         # 包含安全性強化的 Pod 規格
    ├── pvc.yaml                # 10Gi 持久性儲存空間
    └── service.yaml            # 連接埠 18789 上的 ClusterIP
```

## 相關內容

- [Docker](/zh-TW/install/docker)
- [Docker VM 執行階段](/zh-TW/install/docker-vm-runtime)
- [安裝概覽](/zh-TW/install)
