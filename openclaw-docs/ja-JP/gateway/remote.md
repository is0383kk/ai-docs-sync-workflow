---
read_when:
    - リモート Gateway セットアップの実行またはトラブルシューティング
summary: Gateway WS、SSH トンネル、tailnet を使用したリモートアクセス
title: リモートアクセス
x-i18n:
    generated_at: "2026-07-26T09:04:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8f05e32fcfa16d5ddfcd684d0550c9af311914e2b4d91c95edad3490dc2e56d9
    source_path: gateway/remote.md
    workflow: 16
---

OpenClaw はホスト上で 1 つの Gateway（マスター）を実行し、すべてのクライアントをそこに接続します。Gateway はセッション、認証プロファイル、チャンネル、状態を管理し、それ以外はすべてクライアントです。

- **オペレーター**（ユーザーまたは macOS アプリ）: Gateway に到達できる場合は、LAN/Tailnet WebSocket への直接接続が最も簡単です。SSH トンネリングは汎用的なフォールバックです。
- **Node**（iOS/Android およびその他のデバイス）: Gateway の **WebSocket**（LAN/tailnet または SSH トンネル）に接続します。

## 基本的な考え方

Gateway WebSocket はデフォルトでポート `18789`（`gateway.port`）の **ループバック**にバインドされます。リモートで使用するには、Tailscale Serve または信頼済みの LAN-Tailnet バインドを通じて公開するか、SSH 経由でループバックポートを転送します。

## トポロジーの選択肢

| 構成                              | Gateway の実行場所                                                                                          | 最適な用途                                                                                                                                                 |
| --------------------------------- | ----------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| tailnet 内の常時稼働 Gateway      | Tailscale または SSH 経由でアクセスする常設ホスト（VPS またはホームサーバー）                               | 頻繁にスリープするものの、エージェントを常時稼働させる必要があるノート PC。[exe.dev](/ja-JP/install/exe-dev)（簡単な VM）または [Hetzner](/ja-JP/install/hetzner)（本番環境向け VPS）を参照してください。 |
| 自宅のデスクトップ                | デスクトップ。ノート PC は macOS アプリのリモートモード（Settings → Connection → OpenClaw runs）で接続     | 電源が常時オンのハードウェア上でエージェントを稼働させる場合。手順書: [macOS リモートアクセス](/ja-JP/platforms/mac/remote)。                                    |
| ノート PC                         | SSH トンネルまたは Tailscale Serve で安全に公開するノート PC（`gateway.bind: "loopback"` を維持）                  | 単一マシン構成。[Tailscale](/ja-JP/gateway/tailscale) および [Web](/ja-JP/web) を参照してください。                                                                     |

常時稼働構成とノート PC 構成では、`gateway.bind: "loopback"` を維持し、Control UI には **Tailscale Serve** を使用するか、`gateway.remote.transport: "direct"` を設定した信頼済みの LAN/Tailnet バインドを使用することを推奨します。SSH トンネルは、どのマシンからでも機能するフォールバックです。

## コマンドフロー（どこで何が実行されるか）

1 つの Gateway が状態とチャンネルを管理し、Node は周辺機器として機能します。例（Telegram メッセージを Node ツールへルーティングする場合）:

1. Telegram メッセージが **Gateway** に到着します。
2. Gateway が **エージェント**を実行し、Node ツールを呼び出すかどうかを判断します。
3. Gateway が Gateway WebSocket（`node.invoke` RPC）経由で **Node** を呼び出します。
4. Node が結果を返し、Gateway が Telegram に返信します。

Node は Gateway サービスを実行しません。分離されたプロファイルを意図的に実行する場合を除き、ホストごとに 1 つの Gateway のみを実行してください（[複数の Gateway](/ja-JP/gateway/multiple-gateways) を参照）。macOS アプリの「Node モード」は、Gateway WebSocket 経由の Node クライアントにすぎません。

## SSH トンネル（CLI + ツール）

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

トンネルが有効な場合、`openclaw health` と `openclaw status --deep` は `ws://127.0.0.1:18789` 経由でリモート Gateway に到達します。`openclaw gateway status`、`openclaw gateway health`、`openclaw gateway probe`、`openclaw gateway call` も、`--url` を使用して転送 URL を指定できます。

<Note>
`18789` を、設定済みの `gateway.port`（または `--port` / `OPENCLAW_GATEWAY_PORT`）に置き換えてください。
</Note>

<Warning>
`--url` は、設定または環境の認証情報へフォールバックすることはありません。`--token` または `--password` を明示的に渡してください。指定しない場合、クライアントは認証情報を送信せず、接続先 Gateway が認証を要求すると接続に失敗します。
</Warning>

