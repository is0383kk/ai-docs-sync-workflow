---
read_when:
    - microsoft-foundry Pluginini kuruyor, yapılandırıyor veya denetliyorsunuz
summary: OpenClaw'a Microsoft Foundry model sağlayıcısı desteği ekler.
title: Microsoft Foundry Plugin'i
x-i18n:
    generated_at: "2026-07-27T00:11:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f2ea554ce16cffeb4cc315e53d986d6f07b5e113fbb844c61c6575f19f8ad291
    source_path: plugins/reference/microsoft-foundry.md
    workflow: 16
---

# Microsoft Foundry plugin'i

OpenClaw'a Microsoft Foundry model sağlayıcısı desteği ekler.

## Dağıtım

- Paket: `@openclaw/microsoft-foundry`
- Kurulum yolu: OpenClaw'a dahildir

## Yüzey

sağlayıcılar: `microsoft-foundry`; sözleşmeler: `imageGenerationProviders`

<!-- openclaw-plugin-reference:manual-start -->

- Görüntü oluşturma sağlayıcısı: `microsoft-foundry`

## Gereksinimler

- Dağıtımları olan bir Microsoft Foundry veya Azure AI Foundry kaynağı.
- `AZURE_OPENAI_API_KEY` üzerinden API anahtarıyla kimlik doğrulaması veya yapılandırılmış bir sağlayıcı API anahtarı.
- Entra ID kimlik doğrulaması için Azure CLI'ı yükleyin ve ilk katılımdan önce
  `az login` komutunu çalıştırın. OpenClaw, Microsoft Foundry çalışma zamanı tokenlarını
  `az account get-access-token` üzerinden yeniler.

## Sohbet modelleri

Microsoft Foundry sohbet dağıtımları, `microsoft-foundry/<deployment-name>` sağlayıcı model referansını
kullanır. İlk katılım, Azure CLI ile Foundry kaynaklarını ve dağıtımlarını keşfeder,
ardından seçilen dağıtım adını model yapılandırmasına yazar.

OpenClaw, desteklenen OpenAI uyumlu sohbet API'leri için Foundry
`/openai/v1` uç noktasını kullanır:

- GPT, `o*`, `computer-use-preview` ve DeepSeek-V4 model aileleri varsayılan olarak
  `openai-responses` kullanır.
- MAI-DS-R1 ve diğer sohbet tamamlama dağıtımları, desteklenen bir API açıkça yapılandırılmadığı
  sürece `openai-completions` kullanır.
- MAI-DS-R1'in akıl yürütme özelliği `reasoning_effort` üzerinden değil,
  akıl yürütme içeriği üzerinden kaydedilir. Bağlam ve çıktı tokenı meta verileri
  163,840 tokendır.

Microsoft Foundry'deki Anthropic Claude dağıtımları, OpenAI uyumlu
`/openai/v1` biçimini değil, Anthropic Messages API biçimini kullanır. Microsoft
Foundry plugin'i yerel bir Anthropic çalışma zamanı kazanana kadar bunları özel bir
`anthropic-messages` sağlayıcısı olarak yapılandırın. Foundry dağıtım adı Claude model
kimliğinden farklı olduğunda model girdisinde `params.canonicalModelId` değerini ayarlayın;
böylece OpenClaw modele özgü bağlantı sözleşmelerini uygulayabilir,
`/think off` değerini doğru şekilde eşleyebilir ve imzalı düşünmeyi güvenle
koruyabilir.

## MAI görüntü oluşturma

Plugin, mevcut Microsoft AI görüntü modelleriyle `image_generate` için
`microsoft-foundry` kaydeder:

- `MAI-Image-2.5-Flash`
- `MAI-Image-2.5`
- `MAI-Image-2e`
- `MAI-Image-2`

Model referansı olarak dağıtılmış bir MAI görüntü dağıtımının adını kullanın. MAI API,
isteğin `model` alanında dağıtım adınızı gerektirdiğinden sağlayıcı
varsayılan bir görüntü modeli bildirmez:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "microsoft-foundry/<deployment-name>",
        timeoutMs: 600000,
      },
    },
  },
}
```

Yalnızca istem içeren oluşturma işlemleri Microsoft Foundry'nin MAI oluşturma uç
noktasını çağırır: `/mai/v1/images/generations`. Referans görüntü düzenlemeleri
`/mai/v1/images/edits` uç noktasını çağırır ve `MAI-Image-2.5-Flash` ile
`MAI-Image-2.5` dağıtımlarıyla sınırlıdır.

Yalnızca istem içeren oluşturma işlemlerinde, sadece Foundry uç noktası yapılandırılarak
özel bir dağıtım adı kullanılabilir. Özel bir dağıtım adıyla görüntü düzenlemek için
dağıtımı ilk katılım üzerinden seçin veya OpenClaw'ın dağıtımın
`MAI-Image-2.5-Flash` ya da `MAI-Image-2.5` tarafından desteklendiğini
doğrulayabilmesi için model meta verilerini ekleyin.

MAI görüntü kısıtlamaları:

- Çıktı: istek başına bir PNG görüntüsü.
- Boyut: varsayılan `1024x1024`; hem genişlik hem de yükseklik en az 768 px olmalıdır.
- Toplam piksel: genişlik × yükseklik en fazla 1,048,576 olmalıdır.
- Düzenlemeler: bir PNG veya JPEG girdi görüntüsü.
- `aspectRatio`, `resolution`, `quality`,
  `background` ve PNG olmayan `outputFormat` gibi desteklenmeyen ortak ipuçları Microsoft Foundry'ye gönderilmez.

## Sorun giderme

- `az: command not found`: Azure CLI'ı yükleyin veya API anahtarıyla kimlik doğrulamasını kullanın.
- `Microsoft Foundry endpoint missing for MAI image generation`: ilk katılım üzerinden bir
  Foundry dağıtımı seçin veya `models.providers.microsoft-foundry.baseUrl` ekleyin.
- `supports MAI image deployments only`: seçilen görüntü modeli MAI olmayan bir
  dağıtımı gösteriyor. `image_generate` için dağıtılmış bir MAI görüntü modeli kullanın.

<!-- openclaw-plugin-reference:manual-end -->
