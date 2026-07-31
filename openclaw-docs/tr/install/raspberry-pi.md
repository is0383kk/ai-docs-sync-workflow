---
read_when:
    - OpenClaw'u Raspberry Pi üzerinde kurma
    - OpenClaw'u ARM cihazlarda çalıştırma
    - Ucuz, her zaman açık kişisel bir yapay zekâ oluşturma
summary: Sürekli açık, kendi kendine barındırma için OpenClaw'u bir Raspberry Pi üzerinde barındırın
title: Raspberry Pi
x-i18n:
    generated_at: "2026-07-27T00:03:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 60f8f3b23577155658d410993937ebe7c34c21f71c1bd7d9b0c453f15c4aa024
    source_path: install/raspberry-pi.md
    workflow: 16
---

Kalıcı ve her zaman açık bir OpenClaw Gateway'i Raspberry Pi üzerinde çalıştırın. Pi yalnızca ağ geçidi olduğundan (modeller API aracılığıyla bulutta çalışır), mütevazı bir Pi bile iş yükünü rahatlıkla kaldırır -- tipik donanım maliyeti **tek seferlik $35-80** tutarındadır ve aylık ücret yoktur.

## Donanım uyumluluğu

| Pi modeli   | RAM    | Çalışır mı? | Notlar                                      |
| ----------- | ------ | ----------- | ------------------------------------------- |
| Pi 5        | 4/8 GB | En iyi     | En hızlı seçenek, önerilir.                 |
| Pi 4        | 4 GB   | İyi         | Çoğu kullanıcı için ideal denge.            |
| Pi 4        | 2 GB   | Yeterli     | Takas alanı ekleyin.                         |
| Pi 4        | 1 GB   | Kısıtlı     | Takas alanı ve asgari yapılandırmayla mümkün. |
| Pi 3B+      | 1 GB   | Yavaş       | Çalışır ancak ağırdır.                      |
| Pi Zero 2 W | 512 MB | Hayır       | Önerilmez.                                  |

**Asgari:** 1 GB RAM, 1 çekirdek, 500 MB boş disk alanı, 64 bit işletim sistemi.
**Önerilen:** 2 GB+ RAM, 16 GB+ SD kart (veya USB SSD), Ethernet.

## Ön koşullar

- 2 GB+ RAM'e sahip Raspberry Pi 4 veya 5 (4 GB önerilir)
- MicroSD kart (16 GB+) veya USB SSD (daha iyi performans)
- Resmî Pi güç kaynağı
- Ağ bağlantısı (Ethernet veya WiFi)
- 64 bit Raspberry Pi OS (zorunludur -- 32 bit kullanmayın)
- Yaklaşık 30 dakika

## Kurulum

