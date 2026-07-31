---
read_when:
    - OpenClaw でオープンモデルを無料で使用したい場合
    - NVIDIA_API_KEY の設定が必要です
    - NVIDIA 経由で Nemotron 3 Ultra を使用する場合
summary: OpenClaw で NVIDIA の OpenAI 互換 API を使用する
title: NVIDIA
x-i18n:
    generated_at: "2026-07-26T09:16:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b5ac7bcc19400a661b2f2861a1dd4d2306c94e445783929e342e9184003314e9
    source_path: providers/nvidia.md
    workflow: 16
---

NVIDIA は、OpenAI 互換 API を通じてオープンモデルを無料で提供しています。エンドポイントは
`https://integrate.api.nvidia.com/v1` で、認証には
[build.nvidia.com](https://build.nvidia.com/settings/api-keys) で取得した API キーを使用します。OpenClaw
では、NVIDIA プロバイダーのデフォルトとして Nemotron 3 Ultra が設定されています。これは、長いコンテキストを扱うエージェント型タスク向けの、総パラメーター数 550B／有効パラメーター数 55B の
NVIDIA 製推論モデルです。

## はじめに

<Steps>
  <Step title="API キーを取得する">
    [build.nvidia.com](https://build.nvidia.com/settings/api-keys) で API キーを作成します。
  </Step>
  <Step title="キーをエクスポートしてオンボーディングを実行する">
    ```bash
    export NVIDIA_API_KEY="nvapi-..."
    openclaw onboard --auth-choice nvidia-api-key
    ```
  </Step>
  <Step title="NVIDIA モデルを設定する">
    ```bash
    openclaw models set nvidia/nvidia/nemotron-3-ultra-550b-a55b
    ```
  </Step>
</Steps>

非対話型セットアップでは、キーを直接渡します。

```bash
openclaw onboard --auth-choice nvidia-api-key --nvidia-api-key "nvapi-..."
```

<Warning>
`--nvidia-api-key` を使用すると、キーがシェル履歴と `ps` の出力に残ります。可能な場合は
`NVIDIA_API_KEY` 環境変数を使用してください。
</Warning>

## 設定例

```json5
{
  env: { NVIDIA_API_KEY: "nvapi-..." },
  models: {
    providers: {
      nvidia: {
        baseUrl: "https://integrate.api.nvidia.com/v1",
        api: "openai-completions",
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "nvidia/nvidia/nemotron-3-ultra-550b-a55b" },
    },
  },
}
```

## 注目モデルカタログ

NVIDIA API キーが設定されている場合、セットアップおよびモデル選択の処理では
`https://assets.ngc.nvidia.com/products/api-catalog/featured-models.json` から
NVIDIA の公開注目モデルカタログを取得し、その結果を 24 時間キャッシュします（先頭 32 件を自由記述のテキスト入力行としてインポートします）。
そのため、build.nvidia.com に追加された新しい注目モデルは、OpenClaw のリリースを待たずにセットアップ画面とモデル選択画面に表示されます。
ライブフィードが利用可能な場合、NVIDIA のセットアップ時には最初に返されたモデルがあらかじめ選択されます。

取得処理では、`assets.ngc.nvidia.com` に対して固定の HTTPS ホストポリシーを使用します。NVIDIA API キーが設定されていない場合、またはフィードが利用できないか不正な形式の場合、
OpenClaw は以下の同梱カタログと同梱デフォルトにフォールバックします。

## Nemotron 3 Ultra

Nemotron 3 Ultra は、OpenClaw におけるデフォルトの NVIDIA モデルです。NVIDIA の
[`nvidia/nemotron-3-ultra-550b-a55b`](https://build.nvidia.com/nvidia/nemotron-3-ultra-550b-a55b)
のビルドページには、1M トークンのコンテキスト仕様を備えた、利用可能な無料エンドポイントとして掲載されています。

同梱されている Ultra の行では、通常のチャット出力が推論テキストとして露出せず、表示される回答に収まるよう、デフォルトで
`chat_template_kwargs: { enable_thinking: false, force_nonempty_content: true }`
を送信します。

NVIDIA のデフォルトとして最高の性能が必要な場合は Ultra を使用してください。より小規模な Nemotron 3 を使用する場合は Super を選択したままにするか、コンテキスト、レイテンシー、動作がより適している場合は NVIDIA のカタログでホストされているサードパーティ製モデルを選択してください。

## 同梱フォールバックカタログ

選択可能な同梱行は、NVIDIA の注目モデルカタログのスナップショットです。非推奨の
互換性用行は完全一致する参照によって引き続き解決できますが、モデル選択画面には表示されません。

| モデル参照                                 | 名前                  | コンテキスト | 最大出力 |
| ------------------------------------------ | --------------------- | ------------ | -------- |
| `nvidia/nvidia/nemotron-3-ultra-550b-a55b` | Nemotron 3 Ultra 550B | 1,048,576 | 8,192      |
| `nvidia/nvidia/nemotron-3-super-120b-a12b` | Nemotron 3 Super 120B | 1,000,000 | 8,192      |
| `nvidia/z-ai/glm-5.2`                      | GLM 5.2               | 202,752   | 8,192      |
| `nvidia/moonshotai/kimi-k2.6`              | Kimi K2.6             | 262,144   | 8,192      |
| `nvidia/minimaxai/minimax-m3`              | Minimax M3            | 196,608   | 8,192      |
| `nvidia/deepseek-ai/deepseek-v4-pro`       | DeepSeek V4 Pro       | 262,144   | 16,384     |
| `nvidia/qwen/qwen3.5-397b-a17b`            | Qwen3.5 397B A17B     | 262,144   | 16,384     |

完全な互換性カタログには、既存の設定用としてリリース済みの次の参照も保持されています：
`nvidia/moonshotai/kimi-k2.5`、`nvidia/z-ai/glm-5.1`、
`nvidia/minimaxai/minimax-m2.5`、`nvidia/z-ai/glm5`、および
`nvidia/minimaxai/minimax-m2.7`。これらは完全一致する参照によって引き続き利用できますが、
オンボーディングやモデル選択画面には表示されません。

## 高度な設定

<AccordionGroup>
  <Accordion title="自動有効化の動作">
    `NVIDIA_API_KEY` 環境変数が設定されているか、オンボーディング中にキーが保存されている場合、
    プロバイダーは自動的に有効になります。キー以外に明示的なプロバイダー設定は
    必要ありません。
  </Accordion>

  <Accordion title="カタログと料金">
    NVIDIA 認証が設定されている場合、OpenClaw は NVIDIA の公開注目モデルカタログを優先し、
    24 時間キャッシュします。選択可能な同梱フォールバックは、
    NVIDIA の注目モデルカタログの静的スナップショットです。完全一致する参照用の非推奨
    互換性行はモデル選択画面に表示されません。NVIDIA は現在、掲載モデルへの API アクセスを無料で提供しているため、
    ソース内のコストはデフォルトで `0` に設定されています。
  </Accordion>

  <Accordion title="OpenAI 互換エンドポイント">
    OpenClaw は、標準の `/v1` チャット補完ルートに対して
    `openai-completions` アダプターを使用し、NVIDIA と通信します。OpenAI 互換ツールであれば、
    NVIDIA のベース URL を指定するだけで動作します。
  </Accordion>

  <Accordion title="Nemotron 3 Ultra の推論パラメーター">
    NVIDIA の Ultra サンプルリクエストでは、推論出力に `chat_template_kwargs.enable_thinking`
    と `reasoning_budget` を使用します。OpenClaw に同梱されている Ultra の行では、
    通常のチャット用途向けに、テンプレート思考がデフォルトで無効になっています。NVIDIA の推論出力を
    有効にする必要がある場合、または NVIDIA 固有のその他のリクエストフィールドを強制的に指定する場合は、
    モデル単位のパラメーターを設定し、プロバイダー固有のオーバーライドを NVIDIA モデルの
    スコープ内に限定してください。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "nvidia/nvidia/nemotron-3-ultra-550b-a55b": {
              params: {
                chat_template_kwargs: { enable_thinking: true },
                extra_body: { reasoning_budget: 16384 },
              },
            },
          },
        },
      },
    }
    ```

    `params.chat_template_kwargs` はオブジェクト全体を置き換えるのではなく、リクエストにすでに存在する
    `chat_template_kwargs` にマージされます。
    `params.extra_body` は OpenAI 互換リクエスト本文に対する最終オーバーライドであり、
    競合するペイロードキーを上書きします。そのため、選択したエンドポイント向けに NVIDIA が
    文書化しているフィールドにのみ使用してください。

  </Accordion>

  <Accordion title="応答が遅いカスタムプロバイダー">
    NVIDIA でホストされている一部のカスタムモデルでは、最初の応答チャンクが送信されるまでに、
    デフォルトの約 120 秒のモデルアイドル監視時間を超える場合があります。カスタム
    NVIDIA プロバイダーエントリでは、エージェントランタイム全体のタイムアウトではなく、
    プロバイダーのタイムアウトを延長してください。`timeoutSeconds` はプロバイダーの HTTP リクエストに適用され、
    そのプロバイダーのアイドル／ストリーム監視時間の上限も引き上げます。

    ```json5
    {
      models: {
        providers: {
          "custom-integrate-api-nvidia-com": {
            baseUrl: "https://integrate.api.nvidia.com/v1",
            api: "openai-completions",
            apiKey: "NVIDIA_API_KEY",
            timeoutSeconds: 300,
          },
        },
      },
      agents: {
        defaults: {
          models: {
            "custom-integrate-api-nvidia-com/meta/llama-3.1-70b-instruct": {
              params: { thinking: "off" },
            },
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

<Tip>
NVIDIA モデルは現在無料で利用できます。最新の提供状況と
レート制限の詳細については、[build.nvidia.com](https://build.nvidia.com/) を確認してください。
</Tip>

## 関連情報

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択方法。
  </Card>
  <Card title="設定リファレンス" href="/ja-JP/gateway/configuration-reference" icon="gear">
    エージェント、モデル、プロバイダーの完全な設定リファレンス。
  </Card>
</CardGroup>
