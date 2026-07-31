---
read_when:
    - Ajanların kod veya Markdown düzenlemelerini diff olarak göstermesini istiyorsunuz
    - Canvas'a hazır bir görüntüleyici URL'si veya işlenmiş bir diff dosyası istiyorsunuz
    - Güvenli varsayılanlara sahip, denetimli ve geçici diff yapıtlarına ihtiyacınız var
sidebarTitle: Diffs
summary: Ajanlar için salt okunur fark görüntüleyici ve dosya işleyici (isteğe bağlı Plugin aracı)
title: Farklar
x-i18n:
    generated_at: "2026-07-26T23:38:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: baeb5dd1277120e57178f092e3ae1616edd3389a54721c929d8711301535d302
    source_path: tools/diffs.md
    workflow: 16
---

`diffs`, önce/sonra metnini veya birleşik bir yamayı salt okunur bir fark yapıtına dönüştüren, isteğe bağlı bir paketlenmiş Plugin aracıdır. Ayrıca sistem isteminin başına kısa aracı yönergeleri ekler ve daha kapsamlı talimatlar için tamamlayıcı bir Skills paketiyle birlikte gelir.

Girdi: `before` + `after` metni veya birleşik bir `patch` (birbirini dışlar).

Çıktı: tuval sunumu için bir Gateway görüntüleyici URL'si, ileti teslimi için işlenmiş bir PNG/PDF dosya yolu veya her ikisi.

## Hızlı başlangıç

<Steps>
  <Step title="Plugin'i yükleyin">
    ```bash
    openclaw plugins install diffs
    ```
  </Step>
  <Step title="Plugin'i etkinleştirin">
    ```json5
    {
      plugins: {
        entries: {
          diffs: {
            enabled: true,
          },
        },
      },
    }
    ```
  </Step>
  <Step title="Bir mod seçin">
    <Tabs>
      <Tab title="view">
        Önceliği tuvale veren akışlar: aracılar `diffs` öğesini `mode: "view"` ile çağırır ve `details.viewerUrl` öğesini `canvas present` ile açar.
      </Tab>
      <Tab title="file">
        Sohbet dosyası teslimi: aracılar `diffs` öğesini `mode: "file"` ile çağırır ve `details.filePath` öğesini `message` ile, `path` veya `filePath` kullanarak gönderir.
      </Tab>
      <Tab title="both">
        Birleşik (varsayılan): aracılar her iki yapıtı tek çağrıda almak için `diffs` öğesini `mode: "both"` ile çağırır.
      </Tab>
    </Tabs>
  </Step>
</Steps>

## Yerleşik sistem yönergelerini devre dışı bırakma

