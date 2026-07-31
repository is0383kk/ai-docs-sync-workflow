---
read_when:
    - Medya işlem hattını veya ekleri değiştirme
summary: Gönderim, gateway ve agent yanıtları için görüntü ve medya işleme kuralları
title: Görüntü ve medya desteği
x-i18n:
    generated_at: "2026-07-26T23:24:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71f5591f4268593c142056370802b702899787a79f9ca1fbde6ea8e422f34023
    source_path: nodes/images.md
    workflow: 16
---

WhatsApp kanalı Baileys Web üzerinde çalışır. Bu sayfa; gönderim, Gateway ve ajan yanıtları için medya işleme kurallarını kapsar.

## Hedefler

- Medya dosyalarını isteğe bağlı bir açıklamayla `openclaw message send --media` üzerinden gönderme.
- Web gelen kutusundan gönderilen otomatik yanıtların metnin yanında medya da içermesine izin verme.
- Türe göre sınırları makul ve öngörülebilir tutma.

## CLI Yüzeyi

`openclaw message send --target <dest> --media <path-or-url> [--message <caption>]`

- `--media <path-or-url>` — medya (görüntü/ses/video/belge) ekler; yerel yolları veya URL'leri kabul eder. İsteğe bağlıdır; yalnızca medya içeren gönderimlerde açıklama boş olabilir.
- `--gif-playback` — video medyasını GIF oynatımı olarak işler (yalnızca WhatsApp).
- `--force-document` — kanal sıkıştırmasını önlemek için medyayı belge olarak gönderir (Telegram, WhatsApp); görüntüler, GIF'ler ve videolar için geçerlidir.
- `--reply-to <id>`, `--thread-id <id>`, `--pin`, `--silent` — yalnızca metin içeren gönderimlerle paylaşılan teslim ve iş parçacığı seçenekleri.
- `--dry-run` — çözümlenen yükü yazdırır ve gönderimi atlar.
- `--json` — sonucu JSON olarak yazdırır: `{ action, channel, dryRun, handledBy, messageId?, payload }` (`payload`, tüm medya referansları dâhil olmak üzere kanala özgü gönderim sonucunu taşır).

## WhatsApp Web kanalı davranışı

