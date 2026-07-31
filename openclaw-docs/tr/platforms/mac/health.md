---
read_when:
    - Mac uygulaması durum göstergelerinde hata ayıklama
summary: macOS uygulamasının Gateway/kanal sağlık durumlarını nasıl bildirdiği
title: Durum denetimleri (macOS)
x-i18n:
    generated_at: "2026-07-26T22:51:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 095abdbefa7db7c0d14435e2c5db7d1ebc03afa0c539555a7abdd9170d015fb8
    source_path: platforms/mac/health.md
    workflow: 16
---

# macOS'ta sistem durumu denetimleri

Menü çubuğu uygulamasından bağlı kanalın sistem durumu bilgisinin nasıl okunacağı.

## Menü çubuğu

Durum noktası:

- Yeşil: bağlı + yoklama sağlıklı.
- Turuncu: bağlı ancak bir kanal yoklaması performans düşüşü/bağlı değil durumu bildiriyor.
- Kırmızı: henüz bağlı değil.

İkincil satırda "bağlı · kimlik doğrulama 12 dk." ifadesi veya hata nedeni gösterilir.
Menüdeki "Sistem Durumu Denetimini Şimdi Çalıştır" seçeneği, isteğe bağlı bir yoklama başlatır.

## Ayarlar

- Genel sekmesinde bir Sistem Durumu kartı gösterilir: durum noktası, özet satırı (bağlantı durumu +
  kimlik doğrulama yaşı) ve isteğe bağlı bir hata ayrıntısı satırı ile **Şimdi yeniden dene** ve
  **Günlükleri aç** düğmeleri.
- **Kanallar sekmesi**, WhatsApp ve Telegram için kanal bazında durum ve denetimleri (oturum açma QR kodu,
  oturumu kapatma, yoklama, son bağlantı kesilmesi/hata) gösterir.

## Yoklama nasıl çalışır?

Uygulama, mevcut WebSocket bağlantısı üzerinden (CLI kabuk çağrısı kullanmadan) yaklaşık her 60 saniyede bir ve
istek üzerine Gateway'in `health` RPC'sini çağırır. RPC, kimlik bilgilerini yükler
ve mesaj göndermeden durumu bildirir. Uygulama, kullanıcı arayüzünün anında yüklenmesi ve
çevrimdışıyken titrememesi için son başarılı anlık görüntüyü ve son hatayı ayrı ayrı önbelleğe alır.

## Emin olmadığınızda

[Gateway sistem durumu](/tr/gateway/health) sayfasındaki CLI akışını kullanın (`openclaw status`,
`openclaw status --deep`, `openclaw health --json`) ve
`web-heartbeat` / `web-reconnect` için filtreleyerek `openclaw logs --follow` komutunu çalıştırın.

## İlgili

- [Gateway sistem durumu](/tr/gateway/health)
- [macOS uygulaması](/tr/platforms/macos)
