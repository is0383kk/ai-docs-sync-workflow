---
read_when:
    - web_search に Exa を使用する場合
    - EXA_API_KEY が必要です
    - ニューラル検索またはコンテンツ抽出を利用したい場合
summary: Exa AI 検索 -- コンテンツ抽出を備えたニューラル検索とキーワード検索
title: Exa 検索
x-i18n:
    generated_at: "2026-07-26T10:32:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3ddfd6fb471f92e705facf5a2d02361c1a343b9032fa8e0a7b135af634df65b7
    source_path: tools/exa-search.md
    workflow: 16
---

[Exa AI](https://exa.ai/) は、ニューラル、キーワード、ハイブリッドの検索モードに加え、組み込みのコンテンツ抽出（ハイライト、テキスト、要約）を備えた `web_search` プロバイダーです。

## Plugin をインストール

```bash
openclaw plugins install @openclaw/exa-plugin
openclaw gateway restart
```

## API キーを取得

<Steps>
  <Step title="アカウントを作成">
    [exa.ai](https://exa.ai/) でアカウントを登録し、ダッシュボードから API キーを生成します。
  </Step>
  <Step title="キーを保存">
    Gateway 環境で `EXA_API_KEY` を設定するか、次のコマンドで構成します。

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## 設定

```json5
{
  plugins: {
    entries: {
      exa: {
        config: {
          webSearch: {
            apiKey: "exa-...", // EXA_API_KEY が設定されている場合は省略可能
            baseUrl: "https://api.exa.ai", // 省略可能。OpenClaw は /search を付加
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "exa",
      },
    },
  },
}
```

**環境変数を使用する方法:** Gateway の環境に `EXA_API_KEY` を設定します。Gateway をインストールしている場合は、`~/.openclaw/.env` に記述します。詳しくは、[環境変数](/ja-JP/help/faq#env-vars-and-env-loading)を参照してください。

## ベース URL の上書き

Exa の検索リクエストを互換プロキシまたは代替エンドポイント経由でルーティングするには、`plugins.entries.exa.config.webSearch.baseUrl` を設定します。OpenClaw は、ベアホストの先頭に `https://` を付けて正規化し、次の場合を除き `/search` を末尾に追加します。
パスはすでにそこで終わっています。解決されたエンドポイントは検索キャッシュキーの一部であるため、異なるエンドポイントの結果が共有されることはありません。

## ツールパラメーター

<ParamField path="query" type="string" required>
検索クエリ。
</ParamField>

<ParamField path="count" type="number" default="5">
返す結果の数（1～100、Exa の検索タイプの制限に従います）。
</ParamField>

<ParamField path="type" type="'auto' | 'neural' | 'fast' | 'deep' | 'deep-reasoning' | 'instant'">
検索モード。
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
期間フィルター。`date_after`/`date_before` とは併用できません。
</ParamField>

<ParamField path="date_after" type="string">
この日付より後の結果（`YYYY-MM-DD`）。
</ParamField>

<ParamField path="date_before" type="string">
この日付より前の結果（`YYYY-MM-DD`）。
</ParamField>

<ParamField path="contents" type="object">
コンテンツ抽出オプション（以下を参照）。
</ParamField>

### コンテンツ抽出

結果から抽出するコンテンツを制御するには、`contents` オブジェクトを渡します。

```javascript
await web_search({
  query: "トランスフォーマーアーキテクチャの解説",
  type: "neural",
  contents: {
    text: true, // ページ全文
    highlights: { numSentences: 3 }, // 主要な文
    summary: true, // AI による要約
  },
});
```

| コンテンツオプション | 型                                                                    | 説明                   |
| -------------------- | --------------------------------------------------------------------- | ---------------------- |
| `text`          | `boolean \| { maxCharacters }`                                        | ページ全文を抽出       |
| `highlights`    | `boolean \| { maxCharacters, query, numSentences, highlightsPerUrl }` | 主要な文を抽出         |
| `summary`       | `boolean \| { query }`                                                | AI が生成した要約      |

`contents` を省略すると、Exa はデフォルトで `{ highlights: true }` を使用するため、結果には
主要な文の抜粋が含まれます。結果の説明には、ハイライト、要約、全文の順で、
最初に利用可能なものが使用されます。また、利用可能な場合は、Exa API
レスポンスの未加工の `highlightScores` フィールドと `summary` フィールドも結果に保持されます。

### 検索モード

| モード           | 説明                                   |
| ---------------- | -------------------------------------- |
| `auto`           | Exa が最適なモードを選択（デフォルト） |
| `neural`         | セマンティック／意味ベースの検索       |
| `fast`           | 高速なキーワード検索                   |
| `deep`           | 網羅的な詳細検索                       |
| `deep-reasoning` | 推論を伴う詳細検索                     |
| `instant`        | 最速の結果                             |

## 注意事項

- `count` には、Exa の検索タイプの上限に従い、最大 100 まで指定できます。
- 結果はデフォルトで 15 分間キャッシュされます。すべての `web_search` プロバイダー（Exa を含む）の
  キャッシュとリクエストタイムアウトを変更するには、共有の
  `tools.web.search.cacheTtlMinutes`（分）と
  `tools.web.search.timeoutSeconds`（デフォルト 30 秒）を設定します。

## 関連項目

- [Web 検索の概要](/ja-JP/tools/web) -- すべてのプロバイダーと自動検出
- [Brave Search](/ja-JP/tools/brave-search) -- 国／言語フィルターに対応した構造化結果
- [Perplexity Search](/ja-JP/tools/perplexity-search) -- ドメインフィルタリングに対応した構造化結果
