---
permalink: /security/formal-verification/
read_when:
    - Resmî güvenlik modeli garantilerini veya sınırlarını inceleme
    - TLA+/TLC güvenlik modeli kontrollerini yeniden üretme veya güncelleme
summary: OpenClaw'un en yüksek riskli yolları için makine tarafından doğrulanan güvenlik modelleri.
title: Biçimsel doğrulama (güvenlik modelleri)
x-i18n:
    generated_at: "2026-07-26T23:01:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 185ee5c1cff7325f10827330c0c7e55ddc3ca40caf6088d4c930ae5e090d6b27
    source_path: security/formal-verification.md
    workflow: 16
---

OpenClaw'ın biçimsel güvenlik modelleri (günümüzde TLA+/TLC), açıkça belirtilen varsayımlar altında belirli en yüksek riskli yolların — yetkilendirme, oturum yalıtımı, araç geçitleme ve yanlış yapılandırma güvenliği — amaçlanan politikayı uyguladığına ilişkin makine tarafından denetlenmiş bir argüman sunar.

> Not: Bazı eski bağlantılar önceki proje adına atıfta bulunabilir.

## Bu nedir?

Yürütülebilir, saldırgan odaklı bir güvenlik regresyon paketi:

- Her iddia, sonlu bir durum uzayı üzerinde çalıştırılabilir bir model denetimine sahiptir.
- Birçok iddianın, gerçekçi bir hata sınıfı için karşı örnek izi üreten eşleştirilmiş bir negatif modeli vardır.

Bu, OpenClaw'ın her bakımdan güvenli olduğunun **kanıtı değildir** ve TypeScript uygulamasının tamamını doğrulamaz.

## Modellerin bulunduğu yer

