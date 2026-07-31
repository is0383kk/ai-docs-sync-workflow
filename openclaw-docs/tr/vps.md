---
read_when:
    - Gateway'i bir Linux sunucusunda veya bulut VPS'de çalıştırmak istiyorsunuz
    - Barındırma kılavuzlarına hızlı bir genel bakışa ihtiyacınız var
    - OpenClaw için genel Linux sunucu ayarlaması istiyorsunuz
sidebarTitle: Linux Server
summary: OpenClaw'u bir Linux sunucusunda veya bulut VPS'inde çalıştırma — sağlayıcı seçici, mimari ve ince ayar
title: Linux sunucusu
x-i18n:
    generated_at: "2026-07-26T23:41:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 634a246850ab8b854c2c799688fd368ebed3a02124baa85bf38d5ff6ef8cec64
    source_path: vps.md
    workflow: 16
---

OpenClaw Gateway'ini herhangi bir Linux sunucusunda veya bulut VPS'te çalıştırın. Bu sayfa bir
sağlayıcı seçmenize yardımcı olur, bulut dağıtımlarının nasıl çalıştığını açıklar ve her yerde
geçerli olan genel Linux ayarlarını ele alır.

## Bir sağlayıcı seçin

<CardGroup cols={2}>
  <Card title="Azure" href="/tr/install/azure">Linux VM</Card>
  <Card title="DigitalOcean" href="/tr/install/digitalocean">Basit ücretli VPS</Card>
  <Card title="exe.dev" href="/tr/install/exe-dev">HTTPS proxy'li VM</Card>
  <Card title="Fly.io" href="/tr/install/fly">Fly Machines</Card>
  <Card title="GCP" href="/tr/install/gcp">Compute Engine</Card>
  <Card title="Hetzner" href="/tr/install/hetzner">Hetzner VPS üzerinde Docker</Card>
  <Card title="Hostinger" href="/tr/install/hostinger">Tek tıklamayla kurulan VPS</Card>
  <Card title="Northflank" href="/tr/install/northflank">Tek tıklamayla tarayıcıdan kurulum</Card>
  <Card title="Oracle Cloud" href="/tr/install/oracle">Daima Ücretsiz ARM katmanı</Card>
  <Card title="Railway" href="/tr/install/railway">Tek tıklamayla tarayıcıdan kurulum</Card>
  <Card title="Raspberry Pi" href="/tr/install/raspberry-pi">Kendi sunucunuzda barındırılan ARM</Card>
</CardGroup>

