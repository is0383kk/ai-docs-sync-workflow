---
read_when:
    - Belleğinizin arka ucu olarak QMD'yi ayarlamak istiyorsunuz
    - Yeniden sıralama veya ek dizine alınmış yollar gibi gelişmiş bellek özellikleri istiyorsunuz
summary: BM25, vektörler, yeniden sıralama ve sorgu genişletme özelliklerine sahip yerel öncelikli arama yardımcı servisi
title: QMD bellek motoru
x-i18n:
    generated_at: "2026-07-26T23:18:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0e54dc9a18d834036e4c79d6b7bdecb268a29976d9f30ea6e82a56ca5d71fda
    source_path: concepts/memory-qmd.md
    workflow: 16
---

[QMD](https://github.com/tobi/qmd), OpenClaw ile birlikte çalışan, önce yerel yaklaşımını benimseyen bir arama yardımcı hizmetidir. BM25, vektör araması ve yeniden sıralamayı tek bir ikili dosyada birleştirir ve çalışma alanı bellek dosyalarınızın dışındaki içerikleri de indeksleyebilir.

## Yerleşik motora kıyasla sundukları

- Daha iyi geri çağırma için **yeniden sıralama ve sorgu genişletme**.
- **Ek dizinleri indeksleme** - proje belgeleri, ekip notları, diskteki her şey.
- **Oturum dökümlerini indeksleme** - önceki konuşmaları hatırlama.
- **Tamamen yerel** - resmi llama.cpp sağlayıcı Plugin'iyle çalışır ve
  GGUF modellerini otomatik olarak indirir.
- **Otomatik geri dönüş** - QMD kullanılamıyorsa OpenClaw sorunsuz biçimde
  yerleşik motora geri döner.

## Başlarken

### Ön koşullar

- QMD'yi yükleyin: `npm install -g @tobilu/qmd` veya `bun install -g @tobilu/qmd`
- Uzantılara izin veren SQLite derlemesi (macOS'te `brew install sqlite`).
- QMD, gateway'in `PATH` konumunda bulunmalıdır.
- macOS ve Linux doğrudan çalışır. Windows en iyi WSL2 aracılığıyla desteklenir.

### Etkinleştirme

```json5
{
  memory: {
    backend: "qmd",
  },
}
```

OpenClaw, `~/.openclaw/agents/<agentId>/qmd/` altında bağımsız bir QMD ana dizini oluşturur
ve yardımcı hizmetin yaşam döngüsünü otomatik olarak yönetir;
koleksiyonlar, güncellemeler ve gömme çalıştırmaları sizin için gerçekleştirilir.
Güncel QMD koleksiyon ve MCP sorgu biçimlerini tercih eder ancak gerektiğinde
alternatif koleksiyon kalıbı bayraklarına ve eski MCP araç adlarına geri döner.
Başlangıç uzlaştırması ayrıca aynı adı taşıyan eski bir QMD koleksiyonu hâlâ
mevcutsa eskimiş yönetilen koleksiyonları standart kalıplarıyla yeniden oluşturur.

## Yardımcı hizmetin çalışma biçimi

- OpenClaw, çalışma alanı bellek dosyalarından ve yapılandırılmış
  `memory.qmd.paths` öğelerinden koleksiyonlar oluşturur. QMD bağdaştırıcısı güncelleme,
  gömme, gecikmeli birleştirme ve zaman aşımı sezgisellerini yönetir; bunlar kullanıcı
  yapılandırması değildir.
- QMD, ajan başına QMD ana dizini altındaki `index.sqlite`, YAML koleksiyon
  yapılandırması ve model indirmelerini yönetmeye devam eder; bunlar OpenClaw durum
  tabloları değil, harici araç yapıtlarıdır. OpenClaw'a ait eş güdüm yalnızca SQLite'ta
  bulunur: paylaşılan tek bir kira, ajanlar arasındaki gömme çalışmalarını sınırlarken
  her ajan veritabanındaki bir kira, o ajanın koleksiyon, güncelleme ve gömme yazma
  işlemlerini sıralı hâle getirir. Çalışma zamanı artık QMD dosya kilidi yardımcı
  dosyaları oluşturmaz. `openclaw doctor --fix`, kullanımdan kaldırılmış yardımcı dosyaları
  yalnızca eski süreç sahiplerinin geçersiz olduğunu kanıtladıktan sonra kaldırır.
  Yükseltmeler doğrudan geçişle yapılır: yeni sürümü kullanmadan önce durum dizinini
  paylaşan tüm OpenClaw süreçlerini durdurup yeniden başlatın. Eski ve yeni QMD
  yazıcılarının birlikte kullanılması desteklenmez; çalışma zamanı, kullanımdan
  kaldırılan yardımcı dosyaları kasıtlı olarak çift kilitlemez.
- Varsayılan çalışma alanı koleksiyonu, `MEMORY.md` ile `memory/`
  ağacını izler. Küçük harfli `memory.md`, kök bellek dosyası olarak indekslenmez.
- QMD'nin kendi tarayıcısı gizli yolları ve `.git`,
  `.cache`, `node_modules`, `vendor`, `dist` ve
  `build` gibi yaygın bağımlılık/derleme dizinlerini yok sayar.
  Gateway başlangıcı QMD'yi tembel tutar; yönetici, bellek ilk kez kullanıldığında başlatılır.
- Aramalar, yapılandırılmış `searchMode` değerini kullanır (varsayılan:
  `search`; `vsearch` ve `query` da desteklenir).
  `search` yalnızca BM25 kullandığından OpenClaw bu modda anlamsal vektör
  hazırlık yoklamalarını ve gömme bakımını atlar. Bir mod başarısız olursa OpenClaw
  `qmd query` ile yeniden dener.
- `searchMode` değeri `query` olduğunda, yeniden sıralayıcı
  olmadan QMD'nin hibrit sorgu yolunu kullanmak için `memory.qmd.rerank` değerini
  `false` olarak ayarlayın (QMD 2.1 veya daha yenisini gerektirir).
  OpenClaw, doğrudan QMD CLI yoluna `--no-rerank`, QMD'nin MCP sorgu aracına ise
  `rerank: false` iletir.
- Çoklu koleksiyon filtrelerini desteklediğini bildiren QMD sürümlerinde OpenClaw,
  aynı kaynaktan gelen koleksiyonları tek bir QMD arama çağrısında gruplandırır. Eski QMD
  sürümleri, koleksiyon başına uyumlu geri dönüş yolunu kullanmaya devam eder.
- QMD tamamen başarısız olursa OpenClaw yerleşik SQLite motoruna geri döner.
  Bir açma hatasından sonra tekrarlanan sohbet sırası denemeleri kısa süreliğine
  yavaşlatılır; böylece eksik bir ikili dosya veya bozuk yardımcı hizmet bağımlılığı
  yeniden deneme fırtınası oluşturmaz. `openclaw memory status` ve tek seferlik CLI
  yoklamaları yine de QMD'yi doğrudan yeniden denetler.

<Info>
İlk arama yavaş olabilir; QMD, ilk `qmd query` çalıştırmasında yeniden sıralama
ve sorgu genişletme için GGUF modellerini (~2 GB) otomatik olarak indirir.
</Info>

## Arama performansı ve uyumluluk

OpenClaw, QMD arama yolunu hem güncel hem de eski QMD kurulumlarıyla uyumlu tutar.

Başlangıçta OpenClaw, kurulu QMD yardım metnini yönetici başına bir kez denetler.
İkili dosya birden fazla koleksiyon filtresini desteklediğini bildiriyorsa OpenClaw,
aynı kaynaktan gelen tüm koleksiyonları tek komutla arar:

```bash
qmd search "router notes" --json -n 10 -c memory-root-main -c memory-dir-main
```

Bu, her kalıcı bellek koleksiyonu için ayrı bir QMD alt süreci başlatılmasını önler.
Oturum dökümü koleksiyonları kendi kaynak gruplarında kalır; böylece karma
`memory` + `sessions` aramaları sonuç çeşitlendiricisine her iki
kaynaktan da girdi sağlamaya devam eder.

Eski QMD derlemeleri yalnızca bir koleksiyon filtresini kabul eder. OpenClaw bu
derlemelerden birini algıladığında uyumluluk yolunu korur ve sonuçları birleştirip
yinelenenleri kaldırmadan önce her koleksiyonu ayrı ayrı arar.

Kurulu sözleşmeyi elle incelemek için şunu çalıştırın:

```bash
qmd --help | grep -i collection
```

Güncel QMD yardım metni, bir veya daha fazla koleksiyonun hedeflenmesinden söz eder.
Eski yardım metni genellikle tek bir koleksiyonu açıklar.

## Model geçersiz kılmaları

QMD model ortam değişkenleri gateway sürecinden değiştirilmeden aktarılır; böylece
yeni OpenClaw yapılandırması eklemeden QMD'yi genel olarak ayarlayabilirsiniz:

```bash
export QMD_EMBED_MODEL="hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf"
export QMD_RERANK_MODEL="/absolute/path/to/reranker.gguf"
export QMD_GENERATE_MODEL="/absolute/path/to/generator.gguf"
```

Gömme modelini değiştirdikten sonra indeksin yeni vektör uzayıyla eşleşmesi için
gömmeleri yeniden çalıştırın.

## Ek yolları indeksleme

Ek dizinleri aranabilir hâle getirmek için QMD'yi bu dizinlere yönlendirin:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      paths: [{ name: "docs", path: "~/notes", pattern: "**/*.md" }],
    },
  },
}
```

Ek yollardan gelen parçacıklar arama sonuçlarında `qmd/<collection>/<relative-path>` olarak görünür.
`memory_get` bu ön eki tanır ve doğru koleksiyon kökünden okur.

## Oturum dökümlerini indeksleme

Önceki konuşmaları hatırlamak için oturum indekslemeyi etkinleştirin. QMD hem genel
`memory.search` oturum kaynağına hem de QMD döküm dışa aktarıcısına ihtiyaç duyar:

```json5
{
  memory: {
    backend: "qmd",
    search: {
      experimental: { sessionMemory: true },
      sources: ["memory", "sessions"],
    },
    qmd: {
      sessions: { enabled: true },
    },
  },
}
```

Dökümler, `~/.openclaw/agents/<id>/qmd/sessions/` altındaki özel bir QMD koleksiyonuna arındırılmış
Kullanıcı/Asistan sıraları olarak dışa aktarılır. Yalnızca `sources: ["sessions"]`
değerinin ayarlanması dökümleri QMD'ye aktarmaz; ayrıca `rememberAcrossConversations` veya açık
QMD oturum dışa aktarımını etkinleştirin.

Oturum eşleşmeleri yine
[`tools.sessions.visibility`](/tr/gateway/config-tools#toolssessions) tarafından filtrelenir.
Varsayılan `tree` görünürlüğü mevcut oturumu, bu oturumun başlattığı
oturumları ve ortam grup farkındalığı aracılığıyla izlenen aynı ajan grup oturumlarını
içerir. `session.dmScope: "main"` kullanıldığında, çok kullanıcılı bir DM kurulumundaki
kullanıcılar ana oturumu paylaşır ve bu oturumun izlediği gruplardaki içeriği
hatırlayabilir. DM yalıtımı için eş başına `dmScope` kullanın veya ortamdaki
izlenen oturum okumalarını devre dışı bırakmak için görünürlüğü `"self"`
olarak ayarlayın. İlişkisiz diğer aynı ajan oturumları için yine `"agent"`
görünürlüğü gerekir.

## Arama kapsamı

Varsayılan olarak QMD arama sonuçları yalnızca doğrudan oturumlarda gösterilir
(grup veya kanal sohbetlerinde değil). Bunu değiştirmek için `memory.qmd.scope`
değerini yapılandırın:

```json5
{
  memory: {
    qmd: {
      scope: {
        default: "deny",
        rules: [{ action: "allow", match: { chatType: "direct" } }],
      },
    },
  },
}
```

Yukarıdaki parçacık gerçek varsayılan kuraldır. Kapsam bir aramayı reddettiğinde,
boş sonuçların hata ayıklamasını kolaylaştırmak için OpenClaw türetilen kanal ve
sohbet türüyle birlikte bir uyarı kaydeder.

## Atıflar

`memory.citations` değeri `auto` veya `on` olduğunda,
arama parçacıklarına bir `Source: <path>#L<line>` (veya `#L<start>-L<end>`) alt bilgisi
eklenir. `auto` modunda alt bilgi yalnızca doğrudan sohbet oturumları
için eklenir. Yolu ajana dâhilî olarak aktarmaya devam ederken alt bilgiyi kaldırmak
için `memory.citations = "off"` olarak ayarlayın.

