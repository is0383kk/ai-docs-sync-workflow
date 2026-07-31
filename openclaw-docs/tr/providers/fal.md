---
read_when:
    - OpenClaw'da fal görüntü oluşturmayı kullanmak istiyorsunuz
    - FAL_KEY kimlik doğrulama akışına ihtiyacınız var
    - image_generate, video_generate veya music_generate için fal varsayılanlarını istiyorsunuz
summary: OpenClaw'da fal ile görüntü, video ve müzik oluşturma kurulumu
title: Fal
x-i18n:
    generated_at: "2026-07-26T22:58:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9bd868aaf6771f6fa38bb8e2a83133460d150e2a5aa9e5b888e221c07f29e0ad
    source_path: providers/fal.md
    workflow: 16
---

OpenClaw, barındırılan görüntü, video ve müzik üretimi için paketlenmiş bir `fal` sağlayıcısıyla birlikte gelir.

| Özellik    | Değer                                                                           |
| ---------- | ------------------------------------------------------------------------------- |
| Sağlayıcı  | `fal`                                                                           |
| Kimlik doğrulama | `FAL_KEY` (standart; `FAL_API_KEY` yedek olarak da çalışır)                   |
| API        | fal model uç noktaları (`https://fal.run`; video işleri `https://queue.fal.run` kullanır) |
| Temel URL  | `models.providers.fal.baseUrl` ile geçersiz kılın                                    |

## Başlarken

