---
read_when:
    - web_search を有効化または設定する場合
    - x_search を有効化または設定する必要があります
    - 検索プロバイダーを選択する必要があります
    - 自動検出とプロバイダーの選択について理解したい場合
sidebarTitle: Web Search
summary: web_search、x_search、web_fetch -- Web を検索、X の投稿を検索、またはページのコンテンツを取得
title: ウェブ検索
x-i18n:
    generated_at: "2026-07-26T09:25:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 997e51064b0cd08d0f30987aa038e2f4a98da22f1094974b45f59c18491bd979
    source_path: tools/web.md
    workflow: 16
---

`web_search` は設定済みのプロバイダーでウェブを検索し、
正規化された結果を返します。結果はクエリごとに 15 分間キャッシュされます（設定可能）。OpenClaw
には、X（旧 Twitter）の投稿向けの `x_search` と、
軽量な URL 取得向けの `web_fetch` も同梱されています。`web_fetch` は常にローカルで実行され、Grok がプロバイダーの場合、`web_search` は
xAI Responses 経由で処理されます。また、`x_search` は常に
xAI Responses を使用します。

<Info>
  `web_search` は軽量な HTTP ツールであり、ブラウザー自動化ツールではありません。
  JS を多用するサイトやログインには、[ウェブブラウザー](/ja-JP/tools/browser)を使用してください。
  特定の URL を取得するには、[ウェブ取得](/ja-JP/tools/web-fetch)を使用してください。
</Info>

## クイックスタート

<Steps>
  <Step title="プロバイダーを選択">
    プロバイダーを選択し、必要なセットアップを完了します。一部のプロバイダーは
    キーなしで利用できますが、API キーが必要なものもあります。詳細については、
    以下のプロバイダーページを参照してください。
  </Step>
  <Step title="設定">
    ```bash
    openclaw configure --section web
    ```
    これにより、プロバイダーと必要な認証情報が保存されます。API ベースの
    プロバイダーでは、代わりにプロバイダーの環境変数（例：
    `BRAVE_API_KEY`）を設定し、この手順を省略できます。
  </Step>
  <Step title="使用">
    ```javascript
    await web_search({ query: "OpenClaw plugin SDK" });
    ```

    X の投稿の場合：

    ```javascript
    await x_search({ query: "dinner recipes" });
    ```

  </Step>
</Steps>

## プロバイダーの選択

<CardGroup cols={2}>
  <Card title="Brave Search" icon="shield" href="/ja-JP/tools/brave-search">
    スニペット付きの構造化された結果。`llm-context` モードと国／言語フィルターをサポートします。無料枠を利用できます。
  </Card>
  <Card title="Codex Hosted Search" icon="search" href="/ja-JP/plugins/codex-harness">
    Codex app-server アカウントを通じて、根拠に基づき AI が合成した回答を提供します。
  </Card>
  <Card title="DuckDuckGo" icon="bird" href="/ja-JP/tools/duckduckgo-search">
    キー不要のプロバイダーです。API キーは必要ありません。HTML ベースの非公式な統合です。
  </Card>
  <Card title="Exa" icon="brain" href="/ja-JP/tools/exa-search">
    コンテンツ抽出（ハイライト、テキスト、要約）を備えたニューラル検索とキーワード検索。
  </Card>
  <Card title="Firecrawl" icon="flame" href="/ja-JP/tools/firecrawl">
    構造化された結果。詳細な抽出には `firecrawl_search` および `firecrawl_scrape` と組み合わせるのが最適です。
  </Card>
  <Card title="Gemini" icon="sparkles" href="/ja-JP/tools/gemini-search">
    Google Search のグラウンディングを通じて、引用付きの AI 合成回答を提供します。
  </Card>
  <Card title="Grok" icon="zap" href="/ja-JP/tools/grok-search">
    xAI のウェブグラウンディングを通じて、引用付きの AI 合成回答を提供します。
  </Card>
  <Card title="Kimi" icon="moon" href="/ja-JP/tools/kimi-search">
    Moonshot ウェブ検索を通じて、引用付きの AI 合成回答を提供します。グラウンディングされていないチャットへのフォールバックは明示的に失敗します。
  </Card>
  <Card title="MiniMax Search" icon="globe" href="/ja-JP/tools/minimax-search">
    MiniMax Token Plan 検索 API を通じた構造化された結果。
  </Card>
  <Card title="Ollama Web Search" icon="globe" href="/ja-JP/tools/ollama-search">
    サインイン済みのローカル Ollama ホストまたはホスト型 Ollama API を通じて検索します。
  </Card>
  <Card title="Parallel" icon="layer-group" href="/ja-JP/tools/parallel-search">
    有料の Parallel Search API（`PARALLEL_API_KEY`）。より高いレート制限と目的に応じた調整を提供します。
  </Card>
  <Card title="Parallel Search（無料）" icon="layer-group" href="/ja-JP/tools/parallel-search">
    キー不要のオプトイン方式。Parallel の無料 Search MCP で、LLM 向けに最適化された高密度の抜粋を提供し、API キーは不要です。
  </Card>
  <Card title="Perplexity" icon="search" href="/ja-JP/tools/perplexity-search">
    コンテンツ抽出制御とドメインフィルタリングを備えた構造化された結果。
  </Card>
  <Card title="SearXNG" icon="server" href="/ja-JP/tools/searxng-search">
    セルフホスト型のメタ検索。API キーは必要ありません。Google、Bing、DuckDuckGo などの結果を集約します。
  </Card>
  <Card title="Tavily" icon="globe" href="/ja-JP/tools/tavily">
    検索深度、トピックフィルタリング、および URL 抽出用の `tavily_extract` を備えた構造化された結果。
  </Card>
