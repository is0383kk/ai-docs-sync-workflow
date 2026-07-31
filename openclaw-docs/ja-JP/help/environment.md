---
read_when:
    - どの環境変数がどの順序で読み込まれるかを把握する必要があります
    - Gateway で API キーが見つからない問題をデバッグしています
    - プロバイダー認証またはデプロイ環境について文書化している場合
summary: OpenClaw が環境変数を読み込む場所と優先順位
title: 環境変数
x-i18n:
    generated_at: "2026-07-26T09:37:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: db9990dea5df7731e54c8d442f4704bd4d6e0caf6f2c2fdea32d2583cd41128c
    source_path: help/environment.md
    workflow: 16
---

OpenClaw は複数のソースから環境変数を読み込みます。ルールは **既存の値を決して上書きしない** ことです。
ワークスペースの `.env` ファイルは信頼度の低いソースです。OpenClaw は優先順位を適用する前に、ワークスペースの `.env` に含まれるプロバイダー認証情報と保護されたランタイム制御を無視します。

## 優先順位（高い順）

1. **プロセス環境**（親シェルまたはデーモンから Gateway プロセスがすでに継承している環境）。
2. **現在の作業ディレクトリにある `.env`**（dotenv のデフォルト。上書きしません。プロバイダー認証情報と保護されたランタイム制御は無視されます）。
3. `~/.openclaw/.env` にある **グローバル `.env`**（別名 `$OPENCLAW_STATE_DIR/.env`。プロバイダー API キーに推奨。上書きしません）。
4. `~/.openclaw/openclaw.json` 内の **設定 `env` ブロック**（値がない場合にのみ適用）。
5. **オプションのログインシェルからのインポート**（`env.shellEnv.enabled` または `OPENCLAW_LOAD_SHELL_ENV=1`）。想定されるキーがない場合にのみ適用されます。

デフォルトの状態ディレクトリを使用する Ubuntu の新規インストールでは、OpenClaw はグローバル `.env` の後に、`~/.config/openclaw/gateway.env` も互換性フォールバックとして扱います。両方のファイルが存在し、内容が異なる場合、OpenClaw は `~/.openclaw/.env` を維持して警告を表示します。

設定ファイル自体が存在しない場合、手順 4 はスキップされます。シェルからのインポートは、有効になっていれば引き続き実行されます。

## オペレーター向けにサポートされる変数

以下の変数は、オペレーター向けにサポートされる環境変数の契約です。文書化されていない `OPENCLAW_*` 変数は内部実装の詳細であり、予告なく削除される可能性があります。

### パスとインスタンス

| 変数                 | 用途                                                           |
| ------------------------ | ----------------------------------------------------------------- |
| `OPENCLAW_HOME`          | OpenClaw のパスのデフォルトに使用するホームディレクトリを上書きします。      |
| `OPENCLAW_STATE_DIR`     | 変更可能な状態ディレクトリを上書きします。                             |
| `OPENCLAW_CONFIG_PATH`   | アクティブな設定ファイルのパスを上書きします。                             |
| `OPENCLAW_WORKSPACE_DIR` | デフォルトのエージェントワークスペースを上書きします。                             |
| `OPENCLAW_PROFILE`       | 名前付きプロファイルと、その分離されたデフォルトを選択します。                 |
| `OPENCLAW_GIT_DIR`       | 開発チャンネルの更新で使用するソースチェックアウトを上書きします。 |
| `OPENCLAW_INCLUDE_ROOTS` | `$include` が追加のルートから解決されることを許可します。                |

### Gateway と認証

| 変数                    | 用途                                                         |
| --------------------------- | --------------------------------------------------------------- |
| `OPENCLAW_GATEWAY_URL`      | クライアントが使用するリモート Gateway URL を上書きします。                |
| `OPENCLAW_GATEWAY_PORT`     | ローカル Gateway ポートを上書きします。                                |
| `OPENCLAW_GATEWAY_TOKEN`    | Gateway サーバーとクライアントにトークン認証を提供します。    |
| `OPENCLAW_GATEWAY_PASSWORD` | Gateway サーバーとクライアントにパスワード認証を提供します。 |

