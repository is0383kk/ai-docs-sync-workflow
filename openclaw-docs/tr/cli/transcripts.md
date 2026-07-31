---
read_when:
    - Depolanan transkript özetlerini terminalden okumak istiyorsunuz
    - Transkriptlerin Markdown özetinin yoluna ihtiyacınız var
    - Temel transkript depolama düzeninde hata ayıklıyorsunuz
summary: '`openclaw transcripts` için CLI başvurusu (saklanan transkriptleri listeleme, gösterme ve dışa aktarma)'
title: Transkriptler CLI'si
x-i18n:
    generated_at: "2026-07-26T22:42:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c04ba637fb46ec271383b2f0d17655e18018e07f489c34dc3fd14ad926f27aa4
    source_path: cli/transcripts.md
    workflow: 16
---

# `openclaw transcripts`

Kalıcı toplantı dökümleri için inceleme ve dışa aktarma komutu. Google Meet,
Microsoft Teams ve Zoom tarayıcı katılımcıları notları otomatik olarak yakalar;
`transcripts` aracı ayrıca sağlayıcı yakalamayı ve manuel içe aktarmayı destekler.

Standart döküm durumu, `$OPENCLAW_STATE_DIR/state/openclaw.sqlite` konumundaki paylaşılan SQLite veritabanında
bulunur. `show` ve `path`, durum dizini altında kullanıcıya
yönelik yapıtları açıkça oluşturur:

```text
$OPENCLAW_STATE_DIR/transcripts/YYYY-MM-DD/<session>/
  metadata.json
  transcript.jsonl
  summary.json
  summary.md
```

Bu dosyalar dışa aktarımlardır; ikinci bir çalışma zamanı deposu değildir. OpenClaw,
yakalama, özetleme veya listeleme sırasında bunları geri okumaz. Varsayılan durum
dizini `~/.openclaw` konumudur; `OPENCLAW_STATE_DIR` ile geçersiz kılınabilir.
Tarih dizini oturumun başlangıç zamanından gelir; oturum dizini ise oturum kimliğinden
türetilen, dosya sistemi açısından güvenli bir kısa addır.

## Komutlar

```bash
openclaw transcripts list
openclaw transcripts show <session>
openclaw transcripts show YYYY-MM-DD/<session>
openclaw transcripts path <session>
openclaw transcripts path YYYY-MM-DD/<session>
openclaw transcripts path <session> --dir
openclaw transcripts path <session> --metadata
openclaw transcripts path <session> --transcript
openclaw transcripts list --json
openclaw transcripts show <session> --json
openclaw transcripts path <session> --json
```

| Komut                         | Açıklama                                             |
| ----------------------------- | ---------------------------------------------------- |
| `list`            | Depolanan oturumları listeler.                       |
| `show <session>`            | `summary.md` oluşturur ve yazdırır.            |
| `path <session>`            | `summary.md` yolunu oluşturur ve yazdırır.     |
| `path <session> --dir`            | Tüm yapıtları oluşturur ve dizinlerini yazdırır.     |
| `path <session> --metadata`            | `metadata.json` oluşturur ve yazdırır.            |
| `path <session> --transcript`            | `transcript.jsonl` oluşturur ve yazdırır.            |
| `--json`            | Makine tarafından okunabilir çıktı yazdırır (herhangi bir alt komut). |

`<session>`, yalın bir oturum kimliğini veya tarih nitelemeli bir seçiciyi
(`YYYY-MM-DD/<session>`) kabul eder. Aynı oturum kimliği birden fazla günde
geçiyorsa nitelemeli biçimi kullanın; örneğin `openclaw transcripts show
2026-05-22/standup`. Varsayılan oturum
kimlikleri bir zaman damgası ve rastgele bir son ek içerir; bir oturuma yalnızca bu
kimlik gün içinde benzersiz olduğunda sabit bir kimlik verin.

## Çıktı

`list`, her oturum için sekmeyle ayrılmış bir satır yazdırır: seçici,
başlangıç zamanı, başlık, özet yolu.

```text
2026-05-22/standup  2026-05-22T09:00:00.000Z  Haftalık durum toplantısı  /Users/user/.openclaw/transcripts/2026-05-22/standup/summary.md
```

Seçici, `show` veya `path` öğesine geri iletilecek en güvenli değerdir.

`list --json`; `sessionId`, `selector`, `date`, `title`,
`startedAt`, `stoppedAt`, `source`, `path`, `summaryPath`, `hasSummary`
alanlarını içeren nesneler döndürür. Depolanan toplantı kaynak URL'leri yalnızca
kaynağı ve yolu içerir; sorgu dizeleri, parçalar ve gömülü kimlik bilgileri kalıcı
depolamadan önce kaldırılır.

