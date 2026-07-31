---
read_when:
    - Bir mesajlaşma kanalı Plugin'inin alma yolunu oluşturuyor veya yeniden düzenliyorsunuz
    - Paylaşılan gelen bağlam oluşturma, oturum kaydı veya hazırlanmış yanıt gönderimi gerekir
    - Eski kanal turu yardımcılarını gelen/mesaj API'lerine taşıyorsunuz
summary: 'Kanal pluginleri için gelen olay yardımcıları: bağlam oluşturma, paylaşılan çalıştırıcı düzenlemesi, oturum kaydı ve hazırlanmış yanıt gönderimi'
title: Kanal gelen ileti API'si
x-i18n:
    generated_at: "2026-07-26T22:56:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 854408ca42cfe1e1b48e4fd223b176f438e1db28deb9a5aa33eea8238127d9df
    source_path: plugins/sdk-channel-inbound.md
    workflow: 16
---

Kanal alma yolları tek bir akışı izler:

```text
platform olayı -> gelen olgu/bağlam -> ajan yanıtı -> ileti teslimi
```

Gelen olay normalleştirme, biçimlendirme, kökler ve orkestrasyon için
`openclaw/plugin-sdk/channel-inbound` kullanın. Yerel gönderim, alındı, kalıcı
teslim ve canlı önizleme davranışı için
`openclaw/plugin-sdk/channel-outbound` kullanın.

## Temel yardımcılar

```ts
import {
  buildChannelInboundEventContext,
  runChannelInboundEvent,
  dispatchChannelInboundReply,
} from "openclaw/plugin-sdk/channel-inbound";
```

- `buildChannelInboundEventContext(...)`: normalleştirilmiş kanal olgularını
  istem/oturum bağlamına yansıtır. Kanalın sahip olduğu gönderen/sohbet meta verilerini,
  Plugin kancalarının `ctx.channelContext` olarak gördüğü `channelContext`
  üzerinden geçirin. Kanala özgü alanlar için bu alt yoldan
  `PluginHookChannelSenderContext` veya `PluginHookChannelChatContext` değerini genişletin.
- `runChannelInboundEvent(...)`: tek bir gelen platform olayı için alma,
  sınıflandırma, ön kontrol, çözümleme, kaydetme, gönderme ve sonlandırma işlemlerini yürütür.
- `dispatchChannelInboundReply(...)`: önceden oluşturulmuş bir gelen yanıtı
  teslim adaptörüyle kaydeder ve gönderir.

Yalnızca medya içeren gelen olaylarda ileti gövdesini ve komut metnini boş tutun ve
her yerel ek için bir `ChannelInboundMediaInput` olgusu geçirin. Bir ortam
geçmişi satırı veya yalnızca metin içeren başka bir taşıyıcının bu olguları açıklaması gerektiğinde
`formatMediaPlaceholderText(media)` kullanın. Her olguyu `kind`, MIME
türü, ardından yol veya URL uzantısına göre sınıflandırır; indirilmemiş yerel eklerin her biri de
yalnızca tür içeren bir olgu sağlamalıdır. Birincil gelen gövdeyi sentezlemek için
biçimlendiriciyi kullanmayın.

Plugin'e ait ek kayıtlarını `toInboundMediaFacts(...)` ile normalleştirin, ardından
elde edilen sıralı diziyi bağlamın `media` alanından geçirin:

```ts
const media = toInboundMediaFacts([
  { path: saved.path, url: nativeUrl, contentType: saved.contentType, messageId },
]);

const ctx = finalizeInboundContext({ Body: caption, media });
```

Dizi konumu, ek kimliğidir. Olgu başına `transcribed`, `messageId` ve
`workspaceDir`, eski paralel dizin/çalışma alanı alanlarının yerini alır.
`MediaPath`, `MediaPaths`, `MediaUrl`, `MediaUrls`, `MediaType`, `MediaTypes`,
`MediaTranscribedIndexes`, `MediaWorkspaceDir` ve `MediaStaged` bağlam alanları ile
`buildChannelInboundMediaPayload(...)`, yalnızca kullanımdan kaldırılmış
uyumluluk öğeleri olarak kullanılabilir. Yeni Plugin'ler bunları oluşturmamalı veya okumamalıdır.

Eklenen Plugin çalışma zamanı nesnesini zaten alan paketlenmiş/yerel kanallar,
bu alt yolu doğrudan içe aktarmak yerine aynı yardımcıları
`runtime.channel.inbound.*` altında çağırabilir:

```ts
await runtime.channel.inbound.run({
  channel: "demo",
  accountId,
  raw: platformEvent,
  adapter: {
    ingest: normalizePlatformEvent,
    resolveTurn: resolveInboundReply,
  },
});
```

Platform teslimini teslim adaptöründe tutan uyumluluk
göndericileri için `dispatchChannelInboundReply(...)` girdilerini oluşturun. Yeni gönderim
yolları bunun yerine `channel-outbound` içindeki ileti adaptörlerini ve kalıcı ileti yardımcılarını
kullanmalıdır.

## Teslim sonuçlandırma sözleşmesi

`ChannelInboundTurnPlan.delivery`, her mantıksal yanıt yükünün yerel gönderimine
sahiptir. Çekirdek, giden kanca sıralamasına ve adaptör bunu etkinleştirdiğinde
uç `message_sent` gözlemine sahiptir. Tek bir yükün yinelenen uç olaylar
üretmemesi için bu sorumlulukları ayrı tutun.

