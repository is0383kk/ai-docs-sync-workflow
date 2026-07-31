---
read_when:
    - Kanal mesajı kullanıcı arayüzünü, etkileşimli yükleri veya yerel kanal oluşturucularını yeniden düzenleme
    - Mesaj aracı yeteneklerini, teslim ipuçlarını veya bağlamlar arası işaretçileri değiştirme
    - Discord Carbon içe aktarma yayılımında veya kanal Plugin çalışma zamanı tembelliğinde hata ayıklama
summary: Anlamsal ileti sunumunu kanala özgü yerel kullanıcı arayüzü işleyicilerinden ayırın.
title: Kanal sunumu yeniden düzenleme planı
x-i18n:
    generated_at: "2026-07-26T23:46:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0f0c4f64e0c503209ac0a5b763b1b5483bf8d55a28ceacffbbcd1337d4371e
    source_path: plan/ui-channels.md
    workflow: 16
---

## Durum

Paylaşılan ajan, CLI, plugin yeteneği ve giden teslimat yüzeyleri için uygulandı:

- `ReplyPayload.presentation` anlamsal mesaj kullanıcı arayüzünü taşır.
- `ReplyPayload.delivery.pin` gönderilmiş mesaj sabitleme isteklerini taşır.
- Paylaşılan mesaj eylemleri, sağlayıcıya özgü `components`, `blocks`, `buttons` veya `card` yerine `presentation`, `delivery` ve `pin` sunar.
- Çekirdek, sunumu plugin tarafından bildirilen giden yetenekler aracılığıyla işler veya otomatik olarak daha basit bir biçime dönüştürür.
- Discord, Slack, Telegram, Mattermost, MS Teams ve Feishu işleyicileri genel sözleşmeyi kullanır.
- Discord kanalının kontrol düzlemi kodu artık Carbon tabanlı kullanıcı arayüzü kapsayıcılarını içe aktarmaz.

Standart belgeler artık [Mesaj Sunumu](/tr/plugins/message-presentation) bölümünde yer alır.
Bu planı geçmiş uygulama bağlamı olarak saklayın; sözleşme, işleyici veya
geri dönüş davranışı değişikliklerinde standart kılavuzu güncelleyin.

## Sorun

Kanal kullanıcı arayüzü şu anda birbiriyle uyumsuz birkaç yüzeye bölünmüştür:

- Çekirdek, `buildCrossContextComponents` aracılığıyla Discord biçimli, bağlamlar arası bir işleyici kancasına sahiptir.
- Discord `channel.ts`, `DiscordUiContainer` aracılığıyla yerel Carbon kullanıcı arayüzünü içe aktarabilir; bu da çalışma zamanı kullanıcı arayüzü bağımlılıklarını kanal plugininin kontrol düzlemine çeker.
- Ajan ve CLI; Discord `components`, Slack `blocks`, Telegram veya Mattermost `buttons` ve Teams veya Feishu `card` gibi yerel yük kaçış yolları sunar.
- `ReplyPayload.channelData` hem aktarım ipuçlarını hem de yerel kullanıcı arayüzü zarflarını taşır.
- Genel `interactive` modeli mevcuttur ancak Discord, Slack, Teams, Feishu, LINE, Telegram ve Mattermost tarafından hâlihazırda kullanılan daha zengin düzenlerden daha sınırlıdır.

Bu durum çekirdeği yerel kullanıcı arayüzü biçimlerinden haberdar eder, plugin çalışma zamanının tembel yüklenmesini zayıflatır ve ajanlara aynı mesaj amacını ifade etmek için sağlayıcıya özgü gereğinden fazla yol sunar.

## Hedefler

- Çekirdek, bildirilen yeteneklere göre bir mesaj için en iyi anlamsal sunuma karar verir.
- Uzantılar yeteneklerini bildirir ve anlamsal sunumu yerel aktarım yüklerine dönüştürür.
- Web Denetim Kullanıcı Arayüzü, sohbetin yerel kullanıcı arayüzünden ayrı kalır.
- Yerel kanal yükleri, paylaşılan ajan veya CLI mesaj yüzeyi üzerinden kullanıma sunulmaz.
- Desteklenmeyen sunum özellikleri otomatik olarak en uygun metin gösterimine indirgenir.
- Gönderilmiş bir mesajı sabitleme gibi teslimat davranışları sunum değil, genel teslimat meta verileridir.

## Hedef olmayanlar

- `buildCrossContextComponents` için geriye dönük uyumluluk katmanı yoktur.
- `components`, `blocks`, `buttons` veya `card` için herkese açık yerel kaçış yolları yoktur.
- Çekirdek, kanala özgü kullanıcı arayüzü kitaplıklarını içe aktarmaz.
- Paketlenmiş kanallar için sağlayıcıya özgü SDK bağlantı noktaları yoktur.

## Hedef model

`ReplyPayload` öğesine çekirdeğin sahip olduğu bir `presentation` alanı ekleyin.

```ts
type MessagePresentationTone = "neutral" | "info" | "success" | "warning" | "danger";

type MessagePresentation = {
  tone?: MessagePresentationTone;
  title?: string;
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] };

type MessagePresentationButton = {
  label: string;
  value?: string;
  url?: string;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  value: string;
};
```

