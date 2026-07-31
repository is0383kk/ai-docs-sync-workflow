---
read_when:
    - Bir model sağlayıcısı seçmek istiyorsunuz
    - LLM kimlik doğrulaması + model seçimi için hızlı kurulum örnekleri istiyorsunuz
summary: OpenClaw tarafından desteklenen model sağlayıcıları (LLM'ler)
title: Model sağlayıcısı hızlı başlangıç kılavuzu
x-i18n:
    generated_at: "2026-07-27T00:14:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3988d6985cbe203a6a3357d59160190990b1b53245ea25f1538dbc6f567afec1
    source_path: providers/models.md
    workflow: 16
---

Bir sağlayıcı seçin, kimlik doğrulaması yapın ve ardından varsayılan modeli `provider/model` olarak ayarlayın.

## Hızlı başlangıç (iki adım)

1. Sağlayıcıyla kimlik doğrulaması yapın (genellikle `openclaw onboard` aracılığıyla).
2. Varsayılan modeli ayarlayın:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Desteklenen sağlayıcılar (başlangıç kümesi)

- [Alibaba Model Studio](/tr/providers/alibaba)
- [Amazon Bedrock](/tr/providers/bedrock)
- [Anthropic (API + Claude CLI)](/tr/providers/anthropic)
- [Baseten (Inkling + Model API'leri)](/providers/baseten)
- [BytePlus (Uluslararası)](/tr/concepts/model-providers#byteplus-international)
- [Chutes](/tr/providers/chutes)
- [Cloudflare AI Gateway](/tr/providers/cloudflare-ai-gateway)
- [Cohere](/tr/providers/cohere)
- [ComfyUI](/tr/providers/comfy)
- [DeepInfra](/tr/providers/deepinfra)
- [fal](/tr/providers/fal)
- [Fireworks](/tr/providers/fireworks)
- [MiniMax](/tr/providers/minimax)
- [Mistral](/tr/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/tr/providers/moonshot)
- [NovitaAI](/tr/providers/novita)
- [OpenAI (API + Codex)](/tr/providers/openai)
- [OpenCode (Zen + Go)](/tr/providers/opencode)
- [OpenRouter](/tr/providers/openrouter)
- [Qianfan](/tr/providers/qianfan)
- [Qwen](/tr/providers/qwen)
- [Runway](/tr/providers/runway)
- [StepFun](/tr/providers/stepfun)
- [Synthetic](/tr/providers/synthetic)
- [Venice (Venice AI)](/tr/providers/venice)
- [Vercel AI Gateway](/tr/providers/vercel-ai-gateway)
- [xAI](/tr/providers/xai)
- [Z.AI (GLM)](/tr/providers/zai)

Sağlayıcıların tam katalogu ve gelişmiş yapılandırma için
[Sağlayıcı dizini](/tr/providers/index) ve [Model sağlayıcıları](/tr/concepts/model-providers) bölümlerine bakın.

## Ek sağlayıcı çeşitleri

- `anthropic-vertex` - Vertex kimlik bilgileri kullanılabilir olduğunda Google Vertex üzerinde örtük Anthropic desteği için `@openclaw/anthropic-vertex-provider` paketini yükleyin; ayrı bir ilk kurulum kimlik doğrulama seçeneği yoktur
- `copilot-proxy` - yerel VS Code Copilot Proxy köprüsü; `openclaw onboard --auth-choice copilot-proxy` kullanın
- `google-gemini-cli` - resmî olmayan Gemini CLI OAuth akışı; yerel bir `gemini` kurulumu gerektirir (`brew install gemini-cli` veya `npm install -g @google/gemini-cli`); varsayılan model `google-gemini-cli/gemini-3-flash-preview`; `openclaw onboard --auth-choice google-gemini-cli` veya `openclaw models auth login --provider google-gemini-cli --set-default` kullanın

## İlgili

- [Sağlayıcı dizini](/tr/providers/index)
- [Model seçimi](/tr/concepts/model-providers)
- [Model yük devretmesi](/tr/concepts/model-failover)
- [Models CLI](/tr/cli/models)