</CardGroup>

### プロバイダーの比較

| プロバイダー                                         | 結果の形式                                                   | フィルター                                          | API キー                                                                                 |
| ------------------------------------------------ | -------------------------------------------------------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------- |
| [Brave](/ja-JP/tools/brave-search)                     | 構造化されたスニペット                                            | 国、言語、時間、`llm-context` モード      | `BRAVE_API_KEY`                                                                         |
| [Codex Hosted Search](/ja-JP/plugins/codex-harness)    | AI 合成回答 + ソース URL                                   | ドメイン、コンテキストサイズ、ユーザーの所在地             | なし。Codex/OpenAI のサインインを使用                                                         |
| [DuckDuckGo](/ja-JP/tools/duckduckgo-search)           | 構造化されたスニペット                                            | --                                               | なし（キー不要）                                                                         |
| [Exa](/ja-JP/tools/exa-search)                         | 構造化された結果 + 抽出コンテンツ                                         | ニューラル／キーワードモード、日付、コンテンツ抽出    | `EXA_API_KEY`                                                                           |
| [Firecrawl](/ja-JP/tools/firecrawl)                    | 構造化されたスニペット                                            | `firecrawl_search` ツール経由                      | `FIRECRAWL_API_KEY`                                                                     |
| [Gemini](/ja-JP/tools/gemini-search)                   | AI 合成回答 + 引用                                     | --                                               | `GEMINI_API_KEY`                                                                        |
| [Grok](/ja-JP/tools/grok-search)                       | AI 合成回答 + 引用                                     | --                                               | xAI OAuth、`XAI_API_KEY`、または `plugins.entries.xai.config.webSearch.apiKey`              |
| [Kimi](/ja-JP/tools/kimi-search)                       | AI 合成回答 + 引用。グラウンディングされていないチャットへのフォールバック時は失敗 | --                                               | `KIMI_API_KEY` / `MOONSHOT_API_KEY`                                                     |
| [MiniMax Search](/ja-JP/tools/minimax-search)          | 構造化されたスニペット                                            | リージョン（`global` / `cn`）                         | `MINIMAX_CODE_PLAN_KEY` / `MINIMAX_CODING_API_KEY` / `MINIMAX_OAUTH_TOKEN`              |
| [Ollama Web Search](/ja-JP/tools/ollama-search)        | 構造化されたスニペット                                            | --                                               | サインイン済みのローカルホストでは不要。`https://ollama.com` の直接検索には `OLLAMA_API_KEY` |
| [Parallel](/ja-JP/tools/parallel-search)               | LLM コンテキスト向けに順位付けされた高密度の抜粋                          | --                                               | `PARALLEL_API_KEY`（有料）                                                               |
| [Parallel Search（無料）](/ja-JP/tools/parallel-search) | LLM コンテキスト向けに順位付けされた高密度の抜粋                          | --                                               | なし（無料の Search MCP）                                                                  |
| [Perplexity](/ja-JP/tools/perplexity-search)           | 構造化されたスニペット                                            | 国、言語、時間、ドメイン、コンテンツ制限 | `PERPLEXITY_API_KEY` / `OPENROUTER_API_KEY`                                             |
| [SearXNG](/ja-JP/tools/searxng-search)                 | 構造化されたスニペット                                            | カテゴリー、言語                             | なし（セルフホスト）                                                                      |
| [Tavily](/ja-JP/tools/tavily)                          | 構造化されたスニペット                                            | `tavily_search` ツール経由                         | `TAVILY_API_KEY`                                                                        |

