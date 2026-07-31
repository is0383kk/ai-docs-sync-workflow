---
read_when:
    - OpenClaw ile Featherless AI kullanmak istiyorsunuz
    - Featherless API anahtarı ortam değişkenine veya model referansı biçimine ihtiyacınız var
summary: Featherless AI kurulumu, model seçimi ve araç çağırma
title: Featherless AI
x-i18n:
    generated_at: "2026-07-26T22:58:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9112f7e65b4089bf96933c632d0b62f7fb87d42998d985ca85eb92dc392636b6
    source_path: providers/featherless.md
    workflow: 16
---

[Featherless AI](https://featherless.ai), açık modelleri OpenAI uyumlu bir API
üzerinden sunar. OpenClaw, Featherless'ı resmi bir harici sağlayıcı Plugin'i
olarak kurar ve çalışma zamanında Featherless'ın tam model kimliklerini kabul
ederken yerleşik kataloğu küçük tutar.

| Özellik                 | Değer                                    |
| ----------------------- | ---------------------------------------- |
| Sağlayıcı kimliği       | `featherless`                       |
| Paket                   | `@openclaw/featherless-provider`                       |
| Kimlik doğrulama ortam değişkeni | `FEATHERLESS_API_KEY`              |
| İlk kurulum bayrağı     | `--auth-choice featherless-api-key`                       |
| Doğrudan CLI bayrağı    | `--featherless-api-key <key>`                       |
| API                     | OpenAI uyumlu (`openai-completions`)       |
| Temel URL               | `https://api.featherless.ai/v1`                       |
| Varsayılan model        | `featherless/Qwen/Qwen3-32B`                       |

## Kurulum

Plugin'i kurun ve Gateway'i yeniden başlatın:

```bash
openclaw plugins install @openclaw/featherless-provider
openclaw gateway restart
```

İlk kurulumu çalıştırın:

```bash
openclaw onboard --auth-choice featherless-api-key
```

Etkileşimsiz kurulum için:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice featherless-api-key \
  --featherless-api-key "$FEATHERLESS_API_KEY"
```

Alternatif olarak anahtarı Gateway işlemine açın:

```bash
export FEATHERLESS_API_KEY="<your-featherless-api-key>" # pragma: izin verilenler listesi sırrı
```

Sağlayıcıyı doğrulayın:

```bash
openclaw models list --provider featherless
```

## Varsayılan model

Featherless, Qwen 3 ailesi için yerel araç çağırmayı belgelediğinden Plugin,
kurulum varsayılanı olarak `Qwen/Qwen3-32B` kullanır. OpenClaw bunun
32,768 token'lık bağlam penceresini, ölçülü bir 4,096 token'lık çıktı sınırını
ve Qwen sohbet şablonunun düşünme denetimlerini yapılandırır.

Featherless birden fazla faturalandırma modunu desteklediğinden ve OpenClaw
hesaba özgü plan ya da istek fiyatlandırma tarifelerini yerleşik olarak
içermediğinden katalog maliyet alanları sıfırdır.

## Diğer Featherless modelleri

`featherless/` sağlayıcı ön ekinden sonra tam Featherless model kimliğini
kullanın:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "featherless/moonshotai/Kimi-K2-Instruct",
      },
    },
  },
}
```

OpenClaw, Featherless'ın genel model dizininin tamamını kasıtlı olarak seçiciye
kopyalamaz. Dizin büyüktür ve her metin, görsel, gömme ve akıl yürütme modelini
güvenle sınıflandırmak için yeterli yapılandırılmış yetenek meta verisi sunmaz.
Bu nedenle bilinmeyen kimlikler ölçülü, yalnızca metin destekleyen ve akıl
yürütmeyen varsayılanlarla çözümlenir: 4,096 token'lık bağlam penceresi ve
1,024 token'lık çıktı sınırı.

Bir model farklı meta verilere ihtiyaç duyduğunda açık bir sağlayıcı model
girdisi ekleyin:

```json5
{
  models: {
    mode: "merge",
    providers: {
      featherless: {
        baseUrl: "https://api.featherless.ai/v1",
        apiKey: "${FEATHERLESS_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "google/gemma-3-27b-it",
            name: "Gemma 3 27B",
            input: ["text", "image"],
            reasoning: false,
            contextWindow: 32768,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

Özel meta veri eklemeden önce güncel model kullanılabilirliği ve yetenek
etiketleri için Featherless model kataloğunu kontrol edin.

## Sorun giderme

- `401` veya `403`: `FEATHERLESS_API_KEY` değerinin Gateway
  işlemi tarafından görülebildiğini doğrulayın ya da ilk kurulumu yeniden çalıştırın.
- Bilinmeyen model: Featherless'tan alınan, büyük/küçük harfe duyarlı tam kimliği
  `featherless/` ön ekinden sonra kullanın.
- Araç çağrıları metin olarak döndürüldü: Qwen 3 gibi Featherless'ın yerel
  işlev çağırma desteğini belgelediği bir model ailesi seçin.
- Yönetilen Gateway anahtarı göremiyor: anahtarı `~/.openclaw/.env` içine veya
  hizmetin yüklediği başka bir ortam kaynağına koyun, ardından Gateway'i yeniden başlatın.

## İlgili

- [Model sağlayıcıları](/tr/concepts/model-providers)
- [Tüm sağlayıcılar](/tr/providers/index)
- [Düşünme modları](/tr/tools/thinking)
