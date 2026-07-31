---
read_when:
    - Codex, Claude veya Cursor uyumlu bir paket yüklemek istiyorsunuz
    - OpenClaw'ın paket içeriğini yerel özelliklerle nasıl eşlediğini anlamanız gerekir
    - Paket algılamasında veya eksik yeteneklerde hata ayıklıyorsunuz
summary: Codex, Claude ve Cursor paketlerini OpenClaw pluginleri olarak yükleme ve kullanma
title: Plugin paketleri
x-i18n:
    generated_at: "2026-07-26T23:26:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d44006866238f53ee2e3e8126cc4f7ed6f7413534257775f7904c9b877778c59
    source_path: plugins/bundles.md
    workflow: 16
---

OpenClaw üç harici ekosistemden plugin yükleyebilir: **Codex**, **Claude**
ve **Cursor**. Bunlara **paketler** denir; OpenClaw'ın Skills, kancalar ve MCP araçları
gibi yerel özelliklerle eşleştirdiği içerik ve meta veri paketleridir.

<Info>
  Paketler, yerel OpenClaw pluginleriyle **aynı değildir**. Yerel pluginler
  işlem içinde çalışır ve her türlü yeteneği kaydedebilir. Paketler ise seçici
  özellik eşlemesine ve daha dar bir güven sınırına sahip içerik paketleridir.
</Info>

## Paketler neden vardır?

Birçok yararlı plugin Codex, Claude veya Cursor biçiminde yayımlanır. OpenClaw,
yazarların bunları yerel OpenClaw pluginleri olarak yeniden yazmasını istemek
yerine bu biçimleri algılar ve desteklenen içeriklerini yerel özellik kümesiyle
eşleştirir. Bir Claude komut paketini veya Codex Skills paketini yükleyip hemen
kullanabilirsiniz.

## Paket yükleme

<Steps>
  <Step title="Bir dizinden, arşivden veya pazaryerinden yükleyin">
    ```bash
    # Yerel dizin
    openclaw plugins install ./my-bundle

    # Arşiv
    openclaw plugins install ./my-bundle.tgz

    # Claude pazaryeri
    openclaw plugins marketplace list <source>
    openclaw plugins install <plugin> --marketplace <source>
    ```

    `<source>`, yerel bir pazaryeri yolu/deposu veya bir git/GitHub kaynağıdır.

  </Step>

  <Step title="Algılamayı doğrulayın">
    ```bash
    openclaw plugins list
    openclaw plugins inspect <id>
    ```

    Paketler, `Format: bundle` ile birlikte `codex`,
    `claude` veya `cursor` değerine sahip bir `Bundle format:` gösterir.

  </Step>

  <Step title="Yeniden başlatın ve kullanın">
    ```bash
    openclaw gateway restart
    ```

    Eşlenen özellikler (Skills, kancalar, MCP araçları, LSP varsayılanları) bir sonraki oturumda kullanılabilir.

  </Step>
</Steps>

## OpenClaw paketlerden neleri eşler?

Günümüzde her paket özelliği OpenClaw'da çalışmaz. Aşağıda çalışanlar ile
algılanan ancak henüz bağlanmamış olanlar yer almaktadır.

### Şu anda desteklenenler

| Özellik         | Nasıl eşlendiği                                                                                   | Geçerli olduğu biçimler |
| --------------- | ------------------------------------------------------------------------------------------------- | ----------------------- |
| Skills içeriği  | Paket Skills kökleri normal OpenClaw Skills kökleri olarak yüklenir                               | Tüm biçimler            |
| Komutlar        | `commands/` ve `.cursor/commands/`, Skills kökleri olarak değerlendirilir                         | Claude, Cursor          |
| Kanca paketleri | OpenClaw tarzı `HOOK.md` + `handler.ts` düzenleri                                                | Codex                   |
| MCP araçları    | Paket MCP yapılandırması gömülü OpenClaw ayarlarıyla birleştirilir; desteklenen stdio ve HTTP sunucuları yüklenir | Tüm biçimler |
| LSP sunucuları  | Claude `.lsp.json` ve manifestte bildirilen `lspServers`, gömülü OpenClaw LSP varsayılanlarıyla birleştirilir | Claude |
| Ayarlar         | Claude `settings.json`, gömülü OpenClaw varsayılanları olarak içe aktarılır                          | Claude                  |

