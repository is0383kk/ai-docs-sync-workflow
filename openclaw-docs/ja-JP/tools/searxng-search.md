---
read_when:
    - セルフホスト型のウェブ検索プロバイダーを使用したい場合
    - web_search に SearXNG を使用する場合
    - プライバシー重視またはエアギャップ環境向けの検索オプションが必要な場合
summary: SearXNG ウェブ検索 -- セルフホスト型でキー不要のメタ検索プロバイダー
title: SearXNG 検索
x-i18n:
    generated_at: "2026-07-26T10:07:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cae8de9f8e2c8dd9cec615adb48da5c1fd7654bffe96c7afc1acea3effbcf1fc
    source_path: tools/searxng-search.md
    workflow: 16
---

OpenClaw は [SearXNG](https://docs.searxng.org/) を**セルフホスト型で、
キー不要**の `web_search` プロバイダーとしてサポートしています。SearXNG は、Google、Bing、DuckDuckGo、およびその他のソースから結果を集約するオープンソースのメタ検索エンジンです。

利点:

- **無料で無制限** -- API キーや商用サブスクリプションは不要
- **プライバシー / エアギャップ** -- クエリがネットワークの外部に送信されることはありません
- **どこでも利用可能** -- 商用検索 API の地域制限がありません

## セットアップ

<Steps>
  <Step title="Plugin をインストール">
    ```bash
    openclaw plugins install @openclaw/searxng-plugin
    ```
  </Step>
  <Step title="SearXNG インスタンスを実行">
    ```bash
    docker run -d -p 8888:8080 searxng/searxng
    ```

    または、アクセス可能な既存の SearXNG デプロイメントを使用します。本番環境のセットアップについては、
    [SearXNG ドキュメント](https://docs.searxng.org/)を参照してください。

  </Step>
  <Step title="設定">
    ```bash
    openclaw configure --section web
    # プロバイダーとして "searxng" を選択
    ```

    または、環境変数を設定して自動検出に任せます:

    ```bash
    export SEARXNG_BASE_URL="http://localhost:8888"
    ```

  </Step>
</Steps>

## 設定

```json5
{
  tools: {
    web: {
      search: {
        provider: "searxng",
      },
    },
  },
}
```

SearXNG インスタンスの Plugin レベル設定:

```json5
{
  plugins: {
    entries: {
      searxng: {
        config: {
          webSearch: {
            baseUrl: "http://localhost:8888",
            categories: "general,news", // 任意
            language: "en", // 任意
          },
        },
      },
    },
  },
}
```

`baseUrl` は SecretRef オブジェクトも受け付けます（例: `{ source: "env", id: "SEARXNG_BASE_URL" }`）。

## 環境変数

設定の代わりに `SEARXNG_BASE_URL` を設定します:

```bash
export SEARXNG_BASE_URL="http://localhost:8888"
```

解決順序は、設定された `baseUrl` 文字列、次に
`baseUrl` のインライン環境変数 SecretRef、最後に `SEARXNG_BASE_URL` です。いずれの設定パスも設定されておらず、
プロバイダーが明示的に選択されていない状態で `SEARXNG_BASE_URL` が存在する場合、自動検出により
SearXNG が選択されます。

## Plugin 設定リファレンス

| フィールド        | 説明                                                        |
| ------------ | ------------------------------------------------------------------ |
| `baseUrl`    | SearXNG インスタンスのベース URL（必須）                       |
| `categories` | `general`、`news`、`science` などのカンマ区切りのカテゴリー |
| `language`   | `en`、`de`、`fr` などの結果用言語コード              |

`web_search` ツール呼び出しでは、呼び出しごとのオーバーライドとして `count`（1～10 件の結果）、`categories`、
および `language` も受け付けます。

## 注意事項

- **JSON API** -- HTML スクレイピングではなく、SearXNG ネイティブの `format=json` エンドポイントを使用
- **画像結果 URL** -- SearXNG が画像の直接 URL を返す場合、画像カテゴリーの結果に `img_src` が含まれます
- **API キー不要** -- どの SearXNG インスタンスでもすぐに動作
- **ベース URL の検証** -- `baseUrl` は有効な `http://` または `https://`
  URL である必要があります
- **ネットワークガード** -- `http://` ベース URL は、信頼できるプライベートホストまたは
  local loopback ホストを対象とする必要があります（パブリックホストでは `https://` を使用する必要があります）。プライベートまたは内部アドレスに
  解決される `https://` ベース URL には同じセルフホスト許可が適用されますが、
  パブリックに解決される `https://` ベース URL には厳格な SSRF 保護が維持されます
- **自動検出順序** -- SearXNG には設定済みの `baseUrl` が必要です（必要な認証情報がすでに存在するプロバイダーの中での順序は
  200）。DuckDuckGo や Ollama Web Search などのキー不要プロバイダーが暗黙的に自動検出で選ばれることはありません。
  明示的に `provider` を選択した場合にのみ有効になります
- **セルフホスト型** -- インスタンス、クエリ、上流の検索エンジンを自身で管理できます
- **カテゴリー** -- 設定されていない場合のデフォルトは `general`
- **カテゴリーのフォールバック** -- `general` 以外のカテゴリーへのリクエストが成功しても
  結果が 0 件だった場合、OpenClaw は空の結果セットを返す前に、同じクエリを `general` で一度再試行します
- **結果のキャッシュ** -- 同一のクエリ（クエリ、件数、カテゴリー、
  言語、ベース URL が同じもの）は、短い TTL の間プロセス内にキャッシュされます
- **バージョン要件** -- Plugin は `minHostVersion: >=2026.6.9` を宣言します

<Tip>
  SearXNG JSON API を動作させるには、SearXNG インスタンスの `settings.yml` 内の `search.formats` で
  `json` 形式が有効になっていることを確認してください。
</Tip>

## 関連項目

- [Web 検索の概要](/ja-JP/tools/web) -- すべてのプロバイダーと自動検出
- [DuckDuckGo Search](/ja-JP/tools/duckduckgo-search) -- 別のキー不要プロバイダー
- [Brave Search](/ja-JP/tools/brave-search) -- 無料枠付きの構造化された結果
