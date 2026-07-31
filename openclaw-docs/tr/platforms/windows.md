---
read_when:
    - OpenClaw'u Windows'a Yükleme
    - Windows Hub, yerel Windows ve WSL2 arasında seçim yapma
    - Windows yardımcı uygulamasını veya Windows Node modunu ayarlama
summary: 'Windows desteği: Windows Hub, yerel CLI ve Gateway, WSL2 Gateway kurulumu, Node modu ve sorun giderme'
title: Windows
x-i18n:
    generated_at: "2026-07-26T23:28:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c231b81971e1df9f3ee4de1b102c25328c242109331c6465dc802ec003af722b
    source_path: platforms/windows.md
    workflow: 16
---

OpenClaw, yerel bir **Windows Hub** yardımcı uygulaması ve Windows CLI desteğiyle birlikte gelir.
Kurulum, sistem tepsisi durumu, sohbet, Command Center tanılamaları ve Windows
Node yetenekleri sunan bir masaüstü uygulaması için Windows Hub'ı kullanın. CLI/Gateway'i
doğrudan kullanmak için PowerShell yükleyicisini kullanın. Linux ile en uyumlu
Gateway çalışma zamanı için WSL2 kullanın.

## Önerilen: Windows Hub

Windows Hub, Windows 10 20H2+ ve Windows 11 için yerel WinUI yardımcı uygulamasıdır.
Yönetici ayrıcalıkları olmadan yüklenir ve kendi sürüm sayfasında imzalı x64
ve ARM64 yükleyicileri sunar.

