---
read_when:
    - Gateway プロセスの実行またはデバッグ
summary: Gateway サービス、ライフサイクル、運用のためのランブック
title: Gateway 運用手順書
x-i18n:
    generated_at: "2026-07-26T10:01:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d8b50b6041905c321887ea0f579f8d4c3b74552b2b72c37ec655e43a53dfc130
    source_path: gateway/index.md
    workflow: 16
---

このページは、Gateway サービスの初日起動と2日目以降の運用に使用します。

<CardGroup cols={2}>
  <Card title="詳細なトラブルシューティング" icon="siren" href="/ja-JP/gateway/troubleshooting">
    症状を起点とした診断を、正確なコマンド手順とログシグネチャとともに説明します。
  </Card>
  <Card title="設定" icon="sliders" href="/ja-JP/gateway/configuration">
    タスク指向のセットアップガイドと完全な設定リファレンスです。
  </Card>
  <Card title="シークレット管理" icon="key-round" href="/ja-JP/gateway/secrets">
    SecretRef コントラクト、ランタイムスナップショットの動作、移行／再読み込み操作について説明します。
  </Card>
  <Card title="シークレットプランのコントラクト" icon="shield-check" href="/ja-JP/gateway/secrets-plan-contract">
    `secrets apply` の正確なターゲット／パス規則と、参照のみの認証プロファイルの動作について説明します。
  </Card>
</CardGroup>

## 5分で行うローカル起動

<Steps>
  <Step title="Gateway を起動する">

```bash
openclaw gateway --port 18789
# デバッグ／トレースを標準入出力にも出力
openclaw gateway --port 18789 --verbose
# 選択したポートのリスナーを強制終了してから起動
openclaw gateway --force
```

  </Step>

  <Step title="サービスの正常性を確認する">

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

正常時の基準は、`Runtime: running`、`Connectivity probe: ok`、および想定どおりの `Capability` 行です。単なる到達可能性ではなく、読み取りスコープの RPC を検証するには `openclaw gateway status --require-rpc` を使用します。

  </Step>

  <Step title="チャネルの準備状況を検証する">

```bash
openclaw channels status --probe
```

Gateway に到達できる場合、アカウントごとにチャネルのライブプローブと任意の監査を実行します。Gateway に到達できない場合、CLI は設定のみに基づくチャネル概要へフォールバックします。

  </Step>
</Steps>

<Note>
Gateway の設定再読み込みは、アクティブな設定ファイルのパス（プロファイル／状態のデフォルトから解決されるパス、または設定されている場合は `OPENCLAW_CONFIG_PATH`）を監視します。デフォルトモードは `gateway.reload.mode="hybrid"` です。最初の読み込みに成功すると、実行中のプロセスはアクティブなメモリ内設定スナップショットを使用してサービスを提供し、再読み込みが成功するとそのスナップショットをアトミックに置き換えます。
</Note>

## ランタイムモデル

- ルーティング、コントロールプレーン、チャネル接続を担う常時稼働プロセスが1つ。
- 次の用途に単一の多重化ポートを使用：
  - WebSocket 制御／RPC
  - HTTP API（`/v1/models`、`/v1/embeddings`、`/v1/chat/completions`、`/v1/responses`、`/tools/invoke`）
  - 任意の `/api/v1/admin/rpc` などの Plugin HTTP ルート
  - コントロール UI とフック
- デフォルトのバインドモード：`loopback`。コンテナ環境が検出された場合、実効デフォルトは `auto`（ポートフォワーディング用に `0.0.0.0` へ解決）です。ただし、Tailscale serve/funnel が有効な場合は常に `loopback` が強制されます。
- デフォルトでは認証が必須です。共有シークレット構成では `gateway.auth.token`／`gateway.auth.password`（または `OPENCLAW_GATEWAY_TOKEN`／`OPENCLAW_GATEWAY_PASSWORD`）を使用し、非ループバックのリバースプロキシ構成では `gateway.auth.mode: "trusted-proxy"` を使用できます。

## OpenAI 互換エンドポイント