Aracı koruyup sistem isteminin başına eklenen yönergeleri kaldırmak için `plugins.entries.diffs.hooks.allowPromptInjection` değerini `false` olarak ayarlayın:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
      },
    },
  },
}
```

Bu, araç ve Skills kullanılabilir durumda kalırken Plugin'in `before_prompt_build` kancasını engeller. Hem yönergeleri hem de aracı devre dışı bırakmak için bunun yerine Plugin'i devre dışı bırakın.

## Araç girdisi başvurusu

Belirtilmedikçe tüm alanlar isteğe bağlıdır.

<ParamField path="before" type="string">
  Özgün metin. `patch` atlandığında `after` ile birlikte gereklidir.
</ParamField>
<ParamField path="after" type="string">
  Güncellenmiş metin. `patch` atlandığında `before` ile birlikte gereklidir.
</ParamField>
<ParamField path="patch" type="string">
  Birleşik fark metni. `before` ve `after` ile birbirini dışlar.
</ParamField>
<ParamField path="path" type="string">
  Önce/sonra modu için görüntülenecek dosya adı.
</ParamField>
<ParamField path="lang" type="string">
  Önce/sonra modu için dil geçersiz kılma ipucu. Bilinmeyen değerler ve varsayılan görüntüleyici kümesinin dışındaki diller, Diff Viewer Language Pack Plugin'i yüklü olmadığı sürece düz metne geri döner.
</ParamField>
<ParamField path="title" type="string">
  Görüntüleyici başlığını geçersiz kılar.
</ParamField>
<ParamField path="mode" type='"view" | "file" | "both"'>
  Çıktı modu. Varsayılan olarak Plugin'in `defaults.mode` değerini (`both`) kullanır. Kullanımdan kaldırılmış diğer ad: `"image"`, `"file"` ile aynı şekilde davranır.
</ParamField>
<ParamField path="theme" type='"light" | "dark"'>
  Görüntüleyici teması. Varsayılan olarak Plugin'in `defaults.theme` değerini kullanır.
</ParamField>
<ParamField path="layout" type='"unified" | "split"'>
  Fark düzeni. Varsayılan olarak Plugin'in `defaults.layout` değerini kullanır.
</ParamField>
<ParamField path="expandUnchanged" type="boolean">
  Tam bağlam kullanılabilir olduğunda değişmemiş bölümleri genişletir. Yalnızca çağrı başına seçenektir (Plugin varsayılan anahtarı değildir).
</ParamField>
<ParamField path="fileFormat" type='"png" | "pdf"'>
  İşlenmiş dosya biçimi. Varsayılan olarak Plugin'in `defaults.fileFormat` değerini kullanır.
</ParamField>
<ParamField path="fileQuality" type='"standard" | "hq" | "print"'>
  PNG/PDF işleme için kalite önayarı.
</ParamField>
<ParamField path="fileScale" type="number">
  Cihaz ölçeğini geçersiz kılar (`1`-`4`).
</ParamField>
<ParamField path="fileMaxWidth" type="number">
  CSS pikseli cinsinden en fazla işleme genişliği (`640`-`2400`).
</ParamField>
<ParamField path="ttlSeconds" type="number" default="1800">
  Görüntüleyici ve bağımsız dosya çıktıları için saniye cinsinden yapıt TTL'si. En fazla `21600`.
</ParamField>
<ParamField path="baseUrl" type="string">
  Görüntüleyici URL kaynağını geçersiz kılar. Plugin'in `viewerBaseUrl` değerini geçersiz kılar. `http` veya `https` olmalıdır; sorgu/karması içeremez.
</ParamField>

<AccordionGroup>
  <Accordion title="Doğrulama ve sınırlar">
    - `before`/`after`: her biri en fazla 512 KiB.
    - `patch`: en fazla 2 MiB.
    - `path`: en fazla 2048 bayt.
    - `lang`: en fazla 128 bayt.
    - `title`: en fazla 1024 bayt.
    - Yama karmaşıklığı sınırı: en fazla 128 dosya ve toplam 120000 satır.
    - `patch` öğesinin `before`/`after` ile birlikte kullanılması reddedilir.
    - İşlenmiş dosya güvenlik sınırları (PNG ve PDF):
      - `fileQuality: "standard"`: en fazla 8 MP (8,000,000 işlenmiş piksel).
      - `fileQuality: "hq"`: en fazla 14 MP.
      - `fileQuality: "print"`: en fazla 24 MP.
      - PDF ayrıca 50 sayfayla sınırlandırılır.

  </Accordion>
</AccordionGroup>

## Sözdizimi vurgulama

Yerleşik diller:

`javascript`, `typescript`, `tsx`, `jsx`, `json`, `markdown`, `yaml`, `css`, `html`, `sh`, `python`, `go`, `rust`, `java`, `c`, `cpp`, `csharp`, `php`, `sql`, `docker`, `ruby`, `swift`, `kotlin`, `r`, `dart`, `lua`, `powershell`, `xml` ve `toml`.

Yaygın diğer adlar (`js`, `ts`, `bash`, `md`, `yml`, `c++`, `dockerfile`, `rb`, `kt`, `ps1` vb.) bu dillere normalleştirilir.

Daha fazla dil (Astro, Vue, Svelte, MDX, GraphQL, Terraform/HCL, Nix, Clojure, Elixir, Haskell, OCaml, Scala, Zig, Solidity, Verilog/VHDL, Fortran, MATLAB, LaTeX, Mermaid, Sass/Less/SCSS, Nginx, Apache, CSV, dotenv, INI, diff ve diğerleri) için Diff Viewer Language Pack Plugin'ini yükleyin:

```bash
openclaw plugins install clawhub:@openclaw/diffs-language-pack
```

Paket olmadan da desteklenmeyen diller okunabilir düz metin olarak işlenir. Üst kaynak kataloğu için [Diffs Language Pack Plugin'i](/tr/plugins/reference/diffs-language-pack) ve [Shiki dilleri](https://shiki.style/languages) sayfalarına bakın.

## Çıktı ayrıntıları sözleşmesi

Tüm başarılı sonuçlar `changed` içerir: özdeş önce/sonra girdisi yapıt oluşturmadan `false` döndürür; işlenmiş sonuçlar `true` döndürür.

<AccordionGroup>
  <Accordion title="Görüntüleyici alanları (view ve both modları)">
    - `changed`
    - `artifactId`
    - `viewerUrl`
    - `viewerPath`
    - `title`
    - `expiresAt`
    - `inputKind`
    - `fileCount`
    - `mode`
    - `context` (kullanılabilir olduğunda `agentId`, `sessionId`, `messageChannel`, `agentAccountId`)

  </Accordion>
  <Accordion title="Dosya alanları (file ve both modları)">
    - `changed`
    - `artifactId`
    - `expiresAt`
    - `filePath`
    - `path` (ileti aracı uyumluluğu için `filePath` ile aynı değer)
    - `fileBytes`
    - `fileFormat`
    - `fileQuality`
    - `fileScale`
    - `fileMaxWidth`

  </Accordion>
</AccordionGroup>

| Mod      | Döndürülen                                                                                          |
| -------- | --------------------------------------------------------------------------------------------------- |
| `"view"` | Yalnızca görüntüleyici alanları.                                                                    |
| `"file"` | Yalnızca dosya alanları; görüntüleyici yapıtı yoktur.                                               |
| `"both"` | Görüntüleyici alanları ve dosya alanları. Dosya işleme başarısız olursa görüntüleyici yine `fileError` ile döndürülür. |

### Daraltılmış değişmemiş bölümler

Görüntüleyici, `N unmodified lines` gibi satırlar gösterir. Genişletme denetimleri yalnızca işlenmiş farkta genişletilebilir bağlam verileri olduğunda görünür (önce/sonra girdisinde yaygındır). Birçok birleşik yama, parçalarında bağlam gövdelerini içermez; bu nedenle satır genişletme denetimi olmadan görünebilir — bu beklenen bir durumdur, hata değildir. `expandUnchanged` yalnızca genişletilebilir bağlam bulunduğunda geçerlidir.

### Çok dosyalı gezinme

Birden fazla dosyaya dokunan yamalar, değiştirilen dosyaların özet kartıyla başlar: toplam `+N` / `-N` sayıları, dosya başına sayılar, eklendi/silindi/yeniden adlandırıldı rozetleri ve her dosyaya atlayan bağlantı bağlantıları. İşlenmiş PNG/PDF dosyaları dosya başına başlık sayılarını korur, ancak statik dosyada işlevsiz oldukları için etkileşimli görünüm geçişlerini kaldırır.

## Plugin varsayılanları

Plugin genelindeki varsayılanları `~/.openclaw/openclaw.json` içinde ayarlayın:

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          defaults: {
            fontFamily: "Fira Code",
            fontSize: 15,
            lineSpacing: 1.6,
            layout: "unified",
            showLineNumbers: true,
            diffIndicators: "bars",
            wordWrap: true,
            background: true,
            theme: "dark",
            fileFormat: "png",
            fileQuality: "standard",
            fileScale: 2,
            fileMaxWidth: 960,
            mode: "both",
            ttlSeconds: 21600,
          },
        },
      },
    },
  },
}
```

