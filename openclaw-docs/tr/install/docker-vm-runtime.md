---
read_when:
    - OpenClaw'ı Docker ile bir bulut sanal makinesine dağıtıyorsunuz
    - Paylaşılan ikili dosya derleme, kalıcılık ve güncelleme akışına ihtiyacınız var
summary: Uzun süre çalışan OpenClaw Gateway ana makineleri için paylaşılan Docker VM çalışma zamanı adımları
title: Docker sanal makine çalışma zamanı
x-i18n:
    generated_at: "2026-07-27T00:02:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1c474b1f826077ac03c7aaa1e334ed2f38d2de2770f32f2cc907846ecc8bb19
    source_path: install/docker-vm-runtime.md
    workflow: 16
---

GCP, Hetzner ve benzeri VPS sağlayıcılarındaki VM tabanlı Docker kurulumları için paylaşılan çalışma zamanı adımları.

## Gerekli ikili dosyaları imaja dahil edin

Çalışan bir konteynerin içine ikili dosya yüklemek tuzaktır: çalışma zamanında
yüklenen her şey yeniden başlatma sırasında kaybolur. Bir skill'in ihtiyaç duyduğu
tüm harici ikili dosyaları derleme sırasında imaja dahil edin.

Aşağıdaki örnekler alfabetik sırayla yalnızca üç ikili dosyayı kapsar:

- `gog` (kaynak: `gogcli`) Gmail erişimi için
- `goplaces` Google Places için
- `wacli` WhatsApp için

Bunlar örnektir, eksiksiz bir liste değildir. Aynı kalıbı kullanarak
skill'lerinizin ihtiyaç duyduğu kadar ikili dosya yükleyin. Daha sonra yeni bir
ikili dosyaya ihtiyaç duyan bir skill eklediğinizde:

1. Dockerfile'ı güncelleyin.
2. İmajı yeniden derleyin.
3. Konteynerleri yeniden başlatın.

**Örnek Dockerfile**

```dockerfile
FROM node:24-bookworm

RUN apt-get update && apt-get install -y socat && rm -rf /var/lib/apt/lists/*

# Örnek ikili dosya 1: Gmail CLI (gogcli — `gog` olarak yüklenir)
# Güncel Linux varlık URL'sini https://github.com/steipete/gogcli/releases adresinden kopyalayın
RUN curl -L https://github.com/steipete/gogcli/releases/latest/download/gogcli_linux_amd64.tar.gz \
  | tar -xzO gog > /usr/local/bin/gog; \
  chmod +x /usr/local/bin/gog

# Örnek ikili dosya 2: Google Places CLI
# Güncel Linux varlık URL'sini https://github.com/steipete/goplaces/releases adresinden kopyalayın
RUN curl -L https://github.com/steipete/goplaces/releases/latest/download/goplaces_linux_amd64.tar.gz \
  | tar -xzO goplaces > /usr/local/bin/goplaces; \
  chmod +x /usr/local/bin/goplaces

# Örnek ikili dosya 3: WhatsApp CLI
# Güncel Linux varlık URL'sini https://github.com/steipete/wacli/releases adresinden kopyalayın
RUN curl -L https://github.com/steipete/wacli/releases/latest/download/wacli-linux-amd64.tar.gz \
  | tar -xzO wacli > /usr/local/bin/wacli; \
  chmod +x /usr/local/bin/wacli

# Aynı kalıbı kullanarak aşağıya daha fazla ikili dosya ekleyin

WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN corepack enable
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
RUN pnpm ui:install
RUN pnpm ui:build

ENV NODE_ENV=production

CMD ["node","dist/index.js"]
```

<Note>
Yukarıdaki URL'ler örnektir. ARM tabanlı VM'ler için `arm64` varlıklarını seçin. Tekrarlanabilir derlemeler için sürümlendirilmiş sürüm URL'lerini sabitleyin.
</Note>

## Derleyin ve başlatın

```bash
docker compose build
docker compose up -d openclaw-gateway
```

