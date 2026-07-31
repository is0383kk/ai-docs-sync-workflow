---
read_when:
    - OpenClaw ile iletişim kuran harici bir uygulama, betik, pano, CI işi veya IDE uzantısı geliştiriyorsunuz
    - Gateway RPC ile Plugin SDK arasında seçim yapıyorsunuz
    - Gateway ajan çalıştırmaları, oturumları, olayları, onayları, modelleri veya araçlarıyla entegrasyon gerçekleştiriyorsunuz
    - Bir barındırma denetleyicisini harici bir uyandırma zamanlayıcısıyla eşleştiriyorsunuz
sidebarTitle: External apps
summary: Harici uygulamalar, betikler, panolar, CI işleri ve IDE uzantıları için geçerli entegrasyon yolu
title: Harici uygulamalar için Gateway entegrasyonları
x-i18n:
    generated_at: "2026-07-26T22:46:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 276c6f4173197683a60770327e131e6ab2fa4d33f416ba96c170539df7246f83
    source_path: gateway/external-apps.md
    workflow: 16
---

Harici uygulamalar OpenClaw ile Gateway protokolü üzerinden iletişim kurar: WebSocket
aktarımı ve RPC yöntemleri. Bir betik, pano, CI işi, IDE
uzantısı veya başka bir işlem; ajan çalıştırmalarını başlatmak, olayları akış halinde almak, sonuçları
beklemek, çalışmayı iptal etmek ya da Gateway kaynaklarını incelemek istediğinde bunu kullanın.

