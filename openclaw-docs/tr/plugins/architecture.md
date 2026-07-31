---
read_when:
    - Yerel OpenClaw pluginlerini geliştirme veya hata ayıklama
    - Plugin yetenek modelini veya sahiplik sınırlarını anlama
    - Plugin yükleme işlem hattı veya kayıt defteri üzerinde çalışma
    - Sağlayıcı çalışma zamanı kancalarını veya kanal pluginlerini uygulama
sidebarTitle: Internals
summary: 'Plugin iç işleyişi: yetenek modeli, sahiplik, sözleşmeler, yükleme işlem hattı ve çalışma zamanı yardımcıları'
title: Plugin iç yapısı
x-i18n:
    generated_at: "2026-07-26T22:52:13Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d47551b1bc2f71ce2ade3dfdd14bff8ee187616c3807f8101c1a3236e1443cc1
    source_path: plugins/architecture.md
    workflow: 16
---

Bu, OpenClaw plugin sistemi için **ayrıntılı mimari başvuru kaynağıdır**. Uygulamalı kılavuzlar için aşağıdaki odaklanmış sayfalardan biriyle başlayın.

<CardGroup cols={2}>
  <Card title="Pluginleri yükleme ve kullanma" icon="plug" href="/tr/tools/plugin">
    Plugin ekleme, etkinleştirme ve sorunlarını giderme konusunda son kullanıcı kılavuzu.
  </Card>
  <Card title="Plugin geliştirme" icon="rocket" href="/tr/plugins/building-plugins">
    Çalışan en küçük manifesti içeren ilk Plugin öğreticisi.
  </Card>
  <Card title="Kanal pluginleri" icon="comments" href="/tr/plugins/sdk-channel-plugins">
    Bir mesajlaşma kanalı Plugini geliştirin.
  </Card>
  <Card title="Sağlayıcı pluginleri" icon="microchip" href="/tr/plugins/sdk-provider-plugins">
    Bir model sağlayıcı Plugini geliştirin.
  </Card>
  <Card title="SDK'ye genel bakış" icon="book" href="/tr/plugins/sdk-overview">
    İçe aktarma eşlemesi ve kayıt API'si başvuru kaynağı.
  </Card>
</CardGroup>

## Genel yetenek modeli

Yetenekler, OpenClaw içindeki genel **yerel Plugin** modelidir. Her yerel OpenClaw Plugini bir veya daha fazla yetenek türü için kaydolur:

| Yetenek                 | Kayıt yöntemi                                     | Örnek pluginler                                             |
| ----------------------- | ------------------------------------------------- | ----------------------------------------------------------- |
| Metin çıkarımı          | `api.registerProvider(...)`                                | `anthropic`, `openai`                     |
| CLI çıkarım arka ucu    | `api.registerCliBackend(...)`                                | `anthropic`, `openai`                     |
| Gömme                   | `api.registerEmbeddingProvider(...)`                                | Sağlayıcıya ait vektör pluginleri                            |
| Konuşma                 | `api.registerSpeechProvider(...)`                                | `elevenlabs`, `microsoft`                     |
| Gerçek zamanlı yazıya dökme | `api.registerRealtimeTranscriptionProvider(...)`                            | `openai`                                          |
| Gerçek zamanlı ses      | `api.registerRealtimeVoiceProvider(...)`                                | `google`, `openai`                     |
| Medya anlama            | `api.registerMediaUnderstandingProvider(...)`                                | `google`, `openai`                     |
| Transkript kaynağı      | `api.registerTranscriptSourceProvider(...)`                                | `discord`, `google-meet`, `teams-meetings`, `zoom-meetings` |
| Görüntü oluşturma       | `api.registerImageGenerationProvider(...)`                                | `fal`, `google`, `openai` |
| Müzik oluşturma         | `api.registerMusicGenerationProvider(...)`                                | `fal`, `google`, `minimax` |
| Video oluşturma         | `api.registerVideoGenerationProvider(...)`                                | `fal`, `google`, `qwen` |
| Web'den getirme         | `api.registerWebFetchProvider(...)`                                | `firecrawl`                                          |
| Web'de arama            | `api.registerWebSearchProvider(...)`                                | `brave`, `firecrawl`, `google` |
| Kanal / mesajlaşma      | `api.registerChannel(...)`                                | `matrix`, `msteams`                     |
| Gateway keşfi           | `api.registerGatewayDiscoveryService(...)`                                | `bonjour`                                          |

<Note>
Sıfır yetenek kaydeden ancak kancalar, araçlar, keşif hizmetleri veya arka plan hizmetleri sağlayan bir Plugin, **yalnızca eski kancaları kullanan** bir Plugindir. Bu düzen hâlâ tam olarak desteklenmektedir.
</Note>

### Harici uyumluluk yaklaşımı

Yetenek modeli çekirdeğe eklenmiştir ve günümüzde paketle gelen/yerel pluginler tarafından kullanılmaktadır; ancak harici Plugin uyumluluğu için hâlâ "dışa aktarılmışsa dondurulmuştur" yaklaşımından daha sıkı bir ölçüt gerekir.