### プロバイダー認証情報

コアおよびバンドルされたプロバイダー Plugin は、以下の認証情報変数とプロバイダー選択変数を認識します。プロセス全体で共有される単一の値ではなく、スコープを限定した認証情報が必要な場合は、各プロバイダーの設定フィールドまたは SecretRef フィールドを優先してください。

`AI_GATEWAY_API_KEY`, `ANTHROPIC_ADMIN_API_KEY`, `ANTHROPIC_ADMIN_KEY`, `ANTHROPIC_API_KEY`, `ANTHROPIC_OAUTH_TOKEN`, `ARCEEAI_API_KEY`, `AZURE_OPENAI_API_KEY`, `AZURE_SPEECH_API_KEY`, `AZURE_SPEECH_KEY`, `AZURE_SPEECH_REGION`, `BASETEN_API_KEY`, `BRAVE_API_KEY`, `BYTEPLUS_API_KEY`, `BYTEPLUS_SEED_SPEECH_API_KEY`, `CEREBRAS_API_KEY`, `CHUTES_API_KEY`, `CHUTES_OAUTH_TOKEN`, `CLAWROUTER_API_KEY`, `CLOUDFLARE_AI_GATEWAY_API_KEY`, `CODEX_API_KEY`, `COHERE_API_KEY`, `COMFY_API_KEY`, `COMFY_CLOUD_API_KEY`, `COPILOT_GITHUB_TOKEN`, `DASHSCOPE_API_KEY`, `DEEPGRAM_API_KEY`, `DEEPINFRA_API_KEY`, `DEEPSEEK_API_KEY`, `ELEVENLABS_API_KEY`, `EXA_API_KEY`, `FAL_API_KEY`, `FAL_KEY`, `FEATHERLESS_API_KEY`, `FIRECRAWL_API_KEY`, `FIREWORKS_API_KEY`, `GCLOUD_PROJECT`, `GEMINI_API_KEY`, `GH_TOKEN`, `GITHUB_TOKEN`, `GMI_API_KEY`, `GOOGLE_API_KEY`, `GOOGLE_APPLICATION_CREDENTIALS`, `GOOGLE_CLOUD_API_KEY`, `GOOGLE_CLOUD_LOCATION`, `GOOGLE_CLOUD_PROJECT`, `GRADIUM_API_KEY`, `GROQ_API_KEY`, `HF_TOKEN`, `HUGGINGFACE_HUB_TOKEN`, `INWORLD_API_KEY`, `KILOCODE_API_KEY`, `KIMICODE_API_KEY`, `KIMI_API_KEY`, `LITELLM_API_KEY`, `LM_API_TOKEN`, `LONGCAT_API_KEY`, `MINIMAX_API_KEY`, `MINIMAX_CODE_PLAN_KEY`, `MINIMAX_CODING_API_KEY`, `MINIMAX_OAUTH_TOKEN`, `MISTRAL_API_KEY`, `MODELSTUDIO_API_KEY`, `MODEL_API_KEY`, `MOONSHOT_API_KEY`, `NOVITA_API_KEY`, `NVIDIA_API_KEY`, `OLLAMA_API_KEY`, `OPENAI_ADMIN_KEY`, `OPENAI_API_KEY`, `OPENCODE_API_KEY`, `OPENCODE_ZEN_API_KEY`, `OPENROUTER_API_KEY`, `PARALLEL_API_KEY`, `PERPLEXITY_API_KEY`, `PIXVERSE_API_KEY`, `QIANFAN_API_KEY`, `QWEN_API_KEY`, `QWEN_TOKEN_PLAN_API_KEY`, `RUNWAYML_API_SECRET`, `RUNWAY_API_KEY`, `SENSEAUDIO_API_KEY`, `SGLANG_API_KEY`, `SPEECH_KEY`, `SPEECH_REGION`, `STEPFUN_API_KEY`, `SYNTHETIC_API_KEY`, `TAVILY_API_KEY`, `TOGETHER_API_KEY`, `TOKENHUB_API_KEY`, `TOKENPLAN_API_KEY`, `VENICE_API_KEY`, `VLLM_API_KEY`, `VOLCANO_ENGINE_API_KEY`, `VOLCENGINE_TTS_API_KEY`, `VOLCENGINE_TTS_APPID`, `VOLCENGINE_TTS_TOKEN`, `VOYAGE_API_KEY`, `VYDRA_API_KEY`, `XAI_API_KEY`, `XIAOMI_API_KEY`, `XIAOMI_TOKEN_PLAN_API_KEY`, `XI_API_KEY`, `ZAI_API_KEY`, および `Z_AI_API_KEY`。

