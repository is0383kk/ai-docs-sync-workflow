---
read_when:
    - 有料 API を呼び出す可能性がある機能を把握したい場合
    - キー、コスト、使用状況の可視性を監査する必要があります
    - /status または /usage のコスト報告について説明している場合
summary: 支出が発生し得る項目、使用されるキー、使用量の確認方法を監査する
title: API の使用量とコスト
x-i18n:
    generated_at: "2026-07-26T10:19:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 22caad8b8fa168739563223b3663a04adceeef7e83576a53dc9cdf885a35750d
    source_path: reference/api-usage-costs.md
    workflow: 16
---

有料プロバイダー API を呼び出せる OpenClaw の機能、各機能が認証情報を読み取る場所、および発生したコストが表示される場所の一覧です。

## コストが表示される場所

**`/status`**（セッションごとのスナップショット）

- 現在のセッションのモデル、コンテキスト使用量、直前の応答のトークン数を表示します。
- OpenClaw に使用量メタデータとアクティブなモデルのローカル価格設定がある場合、直前の応答の**推定コスト**を追加します。これには、Bedrock `aws-sdk` モデルなど、API キーを使用せず明示的に価格が設定されたプロバイダーも含まれます。
- ライブセッションのスナップショットに含まれる情報が少ない場合、`/status` は最新のトランスクリプト使用量エントリから、トークン数、キャッシュ数、アクティブなモデルのラベルを復元します。ゼロではない既存のライブ値はトランスクリプトデータより優先されます。ただし、保存された合計がないか、それより小さい場合は、プロンプト規模のトランスクリプト合計が優先されることがあります。

**`/usage`**（メッセージごとのフッター）

- `/usage full` は各応答に使用量フッターを追加します。ローカル価格が設定され、使用量メタデータが利用可能な場合は、**推定コスト**も含まれます。
- `/usage tokens` はトークン数のみを表示します。サブスクリプション形式の OAuth／トークンおよび CLI ランタイムでは、互換性のある使用量メタデータと明示的なローカル価格の両方が提供されない限り、トークン数のみが表示されます。
- `/usage cost` はローカルのコスト概要を出力し、`/usage off` はフッターを無効にします。
- Gemini CLI に関する注記：`stream-json` と従来の `json` のどちらの出力でも、使用量は `stats` に含まれます。OpenClaw は `stats.cached` を `cacheRead` に正規化し、必要に応じて `stats.input_tokens - stats.cached` から入力トークン数を算出します。

**Control UI → Usage**（セッション横断分析）

- 選択した日付範囲について、トランスクリプトから算出したトークン合計と推定コスト合計を、プロバイダー、モデル、エージェント、チャンネル、トークン種別ごとの内訳とともに表示します。
- 選択範囲の終了日までの、より短いカレンダー期間を比較します。日付が欠けている場合は使用量ゼロの暦日として数えられ、密度の高い期間を作るためにスキップされることはありません。
- 日次グラフのスケールを直接表示します。`√` バッジは、平方根圧縮によって使用量の少ない日も見えるようにしていることを示します。
- これらの合計は、利用可能なローカルセッション履歴を表すものであり、プロバイダーの請求書や累積請求台帳ではありません。一部のエントリに価格設定がない場合、UI に警告が表示されます。

**CLI の使用量期間**（メッセージごとのコストではなく、プロバイダーのクォータ）

- `openclaw status --usage` と `openclaw channels list` は、プロバイダーの**使用量期間**を `X% left` として表示します。
- 現在の使用量期間対応プロバイダー：Anthropic、ClawRouter、DeepSeek、GitHub Copilot、Gemini CLI、MiniMax、OpenAI（ChatGPT／Codex の OAuth／トークン認証を含む）、Xiaomi、z.ai。プロバイダーとフラグの完全な一覧については、[モデル CLI](/ja-JP/cli/models)および[チャンネル CLI](/ja-JP/cli/channels)を参照してください。
- MiniMax の未加工の `usage_percent`／`usagePercent` フィールドは残りクォータを報告するため、OpenClaw は値を反転します。カウントベースのフィールドが存在する場合はそちらが優先されます。応答に `model_remains` 配列が含まれる場合、OpenClaw はチャットモデルのエントリを選択し、必要に応じてタイムスタンプから期間ラベルを算出し、プランラベルにモデル名を含めます。
- 使用量の認証には、利用可能な場合はプロバイダー固有のフックを使用します。それ以外の場合、OpenClaw は認証プロファイル、環境変数、または設定から一致する OAuth／API キー認証情報を探します。

