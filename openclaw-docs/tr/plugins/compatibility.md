---
read_when:
    - Bir OpenClaw Plugin'inin bakımını yapıyorsunuz
    - Bir plugin uyumluluk uyarısı görüyorsunuz
    - Bir Plugin SDK'sı veya manifest geçişi planlıyorsunuz
summary: Plugin uyumluluk sözleşmeleri, kullanımdan kaldırma meta verileri ve geçiş beklentileri
title: Plugin uyumluluğu
x-i18n:
    generated_at: "2026-07-27T00:05:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80cf1dfce9e0538e78138ff80a6807ee36267a07d3eee6f19bd8e56e5c0c9cd3
    source_path: plugins/compatibility.md
    workflow: 16
---

OpenClaw, eski Plugin sözleşmelerini kaldırmadan önce adlandırılmış uyumluluk
adaptörleri aracılığıyla bağlı tutar. Bu, SDK, bildirim, kurulum, yapılandırma
ve ajan çalışma zamanı sözleşmeleri gelişirken mevcut paketlenmiş ve harici
Pluginleri korur.

## Uyumluluk kayıt defteri

Plugin uyumluluk sözleşmeleri, `src/plugins/compat/registry.ts` konumundaki çekirdek kayıt
defterinde izlenir. Her kayıt şunları içerir:

- kararlı bir uyumluluk kodu
- durum: `active`, `deprecated`, `removal-pending` veya `removed`
- sahip: `sdk`, `config`, `setup`, `channel`, `provider`, `plugin-execution`,
  `agent-runtime` veya `core`
- uygun olduğunda kullanıma sunulma ve kullanımdan kaldırılma tarihleri
- sorumlu bakımcı onayladıktan sonra kesin kaldırma tarihi; `removeAfter`
  değerinin belirtilmemesi, kullanımdan kaldırılmış bir yüzeyi kaldırılmaya
  uygun olmaktan çıkarır
- yerine kullanılacak öğeye ilişkin rehberlik
- eski ve yeni davranışı kapsayan belgeler, tanılamalar ve testler

Kayıt defteri, bakımcı planlamasının ve gelecekteki Plugin denetleyicisi
kontrollerinin kaynağıdır. Pluginlere yönelik bir davranış değişirse adaptörü
ekleyen değişiklikle birlikte uyumluluk kaydını da ekleyin veya güncelleyin.

Doctor onarım ve geçiş uyumluluğu `src/commands/doctor/shared/deprecation-compat.ts` konumunda ayrı olarak
izlenir. Bu kayıtlar, çalışma zamanı uyumluluk yolu kaldırıldıktan sonra da
kullanılabilir kalması gerekebilecek eski yapılandırma biçimlerini, kurulum
defteri düzenlerini ve onarım uyumluluk katmanlarını kapsar.

Sürüm taramaları her iki kayıt defterini de kontrol etmelidir. Eşleşen çalışma
zamanı veya yapılandırma uyumluluk kaydının süresi doldu diye bir doctor
geçişini silmeyin; önce, onarıma hâlâ ihtiyaç duyan desteklenen bir yükseltme
yolu olmadığını doğrulayın. Sağlayıcılar ve kanallar çekirdekten dışarı
taşındıkça Plugin sahipliği ve yapılandırma kapsamı değişebileceğinden, sürüm
planlaması sırasında yerine kullanılacak öğelere ilişkin her açıklamayı da
yeniden doğrulayın.

## Kullanımdan kaldırma politikası

OpenClaw, belgelenmiş bir Plugin sözleşmesini yerine kullanılacak sözleşmenin
sunulduğu aynı sürümde kaldırmamalıdır. Geçiş sırası:

