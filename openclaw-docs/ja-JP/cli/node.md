---
read_when:
    - ヘッドレス Node ホストの実行
    - system.run 用に macOS 以外の Node をペアリングする
summary: '`openclaw node`（ヘッドレス Node ホスト）の CLI リファレンス'
title: Node
x-i18n:
    generated_at: "2026-07-26T08:58:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 341539d05545ddcbf6175c34af7dca49332ba55906283b9933b9c9b1732c0e4d
    source_path: cli/node.md
    workflow: 16
---

# `openclaw node`

Gateway WebSocket に接続し、このマシン上で
`system.run` / `system.which` を公開する**ヘッドレス Node ホスト**を実行します。

macOS では、メニューバーアプリがすでにこの Node ホストランタイムを独自の
Node 接続に組み込み、Mac ネイティブ機能を追加しています。アプリを使用せず、
意図的にヘッドレス Node を実行する場合にのみ、Mac で `openclaw node run` を使用してください。
両方を実行すると、同じマシンに対して 2 つの Node ID が作成されます。

## Node ホストを使用する理由

完全な macOS コンパニオンアプリをインストールせずに、エージェントからネットワーク上の
**他のマシンでコマンドを実行**したい場合は、Node ホストを使用します。

一般的なユースケース:

- リモートの Linux/Windows マシン（ビルドサーバー、ラボマシン、NAS）でコマンドを実行する。
- Gateway 上の exec は**サンドボックス化**したまま、承認済みの実行を他のホストに委任する。
- 自動化または CI Node 向けに、軽量なヘッドレス実行ターゲットを提供する。

実行は引き続き、Node ホスト上の**exec 承認**とエージェントごとの許可リストによって
制御されるため、コマンドアクセスの範囲を限定し、明示的に管理できます。

`openclaw node run` は接続後、Plugin または MCP バックエンドのツールを公開できます。
Gateway は、ペアリング済み Node のディスクリプターをデフォルトで信頼しますが、
各ディスクリプターのコマンドは Node の承認済みコマンドサーフェス内に限定されます。
エージェントには、受け入れられた各ディスクリプターが通常の Plugin ツールとして表示されますが、
実行は引き続き `node.invoke` を経由するため、Node を切断すると、新しい
エージェント実行からそのツールが削除されます。Gateway のオペレーターは
`gateway.nodes.pluginTools.enabled: false` で公開を無効にできます。

