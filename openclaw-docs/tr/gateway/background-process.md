---
read_when:
    - Arka planda yürütme davranışı ekleme veya değiştirme
    - Uzun süre çalışan exec görevlerinde hata ayıklama
summary: Arka planda exec yürütme ve süreç yönetimi
title: Arka planda yürütme ve süreç aracı
x-i18n:
    generated_at: "2026-07-26T23:58:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 37cb65ddf67227e32be972e77d16b9835d592120ecd12e041d05c48536fd2204
    source_path: gateway/background-process.md
    workflow: 16
---

OpenClaw, kabuk komutlarını `exec` aracı üzerinden çalıştırır ve uzun süre çalışan görevleri bellekte tutar. `process` aracı bu arka plan oturumlarını yönetir.

## exec aracı

Parametreler:

| Parametre    | Açıklama                                                                                                                                                |
| ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `command`    | Zorunludur. Çalıştırılacak kabuk komutu.                                                                                                                            |
| `workdir`    | Çalışma dizini; varsayılan mevcut çalışma dizinini kullanmak için belirtilmez.                                                                                                            |
| `env`        | Komut için ek ortam değişkenleri.                                                                                                               |
| `yieldMs`    | Arka plana almadan önce beklenecek milisaniye (varsayılan 10000).                                                                                                 |
| `background` | Hemen arka planda çalıştırır.                                                                                                                             |
| `timeout`    | Saniye cinsinden zaman aşımı (varsayılan `tools.exec.timeoutSeconds`); süre dolduğunda işlemi sonlandırır. Bu çağrı için exec işlem zaman aşımını devre dışı bırakmak üzere `timeout: 0` olarak ayarlayın. |
| `pty`        | Kullanılabildiğinde sözde terminalde çalıştırır (TTY gerektiren CLI'lar, kodlama ajanları).                                                                                |
| `elevated`   | Yükseltilmiş mod etkinse/izin veriliyorsa korumalı alanın dışında çalıştırır (varsayılan olarak `gateway` veya exec hedefi `node` olduğunda `node`).                              |
| `host`       | Exec hedefi: `auto`, `sandbox`, `gateway` veya `node`.                                                                                                      |
| `node`       | `host: "node"` ile kullanılan Node kimliği/adı.                                                                                                                    |

Davranış:

- Ön plandaki çalıştırmalar çıktıyı doğrudan döndürür.
- Arka plana alındığında (açıkça veya `yieldMs` zaman aşımı yoluyla), araç `status: "running"` + `sessionId` ve çıktının kısa bir son bölümünü döndürür.
- Arka plana alınan ve `yieldMs` çalıştırmaları, çağrı açık bir `timeout` iletmediği sürece `tools.exec.timeoutSeconds` değerini devralır.
- Çıktı, oturum yoklanana veya temizlenene kadar bellekte kalır.
- `process` aracına izin verilmiyorsa `exec` eşzamanlı olarak çalışır ve `yieldMs`/`background` değerlerini yok sayar.
- Başlatılan exec komutları, bağlama duyarlı kabuk/profil kuralları için `OPENCLAW_SHELL=exec` değerini alır.
- Şimdi başlayan uzun süreli işler için: işi bir kez başlatın ve komut çıktı verdiğinde veya başarısız olduğunda otomatik tamamlanma uyandırmasına (etkinse) güvenin.
- Otomatik tamamlanma uyandırması kullanılamıyorsa veya çıktı vermeden başarıyla sonlanan bir komut için sessiz başarı onayı gerekiyorsa `process` ile yoklayın.
- Hatırlatıcıları veya gecikmeli takipleri `sleep` döngüleriyle ya da tekrarlanan yoklamalarla taklit etmeyin — gelecekteki işler için Cron kullanın.

### Ortam değişkeni geçersiz kılmaları

| Değişken                                 | Etki                                                                                                           |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `OPENCLAW_BASH_YIELD_MS`                 | Arka plana almadan önce varsayılan bekleme süresi (ms). Varsayılan 10000, 10-120000 aralığıyla sınırlandırılır.                                       |
| `OPENCLAW_BASH_MAX_OUTPUT_CHARS`         | Bellek içi çıktı sınırı (karakter).                                                                                    |
| `OPENCLAW_BASH_PENDING_MAX_OUTPUT_CHARS` | Akış başına bekleyen stdout/stderr sınırı (karakter).                                                                    |
| `OPENCLAW_BASH_JOB_TTL_MS`               | Tamamlanan oturumların yaşam süresi (ms), 1 dk.-3 sa. aralığıyla sınırlandırılır.                                                                |
| `OPENCLAW_PROCESS_INPUT_WAIT_IDLE_MS`    | Yazılabilir arka plan oturumlarının büyük olasılıkla girdi bekliyor olarak işaretlenmesinden önceki boşta çıktı eşiği. Varsayılan 15000. |

### Yapılandırma (ortam değişkeni geçersiz kılmalarına tercih edilir)

| Anahtar                                   | Varsayılan | Etki                                                                          |
| ------------------------------------- | ------- | ------------------------------------------------------------------------------- |
| `tools.exec.backgroundMs`             | 10000   | `OPENCLAW_BASH_YIELD_MS` ile aynıdır.                                               |
| `tools.exec.timeoutSeconds`           | 1800    | Çağrı başına varsayılan zaman aşımı.                                                       |
| `tools.exec.cleanupMs`                | 1800000 | `OPENCLAW_BASH_JOB_TTL_MS` ile aynıdır.                                             |
| `tools.exec.notifyOnExit`             | true    | Arka plana alınmış bir exec sonlandığında bir sistem olayını kuyruğa alır ve Heartbeat ister.      |
| `tools.exec.notifyOnExitEmptySuccess` | false   | Çıktı vermeden başarıyla tamamlanan arka plan çalıştırmaları için de tamamlanma olaylarını kuyruğa alır. |

## Alt süreç köprüleme

Exec/process araçları dışında uzun süre çalışan alt süreçler başlatırken (CLI yeniden başlatmaları, Gateway yardımcıları), sonlandırma sinyallerinin iletilmesi ve çıkış/hata durumunda dinleyicilerin ayrılması için alt süreç köprü yardımcısını ekleyin. Bu, systemd üzerinde sahipsiz süreçleri önler ve kapatmanın platformlar arasında tutarlı kalmasını sağlar.

## process aracı

Eylemler:

| Eylem      | Etki                                                                        |
| ----------- | ----------------------------------------------------------------------------- |
| `list`      | Çalışan + tamamlanan oturumlar.                                                  |
| `poll`      | Bir oturumun yeni çıktısını tüketir (çıkış durumunu da bildirir).                    |
| `log`       | Birleştirilmiş çıktıyı ve girdi kurtarma ipuçlarını okur. `offset` + `limit` destekler. |
| `write`     | stdin gönderir (`data`, isteğe bağlı `eof`).                                          |
| `send-keys` | PTY destekli bir oturuma açık tuş belirteçleri veya baytlar gönderir.                    |
| `submit`    | PTY destekli bir oturuma Enter/satır başı gönderir.                           |
| `paste`     | İsteğe bağlı olarak köşeli ayraçlı yapıştırma moduna sarılmış düz metin gönderir.                |
| `kill`      | Bir arka plan oturumunu sonlandırır.                                               |
| `clear`     | Tamamlanmış bir oturumu bellekten kaldırır.                                        |
| `remove`    | Çalışıyorsa sonlandırır, tamamlanmışsa temizler.                                 |

Notlar:

- Yalnızca arka plana alınan oturumlar listelenir/saklanır — yalnızca bellekte tutulur, diske yazılmaz. İşlem yeniden başlatıldığında oturumlar kaybolur.
- Canlı bir arka plan oturumu, süreç sahibi gerçekten sonlandığını doğrulayana kadar işbirlikçi ana makine askıya almasını ve güvenli Gateway yeniden başlatmasını engeller.
- `process remove`, sonlandırma istendikten hemen sonra çalışan bir oturumu gizleyebilir; askıya alma ve yeniden başlatma, çıkış onaylanana kadar engellenmeye devam eder.
- Oturum günlükleri yalnızca `process poll`/`log` çalıştırılır ve araç sonucu kaydedilirse sohbet geçmişine kaydedilir.
- `process` ajan başına kapsamlandırılır; yalnızca o ajan tarafından başlatılan oturumları görür.
- Otomatik tamamlanma uyandırması kullanılamadığında durum, günlükler veya tamamlanma onayı için `poll`/`log` kullanın.
- Etkileşimli bir CLI'ı kurtarmadan önce `log` kullanın; böylece mevcut transkript, stdin durumu ve girdi bekleme ipucu birlikte görünür.
- Girdi veya müdahale gerektiğinde `write`/`send-keys`/`submit`/`paste`/`kill` kullanın.
- `process list`, hızlı taramalar için türetilmiş bir `name` (komut fiili + hedef) içerir.
- `process list`, `poll` ve `log`, yalnızca oturumda hâlâ yazılabilir stdin varsa ve oturum girdi bekleme eşiğinden (varsayılan 15000 ms, `OPENCLAW_PROCESS_INPUT_WAIT_IDLE_MS`) daha uzun süredir boşta ise `waitingForInput` bildirir.
- `process log`, satır tabanlı `offset`/`limit` kullanır. İkisi de belirtilmediğinde, sayfalama ipucuyla birlikte son 200 satırı döndürür. `offset` ayarlanmış ancak `limit` ayarlanmamışsa `offset` değerinden sona kadar döndürür (200 ile sınırlandırılmaz).
- `poll` öğesinin `timeout` değeri, dönmeden önce en fazla belirtilen milisaniye kadar bekler; 30000 üzerindeki değerler 30000 ile sınırlandırılır.
- Yoklama, isteğe bağlı durum sorgulama içindir; bekleme döngüsü zamanlaması için değildir. İş daha sonra yapılacaksa Cron kullanın.

## Örnekler

Uzun bir görev çalıştırıp daha sonra yoklayın:

```json
{ "tool": "exec", "command": "sleep 5 && echo done", "yieldMs": 1000 }
```

```json
{ "tool": "process", "action": "poll", "sessionId": "<id>" }
```

Girdi göndermeden önce etkileşimli bir oturumu inceleyin:

```json
{ "tool": "process", "action": "log", "sessionId": "<id>" }
```

Hemen arka planda başlatın:

```json
{ "tool": "exec", "command": "npm run build", "background": true }
```

stdin gönderin:

```json
{ "tool": "process", "action": "write", "sessionId": "<id>", "data": "y\n" }
```

PTY tuşlarını gönderin:

```json
{ "tool": "process", "action": "send-keys", "sessionId": "<id>", "keys": ["C-c"] }
```

Geçerli satırı gönderin:

```json
{ "tool": "process", "action": "submit", "sessionId": "<id>" }
```

Düz metin yapıştırın:

```json
{ "tool": "process", "action": "paste", "sessionId": "<id>", "text": "line1\nline2\n" }
```

## İlgili

- [Exec aracı](/tr/tools/exec)
- [Exec onayları](/tr/tools/exec-approvals)
