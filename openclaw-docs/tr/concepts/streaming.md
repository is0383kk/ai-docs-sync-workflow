---
read_when:
    - Kanallarda akışın veya parçalara ayırmanın nasıl çalıştığını açıklama
    - Blok akışı veya kanal parçalama davranışını değiştirme
    - Yinelenen/erken blok yanıtlarında veya kanal önizleme akışında hata ayıklama
summary: Akış + parçalara ayırma davranışı (blok yanıtları, kanal önizleme akışı, mod eşlemesi)
title: Akış ve parçalara ayırma
x-i18n:
    generated_at: "2026-07-26T22:44:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a498f2e490ae6f2ecdebba92f0b992f2e16d212eae6a437eb3a0ef8a59354e13
    source_path: concepts/streaming.md
    workflow: 16
---

OpenClaw'ın iki bağımsız akış katmanı vardır ve günümüzde kanal mesajlarına **gerçek
token-farkı akışı uygulanmaz**:

- **Blok akışı (kanallar):** asistan yazarken tamamlanmış **blokları**
  yayınlar. Bunlar token farkları değil, normal kanal mesajlarıdır.
- **Önizleme akışı (Telegram/Discord/Slack/Matrix/Mattermost/MS Teams):**
  oluşturma sırasında geçici bir **önizleme mesajını** günceller (gönderme + düzenlemeler/eklemeler).

## Control UI başlatma durumu

`chat.send` etkin bir çalışmayı onayladıktan sonra Gateway, asistan metni veya araç
etkinliği görünür olmadan önce türü belirlenmiş, genel bir başlatma durumu gönderebilir.
Control UI bu durumu çalışma göstergesinin yanında; çalışma alanı hazırlığı, ortam sağlama,
bağlam hazırlığı ve model başlatma aşamalarıyla gösterir.

İlk asistan farkı veya araç başlangıcı, söz konusu çalışma için başlatma durumunun kalıcı
olarak yerini alır. Bir araç operatör eylemi beklerken onay durumu önceliklidir.
Çalışma ağacı oluşturma ve ilk bulut yönlendirmesi, bir sohbet çalışması var olmadan önce
gerçekleştiği için bunların çalışma öncesi RPC ilerlemesi çalışma başlatma durumu olarak
sunulmaz; ortam sağlama burada yalnızca etkin bir çalışma geri kazanılmış bir çalışanı
yeniden sağladığında görünür.

## Blok akışı (kanal mesajları)

Blok akışı, asistan çıktısını kullanılabilir hâle geldikçe genel parçalar hâlinde gönderir.

```text
Model çıktısı
  └─ text_delta/events
       ├─ (blockStreamingBreak=text_end)
       │    └─ arabellek büyüdükçe parçalayıcı blokları yayınlar
       └─ (blockStreamingBreak=message_end)
            └─ parçalayıcı message_end sırasında arabelleği boşaltır
                   └─ kanala gönderme (blok yanıtları)
```

- `text_delta/events`: model akışı olayları (akış kullanmayan modellerde seyrek olabilir).
- `chunker`: alt/üst sınırları + kesme tercihini uygulayan `EmbeddedBlockChunker`.
- `channel send`: gerçek giden mesajlar (blok yanıtları).

**Denetimler** (aksi belirtilmedikçe tümü `agents.defaults` altında):

| Anahtar                                                      | Değerler / biçim                                                         | Varsayılan |
| ------------------------------------------------------------ | ------------------------------------------------------------------------ | ---------- |
| `blockStreamingDefault`                                      | `"on"` / `"off"`                                                        | `"off"`    |
| `blockStreamingBreak`                                        | `"text_end"` / `"message_end"`                                          | -          |
| `blockStreamingChunk`                                        | `{ minChars, maxChars, breakPreference? }`                              | -          |
| `blockStreamingCoalesce`                                     | `{ minChars?, maxChars?, idleMs? }` (göndermeden önce akış bloklarını birleştirir) | -          |
| `*.streaming.block.enabled` (kanal geçersiz kılması)               | `true` / `false`, kanal (ve hesap) başına blok akışını zorlar  | -          |
| `*.textChunkLimit` (ör. `channels.whatsapp.textChunkLimit`) | sayı, kesin üst sınır                                                     | 4000       |
| `*.streaming.chunkMode`                                      | `"length"` / `"newline"`                                                | `"length"` |
| `channels.discord.maxLinesPerMessage`                        | sayı, UI kırpmasını önlemek için uzun yanıtları bölen esnek satır sınırı | 17         |

