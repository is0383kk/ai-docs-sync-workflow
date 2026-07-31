---
read_when:
    - Yeni bir makine kurma
    - Kişisel kurulumunuzu bozmadan "en yeni + en iyi" özellikleri istiyorsunuz
summary: OpenClaw için gelişmiş kurulum ve geliştirme iş akışları
title: Kurulum
x-i18n:
    generated_at: "2026-07-26T23:02:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c40d6d2bf2814465f3cc49c65d4c1498671420af728ce8012d13af3fba67025a
    source_path: start/setup.md
    workflow: 16
---

<Note>
İlk kez kurulum yapıyorsanız [Başlarken](/tr/start/getting-started) ile başlayın.
İlk kullanım ayrıntıları için [İlk Kullanım (CLI)](/tr/start/wizard) bölümüne bakın.
</Note>

## Kısaca

Güncellemeleri ne sıklıkta almak istediğinize ve Gateway'i kendiniz çalıştırmak isteyip istemediğinize göre bir kurulum iş akışı seçin:

- **Özelleştirmeler repo dışında tutulur:** repo güncellemelerinin bunlara dokunmaması için yapılandırmanızı ve çalışma alanınızı `~/.openclaw/openclaw.json` ve `~/.openclaw/workspace/` içinde tutun.
- **Kararlı iş akışı (çoğu kullanıcı için önerilir):** macOS uygulamasını yükleyin ve paketle gelen Gateway'i çalıştırmasına izin verin.
- **En yeni geliştirme iş akışı (geliştirme):** Gateway'i `pnpm gateway:watch` aracılığıyla kendiniz çalıştırın, ardından macOS uygulamasının Yerel modda bağlanmasını sağlayın.

## Ön koşullar (kaynaktan)

- Node 24.15+ önerilir (şu anda `22.22.3+` olan Node 22 LTS hâlâ desteklenmektedir)
- Kaynak kod teslim almaları için `pnpm` gereklidir. OpenClaw, geliştirme modunda paketle gelen pluginleri
  `extensions/*` pnpm çalışma alanı paketlerinden yüklediğinden kök `npm install`
  kaynak ağacının tamamını hazırlamaz.
- Docker (isteğe bağlı; yalnızca kapsayıcılı kurulum/e2e için; bkz. [Docker](/tr/install/docker))

## Özelleştirme stratejisi (güncellemelerin sorun çıkarmaması için)

Hem "%100 bana özel" bir yapı _hem de_ kolay güncellemeler istiyorsanız özelleştirmelerinizi şuralarda tutun:

- **Yapılandırma:** `~/.openclaw/openclaw.json` (JSON/JSON5 benzeri)
- **Çalışma alanı:** `~/.openclaw/workspace` (beceriler, istemler, bellekler; bunu özel bir git reposu yapın)

Tam ilk kullanım sihirbazını çalıştırmadan yapılandırma/çalışma alanı klasörlerini bir kez önyükleyin:

```bash
openclaw setup --baseline
```

Henüz genel kurulum yok mu? Bunun yerine bu repodan çalıştırın:

```bash
pnpm openclaw setup --baseline
```

(`--baseline` olmadan yalnızca `openclaw setup` kullanmak, `openclaw onboard` için bir diğer addır ve tam etkileşimli sihirbazı çalıştırır.)

## Gateway'i bu repodan çalıştırma

`pnpm build` sonrasında paketlenmiş CLI'yi doğrudan çalıştırabilirsiniz:

```bash
node openclaw.mjs gateway --port 18789 --verbose
```

## Kararlı iş akışı (önce macOS uygulaması)

1. **OpenClaw.app** uygulamasını yükleyip başlatın (menü çubuğu).
2. İlk kullanım/izinler denetim listesini tamamlayın (TCC istemleri).
3. Gateway'in **Local** olarak ayarlandığından ve çalıştığından emin olun (uygulama bunu yönetir).
4. Hizmetleri bağlayın (örnek: WhatsApp):

```bash
openclaw channels login
```

5. Hızlı doğrulama:

```bash
openclaw health
```

Derlemenizde ilk kullanım özelliği mevcut değilse:

- `openclaw setup`, ardından `openclaw channels login` komutunu çalıştırın ve sonrasında Gateway'i elle başlatın (`openclaw gateway`).

## En yeni geliştirme iş akışı (terminalde Gateway)

Amaç: TypeScript Gateway üzerinde çalışmak, anında yeniden yükleme kullanmak ve macOS uygulaması arayüzünü bağlı tutmak.

### 0) (İsteğe bağlı) macOS uygulamasını da kaynaktan çalıştırma

macOS uygulamasının da en yeni geliştirme sürümünü kullanmak istiyorsanız:

```bash
./scripts/restart-mac.sh
```

### 1) Geliştirme Gateway'ini başlatma

```bash
pnpm install
# Yalnızca ilk çalıştırmada (veya yerel OpenClaw yapılandırmasını/çalışma alanını sıfırladıktan sonra)
pnpm openclaw setup
pnpm gateway:watch
```

