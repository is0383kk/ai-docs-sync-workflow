---
read_when:
    - Chrome Windows’ta çalışırken OpenClaw Gateway’i WSL2’de çalıştırma
    - WSL2 ve Windows genelinde örtüşen tarayıcı/control-ui hatalarının görülmesi
    - Bölünmüş ana makine kurulumlarında ana makineye yerel Chrome MCP ile ham uzak CDP arasında karar verme
summary: WSL2 Gateway + Windows Chrome uzak CDP sorunlarını katmanlar hâlinde giderme
title: WSL2 + Windows + uzak Chrome CDP sorun giderme
x-i18n:
    generated_at: "2026-07-26T23:03:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 66ec4ed5bfccc66b594a43d56296c69242e8b9cf50b36c6cb3990b1d6ea58faa
    source_path: tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
    workflow: 16
---

Yaygın bölünmüş ana bilgisayar kurulumunda OpenClaw Gateway WSL2 içinde, Chrome ise
Windows'ta çalışır ve tarayıcı denetiminin WSL2/Windows sınırını aşması gerekir. Birkaç
bağımsız sorun aynı anda ortaya çıkabilir (bkz.
[sorun #39369](https://github.com/openclaw/openclaw/issues/39369)): CDP
aktarımı, Control UI kaynak güvenliği ve belirteç/eşleştirme ayrı ayrı
başarısız olabilir ve benzer görünen hatalar üretebilir. Hangi bileşenin bozuk
olduğunu tahmin etmek yerine aşağıdaki katmanları sırayla inceleyin.

## Önce doğru tarayıcı modunu seçin

### Seçenek 1: WSL2'den Windows'a ham uzak CDP

WSL2'den Windows Chrome CDP uç noktasına yönelen bir uzak tarayıcı profili
kullanın. Gateway WSL2 içinde kalıyor, Chrome Windows'ta çalışıyor ve tarayıcı
denetiminin WSL2/Windows sınırını aşması gerekiyorsa bunu seçin.

### Seçenek 2: ana bilgisayar yerel Chrome MCP

`existing-session` sürücüsünü (`user` profili) yalnızca Gateway
Chrome ile aynı ana bilgisayarda çalışıyorsa, yerel oturum açılmış tarayıcı
durumunu kullanmak istiyorsanız, ana bilgisayarlar arası tarayıcı aktarımına
ihtiyacınız yoksa ve `responsebody`, PDF dışa aktarma, indirme müdahalesi
veya toplu eylemlere ihtiyacınız yoksa kullanın (Chrome MCP profilleri bunları
desteklemez).

WSL2 Gateway + Windows Chrome için ham uzak CDP kullanın. Chrome MCP,
WSL2'den Windows'a bir köprü değil, ana bilgisayar yerelidir.

## Çalışan mimari

- WSL2, Gateway'i `127.0.0.1:18789` üzerinde çalıştırır
- Windows, Control UI'ı normal bir tarayıcıda `http://127.0.0.1:18789/` adresinde açar
- Windows Chrome, `9222` bağlantı noktasında bir CDP uç noktası sunar
- WSL2 bu Windows CDP uç noktasına erişebilir
- OpenClaw, bir tarayıcı profilini WSL2'den erişilebilen adrese yönlendirir

## Control UI için kritik kural

UI Windows'tan açıldığında, bilinçli olarak yapılandırılmış bir HTTPS
kurulumunuz yoksa Windows localhost'u kullanın:

```text
http://127.0.0.1:18789/
```

Varsayılan olarak bir LAN IP'si kullanmayın. Bir LAN veya tailnet adresinde
düz HTTP kullanmak, CDP'nin kendisiyle ilgisi olmayan güvenli olmayan
kaynak/cihaz kimlik doğrulama davranışını tetikleyebilir. Bkz.
[Control UI](/tr/web/control-ui).

## Katmanlar hâlinde doğrulayın

Yukarıdan aşağıya ilerleyin; sonraki adımlara atlamayın. Bir katmanı düzeltmek,
daha aşağıdaki başka bir katmandan kaynaklanan farklı bir hatanın görünür
kalmasına engel olmayabilir.

### Katman 1: Chrome'un Windows'ta CDP sunduğunu doğrulayın

```powershell
chrome.exe --remote-debugging-port=9222 --user-data-dir="$env:LOCALAPPDATA\OpenClaw\ChromeCDP"
```

Chrome 136 ve sonraki sürümler, varsayılan Chrome veri dizini için uzaktan
hata ayıklama komut satırı anahtarlarını yok sayar. Yukarıda gösterildiği gibi
ayrı ve varsayılan olmayan bir veri dizini kullanın. Chrome'un
[uzaktan hata ayıklama güvenlik değişikliğine](https://developer.chrome.com/blog/remote-debugging-port)
bakın. Bu, normal oturum açılmış Chrome profilini uzaktan denetlenebilir hâle
getirmez.

Önce Windows'tan Chrome'un kendisini doğrulayın:

```powershell
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://127.0.0.1:9222/json/list
```

Bu başarısız olursa aşağıdaki Windows dinleyicilerini tanılayın. Bu aşamada
sorun henüz OpenClaw değildir.

#### portproxy'yi değiştirmeden önce IPv4 ve IPv6'yı tanılayın

Chromium, uzaktan hata ayıklamayı önce `127.0.0.1` adresine bağlamayı
dener ve yalnızca IPv4 bağlama işlemi başarısız olursa `[::1]`
adresine geri döner. `127.0.0.1:9222` üzerinde dinleyen kalıcı bir
`v4tov4` kuralı, Chrome başlamadan önce bu uç noktayı işgal edebilir.
Chrome daha sonra `[::1]:9222` adresine geri dönerken eski kural IPv4
trafiğini kendi dinleyicisine geri yönlendirir ve boş yanıt döndürür.

Chrome sürümünden çıkarım yapmak yerine gerçek dinleyicileri ve proxy
kurallarını Windows'tan kontrol edin:

```powershell
netstat -ano | findstr :9222
netsh interface portproxy show all
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://[::1]:9222/json/version
```

`netstat` içindeki her PID için `tasklist /fi "PID eq <PID>"` kullanın.

- `chrome.exe`, `127.0.0.1` üzerinde yanıt veriyorsa aynı zamanda
  `127.0.0.1:9222` üzerinde dinleyen tüm portproxy kurallarını kaldırın.
  Yalnızca WSL2'den erişilebilen Windows bağdaştırıcı adresini
  `127.0.0.1` adresine yönlendirin.
- `chrome.exe` yalnızca `[::1]` üzerinde yanıt veriyorsa
  kullanılmayan bir IPv4 adresine yönlendirmek yerine WSL2'den erişilebilen
  dinleyiciyi `v4tov6` kullanarak `::1` adresine yönlendirin:

  ```powershell
  netsh interface portproxy add v4tov6 listenaddress=WINDOWS_HOST_OR_IP listenport=9222 connectaddress=::1 connectport=9222
  ```

Dinleyiciyi WSL2'nin ihtiyaç duyduğu bağdaştırıcı adresine bağlayın. CDP
bağlantı noktasını `0.0.0.0`, bir LAN adresi veya bir tailnet adresi
üzerinde açığa çıkarmayın: CDP, tarayıcı oturumunun denetimini sağlar.

### Katman 2: WSL2'nin bu Windows uç noktasına erişebildiğini doğrulayın

WSL2'den, `cdpUrl` içinde kullanmayı planladığınız tam adresi test
edin:

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

Başarılı sonuç:

- `/json/version`, Browser / Protocol-Version meta verilerini içeren JSON döndürür
- `/json/list`, JSON döndürür (hiç sayfa açık değilse boş bir dizi kabul edilir)

Bu başarısız olursa Windows henüz bağlantı noktasını WSL2'ye açmıyordur, adres
WSL2 tarafı için yanlıştır veya güvenlik duvarı/bağlantı noktası
yönlendirmesi/proxy yapılandırması eksiktir. OpenClaw yapılandırmasına
dokunmadan önce bunu düzeltin.

### Katman 3: doğru tarayıcı profilini yapılandırın

OpenClaw'ı WSL2'den erişilebilen adrese yönlendirin:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "remote",
    profiles: {
      remote: {
        cdpUrl: "http://WINDOWS_HOST_OR_IP:9222",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

Notlar:

- yalnızca Windows'ta çalışan bir adresi değil, WSL2'den erişilebilen adresi kullanın
- haricen yönetilen tarayıcılar için `attachOnly: true` değerini koruyun
- `cdpUrl`; `http://`, `https://`, `ws://` veya `wss://` olabilir
- OpenClaw'ın `/json/version` öğesini keşfetmesini istediğinizde HTTP(S) kullanın
- yalnızca tarayıcı sağlayıcısı doğrudan bir DevTools
  soket URL'si veriyorsa WS(S) kullanın
- OpenClaw'ın başarılı olmasını beklemeden önce aynı URL'yi `curl` ile test edin

### Katman 4: Control UI katmanını ayrıca doğrulayın

Windows'tan `http://127.0.0.1:18789/` adresini açın, ardından şunları doğrulayın:

- sayfa kaynağı, `gateway.controlUi.allowedOrigins` tarafından beklenen değerle eşleşiyor
- belirteç kimlik doğrulaması veya eşleştirme doğru yapılandırılmış
- bir Control UI kimlik doğrulama sorununu tarayıcı sorunuymuş gibi
  tanılamıyorsunuz

Yararlı sayfa: [Control UI](/tr/web/control-ui).

### Katman 5: uçtan uca tarayıcı denetimini doğrulayın

WSL2'den:

```bash
openclaw browser --browser-profile remote open https://example.com
openclaw browser --browser-profile remote tabs
```

Başarılı sonuç:

- sekme Windows Chrome'da açılır
- `browser tabs`, hedefi döndürür
- sonraki eylemler (`snapshot`, `screenshot`, `navigate`) aynı
  profilden çalışır

## Yaygın yanıltıcı hatalar

| İleti                                                                                   | Anlamı                                                                                                                                                                            |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `control-ui-insecure-auth`                                                                      | CDP aktarım sorunu değil, UI kaynağı/güvenli bağlam sorunu                                                                                                                        |
| `token_missing`                                                                      | kimlik doğrulama yapılandırması sorunu                                                                                                                                            |
| `pairing required`                                                                      | cihaz onayı sorunu                                                                                                                                                                |
| `Remote CDP for profile "remote" is not reachable`                                                                      | WSL2, yapılandırılmış `cdpUrl` adresine erişemiyor                                                                                                                      |
| bir portproxy üzerinden boş CDP yanıtı / `other side closed`                             | Windows dinleyici uyuşmazlığı veya kendi kendine döngü; her iki geri döngü ailesini ve `netsh interface portproxy show all` öğesini inceleyin                                                       |
| `Browser attachOnly is enabled and CDP websocket for profile "remote" is not reachable`                                                                      | HTTP uç noktası yanıt verdi ancak DevTools WebSocket açılamadı                                                                                                                    |
| uzak oturumdan sonra eski görünüm alanı / koyu mod / yerel ayar / çevrimdışı geçersiz kılmaları | Gateway'i veya harici tarayıcıyı yeniden başlatmadan oturumu kapatmak ve önbelleğe alınmış Playwright/CDP bağlantısını serbest bırakmak için `openclaw browser --browser-profile remote stop` komutunu çalıştırın |
| CDP erişilebilirliği sırasında zaman aşımı                                               | genellikle hâlâ CDP erişilebilirliği sorunu veya yavaş/erişilemeyen bir uzak uç noktası                                                                                           |
| `Playwright page enumeration timed out after 3000ms`                                                                      | uzak CDP bağlandı ancak kalıcı sekme okuması takıldı                                                                                                                              |
| `No Chrome tabs found for profile="user"`                                                                      | ana bilgisayar yerelinde kullanılabilir sekme olmadığı hâlde yerel Chrome MCP profili seçildi                                                                                    |

## Hızlı tanılama kontrol listesi

1. Windows: `127.0.0.1` veya `[::1]` seçeneklerinden hangisi `/json/version` üzerinde yanıt veriyor ve
   bu dinleyici `chrome.exe` öğesine mi ait?
2. WSL2: `curl http://WINDOWS_HOST_OR_IP:9222/json/version` çalışıyor mu?
3. OpenClaw yapılandırması: `browser.profiles.<name>.cdpUrl` tam olarak bu
   WSL2'den erişilebilen adresi mi kullanıyor?
4. Control UI: bir LAN IP'si yerine `http://127.0.0.1:18789/` adresini mi açıyorsunuz?
5. Ham uzak CDP yerine `existing-session` öğesini WSL2 ve Windows
   arasında kullanmaya mı çalışıyorsunuz?

Önce Windows Chrome uç noktasını yerel olarak, ardından aynı uç noktayı
WSL2'den doğrulayın; OpenClaw yapılandırmasında veya Control UI kimlik
doğrulamasında hata ayıklamaya ancak bundan sonra başlayın.

## İlgili

- [Tarayıcı](/tr/tools/browser)
- [Tarayıcı oturum açma](/tr/tools/browser-login)
- [Tarayıcı Linux sorun giderme](/tr/tools/browser-linux-troubleshooting)
