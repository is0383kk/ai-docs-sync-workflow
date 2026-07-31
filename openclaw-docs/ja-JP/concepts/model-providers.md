---
read_when:
    - プロバイダー別のモデル設定リファレンスが必要です
    - モデルプロバイダー向けの設定例または CLI オンボーディングコマンドが必要な場合
sidebarTitle: Model providers
summary: モデルプロバイダーの概要（設定例と CLI フロー付き）
title: モデルプロバイダー
x-i18n:
    generated_at: "2026-07-26T09:18:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 51ce1ca5dde28821596ca667619cd860cebda4787993fadb6b0e20fc0e1624ac
    source_path: concepts/model-providers.md
    workflow: 16
---

**LLM／モデルプロバイダー**（WhatsApp／Telegram などのチャットチャネルではありません）のリファレンスです。モデル選択ルールについては、[モデル](/ja-JP/concepts/models)を参照してください。

## クイックルール

<AccordionGroup>
  <Accordion title="モデル参照と CLI ヘルパー">
    - モデル参照では `provider/model` を使用します（例：`opencode/claude-opus-4-6`）。
    - `agents.defaults.models` にはエイリアスとモデルごとの設定が保存されます。`agents.defaults.modelPolicy.allow` は、任意で明示的に指定できるオーバーライド許可リストです。
    - CLI ヘルパー：`openclaw onboard`、`openclaw models list`、`openclaw models set <provider/model>`。
    - `models.providers.*.contextWindow`／`contextTokens`／`maxTokens` はプロバイダーレベルのデフォルトを設定し、`models.providers.*.models[].contextWindow`／`contextTokens`／`maxTokens` はモデルごとにそれらをオーバーライドします。
    - フォールバックルール、クールダウンプローブ、セッションオーバーライドの永続化については、[モデルのフェイルオーバー](/ja-JP/concepts/model-failover)を参照してください。

  </Accordion>
  <Accordion title="プロバイダー認証を追加してもプライマリモデルは変更されない">
    プロバイダーを追加または再認証するとき、`openclaw configure` は既存の `agents.defaults.model.primary` を維持します。`openclaw models auth login` も、`--set-default` を渡さない限り同様に動作します。プロバイダー Plugin が認証設定パッチで推奨デフォルトモデルを返す場合もありますが、プライマリモデルがすでに存在する場合、OpenClaw はそれを「現在のプライマリモデルを置き換える」ものではなく、「このモデルを利用可能にする」ものとして扱います。

    デフォルトモデルを意図的に切り替えるには、`openclaw models set <provider/model>` または `openclaw models auth login --provider <id> --set-default` を使用します。

  </Accordion>
  <Accordion title="OpenAI のプロバイダー／ランタイム分離">
    OpenAI のモデル参照とエージェントランタイムは別々です。

    - `openai/<model>` は正規の OpenAI プロバイダーとモデルを選択します。プレフィックスだけで Codex が選択されることはありません。
    - プロバイダー／モデルのランタイムポリシーが未設定または `auto` の場合、作成者によるリクエストオーバーライドがなく、公式の HTTPS Platform Responses または ChatGPT Responses のルートと完全に一致するときに限り、OpenAI が暗黙的に Codex を選択することがあります。
    - 作成者が定義した Completions アダプター、カスタムエンドポイント、作成者が定義したリクエスト動作を持つルートは OpenClaw 上で実行されます。公式の平文 HTTP エンドポイントは拒否されます。
    - 従来の Codex モデル参照はレガシー設定であり、doctor によって `openai/<model>` に書き換えられます。
    - プロバイダー／モデルの `agentRuntime.id: "openclaw"` は、本来は対象となるルートを明示的に OpenClaw 上に維持します。`agentRuntime.id: "codex"` は Codex を必須とし、有効なルートが Codex と互換性を持たない場合はフェイルクローズします。

    [OpenAI の暗黙的エージェントランタイム](/ja-JP/providers/openai#implicit-agent-runtime)と[Codex ハーネス](/ja-JP/plugins/codex-harness)を参照してください。プロバイダー／ランタイムの分離が分かりにくい場合は、先に[エージェントランタイム](/ja-JP/concepts/agent-runtimes)を参照してください。

    Plugin の自動有効化も同じ境界に従います。暗黙的に Codex と互換性のある有効なルートでは Codex Plugin を有効化できますが、明示的なプロバイダー／モデルの `agentRuntime.id: "codex"` または従来の `codex/<model>` 参照では Codex Plugin が必要です。`openai/*` プレフィックスだけでは有効化されません。

    新規の OpenAI セットアップでは、ルート固有の GPT-5.6 参照を使用します。API キーのセットアップでは
    `openai/gpt-5.6` が選択され（直接 API の短い ID は Sol に解決されます）、
    ChatGPT／Codex OAuth ではネイティブ Codex
    カタログ用の正確な `openai/gpt-5.6-sol` が選択されます。`openai/gpt-5.5` を含む既存の明示的なプライマリは、
    OpenAI 認証の追加または更新時にも維持されます。GPT-5.6 にアクセスできない
    アカウント向けの明示的な復旧選択肢として、GPT-5.5 は
    どちらのランタイムからも引き続き利用できます。

  </Accordion>
  <Accordion title="CLI ランタイム">
    CLI ランタイムも同じ分離を使用します。`anthropic/claude-*` や `google/gemini-*` などの正規モデル参照を選択し、ローカル CLI バックエンドを使用する場合は、プロバイダー／モデルのランタイムポリシーを `claude-cli` または `google-gemini-cli` に設定します。

    従来の `claude-cli/*` および `google-gemini-cli/*` 参照は、ランタイムを別途記録したうえで、正規のプロバイダー参照に移行されます。従来の `codex-cli/*` 参照は `openai/*` に移行され、Codex app-server ルートを使用します。OpenClaw はバンドルされた Codex CLI バックエンドを保持しなくなりました。

  </Accordion>
</AccordionGroup>

## Control UI でプロバイダーを設定する

Control UI で **設定 → モデルプロバイダー** を開き、`models.providers.<id>.apiKey` に保存されているプロバイダー API キーを追加、置換、または削除します。このページでは、各 API キーが OpenClaw の設定と環境変数のどちらから取得されたかを、認証情報を表示せずに識別できます。環境から提供されるキーは、引き続き Gateway プロセスの環境によって管理されます。

**接続をテスト**を使用すると、プロバイダーへのライブプローブを実行し、レイテンシー、または認証、レート制限、請求、タイムアウト、レスポンスに分類されたエラーを確認できます。プローブは実際にプロバイダーへリクエストを送信するため、少量のトークンを消費する場合があります。OAuth およびトークンプロファイルは、プロバイダーカードからログアウトすることもできます。

**デフォルトモデル**カードでは、設定済みのモデルカタログからプライマリモデル、順序付きフォールバック、ユーティリティモデルを管理します。モデルを選択し、既存の `agents.defaults.model` および `agents.defaults.utilityModel` 設定にまとめて保存します。ユーティリティモデルでは、**自動**を選択すると設定が未設定のままになり、**無効**を選択すると空文字列が保存されてユーティリティルーティングが無効になります。

## Plugin が所有するプロバイダー動作

プロバイダー固有のロジックの大部分はプロバイダー Plugin（`registerProvider(...)`）にあり、OpenClaw は汎用的な推論ループを維持します。Plugin は、オンボーディング、モデルカタログ、認証用環境変数のマッピング、トランスポート／設定の正規化、ツールスキーマのクリーンアップ、フェイルオーバーの分類、OAuth の更新、使用量レポート、思考／推論プロファイルなどを所有します。

プロバイダー SDK フックの完全な一覧と、バンドルされた Plugin の例については、[プロバイダー Plugin](/ja-JP/plugins/sdk-provider-plugins)を参照してください。完全にカスタムのリクエスト実行機構を必要とするプロバイダーには、別の、より深い拡張サーフェスが必要です。

<Note>
プロバイダー所有のランナー動作は、リプレイポリシー、ツールスキーマの正規化、ストリームのラップ、トランスポート／リクエストヘルパーなどの明示的なプロバイダーフックに存在します。従来の `ProviderPlugin.capabilities` 静的バッグは互換性専用であり、共有ランナーロジックからは読み取られなくなりました。
</Note>

## API キーのローテーション

<AccordionGroup>
  <Accordion title="キーの取得元と優先順位">
    複数のキーは次の方法で設定します。

    - `OPENCLAW_LIVE_<PROVIDER>_KEY`（単一のライブオーバーライド、最優先）
    - `<PROVIDER>_API_KEYS`（カンマまたはセミコロン区切りのリスト）
    - `<PROVIDER>_API_KEY`（プライマリキー）
    - `<PROVIDER>_API_KEY_*`（番号付きリスト、例：`<PROVIDER>_API_KEY_1`）

    Google プロバイダーでは、`GOOGLE_API_KEY` もフォールバックとして含まれます。キーの選択順序では優先順位が維持され、重複する値は除外されます。

  </Accordion>
  <Accordion title="ローテーションが開始されるタイミング">
    - リクエストは、レート制限レスポンス（たとえば `429`、`rate_limit`、`quota`、`resource exhausted`、`Too many concurrent requests`、`ThrottlingException`、`concurrency limit reached`、`workers_ai ... quota limit exceeded`、または定期的な使用量制限メッセージ）の場合に限り、次のキーで再試行されます。
    - レート制限以外の失敗は直ちに失敗となり、キーのローテーションは試行されません。
    - 候補となるすべてのキーが失敗した場合は、最後の試行から返された最終エラーが返されます。

  </Accordion>
</AccordionGroup>

## 公式プロバイダー Plugin

公式プロバイダー Plugin は、それぞれ独自のモデルカタログ行を公開します。これらのプロバイダーでは `models.providers` モデルエントリは**不要**です。プロバイダー Plugin を有効にし、認証を設定して、モデルを選択します。`models.providers` は、明示的なカスタムプロバイダー、またはタイムアウトなどの限定的なリクエスト設定にのみ使用してください。

### OpenAI

- プロバイダー：`openai`
- 認証：`OPENAI_API_KEY`
- 任意のローテーション：`OPENAI_API_KEYS`、`OPENAI_API_KEY_1`、`OPENAI_API_KEY_2`、および `OPENCLAW_LIVE_OPENAI_KEY`（単一オーバーライド）
- 新規セットアップのデフォルト：`openai/gpt-5.6`。直接 API では、短い ID は Sol に解決されます。
- モデル例：`openai/gpt-5.6`、`openai/gpt-5.6-terra`、`openai/gpt-5.6-luna`、`openai/gpt-5.5`
- 特定のインストールまたは API キーで動作が異なる場合は、`openclaw models list --provider openai` を使用してアカウント／モデルの利用可否を確認してください。
- CLI：`openclaw onboard --auth-choice openai-api-key`
- デフォルトのトランスポートは `auto` です。OpenClaw はトランスポートの選択を共有モデルランタイムに渡します。
- モデルごとに `agents.defaults.models["openai/<model>"].params.transport`（`"sse"`、`"websocket"`、または `"auto"`）でオーバーライドします。
- OpenAI の優先処理は `agents.defaults.models["openai/<model>"].params.serviceTier` で有効にできます。
- `/fast` および `params.fastMode` は、`openai/*` に直接送信される Responses リクエストを、`api.openai.com` 上の `service_tier=priority` にマッピングします。
- 共有の `/fast` 切り替えではなく明示的な階層を指定する場合は、`params.serviceTier` を使用します。
- 非表示の OpenClaw 帰属ヘッダー（`originator`、`version`、`User-Agent`）は、汎用の OpenAI 互換プロキシではなく、`api.openai.com` へのネイティブ OpenAI トラフィックにのみ適用されます。
- ネイティブ OpenAI ルートでは、Responses の `store`、プロンプトキャッシュのヒント、OpenAI 推論互換のペイロード整形も維持されます。プロキシルートでは維持されません。
- `openai/gpt-5.3-codex-spark` は ChatGPT／Codex OAuth でのみ利用できます。OpenAI API キーおよび Azure API キーを直接使用するルートでは拒否されます。

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.6" } } },
}
```

API 組織が GPT-5.6 を公開していない場合は、
`openai/gpt-5.5` を明示的に設定してください。通常のオンボーディングと再認証では、
既存の明示的なプライマリモデルが維持されます。`models auth login --set-default` と
`models set` は、意図的に置き換えるための手段です。

### Anthropic

- プロバイダー：`anthropic`
- 認証：`ANTHROPIC_API_KEY`
- 任意のローテーション：`ANTHROPIC_API_KEYS`、`ANTHROPIC_API_KEY_1`、`ANTHROPIC_API_KEY_2`、および `OPENCLAW_LIVE_ANTHROPIC_KEY`（単一オーバーライド）
- モデル例：`anthropic/claude-opus-5`
- CLI：`openclaw onboard --auth-choice apiKey`
- 公開 Anthropic への直接リクエストでは、共有の `/fast` 切り替えと `params.fastMode` がサポートされます。これには、`api.anthropic.com` に送信される API キーおよび OAuth 認証済みトラフィックが含まれます。OpenClaw はこれを Anthropic の `service_tier`（`auto` と `standard_only`）にマッピングします。
- 推奨される Claude CLI 設定では、モデル参照を正規のまま維持し、CLI
  バックエンドを別途選択します。モデルスコープの
  `agentRuntime.id: "claude-cli"` を指定した `anthropic/claude-opus-5` を使用します。従来の
  `claude-cli/claude-opus-4-7` 参照も互換性のため引き続き機能します。

<Note>
Claude CLI の再利用（`claude -p`）は、OpenClaw で正式に認められた統合手段です。Anthropic のセットアップトークン認証も引き続きサポートされますが、利用可能な場合、OpenClaw は Claude CLI の再利用を優先します。
</Note>

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-5" } } },
}
```

