---
read_when:
    - OpenClaw'u GMI Cloud modelleriyle çalıştırmak istiyorsunuz
    - GMI sağlayıcı kimliği, anahtarı veya uç noktası gereklidir
summary: OpenClaw ile GMI Cloud'un OpenAI uyumlu API'sini kullanın
title: GMI Cloud
x-i18n:
    generated_at: "2026-07-26T23:57:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a21fd2a997f44e1f78d97a0fba24ca2bbc00dd193323da712d650ed4ba105355
    source_path: providers/gmi.md
    workflow: 16
---

GMI Cloud, OpenAI uyumlu bir API'nin arkasında frontier ve açık ağırlıklı modeller için barındırılan bir çıkarım platformudur. OpenClaw'da resmi bir harici sağlayıcı
pluginidir: bir kez yükleyin, kimlik bilgilerini normal model kimlik doğrulaması üzerinden saklayın ve
`gmi/google/gemini-3.1-flash-lite` gibi model referanslarını kullanın.

Anthropic, DeepSeek, Google, Moonshot, OpenAI ve GMI'ın
kataloğunda sunulan Z.AI rotaları dahil olmak üzere çeşitli barındırılan model aileleri için tek bir API anahtarı istediğinizde GMI'ı kullanın.
Model yedek geçişi için ikincil bir sağlayıcı olarak, satıcılar arasındaki
barındırılan rotaları karşılaştırmak için veya GMI bir modeli birincil
sağlayıcınızdan önce kullanıma sunduğunda kullanılabilir. Sağlayıcı kimliğinin, kimlik doğrulama profilinin, takma adların,
model kataloğu başlangıç verilerinin ve temel URL'nin sahibi OpenClaw'dur; canlı model kullanılabilirliğinin, faturalandırmanın,
hız sınırlarının ve sağlayıcı tarafındaki tüm yönlendirme politikalarının sahibi GMI'dır.

| Özellik      | Değer                                    |
| ------------- | ---------------------------------------- |
| Sağlayıcı kimliği   | `gmi` (takma adlar: `gmi-cloud`, `gmicloud`) |
| Paket       | `@openclaw/gmi-provider`                 |
| Kimlik doğrulama ortam değişkeni  | `GMI_API_KEY`                            |
| API           | OpenAI uyumlu (`openai-completions`) |
| Temel URL      | `https://api.gmi-serving.com/v1`         |
| Varsayılan model | `gmi/google/gemini-3.1-flash-lite`       |

## Kurulum

Plugini yükleyin, Gateway'i yeniden başlatın, ardından GMI Cloud'da
bir API anahtarı oluşturun (`https://www.gmicloud.ai/`):

```bash
openclaw plugins install @openclaw/gmi-provider
openclaw gateway restart
```

Ardından şunu çalıştırın:

```bash
openclaw onboard --auth-choice gmi-api-key
```

Etkileşimsiz kurulumlarda `--gmi-api-key <key>` aktarılabilir veya şu ayarlanabilir:

```bash
export GMI_API_KEY="<your-gmi-api-key>" # pragma: allowlist secret
```

## GMI ne zaman seçilmeli?

- Yerel bir model sunucusu yerine barındırılan, OpenAI uyumlu bir uç nokta istiyorsunuz.
- Tek bir sağlayıcı hesabı üzerinden çeşitli ticari ve açık ağırlıklı model ailelerini
  denemek istiyorsunuz.
- DeepInfra, OpenRouter, Together veya doğrudan satıcı API'lerinden farklı üst kaynak yönlendirmesine sahip
  bir yedek sağlayıcı istiyorsunuz.
- GMI'a özgü model kimliklerine, fiyatlandırmaya veya hesap denetimlerine ihtiyacınız var.

GMI'ın OpenAI uyumlu rotası üzerinden sunmadığı satıcıya özgü özelliklere
ihtiyacınız olduğunda bunun yerine doğrudan satıcı sağlayıcısını seçin. Veri yerelliği veya yerel
GPU denetimi, barındırma kolaylığından daha önemli olduğunda LM Studio, Ollama, SGLang ya da vLLM gibi yerel
bir sağlayıcı seçin.

## Modeller

Plugin kataloğu, yaygın olarak kullanılabilen GMI Cloud rota kimliklerini başlangıç verisi olarak ekler:

| Model referansı                          | Girdi        | Bağlam   | Azami çıktı |
| ---------------------------------- | ------------ | --------- | ---------- |
| `gmi/anthropic/claude-sonnet-4.6`  | metin + görüntü | 200,000   | 64,000     |
| `gmi/deepseek-ai/DeepSeek-V3.2`    | metin         | 163,840   | 65,536     |
| `gmi/google/gemini-3.1-flash-lite` | metin + görüntü | 1,048,576 | 65,536     |
| `gmi/moonshotai/Kimi-K2.5`         | metin + görüntü | 262,144   | 65,536     |
| `gmi/openai/gpt-5.4`               | metin + görüntü | 400,000   | 128,000    |
| `gmi/zai-org/GLM-5.1-FP8`          | metin         | 202,752   | 65,536     |

Katalog bir başlangıç verisidir; her hesabın her zaman her modeli çağırabileceğinin
garantisi değildir. Yapılandırılmış sağlayıcının ortamınızda bildirdiklerini listeleyin:

```bash
openclaw models list --provider gmi
```

## Sorun giderme

- `401` veya `403`: `GMI_API_KEY` değişkeninin OpenClaw'ı çalıştıran işlem için ayarlandığını denetleyin
  ya da anahtarı sağlayıcı kimlik doğrulama profilinde saklamak için ilk kurulumu yeniden çalıştırın.
- Bilinmeyen model hataları: modelin GMI hesabınızda bulunduğunu doğrulayın ve
  `openclaw models list --provider gmi` tarafından gösterilen tam `gmi/<route-id>` referansını kullanın.
- Aralıklı sağlayıcı hataları: farklı bir GMI rotası deneyin veya GMI'ı
  tek birincil model sağlayıcısı yerine yedek olarak yapılandırın.

## İlgili

- [Model sağlayıcıları](/tr/concepts/model-providers)
- [Tüm sağlayıcılar](/tr/providers/index)