Desteklenen `defaults` anahtarları: `fontFamily`, `fontSize`, `lineSpacing`, `layout`, `showLineNumbers`, `diffIndicators`, `wordWrap`, `background`, `theme`, `fileFormat`, `fileQuality`, `fileScale`, `fileMaxWidth`, `mode`, `ttlSeconds`. Açık araç çağrısı parametreleri bunları geçersiz kılar.

### Kalıcı görüntüleyici URL yapılandırması

<ParamField path="viewerBaseUrl" type="string">
  Bir araç çağrısı `baseUrl` değerini geçmediğinde, döndürülen görüntüleyici bağlantıları için Plugin'in sahip olduğu geri dönüş değeri. `http` veya `https` olmalıdır; sorgu/karması içeremez.
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          viewerBaseUrl: "https://gateway.example.com/openclaw",
        },
      },
    },
  },
}
```

## Güvenlik yapılandırması

<ParamField path="security.allowRemoteViewer" type="boolean" default="false">
  `false`: geri döngü dışındaki görüntüleyici rotası istekleri reddedilir. `true`: belirteçli yol geçerliyse uzak görüntüleyicilere izin verilir.
</ParamField>

```json5
{
  plugins: {
    entries: {
      diffs: {
        enabled: true,
        config: {
          security: {
            allowRemoteViewer: false,
          },
        },
      },
    },
  },
}
```

## Yapıt yaşam döngüsü ve depolama

- Görüntüleyici HTML'si ve meta veriler, Diffs plugin blob ad alanı altındaki paylaşılan `state/openclaw.sqlite` veritabanında bulunur. HTML gzip ile sıkıştırılır; SQLite rastgele URL belirtecinin kendisini değil, yalnızca SHA-256 karmasını depolar.
- İşlenen PNG/PDF dosyaları, kanal teslimi bir dosya yolu gerektirdiğinden `$TMPDIR/openclaw-diffs` altında geçici somutlaştırmalar olarak kalır. Bunların süre sonu meta verilerinin sahibi SQLite'tır; hiçbir JSON yan dosyası yazılmaz.
- Varsayılan yapıt TTL'si: 30 dakika. Kabul edilen azami TTL: 6 saat.
- Temizleme, her yapıt oluşturma çağrısından sonra fırsat buldukça çalışır. Önce süresi dolmuş SQLite satırları, ardından bunlara karşılık gelen PNG/PDF dizinleri silinir.
- Yedek bir tarama, satırı olmayan ve 24 saatten eski geçici klasörleri kaldırır. Eski `meta.json`, `file-meta.json` ve `viewer.html` önbellekleri içe aktarılmaz veya okunmaz.

## Görüntüleyici URL'si ve ağ davranışı

Görüntüleyici rotası: `/plugins/diffs/view/{artifactId}/{token}`

Görüntüleyici varlıkları:

- `/plugins/diffs/assets/viewer.js`
- `/plugins/diffs/assets/viewer-runtime.js`
- `/plugins/diffs-language-pack/assets/viewer.js` (yalnızca fark bir dil paketi dili kullandığında)

Görüntüleyici belgesi bu varlıkları görüntüleyici URL'sine göre çözümler; dolayısıyla isteğe bağlı bir `baseUrl` yol öneki de varlık isteklerine aktarılır.

URL çözümleme sırası: araç çağrısı `baseUrl` (katı doğrulamanın ardından) -> plugin `viewerBaseUrl` -> varsayılan geri döngü `127.0.0.1`. Gateway bağlama modu `custom` ise ve `gateway.customBindHost` ayarlanmışsa geri döngü yerine bu ana makine kullanılır.

`baseUrl` kuralları: `http://` veya `https://` olmalıdır; sorgu ve karma reddedilir; kaynak ile isteğe bağlı temel yola izin verilir.

