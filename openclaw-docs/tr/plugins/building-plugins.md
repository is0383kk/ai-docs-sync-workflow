---
doc-schema-version: 1
read_when:
    - Yeni bir OpenClaw plugin'i oluşturmak istiyorsunuz
    - Plugin geliştirme için hızlı başlangıç kılavuzuna ihtiyacınız var
    - Kanal, sağlayıcı, CLI arka ucu, araç veya kanca belgeleri arasında seçim yapıyorsunuz
sidebarTitle: Getting Started
summary: İlk OpenClaw plugininizi dakikalar içinde oluşturun
title: Plugin oluşturma
x-i18n:
    generated_at: "2026-07-26T23:28:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d156ea305e46d3ca311a0b2cfc42e2c4522f6f10eb70cdd5526d9e9fcd7d4ef
    source_path: plugins/building-plugins.md
    workflow: 16
---

Plugin'ler, çekirdeği değiştirmeden OpenClaw'u genişletir. Bir plugin; mesajlaşma
kanalı, model sağlayıcısı, yerel CLI arka ucu, ajan aracı, kanca, medya sağlayıcısı
veya plugin'in sahip olduğu başka bir yetenek ekleyebilir.

OpenClaw deposuna harici bir plugin eklemeniz gerekmez. Paketi
[ClawHub](/clawhub) üzerinde yayımlayın; kullanıcılar paketi şu komutla yükler:

```bash
openclaw plugins install clawhub:<package-name>
```

Çıplak paket belirtimleri, kullanıma geçiş sürecinde npm üzerinden yüklenmeye devam eder. ClawHub
çözümlemesi istediğinizde `clawhub:` önekini kullanın.

## Gereksinimler

- Node 22.22.3+, Node 24.15+ veya Node 25.9+ ve `npm` ya da `pnpm`.
- TypeScript ESM modülleri.
- Depo içindeki paketlenmiş plugin çalışmaları için depoyu klonlayın ve `pnpm install` komutunu çalıştırın.
  OpenClaw, paketlenmiş plugin'leri `extensions/*` çalışma alanı paketlerinden
  keşfettiği için kaynak kod kullanıma alma üzerinden plugin geliştirme yalnızca pnpm ile yapılır.

## Plugin biçimini seçme

<CardGroup cols={2}>
  <Card title="Kanal plugin'i" icon="messages-square" href="/tr/plugins/sdk-channel-plugins">
    OpenClaw'u bir mesajlaşma platformuna bağlayın.
  </Card>
  <Card title="Sağlayıcı plugin'i" icon="cpu" href="/tr/plugins/sdk-provider-plugins">
    Bir model, medya, arama, getirme, konuşma veya gerçek zamanlı sağlayıcı ekleyin.
  </Card>
  <Card title="CLI arka uç plugin'i" icon="terminal" href="/tr/plugins/cli-backend-plugins">
    OpenClaw model geri dönüşü üzerinden yerel bir yapay zekâ CLI'sı çalıştırın.
  </Card>
  <Card title="Araç plugin'i" icon="wrench" href="/tr/plugins/tool-plugins">
    Ajan araçlarını kaydedin.
  </Card>
</CardGroup>

## Hızlı başlangıç

Gerekli bir ajan aracını kaydederek asgari bir araç plugin'i oluşturun. Bu,
kullanışlı en kısa plugin biçimidir ve paketi, manifesti, giriş noktasını ve
yerel doğrulamayı kapsar.

<Steps>
  <Step title="Paket meta verilerini oluşturma">
    <CodeGroup>

```json package.json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "dependencies": {
    "typebox": "1.1.39"
  },
  "peerDependencies": {
    "openclaw": ">=2026.3.24-beta.2"
  },
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2",
      "minGatewayVersion": "2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2",
      "pluginSdkVersion": "2026.3.24-beta.2"
    }
  }
}
```

