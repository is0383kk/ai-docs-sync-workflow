---
read_when:
    - Aracı üzerinden görüntü oluşturma veya düzenleme
    - Görüntü oluşturma sağlayıcılarını ve modellerini yapılandırma
    - image_generate aracı parametrelerini anlama
sidebarTitle: Image generation
summary: OpenAI, Google, fal, Microsoft Foundry, MiniMax, ComfyUI, DeepInfra, OpenRouter, LiteLLM, xAI ve Vydra genelinde image_generate aracılığıyla görüntüler oluşturun ve düzenleyin
title: Görsel oluşturma
x-i18n:
    generated_at: "2026-07-26T23:04:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9688b1bc649713d8ed345a69a28d20b36ecd768b6a6d28a2d6c022d65b081862
    source_path: tools/image-generation.md
    workflow: 16
---

`image_generate` aracı, yapılandırdığınız sağlayıcılar aracılığıyla görseller oluşturur ve düzenler.
Sohbet oturumlarında eşzamansız çalışır: OpenClaw bir arka plan görevi
kaydeder, görev kimliğini hemen döndürür ve sağlayıcı işlemi tamamladığında
ajanı uyandırır. Tamamlama ajanı, oturumun normal görünür yanıt modunu izler:
yapılandırıldığında nihai yanıtı otomatik olarak teslim eder veya oturum
mesaj aracını gerektiriyorsa `message(action="send")` kullanır. İstekte bulunan
oturum etkin değilse veya etkin uyandırma işlemi başarısız olursa OpenClaw,
sonucun kaybolmaması için oluşturulan görsellerle birlikte idempotent bir
doğrudan geri dönüş gönderir.

<Note>
Araç yalnızca en az bir görsel oluşturma sağlayıcısı kullanılabilir olduğunda
görünür. Ajanınızın araçlarında `image_generate` öğesini görmüyorsanız
`agents.defaults.mediaModels.image` yapılandırın, bir sağlayıcı API anahtarı ayarlayın
veya OpenAI ChatGPT/Codex OAuth ile oturum açın.
</Note>

## Hızlı başlangıç

