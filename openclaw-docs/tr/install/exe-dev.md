---
read_when:
    - Gateway için düşük maliyetli, her zaman açık bir Linux sunucusu istiyorsunuz
    - Kendi VPS'nizi çalıştırmadan Control UI'ya uzaktan erişmek istiyorsunuz
summary: Uzaktan erişim için OpenClaw Gateway'i exe.dev üzerinde (VM + HTTPS proxy) çalıştırma
title: exe.dev
x-i18n:
    generated_at: "2026-07-26T22:49:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a768511d2d7e4e4ec10bcdae83684417bde05286468b0534200f8dd5ec015f7b
    source_path: install/exe-dev.md
    workflow: 16
---

**Hedef:** `https://<vm-name>.exe.xyz` adresinden erişilebilen, bir [exe.dev](https://exe.dev) sanal makinesinde çalışan OpenClaw Gateway.

Bu kılavuz, exe.dev'in varsayılan **exeuntu** imajını temel alır. Diğer dağıtımlarda paketleri uygun şekilde eşleştirin.

## Gereksinimler

- exe.dev hesabı
- exe.dev sanal makinelerine `ssh exe.dev` erişimi (manuel kurulum için isteğe bağlı)

## Yeni başlayanlar için hızlı yol

1. [https://exe.new/openclaw](https://exe.new/openclaw) adresini açın
2. Gerektiği şekilde kimlik doğrulama anahtarınızı/tokeninizi girin
3. Sanal makinenizin yanındaki "Agent" seçeneğine tıklayın ve Shelley'nin hazırlama işlemini tamamlamasını bekleyin
4. `https://<vm-name>.exe.xyz/` adresini açın ve yapılandırılmış paylaşılan gizli bilgiyle kimlik doğrulayın (varsayılan olarak token kimlik doğrulaması kullanılır; `gateway.auth.mode` ayarına geçerseniz parola kimlik doğrulaması da çalışır)
5. Bekleyen cihaz eşleştirme isteklerini `openclaw devices approve <requestId>` ile onaylayın

## Shelley ile otomatik kurulum

exe.dev'in agent'ı Shelley, OpenClaw'ı bir istem aracılığıyla kurabilir:

```text
Bu sanal makinede OpenClaw'ı (https://docs.openclaw.ai/install) kur. OpenClaw ilk kurulumunda etkileşimsiz ve riski kabul et bayraklarını kullan. Sağlanan kimlik doğrulamasını veya tokeni gerektiği şekilde ekle. WebSocket desteğini etkinleştirdiğinden emin olarak nginx'i, varsayılan etkin site yapılandırmasının kök konumunda varsayılan 18789 portundan yönlendirme yapacak şekilde yapılandır. Eşleştirme, "openclaw devices list" ve "openclaw devices approve <request id>" ile yapılır. Kontrol panelinin OpenClaw sistem durumunu iyi olarak gösterdiğinden emin ol. exe.dev, 8000 portundan 80/443 portuna yönlendirmeyi ve HTTPS'yi bizim için yönetir; dolayısıyla son "erişilebilir" adres, port belirtilmeden <vm-name>.exe.xyz olmalıdır.
```

## Manuel kurulum

<Steps>
  <Step title="Sanal makineyi oluşturun">
    Cihazınızdan:

    ```bash
    ssh exe.dev new
    ```

    Ardından bağlanın:

    ```bash
    ssh <vm-name>.exe.xyz
    ```

    <Tip>
    Bu sanal makineyi **durum bilgisi kalıcı** olarak tutun. OpenClaw; `openclaw.json`, agent başına `auth-profiles.json`, oturumları ve kanal/sağlayıcı durumunu `~/.openclaw/` altında, çalışma alanını ise `~/.openclaw/workspace/` altında depolar.
    </Tip>

  </Step>

  <Step title="Ön koşulları yükleyin (sanal makinede)">
    ```bash
    sudo apt-get update
    sudo apt-get install -y git curl jq ca-certificates openssl
    ```
  </Step>

  <Step title="OpenClaw'ı yükleyin">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```
  </Step>

  <Step title="nginx'i 8000 portuna proxy uygulayacak şekilde yapılandırın">
    `/etc/nginx/sites-enabled/default` dosyasını düzenleyin:

    ```nginx
    server {
        listen 80 default_server;
        listen [::]:80 default_server;
        listen 8000;
        listen [::]:8000;

        server_name _;

        location / {
            proxy_pass http://127.0.0.1:18789;
            proxy_http_version 1.1;

            # WebSocket desteği
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            # Standart proxy üstbilgileri
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $remote_addr;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Uzun ömürlü bağlantılar için zaman aşımı ayarları
            proxy_read_timeout 86400s;
            proxy_send_timeout 86400s;
        }
    }
    ```

    İstemci tarafından sağlanan zincirleri korumak yerine yönlendirme üstbilgilerinin üzerine yazın. OpenClaw, yönlendirilen IP meta verilerine yalnızca açıkça yapılandırılmış proxy'lerden geldiklerinde güvenir ve ekleme tarzındaki `X-Forwarded-For` zincirleri bir güvenlik güçlendirme riski olarak değerlendirilir.

  </Step>

  <Step title="OpenClaw'a erişin ve cihazları onaylayın">
    `https://<vm-name>.exe.xyz/` adresini açın (ilk kurulumun Control UI çıktısına bakın). Kimlik doğrulaması isterse sanal makinedeki yapılandırılmış paylaşılan gizli bilgiyi yapıştırın.

    Bu kılavuz varsayılan olarak token kimlik doğrulamasını kullanır; bu nedenle `gateway.auth.token` değerini `openclaw config get gateway.auth.token` ile alın veya `openclaw doctor --n` ile yeni bir değer oluşturun. Gateway'i parola kimlik doğrulamasına geçirdiyseniz bunun yerine `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` kullanın.

    Cihazları `openclaw devices list` ve `openclaw devices approve <requestId>` ile onaylayın. Emin değilseniz tarayıcınızdan Shelley'yi kullanın.

  </Step>
</Steps>

## Uzak kanal kurulumu

Uzak ana makineler için `config set` hedefine çok sayıda SSH çağrısı yapmak yerine tek bir `config patch` çağrısını tercih edin. Gerçek tokenleri sanal makine ortamında veya `~/.openclaw/.env` içinde tutun ve `openclaw.json` içine yalnızca SecretRef'leri yerleştirin. SecretRef sözleşmesinin tamamı için [Gizli bilgi yönetimi](/tr/gateway/secrets) sayfasına bakın.

Sanal makinede, hizmet ortamının ihtiyaç duyduğu gizli bilgileri içermesini sağlayın:

```bash
cat >> ~/.openclaw/.env <<'EOF'
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...
DISCORD_BOT_TOKEN=...
OPENAI_API_KEY=sk-...
EOF
```

Yerel makinenizde bir yama dosyası oluşturun ve bunu sanal makineye aktarın:

```json5
// openclaw.remote.patch.json5
{
  secrets: {
    providers: {
      default: { source: "env" },
    },
  },
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      groupPolicy: "open",
      requireMention: false,
    },
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
      dmPolicy: "disabled",
      dm: { enabled: false },
      groupPolicy: "allowlist",
    },
  },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.6-sol" },
      models: {
        "openai/gpt-5.6-sol": { params: { fastMode: true } },
      },
    },
  },
}
```

```bash
ssh <vm-name>.exe.xyz 'openclaw config patch --stdin --dry-run' < ./openclaw.remote.patch.json5
ssh <vm-name>.exe.xyz 'openclaw config patch --stdin' < ./openclaw.remote.patch.json5
ssh <vm-name>.exe.xyz 'openclaw gateway restart && openclaw health'
```

İç içe geçmiş bir izin verilenler listesinin tam olarak yama değerine dönüşmesi gerektiğinde, örneğin bir Discord kanalının izin verilenler listesini değiştirirken `--replace-path` kullanın:

```bash
ssh <vm-name>.exe.xyz 'openclaw config patch --stdin --replace-path "channels.discord.guilds[\"123\"].channels"' < ./discord.patch.json5
```

Kanal yapılandırması referansının tamamı için [Discord](/tr/channels/discord) ve [Slack](/tr/channels/slack) bölümlerine bakın.

## Uzaktan erişim

exe.dev, uzaktan erişim için kimlik doğrulamasını yönetir. Varsayılan olarak 8000 portundan gelen HTTP trafiği, e-posta kimlik doğrulamasıyla `https://<vm-name>.exe.xyz` hedefine yönlendirilir.

## Güncelleme

```bash
openclaw update
```

Kanal geçişleri ve manuel kurtarma için [Güncelleme](/tr/install/updating) sayfasına bakın.

## İlgili

- [Uzak Gateway](/tr/gateway/remote)
- [Kuruluma genel bakış](/tr/install)