`streaming.chunkMode: "newline"`, metin sınırı aştığında uzunluğa göre parçalamaya
başvurmadan önce her satır sonundan değil, boş satırlardan (paragraf
sınırlarından) böler.

Paketlenmiş kanallar bu geçersiz kılmaları
`channels.<id>.streaming.{chunkMode,block.enabled,block.coalesce}` biçiminde yazar. Düz
`*.chunkMode` / `*.blockStreaming` / `*.blockStreamingCoalesce` yazımları
tüm paketlenmiş kanallarda eskidir: `openclaw doctor --fix` bunları iç içe
biçime taşır ve kanal şemaları bunları reddeder. Hâlâ düz yazımları kullanan
harici SDK plugin yapılandırmaları, bir sonraki sürüm serisine kadar kullanımdan
kaldırılmış bir geri dönüş (çalışma zamanı uyarısıyla) üzerinden çalışmaya devam eder.

`blockStreamingBreak` için **sınır semantiği**:

- `text_end`: parçalayıcı yayınlar yayınlamaz blokları aktarır; her `text_end` sırasında boşaltır.
- `message_end`: asistan mesajı bitene kadar bekler, ardından arabelleğe alınan
  çıktıyı boşaltır. Arabelleğe alınan metin `maxChars` değerini aşarsa yine
  parçalayıcıyı kullanır; dolayısıyla sonunda birden fazla parça yayınlayabilir.

### Blok akışıyla medya teslimi

Akış medyası, `mediaUrl` veya `mediaUrls` gibi yapılandırılmış yük alanlarını
kullanmalıdır; akış metni bir ek komutu olarak ayrıştırılmaz. Blok akışı medyayı erken
gönderdiğinde OpenClaw bu teslimi tur boyunca hatırlar. Son asistan yükü aynı medya
URL'sini yinelerse son teslim, eki tekrar göndermek yerine yinelenen medyayı kaldırır.

Tam olarak yinelenen son yükler engellenir. Son yük, daha önce akışla gönderilmiş medyanın
çevresine farklı bir metin eklerse OpenClaw medyayı yalnızca bir kez teslim ederken yeni
metni yine gönderir. Bu, Telegram gibi kanallarda yinelenen sesli notları veya dosyaları
önler.

## Parçalama algoritması (alt/üst sınırlar)

Blok parçalama, `EmbeddedBlockChunker` tarafından uygulanır:

- **Alt sınır:** arabellek >= `minChars` olana kadar yayınlama (zorlanmadıkça).
- **Üst sınır:** `maxChars` öncesindeki bölmeleri tercih et; zorlanırsa `maxChars` konumunda böl.
- **Kesme tercihi zinciri:** `paragraph` -> `newline` -> `sentence` ->
  boşluk -> kesin kesme.
- **Kod çitleri:** çitlerin içinden asla bölme; `maxChars` konumunda zorlandığında
  Markdown'ı geçerli tutmak için çiti kapatıp yeniden aç.

`maxChars`, kanalın `textChunkLimit` değerine sınırlandırılır; dolayısıyla kanal
başına sınırlar aşılamaz.

## Birleştirme (akış bloklarını birleştirme)

Blok akışı etkinleştirildiğinde OpenClaw, ardışık blok parçalarını göndermeden önce
**birleştirerek** aşamalı çıktı sağlamaya devam ederken tek satırlık mesaj kalabalığını
azaltabilir.

