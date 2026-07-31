---
read_when:
    - Yeni bir model sağlayıcı plugini oluşturuyorsunuz
    - OpenClaw'a OpenAI uyumlu bir proxy veya özel LLM eklemek istiyorsunuz
    - Sağlayıcı kimlik doğrulamasını, katalogları ve çalışma zamanı kancalarını anlamanız gerekir
sidebarTitle: Provider plugins
summary: OpenClaw için model sağlayıcı plugini oluşturmaya yönelik adım adım kılavuz
title: Sağlayıcı pluginleri oluşturma
x-i18n:
    generated_at: "2026-07-26T23:34:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9d175fafc034bd52e996d47e047df104f079f2aba66662b22e8dbdf6c21e7e0
    source_path: plugins/sdk-provider-plugins.md
    workflow: 16
---

OpenClaw'a bir model sağlayıcısı (LLM) eklemek için sağlayıcı Plugin'i oluşturun: model
kataloğu, API anahtarıyla kimlik doğrulama ve dinamik model çözümleme.

<Info>
  OpenClaw Plugin'lerinde yeni misiniz? Paket yapısı ve manifest kurulumu için
  önce [Başlarken](/tr/plugins/building-plugins) bölümünü okuyun.
</Info>

<Tip>
  Sağlayıcı Plugin'leri, OpenClaw'ın normal çıkarım döngüsüne modeller ekler. Modelin
  iş parçacıklarını, Compaction'ı veya araç olaylarını yöneten yerel bir ajan daemon'ı
  üzerinden çalışması gerekiyorsa daemon protokolü ayrıntılarını çekirdeğe koymak yerine
  sağlayıcıyı bir [ajan çalışma düzeneği](/tr/plugins/sdk-agent-harness) ile eşleştirin.
</Tip>

## Adım adım açıklama

