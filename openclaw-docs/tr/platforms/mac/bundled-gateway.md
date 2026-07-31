---
read_when:
    - OpenClaw.app'i Paketleme
    - macOS Gateway launchd hizmetinde hata ayıklama
    - macOS için Gateway CLI'yi yükleme
summary: macOS'ta Gateway çalışma zamanı (harici launchd hizmeti)
title: macOS'te Gateway
x-i18n:
    generated_at: "2026-07-26T23:25:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 30c1ae14d8f8eaab73d0e2b725292d7411c2c8b5e0e0c32ad13989c01340d054
    source_path: platforms/mac/bundled-gateway.md
    workflow: 16
---

OpenClaw.app, Node veya Gateway çalışma zamanını paketlemez. macOS uygulaması
**harici** bir `openclaw` CLI kurulumu bekler, Gateway'i
bir alt süreç olarak başlatmaz ve Gateway'in çalışmasını sürdürmek için kullanıcı başına bir launchd hizmetini
yönetir (veya zaten çalışan yerel bir Gateway'e bağlanır).

## Otomatik kurulum

Yeni bir Mac'te ilk katılım sırasında **This Mac** seçeneğini belirleyin. Uygulama,
Gateway sihirbazından önce imzalı, paketlenmiş yükleyici betiğini çalıştırır: kullanıcı
alanına bir Node çalışma zamanı ve `~/.openclaw` altında eşleşen `openclaw` CLI'ı
kurar, ardından kullanıcı başına launchd hizmetini kurup başlatır. Bu yol için
Terminal, Homebrew veya yönetici erişimi gerekmez.

Uygulama yalnızca yükleyici betiğini paketler; Node veya Gateway yükünü paketlemez.
Kurulumun çalışma zamanını ve eşleşen OpenClaw paketini indirmesi için
internet bağlantısı gerekir.

## Manuel kurtarma

Manuel kurulum için Node 24.15+ önerilir; Node 22.22.3+ da çalışır. `openclaw`
paketini global olarak kurun:

```bash
npm install -g openclaw@<version>
```

Başarısız bir otomatik kurulumdan sonra **Retry setup** seçeneğini kullanın. Bu da başarısız olursa,
yukarıdaki komutla CLI'ı manuel olarak kurun ve ardından ilk katılımda **Check again**
seçeneğini belirleyin.

## Launchd (LaunchAgent olarak Gateway)

Etiket: `ai.openclaw.gateway` (varsayılan profil) veya adlandırılmış bir profil
için `ai.openclaw.<profile>`.

Plist konumu (kullanıcı başına): `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
(veya `ai.openclaw.<profile>.plist`).

macOS uygulaması, Yerel modda varsayılan profil için LaunchAgent kurulumunu/güncellemesini
yönetir. CLI da bunu doğrudan kurabilir: `openclaw gateway install`
(adlandırılmış profiller `OPENCLAW_PROFILE` ortam değişkeni aracılığıyla seçilir).

Davranış:

- "OpenClaw Active", LaunchAgent'ı etkinleştirir/devre dışı bırakır.
- Uygulamadan çıkmak Gateway'i **durdurmaz** (launchd onu çalışır durumda tutar).
- Yapılandırılmış bağlantı noktasında zaten bir Gateway çalışıyorsa uygulama,
  yenisini başlatmak yerine ona bağlanır.

Günlük kaydı:

- launchd standart çıktısı: `~/Library/Logs/openclaw/gateway.log` (profiller
  `gateway-<profile>.log` kullanır)
- launchd standart hatası: bastırılır
- Ana makine, tekrarlanan `EADDRINUSE` veya hızlı yeniden başlatmalarla döngüye girerse,
  yinelenen `ai.openclaw.gateway` / `ai.openclaw.node` LaunchAgent'ları ve
  [Gateway sorun giderme](/tr/gateway/troubleshooting#macos-launchd-supervisor-loop-with-duplicate-gatewaynode-launchagents)
  bölümündeki launchd işaretleyicisi geçici çözümünü kontrol edin.

## Sürüm uyumluluğu

macOS uygulaması, Gateway sürümünü kendi sürümüyle karşılaştırır. Mevcut bir CLI
eksik veya uyumsuz olduğunda ilk katılım, yönetilen kurulumu otomatik olarak çalıştırır.
Kurulumu tekrarlamak için **Retry setup** seçeneğini veya harici bir CLI'ı onardıktan sonra
**Check again** seçeneğini kullanın.

## macOS'taki durum dizini

OpenClaw durumunu yerel, eşitlenmeyen bir diskte tutun. iCloud Drive ve diğer
bulutla eşitlenen klasörlerden kaçının; eşitleme gecikmesi ve dosya kilitleri oturumları,
kimlik bilgilerini ve Gateway durumunu etkileyebilir.

Yalnızca geçersiz kılma gerektiğinde `OPENCLAW_STATE_DIR` değerini yerel bir yola ayarlayın.
`openclaw doctor`, yaygın bulutla eşitlenen durum yolları hakkında uyarır ve
yerel depolamaya geri taşımayı önerir. Bkz.
[ortam değişkenleri](/tr/help/environment#path-related-env-vars) ve
[Doctor](/tr/gateway/doctor).

## Uygulama bağlantısında hata ayıklama

Uygulamanın kullandığı Gateway WebSocket el sıkışmasını ve keşif mantığını
çalıştırmak için kaynak kod deposundan macOS hata ayıklama CLI'ını kullanın:

```bash
cd apps/macos
swift run openclaw-mac connect --json
swift run openclaw-mac discover --timeout 3000 --json
```

`connect`; `--url`, `--token`, `--timeout`, `--probe` ve `--json`
seçeneklerini kabul eder (istemci kimliği geçersiz kılmaları da dahildir; tam liste için
`--help` ile çalıştırın).
`discover`; `--timeout`, `--json` ve `--include-local` seçeneklerini kabul eder. CLI keşfini
uygulama tarafındaki bağlantı sorunlarından ayırmanız gerektiğinde keşif çıktısını
`openclaw gateway discover --json` ile karşılaştırın.

## Hızlı kontrol

```bash
openclaw --version

OPENCLAW_SKIP_CHANNELS=1 \
OPENCLAW_SKIP_CANVAS_HOST=1 \
openclaw gateway --port 18999 --bind loopback
```

Ardından:

```bash
openclaw gateway call health --url ws://127.0.0.1:18999 --timeout 3000
```

## İlgili

- [macOS uygulaması](/tr/platforms/macos)
- [Gateway işletim kılavuzu](/tr/gateway)
