---
read_when:
    - Gelen mesajların nasıl yanıtlara dönüştüğünü açıklama
    - Oturumları, kuyruğa alma modlarını veya akış davranışını açıklığa kavuşturma
    - Akıl yürütme görünürlüğünü ve kullanım üzerindeki etkilerini belgeleme
summary: Mesaj akışı, oturumlar, kuyruğa alma ve akıl yürütme görünürlüğü
title: Mesajlar
x-i18n:
    generated_at: "2026-07-26T22:44:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e42bed834e9a57fb8a248c8654b75ea9977928582f68a83859cf6c16ed0b6bf5
    source_path: concepts/messages.md
    workflow: 16
---

Gelen mesajlar yönlendirme, yinelenenleri ayıklama/bekletme, bir aracı çalıştırma ve giden teslimat aşamalarından geçer:

```text
Gelen mesaj
  -> yönlendirme/bağlamalar -> oturum anahtarı
  -> yinelenenleri ayıklama + bekletme
  -> kuyruk (bir çalıştırma zaten etkinse)
  -> aracı çalıştırma (akış + araçlar)
  -> giden yanıtlar (kanal sınırları + parçalara ayırma)
```

Temel yapılandırma yüzeyleri:

- `messages.*`: ön ekler, kuyruğa alma, gelen mesajları bekletme ve grup davranışı için.
- `agents.defaults.*`: blok akışı, parçalara ayırma ve sessiz yanıt varsayılanları için.
- Kanal başına sınırlar ve akış anahtarları için kanal geçersiz kılmaları (`channels.telegram.*`, `channels.whatsapp.*` vb.).

Tam şema için [Yapılandırma](/tr/gateway/configuration) bölümüne bakın.

## Gelen mesajlarda yinelenenleri ayıklama

Kanallar, yeniden bağlantı kurulduktan sonra aynı mesajı yeniden teslim edebilir. OpenClaw; aracı kapsamı, kanal rotası (kanal + eş + hesap + ileti dizisi) ve mesaj kimliğine göre anahtarlanan bir bellek içi önbellek tutar; böylece yeniden teslim edilen bir mesaj ikinci bir aracı çalıştırmayı tetiklemez. Önbellek girdisi, hangisi önce gerçekleşirse 20 dakika sonra veya 5000 girdi izlenmeye başladığında sona erer.

## Gelen mesajları bekletme

Aynı göndericiden hızla art arda gelen metin mesajları, `messages.inbound` aracılığıyla tek bir aracı sırasına toplu olarak alınabilir. Bekletme, kanal + konuşma başına kapsamlandırılır ve yanıt ileti dizisi/kimlikleri için en son mesajı kullanır.

```json5
{
  messages: {
    inbound: {
      debounceMs: 2000,
      byChannel: {
        discord: 1500,
        slack: 1500,
        whatsapp: 5000,
      },
    },
  },
}
```

- Bekletme yalnızca metin mesajlarına uygulanır; medya/ekler hemen gönderilir.
- Denetim komutları (durdur/iptal et/durum vb.), hemen gönderilmeleri için bekletmeyi atlar.
- Varsayılan olarak devre dışıdır: `messages.inbound.debounceMs` yerleşik bir varsayılana sahip değildir; bu nedenle bekletme yalnızca siz ayarladıktan sonra (genel olarak veya kanal başına) etkinleşir.
- iMessage aynı genel bekletme politikasını izler. `imsg` 0.13.1 ve sonraki sürümler, Apple URL önizlemesi nedeniyle bölünmüş gönderimleri OpenClaw almadan önce birleştirir; bu nedenle iMessage'a özgü bir bekletme ayarı gerekmez.

## Oturumlar ve cihazlar

Oturumların sahibi istemciler değil, Gateway'dir.

- Doğrudan sohbetler, aracının ana oturum anahtarında birleştirilir.
- Gruplar/kanallar kendi oturum anahtarlarını alır.
- Oturum deposu ve dökümler Gateway ana makinesinde bulunur.

Birden fazla cihaz/kanal aynı oturumla eşlenebilir, ancak geçmiş her istemciyle tam olarak yeniden eşitlenmez. Bağlamın ayrışmasını önlemek için uzun konuşmalarda tek bir birincil cihaz kullanın. Control UI ve TUI her zaman Gateway destekli oturum dökümünü gösterir; dolayısıyla doğruluk kaynağı bunlardır.

Ayrıntılar: [Oturum yönetimi](/tr/concepts/session).

## İstem gövdeleri ve geçmiş bağlamı

Kanal Plugin'leri, gelen bağlamdaki çeşitli metin alanlarını en çok tercih edilenden en az tercih edilene doğru doldurur:

