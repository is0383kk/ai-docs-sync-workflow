---
read_when:
    - Yalnızca agent araçları ekleyen basit bir OpenClaw plugini oluşturmak istiyorsunuz
    - Plugin manifest meta verilerini elle yazmak yerine defineToolPlugin kullanmak istiyorsunuz
    - Yalnızca araçlardan oluşan bir pluginin iskeletini oluşturmanız, onu üretmeniz, doğrulamanız, test etmeniz veya yayımlamanız gerekiyor
sidebarTitle: Tool Plugins
summary: defineToolPlugin ve openclaw plugins init/build/validate ile basit, türü belirlenmiş aracı araçları oluşturun
title: Araç pluginleri
x-i18n:
    generated_at: "2026-07-26T22:57:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac23d15ba79cbdd1d8b8eab7c87007b44af16361b2866b14123e18f816bf4075
    source_path: plugins/tool-plugins.md
    workflow: 16
---

`defineToolPlugin` yalnızca ajanların çağırabileceği araçlar ekleyen bir plugin oluşturur: kanal,
model sağlayıcısı, kanca, hizmet veya kurulum arka ucu içermez. OpenClaw'un plugin
çalışma zamanı kodunu yüklemeden araçları keşfetmesi için gereken manifest meta verilerini
oluşturur.

Sağlayıcı, kanal, kanca, hizmet veya karma yetenekli pluginler için bunun yerine
[Plugin oluşturma](/tr/plugins/building-plugins), [Kanal Pluginleri](/tr/plugins/sdk-channel-plugins)
veya [Sağlayıcı Pluginleri](/tr/plugins/sdk-provider-plugins) ile başlayın.

## Gereksinimler

- Node 22.22.3+, Node 24.15+ veya Node 25.9+.
- TypeScript ESM paket çıktısı.
- `typebox`, `dependencies` içinde olmalıdır (yalnızca `devDependencies` içinde değil; oluşturulan
  plugin bunu çalışma zamanında içe aktarır).
- `openclaw >=2026.5.17`, `openclaw/plugin-sdk/tool-plugin` dışa aktarımını yapan ilk sürüm.
- `dist/`, `openclaw.plugin.json` ve
  `package.json` dosyalarını dağıtan bir paket kökü.

## Hızlı başlangıç

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm install
npm run plugin:build
npm run plugin:validate
npm test
```

`plugins init` şunları oluşturur:

| Dosya                  | Amaç                                                              |
| ---------------------- | ----------------------------------------------------------------- |
| `src/index.ts`         | Bir `echo` aracı içeren `defineToolPlugin` girişi                 |
| `src/index.test.ts`    | Araç listesini doğrulayan meta veri testi                          |
| `tsconfig.json`        | `dist/` konumuna NodeNext TypeScript çıktısı                       |
| `vitest.config.ts`     | `src/**/*.test.ts` için Vitest yapılandırması                      |
| `package.json`         | Betikler, çalışma zamanı bağımlılıkları, `openclaw.extensions: ["./dist/index.js"]` |
| `openclaw.plugin.json` | İlk araç için oluşturulan manifest meta verileri                   |

`npm run plugin:build`, `npm run build` (tsc) ve ardından
`openclaw plugins build --entry ./dist/index.js` komutunu çalıştırır. `npm run plugin:validate`
yeniden oluşturur ve `openclaw plugins validate --entry ./dist/index.js` komutunu çalıştırır.
Başarılı doğrulama şu çıktıyı verir:

```text
Plugin stock-quotes is valid.
```

`openclaw plugins init <id>` seçenekleri:

| Bayrak               | Varsayılan         | Etki                                   |
| -------------------- | ------------------ | -------------------------------------- |
| `--directory <path>` | `<id>`             | Çıktı dizini                           |
| `--name <name>`      | Başlık biçiminde `<id>` | Görünen ad                             |
| `--type <type>`      | `tool`             | Oluşturma türü: `tool` veya `provider` |
| `--force`            | kapalı             | Mevcut bir çıktı dizininin üzerine yaz |

## Araç yazma

`defineToolPlugin`, plugin kimliğini, isteğe bağlı bir yapılandırma şemasını ve
statik bir araç listesini alır. Parametre ve yapılandırma türleri
TypeBox şemalarından çıkarılır.

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "Fetch stock quote snapshots.",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "Quote API key." })),
    baseUrl: Type.Optional(Type.String({ description: "Quote API base URL." })),
  }),
  tools: (tool) => [
    tool({
      name: "stock_quote",
      label: "Stock Quote",
      description: "Fetch a stock quote snapshot.",
      parameters: Type.Object({
        symbol: Type.String({ description: "Ticker symbol, for example OPEN." }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          configured: Type.Boolean(),
          baseUrl: Type.String(),
        },
        { additionalProperties: false },
      ),
      async execute({ symbol }, config, context) {
        context.signal?.throwIfAborted();
        return {
          symbol: symbol.toUpperCase(),
          configured: Boolean(config.apiKey),
          baseUrl: config.baseUrl ?? "https://api.example.com",
        };
      },
    }),
  ],
});
```