<Steps>
  <Step title="Kimlik doğrulamayı yapılandırın">
    En az bir sağlayıcı için bir API anahtarı ayarlayın (örneğin `OPENAI_API_KEY`,
    `GEMINI_API_KEY`, `OPENROUTER_API_KEY`) veya OpenAI Codex OAuth ile oturum açın.
  </Step>
  <Step title="Varsayılan model seçin (isteğe bağlı)">
    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "openai/gpt-image-2",
            timeoutMs: 180_000,
          },
        },
      },
    }
    ```

    ChatGPT/Codex OAuth aynı `openai/gpt-image-2` model referansını kullanır. Bir
    `openai` OAuth profili yapılandırıldığında OpenClaw, önce
    `OPENAI_API_KEY` öğesini denemek yerine görsel isteklerini bu OAuth profili
    üzerinden yönlendirir. Açık `models.providers.openai` yapılandırması (API anahtarı,
    özel/Azure temel URL'si), doğrudan OpenAI Images API yolunu yeniden etkinleştirir.

  </Step>
  <Step title="Ajandan isteyin">
    _"Sevimli bir robot maskotun görselini oluştur."_

    Ajan `image_generate` öğesini otomatik olarak çağırır. Araç izin listesine
    ekleme gerekmez; bir sağlayıcı kullanılabilir olduğunda varsayılan olarak
    etkindir. Araç bir arka plan görevi kimliği döndürür, ardından tamamlama ajanı
    hazır olduğunda oluşturulan eki `message` aracı üzerinden gönderir.

  </Step>
</Steps>

<Warning>
LocalAI gibi OpenAI uyumlu LAN uç noktalarında özel
`models.providers.openai.baseUrl` değerini koruyun ve
`browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true` ile açıkça etkinleştirin. Özel ve
dahili görsel uç noktaları varsayılan olarak engellenmeye devam eder.
</Warning>

## Yaygın yollar

| Hedef                                                | Model referansı                                     | Kimlik doğrulama                       |
| ---------------------------------------------------- | -------------------------------------------------- | -------------------------------------- |
| API faturalandırmasıyla OpenAI görsel oluşturma      | `openai/gpt-image-2`                               | `OPENAI_API_KEY`                       |
| Codex abonelik kimlik doğrulamasıyla OpenAI görsel oluşturma | `openai/gpt-image-2`                               | OpenAI ChatGPT/Codex OAuth             |
| OpenAI şeffaf arka planlı PNG/WebP                   | `openai/gpt-image-1.5`                             | `OPENAI_API_KEY` veya OpenAI Codex OAuth |
| DeepInfra görsel oluşturma                           | `deepinfra/black-forest-labs/FLUX-1-schnell`       | `DEEPINFRA_API_KEY`                    |
| fal Krea 2 etkileyici/stil yönlendirmeli oluşturma   | `fal/krea/v2/medium/text-to-image`                 | `FAL_KEY`                              |
| OpenRouter görsel oluşturma                          | `openrouter/google/gemini-3.1-flash-image-preview` | `OPENROUTER_API_KEY`                   |
| LiteLLM görsel oluşturma                             | `litellm/gpt-image-2`                              | `LITELLM_API_KEY`                      |
| Microsoft Foundry MAI görsel oluşturma               | `microsoft-foundry/<deployment-name>`              | `AZURE_OPENAI_API_KEY` veya Entra ID     |
| Google Gemini görsel oluşturma                       | `google/gemini-3.1-flash-image`                    | `GEMINI_API_KEY` veya `GOOGLE_API_KEY`   |

Aynı araç, metinden görsel oluşturmayı ve referans görsel düzenlemeyi destekler.
Tek bir referans için `image`, birden fazla referans için
`images` kullanın. fal üzerindeki Krea 2 modellerinde bu referanslar,
düzenleme girdileri yerine stil referansları olarak gönderilir.
`quality`, `outputFormat` ve `background` gibi sağlayıcının
desteklediği çıktı ipuçları, kullanılabilir olduğunda iletilir; sağlayıcı destek
bildirmediğinde ise yok sayılmış olarak raporlanır. Paketlenmiş şeffaf arka plan
desteği OpenAI'a özgüdür; diğer sağlayıcılar, arka uçları bu şekilde çıktı
veriyorsa PNG alfa kanalını koruyabilir.

## Desteklenen sağlayıcılar

| Sağlayıcı         | Varsayılan model                        | Düzenleme desteği                  | Kimlik doğrulama                                       |
| ----------------- | --------------------------------------- | ---------------------------------- | ----------------------------------------------------- |
| ComfyUI           | `workflow`                              | Evet (1 görsel, iş akışıyla yapılandırılır) | `COMFY_API_KEY` veya bulut için `COMFY_CLOUD_API_KEY` |
| DeepInfra         | `black-forest-labs/FLUX-1-schnell`      | Evet (1 görsel)                    | `DEEPINFRA_API_KEY`                                   |
| fal               | `fal-ai/flux/dev`                       | Evet (modele özgü sınırlar)        | `FAL_KEY`                                             |
| Google            | `gemini-3.1-flash-image`                | Evet (en fazla 5 görsel)           | `GEMINI_API_KEY` veya `GOOGLE_API_KEY`                  |
| LiteLLM           | `gpt-image-2`                           | Evet (en fazla 5 girdi görseli)    | `LITELLM_API_KEY`                                     |
| Microsoft Foundry | `<deployment-name>`                     | Evet (yalnızca MAI-Image-2.5 modelleri) | `AZURE_OPENAI_API_KEY` veya Entra ID (`az login`) |
| MiniMax           | `image-01`                              | Evet (özne referansı)              | `MINIMAX_API_KEY` veya MiniMax OAuth (`minimax-portal`) |
| OpenAI            | `gpt-image-2`                           | Evet (en fazla 5 görsel)           | `OPENAI_API_KEY` veya OpenAI ChatGPT/Codex OAuth        |
| OpenRouter        | `google/gemini-3.1-flash-image-preview` | Evet (en fazla 5 girdi görseli)    | `OPENROUTER_API_KEY`                                  |
| Vydra             | `grok-imagine`                          | Hayır                              | `VYDRA_API_KEY`                                       |
| xAI               | `grok-imagine-image`                    | Evet (en fazla 3 görsel)           | `XAI_API_KEY`                                         |

Çalışma zamanında kullanılabilir sağlayıcıları ve modelleri incelemek için
`action: "list"` kullanın:

```text
/tool image_generate action=list
```

Geçerli oturumdaki etkin görsel oluşturma görevini incelemek için
`action: "status"` kullanın:

```text
/tool image_generate action=status
```

## Sağlayıcı yetenekleri

| Yetenek                | ComfyUI                 | DeepInfra | fal                                            | Google             | Microsoft Foundry | MiniMax                  | OpenAI             | Vydra | xAI                |
| --------------------- | ----------------------- | --------- | ---------------------------------------------- | ------------------ | ----------------- | ------------------------ | ------------------ | ----- | ------------------ |
| Oluşturma (azami sayı) | 1                       | 4         | 4                                              | 4                  | 1                 | 9                        | 4                  | 1     | 4                  |
| Düzenleme / referans   | 1 görsel (iş akışı)     | 1 görsel  | Flux: 1; GPT: 10; Krea stil ref.: 10; NB2: 14 | En fazla 5 görsel  | 1 görsel          | 1 görsel (özne ref.)     | En fazla 5 görsel  | -     | En fazla 3 görsel  |
| Boyut denetimi         | -                       | ✓         | ✓                                              | ✓                  | ✓                 | -                        | En fazla 4K        | -     | -                  |
| En-boy oranı           | -                       | -         | ✓                                              | ✓                  | -                 | ✓                        | -                  | -     | ✓                  |
| Çözünürlük (1K/2K/4K)  | -                       | -         | ✓                                              | ✓                  | -                 | -                        | -                  | -     | 1K, 2K             |

## Araç parametreleri

<ParamField path="prompt" type="string" required>
  Görsel oluşturma istemi. `action: "generate"` için gereklidir.
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  Etkin oturum görevini incelemek için `"status"`, çalışma zamanında
  kullanılabilir sağlayıcıları ve modelleri incelemek için `"list"` kullanın.
</ParamField>
<ParamField path="model" type="string">
  Sağlayıcı/model geçersiz kılma değeri (ör. `openai/gpt-image-2`). Şeffaf OpenAI
  arka planları için `openai/gpt-image-1.5` kullanın.
</ParamField>
<ParamField path="image" type="string">
  Düzenleme modu için tek bir referans görsel yolu veya URL'si.
</ParamField>
<ParamField path="images" type="string[]">
  Düzenleme modu veya stil referansı modelleri için birden fazla referans görsel
  (paylaşılan araç üzerinden en fazla 14; sağlayıcıya özgü sınırlar geçerliliğini korur).
</ParamField>
<ParamField path="size" type="string">
  Boyut ipucu: `1024x1024`, `1536x1024`, `1024x1536`, `2048x2048`, `3840x2160`.
</ParamField>
<ParamField path="aspectRatio" type="string">
  En-boy oranı: `1:1`, `2:1`, `20:9`, `19.5:9`, `2:3`, `3:2`, `2.35:1`, `3:4`,
  `4:3`, `4:5`, `5:4`, `9:16`, `9:19.5`, `9:20`, `16:9`, `21:9`, `1:2`, `4:1`,
  `1:4`, `8:1`, `1:8`. Sağlayıcılar kendi modele özgü alt kümelerini doğrular.
</ParamField>
<ParamField path="resolution" type='"1K" | "2K" | "4K"'>Çözünürlük ipucu.</ParamField>
<ParamField path="quality" type='"low" | "medium" | "high" | "auto"'>
  Sağlayıcı desteklediğinde kalite ipucu.
</ParamField>
<ParamField path="outputFormat" type='"png" | "jpeg" | "webp"'>
  Sağlayıcı desteklediğinde çıktı biçimi ipucu.
</ParamField>
<ParamField path="background" type='"transparent" | "opaque" | "auto"'>
  Sağlayıcı desteklediğinde arka plan ipucu. Şeffaflığı destekleyen sağlayıcılar
  için `transparent` ile `outputFormat: "png"` veya `"webp"` kullanın.
</ParamField>
<ParamField path="count" type="number">Oluşturulacak görsel sayısı (1-4).</ParamField>
<ParamField path="timeoutMs" type="number">
  Milisaniye cinsinden isteğe bağlı sağlayıcı isteği zaman aşımı. Codex,
  dinamik araçlar üzerinden `image_generate` çağırdığında bu çağrı başına
  değer yine yapılandırılmış varsayılanı geçersiz kılar ve 600000 ms ile sınırlandırılır.
</ParamField>
<ParamField path="filename" type="string">Çıktı dosya adı ipucu.</ParamField>
<ParamField path="openai" type="object">
  Yalnızca OpenAI'a özgü ipuçları: `background`, `moderation`, `outputCompression` ve `user`.
</ParamField>
<ParamField path="fal.creativity" type='"raw" | "low" | "medium" | "high"'>
  fal Krea 2 yaratıcılık denetimi. Varsayılan değer `medium`.
</ParamField>

<Note>
Tüm sağlayıcılar tüm parametreleri desteklemez. Bir geri dönüş sağlayıcısı,
tam olarak istenen seçenek yerine yakın bir geometri seçeneğini desteklediğinde
OpenClaw, göndermeden önce en yakın desteklenen boyuta, en-boy oranına veya
çözünürlüğe yeniden eşler. Desteklenmeyen çıktı ipuçları, destek bildirmeyen
sağlayıcılar için kaldırılır ve araç sonucunda raporlanır. Araç sonuçları
uygulanan ayarları bildirir; `details.normalization`, istenen değerden uygulanan
değere yapılan dönüşümleri içerir.
</Note>

## Yapılandırma

### Model seçimi

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-2",
        timeoutMs: 180_000,
        fallbacks: [
          "openrouter/google/gemini-3.1-flash-image-preview",
          "google/gemini-3.1-flash-image",
          "fal/fal-ai/flux/dev",
        ],
      },
    },
  },
}
```

