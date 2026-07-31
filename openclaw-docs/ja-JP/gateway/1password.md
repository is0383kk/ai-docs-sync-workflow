---
read_when:
    - API キーを openclaw.json から削除して 1Password 内に保存したい場合
    - Gateway をヘッドレスで実行し、op 用のサービスアカウント認証が必要です
    - op CLI を使用してエージェントにシークレットを読み取らせるか、注入させたい場合
summary: 1Password CLI で Gateway のシークレットを解決し、エージェントが同梱の 1password skill を使用できるようにする
title: 1Password
x-i18n:
    generated_at: "2026-07-26T09:34:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb14944f0b3ce1ee3f90bf666a53e8673e7a9861e3e138a5fabe9c8e070cbd7
    source_path: gateway/1password.md
    workflow: 16
---

OpenClaw は、次の 3 つの独立した方法で **1Password** と連携します。

- **設定シークレット:** `openclaw.json` 内の任意の [SecretRef](/ja-JP/gateway/secrets) フィールドは、実行時に `op` CLI を介して解決できるため、API キーが設定ファイルに保存されることはありません。
- **エージェントワークフロー:** バンドルされている `1password` skill により、エージェントは自身のタスクで `op` を使用してサインインし、シークレットを読み取ったり注入したりできます。
- **ブラウザーサインイン:** `claude-cli` バックエンドは、[1Password for Claude](https://support.1password.com/1password-claude/) と Claude Code の Chrome 連携を使用できます。これにより、パスワードがモデルや OpenClaw に渡ることなく、エージェントが Web サイトにサインインできます。

## 要件

- [1Password CLI](https://developer.1password.com/docs/cli/get-started/)（`op`）が Gateway ホストにインストールされていること（macOS では `brew install 1password-cli`）。
- `op` の認証モード:
  - **サービスアカウント**（ヘッドレス Gateway に推奨）: Gateway サービス環境で `OP_SERVICE_ACCOUNT_TOKEN` をエクスポートします。デスクトップアプリも対話型サインインも不要です。
  - **デスクトップアプリ連携**: CLI 連携を有効にした 1Password アプリを同じマシンで実行します。最初の呼び出しで Touch ID またはシステム認証が要求される場合があります。
  - **スタンドアロンサインイン**: `op signin` はセッションごとに入力を要求します。エージェントは skill を介して利用できますが、ヘッドレス Gateway での設定シークレットの解決には適していません。

## op による設定シークレットの解決

`op://vault/item/field` 参照を指定して `op read` を実行する exec シークレットプロバイダーを宣言し、SecretRef 対応の任意のフィールドからそのプロバイダーを参照します。

```json5
{
  secrets: {
    providers: {
      onepassword_openai: {
        source: "exec",
        command: "/opt/homebrew/bin/op",
        allowSymlinkCommand: true, // Homebrew のシンボリックリンク形式のバイナリに必要
        trustedDirs: ["/opt/homebrew"],
        args: ["read", "op://Personal/OpenClaw QA API Key/password"],
        passEnv: ["HOME"],
        jsonOnly: false,
      },
    },
  },
  models: {
    providers: {
      openai: {
        baseUrl: "https://api.openai.com/v1",
        models: [{ id: "gpt-5", name: "gpt-5" }],
        apiKey: { source: "exec", provider: "onepassword_openai", id: "value" },
      },
    },
  },
}
```

各要素の関係は次のとおりです。

- `command` は絶対パスである必要があります。`trustedDirs` はそのディレクトリを信頼済みとして指定し、Homebrew は `op` をシンボリックリンクとしてインストールするため、`allowSymlinkCommand` が必要です。
- `args` は `op://vault/item/field` 参照をそのまま渡します。OpenClaw 自体は `op://` スキームを解析せず、`op` バイナリが解決します。
- `passEnv` は、一覧に指定された変数を Gateway 環境から転送します。デスクトップアプリ連携には `HOME` が必要です。サービスアカウントの場合は、Gateway サービス環境に `OP_SERVICE_ACCOUNT_TOKEN` も存在する必要があります（`passEnv` に追加するか、トークンが設定ファイルから読み取れることを許容する場合に限り、`env` で設定します）。
- 単一値の出力では `id: "value"` のままにします。`jsonOnly: true` と JSON ペイロードを使用する場合は、代わりに JSON ポインター ID でフィールドを指定します。
- シークレットごとにプロバイダーエントリを 1 つにすると、参照を監査しやすくなります。プロバイダーには利用側に基づいた名前を付けます（`onepassword_openai`、`onepassword_telegram`）。

解決順序、キャッシュ、失敗時の動作については [Gateway のシークレット](/ja-JP/gateway/secrets) を、SecretRef を受け付けるすべてのフィールドについては [SecretRef 認証情報サーフェス](/ja-JP/reference/secretref-credential-surface) を参照してください。

## ヘッドレス Gateway 用のサービスアカウント設定

1. 1Password アカウントでサービスアカウントを作成し、Gateway が必要とする保管庫アイテムのみに読み取りアクセス権を付与します。
2. `OP_SERVICE_ACCOUNT_TOKEN` を Gateway サービスに渡します（launchd plist、systemd ユニット、またはコンテナー環境）。
3. `"OP_SERVICE_ACCOUNT_TOKEN"` をプロバイダーの `passEnv` リストに追加します。
4. Gateway ホスト環境から確認します。`op whoami` を実行すると、入力を求められることなくサービスアカウントが表示される必要があります。

サービスアカウントで読み取るには、`op://` 参照で保管庫名を明示的に指定する必要があります。アカウントのスコープは厳密に制限してください。これはベアラー認証情報です。

## エージェント用の 1password skill

OpenClaw には、エージェントを熟練した `op` オペレーターにする `1password` skill がバンドルされています。この skill は利用可能な認証モード（サービスアカウント、デスクトップアプリ連携、またはスタンドアロンサインイン）を検出し、何かを読み取る前に `op whoami` でアクセスを確認し、シークレット値をディスクに書き込むのではなく `op run` / `op inject` を優先します。この skill には `op` バイナリが必要で、存在しない場合は Homebrew でのインストールを案内します。

エージェントは、タスクの途中でデプロイトークンを読み取ったり、コマンドに環境変数を注入したりするなど、自身のワークフローでこの skill を使用します。これは設定シークレットの解決とは独立しており、Gateway は skill を一切介さずに SecretRef を解決します。

## 1Password for Claude によるブラウザーサインイン

[1Password for Claude](https://support.1password.com/1password-claude/) を使用すると、Claude がログインを要求した際に、1Password ブラウザー拡張機能が暗号化されたチャネルを介して認証情報をページへ直接入力できます。シークレットがモデルコンテキスト、トランスクリプト、または OpenClaw に入ることはありません。OpenClaw が Claude Code の Chrome 連携を有効にして [`claude-cli` バックエンド](/ja-JP/gateway/cli-backends#claude-cli-specifics)を実行すると、実際のサインイン済みセッションを必要とする Web サイトで、エージェントタスクがこのフローを使用できます。

バックエンド自体に加えて、次のものが必要です。

- Chrome、接続済みの [Claude in Chrome extension](https://code.claude.com/docs/en/chrome)、1Password デスクトップアプリ、および 1Password ブラウザー拡張機能（いずれも 8.12.28 以降）がある macOS Gateway ホスト。
- 直接契約した Anthropic プラン（Pro、Max、Team、または Enterprise）にサインインした Claude Code。Chrome 連携は Amazon Bedrock、Google Cloud、その他のサードパーティープロバイダー経由では利用できません。
- Anthropic 側での 1Password の初回接続: 1Password for Claude は、[1Password のガイド](https://support.1password.com/1password-claude/)に記載されている Claude デスクトップアプリまたは拡張機能のフローで設定します。現在は macOS 向けベータ版です。1Password Business では、まず管理者が Policies で "Allow AI agents to autofill for users" を有効にする必要があります。また、Anthropic Team/Enterprise プランでも、Owner が有効にするまで連携は無効です。
- Claude の起動引数に `--chrome` を追加する [CLI バックエンド Plugin](/ja-JP/plugins/cli-backend-plugins)。バンドルされているバックエンドは Chrome を有効にしません。
- Gateway ホストにいる人: 認証情報を使用するたびに 1Password のプロンプトがそこに表示され、確認が必要です（たとえば Touch ID を使用）。制限の厳しい exec ポリシーでは、ブラウザーツールの呼び出し自体も、まず OpenClaw の承認としてチャネルに転送されます。

これを OpenClaw に接続する前に、Gateway ホスト上の対話型セッションで各要素を確認します。`claude --chrome` を実行し、拡張機能が接続されることを確認して、`claude-in-chrome` ツールに認証情報ツールが含まれているか確認します。そこに表示されない場合、OpenClaw 経由でも表示されません。

ワンタイムパスコードは同じページ上で 1Password が入力します。確認コードやパスワードをチャット経由で中継してはなりません。承認とブラウザーの両方が Gateway ホスト上に存在するため、現在、ヘッドレスまたはリモートの Gateway ではこのフローを使用できません。

## セキュリティに関する注意事項

- exec プロバイダーを介して解決されたシークレット値は Gateway のメモリ内に保持されます。設定スナップショットおよび `config.get` の応答では SecretRef フィールドがマスクされます。
- シークレット値を `openclaw.json`、ログ、またはチャットに含めてはなりません。設定にはアイテム名を、1Password には値を保持します。
- 1Password の監査証跡にはサービスアカウントによるすべての読み取りが記録されるため、キーのローテーションやインシデントレビューを実用的に行えます。

## トラブルシューティング

- `command not found` または生成エラー: `op` の絶対パスを使用し、そのディレクトリを `trustedDirs` に含めます。
- `op` は解決されるものの、シンボリックリンクエラーで読み取りに失敗する: Homebrew でインストールした場合は `allowSymlinkCommand: true` を設定します。
- `account is not signed in`: サービスアカウントの場合は、`OP_SERVICE_ACCOUNT_TOKEN` が Gateway サービスに渡され、`passEnv` に記載されていることを確認します。デスクトップ連携の場合は、アプリが実行中でロック解除されていることを確認します。
- 最初の読み取りが遅い場合: プロバイダーの `timeoutMs` を増やします。ビジー状態のホストでは、`op` のコールドスタートが厳しいタイムアウトを超えることがあります。