Araç adları kararlı API'dir. Benzersiz, küçük harfli ve
çekirdek araçlarla veya diğer pluginlerle çakışmayı önleyecek kadar belirgin adlar seçin.

## İsteğe bağlı ve fabrika araçları

Kullanıcıların aracı bir modele gönderilmeden önce açıkça izin listesine alması
gerekiyorsa `optional: true` ayarlayın. `openclaw plugins build`, eşleşen
`toolMetadata.<tool>.optional` manifest girdisini yazar; böylece OpenClaw, plugin çalışma zamanı
kodunu yüklemeden aracın isteğe bağlı olduğunu görebilir.

```typescript
tool({
  name: "workflow_run",
  description: "Run an external workflow.",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  execute: ({ goal }) => ({ queued: true, goal }),
});
```

Bir aracın oluşturulabilmesi için önce çalışma zamanı araç bağlamına ihtiyacı olduğunda;
belirli bir çalıştırmada devre dışı kalmak, sandbox durumunu incelemek veya
çalışma zamanı yardımcılarını bağlamak için `factory` kullanın. Somut araç
çalışma zamanında oluşturulsa da meta veriler statik kalır.

```typescript
tool({
  name: "local_workflow",
  description: "Run a local workflow outside sandboxed sessions.",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  factory({ api, toolContext }) {
    if (toolContext.sandboxed) {
      return null;
    }
    return createLocalWorkflowTool(api);
  },
});
```

Fabrikalar yine de sabit bir araç adını önceden bildirir. Plugin araç adlarını
dinamik olarak hesapladığında veya araçları kancalar, hizmetler, sağlayıcılar
ya da komutlarla birleştirdiğinde doğrudan `definePluginEntry` kullanın.

## Dönüş değerleri

`defineToolPlugin`, düz dönüş değerlerini OpenClaw araç sonucu
biçimine sarar:

- Modelin tam olarak bu metni görmesi gerektiğinde bir dize döndürün.
- Modelin biçimlendirilmiş JSON görmesini ve OpenClaw'un özgün değeri
  `details` içinde tutmasını istediğinizde JSON uyumlu bir değer döndürün.

```typescript
tool({
  name: "echo_text",
  description: "Echo input text.",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => input,
});
```

```typescript
tool({
  name: "echo_json",
  description: "Echo input as structured JSON.",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => ({ input, length: input.length }),
});
```

Özel bir `AgentToolResult` gerektiğinde veya mevcut bir
`api.registerTool` uygulamasını yeniden kullanmak istediğinizde fabrika aracı kullanın.

## Çıktı sözleşmeleri

Bir araç kararlı, JSON uyumlu veriler döndürdüğünde `outputSchema` ekleyin. Bu,
`content` içindeki biçimlendirilmiş metni değil, `AgentToolResult.details` içinde
saklanan özgün değeri açıklar:

```typescript
tool({
  name: "shipment_list",
  description: "List shipments.",
  parameters: Type.Object({
    buyer: Type.Optional(Type.String()),
  }),
  outputSchema: Type.Array(
    Type.Object(
      {
        id: Type.String(),
        buyer: Type.String(),
        paid: Type.Boolean(),
        tons: Type.Number(),
      },
      { additionalProperties: false },
    ),
  ),
  execute: ({ buyer }) => listShipments(buyer),
});
```

[Code Mode](/tr/tools/code-mode) ve [Araç Arama](/tr/tools/tool-search), bu
şemayı sınırlandırılmış TypeScript tarzı bir çıktı ipucuna dönüştürür. Bu sayede model,
sonucun yapısını gözlemlemek için başka bir model turu harcamak yerine bilinen bir
sonucu tek program içinde çağırıp dönüştürebilir.

OpenClaw, bir katalog çağrısını yürütmeden önce şemayı derler; ardından araç kancalarından
sonra nihai `details` değerini köprü üzerinden döndürmeden önce doğrular.
Geçersiz bir şema aracın çalışmasına izin vermez; sonuç uyuşmazlığı tamamlanan
çağrının başarısız olmasına neden olur. Yapılandırılmış hata varyantları da dahil olmak üzere
istisna oluşturmayan tüm sonuç varyantlarını ekleyin veya sonuç kararlı değilse şemayı
kullanmayın. Güvenilir çıktı meta verileri model tarafından görünür hâle gelebileceğinden
şema açıklamalarına gizli veya hassas değerler koymayın.
Eksiksiz ve kompakt bir çıktı ipucu istediğinizde nesne katmanlarında
`{ additionalProperties: false }` kullanın; açık veya kesilmiş şemalar `tools.describe(...)`
üzerinden kullanılabilir kalır ancak eksiksiz hızlı dizin sözleşmeleri olarak duyurulmaz.

