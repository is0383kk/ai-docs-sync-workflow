---
read_when:
    - OpenAI Chat Completions bekleyen araçları entegre etme
summary: Gateway üzerinden OpenAI uyumlu bir /v1/chat/completions HTTP uç noktası sunun
title: OpenAI sohbet tamamlamaları
x-i18n:
    generated_at: "2026-07-26T23:57:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4cc5a1a56972bb9070da0f8f60d6efd673cc1d1d516b730c55bc9d171fc7a5b3
    source_path: gateway/openai-http-api.md
    workflow: 16
---

Gateway, OpenAI uyumlu küçük bir Chat Completions yüzeyi sunabilir. Bu yüzey **varsayılan olarak devre dışıdır**.

Etkinleştirildiğinde bunların tümünü Gateway ile aynı bağlantı noktasında sunar (WS + HTTP çoklama):

| Yöntem | Yol                    |
| ------ | ---------------------- |
| POST   | `/v1/chat/completions` |
| GET    | `/v1/models`           |
| GET    | `/v1/models/{id}`      |
| POST   | `/v1/embeddings`       |
| POST   | `/v1/responses`        |

İstekler normal bir Gateway ajan çalıştırması olarak yürütülür (`openclaw agent` ile aynı kod yolu); dolayısıyla yönlendirme, izinler ve yapılandırma Gateway'inizle eşleşir.

## Uç noktayı etkinleştirme

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: { enabled: true },
      },
    },
  },
}
```

Devre dışı bırakmak için `enabled: false` olarak ayarlayın (veya bu ayarı atlayın).

## Güvenlik sınırı (önemli)

Bu uç noktayı gateway örneğine **tam operatör erişimi** olarak değerlendirin:

- Bu uç nokta için geçerli bir Gateway belirteci/parolası, dar kapsamlı bir kullanıcı başına yetki değil, sahip/operatör kimlik bilgisiyle eşdeğerdir.
- İstekler, güvenilir operatör eylemleriyle aynı kontrol düzlemi ajan yolundan geçer; dolayısıyla hedef ajanın politikası hassas araçlara izin veriyorsa bu uç nokta bunları kullanabilir.
- Yalnızca geri döngü/tailnet/özel giriş üzerinde tutun. Genel internete açmayın.

Kimlik doğrulama matrisi:

| Kimlik doğrulama yolu                                                                                            | Davranış                                                                                                                                                                                                                                                                                                  |
| ---------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"` veya `"password"` + `Authorization: Bearer ...`                            | Paylaşılan gateway gizli bilgisinin elde bulunduğunu kanıtlar. Herhangi bir `x-openclaw-scopes` üst bilgisini yok sayar ve tam varsayılan operatör kapsamı kümesini geri yükler: `operator.admin`, `operator.approvals`, `operator.pairing`, `operator.read`, `operator.talk.secrets`, `operator.write`. Sohbet dönüşlerini sahip-gönderici dönüşleri olarak değerlendirir. |
| Kimlik taşıyan güvenilir HTTP (güvenilir proxy kimlik doğrulaması veya özel girişte `gateway.auth.mode="none"`) | Mevcut olduğunda `x-openclaw-scopes` değerini dikkate alır; olmadığında varsayılan operatör kapsamı kümesine geri döner. Yalnızca çağıran kapsamları açıkça daraltıp `operator.admin` değerini dışarıda bıraktığında sahip semantiğini kaybeder. `x-openclaw-model` gibi sahip düzeyindeki kontroller için `operator.admin` gerektirir.                        |

Bkz. [Operatör kapsamları](/tr/gateway/operator-scopes), [Güvenlik](/tr/gateway/security) ve [Uzaktan erişim](/tr/gateway/remote).

## Kimlik doğrulama

Gateway kimlik doğrulama yapılandırmasını kullanır (bu modun ayrıntıları için bkz. [Güvenilir proxy kimlik doğrulaması](/tr/gateway/trusted-proxy-auth)):