| Alan              | Amaç                                                                                                        |
| ----------------- | ----------------------------------------------------------------------------------------------------------- |
| `BodyForAgent`    | Geçerli sıra için modele yönelik metin. Ayarlanmamışsa `CommandBody` / `RawBody` / `Body` değerlerine geri döner.        |
| `BodyForCommands` | Yönerge/komut ayrıştırmada kullanılan temiz metin. Ayarlanmamışsa `CommandBody` / `RawBody` / `Body` değerlerine geri döner. |
| `CommandBody`     | Eski ara gövde; `BodyForCommands` tercih edilmelidir.                                                        |
| `RawBody`         | `CommandBody` için kullanımdan kaldırılmış diğer ad.                                                        |
| `Body`            | Eski istem gövdesi; kanal zarflarını ve geçmiş sarmalayıcılarını içerebilir.                                  |

Bir kanal geçmiş sağladığında bunu şunlarla sarmalar:

- `[Chat messages since your last reply - for context]`
- `[Current message - respond to this]`

Doğrudan olmayan sohbetlerde (gruplar/kanallar/odalar), geçerli mesaj gövdesinin başına geçmiş girdilerinde kullanılan stille eşleşen gönderici etiketi eklenir. Yönerge çıkarma yalnızca geçerli mesaj bölümüne uygulanır, böylece geçmiş bozulmadan kalır. Geçmişi sarmalayan kanallar, `BodyForCommands` (veya eski `CommandBody` / `RawBody`) alanını özgün mesaj metnine ayarlamalı ve `Body` alanını birleştirilmiş istem olarak tutmalıdır.

Geçmiş tamponları yalnızca bekleyenleri içerir: çalıştırmayı tetiklemeyen grup mesajlarını (örneğin, bahsetme koşullu mesajları) içerir ve zaten oturum dökümünde bulunan mesajları dışlar. Yapılandırılmış geçmiş, yanıt, iletilen mesaj ve kanal meta verileri, istem oluşturma sırasında güvenilmeyen kullanıcı rolü bağlam blokları olarak işlenir.

Geçmiş boyutunu `messages.groupChat.historyLimit` (genel varsayılan) veya `channels.slack.historyLimit` ve `channels.telegram.accounts.<id>.historyLimit` gibi kanal başına geçersiz kılmalarla yapılandırın (devre dışı bırakmak için `0` değerini ayarlayın).

## Araç sonucu meta verileri

Araç sonucundaki `content`, modelin görebildiği sonuçtur; `details` ise UI oluşturma, tanılama, medya teslimatı ve Plugin'ler için çalışma zamanı meta verisidir.

- `toolResult.details`, sağlayıcı yeniden oynatımından ve Compaction girdisinden önce çıkarılır.
- Kalıcı oturum dökümleri yalnızca sınırlandırılmış `details` verilerini tutar; aşırı büyük meta veriler, `persistedDetailsTruncated: true` olarak işaretlenen kısa bir özetle değiştirilir.
- Plugin'ler ve araçlar, modelin okuması gereken metni yalnızca `details` içine değil, `content` içine koymalıdır.

## Kuyruğa alma ve takipler

Bir çalıştırma zaten etkin olduğunda, gelen mesajlar varsayılan olarak ona yön verir. `messages.queue` modu denetler:

| Mod               | Davranış                                            |
| ----------------- | --------------------------------------------------- |
| `steer` (varsayılan) | Yeni istemi etkin çalıştırmaya ekler.               |
| `followup`        | Mesajı etkin çalıştırma bittikten sonra çalıştırır. |
| `collect`         | Uyumlu mesajları daha sonraki tek bir sırada toplar. |
| `interrupt`       | Etkin çalıştırmayı iptal eder, ardından en yeni istemi başlatır. |

Kuyruk; yön verme, takip ve toplama gruplaması için yerleşik 500ms bekletme kullanır. `messages.queue.cap` varsayılan olarak 20 kuyruklanmış mesajdır ve `messages.queue.drop` varsayılan olarak `summarize` değeridir (`old` ve `new` da kullanılabilir). Kanal başına geçersiz kılmaları `messages.queue.byChannel` ve `messages.queue.debounceMsByChannel` aracılığıyla yapılandırın.

Ayrıntılar: [Komut kuyruğu](/tr/concepts/queue) ve [Yönlendirme kuyruğu](/tr/concepts/queue-steering).

## Kanal çalıştırma sahipliği

Kanal Plugin'leri, bir mesaj oturum kuyruğuna girmeden önce sıralamayı koruyabilir, girdiyi bekletebilir ve taşıma geri basıncı uygulayabilir. Aracı sırasının kendisine ayrı bir zaman aşımı uygulamamalıdırlar. Bir mesaj oturuma yönlendirildikten sonra uzun süren işleri oturum, araç ve çalışma zamanı yaşam döngüsü yönetir; böylece tüm kanallar yavaş sıraları tutarlı biçimde bildirir ve bunlardan kurtulur.

## Akış, parçalara ayırma ve gruplama

Blok akışı, model metin blokları üretirken kısmi yanıtlar gönderir; parçalara ayırma, kanal metin sınırlarına uyar ve çitli kodu bölmekten kaçınır.

