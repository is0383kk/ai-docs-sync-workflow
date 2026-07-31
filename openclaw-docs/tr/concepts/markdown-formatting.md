---
read_when:
    - Giden kanallar için Markdown biçimlendirmesini veya parçalama yöntemini değiştiriyorsunuz
    - Yeni bir kanal biçimlendiricisi veya stil eşlemesi ekliyorsunuz
    - Kanallar genelindeki biçimlendirme gerilemelerinde hata ayıklıyorsunuz
summary: Giden kanallar için Markdown biçimlendirme işlem hattı
title: Markdown biçimlendirmesi
x-i18n:
    generated_at: "2026-07-26T22:43:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f9a35fd9a6386068e1e3bec73ec6e692f49239b468f42dd737f919b1c6a88e41
    source_path: concepts/markdown-formatting.md
    workflow: 16
---

OpenClaw, giden Markdown'ı kanala özgü çıktıyı oluşturmadan önce paylaşılan bir ara gösterime
(IR) dönüştürür. IR, düz metnin yanı sıra stil/bağlantı aralıklarını korur; böylece tek bir ayrıştırma adımı
her kanalı besler ve parçalara ayırma, biçimlendirmeyi hiçbir zaman aralığın ortasından
bölmez.

## İşlem hattı

1. **Markdown'ı IR'ye ayrıştırın** (`markdownToIR`) - düz metnin yanı sıra stil aralıkları
   (kalın, italik, üstü çizili, kod, kod bloğu, spoiler, blok alıntı,
   1-6. düzey başlık) ve bağlantı aralıkları. Ofsetler UTF-16 kod birimleridir; böylece Signal stil
   aralıkları doğrudan API'siyle hizalanır. Tablolar yalnızca kanal
   bir tablo modunu etkinleştirdiğinde ayrıştırılır.
2. **IR'yi parçalara ayırın** (`chunkMarkdownIR` / `renderMarkdownIRChunksWithinLimit`)
   - bölme, oluşturmadan önce IR metninde gerçekleşir; böylece satır içi stiller ve
     bağlantılar bir sınırı aşarak bozulmak yerine her parçaya göre dilimlenir.
3. **Kanal başına oluşturun** (`renderMarkdownWithMarkers`) - bir stil işareti eşlemesi
   aralıkları kanalın yerel işaretlemesine dönüştürür.

| Kanal                                                            | Oluşturucu                                                                            | Notlar                                                                                   |
| ---------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- |
| Slack                                                            | mrkdwn belirteçleri (`*bold*`, `_italic_`, `` `code` ``, kod çitleri)               | Bağlantılar `<url\|label>` olur; çift bağlantı oluşturmayı önlemek için ayrıştırma sırasında otomatik bağlantı devre dışıdır |
| Telegram                                                         | HTML etiketleri (`<b>`, `<i>`, `<s>`, `<code>`, `<pre><code>`, `<a href>`, `<tg-spoiler>`) | `richMessages` açıkken zengin ileti tablolarını ve başlıklarını (`<h1>`-`<h6>`) da destekler |
| Signal                                                           | düz metin + `text-style` aralıkları                                                   | Etiket URL'den farklı olduğunda bağlantılar `label (url)` olarak oluşturulur             |
| Discord, WhatsApp, iMessage, Microsoft Teams ve diğer kanallar   | düz metin                                                                             | IR tabanlı biçimlendirme yoktur; Markdown tablo dönüşümü yine `convertMarkdownTables` üzerinden çalışır |

## IR örneği

Girdi Markdown'ı:

```markdown
Merhaba **dünya** - [belgelere](https://docs.openclaw.ai) bakın.
```

IR (şematik):

```json
{
  "text": "Merhaba dünya - belgelere bakın.",
  "styles": [{ "start": 8, "end": 13, "style": "bold" }],
  "links": [{ "start": 16, "end": 25, "href": "https://docs.openclaw.ai" }]
}
```

## Tablo işleme

`markdown.tables`, bir kanalın Markdown tablolarını kanal başına ve
isteğe bağlı olarak hesap başına nasıl dönüştüreceğini denetler:

