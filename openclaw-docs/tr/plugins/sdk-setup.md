---
read_when:
    - Bir Plugin'e kurulum sihirbazı ekliyorsunuz
    - setup-entry.ts ile index.ts arasındaki farkı anlamanız gerekir
    - Plugin yapılandırma şemalarını veya package.json openclaw meta verilerini tanımlıyorsunuz
sidebarTitle: Setup and config
summary: Kurulum sihirbazları, setup-entry.ts, yapılandırma şemaları ve package.json meta verileri
title: Plugin kurulumu ve yapılandırması
x-i18n:
    generated_at: "2026-07-26T23:54:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b07e3fa365939fa9c0885b31b7894f5e734313a7deef2297e316956063d97e45
    source_path: plugins/sdk-setup.md
    workflow: 16
---

Plugin paketleme (`package.json` meta verileri), manifestler (`openclaw.plugin.json`), kurulum girişleri ve yapılandırma şemaları için başvuru kaynağı.

<Tip>
**Adım adım açıklama mı arıyorsunuz?** Nasıl yapılır kılavuzları, paketlemeyi bağlam içinde ele alır: [Kanal pluginleri](/plugins/sdk-channel-plugins#step-1-package-and-manifest) ve [Sağlayıcı pluginleri](/tr/plugins/sdk-provider-plugins#step-1-package-and-manifest).
</Tip>

## Paket meta verileri

`package.json` dosyanız, plugin sistemine plugininizin ne sağladığını bildiren bir `openclaw` alanına ihtiyaç duyar:

<Tabs>
  <Tab title="Kanal plugini">
    ```json
    {
      "name": "@myorg/openclaw-my-channel",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "my-channel",
          "label": "Kanalım",
          "blurb": "Kanalın kısa açıklaması."
        }
      }
    }
    ```
  </Tab>
  <Tab title="Sağlayıcı plugini / ClawHub temel yapılandırması">
    ```json openclaw-clawhub-package.json
    {
      "name": "@myorg/openclaw-my-plugin",
      "version": "1.0.0",
      "type": "module",
      "dependencies": {
        "typebox": "1.1.39"
      },
      "peerDependencies": {
        "openclaw": ">=2026.3.24-beta.2"
      },
      "openclaw": {
        "extensions": ["./index.ts"],
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
  </Tab>
</Tabs>

<Note>
ClawHub'da harici olarak yayımlamak için `compat` ve `build` gereklidir. Standart yayımlama kod parçacıkları `docs/snippets/plugin-publish/` içinde bulunur.
</Note>

### `openclaw` alanları

<ParamField path="extensions" type="string[]">
  Giriş noktası dosyaları (paket köküne göre). Çalışma alanı ve git çalışma kopyası geliştirmesi için geçerli kaynak girişleridir.
</ParamField>
<ParamField path="runtimeExtensions" type="string[]">
  `extensions` için derlenmiş JavaScript eş dosyaları; OpenClaw yüklü bir npm paketini yüklediğinde tercih edilir. Kaynak/derlenmiş çözümleme sırası için [SDK giriş noktaları](/tr/plugins/sdk-entrypoints) bölümüne bakın.
</ParamField>
<ParamField path="setupEntry" type="string">
  Yalnızca kurulum için kullanılan hafif giriş (isteğe bağlı).
</ParamField>
<ParamField path="runtimeSetupEntry" type="string">
  `setupEntry` için derlenmiş JavaScript eş dosyası. `setupEntry` değerinin de ayarlanmasını gerektirir.
</ParamField>
<ParamField path="plugin" type="object">
  Bir pluginin kimlik veya etiket türetilebilecek kanal/sağlayıcı meta verileri olmadığında kullanılan `{ id, label }` yedek plugin kimliği.
</ParamField>
<ParamField path="channel" type="object">
  Kurulum, seçici, hızlı başlangıç ve durum yüzeyleri için kanal kataloğu meta verileri.
</ParamField>
<ParamField path="install" type="object">
  Yükleme ipuçları: `npmSpec`, `localPath`, `defaultChoice`, `minHostVersion`, `expectedIntegrity`, `allowInvalidConfigRecovery`, `requiredPlatformPackages`.
</ParamField>
<ParamField path="startup" type="object">
  Başlangıç davranışı bayrakları.
</ParamField>
<ParamField path="compat" type="object">
  Bu pluginin desteklediği `pluginApi` sürüm aralığı. Harici ClawHub yayımları için gereklidir.
</ParamField>

<Note>
Sağlayıcı kimlikleri (`providers: string[]`) paket meta verileri değil, manifest meta verileridir. Bunları burada değil, `openclaw.plugin.json` içinde bildirin — [Plugin manifesti](/tr/plugins/manifest) bölümüne bakın.
</Note>

### `openclaw.channel`

`openclaw.channel`, çalışma zamanı yüklenmeden önce kanal keşfi ve kurulum yüzeyleri için düşük maliyetli paket meta verileridir.

### Kanalın sahip olduğu kurulum alanları

Kanal pluginleri, kurulum alanlarını çalışma zamanı kodunda `defineChannelSetupContract(...)` ile bir kez tanımlamalı ve eşleşen serileştirilebilir izdüşümü `openclaw.channel.setup.fields` altında yayımlamalıdır. Çalışma zamanı tanımı, plugine özgü yerel girdi türünü çıkarır, hem yönlendirmeli hem de etkileşimsiz değerleri ayrıştırır ve kanala özgü anahtarları çekirdek türlerin dışında tutar. Paket meta verileri, `openclaw channels add <channel-id> --help` ve `openclaw channels add --channel <channel-id> --help` bileşenlerinin plugini yüklemeden yalnızca seçilen kanalın seçeneklerini keşfetmesini sağlar.

```ts
import { defineChannelSetupContract } from "openclaw/plugin-sdk/channel-setup";

export const setupContract = defineChannelSetupContract({
  fields: {
    endpoint: {
      kind: "string",
      cli: { flags: "--endpoint <url>", description: "Hizmet uç noktası" },
    },
    transport: {
      kind: "choice",
      choices: ["native", "container"],
      cli: { flags: "--transport <kind>", description: "Aktarım sahibi" },
    },
  },
  adapter: {
    applyAccountConfig: ({ cfg, input }) => ({
      ...cfg,
      channels: { ...cfg.channels, example: input },
    }),
  },
});
```

```json
{
  "openclaw": {
    "channel": {
      "id": "example",
      "setup": {
        "fields": [
          {
            "key": "endpoint",
            "kind": "string",
            "cli": { "flags": "--endpoint <url>", "description": "Hizmet uç noktası" }
          },
          {
            "key": "transport",
            "kind": "choice",
            "choices": ["native", "container"],
            "cli": { "flags": "--transport <kind>", "description": "Aktarım sahibi" }
          }
        ]
      }
    }
  }
}
```

Desteklenen alan türleri `string`, `boolean`, `integer`, `string-list` ve `choice` şeklindedir. Kimlik bilgileri için `sensitive: true` kullanın. Her alan anahtarı, olumsuz biçimler dâhil olmak üzere uzun CLI bayrağının camelCase biçimli öznitelik adına eşit olmalıdır; örneğin `--api-token` için `apiToken`. Hem olumlu hem de `--no-*` biçimleri gerektiğinde Boolean alanları `cli.negatedFlags` ekleyebilir. `channel`, `account` ve hesap görüntüleme `name` ortak denetim zarfı olarak kalır.

Yayımlanmış `setup`/`ChannelSetupInput` bağdaştırıcısı, mevcut harici pluginler için kullanılabilir olmaya devam eder. Yeni pluginler `setupContract` sunmalıdır; her ikisi de mevcut olduğunda OpenClaw her zaman bunu tercih eder.

| Alan                                   | Tür        | Anlamı                                                                        |
| -------------------------------------- | ---------- | ----------------------------------------------------------------------------- |
| `id`                    | `string` | Standart kanal kimliği.                                                       |
| `label`                    | `string` | Birincil kanal etiketi.                                                       |
| `selectionLabel`                    | `string` | `label` değerinden farklı olması gerektiğinde seçici/kurulum etiketi. |
| `detailLabel`                    | `string` | Daha zengin kanal katalogları ve durum yüzeyleri için ikincil ayrıntı etiketi. |
| `docsPath`                    | `string` | Kurulum ve seçim bağlantıları için doküman yolu.                              |
| `docsLabel`                    | `string` | Kanal kimliğinden farklı olması gerektiğinde doküman bağlantıları için kullanılan geçersiz kılma etiketi. |
| `blurb`                    | `string` | Kısa ilk kullanım/katalog açıklaması.                                        |
| `order`                    | `number` | Kanal kataloglarındaki sıralama düzeni.                                       |
| `aliases`                    | `string[]` | Kanal seçimi için ek arama diğer adları.                                      |
| `preferOver`                    | `string[]` | Bu kanalın önüne geçmesi gereken, daha düşük öncelikli plugin/kanal kimlikleri. |
| `systemImage`                    | `string` | Kanal kullanıcı arayüzü katalogları için isteğe bağlı simge/sistem görüntüsü adı. |
| `selectionDocsPrefix`                    | `string` | Seçim yüzeylerindeki doküman bağlantılarından önce gelen önek metni.          |
| `selectionDocsOmitLabel`                    | `boolean` | Seçim metninde etiketli bir doküman bağlantısı yerine doküman yolunu doğrudan gösterir. |
| `selectionExtras`                    | `string[]` | Seçim metnine eklenen kısa ek dizeler.                                        |
| `markdownCapable`                    | `boolean` | Giden biçimlendirme kararları için kanalı Markdown özellikli olarak işaretler. |
| `exposure`                    | `object` | Kurulum, yapılandırılmış listeler ve doküman yüzeyleri için kanal görünürlük denetimleri. |
| `quickstartAllowFrom`                    | `boolean` | Bu kanalı standart hızlı başlangıç `allowFrom` kurulum akışına dâhil eder. |
| `forceAccountBinding`                    | `boolean` | Yalnızca bir hesap bulunsa bile açık hesap bağlama işlemini zorunlu kılar.     |
| `preferSessionLookupForAnnounceTarget`                    | `boolean` | Bu kanal için duyuru hedeflerini çözümlerken oturum aramasını tercih eder.     |
| `setup`                    | `object` | Gecikmeli CLI seçeneği keşfi için kullanılan, kanalın sahip olduğu serileştirilebilir kurulum alanları. |

Örnek:

```json
{
  "openclaw": {
    "channel": {
      "id": "my-channel",
      "label": "Kanalım",
      "selectionLabel": "Kanalım (kendi sunucunuzda barındırılan)",
      "detailLabel": "Kanalım Botu",
      "docsPath": "/channels/my-channel",
      "docsLabel": "my-channel",
      "blurb": "Webhook tabanlı, kendi sunucunuzda barındırılan sohbet entegrasyonu.",
      "order": 80,
      "aliases": ["mc"],
      "preferOver": ["my-channel-legacy"],
      "selectionDocsPrefix": "Kılavuz:",
      "selectionExtras": ["Markdown"],
      "markdownCapable": true,
      "exposure": {
        "configured": true,
        "setup": true,
        "docs": true
      },
      "quickstartAllowFrom": true
    }
  }
}
```

`exposure` şunları destekler:

- `configured`: kanalı yapılandırılmış/durum tarzı listeleme yüzeylerine dâhil eder
- `setup`: kanalı etkileşimli kurulum/yapılandırma seçicilerine dâhil eder
- `docs`: kanalı doküman/gezinme yüzeylerinde herkese açık olarak işaretler

### `openclaw.install`

`openclaw.install`, manifest meta verisi değil, paket meta verisidir.

| Alan                         | Tür                                 | Anlamı                                                                            |
| ---------------------------- | ----------------------------------- | --------------------------------------------------------------------------------- |
| `clawhubSpec`                | `string`                            | Kurulum/güncelleme ve ilk katılım sırasında isteğe bağlı kurulum akışları için standart ClawHub belirtimi. |
| `npmSpec`                    | `string`                            | Kurulum/güncelleme yedek akışları için standart npm belirtimi.                    |
| `localPath`                  | `string`                            | Yerel geliştirme veya paketle birlikte gelen kurulum yolu.                        |
| `defaultChoice`              | `"clawhub"` \| `"npm"` \| `"local"` | Birden fazla kaynak kullanılabilir olduğunda tercih edilen kurulum kaynağı.       |
| `minHostVersion`             | `string`                            | Desteklenen minimum OpenClaw sürümü; `>=x.y.z` veya `>=x.y.z-prerelease`.         |
| `expectedIntegrity`          | `string`                            | Sabitlenmiş kurulumlar için genellikle `sha512-...` olan, beklenen npm dağıtım bütünlüğü dizesi. |
| `allowInvalidConfigRecovery` | `boolean`                           | Paketle birlikte gelen Plugin yeniden kurulum akışlarının belirli eski yapılandırma hatalarından kurtulmasını sağlar. |
| `requiredPlatformPackages`   | `string[]`                          | npm kurulumu sırasında doğrulanan, platforma özgü gerekli npm takma adları.       |

<AccordionGroup>
  <Accordion title="İlk katılım davranışı">
    Etkileşimli ilk katılım, isteğe bağlı kurulum yüzeyleri için `openclaw.install` kullanır: Plugin'iniz çalışma zamanı yüklenmeden önce sağlayıcı kimlik doğrulama seçeneklerini veya kanal kurulumu/katalog meta verilerini sunuyorsa ilk katılım, ClawHub, npm veya yerel kurulum seçimini isteyebilir; Plugin'i kurabilir ya da etkinleştirebilir ve ardından seçilen akışa devam edebilir. ClawHub seçenekleri `clawhubSpec` kullanır ve mevcut olduklarında tercih edilir; npm seçenekleri, kayıt defteri `npmSpec` içeren güvenilir katalog meta verileri gerektirir (tam sürümler ve `expectedIntegrity` isteğe bağlı sabitlemelerdir; ayarlandıklarında kurulum/güncelleme sırasında zorunlu tutulurlar). “Neyin gösterileceğini” `openclaw.plugin.json` içinde, “nasıl kurulacağını” ise `package.json` içinde tutun.
  </Accordion>
  <Accordion title="minHostVersion zorunluluğu">
    `minHostVersion` ayarlanmışsa hem kurulum hem de paketle birlikte gelmeyen bildirim-kayıt defteri yüklemesi bunu zorunlu tutar. Daha eski ana makineler harici Plugin'leri atlar; geçersiz sürüm dizeleri reddedilir. Paketle birlikte gelen kaynak Plugin'lerin, ana makine kaynak kodu teslimiyle aynı sürümde olduğu varsayılır.
  </Accordion>
  <Accordion title="Sabitlenmiş npm kurulumları">
    Sabitlenmiş npm kurulumlarında tam sürümü `npmSpec` içinde tutun ve beklenen yapıt bütünlüğünü ekleyin:

    ```json
    {
      "openclaw": {
        "install": {
          "npmSpec": "@wecom/wecom-openclaw-plugin@1.2.3",
          "expectedIntegrity": "sha512-REPLACE_WITH_NPM_DIST_INTEGRITY",
          "defaultChoice": "npm"
        }
      }
    }
    ```

  </Accordion>
  <Accordion title="allowInvalidConfigRecovery kapsamı">
    `allowInvalidConfigRecovery`, bozuk yapılandırmalar için genel bir atlatma yöntemi değildir. Yalnızca paketle birlikte gelen Plugin'lere yönelik dar kapsamlı bir kurtarma özelliğidir ve yeniden kurulumun/kurulumun, eksik bir paketle birlikte gelen Plugin yolu veya aynı Plugin'e ait eski bir `channels.<id>` girdisi gibi bilinen yükseltme kalıntılarını onarmasını sağlar. Yapılandırma ilgisiz nedenlerle bozuksa kurulum yine kapalı biçimde başarısız olur ve operatöre `openclaw doctor --fix` çalıştırmasını söyler.
  </Accordion>
</AccordionGroup>

### Ertelenmiş tam yükleme

Kanal Plugin'leri, aşağıdaki yapılandırmayla ertelenmiş yüklemeyi seçebilir:

```json
{
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

Etkinleştirildiğinde OpenClaw, ön dinleme başlangıç aşamasında önceden yapılandırılmış kanallar için bile yalnızca `setupEntry` yükler. Tam giriş, Gateway dinlemeye başladıktan sonra yüklenir.

<Warning>
Ertelenmiş yüklemeyi yalnızca `setupEntry`, Gateway'in dinlemeye başlamadan önce ihtiyaç duyduğu her şeyi (kanal kaydı, HTTP yolları, Gateway yöntemleri) kaydediyorsa etkinleştirin. Gerekli başlangıç yetenekleri tam girişe aitse varsayılan davranışı koruyun.
</Warning>

Kurulum/tam girişiniz Gateway RPC yöntemlerini kaydediyorsa bunları Plugin'e özgü bir ön ek altında tutun. Ayrılmış temel yönetici ad alanları (`config.*`, `exec.approvals.*`, `wizard.*`, `update.*`) temel bileşenin mülkiyetinde kalır ve her zaman `operator.admin` olarak normalleştirilir.

## Plugin bildirimi

Her yerel Plugin, paket kökünde bir `openclaw.plugin.json` sunmalıdır. OpenClaw bunu, Plugin kodunu çalıştırmadan yapılandırmayı doğrulamak için kullanır.

```json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "OpenClaw'a My Plugin yetenekleri ekler",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "webhookSecret": {
        "type": "string",
        "description": "Webhook doğrulama gizli anahtarı"
      }
    }
  }
}
```

Kanal Plugin'leri için `channels` ekleyin (sağlayıcı Plugin'leri de `providers` ekler):

```json
{
  "id": "my-channel",
  "channels": ["my-channel"],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  }
}
```

Yapılandırması olmayan Plugin'ler bile bir şema sunmalıdır. Boş bir şema geçerlidir:

```json
{
  "id": "my-plugin",
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

Şema başvurusunun tamamı için [Plugin bildirimi](/tr/plugins/manifest) bölümüne bakın.

## ClawHub'da yayımlama

Skills ve Plugin paketleri ayrı ClawHub yayımlama komutları kullanır. Plugin paketleri için pakete özgü komutu kullanın:

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

<Note>
`clawhub skill publish <path>`, bir Plugin paketini değil bir beceri klasörünü yayımlamak için kullanılan farklı bir komuttur. Bkz. [ClawHub'da Yayımlama](/tr/clawhub/publishing).
</Note>

## Kurulum girişi

`setup-entry.ts`, OpenClaw'ın yalnızca kurulum yüzeylerine (ilk katılım, yapılandırma onarımı, devre dışı bırakılmış kanal incelemesi) ihtiyaç duyduğunda yüklediği, `index.ts` için hafif bir alternatiftir:

```typescript
// setup-entry.ts
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
import { myChannelPlugin } from "./src/channel.js";

export default defineSetupPluginEntry(myChannelPlugin);
```

Bu, kurulum akışları sırasında ağır çalışma zamanı kodunun (kriptografi kitaplıkları, CLI kayıtları, arka plan hizmetleri) yüklenmesini önler.

Kurulum için güvenli dışa aktarımları yardımcı modüllerde tutan, çalışma alanıyla birlikte paketlenmiş kanallar, `defineSetupPluginEntry(...)` yerine `openclaw/plugin-sdk/channel-entry-contract` kaynağındaki `defineBundledChannelSetupEntry(...)` öğesini kullanabilir. Paketle birlikte gelen bu sözleşme, kurulum zamanı çalışma ortamı bağlantılarının hafif ve açık kalabilmesi için isteğe bağlı bir `runtime` dışa aktarımını da destekler.

<AccordionGroup>
  <Accordion title="OpenClaw'ın tam giriş yerine setupEntry kullandığı durumlar">
    - Kanal devre dışıdır ancak kurulum/ilk katılım yüzeylerine ihtiyaç duyar.
    - Kanal etkindir ancak yapılandırılmamıştır.
    - Ertelenmiş yükleme etkindir (`deferConfiguredChannelFullLoadUntilAfterListen`).

  </Accordion>
  <Accordion title="setupEntry'nin kaydetmesi gerekenler">
    - Kanal Plugin nesnesi (`defineSetupPluginEntry` aracılığıyla).
    - Gateway dinlemeden önce gerekli olan tüm HTTP yolları.
    - Başlangıç sırasında gereken tüm Gateway yöntemleri.

    Bu başlangıç Gateway yöntemleri yine de `config.*` veya `update.*` gibi ayrılmış temel yönetici ad alanlarından kaçınmalıdır.

  </Accordion>
  <Accordion title="setupEntry'nin İÇERMEMESİ gerekenler">
    - CLI kayıtları.
    - Arka plan hizmetleri.
    - Ağır çalışma zamanı içe aktarımları (kriptografi, SDK'lar).
    - Yalnızca başlangıçtan sonra gereken Gateway yöntemleri.

  </Accordion>
</AccordionGroup>

### Dar kapsamlı kurulum yardımcısı içe aktarımları

Yalnızca kurulum amaçlı sıcak yollar için, kurulum yüzeyinin yalnızca bir kısmına ihtiyacınız olduğunda daha geniş kapsamlı `plugin-sdk/setup` çatı öğesi yerine dar kapsamlı kurulum yardımcısı bağlantılarını tercih edin:

| İçe aktarma yolu           | Kullanım amacı                                                                            | Temel dışa aktarımlar                                                                                                                                                                                                                                                                                                  |
| -------------------------- | ----------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `plugin-sdk/setup-runtime` | `setupEntry` / ertelenmiş kanal başlangıcında kullanılabilir durumda kalan kurulum zamanı çalışma ortamı yardımcıları | `createSetupTranslator`, `createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`, `createSetupInputPresenceValidator`, `noteChannelLookupFailure`, `noteChannelLookupSummary`, `promptResolvedAllowFrom`, `splitSetupEntries`, `createAllowlistSetupWizardProxy`, `createDelegatedSetupWizardProxy` |
| `plugin-sdk/setup-tools`   | kurulum/yükleme CLI/arşiv/belge yardımcıları                                              | `formatCliCommand`, `detectBinary`, `extractArchive`, `resolveBrewExecutable`, `formatDocsLink`, `CONFIG_DIR`                                                                                                                                                                                                         |

`moveSingleAccountChannelSectionToDefaultAccount(...)` gibi yapılandırma yaması yardımcılarını da içeren paylaşımlı kurulum araç kutusunun tamamını istediğinizde daha geniş kapsamlı `plugin-sdk/setup` bağlantısını kullanın.

Sabit kurulum sihirbazı metni için `createSetupTranslator(...)` kullanın. Sırasıyla `OPENCLAW_LOCALE`, `LC_ALL`, `LC_MESSAGES` ve `LANG` içindeki ilk boş olmayan değeri kullanır, ardından İngilizceye geri döner. Açık bir İngilizce geçersiz kılma değeri için `OPENCLAW_LOCALE=en` ayarlayın. Plugin'e özgü kurulum metnini Plugin'e ait kodda tutun ve paylaşımlı katalog anahtarlarını yalnızca ortak kurulum etiketleri, durum metni ve resmi paketle birlikte gelen Plugin kurulum metni için kullanın.

Kurulum yaması bağdaştırıcıları içe aktarma sırasında sıcak yol açısından güvenli kalır. Paketle birlikte gelen tek hesap yükseltme sözleşme yüzeyi araması tembel yürütülür; bu nedenle `plugin-sdk/setup-runtime` öğesini içe aktarmak, bağdaştırıcı gerçekten kullanılmadan önce paketle birlikte gelen sözleşme yüzeyi keşfini istekli biçimde yüklemez.

### Kanalın sahip olduğu kurulum giriş alanları

`ChannelSetupInput`, kurulum çağıranları ve kanal
Plugin'leri tarafından paylaşılan genel bir zarftır. Kalıcı olarak türü belirtilmiş alanları `name`, `token`, `tokenFile`,
`useEnv`, `allowFrom` ve `defaultTo` öğeleridir. Plugin'e ait ek anahtarlar çalışma zamanı giriş nesnesinde yine
bulunabilir, ancak paylaşılan tür bir dizin imzası bildirmez.
Her Plugin kendi kurulum alanlarını bildirmeli ve daraltmalı veya
bağdaştırıcı sınırında Plugin'e ait bir şemayla doğrulamalıdır:

```typescript
import type { ChannelSetupAdapter, ChannelSetupInput } from "openclaw/plugin-sdk/channel-setup";

type AcmeSetupInput = ChannelSetupInput & {
  workspaceId?: string;
  webhookUrl?: string;
};

export const acmeSetupAdapter: ChannelSetupAdapter = {
  applyAccountConfig: ({ cfg, input }) => {
    const setupInput = input as AcmeSetupInput;
    return {
      ...cfg,
      channels: {
        ...cfg.channels,
        acme: {
          token: setupInput.token,
          workspaceId: setupInput.workspaceId,
          webhookUrl: setupInput.webhookUrl,
        },
      },
    };
  },
};
```

Daha önce doğrudan
`ChannelSetupInput` üzerinde bildirilen kanala özgü alanlar, harici kaynak uyumluluğu için geçici olarak türlendirilmiş durumda kalır.
Bunlar kullanımdan kaldırılmıştır. Ağaç dışı yayımlanmış 426
kanal Plugin'ini kapsayan 2026-07-22 tarihli kayıt defteri taraması, okuyucusu olmayan 21 alanı kaldırdı ve bilinen
okuyucuları olan 22 alanı korudu. Korunan her alan, yayımlanmış hiçbir Plugin artık onu okumadığı anda silinir;
sürüm sınırı gerekmez. Yeni ve paketle birlikte gelen Plugin'ler bu
katmana dayanmamalıdır; sahip oldukları alanları yerel olarak bildirmelidir.

### Kanalın sahip olduğu tek hesaplı yapılandırmanın yükseltilmesi

Bir kanal, tek hesaplı üst düzey yapılandırmadan `channels.<id>.accounts.*` yapısına yükseltildiğinde, varsayılan paylaşılan davranış, yükseltilen hesap kapsamındaki değerleri `accounts.default` içine taşır.

Her kanal Plugin'i, kurulum bağdaştırıcısı aracılığıyla bu yükseltmeyi genişletebilir veya daraltabilir:

- `singleAccountKeysToMove`: yükseltilen hesaba taşınması gereken ek üst düzey anahtarlar
- `namedAccountPromotionKeys`: adlandırılmış hesaplar zaten mevcutsa yalnızca bu anahtarlar yükseltilen hesaba taşınır; paylaşılan politika/teslimat anahtarları kanal kökünde kalır
- `resolveSingleAccountPromotionTarget(...)`: yükseltilen değerleri hangi mevcut hesabın alacağını seçer

`singleAccountKeysToMove` alanının bulunması, yükseltme sözleşmesinin tamamlandığını belirtir. Eski anahtar yükseltmesini devre dışı bırakmak için alanı boş bir dizi olsa bile bildirin. Alanı atlayan bağdaştırıcılar, önceden yayımlanmış Plugin'ler için okuyucu destekli bildirim öncesi yükseltme katmanını korur. 2026-07-22 tarihli kayıt defteri taraması, yayımlanmış bağımlısı olmayan 23 anahtarı kaldırdı ve altı ortak anahtar ile yalnızca kurulumda kullanılan `rooms` anahtarını korudu. Korunan her anahtar, yayımlanmış okuyucuları bildirimlere geçirildiği anda silinir; sürüm sınırı gerekmez.

Doctor'ın bu bildirimleri hafif, paketle birlikte gelen kurulum yapısından yüklemesi gerektiğinde Plugin paket bildiriminde `openclaw.setupFeatures.configPromotion: true` bildirin. Yalnızca kuruluma yönelik Plugin yüzeyi ile tam kanal Plugin'i aynı bildirimleri sunmalıdır.

Önceden çözümlenmiş bir Plugin ile `moveSingleAccountChannelSectionToDefaultAccount(...)` çağrılırken kurulum bağdaştırıcısını `setupSurface` olarak iletin. Çağıranın sağladığı kurulum yüzeyleri, yüklenen ve paketle birlikte gelen aramalara göre önceliklidir; böylece kapsamlı veya yalnızca kuruluma yönelik Plugin'ler genel kayıttan bağımsız kalır.

<Note>
Matrix, paketle birlikte gelen güncel örnektir. Tam olarak bir adlandırılmış Matrix hesabı zaten mevcutsa veya `defaultAccount`, `Ops` gibi standart olmayan mevcut bir anahtarı gösteriyorsa yükseltme, yeni bir `accounts.default` girdisi oluşturmak yerine bu hesabı korur.
</Note>

## Yapılandırma şeması

Plugin yapılandırması, bildirim dosyanızdaki JSON Schema'ya göre doğrulanır. Kullanıcılar Plugin'leri şu şekilde yapılandırır:

```json5
{
  plugins: {
    entries: {
      "my-plugin": {
        config: {
          webhookSecret: "abc123",
        },
      },
    },
  },
}
```

Plugin'iniz kayıt sırasında bu yapılandırmayı `api.pluginConfig` olarak alır.

Kanala özgü yapılandırma için bunun yerine kanal yapılandırması bölümünü kullanın:

```json5
{
  channels: {
    "my-channel": {
      token: "bot-token",
      allowFrom: ["user1", "user2"],
    },
  },
}
```

### Kanal yapılandırma şemaları oluşturma

Bir Zod şemasını Plugin'in sahip olduğu yapılandırma yapılarında kullanılan `ChannelConfigSchema` sarmalayıcısına dönüştürmek için `buildChannelConfigSchema` kullanın:

```typescript
import { z } from "zod";
import { buildChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const accountSchema = z.object({
  token: z.string().optional(),
  allowFrom: z.array(z.string()).optional(),
  accounts: z.object({}).catchall(z.any()).optional(),
  defaultAccount: z.string().optional(),
});

const configSchema = buildChannelConfigSchema(accountSchema);
```

Sözleşmeyi zaten JSON Schema veya TypeBox biçiminde yazıyorsanız OpenClaw'ın meta veri yollarında Zod'dan JSON Schema'ya dönüştürmeyi atlayabilmesi için doğrudan yardımcıyı kullanın:

```typescript
import { Type } from "typebox";
import { buildJsonChannelConfigSchema } from "openclaw/plugin-sdk/channel-config-schema";

const configSchema = buildJsonChannelConfigSchema(
  Type.Object({
    token: Type.Optional(Type.String()),
    allowFrom: Type.Optional(Type.Array(Type.String())),
  }),
);
```

Üçüncü taraf Plugin'ler için soğuk yol sözleşmesi yine Plugin bildirimidir: yapılandırma şeması, kurulum ve kullanıcı arayüzü yüzeylerinin çalışma zamanı kodunu yüklemeden `channels.<id>` öğesini inceleyebilmesi için oluşturulan JSON Schema'yı `openclaw.plugin.json#channelConfigs` içine yansıtın.

## Kurulum sihirbazları

Kanal Plugin'leri, `openclaw onboard` için etkileşimli kurulum sihirbazları sağlayabilir. Sihirbaz, `ChannelPlugin` üzerindeki bir `ChannelSetupWizard` nesnesidir:

```typescript
import type { ChannelSetupWizard } from "openclaw/plugin-sdk/channel-setup";

const setupWizard: ChannelSetupWizard = {
  channel: "my-channel",
  status: {
    configuredLabel: "Connected",
    unconfiguredLabel: "Not configured",
    resolveConfigured: ({ cfg }) => Boolean((cfg.channels as any)?.["my-channel"]?.token),
  },
  credentials: [
    {
      inputKey: "token",
      providerHint: "my-channel",
      credentialLabel: "Bot token",
      preferredEnvVar: "MY_CHANNEL_BOT_TOKEN",
      envPrompt: "Use MY_CHANNEL_BOT_TOKEN from environment?",
      keepPrompt: "Keep current token?",
      inputPrompt: "Enter your bot token:",
      inspect: ({ cfg, accountId }) => {
        const token = (cfg.channels as any)?.["my-channel"]?.token;
        return {
          accountConfigured: Boolean(token),
          hasConfiguredValue: Boolean(token),
        };
      },
    },
  ],
};
```

`ChannelSetupWizard`; `textInputs`, `dmPolicy`, `allowFrom`, `groupAccess`, `prepare`, `finalize` ve daha fazlasını da destekler. Paketle birlikte gelen tam bir örnek için Discord Plugin'inin `src/setup-core.ts` öğesine bakın.

<AccordionGroup>
  <Accordion title="Paylaşılan allowFrom istemleri">
    Yalnızca standart `note -> prompt -> parse -> merge -> patch` akışına ihtiyaç duyan DM izin listesi istemleri için `openclaw/plugin-sdk/setup` içindeki paylaşılan kurulum yardımcılarını tercih edin: `createPromptParsedAllowFromForAccount(...)` ve `createTopLevelChannelParsedAllowFromPrompt(...)`.
  </Accordion>
  <Accordion title="Standart kanal kurulum durumu">
    Yalnızca etiketler, puanlar ve isteğe bağlı ek satırlara göre değişen kanal kurulum durumu blokları için her Plugin'de aynı `status` nesnesini elle oluşturmak yerine `openclaw/plugin-sdk/setup` içindeki `createStandardChannelSetupStatus(...)` öğesini tercih edin.
  </Accordion>
  <Accordion title="İsteğe bağlı kanal kurulum yüzeyi">
    Yalnızca belirli bağlamlarda görünmesi gereken isteğe bağlı kurulum yüzeyleri için `openclaw/plugin-sdk/channel-setup` içindeki `createOptionalChannelSetupSurface` öğesini kullanın:

    ```typescript
    import { createOptionalChannelSetupSurface } from "openclaw/plugin-sdk/channel-setup";

    const setupSurface = createOptionalChannelSetupSurface({
      channel: "my-channel",
      label: "My Channel",
      npmSpec: "@myorg/openclaw-my-channel",
      docsPath: "/channels/my-channel",
    });
    // Returns { setupAdapter, setupWizard }
    ```

    İsteğe bağlı kurulum yüzeyinin yalnızca bir yarısına ihtiyaç duyduğunuzda `plugin-sdk/channel-setup`, daha düşük seviyeli `createOptionalChannelSetupAdapter(...)` ve `createOptionalChannelSetupWizard(...)` oluşturucularını da sunar.

    Oluşturulan isteğe bağlı bağdaştırıcı/sihirbaz, gerçek yapılandırma yazımlarında kapalı biçimde başarısız olur. `validateInput`, `applyAccountConfig` ve `finalize` genelinde tek bir kurulum gerekli iletisini yeniden kullanır ve `docsPath` ayarlandığında bir dokümantasyon bağlantısı ekler.

  </Accordion>
  <Accordion title="İkili dosya destekli kurulum yardımcıları">
    İkili dosya destekli kurulum kullanıcı arayüzleri için aynı ikili dosya/durum bağlantı kodunu her kanala kopyalamak yerine paylaşılan temsilci yardımcılarını tercih edin:

    - `createDetectedBinaryStatus(...)`: yalnızca etiketler, ipuçları, puanlar ve ikili dosya algılamasına göre değişen durum blokları için
    - `createCliPathTextInput(...)`: yol destekli metin girişleri için
    - `createDelegatedSetupWizardProxy(...)`: `setupEntry` durum, hazırlama veya sonlandırma davranışını daha ağır bir tam sihirbaza tembel olarak iletmek zorunda olduğunda
    - `createDelegatedTextInputShouldPrompt(...)`: `setupEntry` yalnızca bir `textInputs[*].shouldPrompt` kararını temsilciye aktarmak zorunda olduğunda

  </Accordion>
</AccordionGroup>

## Yayımlama ve yükleme

**Harici Plugin'ler:** [ClawHub](/clawhub) üzerinde yayımlayın, ardından yükleyin:

<Tabs>
  <Tab title="npm">
    ```bash
    openclaw plugins install @myorg/openclaw-my-plugin
    ```

    Yalın paket belirtimleri, ad paketle birlikte gelen veya resmî bir Plugin kimliğiyle eşleşmediği sürece başlatma geçişi sırasında npm'den yüklenir; eşleşirse OpenClaw bunun yerine ilgili yerel/resmî kopyayı kullanır. Belirlenimci kaynak seçimi için `clawhub:`, `npm:`, `git:` veya `npm-pack:` kullanın — bkz. [Plugin'leri yönetme](/tr/plugins/manage-plugins).

  </Tab>
  <Tab title="Yalnızca ClawHub">
    ```bash
    openclaw plugins install clawhub:@myorg/openclaw-my-plugin
    ```
  </Tab>
  <Tab title="npm paket belirtimi">
    Bir paket henüz ClawHub'a taşınmadığında veya geçiş sırasında doğrudan
    npm yükleme yoluna ihtiyaç duyduğunuzda npm kullanın:

    ```bash
    openclaw plugins install npm:@myorg/openclaw-my-plugin
    ```

  </Tab>
</Tabs>

**Depo içi Plugin'ler:** paketle birlikte gelen Plugin çalışma alanı ağacının altına yerleştirin; derleme sırasında otomatik olarak keşfedilirler.

<Info>
npm kaynaklı yüklemelerde `openclaw plugins install`, paketi yaşam döngüsü betikleri devre dışı bırakılmış (`--ignore-scripts`) şekilde `~/.openclaw/npm/projects` altında Plugin başına bir projeye yükler. Plugin bağımlılık ağaçlarını saf JS/TS olarak tutun ve `postinstall` derlemeleri gerektiren paketlerden kaçının.
</Info>

<Note>
Gateway başlangıcı Plugin bağımlılıklarını yüklemez. npm/git/ClawHub yükleme akışları bağımlılık yakınsamasının sahibidir; yerel Plugin'lerin bağımlılıkları önceden yüklenmiş olmalıdır.
</Note>

Paketle birlikte gelen paket meta verileri açıkça belirtilir; Gateway başlangıcında derlenmiş JavaScript'ten çıkarılmaz. Çalışma zamanı bağımlılıkları, bunların sahibi olan Plugin paketine aittir; paketlenmiş OpenClaw başlangıcı Plugin bağımlılıklarını hiçbir zaman onarmaz veya yansıtmaz.

## İlgili

- [Plugin oluşturma](/tr/plugins/building-plugins) — adım adım başlangıç kılavuzu
- [Plugin bildirimi](/tr/plugins/manifest) — tam bildirim şeması başvurusu
- [SDK giriş noktaları](/tr/plugins/sdk-entrypoints) — `definePluginEntry` ve `defineChannelPluginEntry`
