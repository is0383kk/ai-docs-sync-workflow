---
read_when:
    - Linux yardımcı uygulamasının durumu aranıyor
    - Linux Node ana makinesinde kamera, konum veya bildirimleri etkinleştirme
    - Platform kapsamını veya katkıları planlama
    - Bir VPS veya container'da Linux OOM sonlandırmalarında ya da 137 çıkış kodunda hata ayıklama
summary: Linux desteği + yardımcı uygulama durumu
title: Linux uygulaması
x-i18n:
    generated_at: "2026-07-26T23:27:33Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fe55d3ec63fcf8291a24126c04638f005c03c3d44ff84a26a925e931066b01cc
    source_path: platforms/linux.md
    workflow: 16
---

Gateway, Linux'ta tam olarak desteklenir ve Node gerektirir. Bun yine de
bağımlılık yükleyicisi veya paket betiği çalıştırıcısı olarak kullanılabilir, ancak
`node:sqlite` sağlamadığı için OpenClaw'ı çalıştıramaz.

## Masaüstü yardımcı uygulaması

OpenClaw Linux yardımcı uygulaması, yerel bir Gateway için Tauri masaüstü uygulamasıdır. Şunları yapar:

- eksik olduklarında OpenClaw CLI'ı ve yönetilen Node çalışma zamanını yükler; sürüm derlemeleri kararlı kanalı otomatik olarak yüklerken geliştirme derlemeleri önce kanalı sorar
- hizmet değişikliklerini denemeden önce sağlıklı bir Gateway'e bağlanır
- yükleme, başlatma, durdurma ve yeniden başlatma işlemlerini CLI tarafından yönetilen systemd kullanıcı hizmetine devreder
- yakındaki Bonjour Gateway'lerini keşfeder ve her Control UI'ı rota kapsamlı bir pencerede açar; böylece birden fazla
  Gateway panosu bağlı kalabilir ve eş zamanlı olarak kullanılabilir
- Gateway tarafından sunulan Control UI'ı çözümlenmiş kimlik doğrulama URL'siyle açar
- ilk çalıştırma yüklemesinden sonra Control UI'ı ilk katılım modunda açar; bu mod,
  algılanan Claude Code, Codex veya Hermes belleklerini agent çalışma alanına
  aktarmayı önerir (aynı içe aktarma daha sonra
  Settings → Import Memory altında da kullanılabilir)
- aynı konumdaki bir CLI Node ana makinesi için agent tarafından yönlendirilen Canvas'ı ve paketlenmiş A2UI içeriğini işler
- penceresi kapatıldığında sistem tepsisinden kullanılabilir olmaya devam eder

