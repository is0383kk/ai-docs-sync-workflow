---
read_when:
    - Web 検索に Perplexity Search を使用する場合
    - PERPLEXITY_API_KEY または OPENROUTER_API_KEY の設定が必要です
summary: web_search向けのPerplexity Search APIおよびSonar/OpenRouter互換性
title: Perplexity 検索
x-i18n:
    generated_at: "2026-07-26T09:22:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a7ca97355110e70a05f1d57acab475dda8dec89393804df40c6e9be5e30780e8
    source_path: tools/perplexity-search.md
    workflow: 16
---

OpenClaw は Perplexity Search API を `web_search` プロバイダーとしてサポートしています。この API は、`title`、`url`、`snippet` フィールドを含む構造化された結果を返します。

互換性のため、OpenClaw は従来の Perplexity Sonar/OpenRouter 設定もサポートしています。`OPENROUTER_API_KEY`、`plugins.entries.perplexity.config.webSearch.apiKey` 内の `sk-or-...` キーを使用するか、`plugins.entries.perplexity.config.webSearch.baseUrl` / `model` を設定すると、プロバイダーはチャット補完パスに切り替わり、構造化された Search API の結果ではなく、引用付きの AI 合成回答を返します。

## Plugin のインストール

公式 Plugin をインストールしてから、Gateway を再起動します。

```bash
openclaw plugins install @openclaw/perplexity-plugin
openclaw gateway restart
```

## Perplexity API キーの取得

1. [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api) で Perplexity アカウントを作成します。
2. ダッシュボードで API キーを生成します。
3. キーを設定に保存するか、Gateway 環境で `PERPLEXITY_API_KEY` を設定します。

## OpenRouter との互換性

Perplexity Sonar に OpenRouter をすでに使用していた場合は、`provider: "perplexity"` を維持して Gateway 環境で `OPENROUTER_API_KEY` を設定するか、`plugins.entries.perplexity.config.webSearch.apiKey` に `sk-or-...` キーを保存します。

オプションの互換性制御：

- `plugins.entries.perplexity.config.webSearch.baseUrl`
- `plugins.entries.perplexity.config.webSearch.model`

## 設定例

### ネイティブ Perplexity Search API

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "pplx-...",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

### OpenRouter / Sonar 互換性

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "<openrouter-api-key>",
            baseUrl: "https://openrouter.ai/api/v1",
            model: "perplexity/sonar-pro",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

## キーを設定する場所

**設定を使用する場合：** `openclaw configure --section web` を実行します。キーは `plugins.entries.perplexity.config.webSearch.apiKey` 配下の `~/.openclaw/openclaw.json` に保存されます。このフィールドは SecretRef オブジェクトも受け入れます。

**環境変数を使用する場合：** Gateway プロセス環境で `PERPLEXITY_API_KEY` または `OPENROUTER_API_KEY` を設定します。Gateway をインストールしている場合は、`~/.openclaw/.env`（またはサービス環境）に配置します。[環境変数](/ja-JP/help/faq#env-vars-and-env-loading)を参照してください。

`provider: "perplexity"` が設定されており、Perplexity キーの SecretRef を解決できず、環境変数によるフォールバックもない場合、起動または再読み込みは即座に失敗します。

## ツールパラメーター

以下のパラメーターは、ネイティブ Perplexity Search API パスに適用されます。

<ParamField path="query" type="string" required>
検索クエリ。
</ParamField>

<ParamField path="count" type="number" default="5">
返す結果の数（1～10）。
</ParamField>

<ParamField path="country" type="string">
2 文字の ISO 国コード（例：`US`、`DE`）。
</ParamField>

<ParamField path="language" type="string">
ISO 639-1 言語コード（例：`en`、`de`、`fr`）。
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
期間フィルター。`day` は 24 時間です。
</ParamField>

<ParamField path="date_after" type="string">
この日付（`YYYY-MM-DD`）より後に公開された結果のみ。
</ParamField>

<ParamField path="date_before" type="string">
この日付（`YYYY-MM-DD`）より前に公開された結果のみ。
</ParamField>

<ParamField path="domain_filter" type="string[]">
ドメインの許可リストまたは拒否リストの配列（最大 20）。
</ParamField>

<ParamField path="max_tokens" type="number" default="25000">
コンテンツの合計割り当て量（最大 1000000）。
</ParamField>

<ParamField path="max_tokens_per_page" type="number" default="2048">
ページごとのトークン上限。
</ParamField>

従来の Sonar/OpenRouter 互換パスの場合：

- `query`、`count`、`freshness` が受け入れられます。
- そのパスでは `count` は互換性のためだけに使用されます。応答は N 件の結果リストではなく、引き続き引用付きの単一の合成回答です。
- Search API 専用フィルター（`country`、`language`、`date_after`、`date_before`、`domain_filter`、`max_tokens`、`max_tokens_per_page`）を使用すると、明示的なエラーが返されます。

**例：**

```javascript
// 国と言語を指定した検索
await web_search({
  query: "再生可能エネルギー",
  country: "DE",
  language: "de",
});

// 最近の結果（過去 1 週間）
await web_search({
  query: "AI ニュース",
  freshness: "week",
});

// 日付範囲検索
await web_search({
  query: "AI の進展",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});

// ドメインフィルタリング（許可リスト）
await web_search({
  query: "気候研究",
  domain_filter: ["nature.com", "science.org", ".edu"],
});

// ドメインフィルタリング（拒否リスト：先頭に - を付ける）
await web_search({
  query: "製品レビュー",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});

// より多くのコンテンツを抽出
await web_search({
  query: "詳細な AI 研究",
  max_tokens: 50000,
  max_tokens_per_page: 4096,
});
```

### ドメインフィルターのルール

- フィルターごとに最大 20 ドメイン。
- 同じリクエスト内で許可リストと拒否リストのエントリを混在させることはできません。
- 拒否リストのエントリには `-` プレフィックスを使用します（例：`["-reddit.com"]`）。

## 注記

- Perplexity Search API は、構造化された Web 検索結果（`title`、`url`、`snippet`）を返します。
- OpenRouter、または明示的な `plugins.entries.perplexity.config.webSearch.baseUrl` / `model` を使用すると、互換性のため Perplexity は Sonar チャット補完に戻ります。
- Sonar/OpenRouter 互換モードでは、構造化された結果行ではなく、引用付きの単一の合成回答が返されます。
- 結果はデフォルトで 15 分間キャッシュされます（`cacheTtlMinutes` で設定可能）。

## 関連項目

<CardGroup cols={2}>
  <Card title="Web 検索の概要" href="/ja-JP/tools/web" icon="globe">
    すべてのプロバイダーと自動検出ルール。
  </Card>
  <Card title="Brave 検索" href="/ja-JP/tools/brave-search" icon="shield">
    国と言語のフィルターを使用した構造化された結果。
  </Card>
  <Card title="Exa 検索" href="/ja-JP/tools/exa-search" icon="magnifying-glass">
    コンテンツ抽出を備えたニューラル検索。
  </Card>
  <Card title="Perplexity Search API ドキュメント" href="https://docs.perplexity.ai/docs/search/quickstart" icon="arrow-up-right-from-square">
    Perplexity Search API の公式クイックスタートとリファレンス。
  </Card>
</CardGroup>
