---
read_when:
    - スクリプトまたは CI でオンボーディングを自動化する場合
    - 特定のプロバイダー向けの非対話型の例が必要です
sidebarTitle: CLI automation
summary: OpenClaw CLI のスクリプトによるオンボーディングとエージェント設定
title: CLI 自動化
x-i18n:
    generated_at: "2026-07-26T09:53:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2a9fd8530379927995641f8033651ff12ada98068f106672e6655a17b8265735
    source_path: start/wizard-cli-automation.md
    workflow: 16
---

`openclaw onboard --non-interactive` を使用してセットアップをスクリプト化します。これには `--accept-risk` が必要です。非対話型セットアップでは、確認プロンプトなしで認証情報とデーモン設定を書き込めるため、このフラグによってリスクを明示的に承認します。

<Note>
`--json` は非対話型モードを意味しません。スクリプトでは `--non-interactive --accept-risk` を明示的に渡してください。
</Note>

## 非対話型の基本例

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice apiKey \
  --anthropic-api-key "$ANTHROPIC_API_KEY" \
  --secret-input-mode plaintext \
  --gateway-bind loopback \
  --install-daemon \
  --daemon-runtime node \
  --skip-bootstrap \
  --skip-skills
```

機械可読なサマリーを出力するには、`--json` を追加します。

- `--gateway-port` のデフォルトは `18789` です。上書きする場合にのみ渡してください。
- 独自のワークスペースを事前に用意する自動化では、`--skip-bootstrap` を指定するとデフォルトのワークスペースファイルの作成をスキップできます。
- `--secret-input-mode ref` は、平文キーの代わりに環境変数を参照する値（`{ source: "env", provider: "default", id: "<ENV_VAR>" }`）を認証プロファイルに保存します。非対話型の `ref` モードでは、プロバイダーの環境変数がプロセス環境にすでに設定されている必要があります。対応する環境変数なしでインラインキーフラグを渡すと、即座に失敗します。

```bash
openclaw onboard --non-interactive --accept-risk \
  --mode local \
  --auth-choice openai-api-key \
  --secret-input-mode ref
```

## プロバイダー別の例

<AccordionGroup>
  <Accordion title="Anthropic API キーの例">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice apiKey \
      --anthropic-api-key "$ANTHROPIC_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Cloudflare AI Gateway の例">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice cloudflare-ai-gateway-api-key \
      --cloudflare-ai-gateway-account-id "your-account-id" \
      --cloudflare-ai-gateway-gateway-id "your-gateway-id" \
      --cloudflare-ai-gateway-api-key "$CLOUDFLARE_AI_GATEWAY_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Gemini の例">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice gemini-api-key \
      --gemini-api-key "$GEMINI_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Mistral の例">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice mistral-api-key \
      --mistral-api-key "$MISTRAL_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Moonshot の例">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice moonshot-api-key \
      --moonshot-api-key "$MOONSHOT_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Ollama の例">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice ollama \
      --custom-model-id "qwen3.5:27b" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="OpenCode の例">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice opencode-zen \
      --opencode-zen-api-key "$OPENCODE_API_KEY" \
      --gateway-bind loopback
    ```
    Go カタログを使用するには `--auth-choice opencode-go --opencode-go-api-key "$OPENCODE_API_KEY"` に変更します。
  </Accordion>
  <Accordion title="Synthetic の例">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice synthetic-api-key \
      --synthetic-api-key "$SYNTHETIC_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Vercel AI Gateway の例">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice ai-gateway-api-key \
      --ai-gateway-api-key "$AI_GATEWAY_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="Z.AI の例">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice zai-api-key \
      --zai-api-key "$ZAI_API_KEY" \
      --gateway-bind loopback
    ```
  </Accordion>
  <Accordion title="カスタムプロバイダーの例">
    ```bash
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice custom-api-key \
      --custom-base-url "https://llm.example.com/v1" \
      --custom-model-id "foo-large" \
      --custom-api-key "$CUSTOM_API_KEY" \
      --custom-provider-id "my-custom" \
      --custom-compatibility anthropic \
      --custom-image-input \
      --gateway-bind loopback
    ```

    `--custom-api-key` は任意です。一部のエンドポイントでは認証が不要です。省略した場合、オンボーディングは環境変数の `CUSTOM_API_KEY` を確認します。`--custom-provider-id` も任意で、省略するとベース URL から自動的に導出されます。`--custom-compatibility` のデフォルトは `openai` です（その他の値：`openai-responses`、`anthropic`）。

    OpenClaw は、既知のビジョンモデル ID パターン（`gpt-4o`、`claude-3/4`、`gemini`、`-vl`/`vision` サフィックスなど）から画像入力のサポートを推測します。認識されないビジョンモデルで強制的に有効にするには `--custom-image-input` を、テキストのみに強制するには `--custom-text-input` を追加します。

    `apiKey` を `{ source: "env", provider: "default", id: "CUSTOM_API_KEY" }` として保存する参照モードの例：

    ```bash
    export CUSTOM_API_KEY="your-key"
    openclaw onboard --non-interactive --accept-risk \
      --mode local \
      --auth-choice custom-api-key \
      --custom-base-url "https://llm.example.com/v1" \
      --custom-model-id "foo-large" \
      --secret-input-mode ref \
      --custom-provider-id "my-custom" \
      --custom-compatibility anthropic \
      --custom-image-input \
      --gateway-bind loopback
    ```

  </Accordion>
</AccordionGroup>

Anthropic のセットアップトークン認証は引き続きサポートされていますが、ローカルの Claude CLI ログインが利用可能な場合、OpenClaw は Claude CLI の再利用を優先します。本番環境では、Anthropic API キーを使用してください。

## 別のエージェントを追加する

`openclaw agents add <name>` は、独自のワークスペース、セッション、認証プロファイルを持つ個別のエージェントを作成します。`--workspace` もその他のフラグも指定せずに実行すると、対話型ウィザードが起動します。`--workspace`、`--model`、`--agent-dir`、`--bind`、`--non-interactive` のいずれかを渡すと非対話型で実行され、その場合は `--workspace` も必要です。

```bash
openclaw agents add work \
  --workspace ~/.openclaw/workspace-work \
  --model openai/gpt-5.6-sol \
  --bind whatsapp:biz \
  --non-interactive \
  --json
```

書き込まれる設定キー（新しいエージェント ID の `agents.entries.*` エントリ）：

- `name`
- `workspace`
- `agentDir`
- `model`（`--model` を渡した場合のみ）

注記：

- デフォルトのワークスペース（対話型ウィザードで `--workspace` を省略した場合）：`~/.openclaw/workspace-<agentId>`。
- `--bind <channel[:accountId]>` は繰り返し指定できます。受信メッセージを新しいエージェントにルーティングするためのバインディングを追加します（ウィザードでも対話形式で設定できます）。
- エージェント名は有効なエージェント ID に正規化されます。`main` は予約済みです。

## 関連ドキュメント

- オンボーディングのハブ：[オンボーディング（CLI）](/ja-JP/start/wizard)
- 完全なリファレンス：[CLI セットアップリファレンス](/ja-JP/start/wizard-cli-reference)
- コマンドリファレンス：[`openclaw onboard`](/ja-JP/cli/onboard)