- `agents.defaults.blockStreamingDefault` (`on|off`, varsayılan `off`)
- `agents.defaults.blockStreamingBreak` (`text_end|message_end`)
- `agents.defaults.blockStreamingChunk` (`minChars|maxChars|breakPreference`)
- `agents.defaults.blockStreamingCoalesce` (boşta kalmaya dayalı gruplama)
- `agents.defaults.humanDelay` (blok yanıtları arasında insan benzeri duraklama)
- Kanal geçersiz kılmaları: paketlenmiş kanallarda `*.streaming.block.enabled` ve `*.streaming.block.coalesce`; eski düz anahtarlar `openclaw doctor --fix` tarafından taşınır. Blok akışı, Telegram dahil her kanalda açıkça etkinleştirilmedikçe kapalıdır. QQ Bot istisnadır: `streaming.block` anahtarları yoktur ve `channels.qqbot.streaming.mode` değeri `"off"` olmadığı sürece blok yanıtlarını akışla gönderir.

Ayrıntılar: [Akış + parçalara ayırma](/tr/concepts/streaming).

## Akıl yürütme görünürlüğü ve tokenlar

- `/reasoning on|off|stream` görünürlüğü denetler.
- Model ürettiğinde, akıl yürütme içeriği yine de token kullanımına dahil edilir.
- Telegram, akıl yürütmenin son teslimattan sonra silinen geçici bir taslak balonuna akışla gönderilmesini destekler; kalıcı akıl yürütme çıktısı için `/reasoning on` kullanın.

Ayrıntılar: [Düşünme + akıl yürütme yönergeleri](/tr/tools/thinking) ve [Token kullanımı](/tr/reference/token-use).

## Ön ekler, ileti dizileri ve yanıtlar

- Giden ön ekler `channels.<channel>.responsePrefix` ve `channels.<channel>.accounts.<id>.responsePrefix` konumlarında bulunur. Hesap değerleri önceliklidir. Doctor, bu standart alanlar ayarlanmamışsa genel geri dönüş değerini yapılandırılmış kanal bloklarına kopyalar; `messages.responsePrefix`, örtük ve özel kanallar için geri dönüş olarak kalır.
- `replyToMode` ve kanal başına varsayılanlar aracılığıyla yanıt ileti dizisi oluşturma.

Ayrıntılar: [Yapılandırma](/tr/gateway/config-agents#messages) ve kanal belgeleri.

## Sessiz yanıtlar

Sessiz token `NO_REPLY` (büyük/küçük harfe duyarlı değildir; dolayısıyla `no_reply` de eşleşir), "kullanıcı tarafından görülebilir bir yanıt teslim etme" anlamına gelir. Bir sırada oluşturulmuş TTS sesi gibi bekleyen araç medyası da bulunduğunda OpenClaw sessiz metni çıkarır ancak medya ekini teslim etmeye devam eder.

Sessizlik politikası konuşma türüne göre çözümlenir:

- Doğrudan konuşmalar hiçbir zaman `NO_REPLY` istem yönlendirmesi almaz. Doğrudan bir çalıştırma yanlışlıkla tek başına bir sessiz token döndürürse OpenClaw bunu yeniden yazmak veya teslim etmek yerine bastırır.
- Gruplar/kanallar varsayılan olarak sessizliğe izin verir. `message_tool` görünür yanıt modunda sessizlik, modelin `message(action=send)` çağrısını yapmaması anlamına gelir.
- Dahili orkestrasyon varsayılan olarak sessizliğe izin verir.

Varsayılanlar `agents.defaults.silentReply` altında bulunur; `surfaces.<id>.silentReply`, yüzey başına grup/dahili politikasını geçersiz kılabilir.

OpenClaw ayrıca doğrudan olmayan sohbetlerdeki genel dahili çalıştırıcı hataları için sessiz yanıtlar kullanır; böylece gruplar/kanallar Gateway hata şablonunu görmez. Eksik kimlik doğrulama, hız sınırı veya aşırı yük bildirimleri gibi kullanıcıya yönelik kurtarma metni bulunan sınıflandırılmış hatalar yine de teslim edilebilir. Doğrudan sohbetler varsayılan olarak kısa hata metni gösterir; ham çalıştırıcı ayrıntıları yalnızca `/verbose full` etkinleştirildiğinde gösterilir.

Tek başına sessiz yanıtlar tüm yüzeylerde bırakılır; böylece üst oturumlar, nöbetçi metni geri dönüş sohbetine dönüştürmek yerine sessiz kalır.

## İlgili

- [Mesaj yaşam döngüsü yeniden düzenlemesi](/tr/concepts/message-lifecycle-refactor) - kalıcı gönderme ve alma tasarımı hedefi
- [Akış](/tr/concepts/streaming) - gerçek zamanlı mesaj teslimatı
- [Yeniden deneme](/tr/concepts/retry) - mesaj teslimatını yeniden deneme davranışı
- [Kuyruk](/tr/concepts/queue) - mesaj işleme kuyruğu
- [Kanallar](/tr/channels) - mesajlaşma platformu entegrasyonları
