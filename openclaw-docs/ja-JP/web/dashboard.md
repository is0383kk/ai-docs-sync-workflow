---
read_when:
    - ダッシュボードの認証または公開モードの変更
summary: Gateway ダッシュボード（コントロール UI）へのアクセスと認証
title: ダッシュボード
x-i18n:
    generated_at: "2026-07-26T10:35:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca531ad2943dfdee1cd90a4efdc1fb69c4517780e2be52237fd558b8638e7cd0
    source_path: web/dashboard.md
    workflow: 16
---

Gateway ダッシュボードは、デフォルトで `/` から提供されるブラウザ版 Control UI です（`gateway.controlUi.basePath` で上書きできます）。

すばやく開く（ローカル Gateway）:

- [http://127.0.0.1:18789/](http://127.0.0.1:18789/)（または [http://localhost:18789/](http://localhost:18789/)）
- `gateway.tls.enabled: true` を使用する場合、WebSocket エンドポイントには `https://127.0.0.1:18789/` と `wss://127.0.0.1:18789` を使用します。

主な参照先:

- 使用方法と UI 機能については [Control UI](/ja-JP/web/control-ui)。
- Serve/Funnel の自動化については [Tailscale](/ja-JP/gateway/tailscale)。
- バインドモードとセキュリティ上の注意事項については [Web サーフェス](/ja-JP/web)。

認証は、構成された Gateway 認証パスを介して WebSocket ハンドシェイク時に適用されます:

- `connect.params.auth.token`
- `connect.params.auth.password`
- `gateway.auth.allowTailscale: true` の場合は Tailscale Serve の ID ヘッダー
- `gateway.auth.mode: "trusted-proxy"` の場合は信頼済みプロキシの ID ヘッダー

[Gateway 構成](/ja-JP/gateway/configuration)の `gateway.auth` を参照してください。

<Warning>
Control UI は**管理サーフェス**です（チャット、構成、実行承認）。公開しないでください。UI は、現在のブラウザタブと選択した Gateway URL に対応するダッシュボード URL トークンを sessionStorage に保持し、読み込み後に URL から削除します。localhost、Tailscale Serve、または SSH トンネルを推奨します。
</Warning>

## 最短手順（推奨）

- オンボーディング後、CLI はダッシュボードを自動的に開き、トークンを含まないクリーンなリンクを出力します。
- いつでも再度開けます: `openclaw dashboard`（リンクをコピーし、可能であればブラウザを開き、ヘッドレス環境では SSH のヒントを出力します）。
- クリップボードとブラウザへの受け渡しが両方とも失敗した場合でも、`openclaw dashboard` はクリーンな URL を出力し、`OPENCLAW_GATEWAY_TOKEN` または `gateway.auth.token` から取得したトークンを URL フラグメントキー `token` として追加するよう案内します。ログにトークン値を出力することはありません。
- UI で共有シークレット認証を求められた場合は、構成済みのトークンまたはパスワードを Control UI の設定に貼り付けます。

## 認証の基本（ローカルとリモート）

- **Localhost**: `http://127.0.0.1:18789/` を開きます。
- **Gateway TLS**: `gateway.tls.enabled: true` の場合、ダッシュボードおよびステータスのリンクは `https://` を使用し、Control UI の WebSocket リンクは `wss://` を使用します。
- **共有シークレットトークンの取得元**: `gateway.auth.token`（または `OPENCLAW_GATEWAY_TOKEN`）。`openclaw dashboard` は、1 回限りのブートストラップ用に URL フラグメント経由でトークンを渡せます。Control UI はトークンを localStorage ではなく、現在のタブと選択した Gateway URL に対応する sessionStorage に保持します。
- **構成がない場合のランタイムトークン**: 起動時にランタイムトークンを生成したと表示された場合、そのトークンは一時的なものであり、`openclaw config get gateway.auth.token` では取得できません。ループバックでも認証は必要です。`openclaw doctor --generate-gateway-token` を実行して Gateway を再起動し、構成済みのトークンを Control UI の設定に貼り付けます。
- `gateway.auth.token` が SecretRef で管理されている場合、外部管理のトークンがシェルログ、クリップボード履歴、またはブラウザ起動引数に露出するのを避けるため、`openclaw dashboard` は設計上、トークンを含まない URL を出力、コピー、または開きます。現在のシェルで参照を解決できない場合でも、トークンを含まない URL と、実行可能な認証設定の案内を出力します。
- **共有シークレットパスワード**: 構成済みの `gateway.auth.password`（または `OPENCLAW_GATEWAY_PASSWORD`）を使用します。ダッシュボードは、再読み込み後までパスワードを保持しません。
- **ID を伴うモード**: `gateway.auth.allowTailscale: true` の場合、Tailscale Serve は ID ヘッダーを介して Control UI/WebSocket 認証を満たします。local loopback 以外の ID 対応リバースプロキシは `gateway.auth.mode: "trusted-proxy"` を満たします。どちらも WebSocket に共有シークレットを貼り付ける必要はありません。
- **localhost 以外**: Tailscale Serve、local loopback 以外への共有シークレット付きバインド、`gateway.auth.mode: "trusted-proxy"` を使用する local loopback 以外の ID 対応リバースプロキシ、または SSH トンネルを使用します。private-ingress の `gateway.auth.mode: "none"` または信頼済みプロキシの HTTP 認証を意図的に実行しない限り、HTTP API では引き続き共有シークレット認証が使用されます。[Web サーフェス](/ja-JP/web)を参照してください。

## Telegram で開く

Telegram ボットは、`/dashboard` を使用してダッシュボードを Telegram Mini App として開けます。

要件:

- Telegram が HTTPS の Mini App URL を取得できるように、`gateway.tailscale.mode: "serve"` または `"funnel"` を使用します。
- Telegram の送信者はボットの所有者である必要があります。つまり、`commands.ownerAllowFrom` または選択したアカウントの有効な `channels.telegram.allowFrom` に含まれる数値の Telegram ユーザー ID です。
- ボットとの DM で `/dashboard` を実行します。グループから呼び出した場合は、DM でコマンドを開くよう案内されるだけで、ボタンは表示されません。
- Docker インストール: Serve/Funnel モードでは、Gateway が `tailscaled` の隣でループバックにバインドされる必要がありますが、ポートを公開するブリッジネットワークではこの要件を満たせません。Gateway コンテナを `network_mode: host` で実行し、ホストの `tailscaled` ソケット（`/var/run/tailscale`）と `tailscale` CLI をコンテナにマウントします。

Mini App は、1 回限りの所有者引き継ぎを実行し、有効期間の短いブートストラップトークンを使用して Control UI にリダイレクトします。URL に共有 Gateway トークンを露出することはありません。

v1 の対象外:

- Telegram Web iframe はサポートされていません。
- 公開 URL の経路としてサポートされるのは Tailscale Serve/Funnel のみです。

<a id="if-you-see-unauthorized-1008"></a>

## 「unauthorized」/ 1008 が表示された場合

- Gateway に到達可能であることを確認します。ローカルでは `openclaw status`、リモートでは SSH トンネル `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host` を使用してから `http://127.0.0.1:18789/` を開きます。
- `AUTH_TOKEN_MISMATCH` の場合、Gateway が再試行のヒントを返すと、クライアントはキャッシュ済みのデバイストークンを使用して信頼済み再試行を 1 回行うことがあります。この再試行では、トークンにキャッシュされた承認済みスコープを再利用します（明示的な `deviceToken`/`scopes` の呼び出し元は、要求したスコープセットを維持します）。その再試行後も認証に失敗する場合は、トークンの不整合を手動で解消します。
- `AUTH_SCOPE_MISMATCH` の場合、デバイストークンは認識されていますが、要求されたスコープが付与されていません。共有 Gateway トークンをローテーションするのではなく、再ペアリングするか、新しいスコープセットを承認します。
- この再試行経路以外では、接続認証の優先順位は、明示的な共有トークンまたはパスワード、明示的な `deviceToken`、保存済みデバイストークン、ブートストラップトークンの順です。
- 非同期の Tailscale Serve 経路では、同じ `{scope, ip}` に対する失敗した試行は、認証失敗リミッターが記録する前に直列化されるため、同時に行われた 2 回目の不正な再試行ですでに `retry later` が表示されることがあります。
- トークンの不整合を修復する手順については、[トークン不整合の復旧チェックリスト](/ja-JP/cli/devices#token-drift-recovery-checklist)を参照してください。
- Gateway ホストから共有シークレットを取得または指定します:
  - トークン: `openclaw config get gateway.auth.token`
  - パスワード: 構成済みの `gateway.auth.password` または `OPENCLAW_GATEWAY_PASSWORD` を解決します
  - SecretRef 管理のトークン: 外部シークレットプロバイダーを解決するか、このシェルで `OPENCLAW_GATEWAY_TOKEN` をエクスポートし、`openclaw dashboard` を再実行します
  - 共有シークレットが構成されていないため生成されたランタイムトークン: `openclaw doctor --generate-gateway-token` を実行し、Gateway を再起動してから、構成済みのトークンを使用します
- ダッシュボードの設定で、認証フィールドにトークンまたはパスワードを貼り付けてから接続します。
- UI の言語選択は **Settings -> General -> Language** にあり、Appearance の下ではありません。

## 関連項目

- [Control UI](/ja-JP/web/control-ui)
- [WebChat](/ja-JP/web/webchat)
