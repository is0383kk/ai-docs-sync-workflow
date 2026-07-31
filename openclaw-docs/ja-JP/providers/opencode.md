---
read_when:
    - OpenCode でホストされているモデルへのアクセスが必要な場合
    - Zen カタログと Go カタログのどちらかを選択したい場合
summary: OpenCode Zen および Go カタログを OpenClaw で使用する
title: OpenCode
x-i18n:
    generated_at: "2026-07-26T10:28:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: de287eb8a349f26c265f95b8b1de3af4035aa2bdc3501c7279f714d297bb8b9b
    source_path: providers/opencode.md
    workflow: 16
---

OpenCode は、OpenClaw で 2 つのホスト型カタログを提供します。

| カタログ | プレフィックス            | ランタイムプロバイダー |
| ------- | ----------------- | ---------------- |
| **Zen** | `opencode/...`    | `opencode`       |
| **Go**  | `opencode-go/...` | `opencode-go`    |

両方のカタログは、1 つの OpenCode API キー（`OPENCODE_API_KEY`、エイリアス
`OPENCODE_ZEN_API_KEY`）を共有します。OpenClaw は、上流のモデル別ルーティングを正しく保つために
ランタイムプロバイダー ID を分離していますが、オンボーディングとドキュメントでは
1 つの OpenCode セットアップとして扱います。

## はじめに

<Tabs>
  <Tab title="Zen カタログ">
    **最適な用途:** 厳選された OpenCode マルチモデルプロキシ（Claude、GPT、Gemini、GLM、
    DeepSeek、Kimi、MiniMax、Qwen）。

    <Steps>
      <Step title="オンボーディングを実行">
        ```bash
        openclaw onboard --auth-choice opencode-zen
        ```

        または、キーを直接渡します。

        ```bash
        openclaw onboard --opencode-zen-api-key "$OPENCODE_API_KEY"
        ```
      </Step>
      <Step title="Zen モデルをデフォルトに設定">
        ```bash
        openclaw config set agents.defaults.model.primary "opencode/claude-opus-4-6"
        ```
      </Step>
      <Step title="モデルが利用可能であることを確認">
        ```bash
        openclaw models list --provider opencode
        ```
      </Step>
    </Steps>

  </Tab>

  <Tab title="Go カタログ">
    **最適な用途:** OpenCode がホストする Kimi、GLM、MiniMax、Qwen、DeepSeek のラインナップ。

    <Steps>
      <Step title="オンボーディングを実行">
        ```bash
        openclaw onboard --auth-choice opencode-go
        ```

        または、キーを直接渡します。

        ```bash
        openclaw onboard --opencode-go-api-key "$OPENCODE_API_KEY"
        ```
      </Step>
      <Step title="Go モデルをデフォルトに設定">
        ```bash
        openclaw config set agents.defaults.model.primary "opencode-go/kimi-k2.6"
        ```
      </Step>
      <Step title="モデルが利用可能であることを確認">
        ```bash
        openclaw models list --provider opencode-go
        ```
      </Step>
    </Steps>

  </Tab>
</Tabs>

## 設定例

```json5
{
  env: { OPENCODE_API_KEY: "sk-..." },
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

## 組み込みカタログ

### Zen

| プロパティ         | 値                                                                                         |
| ---------------- | --------------------------------------------------------------------------------------------- |
| ランタイムプロバイダー | `opencode`                                                                                    |
| モデル例   | `opencode/claude-opus-4-6`、`opencode/gpt-5.5`、`opencode/gemini-3.1-pro`、`opencode/glm-5.2` |

現在の完全な一覧を確認するには `openclaw models list --provider opencode` を実行してください。
この一覧には、`opencode/big-pickle` や `opencode/deepseek-v4-flash-free` などの無料枠の行も
含まれます。

### Go

| プロパティ         | 値                                                                    |
| ---------------- | ------------------------------------------------------------------------ |
| ランタイムプロバイダー | `opencode-go`                                                            |
| モデル例   | `opencode-go/kimi-k2.6`、`opencode-go/glm-5`、`opencode-go/minimax-m2.5` |

Go モデルの完全な表については、[OpenCode Go](/ja-JP/providers/opencode-go) を参照してください。

## 高度な設定

<AccordionGroup>
  <Accordion title="API キーのエイリアス">
    `OPENCODE_ZEN_API_KEY` は、`OPENCODE_API_KEY` のエイリアスとしても使用できます。
  </Accordion>

  <Accordion title="共有認証情報">
    セットアップ中に OpenCode キーを 1 つ入力すると、両方のランタイム
    プロバイダーの認証情報が保存されます。各カタログを個別にオンボーディングする必要はありません。
  </Accordion>

  <Accordion title="API キーの取得">
    OpenCode アカウントを作成し、
    [opencode.ai/auth](https://opencode.ai/auth) で API キーを生成します。請求とカタログの
    利用可否は OpenCode ダッシュボードから管理します。
  </Accordion>

  <Accordion title="Gemini のリプレイ動作">
    Gemini を基盤とする OpenCode 参照はプロキシ Gemini パスに留まるため、OpenClaw は
    ネイティブ Gemini のリプレイ検証やブートストラップ書き換えを有効にせず、そのパスで
    Gemini の思考シグネチャのサニタイズを維持します。
  </Accordion>

  <Accordion title="Gemini 以外のリプレイ動作">
    Gemini 以外の OpenCode 参照では、最小限の OpenAI 互換リプレイポリシーが維持されます。
  </Accordion>
</AccordionGroup>

## 関連情報

<CardGroup cols={2}>
  <Card title="OpenCode Go" href="/ja-JP/providers/opencode-go" icon="server">
    Go カタログの完全なリファレンス。
  </Card>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/configuration-reference" icon="gear">
    エージェント、モデル、プロバイダーの完全な設定リファレンス。
  </Card>
</CardGroup>
