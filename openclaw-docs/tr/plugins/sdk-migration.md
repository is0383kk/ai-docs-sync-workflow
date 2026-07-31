---
read_when:
    - OPENCLAW_PLUGIN_SDK_COMPAT_DEPRECATED uyarısını görüyorsunuz
    - OPENCLAW_EXTENSION_API_DEPRECATED uyarısını görüyorsunuz
    - OpenClaw 2026.4.25'ten önce api.registerEmbeddedExtensionFactory kullandınız
    - Bir plugin’i modern plugin mimarisine güncelliyorsunuz
    - Harici bir OpenClaw pluginini sürdürüyorsunuz
sidebarTitle: Migrate to SDK
summary: Eski geriye dönük uyumluluk katmanından modern plugin SDK'sına geçiş yapın
title: Plugin SDK geçişi
x-i18n:
    generated_at: "2026-07-26T22:57:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a483f9c0f8409505fc2688872995382944e002520ceb651214dbc5ad8e3554fb
    source_path: plugins/sdk-migration.md
    workflow: 16
---

OpenClaw, geniş bir geriye dönük uyumluluk katmanını küçük ve odaklı içe aktarımlardan oluşturulan modern bir plugin
mimarisiyle değiştirdi. Plugin'iniz bu
değişiklikten önce oluşturulduysa bu kılavuz, onu güncel sözleşmelere geçirmenizi sağlar.

## Neler değişti

Önceden son derece geniş olan çeşitli içe aktarım yüzeyleri, plugin'lerin tek bir
giriş noktasından neredeyse her şeye erişmesine olanak tanıyordu:

- **`openclaw/plugin-sdk`** ve **`openclaw/plugin-sdk/compat`** - odaklı SDK oluşturulurken
  düzinelerce yardımcıyı yeniden dışa aktarıyordu. Her iki kök de artık
  kaldırıldı; bunun yerine belgelenmiş bir alt yolu içe aktarın.
- **`openclaw/plugin-sdk/infra-runtime`** - sistem
  olaylarını, heartbeat durumunu, teslimat kuyruklarını, fetch/proxy yardımcılarını, dosya yardımcılarını,
  onay türlerini ve ilgisiz yardımcı programları bir araya getiren geniş kapsamlı bir barrel.
- **`openclaw/plugin-sdk/config-runtime`** - yalnızca sonraki uyumluluk
  dönemi için tutulan geniş kapsamlı bir yapılandırma barrel'ı; doğrudan çalışma zamanı yükleme/yazma yardımcıları
  kaldırıldı.
- **`openclaw/extension-api`** - plugin'lere gömülü agent çalıştırıcısı gibi ana makine
  tarafındaki yardımcılara doğrudan erişim sağlayan, kaldırılmış bir köprü.
