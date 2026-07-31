---
read_when:
    - OpenClaw で Groq を使用する場合
    - API キーの環境変数または CLI 認証の選択が必要です
    - Groq で Whisper 音声文字起こしを設定しています
summary: Groq のセットアップ（認証 + モデル選択 + Whisper 文字起こし）
title: Groq
x-i18n:
    generated_at: "2026-07-26T09:16:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f04f9365127c72aa2f976f453e5d11657b19d6b4a57de1179b88924744db1dc1
    source_path: providers/groq.md
    workflow: 16
---

[Groq](https://groq.com) は、カスタム LPU ハードウェアを使用し、オープンウェイトモデル（Llama、Gemma、Kimi、Qwen、GPT OSS など）で超高速な推論を提供します。Groq Plugin は、OpenAI 互換チャットプロバイダーと音声メディア理解プロバイダーの両方を登録します。

| プロパティ             | 値                                       |
| ---------------------- | ---------------------------------------- |
| プロバイダー ID        | `groq`                       |
| Plugin                 | 公式外部パッケージ                       |
| 認証環境変数           | `GROQ_API_KEY`                       |
| API                    | OpenAI 互換（`openai-completions`）        |
| ベース URL             | `https://api.groq.com/openai/v1`                       |
| 音声文字起こし         | `whisper-large-v3-turbo`（デフォルト）         |
| 推奨チャットデフォルト | `groq/llama-3.3-70b-versatile`                       |

## Plugin のインストール

公式 Plugin をインストールし、Gateway を再起動します。

```bash
openclaw plugins install @openclaw/groq-provider
openclaw gateway restart
```

## はじめに

<Steps>
  <Step title="API キーを取得する">
    [console.groq.com/keys](https://console.groq.com/keys) で API キーを作成します。
  </Step>
  <Step title="API キーを設定する">
    ```bash
export GROQ_API_KEY=gsk_...
```
  </Step>
  <Step title="デフォルトモデルを設定する">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "groq/llama-3.3-70b-versatile" },
        },
      },
    }
    ```
  </Step>
  <Step title="カタログにアクセスできることを確認する">
    ```bash
    openclaw models list --provider groq
    ```
  </Step>
</Steps>

### 設定ファイルの例

```json5
{
  env: { GROQ_API_KEY: "gsk_..." },
  agents: {
    defaults: {
      model: { primary: "groq/llama-3.3-70b-versatile" },
    },
  },
}
```

## 組み込みカタログ

OpenClaw には、推論エントリと非推論エントリの両方を含む、マニフェストベースの Groq カタログが付属しています。インストール済みバージョンの静的な行を確認するには `openclaw models list --provider groq` を実行します。Groq の正式な一覧については、[console.groq.com/docs/models](https://console.groq.com/docs/models) を確認してください。

| モデル参照                                       | 名前                    | 推論 | 入力             | コンテキスト |
| ------------------------------------------------ | ----------------------- | ---- | ---------------- | ------------ |
| `groq/llama-3.3-70b-versatile`                               | Llama 3.3 70B Versatile | なし | テキスト         | 131,072      |
| `groq/llama-3.1-8b-instant`                               | Llama 3.1 8B Instant    | なし | テキスト         | 131,072      |
| `groq/meta-llama/llama-4-scout-17b-16e-instruct`                               | Llama 4 Scout 17B       | なし | テキスト + 画像  | 131,072      |
| `groq/openai/gpt-oss-120b`                               | GPT OSS 120B            | あり | テキスト         | 131,072      |
| `groq/openai/gpt-oss-20b`                               | GPT OSS 20B             | あり | テキスト         | 131,072      |
| `groq/openai/gpt-oss-safeguard-20b`                               | Safety GPT OSS 20B      | あり | テキスト         | 131,072      |
| `groq/qwen/qwen3-32b`                               | Qwen3 32B               | あり | テキスト         | 131,072      |
| `groq/groq/compound`                               | Compound                | あり | テキスト         | 131,072      |
| `groq/groq/compound-mini`                               | Compound Mini           | あり | テキスト         | 131,072      |

<Tip>
  カタログは OpenClaw のリリースごとに更新されます。`openclaw models list --provider groq` では、インストール済みバージョンが認識している行を確認できます。新しく追加されたモデルや非推奨になったモデルについては、[console.groq.com/docs/models](https://console.groq.com/docs/models) と照合してください。
</Tip>

## 推論モデル

Groq の推論モデル（上の表の `reasoning: true`）では、OpenClaw の共通 `/think` レベルが、`low`、`medium`、または `high` の `reasoning_effort` 値に対応付けられます。`/think off` または `/think none` の場合、無効化された値を送信するのではなく、リクエストから `reasoning_effort` が省略されます。

共通の `/think` レベルと、OpenClaw がプロバイダーごとにそれらを変換する方法については、[思考モード](/ja-JP/tools/thinking)を参照してください。

## 音声文字起こし

Groq Plugin は、共有 `tools.media.audio` サーフェスを通じて音声メッセージを文字起こしできるよう、**音声メディア理解プロバイダー**も登録します。

| プロパティ       | 値                                        |
| ---------------- | ----------------------------------------- |
| 共通設定パス     | `tools.media.audio`                        |
| デフォルトベース URL | `https://api.groq.com/openai/v1`                    |
| デフォルトモデル | `whisper-large-v3-turbo`                        |
| 自動優先度       | 20                                        |
| API エンドポイント | OpenAI 互換 `/audio/transcriptions`          |

Groq をデフォルトの音声バックエンドにするには、次のように設定します。

```json5
{
  tools: {
    media: {
      audio: {
        models: [{ provider: "groq" }],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="デーモンでの環境変数の利用可否">
    Gateway が管理対象サービス（launchd、systemd、Docker）として実行されている場合、`GROQ_API_KEY` は対話型シェルだけでなく、そのプロセスからも参照できる必要があります。

    <Warning>
      対話型シェルでのみエクスポートされたキーは、その環境も取り込まない限り、launchd または systemd デーモンでは使用できません。Gateway プロセスから読み取れるようにするには、`~/.openclaw/.env` または `env.shellEnv` を使用してキーを設定してください。
    </Warning>

  </Accordion>

  <Accordion title="カスタム Groq モデル ID">
    OpenClaw は実行時に任意の Groq モデル ID を受け入れます。Groq に表示される正確な ID を使用し、先頭に `groq/` を付けてください。静的カタログは一般的なケースを網羅しています。カタログにない ID には、デフォルトの OpenAI 互換テンプレートが適用されます。

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "groq/<your-model-id>" },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## 関連情報

<CardGroup cols={2}>
  <Card title="モデルプロバイダー" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="思考モード" href="/ja-JP/tools/thinking" icon="brain">
    推論作業量のレベルとプロバイダーポリシーの相互作用。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/configuration-reference" icon="gear">
    プロバイダーと音声設定を含む完全な設定スキーマ。
  </Card>
  <Card title="Groq Console" href="https://console.groq.com" icon="arrow-up-right-from-square">
    Groq のダッシュボード、API ドキュメント、料金。
  </Card>
</CardGroup>
