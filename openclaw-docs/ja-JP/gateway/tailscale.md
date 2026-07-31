---
read_when:
    - localhost 外部への Gateway Control UI の公開
    - tailnet または公開ダッシュボードへのアクセスの自動化
summary: Gateway ダッシュボード向けに統合された Tailscale Serve/Funnel
title: Tailscale
x-i18n:
    generated_at: "2026-07-26T10:04:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e201a64ac427994401fae1b934d94e0c5afe976b4acd34d45b059978f5f1807e
    source_path: gateway/tailscale.md
    workflow: 16
---

OpenClaw は、Gateway ダッシュボードと WebSocket ポート向けに Tailscale **Serve**（tailnet）または **Funnel**（公開）を自動設定できます。これにより、Gateway を loopback にバインドしたまま、Tailscale が HTTPS、ルーティング、および（Serve の場合は）ID ヘッダーを提供します。

## モード

`gateway.tailscale.mode`:

| モード            | 動作                                                                    |
| --------------- | --------------------------------------------------------------------------- |
| `serve`         | `tailscale serve` を介した tailnet 専用の Serve。Gateway は `127.0.0.1` のままです。 |
| `funnel`        | `tailscale funnel` を介した公開 HTTPS。共有パスワードが必要です。            |
| `off`（デフォルト） | Tailscale の自動化なし。                                                    |

ステータスと監査の出力では、この OpenClaw の Serve/Funnel モードを **Tailscale 公開状態** と表記します。`off` は、OpenClaw が Serve または Funnel を管理していないことを意味します。ローカルの Tailscale デーモンが停止している、またはログアウトしているという意味ではありません。

## 設定例

### tailnet 専用（Serve）

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" },
  },
}
```

開く: `https://<magicdns>/`（または設定済みの `gateway.controlUi.basePath`）

デバイスのホスト名ではなく、名前付き Tailscale Service を介して Control UI を公開するには、`gateway.tailscale.serviceName` を Service 名に設定します。

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve", serviceName: "svc:openclaw" },
  },
}
```

起動時には、デバイスのホスト名ではなく、Service URL が `https://openclaw.<tailnet-name>.ts.net/` として報告されます。Tailscale Services では、ホストが tailnet 内で承認済みのタグ付き Node である必要があります。この機能を有効にする前に、Tailscale でタグを設定して Service を承認してください。そうしないと、Gateway の起動中に `tailscale serve --service=...` が失敗します。

### tailnet 専用（Tailnet IP にバインド）

Serve/Funnel を使用せずに、Gateway を Tailnet IP で直接リッスンさせるには、次の設定を使用します。

```json5
{
  gateway: {
    bind: "tailnet",
    auth: { mode: "token", token: "your-token" },
  },
}
```

別の Tailnet デバイスから接続します。

- Control UI: `http://<tailscale-ip>:18789/`
- WebSocket: `ws://<tailscale-ip>:18789`

<Note>
バインド可能な Tailnet IPv4 が存在する場合、認証済みの同一ホストクライアント向けに、Gateway では `http://127.0.0.1:18789` も必要です。起動時に Tailnet アドレスが使用できない場合は、loopback のみにフォールバックします。Tailscale が使用可能になった後、直接 Tailnet アクセスを追加するには再起動してください。どちらの経路でも、LAN または公開アクセスは追加されません。
</Note>

