---
read_when:
    - モデルプロバイダーを選択する場合
    - LLM 認証とモデル選択のクイックセットアップ例が必要な場合
summary: OpenClaw がサポートするモデルプロバイダー（LLM）
title: モデルプロバイダーのクイックスタート
x-i18n:
    generated_at: "2026-07-26T10:17:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3988d6985cbe203a6a3357d59160190990b1b53245ea25f1538dbc6f567afec1
    source_path: providers/models.md
    workflow: 16
---

プロバイダーを選択して認証し、デフォルトモデルを `provider/model` に設定します。

## クイックスタート（2 ステップ）

1. プロバイダーで認証します（通常は `openclaw onboard` を使用）。
2. デフォルトモデルを設定します。

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## 対応プロバイダー（初期セット）

- [Alibaba Model Studio](/ja-JP/providers/alibaba)
- [Amazon Bedrock](/ja-JP/providers/bedrock)
- [Anthropic（API + Claude CLI）](/ja-JP/providers/anthropic)
- [Baseten（Inkling + Model API）](/providers/baseten)
- [BytePlus（国際版）](/ja-JP/concepts/model-providers#byteplus-international)
- [Chutes](/ja-JP/providers/chutes)
- [Cloudflare AI Gateway](/ja-JP/providers/cloudflare-ai-gateway)
- [Cohere](/ja-JP/providers/cohere)
- [ComfyUI](/ja-JP/providers/comfy)
- [DeepInfra](/ja-JP/providers/deepinfra)
- [fal](/ja-JP/providers/fal)
- [Fireworks](/ja-JP/providers/fireworks)
- [MiniMax](/ja-JP/providers/minimax)
- [Mistral](/ja-JP/providers/mistral)
- [Moonshot AI（Kimi + Kimi Coding）](/ja-JP/providers/moonshot)
- [NovitaAI](/ja-JP/providers/novita)
- [OpenAI（API + Codex）](/ja-JP/providers/openai)
- [OpenCode（Zen + Go）](/ja-JP/providers/opencode)
- [OpenRouter](/ja-JP/providers/openrouter)
- [Qianfan](/ja-JP/providers/qianfan)
- [Qwen](/ja-JP/providers/qwen)
- [Runway](/ja-JP/providers/runway)
- [StepFun](/ja-JP/providers/stepfun)
- [Synthetic](/ja-JP/providers/synthetic)
- [Venice（Venice AI）](/ja-JP/providers/venice)
- [Vercel AI Gateway](/ja-JP/providers/vercel-ai-gateway)
- [xAI](/ja-JP/providers/xai)
- [Z.AI（GLM）](/ja-JP/providers/zai)

プロバイダーの完全なカタログと高度な設定については、
[プロバイダーディレクトリ](/ja-JP/providers/index)および[モデルプロバイダー](/ja-JP/concepts/model-providers)を参照してください。

## その他のプロバイダーバリエーション

- `anthropic-vertex` - Vertex の認証情報が利用可能な場合に、Google Vertex 上の Anthropic を暗黙的にサポートするには `@openclaw/anthropic-vertex-provider` をインストールします。オンボーディングで個別に認証を選択する必要はありません
- `copilot-proxy` - ローカルの VS Code Copilot Proxy ブリッジです。`openclaw onboard --auth-choice copilot-proxy` を使用します
- `google-gemini-cli` - 非公式の Gemini CLI OAuth フローです。ローカルに `gemini` をインストールする必要があります（`brew install gemini-cli` または `npm install -g @google/gemini-cli`）。デフォルトモデルは `google-gemini-cli/gemini-3-flash-preview` です。`openclaw onboard --auth-choice google-gemini-cli` または `openclaw models auth login --provider google-gemini-cli --set-default` を使用します

## 関連項目

- [プロバイダーディレクトリ](/ja-JP/providers/index)
- [モデルの選択](/ja-JP/concepts/model-providers)
- [モデルのフェイルオーバー](/ja-JP/concepts/model-failover)
- [モデル CLI](/ja-JP/cli/models)
