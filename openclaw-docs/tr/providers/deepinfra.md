---
read_when:
    - En iyi açık kaynaklı LLM'ler için tek bir API anahtarı istiyorsunuz
    - DeepInfra'nın API'si aracılığıyla OpenClaw'da modeller çalıştırmak istiyorsunuz
summary: OpenClaw'da en popüler açık kaynak ve öncü modellere erişmek için DeepInfra'nın birleşik API'sini kullanın
title: DeepInfra
x-i18n:
    generated_at: "2026-07-26T23:32:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a63bdd4ffd2189cde50f0ee601fd7ee32ca86c943a9899072f0c140823608004
    source_path: providers/deepinfra.md
    workflow: 16
---

DeepInfra, istekleri tek bir OpenAI uyumlu uç nokta ve API anahtarının
ardında popüler açık kaynak ve öncü modellere yönlendirir. Çoğu OpenAI SDK'sı,
temel URL değiştirilerek bununla çalışır.

## Plugin'i yükleme

```bash
openclaw plugins install @openclaw/deepinfra-provider
openclaw gateway restart
```

## API anahtarı edinme

1. [deepinfra.com](https://deepinfra.com/) adresinde oturum açın
2. Dashboard / Keys bölümüne gidip bir anahtar oluşturun veya otomatik oluşturulan anahtarı kullanın

## CLI kurulumu

```bash
openclaw onboard --deepinfra-api-key <key>
```

Alternatif olarak ortam değişkenini ayarlayın:

```bash
export DEEPINFRA_API_KEY="<your-deepinfra-api-key>" # pragma: allowlist secret
```

## Yapılandırma parçacığı

```json5
{
  env: { DEEPINFRA_API_KEY: "<your-deepinfra-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "deepinfra/deepseek-ai/DeepSeek-V4-Flash" },
    },
  },
}
```

## Desteklenen yüzeyler

Sohbet, görüntü oluşturma ve video oluşturma, `DEEPINFRA_API_KEY`
yapılandırıldıktan sonra model kataloglarını canlı olarak `https://api.deepinfra.com/v1/openai/models?sort_by=openclaw&filter=with_meta`
üzerinden yeniler. Canlı keşif, seçilebilir modellerin listesini genişletir;
her yüzeyin varsayılan modeli aşağıdaki statik değer olarak kalır. Diğer
yüzeyler, aynı canlı kataloğa geçene kadar statik katalogları kullanır.

| Yüzey                   | Varsayılan model                                                                  | OpenClaw yapılandırması/aracı                                  |
| ----------------------- | --------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| Sohbet / model sağlayıcı | `deepseek-ai/DeepSeek-V4-Flash` (canlı katalog daha fazla sohbet modeli ekler)                 | `agents.defaults.model`                                             |
| Görüntü oluşturma/düzenleme | `black-forest-labs/FLUX-1-schnell` (canlı katalog daha fazla `image-gen` modeli ekler) | `image_generate`, `agents.defaults.mediaModels.image`                         |
| Medya anlama            | Görüntüler için `moonshotai/Kimi-K2.5`                                                | gelen görüntüleri anlama                                       |
| Konuşmadan metne        | `openai/whisper-large-v3-turbo`                                                                | gelen sesleri yazıya dökme                                     |
| Metinden konuşmaya      | `hexgrad/Kokoro-82M`                                                                | `tts.provider: "deepinfra"`                                             |
| Video oluşturma         | `Pixverse/Pixverse-T2V` (canlı katalog daha fazla `video-gen` modeli ekler)     | `video_generate`, `agents.defaults.mediaModels.video`                         |
| Bellek gömmeleri        | `BAAI/bge-m3`                                                                | `memory.search.provider: "deepinfra"`                                             |

DeepInfra ayrıca yeniden sıralama, sınıflandırma, nesne algılama ve diğer
yerel model türlerini de sunar. OpenClaw henüz bu kategoriler için bir
sağlayıcı sözleşmesine sahip olmadığından bu Plugin bunları kaydetmez.

## Kullanılabilir modeller

OpenClaw, bir anahtar yapılandırıldıktan sonra DeepInfra modellerini dinamik
olarak keşfeder. Güncel listeyi görmek için `/models deepinfra` veya
`openclaw models list --provider deepinfra` kullanın.

[deepinfra.com](https://deepinfra.com/) üzerindeki tüm modeller
`deepinfra/` ön ekiyle çalışır:

```text
deepinfra/deepseek-ai/DeepSeek-V4-Flash
deepinfra/deepseek-ai/DeepSeek-V3.2
deepinfra/MiniMaxAI/MiniMax-M2.5
deepinfra/moonshotai/Kimi-K2.5
deepinfra/nvidia/NVIDIA-Nemotron-3-Super-120B-A12B
deepinfra/zai-org/GLM-5.1
...ve çok daha fazlası
```

## Notlar

- Model referansları `deepinfra/<provider>/<model>` biçimindedir (örneğin `deepinfra/Qwen/Qwen3-Max`).
- Varsayılan sohbet modeli: `deepinfra/deepseek-ai/DeepSeek-V4-Flash`
- Temel URL: `https://api.deepinfra.com/v1/openai`
- Video oluşturma, OpenAI uyumlu eşzamansız `https://api.deepinfra.com/v1/openai/videos` uç noktasını kullanır (gönderin, ardından yoklayın). Yapılandırılmış bir `baseUrl` dikkate alınır. `openclaw doctor --fix`, `api.deepinfra.com` üzerindeki eski `nativeBaseUrl` veya `/v1/inference` değerlerini otomatik olarak `baseUrl` biçimine geçirir; özel yerel uç noktalar bir doctor bildirimiyle kullanımdan kaldırılır ve elle yapılandırılmış, OpenAI uyumlu bir `baseUrl` gerektirir. `baseUrl` hâlâ kullanımdan kaldırılmış `/v1/inference` yüzeyini hedeflerken video oluşturma, herhangi bir istek göndermeden önce uygulanabilir çözüm içeren bir hatayla başarısız olur.

## İlgili

- [Model sağlayıcıları](/tr/concepts/model-providers)
- [Tüm sağlayıcılar](/tr/providers/index)
