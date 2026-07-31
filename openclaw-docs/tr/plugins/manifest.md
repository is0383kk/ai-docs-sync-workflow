---
read_when:
    - Bir OpenClaw Plugin'i oluşturuyorsunuz
    - Bir Plugin yapılandırma şeması yayımlamanız veya Plugin doğrulama hatalarını ayıklamanız gerekiyor
summary: Plugin manifesti + JSON şeması gereksinimleri (katı yapılandırma doğrulaması)
title: Plugin manifestosu
x-i18n:
    generated_at: "2026-07-26T22:52:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 244e5c8265ff79b0ff6e8f4b60c9635cccc3ba66093cecab458676beb9578264
    source_path: plugins/manifest.md
    workflow: 16
---

Bu sayfa **yerel OpenClaw plugin manifestini**, `openclaw.plugin.json` ele alır. Uyumlu paket düzenleri (Codex, Claude, Cursor) için [Plugin paketleri](/tr/plugins/bundles) bölümüne bakın.

Uyumlu paket biçimleri bunun yerine kendi manifest dosyalarını kullanır:

- Codex paketi: `.codex-plugin/plugin.json`
- Claude paketi: `.claude-plugin/plugin.json` veya manifestsiz varsayılan Claude bileşen düzeni
- Cursor paketi: `.cursor-plugin/plugin.json`

OpenClaw bu düzenleri otomatik olarak algılar ancak aşağıdaki `openclaw.plugin.json` şemasına göre doğrulamaz. Uyumlu bir paketin düzeni OpenClaw'ın çalışma zamanı beklentileriyle eşleştiğinde OpenClaw; paket meta verilerini, bildirilen skill köklerini, Claude komut köklerini, Claude `settings.json` varsayılanlarını, Claude LSP varsayılanlarını ve desteklenen kanca paketlerini okur.

Her yerel OpenClaw plugini, **plugin kökünde** `openclaw.plugin.json` dosyasını **mutlaka** sağlamalıdır. OpenClaw, yapılandırmayı **plugin kodunu çalıştırmadan** doğrulamak için bu dosyayı okur. Eksik veya geçersiz bir manifest, yapılandırma doğrulamasını engeller ve plugin hatası olarak değerlendirilir.