## 結果の形式

`web_search` は、コアツールの境界ですべての同梱および外部 Plugin プロバイダーを
正規化します。呼び出し元は、次の閉じた形式のいずれか 1 つだけを受け取ります。

```typescript
type WebSearchOutput =
  | {
      kind: "error";
      provider: string;
      error: "provider_error";
      message: string;
      docs?: string;
    }
  | {
      kind: "results";
      provider: string;
      query: string;
      count: number;
      tookMs?: number;
      results: Array<{
        title: string;
        url: string;
        snippet?: string;
        published?: string;
        siteName?: string;
      }>;
      externalContent: {
        untrusted: true;
        source: "web_search";
        wrapped: true;
        provider: string;
      };
      cached?: true;
    }
  | {
      kind: "answer";
      provider: string;
      query: string;
      tookMs?: number;
      content: string;
      citations?: Array<{ url: string; title?: string }>;
      externalContent: {
        untrusted: true;
        source: "web_search";
        wrapped: true;
        provider: string;
      };
      cached?: true;
    }
  | {
      kind: "raw";
      provider: string;
      data: unknown;
    };
```

構造化プロバイダーは `kind: "results"` を使用し、合成プロバイダーは
`kind: "answer"` を使用します。ペイロードがどちらの形式にも一致しない外部 Plugin プロバイダーは、
互換性のため `kind: "raw"` としてそのまま渡されます。生のスコア、抜粋、関連検索、インライン引用の
オフセット、モデル ID、セッションメタデータなど、プロバイダー固有の
フィールドは、正規化された分岐では渡されません。より豊富な応答が
ワークフローの一部である場合は、プロバイダー専用のツールを使用してください。

`externalContent.wrapped: true` は、境界自体が真であることを保証する信頼マーカーです。
プロバイダーの文章（`title`、`snippet`、`siteName`、`content`、引用の
タイトル、エラーの `message`）から既存のエンベロープ行をすべて除去し、
コア境界で正確に 1 回だけ再ラップするため、プロバイダーのメタデータが
このマーカーを偽装することはできません。`query` は常にリクエストされたクエリであり、引用および結果の URL は
http(s) として解析可能でなければなりません。`published` は ISO 日付形式でなければならず、URL は正規化して出力されます。また、
`error` キーを含むペイロードは常に `kind: "error"` として報告され、
生のプロバイダーコードはラップされたメッセージ内に保持されます。生のパススルー
ペイロードでは、プロバイダーが設定したマーカーがそのまま保持されます。

## 自動検出

ドキュメントおよびセットアップフロー内のプロバイダー一覧はアルファベット順です。自動検出では、
それとは別の固定された優先順位を使用し、認証情報（`requiresCredential !== false`）が必要なプロバイダーは、
その認証情報が設定済みの場合にのみ選択します。
`provider` が設定されていない場合、OpenClaw は以下の順序でプロバイダーを確認し、
最初に利用可能なものを使用します。

最初に API ベースのプロバイダー：