`gateway:watch`, adlandırılmış bir tmux oturumunda
(`openclaw-gateway-watch-main`) Gateway izleme sürecini başlatır veya yeniden başlatır ve etkileşimli
terminallerden otomatik olarak bağlanır. Etkileşimsiz kabuklar bağlantısız kalır ve
`tmux attach -t openclaw-gateway-watch-main` değerini yazdırır; etkileşimli bir çalıştırmayı
bağlantısız tutmak için `OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch`, ön planda izleme modu içinse
`pnpm gateway:watch:raw` kullanın. İzleyici, yapılandırılmış/varsayılan bağlantı noktasını
devralmadan önce etkin profilin yüklü Gateway hizmetini durdurarak hizmet
gözetmeninin kaynak sürecinin yerini almasını önler. Hizmet yüklü kalır; izlemeyi
bitirdiğinizde `pnpm openclaw gateway start` komutunu çalıştırın. Başlatma başarısız olduktan
sonra tmux bölmesi kullanılabilir durumda kalır; böylece başka bir terminal veya
aracı bağlanabilir ya da günlüklerini yakalayabilir. İzleyici, ilgili kaynak,
yapılandırma ve paketle gelen plugin meta verisi değişikliklerinde yeniden yükleme
yapar. İzlenen Gateway başlatma sırasında kapanırsa `gateway:watch`,
`openclaw doctor --fix --non-interactive` komutunu bir kez çalıştırıp yeniden dener; yalnızca geliştirmeye
özgü bu onarım geçişini devre dışı bırakmak için `OPENCLAW_GATEWAY_WATCH_AUTO_DOCTOR=0` ayarını yapın.
`pnpm gateway:watch`, `dist/control-ui` bileşenini yeniden derlemez; bu nedenle
`ui/` değişikliklerinden sonra `pnpm ui:build` komutunu yeniden
çalıştırın veya Denetim Arayüzünü geliştirirken `pnpm ui:dev` kullanın.

### 2) macOS uygulamasını çalışan Gateway'inize yönlendirme

**OpenClaw.app** içinde:

- Connection Mode: **Local**
  Uygulama, yapılandırılmış bağlantı noktasında çalışan gateway'e bağlanır.

### 3) Doğrulama

- Uygulama içindeki Gateway durumu **"Using existing gateway …"** olarak görünmelidir
- Alternatif olarak CLI aracılığıyla:

```bash
openclaw health
```

### Sık karşılaşılan sorunlar

- **Yanlış bağlantı noktası:** Gateway WS varsayılan olarak `ws://127.0.0.1:18789` kullanır; uygulama ile CLI'yi aynı bağlantı noktasında tutun.
- **Durumun saklandığı yer:**
  - Kanal/sağlayıcı durumu: `~/.openclaw/credentials/`
  - Model kimlik doğrulama profilleri: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
  - Oturumlar ve dökümler: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
  - Eski/arşivlenmiş oturum yapıtları: `~/.openclaw/agents/<agentId>/sessions/`
  - Günlükler: `/tmp/openclaw/`

## Kimlik bilgisi depolama haritası

Kimlik doğrulama sorunlarını giderirken veya nelerin yedekleneceğine karar verirken bunu kullanın:

- **WhatsApp**: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- **Telegram bot belirteci**: yapılandırma/ortam veya `channels.telegram.tokenFile` (yalnızca normal dosya; sembolik bağlantılar reddedilir)
- **Discord bot belirteci**: yapılandırma/ortam veya SecretRef (ortam/dosya/çalıştırma sağlayıcıları)
- **Slack belirteçleri**: yapılandırma/ortam (`channels.slack.*`)
- **Eşleştirme izin listeleri**:
  - `~/.openclaw/credentials/<channel>-allowFrom.json` (varsayılan hesap)
  - `~/.openclaw/credentials/<channel>-<accountId>-allowFrom.json` (varsayılan olmayan hesaplar)
- **Model kimlik doğrulama profilleri**: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **Dosya destekli gizli bilgi yükü (isteğe bağlı)**: `~/.openclaw/secrets.json`
- **Eski OAuth içe aktarımı**: `~/.openclaw/credentials/oauth.json`
  Daha fazla ayrıntı: [Güvenlik](/tr/gateway/security#credential-storage-map).

## Güncelleme (kurulumunuzu bozmadan)

- `~/.openclaw/workspace` ve `~/.openclaw/` öğelerini "size ait içerikler" olarak tutun; kişisel istemleri/yapılandırmayı `openclaw` reposuna koymayın.
- Kaynağı güncelleme: `git pull` + `pnpm install` + `pnpm gateway:watch` kullanmaya devam edin.

## Linux (systemd kullanıcı hizmeti)

Linux kurulumları bir systemd **kullanıcı** hizmeti kullanır. Varsayılan olarak systemd,
oturum kapatıldığında veya boşta kalındığında kullanıcı hizmetlerini durdurur ve bu da
Gateway'i sonlandırır. İlk kullanım süreci, kalıcı kullanıcı oturumunu sizin için
etkinleştirmeyi dener (sudo parolası isteyebilir). Hâlâ kapalıysa şunu çalıştırın:

```bash
sudo loginctl enable-linger $USER
```

Sürekli açık veya çok kullanıcılı sunucularda kullanıcı hizmeti yerine bir **sistem**
hizmeti kullanmayı değerlendirin (kalıcı kullanıcı oturumu gerekmez). systemd notları için
[Gateway işletim kılavuzu](/tr/gateway) bölümüne bakın.

## İlgili belgeler

- [Gateway işletim kılavuzu](/tr/gateway) (bayraklar, gözetim, bağlantı noktaları)
- [Gateway yapılandırması](/tr/gateway/configuration) (yapılandırma şeması + örnekler)
- [Discord](/tr/channels/discord) ve [Telegram](/tr/channels/telegram) (yanıt etiketleri + replyToMode ayarları)
- [OpenClaw asistanı kurulumu](/tr/start/openclaw)
- [macOS uygulaması](/tr/platforms/macos) (gateway yaşam döngüsü)
