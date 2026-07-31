---
read_when:
    - Mesaj kartı, grafik, tablo, düğme veya seçim denetimi oluşturmayı ekleme ya da değiştirme
    - Zengin giden iletileri destekleyen bir kanal Plugin'i oluşturma
    - Mesaj aracı sunumunu veya iletim yeteneklerini değiştirme
    - Sağlayıcıya özgü kart/blok/bileşen işleme regresyonlarında hata ayıklama
summary: Kanal pluginleri için anlamsal mesaj kartları, grafikler, tablolar, denetimler, yedek metin ve teslim ipuçları
title: Mesaj sunumu
x-i18n:
    generated_at: "2026-07-26T23:51:21Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fce3874c99627eb87ceb83aebe381b8a8466722703ec6322c609f187d15d9ae
    source_path: plugins/message-presentation.md
    workflow: 16
---

İleti sunumu, zengin giden sohbet kullanıcı arayüzü için OpenClaw'ın paylaşılan sözleşmesidir.
Aracıların, CLI komutlarının, onay akışlarının ve Plugin'lerin ileti
amacını bir kez tanımlamasını sağlarken her kanal Plugin'i, sunabildiği en iyi yerel biçimi oluşturur.

Taşınabilir ileti kullanıcı arayüzü için sunumu kullanın: metin bölümleri, kısa bağlam/altbilgi
metni, ayırıcılar, grafikler, tablolar, düğmeler, seçim menüleri ve kart başlığı/tonu.

Paylaşılan ileti aracına Discord `components`, Slack
`blocks`, Telegram `buttons`, Teams `card` veya Feishu `card` gibi sağlayıcıya özgü yeni alanlar eklemeyin.
Bunlar, kanal Plugin'inin sahip olduğu işleyici çıktılarıdır.

## Sözleşme

Plugin yazarları herkese açık sözleşmeyi şuradan içe aktarır:

```ts
import type {
  MessagePresentation,
  ReplyPayloadDelivery,
} from "openclaw/plugin-sdk/interactive-runtime";
```

Yapı:

```ts
type MessagePresentation = {
  title?: string;
  tone?: "neutral" | "info" | "success" | "warning" | "danger";
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] }
  | {
      type: "chart";
      chartType: "pie";
      title: string;
      segments: Array<{ label: string; value: number }>;
    }
  | {
      type: "chart";
      chartType: "bar" | "area" | "line";
      title: string;
      categories: string[];
      series: Array<{ name: string; values: number[] }>;
      xLabel?: string;
      yLabel?: string;
    }
  | {
      type: "table";
      caption: string;
      headers: string[];
      rows: Array<Array<string | number>>;
      rowHeaderColumnIndex?: number;
    };

type MessagePresentationAction =
  | { type: "command"; command: string }
  | { type: "callback"; value: string }
  | {
      type: "approval";
      approvalId: string;
      approvalKind: "exec" | "plugin";
      decision: "allow-once" | "allow-always" | "deny";
    }
  | {
      type: "question";
      questionId: string;
      optionValue: string;
    }
  | { type: "url"; url: string }
  | {
      type: "web-app";
      url: string;
      widgetId?: string;
    }
  | {
      type: "web-app";
      url?: string;
      widgetId: string;
    };

type MessagePresentationButton = {
  label: string;
  action?: MessagePresentationAction;
  /** Eski geri çağırma değeri. Yeni denetimler için action tercih edin. */
  value?: string;
  /** @deprecated "url" türünde bir action kullanın. */
  url?: string;
  /** @deprecated "web-app" türünde bir action kullanın. */
  webApp?: { url: string };
  /** @deprecated "web-app" türünde bir action kullanın. */
  web_app?: { url: string };
  priority?: number;
  disabled?: boolean;
  reusable?: boolean;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  action?: Extract<MessagePresentationAction, { type: "command" | "callback" }>;
  /** Eski geri çağırma değeri. Yeni denetimler için action tercih edin. */
  value?: string;
};

type ReplyPayloadDelivery = {
  pin?:
    | boolean
    | {
        enabled: boolean;
        notify?: boolean;
        required?: boolean;
      };
};
```

