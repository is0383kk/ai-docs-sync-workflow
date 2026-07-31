---
read_when:
    - Bir mesajlaşma kanalı Plugin gönderim yolu oluşturuyor veya yeniden düzenliyorsunuz
    - Kalıcı nihai yanıt iletimine, alındı onaylarına, canlı önizlemenin sonlandırılmasına veya alma onayı politikasına ihtiyacınız var
    - Kanal mesajı veya eski yanıt gönderim yardımcılarından geçiş yapıyorsunuz
summary: 'Kanal pluginleri için giden mesaj yaşam döngüsü API''si: bağdaştırıcılar, alındı bildirimleri, kalıcı gönderimler, canlı önizleme ve yanıt işlem hattı yardımcıları'
title: Kanal giden API'si
x-i18n:
    generated_at: "2026-07-26T23:29:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8edeca81d2e9261f33be1d538153caaea87caedb90dfccac33dd227c924501f1
    source_path: plugins/sdk-channel-outbound.md
    workflow: 16
---

Kanal pluginleri, giden ileti davranışını
`openclaw/plugin-sdk/channel-outbound` üzerinden kullanıma sunar. Alma/bağlam/sevk
orkestrasyonu için `openclaw/plugin-sdk/channel-inbound` kullanın.

Çekirdek; kuyruğa alma, dayanıklılık, dayanıklı **giriş izleyicisi ve boşaltma**
(`createChannelIngressMonitor`, `createChannelIngressDrain` ve
`openChannelIngressDrain`), genel yeniden deneme ilkesi, tur devralma yaşam döngüsü
(`turnAdoptionLifecycle` / `bindIngressLifecycleToReplyOptions`), kancalar,
alındılar ve paylaşılan `message` aracının sahibidir. Plugin ise yerel
gönderme/düzenleme/silme çağrıları, hedef normalleştirme, platform ileti dizileri, seçili
alıntılar, bildirim bayrakları, hesap durumu, giriş incelemesi ve yük
kodlama, şerit anahtarları, yeniden denenemez koşullar, isteğe bağlı geçersiz kılma
yetkilendirmesi ve platforma özgü yan etkilerin sahibidir.

## Dayanıklı giriş izleyicileri

Bir kanalın kabul edilen taşıma olaylarını sevkten önce kalıcı hâle getirmesi
gerektiğinde `createChannelIngressMonitor(...)` kullanın. Bu, bir kanal giriş kuyruğunu ve boşaltmayı
paylaşılan kabul, yoklama, budama, teslim ve kapatma yaşam döngüsüyle birleştirir.
Daha düşük düzeyli `createChannelIngressDrain(...)` yalnızca taşıma
önemli ölçüde farklı bir kabul veya pompalama sözleşmesine sahipse kullanılmalıdır.

Gerekli seçenekler şunlardır:

| Seçenek                          | Sözleşme                                                                                                                                                                                                                                                                                                         |
| -------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `queue`                          | Bir `ChannelIngressQueue` veya hesap kapsamlı kuyruğu açan tembel bir fabrika.                                                                                                                                                                                                                                  |
| `inspect(raw, context)`          | Kararlı `eventId` ve serileştirilmiş `laneKey` değerlerini ya da yok sayılan bir olay için `null` döndürür. Talep anındaki olgular, kalıcı kimlik ve şeritle eşleşmelidir.                                                                                                                                                                    |
| `payload`                        | Yük sürümünü ve gövde serileştirme/seri durumdan çıkarma işlemlerini sağlar. Standart `{ version, rawEvent }` dize zarfı için `storage: "raw-event"` kullanın veya kanala özgü mevcut bir şekil için özel kodlama/kod çözme geri çağrıları sağlayın. `createClaimError`, geçersiz sürümleri veya değişen kimliği sınıflandırır. |
| `deliver(raw, lifecycle, claim)` | Kodu çözülmüş tek bir olayı sevk eder ve devralma yaşam döngüsünün tamamını alır. `completed`, `deferred`, `failed-retryable` veya hiçbir şey döndürebilir.                                                                                                                                                                |
| `pollIntervalMs`                 | İzleyici çalışırken kurtarma/boşaltma yoklamalarını zamanlar.                                                                                                                                                                                                                                                     |
| `retention`                      | Budama sıklığını ve tamamlanan/başarısız TTL ile girdi sınırlarını sağlar.                                                                                                                                                                                                                                              |

