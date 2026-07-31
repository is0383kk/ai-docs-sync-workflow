---
read_when:
    - defineToolPlugin, definePluginEntry veya defineChannelPluginEntry'nin tam tür imzasına ihtiyacınız var
    - Kayıt modunu (tam, kurulum veya CLI meta verileri) anlamak istiyorsunuz
    - Giriş noktası seçeneklerini arıyorsunuz
sidebarTitle: Entry Points
summary: defineToolPlugin, definePluginEntry, defineChannelPluginEntry ve defineSetupPluginEntry için başvuru kaynağı
title: Plugin giriş noktaları
x-i18n:
    generated_at: "2026-07-27T00:13:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e64fe1d65531fea8f266aa23b73064daf2ed2c5c43af8bb08ea57e347fe566f4
    source_path: plugins/sdk-entrypoints.md
    workflow: 16
---

Her plugin varsayılan bir giriş nesnesi dışa aktarır. SDK, her
giriş biçimi için bir yardımcı sağlar: `defineToolPlugin`, `definePluginEntry`,
`defineChannelPluginEntry`, `defineSetupPluginEntry`.

<Tip>
  **Adım adım açıklama mı arıyorsunuz?** Adım adım kılavuzlar için [Araç Pluginleri](/tr/plugins/tool-plugins),
  [Kanal Pluginleri](/tr/plugins/sdk-channel-plugins) veya
  [Sağlayıcı Pluginleri](/tr/plugins/sdk-provider-plugins) bölümlerine bakın.
</Tip>

## Paket girişleri

Yüklü pluginler, `package.json` `openclaw` alanlarını hem kaynak hem de
derlenmiş girişlere yönlendirir:

```json
{
  "openclaw": {
    "extensions": ["./src/index.ts"],
    "runtimeExtensions": ["./dist/index.js"],
    "setupEntry": "./src/setup-entry.ts",
    "runtimeSetupEntry": "./dist/setup-entry.js"
  }
}
```

- `extensions` ve `setupEntry`, çalışma alanı ve git
  çalışma kopyası geliştirmesinde kullanılan kaynak girişleridir.
- `runtimeExtensions` ve `runtimeSetupEntry`, yüklü
  paketler için tercih edilir: npm paketlerinin çalışma zamanında TypeScript derlemesini atlamasını sağlar.
- `runtimeExtensions` mevcut olduğunda dizi uzunluğu bakımından `extensions` ile eşleşmelidir
  (girişler konumlarına göre eşleştirilir). `runtimeSetupEntry`, `setupEntry` gerektirir.
- Bir `runtimeExtensions`/`runtimeSetupEntry` yapıtı bildirilmiş ancak
  eksikse kurulum/keşif bir paketleme hatasıyla başarısız olur; OpenClaw sessizce
  kaynağa geri dönmez. Kaynağa geri dönüş (aşağıda) yalnızca hiçbir çalışma zamanı
  girişi bildirilmediğinde uygulanır.
- Yüklü bir paket yalnızca bir TypeScript kaynak girişi bildirirse OpenClaw,
  eşleşen bir derlenmiş `dist/*.js` (veya `.mjs`/`.cjs`) eşini arar ve kullanır;
  aksi takdirde TypeScript kaynağına geri döner.
- Tüm giriş yolları plugin paket dizininin içinde kalmalıdır. Çalışma zamanı
  girişleri ve çıkarımlanan derlenmiş JS eşleri, paket dışına çıkan bir `extensions` veya
  `setupEntry` kaynak yolunu geçerli kılmaz.

## `defineToolPlugin`

**İçe aktarma:** `openclaw/plugin-sdk/tool-plugin`

