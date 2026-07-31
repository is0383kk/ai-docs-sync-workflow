---
read_when:
    - web_search に Kimi を使用する場合
    - KIMI_API_KEY または MOONSHOT_API_KEY が必要です
summary: Moonshot web search を介した Kimi web search
title: Kimi 検索
x-i18n:
    generated_at: "2026-07-26T09:47:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 65e5f8c9f3b607dbcc3256c51a6a083864e31f65ed2a751d2d500abeb35ba844
    source_path: tools/kimi-search.md
    workflow: 16
---

Kimi は、Moonshot のネイティブ Web 検索を基盤とする `web_search` プロバイダーです。Moonshot は、順位付けされた結果リストを返すのではなく、Gemini や Grok のグラウンディングされた応答プロバイダーと同様に、インライン引用を含む 1 つの回答を合成します。

## セットアップ

<Steps>
  <Step title="キーを作成">
    [Moonshot AI](https://platform.moonshot.cn/) から API キーを取得します。
  </Step>
  <Step title="キーを保存">
    Gateway 環境に `KIMI_API_KEY` または `MOONSHOT_API_KEY` を設定するか（Gateway インストールの場合は `~/.openclaw/.env` に追加）、次のコマンドで設定します。

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

`openclaw onboard` または `openclaw configure --section web` で **Kimi** を選択すると、次の項目の入力も求められます。

- Moonshot API リージョン: `https://api.moonshot.ai/v1` または `https://api.moonshot.cn/v1`
- Web 検索モデル（デフォルトは `kimi-k2.6`）

## 設定

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // KIMI_API_KEY または MOONSHOT_API_KEY が設定されている場合は省略可能
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.6",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

`tools.web.search.provider` を省略すると、利用可能な API キーから自動検出されます。複数の検索認証情報が設定されている場合は、明示的に `kimi` に設定してください。

Kimi 固有の `apiKey`、`baseUrl`、`model` の値は、`plugins.entries.moonshot.config.webSearch` で設定します。

デフォルト: `baseUrl` を省略した場合のデフォルトは `https://api.moonshot.ai/v1`、`model` のデフォルトは `kimi-k2.6` です。

チャットトラフィックが中国ホスト（`models.providers.moonshot.baseUrl`: `https://api.moonshot.cn/v1`）を使用している場合、Kimi の `baseUrl` が未設定であれば、Kimi の `web_search` はそのホストを自動的に再利用します。これにより、`.cn` キーが誤って国際エンドポイントに送信されることを防ぎます（これらのキーでは HTTP 401 が返されます）。この継承を上書きするには、Kimi の `baseUrl` を明示的に設定します。

## グラウンディング要件

OpenClaw は、Moonshot の応答に `$web_search` ツール呼び出しのリプレイ、`search_results`、引用 URL など、ネイティブ Web 検索のグラウンディング証拠が含まれている場合にのみ、Kimi の `web_search` 結果を返します。Kimi がグラウンディングなしで直接回答した場合（例: 「インターネットを閲覧できません」）、OpenClaw はそのテキストを検索結果として扱わず、代わりに `kimi_web_search_ungrounded` エラーを返します。クエリを再試行するか、Brave などの構造化プロバイダーに切り替えるか、対象 URL がすでに分かっている場合は `web_fetch` / ブラウザツールを使用してください。

## ツールパラメーター

| パラメーター                                                       | 対応                                                                                                                |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `query`                                                         | はい                                                                                                                      |
| `count`                                                         | プロバイダー間の互換性のため受け入れますが、無視されます。Kimi は N 件の結果リストではなく、常に 1 つの合成された回答を返します |
| `country`, `language`, `freshness`, `date_after`, `date_before` | いいえ                                                                                                                       |

## 関連項目

- [Web 検索の概要](/ja-JP/tools/web) - すべてのプロバイダーと自動検出
- [Moonshot AI](/ja-JP/providers/moonshot) - Moonshot モデルと Kimi Coding プロバイダーのドキュメント
- [Gemini Search](/ja-JP/tools/gemini-search) - Google のグラウンディングによる AI 合成回答
- [Grok Search](/ja-JP/tools/grok-search) - xAI のグラウンディングによる AI 合成回答
