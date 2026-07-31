---
read_when:
    - Quiere elegir un proveedor de modelos
    - Se buscan ejemplos de configuración rápida para la autenticación de LLM y la selección de modelos
summary: Proveedores de modelos (LLM) compatibles con OpenClaw
title: Inicio rápido del proveedor de modelos
x-i18n:
    generated_at: "2026-07-26T05:27:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3988d6985cbe203a6a3357d59160190990b1b53245ea25f1538dbc6f567afec1
    source_path: providers/models.md
    workflow: 16
---

Elija un proveedor, autentíquese y, a continuación, establezca el modelo predeterminado como `provider/model`.

## Inicio rápido (dos pasos)

1. Autentíquese con el proveedor (normalmente mediante `openclaw onboard`).
2. Establezca el modelo predeterminado:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Proveedores compatibles (conjunto inicial)

- [Alibaba Model Studio](/es/providers/alibaba)
- [Amazon Bedrock](/es/providers/bedrock)
- [Anthropic (API + CLI de Claude)](/es/providers/anthropic)
- [Baseten (Inkling + API de modelos)](/es/providers/baseten)
- [BytePlus (internacional)](/es/concepts/model-providers#byteplus-international)
- [Chutes](/es/providers/chutes)
- [Cloudflare AI Gateway](/es/providers/cloudflare-ai-gateway)
- [Cohere](/es/providers/cohere)
- [ComfyUI](/es/providers/comfy)
- [DeepInfra](/es/providers/deepinfra)
- [fal](/es/providers/fal)
- [Fireworks](/es/providers/fireworks)
- [MiniMax](/es/providers/minimax)
- [Mistral](/es/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/es/providers/moonshot)
- [NovitaAI](/es/providers/novita)
- [OpenAI (API + Codex)](/es/providers/openai)
- [OpenCode (Zen + Go)](/es/providers/opencode)
- [OpenRouter](/es/providers/openrouter)
- [Qianfan](/es/providers/qianfan)
- [Qwen](/es/providers/qwen)
- [Runway](/es/providers/runway)
- [StepFun](/es/providers/stepfun)
- [Synthetic](/es/providers/synthetic)
- [Venice (Venice AI)](/es/providers/venice)
- [Vercel AI Gateway](/es/providers/vercel-ai-gateway)
- [xAI](/es/providers/xai)
- [Z.AI (GLM)](/es/providers/zai)

Para consultar el catálogo completo de proveedores y la configuración avanzada, véanse
[Directorio de proveedores](/es/providers/index) y [Proveedores de modelos](/es/concepts/model-providers).

## Variantes adicionales de proveedores

- `anthropic-vertex` - instale `@openclaw/anthropic-vertex-provider` para habilitar la compatibilidad implícita con Anthropic en Google Vertex cuando haya credenciales de Vertex disponibles; no hay una opción de autenticación independiente durante la incorporación
- `copilot-proxy` - puente de proxy local de Copilot para VS Code; use `openclaw onboard --auth-choice copilot-proxy`
- `google-gemini-cli` - flujo OAuth no oficial de la CLI de Gemini; requiere una instalación local de `gemini` (`brew install gemini-cli` o `npm install -g @google/gemini-cli`); modelo predeterminado `google-gemini-cli/gemini-3-flash-preview`; use `openclaw onboard --auth-choice google-gemini-cli` o `openclaw models auth login --provider google-gemini-cli --set-default`

## Contenido relacionado

- [Directorio de proveedores](/es/providers/index)
- [Selección de modelos](/es/concepts/model-providers)
- [Conmutación por error de modelos](/es/concepts/model-failover)
- [CLI de modelos](/es/cli/models)
