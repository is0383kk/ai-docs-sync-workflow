---
read_when:
    - /new, /reset, /stop ve ajan yaşam döngüsü olayları için olay odaklı otomasyon istiyorsunuz
    - Hook'ları oluşturmak, yüklemek veya hata ayıklamak istiyorsunuz
summary: 'Hook''lar: komutlar ve yaşam döngüsü olayları için olay güdümlü otomasyon'
title: Kancalar
x-i18n:
    generated_at: "2026-07-26T23:09:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 039a55cca60e0005d7b9c4d950a86aceb6e7c29d5768108b34011bfc21c85be6
    source_path: automation/hooks.md
    workflow: 16
---

Hook'lar, agent olayları tetiklendiğinde Gateway içinde çalışan küçük betiklerdir: `/new`, `/reset`, `/stop` gibi komutlar, oturum Compaction'ı, Gateway yaşam döngüsü ve mesaj akışı. Dizinlerden keşfedilir ve `openclaw hooks` ile yönetilirler. Gateway, dahili Hook'ları yalnızca Hook'ları etkinleştirdikten veya en az bir Hook girdisi, Hook paketi, eski işleyici ya da ek Hook dizini yapılandırdıktan sonra yükler.

OpenClaw'da iki tür Hook vardır:

- **Dahili Hook'lar** (bu sayfa): agent olayları tetiklendiğinde Gateway içinde çalışır.
- **Webhook'lar**: diğer sistemlerin OpenClaw'da iş tetiklemesini sağlayan harici HTTP uç noktalarıdır. Bkz. [Webhook'lar](/tr/automation/cron-jobs#webhooks).

Hook'lar plugin'lerin içinde de paketlenebilir. `openclaw hooks list`, hem bağımsız Hook'ları hem de plugin tarafından yönetilen Hook'ları (`plugin:<id>` olarak gösterilir) görüntüler.

## Doğru yüzeyi seçme

OpenClaw, birbirine benzeyen ancak farklı sorunları çözen çeşitli genişletme yüzeylerine sahiptir:

| Şunu yapmak istiyorsanız...                                                                                                     | Şunu kullanın...                                | Nedeni                                                                                           |
| --------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------- |
| `/new` sırasında anlık görüntü kaydetmek, `/reset` olayını günlüğe kaydetmek, `message:sent` sonrasında harici API çağırmak veya genel operatör otomasyonu eklemek | Dahili Hook'lar (`HOOK.md`, bu sayfa) | Dosya tabanlı Hook'lar, operatör tarafından yönetilen yan etkiler ve komut/yaşam döngüsü otomasyonu için tasarlanmıştır |
| İstemleri yeniden yazmak, araçları engellemek, giden mesajları iptal etmek veya sıralı ara katman/politika eklemek                              | `api.on(...)` aracılığıyla türü belirlenmiş plugin Hook'ları  | Türü belirlenmiş Hook'ların açık sözleşmeleri, öncelikleri, birleştirme kuralları ve engelleme/iptal semantiği vardır      |
| Yalnızca telemetri dışa aktarımı veya gözlemlenebilirlik eklemek                                                                            | Tanılama olayları                     | Gözlemlenebilirlik ayrı bir olay veri yoludur, politika Hook'u yüzeyi değildir                              |

Küçük ve kurulu bir entegrasyon gibi davranan otomasyon istediğinizde dahili Hook'ları kullanın. Çalışma zamanı yaşam döngüsü denetimine ihtiyaç duyduğunuzda türü belirlenmiş plugin Hook'larını kullanın.

## Hızlı başlangıç

```bash
# Kullanılabilir Hook'ları listele
openclaw hooks list

# Bir Hook'u etkinleştir
openclaw hooks enable session-memory

# Hook durumunu denetle
openclaw hooks check

# Ayrıntılı bilgi al
openclaw hooks info session-memory
```

## Olay türleri

Hook'lar bu tablodaki belirli bir anahtara veya o ailedeki her eylemi almak için
yalnızca aile adına (`command`, `session`, `agent`, `gateway`, `message`)
abone olur. OpenClaw çekirdeği başka hiçbir şey yayınlamaz; bu nedenle diğer
adlar neredeyse her zaman Hook'un sessizce etkisiz kalmasına neden olan yazım
hatalarıdır (yalnızca özel olay yayınlayan bir plugin bunu tetikleyebilir).
Hook yükleyici bu tür adlar için bir uyarıyı günlüğe kaydeder (örneğin
`command:nwe`) ve `openclaw hooks info <name>` bunları işaretler; dolayısıyla hiç
çalışmayan bir Hook'un nedeni belirlenebilir.

| Olay                    | Tetiklendiği zaman                                              |
| ------------------------ | ---------------------------------------------------------- |
| `command:new`            | `/new` komutu verildiğinde                                      |
| `command:reset`          | `/reset` komutu verildiğinde                                    |
| `command:stop`           | `/stop` komutu verildiğinde                                     |
| `command`                | Herhangi bir komut olayı (genel dinleyici)                       |
| `session:compact:before` | Compaction geçmişi özetlemeden önce                       |
| `session:compact:after`  | Compaction tamamlandıktan sonra                                 |
| `session:patch`          | Oturum özellikleri değiştirildiğinde                       |
| `agent:bootstrap`        | Çalışma alanı önyükleme dosyaları eklenmeden önce              |
| `gateway:startup`        | Kanallar başlatılıp Hook'lar yüklendikten sonra                  |
| `gateway:shutdown`       | Gateway kapatma işlemi başladığında                               |
| `gateway:pre-restart`    | Beklenen bir Gateway yeniden başlatmasından önce                         |
| `message:received`       | Herhangi bir kanaldan gelen mesaj                           |
| `message:transcribed`    | Ses dökümü tamamlandıktan sonra                        |
| `message:preprocessed`   | Medya ve bağlantı ön işlemesi tamamlandıktan veya atlandıktan sonra |
| `message:sent`           | Giden gönderim denendiğinde (`context.success` sonucu içerir) |

## Hook yazma

### Hook yapısı

Her Hook, iki dosya içeren bir dizindir:

```text
my-hook/
├── HOOK.md          # Meta veriler + belgeler
└── handler.ts       # İşleyici uygulaması
```

İşleyici dosyası `handler.ts`, `handler.js`, `index.ts` veya `index.js` olabilir.

### HOOK.md biçimi

```markdown
---
name: my-hook
description: "Bu Hook'un ne yaptığının kısa açıklaması"
metadata:
  { "openclaw": { "emoji": "🔗", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---

# Hook'um

Ayrıntılı belgeler buraya gelir.
```

**Meta veri alanları** (`metadata.openclaw`):

| Alan      | Açıklama                                          |
| ---------- | ---------------------------------------------------- |
| `emoji`    | CLI için görüntüleme emojisi                                |
| `events`   | Dinlenecek olayların dizisi                        |
| `export`   | Kullanılacak adlandırılmış dışa aktarım (varsayılan: `"default"`)        |
| `os`       | Gerekli platformlar (ör. `["darwin", "linux"]`)     |
| `requires` | Gerekli `bins`, `anyBins`, `env` veya `config` yolları |
| `always`   | Uygunluk denetimlerini atla (boolean)                  |
| `hookKey`  | Yapılandırma anahtarı geçersiz kılma değeri (varsayılan: Hook adı)      |
| `homepage` | `openclaw hooks info` tarafından gösterilen belge URL'si              |
| `install`  | Kurulum yöntemleri                                 |

### İşleyici uygulaması

```typescript
const handler = async (event) => {
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  console.log(`[my-hook] Yeni komut tetiklendi`);
  // Mantığınız buraya gelir

  // İsteğe bağlı olarak yanıtlanabilir yüzeylerde yanıt gönder
  event.messages.push("Hook çalıştırıldı!");
};

export default handler;
```

Her olay şunları içerir: `type`, `action`, `sessionKey`, `timestamp`, `messages` ve `context` (olaya özgü veriler). Agent ve araç Hook'ları için türü belirlenmiş plugin Hook bağlamları, plugin'lerin OTEL korelasyonu amacıyla yapılandırılmış günlüklere aktarabileceği, salt okunur ve W3C uyumlu bir tanılama izleme bağlamı olan `trace` öğesini de içerebilir.

`event.messages` öğesine eklenen dizeler, yalnızca
`command:new` ve `command:reset` için (kaynak konuşmaya yanıt olarak
yönlendirilir) ve `session:compact:before` / `session:compact:after` için
(Compaction durum bildirimleri olarak gönderilir) sohbete geri iletilir.
`command:stop`, `message:*`, `agent:bootstrap`, `session:patch` ve
`gateway:*` dâhil diğer tüm olaylar, eklenen mesajları yok sayar.

### Olay bağlamının önemli noktaları

**Komut olayları** (`command:new`, `command:reset`): `context.sessionEntry`, `context.previousSessionEntry`, `context.commandSource`, `context.senderId`, `context.workspaceDir`, `context.cfg`.

**Komut olayları** (`command:stop`): `context.sessionEntry`, `context.sessionId`, `context.commandSource`, `context.senderId`.

**Mesaj olayları** (`message:received`): `context.from`, `context.content`, `context.channelId`, `context.media` (sıralı, aşamalandırılmış ek bilgileri), uzak medya henüz yerel olarak aşamalandırılmadığında `context.originalMedia` ile birlikte `context.mediaStagingPending` ve `context.metadata` (`senderId`, `senderName`, `guildId` dâhil sağlayıcıya özgü veriler). `context.content`, komut benzeri mesajlarda boş olmayan bir komut gövdesini tercih eder; ardından ham gelen gövdeye ve genel gövdeye geri döner. İleti dizisi geçmişi veya bağlantı özetleri gibi yalnızca agent'a yönelik zenginleştirmeleri içermez. `metadata` içindeki eski medya diğer adları kullanımdan kaldırılmıştır.

**Mesaj olayları** (`message:sent`): `context.to`, `context.content`, `context.success`, `context.channelId` ve gönderim başarısız olduğunda `context.error`.

**Mesaj olayları** (`message:transcribed`): `context.transcript`, `context.from`, `context.channelId` ve `context.media`. `context.mediaPath` ve `context.mediaType`, ilk bilgi için kullanımdan kaldırılmış diğer adlar olarak kalır.

**Mesaj olayları** (`message:preprocessed`): `context.bodyForAgent` (nihai zenginleştirilmiş gövde), `context.from`, `context.channelId`.

**Önyükleme olayları** (`agent:bootstrap`): `context.bootstrapFiles` (değiştirilebilir dizi), `context.agentId`.

**Oturum yama olayları** (`session:patch`): `context.sessionEntry`, `context.patch` (yalnızca değiştirilen alanlar), `context.cfg`. Yama olaylarını yalnızca ayrıcalıklı istemciler tetikleyebilir; bağlam bir kopyadır, bu nedenle işleyiciler canlı oturum girdisini değiştiremez.

**Compaction olayları**: `session:compact:before`, `messageCount` ve `tokenCount` öğelerini içerir. `session:compact:after`; `compactedCount`, `summaryLength`, `tokensBefore` ve `tokensAfter` öğelerini ekler.

`command:stop`, kullanıcının `/stop` komutunu vermesini gözlemler; bu,
bir agent sonlandırma geçidi değil, iptal/komut yaşam döngüsüdür. Doğal bir
nihai yanıtı incelemesi ve agent'dan bir geçiş daha istemesi gereken plugin'ler
bunun yerine türü belirlenmiş `before_agent_finalize` plugin Hook'unu kullanmalıdır.
Bkz. [Plugin Hook'ları](/tr/plugins/hooks).

**Gateway yaşam döngüsü olayları**: `gateway:shutdown`, `reason` ve `restartExpectedMs` öğelerini içerir ve Gateway kapatma işlemi başladığında tetiklenir. `gateway:pre-restart` aynı bağlamı içerir, ancak yalnızca kapatma beklenen bir yeniden başlatmanın parçasıysa ve sonlu bir `restartExpectedMs` değeri sağlanmışsa tetiklenir. Kapatma sırasında her yaşam döngüsü Hook'u için bekleme, en iyi çaba esasına göre ve sınırlı olarak gerçekleştirilir; böylece bir işleyici takılırsa kapatma devam eder. Varsayılan bekleme bütçesi `gateway:shutdown` için 5 saniye, `gateway:pre-restart` için 10 saniyedir.

Kanallar hâlâ kullanılabilir durumdayken kısa yeniden başlatma bildirimleri için `gateway:pre-restart` kullanın:

```typescript
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);

export default async function handler(event) {
  if (event.type !== "gateway" || event.action !== "pre-restart") {
    return;
  }

  const restartInSeconds = Math.ceil(event.context.restartExpectedMs / 1000);
  await execFileAsync("openclaw", [
    "system",
    "event",
    "--mode",
    "now",
    "--text",
    `Gateway yaklaşık ${restartInSeconds} sn. içinde yeniden başlatılıyor (${event.context.reason}). Şimdi denetim noktası oluşturun.`,
  ]);
}
```

`gateway:shutdown` (veya `gateway:pre-restart`) olayı ile kapatma dizisinin geri kalanı arasında Gateway, süreç durduğunda hâlâ etkin olan her oturum için türü belirlenmiş bir `session_end` plugin Hook'u da tetikler. Olayın `reason` değeri, normal bir SIGTERM/SIGINT durdurması için `shutdown`; kapatma beklenen bir yeniden başlatmanın parçası olarak planlandığında ise `restart` olur. Bu boşaltma işlemi sınırlıdır; böylece yavaş bir `session_end` işleyicisi sürecin çıkışını engelleyemez. Çift tetiklemeyi önlemek için replace / reset / delete / Compaction aracılığıyla zaten sonlandırılmış oturumlar atlanır.

## Hook keşfi

Hook'lar dört kaynaktan keşfedilir:

1. **Paketlenmiş hook'lar**: OpenClaw ile birlikte sunulur
2. **Plugin hook'ları**: yüklü plugin'lerin içinde paketlenmiştir; aynı ada sahip paketlenmiş hook'ları geçersiz kılabilir
3. **Yönetilen hook'lar**: `~/.openclaw/hooks/` (kullanıcı tarafından yüklenir, çalışma alanları arasında paylaşılır); paketlenmiş hook'ları ve plugin hook'larını geçersiz kılabilir. `hooks.internal.load.extraDirs` içindeki ek dizinler de bu önceliğe sahiptir.
4. **Çalışma alanı hook'ları**: `<workspace>/hooks/` (ajan başına, açıkça etkinleştirilene kadar varsayılan olarak devre dışıdır)

Çalışma alanı hook'ları yeni hook adları ekleyebilir ancak aynı ada sahip paketlenmiş, yönetilen veya plugin tarafından sağlanan hook'ları geçersiz kılamaz.

Gateway, dahili hook'lar yapılandırılana kadar başlangıçta dahili hook keşfini atlar. `openclaw hooks enable <name>` ile paketlenmiş veya yönetilen bir hook'u etkinleştirin, bir hook paketi yükleyin ya da katılmak için `hooks.internal.enabled=true` ayarını yapın. Adlandırılmış bir hook'u etkinleştirdiğinizde Gateway yalnızca o hook'un işleyicisini yükler; `hooks.internal.enabled=true`, ek hook dizinleri ve eski işleyiciler geniş kapsamlı keşfi etkinleştirir.

### Hook paketleri

Hook paketleri, `package.json` içindeki `openclaw.hooks` aracılığıyla hook'ları dışa aktaran npm paketleridir. Şununla yükleyin:

```bash
openclaw plugins install <path-or-spec>
```

Npm belirtimleri yalnızca kayıt defterinden olabilir (paket adı + isteğe bağlı tam sürüm veya dist-tag). Git/URL/dosya belirtimleri ve semver aralıkları reddedilir. Eski `openclaw hooks install` ve `openclaw hooks update` komutları, `openclaw plugins install` / `openclaw plugins update` için kullanımdan kaldırılmış takma adlardır.

## Paketlenmiş hook'lar

| Hook                  | Olaylar                                            | Yaptığı işlem                                                   |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------- |
| session-memory        | `command:new`, `command:reset`                    | Oturum bağlamını `<workspace>/memory/` konumuna kaydeder                 |
| bootstrap-extra-files | `agent:bootstrap`                                 | Glob kalıplarından ek başlangıç dosyaları ekler          |
| command-logger        | `command`                                         | Tüm komutları `~/.openclaw/logs/commands.log` konumuna kaydeder           |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | Oturum Compaction işlemi başladığında/sona erdiğinde görünür sohbet bildirimleri gönderir |
| boot-md               | `gateway:startup`                                 | Gateway başladığında `BOOT.md` çalıştırır                         |

Paketlenmiş herhangi bir hook'u etkinleştirin:

```bash
openclaw hooks enable <hook-name>
```

<a id="session-memory"></a>

### session-memory ayrıntıları

Son kullanıcı/asistan mesajlarını çıkarır (varsayılan 15, `hooks.internal.entries.session-memory.messages` ile yapılandırılabilir) ve ana makinenin yerel tarihini kullanarak `<workspace>/memory/YYYY-MM-DD-HHMM.md` konumuna kaydeder. Bellek yakalama arka planda çalıştığından `/new` ve `/reset` onayları, transkript okumaları veya isteğe bağlı kısa ad oluşturma nedeniyle gecikmez. Açıklayıcı dosya adı kısa adları oluşturmak için `hooks.internal.entries.session-memory.llmSlug: true` ayarını yapın ve isteğe bağlı olarak `hooks.internal.entries.session-memory.model` değerini `sonnet` gibi yapılandırılmış bir takma ada, ajanın varsayılan sağlayıcısındaki yalın bir model kimliğine veya bir `provider/model` referansına ayarlayın. `model` belirtilmediğinde kısa ad oluşturma, ajanın varsayılan modelini kullanır ve kullanılamadığında zaman damgası kısa adlarına geri döner. `workspace.dir` yapılandırmasının yapılmasını gerektirir.

<a id="bootstrap-extra-files"></a>

### bootstrap-extra-files yapılandırması

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "bootstrap-extra-files": {
          "enabled": true,
          "paths": ["packages/*/AGENTS.md", "packages/*/TOOLS.md"]
        }
      }
    }
  }
}
```

`patterns` ve `files`, `paths` için takma ad olarak kabul edilir. Yollar çalışma alanına göre çözümlenir ve çalışma alanının içinde kalmalıdır. Yalnızca tanınan başlangıç temel adları yüklenir (`AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md`, `MEMORY.md`).

<a id="command-logger"></a>

### command-logger ayrıntıları

Her eğik çizgi komutunu bir JSON satırı (zaman damgası, eylem, oturum anahtarı, gönderen kimliği, kaynak) olarak `~/.openclaw/logs/commands.log` konumuna kaydeder.

<a id="compaction-notifier"></a>

### compaction-notifier ayrıntıları

OpenClaw oturum transkriptini sıkıştırmaya başladığında ve sıkıştırmayı bitirdiğinde mevcut görüşmeye kısa durum mesajları gönderir. Böylece kullanıcı, asistanın bağlamı özetlediğini ve Compaction sonrasında devam edeceğini görebildiği için sohbet yüzeylerindeki uzun dönüşler daha az kafa karıştırıcı olur.

<a id="boot-md"></a>

### boot-md ayrıntıları

Dosya, ajanın çözümlenmiş çalışma alanında mevcutsa yapılandırılmış her ajan kapsamı için Gateway başlangıcında `BOOT.md` çalıştırır.

## Plugin hook'ları

Plugin'ler, daha derin entegrasyon için Plugin SDK üzerinden türü belirlenmiş hook'lar kaydedebilir:
araç çağrılarına müdahale etme, istemleri değiştirme, mesaj akışını denetleme ve daha fazlası.
`before_tool_call`, `before_agent_reply`,
`before_install` veya diğer işlem içi yaşam döngüsü hook'larına ihtiyaç duyduğunuzda plugin hook'larını kullanın.

Plugin tarafından yönetilen dahili hook'lar farklıdır: bu sayfadaki
genel komut/yaşam döngüsü olay sistemine katılır ve `openclaw hooks list` içinde
`plugin:<id>` olarak görünür. Bunları sıralı ara yazılım veya ilke geçitleri için değil,
yan etkiler ve hook paketleriyle uyumluluk için kullanın.

Eksiksiz plugin hook'u referansı için [Plugin hook'ları](/tr/plugins/hooks) bölümüne bakın.

## Yapılandırma

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": { "enabled": true },
        "command-logger": { "enabled": false }
      }
    }
  }
}
```

