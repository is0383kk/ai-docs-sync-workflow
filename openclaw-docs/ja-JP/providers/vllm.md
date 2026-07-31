---
read_when:
    - ローカルの vLLM サーバーで OpenClaw を実行する場合
    - 独自のモデルで OpenAI 互換の /v1 エンドポイントを使用したい場合
summary: vLLM（OpenAI 互換ローカルサーバー）で OpenClaw を実行する
title: vLLM
x-i18n:
    generated_at: "2026-07-26T09:17:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98d1044c0a82efb6c9937e961d765d0cfcea8664cbaa043168921b457756512c
    source_path: providers/vllm.md
    workflow: 16
---

vLLM は、**OpenAI 互換** HTTP API を通じてオープンソース（および一部のカスタム）モデルを提供します。OpenClaw は `openai-completions` API を使用して接続し、`VLLM_API_KEY` でオプトインするとモデルを**自動検出**できます。

| プロパティ       | 値                                         |
| ---------------- | ------------------------------------------ |
| プロバイダー ID  | `vllm`                         |
| API              | `openai-completions`（OpenAI 互換）          |
| 認証             | `VLLM_API_KEY` 環境変数                |
| デフォルトのベース URL | `http://127.0.0.1:8000/v1`                    |
| ストリーミング使用量 | サポート（`stream_options.include_usage`）          |

## はじめに

<Steps>
  <Step title="OpenAI 互換サーバーで vLLM を起動する">
    ベース URL は `/v1` エンドポイント（`/v1/models`、`/v1/chat/completions`）を公開する必要があります。vLLM は通常、次の場所で実行されます。

    ```text
    http://127.0.0.1:8000/v1
    ```

  </Step>
  <Step title="API キー環境変数を設定する">
    サーバーが認証を強制しない場合、空でない任意の値を使用できます。

    ```bash
    export VLLM_API_KEY="vllm-local"
    ```

  </Step>
  <Step title="モデルを選択する">
    vLLM のモデル ID のいずれかに置き換えます。

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "vllm/your-model-id" },
        },
      },
    }
    ```

  </Step>
  <Step title="モデルが利用可能か確認する">
    ```bash
    openclaw models list --provider vllm
    ```
  </Step>
</Steps>

<Tip>
非対話型セットアップ（CI、スクリプト）では、ベース URL、キー、モデルを直接渡します。

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice vllm \
  --custom-base-url "http://127.0.0.1:8000/v1" \
  --custom-api-key "vllm-local" \
  --custom-model-id "your-model-id"
```

</Tip>

## モデル検出（暗黙的プロバイダー）

`VLLM_API_KEY` が設定されている（または認証プロファイルが存在する）場合に、`models.providers.vllm` が定義されて**いなければ**、OpenClaw は `GET http://127.0.0.1:8000/v1/models` に問い合わせ、返された ID をモデルエントリに変換します。

<Note>
`models.providers.vllm` を明示的に設定すると、OpenClaw は宣言されたモデルのみを使用します。OpenClaw が設定済みプロバイダーの `/models` エンドポイントにも問い合わせ、公開されているすべての vLLM モデルを含めるようにするには、`"vllm/*": {}` を `agents.defaults.models` に追加します。
</Note>

## 明示的な設定

vLLM が別のホストまたはポートで実行される場合、`contextWindow`/`maxTokens` を固定する場合、サーバーが実際の API キーを要求する場合、または信頼された loopback、LAN、Tailscale エンドポイントに接続する場合は、明示的に設定します。