Yalnızca ajan araçları ekleyen pluginler içindir. Kaynağı küçük tutar, yapılandırma
ve araç parametresi türlerini TypeBox şemalarından çıkarır, düz dönüş değerlerini
OpenClaw araç sonucu biçiminde sarmalar ve `openclaw plugins build` öğesinin plugin
manifestine (`contracts.tools`, `configSchema`) yazdığı statik meta verileri sunar.

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "Fetch stock quotes.",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "API key." })),
  }),
  tools: (tool) => [
    tool({
      name: "quote",
      label: "Quote",
      description: "Fetch a quote.",
      parameters: Type.Object({
        symbol: Type.String({ description: "Ticker symbol." }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          hasKey: Type.Boolean(),
        },
        { additionalProperties: false },
      ),
      execute: async ({ symbol }, config) => ({ symbol, hasKey: Boolean(config.apiKey) }),
    }),
  ],
});
```

- `configSchema` isteğe bağlıdır; belirtilmemesi katı bir boş nesne şeması
  kullanır (oluşturulan manifest yine de `configSchema` içerir).
- `execute` düz bir dize veya JSON olarak serileştirilebilir bir değer döndürür; yardımcı,
  bunu `details` özgün (dizeleştirilmemiş) dönüş değerine ayarlanmış
  bir metin aracı sonucu olarak sarmalar.
- `outputSchema`, Code Mode ve Tool Search için bu özgün `details` değerini isteğe bağlı olarak açıklar.
  Katalog çağrıları, yürütmeden önce geçersiz bir şemayı reddeder ve
  son değeri döndürmeden önce doğrular.
- Özel araç sonuçları için `openclaw/plugin-sdk/tool-results`,
  `textResult` ve `jsonResult` öğelerini dışa aktarır.
- Araç adları statiktir; bu nedenle `openclaw plugins build`,
  elle yinelenmiş adlar olmadan bildirilen araçlardan `contracts.tools` öğesini türetir.
- Çalışma zamanı yüklemesi katı kalır: yüklü pluginler yine de
  `openclaw.plugin.json` ve `package.json` `openclaw.extensions` gerektirir. OpenClaw,
  eksik manifest verilerini çıkarmak için plugin kodunu hiçbir zaman yürütmez.

## `definePluginEntry`

**İçe aktarma:** `openclaw/plugin-sdk/plugin-entry`

Sağlayıcı pluginleri, gelişmiş araç pluginleri, kanca pluginleri ve mesajlaşma
kanalı **olmayan** her şey içindir.

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Short summary",
  register(api) {
    api.registerProvider({/* ... */});
    api.registerTool({/* ... */});
  },
});
```

| Alan                      | Tür                                                              | Zorunlu  | Varsayılan          |
| ------------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                      | `string`                                                         | Evet     | -                   |
| `name`                    | `string`                                                         | Evet     | -                   |
| `description`             | `string`                                                         | Evet     | -                   |
| `kind`                    | `string` (kullanımdan kaldırıldı, aşağıya bakın)                  | Hayır    | -                   |
| `configSchema`            | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | Hayır    | Boş nesne şeması    |
| `reload`                  | `OpenClawPluginReloadRegistration`                               | Hayır    | -                   |
| `nodeHostCommands`        | `OpenClawPluginNodeHostCommand[]`                                | Hayır    | -                   |
| `securityAuditCollectors` | `OpenClawPluginSecurityAuditCollector[]`                         | Hayır    | -                   |
| `register`                | `(api: OpenClawPluginApi) => void`                               | Evet     | -                   |

- `id`, `openclaw.plugin.json` manifestinizle eşleşmelidir.
- Harici oturum katalogları,
  `openclaw/plugin-sdk/session-catalog` ve
  `api.registerSessionCatalog({ id, label, list, read, continueSession?, archive? })` kullanır.
  Çekirdek, `sessions.catalog.*` Gateway yöntemlerinin sahibidir; sağlayıcılar RPC kaydetmeden
  ana makine, oturum ve normalleştirilmiş transkript izdüşümlerini döndürür. Bir
  liste sağlayıcısı, her ana makinenin işlemi sonuçlandıkça isteğe bağlı `onHost(host)` geri çağrısını
  çağırmalıdır; döndürülen ana makine dizisi, son uyumluluk
  anlık görüntüsü olarak zorunlu kalır.