```json openclaw.plugin.json
{
  "id": "my-plugin",
  "name": "My Plugin",
  "description": "Adds a custom tool to OpenClaw",
  "contracts": {
    "tools": ["my_tool"]
  },
  "activation": {
    "onStartup": true
  },
  "configSchema": {
    "type": "object",
    "additionalProperties": false
  }
}
```

    </CodeGroup>

    Yayımlanan harici plugin'ler, çalışma zamanı girişlerini derlenmiş JavaScript
    dosyalarına yönlendirmelidir. Giriş noktası sözleşmesinin tamamı için
    [SDK giriş noktaları](/tr/plugins/sdk-entrypoints) bölümüne bakın.

    Yapılandırması olmasa bile her plugin'in bir manifeste ihtiyacı vardır. OpenClaw'un
    her plugin çalışma zamanını önceden yüklemeden sahipliği keşfedebilmesi için çalışma zamanı araçları
    `contracts.tools` içinde bulunmalıdır. `activation.onStartup` değerini
    bilinçli olarak ayarlayın; bu örnek Gateway başlatılırken yüklenir.

    Ana makine tarafından güvenilen plugin yüzeyleri de manifest ile sınırlandırılır ve yüklü
    plugin'ler için açık bildirim gerektirir: `api.registerAgentToolResultMiddleware(...)`,
    her hedef çalışma zamanının `contracts.agentToolResultMiddleware` içinde listelenmesini;
    `api.registerTrustedToolPolicy(...)` ise her ilke kimliğinin
    `contracts.trustedToolPolicies` içinde bulunmasını gerektirir. Bu bildirimler, yükleme sırasındaki
    inceleme ile çalışma zamanı kaydını uyumlu tutar.

    Tüm manifest alanları için [Plugin manifesti](/tr/plugins/manifest) bölümüne bakın.

  </Step>

  <Step title="Aracı kaydetme">
    ```typescript index.ts
    import { Type } from "typebox";
    import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

    export default definePluginEntry({
      id: "my-plugin",
      name: "My Plugin",
      description: "Adds a custom tool to OpenClaw",
      register(api) {
        api.registerTool({
          name: "my_tool",
          description: "Echo one input value",
          parameters: Type.Object({ input: Type.String() }),
          outputSchema: Type.Object(
            { input: Type.String() },
            { additionalProperties: false },
          ),
          async execute(_id, params) {
            const details = { input: params.input };
            return {
              content: [{ type: "text", text: `Got: ${params.input}` }],
              details,
            };
          },
        });
      },
    });
    ```

    Kanal dışı plugin'ler için `definePluginEntry` kullanın. Kanal plugin'leri ise
    bunun yerine `openclaw/plugin-sdk/core` içindeki `defineChannelPluginEntry` öğesini kullanır.

  </Step>

  <Step title="Çalışma zamanını test etme">
    Yüklü veya harici bir plugin için yüklenen çalışma zamanını inceleyin:

    ```bash
    openclaw plugins inspect my-plugin --runtime --json
    ```

    Plugin bir CLI komutu kaydediyorsa bu komutu da çalıştırıp çıktıyı
    doğrulayın; örneğin `openclaw demo-plugin ping`.

    Bu depodaki paketlenmiş bir plugin için OpenClaw, kaynak kod kullanıma alma
    plugin paketlerini `extensions/*` çalışma alanından keşfeder. En yakın hedefli
    testi çalıştırın:

    ```bash
    pnpm test extensions/my-plugin/
    pnpm check
    ```

  </Step>

  <Step title="Paket yüklemesini test etme">
    Yayımlamadan önce, paketlemeye hazır bir plugin'i kullanıcıların elde edeceği
    aynı yükleme biçimiyle test edin. Önce bir derleme adımı ekleyin, `openclaw.extensions` gibi çalışma zamanı
    girişlerini `./dist/index.js` gibi derlenmiş JavaScript'e yönlendirin ve
    `npm pack` öğesinin bu `dist/` çıktısını içerdiğinden emin olun. TypeScript kaynak girişleri
    yalnızca kaynak kod kullanıma almaları ve yerel geliştirme yolları içindir.

    Ardından plugin'i paketleyin ve tarball dosyasını `npm-pack:` ile yükleyin:

    ```bash
    npm pack --pack-destination /tmp
    openclaw plugins install npm-pack:/tmp/<plugin-package>.tgz --force
    openclaw plugins inspect my-plugin --runtime --json
    ```

    `npm-pack:`, OpenClaw'un plugin başına yönetilen npm projesini kullanır; böylece
    kaynak kod kullanıma alma testinin gizleyebileceği çalışma zamanı bağımlılığı hatalarını yakalar. Katalogla bağlantılı
    resmî güveni değil, paket ve bağımlılık biçimini doğrular.
    Çalışma zamanı içe aktarımları `dependencies` veya `optionalDependencies` içinde olmalıdır;
    yalnızca `devDependencies` içinde bırakılan bağımlılıklar, yönetilen çalışma zamanı
    projesi için yüklenmez.

    Resmî veya ayrıcalıklı plugin davranışının nihai doğrulaması olarak ham bir arşiv/yol
    yüklemesi kullanmayın. Ham kaynaklar yerel hata ayıklama için kullanışlıdır ancak
    npm veya ClawHub yüklemeleriyle aynı bağımlılık yolunu doğrulamaz. Plugin'iniz
    güvenilen resmî plugin durumuna dayanıyorsa, katalog destekli resmî bir yükleme
    veya resmî güveni kaydeden yayımlanmış bir paket yolu üzerinden ikinci bir doğrulama
    ekleyin. Yükleme kökü ve bağımlılık sahipliği ayrıntıları için
    [Plugin bağımlılık çözümlemesi](/tr/plugins/dependency-resolution) bölümüne
    bakın.

  </Step>

  <Step title="Yayımlama">
    Yayımlamadan önce paketi doğrulayın:

    ```bash
    clawhub package publish your-org/your-plugin --dry-run
    clawhub package publish your-org/your-plugin
    ```

    Standart ClawHub paket parçacıkları `docs/snippets/plugin-publish/` içinde bulunur.

  </Step>

  <Step title="Yükleme">
    Yayımlanan paketi ClawHub üzerinden yükleyin:

    ```bash
    openclaw plugins install clawhub:your-org/your-plugin
    ```

  </Step>