## Ne zaman kullanılmalı

Şunlara ihtiyaç duyduğunuzda QMD'yi seçin:

- Daha yüksek kaliteli sonuçlar için yeniden sıralama.
- Çalışma alanı dışındaki proje belgelerini veya notları arama.
- Geçmiş oturum konuşmalarını hatırlama.
- API anahtarları olmadan tamamen yerel arama.

Daha basit kurulumlarda [yerleşik motor](/tr/concepts/memory-builtin), ek bağımlılık
gerektirmeden iyi çalışır.

## Sorun giderme

**QMD bulunamadı mı?** İkili dosyanın gateway'in `PATH` konumunda
olduğundan emin olun. OpenClaw bir hizmet olarak çalışıyorsa sembolik bağlantı
oluşturun: `sudo ln -s ~/.bun/bin/qmd /usr/local/bin/qmd`.

`qmd --version` kabuğunuzda çalışıyor ancak OpenClaw yine de
`spawn qmd ENOENT` bildiriyorsa gateway sürecinin `PATH` değeri büyük
olasılıkla etkileşimli kabuğunuzdakinden farklıdır. İkili dosyayı açıkça sabitleyin:

```json5
{
  memory: {
    backend: "qmd",
    qmd: {
      command: "/absolute/path/to/qmd",
    },
  },
}
```