Teslim sonucu alanları şu anlamlara gelir:

| Alan                    | Sözleşme                                                                                                                                                                                                                     |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `content`                | Yerel biçimlendirme veya sonlandırma sonrasında sağlayıcı tarafından kabul edilen, mantıksal yüke ait görünür metin. Uç gözlemde hazırlanmış yük metnini kullanmak için bunu atlayın. Yalnızca medya gönderimleri bunu atlayabilir.                             |
| `messageIds` / `receipt` | Görünür gönderimin gerçek sağlayıcı kimlikleri. Bir `MessageReceipt` tercih edin; çekirdek, `message_sent` için bunun birincil sağlayıcı kimliğini kullanır.                                                                                            |
| `visibleReplySent`       | Yalnızca sağlayıcı görünür bir önizleme veya nihai ileti üretmediğinde `false` olarak ayarlayın. Çekirdek, bu sonuç için başarılı bir `message_sent` yaymaz.                                                                          |
| `finalization`           | Yerinde akış kartını kapatma veya düzenleme gibi, aynı mantıksal yükün gecikmeli yerel sonuçlandırılmasına yönelik bir promise. Çözümlenen alanları, uç gözlemden ve `onDelivered` işleminden önce anlık sonucu geçersiz kılar. |

Çekirdeğin bu adaptörün kalıcı olmayan gönderimleri için standart Plugin ve dahili
`message_sent` olaylarını yayması gerektiğinde teslim adaptörünün
`observeMessageSent` seçeneğini `true` olarak ayarlayın. Bu seçeneği
`deliver` içinden döndürmeyin ve bu olayları Plugin içinde de yaymayın.
Kalıcı gönderimler zaten paylaşılan giden öğe sahibi üzerinden yayılır ve yinelenmez.

Mantıksal yük başına bir sonuç döndürün. `finalization` ikinci bir gönderim değildir ve
`reply_payload_sending` veya `message_sending` işlemlerini yeniden çalıştırmamalıdır.
`deliver` döner dönmez çekirdek, işlenmemiş duruma gelmemesi için
sonlandırma promise'inin reddedilmesini gözlemler; çekirdek, yanıt gönderimi
sonuçlandıktan sonra özgün promise'i yine de bekler. Ardından sonlandırılmış içerik ve
sağlayıcı kimliğiyle yük başına en fazla bir uç gözlem yayar. Varsa
`onDelivered`, bu gözlemden sonra sonuçlandırılmış sonucu alır.

Yerel teslim başarısız olduğunda `deliver` veya `finalization` öğesini reddedin.
Hiçbir sağlayıcı gönderimi denenmediyse `openclaw/plugin-sdk/error-runtime` içinden
`PlatformMessageNotDispatchedError` fırlatın; çekirdek hatalı bir `message_sent`
olayını engeller. Yerel bir gönderim görünür olduktan sonra daha sonraki bir işlem
başarısız olduysa hatada görünür alt kümeyi koruyun:

```ts
import { createChannelPartialDeliveryError } from "openclaw/plugin-sdk/channel-inbound";

throw createChannelPartialDeliveryError(cause, {
  visibleReplySent: true,
  content: finalizedVisibleText,
  receipt,
});
```

Çekirdek, sağlayıcının görebildiği bu içerik ve kimlikle başarısız bir uç gözlem
yayar, ardından çağıranların kısmi başarıyı sorunsuz bir gönderim sanmaması için
teslimi başarısız durumda tutar. Herhangi bir önizleme, taslak, ek veya nihai ileti
görünür olduktan sonra `visibleReplySent: false` bildirmeyin.

`reply_payload_sending` veya `message_sending` kaydedildiğinde bu kancalar,
sağlayıcı tarafından görülebilen herhangi bir şey oluşturulmadan önce sonuçlanmalıdır; çünkü her iki kanca da
mantıksal yükü yeniden yazabilir veya iptal edebilir. Erken oluşturulan yerel önizleme,
yeniden yazma öncesindeki içeriği sızdırır veya iptal edilmiş bir taslağı geride bırakır.
Kabul edilen yük `deliver` aşamasına ulaşana kadar önizleme içeriğini
arabelleğe alın; önizlemeleri daha erken başlatan uyumluluk göndericileri, bu kancalardan
biri kayıtlıyken erken önizlemeyi engellemelidir. Yeni önizleme yolları için
[Kanal giden API'si](/tr/plugins/sdk-channel-outbound) bölümündeki sonlandırılabilir canlı önizleme yardımcılarını kullanın.

## Geçiş

`runtime.channel.turn.*` çalışma zamanı takma adları kaldırıldı. Şunları kullanın:

- `runtime.channel.inbound.run(...)`: ham gelen olaylar için.
- `runtime.channel.inbound.dispatchReply(...)`: oluşturulmuş yanıt bağlamları için.
- `runtime.channel.inbound.buildContext(...)`: gelen bağlam yükleri için.
- `runtime.channel.inbound.runPreparedReply(...)`: kullanımdan kaldırılmıştır; yalnızca
  kendi gönderim kapanışlarını zaten oluşturan, kanala ait hazırlanmış
  gönderim yolları içindir.

Yeni Plugin kodu, `turn` adlı kanal API'leri eklememelidir. Model veya
ajan turu terminolojisini ajan/sağlayıcı kodu içinde tutun; kanal Plugin'leri gelen,
ileti, teslim ve yanıt terimlerini kullanır.
