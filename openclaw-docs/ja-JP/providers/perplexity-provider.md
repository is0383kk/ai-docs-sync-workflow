---
read_when:
    - Perplexity をウェブ検索プロバイダーとして設定する場合
    - Perplexity API キーまたは OpenRouter プロキシのセットアップが必要です
summary: Perplexity Web 検索プロバイダーのセットアップ（API キー、検索モード、フィルタリング）
title: Perplexity
x-i18n:
    generated_at: "2026-07-26T09:16:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea76a5cb7befce95756e9bcc8f9c1637fac87711d02d8a486ec2a1b9f51b73dc
    source_path: providers/perplexity-provider.md
    workflow: 16
---

Perplexity Plugin は、2 つのトランスポートを持つ `web_search` プロバイダーを登録します。ネイティブの Perplexity Search API（フィルター付きの構造化された結果）と、直接または OpenRouter 経由で利用する Perplexity Sonar チャット補完（引用付きの AI 合成回答）です。

<Note>
このページでは、Perplexity **プロバイダー**の設定について説明します。Perplexity **ツール**（エージェントがどのように使用するか）については、[Perplexity 検索](/ja-JP/tools/perplexity-search)を参照してください。
</Note>

| プロパティ  | 値                                                                     |
| ----------- | ---------------------------------------------------------------------- |
| 種類        | Web 検索プロバイダー（モデルプロバイダーではありません）              |
| 認証        | `PERPLEXITY_API_KEY`（ネイティブ）または `OPENROUTER_API_KEY`（OpenRouter 経由） |
| 設定パス    | `plugins.entries.perplexity.config.webSearch.apiKey`                                                     |
| 上書き      | `plugins.entries.perplexity.config.webSearch.baseUrl` / `.model`                               |
| キーの取得  | [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)   |

## Plugin のインストール

```bash
openclaw plugins install @openclaw/perplexity-plugin
openclaw gateway restart
```

## はじめに

<Steps>
  <Step title="API キーを設定する">
    ```bash
    openclaw configure --section web
    ```

    または、キーを直接設定します。

    ```bash
    openclaw config set plugins.entries.perplexity.config.webSearch.apiKey "pplx-xxxxxxxxxxxx"
    ```

    Gateway 環境で `PERPLEXITY_API_KEY` または `OPENROUTER_API_KEY` としてエクスポートされたキーも使用できます。

  </Step>
  <Step title="検索を開始する">
    `web_search` は、そのキーが利用可能な検索認証情報になると Perplexity を自動検出します。追加の設定は必要ありません。プロバイダーを明示的に固定するには、次を実行します。

    ```bash
    openclaw config set tools.web.search.provider perplexity
    ```

  </Step>
</Steps>

## 検索モード

Plugin は、次の順序でトランスポートを解決します。

1. `webSearch.baseUrl` または `webSearch.model` が設定されている場合：キーの種類に関係なく、常にそのエンドポイントに対する Sonar チャット補完を経由します。
2. それ以外の場合は、キーの取得元によってエンドポイントが決まります。設定されたキーでは、そのプレフィックスによってトランスポートが選択されます（設定は環境変数より優先されます）。環境変数のキーでは、対応するエンドポイントが直接使用されます。

| キーのプレフィックス | トランスポート                                           | 機能                                           |
| -------------------- | -------------------------------------------------------- | ---------------------------------------------- |
| `pplx-`   | ネイティブ Perplexity Search API（`https://api.perplexity.ai`）   | 構造化された結果、ドメイン／言語／日付フィルター |
| `sk-or-`   | OpenRouter（`https://openrouter.ai/api/v1`）、Sonar モデル           | 引用付きの AI 合成回答                         |

それ以外のプレフィックスを持つ設定済みキーでも、ネイティブ Search API が使用されます。チャット補完パスでは、デフォルトで `perplexity/sonar-pro` モデルが使用されます。`plugins.entries.perplexity.config.webSearch.model` で上書きできます。

## ネイティブ API のフィルタリング

| フィルター                           | 説明                                                            | トランスポート       |
| ------------------------------------ | --------------------------------------------------------------- | -------------------- |
| `count`                   | 1 回の検索あたりの結果数、1～10（デフォルトは 5）               | ネイティブのみ       |
| `freshness`                   | 期間範囲：`day`、`week`、`month`、`year` | 両方                 |
| `country`                   | 2 文字の国コード（`us`、`de`、`jp`） | ネイティブのみ       |
| `language`                   | ISO 639-1 言語コード（`en`、`fr`、`zh`） | ネイティブのみ       |
| `date_after` / `date_before` | `YYYY-MM-DD` 形式の公開日範囲                            | ネイティブのみ       |
| `domain_filter`                   | 最大 20 ドメイン。許可リストまたは `-` プレフィックス付き拒否リスト。混在は不可 | ネイティブのみ       |
| `max_tokens` / `max_tokens_per_page` | 全結果／ページごとのコンテンツ量上限                          | ネイティブのみ       |

ネイティブ専用フィルターをチャット補完パスで使用すると、説明付きのエラーが返されます。
`freshness` は `date_after`／`date_before` と組み合わせることはできません。

## 高度な設定

<AccordionGroup>
  <Accordion title="デーモンプロセス用の環境変数">
    <Warning>
    対話型シェルでのみエクスポートされたキーは、その環境を明示的にインポートしない限り、launchd/systemd の Gateway デーモンからは参照できません。Gateway プロセスが読み取れるように、キーを `~/.openclaw/.env` に設定するか、`env.shellEnv` を使用して設定してください。完全な優先順位については、[環境変数](/ja-JP/help/environment)を参照してください。
    </Warning>
  </Accordion>

  <Accordion title="OpenRouter プロキシの設定">
    Perplexity 検索を OpenRouter 経由でルーティングするには、ネイティブ Perplexity キーの代わりに `OPENROUTER_API_KEY`（プレフィックスは `sk-or-`）を設定します。OpenClaw はキーを検出し、自動的に Sonar トランスポートへ切り替えます。すでに OpenRouter の請求設定があり、プロバイダーをそこに集約したい場合に便利です。
  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="Perplexity 検索ツール" href="/ja-JP/tools/perplexity-search" icon="magnifying-glass">
    エージェントが Perplexity 検索を呼び出し、結果を解釈する方法です。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/configuration-reference" icon="gear">
    Plugin エントリを含む完全な設定リファレンスです。
  </Card>
</CardGroup>