### Sağlayıcı seçim sırası

OpenClaw sağlayıcıları şu sırayla dener:

1. Araç çağrısındaki **`model` parametresi** (agent bir tane belirtirse).
2. Yapılandırmadaki **`imageGenerationModel.primary`**.
3. Sırasıyla **`imageGenerationModel.fallbacks`**.
4. **Otomatik algılama** - yalnızca kimlik doğrulama destekli sağlayıcı varsayılanları:
   - önce geçerli varsayılan sağlayıcı;
   - ardından kalan kayıtlı görüntü oluşturma sağlayıcıları, sağlayıcı kimliği sırasıyla.

Bir sağlayıcı başarısız olursa (kimlik doğrulama hatası, hız sınırı vb.), yapılandırılmış bir sonraki
aday otomatik olarak denenir. Tümü başarısız olursa hata, her denemenin
ayrıntılarını içerir.

<AccordionGroup>
  <Accordion title="Çağrı başına model geçersiz kılmaları kesindir">
    Çağrı başına `model` geçersiz kılması yalnızca ilgili sağlayıcıyı/modeli dener ve
    yapılandırılmış birincil/yedek ya da otomatik algılanan sağlayıcılarla devam etmez.
  </Accordion>
  <Accordion title="Otomatik algılama kimlik doğrulamanın farkındadır">
    Bir sağlayıcı varsayılanı, yalnızca OpenClaw ilgili sağlayıcıda gerçekten
    kimlik doğrulaması yapabiliyorsa aday listesine girer. Kimliği doğrulanmış
    sağlayıcılar arasında otomatik yedekleme her zaman etkindir; çağrı başına `model` belirleyici olmaya devam eder.
  </Accordion>
  <Accordion title="Zaman aşımları">
    Yavaş görüntü arka uçları için `agents.defaults.mediaModels.image.timeoutMs` ayarlayın.
    Çağrı başına `timeoutMs` araç parametresi, yapılandırılmış
    varsayılanı geçersiz kılar; yapılandırılmış varsayılanlar da plugin tarafından yazılmış sağlayıcı
    varsayılanlarını geçersiz kılar. Google ve OpenRouter tarafından barındırılan görüntü sağlayıcıları 180 saniyelik
    varsayılanlar kullanır; Microsoft Foundry MAI, xAI ve Azure OpenAI görüntü oluşturma ise
    600 saniye kullanır. Codex dinamik araç çağrıları 120 saniyelik `image_generate`
    köprü varsayılanını kullanır ve yapılandırıldığında aynı zaman aşımı bütçesine uyar; bu süre
    OpenClaw'ın 600000 ms dinamik araç köprüsü üst sınırıyla sınırlıdır.
  </Accordion>
  <Accordion title="Çalışma zamanında inceleme">
    Şu anda kayıtlı sağlayıcıları, bunların varsayılan modellerini
    ve kimlik doğrulama ortam değişkeni ipuçlarını incelemek için `action: "list"` kullanın.
  </Accordion>
