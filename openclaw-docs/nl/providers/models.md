---
read_when:
    - Je wilt een modelprovider kiezen
    - Je wilt snelle installatievoorbeelden voor LLM-authenticatie en modelselectie
summary: Door OpenClaw ondersteunde modelproviders (LLM's)
title: Snelstart voor modelproviders
x-i18n:
    generated_at: "2026-07-27T06:09:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3988d6985cbe203a6a3357d59160190990b1b53245ea25f1538dbc6f567afec1
    source_path: providers/models.md
    workflow: 16
---

Kies een provider, verifieer je identiteit en stel vervolgens het standaardmodel in als `provider/model`.

## Snel starten (twee stappen)

1. Verifieer je identiteit bij de provider (meestal via `openclaw onboard`).
2. Stel het standaardmodel in:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Ondersteunde providers (startset)

- [Alibaba Model Studio](/nl/providers/alibaba)
- [Amazon Bedrock](/nl/providers/bedrock)
- [Anthropic (API + Claude CLI)](/nl/providers/anthropic)
- [Baseten (Inkling + model-API's)](/providers/baseten)
- [BytePlus (internationaal)](/nl/concepts/model-providers#byteplus-international)
- [Chutes](/nl/providers/chutes)
- [Cloudflare AI Gateway](/nl/providers/cloudflare-ai-gateway)
- [Cohere](/nl/providers/cohere)
- [ComfyUI](/nl/providers/comfy)
- [DeepInfra](/nl/providers/deepinfra)
- [fal](/nl/providers/fal)
- [Fireworks](/nl/providers/fireworks)
- [MiniMax](/nl/providers/minimax)
- [Mistral](/nl/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/nl/providers/moonshot)
- [NovitaAI](/nl/providers/novita)
- [OpenAI (API + Codex)](/nl/providers/openai)
- [OpenCode (Zen + Go)](/nl/providers/opencode)
- [OpenRouter](/nl/providers/openrouter)
- [Qianfan](/nl/providers/qianfan)
- [Qwen](/nl/providers/qwen)
- [Runway](/nl/providers/runway)
- [StepFun](/nl/providers/stepfun)
- [Synthetic](/nl/providers/synthetic)
- [Venice (Venice AI)](/nl/providers/venice)
- [Vercel AI Gateway](/nl/providers/vercel-ai-gateway)
- [xAI](/nl/providers/xai)
- [Z.AI (GLM)](/nl/providers/zai)

Zie voor de volledige providercatalogus en geavanceerde configuratie
[Provideroverzicht](/nl/providers/index) en [Modelproviders](/nl/concepts/model-providers).

## Aanvullende providervarianten

- `anthropic-vertex` - installeer `@openclaw/anthropic-vertex-provider` voor impliciete ondersteuning van Anthropic op Google Vertex wanneer Vertex-referenties beschikbaar zijn; geen afzonderlijke authenticatiekeuze tijdens de onboarding
- `copilot-proxy` - lokale VS Code Copilot Proxy-bridge; gebruik `openclaw onboard --auth-choice copilot-proxy`
- `google-gemini-cli` - niet-officiële OAuth-flow voor Gemini CLI; vereist een lokale installatie van `gemini` (`brew install gemini-cli` of `npm install -g @google/gemini-cli`); standaardmodel `google-gemini-cli/gemini-3-flash-preview`; gebruik `openclaw onboard --auth-choice google-gemini-cli` of `openclaw models auth login --provider google-gemini-cli --set-default`

## Gerelateerd

- [Provideroverzicht](/nl/providers/index)
- [Modelselectie](/nl/concepts/model-providers)
- [Model-failover](/nl/concepts/model-failover)
- [Modellen-CLI](/nl/cli/models)