OpenClaw で最も効果の高い互換性サーフェス：

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/responses`

この一式が重要な理由：

- Open WebUI、LobeChat、LibreChat の統合の多くは、最初に `/v1/models` をプローブします。
- 多くの RAG およびメモリパイプラインは `/v1/embeddings` を前提としています。
- エージェントネイティブなクライアントでは、`/v1/responses` が選ばれることが増えています。

`/v1/models` はエージェント優先で、設定された各エージェントについて `openclaw`、`openclaw/default`、`openclaw/<agentId>` を返します。`openclaw/default` は、設定されたデフォルトエージェントへ常にマッピングされる安定したエイリアスです。バックエンドのプロバイダー／モデルを上書きする場合は `x-openclaw-model` を送信します。送信しない場合は、選択したエージェントの通常のモデルおよび埋め込み設定が引き続き制御します。

これらはすべてメインの Gateway ポートで動作し、Gateway HTTP API の他の部分と同じ、信頼されたオペレーター認証境界を使用します。

管理用 HTTP RPC（`POST /api/v1/admin/rpc`）は、WebSocket RPC を使用できないホストツール向けの、独立したデフォルト無効の Plugin ルートです。[管理用 HTTP RPC](/ja-JP/plugins/admin-http-rpc)を参照してください。

### ポートとバインドの優先順位

| 設定         | 解決順序                                                             |
| ------------ | -------------------------------------------------------------------- |
| Gateway ポート | `--port` → `OPENCLAW_GATEWAY_PORT` → `gateway.port` → `18789`        |
| バインドモード | CLI／オーバーライド → `gateway.bind` → `loopback`（コンテナでは `auto`） |

インストールされた Gateway サービスは、解決済みの `--port` をスーパーバイザーメタデータに記録します。`gateway.port` を変更した後は、launchd／systemd／schtasks が新しいポートでプロセスを起動するように、`openclaw doctor --fix` または `openclaw gateway install --force` を実行します。

Gateway の起動時には、非ループバックバインド用のローカルコントロール UI オリジンを初期設定するときにも、同じ実効ポートとバインドが使用されます。たとえば、`--bind lan --port 3000` はランタイム検証の実行前に `http://localhost:3000` と `http://127.0.0.1:3000` を初期設定します。HTTPS プロキシ URL などのリモートブラウザーのオリジンは、`gateway.controlUi.allowedOrigins` に明示的に追加してください。

### ホットリロードモード

| `gateway.reload.mode` | 動作                                       |
| --------------------- | ------------------------------------------ |
| `off`                 | 設定を再読み込みしない                     |
| `hot`                 | ホット適用が安全な変更のみ適用             |
| `restart`             | 再読み込みに再起動が必要な変更で再起動     |
| `hybrid`（デフォルト） | 安全な場合はホット適用し、必要な場合は再起動 |

## オペレーター向けコマンドセット

```bash
openclaw gateway status
openclaw gateway status --deep   # システムレベルのサービススキャンを追加
openclaw gateway status --json
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw doctor
```

`gateway status --deep` は追加のサービス検出（LaunchDaemons／systemd システムユニット／schtasks）用であり、より詳細な RPC 正常性プローブではありません。

## 複数の Gateway（同一ホスト）

ほとんどのインストールでは、マシンごとに1つの Gateway を実行します。1つの Gateway で複数のエージェントとチャネルをホストできます。複数の Gateway が必要なのは、意図的に分離する場合やレスキューボットを用意する場合のみです。

便利な確認コマンド：

```bash
openclaw gateway status --deep
openclaw gateway probe
```

想定される動作：

- `gateway status --deep` は、古い launchd／systemd／schtasks のインストールが残っている場合に `Other gateway-like services detected (best effort)` を報告し、クリーンアップのヒントを表示することがあります。
- `gateway probe` は、別々の Gateway が応答した場合や、到達可能なターゲットが同じ Gateway であることを OpenClaw が証明できない場合に、`multiple reachable gateway identities` について警告することがあります。同じ Gateway への SSH トンネル、プロキシ URL、または設定済みのリモート URL は、転送ポートが異なっていても、複数のトランスポートを持つ1つの Gateway です。
- 意図的に行う場合は、Gateway ごとにポート、設定／状態、ワークスペースルートを分離します。

