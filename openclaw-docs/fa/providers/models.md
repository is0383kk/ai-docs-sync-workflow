---
read_when:
    - می‌خواهید یک ارائه‌دهندهٔ مدل انتخاب کنید
    - نمونه‌های راه‌اندازی سریع برای احراز هویت LLM و انتخاب مدل می‌خواهید
summary: ارائه‌دهندگان مدل (LLMها) که OpenClaw از آن‌ها پشتیبانی می‌کند
title: شروع سریع ارائه‌دهنده مدل
x-i18n:
    generated_at: "2026-07-27T17:03:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3988d6985cbe203a6a3357d59160190990b1b53245ea25f1538dbc6f567afec1
    source_path: providers/models.md
    workflow: 16
---

یک ارائه‌دهنده انتخاب و احراز هویت کنید، سپس مدل پیش‌فرض را روی `provider/model` تنظیم کنید.

## شروع سریع (دو مرحله)

1. با ارائه‌دهنده احراز هویت کنید (معمولاً از طریق `openclaw onboard`).
2. مدل پیش‌فرض را تنظیم کنید:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## ارائه‌دهندگان پشتیبانی‌شده (مجموعه آغازین)

- [Alibaba Model Studio](/fa/providers/alibaba)
- [Amazon Bedrock](/fa/providers/bedrock)
- [Anthropic (API + Claude CLI)](/fa/providers/anthropic)
- [Baseten (Inkling + APIهای مدل)](/providers/baseten)
- [BytePlus (بین‌المللی)](/fa/concepts/model-providers#byteplus-international)
- [Chutes](/fa/providers/chutes)
- [Cloudflare AI Gateway](/fa/providers/cloudflare-ai-gateway)
- [Cohere](/fa/providers/cohere)
- [ComfyUI](/fa/providers/comfy)
- [DeepInfra](/fa/providers/deepinfra)
- [fal](/fa/providers/fal)
- [Fireworks](/fa/providers/fireworks)
- [MiniMax](/fa/providers/minimax)
- [Mistral](/fa/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/fa/providers/moonshot)
- [NovitaAI](/fa/providers/novita)
- [OpenAI (API + Codex)](/fa/providers/openai)
- [OpenCode (Zen + Go)](/fa/providers/opencode)
- [OpenRouter](/fa/providers/openrouter)
- [Qianfan](/fa/providers/qianfan)
- [Qwen](/fa/providers/qwen)
- [Runway](/fa/providers/runway)
- [StepFun](/fa/providers/stepfun)
- [Synthetic](/fa/providers/synthetic)
- [Venice (Venice AI)](/fa/providers/venice)
- [Vercel AI Gateway](/fa/providers/vercel-ai-gateway)
- [xAI](/fa/providers/xai)
- [Z.AI (GLM)](/fa/providers/zai)

برای فهرست کامل ارائه‌دهندگان و پیکربندی پیشرفته، به
[فهرست ارائه‌دهندگان](/fa/providers/index) و [ارائه‌دهندگان مدل](/fa/concepts/model-providers) مراجعه کنید.

## گونه‌های دیگر ارائه‌دهندگان

- `anthropic-vertex` - برای پشتیبانی ضمنی از Anthropic در Google Vertex، هنگامی که اعتبارنامه‌های Vertex در دسترس‌اند، `@openclaw/anthropic-vertex-provider` را نصب کنید؛ گزینه احراز هویت جداگانه‌ای در راه‌اندازی اولیه وجود ندارد
- `copilot-proxy` - پل محلی VS Code Copilot Proxy؛ از `openclaw onboard --auth-choice copilot-proxy` استفاده کنید
- `google-gemini-cli` - جریان غیررسمی OAuth در Gemini CLI؛ به نصب محلی `gemini` نیاز دارد (`brew install gemini-cli` یا `npm install -g @google/gemini-cli`)؛ مدل پیش‌فرض `google-gemini-cli/gemini-3-flash-preview`؛ از `openclaw onboard --auth-choice google-gemini-cli` یا `openclaw models auth login --provider google-gemini-cli --set-default` استفاده کنید

## مرتبط

- [فهرست ارائه‌دهندگان](/fa/providers/index)
- [انتخاب مدل](/fa/concepts/model-providers)
- [جابه‌جایی خودکار مدل هنگام خرابی](/fa/concepts/model-failover)
- [CLI مدل‌ها](/fa/cli/models)