</AccordionGroup>

### Görüntü düzenleme

OpenAI, OpenRouter, Google, DeepInfra, fal, Microsoft Foundry, MiniMax,
ComfyUI ve xAI, referans görüntülerini düzenlemeyi destekler. fal üzerindeki Krea 2 modelleri,
aynı `image` / `images` alanlarını düzenleme girdileri yerine stil referansı olarak
kullanır. Bir referans görüntüsü yolu veya URL'si iletin:

```text
"Bu fotoğrafın suluboya sürümünü oluştur" + image: "/path/to/photo.jpg"
```

OpenAI, OpenRouter ve Google, `images` parametresi aracılığıyla en fazla 5 referans görüntüsünü;
xAI ise en fazla 3 görüntüyü destekler. fal; Flux görüntüden görüntüye dönüştürme için 1 referans görüntüsünü,
GPT Image 2 düzenlemeleri için en fazla 10 görüntüyü, Krea 2 için en fazla 10 stil referansını
ve Nano Banana 2 düzenlemeleri için en fazla 14 görüntüyü destekler. Microsoft Foundry, MiniMax
ve ComfyUI 1 görüntüyü destekler.

## Sağlayıcılara ayrıntılı bakış

<AccordionGroup>
  <Accordion title="OpenAI gpt-image-2 (ve gpt-image-1.5)">
    OpenAI görüntü oluşturma varsayılan olarak `openai/gpt-image-2` kullanır. Bir
    `openai` OAuth profili yapılandırılmışsa OpenClaw, Codex abonelik sohbet modellerinin
    kullandığı aynı OAuth profilini yeniden kullanır ve görüntü isteğini
    Codex Responses arka ucu üzerinden gönderir. `https://chatgpt.com/backend-api` gibi eski Codex temel
    URL'leri, görüntü istekleri için `https://chatgpt.com/backend-api/codex` biçimine
    standartlaştırılır. OpenClaw bu istek için sessizce
    `OPENAI_API_KEY` seçeneğine **geri dönmez**; doğrudan OpenAI Images API yönlendirmesini zorlamak için
    `models.providers.openai` öğesini bir API anahtarı, özel temel URL
    veya Azure uç noktasıyla açıkça yapılandırın.

    `openai/gpt-image-1.5`, `openai/gpt-image-1` ve
    `openai/gpt-image-1-mini` modelleri yine de açıkça seçilebilir. Şeffaf arka planlı
    PNG/WebP çıktısı için `gpt-image-1.5` kullanın; geçerli
    `gpt-image-2` API'si `background: "transparent"` değerini reddeder.

    `gpt-image-2`, aynı `image_generate` aracı üzerinden hem metinden görüntü oluşturmayı hem de
    referans görüntüsü düzenlemeyi destekler.
    OpenClaw; `prompt`, `count`, `size`, `quality`, `outputFormat`
    ve referans görüntülerini OpenAI'a iletir. OpenAI, `aspectRatio` veya
    `resolution` değerlerini doğrudan **almaz**; mümkün olduğunda OpenClaw bunları
    desteklenen bir `size` değerine eşler, aksi durumda araç bunları yok sayılan
    geçersiz kılmalar olarak bildirir.

    OpenAI'a özgü seçenekler `openai` nesnesi altında bulunur:

    ```json
    {
      "quality": "low",
      "outputFormat": "jpeg",
      "openai": {
        "background": "opaque",
        "moderation": "low",
        "outputCompression": 60,
        "user": "end-user-42"
      }
    }
    ```

    `openai.background`; `transparent`, `opaque` veya `auto` kabul eder;
    şeffaf çıktılar için `outputFormat` değerinin `png` veya `webp` olması ve
    şeffaflığı destekleyen bir OpenAI görüntü modeli gerekir. OpenClaw, varsayılan
    `gpt-image-2` şeffaf arka plan isteklerini `gpt-image-1.5` modeline yönlendirir.
    `openai.outputCompression`, JPEG/WebP çıktılarına uygulanır ve PNG çıktıları için
    yok sayılır.

    Üst düzey `background` ipucu sağlayıcıdan bağımsızdır ve OpenAI sağlayıcısı
    seçildiğinde şu anda aynı OpenAI `background` istek alanına eşlenir.
    Arka plan desteği bildirmeyen sağlayıcılar, desteklenmeyen parametreyi almak yerine
    bunu `ignoredOverrides` içinde döndürür.

    OpenAI görüntü oluşturmayı `api.openai.com` yerine bir Azure OpenAI dağıtımı
    üzerinden yönlendirmek için
    [Azure OpenAI uç noktaları](/tr/providers/openai#azure-openai-endpoints) bölümüne bakın.

  </Accordion>
  <Accordion title="Microsoft Foundry MAI görüntü modelleri">
    Microsoft Foundry görüntü oluşturma, `microsoft-foundry/` sağlayıcı öneki altında
    dağıtılmış MAI görüntü dağıtımı adlarını kullanır. MAI API, dağıtım adınızı
    `model` alanında beklediğinden sağlayıcı düzeyinde varsayılan
    model yoktur:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "microsoft-foundry/<deployment-name>",
            timeoutMs: 600_000,
          },
        },
      },
    }
    ```

    Sağlayıcı, OpenAI Images API'yi değil Microsoft Foundry'nin MAI API'sini kullanır:

    - Oluşturma uç noktası: `/mai/v1/images/generations`
    - Düzenleme uç noktası: `/mai/v1/images/edits`
    - Kimlik doğrulama: `AZURE_OPENAI_API_KEY` / sağlayıcı API anahtarı veya `az login` üzerinden Entra ID
    - Çıktı: bir PNG görüntüsü
    - Boyut: varsayılan `1024x1024`; genişlik ve yüksekliğin her biri en az 768 px olmalı,
      toplam piksel sayısı ise en fazla 1,048,576 olmalıdır
    - Düzenlemeler: yalnızca `MAI-Image-2.5-Flash` ve `MAI-Image-2.5` dağıtımları tarafından desteklenen
      bir PNG veya JPEG referans görüntüsü

    Yalnızca istemle oluşturma, sadece Foundry uç noktası yapılandırılmışken özel bir dağıtım adı
    kullanabilir. Özel dağıtım adlarıyla yapılan düzenlemelerde, OpenClaw'ın dağıtımın
    `MAI-Image-2.5-Flash` veya `MAI-Image-2.5` tarafından desteklendiğini doğrulayabilmesi için
    ilk katılım/model meta verileri gerekir.

    Geçerli MAI görüntü modelleri `MAI-Image-2.5-Flash`, `MAI-Image-2.5`,
    `MAI-Image-2e` ve `MAI-Image-2` modelleridir. Kurulum
    ve sohbet modeli davranışı için [Microsoft Foundry plugin](/tr/plugins/reference/microsoft-foundry) bölümüne bakın.

  </Accordion>
  <Accordion title="OpenRouter görüntü modelleri">
    OpenRouter görüntü oluşturma aynı `OPENROUTER_API_KEY` öğesini kullanır ve
    OpenRouter'ın sohbet tamamlama görüntü API'si üzerinden yönlendirilir.
    OpenRouter görüntü modellerini `openrouter/` önekiyle seçin:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "openrouter/google/gemini-3.1-flash-image-preview",
          },
        },
      },
    }
    ```

    OpenClaw; `prompt`, `count`, referans görüntüleri ve
    Gemini uyumlu `aspectRatio` / `resolution` ipuçlarını OpenRouter'a iletir.
    Geçerli yerleşik OpenRouter görüntü modeli kısayolları arasında
    `google/gemini-3.1-flash-image`,
    `google/gemini-3-pro-image` ve `openai/gpt-5.4-image-2` bulunur. Yapılandırılmış plugin'inizin
    neleri sunduğunu görmek için `action: "list"` kullanın.

  </Accordion>
  <Accordion title="fal Krea 2">
    fal üzerindeki Krea 2 modelleri, Flux tarafından kullanılan genel
    `image_size` şeması yerine fal'ın yerel Krea şemasını kullanır. OpenClaw şunları gönderir:

    - En-boy oranı ipuçları için `aspect_ratio`
    - Varsayılan değeri `medium` olan `creativity`
    - `image` veya `images` sağlandığında `image_style_references`

    Daha hızlı ve etkileyici illüstrasyonlar için Krea 2 Medium'u; daha yavaş,
    daha ayrıntılı, fotogerçekçi ve dokulu görünümler için Krea 2 Large'ı seçin:

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

    Krea 2 şu anda istek başına bir görüntü döndürür. Krea için `aspectRatio` tercih edin;
    OpenClaw, `size` değerini desteklenen en yakın Krea en-boy oranına eşler ve
    kaldırmak yerine Krea için `resolution` değerini reddeder. Yerel bir Krea yaratıcılık düzeyi
    istediğinizde `fal.creativity` kullanın:

    ```json
    {
      "model": "fal/krea/v2/medium/text-to-image",
      "prompt": "Risograf dokulu bir siber fanzin portresi",
      "aspectRatio": "9:16",
      "fal": {
        "creativity": "high"
      }
    }
    ```

  </Accordion>
  <Accordion title="MiniMax çift kimlik doğrulama">
    MiniMax görüntü oluşturma, paketlenmiş MiniMax
    kimlik doğrulama yollarının ikisi üzerinden de kullanılabilir:

    - API anahtarı kurulumları için `minimax/image-01`
    - OAuth kurulumları için `minimax-portal/image-01`

  </Accordion>
  <Accordion title="xAI grok-imagine-image">
    Paketlenmiş xAI sağlayıcısı, yalnızca istem içeren istekler için `/v1/images/generations`;
    `image` veya `images` mevcut olduğunda ise `/v1/images/edits` kullanır.

    - Modeller: `xai/grok-imagine-image`, `xai/grok-imagine-image-quality`
    - Sayı: en fazla 4
    - Referanslar: bir `image` veya en fazla üç `images`
    - En-boy oranları: `1:1`, `16:9`, `9:16`, `4:3`, `3:4`, `3:2`, `2:3`, `2:1`,
      `1:2`, `19.5:9`, `9:19.5`, `20:9`, `9:20`
    - Çözünürlükler: `1K`, `2K`
    - Çıktılar: OpenClaw tarafından yönetilen görüntü ekleri olarak döndürülür

    OpenClaw, bu denetimler sağlayıcılar arası ortak `image_generate` sözleşmesinde bulunana kadar
    xAI'a özgü `quality`, `mask`,
    `user` veya `auto` en-boy oranını kasıtlı olarak kullanıma sunmaz.

  </Accordion>