## Güvenlik modeli

<AccordionGroup>
  <Accordion title="Görüntüleyici sağlamlaştırma">
    - Varsayılan olarak yalnızca geri döngü.
    - Katı kimlik ve belirteç kalıbı doğrulamasına sahip belirteçli görüntüleyici yolları.
    - Görüntüleyici yanıtı CSP'si: `default-src 'none'`; betikler/varlıklar yalnızca aynı kaynaktan; dışarıya yönelik `connect-src` yoktur.
    - Uzak erişim etkinleştirildiğinde uzak isabet etmeme hız sınırlaması: 60 saniyede 40 hata, 60 saniyelik kilitlemeyi tetikler (`429 Too Many Requests`).

  </Accordion>
  <Accordion title="Dosya işleme sağlamlaştırması">
    - Ekran görüntüsü tarayıcısı istek yönlendirmesi varsayılan olarak reddeder.
    - Yalnızca `http://127.0.0.1/plugins/diffs/assets/*` kaynağındaki yerel görüntüleyici varlıklarına izin verilir.
    - Harici ağ istekleri engellenir.

  </Accordion>
</AccordionGroup>

## Dosya modu için tarayıcı gereksinimleri

`mode: "file"` ve `mode: "both"`, Chromium uyumlu bir tarayıcı gerektirir.

Çözümleme sırası:

