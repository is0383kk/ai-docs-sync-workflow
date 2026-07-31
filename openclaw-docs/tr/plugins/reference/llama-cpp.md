---
read_when:
    - llama-cpp Plugin'ini kuruyor, yapılandırıyor veya denetliyorsunuz
summary: node-llama-cpp aracılığıyla yerel GGUF metin çıkarımı ve gömmeleri.
title: Llama Cpp Plugin'i
x-i18n:
    generated_at: "2026-07-26T22:55:53Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2756d4b3e00bbe37b4dedec1d54d28bfe6662e8105504317a402293254ce0240
    source_path: plugins/reference/llama-cpp.md
    workflow: 16
---

# Llama Cpp plugin'i

node-llama-cpp aracılığıyla yerel GGUF metin çıkarımı ve gömmeleri.

## Dağıtım

- Paket: `@openclaw/llama-cpp-provider`
- Kurulum yolu: npm; ClawHub

## Yüzey

sağlayıcılar: `llama-cpp`; sözleşmeler: `embeddingProviders`

<!-- openclaw-plugin-reference:manual-start -->

## Varsayılan metin modeli

Etkileşimli kurulum sırasında OpenClaw, yaklaşık 5.0 GB boyutunda paketlenmiş bir indirme olarak Gemma 4 E4B IT Q4_K_M modelini sunar. Bu teklif en az 16 GiB toplam RAM gerektirir. Daha küçük makinelerde önceden önbelleğe alınmış modeller yine de algılanır.

Başka bir model kullanmak için `params.modelPath` değerini herhangi bir özel GGUF olarak ayarlayın. Özel modeller, paketlenmiş indirme için geçerli RAM gereksinimine tabi değildir. Gereksinimi karşılamayan makinelerde Ollama veya LM Studio aracılığıyla daha küçük bir model çalıştırabilir ya da bir bulut sağlayıcısı seçebilirsiniz.

<!-- openclaw-plugin-reference:manual-end -->

## İlgili belgeler

- [llama-cpp](/tr/plugins/llama-cpp)
