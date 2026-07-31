---
read_when:
    - Kubernetes クラスター上で OpenClaw を実行する場合
    - Kubernetes 環境で OpenClaw をテストしたい場合
summary: Kustomize を使用して OpenClaw Gateway を Kubernetes クラスターにデプロイする
title: Kubernetes
x-i18n:
    generated_at: "2026-07-26T09:06:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c05eb0eb923fa1f515aca1f6dcb6073aba69af0bdf30233243027edfedd45a39
    source_path: install/kubernetes.md
    workflow: 16
---

OpenClaw を Kubernetes で実行するための最小限の出発点であり、本番環境向けのデプロイではありません。コアリソースを扱い、環境に合わせて調整することを前提としています。

## Helm を使用しない理由

OpenClaw は、いくつかの設定ファイルを備えた単一のコンテナです。重要なカスタマイズ対象は、インフラストラクチャのテンプレート化ではなく、エージェントのコンテンツ（Markdown ファイル、Skills、設定の上書き）です。Kustomize なら、Helm チャートのオーバーヘッドなしでオーバーレイを処理できます。デプロイがより複雑になった場合は、これらのマニフェストの上に Helm チャートを重ねてください。

## 必要なもの

- 稼働中の Kubernetes クラスター（AKS、EKS、GKE、k3s、kind、OpenShift など）
- `kubectl` がクラスターに接続されていること
- 少なくとも 1 つのモデルプロバイダーの API キー

## クイックスタート

```bash
# 使用するプロバイダーに置き換えます: ANTHROPIC、GEMINI、OPENAI、または OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh

kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

`deploy.sh` はデフォルトでトークン認証を作成します。Control UI 用に生成された Gateway トークンを取得します。

```bash
kubectl get secret openclaw-secrets -n openclaw -o jsonpath='{.data.OPENCLAW_GATEWAY_TOKEN}' | base64 -d
```

ローカルデバッグでは、`./scripts/k8s/deploy.sh --show-token` を指定するとデプロイ後にトークンが表示されます。

## Kind を使用したローカルテスト

クラスターがない場合は、[Kind](https://kind.sigs.k8s.io/) を使用してローカルに作成します。

```bash
./scripts/k8s/create-kind.sh           # docker または podman を自動検出
./scripts/k8s/create-kind.sh --delete  # 削除
```

その後、通常どおり `./scripts/k8s/deploy.sh` を使用してデプロイします。

## 手順

### 1) デプロイ

**オプション A: 環境変数に API キーを設定（1 ステップ）**

```bash
# 使用するプロバイダーに置き換えます: ANTHROPIC、GEMINI、OPENAI、または OPENROUTER
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh
```

スクリプトは、API キーと自動生成された Gateway トークンを含む Kubernetes Secret を作成してからデプロイします。Secret がすでに存在する場合、現在の Gateway トークンと、変更対象でないプロバイダーキーは保持されます。

**オプション B: Secret を個別に作成**

```bash
export <PROVIDER>_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

ローカルテスト用にトークンを標準出力へ表示するには、いずれかのコマンドに `--show-token` を追加します。

### 2) Gateway にアクセス

```bash
kubectl port-forward svc/openclaw 18789:18789 -n openclaw
open http://localhost:18789
```

## デプロイされるもの

```text
Namespace: openclaw（OPENCLAW_NAMESPACE で設定可能）
├── Deployment/openclaw        # 単一の Pod、init コンテナ + Gateway
├── Service/openclaw           # ポート 18789 の ClusterIP
├── PersistentVolumeClaim      # エージェントの状態と設定用の 10Gi
├── ConfigMap/openclaw-config  # openclaw.json + AGENTS.md
└── Secret/openclaw-secrets    # Gateway トークン + API キー
```

## カスタマイズ

### エージェントの指示

`scripts/k8s/manifests/configmap.yaml` の `AGENTS.md` を編集して再デプロイします。

```bash
./scripts/k8s/deploy.sh
```

### Gateway の設定