</AccordionGroup>

## Örnekler

<Tabs>
  <Tab title="Oluştur (4K yatay)">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="OpenClaw görüntü oluşturma için sade bir editoryal poster" size=3840x2160 count=1
```
  </Tab>
  <Tab title="Oluştur (şeffaf PNG)">
```text
/tool image_generate action=generate model=openai/gpt-image-1.5 prompt="Şeffaf arka plan üzerinde basit bir kırmızı daire çıkartması" outputFormat=png background=transparent
```

Eşdeğer CLI:

```bash
openclaw infer image generate \
  --model openai/gpt-image-1.5 \
  --output-format png \
  --background transparent \
  --prompt "Şeffaf arka plan üzerinde basit bir kırmızı daire çıkartması" \
  --json
```

  </Tab>
  <Tab title="Oluştur (OpenAI düşük kalite)">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="Sakin bir üretkenlik uygulaması için düşük maliyetli taslak poster" quality=low openai='{"moderation":"low"}'
```

Eşdeğer CLI:

```bash
openclaw infer image generate \
  --model openai/gpt-image-2 \
  --quality low \
  --openai-moderation low \
  --prompt "Sakin bir üretkenlik uygulaması için düşük maliyetli taslak poster" \
  --json
```

  </Tab>
  <Tab title="Oluştur (iki kare)">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="Sakin bir üretkenlik uygulaması simgesi için iki görsel yaklaşım" size=1024x1024 count=2
