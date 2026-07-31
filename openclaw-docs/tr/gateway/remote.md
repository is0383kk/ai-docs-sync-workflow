---
read_when:
    - Uzak Gateway kurulumlarını çalıştırma veya sorunlarını giderme
summary: Gateway WS, SSH tünelleri ve tailnet'ler kullanarak uzaktan erişim
title: Uzaktan erişim
x-i18n:
    generated_at: "2026-07-26T22:47:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8f05e32fcfa16d5ddfcd684d0550c9af311914e2b4d91c95edad3490dc2e56d9
    source_path: gateway/remote.md
    workflow: 16
---

OpenClaw, bir ana makinede bir Gateway (yönetici) çalıştırır ve her istemciyi ona bağlar. Gateway; oturumların, kimlik doğrulama profillerinin, kanalların ve durumun sahibidir; diğer her şey bir istemcidir.

- **Operatörler** (siz veya macOS uygulaması): Gateway erişilebilir durumdaysa doğrudan LAN/Tailnet WebSocket en basit seçenektir; SSH tünelleme evrensel yedek seçenektir.
- **Node'lar** (iOS/Android ve diğer cihazlar): Gateway **WebSocket**'ine (LAN/tailnet veya SSH tüneli) bağlanır.

## Temel fikir

Gateway WebSocket, varsayılan olarak `18789` (`gateway.port`) portunda **geri döngü** arabirimine bağlanır. Uzaktan kullanım için ya Tailscale Serve / güvenilir bir LAN-Tailnet bağlantısı üzerinden erişime açın ya da geri döngü portunu SSH üzerinden yönlendirin.

## Topoloji seçenekleri

| Kurulum                           | Gateway'in çalıştığı yer                                                                                   | En uygun kullanım                                                                                                                                 |
| --------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Tailnet'inizde sürekli açık Gateway | Tailscale veya SSH üzerinden erişilen kalıcı ana makine (VPS veya ev sunucusu)                           | Sık sık uyku moduna geçen ancak aracının sürekli açık olması gereken dizüstü bilgisayarlar. [exe.dev](/tr/install/exe-dev) (kolay VM) veya [Hetzner](/tr/install/hetzner) (üretim VPS'si) sayfasına bakın. |
| Ev masaüstü bilgisayarı           | Masaüstü bilgisayar; dizüstü bilgisayar macOS uygulamasının uzak modu (Settings → Connection → OpenClaw runs) üzerinden bağlanır | Aracıyı sürekli açık kalan donanımda tutmak. Çalıştırma kılavuzu: [macOS uzaktan erişimi](/tr/platforms/mac/remote).                                    |
| Dizüstü bilgisayar                | SSH tüneli veya Tailscale Serve üzerinden güvenli biçimde erişime açılan dizüstü bilgisayar (`gateway.bind: "loopback"` ayarını koruyun) | Tek makineli kurulumlar. [Tailscale](/tr/gateway/tailscale) ve [Web](/tr/web) sayfalarına bakın.                                                         |

Sürekli açık ve dizüstü bilgisayar kurulumlarında `gateway.bind: "loopback"` ayarını koruyup Control UI için **Tailscale Serve** kullanmayı veya `gateway.remote.transport: "direct"` ile güvenilir bir LAN/Tailnet bağlantısını tercih edin. SSH tüneli, herhangi bir makineden çalışan yedek seçenektir.

## Komut akışı (ne, nerede çalışır)

Durumun ve kanalların sahibi tek bir Gateway'dir; Node'lar çevre birimleridir. Örnek (bir Node aracına yönlendirilen Telegram mesajı):

1. Telegram mesajı **Gateway**'e ulaşır.
2. Gateway, bir Node aracının çağrılıp çağrılmayacağına karar veren **aracıyı** çalıştırır.
3. Gateway, Gateway WebSocket üzerinden (`node.invoke` RPC) **Node**'u çağırır.
4. Node sonucu döndürür; Gateway Telegram'a yanıt verir.