- **`api.registerEmbeddedExtensionFactory(...)`** - `tool_result` gibi gömülü çalıştırıcı olaylarını
  gözlemleyen, yalnızca gömülü çalıştırıcıya yönelik kaldırılmış bir hook. Bunun yerine agent
  araç sonucu ara yazılımını kullanın (bkz. [Gömülü araç sonucu uzantılarını
  ara yazılıma taşıma](#how-to-migrate)).

Kök SDK, uyumluluk barrel'ı, uzantı köprüsü ve gömülü uzantı fabrikası
kaldırıldı. `infra-runtime` ve `config-runtime` yalnızca ayrı olarak
kaydedilmiş sonraki dönemleri için kalır; yeni plugin'ler odaklı alt yolları kullanmalıdır.

<Warning>
  Kaldırılmış kök, uyumluluk veya uzantı yüzeylerini içe aktaran plugin'ler artık
  yüklenmez. Yükseltmeden önce aşağıdaki eşlemeleri izleyin.
</Warning>

OpenClaw, belgelenmiş plugin davranışını bir alternatif sunduğu
değişiklikle aynı anda kaldırmaz veya yeniden yorumlamaz. Sözleşmeyi bozan değişiklikler önce bir
uyumluluk bağdaştırıcısından, tanılamalardan, belgelerden ve bir kullanımdan kaldırma döneminden geçer. Bu;
SDK içe aktarımları, manifest alanları, kurulum API'leri, hook'lar ve çalışma zamanı
kaydı davranışı için geçerlidir.

### Nedenleri

- **Yavaş başlangıç** - tek bir yardımcının içe aktarılması, ilgisiz düzinelerce modülü yüklüyordu.
- **Döngüsel bağımlılıklar** - geniş kapsamlı yeniden dışa aktarımlar, içe aktarım döngülerinin
  oluşturulmasını kolaylaştırıyordu.
- **Belirsiz API yüzeyi** - kararlı dışa aktarımları dahili olanlardan ayırt etmenin bir yolu yoktu.

Artık her `openclaw/plugin-sdk/<subpath>`, belgelenmiş bir sözleşmeye sahip
küçük ve bağımsız bir modüldür.

Paketle gelen kanallara yönelik eski sağlayıcı kolaylık katmanları da kaldırıldı;
kanala özgü yardımcı kısayollar kararlı plugin sözleşmeleri değil, özel mono-repo kolaylıklarıydı.
Bunun yerine dar kapsamlı genel SDK alt yollarını kullanın. Paketle gelen plugin çalışma alanında,
sağlayıcının sahip olduğu yardımcıları ilgili plugin'in kendi
`api.ts` veya `runtime-api.ts` konumunda tutun:

- Anthropic, Claude'a özgü akış yardımcılarını kendi `api.ts` /
  `contract-api.ts` katmanında tutar.
- OpenAI, sağlayıcı oluşturucularını, varsayılan model yardımcılarını ve gerçek zamanlı sağlayıcı
  oluşturucularını kendi `api.ts` konumunda tutar.
- OpenRouter, sağlayıcı oluşturucusunu ve ilk katılım/yapılandırma yardımcılarını kendi
  `api.ts` konumunda tutar.

## Uyumluluk politikası

Harici plugin uyumluluk çalışmaları şu sırayı izler:

1. Yeni sözleşmeyi ekleyin.
2. Eski davranışı bir uyumluluk bağdaştırıcısı üzerinden bağlı tutun.
3. Eski yolu ve alternatifini belirten bir tanılama veya uyarı yayınlayın.
4. Testlerde her iki yolu da kapsayın.
5. Kullanımdan kaldırma ve geçiş yolunu belgeleyin.
6. Yalnızca duyurulan geçiş döneminden sonra, genellikle büyük bir
   sürümde kaldırın.

Bir manifest alanı hâlâ kabul ediliyorsa belgeler ve
tanılamalar aksini söyleyene kadar kullanmaya devam edin. Yeni kod, belgelenmiş alternatifi tercih etmelidir;
mevcut plugin'ler olağan küçük sürümler sırasında bozulmamalıdır.

### Yayımlanmış kanal kurulumu uyumluluğu

`2026.7.1` üzerinden yayımlanan Slack, Discord, Signal ve Microsoft Teams
paketleri, kanala özgü yapılandırma şemalarını
`openclaw/plugin-sdk/bundled-channel-config-schema` konumundan içe aktarır. Yayımlanmış Slack ve
Discord paketleri ayrıca `openclaw/plugin-sdk/setup-runtime`
konumundan `createLegacyCompatChannelDmPolicy` ve
`promptLegacyChannelAllowFromForAccount` öğelerini içe aktarır.

Bu dışa aktarımlar, kullanımdan kaldırılmış çalışma zamanı uyumluluk bağdaştırıcıları olarak kullanılmaya devam eder.
Yeni ve yeniden yayımlanan plugin'ler, `channel-config-schema` ve
`setup-runtime` konumundaki genel temel öğeleri kullanarak yapılandırma şemalarının ve kurulum politikalarının
sahipliğini yerel olarak üstlenmelidir. Uyumluluk dışa aktarımları yalnızca desteklenen en düşük
yayımlanmış paket sürümleri artık bunları içe aktarmadığında kaldırılabilir.

### Kanal kurulumu giriş alanı uyumluluğu

`ChannelSetupInput` artık yalnızca kanallar arası kurulum zarfını kalıcı olarak
türlü tutar. Kanala özgü alanlar, plugin yazarları bu
alanları plugin'e özgü kurulum giriş türlerine taşırken mevcut harici plugin'lerin derlenmeye devam etmesi için
kullanımdan kaldırılmış bir uyumluluk katmanında türlü kalır.

OpenClaw büyük sürümler yayımlamaz. 2026-07-22 tarihinde yapılan bir kayıt defteri taraması,
ağaç dışındaki 426 yayımlanmış kanal plugin'ini inceledi ve okuyucusu olmayan 21 alanı kaldırdı.
Tutulan 22 alanın her birinin bilinen bir yayımlanmış okuyucusu vardır. Sonraki her alan,
hiçbir yayımlanmış plugin onu okumaz okumaz silinir; plugin yazarları plugin'e özgü
kurulum giriş türlerine geçtikçe tutulan küme küçülür.

Aynı tarama, yayımlanmış bağımlısı olmayan 23 eski bildirilmemiş bağdaştırıcı yükseltme anahtarını kaldırdı.
Altı yaygın anahtar ve yalnızca kuruluma yönelik `rooms` anahtarı kalır.
Yayımlanmış plugin'ler `singleAccountKeysToMove` bildirdikçe bu küme de küçülür.

Paylaşılan türde indeks imzası yoktur. Plugin'in sahip olduğu anahtarlar çalışma zamanı
giriş nesnelerinde bulunmaya devam edebilir; bunları plugin'e özgü bir kesişimde bildirin veya
sahibi olan plugin'in kurulum şeması üzerinden daraltın.

| `code`                                  | `owner`   | `replacement`                                                                                    | Kaldırma koşulu                                                        |
| --------------------------------------- | --------- | ------------------------------------------------------------------------------------------------ | --------------------------------------------------------------------- |
| `plugin-sdk-channel-setup-input-fields` | `channel` | `ChannelSetupInput` öğesini, sahibi olan kanalın alanlarını bildiren plugin'e özgü bir türle kesiştirin | Yayımlanmış plugin kayıt defteri taramasında okuyucu kalmadığında alanı silin |

Eski bildirilmemiş bağdaştırıcı yükseltme katmanı, okuyucuya dayalı aynı
politikayı izler. Plugin'in ek yükseltme anahtarına ihtiyaç duymadığı durumlarda boş bir dizi de dahil olmak üzere
`singleAccountKeysToMove` öğesini bildirin; böylece paylaşılan geri dönüş mekanizması her seferinde bir
anahtar olacak şekilde kullanımdan kaldırılabilir.

#### Okuyucuları doğrulama

1. Her `nextCursor` ile `https://clawhub.ai/api/v1/packages?family=code-plugin&limit=100` içinde sayfalar arasında ilerleyin ve `categories` alanında `channels` bulunan paketleri tutun.
2. `npm search --json --searchlimit=1000 "openclaw channel plugin"` içindeki npm adaylarını ekleyin. `openclaw/plugin-sdk/channel-setup`, `openclaw/plugin-sdk/setup` ve `openclaw/plugin-sdk/core` için GitHub kod aramalarından yalnızca kaynak kodu bulunan adayları ekleyin.
3. Her adayın yayımlanmış en son sürümünü çözümleyin. `npm pack <package>@<version> --json --pack-destination <temp-dir>` komutunu çalıştırın, paketi açın ve doğrudan veya yapı çözümlemeli alan okumaları için gönderilmiş `dist` JavaScript kodunu ve bildirimlerini inceleyin. Bir paketin npm sürümü yoksa ClawHub yapıtını indirin.
4. Paketi, sürümü, alanı veya yükseltme anahtarını ve eşleşen dosyayı kaydedin. Bir alan veya anahtar yalnızca hiçbir yayımlanmış plugin yapıtı onu okumadığında silinebilir. Tutulan alan ve anahtar listelerinin yanındaki kod açıklamalarında bulunan okuyucu adlarını taramayla eşitlenmiş halde tutun.

Bu yalnızca bir kaynak/tür uyumluluk kaydıdır. Çalışma zamanı kurulum giriş nesneleri ve kurulum
davranışı değişmediğinden çalışma zamanı bağdaştırıcısı veya uyumluluk kayıt defteri
girdisi yoktur.

Geçerli geçiş kuyruğunu `pnpm plugins:boundary-report` ile denetleyin:

| Bayrak                                                   | Etki                                                                           |
| ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `--summary` (veya `pnpm plugins:boundary-report:summary`) | Tam ayrıntılar yerine özet sayımlar.                                           |
| `--json`                                                | Makine tarafından okunabilir rapor.                                            |
| `--owner <id>`                                          | Tek bir plugin'e veya uyumluluk sahibine göre filtreleme.                      |
| `--fail-on-cross-owner`                                 | Sahipler arası ayrılmış SDK içe aktarımlarında sıfır dışında bir değerle çıkış. |
| `--fail-on-eligible-compat`                             | Kullanımdan kaldırılmış bir uyumluluk kaydının `removeAfter` tarihi geçtiğinde sıfır dışında bir değerle çıkış. |
| `--fail-on-unclassified-unused-reserved`                | Kullanılmayan ayrılmış SDK shim'lerinde sıfır dışında bir değerle çıkış.        |

`pnpm plugins:boundary-report:ci`, üç hata bayrağının tümüyle çalışır. Kullanımdan kaldırılmış
kayıtlarda genellikle belirsiz bir "sonraki büyük sürüm" ifadesi yerine açık bir `removeAfter` tarihi bulunur.
Sahibi tarafından tarihi onaylanmamış bir kayıtta
`removeAfter` bulunmaz, `no-date` olarak görünür ve hiçbir zaman kaldırılmaya uygun olmaz.
Rapor, kullanımdan kaldırılmış kayıtları tarihe göre gruplandırır, yerel kod/belge referanslarını sayar,
sahipler arası ayrılmış SDK içe aktarımlarını gösterir ve özel
bellek ana makinesi SDK köprüsünü özetler. Ayrılmış SDK alt yollarının izlenen sahip kullanımları olmalıdır;
kullanılmayan ayrılmış dışa aktarımlar genel SDK'dan kaldırılmalıdır.

### Eski medya projeksiyonu

`media-legacy-projection` uyumluluk kaydı; eski paralel
medya alanlarını, yük oluşturucularını, hook meta verisi takma adlarını ve medya şablonu
adlarını kapsar. Onaylanmış `removeAfter` tarihi **2026-10-01**'dir (önce olgular yaklaşımına dayalı alternatiflerin
gönderilmesinden iki sürüm treni sonrası). Kaldırma işlemi ayrıca o tarihte
yayımlanmış plugin yapıtlarının temiz bir şekilde taranmasını gerektirir; bu tarihten önce geçiş yapın.

Kanal girişi için tekil/çoğul `MediaPath`, `MediaUrl`,
`MediaType`, `MediaPaths`, `MediaUrls`, `MediaTypes`,
`MediaTranscribedIndexes`, `MediaWorkspaceDir` ve `MediaStaged` öğelerini sıralı
olgularla değiştirin:

```ts
import { toInboundMediaFacts } from "openclaw/plugin-sdk/channel-inbound";

const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

`inbound_claim` ve `message_received` hook'larında `event.media` kullanın. Uzak
medya yerel olarak hazırlanmadıysa kimlik/tanılama için `event.originalMedia` kullanın
ve `event.media` öğesini bekleyin; `event.mediaStagingPending` bu
durumu ayırt eder. `event.metadata` öğesinden kullanımdan kaldırılmış tekil/çoğul özellikleri
okumayın.

CLI medya modelleri için `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}`
ve `{{MediaDir}}` öğelerini `{{AttachmentPath}}`, `{{AttachmentUrl}}`,
`{{AttachmentContentType}}` ve `{{AttachmentDir}}` ile değiştirin. Ek konumunun
önemli olduğu durumlarda `{{AttachmentIndex}}` kullanın.

Yerel medya okuma politikası için
`openclaw/plugin-sdk/media-local-roots` konumundan `getAgentScopedMediaLocalRoots(...)` veya
`getAgentScopedMediaLocalRootsForSources(...)` öğesini içe aktarın.
`openclaw/plugin-sdk/agent-media-payload` facade'ı ve onun
`buildAgentMediaPayload(...)` projeksiyonu kullanımdan kaldırılmıştır.

## Geçiş yapma

<Steps>
  <Step title="Çalışma zamanı yapılandırması yükleme/yazma yardımcılarını taşıma">
    Paketle gelen plugin'ler `api.runtime.config.loadConfig()` ve
    `api.runtime.config.writeConfigFile(...)` öğelerini doğrudan çağırmayı bırakmalıdır. Etkin çağrı yoluna
    zaten aktarılmış olan yapılandırmayı tercih edin. Geçerli işlem anlık görüntüsüne ihtiyaç duyan
    uzun ömürlü işleyiciler `api.runtime.config.current()` kullanabilir. Uzun ömürlü
    agent araçları, bir yapılandırma yazma işleminden önce oluşturulan aracın yenilenmiş yapılandırmayı
    görmeye devam etmesi için `execute` içinde `ctx.getRuntimeConfig()` öğesini okumalıdır.

    Yapılandırma yazma işlemleri, açık bir yazma sonrası politikasıyla işlemsel
    yardımcı üzerinden gerçekleştirilir:

    ```typescript
    await api.runtime.config.mutateConfigFile({
      afterWrite: { mode: "auto" },
      mutate(draft) {
        draft.plugins ??= {};
      },
    });
    ```

    Değişiklik temiz bir Gateway yeniden başlatması gerektirdiğinde `afterWrite: { mode: "restart", reason: "..." }` kullanın; `afterWrite: { mode: "none", reason: "..." }` ise
    yalnızca çağıran taraf sonraki adımın sorumluluğunu üstlendiğinde ve yeniden
    yükleme planlayıcısını kasıtlı olarak devre dışı bıraktığında kullanılmalıdır. Mutasyon sonuçları, testler
    ve günlük kaydı için türü belirlenmiş bir `followUp` özeti içerir; yeniden başlatmayı uygulama
    veya zamanlama sorumluluğu Gateway'de kalır.

    `loadConfig` ve `writeConfigFile`, Plugin
    çalışma zamanından kaldırılmıştır. Paketlenmiş Plugin'ler ve depo çalışma zamanı kodu,
    `pnpm check:deprecated-api-usage` ve
    `pnpm check:no-runtime-action-load-config` tarafından korunur: üretimde yeni Plugin kullanımı
    doğrudan başarısız olur, doğrudan yapılandırma yazımları başarısız olur, Gateway sunucu yöntemleri
    istek çalışma zamanı anlık görüntüsünü kullanmalıdır, çalışma zamanı kanal gönderme/eylem/istemci yardımcıları
    yapılandırmayı kendi sınırlarından almalıdır ve uzun ömürlü çalışma zamanı modülleri
    sıfır ortam `loadConfig()` çağrısına izin verir.

    Yeni Plugin kodu, geniş `openclaw/plugin-sdk/config-runtime`
    toplu dışa aktarımından kaçınmalıdır. İş için dar alt yolu kullanın:

    | Gereksinim | İçe aktarma |
    | --- | --- |
    | `OpenClawConfig` gibi yapılandırma türleri | `openclaw/plugin-sdk/config-contracts` |
    | Plugin giriş noktası yapılandırma araması | `api.pluginConfig` |
    | Yapılandırma birleştirme | Yapılandırma sınırındaki Plugin'e özgü mantık |
    | Geçerli çalışma zamanı anlık görüntüsü okumaları | `openclaw/plugin-sdk/runtime-config-snapshot` |
    | Yapılandırma yazımları | `openclaw/plugin-sdk/config-mutation` |
    | Oturum deposu yardımcıları | `openclaw/plugin-sdk/session-store-runtime` |
    | Markdown tablo yapılandırması | `openclaw/plugin-sdk/markdown-table-runtime` |
    | Grup ilkesi çalışma zamanı yardımcıları | `openclaw/plugin-sdk/runtime-group-policy` |
    | Gizli bilgi girdisi çözümleme | `openclaw/plugin-sdk/secret-input-runtime` |
    | Model/oturum geçersiz kılmaları | `openclaw/plugin-sdk/model-session-runtime` |

    Paketlenmiş Plugin'ler ve testleri, ihtiyaç duydukları davranışa yönelik içe aktarma
    ve taklitlerin yerel kalması için tarayıcı tarafından geniş toplu dışa aktarıma karşı korunur.
    Toplu dışa aktarım harici uyumluluk için hâlâ mevcuttur, ancak yeni kod
    buna bağımlı olmamalıdır.

  </Step>

  <Step title="Gömülü araç sonucu uzantılarını ara yazılıma geçirin">
    Paketlenmiş Plugin'ler, yalnızca gömülü çalıştırıcıya özgü
    `api.registerEmbeddedExtensionFactory(...)` araç sonucu işleyicilerini
    çalışma zamanından bağımsız ara yazılımla değiştirmelidir:

    ```typescript
    // OpenClaw çalışma zamanı araçları ve Codex çalışma zamanı dinamik araçları (sonuç
    // dönüştürülebilir). Codex'e özgü araç sonuçları da gözlem amacıyla aktarılır,
    // ancak dönüştürülmüş çıktıları modele hiçbir zaman ulaşmaz: Codex
    // PostToolUse kancası sözleşmesi yerel bir araç yanıtının yerini alamaz.
    api.registerAgentToolResultMiddleware(async (event) => {
      return compactToolResult(event);
    }, {
      runtimes: ["openclaw", "codex"],
    });
    ```

    Plugin manifestini aynı anda güncelleyin:

    ```json
    {
      "contracts": {
        "agentToolResultMiddleware": ["openclaw", "codex"]
      }
    }
    ```

    Kurulu Plugin'ler de açıkça etkinleştirildiklerinde ve hedeflenen her çalışma zamanı
    `contracts.agentToolResultMiddleware` içinde bildirildiğinde araç sonucu ara yazılımı kaydedebilir.
    Bildirilmemiş kurulu ara yazılım kayıtları reddedilir.

  </Step>

  <Step title="Onaya özgü işleyicileri yetenek olgularına geçirin">
    Onay özellikli kanal Plugin'leri, yerel onay davranışını
    `approvalCapability.nativeRuntime` ve paylaşılan çalışma zamanı bağlamı
    kayıt defteri aracılığıyla sunar:

    - `approvalCapability.handler.loadRuntime(...)` yerine
      `approvalCapability.nativeRuntime` kullanın.
    - Onaya özgü kimlik doğrulama/teslimatı eski `plugin.auth` /
      `plugin.approvals` bağlantısından `approvalCapability` üzerine taşıyın.
    - `ChannelPlugin.approvals`, genel kanal Plugin'i
      sözleşmesinden kaldırılmıştır; teslimat/yerel/işleme alanlarını
      `approvalCapability` üzerine taşıyın.
    - `plugin.auth` yalnızca kanal oturum açma/kapatma akışları için kalır; çekirdek
      artık buradaki onay kimlik doğrulama kancalarını okumaz.
    - Kanalın sahip olduğu çalışma zamanı nesnelerini (istemciler, belirteçler, Bolt uygulamaları)
      `openclaw/plugin-sdk/channel-runtime-context` aracılığıyla kaydedin.
    - Yerel onay işleyicilerinden Plugin'e ait yeniden yönlendirme bildirimleri göndermeyin;
      gerçek teslimat sonuçlarından gelen başka yere yönlendirilmiş bildirimlerin sahibi çekirdektir.
    - `channelRuntime` değerini `createChannelManager(...)` içine geçirirken gerçek bir
      `createPluginRuntime().channel` yüzeyi sağlayın; kısmi taslaklar
      reddedilir.

    Geçerli onay yeteneği düzeni için [Kanal Plugin'leri](/tr/plugins/sdk-channel-plugins) bölümüne bakın.

  </Step>

  <Step title="Windows sarmalayıcı geri dönüş davranışını denetleyin">
    Plugin'iniz `openclaw/plugin-sdk/windows-spawn` kullanıyorsa çözümlenmemiş Windows
    `.cmd`/`.bat` sarmalayıcıları, açıkça
    `allowShellFallback: true` geçirmediğiniz sürece artık güvenli biçimde başarısız olur:

    ```typescript
    // Önce
    const program = applyWindowsSpawnProgramPolicy({ candidate });

    // Sonra
    const program = applyWindowsSpawnProgramPolicy({
      candidate,
      // Bunu yalnızca kabuk aracılı geri dönüşü kasıtlı olarak kabul eden
      // güvenilir uyumluluk çağıranları için ayarlayın.
      allowShellFallback: true,
    });
    ```

    Çağıran tarafınız kabuk geri dönüşüne kasıtlı olarak bağımlı değilse
    `allowShellFallback` ayarını yapmayın ve bunun yerine fırlatılan hatayı işleyin.

  </Step>

  <Step title="Kullanımdan kaldırılmış içe aktarmaları bulun">
    ```bash
    grep -r "plugin-sdk/compat" my-plugin/
    grep -r "plugin-sdk/infra-runtime" my-plugin/
    grep -r "plugin-sdk/config-runtime" my-plugin/
    grep -r "openclaw/extension-api" my-plugin/
    ```
  </Step>

  <Step title="Odaklı içe aktarmalarla değiştirin">
    Eski yüzeydeki her dışa aktarım belirli bir modern içe aktarma yoluyla eşleşir:

    ```typescript
    // Önce (kullanımdan kaldırılmış geriye dönük uyumluluk katmanı)
    import {
      createChannelReplyPipeline,
      createPluginRuntimeStore,
      resolveControlCommandGate,
    } from "openclaw/plugin-sdk/compat";

    // Sonra (modern odaklı içe aktarmalar)
    import { createChannelReplyPipeline } from "openclaw/plugin-sdk/channel-reply-pipeline";
    import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
    import { resolveControlCommandGate } from "openclaw/plugin-sdk/command-auth";
    ```

    Ana makine tarafındaki yardımcılar için doğrudan içe aktarma yerine
    eklenen Plugin çalışma zamanını kullanın:

    ```typescript
    // Önce (kullanımdan kaldırılmış extension-api köprüsü)
    import { runEmbeddedAgent } from "openclaw/extension-api";
    const result = await runEmbeddedAgent({ sessionId, prompt });

    // Sonra (eklenen çalışma zamanı)
    const result = await api.runtime.agent.runEmbeddedAgent({ sessionId, prompt });
    ```

    Diğer eski köprü yardımcıları için de aynı kalıbı kullanın:

    | Eski içe aktarma | Modern eşdeğer |
    | --- | --- |
    | `resolveAgentDir` | `api.runtime.agent.resolveAgentDir` |
    | `resolveAgentWorkspaceDir` | `api.runtime.agent.resolveAgentWorkspaceDir` |
    | `resolveAgentIdentity` | `api.runtime.agent.resolveAgentIdentity` |
    | `resolveThinkingDefault` | `api.runtime.agent.resolveThinkingDefault` |
    | `resolveAgentTimeoutMs` | `api.runtime.agent.resolveAgentTimeoutMs` |
    | `ensureAgentWorkspace` | `api.runtime.agent.ensureAgentWorkspace` |
    | oturum deposu yardımcıları | `api.runtime.agent.session.*` |

  </Step>

  <Step title="Geniş infra-runtime içe aktarmalarını değiştirin">
    `openclaw/plugin-sdk/infra-runtime` harici
    uyumluluk için hâlâ mevcuttur, ancak yeni kod gerçekten ihtiyaç duyduğu odaklı yüzeyi
    içe aktarmalıdır:

    | Gereksinim | İçe aktarma |
    | --- | --- |
    | Sistem olayı kuyruğu yardımcıları | `openclaw/plugin-sdk/system-event-runtime` |
    | Heartbeat uyandırma, olay ve görünürlük yardımcıları | `openclaw/plugin-sdk/heartbeat-runtime` |
    | Bekleyen teslimat kuyruğunu boşaltma | `openclaw/plugin-sdk/delivery-queue-runtime` |
    | Kanal etkinliği telemetrisi | `openclaw/plugin-sdk/channel-activity-runtime` |
    | Bellek içi ve kalıcı depolama destekli tekilleştirme önbellekleri | `openclaw/plugin-sdk/dedupe-runtime` |
    | Güvenli yerel dosya/medya yolu yardımcıları | `openclaw/plugin-sdk/file-access-runtime` |
    | Dağıtıcı duyarlı getirme | `openclaw/plugin-sdk/runtime-fetch` |
    | Vekil sunucu ve korumalı getirme yardımcıları | `openclaw/plugin-sdk/fetch-runtime` |
    | SSRF dağıtıcı ilkesi türleri | `openclaw/plugin-sdk/ssrf-dispatcher` |
    | Onay isteği/çözümleme türleri | `openclaw/plugin-sdk/approval-runtime` |
    | Onay yanıtı yükü ve komut yardımcıları | `openclaw/plugin-sdk/approval-reply-runtime` |
    | Hata biçimlendirme yardımcıları | `openclaw/plugin-sdk/error-runtime` |
    | Aktarımın hazır olmasını bekleme | `openclaw/plugin-sdk/transport-ready-runtime` |
    | Güvenli belirteç yardımcıları | `openclaw/plugin-sdk/secure-random-runtime` |
    | Sınırlı eşzamansız görev eşzamanlılığı | `openclaw/plugin-sdk/concurrency-runtime` |
    | Kanıtlanabilir değişmezler için gerekli değer doğrulamaları | `openclaw/plugin-sdk/expect-runtime` |
    | Sayısal dönüştürme | `openclaw/plugin-sdk/number-runtime` |
    | İşlem içi eşzamansız kilit | `openclaw/plugin-sdk/async-lock-runtime` |
    | Dosya kilitleri | `openclaw/plugin-sdk/file-lock` |

    Paketlenmiş Plugin'ler `infra-runtime` kullanımına karşı tarayıcı tarafından korunur; böylece depo kodu
    geniş toplu dışa aktarıma geri dönemez.

  </Step>

  <Step title="Kanal rota yardımcılarını geçirin">
    Yeni kanal rota kodu `openclaw/plugin-sdk/channel-route` kullanır. Eski
    rota anahtarı adları uyumluluk takma adları olarak kalır:

    | Eski yardımcı | Modern yardımcı |
    | --- | --- |
    | `channelRouteIdentityKey(...)` | `channelRouteDedupeKey(...)` |
    | `channelRouteKey(...)` | `channelRouteCompactKey(...)` |

    Modern rota yardımcıları `{ channel, to, accountId, threadId }` değerini
    yerel onaylar, yanıt engelleme, gelen ileti tekilleştirme,
    cron teslimatı ve oturum yönlendirme genelinde tutarlı biçimde normalleştirir.

    `plugin-sdk/channel-route` üzerinden `ChannelMessagingAdapter.parseExplicitTarget` veya
    `resolveChannelRouteTargetWithParser(...)` için yeni kullanımlar eklemeyin; bunlar kullanımdan kaldırılmıştır ve yalnızca eski
    Plugin'ler için kalır. Yeni kanal Plugin'leri hedef kimliği normalleştirme
    ve dizinde bulunamama geri dönüşü için
    `messaging.targetResolver.resolveTarget(...)`, çekirdek erken bir eş türüne ihtiyaç duyduğunda
    `messaging.inferTargetChatType(...)`, sağlayıcıya özgü
    oturum ve ileti dizisi kimliği için `messaging.resolveOutboundSessionRoute(...)` kullanmalıdır.

  </Step>

  <Step title="Derleyin ve test edin">
    ```bash
    pnpm build
    pnpm test my-plugin/
    ```
  </Step>
</Steps>

## İçe aktarma yolu başvurusu

İçe aktarılabilir SDK alt yolları için doğruluk kaynağı, genel paket dışa aktarma eşlemesidir.
[SDK genel bakışı](/tr/plugins/sdk-overview) bölümünden bağlantı verilen konuya özgü SDK kılavuzlarını kullanın
ve belgelenmiş en dar genel alt yolu tercih edin. `scripts/lib/plugin-sdk-entrypoints.json` içindeki derleyici envanteri,
paketlenmiş Plugin'leri derlemek için kullanılan özel-yerel girdileri de içerir;
bunların burada bulunması onları genel paket dışa aktarımları yapmaz.

Bu tablo, tam SDK yüzeyi değil, yaygın geçiş alt kümesidir.
Derleyici giriş noktası envanteri `scripts/lib/plugin-sdk-entrypoints.json` içindedir;
paket dışa aktarımları genel alt kümeden oluşturulur.

Açıkça belgelenmiş uyumluluk cepheleri dışında, paketlenmiş Plugin'lere ayrılmış yardımcı
yüzeyler genel SDK dışa aktarma eşlemesinden kaldırılmıştır. Buna, yayımlanmış
`@openclaw/discord` paketini hâlâ doğrudan içe aktaran harici Plugin'ler için tutulan,
kullanımdan kaldırılmış `plugin-sdk/discord` uyumluluk katmanı dahildir. Sahibe özgü
yardımcılar, sahibi olan Plugin paketinin içinde bulunur; paylaşılan ana makine davranışı
`plugin-sdk/gateway-runtime`, `plugin-sdk/security-runtime` ve eklenen Plugin API'si gibi
genel SDK sözleşmeleri üzerinden taşınır.

İşle eşleşen en dar içe aktarmayı kullanın. Bir dışa aktarım bulamıyorsanız
`src/plugin-sdk/` kaynağını kontrol edin veya hangi genel
sözleşmenin buna sahip olması gerektiğini bakımcılara sorun.

## Kaldırılan uyumluluk yüzeyleri

Temmuz 2026 taraması; kök SDK ve uyumluluk toplu dışa aktarımlarını, uzantı API'si
köprüsünü, süresi dolmuş SDK alt yol takma adlarını, kullanılmayan SDK alt yollarını ve
yalnızca paketlenmiş modüllere yönelik genel SDK dışa aktarımlarını kaldırdı. Yalnızca
paketlenmiş modüller, özel-yerel derleme eşlemeleri aracılığıyla depo sahiplerinin
kullanımına açık kalır; yayımlanmış paketten içe aktarılamazlar.

### İşlem geneli API sağlayıcısı yayımlama

`registerApiProvider(...)` ve `unregisterApiProviders(...)`,
`openclaw/plugin-sdk/llm` üzerinden kaldırıldı. Bunlar API aktarımlarını işlem geneli
durumda yayımlıyordu; yaşam döngüsünün sahibi olduğu model çalışma zamanları da bunları
hazırlanan her kayıt defterine kopyalamak zorunda kalıyordu.

Sağlayıcı Plugin'leri metin çıkarımı sağlayıcılarını
`api.registerProvider(...)` aracılığıyla kaydetmelidir. Bir
`ApiRegistry` oluşturan ana makineye ait kod ve testler, sağlayıcı sahipliğinin
ve kapatma işleminin hazırlanan çalışma zamanı kapsamında kalması için doğrudan bu kayıt defterine
kaydolmalıdır.

### Özel test toplu dışa aktarımı

`openclaw/plugin-sdk/testing` depoya özeldi ve yayımlanan paket
yapılarının dışında bırakılıyordu; bu nedenle gelecekteki 2026-07-28 `removeAfter` tarihinden önce kaldırıldı. Depo
testleri `plugin-sdk/plugin-test-runtime`, `plugin-sdk/channel-test-helpers`, `plugin-sdk/channel-target-testing`,
`plugin-sdk/test-env` ve `plugin-sdk/test-fixtures` gibi odaklı alt yolları kullanır.

## Geçiş başvurusu

  Bu eşlemeler hem Temmuz 2026'da kaldırılan yüzeyleri hem de daha sonraki dönemde etkin olan
  kullanımdan kaldırmaları kapsar. Bir eşleme, geçiş rehberidir; eski yüzeyin
  hâlâ kullanılabilir olduğunun kanıtı değildir. Güncel durum için uyumluluk kayıt
  defterine ve kaldırma zaman çizelgesine başvurun.

  <AccordionGroup>
  <Accordion title="command-auth yardım oluşturucuları -> command-status">
    **Eski (`openclaw/plugin-sdk/command-auth`)**: `buildCommandsMessage`,
    `buildCommandsMessagePaginated`, `buildHelpMessage`.

    **Yeni (`openclaw/plugin-sdk/command-status`)**: aynı imzalar, daha dar
    alt yoldan içe aktarılır. `command-auth` uyumluluk yeniden dışa aktarımları
    kaldırılmıştır.

    ```typescript
    // Önce
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-auth";

    // Sonra
    import { buildHelpMessage } from "openclaw/plugin-sdk/command-status";
    ```

  </Accordion>

  <Accordion title="Bahsetme geçidi yardımcıları -> resolveInboundMentionDecision">
    **Eski**: `openclaw/plugin-sdk/channel-inbound` veya
    `openclaw/plugin-sdk/channel-mention-gating` içinden
    `resolveMentionGating(params)` ve
    `resolveMentionGatingWithBypass(params)`.

    **Yeni**: `resolveInboundMentionDecision({ facts, policy })` - iki ayrı çağrı biçimi
    yerine tek bir karar nesnesi.

    Discord, iMessage, Matrix, MS Teams, QQBot, Signal,
    Telegram, WhatsApp ve Zalo genelinde benimsenmiştir. Slack'in kendi `app_mention` olay modeli
    bu yardımcıyı kullanmaz.

  </Accordion>

  <Accordion title="Kanal çalışma zamanı uyumluluk katmanı ve kanal eylemi yardımcıları">
    `openclaw/plugin-sdk/channel-runtime` kaldırılmıştır. Çalışma zamanı
    nesnelerini kaydetmek için `openclaw/plugin-sdk/channel-runtime-context` kullanın.

    `openclaw/plugin-sdk/channel-actions` içindeki yerel ileti şeması yardımcıları,
    ham "actions" kanal dışa aktarımlarıyla birlikte kaldırılmıştır. Bunun yerine yetenekleri
    anlamsal `presentation` yüzeyi üzerinden sunun; kanal plugin'leri
    hangi ham eylem adlarını kabul ettiklerini değil, neyi görüntülediklerini
    (kartlar, düğmeler, seçimler) bildirir.

  </Accordion>

  <Accordion title="Web araması sağlayıcısı tool() yardımcısı -> plugin üzerinde createTool()">
    **Eski**: `openclaw/plugin-sdk/provider-web-search` içinden `tool()` fabrikası.

    **Yeni**: `createTool(...)` öğesini doğrudan sağlayıcı plugin'inde uygulayın.
    OpenClaw artık araç sarmalayıcısını kaydetmek için SDK yardımcısına ihtiyaç duymaz.

  </Accordion>

  <Accordion title="Düz metin kanal zarfları -> BodyForAgent">
    **Eski**: gelen kanal iletilerinden düz bir
    metin istem zarfı oluşturmak için `api.runtime.channel.reply.formatInboundEnvelope(...)` (ve gelen ileti
    nesnelerindeki `channelEnvelope` alanı).

    **Yeni**: `BodyForAgent` ve yapılandırılmış kullanıcı bağlamı blokları. Kanal
    plugin'leri yönlendirme meta verilerini (iş parçacığı, konu, yanıt hedefi, tepkiler)
    bir istem dizesinde birleştirmek yerine türü belirlenmiş alanlar olarak ekler.
    `formatAgentEnvelope(...)` yardımcısı, oluşturulan
    asistan odaklı zarflar için desteklenmeye devam eder; ancak gelen düz metin zarfları
    kullanımdan kaldırılma sürecindedir.

    Etkilenen alanlar: `inbound_claim`, `message_received` ve eski zarf
    metnini sonradan işleyen tüm özel kanal plugin'leri.

  </Accordion>

  <Accordion title="deactivate kancası -> gateway_stop">
    **Eski**: `api.on("deactivate", handler)`.

    **Yeni**: `api.on("gateway_stop", handler)`. Aynı kapatma temizliği
    sözleşmesi geçerlidir; yalnızca kancanın adı değişir.

    ```typescript
    // Önce
    api.on("deactivate", async (event, ctx) => {
      await stopPluginService(ctx);
    });

    // Sonra
    api.on("gateway_stop", async (event, ctx) => {
      await stopPluginService(ctx);
    });
    ```

    `deactivate`, 2026-08-16 sonrasında kaldırılana kadar kullanımdan kaldırılmış
    bir uyumluluk diğer adı olarak bağlı kalır.

  </Accordion>

  <Accordion title="subagent_spawning kancası -> çekirdek iş parçacığı bağlama">
    **Eski**: `threadBindingReady` veya `deliveryOrigin`
    döndüren `api.on("subagent_spawning", handler)`.

    **Yeni**: çekirdeğin kanal oturumu bağlama bağdaştırıcısı üzerinden
    `thread: true` alt ajan bağlamalarını hazırlamasına izin verin.
    `api.on("subagent_spawned", handler)` öğesini yalnızca başlatma sonrası gözlem için kullanın.

    ```typescript
    // Önce
    api.on("subagent_spawning", async () => ({
      status: "ok",
      threadBindingReady: true,
      deliveryOrigin: { channel: "discord", to: "channel:123", threadId: "456" },
    }));

    // Sonra
    api.on("subagent_spawned", async (event) => {
      await observeSubagentLaunch(event);
    });
    ```

    `subagent_spawning`, `PluginHookSubagentSpawningEvent`,
    `PluginHookSubagentSpawningResult` ve
    `SubagentLifecycleHookRunner.runSubagentSpawning(...)`, harici plugin'ler geçiş yaparken yalnızca
    kullanımdan kaldırılmış uyumluluk yüzeyleri olarak kalır ve
    2026-08-30 sonrasında kaldırılır.

  </Accordion>

  <Accordion title="Sağlayıcı keşif türleri -> sağlayıcı katalog türleri">
    Dört keşif türü diğer adı artık katalog dönemi türlerinin ince
    sarmalayıcılarıdır:

    | Eski diğer ad             | Yeni tür                  |
    | ------------------------- | ------------------------- |
    | `ProviderDiscoveryOrder`  | `ProviderCatalogOrder`    |
    | `ProviderDiscoveryContext`| `ProviderCatalogContext`  |
    | `ProviderDiscoveryResult` | `ProviderCatalogResult`   |
    | `ProviderPluginDiscovery` | `ProviderPluginCatalog`   |

    Diğer adlar ve eski `ProviderCapabilities` statik paketi
    kaldırılmıştır. Sağlayıcı plugin'leri statik bir nesne yerine
    `buildReplayPolicy`, `normalizeToolSchemas` ve `wrapStreamFn` gibi
    açık sağlayıcı kancalarını kullanmalıdır.

  </Accordion>

  <Accordion title="Düşünme ilkesi kancaları -> resolveThinkingProfile">
    **Eski** (`ProviderThinkingPolicy` üzerinde üç ayrı kanca):
    `isBinaryThinking(ctx)`, `supportsXHighThinking(ctx)` ve
    `resolveDefaultThinkingLevel(ctx)`.

    **Yeni**: kurallı `id`, isteğe bağlı `label`
    ve derecelendirilmiş bir düzey listesi içeren
    `ProviderThinkingProfile` döndüren tek bir `resolveThinkingProfile(ctx)`. OpenClaw,
    güncelliğini yitirmiş saklanan değerleri profil sıralamasına göre otomatik olarak
    düşürür.

    Bağlam; `provider`, `modelId`, isteğe bağlı birleştirilmiş `reasoning`
    ve isteğe bağlı birleştirilmiş model `compat` olgularını içerir. Sağlayıcı plugin'leri,
    yalnızca yapılandırılmış istek sözleşmesi desteklediğinde modele özgü bir profil
    sunmak için bu katalog olgularını kullanabilir.

    Üç kanca yerine tek bir kanca uygulayın. Eski kancalar kaldırılmıştır.

  </Accordion>

  <Accordion title="Harici kimlik doğrulama sağlayıcıları -> contracts.externalAuthProviders">
    **Eski**: sağlayıcıyı plugin bildiriminde belirtmeden harici kimlik doğrulama
    kancalarını uygulamak.

    **Yeni**: plugin bildiriminde `contracts.externalAuthProviders` öğesini bildirin
    **ve** `resolveExternalAuthProfiles(...)` öğesini uygulayın.

    ```json
    {
      "contracts": {
        "externalAuthProviders": ["anthropic", "openai"]
      }
    }
    ```

  </Accordion>

  <Accordion title="Sağlayıcı ortam değişkeni araması -> setup.providers[].envVars">
    **Eski** bildirim alanı: `providerAuthEnvVars: { anthropic: ["ANTHROPIC_API_KEY"] }`.

    **Yeni**: aynı ortam değişkeni aramasını bildirimdeki `setup.providers[].envVars`
    alanına da yansıtın. Bu, kurulum/durum ortam meta verilerini tek yerde
    birleştirir ve yalnızca ortam değişkeni aramalarını yanıtlamak için plugin çalışma
    zamanını başlatmayı önler.

    `providerAuthEnvVars` artık kabul edilmez.

  </Accordion>

  <Accordion title="Bellek plugin'i kaydı -> registerMemoryCapability">
    **Eski**: üç ayrı çağrı - `api.registerMemoryPromptSection(...)`,
    `api.registerMemoryFlushPlan(...)`, `api.registerMemoryRuntime(...)`.

    **Yeni**: bellek durumu API'sinde tek çağrı -
    `registerMemoryCapability(pluginId, { promptBuilder, flushPlanResolver, runtime })`.

    Aynı yuvalar, tek kayıt çağrısı. Eklemeli istem ve derlem yardımcıları
    (`registerMemoryPromptSupplement`, `registerMemoryCorpusSupplement`) etkilenmez.

  </Accordion>

  <Accordion title="Bellek gömme sağlayıcısı API'si">
    **Eski**: `api.registerMemoryEmbeddingProvider(...)` ve
    `contracts.memoryEmbeddingProviders`.

    **Yeni**: `api.registerEmbeddingProvider(...)` ve
    `contracts.embeddingProviders`.

    Genel gömme sağlayıcısı sözleşmesi bellek dışında da yeniden kullanılabilir
    ve yeni sağlayıcılar için desteklenen yoldur. Belleğe özgü kayıt API'si,
    mevcut sağlayıcılar geçiş yaparken kullanımdan kaldırılmış uyumluluk olarak bağlı
    kalır. Plugin incelemesi, paketle birlikte sunulmayan kullanımı uyumluluk
    borcu olarak bildirir.

  </Accordion>

  <Accordion title="Ham kanal gönderim sonuçları -> OutboundDeliveryResult">
    **Eski**: `ChannelSendRawResult` üzerinden `{ ok, messageId, error }` döndürmek
    ve bunu `createRawChannelSendResultAdapter(...)` ile normalleştirmek.

    **Yeni**: `OutboundDeliveryResult` alanlarını döndürün ve kanalı
    `createAttachedChannelResultAdapter(...)` ile ekleyin. Başarısız gönderimler
    hata dizesi döndürmek yerine hata fırlatmalıdır. Ham sonuç türü bir sonraki
    plugin-SDK ana sürümüne kadar kullanılabilir kalır.

  </Accordion>

  <Accordion title="Alt ajan oturum iletileri türleri yeniden adlandırıldı">
    `src/plugins/runtime/types.ts` içinden hâlâ dışa aktarılan iki eski tür diğer adı:

    | Eski                          | Yeni                            |
    | ----------------------------- | ------------------------------- |
    | `SubagentReadSessionParams`   | `SubagentGetSessionMessagesParams` |
    | `SubagentReadSessionResult`   | `SubagentGetSessionMessagesResult` |

    `readSession` çalışma zamanı yöntemi, `getSessionMessages` lehine
    kullanımdan kaldırılmıştır. İmza aynıdır; eski yöntem çağrıyı yeni yönteme
    aktarır.

  </Accordion>

  <Accordion title="Kaldırılan oturum ve transkript dosyası API'leri">
    SQLite oturum/transkript geçişi; etkin `sessions.json` depolarını, JSONL
    transkript yollarını veya oturum dosyası listelerini sunan plugin odaklı API'leri
    kaldırır ya da kullanımdan kaldırır. Çalışma zamanı plugin'leri, etkin dosyaları
    çözümlemek veya değiştirmek yerine oturum kimliğini ve SDK çalışma zamanı
    yardımcılarını kullanmalıdır.

    | Geçiş yapılan yüzey | Yerine kullanılacak |
    | ----------------- | ----------- |
    | Kullanımdan kaldırılan `loadSessionStore(...)`, `updateSessionStore(...)` ve `resolveSessionStoreEntry(...)` | `getSessionEntry(...)`, `listSessionEntries(...)` ve satır düzeyinde oturum değişiklikleri. |
    | Kullanımdan kaldırılan `resolveSessionFilePath(...)` | Oturum kimliği (`sessionKey`, `sessionId` ve SDK çalışma zamanı hedef yardımcıları) ile geçerli oturum üzerinde çalışan Gateway yöntemleri. |
    | Kaldırılan `saveSessionStore(...)` | Gateway'e ait oturum çalışma zamanı API'leri; plugin kodu etkin depo dosyasına yazmak yerine belgelenmiş çalışma zamanı/bağlam yardımcıları üzerinden oturum durumunu istemeli veya değiştirmelidir. |
    | Kaldırılan `resolveSessionTranscriptPathInDir(...)` ve `resolveAndPersistSessionFile(...)` | Oturum kimliği ve geçerli oturum üzerinde çalışan Gateway yöntemleri. |
    | `readLatestAssistantTextFromSessionTranscript(...)` | Geçerli çalışma zamanı bağlamının sunduğu kimlik destekli transkript okuyucuları veya plugin transkript sahibi yolunun dışındaysa Gateway geçmiş/oturum yöntemleri. |
    | `SessionTranscriptUpdate.sessionFile` | `agentId`, `sessionKey` ve `sessionId` ile `SessionTranscriptUpdate.target`. |
    | `sessionFiles` gibi bellek eşitleme girdileri | Ana makinenin sağladığı kimlik destekli transkript/oturum kaynakları; canlı oturumlar için etkin JSONL dosyalarını taramayın. |
    | Etkin oturumlar için `transcriptPath` veya `sessionFile` adlı çalışma zamanı seçenekleri | Depolamadan bağımsız oturum kimliğini taşıyan `sessionTarget`/çalışma zamanı hedef nesneleri. |

    Eski JSONL transkript dosyaları içe aktarma, arşivleme, dışa aktarma ve
    destek yapıtları olarak geçerliliğini korur. Artık etkin oturumlar için
    kararlı durum çalışma zamanı sözleşmesi değildir.

    `v2026.7.1-beta.5` ile yayımlanan resmî plugin'ler, yukarıdaki kullanımdan
    kaldırılmış dört yardımcıyı içe aktarmıştır. `openclaw/plugin-sdk/session-store-runtime`, bu
    tam köprüyü 2026-10-12 tarihine kadar korur; yeni plugin'ler yerine kullanılacak
    öğeleri kullanmalıdır. `resolveStorePath(...)`, desteklenen bir SDK yardımcısı olarak
    kalır ve bu kullanımdan kaldırmanın parçası değildir.

    `openclaw plugins inspect --all --runtime`, yükleme hataları veya tanılamaları hâlâ bu kaldırılmış
    dosya API'lerine başvuran, paketle birlikte sunulmayan plugin'leri bildirir.
    `@openclaw/plugin-inspector` danışma taraması, harici paket taramalarının da
    yayımdan önce tüm depo oturum yardımcılarını, oturum dosyası yolu yardımcılarını,
    eski transkript dosyası hedeflerini ve düşük düzeyli transkript yardımcılarını
    işaretlemesi için `0.3.17` veya daha yeni bir sürümü kullanmalıdır.

  </Accordion>

  <Accordion title="runtime.tasks.flow -> runtime.tasks.managedFlows">
    **Eski**: `runtime.tasks.flow` (tekil), canlı bir görev akışı
    erişimcisi döndürüyordu.

    **Yeni**: `runtime.tasks.managedFlows`, bir akıştan alt görevler oluşturan,
    güncelleyen, iptal eden veya çalıştıran plugin'ler için yönetilen TaskFlow değişiklik
    çalışma zamanını korur. Plugin yalnızca DTO tabanlı okumalara ihtiyaç duyuyorsa
    `runtime.tasks.flows` kullanın.

    ```typescript
    // Önce
    const flow = api.runtime.tasks.flow.fromToolContext(ctx);
    // Sonra
    const flow = api.runtime.tasks.managedFlows.fromToolContext(ctx);
    ```

    Eski takma adlar Temmuz 2026'da kaldırıldı.

  </Accordion>

  <Accordion title="Gömülü uzantı fabrikaları -> agent araç sonucu ara yazılımı">
    Yukarıdaki [Geçiş nasıl yapılır](#how-to-migrate) bölümünde ele alınmıştır. Eksiksizlik
    adına burada da belirtilmiştir: yalnızca kaldırılan gömülü çalıştırıcıya özgü
    `api.registerEmbeddedExtensionFactory(...)` yolu, `contracts.agentToolResultMiddleware` içinde açık bir çalışma zamanı
    listesiyle `api.registerAgentToolResultMiddleware(...)` tarafından değiştirilmiştir.
  </Accordion>

  <Accordion title="OpenClawSchemaType takma adı -> OpenClawConfig">
    `OpenClawSchemaType` kök SDK takma adı kaldırıldı. Standart
    `OpenClawConfig` adını kullanın.

    ```typescript
    // Önce
    import type { OpenClawSchemaType } from "openclaw/plugin-sdk";
    // Sonra
    import type { OpenClawConfig } from "openclaw/plugin-sdk/config-contracts";
    ```

  </Accordion>
</AccordionGroup>

<Note>
Uzantı düzeyindeki kullanımdan kaldırmalar (`extensions/` altındaki paketle gelen
kanal/sağlayıcı Plugin'lerinin içinde), kendi `api.ts` ve `runtime-api.ts`
barrel'larında izlenir. Bunlar üçüncü taraf Plugin sözleşmelerini etkilemez ve
burada listelenmez. Paketle gelen bir Plugin'in yerel barrel'ını doğrudan
kullanıyorsanız yükseltmeden önce o barrel'daki kullanımdan kaldırma yorumlarını
okuyun.
</Note>

## Talk ve gerçek zamanlı ses geçişi

Gerçek zamanlı ses, telefon, toplantı ve tarayıcı Talk kodu,
`openclaw/plugin-sdk/realtime-voice` tarafından dışa aktarılan tek bir Talk oturum
denetleyicisini paylaşır. Denetleyici; ortak Talk olay zarfının, etkin tur
durumunun, yakalama durumunun, ses çıkışı durumunun, yakın tarihli olay
geçmişinin ve eski tur reddinin sahibidir. Sağlayıcı Plugin'leri, satıcıya özgü
gerçek zamanlı oturumların sahibidir. Tarayıcı toplantısı Plugin'leri; oturum,
tarayıcı, ses, node ana makinesi, agent danışması ve sesli arama mekanikleri
için `openclaw/plugin-sdk/meeting-runtime` kullanır, ardından URL kuralları, DOM betikleri,
manuel eylem eşlemesi, altyazılar, oluşturma ve telefonla katılım planları için
`MeetingPlatformAdapter` uygular. Platform REST API'leri, OAuth, yapıtlar,
seçiciler ve aktarım adları Plugin'de kalır. Tarayıcı izin planları, istenen
toplantı URL'sini alır; böylece her platform yalnızca tam olarak desteklediği
kaynaklara izin verebilir. Oturum çalışma zamanları, tarayıcıdan ayrılmanın
doğrulanmasının ardından platforma özgü canlı sağlık durumunu da
normalleştirmelidir; geçmiş döküm alanları kalabilir ancak ayrıldıktan sonra
altyazı ve ses hazırlığı etkin kalmamalıdır.

Paketle gelen tüm yüzeyler paylaşılan denetleyicide çalışır: tarayıcı aktarımı,
yönetilen oda devri, sesli aramada gerçek zamanlı çalışma, sesli aramada akışlı
STT, Google Meet gerçek zamanlı çalışma ve yerel bas-konuş. Gateway,
`hello-ok.features.events` içinde tek bir canlı Talk olay kanalı duyurur:
`talk.event`.

Yeni kod, düşük düzeyli bir bağdaştırıcı veya test fikstürü uygulamadığı sürece
`createTalkEventSequencer(...)` işlevini doğrudan çağırmamalıdır. Tur kapsamındaki
olayların tur kimliği olmadan yayımlanamaması, eski `turnEnd` /
`turnCancel` çağrılarının daha yeni bir etkin turu temizleyememesi ve
ses çıkışı yaşam döngüsü olaylarının telefon, toplantılar, tarayıcı aktarımı,
yönetilen oda devri ve yerel Talk istemcileri genelinde tutarlı kalması için
paylaşılan denetleyiciyi kullanın.

Genel API biçimi:

```typescript
// Gateway'in sahip olduğu Talk oturumu API'si.
await gateway.request("talk.session.create", {
  mode: "realtime",
  transport: "gateway-relay",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.session.appendAudio", { sessionId, audioBase64 });
await gateway.request("talk.session.cancelOutput", { sessionId, reason: "barge-in" });
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "working" },
  options: { willContinue: true },
});
await gateway.request("talk.session.submitToolResult", {
  sessionId,
  callId,
  result: { status: "already_delivered" },
  options: { suppressResponse: true },
});
await gateway.request("talk.session.submitToolResult", { sessionId, callId, result });
await gateway.request("talk.session.close", { sessionId });

