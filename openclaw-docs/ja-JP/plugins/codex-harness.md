---
read_when:
    - 公式の Codex app-server ハーネスを使用したい場合
    - Codex ハーネスの設定例が必要です
    - Codex のみのデプロイで、OpenClaw にフォールバックせず失敗するようにしたい場合
summary: 公式 Codex app-server ハーネスを介して OpenClaw の埋め込みエージェントターンを実行する
title: Codex ハーネス
x-i18n:
    generated_at: "2026-07-26T09:08:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e016a1689af65c5520d529ce22a87bd25ee29369f7aedca77b27f943a7f21b0f
    source_path: plugins/codex-harness.md
    workflow: 16
---

公式の `codex` Plugin は、組み込みの OpenClaw ハーネスの代わりに Codex
app-server を介して埋め込み OpenAI エージェントターンを実行します。Codex は、
ネイティブなスレッド再開、ネイティブなツール継続、
ネイティブな Compaction、app-server での実行という低レベルのエージェントセッションを管理します。OpenClaw は引き続き、チャット
チャネル、セッションファイル、モデル選択、OpenClaw 動的ツール、承認、
メディア配信、表示されるトランスクリプトのミラーを管理します。

`openai/gpt-5.6-sol` のような正規の OpenAI モデル参照を使用してください。従来の
Codex GPT 参照は設定せず、OpenAI エージェントの認証順序を `auth.order.openai` に指定してください。
従来の Codex 認証プロファイル ID と従来の Codex 認証順序エントリは、
`openclaw doctor --fix` によって修復されます。

