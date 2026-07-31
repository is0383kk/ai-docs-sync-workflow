---
read_when:
    - 現在のトークンを使用して Control UI を開く場合
    - ブラウザを起動せずに URL を表示したい場合
summary: '`openclaw dashboard` の CLI リファレンス（Control UI を開く）'
title: ダッシュボード
x-i18n:
    generated_at: "2026-07-26T09:15:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 168605e1e58827020b4d247afd513880335273e489995549377bc2dc1f8a3b25
    source_path: cli/dashboard.md
    workflow: 16
---

# `openclaw dashboard`

現在の認証を使用して Control UI を開きます。

```bash
openclaw dashboard
openclaw dashboard --no-open
openclaw dashboard --json
openclaw dashboard --yes
```

- `--no-open`: URL を出力しますが、ブラウザは起動しません。
- `--json`: ブラウザを開いたり、クリップボードを使用したり、プロンプトを表示したり、Gateway を起動したりせずに、機械可読な接続オブジェクトを 1 つ出力します。
- `--yes`: 必要な場合、プロンプトを表示せずに Gateway を起動またはインストールします。

## 機械可読出力

解決済みの Control UI URL が必要なデスクトップ統合やスクリプトでは、`--json` を使用します。

```bash
openclaw dashboard --json
```

レスポンスには `url`、`httpUrl`、`wsUrl`、`port`、および `tokenIncluded` が含まれます。Gateway の準備ができていない場合、コマンドは `{"ok":false,"reason":"..."}` を返し、ゼロ以外の終了コードで終了します。SecretRef で管理されるトークンが `url` に含まれることはありません。

注:

- 可能な場合、設定された `gateway.auth.token` SecretRef を解決します。
- `gateway.tls.enabled` に従います。TLS が有効な Gateway は `https://` Control UI URL を出力または開き、`wss://` 経由で接続します。
- `lan` またはワイルドカードの `custom` バインドでは、ワイルドカードはブラウザの接続先ではないため、同一ホストからの起動には常にループバックを使用します。平文の `tailnet` および `custom` バインドでも、ブラウザにセキュアコンテキストを提供するため `127.0.0.1` を使用します。TLS が有効な特定ホストでは、証明書名が一致するように設定済みアドレスを維持します。
- 特定インターフェースへのバインドに対して認証済みループバック URL を提供する前に、コマンドは設定済みインターフェースをプローブし、そのインターフェースと `127.0.0.1` が同じ Gateway プロセスによって所有されていることを検証します。リスナーの所有者が曖昧な場合は安全側に失敗し、ステータス確認の案内を表示します。
- SecretRef で管理されるトークンでは、解決済みか未解決かにかかわらず、出力、コピー、またはブラウザで開かれる URL にトークンが含まれることはありません。そのため、外部シークレットがターミナル出力、クリップボード履歴、またはブラウザ起動引数に漏れることはありません。
- `gateway.auth.token` が SecretRef で管理されているものの未解決の場合、コマンドは無効なトークンプレースホルダーの代わりに、トークンを含まない URL と修復手順を出力します。
- トークン認証済み URL のクリップボードまたはブラウザへの受け渡しに失敗した場合、コマンドはトークン値を出力せずに、`OPENCLAW_GATEWAY_TOKEN`、`gateway.auth.token`、および URL フラグメントキー `token` を示す安全な手動認証のヒントをログに記録します。

## 関連項目

- [CLI リファレンス](/ja-JP/cli)
- [ダッシュボード](/ja-JP/web/dashboard)
