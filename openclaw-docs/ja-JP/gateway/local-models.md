---
read_when:
    - 独自の GPU マシンからモデルを提供したい場合
    - LM Studio または OpenAI 互換プロキシを接続する場合
    - 最も安全なローカルモデルに関するガイダンスが必要です
summary: ローカル LLM（LM Studio、vLLM、LiteLLM、カスタム OpenAI エンドポイント）で OpenClaw を実行する
title: ローカルモデル
x-i18n:
    generated_at: "2026-07-26T09:04:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: af76c9e97bd1d3c9665c347944511b4f466f0b620bb8af7b5f95b1e9145aadec
    source_path: gateway/local-models.md
    workflow: 16
---

ローカルモデルは動作しますが、ハードウェア、コンテキストサイズ、プロンプトインジェクション防御に対する要件が高くなります。小規模なモデルや強く量子化されたモデルでは、コンテキストが切り詰められ、プロバイダー側の安全フィルターが適用されません。このページでは、ハイエンドのローカルスタックとカスタムの OpenAI 互換サーバーについて説明します。最も手軽な方法としては、[LM Studio](/ja-JP/providers/lmstudio) または [Ollama](/ja-JP/providers/ollama) と `openclaw onboard` から始めてください。

選択したモデルが必要とするときだけ起動するローカルサーバーについては、[ローカルモデルサービス](/ja-JP/gateway/local-model-services)を参照してください。

## ハードウェアの最低要件

快適なエージェントループを実現するには、**最大構成の Mac Studio 2 台以上、または同等の GPU リグ（約 $30k 以上）**を目安にしてください。単一の **24 GB** GPU で処理できるのは、レイテンシが高い軽量なプロンプトに限られます。常に、**ホスト可能な最大／フルサイズのバリアント**を実行してください。小規模または強く量子化されたチェックポイントは、プロンプトインジェクションのリスクを高めます（[セキュリティ](/ja-JP/gateway/security)を参照）。

## バックエンドを選ぶ

| バックエンド                                         | 使用する場合                                                                  |
| ---------------------------------------------------- | ----------------------------------------------------------------------------- |
| [ds4](/ja-JP/providers/ds4)                                | OpenAI 互換のツール呼び出しを備えた、macOS Metal 上のローカル DeepSeek V4 Flash |
| [LM Studio](/ja-JP/providers/lmstudio)                     | 初回のローカルセットアップ、GUI ローダー、ネイティブ Responses API             |
| LiteLLM / OAI-proxy / カスタム OpenAI 互換プロキシ   | 別のモデル API を仲介し、OpenClaw に OpenAI として扱わせる場合                  |
| MLX / vLLM / SGLang                                  | OpenAI 互換 HTTP エンドポイントによる高スループットのセルフホスト配信           |
| [Ollama](/ja-JP/providers/ollama)                          | CLI ワークフロー、モデルライブラリ、管理不要の systemd サービス                 |

バックエンドが対応している場合（LM Studio は対応）、`api: "openai-responses"` を使用してください。それ以外の場合は `api: "openai-completions"` を使用してください。`baseUrl` を持つカスタムプロバイダーで `api` を省略すると、OpenClaw はデフォルトで `openai-completions` を使用します。