## CLI のリモートデフォルト

CLI コマンドがデフォルトで使用するリモート接続先を永続化します。

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",
      token: "your-token",
    },
  },
}
```

Gateway がループバック専用の場合は、URL を `ws://127.0.0.1:18789` のままにし、先に SSH トンネルを開いてください。macOS アプリの SSH トンネルトランスポートでは、検出された Gateway のホスト名を `gateway.remote.sshTarget`（`user@host` または `user@host:port`）に指定し、`gateway.remote.url` はローカルトンネル URL のままにします。リモートポートがローカルポートと異なる場合は、`gateway.remote.remotePort` を設定します。

ホストキー検証はデフォルトで厳格です（`gateway.remote.sshHostKeyPolicy: "strict"`）。代わりに有効な OpenSSH 設定へ委任するには、これを `"openssh"` に設定します。有効にする前に、ユーザーおよびシステムの SSH 設定を確認してください。

信頼済みの LAN または Tailnet 上ですでに到達可能な Gateway には、直接モードを使用します。

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      transport: "direct",
      url: "ws://192.168.0.202:18789",
      token: "your-token",
    },
  },
}
```

## 認証情報の優先順位

Gateway の認証情報の解決は、呼び出し、プローブ、ステータスの各パスと Discord の実行承認監視で、1 つの共通コントラクトに従います。Node ホストも、ローカルモードの例外が 1 つある点を除き、同じコントラクトを使用します（`gateway.remote.*` は無視されます）。

- 明示的な認証情報（`--token`、`--password`、またはツールの `gatewayToken`）は、明示的な認証を受け付ける呼び出しパスでは常に優先されます。
- URL オーバーライドの安全性:
  - CLI の `--url` は、暗黙的な設定または環境の認証情報を再利用しません。
  - 環境の `OPENCLAW_GATEWAY_URL` は、環境の認証情報（`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`）のみを使用できます。
- ローカルモードのデフォルト:
  - トークン: `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token` -> `gateway.remote.token`（ローカルトークンが未設定の場合のみリモートへフォールバック）
  - パスワード: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.auth.password` -> `gateway.remote.password`（ローカルパスワードが未設定の場合のみリモートへフォールバック）
- リモートモードのデフォルト:
  - トークン: `gateway.remote.token` -> `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token`
  - パスワード: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.remote.password` -> `gateway.auth.password`
- Node ホストのローカルモードの例外: `gateway.remote.token` / `gateway.remote.password` は無視されます。
- リモートのプローブ/ステータスにおけるトークン確認は、デフォルトで厳格です。リモートモードを接続先とする場合は、`gateway.remote.token` のみを使用します（ローカルトークンへのフォールバックはありません）。
- Gateway の環境オーバーライドは、`OPENCLAW_GATEWAY_*` のみを使用します。

## Chat UI のリモートアクセス

WebChat には独立した HTTP ポートはありません。SwiftUI のチャット UI は Gateway WebSocket に直接接続します。

- SSH 経由で `18789` を転送し（前述を参照）、クライアントを `ws://127.0.0.1:18789` に接続します。
- LAN/Tailnet の直接モードでは、設定済みのプライベート `ws://` またはセキュアな `wss://` URL にクライアントを接続します。
- macOS では、アプリのリモートモードが選択されたトランスポートを自動的に管理します。

## macOS アプリのリモートモード

macOS メニューバーアプリは、リモートステータス確認、WebChat、Voice Wake 転送を含む同じ構成をエンドツーエンドで処理します。手順書: [macOS リモートアクセス](/ja-JP/platforms/mac/remote)。

## セキュリティルール（リモート/VPN）

バインドが必要であると確信している場合を除き、Gateway は **ループバック専用**にしてください。

