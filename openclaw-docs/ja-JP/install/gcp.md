---
read_when:
    - GCP 上で OpenClaw を 24 時間 365 日稼働させたい場合
    - 自分の VM 上で本番環境向けの常時稼働 Gateway を運用したい場合
    - 永続化、バイナリ、再起動時の動作を完全に制御したい場合
summary: 永続的な状態を保持しながら、GCP Compute Engine VM（Docker）で OpenClaw Gateway を 24 時間 365 日稼働させる
title: GCP
x-i18n:
    generated_at: "2026-07-26T09:38:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6ca46b2ee78731162261cae6ea5a26b718be6035b998fa92e4ee5c9ea2e7ae07
    source_path: install/gcp.md
    workflow: 16
---

  永続的な OpenClaw Gateway を GCP Compute Engine VM 上で Docker を使用して実行します。永続化された状態、組み込み済みバイナリ、安全な再起動動作を備えます。

  料金はマシンタイプとリージョンによって異なります。ワークロードに適した最小の VM を選択し、OOM が発生した場合はスケールアップしてください。

  Gateway には、ノート PC から SSH ポートフォワーディング経由でアクセスできます。また、ファイアウォールとトークンを自身で管理する場合は、ポートを直接公開してアクセスすることもできます。

  このガイドでは、GCP Compute Engine 上の Debian を使用します。Ubuntu も利用できますが、パッケージは適宜読み替えてください。一般的な Docker の手順については、[Docker](/ja-JP/install/docker)を参照してください。

  ## 必要なもの

