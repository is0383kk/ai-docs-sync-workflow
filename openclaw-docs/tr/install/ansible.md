---
read_when:
    - Güvenlik sıkılaştırmasıyla otomatik sunucu dağıtımı istiyorsunuz
    - VPN erişimiyle güvenlik duvarı tarafından yalıtılmış bir kurulum gerekir
    - Uzak Debian/Ubuntu sunucularına dağıtım yapıyorsunuz
summary: Ansible, Tailscale VPN ve güvenlik duvarı yalıtımıyla otomatikleştirilmiş, güçlendirilmiş OpenClaw kurulumu
title: Ansible
x-i18n:
    generated_at: "2026-07-26T22:48:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2f6b473cd5a8b80389b5ed746c4e2f2729d95bb15a2daaaa183fbdfbe144e647
    source_path: install/ansible.md
    workflow: 16
---

OpenClaw'u, güvenliği önceliklendiren bir mimariye sahip otomatik yükleyici **[openclaw-ansible](https://github.com/openclaw/openclaw-ansible)** ile üretim sunucularına dağıtın.

<Info>
Ansible dağıtımı için esas kaynak [openclaw-ansible](https://github.com/openclaw/openclaw-ansible) deposudur. Bu sayfa kısa bir genel bakış sunar.
</Info>

## Ön koşullar

| Gereksinim | Ayrıntılar                                                |
| ----------- | --------------------------------------------------------- |
| İşletim sistemi | Debian 11+ veya Ubuntu 20.04+                          |
| Erişim      | Root veya sudo ayrıcalıkları                              |
| Ağ          | Paket kurulumu için internet bağlantısı                   |
| Ansible     | 2.14+ (hızlı başlangıç betiği tarafından otomatik yüklenir) |

## Sağlananlar

- Güvenlik duvarını önceliklendiren güvenlik: UFW + Docker yalıtımı (yalnızca SSH + Tailscale erişilebilir)
- Hizmetleri herkese açık hâle getirmeden uzaktan erişim için Tailscale VPN
- Yalnızca localhost bağlamalarıyla yalıtılmış korumalı alan konteynerleri için Docker
- Güçlendirme ve önyüklemede otomatik başlatma özellikli systemd entegrasyonu
- Tek komutla kurulum

## Hızlı başlangıç

```bash
curl -fsSL https://raw.githubusercontent.com/openclaw/openclaw-ansible/main/install.sh | bash
```

## Yüklenenler

1. Tailscale (güvenli uzaktan erişim için örgü VPN)
2. UFW güvenlik duvarı (yalnızca SSH + Tailscale portları)
3. Docker CE + Compose V2 (varsayılan ajan korumalı alan arka ucu)
4. Node.js ve pnpm (OpenClaw için Node 22.22.3+, 24.15+ veya 25.9+ gerekir; Node 24 önerilir)
5. Konteynerleştirilmeden, ana makine tabanlı olarak yüklenen OpenClaw
6. Güvenlik güçlendirmeli bir systemd hizmeti

<Note>
Gateway, Docker içinde değil, doğrudan ana makinede çalışır. Ajan korumalı alanı
isteğe bağlıdır; bu playbook, varsayılan korumalı alan arka ucu olduğu için Docker'ı
yükler. Diğer arka uçlar için [Korumalı Alan](/tr/gateway/sandboxing) bölümüne bakın.
</Note>

## Kurulum sonrası yapılandırma

<Steps>
  <Step title="openclaw kullanıcısına geçin">
    ```bash
    sudo -i -u openclaw
    ```
  </Step>
  <Step title="İlk kullanım sihirbazını çalıştırın">
    Kurulum sonrası betik, OpenClaw yapılandırması boyunca size rehberlik eder.
  </Step>
  <Step title="Mesajlaşma kanallarını bağlayın">
    WhatsApp, Telegram, Discord veya Signal'de oturum açın:
    ```bash
    openclaw channels login --channel <name>
    ```
  </Step>
  <Step title="Kurulumu doğrulayın">
    ```bash
    sudo systemctl status openclaw
    sudo journalctl -u openclaw -f
    ```
  </Step>
  <Step title="Tailscale'e bağlanın">
    Güvenli uzaktan erişim için VPN örgünüze katılın.
  </Step>
</Steps>

### Hızlı komutlar

```bash
# Hizmet durumunu denetleyin
sudo systemctl status openclaw

# Canlı günlükleri görüntüleyin
sudo journalctl -u openclaw -f

# Gateway'i yeniden başlatın
sudo systemctl restart openclaw

# Kanal oturumu açın (openclaw kullanıcısı olarak çalıştırın)
sudo -i -u openclaw
openclaw channels login --channel <name>
```

## Güvenlik mimarisi

Dört katmanlı savunma modeli:

1. Güvenlik duvarı (UFW): yalnızca SSH (22) ve Tailscale (41641/udp) herkese açıktır
2. VPN (Tailscale): Gateway'e yalnızca VPN örgüsü üzerinden erişilebilir
3. Docker yalıtımı: `DOCKER-USER` iptables zinciri, portların dışarıya açılmasını önler
4. Systemd güçlendirmesi: `NoNewPrivileges`, `PrivateTmp`, ayrıcalıksız kullanıcı

Harici saldırı yüzeyinizi doğrulayın:

```bash
nmap -p- YOUR_SERVER_IP
```

Yalnızca 22 numaralı port (SSH) açık olmalıdır. Gateway ve Docker erişime kapalı kalır.

Docker, Gateway'i çalıştırmak için değil, ajan korumalı alanları (yalıtılmış araç yürütme) için yüklenir. Korumalı alan yapılandırması için [Çok Ajanlı Korumalı Alan ve Araçlar](/tr/tools/multi-agent-sandbox-tools) bölümüne bakın.

## Manuel kurulum

<Steps>
  <Step title="Ön koşulları yükleyin">
    ```bash
    sudo apt update && sudo apt install -y ansible git
    ```
  </Step>
  <Step title="Depoyu klonlayın">
    ```bash
    git clone https://github.com/openclaw/openclaw-ansible.git
    cd openclaw-ansible
    ```
  </Step>
  <Step title="Ansible koleksiyonlarını yükleyin">
    ```bash
    ansible-galaxy collection install -r requirements.yml
    ```
  </Step>
  <Step title="Playbook'u çalıştırın">
    ```bash
    ./run-playbook.sh
    ```

    Alternatif olarak playbook'u doğrudan çalıştırın, ardından kurulum betiğini manuel olarak çalıştırın:
    ```bash
    ansible-playbook playbook.yml --ask-become-pass
    # Ardından çalıştırın: /tmp/openclaw-setup.sh
    ```

  </Step>
</Steps>

## Güncelleme

Ansible yükleyicisi, OpenClaw'u manuel güncellemelere uygun biçimde kurar; standart akış için [Güncelleme](/tr/install/updating) bölümüne bakın.

Playbook'u yeniden çalıştırmak için (örneğin yapılandırma değişikliklerinden sonra):

```bash
cd openclaw-ansible
./run-playbook.sh
```

Bu işlem eş etkisizdir ve birden çok kez güvenle çalıştırılabilir.

## Sorun giderme

<AccordionGroup>
  <Accordion title="Güvenlik duvarı bağlantımı engelliyor">
    - Önce Tailscale VPN üzerinden bağlanın; Gateway tasarım gereği yalnızca bu yolla erişilebilir.
    - SSH'ye (port 22) her zaman izin verilir.

  </Accordion>
  <Accordion title="Hizmet başlatılamıyor">
    ```bash
    # Günlükleri denetleyin
    sudo journalctl -u openclaw -n 100

    # İzinleri doğrulayın
    sudo ls -la /opt/openclaw

    # Manuel başlatmayı test edin
    sudo -i -u openclaw
    cd ~/openclaw
    openclaw gateway run
    ```

  </Accordion>
  <Accordion title="Docker korumalı alan sorunları">
    ```bash
    # Docker'ın çalıştığını doğrulayın
    sudo systemctl status docker

    # Korumalı alan imajını denetleyin
    sudo docker images | grep openclaw-sandbox

    # Eksikse korumalı alan imajını oluşturun (kaynak kod çalışma kopyası gerektirir)
    cd /opt/openclaw/openclaw
    sudo -u openclaw ./scripts/sandbox-setup.sh
    # Kaynak kod çalışma kopyası olmadan yapılan npm kurulumları için bkz.
    # https://docs.openclaw.ai/gateway/sandboxing#images-and-setup
    ```

  </Accordion>
  <Accordion title="Kanal oturumu açılamıyor">
    `openclaw` kullanıcısı olarak çalıştırdığınızdan emin olun:
    ```bash
    sudo -i -u openclaw
    openclaw channels login --channel <name>
    ```
  </Accordion>
</AccordionGroup>

## Gelişmiş yapılandırma

Ayrıntılı güvenlik mimarisi ve sorun giderme bilgileri için openclaw-ansible deposuna bakın:

- [Güvenlik Mimarisi](https://github.com/openclaw/openclaw-ansible/blob/main/docs/security.md)
- [Teknik Ayrıntılar](https://github.com/openclaw/openclaw-ansible/blob/main/docs/architecture.md)
- [Sorun Giderme Kılavuzu](https://github.com/openclaw/openclaw-ansible/blob/main/docs/troubleshooting.md)

## İlgili

- [openclaw-ansible](https://github.com/openclaw/openclaw-ansible): tam dağıtım kılavuzu
- [Docker](/tr/install/docker): konteynerleştirilmiş Gateway kurulumu
- [Korumalı Alan](/tr/gateway/sandboxing): ajan korumalı alanı yapılandırması
- [Çok Ajanlı Korumalı Alan ve Araçlar](/tr/tools/multi-agent-sandbox-tools): ajan başına yalıtım