Node'lar Gateway hizmetini çalıştırmaz. Bilerek yalıtılmış profiller çalıştırmadığınız sürece ana makine başına yalnızca bir Gateway çalışmalıdır ([Birden fazla Gateway](/tr/gateway/multiple-gateways) sayfasına bakın). macOS uygulamasındaki "Node modu", yalnızca Gateway WebSocket üzerinden çalışan bir Node istemcisidir.

## SSH tüneli (CLI + araçlar)

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

Tünel açıkken `openclaw health` ve `openclaw status --deep`, `ws://127.0.0.1:18789` üzerinden uzaktaki Gateway'e erişir. `openclaw gateway status`, `openclaw gateway health`, `openclaw gateway probe` ve `openclaw gateway call` da `--url` aracılığıyla yönlendirilmiş bir URL'yi hedefleyebilir.

<Note>
`18789` değerini yapılandırılmış `gateway.port` değerinizle (veya `--port` / `OPENCLAW_GATEWAY_PORT`) değiştirin.
</Note>

<Warning>
`--url`, hiçbir zaman yapılandırma veya ortam kimlik bilgilerine geri dönmez. `--token` veya `--password` değerini açıkça iletin; bunlar olmadan istemci hiçbir kimlik bilgisi göndermez ve hedef Gateway kimlik doğrulaması gerektiriyorsa bağlantı başarısız olur.
</Warning>

## CLI uzak bağlantı varsayılanları

CLI komutlarının varsayılan olarak kullanması için uzak hedefi kalıcı hâle getirin:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",
      token: "your-token",
    },
  },
}
```

Gateway yalnızca geri döngüdeyse URL'yi `ws://127.0.0.1:18789` olarak bırakın ve önce SSH tünelini açın. macOS uygulamasının SSH tüneli aktarımında keşfedilen Gateway ana makine adı `gateway.remote.sshTarget` alanına (`user@host` veya `user@host:port`) girilir; `gateway.remote.url` yerel tünel URL'si olarak kalır. Uzak port yerel porttan farklıysa `gateway.remote.remotePort` ayarını belirleyin.

Ana makine anahtarı doğrulaması varsayılan olarak katıdır (`gateway.remote.sshHostKeyPolicy: "strict"`). Bunun yerine etkin OpenSSH yapılandırmanıza devretmek için değeri `"openssh"` olarak ayarlayın; etkinleştirmeden önce kullanıcı ve sistem SSH ayarlarınızı gözden geçirin.

Güvenilir bir LAN veya Tailnet üzerinden zaten erişilebilen bir Gateway için doğrudan modu kullanın:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      transport: "direct",
      url: "ws://192.168.0.202:18789",
      token: "your-token",
    },
  },
}
```

## Kimlik bilgisi önceliği

Gateway kimlik bilgisi çözümlemesi; çağrı/yoklama/durum yollarında ve Discord çalıştırma onayı izlemesinde ortak bir sözleşmeyi izler. Node ana makinesi de tek bir yerel mod istisnasıyla aynı sözleşmeyi kullanır (`gateway.remote.*` değerini yok sayar).

- Açık kimlik bilgileri (`--token`, `--password` veya bir aracın `gatewayToken` değeri), açık kimlik doğrulamasını kabul eden çağrı yollarında her zaman önceliklidir.
- URL geçersiz kılma güvenliği:
  - CLI `--url`, örtük yapılandırma/ortam kimlik bilgilerini hiçbir zaman yeniden kullanmaz.
  - Ortam `OPENCLAW_GATEWAY_URL`, yalnızca ortam kimlik bilgilerini (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`) kullanabilir.
- Yerel mod varsayılanları:
  - token: `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token` -> `gateway.remote.token` (uzak yedek yalnızca yerel token ayarlanmamışsa kullanılır)
  - parola: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.auth.password` -> `gateway.remote.password` (uzak yedek yalnızca yerel parola ayarlanmamışsa kullanılır)
- Uzak mod varsayılanları:
  - token: `gateway.remote.token` -> `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token`
  - parola: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.remote.password` -> `gateway.auth.password`
