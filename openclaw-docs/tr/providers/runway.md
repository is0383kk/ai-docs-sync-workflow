---
read_when:
    - OpenClaw'da Runway video oluşturmayı kullanmak istiyorsunuz
    - Runway API anahtarı/ortam değişkeni kurulumuna ihtiyacınız var
    - Runway'i varsayılan video sağlayıcısı yapmak istiyorsunuz
summary: OpenClaw'da Runway video oluşturma kurulumu
title: Runway
x-i18n:
    generated_at: "2026-07-27T00:14:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a56e768893e327b56d70e8b8c2d426123a861b3cf05c0107d98104e2cee856c
    source_path: providers/runway.md
    workflow: 16
---

OpenClaw, barındırılan video üretimi için paketlenmiş bir `runway` sağlayıcısıyla birlikte gelir; bu sağlayıcı varsayılan olarak etkindir ve `videoGenerationProviders` sözleşmesine göre kaydedilmiştir.

| Özellik              | Değer                                                             |
| -------------------- | ----------------------------------------------------------------- |
| Sağlayıcı kimliği    | `runway`                                                |
| Plugin               | paketlenmiş, `enabledByDefault: true`                                   |
| Kimlik doğrulama ortam değişkenleri | `RUNWAYML_API_SECRET` (standart) veya `RUNWAY_API_KEY` |
| İlk kurulum bayrağı  | `--auth-choice runway-api-key`                                                |
| Doğrudan CLI bayrağı | `--runway-api-key <key>`                                                |
| API                  | Runway görev tabanlı video üretimi (`GET /v1/tasks/{id}` yoklaması) |
| Varsayılan model     | `runway/gen4.5`                                                |

## Başlarken

<Steps>
  <Step title="API anahtarını ayarlayın">
    ```bash
    openclaw onboard --auth-choice runway-api-key
    ```
  </Step>
  <Step title="Runway'i varsayılan video sağlayıcısı olarak ayarlayın">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "runway/gen4.5"
    ```
  </Step>
  <Step title="Video oluşturun">
    Agent'tan bir video oluşturmasını isteyin. Runway otomatik olarak kullanılır.
  </Step>
</Steps>

## Desteklenen modlar ve modeller

Sağlayıcı, üç moda ayrılmış yedi Runway modeli sunar. Aynı model kimliği birden fazla modda kullanılabilir (örneğin `gen4.5`, hem metinden videoya hem de görüntüden videoya dönüştürme için çalışır).

| Mod                | Modeller                                                                | Referans girdisi                  |
| ------------------ | ----------------------------------------------------------------------- | --------------------------------- |
| Metinden videoya   | `gen4.5` (varsayılan), `veo3.1`, `veo3.1_fast`, `veo3` | Yok                               |
| Görüntüden videoya | `gen4.5`, `gen4_turbo`, `gen3a_turbo`, `veo3.1`, `veo3.1_fast`, `veo3` | 1 yerel veya uzak görüntü         |
| Videodan videoya   | `gen4_aleph`                                                      | 1 yerel veya uzak video           |

Yerel görüntü ve video referansları, veri URI'leri aracılığıyla desteklenir.

| En-boy oranları                 | İzin verilen değerler                         |
| ------------------------------- | --------------------------------------------- |
| Metinden videoya                | `16:9`, `9:16`        |
| Görüntü ve video düzenlemeleri  | `1:1`, `16:9`, `9:16`, `3:4`, `4:3`, `21:9` |

<Warning>
  Videodan videoya dönüştürme şu anda `runway/gen4_aleph` gerektirir. Diğer Runway model kimlikleri video referansı girdilerini reddeder.
</Warning>

<Note>
  Yanlış sütundan bir Runway model kimliği seçilmesi, API isteği OpenClaw'dan çıkmadan önce açık bir hataya neden olur. Sağlayıcı, `extensions/runway/video-generation-provider.ts` içinde `model` değerini modun izin verilenler listesine (`TEXT_ONLY_MODELS`, `IMAGE_MODELS`, `VIDEO_MODELS`) göre doğrular.
</Note>

## Yapılandırma

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "runway/gen4.5",
      },
    },
  },
}
```

## Gelişmiş yapılandırma

<AccordionGroup>
  <Accordion title="Ortam değişkeni diğer adları">
    OpenClaw hem `RUNWAYML_API_SECRET` (standart) hem de `RUNWAY_API_KEY` değişkenini tanır.
    Her iki değişken de Runway sağlayıcısının kimliğini doğrular.
  </Accordion>

  <Accordion title="Görev yoklaması">
    Runway görev tabanlı bir API kullanır. Bir üretim isteği gönderildikten sonra OpenClaw,
    video hazır olana kadar `GET /v1/tasks/{id}` için yoklama yapar. Yoklama davranışı için
    ek yapılandırma gerekmez.
  </Accordion>
</AccordionGroup>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Video üretimi" href="/tr/tools/video-generation" icon="video">
    Paylaşılan araç parametreleri, sağlayıcı seçimi ve eşzamansız davranış.
  </Card>
  <Card title="Yapılandırma referansı" href="/tr/gateway/config-agents#agent-defaults" icon="gear">
    Video üretim modeli dâhil olmak üzere Agent varsayılan ayarları.
  </Card>
</CardGroup>