- Birleştirme, boşaltmadan önce **boşta kalma aralıklarını** (`idleMs`) bekler.
- Arabellekler `maxChars` ile sınırlandırılır ve bunu aşarlarsa boşaltılır.
- `minChars`, yeterli metin birikene kadar küçük parçaların gönderilmesini
  önler (son boşaltma kalan metni her zaman gönderir).
- Birleştirici `blockStreamingChunk.breakPreference` değerinden türetilir: `paragraph` ->
  `\n\n`, `newline` -> `\n`, `sentence` -> boşluk.
- Kanal geçersiz kılmaları `*.streaming.block.coalesce` üzerinden kullanılabilir
  (hesap başına yapılandırmalar dâhil).
- Discord, Signal ve Slack, geçersiz kılınmadıkça varsayılan olarak `{ minChars: 1500, idleMs: 1000 }`
  değerine göre birleştirilir.

## Bloklar arasında insana benzer tempo

Blok akışı etkinleştirildiğinde, çok baloncuklu yanıtların daha doğal görünmesi için
ilk bloktan sonra blok yanıtları arasına **rastgele bir duraklama** eklenir.

| `agents.defaults.humanDelay.mode` | Davranış                |
| --------------------------------- | ----------------------- |
| `off` (varsayılan)                   | Duraklama yok                |
| `natural`                         | 800-2500ms rastgele duraklama |
| `custom`                          | `minMs`/`maxMs`         |

Agent başına `agents.entries.*.humanDelay` üzerinden geçersiz kılın. Yalnızca **blok
yanıtlarına** uygulanır; son yanıtlara veya araç özetlerine uygulanmaz.

## "Parçaları veya her şeyi aktar"

- **Parçaları aktar:** `blockStreamingDefault: "on"` + `blockStreamingBreak: "text_end"`
  (ilerledikçe yayınla). Telegram dışındaki kanallar ayrıca
  `*.streaming.block.enabled: true` gerektirir.
- **Her şeyi sonunda aktar:** `blockStreamingBreak: "message_end"` (bir kez
  boşaltır; çok uzunsa birden fazla parça olabilir).
- **Blok akışı yok:** `blockStreamingDefault: "off"` (yalnızca son yanıt).

`*.streaming.block.enabled` açıkça `true` olarak ayarlanmadıkça blok akışı
**kapalıdır** (istisna: QQ Bot'ta `streaming.block` anahtarları yoktur ve
`channels.qqbot.streaming.mode`, `"off"` olmadığı sürece blok yanıtlarını aktarır).
Kanallar, blok yanıtları olmadan canlı önizleme (`channels.<channel>.streaming.mode`) aktarabilir.
`blockStreaming*` varsayılanları yapılandırma kökünde değil,
`agents.defaults` altında bulunur.

## Önizleme akışı modları

Standart anahtar: `channels.<channel>.streaming` (iç içe `{ mode, ... }`; eski üst düzey
boole/dize yazımları `openclaw doctor --fix` tarafından yeniden yazılır).

| Mod        | Davranış                                                              |
| ---------- | --------------------------------------------------------------------- |
| `off`      | Önizleme akışını devre dışı bırakır                                   |
| `partial`  | Tek önizlemeyi en son metinle değiştirir                               |
| `block`    | Önizlemeyi parçalı/eklemeli adımlarla günceller                        |
| `progress` | Oluşturma sırasında ilerleme/durum önizlemesi, tamamlandığında son yanıt |

`streaming.mode: "block"`, Discord ve Telegram gibi düzenlemeyi destekleyen
kanallar için bir önizleme akışı modudur; tek başına bu kanallarda blok
teslimini etkinleştirmez. Normal blok yanıtları için `streaming.block.enabled`
kullanın. Microsoft Teams istisnadır: taslak önizleme blok aktarımı olmadığı
için `streaming.mode:
"block"` yerel akışı tamamen devre dışı bırakır ve yanıt,
yerel kısmi/ilerleme akışı yerine normal blok teslimi olarak ulaşır. Mattermost
da farklıdır: `block` modunda önizlemeyi tamamlanmış metin ile araç
etkinliği blokları arasında döndürür; böylece önceki bloklar tek bir
düzenlenebilir taslakta üzerlerine yazılmak yerine ayrı gönderiler olarak
görünür kalır.

