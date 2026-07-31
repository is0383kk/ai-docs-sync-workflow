---
read_when:
    - OpenResponses API ile iletişim kuran istemcileri entegre etme
    - Öğe tabanlı girdiler, istemci araç çağrıları veya SSE olayları istiyorsunuz
summary: Gateway üzerinden OpenResponses uyumlu bir /v1/responses HTTP uç noktası sunun
title: OpenResponses API'si
x-i18n:
    generated_at: "2026-07-26T23:59:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bfd6ca3bf0cecd761fde865b41a95cff3fc5681f74f31b3adae5cd2e0b0be95
    source_path: gateway/openresponses-http-api.md
    workflow: 16
---

Gateway, OpenResponses uyumlu bir `POST /v1/responses` uç noktası sunabilir. Bu uç nokta **varsayılan olarak devre dışıdır** ve portunu Gateway ile paylaşır (WS + HTTP çoklama): `http://<gateway-host>:<port>/v1/responses`.

İstekler normal bir Gateway ajan çalıştırması olarak yürütülür (`openclaw agent` ile aynı kod yolu); dolayısıyla yönlendirme, izinler ve yapılandırma Gateway'inizle aynıdır.

`gateway.http.endpoints.responses.enabled` ile etkinleştirin veya devre dışı bırakın. Etkinleştirildiğinde aynı uyumluluk yüzeyi ayrıca `GET /v1/models`, `GET /v1/models/{id}`, `POST /v1/embeddings` ve `POST /v1/chat/completions` uç noktalarını sunar.

## Kimlik doğrulama, güvenlik ve yönlendirme

İşletim davranışı [OpenAI Chat Completions](/tr/gateway/openai-http-api) ile aynıdır:

- Kimlik doğrulama yolu `gateway.auth.mode` ile aynıdır: paylaşılan gizli anahtar (`token`/`password`) `Authorization: Bearer <token-or-password>` kullanır; güvenilen proxy, kimlik algılayan proxy üstbilgilerini kullanır (aynı ana makinedeki geri döngü proxy'leri `gateway.auth.trustedProxy.allowLoopback = true` gerektirir; `Forwarded`/`X-Forwarded-*`/`X-Real-IP` üstbilgilerinden hiçbiri mevcut değilse `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` aracılığıyla aynı ana makinede doğrudan geri dönüş sağlanır); özel girişteki `none` kimlik doğrulama üstbilgisi gerektirmez. Bkz. [Güvenilen proxy kimlik doğrulaması](/tr/gateway/trusted-proxy-auth).
- Uç noktayı Gateway örneğine tam operatör erişimi olarak değerlendirin.
- Paylaşılan gizli anahtar kimlik doğrulama modları, bearer tarafından bildirilen daha dar kapsamlı bir `x-openclaw-scopes` değerini yok sayar ve tam varsayılan operatör kapsamı kümesini geri yükler: `operator.admin`, `operator.approvals`, `operator.pairing`, `operator.read`, `operator.talk.secrets`, `operator.write`. Bu uç noktadaki sohbet turları, sahibi gönderen turlar olarak değerlendirilir.
- Güvenilen, kimlik taşıyan HTTP modları (güvenilen proxy veya `gateway.auth.mode="none"`), mevcut olduğunda `x-openclaw-scopes` değerini dikkate alır; aksi takdirde varsayılan operatör kapsamı kümesine geri döner. Sahip semantiği yalnızca çağıran taraf kapsamları açıkça daralttığında ve `operator.admin` değerini dışarıda bıraktığında kaybolur.
- Ajanları `model: "openclaw"`, `"openclaw/default"`, `"openclaw/<agentId>"` veya `x-openclaw-agent-id` üstbilgisiyle seçin.
- Seçilen ajanın arka uç modelini geçersiz kılmak için `x-openclaw-model` kullanın (kimlik taşıyan kimlik doğrulama yollarında `operator.admin` gerektirir).
- Açık oturum yönlendirmesi için `x-openclaw-session-key` kullanın (ayrılmış bir ad alanı kullanıyorsa `400 invalid_request_error` ile reddedilir: `subagent:`, `cron:`, `acp:`).
- Varsayılan olmayan sentetik giriş kanalı bağlamı için `x-openclaw-message-channel` kullanın.

Ajan hedefli modeller, `openclaw/default`, gömme geçişi ve arka uç modeli geçersiz kılmalarına ilişkin kurallı açıklama için [OpenAI Chat Completions](/tr/gateway/openai-http-api#agent-first-model-contract) sayfasına bakın.

Bkz. [Operatör kapsamları](/tr/gateway/operator-scopes) ve [Güvenlik](/tr/gateway/security).

## Oturum davranışı

Uç nokta varsayılan olarak **her istek için durumsuzdur** (her çağrıda yeni bir oturum anahtarı oluşturulur).

İstek bir OpenResponses `user` dizesi içeriyorsa Gateway, yinelenen çağrıların aynı ajan oturumunu paylaşabilmesi için bu dizeden kararlı bir oturum anahtarı türetir.

İstek aynı ajan/kullanıcı/istenen oturum kapsamında kaldığında (kimlik doğrulama konusu, ajan kimliği ve `x-openclaw-session-key` ile eşleştirilir) `previous_response_id` önceki yanıtın oturumunu yeniden kullanır.

## İstek biçimi

| Alan                                                            | Destek                                                                                                                        |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `input`                                                          | Dize veya öğe nesneleri dizisi.                                                                                               |
| `instructions`                                                   | Sistem istemiyle birleştirilir.                                                                                                 |
| `tools`                                                          | İstemci araç tanımları (işlev araçları).                                                                                      |
| `tool_choice`                                                    | İstemci araçlarını filtrelemek veya zorunlu kılmak için `"auto"`, `"none"`, `"required"` veya `{ "type": "function", "name": "..." }`.                |
| `stream`                                                         | SSE akışını etkinleştirir.                                                                                                         |
| `max_output_tokens`                                              | En iyi çabayla uygulanan çıktı sınırı (sağlayıcıya bağlıdır).                                                                                 |
| `temperature`                                                    | En iyi çabayla uygulanan örnekleme sıcaklığı. Sabit sunucu tarafı örneklemesi kullanan ChatGPT tabanlı Codex Responses arka ucu tarafından yok sayılır. |
| `top_p`                                                          | En iyi çabayla uygulanan çekirdek örneklemesi. `temperature` ile aynı Codex Responses kısıtlaması geçerlidir.                                                    |
| `user`                                                           | Kararlı oturum yönlendirmesi.                                                                                                        |
| `previous_response_id`                                           | Oturum sürekliliği (yukarıya bakın).                                                                                                |
| `max_tool_calls`, `reasoning`, `metadata`, `store`, `truncation` | Kabul edilir ancak şu anda yok sayılır.                                                                                                |

## Öğeler (girdi)

### `message`

Roller: `system`, `developer`, `user`, `assistant`.

- `system` ve `developer` sistem istemine eklenir.
- En son `user` veya `function_call_output` öğesi "geçerli ileti" olur.
- Önceki kullanıcı/asistan iletileri bağlam amacıyla geçmişe dahil edilir.

### `function_call_output` (tur tabanlı araçlar)

Araç sonuçlarını modele geri gönderin:

```json
{
  "type": "function_call_output",
  "call_id": "call_123",
  "output": "{\"temperature\": \"72F\"}"
}
```

### `reasoning` ve `item_reference`

Şema uyumluluğu için kabul edilir ancak istem oluşturulurken yok sayılır.

## Araçlar (istemci tarafı işlev araçları)

Araçları `tools: [{ type: "function", name, description?, parameters? }]` ile sağlayın.

Ajan bir aracı çağırırsa yanıt bir `function_call` çıktı öğesi döndürür. Turu sürdürmek için `function_call_output` içeren bir takip isteği gönderin.

`tool_choice: "required"` ve işleve sabitlenmiş `tool_choice` için uç nokta, sunulan istemci işlev araçları kümesini daraltır, çalışma zamanına yanıt vermeden önce bir istemci aracını çağırması talimatını verir ve tur eşleşen yapılandırılmış bir istemci aracı çağrısı içermiyorsa `/v1/chat/completions` sözleşmesine uygun olarak turu reddeder. Akışsız istekler bir `api_error` ile `502` döndürür; akışlı istekler bir `response.failed` olayı yayınlar.

## Görüntüler (`input_image`)

Base64 veya URL kaynaklarını destekler:

```json
{
  "type": "input_image",
  "source": { "type": "url", "url": "https://example.com/image.png" }
}
```

İzin verilen MIME türleri (varsayılan): `image/jpeg`, `image/png`, `image/gif`, `image/webp`, `image/heic`, `image/heif`. Azami boyut (varsayılan): 10MB.

## Dosyalar (`input_file`)

Base64 veya URL kaynaklarını destekler:

```json
{
  "type": "input_file",
  "source": {
    "type": "base64",
    "media_type": "text/plain",
    "data": "SGVsbG8gV29ybGQh",
    "filename": "hello.txt"
  }
}
```

İzin verilen MIME türleri (varsayılan): `text/plain`, `text/markdown`, `text/html`, `text/csv`, `application/json`, `application/pdf`. Azami boyut (varsayılan): 5MB.

Geçerli davranış:

- Dosya içeriğinin kodu çözülür ve kullanıcı iletisine değil **sistem istemine** eklenir; böylece geçici kalır (oturum geçmişinde kalıcılaştırılmaz).
- Kodu çözülmüş dosya metni eklenmeden önce **güvenilmeyen harici içerik** olarak sarmalanır; böylece dosya baytları güvenilen talimatlar olarak değil, veri olarak değerlendirilir. Eklenen blok açık sınır işaretçileri (`<<<EXTERNAL_UNTRUSTED_CONTENT id="...">>>` / `<<<END_EXTERNAL_UNTRUSTED_CONTENT id="...">>>`) ve bir `Source: External` meta veri satırı kullanır. İstem bütçesini korumak için uzun `SECURITY NOTICE:` başlığını kasıtlı olarak dışarıda bırakır; sınır işaretçileri ve meta veriler yine de geçerlidir.
- PDF'ler önce metin için ayrıştırılır. Az miktarda metin bulunursa ilk sayfalar görüntülere dönüştürülüp modele iletilir ve eklenen dosya bloğunda `[PDF content rendered to images]` yer tutucusu kullanılır.

PDF ayrıştırması; metin çıkarma ve sayfa oluşturma için `clawpdf` ile paketlenmiş PDFium WebAssembly çalışma zamanını kullanan, paketle birlikte sunulan `document-extract` plugin'i tarafından sağlanır.

URL getirme varsayılanları:

- `files.allowUrl`: `true`
- `images.allowUrl`: `true`
- `maxUrlParts`: `8` (istek başına toplam URL tabanlı `input_file` + `input_image` bölümü)
- İstekler koruma altındadır (DNS çözümlemesi, özel IP engelleme, yönlendirme sınırları, zaman aşımları).
- Her girdi türü için isteğe bağlı ana makine adı izin listeleri desteklenir (`files.urlAllowlist`, `images.urlAllowlist`): tam ana makine (`"cdn.example.com"`) veya joker karakterli alt etki alanları (`"*.assets.example.com"`, kök etki alanıyla eşleşmez). Boş veya belirtilmemiş izin listeleri, ana makine adı izin listesi kısıtlaması olmadığı anlamına gelir.
- URL tabanlı getirmeleri tamamen devre dışı bırakmak için `files.allowUrl: false` ve/veya `images.allowUrl: false` ayarlayın.

## Dosya ve görüntü sınırları

Uç nokta yerleşik bir 20 MB istek gövdesi sınırı kullanır. Dosya ve görüntü kaynağı
ilkesi `gateway.http.endpoints.responses` altında yapılandırılabilir durumda kalır:

```json5
{
  gateway: {
    http: {
      endpoints: {
        responses: {
          enabled: true,
          maxUrlParts: 8,
          files: {
            allowUrl: true,
            urlAllowlist: ["cdn.example.com", "*.assets.example.com"],
            allowedMimes: [
              "text/plain",
              "text/markdown",
              "text/html",
              "text/csv",
              "application/json",
              "application/pdf",
            ],
            maxBytes: 5242880,
            maxChars: 60000,
            maxRedirects: 3,
            timeoutMs: 10000,
            pdf: {
              maxPages: 4,
              maxPixels: 4000000,
              minTextChars: 200,
            },
          },
          images: {
            allowUrl: true,
            urlAllowlist: ["images.example.com"],
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

Belirtilmediğindeki varsayılanlar:

| Anahtar                      | Varsayılan   |
| ------------------------ | --------- |
| `maxUrlParts`            | 8         |
| `files.maxBytes`         | 5MB       |
| `files.maxChars`         | 60k       |
| `files.maxRedirects`     | 3         |
| `files.timeoutMs`        | 10s       |
| `files.pdf.maxPages`     | 4         |
| `files.pdf.maxPixels`    | 4,000,000 |
| `files.pdf.minTextChars` | 200       |
| `images.maxBytes`        | 10MB      |
| `images.maxRedirects`    | 3         |
| `images.timeoutMs`       | 10s       |

HEIC/HEIF `input_image` kaynakları, paylaşılan OpenClaw görüntü işlemcisi (Rastermill) üzerinden sağlayıcıya iletilmeden önce JPEG biçimine normalleştirilir; işlemci, harici codec desteği gerektiren biçimler için bir sistem dönüştürücüsüne (`sips`, ImageMagick, GraphicsMagick veya ffmpeg) geri döner.

Güvenlik notu: URL izin listeleri, getirme işleminden önce ve yönlendirme adımlarında uygulanır. Bir ana bilgisayar adını izin listesine eklemek, özel/dahili IP engellemesini atlamaz. İnternete açık Gateway'ler için uygulama düzeyindeki korumalara ek olarak ağ çıkış denetimleri uygulayın. Bkz. [Güvenlik](/tr/gateway/security).

## Akış (SSE)

Server-Sent Events almak için `stream: true` olarak ayarlayın:

- `Content-Type: text/event-stream`
- Her olay satırı `event: <type>` ve `data: <json>`
- Akış `data: [DONE]` ile sona erer

Şu anda yayımlanan olay türleri: `response.created`, `response.in_progress`, `response.output_item.added`, `response.content_part.added`, `response.output_text.delta`, `response.output_text.done`, `response.content_part.done`, `response.output_item.done`, `response.completed`, `response.failed` (hata durumunda).

## Kullanım

Temel sağlayıcı token sayılarını bildirdiğinde `usage` doldurulur. OpenClaw, bu sayaçlar aşağı akış durum/oturum yüzeylerine ulaşmadan önce `input_tokens` / `output_tokens` ve `prompt_tokens` / `completion_tokens` dâhil olmak üzere yaygın OpenAI tarzı diğer adları normalleştirir.

## Hatalar

Hatalar aşağıdakine benzer bir JSON nesnesi kullanır:

```json
{ "error": { "message": "...", "type": "invalid_request_error" } }
```

Yaygın durumlar: `400` geçersiz istek gövdesi, `401` eksik/geçersiz kimlik doğrulama, `403` eksik operatör kapsamı, `405` yanlış yöntem, `429` çok fazla başarısız kimlik doğrulama denemesi (`Retry-After` ile).

## Örnekler

Akışsız:

```bash
curl -sS http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "input": "hi"
  }'
```

Akışlı:

```bash
curl -N http://127.0.0.1:18789/v1/responses \
  -H 'Authorization: Bearer YOUR_TOKEN' \
  -H 'Content-Type: application/json' \
  -H 'x-openclaw-agent-id: main' \
  -d '{
    "model": "openclaw",
    "stream": true,
    "input": "hi"
  }'
```

## İlgili

- [OpenAI sohbet tamamlamaları](/tr/gateway/openai-http-api)
- [Operatör kapsamları](/tr/gateway/operator-scopes)
- [OpenAI](/tr/providers/openai)
