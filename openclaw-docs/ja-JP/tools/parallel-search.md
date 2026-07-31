---
read_when:
    - API キーなしで Web 検索を利用したい場合
    - Parallel の有料 Search API を利用する場合
    - LLM のコンテキスト効率を重視して順位付けされた高密度な抜粋が必要な場合
summary: 並列検索 -- Web ソースから抽出した LLM 向けに最適化された高密度な抜粋
title: 並列検索
x-i18n:
    generated_at: "2026-07-26T10:07:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eff693f286015b287bbdacf44f11ff6f07f2f7d2605ef6f09259e7402b40515e
    source_path: tools/parallel-search.md
    workflow: 16
---

Parallel Plugin は、2つの [Parallel](https://parallel.ai/) `web_search`
プロバイダーを提供します。どちらも、AI エージェント向けに構築された Web インデックスから、
順位付けされた LLM 最適化済みの抜粋を返します。

| プロバイダー               | id              | 認証                                                                                       |
| ---------------------- | --------------- | ------------------------------------------------------------------------------------------ |
| Parallel Search（無料） | `parallel-free` | なし -- Parallel の無料 [Search MCP](https://docs.parallel.ai/integrations/mcp/search-mcp) |
| Parallel Search        | `parallel`      | `PARALLEL_API_KEY` -- 有料の Search API、より高いレート制限と目的の調整機能             |

明示的に選択するには、`tools.web.search.provider` を `parallel-free` または
`parallel` に設定します。どちらも自動検出されません。

<Note>
  OpenAI Responses の直接モデル（`api: "openai-responses"`、プロバイダー
  `openai`、公式 API ベース URL）は、`tools.web.search.provider` が未設定、空、
  `"auto"`、または `"openai"` の場合、OpenAI がホストするネイティブ Web 検索を
  自動的に使用するため、デフォルトでは Parallel を経由しません。代わりに Parallel を経由させるには、
  `tools.web.search.provider` を `parallel-free` または `parallel` に設定します。
  [Web 検索の概要](/ja-JP/tools/web)を参照してください。
</Note>

## Plugin のインストール

```bash
openclaw plugins install @openclaw/parallel-plugin
openclaw gateway restart
```

## API キー（有料プロバイダー）

`parallel-free` にはキーは不要ですが、明示的に選択する必要があります。有料の
`parallel` プロバイダーには API キーが必要です。

<Steps>
  <Step title="アカウントを作成">
    [platform.parallel.ai](https://platform.parallel.ai) で登録し、
    ダッシュボードから API キーを生成します。
  </Step>
  <Step title="キーを保存">
    Gateway 環境に `PARALLEL_API_KEY` を設定するか、次のコマンドで構成します。

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## 構成

```json5
{
  plugins: {
    entries: {
      parallel: {
        config: {
          webSearch: {
            apiKey: "par-...", // PARALLEL_API_KEY が設定されている場合は省略可能
            baseUrl: "https://api.parallel.ai", // 省略可能。OpenClaw が /v1/search を追加
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        // 無料の Search MCP には "parallel-free"、ここに示す
        // 有料 API ベースのプロバイダーには "parallel"。
        provider: "parallel",
      },
    },
  },
}
```

**環境変数による代替方法:** Gateway 環境に `PARALLEL_API_KEY` を設定します。
Gateway インストールでは、`~/.openclaw/.env` に記述します。

## ベース URL の上書き

有料の `parallel` プロバイダーにのみ適用されます。`parallel-free` は常に
`https://search.parallel.ai/mcp` を使用し、この設定を無視します。

互換性のあるプロキシまたは代替エンドポイント（たとえば Cloudflare AI Gateway）を
経由して有料リクエストを送信するには、`plugins.entries.parallel.config.webSearch.baseUrl` を設定します。
OpenClaw はベアホストの先頭に `https://` を付加して正規化し、
パスがすでに `/v1/search` で終わっていない限り、それを末尾に追加します。
解決されたエンドポイントは検索キャッシュキーの一部になるため、異なる
エンドポイントの結果が共有されることはありません。

## ツールパラメーター

どちらのプロバイダーも Parallel のネイティブ検索形式を公開するため、モデルは
自然言語による目標と、少数の短いキーワードクエリを入力します。この組み合わせは、
最良の結果を得るために Parallel が[推奨](https://docs.parallel.ai/search/best-practices)しています。

<ParamField path="objective" type="string" required>
根本となる質問または目標の自然言語による説明（最大 5000 文字）。
それだけで内容が完結している必要があります。
</ParamField>

<ParamField path="search_queries" type="string[]" required>
簡潔なキーワード検索クエリ。各 3～6 語（1～5 件、各最大 200 文字）。
最良の結果を得るには、多様なクエリを 2～3 件指定します。
</ParamField>

<ParamField path="count" type="number">
返す結果数（1～40）。
</ParamField>

<ParamField path="session_id" type="string">
以前の結果の `sessionId` から取得した、任意の Parallel セッション ID。
同じタスク内の後続検索で渡すと、Parallel が関連する呼び出しをグループ化し、
後続の結果を改善します。`parallel` では最大 1000 文字です。無料の
`parallel-free` Search MCP では最大 100 文字に制限されます。上限を超える ID は、
有料版では破棄され、無料版では新しい ID が生成されます。
</ParamField>

<ParamField path="client_model" type="string">
呼び出しを行うモデルの任意の識別子（例: `claude-opus-4-7`、
`gpt-5.6-sol`）。最大 100 文字です。Parallel がモデルの機能に合わせて
デフォルト設定を調整できるようにします。正確なアクティブモデルのスラッグを渡し、
ファミリーのエイリアスに短縮しないでください。
</ParamField>

## 注記

- Parallel は、人間によるリンクのクリックではなく LLM の推論での有用性を基準に結果を順位付けして圧縮します。
  ページ全体のコンテンツではなく、結果ごとに情報密度の高い抜粋が返されます。
- 結果の抜粋は `excerpts` 配列として返されるほか、汎用の
  `web_search` コントラクトとの互換性のために結合されて `description` にも格納されます。
- どちらのプロバイダーも `session_id` を返します。OpenClaw はツールペイロード内で
  これを `sessionId` として公開し、呼び出し元が後続検索をグループ化できるようにします。
  Parallel が生成したセッション ID（呼び出し元が指定していないもの）はキャッシュエントリから除外されます。
  これは、同一のクエリを使用する無関係なタスクがその ID を引き継がないようにするためです。
- Parallel からの `searchId`、`warnings`、および `usage` は、
  存在する場合、そのまま渡されます。
- OpenClaw は、解決された結果数を常に `advanced_settings.max_results`（`parallel`）として
  Parallel に転送するか、Parallel の固定サイズのレスポンス（`parallel-free`）に対して
  クライアント側で `count` を適用します。呼び出し元の `count` 引数が最優先され、
  次に `tools.web.search.maxResults`、それ以外の場合は OpenClaw の汎用 `web_search` のデフォルト値（5）が
  使用されます。Parallel 自体の API のデフォルト値は 10 です。
- 結果はデフォルトで 15 分間キャッシュされます（`cacheTtlMinutes`）。
- 呼び出し元が指定しない場合、`parallel-free` は MCP ハンドシェイクを介して
  呼び出しごとに新しい `session_id` を生成します。一方、`parallel` は
  その場合、未設定のままにします。

## 関連項目

- [Web 検索の概要](/ja-JP/tools/web) -- すべてのプロバイダーと自動検出
- [Exa 検索](/ja-JP/tools/exa-search) -- コンテンツ抽出を備えたニューラル検索
- [Perplexity Search](/ja-JP/tools/perplexity-search) -- ドメインフィルタリングを備えた構造化結果
