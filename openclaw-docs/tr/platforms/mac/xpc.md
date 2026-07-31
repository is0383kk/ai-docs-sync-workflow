---
read_when:
    - IPC sözleşmelerini veya menü çubuğu uygulamasının IPC'sini düzenleme
summary: OpenClaw uygulaması, Gateway node aktarımı ve PeekabooBridge için macOS IPC mimarisi
title: macOS IPC
x-i18n:
    generated_at: "2026-07-27T00:04:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39e11af2bb9348d1c1f6e4fe6be95e825d23d5c1aa66e32dae713a89afb12b4f
    source_path: platforms/mac/xpc.md
    workflow: 16
---

# OpenClaw macOS IPC mimarisi

Yerel bir Unix soketi, exec onayları ve `system.run` için Node ana makine hizmetini macOS uygulamasına bağlar. Keşif/bağlantı denetimleri için bir `openclaw-mac` hata ayıklama CLI'ı (`apps/macos/Sources/OpenClawMacCLI`) bulunur; ajan eylemleri yine Gateway WebSocket ve `node.invoke` üzerinden akar. Node destekli `computer.act` yolu, gömülü Peekaboo otomasyonunu işlem içinde çalıştırır; bağımsız Peekaboo istemcileri PeekabooBridge kullanır.

## Hedefler

- TCC ile ilişkili tüm işleri (bildirimler, ekran kaydı, mikrofon, konuşma, AppleScript) sahiplenen tek GUI uygulaması örneği.
- Otomasyon için küçük bir yüzey: Gateway + Node komutları, işlem içi `computer.act` ve bağımsız kullanıcı arayüzü otomasyon istemcileri için PeekabooBridge.
- Öngörülebilir izinler: TCC izinlerinin kalıcı olması için her zaman launchd tarafından başlatılan aynı imzalı paket kimliği.

## Nasıl çalışır?

### Gateway + Node aktarımı

- Uygulama Gateway'i (yerel modda) çalıştırır ve ona bir Node olarak bağlanır.
- Ajan eylemleri `node.invoke` üzerinden gerçekleştirilir (ör. `system.run`, `system.notify`, `canvas.*`).
- Node komutları arasında `canvas.*`, `camera.snap`, `camera.clip`, `screen.snapshot`, `screen.record`, `computer.act`, `system.run` ve `system.notify` bulunur.
- Node, ajanların ekran, kamera, mikrofon, konuşma, otomasyon veya erişilebilirlik erişiminin kullanılabilir olup olmadığını görebilmesi için bir `permissions` eşlemesi bildirir.

### Node hizmeti + uygulama IPC'si

- Başsız bir Node ana makine hizmeti Gateway WebSocket'e bağlanır.
- `system.run` istekleri, yerel bir Unix soketi (`ExecApprovalsSocket.swift`) üzerinden macOS uygulamasına iletilir.
- Uygulama exec işlemini kullanıcı arayüzü bağlamında gerçekleştirir, gerekirse istem gösterir ve çıktıyı döndürür.

Diyagram (SCI):

```text
Ajan -> Gateway -> Node Hizmeti (WS)
                      |  IPC (UDS + belirteç + HMAC + TTL)
                      v
                  Mac Uygulaması (UI + TCC + system.run)
```

### PeekabooBridge (kullanıcı arayüzü otomasyonu)

- Yerleşik ajan `computer` aracı bu soketi **kullanmaz**. Eşleştirilmiş bir macOS Node'u, gömülü Peekaboo hizmetleriyle uygulama işlemi içinde `computer.act` işlemini gerçekleştirir.
- Kullanıcı arayüzü otomasyonu ayrı bir UNIX soketi (`~/Library/Application Support/OpenClaw/<socket>`) ve PeekabooBridge JSON protokolünü kullanır.
- Ana makine tercih sırası (istemci tarafında): Peekaboo.app -> Claude.app -> OpenClaw.app -> yerel yürütme.
- Güvenlik: köprü ana makineleri izin verilenler listesindeki bir TeamID'yi gerektirir (paketlenmiş `PeekabooBridgeHostCoordinator`, sabit bir ekibi ve uygulamanın kendi imzalama ekibini izin verilenler listesine ekler); yalnızca DEBUG için aynı UID'ye yönelik bir kaçış yolu `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` ile korunur (Peekaboo kuralı).
- Ayrıntılar için bkz. [PeekabooBridge kullanımı](/tr/platforms/mac/peekaboo).

## İşletim akışları

- Yeniden başlatma/derleme: `scripts/restart-mac.sh` mevcut örnekleri sonlandırır, Swift aracılığıyla yeniden derler, yeniden paketler ve yeniden başlatır. Kullanılabilir bir imzalama kimliğini otomatik olarak algılar ve kimlik bulunamazsa `--no-sign` seçeneğine geri döner; imzalamayı zorunlu kılmak için `--sign` iletin (anahtar yoksa başarısız olur) veya imzasız yolu zorlamak için `--no-sign` iletin. Ortamda ayarlanan `SIGN_IDENTITY`, imzalı yolda kaldırılır; böylece `scripts/codesign-mac-app.sh` öğesinin kendi kimlik otomatik algılaması sertifikayı seçer.
- Tek örnek: uygulama, yinelenen bir paket kimliği için `NSWorkspace.runningApplications` öğesini denetler ve birden fazla örnek bulunursa çıkar (`MenuBar.swift` içindeki `isDuplicateInstance()`).

## Güçlendirme notları

- Tüm ayrıcalıklı yüzeyler için TeamID eşleşmesini zorunlu kılmayı tercih edin.
- PeekabooBridge: `PEEKABOO_ALLOW_UNSIGNED_SOCKET_CLIENTS=1` (yalnızca DEBUG), yerel geliştirme için aynı UID'ye sahip çağıranlara izin verebilir.
- Tüm iletişim yalnızca yerel olarak kalır; hiçbir ağ soketi açığa çıkarılmaz.
- TCC istemleri yalnızca GUI uygulama paketinden kaynaklanır; imzalı paket kimliğini yeniden derlemeler arasında kararlı tutun.
- Exec onayları soketi güçlendirmesi: dosya modu `0600`, paylaşılan belirteç, eş UID denetimi (`getpeereid`), HMAC-SHA256 sınama/yanıtı ve istekler için kısa bir TTL.

## İlgili

- [macOS uygulaması](/tr/platforms/macos)
- [macOS IPC akışı (Exec onayları)](/tr/tools/exec-approvals-advanced#macos-ipc-flow)
