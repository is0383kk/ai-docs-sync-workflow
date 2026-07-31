---
read_when:
    - macOS günlüklerini yakalama veya özel veri günlüğe kaydını araştırma
    - Sesle uyandırma/oturum yaşam döngüsü sorunlarında hata ayıklama
summary: 'OpenClaw günlük kaydı: döngüsel tanılama dosyası günlüğü + birleşik günlük gizlilik bayrakları'
title: macOS günlük kaydı
x-i18n:
    generated_at: "2026-07-26T23:28:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ef0fd91bd7fc0a8b5f598cfe8f5de551795a4badd0f6634c5bcbd4f3916bfc64
    source_path: platforms/mac/logging.md
    workflow: 16
---

# Günlük Kaydı (macOS)

## Dönen tanılama dosyası günlüğü (Hata Ayıklama bölmesi)

macOS uygulaması swift-log üzerinden günlük kaydı tutar (varsayılan olarak birleşik günlük kaydı) ve kalıcı kayıt için dönen bir yerel dosya günlüğüne de yazabilir (`DiagnosticsFileLog`).

- Etkinleştirme: **Debug pane -> Logs -> App logging -> "Write rolling diagnostics log (JSONL)"** (varsayılan olarak kapalı).
- Ayrıntı düzeyi: **Debug pane -> Logs -> App logging -> Verbosity** seçicisi.
- Konum: `~/Library/Logs/OpenClaw/diagnostics.jsonl`.
- Döndürme: 5 MB boyutunda döndürülür; `.1`...`.5` son eklerini taşıyan en fazla 5 yedek tutulur (en eskisi silinir).
- Temizleme: **Debug pane -> Logs -> App logging -> "Clear"**, etkin dosyayı ve tüm yedekleri siler.

Dosyayı hassas olarak değerlendirin; incelemeden paylaşmayın.

## macOS'ta birleşik günlük kaydındaki özel veriler

Birleşik günlük kaydı, bir alt sistem `privacy -off` kullanımını açıkça etkinleştirmediği sürece çoğu yükü sansürler. Bu, `/Library/Preferences/Logging/Subsystems/` içindeki alt sistem adına göre anahtarlanmış bir plist tarafından denetlenir. Yalnızca yeni günlük girdileri bu ayarı kullanır; bu nedenle bir sorunu yeniden oluşturmadan önce etkinleştirin. Arka plan bilgisi: [macOS günlük kaydı gizliliğiyle ilgili karmaşalar](https://steipete.me/posts/2025/logging-privacy-shenanigans).

## OpenClaw için etkinleştirme (`ai.openclaw`)

Önce plist'i geçici bir dosyaya yazın, ardından root olarak atomik biçimde yükleyin:

```bash
cat <<'EOF' >/tmp/ai.openclaw.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>DEFAULT-OPTIONS</key>
    <dict>
        <key>Enable-Private-Data</key>
        <true/>
    </dict>
</dict>
</plist>
EOF
sudo install -m 644 -o root -g wheel /tmp/ai.openclaw.plist /Library/Preferences/Logging/Subsystems/ai.openclaw.plist
```

Yeniden başlatma gerekmez; logd dosyayı kısa sürede algılar, ancak özel yükler yalnızca yeni günlük satırlarına dahil edilir. Daha zengin çıktıyı `./scripts/clawlog.sh --category WebChat --last 5m` ile görüntüleyin (`--last`/`-l` zaman aralığını belirler; varsayılan `5m`; `--category`/`-c` kategoriye göre filtreler).

## Hata ayıklamadan sonra devre dışı bırakma

- Geçersiz kılmayı kaldırın: `sudo rm /Library/Preferences/Logging/Subsystems/ai.openclaw.plist`.
- İsteğe bağlı olarak, logd'nin geçersiz kılmayı hemen bırakmasını sağlamak için `sudo log config --reload` komutunu çalıştırın.
- Bu yüzey telefon numaralarını ve ileti gövdelerini içerebilir; plist'i yalnızca etkin olarak gerektiği sürece yerinde tutun.

## İlgili

- [macOS uygulaması](/tr/platforms/macos)
- [Gateway günlük kaydı](/tr/gateway/logging)