インスタンスごとのチェックリスト：

- 一意の `gateway.port`
- 一意の `OPENCLAW_CONFIG_PATH`
- 一意の `OPENCLAW_STATE_DIR`
- 一意の `agents.defaults.workspace`

例：

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

詳細なセットアップ：[/gateway/multiple-gateways](/ja-JP/gateway/multiple-gateways)。

## リモートアクセス

推奨：Tailscale／VPN。
代替手段：SSH トンネル。

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

その後、クライアントをローカルから `ws://127.0.0.1:18789` に接続します。

<Warning>
SSH トンネルは Gateway 認証を迂回しません。共有シークレット認証では、トンネル経由であってもクライアントは
`token`／`password` を送信する必要があります。アイデンティティを伴うモードでは、
リクエストが引き続きその認証経路を満たす必要があります。
</Warning>

参照：[リモート Gateway](/ja-JP/gateway/remote)、[認証](/ja-JP/gateway/authentication)、[Tailscale](/ja-JP/gateway/tailscale)。

## 監視とサービスのライフサイクル

本番環境相当の信頼性を確保するには、スーパーバイザー管理下で実行します。

<Tabs>
  <Tab title="macOS (launchd)">

```bash
openclaw gateway install
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
```

再起動には `openclaw gateway restart` を使用します。再起動の代わりに `openclaw gateway stop` と `openclaw gateway start` を連結して実行しないでください。

macOS では、`gateway stop` はデフォルトで `launchctl bootout` を使用します。これにより、無効化を永続化せずに現在のブートセッションから LaunchAgent が削除されるため、予期しないクラッシュ後も KeepAlive による自動復旧が機能し、`gateway start` で正常に再有効化できます。再起動後も自動再生成を永続的に抑止するには、`--disable` を渡します：`openclaw gateway stop --disable`。

LaunchAgent のラベルは `ai.openclaw.gateway`（デフォルト）または `ai.openclaw.<profile>`（名前付きプロファイル）です。`openclaw doctor` はサービス設定のドリフトを監査して修復します。

  </Tab>

  <Tab title="Linux (systemd user)">

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway[-<profile>].service
openclaw gateway status
```

ログアウト後も永続化するには、linger を有効にします：

```bash
sudo loginctl enable-linger $(whoami)
```

デスクトップセッションのないヘッドレスサーバーでは、`systemctl --user` コマンドを再試行する前に、`XDG_RUNTIME_DIR` が設定されていること（`export XDG_RUNTIME_DIR=/run/user/$(id -u)`）も確認してください。

カスタムインストールパスが必要な場合の手動ユーザーユニットの例：

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target
StartLimitBurst=5
StartLimitIntervalSec=60

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
RestartPreventExitStatus=78
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
OOMPolicy=continue
KillMode=control-group

[Install]
WantedBy=default.target
```

  </Tab>

  <Tab title="Windows (native)">

```powershell
openclaw gateway install
openclaw gateway status --json
openclaw gateway restart
openclaw gateway stop
```

Windows ネイティブの管理対象起動では、`OpenClaw Gateway`
（名前付きプロファイルの場合は `OpenClaw Gateway (<profile>)`）という名前の Scheduled Task を使用します。Scheduled Task
の作成が拒否された場合、OpenClaw は状態ディレクトリ内の `gateway.cmd` を参照する
ユーザー単位の Startup フォルダーランチャーへフォールバックします。

  </Tab>

  <Tab title="Linux (system service)">

