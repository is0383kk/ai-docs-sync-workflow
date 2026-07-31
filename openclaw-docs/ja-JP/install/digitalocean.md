---
read_when:
    - DigitalOcean で OpenClaw をセットアップする
    - OpenClaw向けのシンプルな有料VPSを探す
summary: DigitalOcean Droplet で OpenClaw をホストする
title: DigitalOcean
x-i18n:
    generated_at: "2026-07-26T10:06:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e124a59c079efda0c8e880018f2657fad784af1489ca3f98ed8ab609249e35bd
    source_path: install/digitalocean.md
    workflow: 16
---

DigitalOcean Droplet 上で永続的な OpenClaw Gateway を実行します（1 GB Basic プランで月額約 $6）。

DigitalOcean は、シンプルに利用できる有料 VPS です。より安価な、または無料の選択肢については以下を参照してください。

- [Hetzner](/ja-JP/install/hetzner) -- 1 ドルあたりのコア数と RAM 容量が多い。
- [Oracle Cloud](/ja-JP/install/oracle) -- Always Free ARM ティア（最大 4 OCPU、24 GB RAM）がありますが、登録がうまくいかない場合があり、ARM のみです。

## 前提条件

- DigitalOcean アカウント（[登録](https://cloud.digitalocean.com/registrations/new)）
- SSH キーペア（またはパスワード認証を使用する意思）
- 約 20 分

## セットアップ

<Steps>
  <Step title="Droplet を作成する">
    <Warning>
    クリーンなベースイメージ（Ubuntu 24.04 LTS）を使用してください。起動スクリプトとファイアウォールのデフォルト設定を確認していない限り、サードパーティ製 Marketplace の 1-click イメージは避けてください。
    </Warning>

    1. [DigitalOcean](https://cloud.digitalocean.com/) にログインします。
    2. **Create > Droplets** をクリックします。
    3. 以下を選択します。
       - **Region:** 最も近い場所
       - **Image:** Ubuntu 24.04 LTS
       - **Size:** Basic、Regular、1 vCPU / 1 GB RAM / 25 GB SSD
       - **Authentication:** SSH key（推奨）または password
    4. **Create Droplet** をクリックし、IP アドレスを控えます。

  </Step>

  <Step title="接続してインストールする">
    ```bash
    ssh root@YOUR_DROPLET_IP

    apt update && apt upgrade -y

    # Node.js 24 をインストール
    curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
    apt install -y nodejs

    # OpenClaw をインストール
    curl -fsSL https://openclaw.ai/install.sh | bash

    # OpenClaw の状態とサービスを所有する非 root ユーザーを作成します。
    adduser openclaw
    usermod -aG sudo openclaw
    loginctl enable-linger openclaw

    su - openclaw
    openclaw --version
    ```

    root シェルはシステムの初期セットアップにのみ使用してください。OpenClaw コマンドは非 root の `openclaw` ユーザーとして実行し、状態が `/home/openclaw/.openclaw/` 配下に保存され、Gateway がそのユーザーの systemd `--user` サービスとしてインストールされるようにします。

  </Step>

  <Step title="オンボーディングを実行する">
    ```bash
    openclaw onboard --install-daemon
    ```

    ウィザードの案内に従って、モデル認証、チャンネルのセットアップ、Gateway トークンの生成、デーモンのインストール（systemd ユーザーサービス）を行います。

  </Step>

  <Step title="スワップを追加する（1 GB Droplet では推奨）">
    ```bash
    fallocate -l 2G /swapfile
    chmod 600 /swapfile
    mkswap /swapfile
    swapon /swapfile
    echo '/swapfile none swap sw 0 0' >> /etc/fstab
    ```
  </Step>

  <Step title="Gateway を確認する">
    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```
  </Step>

  <Step title="Control UI にアクセスする">
    Gateway はデフォルトでループバックにバインドされます。以下のいずれかの方法を選択してください。

    **オプション A：SSH トンネル（最も簡単）**

    ```bash
    # ローカルマシンから
    ssh -L 18789:localhost:18789 root@YOUR_DROPLET_IP
    ```

    次に、`http://localhost:18789` を開きます。

    **オプション B：Tailscale Serve**

    ```bash
    curl -fsSL https://tailscale.com/install.sh | sudo sh
    sudo tailscale up
    openclaw config set gateway.tailscale.mode serve
    openclaw gateway restart
    ```

    次に、tailnet 上の任意のデバイスから `https://<magicdns>/` を開きます。

    Tailscale Serve は tailnet の ID ヘッダーを使用して Control UI と WebSocket のトラフィックを認証します。これは、Gateway ホスト自体が信頼されていることを前提とします。それにかかわらず、HTTP API エンドポイントは引き続き Gateway の通常の認証モード（トークン／パスワード）に従います。Serve 経由で明示的な共有シークレット認証情報を必須にするには、`gateway.auth.allowTailscale: false` を設定し、`gateway.auth.mode: "token"` または `"password"` を使用します。

    **オプション C：tailnet バインド（Serve なし）**

    ```bash
    openclaw config set gateway.bind tailnet
    openclaw gateway restart
    ```

    次に、`http://<tailscale-ip>:18789` を開きます（トークンが必要です）。

  </Step>
</Steps>

## 永続化とバックアップ

OpenClaw の状態は以下に保存されます。

- `~/.openclaw/` -- `openclaw.json`、チャンネル／プロバイダーの認証情報、エージェントごとの `auth-profiles.json`、セッションデータ。
- `~/.openclaw/workspace/` -- エージェントのワークスペース（SOUL.md、メモリ、成果物）。

これらは Droplet を再起動しても保持されます。移植可能なスナップショットを作成するには、以下を実行します。

```bash
openclaw backup create
```

DigitalOcean のスナップショットは Droplet 全体をバックアップします。`openclaw backup create` はホスト間で移植可能です。

## 1 GB RAM のヒント

$6 の Droplet には 1 GB の RAM しかありません。快適に動作させるには、以下を実施してください。

- 再起動後も保持されるように、上記のスワップ設定が `/etc/fstab` に含まれていることを確認します。
- ローカルモデルよりも API ベースのモデル（Claude、GPT）を優先してください。ローカル LLM の推論は 1 GB では実行できません。
- 大きなプロンプトで OOM が発生する場合は、`agents.defaults.model.primary` をより小さいモデルに設定します。
- `free -h` と `htop` で監視します。

## トラブルシューティング

**Gateway が起動しない** -- `openclaw doctor --non-interactive` を実行し、`journalctl --user -u openclaw-gateway.service -n 50` でログを確認します。

**ポートがすでに使用されている** -- `lsof -i :18789` を実行してプロセスを特定し、停止します。

**メモリ不足** -- `free -h` でスワップが有効であることを確認します。それでも OOM が発生する場合は、ローカルモデルではなく API ベースのモデル（Claude、GPT）に切り替えるか、2 GB の Droplet にアップグレードします。

## 次のステップ

- [チャンネル](/ja-JP/channels) -- Telegram、WhatsApp、Discord などに接続
- [Gateway の設定](/ja-JP/gateway/configuration) -- すべての設定オプション
- [アップデート](/ja-JP/install/updating) -- OpenClaw を最新の状態に維持

## 関連項目

- [インストールの概要](/ja-JP/install)
- [Fly.io](/ja-JP/install/fly)
- [Hetzner](/ja-JP/install/hetzner)
- [VPS ホスティング](/ja-JP/vps)
