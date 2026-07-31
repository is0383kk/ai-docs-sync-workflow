---
read_when:
    - Uzun süren sohbet turları için görünür ilerleme güncellemelerini yapılandırma
    - Kısmi, blok ve ilerleme akışı modları arasında seçim yapma
    - OpenClaw'ın çalışma devam ederken tek bir kanal mesajını nasıl güncellediğini açıklama
    - Sorun giderme ilerleme taslakları, bağımsız ilerleme mesajları veya sonlandırma yedeği
summary: 'İlerleme taslakları: bir aracı çalışırken güncellenen, görünür tek bir devam eden çalışma mesajı'
title: İlerleme taslakları
x-i18n:
    generated_at: "2026-07-26T23:18:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4ef66dd4d7a31c753f5faa0b88b83ec3760beecf3118cf8aae84f5e57652e809
    source_path: concepts/progress-drafts.md
    workflow: 16
---

İlerleme taslakları, bir agent çalışırken tek bir kanal mesajını canlı bir durum
satırına dönüştürür; böylece geçici "hâlâ çalışıyor" yanıtlarından oluşan bir yığın
yerine tek mesaj kullanılır. `channels.<channel>.streaming.mode: "progress"` değerini ayarladığınızda OpenClaw,
gerçek çalışma başlar başlamaz mesajı oluşturur; agent okurken, planlarken, araçları
çağırırken veya onay beklerken mesajı düzenler ve ardından nihai yanıta dönüştürür.

```text
Çalışıyor...
📖 docs/concepts/progress-drafts.md dosyasından
🔎 Web Araması: "discord mesaj düzenleme" için
🛠️ Bash: testleri çalıştır
```

<Note>
  `channels.discord.streaming` ayarlanmamışsa Discord zaten varsayılan olarak
  `streaming.mode: "progress"` değerini kullanır; dolayısıyla ilerleme taslakları
  herhangi bir yapılandırma olmadan burada görünür. Diğer tüm kanalların varsayılanı
  `partial` veya `off` değeridir; kanallara göre varsayılanların
  tam tablosu için [Akış ve parçalara ayırma](/tr/concepts/streaming#channel-mapping)
  bölümüne bakın.
</Note>

## Hızlı başlangıç

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
      },
    },
  },
}
```

Buradan itibaren varsayılanlar: 5 saniyelik başlangıç gecikmesi, yararlı çalışmalar
yapılırken kısa ilerleme satırları ve o tur için eski bağımsız ilerleme mesajlarının
gizlenmesi. Ham araç satırı taslakları otomatik tek kelimelik bir etiket kullanır;
açıkça bir etiket yapılandırmadığınız sürece durum başlığı bu gereksiz başlığı
kullanmaz.

Bu sayfa, ilerleme taslağı deneyimini ve yapılandırma seçeneklerini kapsar. Tam
akış modu matrisi, kanallara göre çalışma zamanı notları ve eski anahtar
geçişleri için [Akış ve parçalara ayırma](/tr/concepts/streaming) bölümüne bakın.

## Kullanıcıların gördükleri

| Bölüm           | Amaç                                                                                      |
| --------------- | ----------------------------------------------------------------------------------------- |
| Durum başlığı   | Discord ve Telegram'da model önsözü; Discord bir yardımcı dolgu metni ekler.              |
| Etiket          | `Working` gibi isteğe bağlı başlangıç/durum satırı.                              |
| İlerleme satırları | `/verbose` ile aynı araç simgelerini ve ayrıntı biçimlendiricisini kullanan kısa çalışma güncellemeleri. |

Ham araç ilerlemesinde etiket, agent anlamlı bir çalışmaya başlayıp ilk gecikme
süresince meşgul kaldığında görünür.
Kayan ilerleme satırları listesinin en üstünde yer aldığından, yeterli sayıda
somut çalışma satırı göründüğünde kaydırılarak görünümden çıkar. Bir durum başlığı,
açıkça bir etiket yapılandırılmadığı sürece yalnızca agent'ın sade dille yazılmış
durumunu gösterir. Yalnızca düz metin içeren yanıtlar hiçbir zaman ilerleme taslağı
göstermez; bir satır yalnızca `🛠️ Bash: run tests`, `🔎 Web Search: for "discord edit message"`
veya `✍️ Write: to /tmp/file` gibi gerçek çalışma güncellemeleri için görünür.

Kanal bunu güvenli biçimde yapabiliyorsa nihai yanıt taslağın yerini alır; aksi
takdirde OpenClaw nihai yanıtı normal teslimat yoluyla gönderir ve taslağı temizler
veya güncellemeyi durdurur (bkz. [Sonlandırma](#finalization)).

## Bir mod seçin

`channels.<channel>.streaming.mode`, görünür devam eden çalışma davranışını kontrol eder:

| Mod        | En uygun kullanım               | Sohbette görünenler                               |
| ---------- | -------------------------------- | ------------------------------------------------- |
| `off`      | Sessiz kanallar                  | Yalnızca nihai yanıt.                             |
| `partial`  | Yanıt metninin görünmesini izleme | En son yanıt metniyle düzenlenen tek bir taslak.  |
| `block`    | Daha büyük yanıt önizleme parçaları | Daha büyük parçalarla güncellenen veya eklenen tek bir önizleme. |
| `progress` | Araç ağırlıklı veya uzun süren turlar | Tek bir durum taslağı, ardından nihai yanıt.      |

Kullanıcılar yanıt metninin belirteç belirteç akışını izlemekten çok "neler olup
bittiğini" önemsiyorsa `progress`; ilerleme sinyali yanıt metninin kendisiyse
`partial`; daha büyük önizleme parçaları için `block` seçin.
Discord ve Telegram'da `streaming.mode: "block"` hâlâ önizleme akışıdır, normal
blok yanıt teslimatı değildir — bunun için `streaming.block.enabled` kullanın.

## Etiketleri yapılandırma

İlerleme etiketleri `channels.<channel>.streaming.progress` altında bulunur. Varsayılan
ham araç satırı etiketi `"auto"` olup yerleşik düz `Working`
etiketini kullanır. Durum başlığı bu örtük etiketi gizler; başlığın üzerinde de
bir etiket istiyorsanız `label: "auto"` değerini açıkça ayarlayın:

```text
Çalışıyor
```

Sabit bir etiket kullanın:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: "İnceleniyor",
        },
      },
    },
  },
}
```

