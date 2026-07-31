---
read_when:
    - Mesajlaşma kanalı Plugin'i oluşturma veya taşıma
    - DM veya grup izin listelerini, yönlendirme geçitlerini, komut kimlik doğrulamasını, olay kimlik doğrulamasını ya da bahsetmeyle etkinleştirmeyi değiştirme
    - Kanal girişindeki redaksiyon veya SDK uyumluluk sınırlarını inceleme
sidebarTitle: Channel Ingress
summary: Gelen ileti yetkilendirmesi için deneysel kanal giriş API'si
title: Kanal giriş API'si
x-i18n:
    generated_at: "2026-07-26T22:56:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 60feecb7bcf203cf37d2543a7855e89b5bfb2eb9d8263d804219e140facb8fc6
    source_path: plugins/sdk-channel-ingress.md
    workflow: 16
---

Kanal girişi, gelen kanal olayları için deneysel erişim denetimi sınırıdır.
Plugin’ler platform olgularının ve yan etkilerin sahibidir; çekirdek ise
genel politikanın sahibidir: DM/grup izin listeleri, eşleştirme deposundaki DM girdileri, rota kapıları,
komut kapıları, olay yetkilendirmesi, bahsetmeyle etkinleştirme, hassas bilgileri çıkarılmış tanılamalar ve
kabul.

Alma yolları için `openclaw/plugin-sdk/channel-ingress-runtime` kullanın.

## Çalışma zamanı çözümleyicisi

```ts
import {
  defineStableChannelIngressIdentity,
  resolveChannelMessageIngress,
} from "openclaw/plugin-sdk/channel-ingress-runtime";

const identity = defineStableChannelIngressIdentity({
  key: "platform-user-id",
  normalize: normalizePlatformUserId,
  sensitivity: "pii",
});

const result = await resolveChannelMessageIngress({
  channelId: "my-channel",
  accountId,
  identity,
  subject: { stableId: platformUserId },
  conversation: { kind: isGroup ? "group" : "direct", id: conversationId },
  event: { kind: "message", authMode: "inbound", mayPair: !isGroup },
  policy: {
    dmPolicy: config.dmPolicy,
    groupPolicy: config.groupPolicy,
    groupAllowFromFallbackToAllowFrom: true,
  },
  allowFrom: config.allowFrom,
  groupAllowFrom: config.groupAllowFrom,
  accessGroups: cfg.accessGroups,
  route,
  readStoreAllowFrom,
  command: hasControlCommand ? { allowTextCommands: true, hasControlCommand } : undefined,
});
```

Etkin izin listelerini, komut sahiplerini veya komut gruplarını önceden hesaplamayın.
Çözümleyici bunları ham izin listelerinden, depo geri çağırımlarından, rota
tanımlayıcılarından, erişim gruplarından, politikadan ve konuşma türünden türetir.

## Sonuç

Birlikte gelen Plugin’ler modern izdüşümleri doğrudan kullanmalıdır:

| Alan               | Anlam                                                              |
| ------------------ | ------------------------------------------------------------------ |
| `ingress`          | sıralı kapı kararı ve kabul                                        |
| `senderAccess`     | yalnızca gönderen/konuşma yetkilendirmesi                           |
| `routeAccess`      | rota ve rota göndereni izdüşümü                                    |
| `commandAccess`    | komut yetkilendirmesi; komut kapısı çalıştırılmadığında `requested: false` |
| `activationAccess` | bahsetme/etkinleştirme sonucu                                      |

Olay yetkilendirmesi, sıralı `ingress.graph` ve belirleyici
`ingress.reasonCode` üzerinde kullanılabilir durumda kalır; ayrı bir olay izdüşümü üretilmez.

Kullanımdan kaldırılmış üçüncü taraf SDK yardımcıları eski şekilleri dahili olarak yeniden oluşturabilir. Birlikte gelen yeni
alma yolları, modern sonuçları yeniden yerel
DTO’lara dönüştürmemelidir.

## Erişim grupları

`accessGroup:<name>` girdilerinin hassas bilgileri çıkarılmış olarak kalır. Çekirdek, statik
`message.senders` gruplarını kendisi çözümler ve `resolveAccessGroupMembership` öğesini yalnızca
platform araması gerektiren dinamik gruplar için çağırır. Eksik, desteklenmeyen ve
başarısız gruplar kapalı durumda başarısız olur.

## Olay kipleri

| `authMode`       | Anlam                                            |
| ---------------- | ------------------------------------------------ |
| `inbound`        | normal gelen gönderen kapıları                   |
| `command`        | geri çağırımlar veya kapsamlı düğmeler için komut kapıları |
| `origin-subject` | aktör, özgün ileti öznesiyle eşleşmelidir         |
| `route-only`     | yalnızca rota kapsamlı güvenilir olaylar için rota kapıları |
| `none`           | Plugin’e ait dahili olaylar paylaşılan yetkilendirmeyi atlar |

Tepkiler, düğmeler, geri çağırımlar ve yerel komutlar için `mayPair: false` kullanın.

## Rotalar ve etkinleştirme

Oda, konu, sunucu, iş parçacığı veya iç içe rota politikası için rota tanımlayıcıları kullanın:

```ts
route: {
  id: "room",
  allowed: roomAllowed,
  enabled: roomEnabled,
  senderPolicy: "replace",
  senderAllowFrom: roomAllowFrom,
  blockReason: "room_sender_not_allowlisted",
}
```

Bir Plugin’in birkaç isteğe bağlı rota
tanımlayıcısı olduğunda `channelIngressRoutes(...)` kullanın; bu, rota olgularını genel
ve her tanımlayıcının `precedence` değerine göre sıralı tutarken devre dışı dalları filtreler.

Bahsetme kapısı bir etkinleştirme kapısıdır. Başarısız bir bahsetme eşleşmesi
`admission: "skip"` döndürür; böylece tur çekirdeği yalnızca gözlem amaçlı bir turu işlemez.
Çoğu kanal etkinleştirmeyi gönderen ve komut kapılarından sonra bırakmalıdır. Gönderen izin listesi
gürültüsünden önce bahsedilmeyen trafiği susturması gereken herkese açık
sohbet yüzeyleri, metin komutu atlaması devre dışı olduğunda `activation.order: "before-sender"` seçeneğini
etkinleştirebilir. Bot iş parçacıklarındaki yanıtlar gibi örtük etkinleştirmeye sahip kanallar,
`channels.defaults.implicitMentions` ile kanal ve hesap
geçersiz kılmalarını `resolveChannelImplicitMentions(...)` kullanarak çözümler, ardından sonucu
`activation.implicitMentions` olarak iletir. İzdüşümü yapılan
`activationAccess.shouldBypassMention`, komut veya örtük
etkinleştirmenin açık bir bahsetmeyi ne zaman atladığını bildirir.

## Hassas bilgileri çıkarma

Ham gönderen değerleri ve ham izin listesi girdileri yalnızca çözümleyici girdisidir. Bunlar
çözümlenmiş durumda, kararlarda, tanılamalarda, anlık görüntülerde veya
uyumluluk olgularında görünmemelidir. Opak özne kimlikleri, girdi kimlikleri, rota kimlikleri ve
tanılama kimlikleri kullanın.

## Doğrulama

```bash
pnpm test src/channels/message-access/message-access.test.ts src/plugin-sdk/channel-ingress-runtime.test.ts
pnpm plugin-sdk:api:check
```
