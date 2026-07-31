---
read_when:
    - Gateway günlüklerini uzaktan takip etmeniz gerekiyor (SSH kullanmadan)
    - Araçlar için JSON günlük satırları istiyorsunuz
summary: '`openclaw logs` için CLI başvurusu (RPC üzerinden Gateway günlüklerini takip etme)'
title: Günlükler
x-i18n:
    generated_at: "2026-07-26T22:41:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c8dc40e70f2eb4f8d6ba8b75b91a33337786a146abbe401079ee374daa5a0c6
    source_path: cli/logs.md
    workflow: 16
---

# `openclaw logs`

Gateway dosya günlüklerini RPC üzerinden canlı olarak izleyin. Uzak modda çalışır.

## Seçenekler

- `--limit <n>`: döndürülecek en fazla günlük satırı sayısı (varsayılan `200`)
- `--max-bytes <n>`: günlük dosyasından okunacak en fazla bayt sayısı (varsayılan `250000`)
- `--follow`: günlük akışını takip et
- `--interval <ms>`: takip sırasındaki yoklama aralığı (varsayılan `1000`)
- `--json`: satırlarla ayrılmış JSON olayları üret
- `--plain`: biçemli biçimlendirme olmadan düz metin çıktısı
- `--no-color`: ANSI renklerini devre dışı bırak
- `--local-time`: zaman damgalarını yerel saat diliminizde göster (varsayılan)
- `--utc`: zaman damgalarını UTC olarak göster

## Paylaşılan Gateway RPC seçenekleri

- `--url <url>`: Gateway WebSocket URL'si
- `--token <token>`: Gateway belirteci
- `--timeout <ms>`: ms cinsinden zaman aşımı (varsayılan `30000`)
- `--expect-final`: Gateway çağrısı bir ajan tarafından destekleniyorsa nihai yanıtı bekle

`--url` iletmek, otomatik uygulanan yapılandırma kimlik bilgilerini atlar; hedef Gateway kimlik doğrulaması gerektiriyorsa `--token` değerini açıkça ekleyin.

## Örnekler

```bash
openclaw logs
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
openclaw logs --follow --interval 2000
openclaw logs --limit 500 --max-bytes 500000
openclaw logs --json
openclaw logs --plain
openclaw logs --no-color
openclaw logs --utc
openclaw logs --follow --local-time
openclaw logs --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

Seçilen kök profil, Gateway'in dönüşümlü dosyasıyla eşleşir: varsayılan
profil `openclaw-YYYY-MM-DD.log` kullanırken adlandırılmış profiller
`openclaw-<profile>-YYYY-MM-DD.log` kullanır (örneğin,
`openclaw-dev-YYYY-MM-DD.log`).

## Geri dönüş ve kurtarma davranışı

- Örtük yerel geri döngü Gateway'i eşleştirme isterse, bağlantı sırasında kapanırsa veya `logs.tail` yanıt vermeden önce zaman aşımına uğrarsa `openclaw logs`, yapılandırılmış Gateway dosya günlüğüne otomatik olarak geri döner. Açık `--url` hedefleri bu geri dönüşü hiçbir zaman kullanmaz.
- `--follow`, örtük yerel Gateway RPC hatasından sonra bu yapılandırılmış dosyaya geri dönmez; güncelliğini yitirmiş yan yana bir dosya, canlı izlemeyi yanıltabilir. Bunun yerine Linux'ta, mevcut olduğunda etkin kullanıcı-systemd Gateway günlüğünü PID'ye göre kullanır (seçilen kaynağı yazdırır); aksi takdirde canlı Gateway'e yeniden bağlanmayı sürdürür.
- `--follow` sırasında geçici bağlantı kesintileri (WebSocket kapanması, zaman aşımı, bağlantının kopması), üstel geri çekilmeyle otomatik yeniden bağlantıyı tetikler: en fazla 8 yeniden deneme ve denemeler arasında en fazla 30 sn. Her yeniden denemede stderr'e bir uyarı yazdırılır ve bir yoklama başarılı olduğunda bir kez `[logs] gateway reconnected` bildirimi yazdırılır. `--json` modunda her ikisi de stderr'de `{"type":"notice"}` kayıtları olarak üretilir. Kurtarılamayan hatalar (kimlik doğrulama hatası, hatalı yapılandırma) hâlâ hemen çıkılmasına neden olur.
- `--follow --json` modunda günlük kaynağı geçişleri `{"type":"meta"}` kayıtları olarak üretilir. İmleçleri her `sourceKind` için ayrı izleyin: bir akış, Gateway dosya çıktısından (`sourceKind: "file"`) yerel günlük geri dönüşüne (`sourceKind: "journal"`, `localFallback: true`, `service.pid`/`service.unit` ile) geçebilir ve kurtarma sonrasında Gateway dosya çıktısına geri dönebilir. Oturumun tamamı boyunca tek bir sabit kaynak veya imleç olduğunu varsaymayın ve kurtarma Gateway dosya imlecini yeniden oynattığında çakışan satırlara tolerans gösterin.

## İlgili

- [Günlük kaydına genel bakış](/tr/logging)
- [Gateway CLI](/tr/cli/gateway)
- [CLI başvurusu](/tr/cli)
- [Gateway günlük kaydı](/tr/gateway/logging)