**AWS (EC2 / Lightsail / ücretsiz katman)** da iyi çalışır.
Topluluk tarafından hazırlanmış bir videolu anlatıma
[x.com/techfrenAJ/status/2014934471095812547](https://x.com/techfrenAJ/status/2014934471095812547)
adresinden ulaşılabilir (topluluk kaynağıdır -- kullanılamaz hâle gelebilir).

## Bulut kurulumları nasıl çalışır?

- **Gateway VPS üzerinde çalışır** ve durum ile çalışma alanının sahibidir.
- Dizüstü bilgisayarınızdan veya telefonunuzdan **Control UI** ya da **Tailscale/SSH** aracılığıyla bağlanırsınız.
- VPS'yi doğruluk kaynağı olarak kabul edin ve durum ile çalışma alanını düzenli olarak **yedekleyin**.
- Güvenli varsayılan: Gateway'i geri döngü arabiriminde tutun ve SSH tüneli veya Tailscale Serve üzerinden erişin.
  `lan` veya `tailnet` adresine bağlarsanız kimlik doğrulama güvenilir bir
  proxy'ye devredilmediği sürece Gateway, paylaşılan bir gizli anahtar
  (`gateway.auth.token` veya `gateway.auth.password`) gerektirir.

İlgili sayfalar: [Gateway'e uzaktan erişim](/tr/gateway/remote), [Platformlar merkezi](/tr/platforms).

## Önce yönetici erişimini güçlendirin

OpenClaw'ı herkese açık bir VPS'ye kurmadan önce sunucunun kendisini nasıl
yönetmek istediğinize karar verin.

- Yalnızca Tailnet üzerinden yönetici erişimi için: önce Tailscale'i kurun, VPS'yi
  tailnet'inize dahil edin, Tailscale IP'si veya MagicDNS adı üzerinden ikinci bir SSH oturumunu doğrulayın,
  ardından herkese açık SSH erişimini kısıtlayın.
- Tailscale olmadan: daha fazla hizmeti dışarı açmadan önce SSH yolunuz için
  eşdeğer güçlendirmeyi uygulayın.
- Bu, Gateway erişiminden ayrıdır. OpenClaw'ı yine geri döngü arabirimine bağlı tutabilir
  ve pano için bir SSH tüneli veya Tailscale Serve kullanabilirsiniz.

Tailscale'e özgü Gateway seçenekleri [Tailscale](/tr/gateway/tailscale) sayfasındadır.

## VPS üzerinde paylaşılan şirket ajanı

Tüm kullanıcılar aynı güven sınırı içindeyse ve ajan yalnızca iş amaçlıysa
bir ekip için tek bir ajan çalıştırmak geçerli bir kurulumdur.

- Ajanı özel bir çalışma ortamında tutun (VPS/VM/konteyner + özel işletim sistemi kullanıcısı/hesapları).
- Bu çalışma ortamında kişisel Apple/Google hesaplarına veya kişisel tarayıcı/parola yöneticisi profillerine giriş yapmayın.
- Kullanıcılar birbirine karşı kötü niyetli olabilecekse gateway/ana makine/işletim sistemi kullanıcısı bazında ayırın.

Güvenlik modeli ayrıntıları: [Güvenlik](/tr/gateway/security).

## VPS ile Node kullanımı

Gateway'i bulutta tutabilir ve yerel cihazlarınızdaki
(Mac/iOS/Android/başsız) **Node**'ları eşleştirebilirsiniz. Gateway bulutta kalırken Node'lar yerel ekran/kamera/canvas ve `system.run`
yetenekleri sağlar.

Belgeler: [Node'lar](/tr/nodes), [Node CLI](/tr/cli/nodes).

## Küçük VM'ler ve ARM ana makineleri için başlangıç ayarları

Düşük güçlü VM'lerde (veya ARM ana makinelerinde) CLI komutları yavaş geliyorsa Node'un modül derleme önbelleğini etkinleştirin:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF'
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

- `NODE_COMPILE_CACHE`, tekrarlanan komutların başlangıç sürelerini iyileştirir; ilk çalıştırma önbelleği ısıtır.
- `OPENCLAW_NO_RESPAWN=1`, rutin Gateway yeniden başlatmalarını aynı süreç içinde tutar; böylece ek süreç aktarımlarını önler ve küçük ana makinelerde PID takibini basit tutar.
- Raspberry Pi'ye özgü ayrıntılar için [Raspberry Pi](/tr/install/raspberry-pi) bölümüne bakın.

### systemd ayarları kontrol listesi (isteğe bağlı)

`systemd` kullanan VM ana makinelerinde şunları değerlendirin:

- Kararlı bir başlangıç yolu için hizmet ortamı: `OPENCLAW_NO_RESPAWN=1` ve
  `NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache`
- Açık yeniden başlatma davranışı: `Restart=always`, `RestartSec=2`, `TimeoutStartSec=90`
- Rastgele G/Ç kaynaklı soğuk başlangıç gecikmelerini azaltmak için durum/önbellek yollarında SSD destekli diskler.

Standart `openclaw onboard --install-daemon` yolu bir systemd kullanıcı
birimi kurar; birimi şu komutla düzenleyin:

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

Bunun yerine bilinçli olarak bir sistem birimi kurduysanız birimi
`sudo systemctl edit openclaw-gateway.service` aracılığıyla düzenleyin.

`Restart=` ilkeleri otomatik kurtarmaya nasıl yardımcı olur:
[systemd hizmet kurtarmayı otomatikleştirebilir](https://www.redhat.com/en/blog/systemd-automate-recovery).

Linux OOM davranışı, alt süreç kurbanı seçimi ve `exit 137`
tanılamaları için [Linux bellek baskısı ve OOM sonlandırmaları](/tr/platforms/linux#memory-pressure-and-oom-kills) sayfasına bakın.

## İlgili

- [Kuruluma genel bakış](/tr/install)
- [DigitalOcean](/tr/install/digitalocean)
- [Fly.io](/tr/install/fly)
- [Hetzner](/tr/install/hetzner)