1. **Brave** -- `BRAVE_API_KEY` または `plugins.entries.brave.config.webSearch.apiKey`（順序 10）
2. **MiniMax Search** -- `MINIMAX_CODE_PLAN_KEY` / `MINIMAX_CODING_API_KEY` / `MINIMAX_OAUTH_TOKEN` / `MINIMAX_API_KEY` または `plugins.entries.minimax.config.webSearch.apiKey`（順序 15）
3. **Gemini** -- `plugins.entries.google.config.webSearch.apiKey`、`GEMINI_API_KEY`、または `models.providers.google.apiKey`（順序 20）
4. **Grok** -- xAI OAuth、`XAI_API_KEY`、または `plugins.entries.xai.config.webSearch.apiKey`（順序 30）
5. **Kimi** -- `KIMI_API_KEY` / `MOONSHOT_API_KEY` または `plugins.entries.moonshot.config.webSearch.apiKey`（順序 40）
6. **Perplexity** -- `PERPLEXITY_API_KEY` / `OPENROUTER_API_KEY` または `plugins.entries.perplexity.config.webSearch.apiKey`（順序 50）
7. **Firecrawl** -- `FIRECRAWL_API_KEY` または `plugins.entries.firecrawl.config.webSearch.apiKey`（順序 60）
8. **Exa** -- `EXA_API_KEY` または `plugins.entries.exa.config.webSearch.apiKey`。オプションの `plugins.entries.exa.config.webSearch.baseUrl` で Exa エンドポイントを上書きできます（順序 65）
9. **Tavily** -- `TAVILY_API_KEY` または `plugins.entries.tavily.config.webSearch.apiKey`（順序 70）
10. **Parallel** -- `PARALLEL_API_KEY` または `plugins.entries.parallel.config.webSearch.apiKey` を介した有料の Parallel Search API。オプションの `plugins.entries.parallel.config.webSearch.baseUrl` でエンドポイントを上書きできます（順序 75）

その後に、エンドポイントが設定されたプロバイダーが続きます。

11. **SearXNG** -- `SEARXNG_BASE_URL` または `plugins.entries.searxng.config.webSearch.baseUrl`（順序 200）

**Parallel Search (Free)**、**DuckDuckGo**、
**Ollama Web Search**、**Codex Hosted Search** など、キー不要のプロバイダーは、
内部的な順序値を持っていても自動検出では選択されません。これらが使用されるのは、
`tools.web.search.provider` または
`openclaw configure --section web` を介して明示的に選択した場合のみです。API 対応のプロバイダーが
設定されていないという理由だけで、OpenClaw が管理対象の
`web_search` クエリをキー不要のプロバイダーに送信することはありません。

OpenAI Responses モデルは例外です。`tools.web.search.provider`
が未設定の場合、これらのモデルは上記の管理対象プロバイダーではなく、OpenAI ネイティブの
Web 検索を使用します（以下を参照）。代わりに管理対象の経路を使用するには、
`tools.web.search.provider` を
`parallel-free`（または別のプロバイダー）に設定します。

<Note>
  すべてのプロバイダーのキーフィールドは SecretRef オブジェクトに対応しています。
  `plugins.entries.<plugin>.config.webSearch.apiKey` 配下の Plugin スコープの SecretRef は、
  インストール済みの API 対応 Web 検索プロバイダーについて解決されます。対象には Brave、Exa、Firecrawl、
  Gemini、Grok、Kimi、MiniMax、Parallel、Perplexity、Tavily が含まれ、
  `tools.web.search.provider` でプロバイダーを明示的に指定した場合でも、
  自動検出で選択された場合でも同様です。自動検出モードでは、OpenClaw は選択された
  プロバイダーのキーだけを解決します。選択されなかった SecretRef は非アクティブなままなので、
  使用していないプロバイダーの解決コストを負担せずに、
  複数のプロバイダーを設定しておけます。
</Note>

## OpenAI ネイティブ Web 検索

OpenClaw の Web 検索が有効で、管理対象プロバイダーが固定されていない場合、直接接続の OpenAI Responses モデル
（`api: "openai-responses"`、プロバイダー `openai`、
ベース URL なし、または公式 OpenAI API ベース URL）は、OpenAI がホストする
`web_search` ツールを自動的に使用します。これはバンドルされた
OpenAI Plugin によるプロバイダー所有の動作であり、OpenAI 互換プロキシのベース URL や Azure
経路には適用されません。OpenAI モデルで管理対象の `web_search` ツールを
引き続き使用するには、`tools.web.search.provider` を `brave` などの別のプロバイダーに設定します。
管理対象検索と OpenAI ネイティブ検索の両方を無効にするには、
`tools.web.search.enabled: false` を設定します。