Düğme semantiği:

- `action.type: "command"`, çekirdeğin komut yolu üzerinden yerel bir eğik çizgi komutu
  çalıştırır. Yerleşik komut düğmeleri ve menüleri için bunu kullanın.
- `action.type: "callback"`, belirsiz Plugin verilerini kanalın
  etkileşim yolu üzerinden taşır. Kanal Plugin'leri, geri çağırma verilerini eğik çizgi
  komutları olarak yeniden yorumlamamalıdır.
- `action.type: "approval"`, tek bir kalıcı operatör onayını, açık
  `exec` veya `plugin` türünü ve istenen kararı tanımlar. Kanal Plugin'leri
  bu eylemi aktarıma özel bir geri çağırmaya kodlayıp onay hizmeti üzerinden
  çözümler; `/approve` komut metnini ayrıştırmamalı veya
  kimlikten tür çıkarımı yapmamalıdır.
- `action.type: "question"`, çalışma zamanında oluşturulan canlı bir
  `ask_user` sorusunun tek bir seçeneğini tanımlar. `approval` gibi bu da bir OpenClaw çalışma zamanı eylemidir;
  aracılar ve Plugin'ler soru kimlikleri oluşturmamalıdır. Telegram, Discord ve
  Slack bunu aktarıma özel yerel geri çağırmalara eşler ve seçimi
  Gateway üzerinden çözümler. Soru yanıtlandığında, süresi dolduğunda veya
  iptal edildiğinde bu kanallar teslim edilen iletiyi düzenler, eylemlerini kaldırır
  ve son durumu ekler. WhatsApp, Signal ve iMessage, en fazla
  dört tek seçimli seçeneği `1️⃣` ile `4️⃣` tepkileri olarak oluşturur. Diğer soru
  biçimleri etiket metnine indirgenir ve kullanıcı düz metinli
  yanıt verebilir.
- `action.type: "url"`, normal bir bağlantı açar.
- `action.type: "web-app"`, kanala özgü yerel bir web uygulaması başlatır. URL destekli bir
  uygulama için `url`, başlatma mekanizması kanalın
  sorumluluğunda olan OpenClaw barındırmalı bir bileşen için `widgetId` ayarlayın; en az biri gereklidir. Her ikisi de
  bulunduğunda kanal, kendi yerel barındırılan bileşen başlatma mekanizmasını tercih edebilir ve bu mekanizmanın
  kullanılamadığı yerde URL'yi kullanabilir.
- `value`, eski belirsiz geri çağırma değeridir. Yeni denetimler `action`
  kullanmalıdır; böylece kanal Plugin'leri metinden tahmin yürütmeden komutları ve geri çağırmaları eşleyebilir.
- `url`, `webApp` ve `web_app`, kullanımdan kaldırılmış sınır girdileri olarak kabul edilmeye devam eder.
  Normalleştiriciler bu alanları korur; böylece işleyiciler yayımlanmış eski
  semantiği açıkça türü belirtilmiş eylemlerden ayırt edebilir. Yeni üreticiler `action` kullanmalıdır.
- `label` gereklidir ve metin geri dönüşünde de kullanılır.
- `style` tavsiye niteliğindedir. İşleyiciler desteklenmeyen stilleri güvenli bir
  varsayılana eşlemeli, gönderimi başarısız kılmamalıdır.
- `priority` isteğe bağlıdır. Bir kanal eylem sınırlarını bildirdiğinde ve denetimlerin
  kaldırılması gerektiğinde çekirdek, daha yüksek öncelikli düğmeleri önce tutar ve
  eşit öncelikli düğmelerin özgün sırasını korur. Tüm denetimler sığdığında, oluşturuldukları
  sıra korunur.
- `disabled` isteğe bağlıdır. Kanallar `supportsDisabled` ile açıkça katılmalıdır; aksi takdirde
  çekirdek, devre dışı bırakılan denetimi etkileşimsiz geri dönüş metnine indirger. Devre dışı
  bir düğme, `command` eylemi taşısa bile geri dönüş metninde her zaman yalnızca etiket olarak oluşturulur.