`show --json`, depolanan oturum meta verilerini, seçiciyi, oturum
dizinini, özet yolunu ve özet Markdown metnini döndürür.

`path --json`, seçilen yolu ve bu yapıtın oluşturulup oluşturulamadığını
döndürür. Depolanan bir oturum için meta veri ve döküm dışa aktarımları her zaman
mevcuttur; oturumun özeti oluşana kadar özet yolu `exists: false` bildirir.

## Gün başına birden fazla oturum

Oturumlar önce tarihe, ardından oturum kimliğine göre gruplanır. Bir gündeki on
toplantı, aynı düzeyde on klasör hâline gelir:

```text
~/.openclaw/transcripts/2026-05-22/
  transcript-2026-05-22T09-00-00-000Z-a1b2c3d4/
  transcript-2026-05-22T10-30-00-000Z-b2c3d4e5/
  standup/
```

Otomasyon için varsayılan olarak oluşturulan kimlikleri kullanın. `standup`
gibi sabit bir kimliği yalnızca aynı tarihte yinelenmeyecekse kullanın.

## Eksik özetler

Canlı oturumlar, oturum durduğunda `summary.md` depolar ve oluşturur;
içe aktarılan dökümler ise bunu içe aktarmanın hemen ardından yapar. Yakalama hâlâ
etkinken, bir sağlayıcı durdurma sırasında başarısız olduğunda veya herhangi bir
ifade ulaşmadan önce meta veriler depolandığında bir oturum özetsiz olarak
`list` içinde görünebilir.

Ham, yalnızca sona eklemeli dökümü incelemek için `path <session> --transcript` kullanın
veya Markdown özetini yeniden oluşturmak için `transcripts` aracının
`summarize` eylemini çalıştırın.

## Eski dosya deposunu yükseltme

SQLite deposundan önceki OpenClaw sürümleri, standart çalışma zamanı durumunu
doğrudan `$OPENCLAW_STATE_DIR/transcripts/` altına yazıyordu. Şunu çalıştırın:

```bash
openclaw doctor --fix
```

Doctor, eski ağacın tamamını SQLite'a aktarır, satır sayılarını ve sıralamayı
doğrular, geçiş kayıtlarını kaydeder ve doğrulanan kaynak ağacı zaman damgalı
bir `transcripts.migrated-*` arşivine taşır. Çalışma zamanı komutları eski dosyalara
geri dönmez. İçe aktarılan oturumları ve kullandığınız dışa aktarımları
doğrulayana kadar arşivi saklayın.

## Yapılandırma

Toplantı dökümü yakalama varsayılan olarak etkindir. Genel olarak devre dışı
bırakmak için:

```json
{
  "transcripts": {
    "enabled": false
  }
}
```

- `enabled` (varsayılan `true`): otomatik toplantı notlarını, dökümler
  aracını ve yapılandırılmış otomatik başlatma kaynaklarını etkinleştirir. Toplantı
  notlarının ana makinede kalıcı olarak saklanmaması gerekiyorsa bunu
  `false` olarak ayarlayın. Açıkça istenen toplantı
  `transcribe` modu, mevcut sınırlı canlı altyazı kuyruğunu korur ancak bu ayar
  false olduğunda kalıcı satırlar yazmaz.
  Otomatik başlatma kaynaklarını `transcripts.autoStart` ile yapılandırın. Her girdi
  mevcut olduğunda etkinleşir; ilgili kaynağı devre dışı bırakmak için girdiyi
  atlayın. `discord-voice`, paketle birlikte gelen otomatik başlatma destekli
  kaynaktır ve `guildId` ile `channelId` gerektirir:

```json
{
  "transcripts": {
    "enabled": true,
    "autoStart": [
      {
        "providerId": "discord-voice",
        "guildId": "1234567890",
        "channelId": "2345678901"
      }
    ]
  }
}
```

Toplantı sağlayıcısı kimlikleri sırasıyla `google-meet`, `teams`
ve `zoom` değerleridir. Bunların takma adları sırasıyla
`googlemeet`/`meet`, `teams-meetings`/`microsoft-teams`/`msteams`
ve `zoom-meetings` değerleridir. Toplantı sağlayıcıları zaten etkin olan bir
toplantı botu oturumuna bağlanır; normal toplantı katılımları için bir
`autoStart` girdisi gerekmez.