<Steps>
  <Step title="Paket ve manifest">
    ### 1. Adım: Paket ve manifest

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-ai",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "providers": ["acme-ai"],
        "compat": {
          "pluginApi": ">=2026.3.24-beta.2",
          "minGatewayVersion": "2026.3.24-beta.2"
        },
        "build": {
          "openclawVersion": "2026.3.24-beta.2",
          "pluginSdkVersion": "2026.3.24-beta.2"
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-ai",
      "name": "Acme AI",
      "description": "Acme AI model sağlayıcısı",
      "providers": ["acme-ai"],
      "modelSupport": {
        "modelPrefixes": ["acme-"]
      },
      "setup": {
        "providers": [
          {
            "id": "acme-ai",
            "envVars": ["ACME_AI_API_KEY"]
          }
        ]
      },
      "providerAuthAliases": {
        "acme-ai-coding": "acme-ai"
      },
      "providerAuthChoices": [
        {
          "provider": "acme-ai",
          "method": "api-key",
          "choiceId": "acme-ai-api-key",
          "choiceLabel": "Acme AI API anahtarı",
          "groupId": "acme-ai",
          "groupLabel": "Acme AI",
          "cliFlag": "--acme-ai-api-key",
          "cliOption": "--acme-ai-api-key <key>",
          "cliDescription": "Acme AI API anahtarı"
        }
      ],
      "configSchema": {
        "type": "object",
        "additionalProperties": false
      }
    }
    ```
    </CodeGroup>

    `setup.providers[].envVars`, OpenClaw'ın Plugin çalışma zamanınızı yüklemeden
    kimlik bilgilerini algılamasını sağlar. Bir sağlayıcı varyantının başka bir
    sağlayıcı kimliğinin kimlik doğrulamasını yeniden kullanması gerektiğinde
    `providerAuthAliases` ekleyin. `modelSupport` isteğe bağlıdır ve çalışma
    zamanı kancaları mevcut olmadan önce OpenClaw'ın sağlayıcı Plugin'inizi
    `acme-large` gibi kısaltılmış model kimliklerinden otomatik olarak
    yüklemesini sağlar. `package.json` içindeki `openclaw.compat`
    ve `openclaw.build`, ClawHub'da yayımlama için gereklidir
    (`openclaw.compat.pluginApi` ve `openclaw.build.openclawVersion` gerekli iki alandır;
    `minGatewayVersion` belirtilmediğinde `openclaw.install.minHostVersion` değerine geri döner).

  </Step>

  <Step title="Sağlayıcıyı kaydetme">
    Asgari bir metin sağlayıcısı için `id`, `label`, `auth` ve `catalog` gerekir.
    `catalog`, sağlayıcının sahip olduğu çalışma zamanı/yapılandırma kancasıdır;
    canlı satıcı API'lerini çağırabilir ve `models.providers` girdileri döndürür.

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import { createProviderApiKeyAuthMethod } from "openclaw/plugin-sdk/provider-auth";

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model provider",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          docsPath: "/providers/acme-ai",
          envVars: ["ACME_AI_API_KEY"],

          auth: [
            createProviderApiKeyAuthMethod({
              providerId: "acme-ai",
              methodId: "api-key",
              label: "Acme AI API key",
              hint: "API key from your Acme AI dashboard",
              optionKey: "acmeAiApiKey",
              flagName: "--acme-ai-api-key",
              envVar: "ACME_AI_API_KEY",
              promptMessage: "Enter your Acme AI API key",
              defaultModel: "acme-ai/acme-large",
            }),
          ],

          catalog: {
            order: "simple",
            run: async (ctx) => {
              const apiKey =
                ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: {
                  baseUrl: "https://api.acme-ai.com/v1",
                  apiKey,
                  api: "openai-completions",
                  models: [
                    {
                      id: "acme-large",
                      name: "Acme Large",
                      reasoning: true,
                      input: ["text", "image"],
                      cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
                      contextWindow: 200000,
                      maxTokens: 32768,
                    },
                    {
                      id: "acme-small",
                      name: "Acme Small",
                      reasoning: false,
                      input: ["text"],
                      cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
                      contextWindow: 128000,
                      maxTokens: 8192,
                    },
                  ],
                },
              };
            },
          },
        });

        api.registerModelCatalogProvider({
          provider: "acme-ai",
          kinds: ["text"],
          liveCatalog: async (ctx) => {
            const apiKey = ctx.resolveProviderApiKey("acme-ai").apiKey;
            if (!apiKey) return null;
            return [
              {
                kind: "text",
                provider: "acme-ai",
                model: "acme-large",
                label: "Acme Large",
                source: "live",
              },
            ];
          },
        });
      },
    });
    ```

    `registerModelCatalogProvider`; `text`, `voice`,
    `image_generation`, `video_generation` ve `music_generation` satırlarını
    kapsayan liste/yardım/seçici kullanıcı arayüzü için daha yeni denetim düzlemi
    katalog yüzeyidir. Satıcı uç noktası çağrılarını ve yanıt eşlemesini Plugin'de
    tutun; paylaşılan satır şekli, kaynak etiketleri ve yardım oluşturma OpenClaw'a aittir.

    Bu, çalışan bir sağlayıcıdır. Kullanıcılar artık `openclaw onboard --acme-ai-api-key <key>`
    komutunu çalıştırabilir ve model olarak `acme-ai/acme-large` seçebilir.

    ### Canlı model keşfi

    Sağlayıcınız OpenAI uyumlu bir `/models` API'si sunuyorsa tek
    sağlayıcılı yardımcıyı paylaşılan keşfe dahil edin:

    ```typescript
    catalog: {
      buildProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      buildStaticProvider: () => ({
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        models: [...STATIC_MODELS],
      }),
      liveModelDiscovery: true,
    },
    ```

    `liveModelDiscovery: true`, aşağıdaki davranışlara sahip genel bir Plugin SDK sözleşmesidir:

    | Alan | Sözleşme |
    | --- | --- |
    | Kimlik bilgileri | Keşif, kimlik doğrulama bir değer sağladığında `discoveryApiKey` değerini tercih ederek kataloğun çözümlenmiş sağlayıcı kimlik bilgisini kullanır. Gizli bilgi referansı işaretçileri hiçbir zaman belirteç olarak gönderilmez. Varsayılan istek `Authorization: Bearer <token>` kullanır; başka bir satıcı kimlik doğrulama şeması için `buildRequestHeaders` kullanın. |
    | Uç nokta | Varsayılan URL, `allowExplicitBaseUrl` etkinleştirildiğinde operatör geçersiz kılması da dahil olmak üzere etkin sağlayıcı `baseUrl` değerine göreli `models` değeridir. Başka bir göreli yol için `endpointPath` kullanın. `endpointUrl: { url, requireBaseUrl }` değerini yalnızca sabit bir satıcı URL'si için kullanın; etkin temel URL hâlâ `requireBaseUrl` değerine eşit olmadığı sürece keşif atlanır, böylece özel proxy kimlik bilgisi satıcıya gönderilmez. |
    | Ağ sınırları | Getirme işlemleri OpenClaw'ın SSRF korumasını, sayfalandırmanın tamamında tek bir 5 saniyelik zaman aşımı bütçesini, sayfa başına 4 MiB yanıt sınırını ve 50 sayfalık sınırı kullanır. Farklı kaynaklı sayfalandırma bağlantıları reddedilir; farklı kaynaklı bir yönlendirmeden sonra kimlik bilgileri kaldırılır. |
    | Önbellek | Başarılı ve boş olmayan kataloglar; sağlayıcı, uç nokta ve çözümlenmiş kimlik bilgisine göre 60 saniye boyunca önbelleğe alınır. Boş veya kullanılamaz sonuçlar önbelleğe alınmaz. |
    | Filtreleme | Tam eşleşen canlı kimlikler, güvenilir statik meta verilerini korur. Yeni satırlar ihtiyatlı biçimde metin/sohbet modelleri olarak yansıtılır. Devre dışı bırakılmış, arşivlenmiş, kullanımdan kaldırılmış, açıkça sohbet dışı, gömme, yeniden sıralama, moderasyon, konuşma, yalnızca görüntü ve yalnızca video satırları hariç tutulur. `readRows` değerini yalnızca standart olmayan bir yanıt zarfından satır seçmek için kullanın; sağlayıcıya özgü model semantiği yine özel bir kataloğa aittir. |
    | Hata | Canlı keşif tavsiye niteliğindedir. Kimlik doğrulama, ağ, zaman aşımı, sayfalandırma, ayrıştırma, boş katalog ve filtreleme hataları sağlayıcıyı kaldırmak yerine sağlayıcının sahip olduğu statik başlangıç kümesini döndürür. |

    Bearer kullanmayan veya standart dışı bir liste uç noktası için
    `true` yerine seçenekleri iletin:

    ```typescript
    liveModelDiscovery: {
      endpointPath: "model-catalog",
      buildRequestHeaders: ({ apiKey, discoveryApiKey }) => ({
        "vendor-version": "2026-01-01",
        "x-api-key": discoveryApiKey ?? apiKey ?? "",
      }),
      readRows: (body) =>
        body && typeof body === "object" &&
        Array.isArray((body as { models?: unknown }).models)
          ? (body as { models: unknown[] }).models
          : [],
    },
    ```

    `endpointUrl` değerini koşulsuz bir alternatif ana makine olarak kullanmayın.
    Bunun `requireBaseUrl` denetimi, model listesi ana makinesi çıkarım ana makinesinden
    farklı olan sağlayıcılar için kimlik bilgisi yalıtım sınırıdır.

    Sağlayıcı, ihtiyatlı OpenAI uyumlu yansıtma yerine özel model semantiğine
    ihtiyaç duyuyorsa bu yansıtmayı Plugin'de tutun ve paylaşılan getirme yaşam
    döngüsü için `openclaw/plugin-sdk/provider-catalog-live-runtime` kullanın. Yardımcı; sağlayıcı politikasını
    OpenClaw çekirdeğine koymadan korumalı HTTP getirmeleri, sağlayıcı kimlik doğrulama
    üst bilgileri, yapılandırılmış HTTP hataları, TTL önbelleğe alma ve statik geri
    dönüş davranışı sağlar.

    Canlı API yalnızca sağlayıcının sahip olduğu statik katalog satırlarından
    hangilerinin o anda kullanılabilir olduğunu bildiriyorsa `buildLiveModelProviderConfig` kullanın:

    ```typescript index.ts
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
    import {
      buildLiveModelProviderConfig,
      type LiveModelCatalogFetchGuard,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    const STATIC_MODELS = [
      {
        id: "acme-large",
        name: "Acme Large",
        reasoning: true,
        input: ["text", "image"],
        cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 3.75 },
        contextWindow: 200000,
        maxTokens: 32768,
      },
      {
        id: "acme-small",
        name: "Acme Small",
        reasoning: false,
        input: ["text"],
        cost: { input: 1, output: 5, cacheRead: 0.1, cacheWrite: 1.25 },
        contextWindow: 128000,
        maxTokens: 8192,
      },
    ] as const;

    async function buildAcmeLiveProvider(params: {
      apiKey: string;
      discoveryApiKey?: string;
      fetchGuard?: LiveModelCatalogFetchGuard;
    }) {
      return await buildLiveModelProviderConfig({
        providerId: "acme-ai",
        endpoint: "https://api.acme-ai.com/v1/models",
        providerConfig: {
          baseUrl: "https://api.acme-ai.com/v1",
          api: "openai-completions",
        },
        models: STATIC_MODELS,
        apiKey: params.apiKey,
        discoveryApiKey: params.discoveryApiKey,
        fetchGuard: params.fetchGuard,
        ttlMs: 60_000,
        auditContext: "acme-ai-model-discovery",
      });
    }

    export default definePluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      register(api) {
        api.registerProvider({
          id: "acme-ai",
          label: "Acme AI",
          catalog: {
            order: "simple",
            run: async (ctx) => {
              const auth = ctx.resolveProviderAuth("acme-ai");
              const apiKey =
                auth.apiKey ?? ctx.resolveProviderApiKey("acme-ai").apiKey;
              if (!apiKey) return null;
              return {
                provider: await buildAcmeLiveProvider({
                  apiKey,
                  discoveryApiKey: auth.discoveryApiKey,
                }),
              };
            },
          },
          staticCatalog: {
            order: "simple",
            run: async () => ({
              provider: {
                baseUrl: "https://api.acme-ai.com/v1",
                api: "openai-completions",
                models: [...STATIC_MODELS],
              },
            }),
          },
        });
      },
    });
    ```

    Sağlayıcı API'si daha zengin meta veriler döndürdüğünde ve Plugin'in
    satırları bizzat OpenClaw model tanımlarına dönüştürmesi gerektiğinde
    `getCachedLiveProviderModelRows` kullanın:

    ```typescript index.ts
    import {
      getCachedLiveProviderModelRows,
      LiveModelCatalogHttpError,
    } from "openclaw/plugin-sdk/provider-catalog-live-runtime";

    async function discoverAcmeModels(apiKey: string) {
      try {
        const rows = await getCachedLiveProviderModelRows({
          providerId: "acme-ai",
          endpoint: "https://api.acme-ai.com/v1/models",
          apiKey,
          ttlMs: 60_000,
          auditContext: "acme-ai-model-discovery",
        });
        return rows
          .map((row) => projectAcmeModel(row))
          .filter((model) => model !== null);
      } catch (error) {
        if (error instanceof LiveModelCatalogHttpError) {
          return STATIC_MODELS;
        }
        throw error;
      }
    }
    ```

    `run` kimlik doğrulama korumalı kalmalı ve kullanılabilir kimlik bilgisi
    olmadığında `null` döndürmelidir. Kurulum, belgeler,
    testler ve seçici yüzeylerinin canlı ağ erişimine bağlı olmaması için çevrimdışı bir
    `staticRun` veya statik geri dönüş bulundurun. Model listesi güncelliğine
    uygun bir TTL kullanın, istek sırasında dosya sistemi yoklamasından kaçının
    ve yalnızca üst sistem yanıtı OpenAI uyumlu bir `{ data: [{ id, object }] }`
    biçiminde değilse sağlayıcıya özgü bir `readRows` / `readModelId` iletin.

    Üst sistem sağlayıcısı OpenClaw'dan farklı kontrol belirteçleri kullanıyorsa akış
    yolunu değiştirmek yerine küçük, çift yönlü bir metin dönüşümü ekleyin:

    ```typescript
    api.registerTextTransforms({
      input: [
        { from: /red basket/g, to: "blue basket" },
        { from: /paper ticket/g, to: "digital ticket" },
        { from: /left shelf/g, to: "right shelf" },
      ],
      output: [
        { from: /blue basket/g, to: "red basket" },
        { from: /digital ticket/g, to: "paper ticket" },
        { from: /right shelf/g, to: "left shelf" },
      ],
    });
    ```

    `input`, aktarımdan önce son sistem istemini ve metin mesajı içeriğini
    yeniden yazar. `output`, OpenClaw kendi kontrol işaretleyicilerini ayrıştırmadan
    veya kanal teslimatı gerçekleşmeden önce asistan metin deltalarını ve son metni yeniden yazar.

    Yalnızca API anahtarıyla kimlik doğrulanan tek bir metin sağlayıcısını ve
    katalog destekli tek bir çalışma zamanını kaydeden paketlenmiş sağlayıcılar için
    daha dar kapsamlı `defineSingleProviderPluginEntry(...)` yardımcısını tercih edin:

    ```typescript
    import { defineSingleProviderPluginEntry } from "openclaw/plugin-sdk/provider-entry";

    export default defineSingleProviderPluginEntry({
      id: "acme-ai",
      name: "Acme AI",
      description: "Acme AI model sağlayıcısı",
      provider: {
        label: "Acme AI",
        docsPath: "/providers/acme-ai",
        auth: [
          {
            methodId: "api-key",
            label: "Acme AI API anahtarı",
            hint: "Acme AI kontrol panelinizdeki API anahtarı",
            optionKey: "acmeAiApiKey",
            flagName: "--acme-ai-api-key",
            envVar: "ACME_AI_API_KEY",
            promptMessage: "Acme AI API anahtarınızı girin",
            defaultModel: "acme-ai/acme-large",
          },
        ],
        catalog: {
          buildProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
          buildStaticProvider: () => ({
            api: "openai-completions",
            baseUrl: "https://api.acme-ai.com/v1",
            models: [{ id: "acme-large", name: "Acme Large" }],
          }),
        },
      },
    });
    ```

    `buildProvider`, OpenClaw gerçek sağlayıcı kimlik doğrulamasını çözümleyebildiğinde
    kullanılan canlı katalog yoludur. Sağlayıcıya özgü keşif yapabilir. Kimlik
    doğrulama yapılandırılmadan önce gösterilmesi güvenli olan çevrimdışı satırlar için
    yalnızca `buildStaticProvider` kullanın; kimlik bilgisi gerektirmemeli veya ağ isteği
    yapmamalıdır. OpenClaw'ın `models list --all` görünümü şu anda statik katalogları
    yalnızca paketlenmiş sağlayıcı Plugin'leri için; boş yapılandırma, boş ortam ve
    hiçbir ajan/çalışma alanı yolu olmadan çalıştırır.

    Kimlik doğrulama akışınızın ilk katılım sırasında ayrıca `models.providers.*`, takma
    adları ve ajanın varsayılan modelini yamaması gerekiyorsa `openclaw/plugin-sdk/provider-onboard`
    içindeki hazır ayar yardımcılarını kullanın. En dar kapsamlı yardımcılar
    `createDefaultModelPresetAppliers(...)`,
    `createDefaultModelsPresetAppliers(...)` ve
    `createModelCatalogPresetAppliers(...)` şeklindedir.

    Bir sağlayıcının yerel uç noktası normal `openai-completions` aktarımında akışlı
    kullanım bloklarını desteklediğinde sağlayıcı kimliği denetimlerini sabit kodlamak
    yerine `openclaw/plugin-sdk/provider-catalog-shared` içindeki ortak katalog yardımcılarını tercih edin.
    `supportsNativeStreamingUsageCompat(...)` ve `applyProviderNativeStreamingUsageCompat(...)`, desteği uç nokta yetenek haritasından
    algılar; böylece yerel Moonshot/DashScope tarzı uç noktalar, bir Plugin özel
    sağlayıcı kimliği kullandığında bile özelliği etkinleştirebilir.

    Yukarıdaki canlı keşif örnekleri `/models` tarzı sağlayıcı API'lerini kapsar.
    Bu keşfi kullanılabilir kimlik doğrulamayla korunan `catalog.run` içinde tutun
    ve çevrimdışı katalog üretimi için `staticRun` öğesini ağdan bağımsız tutun.

  </Step>

  <Step title="Dinamik model çözümleme ekleyin">
    Sağlayıcınız rastgele model kimliklerini kabul ediyorsa (bir proxy veya yönlendirici
    gibi), `resolveDynamicModel` ekleyin:

    ```typescript
    api.registerProvider({
      // ... id, label, auth, catalog from above

      resolveDynamicModel: (ctx) => ({
        id: ctx.modelId,
        name: ctx.modelId,
        provider: "acme-ai",
        api: "openai-completions",
        baseUrl: "https://api.acme-ai.com/v1",
        reasoning: false,
        input: ["text"],
        cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
        contextWindow: 128000,
        maxTokens: 8192,
      }),
    });
    ```

    Çözümleme bir ağ çağrısı gerektiriyorsa eşzamansız ısınma için `prepareDynamicModel`
    kullanın; tamamlandıktan sonra `resolveDynamicModel` yeniden çalışır.

  </Step>

  <Step title="Çalışma zamanı kancaları ekleyin (gerektiğinde)">
    Çoğu sağlayıcı yalnızca `catalog` + `resolveDynamicModel` gerektirir.
    Sağlayıcınız ihtiyaç duydukça kancaları aşamalı olarak ekleyin.

    Ortak yardımcı oluşturucular artık en yaygın yeniden oynatma/araç uyumluluğu
    ailelerini kapsadığından Plugin'lerin genellikle her kancayı tek tek elle
    bağlaması gerekmez:

    ```typescript
    import { buildProviderReplayFamilyHooks } from "openclaw/plugin-sdk/provider-model-shared";
    import { buildProviderStreamFamilyHooks } from "openclaw/plugin-sdk/provider-stream";
    import { buildProviderToolCompatFamilyHooks } from "openclaw/plugin-sdk/provider-tools";

    const GOOGLE_FAMILY_HOOKS = {
      ...buildProviderReplayFamilyHooks({ family: "google-gemini" }),
      ...buildProviderStreamFamilyHooks("google-thinking"),
      ...buildProviderToolCompatFamilyHooks("gemini"),
    };

    api.registerProvider({
      id: "acme-gemini-compatible",
      // ...
      ...GOOGLE_FAMILY_HOOKS,
    });
    ```

    Günümüzde kullanılabilen yeniden oynatma aileleri:

    | Aile | Bağladığı özellikler | Paketlenmiş örnekler |
    | --- | --- | --- |
    | `openai-compatible` | Araç çağrısı kimliği temizleme, asistan öncelikli sıralama düzeltmeleri ve aktarımın gerektirdiği durumlarda genel Gemini dönüşü doğrulaması dâhil olmak üzere OpenAI uyumlu aktarımlar için ortak OpenAI tarzı yeniden oynatma politikası | `moonshot`, `ollama`, `xai`, `zai` |
    | `anthropic-by-model` | `modelId` tarafından seçilen Claude uyumlu yeniden oynatma politikası; böylece Anthropic mesaj aktarımları, yalnızca çözümlenen model gerçekten bir Claude kimliği olduğunda Claude'a özgü düşünme bloğu temizliği alır | `amazon-bedrock` |
    | `native-anthropic-by-model` | `anthropic-by-model` ile aynı modele göre Claude politikası; ayrıca satıcıya özgü yerel kimlikleri koruması gereken aktarımlar için araç çağrısı kimliği temizleme ve yerel Anthropic araç kullanımı kimliğini koruma | `anthropic-vertex`, `clawrouter` |
    | `google-gemini` | Yerel Gemini yeniden oynatma politikası ve önyükleme yeniden oynatma temizliği. Ortak aile, metin çıktılı Gemini CLI'ı etiketli akıl yürütmede tutar; doğrudan `google` sağlayıcısı ise Gemini API düşünmesi yerel düşünce parçaları olarak geldiği için `resolveReasoningOutputMode` değerini `native` olarak geçersiz kılar. | `google`, `google-gemini-cli` |
    | `passthrough-gemini` | OpenAI uyumlu proxy aktarımları üzerinden çalışan Gemini modelleri için Gemini düşünce imzası temizliği; yerel Gemini yeniden oynatma doğrulamasını veya önyükleme yeniden yazımlarını etkinleştirmez | `openrouter`, `kilocode`, `opencode`, `opencode-go` |
    | `hybrid-anthropic-openai` | Tek bir Plugin'de Anthropic mesajlarıyla OpenAI uyumlu model yüzeylerini birleştiren sağlayıcılar için karma politika; isteğe bağlı, yalnızca Claude'a özgü düşünme bloğu kaldırma işlemi Anthropic tarafıyla sınırlı kalır | `minimax` |

    Günümüzde kullanılabilen akış aileleri:

    | Aile | Bağladığı öğeler | Birlikte sunulan örnekler |
    | --- | --- | --- |
    | `google-thinking` | Paylaşılan akış yolunda Gemini düşünme yükü normalleştirmesi | `google`, `google-gemini-cli` |
    | `kilocode-thinking` | Paylaşılan proxy akış yolunda Kilo akıl yürütme sarmalayıcısı; `kilo-auto/balanced` ve desteklenmeyen proxy akıl yürütme kimlikleri, eklenen düşünmeyi atlar | `kilocode` |
    | `moonshot-thinking` | Yapılandırma + `/think` düzeyinden Moonshot ikili yerel düşünme yükü eşlemesi | `moonshot` |
    | `minimax-fast-mode` | Paylaşılan akış yolunda MiniMax hızlı mod model yeniden yazımı | `minimax`, `minimax-portal` |
    | `openai-responses-defaults` | Paylaşılan yerel OpenAI/Codex Responses sarmalayıcıları: atıf üstbilgileri, `/fast`/`serviceTier`, metin ayrıntı düzeyi, yerel Codex web araması, akıl yürütme uyumluluğu yükü biçimlendirmesi ve Responses bağlam yönetimi | `openai` |
    | `openrouter-thinking` | Proxy rotaları için OpenRouter akıl yürütme sarmalayıcısı; desteklenmeyen model/`auto` atlamaları merkezi olarak işlenir | `openrouter` |
    | `tool-stream-default-on` | Açıkça devre dışı bırakılmadığı sürece araç akışı isteyen Z.AI gibi sağlayıcılar için varsayılan olarak etkin `tool_stream` sarmalayıcısı | `zai` |

    <Accordion title="Aile oluşturucularını destekleyen SDK bağlantı noktaları">
      Her aile oluşturucusu, aynı paketten dışa aktarılan daha düşük düzeyli genel yardımcılarla oluşturulur; bir sağlayıcının ortak kalıbın dışına çıkması gerektiğinde bunları kullanabilirsiniz:

      - `openclaw/plugin-sdk/provider-model-shared` - `ProviderReplayFamily`, `buildProviderReplayFamilyHooks(...)` ve ham yeniden oynatma oluşturucuları (`buildOpenAICompatibleReplayPolicy`, `buildAnthropicReplayPolicyForModel`, `buildGoogleGeminiReplayPolicy`, `buildHybridAnthropicOrOpenAIReplayPolicy`). Ayrıca Gemini yeniden oynatma yardımcılarını (`sanitizeGoogleGeminiReplayHistory`, `resolveTaggedReasoningOutputMode`) ve uç nokta/model yardımcılarını (`resolveProviderEndpoint`, `normalizeProviderId`, `normalizeGooglePreviewModelId`) dışa aktarır.
      - `openclaw/plugin-sdk/provider-stream` - `ProviderStreamFamily`, `buildProviderStreamFamilyHooks(...)`, `composeProviderStreamWrappers(...)`; ayrıca paylaşılan OpenAI/Codex sarmalayıcıları (`createOpenAIAttributionHeadersWrapper`, `createOpenAIFastModeWrapper`, `createOpenAIServiceTierWrapper`, `createOpenAIResponsesContextManagementWrapper`, `createCodexNativeWebSearchWrapper`), DeepSeek V4 OpenAI uyumlu sarmalayıcısı (`createDeepSeekV4OpenAICompatibleThinkingWrapper`), Anthropic Messages düşünme ön dolgu temizliği (`createAnthropicThinkingPrefillPayloadWrapper`), düz metin araç çağrısı uyumluluğu (`createPlainTextToolCallCompatWrapper`) ve paylaşılan proxy/sağlayıcı sarmalayıcıları (`createOpenRouterWrapper`, `createToolStreamWrapper`, `createMinimaxFastModeWrapper`).
      - `openclaw/plugin-sdk/provider-stream-shared` - `createOpenAICompatibleCompletionsThinkingOffWrapper`, `createPayloadPatchStreamWrapper`, `createPlainTextToolCallCompatWrapper`, `normalizeOpenAICompatibleReasoningPayload(...)` ve `setQwenChatTemplateThinking(...)` dâhil olmak üzere yoğun kullanılan sağlayıcı yolları için hafif yük ve olay sarmalayıcıları.
      - `openclaw/plugin-sdk/provider-tools` - `ProviderToolCompatFamily`, `buildProviderToolCompatFamilyHooks("deepseek" | "gemini" | "openai")` ve temel sağlayıcı şeması yardımcıları.

      Gemini ailesi sağlayıcılarında akıl yürütme çıktısı modunu
      aktarımla uyumlu tutun. Doğrudan Google Gemini API sağlayıcıları, OpenClaw'ın
      `<think>` / `<final>` istem yönergeleri eklemeden yerel düşünce parçalarını
      tüketebilmesi için `native` akıl yürütme çıktısını kullanmalıdır. Son bir
      JSON/metin yanıtını ayrıştıran, yalnızca metin kullanan Gemini CLI tarzı arka uçlar,
      paylaşılan `google-gemini` etiketli sözleşmeyi kullanmaya devam edebilir.

      Bazı akış yardımcıları bilerek sağlayıcıya özgü tutulur. `@openclaw/anthropic-provider`; Claude OAuth beta işlemesini ve `context1m` geçitlemesini kodladıkları için `wrapAnthropicProviderStream`, `resolveAnthropicBetas`, `resolveAnthropicFastMode`, `resolveAnthropicServiceTier` ve daha düşük düzeyli Anthropic sarmalayıcı oluşturucularını kendi genel `api.ts` / `contract-api.ts` bağlantı noktasında tutar. xAI plugini de benzer biçimde yerel xAI Responses biçimlendirmesini kendi `wrapStreamFn` öğesinde tutar (`/fast` diğer adları, varsayılan `tool_stream`, desteklenmeyen katı araç temizliği, xAI'ye özgü akıl yürütme yükü kaldırma).

      Aynı paket kökü kalıbı ayrıca `@openclaw/openai-provider` (sağlayıcı oluşturucuları, varsayılan model yardımcıları, gerçek zamanlı sağlayıcı oluşturucuları) ve `@openclaw/openrouter-provider` (sağlayıcı oluşturucusu ile ilk katılım/yapılandırma yardımcıları) öğelerini destekler.
    </Accordion>

    <Tabs>
      <Tab title="Token değişimi">
        Her çıkarım çağrısından önce token değişimi gerektiren sağlayıcılar için:

        ```typescript
        prepareRuntimeAuth: async (ctx) => {
          const exchanged = await exchangeToken(ctx.apiKey);
          return {
            apiKey: exchanged.token,
            baseUrl: exchanged.baseUrl,
            expiresAt: exchanged.expiresAt,
          };
        },
        ```
      </Tab>
      <Tab title="Özel üstbilgiler">
        Özel istek üstbilgileri veya gövde değişiklikleri gerektiren sağlayıcılar için:

        ```typescript
        // wrapStreamFn, ctx.streamFn'den türetilmiş bir StreamFn döndürür
        wrapStreamFn: (ctx) => {
          if (!ctx.streamFn) return undefined;
          const inner = ctx.streamFn;
          return async (params) => {
            params.headers = {
              ...params.headers,
              "X-Acme-Version": "2",
            };
            return inner(params);
          };
        },
        ```
      </Tab>
      <Tab title="Yerel aktarım kimliği">
        Genel HTTP veya WebSocket aktarımlarında yerel istek/oturum
        üstbilgilerine ya da meta verilere ihtiyaç duyan sağlayıcılar için:

        ```typescript
        resolveTransportTurnState: (ctx) => ({
          headers: {
            "x-request-id": ctx.turnId,
          },
          metadata: {
            session_id: ctx.sessionId ?? "",
            turn_id: ctx.turnId,
          },
        }),
        resolveWebSocketSessionPolicy: (ctx) => ({
          headers: {
            "x-session-id": ctx.sessionId ?? "",
          },
          degradeCooldownMs: 60_000,
        }),
        ```
      </Tab>
      <Tab title="Kullanım ve faturalandırma">
        Kullanım/faturalandırma verilerini sunan sağlayıcılar için:

        ```typescript
        resolveUsageAuth: async (ctx) => {
          const auth = await ctx.resolveOAuthToken();
          return auth ? { token: auth.token } : null;
        },
        fetchUsageSnapshot: async (ctx) => {
          return await fetchAcmeUsage(ctx.token, ctx.timeoutMs);
        },
        ```

        `resolveUsageAuth` üç sonuca sahiptir. Sağlayıcının bir
        kullanım/faturalandırma kimlik bilgisi olduğunda `{ token, accountId?, subscriptionType?, rateLimitTier? }`
        döndürün (isteğe bağlı alanlar, çözümlenen profildeki gizli olmayan plan
        meta verilerini `fetchUsageSnapshot` öğesine taşır).
        Yalnızca sağlayıcı kullanım kimlik doğrulamasını kesin olarak işlediyse
        ancak kullanılabilir bir kullanım tokenına sahip değilse ve OpenClaw'ın
        genel API anahtarı/OAuth geri dönüşünü atlaması gerekiyorsa
        `{ handled: true }` döndürün. Sağlayıcı isteği işlemediyse ve OpenClaw'ın
        genel geri dönüşle devam etmesi gerekiyorsa `null` veya
        `undefined` döndürün.

        Sağlayıcı kimliğini `contracts.usageProviders` içinde bildirin. Bu manifest
        sözleşmesi ve **her iki** kanca mevcut olduğunda OpenClaw, ilgisiz sağlayıcı
        pluginlerini yüklemeden sağlayıcıyı otomatik olarak kullanım toplamaya
        dâhil eder. Çekirdek izin listesinin güncellenmesi gerekmez.
        `fetchUsageSnapshot`, paylaşılan ve sağlayıcıdan bağımsız biçimi döndürür:

        - `plan`: sağlayıcı tarafından bildirilen abonelik veya anahtar etiketi
        - `windows`: kullanılan yüzdeler biçiminde sıfırlanabilir kota aralıkları
        - `billing`: türü belirlenmiş `balance`, `spend` veya `budget` girdileri; `unit`,
          bir ISO para birimi veya `credits` gibi bir sağlayıcı birimi olabilir
        - `summary`: bu yapılandırılmış alanlara sığmayan, sağlayıcıya özgü kısa bağlam

        Para birimi anlamını aynen koruyun. Üst sistem sözleşmesi aksini
        belirtmedikçe bir sağlayıcı kredisi USD değildir. Yalnızca
        `fetchUsageSnapshot` uygulayan bir plugin, açık/sentetik çağıranlar için
        kullanılabilir olmaya devam eder ancak OpenClaw kullanım kimlik bilgisini
        çözümleyemediğinden otomatik olarak keşfedilmez.
      </Tab>
    </Tabs>

    <Accordion title="Yaygın sağlayıcı kancaları">
      OpenClaw, model/sağlayıcı pluginleri için kancaları yaklaşık olarak bu sırayla çağırır.
      Çoğu sağlayıcı yalnızca 2-3 tanesini kullanır. Bu, `ProviderPlugin`
      sözleşmesinin tamamı değildir; eksiksiz ve güncel kanca listesi ile geri dönüş
      notları için [İç Yapı: Sağlayıcı Çalışma Zamanı
      Kancaları](/tr/plugins/architecture-internals#provider-runtime-hooks) bölümüne bakın.
      `ProviderPlugin.capabilities` ve `suppressBuiltInModel` gibi OpenClaw'ın artık
      çağırmadığı, yalnızca uyumluluk amaçlı sağlayıcı alanları burada listelenmez.

      | Kanca | Ne zaman kullanılmalı |
      | --- | --- |
      | `catalog` | Model kataloğu veya temel URL varsayılanları |
      | `applyConfigDefaults` | Yapılandırma somutlaştırması sırasında sağlayıcıya ait genel varsayılanlar |
      | `normalizeModelId` | Aramadan önce eski/önizleme model kimliği diğer adlarını temizleme |
      | `normalizeTransport` | Genel model birleştirmesinden önce sağlayıcı ailesine özgü `api` / `baseUrl` temizliği |
      | `normalizeConfig` | `models.providers.<id>` yapılandırmasını normalleştirme |
      | `applyNativeStreamingUsageCompat` | Yapılandırma sağlayıcıları için yerel akış kullanımı uyumluluk yeniden yazımları |
      | `resolveConfigApiKey` | Sağlayıcıya ait ortam işaretleyicisi kimlik doğrulama çözümlemesi |
      | `resolveSyntheticAuth` | Yerel/kendi kendine barındırılan veya yapılandırma destekli sentetik kimlik doğrulama |
      | `resolveExternalAuthProfiles` | CLI/uygulama tarafından yönetilen kimlik bilgileri için sağlayıcıya ait harici kimlik doğrulama profillerini katmanlama |
      | `shouldDeferSyntheticProfileAuth` | Ortam/yapılandırma kimlik doğrulamasının arkasındaki sentetik saklanan profil yer tutucularını daha düşük önceliğe alma |
      | `resolveDynamicModel` | Rastgele üst sistem model kimliklerini kabul etme |
      | `prepareDynamicModel` | Çözümlemeden önce eşzamansız meta veri getirme |
      | `normalizeResolvedModel` | Çalıştırıcıdan önce aktarım yeniden yazımları |
      | `normalizeToolSchemas` | Kaydetmeden önce sağlayıcıya ait araç şeması temizliği |
      | `inspectToolSchemas` | Sağlayıcıya ait araç şeması tanılamaları |
      | `resolveReasoningOutputMode` | Etiketli ve yerel akıl yürütme çıktısı sözleşmesi |
      | `prepareExtraParams` | Varsayılan istek parametreleri |
      | `createStreamFn` | Tamamen özel StreamFn aktarımı |
      | `wrapStreamFn` | Normal akış yolunda özel üstbilgi/gövde sarmalayıcıları |
      | `resolveTransportTurnState` | Her tur için yerel üstbilgiler/meta veriler |
      | `resolveWebSocketSessionPolicy` | Yerel WS oturum üstbilgileri/bekleme süresi |
      | `formatApiKey` | Özel çalışma zamanı token biçimi |
      | `refreshOAuth` | Özel OAuth yenileme |
      | `buildAuthDoctorHint` | Kimlik doğrulama onarım rehberliği |
      | `matchesContextOverflowError` | Sağlayıcıya ait taşma algılama |
      | `classifyFailoverReason` | Sağlayıcıya ait hız sınırı/aşırı yük sınıflandırması |
      | `isCacheTtlEligible` | İstem önbelleği TTL geçitlemesi |
      | `buildMissingAuthMessage` | Özel eksik kimlik doğrulama ipucu |
      | `augmentModelCatalog` | Sentetik ileriye dönük uyumluluk satırları (kullanımdan kaldırıldı - `registerModelCatalogProvider` tercih edin) |
      | `resolveThinkingProfile` | Modele özgü `/think` seçenek kümesi |
      | `isBinaryThinking` | İkili düşünmeyi açma/kapatma uyumluluğu (kullanımdan kaldırıldı - `resolveThinkingProfile` tercih edin) |
      | `supportsXHighThinking` | `xhigh` akıl yürütme desteği uyumluluğu (kullanımdan kaldırıldı - `resolveThinkingProfile` tercih edin) |
      | `resolveDefaultThinkingLevel` | Varsayılan `/think` ilkesi uyumluluğu (kullanımdan kaldırıldı - `resolveThinkingProfile` tercih edin) |
      | `isModernModelRef` | Canlı/duman testi modeli eşleştirme |
      | `prepareRuntimeAuth` | Çıkarımdan önce token değişimi |
      | `resolveUsageAuth` | Özel kullanım kimlik bilgisi ayrıştırma |
      | `fetchUsageSnapshot` | Özel kullanım uç noktası |
      | `createEmbeddingProvider` | Bellek/arama için sağlayıcıya ait gömme bağdaştırıcısı |
      | `buildReplayPolicy` | Özel transkript yeniden oynatma/Compaction ilkesi |
      | `sanitizeReplayHistory` | Genel temizlikten sonra sağlayıcıya özgü yeniden oynatma yeniden yazımları |
      | `validateReplayTurns` | Gömülü çalıştırıcıdan önce katı yeniden oynatma turu doğrulaması |
      | `onModelSelected` | Seçim sonrası geri çağırma (ör. telemetri) |

      Çalışma zamanı geri dönüş notları:

      - `normalizeConfig`, sağlayıcı kimliği başına bir sahip Plugin'i çözümler (önce paketlenmiş sağlayıcılar, ardından eşleşen çalışma zamanı Plugin'i) ve yalnızca bu kancayı çağırır; diğer sağlayıcılar arasında tarama yapılmaz. `google` / `google-vertex` / `google-antigravity` yapılandırma girdilerini normalleştiren, Google'ın kendi `normalizeConfig` kancasıdır; bu, ayrı bir çekirdek geri dönüşü değildir.
      - `resolveConfigApiKey`, kullanıma sunulduğunda sağlayıcı kancasını kullanır. Amazon Bedrock, AWS ortam işareti çözümlemesini kendi sağlayıcı Plugin'inde tutar; çalışma zamanı kimlik doğrulaması ise `auth: "aws-sdk"` ile yapılandırıldığında AWS SDK varsayılan zincirini kullanmaya devam eder.
      - `resolveThinkingProfile(ctx)`; seçilen `provider`, `modelId`, isteğe bağlı birleştirilmiş `reasoning` katalog ipucu ve isteğe bağlı birleştirilmiş model `compat` olgularını alır. `compat` öğesini yalnızca sağlayıcının düşünme kullanıcı arayüzünü/profilini seçmek için kullanın.
      - `resolveSystemPromptContribution`, bir sağlayıcının model ailesi için önbellek duyarlı sistem istemi yönlendirmesi eklemesini sağlar. Davranış tek bir sağlayıcı/model ailesine aitse ve kararlı/dinamik önbellek ayrımını koruması gerekiyorsa eski, Plugin genelindeki `before_prompt_build` kancası yerine bunu tercih edin.

    </Accordion>

  </Step>

  <Step title="Ek yetenekler ekleyin (isteğe bağlı)">
    ### 5. Adım: Ek yetenekler ekleyin

    Bir sağlayıcı Plugin'i, metin çıkarımının yanı sıra gömme, konuşma, gerçek zamanlı transkripsiyon,
    gerçek zamanlı ses, medya anlama, görüntü oluşturma, video oluşturma,
    web'den getirme ve web araması kaydedebilir. OpenClaw bunu,
    şirket Plugin'leri için önerilen kalıp olan (tedarikçi başına bir Plugin)
    **karma yetenekli** Plugin olarak sınıflandırır. Bkz.
    [Dahili Yapı: Yetenek Sahipliği](/tr/plugins/architecture#capability-ownership-model).

    Her yeteneği, mevcut `api.registerProvider(...)` çağrınızın yanında `register(api)`
    içinde kaydedin. Yalnızca ihtiyaç duyduğunuz sekmeleri seçin:

    <Tabs>
      <Tab title="Konuşma (TTS)">
        ```typescript
        import {
          assertOkOrThrowProviderError,
          postJsonRequest,
        } from "openclaw/plugin-sdk/provider-http";

        api.registerSpeechProvider({
          id: "acme-ai",
          label: "Acme Speech",
          defaultTimeoutMs: 120_000,
          isConfigured: ({ config }) => Boolean(config.messages?.tts),
          synthesize: async (req) => {
            const { response, release } = await postJsonRequest({
              url: "https://api.example.com/v1/speech",
              headers: new Headers({ "Content-Type": "application/json" }),
              body: { text: req.text },
              timeoutMs: req.timeoutMs,
              fetchFn: fetch,
              auditContext: "acme speech",
            });
            try {
              await assertOkOrThrowProviderError(response, "Acme Speech API error");
              return {
                audioBuffer: Buffer.from(await response.arrayBuffer()),
                outputFormat: "mp3",
                fileExtension: ".mp3",
                voiceCompatible: false,
              };
            } finally {
              await release();
            }
          },
        });
        ```

        Sağlayıcı HTTP hataları için `assertOkOrThrowProviderError(...)` kullanın; böylece
        Plugin'ler sınırlandırılmış hata gövdesi okumalarını, JSON hata ayrıştırmasını ve
        istek kimliği son eklerini paylaşır.
      </Tab>
      <Tab title="Gerçek zamanlı transkripsiyon">
        `createRealtimeTranscriptionWebSocketSession(...)` öğesini tercih edin; paylaşılan
        yardımcı, proxy yakalamayı, yeniden bağlanma geri çekilmesini, kapanışta boşaltmayı, hazır olma
        el sıkışmalarını, ses kuyruğa almayı ve kapanış olayı tanılamalarını yönetir. Plugin'iniz
        yalnızca yukarı akış olaylarını eşler.

        ```typescript
        api.registerRealtimeTranscriptionProvider({
          id: "acme-ai",
          label: "Acme Realtime Transcription",
          isConfigured: () => true,
          createSession: (req) => {
            const apiKey = String(req.providerConfig.apiKey ?? "");
            return createRealtimeTranscriptionWebSocketSession({
              providerId: "acme-ai",
              callbacks: req,
              url: "wss://api.example.com/v1/realtime-transcription",
              headers: { Authorization: `Bearer ${apiKey}` },
              onMessage: (event, transport) => {
                if (event.type === "session.created") {
                  transport.sendJson({ type: "session.update" });
                  transport.markReady();
                  return;
                }
                if (event.type === "transcript.final") {
                  req.onTranscript?.(event.text);
                }
              },
              sendAudio: (audio, transport) => {
                transport.sendJson({
                  type: "audio.append",
                  audio: audio.toString("base64"),
                });
              },
              onClose: (transport) => {
                transport.sendJson({ type: "audio.end" });
              },
            });
          },
        });
        ```

        Çok parçalı ses verisini POST eden toplu STT sağlayıcıları,
        `openclaw/plugin-sdk/provider-http` içindeki
        `buildAudioTranscriptionFormData(...)` öğesini kullanmalıdır. Yardımcı, uyumlu
        transkripsiyon API'leri için M4A tarzı bir dosya adına ihtiyaç duyan AAC yüklemeleri
        dâhil olmak üzere yükleme dosyası adlarını normalleştirir.
      </Tab>
      <Tab title="Gerçek zamanlı ses">
        ```typescript
        api.registerRealtimeVoiceProvider({
          id: "acme-ai",
          label: "Acme Realtime Voice",
          capabilities: {
            transports: ["gateway-relay"],
            inputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            outputAudioFormats: [{ encoding: "pcm16", sampleRateHz: 24000, channels: 1 }],
            supportsBargeIn: true,
            handlesInputAudioBargeIn: true,
            supportsToolCalls: true,
          },
          isConfigured: ({ providerConfig }) => Boolean(providerConfig.apiKey),
          createBridge: (req) => ({
            // Bunu yalnızca sağlayıcı tek bir çağrı için birden fazla araç yanıtını
            // kabul ediyorsa ayarlayın; örneğin, hemen verilen bir "çalışıyor" yanıtının
            // ardından nihai sonucun gelmesi.
            supportsToolResultContinuation: false,
            connect: async () => {},
            sendAudio: () => {},
            setMediaTimestamp: () => {},
            handleBargeIn: () => {},
            submitToolResult: () => {},
            acknowledgeMark: () => {},
            close: () => {},
            isConnected: () => true,
          }),
        });
        ```

        `talk.catalog` öğesinin tarayıcı ve yerel Talk
        istemcilerine geçerli modları, aktarımları, ses biçimlerini ve özellik bayraklarını sunabilmesi için
        `capabilities` öğesini bildirin. Bir aktarım, bir insanın
        asistan oynatmasını kestiğini algılayabiliyorsa ve sağlayıcı etkin ses yanıtını
        kısaltmayı veya temizlemeyi destekliyorsa `handleBargeIn` öğesini uygulayın.
        `submitToolResult`, eşzamanlı gönderim için `void` veya sağlayıcı
        köprüsünün sunabileceği eşzamansız bir tamamlanma sınırı için
        `Promise<void>` döndürebilir. Gateway aktarma oturumları, nihai sonucu
        onaylamadan veya bağlantılı çalıştırmayı temizlemeden önce bu sözü bekler; gönderim
        başarısız olduğunda sözü reddedin.
        Sağlayıcı `options.suppressResponse` davranışını
        yerine getiremiyorsa `supportsToolResultSuppression: false` öğesini ayarlayın. Böylece OpenClaw,
        dâhili zorunlu danışma ve iptal sonuçları için bastırmadan kaçınır ve sessizce
        bir yanıt başlatmak yerine doğrudan bastırılmış sonuç isteklerini reddeder.
        `createRealtimeVoiceBridgeSession` tüketicileri de benzer şekilde
        `onToolCall` içinden bir söz döndürebilir; eşzamanlı fırlatmalar ve retler
        oturumun `onError` geri çağrısına yönlendirilir.
        `handlesInputAudioBargeIn` öğesini yalnızca sağlayıcı VAD,
        `onClearAudio("barge-in")` öğesini çağırarak bir kesintiyi doğruladığında ayarlayın. Bayrağı
        atlayan sağlayıcılar, OpenClaw'ın yerel giriş sesi geri dönüş algılamasını kullanır.
      </Tab>
      <Tab title="Medya anlama">
        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "acme-ai",
          capabilities: ["image", "audio"],
          describeImage: async (req) => ({ text: "Şunun fotoğrafı..." }),
          transcribeAudio: async (req) => ({ text: "Transkript..." }),
        });
        ```

        Bilerek kimlik bilgileri gerektirmeyen yerel veya kendi kendine barındırılan medya
        sağlayıcıları, `resolveAuth` öğesini sunabilir ve `kind: "none"`
        döndürebilir. OpenClaw, açıkça katılmayı seçmeyen sağlayıcılar için normal
        kimlik doğrulama geçidini yine korur. Mevcut sağlayıcılar `req.apiKey` öğesini
        okumaya devam edebilir; yeni sağlayıcılar `req.auth` öğesini tercih etmelidir.

        ```typescript
        api.registerMediaUnderstandingProvider({
          id: "local-audio",
          capabilities: ["audio"],
          resolveAuth: () => ({
            kind: "none",
            source: "local-audio Plugin'i kimlik doğrulama kullanmıyor",
          }),
          transcribeAudio: async (req) => ({ text: "Transkript..." }),
        });
        ```
      </Tab>
      <Tab title="Gömmeler">
        ```typescript
        api.registerEmbeddingProvider({
          id: "acme-ai",
          defaultModel: "acme-embed",
          transport: "remote",
          authProviderId: "acme-ai",
          create: async ({ model }) => ({
            provider: {
              id: "acme-ai",
              model,
              dimensions: 1536,
              embed: async (input) => {
                const text = typeof input === "string" ? input : input.text;
                return fetchAcmeEmbedding(text);
              },
              embedBatch: async (inputs) =>
                Promise.all(
                  inputs.map((input) =>
                    fetchAcmeEmbedding(typeof input === "string" ? input : input.text),
                  ),
                ),
            },
          }),
        });
        ```

        Aynı kimliği `contracts.embeddingProviders` içinde bildirin. Bu,
        bellek araması dâhil yeniden kullanılabilir vektör oluşturma için genel
        gömme sözleşmesidir. `registerMemoryEmbeddingProvider(...)`, mevcut
        belleğe özgü bağdaştırıcılar için kullanımdan kaldırılmış uyumluluk desteğidir.
      </Tab>
      <Tab title="Görüntü ve video oluşturma">
        Görüntü ve video yetenekleri **mod duyarlı** bir şekil kullanır. Görüntü
        sağlayıcıları gerekli `generate` ve `edit` yetenek bloklarını;
        video sağlayıcıları ise `generate`, `imageToVideo` ve
        `videoToVideo` bloklarını bildirir. `maxInputImages` /
        `maxInputVideos` / `maxDurationSeconds` gibi düz toplu alanlar, dönüştürme modu
        desteğini veya devre dışı modları açıkça duyurmak için yeterli değildir. Müzik oluşturma
        aynı `generate` / `edit` kalıbını izler.

        ```typescript
        api.registerImageGenerationProvider({
          id: "acme-ai",
          label: "Acme Görselleri",
          capabilities: {
            generate: { maxCount: 4, supportsSize: true },
            edit: { enabled: false },
          },
          generateImage: async (req) => ({ images: [] }),
        });

        api.registerVideoGenerationProvider({
          id: "acme-ai",
          label: "Acme Video",
          defaultTimeoutMs: 600_000,
          models: ["acme-video", "acme-image-video"],
          capabilities: {
            generate: { maxVideos: 1, maxDurationSeconds: 10, supportsResolution: true },
            imageToVideo: {
              enabled: true,
              maxVideos: 1,
              maxInputImages: 1,
              maxInputImagesByModel: { "acme/reference-to-video": 9 },
              maxDurationSeconds: 5,
            },
            videoToVideo: { enabled: false },
          },
          catalogByModel: {
            "acme-image-video": {
              modes: ["imageToVideo"],
              capabilities: {
                imageToVideo: {
                  enabled: true,
                  maxVideos: 1,
                  maxInputImages: 1,
                  resolutions: ["480P", "720P", "1080P"],
                  supportsResolution: true,
                },
                videoToVideo: { enabled: false },
              },
            },
          },
          generateVideo: async (req) => ({ videos: [] }),
        });
        ```

        `capabilities` her iki sağlayıcı türünde de gereklidir; `edit` ve
        video dönüştürme blokları (`imageToVideo`, `videoToVideo`) her zaman açık bir
        `enabled` bayrağı gerektirir.

        Listelenen bir modelin statik modları veya yetenekleri sağlayıcının
        varsayılanlarından farklı olduğunda `catalogByModel` kullanın. Bu meta veriler,
        sağlayıcı kodunu çağırmadan `video_generate action=list` ve model kataloglarının
        doğru kalmasını sağlar. İstek anındaki yetenek arama ve uygulama işlemleri
        yine `resolveModelCapabilities` ve `generateVideo` içinde yer almalıdır; mümkün olduğunda
        her iki yol için de aynı yetenek sabitini yeniden kullanın.
      </Tab>
      <Tab title="Web getirme ve arama">
        ```typescript
        api.registerWebFetchProvider({
          id: "acme-ai-fetch",
          label: "Acme Fetch",
          hint: "Sayfaları Acme'nin işleme arka ucu üzerinden getirin.",
          envVars: ["ACME_FETCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/fetch",
          credentialPath: "plugins.entries.acme.config.webFetch.apiKey",
          getCredentialValue: (fetchConfig) => fetchConfig?.acme?.apiKey,
          setCredentialValue: (fetchConfigTarget, value) => {
            const acme = (fetchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "Bir sayfayı Acme Fetch üzerinden getirin.",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });

        api.registerWebSearchProvider({
          id: "acme-ai-search",
          label: "Acme Search",
          hint: "Web'de Acme'nin arama arka ucu üzerinden arama yapın.",
          envVars: ["ACME_SEARCH_API_KEY"],
          placeholder: "acme-...",
          signupUrl: "https://acme.example.com/search",
          credentialPath: "plugins.entries.acme.config.webSearch.apiKey",
          getCredentialValue: (searchConfig) => searchConfig?.acme?.apiKey,
          setCredentialValue: (searchConfigTarget, value) => {
            const acme = (searchConfigTarget.acme ??= {});
            acme.apiKey = value;
          },
          createTool: () => ({
            description: "Web'de Acme Search üzerinden arama yapın.",
            parameters: {},
            execute: async (args) => ({ content: [] }),
          }),
        });
        ```

        Her iki sağlayıcı türü de aynı kimlik bilgisi bağlantı yapısını paylaşır:
        `hint`, `envVars`, `placeholder`, `signupUrl`, `credentialPath`,
        `getCredentialValue`, `setCredentialValue` ve `createTool` öğelerinin tümü
        gereklidir.
      </Tab>
    </Tabs>

  </Step>

  <Step title="Test">
    ### Adım 6: Test

    ```typescript src/provider.test.ts
    import { describe, it, expect } from "vitest";
    // Sağlayıcı yapılandırma nesnenizi index.ts veya özel bir dosyadan dışa aktarın
    import { acmeProvider } from "./provider.js";

    describe("acme-ai sağlayıcısı", () => {
      it("dinamik modelleri çözümler", () => {
        const model = acmeProvider.resolveDynamicModel!({
          modelId: "acme-beta-v3",
        } as any);
        expect(model.id).toBe("acme-beta-v3");
        expect(model.provider).toBe("acme-ai");
      });

      it("anahtar kullanılabilir olduğunda kataloğu döndürür", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: "test-key" }),
        } as any);
        expect(result?.provider?.models).toHaveLength(2);
      });

      it("anahtar olmadığında null katalog döndürür", async () => {
        const result = await acmeProvider.catalog!.run({
          resolveProviderApiKey: () => ({ apiKey: undefined }),
        } as any);
        expect(result).toBeNull();
      });
    });
    ```

  </Step>
</Steps>

## ClawHub'da yayımlama

Sağlayıcı pluginleri, diğer tüm harici kod pluginleriyle aynı şekilde yayımlanır:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

`clawhub skill publish <path>`, bir plugin paketini değil bir skill
klasörünü yayımlamak için kullanılan farklı bir komuttur; burada kullanmayın.

## Dosya yapısı

```
<bundled-plugin-root>/acme-ai/
├── package.json              # openclaw.providers meta verileri
├── openclaw.plugin.json      # Sağlayıcı kimlik doğrulama meta verilerini içeren bildirim
├── index.ts                  # definePluginEntry + registerProvider
└── src/
    ├── provider.test.ts      # Testler
    └── usage.ts              # Kullanım uç noktası (isteğe bağlı)