```
  </Tab>
  <Tab title="Düzenle (tek referans)">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="Konuyu koru, arka planı aydınlık bir stüdyo düzeniyle değiştir" image=/path/to/reference.png size=1024x1536
```
  </Tab>
  <Tab title="Düzenle (birden fazla referans)">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="İlk görüntüdeki karakter kimliğini ikinci görüntüdeki renk paletiyle birleştir" images='["/path/to/character.png","/path/to/palette.jpg"]' size=1536x1024
```
  </Tab>
  <Tab title="Krea stil referansları">
```text
/tool image_generate action=generate model=fal/krea/v2/medium/text-to-image prompt="Bu renk paletini ve baskı dokusunu kullanan etkileyici bir editoryal portre" images='["/path/to/palette.png","/path/to/texture.jpg"]' aspectRatio=9:16 fal='{"creativity":"high"}'
```
  </Tab>
</Tabs>

Aynı `--output-format`, `--background`, `--quality` ve
`--openai-moderation` bayrakları `openclaw infer image edit` üzerinde de kullanılabilir;
`--openai-background`, OpenAI'ye özgü bir diğer ad olarak kalır. OpenAI dışındaki
paketle gelen sağlayıcılar şu anda açık arka plan denetimi bildirmediğinden,
`background: "transparent"` bunlar için yok sayılmış olarak bildirilir.

## İlgili

- [Araçlara genel bakış](/tr/tools) - kullanılabilir tüm ajan araçları
- [ComfyUI](/tr/providers/comfy) - yerel ComfyUI ve Comfy Cloud iş akışı kurulumu
- [fal](/tr/providers/fal) - fal görüntü ve video sağlayıcısı kurulumu
- [Google (Gemini)](/tr/providers/google) - Gemini görüntü sağlayıcısı kurulumu
- [Microsoft Foundry plugin'i](/tr/plugins/reference/microsoft-foundry) - Microsoft Foundry sohbet ve MAI görüntü kurulumu
- [MiniMax](/tr/providers/minimax) - MiniMax görüntü sağlayıcısı kurulumu
- [OpenAI](/tr/providers/openai) - OpenAI Images sağlayıcısı kurulumu
- [Vydra](/tr/providers/vydra) - Vydra görüntü, video ve konuşma kurulumu
- [xAI](/tr/providers/xai) - Grok görüntü, video, arama, kod yürütme ve TTS kurulumu
- [Yapılandırma referansı](/tr/gateway/config-agents#agent-defaults) - `imageGenerationModel` yapılandırması
- [Modeller](/tr/concepts/models) - model yapılandırması ve yük devretme
