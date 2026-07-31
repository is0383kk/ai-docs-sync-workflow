---
read_when:
    - Sağlayıcı çalışma zamanı kancalarını, kanal yaşam döngüsünü veya paket paketlerini uygulama
    - Plugin yükleme sırası veya kayıt defteri durumunda hata ayıklama
    - Yeni bir Plugin özelliği veya bağlam motoru Plugin'i ekleme
summary: 'Plugin mimarisi iç işleyişi: yükleme işlem hattı, kayıt defteri, çalışma zamanı kancaları, HTTP rotaları ve referans tabloları'
title: Plugin mimarisi iç işleyişi
x-i18n:
    generated_at: "2026-07-27T00:06:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 278ac23a9454ab69407c59fa197e75756fa0dc5880fcae6c3eecc15bd4733a09
    source_path: plugins/architecture-internals.md
    workflow: 16
---

Genel yetenek modeli, plugin biçimleri ve sahiplik/yürütme
sözleşmeleri için [Plugin mimarisi](/tr/plugins/architecture) bölümüne bakın. Bu sayfa
dahili mekanikleri kapsar: yükleme işlem hattı, kayıt defteri, çalışma zamanı kancaları, Gateway HTTP
rotaları, içe aktarma yolları ve şema tabloları.

## Yükleme işlem hattı

Başlangıçta OpenClaw kabaca şunları yapar:

1. aday plugin köklerini keşfeder
2. yerel veya uyumlu paket bildirimlerini ve paket meta verilerini okur
3. güvenli olmayan adayları reddeder
4. plugin yapılandırmasını normalleştirir (`plugins.enabled`, `allow`, `deny`, `entries`,
   `slots`, `load.paths`)
5. her aday için etkinleştirme durumuna karar verir
6. etkin yerel modülleri yükler: derlenmiş paket içi modüller yerel bir yükleyici kullanır;
   üçüncü taraf yerel kaynak TypeScript, acil durum Jiti yedek mekanizmasını kullanır
7. yerel `register(api)` kancalarını çağırır ve kayıtları plugin kayıt defterinde toplar
8. kayıt defterini komutlara/çalışma zamanı yüzeylerine sunar

Güvenlik geçitleri çalışma zamanı yürütmesinden **önce** çalışır. Keşif, aşağıdaki
durumlarda bir adayı engeller:

- çözümlenen giriş noktası plugin kökünün dışına çıkıyorsa
- yolu (veya kök dizini) herkes tarafından yazılabiliyorsa
- paket içi olmayan pluginlerde yol sahipliği mevcut uid (veya root) ile eşleşmiyorsa

Herkes tarafından yazılabilen paket içi dizinlerde, geçit yeniden
denetlemeden önce yerinde bir `chmod` onarım denemesi yapılır
(npm/genel kurulumlar paket dizinlerini `0777` konumunda sunabilir);
paket içi kaynaklar için sahiplik denetimleri tamamen atlanır.

Engellenen adaylar, biliniyorsa plugin kimliklerini yayımlanan tanılamada
taşımaya devam eder (başka nedenlerle reddedilen bir dizinin içindeki
bildirimden çözümlenen kimlikler dâhil). Böylece bu kimliğe başvuran yapılandırma,
ilgisiz bir "bilinmeyen plugin" hatası yerine yol güvenliği uyarısına bağlı
engellenmiş bir plugin görür.

### Bildirim öncelikli davranış

Bildirim, kontrol düzleminin doğruluk kaynağıdır. OpenClaw bunu şu amaçlarla kullanır:

- plugini tanımlamak
- bildirilen kanalları/Skills/yapılandırma şemasını veya paket yeteneklerini keşfetmek
- `plugins.entries.<id>.config` değerini doğrulamak
- Control UI etiketlerini/yer tutucularını zenginleştirmek
- kurulum/katalog meta verilerini göstermek
- plugin çalışma zamanını yüklemeden düşük maliyetli etkinleştirme ve kurulum tanımlayıcılarını korumak

Yerel pluginlerde çalışma zamanı modülü, veri düzlemi kısmıdır. Kancalar,
araçlar, komutlar veya sağlayıcı akışları gibi gerçek davranışları kaydeder.

İsteğe bağlı bildirim `activation` ve `setup` blokları kontrol düzleminde kalır.
Bunlar etkinleştirme planlaması ve kurulum keşfi için yalnızca meta veri
tanımlayıcılarıdır; çalışma zamanı kaydının, `register(...)` veya `setupEntry` yerini almazlar.
Canlı etkinleştirme tüketicileri, daha geniş kayıt defteri oluşturulmadan önce
plugin yüklemesini daraltmak için bildirimdeki komut, kanal ve sağlayıcı ipuçlarını kullanır:

- CLI yüklemesi, istenen birincil komutun sahibi olan pluginlerle sınırlandırılır
- kanal kurulumu/plugin çözümlemesi, istenen kanal kimliğinin sahibi olan
  pluginlerle sınırlandırılır
- açık sağlayıcı kurulumu/çalışma zamanı çözümlemesi, istenen sağlayıcı kimliğinin sahibi olan
  pluginlerle sınırlandırılır
- Gateway başlangıç planlaması açık başlangıç içe aktarmaları için `activation.onStartup` kullanır;
  başlangıç meta verisi olmayan pluginler yalnızca daha dar
  etkinleştirme tetikleyicileri üzerinden yüklenir

Etkinleştirme planlayıcısı, mevcut çağıranlar için yalnızca kimliklerden oluşan
bir API'nin yanı sıra tanılama için bir plan API'si sunar. Plan girdileri, bir
pluginin neden seçildiğini bildirerek açık `activation.*` ipuçlarını bildirim sahipliği yedek davranışından ayırır:

| Neden (`activation.*` ipuçlarından)   | Neden (bildirim sahipliğinden)                                                             |
| ------------------------------------ | -------------------------------------------------------------------------------------------- |
| `activation-agent-harness-hint`      | —                                                                                            |
| `activation-capability-hint`         | —                                                                                            |
| `activation-channel-hint`            | `manifest-channel-owner` (`channels`)                                                        |
| `activation-command-hint`            | `manifest-command-alias` (`commandAliases`)                                                  |
| `activation-provider-hint`           | `manifest-provider-owner` (`providers`), `manifest-setup-provider-owner` (`setup.providers`) |
| `activation-route-hint`              | —                                                                                            |
| — (kanca tetikleyicisinin ipucu çeşidi yoktur) | `manifest-hook-owner` (`hooks`), `manifest-tool-contract` (`contracts.tools`)                |

Bu neden ayrımı uyumluluk sınırıdır: mevcut plugin meta verileri çalışmaya devam
ederken yeni kod, çalışma zamanı yükleme anlamını değiştirmeden geniş ipuçlarını
veya yedek davranışı algılayabilir.

Geniş `all` kapsamını isteyen istek zamanındaki çalışma zamanı ön yüklemeleri,
yine de yapılandırma, başlangıç planlaması, yapılandırılmış kanallar, yuvalar ve
otomatik etkinleştirme kurallarından
(`src/plugins/effective-plugin-ids.ts` içindeki `resolveEffectivePluginIds`) açık bir etkin plugin kimliği kümesi türetir.
Bu türetilmiş küme boşsa OpenClaw, kapsamı keşfedilebilir her plugini içerecek
şekilde genişletmek yerine boş tutar.

Kurulum keşfi, aday pluginleri daraltmak için `setup.providers` ve
`setup.cliBackends` gibi tanımlayıcıya ait kimlikleri tercih eder; ardından
kurulum zamanı çalışma kancalarına hâlâ ihtiyaç duyan pluginler için
`setup-api` değerine geri döner. Sağlayıcı kurulum listeleri, sağlayıcı
çalışma zamanını yüklemeden bildirim `providerAuthChoices`, tanımlayıcıdan türetilen
kurulum seçenekleri ve kurulum kataloğu meta verilerini kullanır. Açık
`setup.requiresRuntime: false` yalnızca tanımlayıcıya dayalı bir kesme noktasıdır; belirtilmeyen
`requiresRuntime`, uyumluluk için eski kurulum API'si yedek davranışını korur.
Keşfedilen birden fazla plugin aynı normalleştirilmiş kurulum sağlayıcısı veya
CLI arka uç kimliğini sahiplenirse kurulum araması, keşif sırasına güvenmek
yerine belirsiz sahibi reddeder. Kurulum çalışma zamanı yürütüldüğünde kayıt
defteri tanılamaları, eski pluginleri engellemeden `setup.providers` /
`setup.cliBackends` ile kurulum API'si tarafından gerçekten kaydedilen sağlayıcılar
veya CLI arka uçları arasındaki sapmayı bildirir.

### Plugin önbellek sınırı

OpenClaw, plugin keşif sonuçlarını veya doğrudan bildirim kayıt defteri
verilerini duvar saati zaman aralıklarının arkasında önbelleğe almaz. Kurulumlar,
bildirim düzenlemeleri ve yükleme yolu değişiklikleri bir sonraki açık meta veri
okumasında veya anlık görüntü yeniden oluşturmasında görünür hâle gelmelidir.
Bildirim dosyası ayrıştırıcısı, açılan bildirim yolu ile aygıt/inode, boyut ve
mtime/ctime değerlerine göre anahtarlanan sınırlı bir dosya imzası önbelleği
tutar; bu önbellek yalnızca değişmemiş baytların yeniden ayrıştırılmasını önler
ve keşif, kayıt defteri, sahip veya ilke yanıtlarını önbelleğe almamalıdır.

Güvenli ve hızlı meta veri yolu, gizli bir önbellek değil açık nesne sahipliğidir.
Gateway başlangıç hızlı yolları mevcut `PluginMetadataSnapshot`, türetilmiş
`PluginLookUpTable` veya açık bir bildirim kayıt defterini çağrı zinciri boyunca
aktarmalıdır. Yapılandırma doğrulaması, başlangıçta otomatik etkinleştirme, plugin
önyüklemesi ve sağlayıcı seçimi, bu nesneler mevcut yapılandırmayı ve plugin
envanterini temsil ettiği sürece bunları yeniden kullanabilir. İlgili kurulum
yoluna açık bir bildirim kayıt defteri verilmediği sürece kurulum araması,
bildirim meta verilerini yine isteğe bağlı olarak yeniden oluşturur; gizli arama
önbellekleri eklemek yerine bunu soğuk yol yedek davranışı olarak tutun. Girdi
değiştiğinde anlık görüntüyü değiştirmek veya geçmiş kopyaları saklamak yerine
yeniden oluşturup değiştirin. Etkin plugin kayıt defteri üzerindeki görünümler
ve paket içi kanal önyükleme yardımcıları mevcut kayıt defterinden/kökten
yeniden hesaplanmalıdır. Kısa ömürlü eşlemeler, tek bir çağrı içinde yinelenen
işleri kaldırmak veya yeniden girişi engellemek için uygundur; süreç meta verisi
önbelleklerine dönüşmemelidir.

