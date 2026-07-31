---
read_when:
    - OpenClaw'u bir masaüstü veya sunucu uygulamasına gömme
    - Gateway'i alt süreç olarak denetleme
    - Logları ayrıştırmadan Gateway'in hazır olma, yeniden başlatma, kapatma veya geçersiz yapılandırma durumlarını yönetme
summary: OpenClaw Gateway'i Electron veya başka bir ana uygulamadan alt süreç olarak denetleyin
title: OpenClaw'u Gömme
x-i18n:
    generated_at: "2026-07-26T22:45:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ca67e03994f21446bfeca58c95c2cb624dde767b9983a89982627145f80dfb90
    source_path: gateway/embedding.md
    workflow: 16
---

Bir gömme ana bilgisayarı, kurulu `openclaw` yürütülebilir dosyasını denetlemeli, denetim düzlemi olarak
Gateway WebSocket protokolünü kullanmalı ve alt süreci değiştirilebilir bir
çalışma zamanı olarak ele almalıdır. Bu yaklaşım; süreç sahipliğini, hazır olma durumunu, hata kurtarmayı
ve yükseltmeleri OpenClaw'ın özel durum düzenine bağlı kalmadan açık hâle getirir.

İstemci kimlik doğrulaması ve yeniden bağlanma durumu için
[Gateway istemcisi oluşturma](https://docs.openclaw.ai/gateway/clients) bölümünü okuyun.

## Alt süreci bir gömme ön ayarıyla başlatma

Gerçek bir `node_modules` kurulumu kullanın ve paket yürütülebilir dosyasını başlatın. Keşif,
yeniden başlatma ve kanal yaşam döngüsünün sahibi olan bir ana bilgisayar için kullanışlı bir
temel yapı şöyledir:

```ts
import { spawn } from "node:child_process";
import { dirname, resolve } from "node:path";
import { fileURLToPath } from "node:url";

// Ana bilgisayar uygulaması tarafından yönetilen gerçek bir Node çalışma zamanının mutlak yolunu sağlayın.
declare const hostNodeExecutable: string;

const packageEntry = fileURLToPath(import.meta.resolve("openclaw"));
const openclawEntry = resolve(dirname(packageEntry), "..", "openclaw.mjs");
const gateway = spawn(hostNodeExecutable, [openclawEntry, "gateway", "--allow-unconfigured"], {
  env: {
    ...process.env,
    OPENCLAW_DISABLE_BONJOUR: "1",
    OPENCLAW_EXEC_SHELL_SNAPSHOT: "0",
    OPENCLAW_NO_RESPAWN: "1",
    OPENCLAW_SKIP_CHANNELS: "1",
  },
  stdio: ["ignore", "inherit", "inherit"],
});
```

OpenClaw'ı gösterildiği gibi kurulu paket üzerinden çözümleyin; proje yerelindeki bir
`openclaw` ikili dosyasının ana bilgisayar sürecinin `PATH` değişkeninde bulunduğunu varsaymayın. Örnek,
alt sürecin dolu stdout veya stderr kanallarında engellenememesi için çıktıyı devralır. Ana
bilgisayar bunun yerine bu akışları yakalıyorsa, tüketicileri alt süreç başlatıldıktan hemen sonra bağlayın.

| Ayar                             | Gömme etkisi                                                                                                                                                                               |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `OPENCLAW_DISABLE_BONJOUR=1`     | Ana bilgisayar keşfin sahibi olduğunda Gateway'in yönettiği LAN çok noktaya yayın duyurularını devre dışı bırakır.                                                                          |
| `OPENCLAW_NO_RESPAWN=1`          | Yönetilmeyen bir gömme alt sürecinde, OpenClaw'ın güncelleme yeniden başlatmasını ayrılmış bir alt sürece devretmesini önler. Rutin yeniden başlatmalar süreç içinde kalır; böylece ana bilgisayar izlenen PID'nin sahipliğini korur. |
| `OPENCLAW_EXEC_SHELL_SNAPSHOT=0` | Ana bilgisayar yürütme komutları için oturum açma kabuğu anlık görüntüsü yakalamayı devre dışı bırakır.                                                                                     |
| `OPENCLAW_SKIP_CHANNELS=1`       | Kanal başlatma ve yeniden yüklemeyi atlar. Yalnızca gömme uygulaması bir denetim düzlemi veya yalnızca WebChat için kullanılan bir Gateway istiyorsa ayarlayın.                             |

`--allow-unconfigured` yalnızca `gateway.mode=local` başlangıç korumasını atlar.
Yapılandırma yazmaz veya geçersiz bir dosyayı onarmaz. Gömme
uygulaması ilk katılım, yapılandırma CLI'si
veya Gateway RPC üzerinden normal bir yerel yapılandırma sağlıyorsa bunu kullanmayın.

### Electron kabuk anlık görüntüsü uyarısı

Kabuk anlık görüntüsü yakalama, bir oturum açma kabuğundan `process.execPath -e <script>` çalıştırır. Normal
bir Node sürecinde `process.execPath`, Node yürütülebilir dosyasıdır. Electron altında
ise Electron ikili dosyasıdır; bu dosya çağrıyı bir uygulama
başlatma işlemi olarak yorumlayıp "Unable to find Electron app" açılır penceresini gösterebilir.
`OPENCLAW_EXEC_SHELL_SNAPSHOT=0` değerini yalnızca
işleyici sürecinde değil, Gateway alt sürecinin ortamında ayarlayın. Aynı nedenle `hostNodeExecutable`,
Electron'ın `process.execPath` değeri yerine gerçek bir Node çalışma zamanını göstermelidir.

## Geçersiz yapılandırmayı çıkış koduna göre işleme

Gateway başlangıcı, geçersiz bir yapılandırma dâhil olmak üzere yapılandırma sınıfındaki başlangıç
hataları için `78` (`EX_CONFIG`) çıkış kodunu kullanır. İnsan tarafından
okunabilen stderr çıktısını ayrıştırmak yerine çıkış koduna göre dallanın:

1. Gateway alt süreciyle aynı yapılandırma ve
   durum ortamında `openclaw doctor --fix --yes --non-interactive` çalıştırın.
2. Doctor başarıyla çıktıktan sonra Gateway başlangıcını bir kez yeniden deneyin.
3. Alt süreç yeniden `78` koduyla çıkarsa onarım döngüsünü durdurun ve yapılandırma
   hatasını kullanıcıya gösterin.

stderr çıktısını tanılama amacıyla saklayın ancak yaşam döngüsü kararlarını çıktının ifadelerine göre vermeyin.

Başarılı bir başlangıçtan sonra, geçersiz bir canlı yapılandırma düzenlemesi daha az yıkıcıdır.
Yapılandırma izleyicisi yeniden yüklemenin atlandığını günlüğe kaydeder ve en son kabul edilen
bellek içi yapılandırmayla hizmet vermeyi sürdürür. Dosyayı onarın, ardından izleyicinin bir sonraki geçerli
anlık görüntüyü kabul etmesini bekleyin.

## Protokolün hazır olmasını bekleme

Günlük alt dizesi yerine WebSocket sinyallerini kullanın:

1. Gateway WebSocket'ini açın.
2. `connect.challenge` olayını bekleyin. Bu olay, dinleyicinin
   WebSocket'i kabul ettiğini ve sınama el sıkışmasının başlayabileceğini kanıtlar.
3. Sınamaya bağlı cihaz imzasıyla `connect` gönderin.
4. Kimliği doğrulanmış RPC için `hello-ok` değerini uygulamanın hazır olması olarak kabul edin.

Sınama, tam başlatmadan bilinçli olarak daha erken gerçekleşir. Başlangıç
yardımcı süreçleri hâlâ beklemedeyse `connect`, `details.reason: "startup-sidecars"` ve sınırlı bir
`retryAfterMs` içeren, yeniden denenebilir bir `UNAVAILABLE` hatası döndürür; ardından
`1013` kodu ve `gateway starting` nedeni ile bağlantıyı kapatır.
`@openclaw/gateway-protocol/startup-unavailable` içindeki
`resolveGatewayStartupRetryAfterMs` değerini veya referans istemcinin yerleşik
ilkesini kullanın, ardından yeniden bağlanın.

## Yeniden başlatma ve kapatmayı yorumlama

Düzenli bir kapatmadan önce Gateway, `reason` ve `restartExpectedMs` içeren bir `shutdown` olayı yayınlar.
Null olmayan bir `restartExpectedMs`, süreç içi veya
denetimli bir yeniden başlatma beklendiği; `null` ise nihai bir kapatma olduğu anlamına gelir.

Sonraki WebSocket kapatma kodu her iki durumda da `1012` olur. Normal istemci
kapatma nedeni de her iki durumda `service restart` olduğundan ne kapatma kodu ne de
neden, yeniden başlatmayı kapatmadan ayırır. Geldiğinde önceki `shutdown`
yükünü koruyun ve bunu ana bilgisayarın kendi durdurma niyetiyle ve
alt sürecin çıkış durumuyla birleştirin. Bağlantı olay olmadan kesilirse normal
sınırlı yeniden bağlanma ve alt süreç denetleme ilkesini kullanın.

## Durum dosyaları yerine RPC kullanma

Gateway'i OpenClaw durumunun tek sahibi olarak tutun. Yaygın gömme işlemlerinin
zaten RPC yöntemleri vardır:

| Görev                         | RPC yöntemleri                                        |
| ----------------------------- | ----------------------------------------------------- |
| Oturum kataloğu ve yaşam döngüsü | `sessions.list`, `sessions.patch`, `sessions.delete` |
| Transkript görüntüleme        | `chat.history`                                    |
| Maliyet ve kullanım raporları | `usage.cost`, `sessions.usage`                |
| Model kimlik bilgisi durumu   | `models.authStatus`                                    |
| Yapılandırma                  | `config.get`, `config.patch`                |

`config.get`, anlık görüntüyü döndürmeden önce hassas değerleri ve SecretRef tanımlayıcılarını
gizler. Yazma yöntemleri de gizlenmiş yapılandırma döndürür. İstemci,
gizleme belirtecini opak olarak ele almalı ve belgelenmiş yapılandırma yazma sözleşmesini kullanmalıdır;
Gateway'in düz metin gizli değerler döndürmesini asla beklememelidir.

Uygulama özelliklerini gerçekleştirmek için `~/.openclaw` altındaki dosyaları, SQLite tablolarını,
transkript dosyalarını veya önbellek dizinlerini okumayın ya da değiştirmeyin. Bu düzenler özel çalışma zamanı
uygulama ayrıntılarıdır ve protokol uyumluluğu olmadan taşınabilir veya değiştirilebilir.

## Kurun; düzleştirmeyin

Kök `openclaw` paketi, tek dosyalı bir satıcıya dâhil etme hedefi değildir. `dist/extensions` altındaki paketlenmiş çalışma zamanı
dosyaları `openclaw/plugin-sdk/*` gibi çıplak öz içe aktarımları korurken,
npm paketi uzantı başına `node_modules` ağaçlarını bilinçli olarak hariç tutar.

Node'un paket dışa aktarımlarını ve kök bağımlılık ağacını çözümleyebilmesi için OpenClaw'ı
npm, pnpm veya başka bir normal Node paket kurulumu üzerinden kurun. Kurulu
`openclaw` yürütülebilir dosyasını başlatın. Yalnızca `dist` dosyasını kopyalamayın, paketi bir uygulama
paketi içinde düzleştirmeyin veya seçili uzantı dosyalarını satıcıya dâhil etmeyin.

## İlgili

- [Gateway istemcisi oluşturma](https://docs.openclaw.ai/gateway/clients)
- [Gateway protokolü](https://docs.openclaw.ai/gateway/protocol)
- [Gateway CLI](https://docs.openclaw.ai/cli/gateway)
- [Harici uygulamalar için Gateway entegrasyonları](https://docs.openclaw.ai/gateway/external-apps)
