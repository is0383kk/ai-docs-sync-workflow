---
read_when:
    - web_search に Ollama を使用する場合
    - API キー不要の web_search プロバイダーを使用したい場合
    - OLLAMA_API_KEY を使用して、ホスト型 Ollama Web Search を利用する場合
    - Ollama Web Search のセットアップガイドが必要です
summary: ローカルの Ollama ホストまたはホスト型 Ollama API を介した Ollama Web 検索
title: Ollama ウェブ検索
x-i18n:
    generated_at: "2026-07-26T09:55:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: edbbd887841339ab4c0c62ab7682a22fe99434a788957a91989fce6942187e9a
    source_path: tools/ollama-search.md
    workflow: 16
---

OpenClaw は、バンドルされた `web_search` プロバイダーとして **Ollama Web Search** をサポートしており、
Ollama の Web 検索 API からタイトル、URL、スニペットを返します。

ローカルまたはセルフホストの Ollama では、デフォルトで API キーは不要です。到達可能な
Ollama ホストと `ollama signin` が必要です。直接ホスト型検索（ローカル Ollama なし）には、
`baseUrl: "https://ollama.com"` と実際の `OLLAMA_API_KEY` が必要です。

## セットアップ

<Steps>
  <Step title="Ollama を起動">
    Ollama がインストールされ、実行中であることを確認します。
  </Step>
  <Step title="サインイン">
    ```bash
    ollama signin
    ```
  </Step>
  <Step title="Ollama Web Search を選択">
    ```bash
    openclaw configure --section web
    ```

    プロバイダーとして **Ollama Web Search** を選択します。

  </Step>
</Steps>

モデルにすでに Ollama を使用している場合、Ollama Web Search は同じ設定済みホストを
再利用します。

<Note>
  OpenClaw が、優先度の高い認証情報付きプロバイダーよりも Ollama Web Search を
  自動的に優先して選択することはありません。`tools.web.search.provider: "ollama"` を使用して
  明示的に選択する必要があります。
</Note>

## 設定

```json5
{
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

Web 検索のみに適用される、任意のホスト上書き設定：

```json5
{
  plugins: {
    entries: {
      ollama: {
        config: {
          webSearch: {
            baseUrl: "http://ollama-host:11434",
          },
        },
      },
    },
  },
}
```

または、Ollama モデルプロバイダーにすでに設定されているホストを再利用します：

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "http://ollama-host:11434",
      },
    },
  },
}
```

`models.providers.ollama.baseUrl` が正規のキーです。Web 検索プロバイダーでは、OpenAI SDK 形式の
設定例との互換性のため、そこに `baseURL` を指定することもできます。
何も設定されていない場合、OpenClaw はデフォルトで
`http://127.0.0.1:11434` を使用します。

直接ホスト型の Ollama Web Search（ローカル Ollama なし）：

```json5
{
  models: {
    providers: {
      ollama: {
        baseUrl: "https://ollama.com",
        apiKey: "OLLAMA_API_KEY",
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "ollama",
      },
    },
  },
}
```

## 認証とリクエストルーティング

- Web 検索専用の API キーフィールドはありません。設定済みホストが認証で保護されている場合、プロバイダーは
  `models.providers.ollama.apiKey`（または対応する環境変数を使用したプロバイダー認証）を
  再利用します。
- ホストの解決順序：`plugins.entries.ollama.config.webSearch.baseUrl` →
  `models.providers.ollama.baseUrl`（または `baseURL`）→ `http://127.0.0.1:11434`。
- 解決されたホストが `https://ollama.com` の場合、OpenClaw は
  API キーを Bearer 認証として使用し、`https://ollama.com/api/web_search` を
  直接呼び出します。
- それ以外の場合、OpenClaw はまずローカルプロキシエンドポイント
  `/api/experimental/web_search`（署名して Ollama Cloud に転送）を呼び出し、その後、
  同じホスト上の `/api/web_search` にフォールバックします。両方とも失敗し、
  `OLLAMA_API_KEY` が設定されている場合、そのキーを使用して
  `https://ollama.com/api/web_search` に対して 1 回再試行します。このキーが
  ローカルホストに送信されることはありません。
- セットアップ中に Ollama に到達できない場合、またはサインインしていない場合、OpenClaw は警告しますが、
  プロバイダーの選択は妨げません。

## 関連項目

- [Web 検索の概要](/ja-JP/tools/web) -- すべてのプロバイダーと自動検出
- [Ollama](/ja-JP/providers/ollama) -- Ollama モデルのセットアップとクラウド／ローカルモード
