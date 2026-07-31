---
read_when:
    - Bir istemci `rate limit exceeded for <method>`, `AUTH_RATE_LIMITED` veya kilitlenme hataları görüyor
    - '`gateway.auth.rateLimit` ayarını yapmak istiyorsunuz'
    - Açıkta bulunan bir Gateway için kaba kuvvet saldırısı korumasını değerlendiriyorsunuz
    - Hangi Gateway yüzeylerinin hangi limitlerle kısıtlandığını bilmeniz gerekir
summary: 'Her Gateway hız sınırı için referans: kimlik doğrulama öncesi kilitlemeler, tarayıcı ve Webhook kısıtlamaları, kontrol düzlemi yazma güvenlik sınırı, ACP oturum üst sınırları ve yeniden başlatma bekleme süresi'
title: Hız sınırlama
x-i18n:
    generated_at: "2026-07-26T23:22:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7aa37b65347610bedfb1db8f661e7ba75ef3cdfed0ba73c4ce53d80acace1e48
    source_path: gateway/security/rate-limiting.md
    workflow: 16
---

Gateway birkaç bağımsız hız sınırı uygular. Bunlar farklı
sınırları korur, farklı kimlikleri anahtar olarak kullanır ve farklı hata biçimleriyle
başarısız olur. Bu sayfa, bunların tümü için başvuru kaynağıdır.

Bir bakışta:

| Yüzey                               | Sınır (varsayılan)                 | Anahtar                          | Yapılandırılabilir        |
| ----------------------------------- | ---------------------------------- | -------------------------------- | ------------------------- |
| Başarısız kimlik doğrulama (token/parola/cihaz) | 60 sn'de 10 başarısızlık, 5 dk kilitleme | IP + kimlik bilgisi kapsamı      | `gateway.auth.rateLimit` |
| Tarayıcı kaynaklı WS kimlik doğrulama başarısızlıkları | aynı, geri döngü **muaf değil** | IP veya geri döngüden sayfa kaynağı | `gateway.auth.rateLimit` |
| Webhook (`/hooks`) kimlik doğrulama başarısızlıkları | 60 sn'de 20 başarısızlık, 60 sn kilitleme | IP                               | hayır                     |
| Denetim düzlemi yazma RPC'leri      | yöntem başına 60 sn'de 30 istek    | yöntem + cihaz + IP              | hayır                     |
| ACP oturumu oluşturma               | 10 sn'de 120 oturum                | çevirmen örneği                  | dahili                    |
| Gateway yeniden başlatma döngüleri  | yeniden başlatmalar arasında 30 sn bekleme süresi | süreç                            | hayır                     |

## Kimlik doğrulama denemeleri (kimlik doğrulama öncesi)

Başarısız kimlik doğrulama denemeleri, herhangi bir istek işlenmeden önce
istemci IP'si başına kısıtlanır. Bu, dışarıya açık Gateway'ler için kaba kuvvet korumasıdır.

- Yalnızca _yanlış_ kimlik bilgileri sayılır. Eksik kimlik bilgileri (hiç
  token göndermemiş bir istemci) ve başarılı kimlik doğrulamalar bütçeyi tüketmez;
  başarılı bir kimlik doğrulama, söz konusu IP'nin sayacını sıfırlar.
- Varsayılanlar: 60 saniyede 10 başarısızlık, ardından söz konusu IP için 5 dakikalık kilitleme.
- Geri döngü (`127.0.0.1` / `::1`) varsayılan olarak muaftır; böylece yerel CLI oturumları
  kilitlenemez.
- Sayaçlar kimlik bilgisi sınıfına göre kapsamlandırılır; dolayısıyla bir yüzeye yönelik
  istek seli diğerini etkilemez. Kapsamlar arasında paylaşılan gateway
  token'ı/parolası, cihaz token'ları, Node eşleştirme, eşleştirilmiş Node'un yeniden onaylanması,
  cihaz önyükleme token'ları ve watchOS sınaması oluşturma bulunur.

Kilitliyken bağlantı denemeleri şu hatayla başarısız olur:

```json
{
  "code": "INVALID_REQUEST",
  "message": "yetkisiz: çok fazla başarısız kimlik doğrulama denemesi (daha sonra yeniden deneyin)",
  "retryable": true,
  "retryAfterMs": 297000,
  "details": {
    "code": "AUTH_RATE_LIMITED",
    "authReason": "rate_limited",
    "recommendedNextStep": "wait_then_retry"
  }
}
```

Kilitleme sırasında diğer IP'lerden (geri döngü dâhil) yapılan denemeler etkilenmez.

Bunu `openclaw.json` içindeki `gateway.auth.rateLimit` altında ayarlayın:

```json
{
  "gateway": {
    "auth": {
      "rateLimit": {
        "maxAttempts": 10,
        "windowMs": 60000,
        "lockoutMs": 300000,
        "exemptLoopback": true
      }
    }
  }
}
```

Gateway günlüğünde yinelenen `AUTH_RATE_LIMITED` girdileri, birinin
kimlik bilgilerini tahmin etmeye çalıştığı anlamına gelir; [dışa açılma çalışma kılavuzuna](/tr/gateway/security/exposure-runbook) bakın.

### Tarayıcı kaynaklı bağlantılar

Tarayıcı `Origin` üstbilgisi taşıyan WebSocket bağlantıları aynı
sınırları kullanır, ancak geri döngü muafiyeti **her zaman kapalıdır** — yerel
tarayıcıdaki kötü amaçlı bir sayfa hâlâ güvenilmeyen bir istemcidir; dolayısıyla localhost bu
yolda ayrıcalık kazanmaz. Böyle bir bağlantı bir geri döngü adresinden _geldiğinde_,
başarısızlıkları paylaşılan geri döngü IP'si yerine normalleştirilmiş sayfa kaynağına göre
(örneğin `browser-origin:https://evil.example`) anahtarlanır;
böylece her kaynak kendi kovasına sahip olur. Geri döngü dışı adreslerden geldiğinde anahtar
istemci IP'si olarak kalır. Bu yapılandırılamaz.