インストールされたサードパーティ Plugin は、Plugin マニフェストで追加の認証情報変数を宣言できます。それらの変数は宣言元 Plugin の契約であり、OpenClaw コアの変数ではありません。

### ログと診断

| 変数                             | 用途                                                       |
| ------------------------------------ | ------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`                 | ファイルおよびコンソールのログレベルを上書きします。                         |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT`     | モデルトランスポートのタイミング診断を有効にします。                    |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`       | 編集済みモデルペイロードの診断を選択します。                    |
| `OPENCLAW_DEBUG_SSE`                 | SSE タイミングまたはイベント確認の診断を選択します。                  |
| `OPENCLAW_DEBUG_CODE_MODE`           | コードモードのサーフェス診断を有効にします。                         |
| `OPENCLAW_DIAGNOSTICS`               | 名前付き診断フラグを有効にするか、`0` ですべてのフラグを無効にします。 |
| `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH` | タイムライン診断用の JSONL パスを選択します。               |
| `OPENCLAW_DIAGNOSTICS_EVENT_LOOP`    | タイムライン診断にイベントループのサンプルを追加します。               |

### 機能とランタイムの切り替え

| 変数                             | 用途                                                                      |
| ------------------------------------ | ---------------------------------------------------------------------------- |
| `OPENCLAW_LOAD_SHELL_ENV`            | ログインシェルから、存在しない想定環境変数をインポートします。                      |
| `OPENCLAW_SHELL_ENV_TIMEOUT_MS`      | ログインシェルからのインポートのタイムアウトを設定します。                                          |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT`       | `0` で exec シェルスナップショットを無効にします。                                       |
| `OPENCLAW_OFFLINE`                   | 固定バージョンのエージェント補助バイナリのダウンロードを禁止します。                           |
| `OPENCLAW_BROWSER_HEADLESS`          | 管理対象ブラウザーの起動をヘッド付き（`0`）またはヘッドレス（`1`）に強制します。               |
| `OPENCLAW_DISABLE_BONJOUR`           | Bonjour 広告をオン（`0`）またはオフ（`1`）に強制します。                             |
| `OPENCLAW_NO_AUTO_UPDATE`            | 更新の自動適用を無効にします。                                            |
| `OPENCLAW_ALLOW_INSECURE_PRIVATE_WS` | 緊急時のオーバーライドとして、信頼されたプライベート DNS の `ws://` 接続を許可します。     |
| `OPENCLAW_ALLOW_MULTI_GATEWAY`       | 状態ごとの所有権ロックを維持しながら、複数の Gateway プロセスを許可します。 |
| `OPENCLAW_SKIP_CHANNELS`             | トラブルシューティングのため、チャンネルトランスポートなしで Gateway を起動します。            |
| `OPENCLAW_THEME`                     | TUI パレットを `light` または `dark` に強制します。                                  |

## プロバイダー認証情報とワークスペースの `.env`