| Plugin durumu                                      | Yönlendirme                                                                                                        |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| Mevcut harici pluginler                            | Kanca tabanlı entegrasyonların çalışmasını sürdürün; uyumluluğun temel çizgisi budur.                               |
| Paketle gelen/yeni yerel pluginler                 | Sağlayıcıya özgü doğrudan erişimler veya yalnızca kancaları kullanan yeni tasarımlar yerine açık yetenek kaydını tercih edin. |
| Yetenek kaydını benimseyen harici pluginler        | İzin verilir; ancak belgeler kararlı olduğunu belirtmedikçe yeteneğe özgü yardımcı yüzeyleri gelişmekte kabul edin. |

Amaçlanan yön yetenek kaydıdır. Geçiş sırasında eski kancalar, harici pluginler için bozulmaya yol açmayan en güvenli yol olmaya devam eder. Dışa aktarılan yardımcı alt yolların tümü eşit değildir; rastlantısal yardımcı dışa aktarımlar yerine dar kapsamlı, belgelenmiş sözleşmeleri tercih edin.

### Plugin biçimleri

OpenClaw, yüklenen her Plugini yalnızca statik meta verilere göre değil, gerçek kayıt davranışına göre bir biçimde sınıflandırır:

<AccordionGroup>
  <Accordion title="yalın-yetenek">
    Tam olarak bir yetenek türü kaydeder (örneğin `arcee` veya `chutes` gibi yalnızca sağlayıcı görevi gören bir Plugin).
  </Accordion>
  <Accordion title="hibrit-yetenek">
    Birden fazla yetenek türü kaydeder (örneğin `openai`; metin çıkarımı, konuşma, medya anlama ve görüntü oluşturmanın sahibidir).
  </Accordion>
  <Accordion title="yalnızca-kanca">
    Yalnızca kancaları (türlü veya özel) kaydeder; yetenek, araç, komut veya hizmet kaydetmez.
  </Accordion>
  <Accordion title="yetenek-dışı">
    Araçları, komutları, hizmetleri veya rotaları kaydeder ancak yetenek kaydetmez.
  </Accordion>
</AccordionGroup>