Kendi etiket havuzunuzu kullanın (`label: "auto"` olduğunda seçim yine rastgele/tohuma göre yapılır):

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: "auto",
          labels: ["Kontrol ediliyor", "Okunuyor", "Test ediliyor", "Tamamlanıyor"],
        },
      },
    },
  },
}
```

Etiketi gizleyip yalnızca ilerleme satırlarını gösterin:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          label: false,
        },
      },
    },
  },
}
```

## İlerleme satırlarını kontrol etme

İlerleme satırları gerçek çalışma olaylarından gelir: araç başlangıçları, öğe
güncellemeleri, görev planları, onaylar, komut çıktıları, yama özetleri ve benzer
agent etkinlikleri. Varsayılan olarak etkindir (`progress.toolProgress`, varsayılan
`true`).

Araçlar, tek bir çağrı çalışmaya devam ederken türü belirlenmiş ilerleme bilgileri
de yayınlayabilir. Yavaş bir getirme veya arama işlemi, araç nihai sonucunu
döndürmeden önce görünür taslağı bu şekilde günceller. İlerleme güncellemesi,
boş model içeriği ve açık genel kanal meta verileri içeren kısmi bir araç
sonucudur:

```json
{
  "content": [],
  "progress": {
    "text": "Sayfa içeriği getiriliyor...",
    "visibility": "channel",
    "privacy": "public",
    "id": "web_fetch:fetching"
  }
}
```

OpenClaw, kanal ilerleme arayüzünde yalnızca `progress.text` öğesini işler.
Normal araç sonucu daha sonra `content`/`details` olarak gelir
ve modele döndürülen tek bölümdür.

Bir araca ilerleme eklerken kısa ve genel bir mesaj yayınlayın ve işlemin yararlı
olacak kadar uzun süre beklemede kalmasına kadar bunu geciktirin.
`web_fetch`, 5 saniyelik gecikmeyle tam olarak bunu yapar:

```typescript
const clearProgressTimer = scheduleToolProgress(
  onUpdate,
  { text: "Sayfa içeriği getiriliyor...", id: "web_fetch:fetching" },
  5_000,
  { signal },
);

try {
  return await runToolWork();
} finally {
  clearProgressTimer();
}
```