| Mod                                | Kimlik doğrulama yöntemi                                                                                                                                                                     |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `gateway.auth.mode="token"`         | `Authorization: Bearer <token>`. `gateway.auth.token` veya `OPENCLAW_GATEWAY_TOKEN` aracılığıyla ayarlayın.                                                                                              |
| `gateway.auth.mode="password"`      | `Authorization: Bearer <password>`. `gateway.auth.password` veya `OPENCLAW_GATEWAY_PASSWORD` aracılığıyla ayarlayın.                                                                                     |
| `gateway.auth.mode="trusted-proxy"` | Yapılandırılmış kimlik duyarlı proxy üzerinden yönlendirin; gerekli kimlik üst bilgilerini ekler. Aynı ana makinedeki geri döngü proxy'leri açıkça `gateway.auth.trustedProxy.allowLoopback = true` gerektirir. |
| `gateway.auth.mode="none"`          | Kimlik doğrulama üst bilgisi gerekmez (yalnızca özel giriş).                                                                                                                                         |

Notlar:

- Bir `trusted-proxy` gateway üzerinde proxy'yi atlayan aynı ana makine çağıranları doğrudan `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` kullanımına geri dönebilir. Herhangi bir `Forwarded`, `X-Forwarded-*` veya `X-Real-IP` üst bilgisi kanıtı, isteğin bunun yerine güvenilir proxy yolunda kalmasını sağlar.
- `gateway.auth.rateLimit` yapılandırılmışsa ve çok fazla kimlik doğrulama girişimi başarısız olursa uç nokta, `Retry-After` üst bilgisiyle `429` döndürür.

## Bu uç nokta ne zaman kullanılmalı?

- Entegrasyonunuz aynı gateway için yalnızca başka bir operatör/istemci yüzeyiyse yeni bir yerleşik kanal eklemek yerine bunu tercih edin.
- Uzak bir gateway'e doğrudan bağlanan yerel mobil istemcilerde, cihazın paylaşılan bir HTTP belirtecine/parolasına ihtiyaç duymaması için eşleştirilmiş cihaz önyükleme/cihaz belirteci akışıyla [WebChat](/tr/web/webchat) veya [Gateway Protokolünü](/tr/gateway/protocol) tercih edin.
- Kendi kullanıcıları, odaları, Webhook teslimatı veya giden aktarımı olan harici bir mesajlaşma ağıyla entegrasyon yaparken bunun yerine bir kanal Plugin'i oluşturun. Bkz. [Plugin oluşturma](/tr/plugins/building-plugins).

## Ajan öncelikli model sözleşmesi

OpenClaw, OpenAI `model` alanını ham sağlayıcı model kimliği olarak değil, bir **ajan hedefi** olarak değerlendirir.

| `model` değeri                                | Yönlendirildiği yer                                                                                                                |
| -------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `openclaw`                                   | Yapılandırılmış varsayılan ajan                                                                                                 |
| `openclaw/default`                           | Yapılandırılmış varsayılan ajan (kararlı takma ad; gerçek varsayılan ajan kimliği ortamlar arasında değişse bile sabit kodlamak güvenlidir) |
| `openclaw/<agentId>` veya `openclaw:<agentId>` | Belirli ajan                                                                                                           |
| `agent:<agentId>`                            | Belirli ajan (uyumluluk takma adı)                                                                                     |

İsteğe bağlı istek üst bilgileri:

| Üst bilgi                                          | Etki                                                                                                                                                                                                                                                                      |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `x-openclaw-model: <provider/model-or-bare-id>` | Seçilen ajanın arka uç modelini geçersiz kılar. Paylaşılan gizli bilgi taşıyıcı belirtecini kullanan çağıranlar bunu doğrudan kullanabilir; kimlik taşıyan çağıranlar (güvenilir proxy veya `x-openclaw-scopes` içeren özel kimlik doğrulamasız giriş) `operator.admin` gerektirir, aksi takdirde `403 missing scope: operator.admin`. |
| `x-openclaw-agent-id: <agentId>`                | Ajan seçimi için uyumluluk geçersiz kılması.                                                                                                                                                                                                                                 |
| `x-openclaw-session-key: <sessionKey>`          | Açık oturum yönlendirmesi. Ayrılmış bir dahili ad alanı (`subagent:`, `cron:`, `acp:`) kullanıyorsa `400 invalid_request_error` ile reddedilir.                                                                                                                                |
| `x-openclaw-message-channel: <channel>`         | Kanal duyarlı istemler/politikalar için sentetik giriş kanalı bağlamını ayarlar.                                                                                                                                                                                              |