<Note>
  npm paketleri, cihaz eşleştirme, yeniden bağlanma kurtarması, geçmiş, abonelikler
  ve onaylar için
  [Gateway istemcisi oluşturma](https://docs.openclaw.ai/gateway/clients) ile başlayın. Uygulamanız
  Gateway'i bir alt işlem olarak denetliyorsa ayrıca
  [OpenClaw'ı gömme](https://docs.openclaw.ai/gateway/embedding) bölümünü okuyun. İlk
  paket kullanıma sunma sürecinde, paket içeren ilk OpenClaw sürümü
  yayımlanana kadar npm `E404` döndürebilir.
</Note>

<Note>
  Bu sayfa, OpenClaw işleminin dışındaki kodlar içindir. OpenClaw
  içinde çalışan Plugin kodu bunun yerine belgelenmiş `openclaw/plugin-sdk/*` alt yollarını kullanmalıdır.
</Note>

## Bugün kullanılabilenler

| Yüzey                                                            | Durum         | Kullanım amacı                                                                                           |
| ---------------------------------------------------------------- | ------------- | -------------------------------------------------------------------------------------------------------- |
| [Gateway istemci kılavuzu](https://docs.openclaw.ai/gateway/clients) | Sürüm dizisi | npm paketleri, kimlik doğrulama, yeniden bağlanma, geçmiş, olaylar, onaylar ve sürüm politikası.          |
| [Gömme kılavuzu](https://docs.openclaw.ai/gateway/embedding)     | Sürüm dizisi  | Alt işlem ortamı, hazır olma, yaşam döngüsü, kurtarma, RPC sahipliği ve paketleme.                        |
| [Gateway protokolü](/tr/gateway/protocol)                           | Hazır         | WebSocket aktarımı, bağlantı el sıkışması, kimlik doğrulama kapsamları, protokol sürümleme ve olaylar.    |
| [Gateway RPC referansı](/tr/reference/rpc)                          | Hazır         | Ajanlar, oturumlar, görevler, modeller, araçlar, yapıtlar ve onaylar için güncel Gateway yöntemleri.      |
| [`openclaw agent`](/tr/cli/agent)                                 | Hazır         | CLI'yi kabuk üzerinden çağırmanın yeterli olduğu tek seferlik betik entegrasyonu.                         |
| [`openclaw message`](/tr/cli/message)                               | Hazır         | Betiklerden mesaj veya kanal eylemleri gönderme.                                                         |

## Önerilen yol

1. Bir Gateway çalıştırın veya keşfedin.
2. [Gateway protokolü](/tr/gateway/protocol) üzerinden bağlanın.
3. [Gateway RPC referansında](/tr/reference/rpc) belgelenen RPC yöntemlerini çağırın.
4. Test ettiğiniz OpenClaw sürümünü sabitleyin.
5. OpenClaw'ı yükseltirken RPC referansını yeniden kontrol edin.

Ajan çalıştırmaları için `agent` RPC'siyle başlayın ve terminal
sonucu için bunu `agent.wait` ile eşleştirin. Kalıcı konuşma durumu için `sessions.*` yöntemlerini kullanın.
Kullanıcı arayüzü entegrasyonlarında Gateway olaylarına abone olun ve yalnızca uygulamanızın
anladığı olay ailelerini işleyin.

## İş birliğine dayalı ana makine askıya alma

Çalışan bir işlemi donduran veya anlık görüntüsünü alan barındırma denetleyicileri,
ana makineden bağımsız askıya alma el sıkışmasını kullanabilir:

1. Ana makinenin denetlediği harici girişlerin kabulünü durdurun.
2. Kararlı ve benzersiz bir `requestId` ile `gateway.suspend.prepare` çağrısı yapın.
3. Yanıt `busy` ise işlemi çalışır durumda tutun ve daha sonra yeniden deneyin.
4. Yanıt `ready` ise döndürülen `suspensionId` değerini kaydedin, ardından
   `expiresAtMs` öncesinde işlemi dondurun veya anlık görüntüsünü alın.
5. Çözüldükten sonra veya askıya alma işleminden vazgeçilirse mevcut WebSocket
   ya da Admin HTTP denetim yolu üzerinden bu `suspensionId` ile `gateway.suspend.resume`
   çağrısı yapın.

Hazırlanmış bir Gateway, yeni WebSocket el sıkışmalarını reddeder. Bir WebSocket denetleyicisi,
ana makine işlemi boyunca kimliği doğrulanmış bağlantısını açık tutmalıdır. Bu
garanti edilemiyorsa hazırlamadan önce
[Admin HTTP RPC Pluginini](/tr/plugins/admin-http-rpc) etkinleştirin ve kullanın. Denetim
yolu kaybolursa yeniden bağlanmadan önce iki dakikalık kiralamanın süresinin
dolmasını bekleyin; süre dolduğunda kabul otomatik olarak yeniden açılır.

RPC sözleşmesi şöyledir:

- `gateway.suspend.prepare` — `operator.admin`; parametreler
  `{ "requestId": "stable-host-operation-id" }`
- `gateway.suspend.status` — `operator.read`; parametreler
  `{ "suspensionId": "id-from-prepare" }`
- `gateway.suspend.resume` — `operator.admin`; parametreler
  `{ "suspensionId": "id-from-prepare" }`

Kimliklerin başındaki ve sonundaki boşluklar kaldırılır, en az bir boşluk olmayan karakter
içermeleri gerekir ve uzunlukları 128 karakterle sınırlıdır. Meşgul bir hazırlama sonucunda
`status: "busy"`, `reason`, `retryAfterMs`, `activeCount` ve
`blockers` bulunur. Hazır bir sonuç şu biçimdedir:

```json
{
  "status": "ready",
  "suspensionId": "2c3f...",
  "expiresAtMs": 1770000000000,
  "activeCount": 0,
  "blockers": []
}
```

Durum, `{"status":"running"}` veya `expiresAtMs` içeren hazır bir sonuç döndürür.
Devam ettirme `{"ok":true,"status":"running","resumed":true}` döndürür; başarılı bir devam ettirmeden
sonra yinelenmesi `resumed: false` döndürür.

Çakışan bir istek kimliği veya geçici zamanlayıcı devam ettirme hatası,
`retryAfterMs` içeren yeniden denenebilir `UNAVAILABLE` döndürür. Zamanlayıcı kurtarması sırasında hazırlama, durum
ve devam ettirme işlemlerinin tümü bu hatayı döndürür; Gateway hazır olmayan
ve kapalı başarısızlık durumunda kalır, ana makine de onu dondurmamalı veya anlık görüntüsünü almamalıdır. OpenClaw,
zamanlayıcıyı otomatik olarak yeniden dener ve kabulü yalnızca kurtarma başarılı olduktan sonra yeniden açar.
Eşleşmeyen bir devam ettirme kimliği `INVALID_REQUEST` döndürür. Hazırlama, Gateway'in
dakikada üç denemelik denetim düzlemi yazma bütçesini paylaşır; döndürülen
yeniden deneme gecikmesine uyun. WebSocket istemcileri cihaz ve IP'ye göre gruplandırılır. Admin HTTP
denetleyicileri çözümlenen istemci IP'sine göre gruplandırılır; dolayısıyla tek bir
proxy arkasındaki denetleyiciler bir bütçeyi paylaşabilir.

Hazırlama yalnızca reddetme amaçlıdır: OpenClaw yeni kök/oturum/komut kabulünü kapatır,
otomatik Cron döngülerini duraklatır ve çalışmayı eşzamanlı olarak inceler. Etkin bir şey
varsa `busy` döndürmeden önce zamanlayıcıyı devam ettirir ve kabulü yeniden açar;
bu çalışmayı kesintiye uğratmaz veya boşaltmaz. Hazır kiralama iki dakika sürer.
Aynı `requestId` ile `prepare` çağrısının yinelenmesi kiralamayı yeniler; süre dolduğunda
kabul yeniden açılmadan önce zamanlayıcı devam ettirilir.
Hazır kiralama sırasında zamanı gelen yeniden başlatma yayımı, kiralama devam ettirilene kadar bekler;
devam eden bir yeniden başlatma, hazırlamanın `busy` döndürmesine neden olur.

Hazır durumdayken `/healthz` çalışmaya devam eder ve `/readyz`, `503` döndürür. Yerel veya
kimliği doğrulanmış hazır olma yanıtları `gateway-draining` içerir; kimliği doğrulanmamış
uzak yoklamalar yalnızca `{ "ready": false }` alır. HTTP sağlık yoklaması,
mevcut WebSocket bağlantılarındaki askıya alma yöntemleri ve önceden etkinleştirilmiş
Admin HTTP RPC rotası kullanılabilir durumda kalır. Diğer RPC'ler yeniden denenebilir
`UNAVAILABLE` döndürür. OpenAI uyumlu API'ler, araç/oturum işlemleri, Node izlemeleri ve
yapılandırılmış kancalar dâhil olmak üzere yerleşik HTTP kullanıcı-çalışma rotaları ve sıradan Plugin HTTP rotaları,
`error.code: "gateway_unavailable"` ile `503` döndürür. Yeni
Plugin sahipli WebSocket yükseltmeleri de `503` döndürür; bu, yükseltme
sahipliğini kapsar, kurulu bir Plugin soketi üzerinden daha sonra gerçekleştirilen çalışmayı kapsamaz.

Bu el sıkışma gelen mesajları kalıcılaştırmaz, üçüncü taraf kanal
aktarımlarını durdurmaz veya barındırma platformunu denetlemez. Ana makine, hazırlamadan önce
girişlerini sınırlandırmalı; uyandırma, anlık görüntü/dondurma ve
durdurma sorumluluğunu taşımaya devam etmelidir. `activeCount` toplam izlenen çalışma sayısıdır; `blockers`
ise sıfır olmayan kategori sayılarını ve sınırlandırılmış görev ayrıntılarını içerir. Bu,
genel bir işlem durağanlığı engeli değildir. Bir `background-exec` engelleyicisi yalnızca
toplam bilgidir: komut metni, işlem kimlikleri, çıktı ve oturum veya kapsam tanımlayıcıları
protokol üzerinden hiçbir zaman aktarılmaz. Kanal sağlığı, bakım, önbellek yenileme, kurulmuş
Plugin WebSocket oturumları ve kaydedilmemiş Plugin sahipli arka plan çalışmaları
etkin kalabilir.
Barındırma platformu, işlem ağacının tamamını ve dosya sistemini
tutarlı biçimde dondurmalı veya anlık görüntüsünü almalıdır; bu ilk sözleşme, kaydedilmemiş
çalışmanın boşta olduğunu kanıtlayamaz.

<Tip>
  Ana makine uyandırma zamanlaması için OpenClaw'a bakan kısmı işlem içi
  bir Pluginde tutun ve idempotent tam anlık görüntüleri harici ana makine bağdaştırıcısına yansıtın.
  Barındırma denetleyicisi Plugin SDK'yı içe aktarmamalı veya Cron
  durumunu olay deltalarından yeniden oluşturmamalıdır. Bkz. [Güvenli harici Cron
  yansıtması](/tr/plugins/hooks#safe-external-cron-projection).
</Tip>

## Uygulama kodu ve Plugin kodu

Kod OpenClaw dışında bulunuyorsa Gateway RPC kullanın:

- Ajan çalıştırmalarını başlatan veya gözlemleyen Node betikleri
- Bir Gateway çağıran CI işleri
- panolar ve yönetim panelleri
- IDE uzantıları
- kanal Pluginlerine dönüşmesi gerekmeyen harici köprüler
- sahte veya gerçek Gateway aktarımlarıyla entegrasyon testleri

Kod OpenClaw içinde çalışıyorsa Plugin SDK'yı kullanın:

- sağlayıcı Pluginleri
- kanal Pluginleri
- araç veya yaşam döngüsü kancaları
- ajan çalıştırma düzeneği Pluginleri
- güvenilir çalışma zamanı yardımcıları

Harici uygulamalar `openclaw/plugin-sdk/*` öğesini içe aktarmamalıdır; bu alt yollar
OpenClaw tarafından yüklenen Pluginler içindir.

## İlgili

- [Gateway istemcisi oluşturma](https://docs.openclaw.ai/gateway/clients)
- [OpenClaw'ı gömme](https://docs.openclaw.ai/gateway/embedding)
- [Gateway protokolü](/tr/gateway/protocol)
- [Gateway RPC referansı](/tr/reference/rpc)
- [CLI ajan komutu](/tr/cli/agent)
- [CLI mesaj komutu](/tr/cli/message)
- [Ajan döngüsü](/tr/concepts/agent-loop)
- [Ajan çalışma zamanları](/tr/concepts/agent-runtimes)
- [Oturumlar](/tr/concepts/session)
- [Arka plan görevleri](/tr/automation/tasks)
- [ACP ajanları](/tr/tools/acp-agents)
- [Plugin SDK'ya genel bakış](/tr/plugins/sdk-overview)