`main` kaynağından oluşturulan kararlı sürümler, etiketin
[GitHub sürümünde](https://github.com/openclaw/openclaw/releases) varlık olarak `.deb` ve AppImage paketlerini,
`OpenClaw-<version>-amd64.deb` ve `OpenClaw-<version>-amd64.AppImage`
adlarıyla ve yanlarında bir `SHA256SUMS.linux-app.txt` sağlama toplamı dosyasıyla sunar.
`.deb` dosyasını indirin ve `sudo apt install ./OpenClaw-<version>-amd64.deb` ile yükleyin
veya AppImage dosyasını yürütülebilir olarak işaretleyip doğrudan çalıştırın. AppImage çalışma zamanı
FUSE 2 gerektirir (`sudo apt install libfuse2` veya Ubuntu 24.04+ üzerinde `libfuse2t64`);
bu olmadan AppImage'ı `APPIMAGE_EXTRACT_AND_RUN=1` ile çalıştırın.

Aynı paketleri bir kaynak kod deposu kopyasından da oluşturabilirsiniz:

```bash
cd apps/linux/src-tauri
pnpm dlx @tauri-apps/cli@2.11.4 build --bundles deb,appimage
```

`Linux App` CI iş akışı, uygulamaya dokunan pull request'ler ve
elle çalıştırmalar için aynı paketleri `openclaw-linux-companion` yapıtı olarak yükler.
Linux derleme bağımlılıkları ve geliştirme komutları için depodaki
`apps/linux/README.md` bölümüne bakın.

### Hızlı Sohbet

Hızlı Sohbet'i `Ctrl+Shift+Space` veya tepsideki **Hızlı Sohbet** öğesiyle açın. Agent
çipi yapılandırılan avatarı, emojiyi veya monogramı gösterir; agent'lar arasında geçiş yapmak için seçin.
İletiler, seçilen agent'ın ana oturumunu kullanır ve genel oturum kapsamına uyar.
Yerel Rust istemcisi kalıcı bir Ed25519 cihaz kimliğini yönetir. Eşleştirmeyi
başlatmak için yalnızca CLI aktarımındaki paylaşılan token'ı veya parolayı kullanır, ardından
Gateway tarafından verilen cihaz token'ını depolar ve sonraki bağlantılarda
bunu tercih eder. Kimlik ve cihaz token'ı, uygulama yapılandırma dizinindeki `0600`
modlu bir dosyada bulunur; Hızlı Sohbet'in WebView'ı ne kimlik bilgilerini ne de WebSocket'i alır.

Yerel bağlantı kullanılamadığında Hızlı Sohbet **Gateway'e
ulaşılamıyor — yeniden deneniyor** iletisini gösterir ve yeniden bağlantı kurulana kadar göndermeyi devre dışı bırakır. Eşleştirme
aşamasına ulaşmış uzak bir cihaz bunun yerine **Bu cihazı panoda
onaylayın (Nodes)** iletisini, Gateway sağladığında kısa bir cihaz kimliğiyle birlikte gösterir.
Eksik bir paylaşılan kimlik bilgisi gerektiren Gateway, **Gateway bir
kimlik bilgisi gerektiriyor — panoyu gateway ana makinesinde açın** iletisini gösterir; bu durumda onay
bekleyen bir eşleştirme isteği yoktur. Sunucu tarafından sağlanan düzeltme yönergeleri
daha ayrıntılı olduğunda bu yedek bildirimlerin yerini alır.
TLS Gateway'leri için CLI, Gateway sertifikasının SHA-256
parmak izini uygulamaya aktarır; yerel istemci bu sertifikayı sabitler ve **Gateway TLS
güveni başarısız oldu — sertifika parmak izini kontrol edin** durumunu kesintiden ayrı olarak bildirir.
Paylaşılan gizli değeri SecretRef üzerinden yapılandırılan Gateway'ler bunu
CLI aktarımına dahil etmez. Mevcut eşleştirilmiş kurulumlar, depolanan cihaz
token'ları üzerinden çalışmaya devam eder; ancak yeni bir kurulum, bu başlangıç kimlik bilgisi olmadan paylaşılan gizli değerli
kimlik doğrulama altında bekleyen bir eşleştirme isteği oluşturamaz.
Kurulum kodu ve `bootstrapToken` kullanımı özel ürün kullanıcı arayüzü gerektirir ve
ileride ele alınacaktır; Hızlı Sohbet iki akışı da denemez.

X11'de özel bir kısayolu kaydetmek veya sıfırlamak için Hızlı Sohbet'teki dişli simgesini kullanın.
**Hızlı Sohbet kısayolu** tepsi geçişi, düz **Hızlı Sohbet** tepsi öğesini
devre dışı bırakmadan kısayolu etkinleştirir veya devre dışı bırakır. Genel kısayollar Wayland'de kullanılamadığından
kısayol ayarları gizlenir ve tepsi öğesi giriş noktası olarak kalır.
Kabul edilen bir gönderimden sonra Hızlı Sohbet açık kalır ve seçilen agent'ın
düz metin yanıtını düzenleyicinin altında akış halinde gösterir. Çubuğu ve yanıtını kapatmak için `Esc` tuşuna basın;
`Ctrl+Enter` yine de panoyu açar.

### Canvas

Linux Canvas iki iş birliği yapan işlem kullanır. `openclaw node run` tek Gateway Node bağlantısı olarak kalır; paketlenmiş `linux-canvas` Plugin'i, `canvas.*` çağrılarını yalnızca kullanıcıya açık bir Unix soketi üzerinden çalışan masaüstü uygulamasına iletir. Uygulama, paketlenmiş A2UI işleyicisi ve agent'a geri dönen eylem köprüsü dahil olmak üzere isteğe bağlı tek bir WebView penceresini yönetir.

Plugin varsayılan olarak etkindir. Canvas'ın duyurusunu yalnızca masaüstü soketi `$XDG_RUNTIME_DIR/openclaw-canvas.sock` konumunda veya `XDG_RUNTIME_DIR` kullanılamadığında `/tmp/openclaw-canvas-$UID.sock` konumunda mevcutsa yapar. `plugins.entries.linux-canvas.enabled: false` ile devre dışı bırakın. Masaüstü uygulaması olmayan başsız bir Linux sunucusunda Canvas duyurulmaz.

Linux v1 tek bir Canvas penceresi kullanır. HTTP ve HTTPS sayfaları işlenebilir, ancak A2UI eylemleri yalnızca paketlenmiş işleyiciden kabul edilir.

## CLI ve SSH alternatifi

CLI, başsız bir sunucu, VPS veya uzak Gateway için en basit seçenek olmaya devam eder:

1. Node 24.15+ (önerilen), Node 22.22.3+ (LTS) veya Node 25.9+ yükleyin.
2. `npm i -g openclaw@latest`
3. `openclaw onboard --install-daemon`
4. Dizüstü bilgisayarınızdan: `ssh -N -L 18789:127.0.0.1:18789 <user>@<host>`
5. `http://127.0.0.1:18789/` adresini açın ve yapılandırılmış paylaşılan
   gizli değerle kimlik doğrulayın (varsayılan olarak token; `gateway.auth.mode`, `"password"` ise parola).

Tam sunucu kılavuzu: [Linux Sunucusu](/tr/vps). Adım adım VPS örneği:
[exe.dev](/tr/install/exe-dev).

## Node yetenekleri

Paketlenmiş Linux Node Plugin'i, masaüstü uygulamasını gerektirmeden CLI'a `openclaw node` hizmet cihazı yeteneklerini sağlar. Komutlar, yalnızca yetenekleri etkinleştirildiğinde ve gerekli yerel araç mevcut olduğunda Gateway'e duyurulur.

| Yetenek                                     | Varsayılan | Gereksinim                                                            |
| ------------------------------------------- | ---------- | --------------------------------------------------------------------- |
| Masaüstü bildirimleri (`system.notify`)  | Açık       | libnotify'dan `notify-send` ve bir masaüstü bildirim oturumu     |
| Kamera fotoğrafları ve klipleri (`camera.*`) | Kapalı | FFmpeg, V4L2 kamera erişimi ve klip sesi için PulseAudio veya PipeWire |
| Konum (`location.get`)                  | Kapalı     | GeoClue2 ve `where-am-i` demosu                                 |

Plugin'i `openclaw.json` içinde yapılandırın:

```json5
{
  plugins: {
    entries: {
      "linux-node": {
        config: {
          notify: { enabled: true },
          camera: { enabled: true },
          location: { enabled: true },
        },
      },
    },
  },
}
```

Bu ayarları değiştirdikten sonra Node hizmetini yeniden başlatın. Kullanılabilirlik işlem başına bir kez belirlenir ve Node duyurusu yeniden başlatmada yeniden oluşturulur.

Gateway, Node'un komut ve yetenek yüzeyini cihaz eşleştirmesinden ayrı olarak onaylar. İlk başlatmada veya daha fazla yeteneği etkinleştirdikten sonra bekleyen yüzeyi onaylayın:

```bash
openclaw nodes pending
openclaw nodes approve <requestId>
```

Bir Node bağlı ve cihazla eşleştirilmiş olabilir ancak etkin `caps` ve `commands` değerleri bu onay tamamlanana kadar boş kalabilir.

Kamera cihazları hizmet kullanıcısı tarafından, genellikle `video` grubu aracılığıyla okunabilir olmalıdır. Kamera klipleri, `includeAudio` true olduğunda varsayılan PulseAudio veya PipeWire kaynağını kullanır; mikrofon sesi bağımsız bir komut olarak değil, yalnızca bu klibin ses parçası olarak bulunur. Konum, Node hizmeti kullanıcısına ana makinenin GeoClue politikası tarafından izin verilmesini gerektirir.

`camera.snap` ve `camera.clip`, ayrıca `gateway.nodes.commands.allow` üzerinden açık Gateway etkinleştirmesi gerektirir. Yükler, sınırlar ve hatalar için [Kamera yakalama](/tr/nodes/camera) ve [Konum komutu](/tr/nodes/location-command) bölümlerine bakın.

## Yükleme

- [Başlarken](/tr/start/getting-started)
- [Yükleme ve güncellemeler](/tr/install/updating)
- İsteğe bağlı: [Bun paket iş akışı](/tr/install/bun), [Nix](/tr/install/nix), [Docker](/tr/install/docker)

## Gateway hizmeti (systemd)

Şunlardan biriyle yükleyin:

```bash
openclaw onboard --install-daemon
openclaw gateway install
openclaw configure   # sorulduğunda "Gateway service" seçeneğini belirleyin
```

Mevcut bir kurulumu onarın veya taşıyın:

```bash
openclaw doctor
```

`openclaw gateway install` varsayılan olarak bir systemd **kullanıcı** birimi oluşturur.
Paylaşılan veya sürekli açık ana makineler için **sistem** düzeyindeki birim çeşidi dahil olmak üzere
tam hizmet yönergeleri [Gateway çalışma kılavuzunda](/tr/gateway#supervision-and-service-lifecycle) yer alır.

Yalnızca özel bir kurulum için birimi elle yazın. Asgari kullanıcı birimi örneği
(`~/.config/systemd/user/openclaw-gateway[-<profile>].service`):

```ini
[Unit]
Description=OpenClaw Gateway (profile: <profile>, v<version>)
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

Elle yazılmış birimler, `openclaw gateway install` tarafından yönetilen Gateway hizmetleri için yazılan uyarlanabilir yığın boyutlandırmasını devralmaz. Yönetilen yükleyiciyi tercih edin veya yerel bellek için yeterli payı hesaba kattıktan sonra özel gözeticide açık bir yığın sınırı belirleyin.

Etkinleştirin:

```bash
systemctl --user enable --now openclaw-gateway[-<profile>].service
```

## Bellek baskısı ve OOM sonlandırmaları

Linux'ta bir ana makine, VM veya konteyner cgroup'unda bellek tükendiğinde çekirdek
bir OOM kurbanı seçer. Gateway, uzun ömürlü
oturumları ve kanal bağlantılarını yönettiği için kötü bir kurbandır; bu nedenle OpenClaw, mümkün olduğunda geçici alt
işlemlerin önce sonlandırılmasına öncelik verir.

Uygun Linux alt işlem başlatmalarında OpenClaw, komutu alt işlemin kendi
`oom_score_adj` değerini `1000` değerine yükselten kısa bir
`/bin/sh` sarmalayıcısıyla çalıştırır, ardından gerçek komutu
`exec` ile çalıştırır. Bu işlem ayrıcalıksızdır: bir işlem kendi OOM puanını her zaman yükseltebilir.

Kapsanan alt işlem yüzeyleri:

- Gözetici tarafından yönetilen komut alt işlemleri
- PTY kabuk alt işlemleri
- MCP stdio sunucusu alt işlemleri
- OpenClaw tarafından başlatılan tarayıcı/Chrome işlemleri (Plugin SDK işlem çalışma zamanı üzerinden)

Sarmalayıcı yalnızca Linux içindir ve `/bin/sh` kullanılamadığında veya
alt işlem ortamı `OPENCLAW_CHILD_OOM_SCORE_ADJ` değerini `0`, `false`, `no` ya da
`off` olarak ayarladığında atlanır.

Bir alt işlemi doğrulayın:

```bash
cat /proc/<child-pid>/oom_score_adj
```

Kapsanan alt işlemler için beklenen değer `1000` şeklindedir; Gateway işleminin kendisi
normal puanını (genellikle `0`) korur.

systemd birimindeki `OOMPolicy=continue`, OOM sonlandırıcısı geçici bir alt işlemi seçtiğinde
tüm birimi başarısız olarak işaretleyip tüm kanalları yeniden başlatmak yerine Gateway hizmetini çalışır durumda tutar;
başarısız olan alt işlem/oturum kendi hatasını bildirir.

Bu, normal bellek ayarlamasının yerini almaz. Bir VPS veya konteyner alt işlemleri tekrar tekrar
sonlandırıyorsa bellek sınırını yükseltin, eş zamanlılığı azaltın veya daha güçlü
kaynak denetimleri (systemd `MemoryMax=`, konteyner bellek sınırları) ekleyin.

## İlgili

- [Kuruluma genel bakış](/tr/install)
- [Linux sunucusu](/tr/vps)
- [Raspberry Pi](/tr/install/raspberry-pi)
- [Gateway işletim kılavuzu](/tr/gateway)
- [Gateway yapılandırması](/tr/gateway/configuration)
