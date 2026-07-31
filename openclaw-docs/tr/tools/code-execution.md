---
read_when:
    - code_execution özelliğini etkinleştirmek veya yapılandırmak istiyorsunuz
    - Yerel kabuk erişimi olmadan uzaktan analiz yapmak istiyorsunuz
    - x_search veya web_search'ü uzaktan Python analiziyle birleştirmek istiyorsunuz
summary: 'code_execution: xAI ile korumalı alanda uzaktan Python analizi çalıştırın'
title: Kod yürütme
x-i18n:
    generated_at: "2026-07-26T23:03:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1ab391daed9154f113535e6d241c45d5c08c22abdc012148a9f0f2ae5ec548b3
    source_path: tools/code-execution.md
    workflow: 16
---

`code_execution`, xAI'nin Responses API'sinde korumalı alanda uzaktan Python analizi çalıştırır
(`https://api.x.ai/v1/responses`, `x_search` tarafından kullanılan uç noktayla aynı). Paketle birlikte gelen `xai` Plugin'i tarafından `tools` sözleşmesi kapsamında kaydedilir.

<Warning>
  `code_execution`, xAI sunucularında çalışır. xAI, modelin giriş ve çıkış tokenlerine
  ek olarak her 1.000 araç çağrısı için $5 ücret alır.
</Warning>

| Özellik            | Değer                                                                             |
| ------------------ | --------------------------------------------------------------------------------- |
| Araç adı           | `code_execution`                                                                  |
| Sağlayıcı Plugin'i | `xai` (paketle birlikte gelir, `enabledByDefault: true`)                                         |
| Kimlik doğrulama   | xAI kimlik doğrulama profili, `XAI_API_KEY` veya `plugins.entries.xai.config.webSearch.apiKey` |
| Varsayılan model   | `grok-4.3`                                                                        |
| Varsayılan zaman aşımı | 30 saniye                                                                        |
| Varsayılan `maxTurns` | ayarlanmamış (xAI kendi dahili sınırını uygular)                                        |

Hesaplamalar, tablolama, hızlı istatistikler ve grafik tarzı analizler için,
`x_search` veya `web_search` tarafından döndürülen veriler dâhil olmak üzere kullanın. Yerel
dosyalara, kabuğunuza, deponuza veya eşleştirilmiş cihazlara erişimi yoktur ve
çağrılar arasında durumu kalıcı hâle getirmez; bu nedenle her çağrıyı bir not
defteri oturumu değil, geçici bir analiz olarak değerlendirin. Güncel X verileri için önce
[`x_search`](/tr/tools/web#x_search) çalıştırın ve sonucu aktarın.

Yerel yürütme için bunun yerine [`exec`](/tr/tools/exec) kullanın.

## Kurulum

<Steps>
  <Step title="xAI kimlik bilgilerini sağlayın">
    OAuth, uygun bir SuperGrok veya X Premium aboneliği gerektirir
    (cihaz koduyla doğrulama kullandığından localhost geri çağrısı olmadan uzak
    ana makinelerden çalışır):

    ```bash
    openclaw models auth login --provider xai --method oauth
    ```

    Yeni bir kurulum sırasında aynı seçenek ilk katılım akışında da kullanılabilir:

    ```bash
    openclaw onboard --install-daemon --auth-choice xai-oauth
    ```

    Veya bir API anahtarı:

    ```bash
    openclaw models auth login --provider xai --method api-key
    export XAI_API_KEY=xai-...
    ```

    Ya da yapılandırma aracılığıyla:

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              webSearch: {
                apiKey: "xai-...",
              },
            },
          },
        },
      },
    }
    ```

    Bu üç seçeneğin tümü `x_search` ve Grok `web_search` özelliklerini de çalıştırır.

  </Step>

  <Step title="code_execution'ı etkinleştirin ve ayarlayın">
    `enabled` belirtilmediğinde `code_execution`, yalnızca etkin
    modelin sağlayıcısı `xai` olduğunda ve xAI kimlik bilgileri çözümlendiğinde kullanıma sunulur. Sağlayıcısı
    xAI olmadığı bilinen etkin bir modelde, sağlayıcılar arası kullanımı etkinleştirmek için
    `plugins.entries.xai.config.codeExecution.enabled` değerini `true` olarak ayarlayın.
    Etkin model sağlayıcısı eksikse veya çözümlenemiyorsa araç gizli kalır. Tüm
    sağlayıcılar için devre dışı bırakmak üzere `enabled` değerini `false` olarak ayarlayın.
    xAI kimlik bilgileri her zaman gereklidir.

    Modeli, tur sınırını veya zaman aşımını geçersiz kılmak için aynı bloğu kullanın:

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              codeExecution: {
                enabled: true, // xAI olmadığı bilinen bir model sağlayıcısı için gereklidir
                model: "grok-4.3", // varsayılan xAI kod yürütme modelini geçersiz kıl
                maxTurns: 2,            // dahili araç turları için isteğe bağlı sınır
                timeoutSeconds: 30,     // istek zaman aşımı (varsayılan: 30)
              },
            },
          },
        },
      },
    }
    ```

  </Step>

  <Step title="Gateway'i yeniden başlatın">
    ```bash
    openclaw gateway restart
    ```

    xAI Plugin'i yeniden kaydolduktan ve yukarıdaki sağlayıcı, etkinleştirme ve kimlik doğrulama
    denetimleri geçildikten sonra `code_execution`, ajanın araç listesinde görünür.

  </Step>
</Steps>

## Kullanımı

Analiz amacını açıkça belirtin; araç tek bir `task` parametresi alır,
bu nedenle isteğin tamamını ve satır içi verileri tek bir istemde gönderin:

```text
Bu sayıların 7 günlük hareketli ortalamasını hesaplamak için code_execution kullan: ...
```

```text
Bu hafta OpenClaw'dan bahseden gönderileri bulmak için x_search kullan, ardından bunları güne göre saymak için code_execution kullan.
```

```text
En güncel yapay zekâ kıyaslama sayılarını toplamak için web_search kullan, ardından yüzdelik değişimleri karşılaştırmak için code_execution kullan.
```

## Hatalar

Kimlik doğrulama olmadan araç, oluşturulan bir istisna yerine yapılandırılmış
bir JSON hatası döndürür; böylece ajan kendi kendine düzeltebilir:

```json
{
  "error": "missing_xai_api_key",
  "message": "code_execution için xAI kimlik bilgileri gerekir. Grok ile oturum açmak için `openclaw onboard --auth-choice xai-oauth` komutunu çalıştırın, `openclaw onboard --auth-choice xai-api-key` komutunu çalıştırın, Gateway ortamında `XAI_API_KEY` değişkenini ayarlayın veya `plugins.entries.xai.config.webSearch.apiKey` değerini yapılandırın.",
  "docs": "https://docs.openclaw.ai/tools/code-execution"
}
```

## İlgili

<CardGroup cols={2}>
  <Card title="Exec aracı" href="/tr/tools/exec" icon="terminal">
    Makinenizde veya eşleştirilmiş Node üzerinde yerel kabuk yürütme.
  </Card>
  <Card title="Exec onayları" href="/tr/tools/exec-approvals" icon="shield">
    Kabuk yürütme için izin verme/reddetme ilkesi.
  </Card>
  <Card title="Web araçları" href="/tr/tools/web" icon="globe">
    `web_search`, `x_search` ve `web_fetch`.
  </Card>
  <Card title="xAI sağlayıcısı" href="/tr/providers/xai" icon="microchip">
    Grok modelleri, web/X araması ve kod yürütme yapılandırması.
  </Card>
</CardGroup>