プロバイダー／モデルのランタイムポリシーが未設定または `auto` の場合、`openai/*` プレフィックスだけでは、
このハーネスは決して選択されません。OpenAI が Codex を暗黙的に選択できるのは、
明示的なリクエストオーバーライドがなく、公式の HTTPS Platform Responses または ChatGPT Responses
ルートと完全に一致する場合に限られます。
[OpenAI の暗黙的なエージェントランタイム](/ja-JP/providers/openai#implicit-agent-runtime)を参照してください。
Platform と ChatGPT のどちらへルーティングするかが判明する前に Codex が認証を管理する場合でも、OpenClaw は、
すべての候補ルートが Codex 互換性を宣言していることを要求します。ネイティブな
認証の管理だけで、このルートチェックが省略されることはありません。

OpenClaw サンドボックスが有効でない場合、OpenClaw は Codex のネイティブコードモードを有効にして
Codex app-server スレッドを開始します（コードモード専用はデフォルトで無効のままです）。これにより、
app-server の `item/tool/call` ブリッジ経由でルーティングされる OpenClaw
動的ツールとともに、ネイティブなワークスペース／コード機能を引き続き利用できます。有効な
OpenClaw サンドボックスまたは制限付きツールポリシーがある場合、実験的なサンドボックス exec-server パスを
明示的に有効にしない限り、ネイティブコードモードは完全に無効になります。

デフォルトの `tools.exec.host: "auto"` を使用し、OpenClaw サンドボックスが有効でない場合、
Codex はペアリング済み Node でコマンドを実行するための `node_exec` および `node_process` ツールも受け取ります。
ネイティブシェルは Codex app-server のホストとワークスペース上で動作し続けます
（デフォルトの stdio デプロイでは Gateway ローカル）。`node_exec` は Node を
名前または ID で選択し、OpenClaw の Node 承認ポリシーを引き続き適用します。有限の
ランタイム許可リストによってネイティブコードモードが無効化され、ターンに実行環境が残らない場合、
OpenClaw は代わりに、ポリシーでフィルタリングされた `exec` および `process`
ツールを、サンドボックスなしの直接実行用として引き続き利用可能にします。

この Codex ネイティブ機能は、汎用 OpenClaw 実行向けのオプトイン QuickJS-WASI ランタイムである
[OpenClaw コードモード](/ja-JP/tools/code-mode)とは別のものであり、
`exec` の入力形式も異なります。モデル／プロバイダー／ランタイムのより広範な区分については、
[エージェントランタイム](/ja-JP/concepts/agent-runtimes)から始めてください。`openai/gpt-5.6-sol` はモデル
参照、`codex` はランタイム、Telegram、Discord、Slack、またはその他の
チャネルは通信サーフェスです。

## 要件

- 公式の `@openclaw/codex` Plugin がインストールされていること。設定で許可リストを使用している場合は、
  `plugins.allow` に `codex` を含めてください。
- `0.143.0` から `0.145.0` までの安定版 Codex app-server。Plugin はデフォルトで互換性のある
  バイナリを管理するため、`PATH` 上の `codex` コマンドは通常の
  起動に影響しません。
- `openclaw models auth login --provider openai`、エージェントの Codex ホームにすでに存在する
  app-server アカウント、または明示的な Codex API キー認証プロファイルを介した Codex 認証。

認証の優先順位、環境分離、カスタム app-server コマンド、
モデル検出、設定フィールドの完全な一覧については、
[Codex ハーネスリファレンス](/ja-JP/plugins/codex-harness-reference)を参照してください。

## クイックスタート

公式 Plugin をインストールしてから、Codex OAuth でサインインします。

```bash
openclaw plugins install @openclaw/codex
openclaw models auth login --provider openai
```

`codex` Plugin を有効にし、OpenAI エージェントモデルを選択します。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

設定で `plugins.allow` を使用している場合は、そこに `codex` も追加します。

```json5
{
  plugins: {
    allow: ["codex"],
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Plugin の設定を変更した後は Gateway を再起動してください。チャットにすでに
セッションがある場合、次のターンで現在の設定からハーネスを解決できるよう、先に `/new` または `/reset` を実行してください。

## Codex Desktop および CLI とスレッドを共有する

デフォルトの `appServer.homeScope: "agent"` は、各 OpenClaw エージェントを
オペレーターのネイティブ Codex 状態から分離します。所有者が Codex Desktop および Codex CLI に表示されるものと
同じネイティブスレッドを確認および管理できるようにするには、ユーザーの Codex ホームを
明示的に有効にします。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            homeScope: "user",
          },
        },
      },
    },
  },
}
```

ユーザーホームモードでは、ローカルの管理対象 stdio プロセスまたは共有 Unix ソケット
トランスポートを利用できます。設定されている場合は `$CODEX_HOME`、それ以外の場合は `~/.codex` を使用し、
そのホームのネイティブ Codex 認証、設定、Plugin、スレッドストアも含まれます。OpenClaw は
この app-server に OpenClaw 認証プロファイルを注入しません。

所有者のターンでは `codex_threads` ツールを利用できます。ネイティブスレッドの一覧表示、検索、読み取り、フォーク、名前変更、
アーカイブ、復元が可能です。スレッドをフォークして OpenClaw で継続できます。
フォークは現在の OpenClaw セッションに接続され、他のネイティブ Codex クライアントからも
引き続き表示されます。アーカイブするには、そのスレッドが他の場所で閉じられていることを明示的に
確認する必要があります。監視も有効な場合、トランスクリプトフィールドと変更操作には、
対応する `supervision.allowRawTranscripts` または `supervision.allowWriteControls` のオプトインが必要です。

独立した管理対象 stdio App Server を介して、同じスレッドを同時に再開または書き込みしないでください。
Codex が同時書き込みを調整するのは単一の App Server 内であり、
別々のプロセス間ではありません。通常のユーザーホーム stdio セッションで安全に共存するには、
フォークを使用してください。

`appServer.homeScope: "user"` だけではフリートカタログを制御しません。Plugin が有効な間は
ネイティブセッション検出が有効になります。Codex を無効にせずに OpenClaw サイドバーから
削除するには、`sessionCatalog.enabled: false` を設定してください。カタログは別個の監視接続を使用します。
明示的な `appServer` 接続設定がない場合、通常のハーネスはエージェントスコープのまま、
その接続はデフォルトで管理対象のユーザーホーム stdio を使用します。明示的な
`appServer` 設定は両方のパスで適用されます。通常のハーネスでもネイティブ状態を共有する場合は、
上記のように `homeScope: "user"` を明示的に設定してください。

## Codex セッションを監視する

同じ `codex` Plugin で、Gateway コンピューターおよびオプトイン済みのペアリング済み Node にある
アーカイブされていない Codex セッションを一覧表示できます。保存済みまたはアイドル状態の Gateway ローカルセッションから、
永続化されたユーザーとアシスタントの履歴を上限内でミラーする、モデル固定のチャットを
作成できます。そのプライベートバインディングは、ネイティブ
スナップショット、正規ブランチ、および後続のターンに監視接続を使用しますが、通常の Codex セッションは
エージェントスコープのままです。最初の正規開始では、スナップショットのフォークについて Codex が返したモデルとプロバイダーを
そのまま使用します。後続の再開では、Codex の
ネイティブ設定に選択を委ねます。外側の OpenClaw モデルおよびフォールバックチェーンが
それを置き換えることはありません。保存済みおよびアイドル状態の行は、他に実行中のものがないことを明示的に
確認した後でアーカイブできます。アクティブなソースからブランチを作成したりアーカイブしたりすることはできませんが、既存の
監視対象チャットは引き続き開けます。ペアリング済み Node のセッションはメタデータのみのままです。

セットアップ、ブランチルール、ペアリング済み Node の制限、メタデータの公開、トラブルシューティングについては、
[Codex セッションを監視する](/ja-JP/plugins/codex-supervision)を参照してください。

## 設定

| 目的                                                | 設定                                                                                              | 場所                              |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------ | ---------------------------------- |
| ハーネスを有効にする                                  | `plugins.entries.codex.enabled: true`                                                            | OpenClaw 設定                    |
| ネイティブ Codex セッション検出を非表示にする                 | `plugins.entries.codex.config.sessionCatalog.enabled: false`                                     | Codex Plugin 設定                |
| 許可リスト方式の Plugin インストールを維持する                  | `plugins.allow` に `codex` を含める                                                               | OpenClaw 設定                    |
| 対象となる OpenAI ターンが Codex を暗黙的に使用できるようにする | 公式の HTTPS Responses／ChatGPT ルートと完全に一致し、明示的なリクエストオーバーライドがなく、ランタイムが未設定／`auto` | OpenAI プロバイダー／モデル設定       |
| ChatGPT／Codex OAuth でサインインする                    | `openclaw models auth login --provider openai`                                                   | CLI 認証プロファイル                   |
| Codex 実行用の API キーバックアップを追加する                   | `auth.order.openai` でサブスクリプション認証の後に列挙された `openai:*` API キープロファイル                 | CLI 認証プロファイル + OpenClaw 設定 |
| Codex が利用できない場合にフェイルクローズする               | プロバイダーまたはモデルの `agentRuntime.id: "codex"`                                                     | OpenClaw モデル／プロバイダー設定     |
| OpenAI API トラフィックを直接使用する                       | 通常の OpenAI 認証を使用するプロバイダーまたはモデルの `agentRuntime.id: "openclaw"`                          | OpenClaw モデル／プロバイダー設定     |
| app-server の動作を調整する                            | `plugins.entries.codex.config.appServer.*`                                                       | Codex Plugin 設定                |
| ネイティブ Codex Plugin アプリを有効にする                  | `plugins.entries.codex.config.codexPlugins.*`                                                    | Codex Plugin 設定                |
| Codex Computer Use を有効にする                       | `plugins.entries.codex.config.computerUse.*`                                                     | Codex Plugin 設定                |

サブスクリプション優先／API キーバックアップの順序には `auth.order.openai` を推奨します。
既存の従来の Codex 認証プロファイル ID と従来の Codex 認証順序は、
doctor 専用のレガシー状態です。新しい従来の Codex GPT 参照は記述しないでください。

```json5
{
  auth: {
    order: {
      openai: ["openai:user@example.com", "openai:api-key-backup"],
    },
  },
}
```

Codex 互換の有効ルートでは、上記の両プロファイルが同じ Codex 実行の
候補として残ります。プロファイルの順序は認証情報を選択するものであり、ランタイムを選択するものではありません。
認証順序を変更しても、カスタムルート、Completions ルート、HTTP ルート、または
リクエストでオーバーライドされたルートが Codex 互換になることはありません。

### Compaction

Codex ベースのエージェントには `compaction.model` または `compaction.provider` を設定しないでください。
Codex はネイティブの app-server スレッド状態を介して Compaction を行うため、
OpenClaw は実行時にそれらのローカル要約機能のオーバーライドを無視し、
エージェントが Codex を使用する場合、`openclaw doctor --fix` がそれらを削除します。

Lossless は引き続き、Codex ターン周辺の組み立て、取り込み、保守用のコンテキストエンジンとしてサポートされます。
設定には
`plugins.slots.contextEngine: "lossless-claw"` および
`plugins.entries.lossless-claw.config.summaryModel` を使用し、
`agents.defaults.compaction.provider` は使用しません。Codex がアクティブなランタイムの場合、
`openclaw doctor --fix` は古い `compaction.provider: "lossless-claw"` 形式を Lossless
コンテキストエンジンのスロットへ移行しますが、Compaction は引き続きネイティブ Codex が
管理します。ネイティブ app-server ハーネスは、プロンプト前の組み立てを必要とする
コンテキストエンジンをサポートしますが、`codex-cli` を含む汎用 CLI バックエンドは、
そのホスト機能を提供しません。

Codex ベースのエージェントでは、`/compact` がバインド済みスレッド上でネイティブ Codex app-server の
Compaction を開始し、その終了結果を待機します。共有の
`agents.defaults.compaction.timeoutSeconds` 予算が適用されます。タイムアウトすると、
OpenClaw は Codex にネイティブターンの中断を要求し、終了が確認されるまでスレッドごとのフェンスを
維持します。コンテキストエンジンや公開 OpenAI 要約機能に
フォールバックすることはありません。ネイティブ Codex スレッドのバインディングが欠落しているか
古い場合、Compaction バックエンドを暗黙的に切り替えず、コマンドはフェイルクローズします。

### 直接 API のロングコンテキスト

Codex サブスクリプションと OpenAI API への直接トラフィックは別個の契約です。
ライブの ChatGPT/Codex カタログでは通常、`272000` トークンのモデルウィンドウが公開されますが、
OpenAI のドキュメントでは、GPT-5.5 および GPT-5.6 について Platform API のウィンドウは `1050000` トークン、最大出力は `128000` とされています。出力許容量をすべて確保すると、
算出される入力予算は `922000` トークンになります。入力トークンが `272000` を超えるリクエストには、
OpenAI のより高い長文コンテキスト料金が適用されます。

インストール済みの Codex バージョンと互換性のある完全な Codex モデルカタログから始めます。
長文コンテキストを使用する各 GPT-5.5 または GPT-5.6 の直接エントリについて、
記述子の残りの部分を保持し、次のように設定します。

```json
{
  "context_window": 922000,
  "max_context_window": 922000,
  "auto_compact_token_limit": 700000
}
```

Codex は、カタログの `922000` 値に通常の 95% の有効ウィンドウ予約を適用するため、
使用可能なトークンは約 `875900` と報告されます。`700000` で Compaction すると、
その有効ガードまで `175900` トークン、プロバイダーの安全な入力許容量まで `222000` トークンが残ります。
この大きめの余裕は意図的なものです。Codex は、次のユーザーメッセージとコンテキスト更新を追加する前に、
すでに記録されているコンテキストを確認するため、しきい値には、1 回の大きな受信ターンに加えて、
ツール、指示、シリアライズ、および Compaction ターン自体を収容できる余裕が必要です。

スタンドアロンの Codex CLI または Desktop で使用する場合、コマンド認証のカスタムプロバイダーは、
通常の ChatGPT ログインをコネクター用に引き続き使用可能にしたまま、
システムのキーチェーンまたはシークレットマネージャーから API キーを読み取れます。

```toml
model = "gpt-5.6-terra"
model_provider = "openai_api_direct"
model_context_window = 922000
model_auto_compact_token_limit = 700000
model_auto_compact_token_limit_scope = "total"
model_catalog_json = "/absolute/path/to/models-api-1m.json"

[model_providers.openai_api_direct]
name = "OpenAI API direct"
base_url = "https://api.openai.com/v1"
wire_api = "responses"
requires_openai_auth = false

[model_providers.openai_api_direct.auth]
command = "/absolute/path/to/read-openai-inference-key"
timeout_ms = 5000
refresh_interval_ms = 300000
```

認証ヘルパーは、キーのみを stdout に出力する必要があります。TOML には記載しないでください。

OpenClaw Codex app-server ハーネスでは、デフォルトのエージェントスコープの Codex
ホームを維持し、OpenClaw に `openai` API キープロファイルを注入させます。カタログと
コンテキスト制限は、Codex app-server のネイティブ引数として渡します。

```json5
{
  auth: {
    order: {
      openai: ["openai:api-key"],
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            args: [
              "app-server",
              "--listen",
              "stdio://",
              "-c",
              'model_catalog_json="/absolute/path/to/models-api-1m.json"',
              "-c",
              "model_context_window=922000",
              "-c",
              "model_auto_compact_token_limit=700000",
              "-c",
              "model_auto_compact_token_limit_scope=total",
            ],
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-terra",
      models: {
        "openai/gpt-5.6-terra": { agentRuntime: { id: "codex" } },
      },
    },
  },
}
```

必要に応じて、`openai:api-key` を実際の API キープロファイル ID に置き換えます。
エージェントスコープの app-server が受け取るのは、その準備済みキーのみです。オペレーターのネイティブな
`~/.codex` ChatGPT ログイン、Plugin、コネクター、スレッドストアには変更を加えません。
Codex app-server の `0.144.6` は、app-server ターンにコマンド認証カスタムプロバイダーの
Bearer を付加しないため、この経路では `homeScope: "user"` ではなく、上記の注入 API キー経路を使用します。

カタログまたは app-server 引数を変更した後は、Gateway を再起動し、
新しいチャットを開始します。既存のネイティブスレッドは、記録済みのプロバイダーと
モデル設定を保持します。`/status` と `/codex status` でランタイムを確認してから、
長時間のセッションを開始する前に、無害な直接 API ターンを送信します。

<Warning>
長文コンテキストは意図的にオプトインです。入力が `272000` トークンを超えると、
OpenAI はリクエスト全体に対して入力 2 倍、出力 1.5 倍の料金を請求します。アクセス権、実際の制限、
請求については API が最終的な基準です。
[OpenAI のモデル制限](https://developers.openai.com/api/docs/models/compare)および
[API の料金](https://developers.openai.com/api/docs/pricing)を参照してください。
</Warning>

このページの残りでは、デプロイ形態、フェイルクローズルーティング、ガーディアンの
承認ポリシー、ネイティブ Codex Plugin、および Computer Use について説明します。オプション、
デフォルト、列挙値、検出、環境の分離、タイムアウト、および app-server トランスポートフィールドの
完全な一覧については、
[Codex ハーネスリファレンス](/ja-JP/plugins/codex-harness-reference)を参照してください。

## Codex ランタイムを確認する

Codex が使用されると想定されるチャットで `/status` を使用します。Codex をバックエンドとする OpenAI
エージェントターンでは、次のように表示されます。

```text
ランタイム：OpenAI Codex
```

次に、Codex app-server の状態を確認します。

```text
/codex status
/codex models
/codex binding
```

`/codex binding` は、接続されたネイティブスレッドと現在のモデル設定を報告します。
`/codex status` は、app-server の接続状態、アカウント、レート制限、MCP
サーバー、および Skills を報告します。`/codex models` は、ハーネスとアカウントで使用できる
ライブの Codex app-server カタログを一覧表示します。`/status` が想定外の場合は、
[トラブルシューティング](#troubleshooting)を参照してください。

## ルーティングとモデル選択

プロバイダー参照とランタイムポリシーは分離しておきます。

- 正規の OpenAI モデル選択には `openai/gpt-*` を使用します。プレフィックスだけでは、
  Codex が選択されることはありません。
- ランタイムが未設定または `auto` の場合、作成者によるリクエストのオーバーライドがない、
  正確な公式 HTTPS Platform Responses または ChatGPT Responses 経路だけが、暗黙的に Codex を選択できます。
- 設定では従来の Codex GPT 参照を使用しないでください。従来の参照と古いセッションルート固定を
  修復するには、`openclaw doctor --fix` を実行します。
- `agentRuntime.id: "codex"` は、互換性のある経路に対して Codex を
  フェイルクローズ要件にします。互換性のない実効経路を互換性のあるものにするわけではありません。
- `agentRuntime.id: "openclaw"` は、意図的に使用する場合に、プロバイダーまたはモデルを
  組み込み OpenClaw ランタイムにオプトインします。
- `/codex ...` は、チャットからネイティブ Codex app-server の会話を制御します。
- ACP/acpx は、独立した外部ハーネス経路です。ユーザーが ACP/acpx または外部ハーネスアダプターを
  求めた場合にのみ使用します。

| ユーザーの意図                                             | 使用                                                                                                  |
| ---------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| 現在のチャットを接続する                                   | `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]`                    |
| 既存の Codex スレッドを再開する                            | `/codex resume <thread-id>`                                                                           |
| Codex スレッドを一覧表示または絞り込む                     | `/codex threads [filter]`                                                                             |
| バインド済みスレッドのネイティブ目標を読み取る、または更新する | `/codex goal [status\|set <objective>\|pause\|resume\|block\|complete\|clear]`                        |
| ネイティブ Codex Plugin を一覧表示する                     | `/codex plugins list`                                                                                 |
| 設定済みのネイティブ Codex Plugin を有効または無効にする   | `/codex plugins enable <name>`, `/codex plugins disable <name>`                                       |
| 保存済み Codex CLI セッションをペア Node ターンとして再開する | `/codex sessions --host <node> [filter]`、続いて `/codex resume <session-id> --host <node> --bind here` |
| 複数のコンピューターにある未アーカイブの Codex セッションを表示する | Codex 監視を有効にして **Codex セッション** を開く                                                  |
| バインド済みスレッドのモデル、高速モード、または権限を変更する | `/codex model <model>`, `/codex fast [on\|off\|status]`, `/codex permissions [default\|yolo\|status]` |
| アクティブなターンを停止または誘導する                     | `/codex stop`, `/codex steer <text>`                                                                  |
| 現在のバインドを解除する                                   | `/codex detach`（別名 `/codex unbind`）                                                               |
| Codex フィードバックのみを送信する                         | `/codex diagnostics [note]`                                                                           |
| ACP/acpx タスクを開始する                                  | `/codex` ではなく、ACP/acpx セッションコマンド                                                               |

| ユースケース                                    | 設定                                                                                                        | 確認                                    | 注記                                         |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------- | --------------------------------------- | -------------------------------------------- |
| ネイティブ Codex ランタイムを使用できる OpenAI 経路 | 作成者によるリクエストのオーバーライドがない正確な公式 HTTPS Responses/ChatGPT 経路、および有効な `codex` Plugin | `/status` に `Runtime: OpenAI Codex` が表示される | ランタイムが未設定/`auto` の場合の暗黙的な経路 |
| Codex が利用できない場合にフェイルクローズする  | プロバイダーまたはモデルの `agentRuntime.id: "codex"`                                                               | 組み込みへのフォールバックではなくターンが失敗する | Codex 専用デプロイで使用する                  |
| OpenClaw 経由の OpenAI API キー直接トラフィック | プロバイダーまたはモデルの `agentRuntime.id: "openclaw"` と通常の OpenAI 認証                                          | `/status` に OpenClaw ランタイムが表示される | OpenClaw の使用が意図的な場合にのみ使用する   |
| 従来の設定                                      | 従来の Codex GPT 参照                                                                                       | `openclaw doctor --fix` が書き換える          | この方法で新しい設定を記述しない             |
| ACP/acpx Codex アダプター                       | ACP `sessions_spawn({ runtime: "acp" })`                                                                                      | ACP タスク/セッションの状態              | ネイティブ Codex ハーネスとは別              |

`agents.defaults.imageModel` も同じプレフィックス分割に従います。通常の OpenAI 経路には `openai/gpt-*` を使用し、
画像理解を制限付き Codex app-server ターンで実行する必要がある場合にのみ `codex/gpt-*` を使用します。
Doctor は、従来の Codex GPT 参照を `openai/gpt-*` に書き換えます。

## デプロイパターン

### 基本的な Codex デプロイ

実効的な公式 HTTPS 経路が Codex を暗黙的に選択できる OpenAI モデルには、
クイックスタート設定を使用します。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

### 混在プロバイダーデプロイ

Claude をデフォルトエージェントとして維持し、名前付き Codex エージェントを追加します。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
  agents: {
    defaults: {
      model: "anthropic/claude-opus-4-6",
    },
    list: [
      {
        id: "main",
        default: true,
        model: "anthropic/claude-opus-4-6",
      },
      {
        id: "codex",
        name: "Codex",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

`main` エージェントは、通常のプロバイダー経路を使用します。`codex` エージェントは、
実効的な OpenAI 経路に互換性が保たれている場合に Codex app-server を使用します。これをフェイルクローズ要件にする必要がある場合は、
モデルスコープの明示的な `agentRuntime.id: "codex"` を追加します。

### フェイルクローズ Codex デプロイ

バンドルされた Plugin が利用可能な場合、適格な正確な公式 HTTPS OpenAI 経路は Codex に解決できます。
明文化されたフェイルクローズルールには、明示的なランタイムポリシーを追加します。

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: {
          id: "codex",
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Codex を強制した場合、有効なルートが Codex 互換として宣言されていない、Plugin が無効になっている、app-server が古すぎる、または app-server を起動できないと、OpenClaw は早期に失敗します。

## App-server のポリシー

デフォルトでは、Plugin は OpenClaw が管理する Codex バイナリを stdio トランスポートでローカル起動します。意図的に別の実行可能ファイルを実行する場合にのみ `appServer.command` を設定してください。Codex は WebSocket トランスポートを実験的かつ未サポートと分類しています。別の場所ですでに稼働している app-server に対する非本番テストにのみ使用してください。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            transport: "websocket",
            url: "ws://gateway-host:39175",
            authToken: "${CODEX_APP_SERVER_TOKEN}",
          },
        },
      },
    },
  },
}
```

ローカルの stdio app-server セッションは、信頼されたローカルオペレーターの体制である `approvalPolicy: "never"`、`approvalsReviewer: "user"`、および `sandbox: "danger-full-access"` をデフォルトとします。ローカルの Codex 要件でこの暗黙的な YOLO 体制が許可されない場合、OpenClaw は代わりに許可された Guardian 権限を選択します。セッションで OpenClaw サンドボックスが有効な場合、OpenClaw は Codex のホスト側サンドボックスに依存せず、そのターンでは Codex ネイティブの Code Mode、ユーザー MCP サーバー、およびアプリに基づく Plugin の実行を無効にします。代わりに、通常の exec/process ツールが利用可能であれば、シェルアクセスは `sandbox_exec` や `sandbox_process` などの OpenClaw サンドボックスに基づく動的ツールを経由します。

サンドボックスからの脱出や追加権限より先に Codex ネイティブの自動レビューを行うには、正規化された OpenClaw exec モードを使用します。

```json5
{
  tools: {
    exec: {
      mode: "auto",
    },
  },
  plugins: {
    entries: {
      codex: {
        enabled: true,
      },
    },
  },
}
```

Codex app-server セッションでは、`tools.exec.mode: "auto"` は Codex Guardian がレビューする承認に対応します。ローカル要件でそれらの値が許可されている場合、通常は `approvalPolicy: "on-request"`、`approvalsReviewer: "auto_review"`、および `sandbox: "workspace-write"` です。`tools.exec.mode: "auto"` では、OpenClaw は従来の安全でない Codex の `approvalPolicy: "never"` または `sandbox: "danger-full-access"` オーバーライドを保持しません。承認なしの Codex 体制を意図的に使用する場合は `tools.exec.mode: "full"` を使用してください。従来の `plugins.entries.codex.config.appServer.mode: "guardian"` プリセットも引き続き機能しますが、正規化された OpenClaw サーフェスは `tools.exec.mode: "auto"` です。

ホストの exec 承認および ACPX 権限とのモード単位の比較については、[権限モード](/ja-JP/tools/permission-modes)を参照してください。すべての app-server フィールド、認証順序、環境分離、およびタイムアウト動作については、[Codex ハーネスリファレンス](/ja-JP/plugins/codex-harness-reference)を参照してください。

## コマンドと診断

`codex` Plugin は、OpenClaw テキストコマンドをサポートするすべてのチャンネルで、`/codex` をスラッシュコマンドとして登録します。

ネイティブの実行と制御には、所有者または `operator.admin` Gateway クライアントが必要です。対象となる操作は、スレッドのバインドまたは再開、ターンの送信または停止、モデル、fast-mode、または権限状態の変更、Compaction またはレビュー、およびバインドの解除です。その他の認可済み送信者は、ステータス、ヘルプ、アカウント、モデル、スレッド、ネイティブ目標、MCP サーバー、スキル、およびバインドを検査する読み取り専用コマンドを引き続き使用できます。

一般的な形式：

- `/codex status` は、app-server の接続性、モデル、アカウント、レート制限、MCP サーバー、およびスキルを確認します。
- `/codex models` は、稼働中の Codex app-server モデルを一覧表示します。
- `/codex threads [filter]` は、最近の Codex app-server スレッドを一覧表示します。
- `/codex goal` は、接続されたスレッドのネイティブ Codex 目標を読み取るか更新します。Codex の自動的な目標継続は無効のままです。OpenClaw はまだ自律的な後続ターンを所有していません。
- `/codex resume <thread-id>` は、現在の OpenClaw セッションを既存の Codex スレッドに接続します。
- `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]` は、現在のチャットを接続します。
- `/codex detach`（または `/codex unbind`）は、現在のバインドを解除します。
- `/codex binding` は、現在のバインドを説明します。
- `/codex stop` はアクティブなターンを停止し、`/codex steer <text>` はそのターンを誘導します。
- `/codex model <model>`、`/codex fast [on|off|status]`、および `/codex permissions [default|yolo|status]` は、会話ごとの状態を変更します。
- `/codex compact` は、接続されたスレッドを Compaction するよう Codex app-server に要求します。
- `/codex review` は、接続されたスレッドに対する Codex ネイティブレビューを開始します。
- `/codex diagnostics [note]` は、接続されたスレッドの Codex フィードバックを送信する前に確認を求めます。
- `/codex account` は、アカウントとレート制限のステータスを表示します。
- `/codex mcp` は、Codex app-server の MCP サーバーステータスを一覧表示します。
- `/codex skills` は、Codex app-server のスキルを一覧表示します。
- `/codex plugins list`、`/codex plugins enable <name>`、および `/codex plugins disable <name>` は、設定済みのネイティブ Codex Plugin を管理します。
- `/codex computer-use [status|install]` は、Codex Computer Use を管理します。
- `/codex help` は、完全なコマンドツリーを一覧表示します。

ほとんどのサポート報告では、バグが発生した会話で `/diagnostics [note]` を実行することから始めてください。これにより Gateway 診断レポートが 1 件作成され、Codex ハーネスセッションの場合は、関連する Codex フィードバックバンドルを送信するための承認を求めます。プライバシーモデルとグループチャットでの動作については、[診断のエクスポート](/ja-JP/gateway/diagnostics)を参照してください。完全な Gateway 診断バンドルを含めず、現在接続されているスレッドの Codex フィードバックのみを明示的にアップロードする場合に限り、`/codex diagnostics [note]` を使用してください。

### Codex スレッドをローカルで検査する

問題のある Codex 実行を検査する最も速い方法は、多くの場合、ネイティブ Codex スレッドを直接開くことです。

```bash
codex resume <thread-id>
```

完了した `/diagnostics` の応答、`/codex binding`、または `/codex threads [filter]` からスレッド ID を取得します。

アップロードの仕組みとランタイムレベルの診断境界については、[Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime#codex-feedback-upload)を参照してください。

### 認証順序

デフォルトのエージェントごとのホームでは、認証は次の順序で選択されます。

1. エージェント用に順序付けされた OpenAI 認証プロファイル。できれば `auth.order.openai` 配下に配置します。古い従来の Codex 認証プロファイル ID と従来の Codex 認証順序を移行するには、`openclaw doctor --fix` を実行します。
2. そのエージェントの Codex ホームにある app-server の既存アカウント。
3. ローカルの stdio app-server 起動に限り、app-server アカウントが存在せず、OpenAI 認証が引き続き必要な場合は、`CODEX_API_KEY`、次に `OPENAI_API_KEY`。

OpenClaw が ChatGPT サブスクリプション形式の Codex 認証プロファイルを検出すると、生成される Codex 子プロセスから `CODEX_API_KEY` と `OPENAI_API_KEY` を削除します。これにより、Gateway レベルの API キーを埋め込みや直接の OpenAI モデルで利用可能な状態に保ちつつ、ネイティブ Codex app-server のターンが誤って API 経由で請求されることを防ぎます。明示的な Codex API キープロファイルとローカル stdio の環境変数キーへのフォールバックでは、継承された子プロセスの環境変数ではなく app-server ログインを使用します。WebSocket app-server 接続には Gateway の環境変数 API キーフォールバックは渡されません。明示的な認証プロファイルまたはリモート app-server 自身のアカウントを使用してください。

サブスクリプションプロファイルが Codex の使用量制限に達した場合、Codex がリセット時刻を報告すれば OpenClaw はその時刻を記録し、同じ Codex 実行に対して次の順序付き認証プロファイルを試します。リセット時刻を過ぎると、選択されている `openai/gpt-*` モデルまたは Codex ランタイムを変更することなく、サブスクリプションプロファイルが再び利用可能になります。

ネイティブ Codex Plugin が設定されている場合、OpenClaw は Plugin が所有するアプリを Codex スレッドに公開する前に、接続された app-server を通じてそれらの Plugin をインストールまたは更新します。`app/list` は引き続きアプリ ID、アクセシビリティ、およびメタデータの信頼できる情報源ですが、スレッドごとの有効化判断は OpenClaw が所有します。ポリシーで一覧内のアクセス可能なアプリが許可されている場合、`app/list` が現在そのアプリを無効と報告していても、OpenClaw は `thread/start.config.apps[appId].enabled = true` を送信します。この経路では、未知の ID に対してアプリのインストールを作り出すことはありません。OpenClaw は `plugin/install` を持つマーケットプレイス Plugin のみを有効化し、その後インベントリを更新します。

### 環境分離

ローカルの stdio app-server 起動では、OpenClaw は `CODEX_HOME` をエージェントごとのディレクトリに設定します。これにより、Codex の設定、認証・アカウントファイル、Plugin のキャッシュ・データ、およびネイティブスレッド状態は、デフォルトでオペレーター個人の `~/.codex` を読み書きしません。OpenClaw は通常のプロセスの `HOME` を保持します。Codex が実行するサブプロセスは引き続きユーザーホームの設定やトークンを見つけることができ、Codex は共有の `$HOME/.agents/skills` および `$HOME/.agents/plugins/marketplace.json` エントリを検出する場合があります。`appServer.homeScope: "user"` を使用すると、OpenClaw は代わりにネイティブユーザーの Codex ホームとその既存アカウントを使用し、OpenClaw 認証プロファイルを注入しません。

デプロイで追加の環境分離が必要な場合は、それらの変数を `appServer.clearEnv` に追加します。

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          appServer: {
            clearEnv: ["CODEX_API_KEY", "OPENAI_API_KEY"],
          },
        },
      },
    },
  },
}
```

`appServer.clearEnv` は、生成された Codex app-server 子プロセスにのみ影響します。OpenClaw はローカル起動の正規化中に、このリストから `CODEX_HOME` と `HOME` を削除します。`CODEX_HOME` は選択されたエージェントまたはユーザースコープを指したままとなり、`HOME` は継承されたままなので、サブプロセスは通常のユーザーホーム状態を使用できます。

### 動的ツールとウェブ検索

Codex の動的ツールは、デフォルトで `searchable` 読み込みを使用します。OpenClaw は通常、Codex ネイティブのワークスペース操作と重複する動的ツールを公開しません。対象は `read`、`write`、`edit`、`apply_patch`、`exec`、`process`、`update_plan`、`get_goal`、`create_goal`、`update_goal`、`tool_call`、`tool_describe`、`tool_search`、および `tool_search_code` です。目標操作は Codex ネイティブのままなので、OpenClaw は 2 つ目の目標ストアを Codex ターンに投影しません。メッセージング、メディア、Cron、ブラウザー、Node、Gateway、および `heartbeat_respond` など、残りのほとんどの OpenClaw 統合ツールは、`openclaw` 名前空間の Codex ツール検索を通じて利用でき、初期モデルコンテキストを小さく保ちます。制限付きターンのシェルフォールバックは、有限の許可リストによってネイティブ Code Mode が無効になる場合の `exec` と `process` に対する例外です。ランタイムの許可リストと `codexDynamicToolsExclude` は引き続き適用されます。

OpenClaw の `computer` ツールを含む、`catalogMode: "direct-only"` とマークされたツールは、代わりに `openclaw_direct` 名前空間を使用します。Codex はその名前空間を `DirectModelOnly` として扱うため、それらのツールはネストされた Code Mode の `tools.*` 呼び出しを経由せず、通常のスレッドと Code Mode 専用スレッドの両方でモデルから直接参照可能な状態を維持します。

検索が有効で管理対象プロバイダーが選択されていない場合、ウェブ検索はデフォルトで Codex のホスト型 `web_search` ツールを使用します。ネイティブのホスト型検索と OpenClaw が管理する `web_search` 動的ツールは相互排他的であるため、管理対象検索がネイティブのドメイン制限を回避することはできません。ホスト型検索が利用できない、明示的に無効化されている、または選択された管理対象プロバイダーに置き換えられている場合、OpenClaw は管理対象ツールを使用します。本番 app-server トラフィックはユーザー定義の `web` 名前空間を拒否するため、OpenClaw は Codex のスタンドアロン `web.run` 拡張機能を無効のままにします。`tools.web.search.enabled: false` は両方の経路を無効にし、ツールが無効な LLM 専用実行も同様です。Codex は `"cached"` を設定として扱い、制限のない app-server ターンでは実際の外部アクセスへ解決します。ネイティブの `allowedDomains` が設定されている場合、自動的な管理対象フォールバックはフェイルクローズとなるため、許可リストを回避できません。永続的な有効検索ポリシーの変更では、次のターンの前にバインドされた Codex スレッドをローテーションします。一時的なターン単位の制限では、一時的な制限付きスレッドを使用し、後で再開できるよう既存のバインドを保持します。

`sessions_yield`、`sessions_spawn`、およびメッセージツール専用のソース返信は、
ターン制御または委任の契約であるため、引き続き直接処理されます。ガイダンスでは依然として、
Codex の主要なサブエージェントサーフェスとしてネイティブの `spawn_agent` を優先しますが、
明示的な OpenClaw または ACP の委任は `sessions_spawn` を通じて
引き続き直接呼び出せます。Codex Code Mode では、汎用 OpenClaw
動的ツールの結果は JavaScript オブジェクトではなく JSON テキストであるため、
フィールドを読み取る前に JSON に見える結果を解析してください。Codex はネストされた
動的呼び出しも直列化します。`Promise.all` がそれらを同時に起動することを
期待せず、複数の `sessions_spawn` 呼び出しを上限付きループで送信してください。
すでに受け付けられた子は、後続の呼び出しが送信されている間も並行して実行できます。完全なパターンについては、
[Swarm](/tools/swarm#use-swarm-from-other-harnesses)を参照してください。
Heartbeat のコラボレーション手順では、
ツールがまだ読み込まれていない場合、Heartbeat ターンを終了する前に
`heartbeat_respond` を検索するよう Codex に指示します。

遅延動的ツールを検索できないカスタム
Codex app-server に接続する場合、または
完全なツールペイロードをデバッグする場合にのみ、`codexDynamicToolsLoading: "direct"` を設定してください。

### 設定フィールド

サポートされているトップレベルの Codex Plugin フィールド：

| フィールド                 | デフォルト     | 意味                                                                                     |
| -------------------------- | -------------- | ---------------------------------------------------------------------------------------- |
| `codexDynamicToolsLoading` | `"searchable"` | `"direct"` を使用して、OpenClaw 動的ツールを初期 Codex ツールコンテキストに直接配置します。 |
| `codexDynamicToolsExclude` | `[]`           | Codex app-server のターンから除外する追加の OpenClaw 動的ツール名。                       |
| `codexPlugins`             | 無効           | 移行済みのソースインストール型キュレーション済み Plugin に対するネイティブ Codex Plugin/app サポート。 |
| `sessionCatalog`           | 有効           | この Gateway および対象となるペアリング済み Node 上のネイティブ Codex セッションをサイドバーで検出。 |
| `supervision`              | 無効           | エージェント向けのネイティブセッショントランスクリプトおよび書き込み制御ポリシー。       |

サポートされている `appServer` フィールド：

| フィールド                                         | デフォルト                                                | 意味                                                                                                                                                                                                                                                                                                                                                                                         |
| --------------------------------------------- | ------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `transport`                                   | `"stdio"`                                              | `"stdio"` は Codex を起動します。明示的な `"unix"` はローカル制御ソケットに接続し、`"websocket"` は `url` に接続します。                                                                                                                                                                                                                                                                                |
| `homeScope`                                   | `"agent"`                                              | `"agent"` は通常のハーネス状態を OpenClaw エージェントごとに分離します。`"user"` は明示的なオプトインで、ネイティブの `$CODEX_HOME` または `~/.codex` を共有し、ネイティブ認証を使用して、所有者のみのスレッド管理を有効にします。ユーザースコープでは、ローカル stdio または Unix トランスポートがサポートされます。個別の監視接続では、値が未設定の場合、stdio または Unix では `"user"`、WebSocket では `"agent"` に解決されます。     |
| `command`                                     | 管理対象の Codex バイナリ                                   | stdio トランスポート用の実行可能ファイルです。管理対象バイナリを使用する場合は未設定のままにし、明示的にオーバーライドする場合にのみ設定してください。                                                                                                                                                                                                                                                                                    |
| `args`                                        | `["app-server", "--listen", "stdio://"]`               | stdio トランスポート用の引数です。                                                                                                                                                                                                                                                                                                                                                                  |
| `url`                                         | 未設定                                                  | WebSocket App Server URL または `unix://` URL です。明示的に空の Unix パスを指定すると、標準のユーザーホーム制御ソケットが選択されます。                                                                                                                                                                                                                                                                          |
| `authToken`                                   | 未設定                                                  | WebSocket トランスポート用の Bearer トークンです。リテラル文字列、または `${CODEX_APP_SERVER_TOKEN}` などの SecretInput を受け付けます。                                                                                                                                                                                                                                                                              |
| `headers`                                     | `{}`                                                   | 追加の WebSocket ヘッダーです。ヘッダー値には、リテラル文字列または SecretInput 値（例: `x-codex-client-session-token: "${CODEX_CLIENT_SESSION_TOKEN}"`）を指定できます。                                                                                                                                                                                                                               |
| `clearEnv`                                    | `[]`                                                   | OpenClaw が継承環境を構築した後、起動された stdio app-server プロセスから削除される追加の環境変数名です。ローカル起動では、OpenClaw は選択された `CODEX_HOME` と継承された `HOME` を保持します。                                                                                                                                                                           |
| `codeModeOnly`                                | `false`                                                | Codex のコードモード専用ツールサーフェスを有効にします。通常の OpenClaw 動的ツールは、ネストされた `tools.*` 呼び出しを通じて引き続き利用できます。`openclaw_direct` ツールはモデルから直接参照可能なままです。                                                                                                                                                                                                             |
| `remoteWorkspaceRoot`                         | 未設定                                                  | リモート Codex app-server のワークスペースルートです。設定すると、OpenClaw は解決済みの OpenClaw ワークスペースからローカルワークスペースルートを推定し、現在の cwd のサフィックスをこのリモートルート配下に保持して、最終的な app-server の cwd のみを Codex に送信します。cwd が解決済みの OpenClaw ワークスペースルート外にある場合、OpenClaw は Gateway ローカルのパスをリモート app-server に送信せず、フェイルクローズします。 |
| `requestTimeoutMs`                            | `60000`                                                | app-server のコントロールプレーン呼び出しのタイムアウトです。                                                                                                                                                                                                                                                                                                                                                     |
| `turnCompletionIdleTimeoutMs`                 | `60000`                                                | Codex がターンを受け入れた後、またはターンスコープの app-server リクエスト後に、OpenClaw が `turn/completed` を待機する無通信時間です。                                                                                                                                                                                                                                                                    |
| `turnAssistantCompletionIdleTimeoutMs`        | `10000`                                                | 最終的な、または非 commentary のアシスタント項目、あるいはツール実行前の生のアシスタント完了によってアシスタント出力の解放が準備された後、OpenClaw が引き続き `turn/completed` を待機する無通信時間です。この値を大きくすると、OpenClaw が割り込んでセッションレーンを解放する前に、Codex が `turn/completed` を出力するための時間が増えます。                                                                                            |
| `postToolRawAssistantCompletionIdleTimeoutMs` | `300000`                                               | ツールへの引き渡し、ネイティブツールの完了、ツール実行後の生のアシスタント進行、生の推論完了、または推論進行の後、OpenClaw が `turn/completed` を待機する間に使用される、完了時アイドルおよび進行状況のガードです。ツール実行後の統合処理が、最終アシスタント解放の時間枠よりも長く正当に無通信状態になり得る、信頼済みまたは高負荷のワークロードに使用してください。                                |
| `mode`                                        | ローカル Codex の要件で YOLO が許可されない場合を除き `"yolo"` | YOLO または guardian レビュー済み実行のプリセットです。`danger-full-access`、`never` 承認、または `user` レビュアーを省略するローカル stdio 要件では、暗黙のデフォルトが guardian になります。                                                                                                                                                                                                           |
| `approvalPolicy`                              | `"never"` または許可された guardian 承認ポリシー       | スレッドの開始、再開、ターンに送信されるネイティブ Codex 承認ポリシーです。guardian のデフォルトでは、許可されている場合 `"on-request"` が優先されます。                                                                                                                                                                                                                                                                            |
| `sandbox`                                     | `"danger-full-access"` または許可された guardian サンドボックス  | スレッドの開始または再開時に送信されるネイティブ Codex サンドボックスモードです。guardian のデフォルトでは、許可されている場合は `"workspace-write"`、それ以外の場合は `"read-only"` が優先されます。OpenClaw サンドボックスが有効な場合、`danger-full-access` ターンでは Codex `workspace-write` が使用され、ネットワークアクセスは OpenClaw サンドボックスのエグレス設定から導出されます。                                                                                     |
| `approvalsReviewer`                           | `"user"` または許可された guardian レビュアー               | 許可されている場合、Codex にネイティブ承認プロンプトをレビューさせるには `"auto_review"` を使用します。それ以外の場合は `guardian_subagent` または `user` を使用します。`guardian_subagent` はレガシーエイリアスとして残されています。                                                                                                                                                                                                                              |
| `serviceTier`                                 | 未設定                                                  | オプションの Codex app-server サービス階層です。`"priority"` は高速モードのルーティングを有効にし、`"flex"` はフレックス処理を要求し、`null` はオーバーライドを解除します。また、レガシーの `"fast"` は `"priority"` として受け付けられます。                                                                                                                                                                                                 |
| `networkProxy`                                | 無効                                               | app-server コマンドで Codex の権限プロファイルネットワーキングを有効にします。OpenClaw は選択された `permissions.<profile>.network` 設定を定義し、`sandbox` を送信する代わりに `default_permissions` で選択します。                                                                                                                                                                             |
| `experimental.sandboxExecServer`              | `false`                                                | サポート対象の Codex app-server に OpenClaw サンドボックスベースの Codex 環境を登録し、ネイティブ Codex 実行をアクティブな OpenClaw サンドボックス内で実行できるようにするプレビュー版オプトインです。                                                                                                                                                                                                            |

`appServer.networkProxy` は Codex サンドボックスの契約を変更するため、明示的に指定します。有効にすると、OpenClaw は Codex スレッド設定に `features.network_proxy.enabled`
と `default_permissions` も設定し、生成された権限プロファイルが Codex 管理ネットワークを開始できるようにします。デフォルトでは、OpenClaw はプロファイル本体から衝突耐性のある `openclaw-network-<fingerprint>` プロファイル名を生成します。安定したローカル名が必要な場合にのみ `profileName` を使用してください。

```json5
{
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            sandbox: "workspace-write",
            networkProxy: {
              enabled: true,
              domains: {
                "api.openai.com": "allow",
                "blocked.example.com": "deny",
              },
              unixSockets: {
                "/tmp/proxy.sock": "allow",
                "/tmp/blocked.sock": "none",
              },
              allowUpstreamProxy: true,
              proxyUrl: "http://127.0.0.1:3128",
            },
          },
        },
      },
    },
  },
}
```

通常の app-server ランタイムが `danger-full-access` になる場合、`networkProxy` を有効にすると、生成される権限プロファイルにワークスペース形式のファイルシステムアクセスが使用されます。Codex 管理ネットワークの適用はサンドボックス化されたネットワーク機能であるため、フルアクセスプロファイルでは送信トラフィックを保護できません。
ドメインエントリでは `allow` または `deny` を使用し、Unix ソケットエントリでは Codex の `allow` または `none` の値を使用します。

### 動的ツール呼び出しのタイムアウト

OpenClaw が所有する動的ツール呼び出しは `appServer.requestTimeoutMs` とは独立して制限されます。Codex の `item/tool/call` リクエストでは、デフォルトで 90 秒の OpenClaw ウォッチドッグを使用します。呼び出しごとの正の `timeoutMs` 引数により、そのツールだけの時間枠を延長または短縮でき、上限は 600000 ms です。
`image_generate` ツールでは、ツール呼び出しが独自のタイムアウトを指定しない場合は `agents.defaults.mediaModels.image.timeoutMs` を使用し、それ以外の場合は画像生成用のデフォルトである 120 秒を使用します。メディア理解用の `image` ツールでは、選択した画像対応の `tools.media.models[]` エントリの `timeoutSeconds`、またはメディア用のデフォルトである 60 秒を使用します。画像理解では、このタイムアウトはリクエスト自体に適用され、それ以前の準備作業によって短縮されません。タイムアウト時、OpenClaw は対応している場合はツールのシグナルを中止し、失敗した動的ツール応答を Codex に返します。これにより、セッションを `processing` のまま残さずにターンを続行できます。
このウォッチドッグは、動的な `item/tool/call` の外側の時間枠です。プロバイダー固有のリクエストタイムアウトはその呼び出し内で動作し、それぞれ独自のタイムアウトのセマンティクスを維持します。

Codex がターンを受け入れた後、および OpenClaw がターン単位の app-server リクエストに応答した後、ハーネスは Codex が現在のターンを進行させ、最終的に `turn/completed` でネイティブターンを完了することを期待します。app-server が `appServer.turnCompletionIdleTimeoutMs` の間応答しない場合、OpenClaw はベストエフォートで Codex ターンを中断し、診断用タイムアウトを記録して OpenClaw セッションレーンを解放します。これにより、後続のチャットメッセージが古いネイティブターンの後ろでキューに入ることを防ぎます。同じターンに対する非終端通知の大半は、Codex がターンの存続を示したことになるため、この短いウォッチドッグを解除します。

ツールの引き継ぎでは、ツール後のアイドル時間枠をより長く設定します。対象となるのは、OpenClaw が `item/tool/call` 応答を返した後、`commandExecution` などのネイティブツール項目が完了した後、生の `custom_tool_call_output` 完了後、およびツール後の生のアシスタント進行、生の推論完了、または推論進行後です。ガードは、設定されている場合は `appServer.postToolRawAssistantCompletionIdleTimeoutMs` を使用し、それ以外の場合はデフォルトで 5 分を使用します。同じ時間枠により、Codex が次の現在ターンイベントを発行する前の無音の合成期間に対する進行ウォッチドッグも延長されます。レート制限の更新など、グローバルな app-server 通知はターンアイドルの進行時間をリセットしません。推論完了、commentary `agentMessage` 完了、およびツール前の生の推論またはアシスタント進行の後には、自動的な最終応答が続く可能性があるため、セッションレーンを即座に解放する代わりに、進行後の応答ガードを使用します。

最終かつ非 commentary の完了済み `agentMessage` 項目と、ツール前の生のアシスタント完了だけが、アシスタント出力による解放を作動させます。その後 Codex が `turn/completed` なしで応答しなくなった場合、OpenClaw はベストエフォートでネイティブターンを中断し、セッションレーンを解放します。別のターン監視がこの解放競合に勝った場合でも、ネイティブリクエスト、項目、動的ツール完了のいずれもアクティブでなく、アシスタント出力の解放が引き続き最新の完了項目に属し、それ以降の項目完了がない場合、OpenClaw は完了した最終アシスタント項目を受け入れます。これにより、ターンを再実行せずに、完了したツール作業後の最終回答を保持できます。部分的なアシスタント差分、古い以前の応答、および後から発生した空の完了は対象外です。

アシスタント、ツール、アクティブ項目、または副作用の証拠がないターン完了アイドルタイムアウトを含む、再実行しても安全な stdio app-server 障害は、新しい app-server 試行で 1 回再試行されます。安全でないタイムアウトでは、停止した app-server クライアントを引き続き廃止して OpenClaw セッションレーンを解放します。また、自動的に再実行する代わりに、古いネイティブスレッドのバインディングも解除します。完了監視のタイムアウトでは、Codex 固有のタイムアウトテキストが表示されます。再実行しても安全な場合は応答が不完全な可能性があることを示し、安全でない場合は再試行する前に現在の状態を確認するようユーザーに伝えます。公開されるタイムアウト診断には、最後の app-server 通知メソッド、生のアシスタント応答項目の ID／型／ロール、アクティブなリクエスト／項目数、作動中の監視状態などの構造化フィールドが含まれます。最後の通知が生のアシスタント応答項目の場合は、長さを制限したアシスタントテキストのプレビューも含まれます。生のプロンプトやツール内容は含まれません。

### ローカルテスト用の環境変数オーバーライド

- `OPENCLAW_CODEX_APP_SERVER_BIN` は、
  `appServer.command` が未設定の場合に管理対象バイナリを迂回します。
- `OPENCLAW_CODEX_APP_SERVER_ARGS`
- `OPENCLAW_CODEX_APP_SERVER_MODE=yolo|guardian`
- `OPENCLAW_CODEX_APP_SERVER_APPROVAL_POLICY`
- `OPENCLAW_CODEX_APP_SERVER_SANDBOX`

`OPENCLAW_CODEX_APP_SERVER_GUARDIAN=1` は削除されました。代わりに `plugins.entries.codex.config.appServer.mode: "guardian"` を使用するか、1 回限りのローカルテストには `OPENCLAW_CODEX_APP_SERVER_MODE=guardian` を使用してください。繰り返し可能なデプロイでは設定の使用が推奨されます。これにより、Plugin の動作を Codex ハーネスの残りのセットアップと同じレビュー済みファイル内に保持できます。

## ネイティブ Codex Plugin

ネイティブ Codex Plugin のサポートでは、OpenClaw ハーネスターンと同じ Codex スレッド内で、Codex app-server 独自のアプリおよび Plugin 機能を使用します。OpenClaw は Codex Plugin を合成された `codex_plugin_*` OpenClaw 動的ツールに変換しません。

`codexPlugins` が影響するのは、ネイティブ Codex ハーネスを選択するセッションだけです。組み込みハーネスの実行、通常の OpenAI プロバイダーの実行、ACP 会話バインディング、その他のハーネスには影響しません。

最小限の移行済み設定：

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_destructive_actions: true,
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
              },
            },
          },
        },
      },
    },
  },
}
```

スレッドアプリ設定は、OpenClaw が Codex ハーネスセッションを確立するか、古い Codex スレッドバインディングを置き換えるときに計算されます。ターンごとには再計算されません。`codexPlugins` を変更した後は、`/new`、`/reset` を使用するか、Gateway を再起動して、今後の Codex ハーネスセッションが更新されたアプリセットで開始されるようにしてください。

移行対象の条件、アプリインベントリ、破壊的アクションのポリシー、情報要求、ネイティブ Plugin の診断については、[ネイティブ Codex Plugin](/ja-JP/plugins/codex-native-plugins)を参照してください。

OpenAI 側のアプリおよび Plugin へのアクセスは、サインイン中の Codex アカウントによって制御されます。Business および Enterprise/Edu ワークスペースでは、ワークスペースのアプリ制御によっても制御されます。OpenAI のアカウントおよびワークスペース制御の概要については、[ChatGPT プランで Codex を使用する](https://help.openai.com/en/articles/11369540-using-codex-with-your-chatgpt-plan)を参照してください。

## コンピューター操作

コンピューター操作には専用のセットアップガイドがあります：
[Codex コンピューター操作](/ja-JP/plugins/codex-computer-use)。

要点：OpenClaw はデスクトップ制御アプリを同梱せず、デスクトップアクション自体も実行しません。Codex app-server を準備し、`computer-use` MCP サーバーが利用可能であることを確認した後、Codex モードのターン中はネイティブ MCP ツール呼び出しを Codex に管理させます。

## ランタイム境界

Codex ハーネスが変更するのは、低レベルの組み込みエージェント実行系だけです。

- OpenClaw 動的ツールはサポートされています。Codex は OpenClaw にそれらのツールの実行を要求するため、OpenClaw は引き続き実行経路に含まれます。
- Codex ネイティブのシェル、パッチ、MCP、およびネイティブアプリツールは Codex が所有します。OpenClaw は、サポートされているリレーを通じて選択されたネイティブイベントを監視またはブロックできますが、ネイティブツールの引数は書き換えません。
- Codex はネイティブ Compaction を所有します。OpenClaw は、チャンネル履歴、検索、`/new`、`/reset`、および将来のモデルまたはハーネス切り替えのためにトランスクリプトのミラーを保持しますが、Codex の Compaction を OpenClaw またはコンテキストエンジンの要約機能に置き換えることはありません。
- メディア生成、メディア理解、TTS、承認、およびメッセージングツールの出力は、引き続き対応する OpenClaw のプロバイダー／モデル設定を経由します。
- `tool_result_persist` は OpenClaw が所有するトランスクリプトのツール結果に適用され、Codex ネイティブのツール結果レコードには適用されません。

フック層、サポート対象の V1 サーフェス、ネイティブ権限処理、キュー制御、Codex フィードバックのアップロード機構、および Compaction の詳細については、[Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime)を参照してください。

## トラブルシューティング

**Codex が通常の `/model` プロバイダーとして表示されない：** 新しい設定では想定どおりです。`openai/gpt-*` モデルを選択し、`plugins.entries.codex.enabled` を有効にして、`plugins.allow` が `codex` を除外していないか確認してください。

**OpenClaw が Codex ではなく組み込みハーネスを使用する：** 有効なルートが、公式の HTTPS Platform Responses または ChatGPT Responses の正確なルートであり、作成者によるリクエストオーバーライドがなく、Codex Plugin がインストールされ有効になっていることを確認してください。`openai/gpt-*` プレフィックスだけでは不十分です。テスト中に厳密に確認するには、プロバイダーまたはモデルの `agentRuntime.id: "codex"` を設定してください。Codex を強制すると、ルートまたはハーネスに互換性がない場合はフォールバックせず失敗します。

**OpenAI Codex ランタイムが API キー経路にフォールバックする：** モデル、ランタイム、選択されたプロバイダー、および障害を示す、機密情報を除去した Gateway の抜粋を収集してください。影響を受けた共同作業者に、OpenClaw ホスト上で次の読み取り専用コマンドを実行するよう依頼してください：

```bash
(
  pattern='openai/gpt-5\.[45]|openai[-]codex|agentRuntime(\.id)?|harnessRuntime|Runtime: OpenAI Codex|legacy OpenAI Codex prefix|resolveSelectedOpenAIRuntimeProvider|candidateProvider[": ]+openai|status[": ]+401|Incorrect API key|No API key|api-key path|API-key path|OAuth'

  if ls /tmp/openclaw/openclaw-*.log >/dev/null 2>&1; then
    grep -E -i -n "$pattern" /tmp/openclaw/openclaw-*.log 2>/dev/null || true
  else
    journalctl --user -u openclaw-gateway --since today --no-pager 2>/dev/null \
      | grep -E -i "$pattern" || true
  fi
) | sed -E \
    -e 's/(Authorization: Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(Bearer )[A-Za-z0-9._~+\/-]+/\1[REDACTED]/Ig' \
    -e 's/(api[_ -]?key[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/(OPENAI_API_KEY[=: ]+)[^ ,}"]+/\1[REDACTED]/Ig' \
    -e 's/sk-[A-Za-z0-9_-]{12,}/sk-[REDACTED]/g' \
    -e 's/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/[EMAIL-REDACTED]/g' \
  | tail -200
