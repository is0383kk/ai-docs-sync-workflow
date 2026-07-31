---
read_when:
    - می‌خواهید یک ارائه‌دهندهٔ مدل انتخاب کنید
    - به یک مرور سریع از بک‌اندهای LLM پشتیبانی‌شده نیاز دارید
summary: ارائه‌دهندگان مدل (LLMها) که OpenClaw از آن‌ها پشتیبانی می‌کند
title: فهرست ارائه‌دهندگان
x-i18n:
    generated_at: "2026-07-27T15:51:34Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e98910f016e461dedcd06e40a2933631bbd6ac09ceebd340bab82f14805e06a6
    source_path: providers/index.md
    workflow: 16
---

OpenClaw می‌تواند از ارائه‌دهندگان متعدد LLM استفاده کند. یک ارائه‌دهنده را انتخاب و احراز هویت کنید، سپس
مدل پیش‌فرض را روی `provider/model` تنظیم کنید.

به‌دنبال مستندات کانال‌های چت (WhatsApp/Telegram/Discord/Slack/Mattermost (Plugin)/و غیره) هستید؟ به [کانال‌ها](/fa/channels) مراجعه کنید.

## شروع سریع

1. با ارائه‌دهنده احراز هویت کنید (معمولاً از طریق `openclaw onboard`).
2. مدل پیش‌فرض را تنظیم کنید:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## مستندات ارائه‌دهندگان

- [Alibaba Model Studio](/fa/providers/alibaba)
- [Amazon Bedrock](/fa/providers/bedrock)
- [Amazon Bedrock Mantle](/fa/providers/bedrock-mantle)
- [Anthropic (API + Claude CLI)](/fa/providers/anthropic)
- [Arcee AI (مدل‌های Trinity)](/fa/providers/arcee)
- [Azure Speech](/fa/providers/azure-speech)
- [Baseten (‏Inkling + ‏APIهای مدل)](/providers/baseten)
- [BytePlus (بین‌المللی)](/fa/concepts/model-providers#byteplus-international)
- [Cerebras](/fa/providers/cerebras)
- [Chutes](/fa/providers/chutes)
- [ClawRouter (مسیریابی مدیریت‌شده میان چند ارائه‌دهنده)](/fa/providers/clawrouter)
- [Cloudflare AI Gateway](/fa/providers/cloudflare-ai-gateway)
- [Cohere](/fa/providers/cohere)
- [ComfyUI](/fa/providers/comfy)
- [DeepSeek](/fa/providers/deepseek)
- [ds4 (‏DeepSeek V4 محلی)](/fa/providers/ds4)
- [ElevenLabs](/fa/providers/elevenlabs)
- [fal](/fa/providers/fal)
- [Featherless AI](/fa/providers/featherless)
- [Fireworks](/fa/providers/fireworks)
- [GitHub Copilot](/fa/providers/github-copilot)
- [GMI Cloud](/fa/providers/gmi)
- [Google (Gemini)](/fa/providers/google)
- [Gradium](/fa/providers/gradium)
- [Groq (استنتاج LPU)](/fa/providers/groq)
- [Hugging Face (استنتاج)](/fa/providers/huggingface)
- [inferrs (مدل‌های محلی)](/fa/providers/inferrs)
- [Kilocode](/fa/providers/kilocode)
- [LiteLLM (Gateway یکپارچه)](/fa/providers/litellm)
- [LM Studio (مدل‌های محلی)](/fa/providers/lmstudio)
- [LongCat](/fa/providers/longcat)
- [MiniMax](/fa/providers/minimax)
- [Mistral](/fa/providers/mistral)
- [Moonshot AI (‏Kimi + ‏Kimi Coding)](/fa/providers/moonshot)
- [NovitaAI](/fa/providers/novita)
- [NVIDIA](/fa/providers/nvidia)
- [Ollama (مدل‌های ابری + محلی)](/fa/providers/ollama)
- [Ollama Cloud](/fa/providers/ollama-cloud)
- [OpenAI (‏API + ‏Codex)](/fa/providers/openai)
- [OpenCode](/fa/providers/opencode)
- [OpenCode Go](/fa/providers/opencode-go)
- [OpenRouter](/fa/providers/openrouter)
- [Perplexity (جست‌وجوی وب)](/fa/providers/perplexity-provider)
- [Qianfan](/fa/providers/qianfan)
- [Qwen Cloud](/fa/providers/qwen)
- [Runway](/fa/providers/runway)
- [SenseAudio](/fa/providers/senseaudio)
- [SGLang (مدل‌های محلی)](/fa/providers/sglang)
- [StepFun](/fa/providers/stepfun)
- [Synthetic](/fa/providers/synthetic)
- [Tencent Cloud (‏TokenHub / ‏TokenPlan)](/fa/providers/tencent)
- [Together AI](/fa/providers/together)
- [Venice (‏Venice AI، متمرکز بر حریم خصوصی)](/fa/providers/venice)
- [Vercel AI Gateway](/fa/providers/vercel-ai-gateway)
- [vLLM (مدل‌های محلی)](/fa/providers/vllm)
- [Volcengine (Doubao)](/fa/providers/volcengine)
- [Vydra](/fa/providers/vydra)
- [xAI](/fa/providers/xai)
- [Xiaomi](/fa/providers/xiaomi)
- [Z.AI (GLM)](/fa/providers/zai)

## صفحه‌های نمای کلی مشترک

- [گونه‌های بیشتر ارائه‌دهندگان](/fa/providers/models#additional-provider-variants) - ‏Anthropic Vertex، ‏Copilot Proxy و Gemini CLI OAuth
- [تولید تصویر](/fa/tools/image-generation) - ابزار مشترک `image_generate`، انتخاب ارائه‌دهنده و انتقال خودکار هنگام خرابی
- [تولید موسیقی](/fa/tools/music-generation) - ابزار مشترک `music_generate`، انتخاب ارائه‌دهنده و انتقال خودکار هنگام خرابی
- [تولید ویدئو](/fa/tools/video-generation) - ابزار مشترک `video_generate`، انتخاب ارائه‌دهنده و انتقال خودکار هنگام خرابی

## ارائه‌دهندگان رونویسی

- [Deepgram (رونویسی صوتی)](/fa/providers/deepgram)
- [ElevenLabs](/fa/providers/elevenlabs#speech-to-text)
- [Mistral](/fa/providers/mistral#audio-transcription-voxtral)
- [OpenAI](/fa/providers/openai)
- [SenseAudio](/fa/providers/senseaudio)
- [xAI](/fa/providers/xai)

## ابزارهای جامعه

- [Claude Max API Proxy](/fa/providers/claude-max-api-proxy) - پراکسی جامعه برای اطلاعات اصالت‌سنجی اشتراک Claude (پیش از استفاده، خط‌مشی/شرایط Anthropic را بررسی کنید)

برای فهرست کامل ارائه‌دهندگان (xAI، ‏Groq، ‏Mistral و غیره) و پیکربندی پیشرفته،
به [ارائه‌دهندگان مدل](/fa/concepts/model-providers) مراجعه کنید.