宣言型 MCP ツールの場合は、Node マシン上の `openclaw.json` にある
`nodeHost.mcp.servers` の下へ通常の MCP サーバー構成を追加し、Node ホストを
再起動します。Node は承認制の `mcp.tools.call.v1` コマンドファミリーを宣言し、
接続後にリストされたツールを公開します。後でサーバーリストを変更しても、
再ペアリングは不要です。
[Node ホスト型 MCP サーバー](/ja-JP/nodes#node-hosted-mcp-servers)を参照してください。

## ブラウザプロキシ（設定不要）

Node 上で `browser.enabled` が無効になっていない場合、Node ホストは
ブラウザプロキシを自動的にアドバタイズします。これにより、追加設定なしで
エージェントがその Node 上のブラウザ自動化を使用できます。

デフォルトでは、プロキシは Node の通常のブラウザプロファイルサーフェスを公開します。
`nodeHost.browserProxy.allowProfiles` を設定すると、プロキシは制限モードになります。
許可リストにないプロファイルの指定は拒否され、永続プロファイルの
作成・削除ルートはプロキシ経由ではブロックされます。

必要に応じて Node 上で無効にします。

```json5
{
  nodeHost: {
    browserProxy: {
      enabled: false,
    },
  },
}
```

## 実行（フォアグラウンド）

```bash
openclaw node run --host <gateway-host> --port 18789
```

オプション:

- `--host <host>`: Gateway WebSocket ホスト（デフォルト: `127.0.0.1`）
- `--port <port>`: Gateway WebSocket ポート（デフォルト: `18789`）
- `--context-path <path>`: Gateway WebSocket コンテキストパス（例: `/openclaw-gw`）。WebSocket URL に追加されます。
- `--tls`: Gateway 接続に TLS を使用する
- `--no-tls`: ローカルの Gateway 設定で TLS が有効な場合でも、平文の Gateway 接続を強制する
- `--tls-fingerprint <sha256>`: 想定される TLS 証明書フィンガープリント（sha256）
- `--node-id <id>`: 共有 SQLite 状態に保存されたクライアントインスタンス ID を上書きする（ペアリングはリセットされません）
- `--display-name <name>`: Node の表示名を上書きする

## Node ホストの Gateway 認証

`openclaw node run` と `openclaw node install` は、設定または環境変数から Gateway 認証を解決します（Node コマンドには `--token`/`--password` フラグはありません）。

- `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD` が最初に確認されます。
- 次にローカル設定へフォールバックします: `gateway.auth.token` / `gateway.auth.password`。
- ローカルモードでは、Node ホストは意図的に `gateway.remote.token` / `gateway.remote.password` を継承しません。
- `gateway.auth.token` / `gateway.auth.password` が SecretRef 経由で明示的に設定され、解決できない場合、Node 認証の解決はフェイルクローズします（リモートへのフォールバックで隠蔽されません）。
- `gateway.mode=remote` では、リモートクライアントフィールド（`gateway.remote.token` / `gateway.remote.password`）もリモートの優先順位ルールに従って対象になります。
- Node ホストの認証解決では、`OPENCLAW_GATEWAY_*` 環境変数のみが使用されます。

平文の `ws://` Gateway に接続する Node では、loopback、プライベート IP
リテラル、`.local`、および Tailnet の `*.ts.net` ホストが許可されます。
その他の信頼済みプライベート DNS 名を使用する場合は `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1` を設定してください。
設定しない場合、Node の起動はフェイルクローズし、`wss://`、SSH トンネル、または
Tailscale を使用するよう求められます。これはプロセス環境によるオプトインであり、
`openclaw.json` の設定キーではありません。
`openclaw node install` は、インストールコマンドの環境にこの値が存在する場合、
管理対象の Node サービスに永続化します。

## サービス（バックグラウンド）

ヘッドレス Node ホストをユーザーサービスとしてインストールします（macOS では launchd、
Linux では systemd、Windows では Windows Task Scheduler）。

```bash
openclaw node install --host <gateway-host> --port 18789
```

オプション:

- `--host <host>`: Gateway WebSocket ホスト（デフォルト: `127.0.0.1`）
- `--port <port>`: Gateway WebSocket ポート（デフォルト: `18789`）
- `--context-path <path>`: Gateway WebSocket コンテキストパス（例: `/openclaw-gw`）。WebSocket URL に追加されます。
- `--tls`: Gateway 接続に TLS を使用する
- `--tls-fingerprint <sha256>`: 想定される TLS 証明書フィンガープリント（sha256）
- `--node-id <id>`: 共有 SQLite 状態に保存されたクライアントインスタンス ID を上書きする（ペアリングはリセットされません）
- `--display-name <name>`: Node の表示名を上書きする
- `--runtime <runtime>`: サービスランタイム（`node`）
- `--force`: すでにインストール済みの場合、再インストールまたは上書きする

サービスを管理します。

```bash
openclaw node status
openclaw node start
openclaw node stop
openclaw node restart
openclaw node uninstall
```

フォアグラウンドの Node ホスト（サービスなし）には `openclaw node run` を使用します。

サービスコマンドでは、機械可読な出力のために `--json` を指定できます。

Node ホストは、Gateway の再起動やネットワーク切断に対してプロセス内で再試行します。
Gateway がトークン、パスワード、またはブートストラップ認証の終端的な一時停止を報告した場合、
Node ホストは切断の詳細をログに記録し、非ゼロで終了します。これにより launchd/systemd/Task Scheduler は、
新しい設定と認証情報で Node ホストを再起動できます。ペアリングが必要な一時停止は、
保留中のリクエストを承認できるよう、フォアグラウンドフローに留まります。

## ペアリング

最初の接続時に、Gateway 上へ保留中のデバイスペアリングリクエスト（`role: node`）が作成されます。

Gateway ホストから Node ホストへ非対話的に SSH 接続できる場合（同一ユーザー、
信頼済みホストキー）、保留中のリクエストは自動的に承認されます。Gateway は
SSH 経由で Node ホスト上の `openclaw node identity --json` を実行し、
デバイスキーが完全に一致した場合に承認します。これはデフォルトで有効です。
要件と無効化方法（`gateway.nodes.pairing.sshVerify: false`）については、
[SSH 検証済みデバイスの自動承認](/ja-JP/gateway/pairing#ssh-verified-device-auto-approval-default)
を参照してください。

それ以外の場合は、次のコマンドで手動承認します。

```bash
openclaw devices list
openclaw devices approve <requestId>
```

Gateway が照合するローカル Node ID を確認します。

```bash
openclaw node identity --json
```

`state/openclaw.sqlite` の `primary` 行からデバイス ID と公開鍵を出力し、
データベースや新しい ID を作成することはありません。

厳格に管理された Node ネットワークでは、Gateway のオペレーターが信頼済み CIDR からの
初回 Node ペアリングを自動承認するよう明示的にオプトインできます。

```json5
{
  gateway: {
    nodes: {
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
      },
    },
  },
}
```

これはデフォルトで無効です（`autoApproveCidrs` は未設定）。Gateway が信頼する
クライアント IP からの、要求スコープを持たない新規 `role: node` ペアリングにのみ適用されます。
オペレーター／ブラウザクライアント、Control UI、WebChat、およびロール、スコープ、
メタデータ、公開鍵のアップグレードには、引き続き手動承認が必要です。

Node が変更された認証詳細（ロール／スコープ／公開鍵）でペアリングを再試行すると、
以前の保留中リクエストは置き換えられ、新しい `requestId` が作成されます。
承認前に `openclaw devices list` をもう一度実行してください。

### ID とペアリング状態

ヘッドレス Node では、クライアントインスタンス ID と、Gateway がペアリングおよび
ルーティングに使用する署名済みデバイス ID が分離されています。この状態は
OpenClaw の状態ディレクトリ（デフォルトでは `~/.openclaw`、
設定されている場合は `$OPENCLAW_STATE_DIR`）に保存されます。

| 状態                                                    | 用途                                                                                                                          |
| -------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `state/openclaw.sqlite` (`node_host_config`)             | クライアントインスタンス ID、表示名、Gateway 接続メタデータ。クライアントはこの ID を `instanceId` として送信します。                     |
| `state/openclaw.sqlite` (`device_identities`, `primary`) | 署名済み Ed25519 鍵ペアと、それから導出されたデバイス ID。署名付き接続では、このデバイス ID がルーティング対象の Node ID およびペアリング ID になります。 |
| `state/openclaw.sqlite` (`device_auth_tokens`)           | 暗号学的デバイス ID とロールをキーとする、ペアリング済みデバイストークン。                                                                 |

`--node-id` は、共有 SQLite 状態のクライアントインスタンス ID のみを変更します。
暗号学的デバイス ID の変更や、ペアリング認証の消去は行いません。廃止された
`node.json` を `openclaw doctor --fix` で移行しても、同様にペアリングはリセットされません。
Node を失効させて再ペアリングするには、次の手順を実行します。

1. Gateway 上で `openclaw nodes remove --node <id|name|ip>` を実行します。
2. Node 上で、インストール済みサービスを `openclaw node restart` で再起動するか、
   停止してフォアグラウンドの `openclaw node run` コマンドを再実行します。これにより、
   デバイスペアリングフローが開始されます。`openclaw devices list` にリクエストが表示されず、
   Node が `AUTH_DEVICE_TOKEN_MISMATCH` を報告する場合は、もう一度再起動または再実行してください。
   拒否された試行によって、失効済みのローカルトークンが消去され、次の試行で
   ペアリングを要求できるようになります。
3. Gateway 上で `openclaw devices list` を実行し、続いて
   `openclaw devices approve <deviceRequestId>` を実行します。
4. Node をもう一度再起動または再実行します。ペアリングのために一時停止している
   クライアントは、承認後も自動的には再開されません。この再接続により、
   別個のコマンドサーフェスリクエストが作成されます。
5. Gateway 上で `openclaw nodes pending` を実行し、続いて
   `openclaw nodes approve <nodeRequestId>` を実行します。

2 つのリクエスト ID は異なります。適用可能な信頼済み CIDR ポリシーによって、
初回のデバイスペアリング手順を自動承認できますが、コマンドサーフェスの承認は
引き続き別個に確認されます。

旧バージョンの OpenClaw では、Node ホストの状態を `node.json`、
署名済み ID を `identity/device.json`、ペアリング済み認証を
`identity/device-auth.json` に保存していました。Node ホストを停止し、
`openclaw doctor --fix` を一度実行してください。Doctor は廃止された各ソースを確保し、
検証してから、正規の SQLite 行へインポートして検証し、古いファイルを削除します。
廃止されたファイルまたは中断された Doctor の確保処理が残っている間、通常の
Node コマンドはフェイルクローズし、この修復手順を案内します。
`state/openclaw.sqlite` は非公開に保ってください。デバイス鍵ペアと認証トークンが含まれています。

## Exec 承認

`system.run` は、ローカルの exec 承認によって制御されます。

- `$OPENCLAW_STATE_DIR/exec-approvals.json`、または
  変数が未設定の場合は `~/.openclaw/exec-approvals.json`
- [Exec 承認](/ja-JP/tools/exec-approvals)
- `openclaw approvals --node <id|name|ip>`（Gateway から編集）

承認済みの非同期 Node exec では、OpenClaw はプロンプトを表示する前に正規の
`systemRunPlan` を準備します。その後の承認済み `system.run` 転送では、
保存された計画が再利用されます。そのため、承認リクエストの作成後に
コマンド／cwd／セッションフィールドを編集しても、Node が実行する内容を変更するのではなく、
拒否されます。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [Nodes](/ja-JP/nodes)