### OpenAI ChatGPT／Codex OAuth

- プロバイダー: `openai`
- 認証: OAuth (ChatGPT)
- 最新のネイティブ Codex app-server ハーネス参照: `openai/gpt-5.6-sol`
- ネイティブ Codex app-server ハーネスのドキュメント: [Codex ハーネス](/ja-JP/plugins/codex-harness)
- レガシーモデル参照: `codex/gpt-*`、`openai-codex/gpt-*`
- Plugin 境界: `openai/*` は OpenAI Plugin を読み込みます。明示的なランタイムポリシーまたはプロバイダー所有の有効なルートによって、ネイティブ Codex app-server Plugin が選択されるかどうかが決まります。
- CLI: `openclaw onboard --auth-choice openai` または `openclaw models auth login --provider openai`
- OpenClaw に組み込まれた ChatGPT Responses トランスポートのデフォルトは `auto`（WebSocket 優先、SSE フォールバック）です。
- `agents.defaults.models["openai/<model>"].params.transport`、`params.serviceTier`、`params.fastMode` は、明示的に設定された組み込みリクエスト設定です。暗黙的なランタイム選択は OpenClaw が担当し、ネイティブ Codex は独自の app-server トランスポートとサービス階層を管理します。
- 非表示の OpenClaw 帰属ヘッダー（`originator`、`version`、`User-Agent`）は、汎用の OpenAI 互換プロキシではなく、`chatgpt.com/backend-api` へのネイティブ Codex トラフィックにのみ付加されます
- 共有の `/fast` トグルは引き続きランタイム制御として利用できます。これは明示的に設定されたモデルパラメーターとは別のものです。
- ネイティブ Codex カタログでは、アカウントのアクセス権に応じて、正確な `openai/gpt-5.6-sol`、`openai/gpt-5.6-terra`、`openai/gpt-5.6-luna` 参照を公開できます。直接 API の短縮 `gpt-5.6` エイリアスをクライアント側で適用することはありません。
- `openai/gpt-5.5` は Codex カタログのネイティブ `contextWindow = 400000` とデフォルトランタイム `contextTokens = 272000` を使用します。ランタイムの上限は `models.providers.openai.models[].contextTokens` で上書きします
- `openai` 認証でサインインし、サブスクリプションを利用する新規セットアップには `openai/gpt-5.6-sol` を使用します。その Codex ワークスペースで GPT-5.6 が公開されていない場合は、`openai/gpt-5.5` を明示的に選択します。
- ほかの条件では対象となるルートを組み込みランタイムに維持するには、プロバイダー/モデル `agentRuntime.id: "openclaw"` を使用します。ランタイムが未設定または `auto` の場合、明示的なリクエスト上書きがない、公式の HTTPS Responses/ChatGPT 互換ルートと正確に一致する場合にのみ、Codex が暗黙的に選択される可能性があります。
- レガシー Codex GPT 参照はレガシー状態であり、稼働中のプロバイダールートではありません。新しいエージェント設定には正規の `openai/*` 参照を使用し、`openclaw doctor --fix` を実行して `codex/*` と `openai-codex/*` の参照を移行します。その際、モデルスコープの `agentRuntime.id: "codex"` によって、ネイティブ Codex のセマンティクスが維持されます。既存の明示的な正規 `openai/gpt-5.5` 選択はアップグレードされません。