</Steps>

<a id="registering-agent-tools"></a>

## Araçları kaydetme

Araçlar gerekli veya isteğe bağlı olabilir. Gerekli araçlar, plugin
etkin olduğunda her zaman kullanılabilir. İsteğe bağlı araçların, OpenClaw
sahip plugin çalışma zamanını yüklemeden önce kullanıcının açıkça kabul etmesini gerektirir.

Araç fabrikaları; `deliveryContext`, mevcut olduğunda etkin platform görüşmesi için
`nativeChannelId` ve `requesterSenderId` dahil olmak üzere güvenilen çalışma zamanı bağlamını alır.

```typescript
register(api) {
  api.registerTool(
    {
      name: "workflow_tool",
      description: "Run a workflow",
      parameters: Type.Object({ pipeline: Type.String() }),
      outputSchema: Type.Object(
        { pipeline: Type.String() },
        { additionalProperties: false },
      ),
      async execute(_id, params) {
        return {
          content: [{ type: "text", text: params.pipeline }],
          details: { pipeline: params.pipeline },
        };
      },
    },
    { optional: true },
  );
}
```

`outputSchema` isteğe bağlıdır. [Kod Modu](/tools/code-mode) ve
[Araç Arama](/tr/tools/tool-search) tarafından kullanılan yapılandırılmış `details` değerini açıklar. Katalog
çağrıları, yürütmeden önce geçersiz şemaları reddeder ve araç kancalarından sonra nihai değeri
doğrular. Kararlı bir JSON sonucu olmayan araçlarda bunu kullanmayın. Sözleşmenin tamamı için
[Araç plugin'leri](/tr/plugins/tool-plugins#output-contracts) bölümüne bakın.

`api.registerTool(...)` ile kaydedilen her araç, plugin manifestinde de
bildirilmelidir:

```json
{
  "contracts": {
    "tools": ["workflow_tool"]
  },
  "toolMetadata": {
    "workflow_tool": {
      "optional": true
    }
  }
}
```

Kullanıcılar `tools.allow` ile katılım sağlar:

```json5
{
  tools: { allow: ["workflow_tool"] }, // veya bir plugin'in tüm araçları için ["my-plugin"]
}
```

İsteğe bağlı araçlar, bir aracın modele sunulup sunulmayacağını denetler. Bir araç
veya kancanın model tarafından seçilmesinden sonra ve eylem çalışmadan önce
onay istemesi gerekiyorsa [plugin izin isteklerini](/tr/plugins/plugin-permission-requests) kullanın.

İsteğe bağlı araçları yan etkiler, sıra dışı ikili dosyalar veya varsayılan olarak
sunulmaması gereken yetenekler için kullanın. Araç adları çekirdek araç
adlarıyla çakışmamalıdır; çakışmalar atlanır ve plugin tanılamalarında bildirilir. Hatalı
kayıtlar da aynı şekilde atlanır ve bildirilir: boş olmayan bir
`name` öğesinin eksik olması, `execute` öğesinin işlev olmaması veya `parameters`
nesnesi bulunmayan bir araç tanımlayıcısı.

Araç fabrikaları, çalışma zamanının sağladığı bir bağlam nesnesi alır. Bir aracın geçerli
dönüşte etkin modeli günlüğe kaydetmesi, görüntülemesi veya modele uyarlanması gerektiğinde
`ctx.activeModel` kullanın; bu, `provider`, `modelId` ve `modelRef` içerebilir. Bunu
yerel operatöre, yüklü plugin koduna veya değiştirilmiş bir OpenClaw çalışma zamanına karşı
güvenlik sınırı olarak değil, bilgilendirici çalışma zamanı meta verisi olarak değerlendirin. Hassas
yerel araçlar yine de açık bir plugin veya operatör katılımı gerektirmeli ve
etkin model meta verisi eksik ya da uygun değilse kapalı şekilde başarısız olmalıdır.

Manifest sahipliği ve keşfi bildirir; yürütme yine de canlı
kayıtlı araç uygulamasını çağırır. OpenClaw'un araç açıkça izin verilenler listesine
eklenene kadar bu plugin çalışma zamanını yüklemekten kaçınabilmesi için `toolMetadata.<tool>.optional: true`
ile `api.registerTool(..., { optional: true })` öğelerini uyumlu tutun.

## İçe aktarma kuralları

Odaklanmış SDK alt yollarından içe aktarın:

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createPluginRuntimeStore } from "openclaw/plugin-sdk/runtime-store";
```

Plugin paketinizde, dahili içe aktarımlar için `api.ts` ve
`runtime-api.ts` gibi yerel barrel dosyalarını kullanın. Kendi plugin'inizi bir
SDK yolu üzerinden içe aktarmayın. Sağlayıcıya özgü yardımcılar, bağlantı gerçekten
genel olmadığı sürece sağlayıcı paketinde kalmalıdır.

Özel Gateway RPC yöntemleri gelişmiş bir giriş noktasıdır. Bunları
plugin'e özgü bir önekte tutun; `config.*`,
`exec.approvals.*`, `operator.admin.*`, `wizard.*` ve `update.*` gibi çekirdek yönetici ad alanları ayrılmış
olarak kalır ve `operator.admin` olarak çözümlenir.
`openclaw/plugin-sdk/gateway-method-runtime` köprüsü, `contracts.gatewayMethodDispatch: ["authenticated-request"]` bildiren plugin HTTP
rotaları için ayrılmıştır.

İçe aktarma haritasının tamamı için [Plugin SDK'ya genel bakış](/tr/plugins/sdk-overview) bölümüne bakın.

OpenClaw SDK uyumluluk alanları, düzenleyicilerin geçiş uyarıları olarak gösterdiği TypeScript
`@deprecated` ek açıklamalarını taşır. Bunları derleme sırasında zorunlu kılmak için
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/) gibi tür bilgisine duyarlı bir kuralı
etkinleştirin. Oxlint tür bilgisine duyarlı olmadığından bu ek açıklamaları zorunlu kılamaz.

## Gönderim öncesi kontrol listesi

<Check>**package.json** doğru `openclaw` meta verilerine sahip</Check>
<Check>**openclaw.plugin.json** manifesti mevcut ve geçerli</Check>
<Check>Giriş noktası `defineChannelPluginEntry` veya `definePluginEntry` kullanıyor</Check>
<Check>Tüm içe aktarmalar odaklanmış `plugin-sdk/<subpath>` yollarını kullanıyor</Check>
<Check>Dahili içe aktarmalar, SDK'nın kendi kendine içe aktarımlarını değil yerel modülleri kullanıyor</Check>
<Check>Testler geçiyor (`pnpm test <bundled-plugin-root>/my-plugin/`)</Check>
<Check>`pnpm check` geçiyor (depo içi pluginler)</Check>

## Beta sürümlerine karşı test etme

1. [openclaw/openclaw](https://github.com/openclaw/openclaw/releases) sürümlerini izleyin (`Watch` > `Releases`). Beta etiketleri `v2026.3.N-beta.1` biçimindedir. Sürüm duyuruları için X'te [@openclaw](https://x.com/openclaw) hesabını da takip edebilirsiniz.
2. Plugininizi beta etiketi yayımlanır yayımlanmaz buna karşı test edin. Kararlı sürümden önceki süre genellikle yalnızca birkaç saattir.
3. Testten sonra `plugin-forum` Discord kanalındaki ([discord.gg/clawd](https://discord.gg/clawd)) plugin başlığınıza `all good` veya neyin bozulduğunu yazın. Henüz bir başlığınız yoksa oluşturun.
4. Bir şey bozulursa `Beta blocker: <plugin-name> - <summary>` başlıklı bir sorun açın veya mevcut sorunu güncelleyin ve `beta-blocker` etiketini uygulayın. Sorunu başlığınızda bağlantı olarak paylaşın.
5. `main` için `fix(<plugin-id>): beta blocker - <summary>` başlıklı bir PR açın ve sorunu hem PR'da hem de Discord başlığınızda bağlantı olarak paylaşın. Katkıda bulunanlar PR'lara etiket uygulayamaz; bu nedenle başlık, bakımcılar ve otomasyon için PR tarafındaki sinyaldir. PR'ı olan engelleyiciler birleştirilir; olmayan engelleyicilere rağmen sürüm yayımlanabilir.
6. Sessizlik, her şeyin yolunda olduğu anlamına gelir. Bu süreyi kaçırmak, düzeltmenizin genellikle bir sonraki döngüde dahil edilmesi demektir.

## Sonraki adımlar

<CardGroup cols={2}>
  <Card title="Kanal Pluginleri" icon="messages-square" href="/tr/plugins/sdk-channel-plugins">
    Bir mesajlaşma kanalı plugini oluşturun
  </Card>
  <Card title="Sağlayıcı Pluginleri" icon="cpu" href="/tr/plugins/sdk-provider-plugins">
    Bir model sağlayıcı plugini oluşturun
  </Card>
  <Card title="CLI Arka Uç Pluginleri" icon="terminal" href="/tr/plugins/cli-backend-plugins">
    Yerel bir yapay zekâ CLI arka ucu kaydedin
  </Card>
  <Card title="SDK'ya Genel Bakış" icon="book-open" href="/tr/plugins/sdk-overview">
    İçe aktarma eşlemesi ve kayıt API'si başvurusu
  </Card>
  <Card title="Çalışma Zamanı Yardımcıları" icon="settings" href="/tr/plugins/sdk-runtime">
    api.runtime aracılığıyla TTS, arama ve alt aracı
  </Card>
  <Card title="Test Etme" icon="test-tubes" href="/tr/plugins/sdk-testing">
    Test yardımcı araçları ve kalıpları
  </Card>
  <Card title="Plugin Manifesti" icon="file-json" href="/tr/plugins/manifest">
    Tam manifest şeması başvurusu
  </Card>
</CardGroup>

## İlgili

- [Plugin kancaları](/tr/plugins/hooks)
- [Plugin mimarisi](/tr/plugins/architecture)