Derleme, `pnpm install --frozen-lockfile` sırasında `Killed` veya 137 çıkış koduyla başarısız olursa VM'in belleği tükenmiştir. Yeniden denemeden önce daha büyük bir makine sınıfı kullanın.

İkili dosyaları doğrulayın:

```bash
docker compose exec openclaw-gateway which gog
docker compose exec openclaw-gateway which goplaces
docker compose exec openclaw-gateway which wacli
```

Beklenen çıktı:

```text
/usr/local/bin/gog
/usr/local/bin/goplaces
/usr/local/bin/wacli
```

Gateway'in çalıştığını doğrulayın:

```bash
docker compose logs -f openclaw-gateway
curl -fsS http://127.0.0.1:18789/healthz
```

`/healthz` tarafından 200 yanıtı döndürülmesi, Gateway işleminin bağlantıları dinlediğini ve sağlıklı olduğunu doğrular; yerleşik imaj `HEALTHCHECK` aynı uç noktayı yoklar.

## Neler nerede kalıcıdır

OpenClaw Docker'da çalışır, ancak doğruluğun kaynağı Docker değildir. Tüm uzun ömürlü durumlar yeniden başlatmalardan, yeniden derlemelerden ve yeniden açılışlardan sonra korunmalıdır.

| Bileşen                | Konum                                                  | Kalıcılık mekanizması     | Notlar                                                                                                              |
| ---------------------- | ------------------------------------------------------ | ------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Gateway yapılandırması | `/home/node/.openclaw/`                                     | Ana makine birim bağlama  | `openclaw.json` dahildir                                                                                         |
| Kanal/sağlayıcı kimlik bilgileri | `/home/node/.openclaw/credentials/`                           | Ana makine birim bağlama  | Kanal ve sağlayıcı kimlik bilgisi materyali                                                                         |
| Model kimlik doğrulama profilleri | `/home/node/.openclaw/agents/`                          | Ana makine birim bağlama  | `agents/<agentId>/agent/auth-profiles.json` (OAuth, API anahtarları)                                                                         |
| Eski OAuth anahtar dosyası | `/home/node/.config/openclaw/`                                 | Ana makine birim bağlama  | Geçiş öncesi OAuth yan dosyaları için salt okunur uyumluluk; `openclaw doctor --fix` bunları `auth-profiles.json` içine taşır |
| Skill yapılandırmaları | `/home/node/.openclaw/skills/`                                     | Ana makine birim bağlama  | Skill düzeyinde durum                                                                                               |
| Ajan çalışma alanı     | `/home/node/.openclaw/workspace/`                                     | Ana makine birim bağlama  | Kod ve ajan eserleri                                                                                                |
| WhatsApp oturumu       | `/home/node/.openclaw/`                                     | Ana makine birim bağlama  | QR ile oturum açmayı korur                                                                                          |
| Gmail anahtarlığı      | `/home/node/.openclaw/`                                     | Ana makine birimi + parola | `GOG_KEYRING_PASSWORD` gerektirir                                                                                       |
| Plugin paketleri       | `/home/node/.openclaw/npm`, `/home/node/.openclaw/git`                | Ana makine birim bağlama  | İndirilebilir plugin paket kökleri                                                                                   |
| Harici ikili dosyalar  | `/usr/local/bin/`                                     | Docker imajı              | Derleme sırasında imaja dahil edilmelidir                                                                           |
| Node çalışma zamanı    | Konteyner dosya sistemi                                | Docker imajı              | Her imaj derlemesinde yeniden oluşturulur                                                                           |
| İşletim sistemi paketleri | Konteyner dosya sistemi                             | Docker imajı              | Çalışma zamanında yüklemeyin                                                                                         |
| Docker konteyneri      | Geçici                                                 | Yeniden başlatılabilir    | Güvenle yok edilebilir                                                                                               |

## Güncellemeler

VM'de OpenClaw'ı güncellemek için:

```bash
git pull
docker compose build
docker compose up -d
```

## İlgili

- [Docker](/tr/install/docker)
- [Podman](/tr/install/podman)
- [ClawDock](/tr/install/clawdock)