Plugin yüklemesinde kalıcı önbellek katmanı çalışma zamanı yüklemesidir. Kod veya
kurulu yapılar gerçekten yüklendiğinde yükleyici durumunu yeniden kullanabilir;
örneğin:

- `PluginLoaderCacheState` ve uyumlu etkin çalışma zamanı kayıt defterleri
- aynı çalışma zamanı yüzeyini tekrar tekrar içe aktarmaktan kaçınmak için kullanılan jiti/modül önbellekleri ve genel yüzey yükleyici önbellekleri
- kurulu plugin yapıları için dosya sistemi önbellekleri
- yol normalleştirme veya yinelenen çözümleme için kısa ömürlü çağrı başına eşlemeler

Bu önbellekler veri düzlemi uygulama ayrıntılarıdır. Çağıran bilinçli olarak
çalışma zamanı yüklemesi istemediği sürece "bu sağlayıcının sahibi hangi plugin?"
gibi kontrol düzlemi sorularını yanıtlamamalıdır.

Şunlar için kalıcı veya duvar saati tabanlı önbellekler eklemeyin:

- keşif sonuçları
- doğrudan bildirim kayıt defterleri
- kurulu plugin dizininden yeniden oluşturulan bildirim kayıt defterleri
- sağlayıcı sahibi araması, model engelleme, sağlayıcı ilkesi veya genel yapı
  meta verileri
- değişen bir bildirimin, kurulu dizinin veya yükleme yolunun bir sonraki meta veri okumasında görünmesi gereken, bildirimden türetilmiş diğer tüm yanıtlar

Kalıcı kurulu plugin dizininden bildirim meta verilerini yeniden oluşturan
çağıranlar, bu kayıt defterini isteğe bağlı olarak yeniden oluşturur. Kurulu
dizin, kalıcı kaynak düzlemi durumudur; gizli bir süreç içi meta veri önbelleği
değildir.

## Kayıt defteri modeli

Yüklenen pluginler rastgele çekirdek genel değişkenlerini doğrudan değiştirmez.
Plugin kayıtlarını (kimlik, kaynak, köken, durum, tanılamalar) ve her yetenek
için dizileri izleyen merkezi bir plugin kayıt defterine
(`src/plugins/registry-types.ts` içindeki `PluginRegistry`) kayıt yaparlar: araçlar, eski
kancalar ve türü belirlenmiş kancalar, kanallar, sağlayıcılar, Gateway RPC
işleyicileri, HTTP rotaları, CLI kaydedicileri, arka plan hizmetleri, plugine ait
komutlar ve düzinelerce başka türü belirlenmiş sağlayıcı ailesi (konuşma,
gömmeler, görüntü/video/müzik üretimi, web getirme/arama, aracı düzenekleri,
oturum eylemleri ve benzerleri).

Ardından çekirdek özellikler plugin modülleriyle doğrudan iletişim kurmak yerine
bu kayıt defterinden okur. Bu, yüklemeyi tek yönlü tutar:

- plugin modülü -> kayıt defteri kaydı
- çekirdek çalışma zamanı -> kayıt defteri tüketimi

Bu ayrım sürdürülebilirlik açısından önemlidir. Çoğu çekirdek yüzeyin "her
plugin modülünü özel olarak işlemesi" değil, yalnızca tek bir entegrasyon
noktasına, yani "kayıt defterini okuması" gerektiği anlamına gelir.

## Konuşma bağlama geri çağrıları

Bir konuşmayı bağlayan pluginler, onay sonuçlandığında tepki verebilir.

Bir bağlama isteği onaylandıktan veya reddedildikten sonra geri çağrı almak için
`api.onConversationBindingResolved(...)` kullanın:

```ts
export default {
  id: "my-plugin",
  register(api) {
    api.onConversationBindingResolved(async (event) => {
      if (event.status === "approved") {
        // Artık bu plugin + konuşma için bir bağlama mevcut.
        console.log(event.binding?.conversationId);
        return;
      }

      // İstek reddedildi; bekleyen tüm yerel durumları temizleyin.
      console.log(event.request.conversation.conversationId);
    });
  },
};
```

Geri çağrı yükü alanları:

- `status`: `"approved"` veya `"denied"`
- `decision`: `"allow-once"`, `"allow-always"` veya `"deny"`
- `binding`: onaylanan istekler için çözümlenen bağlama
- `request`: özgün istek özeti, ayırma ipucu, gönderen kimliği ve
  konuşma meta verileri

Bu geri çağrı yalnızca bildirim amaçlıdır. Bir konuşmayı bağlamasına izin verilen
kişileri değiştirmez ve çekirdek onay işlemi tamamlandıktan sonra çalışır.

## Sağlayıcı çalışma zamanı kancaları

Sağlayıcı pluginleri üç katmana sahiptir:

- Çalışma zamanı öncesi düşük maliyetli arama için **bildirim meta verileri**:
  `setup.providers[].envVars`, `providerAuthAliases`, `providerAuthChoices`
  ve `channelConfigs`.
