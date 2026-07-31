---
read_when:
    - Temsilcilerden gelen PDF'leri analiz etmek istiyorsunuz
    - Tam PDF aracı parametrelerine ve sınırlarına ihtiyacınız var
    - Yerel PDF modu ile ayıklama yedek mekanizması arasındaki farkta hata ayıklıyorsunuz
summary: Yerel sağlayıcı desteği ve ayıklama yedeğiyle bir veya daha fazla PDF belgesini analiz edin
title: PDF aracı
x-i18n:
    generated_at: "2026-07-27T00:21:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0e5b897e1e122af4b2f6f9a3eaeb73f6e93af1051d306ad82539b258de90c49
    source_path: tools/pdf.md
    workflow: 16
---

`pdf` bir veya daha fazla PDF belgesini analiz eder ve metin döndürür. Anthropic ve Google modellerinde yerel belge girdisini kullanır; diğer tüm sağlayıcılarda ise metin/görüntü ayıklamaya geri döner.

## Kullanılabilirlik

Araç yalnızca OpenClaw, agent için PDF destekli bir model çözümleyebildiğinde kaydedilir. Çözümleme sırası:

1. `agents.defaults.pdfModel` (açıkça belirtilen birincil/yedekler)
2. `agents.defaults.imageModel` (açıkça belirtilen birincil/yedekler)
3. Sağlayıcısı yerel PDF girdisini (Anthropic, Google) destekliyorsa veya zaten yapılandırılmış bir görüntü modeli varsa agent'ın çözümlenen oturum/varsayılan modeli
4. Önce yerel PDF sağlayıcıları tercih edilerek, kullanılabilir kimlik doğrulamasına sahip otomatik algılanmış görüntü/görsel destekli sağlayıcılar

Her yedek adayın kimlik doğrulaması kullanımdan önce denetlenir; bu nedenle yapılandırılmış bir `provider/model`, yalnızca OpenClaw agent için söz konusu sağlayıcıda kimlik doğrulaması yapabiliyorsa dikkate alınır. Kullanılabilir bir model çözümlenemezse `pdf` aracı sunulmaz.

## Girdi referansı

<ParamField path="pdf" type="string">
Bir PDF yolu veya URL'si.
</ParamField>

<ParamField path="pdfs" type="string[]">
Toplam en fazla 10 olmak üzere birden fazla PDF yolu veya URL'si.
</ParamField>

<ParamField path="prompt" type="string" default="Analyze this PDF document.">
Analiz istemi.
</ParamField>

<ParamField path="pages" type="string">
`1-5` veya `1,3,7-9` gibi sayfa filtresi. Yerel sağlayıcı modunda desteklenmez.
</ParamField>

<ParamField path="password" type="string">
Şifrelenmiş PDF'ler için parola. İstekteki her PDF'ye uygulanır; yalnızca ayıklama yedek modunda kullanılır.
</ParamField>

<ParamField path="model" type="string">
`provider/model` biçiminde isteğe bağlı model geçersiz kılması.
</ParamField>

<ParamField path="maxBytesMb" type="number">
PDF başına MB cinsinden boyut sınırı. Varsayılan olarak `agents.defaults.pdfMaxMb`; ayarlanmamışsa `10` kullanılır.
</ParamField>

Notlar:

- `pdf` ve `pdfs` yüklemeden önce birleştirilir ve yinelenenler kaldırılır; en az biri gereklidir.
- `pages`, 1 tabanlı sayfa numaraları olarak ayrıştırılır, yinelenenler kaldırılır, sıralanır ve `agents.defaults.pdfMaxPages` (varsayılan `20`) ile sınırlandırılır. Sınırlar içindeki hiçbir sayfayla eşleşmeyen bir aralık, model çağrısından önce hataya neden olur.

## Desteklenen PDF referansları

- Yerel dosya yolu (`~` genişletmesi dâhil)
- `file://` URL'si
- `http://` ve `https://` URL'si
- `media://inbound/<id>` gibi OpenClaw tarafından yönetilen gelen referanslar

Diğer URI şemaları (örneğin `ftp://`) `details.error = "unsupported_pdf_reference"` döndürür. Araç korumalı alanda çalışırken uzak `http(s)` URL'leri reddedilir. Yalnızca çalışma alanına izin veren dosya politikası etkinleştirildiğinde, izin verilen köklerin dışındaki yerel yollar reddedilir; OpenClaw'ın gelen medya deposundaki yönetilen gelen referanslara ve yeniden oynatılan yollara yine izin verilir.

## Yürütme modları

### Yerel sağlayıcı modu

`anthropic` ve `google` sağlayıcıları için kullanılır (şu anda yerel PDF belgesi desteği bildiren tek sağlayıcılar). Ham PDF baytları, dosya başına yerel belge/satır içi PDF parçası olarak doğrudan sağlayıcı API'sine gönderilir.

Sınırlar:

- `pages` desteklenmez; ayarlanırsa araç `pages is not supported with native PDF providers` hatasını fırlatır.
- `password` desteklenmez; ayarlanırsa araç `password is not supported with native PDF providers` hatasını fırlatır. Şifrelenmiş PDF'ler için yerel olmayan bir model kullanın.

