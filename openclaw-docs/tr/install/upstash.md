---
read_when:
    - OpenClaw'u Upstash Box'a Dağıtma
    - OpenClaw için SSH tüneli üzerinden pano erişimi sunan yönetilen bir Linux ortamı istiyorsunuz
summary: OpenClaw'u keep-alive ve SSH tüneli erişimiyle Upstash Box'ta barındırma
title: Upstash Box
x-i18n:
    generated_at: "2026-07-26T23:26:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 29232c43e0e4940b7445ab8896c9ccd3e81d0fdbdd522d7f50cb8c8057ac18f0
    source_path: install/upstash.md
    workflow: 16
---

Kalıcı çalışma döngüsü desteğine sahip, yönetilen bir Linux ortamı olan Upstash Box üzerinde kalıcı bir OpenClaw Gateway çalıştırın.

Kontrol paneline erişmek için bir SSH tüneli kullanın. Gateway bağlantı noktasını doğrudan genel internete açmayın.

## Ön koşullar

- Upstash hesabı
- Kalıcı çalışma döngülü Upstash Box
- Yerel makinenizde SSH istemcisi

## Box oluşturma

Upstash Console'da kalıcı çalışma döngülü bir Box oluşturun. Box kimliğini (örneğin
`right-flamingo-14486`) ve Box API anahtarınızı not edin.

Upstash, güncel OpenClaw Box kılavuzunu
[OpenClaw Kurulumu](https://upstash.com/docs/box/guides/openclaw-setup) sayfasında tutar.

## SSH tüneliyle bağlanma

OpenClaw kontrol paneli bağlantı noktasını yerel makinenize yönlendirin. İstendiğinde
SSH parolası olarak Box API anahtarınızı kullanın:

```bash
ssh -o ServerAliveInterval=15 -o ServerAliveCountMax=3 -L 18789:127.0.0.1:18789 <box-id>@us-east-1.box.upstash.com
```

Kalıcı bağlantı seçenekleri, ilk katılım sırasında tünelin boşta kalma nedeniyle kopmasını azaltır.

## OpenClaw'ı yükleme

Box içinde:

```bash
sudo npm install -g openclaw
```

## İlk katılımı çalıştırma

```bash
openclaw onboard --install-daemon
```

İstemleri izleyin. İlk katılım tamamlandığında kontrol paneli URL'sini ve token'ı kopyalayın.

## Gateway'i başlatma

Gateway'i Box ağı için yapılandırın ve arka planda başlatın:

```bash
openclaw config set gateway.bind lan
nohup openclaw gateway > gateway.log 2>&1 &
```

SSH tüneli etkinken kontrol paneli URL'sini yerel olarak açın:

```text
http://127.0.0.1:18789/#token=<your-token>
```

## Otomatik yeniden başlatma

Box başlatıldığında Gateway'in yeniden başlaması için bu komutu Box başlangıç betiği olarak ayarlayın:

```bash
nohup openclaw gateway > gateway.log 2>&1 &
```

## Sorun giderme

İlk katılım sırasında SSH donarsa temiz bir SSH yapılandırması ve kalıcı bağlantı seçenekleriyle yeniden bağlanın:

```bash
ssh -F /dev/null -o ControlMaster=no -o ServerAliveInterval=15 -o ServerAliveCountMax=3 -L 18789:127.0.0.1:18789 <box-id>@us-east-1.box.upstash.com
```

Bu işlem, eski yerel `~/.ssh/config` ayarlarını atlar ve ağın boşta kaldığı dönemlerde tüneli etkin tutar.

## İlgili

- [Uzaktan erişim](/tr/gateway/remote)
- [Gateway güvenliği](/tr/gateway/security)
- [OpenClaw'ı güncelleme](/tr/install/updating)
