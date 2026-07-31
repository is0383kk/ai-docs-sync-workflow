---
read_when:
    - Yeni bir mesajlaşma kanalı plugini oluşturuyorsunuz
    - OpenClaw'u bir mesajlaşma platformuna bağlamak istiyorsunuz
    - ChannelPlugin bağdaştırıcı yüzeyini anlamanız gerekir
sidebarTitle: Channel Plugins
summary: OpenClaw için mesajlaşma kanalı plugini oluşturmaya yönelik adım adım kılavuz
title: Kanal pluginleri oluşturma
x-i18n:
    generated_at: "2026-07-27T00:50:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ff8ad04346babf3eece7e10bd38946ee290947b2e504b6b5ca438865531bf38
    source_path: plugins/sdk-channel-plugins.md
    workflow: 16
---

Bu kılavuz, OpenClaw'u bir mesajlaşma platformuna bağlayan bir kanal plugini oluşturur: DM güvenliği, eşleştirme, yanıt dizileri ve giden mesajlaşma.

<Info>
  OpenClaw pluginlerinde yeni misiniz? Paket yapısı ve manifest kurulumu için önce
  [Başlarken](/tr/plugins/building-plugins) bölümünü okuyun.
</Info>

## Plugininizin sorumlulukları

Kanal pluginleri gönderme/düzenleme/tepki araçlarını uygulamaz; çekirdek tek bir
paylaşılan `message` aracı sağlar. Plugininizin sorumlulukları:

- **Yapılandırma** - hesap çözümleme ve kurulum sihirbazı
- **Güvenlik** - DM politikası ve izin listeleri
- **Eşleştirme** - DM onay akışı
- **Oturum dil bilgisi** - sağlayıcıya özgü konuşma kimliklerinin temel
  sohbetlere, dizi kimliklerine ve üst öğe geri dönüşlerine nasıl eşlendiği
- **Giden** - platforma metin, medya ve anket gönderme
- **Dizileme** - yanıtların nasıl dizilendiği
- **Heartbeat yazıyor göstergesi** - Heartbeat teslimat
  hedefleri için isteğe bağlı yazıyor/meşgul sinyalleri

Çekirdek; paylaşılan mesaj aracını, istem bağlantılarını, dış oturum anahtarı biçimini,
genel `:thread:` kaydını ve yönlendirmeyi yönetir.

## Mesaj adaptörü