### Ayıklama yedek modu

Diğer tüm sağlayıcılar için kullanılır.

1. Metin ve görüntü ayıklamak için `clawpdf` paketini (PDFium WebAssembly) kullanan paketlenmiş `document-extract` Plugin'i aracılığıyla seçili sayfalardan (`agents.defaults.pdfMaxPages` değerine kadar, varsayılan `20`) metin ayıklayın.
2. Ayıklanan metin `200` karakterden kısaysa aynı sayfaları PNG görüntüleri olarak işleyin. İşleme bütçesi toplam `4,000,000` pikseldir ve görüntü gerektiren tüm sayfalar arasında paylaşılır (sayfa başına değil, kalan sayfa başına orantılı olarak tahsis edilir); bu nedenle zaten yeterli metin içeren metin sayfalarının işlenmesi tamamen atlanır.
3. Ayıklanan metni (ve işlenen tüm görüntüleri) istemle birlikte seçili modele gönderin.

Ayrıntılar:

- Şifrelenmiş PDF'ler üst düzey `password` parametresiyle açılır.
- Model görüntü girdisini desteklemiyorsa ve ayıklanabilir metin yoksa araç hata verir.
- Görüntü işleme başarısız olursa OpenClaw görüntüleri çıkarır ve ayıklanan metinle devam eder.
- Hedef model yalnızca metin destekliyorsa ve ayıklama görüntü ürettiyse OpenClaw görüntüleri çıkarır ve yalnızca metni gönderir.

## Yapılandırma

```json5
{
  agents: {
    defaults: {
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      pdfMaxBytesMb: 10,
      pdfMaxPages: 20,
    },
  },
}
```

| Anahtar                       | Varsayılan | Anlam                                                                                     |
| ----------------------------- | ---------- | ----------------------------------------------------------------------------------------- |
| `agents.defaults.pdfModel`    | ayarlanmamış | Açıkça belirtilen birincil/yedek PDF modelleri; `imageModel`, ardından oturum modeline geri döner. |
| `agents.defaults.pdfMaxMb`    | `10`    | PDF başına MB cinsinden boyut sınırı.                                                      |
| `agents.defaults.pdfMaxPages` | `20`    | PDF başına işlenen azami sayfa sayısı.                                                     |

Alanların tüm ayrıntıları için [Yapılandırma Referansı](/tr/gateway/config-agents#agent-defaults) bölümüne bakın.

## Çıktı ayrıntıları

Araç, `content[0].text` içinde metin ve `details` içinde yapılandırılmış meta veriler döndürür.

Yaygın `details` alanları:

- `model`: çözümlenen model referansı (`provider/model`)
- `native`: yerel sağlayıcı modu için `true`, yedek mod için `false`
- `attempts`: başarıdan önce başarısız olan yedek denemeleri

Yol alanları:

- Tek PDF girdisi: `details.pdf`
- Birden fazla PDF girdisi: `pdf` girdileriyle `details.pdfs[]`
- Korumalı alan yol yeniden yazma meta verileri (uygun olduğunda): `rewrittenFrom`

## Hata davranışı

| Koşul                             | Sonuç                                                          |
| --------------------------------- | -------------------------------------------------------------- |
| PDF girdisi yok                   | `pdf required: provide a path or URL to a PDF document` hatasını fırlatır |
| 10'dan fazla PDF                  | `details.error = "too_many_pdfs"`                              |
| Desteklenmeyen referans şeması    | `details.error = "unsupported_pdf_reference"`                  |
| Yerel sağlayıcıyla `pages`    | `pages is not supported with native PDF providers` hatasını fırlatır      |
| Yerel sağlayıcıyla `password` | `password is not supported with native PDF providers` hatasını fırlatır   |

## Örnekler

Tek PDF:

```json
{
  "pdf": "/tmp/report.pdf",
  "prompt": "Bu raporu 5 maddeyle özetleyin"
}
```

Birden fazla PDF:

```json
{
  "pdfs": ["/tmp/q1.pdf", "/tmp/q2.pdf"],
  "prompt": "Her iki belgedeki riskleri ve zaman çizelgesi değişikliklerini karşılaştırın"
}
```

Sayfa filtreli yedek model:

```json
{
  "pdf": "https://example.com/report.pdf",
  "pages": "1-3,7",
  "model": "openai/gpt-5.4-mini",
  "prompt": "Yalnızca müşterileri etkileyen olayları ayıklayın"
}
```

Ayıklama yedeğiyle şifrelenmiş PDF:

```json
{
  "pdf": "/tmp/locked.pdf",
  "password": "example-password",
  "model": "openai/gpt-5.4-mini",
  "prompt": "Bu sözleşmeyi özetleyin"
}
```

## İlgili

- [Araçlara Genel Bakış](/tr/tools) - kullanılabilir tüm agent araçları
- [Yapılandırma Referansı](/tr/gateway/config-agents#agent-defaults) - pdfMaxBytesMb ve pdfMaxPages yapılandırması