Hızlı çağrılarda ilerleme satırı gösterilmez; uzun çağrılarda işlem hâlâ
beklemedeyken bir satır gösterilir; iptal edilen çağrılar, eski ilerleme
bilgisinin görünmesini önlemek için zamanlayıcıyı temizler. İlerleme metni,
genel bir kullanıcı arayüzü yan kanalıdır; bu nedenle hiçbir zaman gizli
bilgiler, ham bağımsız değişkenler, getirilen içerik, komut çıktısı veya sayfa
metni içermemelidir.

### Ayrıntı modu

OpenClaw, ilerleme taslakları ve `/verbose` için aynı biçimlendiriciyi kullanır:

```json5
{
  agents: {
    defaults: {
      toolProgressDetail: "explain", // explain | raw
    },
  },
}
```

`"explain"` varsayılandır ve kısa etiketlerle taslakların kararlı
kalmasını sağlar. `"raw"`, kullanılabilir olduğunda temel komutu
ekler; bu, hata ayıklama sırasında yararlıdır ancak sohbeti daha gürültülü
hâle getirir. Örneğin bir `node --check /tmp/app.js` çağrısı, moda göre farklı işlenir:

| Mod       | İlerleme satırı                                                  |
| --------- | ---------------------------------------------------------------- |
| `explain` | `🛠️ check js syntax for /tmp/app.js`                                      |
| `raw` | `🛠️ check js syntax for /tmp/app.js · node --check /tmp/app.js`                                      |

### Komut/exec metni

`streaming.progress.commandText` (varsayılan `"raw"`), yukarıdaki ayrıntı modundan
bağımsız olarak exec/bash ilerleme satırlarının yanında ne kadar komut ayrıntısı
gösterileceğini kontrol eder. Komut metnini tamamen gizlerken araç ilerleme
satırını görünür tutmak için bunu `"status"` olarak ayarlayın:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          commandText: "status",
        },
      },
    },
  },
}
```

### Yorum akışı

`streaming.progress.commentary` (varsayılan `false`), modelin araç öncesi
yorumunu/önsöz anlatımını (💬, örneğin "Kontrol edeceğim... ardından
...") taslaktaki araç satırlarının arasına ekler. Kanallar arasında paylaşılan
yapılandırma biçimi için
[Akış ve parçalara ayırma](/tr/concepts/streaming#commentary-progress-lane) bölümüne bakın.

Yorum akışı etkinken önsözler yalnızca araya eklenen bu 💬 satırları olarak
işlenir; aşağıdaki durum başlığı görünmez ve böylece akış belgelenen biçimini korur.

### Durum başlığı

Discord ve Telegram'da ilerleme modundayken modelin türü belirlenmiş araç öncesi
önsözü, kullanılabilir olduğunda taslağın durum başlığına dönüşür. İlerleme
modundaki diğer kanallar mevcut durum davranışlarını korur. Başlık varsayılan
olarak açıktır ve kısa turlar için normal etkinlik eşiğini atlamaz;
`streaming.progress.commentary` etkinleştirildiğinde önsözler bunun yerine araya eklenen
yorum akışına aktarılır.

Discord'da agent için bir yardımcı model çözümlendiğinde — açık bir
[`utilityModel`](/tr/gateway/config-agents#utilitymodel) veya birincil
sağlayıcının bildirdiği varsayılan küçük model (OpenAI → `gpt-5.6-luna`,
Anthropic → `claude-haiku-4-5`) — model bir önsöz yayınlamadığında veya yaklaşık
20 saniye boyunca sessiz kaldığında kısa ve sade bir dolgu metni sağlar
(Telegram başlığı günümüzde yalnızca önsöz kullanır):

```text
Yapılandırmanızdaki varsayılan model güncelleniyor, ardından değişikliğin
uygulanması için Gateway yeniden başlatılıyor. Bir agent listeleme çağrısı
başarısız oldu ve yeniden deneniyor.
```

Yardımcı anlatım varsayılan olarak açıktır (`streaming.progress.narration`, varsayılan
`true`) ve hiçbir zaman birincil modele geri dönmez: yalnızca açık
bir `utilityModel` veya agent'ın birincil sağlayıcısı için sağlayıcı
tarafından bildirilmiş bir varsayılan olduğunda çalışır. Yardımcı yönlendirmeyi
tamamen devre dışı bırakmak için `utilityModel: ""` değerini ayarlayın. Araç
satırları altta birikmeye devam eder ve her iki durum kaynağı da durursa yeniden
görünür. Taslak düzenlemeleri yine normal etkinlik eşiğini ve gerçek bir metin
değişikliğini bekler; bu, hızlı turlarda kısa süreli görünümleri önler ve yoğun
kanallardaki düzenleme hareketliliğini azaltır. Yalnızca yardımcı model dolgu
metnini devre dışı bırakmak için `narration: false` değerini ayarlayın; model
önsözü başlıkları etkin kalır:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          narration: false,
        },
      },
    },
  },
}
```

