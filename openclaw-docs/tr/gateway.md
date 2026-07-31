---
read_when:
    - Gateway işlemini çalıştırma veya hata ayıklama
summary: Gateway hizmeti, yaşam döngüsü ve operasyonları için çalışma kılavuzu
title: Gateway operasyon kılavuzu
x-i18n:
    generated_at: "2026-07-26T23:57:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d8b50b6041905c321887ea0f579f8d4c3b74552b2b72c37ec655e43a53dfc130
    source_path: gateway/index.md
    workflow: 16
---

Bu sayfayı Gateway hizmetinin ilk gün başlatılması ve sonraki günlerde işletilmesi için kullanın.

<CardGroup cols={2}>
  <Card title="Derinlemesine sorun giderme" icon="siren" href="/tr/gateway/troubleshooting">
    Kesin komut sıraları ve günlük imzalarıyla, belirtilerden başlayan tanılama.
  </Card>
  <Card title="Yapılandırma" icon="sliders" href="/tr/gateway/configuration">
    Görev odaklı kurulum kılavuzu + eksiksiz yapılandırma başvurusu.
  </Card>
  <Card title="Gizli bilgi yönetimi" icon="key-round" href="/tr/gateway/secrets">
    SecretRef sözleşmesi, çalışma zamanı anlık görüntüsü davranışı ve taşıma/yeniden yükleme işlemleri.
  </Card>
  <Card title="Gizli bilgi planı sözleşmesi" icon="shield-check" href="/tr/gateway/secrets-plan-contract">
    Kesin `secrets apply` hedef/yol kuralları ve yalnızca referans kullanan kimlik doğrulama profili davranışı.
  </Card>
</CardGroup>

## 5 dakikada yerel başlatma

<Steps>
  <Step title="Gateway'i başlatın">