// İstemcinin sahip olduğu sağlayıcı oturumu API'si.
await gateway.request("talk.client.create", {
  mode: "realtime",
  transport: "webrtc",
  brain: "agent-consult",
  sessionKey: "main",
});
await gateway.request("talk.client.toolCall", { sessionKey, callId, name, args });
await gateway.request("talk.client.steer", { sessionKey, text, mode: "steer" });
```

Tarayıcının sahip olduğu WebRTC/sağlayıcı websocket oturumları
`talk.client.create` kullanır; çünkü sağlayıcı anlaşmasının ve medya aktarımının
sahibi tarayıcı, kimlik bilgilerinin, talimatların ve araç politikasının sahibi
ise Gateway'dir. `talk.session.*`; Gateway aktarımı üzerinden gerçek zamanlı
çalışma, Gateway aktarımı üzerinden döküm ve yönetilen oda yerel STT/TTS
oturumları için Gateway tarafından yönetilen ortak yüzeydir.

Gerçek zamanlı seçicileri `talk.provider` / `talk.providers` yanına
yerleştiren eski yapılandırmalar `openclaw doctor --fix` ile onarılmalıdır;
çalışma zamanı Talk, konuşma/TTS sağlayıcı yapılandırmasını gerçek zamanlı
sağlayıcı yapılandırması olarak yeniden yorumlamaz.

Desteklenen `talk.session.create` birleşimleri kasıtlı olarak sınırlıdır:

| Mod             | Aktarım         | Beyin           | Sahip              | Notlar                                                                                                                      |
| --------------- | --------------- | --------------- | ------------------ | --------------------------------------------------------------------------------------------------------------------------- |
| `realtime`      | `gateway-relay` | `agent-consult` | Gateway            | Gateway üzerinden köprülenen tam çift yönlü sağlayıcı sesi; araç çağrıları agent danışma aracı üzerinden yönlendirilir.     |
| `transcription` | `gateway-relay` | `none`          | Gateway            | Yalnızca akışlı STT; çağıranlar giriş sesi gönderir ve döküm olayları alır.                                                  |
| `stt-tts`       | `managed-room`  | `agent-consult` | Yerel/istemci odası | İstemcinin yakalama/oynatma, Gateway'in ise tur durumu sahibi olduğu bas-konuş ve telsiz tarzı odalar.                       |
| `stt-tts`       | `managed-room`  | `direct-tools`  | Yerel/istemci odası | Gateway araç eylemlerini doğrudan yürüten, güvenilir birinci taraf yüzeylerine yönelik yalnızca yöneticiye açık oda modu.    |

Eski `talk.realtime.*` / `talk.transcription.*` / `talk.handoff.*`
ailelerinden geçiş yapan okuyucular için yöntem eşlemesi (tümü kaldırıldı):

| Eski                             | Yeni                                                     |
| -------------------------------- | -------------------------------------------------------- |
| `talk.realtime.session`          | `talk.client.create`                                     |
| `talk.realtime.toolCall`         | `talk.client.toolCall`                                   |
| `talk.realtime.relayAudio`       | `talk.session.appendAudio`                               |
| `talk.realtime.relayCancel`      | `talk.session.cancelOutput` veya `talk.session.cancelTurn` |
| `talk.realtime.relayToolResult`  | `talk.session.submitToolResult`                          |
| `talk.realtime.relayStop`        | `talk.session.close`                                     |
| `talk.transcription.session`     | `talk.session.create({ mode: "transcription" })`         |
| `talk.transcription.relayAudio`  | `talk.session.appendAudio`                               |
| `talk.transcription.relayCancel` | `talk.session.cancelTurn`                                |
| `talk.transcription.relayStop`   | `talk.session.close`                                     |
| `talk.handoff.create`            | `talk.session.create({ transport: "managed-room" })`     |
| `talk.handoff.join`              | `talk.session.join`                                      |
| `talk.handoff.revoke`            | `talk.session.close`                                     |

Birleşik denetim söz dağarcığı da bilinçli olarak dardır:

| Yöntem                          | Uygulandığı yer                                         | Sözleşme                                                                                                                                                                                                                  |
| ------------------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `talk.session.appendAudio`      | `realtime/gateway-relay`, `transcription/gateway-relay` | Aynı Gateway bağlantısının sahip olduğu sağlayıcı oturumuna base64 PCM ses parçası ekler.                                                                                                                                  |
| `talk.session.startTurn`        | `stt-tts/managed-room`                                  | Yönetilen oda kullanıcı turunu başlatır.                                                                                                                                                                                  |
| `talk.session.endTurn`          | `stt-tts/managed-room`                                  | Eski tur doğrulamasından sonra etkin turu sonlandırır.                                                                                                                                                                    |
| `talk.session.cancelTurn`       | Gateway'in sahip olduğu tüm oturumlar                   | Bir tur için etkin yakalama/sağlayıcı/agent/TTS çalışmasını iptal eder.                                                                                                                                                    |
| `talk.session.cancelOutput`     | `realtime/gateway-relay`                                | Kullanıcı turunu mutlaka sonlandırmadan asistanın ses çıkışını durdurur.                                                                                                                                                   |
| `talk.session.submitToolResult` | `realtime/gateway-relay`                                | Köprüsünün sunduğu eşzamansız tamamlanmalardan sonra bir sağlayıcı araç çağrısını tamamlar; ara çıktı için `options.willContinue`, desteklendiğinde başka bir asistan yanıtını önlemek için `options.suppressResponse` iletin. |
| `talk.session.steer`            | agent destekli Talk oturumları                          | Talk oturumundan çözümlenen etkin gömülü çalıştırmaya sözlü `status`, `steer`, `cancel` veya `followup` denetimi gönderir.                                                          |
| `talk.session.close`            | tüm birleşik oturumlar                                  | Aktarım oturumlarını durdurur veya yönetilen oda durumunu iptal eder, ardından birleşik oturum kimliğini unutur.                                                                                                           |

Bunun çalışmasını sağlamak için çekirdeğe sağlayıcıya veya platforma özgü özel
durumlar eklemeyin. Talk oturumu semantiğinin sahibi çekirdektir. Satıcı
oturumu kurulumunun sahibi sağlayıcı Plugin'leridir. Telefon/toplantı
bağdaştırıcılarının sahibi sesli arama ve Google Meet'tir. Cihaz
yakalama/oynatma kullanıcı deneyiminin sahibi tarayıcı ve yerel
uygulamalardır.

## Kaldırma zaman çizelgesi

| Ne zaman                                        | Ne olur                                                                                                                              |
| ------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Şimdi**                                     | Uyarı verebilen kullanımdan kaldırılmış yüzeyler çalışma zamanı uyarıları yayınlar; depo korumaları, çekirdekten ve paketlenmiş pluginlerden kullanımdan kaldırılmış SDK içe aktarımlarını reddeder. |
| **Sahip kararı bekleniyor**                  | Tarihsiz kayıtlar, sahipleri bir `removeAfter` tarihi yayımlayana kadar kullanımdan kaldırılmış durumda kalır ve silinmeye uygun olmaz.                          |
| **Her uyumluluk kaydının `removeAfter` tarihi** | Söz konusu yüzey silinmeye uygun hâle gelir; tarih geçtikten sonra `pnpm plugins:boundary-report --fail-on-eligible-compat` CI'da başarısız olur.    |
| **Sonraki ana sürüm**                      | Tarihli yüzeyler yalnızca `removeAfter` tarihlerinden sonra silinebilir; tarihsiz kayıtlar için hâlâ sahip onayı ve yayımlanmış bir tarih gerekir.   |

Aşağıdaki kalan genel SDK alt yollarının kayıt defteri destekli kaldırma zaman aralıkları vardır.
30 Temmuz satırları, bakımcıların erken yetkilendirdiği taramadan sonra kaldırıldı:
kullanılmayan alt yollar silindi, önceki uyumluluk takma adları silindi ve
yalnızca paketlenmiş modüller özel-yerel derleme eşlemelerine indirgendi.

| `removeAfter` | Kademe                               | SDK alt yolları                                                                                                                                                                        |
| ------------- | ---------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `2026-08-15`  | Önceki uyumluluk kullanımdan kaldırmaları | `agent-config-primitives`, `channel-logging`, `channel-secret-runtime`, `channel-streaming`, `group-access`, `inbound-reply-dispatch`, `matrix`, `text-runtime`, `zod`              |
| `2026-09-01`  | Önceki uyumluluk kullanımdan kaldırmaları | `channel-lifecycle`, `channel-message`, `channel-reply-pipeline`, `config-runtime`, `infra-runtime`                                                                                 |
| `2026-10-01`  | Eski medya projeksiyonu            | `agent-media-payload`; ayrıca alt yol olmayan `MsgContext Media*` alanları, kanal gelen medya yükü oluşturucuları, `buildMediaPayload`, kanca medya takma adları ve `{{Media*}}` şablonları |

Tüm çekirdek pluginler zaten geçirildi. Harici pluginler
sonraki ana sürümden önce geçiş yapmalıdır. Plugininizin kullandığı yüzeylerde
en yakında süresi dolacak uyumluluk kayıtlarını görmek için `pnpm plugins:boundary-report` komutunu çalıştırın.

## Uyarıları geçici olarak bastırma

```bash
OPENCLAW_SUPPRESS_PLUGIN_SDK_COMPAT_WARNING=1 openclaw gateway run
OPENCLAW_SUPPRESS_EXTENSION_API_WARNING=1 openclaw gateway run
```

Bu, kalıcı bir çözüm değil, geçici bir kaçış yoludur.

## İlgili

- [Başlarken](/tr/plugins/building-plugins) - ilk plugininizi oluşturun
- [SDK'ya Genel Bakış](/tr/plugins/sdk-overview) - tam alt yol içe aktarma referansı
- [Kanal Pluginleri](/tr/plugins/sdk-channel-plugins) - kanal pluginleri oluşturma
- [Sağlayıcı Pluginleri](/tr/plugins/sdk-provider-plugins) - sağlayıcı pluginleri oluşturma
- [Plugin İç Yapısı](/tr/plugins/architecture) - ayrıntılı mimari incelemesi
- [Plugin Manifesti](/tr/plugins/manifest) - manifest şeması referansı