İzleyici, ekleme geri çekilmesinin bir şeridin sırasını tersine çevirememesi için
kabulleri seri hâle getirir. Varsayılan sınırlı ekleme gecikmeleri `0`, `100` ve `300` ms'dir; bunların tükenmesi,
dayanıklı hâle getirilmemiş bir olayı sevk etmek yerine taşıma geri çağrısını
reddeder. Talep anında sürümlü yükün kodunu çözer, `inspect` işlemini yeniden çalıştırır ve
teslimden önce kimlik veya şerit uyuşmazlığını reddeder.

`deliver`; `onAdopted`, `onDeferred`, `onAdoptionFinalizing`,
`onAbandoned` ve `abortSignal` değerlerini alır. Açık bir devir olmadan dönmek,
sevk edilmeyen sonlandırıcı bir olayı devralınmış olarak işaretler. `admission` her zaman `exclusive` değerindedir.
Ertelenmiş bir devir talebi elde tutarken kapatma veya iptal, devralınmamış
işin yeniden denenebilir kalmasını sağlar. İzleyici teslimi talep uzlaşmasından
bağımsız olarak izler; çünkü devralma, kanalın teslim sözü
dönmeden önce bir satırı mezar taşıyla işaretleyebilir.

İsteğe bağlı ayarlar arasında özel ekleme gecikmeleri, gelişmiş boşaltma
sıralaması/eşzamanlılığı/yeniden deneme ilkesi için bir `drain` seçenek bloğu, harici bir `abortSignal`,
saat, pompa hata raporlaması, durduruldu-hatası fabrikası ve kabul ilkesi bulunur.
Döndürülen izleyici; `admit`, `start`, `pause`, `stop`, `waitForIdle`,
`isRunning` ve `isStopped` değerlerini kullanıma sunar. `stop` önce kabul edilmiş kabulleri uzlaştırır, ardından
boşaltmayı iptal edip bertaraf eder, pompayı ve etkin teslimleri bekler ve
tembel oluşturma yarışını kapatmak için yeniden bertaraf eder.

Taşımaya özgü gizlemeyi, ham zarf doğrulamasını, yeniden denenemez
sınıflandırmayı ve kalıcı yük şeklini Plugin içinde tutun. Webhook taşımaları
yalnızca `admit` çözümlendikten sonra onay vermelidir; yeniden oynatılamayan taşımalar ise
sessizce sevk etmek yerine dayanıklı ekleme tükenmesini bildirmelidir.

## Bağdaştırıcı

Çoğu Plugin tek bir `message` bağdaştırıcısı tanımlar:

```ts
import {
  defineChannelMessageAdapter,
  createMessageReceiptFromOutboundResults,
} from "openclaw/plugin-sdk/channel-outbound";

export const demoMessageAdapter = defineChannelMessageAdapter({
  id: "demo",
  durableFinal: {
    capabilities: {
      text: true,
      replyTo: true,
      thread: true,
      messageSendingHooks: true,
    },
  },
  send: {
    text: async ({ cfg, to, text, accountId, replyToId, threadId, signal }) => {
      const sent = await sendDemoMessage({
        cfg,
        to,
        text,
        accountId: accountId ?? undefined,
        replyToId: replyToId ?? undefined,
        threadId: threadId == null ? undefined : String(threadId),
        signal,
      });

      return {
        receipt: createMessageReceiptFromOutboundResults({
          results: [{ channel: "demo", messageId: sent.id, conversationId: to }],
          kind: "text",
          threadId: threadId == null ? undefined : String(threadId),
          replyToId: replyToId ?? undefined,
        }),
      };
    },
  },
});
```