`/v1/models`, arka uç sağlayıcı modellerini veya alt ajanları değil, üst düzey ajan hedeflerini (`openclaw`, `openclaw/default`, `openclaw/<agentId>`) listeler; alt ajanlar dahili yürütme topolojisi olarak kalır. `x-openclaw-model` değerini atlarsanız seçilen ajan normal yapılandırılmış modeliyle çalışır.

`/v1/embeddings`, aynı ajan hedefi `model` kimliklerini kullanır. Belirli bir gömme modeli seçmek için `x-openclaw-model` gönderin (paylaşılan gizli bilgi çağıranından veya `operator.admin` kapsamına sahip kimlik taşıyan bir çağırandan); aksi takdirde istek, seçilen ajanın normal gömme kurulumunu kullanır.

## Oturum davranışı

Uç nokta varsayılan olarak **istek başına durumsuzdur** (her çağrıda yeni bir oturum anahtarı oluşturulur).

İstek bir OpenAI `user` dizesi içeriyorsa Gateway, yinelenen çağrıların bir ajan oturumunu paylaşabilmesi için bundan kararlı bir oturum anahtarı türetir. Özel uygulamalarda konuşma dizisi başına aynı `user` değerini yeniden kullanın; birden fazla konuşmanın/cihazın tek bir OpenClaw oturumunu paylaşmasını istemiyorsanız hesap düzeyindeki tanımlayıcılardan kaçının. `x-openclaw-session-key` değerini yalnızca birden fazla istemci/dizi arasında açık yönlendirme denetimine ihtiyaç duyduğunuzda, yukarıdaki ayrılmış ad alanlarından kaçınan uygulama sahipli anahtarlarla kullanın.

## İstek sınırları

Uç nokta, istek gövdesi başına 20 MB, en son kullanıcı iletisinden 8 `image_url`
parçası ve toplam 20 MB kodu çözülmüş görüntü
verisi için yerleşik sınırlar kullanır. Görüntü kaynağı politikası
`gateway.http.endpoints.chatCompletions.images` altında yapılandırılabilir olmaya devam eder:

```json5
{
  gateway: {
    http: {
      endpoints: {
        chatCompletions: {
          enabled: true,
          images: {
            allowUrl: false,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "image/jpeg",
              "image/png",
              "image/gif",
              "image/webp",
              "image/heic",
              "image/heif",
            ],
            maxBytes: 10485760,
            maxRedirects: 3,
            timeoutMs: 10000,
          },
        },
      },
    },
  },
}
```

Görüntü ayarlarının varsayılanları:

| Anahtar                   | Varsayılan                                                             |
| --------------------- | ------------------------------------------------------------------- |
| `images.allowUrl`     | `false` (URL kaynaklı `image_url` parçaları etkinleştirilmedikçe reddedilir) |
| `images.maxBytes`     | Görüntü başına 10MB                                                      |
| `images.maxRedirects` | 3                                                                   |
| `images.timeoutMs`    | 10s                                                                 |

HEIC/HEIF `image_url` kaynakları kabul edilir ve paylaşılan OpenClaw görüntü işleyicisi (Rastermill) üzerinden sağlayıcıya teslim edilmeden önce JPEG'e dönüştürülür; bu işleyici, harici kodek desteği gerektiren biçimler için sistem dönüştürücüsüne (`sips`, ImageMagick, GraphicsMagick veya ffmpeg) geri döner.

Güvenlik notu: Bir ana makine adının izin verilenler listesine eklenmesi, özel/dahili IP engellemesini aşmaz. İnternete açık gateway'lerde uygulama düzeyindeki korumalara ek olarak ağ çıkış kontrolleri uygulayın. Bkz. [Güvenlik](/tr/gateway/security).

## Sohbet aracı sözleşmesi

