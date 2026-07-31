---
read_when:
    - Tailscale 経由で Gateway にアクセスしたい場合
    - ブラウザの Control UI と設定編集が必要な場合
summary: Gateway の Web サーフェス：Control UI、バインドモード、セキュリティ
title: Web
x-i18n:
    generated_at: "2026-07-26T09:25:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 413fb029d95241f5c6043b28825727cdee52b2fa8cbe998fbbd6e3ff7b81467b
    source_path: web/index.md
    workflow: 16
---

Gateway は、Gateway WebSocket と同じポートから小規模な**ブラウザ Control UI**（Vite + Lit）を提供します。

- デフォルト: `http://<host>:18789/`
- `gateway.tls.enabled: true` を使用する場合: `https://<host>:18789/`
- 任意のプレフィックス: `gateway.controlUi.basePath` を設定（例: `/openclaw`）

機能については [Control UI](/ja-JP/web/control-ui) を参照してください。このページでは、バインドモード、セキュリティ、およびその他の Web 向けインターフェースについて説明します。

## 設定（デフォルトで有効）

アセットが存在する場合（`dist/control-ui`）、Control UI は**デフォルトで有効**です。

```json5
{
  gateway: {
    controlUi: { enabled: true, basePath: "/openclaw" }, // basePath は任意
  },
}
```

## Webhook

`hooks.enabled=true` の場合、Gateway は同じ HTTP サーバー上で Webhook エンドポイントも公開します。認証とペイロードについては、[Gateway 設定リファレンス](/ja-JP/gateway/configuration-reference#hooks)の `hooks` を参照してください。

## 管理 HTTP RPC

`POST /api/v1/admin/rpc` は、選択された Gateway コントロールプレーンメソッドを HTTP 経由で公開します。デフォルトでは無効で、`admin-http-rpc` Plugin が有効な場合にのみ登録されます。認証モデル、許可されるメソッド、および WebSocket API との比較については、[管理 HTTP RPC](/ja-JP/plugins/admin-http-rpc)を参照してください。

## Tailscale アクセス

<Tabs>
  <Tab title="統合 Serve（推奨）">
    Gateway を loopback 上に維持し、Tailscale Serve でプロキシします。

    ```json5
    {
      gateway: {
        bind: "loopback",
        tailscale: { mode: "serve" },
      },
    }
    ```

    Gateway を起動します。

    ```bash
    openclaw gateway
    ```

    `https://<magicdns>/`（または設定した `gateway.controlUi.basePath`）を開きます。

  </Tab>
  <Tab title="Tailnet バインド + トークン">
    ```json5
    {
      gateway: {
        bind: "tailnet",
        controlUi: { enabled: true },
        auth: { mode: "token", token: "your-token" },
      },
    }
    ```

    Gateway を起動します（この非 loopback の例では共有シークレットトークン認証を使用します）。

    ```bash
    openclaw gateway
    ```

    `http://<tailscale-ip>:18789/`（または設定した `gateway.controlUi.basePath`）を開きます。

  </Tab>
  <Tab title="パブリックインターネット（Funnel）">
    ```json5
    {
      gateway: {
        bind: "loopback",
        tailscale: { mode: "funnel" },
        auth: { mode: "password" }, // または OPENCLAW_GATEWAY_PASSWORD
      },
    }
    ```

    `tailscale.mode: "funnel"` には `gateway.auth.mode: "password"` が必要です。Serve と Funnel はどちらも `gateway.bind: "loopback"` を必要とします。

  </Tab>
</Tabs>

## セキュリティ上の注意

- Gateway 認証はデフォルトで必須です。有効な方式は、トークン、パスワード、信頼済みプロキシ、または有効化されている場合の Tailscale Serve ID ヘッダーです。
- 非 loopback バインドでも Gateway 認証は**必須**です。トークン／パスワード認証、または `gateway.auth.mode: "trusted-proxy"` を備えた ID 対応リバースプロキシを使用してください。
- オンボーディングウィザードは、デフォルトで共有シークレット認証を作成し、loopback 上でも通常は Gateway トークンを生成します。
- 共有シークレットモードでは、UI は WebSocket ハンドシェイク中に `connect.params.auth.token` または `connect.params.auth.password` を送信します。
- `gateway.tls.enabled: true` を使用すると、ローカルのダッシュボード／ステータスヘルパーは `https://` URL と `wss://` WebSocket URL を表示します。
- ID 情報を伴うモード（Tailscale Serve、`trusted-proxy`）では、WebSocket 認証チェックは共有シークレットではなく、リクエストヘッダーによって満たされます。
- 公開された非 loopback Control UI デプロイでは、`gateway.controlUi.allowedOrigins` を明示的に設定してください（完全なオリジン）。loopback、RFC1918/link-local、`.local`、`.ts.net`、および Tailscale CGNAT ホストでは、プライベートな同一オリジンからの読み込みは、この設定がなくても許可されます。
- `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback: true` は Host ヘッダーによるオリジンフォールバックを有効にします。これは危険なセキュリティ低下です。
- Serve を使用する場合、`gateway.auth.allowTailscale: true` であれば、Tailscale ID ヘッダーが Control UI/WebSocket 認証を満たします（トークン／パスワードは不要です）。HTTP API エンドポイントは Tailscale ID ヘッダーを使用せず、常に Gateway の通常の HTTP 認証モードに従います。Serve 経由でも明示的な認証情報を必須にするには、`gateway.auth.allowTailscale: false` を設定してください。このトークンレスフローでは、Gateway ホスト自体が信頼されていることを前提とします。[Tailscale](/ja-JP/gateway/tailscale)および[セキュリティ](/ja-JP/gateway/security)を参照してください。

## UI のビルド

Gateway は `dist/control-ui` から静的ファイルを提供します。

```bash
pnpm ui:build
```