Modeller ayrı bir depoda tutulur: [vignesh07/openclaw-formal-models](https://github.com/vignesh07/openclaw-formal-models).

<Note>
Bu yazının hazırlandığı sırada söz konusu depoya erişilemiyor (GitHub, "Repository not found" yanıtını döndürüyor). Sizin için de hâlâ erişilemiyorsa modellerin kaldırıldığını varsaymadan önce güncel konumu OpenClaw bakım sorumlusu kanallarında sorun.
</Note>

## Uyarılar

- Bunlar TypeScript uygulamasının tamamı değil, modellerdir — model ile kod arasında sapma olması mümkündür.
- Sonuçlar, TLC'nin araştırdığı durum uzayıyla sınırlıdır. Yeşil sonuç, modellenen varsayımların ve sınırların ötesinde güvenlik anlamına gelmez.
- Bazı iddialar açık ortam varsayımlarına dayanır (örneğin doğru dağıtım ve doğru yapılandırma girdileri).

## Sonuçları yeniden üretme

Model deposunu klonlayın ve TLC'yi çalıştırın:

```bash
git clone https://github.com/vignesh07/openclaw-formal-models
cd openclaw-formal-models

# Java 11+ gereklidir (TLC, JVM üzerinde çalışır).
# Depo, sabitlenmiş bir tla2tools.jar içerir ve bin/tlc ile Make hedefleri sağlar.

make <target>
```

Henüz bu depoya geri bağlanan bir CI entegrasyonu yoktur; gelecekteki bir yineleme, herkese açık yapıtlarla (karşı örnek izleri, çalıştırma günlükleri) CI tarafından çalıştırılan modeller veya küçük ve sınırlı denetimler için barındırılan bir "bu modeli çalıştır" iş akışı ekleyebilir.

## İddialar ve hedefler

### Gateway erişimi ve açık Gateway yanlış yapılandırması

**İddia:** Modelin varsayımlarına göre, kimlik doğrulama olmadan geri döngü dışına bağlanmak uzaktan ele geçirmeyi mümkün kılabilir ve erişimi genişletir; bir belirteç/parola, kimliği doğrulanmamış saldırganları engeller.

| Sonuç         | Hedefler                                                          |
| -------------- | ---------------------------------------------------------------- |
| Yeşil          | `make gateway-exposure-v2`, `make gateway-exposure-v2-protected` |
| Kırmızı (beklenen) | `make gateway-exposure-v2-negative`                              |

Ayrıca model deposundaki `docs/gateway-exposure-matrix.md` bölümüne bakın.

### Node yürütme işlem hattı (en yüksek riskli yetenek)

**İddia:** Modelde `exec host=node`; (a) bir Node komutu izin listesi ile bildirilmiş komutları ve (b) yapılandırıldığında canlı onayı gerektirir; yeniden oynatmayı önlemek için onaylar belirteçleştirilir.

| Sonuç         | Hedefler                                                         |
| -------------- | --------------------------------------------------------------- |
| Yeşil          | `make nodes-pipeline`, `make approvals-token`                   |
| Kırmızı (beklenen) | `make nodes-pipeline-negative`, `make approvals-token-negative` |

### Eşleştirme deposu (DM geçitleme)

**İddia:** Eşleştirme istekleri TTL'ye ve bekleyen istek sınırlarına uyar.

| Sonuç         | Hedefler                                              |
| -------------- | ---------------------------------------------------- |
| Yeşil          | `make pairing`, `make pairing-cap`                   |
| Kırmızı (beklenen) | `make pairing-negative`, `make pairing-cap-negative` |

### Giriş geçitleme (bahsetmeler ve denetim komutu atlaması)

**İddia:** Bahsetme gerektiren grup bağlamlarında, yetkisiz bir denetim komutu bahsetme geçitlemesini atlayamaz.

| Sonuç         | Hedefler                        |
| -------------- | ------------------------------ |
| Yeşil          | `make ingress-gating`          |
| Kırmızı (beklenen) | `make ingress-gating-negative` |

### Yönlendirme ve oturum anahtarı yalıtımı

**İddia:** Farklı eşlerden gelen DM'ler, açıkça bağlanmadıkları veya bu şekilde yapılandırılmadıkları sürece aynı oturumda birleşmez.

| Sonuç         | Hedefler                           |
| -------------- | --------------------------------- |
| Yeşil          | `make routing-isolation`          |
| Kırmızı (beklenen) | `make routing-isolation-negative` |

## v1++ modelleri: eşzamanlılık, yeniden denemeler ve iz doğruluğu

Atomik olmayan güncellemeler, yeniden denemeler ve ileti dağıtımı gibi gerçek dünyadaki hata modlarına ilişkin doğruluğu artıran devam modelleri.

### Eşleştirme deposunda eşzamanlılık ve eşgüçlülük

**İddia:** Eşleştirme deposu, yürütmeler iç içe geçtiğinde bile `MaxPending` ve eşgüçlülüğü uygular — denetle-sonra-yaz işlemi atomik/kilitli olmalı ve yenileme yinelenen kayıtlar oluşturmamalıdır. Somut olarak: eşzamanlı istekler bir kanal için `MaxPending` sınırını aşamaz ve aynı `(channel, sender)` için tekrarlanan istekler/yenilemeler, yinelenen etkin bekleyen satırlar oluşturmaz.

| Sonuç         | Hedefler                                                                                                                                                                     |
| -------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Yeşil          | `make pairing-race` (atomik/kilitli sınır denetimi), `make pairing-idempotency`, `make pairing-refresh`, `make pairing-refresh-race`                                              |
| Kırmızı (beklenen) | `make pairing-race-negative` (atomik olmayan başlatma/işleme sınırı yarışı), `make pairing-idempotency-negative`, `make pairing-refresh-negative`, `make pairing-refresh-race-negative` |

### Giriş izi korelasyonu ve eşgüçlülük

**İddia:** İçeri alma işlemi, dağıtım boyunca iz korelasyonunu korur ve sağlayıcının yeniden denemeleri altında eşgüçlüdür. Bir harici olay birden çok dahili iletiye dönüştüğünde her parça aynı iz/olay kimliğini korur; yeniden denemeler çift işlemeye yol açmaz; sağlayıcı olay kimlikleri eksikse farklı olayların atılmasını önlemek için tekilleştirme güvenli bir anahtara (örneğin iz kimliğine) geri döner.

| Sonuç         | Hedefler                                                                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Yeşil          | `make ingress-trace`, `make ingress-trace2`, `make ingress-idempotency`, `make ingress-dedupe-fallback`                                     |
| Kırmızı (beklenen) | `make ingress-trace-negative`, `make ingress-trace2-negative`, `make ingress-idempotency-negative`, `make ingress-dedupe-fallback-negative` |

### Yönlendirmede dmScope önceliği ve identityLinks

**İddia:** `dmScope` önceliği ve kimlik bağlantıları belirlenimci biçimde davranır: varsayılan `main` kapsamı, tek bir sahibin DM'leri arasında tek bir devreden oturumu paylaşırken (kişisel aracı varsayılanı), yapılandırılmış herhangi bir yalıtıcı kapsam (`per-peer`, `per-channel-peer`, `per-account-channel-peer`) DM oturumlarını kesin biçimde ayrı tutar. Kanala özgü `dmScope` geçersiz kılmaları, genel varsayılanlara göre önceliklidir; `identityLinks`, oturumları yalnızca açıkça bağlantılı gruplar içinde birleştirir, ilgisiz eşler arasında birleştirmez. Çok kullanıcılı gelen kutularının yalıtıcı bir kapsamı etkinleştirmesi beklenir (çalışma zamanı güvenlik denetimi, çok kullanıcılı DM trafiği algıladığında bunu önerir).

| Sonuç         | Hedefler                                                                   |
| -------------- | ------------------------------------------------------------------------- |
| Yeşil          | `make routing-precedence`, `make routing-identitylinks`                   |
| Kırmızı (beklenen) | `make routing-precedence-negative`, `make routing-identitylinks-negative` |

## İlgili

- [Tehdit modeli](/tr/security/THREAT-MODEL-ATLAS)
- [Tehdit modeline katkıda bulunma](/tr/security/CONTRIBUTING-THREAT-MODEL)
- [Olay müdahalesi](/tr/security/incident-response)