`/v1/chat/completions`, yaygın OpenAI Chat istemcileriyle uyumlu bir işlev aracı alt kümesini destekler.

### Desteklenen istek alanları

| Alan                       | Notlar                                                                                                                                        |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| `tools`                    | `{ "type": "function", "function": { ... } }` dizisi                                                                                         |
| `tool_choice`              | `"auto"`, `"none"`, `"required"` veya `{ "type": "function", "function": { "name": "..." } }`                                                 |
| `messages[*].role: "tool"` | Takip turları                                                                                                                                |
| `messages[*].tool_call_id` | Bir araç sonucunu önceki bir araç çağrısına bağlar                                                                                           |
| `max_completion_tokens`    | Sayı; çağrı başına toplam tamamlama token'ları üst sınırı (akıl yürütme token'ları dâhil). Geçerli alan adı; hem bu hem de `max_tokens` gönderildiğinde kullanılır. |
| `max_tokens`               | Sayı; eski takma ad, `max_completion_tokens` de mevcutsa yok sayılır.                                                                        |
| `temperature`              | 0-2 arası sayı; mümkün olan en iyi şekilde üst sağlayıcıya iletilir. Aralık dışındaysa `400 invalid_request_error`.                           |
| `top_p`                    | 0-1 arası sayı; mümkün olan en iyi şekilde uygulanır. Aralık dışındaysa `400 invalid_request_error`.                                          |
| `frequency_penalty`        | -2.0 ile 2.0 arası sayı; mümkün olan en iyi şekilde uygulanır. Aralık dışındaysa `400 invalid_request_error`.                                |
| `presence_penalty`         | -2.0 ile 2.0 arası sayı; mümkün olan en iyi şekilde uygulanır. Aralık dışındaysa `400 invalid_request_error`.                                |
| `seed`                     | Tam sayı; mümkün olan en iyi şekilde uygulanır. Tam sayı olmayan değerler için `400 invalid_request_error`.                                  |
| `stop`                     | Dize veya en fazla 4 dizelik dizi; mümkün olan en iyi şekilde uygulanır. 4'ten fazla dizi ya da dize olmayan/boş girdiler için `400 invalid_request_error`. |

Tüm örnekleme ve token üst sınırı alanları aynı ajan akış parametresi kanalını kullanır ve mümkün olan en iyi şekilde iletilir:

- Token üst sınırı: kablo alanı adı sağlayıcı aktarımı tarafından seçilir: OpenAI ailesi uç noktaları için `max_completion_tokens`, yalnızca eski adı kabul eden sağlayıcılar (Mistral, Chutes) için `max_tokens`.
- `stop`, aktarımın durdurma alanına eşlenir: Chat Completions arka uçları için `stop`, Anthropic için `stop_sequences`. OpenAI Responses API'sinde durdurma parametresi bulunmadığından `stop`, Responses tabanlı modellere uygulanmaz.
- ChatGPT tabanlı Codex Responses arka ucu sabit sunucu tarafı örnekleme kullanır ve istek bu arka uca ulaşmadan önce `temperature`/`top_p` alanlarını (`max_output_tokens`, `metadata`, `prompt_cache_retention`, `service_tier` ile birlikte) kaldırır.

### Desteklenmeyen varyantlar

Şunlar için `400 invalid_request_error` döndürür:

- dizi olmayan `tools`, işlev olmayan araç girdileri veya eksik `tool.function.name`
- `allowed_tools` ve `custom` gibi `tool_choice` varyantları
- sağlanan bir araçla eşleşmeyen `tool_choice.function.name` değerleri

`tool_choice: "required"` ve işleve sabitlenmiş `tool_choice` için uç nokta, sunulan istemci işlev aracı kümesini daraltır, çalışma zamanına yanıt vermeden önce bir istemci aracını çağırması talimatını verir ve ajan yanıtında eşleşen yapılandırılmış bir istemci aracı çağrısı yoksa hata verir. Bu, tüm dahili OpenClaw ajan araçlarına değil, çağıranın sağladığı HTTP `tools` listesine uygulanır.

### Akışsız araç yanıtı biçimi