- `kind` kullanımdan kaldırılmıştır: bunun yerine `openclaw.plugin.json`
  manifestinin `kind` alanında özel bir yuva (`"memory"` veya
  `"context-engine"`) bildirin. Çalışma zamanı girişi `kind`, yalnızca
  eski pluginler için uyumluluk geri dönüşü olarak kalır.
- `configSchema`, geç değerlendirme için bir işlev olabilir. OpenClaw şemayı
  ilk erişimde çözümler ve belleğe alır; böylece maliyetli şema oluşturucular yalnızca
  bir kez çalışır.
- Bir `nodeHostCommands` tanımlayıcısı `isAvailable({ config, env })` tanımlayabilir.
  `false` döndürülmesi, bu komutu ve yeteneğini başsız
  Node'un Gateway bildiriminden çıkarır. OpenClaw bunu Node'a özgü
  başlangıç yapılandırmasına göre değerlendirir; komut işleyicileri çağrıldıklarında
  kullanılabilirliği yine de doğrulamalıdır.

## `defineChannelPluginEntry`

**İçe aktarma:** `openclaw/plugin-sdk/channel-core`

`definePluginEntry` öğesini kanala özgü bağlantılarla sarmalar: otomatik olarak
`api.registerChannel({ plugin })` çağrısı yapar, isteğe bağlı bir kök yardım CLI
meta veri bağlantı noktası sunar ve `registerFull` öğesini kayıt moduna göre sınırlar.

```typescript
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Short summary",
  plugin: myChannelPlugin,
  setRuntime: setMyRuntime,
  registerCliMetadata(api) {
    api.registerCli(/* ... */);
  },
  registerFull(api) {
    api.registerGatewayMethod(/* ... */);
  },
});
```

| Alan                  | Tür                                                              | Zorunlu  | Varsayılan          |
| --------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                  | `string`                                                         | Evet     | -                   |
| `name`                | `string`                                                         | Evet     | -                   |
| `description`         | `string`                                                         | Evet     | -                   |
| `plugin`              | `ChannelPlugin`                                                  | Evet     | -                   |
| `configSchema`        | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | Hayır    | Boş nesne şeması    |
| `setRuntime`          | `(runtime: PluginRuntime) => void`                               | Hayır    | -                   |
| `registerCliMetadata` | `(api: OpenClawPluginApi) => void`                               | Hayır    | -                   |
| `registerFull`        | `(api: OpenClawPluginApi) => void`                               | Hayır    | -                   |