#### Skills içeriği

- Paket Skills kökleri normal OpenClaw Skills kökleri olarak yüklenir.
- Claude `commands/` kökleri ek Skills kökleri olarak değerlendirilir.
- Cursor `.cursor/commands/` kökleri ek Skills kökleri olarak değerlendirilir.

Claude Markdown komut dosyaları ve Cursor komut Markdown'ları, normal OpenClaw
Skills yükleyicisi üzerinden çalışır.

#### Kanca paketleri

Paket kanca kökleri **yalnızca** normal OpenClaw kanca paketi düzenini
kullandıklarında çalışır: `HOOK.md` ile birlikte `handler.ts` veya `handler.js`. Günümüzde bu öncelikle
Codex uyumlu durumdur.

#### Gömülü OpenClaw için MCP

- Etkin paketler MCP sunucu yapılandırmasına katkıda bulunabilir.
- OpenClaw, paket MCP yapılandırmasını geçerli gömülü OpenClaw
  ayarlarına `mcpServers` olarak birleştirir.
- OpenClaw, stdio sunucularını başlatarak veya HTTP sunucularına bağlanarak
  gömülü OpenClaw ajan dönüşleri sırasında desteklenen paket MCP araçlarını sunar.
- `coding` ve `messaging` araç profilleri varsayılan olarak paket MCP araçlarını
  içerir; bir ajan veya Gateway için kapsam dışında bırakmak üzere `tools.deny: ["bundle-mcp"]` kullanın.
- Proje yerelindeki gömülü ajan ayarları paket varsayılanlarından sonra uygulanmaya
  devam eder; böylece çalışma alanı ayarları gerektiğinde paket MCP girdilerini geçersiz kılabilir.
- Paket MCP araç katalogları kayıttan önce belirlenimsel olarak sıralanır; böylece
  üst kaynak `listTools()` sırası değişiklikleri istem önbelleğinin araç bloklarında gereksiz değişime yol açmaz.

##### Aktarımlar

MCP sunucuları stdio veya HTTP aktarımını kullanabilir.