Ajan araçları çağırdığında yanıt şunları kullanır:

- `choices[0].finish_reason = "tool_calls"`
- `id`, `type: "function"`, `function.name`, `function.arguments` (JSON dizesi) içeren `choices[0].message.tool_calls[]` girdileri
- Araç çağrısından önce `choices[0].message.content` içindeki asistan açıklaması (boş olabilir)

### Akışlı araç yanıtı biçimi

`stream: true` olduğunda araç çağrıları artımlı SSE parçaları olarak gelir: ilk asistan rolü deltası, isteğe bağlı asistan açıklaması deltaları, araç kimliğini ve bağımsız değişken parçalarını taşıyan bir veya daha fazla `delta.tool_calls` parçası, ardından `finish_reason: "tool_calls"` ve `data: [DONE]` içeren son bir parça.

`stream_options.include_usage=true` ise `[DONE]` öncesinde sonda bir kullanım parçası yayınlanır.

### Araç takip döngüsü

`tool_calls` alındıktan sonra istenen işlevleri yürütün ve önceki asistan araç çağrısı iletisini ve eşleşen `tool_call_id` değerlerine sahip bir veya daha fazla `role: "tool"` iletisini içeren bir takip isteği gönderin. Bu işlem, nihai yanıtı üretmek için aynı ajan akıl yürütme döngüsünü sürdürür.

## Akış (SSE)

Server-Sent Events almak için `stream: true` ayarlayın:

- `Content-Type: text/event-stream`
- Her olay satırı `data: <json>` biçimindedir
- Akış `data: [DONE]` ile sona erer

## Open WebUI hızlı kurulumu

- Temel URL: `http://127.0.0.1:18789/v1`
- macOS'ta Docker temel URL'si: `http://host.docker.internal:18789/v1`
- API anahtarı: Gateway bearer token'ınız
- Model: `openclaw/default`

Beklenen davranış: `GET /v1/models`, `openclaw/default` öğesini listeler ve Open WebUI bunu sohbet modeli kimliği olarak kullanır. Belirli bir arka uç sağlayıcısı/modeli için ajanın normal varsayılan modelini ayarlayın veya `x-openclaw-model` gönderin (paylaşılan gizli anahtar kullanan çağıran ya da `operator.admin` değerine sahip, kimlik taşıyan çağıran).

Hızlı duman testi:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Bu, `openclaw/default` döndürürse çoğu Open WebUI kurulumu aynı temel URL ve token ile bağlanabilir.

## Örnekler

Tek bir uygulama konuşması için kararlı oturum:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "user": "conv:YOUR_CONVERSATION_ID",
    "messages": [{"role":"user","content":"Bugünkü görevlerimi özetle"}]
  }'
```

Aynı ajan oturumunu sürdürmek için bu konuşmanın sonraki çağrılarında aynı `user` değerini yeniden kullanın.

Akışsız:

```bash
curl -sS http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "openclaw/default",
    "messages": [{"role":"user","content":"merhaba"}]
  }'
```

Akışlı:

```bash
curl -N http://127.0.0.1:18789/v1/chat/completions \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/gpt-5.4' \
  -d '{
    "model": "openclaw/research",
    "stream": true,
    "messages": [{"role":"user","content":"merhaba"}]
  }'
```

Modelleri listeleyin:

```bash
curl -sS http://127.0.0.1:18789/v1/models \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Tek bir modeli getirin:

```bash
curl -sS http://127.0.0.1:18789/v1/models/openclaw%2Fdefault \
  -H 'Authorization: Bearer YOUR_TOKEN'
```

Gömme vektörleri oluşturun:

```bash
curl -sS http://127.0.0.1:18789/v1/embeddings \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-model: openai/text-embedding-3-small' \
  -d '{
    "model": "openclaw/default",
    "input": ["alpha", "beta"]
  }'
```

`/v1/embeddings`, dize veya dize dizisi olarak `input` destekler.

## İlgili

- [Yapılandırma referansı](/tr/gateway/configuration-reference)
- [Operatör kapsamları](/tr/gateway/operator-scopes)
- [OpenAI](/tr/providers/openai)