Windows Hub, OpenClaw CLI ve Gateway'den bağımsız olarak yayımlanır. En son
kararlı Hub yükleyicisini
[Windows Hub sürümleri sayfasından](https://github.com/openclaw/openclaw-windows-node/releases/latest)
veya doğrudan `releases/latest/download` aracılığıyla indirin:

- [OpenClawCompanion-Setup-x64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-x64.exe)
- [OpenClawCompanion-Setup-arm64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-arm64.exe)

Yukarıdaki bağlantılardan biri 404 hatası verirse [Windows Hub sürümleri sayfasını](https://github.com/openclaw/openclaw-windows-node/releases)
ziyaret edin ve en yeni kararlı Windows Hub sürümünü açın. Normal kararlı OpenClaw sürümleri
ayrıca sabitlenmiş ve sürüm doğrulamasından geçmiş bir Windows Hub derlemesini yansıtır; bu
yansıma, daha yeni bağımsız bir Hub sürümünün gerisinde kalabilir.

Yüklemeden sonra Başlat menüsünden veya sistem tepsisinden **OpenClaw Companion** uygulamasını
başlatın. Yükleyici ayrıca Gateway Setup, Chat, Settings,
Check for Updates ve kaldırma için kısayollar ekler.

### Windows Hub'ın içerdikleri

- Sistem tepsisi durumu ve oturum açıldığında başlatma.
- Uygulamaya ait yerel bir WSL Gateway için ilk çalıştırma kurulumu.
- Yerel, uzak ve SSH tünelli Gateway'ler için bağlantı ayarları.
- Yerel sohbet penceresi ve tarayıcıdaki Control UI'a erişim.
- Oturumlar, kullanım, kanallar, Node'lar, eşleştirme
  ve onarım komutları için Command Center tanılamaları.
- Aracı tarafından denetlenen tuval, ekran, kamera,
  bildirimler, cihaz durumu, konuşma ve denetimli `system.run` için Windows Node modu.
- Claude Desktop, Claude Code
  ve Cursor gibi MCP istemcileri için yerel MCP sunucusu modu.

### İlk başlatma

İlk başlatmada Windows Hub, kullanılabilir kayıtlı bir
Gateway yoksa kurulumu açar. En hızlı yol, uygulamaya ait bir
`OpenClawGateway` WSL dağıtımı hazırlayan, Gateway'i bunun içine yükleyen ve
uygulamayı eşleştiren **Set up locally** seçeneğidir. Bu işlem mevcut Ubuntu dağıtımınızı dışa aktarmaz
veya değiştirmez.

Zaten bir Gateway'iniz varsa **Advanced setup** seçeneğini belirleyin veya Connections sekmesini açın.
Şunlara bağlanabilirsiniz:

- bu bilgisayardaki yerel bir Gateway
- bu bilgisayardaki bir WSL Gateway
- URL ve token ya da kurulum koduyla uzak bir Gateway
- SSH tüneli üzerinden erişilen bir Gateway

Kurulum tamamlandığında sistem tepsisi simgesi yeşile döner. Bağlantıyı, eşleştirmeyi,
Node durumunu ve kanal sağlığını doğrulamak için sistem tepsisinden **Command Center**'ı açın.

## Windows Node modu

Windows Hub, aracının bildirilen Windows'a özgü yetenekleri Gateway üzerinden
kullanabilmesi için bir OpenClaw Node'u olarak kaydolabilir. Node komutlarının
çalıştırılmadan önce Node tarafından bildirilmesi ve Gateway ilkesi tarafından izin verilmesi
gerekir; izin verme/reddetme modelinin tamamı için
[Node'lar](/tr/nodes#command-policy) bölümüne bakın.

Yaygın komutlar:

| Aile   | Komutlar                                                                             |
| ------ | ------------------------------------------------------------------------------------ |
| Tuval  | `canvas.present`, `canvas.hide`, `canvas.navigate`, `canvas.eval`, `canvas.snapshot` |
| Ekran  | `screen.snapshot`; `screen.record` açıkça etkinleştirme gerektirir                          |
| Kamera | `camera.list`; `camera.snap`, `camera.clip` açıkça etkinleştirme gerektirir                  |
| Sistem | `system.notify`, `system.run`, `system.run.prepare`, `system.which`                  |
| Cihaz  | `location.get`, `device.info`, `device.status`                                       |
| Konuşma | `talk.ptt.start`, `talk.ptt.stop`, `talk.ptt.cancel`, `talk.ptt.once`, `talk.speak`  |

Node modu Gateway eşleştirmesi gerektirir. Uygulama bir eşleştirme isteği gösterirse
Gateway ana makinesinden onaylayın:

```powershell
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Gateway yalnızca Node'un bildirdiği ve sunucu ilkesinin
izin verdiği komutları iletir. `screen.record`, `camera.snap`
ve `camera.clip` gibi gizliliğe duyarlı komutlar açık `gateway.nodes.commands.allow` etkinleştirmesi gerektirir.

## Yerel MCP modu

Windows Hub, aynı Windows'a özgü yetenek kayıt defterini geri döngü üzerinde yerel bir
MCP sunucusu olarak kullanıma sunabilir; böylece yerel MCP istemcileri, çalışan bir
OpenClaw Gateway olmadan Windows yeteneklerini kullanabilir.

Windows Hub Settings içindeki developer/advanced bölümünden etkinleştirin. Sunucu
etkinleştirildiğinde uygulama geri döngü uç noktasını ve bearer token'ı gösterir.

Mod matrisi:

| Node modu | MCP sunucusu | Davranış                           |
| --------- | ------------ | ---------------------------------- |
| kapalı    | kapalı       | Yalnızca operatöre yönelik masaüstü uygulaması |
| açık      | kapalı       | Gateway'e bağlı Windows Node'u     |
| kapalı    | açık         | Yalnızca yerel MCP sunucusu        |
| açık      | açık         | Gateway Node'u ve yerel MCP sunucusu |

## Yerel Windows CLI ve Gateway

Terminal odaklı kullanım için OpenClaw'ı PowerShell'den yükleyin:

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

Doğrulayın:

```powershell
openclaw --version
openclaw doctor
openclaw gateway status --json
```

Yönetilen başlatma, kullanılabilir olduğunda Windows Scheduled Tasks kullanır. Görev,
okunabilir `gateway.cmd` betiğini OpenClaw durum dizininde tutar ancak
oluşturulan bir `gateway.vbs` WScript sarmalayıcısı üzerinden başlatır; böylece arka plandaki Gateway
görünür bir konsol penceresi açmaz. Görev oluşturma reddedilirse OpenClaw,
kullanıcı başına Startup klasörü oturum açma öğesine geri döner.

Gateway hizmetini yükleyin:

```powershell
openclaw gateway install
openclaw gateway status --json
```

Yönetilen bir Gateway hizmeti olmadan yalnızca CLI kullanımı için:

```powershell
openclaw onboard --non-interactive --skip-health
openclaw gateway run
```

## WSL2 Gateway

WSL2, Windows'ta Linux ile en uyumlu Gateway çalışma zamanı olmayı sürdürür. Windows
Hub sizin için uygulamaya ait bir WSL Gateway kurabilir veya kendi dağıtımınızın
içine elle yükleyebilirsiniz.

Elle kurulum:

```powershell
wsl --install
# Veya açıkça bir dağıtım seçin:
wsl --list --online
wsl --install -d Ubuntu-24.04
```

WSL içinde systemd'yi etkinleştirin:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

WSL'yi PowerShell'den yeniden başlatın:

```powershell
wsl --shutdown
```

Ardından Linux hızlı başlangıç yöntemiyle OpenClaw'ı WSL içine yükleyin:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw gateway status
```

## Windows oturum açma işleminden önce Gateway'i otomatik başlatma

Ekransız WSL kurulumlarında, Windows'ta kimse oturum açmasa bile tam önyükleme
zincirinin çalıştığından emin olun.

WSL içinde:

```bash
sudo apt-get install -y dbus-x11
sudo loginctl enable-linger "$(whoami)"
openclaw gateway install
```

Yönetici olarak PowerShell'de:

```powershell
schtasks /create /tn "WSL Boot" /tr "wsl.exe -d Ubuntu --exec dbus-launch true" /sc onstart /ru "$env:USERNAME"
```

`Ubuntu` değerini şu komuttan aldığınız dağıtım adıyla değiştirin:

```powershell
wsl --list --verbose
```

<Note>
Eski yöntemlere göre iki değişiklik:

- **`dbus-launch true` yerine `/bin/true`**: WSL >= 2.6.1.0 sürümünde bir
  regresyon ([microsoft/WSL #13416](https://github.com/microsoft/WSL/issues/13416)),
  linger etkin olsa bile son istemci çıktıktan 15-20 saniye sonra
  dağıtımı boşta olduğu için sonlandırır. Geçici çözüm olarak `dbus-launch true`,
  init alt sürecini çalışır durumda tutar (topluluk tartışması, [microsoft/WSL #9245](https://github.com/microsoft/WSL/discussions/9245)).
- **`/ru "$env:USERNAME"` yerine `/ru SYSTEM`**: kullanıcı başına WSL dağıtımları
  (varsayılan kurulum) SYSTEM hesabı tarafından görülemez; bu nedenle görev çalışıyor
  görünür ancak dağıtım hiçbir zaman başlamaz. Kendi hesabınızla çalıştırmak
  bunu önler; görev oluşturulurken Windows parolanızı ister.

</Note>

Yeniden başlatmanın ardından WSL'den doğrulayın:

```bash
systemctl --user is-enabled openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

## WSL hizmetlerini LAN üzerinden kullanıma sunma

WSL'nin kendi sanal ağı vardır. Başka bir makinenin WSL içindeki bir hizmete
erişmesi gerekiyorsa bir Windows bağlantı noktasını geçerli WSL IP'sine yönlendirin. WSL IP'si
yeniden başlatmalardan sonra değişebilir; bu nedenle gerektiğinde yönlendirme kuralını yenileyin.

Yönetici olarak PowerShell örneği:

```powershell
$Distro = "Ubuntu-24.04"
$ListenPort = 2222
$TargetPort = 22

$WslIp = (wsl -d $Distro -- hostname -I).Trim().Split(" ")[0]
if (-not $WslIp) { throw "WSL IP bulunamadı." }

netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=$ListenPort `
  connectaddress=$WslIp connectport=$TargetPort

New-NetFirewallRule -DisplayName "WSL SSH $ListenPort" -Direction Inbound `
  -Protocol TCP -LocalPort $ListenPort -Action Allow
```

Notlar:

- Başka bir makineden SSH, Windows ana makinesinin IP'sini hedefler; ör. `ssh user@windows-host -p 2222`.
- Uzak Node'lar `127.0.0.1` yerine erişilebilir bir Gateway URL'sini göstermelidir.
- LAN erişimi için `listenaddress=0.0.0.0`, yalnızca yerel erişim için `127.0.0.1` kullanın.

## Sorun giderme

### Sistem tepsisi simgesi görünmüyor

Görev Yöneticisi'nde `OpenClaw.Tray.WinUI.exe` işlemini kontrol edin. Çalışıyorsa
gizli sistem tepsisi simgeleri alanını açıp sabitleyin. Çalışmıyorsa Başlat
menüsünden **OpenClaw Companion** uygulamasını başlatın.

### Yerel kurulum başarısız oluyor

Kurulum günlüğünü Windows Hub'dan açın veya şunu inceleyin:

```powershell
notepad "$env:LOCALAPPDATA\OpenClawTray\Logs\Setup\easy-setup-latest.txt"
```

Yaygın nedenler: devre dışı WSL, engellenmiş sanallaştırma, uygulamaya ait eski WSL
durumu veya Gateway paketi yüklenirken oluşan bir ağ hatası.

### Uygulama eşleştirme gerektiğini söylüyor

Operatör veya Node isteğini Gateway'den onaylayın:

```powershell
openclaw devices list
openclaw devices approve <requestId>
```

Cihazın zaten bir token'ı varsa onaydan sonra Connections sekmesinden
yeniden bağlanın.

### Web sohbeti uzak bir Gateway'e erişemiyor

Uzak web sohbeti HTTPS veya localhost gerektirir. Kendinden imzalı sertifikalar için
sertifikaya Windows'ta güvenin veya bir localhost URL'sine SSH tüneli kullanın.

### `screen.snapshot`, kamera veya ses komutları başarısız oluyor

Kamera, mikrofon, ekran yakalama ve bildirimler için Windows izinlerini
doğrulayın. Paketlenmiş yüklemeler korunan yetenekleri bildirir ancak
bir komut bunları ilk kez kullandığında Windows yine de istem gösterebilir.

### Git veya GitHub bağlantısı başarısız oluyor

Bazı ağlar GitHub'a HTTPS erişimini engeller veya yavaşlatır. `git clone` ya da
`gh auth login` başarısız olursa başka bir ağ, VPN veya HTTP/HTTPS proxy deneyin.

Geçerli oturumda token tabanlı `gh` kimlik doğrulaması için:

```powershell
$env:GH_TOKEN="<your-token>"
gh auth status
gh auth setup-git
```

Token'ları asla commit etmeyin veya issue'lara ya da pull request'lere yapıştırmayın.

## İlgili

- [Yüklemeye genel bakış](/tr/install)
- [Node.js kurulumu](/tr/install/node)
- [Node'lar](/tr/nodes)
- [Control UI](/tr/web/control-ui)
- [Gateway yapılandırması](/tr/gateway/configuration)