Geçiş sırasında `interactive`, `presentation` öğesinin bir alt kümesi hâline gelir:

- `interactive` metin bloğu `presentation.blocks[].type = "text"` öğesine eşlenir.
- `interactive` düğmeler bloğu `presentation.blocks[].type = "buttons"` öğesine eşlenir.
- `interactive` seçim bloğu `presentation.blocks[].type = "select"` öğesine eşlenir.

Haricî ajan ve CLI şemaları artık `presentation` kullanır; `interactive`, mevcut yanıt üreticileri için dahili ve eski bir ayrıştırma/işleme yardımcısı olarak kalır.
Herkese açık, üreticilere yönelik API, `interactive` öğesini kullanımdan kaldırılmış kabul eder. Yeni kod `presentation` üretirken mevcut onay yardımcılarının ve eski pluginlerin çalışmaya devam etmesi için çalışma zamanı
desteği korunur.

## Teslimat meta verileri

Kullanıcı arayüzüyle ilgili olmayan gönderim davranışı için çekirdeğin sahip olduğu bir `delivery` alanı ekleyin.

```ts
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

Anlamlar:

- `delivery.pin = true`, başarıyla teslim edilen ilk mesajı sabitlemek anlamına gelir.
- `notify` varsayılan olarak `false` değerini alır.
- `required` varsayılan olarak `false` değerini alır; desteklenmeyen kanallar veya başarısız sabitleme işlemleri teslimata devam edilerek otomatik olarak daha basit bir davranışa indirgenir.
- Mevcut mesajlar için elle kullanılan `pin`, `unpin` ve `list-pins` mesaj eylemleri korunur.

Mevcut Telegram ACP konu bağlama işlemi `channelData.telegram.pin = true` öğesinden `delivery.pin = true` öğesine taşınmalıdır.

## Çalışma zamanı yetenek sözleşmesi

Sunum ve teslimat işleme kancalarını kontrol düzlemi kanal pluginine değil, çalışma zamanı giden bağdaştırıcısına ekleyin.

```ts
type ChannelPresentationCapabilities = {
  supported: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  tones?: MessagePresentationTone[];
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

type ChannelDeliveryCapabilities = {
  pinSentMessage?: boolean;
};

type ChannelOutboundAdapter = {
  presentationCapabilities?: ChannelPresentationCapabilities;

  renderPresentation?: (params: {
    payload: ReplyPayload;
    presentation: MessagePresentation;
    ctx: ChannelOutboundSendContext;
  }) => ReplyPayload | null;

  deliveryCapabilities?: ChannelDeliveryCapabilities;

  pinDeliveredMessage?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    to: string;
    threadId?: string | number | null;
    messageId: string;
    notify: boolean;
  }) => Promise<void>;
};
```

Çekirdek davranışı:

- Hedef kanalı ve çalışma zamanı bağdaştırıcısını çözümleyin.
- Sunum yeteneklerini sorgulayın.
- İşlemeden önce desteklenmeyen blokları daha basit biçimlere dönüştürün ve genel yetenek sınırlarını
  uygulayın.
- `renderPresentation` çağrısını yapın.
- İşleyici yoksa sunumu yedek metne dönüştürün.
- Başarılı gönderimden sonra `delivery.pin` istenmiş ve destekleniyorsa `pinDeliveredMessage` çağrısını yapın.

## Kanal eşlemesi

Discord:

- `presentation` öğesini yalnızca çalışma zamanı modüllerinde v2 bileşenlerine ve Carbon kapsayıcılarına dönüştürün.
- Vurgu rengi yardımcılarını hafif modüllerde tutun.
- Kanal plugininin kontrol düzlemi kodundan `DiscordUiContainer` içe aktarımlarını kaldırın.

Slack:

- `presentation` öğesini Block Kit'e dönüştürün.
- Ajan ve CLI `blocks` girdisini kaldırın.

Telegram:

- Metni, bağlamı ve ayırıcıları metin olarak işleyin.
- Yapılandırıldığında ve hedef yüzey için izin verildiğinde eylemleri ve seçimi satır içi klavyeler olarak işleyin.
- Satır içi düğmeler devre dışı bırakıldığında yedek metni kullanın.
- ACP konu sabitlemesini `delivery.pin` öğesine taşıyın.

Mattermost:

- Yapılandırıldığı yerlerde eylemleri etkileşimli düğmeler olarak işleyin.
- Diğer blokları yedek metin olarak işleyin.

MS Teams:

- `presentation` öğesini Adaptive Cards biçimine dönüştürün.
- Elle kullanılan sabitleme, sabitlemeyi kaldırma ve sabitlenmiş öğeleri listeleme eylemlerini koruyun.
- Graph desteği hedef konuşma için güvenilir durumdaysa isteğe bağlı olarak `pinDeliveredMessage` öğesini uygulayın.

Feishu:

- `presentation` öğesini etkileşimli kartlara dönüştürün.
- Elle kullanılan sabitleme, sabitlemeyi kaldırma ve sabitlenmiş öğeleri listeleme eylemlerini koruyun.
- API davranışı güvenilir durumdaysa gönderilmiş mesajları sabitlemek için isteğe bağlı olarak `pinDeliveredMessage` öğesini uygulayın.

LINE:

- Mümkün olduğunda `presentation` öğesini Flex veya şablon mesajlara dönüştürün.
- Desteklenmeyen bloklar için yedek metne dönün.
- LINE kullanıcı arayüzü yüklerini `channelData` öğesinden kaldırın.

Düz veya sınırlı kanallar:

- Sunumu ölçülü biçimlendirmeyle metne dönüştürün.

## Yeniden düzenleme adımları

1. `ui-colors.ts` öğesini Carbon tabanlı kullanıcı arayüzünden ayıran ve `extensions/discord/src/channel.ts` içinden `DiscordUiContainer` öğesini kaldıran Discord sürüm düzeltmesini yeniden uygulayın.
2. `ReplyPayload`, giden yük normalleştirmesi, teslimat özetleri ve kanca yüklerine `presentation` ile `delivery` ekleyin.
3. Dar kapsamlı bir SDK/çalışma zamanı alt yoluna `MessagePresentation` şemasını ve ayrıştırıcı yardımcılarını ekleyin.
4. Mesaj yetenekleri `buttons`, `cards`, `components` ve `blocks` öğelerini anlamsal sunum yetenekleriyle değiştirin.
5. Sunum işleme ve teslimat sabitlemesi için çalışma zamanı giden bağdaştırıcı kancaları ekleyin.
6. Bağlamlar arası bileşen oluşturmayı `buildCrossContextPresentation` ile değiştirin.
7. `src/infra/outbound/channel-adapters.ts` öğesini silin ve `buildCrossContextComponents` öğesini kanal plugin türlerinden kaldırın.
8. `maybeApplyCrossContextMarker` öğesini yerel parametreler yerine `presentation` ekleyecek şekilde değiştirin.
9. Plugin yönlendirmesinin gönderim yollarını yalnızca anlamsal sunumu ve teslimat meta verilerini kullanacak şekilde güncelleyin.
10. Ajan ve CLI yerel yük parametrelerini kaldırın: `components`, `blocks`, `buttons` ve `card`.
11. Yerel mesaj aracı şemaları oluşturan SDK yardımcılarını kaldırıp bunların yerine sunum şeması yardımcılarını ekleyin.
12. Kullanıcı arayüzü/yerel zarfları `channelData` öğesinden kaldırın; kalan her alan incelenene kadar yalnızca aktarım meta verilerini tutun.
13. Discord, Slack, Telegram, Mattermost, MS Teams, Feishu ve LINE işleyicilerini geçirin.
14. Mesaj CLI'si, kanal sayfaları, plugin SDK'sı ve yetenek tarifleri için belgeleri güncelleyin.
15. Discord ve etkilenen kanal giriş noktaları için içe aktarma yayılımı profillemesi çalıştırın.

Bu yeniden düzenlemede paylaşılan ajan, CLI, plugin yeteneği ve giden bağdaştırıcı sözleşmeleri için 1-11 ve 13-14. adımlar uygulanmıştır. 12. adım, sağlayıcıya özel `channelData` aktarım zarfları için daha kapsamlı bir dahili temizleme aşaması olarak kalır. Tür/test geçidinin ötesinde nicel içe aktarma yayılımı sayıları isteniyorsa 15. adım sonraki doğrulama çalışması olarak kalır.

## Testler

Ekleyin veya güncelleyin:

- Sunum normalleştirme testleri.
- Desteklenmeyen bloklar için sunumu otomatik olarak daha basit biçime dönüştürme testleri.
- Plugin yönlendirmesi ve çekirdek teslimat yolları için bağlamlar arası işaretleyici testleri.
- Discord, Slack, Telegram, Mattermost, MS Teams, Feishu, LINE ve yedek metin için kanal işleme matrisi testleri.
- Yerel alanların kaldırıldığını kanıtlayan mesaj aracı şeması testleri.
- Yerel bayrakların kaldırıldığını kanıtlayan CLI testleri.
- Carbon'u kapsayan Discord giriş noktası içe aktarma tembelliği regresyon testi.
- Telegram'ı ve genel geri dönüş davranışını kapsayan teslimat sabitleme testleri.

## Açık sorular

- İlk aşamada `delivery.pin` Discord, Slack, MS Teams ve Feishu için mi uygulanmalı, yoksa önce yalnızca Telegram için mi?
- İleride `delivery`; `replyToId`, `replyToCurrent`, `silent` ve `audioAsVoice` gibi mevcut alanları bünyesine katmalı mı, yoksa gönderim sonrası davranışlara odaklanmayı sürdürmeli mi?
- Sunum, görüntüleri veya dosya referanslarını doğrudan desteklemeli mi, yoksa medya şimdilik kullanıcı arayüzü düzeninden ayrı mı kalmalı?

## İlgili

- [Kanallara genel bakış](/tr/channels)
- [İleti sunumu](/tr/plugins/message-presentation)
