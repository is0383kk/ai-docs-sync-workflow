---
read_when:
    - OpenClaw'ı sık sık Docker ile çalıştırıyorsunuz ve günlük kullanımda daha kısa komutlar istiyorsunuz
    - Pano, günlükler, token kurulumu ve eşleştirme akışları için bir yardımcı katman istiyorsunuz
summary: Docker tabanlı OpenClaw kurulumları için ClawDock kabuk yardımcıları
title: ClawDock
x-i18n:
    generated_at: "2026-07-26T23:22:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb829a3301178503f910931e86a39f7befeaf186044f4088a25dc80ea99130d
    source_path: install/clawdock.md
    workflow: 16
---

ClawDock, Docker tabanlı OpenClaw kurulumları için küçük bir kabuk yardımcı katmanıdır.

Daha uzun `docker compose ...` çağrıları yerine `clawdock-start`, `clawdock-dashboard` ve `clawdock-fix-token` gibi kısa komutlar sağlar.

Docker'ı henüz kurmadıysanız [Docker](/tr/install/docker) ile başlayın.

## Kurulum

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

ClawDock'u daha önce `scripts/shell-helpers/clawdock-helpers.sh` konumundan yüklediyseniz geçerli `scripts/clawdock/clawdock-helpers.sh` yolundan yeniden yükleyin; eski ham GitHub yolu kaldırıldı.

Yardımcılar ilk kullanımda OpenClaw çalışma kopyanızı otomatik olarak algılar (`~/openclaw`, `~/projects/openclaw` gibi yaygın yolları denetler) ve sonucu `~/.clawdock/config` içinde önbelleğe alır. Çalışma kopyanız başka bir yerdeyse `CLAWDOCK_DIR` değerini kendiniz ayarlayın.

## Sağlananlar

### Temel işlemler

| Komut              | Açıklama                     |
| ------------------ | ---------------------------- |
| `clawdock-start` | Gateway'i başlat             |
| `clawdock-stop` | Gateway'i durdur             |
| `clawdock-restart` | Gateway'i yeniden başlat     |
| `clawdock-status` | Konteyner durumunu denetle   |
| `clawdock-logs` | Gateway günlüklerini takip et |

### Konteyner erişimi

| Komut                   | Açıklama                                         |
| ----------------------- | ------------------------------------------------ |
| `clawdock-shell`      | Gateway konteynerinin içinde bir kabuk aç        |
| `clawdock-cli <command>`      | OpenClaw CLI komutlarını Docker içinde çalıştır  |
| `clawdock-exec <command>`      | Konteynerde rastgele bir komut yürüt             |

### Web arayüzü ve eşleştirme

| Komut                   | Açıklama                              |
| ----------------------- | ------------------------------------- |
| `clawdock-dashboard`      | Denetim Arayüzü URL'sini aç           |
| `clawdock-devices`      | Bekleyen cihaz eşleştirmelerini listele |
| `clawdock-approve <id>`      | Bir eşleştirme isteğini onayla        |

### Kurulum ve bakım

| Komut                   | Açıklama                                          |
| ----------------------- | ------------------------------------------------- |
| `clawdock-fix-token`      | Gateway tokenini konteyner yapılandırmasına yaz   |
| `clawdock-update`      | Çek, yeniden oluştur ve yeniden başlat            |
| `clawdock-rebuild`      | Yalnızca Docker imajını yeniden oluştur           |
| `clawdock-clean`      | Konteynerleri ve birimleri kaldır                 |

### Yardımcı araçlar

| Komut                   | Açıklama                                      |
| ----------------------- | --------------------------------------------- |
| `clawdock-health`      | Gateway sistem durumu denetimi çalıştır       |
| `clawdock-token`      | Gateway tokenini yazdır                       |
| `clawdock-cd`      | OpenClaw proje dizinine geç                   |
| `clawdock-config`      | `~/.openclaw` dosyasını aç               |
| `clawdock-show-config`      | Yapılandırma dosyalarını gizlenmiş değerlerle yazdır |
| `clawdock-workspace`      | Çalışma alanı dizinini aç                     |
| `clawdock-help`      | Tüm ClawDock komutlarını listele              |

## İlk kullanım akışı

```bash
clawdock-start
clawdock-fix-token
clawdock-dashboard
```

Tarayıcı eşleştirmenin gerekli olduğunu bildirirse:

```bash
clawdock-devices
clawdock-approve <request-id>
```

## Yapılandırma ve gizli bilgiler

ClawDock, [Docker](/tr/install/docker) sayfasında açıklanan ayrıma uygun olarak iki ayrı `.env` dosyasını okur:

- `docker-compose.yml` dosyasının yanındaki proje `.env`: imaj adı, portlar ve `OPENCLAW_GATEWAY_TOKEN` gibi Docker'a özgü değerler. `clawdock-token` tokeni buradan okur.
- `~/.openclaw/.env` (konteynere bağlanır): `openclaw.json` ve `agents/<agentId>/agent/auth-profiles.json` ile birlikte OpenClaw'un yönettiği ortam değişkeni destekli gizli bilgiler.

`clawdock-fix-token`, tokeni proje `.env` dosyasından konteynerin `gateway.remote.token` ve `gateway.auth.token` yapılandırma değerlerine kopyalar ve Gateway'i yeniden başlatır.

`openclaw.json` ile her iki `.env` dosyasını hızlıca incelemek için `clawdock-show-config` komutunu kullanın; yazdırılan çıktıda `.env` değerlerini gizler.

## İlgili

<CardGroup cols={2}>
  <Card title="Docker" href="/tr/install/docker" icon="docker">
    OpenClaw için standart Docker kurulumu.
  </Card>
  <Card title="Docker VM çalışma zamanı" href="/tr/install/docker-vm-runtime" icon="cube">
    Güçlendirilmiş yalıtım için Docker tarafından yönetilen VM çalışma zamanı.
  </Card>
  <Card title="Güncelleme" href="/tr/install/updating" icon="arrow-up-right-from-square">
    OpenClaw paketini ve yönetilen hizmetleri güncelleme.
  </Card>
</CardGroup>