QMD'nin kurulu olduğu ortamda `command -v qmd` kullanın, ardından
`openclaw memory status --deep` ile yeniden denetleyin.

**İlk arama çok mu yavaş?** QMD, ilk kullanımda GGUF modellerini indirir. OpenClaw'ın
kullandığı XDG dizinleriyle `qmd query "test"` kullanarak önceden ısıtın.

**Arama sırasında çok sayıda QMD alt süreci mi oluşuyor?** Mümkünse QMD'yi güncelleyin.
OpenClaw, aynı kaynaktan gelen çoklu koleksiyon aramalarında yalnızca kurulu QMD
birden fazla `-c` filtresini desteklediğini bildiriyorsa tek süreç
kullanır; aksi takdirde doğruluk için eski koleksiyon başına geri dönüş yolunu korur.

**Yalnızca BM25 kullanan QMD yine de llama.cpp oluşturmaya mı çalışıyor?**
`memory.qmd.searchMode = "search"` olarak ayarlayın. OpenClaw bu modu yalnızca sözcüksel olarak
değerlendirir, QMD vektör durum yoklamalarını ve gömme bakımını atlar ve anlamsal
hazırlık denetimlerini `vsearch` veya `query` kurulumlarına
bırakır.

**Arama zaman aşımına mı uğruyor?** `memory.qmd.limits.timeoutMs` değerini artırın (varsayılan:
4000ms). Daha yavaş donanımlar için daha yüksek bir değere, örneğin
`120000`, ayarlayın. Bu sınır, ajan `memory_search` çağrıları sırasında
QMD'nin kendi arama komutlarına uygulanır; kurulum, eşitleme, yerleşik geri dönüş ve
tamamlayıcı derlem çalışmaları kendi daha kısa süre sınırlarını korur.