- GCP アカウント（`e2-micro` は無料枠の対象）
- `gcloud` CLI、または [Cloud Console](https://console.cloud.google.com)
- ノート PC からの SSH アクセス
- Docker と Docker Compose
- モデル認証情報
- 任意のプロバイダー認証情報（WhatsApp QR、Telegram ボットトークン、Gmail OAuth）
- 約20～30分

  ## クイック手順

1. GCP プロジェクトを作成し、課金と Compute Engine API を有効化
2. Compute Engine VM（`e2-small`、Debian 12、20GB）を作成
3. VM に SSH 接続し、Docker をインストール
4. OpenClaw リポジトリをクローン
5. 永続化するホストディレクトリを作成
6. `.env` と `docker-compose.yml` を設定
7. 必要なバイナリを組み込み、ビルドして起動

  <Steps>
  <Step title="gcloud CLI をインストール（または Console を使用）">
    [cloud.google.com/sdk/docs/install](https://cloud.google.com/sdk/docs/install)からインストールし、次を実行します。

    ```bash
    gcloud init
    gcloud auth login
    ```

    または、以下のすべての手順を [Cloud Console](https://console.cloud.google.com) の Web UI から実行します。

  </Step>

  <Step title="GCP プロジェクトを作成">
    ```bash
    gcloud projects create my-openclaw-project --name="OpenClaw Gateway"
    gcloud config set project my-openclaw-project
    gcloud services enable compute.googleapis.com
    ```

    [console.cloud.google.com/billing](https://console.cloud.google.com/billing) で課金を有効にします（Compute Engine に必要です）。

    Console での同等の操作: IAM & Admin > Create Project でプロジェクトと課金を有効にし、APIs & Services > Enable APIs > "Compute Engine API" > Enable の順に操作します。

  </Step>

  <Step title="VM を作成">
    | タイプ    | スペック                 | 料金               | 備考                                          |
    | --------- | ------------------------ | ------------------ | --------------------------------------------- |
    | e2-medium | 2 vCPU、4GB RAM          | 月額約$25          | ローカル Docker ビルドで最も安定              |
    | e2-small  | 2 vCPU、2GB RAM          | 月額約$12          | Docker ビルドに推奨される最小構成             |
    | e2-micro  | 2 vCPU（共有）、1GB RAM  | 無料枠の対象       | Docker ビルドで OOM（終了コード 137）が頻発  |

    ```bash
    gcloud compute instances create openclaw-gateway \
      --zone=us-central1-a \
      --machine-type=e2-small \
      --boot-disk-size=20GB \
      --image-family=debian-12 \
      --image-project=debian-cloud
    ```

  </Step>

  <Step title="VM に SSH 接続">
    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    Console: Compute Engine ダッシュボードで VM の横にある "SSH" をクリックします。

    VM 作成後、SSH 鍵の反映には1～2分かかる場合があります。接続が拒否された場合は、しばらく待ってから再試行してください。

  </Step>

  <Step title="Docker をインストール（VM 上）">
    ```bash
    sudo apt-get update
    sudo apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sudo sh
    sudo usermod -aG docker $USER
    ```

    グループの変更を反映するためにログアウトしてから、再度 SSH 接続します。

    ```bash
    exit
    ```

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a
    ```

    確認します。

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="OpenClaw リポジトリをクローン">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    このガイドでは、組み込んだバイナリが再起動後も維持されるように、カスタムイメージをビルドします。

  </Step>

  <Step title="永続化するホストディレクトリを作成">
    Docker コンテナは一時的なものです。長期間保持するすべての状態は、ホスト上に保存する必要があります。

    ```bash
    mkdir -p ~/.openclaw
    mkdir -p ~/.openclaw/workspace
    ```

  </Step>

  <Step title="環境変数を設定">
    リポジトリのルートに `.env` を作成します。

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/home/$USER/.openclaw
    OPENCLAW_WORKSPACE_DIR=/home/$USER/.openclaw/workspace

    GOG_KEYRING_PASSWORD=
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    `.env` を通じて安定した Gateway トークンを管理するには、`OPENCLAW_GATEWAY_TOKEN` を設定します。それ以外の場合は、再起動をまたいでクライアントを利用する前に `gateway.auth.token` を設定してください。どちらも設定されていない場合、OpenClaw はその起動時にのみ有効なランタイムトークンを使用します。`GOG_KEYRING_PASSWORD` 用のキーリングパスワードを生成します。

    ```bash
    openssl rand -hex 32
    ```

    **このファイルをコミットしないでください。** このファイルには、`OPENCLAW_GATEWAY_TOKEN` などのコンテナ／ランタイム環境変数が含まれます。保存されたプロバイダーの OAuth／API キー認証情報は、マウントされた `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` に格納されます。

  </Step>

  <Step title="Docker Compose の設定">
    `docker-compose.yml` を作成または更新します。

    ```yaml
    services:
      openclaw-gateway:
        image: ${OPENCLAW_IMAGE}
        build: .
        restart: unless-stopped
        env_file:
          - .env
        environment:
          - HOME=/home/node
          - NODE_ENV=production
          - TERM=xterm-256color
          - OPENCLAW_GATEWAY_BIND=${OPENCLAW_GATEWAY_BIND}
          - OPENCLAW_GATEWAY_PORT=${OPENCLAW_GATEWAY_PORT}
          - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
          - GOG_KEYRING_PASSWORD=${GOG_KEYRING_PASSWORD}
          - XDG_CONFIG_HOME=${XDG_CONFIG_HOME}
          - PATH=/home/linuxbrew/.linuxbrew/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        volumes:
          - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
          - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
        ports:
          # 推奨: VM 上の Gateway はループバックのみに制限し、SSH トンネル経由でアクセスします。
          # 公開するには、`127.0.0.1:` プレフィックスを削除し、それに応じてファイアウォールを設定します。
          - "127.0.0.1:${OPENCLAW_GATEWAY_PORT}:18789"
        command:
          [
            "node",
            "dist/index.js",
            "gateway",
            "--bind",
            "${OPENCLAW_GATEWAY_BIND}",
            "--port",
            "${OPENCLAW_GATEWAY_PORT}",
            "--allow-unconfigured",
          ]
    ```

    `--allow-unconfigured` はブートストラップを簡単にするためだけのものであり、実際の Gateway 設定の代わりにはなりません。デプロイ環境に合わせて、認証（`gateway.auth.token` またはパスワード）と安全なバインドモードも設定してください。

  </Step>

  <Step title="共有 Docker VM ランタイムの手順">
    一般的な Docker ホストのフローについては、共有ランタイムガイドに従ってください。

    - [必要なバイナリをイメージに組み込む](/ja-JP/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [ビルドして起動する](/ja-JP/install/docker-vm-runtime#build-and-launch)
    - [永続化される場所](/ja-JP/install/docker-vm-runtime#what-persists-where)
    - [更新](/ja-JP/install/docker-vm-runtime#updates)

  </Step>

  <Step title="GCP 固有の起動時の注意事項">
    `pnpm install --frozen-lockfile` 中に `Killed` または `exit code 137` が発生してビルドに失敗した場合、VM のメモリが不足しています。最低でも `e2-small`、初回ビルドをより確実にするには `e2-medium` を使用してください。

    LAN（`OPENCLAW_GATEWAY_BIND=lan`）にバインドする場合は、続行する前に信頼できるブラウザオリジンを設定します。

    ```bash
    docker compose run --rm openclaw-cli config set gateway.controlUi.allowedOrigins '["http://127.0.0.1:18789"]' --strict-json
    ```

    ポートを変更した場合は、`18789` を設定したポートに置き換えてください。

  </Step>

  <Step title="ノート PC からアクセスする">
    Gateway ポートを転送する SSH トンネルを作成します。

    ```bash
    gcloud compute ssh openclaw-gateway --zone=us-central1-a -- -L 18789:127.0.0.1:18789
    ```

    ブラウザで `http://127.0.0.1:18789/` を開きます。

    不要な情報を含まないダッシュボードリンクを再表示します。

    ```bash
    docker compose run --rm openclaw-cli dashboard --no-open
    ```

    UI で共有シークレット認証を求められた場合は、設定したトークンまたは
    パスワードを Control UI 設定に貼り付けます（この Docker フローでは
    デフォルトでトークンが書き込まれます。パスワード認証に切り替えた場合は、
    代わりに設定したパスワードを使用してください）。

    Control UI に `unauthorized` または `disconnected (1008): pairing required` が表示された場合は、ブラウザデバイスを承認します。

    ```bash
    docker compose run --rm openclaw-cli devices list
    docker compose run --rm openclaw-cli devices approve <requestId>
    ```

    共有の永続化マップについては [Docker VM ランタイム](/ja-JP/install/docker-vm-runtime#what-persists-where)、[更新フロー](/ja-JP/install/docker-vm-runtime#updates)を参照してください。

  </Step>
</Steps>

## トラブルシューティング

**SSH 接続が拒否される**

VM の作成後、SSH 鍵の反映には 1～2 分かかる場合があります。しばらく待ってから再試行してください。

**OS Login の問題**

OS Login プロファイルを確認します。

```bash
gcloud compute os-login describe-profile
```

アカウントに必要な IAM 権限（Compute OS Login または Compute OS Admin Login）が付与されていることを確認してください。

**メモリ不足（OOM）**

Docker のビルドが `Killed` と `exit code 137` で失敗した場合、VM は OOM によって強制終了されています。

```bash
# 最初に VM を停止する
gcloud compute instances stop openclaw-gateway --zone=us-central1-a

# マシンタイプを変更する
gcloud compute instances set-machine-type openclaw-gateway \
  --zone=us-central1-a \
  --machine-type=e2-small

# VM を起動する
gcloud compute instances start openclaw-gateway --zone=us-central1-a
```

## サービスアカウント（セキュリティのベストプラクティス）

個人利用では、デフォルトのユーザーアカウントで問題ありません。自動化または CI/CD では、最小限の権限を持つ専用のサービスアカウントを作成します。

```bash
gcloud iam service-accounts create openclaw-deploy \
  --display-name="OpenClaw Deployment"

gcloud projects add-iam-policy-binding my-openclaw-project \
  --member="serviceAccount:openclaw-deploy@my-openclaw-project.iam.gserviceaccount.com" \
  --role="roles/compute.instanceAdmin.v1"
```

自動化には Owner ロールを使用せず、機能する範囲で最も限定的なロールを使用してください。[ロールについて](https://cloud.google.com/iam/docs/understanding-roles)を参照してください。

## 次のステップ

- メッセージングチャネルを設定する: [チャネル](/ja-JP/channels)
- ローカルデバイスを Node としてペアリングする: [Node](/ja-JP/nodes)
- Gateway を設定する: [Gateway の設定](/ja-JP/gateway/configuration)

## 関連項目

- [インストールの概要](/ja-JP/install)
- [Azure](/ja-JP/install/azure)
- [VPS ホスティング](/ja-JP/vps)