1. Yeni sözleşmeyi ekleyin.
2. Eski davranışı adlandırılmış bir uyumluluk adaptörü aracılığıyla bağlı tutun.
3. Plugin yazarları işlem yapabildiğinde tanılama mesajları veya uyarılar yayınlayın.
4. Yerine kullanılacak öğeyi ve zaman çizelgesini belgeleyin.
5. Hem eski hem de yeni yolları test edin.
6. Duyurulan geçiş süresi boyunca bekleyin.
7. Yalnızca açık bir uyumsuz değişiklik sürümü onayıyla kaldırın.

Kullanımdan kaldırılmış kayıtlar bir uyarı başlangıç tarihi, yerine kullanılacak
öğe, belge bağlantısı ve uyarının başlamasından en fazla üç ay sonrasına ait
nihai kaldırma tarihi içermelidir. Bakımcılar bunun kalıcı uyumluluk olduğuna
açıkça karar verip `active` olarak işaretlemediği sürece, ucu açık
bir kaldırma süresine sahip kullanımdan kaldırılmış bir uyumluluk yolu
eklemeyin.

## Mevcut uyumluluk alanları

Temmuz 2026 taraması; süresi dolmuş kök SDK, bildirim, sağlayıcı, çalışma
zamanı, kayıt defteri bayrağı ve Pluginlere ait web yapılandırması takma
adlarını kaldırdı. Desteklenen yükseltme yollarının eski yapılandırmayı
onarabilmesi için doctor geçişleri ayrı olarak izlenmeye devam eder.

Tarihli kalan uyumluluk alanları şunlardır:

- geçiş rehberinde listelenen Ağustos ve Eylül SDK alt yol süreleri
- `api.on("deactivate", ...)` ve `api.on("subagent_spawning", ...)` kanca takma adları
- belleğe özgü gömme kaydı ve beta.5 oturum deposu köprüsü
- aşağıda açıklanan WhatsApp gelen geri çağrı takma adları
- açık kanal hedefi ayrıştırması ve `openclaw/plugin-sdk/messaging-targets`
- gömülü Pi ajanı takma adları
- kaldırılması, haricen belgelenmiş yeni bir geçiş kararı bekleyen,
  yayımlanmış ajan donanımı SDK takma adları

Etkin ve tarihsiz kayıt defteri kayıtları; etkinleştirme ipuçları, Plugin
yakalama, paketlenmiş Plugin etkinleştirme ve oluşturulan kanal yapılandırması
geri dönüşü dâhil olmak üzere kaldırma borcu yerine desteklenen davranışı
kapsar.

### WhatsApp gelen geri çağrı düz takma adları

WhatsApp çalışma zamanı geri çağrıları, standart iç içe `event`,
`payload`, `quote`, `group` ve
`platform` bağlamları ile yayımlanmış geri çağrı alanları için
kullanımdan kaldırılmış düz takma adları içeren `WebInboundMessage` değerini
iletir. Yeni geri çağrı kodu iç içe bağlamları okumalıdır. Temiz, iç içe geri
çağrı mesajları oluşturan kod `WebInboundCallbackMessage` kullanabilir; hâlâ eski düz
test veya Plugin mesajları ekleyen uyumluluk dinleyicileri
`LegacyFlatWebInboundMessage` ya da `WebInboundMessageInput` kullanmalıdır.

Düz takma adlar **2026-08-30** tarihine kadar kullanılabilir kalır; bu süre,
standart çalışma zamanı sözleşmesi olan iç içe biçime değil, yalnızca düz
takma ad erişimine uygulanır. Her düz takma adın TypeScript
`@deprecated` açıklaması, onun tam iç içe karşılığını belirtir. Yaygın
örnekler:

- `id`, `timestamp` ve `isBatched`, `event` altına taşınır.
- `body`, `mediaPath`, `mediaType`, `mediaFileName`, `mediaUrl`, `location`
  ve `untrustedStructuredContext`, `payload` altına taşınır.
- `to`, `chatId`, gönderen/kendi alanları, `sendComposing`, `reply(...)` ve
  `sendMedia(...)`, `platform` altına taşınır.
- `replyTo*` alanları `quote` altına; grup konusu/katılımcı/bahsetme
  alanları `group` altına taşınır.