- **Yapılandırma zamanı kancaları**: `catalog` ile `applyConfigDefaults`.
- **Çalışma zamanı kancaları**: kimlik doğrulama, model çözümleme,
  akış sarmalama, düşünme düzeyleri, yeniden oynatma ilkesi ve kullanım uç
  noktalarını kapsayan 40'tan fazla isteğe bağlı kanca. Bkz.
  [Kanca sırası ve kullanımı](#hook-order-and-usage).

OpenClaw genel ajan döngüsünü, yük devretmeyi, transkript işlemeyi ve
araç politikasını hâlâ yönetir. Bu kancalar, tamamen özel bir çıkarım aktarımına
gerek kalmadan sağlayıcıya özgü davranışlar için genişletme yüzeyidir.

Sağlayıcının, Plugin çalışma zamanı yüklenmeden genel kimlik doğrulama/durum/model seçici
yollarının görmesi gereken ortam tabanlı kimlik bilgileri olduğunda manifest `setup.providers[].envVars`
kullanın. Bir sağlayıcı kimliğinin başka bir sağlayıcı kimliğine ait ortam değişkenlerini,
kimlik doğrulama profillerini, yapılandırma destekli kimlik doğrulamayı ve API anahtarı ilk katılım
seçeneğini yeniden kullanması gerektiğinde manifest `providerAuthAliases`
kullanın. İlk katılım/kimlik doğrulama seçimi CLI yüzeylerinin, sağlayıcı çalışma zamanı
yüklenmeden sağlayıcının seçim kimliğini, grup etiketlerini ve basit tek bayraklı kimlik doğrulama
bağlantısını bilmesi gerektiğinde manifest `providerAuthChoices` kullanın. İlk katılım etiketleri veya OAuth
istemci kimliği/istemci gizli anahtarı kurulum değişkenleri gibi operatöre yönelik ipuçları için sağlayıcı çalışma zamanı
`envVars` kullanın.

Ortam tarafından yönlendirilen kanal kurulumunu ve kimlik doğrulamayı bunların sahibi olan
`channelConfigs.<id>.schema` ve kurulum tanımlayıcıları aracılığıyla açıklayın.

### Kanca sırası ve kullanımı

Model/sağlayıcı Pluginleri için OpenClaw, kancaları kabaca bu sırayla çağırır.
"Ne zaman kullanılmalı" sütunu hızlı karar kılavuzudur.
OpenClaw'ın artık çağırmadığı, yalnızca uyumluluk amaçlı `ProviderPlugin.capabilities`
ve `suppressBuiltInModel` gibi sağlayıcı alanları kasıtlı olarak
burada listelenmemiştir.

| Hook                              | Ne yaptığı                                                                                                   | Ne zaman kullanılacağı                                                                                                                                   |
| --------------------------------- | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `catalog`                         | `models.json` oluşturulurken sağlayıcı yapılandırmasını `models.providers` içine yayımlar                                | Sağlayıcı bir kataloğun veya temel URL varsayılanlarının sahibidir                                                                                                  |
| `applyConfigDefaults`             | Yapılandırma somutlaştırılırken sağlayıcının sahip olduğu genel yapılandırma varsayılanlarını uygular                                      | Varsayılanlar kimlik doğrulama moduna, ortama veya sağlayıcı model ailesi semantiğine bağlıdır                                                                         |
| _(yerleşik model araması)_         | OpenClaw önce normal kayıt defteri/katalog yolunu dener                                                          | _(bir plugin kancası değildir)_                                                                                                                         |
| `normalizeModelId`                | Aramadan önce eski veya önizleme model kimliği diğer adlarını normalleştirir                                                     | Sağlayıcı, kurallı model çözümlemesinden önce diğer ad temizliğinin sahibidir                                                                                 |
| `normalizeTransport`              | Genel model derlemesinden önce sağlayıcı ailesine ait `api` / `baseUrl` öğelerini normalleştirir                                      | Sağlayıcı, aynı aktarım ailesindeki özel sağlayıcı kimlikleri için aktarım temizliğinin sahibidir                                                          |
| `normalizeConfig`                 | Çalışma zamanı/sağlayıcı çözümlemesinden önce `models.providers.<id>` öğesini normalleştirir                                           | Sağlayıcı, plugin ile birlikte bulunması gereken yapılandırma temizliğine ihtiyaç duyar; paketlenmiş Google ailesi yardımcıları da desteklenen Google yapılandırma girdileri için güvence sağlar   |
| `applyNativeStreamingUsageCompat` | Yerel akış kullanımı uyumluluk yeniden yazımlarını yapılandırma sağlayıcılarına uygular                                               | Sağlayıcı, uç nokta güdümlü yerel akış kullanımı meta verisi düzeltmelerine ihtiyaç duyar                                                                          |
| `resolveConfigApiKey`             | Çalışma zamanı kimlik doğrulaması yüklenmeden önce yapılandırma sağlayıcıları için ortam işaretçisi kimlik doğrulamasını çözümler                                       | Sağlayıcılar kendi ortam işaretçisi API anahtarı çözümleme kancalarını sunar                                                                                |
| `resolveSyntheticAuth`            | Düz metni kalıcılaştırmadan yerel/kendi barındırılan veya yapılandırma destekli kimlik doğrulamasını gösterir                                   | Sağlayıcı, sentetik/yerel bir kimlik bilgisi işaretçisiyle çalışabilir                                                                                 |
| `resolveExternalAuthProfiles`     | Sağlayıcıya ait harici kimlik doğrulama profillerini katmanlar; varsayılan `persistence`, CLI/uygulama sahipli kimlik bilgileri için `runtime-only` değeridir | Sağlayıcı, kopyalanan yenileme belirteçlerini kalıcılaştırmadan harici kimlik doğrulama bilgilerini yeniden kullanır; manifestte `contracts.externalAuthProviders` bildirin |
| `shouldDeferSyntheticProfileAuth` | Saklanan sentetik profil yer tutucularının önceliğini ortam/yapılandırma destekli kimlik doğrulamasının arkasına düşürür                                      | Sağlayıcı, öncelik kazanmaması gereken sentetik yer tutucu profilleri saklar                                                                 |
| `resolveDynamicModel`             | Henüz yerel kayıt defterinde bulunmayan sağlayıcı sahipli model kimlikleri için eşzamanlı geri dönüş                                       | Sağlayıcı rastgele üst akış model kimliklerini kabul eder                                                                                                 |
| `prepareDynamicModel`             | Eşzamansız ısınma gerçekleştirir, ardından `resolveDynamicModel` yeniden çalışır                                                           | Sağlayıcı, bilinmeyen kimlikleri çözümlemeden önce ağ meta verilerine ihtiyaç duyar                                                                                  |
| `normalizeResolvedModel`          | Gömülü çalıştırıcı çözümlenen modeli kullanmadan önce son yeniden yazımı gerçekleştirir                                               | Sağlayıcı aktarım yeniden yazımlarına ihtiyaç duyar ancak yine de çekirdek aktarımı kullanır                                                                             |
| `normalizeToolSchemas`            | Gömülü çalıştırıcı görmeden önce araç şemalarını normalleştirir                                                    | Sağlayıcı, aktarım ailesi şema temizliğine ihtiyaç duyar                                                                                                |
| `inspectToolSchemas`              | Normalleştirmeden sonra sağlayıcıya ait şema tanılamalarını gösterir                                                  | Sağlayıcı, çekirdeğe sağlayıcıya özgü kuralları öğretmeden anahtar sözcük uyarıları ister                                                                 |
| `resolveReasoningOutputMode`      | Yerel ve etiketli akıl yürütme çıktısı sözleşmesi arasında seçim yapar                                                              | Sağlayıcı, yerel alanlar yerine etiketli akıl yürütme/nihai çıktıya ihtiyaç duyar                                                                         |
| `prepareExtraParams`              | Genel akış seçeneği sarmalayıcılarından önce istek parametresi normalleştirmesi yapar                                              | Sağlayıcı, varsayılan istek parametrelerine veya sağlayıcı başına parametre temizliğine ihtiyaç duyar                                                                           |
| `createStreamFn`                  | Normal akış yolunu özel bir aktarımla tamamen değiştirir                                                   | Sağlayıcı yalnızca bir sarmalayıcıya değil, özel bir kablo protokolüne ihtiyaç duyar                                                                                     |
| `wrapStreamFn`                    | Genel sarmalayıcılar uygulandıktan sonraki akış sarmalayıcısı                                                              | Sağlayıcı, özel bir aktarım olmadan istek üstbilgisi/gövdesi/model uyumluluk sarmalayıcılarına ihtiyaç duyar                                                          |
| `resolveTransportTurnState`       | Her tur için yerel aktarım üstbilgilerini veya meta verileri ekler                                                           | Sağlayıcı, genel aktarımların sağlayıcıya özgü tur kimliğini göndermesini ister                                                                       |
| `resolveWebSocketSessionPolicy`   | Yerel WebSocket üstbilgilerini veya oturum bekleme politikasını ekler                                                    | Sağlayıcı, genel WS aktarımlarının oturum üstbilgilerini veya geri dönüş politikasını ayarlamasını ister                                                               |
| `formatApiKey`                    | Kimlik doğrulama profili biçimlendiricisi: saklanan profil, çalışma zamanı `apiKey` dizesine dönüşür                                     | Sağlayıcı ek kimlik doğrulama meta verileri saklar ve özel bir çalışma zamanı belirteci biçimine ihtiyaç duyar                                                                    |
| `refreshOAuth`                    | Özel yenileme uç noktaları veya yenileme hatası politikası için OAuth yenileme geçersiz kılması                                  | Sağlayıcı, paylaşılan OpenClaw yenileyicilerine uymaz                                                                                          |
| `buildAuthDoctorHint`             | OAuth yenilemesi başarısız olduğunda eklenen onarım ipucu                                                                  | Sağlayıcı, yenileme hatasından sonra sağlayıcıya ait kimlik doğrulama onarım rehberliğine ihtiyaç duyar                                                                      |
| `matchesContextOverflowError`     | Sağlayıcıya ait bağlam penceresi taşması eşleştiricisi                                                                 | Sağlayıcının, genel sezgisel yöntemlerin kaçıracağı ham taşma hataları vardır                                                                                |
| `classifyFailoverReason`          | Sağlayıcıya ait yük devretme nedeni sınıflandırması                                                                  | Sağlayıcı, ham API/aktarım hatalarını hız sınırı/aşırı yük/vb. olarak eşleyebilir                                                                          |
| `isCacheTtlEligible`              | Proxy/arka taşıma sağlayıcıları için istem önbelleği politikası                                                               | Sağlayıcı, proxy'ye özgü önbellek TTL denetimine ihtiyaç duyar                                                                                                |
| `buildMissingAuthMessage`         | Genel eksik kimlik doğrulama kurtarma iletisinin yerine geçer                                                      | Sağlayıcı, sağlayıcıya özgü bir eksik kimlik doğrulama kurtarma ipucuna ihtiyaç duyar                                                                                 |
| `augmentModelCatalog`             | Keşiften sonra eklenen sentetik/nihai katalog satırları (kullanımdan kaldırıldı, aşağıya bakın)                                  | Sağlayıcı, `models list` ve seçicilerde ileriye dönük uyumluluk için sentetik satırlara ihtiyaç duyar                                                                     |
| `resolveThinkingProfile`          | Modele özgü `/think` düzey kümesi, görüntüleme etiketleri ve varsayılan değer                                                 | Sağlayıcı, seçili modeller için özel bir düşünme basamakları dizisi veya ikili etiket sunar                                                                 |
| `isBinaryThinking`                | Açık/kapalı akıl yürütme anahtarı uyumluluk kancası                                                                     | Sağlayıcı yalnızca ikili düşünme açık/kapalı durumunu sunar                                                                                                  |
| `supportsXHighThinking`           | `xhigh` akıl yürütme desteği uyumluluk kancası                                                                   | Sağlayıcı, yalnızca modellerin bir alt kümesinde `xhigh` olmasını ister                                                                                             |
| `resolveDefaultThinkingLevel`     | Varsayılan `/think` düzeyi uyumluluk kancası                                                                      | Sağlayıcı, bir model ailesi için varsayılan `/think` politikasının sahibidir                                                                                      |
| `isModernModelRef`                | Canlı profil filtreleri ve duman testi seçimi için modern model eşleştiricisi                                              | Sağlayıcı, canlı/duman testi için tercih edilen model eşleştirmesinin sahibidir                                                                                             |
| `prepareRuntimeAuth`              | Yapılandırılmış bir kimlik bilgisini çıkarımdan hemen önce gerçek çalışma zamanı belirtecine/anahtarına dönüştürür                       | Sağlayıcı, belirteç değişimine veya kısa ömürlü istek kimlik bilgisine ihtiyaç duyar                                                                             |
| `resolveUsageAuth`                | `/usage` ve ilgili durum yüzeyleri için kullanım/faturalandırma kimlik bilgilerini çözümler                                     | Sağlayıcı, özel kullanım/kota belirteci ayrıştırmasına veya farklı bir kullanım kimlik bilgisine ihtiyaç duyar                                                               |
| `fetchUsageSnapshot`              | Kimlik doğrulama çözümlendikten sonra sağlayıcıya özgü kullanım/kota anlık görüntülerini getirir ve normalleştirir                             | Sağlayıcı, sağlayıcıya özgü bir kullanım uç noktasına veya yük ayrıştırıcısına ihtiyaç duyar                                                                           |
| `createEmbeddingProvider`         | Bellek/arama için sağlayıcıya ait bir gömme adaptörü oluşturun                                                     | Bellek gömme davranışı sağlayıcı Plugin'ine aittir                                                                                    |
| `buildReplayPolicy`               | Sağlayıcı için transkript işlemeyi denetleyen bir yeniden oynatma ilkesi döndürün                                        | Sağlayıcı özel bir transkript ilkesine ihtiyaç duyar (örneğin, düşünme bloklarını kaldırma)                                                               |
| `sanitizeReplayHistory`           | Genel transkript temizliğinden sonra yeniden oynatma geçmişini yeniden yazın                                                        | Sağlayıcı, paylaşılan Compaction yardımcılarının ötesinde sağlayıcıya özgü yeniden oynatma düzenlemelerine ihtiyaç duyar                                                             |
| `validateReplayTurns`             | Gömülü çalıştırıcıdan önce son yeniden oynatma turu doğrulamasını veya yeniden şekillendirmesini gerçekleştirin                                           | Sağlayıcı aktarımı, genel temizlemeden sonra daha sıkı tur doğrulamasına ihtiyaç duyar                                                                    |
| `onModelSelected`                 | Sağlayıcıya ait seçim sonrası yan etkileri çalıştırın                                                                 | Bir model etkinleştiğinde sağlayıcı telemetriye veya sağlayıcıya ait duruma ihtiyaç duyar                                                                  |

`normalizeModelId`, `normalizeTransport` ve `normalizeConfig` önce
eşleşen sağlayıcı Plugin'ini denetler, ardından biri model kimliğini veya
aktarım/yapılandırmayı gerçekten değiştirene kadar kanca özelliğine sahip diğer
sağlayıcı Plugin'lerine geçer. Bu, çağıranın yeniden yazmanın hangi paketlenmiş
Plugin'e ait olduğunu bilmesini gerektirmeden takma ad/uyumluluk sağlayıcı
katmanlarının çalışmasını sürdürür. Hiçbir sağlayıcı kancası desteklenen bir
Google ailesi yapılandırma girdisini yeniden yazmazsa paketlenmiş Google
yapılandırma normalleştiricisi bu uyumluluk temizliğini yine uygular.

Sağlayıcı tamamen özel bir kablo protokolüne veya özel bir istek yürütücüsüne
ihtiyaç duyuyorsa bu, farklı bir uzantı sınıfıdır. Bu kancalar, OpenClaw'ın
normal çıkarım döngüsünde çalışmaya devam eden sağlayıcı davranışı içindir.

`resolveUsageAuth`, OpenClaw'ın `fetchUsageSnapshot` öğesini çağırması mı yoksa
kullanım/durum yüzeyleri için genel kimlik bilgisi çözümlemeye geri dönmesi mi
gerektiğine karar verir. Sağlayıcının bir kullanım kimlik bilgisi olduğunda
`{ token, accountId?, subscriptionType?, rateLimitTier? }` döndürün (isteğe bağlı plan meta verileri
`fetchUsageSnapshot` içine aktarılır), sağlayıcıya ait kullanım kimlik doğrulaması
isteği işlediğinde ve genel API anahtarı/OAuth geri dönüşünü engellemesi
gerektiğinde `{ handled: true }` döndürün; sağlayıcı kullanım kimlik doğrulamasını
işlemediğinde ise `null` veya `undefined` döndürün.

Kuruluş veya faturalandırma kimlik bilgilerini manifest
`providerUsageAuthEnvVars` içinde bildirin. Bu, genel keşif ve gizli bilgi temizleme
yüzeylerinin bunları çıkarım kimlik doğrulaması adayları hâline getirmeden
tanımasını sağlar.

### Sağlayıcı örneği

```ts
api.registerProvider({
  id: "example-proxy",
  label: "Example Proxy",
  auth: [],
  catalog: {
    order: "simple",
    run: async (ctx) => {
      const apiKey = ctx.resolveProviderApiKey("example-proxy").apiKey;
      if (!apiKey) {
        return null;
      }
      return {
        provider: {
          baseUrl: "https://proxy.example.com/v1",
          apiKey,
          api: "openai-completions",
          models: [{ id: "auto", name: "Auto" }],
        },
      };
    },
  },
  resolveDynamicModel: (ctx) => ({
    id: ctx.modelId,
    name: ctx.modelId,
    provider: "example-proxy",
    api: "openai-completions",
    baseUrl: "https://proxy.example.com/v1",
    reasoning: false,
    input: ["text"],
    cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
    contextWindow: 128000,
    maxTokens: 8192,
  }),
  prepareRuntimeAuth: async (ctx) => {
    const exchanged = await exchangeToken(ctx.apiKey);
    return {
      apiKey: exchanged.token,
      baseUrl: exchanged.baseUrl,
      expiresAt: exchanged.expiresAt,
    };
  },
  resolveUsageAuth: async (ctx) => {
    const auth = await ctx.resolveOAuthToken();
    return auth ? { token: auth.token } : null;
  },
  fetchUsageSnapshot: async (ctx) => {
    return await fetchExampleProxyUsage(ctx.token, ctx.timeoutMs, ctx.fetchFn);
  },
});
```

### Yerleşik örnekler

Paketlenmiş sağlayıcı Plugin'leri, her satıcının katalog, kimlik doğrulama,
düşünme, yeniden oynatma ve kullanım gereksinimlerine uyum sağlamak için
yukarıdaki kancaları birleştirir. Yetkili kanca kümesi `extensions/`
altındaki her Plugin ile birlikte bulunur; bu sayfa listeyi yansıtmak yerine
biçimleri gösterir.

<AccordionGroup>
  <Accordion title="Doğrudan geçişli katalog sağlayıcıları">
    OpenRouter, Kilocode, Z.AI ve xAI, üst kaynak model kimliklerini OpenClaw'ın
    statik kataloğundan önce sunabilmek için `catalog` ile birlikte
    `resolveDynamicModel` / `prepareDynamicModel` kaydeder.
  </Accordion>
  <Accordion title="OAuth ve kullanım uç noktası sağlayıcıları">
    GitHub Copilot, Gemini CLI, ChatGPT Codex, MiniMax, Xiaomi ve z.ai, belirteç
    değişimini ve `/usage` entegrasyonunu sahiplenmek için
    `prepareRuntimeAuth` veya `formatApiKey` öğesini `resolveUsageAuth` +
    `fetchUsageSnapshot` ile eşleştirir.
  </Accordion>
  <Accordion title="Yeniden oynatma ve transkript temizleme aileleri">
    Paylaşılan adlandırılmış aileler (`google-gemini`, `passthrough-gemini`,
    `anthropic-by-model`, `hybrid-anthropic-openai`), her Plugin'in temizliği yeniden
    uygulaması yerine sağlayıcıların `buildReplayPolicy` aracılığıyla transkript
    politikasına katılmasını sağlar.
  </Accordion>
  <Accordion title="Yalnızca katalog sağlayıcıları">
    `byteplus`, `cloudflare-ai-gateway`, `huggingface`, `kimi-coding`, `nvidia`,
    `qianfan`, `synthetic`, `together`, `venice`, `vercel-ai-gateway` ve
    `volcengine` yalnızca `catalog` kaydeder ve paylaşılan çıkarım döngüsünü kullanır.
  </Accordion>
  <Accordion title="Anthropic'e özgü akış yardımcıları">
    Beta üst bilgileri, `/fast` / `serviceTier` ve `context1m`,
    genel SDK yerine Anthropic Plugin'inin herkese açık `api.ts` /
    `contract-api.ts` bağlantı noktasında (`wrapAnthropicProviderStream`, `resolveAnthropicBetas`,
    `resolveAnthropicFastMode`, `resolveAnthropicServiceTier`) bulunur.
  </Accordion>
</AccordionGroup>

## Çalışma zamanı yardımcıları

Plugin'ler, `api.runtime` aracılığıyla seçili çekirdek yardımcılarına
erişebilir. TTS için:

```ts
const clip = await api.runtime.tts.textToSpeech({
  text: "OpenClaw'dan merhaba",
  cfg: api.config,
});

const result = await api.runtime.tts.textToSpeechTelephony({
  text: "OpenClaw'dan merhaba",
  cfg: api.config,
});

const voices = await api.runtime.tts.listVoices({
  provider: "elevenlabs",
  cfg: api.config,
});
```

Notlar:

- `textToSpeech`, dosya/sesli not yüzeyleri için normal çekirdek TTS çıktı yükünü döndürür.
- Çekirdek `tts` yapılandırmasını ve sağlayıcı seçimini kullanır.
- PCM ses arabelleği + örnekleme hızı döndürür. Plugin'ler sağlayıcılar için yeniden örneklemeli/kodlamalıdır.
- `listVoices`, sağlayıcı başına isteğe bağlıdır. Satıcıya ait ses seçiciler veya kurulum akışları için kullanın.
- Çekirdek, sağlayıcı `listVoices` kancalarına çözümlenmiş bir istek son tarihi iletir; sağlayıcıya özgü zaman aşımı ayarları bunu geçersiz kılabilir.
- Ses listeleri, sağlayıcıya duyarlı seçiciler için yerel ayar, cinsiyet ve kişilik etiketleri gibi daha zengin meta veriler içerebilir.
- OpenAI ve ElevenLabs şu anda telefonu destekler. Microsoft desteklemez.

Plugin'ler ayrıca `api.registerSpeechProvider(...)` aracılığıyla konuşma sağlayıcıları
kaydedebilir.

```ts
api.registerSpeechProvider({
  id: "acme-speech",
  label: "Acme Speech",
  isConfigured: ({ config }) => Boolean(config.messages?.tts),
  synthesize: async (req) => {
    return {
      audioBuffer: Buffer.from([]),
      outputFormat: "mp3",
      fileExtension: ".mp3",
      voiceCompatible: false,
    };
  },
});
```

Notlar:

- TTS politikasını, geri dönüşü ve yanıt teslimini çekirdekte tutun.
- Satıcıya ait sentez davranışı için konuşma sağlayıcılarını kullanın.
- Eski Microsoft `edge` girdisi, `microsoft` sağlayıcı kimliğine normalleştirilir.
- Tercih edilen sahiplik modeli şirket odaklıdır: OpenClaw bu
  yetenek sözleşmelerini ekledikçe tek bir satıcı Plugin'i metin, konuşma,
  görüntü ve gelecekteki medya sağlayıcılarına sahip olabilir.

Görüntü/ses/video anlama için Plugin'ler, genel bir anahtar/değer paketi yerine
türlü tek bir medya anlama sağlayıcısı kaydeder:

```ts
api.registerMediaUnderstandingProvider({
  id: "google",
  capabilities: ["image", "audio", "video"],
  describeImage: async (req) => ({ text: "..." }),
  transcribeAudio: async (req) => ({ text: "..." }),
  describeVideo: async (req) => ({ text: "..." }),
});
```

Notlar:

- Orkestrasyonu, geri dönüşü, yapılandırmayı ve kanal bağlantılarını çekirdekte tutun.
- Satıcı davranışını sağlayıcı Plugin'inde tutun.
- Eklemeli genişletme türlü kalmalıdır: yeni isteğe bağlı yöntemler, yeni isteğe bağlı
  sonuç alanları, yeni isteğe bağlı yetenekler.
- Video üretimi zaten aynı örüntüyü izler:
  - çekirdek, yetenek sözleşmesine ve çalışma zamanı yardımcısına sahiptir
  - satıcı Plugin'leri `api.registerVideoGenerationProvider(...)` kaydeder
  - özellik/kanal Plugin'leri `api.runtime.videoGeneration.*` tüketir

Medya anlama çalışma zamanı yardımcıları için Plugin'ler şunları çağırabilir:

```ts
const image = await api.runtime.mediaUnderstanding.describeImageFile({
  filePath: "/tmp/inbound-photo.jpg",
  cfg: api.config,
  agentDir: "/tmp/agent",
});

const video = await api.runtime.mediaUnderstanding.describeVideoFile({
  filePath: "/tmp/inbound-video.mp4",
  cfg: api.config,
});

const extraction = await api.runtime.mediaUnderstanding.extractStructuredWithModel({
  provider: "codex",
  model: "gpt-5.6-sol",
  input: [
    {
      type: "image",
      buffer: receiptImageBuffer,
      fileName: "receipt.png",
      mime: "image/png",
    },
    { type: "text", text: "Doğruluk kaynağı olarak basılı alanları kullanın." },
  ],
  instructions: "Varlıkları ve aranabilir etiketleri döndürün.",
  schemaName: "example.evidence",
  jsonSchema: {
    type: "object",
    properties: {
      entities: { type: "array", items: { type: "string" } },
      tags: { type: "array", items: { type: "string" } },
    },
  },
  cfg: api.config,
});
```

Ses transkripsiyonu için Plugin'ler medya anlama çalışma zamanını veya eski STT
takma adını kullanabilir:

```ts
const { text } = await api.runtime.mediaUnderstanding.transcribeAudioFile({
  filePath: "/tmp/inbound-audio.ogg",
  cfg: api.config,
  // MIME güvenilir biçimde çıkarılamadığında isteğe bağlıdır:
  mime: "audio/ogg",
});
```

Notlar:

- `api.runtime.mediaUnderstanding.*`, görüntü/ses/video anlama için tercih edilen
  paylaşılan yüzeydir.
- `extractStructuredWithModel(...)`, sınırlı ve sağlayıcıya ait, öncelikle
  görüntü tabanlı çıkarım için Plugin'e dönük bağlantı noktasıdır. En az bir
  görüntü girdisi ekleyin; metin girdileri tamamlayıcı bağlamdır. Ürün
  Plugin'leri kendi rotalarına ve şemalarına sahipken OpenClaw
  sağlayıcı/çalışma zamanı sınırına sahiptir.
- Çekirdek medya anlama ses yapılandırmasını (`tools.media.audio`) ve sağlayıcı geri dönüş sırasını kullanır.
- Hiçbir transkripsiyon çıktısı üretilmediğinde (örneğin atlanan/desteklenmeyen girdi) `{ text: undefined }` döndürür.

Plugin'ler ayrıca `api.runtime.subagent` aracılığıyla arka planda alt aracı
çalıştırmaları başlatabilir:

```ts
const result = await api.runtime.subagent.run({
  sessionKey: "agent:main:subagent:search-helper",
  message: "Bu sorguyu odaklanmış takip aramalarına genişletin.",
  toolsAlsoAllow: ["my_plugin_progress"],
  provider: "openai",
  model: "gpt-4.1-mini",
  deliver: false,
});
```

Notlar:

- `provider` ve `model`, kalıcı oturum değişiklikleri değil, çalıştırma başına isteğe bağlı geçersiz kılmalardır.
- `toolsAlsoAllow`, çağıran Plugin tarafından kaydedilmiş tam ve benzersiz sahipli araç adlarını kabul eder. Çekirdek ve belirsiz adlar reddedilir. Normal profile eklemelidir ancak operatör izin listeleri ve retleri yetkili olmaya devam eder.
- OpenClaw bu geçersiz kılma alanlarını yalnızca güvenilir çağıranlar için dikkate alır.
- Plugin'e ait geri dönüş çalıştırmaları için operatörlerin `plugins.entries.<id>.subagent.allowModelOverride: true` ile katılım sağlaması gerekir.
- Güvenilir Plugin'leri belirli standart `provider/model` hedefleriyle sınırlamak için `plugins.entries.<id>.subagent.allowedModels`, herhangi bir hedefe açıkça izin vermek için ise `"*"` kullanın.
- Güvenilmeyen Plugin alt aracı çalıştırmaları yine çalışır ancak geçersiz kılma istekleri sessizce geri dönmek yerine reddedilir.
- Plugin tarafından oluşturulan alt aracı oturumları, oluşturan Plugin kimliğiyle etiketlenir. Geri dönüş `api.runtime.subagent.deleteSession(...)` yalnızca sahip olunan bu oturumları silebilir; rastgele oturum silme işlemi yine de yönetici kapsamlı bir Gateway isteği gerektirir.

Web araması için Plugin'ler, aracı araç bağlantılarına erişmek yerine paylaşılan
çalışma zamanı yardımcısını kullanabilir:

```ts
const providers = api.runtime.webSearch.listProviders({
  config: api.config,
});

const result = await api.runtime.webSearch.search({
  config: api.config,
  args: {
    query: "OpenClaw Plugin çalışma zamanı yardımcıları",
    count: 5,
  },
});
```

Plugin'ler ayrıca `api.registerWebSearchProvider(...)` aracılığıyla web araması sağlayıcıları
kaydedebilir.

Notlar:

- Sağlayıcı seçimini, kimlik bilgisi çözümlemesini ve paylaşılan istek semantiğini çekirdekte tutun.
- Satıcıya özgü arama aktarımları için web araması sağlayıcılarını kullanın.
- `api.runtime.webSearch.*`, aracı araç sarmalayıcısına bağımlı olmadan arama davranışına ihtiyaç duyan özellik/kanal Plugin'leri için tercih edilen paylaşılan yüzeydir.

### `api.runtime.imageGeneration`

```ts
const result = await api.runtime.imageGeneration.generate({
  config: api.config,
  args: { prompt: "Dost canlısı bir ıstakoz maskotu", size: "1024x1024" },
});

const providers = api.runtime.imageGeneration.listProviders({
  config: api.config,
});
```

- `generate(...)`: yapılandırılmış görüntü oluşturma sağlayıcı zincirini kullanarak bir görüntü oluşturur.
- `listProviders(...)`: kullanılabilir görüntü oluşturma sağlayıcılarını ve bunların yeteneklerini listeler.

## Gateway HTTP rotaları

Plugin'ler, `api.registerHttpRoute(...)` ile HTTP uç noktaları sunabilir.

```ts
api.registerHttpRoute({
  path: "/acme/webhook",
  auth: "plugin",
  match: "exact",
  handler: async (_req, res) => {
    res.statusCode = 200;
    res.end("ok");
    return true;
  },
});
```

Rota alanları:

- `path`: Gateway HTTP sunucusu altındaki rota yolu.
- `auth`: zorunludur; `"gateway"` veya `"plugin"`. Normal Gateway kimlik doğrulamasını zorunlu kılmak için `"gateway"`, Plugin tarafından yönetilen kimlik doğrulama/Webhook doğrulaması için `"plugin"` kullanın.
- `match`: isteğe bağlıdır. `"exact"` (varsayılan) veya `"prefix"`.
- `handleUpgrade`: aynı rotadaki WebSocket yükseltme istekleri için isteğe bağlı işleyici.
- `replaceExisting`: isteğe bağlıdır. Aynı Plugin'in kendi mevcut rota kaydını değiştirmesine izin verir.
- `handler`: rota isteği işlediğinde `true` döndürün.

Notlar:

- `api.registerHttpHandler(...)` kaldırılmıştır ve Plugin yükleme hatasına neden olur. Bunun yerine `api.registerHttpRoute(...)` kullanın.
- Plugin rotaları `auth` değerini açıkça bildirmelidir.
- Tam `path + match` çakışmaları, `replaceExisting: true` olmadığı sürece reddedilir ve bir Plugin başka bir Plugin'in rotasını değiştiremez.
- Farklı `auth` düzeylerine sahip örtüşen rotalar reddedilir. `exact`/`prefix` zincirlerini yalnızca aynı kimlik doğrulama düzeyinde tutun.
- `auth: "plugin"` rotaları operatör çalışma zamanı kapsamlarını otomatik olarak **almaz**. Bunlar ayrıcalıklı Gateway yardımcı çağrıları için değil, Plugin tarafından yönetilen Webhook'lar/imza doğrulaması içindir.
- `auth: "gateway"` rotaları bir Gateway isteği çalışma zamanı kapsamı içinde çalışır. Varsayılan yüzey (`gatewayRuntimeScopeSurface: "write-default"`) kasıtlı olarak ihtiyatlıdır:
  - paylaşılan gizli anahtar taşıyıcı kimlik doğrulaması (`gateway.auth.mode = "token"` / `"password"`) ve güvenilir proxy dışındaki tüm kimlik doğrulama yöntemleri, çağıran `x-openclaw-scopes` gönderse bile tek bir `operator.write` kapsamı alır
  - açık bir `x-openclaw-scopes` üstbilgisi olmayan `trusted-proxy` çağıranları da eski, yalnızca `operator.write` yüzeyini korur
  - `x-openclaw-scopes` gönderen `trusted-proxy` çağıranları bunun yerine bildirilen kapsamları alır
  - bir rota, kimlik taşıyan kimlik doğrulama modlarında `x-openclaw-scopes` değerini her zaman dikkate almak için `gatewayRuntimeScopeSurface: "trusted-operator"` seçeneğini etkinleştirebilir (üstbilgi yoksa tam CLI varsayılan kapsam kümesine geri döner)
- `auth: "gateway"` rotaları tarafından desteklenen korumalı alan içindeki harici Control UI sekmeleri, yalnızca kimliği doğrulanmış başlangıç işlemi tarafından oluşturulan kısa ömürlü, imzalı bir çerez izni kullanır; Plugin kimlik doğrulamalı sekmeler doğrudan iframe yollarını korur. Üst öğe, bağlamadan önce aynı opak korumalı alan içinde rotanın sahip olduğu bir yoklama çalıştırır ve tarayıcı gizlilik politikası çerezi engellediğinde güvenli biçimde başarısız olur. İzin, sahip olan Plugin'e, eşleşen rota köküne ve geçerli kimlik doğrulama nesline bağlıdır; süreç için rastgele oluşturulan çerez adı, aynı ana bilgisayardaki güvenilir Gateway'lerin birbirlerinin üzerine yazmasını önler ancak çerezler TCP bağlantı noktalarını hiçbir zaman birbirinden yalıtmaz. Bu nedenle Gateway ana bilgisayar adı tek bir kimlik bilgisi sınırıdır: diğer bağlantı noktaları dâhil olmak üzere karşılıklı olarak güvenilmeyen hizmetleri bu ana bilgisayar adında birlikte barındırmayın. Rota gönderimi, başka bir Plugin'e ait iç içe geçmiş bir rotada yeniden kullanımı reddeder. Korumalı alan alt öğeleri çerez amaçları bakımından siteler arası olduğundan izin, yalnızca `operator.read` ile `GET` ve `HEAD` kabul eder; değişiklikler ve WebSocket yükseltmeleri açıkça Gateway kimlik doğrulamalı yüzeylerde kalır. Çerez kasıtlı olarak CHIPS kullanamaz: güncel tarayıcılar bölüm anahtarına bir siteler arası üst öğe biti eklediğinden, iç içe opak korumalı alan çerçeveleri aynı rotadaki varlıklara erişimi kaybeder. Çerez güvenli bir bağlam ve siteler arası çerezler için tarayıcı izni gerektirir; bu nedenle Gateway kimlik doğrulamalı harici sekmeler düz HTTP kullanan LAN kaynaklarında veya üçüncü taraf çerezlerin tamamen engellendiği durumlarda kullanılamaz; HTTPS/Tailscale Serve veya uyumlu bir çerez politikasıyla tarayıcının güvendiği geri döngüyü kullanın.
- İzin, Gateway taşıyıcı belirtecinin açığa çıkmasını ve rotanın/kapsamın yanlışlıkla yeniden kullanılmasını önler; yerel Plugin'ler arasında bir güvenlik sınırı oluşturmaz. Yerel Plugin kodu ve sunduğu UI içeriği, aynı güvenilir işlem içi Plugin sınırının parçası olmaya devam eder.
- Pratik kural: Gateway kimlik doğrulamalı bir Plugin rotasının örtük bir yönetici yüzeyi olduğunu varsaymayın. Rotanız yalnızca yöneticiye özel davranış gerektiriyorsa `trusted-operator` kapsam yüzeyini etkinleştirin, kimlik taşıyan bir kimlik doğrulama modunu zorunlu kılın ve açık `x-openclaw-scopes` üstbilgi sözleşmesini belgeleyin.
- Rota eşleştirmesi ve kimlik doğrulamasından sonra sıradan işleyiciler Gateway kök iş kabulüne katılır. Hazırlanmakta veya yeniden başlatılmakta olan bir Gateway, işleyiciyi çağırmadan önce `503` döndürür. Dar kapsamlı istisna, rota özelindeki `trusted-operator` yüzeyini de etkinleştiren ve manifest tarafından yetkilendirilmiş bir `auth: "gateway"` rotasıdır; askıya alma denetimi gönderiminin erişilemez hâle gelmemesi için rota erişilebilir kalırken aynı Plugin'in sıradan kardeş rotaları kabul sınırının arkasında kalır. WebSocket `handleUpgrade` sahipliği aynı atomik kabul sınırını kullanır; işleyici bir soketi kabul ettikten sonra soketin sonraki yaşam döngüsünün sahibi Plugin olur ve bu sınır tarafından izlenmez.

## Plugin SDK içe aktarma yolları

Yeni Plugin'ler yazarken tek parça `openclaw/plugin-sdk` kök
barrel'ı yerine dar SDK alt yollarını kullanın. Temel alt yollar:

| Alt yol                            | Amaç                                         |
| ---------------------------------- | -------------------------------------------- |
| `openclaw/plugin-sdk/plugin-entry` | Plugin kayıt ilkelleri                       |
| `openclaw/plugin-sdk/channel-core` | Kanal giriş/oluşturma yardımcıları           |
| `openclaw/plugin-sdk/core`         | Genel paylaşılan yardımcılar ve şemsiye sözleşme |

Kanal Plugin'leri dar bağlantı noktaları ailesinden seçim yapar: `channel-setup`,
`setup-runtime`, `setup-tools`, `channel-pairing`,
`channel-contract`, `channel-feedback`, `channel-inbound`, `channel-outbound`,
`command-auth`, `secret-input`, `webhook-ingress`,
`channel-targets` ve `channel-actions`. Onay davranışı, ilgisiz
Plugin alanları arasında karıştırılmak yerine tek bir `approvalCapability`
sözleşmesinde birleştirilmelidir. Bkz. [Kanal Plugin'leri](/tr/plugins/sdk-channel-plugins).

Çalışma zamanı ve yapılandırma yardımcıları, eşleşen odaklanmış `*-runtime` alt yolları
altında bulunur (`approval-runtime`, `agent-runtime`, `lazy-runtime`, `directory-runtime`,
`text-runtime`, `runtime-store`, `system-event-runtime`, `heartbeat-runtime`,
`channel-activity-runtime` vb.). Geniş `config-runtime` uyumluluk barrel'ı
yerine `config-contracts`, `plugin-config-runtime`, `runtime-config-snapshot`
ve `config-mutation` tercih edin.

<Info>
`openclaw/plugin-sdk/channel-lifecycle`, küçük kanal yardımcı cepheleri,
`openclaw/plugin-sdk/config-runtime` ve `openclaw/plugin-sdk/infra-runtime`,
eski Plugin'ler için kullanımdan kaldırılmış uyumluluk katmanlarıdır. Yeni kod bunun yerine
daha dar genel ilkelleri içe aktarmalıdır.
</Info>

Depo içi giriş noktaları (paketle gelen her Plugin paket kökü için):

- `index.js` — paketle gelen Plugin girişi
- `api.js` — yardımcı/tür barrel'ı
- `runtime-api.js` — yalnızca çalışma zamanı barrel'ı
- `setup-entry.js` — kurulum Plugin girişi

Harici Plugin'ler yalnızca `openclaw/plugin-sdk/*` alt yollarını içe aktarmalıdır. Çekirdekten
veya başka bir Plugin'den, başka bir Plugin paketinin `src/*` ögesini hiçbir zaman içe aktarmayın.
Cephe üzerinden yüklenen giriş noktaları, varsa etkin çalışma zamanı yapılandırma anlık görüntüsünü
tercih eder, ardından diskteki çözümlenmiş yapılandırma dosyasına geri döner.

`image-generation`, `media-understanding`
ve `speech` gibi yeteneğe özel alt yollar, paketle gelen Plugin'ler bunları bugün kullandığı için vardır. Bunlar
otomatik olarak uzun vadeli, değişmez harici sözleşmeler değildir; bunlara güvenirken ilgili SDK
başvuru sayfasını kontrol edin.

## İleti aracı şemaları

Plugin'ler; tepkiler, okumalar ve anketler gibi ileti dışı ilkeller için kanala özel
`describeMessageTool(...)` şema katkılarının sahibi olmalıdır.
Paylaşılan gönderim sunumu, sağlayıcıya özgü düğme, bileşen, blok veya kart alanları
yerine genel `MessagePresentation` sözleşmesini kullanmalıdır.
Sözleşme, geri dönüş kuralları, sağlayıcı eşlemesi ve Plugin yazarı kontrol listesi için
bkz. [İleti Sunumu](/tr/plugins/message-presentation).

Gönderim yapabilen Plugin'ler, ileti yetenekleri aracılığıyla neleri işleyebileceklerini bildirir:

- anlamsal sunum blokları için `presentation` (`text`, `context`,
  `divider`, `chart`, `table`, `buttons`, `select`)
- sabitlenmiş teslimat istekleri için `delivery-pin`

Çekirdek, sunumun yerel olarak işlenmesine veya metne indirgenmesine karar verir.
Genel ileti aracından sağlayıcıya özgü UI kaçış yolları sunmayın.
Eski yerel şemalara yönelik kullanımdan kaldırılmış SDK yardımcıları, mevcut
üçüncü taraf Plugin'ler için dışa aktarılmaya devam eder ancak yeni Plugin'ler bunları kullanmamalıdır.

## Kanal hedefi çözümleme

Kanal Plugin'leri kanala özel hedef anlamlarının sahibi olmalıdır. Paylaşılan
giden ana bilgisayarı genel tutun ve sağlayıcı kuralları için mesajlaşma bağdaştırıcısı yüzeyini kullanın:

- `messaging.inferTargetChatType({ to })`, normalleştirilmiş bir hedefin
  dizin aramasından önce `direct`, `group` veya `channel` olarak değerlendirilip değerlendirilmeyeceğine karar verir.
- `messaging.targetResolver.looksLikeId(raw, normalized)`, bir girdinin
  dizin araması yerine doğrudan kimlik benzeri çözümlemeye geçip geçmemesi gerektiğini çekirdeğe bildirir.
- `messaging.targetResolver.reservedLiterals`, söz konusu sağlayıcı için
  kanal/oturum başvurusu olan yalın sözcükleri listeler. Çözümleme, ayrılmış sabit değerleri reddetmeden önce yapılandırılmış
  dizin girdilerini korur, ardından bir dizin eşleşmesi bulunamazsa güvenli biçimde başarısız olur.
- `messaging.targetResolver.resolveTarget(...)`, normalleştirmeden veya bir
  dizin eşleşmesi bulunamadıktan sonra çekirdeğin sağlayıcının sahip olduğu son bir çözümlemeye ihtiyacı olduğunda Plugin geri dönüşüdür.
- `messaging.resolveOutboundSessionRoute(...)`, bir hedef çözümlendikten sonra sağlayıcıya özel oturum
  rotası oluşturmanın sahibidir.

Önerilen ayrım:

- Eşleri/grupları aramadan önce gerçekleşmesi gereken kategori kararları için `inferTargetChatType` kullanın.
- “Bunu açık/yerel bir hedef kimliği olarak değerlendir” kontrolleri için `looksLikeId` kullanın.
- Geniş dizin araması için değil, sağlayıcıya özel normalleştirme geri dönüşü için `resolveTarget` kullanın.
- Sohbet kimlikleri, ileti dizisi kimlikleri, JID'ler, tanıtıcılar ve oda
  kimlikleri gibi sağlayıcıya özgü kimlikleri genel SDK alanlarında değil, `target` değerlerinde veya sağlayıcıya özel parametrelerde tutun.

## Yapılandırma destekli dizinler

Yapılandırmadan dizin girdileri türeten Plugin'ler bu mantığı
Plugin içinde tutmalı ve `openclaw/plugin-sdk/directory-runtime`
içindeki paylaşılan yardımcıları yeniden kullanmalıdır.

Bir kanal aşağıdakiler gibi yapılandırma destekli eşlere/gruplara ihtiyaç duyduğunda bunu kullanın:

- izin listesi tarafından yönlendirilen DM eşleri
- yapılandırılmış kanal/grup eşlemeleri
- hesap kapsamlı statik dizin geri dönüşleri

`directory-runtime` içindeki paylaşılan yardımcılar yalnızca genel işlemleri gerçekleştirir:

- sorgu filtreleme
- sınır uygulama
- yinelenenleri kaldırma/normalleştirme yardımcıları
- `ChannelDirectoryEntry[]` oluşturma

Kanala özel hesap incelemesi ve kimlik normalleştirmesi
Plugin uygulamasında kalmalıdır.

## Sağlayıcı katalogları

Sağlayıcı Plugin'leri, `registerProvider({ catalog: { run(...) { ... } } })` ile çıkarım için
model katalogları tanımlayabilir.

`catalog.run(...)`, OpenClaw'ın `models.providers`
içine yazdığı biçimin aynısını döndürür:

- `{ provider }` tek bir sağlayıcı girdisi için
- `{ providers }` birden çok sağlayıcı girdisi için

Plugin sağlayıcıya özgü model kimliklerinin, temel URL varsayılanlarının
veya kimlik doğrulama koşullu model meta verilerinin sahibiyse `catalog` kullanın.

`catalog.order`, bir Plugin kataloğunun OpenClaw'ın yerleşik örtük
sağlayıcılarına göre ne zaman birleştirileceğini denetler:

- `simple`: düz API anahtarlı veya ortam tarafından yönlendirilen sağlayıcılar
- `profile`: kimlik doğrulama profilleri mevcut olduğunda görünen sağlayıcılar
- `paired`: birbiriyle ilişkili birden çok sağlayıcı girdisi sentezleyen sağlayıcılar
- `late`: diğer örtük sağlayıcılardan sonraki son geçiş

Anahtar çakışmasında sonraki sağlayıcılar kazanır; böylece Plugin'ler aynı
sağlayıcı kimliğine sahip yerleşik bir sağlayıcı girdisini kasıtlı olarak geçersiz kılabilir.

Plugin'ler ayrıca `api.registerModelCatalogProvider({ provider, kinds, staticCatalog, liveCatalog
})` aracılığıyla salt okunur model
satırları yayımlayabilir. Bu, liste/yardım/seçici yüzeyleri için ileriye dönük yoldur ve
`text`, `voice`, `image_generation`, `video_generation` ve `music_generation`
satırlarını destekler. Sağlayıcı Plugin'leri canlı uç nokta çağrılarının, token değişiminin ve
satıcı yanıtı eşlemesinin sahibi olmaya devam eder; ortak satır şeklinin, kaynak etiketlerinin ve
medya aracı yardım biçimlendirmesinin sahibi çekirdektir. Medya üretimi sağlayıcı kayıtları,
`defaultModel`, `models` ve `capabilities` üzerinden statik katalog
satırlarını otomatik olarak sentezler.

Uyumluluk:

- `discovery` eski bir diğer ad olarak çalışmaya devam eder ancak kullanımdan kaldırma uyarısı verir
- hem `catalog` hem de `discovery` kaydedilirse OpenClaw, `catalog`
  kullanır ve bir uyarı verir
- `augmentModelCatalog` kullanımdan kaldırılmıştır; paketlenmiş sağlayıcılar ek
  satırları `registerModelCatalogProvider` aracılığıyla yayımlamalıdır

## Salt okunur kanal incelemesi

Plugin'iniz bir kanal kaydediyorsa `resolveAccount(...)` ile birlikte
`plugin.config.inspectAccount(cfg, accountId)` uygulamayı tercih edin.

Nedeni:

- `resolveAccount(...)` çalışma zamanı yoludur. Kimlik bilgilerinin tamamen
  oluşturulduğunu varsayabilir ve gerekli gizli bilgiler eksik olduğunda hemen başarısız olabilir.
- `openclaw status`, `openclaw status --all`,
  `openclaw channels status`, `openclaw channels resolve` gibi salt okunur komut yollarının ve doctor/yapılandırma
  onarım akışlarının yalnızca yapılandırmayı açıklamak için çalışma zamanı kimlik bilgilerini
  oluşturması gerekmemelidir.

Önerilen `inspectAccount(...)` davranışı:

- Yalnızca açıklayıcı hesap durumunu döndürün.
- `enabled` ve `configured` değerlerini koruyun.
- Uygun olduğunda aşağıdakiler gibi kimlik bilgisi kaynağı/durum alanlarını ekleyin:
  - `tokenSource`, `tokenStatus`
  - `botTokenSource`, `botTokenStatus`
  - `appTokenSource`, `appTokenStatus`
  - `signingSecretSource`, `signingSecretStatus`
- Yalnızca salt okunur kullanılabilirliği bildirmek için ham token
  değerlerini döndürmeniz gerekmez. Durum tarzı komutlar için `tokenStatus: "available"`
  (ve eşleşen kaynak alanını) döndürmek yeterlidir.
- Bir kimlik bilgisi SecretRef aracılığıyla yapılandırılmış ancak geçerli
  komut yolunda kullanılamıyorsa `configured_unavailable` kullanın.

Bu, salt okunur komutların çökmesi veya hesabı yapılandırılmamış olarak yanlış
bildirmesi yerine "yapılandırılmış ancak bu komut yolunda kullanılamıyor" bildirmesini sağlar.

## Paket paketleri

Bir Plugin dizini, `openclaw.extensions` içeren bir `package.json` barındırabilir:

```json
{
  "name": "my-pack",
  "openclaw": {
    "extensions": ["./src/safety.ts", "./src/tools.ts"],
    "setupEntry": "./src/setup-entry.ts"
  }
}
```

Her girdi bir Plugin'e dönüşür. Paket birden çok uzantı listeliyorsa Plugin
kimliği `<manifestOrPackageName>/<fileBase>` olur (mevcutsa manifest kimliği
önceliklidir; aksi takdirde kapsamlandırılmamış `package.json` adı kullanılır).

Plugin'iniz npm bağımlılıklarını içe aktarıyorsa `node_modules`
kullanılabilir olacak şekilde bunları bu dizine yükleyin (`npm install` / `pnpm install`).

Güvenlik koruması: Her `openclaw.extensions` girdisi, sembolik bağlantı
çözümlemesinden sonra Plugin dizininin içinde kalmalıdır. Paket dizininin dışına
çıkan girdiler reddedilir.

Güvenlik notu: `openclaw plugins install`, devralınan genel npm yükleme ayarlarını
yok sayarak Plugin bağımlılıklarını proje yerelinde bir `npm install --omit=dev --ignore-scripts` ile
yükler (yaşam döngüsü betikleri yoktur, çalışma zamanında geliştirme bağımlılıkları yoktur).
Plugin bağımlılık ağaçlarını "saf JS/TS" olarak tutun ve
`postinstall` derlemeleri gerektiren paketlerden kaçının.

İsteğe bağlı: `openclaw.setupEntry`, yalnızca kurulum için kullanılan hafif bir modülü gösterebilir.
OpenClaw devre dışı bırakılmış bir kanal Plugin'i için kurulum yüzeylerine ihtiyaç duyduğunda
veya bir kanal Plugin'i etkin ancak hâlâ yapılandırılmamış olduğunda tam Plugin girdisi yerine
`setupEntry` yükler. Bu, ana Plugin girdiniz araçları, kancaları veya yalnızca çalışma
zamanına ait başka kodları da bağladığında başlangıcı ve kurulumu daha hafif tutar.

İsteğe bağlı: `openclaw.startup.deferConfiguredChannelFullLoadUntilAfterListen`, kanal zaten yapılandırılmış olsa bile bir kanal
Plugin'ini Gateway'in dinleme öncesi başlangıç aşamasında aynı `setupEntry`
yoluna dahil edebilir.

Bunu yalnızca `setupEntry`, Gateway dinlemeye başlamadan önce mevcut olması gereken
başlangıç yüzeyini tamamen kapsıyorsa kullanın. Pratikte bu, kurulum girdisinin başlangıcın
bağımlı olduğu kanala ait her yeteneği kaydetmesi gerektiği anlamına gelir; örneğin:

- kanal kaydının kendisi
- Gateway dinlemeye başlamadan önce kullanılabilir olması gereken tüm HTTP rotaları
- aynı zaman aralığında mevcut olması gereken tüm Gateway yöntemleri, araçları veya hizmetleri

Tam girdiniz gerekli herhangi bir başlangıç yeteneğinin hâlâ sahibiyse bu
bayrağı etkinleştirmeyin. Plugin'i varsayılan davranışta tutun ve OpenClaw'ın
başlangıç sırasında tam girdiyi yüklemesine izin verin.

Paketlenmiş kanallar da tam kanal çalışma zamanı yüklenmeden önce çekirdeğin
başvurabileceği, yalnızca kuruluma yönelik sözleşme yüzeyi yardımcıları yayımlayabilir.
Geçerli kurulum yükseltme yüzeyi şudur:

- `singleAccountKeysToMove`
- `namedAccountPromotionKeys`
- `resolveSingleAccountPromotionTarget(...)`

Çekirdek, tam Plugin girdisini yüklemeden eski bir tek hesaplı kanal
yapılandırmasını `channels.<id>.accounts.*` biçimine yükseltmesi gerektiğinde bu yüzeyi kullanır.
Matrix, mevcut paketlenmiş örnektir: Adlandırılmış hesaplar zaten mevcut olduğunda yalnızca
kimlik doğrulama/önyükleme anahtarlarını adlandırılmış ve yükseltilmiş bir hesaba taşır;
ayrıca her zaman `accounts.default` oluşturmak yerine yapılandırılmış, standart dışı bir
varsayılan hesap anahtarını koruyabilir.

Bu kurulum yama bağdaştırıcıları, paketlenmiş sözleşme yüzeyi keşfini tembel tutar.
İçe aktarma süresi hafif kalır; yükseltme yüzeyi, modül içe aktarılırken paketlenmiş kanal
başlangıcına yeniden girmek yerine yalnızca ilk kullanımda yüklenir.

Bu başlangıç yüzeyleri Gateway RPC yöntemlerini içerdiğinde bunları Plugin'e özgü bir
önek altında tutun. Çekirdek yönetim ad alanları (`config.*`,
`exec.approvals.*`, `wizard.*`, `update.*`) ayrılmış olarak kalır ve bir Plugin
daha dar bir kapsam istese bile her zaman `operator.admin` olarak çözümlenir.

Örnek:

```json
{
  "name": "@scope/my-channel",
  "openclaw": {
    "extensions": ["./index.ts"],
    "setupEntry": "./setup-entry.ts",
    "startup": {
      "deferConfiguredChannelFullLoadUntilAfterListen": true
    }
  }
}
```

### Kanal kataloğu meta verileri

Kanal Plugin'leri `openclaw.channel` aracılığıyla kurulum/keşif meta verilerini,
`openclaw.install` aracılığıyla da yükleme ipuçlarını duyurabilir. Bu, çekirdek
kataloğunu veriden bağımsız tutar.

Örnek:

```json
{
  "name": "@openclaw/nextcloud-talk",
  "openclaw": {
    "extensions": ["./index.ts"],
    "channel": {
      "id": "nextcloud-talk",
      "label": "Nextcloud Talk",
      "selectionLabel": "Nextcloud Talk (kendi sunucunuzda barındırılan)",
      "docsPath": "/channels/nextcloud-talk",
      "docsLabel": "nextcloud-talk",
      "blurb": "Nextcloud Talk Webhook botları aracılığıyla kendi sunucunuzda barındırılan sohbet.",
      "order": 65,
      "aliases": ["nc-talk", "nc"]
    },
    "install": {
      "npmSpec": "@openclaw/nextcloud-talk",
      "localPath": "<bundled-plugin-local-path>",
      "defaultChoice": "npm"
    }
  }
}
```

Asgari örneğin ötesindeki yararlı `openclaw.channel` alanları:

- `detailLabel`: daha zengin katalog/durum yüzeyleri için ikincil etiket
- `docsLabel`: doküman bağlantısının bağlantı metnini geçersiz kılma
- `preferOver`: bu katalog girdisinin geride bırakması gereken daha düşük öncelikli Plugin/kanal kimlikleri
- `selectionDocsPrefix`, `selectionDocsOmitLabel`, `selectionExtras`: seçim yüzeyi metin denetimleri
- `markdownCapable`: giden biçimlendirme kararları için kanalı Markdown özellikli olarak işaretler
- `exposure.configured`: `false` olarak ayarlandığında kanalı yapılandırılmış kanal listeleme yüzeylerinden gizler
- `exposure.setup`: `false` olarak ayarlandığında kanalı etkileşimli kurulum/yapılandırma seçicilerinden gizler
- `exposure.docs`: doküman gezinme yüzeyleri için kanalı dahili/özel olarak işaretler
- `quickstartAllowFrom`: kanalı standart hızlı başlangıç `allowFrom` akışına dahil eder
- `forceAccountBinding`: yalnızca bir hesap mevcut olsa bile açık hesap bağlaması gerektirir
- `preferSessionLookupForAnnounceTarget`: duyuru hedeflerini çözümlerken oturum aramasını tercih eder

OpenClaw ayrıca **harici kanal kataloglarını** (örneğin bir MPM kayıt
dışa aktarımını) birleştirebilir. Aşağıdakilerden birine bir JSON dosyası bırakın:

- `~/.openclaw/mpm/plugins.json`
- `~/.openclaw/mpm/catalog.json`
- `~/.openclaw/plugins/catalog.json`

Alternatif olarak `OPENCLAW_PLUGIN_CATALOG_PATHS` (veya `OPENCLAW_MPM_CATALOG_PATHS`) öğesini
bir veya daha fazla JSON dosyasına yönlendirin (virgül/noktalı virgül/`PATH` ile ayrılmış).
Her dosya `{ "entries": [ { "name": "@scope/pkg", "openclaw": { "channel": {...}, "install": {...} } } ] }` içermelidir. Ayrıştırıcı ayrıca `"entries"` anahtarının
eski diğer adları olarak `"packages"` veya `"plugins"` kabul eder.

Oluşturulan kanal kataloğu girdileri ve sağlayıcı yükleme kataloğu girdileri,
ham `openclaw.install` bloğunun yanında normalleştirilmiş yükleme kaynağı bilgilerini sunar.
Normalleştirilmiş bilgiler npm belirtiminin kesin bir sürüm mü yoksa değişken bir seçici mi
olduğunu, beklenen bütünlük meta verilerinin bulunup bulunmadığını ve yerel bir kaynak yolunun da
kullanılabilir olup olmadığını belirtir. Katalog/paket kimliği biliniyorsa normalleştirilmiş bilgiler,
ayrıştırılan npm paket adı bu kimlikten saparsa uyarır. Ayrıca `defaultChoice` geçersizse veya
kullanılamayan bir kaynağı gösteriyorsa ve geçerli bir npm kaynağı olmadan npm bütünlük meta verileri
mevcutsa uyarır. Tüketiciler, elle oluşturulmuş girdilerin ve katalog uyumluluk katmanlarının bunu
sentezlemesi gerekmemesi için `installSource` alanını eklenebilir, isteğe bağlı bir alan olarak
ele almalıdır.
Bu, ilk kullanım ve tanılama süreçlerinin Plugin çalışma zamanını içe aktarmadan
kaynak düzlemi durumunu açıklamasını sağlar.

Resmî harici npm girdileri, kesin bir `npmSpec` ile birlikte
`expectedIntegrity` kullanmayı tercih etmelidir. Yalın paket adları ve dist-tag'ler uyumluluk
için çalışmaya devam eder ancak kataloğun mevcut Plugin'leri bozmadan sabitlenmiş, bütünlüğü
denetlenmiş yüklemelere geçebilmesi için kaynak düzlemi uyarıları gösterir.
İlk kullanım yerel bir katalog yolundan yükleme yaptığında `source: "path"` ve mümkün olduğunda
çalışma alanına göreli bir `sourcePath` içeren yönetilen bir Plugin Plugin dizini girdisi
kaydeder. Mutlak operasyonel yükleme yolu `plugins.load.paths` içinde kalır; yükleme kaydı yerel
iş istasyonu yollarını uzun ömürlü yapılandırmaya çoğaltmaz. Bu, yerel geliştirme yüklemelerini
ikinci bir ham dosya sistemi yolu ifşa yüzeyi eklemeden kaynak düzlemi tanılamasında görünür tutar.
Kalıcı `installed_plugin_index` SQLite tablosu, yükleme için tek doğruluk kaynağıdır ve Plugin çalışma
zamanı modülleri yüklenmeden yenilenebilir. Bir Plugin manifesti eksik veya geçersiz olsa bile
`installRecords` eşlemesi kalıcıdır; `plugins` yükü ise yeniden oluşturulabilir bir
manifest görünümüdür.

## Bağlam motoru Plugin'leri

Bağlam motoru Plugin'leri; alım, birleştirme ve Compaction için oturum bağlamı
orkestrasyonunun sahibidir. Bunları Plugin'inizden `api.registerContextEngine(id, factory)` ile
kaydedin, ardından etkin motoru `plugins.slots.contextEngine` ile seçin.

Plugin'inizin yalnızca bellek araması veya kancalar eklemek yerine varsayılan bağlam
işlem hattını değiştirmesi ya da genişletmesi gerektiğinde bunu kullanın.

```ts
import { buildMemorySystemPromptAddition } from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("lossless-claw", (ctx) => ({
    info: { id: "lossless-claw", name: "Lossless Claw", ownsCompaction: true },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact() {
      return { ok: true, compacted: false };
    },
  }));
}
```

Fabrika `ctx`, oluşturma zamanında başlatma için isteğe bağlı `config`, `agentDir` ve `workspaceDir`
değerlerini kullanıma sunar.

Ana makine, eski olmayan bir motorun `assemble()` çağrısını yapmadan önce kayıtlı eşzamansız bellek istemi hazırlığını tamamlar. `buildMemorySystemPromptAddition(...)`
eşzamanlı kalır ve `assemble()` etkinken bu değişmez çalıştırma anlık görüntüsünü okur.
Sağlanan araç ve alıntı bağlamını değiştirmeden aktarın; böylece anlık görüntü
çalıştırma sınırlarını aşamaz.

Etkin test düzeneğinin kalıcı bir arka uç iş parçacığı olduğunda
`assemble()`, `contextProjection` döndürebilir. Eski tur başına yansıtma için bunu atlayın.
Birleştirilmiş bağlamın bir arka uç iş parçacığına bir kez eklenmesi ve dönem
değişene kadar yeniden kullanılması gerektiğinde `{ mode: "thread_bootstrap", epoch }` döndürün. Motorun anlamsal bağlamı değiştikten sonra,
örneğin motorun sahip olduğu bir Compaction geçişinden sonra dönemi değiştirin.
Ana makineler, yeni arka uç iş parçacıklarının ham, gizli bilgi içeren
yükleri kopyalamadan araç sürekliliğini koruması için iş parçacığı önyükleme yansıtmasında araç çağrısı meta verilerini, girdi
şeklini ve sansürlenmiş araç sonuçlarını koruyabilir.

Motorunuz Compaction algoritmasının sahibi **değilse**, `compact()`
uygulanmış durumda tutun ve açıkça devredin:

```ts
import {
  buildMemorySystemPromptAddition,
  delegateCompactionToRuntime,
} from "openclaw/plugin-sdk/core";

