---
read_when:
    - Control UI'da asistan çıktısının işlenmesini değiştirme
    - '`[embed ...]`, yapılandırılmış medya, yanıt veya ses sunumu yönergelerinde hata ayıklama'
summary: Yapılandırılmış medya, gömmeler, ses ipuçları ve yanıtlar için zengin çıktı protokolü
title: Zengin çıktı protokolü
x-i18n:
    generated_at: "2026-07-26T23:40:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbfe68f38c871f5f6d2811eb52b18d0143606f30283023ae96db64543eed95a1
    source_path: reference/rich-output-protocol.md
    workflow: 16
---

Asistan çıktısı, teslimat/işleme yönergelerini birkaç özel kanal üzerinden taşır:

- Ek teslimatı için yapılandırılmış `mediaUrl` / `mediaUrls` alanları.
- Ses sunumu ipuçları için `[[audio_as_voice]]`.
- Yanıt meta verileri için `[[reply_to_current]]` / `[[reply_to:<id>]]`.
- Control UI zengin işlemesi için `[embed ...]`.

Yapılandırılmış medya alanları ve `[[...]]` etiketleri teslimat meta verileridir. `[embed ...]`, yalnızca web'e yönelik ayrı zengin işleme yoludur; bir medya takma adı değildir.

## Medya ekleri

Uzak ekler, herkese açık `https:` URL'leri olmalıdır. `http:`, geri döngü, bağlantı yerel, özel ve dahili ana bilgisayar adları ek yönergeleri olarak reddedilir; sunucu tarafındaki medya getiricileri bunlara ek olarak kendi ağ korumalarını uygular.

Yerel ekler mutlak yolları, çalışma alanına göreli yolları veya ana dizine göreli `~/` yollarını kabul eder. Teslimattan önce yine de aracı dosya okuma politikasından ve medya türü kontrollerinden geçerler.

<Warning>
Araçlardan, Plugin'lerden, akış bloklarından, tarayıcı çıktısından veya mesaj eylemlerinden ekler için metin komutları yayınlamayın. Bunun yerine yapılandırılmış medya alanlarını kullanın:

```json
{ "message": "İşte görseliniz.", "mediaUrl": "/workspace/image.png" }
```

Eski nihai yanıt metni uyumluluk amacıyla hâlâ normalleştirilebilir, ancak bu genel bir Plugin/araç protokolü değildir.
</Warning>

Düz Markdown görsel söz dizimi (`![alt](url)`) varsayılan olarak metin şeklinde kalır. Markdown görsellerini medya yanıtları olarak işlemek isteyen kanallar, giden bağdaştırıcılarında bunu etkinleştirir; Telegram bunu yaptığı için `![alt](url)` bir medya ekine dönüşür.

Blok akışı etkinleştirildiğinde medya, yapılandırılmış yük alanlarında taşınmalıdır. Aynı medya URL'si hem akışla gönderilen bir blokta hem de nihai asistan yükünde tekrar görünürse OpenClaw bunu bir kez teslim eder ve yinelenen öğeyi nihai yükten çıkarır.

## `[embed ...]`

`[embed ...]`, Control UI için aracıya yönelik tek zengin işleme söz dizimidir. Kendiliğinden kapanan örnek:

```text
[embed ref="cv_123" title="Status" /]
```

Kurallar:

- `[view ...]` artık yeni çıktılar için geçerli değildir.
- Gömme kısa kodları yalnızca asistan mesajı yüzeyinde işlenir.
- Yalnızca URL destekli gömmeler işlenir; `ref="..."` veya `url="..."` kullanın.
- Blok biçimindeki satır içi HTML gömme kısa kodları işlenmez.
- Web kullanıcı arayüzü kısa kodu görünür metinden çıkarır ve gömmeyi satır içinde işler.

## Depolanan işleme biçimi

Normalleştirilmiş/depolanmış asistan içerik bloğu, yapılandırılmış bir `canvas` öğesidir:

```json
{
  "type": "canvas",
  "preview": {
    "kind": "canvas",
    "surface": "assistant_message",
    "render": "url",
    "viewId": "cv_123",
    "url": "/__openclaw__/canvas/documents/cv_123/index.html",
    "title": "Status",
    "preferredHeight": 320
  }
}
```

`present_view` tanınmaz; depolanan/işlenen zengin bloklar her zaman bu `canvas` biçimini kullanır.

## İlgili

- [RPC bağdaştırıcıları](/tr/reference/rpc)
- [Typebox](/tr/concepts/typebox)