`payload.untrustedStructuredContext`, gelen sağlayıcı yüklerinden çıkarılır. Pluginler,
`payload` değerini yetkili kabul etmeden önce
`label`, `source` ve `type` değerlerini
incelemelidir.

### WhatsApp gelen kabul alanları

Kabul edilen WhatsApp geri çağrı mesajları, mesajı kabul eden erişim denetimi
kararı için herkese açık ve güvenli bir zarf olan `admission` değerini
taşır. Yeni geri çağrı kodu, kabul bilgilerini eski üst düzey kabul alanları
yerine `msg.admission` üzerinden okumalıdır.

Üst düzey alanlar **2026-08-30** tarihine kadar kullanılabilir kalır. Her alanın
TypeScript `@deprecated` açıklaması, yerine kullanılacak öğeyi belirtir:

- `from` ve `conversationId`, `admission.conversation.id` konumuna taşınır.
- `accountId`, `admission.accountId` konumuna taşınır.
- `accessControlPassed`, `admission.ingress.decision === "allow"` değerinin türetilmiş bir uyumluluk görünümüdür;
  zaten `admission` taşıyan mesajlarda eski Boole değerini yazmak,
  giriş grafiğini yeniden yazmaz.
- `chatType`, `admission.conversation.kind` konumuna taşınır.

## Plugin denetleyicisi paketi

Plugin denetleyicisi, sürümlenmiş uyumluluk ve bildirim sözleşmeleriyle
desteklenen ayrı bir paket/depo olarak çekirdek OpenClaw deposunun dışında
bulunmalıdır. İlk gün CLI komutu şöyle olmalıdır:

```sh
openclaw-plugin-inspector ./my-plugin
```

Bildirim/şema doğrulamasını, denetlenen sözleşme uyumluluk sürümünü,
kurulum/kaynak meta verisi kontrollerini, soğuk yol içe aktarma kontrollerini
ve kullanımdan kaldırma/uyumluluk uyarılarını yayımlamalıdır. CI ek
açıklamalarında kararlı ve makine tarafından okunabilir çıktı için
`--json` kullanın. OpenClaw çekirdeği, denetleyicinin
tüketebileceği sözleşmeleri ve fikstürleri sunmalı ancak denetleyici ikili
dosyasını ana `openclaw` paketinden yayımlamamalıdır.

### Bakımcı kabul yolu

Harici denetleyiciyi OpenClaw Plugin paketlerine karşı doğrularken
kurulabilir paket kabul yolu için Crabbox destekli Blacksmith Testbox
kullanın. Paket oluşturulduktan sonra bunu temiz bir OpenClaw çalışma
kopyasından çalıştırın:

```sh
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "pnpm install && pnpm build && npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/telegram --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- ./extensions/discord --json"
pnpm crabbox:run -- --provider blacksmith-testbox --timing-json --shell -- "npm exec --yes @openclaw/plugin-inspector@0.1.0 -- <clawhub-plugin-dir> --json"
```

Harici bir npm paketi kurduğu ve depo dışında klonlanan Plugin paketlerini
inceleyebileceği için bu yolu bakımcılar açısından isteğe bağlı tutun. Yerel
depo korumaları SDK dışa aktarma eşlemesini, uyumluluk kayıt defteri meta
verilerini, kullanımdan kaldırılmış SDK içe aktarımlarının azaltılmasını ve
paketlenmiş eklentilerin içe aktarma sınırlarını kapsar; Testbox denetleyicisi
kanıtı ise paketi harici Plugin yazarlarının kullandığı biçimde kapsar.

## Sürüm notları

Sürüm notları, bir uyumluluk yolu `removal-pending` veya
`removed` durumuna geçmeden önce, hedef tarihleri ve geçiş
belgelerine bağlantılarıyla birlikte yaklaşan Plugin kullanımdan kaldırma
işlemlerini içermelidir.