```

## Katalog sırası referansı

`catalog.order`, kataloğunuzun yerleşik sağlayıcılara göre ne zaman
birleştirileceğini denetler:

| Sıra     | Zaman          | Kullanım alanı                                        |
| --------- | ------------- | ----------------------------------------------- |
| `simple`  | İlk geçiş    | Düz API anahtarı sağlayıcıları                         |
| `profile` | Basit aşamadan sonra  | Kimlik doğrulama profillerine bağlı sağlayıcılar                |
| `paired`  | Profil aşamasından sonra | Birbiriyle ilişkili birden fazla giriş oluşturma             |
| `late`    | Son geçiş     | Mevcut sağlayıcıları geçersiz kılma (çakışmada kazanır) |

## Sonraki adımlar

- [Kanal Pluginleri](/tr/plugins/sdk-channel-plugins) - plugininiz aynı zamanda bir kanal sağlıyorsa
- [SDK Çalışma Zamanı](/tr/plugins/sdk-runtime) - `api.runtime` yardımcıları (TTS, arama, alt ajan)
- [SDK Genel Bakışı](/tr/plugins/sdk-overview) - tam alt yol içe aktarma referansı
- [Plugin İç Yapısı](/tr/plugins/architecture-internals#provider-runtime-hooks) - kanca ayrıntıları ve paketlenmiş örnekler

## İlgili

- [Plugin SDK kurulumu](/tr/plugins/sdk-setup)
- [Plugin oluşturma](/tr/plugins/building-plugins)
- [Kanal pluginleri oluşturma](/tr/plugins/sdk-channel-plugins)