- Node ana makinesi yerel mod istisnası: `gateway.remote.token` / `gateway.remote.password` yok sayılır.
- Uzak yoklama/durum token denetimleri varsayılan olarak katıdır: uzak mod hedeflenirken yalnızca `gateway.remote.token` kullanılır (yerel token yedeği yoktur).
- Gateway ortam geçersiz kılmaları yalnızca `OPENCLAW_GATEWAY_*` kullanır.

## Sohbet kullanıcı arayüzüne uzaktan erişim

WebChat'in ayrı bir HTTP portu yoktur; SwiftUI sohbet kullanıcı arayüzü doğrudan Gateway WebSocket'e bağlanır.

- `18789` portunu SSH üzerinden yönlendirin (yukarıya bakın), ardından istemcileri `ws://127.0.0.1:18789` adresine bağlayın.
- LAN/Tailnet doğrudan modu için istemcileri yapılandırılmış özel `ws://` veya güvenli `wss://` URL'sine bağlayın.
- macOS'ta uygulamanın uzak modu, seçilen aktarımı otomatik olarak yönetir.

## macOS uygulaması uzak modu

macOS menü çubuğu uygulaması aynı kurulumu uçtan uca yürütür: uzak durum denetimleri, WebChat ve Voice Wake yönlendirmesi. Çalıştırma kılavuzu: [macOS uzaktan erişimi](/tr/platforms/mac/remote).

## Güvenlik kuralları (uzak/VPN)

Bağlantı açmanız gerektiğinden emin olmadığınız sürece Gateway'i **yalnızca geri döngüde** tutun.

- **Geri döngü + SSH/Tailscale Serve** en güvenli varsayılandır (herkese açık erişim yoktur).
- Düz metin `ws://`; geri döngü, özel/LAN (RFC 1918), bağlantı-yerel, CGNAT, `.local` ve `.ts.net` ana makineleri için kabul edilir. Herkese açık uzak ana makineler `wss://` kullanmalıdır.
- **Geri döngü dışı bağlantılar** (`lan`/`tailnet`/`custom` veya geri döngü kullanılamadığında `auto`) Gateway kimlik doğrulaması kullanmalıdır: token, parola veya `gateway.auth.mode: "trusted-proxy"` ile kimlik farkındalığına sahip ters proxy.
- `gateway.remote.token` / `.password` istemci kimlik bilgisi kaynaklarıdır; sunucu kimlik doğrulamasını kendi başlarına yapılandırmazlar.
- Yerel çağrı yolları, yalnızca `gateway.auth.*` ayarlanmamışsa `gateway.remote.*` değerini yedek olarak kullanabilir.
- `gateway.auth.token` / `gateway.auth.password` SecretRef üzerinden açıkça yapılandırılmış ancak çözümlenememişse çözümleme güvenli biçimde başarısız olur (uzak yedek başarısızlığı gizlemez).
- `gateway.remote.tlsFingerprint`, hem operatör/kontrol trafiği hem de macOS doğrudan modundaki eşlikçi Node dâhil olmak üzere `wss://` için uzak TLS sertifikasını sabitler. Kayıtlı bir sabitleme olmadan macOS, yalnızca normal sistem güven denetimi başarılı olduktan sonra ilk kullanımda sabitler; kendinden imzalı veya özel CA kullanan Gateway'ler açık bir parmak izi ya da SSH üzerinden uzak bağlantı gerektirir.
- `gateway.auth.allowTailscale: true` olduğunda **Tailscale Serve**, Control UI/WebSocket trafiğinin kimliğini kimlik başlıkları aracılığıyla doğrulayabilir. HTTP API uç noktaları bu başlık kimlik doğrulamasını kullanmaz; bunun yerine Gateway'in normal HTTP kimlik doğrulama modunu izler. Bu tokensız akış, Gateway ana makinesinin güvenilir olduğunu varsayar; her yerde paylaşılan gizli anahtar kimlik doğrulaması için değeri `false` olarak ayarlayın.
- **Güvenilir proxy** kimlik doğrulaması varsayılan olarak geri döngü dışındaki kimlik farkındalığına sahip bir proxy bekler. Aynı ana makinedeki geri döngü ters proxy'leri açıkça `gateway.auth.trustedProxy.allowLoopback = true` gerektirir.
- Tarayıcı denetimini operatör erişimi gibi değerlendirin: yalnızca tailnet ve bilinçli Node eşleştirmesi.

