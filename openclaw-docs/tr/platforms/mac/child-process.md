---
read_when:
    - Mac uygulamasını Gateway yaşam döngüsüyle bütünleştirme
summary: macOS'te Gateway yaşam döngüsü (launchd)
title: macOS'ta Gateway yaşam döngüsü
x-i18n:
    generated_at: "2026-07-27T00:04:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89a27334afcecb322feb2732cf6282b4c286ef27828a1b57157f9d4fc161aed6
    source_path: platforms/mac/child-process.md
    workflow: 16
---

macOS uygulaması varsayılan olarak Gateway'i **launchd** aracılığıyla yönetir ve
Gateway'i bir alt süreç olarak başlatmaz. Önce yapılandırılan bağlantı noktasında
zaten çalışmakta olan bir Gateway'e bağlanmayı dener; hiçbirine erişilemiyorsa
harici `openclaw` CLI aracılığıyla launchd hizmetini etkinleştirir (gömülü
çalışma zamanı yoktur). Bu, oturum açıldığında güvenilir otomatik başlatma ve çökmelerden sonra yeniden başlatma sağlar.

Alt süreç modu (Gateway'in doğrudan uygulama tarafından başlatılması) günümüzde
**kullanılmamaktadır**. Kullanıcı arayüzüyle daha sıkı bir bağlantı gerekiyorsa Gateway'i bir
terminalde manuel olarak çalıştırın.

## Varsayılan davranış (launchd)

- Uygulama, `ai.openclaw.gateway` etiketli kullanıcı başına bir LaunchAgent yükler (`--profile`/`OPENCLAW_PROFILE` kullanılırken
  `ai.openclaw.<profile>`).
- Yerel mod etkinleştirildiğinde uygulama, LaunchAgent'ın yüklendiğinden emin olur ve
  gerekirse Gateway'i başlatır.
- Günlükler launchd Gateway günlük yoluna yazılır (Hata Ayıklama Ayarları'nda görülebilir).

Yaygın komutlar:

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

Adlandırılmış bir profil çalıştırırken etiketi `ai.openclaw.<profile>` ile değiştirin.

## İmzasız geliştirme derlemeleri

`scripts/restart-mac.sh --no-sign`, imzalama anahtarları olmadan hızlı yerel derlemeler içindir.
launchd'nin imzasız bir aktarma ikili dosyasını göstermesini önlemek için
`~/.openclaw/disable-launchagent` dosyasını yazar.

`scripts/restart-mac.sh` ile imzalı çalıştırmalar, işaretçi mevcutsa bu geçersiz kılmayı
temizler. Manuel olarak sıfırlamak için:

```bash
rm ~/.openclaw/disable-launchagent
```

## Yalnızca bağlanma modu

macOS uygulamasını launchd'yi hiçbir zaman yüklememeye veya yönetmemeye zorlamak için
uygulamayı `--attach-only` (veya `--no-launchd`) ile başlatın. Bu,
`~/.openclaw/disable-launchagent` ayarını yapar; böylece uygulama yalnızca zaten
çalışmakta olan bir Gateway'e bağlanır. Aynı davranışı Hata Ayıklama Ayarları'ndan açıp kapatın.

## Uzak mod

Uzak mod hiçbir zaman yerel bir Gateway başlatmaz. Uygulama uzak ana makineye bir SSH tüneli
kullanır ve bu tünel üzerinden bağlanır.

## Neden launchd'yi tercih ediyoruz?

- Oturum açıldığında otomatik başlatma.
- Yerleşik yeniden başlatma/KeepAlive semantiği.
- Öngörülebilir günlükler ve denetim.

Gerçek bir alt süreç moduna yeniden ihtiyaç duyulursa bu, ayrı ve açıkça yalnızca geliştirmeye
özel bir mod olarak belgelenmelidir.

## İlgili

- [macOS uygulaması](/tr/platforms/macos)
- [Gateway çalışma kılavuzu](/tr/gateway)