Yalnızca yerel taşımanın gerçekten koruduğu yetenekleri bildirin. Bildirilen
her gönderme, alındı, canlı önizleme ve alma-onayı yeteneğini
bu alt yoldan dışa aktarılan sözleşme yardımcılarıyla kapsayın.

## Giden yankının engellenmesi

Bir platform, Plugin'in kendi giden iletisini gelen ileti olarak yeniden teslim edebiliyorsa kanal, hesap, konuşma ve kararlı bir platform iletisi veya kaynak kimliğiyle `recordOutboundMessageIdentity(...)` çağrısını yapın. Paylaşılan gelen tur yolu, oturum kaydından veya aracı sevkinden önce eşleşen kimlikleri 30 saniyelik sınırlı bir süre boyunca düşürür; teslim yarışlarını kapatmak için bir kaynak kimliği göndermeden önce ayrılabilir veya bir kanal rotası kaldırıldığında yenilenebilir. `isRecentOutboundMessageIdentity(...)`, kanal tanılamaları ve testleri için aynı sorguyu kullanıma sunar. Aynı kararlı kimlik için paralel bir kanala yerel TTL önbelleği tutmayın.

## Düz metin temizleme

Bir giden bağdaştırıcının desteklenen HTML biçimlendirme etiketlerini
hafif metin işaretlemesine dönüştürmesi gerektiğinde `sanitizeForPlainText(...)` kullanın.
Varsayılan ayar mevcut sohbet tarzı kalın ve üstü çizili işaretleyicileri korur.
Yalnızca kanal sonucu Markdown olarak yeniden ayrıştırıyorsa
`{ style: "markdown" }` geçirin:

```ts
import { sanitizeForPlainText } from "openclaw/plugin-sdk/channel-outbound";

const chatText = sanitizeForPlainText(text);
const markdownText = sanitizeForPlainText(text, { style: "markdown" });
```

Markdown stili `**bold**` ve `~~strikethrough~~` kullanır; italik ve satır içi
kod, her iki stilde de `_italic_` ve ters tırnak işaretleyicilerini korur. Temizlemeden
sonra işaretleyici metnini yeniden yazmak yerine stili kanal sınırında seçin.

## Teslim Kanıtı

Bir `MessageReceipt`, kanal bağdaştırıcısının döndürdüğü sonucu kaydeder. Somut
platform ileti tanımlayıcıları, platform gönderme yolunun iletiyi kabul ettiğini
gösterir; alıcının cihazında görüntülendiğini veya okunduğunu kanıtlamaz.
Platform ileti tanımlayıcısı olmayan alındılar yalnızca yerel alındı meta verileridir.
Okundu bilgisi veya cihaz teslim durumu bulunan kanallar bu olguları
kanala özgü ayrı bir yol üzerinden izlemelidir.

Bir kanal bağdaştırıcısı, bir hatayı yeniden denemenin alıcıya görünür bir gönderimi
çoğaltamayacağını ve sonlandırma yapabilen hiçbir çağrının başlamadığını kanıtlayabiliyorsa
`openclaw/plugin-sdk/error-runtime` içinden
`new PlatformMessageNotDispatchedError("...", { cause: error })` fırlatın. Böylece çekirdek, eski gönderme girişimi
kanıtlarını temizleyebilir ve kuyruğa alınmış amacı güvenle yeniden deneyebilir. Bu iddiayı
yalnızca nihai sevk sınırının sahibi olan bağdaştırıcı ileri sürebilir. İşaretleyiciyi asla
bir sonlandırma/gönderme çağrısı başladıktan veya belirsiz bir sonuç döndürdükten sonra kullanmayın; yanlış işaretleme
iletileri çoğaltabilir.

## Mevcut giden bağdaştırıcılar