<Steps>
  <Step title="API anahtarını ayarlayın">
    ```bash
    openclaw onboard --auth-choice fal-api-key
    ```

    Etkileşimsiz kurulumlar `--fal-api-key <key>` iletebilir veya `FAL_KEY` dışa aktarabilir.
    İlk katılım, yapılandırılmış bir model olmadığında `fal/fal-ai/flux/dev` değerini
    varsayılan görüntü modeli olarak da ayarlar.

  </Step>
  <Step title="Varsayılan görüntü modelini ayarlayın">
    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "fal/fal-ai/flux/dev",
          },
        },
      },
    }
    ```
  </Step>
</Steps>

## Görüntü üretimi

Paketlenmiş `fal` görüntü üretimi sağlayıcısının varsayılanı
`fal/fal-ai/flux/dev` değeridir.

| Yetenek               | Değer                                                              |
| --------------------- | ------------------------------------------------------------------ |
| En fazla görüntü      | İstek başına 4; Krea 2: istek başına 1                             |
| Boyut geçersiz kılmaları | `1024x1024`, `1024x1536`, `1536x1024`, `1024x1792`, `1792x1024`    |
| En-boy oranı          | Flux görüntüden görüntüye dışında her yerde desteklenir            |
| Çözünürlük            | `1K`, `2K`, `4K` (model başına sınırlar aşağıdadır)                          |
| Çıktı biçimi          | `png` (varsayılan) veya `jpeg`; Krea 2, `outputFormat` geçersiz kılmalarını reddeder |

Düzenleme istekleri (paylaşılan `image` / `images` parametreleri aracılığıyla referans görüntüler)
model başına referans sınırları bulunan model bazlı bir düzenleme uç noktasına yönlendirilir:

| Model ailesi              | `fal/` sonrasındaki model başvurusu | Düzenleme uç noktası | En fazla referans görüntü |
| ------------------------- | -------------------------------------- | -------------------- | ------------------------- |
| Flux ve diğer fal modelleri | `fal-ai/flux/dev` (varsayılan)      | `/image-to-image`   | 1                         |
| GPT Image                 | `openai/gpt-image-*`                   | `/edit`           | 10                   |
| Grok Imagine              | `xai/grok-imagine-image`               | `/edit`           | 3                    |
| Nano Banana (eski)        | `fal-ai/nano-banana`                   | `/edit`           | 3                    |
| Nano Banana 2             | `fal-ai/nano-banana-*`                 | `/edit`           | 14                   |
| Nano Banana 2 Lite        | `google/nano-banana-2-lite`            | `/edit`           | 14                   |
| Krea 2                    | `krea/v2/{medium,large}/text-to-image` | yok (stil referansları) | 10 stil referansı  |

<Warning>
Flux görüntüden görüntüye istekleri `aspectRatio` geçersiz kılmalarını **desteklemez**. GPT
Image ve Nano Banana 2 düzenleme istekleri fal'ın `/edit` uç noktasını kullanır ve
en-boy oranı ipuçlarını kabul eder. Nano Banana 2 ayrıca `4:1`,
`1:4`, `8:1` ve `1:8` gibi ek yerel geniş/uzun oranları kabul eder; Krea 2 kendi daha küçük
en-boy oranı alt kümesini doğrular. Grok Imagine'ın kendi oran listesi vardır (`2:1`,
`20:9`, `19.5:9` ve bunların tersleri dâhil) ve yalnızca `1K`/`2K` çözünürlüklerini kabul eder;
eski Nano Banana ve Nano Banana 2 Lite, `resolution` geçersiz kılmalarını reddeder.
</Warning>

Krea 2 modelleri fal'ın yerel Krea yük şemasını kullanır. OpenClaw, Flux tarafından kullanılan
genel `image_size` / düzenleme uç noktası yükü yerine
`aspect_ratio`, `creativity` ve `image_style_references` gönderir. Model başvuruları şunlardır:

- `fal/krea/v2/medium/text-to-image`
- `fal/krea/v2/large/text-to-image`

Daha hızlı, etkileyici illüstrasyonlar, anime, resim ve sanatsal
stiller için Medium kullanın. Daha yavaş, fotogerçekçi, ham dokulu, film grenli ve ayrıntılı
görünümler için Large kullanın. Krea'nın varsayılanı `fal.creativity: "medium"` değeridir; desteklenen değerler
`raw`, `low`, `medium` ve `high` değerleridir.

Krea 2, fal'ın istek şemasında `image_size` değil, en-boy oranı sunar.
`aspectRatio` tercih edin; OpenClaw, `size` değerini desteklenen en yakın Krea en-boy oranıyla eşler
ve Krea için `resolution` değerini yok saymak yerine reddeder.

`output_format` sunan fal modellerinden PNG çıktısı istediğinizde
`outputFormat: "png"` kullanın. fal, OpenClaw'da açık bir şeffaf arka plan
denetimi bildirmez; bu nedenle `background: "transparent"`, fal modelleri için yok sayılan bir
geçersiz kılma olarak bildirilir.
Krea 2 uç noktaları fal üzerinden bir `output_format` istek alanı sunmaz; bu nedenle
OpenClaw, Krea isteklerinde `outputFormat` geçersiz kılmalarını reddeder.

Krea 2 Medium'u kullanmak için:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "fal/krea/v2/medium/text-to-image",
      },
    },
  },
}
```

## Video üretimi

Paketlenmiş `fal` video üretimi sağlayıcısının varsayılanı
`fal/fal-ai/minimax/video-01-live` değeridir.

| Yetenek         | Değer                                                              |
| --------------- | ------------------------------------------------------------------ |
| Modlar          | Metinden videoya, tek görüntü referansı, Seedance referanstan videoya |
| Çalışma zamanı  | Uzun süren işler için kuyruk destekli gönderme/durum/sonuç akışı    |
| Zaman aşımı     | Varsayılan olarak iş başına 20 dakika; durum her 5 saniyede bir sorgulanır |