<Steps>
  <Step title="İşletim sistemini yazdırın">
    **Raspberry Pi OS Lite (64-bit)** kullanın -- başsız bir sunucu için masaüstü gerekmez.

    1. [Raspberry Pi Imager](https://www.raspberrypi.com/software/) uygulamasını indirin.
    2. İşletim sistemini seçin: **Raspberry Pi OS Lite (64-bit)**.
    3. Ayarlar iletişim kutusunda şunları önceden yapılandırın:
       - Ana makine adı: `gateway-host`
       - SSH'yi etkinleştirin
       - Kullanıcı adı ve parola belirleyin
       - WiFi'yi yapılandırın (Ethernet kullanmıyorsanız)
    4. SD kartınıza veya USB sürücünüze yazdırın, sürücüyü takın ve Pi'yi başlatın.

  </Step>

  <Step title="SSH üzerinden bağlanın">
    ```bash
    ssh user@gateway-host
    ```
  </Step>

  <Step title="Sistemi güncelleyin">
    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install -y git curl build-essential

    # Saat dilimini ayarlayın (cron ve anımsatıcılar için önemlidir)
    sudo timedatectl set-timezone America/Chicago
    ```

  </Step>

  <Step title="Node.js 24'ü yükleyin">
    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt install -y nodejs
    node --version
    ```
  </Step>

  <Step title="Takas alanı ekleyin (2 GB veya daha azı için önemlidir)">
    ```bash
    sudo fallocate -l 2G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

    # Düşük RAM'li cihazlar için takas eğilimini azaltın
    echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
    sudo sysctl -p
    ```

  </Step>

  <Step title="OpenClaw'u yükleyin">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```
  </Step>

  <Step title="İlk kurulumu çalıştırın">
    ```bash
    openclaw onboard --install-daemon
    ```

    Sihirbazı izleyin. Başsız cihazlarda OAuth yerine API anahtarları önerilir. Başlamak için en kolay kanal Telegram'dır.

  </Step>

  <Step title="Doğrulayın">
    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```
  </Step>

  <Step title="Kontrol kullanıcı arayüzüne erişin">
    Bilgisayarınızdan Pi üzerindeki kontrol paneli URL'sini alın:

    ```bash
    ssh user@gateway-host 'openclaw dashboard --no-open'
    ```

    Ardından başka bir terminalde SSH tüneli oluşturun:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
    ```

    Yazdırılan URL'yi yerel tarayıcınızda açın. Her zaman açık uzaktan erişim için [Tailscale entegrasyonu](/tr/gateway/tailscale) bölümüne bakın.

  </Step>
</Steps>

## Performans ipuçları

**USB SSD kullanın** -- SD kartlar yavaştır ve zamanla aşınır. USB SSD performansı önemli ölçüde artırır ve daha fazla yazma döngüsüne dayanır; işletim sistemini SD kartta tutuyorsanız `OPENCLAW_STATE_DIR` için SSD'yi kullanın. [Pi USB önyükleme kılavuzuna](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#usb-mass-storage-boot) bakın.

**Modül derleme önbelleğini etkinleştirin** -- Düşük güçlü Pi ana makinelerinde yinelenen CLI çağrılarını hızlandırır. `OPENCLAW_NO_RESPAWN=1`, rutin Gateway yeniden başlatmalarını işlem içinde tutarak ek süreç aktarımlarını önler ve küçük ana makinelerde PID takibini basit tutar:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF' # pragma: allowlist secret
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

`/tmp` değil, `/var/tmp` kullanın -- bazı dağıtımlar önyükleme sırasında `/tmp` öğesini temizleyerek hazırlanmış önbelleği siler.

**Bellek kullanımını azaltın** -- Başsız kurulumlarda GPU belleğini serbest bırakın ve kullanılmayan hizmetleri devre dışı bırakın:

```bash
echo 'gpu_mem=16' | sudo tee -a /boot/config.txt
sudo systemctl disable bluetooth
```

**Kararlı yeniden başlatmalar için systemd ek yapılandırması** -- Bu Pi çoğunlukla OpenClaw çalıştırıyorsa bir hizmet ek yapılandırması ekleyin:

```bash
systemctl --user edit openclaw-gateway.service
```

```ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90
```

Ardından `systemctl --user daemon-reload && systemctl --user restart openclaw-gateway.service`. Başsız bir Pi'de, kullanıcı hizmetinin oturum kapatıldıktan sonra çalışmayı sürdürmesi için kalıcılığı bir kez etkinleştirin: `sudo loginctl enable-linger "$(whoami)"`.

## Önerilen model kurulumu

Pi yalnızca Gateway'i çalıştırdığından bulutta barındırılan API modellerini kullanın -- yerel LLM'leri Pi üzerinde çalıştırmayın; küçük modeller bile kullanışlı olamayacak kadar yavaştır:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-6",
        "fallbacks": ["openai/gpt-5.4-mini"]
      }
    }
  }
}
```

## ARM ikili dosya notları

OpenClaw özelliklerinin çoğu ARM64 üzerinde değişiklik yapılmadan çalışır (Node.js, Telegram, WhatsApp/Baileys, Chromium). Zaman zaman ARM derlemeleri bulunmayan ikili dosyalar genellikle Skills tarafından sunulan isteğe bağlı Go/Rust CLI araçlarıdır. Mimarinin `uname -m` ile doğrulayın (`aarch64` göstermelidir), ardından kaynaktan derlemeye başvurmadan önce eksik ikili dosyanın sürüm sayfasında `linux-arm64` / `aarch64` yapıtlarını kontrol edin.

## Kalıcılık ve yedeklemeler

OpenClaw durumu şurada bulunur:

- `~/.openclaw/` -- `openclaw.json`, aracı başına `auth-profiles.json`, kanal/sağlayıcı durumu, oturumlar.
- `~/.openclaw/workspace/` -- aracı çalışma alanı (SOUL.md, bellek, yapıtlar).

Bunlar yeniden başlatmalardan etkilenmez ve hem performans hem de kullanım ömrü açısından SD kart yerine SSD kullanılmasından yarar görür. Şu komutla taşınabilir bir anlık görüntü alın:

```bash
openclaw backup create
```

## Sorun giderme

**Bellek yetersizliği** -- Takas alanının etkin olduğunu `free -h` ile doğrulayın. Kullanılmayan hizmetleri devre dışı bırakın (`sudo systemctl disable cups bluetooth avahi-daemon`). Yalnızca API tabanlı modeller kullanın.

**Yavaş performans** -- SD kart yerine USB SSD kullanın. CPU hız kısıtlamasını `vcgencmd get_throttled` ile kontrol edin (`0x0` döndürmelidir).

**Hizmet başlamıyor** -- Günlükleri `journalctl --user -u openclaw-gateway.service --no-pager -n 100` ile kontrol edin ve `openclaw doctor --non-interactive` komutunu çalıştırın. Bu başsız bir Pi ise kalıcılığın etkin olduğunu da doğrulayın: `sudo loginctl enable-linger "$(whoami)"`.

**ARM ikili dosya sorunları** -- Bir skill "exec format error" hatasıyla başarısız olursa ikili dosyanın ARM64 derlemesi olup olmadığını kontrol edin. Mimarinin `uname -m` ile doğrulayın (`aarch64` göstermelidir).

**WiFi bağlantısı kesiliyor** -- WiFi güç yönetimini devre dışı bırakın: `sudo iwconfig wlan0 power off`.

## Sonraki adımlar

- [Kanallar](/tr/channels) -- Telegram, WhatsApp, Discord ve daha fazlasını bağlayın
- [Gateway yapılandırması](/tr/gateway/configuration) -- tüm yapılandırma seçenekleri
- [Güncelleme](/tr/install/updating) -- OpenClaw'u güncel tutun

## İlgili konular

- [Kuruluma genel bakış](/tr/install)
- [Linux sunucusu](/tr/vps)
- [Platformlar](/tr/platforms)