Plugin sisteminin eksiksiz kılavuzu için [Pluginler](/tr/tools/plugin), yerel yetenek modeli ve harici uyumluluğa ilişkin güncel yönergeler için [Yetenek modeli](/tr/plugins/architecture#public-capability-model) bölümüne bakın.

## Bu dosyanın işlevi

`openclaw.plugin.json`, OpenClaw'ın **plugin kodunuzu yüklemeden önce** okuduğu meta verilerdir. İçindeki her şey, plugin çalışma zamanını başlatmadan incelenebilecek kadar düşük maliyetli olmalıdır.

**Şunlar için kullanın:**

- plugin kimliği, yapılandırma doğrulaması ve yapılandırma kullanıcı arayüzü ipuçları
- kimlik doğrulama, ilk kullanım ve kurulum meta verileri (takma ad, otomatik etkinleştirme, sağlayıcı ortam değişkenleri, kimlik doğrulama seçenekleri)
- kontrol düzlemi yüzeyleri için etkinleştirme ipuçları
- kısaltılmış model ailesi sahipliği
- statik yetenek sahipliği anlık görüntüleri (`contracts`)
- kontrol paneli bileşeni veri bağlamaları ve eylem fiilleri
- plugin etkinken bulunması gereken statik MCP sunucuları
- paylaşılan `openclaw qa` ana makinesinin inceleyebileceği QA çalıştırıcısı meta verileri
- katalog ve doğrulama yüzeyleriyle birleştirilen kanala özgü yapılandırma meta verileri

**Şunlar için kullanmayın:** yerel çalışma zamanı kancalarını kaydetme, plugin kodu giriş noktalarını bildirme veya npm kurulum meta verileri. Bunlar plugin kodunuzda ve `package.json` içinde yer almalıdır.

## Minimal örnek

```json
{
  "id": "voice-call",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

## Kapsamlı örnek

```json
{
  "id": "openrouter",
  "name": "OpenRouter",
  "description": "OpenRouter sağlayıcı plugini",
  "version": "1.0.0",
  "providers": ["openrouter"],
  "modelSupport": {
    "modelPrefixes": ["router-"]
  },
  "modelIdNormalization": {
    "providers": {
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  },
  "providerEndpoints": [
    {
      "endpointClass": "openrouter",
      "hostSuffixes": ["openrouter.ai"]
    }
  ],
  "providerRequest": {
    "providers": {
      "openrouter": {
        "family": "openrouter"
      }
    }
  },
  "cliBackends": ["openrouter-cli"],
  "syntheticAuthRefs": ["openrouter-cli"],
  "setup": {
    "providers": [
      {
        "id": "openrouter",
        "envVars": ["OPENROUTER_API_KEY"]
      }
    ]
  },
  "providerAuthAliases": {
    "openrouter-coding": "openrouter"
  },
  "providerAuthChoices": [
    {
      "provider": "openrouter",
      "method": "api-key",
      "choiceId": "openrouter-api-key",
      "choiceLabel": "OpenRouter API anahtarı",
      "groupId": "openrouter",
      "groupLabel": "OpenRouter",
      "optionKey": "openrouterApiKey",
      "cliFlag": "--openrouter-api-key",
      "cliOption": "--openrouter-api-key <key>",
      "cliDescription": "OpenRouter API anahtarı",
      "onboardingScopes": ["text-inference"]
    }
  ],
  "uiHints": {
    "apiKey": {
      "label": "API anahtarı",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "apiKey": {
        "type": "string"
      }
    }
  }
}
```

## Üst düzey alan başvurusu

| Alan                                 | Gerekli   | Tür                          | Anlamı                                                                                                                                                                                                                                                                                         |
| ------------------------------------ | --------- | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                   | Evet      | `string`           | Standart plugin kimliği. Bu, `plugins.entries.<id>` içinde kullanılan kimliktir.                                                                                                                                                                                                                    |
| `configSchema`                   | Evet      | `object`           | Bu plugin yapılandırması için satır içi JSON Schema.                                                                                                                                                                                                                                           |
| `requiresPlugins`                   | Hayır     | `string[]`           | Bu pluginin etkili olabilmesi için ayrıca yüklenmesi gereken plugin kimlikleri. Keşif, plugini yüklenebilir durumda tutar ancak gerekli pluginlerden biri eksik olduğunda uyarır.                                                                                                                |
| `enabledByDefault`                   | Hayır     | `true`           | Paketle gelen bir plugini varsayılan olarak etkin şeklinde işaretler. Pluginin varsayılan olarak devre dışı kalması için bunu atlayın veya `true` dışında herhangi bir değere ayarlayın.                                                                                              |
| `enabledByDefaultOnPlatforms`                   | Hayır     | `string[]`           | Paketle gelen bir plugini yalnızca listelenen Node.js platformlarında varsayılan olarak etkin şeklinde işaretler; örneğin `["darwin"]`. Açık yapılandırma yine de önceliklidir.                                                                                                            |
| `legacyPluginIds`                   | Hayır     | `string[]`           | Bu standart plugin kimliğine normalleştirilen eski kimlikler.                                                                                                                                                                                                                                  |
| `autoEnableWhenConfiguredProviders`                   | Hayır     | `string[]`           | Kimlik doğrulama, yapılandırma veya model başvuruları bunlardan söz ettiğinde bu plugini otomatik olarak etkinleştirmesi gereken sağlayıcı kimlikleri.                                                                                                                                          |
| `kind`                   | Hayır     | `PluginKind \| PluginKind[]`           | `plugins.slots.*` tarafından kullanılan bir veya daha fazla özel plugin türünü (`"memory"`, `"context-engine"`) bildirir. Her iki yuvanın da sahibi olan bir plugin, iki türü tek bir dizide bildirir.                                                                                   |
| `channels`                   | Hayır     | `string[]`           | Bu pluginin sahibi olduğu kanal kimlikleri. Keşif ve yapılandırma doğrulaması için kullanılır.                                                                                                                                                                                                  |
| `providers`                   | Hayır     | `string[]`           | Bu pluginin sahibi olduğu sağlayıcı kimlikleri.                                                                                                                                                                                                                                                |
| `providerCatalogEntry`                   | Hayır     | `string`           | Plugin köküne göreli, tam plugin çalışma zamanını etkinleştirmeden yüklenebilen ve manifest kapsamındaki sağlayıcı kataloğu meta verilerini içeren hafif sağlayıcı kataloğu modül yolu.                                                                                                          |
| `modelSupport`                   | Hayır     | `object`           | Plugini çalışma zamanından önce otomatik olarak yüklemek için kullanılan, manifestin sahibi olduğu kısaltılmış model ailesi meta verileri.                                                                                                                                                      |
| `modelCatalog`                   | Hayır     | `object`           | Bu pluginin sahibi olduğu sağlayıcılar için bildirimsel model kataloğu meta verileri. Bu, plugin çalışma zamanını yüklemeden gelecekte salt okunur listeleme, ilk kullanım kurulumu, model seçiciler, takma adlar ve gizleme işlemleri için denetim düzlemi sözleşmesidir.                          |
| `modelPricing`                   | Hayır     | `object`           | Sağlayıcının sahibi olduğu harici fiyat arama politikası. Yerel/kendi barındırılan sağlayıcıları uzak fiyatlandırma kataloglarının dışında bırakmak veya sağlayıcı başvurularını çekirdekte sağlayıcı kimliklerini sabit kodlamadan OpenRouter/LiteLLM katalog kimlikleriyle eşlemek için kullanın. |
| `modelIdNormalization`                   | Hayır     | `object`           | Sağlayıcı çalışma zamanı yüklenmeden önce çalışması gereken, sağlayıcının sahibi olduğu model kimliği takma adı/ön ek temizliği.                                                                                                                                                                |
| `providerEndpoints`                   | Hayır     | `object[]`           | Sağlayıcı çalışma zamanı yüklenmeden önce çekirdeğin sınıflandırması gereken sağlayıcı yolları için manifestin sahibi olduğu uç nokta ana makine/baseUrl meta verileri.                                                                                                                         |
| `providerRequest`                   | Hayır     | `object`           | Sağlayıcı çalışma zamanı yüklenmeden önce genel istek politikası tarafından kullanılan düşük maliyetli sağlayıcı ailesi ve istek uyumluluğu meta verileri.                                                                                                                                      |
| `secretProviderIntegrations`                   | Hayır     | `Record<string, object>`           | Kurulum yüzeylerinin, çekirdekte sağlayıcıya özgü entegrasyonları sabit kodlamadan sunabileceği bildirimsel SecretRef yürütme sağlayıcısı ön ayarları.                                                                                                                                           |
| `cliBackends`                   | Hayır     | `string[]`           | Bu pluginin sahibi olduğu CLI çıkarım arka uç kimlikleri. Açık yapılandırma başvurularından başlangıçta otomatik etkinleştirme için kullanılır.                                                                                                                                                  |
| `syntheticAuthRefs`                   | Hayır     | `string[]`           | Çalışma zamanı yüklenmeden önce soğuk model keşfi sırasında pluginin sahibi olduğu sentetik kimlik doğrulama kancasının yoklanması gereken sağlayıcı veya CLI arka uç başvuruları.                                                                                                               |
| `nonSecretAuthMarkers`                   | Hayır     | `string[]`           | Gizli olmayan yerel, OAuth veya ortam kimlik bilgisi durumunu temsil eden, paketle gelen pluginin sahibi olduğu yer tutucu API anahtarı değerleri.                                                                                                                                               |
| `commandAliases`                   | Hayır     | `object[]`           | Bu pluginin sahibi olduğu ve çalışma zamanı yüklenmeden önce plugin farkındalığına sahip yapılandırma ve CLI tanılamaları üretmesi gereken komut adları.                                                                                                                                        |
| `providerUsageAuthEnvVars`                   | Hayır     | `Record<string, string[]>`           | Yalnızca kullanım/faturalandırma amaçlı sağlayıcı kimlik bilgileri. OpenClaw bu adları kullanım keşfi ve gizli bilgi temizliği için kullanır ancak çıkarım kimlik doğrulaması için asla kullanmaz.                                                                                               |
| `providerAuthAliases`                   | Hayır     | `Record<string, string>`           | Kimlik doğrulama araması için başka bir sağlayıcı kimliğini yeniden kullanması gereken sağlayıcı kimlikleri; örneğin temel sağlayıcının API anahtarını ve kimlik doğrulama profillerini paylaşan bir kodlama sağlayıcısı.                                                                         |
| `providerAuthChoices`                   | Hayır     | `object[]`           | İlk kullanım kurulumu seçicileri, tercih edilen sağlayıcı çözümlemesi ve basit CLI bayrağı bağlantıları için düşük maliyetli kimlik doğrulama seçeneği meta verileri.                                                                                                                            |
| `activation`                   | Hayır     | `object`           | Başlangıç, sağlayıcı, komut, kanal, yol ve yetenek tarafından tetiklenen yükleme için düşük maliyetli etkinleştirme planlayıcısı meta verileri. Yalnızca meta verilerdir; gerçek davranışın sahibi yine plugin çalışma zamanıdır.                                                                 |
| `setup`                   | Hayır     | `object`           | Keşif ve kurulum yüzeylerinin plugin çalışma zamanını yüklemeden inceleyebileceği düşük maliyetli kurulum/ilk kullanım kurulumu tanımlayıcıları.                                                                                                                                                 |
| `qaRunners`                   | Hayır     | `object[]`           | Plugin çalışma zamanı yüklenmeden önce paylaşılan `openclaw qa` ana makinesi tarafından kullanılan düşük maliyetli QA çalıştırıcısı tanımlayıcıları.                                                                                                                                      |
| `dashboard`                   | Hayır     | `object`           | Pano pencere öğesi veri bağlamaları ve eylem fiilleri. Her girdi, bu plugin tarafından gerekli okuma veya yazma kapsamıyla kaydedilen bir Gateway yöntemine göre doğrulanır. Bkz. [pano başvurusu](#dashboard-reference).                                                                          |
| `mcpServers`                         | Hayır       | `Record<string, object>`     | Bu plugin etkin olduğu sürece sağlanan statik MCP sunucusu tanımları. Göreli komut bağımsız değişkenleri ve çalışma dizinleri plugin kökünden çözümlenir. Operatör `mcp.servers` girdileri, aynı ada sahip tanımları geçersiz kılar veya devre dışı bırakır. Bkz. [MCP sunucusu referansı](#mcp-server-reference). |
| `contracts`                          | Hayır       | `object`                     | Harici kimlik doğrulama kancaları, gömmeler, konuşma, gerçek zamanlı transkripsiyon, gerçek zamanlı ses, medya anlama, görüntü/video/müzik üretimi, web'den getirme, web araması, çalışan sağlayıcıları, belge/web içeriği çıkarma ve araç sahipliği için statik yetenek sahipliği anlık görüntüsü.                     |
| `configContracts`                    | Hayır       | `object`                     | Genel çekirdek yardımcıları tarafından kullanılan, manifestin sahip olduğu yapılandırma davranışı: tehlikeli bayrak algılama, SecretRef geçiş hedefleri ve eski yapılandırma yolu daraltma. Bkz. [configContracts referansı](#configcontracts-reference).                                                                         |
| `mediaUnderstandingProviderMetadata` | Hayır       | `Record<string, object>`     | `contracts.mediaUnderstandingProviders` içinde bildirilen sağlayıcı kimlikleri için düşük maliyetli medya anlama varsayılanları.                                                                                                                                                                                       |
| `imageGenerationProviderMetadata`    | Hayır       | `Record<string, object>`     | Sağlayıcıya ait kimlik doğrulama takma adları ve temel URL korumaları dahil olmak üzere, `contracts.imageGenerationProviders` içinde bildirilen sağlayıcı kimlikleri için düşük maliyetli görüntü üretimi kimlik doğrulama meta verileri.                                                                                                                             |
| `videoGenerationProviderMetadata`    | Hayır       | `Record<string, object>`     | Sağlayıcıya ait kimlik doğrulama takma adları ve temel URL korumaları dahil olmak üzere, `contracts.videoGenerationProviders` içinde bildirilen sağlayıcı kimlikleri için düşük maliyetli video üretimi kimlik doğrulama meta verileri.                                                                                                                             |
| `musicGenerationProviderMetadata`    | Hayır       | `Record<string, object>`     | Sağlayıcıya ait kimlik doğrulama takma adları ve temel URL korumaları dahil olmak üzere, `contracts.musicGenerationProviders` içinde bildirilen sağlayıcı kimlikleri için düşük maliyetli müzik üretimi kimlik doğrulama meta verileri.                                                                                                                             |
| `toolMetadata`                       | Hayır       | `Record<string, object>`     | `contracts.tools` içinde bildirilen, plugin'e ait araçlar için düşük maliyetli kullanılabilirlik meta verileri. Bir aracın yalnızca yapılandırma, ortam veya kimlik doğrulama kanıtı mevcut olduğunda çalışma zamanını yüklemesi gerekiyorsa bunu kullanın.                                                                                                                      |
| `channelConfigs`                     | Hayır       | `Record<string, object>`     | Çalışma zamanı yüklenmeden önce keşif ve doğrulama yüzeyleriyle birleştirilen, manifestin sahip olduğu kanal yapılandırma meta verileri.                                                                                                                                                                                     |
| `skills`                             | Hayır       | `string[]`                   | Plugin köküne göre yüklenecek Skills dizinleri.                                                                                                                                                                                                                                        |
| `name`                               | Hayır       | `string`                     | İnsanların okuyabileceği plugin adı.                                                                                                                                                                                                                                                                    |
| `description`                        | Hayır       | `string`                     | Plugin yüzeylerinde gösterilen kısa özet.                                                                                                                                                                                                                                                        |
| `catalog`                            | Hayır       | `object`                     | Plugin kataloğu yüzeyleri için isteğe bağlı sunum ipuçları. Bu meta veriler bir plugin'i yüklemez, etkinleştirmez veya ona güven vermez.                                                                                                                                                                   |
| `icon`                               | Hayır       | `string`                     | Pazar yeri/katalog kartları için HTTPS görüntü URL'si. ClawHub, geçerli herhangi bir `https://` URL'sini kabul eder ve bu değer belirtilmediğinde veya geçersiz olduğunda varsayılan plugin simgesine geri döner.                                                                                                                             |
| `version`                            | Hayır       | `string`                     | Bilgilendirme amaçlı plugin sürümü.                                                                                                                                                                                                                                                                  |
| `uiHints`                            | Hayır       | `Record<string, object>`     | Yapılandırma alanları için kullanıcı arayüzü etiketleri, yer tutucular ve hassasiyet ipuçları.                                                                                                                                                                                                                              |

## MCP sunucusu referansı

`mcpServers`, operatörlerin statik süreç tanımını `openclaw.json` içinde çoğaltmasını gerektirmeden yerel bir pluginin, bir MCP App dâhil olmak üzere bir MCP sunucusu sunmasına olanak tanır:

```json
{
  "mcpServers": {
    "example": {
      "transport": "stdio",
      "command": "node",
      "args": ["./mcp-server.js"]
    }
  }
}
```

OpenClaw bu sunucuları yalnızca sahibi olan plugin etkin durumdayken dâhil eder. Göreli `command`, `args`, `cwd` ve `workingDirectory` yolları plugin kökünden çözümlenir. Kullanıcı yapılandırması belirleyici olmaya devam eder: `mcp.servers.<name>` bir plugin varsayılanını değiştirebilir veya sunucuyu hariç tutmak için `enabled: false` değerini ayarlayabilir. MCP App görüntüleme ve sunucu aracı çağrıları için yine normal MCP Apps ayarı ve geçerli araç politikası gerekir; bir sunucu bildirmek bu sınırların hiçbirini aşmaz.

## dashboard referansı

`dashboard`, etkin bir pluginin çekirdeğe plugin politikası eklemeden mevcut Gateway RPC'lerini izin verilmiş dashboard widget'larına sunmasına olanak tanır. Veri bağlamaları, aynı pluginin `operator.read` ile kaydettiği bir yöntemi adlandırmalıdır; eylem fiilleri ise `operator.write` ile kaydettiği bir yöntemi adlandırmalıdır. Bir uyuşmazlık, kayıt sırasında pluginin reddedilmesine neden olur.

```json
{
  "dashboard": {
    "dataBindings": [
      {
        "id": "items.list",
        "method": "example.items.list",
        "description": "Örnek öğeleri listele."
      }
    ],
    "actionVerbs": [
      {
        "id": "refresh",
        "method": "example.items.refresh",
        "description": "Örnek öğeleri yenile.",
        "paramShape": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "force": { "type": "boolean" }
          }
        }
      }
    ]
  }
}
```

Manifest kimlikleri plugine özeldir. Widget izinleri, `example.items.list` ve `example.refresh` gibi `<plugin-id>.<id>` değerlerini kullanır. Kalıcı izin ad alanını belirsizlikten uzak tutmak için OpenClaw, plugin kimliği segmentindeki `%` ve `.` değerlerini `%25` ve `%2E` olarak kaçışlar; sıradan plugin kimlikleri doğal biçimini korur. `paramShape`, OpenClaw plugin RPC'sini çağırmadan önce eylem parametreleri nesnesine uygulanan isteğe bağlı bir JSON Schema'dır.

## katalog referansı

`catalog`, plugin tarayıcılarına isteğe bağlı görüntüleme ipuçları sağlar. Ana makineler bu ipuçlarını yok sayabilir. Bunlar plugini hiçbir zaman yüklemez veya etkinleştirmez ve pluginin çalışma zamanı davranışını ya da güven düzeyini değiştirmez.

```json
{
  "catalog": {
    "featured": true,
    "order": 10
  }
}
```

| Alan       | Tür       | Anlamı                                                                     |
| ---------- | --------- | -------------------------------------------------------------------------- |
| `featured` | `boolean` | Katalog yüzeylerinin bu plugini öne çıkarıp çıkarmaması.                    |
| `order`    | `number`  | Seçilmiş pluginler arasındaki artan görüntüleme ipucu; düşük değerler daha önce görünür. |

## Üretim sağlayıcısı meta verileri referansı

Üretim sağlayıcısı meta veri alanları, eşleşen `contracts.*GenerationProviders` listesinde bildirilen sağlayıcıların statik kimlik doğrulama sinyallerini açıklar. OpenClaw bu alanları sağlayıcı çalışma zamanı yüklenmeden önce okur; böylece çekirdek araçlar, her sağlayıcı pluginini içe aktarmadan bir üretim sağlayıcısının kullanılabilir olup olmadığına karar verebilir.

Bu alanları yalnızca düşük maliyetli, bildirime dayalı olgular için kullanın. Aktarım, istek dönüşümleri, token yenileme, kimlik bilgisi doğrulama ve gerçek üretim davranışı plugin çalışma zamanında kalır.

```json
{
  "contracts": {
    "imageGenerationProviders": ["example-image"]
  },
  "imageGenerationProviderMetadata": {
    "example-image": {
      "aliases": ["example-image-oauth"],
      "authProviders": ["example-image"],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example-image.config",
          "overlayPath": "image",
          "mode": {
            "path": "mode",
            "default": "local",
            "allowed": ["local"]
          },
          "requiredAny": ["workflow", "workflowPath"],
          "required": ["promptNodeId"]
        }
      ],
      "authSignals": [
        {
          "provider": "example-image"
        },
        {
          "provider": "example-image-oauth",
          "providerBaseUrl": {
            "provider": "example-image",
            "defaultBaseUrl": "https://api.example.com/v1",
            "allowedBaseUrls": ["https://api.example.com/v1"]
          }
        }
      ]
    }
  }
}
```

Her meta veri girdisi şunları destekler:

| Alan                   | Gerekli   | Tür        | Anlamı                                                                                                                                              |
| ---------------------- | -------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `aliases`              | Hayır    | `string[]` | Üretim sağlayıcısı için statik kimlik doğrulama diğer adları olarak sayılması gereken ek sağlayıcı kimlikleri.                                        |
| `authProviders`        | Hayır    | `string[]` | Yapılandırılmış kimlik doğrulama profillerinin bu üretim sağlayıcısı için kimlik doğrulama olarak sayılması gereken sağlayıcı kimlikleri.              |
| `configSignals`        | Hayır    | `object[]` | Kimlik doğrulama profilleri veya ortam değişkenleri olmadan yapılandırılabilen yerel ya da kendi barındırılan sağlayıcılar için düşük maliyetli, yalnızca yapılandırmaya dayalı kullanılabilirlik sinyalleri. |
| `authSignals`          | Hayır    | `object[]` | Açık kimlik doğrulama sinyalleri. Mevcut olduğunda bunlar, sağlayıcı kimliği, `aliases` ve `authProviders` kaynaklı varsayılan sinyal kümesinin yerini alır. |
| `referenceAudioInputs` | Hayır    | `boolean`  | Yalnızca video üretimi. Sağlayıcı referans ses varlıklarını kabul ettiğinde `true` olarak ayarlayın; aksi takdirde `video_generate` ses referansı parametrelerini gizler. |

Her `configSignals` girdisi şunları destekler:

| Alan             | Gerekli   | Tür        | Anlamı                                                                                                                                                                                    |
| ---------------- | -------- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `rootPath`       | Evet      | `string`   | İncelenecek, pluginin sahip olduğu yapılandırma nesnesine giden noktalı yol; örneğin `plugins.entries.example.config`.                                                                                    |
| `overlayPath`    | Hayır    | `string`   | Sinyal değerlendirilmeden önce nesnesi kök nesnenin üzerine bindirilecek olan kök yapılandırma içindeki noktalı yol. Bunu `image`, `video` veya `music` gibi yeteneğe özgü yapılandırmalar için kullanın. |
| `overlayMapPath` | Hayır    | `string`   | Nesne değerlerinin her biri kök nesnenin üzerine bindirilecek olan kök yapılandırma içindeki noktalı yol. Bunu, yapılandırılmış herhangi bir hesabın yeterli sayılması gereken `accounts` gibi adlandırılmış hesap eşlemeleri için kullanın. |
| `required`       | Hayır    | `string[]` | Geçerli yapılandırma içinde yapılandırılmış değerlere sahip olması gereken noktalı yollar. Dizeler boş olmamalıdır; nesneler ve diziler boş olmamalıdır.                                    |
| `requiredAny`    | Hayır    | `string[]` | Geçerli yapılandırma içinde en az birinin yapılandırılmış bir değere sahip olması gereken noktalı yollar.                                                                                   |
| `mode`           | Hayır    | `object`   | Geçerli yapılandırma içindeki isteğe bağlı dize modu koruması. Bunu yalnızca yapılandırmaya dayalı kullanılabilirlik tek bir mod için geçerli olduğunda kullanın.                            |

Her `mode` koruması şunları destekler:

| Alan         | Gerekli   | Tür        | Anlamı                                                                             |
| ------------ | -------- | ---------- | ---------------------------------------------------------------------------------- |
| `path`       | Hayır    | `string`   | Geçerli yapılandırma içindeki noktalı yol. Varsayılan değer `mode`.     |
| `default`    | Hayır    | `string`   | Yapılandırma yolu içermediğinde kullanılacak mod değeri.                            |
| `allowed`    | Hayır    | `string[]` | Mevcutsa sinyal yalnızca geçerli mod bu değerlerden biri olduğunda geçer.           |
| `disallowed` | Hayır    | `string[]` | Mevcutsa sinyal, geçerli mod bu değerlerden biri olduğunda başarısız olur.           |

Her `authSignals` girdisi şunları destekler:

| Alan              | Gerekli   | Tür      | Anlamı                                                                                                                                                                        |
| ----------------- | -------- | -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | Evet      | `string` | Yapılandırılmış kimlik doğrulama profillerinde denetlenecek sağlayıcı kimliği.                                                                                                 |
| `providerBaseUrl` | Hayır    | `object` | Sinyalin yalnızca başvurulan yapılandırılmış sağlayıcı izin verilen bir temel URL kullandığında sayılmasını sağlayan isteğe bağlı koruma. Bunu bir kimlik doğrulama diğer adı yalnızca belirli API'ler için geçerli olduğunda kullanın. |

Her `providerBaseUrl` koruması şunları destekler:

| Alan              | Gerekli   | Tür        | Anlamı                                                                                                                                               |
| ----------------- | -------- | ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`        | Evet      | `string`   | `baseUrl` değeri denetlenecek sağlayıcı yapılandırma kimliği.                                                                                |
| `defaultBaseUrl`  | Hayır    | `string`   | Sağlayıcı yapılandırması `baseUrl` değerini içermediğinde varsayılacak temel URL.                                                            |
| `allowedBaseUrls` | Evet      | `string[]` | Bu kimlik doğrulama sinyali için izin verilen temel URL'ler. Yapılandırılmış veya varsayılan temel URL bu normalleştirilmiş değerlerden biriyle eşleşmediğinde sinyal yok sayılır. |

## Araç meta verileri referansı

`toolMetadata`, araç adına göre anahtarlanmış üretim sağlayıcısı meta verileriyle aynı `configSignals` ve `authSignals` biçimlerini kullanır. `contracts.tools` sahipliği bildirir. `toolMetadata`, yalnızca araç fabrikasının `null` döndürmesi için OpenClaw'ın bir plugin çalışma zamanını içe aktarmaktan kaçınabilmesini sağlayan düşük maliyetli kullanılabilirlik kanıtını bildirir.

```json
{
  "setup": {
    "providers": [
      {
        "id": "example",
        "envVars": ["EXAMPLE_API_KEY"]
      }
    ]
  },
  "contracts": {
    "tools": ["example_search"]
  },
  "toolMetadata": {
    "example_search": {
      "authSignals": [
        {
          "provider": "example"
        }
      ],
      "configSignals": [
        {
          "rootPath": "plugins.entries.example.config",
          "overlayPath": "search",
          "required": ["apiKey"]
        }
      ]
    }
  }
}
```

`toolMetadata` girdileri ayrıca, yukarıdaki ortak `configSignals`/`authSignals` alanlarına ek olarak, `optional` (aracı plugin etkinleştirmesi için zorunlu değil olarak işaretler) ve `replaySafe` (tamamlanmamış bir model turundan sonra araç yürütmesini tekrarlamanın güvenli olduğunu işaretler) değerlerini de kabul eder.

Bir aracın `toolMetadata` değeri yoksa OpenClaw mevcut davranışı korur ve araç sözleşmesi politikayla eşleştiğinde aracın sahibi olan plugini yükler. Fabrikası kimlik doğrulamaya/yapılandırmaya bağlı olan yoğun kullanılan araçlar için plugin yazarları, çekirdeğin sormak üzere çalışma zamanını içe aktarmasını sağlamak yerine `toolMetadata` bildirmelidir.

## providerAuthChoices başvurusu

Her `providerAuthChoices` girdisi bir ilk katılım veya kimlik doğrulama seçeneğini açıklar. OpenClaw bunu sağlayıcı çalışma zamanı yüklenmeden önce okur. Sağlayıcı kurulum listeleri, sağlayıcı çalışma zamanını yüklemeden bu manifest seçeneklerini, tanımlayıcıdan türetilmiş kurulum seçeneklerini ve kurulum kataloğu meta verilerini kullanır.

| Alan                  | Zorunlu | Tür                                                                   | Anlamı                                                                                                            |
| --------------------- | ------- | --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `provider`            | Evet    | `string`                                                              | Bu seçeneğin ait olduğu sağlayıcı kimliği.                                                                        |
| `method`              | Evet    | `string`                                                              | Yönlendirilecek kimlik doğrulama yöntemi kimliği.                                                                 |
| `choiceId`            | Evet    | `string`                                                              | İlk katılım ve CLI akışlarında kullanılan kararlı kimlik doğrulama seçeneği kimliği.                              |
| `choiceLabel`         | Hayır   | `string`                                                              | Kullanıcıya gösterilen etiket. Belirtilmezse OpenClaw, `choiceId` değerine geri döner.                    |
| `choiceHint`          | Hayır   | `string`                                                              | Seçici için kısa yardımcı metin.                                                                                  |
| `icon`                | Hayır   | HTTPS URL                                                             | Desteklenen ilk katılım istemcilerinde bu seçeneğin yanında gösterilen görsel.                                    |
| `website`             | Hayır   | HTTPS URL                                                             | Desteklenen ilk katılım istemcilerinin gösterdiği ürün, oturum açma veya kurulum sayfası.                         |
| `assistantPriority`   | Hayır   | `number`                                                              | Daha düşük değerler, asistan odaklı etkileşimli seçicilerde daha önce sıralanır.                                  |
| `assistantVisibility` | Hayır   | `"visible"` \| `"manual-only"`                                        | Manuel CLI seçimine izin vermeye devam ederken seçeneği asistan seçicilerinden gizler.                            |
| `deprecatedChoiceIds` | Hayır   | `string[]`                                                            | Kullanıcıları bu yedek seçeneğe yönlendirmesi gereken eski seçenek kimlikleri.                                    |
| `groupId`             | Hayır   | `string`                                                              | İlgili seçenekleri gruplandırmak için isteğe bağlı grup kimliği.                                                  |
| `groupLabel`          | Hayır   | `string`                                                              | Bu grup için kullanıcıya gösterilen etiket.                                                                       |
| `groupHint`           | Hayır   | `string`                                                              | Grup için kısa yardımcı metin.                                                                                    |
| `onboardingFeatured`  | Hayır   | `boolean`                                                             | Bu grubu, "More..." girdisinden önce etkileşimli ilk katılım seçicisinin öne çıkan katmanında gösterir.           |
| `optionKey`           | Hayır   | `string`                                                              | Tek bayraklı basit kimlik doğrulama akışları için dahili seçenek anahtarı.                                        |
| `cliFlag`             | Hayır   | `string`                                                              | `--openrouter-api-key` gibi CLI bayrağı adı.                                                                          |
| `cliOption`           | Hayır   | `string`                                                              | `--openrouter-api-key <key>` gibi tam CLI seçeneği biçimi.                                                                  |
| `cliDescription`      | Hayır   | `string`                                                              | CLI yardımında kullanılan açıklama.                                                                               |
| `appGuidedSecret`     | Hayır   | `boolean`                                                             | Yapıştırılan tek bir gizli değer ve sağlayıcı varsayılanları, uygulama yönlendirmeli kurulum için yeterlidir.     |
| `appGuidedDiscovery`  | Hayır   | `boolean`                                                             | Eşleşen çalışma zamanı kimlik doğrulama yöntemi, `appGuidedSetup` aracılığıyla salt okunur yerel keşfin sahibidir. |
| `appGuidedAuth`       | Hayır   | `"oauth"` \| `"device-code"`                                          | Yerel kurulum istemcilerinin genel biçimde işleyebileceği, sağlayıcıya ait etkileşimli oturum açma.               |
| `onboardingScopes`    | Hayır   | `Array<"text-inference" \| "image-generation" \| "music-generation">` | Bu seçeneğin hangi ilk katılım yüzeylerinde görünmesi gerektiği. Belirtilmezse varsayılanı `["text-inference"]` olur. |

`appGuidedDiscovery` doğru olduğunda, eşleşen sağlayıcı kimlik doğrulama yöntemi
`appGuidedSetup.detect` ve `appGuidedSetup.prepare` değerlerini sunmalıdır. Algılama
salt okunur olmalıdır: oturum açma, model çekme, indirme veya yapılandırma yazma işlemi yapılmaz. Hazırlık,
seçilen tam modeli yeniden denetler ve bir yapılandırma önerisi döndürür; OpenClaw bu
öneriyi yalıtılmış biçimde canlı olarak test eder ve yalnızca başarılı olduktan sonra kaydeder.

## commandAliases başvurusu

Bir plugin, kullanıcıların yanlışlıkla `plugins.allow` içine koyabileceği veya kök CLI komutu olarak çalıştırmayı deneyebileceği bir çalışma zamanı komut adının sahibiyse `commandAliases` kullanın. OpenClaw bu meta verileri, plugin çalışma zamanı kodunu içe aktarmadan tanılama amacıyla kullanır.

```json
{
  "commandAliases": [
    {
      "name": "dreaming",
      "kind": "runtime-slash",
      "cliCommand": "memory"
    }
  ]
}
```

| Alan         | Zorunlu | Tür               | Anlamı                                                                         |
| ------------ | ------- | ----------------- | ------------------------------------------------------------------------------ |
| `name`       | Evet    | `string`          | Bu plugine ait komut adı.                                                       |
| `kind`       | Hayır   | `"runtime-slash"` | Takma adı, kök CLI komutu yerine bir sohbet eğik çizgi komutu olarak işaretler. |
| `cliCommand` | Hayır   | `string`          | Varsa CLI işlemleri için önerilecek ilgili kök CLI komutu.                      |

## activation başvurusu

Plugin, hangi kontrol düzlemi olaylarının kendisini bir etkinleştirme/yükleme planına dahil etmesi gerektiğini düşük maliyetle bildirebiliyorsa `activation` kullanın.

Bu blok, yaşam döngüsü API'si değil, planlayıcı meta verisidir. Çalışma zamanı davranışını kaydetmez, `register(...)` yerine geçmez ve plugin kodunun zaten yürütülmüş olduğunu garanti etmez. Etkinleştirme planlayıcısı; `providers`, `channels`, `commandAliases`, `setup.providers`, `contracts.tools` ve kancalar gibi mevcut manifest sahipliği meta verilerine geri dönmeden önce aday pluginleri daraltmak için bu alanları kullanır.

Sahipliği zaten açıklayan en dar kapsamlı meta verileri tercih edin. İlişkiyi bu alanlar ifade ediyorsa `providers`, `channels`, `commandAliases`, kurulum tanımlayıcıları veya `contracts` kullanın. Bu sahiplik alanlarıyla ifade edilemeyen ek planlayıcı ipuçları için `activation` kullanın. `claude-cli`, `my-cli` veya `google-gemini-cli` gibi CLI çalışma zamanı takma adları için üst düzey `cliBackends` kullanın; `activation.onAgentHarnesses` yalnızca henüz bir sahiplik alanı bulunmayan gömülü aracı çalıştırma ortamı kimlikleri içindir.

Her plugin `activation.onStartup` değerini bilinçli olarak ayarlamalıdır. Yalnızca pluginin Gateway başlatılırken çalışması gerekiyorsa bunu `true` olarak ayarlayın. Plugin başlangıçta etkisizse ve yalnızca daha dar kapsamlı tetikleyicilerden yüklenmesi gerekiyorsa bunu `false` olarak ayarlayın. `onStartup` değerinin atlanması artık plugini başlangıçta örtük olarak yüklemez; başlangıç, kanal, yapılandırma, aracı çalıştırma ortamı, bellek veya diğer daha dar kapsamlı etkinleştirme tetikleyicileri için açık etkinleştirme meta verileri kullanın.

```json
{
  "activation": {
    "onStartup": false,
    "onProviders": ["openai"],
    "onCommands": ["models"],
    "onChannels": ["web"],
    "onRoutes": ["gateway-webhook"],
    "onConfigPaths": ["browser"],
    "onCapabilities": ["provider", "tool"]
  }
}
```

| Alan               | Gerekli | Tür                                                  | Anlamı                                                                                                                                                                                      |
| ------------------ | -------- | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `onStartup`        | Hayır    | `boolean`                                            | Açık Gateway başlangıç etkinleştirmesi. Her plugin bunu ayarlamalıdır. `true`, başlangıç sırasında plugin'i içe aktarır; `false`, eşleşen başka bir tetikleyici yüklemeyi gerektirmediği sürece başlangıçta geç yüklenmesini sağlar. |
| `onProviders`      | Hayır    | `string[]`                                           | Etkinleştirme/yükleme planlarına bu plugin'i dahil etmesi gereken sağlayıcı kimlikleri.                                                                                                      |
| `onAgentHarnesses` | Hayır    | `string[]`                                           | Etkinleştirme/yükleme planlarına bu plugin'i dahil etmesi gereken gömülü aracı donanım çalışma zamanı kimlikleri. CLI arka uç diğer adları için üst düzey `cliBackends` kullanın.             |
| `onCommands`       | Hayır    | `string[]`                                           | Etkinleştirme/yükleme planlarına bu plugin'i dahil etmesi gereken komut kimlikleri.                                                                                                          |
| `onChannels`       | Hayır    | `string[]`                                           | Etkinleştirme/yükleme planlarına bu plugin'i dahil etmesi gereken kanal kimlikleri.                                                                                                          |
| `onRoutes`         | Hayır    | `string[]`                                           | Etkinleştirme/yükleme planlarına bu plugin'i dahil etmesi gereken rota türleri.                                                                                                              |
| `onConfigPaths`    | Hayır    | `string[]`                                           | Yol mevcutsa ve açıkça devre dışı bırakılmamışsa başlangıç/yükleme planlarına bu plugin'i dahil etmesi gereken köke göre yapılandırma yolları.                                               |
| `onCapabilities`   | Hayır    | `Array<"provider" \| "channel" \| "tool" \| "hook">` | Denetim düzlemi etkinleştirme planlamasında kullanılan geniş yetenek ipuçları. Mümkün olduğunda daha dar alanları tercih edin.                                                               |

Mevcut canlı tüketiciler:

- Gateway başlangıç planlaması, açık başlangıç içe aktarımı için `activation.onStartup` kullanır.
- Komutla tetiklenen CLI planlaması, eski `commandAliases[].cliCommand` veya `commandAliases[].name` seçeneğine geri döner.
- Aracı çalışma zamanı başlangıç planlaması, gömülü donanımlar için `activation.onAgentHarnesses`, CLI çalışma zamanı diğer adları için üst düzey `cliBackends[]` kullanır.
- Kanalla tetiklenen kurulum/kanal planlaması, açık kanal etkinleştirme meta verileri eksik olduğunda eski `channels[]` sahipliğine geri döner.
- Başlangıç plugin planlaması, paketlenmiş tarayıcı plugin'inin `browser` bloğu gibi kanal dışı kök yapılandırma yüzeyleri için `activation.onConfigPaths` kullanır.
- Sağlayıcıyla tetiklenen kurulum/çalışma zamanı planlaması, açık sağlayıcı etkinleştirme meta verileri eksik olduğunda eski `providers[]` ve üst düzey `cliBackends[]` sahipliğine geri döner.

Planlayıcı tanılamaları, açık etkinleştirme ipuçlarını manifest sahipliği geri dönüşünden ayırt edebilir. Örneğin `activation-command-hint`, `activation.onCommands` öğesinin eşleştiği anlamına gelirken `manifest-command-alias`, planlayıcının bunun yerine `commandAliases` sahipliğini kullandığı anlamına gelir. Bu neden etiketleri ana makine tanılamaları ve testler içindir; plugin yazarları sahipliği en iyi açıklayan meta verileri bildirmeye devam etmelidir.

## qaRunners başvurusu

Bir plugin, paylaşılan `openclaw qa` kökünün altında bir veya daha fazla aktarım çalıştırıcısı sağladığında
`qaRunners` kullanın. Bu meta verileri düşük maliyetli ve statik tutun; plugin
çalışma zamanı, eşleşen `qaRunnerCliRegistrations` öğelerini dışa aktaran hafif bir
`runtime-api.ts` yüzeyi aracılığıyla gerçek CLI kaydının sahibi olmaya devam eder. İsteğe
bağlı `adapterFactory`, kayıtlı komutun çalıştırıcısını değiştirmeden aktarımı paylaşılan QA senaryolarına açar.

```json
{
  "qaRunners": [
    {
      "commandName": "matrix",
      "description": "Docker destekli Matrix canlı QA hattını tek kullanımlık bir homeserver'a karşı çalıştır"
    }
  ]
}
```

| Alan          | Gerekli | Tür      | Anlamı                                                             |
| ------------- | -------- | -------- | ------------------------------------------------------------------ |
| `commandName` | Evet     | `string` | `openclaw qa` altına bağlanan alt komut; örneğin `matrix`.       |
| `description` | Hayır    | `string` | Paylaşılan ana makinenin yer tutucu bir komuta ihtiyaç duyduğunda kullandığı yedek yardım metni. |

`adapterFactory` kimliği `commandName` ile eşleşmelidir. Manifestte bulunmayan
komutlar için kayıtları dışa aktarmayın.

## setup başvurusu

Kurulum ve ilk katılım yüzeyleri, çalışma zamanı yüklenmeden önce düşük maliyetli, plugin'e ait meta verilere ihtiyaç duyduğunda `setup` kullanın.

```json
{
  "setup": {
    "providers": [
      {
        "id": "openai",
        "authMethods": ["api-key"],
        "envVars": ["OPENAI_API_KEY"],
        "authEvidence": [
          {
            "type": "local-file-with-env",
            "fileEnvVar": "OPENAI_CREDENTIALS_FILE",
            "requiresAllEnv": ["OPENAI_PROJECT"],
            "credentialMarker": "openai-local-credentials",
            "source": "openai yerel kimlik bilgileri"
          }
        ]
      }
    ],
    "cliBackends": ["openai-cli"],
    "configMigrations": ["legacy-openai-auth"],
    "requiresRuntime": false
  }
}
```

Üst düzey `cliBackends` geçerliliğini korur ve CLI çıkarım arka uçlarını açıklamaya devam eder. `setup.cliBackends`, yalnızca meta veri olarak kalması gereken denetim düzlemi/kurulum akışlarına yönelik, kuruluma özgü tanımlayıcı yüzeydir.

Mevcut olduklarında `setup.providers` ve `setup.cliBackends`, kurulum keşfi için tercih edilen, önce tanımlayıcıya dayalı arama yüzeyidir. Tanımlayıcı yalnızca aday plugin'i daraltıyorsa ve kurulum hâlâ daha zengin kurulum zamanı çalışma zamanı kancalarına ihtiyaç duyuyorsa `requiresRuntime: true` ayarlayın ve yedek yürütme yolu olarak `setup-api` öğesini yerinde tutun.

OpenClaw, genel sağlayıcı kimlik doğrulaması ve ortam değişkeni aramalarına `setup.providers[].envVars` öğesini dahil eder. Kurulum ve durum ortam meta verilerini buraya yerleştirin.

Bir faturalandırma veya kuruluş düzeyi kimlik bilgisinin, çıkarım kimlik bilgisine dönüşmeden `resolveUsageAuth` öğesini etkinleştirmesi gerektiğinde `providerUsageAuthEnvVars` kullanın. Bu adlar çalışma alanı dotenv engelleme, ACP alt süreçlerinden çıkarma, korumalı alan gizli bilgi filtreleme ve geniş kapsamlı gizli bilgi temizleme işlemlerine katılır. Sağlayıcı çalışma zamanı, değeri yine de `resolveUsageAuth` içinde okur ve sınıflandırır.

OpenClaw ayrıca, kurulum girdisi bulunmadığında veya `setup.requiresRuntime: false` kurulum çalışma zamanının gereksiz olduğunu bildirdiğinde `setup.providers[].authMethods` öğesinden basit kurulum seçenekleri türetebilir. Açık `providerAuthChoices` girdileri; özel etiketler, CLI bayrakları, ilk katılım kapsamı ve asistan meta verileri için tercih edilmeye devam eder.

`requiresRuntime: false` öğesini yalnızca bu tanımlayıcılar kurulum yüzeyi için yeterli olduğunda ayarlayın. OpenClaw, açık `false` öğesini yalnızca tanımlayıcı sözleşmesi olarak değerlendirir ve kurulum araması için `setup-api` veya `openclaw.setupEntry` öğesini yürütmez. Yalnızca tanımlayıcı kullanan bir plugin yine de bu kurulum çalışma zamanı girdilerinden birini sunuyorsa OpenClaw ek bir tanılama bildirir ve onu yok saymaya devam eder. `requiresRuntime` öğesinin belirtilmemesi eski geri dönüş davranışını korur; böylece bayrak olmadan tanımlayıcı eklemiş mevcut plugin'ler bozulmaz.

Kurulum araması plugin'e ait `setup-api` kodunu yürütebildiğinden, normalize edilmiş `setup.providers[].id` ve `setup.cliBackends[]` değerleri keşfedilen plugin'ler genelinde benzersiz kalmalıdır. Belirsiz sahiplik, keşif sırasından bir kazanan seçmek yerine kapalı şekilde başarısız olur.

Kurulum çalışma zamanı yürütüldüğünde, `setup-api` manifest tanımlayıcılarının bildirmediği bir sağlayıcıyı veya CLI arka ucunu kaydederse ya da bir tanımlayıcının eşleşen çalışma zamanı kaydı yoksa kurulum kayıt defteri tanılamaları tanımlayıcı sapmasını bildirir. Bu tanılamalar ek niteliktedir ve eski plugin'leri reddetmez.

### setup.providers başvurusu

| Alan           | Gerekli | Tür        | Anlamı                                                                                                 |
| -------------- | -------- | ---------- | ------------------------------------------------------------------------------------------------------ |
| `id`           | Evet     | `string`   | Kurulum veya ilk katılım sırasında sunulan sağlayıcı kimliği. Normalize edilmiş kimlikleri genel olarak benzersiz tutun. |
| `authMethods`  | Hayır    | `string[]` | Bu sağlayıcının tam çalışma zamanını yüklemeden desteklediği kurulum/kimlik doğrulama yöntemi kimlikleri.                 |
| `envVars`      | Hayır    | `string[]` | Genel kurulum/durum yüzeylerinin plugin çalışma zamanı yüklenmeden önce denetleyebileceği ortam değişkenleri.            |
| `authEvidence` | Hayır    | `object[]` | Gizli olmayan işaretçiler aracılığıyla kimlik doğrulaması yapabilen sağlayıcılar için düşük maliyetli yerel kimlik doğrulama kanıtı denetimleri. |

`authEvidence`, çalışma zamanı kodu yüklenmeden doğrulanabilen, sağlayıcıya ait yerel kimlik bilgisi işaretçileri içindir. Bu denetimler düşük maliyetli ve yerel kalmalıdır: ağ çağrıları, anahtarlık veya gizli bilgi yöneticisi okumaları, kabuk komutları ve sağlayıcı API yoklamaları olmamalıdır.

Desteklenen kanıt girdileri:

| Alan               | Gerekli | Tür        | Anlamı                                                                                                         |
| ------------------ | -------- | ---------- | -------------------------------------------------------------------------------------------------------------- |
| `type`             | Evet     | `string`   | Şu anda `local-file-with-env`.                                                                                 |
| `fileEnvVar`       | Hayır    | `string`   | Açık bir kimlik bilgisi dosyası yolu içeren ortam değişkeni.                                                    |
| `fallbackPaths`    | Hayır    | `string[]` | `fileEnvVar` yoksa veya boşsa denetlenen yerel kimlik bilgisi dosyası yolları. `${HOME}` ve `${APPDATA}` desteklenir. |
| `requiresAnyEnv`   | Hayır    | `string[]` | Kanıtın geçerli olması için listelenen ortam değişkenlerinden en az biri boş olmamalıdır.                       |
| `requiresAllEnv`   | Hayır    | `string[]` | Kanıtın geçerli olması için listelenen ortam değişkenlerinin tümü boş olmamalıdır.                              |
| `credentialMarker` | Evet     | `string`   | Kanıt mevcut olduğunda döndürülen gizli olmayan işaretçi.                                                       |
| `source`           | Hayır    | `string`   | Kimlik doğrulama/durum çıktısı için kullanıcıya yönelik kaynak etiketi.                                        |

### setup alanları

| Alan              | Gerekli | Tür       | Anlamı                                                                                       |
| ------------------ | -------- | ---------- | --------------------------------------------------------------------------------------------------- |
| `providers`        | Hayır       | `object[]` | Kurulum ve ilk yapılandırma sırasında sunulan sağlayıcı kurulum tanımlayıcıları.                                     |
| `cliBackends`      | Hayır       | `string[]` | Önce tanımlayıcı yaklaşımını kullanan kurulum araması için kurulum zamanı arka uç kimlikleri. Normalleştirilmiş kimlikleri genel olarak benzersiz tutun. |
| `configMigrations` | Hayır       | `string[]` | Bu Plugin'in kurulum yüzeyinin sahip olduğu yapılandırma taşıma kimlikleri.                                          |
| `requiresRuntime`  | Hayır       | `boolean`  | Tanımlayıcı aramasından sonra kurulumun hâlâ `setup-api` yürütmesini gerektirip gerektirmediği.                            |

## uiHints referansı

`uiHints`, yapılandırma alanı adlarını küçük işleme ipuçlarıyla eşleyen bir haritadır. Anahtarlar, iç içe yapılandırma alanları için nokta kullanabilir ancak hiçbir yol segmenti `__proto__`, `constructor` veya `prototype` olamaz; kurulum bu adları reddeder.

```json
{
  "uiHints": {
    "apiKey": {
      "label": "API anahtarı",
      "help": "OpenRouter istekleri için kullanılır",
      "placeholder": "sk-or-v1-...",
      "sensitive": true
    }
  }
}
```

Her alan ipucu şunları içerebilir:

| Alan          | Tür             | Anlamı                                                                                                     |
| -------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------- |
| `label`        | `string`         | Kullanıcıya gösterilen alan etiketi.                                                                                          |
| `help`         | `string`         | Kısa yardımcı metin.                                                                                                |
| `tags`         | `string[]`       | İsteğe bağlı kullanıcı arayüzü etiketleri.                                                                                                 |
| `advanced`     | `boolean`        | Alanı gelişmiş olarak işaretler.                                                                                      |
| `sensitive`    | `boolean`        | Alanı gizli veya hassas olarak işaretler.                                                                           |
| `placeholder`  | `string`         | Form girişleri için yer tutucu metin.                                                                                 |
| `presentation` | `"phone-number"` | Ayrıştırılabilir uluslararası (`+...`) değerler için yalnızca görüntülemeye yönelik yerelleştirilmiş telefon biçimlendirmesi; ham değerler değişmeden kalır. |

## contracts referansı

`contracts` öğesini yalnızca OpenClaw'un Plugin çalışma zamanını içe aktarmadan okuyabildiği statik yetenek sahipliği meta verileri için kullanın.

```json
{
  "contracts": {
    "agentToolResultMiddleware": ["openclaw", "codex"],
    "trustedToolPolicies": ["workflow-budget"],
    "externalAuthProviders": ["acme-ai"],
    "embeddingProviders": ["openai-compatible"],
    "speechProviders": ["openai"],
    "realtimeTranscriptionProviders": ["openai"],
    "realtimeVoiceProviders": ["openai"],
    "memoryEmbeddingProviders": ["local"],
    "mediaUnderstandingProviders": ["openai"],
    "imageGenerationProviders": ["openai"],
    "videoGenerationProviders": ["qwen"],
    "musicGenerationProviders": ["stability-audio"],
    "documentExtractors": ["example-docs"],
    "webContentExtractors": ["firecrawl"],
    "webFetchProviders": ["firecrawl"],
    "webSearchProviders": ["gemini"],
    "workerProviders": ["example-worker"],
    "usageProviders": ["acme-ai"],
    "migrationProviders": ["hermes"],
    "gatewayMethodDispatch": ["authenticated-request"],
    "tools": ["firecrawl_search", "firecrawl_scrape"]
  }
}
```

Her liste isteğe bağlıdır:

| Alan                            | Tür       | Anlamı                                                                                                                        |
| -------------------------------- | ---------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `embeddedExtensionFactories`     | `string[]` | Codex uygulama sunucusu uzantı fabrikası kimlikleri; şu anda `codex-app-server`.                                                                |
| `agentToolResultMiddleware`      | `string[]` | Bu Plugin'in araç sonucu ara yazılımı kaydedebileceği çalışma zamanı kimlikleri.                                                                     |
| `trustedToolPolicies`            | `string[]` | Yüklü bir Plugin'in kaydedebileceği, Plugin'e özgü güvenilir araç öncesi politika kimlikleri. Paketlenmiş Plugin'ler bu alan olmadan politika kaydedebilir. |
| `externalAuthProviders`          | `string[]` | Bu Plugin'in harici kimlik doğrulama profili kancasına sahip olduğu sağlayıcı kimlikleri.                                                                      |
| `embeddingProviders`             | `string[]` | Bellek dâhil yeniden kullanılabilir vektör gömme işlemleri için bu Plugin'in sahip olduğu genel gömme sağlayıcısı kimlikleri.                                 |
| `speechProviders`                | `string[]` | Bu Plugin'in sahip olduğu konuşma sağlayıcısı kimlikleri.                                                                                                |
| `realtimeTranscriptionProviders` | `string[]` | Bu Plugin'in sahip olduğu gerçek zamanlı transkripsiyon sağlayıcısı kimlikleri.                                                                                |
| `realtimeVoiceProviders`         | `string[]` | Bu Plugin'in sahip olduğu gerçek zamanlı ses sağlayıcısı kimlikleri.                                                                                        |
| `memoryEmbeddingProviders`       | `string[]` | Bu Plugin'in sahip olduğu, kullanımdan kaldırılmış belleğe özgü gömme sağlayıcısı kimlikleri.                                                                  |
| `mediaUnderstandingProviders`    | `string[]` | Bu Plugin'in sahip olduğu medya anlama sağlayıcısı kimlikleri.                                                                                   |
| `transcriptSourceProviders`      | `string[]` | Bu Plugin'in sahip olduğu transkript kaynağı sağlayıcısı kimlikleri.                                                                                     |
| `documentExtractors`             | `string[]` | Bu Plugin'in sahip olduğu belge (örneğin PDF) ayıklayıcı sağlayıcısı kimlikleri.                                                                  |
| `imageGenerationProviders`       | `string[]` | Bu Plugin'in sahip olduğu görüntü oluşturma sağlayıcısı kimlikleri.                                                                                      |
| `videoGenerationProviders`       | `string[]` | Bu Plugin'in sahip olduğu video oluşturma sağlayıcısı kimlikleri.                                                                                      |
| `musicGenerationProviders`       | `string[]` | Bu Plugin'in sahip olduğu müzik oluşturma sağlayıcısı kimlikleri.                                                                                      |
| `webContentExtractors`           | `string[]` | Bu Plugin'in sahip olduğu web sayfası içerik ayıklama sağlayıcısı kimlikleri.                                                                           |
| `webFetchProviders`              | `string[]` | Bu Plugin'in sahip olduğu web getirme sağlayıcısı kimlikleri.                                                                                             |
| `webSearchProviders`             | `string[]` | Bu Plugin'in sahip olduğu web arama sağlayıcısı kimlikleri.                                                                                            |
| `workerProviders`                | `string[]` | Sağlama ve profil destekli kiralama yaşam döngüsü için bu Plugin'in sahip olduğu bulut çalışanı sağlayıcısı kimlikleri.                                      |
| `usageProviders`                 | `string[]` | Bu Plugin'in kullanım kimlik doğrulaması ve kullanım anlık görüntüsü kancalarına sahip olduğu sağlayıcı kimlikleri.                                                             |
| `migrationProviders`             | `string[]` | Bu Plugin'in `openclaw migrate` için sahip olduğu içe aktarma sağlayıcısı kimlikleri.                                                                         |
| `gatewayMethodDispatch`          | `string[]` | Gateway yöntemlerini işlem içinde yönlendiren, kimliği doğrulanmış Plugin HTTP rotaları için ayrılmış yetkilendirme.                                  |
| `tools`                          | `string[]` | Bu Plugin'in sahip olduğu ajan aracı adları.                                                                                                   |

`contracts.embeddedExtensionFactories`, paketlenmiş ve yalnızca Codex uygulama sunucusuna yönelik uzantı fabrikaları için korunur. Paketlenmiş araç sonucu dönüşümleri bunun yerine `contracts.agentToolResultMiddleware` bildirmeli ve `api.registerAgentToolResultMiddleware(...)` ile kaydolmalıdır. Yüklü Plugin'ler aynı ara yazılım bağlantısını yalnızca açıkça etkinleştirildiğinde ve yalnızca `contracts.agentToolResultMiddleware` içinde bildirdikleri çalışma zamanları için kullanabilir.

Ana bilgisayar tarafından güvenilen araç öncesi politika katmanına ihtiyaç duyan yüklü Plugin'ler, kayıtlı her yerel kimliği `contracts.trustedToolPolicies` içinde bildirmeli ve açıkça etkinleştirilmelidir. Paketlenmiş Plugin'ler mevcut güvenilir politika yolunu korur ancak bildirilmemiş politika kimliklerine sahip yüklü Plugin'ler kayıttan önce reddedilir. Politika kimliklerinin kapsamı kaydeden Plugin ile sınırlıdır; bu nedenle iki Plugin de `workflow-budget` öğesini bildirip kaydedebilir ancak tek bir Plugin aynı yerel kimliği iki kez kaydedemez.

Çalışma zamanı `api.registerTool(...)` kayıtları `contracts.tools` ile eşleşmelidir. Araç keşfi, yalnızca istenen araçlara sahip olabilecek Plugin çalışma zamanlarını yüklemek için bu listeyi kullanır.

`resolveExternalAuthProfiles` uygulayan sağlayıcı Plugin'leri `contracts.externalAuthProviders` bildirmelidir; bildirilmemiş harici kimlik doğrulama kancaları yok sayılır.

Hem `resolveUsageAuth` hem de `fetchUsageSnapshot` uygulayan sağlayıcı Plugin'leri, otomatik olarak keşfedilen her sağlayıcı kimliğini `contracts.usageProviders` içinde bildirmelidir. Kullanım keşfi, çalışma zamanı kodunu yüklemeden önce bu sözleşmeyi okur ve ardından yalnızca bildirilen sahipleri yükledikten sonra her iki kancayı da doğrular.

Genel gömme sağlayıcıları, `api.registerEmbeddingProvider(...)` ile kaydedilen her bağdaştırıcı için `contracts.embeddingProviders` bildirmelidir. Bellek araması tarafından kullanılan sağlayıcılar dâhil, yeniden kullanılabilir vektör oluşturma için genel sözleşmeyi kullanın. `contracts.memoryEmbeddingProviders`, kullanımdan kaldırılmış belleğe özgü uyumluluktur ve yalnızca mevcut sağlayıcılar genel gömme sağlayıcısı bağlantısına geçerken korunur.

Çalışan sağlayıcıları, her `api.registerWorkerProvider(...)` kimliğini `contracts.workerProviders` içinde bildirmelidir. Çekirdek, `provision` çağrısından önce kalıcı amacı saklar; sağlayıcılar harici tahsisten önce ayarlarını doğrular ve aynı işlem kimliğiyle tekrarlanan çağrılar aynı kiralamayı benimsemelidir. Çekirdek ayrıca doğrulanmış ayar anlık görüntüsünü saklar ve adlandırılmış profil değiştirildikten veya kaldırıldıktan sonra bile `leaseId` ile birlikte `inspect({ leaseId, profile })` ve `destroy({ leaseId, profile })` öğelerine iletir. Yok etme işlemi eşgüçlüdür, inceleme kapalı `active` / `destroyed` / `unknown` durum birleşimini döndürür ve SSH özel anahtar malzemesine yalnızca `SecretRef` üzerinden başvurulur. Sağlanan SSH uç noktaları, çekirdeğin bağlanmadan önce ana bilgisayarı sabitleyebilmesi için güvenilir sağlama çıktısından, ana bilgisayar adı veya açıklama olmadan tam olarak `algorithm base64` biçiminde genel bir `hostKey` da içermelidir. Dinamik kimlik referansları oluşturan sağlayıcılar yetkili `resolveSshIdentity({ leaseId, profile, keyRef })` uygulayabilir; bunu uygulamayan sağlayıcılar çekirdeğin genel gizli bilgi çözümleyicisini kullanır. Yetkili bir `unknown`, etkin bir yerel kaydı sahipsiz bırakır; kalıcı bir yok etme isteğinden sonra kapatma işlemini doğrular.

`contracts.gatewayMethodDispatch` şu anda `"authenticated-request"` kabul eder. Bu, işlem içinde Gateway kontrol düzlemi yöntemlerini kasıtlı olarak yönlendiren yerel plugin HTTP rotaları için bir API hijyeni kapısıdır; kötü amaçlı yerel pluginlere karşı bir sandbox değildir. Bunu yalnızca zaten Gateway HTTP kimlik doğrulaması gerektiren, sıkı biçimde incelenmiş paketlenmiş/operatör yüzeyleri için kullanın. Yetkilendirilmiş bir rota, Gateway kök iş kabulü kapalıyken yalnızca ayrıca `auth: "gateway"` ve rotaya özgü `gatewayRuntimeScopeSurface: "trusted-operator"` bildirdiğinde erişilebilir kalır; aynı pluginden gelen sıradan eş rotalar kabul sınırının arkasında kalır. Bu, pluginin tamamına kabul atlama izni vermeden askıya alma durumu ve sürdürme işlevinin erişilebilir kalmasını sağlar. Ayrıştırma ve yanıt biçimlendirmeyi yönlendirme dışında sınırlı tutun; esaslı veya değişiklik yapan işler, kabul ve kapsam uygulamasının sahibi olan Gateway yöntemi yönlendirmesinden geçmelidir.

## configContracts referansı

Plugin çalışma zamanını içe aktarmadan genel çekirdek yardımcılarının ihtiyaç duyduğu manifestin sahip olduğu yapılandırma davranışı için `configContracts` kullanın: tehlikeli bayrak algılama, SecretRef geçiş hedefleri ve eski yapılandırma yolu daraltma.

```json
{
  "configContracts": {
    "compatibilityMigrationPaths": ["legacyProvider"],
    "compatibilityRuntimePaths": ["legacyProvider.webhook"],
    "dangerousFlags": [
      {
        "path": "accounts.*.allowUnverifiedSenders",
        "equals": true
      }
    ],
    "secretInputs": {
      "bundledDefaultEnabled": false,
      "paths": [
        {
          "path": "routes.*.secret",
          "expected": "string",
          "ownerKind": "route"
        }
      ]
    }
  }
}
```

| Alan                          | Zorunlu | Tür        | Anlamı                                                                                                                                                                                                                                  |
| ----------------------------- | ------- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `compatibilityMigrationPaths` | Hayır   | `string[]` | Bu pluginin kurulum zamanı uyumluluk geçişlerinin uygulanabileceğini belirten, köke göre yapılandırma yolları. Yapılandırma plugine hiç başvurmuyorsa genel çalışma zamanı yapılandırma okumalarının tüm plugin kurulum yüzeylerini atlamasını sağlar. |
| `compatibilityRuntimePaths`   | Hayır   | `string[]` | Plugin kodu tamamen etkinleşmeden önce bu pluginin çalışma zamanında işleyebileceği, köke göre uyumluluk yolları. Her uyumlu plugin çalışma zamanını içe aktarmadan paketlenmiş aday kümelerini daraltması gereken eski yüzeyler için bunu kullanın. |
| `dangerousFlags`              | Hayır   | `object[]` | Etkinleştirildiğinde `openclaw doctor` tarafından güvensiz veya tehlikeli olarak işaretlenmesi gereken yapılandırma sabit değerleri. Aşağıya bakın.                                                                                       |
| `secretInputs`                | Hayır   | `object`   | SecretRef geçişi, denetimi, başlangıçta somutlaştırma ve isteğe bağlı çalışma zamanı sahibi yalıtımı için `plugins.entries.<id>.config` altındaki yapılandırma yolları. Aşağıya bakın.                                                        |

Her `dangerousFlags` girdisi şunları destekler:

| Alan     | Zorunlu | Tür                                   | Anlamı                                                                                                                 |
| -------- | ------- | ------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `path`   | Evet    | `string`                              | `plugins.entries.<id>.config` öğesine göre noktayla ayrılmış yapılandırma yolu. Eşleme/dizi bölümleri için `*` joker karakterlerini destekler. |
| `equals` | Evet    | `string \| number \| boolean \| null` | Bu yapılandırma değerini tehlikeli olarak işaretleyen tam sabit değer.                                                  |

`secretInputs` şunları destekler:

| Alan                    | Zorunlu | Tür        | Anlamı                                                                                                                                                                                                                                                                                                                                                         |
| ----------------------- | ------- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bundledDefaultEnabled` | Hayır   | `boolean`  | Bu SecretRef yüzeyinin etkin olup olmadığına karar verirken paketlenmiş pluginin varsayılan etkinleştirme durumunu geçersiz kılar. Plugin paketlenmiş olduğu hâlde yüzeyin yapılandırmada açıkça etkinleştirilene kadar devre dışı kalması gerektiğinde bunu kullanın.                                                                                               |
| `paths`                 | Evet    | `object[]` | Her biri `path` (`plugins.entries.<id>.config` öğesine göre noktayla ayrılmıştır, `*` joker karakterlerini destekler), isteğe bağlı `expected` (şu anda yalnızca `"string"`) ve isteğe bağlı `ownerKind` (şu anda yalnızca `"route"`) içeren gizli bilgi biçimli yapılandırma yolları. Bildirilmiş bir sahip, çözümleme başarısız olduğunda yalnızca tam olarak eşleşen yolu yalıtır; sahip kimliği tam yapılandırma yoludur. |

## mediaUnderstandingProviderMetadata referansı

Bir medya anlama sağlayıcısının çalışma zamanı yüklenmeden önce genel çekirdek yardımcılarının ihtiyaç duyduğu varsayılan modelleri, otomatik kimlik doğrulama geri dönüş önceliği veya yerel belge desteği olduğunda `mediaUnderstandingProviderMetadata` kullanın. Anahtarlar ayrıca `contracts.mediaUnderstandingProviders` içinde bildirilmelidir.

```json
{
  "contracts": {
    "mediaUnderstandingProviders": ["example"]
  },
  "mediaUnderstandingProviderMetadata": {
    "example": {
      "capabilities": ["image", "audio"],
      "defaultModels": {
        "image": "example-vision-latest",
        "audio": "example-transcribe-latest"
      },
      "autoPriority": {
        "image": 40
      },
      "nativeDocumentInputs": ["pdf"],
      "documentModels": {
        "pdf": {
          "textExtraction": "example-doc-text-latest",
          "image": "example-doc-vision-latest"
        }
      }
    }
  }
}
```

Her sağlayıcı girdisi şunları içerebilir:

| Alan                   | Tür                                                              | Anlamı                                                                                                                |
| ---------------------- | ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `capabilities`         | `("image" \| "audio" \| "video")[]`                              | Bu sağlayıcı tarafından sunulan medya yetenekleri.                                                                    |
| `defaultModels`        | `Record<string, string>`                                         | Yapılandırma bir model belirtmediğinde kullanılan yetenek-model varsayılanları.                                       |
| `autoPriority`         | `Record<string, number>`                                         | Otomatik kimlik bilgisi tabanlı sağlayıcı geri dönüşünde düşük sayılar daha önce sıralanır.                            |
| `nativeDocumentInputs` | `"pdf"[]`                                                        | Sağlayıcı tarafından desteklenen yerel belge girdileri.                                                               |
| `documentModels`       | `{ pdf?: { textExtraction?: string; image?: string \| false } }` | Belge türüne göre model geçersiz kılmaları. Bu belge türü için görüntü tabanlı ayıklamayı devre dışı bırakmak üzere `image: false` değerini ayarlayın. |

## channelConfigs referansı

Bir kanal plugini çalışma zamanı yüklenmeden önce düşük maliyetli yapılandırma meta verilerine ihtiyaç duyduğunda `channelConfigs` kullanın. Salt okunur kanal kurulumu/durum keşfi, kurulum girdisi bulunmadığında veya `setup.requiresRuntime: false` kurulum çalışma zamanının gereksiz olduğunu bildirdiğinde yapılandırılmış harici kanallar için bu meta verileri doğrudan kullanabilir.

`channelConfigs`, yeni bir üst düzey kullanıcı yapılandırma bölümü değil, plugin manifesti meta verisidir. Kullanıcılar kanal örneklerini yine `channels.<channel-id>` altında yapılandırır. OpenClaw, plugin çalışma zamanı kodu yürütülmeden önce yapılandırılmış kanalın hangi plugine ait olduğuna karar vermek için manifest meta verilerini okur.

Bir kanal plugini için `configSchema` ve `channelConfigs` farklı yolları açıklar:

- `configSchema`, `plugins.entries.<plugin-id>.config` öğesini doğrular
- `channelConfigs.<channel-id>.schema`, `channels.<channel-id>` öğesini doğrular

`channels[]` bildiren paketlenmemiş pluginler, eşleşen `channelConfigs` girdilerini de bildirmelidir. Bunlar olmadan OpenClaw yine de plugini yükleyebilir; ancak soğuk yol yapılandırma şeması, kurulum ve Control UI yüzeyleri, plugin çalışma zamanı yürütülene kadar kanala ait seçenek biçimini veya yalnızca görüntülemeye yönelik kullanıcı arayüzü ipuçlarını bilemez.

`channelConfigs.<channel-id>.commands.nativeCommandsAutoEnabled` ve `nativeSkillsAutoEnabled`, kanal çalışma zamanı yüklenmeden önce çalışan komut yapılandırması denetimleri için statik `auto` varsayılanlarını bildirebilir. Paketlenmiş kanallar da aynı varsayılanları, pakete ait diğer kanal kataloğu meta verileriyle birlikte `package.json#openclaw.channel.commands` üzerinden yayımlayabilir.

```json
{
  "channelConfigs": {
    "matrix": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "homeserverUrl": { "type": "string" }
        }
      },
      "uiHints": {
        "homeserverUrl": {
          "label": "Ana Sunucu URL'si",
          "placeholder": "https://matrix.example.com"
        }
      },
      "label": "Matrix",
      "description": "Matrix ana sunucu bağlantısı",
      "commands": {
        "nativeCommandsAutoEnabled": true,
        "nativeSkillsAutoEnabled": true
      },
      "preferOver": ["matrix-legacy"]
    }
  }
}
```

Her kanal girdisi şunları içerebilir:

| Alan          | Tür                      | Anlamı                                                                                                                  |
| ------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------- |
| `schema`      | `object`                 | `channels.<id>` için JSON Schema. Bildirilen her kanal yapılandırma girdisi için zorunludur.                            |
| `uiHints`     | `Record<string, object>` | Bu kanal yapılandırma bölümü için isteğe bağlı etiketler, yer tutucular, hassasiyet ve yalnızca görüntülemeye yönelik sunum ipuçları. |
| `label`       | `string`                 | Çalışma zamanı meta verileri hazır olmadığında seçici ve inceleme yüzeylerine birleştirilen kanal etiketi.                |
| `description` | `string`                 | İnceleme ve katalog yüzeyleri için kısa kanal açıklaması.                                                                |
| `commands`    | `object`                 | Çalışma zamanı öncesi yapılandırma denetimleri için statik yerel komut ve yerel skill otomatik varsayılanları.             |
| `preferOver`  | `string[]`               | Bu kanalın seçim yüzeylerinde önüne geçmesi gereken eski veya daha düşük öncelikli plugin kimlikleri.                     |

### Başka bir kanal pluginini değiştirme

Plugininiz başka bir pluginin de sağlayabildiği bir kanal kimliği için tercih edilen sahip olduğunda `preferOver` kullanın. Yaygın durumlar; yeniden adlandırılmış bir plugin kimliği, paketlenmiş bir pluginin yerini alan bağımsız bir plugin veya yapılandırma uyumluluğu için aynı kanal kimliğini koruyan bakımlı bir fork olabilir.

```json
{
  "id": "acme-chat",
  "channels": ["chat"],
  "channelConfigs": {
    "chat": {
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "webhookUrl": { "type": "string" }
        }
      },
      "preferOver": ["chat"]
    }
  }
}
```

`channels.chat` yapılandırıldığında OpenClaw hem kanal kimliğini hem de tercih edilen plugin kimliğini dikkate alır. Daha düşük öncelikli plugin yalnızca paketle birlikte geldiği veya varsayılan olarak etkinleştirildiği için seçilmişse OpenClaw, kanalın ve araçlarının tek bir plugin tarafından yönetilmesi için bu plugini etkin çalışma zamanı yapılandırmasında devre dışı bırakır. Açık kullanıcı seçimi yine de önceliklidir: Kullanıcı her iki plugini de açıkça etkinleştirirse (`plugins.allow` veya esaslı bir `plugins.entries` yapılandırması aracılığıyla), OpenClaw bu seçimi korur ve istenen plugin kümesini sessizce değiştirmek yerine yinelenen kanal/araç tanılamalarını bildirir.

`preferOver` kapsamını gerçekten aynı kanalı sağlayabilen plugin kimlikleriyle sınırlı tutun. Bu genel bir öncelik alanı değildir ve kullanıcı yapılandırma anahtarlarını yeniden adlandırmaz.

## modelSupport referansı

OpenClaw'ın plugin çalışma zamanı yüklenmeden önce `gpt-5.6-sol` veya `claude-sonnet-4.6` gibi kısaltılmış model kimliklerinden sağlayıcı plugininizi çıkarsaması gerektiğinde `modelSupport` kullanın.

```json
{
  "modelSupport": {
    "modelPrefixes": ["gpt-", "o1", "o3", "o4"],
    "modelPatterns": ["^computer-use-preview"]
  }
}
```

OpenClaw şu öncelik sırasını uygular:

- açık `provider/model` referansları, sahip olan `providers` manifest meta verilerini kullanır
- `modelPatterns`, `modelPrefixes` öğelerinden önceliklidir
- paketle gelmeyen bir plugin ile paketle gelen bir plugin eşleşirse paketle gelmeyen plugin önceliklidir
- kalan belirsizlik, kullanıcı veya yapılandırma bir sağlayıcı belirtinceye kadar yok sayılır

Alanlar:

| Alan            | Tür        | Anlamı                                                                          |
| --------------- | ---------- | ------------------------------------------------------------------------------- |
| `modelPrefixes` | `string[]` | Kısaltılmış model kimliklerine karşı `startsWith` ile eşleştirilen ön ekler. |
| `modelPatterns` | `string[]` | Profil son eki kaldırıldıktan sonra kısaltılmış model kimliklerine karşı eşleştirilen regex kaynakları. |

`modelPatterns` girdileri, iç içe yineleme içeren kalıpları (örneğin `(a+)+$`) reddeden `compileSafeRegex` üzerinden derlenir. Güvenlik denetiminden geçemeyen kalıplar, sözdizimsel olarak geçersiz regex kalıpları gibi sessizce atlanır. Kalıpları basit tutun ve iç içe niceleyicilerden kaçının.

## modelCatalog referansı

OpenClaw'ın plugin çalışma zamanını yüklemeden önce sağlayıcı model meta verilerini bilmesi gerektiğinde `modelCatalog` kullanın. Bu, sabit katalog satırları, sağlayıcı takma adları, gizleme kuralları ve keşif modu için manifestin sahip olduğu kaynaktır. Çalışma zamanı yenilemesi yine sağlayıcı çalışma zamanı koduna aittir; ancak manifest, çekirdeğe çalışma zamanının ne zaman gerekli olduğunu bildirir.

```json
{
  "providers": ["openai"],
  "modelCatalog": {
    "providers": {
      "openai": {
        "baseUrl": "https://api.openai.com/v1",
        "api": "openai-responses",
        "models": [
          {
            "id": "gpt-5.4",
            "name": "GPT-5.4",
            "input": ["text", "image"],
            "reasoning": true,
            "contextWindow": 256000,
            "maxTokens": 128000,
            "cost": {
              "input": 1.25,
              "output": 10,
              "cacheRead": 0.125
            },
            "status": "available",
            "tags": ["default"]
          }
        ]
      }
    },
    "aliases": {
      "azure-openai-responses": {
        "provider": "openai",
        "api": "azure-openai-responses"
      }
    },
    "suppressions": [
      {
        "provider": "azure-openai-responses",
        "model": "gpt-5.3-codex-spark",
        "reason": "not available on Azure OpenAI Responses"
      }
    ],
    "discovery": {
      "openai": "static"
    }
  }
}
```

Üst düzey alanlar:

| Alan             | Tür                                                      | Anlamı                                                                                                      |
| ---------------- | -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `providers`      | `Record<string, object>`                                 | Bu pluginin sahip olduğu sağlayıcı kimliklerinin katalog satırları. Anahtarlar üst düzey `providers` içinde de bulunmalıdır. |
| `aliases`        | `Record<string, object>`                                 | Katalog veya gizleme planlaması için sahip olunan bir sağlayıcıya çözümlenmesi gereken sağlayıcı takma adları. |
| `suppressions`   | `object[]`                                               | Bu pluginin sağlayıcıya özgü bir nedenle gizlediği, başka bir kaynaktan gelen model satırları.               |
| `discovery`      | `Record<string, "static" \| "refreshable" \| "runtime">` | Sağlayıcı kataloğunun manifest meta verilerinden okunup okunamayacağı, önbelleğe yenilenip yenilenemeyeceği veya çalışma zamanı gerektirip gerektirmediği. |
| `runtimeAugment` | `boolean`                                                | Yalnızca sağlayıcı çalışma zamanının manifest/yapılandırma planlamasından sonra katalog satırları eklemesi gerektiğinde `true` olarak ayarlayın. |

`aliases`, model kataloğu planlaması için sağlayıcı sahipliği aramasına katılır. Takma ad hedefleri, aynı pluginin sahip olduğu üst düzey sağlayıcılar olmalıdır. Sağlayıcıya göre filtrelenmiş bir liste takma ad kullandığında OpenClaw, sağlayıcı çalışma zamanını yüklemeden sahip manifesti okuyabilir ve takma ad API/temel URL geçersiz kılmalarını uygulayabilir. Takma adlar filtrelenmemiş katalog listelerini genişletmez; geniş listeler yalnızca sahip olan kurallı sağlayıcının satırlarını yayımlar.

`suppressions`, eski sağlayıcı çalışma zamanı `suppressBuiltInModel` kancasının yerini alır. Gizleme girdileri yalnızca sağlayıcı pluginin sahipliğindeyse veya sahip olunan bir sağlayıcıyı hedefleyen bir `modelCatalog.aliases` anahtarı olarak bildirilmişse uygulanır. Model çözümlemesi sırasında çalışma zamanı gizleme kancaları artık çağrılmaz.

Sağlayıcı alanları:

| Alan                  | Tür                      | Anlamı                                                                                                                                                                                                            |
| --------------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `baseUrl`             | `string`                 | Bu sağlayıcı kataloğundaki modeller için isteğe bağlı varsayılan temel URL.                                                                                                                                        |
| `api`                 | `ModelApi`               | Bu sağlayıcı kataloğundaki modeller için isteğe bağlı varsayılan API bağdaştırıcısı.                                                                                                                               |
| `headers`             | `Record<string, string>` | Bu sağlayıcı kataloğuna uygulanan isteğe bağlı statik üstbilgiler.                                                                                                                                                 |
| `defaultUtilityModel` | `string`                 | Kısa dahili yardımcı görevler (başlıklar, ilerleme anlatımı) için sağlayıcının önerdiği isteğe bağlı küçük model kimliği. `agents.defaults.utilityModel` ayarlanmamışsa ve bu sağlayıcı aracının birincil modelini sunuyorsa kullanılır. |
| `models`              | `object[]`               | Gerekli model satırları. `id` içermeyen satırlar yok sayılır.                                                                                                                                       |

Model alanları:

| Alan               | Tür                                                            | Anlamı                                                                      |
| ------------------ | -------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `id`               | `string`                                                       | `provider/` ön eki olmadan, sağlayıcıya yerel model kimliği.                 |
| `name`             | `string`                                                       | İsteğe bağlı görünen ad.                                                     |
| `api`              | `ModelApi`                                                     | İsteğe bağlı model başına API geçersiz kılması.                              |
| `baseUrl`          | `string`                                                       | İsteğe bağlı model başına temel URL geçersiz kılması.                        |
| `headers`          | `Record<string, string>`                                       | İsteğe bağlı model başına statik üstbilgiler.                                |
| `input`            | `Array<"text" \| "image" \| "document">`                       | Modelin kabul ettiği kiplikler. Diğer değerler sessizce kaldırılır.          |
| `reasoning`        | `boolean`                                                      | Modelin akıl yürütme davranışı sunup sunmadığı.                               |
| `contextWindow`    | `number`                                                       | Sağlayıcının yerel bağlam penceresi.                                         |
| `contextTokens`    | `number`                                                       | `contextWindow` değerinden farklı olduğunda isteğe bağlı etkin çalışma zamanı bağlam sınırı. |
| `maxTokens`        | `number`                                                       | Biliniyorsa azami çıktı token sayısı.                                        |
| `thinkingLevelMap` | `Record<string, string \| null>`                               | Düşünme düzeyi başına isteğe bağlı model kimliği veya parametre geçersiz kılmaları. |
| `cost`             | `object`                                                       | İsteğe bağlı `tieredPricing` dahil, milyon token başına isteğe bağlı USD fiyatlandırması. |
| `compat`           | `object`                                                       | OpenClaw model yapılandırması uyumluluğuyla eşleşen isteğe bağlı uyumluluk bayrakları. |
| `mediaInput`       | `object`                                                       | Şu anda yalnızca görüntü için, kiplik başına isteğe bağlı girdi yapılandırması. |
| `status`           | `"available"` \| `"preview"` \| `"deprecated"` \| `"disabled"` | Listeleme durumu. Yalnızca satırın hiç görünmemesi gerekiyorsa gizleyin.     |
| `statusReason`     | `string`                                                       | Kullanılamaz durumuyla birlikte gösterilen isteğe bağlı neden.               |
| `replaces`         | `string[]`                                                     | Bu modelin yerini aldığı eski sağlayıcıya yerel model kimlikleri.            |
| `replacedBy`       | `string`                                                       | Kullanımdan kaldırılmış satırlar için sağlayıcıya yerel yedek model kimliği. |
| `tags`             | `string[]`                                                     | Seçiciler ve filtreler tarafından kullanılan kararlı etiketler.              |

Gizleme alanları:

| Alan                       | Tür        | Anlamı                                                                                                    |
| -------------------------- | ---------- | --------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`   | Gizlenecek üst kaynak satırının sağlayıcı kimliği. Bu plugin'e ait olmalı veya sahip olunan bir diğer ad olarak bildirilmelidir. |
| `model`                    | `string`   | Gizlenecek, sağlayıcıya özgü model kimliği.                                                                |
| `reason`                   | `string`   | Gizlenen satır doğrudan istendiğinde gösterilen isteğe bağlı mesaj.                                       |
| `when.baseUrlHosts`        | `string[]` | Gizlemenin uygulanabilmesi için bulunması gereken etkin sağlayıcı temel URL ana makinelerinin isteğe bağlı listesi. |
| `when.providerConfigApiIn` | `string[]` | Gizlemenin uygulanabilmesi için bulunması gereken tam sağlayıcı yapılandırması `api` değerlerinin isteğe bağlı listesi. |

Yalnızca çalışma zamanına ait verileri `modelCatalog` içine koymayın. `static` yalnızca manifest satırları, sağlayıcıya göre filtrelenen liste ve seçici yüzeylerinin kayıt defteri/çalışma zamanı keşfini atlamasına yetecek kadar eksiksiz olduğunda kullanılmalıdır. Manifest satırları listelenebilir başlangıç verileri veya eklemeler olarak yararlıysa ancak yenileme/önbellek daha sonra başka satırlar ekleyebiliyorsa `refreshable` kullanın; yenilenebilir satırlar tek başlarına yetkili değildir. OpenClaw'ın listeyi bilmek için sağlayıcı çalışma zamanını yüklemesi gerektiğinde `runtime` kullanın.

## modelIdNormalization başvurusu

Sağlayıcı çalışma zamanı yüklenmeden önce yapılması gereken, düşük maliyetli ve sağlayıcının sahip olduğu model kimliği temizliği için `modelIdNormalization` kullanın. Bu, kısa model adları, sağlayıcıya özgü eski kimlikler ve proxy ön ek kuralları gibi diğer adları temel model seçimi tabloları yerine sahibi olan plugin manifestinde tutar.

```json
{
  "providers": ["anthropic", "openrouter"],
  "modelIdNormalization": {
    "providers": {
      "anthropic": {
        "aliases": {
          "sonnet-4.6": "claude-sonnet-4-6"
        }
      },
      "openrouter": {
        "prefixWhenBare": "openrouter"
      }
    }
  }
}
```

Sağlayıcı alanları:

| Alan                                 | Tür                     | Anlamı                                                                                         |
| ------------------------------------ | ----------------------- | ---------------------------------------------------------------------------------------------- |
| `aliases`                            | `Record<string,string>` | Büyük/küçük harfe duyarsız tam model kimliği diğer adları. Değerler yazıldıkları biçimde döndürülür. |
| `stripPrefixes`                      | `string[]`              | Diğer ad aramasından önce kaldırılacak ön ekler; eski sağlayıcı/model yinelemeleri için yararlıdır. |
| `prefixWhenBare`                     | `string`                | Normalleştirilmiş model kimliği zaten `/` içermiyorsa eklenecek ön ek.                       |
| `prefixWhenBareAfterAliasStartsWith` | `object[]`              | Diğer ad aramasından sonra, `modelPrefix` ve `prefix` anahtarlarıyla belirlenen koşullu yalın kimlik ön eki kuralları. |

## providerEndpoints başvurusu

Genel istek politikasının sağlayıcı çalışma zamanı yüklenmeden önce bilmesi gereken uç nokta sınıflandırması için `providerEndpoints` kullanın. Her `endpointClass` değerinin anlamı yine temel sistemin sorumluluğundadır; ana makine ve temel URL meta verileri ise plugin manifestlerinin sorumluluğundadır.

Resmî olarak haricîleştirilmiş sağlayıcı plugin'leri temel dağıtıma dahil edilmez, bu nedenle
manifestleri yüklenene kadar görünmez. Uç nokta sınıflandırmasının plugin olmadan da
çalışmayı sürdürmesi için bunların `providerEndpoints` değerleri
`scripts/lib/official-external-provider-catalog.json` içinde de yansıtılmalıdır;
bir sözleşme testi bu yansıtmayı zorunlu kılar.

Uç nokta alanları:

| Alan                           | Tür        | Anlamı                                                                                              |
| ------------------------------ | ---------- | --------------------------------------------------------------------------------------------------- |
| `endpointClass`                | `string`   | `openrouter`, `moonshot-native` veya `google-vertex` gibi bilinen temel uç nokta sınıfı.          |
| `hosts`                        | `string[]` | Uç nokta sınıfıyla eşleşen tam ana makine adları.                                                    |
| `hostSuffixes`                 | `string[]` | Uç nokta sınıfıyla eşleşen ana makine son ekleri. Yalnızca etki alanı son ekiyle eşleştirme için `.` ön ekini kullanın. |
| `baseUrls`                     | `string[]` | Uç nokta sınıfıyla eşleşen, normalleştirilmiş tam HTTP(S) temel URL'leri.                            |
| `googleVertexRegion`           | `string`   | Tam küresel ana makineler için statik Google Vertex bölgesi.                                        |
| `googleVertexRegionHostSuffix` | `string`   | Google Vertex bölge ön ekini ortaya çıkarmak için eşleşen ana makinelerden çıkarılacak son ek.       |

## providerRequest başvurusu

Genel istek politikasının sağlayıcı çalışma zamanını yüklemeden ihtiyaç duyduğu düşük maliyetli istek uyumluluğu meta verileri için `providerRequest` kullanın. Davranışa özgü yük yeniden yazma işlemlerini sağlayıcı çalışma zamanı kancalarında veya paylaşılan sağlayıcı ailesi yardımcılarında tutun.

```json
{
  "providerRequest": {
    "providers": {
      "vllm": {
        "family": "vllm",
        "openAICompletions": {
          "supportsStreamingUsage": true
        }
      }
    }
  }
}
```

Sağlayıcı alanları:

| Alan                  | Tür          | Anlamı                                                                                       |
| --------------------- | ------------ | -------------------------------------------------------------------------------------------- |
| `family`              | `string`     | Genel istek uyumluluğu kararlarında ve tanılamada kullanılan sağlayıcı ailesi etiketi.        |
| `compatibilityFamily` | `"moonshot"` | Paylaşılan istek yardımcıları için isteğe bağlı sağlayıcı ailesi uyumluluk grubu.             |
| `openAICompletions`   | `object`     | OpenAI uyumlu tamamlama isteği bayrakları; şu anda `supportsStreamingUsage`.                 |

## secretProviderIntegrations başvurusu

Bir plugin yeniden kullanılabilir bir SecretRef exec sağlayıcı ön ayarı yayımlayabildiğinde `secretProviderIntegrations` kullanın. OpenClaw bu meta verileri plugin çalışma zamanı yüklenmeden önce okur, plugin sahipliğini `secrets.providers.<alias>.pluginIntegration` içinde saklar ve gerçek gizli değer çözümlemesini SecretRef çalışma zamanına bırakır. Ön ayarlar yalnızca paketlenmiş plugin'ler ve git ile ClawHub kurulumları gibi yönetilen plugin kurulum köklerinden keşfedilen yüklü plugin'ler için sunulur.

```json
{
  "secretProviderIntegrations": {
    "secret-store": {
      "providerAlias": "team-secrets",
      "displayName": "Team secrets",
      "source": "exec",
      "command": "${node}",
      "args": ["./bin/resolve-secrets.mjs"]
    }
  }
}
```

Eşleme anahtarı entegrasyon kimliğidir. `providerAlias` belirtilmezse OpenClaw, entegrasyon kimliğini SecretRef sağlayıcı diğer adı olarak kullanır. Sağlayıcı diğer adları normal SecretRef sağlayıcı diğer adı kalıbıyla eşleşmelidir; örneğin `team-secrets` veya `onepassword-work`.

Bir operatör ön ayarı seçtiğinde OpenClaw aşağıdakine benzer bir sağlayıcı başvurusu yazar:

```json
{
  "secrets": {
    "providers": {
      "team-secrets": {
        "source": "exec",
        "pluginIntegration": {
          "pluginId": "acme-secrets",
          "integrationId": "secret-store"
        }
      }
    }
  }
}
```

Başlatma/yeniden yükleme sırasında OpenClaw, güncel plugin manifesti meta verilerini yükleyerek, sahibi olan plugin'in yüklü ve etkin olduğunu denetleyerek ve exec komutunu manifestten somutlaştırarak bu sağlayıcıyı çözümler. Plugin'in devre dışı bırakılması veya kaldırılması, etkin SecretRef'ler için sağlayıcıyı geçersiz kılar. Bağımsız exec yapılandırması isteyen operatörler, elle `command`/`args` sağlayıcılarını doğrudan yazmaya devam edebilir.

Şu anda yalnızca `source: "exec"` ön ayarları desteklenir. `command`, `${node}` olmalı ve `args[0]`, plugin köküne göreli bir `./` çözümleyici betiği olmalıdır. OpenClaw bunu başlatma/yeniden yükleme sırasında geçerli Node yürütülebilir dosyasına ve plugin içindeki betiğin mutlak yoluna dönüştürür. `--require`, `--import`, `--loader`, `--env-file`, `--eval` ve `--print` gibi Node seçenekleri manifest ön ayarı sözleşmesinin parçası değildir. Node dışı komutlara ihtiyaç duyan operatörler, bağımsız elle yapılandırılmış exec sağlayıcılarını doğrudan yapılandırabilir.

OpenClaw, manifest ön ayarlarının `trustedDirs` değerini plugin kökünden ve `${node}` ön ayarları için geçerli Node yürütülebilir dosyasının dizininden türetir. Manifestte yazılan `trustedDirs` yok sayılır. `timeoutMs`, `noOutputTimeoutMs`, `maxOutputBytes`, `jsonOnly`, `env`, `passEnv` ve `allowInsecurePath` gibi diğer exec sağlayıcı seçenekleri normal SecretRef exec sağlayıcı yapılandırmasına aktarılır.

## modelPricing başvurusu

Bir sağlayıcının çalışma zamanı yüklenmeden önce kontrol düzlemi fiyatlandırma davranışına ihtiyaç duyması durumunda `modelPricing` kullanın. Gateway fiyatlandırma önbelleği, sağlayıcı çalışma zamanı kodunu içe aktarmadan bu meta verileri okur.

```json
{
  "providers": ["ollama", "openrouter"],
  "modelPricing": {
    "providers": {
      "ollama": {
        "external": false
      },
      "openrouter": {
        "openRouter": {
          "passthroughProviderModel": true
        },
        "liteLLM": false
      }
    }
  }
}
```

Sağlayıcı alanları:

| Alan         | Tür               | Anlamı                                                                                                      |
| ------------ | ----------------- | ----------------------------------------------------------------------------------------------------------- |
| `external`   | `boolean`         | OpenRouter veya LiteLLM fiyatlandırmasını hiçbir zaman almaması gereken yerel/kendi barındırılan sağlayıcılar için `false` olarak ayarlayın. |
| `openRouter` | `false \| object` | OpenRouter fiyatlandırma araması eşlemesi. `false`, bu sağlayıcı için OpenRouter aramasını devre dışı bırakır. |
| `liteLLM`    | `false \| object` | LiteLLM fiyatlandırma araması eşlemesi. `false`, bu sağlayıcı için LiteLLM aramasını devre dışı bırakır. |

Kaynak alanları:

| Alan                       | Tür                | Anlamı                                                                                                               |
| -------------------------- | ------------------ | -------------------------------------------------------------------------------------------------------------------- |
| `provider`                 | `string`           | OpenClaw sağlayıcı kimliğinden farklı olduğunda harici katalog sağlayıcı kimliği; örneğin bir `zai` sağlayıcısı için `z-ai`. |
| `passthroughProviderModel` | `boolean`          | Eğik çizgi içeren model kimliklerini iç içe sağlayıcı/model başvuruları olarak ele alır; OpenRouter gibi proxy sağlayıcılar için yararlıdır. |
| `modelIdTransforms`        | `"version-dots"[]` | Ek harici katalog model kimliği çeşitleri. `version-dots`, `claude-opus-4.6` gibi noktalı sürüm kimliklerini dener.       |

### OpenClaw Sağlayıcı Dizini

OpenClaw Sağlayıcı Dizini, plugin'leri henüz yüklenmemiş olabilecek sağlayıcılar için OpenClaw'a ait önizleme meta verileridir. Bir plugin manifestinin parçası değildir. Plugin manifestleri, yüklü plugin'ler için yetkili kaynak olmayı sürdürür. Sağlayıcı Dizini, bir sağlayıcı plugin'i yüklü olmadığında gelecekteki yüklenebilir sağlayıcı ve yükleme öncesi model seçici yüzeylerinin kullanacağı dahili geri dönüş sözleşmesidir.

Katalog yetki sırası:

1. Kullanıcı yapılandırması.
2. Yüklü plugin manifesti `modelCatalog`.
3. Açık yenilemeden gelen model kataloğu önbelleği.
4. OpenClaw Sağlayıcı Dizini önizleme satırları.

Provider Dizini; gizli bilgiler, etkin durumu, çalışma zamanı kancaları veya canlı hesaba özgü model verileri içermemelidir. Önizleme katalogları, plugin bildirimleriyle aynı `modelCatalog` sağlayıcı satırı biçimini kullanır; ancak `api`, `baseUrl`, fiyatlandırma veya uyumluluk bayrakları gibi çalışma zamanı bağdaştırıcısı alanları kurulu plugin bildirimiyle kasıtlı olarak uyumlu tutulmadıkça kararlı görüntüleme meta verileriyle sınırlı kalmalıdır. Canlı `/models` keşfine sahip sağlayıcılar, normal listeleme veya ilk katılım sırasında sağlayıcı API'lerini çağırmak yerine yenilenen satırları açık model kataloğu önbellek yolu üzerinden yazmalıdır.

Provider Dizini girdileri, plugini çekirdekten çıkarılmış veya henüz kurulmamış sağlayıcılar için kurulabilir plugin meta verileri de taşıyabilir. Bu meta veriler kanal kataloğu düzenini yansıtır: paket adı, npm kurulum tanımı, beklenen bütünlük ve basit kimlik doğrulama seçeneği etiketleri, kurulabilir bir yapılandırma seçeneğini göstermek için yeterlidir. Plugin kurulduktan sonra kendi bildirimi öncelik kazanır ve ilgili sağlayıcının Provider Dizini girdisi yok sayılır.

`openclaw doctor --fix`, eski üst düzey bildirim yeteneği anahtarlarından oluşan küçük ve kapalı bir kümeyi `contracts.*` içine taşır: `speechProviders`, `mediaUnderstandingProviders`, `imageGenerationProviders` ve `tools`. Bunların hiçbiri (veya başka herhangi bir yetenek listesi) artık üst düzey bildirim alanları olarak okunmaz; normal bildirim yükleme bunları yalnızca `contracts` altında tanır.

## Bildirim ile package.json karşılaştırması

İki dosya farklı görevler üstlenir:

| Dosya                   | Kullanım amacı                                                                                                                       |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.plugin.json` | Plugin kodu çalışmadan önce mevcut olması gereken keşif, yapılandırma doğrulaması, kimlik doğrulama seçeneği meta verileri ve kullanıcı arayüzü ipuçları                         |
| `package.json`         | npm meta verileri, bağımlılık kurulumu ve giriş noktaları, kurulum kısıtlaması, yapılandırma veya katalog meta verileri için kullanılan `openclaw` bloğu |

Bir meta veri parçasının nereye ait olduğundan emin değilseniz şu kuralı kullanın:

- OpenClaw'ın plugin kodunu yüklemeden önce bunu bilmesi gerekiyorsa `openclaw.plugin.json` içine koyun
- paketleme, giriş dosyaları veya npm kurulum davranışıyla ilgiliyse `package.json` içine koyun

### Keşfi etkileyen package.json alanları

Bazı çalışma zamanı öncesi plugin meta verileri, kasıtlı olarak `openclaw.plugin.json` yerine `package.json` içindeki `openclaw` bloğunda bulunur. `openclaw.bundle` ve `openclaw.bundle.json`, OpenClaw plugin sözleşmeleri değildir; yerel pluginler `openclaw.plugin.json` ile aşağıdaki desteklenen `package.json#openclaw` alanlarını kullanmalıdır.

Önemli örnekler:

| Alan                                                                                      | Anlamı                                                                                                                                                                             |
| ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw.extensions`                                                                      | Yerel plugin giriş noktalarını bildirir. Plugin paketi dizini içinde kalmalıdır.                                                                                                        |
| `openclaw.runtimeExtensions`                                                               | Kurulu paketler için derlenmiş JavaScript çalışma zamanı giriş noktalarını bildirir. Plugin paketi dizini içinde kalmalıdır.                                                                      |
| `openclaw.setupEntry`                                                                      | İlk katılım, ertelenmiş kanal başlatma ve salt okunur kanal durumu/SecretRef keşfi sırasında kullanılan hafif, yalnızca yapılandırmaya yönelik giriş noktasıdır. Plugin paketi dizini içinde kalmalıdır.      |
| `openclaw.runtimeSetupEntry`                                                               | Kurulu paketler için derlenmiş JavaScript yapılandırma giriş noktasını bildirir. `setupEntry` gerektirir, mevcut olmalı ve plugin paketi dizini içinde kalmalıdır.                              |
| `openclaw.channel`                                                                         | Etiketler, dokümantasyon yolları, diğer adlar ve seçim metni gibi basit kanal kataloğu meta verileridir.                                                                                                      |
| `openclaw.channel.approvalFlags`                                                           | Çalışma zamanı yüklenmeden önce kullanılabilen kapalı onay davranışı bayraklarıdır. `native`, kanalın yerel onay kullanıcı arayüzünü ve aynı turda çözümlemeyi yönettiği anlamına gelir.                                                |
| `openclaw.channel.commands`                                                                | Kanal çalışma zamanı yüklenmeden önce yapılandırma, denetim ve komut listesi yüzeyleri tarafından kullanılan statik yerel komut ve yerel beceri otomatik varsayılan meta verileridir.                                               |
| `openclaw.channel.cliAddOptions`                                                           | Pluginin yönettiği `openclaw channels add` seçenekleridir. Her girdi `flags`, `description`, isteğe bağlı `defaultValue` ve genel girdi türü dönüştürmesi için isteğe bağlı `valueType` (`int` veya `list`) bildirir. |
| `openclaw.channel.configuredState`                                                         | Tam kanal çalışma zamanını yüklemeden "yalnızca ortam üzerinden yapılan yapılandırma zaten mevcut mu?" sorusunu yanıtlayabilen hafif yapılandırılmış durum denetleyicisi meta verileridir.                                              |
| `openclaw.channel.persistedAuthState`                                                      | Tam kanal çalışma zamanını yüklemeden "herhangi bir yerde zaten oturum açılmış mı?" sorusunu yanıtlayabilen hafif kalıcı kimlik doğrulama denetleyicisi meta verileridir.                                                    |
| `openclaw.install.clawhubSpec` / `openclaw.install.npmSpec` / `openclaw.install.localPath` | Birlikte sunulan ve harici olarak yayımlanan pluginler için kurulum/güncelleme ipuçlarıdır.                                                                                                                        |
| `openclaw.install.defaultChoice`                                                           | Birden fazla kurulum kaynağı mevcut olduğunda tercih edilen kurulum yoludur.                                                                                                                       |
| `openclaw.install.minHostVersion`                                                          | `>=2026.3.22` veya `>=2026.5.1-beta.1` gibi bir semver alt sınırı kullanan, desteklenen en düşük OpenClaw ana makine sürümüdür.                                                                                  |
| `openclaw.compat.pluginApi`                                                                | Bu paketin gerektirdiği, `>=2026.5.27` gibi bir semver alt sınırı kullanan en düşük OpenClaw plugin API aralığıdır.                                                                                      |
| `openclaw.install.expectedIntegrity`                                                       | `sha512-...` gibi beklenen npm dağıtım bütünlüğü dizesidir; kurulum ve güncelleme akışları getirilen yapıtı buna göre doğrular.                                                                 |
| `openclaw.install.allowInvalidConfigRecovery`                                              | Yapılandırma geçersiz olduğunda birlikte sunulan plugin için dar kapsamlı bir yeniden kurulum kurtarma yoluna izin verir.                                                                                                            |
| `openclaw.install.requiredPlatformPackages`                                                | Kilit dosyasındaki platform kısıtlamaları mevcut ana makineyle eşleştiğinde somutlaştırılması gereken npm paket diğer adlarıdır.                                                                                |
| `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`                          | Yapılandırma çalışma zamanı kanal yüzeylerinin dinlemeden önce yüklenmesine izin verir, ardından tam yapılandırılmış kanal pluginini dinleme sonrası etkinleştirmeye erteler.                                                      |

Bildirim meta verileri, çalışma zamanı yüklenmeden önce ilk katılımda hangi sağlayıcı/kanal/yapılandırma seçeneklerinin görüneceğini belirler. `package.json#openclaw.install`, kullanıcı bu seçeneklerden birini belirlediğinde ilk katılıma ilgili pluginin nasıl getirileceğini veya etkinleştirileceğini bildirir. Kurulum ipuçlarını `openclaw.plugin.json` içine taşımayın.

`openclaw.channel.cliAddOptions` için `--initial-sync-limit <n>` gibi Commander uzun seçenek söz dizimini kullanın. Plugin yapılandırma bağdaştırıcısı değeri almadan önce negatif olmayan bir tam sayıyı ayrıştırmak için `valueType: "int"`, virgül, noktalı virgül veya yeni satırla ayrılmış girdiyi dizelere bölmek içinse `valueType: "list"` ayarlayın. Ayrıştırılmış Commander değerini değiştirmeden iletmek için `valueType` öğesini atlayın.

`openclaw.install.minHostVersion`, birlikte sunulmayan plugin kaynakları için kurulum ve bildirim kayıt defteri yüklemesi sırasında uygulanır. Geçersiz değerler reddedilir; daha yeni ancak geçerli değerler, eski ana makinelerde harici pluginlerin atlanmasına neden olur. Birlikte sunulan kaynak pluginlerin ana makine kod deposuyla aynı sürümde olduğu varsayılır.

`openclaw.install.requiredPlatformPackages`, gerekli yerel ikili dosyaları isteğe bağlı ve platforma özgü diğer adlar üzerinden sunan npm paketlerine yöneliktir. Desteklenen her platform diğer adı için yalın npm paket adını listeleyin. npm kurulumu sırasında OpenClaw yalnızca kilit dosyası kısıtlamaları mevcut ana makineyle eşleşen bildirilmiş diğer adı doğrular. npm başarı bildirdiği hâlde bu diğer adı dahil etmezse OpenClaw temiz bir önbellekle bir kez yeniden dener ve diğer ad hâlâ eksikse kurulumu geri alır.

`openclaw.compat.pluginApi`, birlikte sunulmayan plugin kaynakları için paket kurulumu sırasında uygulanır. Bunu, paketin temel aldığı en düşük OpenClaw plugin SDK/çalışma zamanı API sürümü için kullanın. Bir plugin paketi daha yeni bir API gerektirirken diğer akışlar için daha düşük bir kurulum ipucunu koruduğunda, `minHostVersion` değerinden daha katı olabilir. Resmî OpenClaw sürüm eşitlemesi, mevcut resmî plugin API alt sınırlarını varsayılan olarak OpenClaw sürümüne yükseltir; ancak yalnızca plugin içeren sürümler, paket kasıtlı olarak eski ana makineleri desteklediğinde daha düşük bir alt sınırı koruyabilir. Uyumluluk sözleşmesi olarak yalnızca paket sürümünü kullanmayın. `peerDependencies.openclaw`, npm paket meta verisi olarak kalır; OpenClaw, kurulum uyumluluğu kararları için `openclaw.compat.pluginApi` sözleşmesini kullanır.

Resmî isteğe bağlı kurulum meta verileri, plugin ClawHub'da yayımlandığında `clawhubSpec` kullanmalıdır; ilk katılım bunu tercih edilen uzak kaynak olarak değerlendirir ve kurulumdan sonra ClawHub yapıtı bilgilerini kaydeder. `npmSpec`, henüz ClawHub'a taşınmamış paketler için uyumluluk geri dönüşü olarak kalır.

Tam npm sürümü sabitlemesi zaten `npmSpec` içinde bulunur; örneğin `"npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3"`. Resmî harici katalog girdileri, tam tanımları `expectedIntegrity` ile eşleştirmelidir; böylece getirilen npm yapıtı artık sabitlenen sürümle eşleşmiyorsa güncelleme akışları güvenli biçimde başarısız olur. Etkileşimli ilk katılım, uyumluluk için yalın paket adları ve dağıtım etiketleri dahil güvenilir kayıt defteri npm tanımlarını sunmaya devam eder. Katalog tanılamaları; tam, değişken, bütünlüğü sabitlenmiş, bütünlüğü eksik, paket adı uyuşmazlığı bulunan ve geçersiz varsayılan seçim kaynaklarını ayırt edebilir. Ayrıca `expectedIntegrity` mevcut olduğu hâlde sabitleyebileceği geçerli bir npm kaynağı bulunmadığında uyarı verir. `expectedIntegrity` mevcut olduğunda kurulum/güncelleme akışları bunu zorunlu kılar; atlandığında kayıt defteri çözümlemesi bütünlük sabitlemesi olmadan kaydedilir.

Kanal pluginleri; durum, kanal listesi veya SecretRef taramalarının tam çalışma zamanını yüklemeden yapılandırılmış hesapları tanımlaması gerektiğinde `openclaw.setupEntry` sağlamalıdır. Yapılandırma girdisi, kanal meta verilerinin yanı sıra yapılandırmada güvenle kullanılabilen yapılandırma, durum ve gizli bilgi bağdaştırıcılarını sunmalıdır; ağ istemcilerini, Gateway dinleyicilerini ve aktarım çalışma zamanlarını ana uzantı giriş noktasında tutun.

Çalışma zamanı giriş noktası alanları, kaynak giriş noktası alanları için paket sınırı denetimlerini geçersiz kılmaz. Örneğin, `openclaw.runtimeExtensions` paket sınırının dışına çıkan bir `openclaw.extensions` yolunu yüklenebilir hâle getiremez.

`openclaw.install.allowInvalidConfigRecovery` kasıtlı olarak dar kapsamlıdır. İsteğe bağlı bozuk yapılandırmaların kurulabilmesini sağlamaz. Şu anda yalnızca eksik bir paketle birlikte gelen plugin yolu veya aynı paketle birlikte gelen plugin için eski bir `channels.<id>` girdisi gibi belirli eski paketle birlikte gelen plugin yükseltme hatalarından kurulum akışlarının kurtulmasına izin verir. İlgisiz yapılandırma hataları kurulumu engellemeye ve operatörleri `openclaw doctor --fix` komutuna yönlendirmeye devam eder.

`openclaw.channel.persistedAuthState`, küçük bir denetleyici modülü için paket meta verisidir:

```json
{
  "openclaw": {
    "channel": {
      "id": "whatsapp",
      "persistedAuthState": {
        "specifier": "./auth-presence",
        "exportName": "hasAnyWhatsAppAuth"
      }
    }
  }
}
```

Kurulum, doctor, durum veya salt okunur varlık akışlarının tam kanal plugini yüklenmeden önce düşük maliyetli bir evet/hayır kimlik doğrulama yoklamasına ihtiyaç duyduğu durumlarda bunu kullanın. Kalıcı kimlik doğrulama durumu, yapılandırılmış kanal durumu değildir: pluginleri otomatik olarak etkinleştirmek, çalışma zamanı bağımlılıklarını onarmak veya bir kanal çalışma zamanının yüklenip yüklenmeyeceğine karar vermek için bu meta veriyi kullanmayın. Hedef dışa aktarım, yalnızca kalıcı durumu okuyan küçük bir işlev olmalıdır; bunu tam kanal çalışma zamanı barrel'ı üzerinden yönlendirmeyin.

`openclaw.channel.configuredState`, düşük maliyetli yapılandırılma denetimlerini destekler. Ortam değişkenleri yeterli olduğunda bildirimsel ortam meta verisini tercih edin:

```json
{
  "openclaw": {
    "channel": {
      "id": "telegram",
      "configuredState": {
        "env": {
          "allOf": ["TELEGRAM_BOT_TOKEN"]
        }
      }
    }
  }
}
```

Listelenen her değişken gerektiğinde `env.allOf`, boş olmayan herhangi bir değişken yeterli olduğunda ise `env.anyOf` kullanın. Küçük ve çalışma zamanı dışı bir denetim, ortam meta verisinden daha fazlasına ihtiyaç duyuyorsa `persistedAuthState` için gösterildiği gibi `specifier` ile `exportName` kullanın; `env` mevcut olduğunda OpenClaw bunu ilgili modülü yüklemeden kullanır. Denetim tam yapılandırma çözümlemesine veya gerçek kanal çalışma zamanına ihtiyaç duyuyorsa bu mantığı plugin `config.hasConfiguredState` kancasında tutun.

## Keşif önceliği (yinelenen plugin kimlikleri)

OpenClaw pluginleri şu sırayla denetlenen üç kökten keşfeder: OpenClaw ile birlikte gönderilen paketle birlikte gelen pluginler, genel kurulum kökü (`~/.openclaw/extensions`) ve geçerli çalışma alanı kökü (`<workspace>/.openclaw/extensions`); bunlara ek olarak açıkça belirtilen `plugins.load.paths` girdileri.

İki keşif aynı `id` değerini paylaşıyorsa yalnızca **en yüksek öncelikli** manifest tutulur; daha düşük öncelikli yinelenenler onun yanında yüklenmek yerine atılır. En yüksekten en düşüğe öncelik sırası:

1. **Yapılandırmayla seçilen** — `plugins.entries.<id>` içinde açıkça sabitlenmiş bir yol
2. **İzlenen kurulum kaydıyla eşleşen genel kurulum** — kimlik paketle birlikte gelen bir plugine de ait olsa bile, OpenClaw'ın kurulum takibinin aynı kimlik için tanıdığı ve `openclaw plugin install`/`openclaw plugin update` aracılığıyla kurulmuş bir plugin
3. **Paketle birlikte gelen** — OpenClaw ile birlikte gönderilen pluginler
4. **Çalışma alanı** — geçerli çalışma alanına göre keşfedilen pluginler
5. Keşfedilen diğer tüm adaylar

Sonuçlar:

- Çalışma alanında veya genel kökte izlenmeden duran, paketle birlikte gelen bir pluginin çatallanmış ya da eski bir kopyası, paketle birlikte gelen derlemeyi gölgeleyemez.
- Paketle birlikte gelen bir plugini geçersiz kılmak için ya ilgili kimlik adına `openclaw plugin install` komutunu çalıştırarak izlenen genel kurulumun paketle birlikte gelen kopyadan daha yüksek öncelik kazanmasını sağlayın ya da `plugins.entries.<id>` aracılığıyla belirli bir yolu sabitleyerek yapılandırmayla seçilen öncelik sayesinde kazanmasını sağlayın.
- Yinelenenlerin atılması günlüğe kaydedilir; böylece Doctor ve başlangıç tanılamaları atılan kopyayı gösterebilir.
- Yapılandırmayla seçilen yinelenen geçersiz kılmalar, tanılamalarda açık geçersiz kılmalar olarak ifade edilir ancak eski çatalların ve yanlışlıkla oluşan gölgelemelerin görünür kalması için yine de uyarı verir.

## JSON Schema gereksinimleri

- **Her plugin, hiçbir yapılandırma kabul etmese bile bir JSON Schema ile gönderilmelidir.**
- Boş bir şema kabul edilebilir (örneğin, `{ "type": "object", "additionalProperties": false }`).
- Şemalar çalışma zamanında değil, yapılandırma okuma/yazma sırasında doğrulanır.
- Paketle birlikte gelen bir plugini yeni yapılandırma anahtarlarıyla genişletirken veya çatallarken aynı anda ilgili pluginin `openclaw.plugin.json` `configSchema` öğesini de güncelleyin. Paketle birlikte gelen plugin şemaları katıdır; bu nedenle `configSchema.properties` içine `myNewKey` eklemeden kullanıcı yapılandırmasına `plugins.entries.<id>.config.myNewKey` eklenmesi, plugin çalışma zamanı yüklenmeden önce reddedilir.

Örnek şema genişletmesi:

```json
{
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "myNewKey": {
        "type": "string"
      }
    }
  }
}
```

## Doğrulama davranışı

- Kanal kimliği bir plugin manifesti tarafından bildirilmediği sürece bilinmeyen `channels.*` anahtarları **hatadır**. Aynı kimlik `plugins.allow`, `plugins.entries` veya `plugins.installs` içinde de görünüyorsa (başvurulan ancak şu anda keşfedilemeyen bir plugin), OpenClaw bunu bunun yerine bir **uyarıya** indirger.
- Bilinmeyen plugin kimliklerine başvuran `plugins.entries.<id>`, `plugins.allow` ve `plugins.deny`, hata değil **uyarıdır** ("eski yapılandırma girdisi yok sayıldı"); böylece yükseltmeler ve kaldırılmış/yeniden adlandırılmış pluginler Gateway başlangıcını engellemez.
- Bilinmeyen bir plugin kimliğine başvuran `plugins.slots.memory`, uyarı veren bilinen resmî harici `memory-lancedb` plugini dışında bir **hatadır**.
- Bir plugin kurulmuş ancak manifesti veya şeması bozuk ya da eksikse doğrulama başarısız olur ve Doctor plugin hatasını bildirir.
- Plugin yapılandırması mevcut ancak plugin **devre dışıysa**, yapılandırma korunur ve Doctor ile günlüklerde bir **uyarı** gösterilir.

Tam `plugins.*` şeması için [Yapılandırma referansına](/tr/gateway/configuration) bakın.

## Notlar

- Manifest, yerel dosya sistemi yüklemeleri dâhil olmak üzere **yerel OpenClaw pluginleri için gereklidir**. Çalışma zamanı plugin modülünü yine ayrı olarak yükler; manifest yalnızca keşif ve doğrulama içindir.
- Yerel manifestler JSON5 ile ayrıştırılır; bu nedenle nihai değer yine bir nesne olduğu sürece yorumlar, sondaki virgüller ve tırnaksız anahtarlar kabul edilir.
- Manifest yükleyicisi yalnızca belgelenmiş manifest alanlarını okur. Özel üst düzey anahtarlardan kaçının.
- Bir plugin bunlara ihtiyaç duymadığında `channels`, `providers`, `cliBackends` ve `skills` alanlarının tümü atlanabilir.
- `providerCatalogEntry` hafif kalmalı ve geniş kapsamlı çalışma zamanı kodunu içe aktarmamalıdır; bunu istek zamanı yürütmesi için değil, statik sağlayıcı kataloğu meta verileri veya dar kapsamlı keşif tanımlayıcıları için kullanın.
- Münhasır plugin türleri `plugins.slots.*` üzerinden seçilir: `plugins.slots.memory` aracılığıyla `kind: "memory"` (varsayılan `memory-core`), `plugins.slots.contextEngine` aracılığıyla `kind: "context-engine"` (varsayılan `legacy`).
- Münhasır plugin türünü bu manifestte bildirin. Çalışma zamanı girdisi `OpenClawPluginDefinition.kind` kullanımdan kaldırılmıştır ve yalnızca eski pluginler için bir uyumluluk geri dönüşü olarak kalır.
- `setup.providers[].envVars` içindeki ortam değişkeni meta verisi yalnızca bildirimsel niteliktedir. Durum, denetim, Cron teslim doğrulaması ve diğer salt okunur yüzeyler, bir ortam değişkenini yapılandırılmış olarak değerlendirmeden önce yine de plugin güvenini ve etkin etkinleştirme politikasını uygular.
- Sağlayıcı kodu gerektiren çalışma zamanı sihirbazı meta verileri için [Sağlayıcı çalışma zamanı kancalarına](/tr/plugins/architecture-internals#provider-runtime-hooks) bakın.
- Plugininiz yerel modüllere bağımlıysa derleme adımlarını ve paket yöneticisi izin listesi gereksinimlerini (örneğin, pnpm `allow-build-scripts` + `pnpm rebuild <package>`) belgeleyin.

## İlgili

<CardGroup cols={3}>
  <Card title="Plugin oluşturma" href="/tr/plugins/building-plugins" icon="rocket">
    Pluginleri kullanmaya başlama.
  </Card>
  <Card title="Plugin mimarisi" href="/tr/plugins/architecture" icon="diagram-project">
    İç mimari ve yetenek modeli.
  </Card>
  <Card title="SDK'ya genel bakış" href="/tr/plugins/sdk-overview" icon="book">
    Plugin SDK referansı ve alt yol içe aktarımları.
  </Card>
</CardGroup>