- Girdi: yerel dosya yolu **veya** HTTP(S) URL'si.
- Akış: tampon belleğe yükler, medya türünü algılar, ardından türe göre giden yükü oluşturur:
  - **Görüntüler:** `channels.whatsapp.mediaMaxMb` (varsayılan 50MB) altında kalacak şekilde optimize edilir. Saydam olmayan görüntüler yeniden JPEG olarak sıkıştırılır (varsayılan kenar basamakları 2048px'ten başlar ve boyut sınırı tekrar tekrar aşıldığında azalır); saydamlık içeren görüntüler PNG olarak tutulur. Kaynak, boyut ve kenar uzunluğu bütçesi içinde zaten kabul edilebilir bir JPEG/PNG/WebP ise yeniden sıkıştırılmak yerine özgün baytlar değiştirilmeden korunur. Animasyonlu GIF'ler hiçbir zaman yeniden kodlanmaz, yalnızca boyut denetiminden geçirilir.
  - **Ses/sesli mesaj:** Zaten yerel sesli mesaj biçiminde olmadığı sürece (`.ogg`/`.opus` veya `audio/ogg`/`audio/opus`), giden ses, sesli mesaj (`ptt: true`) olarak gönderilmeden önce `ffmpeg` aracılığıyla Opus/OGG biçimine dönüştürülür (48kHz mono, 64kbps, en fazla 20 dakika).
  - **Video:** 16MB'a kadar doğrudan geçirilir.
  - **Belgeler:** Diğer her şey, varsa dosya adı korunarak 100MB'a kadar gönderilir.
- WhatsApp GIF tarzı oynatım: mobil istemcilerin satır içinde döngüsel oynatması için `gifPlayback: true` (CLI: `--gif-playback`) ile bir MP4 gönderilir.
- MIME algılama öncelikle incelenen sihirli baytları, ardından dosya uzantısını ve son olarak yanıt üst bilgilerini kullanır; incelenen genel bir kapsayıcı (`application/octet-stream`, `zip`) daha özel bir uzantı eşlemesini (örneğin XLSX ile ZIP) hiçbir zaman geçersiz kılmaz.
- Açıklama `--message` veya `reply.text` değerinden gelir; boş açıklamaya izin verilir.
- Günlük kaydı: ayrıntısız modda `↩️`/`✅` gösterilir; ayrıntılı mod boyutu ve kaynak yolunu/URL'sini içerir.

<Note>
Yukarıdaki 16MB ses/video ve 100MB belge değerleri, açık bir bayt üst sınırı geçirilmediğinde kullanılan, türe göre paylaşılan varsayılan medya sınırlarıdır. WhatsApp gönderimleri, `channels.whatsapp.mediaMaxMb` üzerinden açık bir üst sınır (varsayılan 50MB) belirler ve bu sınır söz konusu hesap için tüm türlere aynı şekilde uygulanır.
</Note>

## Otomatik Yanıt İşlem Hattı

- `getReplyFromConfig`, diğer alanların yanı sıra `text?`, `mediaUrl?` ve `mediaUrls?` içeren bir yanıt yükü (veya yük dizisi) döndürür.
- Medya mevcut olduğunda web göndericisi, yerel yolları veya URL'leri `openclaw message send` ile aynı işlem hattını kullanarak çözümler.
- Birden fazla medya girdisi sağlanırsa bunlar sırayla gönderilir.

## Komutlara Gelen Medya

- Gelen web mesajları medya içerdiğinde OpenClaw, medyayı geçici bir dosyaya indirir ve şablon değişkenlerini kullanıma sunar:
  - `{{AttachmentUrl}}` — geçerli ekin özgün URL'si veya sağlayıcı referansı.
  - `{{AttachmentPath}}` — komut çalıştırılmadan önce yazılan yerel geçici yol.
  - `{{AttachmentContentType}}` — MIME içerik türü.
  - `{{AttachmentDir}}` — yerel yolu içeren dizin.
  - `{{AttachmentIndex}}` — sıfır tabanlı kaynak olgu dizini.
- Oturum başına Docker korumalı alanı etkinleştirildiğinde gelen medya korumalı alan çalışma alanına kopyalanır ve ek yolu/referansı `media/inbound/<filename>` gibi korumalı alana göreli bir yol olarak yeniden yazılır.
- `{{MediaPath}}`, `{{MediaUrl}}`, `{{MediaType}}` ve `{{MediaDir}}`, plugin SDK geçiş dönemi boyunca kullanımdan kaldırılmış uyumluluk takma adları olarak kalır.
- Medya anlama (`tools.media.*` veya paylaşılan `tools.media.models` üzerinden yapılandırılır), şablonlamadan önce çalışır ve `Body` içine `[Image]`, `[Audio]` ve `[Video]` blokları ekleyebilir.
  - Ses, `{{Transcript}}` değerini ayarlar ve eğik çizgi komutlarının çalışmaya devam etmesi için komut ayrıştırmada transkripti kullanır.
  - Video ve görüntü açıklamaları, komut ayrıştırma için tüm açıklama metinlerini korur.
  - Etkin birincil model görsel algılamayı zaten yerel olarak destekliyorsa OpenClaw, `[Image]` özet bloğunu atlar ve bunun yerine özgün görüntüyü modele iletir.
- Varsayılan olarak yalnızca eşleşen ilk görüntü/ses/video eki işlenir; birden fazla ek seçmek için `tools.media.<capability>.attachments` kullanılır.

## Sınırlar ve hatalar

**Giden gönderim üst sınırları (WhatsApp web gönderimi)**

- Görüntüler: optimizasyondan sonra `channels.whatsapp.mediaMaxMb` değerine kadar (varsayılan 50MB).
- Ses/video: 16MB üst sınırı (paylaşılan varsayılan; WhatsApp üzerinden gönderilirken `mediaMaxMb` tarafından geçersiz kılınır).
- Belgeler: 100MB üst sınırı (paylaşılan varsayılan; WhatsApp üzerinden gönderilirken `mediaMaxMb` tarafından geçersiz kılınır).
- Boyut sınırını aşan veya okunamayan medya, günlüklerde açık bir hata oluşturur ve yanıt atlanır.

**Medya anlama üst sınırları (transkripsiyon/açıklama)**

- Görüntü varsayılanı: 10MB (`tools.media.image.maxBytes` ile veya her
  `tools.media.models[]` girdisi için `maxBytes` ile geçersiz kılınır).
- Ses varsayılanı: 20MB (`tools.media.audio.maxBytes` ile veya her girdi için geçersiz kılınır).
- Video varsayılanı: 50MB (`tools.media.video.maxBytes` ile veya her girdi için geçersiz kılınır).
- Boyut sınırını aşan medyada anlama işlemi atlanır ancak yanıt özgün gövdeyle yine de iletilir.

## Test Notları

- Görüntü/ses/belge durumları için gönderim ve yanıt akışlarını kapsayın.
- Görüntü optimizasyonundan sonra boyut sınırlarını ve ses için sesli mesaj bayrağını doğrulayın.
- Birden fazla medya içeren yanıtların ardışık gönderimler hâlinde dağıtıldığından emin olun.

## İlgili

- [Kamera çekimi](/tr/nodes/camera)
- [Medya anlama](/tr/nodes/media-understanding)
- [Ses ve sesli mesajlar](/tr/nodes/audio)