## Codex ネイティブ Web 検索

Web 検索が有効で、管理対象プロバイダーが選択されていない場合、Codex app-server ランタイムは、
Codex がホストする `web_search` ツールを自動的に使用します。ネイティブのホスト型検索と
OpenClaw の管理対象 `web_search` 動的ツールは相互排他的であるため、
管理対象検索でネイティブのドメイン制限を回避することはできません。ホスト型検索が利用できない場合、
明示的に無効化されている場合、または選択した管理対象プロバイダーに置き換えられている場合、
OpenClaw は管理対象ツールを使用します。OpenClaw は Codex のスタンドアロン
`web.run` 拡張機能を無効（`features.standalone_web_search: false`）のままにします。
これは、本番環境の app-server トラフィックが、そのユーザー定義の `web`
名前空間を拒否するためです。

- `tools.web.search.openaiCodex` 配下でネイティブ検索を設定する
- `tools.web.search.provider: "codex"` を設定すると、任意の親モデルの管理対象
  `web_search` プロバイダーとして Codex Hosted Search がプロビジョニングされます。各呼び出しでは、
  制限付きの一時的な Codex app-server ターンが実行され、Codex がホスト型
  `webSearch` 項目を出力しなかった場合は失敗します。
- `mode: "cached"` がデフォルトの設定ですが、Codex は制限のない
  app-server ターンではこれをライブの外部アクセスとして解決します。ライブアクセスを明示的に要求するには、
  `"live"` を設定します
- OpenClaw の管理対象 `web_search` を使用するには、
  `tools.web.search.provider` を `brave` などの管理対象プロバイダーに設定します
- Codex ホスト型検索をオプトアウトするには、`tools.web.search.openaiCodex.enabled: false` を設定します。
  その他の管理対象プロバイダーは引き続き利用できます
- Codex ネイティブツールの範囲を制限しても、管理対象の `web_search` は
  引き続き利用できます
- `allowedDomains` が設定されている場合、ホスト型検索を利用できなければ、
  自動的な管理対象フォールバックはフェイルクローズとなり、ネイティブの許可リストを回避できません
- ツールを無効化した LLM のみの実行では、ネイティブ検索と管理対象検索の両方が無効になります
- `tools.web.search.enabled: false` は、管理対象検索とネイティブ検索の両方を無効にします

永続的に有効な Codex 検索ポリシーを変更すると、新しいバインド済みスレッドが開始されるため、
すでに読み込まれている app-server スレッドが古いホスト型検索アクセスを維持することはありません。
ターン単位の一時的な制限では、一時的な制限付きスレッドが使用され、
後で再開できるよう既存のバインディングが維持されます。