Hook başına ortam değerleri, hook'un `requires.env` uygunluk kontrollerini (işlem ortamıyla birlikte) karşılar ve işleyiciler bunları hook yapılandırma girdilerinden okuyabilir:

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "my-hook": {
          "enabled": true,
          "env": { "MY_CUSTOM_VAR": "value" }
        }
      }
    }
  }
}
```

Ek hook dizinleri:

```json
{
  "hooks": {
    "internal": {
      "load": {
        "extraDirs": ["/path/to/more/hooks"]
      }
    }
  }
}
```

<Note>
Eski `hooks.internal.handlers` dizi yapılandırma biçimi geriye dönük uyumluluk için hâlâ desteklenmektedir, ancak yeni hook'lar keşif tabanlı sistemi kullanmalıdır.
</Note>

## CLI referansı

```bash
# Tüm hook'ları listele (--eligible, --verbose veya --json ekleyin)
openclaw hooks list

# Bir hook hakkında ayrıntılı bilgi göster
openclaw hooks info <hook-name>

# Uygunluk özetini göster
openclaw hooks check

# Etkinleştir/devre dışı bırak
openclaw hooks enable <hook-name>
openclaw hooks disable <hook-name>
```

## En iyi uygulamalar

- **İşleyicileri hızlı tutun.** Hook'lar komut işleme sırasında çalışır. Ağır işleri `void processInBackground(event)` ile başlatıp beklemeden devam edin.
- **Hataları düzgün biçimde işleyin.** Riskli işlemleri try/catch içine alın; diğer işleyicilerin çalışabilmesi için hata fırlatmayın.
- **Olayları erkenden filtreleyin.** Olay türü/eylem ilgili değilse hemen dönün.
- **Belirli olay anahtarları kullanın.** Ek yükü azaltmak için `"events": ["command"]` yerine `"events": ["command:new"]` tercih edin.

## Sorun giderme

### Hook keşfedilmiyor

```bash
# Dizin yapısını doğrula
ls -la ~/.openclaw/hooks/my-hook/
# Şunları göstermelidir: HOOK.md, handler.ts

# Keşfedilen tüm hook'ları listele
openclaw hooks list
```

### Hook uygun değil

```bash
openclaw hooks info my-hook
```

Eksik ikili dosyaları (PATH), ortam değişkenlerini, yapılandırma değerlerini veya işletim sistemi uyumluluğunu kontrol edin.

### Hook çalışmıyor

1. Hook'un etkinleştirildiğini doğrulayın: `openclaw hooks list`
2. Hook'ların yeniden yüklenmesi için Gateway işleminizi yeniden başlatın.
3. Gateway günlüklerini kontrol edin: `openclaw logs --follow | grep -i hook`

## İlgili

- [CLI Referansı: hook'lar](/tr/cli/hooks)
- [Webhook'lar](/tr/automation/cron-jobs#webhooks)
- [Plugin hook'ları](/tr/plugins/hooks) — işlem içi plugin yaşam döngüsü hook'ları
- [Yapılandırma](/tr/gateway/configuration-reference#hooks)