**Grup veya kanal sohbetlerinde sonuçlar boş mu?** Bu, yalnızca doğrudan oturumlara
izin veren varsayılan `memory.qmd.scope` ile beklenen bir durumdur. QMD sonuçlarını
orada da istiyorsanız `group` veya `channel` sohbet türleri için
bir `allow` kuralı ekleyin.

**Kök bellek araması aniden çok mu genişledi?** Gateway'i yeniden başlatın veya bir
sonraki başlangıç uzlaştırmasını bekleyin. OpenClaw aynı ad çakışması algıladığında
eskimiş yönetilen koleksiyonları standart `MEMORY.md` ve
`memory/` kalıplarıyla yeniden oluşturur.

**Çalışma alanında görünen geçici depolar `ENAMETOOLONG` veya bozuk indekslemeye
mi neden oluyor?** QMD geçişi, OpenClaw'ın yerleşik sembolik bağlantı kuralları yerine
temeldeki QMD tarayıcısını izler. QMD döngü güvenli geçiş veya açık dışlama denetimleri
sunana kadar geçici monorepo kullanıma alma kopyalarını `.tmp/` gibi gizli
dizinlerin altında veya indekslenen QMD köklerinin dışında tutun.

## Yapılandırma

Tam yapılandırma yüzeyi (`memory.qmd.*`), arama modları, güncelleme aralıkları,
kapsam kuralları ve diğer tüm ayarlar için
[Bellek yapılandırması referansına](/tr/reference/memory-config) bakın.

## İlgili

- [Belleğe genel bakış](/tr/concepts/memory)
- [Yerleşik bellek motoru](/tr/concepts/memory-builtin)
- [Honcho belleği](/tr/concepts/memory-honcho)
