---
read_when: Browser control fails on Linux, especially with snap Chromium
summary: Linux'ta OpenClaw tarayıcı denetimi için Chrome/Brave/Edge/Chromium CDP başlatma sorunlarını düzeltin
title: Tarayıcı sorunlarını giderme
x-i18n:
    generated_at: "2026-07-27T00:19:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5db2da2d43129862f0c005213df828f6eae81f5561e57d41795ea90787822a
    source_path: tools/browser-linux-troubleshooting.md
    workflow: 16
---

## Sorun: 18800 numaralı bağlantı noktasında Chrome CDP başlatılamadı

```json
{ "error": "Hata: \"openclaw\" profili için 18800 numaralı bağlantı noktasında Chrome CDP başlatılamadı." }
```

### Temel neden

Ubuntu ve çoğu Linux dağıtımında `apt install chromium` gerçek bir tarayıcı değil, bir snap
sarmalayıcısı yükler:

```text
Not: 'chromium' yerine 'chromium-browser' seçiliyor
chromium-browser zaten en yeni sürümde (2:1snap1-0ubuntu2).
```

Snap'in AppArmor kısıtlaması, OpenClaw'un tarayıcı işlemini başlatma ve izleme
biçimine müdahale eder.

Linux'ta sık görülen diğer başlatma hataları:

- `The profile appears to be in use by another Chromium process`: yönetilen profil dizinindeki eski
  `Singleton*` kilit dosyaları. Kilit, sonlanmış veya
  farklı bir ana makinedeki işleme işaret ettiğinde OpenClaw bu kilitleri kaldırır
  ve bir kez yeniden dener.
- `Missing X server or $DISPLAY`: masaüstü oturumu olmayan bir ana makinede görünür bir tarayıcı açıkça istendi.
  Hem `DISPLAY` hem de `WAYLAND_DISPLAY` ayarlanmamışsa yerel yönetilen profiller Linux'ta
  başsız moda geri döner. `OPENCLAW_BROWSER_HEADLESS=0`, `browser.headless: false` veya
  `browser.profiles.<name>.headless: false` ayarladıysanız bu görünür mod geçersiz kılmasını kaldırın,
  `OPENCLAW_BROWSER_HEADLESS=1` ayarlayın, `Xvfb` başlatın, tek seferlik
  yönetilen başlatma için `openclaw browser start --headless` çalıştırın veya OpenClaw'u
  gerçek bir masaüstü oturumunda çalıştırın.

### Çözüm 1: Google Chrome'u yükleyin (önerilen)

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y  # bağımlılık hataları varsa
```

`~/.openclaw/openclaw.json` öğesini güncelleyin:

```json
{
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  }
}
```

### Çözüm 2: snap Chromium'u yalnızca bağlanma modunda kullanın

Snap Chromium'u kullanmaya devam etmeniz gerekiyorsa OpenClaw'u tarayıcıyı
başlatmak yerine elle başlatılmış bir tarayıcıya bağlanacak şekilde yapılandırın:

```json
{
  "browser": {
    "enabled": true,
    "attachOnly": true,
    "headless": true,
    "noSandbox": true
  }
}
```

Chromium'u elle başlatın:

```bash
chromium-browser --headless --no-sandbox --disable-gpu \
  --remote-debugging-port=18800 \
  --user-data-dir=$HOME/.openclaw/browser/openclaw/user-data \
  about:blank &
```

İsteğe bağlı olarak bir systemd kullanıcı hizmetiyle otomatik olarak başlatın:

```ini
# ~/.config/systemd/user/openclaw-browser.service
[Unit]
Description=OpenClaw Tarayıcısı (Chrome CDP)
After=network.target

[Service]
ExecStart=/snap/bin/chromium --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --user-data-dir=%h/.openclaw/browser/openclaw/user-data about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now openclaw-browser.service
```

### Tarayıcının çalıştığını doğrulayın

```bash
curl -s http://127.0.0.1:18791/ | jq '{running, pid, chosenBrowser}'
curl -s -X POST http://127.0.0.1:18791/start
curl -s http://127.0.0.1:18791/tabs
```

### Yapılandırma referansı

| Seçenek                      | Açıklama                                                          | Varsayılan                                                            |
| --------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `browser.enabled`           | Tarayıcı denetimini etkinleştir                                               | `true`                                                             |
| `browser.executablePath`    | Chromium tabanlı tarayıcı ikili dosyasının yolu (Chrome/Brave/Edge/Chromium) | otomatik algılanır (Chromium tabanlıysa işletim sisteminin varsayılan tarayıcısı tercih edilir) |
| `browser.headless`          | Grafiksel kullanıcı arayüzü olmadan çalıştır                                                      | `false`                                                            |
| `OPENCLAW_BROWSER_HEADLESS` | Yerel yönetilen tarayıcının başsız modu için işlem başına geçersiz kılma         | ayarlanmamış                                                              |
| `browser.noSandbox`         | `--no-sandbox` bayrağını ekle (bazı Linux kurulumlarında gereklidir)               | `false`                                                            |
| `browser.attachOnly`        | Tarayıcı başlatma; yalnızca mevcut bir tarayıcıya bağlan              | `false`                                                            |

Raspberry Pi, eski VPS ana makineleri veya yavaş depolama sistemlerinde Chrome'un
CDP HTTP uç noktasını sunması ya da hazır hâle gelmesi yönetilen tarayıcı zaman
sınırının izin verdiğinden daha uzun sürüyorsa `attachOnly` ile elle başlatılmış
bir tarayıcı kullanın.

### Sorun: profile="user" için Chrome sekmesi bulunamadı

`user` (`existing-session` / Chrome MCP) profilini kullanıyorsunuz ve bağlanılabilecek
açık sekme yok.

Düzeltme seçenekleri:

1. Bunun yerine yönetilen tarayıcıyı kullanın:
   `openclaw browser --browser-profile openclaw start` (veya
   `browser.defaultProfile: "openclaw"` ayarlayın).
2. Yerel Chrome'u en az bir açık sekmeyle çalışır durumda tutun, ardından
   `--browser-profile user` ile yeniden deneyin.

Notlar:

- `user` yalnızca ana makinede çalışır. Linux sunucularında, konteynerlerde veya uzak ana makinelerde
  bunun yerine CDP profillerini tercih edin.
- `user` ve diğer `existing-session` profilleri mevcut Chrome MCP
  sınırlarını paylaşır: yalnızca referans odaklı eylemler, yükleme başına bir dosya,
  iletişim kutusu `timeoutMs` geçersiz kılmaları yok, `wait --load networkidle` yok ve
  `responsebody`, PDF dışa aktarma, indirme müdahalesi veya toplu eylemler yok.
- Yerel `openclaw` sürücü profilleri `cdpPort`/`cdpUrl` değerlerini otomatik olarak atar;
  bunları yalnızca uzak CDP için elle ayarlayın.
- Uzak CDP profilleri `http://`, `https://`, `ws://` ve `wss://` kabul eder.
  `/json/version` keşfi için HTTP(S) veya tarayıcı hizmetiniz size doğrudan
  DevTools yuva URL'si sağlıyorsa WS(S) kullanın.

## İlgili

- [Tarayıcı](/tr/tools/browser)
- [Tarayıcıda oturum açma](/tr/tools/browser-login)
- [Tarayıcı WSL2 sorun giderme](/tr/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