Bir Pluginin biçimini ve yetenek dökümünü görmek için `openclaw plugins inspect <id>` kullanın. Ayrıntılar için [CLI başvuru kaynağına](/tr/cli/plugins#inspect) bakın.

### Uyumluluk sinyalleri

`openclaw doctor`, `openclaw plugins inspect <id>`, `openclaw status --all` ve `openclaw plugins doctor` şu uyumluluk bildirimlerini gösterir:

| Sinyal                                             | Anlamı                                                                                                                     |
| -------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| **yapılandırma geçerli**                           | Yapılandırma sorunsuz ayrıştırılır ve pluginler çözümlenir                                                                 |
| **yalnızca-kanca** (bilgi)                         | Plugin yalnızca kancaları kaydeder; bu desteklenen bir yoldur ancak henüz yetenek kaydına geçirilmemiştir                   |
| **kullanımdan kaldırılmış bellek gömme API'si** (uyarı) | Paketle gelmeyen Plugin, `registerEmbeddingProvider` yerine belleğe özgü eski gömme sağlayıcısı API'sini kullanır                 |
| **kesin hata**                                     | Yapılandırma geçersizdir veya Plugin yüklenememiştir                                                                       |

Bilgilendirme/uyarı sinyallerinin hiçbiri bugün Plugininizi bozmaz. Bu sinyaller `openclaw status --all` ve `openclaw plugins doctor` içinde de görünür.

## Mimariye genel bakış

OpenClaw'ın Plugin sistemi dört katmandan oluşur:

<Steps>
  <Step title="Manifest + keşif">
    OpenClaw; yapılandırılmış yollarda, çalışma alanı köklerinde, genel Plugin köklerinde ve paketle gelen pluginlerde aday pluginleri bulur. Keşif önce yerel `openclaw.plugin.json` manifestlerini ve desteklenen paket manifestlerini okur.
  </Step>
  <Step title="Etkinleştirme + doğrulama">
    Çekirdek, keşfedilen bir Pluginin etkinleştirildiğine, devre dışı bırakıldığına, engellendiğine veya bellek gibi özel bir yuva için seçildiğine karar verir.
  </Step>
  <Step title="Çalışma zamanı yüklemesi">
    Yerel OpenClaw pluginleri işlem içinde yüklenir ve yetenekleri merkezi bir kayıt defterine kaydeder. Paketlenmiş JavaScript, yerel `require` üzerinden yüklenir; üçüncü taraf yerel kaynak TypeScript için acil durum alternatifi Jiti'dir. Uyumlu paketler, çalışma zamanı kodu içe aktarılmadan kayıt defteri girdilerine dönüştürülür.
  </Step>
  <Step title="Yüzey kullanımı">
    OpenClaw'ın geri kalanı araçları, kanalları, sağlayıcı kurulumunu, kancaları, HTTP rotalarını, CLI komutlarını ve hizmetleri sunmak için kayıt defterini okur.
  </Step>
</Steps>

Özellikle Plugin CLI'sinde kök komut keşfi iki aşamaya ayrılır:

- ayrıştırma zamanı meta verileri `registerCli(..., { descriptors: [...] })` kaynağından gelir
- gerçek Plugin CLI modülü tembel kalabilir ve ilk çağrıda kaydolabilir

Böylece OpenClaw, ayrıştırmadan önce kök komut adlarını ayırabilirken Plugine ait CLI kodu Plugin içinde kalır.

Önemli tasarım sınırı:

- manifest/yapılandırma doğrulaması, Plugin kodu çalıştırılmadan **manifest/şema meta verilerinden** yapılabilmelidir
- yerel yetenek keşfi, etkinleştirme yapmayan bir kayıt defteri anlık görüntüsü oluşturmak için güvenilir Plugin giriş kodunu yükleyebilir
- yerel çalışma zamanı davranışı, Plugin modülünün `register(api)` yolundan ve `api.registrationMode === "full"` ile gelir

Bu ayrım, tam çalışma zamanı etkinleşmeden önce OpenClaw'ın yapılandırmayı doğrulamasına, eksik/devre dışı pluginleri açıklamasına ve kullanıcı arayüzü/şema ipuçları oluşturmasına olanak tanır.

### Plugin meta verisi anlık görüntüsü ve arama tablosu

Gateway başlangıcı, geçerli yapılandırma anlık görüntüsü için bir `PluginMetadataSnapshot` oluşturur. Anlık görüntü yalnızca meta verilerden oluşur: yüklü Plugin dizinini, manifest kayıt defterini, manifest tanılamalarını, sahip eşlemelerini, Plugin kimliği normalleştiricisini ve manifest kayıtlarını depolar. Yüklenmiş Plugin modüllerini, sağlayıcı SDK'lerini, paket içeriklerini veya çalışma zamanı dışa aktarımlarını içermez.

Pluginleri dikkate alan yapılandırma doğrulaması, başlangıçta otomatik etkinleştirme ve Gateway Plugin önyüklemesi; manifest/dizin meta verilerini birbirinden bağımsız olarak yeniden oluşturmak yerine bu anlık görüntüyü kullanır. `PluginLookUpTable` aynı anlık görüntüden türetilir ve geçerli çalışma zamanı yapılandırması için başlangıç Plugin planını ekler.

Gateway, başlangıçtan sonra geçerli meta veri anlık görüntüsünü değiştirilebilir bir çalışma zamanı ürünü olarak tutar. Yinelenen çalışma zamanı sağlayıcı keşfi, her sağlayıcı kataloğu geçişinde yüklü dizini ve manifest kayıt defterini yeniden oluşturmak yerine bu anlık görüntüyü ödünç alabilir. Gateway kapatıldığında, yapılandırma/Plugin envanteri değiştiğinde ve yüklü dizine yazıldığında anlık görüntü temizlenir veya değiştirilir; uyumlu ve geçerli bir anlık görüntü bulunmadığında çağıranlar soğuk manifest/dizin yoluna geri döner. Uyumluluk denetimleri, `plugins.load.paths` ve varsayılan ajan çalışma alanı gibi Plugin keşif köklerini içermelidir; çünkü çalışma alanı pluginleri meta veri kapsamının parçasıdır.

Anlık görüntü ve arama tablosu, yinelenen başlangıç kararlarını hızlı yolda tutar:

- kanal sahipliği
- ertelenmiş kanal başlangıcı
- başlangıç Plugin kimlikleri
- sağlayıcı ve CLI arka uç sahipliği
- kurulum sağlayıcısı, komut diğer adı, model kataloğu sağlayıcısı ve manifest sözleşmesi sahipliği
- Plugin yapılandırma şeması ve kanal yapılandırma şeması doğrulaması
- başlangıçta otomatik etkinleştirme kararları

Güvenlik sınırı mutasyon değil, anlık görüntünün değiştirilmesidir. Yapılandırma, Plugin envanteri, yükleme kayıtları veya kalıcı dizin ilkesi değiştiğinde anlık görüntüyü yeniden oluşturun. Bunu geniş kapsamlı, değiştirilebilir bir genel kayıt defteri olarak kullanmayın ve sınırsız sayıda geçmiş anlık görüntüyü tutmayın. Eski çalışma zamanı durumunun bir meta veri önbelleğinin arkasına gizlenememesi için çalışma zamanı Plugin yüklemesi, meta veri anlık görüntülerinden ayrı kalır.

Önbellek kuralı [Plugin mimarisi iç işleyişi](/tr/plugins/architecture-internals#plugin-cache-boundary) bölümünde belgelenmiştir: çağıran geçerli akış için açık bir anlık görüntü, arama tablosu veya manifest kayıt defteri tutmadığı sürece manifest ve keşif meta verileri günceldir. Gizli meta veri önbellekleri ve duvar saati TTL'leri Plugin yüklemesinin parçası değildir. Kod veya yüklü yapılar gerçekten yüklendikten sonra yalnızca çalışma zamanı yükleyicisi, modül ve bağımlılık yapısı önbellekleri kalıcı olabilir.

Bazı soğuk yol çağıranları, bir Gateway `PluginLookUpTable` almak yerine manifest kayıtlarını kalıcı yüklenmiş plugin dizininden doğrudan yeniden oluşturmaya devam ediyor. Bu yol artık kaydı istek üzerine yeniden oluşturuyor; çağıranda zaten mevcutsa çalışma zamanı akışları üzerinden geçerli arama tablosunu veya açık bir manifest kaydını aktarmayı tercih edin.

### Etkinleştirme planlaması

Etkinleştirme planlaması, kontrol düzleminin bir parçasıdır. Çağıranlar, daha geniş çalışma zamanı kayıtlarını yüklemeden önce somut bir komut, sağlayıcı, kanal, rota, aracı koşum takımı veya yetenek için hangi pluginlerin ilgili olduğunu sorgulayabilir.

Planlayıcı, mevcut manifest davranışının uyumluluğunu korur:

- `activation.*` alanları açık planlayıcı ipuçlarıdır
- `providers`, `channels`, `commandAliases`, `setup.providers`, `contracts.tools` ve hook'lar manifest sahipliği geri dönüşü olmaya devam eder
- yalnızca kimliklerden oluşan planlayıcı API'si mevcut çağıranlar için kullanılabilir kalır
- plan API'si neden etiketlerini bildirir; böylece tanılama, açık ipuçlarını sahiplik geri dönüşünden ayırt edebilir

<Warning>
`activation` öğesini bir yaşam döngüsü hook'u veya `register(...)` yerine geçen bir unsur olarak değerlendirmeyin. Bu, yüklemeyi daraltmak için kullanılan metaveridir. İlişkiyi zaten tanımlıyorlarsa sahiplik alanlarını tercih edin; `activation` öğesini yalnızca ek planlayıcı ipuçları için kullanın.
</Warning>

### Kanal pluginleri ve paylaşılan mesaj aracı

Kanal pluginlerinin normal sohbet eylemleri için ayrı bir gönderme/düzenleme/tepki verme aracı kaydetmesi gerekmez. OpenClaw, çekirdekte tek bir paylaşılan `message` aracı tutar ve kanal pluginleri bunun arkasındaki kanala özgü keşif ile yürütmenin sahipliğini üstlenir.

Geçerli sınır şöyledir:

- çekirdek; paylaşılan `message` araç barındırıcısının, istem bağlantılarının, oturum/ileti dizisi kaydının ve yürütme yönlendirmenin sahibidir
- kanal pluginleri; kapsamlı eylem keşfinin, yetenek keşfinin ve kanala özgü tüm şema parçalarının sahibidir
- kanal pluginleri; konuşma kimliklerinin ileti dizisi kimliklerini nasıl kodladığı veya üst konuşmalardan nasıl devralındığı gibi, sağlayıcıya özgü oturum konuşması dilbilgisinin sahibidir
- kanal pluginleri son eylemi kendi eylem bağdaştırıcıları üzerinden yürütür

Kanal pluginleri için SDK yüzeyi `ChannelMessageActionAdapter.describeMessageTool(...)` şeklindedir. Bu birleşik keşif çağrısı, söz konusu parçaların birbirinden sapmaması için bir pluginin görünür eylemlerini, yeteneklerini ve şema katkılarını birlikte döndürmesine olanak tanır.

Mesaj eylemi adları, her aktarımın her eylemi işleyebilmesi için kasıtlı olarak kapalı ve çekirdeğin sahip olduğu bir söz dağarcığını kullanır. Pluginler eylem adlarını bir çekirdek PR'ı aracılığıyla ekler; çalışma zamanında kayıt kasıtlı olarak desteklenmez.

Kanala özgü bir mesaj aracı parametresi, yerel yol veya uzak medya URL'si gibi bir medya kaynağı taşıdığında plugin ayrıca `describeMessageTool(...)` içinden `mediaSourceParams` döndürmelidir. Çekirdek, pluginin sahip olduğu parametre adlarını sabit kodlamadan korumalı alan yol normalleştirmesi ve giden medya erişimi ipuçlarını uygulamak için bu açık listeyi kullanır. Profil kapsamlı bir medya parametresinin `send` gibi ilgisiz eylemlerde normalleştirilmemesi için burada kanal genelinde tek bir düz liste yerine eylem kapsamlı eşlemeleri tercih edin.

Çekirdek, çalışma zamanı kapsamını bu keşif adımına aktarır. Önemli alanlar şunlardır:

- `accountId`
- `currentChannelId`
- `currentThreadTs`
- `currentMessageId`
- `sessionKey`
- `sessionId`
- `agentId`
- güvenilir gelen `requesterSenderId`

Bu, bağlama duyarlı pluginler için önemlidir. Bir kanal; çekirdek `message` aracında kanala özgü dalları sabit kodlamadan, etkin hesaba, geçerli odaya/ileti dizisine/mesaja veya güvenilir istekte bulunanın kimliğine göre mesaj eylemlerini gizleyebilir ya da gösterebilir.

Gömülü çalıştırıcı yönlendirme değişikliklerinin hâlâ plugin işi olmasının nedeni budur: çalıştırıcı, geçerli sohbet/oturum kimliğini plugin keşif sınırına iletmekten sorumludur; böylece paylaşılan `message` aracı geçerli tur için kanalın sahip olduğu doğru yüzeyi sunar.

Kanalın sahip olduğu yürütme yardımcıları için kanal pluginleri, yürütme çalışma zamanını kendi plugin modülleri içinde tutmalıdır. Çekirdek artık `src/agents/tools` altında Discord, Slack, Telegram veya WhatsApp mesaj eylemi çalışma zamanlarının sahibi değildir. Ayrı `plugin-sdk/*-action-runtime` alt yolları yayımlamıyoruz ve bu pluginler kendi yerel çalışma zamanı kodlarını doğrudan pluginin sahip olduğu modüllerden içe aktarmalıdır.

Aynı sınır genel olarak sağlayıcı adını taşıyan SDK bağlantı noktaları için de geçerlidir: çekirdek Discord, Signal, Slack, WhatsApp veya benzer pluginler için kanala özgü kolaylık barrel'larını içe aktarmamalıdır. Çekirdeğin bir davranışa ihtiyacı varsa ya paketlenmiş pluginin kendi `api.ts` / `runtime-api.ts` barrel'ını kullanmalı ya da ihtiyacı paylaşılan SDK'da dar kapsamlı, genel bir yeteneğe yükseltmelidir.

Paketlenmiş pluginler de aynı kuralı izler. Paketlenmiş bir pluginin `runtime-api.ts` öğesi, kendi markalı `openclaw/plugin-sdk/<plugin-id>` cephesini yeniden dışa aktarmamalıdır. Bu markalı cepheler haricî pluginler ve eski tüketiciler için uyumluluk shim'leri olarak kalır; ancak paketlenmiş pluginler yerel dışa aktarımları ve `openclaw/plugin-sdk/channel-policy`, `openclaw/plugin-sdk/runtime-store` veya `openclaw/plugin-sdk/webhook-ingress` gibi dar kapsamlı genel SDK alt yollarını kullanmalıdır. Yeni kod, mevcut bir haricî ekosistemin uyumluluk sınırı gerektirmediği sürece plugin kimliğine özgü SDK cepheleri eklememelidir.

Özellikle anketler için iki yürütme yolu vardır:

- `outbound.sendPoll`, ortak anket modeline uyan kanallar için paylaşılan temeldir
- `actions.handleAction("poll")`, kanala özgü anket semantiği veya ek anket parametreleri için tercih edilen yoldur

Çekirdek artık paylaşılan anket ayrıştırmasını, plugin anket yönlendirmesi eylemi reddedene kadar erteler; böylece pluginin sahip olduğu anket işleyicileri önce genel anket ayrıştırıcısı tarafından engellenmeden kanala özgü anket alanlarını kabul edebilir.

Başlatma sırasının tamamı için [Plugin mimarisi iç ayrıntıları](/tr/plugins/architecture-internals) bölümüne bakın.

## Yetenek sahipliği modeli

OpenClaw, yerel bir plugini ilgisiz entegrasyonların toplandığı bir paket olarak değil, bir **şirketin** veya **özelliğin** sahiplik sınırı olarak değerlendirir.

Bunun anlamı şudur:

- bir şirket plugini genellikle o şirketin OpenClaw'a dönük tüm yüzeylerinin sahibi olmalıdır
- bir özellik plugini genellikle sunduğu özellik yüzeyinin tamamının sahibi olmalıdır
- kanallar, sağlayıcı davranışını geçici biçimde yeniden uygulamak yerine paylaşılan çekirdek yeteneklerini kullanmalıdır

<AccordionGroup>
  <Accordion title="Çok yetenekli tedarikçi">
    `google`; metin çıkarımı, CLI arka ucu, gömmeler, konuşma, gerçek zamanlı ses, medya anlama, görüntü/müzik/video üretimi ve web aramasının sahibidir. `openai`; metin çıkarımı, gömmeler, konuşma, gerçek zamanlı transkripsiyon, gerçek zamanlı ses, medya anlama ve görüntü/video üretiminin sahibidir. `minimax`; metin çıkarımının yanı sıra medya anlama, konuşma, görüntü/müzik/video üretimi ve web aramasının sahibidir.
  </Accordion>
  <Accordion title="Tek yetenekli tedarikçi">
    `arcee` ve `chutes` yalnızca metin çıkarımının; `microsoft` ise yalnızca konuşmanın sahibidir. Bir tedarikçi plugini, o tedarikçinin daha geniş yüzeyini kapsaması gerekene kadar bu kadar dar kalabilir.
  </Accordion>
  <Accordion title="Özellik plugini">
    `voice-call`; çağrı aktarımı, araçlar, CLI, rotalar ve Twilio medya akışı köprülemesinin sahibidir; ancak tedarikçi pluginlerini doğrudan içe aktarmak yerine paylaşılan konuşma, gerçek zamanlı transkripsiyon ve gerçek zamanlı ses yeteneklerini kullanır.
  </Accordion>
</AccordionGroup>

Amaçlanan son durum şöyledir:

- bir tedarikçinin OpenClaw'a dönük yüzeyi, metin modelleri, konuşma, görüntüler ve videoyu kapsasa bile tek bir pluginde bulunur
- diğer tedarikçiler de kendi yüzey alanları için aynısını yapabilir
- kanallar, sağlayıcının hangi tedarikçi pluginine ait olduğunu önemsemez; çekirdeğin sunduğu paylaşılan yetenek sözleşmesini kullanırlar

Temel ayrım şudur:

- **plugin** = sahiplik sınırı
- **yetenek** = birden fazla pluginin uygulayabildiği veya kullanabildiği çekirdek sözleşmesi

Dolayısıyla OpenClaw video gibi yeni bir alan eklerse ilk soru "hangi sağlayıcı video işlemeyi sabit kodlamalı?" değildir. İlk soru "temel video yeteneği sözleşmesi nedir?" olmalıdır. Bu sözleşme oluşturulduktan sonra tedarikçi pluginleri buna göre kaydolabilir ve kanal/özellik pluginleri bunu kullanabilir.

Yetenek henüz mevcut değilse doğru yaklaşım genellikle şöyledir:

<Steps>
  <Step title="Yeteneği tanımlayın">
    Eksik yeteneği çekirdekte tanımlayın.
  </Step>
  <Step title="SDK üzerinden sunun">
    Plugin API'si/çalışma zamanı üzerinden tür güvenli biçimde sunun.
  </Step>
  <Step title="Tüketicileri bağlayın">
    Kanalları/özellikleri bu yeteneğe bağlayın.
  </Step>
  <Step title="Tedarikçi uygulamaları">
    Tedarikçi pluginlerinin uygulamaları kaydetmesine olanak tanıyın.
  </Step>
</Steps>

Bu, tek bir tedarikçiye veya tek seferlik plugine özgü bir kod yoluna bağımlı çekirdek davranışından kaçınırken sahipliği açık tutar.

### Yetenek katmanlandırması

Kodun nereye ait olduğuna karar verirken şu zihinsel modeli kullanın:

<Tabs>
  <Tab title="Çekirdek yetenek katmanı">
    Paylaşılan orkestrasyon, politika, geri dönüş, yapılandırma birleştirme kuralları, teslim semantiği ve tür güvenli sözleşmeler.
  </Tab>
  <Tab title="Tedarikçi plugin katmanı">
    Tedarikçiye özgü API'ler, kimlik doğrulama, model katalogları, konuşma sentezi, görüntü üretimi, video arka uçları ve kullanım uç noktaları.
  </Tab>
  <Tab title="Kanal/özellik plugin katmanı">
    Çekirdek yetenekleri kullanan ve bunları bir yüzeyde sunan Discord/Slack/sesli arama/vb. entegrasyonu.
  </Tab>
</Tabs>

Örneğin TTS şu yapıyı izler:

- çekirdek; yanıt zamanı TTS politikasının, geri dönüş sırasının, tercihlerin ve kanal tesliminin sahibidir
- `elevenlabs`, `google`, `microsoft` ve `openai` sentez uygulamalarının sahibidir
- `voice-call` telefon TTS çalışma zamanı yardımcısını kullanır

Gelecekteki yetenekler için de aynı kalıp tercih edilmelidir.

### Çok yetenekli şirket plugini örneği

Bir şirket plugini dışarıdan bakıldığında bütünlüklü hissettirmelidir. OpenClaw'ın modeller, konuşma, gerçek zamanlı transkripsiyon, gerçek zamanlı ses, medya anlama, görüntü üretimi, video üretimi, web getirme ve web araması için paylaşılan sözleşmeleri varsa bir tedarikçi tüm yüzeylerine tek bir yerde sahip olabilir:

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { exampleAiMedia } from "./exampleai-media.js";

export default definePluginEntry({
  id: "exampleai",
  name: "ExampleAI",
  description: "ExampleAI modelleri ve medya yetenekleri.",
  register(api) {
    api.registerProvider({
      id: "exampleai",
      // kimlik doğrulama/model kataloğu/çalışma zamanı hook'ları
    });

    api.registerSpeechProvider({
      id: "exampleai",
      // tedarikçi konuşma yapılandırması — SpeechProviderPlugin arayüzünü doğrudan uygulayın
    });

    api.registerMediaUnderstandingProvider({
      id: "exampleai",
      capabilities: ["image", "audio", "video"],
      describeImage: (req) => exampleAiMedia.describeImage(req),
      transcribeAudio: (req) => exampleAiMedia.transcribeAudio(req),
      describeVideo: (req) => exampleAiMedia.describeVideo(req),
    });

    api.registerWebSearchProvider({
      id: "exampleai-search",
      createTool() {
        // Tedarikçinin sahip olduğu web arama aracını döndürün.
      },
    });
  },
});
```

Önemli olan yardımcıların tam adları değildir. Önemli olan yapıdır:

- tek bir plugin tedarikçi yüzeyinin sahibidir
- yetenek sözleşmelerinin sahibi hâlâ çekirdektir
- sağlayıcı isteği dönüştürme ve HTTP yardımcıları tedarikçi plugininde kalır
- kanallar ve özellik pluginleri tedarikçi kodunu değil, `api.runtime.*` yardımcılarını kullanır
- sözleşme testleri, pluginin sahip olduğunu belirttiği yetenekleri kaydettiğini doğrulayabilir

### Yetenek örneği: video anlama

OpenClaw görüntü/ses/video anlamayı zaten tek bir paylaşılan yetenek olarak değerlendirir. Aynı sahiplik modeli burada da geçerlidir:

<Steps>
  <Step title="Core sözleşmeyi tanımlar">
    Core, medya anlama sözleşmesini tanımlar.
  </Step>
  <Step title="Sağlayıcı pluginleri kaydolur">
    Sağlayıcı pluginleri, uygun olduğunda `describeImage`, `transcribeAudio` ve `describeVideo` öğelerini kaydeder.
  </Step>
  <Step title="Tüketiciler ortak davranışı kullanır">
    Kanallar ve özellik pluginleri, doğrudan sağlayıcı koduna bağlanmak yerine ortak core davranışını kullanır.
  </Step>
</Steps>

Bu, tek bir sağlayıcının video varsayımlarının core içine gömülmesini önler. Sağlayıcı yüzeyinin sahibi plugindir; yetenek sözleşmesinin ve geri dönüş davranışının sahibi ise core'dur.

Video oluşturma zaten aynı sıralamayı kullanır: Türü belirlenmiş yetenek sözleşmesinin ve çalışma zamanı yardımcısının sahibi core'dur; sağlayıcı pluginleri ise `api.registerVideoGenerationProvider(...)` uygulamalarını buna göre kaydeder.

Somut bir kullanıma sunma kontrol listesine mi ihtiyacınız var? [Yetenek Tarifleri](/tr/plugins/adding-capabilities) sayfasına bakın.

## Sözleşmeler ve uygulama

Plugin API yüzeyi, `OpenClawPluginApi` içinde kasıtlı olarak türü belirlenmiş ve merkezileştirilmiştir. Bu sözleşme, desteklenen kayıt noktalarını ve bir pluginin güvenebileceği çalışma zamanı yardımcılarını tanımlar.

Bunun önemi:

- plugin yazarları tek bir kararlı dahili standarda sahip olur
- core, aynı sağlayıcı kimliğini kaydeden iki plugin gibi yinelenen sahiplikleri reddedebilir
- başlangıç, hatalı biçimlendirilmiş kayıtlar için eyleme dönüştürülebilir tanılamalar gösterebilir
- sözleşme testleri, paketlenmiş plugin sahipliğini uygulayabilir ve sessiz sapmaları önleyebilir

İki uygulama katmanı vardır:

<AccordionGroup>
  <Accordion title="Çalışma zamanı kayıt uygulaması">
    Plugin kayıt defteri, pluginler yüklenirken kayıtları doğrular. Örneğin yinelenen sağlayıcı kimlikleri, yinelenen konuşma sağlayıcısı kimlikleri ve hatalı biçimlendirilmiş kayıtlar, tanımsız davranış yerine plugin tanılamaları üretir.
  </Accordion>
  <Accordion title="Sözleşme testleri">
    OpenClaw'un sahipliği açıkça doğrulayabilmesi için paketlenmiş pluginler, test çalıştırmaları sırasında sözleşme kayıt defterlerinde yakalanır. Günümüzde bu; model sağlayıcıları, konuşma sağlayıcıları, web arama sağlayıcıları ve paketlenmiş kayıt sahipliği için kullanılır.
  </Accordion>
</AccordionGroup>

Bunun pratik etkisi, OpenClaw'un hangi yüzeyin hangi plugine ait olduğunu en baştan bilmesidir. Sahiplik örtük olmak yerine bildirilmiş, türü belirlenmiş ve test edilebilir olduğundan core ile kanallar sorunsuz biçimde birleştirilebilir.

### Bir sözleşmede neler bulunmalı

<Tabs>
  <Tab title="İyi sözleşmeler">
    - türü belirlenmiş
    - küçük
    - yeteneğe özgü
    - core'a ait
    - birden fazla plugin tarafından yeniden kullanılabilir
    - sağlayıcı bilgisi gerektirmeden kanallar/özellikler tarafından kullanılabilir

  </Tab>
  <Tab title="Kötü sözleşmeler">
    - core içinde gizlenen sağlayıcıya özgü politika
    - kayıt defterini atlayan tek kullanımlık plugin kaçış yolları
    - doğrudan bir sağlayıcı uygulamasına erişen kanal kodu
    - `OpenClawPluginApi` veya `api.runtime` kapsamında olmayan geçici çalışma zamanı nesneleri

  </Tab>
</Tabs>

Şüpheye düştüğünüzde soyutlama düzeyini yükseltin: önce yeteneği tanımlayın, ardından pluginlerin buna bağlanmasını sağlayın.

## Yürütme modeli

Yerel OpenClaw pluginleri, Gateway ile aynı **işlem içinde** çalışır. Korumalı alanda çalıştırılmazlar. Yüklenen bir yerel plugin, core koduyla aynı işlem düzeyindeki güven sınırına sahiptir.

<Warning>
Yerel pluginlerin etkileri: Bir plugin araçları, ağ işleyicilerini, kancaları ve hizmetleri kaydedebilir; bir plugin hatası Gateway'in çökmesine veya kararsızlaşmasına yol açabilir; kötü amaçlı bir yerel plugin ise OpenClaw işlemi içinde rastgele kod yürütülmesine eşdeğerdir.
</Warning>

OpenClaw şu anda uyumlu paketleri meta veri/içerik paketleri olarak ele aldığından, bunlar varsayılan olarak daha güvenlidir. Mevcut sürümlerde bu çoğunlukla paketlenmiş Skills anlamına gelir.

Paketlenmemiş pluginler için izin listeleri ve açık kurulum/yükleme yolları kullanın. Çalışma alanı pluginlerini üretim varsayılanları olarak değil, geliştirme zamanı kodu olarak değerlendirin.

Paketlenmiş çalışma alanı paket adlarında plugin kimliğini npm adına bağlı tutun: varsayılan olarak `@openclaw/<id>`; paket kasıtlı olarak daha dar bir plugin rolü sunuyorsa `-provider`, `-plugin`, `-speech`, `-sandbox` veya `-media-understanding` gibi onaylanmış, türü belirlenmiş bir sonek kullanın.

<Note>
**Güven notu:** `plugins.allow`, kaynak menşeine değil **plugin kimliklerine** güvenir. Paketlenmiş bir pluginle aynı kimliğe sahip çalışma alanı plugini etkinleştirildiğinde/izin listesine eklendiğinde, kasıtlı olarak paketlenmiş kopyanın yerine geçer. Bu normaldir ve yerel geliştirme, yama testi ve acil düzeltmeler için kullanışlıdır. Paketlenmiş plugin güveni, kurulum meta verilerinden değil kaynak anlık görüntüsünden — yükleme anında diskte bulunan manifest ve koddan — belirlenir. Bozulmuş veya değiştirilmiş bir kurulum kaydı, paketlenmiş bir pluginin güven yüzeyini gerçek kaynağın beyan ettiğinin ötesine sessizce genişletemez.
</Note>

## Dışa aktarma sınırı

OpenClaw, uygulama kolaylıklarını değil yetenekleri dışa aktarır.

Yetenek kaydını herkese açık tutun. Sözleşme dışı yardımcı dışa aktarımları azaltın:

- paketlenmiş plugine özgü yardımcı alt yollar
- genel API olarak tasarlanmamış çalışma zamanı tesisatı alt yolları
- sağlayıcıya özgü kolaylık yardımcıları
- uygulama ayrıntısı olan kurulum/ilk katılım yardımcıları

Ayrılmış paketlenmiş plugin yardımcı alt yolları, oluşturulan SDK dışa aktarma haritasından kaldırılmıştır. Sahibe özgü yardımcıları sahip plugin paketinin içinde tutun; yalnızca yeniden kullanılabilir ana makine davranışını `plugin-sdk/gateway-runtime`, `plugin-sdk/security-runtime` ve enjekte edilen plugin API yetenekleri gibi genel SDK sözleşmelerine yükseltin.

## Dahili işleyiş ve başvuru

Yükleme işlem hattı, kayıt defteri modeli, sağlayıcı çalışma zamanı kancaları, Gateway HTTP rotaları, mesaj aracı şemaları, kanal hedefi çözümleme, sağlayıcı katalogları, bağlam motoru pluginleri ve yeni bir yetenek ekleme kılavuzu için [Plugin mimarisi dahili işleyişi](/tr/plugins/architecture-internals) sayfasına bakın.

## İlgili

- [Plugin oluşturma](/tr/plugins/building-plugins)
- [Plugin manifesti](/tr/plugins/manifest)
- [Plugin SDK kurulumu](/tr/plugins/sdk-setup)