<Steps>
  <Step title="Yapılandırma">
    OpenClaw yapılandırmasındaki `browser.executablePath`.
  </Step>
  <Step title="Ortam değişkenleri">
    - `OPENCLAW_BROWSER_EXECUTABLE_PATH`
    - `BROWSER_EXECUTABLE_PATH`
    - `PLAYWRIGHT_CHROMIUM_EXECUTABLE_PATH`

  </Step>
  <Step title="Platform yedeği">
    Chrome, Chromium, Edge ve Brave için yaygın kurulum yolları ve `PATH` aramaları.
  </Step>
</Steps>

Yaygın hata metni: `Diff PNG/PDF rendering requires a Chromium-compatible browser...`. Chrome, Chromium, Edge veya Brave'i yükleyerek ya da yukarıdaki yürütülebilir dosya yolu seçeneklerinden birini ayarlayarak düzeltin.

## Sorun giderme

<AccordionGroup>
  <Accordion title="Girdi doğrulama hataları">
    - `Provide patch or both before and after text.` -- hem `before` hem de `after` değerlerini ekleyin veya `patch` sağlayın.
    - `Provide either patch or before/after input, not both.` -- girdi modlarını karıştırmayın.
    - `Invalid baseUrl: ...` -- isteğe bağlı yola sahip, sorgu/karması olmayan bir `http(s)` kaynağı kullanın.
    - `{field} exceeds maximum size (...)` -- yük boyutunu azaltın.
    - Büyük yama reddi -- yama dosyası sayısını veya toplam satır sayısını azaltın.

  </Accordion>
  <Accordion title="Görüntüleyici erişilebilirliği">
    - Görüntüleyici URL'si varsayılan olarak `127.0.0.1` olarak çözümlenir.
    - Uzak erişim için plugin `viewerBaseUrl` değerini ayarlayın, her çağrıda `baseUrl` iletin veya `gateway.bind=custom` değerini `gateway.customBindHost` ile kullanın.
    - Aynı ana makinedeki bir proxy için (örneğin Tailscale Serve) `gateway.trustedProxies` geri döngüyü içeriyorsa iletilen istemci IP'si üst bilgileri bulunmayan ham geri döngü görüntüleyici istekleri tasarım gereği kapalı biçimde başarısız olur.
    - Bu proxy topolojisinde bir ek için `mode: "file"`/`"both"` kullanmayı tercih edin veya paylaşılabilir bir görüntüleyici bağlantısı için `security.allowRemoteViewer` ile birlikte plugin `viewerBaseUrl`/bir proxy `baseUrl` değerini bilinçli olarak etkinleştirin.
    - `security.allowRemoteViewer` seçeneğini yalnızca harici görüntüleyici erişimi amaçlandığında etkinleştirin.

  </Accordion>
  <Accordion title="Değiştirilmemiş satırlar satırında genişletme düğmesi yok">
    Genişletilebilir bağlam içermeyen yama girdisi için beklenen davranıştır; görüntüleyici hatası değildir.
  </Accordion>
  <Accordion title="Yapıt bulunamadı">
    - Yapıtın süresi TTL nedeniyle doldu.
    - Belirteç veya yol değişti.
    - Temizleme eski verileri kaldırdı.

  </Accordion>
</AccordionGroup>

## Operasyonel rehberlik

- Canvas'taki yerel etkileşimli incelemeler için `mode: "view"` kullanmayı tercih edin.
- Ek gerektiren giden sohbet kanalları için `mode: "file"` kullanmayı tercih edin.
- Dağıtımınız uzak görüntüleyici URL'leri gerektirmedikçe `allowRemoteViewer` seçeneğini devre dışı tutun.
- Hassas farklar için açıkça kısa bir `ttlSeconds` ayarlayın.
- Gerekli olmadığında fark girdisinde gizli bilgiler göndermekten kaçının.
- Kanalınız görüntüleri yoğun biçimde sıkıştırıyorsa (örneğin Telegram veya WhatsApp), PDF çıktısını (`fileFormat: "pdf"`) tercih edin.

<Note>
Fark işleme motoru [Diffs](https://diffs.com) tarafından desteklenmektedir.
</Note>

## İlgili konular

- [Tarayıcı](/tr/tools/browser)
- [Pluginler](/tr/tools/plugin)
- [Araçlara genel bakış](/tr/tools)
