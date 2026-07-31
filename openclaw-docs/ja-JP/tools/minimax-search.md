---
read_when:
    - web_search に MiniMax を使用する場合
    - MiniMax Token Plan キーまたは OAuth トークンが必要です
    - MiniMax の中国向け／グローバル向け検索ホストに関するガイダンスが必要な場合
summary: Token Plan 検索 API を使用した MiniMax Search
title: MiniMax 検索
x-i18n:
    generated_at: "2026-07-26T09:22:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb851614bbe43f011e07fe3e80d5390f1ba515f3e00ba749c91999617ad2d1e2
    source_path: tools/minimax-search.md
    workflow: 16
---

OpenClaw は、MiniMax Token Plan 検索 API を介して MiniMax を `web_search` プロバイダーとしてサポートしています。この API は、タイトル、URL、スニペット、関連クエリを含む構造化された検索結果を返します。

## Token Plan 認証情報を取得する

<Steps>
  <Step title="キーを作成する">
    [MiniMax Platform](https://platform.minimax.io/user-center/basic-information/interface-key) で MiniMax Token Plan キーを作成またはコピーします。
    OAuth セットアップでは、代わりに `MINIMAX_OAUTH_TOKEN` を再利用できます。
  </Step>
  <Step title="キーを保存する">
    Gateway 環境に `MINIMAX_CODE_PLAN_KEY` を設定するか、次のコマンドで構成します。

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

OpenClaw は、環境変数のエイリアスとして `MINIMAX_CODING_API_KEY`、`MINIMAX_OAUTH_TOKEN`、`MINIMAX_API_KEY` も受け付けます。これらは `MINIMAX_CODE_PLAN_KEY` の後に、この順序で確認されます。`MINIMAX_API_KEY` には、検索が有効な Token Plan 認証情報を指定する必要があります。通常の MiniMax モデル API キーは、Token Plan 検索エンドポイントで受け付けられない場合があります。

## 構成

```json5
{
  plugins: {
    entries: {
      minimax: {
        config: {
          webSearch: {
            apiKey: "sk-cp-...", // MiniMax Token Plan の環境変数が設定されている場合は省略可能
            region: "global", // または "cn"
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "minimax",
      },
    },
  },
}
```

**環境変数を使用する方法:** Gateway 環境に `MINIMAX_CODE_PLAN_KEY`、`MINIMAX_CODING_API_KEY`、`MINIMAX_OAUTH_TOKEN`、または `MINIMAX_API_KEY` を設定します。
Gateway のインストール環境では、`~/.openclaw/.env` に記述します。

## リージョンの選択

MiniMax Search は次のエンドポイントを使用します。

- グローバル: `https://api.minimax.io/v1/coding_plan/search`
- 中国: `https://api.minimaxi.com/v1/coding_plan/search`

`plugins.entries.minimax.config.webSearch.region` が未設定の場合、OpenClaw は次の順序でリージョンを決定します。

1. Plugin が所有する `webSearch.region`
2. `MINIMAX_API_HOST`
3. `models.providers.minimax.baseUrl`
4. `models.providers.minimax-portal.baseUrl`

つまり、中国向けのオンボーディングまたは `MINIMAX_API_HOST=https://api.minimaxi.com/...` を使用すると、MiniMax Search も自動的に中国ホストを使用します。

OAuth の `minimax-portal` パスを介して MiniMax を認証した場合でも、Web 検索はプロバイダー ID `minimax` として登録されます。OAuth プロバイダーのベース URL は、中国／グローバルホストを選択する際のリージョンのヒントとして使用され、`MINIMAX_OAUTH_TOKEN` を MiniMax Search の Bearer 認証情報として使用できます。

## サポートされるパラメーター

| パラメーター | 型      | 制約            | 説明                                                                     |
| --------- | ------- | --------------- | --------------------------------------------------------------------------- |
| `query`   | 文字列  | 必須             | 検索クエリ文字列。                                                        |
| `count`   | 整数 | 1-10、デフォルト 5 | 返す結果の数。OpenClaw は返されたリストをこの件数に切り詰めます。 |

現在、プロバイダー固有のフィルターはサポートされていません。

## 関連項目

- [Web 検索の概要](/ja-JP/tools/web) -- すべてのプロバイダーと自動検出
- [MiniMax](/ja-JP/providers/minimax) -- モデル、画像、音声、認証のセットアップ