```json5
{
  models: {
    providers: {
      vllm: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "${VLLM_API_KEY}",
        api: "openai-completions",
        timeoutSeconds: 300, // 任意: 低速なローカルモデル向けにリクエストのタイムアウトを延長
        models: [
          {
            id: "your-model-id",
            name: "ローカル vLLM モデル",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

すべてのモデルを列挙せずにプロバイダーを動的に保つには、表示されるモデルカタログにワイルドカードを追加します。

```json5
{
  agents: {
    defaults: {
      models: {
        "vllm/*": {},
      },
    },
  },
}
```

## 高度な設定

<AccordionGroup>
  <Accordion title="プロキシ形式の動作">
    vLLM は、ネイティブ OpenAI エンドポイントではなく、プロキシ形式の OpenAI 互換 `/v1` バックエンドとして扱われます。

    | 動作                                    | 適用されるか                     |
    | --------------------------------------- | -------------------------------- |
    | ネイティブ OpenAI リクエスト整形        | いいえ                           |
    | `service_tier`                      | 送信されない                     |
    | Responses `store`            | 送信されない                     |
    | プロンプトキャッシュのヒント            | 送信されない                     |
    | OpenAI reasoning 互換ペイロードの整形   | 適用されない                     |
    | 非表示の OpenClaw 帰属ヘッダー           | カスタムベース URL では挿入されない |

  </Accordion>

  <Accordion title="Qwen の thinking 制御">
    Qwen モデルでは、サーバーが Qwen チャットテンプレートの kwargs を期待する場合、モデル行に `compat.thinkingFormat: "qwen-chat-template"` を設定します。Qwen チャットテンプレートの thinking は OpenAI 形式の effort 段階ではなくオン/オフのフラグであるため、これらのモデルは二値の `/think` プロファイル（`off`、`on`）を公開します。

    ```json5
    {
      models: {
        providers: {
          vllm: {
            models: [
              {
                id: "Qwen/Qwen3-8B",
                name: "Qwen3 8B",
                reasoning: true,
                compat: { thinkingFormat: "qwen-chat-template" },
              },
            ],
          },
        },
      },
    }
    ```

    OpenClaw は `/think off` を次のようにマッピングします。

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "preserve_thinking": true
      }
    }
    ```

    `off` 以外の thinking レベルでは `enable_thinking: true` が送信されます。エンドポイントが代わりに DashScope 形式のトップレベルフラグを期待する場合は、`compat.thinkingFormat: "qwen"` を使用してリクエストルートに `enable_thinking` を送信します。

  </Accordion>

  <Accordion title="Nemotron 3 の thinking 制御">
    thinking がオフの `vllm/nemotron-3-*` モデルでは、同梱 Plugin が次を送信します。

    ```json
    {
      "chat_template_kwargs": {
        "enable_thinking": false,
        "force_nonempty_content": true
      }
    }
    ```

    これらの値をカスタマイズするには、モデルの params 配下に `chat_template_kwargs` を設定します。`params.extra_body.chat_template_kwargs` も設定した場合、`extra_body` が最後のリクエスト本文オーバーライドであるため、その値が優先されます。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/nemotron-3-super": {
              params: {
                chat_template_kwargs: {
                  enable_thinking: false,
                  force_nonempty_content: true,
                },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="Qwen のツール呼び出しがテキストとして表示される">
    まず、モデルに適したツール呼び出しパーサーとチャットテンプレートを指定して vLLM が起動されたことを確認します。vLLM では、Qwen2.5 モデル向けに `hermes`、Qwen3-Coder モデル向けに `qwen3_xml` が記載されています。

    症状: Skills/ツールがまったく実行されない、アシスタントが `{"name":"read","arguments":...}` のような未加工の JSON/XML を出力する、または OpenClaw が `tool_choice: "auto"` を送信したときに vLLM が空の `tool_calls` 配列を返す。

    一部の Qwen/vLLM の組み合わせでは、リクエストで `tool_choice: "required"` を使用した場合にのみ、構造化されたツール呼び出しが返されます。`params.extra_body` を使用してモデルごとに強制します。

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "vllm/Qwen-Qwen2.5-Coder-32B-Instruct": {
              params: {
                extra_body: {
                  tool_choice: "required",
                },
              },
            },
          },
        },
      },
    }
    ```

    モデル ID を `openclaw models list --provider vllm` の正確な ID に置き換えるか、CLI から同じオーバーライドを適用します。

    ```bash
    openclaw config set agents.defaults.models '{"vllm/Qwen-Qwen2.5-Coder-32B-Instruct":{"params":{"extra_body":{"tool_choice":"required"}}}}' --strict-json --merge
    ```

    これはオプトインの回避策です。ツールを使用するすべてのターンでツール呼び出しを強制するため、許容できる専用のモデルエントリにのみ使用してください。すべての vLLM モデルのグローバルデフォルトとして設定しないでください。また、任意のアシスタントテキストを実行可能なツール呼び出しに変換するプロキシと組み合わせないでください。

  </Accordion>

  <Accordion title="カスタムベース URL">
    vLLM サーバーがデフォルト以外のホストまたはポートで実行されている場合は、明示的なプロバイダー設定で `baseUrl` を設定します。

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:9000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [
              {
                id: "my-custom-model",
                name: "リモート vLLM モデル",
                reasoning: false,
                input: ["text"],
                contextWindow: 64000,
                maxTokens: 4096,
              },
            ],
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## トラブルシューティング

<AccordionGroup>
  <Accordion title="最初の応答が遅い、またはリモートサーバーがタイムアウトする">
    大規模なローカルモデル、リモート LAN ホスト、または tailnet リンクでは、プロバイダー固有のリクエストタイムアウトを設定します。

    ```json5
    {
      models: {
        providers: {
          vllm: {
            baseUrl: "http://192.168.1.50:8000/v1",
            apiKey: "${VLLM_API_KEY}",
            api: "openai-completions",
            timeoutSeconds: 300,
            models: [{ id: "your-model-id", name: "ローカル vLLM モデル" }],
          },
        },
      },
    }
    ```

    `timeoutSeconds` は、vLLM モデルへの HTTP リクエスト（接続の確立、レスポンスヘッダー、本文のストリーミング、および保護された fetch 全体の中止）にのみ適用されます。また、このプロバイダーの LLM アイドル/ストリーム監視の上限を暗黙的なデフォルトの約 120s より長くします。エージェント実行全体を制御する `agents.defaults.timeoutSeconds` を増やすよりも、こちらを優先してください。

  </Accordion>

  <Accordion title="サーバーに接続できない">
    vLLM サーバーが実行中で、アクセス可能であることを確認します。

    ```bash
    curl http://127.0.0.1:8000/v1/models
    ```

    接続エラーが表示される場合は、ホスト、ポート、および vLLM が OpenAI 互換サーバーモードで起動されたことを確認します。OpenClaw は、loopback、LAN、Tailscale エンドポイントに対する保護されたモデルリクエストについて、設定された正確な `models.providers.vllm.baseUrl` オリジンを信頼します。メタデータ/link-local オリジンは、明示的にオプトインしない限り引き続きブロックされます。vLLM リクエストが別のプライベートオリジンに到達する必要がある場合にのみ `models.providers.vllm.request.allowPrivateNetwork: true` を設定するか、正確なオリジンの信頼を無効にするには `false` を設定します。

  </Accordion>

  <Accordion title="リクエストの認証エラー">
    リクエストが認証エラーで失敗する場合は、サーバー設定と一致する実際の `VLLM_API_KEY` を設定するか、`models.providers.vllm` 配下でプロバイダーを明示的に設定します。

    <Tip>
    vLLM サーバーが認証を強制しない場合、`VLLM_API_KEY` の空でない任意の値が、OpenClaw に対するオプトイン信号として機能します。
    </Tip>

  </Accordion>

  <Accordion title="モデルが検出されない">
    自動検出には `VLLM_API_KEY` の設定が必要です。`models.providers.vllm` を定義している場合、`agents.defaults.models` に `"vllm/*": {}` が含まれていない限り、OpenClaw は宣言されたモデルのみを使用します。
  </Accordion>

  <Accordion title="ツールが未加工のテキストとして表示される">
    Qwen モデルが Skills を実行せずに JSON/XML のツール構文を出力する場合:

    - そのモデルに適したパーサー/テンプレートを使用して vLLM を起動します。
    - `openclaw models list --provider vllm` で正確なモデル ID を確認します。
    - `tool_choice: "auto"` が依然として空またはテキストのみのツール呼び出しを返す場合に限り、モデルごとに専用の `params.extra_body.tool_choice: "required"` オーバーライドを追加します。

  </Accordion>
</AccordionGroup>

<Warning>
詳細なヘルプ: [トラブルシューティング](/ja-JP/help/troubleshooting)および[よくある質問](/ja-JP/help/faq)。
</Warning>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="OpenAI" href="/ja-JP/providers/openai" icon="bolt">
    ネイティブ OpenAI プロバイダーと OpenAI 互換ルートの動作。
  </Card>
  <Card title="OAuth と認証" href="/ja-JP/gateway/authentication" icon="key">
    認証の詳細と認証情報の再利用ルール。
  </Card>
  <Card title="トラブルシューティング" href="/ja-JP/help/troubleshooting" icon="wrench">
    よくある問題とその解決方法。
  </Card>
</CardGroup>
