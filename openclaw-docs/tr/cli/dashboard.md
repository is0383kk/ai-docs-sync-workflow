---
read_when:
    - Control UI'ı mevcut token'ınızla açmak istiyorsunuz
    - Tarayıcı başlatmadan URL'yi yazdırmak istiyorsunuz
summary: '`openclaw dashboard` için CLI başvurusu (Kontrol Arayüzünü açın)'
title: Kontrol Paneli
x-i18n:
    generated_at: "2026-07-26T23:12:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 168605e1e58827020b4d247afd513880335273e489995549377bc2dc1f8a3b25
    source_path: cli/dashboard.md
    workflow: 16
---

# `openclaw dashboard`

Mevcut kimlik doğrulamanızı kullanarak Denetim Arayüzü'nü açın.

```bash
openclaw dashboard
openclaw dashboard --no-open
openclaw dashboard --json
openclaw dashboard --yes
```

- `--no-open`: URL'yi yazdırır ancak tarayıcı başlatmaz.
- `--json`: tarayıcı açmadan, panoyu kullanmadan, istem göstermeden veya Gateway'i başlatmadan makine tarafından okunabilir tek bir bağlantı nesnesi yazdırır.
- `--yes`: gerektiğinde istem göstermeden Gateway'i başlatır/kurar.

## Makine tarafından okunabilir çıktı

Çözümlenmiş Denetim Arayüzü URL'sine ihtiyaç duyan masaüstü entegrasyonları ve betikler için `--json` kullanın:

```bash
openclaw dashboard --json
```

Yanıt; `url`, `httpUrl`, `wsUrl`, `port` ve `tokenIncluded` içerir. Gateway hazır değilse komut `{"ok":false,"reason":"..."}` döndürür ve sıfırdan farklı bir kodla çıkar. SecretRef tarafından yönetilen tokenlar hiçbir zaman `url` içine eklenmez.

Notlar:

- Yapılandırılmış `gateway.auth.token` SecretRef'lerini mümkün olduğunda çözümler.
- `gateway.tls.enabled` ayarını izler: TLS etkin Gateway'ler `https://` Denetim Arayüzü URL'lerini yazdırır/açar ve `wss://` üzerinden bağlanır.
- `lan` veya joker karakterli bir `custom` bağlaması için aynı ana makinedeki başlatmalar her zaman geri döngüyü kullanır; çünkü joker karakter bir tarayıcı hedefi değildir. Düz metin `tailnet` ve `custom` bağlamaları da tarayıcının güvenli bir bağlama sahip olması için `127.0.0.1` kullanır; TLS etkin belirli ana makineler, sertifika adlarının eşleşmesi için yapılandırılmış adresi korur.
- Komut, belirli bir arayüz bağlaması için kimliği doğrulanmış bir geri döngü URL'si sunmadan önce yapılandırılmış arayüzü yoklar ve bu arayüz ile `127.0.0.1` öğesinin aynı Gateway işleminin mülkiyetinde olduğunu doğrular. Belirsiz dinleyici mülkiyetinde işlem, durum yönergeleriyle güvenli biçimde başarısız olur.
- SecretRef tarafından yönetilen tokenlarda (çözümlenmiş veya çözümlenmemiş) yazdırılan/kopyalanan/açılan URL hiçbir zaman tokenı içermez; böylece harici gizli değerler terminal çıktısına, pano geçmişine veya tarayıcı başlatma bağımsız değişkenlerine sızmaz.
- `gateway.auth.token` SecretRef tarafından yönetiliyor ancak çözümlenemiyorsa komut, geçersiz bir token yer tutucusu yerine tokensız bir URL ve düzeltme yönergeleri yazdırır.
- Token ile kimliği doğrulanmış bir URL'nin panoya veya tarayıcıya iletilmesi başarısız olursa komut, token değerini yazdırmadan `OPENCLAW_GATEWAY_TOKEN`, `gateway.auth.token` ve URL parçası anahtarı `token` adlarını belirten güvenli bir manuel kimlik doğrulama ipucu kaydeder.

## İlgili

- [CLI referansı](/tr/cli)
- [Pano](/tr/web/dashboard)