Ayrıntılı inceleme: [Güvenlik](/tr/gateway/security).

### macOS: LaunchAgent üzerinden kalıcı SSH tüneli

macOS istemcileri için en kolay kalıcı kurulum, bir SSH `LocalForward` yapılandırma girdisiyle yeniden başlatmalar ve çökmeler boyunca tüneli açık tutan bir LaunchAgent kullanır.

#### 1. adım: SSH yapılandırmasını ekleyin

`~/.ssh/config` dosyasını düzenleyin:

```ssh
Host remote-gateway
    HostName <REMOTE_IP>
    User <REMOTE_USER>
    LocalForward 18789 127.0.0.1:18789
    IdentityFile ~/.ssh/id_rsa
```

`<REMOTE_IP>` ve `<REMOTE_USER>` değerlerini kendi değerlerinizle değiştirin.

#### 2. adım: SSH anahtarını kopyalayın (bir kez)

```bash
ssh-copy-id -i ~/.ssh/id_rsa <REMOTE_USER>@<REMOTE_IP>
```

#### 3. adım: Gateway token'ını yapılandırın

```bash
openclaw config set gateway.remote.token "<your-token>"
```

Uzak Gateway parola kimlik doğrulaması kullanıyorsa bunun yerine `gateway.remote.password` kullanın. `OPENCLAW_GATEWAY_TOKEN` kabuk düzeyinde geçersiz kılma olarak hâlâ geçerlidir, ancak kalıcı uzak istemci kurulumu `gateway.remote.token` / `gateway.remote.password` şeklindedir.

#### 4. adım: LaunchAgent'ı oluşturun

`~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist` olarak kaydedin:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.ssh-tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/ssh</string>
        <string>-N</string>
        <string>remote-gateway</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

#### 5. adım: LaunchAgent'ı yükleyin

```bash
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist
```

Tünel oturum açıldığında otomatik olarak başlar, çökme durumunda yeniden başlatılır ve yönlendirilmiş portu etkin tutar.

<Note>
Eski bir kurulumdan kalan `com.openclaw.ssh-tunnel` LaunchAgent'ınız varsa kaldırın ve silin.
</Note>

#### Sorun giderme

```bash
# Tünelin çalışıp çalışmadığını denetleyin
ps aux | grep "ssh -N remote-gateway" | grep -v grep
lsof -i :18789

# Tüneli yeniden başlatın
launchctl kickstart -k gui/$UID/ai.openclaw.ssh-tunnel

# Tüneli durdurun
launchctl bootout gui/$UID/ai.openclaw.ssh-tunnel
```

| Yapılandırma girdisi                 | İşlevi                                                       |
| ------------------------------------ | ------------------------------------------------------------ |
| `LocalForward 18789 127.0.0.1:18789` | Yerel 18789 numaralı bağlantı noktasını uzak 18789 numaralı bağlantı noktasına yönlendirir |
| `ssh -N`                             | Uzak komutları çalıştırmadan SSH bağlantısı kurar (yalnızca bağlantı noktası yönlendirme) |
| `KeepAlive`                          | Tünel çökerse otomatik olarak yeniden başlatır               |
| `RunAtLoad`                          | Oturum açıldığında LaunchAgent yüklenirken tüneli başlatır   |

## İlgili

- [Tailscale](/tr/gateway/tailscale)
- [Kimlik doğrulama](/tr/gateway/authentication)
- [Uzak Gateway kurulumu](/tr/gateway/remote-gateway-readme)