```bash
openclaw gateway --port 18789
# hata ayıklama/izleme stdio'ya yansıtılır
openclaw gateway --port 18789 --verbose
# seçili porttaki dinleyiciyi zorla sonlandırın, ardından başlatın
openclaw gateway --force
```

  </Step>

  <Step title="Hizmet durumunu doğrulayın">

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
```

Sağlıklı temel durum: `Runtime: running`, `Connectivity probe: ok` ve beklediğinizle eşleşen bir `Capability` satırı. Yalnızca erişilebilirliği değil, okuma kapsamlı RPC'yi kanıtlamak için `openclaw gateway status --require-rpc` kullanın.

  </Step>

  <Step title="Kanal hazırlığını doğrulayın">

```bash
openclaw channels status --probe
```

Erişilebilir bir Gateway ile bu komut, hesap başına canlı kanal yoklamalarını ve isteğe bağlı denetimleri çalıştırır. Gateway'e erişilemiyorsa CLI, yalnızca yapılandırmaya dayalı kanal özetlerine geri döner.

  </Step>
</Steps>

<Note>
Gateway yapılandırmasının yeniden yüklenmesi, etkin yapılandırma dosyası yolunu izler (profil/durum varsayılanlarından çözümlenir veya ayarlanmışsa `OPENCLAW_CONFIG_PATH` kullanılır). Varsayılan mod `gateway.reload.mode="hybrid"` şeklindedir. İlk başarılı yüklemeden sonra çalışan süreç, bellekteki etkin yapılandırma anlık görüntüsünü sunar; başarılı bir yeniden yükleme bu anlık görüntüyü atomik olarak değiştirir.
</Note>

## Çalışma zamanı modeli

- Yönlendirme, denetim düzlemi ve kanal bağlantıları için sürekli çalışan tek süreç.
- Şunlar için çoklanmış tek port:
  - WebSocket denetimi/RPC
  - HTTP API'leri (`/v1/models`, `/v1/embeddings`, `/v1/chat/completions`, `/v1/responses`, `/tools/invoke`)
  - İsteğe bağlı `/api/v1/admin/rpc` gibi Plugin HTTP rotaları
  - Denetim Arayüzü ve kancalar
- Varsayılan bağlama modu: `loopback`. Algılanan bir konteyner ortamında geçerli varsayılan `auto` olur (port yönlendirme için `0.0.0.0` olarak çözümlenir); ancak Tailscale sunma/tünelleme etkinse her zaman `loopback` kullanılması zorlanır.
- Kimlik doğrulama varsayılan olarak zorunludur. Paylaşılan gizli bilgi kurulumları `gateway.auth.token` / `gateway.auth.password` (veya `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`) kullanır; döngüsel olmayan ters proxy kurulumları ise `gateway.auth.mode: "trusted-proxy"` kullanabilir.

## OpenAI uyumlu uç noktalar

OpenClaw'ın en yüksek etkili uyumluluk yüzeyi:

- `GET /v1/models`
- `GET /v1/models/{id}`
- `POST /v1/embeddings`
- `POST /v1/chat/completions`
- `POST /v1/responses`

Bu kümenin önemli olmasının nedenleri:

- Open WebUI, LobeChat ve LibreChat entegrasyonlarının çoğu önce `/v1/models` yoklaması yapar.
- Birçok RAG ve bellek işlem hattı `/v1/embeddings` bekler.
- Ajan odaklı istemciler giderek daha fazla `/v1/responses` tercih etmektedir.

`/v1/models` öncelikle ajanlara yöneliktir: yapılandırılan her ajan için `openclaw`, `openclaw/default` ve `openclaw/<agentId>` döndürür. `openclaw/default`, her zaman yapılandırılmış varsayılan ajanla eşlenen kararlı takma addır. Arka uç sağlayıcısını/modelini geçersiz kılmak istediğinizde `x-openclaw-model` gönderin; aksi takdirde seçili ajanın normal modeli ve gömme kurulumu denetimi elinde tutar.

Bunların tümü ana Gateway portunda çalışır ve Gateway HTTP API'sinin geri kalanıyla aynı güvenilir operatör kimlik doğrulama sınırını kullanır.

Yönetici HTTP RPC'si (`POST /api/v1/admin/rpc`), WebSocket RPC kullanamayan ana makine araçları için ayrı ve varsayılan olarak kapalı bir Plugin rotasıdır. Bkz. [Yönetici HTTP RPC'si](/tr/plugins/admin-http-rpc).

### Port ve bağlama önceliği

| Ayar         | Çözümleme sırası                                                     |
| ------------ | -------------------------------------------------------------------- |
| Gateway portu | `--port` → `OPENCLAW_GATEWAY_PORT` → `gateway.port` → `18789`        |
| Bağlama modu | CLI/geçersiz kılma → `gateway.bind` → `loopback` (veya konteynerlerde `auto`) |

Kurulu Gateway hizmetleri, çözümlenen `--port` değerini gözetmen meta verilerine kaydeder. `gateway.port` değerini değiştirdikten sonra launchd/systemd/schtasks'in süreci yeni portta başlatması için `openclaw doctor --fix` veya `openclaw gateway install --force` çalıştırın.

Gateway başlatılırken, döngüsel olmayan bağlamalar için yerel Denetim Arayüzü kaynakları oluşturulurken aynı etkin port ve bağlama kullanılır. Örneğin `--bind lan --port 3000`, çalışma zamanı doğrulaması çalışmadan önce `http://localhost:3000` ve `http://127.0.0.1:3000` değerlerini oluşturur. HTTPS proxy URL'leri gibi tüm uzak tarayıcı kaynaklarını `gateway.controlUi.allowedOrigins` öğesine açıkça ekleyin.

### Çalışırken yeniden yükleme modları

| `gateway.reload.mode` | Davranış                                   |
| --------------------- | ------------------------------------------ |
| `off`                 | Yapılandırma yeniden yüklenmez             |
| `hot`                 | Yalnızca çalışırken güvenle uygulanabilen değişiklikleri uygular |
| `restart`             | Yeniden yükleme gerektiren değişikliklerde yeniden başlatır |
| `hybrid` (varsayılan) | Güvenli olduğunda çalışırken uygular, gerektiğinde yeniden başlatır |

## Operatör komut kümesi