`scripts/k8s/manifests/configmap.yaml` の `openclaw.json` を編集します。詳細なリファレンスについては、[Gateway の設定](/ja-JP/gateway/configuration)を参照してください。

### プロバイダーを追加

追加のキーをエクスポートして再実行します。

```bash
export ANTHROPIC_API_KEY="..."
export OPENAI_API_KEY="..."
./scripts/k8s/deploy.sh --create-secret
./scripts/k8s/deploy.sh
```

既存のプロバイダーキーは、上書きしない限り Secret に保持されます。

または、Secret に直接パッチを適用します。

```bash
kubectl patch secret openclaw-secrets -n openclaw \
  -p '{"stringData":{"<PROVIDER>_API_KEY":"..."}}'
kubectl rollout restart deployment/openclaw -n openclaw
```

### カスタム名前空間

```bash
OPENCLAW_NAMESPACE=my-namespace ./scripts/k8s/deploy.sh
```

### カスタムイメージ

`scripts/k8s/manifests/deployment.yaml` の `image` フィールドを編集します。

```yaml
image: ghcr.io/openclaw/openclaw:slim # プライマリ。公式 Docker Hub ミラー: openclaw/openclaw
```

### ポートフォワード以外で公開

デフォルトのマニフェストでは、Pod 内の Gateway がループバックにバインドされます。これは `kubectl port-forward` では機能しますが、Pod IP に直接到達する必要がある Kubernetes `Service` や Ingress パスでは機能しません。

Ingress またはロードバランサー経由で Gateway を公開するには、次のようにします。

- `scripts/k8s/manifests/configmap.yaml` の Gateway バインドを `loopback` から、デプロイモデルに適した非ループバックバインドへ変更します。
- Gateway 認証を有効なままにし、TLS が適切に終端されるエントリポイントを使用します。
- サポートされている Web セキュリティモデルを使用して、リモートアクセス用に Control UI を設定します（たとえば HTTPS/Tailscale Serve、および必要に応じて明示的に許可するオリジン）。

## 再デプロイ

```bash
./scripts/k8s/deploy.sh
```

これにより、すべてのマニフェストが適用され、設定や Secret の変更を反映するために Pod が再起動されます。

## 削除

```bash
./scripts/k8s/deploy.sh --delete
```

これにより、名前空間と、その中にある PVC を含むすべてのリソースが削除されます。

## アーキテクチャに関する注意事項

- デフォルトでは、Gateway は Pod 内のループバックにバインドされるため、同梱のセットアップは `kubectl port-forward` 用です。
- クラスター全体を対象とするリソースはなく、すべて単一の名前空間内に配置されます。
- セキュリティ強化: `readOnlyRootFilesystem`、`drop: ALL` ケイパビリティ、非 root ユーザー（UID 1000）。
- デフォルト設定では、Control UI はより安全なローカルアクセス経路に維持されます。ループバックバインドに加え、`kubectl port-forward` を `http://127.0.0.1:18789` に設定します。
- localhost 以外からアクセスする場合は、サポートされているリモートモデルを使用します。HTTPS/Tailscale に加え、適切な Gateway バインドと Control UI のオリジン設定を使用してください。
- Secret は一時ディレクトリ内で生成され、クラスターへ直接適用されます。Secret の内容がリポジトリのチェックアウトに書き込まれることはありません。

## ファイル構成

```text
scripts/k8s/
├── deploy.sh                   # 名前空間と Secret を作成し、kustomize でデプロイ
├── create-kind.sh              # ローカル Kind クラスター（docker/podman を自動検出）
└── manifests/
    ├── kustomization.yaml      # Kustomize ベース
    ├── configmap.yaml          # openclaw.json + AGENTS.md
    ├── deployment.yaml         # セキュリティ強化を含む Pod 仕様
    ├── pvc.yaml                # 10Gi の永続ストレージ
    └── service.yaml            # 18789 の ClusterIP
```

## 関連項目

- [Docker](/ja-JP/install/docker)
- [Docker VM ランタイム](/ja-JP/install/docker-vm-runtime)
- [インストールの概要](/ja-JP/install)