Anlatım girdisi sınırlandırılmış ve hassas verilerden arındırılmıştır: yardımcı
model, gelen istek metninin yanı sıra taslağın işleyeceği aynı kısa ve
hassas verilerden arındırılmış araç özetlerini alır; ham komut çıktısını veya
araç sonuçlarını hiçbir zaman almaz. `commandText: "status"` ile anlatım girdisi,
taslağın gösterdiğiyle uyumlu olarak exec/bash komut metnini de içermez.

### Satır sınırları

Kaç satırın görünür kalacağını sınırlayın (varsayılan 8):

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLines: 4,
        },
      },
    },
  },
}
```

Taslak düzenlenirken sohbet balonunun yeniden akışını azaltmak için ilerleme
satırları otomatik olarak sıkıştırılır. Ayrıca OpenClaw, yinelenen taslak
düzenlemelerinin her güncellemede farklı şekilde kaymaması için uzun satırları
kısaltır. Varsayılan satır başına sınır 120 karakterdir; düz yazı bir sözcük
sınırında kesilirken yollar veya ham komutlar gibi uzun ayrıntılar, son bölümün
görünür kalması için ortada üç noktayla kısaltılır.

Satır başına sınırı ayarlayın:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          maxLineChars: 160,
        },
      },
    },
  },
}
```

### Zengin işleme (Slack)

Slack, ilerleme satırlarını düz metin yerine yapılandırılmış Block Kit alanları
olarak işleyebilir:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          render: "rich",
        },
      },
    },
  },
}
```

Zengin işleme, Block Kit alanlarının yanında her zaman aynı düz metin gövdesini
de gönderir; böylece daha zengin biçimi işleyemeyen istemciler yine kısa
ilerleme metnini gösterir.

### Araç/görev satırlarını gizleme

Tek ilerleme taslağını koruyup araç ve görev satırlarını gizleyin:

```json5
{
  channels: {
    discord: {
      streaming: {
        mode: "progress",
        progress: {
          toolProgress: false,
        },
      },
    },
  },
}
```

`toolProgress: false` ile OpenClaw, söz konusu tur için eski bağımsız
araç ilerleme mesajlarını yine de bastırır — bir etiket yapılandırılmışsa etiket
dışında kanal, nihai yanıt gelene kadar görsel olarak sessiz kalır.

## Kanal davranışı

| Kanal           | İlerleme aktarımı                       | Notlar                                                                                                                                                       |
| --------------- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Discord         | Bir mesaj gönderir, ardından düzenler.  | Varsayılan olarak `progress` modunu kullanır; nihai yanıt bir `-#` etkinlik makbuzu taşır ve yanıt ulaştıktan sonra durum taslağı silinir. |
| Matrix          | Bir etkinlik gönderir, ardından düzenler. | Hesap düzeyindeki akış yapılandırması, hesap düzeyindeki taslakları denetler.                                                                                 |
| Microsoft Teams | Kişisel sohbetlerde yerel Teams akışı.  | `streaming.mode: "block"`, bunun yerine Teams blok teslimine eşlenir.                                                                                                |
| Slack           | Yerel akış veya düzenlenebilir taslak gönderi. | Bir yanıt dizisi hedefi gerektirir; hedefi olmayan üst düzey doğrudan mesajlarda yine de taslak önizleme gönderileri ve düzenlemeleri kullanılır.              |
| Telegram        | Bir mesaj gönderir, ardından düzenler.  | İlerleme taslağı ile yanıt arasına bir mesaj gelirse taslak, istemcinin kaydırma konumunu sıçratmak yerine onun altına yeniden gönderilir (önce yeniyi gönder, sonra eskiyi sil). |
| Mattermost      | Düzenlenebilir taslak gönderi.          | `block` modu, tamamlanan metin ile araç etkinliği gönderileri arasında geçiş yapar; diğer modlar araç etkinliğini aynı taslak tarzı gönderide birleştirir. |