### 公開インターネット（Funnel + 共有パスワード）

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "funnel" },
    auth: { mode: "password", password: "replace-me" },
  },
}
```

パスワードをディスクにコミットするのではなく、`OPENCLAW_GATEWAY_PASSWORD` の使用を推奨します。

## CLI の例

```bash
openclaw gateway --tailscale serve
openclaw gateway --tailscale funnel --auth password
```

## 認証

`gateway.auth.mode` がハンドシェイクを制御します。

| モード                                                   | ユースケース                                                                            |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| `none`                                                 | プライベートな受信接続のみ                                                                |
| `token`（`OPENCLAW_GATEWAY_TOKEN` が設定されている場合のデフォルト） | 共有トークン                                                                        |
| `password`                                             | `OPENCLAW_GATEWAY_PASSWORD` または設定を介した共有シークレット                             |
| `trusted-proxy`                                        | ID 対応リバースプロキシ。[信頼済みプロキシ認証](/ja-JP/gateway/trusted-proxy-auth)を参照してください |

### Tailscale ID ヘッダー（Serve のみ）

`tailscale.mode: "serve"` かつ `gateway.auth.allowTailscale` が `true` の場合、Control UI/WebSocket の認証では、トークンやパスワードの代わりに Tailscale ID ヘッダー（`tailscale-user-login`）を使用できます。OpenClaw は、ローカルの Tailscale デーモン（`tailscale whois`）を介してリクエストの `x-forwarded-for` アドレスを解決し、ヘッダーのログイン情報と一致することを確認してから、そのヘッダーを受け入れます。リクエストがこの経路の対象になるのは、Tailscale の `x-forwarded-for`、`x-forwarded-proto`、および `x-forwarded-host` ヘッダーを伴って loopback から到着した場合のみです。

このトークン不要のフローでは、Gateway ホストが信頼されていることを前提とします。信頼できないローカルコードが同じホスト上で実行される可能性がある場合は、`gateway.auth.allowTailscale: false` を設定し、代わりにトークンまたはパスワード認証を必須にしてください。

バイパスの範囲:

- Control UI の WebSocket 認証面にのみ適用されます。HTTP API エンドポイント（`/v1/*`、`/tools/invoke`、`/api/channels/*` など）では、Tailscale ID ヘッダー認証は使用されません。常に Gateway の通常の HTTP 認証モードに従います。
- ブラウザーのデバイス ID をすでに保持している Control UI のオペレーターセッションでは、検証済みの Tailscale ID により、ブートストラップトークン/QR ペアリングの往復処理が省略されます。
- デバイス ID 自体はバイパスされません。デバイスのないクライアントは引き続き拒否され、Node ロールの接続も通常のペアリングと認証チェックを通過します。

## 注意事項

- Tailscale Serve/Funnel には、`tailscale` CLI がインストールされ、ログイン済みである必要があります。
- 公開状態になることを避けるため、認証モードが `password` でない限り、`tailscale.mode: "funnel"` は起動を拒否します。
- `gateway.tailscale.serviceName` は Serve モードにのみ適用され、`tailscale serve --service=<name>` に渡されます。値には Tailscale の `svc:<dns-label>` 形式（例: `svc:openclaw`）を使用する必要があります。Tailscale では Service ホストがタグ付き Node である必要があり、Serve で公開する前に管理コンソールでの Service の承認が必要になる場合があります。
- `gateway.tailscale.resetOnExit` は、シャットダウン時に `tailscale serve`/`tailscale funnel` の設定を元に戻します。
- `gateway.tailscale.preserveFunnel: true` は、外部で設定された `tailscale funnel` ルートを Gateway の再起動後も維持します。`mode: "serve"` を使用すると、OpenClaw は Serve を再適用する前に `tailscale funnel status` を確認し、Funnel ルートがすでに Gateway ポートを対象としている場合はスキップします。OpenClaw が管理する Funnel のパスワード専用ポリシーに変更はありません。
- `gateway.bind: "tailnet"` は、Tailnet IPv4 が使用可能な場合に、直接 Tailnet バインド（HTTPS なし、Serve/Funnel なし）と必須のローカル `127.0.0.1` を使用します。それ以外の場合は、loopback のみにフォールバックします。
- `gateway.bind: "auto"` は loopback を優先します。同一ホストからの loopback アクセスを維持しながら、ネットワーク公開範囲を Tailnet に制限するには、`tailnet` を使用してください。
- Serve/Funnel が公開するのは、**Gateway の Control UI + WS** のみです。Node は同じ Gateway WS エンドポイント経由で接続するため、Serve は Node アクセスにも使用できます。

### Tailscale の前提条件と制限

- Serve では、tailnet で HTTPS が有効になっている必要があります。有効でない場合は CLI が入力を求めます。
- Serve は Tailscale ID ヘッダーを挿入しますが、Funnel は挿入しません。
- Funnel には、Tailscale v1.38.3+、MagicDNS、HTTPS の有効化、および Funnel Node 属性が必要です。
- Funnel が TLS 経由でサポートするポートは、`443`、`8443`、および `10000` のみです。
- macOS で Funnel を使用するには、オープンソース版の Tailscale アプリが必要です。

## ブラウザー制御（リモート Gateway + ローカルブラウザー）

1 台のマシンで Gateway を実行し、別のマシンのブラウザーを操作するには、ブラウザー側のマシンで **Node ホスト** を実行し、両方を同じ tailnet に接続します。Gateway はブラウザー操作を Node にプロキシします。個別の制御サーバーや Serve URL は必要ありません。

ブラウザー制御には Funnel を使用しないでください。Node のペアリングはオペレーターアクセスと同様に扱ってください。

## 詳細情報

- Tailscale Serve の概要: [https://tailscale.com/kb/1312/serve](https://tailscale.com/kb/1312/serve)
- `tailscale serve` コマンド: [https://tailscale.com/kb/1242/tailscale-serve](https://tailscale.com/kb/1242/tailscale-serve)
- Tailscale Funnel の概要: [https://tailscale.com/kb/1223/tailscale-funnel](https://tailscale.com/kb/1223/tailscale-funnel)
- `tailscale funnel` コマンド: [https://tailscale.com/kb/1311/tailscale-funnel](https://tailscale.com/kb/1311/tailscale-funnel)

## 関連項目

- [リモートアクセス](/ja-JP/gateway/remote)
- [検出](/ja-JP/gateway/discovery)
- [認証](/ja-JP/gateway/authentication)
