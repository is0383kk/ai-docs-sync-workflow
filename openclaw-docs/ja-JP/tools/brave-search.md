---
read_when:
    - web_search に Brave Search を使用する場合
    - BRAVE_API_KEY またはプランの詳細が必要です
summary: web_search 用 Brave Search API のセットアップ
title: Brave 検索
x-i18n:
    generated_at: "2026-07-26T10:06:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52168db93abb564eda5868584261e0530ce3cff57c3463a2fc1eded351df30f2
    source_path: tools/brave-search.md
    workflow: 16
---

OpenClaw は `web_search` プロバイダーとして Brave Search API をサポートしています。

## API キーを取得する

1. [https://brave.com/search/api/](https://brave.com/search/api/) で Brave Search API アカウントを作成します。
2. ダッシュボードで **Search** プランを選択し、API キーを生成します。
3. キーを設定に保存するか、Gateway 環境で `BRAVE_API_KEY` を設定します。

## 設定例

```json5
{
  plugins: {
    entries: {
      brave: {
        config: {
          webSearch: {
            apiKey: "BRAVE_API_KEY_HERE",
            mode: "web", // または "llm-context"
            baseUrl: "https://api.search.brave.com", // オプションのプロキシ／ベース URL の上書き
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "brave",
        maxResults: 5,
        timeoutSeconds: 30,
      },
    },
  },
}
```

Brave 検索固有の設定は `plugins.entries.brave.config.webSearch.*` に配置します。これが正規の設定パスです。

`webSearch.mode` は Brave の通信方式を制御します。

- `web`（デフォルト）：タイトル、URL、スニペットを含む通常の Brave ウェブ検索
- `llm-context`：グラウンディング用に事前抽出されたテキストチャンクとソースを提供する Brave LLM Context API

`webSearch.baseUrl` を使用すると、Brave リクエストを信頼できる Brave 互換プロキシ
またはゲートウェイに送信できます。OpenClaw は設定されたベース URL に
`/res/v1/web/search` または `/res/v1/llm/context` を追加し、そのベース URL をキャッシュキーに含めます。公開
エンドポイントでは `https://` を使用する必要があります。`http://` は、信頼できるループバック
またはプライベートネットワークのプロキシホストでのみ使用できます。

## ツールパラメーター

<ParamField path="query" type="string" required>
検索クエリ。
</ParamField>

<ParamField path="count" type="number" default="5">
返す結果の件数（1～10）。
</ParamField>

<ParamField path="country" type="string">
2 文字の ISO 国コード（例：`US`、`DE`）。
</ParamField>

<ParamField path="language" type="string">
検索結果の ISO 639-1 言語コード（例：`en`、`de`、`fr`）。
</ParamField>

<ParamField path="search_lang" type="string">
Brave の検索言語コード（例：`en`、`en-gb`、`zh-hans`）。
</ParamField>

<ParamField path="ui_lang" type="string">
UI 要素の ISO 言語コード。
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
期間フィルター — `day` は 24 時間です。
</ParamField>

<ParamField path="date_after" type="string">
この日付より後に公開された結果のみ（`YYYY-MM-DD`）。
</ParamField>

<ParamField path="date_before" type="string">
この日付より前に公開された結果のみ（`YYYY-MM-DD`）。
</ParamField>

**例：**

```javascript
// 国と言語を指定した検索
await web_search({
  query: "renewable energy",
  country: "DE",
  language: "de",
});

// 最近の結果（過去 1 週間）
await web_search({
  query: "AI news",
  freshness: "week",
});

// 日付範囲検索
await web_search({
  query: "AI developments",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});
```

## 注意事項

- OpenClaw は Brave の **Search** プランを使用します。従来のサブスクリプション（例：月 2,000 クエリの旧 Free プラン）を利用している場合、そのサブスクリプションは引き続き有効ですが、LLM Context やより高いレート制限などの新しい機能は含まれません。
- Brave の各プランには、毎月更新される **月額 \$5 の無料クレジット**が含まれます。Search プランの料金は 1,000 リクエストあたり \$5 なので、このクレジットで月 1,000 クエリを利用できます。予期しない請求を避けるため、Brave ダッシュボードで使用量の上限を設定してください。現在のプランについては、[Brave API ポータル](https://brave.com/search/api/)を参照してください。
- Search プランには、LLM Context エンドポイントと AI 推論権が含まれます。モデルのトレーニングやチューニングを目的として結果を保存するには、明示的な保存権を含むプランが必要です。Brave の[利用規約](https://api-dashboard.search.brave.com/terms-of-service)を参照してください。
- `llm-context` モードでは、通常のウェブ検索スニペット形式ではなく、グラウンディングされたソースエントリが返されます。
- `llm-context` モードは、`freshness` と、範囲が限定された `date_after` + `date_before` をサポートします。`ui_lang` はサポートしません。Brave ではカスタム期間範囲に開始日と終了日の両方を含める必要があるため、`date_after` を伴わない `date_before` は拒否されます。
- `ui_lang` には、`en-US` のような地域サブタグを含める必要があります。
- 結果はデフォルトで 15 分間キャッシュされます（`cacheTtlMinutes` で設定可能）。
- カスタムの `webSearch.baseUrl` 値は Brave のキャッシュ識別情報に含まれるため、
  プロキシ固有のレスポンスが競合することはありません。
- トラブルシューティング中に Brave のリクエスト URL／クエリパラメーター、レスポンスのステータス／所要時間、および検索キャッシュのヒット／ミス／書き込みイベントをログに記録するには、`brave.http` 診断フラグを有効にします。このフラグが API キーやレスポンス本文をログに記録することはありませんが、検索クエリには機密情報が含まれる可能性があります。

## 関連情報

- [ウェブ検索の概要](/ja-JP/tools/web) -- すべてのプロバイダーと自動検出
- [Perplexity Search](/ja-JP/tools/perplexity-search) -- ドメインフィルタリングを備えた構造化結果
- [Exa Search](/ja-JP/tools/exa-search) -- コンテンツ抽出を備えたニューラル検索