### Kanal eşlemesi

| Kanal      | `off` | `partial` | `block` | `progress`              |
| ---------- | ----- | --------- | ------- | ----------------------- |
| Telegram   | Evet  | Evet      | Evet    | düzenlenebilir ilerleme taslağı |
| Discord    | Evet  | Evet      | Evet    | düzenlenebilir ilerleme taslağı |
| Slack      | Evet  | Evet      | Evet    | Evet                    |
| Mattermost | Evet  | Evet      | Evet    | Evet                    |
| MS Teams   | Evet  | Evet      | Evet    | yerel ilerleme akışı    |

Önizleme parça yapılandırmasının (`streaming.preview.chunk.*`; ör. `channels.discord.streaming`
veya `channels.telegram.streaming` altında) varsayılanları `minChars: 200`,
`maxChars: 800` (kanalın `textChunkLimit` değerine sınırlandırılır) ve
`breakPreference: "paragraph"` değerleridir.

Yalnızca Slack:

- `channels.slack.streaming.nativeTransport`, `channels.slack.streaming.mode="partial"` olduğunda Slack yerel akış API'si
  çağrılarını (`chat.startStream`/`chat.appendStream`/`chat.stopStream`)
  açar veya kapatır (varsayılan: `true`).
- Slack yerel akışı ve Slack asistan ileti dizisi durumu, bir yanıt
  ileti dizisi hedefi gerektirir. Üst düzey DM'ler bu ileti dizisi tarzı önizlemeyi
  göstermez ancak yine de Slack taslak önizleme gönderilerini ve düzenlemelerini
  kullanabilir.

### Eski anahtar taşıma

