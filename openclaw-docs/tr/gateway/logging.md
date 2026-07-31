---
read_when:
    - Günlük kaydı çıktısını veya biçimlerini değiştirme
    - CLI veya Gateway çıktısında hata ayıklama
summary: Günlük kaydı yüzeyleri, dosya günlükleri, WS günlük stilleri ve konsol biçimlendirmesi
title: Gateway günlük kaydı
x-i18n:
    generated_at: "2026-07-26T22:46:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f0b11a68611032c29c31091b2411982487e7f5df3ecf4f1e3b586e7d21e543d3
    source_path: gateway/logging.md
    workflow: 16
---

# Günlük Kaydı

Kullanıcıya yönelik genel bakış (CLI + Control UI + yapılandırma) için [/logging](/tr/logging) sayfasına bakın.

OpenClaw iki günlük yüzeyine sahiptir:

- **Konsol çıktısı** - terminalde / Hata Ayıklama Arayüzünde gördükleriniz.
- **Dosya günlükleri** - Gateway günlük kaydedicisi tarafından yazılan JSON satırları.

Başlangıçta Gateway, çözümlenen varsayılan aracı modelini ve yeni oturumları etkileyen mod varsayılanlarını günlüğe kaydeder:

```text
agent model: openai/gpt-5.6-sol (thinking=medium, fast=on)
```

`thinking` varsayılan aracıdan, model parametrelerinden veya genel aracı varsayılanından gelir; ayarlanmamışsa `medium` gösterilir. `fast` varsayılan aracıdan veya modelin `fastMode` parametrelerinden gelir.

## Dosya tabanlı günlük kaydedici

- Varsayılan döngüsel günlük dosyaları `/tmp/openclaw/` altında bulunur (günde bir dosya) ve Gateway ana makinesinin yerel saat dilimine göre tarihlendirilir. Varsayılan profil `openclaw-YYYY-MM-DD.log` kullanır; adlandırılmış profiller `openclaw-<profile>-YYYY-MM-DD.log` kullanır (örneğin `openclaw-dev-YYYY-MM-DD.log`). Bu dizin güvenli değilse veya yazılabilir değilse (yanlış sahip, herkesçe yazılabilir ya da sembolik bağlantıysa) OpenClaw bunun yerine kullanıcı kapsamlı bir `os.tmpdir()/openclaw-<uid>` yoluna geri döner; Windows'ta her zaman bu işletim sistemi geçici dizini yedeğini kullanır.
- Etkin günlük dosyaları `logging.maxFileBytes` boyutunda döndürülür (varsayılan: 100 MB); en fazla beş numaralı arşiv (`.1` ile `.5` arası) tutulur ve yeni bir etkin dosyaya yazılmaya devam edilir.
- Günlük dosyasının yolunu ve düzeyini `~/.openclaw/openclaw.json` üzerinden yapılandırın: `logging.file`, `logging.level`.
- Dosya biçimi, satır başına bir JSON nesnesidir.

Konuşma, gerçek zamanlı ses ve yönetilen oda kod yolları; operasyonel hata ayıklama ve OTLP günlük dışa aktarımı için tasarlanmış, sınırlandırılmış yaşam döngüsü kayıtlarında paylaşılan dosya günlük kaydedicisini kullanır. Transkript metni, ses yükleri, tur kimlikleri, çağrı kimlikleri ve sağlayıcı öğe kimlikleri hiçbir zaman günlük kaydına kopyalanmaz.

Control UI Günlükler sekmesi, Gateway üzerinden bu dosyayı takip eder (`logs.tail`). CLI da aynısını yapar:

```bash
openclaw logs --follow
```

### Ayrıntılı mod ve günlük düzeyleri