- `reusable` isteğe bağlıdır. Yeniden kullanılabilir yerel geri çağırmaları destekleyen kanallar,
  başarılı bir etkileşimden sonra eylemi kullanılabilir tutabilir. Yenileme, inceleme veya daha fazla ayrıntı
  gibi yinelenebilir ya da eşgüçlü eylemler için kullanın;
  normal tek seferlik onaylar ve yıkıcı eylemler için ayarlamayın.

Seçim semantiği:

- `options[].action` yalnızca `command` veya `callback` kabul eder; onay ve bağlantı eylemleri yalnızca düğmelerde kullanılabilir.
- `options[].value`, seçilen eski uygulama değeridir.
- `placeholder` tavsiye niteliğindedir ve yerel seçim desteği olmayan kanallar tarafından
  yok sayılabilir.
- Bir kanal seçimleri desteklemiyorsa geri dönüş metni etiketleri listeler.

Grafik semantiği:

- `pie`, pozitif dilim değerleri gerektirir.
- `bar`, `area` ve `line`, sıralı tek bir `categories` dizisi kullanır. Her seri,
  aynı sırayla her kategori için tam olarak bir sonlu değer sağlar.
- Kategori etiketleri ve seri adları benzersiz olmalıdır. Geçersiz veya eksik grafik
  blokları, verileri sessizce değiştirmek yerine normalleştirme sırasında kaldırılır.
- Yerel grafik oluşturma, `presentationCapabilities.charts` üzerinden isteğe bağlı olarak etkinleştirilir.
  Diğer kanallar grafik başlığını, eksenleri, kategorileri, serileri ve değerleri
  belirlenimci metin olarak alır. Bu aynı zamanda erişilebilirlik geri dönüşüdür.

Tablo semantiği:

- `caption`, gerekli olan kısa bir başlıktır. `headers` en az bir
  benzersiz ve boş olmayan sütun etiketi içermelidir.
- `rows` en az bir satır içermelidir. Her satırda her başlık için tam olarak bir hücre
  bulunmalı ve her hücre boş olmayan bir dize veya sonlu bir sayı olmalıdır.
- `rowHeaderColumnIndex`, hücreleri yerel işleyiciler tarafından satır başlığı olarak
  sunulması gereken sütunu tanımlayan, sıfır tabanlı isteğe bağlı bir dizindir.
- Tablo normalleştirmesi atomiktir. Geçersiz bir açıklama, başlık, satır genişliği, hücre
  veya satır başlığı dizini, verileri kesmek ya da onarmak yerine tablo bloğunun
  kaldırılmasına neden olur.
- Yerel tablo oluşturma, `presentationCapabilities.tables` üzerinden isteğe bağlı olarak etkinleştirilir.
  Diğer kanallar açıklamayı ve her satırı, iç boşlukları daraltılmış
  belirlenimci doğrusal metin olarak alır:

  ```text
  Açık işlem hattı (tablo)
  - Hesap: Acme; Aşama: Kazanıldı; ARR: 125000
  - Hesap: Globex; Aşama: İnceleme; ARR: 82000
  ```

Ayrı bir `report` ayırt edicisi yoktur. `title`,
`tone`, `text`, `context`, `chart`, `table` ve eylem bloklarından bir rapor oluşturun. Böylece her
blok bağımsız olarak oluşturulabilir ve tam rapor aynı
belirlenimci metin geri dönüşüne sahip olur.

## Üretici örnekleri

Basit kart:

```json
{
  "title": "Dağıtım onayı",
  "tone": "warning",
  "blocks": [
    { "type": "text", "text": "Canary yükseltilmeye hazır." },
    { "type": "context", "text": "Derleme 1234, hazırlık ortamını geçti." },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Onayla",
          "action": { "type": "callback", "value": "deploy:approve" },
          "style": "success"
        },
        {
          "label": "Reddet",
          "action": { "type": "callback", "value": "deploy:decline" },
          "style": "danger"
        }
      ]
    }
  ]
}
```

Yalnızca URL içeren bağlantı düğmesi:

```json
{
  "blocks": [
    { "type": "text", "text": "Sürüm notları hazır." },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Notları aç",
          "action": { "type": "url", "url": "https://example.com/release" }
        }
      ]
    }
  ]
}
```