### Webhook'lar

HTTP `/hooks` girişi kendi başarısızlık sınırlayıcısına sahiptir: istemci IP'si başına
60 saniyede 20 başarısız kimlik doğrulama, ardından 60 saniyelik kilitleme.
Geri döngü muaf değildir. Başarılı hook kimlik doğrulaması sayacı sıfırlar. Kısıtlanan
istekler, `Retry-After` üstbilgisiyle (saniye cinsinden) düz HTTP
`429 Too Many Requests` yanıtı alır. Sınırlar sabittir; geçerli bir entegrasyon bu sınıra takılıyorsa
daha agresif yeniden denemek yerine kimlik bilgilerini düzeltin.

## Denetim düzlemi yazmaları (kimlik doğrulama sonrası güvenlik ağı)

Yazma tarafındaki yönetici RPC'leri (`config.apply`, `config.patch`, `plugins.install`,
`plugins.setEnabled`, `plugins.uninstall`, `update.run`, `worktrees.*`,
`gateway.restart.request`, ...) yetkilendirmeden **sonra** ayrıca hız sınırına tabidir:
`deviceId+clientIp` başına, yöntem başına, 60 saniyede 30 istek.

Bu bir güvenlik sınırı değildir — çağıranlar zaten `operator.admin` sahibidir — pahalı
işlemleri yoğun biçimde çağıran kontrolden çıkmış istemci veya agent döngülerini sınırlayan
bir güvenlik ağıdır. Etkileşimli kullanım bu sınıra asla ulaşmaz; her yöntemin kendi kovası vardır,
bu nedenle bir Plugin'i açıp kapatmak yapılandırma yazma bütçesini tüketmez.

Sınır aşıldığında istek, yeniden denenebilir bir hatayla başarısız olur:

```json
{
  "code": "UNAVAILABLE",
  "message": "config.patch için hız sınırı aşıldı; 35 sn sonra yeniden deneyin",
  "retryable": true,
  "retryAfterMs": 34539,
  "details": { "method": "config.patch", "limit": "60 sn'de 30" }
}
```

İstemciler `retryAfterMs` değerine uymalıdır. Sınır sabittir (yapılandırılamaz);
kovaların süresi kendiliğinden dolar ve Gateway bakımı tarafından temizlenir.

## ACP oturumu oluşturma

ACP çevirmeni, oturum oluşturmayı çevirmen örneği başına her 10 saniyelik
pencerede 120 yeni oturumla sınırlar. Bu sınırın aşılması, iletisinde bekleme süresini
taşıyan bir hatayla isteğin başarısız olmasına neden olur (bu yolda yapılandırılmış bir
`retryAfterMs` alanı yoktur):

```
<method> için ACP oturumu oluşturma hız sınırı aşıldı; <n> sn sonra yeniden deneyin.
```

Bu, döngü içinde oturum oluşturan kontrolden çıkmış istemcileri sınırlar; normal IDE ve
agent kullanımı bunun çok altında kalır.

## Yeniden başlatma bekleme süresi

Gateway yeniden başlatma istekleri birleştirilir, ardından yeniden başlatma
döngüleri arasında 30 saniyelik bekleme süresi uygulanır. Bekleme süresi sırasında istenen
bir yeniden başlatma reddedilmek yerine sürenin dolmasından sonraya zamanlanır. Bu, yukarıdaki
denetim düzlemi sınırlayıcısından ayrıdır: `gateway.restart.request` bir denetim düzlemi bütçe yuvası
tüketir _ve_ bunun sonucunda gerçekleşen yeniden başlatma bekleme süresine uyar.

## İşletim notları

- Tüm sınırlayıcılar bellektedir ve süreç başınadır; birden fazla Gateway
  durumu paylaşmaz. Gateway sürecinin değiştirilmesi, Gateway'in sahip olduğu
  sayaçları (kimlik doğrulama kilitlemeleri, Webhook kısıtlaması, denetim düzlemi kovaları) temizler.
  Yeniden başlatma bekleme süresi, süreç içi yeniden başlatma döngülerinde kasıtlı olarak korunur —
  çünkü sınırladığı şey budur — ve yalnızca süreçle birlikte sıfırlanır. ACP oturum sınırı
  kendi çevirmen örneğine aittir ve Gateway yeniden başlatıldığında değil, bu örnek
  yeniden oluşturulduğunda sıfırlanır.
- Kova eşlemeleri sınırlıdır (katı girdi üst sınırları ve düzenli temizleme);
  bu nedenle benzersiz anahtar taşmaları belleği sınırsız büyütemez.
- Bir istemci ters proxy arkasındayken geçerli IP, çözümlenen
  istemci IP'sidir; proxy üstbilgilerinin bunu etkileyebilmeden önce nasıl
  doğrulandığı için [güvenilen proxy kimlik doğrulamasına](/tr/gateway/trusted-proxy-auth) bakın.
- Yeniden deneme sinyali yüzeye göre değişir: Gateway RPC sınırlayıcıları
  `retryable: true` ile birlikte `retryAfterMs` döndürür, Webhook girişi
  `Retry-After` üstbilgisiyle HTTP 429 kullanır ve ACP bekleme süresini hata iletisine gömer.
  Her durumda hemen yeniden denemek yerine belirtilen süre boyunca geri çekilin.
