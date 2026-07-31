---
read_when:
    - Bir plugin için testler yazıyorsunuz
    - Plugin SDK'sından test yardımcı programlarına ihtiyacınız var
    - Paketle birlikte gelen pluginler için sözleşme testlerini anlamak istiyorsunuz
sidebarTitle: Testing
summary: OpenClaw Pluginleri için test yardımcıları ve kalıpları
title: Plugin testi
x-i18n:
    generated_at: "2026-07-27T00:13:16Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9c6c050826dae3cd2c794d50b2dd95e20e6533d838161cce037742ee5fdf7e0e
    source_path: plugins/sdk-testing.md
    workflow: 16
---

OpenClaw Pluginleri için test yardımcı programları, kalıpları ve lint uygulamasına ilişkin referans.

<Tip>
  **Test örnekleri mi arıyorsunuz?** Nasıl yapılır kılavuzları, ayrıntılı test örnekleri içerir:
  [Kanal Plugin testleri](/tr/plugins/sdk-channel-plugins#step-6-test) ve
  [Sağlayıcı Plugin testleri](/tr/plugins/sdk-provider-plugins#step-6-test).
</Tip>

## Test yardımcı programları

Bu alt yollar, OpenClaw'ın kendi paketle birlikte sunulan Plugin testleri için
depoya yerel kaynak giriş noktalarıdır. Üçüncü taraf Pluginler için yayımlanmış
`package.json` dışa aktarımları değildir ve Vitest'i veya yalnızca depoda
bulunan diğer test bağımlılıklarını içe aktarabilirler.

```typescript
import {
  shouldAckReaction,
  removeAckReactionAfterReply,
} from "openclaw/plugin-sdk/channel-feedback";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/channel-target-testing";
import { AUTH_PROFILE_RUNTIME_CONTRACT } from "openclaw/plugin-sdk/agent-runtime-test-contracts";
import { createTestPluginApi } from "openclaw/plugin-sdk/plugin-test-api";
import { expectChannelInboundContextContract } from "openclaw/plugin-sdk/channel-contract-testing";
import { createStartAccountContext } from "openclaw/plugin-sdk/channel-test-helpers";
import { describePluginRegistrationContract } from "openclaw/plugin-sdk/plugin-test-contracts";
import { registerSingleProviderPlugin } from "openclaw/plugin-sdk/plugin-test-runtime";
import { describeOpenAIProviderRuntimeContract } from "openclaw/plugin-sdk/provider-test-contracts";
import { getProviderHttpMocks } from "openclaw/plugin-sdk/provider-http-test-mocks";
import { withEnv, withFetchPreconnect, withServer } from "openclaw/plugin-sdk/test-env";
import { isLiveTestEnabled } from "openclaw/plugin-sdk/test-live";
import { createRequestCaptureJsonFetch } from "openclaw/plugin-sdk/test-media-understanding";
import {
  bundledPluginRoot,
  createCliRuntimeCapture,
  typedCases,
} from "openclaw/plugin-sdk/test-fixtures";
import { mockNodeBuiltinModule } from "openclaw/plugin-sdk/test-node-mocks";
```

Paketle birlikte sunulan Plugin testleri için bu odaklanmış alt yolları kullanın. Önceki
`openclaw/plugin-sdk/testing` barrel'ı depoya yereldi, yayımlanan
paketlerin dışında tutuluyordu ve kaldırıldı. Önceki `openclaw/plugin-sdk/test-utils`
takma adı da onunla birlikte kaldırıldı. `pnpm run lint:plugins:no-extension-test-core-imports`
(`scripts/check-no-extension-test-core-imports.ts`) uzantı testlerinin yukarıdaki
odaklanmış test alt yollarını kullanmasını sağlar.

### Kullanılabilir dışa aktarımlar

| Dışa Aktarım                                               | Amaç                                                                                                                                  |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `createTestPluginApi`                                | Doğrudan kayıt birim testleri için asgari bir plugin API taklidi oluşturur. `plugin-sdk/plugin-test-api` kaynağından içe aktarın                             |
| `AUTH_PROFILE_RUNTIME_CONTRACT`                      | Yerel ajan çalışma zamanı bağdaştırıcıları için paylaşılan kimlik doğrulama profili sözleşmesi fikstürü. `plugin-sdk/agent-runtime-test-contracts` kaynağından içe aktarın            |
| `DELIVERY_NO_REPLY_RUNTIME_CONTRACT`                 | Yerel ajan çalışma zamanı bağdaştırıcıları için paylaşılan teslimat engelleme sözleşmesi fikstürü. `plugin-sdk/agent-runtime-test-contracts` kaynağından içe aktarın    |
| `OUTCOME_FALLBACK_RUNTIME_CONTRACT`                  | Yerel ajan çalışma zamanı bağdaştırıcıları için paylaşılan geri dönüş sınıflandırma sözleşmesi fikstürü. `plugin-sdk/agent-runtime-test-contracts` kaynağından içe aktarın |
| `createParameterFreeTool`                            | Yerel çalışma zamanı sözleşme testleri için dinamik araç şeması fikstürleri oluşturur. `plugin-sdk/agent-runtime-test-contracts` kaynağından içe aktarın              |
| `expectChannelInboundContextContract`                | Kanalın gelen bağlam biçimini doğrular. `plugin-sdk/channel-contract-testing` kaynağından içe aktarın                                                  |
| `installChannelOutboundPayloadContractSuite`         | Kanalın giden yük sözleşmesi durumlarını kurar. `plugin-sdk/channel-contract-testing` kaynağından içe aktarın                                       |
| `createStartAccountContext`                          | Kanal hesabı yaşam döngüsü bağlamları oluşturur. `plugin-sdk/channel-test-helpers` kaynağından içe aktarın                                                  |
| `installChannelActionsContractSuite`                 | Genel kanal mesaj eylemi sözleşmesi durumlarını kurar. `plugin-sdk/channel-test-helpers` kaynağından içe aktarın                                     |
| `installChannelSetupContractSuite`                   | Genel kanal kurulum sözleşmesi durumlarını kurar. `plugin-sdk/channel-test-helpers` kaynağından içe aktarın                                              |
| `installChannelStatusContractSuite`                  | Genel kanal durum sözleşmesi durumlarını kurar. `plugin-sdk/channel-test-helpers` kaynağından içe aktarın                                             |
| `expectDirectoryIds`                                 | Bir dizin listeleme işlevinden gelen kanal dizini kimliklerini doğrular. `plugin-sdk/channel-test-helpers` kaynağından içe aktarın                               |
| `assertBundledChannelEntries`                        | Paketlenmiş kanal giriş noktalarının beklenen genel sözleşmeyi sunduğunu doğrular. `plugin-sdk/channel-test-helpers` kaynağından içe aktarın                    |
| `formatEnvelopeTimestamp`                            | Belirlenimci zarf zaman damgalarını biçimlendirir. `plugin-sdk/channel-test-helpers` kaynağından içe aktarın                                                  |
| `expectPairingReplyText`                             | Kanal eşleştirme yanıt metnini doğrular ve kodunu çıkarır. `plugin-sdk/channel-test-helpers` kaynağından içe aktarın                                    |
| `describePluginRegistrationContract`                 | Plugin kayıt sözleşmesi kontrollerini kurar. `plugin-sdk/plugin-test-contracts` kaynağından içe aktarın                                              |
| `registerSingleProviderPlugin`                       | Yükleyici duman testlerinde bir sağlayıcı plugin'i kaydeder. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                         |
| `registerProviderPlugin`                             | Tek bir plugin'deki tüm sağlayıcı türlerini yakalar. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                                 |
| `registerProviderPlugins`                            | Birden çok plugin'deki sağlayıcı kayıtlarını yakalar. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                     |
| `requireRegisteredProvider`                          | Bir sağlayıcı koleksiyonunun bir kimlik içerdiğini doğrular. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                           |
| `createRuntimeEnv`                                   | Taklit edilmiş bir CLI/plugin çalışma zamanı ortamı oluşturur. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                              |
| `createPluginRuntimeMock`                            | Taklit edilmiş bir plugin çalışma zamanı yüzeyi oluşturur. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                                      |
| `createPluginSetupWizardStatus`                      | Kanal plugin'leri için kurulum durumu yardımcıları oluşturur. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                             |
| `createTestWizardPrompter`                           | Taklit edilmiş bir kurulum sihirbazı istemcisi oluşturur. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                                       |
| `createRuntimeTaskFlow`                              | Yalıtılmış çalışma zamanı görev akışı durumu oluşturur. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                                    |
| `runProviderCatalog`                                 | Test bağımlılıklarıyla bir sağlayıcı kataloğu kancasını yürütür. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                     |
| `resolveProviderWizardOptions`                       | Sözleşme testlerinde sağlayıcı kurulum sihirbazı seçimlerini çözümler. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                    |
| `resolveProviderModelPickerEntries`                  | Sözleşme testlerinde sağlayıcı model seçici girdilerini çözümler. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                    |
| `buildProviderPluginMethodChoice`                    | Doğrulamalar için sağlayıcı sihirbazı seçim kimlikleri oluşturur. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                            |
| `setProviderWizardProvidersResolverForTest`          | Yalıtılmış testler için sağlayıcı sihirbazı sağlayıcılarını enjekte eder. `plugin-sdk/plugin-test-runtime` kaynağından içe aktarın                                        |
| `describeOpenAIProviderRuntimeContract`              | Sağlayıcı ailesi çalışma zamanı sözleşmesi kontrollerini kurar. `plugin-sdk/provider-test-contracts` kaynağından içe aktarın                                        |
| `expectPassthroughReplayPolicy`                      | Sağlayıcı yeniden oynatma ilkelerinin sağlayıcının sahip olduğu araçlardan ve meta verilerden geçtiğini doğrular. `plugin-sdk/provider-test-contracts` kaynağından içe aktarın         |
| `runRealtimeSttLiveTest`                             | Paylaşılan ses fikstürleriyle canlı, gerçek zamanlı bir STT sağlayıcı testi çalıştırır. `plugin-sdk/provider-test-contracts` kaynağından içe aktarın                       |
| `normalizeTranscriptForMatch`                        | Bulanık doğrulamalardan önce canlı döküm çıktısını normalleştirir. `plugin-sdk/provider-test-contracts` kaynağından içe aktarın                               |
| `expectExplicitVideoGenerationCapabilities`          | Video sağlayıcılarının açık üretim modu yetenekleri bildirdiğini doğrular. `plugin-sdk/provider-test-contracts` kaynağından içe aktarın                   |
| `expectExplicitMusicGenerationCapabilities`          | Müzik sağlayıcılarının açık üretim/düzenleme yetenekleri bildirdiğini doğrular. `plugin-sdk/provider-test-contracts` kaynağından içe aktarın                   |
| `mockSuccessfulDashscopeVideoTask`                   | Başarılı bir DashScope uyumlu video görevi yanıtı kurar. `plugin-sdk/provider-test-contracts` kaynağından içe aktarın                          |
| `getProviderHttpMocks`                               | İsteğe bağlı sağlayıcı HTTP/kimlik doğrulama Vitest taklitlerine erişir. `plugin-sdk/provider-http-test-mocks` kaynağından içe aktarın                                         |
| `installProviderHttpMockCleanup`                     | Her testten sonra sağlayıcı HTTP/kimlik doğrulama taklitlerini sıfırlar. `plugin-sdk/provider-http-test-mocks` kaynağından içe aktarın                                        |
| `installCommonResolveTargetErrorCases`               | Hedef çözümleme hata işleme için paylaşılan test durumları. `plugin-sdk/channel-target-testing` kaynağından içe aktarın                                  |
| `shouldAckReaction`                                  | Bir kanalın onay tepkisi ekleyip eklememesi gerektiğini denetler. `plugin-sdk/channel-feedback` kaynağından içe aktarın                                            |
| `removeAckReactionAfterReply`                        | Yanıt teslim edildikten sonra onay tepkisini kaldırır. `plugin-sdk/channel-feedback` kaynağından içe aktarın                                                      |
| `createTestRegistry`                                 | Bir kanal plugin kayıt defteri fikstürü oluşturur. `plugin-sdk/plugin-test-runtime` veya `plugin-sdk/channel-test-helpers` kaynağından içe aktarın               |
| `createEmptyPluginRegistry`                          | Boş bir plugin kayıt defteri fikstürü oluşturur. `plugin-sdk/plugin-test-runtime` veya `plugin-sdk/channel-test-helpers` kaynağından içe aktarın                |
| `setActivePluginRegistry`                            | Plugin çalışma zamanı testleri için bir kayıt defteri fikstürü kurar. `plugin-sdk/plugin-test-runtime` veya `plugin-sdk/channel-test-helpers` kaynağından içe aktarın   |
| `createRequestCaptureJsonFetch`                      | Medya yardımcısı testlerinde JSON getirme isteklerini yakalar. `plugin-sdk/test-media-understanding` kaynağından içe aktarın                                     |
| `isLiveTestEnabled`                                  | İsteğe bağlı canlı sağlayıcı testlerini denetler. `plugin-sdk/test-live` kaynağından içe aktarın                                                                      |
| `collectProviderApiKeys`                             | Canlı sağlayıcı testleri için kimlik bilgilerini keşfeder. `plugin-sdk/test-live-auth` kaynağından içe aktarın                                                    |
| `parseProviderModelMap`                              | Müzik/video canlı test modeli geçersiz kılmalarını ayrıştırır. `plugin-sdk/test-media-generation` kaynağından içe aktarın                                              |
| `withServer`                                         | Tek kullanımlık bir yerel HTTP sunucusuna karşı testler çalıştırır. `plugin-sdk/test-env` kaynağından içe aktarın                                                      |
| `createMockIncomingRequest`                          | Asgari bir gelen HTTP isteği nesnesi oluşturur. `plugin-sdk/test-env` kaynağından içe aktarın                                                          |
| `withFetchPreconnect`                                | Ön bağlantı kancaları kurulu olarak getirme testlerini çalıştırır. `plugin-sdk/test-env` kaynağından içe aktarın                                                       |
| `withEnv` / `withEnvAsync`                           | Ortam değişkenlerini geçici olarak yamalar. `plugin-sdk/test-env` kaynağından içe aktarın                                                               |
| `createTempHomeEnv` / `withTempHome` / `withTempDir` | Yalıtılmış dosya sistemi test fikstürleri oluşturur. `plugin-sdk/test-env` kaynağından içe aktarın                                                              |
| `createMockServerResponse`                           | Asgari bir HTTP sunucusu yanıt taklidi oluşturur. `plugin-sdk/test-env` kaynağından içe aktarın                                                            |
| `createProviderUsageFetch`                           | Sağlayıcı kullanımını getirme fikstürleri oluşturur. `plugin-sdk/test-env` kaynağından içe aktarın                                                                   |
| `useFrozenTime` / `useRealTime`                      | Zamana duyarlı testler için zamanlayıcıları dondurur ve geri yükler. `plugin-sdk/test-env` kaynağından içe aktarın                                                    |
| `createCliRuntimeCapture`                            | Testlerde CLI çalışma zamanı çıktısını yakalar. `plugin-sdk/test-fixtures` kaynağından içe aktarın                                                              |
| `importFreshModule`                                  | Modül önbelleğini atlamak için yeni bir sorgu belirteciyle bir ESM modülünü içe aktarır. `plugin-sdk/test-fixtures` kaynağından içe aktarın                             |
| `bundledPluginRoot` / `bundledPluginFile`            | Paketlenmiş plugin kaynak veya dağıtım fikstürü yollarını çözümler. `plugin-sdk/test-fixtures` kaynağından içe aktarın                                              |
| `mockNodeBuiltinModule`                              | Dar kapsamlı yerleşik Node Vitest taklitlerini kurar. `plugin-sdk/test-node-mocks` kaynağından içe aktarın                                                       |
| `createSandboxTestContext`                           | Korumalı alan test bağlamları oluşturur. `plugin-sdk/test-fixtures` kaynağından içe aktarın                                                                      |
| `writeSkill`                                         | Beceri fikstürleri yazar. `plugin-sdk/test-fixtures` kaynağından içe aktarın                                                                             |
| `makeAgentAssistantMessage`                          | Ajan dökümü mesaj fikstürleri oluşturur. `plugin-sdk/test-fixtures` kaynağından içe aktarın                                                          |
| `peekSystemEvents` / `resetSystemEventsForTest`      | Sistem olayı fikstürlerini inceler ve sıfırlar. `plugin-sdk/test-fixtures` kaynağından içe aktarın                                                          |
| `sanitizeTerminalText`                               | Doğrulamalar için terminal çıktısını temizler. `plugin-sdk/test-fixtures` kaynağından içe aktarın                                                          |
| `countLines` / `hasBalancedFences`                   | Parçalama çıktısının biçimini doğrulayın. `plugin-sdk/test-fixtures` üzerinden içe aktarın                                                |
| `typedCases`                                         | Tablo güdümlü testler için değişmez türleri koruyun. `plugin-sdk/test-fixtures` üzerinden içe aktarın                                     |

Paketle birlikte sunulan Plugin sözleşme paketleri ayrıca yalnızca test amaçlı kayıt defteri, manifest, herkese açık yapıt ve çalışma zamanı fikstürü yardımcıları için bu SDK test alt yollarını kullanır.
Paketle birlikte sunulan OpenClaw envanterine bağımlı olan yalnızca çekirdeğe yönelik paketler ise bunun yerine
`src/plugins/contracts` altında kalır.

### Türler

Odaklı test alt yolları, test dosyalarında yararlı olan türleri de yeniden dışa aktarır:

```typescript
import type {
  ChannelAccountSnapshot,
  ChannelGatewayContext,
} from "openclaw/plugin-sdk/channel-contract";
import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
import type { MockFn, PluginRuntime, RuntimeEnv } from "openclaw/plugin-sdk/plugin-test-runtime";
```

## Test hedefi çözümleme

Kanal hedefi çözümlemesine yönelik standart hata durumlarını eklemek için
`installCommonResolveTargetErrorCases` kullanın:

```typescript
import { describe } from "vitest";
import { installCommonResolveTargetErrorCases } from "openclaw/plugin-sdk/channel-target-testing";

describe("my-channel hedef çözümlemesi", () => {
  installCommonResolveTargetErrorCases({
    resolveTarget: ({ to, mode, allowFrom }) => {
      // Kanalınızın hedef çözümleme mantığı
      return myChannelResolveTarget({ to, mode, allowFrom });
    },
    implicitAllowFrom: ["user1", "user2"],
  });

  // Kanala özgü test durumları ekleyin
  it("@username hedeflerini çözümlemelidir", () => {
    // ...
  });
});
```

## Test kalıpları

### Kayıt sözleşmelerini test etme

El ile yazılmış bir `api` taklidini `register(api)` öğesine ileten birim testleri,
OpenClaw'ın yükleyici kabul denetimlerini çalıştırmaz. Plugin'inizin bağımlı olduğu
her kayıt yüzeyi için, özellikle kancalar ve bellek gibi özel yetenekler için
yükleyici destekli en az bir duman testi ekleyin.

Gerekli meta veriler eksik olduğunda veya bir Plugin sahip olmadığı bir yetenek
API'sini çağırdığında gerçek yükleyici Plugin kaydını başarısız kılar. Örneğin,
`api.registerHook(...)` bir kanca adı gerektirir ve
`api.registerMemoryCapability(...)`, Plugin manifestinin veya dışa aktarılan
girdinin `kind: "memory"` bildirmesini gerektirir.

### Çalışma zamanı yapılandırmasına erişimi test etme

`openclaw/plugin-sdk/plugin-test-runtime` içindeki paylaşılan Plugin çalışma zamanı taklidini
tercih edin. Çalışma zamanı yapılandırma yardımcıları, geçerli anlık görüntü
ve değişiklik API'lerini modeller.

### Bir kanal Plugin'ini birim testiyle sınama

```typescript
import { describe, it, expect, vi } from "vitest";

describe("my-channel Plugin'i", () => {
  it("hesabı yapılandırmadan çözümlemelidir", () => {
    const cfg = {
      channels: {
        "my-channel": {
          token: "test-token",
          allowFrom: ["user1"],
        },
      },
    };

    const account = myPlugin.setup.resolveAccount(cfg, undefined);
    expect(account.token).toBe("test-token");
  });

  it("gizli değerleri somutlaştırmadan hesabı incelemelidir", () => {
    const cfg = {
      channels: {
        "my-channel": { token: "test-token" },
      },
    };

    const inspection = myPlugin.setup.inspectAccount(cfg, undefined);
    expect(inspection.configured).toBe(true);
    expect(inspection.tokenStatus).toBe("available");
    // Token değeri açığa çıkarılmaz
    expect(inspection).not.toHaveProperty("token");
  });
});
```

### Bir sağlayıcı Plugin'ini birim testiyle sınama

```typescript
import { describe, it, expect } from "vitest";

describe("my-provider Plugin'i", () => {
  it("dinamik modelleri çözümlemelidir", () => {
    const model = myProvider.resolveDynamicModel({
      modelId: "custom-model-v2",
      // ... bağlam
    });

    expect(model.id).toBe("custom-model-v2");
    expect(model.provider).toBe("my-provider");
    expect(model.api).toBe("openai-completions");
  });

  it("API anahtarı kullanılabilir olduğunda kataloğu döndürmelidir", async () => {
    const result = await myProvider.catalog.run({
      resolveProviderApiKey: () => ({ apiKey: "test-key" }),
      // ... bağlam
    });

    expect(result?.provider?.models).toHaveLength(2);
  });
});
```

### Plugin çalışma zamanını taklit etme

`createPluginRuntimeStore` kullanan kod için testlerde çalışma zamanını taklit edin:

```typescript
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
import type { PluginRuntime } from "openclaw/plugin-sdk/runtime-store";

const store = createPluginRuntimeStore<PluginRuntime>({
  pluginId: "test-plugin",
  errorMessage: "test çalışma zamanı ayarlanmadı",
});

// Test kurulumunda
const mockRuntime = {
  agent: {
    resolveAgentDir: vi.fn().mockReturnValue("/tmp/agent"),
    // ... diğer taklitler
  },
  config: {
    current: vi.fn(() => ({}) as const),
    mutateConfigFile: vi.fn(),
    replaceConfigFile: vi.fn(),
  },
  // ... diğer ad alanları
} as unknown as PluginRuntime;

store.setRuntime(mockRuntime);

// Testlerden sonra
store.clearRuntime();
```

### Örnek başına saplamalarla test etme

Prototip değişikliği yerine örnek başına saplamaları tercih edin:

```typescript
// Tercih edilen: örnek başına saplama
const client = new MyChannelClient();
client.sendMessage = vi.fn().mockResolvedValue({ id: "msg-1" });

// Kaçının: prototip değişikliği
// MyChannelClient.prototype.sendMessage = vi.fn();
```

## Sözleşme testleri (depo içi Plugin'ler)

Paketle birlikte sunulan Plugin'lerin kayıt sahipliğini doğrulayan sözleşme testleri vardır:

```bash
pnpm test src/plugins/contracts/
```

Bu testler şunları doğrular:

- Hangi Plugin'lerin hangi sağlayıcıları kaydettiği
- Hangi Plugin'lerin hangi konuşma sağlayıcılarını kaydettiği
- Kayıt biçiminin doğruluğu
- Çalışma zamanı sözleşmesine uygunluk

### Kapsamlı testleri çalıştırma

Belirli bir Plugin için:

```bash
pnpm test <bundled-plugin-root>/my-channel/
```

Yalnızca sözleşme testleri için:

```bash
pnpm test src/plugins/contracts/shape.contract.test.ts
pnpm test src/plugins/contracts/auth-choice.contract.test.ts
pnpm test src/plugins/contracts/runtime-seams.contract.test.ts
```

## Lint uygulaması (depo içi Plugin'ler)

`scripts/run-additional-boundary-checks.mjs`, CI'da bir dizi `lint:plugins:*`
içe aktarma sınırı denetimi çalıştırır; bunların her biri yerel olarak bağımsız biçimde de çalıştırılabilir:

| Komut                                                        | Uyguladığı kural                                                                                     |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| `pnpm run lint:plugins:no-monolithic-plugin-sdk-entry-imports` | Paketle birlikte sunulan Plugin'ler, yekpare `openclaw/plugin-sdk` kök barrel'ını içe aktaramaz.              |
| `pnpm run lint:plugins:no-extension-src-imports`               | Üretim uzantısı dosyaları, deponun `src/**` ağacını doğrudan içe aktaramaz (`../../src/...`).  |
| `pnpm run lint:plugins:no-extension-test-core-imports`         | Uzantı test dosyaları, kaldırılmış SDK test takma adlarını veya yalnızca çekirdeğe yönelik diğer test yardımcılarını içe aktaramaz. |

Harici Plugin'ler bu lint kurallarına tabi değildir, ancak aynı kalıpların
izlenmesi önerilir.

## Test yapılandırması

OpenClaw, bilgilendirici V8 kapsam raporlamasıyla Vitest 4 kullanır. Plugin testleri için:

```bash
# Tüm testleri çalıştır
pnpm test

# Belirli Plugin testlerini çalıştır
pnpm test <bundled-plugin-root>/my-channel/src/channel.test.ts

# Belirli bir test adı filtresiyle çalıştır
pnpm test <bundled-plugin-root>/my-channel/ -t "resolves account"

# Kapsamla çalıştır
pnpm test:coverage
```

Yerel çalıştırmalar bellek baskısına neden olursa:

```bash
OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test
```

## İlgili

- [SDK'ya Genel Bakış](/tr/plugins/sdk-overview) -- içe aktarma kuralları
- [SDK Kanal Plugin'leri](/tr/plugins/sdk-channel-plugins) -- kanal Plugin'i arayüzü
- [SDK Sağlayıcı Plugin'leri](/tr/plugins/sdk-provider-plugins) -- sağlayıcı Plugin kancaları
- [Plugin Oluşturma](/tr/plugins/building-plugins) -- başlangıç kılavuzu