```bash
openclaw gateway status
openclaw gateway status --deep   # sistem düzeyinde hizmet taraması ekler
openclaw gateway status --json
openclaw gateway install
openclaw gateway restart
openclaw gateway stop
openclaw secrets reload
openclaw logs --follow
openclaw doctor
```

`gateway status --deep`, daha derin bir RPC durum yoklaması için değil, ek hizmet keşfi (LaunchDaemons/systemd sistem birimleri/schtasks) içindir.

## Birden fazla Gateway (aynı ana makine)

Çoğu kurulum, makine başına bir Gateway çalıştırmalıdır. Tek bir Gateway birden fazla ajanı ve kanalı barındırabilir. Yalnızca bilinçli olarak yalıtım veya bir kurtarma botu istediğinizde birden fazla Gateway'e ihtiyacınız vardır.

Yararlı kontroller:

```bash
openclaw gateway status --deep
openclaw gateway probe
```

Beklenecekler:

- `gateway status --deep`, eski launchd/systemd/schtasks kurulumları hâlâ mevcutsa `Other gateway-like services detected (best effort)` bildirebilir ve temizleme ipuçları yazdırabilir.
- `gateway probe`, farklı Gateway'ler yanıt verdiğinde veya OpenClaw erişilebilir hedeflerin aynı Gateway olduğunu kanıtlayamadığında `multiple reachable gateway identities` hakkında uyarabilir. Aynı Gateway'e yönelik bir SSH tüneli, proxy URL'si veya yapılandırılmış uzak URL, aktarım portları farklı olsa bile birden fazla aktarıma sahip tek bir Gateway'dir.
- Bu bilinçliyse her Gateway için portları, yapılandırmayı/durumu ve çalışma alanı köklerini yalıtın.

Örnek başına kontrol listesi:

- Benzersiz `gateway.port`
- Benzersiz `OPENCLAW_CONFIG_PATH`
- Benzersiz `OPENCLAW_STATE_DIR`
- Benzersiz `agents.defaults.workspace`

Örnek:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json OPENCLAW_STATE_DIR=~/.openclaw-a openclaw gateway --port 19001
OPENCLAW_CONFIG_PATH=~/.openclaw/b.json OPENCLAW_STATE_DIR=~/.openclaw-b openclaw gateway --port 19002
```

Ayrıntılı kurulum: [/gateway/multiple-gateways](/tr/gateway/multiple-gateways).

## Uzaktan erişim

Tercih edilen: Tailscale/VPN.
Alternatif: SSH tüneli.

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

Ardından istemcileri yerel olarak `ws://127.0.0.1:18789` adresine bağlayın.

<Warning>
SSH tünelleri Gateway kimlik doğrulamasını atlamaz. Paylaşılan gizli bilgi kimlik doğrulamasında istemciler, tünel üzerinden bile
`token`/`password` göndermek zorundadır. Kimlik taşıyan modlarda
istek yine de bu kimlik doğrulama yolunun gerekliliklerini karşılamalıdır.
</Warning>

Bkz: [Uzak Gateway](/tr/gateway/remote), [Kimlik doğrulama](/tr/gateway/authentication), [Tailscale](/tr/gateway/tailscale).

## Gözetim ve hizmet yaşam döngüsü

Üretim benzeri güvenilirlik için gözetimli çalıştırmaları kullanın.

<Tabs>
  <Tab title="macOS (launchd)">

```bash
openclaw gateway install
openclaw gateway status
openclaw gateway restart
openclaw gateway stop
```

Yeniden başlatmalar için `openclaw gateway restart` kullanın. Yeniden başlatma yerine `openclaw gateway stop` ve `openclaw gateway start` komutlarını zincirlemeyin.

macOS'te `gateway stop` varsayılan olarak `launchctl bootout` kullanır. Bu, LaunchAgent'ı kalıcı olarak devre dışı bırakmadan geçerli önyükleme oturumundan kaldırır; böylece beklenmeyen çökmelerden sonra KeepAlive otomatik kurtarması çalışmaya devam eder ve `gateway start` temiz şekilde yeniden etkinleştirir. Otomatik yeniden oluşturmayı yeniden başlatmalar arasında kalıcı olarak engellemek için `--disable` iletin: `openclaw gateway stop --disable`.