| Kanal    | Eski anahtarlar                                             | Durum                                                                                                                                               |
| -------- | ----------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| Telegram | `streamMode`, skaler/boole `streaming`         | `openclaw doctor --fix` tarafından `streaming.mode` olarak yeniden yazılır; çalışma zamanında okunmaz                                                  |
| Discord  | `streamMode`, boole `streaming`                | `openclaw doctor --fix` tarafından `streaming.mode` olarak yeniden yazılır; çalışma zamanında okunmaz                                                  |
| Slack    | `streamMode`; boole `streaming`; eski `nativeStreaming` | `openclaw doctor --fix` tarafından `streaming.mode` (boole/eski biçimler için de `streaming.nativeTransport`) olarak yeniden yazılır; çalışma zamanında okunmaz |
| Matrix   | skaler/boole `streaming`                             | `openclaw doctor --fix` tarafından `streaming.mode` (Matrix'in `"quiet"` modu dâhil) olarak yeniden yazılır; çalışma zamanında okunmaz         |
| Feishu   | boole `streaming`                                    | `openclaw doctor --fix` tarafından `streaming.mode` olarak yeniden yazılır; çalışma zamanında okunmaz                                                  |
| QQ Bot   | boole `streaming`; `streaming.c2cStreamApi`                | `openclaw doctor --fix` tarafından `streaming.mode` (boole/`c2cStreamApi` biçimleri için de `streaming.nativeTransport`) olarak yeniden yazılır; çalışma zamanında okunmaz |

## Çalışma zamanı davranışı

### Telegram

- Özel mesajlar ve grup/konular genelinde `sendMessage` + `editMessageText` önizleme güncellemelerini kullanır;
  son metin etkin önizlemeyi yerinde düzenler. Telegram'ın
  30 saniyelik geçici "yazıyor" taslakları (`sendMessageDraft`) yanıt
  akışı için kullanılmaz.
- Kısa ilk önizlemeler, anlık bildirim kullanıcı deneyimi için hâlâ geciktirilir ancak
  etkin çalıştırmaların görsel olarak sessiz kalmaması için sınırlı bir gecikmenin ardından
  görünür hâle gelir.
- Uzun nihai yanıtlar ilk parça için önizleme mesajını yeniden kullanır ve yalnızca
  kalan parçaları gönderir.
- `block` modu, önizlemeyi
  `streaming.preview.chunk.maxChars` değerinde (varsayılan 800, Telegram'ın 4096
  düzenleme sınırıyla kısıtlı) yeni bir mesaja dönüştürür; diğer modlar tek bir önizlemeyi 4096 karaktere kadar büyütür.
- `progress` modu araç ilerlemesini düzenlenebilir bir durum taslağında tutar, yanıt akışı
  etkinken ancak henüz bir araç satırı yokken durum etiketini görünür hâle getirir,
  tamamlandığında taslağı temizler ve nihai yanıtı
  normal teslimat yoluyla gönderir.
- Tamamlanmış metin doğrulanmadan önce son düzenleme başarısız olursa OpenClaw
  normal nihai teslimatı kullanır ve eski önizlemeyi temizler.
- Çift akışı önlemek için Telegram blok akışı açıkça
  etkinleştirildiğinde önizleme akışı atlanır.
- `/reasoning stream`, akıl yürütmeyi nihai teslimattan sonra
  silinen geçici bir önizlemeye yazabilir.
- Telegram seçili alıntı yanıtları bir istisnadır: `replyToMode`, `"off"`
  değilse ve seçili alıntı metni varsa OpenClaw o turda yanıt önizleme
  akışını atlar (nihai yanıt yerel alıntı-yanıt
  yolundan geçmelidir); bu nedenle araç ilerleme önizleme satırları görüntülenemez. Seçili alıntı metni
  bulunmayan geçerli mesaj yanıtlarında önizleme akışı devam eder. Ayrıntılar için
  [Telegram kanal belgelerine](/tr/channels/telegram) bakın.

### Discord

- Gönderme + düzenleme önizleme mesajlarını kullanır.
- `block` modu taslak parçalamayı (`draftChunk`) kullanır.
- Discord blok akışı açıkça
  etkinleştirildiğinde önizleme akışı atlanır.
- `progress` modu, nihai yanıta küçük bir `-#` etkinlik kaydı (düşünce/araç çağrısı
  sayıları ve geçen süre) ekler ve yanıt teslim edildikten sonra durum taslağını
  siler; böylece yoğun kanallarda yanıtın üzerinde sahipsiz araç günlüğü
  kalmaz. Hatalı nihai yanıtlar, başarısız turun kaydı olarak taslağı korur.
- Nihai medya, hata ve açık yanıt yükleri, yeni bir taslağı aktarmadan
  bekleyen önizlemeleri iptal eder ve ardından normal teslimatı kullanır.

### Slack

- `partial`, kullanılabilir olduğunda Slack'in yerel akışını (`chat.startStream`/`append`/`stop`)
  kullanabilir.
- `block`, ekleme tarzı taslak önizlemelerini kullanır.
- `progress`, durum önizleme metnini ve ardından nihai yanıtı kullanır.
- Yanıt dizisi bulunmayan üst düzey özel mesajlar, Slack'in yerel akışı yerine
  taslak önizleme gönderileri ve düzenlemeleri kullanır.
- Yerel ve taslak önizleme akışı, o tur için blok yanıtlarını engeller; böylece bir
  Slack yanıtı yalnızca tek bir teslimat yoluyla aktarılır.
- Nihai medya/hata yükleri ve ilerleme nihai yanıtları, geçici taslak
  mesajlar oluşturmaz; yalnızca önizlemeyi düzenleyebilen metin/blok nihai yanıtları bekleyen
  taslak metnini aktarır.

### Mattermost

- `partial` modunda düşünme ve kısmi yanıt metnini, nihai yanıtın güvenle gönderilebildiği anda
  yerinde sonlandırılan tek bir taslak önizleme gönderisine aktarır.
- `progress` modunda düşünme ve araç etkinliğini, nihai yanıtın güvenle gönderilebildiği anda
  yerinde sonlandırılan tek bir durum önizlemesine aktarır.
- `block` modunda tamamlanmış metin ile araç etkinliği gönderileri arasında geçiş yapar;
  paralel ve ardışık araç güncellemeleri mevcut araç etkinliği gönderisini paylaşır.
- Önizleme gönderisi silinmişse veya sonlandırma anında başka bir nedenle
  kullanılamıyorsa yeni bir nihai gönderi göndermeye geri döner.
- Nihai medya/hata yükleri, geçici bir önizleme gönderisini aktarmak yerine normal
  teslimattan önce bekleyen önizleme güncellemelerini iptal eder.

### Matrix

- Nihai metin önizleme olayını yeniden kullanabildiğinde taslak önizlemeleri
  yerinde sonlandırılır.
- Yalnızca medya içeren, hatalı ve yanıt hedefi uyuşmayan nihai yanıtlar, normal teslimattan önce bekleyen önizleme
  güncellemelerini iptal eder; zaten görünür olan eski önizleme karartılır.

## Araç ilerleme önizleme güncellemeleri

Önizleme akışı ayrıca **araç ilerlemesi** güncellemelerini de içerebilir: araçlar
çalışırken aynı önizleme mesajında, nihai yanıttan önce görünen "web'de aranıyor",
"dosya okunuyor" veya "araç çağrılıyor" gibi kısa durum satırları.
Codex uygulama sunucusu modunda Codex önsöz/açıklama mesajları da aynı
önizleme yolunu kullanır; böylece kısa "Kontrol ediyorum..." ilerleme notları, nihai
yanıtın parçası olmadan düzenlenebilir taslağa aktarılabilir. Bu, çok adımlı
araç turlarının ilk düşünme önizlemesi ile nihai yanıt arasında sessiz kalmak yerine
görsel olarak etkin görünmesini sağlar.

Uzun süre çalışan araçlar dönmeden önce tür belirtilmiş ilerleme bilgisi yayabilir. Örneğin,
`web_fetch` başladığında beş saniyelik bir zamanlayıcı kurar: getirme işlemi hâlâ
beklemedeyse önizlemede `Fetching page content...` gösterilir; getirme işlemi bundan önce tamamlanır
veya iptal edilirse ilerleme satırı yayımlanmaz. Daha sonraki nihai araç
sonucu yine modele normal şekilde teslim edilir.

Desteklenen yüzeyler:

- **Discord**, **Slack**, **Telegram** ve **Matrix**, önizleme
  akışı etkinken araç ilerlemesini ve Codex önsöz güncellemelerini varsayılan olarak canlı önizleme düzenlemesine aktarır.
  Microsoft Teams, kişisel sohbetlerde yerel ilerleme akışını kullanır.
- Telegram, `v2026.4.22` sürümünden bu yana araç ilerleme önizleme güncellemeleri etkin olarak
  yayımlanmaktadır; bunları etkin tutmak, yayımlanmış davranışı korur.
- **Mattermost**, araç etkinliğini `partial` ve
  `progress` modlarında tek bir önizleme gönderisinde veya `block`
  modunda metin blokları arasındaki tek bir araç etkinliği gönderisinde birleştirir (yukarıya bakın).
- Araç ilerleme düzenlemeleri etkin önizleme akışı modunu izler; önizleme akışı
  `off` olduğunda veya blok akışı mesajı devraldığında
  atlanır. Telegram'da `streaming.mode: "off"` yalnızca nihai yanıt içindir: genel
  ilerleme mesajları da bağımsız durum mesajları olarak teslim edilmek yerine
  engellenir; onay istemleri, medya yükleri ve hatalar ise normal şekilde
  yönlendirilir.
- Önizleme akışını koruyup araç ilerleme satırlarını gizlemek için
  ilgili kanalda `streaming.preview.toolProgress` değerini `false` olarak ayarlayın (varsayılan
  `true`). Komut/çalıştırma metnini gizlerken araç ilerleme satırlarını görünür tutmak için
  `streaming.preview.commandText` değerini `"status"` veya
  `streaming.progress.commandText` değerini `"status"` olarak ayarlayın; yayımlanmış
  davranışı korumak için varsayılan değer `"raw"`'tür. Bu politika, Discord, Matrix,
  Microsoft Teams, Mattermost, Slack taslak önizlemeleri ve Telegram dâhil olmak üzere
  OpenClaw'ın kompakt ilerleme oluşturucusunu kullanan taslak/ilerleme kanalları tarafından paylaşılır.
  Önizleme düzenlemelerini tamamen devre dışı bırakmak için `streaming.mode` değerini
  `off` olarak ayarlayın.

## İlerleme taslağının oluşturulması

İlerleme modu taslakları (`streaming.progress.*`) kanal başına sınırlı ve
yapılandırılabilirdir:

| Anahtar                           | Varsayılan    | Davranış                                                               |
| --------------------------------- | ------------- | ---------------------------------------------------------------------- |
| `streaming.progress.maxLines`                | `8` | Taslak etiketinin altında tutulan azami kompakt ilerleme satırı sayısı |
| `streaming.progress.maxLineChars`                | `120` | Kısaltma öncesinde kompakt satır başına azami karakter sayısı (kelime duyarlı) |
| `streaming.progress.label`                | `"auto"` | Taslak başlığı; özel bir dize veya gizlemek için `false`    |
| `streaming.progress.labels`                | yerleşik havuz | `label: "auto"` olduğunda kullanılan aday etiketler                |

### Açıklama ilerleme şeridi

Araç ilerlemesinin ötesinde, kompakt ilerleme oluşturucu taslakta bir şerit daha
gösterebilir:

- **`streaming.progress.commentary`** - modelin araç öncesi
  **açıklamasını** (kısa bir "Kontrol edeceğim... ardından..." anlatımı), ilerleme taslağındaki
  araç satırlarıyla iç içe görüntüler. İlerleme modunda Discord ve Telegram'da,
  bu isteğe bağlı şerit kapalı olsa bile aynı önsöz durum başlığını sağlar;
  diğer kanallar mevcut ilerleme davranışlarını korur. Bkz.
  [İlerleme taslakları](/tr/concepts/progress-drafts#status-headline).

```json
{
  "channels": {
    "discord": {
      "streaming": { "mode": "progress", "progress": { "commentary": true } }
    }
  }
}
```

İlerleme satırlarını görünür tutarken ham komut/çalıştırma metnini gizleyin:

```json
{
  "channels": {
    "telegram": {
      "streaming": {
        "mode": "partial",
        "preview": {
          "toolProgress": true,
          "commandText": "status"
        }
      }
    }
  }
}
```

Aynı yapıyı başka bir kompakt ilerleme kanalı anahtarı altında kullanın; örneğin
`channels.discord`, `channels.matrix`, `channels.msteams`,
`channels.mattermost` veya Slack taslak önizlemeleri. İlerleme taslağı modu için
aynı politikayı `streaming.progress` altına yerleştirin:

```json
{
  "channels": {
    "telegram": {
      "streaming": {
        "mode": "progress",
        "progress": {
          "toolProgress": true,
          "commandText": "status"
        }
      }
    }
  }
}
```

## İlgili

- [Mesaj yaşam döngüsü yeniden düzenlemesi](/tr/concepts/message-lifecycle-refactor) - paylaşılan önizleme, düzenleme, akış ve sonlandırma tasarımını hedefler
- [İlerleme taslakları](/tr/concepts/progress-drafts) - uzun turlar sırasında güncellenen görünür devam eden çalışma mesajları
- [Mesajlar](/tr/concepts/messages) - mesaj yaşam döngüsü ve teslimatı
- [Yeniden deneme](/tr/concepts/retry) - teslimat başarısızlığında yeniden deneme davranışı
- [Kanallar](/tr/channels) - kanal başına akış desteği