Telegram Mini App düğmesi:

```json
{
  "blocks": [
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "Başlat",
          "action": { "type": "web-app", "url": "https://example.com/app" }
        }
      ]
    }
  ]
}
```

Seçim menüsü:

```json
{
  "title": "Ortam seçin",
  "blocks": [
    {
      "type": "select",
      "placeholder": "Ortam",
      "options": [
        { "label": "Canary", "value": "env:canary" },
        { "label": "Üretim", "value": "env:prod" }
      ]
    }
  ]
}
```

Grafik:

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "line",
      "title": "Üç aylık gelir",
      "categories": ["1. Çeyrek", "2. Çeyrek", "3. Çeyrek"],
      "series": [
        { "name": "Ürün", "values": [120, 145, 138] },
        { "name": "Hizmetler", "values": [80, 95, 104] }
      ],
      "xLabel": "Çeyrek",
      "yLabel": "Gelir"
    }
  ]
}
```

Tablo raporu:

```json
{
  "title": "İşlem hattı raporu",
  "tone": "info",
  "blocks": [
    { "type": "text", "text": "Aşamaya göre güncel fırsatlar." },
    {
      "type": "table",
      "caption": "Açık işlem hattı",
      "headers": ["Hesap", "Aşama", "ARR"],
      "rows": [
        ["Acme", "Kazanıldı", 125000],
        ["Globex", "İnceleme", 82000]
      ],
      "rowHeaderColumnIndex": 0
    },
    { "type": "context", "text": "CRM anlık görüntüsünden güncellendi." }
  ]
}
```

CLI ile gönderme:

```bash
openclaw message send --channel slack \
  --target channel:C123 \
  --message "Dağıtım onayı" \
  --presentation '{"title":"Dağıtım onayı","tone":"warning","blocks":[{"type":"text","text":"Canary hazır."},{"type":"buttons","buttons":[{"label":"Onayla","value":"deploy:approve","style":"success"},{"label":"Reddet","value":"deploy:decline","style":"danger"}]}]}'
```

Sabitlenmiş teslimat:

```bash
openclaw message send --channel telegram \
  --target -1001234567890 \
  --message "Konu açıldı" \
  --pin