LaunchAgent etiketleri `ai.openclaw.gateway` (varsayılan) veya `ai.openclaw.<profile>` (adlandırılmış profil) şeklindedir. `openclaw doctor`, hizmet yapılandırması sapmasını denetler ve onarır.

  </Tab>

  <Tab title="Linux (systemd kullanıcısı)">

```bash
openclaw gateway install
systemctl --user enable --now openclaw-gateway[-<profile>].service
openclaw gateway status
```

Oturum kapatıldıktan sonra kalıcı olması için lingering özelliğini etkinleştirin:

```bash
sudo loginctl enable-linger $(whoami)
```

Masaüstü oturumu olmayan başsız bir sunucuda, `systemctl --user` komutlarını yeniden denemeden önce `XDG_RUNTIME_DIR` değerinin de ayarlandığından (`export XDG_RUNTIME_DIR=/run/user/$(id -u)`) emin olun.

Özel bir kurulum yoluna ihtiyacınız olduğunda manuel kullanıcı birimi örneği:

```ini
[Unit]
Description=OpenClaw Gateway
After=network-online.target
Wants=network-online.target
StartLimitBurst=5
StartLimitIntervalSec=60

[Service]
ExecStart=/usr/local/bin/openclaw gateway --port 18789
Restart=always
RestartSec=5
RestartPreventExitStatus=78
TimeoutStopSec=30
TimeoutStartSec=30
SuccessExitStatus=0 143
OOMPolicy=continue
KillMode=control-group

[Install]
WantedBy=default.target
```

  </Tab>

  <Tab title="Windows (yerel)">

```powershell
openclaw gateway install
openclaw gateway status --json
openclaw gateway restart
openclaw gateway stop
```

Yerel Windows yönetimli başlatma, `OpenClaw Gateway` adlı bir Zamanlanmış Görev kullanır
(adlandırılmış profiller için `OpenClaw Gateway (<profile>)`). Zamanlanmış Görev
oluşturma reddedilirse OpenClaw, durum dizinindeki
`gateway.cmd` konumunu gösteren kullanıcı başına Başlangıç klasörü başlatıcısına geri döner.

  </Tab>

  <Tab title="Linux (sistem hizmeti)">