export default function (api) {
  api.registerContextEngine("my-memory-engine", (ctx) => ({
    info: {
      id: "my-memory-engine",
      name: "My Memory Engine",
      ownsCompaction: false,
    },
    async ingest() {
      return { ingested: true };
    },
    async assemble({ messages, sessionKey, availableTools, citationsMode }) {
      return {
        messages,
        estimatedTokens: 0,
        systemPromptAddition: buildMemorySystemPromptAddition({
          availableTools: availableTools ?? new Set(),
          citationsMode,
          agentSessionKey: sessionKey,
        }),
      };
    },
    async compact(params) {
      return await delegateCompactionToRuntime(params);
    },
  }));
}
```

## Yeni bir yetenek ekleme

Bir Plugin, mevcut API'ye uymayan bir davranışa ihtiyaç duyduğunda özel bir
iç erişimle Plugin sistemini atlamayın. Eksik yeteneği ekleyin.

Önerilen sıra:

1. **Çekirdek sözleşmesini tanımlayın.** Çekirdeğin hangi paylaşılan davranışa sahip olması gerektiğine karar verin:
   ilke, geri dönüş, yapılandırma birleştirme, yaşam döngüsü, kanal odaklı anlamlar ve
   çalışma zamanı yardımcısının şekli.
2. **Türü belirlenmiş Plugin kayıt/çalışma zamanı yüzeylerini ekleyin.**
   En küçük kullanışlı türü belirlenmiş yetenek yüzeyiyle
   `OpenClawPluginApi` ve/veya `api.runtime` öğesini genişletin.
3. **Çekirdek ile kanal/özellik tüketicilerini bağlayın.** Kanallar ve özellik Pluginleri,
   bir sağlayıcı uygulamasını doğrudan içe aktarmak yerine yeni yeteneği
   çekirdek üzerinden tüketmelidir.
4. **Sağlayıcı uygulamalarını kaydedin.** Ardından sağlayıcı Pluginleri kendi
   arka uçlarını yeteneğe kaydeder.
5. **Sözleşme kapsamı ekleyin.** Sahiplik ve kayıt şeklinin
   zaman içinde açık kalması için testler ekleyin.

OpenClaw, tek bir sağlayıcının dünya görüşüne sabit kodlanmadan bu şekilde
belirgin tercihlere sahip kalır. Somut bir dosya kontrol listesi ve işlenmiş bir örnek için
[Yetenek Tarifleri](/tr/plugins/adding-capabilities) bölümüne bakın.

### Yetenek kontrol listesi

Yeni bir yetenek eklediğinizde uygulama genellikle aşağıdaki
yüzeylere birlikte dokunmalıdır:

- `src/<capability>/types.ts` içindeki çekirdek sözleşme türleri
- `src/<capability>/runtime.ts` içindeki çekirdek yürütücüsü/çalışma zamanı yardımcısı
- `src/plugins/types.ts` içindeki Plugin API kayıt yüzeyi
- `src/plugins/registry.ts` içindeki Plugin kayıt defteri bağlantıları
- özellik/kanal Pluginlerinin bunu tüketmesi gerektiğinde `src/plugins/runtime/*`
  içindeki Plugin çalışma zamanı sunumu
- `src/test-utils/plugin-registration.ts` içindeki yakalama/test yardımcıları
- `src/plugins/contracts/registry.ts` içindeki sahiplik/sözleşme doğrulamaları
- `docs/` içindeki operatör/Plugin belgeleri

Bu yüzeylerden biri eksikse bu genellikle yeteneğin henüz
tam olarak entegre edilmediğinin işaretidir.

### Yetenek şablonu

Asgari kalıp:

```ts
// core contract
export type VideoGenerationProviderPlugin = {
  id: string;
  label: string;
  generateVideo: (req: VideoGenerationRequest) => Promise<VideoGenerationResult>;
};

// plugin API
api.registerVideoGenerationProvider({
  id: "openai",
  label: "OpenAI",
  async generateVideo(req) {
    return await generateOpenAiVideo(req);
  },
});

// shared runtime helper for feature/channel plugins
const clip = await api.runtime.videoGeneration.generate({
  prompt: "Show the robot walking through the lab.",
  cfg,
});
```

Sözleşme testi kalıbı (`src/plugins/contracts/registry.ts`, `providerContractPluginIds` gibi sahiplik
aramalarını kullanıma sunar; testler bir Pluginin
`contracts.videoGenerationProviders` listesinin gerçekten kaydettiği öğelerle eşleştiğini doğrular):

```ts
expect(pluginManifest.contracts?.videoGenerationProviders).toEqual(["openai"]);
```

Bu, kuralı basit tutar:

- çekirdek, yetenek sözleşmesinin ve orkestrasyonun sahibidir
- sağlayıcı Pluginleri, sağlayıcı uygulamalarının sahibidir
- özellik/kanal Pluginleri çalışma zamanı yardımcılarını tüketir
- sözleşme testleri sahipliği açık tutar

## İlgili

- [Plugin mimarisi](/tr/plugins/architecture) — genel yetenek modeli ve şekilleri
- [Plugin SDK alt yolları](/tr/plugins/sdk-subpaths)
- [Plugin SDK kurulumu](/tr/plugins/sdk-setup)
- [Plugin oluşturma](/tr/plugins/building-plugins)