**Stdio**, bir alt süreç başlatır:

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "command": "node",
        "args": ["server.js"],
        "env": { "PORT": "3000" }
      }
    }
  }
}
```

**HTTP**, çalışan bir MCP sunucusuna bağlanır ve `streamable-http` istenmediği sürece
varsayılan olarak `sse` kullanır:

```json
{
  "mcp": {
    "servers": {
      "my-server": {
        "url": "http://localhost:3100/mcp",
        "transport": "streamable-http",
        "headers": {
          "Authorization": "Bearer ${MY_SECRET_TOKEN}"
        },
        "connectionTimeoutMs": 30000
      }
    }
  }
}
```

- `transport`, `"streamable-http"` veya `"sse"` kabul eder; belirtilmezse varsayılan değer `sse` olur.
- `type: "http"`, CLI'a özgü bir alt kaynak biçimidir; OpenClaw yapılandırmasında `transport: "streamable-http"` kullanın. `openclaw mcp set` ve `openclaw doctor --fix` yaygın diğer adı normalleştirir.
- Yalnızca `http:` ve `https:` URL şemalarına izin verilir.
- `headers` değerleri `${ENV_VAR}` yerleştirmesini destekler.
- Hem `command` hem de `url` içeren bir sunucu girdisi reddedilir.
- URL kimlik bilgileri (kullanıcı bilgisi ve sorgu parametreleri), araç
  açıklamalarında ve günlüklerde gizlenir.
- `connectionTimeoutMs`, hem stdio hem de HTTP aktarımları için varsayılan
  30 saniyelik bağlantı zaman aşımını geçersiz kılar. İstek zaman aşımı varsayılan olarak 60 saniyedir ve
  `requestTimeoutMs` ile geçersiz kılınabilir.

##### Araç adlandırma

OpenClaw, paket MCP araçlarını `serverName__toolName` biçiminde sağlayıcı açısından
güvenli adlarla kaydeder. Örneğin, `memory_search` aracını sunan ve `"vigil-harbor"`
anahtarıyla tanımlanan bir sunucu, `vigil-harbor__memory_search` olarak kaydedilir.

- `A-Za-z0-9_-` dışındaki karakterler `-` ile değiştirilir.
- Harf olmayan bir karakterle başlayacak parçalara harf öneki eklenir; böylece
  `12306` gibi sayısal sunucu anahtarları sağlayıcı açısından güvenli araç öneklerine dönüşür.
- Sunucu önekleri en fazla 30 karakter olabilir.
- Tam araç adları en fazla 64 karakter olabilir.
- Boş sunucu adları için `mcp` kullanılır.
- Çakışan temizlenmiş adlar sayısal son eklerle birbirinden ayrılır.
- Sunulan nihai araç sırası güvenli ada göre belirlenimseldir ve yinelenen
  gömülü ajan dönüşlerinde önbellek kararlılığını korur.
- Profil filtreleme, bir paket MCP sunucusundaki her aracı
  `bundle-mcp` tarafından sahip olunan bir plugin olarak değerlendirir; böylece profil izin/verme
  listeleri tek tek sunulan araç adlarına veya `bundle-mcp` plugin anahtarına başvurabilir.

#### Gömülü OpenClaw ayarları

Paket etkinleştirildiğinde Claude `settings.json`, varsayılan gömülü OpenClaw
ayarları olarak içe aktarılır. OpenClaw, uygulamadan önce kabuk geçersiz kılma anahtarlarını
temizler:

- `shellPath`
- `shellCommandPrefix`

#### Gömülü OpenClaw LSP

- Etkin Claude paketleri LSP sunucu yapılandırmasına katkıda bulunabilir.
- OpenClaw, `.lsp.json` ile manifestte bildirilen tüm `lspServers` yollarını yükler.
- Paket LSP yapılandırması, geçerli gömülü OpenClaw LSP
  varsayılanlarıyla birleştirilir.
- Günümüzde yalnızca desteklenen stdio tabanlı LSP sunucuları çalıştırılabilir; desteklenmeyen
  aktarımlar yine de `openclaw plugins inspect <id>` içinde görünür.

### Algılanan ancak çalıştırılmayanlar

Bunlar tanınır ve tanılamalarda gösterilir ancak OpenClaw bunları çalıştırmaz:

- Claude `agents`, `hooks/hooks.json` otomasyonu, `outputStyles`
- Cursor `.cursor/agents`, `.cursor/hooks.json`, `.cursor/rules`
- Yetenek raporlamasının ötesindeki Codex `.app.json` meta verileri

## Paket biçimleri

<AccordionGroup>
  <Accordion title="Codex paketleri">
    İşaretleyiciler: `.codex-plugin/plugin.json`

    İsteğe bağlı içerik: `skills/`, `hooks/`, `.mcp.json`, `.app.json`

    Codex paketleri, Skills kökleri ve OpenClaw tarzı kanca paketi
    dizinleri (`HOOK.md` + `handler.ts`) kullandıklarında OpenClaw'a en iyi şekilde uyar.

  </Accordion>

  <Accordion title="Claude paketleri">
    İki algılama modu:

    - **Manifest tabanlı:** `.claude-plugin/plugin.json`
    - **Manifestsiz:** varsayılan Claude düzeni (`skills/`, `commands/`, `agents/`, `hooks/`, `.mcp.json`, `.lsp.json`, `settings.json`)

    Claude'a özgü davranış:

    - `commands/`, Skills içeriği olarak değerlendirilir
    - `settings.json`, gömülü OpenClaw ayarlarına içe aktarılır (kabuk geçersiz kılma anahtarları temizlenir)
    - `.mcp.json`, desteklenen stdio araçlarını gömülü OpenClaw'a sunar
    - `.lsp.json` ile manifestte bildirilen `lspServers` yolları, gömülü OpenClaw LSP varsayılanlarına yüklenir
    - `hooks/hooks.json` algılanır ancak çalıştırılmaz
    - Manifestteki özel bileşen yolları eklemelidir; varsayılanları değiştirmek yerine genişletir

  </Accordion>

  <Accordion title="Cursor paketleri">
    İşaretleyiciler: `.cursor-plugin/plugin.json`

    İsteğe bağlı içerik: `skills/`, `.cursor/commands/`, `.cursor/agents/`, `.cursor/rules/`, `.cursor/hooks.json`, `.mcp.json`

    - `.cursor/commands/`, Skills içeriği olarak değerlendirilir
    - `.cursor/rules/`, `.cursor/agents/` ve `.cursor/hooks.json` yalnızca algılanır

  </Accordion>
</AccordionGroup>

## Algılama önceliği

OpenClaw önce yerel plugin biçimini denetler:

1. `openclaw.plugin.json` veya `openclaw.extensions` içeren geçerli bir `package.json` - **yerel plugin** olarak değerlendirilir
2. Paket işaretleyicileri (`.codex-plugin/`, `.claude-plugin/` veya varsayılan Claude/Cursor düzeni) - **paket** olarak değerlendirilir

Bir dizin her ikisini de içeriyorsa OpenClaw yerel yolu kullanır. Bu, çift
biçimli paketlerin kısmen paket olarak yüklenmesini önler.

## Çalışma zamanı bağımlılıkları ve temizleme

- Üçüncü taraf uyumlu paketlere başlangıçta `npm install` onarımı uygulanmaz. Bunlar,
  `openclaw plugins install` üzerinden yüklenmeli ve ihtiyaç duydukları her şeyi
  yüklü plugin dizininde sağlamalıdır.
- OpenClaw'a ait paketlenmiş pluginler ya çekirdekte hafif biçimde sunulur ya da
  plugin yükleyicisi üzerinden indirilebilir. Gateway başlangıcı bunlar için hiçbir zaman
  paket yöneticisi çalıştırmaz.
- `openclaw doctor --fix`, eski yerel paketlenmiş plugin yükleme kayıtlarını kaldırır
  ve yapılandırma hâlâ bunlara başvuruyorsa yerel plugin dizininde bulunmayan
  indirilebilir pluginleri kurtarabilir.

## Güvenlik

Paketlerin güven sınırı yerel pluginlere göre daha dardır:

- OpenClaw, rastgele paket çalışma zamanı modüllerini işlem içinde **yüklemez**.
- Skills ve kanca paketi yolları plugin kökü içinde kalmalıdır (sınır denetimli).
- Ayar dosyaları aynı sınır denetimleriyle okunur.
- Desteklenen stdio MCP sunucuları alt süreçler olarak başlatılabilir.

Bu, paketleri varsayılan olarak daha güvenli kılar; ancak üçüncü taraf
paketleri sundukları özellikler açısından yine de güvenilir içerik olarak değerlendirmelisiniz.

## Sorun giderme

<AccordionGroup>
  <Accordion title="Paket algılanıyor ancak yetenekler çalışmıyor">
    `openclaw plugins inspect <id>` komutunu çalıştırın. Bir yetenek listeleniyor ancak bağlı değil olarak
    işaretleniyorsa bu, bozuk bir kurulum değil, ürün sınırlamasıdır.
  </Accordion>

  <Accordion title="Claude komut dosyaları görünmüyor">
    Paketin etkinleştirildiğinden ve markdown dosyalarının algılanan bir
    `commands/` veya `skills/` kökü içinde bulunduğundan emin olun.
  </Accordion>

  <Accordion title="Claude ayarları uygulanmıyor">
    Yalnızca `settings.json` içindeki gömülü OpenClaw ayarları desteklenir. OpenClaw,
    paket ayarlarını ham yapılandırma yamaları olarak değerlendirmez.
  </Accordion>

  <Accordion title="Claude kancaları yürütülmüyor">
    `hooks/hooks.json` yalnızca algılama amaçlıdır. Çalıştırılabilir kancalara ihtiyacınız varsa
    OpenClaw kanca paketi düzenini kullanın veya yerel bir plugin sunun.
  </Accordion>
</AccordionGroup>

## İlgili

- [Pluginleri Yükleme ve Yapılandırma](/tr/tools/plugin)
- [Plugin Oluşturma](/tr/plugins/building-plugins) - yerel bir plugin oluşturun
- [Plugin Manifesti](/tr/plugins/manifest) - yerel manifest şeması