OpenAI ChatGPT Responses に直接送信されるトラフィックでも、OpenAI がホストする
`web_search` ツールを使用できます。この別経路は、
`tools.web.search.openaiCodex.enabled: true` を通じてオプトインした場合に限られ、
`api: "openai-chatgpt-responses"` を使用する対象の
`openai/*` モデルにのみ適用されます。

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        // オプション: Codex 以外の親モデルからも Codex Hosted Search を使用します。
        provider: "codex",
        openaiCodex: {
          enabled: true,
          mode: "cached",
          allowedDomains: ["example.com"],
          contextSize: "high",
          userLocation: {
            country: "US",
            city: "New York",
            timezone: "America/New_York",
          },
        },
      },
    },
  },
}
```

Codex ネイティブ検索に対応していないランタイムとプロバイダーでは、Codex は
OpenClaw の動的ツール名前空間を介して、管理対象の `web_search` フォールバックを使用できます。
Codex ホスト型検索ではなく、OpenClaw のプロバイダー固有のネットワーク制御が必要な場合は、
管理対象プロバイダーを明示的に指定してください。

`provider: "codex"` を選択すると、バンドルされた `codex` Plugin が有効になり、
上記と同じ `tools.web.search.openaiCodex` の制限が使用されます。まず
`openclaw models auth login --provider openai` で Codex app-server を認証してください。
親エージェントは任意のモデルまたはランタイムを使用できます。Codex を介して実行されるのは、
制限付きの検索ワーカーだけです。

## ネットワークの安全性

管理対象 HTTP `web_search` プロバイダーの呼び出しでは、
現在のプロバイダー自身のホスト名に限定された OpenClaw の保護付きフェッチ経路が使用されます。
そのホスト名に限り、OpenClaw は `198.18.0.0/15` と `fc00::/7` に含まれる
Surge、Clash、sing-box の fake-IP DNS 応答を許可します。それ以外のプライベート、ループバック、
リンクローカル、メタデータの宛先は引き続きブロックされます。Codex Hosted Search は例外です。
その制限付きワーカーは、ネットワークアクセスを Codex app-server がホストする
`web_search` ツールに委任します。

この自動許可は、任意の `web_fetch` URL には適用されません。
`web_fetch` では、信頼できるプロキシがこれらの合成範囲を所有している場合に限り、
`tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` と
`tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange` を明示的に有効にしてください。

## 設定

```json5
{
  tools: {
    web: {
      search: {
        enabled: true, // デフォルト: true
        provider: "brave", // または自動検出する場合は省略
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
    },
  },
}
```

プロバイダー固有の設定（API キー、ベース URL、モード）は、
`plugins.entries.<plugin>.config.webSearch.*` 配下に置かれます。Gemini は専用の Web 検索設定と
`GEMINI_API_KEY` の後に、優先度の低いフォールバックとして
`models.providers.google.apiKey` と `models.providers.google.baseUrl` を再利用することもできます。
例については各プロバイダーのページを参照してください。
Grok は `openclaw models auth login
--provider xai --method oauth` の xAI OAuth 認証プロファイルを再利用することもできます。API キー設定は引き続きフォールバックです。

`tools.web.search.provider` は、バンドル済みおよびインストール済みの Plugin マニフェストで
宣言されている Web 検索プロバイダー ID に対して検証されます。`"brvae"` のような
入力ミスは、自動検出へ暗黙的にフォールバックするのではなく、設定検証で失敗します。
設定されたプロバイダーに、サードパーティー Plugin のアンインストール後に残った
`plugins.entries.<plugin>` ブロックなど、古い Plugin の情報しかない場合、
OpenClaw は起動時の堅牢性を維持しながら警告を報告します。その場合は Plugin を再インストールするか、
`openclaw doctor --fix` を実行して古い設定をクリーンアップできます。

`web_fetch` のフォールバックプロバイダー選択は別個に行われます。

- `tools.web.fetch.provider` で選択します
- または、そのフィールドを省略すると、OpenClaw が設定済みの認証情報から
  最初に利用可能な Web フェッチプロバイダーを自動検出します
- サンドボックス化されていない `web_fetch` では、
  `contracts.webFetchProviders` を宣言するインストール済み Plugin プロバイダーを使用できます。
  サンドボックス化されたフェッチでは、バンドル済みプロバイダーと検証済みの公式 Plugin インストールが許可されますが、
  サードパーティーの外部 Plugin は除外されます
- 現在、公式 Firecrawl Plugin は、バンドルされた `webFetchProviders` の
  唯一のコントリビューターであり、
  `plugins.entries.firecrawl.config.webFetch.*` 配下で設定されます

`openclaw onboard` または
`openclaw configure --section web` で **Kimi** を選択すると、OpenClaw は次の項目も確認できます。

- Moonshot API のリージョン（`https://api.moonshot.ai/v1` または `https://api.moonshot.cn/v1`）
- デフォルトの Kimi Web 検索モデル（デフォルトは `kimi-k2.6`）

`x_search` では、`plugins.entries.xai.config.xSearch.*` を設定します。チャットと同じ
xAI 認証プロファイル、または Grok Web 検索で使用される `XAI_API_KEY` / Plugin の
Web 検索認証情報を使用します。
従来の `tools.web.x_search.*` 設定は、`openclaw doctor --fix` によって自動移行されます。
`openclaw onboard` または `openclaw configure --section web` で Grok を選択すると、
OpenClaw は Grok のセットアップ完了直後に、同じ認証情報を使用するオプションの
`x_search` セットアップも提示します。これは Grok 経路内の独立した後続ステップであり、
トップレベルで独立した Web 検索プロバイダーの選択肢ではありません。別のプロバイダーを選択した場合、
OpenClaw は `x_search` のプロンプトを表示しません。

### API キーの保存

<Tabs>
  <Tab title="設定ファイル">
    `openclaw configure --section web` を実行するか、キーを直接設定します。

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "YOUR_KEY", // pragma: allowlist secret
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="環境変数">
    Gateway プロセスの環境にプロバイダーの環境変数を設定します。

    ```bash
    export BRAVE_API_KEY="YOUR_KEY"
    ```

    Gateway のインストール環境では、`~/.openclaw/.env` に配置します。
    [環境変数](/ja-JP/help/faq#env-vars-and-env-loading)を参照してください。

  </Tab>
</Tabs>

## ツールパラメーター

| パラメーター             | 説明                                                        |
| --------------------- | ------------------------------------------------------------------ |
| `query`               | 検索クエリ（必須）                                            |
| `count`               | 返す結果の数（1〜10、デフォルト: 5）                               |
| `country`             | 2文字の ISO 国コード（例: "US"、"DE"）                        |
| `language`            | ISO 639-1 言語コード（例: "en"、"de"）                          |
| `search_lang`         | 検索言語コード（Brave のみ）                                  |
| `freshness`           | 時間フィルター: `day`、`week`、`month`、または `year`                     |
| `date_after`          | この日付より後の結果（YYYY-MM-DD）                               |
| `date_before`         | この日付より前の結果（YYYY-MM-DD）                              |
| `ui_lang`             | UI 言語コード（Brave のみ）                                      |
| `domain_filter`       | ドメイン許可リスト／拒否リストの配列（Perplexity のみ）                  |
| `max_tokens`          | コンテンツ全体のトークン予算（ネイティブ Perplexity Search API のみ）      |
| `max_tokens_per_page` | ページごとの抽出トークン上限（ネイティブ Perplexity Search API のみ） |

<Warning>
  すべてのパラメーターがすべてのプロバイダーで機能するわけではありません。Brave の `llm-context` モードでは
  `ui_lang` が拒否されます。また、Brave のカスタム鮮度範囲には開始日と終了日の両方が必要なため、
  `date_before` には `date_after` も必要です。
  Gemini、Grok、Kimi は、引用付きの統合された回答を 1 つ返します。共有ツールとの互換性のために
  `count` を受け付けますが、根拠付き回答の形式は変わりません。
  Gemini は `day` の鮮度を新しさのヒントとして扱います。より広い鮮度の値や
  明示的な日付を指定すると、Google Search のグラウンディング時間範囲が設定されます。
  Sonar/OpenRouter 互換パス（`plugins.entries.perplexity.config.webSearch.baseUrl` /
  `model` または `OPENROUTER_API_KEY`）を使用する場合、Perplexity も同様に動作します。
  このパスでは `max_tokens` と `max_tokens_per_page` のサポートも無効になります。
  SearXNG は、信頼できるプライベートネットワークまたは local loopback ホストに対してのみ
  `http://` を受け付けます。公開 SearXNG エンドポイントでは `https://` を使用する必要があります。
  Firecrawl と Tavily は、`web_search` を介した `query` と `count` のみをサポートします。
  高度なオプションには専用ツールを使用してください。
</Warning>

## x_search

`x_search` は xAI を使用して X（旧 Twitter）の投稿を検索し、
引用付きの AI 統合回答を返します。自然言語クエリと、
オプションの構造化フィルターを受け付けます。OpenClaw は組み込みの xAI `x_search`
ツールを常時登録するのではなく、リクエストごとに構築するため、
実際に呼び出したターンでのみ有効になります。

<Warning>
  `x_search` は xAI のサーバー上で実行されます。xAI の料金はツール呼び出し 1,000 回あたり $5 に加え、
  モデルの入力トークンと出力トークンの料金がかかります。
</Warning>

<Note>
  xAI のドキュメントによると、`x_search` はキーワード検索、セマンティック検索、ユーザー検索、
  スレッド取得をサポートしています。リポスト、返信、ブックマーク、閲覧数などの投稿ごとの
  エンゲージメント統計には、正確な投稿 URL またはステータス ID を対象とした検索を推奨します。
  広範なキーワード検索でも該当する投稿を見つけられる場合がありますが、投稿ごとのメタデータが
  不完全になる可能性があります。まず投稿を特定し、次にその投稿だけを対象とする
  2 回目の `x_search` クエリを実行する方法が効果的です。
</Note>

### x_search の設定

`enabled` を省略した場合、`x_search` はアクティブなモデルの
プロバイダーが `xai` であり、xAI の認証情報を解決できる場合にのみ公開されます。
既知の非 xAI プロバイダーを使用するアクティブなモデルでプロバイダーをまたいで使用するには、
`plugins.entries.xai.config.xSearch.enabled` を `true` に設定してオプトインします。
アクティブなモデルのプロバイダーが指定されていないか解決できない場合、ツールは非表示のままです。
すべてのプロバイダーで無効にするには、`enabled` を `false` に設定します。
xAI の認証情報は常に必要です。

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          xSearch: {
            enabled: true, // 既知の非 xAI モデルプロバイダーでは必須
            model: "grok-4.3",
            baseUrl: "https://api.x.ai/v1", // オプション。webSearch.baseUrl を上書き
            inlineCitations: false,
            maxTurns: 2,
            timeoutSeconds: 30,
            cacheTtlMinutes: 15,
          },
          webSearch: {
            apiKey: "xai-...", // xAI 認証プロファイルまたは XAI_API_KEY が設定されている場合は省略可能
            baseUrl: "https://api.x.ai/v1", // オプションの共有 xAI Responses ベース URL
          },
        },
      },
    },
  },
}
```

`plugins.entries.xai.config.xSearch.baseUrl` が設定されている場合、`x_search` は
`<baseUrl>/responses` に POST します。このフィールドを省略すると、
`plugins.entries.xai.config.webSearch.baseUrl`、続いて公開 xAI エンドポイント
（`https://api.x.ai/v1`）へフォールバックします。