- **Dosya günlükleri** yalnızca `logging.level` tarafından denetlenir.
- `--verbose` yalnızca **konsol ayrıntı düzeyini** (ve WS günlük stilini) etkiler; dosya günlük düzeyini **yükseltmez**.
- Yalnızca ayrıntılı modda bulunan bilgileri dosya günlüklerine kaydetmek için `logging.level` değerini `debug` veya `trace` olarak ayarlayın.
- İzleme günlükleri, Plugin araç fabrikası hazırlığı gibi seçili sıcak yollar için tanılama zamanlama özetlerini de içerir. [/tools/plugin#slow-plugin-tool-setup](/tr/tools/plugin#slow-plugin-tool-setup) sayfasına bakın.

## Konsol yakalama

CLI, `console.log/info/warn/error/debug/trace` öğelerini yakalar, dosya günlüklerine yazar ve stdout/stderr'e yazdırmaya devam eder.

Konsol ayrıntı düzeyini bağımsız olarak ayarlayın:

- `logging.consoleLevel` (varsayılan `info`)
- `logging.consoleStyle` (`pretty` | `compact` | `json`; TTY'de varsayılan olarak `pretty`, aksi durumda `compact`)

## Gizleme

OpenClaw, günlük veya transkript çıktısı süreçten ayrılmadan önce hassas belirteçleri maskeler. Bu gizleme politikası konsol, dosya günlüğü, OTLP günlük kaydı ve oturum transkripti metin hedeflerinde uygulanır; böylece eşleşen gizli değerler, JSONL satırları veya iletiler diske yazılmadan önce maskelenir.

- Hassas değerlerin gizlenmesi her zaman etkindir.
- `logging.redactPatterns`: regex dizelerinden oluşan dizi (varsayılanları geçersiz kılar)
  - Ham regex dizeleri (otomatik `gi`) veya özel bayraklar için `/pattern/flags` kullanın.
  - Eşleşmeler, ilk 6 + son 4 karakter korunarak maskelenir (değerler >= 18 karakterse); daha kısa değerler `***` olur.
  - Varsayılanlar yaygın anahtar atamalarını, CLI bayraklarını, JSON alanlarını, taşıyıcı başlıklarını, PEM bloklarını, popüler sağlayıcı belirteç öneklerini ve ödeme kimlik bilgisi alan adlarını (kart numarası, CVC/CVV, paylaşılan ödeme belirteci, ödeme kimlik bilgisi) kapsar.

Control UI araç çağrısı olayları, `sessions_history` çıktısı, tanılama dışa aktarımları, sağlayıcı hataları, yürütme onayı gösterimi ve Gateway WebSocket günlükleri gibi güvenlik sınırlarında gizleme her zaman uygulanır. `logging.redactPatterns`, dağıtıma özgü kalıplar ekler.

## Gateway WebSocket günlükleri

Gateway, WebSocket protokol günlüklerini iki modda yazdırır:

- **Normal mod (`--verbose` olmadan)**: yalnızca "dikkate değer" RPC sonuçları yazdırılır: hatalar (`ok=false`), yavaş çağrılar (varsayılan eşik: `>= 50ms`) ve ayrıştırma hataları.
- **Ayrıntılı mod (`--verbose`)**: tüm WS istek/yanıt trafiğini yazdırır.

### WS günlük stili

`openclaw gateway`, Gateway başına bir stil anahtarını destekler:

- `--ws-log auto` (varsayılan): normal mod eniyilenmiştir; ayrıntılı mod kompakt çıktı kullanır.
- `--ws-log compact`: ayrıntılı modda kompakt çıktı (eşleştirilmiş istek/yanıt).
- `--ws-log full`: ayrıntılı modda kare başına tam çıktı.
- `--compact`: `--ws-log compact` için diğer ad.

```bash
# eniyilenmiş (yalnızca hatalar/yavaş çağrılar)
openclaw gateway

# tüm WS trafiğini göster (eşleştirilmiş)
openclaw gateway --verbose --ws-log compact

# tüm WS trafiğini göster (tam meta veriler)
openclaw gateway --verbose --ws-log full
```

## Konsol biçimlendirmesi (alt sistem günlük kaydı)

Konsol biçimlendiricisi **TTY'ye duyarlıdır** ve tutarlı, önekli satırlar yazdırır. Alt sistem günlük kaydedicileri çıktıyı gruplandırılmış ve kolay taranabilir durumda tutar:

- Her satırda **alt sistem önekleri** (ör. `[gateway]`, `[canvas]`, `[tailscale]`).
- **Alt sistem renkleri** (alt sistem başına sabittir, addan karma oluşturularak belirlenir) ve düzey renklendirmesi.
- **Çıktı bir TTY olduğunda** veya ortam zengin bir terminal gibi göründüğünde (`TERM`/`COLORTERM`/`TERM_PROGRAM`) renk kullanılır; `NO_COLOR` ve `FORCE_COLOR` dikkate alınır.
- **Kısaltılmış alt sistem önekleri**: baştaki `gateway/`, `channels/` veya `providers/` parçası kaldırılır, ardından kalan parçalardan en fazla son 2 tanesi tutulur (ör. `channels/turn/kernel`, `turn/kernel` olarak görüntülenir). Bilinen kanal alt sistemleri (`telegram`, `whatsapp`, `slack` vb.) her zaman yalnızca kanal adına indirgenir.
- **Alt sisteme göre alt günlük kaydedicileri** (otomatik önek + yapılandırılmış `{ subsystem }` alanı).
- QR/UX çıktısı için **`logRaw()`** (önek ve biçimlendirme yok).
- **Konsol stilleri**: `pretty` | `compact` | `json`.
- **Konsol günlük düzeyi**, dosya günlük düzeyinden ayrıdır (`logging.level`, `debug`/`trace` olduğunda dosya tüm ayrıntıları korur).
- **WhatsApp ileti gövdeleri** `debug` düzeyinde günlüğe kaydedilir (bunları görmek için `--verbose` kullanın).

Bu yaklaşım, dosya günlüklerini kararlı tutarken etkileşimli çıktının kolayca taranabilmesini sağlar.

## İlgili Konular

- [Günlük Kaydı](/tr/logging)
- [OpenTelemetry dışa aktarımı](/tr/gateway/opentelemetry)
- [Tanılama dışa aktarımı](/tr/gateway/diagnostics)
