---
read_when:
    - Kanal gönderme veya alma davranışını yeniden düzenleme
    - Kanalın gelen iletilerini, yanıt yönlendirmesini, giden kuyruğunu, önizleme akışını veya Plugin SDK mesaj API'lerini değiştirme
    - Kalıcı gönderimler, alındı bilgileri, önizlemeler, düzenlemeler veya yeniden denemeler gerektiren yeni bir kanal plugini tasarlama
summary: 'Kalıcı mesaj alma/gönderme yaşam döngüsünün durumu: nelerin yayımlandığı, özgün tasarımdan nelerin değiştiği ve nelerin hâlâ açık olduğu'
title: Mesaj yaşam döngüsü yeniden düzenlemesi
x-i18n:
    generated_at: "2026-07-26T23:56:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d21eda70b8be0de78677f4ff6d7547317112731d9e86a5bef58eac0268899818
    source_path: concepts/message-lifecycle-refactor.md
    workflow: 16
---

<Note>
Bu sayfa ileriye dönük bir tasarım önerisi olarak ortaya çıktı. Bu tasarımın
özü o zamandan beri `src/channels/message/*` ve herkese açık
`openclaw/plugin-sdk/channel-outbound` / `channel-inbound` alt yollarında kullanıma sunuldu. Güncel
API için [Kanal giden API'si](/tr/plugins/sdk-channel-outbound) ve
[Kanal gelen API'si](/tr/plugins/sdk-channel-inbound) sayfalarını kullanın. Bu sayfa nelerin
kullanıma sunulduğunu, uygulamanın özgün taslaktan nerelerde ayrıldığını ve nelerin
hâlâ açık olduğunu izler.
</Note>

## Bu yeniden düzenleme neden yapıldı

Kanal yığını birkaç yerel düzeltmeden büyüdü: her olgunluk düzeyi için ayrı gelen
yardımcıları (basit bağdaştırıcılar için `runtime.channel.inbound.run`,
zengin olanlar için `runtime.channel.inbound.runPreparedReply`), eski yanıt dağıtım
yardımcıları (`dispatchInboundReplyWithBase`, `recordInboundSessionAndDispatchReply`),
kanala özgü önizleme akışı ve mevcut yanıt yükü yollarına sonradan eklenen
nihai teslimat dayanıklılığı. Bu yapı, çok fazla herkese açık kavram ve
teslimat semantiğinin birbirinden sapabileceği çok fazla nokta üretti.

Yeniden tasarımı zorunlu kılan güvenilirlik açığı:

```text
Telegram yoklama güncellemesi onaylandı
  -> asistanın nihai metni mevcut
  -> sendMessage başarılı olmadan önce süreç yeniden başlatılıyor
  -> nihai yanıt kayboluyor
```

Hedef değişmez: çekirdek, görünür bir giden mesajın var olması gerektiğine karar
verdiğinde, gönderme niyeti platform çağrısı denenmeden önce kalıcı olmalı ve
platform alındısı başarılı işlemden sonra kaydedilmelidir. Bu, varsayılan olarak
en az bir kez kurtarma sağlar. Tam olarak bir kez davranışı yalnızca bir bağdaştırıcının
yerel eşgüçlülüğü kanıtladığı veya gönderimden sonra sonucu bilinmeyen bir denemeyi
yeniden oynatmadan önce platform durumuyla uzlaştırdığı yerlerde bulunur.

## Kullanıma sunulanlar

Dahili etki alanı `src/channels/message/*` içinde bulunur:

| Dosya                        | Sorumluluğu                                                                                                               |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `types.ts`                  | Bağdaştırıcı, gönderme bağlamı, alındı ve kalıcı niyet türü sözleşmeleri                                                  |
| `send.ts`                   | `withDurableMessageSendContext` / `sendDurableMessageBatch` — kalıcı gönderme bağlamı                             |
| `receive.ts`                | `createMessageReceiveContext` — gelen onay politikası durum makinesi                                                   |
| `live.ts`                   | Canlı önizleme durumu ve yerinde sonlandırma veya geri dönüş mantığı                                                        |
| `state.ts`                  | `classifyDurableSendRecoveryState` — kesintiden sonra kurtarma sınıflandırması                                    |
| `receipt.ts`                | Platform gönderme sonuçlarını `MessageReceipt` biçimine normalleştirir                                                             |
| `capabilities.ts`           | Bir yükten gerekli kalıcı nihai özellikleri türetir                                                         |
| `contracts.ts`              | Bildirilen bağdaştırıcı özellikleri için sözleşme kanıtı doğrulaması                                                      |
| `adapter.ts`                | `defineChannelMessageAdapter`                                                                                      |
| `outbound-bridge.ts`        | `createChannelMessageAdapterFromOutbound` — eski `sendText`/`sendMedia`/`sendPayload`/`sendPoll` işlevlerini sarmalar |
| `ingress-queue.ts`          | `createChannelIngressQueue` — kalıcı gelen olay kuyruğu                                                          |
| `durable-receive.ts`        | `createDurableInboundReceiveJournal` — gelen tekilleştirmesi için kabul/beklemede/tamamlama/serbest bırakma günlüğü                  |
| `inbound-reply-dispatch.ts` | `dispatchChannelInboundReply` ve eski adlandırmalı sarmalayıcılar                                                            |
| `reply-pipeline.ts`         | `createChannelReplyPipeline`, yanıt öneki ve yazıyor geri çağırma yardımcıları                                             |

Herkese açık yüzey: `openclaw/plugin-sdk/channel-outbound` (gönderme/alındı/kalıcı/canlı/yanıt işlem hattı
yardımcıları) ve `openclaw/plugin-sdk/channel-inbound` (gelen bağlam, `runChannelInboundEvent`,
`dispatchChannelInboundReply`). Bağdaştırıcı örnekleri, güncel
tür adları ve geçiş notları için bu sayfalara bakın; API
yapısı için doğruluk kaynağı aşağıdaki taslaklar değil, bu sayfalardır.

### Gönderme bağlamı

`withDurableMessageSendContext`, kanal koduna tek bir giden mesaj etrafında `render`, `previewUpdate`,
`send`, `edit`, `delete`, `commit` ve `fail` adımlarını sağlar.
`sendDurableMessageBatch` yaygın durum sarmalayıcısıdır: işler, gönderir,
ardından `sent`/`suppressed` üzerinde kaydeder veya hatada başarısız olur.

`sendDurableMessageBatch` tek bir ayrıştırılmış sonuç döndürür:

| Durum           | Anlamı                                                                          |
| ---------------- | -------------------------------------------------------------------------------- |
| `sent`           | En az bir görünür platform mesajı teslim edildi                              |
| `suppressed`     | Hiçbir platform mesajı eksik sayılmamalıdır (kanca tarafından iptal edildi, deneme çalıştırması vb.) |
| `partial_failed` | Daha sonraki bir yük veya yan etki başarısız olmadan önce en az bir mesaj teslim edildi      |
| `failed`         | Hiçbir platform alındısı üretilmedi                                                 |

Dayanıklılık, `required`, `best_effort` veya `disabled`
seçeneklerinden biridir (`src/channels/message/types.ts` içinde `MessageDurabilityPolicy`). `required`,
kalıcı niyet yazılamadığında güvenli biçimde başarısız olur; `best_effort`, kalıcılık
kullanılamadığında doğrudan gönderime geçer; `disabled`, yeniden düzenleme
öncesindeki doğrudan gönderme davranışını korur. Eski uyumluluk yardımcıları varsayılan olarak
`disabled` kullanır ve bir kanalın genel bir giden
bağdaştırıcısı olması nedeniyle `required` çıkarımında bulunmaz.

Tehlikeli olmaya devam eden sınır: platform çağrısı başarılı olduktan sonra ve
alındı kaydedilmeden önce. Süreç bu noktada sonlanırsa bağdaştırıcı
`reconcileUnknownSend` bildirmediği sürece çekirdek platform mesajının
var olup olmadığını bilemez. Bu kanca, kesintiye uğrayan bir gönderimi `sent`, `not_sent` veya
`unresolved` olarak sınıflandırır; yalnızca `not_sent` yeniden oynatmaya izin verir. Uzlaştırması
olmayan kanallar `unknown_after_send` durumuna (`src/channels/message/state.ts`,
`src/infra/outbound/delivery-queue-recovery.ts`) geri döner ve yalnızca yinelenen görünür mesajlar
ilgili kanal için kabul edilebilir, belgelenmiş bir ödünleşimse en az bir kez
yeniden oynatmayı seçebilir.

### Alma bağlamı

`createMessageReceiveContext`, eşgüçlü bir `ack()` ve açık bir
`nack(error)` ile gelen olay başına onay/ret durumunu izler. Onay politikası
(`ChannelMessageReceiveAckPolicy`) şunlardan biridir:

| Politika                 | Onay zamanı                                                                                     |
| ---------------------- | --------------------------------------------------------------------------------------------- |
| `after_receive_record` | Çekirdek, yeniden teslimatı tekilleştirmek/yönlendirmek için yeterli gelen meta veriyi kalıcı hâle getirdiğinde                           |
| `after_agent_dispatch` | Aracı çalıştırması dağıtıldığında                                                             |
| `after_durable_send`   | Bu tur için kalıcı giden gönderme kaydedildiğinde                                             |
| `manual`               | Çağıran, onay zamanlamasını açıkça denetler (bir politika bildirmeyen bağdaştırıcılar için varsayılan) |

Telegram yoklaması, güvenli biçimde tamamlanmış bir güncelleme filigranını
kalıcı hâle getirmek için bunu kullanır (`extensions/telegram/src/bot-update-tracker.ts` içinde `safeCompletedUpdateId`):
grammY, ara yazılım zincirine girerken her güncellemeyi görmeye devam eder; ancak
OpenClaw, kalıcı yeniden başlatma filigranını yalnızca dağıtımı tamamlanan
güncellemelerin ötesine ilerletir; böylece başarısız veya hâlâ bekleyen güncellemeler
yeniden başlatmadan sonra yeniden oynatılır. Telegram'ın üst `getUpdates` ofseti
hâlâ grammY tarafından yönetilir; bu filigranın ötesindeki platform düzeyinde
yeniden teslimatı denetleyen tamamen kalıcı bir yoklama kaynağı oluşturulmamıştır
(bkz. Açık sorular).

### Canlı önizleme

`src/channels/message/live.ts`, önizleme/düzenleme/sonlandırmayı tek bir yaşam döngüsü olarak modeller:
`createLiveMessageState`, `markLiveMessagePreviewUpdated`,
`markLiveMessageFinalized`, `markLiveMessageCancelled` ve
`deliverFinalizableLivePreviewAdapter` (taslaktan nihai bir düzenleme oluşturur, bunu
uygular ve düzenleme mümkün olmadığında veya başarısız olduğunda normal gönderime
geri döner). `LiveMessageState.phase`, `idle | previewing | finalizing | finalized |
cancelled` değeridir; `canFinalizeInPlace`, bir önizlemenin yeni
bir gönderim yerine düzenleme yoluyla nihai mesaj olup olamayacağını belirler.

### Kalıcı alındılar

`MessageReceipt` (`src/channels/message/types.ts`), tek bir mantıksal gönderimdeki bir veya daha fazla
platform mesaj kimliğini `platformMessageIds` ve parça başına
`parts` (tür, dizin, ileti dizisi kimliği, yanıtlanan ileti kimliği) biçiminde normalleştirir. İleti dizileri
ve sonraki düzenlemeler için birincil kimlik saklanır. Çok parçalı teslimatları (metin
ve medya, parçalara ayrılmış metin, kart geri dönüşü) yeniden başlatma sonrasında
yeniden oynatılabilir ve yinelenenleri ayıklanabilir kılan budur.

### Herkese açık SDK'nın azaltılması

Yeniden düzenleme şunları bünyesine kattı veya kullanımdan kaldırdı: `reply-runtime`, `reply-dispatch-runtime`,
`reply-reference`, `reply-chunking`, herkese açık
API olarak sunulan `reply-payload` yardımcıları, `inbound-reply-dispatch`, `channel-reply-pipeline` ve eski
giden cephenin herkese açık kullanımlarının çoğu. `src/plugin-sdk/channel-message.ts` artık
`channel-outbound` / `channel-inbound` öğelerine işaret eden bir
`@deprecated` yeniden dışa aktarma varilidir; `channel.turn` çalışma zamanı takma adları kaldırıldı ve eski
`/plugins/sdk-channel-turn` belge sayfası
[Kanal gelen API'si](/tr/plugins/sdk-channel-inbound) sayfasına yönlendiriliyor. Yeni Plugin kodu
doğrudan `channel-outbound` ve `channel-inbound` öğelerini hedeflemelidir.

## Uygulamanın özgün tasarımdan ayrıldığı noktalar

Aşağıdaki tasarım taslağı hiçbir zaman tam anlamıyla açıklandığı biçimde kullanıma sunulmadı. Kayıt
tarihsel doğruluk için tutulmaktadır; bu tür adlarını güncel API olarak değerlendirmeyin.

- **`MessageOrigin` / `shouldDropOpenClawEcho` yok.** Özgün plan,
  Gateway arıza mesajlarında bir `source: "openclaw"` kaynak etiketi ve
  `allowBots` yetkilendirmesinden önce paylaşılan odalarda etiketli, bot tarafından yazılmış
  yankıları bırakan ortak bir koşul gerektiriyordu. Bu tür ve koşul
  kod tabanında mevcut değildir. `allowBots` gerçek bir kanal başına yapılandırma anahtarıdır (Slack,
  Discord, Google Chat ve diğerleri), ancak onu koruması amaçlanan kaynak etiketleme mekanizması
  hiçbir zaman oluşturulmadı. Bot özellikli odalarda Gateway arıza yankısının bastırılması,
  kullanıma sunulmuş bir garanti değil, açık bir eksik olmaya devam ediyor.
- **Birleşik `core.messages.receive/send/live/state` ad alanı yok.** Kullanıma
  sunulan işlevler, bir `core.messages.*` cephesinin arkasında olmak yerine doğrudan `src/channels/message/*`
  (`withDurableMessageSendContext`, `createMessageReceiveContext`,
  `createLiveMessageState`, `classifyDurableSendRecoveryState`) içinde bulunur.
- **Genel `ChannelMessage` / `MessageTarget` / `MessageRelation`
  normalleştirilmiş mesaj türü yok.** Çekirdek, platformdan bağımsız tek bir
  `kind: "reply" |
"followup" | "broadcast" | "system"` ilişkili mesaj biçimi yerine gönderme bağdaştırıcıları üzerinden hâlâ somut yanıt yüklerini
  (`ReplyPayload`) ve kanala özgü bağlamları geçirir.
- **Onay politikası adları taslaktan farklı.** Kullanıma sunulan:
  `after_receive_record | after_agent_dispatch | after_durable_send | manual`.
  Özgün taslak, Webhook zaman aşımı nedeni alanıyla birlikte `immediate | after-record | after-durable-send |
manual` kullanıyordu; bu yapı oluşturulmadı.
- **`DurableFinalDeliveryRequirementMap` özellik anahtarları, taslaktaki
  `MessageCapabilities` nesnesinin yerini aldı.** Özellikler, iç içe bir
  `text.chunking` / `attachments.voice` tarzı yapı yerine `verifyDurableFinalCapabilityProofs` aracılığıyla doğrulanan düz Boole bayraklarıdır (`text`,
  `media`, `poll`, `payload`, `silent`, `replyTo`, `thread`, `nativeQuote`,
  `messageSendingHooks`, `batch`, `reconcileUnknownSend`, `afterSendSuccess`,
  `afterCommit`).

## Somut geçiş tehlikeleri (hâlâ geçerli)

Kanala özgü bu yan etkiler yeniden düzenlemeden önceye dayanır ve yeni
gönderme yolları üzerinden çalışmaya devam etmelidir. Varsayımsal değildirler: her biri
bugün uygulanmış durumdadır ve kritik öneme sahiptir.

- **iMessage** (`extensions/imessage/src/monitor/echo-cache.ts`,
  `persisted-echo-cache.ts`): izleyici, başarılı bir gönderimden sonra gönderilen iletileri bir yankı
  önbelleğine kaydeder. Kalıcı nihai gönderimler yine de bu önbelleği
  doldurmalıdır; aksi takdirde OpenClaw kendi yanıtlarını gelen kullanıcı iletileri olarak yeniden alabilir.
- **Tlon** (`extensions/tlon/src/monitor/index.ts`): isteğe bağlı bir model
  imzası ekler ve grup yanıtlarından sonra katılım sağlanan ileti dizilerini kaydeder. Kalıcı
  teslimat bu etkileri atlamamalıdır.
- **Discord ve diğer hazırlanmış göndericiler** doğrudan teslimat ve
  önizleme davranışını zaten kendileri yönetir. Hazırlanmış
  göndericisi nihai iletileri gönderim bağlamı üzerinden açıkça yönlendirene kadar bir kanal uçtan uca kalıcı değildir;
  yalnızca genel bağdaştırıcının kapsam sağladığını varsaymayın.
- **Telegram sessiz geri dönüş teslimatı**, parçalama/geri dönüş
  projeksiyonundan sonra yalnızca ilk yükü değil, projeksiyonu yapılmış
  yük dizisinin tamamını teslim etmelidir.
- **LINE, Zalo, Nostr** ve benzeri yardımcı yollar; yanıt belirteci
  işleme, medya vekilleme, gönderilmiş ileti önbellekleri veya yalnızca geri çağrı hedeflerine sahip olabilir.
  Bu anlamlar gönderim bağdaştırıcısında temsil edilip
  testlerle kapsanana kadar kanal tarafından yönetilen teslimatta kalırlar.
- **Doğrudan DM yardımcıları**, tek doğru
  aktarım hedefi olan bir yanıt geri çağrısına sahip olabilir. Genel giden ileti mekanizması, ham
  platform alanlarından bir hedef tahmin edip bu geri çağrıyı atlamamalıdır.

## Hata sınıflandırması

Bağdaştırıcılar aktarım hatalarını `DeliveryFailureKind` tarzı kapalı
kategorilerde sınıflandırır (geçici, hız sınırı, kimlik doğrulama, izin, bulunamadı, geçersiz
yük, çakışma, iptal edildi, bilinmiyor). Temel politika:

- Geçici ve hız sınırı hatalarını yeniden deneyin.
- Bir işleme geri dönüşü yoksa geçersiz yük hatalarını yeniden denemeyin.
- Yapılandırma değişene kadar kimlik doğrulama veya izin hatalarını yeniden denemeyin.
- Bulunamadı durumunda, kanal bunun güvenli olduğunu bildiriyorsa canlı sonlandırmanın
  düzenlemeden yeni bir gönderime geri dönmesine izin verin.
- Çakışma durumunda, iletinin
  zaten var olup olmadığına karar vermek için alındı/idempotans durumunu kullanın.
- Platform çağrısı başarılı olmuş olabilecekken ancak alındı kaydı
  kesinleşmeden önce oluşan her hata, bağdaştırıcı platform
  işleminin gerçekleşmediğini kanıtlamadığı sürece `unknown_after_send` olur.

## Açık sorular

- Telegram'ın sonunda grammY (`1.43.0`) yoklama
  çalıştırıcısını, yalnızca OpenClaw'ın kalıcı yeniden başlatma filigranını
  (`safeCompletedUpdateId`) değil, platform düzeyinde yeniden teslimatı da denetleyen tamamen kalıcı bir yoklama kaynağıyla
  değiştirip değiştirmemesi gerektiği.
- Canlı önizleme durumunun nihai gönderim
  amacıyla aynı kayıtta mı yoksa kardeş bir canlı durum deposunda mı tutulması gerektiği.
- Paylaşılan bot etkin odalarda Gateway arızası yankı engellemenin
  başlangıçta planlanan kaynak etiketleme mekanizmasına mı, kanal başına daha basit
  bir sözleşmeye mi ihtiyaç duyduğu, yoksa kapsam dışında mı olduğu.
- Hangi kanalların botlar arası yankı
  engelleme için yerel kaynak/meta veri desteğine sahip olduğu ve hangilerinin kalıcı bir giden ileti kayıt defterine ihtiyaç duyduğu.

## İlgili

- [İletiler](/tr/concepts/messages)
- [Akış ve parçalama](/tr/concepts/streaming)
- [İlerleme taslakları](/tr/concepts/progress-drafts)
- [Yeniden deneme politikası](/tr/concepts/retry)
- [Kanal giden ileti API'si](/tr/plugins/sdk-channel-outbound)
- [Kanal gelen ileti API'si](/tr/plugins/sdk-channel-inbound)
