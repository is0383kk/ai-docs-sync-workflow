---
read_when:
    - Northflank への OpenClaw のデプロイ
    - ブラウザベースの Control UI を備えた、ワンクリックでのクラウドデプロイを希望する場合
summary: ワンクリックテンプレートを使用して OpenClaw を Northflank にデプロイする
title: Northflank
x-i18n:
    generated_at: "2026-07-26T09:06:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 16bb96fdf470999e15e163b6227d228ce8b60b9a172eb74cadc87bddd3955957
    source_path: install/northflank.mdx
    workflow: 16
---

Northflank にワンクリックテンプレートで OpenClaw をデプロイし、Web Control UI からアクセスします。これは「サーバー上でターミナルを使わない」最も簡単な方法です。Northflank が Gateway を実行します。

## はじめに

1. [Deploy OpenClaw](https://northflank.com/stacks/deploy-openclaw) をクリックしてテンプレートを開きます。
2. まだアカウントがない場合は、[Northflank でアカウントを作成](https://app.northflank.com/signup)します。
3. **Deploy OpenClaw now** をクリックします。
4. 必須の環境変数を設定します：`OPENCLAW_GATEWAY_TOKEN`（強力なランダム値を使用してください）。
5. **Deploy stack** をクリックして、OpenClaw テンプレートをビルドして実行します。
6. デプロイが完了するまで待ってから、**View resources** をクリックします。
7. OpenClaw サービスを開きます。
8. `/openclaw` にある公開 OpenClaw URL を開き、設定した共有シークレットを使用して接続します。このテンプレートではデフォルトで `OPENCLAW_GATEWAY_TOKEN` を使用します。パスワード認証に置き換えた場合は、代わりにそのパスワードを使用してください。

## 利用できるもの

- ホストされた OpenClaw Gateway + Control UI
- Northflank Volume（`/data`）による永続ストレージ。これにより、`openclaw.json`、エージェントごとの `auth-profiles.json`、チャンネル／プロバイダーの状態、セッション、ワークスペースが再デプロイ後も保持されます

## チャンネルを接続する

`/openclaw` の Control UI を使用するか、SSH 経由で `openclaw onboard` を実行して、チャンネルの設定手順を確認します。

- [Telegram](/ja-JP/channels/telegram)（最速、必要なのはボットトークンだけです）
- [Discord](/ja-JP/channels/discord)
- [すべてのチャンネル](/ja-JP/channels)

## 次のステップ

- メッセージングチャンネルを設定する：[チャンネル](/ja-JP/channels)
- Gateway を設定する：[Gateway の設定](/ja-JP/gateway/configuration)
- OpenClaw を最新の状態に保つ：[更新](/ja-JP/install/updating)
