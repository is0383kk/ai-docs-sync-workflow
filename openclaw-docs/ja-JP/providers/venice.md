---
read_when:
    - OpenClaw でプライバシー重視の推論を利用したい場合
    - Venice AI のセットアップ手順を確認したい場合
summary: OpenClaw で Venice AI のプライバシー重視モデルを使用する
title: Venice AI
x-i18n:
    generated_at: "2026-07-26T10:18:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 13c32b783394eb3092ff94a532b69e34c00624127b0e76e4e2812751d39073a1
    source_path: providers/venice.md
    workflow: 16
---

[Venice AI](https://venice.ai) はプライバシーを重視した推論を提供します。オープンモデルは
ログを記録せずに実行され、さらに Claude、GPT、Gemini、Grok への匿名化プロキシアクセスも利用できます。
すべてのエンドポイントは OpenAI 互換です（`/v1`）。

## プライバシーモード

| モード           | 動作                                                         | モデル                                                        |
| -------------- | ---------------------------------------------------------------- | ------------------------------------------------------------- |
| **プライベート**    | プロンプトと応答は保存もログ記録もされません。一時的にのみ保持されます。         | Llama、Qwen、DeepSeek、Kimi、MiniMax、Venice Uncensored など |
| **匿名化** | 転送前にメタデータを除去し、Venice 経由でプロキシされます。 | Claude、GPT、Gemini、Grok                                     |

<Warning>
匿名化モデルは完全にプライベートではありません。Venice は転送前にメタデータを除去しますが、基盤となるプロバイダー（OpenAI、Anthropic、Google、xAI）は引き続きリクエストを処理します。完全なプライバシーが必要な場合は、プライベートモデルを使用してください。
</Warning>

## はじめに

<Steps>
  <Step title="Plugin をインストールする">
    ```bash
    openclaw plugins install @openclaw/venice-provider
    ```
  </Step>
  <Step title="API キーを取得する">
    1. [venice.ai](https://venice.ai) で登録する
    2. **Settings > API Keys > Create new key** に移動する
    3. API キーをコピーする（形式：`vapi_xxxxxxxxxxxx`）
  </Step>
  <Step title="OpenClaw を設定する">
    <Tabs>
      <Tab title="対話式（推奨）">
        ```bash
        openclaw onboard --auth-choice venice-api-key
        ```

        API キーの入力を求め（または既存の `VENICE_API_KEY` を再利用し）、利用可能な Venice モデルを一覧表示して、デフォルトモデルを設定します。
      </Tab>
      <Tab title="環境変数">
        ```bash
        export VENICE_API_KEY="vapi_xxxxxxxxxxxx"
        ```
      </Tab>
      <Tab title="非対話式">
        ```bash
        openclaw onboard --non-interactive \
          --auth-choice venice-api-key \
          --venice-api-key "vapi_xxxxxxxxxxxx"
        ```
      </Tab>
    </Tabs>

  </Step>
  <Step title="セットアップを確認する">
    ```bash
    openclaw agent --model venice/kimi-k2-5 --message "こんにちは、動作していますか？"
    ```
  </Step>
</Steps>

## モデルの選択

- **デフォルト**：`venice/kimi-k2-5`（プライベート、推論、ビジョン）。
- **最も高性能な匿名化オプション**：`venice/claude-opus-4-6`。

```bash
openclaw models set venice/kimi-k2-5
openclaw models list --all --provider venice
```

`openclaw configure` を実行し、**Model/auth provider > Venice AI** を選択することもできます。

<Tip>
| ユースケース              | モデル                                        | 理由                                    |
| --------------------- | -------------------------------------------- | -------------------------------------- |
| 一般的なチャット（デフォルト） | `kimi-k2-5`                                  | 高性能なプライベート推論とビジョン   |
| 総合的に最高の品質   | `claude-opus-4-6`                            | 最も高性能な Venice の匿名化オプション     |
| プライバシーとコーディング       | `qwen3-coder-480b-a35b-instruct-turbo`       | 大きなコンテキストを備えたプライベートなコーディングモデル |
| 高速かつ低コスト           | `llama-3.2-3b`                               | コンパクトなプライベートモデル                  |
| 複雑なプライベートタスク  | `deepseek-v3.2`                              | 高性能な推論。ツール呼び出しは無効 |
| 無検閲             | `venice-uncensored-1-2`                      | 現在の Venice 無検閲モデル        |
</Tip>

## 組み込みカタログ（30 モデル）

<AccordionGroup>
  <Accordion title="プライベートモデル（20）— 完全にプライベート、ログ記録なし">
    | モデル ID                               | 名前                                 | コンテキスト | 注記                      |
    | -------------------------------------- | ------------------------------------- | ------- | --------------------------- |
    | `kimi-k2-5`                            | Kimi K2.5                             | 256k    | デフォルト、推論、ビジョン  |
    | `llama-3.3-70b`                        | Llama 3.3 70B                         | 128k    | 汎用                     |
    | `llama-3.2-3b`                         | Llama 3.2 3B                          | 128k    | 汎用                     |
    | `hermes-3-llama-3.1-405b`              | Hermes 3 Llama 3.1 405B               | 128k    | 汎用、ツール無効     |
    | `qwen3-235b-a22b-thinking-2507`        | Qwen3 235B Thinking                   | 128k    | 推論                   |
    | `qwen3-235b-a22b-instruct-2507`        | Qwen3 235B Instruct                   | 128k    | 汎用                     |
    | `qwen3-coder-480b-a35b-instruct-turbo` | Qwen3 Coder 480B Turbo                | 256k    | コーディング                      |
    | `qwen3-5-35b-a3b`                      | Qwen3.5 35B A3B                       | 256k    | 推論、ビジョン           |
    | `qwen3-next-80b`                       | Qwen3 Next 80B                        | 256k    | 汎用                     |
    | `qwen3-vl-235b-a22b`                   | Qwen3 VL 235B（ビジョン）                | 256k    | ビジョン                      |
    | `deepseek-v3.2`                        | DeepSeek V3.2                         | 160k    | 推論、ツール無効    |
    | `google-gemma-3-27b-it`                | Google Gemma 3 27B Instruct           | 198k    | ビジョン                       |
    | `openai-gpt-oss-120b`                  | OpenAI GPT OSS 120B                   | 128k    | 汎用                      |
    | `nvidia-nemotron-3-nano-30b-a3b`       | NVIDIA Nemotron 3 Nano 30B            | 128k    | 汎用                      |
    | `olafangensan-glm-4.7-flash-heretic`   | GLM 4.7 Flash Heretic                 | 128k    | 推論                    |
    | `zai-org-glm-4.6`                      | GLM 4.6                               | 198k    | 汎用                      |
    | `zai-org-glm-4.7`                      | GLM 4.7                               | 198k    | 推論                    |
    | `zai-org-glm-4.7-flash`                | GLM 4.7 Flash                         | 128k    | 推論                    |
    | `zai-org-glm-5`                        | GLM 5                                 | 198k    | 推論                    |
    | `minimax-m25`                          | MiniMax M2.5                          | 198k    | 推論                    |
  </Accordion>

  <Accordion title="匿名化モデル（10）— Venice プロキシ経由">
    | モデル ID                        | 名前                           | コンテキスト | 注記                      |
    | -------------------------------- | -------------------------------- | ------- | ---------------------------- |
    | `claude-opus-4-6`               | Claude Opus 4.6（Venice 経由）    | 1M      | 推論、ビジョン            |
    | `claude-sonnet-4-6`             | Claude Sonnet 4.6（Venice 経由）  | 1M      | 推論、ビジョン            |
    | `openai-gpt-54`                 | GPT-5.4（Venice 経由）            | 1M      | 推論、ビジョン            |
    | `openai-gpt-53-codex`           | GPT-5.3 Codex（Venice 経由）      | 400k    | 推論、ビジョン、コーディング     |
    | `openai-gpt-52`                 | GPT-5.2（Venice 経由）            | 256k    | 推論                    |
    | `openai-gpt-52-codex`           | GPT-5.2 Codex（Venice 経由）      | 256k    | 推論、ビジョン、コーディング     |
    | `openai-gpt-4o-2024-11-20`      | GPT-4o（Venice 経由）             | 128k    | ビジョン                        |
    | `openai-gpt-4o-mini-2024-07-18` | GPT-4o Mini（Venice 経由）        | 128k    | ビジョン                        |
    | `gemini-3-1-pro-preview`        | Gemini 3.1 Pro（Venice 経由）     | 1M      | 推論、ビジョン             |
    | `gemini-3-flash-preview`        | Gemini 3 Flash（Venice 経由）     | 256k    | 推論、ビジョン             |
  </Accordion>
</AccordionGroup>

Grok を基盤とする Venice モデル（`grok-4-3` など）には、ネイティブ xAI プロバイダーと同じツールスキーマ
互換パッチが適用されます。これは、両者が同じアップストリームの
ツール呼び出し形式を共有しているためです。

## モデルの検出

上記の組み込みカタログは、マニフェストを基にしたシードリストです。実行時に OpenClaw は
Venice の `/models` API からこれを更新し、API に到達できない場合は
シードリストにフォールバックします。`/models` エンドポイントは公開されており（一覧表示に認証は不要）、
推論には有効な API キーが必要です。

Venice は、廃止されたモデル ID をプロバイダー所有のエイリアスとして引き続き受け付ける場合があります。
OpenClaw カタログでは、`/models` が返す正規モデル ID のみを掲載します。

## DeepSeek V4 のリプレイ動作

Venice が `deepseek-v4-pro` や
`deepseek-v4-flash` などの DeepSeek V4 モデルを公開している場合、Venice が省略したときに、OpenClaw はアシスタントメッセージの必須
`reasoning_content` リプレイフィールドを補完し、リクエストペイロードから `thinking`/
`reasoning`/`reasoning_effort` を除去します（Venice はこれらのモデルで
DeepSeek ネイティブの `thinking` 制御を拒否します）。このリプレイ修正は、
ネイティブ DeepSeek プロバイダー独自の思考制御とは別のものです。

## ストリーミングとツールのサポート

| 機能          | サポート                                           |
| ---------------- | ------------------------------------------------- |
| ストリーミング        | すべてのモデル                                        |
| 関数呼び出し | ほとんどのモデル。上記に記載のあるモデルでは個別に無効 |
| ビジョン／画像    | 上記で「ビジョン」と記載されたモデル                      |
| JSON モード        | `response_format` 経由                             |

## 料金

Venice はクレジットベースのシステムを使用します。匿名化モデルの料金は、おおむね
API の直接利用料金に少額の Venice 手数料を加えたものです。現在の料金については、
[venice.ai/pricing](https://venice.ai/pricing) を参照してください。

## 使用例

```bash
# デフォルトのプライベートモデル
openclaw agent --model venice/kimi-k2-5 --message "簡単なヘルスチェック"

# Venice 経由の Claude Opus（匿名化）
openclaw agent --model venice/claude-opus-4-6 --message "このタスクを要約してください"

# 無検閲モデル
openclaw agent --model venice/venice-uncensored-1-2 --message "選択肢を作成してください"

# 画像を使用するビジョンモデル
openclaw agent --model venice/qwen3-vl-235b-a22b --message "添付画像を確認してください"

# コーディングモデル
openclaw agent --model venice/qwen3-coder-480b-a35b-instruct-turbo --message "この関数をリファクタリングしてください"
```

## トラブルシューティング

<AccordionGroup>
  <Accordion title="API キーが認識されない">
    ```bash
    echo $VENICE_API_KEY
    openclaw models list | grep venice
    ```

    キーが `vapi_` で始まることを確認してください。

  </Accordion>

  <Accordion title="モデルが利用できない">
    現在利用可能なモデルを確認するには `openclaw models list --all --provider venice` を実行してください。
    Venice がモデルを追加または廃止すると、カタログも変更されます。
  </Accordion>

  <Accordion title="接続の問題">
    Venice API は `https://api.venice.ai/api/v1` にあります。ネットワークからそのホストへの HTTPS 接続が許可されていることを確認してください。
  </Accordion>
</AccordionGroup>

<Note>
詳しいヘルプ：[トラブルシューティング](/ja-JP/help/troubleshooting)と[よくある質問](/ja-JP/help/faq)。
</Note>

## 高度な設定

<AccordionGroup>
  <Accordion title="設定ファイルの例">
    ```json5
    {
      env: { VENICE_API_KEY: "vapi_..." },
      agents: { defaults: { model: { primary: "venice/kimi-k2-5" } } },
      models: {
        mode: "merge",
        providers: {
          venice: {
            baseUrl: "https://api.venice.ai/api/v1",
            apiKey: "${VENICE_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2-5",
                name: "Kimi K2.5",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 256000,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## 関連項目

<CardGroup cols={2}>
  <Card title="モデルの選択" href="/ja-JP/concepts/model-providers" icon="layers">
    プロバイダー、モデル参照、フェイルオーバー動作の選択。
  </Card>
  <Card title="Venice AI" href="https://venice.ai" icon="globe">
    Venice AI のホームページとアカウント登録。
  </Card>
  <Card title="API ドキュメント" href="https://docs.venice.ai" icon="book">
    Venice API リファレンスと開発者向けドキュメント。
  </Card>
  <Card title="料金" href="https://venice.ai/pricing" icon="credit-card">
    現在の Venice クレジット料金とプラン。
  </Card>
</CardGroup>