Geri çağrılar kayıt moduna göre çalışır (tam tablo
[Kayıt modu](#registration-mode) altında):

- `setRuntime`, `"cli-metadata"` ve
  `"tool-discovery"` dışındaki her modda çalışır. Çalışma zamanı başvurusunu burada, genellikle
  `createPluginRuntimeStore` aracılığıyla saklayın.
- `registerCliMetadata`; `"cli-metadata"`, `"discovery"` ve
  `"full"` için çalışır. Kök yardımın etkinleştirme yapmaması, keşif anlık görüntülerinin statik
  komut meta verilerini içermesi ve normal CLI kaydının tam
  plugin yüklemeleriyle uyumlu kalması için bunu kanala ait CLI tanımlayıcılarının temel konumu olarak kullanın.
- `registerFull`, yalnızca `"full"` ve `"tool-discovery"` için çalışır.
  `"tool-discovery"` için kanal kaydı _yerine_ çalışır: OpenClaw,
  `registerChannel`/`setRuntime` öğelerini tamamen atlar ve yalnızca
  `registerFull` çağrısını yapar; dolayısıyla kanalınızın bağımsız araç keşfi veya yürütmesi için
  ihtiyaç duyduğu tüm sağlayıcı/araç kayıtları normal kanal kurulumunun arkasında değil, burada
  bulunmalıdır.
- Keşif kaydı etkinleştirme yapmaz ancak içe aktarmasız değildir: OpenClaw,
  anlık görüntüyü oluşturmak için güvenilir plugin girişini ve kanal plugin modülünü
  değerlendirebilir. Üst düzey içe aktarmaları yan etkisiz tutun; soketleri,
  istemcileri, çalışanları ve hizmetleri yalnızca `"full"` yollarının arkasına yerleştirin.
- `definePluginEntry` gibi, `configSchema` de geç yüklenen bir fabrika olabilir; OpenClaw,
  çözümlenen şemayı ilk erişimde belleğe alır.

CLI kaydı:

- Kök CLI ayrıştırma ağacından kaybolmadan gecikmeli yüklenmesini istediğiniz, Plugin tarafından sahiplenilen kök
  CLI komutları için `api.registerCli(..., { descriptors: [...] })` kullanın.
  Tanımlayıcı adları bir harf veya rakamla başlamalı; yalnızca harfler, rakamlar, kısa çizgi ve
  alt çizgi içermelidir. OpenClaw diğer biçimleri reddeder ve yardım metnini
  oluşturmadan önce açıklamalardaki terminal kontrol dizilerini kaldırır.
  Kaydedicinin sunduğu her üst düzey komut kökünü kapsayın.
  Tek başına `commands`, istekli uyumluluk yolunda kalır.
- Eşlenmiş Node özellik komutlarının `openclaw nodes` altında
  (başka bir deyişle `registerCli(registrar, { parentPath: ["nodes"], ... })`) yer alması için
  `api.registerNodeCliFeature(...)` kullanın.
- Diğer iç içe Plugin komutları için `parentPath` ekleyin ve komutları
  kaydediciye iletilen `program` nesnesine kaydedin; OpenClaw, Plugin'i çağırmadan
  önce bunu üst komuta çözümler.
- Kanal Plugin'leri için CLI tanımlayıcılarını `registerCliMetadata` üzerinden
  kaydedin ve `registerFull` öğesini yalnızca çalışma zamanı işlerine odaklı tutun.
- `registerFull` ayrıca Gateway RPC yöntemlerini kaydediyorsa bunları
  Plugin'e özgü bir ön ek altında tutun. Ayrılmış çekirdek yönetici ad alanları (`config.*`,
  `exec.approvals.*`, `wizard.*`, `update.*`) her zaman
  `operator.admin` değerine zorlanır.

## `defineSetupPluginEntry`

**İçe aktarma:** `openclaw/plugin-sdk/channel-core`

Hafif `setup-entry.ts` dosyası içindir. Çalışma zamanı veya CLI bağlantısı olmadan
yalnızca `{ plugin }` döndürür.

```typescript
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineSetupPluginEntry(myChannelPlugin);
```

Bir kanal devre dışı, yapılandırılmamış olduğunda veya ertelenmiş yükleme
etkinleştirildiğinde OpenClaw tam giriş yerine bunu yükler. Bunun ne zaman önemli
olduğunu öğrenmek için [Kurulum ve Yapılandırma](/tr/plugins/sdk-setup#setup-entry) bölümüne bakın.

`defineSetupPluginEntry(...)` öğesini dar kapsamlı kurulum yardımcı aileleriyle eşleştirin:

| İçe aktarma                        | Kullanım amacı                                                                                                                                                                      |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw/plugin-sdk/setup-runtime` | Çalışma zamanı açısından güvenli kurulum yardımcıları: `createSetupTranslator`, içe aktarma açısından güvenli kurulum yaması bağdaştırıcıları, arama notu çıktısı, `promptResolvedAllowFrom`, `splitSetupEntries`, devredilmiş kurulum vekilleri |
| `openclaw/plugin-sdk/channel-setup` | İsteğe bağlı yükleme kurulum yüzeyleri                                                                                                                                              |
| `openclaw/plugin-sdk/setup-tools`   | Kurulum/yükleme CLI, arşiv ve dokümantasyon yardımcıları                                                                                                                            |

Ağır SDK'ları, CLI kaydını ve uzun ömürlü çalışma zamanı hizmetlerini
tam girişte tutun.

Kurulum ve çalışma zamanı yüzeylerini ayıran paketlenmiş çalışma alanı kanalları
bunun yerine `openclaw/plugin-sdk/channel-entry-contract` içindeki
`defineBundledChannelSetupEntry(...)` öğesini kullanabilir. Bu, kurulum
girişinin kurulum açısından güvenli Plugin/gizli bilgi dışa aktarımlarını korurken bir çalışma zamanı
ayarlayıcısını sunmaya devam etmesini sağlar:

```typescript
import { defineBundledChannelSetupEntry } from "openclaw/plugin-sdk/channel-entry-contract";

export default defineBundledChannelSetupEntry({
  importMetaUrl: import.meta.url,
  plugin: {
    specifier: "./channel-plugin-api.js",
    exportName: "myChannelPlugin",
  },
  runtime: {
    specifier: "./runtime-api.js",
    exportName: "setMyChannelRuntime",
  },
  registerSetupRuntime(api) {
    api.registerHttpRoute({
      path: "/my-channel/events",
      auth: "plugin",
      handler: async (req, res) => {
        /* kurulum açısından güvenli rota */
      },
    });
  },
});
```

Bunu yalnızca bir kurulum akışı, tam kanal girişi yüklenmeden önce gerçekten hafif bir
çalışma zamanı ayarlayıcısına veya kurulum açısından güvenli bir Gateway yüzeyine ihtiyaç duyduğunda kullanın.
`registerSetupRuntime` yalnızca `"setup-runtime"` yüklemeleri için çalışır; bunu
yapılandırmaya özel rotalarla veya ertelenmiş tam etkinleştirmeden önce var olması gereken
yöntemlerle sınırlı tutun.

## Kayıt modu

`api.registrationMode`, Plugin'inize nasıl yüklendiğini bildirir:

| Mod                | Zaman                                               | Kaydedilecekler                                                                                                          |
| ------------------ | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `"full"`           | Normal Gateway başlatması                          | Her şey                                                                                                                  |
| `"discovery"`      | Salt okunur yetenek keşfi                          | Kanal kaydı ve statik CLI tanımlayıcıları; giriş kodu yüklenebilir ancak yuvaları, çalışanları, istemcileri ve hizmetleri atlayın |
| `"tool-discovery"` | Belirli Plugin'lerin araçlarını listelemek veya çalıştırmak için kapsamlı yükleme | Yalnızca yetenek/araç kaydı; kanal etkinleştirmesi yok                                                                    |
| `"setup-only"`     | Devre dışı/yapılandırılmamış kanal                  | Yalnızca kanal kaydı                                                                                                     |
| `"setup-runtime"`  | Çalışma zamanının kullanılabildiği kurulum akışı    | Kanal kaydı ve yalnızca tam giriş yüklenmeden önce gereken hafif çalışma zamanı                                           |
| `"cli-metadata"`   | Kök yardım / CLI meta verisi yakalama              | Yalnızca CLI tanımlayıcıları                                                                                              |

`defineChannelPluginEntry` bu ayrımı otomatik olarak işler. Bir kanal için
doğrudan `definePluginEntry` kullanıyorsanız modu kendiniz denetleyin ve
`"tool-discovery"` öğesinin kanal kaydını atladığını unutmayın:

```typescript
register(api) {
  if (
    api.registrationMode === "cli-metadata" ||
    api.registrationMode === "discovery" ||
    api.registrationMode === "full"
  ) {
    api.registerCli(/* ... */);
    if (api.registrationMode === "cli-metadata") return;
  }

  if (api.registrationMode === "tool-discovery") {
    // Yalnızca yetenek yüzeylerini (sağlayıcılar/araçlar) kaydedin; kanal yok.
    return;
  }

  api.registerChannel({ plugin: myPlugin });
  if (api.registrationMode !== "full") return;

  // Yalnızca ağır çalışma zamanı kayıtları
  api.registerService(/* ... */);
}
```

Uzun ömürlü hizmetler, hizmet bağlamları üzerinden küçük geçersiz kılma veya yaşam döngüsü
olayları yayınlayabilir:

```typescript
api.registerService({
  id: "index-events",
  start(ctx) {
    ctx.gatewayEvents?.emit("changed", { revision: 1 }, { scope: "operator.read" });
  },
});
```

OpenClaw bunu `plugin.<plugin-id>.changed` olarak ad alanına alır. Olay adları tek bir
küçük harfli bölümden oluşmalı, yükler sınırlı JSON olmalı ve kapsam
`operator.read`, `operator.write` veya `operator.admin` olmalıdır. Yayıcı yalnızca
hizmetin ömrü boyunca var olur ve hizmet durdurulduktan veya başlatma başarısız olduktan sonra iptal edilir.
Yetkili istemcilerin Plugin'in kapsamlı Gateway yöntemleri üzerinden
standart durumu yeniden okuması için tam kayıtlar yerine sürüm veya geçersiz kılma yüklerini tercih edin.

Keşif modu, etkinleştirme yapmayan bir kayıt defteri anlık görüntüsü oluşturur. OpenClaw'ın
kanal yeteneklerini ve statik CLI tanımlayıcılarını kaydedebilmesi için Plugin girişini ve
kanal Plugin nesnesini yine de değerlendirebilir. Keşif sırasında modül
değerlendirmesini güvenilir ancak hafif olarak ele alın: üst düzeyde ağ istemcileri,
alt süreçler, dinleyiciler, veritabanı bağlantıları, arka plan çalışanları,
kimlik bilgisi okumaları veya diğer canlı çalışma zamanı yan etkileri bulunmamalıdır.

`"setup-runtime"` öğesini, tam paketlenmiş kanal çalışma zamanına yeniden girmeden
yalnızca kuruluma yönelik başlatma yüzeylerinin var olması gereken pencere olarak ele alın.
Kanal kaydı, kurulum açısından güvenli HTTP rotaları, kurulum açısından güvenli Gateway yöntemleri
ve devredilmiş kurulum yardımcıları buna uygundur. Ağır arka plan hizmetleri, CLI kaydedicileri ve
sağlayıcı/istemci SDK önyüklemeleri yine `"full"` içinde yer almalıdır.

## Plugin biçimleri

OpenClaw, yüklenen Plugin'leri kayıt davranışlarına göre sınıflandırır:

| Biçim                 | Açıklama                                           |
| --------------------- | -------------------------------------------------- |
| **plain-capability**  | Tek bir yetenek türü (ör. yalnızca sağlayıcı)      |
| **hybrid-capability** | Birden fazla yetenek türü (ör. sağlayıcı + konuşma) |
| **hook-only**         | Yalnızca kancalar, yetenek yok                     |
| **non-capability**    | Araçlar/komutlar/hizmetler var ancak yetenek yok   |

Bir Plugin'in biçimini görmek için `openclaw plugins inspect <id>` kullanın.

## İlgili

- [SDK Genel Bakışı](/tr/plugins/sdk-overview) - kayıt API'si ve alt yol başvurusu
- [Çalışma Zamanı Yardımcıları](/tr/plugins/sdk-runtime) - `api.runtime` ve `createPluginRuntimeStore`
- [Kurulum ve Yapılandırma](/tr/plugins/sdk-setup) - bildirim, kurulum girişi, ertelenmiş yükleme
- [Kanal Plugin'leri](/tr/plugins/sdk-channel-plugins) - `ChannelPlugin` nesnesini oluşturma
- [Sağlayıcı Plugin'leri](/tr/plugins/sdk-provider-plugins) - sağlayıcı kaydı ve kancalar
