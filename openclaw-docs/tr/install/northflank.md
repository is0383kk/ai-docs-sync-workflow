---
read_when:
    - OpenClaw'u Northflank'e Dağıtma
    - Tarayıcı tabanlı Kontrol Arayüzü ile tek tıklamayla buluta dağıtım istiyorsunuz
summary: OpenClaw'u tek tıklamalı şablonla Northflank üzerinde dağıtın
title: Northflank
x-i18n:
    generated_at: "2026-07-26T22:50:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 16bb96fdf470999e15e163b6227d228ce8b60b9a172eb74cadc87bddd3955957
    source_path: install/northflank.mdx
    workflow: 16
---

OpenClaw'ı tek tıklamalı bir şablonla Northflank üzerinde dağıtın ve web Control UI üzerinden erişin. Bu, en kolay "sunucuda terminal yok" yoludur: Northflank, gateway'i sizin için çalıştırır.

## Başlarken

1. [OpenClaw'ı dağıt](https://northflank.com/stacks/deploy-openclaw) bağlantısına tıklayarak şablonu açın.
2. Henüz hesabınız yoksa [Northflank'te bir hesap](https://app.northflank.com/signup) oluşturun.
3. **Deploy OpenClaw now** seçeneğine tıklayın.
4. Gerekli ortam değişkenini ayarlayın: `OPENCLAW_GATEWAY_TOKEN` (güçlü ve rastgele bir değer kullanın).
5. OpenClaw şablonunu derleyip çalıştırmak için **Deploy stack** seçeneğine tıklayın.
6. Dağıtımın tamamlanmasını bekleyin, ardından **View resources** seçeneğine tıklayın.
7. OpenClaw hizmetini açın.
8. `/openclaw` adresindeki herkese açık OpenClaw URL'sini açın ve yapılandırılmış paylaşılan gizli anahtarı kullanarak bağlanın. Bu şablon varsayılan olarak `OPENCLAW_GATEWAY_TOKEN` kullanır; bunu parola kimlik doğrulamasıyla değiştirirseniz bunun yerine söz konusu parolayı kullanın.

## Neler elde edersiniz?

- Barındırılan OpenClaw Gateway + Control UI
- `openclaw.json`, aracı başına `auth-profiles.json`, kanal/sağlayıcı durumu, oturumlar ve çalışma alanının yeniden dağıtımlardan sonra korunması için Northflank Volume (`/data`) üzerinden kalıcı depolama

## Kanal bağlama

Kanal kurulum talimatları için `/openclaw` adresindeki Control UI'ı kullanın veya SSH üzerinden `openclaw onboard` komutunu çalıştırın:

- [Telegram](/tr/channels/telegram) (en hızlısı, yalnızca bir bot token'ı gerekir)
- [Discord](/tr/channels/discord)
- [Tüm kanallar](/tr/channels)

## Sonraki adımlar

- Mesajlaşma kanallarını ayarlayın: [Kanallar](/tr/channels)
- Gateway'i yapılandırın: [Gateway yapılandırması](/tr/gateway/configuration)
- OpenClaw'ı güncel tutun: [Güncelleme](/tr/install/updating)