Güvenli düzenleme desteği olmayan kanallar, yazıyor göstergelerine veya
yalnızca nihai yanıt teslimine geri döner. Kanal başına eksiksiz çalışma zamanı
davranışı dökümü için [Akış ve parçalara ayırma](/tr/concepts/streaming) bölümüne bakın.

## Sonlandırma

Nihai yanıt hazır olduğunda OpenClaw sohbeti temiz tutmaya çalışır:

- Discord'da `progress` modunda nihai yanıt, sonuna küçük bir
  `-#` etkinlik makbuzu eklenmiş yeni bir mesaj olarak gönderilir
  (örneğin `-# 🧠 2 thoughts · 🛠️ 5 tool calls · ⏱️ 12s`) ve bu yanıt teslim
  edildikten sonra durum taslağı silinir. Yoğun kanallarda yanıtın üzerinde
  sahipsiz bir araç günlüğü kalmaz; hatalı nihai yanıtlarda taslak, başarısız
  turun görünür kaydı olarak tutulur.
- Taslak güvenli biçimde nihai yanıta dönüştürülebiliyorsa (`partial`/`block` modları),
  OpenClaw taslağı yerinde düzenler.
- Kanal yerel ilerleme akışı kullanıyorsa yerel aktarım nihai metni
  kabul ettiğinde OpenClaw bu akışı sonlandırır.
- Aksi takdirde (medya, onay istemi, açık bir yanıt hedefi, çok fazla
  parça ya da başarısız bir düzenleme/gönderme) OpenClaw, taslağın üzerine
  yazmak yerine nihai yanıtı normal kanal teslim yolu üzerinden gönderir.

Bu geri dönüş kasıtlıdır: yeni bir nihai yanıt göndermek; metni kaybetmekten,
yanıtı yanlış diziye göndermekten veya kanalın güvenli biçimde temsil edemediği
bir yükle taslağın üzerine yazmaktan daha iyidir.

## Sorun giderme

**Yalnızca nihai yanıtı görüyorum.**

Mesajı işleyen hesap veya kanal için `channels.<channel>.streaming.mode` değerinin
`progress` olduğunu denetleyin. Kanal doğru mesajı güvenli biçimde
düzenleyemediğinde bazı grup veya alıntılı yanıt yolları, ilgili tur için
taslak önizlemelerini devre dışı bırakır.

**Etiketi görüyorum ancak araç satırlarını görmüyorum.**

`streaming.progress.toolProgress` değerini denetleyin. Değer `false` ise OpenClaw
tek taslak davranışını korur ancak araç ve görev ilerleme satırlarını gizler.

**Düzenlenmiş bir taslak yerine yeni bir nihai mesaj görüyorum.**

Bu, [Sonlandırma](#finalization) bölümünde açıklanan güvenlik amaçlı geri
dönüştür. Medya yanıtlarında, uzun yanıtlarda, açık yanıt hedeflerinde, eski
Telegram taslaklarında, eksik Slack ileti dizisi hedeflerinde, silinmiş önizleme
mesajlarında veya yerel akışın sonlandırılamadığı durumlarda gerçekleşebilir.

**Hâlâ bağımsız ilerleme mesajları görüyorum.**

İlerleme modu, bir taslak etkin olduğunda varsayılan bağımsız araç ilerleme
mesajlarını bastırır. Bağımsız mesajlar hâlâ görünüyorsa turun gerçekten
`progress` modunu kullandığını ve `streaming.mode: "off"` modunu ya da
söz konusu mesaj için taslak oluşturamayan bir kanal yolunu kullanmadığını
doğrulayın.

**Teams, Discord veya Telegram'dan farklı davranıyor.**

Microsoft Teams, genel gönder-ve-düzenle önizleme aktarımı yerine kişisel
sohbetlerde yerel bir akış kullanır ve Discord ile Telegram'daki gibi bir
taslak önizleme blok modu bulunmadığından `streaming.mode: "block"` değerini Teams
blok teslimine eşler.

## İlgili

- [Akış ve parçalara ayırma](/tr/concepts/streaming)
- [Mesajlar](/tr/concepts/messages)
- [Kanal yapılandırması](/tr/gateway/config-channels)
- [Discord](/tr/channels/discord)
- [Matrix](/tr/channels/matrix)
- [Microsoft Teams](/tr/channels/msteams)
- [Slack](/tr/channels/slack)
- [Telegram](/tr/channels/telegram)
- [Mattermost](/tr/channels/mattermost)