詳しい例については、[トークン使用量とコスト](/ja-JP/reference/token-use)を参照してください。

<Note>
Anthropic は、新しいポリシーを公開しない限り、Claude CLI の再利用（`claude -p` を含む）が認められた統合パターンであることを確認しています。Anthropic はメッセージごとの金額見積もりを公開していないため、`/usage full` では Claude CLI の使用コストを表示できません。
</Note>

## キーの検出方法

- **認証プロファイル**：エージェントごとに `auth-profiles.json` に保存されます。
- **環境変数**：例として `OPENAI_API_KEY`、`BRAVE_API_KEY`、`FIRECRAWL_API_KEY`。
- **設定**：`models.providers.*.apiKey`、`plugins.entries.*.config.webSearch.apiKey`、`plugins.entries.firecrawl.config.webFetch.apiKey`、`memory.search.*`、`talk.providers.*.apiKey`。
- **Skills**：`skills.entries.<name>.apiKey`。キーを Skills のプロセス環境にエクスポートする場合があります。

## キーを使用してコストが発生する可能性がある機能

### コアモデルの応答（チャット＋ツール）

すべての応答またはツール呼び出しは、現在のモデルプロバイダーで実行されます。これは使用量とコストの主な発生源です。OpenClaw のローカル UI 外で請求されるサブスクリプション形式のホスティングプランも含まれます。対象には、OpenAI Codex、Alibaba Cloud Model Studio Coding Plan、MiniMax Coding Plan、Z.AI／GLM Coding Plan、および Extra Usage が有効な Anthropic の Claude ログイン経路があります。

価格設定については[モデル](/ja-JP/providers/models)、表示については[トークン使用量とコスト](/ja-JP/reference/token-use)を参照してください。

### メディア理解（音声／画像／動画）

受信したメディアは、応答パイプラインの実行前にプロバイダー API を通じて要約または文字起こしできます。プロバイダー対応は Plugin ごとに登録され、Plugin の追加に伴って変更されます。現在の一覧と設定については、[メディア理解](/ja-JP/nodes/media-understanding)を参照してください。

### 画像と動画の生成

`image_generate` と `video_generate` は、利用可能な認証済みプロバイダーのいずれかに処理を振り分けます。どちらも、`agents.defaults.mediaModels` エントリが未設定の場合、認証情報に基づいてデフォルトのプロバイダーを推定できます。

現在のプロバイダー一覧については、[画像生成](/ja-JP/tools/image-generation)および[動画生成](/ja-JP/tools/video-generation)を参照してください。

### メモリ埋め込みとセマンティック検索

`memory.search.provider` にリモートアダプター（例：`openai`、`gemini`、`voyage`、`mistral`、`deepinfra`、`github-copilot`、`amazon-bedrock`）が指定されている場合、セマンティックメモリ検索は埋め込み API を使用します。`memory.search.provider = "lmstudio"` または `"ollama"` はローカル／セルフホストサーバーに対して実行され、通常はホスティング料金が発生しません。`memory.search.provider = "local"` はすべてをデバイス上で処理し、API を使用しません。オプションの `memory.search.fallback` プロバイダーで、ローカル埋め込みの失敗を補うことができます。

[メモリ](/ja-JP/concepts/memory)を参照してください。

### Web 検索ツール

`web_search` は、選択したプロバイダーによっては使用料金が発生する可能性があります。各プロバイダーは、まず環境変数からキーを読み取り、次に `plugins.entries.<id>.config.webSearch.apiKey` から読み取ります。