| Mod       | Davranış                                                                             |
| --------- | ------------------------------------------------------------------------------------ |
| `code`    | Kod bloğu içinde hizalanmış bir ASCII tablosu olarak oluşturur (varsayılan geri dönüş) |
| `bullets` | Her satırı `label: value` madde işaretlerine dönüştürür                              |
| `block`   | Aktarımın desteklediği yerlerde yerel tabloları korur; aksi takdirde `code` moduna geri döner |
| `off`     | Tablo ayrıştırmayı devre dışı bırakır; ham tablo metni değiştirilmeden aktarılır      |

Kanal başına Plugin varsayılanları: Signal, WhatsApp ve Matrix için varsayılan
`bullets`; Mattermost için `off`; Telegram için `block` değeridir (hesapta
`richMessages` etkin değilse `code` olarak çözümlenir). Açık bir Plugin varsayılanı
bulunmayan tüm kanallar `code` değerine geri döner.

```yaml
channels:
  discord:
    markdown:
      tables: code
    accounts:
      work:
        markdown:
          tables: off
```

## Parçalama kuralları

- Parça sınırları kanal bağdaştırıcılarından/yapılandırmasından gelir ve
  oluşturulan çıktıya değil IR metnine uygulanır.
- Çitli kod blokları, kanalların kapanış çitini doğru oluşturabilmesi için
  sonunda yeni satır bulunan tek bir blok olarak korunur.
- Liste ve blok alıntı önekleri IR metninin parçasıdır; bu nedenle parçalama hiçbir zaman
  önekin ortasından bölmez.
- Satır içi stiller hiçbir zaman parçalar arasında bölünmez; oluşturucu, açık bir
  stili sonraki parçanın başında yeniden açar.

Kanallardaki parça sınırı ve teslim davranışı için [Akış ve parçalama](/concepts/streaming)
sayfasına bakın.

## Bağlantı politikası

- **Slack:** `[label](url)` -> `<url|label>`; çıplak URL'ler çıplak kalır.
- **Telegram:** `[label](url)` -> `<a href="url">label</a>` (HTML ayrıştırma modu).
- **Signal:** Etiket zaten URL ile eşleşmiyorsa `[label](url)` -> `label (url)`.

## Spoiler'lar

Spoiler işaretleri (`||spoiler||`) Signal için ayrıştırılır (`SPOILER`
stil aralıklarına eşlenir) ve Telegram için ayrıştırılır (`<tg-spoiler>` olarak eşlenir). Diğer kanallar
`||...||` değerini düz metin olarak işler.

## Kanal biçimlendiricisi ekleme veya güncelleme

1. Kanala uygun seçenekleri (`autolink`, `headingStyle`, `blockquotePrefix`, `tableMode`)
   ileterek `markdownToIR(...)` ile **bir kez ayrıştırın**.
2. Bir stil işareti eşlemesiyle (veya Signal gibi aktarımlar için
   özel stil aralığı mantığıyla) `renderMarkdownWithMarkers(...)` kullanarak **oluşturun**.
3. Her parçayı oluşturmadan önce `chunkMarkdownIR(...)` veya
   `renderMarkdownIRChunksWithinLimit(...)` ile **parçalara ayırın**.
4. Giden gönderim yolundan yeni parçalayıcıyı ve oluşturucuyu çağırması için
   **bağdaştırıcıyı bağlayın**.
5. Biçim testleriyle ve kanal parçalara ayırıyorsa bir giden teslim testiyle
   **test edin**.

## Yaygın sorunlar

- Slack açılı ayraç belirteçleri (`<@U123>`, `<#C123>`, `<https://...>`)
  kaçış işleminden sağ çıkmalıdır; ham HTML'nin yine de güvenli biçimde kaçışlanması gerekir.
- Telegram HTML'sinde bozuk işaretlemeyi önlemek için etiketlerin dışındaki metnin kaçışlanması gerekir.
- Signal stil aralıkları kod noktası ofsetlerini değil, UTF-16 ofsetlerini kullanır.
- Kapanış işaretinin kendi satırına yerleşmesi için çitli kod bloklarının
  sonundaki yeni satırları koruyun.

## İlgili

<CardGroup cols={2}>
  <Card title="Akış ve parçalama" href="/tr/concepts/streaming" icon="bars-staggered">
    Giden akış davranışı, parça sınırları ve kanala özgü teslim.
  </Card>
  <Card title="Sistem istemi" href="/tr/concepts/system-prompt" icon="message-lines">
    Eklenen çalışma alanı dosyaları dahil olmak üzere modelin konuşmadan önce gördükleri.
  </Card>
</CardGroup>