`openclaw/plugin-sdk/channel-outbound` içindeki `defineChannelMessageAdapter` ile bir
`message` adaptörü sunun. Yalnızca yerel aktarımınızın gerçekten
desteklediği kalıcı son gönderim yeteneklerini bildirin ve bunları yerel yan
etkiyi ve döndürülen alındı belgesini kanıtlayan bir sözleşme testiyle destekleyin.
Metin/medya gönderimlerini eski `outbound` adaptörünün kullandığı aktarım
işlevlerine yönlendirin. Tam API sözleşmesi, yetenek matrisi, alındı belgesi
kuralları, canlı önizleme sonlandırma, alım onayı politikası, testler ve geçiş
tablosu için [Kanal giden API'sine](/tr/plugins/sdk-channel-outbound) bakın.

Mevcut `outbound` adaptörünüz doğru gönderim yöntemlerine ve yetenek
meta verilerine zaten sahipse başka bir köprüyü elle yazmak yerine
`createChannelMessageAdapterFromOutbound(...)` ile `message` adaptörünü türetin.
Adaptör gönderimleri `MessageReceipt` değerleri döndürür. Eski kimlikler için
paralel `messageIds` alanlarını korumak yerine bunları
`listMessageReceiptPlatformIds(...)` veya `resolveMessageReceiptPrimaryId(...)` ile türetin.

Canlı ve sonlandırıcı yeteneklerini kesin biçimde bildirin; çekirdek bir kanalın
neler yapabileceğine karar vermek için bunları kullanır ve bildirilen davranışla
gerçek davranış arasındaki sapma bir sözleşme testi hatasıdır:

| Yüzey                                | Değerler                                                                                         |
| ------------------------------------ | ------------------------------------------------------------------------------------------------ |
| `message.live.capabilities`                   | `draftPreview`, `previewFinalization`, `progressUpdates`, `nativeStreaming`, `quietFinalization` |
| `message.live.finalizer.capabilities`                   | `finalEdit`, `normalFallback`, `discardPending`, `previewReceipt`, `retainOnAmbiguousFailure` |

Taslak önizlemesini yerinde sonlandıran kanallar, çalışma zamanı mantığını
`defineFinalizableLivePreviewAdapter(...)` ile `deliverWithFinalizableLivePreviewAdapter(...)` üzerinden yönlendirmeli ve
yerel önizleme, ilerleme, düzenleme, geri dönüş/saklama, temizleme ve alındı
belgesi davranışının sessizce sapmaması için bildirilen yetenekleri
`verifyChannelMessageLiveCapabilityAdapterProofs(...)` ve `verifyChannelMessageLiveFinalizerProofs(...)` testleriyle desteklemelidir.

Platform onaylarını erteleyen gelen ileti alıcıları, onay zamanlamasını izleyiciye
özgü durumda gizlemek yerine `message.receive.defaultAckPolicy` ve `supportedAckPolicies`
bildiriminde bulunmalıdır. Bildirilen her politikayı `verifyChannelMessageReceiveAckPolicyAdapterProofs(...)` ile
kapsayın.

`dispatchInboundReplyWithBase` ve `recordInboundSessionAndDispatchReply` gibi eski yanıt yardımcıları
uyumluluk yönlendiricileri için kullanılabilir olmaya devam eder. Bunları yeni
kanal kodunda kullanmayın; bunun yerine `message` adaptörü, alındı
belgeleri ve `openclaw/plugin-sdk/channel-outbound` üzerindeki alma/gönderme yaşam döngüsü
yardımcılarıyla başlayın.

### Gelen giriş (deneysel)

Gelen yetkilendirmeyi taşıyan kanallar, çalışma zamanı alma yollarındaki deneysel
`openclaw/plugin-sdk/channel-ingress-runtime` alt yolunu kullanabilir. Platform olgularını, ham izin
listelerini, rota tanımlayıcılarını, komut olgularını ve erişim grubu
yapılandırmasını kabul eder; ardından sıralı giriş grafiğiyle birlikte
gönderen/rota/komut/etkinleştirme izdüşümlerini döndürür. Bu sırada platform
araması ve yan etkiler pluginde kalır. Plugin kimliği normalleştirmesini
çözümleyiciye ilettiğiniz tanımlayıcıda tutun; çözümlenen durumdan veya karardan
ham eşleşme değerlerini serileştirmeyin. API tasarımı, sorumluluk sınırı ve test
beklentileri için [Kanal giriş API'sine](/tr/plugins/sdk-channel-ingress) bakın.

### Kalıcı giriş ve yeniden oynatma tekilleştirmesi

Kalıcı girişi benimseyen kanallar, önemli ölçüde farklı bir kabul veya pompa
sözleşmesine ihtiyaç duymadıkları sürece `openclaw/plugin-sdk/channel-outbound` içindeki
`createChannelIngressMonitor` öğesini kullanmalıdır. Ham aktarım zarfını tek bir alma dar
boğazında kuyruğa alın (alma sırasında normalleştirme yapmayın), Webhook
aktarımları için aktarım onayını kalıcı eklemeye bağlayın, konuşma başına bir
serileştirilmiş şerit türetin ve yönlendirme benimsendiğinde olayı tamamlandı
olarak işaretleyin. Kuyruğun birincil anahtarı `(queue_name, event_id)` değeridir ve
tamamlama işlemi satırı silmek yerine bir mezar taşı oluşturur; böylece aynı
`event_id` değerinin platform tarafından geç yeniden teslim edilmesi
meza taşı saklama süresi boyunca kalıcı olarak reddedilir. İzleyici API'si ve
kapatma sözleşmesi için [Kanal giden API'sine](/tr/plugins/sdk-channel-outbound#durable-ingress-monitors)
bakın.

Bu mezar taşı, yeniden oynatma korumaları (`openclaw/plugin-sdk/persistent-dedupe`) için katmanlama
kuralıdır: boşaltılan bir kanal yalnızca korumanın kimliği veya saklama süresi
kuyruğunkini aştığında ayrı bir yeniden oynatma koruması tutar — aktarım teslimat
kimliğinden farklı bir mantıksal mesaj anahtarı (Telegram,
`chat_id:message_id` değerlerini tekilleştirir; çünkü geri sekme birleştirmeleri
bir mesajı yeni bir `update_id` altında yeniden ortaya çıkarabilir) veya
kanalın mezar taşı saklama süresinden daha uzun bir zaman aralığı. Koruma
anahtarınız boşaltma `event_id` değerine eşit olacaksa boşaltmayı
benimserken korumayı silin ve bunun yerine `completedTtlMs`/`completedMaxEntries`
boyutlarını eski koruma zaman aralığını kapsayacak şekilde ayarlayın.
Yaş sınırları gibi tekilleştirme dışı korumalar bu kuralla ilgili değildir.
Kararlı giden mesaj kimlikleri, kanala özgü bir TTL önbelleği yerine
`openclaw/plugin-sdk/channel-outbound` içindeki paylaşılan giden yankı kaydını kullanır.

#### Aktarım sınıfları ve saklama

Bir aktarımı, alma sınırındaki kurtarma garantisine göre sınıflandırın:

- **Onay kapılı Webhook veya olay teslimatı:** yalnızca kalıcı eklemeden sonra
  onay verin veya başarı döndürün. Ekleme hatası, teslimatı yeniden denemeye
  uygun bırakmalı veya alma sınırının başarısız olmasına neden olmalıdır. Bu
  sınıf Slack, SMS, Zalo, Microsoft Teams, Google Chat, LINE ve Synology Chat'i
  içerir.
- **Beklenen yoklama veya akış teslimatı:** uzak imleci ilerletin veya aktarım
  onayını yalnızca eklemeden sonra gönderin. Açık bir imleç yoksa bir ekleme
  hatasının alma döngüsünün öne geçmesine izin vermemesi için alma geri çağrısını
  serileştirilmiş ve beklenen durumda tutun. Telegram yoklaması, Signal ve Tlon
  bu sınıfı kullanır; Telegram Webhook teslimatı yukarıdaki onay kapılı kurala
  uyar.
- **Yeniden oynatılamayan soketler:** IRC, Mattermost, Twitch ve Zalo Personal,
  platformdan kabul edilen bir olayı yeniden teslim etmesini isteyemez. Kalıcı
  kuyrukları, süreç çökmesi zaman aralığına karşı koruma sağlar ve yerel yeniden
  başlatma kurtarmasını destekler; tamamlama mezar taşları platform yeniden
  oynatmasına karşı neredeyse etkisizdir.

Filo mezar taşı TTL geleneği olarak 30 gün kullanın; bunu SDK varsayılanı olarak
kullanmayın. Yüksek hacimli yeniden teslimat zaman aralığı normalde 20.000
girdilik tamamlanmış öğe sınırı kullanır; daha düşük hacimli beklenen ve yeniden
oynatılamayan aktarımlar normalde 1.000-2.000 kullanır. Mevcut istisnalar arasında
LINE'ın 4.096 girdilik sınırları, SMS'in 24 saatlik tamamlanmış TTL'si ve Tlon'un
yalnızca sınırla belirlenen tamamlanmış öğe saklaması bulunur. Başarısız satır
sınırları da tamamlanmış öğe sınırlarından daha düşük olabilir. Hem TTL hem de
sınır satırları budar; dolayısıyla etkin saklama ilk sınıra ulaşıldığında sona
erer. Yalnızca belgelenmiş bir platform yeniden deneme ufku, korunmuş yayımlanmış
yeniden oynatma koruması zaman aralığı, beklenen hacim veya disk bütçesi ya da
yeniden oynatılamayan aktarım için bundan sapın ve saklama sözleşmesini testlerle
kapsayın.

#### En az bir kez yan etkiler

Boşaltma yönlendirmesi, giriş satırı tamamlama mezar taşına ulaşmadan önce komut
yan etkilerini çalıştırır. Bu adımlar arasındaki bir süreç çökmesi satırı yeniden
oynatır ve yan etkinin yeniden yürütülmesine neden olabilir. Bu en az bir kez
çökme zaman aralığı varsayılan sözleşmedir. Yapılandırma yazmaları, depolama
temizlemeleri veya yanıt şeridi dışındaki görünür onaylar gibi idempotent olmayan
işler için `openclaw/plugin-sdk/ingress-effect-once` içindeki `createIngressEffectOnce(...)` öğesini kullanın.
Her çağrıya kararlı giriş `eventId` değerini ve bir etki adını verin.
Her giriş kuyruğu/hesabı için bir yardımcı oluşturun ve aktarım olay kimlikleri
kuyruğa özgü olabileceğinden bu kapsam için kararlı, benzersiz bir
`namespacePrefix` kullanın. Yardımcı, kalıcı talebini yalnızca etki başarıyla
tamamlandıktan sonra işler; fırlatılan bir etki talebi serbest bırakır, böylece
boşaltma yeniden denemesi onu tekrar yürütebilir, eşzamanlı çağıranlar ise etkin
talebi bekler. Kalıcı durum hataları, sağlandığında `onDiskError` öğesini
çağırır ve süreç belleğine geri dönmek yerine isteği reddeder.

Yardımcının `ttlMs` değerini, kanalın giriş mezar taşı saklama
süresine ek olarak etki işlemesi ile satır tamamlama arasındaki, sınırlı kapalı
kalma süresi ve boşaltma yeniden denemeleri dâhil, azami gecikmeye eşit veya daha
yüksek ayarlayın. Etki kaydının TTL'si işleme anında başlar, mezar taşı saklama
süresi ise daha sonra tamamlanma anında başlar; bekleyen satırın ömrü sınırsızsa
hiçbir sonlu TTL rastgele bir kapalı kalma süresini kapsayamaz. Mezar taşı artık
satırı yeniden oynatamadığında daha eski etki kayıtları gereksizdir.
`stateMaxEntries` boyutunu, kuyruğun tamamlanmış girdi sınırını ve olay başına
azami etki sayısını hesaba katarak bu saklama zaman aralığında bulunabilecek her
farklı olay/etki anahtarı için ayarlayın. Daha düşük bir sınır, en eski kaydı
TTL'sinden önce çıkarır ve bu etkinin yeniden yürütülmesine izin verir. Süreç,
etki başarıyla tamamlandıktan sonra ancak talep işlenmeden önce sonlanırsa;
kalıcılık başarısız olursa veya giriş satırı hâlâ beklerken kaydın süresi dolarsa
en az bir kez yürütme zaman aralıkları kalmaya devam eder.

#### Hesap kapsamlı yeniden başlatma sözleşmesi

Kanal yapılandırması değişiklikleri varsayılan olarak tüm kanalı yeniden
başlatır. Çok hesaplı bir kanal, yalnızca yapılandırma çözümlemesi kanal genelinde
paylaşılan alanları ve seçili hesabı okuyup hiçbir zaman eşdüzey bir hesabı
okumadığında ve Gateway, eşdüzey çalışma zamanlarını değiştirmeden tek bir
`(channel, accountId)` çalışma zamanını durdurup başlatabildiğinde
`reload.accountScopedRestart: true` ayarlayabilir.

Kapsamlı yol yalnızca `channels.<channel>.accounts.<non-default-id>.*` altındaki değişikliklere uygulanır.
Paylaşılan kanal alanlarındaki, `accounts.default` içindeki, kaldırılmış veya
çözümlenemeyen hesaplardaki değişiklikler ve kalıtımı etkileyebilecek karma
değişiklikler tüm kanalın yeniden başlatılmasına yükseltilir. Bu özelliği
etkinleştirmeyen pluginler her zaman tüm kanal yolunu kullanır.

Kalıcı giriş boşaltmasını kullanan kanallarda hesap izleyicisinin durdurma yolu,
önce kabul edilen tüm aktarım kabullerini sonuçlandırmalı, ardından boşaltmasını
elden çıkarıp tamamlanmasını beklemelidir. Hesabın başlatılması, hesap anahtarlı
aynı kuyruğu açar ve ilk boşaltma, yönlendirilmemiş kalıcı satırları kurtarır.
Yeniden yüklemeye özgü ikinci bir yeniden oynatma geçişi eklemeyin; kuyruk
kurtarma, standart yeniden başlatma yoludur.

Bu bayrağı performans tercihi olarak değil, bir yetenek beyanı olarak
değerlendirin. Sözleşme testleri, adlandırılmış bir hesabı eklemenin ve
düzenlemenin eşdüzey hesabın çözümlenen yapılandırmasını değiştirmediğini; bir
hesabı durdurmanın yalnızca o hesabın izleyicisini ve boşaltmasını
sonuçlandırdığını ve yeni bir izleyicinin o hesabın satırlarını tam olarak bir
kez kurtardığını kanıtlamalıdır. Herhangi bir garanti kanıtlanamıyorsa bayrağı
kullanmayın.

### Yazıyor göstergeleri

Kanalınız gelen yanıtlardan bağımsız yazıyor göstergelerini destekliyorsa kanal
plugininde `heartbeat.sendTyping(...)` öğesini sunun. Çekirdek, Heartbeat model
çalışması başlamadan önce bunu çözümlenmiş Heartbeat teslimat hedefiyle çağırır
ve paylaşılan yazıyor göstergesini canlı tutma/temizleme yaşam döngüsünü
kullanır. Platform açık bir durdurma sinyali gerektiriyorsa
`heartbeat.clearTyping(...)` ekleyin.

### Medya kaynağı parametreleri

Kanalınız medya kaynakları taşıyan mesaj aracı parametreleri ekliyorsa bu
parametre adlarını `plugin.actions.describeMessageTool(...).mediaSourceParams` üzerinden sunun.
Çekirdek, korumalı alan yolu normalleştirmesi ve giden medya erişim politikası
için bu açık listeyi kullanır; böylece pluginler sağlayıcıya özgü avatar, ek
veya kapak görseli parametreleri için paylaşılan çekirdekte özel durumlara
ihtiyaç duymaz.

Her eylem için ayrı anahtar içeren `{ "set-profile": ["avatarUrl", "avatarPath"] }` gibi bir eşlemeyi tercih edin;
böylece ilgisiz eylemler başka bir eylemin medya bağımsız değişkenlerini devralmaz. Düz bir dizi,
sunulan tüm eylemler arasında kasıtlı olarak paylaşılan parametreler için hâlâ kullanılabilir.

Platform tarafındaki medya alımı için geçici bir herkese açık URL sunması gereken
kanallar, Plugin durum depolarıyla `openclaw/plugin-sdk/outbound-media`
üzerinden `createHostedOutboundMediaStore(...)` kullanabilir. Platform
rota ayrıştırmasını ve belirteç uygulamasını kanal Plugin'inde tutun; paylaşılan yardımcı
yalnızca medya yükleme, süre sonu meta verileri, parça satırları ve temizliğin sahibidir.

Gelen ekler, paralel `Media*` alanları değil, sıralı olgular kullanır. Kanal
kayıtlarını `openclaw/plugin-sdk/channel-inbound`
üzerinden `toInboundMediaFacts(...)` ile normalleştirin ve gelen bağlamı
oluştururken bunları `media` olarak iletin. Bir Plugin'in yerel medya okumalarını
yetkilendirmesi gerektiğinde, odaklanmış
`openclaw/plugin-sdk/media-local-roots` alt yolundan
`getAgentScopedMediaLocalRoots(...)` veya
`getAgentScopedMediaLocalRootsForSources(...)` içe aktarın. Eski
`agent-media-payload` oluşturucusu/kök cephesi, kullanımdan kaldırılmış uyumluluk katmanıdır.

### Yerel yük biçimlendirme

Kanalınızın `message(action="send")` için sağlayıcıya özgü biçimlendirmeye ihtiyacı varsa,
`actions.prepareSendPayload(...)` tercih edin. Yerel kartları, blokları, gömmeleri veya
diğer kalıcı verileri `payload.channelData.<channel>` altına yerleştirin ve çekirdeğin
giden/ileti bağdaştırıcısı üzerinden göndermesine izin verin. `actions.handleAction(...)` değerini yalnızca
serileştirilemeyen ve yeniden denenemeyen yükler için uyumluluk geri dönüşü
olarak kullanın.

### Oturum konuşması dil bilgisi

Platformunuz konuşma kimliklerinin içinde ek kapsam depoluyorsa, bu ayrıştırmayı
`messaging.resolveSessionConversation(...)` ile Plugin içinde tutun. Bu,
`rawId` değerini temel konuşma kimliğine, isteğe bağlı
ileti dizisi kimliğine, açık `baseConversationId` değerine ve herhangi bir
`parentConversationCandidates` değerine eşlemek için standart kancadır. `parentConversationCandidates`
döndürdüğünüzde, bunları en dar üst öğeden en geniş/temel konuşmaya doğru sıralayın.

`messaging.resolveParentConversationCandidates(...)`, yalnızca genel/ham kimliğin üzerinde üst öğe geri dönüşlerine
ihtiyaç duyan Plugin'ler için kullanımdan kaldırılmış bir
uyumluluk geri dönüşüdür. Her iki kanca da varsa çekirdek önce
`resolveSessionConversation(...).parentConversationCandidates` kullanır ve yalnızca standart
kanca bunları atladığında `resolveParentConversationCandidates(...)` değerine
geri döner.

Kanal kayıt defteri başlatılmadan önce aynı ayrıştırmaya ihtiyaç duyan paketlenmiş
Plugin'ler, eşleşen bir `resolveSessionConversation(...)` dışa aktarımına sahip üst düzey
bir `session-key-api.ts` dosyası sunabilir (Feishu ve Telegram
Plugin'lerine bakın). Çekirdek, yalnızca çalışma zamanı Plugin
kayıt defteri henüz kullanılamadığında bu önyükleme açısından güvenli yüzeyi kullanır.

Plugin kodunun rota benzeri alanları normalleştirmesi, bir alt ileti dizisini üst
rotasıyla karşılaştırması veya `{ channel, to, accountId, threadId }` üzerinden kararlı bir
yinelenenleri ayıklama anahtarı oluşturması gerektiğinde `openclaw/plugin-sdk/channel-route` kullanın. Yardımcı,
sayısal ileti dizisi kimliklerini çekirdekle aynı şekilde normalleştirir; bu nedenle geçici
`String(threadId)` karşılaştırmaları yerine onu tercih edin. Sağlayıcıya özgü hedef dil bilgisine
sahip Plugin'ler, çekirdeğin ayrıştırıcı uyumluluk katmanları olmadan sağlayıcıya
özgü oturum ve ileti dizisi kimliğini alabilmesi için `messaging.resolveOutboundSessionRoute(...)` sunmalıdır.

### Hesap kapsamlı konuşma bağlama desteği

Kanal genel geçerli konuşma bağlamalarını desteklediğinde
`conversationBindings.supportsCurrentConversationBinding` ayarlayın. `createChatChannelPlugin(...)`,
bu statik yeteneği varsayılan olarak `true` değerine ayarlar.

Destek yapılandırılmış hesaba göre değişiyorsa ayrıca
`conversationBindings.isCurrentConversationBindingSupported({ accountId })` uygulayın.
Çekirdek bu eşzamanlı kancayı yalnızca statik yetenek etkinleştirildikten sonra
değerlendirir. `false` döndürülmesi, genel geçerli konuşma yeteneği ile
bağlama, arama, listeleme, dokunma ve bağ kaldırma işlemlerini o hesap için kullanılamaz
hâle getirir. Kancanın atlanması, statik yeteneği her hesaba uygular.

Yanıtı önceden yüklenmiş hesap yapılandırmasından veya çalışma zamanı durumundan çözümleyin. Bu
kanca yalnızca genel geçerli konuşma bağlamalarını denetler; yapılandırılmış
bağlama kurallarının veya Plugin'e ait oturum yönlendirmesinin yerini almaz. Sözleşme testleri,
`openclaw/plugin-sdk/channel-core` tarafından dışa aktarılan
`ChannelPlugin["conversationBindings"]` sözleşmesi üzerinden en az bir desteklenen ve bir desteklenmeyen hesabı
kapsamalıdır.

## Onaylar ve kanal yetenekleri

Çoğu kanal Plugin'i onaya özgü koda ihtiyaç duymaz. Çekirdek, aynı sohbet
`/approve`, paylaşılan onay düğmesi yükleri ve genel geri dönüş teslimatının sahibidir.
`ChannelPlugin.approvals` kaldırıldı; bunun yerine onay teslimatı/yerel/işleme/yetkilendirme
olgularını tek bir `approvalCapability` nesnesine yerleştirin. `plugin.auth` yalnızca
oturum açma/kapatma içindir; çekirdek artık bu nesneden onay yetkilendirme kancalarını okumaz.

`approvalCapability.delivery` değerini yalnızca yerel onay yönlendirmesi veya geri dönüş
baskılama için, `approvalCapability.render` değerini ise yalnızca bir kanalın paylaşılan işleyici
yerine gerçekten özel onay yüklerine ihtiyaç duyduğu durumlarda kullanın.

### Onay yetkilendirmesi

- `approvalCapability.authorizeActorAction` ve
  `approvalCapability.getActionAvailabilityState` standart
  onay yetkilendirme bağlantı noktasıdır.
- Aynı sohbet onay yetkilendirmesi kullanılabilirliği için `getActionAvailabilityState` kullanın.
  Yerel teslimat devre dışı olsa bile yapılandırılmış onaylayıcıları `/approve` için
  kullanılabilir tutun; teslimat/kurulum rehberliği için bunun yerine yerel başlatma yüzeyi
  durumunu kullanın.
- Kanalınız yerel yürütme onayları sunuyorsa, aynı sohbet
  onay yetkilendirmesinden farklı olduğunda başlatma yüzeyi/yerel istemci durumu için
  `approvalCapability.getExecInitiatingSurfaceState` kullanın.
  Çekirdek, `enabled` ile
  `disabled` arasında ayrım yapmak, başlatan kanalın yerel yürütme
  onaylarını destekleyip desteklemediğine karar vermek ve kanalı yerel istemci geri dönüş
  rehberliğine dâhil etmek için yürütmeye özgü bu kancayı kullanır.
  `createApproverRestrictedNativeApprovalCapability(...)`, yaygın durum için bunu
  doldurur.
- Bir kanal mevcut yapılandırmadan kararlı, sahip benzeri DM kimlikleri çıkarabiliyorsa,
  onaya özgü çekirdek mantığı eklemeden aynı sohbet `/approve`
  değerini kısıtlamak için `openclaw/plugin-sdk/approval-runtime` üzerinden
  `createResolvedApproverActionAuthAdapter` kullanın.
- Özel onay yetkilendirmesi kasıtlı olarak yalnızca aynı sohbet geri dönüşüne izin veriyorsa
  `openclaw/plugin-sdk/approval-auth-runtime` üzerinden
  `markImplicitSameChatApprovalAuthorization({ authorized: true })` döndürün; aksi takdirde çekirdek sonucu
  açık onaylayıcı yetkilendirmesi olarak değerlendirir.
- Kanala ait yerel bir geri çağırma onayları doğrudan çözümlüyorsa, örtük
  geri dönüşün yine de kanalın normal aktör yetkilendirmesinden geçmesi için çözümlemeden önce
  `isImplicitSameChatApprovalAuthorization(...)` kullanın.

### Yük yaşam döngüsü ve kurulum rehberliği

- Yinelenen yerel onay istemlerini gizleme veya teslimattan önce yazıyor
  göstergeleri gönderme gibi kanala özgü yük yaşam döngüsü davranışları için
  `outbound.shouldSuppressLocalPayloadPrompt` ya da
  `outbound.beforeDeliverPayload` kullanın.
- Kanal, devre dışı bırakılmış yol yanıtının yerel yürütme
  onaylarını etkinleştirmek için gereken tam yapılandırma ayarlarını açıklamasını istediğinde
  `approvalCapability.describeExecApprovalSetup` kullanın. Kanca `{ channel, channelLabel, accountId }` alır;
  adlandırılmış hesap kanalları üst düzey
  varsayılanlar yerine `channels.<channel>.accounts.<id>.execApprovals.*` gibi hesap kapsamlı yolları
  işlemelidir.
- Plugin onay hatası rehberliğinin, Plugin onayının rota bulunamaması ve zaman aşımı
  hataları için gösterilmesi güvenli olduğunda `approvalCapability.describePluginApprovalSetup` kullanın.
  `createApproverRestrictedNativeApprovalCapability(...)` bunu
  `describeExecApprovalSetup` üzerinden çıkarmaz; aynı yardımcıyı yalnızca Plugin ve yürütme
  onayları gerçekten aynı yerel kurulumu kullandığında açıkça iletin.

### Yerel onay teslimatı

Bir kanal yerel onay teslimatına ihtiyaç duyuyorsa kanal kodunu hedef
normalleştirme ile aktarım/sunum olgularına odaklı tutun.
`openclaw/plugin-sdk/approval-runtime` üzerinden
`createChannelExecApprovalProfile`, `createChannelNativeOriginTargetResolver`,
`createChannelApproverDmTargetResolver` ve
`createApproverRestrictedNativeApprovalCapability` kullanın. Kanala özgü olguları
`approvalCapability.nativeRuntime` arkasına, ideal olarak
`createChannelApprovalNativeRuntimeAdapter(...)` veya
`createLazyChannelApprovalNativeRuntimeAdapter(...)` üzerinden yerleştirin; böylece çekirdek
işleyiciyi birleştirebilir ve istek filtreleme, yönlendirme, yinelenenleri ayıklama, süre sonu, Gateway
aboneliği ve başka yere yönlendirildi bildirimlerinin sahibi olabilir.

`nativeRuntime` birkaç küçük bağlantı noktasına ayrılmıştır:

- `availability` - hesabın yapılandırılmış olup olmadığı ve bir isteğin
  işlenip işlenmeyeceği
- `presentation` - paylaşılan onay görünüm modelini
  bekleyen/çözümlenen/süresi dolan yerel yüklere veya son eylemlere eşleme
- `transport` - hedefleri hazırlama ve yerel onay
  iletilerini gönderme/güncelleme/silme
- `interactions` - yerel düğmeler
  veya tepkiler için isteğe bağlı bağlama/bağ kaldırma/eylem temizleme kancaları ve isteğe bağlı bir `cancelDelivered` kancası. `deliverPending` işlem içi veya kalıcı
  durum (tepki hedefi deposu gibi) kaydediyorsa `cancelDelivered` uygulayın;
  böylece bir işleyicinin durdurulması teslimatı `bindPending` çalışmadan önce iptal ederse
  ya da `bindPending` hiçbir tanıtıcı döndürmezse bu durum serbest bırakılabilir
- `observe` - isteğe bağlı teslimat tanılama kancaları

Diğer onay yardımcıları:

- Bir kanal hem oturum kaynağından yerel teslimatı hem de açık onay iletme hedeflerini
  desteklediğinde `openclaw/plugin-sdk/approval-native-runtime` üzerinden
  `createNativeApprovalChannelRouteGates` kullanın. Yardımcı; onay yapılandırması seçimini,
  `mode` işlemeyi, aracı/oturum filtrelerini, hesap bağlamayı, oturum-hedef
  eşleştirmeyi ve hedef listesi eşleştirmeyi merkezîleştirirken çağıranlar kanal kimliği,
  varsayılan iletme modu, hesap araması, aktarımın etkin olup olmadığı denetimi, hedef
  normalleştirme ve tur kaynağı hedef çözümlemesinin sahibi olmaya devam eder. Bunu
  çekirdeğe ait kanal ilkesi varsayılanları oluşturmak için kullanmayın; kanalın belgelenmiş
  varsayılan modunu açıkça iletin.
- `createChannelNativeOriginTargetResolver`, `{ to, accountId, threadId }` hedefleri için varsayılan olarak
  paylaşılan kanal rotası eşleştiricisini kullanır.
  `targetsMatch` değerini yalnızca bir kanalın Slack zaman damgası öneki eşleştirmesi gibi
  sağlayıcıya özgü eşdeğerlik kuralları olduğunda iletin. Kanalın, özgün hedefi teslimat için
  korurken varsayılan rota eşleştiricisi veya özel bir `targetsMatch` geri çağırması
  çalışmadan önce sağlayıcı kimliklerini standartlaştırması gerektiğinde `normalizeTargetForMatch` iletin.
  `normalizeTarget` değerini yalnızca çözümlenen teslimat hedefinin kendisi
  standartlaştırılacaksa kullanın.
- Kanal istemci, belirteç, Bolt
  uygulaması veya Webhook alıcısı gibi çalışma zamanına ait nesnelere ihtiyaç duyuyorsa bunları
  `openclaw/plugin-sdk/channel-runtime-context` üzerinden kaydedin. Genel çalışma zamanı bağlamı
  kayıt defteri, çekirdeğin onaya özgü sarmalayıcı bağlantı kodu eklemeden kanal
  başlangıç durumundan yetenek odaklı işleyicileri önyüklemesini sağlar.
- Daha düşük düzeyli `createChannelApprovalHandler` veya
  `createChannelNativeApprovalRuntime` değerlerine yalnızca yetenek odaklı bağlantı noktası
  henüz yeterince ifade gücüne sahip olmadığında başvurun.
- Yerel onay kanalları hem `accountId` hem de `approvalKind`
  değerlerini bu yardımcılar üzerinden yönlendirmelidir. `accountId`, çok hesaplı onay ilkesini
  doğru bot hesabıyla kapsamlı tutar; `approvalKind` ise çekirdekte sabit kodlanmış dallar
  olmadan yürütme ve Plugin onayı davranışını kanal için kullanılabilir tutar.
- Onay yeniden yönlendirme bildirimlerinin sahibi de çekirdektir. Kanal Plugin'leri
  `createChannelNativeApprovalRuntime` içinden kendi "onay DM'lere / başka bir kanala gitti" takip
  iletilerini göndermemelidir; bunun yerine paylaşılan onay yeteneği yardımcıları üzerinden doğru kaynak +
  onaylayıcı DM yönlendirmesini sunmalı ve başlatan sohbete herhangi bir bildirim göndermeden önce
  çekirdeğin gerçek teslimatları toplamasına izin vermelidir.
- Teslim edilen onay kimliği türünü uçtan uca koruyun. Yerel istemciler,
  yürütme ve Plugin onayı yönlendirmesini kanalın yerel durumundan tahmin etmemeli veya
  yeniden yazmamalıdır.
- Bu açık `approvalKind` değerini `resolveApprovalOverGateway` öğesine iletin. Bu,
  standart `approval.resolve` hizmetini kullanır ve başka bir yüzey önce yanıt verdiğinde
  kaydedilen kazananı döndürür. Eski açık `resolveMethod` girdisi
  komut destekli denetimler için korunur; yeni yerel eylemler bunu kullanmamalı veya
  türü bir kimlikten çıkarmamalıdır.
- Farklı onay türleri kasıtlı olarak farklı yerel
  yüzeyler sunabilir. Geçerli paketlenmiş örnekler: Matrix, yetkilendirmenin onay türüne göre
  farklılaşmasına yine de izin verirken yürütme ve Plugin onayları için aynı yerel DM/kanal
  yönlendirmesini ve tepki kullanıcı deneyimini korur; Slack ise yerel onay yönlendirmesini
  hem yürütme hem de Plugin kimlikleri için kullanılabilir tutar.
- `createApproverRestrictedNativeApprovalAdapter` hâlâ bir
  uyumluluk sarmalayıcısı olarak bulunur; ancak yeni kod yetenek oluşturucuyu tercih etmeli
  ve Plugin üzerinde `approvalCapability` sunmalıdır.

### Daha dar onay çalışma zamanı alt yolları

Yoğun kullanılan kanal giriş noktalarında, bu ailenin yalnızca bir bölümüne
ihtiyaç duyduğunuzda daha geniş `approval-runtime` varili yerine şu daha dar alt yolları tercih edin:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-reference-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

Benzer şekilde, tümüne ihtiyacınız olmadığında daha geniş kapsamlı yüzeyler yerine
`openclaw/plugin-sdk/reply-runtime`,
`openclaw/plugin-sdk/reply-dispatch-runtime`,
`openclaw/plugin-sdk/reply-reference` ve
`openclaw/plugin-sdk/reply-chunking` tercih edin.

### Kurulum alt yolları

- `openclaw/plugin-sdk/setup-runtime`, çalışma zamanı açısından güvenli kurulum yardımcılarını kapsar:
  `createSetupTranslator`, içe aktarımı güvenli kurulum yaması adaptörleri
  (`createPatchedAccountSetupAdapter`, `createEnvPatchedAccountSetupAdapter`,
  `createSetupInputPresenceValidator`), arama notu çıktısı,
  `promptResolvedAllowFrom`, `splitSetupEntries` ve devredilen
  kurulum proxy'si oluşturucuları.
- `openclaw/plugin-sdk/channel-setup`, isteğe bağlı yükleme kurulumu
  oluşturucularının yanı sıra kurulum açısından güvenli birkaç temel öğeyi kapsar: `createOptionalChannelSetupSurface`,
  `createOptionalChannelSetupAdapter`, `createOptionalChannelSetupWizard`,
  `DEFAULT_ACCOUNT_ID`, `createTopLevelChannelDmPolicy`,
  `setSetupChannelEnabled` ve `splitSetupEntries`.
- Daha geniş `openclaw/plugin-sdk/setup` bağlantı noktasını yalnızca
  `moveSingleAccountChannelSectionToDefaultAccount(...)` gibi daha ağır paylaşılan kurulum/yapılandırma
  yardımcılarına da ihtiyacınız olduğunda kullanın.

Kanalınız kurulum yüzeylerinde yalnızca "önce bu plugin'i yükleyin" mesajını
duyurmak istiyorsa `createOptionalChannelSetupSurface(...)` tercih edin. Oluşturulan
adaptör/sihirbaz, yapılandırma yazma ve sonlandırma işlemlerinde güvenli biçimde başarısız olur ve
doğrulama, sonlandırma ve doküman bağlantısı metninde yükleme gerekliliğine ilişkin
aynı mesajı yeniden kullanır.

Kanalınız ortam değişkeniyle yönetilen kurulumu veya kimlik doğrulamayı destekliyorsa bunu
kanal yapılandırma şeması ve kurulum tanımlayıcıları üzerinden sunun. Kanal çalışma zamanı `envVars` veya
yerel sabitlerini yalnızca operatöre yönelik metinler için kullanın.

Kanalınız plugin çalışma zamanı başlamadan önce `status`, `channels list`, `channels status` veya
SecretRef taramalarında görünebiliyorsa
`package.json` içine `openclaw.setupEntry` ekleyin. Bu giriş noktası, salt okunur komut
yollarında güvenli biçimde içe aktarılabilmeli ve bu özetler için gereken
kanal meta verilerini, kurulum açısından güvenli yapılandırma adaptörünü,
durum adaptörünü ve kanal gizli bilgisi hedefi meta verilerini döndürmelidir.
Kurulum girişinden istemcileri, dinleyicileri veya taşıma çalışma zamanlarını başlatmayın.

Ana kanal girişinin içe aktarma yolunu da dar tutun. Keşif,
kanalı etkinleştirmeden yetenekleri kaydetmek için girişi ve kanal plugin
modülünü değerlendirebilir. `channel-plugin-api.ts` gibi dosyalar
kurulum sihirbazlarını, taşıma istemcilerini, soket dinleyicilerini,
alt süreç başlatıcılarını veya hizmet başlatma modüllerini içe aktarmadan
kanal plugin nesnesini dışa aktarmalıdır. Bu çalışma zamanı parçalarını
`registerFull(...)` üzerinden yüklenen modüllere, çalışma zamanı ayarlayıcılarına
veya tembel yetenek adaptörlerine yerleştirin.

### Diğer dar kanal alt yolları

Diğer yoğun kanal yollarında daha geniş eski yüzeyler yerine dar
yardımcıları tercih edin:

- Çoklu hesap yapılandırması ve varsayılan hesap
  geri dönüşü için `openclaw/plugin-sdk/account-core`, `openclaw/plugin-sdk/account-id`,
  `openclaw/plugin-sdk/account-resolution` ve
  `openclaw/plugin-sdk/account-helpers`
- Gelen rota/zarf ve kaydetme-dağıtma
  bağlantıları için `openclaw/plugin-sdk/inbound-envelope` ve
  `openclaw/plugin-sdk/channel-inbound`
- Hedef ayrıştırma yardımcıları için `openclaw/plugin-sdk/channel-targets`
- Giden kimlik/gönderme temsilcileri ve türü belirlenmiş
  yük planlaması için `openclaw/plugin-sdk/channel-outbound`
- Giden bir rotanın açık bir
  `replyToId`/`threadId` değerini koruması veya temel oturum anahtarı hâlâ eşleşirken
  mevcut `:thread:` oturumunu kurtarması gerektiğinde
  `openclaw/plugin-sdk/channel-core` içindeki `buildThreadAwareOutboundSessionRoute(...)`. Sağlayıcı plugin'leri,
  platformlarında yerel ileti dizisi teslimi semantiği bulunduğunda önceliği,
  sonek davranışını ve ileti dizisi kimliği normalleştirmesini geçersiz kılabilir.
- İleti dizisi bağlama yaşam döngüsü ve adaptör
  kaydı için `openclaw/plugin-sdk/thread-bindings-runtime`

Yalnızca kimlik doğrulama kullanan kanallar genellikle varsayılan yolda kalabilir:
çekirdek onayları işler, plugin ise yalnızca giden/kimlik doğrulama yeteneklerini sunar.
Matrix, Slack, Telegram gibi yerel onay kanalları ve özel sohbet taşımaları,
kendi onay yaşam döngülerini oluşturmak yerine paylaşılan yerel yardımcıları
kullanmalıdır.

## Gelen bahsetme politikası

Gelen bahsetme işlemeyi iki katmana ayırın:

- plugin'e ait kanıt toplama
- paylaşılan politika değerlendirmesi

Bahsetme politikası kararları için `openclaw/plugin-sdk/channel-mention-gating` kullanın.
Yalnızca daha geniş gelen yardımcıları dışa aktarma paketine ihtiyacınız olduğunda
`openclaw/plugin-sdk/channel-inbound` kullanın.

Plugin'e özgü mantık için uygun olanlar:

- bota yanıt algılama
- alıntılanan botu algılama
- ileti dizisine katılım denetimleri
- hizmet/sistem mesajı hariç tutmaları
- bot katılımını kanıtlamak için gereken platforma özgü önbellekler

Paylaşılan yardımcı için uygun olanlar:

- `requireMention`
- açık bahsetme sonucu
- örtük bahsetme izin listesi
- komut atlaması
- nihai atlama kararı

Tercih edilen akış:

1. Yerel bahsetme olgularını hesaplayın.
2. Bu olguları `resolveInboundMentionDecision({ facts, policy })` içine aktarın.
3. Gelen geçidinizde `decision.effectiveWasMentioned`, `decision.shouldBypassMention` ve
   `decision.shouldSkip` kullanın.

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";
import { resolveChannelImplicitMentions } from "openclaw/plugin-sdk/channel-ingress-runtime";

const wasMentioned = matchesMentionWithExplicit({
  text,
  mentionRegexes,
  explicit: {
    hasAnyMention,
    isExplicitlyMentioned,
    canResolveExplicit,
  },
});

const facts = {
  canDetectMention: true,
  wasMentioned,
  hasAnyMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const implicitMentions = resolveChannelImplicitMentions({
  cfg,
  channel: channelId,
  accountId,
});

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    implicitMentions,
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`matchesMentionWithExplicit(...)` bir Boole değeri döndürür. `hasAnyMention`,
`isExplicitlyMentioned` ve `canResolveExplicit`, kanalın kendi
yerel bahsetme meta verilerinden (mesaj varlıkları, bota yanıt bayrakları ve benzerleri)
gelir; platformunuz bunları algılayamıyorsa `false`/`undefined`
değerlerini sağlayın.

`api.runtime.channel.mentions`, çalışma zamanı yerleştirmesine zaten bağımlı olan
paketle sunulan kanal plugin'leri için aynı paylaşılan bahsetme yardımcılarını sunar:
`buildMentionRegexes`, `matchesMentionPatterns`, `matchesMentionWithExplicit`,
`implicitMentionKindWhen`, `resolveInboundMentionDecision`.

Yalnızca `implicitMentionKindWhen` ve `resolveInboundMentionDecision` gerekiyorsa
ilgisiz gelen çalışma zamanı yardımcılarını yüklememek için
`openclaw/plugin-sdk/channel-mention-gating` üzerinden içe aktarın.

## Adım adım açıklama

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="Paket ve bildirim">
    Standart plugin dosyalarını oluşturun. Bir bildirimin bir kanala
    sahip olduğunu belirten, `openclaw.plugin.json` içindeki `channels` alanıdır
    (`kind` alanı değildir). Paket meta verilerinin tamamı için
    [Plugin Kurulumu ve Yapılandırması](/tr/plugins/sdk-setup#openclaw-channel) bölümüne bakın:

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "Acme Chat",
          "blurb": "OpenClaw'u Acme Chat'e bağlayın."
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "channels": ["acme-chat"],
      "name": "Acme Chat",
      "description": "Acme Chat kanal plugin'i",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {}
      },
      "channelConfigs": {
        "acme-chat": {
          "schema": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          },
          "uiHints": {
            "token": {
              "label": "Bot belirteci",
              "sensitive": true
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

    `configSchema`, `plugins.entries.acme-chat.config` değerini doğrular. Kanal hesabı
    yapılandırması olmayan, plugin'e ait ayarlar için bunu kullanın.
    `channelConfigs.acme-chat.schema`, `channels.acme-chat` değerini doğrular ve plugin
    çalışma zamanı yüklenmeden önce yapılandırma şeması, kurulum ve kullanıcı arayüzü
    yüzeyleri tarafından kullanılan soğuk yol kaynağıdır. Üst düzey alanların tamamına
    ilişkin başvuru için [Plugin bildirimi](/tr/plugins/manifest) bölümüne bakın.

  </Step>

  <Step title="Kanal plugin nesnesini oluşturun">
    `ChannelPlugin` arayüzünde birçok isteğe bağlı adaptör yüzeyi bulunur. En az
    `id`, `config` ve `setup` ile başlayın ve ihtiyaç
    duydukça adaptör ekleyin.

    `src/channel.ts` oluşturun:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        // Account resolution/inspection belongs on `config`, not `setup`.
        // `setup` covers onboarding writes (applyAccountConfig, validateInput).
        config: {
          listAccountIds: () => ["default"],
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
        setup: {
          applyAccountConfig: ({ cfg, input }) => ({
            ...cfg,
            channels: {
              ...cfg.channels,
              "acme-chat": { ...(cfg.channels as any)?.["acme-chat"], ...input },
            },
          }),
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          channel: "acme-chat",
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    Hem standart üst düzey DM anahtarlarını hem de eski iç içe anahtarları kabul eden kanallar için `plugin-sdk/channel-config-helpers` içindeki yardımcıları kullanın: `resolveChannelDmAccess`, `resolveChannelDmPolicy`, `resolveChannelDmAllowFrom` ve `normalizeChannelDmPolicy`, hesap yerelindeki değerleri devralınan kök değerlerin önünde tutar. Çalışma zamanı ile geçişin aynı sözleşmeyi okuması için aynı çözümleyiciyi `normalizeLegacyDmAliases` üzerinden doctor onarımıyla eşleştirin.

    <Accordion title="createChatChannelPlugin sizin için ne yapar">
      Düşük düzeyli bağdaştırıcı arayüzlerini elle uygulamak yerine,
      bildirimsel seçenekleri iletirsiniz ve oluşturucu bunları bir araya getirir:

      | Seçenek | Bağladığı bileşen |
      | --- | --- |
      | `security.dm` | Yapılandırma alanlarından kapsamlı DM güvenlik çözümleyicisi |
      | `pairing.text` | Kod değişimiyle metin tabanlı DM eşleştirme akışı |
      | `threading` | Yanıt modu çözümleyicisi (sabit, hesap kapsamlı veya özel) |
      | `outbound.attachedResults` | Sonuç meta verilerini (ileti kimlikleri) döndüren gönderme işlevleri; çekirdeğin döndürülen teslim sonucuna damga vurabilmesi için eş düzeyde bir `channel` kimliği gerektirir |

      Tam denetime ihtiyaç duyarsanız bildirimsel seçenekler yerine ham adaptör
      nesneleri de iletebilirsiniz.

      Ham giden adaptörler bir `chunker(text, limit, ctx)` işlevi tanımlayabilir.
      İsteğe bağlı `ctx.formatting`, `maxLinesPerMessage` gibi teslimat zamanı
      biçimlendirme kararlarını taşır; yanıt zincirleme ve parça sınırlarının
      paylaşılan giden teslimat tarafından tek seferde çözümlenmesi için bunu
      göndermeden önce uygulayın. Gönderme bağlamları, yerel bir yanıt hedefi
      çözümlendiğinde `replyToIdSource` (`implicit` veya `explicit`)
      öğesini de içerir; böylece yük yardımcıları, örtük ve tek kullanımlık bir
      yanıt yuvasını tüketmeden açık yanıt etiketlerini koruyabilir.
    </Accordion>

    ### Grup araç ilkesi adaptörleri

    `group.resolveToolPolicy` uygulayan ve
    `toolsBySender` desteği sunan bir kanal, eksiksiz `ChannelGroupContext` öğesini
    paylaşılan politika çözümleyicisine iletmelidir. Özellikle, temel
    `tools` politikasını uygulamaya devam ederken hem eşleşen grup hem de joker
    kapsamlarında göndericiye özgü katmanları atlayarak `senderPolicyMode: "never"`
    öğesine uymalıdır.

    OpenClaw bu modu yalnızca, gönderici yetkisinin sunucunun sahip olduğu bir zarf içinde
    önceden yakalandığı güvenilir, giriş dışı yürütmeler için ayarlar; açıkça
    sınırlandırılmış zamanlanmış bir çalıştırma buna örnektir. Plugin'ler bu modu
    gelen meta verilerden türetmemeli, kanal durumu olarak kalıcı hâle getirmemeli veya
    yapılandırma olarak sunmamalıdır. Modun, eşleşen temel `tools`
    kısıtlamasını kaldırmadan bir joker `toolsBySender` girdisini atladığını kanıtlayan
    bir adaptör testi ekleyin.

  </Step>

  <Step title="Giriş noktasını bağlayın">
    `index.ts` oluşturun:

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat kanal plugin'i",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat yönetimi");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme Chat yönetimi",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    Kanala ait CLI tanımlayıcılarını `registerCliMetadata(...)` içine yerleştirin; böylece OpenClaw,
    tam kanal çalışma zamanını etkinleştirmeden bunları kök yardımında gösterebilirken
    normal tam yüklemeler gerçek komut kaydı için aynı tanımlayıcıları almaya devam eder.
    `registerFull(...)` öğesini yalnızca çalışma zamanına özgü işler için kullanın.
    `defineChannelPluginEntry`, kayıt modu ayrımını otomatik olarak gerçekleştirir.
    `registerFull(...)` Gateway RPC yöntemlerini kaydediyorsa Plugin'e özgü bir
    ön ek kullanın. Çekirdek yönetim ad alanları (`config.*`,
    `exec.approvals.*`, `wizard.*`, `update.*`) ayrılmış olarak kalır ve her zaman
    `operator.admin` sonucuna çözümlenir. Tüm seçenekler için
    [Giriş Noktaları](/tr/plugins/sdk-entrypoints#definechannelpluginentry) bölümüne bakın.

  </Step>

  <Step title="Bir kurulum girdisi ekleyin">
    İlk katılım sırasında hafif yükleme için `setup-entry.ts` oluşturun:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    OpenClaw, kanal devre dışı veya yapılandırılmamış olduğunda tam giriş
    yerine bunu yükler. Kurulum akışları sırasında ağır çalışma zamanı kodunun
    yüklenmesini önler. Ayrıntılar için [Kurulum ve Yapılandırma](/tr/plugins/sdk-setup#setup-entry) bölümüne bakın.

    Kurulum için güvenli dışa aktarımları yardımcı
    modüllere ayıran paketlenmiş çalışma alanı kanalları, açık bir
    kurulum zamanı çalışma ortamı ayarlayıcısına da ihtiyaç duyduklarında
    `openclaw/plugin-sdk/channel-entry-contract` içindeki `defineBundledChannelSetupEntry(...)` öğesini kullanabilir.

  </Step>

  <Step title="Gelen mesajları işleme">
    Plugin'inizin platformdan mesajları alıp OpenClaw'a iletmesi gerekir.
    Tipik kalıp, isteği doğrulayan ve kanalınızın gelen ileti işleyicisi
    üzerinden yönlendiren bir Webhook'tur:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // Plugin tarafından yönetilen kimlik doğrulama (imzaları kendiniz doğrulayın)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // Gelen ileti işleyiciniz mesajı OpenClaw'a yönlendirir.
          // Tam bağlantı düzeni platform SDK'nıza bağlıdır -
          // gerçek bir örnek için paketlenmiş Microsoft Teams veya Google Chat Plugin paketine bakın.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      Gelen mesajların işlenmesi kanala özgüdür. Her kanal Plugin'i
      kendi gelen ileti işlem hattına sahiptir. Gerçek kalıplar için paketlenmiş kanal Plugin'lerine
      (örneğin Microsoft Teams veya Google Chat Plugin paketine) bakın.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="Test">
Ortak konumlu testleri `src/channel.test.ts` içinde yazın:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("acme-chat plugin", () => {
      it("resolves account from config", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.config.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("inspects account without materializing secrets", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("reports missing config", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test <bundled-plugin-root>/acme-chat/
    ```

    Paylaşılan test yardımcıları için [Test Etme](/tr/plugins/sdk-testing) bölümüne bakın.

</Step>
</Steps>

## Dosya yapısı

```text
<bundled-plugin-root>/acme-chat/
├── package.json              # openclaw.channel meta verileri
├── openclaw.plugin.json      # Yapılandırma şemasını içeren bildirim
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # Genel dışa aktarımlar (isteğe bağlı)
├── runtime-api.ts            # Dahili çalışma ortamı dışa aktarımları (isteğe bağlı)
└── src/
    ├── channel.ts            # createChatChannelPlugin aracılığıyla ChannelPlugin
    ├── channel.test.ts       # Testler
    ├── client.ts             # Platform API istemcisi
    └── runtime.ts            # Çalışma ortamı deposu (gerekirse)
```

## İleri düzey konular

<CardGroup cols={2}>
  <Card title="İş parçacığı seçenekleri" icon="git-branch" href="/tr/plugins/sdk-entrypoints#registration-mode">
    Sabit, hesap kapsamlı veya özel yanıt modları
  </Card>
  <Card title="Mesaj aracı entegrasyonu" icon="puzzle" href="/tr/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool ve eylem keşfi
  </Card>
  <Card title="Hedef çözümleme" icon="crosshair" href="/tr/plugins/architecture-internals#channel-target-resolution">
    inferTargetChatType, looksLikeId, reservedLiterals, resolveTarget
  </Card>
  <Card title="Çalışma zamanı yardımcıları" icon="settings" href="/tr/plugins/sdk-runtime">
    api.runtime aracılığıyla TTS, STT, medya ve alt ajan
  </Card>
  <Card title="Kanal gelen ileti API'si" icon="bolt" href="/tr/plugins/sdk-channel-inbound">
    Paylaşılan gelen olay yaşam döngüsü: alma, çözümleme, kaydetme, yönlendirme, sonlandırma
  </Card>
</CardGroup>

<Note>
Paketle gelen Plugin'lerin bakımı ve uyumluluk için paketle gelen bazı yardımcı
bağlantı noktaları hâlâ mevcuttur. Bunlar yeni kanal Plugin'leri için önerilen
kalıp değildir; söz konusu paketle gelen Plugin ailesinin bakımını doğrudan
yapmıyorsanız ortak SDK yüzeyindeki genel kanal/kurulum/yanıt/çalışma zamanı alt
yollarını tercih edin.
</Note>

## Sonraki adımlar

- [Sağlayıcı Plugin'leri](/tr/plugins/sdk-provider-plugins) - Plugin'iniz modeller de sağlıyorsa
- [SDK'ye genel bakış](/tr/plugins/sdk-overview) - alt yol içe aktarımlarının tam başvurusu
- [SDK testi](/tr/plugins/sdk-testing) - test yardımcı programları ve sözleşme testleri
- [Plugin manifesti](/tr/plugins/manifest) - tam manifest şeması

## İlgili

- [Plugin SDK kurulumu](/tr/plugins/sdk-setup)
- [Plugin oluşturma](/tr/plugins/building-plugins)
- [Ajan çalıştırma çerçevesi Plugin'leri](/tr/plugins/sdk-agent-harness)