| プロバイダー               | 環境変数                                                                                                                                                             |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Brave Search           | `BRAVE_API_KEY`                                                                                                                                                        |
| DuckDuckGo             | キー不要、非公式、HTML ベース、請求なし                                                                                                                           |
| Exa                    | `EXA_API_KEY`                                                                                                                                                          |
| Firecrawl              | `FIRECRAWL_API_KEY`                                                                                                                                                    |
| Gemini (Google Search) | `GEMINI_API_KEY`                                                                                                                                                       |
| Grok (xAI)             | xAI OAuth プロファイルまたは `XAI_API_KEY`                                                                                                                                     |
| Kimi (Moonshot)        | `KIMI_API_KEY` または `MOONSHOT_API_KEY`                                                                                                                                   |
| MiniMax Search         | `MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY`、`MINIMAX_OAUTH_TOKEN`、または `MINIMAX_API_KEY`                                                                         |
| Ollama Web Search      | 到達可能でサインイン済みのローカルホストではキー不要。`https://ollama.com` の直接検索では `OLLAMA_API_KEY` を使用。認証で保護されたホストでは通常の Ollama プロバイダーの Bearer 認証を再利用 |
| Parallel               | `PARALLEL_API_KEY`                                                                                                                                                     |
| Perplexity Search API  | `PERPLEXITY_API_KEY` または `OPENROUTER_API_KEY`                                                                                                                           |
| SearXNG                | `SEARXNG_BASE_URL`、キー不要／セルフホスト、ホスティング料金なし                                                                                                            |
| Tavily                 | `TAVILY_API_KEY`                                                                                                                                                       |

従来の `tools.web.search.*` 設定パスは互換性シムを通じて引き続き読み込まれますが、推奨される設定方法ではなくなりました。

**Brave Search の無料クレジット**：各プランには、毎月更新される $5／月の無料クレジットが含まれます。Search プランは 1,000 リクエストあたり $5 であるため、このクレジットにより毎月 1,000 リクエストを無料で利用できます。予期しない請求を避けるため、Brave ダッシュボードで使用量上限を設定してください。

[Web ツール](/ja-JP/tools/web)を参照してください。

### Web 取得ツール（Firecrawl）

`web_fetch` は、キー不要のスターターアクセスで Firecrawl を呼び出せます。上限を引き上げるには、`FIRECRAWL_API_KEY`（または `plugins.entries.firecrawl.config.webFetch.apiKey`）を追加します。Firecrawl が設定されていない場合、ツールは直接取得と同梱の `web-readability` Plugin にフォールバックします（有料 API は使用しません）。ローカルでの Readability 抽出を省略するには、`plugins.entries.web-readability.enabled` を無効にします。

[Web ツール](/ja-JP/tools/web)を参照してください。

### プロバイダー使用量のスナップショット（ステータス／健全性）

`openclaw status --usage` と `openclaw models status --json` はプロバイダーの使用量エンドポイントを呼び出し、クォータ期間または認証の健全性を表示します。呼び出し頻度は低いものの、プロバイダー API にはアクセスします。

[モデル CLI](/ja-JP/cli/models)を参照してください。

### Compaction セーフガードによる要約

Compaction セーフガードは現在のモデルを使用してセッション履歴を要約でき、実行時にはプロバイダー API を呼び出します。

[セッション管理と Compaction](/ja-JP/reference/session-management-compaction)を参照してください。

### モデルのスキャン／プローブ

`openclaw models scan` は OpenRouter モデルをプローブでき、プローブが有効な場合は `OPENROUTER_API_KEY` を使用します。

[モデル CLI](/ja-JP/cli/models)を参照してください。

### トーク（音声）

トークモードでは、設定されている場合に ElevenLabs を呼び出せます：`ELEVENLABS_API_KEY` または `talk.providers.elevenlabs.apiKey`。

[トークモード](/ja-JP/nodes/talk)を参照してください。

### Skills（サードパーティ API）

Skills は `apiKey` を `skills.entries.<name>.apiKey` に保存できます。Skills がそのキーを外部 API に使用する場合、コストはその Skills のプロバイダーに従います。

[Skills](/ja-JP/tools/skills)を参照してください。

## 関連項目

- [トークン使用量とコスト](/ja-JP/reference/token-use)
- [プロンプトキャッシュ](/ja-JP/reference/prompt-caching)
- [使用量の追跡](/ja-JP/concepts/usage-tracking)
