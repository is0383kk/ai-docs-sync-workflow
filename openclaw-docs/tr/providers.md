---
read_when:
    - Bir model sağlayıcısı seçmek istiyorsunuz
    - Desteklenen LLM arka uçlarına hızlı bir genel bakışa ihtiyacınız var
summary: OpenClaw tarafından desteklenen model sağlayıcıları (LLM'ler)
title: Sağlayıcı dizini
x-i18n:
    generated_at: "2026-07-26T23:37:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e98910f016e461dedcd06e40a2933631bbd6ac09ceebd340bab82f14805e06a6
    source_path: providers/index.md
    workflow: 16
---

OpenClaw birçok LLM sağlayıcısını kullanabilir. Bir sağlayıcı seçin, kimlik doğrulaması yapın, ardından
varsayılan modeli `provider/model` olarak ayarlayın.

Sohbet kanalı belgelerini mi arıyorsunuz (WhatsApp/Telegram/Discord/Slack/Mattermost (plugin)/vb.)? Bkz. [Kanallar](/tr/channels).

## Hızlı başlangıç

1. Sağlayıcıyla kimlik doğrulaması yapın (genellikle `openclaw onboard` aracılığıyla).
2. Varsayılan modeli ayarlayın:

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-4-6" } } },
}
```

## Sağlayıcı belgeleri

- [Alibaba Model Studio](/tr/providers/alibaba)
- [Amazon Bedrock](/tr/providers/bedrock)
- [Amazon Bedrock Mantle](/tr/providers/bedrock-mantle)
- [Anthropic (API + Claude CLI)](/tr/providers/anthropic)
- [Arcee AI (Trinity modelleri)](/tr/providers/arcee)
- [Azure Speech](/tr/providers/azure-speech)
- [Baseten (Inkling + Model API'leri)](/providers/baseten)
- [BytePlus (Uluslararası)](/tr/concepts/model-providers#byteplus-international)
- [Cerebras](/tr/providers/cerebras)
- [Chutes](/tr/providers/chutes)
- [ClawRouter (yönetilen çok sağlayıcılı yönlendirme)](/tr/providers/clawrouter)
- [Cloudflare AI Gateway](/tr/providers/cloudflare-ai-gateway)
- [Cohere](/tr/providers/cohere)
- [ComfyUI](/tr/providers/comfy)
- [DeepSeek](/tr/providers/deepseek)
- [ds4 (yerel DeepSeek V4)](/tr/providers/ds4)
- [ElevenLabs](/tr/providers/elevenlabs)
- [fal](/tr/providers/fal)
- [Featherless AI](/tr/providers/featherless)
- [Fireworks](/tr/providers/fireworks)
- [GitHub Copilot](/tr/providers/github-copilot)
- [GMI Cloud](/tr/providers/gmi)
- [Google (Gemini)](/tr/providers/google)
- [Gradium](/tr/providers/gradium)
- [Groq (LPU çıkarımı)](/tr/providers/groq)
- [Hugging Face (Çıkarım)](/tr/providers/huggingface)
- [inferrs (yerel modeller)](/tr/providers/inferrs)
- [Kilocode](/tr/providers/kilocode)
- [LiteLLM (birleşik gateway)](/tr/providers/litellm)
- [LM Studio (yerel modeller)](/tr/providers/lmstudio)
- [LongCat](/tr/providers/longcat)
- [MiniMax](/tr/providers/minimax)
- [Mistral](/tr/providers/mistral)
- [Moonshot AI (Kimi + Kimi Coding)](/tr/providers/moonshot)
- [NovitaAI](/tr/providers/novita)
- [NVIDIA](/tr/providers/nvidia)
- [Ollama (bulut + yerel modeller)](/tr/providers/ollama)
- [Ollama Cloud](/tr/providers/ollama-cloud)
- [OpenAI (API + Codex)](/tr/providers/openai)
- [OpenCode](/tr/providers/opencode)
- [OpenCode Go](/tr/providers/opencode-go)
- [OpenRouter](/tr/providers/openrouter)
- [Perplexity (web araması)](/tr/providers/perplexity-provider)
- [Qianfan](/tr/providers/qianfan)
- [Qwen Cloud](/tr/providers/qwen)
- [Runway](/tr/providers/runway)
- [SenseAudio](/tr/providers/senseaudio)
- [SGLang (yerel modeller)](/tr/providers/sglang)
- [StepFun](/tr/providers/stepfun)
- [Synthetic](/tr/providers/synthetic)
- [Tencent Cloud (TokenHub / TokenPlan)](/tr/providers/tencent)
- [Together AI](/tr/providers/together)
- [Venice (Venice AI, gizlilik odaklı)](/tr/providers/venice)
- [Vercel AI Gateway](/tr/providers/vercel-ai-gateway)
- [vLLM (yerel modeller)](/tr/providers/vllm)
- [Volcengine (Doubao)](/tr/providers/volcengine)
- [Vydra](/tr/providers/vydra)
- [xAI](/tr/providers/xai)
- [Xiaomi](/tr/providers/xiaomi)
- [Z.AI (GLM)](/tr/providers/zai)

## Paylaşılan genel bakış sayfaları

- [Ek sağlayıcı varyantları](/tr/providers/models#additional-provider-variants) - Anthropic Vertex, Copilot Proxy ve Gemini CLI OAuth
- [Görüntü Oluşturma](/tr/tools/image-generation) - Paylaşılan `image_generate` aracı, sağlayıcı seçimi ve yük devretme
- [Müzik Oluşturma](/tr/tools/music-generation) - Paylaşılan `music_generate` aracı, sağlayıcı seçimi ve yük devretme
- [Video Oluşturma](/tr/tools/video-generation) - Paylaşılan `video_generate` aracı, sağlayıcı seçimi ve yük devretme

## Transkripsiyon sağlayıcıları

- [Deepgram (ses transkripsiyonu)](/tr/providers/deepgram)
- [ElevenLabs](/tr/providers/elevenlabs#speech-to-text)
- [Mistral](/tr/providers/mistral#audio-transcription-voxtral)
- [OpenAI](/tr/providers/openai)
- [SenseAudio](/tr/providers/senseaudio)
- [xAI](/tr/providers/xai)

## Topluluk araçları

- [Claude Max API Proxy](/tr/providers/claude-max-api-proxy) - Claude abonelik kimlik bilgileri için topluluk proxy'si (kullanmadan önce Anthropic politikalarını/koşullarını doğrulayın)

Sağlayıcıların tam kataloğu (xAI, Groq, Mistral vb.) ve gelişmiş yapılandırma için
bkz. [Model sağlayıcıları](/tr/concepts/model-providers).