### x_search のパラメーター

| パラメーター                    | 説明                                            |
| ---------------------------- | ------------------------------------------------------ |
| `query`                      | 検索クエリ（必須）                                |
| `allowed_x_handles`          | 結果を最大 20 個の X ハンドルに限定               |
| `excluded_x_handles`         | 最大 20 個の X ハンドルを除外                           |
| `from_date`                  | この日付以降の投稿のみを含める（YYYY-MM-DD）  |
| `to_date`                    | この日付以前の投稿のみを含める（YYYY-MM-DD） |
| `enable_image_understanding` | 一致する投稿に添付された画像を xAI が確認できるようにする      |
| `enable_video_understanding` | 一致する投稿に添付された動画を xAI が確認できるようにする      |

`allowed_x_handles` と `excluded_x_handles` は同時に指定できません。

### x_search の例

```javascript
await x_search({
  query: "dinner recipes",
  allowed_x_handles: ["nytfood"],
  from_date: "2026-03-01",
});
```

```javascript
// 投稿ごとの統計: 可能な場合は正確なステータス URL またはステータス ID を使用
await x_search({
  query: "https://x.com/huntharo/status/1905678901234567890",
});
```

## 例

```javascript
// 基本検索
await web_search({ query: "OpenClaw plugin SDK" });

// ドイツ向け検索
await web_search({ query: "TV online schauen", country: "DE", language: "de" });

// 最近の結果（過去 1 週間）
await web_search({ query: "AI developments", freshness: "week" });

// 日付範囲
await web_search({
  query: "climate research",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});

// ドメインフィルタリング（Perplexity のみ）
await web_search({
  query: "product reviews",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});
```

## ツールプロファイル

ツールプロファイルまたは許可リストを使用する場合は、`web_search`、`x_search`、または `group:web` を追加します。

```json5
{
  tools: {
    allow: ["web_search", "x_search"],
    // または: allow: ["group:web"]  （web_search、x_search、web_fetch を含む）
  },
}
```

## 関連項目

- [Web Fetch](/ja-JP/tools/web-fetch) -- URL を取得して読みやすいコンテンツを抽出
- [Web Browser](/ja-JP/tools/browser) -- JavaScript を多用するサイト向けの完全なブラウザー自動化
- [Grok Search](/ja-JP/tools/grok-search) -- `web_search` プロバイダーとしての Grok
- [Ollama Web Search](/ja-JP/tools/ollama-search) -- Ollama ホストを介したキー不要のウェブ検索
