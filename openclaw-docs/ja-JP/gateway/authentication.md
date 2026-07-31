---
read_when:
    - モデル認証または OAuth の有効期限切れのデバッグ
    - 認証または認証情報の保存に関するドキュメント作成
summary: モデル認証：OAuth、API キー、Claude CLI の再利用、Anthropic セットアップトークン
title: 認証
x-i18n:
    generated_at: "2026-07-26T09:59:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fd4bf1c73f41d297638811f568c1b11e920eba3bd1527206cbb760df51531f2
    source_path: gateway/authentication.md
    workflow: 16
---

<Note>
このページでは、**モデルプロバイダー**の認証（API キー、OAuth、Claude CLI の再利用、Anthropic セットアップトークン）について説明します。**Gateway 接続**の認証（トークン、パスワード、信頼済みプロキシ）については、[設定](/ja-JP/gateway/configuration)および[信頼済みプロキシ認証](/ja-JP/gateway/trusted-proxy-auth)を参照してください。
</Note>

OpenClaw は、モデルプロバイダーの OAuth と API キーをサポートしています。常時稼働する Gateway ホストでは、API キーが最も予測可能な選択肢です。サブスクリプション/OAuth フローも、プロバイダーアカウントのモデルに合致する場合は利用できます。

- OAuth フローとストレージレイアウトの全容: [/concepts/oauth](/ja-JP/concepts/oauth)
- SecretRef ベースの認証（`env`/`file`/`exec` プロバイダー）: [シークレット管理](/ja-JP/gateway/secrets)
- `models status --probe` が使用する認証情報の適格性/理由コード: [認証情報のセマンティクス](/ja-JP/auth-credential-semantics)

## 推奨セットアップ: API キー（任意のプロバイダー）

1. プロバイダーのコンソールで API キーを作成します。
2. API キーを **Gateway ホスト**（`openclaw gateway` を実行しているマシン）に配置します。

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. Gateway が systemd/launchd で実行されている場合は、デーモンが読み取れるようにキーを `~/.openclaw/.env` に配置します。

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

4. Gateway プロセス（またはデーモン）を再起動し、再確認します。

```bash
openclaw models status
openclaw doctor
```

環境変数を自分で管理したくない場合は、`openclaw onboard` でデーモン用の API キーを保存することもできます。環境変数の読み込み優先順位（`env.shellEnv`、`~/.openclaw/.env`、systemd/launchd）の全容については、[環境変数](/ja-JP/help/environment)を参照してください。

## Anthropic: Claude CLI の再利用

Anthropic セットアップトークン認証は、引き続きサポートされる方法です。この連携では、Claude CLI の再利用（`claude -p` 形式の使用）も正式に認められています。ホストで Claude CLI ログインを利用できる場合、ローカル/デスクトップでの使用にはそれが推奨されます。長期間稼働する Gateway ホストでは、サーバー側の課金を明示的に制御できる Anthropic API キーが、依然として最も予測可能な選択肢です。

Claude CLI を再利用するためのホスト設定:

```bash
# Gateway ホストで実行
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

これは 2 段階の手順です。まずホスト上の Claude Code を Anthropic にログインさせ、次に Anthropic モデルの選択をローカルの `claude-cli` バックエンド経由でルーティングし、対応する OpenClaw 認証プロファイルを保存するよう OpenClaw に指示します。

Gateway サービスは、`PATH` 上で `claude` を解決できる必要があります。デプロイで標準外の実行ファイルパスが必要な場合は、
[CLI バックエンド Plugin](/ja-JP/plugins/cli-backend-plugins)を介してラッパーを登録してください。

## トークンの手動入力

任意のプロバイダーで使用でき、エージェントごとの SQLite 認証ストアに書き込み、設定を更新します。

```bash
openclaw models auth paste-token --provider openrouter
```

OpenClaw は、各エージェントの `openclaw-agent.sqlite` から認証プロファイルを読み取ります。エンドポイントの詳細（`baseUrl`、`api`、モデル ID、ヘッダー、タイムアウト）は認証プロファイルではなく、`openclaw.json` または `models.json` の `models.providers.<id>` に含めます。

古いインストールに `auth-profiles.json`、`auth-state.json`、または `{ "openrouter": { "apiKey": "..." } }` のようなフラットな形式がまだ存在する場合は、`openclaw doctor --fix` を実行して SQLite にインポートしてください。doctor は、元の JSON ファイルの隣にタイムスタンプ付きのバックアップを保持します。

Bedrock の `auth: "aws-sdk"` のような外部認証ルートは、認証情報ではありません。名前付き Bedrock ルートでは、`openclaw.json` に `auth.profiles.<id>.mode: "aws-sdk"` を設定してください。認証プロファイルストアに `type: "aws-sdk"` を書き込まないでください。`openclaw doctor --fix` は、従来の AWS SDK マーカーを認証情報ストアから設定メタデータへ移行します。

### SecretRef ベースの認証情報

- `api_key` 認証情報では `keyRef: { source, provider, id }` を使用できます
- `token` 認証情報では `tokenRef: { source, provider, id }` を使用できます
- OAuth モードのプロファイルは SecretRef 認証情報を拒否します。`auth.profiles.<id>.mode` が `"oauth"` の場合、そのプロファイルの SecretRef ベースの `keyRef`/`tokenRef` は拒否されます。

## モデル認証状態の確認

```bash
openclaw models status
openclaw doctor
```

自動化向けのチェックでは、期限切れ/欠落時に終了コード `1`、期限切れが近い場合に `2` を返します。

```bash
openclaw models status --check
```

ライブ認証プローブ（範囲を絞り込むには `--probe-provider`、`--probe-profile`、`--probe-timeout`、`--probe-concurrency`、または `--probe-max-tokens` を追加）:

```bash
openclaw models status --probe
```

注:

- プローブ行は、認証プロファイル、環境変数の認証情報、または `models.json` から生成されることがあります。
- `auth.order.<provider>` で保存済みプロファイルが省略されている場合、プローブはそのプロファイルを試行せず、`excluded_by_auth_order` を報告します。
- 認証情報が存在していても、そのプロバイダーでプローブ可能なモデルを OpenClaw が解決できない場合、プローブは `status: no_model` を報告します。
- レート制限のクールダウンは、モデル単位になる場合があります。あるモデルでクールダウン中のプロファイルでも、同じプロバイダー上の兄弟モデルには引き続き使用できます。

任意の運用スクリプト（systemd/Termux）: [認証監視スクリプト](/ja-JP/help/scripts#auth-monitoring-scripts)。

## API キーのローテーション（Gateway）

一部のプロバイダーでは、呼び出しがプロバイダーのレート制限に達すると、設定された別のキーを使用してリクエストを再試行します。

プロバイダーごとのキーの優先順位:

1. `OPENCLAW_LIVE_<PROVIDER>_KEY`（単一の上書き。1 つのキーに固定）
2. `<PROVIDER>_API_KEYS`（カンマ/空白/セミコロン区切りのリスト）
3. `<PROVIDER>_API_KEY`
4. `<PROVIDER>_API_KEY_*`（このプレフィックスを持つ任意の環境変数）

Google プロバイダー（`google`、`google-vertex`）は、さらに `GOOGLE_API_KEY` にフォールバックします。結合されたリストは、使用前に重複が排除されます。

OpenClaw は、エラーメッセージが `rate_limit`、`rate limit`、`429`、`quota exceeded`/`quota_exceeded`、`resource exhausted`/`resource_exhausted`、または `too many requests` に一致する場合にのみ、次のキーへローテーションします。その他のエラーでは、別のキーによる再試行は行われません。すべてのキーが失敗した場合は、最後の試行で発生した最終エラーが返されます。

<Note>
`ThrottlingException`、`concurrency limit reached`、`workers_ai ... quota limit exceeded` のようなプロバイダー固有のフレーズは、**フェイルオーバー/再試行の分類**（失敗が繰り返された場合のモデルまたはプロバイダーの切り替え）に使用されます。これは、前述の API キーローテーションとは別のメカニズムです。
</Note>

保存済みの認証情報を削除しても、プロバイダー側のキーは無効化されません。プロバイダー側で無効化する必要がある場合は、プロバイダーのダッシュボードでキーをローテーションまたは無効化してください。

## Gateway の稼働中にプロバイダー認証を削除する

Gateway コントロールプレーンを介してプロバイダー認証を削除すると、OpenClaw はそのプロバイダーの保存済み認証プロファイルを削除し、選択されたモデルプロバイダーが削除対象と一致するアクティブなチャット/エージェント実行を中止します。中止された実行では、`stopReason: "auth-revoked"` を伴う通常のキャンセル/ライフサイクルイベントが発行されるため、接続中のクライアントは、認証情報が削除されたために実行が停止したことを表示できます。

## 使用する認証情報の制御

### OpenAI と従来の `openai-codex` ID

OpenAI API キープロファイルと ChatGPT/Codex OAuth プロファイルは、どちらも正規プロバイダー ID `openai` を使用します。新しい設定には `openai:*` プロファイル ID と `auth.order.openai` を使用してください。

古い設定、認証プロファイル ID、または `auth.order.openai-codex` に `openai-codex` がある場合は、従来の移行入力として扱ってください。新しい `openai-codex` プロファイルを作成しないでください。次を実行します。

```bash
openclaw doctor --fix
openclaw models auth list --provider openai
```

doctor は、従来の `openai-codex:*` プロファイル ID と `auth.order.openai-codex` エントリを、正規の `openai` ルートに書き換えます。OpenAI 固有のモデル/ランタイムルーティングについては、[OpenAI](/ja-JP/providers/openai)を参照してください。

### ログイン時（CLI）

```bash
openclaw models auth login --provider openai --profile-id openai:ritsuko
openclaw models auth login --provider openai --profile-id openai:lain
```

`--profile-id` は、同じプロバイダーに対する複数の OAuth ログインを、1 つのエージェント内で個別に保持します。

`--force` は、選択されたエージェントディレクトリにあるそのプロバイダーの保存済み認証プロファイルを削除し、同じ認証フローを再実行します。保存済みプロファイルが停止状態、期限切れ、または誤ったアカウントに関連付けられている場合に使用してください。プロバイダー側の認証情報は無効化されません。

```bash
openclaw models auth login --provider anthropic --force
```

### セッション単位（チャットコマンド）

- `/model <alias-or-id>@<profileId>` は、現在のセッションに特定のプロバイダー認証情報を固定します（プロファイル ID の例: `anthropic:default`、`anthropic:work`）。
- `/model`（または `/model list`）はコンパクトな選択画面を表示し、`/model status` は完全なビュー（候補と次の認証プロファイル、および設定されている場合はプロバイダーのエンドポイント詳細）を表示します。

すでに実行中のチャットについて認証順序またはプロファイル固定を変更した場合は、`/new` または `/reset` を送信して新しいセッションを開始してください。既存のセッションでは、リセットされるまで現在のモデル/プロファイル選択が維持されます。

### エージェント単位（CLI 上書き）

認証順序の上書きは、そのエージェントの SQLite 認証状態に保存されます。

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

特定のエージェントを対象にするには `--agent <id>` を使用します。省略すると、設定済みのデフォルトエージェントが使用されます。`openclaw models status --probe` は、省略された保存済みプロファイルを暗黙的にスキップせず、`excluded_by_auth_order` として表示します。

## トラブルシューティング

### 「認証情報が見つかりません」

**Gateway ホスト**で Anthropic API キーを設定するか、Anthropic セットアップトークンの方法をセットアップしてから、再確認します。

```bash
openclaw models status
```

### トークンの期限切れが近い/期限切れ

どのプロファイルが期限切れ間近かを確認するには、`openclaw models status` を実行します。Anthropic トークンプロファイルが欠落しているか期限切れの場合は、セットアップトークンで更新するか、Anthropic API キーへ移行してください。

## 関連項目

- [シークレット管理](/ja-JP/gateway/secrets)
- [リモートアクセス](/ja-JP/gateway/remote)
- [認証ストレージ](/ja-JP/concepts/oauth)