```json5
{
  plugins: { entries: { codex: { enabled: true } } },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.6-sol" },
    },
  },
}
```

```json5
{
  models: {
    providers: {
      openai: {
        models: [{ id: "gpt-5.5", contextTokens: 160000 }],
      },
    },
  },
}
```

### その他のサブスクリプション形式のホスティングオプション

<CardGroup cols={3}>
  <Card title="MiniMax" href="/ja-JP/providers/minimax">
    MiniMax Coding Plan の OAuth または API キーによるアクセス。
  </Card>
  <Card title="Qwen Cloud" href="/ja-JP/providers/qwen">
    Qwen Cloud のプロバイダーサーフェス、および Alibaba DashScope と Coding Plan のエンドポイントマッピング。
  </Card>
  <Card title="Z.AI (GLM)" href="/ja-JP/providers/zai">
    Z.AI Coding Plan または汎用 API エンドポイント。
  </Card>
</CardGroup>

### OpenCode

- 認証: `OPENCODE_API_KEY`（または `OPENCODE_ZEN_API_KEY`）
- Zen ランタイムプロバイダー: `opencode`
- Go ランタイムプロバイダー: `opencode-go`
- モデル例: `opencode/claude-opus-4-6`、`opencode-go/kimi-k2.6`
- CLI: `openclaw onboard --auth-choice opencode-zen` または `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini（API キー）

- プロバイダー: `google`
- 認証: `GEMINI_API_KEY`
- オプションのローテーション: `GEMINI_API_KEYS`、`GEMINI_API_KEY_1`、`GEMINI_API_KEY_2`、`GOOGLE_API_KEY` フォールバック、および `OPENCLAW_LIVE_GEMINI_KEY`（単一の上書き）
- モデル例: `google/gemini-3.1-pro-preview`、`google/gemini-3.5-flash`
- 互換性: `google/gemini-3.1-flash-preview` を使用するレガシー OpenClaw 設定は `google/gemini-3-flash-preview` に正規化されます
- エイリアス: `google/gemini-3.1-pro` が受け入れられ、Google の稼働中の Gemini API ID である `google/gemini-3.1-pro-preview` に正規化されます
- CLI: `openclaw onboard --auth-choice gemini-api-key`
- 思考: `/think adaptive` は Google の動的思考を使用します。Gemini 3/3.1 では固定の `thinkingLevel` が省略され、Gemini 2.5 では `thinkingBudget: -1` が送信されます。
- Gemini の直接実行では、プロバイダー固有の `cachedContents/...` ハンドルを転送するために `agents.defaults.models["google/<model>"].params.cachedContent`（またはレガシーの `cached_content`）も使用できます。Gemini のキャッシュヒットは OpenClaw の `cacheRead` として公開されます

### Google Vertex と Gemini CLI

- プロバイダー: `google-vertex`、`google-gemini-cli`
- 認証: Vertex は gcloud ADC を使用し、Gemini CLI は独自の OAuth フローを使用します

<Warning>
OpenClaw の Gemini CLI OAuth は非公式の統合です。サードパーティークライアントの使用後に Google アカウントが制限されたという報告があります。続行する場合は、Google の利用規約を確認し、重要ではないアカウントを使用してください。
</Warning>

Gemini CLI OAuth は、バンドルされた `google` Plugin の一部として提供されます。

<Steps>
  <Step title="Gemini CLI をインストール">
    <Tabs>
      <Tab title="brew">
        ```bash
        brew install gemini-cli
        ```
      </Tab>
      <Tab title="npm">
        ```bash
        npm install -g @google/gemini-cli
        ```
      </Tab>
    </Tabs>
  </Step>
  <Step title="Plugin を有効化">
    ```bash
    openclaw plugins enable google
    ```
  </Step>
  <Step title="ログイン">
    ```bash
    openclaw models auth login --provider google-gemini-cli --set-default
    ```

    デフォルトモデル: `google-gemini-cli/gemini-3-flash-preview`。クライアント ID やシークレットを `openclaw.json` に貼り付ける必要は**ありません**。CLI のログインフローは、Gateway ホストの認証プロファイルにトークンを保存します。

  </Step>
  <Step title="プロジェクトを設定（必要な場合）">
    ログイン後にリクエストが失敗する場合は、Gateway ホストで `GOOGLE_CLOUD_PROJECT` または `GOOGLE_CLOUD_PROJECT_ID` を設定します。
  </Step>
</Steps>

Gemini CLI はデフォルトで `stream-json` を使用します。OpenClaw はアシスタントのストリーム
メッセージを読み取り、`stats.cached` を `cacheRead` に正規化します。レガシーの
`--output-format json` 上書きでは、引き続き `response` から応答テキストを読み取ります。

### Z.AI (GLM)

- プロバイダー: `zai`
- 認証: `ZAI_API_KEY`
- モデル例: `zai/glm-5.2`
- CLI: `openclaw onboard --auth-choice zai-api-key`
  - モデル参照では、正規の `zai/*` プロバイダー ID を使用します。
  - `zai-api-key` は一致する Z.AI エンドポイントを自動検出します。`zai-coding-global`、`zai-coding-cn`、`zai-global`、`zai-cn` は特定のサーフェスを強制します

### Vercel AI Gateway

- プロバイダー: `vercel-ai-gateway`
- 認証: `AI_GATEWAY_API_KEY`
- モデル例: `vercel-ai-gateway/anthropic/claude-opus-4.6`、`vercel-ai-gateway/moonshotai/kimi-k2.6`
- CLI: `openclaw onboard --auth-choice ai-gateway-api-key`

### その他のバンドル済みプロバイダー Plugin

| プロバイダー                              | ID                               | 認証環境変数                                           | モデル例                                               |
| --------------------------------------- | -------------------------------- | ---------------------------------------------------- | ------------------------------------------------------ |
| Arcee                                   | `arcee`                          | `ARCEEAI_API_KEY` または `OPENROUTER_API_KEY`            | `arcee/trinity-large-thinking`                         |
| BytePlus                                | `byteplus` / `byteplus-plan`     | `BYTEPLUS_API_KEY`                                   | `byteplus-plan/ark-code-latest`                        |
| Cerebras                                | `cerebras`                       | `CEREBRAS_API_KEY`                                   | `cerebras/zai-glm-4.7`                                 |
| Chutes                                  | `chutes`                         | `CHUTES_API_KEY` または `CHUTES_OAUTH_TOKEN`             | `chutes/zai-org/GLM-5-TEE`                             |
| ClawRouter                              | `clawrouter`                     | `CLAWROUTER_API_KEY`                                 | `clawrouter/anthropic/claude-sonnet-4-6`               |
| Cohere                                  | `cohere`                         | `COHERE_API_KEY`                                     | `cohere/command-a-plus-05-2026`                        |
| DeepInfra                               | `deepinfra`                      | `DEEPINFRA_API_KEY`                                  | `deepinfra/deepseek-ai/DeepSeek-V4-Flash`              |
| DeepSeek                                | `deepseek`                       | `DEEPSEEK_API_KEY`                                   | `deepseek/deepseek-v4-flash`                           |
| Featherless AI                          | `featherless`                    | `FEATHERLESS_API_KEY`                                | `featherless/Qwen/Qwen3-32B`                           |
| GitHub Copilot                          | `github-copilot`                 | `COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN` | -                                                      |
| GMI Cloud                               | `gmi`                            | `GMI_API_KEY`                                        | `gmi/google/gemini-3.1-flash-lite`                     |
| Groq                                    | `groq`                           | `GROQ_API_KEY`                                       | `groq/llama-3.3-70b-versatile`                         |
| Hugging Face Inference                  | `huggingface`                    | `HUGGINGFACE_HUB_TOKEN` または `HF_TOKEN`                | `huggingface/deepseek-ai/DeepSeek-R1`                  |
| MiniMax                                 | `minimax` / `minimax-portal`     | `MINIMAX_API_KEY` / `MINIMAX_OAUTH_TOKEN`            | `minimax/MiniMax-M3`                                   |
| Mistral                                 | `mistral`                        | `MISTRAL_API_KEY`                                    | `mistral/mistral-large-latest`                         |
| Moonshot                                | `moonshot`                       | `MOONSHOT_API_KEY`                                   | `moonshot/kimi-k2.6`                                   |
| NVIDIA                                  | `nvidia`                         | `NVIDIA_API_KEY`                                     | `nvidia/nvidia/nemotron-3-ultra-550b-a55b`             |
| NovitaAI                                | `novita`                         | `NOVITA_API_KEY`                                     | `novita/deepseek/deepseek-v3-0324`                     |
| [Ollama Cloud](/ja-JP/providers/ollama-cloud) | `ollama-cloud`                   | `OLLAMA_API_KEY`                                     | `ollama-cloud/kimi-k2.6`                               |
| OpenRouter                              | `openrouter`                     | OpenRouter OAuth または `OPENROUTER_API_KEY`             | `openrouter/auto`                                      |
| Qianfan                                 | `qianfan`                        | `QIANFAN_API_KEY`                                    | `qianfan/deepseek-v3.2`                                |
| Tencent TokenHub                        | `tencent-tokenhub`               | `TOKENHUB_API_KEY`                                   | `tencent-tokenhub/hy3-preview`                         |
| Together                                | `together`                       | `TOGETHER_API_KEY`                                   | `together/meta-llama/Llama-3.3-70B-Instruct-Turbo`     |
| Venice                                  | `venice`                         | `VENICE_API_KEY`                                     | -                                                      |
| Vercel AI Gateway                       | `vercel-ai-gateway`              | `AI_GATEWAY_API_KEY`                                 | `vercel-ai-gateway/anthropic/claude-opus-4.6`          |
| Volcano Engine (Doubao)                 | `volcengine` / `volcengine-plan` | `VOLCANO_ENGINE_API_KEY`                             | `volcengine-plan/ark-code-latest`                      |
| xAI                                     | `xai`                            | SuperGrok/X Premium OAuth または `XAI_API_KEY`           | `xai/grok-4.3`                                         |
| Xiaomi                                  | `xiaomi` / `xiaomi-token-plan`   | `XIAOMI_API_KEY` / `XIAOMI_TOKEN_PLAN_API_KEY`       | `xiaomi/mimo-v2.5` / `xiaomi-token-plan/mimo-v2.5-pro` |

#### 知っておくべき特記事項

<AccordionGroup>
  <Accordion title="OpenRouter">
    アプリ帰属ヘッダーと Anthropic `cache_control` マーカーは、検証済みの `openrouter.ai` ルートにのみ適用されます。DeepSeek、Moonshot、ZAI の参照は、OpenRouter が管理するプロンプトキャッシュでキャッシュ TTL の対象になりますが、Anthropic キャッシュマーカーは付与されません。プロキシ形式の OpenAI 互換パスであるため、ネイティブ OpenAI 専用の整形（`serviceTier`、Responses `store`、プロンプトキャッシュのヒント、OpenAI 推論互換性）はスキップされます。Gemini ベースの参照では、プロキシ Gemini の思考署名サニタイズのみ維持されます。
  </Accordion>
  <Accordion title="Kilo Gateway">
    Gemini ベースの参照は同じプロキシ Gemini サニタイズパスに従います。`kilocode/kilo-auto/balanced` およびその他のプロキシ推論をサポートしない参照では、プロキシ推論の注入がスキップされます。
  </Accordion>
  <Accordion title="MiniMax">
    API キーによるオンボーディングでは、M3 および M2.7 のチャットモデル定義が明示的に書き込まれます。画像理解には引き続き、Plugin が所有する `MiniMax-VL-01` メディアプロバイダーが使用されます。
  </Accordion>
  <Accordion title="NVIDIA">
    モデル ID は `nvidia/<vendor>/<model>` 名前空間を使用します（例: `nvidia/nvidia/nemotron-...`）。選択画面ではリテラルの `<provider>/<model-id>` 構成が維持されますが、API に送信される正規キーにはプレフィックスが 1 つだけ付きます。
  </Accordion>
  <Accordion title="xAI">
    xAI Responses パスを使用します。推奨パスは SuperGrok/X Premium OAuth です。API キーも `XAI_API_KEY` または Plugin 設定を介して引き続き使用でき、Grok `web_search` は API キーへのフォールバック前に同じ認証プロファイルを再利用します。利用可能な場合、Grok 4.5 はチャット、コーディング、エージェント型作業向けに選択できます。`grok-4.3` は引き続き地域安全性を考慮した同梱デフォルトです。古い `/fast` および `params.fastMode: true` 設定も、xAI の Grok 4.3 互換リダイレクトを通じて引き続き解決されますが、新しい設定では現在のモデルを直接選択する必要があります。`tool_stream` はデフォルトで有効です。無効にするには `agents.defaults.models["xai/<model>"].params.tool_stream=false` を使用します。
  </Accordion>
</AccordionGroup>

## `models.providers`（カスタム/ベース URL）経由のプロバイダー

**カスタム**プロバイダーまたは OpenAI/Anthropic 互換プロキシを追加するには、`models.providers`（または `models.json`）を使用します。

以下の同梱プロバイダー Plugin の多くは、すでにデフォルトカタログを公開しています。デフォルトのベース URL、ヘッダー、またはモデル一覧を上書きする場合にのみ、明示的な `models.providers.<id>` エントリを使用します。

同梱されているルートおよびカタログで既知のルートは、その所有元プロバイダー Plugin から `compat` 機能を取得します。設定の `compat` ブロックは、カスタムプロバイダー/モデル、またはエンドポイント契約を検証済みの別の `api`/`baseUrl` ルート用です。[カスタムプロバイダー機能ガイド](/ja-JP/gateway/config-tools#custom-provider-capability-declarations)を参照してください。Doctor は、カタログを単に繰り返すだけのレガシー値を削除し、異なる値はオペレーターが確認できるように残します。

Gateway のモデル機能チェックでは、明示的な `models.providers.<id>.models[]` メタデータも読み取ります。カスタムモデルまたはプロキシモデルが画像を受け付ける場合、そのモデルに `input: ["text", "image"]` を設定すると、WebChat および Node 起点の添付ファイルパスで、画像がテキストのみのメディア参照ではなくネイティブモデル入力として渡されます。

`agents.defaults.models["provider/model"]` は、エージェントのエイリアスとモデルごとのメタデータを制御します。これ自体は上書きを制限せず、新しいランタイムモデルも登録しません。カスタムプロバイダーモデルの場合は、少なくとも一致する `id` を持つ `models.providers.<provider>.models[]` も追加します。上書きを制限する場合は、`agents.defaults.modelPolicy.allow` を別途使用します。

### Moonshot AI（Kimi）

オンボーディングの前に `@openclaw/moonshot-provider` をインストールします。ベース URL またはモデルメタデータを上書きする必要がある場合にのみ、明示的な `models.providers.moonshot` エントリを追加します。

- プロバイダー: `moonshot`
- 認証: `MOONSHOT_API_KEY`
- モデル例: `moonshot/kimi-k3`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` または `openclaw onboard --auth-choice moonshot-api-key-cn`

Kimi モデル ID:

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.6`
- `moonshot/kimi-k3`
- `moonshot/kimi-k2.7-code`
- `moonshot/kimi-k2.7-code-highspeed`
- `moonshot/kimi-k2.5`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.6" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.6", name: "Kimi K2.6" }],
      },
    },
  },
}
```

完全なセットアップガイドについては、[Moonshot AI（Kimi + Kimi Coding）](/ja-JP/providers/moonshot)を参照してください。

### Kimi Coding

Kimi Coding は Moonshot AI の Anthropic 互換エンドポイントを使用します。

- プロバイダー: `kimi`
- 認証: `KIMI_API_KEY`
- Kimi K3: `kimi/k3`（256K）または `kimi/k3[1m]`（1M プラン）
- Kimi Code: `kimi/kimi-for-coding`
- Kimi Code HighSpeed: `kimi/kimi-for-coding-highspeed`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-for-coding" } },
  },
}
```

レガシーの `kimi/kimi-code` および `kimi/k2p5` は互換モデル ID として引き続き受け付けられ、Kimi の安定版 API モデル ID に正規化されます。

### Volcano Engine（Doubao）

Volcano Engine（火山引擎）は、中国で Doubao およびその他のモデルへのアクセスを提供します。

- プロバイダー: `volcengine`（コーディング: `volcengine-plan`）
- 認証: `VOLCANO_ENGINE_API_KEY`
- モデル例: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

オンボーディングではデフォルトでコーディング用サーフェスが使用されますが、一般的な `volcengine/*` カタログも同時に登録されます。

オンボーディング/設定のモデル選択画面では、Volcengine の認証選択時に `volcengine/*` と `volcengine-plan/*` の両方の行が優先されます。これらのモデルがまだ読み込まれていない場合、OpenClaw は空のプロバイダー限定選択画面を表示する代わりに、フィルターなしのカタログへフォールバックします。

<Tabs>
  <Tab title="標準モデル">
    - `volcengine/doubao-seed-1-8-251228`（Doubao Seed 1.8）
    - `volcengine/doubao-seed-code-preview-251028`
    - `volcengine/kimi-k2-5-260127`（Kimi K2.5）
    - `volcengine/glm-4-7-251222`（GLM 4.7）
    - `volcengine/deepseek-v3-2-251201`（DeepSeek V3.2）

  </Tab>
  <Tab title="コーディングモデル (volcengine-plan)">
    - `volcengine-plan/ark-code-latest`
    - `volcengine-plan/doubao-seed-code`

  </Tab>
</Tabs>

### BytePlus（国際版）

BytePlus ARK は、海外ユーザー向けに Volcano Engine と同じモデルへのアクセスを提供します。

- プロバイダー: `byteplus`（コーディング: `byteplus-plan`）
- 認証: `BYTEPLUS_API_KEY`
- モデル例: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

オンボーディングではデフォルトでコーディング用サーフェスが使用されますが、一般的な `byteplus/*` カタログも同時に登録されます。

オンボーディングおよび設定のモデル選択画面では、BytePlus の認証選択時に `byteplus/*` と `byteplus-plan/*` の両方の行が優先されます。これらのモデルがまだ読み込まれていない場合、OpenClaw はプロバイダーに限定された空の選択画面を表示する代わりに、フィルタリングされていないカタログにフォールバックします。

<Tabs>
  <Tab title="標準モデル">
    - `byteplus/seed-1-8-251228` (Seed 1.8)
    - `byteplus/kimi-k2-5-260127` (Kimi K2.5)
    - `byteplus/glm-4-7-251222` (GLM 4.7)

  </Tab>
  <Tab title="コーディングモデル (byteplus-plan)">
    - `byteplus-plan/ark-code-latest`
    - `byteplus-plan/kimi-k2.5`
    - `byteplus-plan/glm-4.7`

  </Tab>
</Tabs>

### Synthetic

Synthetic は、`synthetic` プロバイダーを介して Anthropic 互換モデルを提供します。

- プロバイダー: `synthetic`
- 認証: `SYNTHETIC_API_KEY`
- モデル例: `synthetic/hf:MiniMaxAI/MiniMax-M3`
- CLI: `openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M3", name: "MiniMax M3" }],
      },
    },
  },
}
```

### MiniMax

MiniMax はカスタムエンドポイントを使用するため、`models.providers` を介して設定します。

- MiniMax OAuth（グローバル）: `--auth-choice minimax-global-oauth`
- MiniMax OAuth（中国）: `--auth-choice minimax-cn-oauth`
- MiniMax API キー（グローバル）: `--auth-choice minimax-global-api`
- MiniMax API キー（中国）: `--auth-choice minimax-cn-api`
- 認証: `minimax` には `MINIMAX_API_KEY`、`minimax-portal` には `MINIMAX_OAUTH_TOKEN` または `MINIMAX_API_KEY`

設定の詳細、モデルのオプション、設定スニペットについては、[/providers/minimax](/ja-JP/providers/minimax) を参照してください。

<Note>
MiniMax の Anthropic 互換ストリーミングパスでは、明示的に設定しない限り、OpenClaw は M2.x ファミリーの思考をデフォルトで無効にします。MiniMax-M3（および M3.x）は、デフォルトでプロバイダーの省略時／適応型思考パスを維持します。`/fast on` は `MiniMax-M2.7` を `MiniMax-M2.7-highspeed` に書き換えます。
</Note>

Plugin が所有する機能の分割:

- テキスト／チャットのデフォルトは引き続き `minimax/MiniMax-M3`
- 画像生成は `minimax/image-01` または `minimax-portal/image-01`
- 画像理解は、両方の MiniMax 認証パスで Plugin が所有する `MiniMax-VL-01`
- ウェブ検索では引き続きプロバイダー ID `minimax` を使用

### LM Studio

LM Studio は、ネイティブ API を使用するバンドル済みプロバイダー Plugin として提供されます。

- プロバイダー: `lmstudio`
- 認証: `LM_API_TOKEN`
- デフォルトの推論ベース URL: `http://localhost:1234/v1`

次にモデルを設定します（`http://localhost:1234/api/v1/models` が返す ID のいずれかに置き換えてください）。

```json5
{
  agents: {
    defaults: { model: { primary: "lmstudio/openai/gpt-oss-20b" } },
  },
}
```

OpenClaw は、検出と自動読み込みに LM Studio ネイティブの `/api/v1/models` と `/api/v1/models/load` を使用し、推論にはデフォルトで `/v1/chat/completions` を使用します。LM Studio の JIT 読み込み、TTL、自動削除にモデルのライフサイクルを管理させる場合は、`models.providers.lmstudio.params.preload: false` を設定します。設定とトラブルシューティングについては、[/providers/lmstudio](/ja-JP/providers/lmstudio) を参照してください。

### Ollama

Ollama はバンドル済みプロバイダー Plugin として提供され、Ollama のネイティブ API を使用します。

- プロバイダー: `ollama`
- 認証: 不要（ローカルサーバー）
- モデル例: `ollama/llama3.3`
- インストール: [https://ollama.com/download](https://ollama.com/download)

```bash
# Ollama をインストールしてから、モデルを取得します:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

`OLLAMA_API_KEY` でオプトインすると、Ollama はローカルの `http://127.0.0.1:11434` で検出され、バンドル済みプロバイダー Plugin によって Ollama が `openclaw onboard` とモデル選択画面に直接追加されます。オンボーディング、クラウド／ローカルモード、カスタム設定については、[/providers/ollama](/ja-JP/providers/ollama) を参照してください。

### vLLM

vLLM は、ローカル／セルフホストの OpenAI 互換サーバー向けに、バンドル済みプロバイダー Plugin として提供されます。

- プロバイダー: `vllm`
- 認証: 任意（サーバーによって異なります）
- デフォルトのベース URL: `http://127.0.0.1:8000/v1`

ローカルで自動検出を有効にするには、次のように設定します（サーバーが認証を強制しない場合、値は任意です）。

```bash
export VLLM_API_KEY="vllm-local"
```

次にモデルを設定します（`/v1/models` が返す ID のいずれかに置き換えてください）。

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

詳細については、[/providers/vllm](/ja-JP/providers/vllm) を参照してください。

### SGLang

SGLang は、高速なセルフホストの OpenAI 互換サーバー向けに、バンドル済みプロバイダー Plugin として提供されます。

- プロバイダー: `sglang`
- 認証: 任意（サーバーによって異なります）
- デフォルトのベース URL: `http://127.0.0.1:30000/v1`

ローカルで自動検出を有効にするには、次のように設定します（サーバーが認証を強制しない場合、値は任意です）。

```bash
export SGLANG_API_KEY="sglang-local"
```

次にモデルを設定します（`/v1/models` が返す ID のいずれかに置き換えてください）。

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

詳細については、[/providers/sglang](/ja-JP/providers/sglang) を参照してください。

### ローカルプロキシ（LM Studio、vLLM、LiteLLM など）

例（OpenAI 互換）:

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Local" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "${LM_API_TOKEN}",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="デフォルトの任意フィールド">
    カスタムプロバイダーでは、`reasoning`、`input`、`cost`、`contextWindow`、`maxTokens` は任意です。省略した場合、OpenClaw は次の値をデフォルトとして使用します。

    - `reasoning: false`
    - `input: ["text"]`
    - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
    - `contextWindow: 200000`
    - `maxTokens: 8192`

    推奨: プロキシ／モデルの上限に一致する値を明示的に設定してください。

  </Accordion>
  <Accordion title="プロキシルートの整形ルール">
    - 非ネイティブエンドポイント（ホストが `api.openai.com` ではない、空でない任意の `baseUrl`）上の `api: "openai-completions"` では、サポートされていない `developer` ロールによるプロバイダーの 400 エラーを回避するため、OpenClaw は `compat.supportsDeveloperRole: false` を強制します。
    - プロキシ形式の OpenAI 互換ルートでは、OpenAI ネイティブ専用のリクエスト整形もスキップされます。`service_tier`、Responses の `store`、Completions の `store`、プロンプトキャッシュのヒント、OpenAI の推論互換ペイロード整形、非表示の OpenClaw 帰属ヘッダーは使用されません。
    - ベンダー固有のフィールドが必要な OpenAI 互換 Completions プロキシでは、`agents.defaults.models["provider/model"].params.extra_body`（または `extraBody`）を設定し、送信リクエスト本文に追加の JSON をマージします。
    - vLLM のチャットテンプレート制御には、`agents.defaults.models["provider/model"].params.chat_template_kwargs` を設定します。バンドル済み vLLM Plugin は、セッションの思考レベルがオフの場合、`vllm/nemotron-3-*` に対して `enable_thinking: false` と `force_nonempty_content: true` を自動的に送信します。
    - 低速なローカルモデルやリモートの LAN／tailnet ホストには、`models.providers.<id>.timeoutSeconds` を設定します。これにより、エージェント全体のランタイムタイムアウトを増やすことなく、接続、ヘッダー、本文のストリーミング、保護されたフェッチ全体の中止処理を含む、プロバイダーモデルの HTTP リクエスト処理が延長されます。`agents.defaults.timeoutSeconds` または実行固有のタイムアウトのほうが短い場合は、その上限も引き上げてください。プロバイダーのタイムアウトで実行全体を延長することはできません。
    - モデルプロバイダーの HTTP 呼び出しでは、設定されたプロバイダーの `baseUrl` ホスト名に限り、`198.18.0.0/15` および `fc00::/7` 内の Surge、Clash、sing-box の fake-IP DNS 応答が許可されます。カスタム／ローカルのプロバイダーエンドポイントでは、local loopback、LAN、tailnet ホストを含む、設定された正確な `scheme://host:port` オリジンも、保護されたモデルリクエストで信頼されます。これは新しい設定オプションではありません。設定した `baseUrl` により、そのオリジンに限ってリクエストポリシーが拡張されます。fake-IP ホスト名の許可と正確なオリジンの信頼は、互いに独立した仕組みです。その他のプライベート、local loopback、リンクローカル、メタデータの宛先、および異なるポートでは、引き続き明示的な `models.providers.<id>.request.allowPrivateNetwork: true` によるオプトインが必要です。正確なオリジンの信頼を無効にするには、`models.providers.<id>.request.allowPrivateNetwork: false` を設定します。
    - `baseUrl` が空または省略されている場合、OpenClaw は OpenAI のデフォルト動作（`api.openai.com` に解決されます）を維持します。
    - 安全のため、明示的な `compat.supportsDeveloperRole: true` も、非ネイティブの `openai-completions` エンドポイントでは引き続き上書きされます。
    - 非直接エンドポイント（正規の `anthropic` 以外のプロバイダー、またはホストが公開 `api.anthropic.com` エンドポイントではないカスタム `models.providers.anthropic.baseUrl`）上の `api: "anthropic-messages"` では、カスタム Anthropic 互換プロキシがサポートされていないベータフラグを拒否しないように、OpenClaw は `claude-code-20250219`、`interleaved-thinking-2025-05-14`、OAuth マーカーなどの暗黙的な Anthropic ベータヘッダーを抑制します。プロキシで特定のベータ機能が必要な場合は、`models.providers.<id>.headers["anthropic-beta"]` を明示的に設定します。

  </Accordion>
</AccordionGroup>

## CLI の例

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

完全な設定例については、[設定](/ja-JP/gateway/configuration) も参照してください。

## 関連項目

- [設定リファレンス](/ja-JP/gateway/config-agents#agent-defaults) - モデル設定キー
- [モデルのフェイルオーバー](/ja-JP/concepts/model-failover) - フォールバックチェーンと再試行動作
- [モデル](/ja-JP/concepts/models) - モデル設定とエイリアス
- [プロバイダー](/ja-JP/providers) - プロバイダー別の設定ガイド
