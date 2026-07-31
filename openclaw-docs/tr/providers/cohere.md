---
read_when:
    - OpenClaw ile Cohere kullanmak istiyorsunuz
    - Cohere API anahtarı ortam değişkenine veya CLI kimlik doğrulama seçeneğine ihtiyacınız var
summary: Cohere kurulumu (kimlik doğrulama + model seçimi)
title: Cohere
x-i18n:
    generated_at: "2026-07-26T22:58:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fee46bf80609bd5e8211d6be507713f4de178653941effb81ebae48d8bb6528a
    source_path: providers/cohere.md
    workflow: 16
---

[Cohere](https://cohere.com), Compatibility API'si aracılığıyla OpenAI uyumlu çıkarım sağlar. OpenClaw, Cohere sağlayıcısını haricileştirme geçişi sırasında paketle birlikte sunar ve ayrıca resmî bir harici plugin olarak yayımlar.

| Özellik               | Değer                                                  |
| --------------------- | ------------------------------------------------------ |
| Sağlayıcı kimliği     | `cohere`                                     |
| Plugin                | geçiş sırasında paketle birlikte; resmî harici paket   |
| Kimlik doğrulama ortam değişkeni | `COHERE_API_KEY`                           |
| İlk kurulum bayrağı   | `--auth-choice cohere-api-key`                                     |
| Doğrudan CLI bayrağı  | `--cohere-api-key <key>`                                     |
| API                   | OpenAI uyumlu (`openai-completions`)                     |
| Temel URL             | `https://api.cohere.ai/compatibility/v1`                                     |
| Varsayılan model      | `cohere/command-a-plus-05-2026`                                     |
| Bağlam penceresi      | 128,000 token                                          |

## Yerleşik katalog

| Model referansı                      | Girdi       | Bağlam  | En fazla çıktı | Notlar                                               |
| ------------------------------------ | ----------- | ------- | -------------- | ---------------------------------------------------- |
| `cohere/command-a-plus-05-2026`                   | metin, görsel | 128,000 | 64,000       | Varsayılan; amiral gemisi eyleyici ve akıl yürütme modeli |
| `cohere/command-a-03-2025`                   | metin       | 256,000 | 8,000          | Önceki Command A modeli                              |
| `cohere/command-a-reasoning-08-2025`                   | metin       | 256,000 | 32,000         | Eyleyici akıl yürütme ve araç kullanımı              |
| `cohere/command-a-vision-07-2025`                   | metin, görsel | 128,000 | 8,000        | Görsel ve belge analizi; araç kullanımı yok          |
| `cohere/north-mini-code-1-0`                   | metin, görsel | 256,000 | 64,000       | Eyleyici kodlama; akıl yürütme; ücretsiz limitler    |

Akıl yürütme yeteneğine sahip Cohere modelleri, iki Compatibility API akıl yürütme modunu destekler. OpenClaw, **kapalı** seçeneğini `none` ile ve etkinleştirilmiş tüm düşünme düzeylerini `high` ile eşler. Command A Vision araç kullanımını desteklemediğinden OpenClaw, bu model için eyleyici araçlarını devre dışı tutar.

## Başlarken

1. Cohere, güncel OpenClaw paketleriyle birlikte gelir. Eksikse harici paketi yükleyin ve Gateway'i yeniden başlatın:

```bash
openclaw plugins install @openclaw/cohere-provider
openclaw gateway restart
```

2. Bir Cohere API anahtarı oluşturun.
3. İlk kurulumu çalıştırın:

```bash
openclaw onboard --non-interactive \
  --auth-choice cohere-api-key \
  --cohere-api-key "$COHERE_API_KEY"
```

4. Kataloğun kullanılabilir olduğunu doğrulayın:

```bash
openclaw models list --provider cohere
```

İlk kurulum, yalnızca henüz birincil model yapılandırılmamışsa Cohere'ı birincil model olarak ayarlar.

## Yalnızca ortam değişkeniyle kurulum

`COHERE_API_KEY` değişkenini Gateway işleminin kullanımına sunun, ardından Cohere modelini seçin:

```json5
{
  agents: {
    defaults: {
      model: { primary: "cohere/command-a-plus-05-2026" },
    },
  },
}
```

<Note>
Gateway bir arka plan hizmeti olarak veya Docker'da çalışıyorsa söz konusu hizmet için `COHERE_API_KEY` değişkenini ayarlayın. Bunu yalnızca etkileşimli bir kabukta dışa aktarmak, değişkeni zaten çalışmakta olan bir Gateway'in kullanımına sunmaz.
</Note>

## İlgili

- [Model sağlayıcıları](/tr/concepts/model-providers)
- [Modeller CLI'si](/tr/cli/models)
- [Sağlayıcı dizini](/tr/providers/index)