<Warning>
**WSL2 + Ollama + NVIDIA/CUDA:** 公式の Ollama Linux インストーラーは、`Restart=always` を使用する systemd サービスを有効にします。WSL2 の GPU セットアップでは、自動起動によってブート時に最後のモデルが再読み込みされ、ホストメモリが占有されるため、VM が繰り返し再起動する可能性があります。[WSL2 のクラッシュループ](/ja-JP/providers/ollama#troubleshooting)を参照してください。
</Warning>

## LM Studio + 大規模ローカルモデル（Responses API）

これは現在最適なローカルスタックです。LM Studio で大規模モデル（フルサイズの Qwen、DeepSeek、または Llama ビルド）を読み込み、ローカルサーバー（デフォルトは `http://127.0.0.1:1234`）を有効にして、推論を最終テキストから分離するために Responses API を使用します。

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: {
        "anthropic/claude-opus-4-6": { alias: "Opus" },
        "lmstudio/my-local-model": { alias: "Local" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

セットアップチェックリスト：

- LM Studio をインストールする：[https://lmstudio.ai](https://lmstudio.ai)
- **利用可能な最大のモデルビルド**をダウンロードし（「small」や強く量子化されたバリアントは避ける）、サーバーを起動して、`http://127.0.0.1:1234/v1/models` にモデルが表示されることを確認します。
- `my-local-model` を、LM Studio に表示される実際のモデル ID に置き換えます。
- モデルを読み込んだままにします。コールドロードでは起動レイテンシが増加します。
- LM Studio のビルドが異なる場合は、`contextWindow`/`maxTokens` を調整します。
- WhatsApp では、最終テキストだけが送信されるように Responses API を使用してください。
- ホスト型モデルをフォールバックとして引き続き利用できるよう、`models.mode: "merge"` を維持します。

### ハイブリッド構成：ホスト型をプライマリ、ローカルをフォールバックにする

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "anthropic/claude-sonnet-4-6",
        fallbacks: ["lmstudio/my-local-model", "anthropic/claude-opus-4-6"],
      },
      models: {
        "anthropic/claude-sonnet-4-6": { alias: "Sonnet" },
        "lmstudio/my-local-model": { alias: "Local" },
        "anthropic/claude-opus-4-6": { alias: "Opus" },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      lmstudio: {
        baseUrl: "http://127.0.0.1:1234/v1",
        apiKey: "lmstudio",
        api: "openai-responses",
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 196608,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

ホスト型モデルをセーフティネットとして使用するローカル優先構成では、`primary`/`fallbacks` の順序を入れ替え、同じ `providers` ブロックと `models.mode: "merge"` を維持します。

### リージョン別ホスティング／データルーティング

ホスト型の MiniMax/Kimi/GLM バリアントは、リージョンが固定されたエンドポイント（たとえば米国ホスト）として OpenRouter でも提供されています。選択した法域内にトラフィックを維持しつつ、Anthropic/OpenAI のフォールバック用に `models.mode: "merge"` を維持するには、リージョン別バリアントを選択してください。プライバシーを最も強く保護できるのは依然としてローカル限定構成です。プロバイダー機能が必要でありながらデータフローを制御したい場合、ホスト型のリージョン別ルーティングが中間的な選択肢になります。

## その他の OpenAI 互換ローカルプロキシ

MLX（`mlx_lm.server`）、vLLM、SGLang、LiteLLM、OAI-proxy、または任意のカスタム Gateway は、OpenAI 形式の `/v1/chat/completions` エンドポイントを公開していれば動作します。バックエンドに `/v1/responses` 対応が明記されていない限り、`openai-completions` を使用してください。

```json5
{
  agents: {
    defaults: {
      model: { primary: "local/my-local-model" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "sk-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 120000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

カスタム／ローカルプロバイダーのエントリは、local loopback、LAN、tailnet、プライベート DNS ホストを含め、保護されたモデルリクエストについて、設定された正確な `baseUrl` オリジンを信頼します。メタデータ／リンクローカルのオリジンは、これに関係なく常にブロックされます。その他のプライベートオリジンへのリクエストには、引き続き `models.providers.<id>.request.allowPrivateNetwork: true` が必要です。正確なオリジンの信頼を無効にするには、信頼フラグを `false` に設定します。

`models.providers.<id>.models[].id` はプロバイダー内でローカルな値です。プロバイダーのプレフィックスを含めないでください。`mlx_lm.server --model mlx-community/Qwen3-30B-A3B-6bit` で起動した MLX サーバーの場合：

- `models.providers.mlx.models[].id: "mlx-community/Qwen3-30B-A3B-6bit"`
- `agents.defaults.model.primary: "mlx/mlx-community/Qwen3-30B-A3B-6bit"`

ローカルまたはプロキシされたビジョンモデルでは、画像添付ファイルがエージェントターンに挿入されるように `input: ["text", "image"]` を設定してください。対話型のカスタムプロバイダーオンボーディングでは、一般的なビジョンモデル ID を推測し、不明な名前についてのみ質問します。非対話型オンボーディングでも同じ推測を使用し、`--custom-image-input` / `--custom-text-input` で上書きできます。

`agents.defaults.timeoutSeconds` を増やす前に、低速なローカル／リモートモデルサーバーには `models.providers.<id>.timeoutSeconds` を使用してください。プロバイダーのタイムアウトは、モデルの HTTP リクエストに限り、接続、ヘッダー、本文のストリーミング、および保護されたフェッチの中断までの合計時間を対象とします。エージェント／実行のタイムアウトがそれより短い場合は、そちらも増やしてください。プロバイダーのタイムアウトでは、実行全体を延長できません。

<Note>
カスタムの OpenAI 互換プロバイダーでは、`baseUrl` が local loopback、プライベート LAN、`.local`、またはベアホスト名に解決される場合、`apiKey: "ollama-local"` のような秘密ではないローカルマーカーが受け入れられます。OpenClaw はキー不足として報告せず、有効なローカル認証情報として扱います。パブリックホスト名を受け入れるプロバイダーには、実際の値を使用してください。
</Note>

ローカル／プロキシされた `/v1` バックエンドの動作に関する注意事項：

- OpenClaw はこれらをネイティブ OpenAI エンドポイントではなく、プロキシ形式の OpenAI 互換ルートとして扱います。
- ネイティブ OpenAI 専用のリクエスト整形は適用されません。`service_tier`、Responses の `store`、OpenAI の推論互換ペイロード整形、プロンプトキャッシュのヒントはありません。
- 非表示の OpenClaw 帰属ヘッダー（`originator`、`version`、`User-Agent`）は、カスタムプロキシ URL には挿入されません。

互換性宣言は、このプロバイダー行で記述されるカスタムエンドポイントにのみ適用されます。カタログで既知のルートでは、代わりにプロバイダー所有の機能が使用されます。[カスタムプロバイダー機能ガイド](/ja-JP/gateway/config-tools#custom-provider-capability-declarations)を参照してください。

より厳格な OpenAI 互換バックエンド向けの互換性オーバーライド：

- **文字列のみのコンテンツ**：一部のサーバーは、構造化されたコンテンツパート配列ではなく、文字列の `messages[].content` のみを受け入れます。`models.providers.<provider>.models[].compat.requiresStringContent: true` を設定してください。
- **厳格なメッセージキー**：サーバーが `role`/`content` 以外を含むメッセージエントリを拒否する場合は、`compat.strictMessageKeys: true` を設定してください。
- **角括弧で囲まれたツールテキスト**：一部のローカルモデルは、`[tool_name]`、JSON、`[END_TOOL_REQUEST]` の順で、独立した角括弧付きツールリクエストをテキストとして出力します。OpenClaw は、その名前が対象ターンに登録されたツールと完全に一致する場合にのみ、それらを実際のツール呼び出しに昇格させます。それ以外の場合は、非表示の未対応テキストとして残ります。
- **構造化されていないツール呼び出し風テキスト**：モデルがツール呼び出しのように見える JSON/XML/ReAct 形式のテキストを出力しても、それが構造化された呼び出しでなかった場合、OpenClaw はテキストのまま保持し、実行 ID、プロバイダー／モデル、検出されたパターン、および利用可能な場合はツール名を含む警告をログに記録します。これはプロバイダー／モデルの非互換性であり、完了したツール実行ではありません。
- **ツール使用の強制**：ツールがアシスタントのテキスト（未加工の JSON/XML/ReAct、または空の `tool_calls` 配列）として表示される場合は、まずサーバーのチャットテンプレート／パーサーがツール呼び出しに対応していることを確認してください。ツール使用を強制した場合にのみパーサーが動作する場合は、モデルごとにデフォルトのプロキシ値 `tool_choice: "auto"` を上書きします：

  ```json5
  {
    agents: {
      defaults: {
        models: {
          "local/my-local-model": {
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

  これは、通常のすべてのターンでツールを呼び出す必要がある場合にのみ使用してください。`local/my-local-model` を `openclaw models list` の正確な参照に置き換えるか、CLI で設定します：

  ```bash
  openclaw config set agents.defaults.models '{"local/my-local-model":{"params":{"extra_body":{"tool_choice":"required"}}}}' --strict-json --merge
  ```

- **追加の推論エフォート**：カスタムの OpenAI 互換モデルが組み込みプロファイル以外の OpenAI 推論エフォートを受け入れる場合は、モデルの互換性ブロックでそれらを宣言します。`"xhigh"` を追加すると、そのモデル参照について、`/think xhigh`、セッション選択画面、Gateway の検証、および `llm-task` の検証で公開されます：

  ```json5
  {
    models: {
      providers: {
        local: {
          baseUrl: "http://127.0.0.1:8000/v1",
          apiKey: "sk-local",
          api: "openai-responses",
          models: [
            {
              id: "gpt-5.4",
              name: "ローカルプロキシ経由の GPT 5.4",
              reasoning: true,
              input: ["text"],
              cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
              contextWindow: 196608,
              maxTokens: 8192,
              compat: {
                supportedReasoningEfforts: ["low", "medium", "high", "xhigh"],
                reasoningEffortMap: { xhigh: "xhigh" },
              },
            },
          ],
        },
      },
    },
  }
  ```

## より小規模または制約の厳しいバックエンド

モデルが正常に読み込まれても完全なエージェントターンが正しく動作しない場合は、上位から順に確認します。まず通信を確認してから、対象範囲を絞り込みます。

1. **ローカルモデルが応答することを確認** - ツールもエージェントコンテキストも使用しません。

   ```bash
   openclaw infer model run --local --model <provider/model> --prompt "正確に次のように返信してください: pong" --json
   ```

2. **Gateway のルーティングを確認** - プロンプトのみを送信し、トランスクリプト、AGENTS のブートストラップ、コンテキストエンジンの組み立て、ツール、同梱の MCP サーバーを省略しますが、Gateway のルーティング、認証、プロバイダー選択は引き続き実行します。

   ```bash
   openclaw infer model run --gateway --model <provider/model> --prompt "正確に次のように返信してください: pong" --json
   ```

3. 両方のプローブが成功しても、実際のエージェントターンで不正なツール呼び出しや過大なプロンプトにより失敗する場合は、**軽量モードを試します**。`agents.defaults.experimental.localModelLean: true` を設定してください。明示的に必要とされない限り、負荷の高いブラウザー、Cron、メッセージ、メディア生成、音声、PDF の各ツールを除外し、`exec` は直接表示したまま、より大規模なツールカタログをデフォルトで構造化された Tool Search コントロールの背後に配置します。詳細と有効化を確認する方法については、[試験的機能 -> ローカルモデルの軽量モード](/ja-JP/concepts/experimental-features#local-model-lean-mode)を参照してください。

4. 最後の手段として、そのモデルに `models.providers.<provider>.models[].compat.supportsTools: false` を設定して**ツールを完全に無効化**します。これにより、エージェントはツール呼び出しなしで実行されます。

5. **それでも失敗する場合、ボトルネックはアップストリームにあります。** 軽量モードと `supportsTools: false` を使用した後も、より大規模な OpenClaw の実行時に限ってバックエンドが失敗する場合、残る問題は通常、OpenClaw の通信レイヤーではなく、コンテキストウィンドウ、GPU メモリ、kv-cache の退避、バックエンドのバグなど、モデルまたはサーバー自体にあります。

## トラブルシューティング

- **Gateway がプロキシに到達できない場合** `curl http://127.0.0.1:1234/v1/models`。
- **LM Studio のモデルがアンロードされている場合** 再読み込みしてください。コールドスタートは「ハング」する一般的な原因です。
- **ローカルサーバーが `terminated`、`ECONNRESET` と報告するか、ターンの途中でストリームを閉じる場合** OpenClaw は、カーディナリティの低い `model.call.error.failureKind` と OpenClaw プロセスの RSS／ヒープスナップショットを診断情報に記録します。LM Studio／Ollama のメモリ負荷については、そのタイムスタンプをサーバーログまたは macOS のクラッシュ／jetsam ログと照合し、モデルサーバーが強制終了されたかどうかを確認してください。
- **コンテキストエラーが発生する場合** OpenClaw は、検出されたモデルウィンドウ（または `agents.defaults.contextTokens` により縮小された場合はその上限付きウィンドウ）から、コンテキストウィンドウの事前チェックしきい値を導出します。20% 未満では最小値 **8k** で警告し、10% 未満では最小値 **4k** でハードブロックします（過大なモデルメタデータによって有効なユーザー上限が拒否されないよう、有効なコンテキストウィンドウを上限とします）。`contextWindow` を下げるか、サーバー／モデルのコンテキスト上限を引き上げてください。
- **`messages[].content ... expected a string` の場合** そのモデルエントリに `compat.requiresStringContent: true` を追加してください。
- **`validation.keys`、または「メッセージエントリでは `role` と `content` のみが許可されます」と表示される場合** そのモデルエントリに `compat.strictMessageKeys: true` を追加してください。
- **`/v1/chat/completions` の直接呼び出しは動作するものの、Gemma または別のローカルモデルで `openclaw infer model run --local` が失敗する場合** まずプロバイダー URL、モデル参照、認証マーカー、サーバーログを確認してください。`model run` はエージェントツールを完全に省略します。`model run` が成功しても、より大規模なエージェントターンが失敗する場合は、`localModelLean` または `compat.supportsTools: false` を使用してツールの対象範囲を縮小してください。
- **ツール呼び出しが生の JSON／XML／ReAct テキストとして表示されるか、プロバイダーが空の `tool_calls` 配列を返す場合** アシスタントのテキストを無差別にツール実行へ変換するプロキシを追加しないでください。まずサーバーのチャットテンプレート／パーサーを修正してください。ツール使用を強制した場合に限ってモデルが動作する場合は、上記の `params.extra_body.tool_choice: "required"` オーバーライドを追加し、毎ターンのツール呼び出しが想定されるセッションでのみ、そのモデルエントリを使用してください。
- **安全性**：ローカルモデルでは、プロバイダー側のフィルターが省略されます。プロンプトインジェクションの影響範囲を抑えるため、エージェントの対象範囲を限定し、Compaction を有効にしてください。

## 関連項目

- [設定リファレンス](/ja-JP/gateway/configuration-reference)
- [モデルのフェイルオーバー](/ja-JP/concepts/model-failover)
