---
read_when:
    - EasyRunner への OpenClaw のデプロイ
    - EasyRunner の Caddy プロキシの背後で Gateway を実行する
    - ホストされた Gateway の永続ボリュームと認証の選択
summary: Podman と Caddy を使用して EasyRunner 上で OpenClaw Gateway を実行する
title: EasyRunner
x-i18n:
    generated_at: "2026-07-26T10:08:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80cbde016a8bf7662d4b4a056a3d122a423264179daf70b5705e8f10b0dad5cb
    source_path: platforms/easyrunner.md
    workflow: 16
---

EasyRunner は、Caddy プロキシの背後で OpenClaw Gateway を小規模なコンテナ化アプリとしてホストします。このガイドでは、Podman 互換の Compose アプリを実行し、Caddy を介して HTTPS を終端する EasyRunner ホストを前提としています。

## 始める前に

- ドメインがルーティングされた EasyRunner サーバー。
- 公式の OpenClaw イメージ（`ghcr.io/openclaw/openclaw`）または独自のビルド。
- `/home/node/.openclaw` 用の永続設定ボリューム。
- `/home/node/.openclaw/workspace` 用の永続ワークスペースボリューム。
- 強力な Gateway トークンまたはパスワード。

可能な限りデバイス認証を有効にしておいてください。リバースプロキシがデバイス ID を正しく伝達できない場合は、まず信頼済みプロキシの設定を修正してください（[信頼済みプロキシ認証](/ja-JP/gateway/trusted-proxy-auth)を参照）。危険な認証バイパスは、完全にプライベートでオペレーターが管理するネットワークでのみ使用してください。

## Compose アプリ

次のような Compose ファイルを使用して EasyRunner アプリを作成します。

```yaml
services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:latest
    restart: unless-stopped
    environment:
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      OPENCLAW_HOME: /home/node
      OPENCLAW_STATE_DIR: /home/node/.openclaw
      OPENCLAW_CONFIG_PATH: /home/node/.openclaw/openclaw.json
      OPENCLAW_WORKSPACE_DIR: /home/node/.openclaw/workspace
    volumes:
      - openclaw-config:/home/node/.openclaw
      - openclaw-workspace:/home/node/.openclaw/workspace
    labels:
      caddy: openclaw.example.com
      caddy.reverse_proxy: "{{upstreams 1455}}"
    command: ["node", "openclaw.mjs", "gateway", "--bind", "lan", "--port", "1455"]

volumes:
  openclaw-config:
  openclaw-workspace:
```

`openclaw.example.com` を Gateway のホスト名に置き換えます。`OPENCLAW_GATEWAY_TOKEN` はアプリ定義にコミットせず、EasyRunner のシークレット／環境マネージャーに保存してください。イメージはデフォルトでループバックにバインドされるため、Caddy がコンテナに到達するには、`command` で `--bind lan --port 1455` を明示的に指定する必要があります。

## OpenClaw を設定する

永続設定ボリューム内では、Gateway にプロキシ経由でのみ到達できるようにし、認証を必須にします。

```json5
{
  gateway: {
    bind: "lan",
    port: 1455,
    auth: {
      token: "${OPENCLAW_GATEWAY_TOKEN}",
    },
  },
}
```

Caddy が Gateway の TLS を終端する場合は、認証チェックをグローバルに無効化するのではなく、正確なプロキシパスに対して信頼済みプロキシを設定してください。[信頼済みプロキシ認証](/ja-JP/gateway/trusted-proxy-auth)を参照してください。

## 確認

ワークステーションから次を実行します。

```bash
openclaw gateway probe --url https://openclaw.example.com --token <token>
openclaw gateway status --url https://openclaw.example.com --token <token>
```

EasyRunner ホストからは、`GET /healthz`（生存性）と `GET /readyz`（準備完了性）に認証なしでアクセスでき、イメージに組み込まれたコンテナのヘルスチェックに使用されます。また、アプリのログを確認し、Gateway がリッスンしていること、および起動時に SecretRef、Plugin、チャンネルの認証エラーが発生していないことを確認してください。

## 更新とバックアップ

- 新しい OpenClaw イメージをプルまたはビルドしてから、EasyRunner アプリを再デプロイします。
- 更新前に `openclaw-config` ボリュームをバックアップします。このボリュームには、
  `openclaw.json`、`agents/<agentId>/agent/auth-profiles.json`、およびインストール済みの
  Plugin パッケージの状態が保存されています。
- エージェントが永続的なプロジェクトデータを `openclaw-workspace` に書き込む場合は、これもバックアップします。
- メジャーアップデート後に `openclaw doctor` を実行し、設定の移行と
  サービスの警告を検出します。

## トラブルシューティング

- `gateway probe` が接続できない：Caddy のホスト名がアプリを指していること、およびコンテナが `0.0.0.0:1455` でリッスンしていることを確認します。
- 認証に失敗する：EasyRunner のシークレットとローカルクライアントのコマンドに設定されたトークンを同時にローテーションします。
- 復元後にファイルが root 所有になる：イメージは `node`（uid 1000）として実行されます。そのユーザーが `/home/node/.openclaw` と `/home/node/.openclaw/workspace` に書き込めるよう、マウントされたボリュームの所有権を修正します。
- ブラウザまたはチャンネルの Plugin が失敗する：必要な外部バイナリ、外向きネットワーク通信、およびマウントされた認証情報がコンテナ内で利用できるか確認します。
