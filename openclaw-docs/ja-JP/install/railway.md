---
read_when:
    - OpenClaw を Railway にデプロイする
    - ブラウザベースの Control UI を備えた、ワンクリックでのクラウドデプロイを希望する場合
summary: ワンクリックテンプレートを使用して OpenClaw を Railway にデプロイする
title: Railway
x-i18n:
    generated_at: "2026-07-26T09:46:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbef00b8de61545e9971b18164472c2f47fe607f69ec36f83a27a11b65ea863f
    source_path: install/railway.mdx
    workflow: 16
---

Railway でワンクリックテンプレートを使用して OpenClaw をデプロイし、Web Control UI からアクセスします。これは最も簡単な「サーバーでターミナルを使わない」方法です。Railway が Gateway を実行します。

## ワンクリックデプロイ

<a href="https://railway.com/deploy/clawdbot-railway-template" target="_blank" rel="noreferrer">
  Railway にデプロイ
</a>

<Steps>
  <Step title="テンプレートをデプロイ">
    上の **Deploy on Railway** をクリックします。
  </Step>

<Step title="ボリュームを追加">
  `/data` にマウントするボリュームを接続します（状態を永続化するために必須）。
</Step>

  <Step title="変数を設定">
    サービスに必要な **Variables** を設定します。

    - `OPENCLAW_GATEWAY_PORT=8080`（必須 -- Public Networking のポートと一致させる必要があります）
    - `OPENCLAW_GATEWAY_TOKEN`（必須。管理者用シークレットとして扱ってください）
    - `OPENCLAW_STATE_DIR=/data/.openclaw`（推奨）
    - `OPENCLAW_WORKSPACE_DIR=/data/workspace`（推奨）

  </Step>

<Step title="パブリックネットワークを有効化">
  **Public Networking** で、ポート `8080` のサービスに対して **HTTP Proxy** を有効にします。
</Step>

  <Step title="接続">
    **Railway -> your service -> Settings -> Domains** で公開 URL を確認します。生成されたドメイン（多くの場合 `https://<something>.up.railway.app`）または接続したカスタムドメインです。

    `https://<your-railway-domain>/openclaw` を開き、設定した共有シークレットを使用して接続します。テンプレートはデフォルトで `OPENCLAW_GATEWAY_TOKEN` を使用します。パスワード認証に置き換えた場合は、代わりにそのパスワードを使用します。

  </Step>
</Steps>

## 提供されるもの

- ホストされた OpenClaw Gateway + Control UI
- Railway Volume（`/data`）による永続ストレージ。これにより、`openclaw.json`、エージェントごとの `auth-profiles.json`、チャンネルおよびプロバイダーの状態、セッション、ワークスペースが再デプロイ後も保持されます

## チャンネルを接続

`/openclaw` の Control UI を使用するか、Railway のシェルから `openclaw onboard` を実行して、チャンネルの設定手順を確認します。

- [Discord](/ja-JP/channels/discord)
- [Telegram](/ja-JP/channels/telegram)（最速 -- ボットトークンだけで設定できます）
- [すべてのチャンネル](/ja-JP/channels)

## バックアップと移行

状態、設定、認証プロファイル、ワークスペースをエクスポートします。

```bash
openclaw backup create
```

これにより、OpenClaw の状態と設定済みのワークスペースを含む、移植可能なバックアップアーカイブが作成されます。詳細については、[バックアップ](/ja-JP/cli/backup)を参照してください。

## 次のステップ

- メッセージングチャンネルを設定する：[チャンネル](/ja-JP/channels)
- Gateway を設定する：[Gateway の設定](/ja-JP/gateway/configuration)
- OpenClaw を最新の状態に保つ：[更新](/ja-JP/install/updating)