- **ループバック + SSH/Tailscale Serve** が最も安全なデフォルトです（公開されません）。
- 平文の `ws://` は、ループバック、プライベート/LAN（RFC 1918）、リンクローカル、CGNAT、`.local`、`.ts.net` のホストで許可されます。公開リモートホストでは `wss://` を使用する必要があります。
- **非ループバックバインド**（`lan`/`tailnet`/`custom`、またはループバックが利用できない場合の `auto`）では、Gateway 認証（トークン、パスワード、または `gateway.auth.mode: "trusted-proxy"` を設定した ID 対応リバースプロキシ）を使用する必要があります。
- `gateway.remote.token` / `.password` はクライアント認証情報のソースです。それだけではサーバー認証を設定しません。
- ローカル呼び出しパスでは、`gateway.auth.*` が未設定の場合にのみ、`gateway.remote.*` をフォールバックとして使用できます。
- `gateway.auth.token` / `gateway.auth.password` が SecretRef で明示的に設定されているにもかかわらず解決できない場合、解決はフェイルクローズします（リモートフォールバックによる隠蔽はありません）。
- `gateway.remote.tlsFingerprint` は、`wss://` のリモート TLS 証明書を固定します。これには、オペレーター/制御トラフィックと、macOS の直接モードにおけるコンパニオン Node の両方が含まれます。保存済みのピンがない場合、macOS は通常のシステム信頼検証に合格した後にのみ初回使用時に固定します。自己署名またはプライベート CA の Gateway では、明示的なフィンガープリントまたは SSH 経由のリモート接続が必要です。
- __Tailscale Serve** は、`gateway.auth.allowTailscale: true` の場合、ID ヘッダーを介して Control UI/WebSocket トラフィックを認証できます。HTTP API エンドポイントではこのヘッダー認証を使用せず、代わりに Gateway の通常の HTTP 認証モードに従います。このトークンレスフローでは Gateway ホストが信頼されていることを前提とします。すべての接続で共有シークレット認証を使用するには、これを `false` に設定します。
- **信頼済みプロキシ**認証は、デフォルトで非ループバックの ID 対応プロキシを想定します。同一ホストのループバックリバースプロキシでは、`gateway.auth.trustedProxy.allowLoopback = true` を明示的に設定する必要があります。
- ブラウザーによる制御はオペレーターアクセスと同等に扱ってください。tailnet のみに制限し、Node のペアリングを意図的に行います。

詳細: [セキュリティ](/ja-JP/gateway/security)。

### macOS: LaunchAgent による永続的な SSH トンネル

macOS クライアントでは、SSH の `LocalForward` 設定エントリと、再起動やクラッシュ後もトンネルを維持する LaunchAgent を使用するのが、最も簡単な永続構成です。

#### ステップ 1: SSH 設定を追加する

`~/.ssh/config` を編集します。

```ssh
Host remote-gateway
    HostName <REMOTE_IP>
    User <REMOTE_USER>
    LocalForward 18789 127.0.0.1:18789
    IdentityFile ~/.ssh/id_rsa
```

`<REMOTE_IP>` と `<REMOTE_USER>` を実際の値に置き換えてください。

#### ステップ 2: SSH キーをコピーする（初回のみ）

```bash
ssh-copy-id -i ~/.ssh/id_rsa <REMOTE_USER>@<REMOTE_IP>
```

#### ステップ 3: Gateway トークンを設定する

```bash
openclaw config set gateway.remote.token "<your-token>"
```

リモート Gateway がパスワード認証を使用する場合は、代わりに `gateway.remote.password` を使用します。`OPENCLAW_GATEWAY_TOKEN` もシェルレベルのオーバーライドとして引き続き有効ですが、永続的なリモートクライアント構成には `gateway.remote.token` / `gateway.remote.password` を使用します。

#### ステップ 4: LaunchAgent を作成する

`~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist` として保存します。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.ssh-tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/ssh</string>
        <string>-N</string>
        <string>remote-gateway</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

#### ステップ 5: LaunchAgent を読み込む

```bash
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist
```

トンネルはログイン時に自動的に起動し、クラッシュ時に再起動して、転送ポートを利用可能な状態に保ちます。

<Note>
古い構成の `com.openclaw.ssh-tunnel` LaunchAgent が残っている場合は、アンロードして削除してください。
</Note>

#### トラブルシューティング

```bash
# トンネルが実行中か確認する
ps aux | grep "ssh -N remote-gateway" | grep -v grep
lsof -i :18789

# トンネルを再起動する
launchctl kickstart -k gui/$UID/ai.openclaw.ssh-tunnel

# トンネルを停止する
launchctl bootout gui/$UID/ai.openclaw.ssh-tunnel
```

| 設定項目                             | 機能                                                         |
| ------------------------------------ | ------------------------------------------------------------ |
| `LocalForward 18789 127.0.0.1:18789` | ローカルポート 18789 をリモートポート 18789 に転送します     |
| `ssh -N`                             | リモートコマンドを実行せずに SSH 接続します（ポート転送のみ） |
| `KeepAlive`                          | クラッシュした場合、トンネルを自動的に再起動します           |
| `RunAtLoad`                          | ログイン時に LaunchAgent が読み込まれるとトンネルを起動します |

## 関連項目

- [Tailscale](/ja-JP/gateway/tailscale)
- [認証](/ja-JP/gateway/authentication)
- [リモート Gateway のセットアップ](/ja-JP/gateway/remote-gateway-readme)