```

有用な抜粋には通常、`openai/gpt-5.6-sol` または `openai/gpt-5.6-luna`、`Runtime: OpenAI Codex`、`agentRuntime.id` または `harnessRuntime`、`candidateProvider: "openai"`、および `401`、`Incorrect API key`、または `No API key` の結果が含まれます。修正後の実行では、通常の OpenAI API キー障害ではなく、OpenAI OAuth 経路が表示されるはずです。

**従来の Codex モデル参照設定が残っている場合:** `openclaw doctor --fix` を実行します。
Doctor は従来のモデル参照を `openai/*` に書き換え、古いセッションおよび
エージェント全体のランタイム固定を削除し、既存の認証プロファイルのオーバーライドを保持します。

**app-server が拒否される場合:** バンドルされている `0.145.0` を通じて、
`0.143.0` の安定版 Codex app-server を使用します。OpenClaw は生成されたスキーマを
バンドルされている app-server のバージョンに照らして検証するため、プレリリース版、ビルドサフィックス付きバージョン、
および検証されていない新しいリリースは拒否されます。

**`/codex status` が接続できない場合:** `codex` Plugin が
有効になっていること、許可リストが設定されている場合は `plugins.allow` に
その Plugin が含まれていること、およびカスタムの `appServer.command`、`url`、`authToken`、
またはヘッダーが有効であることを確認します。

**Codex app-server がメモリを過剰に使用する場合:** まず、2 つのプロセスを
区別します。OpenClaw はローカルの Codex app-server を独立した Rust 子プロセスとして実行します。
`NODE_OPTIONS=--max-old-space-size=...` が変更するのは Gateway の Node.js V8
ヒープのみで、Codex の上限を設定したり拡張したりするものではありません。管理対象の Gateway インストールでは、
すでに適応型 V8 ヒープが選択されており、これを増やすと Codex が使用できるホストメモリが減る可能性があります。
Gateway のメモリ負荷については
[Gateway のメモリに関するトラブルシューティング](/ja-JP/gateway/troubleshooting#gateway-exits-during-high-memory-use)
を参照し、Codex 子プロセスについてはホストまたはコンテナのメモリを調査してください。

バンドルされている Codex にはヒープまたは RSS の上限がなく、設定可能なアイドル時のアンロード
遅延もありません。最後のクライアントが購読を解除した後も、非アクティブなスレッドが最大 30 分間
ロードされたままになることがあります。リソースが限られたホストでは、Gateway のヒープを増やす前に、
ネイティブ Codex サブエージェントのファンアウトを減らします。

```json5
{
  plugins: {
    entries: {
      codex: {
        config: {
          appServer: {
            args: ["-c", "agents.max_threads=3", "app-server", "--listen", "stdio://"],
          },
        },
      },
    },
  },
}
```

この設定は、バンドルされている Codex のデフォルトのマルチエージェントバックエンドにおける
ネイティブ子スレッドを制限します。Codex マルチエージェント v2 を明示的に有効にする場合は、
代わりに `features.multi_agent_v2.max_concurrent_threads_per_session=3` を使用します。v2 の
上限にはルートスレッドが含まれ、`agents.max_threads` と組み合わせることはできません。
Codex でより多くの余裕を確保するには、ホスト、コンテナ、または cgroup のメモリ
割り当てを増やします。OS のハードリミットを設定すると、Codex にバックプレッシャーがかかるのではなく、
プロセスが終了する可能性があります。

**モデル検出が遅い場合:** `plugins.entries.codex.config.discovery.timeoutMs` を
小さくするか、検出を無効にします。
[Codex ハーネスリファレンス](/ja-JP/plugins/codex-harness-reference#model-discovery)を参照してください。

**WebSocket トランスポートが即座に失敗する場合:** `appServer.url`、
`authToken`、ヘッダー、およびリモート app-server が同じ Codex
app-server プロトコルバージョンを使用していることを確認します。Codex WebSocket トランスポートは引き続き実験的で
サポート対象外です。管理対象の stdio またはローカル Unix 制御ソケットを使用してください。

**ネイティブシェルまたはパッチツールが `Native hook relay
unavailable` でブロックされる場合:** Codex スレッドが、OpenClaw に登録されなくなった
ネイティブフックリレー ID を使用しようとしています。これはネイティブ Codex フックの
トランスポートの問題であり、ACP バックエンド、プロバイダー、GitHub、またはシェルコマンドの
障害ではありません。影響を受けたチャットで `/new` または `/reset` を使用して新しいセッションを開始し、
無害なコマンドを再試行します。一度は成功しても、次のネイティブツール
呼び出しが再び失敗する場合は、`/new` を一時的な回避策としてのみ使用してください。Codex app-server または
OpenClaw Gateway を再起動した後、プロンプトを新しいセッションにコピーすると、
古いスレッドが破棄され、ネイティブフックの登録が再作成されます。

**Codex ツール呼び出しによって短命なフックプロセスが過剰に作成される場合:** 
`plugins.entries.codex.config.appServer.loopDetectionPreToolUseRelay: false` を設定し、
Gateway を再起動します。これにより無効になるのは、OpenClaw のループ検出と
そのポリシーなしマーカーに使用される Codex `PreToolUse` サブプロセスのみです。必須の
`before_tool_call` および信頼済みツールのポリシーリレーは引き続き有効です。

**Codex 以外のモデルが組み込みハーネスを使用する場合:** プロバイダーまたは
モデルのランタイムポリシーによって別のハーネスにルーティングされない限り、これは想定どおりです。通常の OpenAI 以外の
プロバイダー参照は、`auto` モードでも通常のプロバイダーパスを使用します。

**Computer Use はインストールされているが、ツールが実行されない場合:** 新しいセッションから
`/codex computer-use status` を確認します。ツールが
`Native hook relay unavailable` を報告する場合は、前述のネイティブフックリレーの復旧手順を使用します。
[Codex Computer Use](/ja-JP/plugins/codex-computer-use#troubleshooting)を参照してください。

## 関連項目

- [Codex ハーネスリファレンス](/ja-JP/plugins/codex-harness-reference)
- [Codex ハーネスランタイム](/ja-JP/plugins/codex-harness-runtime)
- [Codex の監視](/ja-JP/plugins/codex-supervision)
- [ネイティブ Codex Plugin](/ja-JP/plugins/codex-native-plugins)
- [Codex Computer Use](/ja-JP/plugins/codex-computer-use)
- [エージェントランタイム](/ja-JP/concepts/agent-runtimes)
- [モデルプロバイダー](/ja-JP/concepts/model-providers)
- [OpenAI プロバイダー](/ja-JP/providers/openai)
- [OpenAI Codex ヘルプ](https://help.openai.com/en/collections/14937394-codex)
- [エージェントハーネス Plugin](/ja-JP/plugins/sdk-agent-harness)
- [Plugin フック](/ja-JP/plugins/hooks)
- [診断のエクスポート](/ja-JP/gateway/diagnostics)
- [ステータス](/ja-JP/cli/status)
- [テスト](/ja-JP/help/testing-live#live-codex-app-server-harness-smoke)