Fabrika araçları, döndürdükleri somut `AnyAgentTool` üzerinde
`outputSchema` bildirir. Statik `tool({ factory })` bildirimi, çalışma zamanı
aracıyla uyumsuz hâle gelebileceği için ayrı bir çıktı şeması kabul etmez.

## Yapılandırma

`configSchema` isteğe bağlıdır. Bunu atladığınızda OpenClaw katı bir boş nesne
şeması uygular; oluşturulan manifest yine de `configSchema` içerir.

```typescript
export default defineToolPlugin({
  id: "no-config-tools",
  name: "No Config Tools",
  description: "Adds tools that do not need configuration.",
  tools: () => [],
});
```

Bir `configSchema` kullanıldığında ikinci `execute` bağımsız değişkeninin
türü bundan çıkarılır:

```typescript
const configSchema = Type.Object({
  apiKey: Type.String(),
});

export default defineToolPlugin({
  id: "configured-tools",
  name: "Configured Tools",
  description: "Adds configured tools.",
  configSchema,
  tools: (tool) => [
    tool({
      name: "configured_ping",
      description: "Check whether configuration is available.",
      parameters: Type.Object({}),
      execute: (_params, config) => ({ hasKey: config.apiKey.length > 0 }),
    }),
  ],
});
```

OpenClaw, plugin yapılandırmasını Gateway yapılandırmasındaki plugin girdisinden okur.
Gizli değerleri kaynak koduna veya dokümantasyon örneklerine sabit kodlamayın; pluginin
güvenlik modeline uygun olarak yapılandırma, ortam değişkenleri veya SecretRef'ler kullanın.

## Oluşturulan meta veriler

OpenClaw, plugin çalışma zamanı kodunu içe aktarmadan önce plugin manifestini okumalıdır.
`defineToolPlugin` bunun için statik meta verileri sunar ve
`openclaw plugins build` bunları pakete yazar. Plugin kimliğini, adını, açıklamasını,
yapılandırma şemasını, etkinleştirmesini veya araç adlarını değiştirdikten sonra
oluşturucuyu yeniden çalıştırın:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

Tek araçlı bir plugin için oluşturulan manifest:

```json
{
  "id": "stock-quotes",
  "name": "Stock Quotes",
  "description": "Fetch stock quote snapshots.",
  "version": "0.1.0",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  },
  "activation": {
    "onStartup": true
  },
  "contracts": {
    "tools": ["stock_quote"]
  }
}
```

`contracts.tools` önemli keşif sözleşmesidir: OpenClaw'a, kurulu her pluginin
çalışma zamanını yüklemeden her aracın hangi plugine ait olduğunu bildirir. Güncel olmayan
bir manifest, aracın keşifte bulunamamasına veya kayıt hatasının yanlış plugine
yüklenmesine neden olabilir.

## Paket meta verileri

`openclaw plugins build` ayrıca `package.json` değerini seçilen çalışma zamanı
girdisiyle hizalar:

```json
{
  "type": "module",
  "files": ["dist", "openclaw.plugin.json", "README.md"],
  "dependencies": {
    "typebox": "^1.1.38"
  },
  "peerDependencies": {
    "openclaw": ">=2026.5.17"
  },
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

TypeScript kaynak girdisini değil, oluşturulmuş JavaScript'i (`./dist/index.js`) dağıtın.
Kaynak girdileri yalnızca çalışma alanı içindeki yerel geliştirmede çalışır.

## CI'da doğrulama

Oluşturulan meta veriler güncel değilse `plugins build --check`, dosyaları yeniden
yazmadan başarısız olur:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js --check
openclaw plugins validate --entry ./dist/index.js
npm test
```

OpenClaw SDK uyumluluk alanları, düzenleyicilerin geçiş uyarıları olarak gösterdiği
TypeScript `@deprecated` ek açıklamalarını taşır. Bunları CI'da zorunlu kılmak için
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/) gibi
tür bilgisine duyarlı bir kuralı etkinleştirin.
Oxlint tür bilgisine duyarlı olmadığından bu ek açıklamaları zorunlu kılamaz. Bu nedenle
oluşturulan `plugins init` iskeleti bir kullanımdan kaldırma lint yapılandırması eklemez.

`plugins validate` şunları denetler:

- `openclaw.plugin.json` mevcut ve normal manifest yükleyicisinden geçiyor.
- Geçerli giriş, `defineToolPlugin` meta verilerini dışa aktarıyor.
- Oluşturulan manifest alanları giriş meta verileriyle eşleşiyor.
- `contracts.tools` bildirilen araç adlarıyla eşleşiyor.
- `package.json`, `openclaw.extensions` öğesini seçilen çalışma zamanı girişine yönlendiriyor.

## Yerel olarak yükleme ve inceleme

Ayrı bir OpenClaw çalışma kopyasından veya yüklü CLI'dan paket yolunu yükleyin:

```bash
openclaw plugins install ./stock-quotes
openclaw plugins inspect stock-quotes --runtime
```

Paketlenmiş bir temel doğrulama testi için önce paketi oluşturun ve tarball dosyasını yükleyin:

```bash
npm pack
openclaw plugins install npm-pack:./openclaw-plugin-stock-quotes-0.1.0.tgz
openclaw plugins inspect stock-quotes --runtime --json
```

Yüklemeden sonra Gateway'i yeniden başlatın veya yeniden yükleyin ve ajandan
aracı kullanmasını isteyin. Araç görünmüyorsa kodu değiştirmeden önce Plugin
çalışma zamanını ve etkin araç kataloğunu inceleyin (bkz.
[Sorun giderme](#troubleshooting)).

## Yayımlama

Paket hazır olduğunda ClawHub aracılığıyla yayımlayın. `clawhub package publish`
bir kaynak alır: yerel klasör, GitHub deposu (`owner/repo[@ref]`) veya
tarball URL'si.

```bash
clawhub package publish ./stock-quotes --dry-run
clawhub package publish ./stock-quotes
```

Açık bir ClawHub konum belirleyicisiyle yükleyin:

```bash
openclaw plugins install clawhub:your-org/stock-quotes
```

Yalın npm paket belirtimleri, kullanıma geçiş sırasında npm'den yüklenmeye
devam eder; ancak ClawHub, OpenClaw Plugin'leri için tercih edilen keşif ve
dağıtım yüzeyidir. Sahip kapsamı ve sürüm incelemesi için
[ClawHub'da yayımlama](/tr/clawhub/publishing) bölümüne bakın.

## Sorun giderme

### `plugin entry not found: ./dist/index.js`

Seçilen giriş dosyası mevcut değil. `npm run build` komutunu çalıştırın,
ardından `openclaw plugins build --entry ./dist/index.js` veya
`openclaw plugins validate --entry ./dist/index.js` komutunu yeniden çalıştırın.

### `plugin entry does not expose defineToolPlugin metadata`

Giriş, `defineToolPlugin` tarafından oluşturulan bir değeri dışa aktarmadı.
Modülün varsayılan dışa aktarımının `defineToolPlugin(...)` sonucu olduğunu
doğrulayın veya `--entry` ile doğru girişi iletin.

### `openclaw.plugin.json generated metadata is stale`

Manifest artık giriş meta verileriyle eşleşmiyor. Şunları çalıştırın:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

Hem `openclaw.plugin.json` hem de `package.json` değişikliklerini kaydedin.

### `package.json openclaw.extensions must include ./dist/index.js`

Paket meta verileri farklı bir çalışma zamanı girişine işaret ediyor.
Oluşturucunun paket meta verilerini yayımlamayı amaçladığınız girişle
hizalaması için `openclaw plugins build --entry ./dist/index.js` komutunu çalıştırın.

### `Cannot find package 'typebox'`

Derlenen Plugin, çalışma zamanında `typebox` öğesini içe aktarıyor.
Bunu `dependencies` içinde tutun; yeniden yükleyin, yeniden derleyin ve
doğrulamayı tekrar çalıştırın.

### Araç yüklemeden sonra görünmüyor

Şunları sırayla kontrol edin:

1. `openclaw plugins inspect <plugin-id> --runtime`
2. `openclaw plugins validate --root <plugin-root> --entry ./dist/index.js`
3. `openclaw.plugin.json`, beklenen araç adlarını içeren `contracts.tools` öğesine sahip.
4. `package.json`, `openclaw.extensions: ["./dist/index.js"]` öğesine sahip.
5. Plugin yüklendikten sonra Gateway yeniden başlatıldı veya yeniden yüklendi.

## Ayrıca bkz.

- [Plugin oluşturma](/tr/plugins/building-plugins)
- [Plugin giriş noktaları](/tr/plugins/sdk-entrypoints)
- [Plugin SDK alt yolları](/tr/plugins/sdk-subpaths)
- [Plugin manifesti](/tr/plugins/manifest)
- [Plugin'ler CLI'ı](/tr/cli/plugins)
- [ClawHub'da yayımlama](/tr/clawhub/publishing)