<AccordionGroup>
  <Accordion title="Kullanılabilir video modelleri">
    **MiniMax (varsayılan):**

    - `fal/fal-ai/minimax/video-01-live`

    **HeyGen video-agent:**

    - `fal/fal-ai/heygen/v2/video-agent`

    **Kling ve Wan:**

    - `fal/fal-ai/kling-video/v2.1/master/text-to-video`
    - `fal/fal-ai/wan/v2.2-a14b/text-to-video`
    - `fal/fal-ai/wan/v2.2-a14b/image-to-video`

    **Seedance 2.0:**

    - `fal/bytedance/seedance-2.0/fast/text-to-video`
    - `fal/bytedance/seedance-2.0/fast/image-to-video`
    - `fal/bytedance/seedance-2.0/fast/reference-to-video`
    - `fal/bytedance/seedance-2.0/text-to-video`
    - `fal/bytedance/seedance-2.0/image-to-video`
    - `fal/bytedance/seedance-2.0/reference-to-video`

    MiniMax Live ve HeyGen istekleri yalnızca istemi ve isteğe bağlı
    tek bir referans görüntüyü gönderir; diğer geçersiz kılmalar iletilmez. Seedance modelleri
    `aspectRatio`, `size`, `resolution`, 4-15 saniyelik süreler ve
    bir ses açma/kapatma seçeneğini kabul eder.

  </Accordion>

  <Accordion title="Seedance 2.0 yapılandırma örneği">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/bytedance/seedance-2.0/fast/text-to-video",
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="Seedance 2.0 referanstan videoya yapılandırma örneği">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/bytedance/seedance-2.0/fast/reference-to-video",
          },
        },
      },
    }
    ```

    Referanstan videoya modu, paylaşılan `video_generate` `images`, `videos` ve `audioRefs`
    parametreleri aracılığıyla en fazla 9 görüntü, 3 video ve 3 ses referansını,
    toplamda en fazla 12 referans dosyası olacak şekilde kabul eder. Ses referansları, aynı istekte
    en az bir görüntü veya video referansı gerektirir.

  </Accordion>

  <Accordion title="HeyGen video-agent yapılandırma örneği">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/fal-ai/heygen/v2/video-agent",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## Müzik üretimi

Paketlenmiş `fal` plugin'i, paylaşılan
`music_generate` aracı için bir müzik üretimi sağlayıcısı da kaydeder.

| Yetenek          | Değer                                                                                                                    |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------ |
| Varsayılan model | `fal/fal-ai/minimax-music/v2.6`                                                                                          |
| Modeller         | `fal-ai/minimax-music/v2.6` (mp3), `fal-ai/ace-step/prompt-to-audio` (wav), `fal-ai/stable-audio-25/text-to-audio` (wav) |
| En fazla süre    | 240 saniye                                                                                                               |
| Çalışma zamanı   | Eşzamanlı istek ve ardından üretilen sesin indirilmesi                                                                   |

fal'ı varsayılan müzik sağlayıcısı olarak kullanın:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "fal/fal-ai/minimax-music/v2.6",
      },
    },
  },
}
```

`fal-ai/minimax-music/v2.6`, açık şarkı sözlerini ve enstrümantal modu destekler,
ancak ikisini aynı istekte desteklemez. ACE-Step ve Stable Audio,
istemden sese uç noktalarıdır; bu model ailelerini istediğinizde `model` geçersiz kılmasıyla
bunları seçin. ACE-Step, açık şarkı sözlerini reddeder; Stable Audio ise
hem şarkı sözlerini hem de enstrümantal modu reddeder.

<Tip>
Yukarıdaki tablolar ve açılır bölümler, paketlenmiş fal
sağlayıcısının özel olarak işlediği model ailelerini kapsar. Diğer fal görüntü uç noktası kimlikleri de
görüntü modeli olarak seçilebilir; bunlar Flux gibi işlenir (genel `image_size` yükü,
`/image-to-image` aracılığıyla bir referans görüntü).
</Tip>

## İlgili içerikler

<CardGroup cols={2}>
  <Card title="Görüntü üretimi" href="/tr/tools/image-generation" icon="image">
    Paylaşılan görüntü aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="Video üretimi" href="/tr/tools/video-generation" icon="video">
    Paylaşılan video aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="Müzik üretimi" href="/tr/tools/music-generation" icon="music">
    Paylaşılan müzik aracı parametreleri ve sağlayıcı seçimi.
  </Card>
  <Card title="Yapılandırma başvurusu" href="/tr/gateway/config-agents#agent-defaults" icon="gear">
    Görüntü, video ve müzik modeli seçimi dâhil agent varsayılanları.
  </Card>
</CardGroup>
