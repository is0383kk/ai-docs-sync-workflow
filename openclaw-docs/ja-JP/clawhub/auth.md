---
read_when:
    - ClawHub へのサインイン
    - ClawHub CLI の使用方法
    - 401 エラーのデバッグ
summary: ClawHubへのサインイン、APIトークン、CLIログイン、トークンの保存、取り消し。
x-i18n:
    generated_at: "2026-07-26T09:28:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4f39be61235d71ff7a563c11a16cfd3b90562b664314c9cffd184dddd2199dbc
    source_path: clawhub/auth.md
    workflow: 16
---

# 認証

ClawHub は Web サインインに GitHub を使用します。CLI は、サインイン済みのアカウントを通じて作成された ClawHub API トークンを使用します。

## Web サインイン

[clawhub.ai](https://clawhub.ai) で GitHub を使用してサインインします。

削除、BAN、または無効化されたアカウントでは、通常の ClawHub サインインを完了できません。サインイン後にログアウト状態へ戻る場合は、アカウントが良好な状態ではない可能性があります。アカウントが BAN または無効化されており、それが誤りだと思われる場合は、[ClawHub 異議申し立てフォーム](https://appeals.openclaw.ai/)を使用してください。

## CLI ログイン

デフォルトの CLI ログインフローでは、ブラウザーが開きます。

```bash
clawhub login
clawhub whoami
```

処理の流れ：

1. CLI は `127.0.0.1` で一時的なコールバックサーバーを起動します。
2. ブラウザーで ClawHub のサインインページが開きます。
3. GitHub でのサインイン後、ClawHub が API トークンを作成します。
4. ブラウザーがローカルコールバックへリダイレクトされます。
5. CLI がトークンを ClawHub 設定ファイルに保存します。

ファイアウォール、VPN、またはプロキシのルールによりブラウザーからローカルコールバックへ接続できない場合は、ヘッドレストークンフローを使用してください。

## ヘッドレスログイン

ClawHub Web UI でトークンを作成し、CLI に渡します。

```bash
clawhub login --token clh_...
```

サーバー、CI ジョブ、またはターミナルのみの環境では、このフローを使用します。

別の場所でブラウザーを開けるリモートシェルでは、次を実行します。

```bash
clawhub login --device
```

CLI はワンタイムコードを表示し、`https://clawhub.ai/cli/device` で認可が完了するまで待機します。

## トークンの保存

デフォルトの設定パス：

- macOS：`~/Library/Application Support/clawhub/config.json`
- Linux/XDG：`$XDG_CONFIG_HOME/clawhub/config.json` または `~/.config/clawhub/config.json`
- Windows：`%APPDATA%\\clawhub\\config.json`

次のようにパスを上書きできます。

```bash
export CLAWHUB_CONFIG_PATH=/path/to/config.json
```

CI のセットアップ用に、保存されたトークンを表示するには次を実行します。

```bash
clawhub token
```

## 失効

ClawHub Web UI で API トークンを失効させることができます。

失効済み、無効、または欠落しているトークンは `401 Unauthorized` を返します。`clawhub login` で再度サインインするか、`clawhub login --token` で新しいトークンを指定してください。

削除、BAN、または無効化されたアカウントでは、既存の API トークンを引き続き使用できません。アカウントが BAN または無効化されており、それが誤りだと思われる場合は、[ClawHub 異議申し立てフォーム](https://appeals.openclaw.ai/)を使用してください。