Kanalda zaten uyumlu bir `outbound` bağdaştırıcısı varsa gönderme kodunu
çoğaltmak yerine ileti bağdaştırıcısını bundan türetin:

```ts
import { createChannelMessageAdapterFromOutbound } from "openclaw/plugin-sdk/channel-outbound";

export const messageAdapter = createChannelMessageAdapterFromOutbound({
  id: "demo",
  outbound,
  durableFinal: {
    capabilities: {
      text: true,
      media: true,
    },
  },
});
```

## Dayanıklı göndermeler

Çalışma zamanı gönderme yardımcıları da `channel-outbound` üzerinde bulunur:

- `sendDurableMessageBatch(...)`
- `withDurableMessageSendContext(...)`
- `deliverInboundReplyWithMessageSendContext(...)`
- `resolveChannelDraftStreamingChunking(...)` gibi taslak akışı/ilerleme yardımcıları

`sendDurableMessageBatch(...)` tek bir açık sonuç döndürür:

| Sonuç            | Anlamı                                                                                  |
| ---------------- | --------------------------------------------------------------------------------------- |
| `sent`           | platform gönderme yolu tarafından en az bir görünür platform iletisi kabul edildi        |
| `suppressed`     | hiçbir platform iletisi eksik olarak değerlendirilmemelidir                             |
| `partial_failed` | sonraki bir yük veya yan etki başarısız olmadan önce en az bir platform iletisi kabul edildi |
| `failed`         | hiçbir platform alındısı üretilmedi                                                     |

Bir toplu işlem gönderilmiş, engellenmiş ve başarısız yükleri karıştırdığında
`payloadOutcomes` kullanın. Boş bir eski doğrudan teslim
sonucundan kanca iptalini çıkarsamayın.

## Ertelenmiş teslim kabulü

Çözümlenmiş bir hesap, çekirdek tarafından yönetilen giden veya ertelenmiş teslimi
güvenle kabul edemediğinde `message.durableFinal.admitDeferredDelivery(...)` kullanın.
Çekirdek bu kancayı, kuyruk kalıcılığını atlayan yollar dâhil olmak üzere canlı
giden işten önce ve kurtarılan bir amacı yeniden oynatmadan önce eşzamanlı olarak
çağırır. Bağlam; `cfg`, `channel`, `to`, `accountId` ve `live` ya da
`recovery` değerinde bir `phase` içerir.

Devam etmek için `{ status: "allowed" }` döndürün. Teslimin
kalıcı hâle getirilmemesi, doğrudan gönderilmemesi veya yeniden oynatılmaması gerektiğinde
`{ status: "permanent_rejection", reason }` döndürün. Canlı bir ret, kuyruk
oluşturulmadan, ileti kancaları çalıştırılmadan veya platform işi yapılmadan önce başarısız olur.
Kurtarma reddi, kuyruğa alınmış kaydı başarısız olarak işaretler ve
uzlaştırma ile yeniden oynatmayı atlar. Kancanın belirtilmemesi izin verildiği anlamına gelir.

Kanca, bir gönderim yolu değil, eşzamanlı bir kabul kararıdır. Yalnızca
önceden yüklenmiş yapılandırmayı veya çalışma zamanı durumunu okuyun; ağ, dosya sistemi ya da
başka eşzamansız G/Ç işlemleri gerçekleştirmeyin. Sözleşme testleri, hem aşamaları hem de
sonuç değişkenlerini `openclaw/plugin-sdk/channel-outbound` içindeki
`ChannelMessageDurableFinalAdapter` üzerinden sınamalıdır.

## Uyumluluk dağıtımı

Gelen yanıt dağıtımını `channel-inbound` içindeki
`dispatchChannelInboundReply(...)` üzerinden oluşturun. Platform teslimatını teslimat bağdaştırıcısında tutun;
mesaj bağdaştırıcıları, kalıcı gönderimler, alındı bildirimleri, canlı
önizleme ve yanıt işlem hattı seçenekleri için `channel-outbound` kullanın.