プロバイダー API キーをワークスペースの `.env` だけに保存しないでください。OpenClaw は、既知のすべてのプロバイダー認証環境変数（例: `GEMINI_API_KEY`、`GOOGLE_API_KEY`、`XAI_API_KEY`、`MISTRAL_API_KEY`、`GROQ_API_KEY`、`DEEPSEEK_API_KEY`、`PERPLEXITY_API_KEY`、`BRAVE_API_KEY`、`TAVILY_API_KEY`、`EXA_API_KEY`、`FIRECRAWL_API_KEY`）に加え、`_API_HOST`、`_BASE_URL`、`_ENDPOINT`、または `_HOMESERVER` で終わるすべてのキー、および `OPENCLAW_*`、`CLAWHUB_*`、`ANTHROPIC_API_KEY_*`、`OPENAI_API_KEY_*` の各名前空間全体を含む、多数のプロバイダー認証情報キーとエンドポイントリダイレクトキーをワークスペースの `.env` ファイルから読み込まないようにしています。

代わりに、プロバイダー認証情報には次のいずれかの信頼されたソースを使用してください。

- シェル、launchd/systemd ユニット、コンテナシークレット、CI シークレットなどの Gateway プロセス環境。
- `~/.openclaw/.env` または `$OPENCLAW_STATE_DIR/.env` にあるグローバルランタイム dotenv ファイル。
- `~/.openclaw/openclaw.json` 内の設定 `env` ブロック。
- `env.shellEnv.enabled` または `OPENCLAW_LOAD_SHELL_ENV=1` が有効な場合の、オプションのログインシェルからのインポート。

以前にプロバイダーキーやエンドポイントルーティング値をワークスペースの `.env` だけに保存していた場合は、上記の信頼されたソースのいずれかに移動してください。ワークスペースの `.env` では、認証情報、エンドポイントリダイレクト、ホストの上書き、または `OPENCLAW_*` ランタイム制御ではない通常のプロジェクト変数を引き続き提供できます。