マルチユーザー／常時稼働ホストでは、システムユニットを使用します。

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway[-<profile>].service
```

ユーザーユニットと同じサービス本体を使用しますが、
`/etc/systemd/system/openclaw-gateway[-<profile>].service` の下にインストールし、`openclaw` バイナリが別の場所にある場合は
`ExecStart=` を調整します。

同じプロファイル／ポートに対して、`openclaw doctor --fix` がユーザーレベルの Gateway サービスもインストールしないようにしてください。システムレベルの OpenClaw Gateway サービスが見つかると、Doctor はその自動インストールを拒否します。システムユニットがライフサイクルを管理する場合は、`OPENCLAW_SERVICE_REPAIR_POLICY=external` を使用してください。

  </Tab>
</Tabs>

無効な設定エラーは終了コード `78` で終了します。Linux の systemd ユニットは、設定が修正されるまで再起動を停止するために `RestartPreventExitStatus=78` を使用します。launchd と Windows Task Scheduler には終了コードごとの同等の停止ルールがないため、Gateway は短時間に繰り返された異常起動の履歴も永続化し、起動失敗が繰り返された場合はチャネル／プロバイダーアカウントの自動起動を抑止します。このセーフモードでもコントロールプレーンは検査と修復のために起動しますが、設定のホットリロードと `secrets.reload` はチャネルの自動再起動を拒否します。オペレーターが明示的に `channels.start` を要求すると、この抑止を上書きできます。

## 開発プロファイルのクイックパス

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

デフォルトには、分離された状態／設定と基本 Gateway ポート `19001` が含まれます。

## プロトコルのクイックリファレンス（オペレーター向け）

- 最初のクライアントフレームは `connect` でなければなりません。
- Gateway は、`snapshot`（`presence`、`health`、`stateVersion`、`uptimeMs`）と `policy` の制限（`maxPayload`、`maxBufferedBytes`、`tickIntervalMs`）を含む `hello-ok` フレームを返します。
- `hello-ok.features.methods` / `events` は控えめな検出用リストであり、
  呼び出し可能なすべてのヘルパールートを生成したダンプではありません。
- リクエスト：`req(method, params)` → `res(ok/payload|error)`。
- 一般的なイベントには、`connect.challenge`、`agent`、`chat`、
  `session.message`、`session.operation`、`session.tool`、オプトインの
  `session.approval`、`sessions.changed`、`presence`、`tick`、`health`、
  `heartbeat`、ペアリング／承認ライフサイクルイベント、および `shutdown` があります。

エージェントの実行は2段階です：

1. 即時の受理確認（`status:"accepted"`）
2. 最終完了レスポンス（`status:"ok"|"error"`）。その間に `agent` イベントがストリーミングされます。

完全なプロトコルドキュメントについては、[Gateway プロトコル](/ja-JP/gateway/protocol)を参照してください。

## 運用チェック

### 稼働状況

- WS を開き、`connect` を送信します。
- スナップショットを含む `hello-ok` レスポンスが返されることを確認します。

### 準備状況

```bash
openclaw gateway status
openclaw channels status --probe
openclaw health
```

### ギャップからの復旧

イベントは再生されません。シーケンスにギャップがある場合は、続行する前に状態（`health`、`system-presence`）を更新してください。

## 一般的な障害の兆候

| 兆候                                                           | 想定される問題                                                                  |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `refusing to bind gateway ... without auth`                    | 有効な Gateway 認証パスなしで非ループバックにバインドしている                           |
| `another gateway instance is already listening` / `EADDRINUSE` | ポートの競合                                                                 |
| `Gateway start blocked: set gateway.mode=local`                | 設定がリモートモードになっているか、破損した設定に `gateway.mode` がない |
| 接続中の `unauthorized`                                  | クライアントと Gateway 間の認証の不一致                                      |

完全な診断手順については、[Gateway のトラブルシューティング](/ja-JP/gateway/troubleshooting)を参照してください。

## 安全性の保証

- Gateway プロトコルクライアントは、Gateway が利用できない場合に即座に失敗します（暗黙的なダイレクトチャンネルへのフォールバックはありません）。
- 無効な最初のフレーム、または接続フレームではない最初のフレームは拒否され、接続が閉じられます。
- 正常終了時には、ソケットを閉じる前に `shutdown` イベントが送出されます。

## 関連項目

- [設定](/ja-JP/gateway/configuration)
- [Gateway のトラブルシューティング](/ja-JP/gateway/troubleshooting)
- [バックグラウンドプロセス](/ja-JP/gateway/background-process)
- [ヘルスチェック](/ja-JP/gateway/health)
- [Doctor](/ja-JP/gateway/doctor)
- [認証](/ja-JP/gateway/authentication)
- [リモートアクセス](/ja-JP/gateway/remote)
- [シークレット管理](/ja-JP/gateway/secrets)