```

Açık JSON ile sabitlenmiş teslim:

```json
{
  "pin": {
    "enabled": true,
    "notify": true,
    "required": false
  }
}
```

## İşleyici sözleşmesi

Kanal pluginleri, giden bağdaştırıcılarında işleme desteğini bildirir:

```ts
const adapter: ChannelOutboundAdapter = {
  deliveryMode: "direct",
  presentationCapabilities: {
    supported: true,
    buttons: true,
    selects: true,
    context: true,
    divider: true,
    charts: false,
    tables: false,
    limits: {
      actions: {
        maxActions: 25,
        maxActionsPerRow: 5,
        maxRows: 5,
        maxLabelLength: 80,
        maxValueBytes: 100,
        supportsStyles: true,
        supportsDisabled: false,
      },
      selects: {
        maxOptions: 25,
        maxLabelLength: 100,
        maxValueBytes: 100,
      },
      text: {
        maxLength: 2000,
        encoding: "characters",
        markdownDialect: "discord-markdown",
      },
    },
  },
  deliveryCapabilities: {
    pin: true,
  },
  renderPresentation({ payload, presentation, ctx }) {
    return renderNativePayload(payload, presentation, ctx);
  },
  async pinDeliveredMessage({ target, messageId, pin }) {
    await pinNativeMessage(target, messageId, { notify: pin.notify === true });
  },
};
```

Yetenek boole değerleri, işleyicinin neleri etkileşimli hâle getirebildiğini açıklar. İsteğe bağlı
`limits`, çekirdeğin işleyiciyi çağırmadan önce uyarlayabileceği genel zarfı açıklar:

```ts
type ChannelPresentationCapabilities = {
  supported?: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  charts?: boolean;
  tables?: boolean;
  limits?: {
    actions?: {
      maxActions?: number;
      maxActionsPerRow?: number;
      maxRows?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
      supportsStyles?: boolean;
      supportsDisabled?: boolean;
      supportsLayoutHints?: boolean;
    };
    selects?: {
      maxOptions?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
    };
    text?: {
      maxLength?: number;
      encoding?: "characters" | "utf8-bytes" | "utf16-units";
      markdownDialect?: "plain" | "markdown" | "html" | "slack-mrkdwn" | "discord-markdown";
      supportsEdit?: boolean;
    };
  };
};
```

Çekirdek, işlemeden önce anlamsal denetimlere genel sınırları uygular. İşleyiciler;
yerel blok sayısı, kart boyutu, URL sınırları ve genel sözleşmede ifade edilemeyen
sağlayıcıya özgü özellikler için son doğrulama ve kırpma işlemlerinin sahibi olmaya
devam eder. Sınırlar bir bloktaki tüm denetimleri kaldırırsa çekirdek, teslim edilen
iletide görünür bir geri dönüş seçeneği kalması için etiketleri etkileşimsiz bağlam
metni olarak korur.

## Çekirdek işleme akışı

CLI ve standart ileti eylemlerinin kullandığı kurallı giden yolda çekirdek:

1. Sunum yükünü normalleştirir.
2. Hedef kanalın giden bağdaştırıcısını çözümler.
3. `presentationCapabilities` değerini okur.
4. Bağdaştırıcı bunları bildirdiğinde eylem sayısı, etiket uzunluğu ve
   seçim seçeneği sayısı gibi genel yetenek sınırlarını uygular. Bağdaştırıcı
   sırasıyla `charts: true` veya `tables: true` desteğini açıkça bildirmediği
   sürece grafik ve tablo blokları deterministik metne dönüşür.
5. Bağdaştırıcı yükü işleyebildiğinde `renderPresentation` çağrısını yapar.
6. Bağdaştırıcı yoksa veya işleyemiyorsa korumacı metne geri döner.
7. Ortaya çıkan yükü normal kanal teslim yolu üzerinden gönderir.
8. İlk ileti başarıyla gönderildikten sonra `delivery.pin` gibi teslim
   meta verilerini uygular.

`ReplyPayload` değerini doğrudan tüketen kanala özgü yanıt veya önizleme
hunileri, kurallı yola girmeli ya da yükü düz metne/ortama indirgemeden önce aynı
sunum geri dönüşünü somutlaştırmalıdır.

Çekirdek, üreticilerin kanaldan bağımsız kalabilmesi için geri dönüş davranışının
sahibidir. Kanal pluginleri yerel işleme ve etkileşim yönetiminin sahibidir.

## Seviyesi düşürme kuralları

Sunum, sınırlı kanallarda güvenle gönderilebilmelidir.

Geri dönüş metni şunları içerir:

- ilk satır olarak `title`
- normal paragraflar olarak `text` blokları
- kısa bağlam satırları olarak `context` blokları
- görsel ayırıcı olarak `divider` blokları
- bağlantı düğmelerinin URL'leri dâhil olmak üzere düğme etiketleri
- seçim seçeneği etiketleri
- grafik başlığı, türü, eksenleri, kategorileri, serileri ve değerleri
- tablo açıklaması, başlıkları ve her satır değeri

### Düğme değeri geri dönüşünün görünürlüğü

Bir kanal etkileşimli denetimleri işleyemediğinde düğme ve seçim değerleri düz
metne geri döner. Geri dönüş davranışı, anlaşılmaz geri çağırma verilerini gizli
tutarken kullanılabilirliği korur:

- **`command` türündeki eylemler** `` label: `command` `` olarak işlenir;
  böylece kullanıcılar komutu kopyalayıp kanal girişinde elle çalıştırabilir.
- **`callback` türündeki eylemler** ve eski **`value`**
  alanları yalnızca etiket olarak işlenir. Anlaşılmaz geri çağırma değeri geri dönüş
  metninde gösterilmez.
- **`approval` türündeki eylemler** yalnızca etiket olarak işlenir.
  Onay kimlikleri ve kararlar taşıma verisidir; genel skaler yardımcılar veya geri
  dönüş metni üzerinden gösterilmez.
- **`url` eylemleri**, URL destekli **`web-app`
  eylemleri** ve kullanım dışı **`url` / `webApp` /
  `web_app`** girdileri, URL kullanıcıya yönelik olduğu için URL metnini
  düğme etiketiyle birlikte işler. Yalnızca barındırılan pencere öğesine özgü
  eylemler, yerel pencere öğesi başlatma desteği bulunmayan kanallarda yalnızca
  etiket olarak işlenir.
- **Seçim seçenekleri** yalnızca etiket olarak işlenir. Temel seçenek
  değeri geri dönüş metninde gösterilmez.

Geri dönüş kullanıcı arayüzlerine elle komut kullanma yönlendirmesi ekleyen kanal
bağdaştırıcıları (ör. Feishu belge yorumu talimatları), komut varlığı denetimini
geri dönüş işleyicisinin kullandığı sunum bloklarından türetmelidir; böylece
yönlendirme metni yalnızca gerçekten bir elle çalıştırılan komut gösterildiğinde
görünür.

Desteklenmeyen yerel denetimler, gönderimin tamamının başarısız olmasına neden
olmak yerine seviyesini düşürmelidir. Örnekler:

- Satır içi düğmeleri devre dışı olan Telegram metin geri dönüşü gönderir.
- Seçim desteği olmayan bir kanal, seçim seçeneklerini metin olarak listeler.
- Yerel grafik desteği olmayan bir kanal, grafik verilerini metin olarak listeler.
- Yerel tablo desteği olmayan bir kanal, her tablo satırını metin olarak listeler.
- Yalnızca URL içeren bir düğme, yerel bağlantı düğmesine veya geri dönüş URL satırına dönüşür.
- İsteğe bağlı sabitleme hataları, teslim edilen iletinin başarısız olmasına neden olmaz.

Temel istisna `delivery.pin.required: true` değeridir; sabitleme zorunlu olarak istenir ve
kanal gönderilen iletiyi sabitleyemezse teslim başarısızlık bildirir.

## Sağlayıcı eşlemesi

Mevcut paketlenmiş işleyiciler:

| Kanal           | Yerel işleme hedefi                      | Notlar                                                                                                                                                                                                             |
| --------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discord         | Bileşenler ve bileşen kapsayıcıları       | Mevcut sağlayıcıya özgü yük üreticileri için eski `channelData.discord.components` değerini korur, ancak yeni paylaşımlı gönderimler `presentation` kullanmalıdır.                                                                 |
| Feishu          | Etkileşimli kartlar                        | Kart başlığı `title` kullanabilir; gövde bu başlığı yinelemekten kaçınır.                                                                                                                                                  |
| Matrix          | Metin geri dönüşü ve yapılandırılmış olay alanı | Düğmeler/seçimler destekleniyor olarak bildirilir, ancak her blok şu anda yerel etkileşimli pencere öğeleri olarak değil, bir `com.openclaw.presentation` olay alanında taşınan `renderMessagePresentationFallbackText` çıktısı olarak işlenir. |
| Mattermost      | Metin ve etkileşimli özellikler            | Seçimler ve ayırıcılar desteklenmez; bu bloklar metne düşürülür.                                                                                                                                             |
| Microsoft Teams | Adaptive Cards                             | Her ikisi de sağlandığında düz `message` metni kartla birlikte eklenir. Seçimler, stiller ve devre dışı durumu desteklenmez.                                                                                     |
| Slack           | Block Kit                                  | `chart` değerini yerel `data_visualization`, `table` değerini yerel `data_table` olarak işler; eski `channelData.slack.blocks` değerini korur, ancak yeni paylaşımlı gönderimler `presentation` kullanmalıdır.                                   |
| Telegram        | Metin ve satır içi klavyeler               | Düğmeler/seçimler hedef yüzey için satır içi düğme yeteneği gerektirir; aksi takdirde metin geri dönüşü kullanılır.                                                                                                         |
| Düz kanallar    | Metin geri dönüşü                          | İşleyicisi olmayan kanallar da okunabilir çıktı alır.                                                                                                                                                            |

Sağlayıcıya özgü yerel yük uyumluluğu, mevcut yanıt üreticileri için bir geçiş
kolaylığıdır. Yeni paylaşımlı yerel alanlar eklemek için gerekçe değildir.

## Sunum ve InteractiveReply karşılaştırması

`InteractiveReply`, onay ve etkileşim yardımcılarının kullandığı eski dâhilî alt
kümedir. Şunları destekler:

- metin
- düğmeler
- seçimler

`MessagePresentation`, kurallı paylaşımlı gönderim sözleşmesidir. Şunları ekler:

- başlık
- ton
- bağlam
- ayırıcı
- grafik
- tablo
- yalnızca URL içeren düğmeler
- `ReplyPayload.delivery` üzerinden genel teslim meta verileri

Eski kod arasında köprü kurarken `openclaw/plugin-sdk/interactive-runtime` yardımcılarını kullanın:

```ts
import {
  adaptMessagePresentationForChannel,
  applyPresentationActionLimits,
  hasMessagePresentationBlocks,
  interactiveReplyToPresentation,
  isMessagePresentationInteractiveBlock,
  normalizeMessagePresentation,
  presentationPageSize,
  presentationToInteractiveControlsReply,
  presentationToInteractiveReply,
  renderMessagePresentationChartFallbackText,
  renderMessagePresentationFallbackText,
  renderMessagePresentationTableFallbackText,
  resolveMessagePresentationActionValue,
  resolveMessagePresentationButtonAction,
  resolveMessagePresentationControlValue,
  resolveMessagePresentationOptionAction,
} from "openclaw/plugin-sdk/interactive-runtime";
```

Yeni kod doğrudan `MessagePresentation` kabul etmeli veya üretmelidir. Mevcut
`interactive` yükleri, `presentation` değerinin kullanım dışı bir alt
kümesidir; eski üreticiler için çalışma zamanı desteği devam eder.

Bilinmesi yararlı, kullanım dışı olmayan yardımcılar:

- `normalizeMessagePresentation(raw)` / `hasMessagePresentationBlocks(value)`
  türsüz bir yükü (örneğin, CLI
  `--presentation` bayrağından gelen JSON'u) doğrular ve `MessagePresentation` türüne dönüştürür.
- `isMessagePresentationInteractiveBlock(block)`, bir bloğun türünü
  `buttons` | `select` birleşimine daraltır.
- `resolveMessagePresentationButtonAction(button)` ve
  `resolveMessagePresentationOptionAction(option)`, kullanımdan kaldırılmış sınır alanlarını kabul ederken kurallı, türü belirlenmiş
  eylemi döndürür. Açıkça belirtilmiş bir `action`
  her zaman önceliklidir.
- `resolveMessagePresentationActionValue(action)` /
  `resolveMessagePresentationControlValue(control)` yalnızca komut/geri çağırma
  skaler değerlerini okur. Skaler olmayan kurallı bir eylem hiçbir zaman
  eski bir gölge `value` alanına düşmez; böylece onay kimlikleri ve bağlantı hedefleri türlerini korur.
- `renderMessagePresentationChartFallbackText(block)` /
  `renderMessagePresentationTableFallbackText(block)`, kanala özgü geri dönüş yolları için yapılandırılmış tek bir
  veri bloğunu belirlenimci metin olarak işler.

Eski `InteractiveReply*` türleri ve dönüştürme yardımcıları SDK'da
`@deprecated` olarak işaretlenmiştir:

- `InteractiveReply`, `InteractiveReplyBlock`, `InteractiveReplyButton` ve
  `InteractiveReplyOption`
- `normalizeInteractiveReply(...)`
- `hasInteractiveReplyBlocks(...)`
- `interactiveReplyToPresentation(...)`
- `presentationToInteractiveReply(...)`
- `presentationToInteractiveControlsReply(...)`
- `resolveInteractiveTextFallback(...)`
- `reduceInteractiveReply(...)`

`presentationToInteractiveReply(...)` ve
`presentationToInteractiveControlsReply(...)`, eski kanal uygulamaları için işleyici
köprüleri olarak kullanılabilir durumda kalır. Yeni üretici kodu bunları çağırmamalıdır;
`presentation` göndermeli ve işlemeyi çekirdek/kanal uyarlamasına bırakmalıdır.

Onay yardımcılarının da sunum öncelikli karşılıkları vardır:

- `buildApprovalInteractiveReply(...)` yerine
  `buildApprovalPresentation(...)` kullanın
- `buildExecApprovalInteractiveReply(...)` yerine
  `buildExecApprovalPresentation(...)` kullanın

Yayımlanmış bu oluşturucular, plugin uyumluluğu için komut tabanlı kalır. Kalıcı bir onay türünün sahibi olan Gateway
ve paketle birlikte gelen kanal kodu,
taşıma katmanlarının anlamı `/approve` metninden çıkarsamak yerine
açık bir `approval` eylemi alması için
`buildTypedApprovalPresentation(...)`,
`buildTypedExecApprovalPendingReplyPayload(...)` veya
`buildTypedPluginApprovalPendingReplyPayload(...)` kullanmalıdır.

`renderMessagePresentationFallbackText(...)`, yalnızca ayırıcıdan oluşan bir
sunum gibi metinsel geri dönüşü olmayan sunum blokları için boş dize döndürür.
Boş olmayan bir gönderim gövdesi gerektiren taşıma katmanları, varsayılan geri dönüş
sözleşmesini değiştirmeden asgari bir gövdeyi etkinleştirmek için `emptyFallback` iletebilir.

## Teslimatta sabitleme

Sabitleme, sunum değil teslimat davranışıdır. `channelData.telegram.pin` gibi
sağlayıcıya özgü alanlar yerine `delivery.pin` kullanın.

Anlamsal kurallar:

- `pin: true`, başarıyla teslim edilen ilk mesajı sabitler.
- `pin.notify` varsayılan olarak `false` değerini alır.
- `pin.required` varsayılan olarak `false` değerini alır.
- İsteğe bağlı sabitleme hatalarında işlevsellik azaltılır ve gönderilen mesaj olduğu gibi kalır.
- Zorunlu sabitleme hatalarında teslimat başarısız olur.
- Parçalı mesajlarda son parça değil, teslim edilen ilk parça sabitlenir.

Sağlayıcının bu işlemleri desteklediği mevcut
mesajlar için elle uygulanan `pin`, `unpin` ve `pins` mesaj eylemleri hâlâ mevcuttur.

## Plugin yazarı kontrol listesi

- Kanal, anlamsal sunumu işleyebildiğinde veya güvenli biçimde indirgeyebildiğinde
  `describeMessageTool(...)` üzerinden `presentation` bildirin.
- Çalışma zamanı giden bağdaştırıcısına `presentationCapabilities` ekleyin.
- `renderPresentation` öğesini kontrol düzlemi plugin
  kurulum kodunda değil, çalışma zamanı kodunda uygulayın.
- Yerel kullanıcı arayüzü kitaplıklarını sık kullanılan kurulum/katalog yollarından uzak tutun.
- Biliniyorlarsa genel yetenek sınırlarını `presentationCapabilities.limits` üzerinde
  bildirin.
- İşleyicide ve testlerde nihai platform sınırlarını koruyun.
- Desteklenmeyen grafikler, tablolar, düğmeler, seçimler, URL
  düğmeleri, başlık/metin yinelemesi ve karma `message` ile `presentation`
  gönderimleri için geri dönüş testleri ekleyin.
- Yalnızca sağlayıcı gönderilen mesaj kimliğini sabitleyebiliyorsa `deliveryCapabilities.pin` ve
  `pinDeliveredMessage` üzerinden teslimatta sabitleme desteği ekleyin.
- Paylaşılan mesaj eylemi şeması üzerinden sağlayıcıya özgü yeni kart/blok/bileşen/düğme
  alanlarını kullanıma sunmayın.

## İlgili belgeler

- [Mesaj CLI'si](/tr/cli/message)
- [Plugin SDK'ya Genel Bakış](/tr/plugins/sdk-overview)
- [Plugin Mimarisi](/tr/plugins/architecture-internals#message-tool-schemas)
- [Kanal Sunumu Yeniden Düzenleme Planı](/tr/plan/ui-channels)