セキュリティ上の理由については、[ワークスペースの `.env` ファイル](/ja-JP/gateway/security#workspace-env-files)を参照してください。

## 設定 `env` ブロック

インライン環境変数を設定する、同等の 2 つの方法があります（どちらも上書きしません）。

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
  },
}
```

設定 `env` ブロックは、リテラル文字列値のみを受け付けます。
`file:...` の値は展開されません。たとえば、`XAI_API_KEY: "file:secrets/xai-api-key.txt"`
は、そのままの文字列としてプロバイダーに渡されます。

ファイルに保存されたプロバイダーキーには、それをサポートする認証情報フィールドで SecretRef を使用してください。

```json5
{
  secrets: {
    providers: {
      xai_key_file: {
        source: "file",
        path: "~/.openclaw/secrets/xai-api-key.txt",
        mode: "singleValue",
      },
    },
  },
  models: {
    providers: {
      xai: {
        apiKey: { source: "file", provider: "xai_key_file", id: "value" },
      },
    },
  },
}
```

サポートされるフィールドについては、[シークレット管理](/ja-JP/gateway/secrets)および
[SecretRef 認証情報サーフェス](/ja-JP/reference/secretref-credential-surface)を
参照してください。

## シェル環境のインポート

`env.shellEnv` はログインシェルを実行し、存在しない想定キーのみをインポートします。

```json5
{
  env: {
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

同等の環境変数:

- `OPENCLAW_LOAD_SHELL_ENV=1`
- `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`（デフォルト `15000`）

## exec シェルスナップショット

Windows 以外の Gateway ホストでは、bash および zsh の `exec` コマンドはデフォルトで起動時スナップショットを使用します。
このパスを無効にするには、Gateway プロセス環境に `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` を設定します。
値 `false`、`no`、および `off` でも無効になります。呼び出しごとの `exec.env` 値では、
スナップショットの切り替えやスナップショットキャッシュのリダイレクトはできません。

## ランタイムによって注入される環境変数

OpenClaw は、生成した子プロセスにコンテキストマーカーも注入します。

- `OPENCLAW_SHELL=exec`: `exec` ツールを通じて実行されるコマンドに設定されます。
- `OPENCLAW_SHELL=acp-client`: ACP ブリッジプロセスを生成する際に `openclaw acp client` に設定されます。
- `OPENCLAW_SHELL=tui-local`: ローカル TUI の `!` シェルコマンドに設定されます。
- `OPENCLAW_CLI=1`: CLI エントリポイントによって生成される子プロセスに設定されます。

これらはランタイムマーカーです（ユーザー設定には必須ではありません）。シェルやプロファイルのロジックで使用して、
コンテキスト固有のルールを適用できます。

## UI 環境変数

- `OPENCLAW_THEME=light`: 端末の背景が明るい場合に、TUI の明るいパレットを強制します。
- `OPENCLAW_THEME=dark`: TUI の暗いパレットを強制します。
- `COLORFGBG`: 端末がこれをエクスポートしている場合、OpenClaw は背景色のヒントを使用して TUI パレットを自動選択します。

## 設定内での環境変数の置換

`${VAR_NAME}` 構文を使用すると、設定の文字列値で環境変数を直接参照できます。

```json5
{
  models: {
    providers: {
      "vercel-gateway": {
        apiKey: "${VERCEL_GATEWAY_API_KEY}",
      },
    },
  },
}
```

詳細については、[設定：環境変数の置換](/ja-JP/gateway/configuration-reference#env-var-substitution)を参照してください。

## シークレット参照と `${ENV}` 文字列の違い

OpenClaw は、環境変数を利用する 2 つのパターンをサポートしています。

- 設定値内での `${VAR}` 文字列置換。
- シークレット参照をサポートするフィールド用の SecretRef オブジェクト（`{ source: "env", provider: "default", id: "VAR" }`）。

どちらも有効化時にプロセスの環境変数から解決されます。SecretRef の詳細については、[シークレット管理](/ja-JP/gateway/secrets)を参照してください。
設定の `env` ブロック自体は、SecretRef や `file:...`
短縮値を解決しません。

## パス関連の環境変数

| 変数                 | 用途                                                                                                                                                                                                                                 |
| ------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_HOME`          | OpenClaw の内部パスのデフォルト（`~/.openclaw/`、エージェントディレクトリ、セッション、認証情報、インストーラーのオンボーディング、デフォルトの開発用チェックアウト）に使用されるホームディレクトリを上書きします。OpenClaw を専用サービスユーザーとして実行する場合に便利です。 |
| `OPENCLAW_STATE_DIR`     | 状態ディレクトリ（デフォルトは `~/.openclaw`）を上書きします。                                                                                                                                                                                   |
| `OPENCLAW_CONFIG_PATH`   | 設定ファイルのパス（デフォルトは `~/.openclaw/openclaw.json`）を上書きします。                                                                                                                                                                    |
| `OPENCLAW_INCLUDE_ROOTS` | `$include` ディレクティブが設定ディレクトリ外のファイルを解決できるディレクトリのパスリストです（デフォルト：なし — `$include` は設定ディレクトリ内に制限されます）。チルダが展開されます。                                                         |

## エージェント補助ツールのダウンロード

OpenClaw が固定バージョンの `fd`
および `ripgrep` 補助バイナリをダウンロードしないようにするには、`OPENCLAW_OFFLINE=1` を設定します。OpenClaw のツール
ディレクトリにある既存の補助ツールと、動作するシステムバイナリは引き続き使用できます。補助ツールが存在しない場合は、
ネットワークリクエストを開始せず、利用不可のままとなります。

## ログ

| 変数                         | 用途                                                                                                                                                                                      |
| -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_LOG_LEVEL`             | ファイルとコンソールの両方のログレベル（例：`debug`、`trace`）を上書きします。設定内の `logging.level` および `logging.consoleLevel` より優先されます。無効な値は警告とともに無視されます。 |
| `OPENCLAW_DEBUG_MODEL_TRANSPORT` | グローバルデバッグログを有効にせず、`info` レベルで対象を限定したモデルのリクエスト／レスポンスのタイミング診断を出力します。                                                                                  |
| `OPENCLAW_DEBUG_MODEL_PAYLOAD`   | モデルペイロード診断：`summary`、`tools`、または `full-redacted`。`full-redacted` は上限が設定され、編集処理されますが、プロンプトやメッセージのテキストを含む場合があります。                                               |
| `OPENCLAW_DEBUG_SSE`             | ストリーミング診断：開始／完了のタイミングには `events`、編集処理された最初の 5 件の SSE イベントを含めるには `peek`。                                                                                 |
| `OPENCLAW_DEBUG_CODE_MODE`       | プロバイダーツールの非表示や、コンパクトな制御／直接実行の強制を含む、コードモードのモデルサーフェス診断。                                                                                  |

### `OPENCLAW_HOME`

設定すると、OpenClaw の内部パスのデフォルトでは、`OPENCLAW_HOME` がシステムのホームディレクトリ（`$HOME` / `os.homedir()`）を置き換えます。これには、デフォルトの状態ディレクトリ、設定パス、エージェントディレクトリ、認証情報、インストーラーのオンボーディング用ワークスペース、`openclaw update --channel dev` で使用されるデフォルトの開発用チェックアウトが含まれます。

**優先順位：** `OPENCLAW_HOME` > `$HOME` > `USERPROFILE` > Android 上の Termux `PREFIX` ホームフォールバック > `os.homedir()`

**例**（macOS LaunchDaemon）：

```xml
<key>EnvironmentVariables</key>
<dict>
  <key>OPENCLAW_HOME</key>
  <string>/Users/user</string>
</dict>
```

`OPENCLAW_HOME` にはチルダパス（例：`~/svc`）も設定でき、使用前に同じ OS ホームフォールバックチェーンを使って展開されます。

`OPENCLAW_STATE_DIR`、`OPENCLAW_CONFIG_PATH`、`OPENCLAW_GIT_DIR` などの明示的なパス変数が引き続き優先されます。シェル起動ファイルの検出、パッケージマネージャーのセットアップ、ホストの `~` 展開など、OS アカウントに関する処理では、実際のシステムホームが引き続き使用される場合があります。

## nvm ユーザー：web_fetch の TLS エラー

Node.js がシステムのパッケージマネージャーではなく **nvm** 経由でインストールされている場合、組み込みの `fetch()` は
nvm に同梱された CA ストアを使用します。このストアには、最新のルート CA（Let's Encrypt の ISRG Root X1/X2、
DigiCert Global Root G2 など）が含まれていない場合があります。その結果、ほとんどの HTTPS サイトで `web_fetch` が `"fetch failed"` により失敗します。

Linux では、OpenClaw が nvm を自動検出し、実際の起動環境に修正を適用します。

- `openclaw gateway install` は、systemd サービス環境に `NODE_EXTRA_CA_CERTS` を書き込みます
- `openclaw` CLI エントリポイントは、Node の起動前に `NODE_EXTRA_CA_CERTS` を設定して自身を再実行します

**手動修正（古いバージョンまたは `node ...` の直接起動用）：**

OpenClaw を起動する前に変数をエクスポートします。

```bash
export NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt
openclaw gateway run
```

この変数について、`~/.openclaw/.env` への書き込みだけに依存しないでください。Node はプロセスの起動時に
`NODE_EXTRA_CA_CERTS` を読み取ります。

## レガシー環境変数

OpenClaw が読み取るのは `OPENCLAW_*` 環境変数のみです。以前のリリースのレガシーな
`CLAWDBOT_*` および `MOLTBOT_*` プレフィックスは暗黙的に
無視されます。

Gateway プロセスの起動時にいずれかがまだ設定されている場合、OpenClaw は
検出されたプレフィックスと合計数を一覧表示する単一の Node 非推奨警告（`OPENCLAW_LEGACY_ENV_VARS`）を
出力します。レガシープレフィックスを `OPENCLAW_` に置き換えて各値の名前を変更してください（例：`CLAWDBOT_GATEWAY_TOKEN` から
`OPENCLAW_GATEWAY_TOKEN`）。古い名前は一切機能しません。

## 関連項目

- [Gateway の設定](/ja-JP/gateway/configuration)
- [FAQ：環境変数と .env の読み込み](/ja-JP/help/faq#env-vars-and-env-loading)
- [モデルの概要](/ja-JP/concepts/models)