Çok kullanıcılı/sürekli açık ana makineler için bir sistem birimi kullanın.

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-gateway[-<profile>].service
```

Kullanıcı birimiyle aynı hizmet gövdesini kullanın ancak bunu
`/etc/systemd/system/openclaw-gateway[-<profile>].service` altına kurun ve `openclaw` ikili dosyanız başka bir konumdaysa
`ExecStart=` değerini ayarlayın.

Aynı profil/port için `openclaw doctor --fix` komutunun kullanıcı düzeyinde bir Gateway hizmeti kurmasına da izin vermeyin. Doctor, sistem düzeyinde bir OpenClaw Gateway hizmeti bulduğunda bu otomatik kurulumu reddeder; yaşam döngüsünü sistem birimi yönetiyorsa `OPENCLAW_SERVICE_REPAIR_POLICY=external` kullanın.

  </Tab>
</Tabs>

Geçersiz yapılandırma hataları `78` koduyla çıkar. Linux systemd birimleri, yapılandırma düzeltilene kadar yeniden başlatmayı durdurmak için `RestartPreventExitStatus=78` kullanır. launchd ve Windows Görev Zamanlayıcı'da çıkış koduna göre durdurmaya yönelik eşdeğer bir kural bulunmadığından Gateway ayrıca hızlı ve temiz olmayan önyükleme geçmişini kalıcı olarak saklar ve yinelenen başlatma hatalarından sonra kanal/sağlayıcı hesaplarının otomatik başlatılmasını engeller. Bu güvenli modda denetim düzlemi inceleme ve onarım için çalışmaya devam eder; yapılandırmanın çalışırken yeniden yüklenmesi ve `secrets.reload`, kanalların otomatik yeniden başlatılmasını reddeder ve operatörün açık bir `channels.start` isteği engellemeyi geçersiz kılabilir.

## Geliştirme profili için hızlı yol

```bash
openclaw --dev setup
openclaw --dev gateway --allow-unconfigured
openclaw --dev status
```

Varsayılanlar, yalıtılmış durum/yapılandırmayı ve `19001` temel Gateway portunu içerir.

## Protokol hızlı başvurusu (operatör görünümü)

- İlk istemci çerçevesi `connect` olmalıdır.
- Gateway, bir `snapshot` (`presence`, `health`, `stateVersion`, `uptimeMs`) ve `policy` sınırları (`maxPayload`, `maxBufferedBytes`, `tickIntervalMs`) içeren bir `hello-ok` çerçevesi döndürür.
- `hello-ok.features.methods` / `events`, çağrılabilir tüm yardımcı rotaların
  oluşturulmuş bir dökümü değil, ölçülü bir keşif listesidir.
- İstekler: `req(method, params)` → `res(ok/payload|error)`.
- Yaygın olaylar arasında `connect.challenge`, `agent`, `chat`,
  `session.message`, `session.operation`, `session.tool`, isteğe bağlı
  `session.approval`, `sessions.changed`, `presence`, `tick`, `health`,
  `heartbeat`, eşleştirme/onay yaşam döngüsü olayları ve `shutdown` bulunur.

Aracı çalıştırmaları iki aşamalıdır:

1. Anında kabul bildirimi (`status:"accepted"`)
2. Arada akışla iletilen `agent` olaylarıyla birlikte nihai tamamlanma yanıtı (`status:"ok"|"error"`).

Protokol belgelerinin tamamına bakın: [Gateway Protokolü](/tr/gateway/protocol).

## İşletim denetimleri

### Canlılık

- WS bağlantısını açın ve `connect` gönderin.
- Anlık görüntüyü içeren `hello-ok` yanıtını bekleyin.

### Hazır olma durumu

```bash
openclaw gateway status
openclaw channels status --probe
openclaw health
```

### Boşluk kurtarma

Olaylar yeniden oynatılmaz. Sıra boşluklarında devam etmeden önce durumu (`health`, `system-presence`) yenileyin.

## Yaygın hata belirtileri

| Belirti                                                        | Olası sorun                                                                    |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| `refusing to bind gateway ... without auth`                    | Geçerli bir Gateway kimlik doğrulama yolu olmadan geri döngü dışı bağlama      |
| `another gateway instance is already listening` / `EADDRINUSE` | Bağlantı noktası çakışması                                                     |
| `Gateway start blocked: set gateway.mode=local`                | Yapılandırma uzak moda ayarlanmış veya bozulmuş bir yapılandırmada `gateway.mode` eksik |
| Bağlantı sırasında `unauthorized`                                  | İstemci ile Gateway arasında kimlik doğrulama uyuşmazlığı                       |

Tanılama adımlarının tamamı için [Gateway Sorun Giderme](/tr/gateway/troubleshooting) bölümünü kullanın.

## Güvenlik garantileri

- Gateway kullanılamadığında Gateway protokolü istemcileri hızla başarısız olur (örtük doğrudan kanal geri dönüşü yoktur).
- Geçersiz/bağlantı kurma amaçlı olmayan ilk çerçeveler reddedilir ve bağlantı kapatılır.
- Sorunsuz kapatma, soket kapanmadan önce `shutdown` olayını yayınlar.

## İlgili

- [Yapılandırma](/tr/gateway/configuration)
- [Gateway sorun giderme](/tr/gateway/troubleshooting)
- [Arka plan işlemi](/tr/gateway/background-process)
- [Sistem durumu](/tr/gateway/health)
- [Doctor](/tr/gateway/doctor)
- [Kimlik doğrulama](/tr/gateway/authentication)
- [Uzaktan erişim](/tr/gateway/remote)
- [Gizli bilgi yönetimi](/tr/gateway/secrets)
