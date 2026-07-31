---
read_when:
    - OpenClaw model kullanımını, mesaj akışını veya oturum metriklerini bir OpenTelemetry toplayıcısına göndermek istiyorsunuz
    - İzleri, metrikleri veya günlükleri Grafana, Datadog, Honeycomb, New Relic, Tempo ya da başka bir OTLP arka ucuna bağlıyorsunuz
    - Panolar veya uyarılar oluşturmak için tam metrik adlarına, span adlarına ya da öznitelik yapılarına ihtiyacınız var
summary: diagnostics-otel Plugin aracılığıyla OpenClaw tanılama verilerini OpenTelemetry toplayıcılarına veya stdout JSONL'ye aktarın
title: OpenTelemetry dışa aktarımı
x-i18n:
    generated_at: "2026-07-26T22:46:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6ed37f094c6c151379d8e0aaa2633b3ebebdb08b7dcbc9403c4bdeb6e5b8cf76
    source_path: gateway/opentelemetry.md
    workflow: 16
---

OpenClaw, tanılama verilerini resmi `diagnostics-otel` plugin'i aracılığıyla
**OTLP/HTTP (protobuf)** kullanarak dışa aktarır. Günlükler, konteyner ve
korumalı alan günlük işlem hatları için stdout JSONL olarak da yazılabilir.
OTLP/HTTP kabul eden herhangi bir toplayıcı veya arka uç, kod değişikliği
gerektirmeden çalışır. Yerel dosya günlükleri için
[Günlük Kaydı](/tr/logging) bölümüne bakın.

- **Tanılama olayları**, model çalıştırmaları, ileti akışı, oturumlar, kuyruklar
  ve exec için Gateway ile paketlenmiş plugin'ler tarafından yayımlanan,
  yapılandırılmış işlem içi kayıtlardır.
- **`diagnostics-otel`**, bu olaylara abone olur ve bunları OTLP/HTTP üzerinden
  OpenTelemetry **metrikleri**, **izleri** ve **günlükleri** olarak dışa aktarır;
  ayrıca günlük kayıtlarını stdout JSONL'a yansıtabilir.
- **Sağlayıcı çağrıları**, sağlayıcı aktarımı özel üst bilgileri kabul ettiğinde
  OpenClaw'ın güvenilir model çağrısı span bağlamından bir W3C
  `traceparent` üst bilgisi alır. Plugin tarafından yayımlanan iz bağlamı
  aktarılmaz.
- Dışa aktarıcılar yalnızca hem tanılama yüzeyi hem de plugin etkin olduğunda
  bağlanır; böylece işlem içi maliyet varsayılan olarak sıfıra yakın kalır.

## Hızlı başlangıç

```bash
openclaw plugins install clawhub:@openclaw/diagnostics-otel
```

```json5
{
  plugins: {
    allow: ["diagnostics-otel"],
    entries: {
      "diagnostics-otel": { enabled: true },
    },
  },
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      protocol: "http/protobuf",
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: true,
      sampleRate: 0.2,
      flushIntervalMs: 60000,
    },
  },
}
```

Ya da plugin'i CLI'dan etkinleştirin: `openclaw plugins enable diagnostics-otel`.

<Note>
`protocol` yalnızca `http/protobuf` destekler. `traces` ve `metrics` varsayılan olarak etkin olduğundan, diğer tüm değerler (`grpc` dâhil) `unsupported protocol` uyarısıyla diagnostics-otel aboneliğinin tamamını iptal eder; bu, stdout günlük dışa aktarımını da durdurur. Yalnızca OTLP dışı bir protokol değeriyle `logsExporter: "stdout"` kullanmak istiyorsanız `traces: false` ve `metrics: false` değerlerini açıkça ayarlayın.
</Note>

## Dışa aktarılan sinyaller

| Sinyal      | İçeriği                                                                                                                                                                                              |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Metrikler** | Token kullanımı, maliyet, çalıştırma süresi, yük devretme, skill kullanımı, ileti akışı, Talk olayları, kuyruk şeritleri, oturum durumu/kurtarma, araç yürütme, exec, bellek, canlılık ve dışa aktarıcı sağlığı için sayaçlar/histogramlar. |
| **İzler**  | Model kullanımı, model çağrıları, harness yaşam döngüsü, skill kullanımı, araç yürütme, exec, webhook/ileti işleme, bağlam oluşturma ve araç döngüleri için span'ler.                                                      |
| **Günlükler**    | `diagnostics.otel.logs` etkin olduğunda OTLP veya stdout JSONL üzerinden dışa aktarılan yapılandırılmış `logging.file` kayıtları; içerik yakalama açıkça etkinleştirilmediği sürece günlük gövdeleri dışarıda bırakılır.                          |

`traces`, `metrics` ve `logs` ayarlarını birbirinden bağımsız olarak değiştirin. `diagnostics.otel.enabled` true olduğunda izler ve metrikler
varsayılan olarak açık, günlükler ise varsayılan olarak kapalıdır
ve yalnızca `diagnostics.otel.logs` açıkça `true` olduğunda dışa aktarılır. Günlük dışa aktarımı
varsayılan olarak OTLP kullanır; stdout üzerinde JSONL için `diagnostics.otel.logsExporter` değerini
`stdout`, her ikisi için ise `both` olarak ayarlayın.

## Yapılandırma başvurusu

```json5
{
  diagnostics: {
    enabled: true,
    otel: {
      enabled: true,
      endpoint: "http://otel-collector:4318",
      tracesEndpoint: "http://otel-collector:4318/v1/traces",
      metricsEndpoint: "http://otel-collector:4318/v1/metrics",
      logsEndpoint: "http://otel-collector:4318/v1/logs",
      protocol: "http/protobuf", // grpc, OTLP dışa aktarımını devre dışı bırakır
      serviceName: "openclaw-gateway", // ayarlanmazsa OTEL_SERVICE_NAME, ardından "openclaw" kullanılır
      headers: { "x-collector-token": "..." },
      traces: true,
      metrics: true,
      logs: true,
      logsExporter: "otlp", // otlp | stdout | both
      sampleRate: 0.2, // kök span örnekleyicisi, 0.0..1.0
      flushIntervalMs: 60000, // metrik dışa aktarma aralığı (en az 1000ms)
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },
  },
}
```

### Ortam değişkenleri

| Değişken                                                                                                          | Amaç                                                                                                                                                                                                                                                                                                        |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                                                                                     | Yapılandırma anahtarı ayarlanmamışsa `diagnostics.otel.endpoint` için yedek değer.                                                                                                                                                                                                                                         |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` / `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` / `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | Eşleşen `diagnostics.otel.*Endpoint` yapılandırma anahtarı ayarlanmamışsa kullanılan sinyale özgü uç nokta yedekleri. Sinyale özgü yapılandırma, sinyale özgü ortam değişkeninden; sinyale özgü ortam değişkeni ise paylaşılan uç noktadan önceliklidir.                                                                                                         |
| `OTEL_SERVICE_NAME`                                                                                               | Yapılandırma anahtarı ayarlanmamışsa `diagnostics.otel.serviceName` için yedek değer. Varsayılan hizmet adı `openclaw` değeridir.                                                                                                                                                                                                  |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                                                                                     | `diagnostics.otel.protocol` ayarlanmamışsa kablo protokolü için yedek değer. Yalnızca `http/protobuf` dışa aktarımı etkinleştirir.                                                                                                                                                                                                 |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                                                                                   | En yeni GenAI çıkarım span biçimini yayımlamak için `gen_ai_latest_experimental` olarak ayarlayın: `{gen_ai.operation.name} {gen_ai.request.model}` span adları, `CLIENT` span türü ve eski `gen_ai.system` yerine `gen_ai.provider.name`. GenAI metrikleri, bundan bağımsız olarak her zaman sınırlı ve düşük kardinaliteli öznitelikler kullanır. |
| `OPENCLAW_OTEL_PRELOADED`                                                                                         | Başka bir ön yükleme veya ana makine işlemi, genel OpenTelemetry SDK'sını zaten kaydettiğinde `1` olarak ayarlayın. Plugin bu durumda kendi NodeSDK yaşam döngüsünü atlar ancak tanılama dinleyicilerini bağlamaya ve `traces`/`metrics`/`logs` ayarlarına uymaya devam eder.                                                                                    |

## Gizlilik ve içerik yakalama

Ham model/araç içeriği varsayılan olarak dışa **aktarılmaz**. Span'ler sınırlı
tanımlayıcılar (kanal, sağlayıcı, model, hata kategorisi, yalnızca karma değer içeren istek kimlikleri,
araç kaynağı, araç sahibi, skill adı/kaynağı) taşır ve hiçbir zaman istem metnini,
yanıt metnini, araç girdilerini, araç çıktılarını, skill dosyası yollarını veya oturum anahtarlarını içermez.
Kapsamlı aracı oturum anahtarlarına benzeyen değerler (örneğin
`agent:` ile başlayanlar), düşük kardinaliteli özniteliklerde `unknown` ile değiştirilir. OTLP günlük
kayıtları varsayılan olarak önem derecesini, günlük kaydediciyi, kod konumunu, güvenilir iz bağlamını ve
temizlenmiş öznitelikleri korur; ham günlük ileti gövdesi yalnızca
`diagnostics.otel.captureContent` boole değeri `true` olduğunda dışa aktarılır. Ayrıntılı
`captureContent.*` alt anahtarları günlük gövdelerini hiçbir zaman etkinleştirmez. Talk metrikleri yalnızca
sınırlı olay meta verilerini (mod, aktarım, sağlayıcı, olay türü) dışa aktarır; konuşma dökümleri,
ses yükleri, oturum kimlikleri, tur kimlikleri, çağrı kimlikleri, oda kimlikleri veya
devir token'ları dışa aktarılmaz.

Giden model istekleri yalnızca etkin model çağrısına ait OpenClaw'ın sahip olduğu
tanılama iz bağlamından oluşturulan bir W3C `traceparent` üst bilgisi içerebilir.
Çağrıyı yapan tarafından sağlanan mevcut `traceparent` üst bilgileri değiştirilir; böylece plugin'ler veya
özel sağlayıcı seçenekleri hizmetler arası iz üst soyunu taklit edemez.

Yalnızca toplayıcınız ve saklama politikanız istem, yanıt, araç veya
sistem istemi metni için onaylanmışsa `diagnostics.otel.captureContent.*` değerini `true` olarak ayarlayın.
Her alt anahtar bağımsızdır:

- `inputMessages` - kullanıcı istemi içeriği.
- `outputMessages` - model yanıtı içeriği.
- `toolInputs` - araç bağımsız değişkeni yükleri.
- `toolOutputs` - araç sonucu yükleri.
- `systemPrompt` - birleştirilmiş sistem/geliştirici istemi.
- `toolDefinitions` - model aracı adları, açıklamaları ve şemaları.

Herhangi bir alt anahtar etkinleştirildiğinde model ve araç span'leri yalnızca ilgili sınıf için
sınırlı, redakte edilmiş `openclaw.content.*` öznitelikleri alır.

<Note>
Boole `captureContent: true`, `inputMessages`, `outputMessages`, `toolInputs`, `toolOutputs`, `toolDefinitions` ve OTLP günlük gövdelerini birlikte etkinleştirir, ancak `systemPrompt` değerini **etkinleştirmez**; birleştirilmiş sistem istemine de ihtiyacınız varsa `captureContent.systemPrompt: true` değerini açıkça ayarlayın.
</Note>

`toolInputs`/`toolOutputs` içeriği, yerleşik aracı
çalışma zamanının araç yürütmeleri için yakalanır (tamamlanan/hata span'lerinde
`openclaw.content.tool_input` ve `gen_ai.tool.call.arguments`;
tamamlanan span'lerde `openclaw.content.tool_output` ve
`gen_ai.tool.call.result`). `openclaw.content.*` adları kararlı OpenClaw öznitelik
adları olarak kalır; `gen_ai.tool.call.*` kopyaları semconv yerel görüntüleyiciler için bunları yansıtır.
Harici harness araç çağrıları (Codex, Claude CLI), içerik yükleri olmadan
`tool.execution.*` span'leri yayımlar. Yakalanan içerik güvenilir, yalnızca dinleyiciye açık bir
kanal üzerinden taşınır ve hiçbir zaman genel tanılama olay veri yoluna
yerleştirilmez.

## Örnekleme ve boşaltma

- **İzler:** `diagnostics.otel.sampleRate` yalnızca kök span üzerinde bir `TraceIdRatioBasedSampler`
  ayarlar (`0.0` tümünü bırakır, `1.0` tümünü tutar). Ayarlanmadığında
  OpenTelemetry SDK varsayılanı (her zaman açık) kullanılır.
- **Metrikler:** `diagnostics.otel.flushIntervalMs` (en az
  `1000` olacak şekilde sınırlandırılır); ayarlanmadığında SDK'nın periyodik dışa aktarma varsayılanı kullanılır.
- **Günlükler:** OTLP günlükleri `logging.level` değerine (dosya günlük düzeyi) uyar ve
  konsol biçimlendirmesini değil, tanılama günlük kaydı redaksiyon yolunu kullanır. Yüksek hacimli
  kurulumlar, yerel örnekleme yerine OTLP toplayıcı örneklemesini/filtrelemesini
  tercih etmelidir. Platformunuz stdout/stderr'ı zaten bir günlük işleyicisine
  gönderiyorsa ve OTLP günlük toplayıcınız yoksa `diagnostics.otel.logsExporter: "stdout"`
  değerini ayarlayın. Stdout kayıtları, her satırda `ts`, `signal`,
  `service.name`, önem derecesi, gövde, redakte edilmiş öznitelikler ve kullanılabildiğinde
  güvenilir iz alanları içeren bir JSON nesnesidir.
- **Dosya-günlük korelasyonu:** JSONL dosya günlükleri, günlük çağrısı geçerli bir
  tanılama iz bağlamı taşıdığında üst düzey `traceId`,
  `spanId`, `parentSpanId` ve `traceFlags` alanlarını içerir; bu sayede günlük işleyicileri
  yerel günlük satırlarını dışa aktarılan span'lerle birleştirebilir.
- **İstek korelasyonu:** Gateway HTTP istekleri ve WebSocket çerçeveleri
  dahili bir istek iz kapsamı oluşturur. Bu kapsam içindeki günlükler ve tanılama olayları,
  varsayılan olarak istek izini devralırken sağlayıcı `traceparent` üstbilgilerinin aynı
  izde kalması için ajan çalıştırma ve model çağrısı span'leri alt öğeler olarak oluşturulur.
- **Model çağrısı korelasyonu:** `openclaw.model.call` span'leri varsayılan olarak güvenli istem
  bileşeni boyutlarını ve sağlayıcı sonucu kullanımı sunduğunda çağrı başına token özniteliklerini
  içerir. `openclaw.model.usage`, toplam maliyet, bağlam ve kanal
  panoları için çalıştırma düzeyindeki hesaplama span'i olmaya devam eder ve olayı yayan çalışma zamanı
  güvenilir iz bağlamına sahip olduğunda aynı tanılama izinde kalır.

### Model çağrısı gözlem birimleri

Her `openclaw.model.call` span'i, yaşam döngüsünün neyi ölçtüğünü
`openclaw.model_call.observation_unit` aracılığıyla tanımlar:

- `request` - gözlemlenebilir tek bir model/sağlayıcı isteği. Yerel gömülü model
  çağrıları bu birimi kullanır ve dışa aktarıcılar, eski veya harici yayıcılarla uyumluluk
  için eksik bir değeri `request` olarak kabul eder.
- `turn` - gizli model istekleri, yeniden denemeler, araç çalışmaları veya arka plan
  çalışmaları içerebilen tek bir opak ajan CLI turu. Claude Code CLI ve Codex app-server
  çağrıları bu birimi kullanır.

İz arka uçlarının model girdisini, çıktısını, kullanımını ve hiyerarşisini işleyebilmesi için
her iki birim de model çağrısı span'leri olarak kalır. İstek span'leri API'den türetilen GenAI işlemini
(`chat`, `generate_content` veya `text_completion`) kullanırken tur span'leri
`gen_ai.operation.name = invoke_agent` kullanır. Her ikisi de
`gen_ai.client.operation.duration` metriğine katkıda bulunur; burada işlem adı doğrudan
istek gecikmesini tam tur gecikmesinden ayrı tutar. OpenClaw'ın OTEL model çağrısı
metrikleri ayrıca `openclaw.model_call.observation_unit` içerir; Prometheus
model çağrısı metrikleri eşdeğer `observation_unit` etiketini sunar.

### Claude Code CLI model çağrısı doğruluğu

Claude Code CLI turları, tur düzeyinde tek bir sentetik `openclaw.model.call`
span'i yayar. Bunlar Anthropic HTTP istek span'leri değildir. `openclaw.api =
claude-code` ve `openclaw.model_call.observation_unit = turn` kullanırlar ve
işlemi `gen_ai.operation.name = invoke_agent` olarak tanımlarlar. OpenClaw'ın CLI sınırını
`openclaw.transport` aracılığıyla tanımlarlar:

- `stdio` - tek seferlik yerel Claude Code işlemi.
- `stdio-live` - yönetilen kalıcı bir Claude stdio oturumundaki tek tur.
- `paired-node-cli` - eşleştirilmiş bir Node'a devredilen tek seferlik Claude Code
  yürütmesi.

Claude CLI tanılamaları yalnızca işlem tanılama dağıtıcısı etkinleştirildiğinde
ve dahili veya güvenilir bir olay dinleyicisi bağlı olduğunda örneklenir.
Etkin bir gözlemlenebilirlik Plugin'i veya başka bir dinleyici olmadığında Claude CLI turları,
sentetik iz hiyerarşisini, içerik arabelleklerini ve tanılama akışı bayt
hesaplamasını atlar. İçerik yakalama etkinleştirildiğinde istem ve sistem istemi alanlarının
her biri 128 KiB ile sınırlandırılır; yardımcı çıktısı en fazla 200 zarf genelinde 128 KiB ile
sınırlandırılır ve son görünür yedek yanıt için 16 KiB ve bir öğe ayrılır.
Sınıra ulaşıldığında bir işaretçi kesilmeyi kaydeder.

OpenClaw, Claude CLI turlarına diğer ajan çalışma zamanları tarafından kullanılan aynı sahiplik
hiyerarşisini verir: `openclaw.harness.run` (`openclaw.harness.id = claude-cli`)
`openclaw.run` öğesini, o da Claude `openclaw.model.call`
span'ini içerir. Donanım ve çalıştırma span'leri, Claude Code'un dahili aşamaları değil,
sentetik OpenClaw tur sınırlarıdır. Tek seferlik ve yönetilen stdio turları aynı
hiyerarşiyi kullanır; gerçek bir yeni oturum yeniden denemesi, aynı OpenClaw çalıştırması içinde
başka bir model çağrısı alt öğesi oluşturur.

Span, OpenClaw hazırlanmış CLI turunu kabul ettiğinde başlar ve yalnızca
bu tur başarılı olduğunda veya başarısız olduğunda sona erer. Yönetilen oturumlarda Claude,
sonucu bekleten arka plan ajanları veya iş akışları bildirirken geçici bir başarı sonucu
span'i sonlandırmaz; boşaltma sonrası nihai sonuç sonlandırır. İptal, zaman aşımı, işlem hatası,
çıktı/ayrıştırma hatası ve diğer tur hataları aynı span'i hatayla sonlandırır.

Claude Code, yardımcı mesajı başına kullanım bildirir ve terminal sonucunda birikimli
kullanım da bildirebilir. OpenClaw yanıt hesaplaması, mevcut maliyet anlamlarının değişmemesi için
son yardımcı mesajını kullanmaya devam eder; tur düzeyindeki model çağrısı span'i ise kullanılabildiğinde
önbellek okuma ve önbellek oluşturma token'ları dahil olmak üzere terminaldeki birikimli kullanımı
kullanır.

Bu CLI span'lerinde bayt ve zamanlama alanları, gözlemlenebilir OpenClaw
CLI sınırını açıklar:

- `openclaw.model_call.request_bytes`, tek seferlik stdin/argv üzerinden gönderilen istem değerinin
  veya yönetilen stdio JSONL kullanıcı zarfının UTF-8 boyutudur. Claude Code'un gizli
  model isteğinin boyutu değildir.
- `openclaw.model_call.response_bytes`, tur sırasında gözlemlenen Claude CLI stdout'unun
  UTF-8 boyutudur. Anthropic HTTP yanıt boyutu değildir.
- `openclaw.model_call.time_to_first_byte_ms`, gözlemlenebilir ilk Claude CLI stdout veya stderr
  çıktısına kadar geçen süredir. Ağ TTFB'si değildir.

Eşleşen ayrıntılı `captureContent` alanları etkinleştirildiğinde span,
OpenClaw'ın Claude Code'a gönderdiği etkin istemi, OpenClaw'ın eklediği sistem
istemini ve görünür yardımcı metnini/akıl yürütmesini/araç çağrısı kimliğini
`gen_ai.input.messages`, `gen_ai.output.messages` ve
`gen_ai.system_instructions` aracılığıyla dışa aktarır. Araç bağımsız değişkenleri, opak düşünme imzaları ve
araç sonuçları Claude yardımcı zarfından çıkarılır. OpenClaw; Claude Code'un özel
sistem istemine, gizli sürdürülen veya Compaction uygulanmış istek yüküne, yerel dahili araç şemalarına,
ham Anthropic HTTP isteğine, dahili yeniden denemelere, yukarı akış istek kimliğine veya gerçek ağ
TTFB'sine eriştiğini iddia etmez. Claude Code, etkin yerel araç tanımlarını doğru biçimde
sunmadığından bu span'ler `gen_ai.tool.definitions` alanını doldurmaz.

Harici Claude donanım aracı span'leri, araç içeriği yakalama etkinleştirilmiş olsa bile
yalnızca meta veri içerir. Her model span'inde olduğu gibi yakalanan Claude CLI içeriği,
yalnızca güvenilir dinleyici yolunu ve dışa aktarıcının mevcut redaksiyon ve boyut
sınırlarını kullanır; içerik varsayılan olarak kapalı kalır.

## Dışa aktarılan metrikler

### Model kullanımı

- `openclaw.tokens` (sayaç, öznitelikler: `openclaw.token`, `openclaw.channel`, `openclaw.provider`, `openclaw.model`, `openclaw.agent`)
- `openclaw.cost.usd` (sayaç, öznitelikler: `openclaw.channel`, `openclaw.provider`, `openclaw.model`)
- `openclaw.run.duration_ms` (histogram, öznitelikler: `openclaw.channel`, `openclaw.provider`, `openclaw.model`)
- `openclaw.context.tokens` (histogram, öznitelikler: `openclaw.context`, `openclaw.channel`, `openclaw.provider`, `openclaw.model`)
- `gen_ai.client.token.usage` (histogram, GenAI anlamsal kuralları metriği, öznitelikler: `gen_ai.token.type` = `input`/`output`, `gen_ai.provider.name`, `gen_ai.operation.name`, `gen_ai.request.model`)
- `gen_ai.client.operation.duration` (histogram, saniye, model istekleri ve sentetik ajan turları için GenAI anlamsal kuralları metriği; öznitelikler: `gen_ai.provider.name`, `gen_ai.operation.name`, `gen_ai.request.model`, isteğe bağlı `error.type`; tur gözlemleri `gen_ai.operation.name = invoke_agent` kullanır)
- `openclaw.model_call.duration_ms` (histogram, öznitelikler: `openclaw.provider`, `openclaw.model`, `openclaw.api`, `openclaw.transport`, `openclaw.model_call.observation_unit`; ayrıca sınıflandırılmış hatalarda `openclaw.errorCategory` ve `openclaw.failureKind`)
- `openclaw.model_call.request_bytes` (histogram, nihai model isteği yükünün UTF-8 bayt boyutu; Claude Code CLI için yukarıda açıklanan gözlemlenebilir istem girdisi/zarfı; ham yük içeriği yoktur)
- `openclaw.model_call.response_bytes` (histogram, akışla iletilen yanıt parçası yüklerinin UTF-8 bayt boyutu; yüksek frekanslı metin, düşünme ve araç çağrısı deltalarında yalnızca artımlı `delta` baytları sayılır; Claude Code CLI için gözlemlenen stdout baytları; ham yanıt içeriği yoktur)
- `openclaw.model_call.time_to_first_byte_ms` (histogram, akışla iletilen ilk yanıt olayından önce geçen süre; Claude Code CLI için ağ TTFB'si yerine gözlemlenebilir ilk CLI çıktısı)
- `openclaw.model.failover` (sayaç, öznitelikler: `openclaw.provider`, `openclaw.model`, `openclaw.failover.to_provider`, `openclaw.failover.to_model`, `openclaw.failover.reason`, `openclaw.failover.suspended`, `openclaw.lane`)
- `openclaw.skill.used` (sayaç, öznitelikler: `openclaw.skill.name`, `openclaw.skill.source`, `openclaw.skill.activation`, isteğe bağlı `openclaw.agent`, isteğe bağlı `openclaw.toolName`)

### Mesaj akışı

- `openclaw.webhook.received` (sayaç, öznitelikler: `openclaw.channel`, `openclaw.webhook`)
- `openclaw.webhook.error` (sayaç, öznitelikler: `openclaw.channel`, `openclaw.webhook`)
- `openclaw.webhook.duration_ms` (histogram, öznitelikler: `openclaw.channel`, `openclaw.webhook`)
- `openclaw.message.queued` (sayaç, öznitelikler: `openclaw.channel`, `openclaw.source`)
- `openclaw.message.received` (sayaç, öznitelikler: `openclaw.channel`, `openclaw.source`)
- `openclaw.message.dispatch.started` (sayaç, öznitelikler: `openclaw.channel`, `openclaw.source`)
- `openclaw.message.dispatch.completed` (sayaç, öznitelikler: `openclaw.channel`, `openclaw.outcome`, `openclaw.reason`, `openclaw.source`)
- `openclaw.message.dispatch.duration_ms` (histogram, öznitelikler: `openclaw.channel`, `openclaw.outcome`, `openclaw.reason`, `openclaw.source`)
- `openclaw.message.processed` (sayaç, öznitelikler: `openclaw.channel`, `openclaw.outcome`)
- `openclaw.message.duration_ms` (histogram, öznitelikler: `openclaw.channel`, `openclaw.outcome`)
- `openclaw.message.delivery.started` (sayaç, öznitelikler: `openclaw.channel`, `openclaw.delivery.kind`)
- `openclaw.message.delivery.duration_ms` (histogram, öznitelikler: `openclaw.channel`, `openclaw.delivery.kind`, `openclaw.outcome`, `openclaw.errorCategory`)

### Konuşma

- `openclaw.talk.event` (sayaç, öznitelikler: `openclaw.talk.event_type`, `openclaw.talk.mode`, `openclaw.talk.transport`, `openclaw.talk.brain`, `openclaw.talk.provider`)
- `openclaw.talk.event.duration_ms` (histogram, öznitelikler: `openclaw.talk.event` ile aynı; bir Konuşma olayı süre bildirdiğinde yayılır)
- `openclaw.talk.audio.bytes` (histogram, öznitelikler: `openclaw.talk.event` ile aynı; bayt uzunluğu bildiren Konuşma ses çerçevesi olayları için yayılır)

### Kuyruklar ve oturumlar

- `openclaw.queue.lane.enqueue` (sayaç, öznitelikler: `openclaw.lane`)
- `openclaw.queue.lane.dequeue` (sayaç, öznitelikler: `openclaw.lane`)
- `openclaw.queue.depth` (histogram, öznitelikler: `openclaw.lane` veya `openclaw.channel=heartbeat`)
- `openclaw.queue.wait_ms` (histogram, öznitelikler: `openclaw.lane`)
- `openclaw.session.state` (sayaç, öznitelikler: `openclaw.state`, `openclaw.reason`)
- `openclaw.session.stuck` (sayaç, öznitelikler: `openclaw.state`; kurtarılabilir eski oturum kayıt yönetimi için yayımlanır)
- `openclaw.session.stuck_age_ms` (histogram, öznitelikler: `openclaw.state`; kurtarılabilir eski oturum kayıt yönetimi için yayımlanır)
- `openclaw.session.turn.created` (sayaç, öznitelikler: `openclaw.agent`, `openclaw.channel`, `openclaw.trigger`)
- `openclaw.session.recovery.requested` (sayaç, öznitelikler: `openclaw.state`, `openclaw.action`, `openclaw.active_work_kind`, `openclaw.reason`)
- `openclaw.session.recovery.completed` (sayaç, öznitelikler: `openclaw.state`, `openclaw.action`, `openclaw.status`, `openclaw.active_work_kind`, `openclaw.reason`)
- `openclaw.session.recovery.age_ms` (histogram, öznitelikler: eşleşen kurtarma sayacıyla aynı)
- `openclaw.run.attempt` (sayaç, öznitelikler: `openclaw.attempt`)

### Oturum canlılığı telemetrisi

OpenClaw yanıt, araç, durum, blok veya ACP çalışma zamanı ilerlemesi gözlemlediği sürece bir `processing` oturumu, yerleşik canlılık eşiğine doğru eskimez. Yazma etkinliğini sürdüren sinyaller ilerleme sayılmaz; böylece sessiz bir model veya çalıştırma düzeneği yine de algılanabilir.

OpenClaw, oturumları hâlâ gözlemleyebildiği çalışmaya göre sınıflandırır:

- `session.long_running`: etkin gömülü çalışma, model çağrıları veya araç çağrıları
  ilerlemeyi sürdürmektedir. Sahipli sessiz model çağrıları da yerleşik iptal eşiğinden önce uzun süreli olarak bildirilir; böylece yavaş veya akışsız model sağlayıcıları, iptal gözlemlenebildiği sürece takılmış Gateway oturumları gibi görünmez.
- `session.stalled`: etkin çalışma vardır ancak etkin çalıştırma yakın zamanda
  ilerleme bildirmemiştir. Sahipli model çağrıları, yerleşik iptal eşiğinde veya sonrasında `session.long_running` durumundan
  `session.stalled` durumuna geçer; sahipsiz
  eski model/araç etkinliği zararsız, uzun süreli çalışma olarak değerlendirilmez.
  Takılmış gömülü çalıştırmalar başlangıçta yalnızca gözlemlenir, ardından
  ilerleme olmadan iptal eşiğine ulaşıldığında iptal edilip boşaltılır; böylece kulvarda arkada bekleyen sıralı işlemler devam edebilir.
- `session.stuck`: etkin çalışması olmayan eski oturum kayıt yönetimi veya eski
  sahipsiz model/araç etkinliğine sahip, sırada bekleyen boşta bir oturum. Bu, kurtarma geçitleri geçildikten hemen sonra
  etkilenen oturum kulvarını serbest bırakır.

Kurtarma, yapılandırılmış `session.recovery.requested` ve
`session.recovery.completed` olayları yayımlar. Tanılama oturumu durumu yalnızca
durumu değiştiren bir kurtarma sonucundan (`aborted` veya `released`) sonra ve yalnızca
aynı işleme nesli hâlâ güncelse boşta olarak işaretlenir.

Yalnızca `session.stuck`, `openclaw.session.stuck` sayacını,
`openclaw.session.stuck_age_ms` histogramını ve `openclaw.session.stuck`
yayılımını yayımlar. Yinelenen `session.stuck` tanılamaları, oturum değişmeden kaldığı sürece
geri çekilir; bu nedenle panolar her Heartbeat tikinde değil, sürekli artışlarda
uyarı vermelidir. Yapılandırma ayarı ve varsayılanlar için
[Yapılandırma başvurusu](/tr/gateway/configuration-reference#diagnostics) bölümüne bakın.

Canlılık uyarıları ayrıca şunları yayımlar:

- `openclaw.liveness.warning` (sayaç, öznitelikler: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_delay_p99_ms` (histogram, öznitelikler: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_delay_max_ms` (histogram, öznitelikler: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_utilization` (histogram, öznitelikler: `openclaw.liveness.reason`)
- `openclaw.liveness.cpu_core_ratio` (histogram, öznitelikler: `openclaw.liveness.reason`)

### Çalıştırma düzeneği yaşam döngüsü

- `openclaw.harness.duration_ms` (histogram, öznitelikler: `openclaw.harness.id`, `openclaw.harness.plugin`, `openclaw.outcome`, hatalarda `openclaw.harness.phase`)

### Araç yürütme ve döngü algılama

- `openclaw.tool.execution.duration_ms` (histogram, öznitelikler: `gen_ai.tool.name`, `openclaw.toolName`, `openclaw.tool.source`, `openclaw.tool.owner`, `openclaw.tool.params.kind`, ayrıca hatalarda `openclaw.errorCategory`)
- `openclaw.tool.execution.blocked` (sayaç, öznitelikler: `gen_ai.tool.name`, `openclaw.toolName`, `openclaw.tool.source`, `openclaw.tool.owner`, `openclaw.tool.params.kind`, `openclaw.deniedReason`)
- `openclaw.tool.loop` (sayaç, öznitelikler: `openclaw.toolName`, `openclaw.loop.level`, `openclaw.loop.action`, `openclaw.loop.detector`, `openclaw.loop.count`, isteğe bağlı `openclaw.loop.paired_tool`; yinelenen bir araç çağrısı döngüsü algılandığında yayımlanır)

### Exec

- `openclaw.exec.duration_ms` (histogram, öznitelikler: `openclaw.exec.target`, `openclaw.exec.mode`, `openclaw.outcome`, `openclaw.failureKind`)

### Tanılama iç bileşenleri (bellek, yükler, dışa aktarıcı sağlığı)

- `openclaw.payload.large` (sayaç, öznitelikler: `openclaw.payload.surface`, `openclaw.payload.action`, `openclaw.channel`, `openclaw.plugin`, `openclaw.reason`)
- `openclaw.payload.large_bytes` (histogram, öznitelikler: `openclaw.payload.large` ile aynı)
- `openclaw.memory.rss_bytes` / `openclaw.memory.heap_used_bytes` / `openclaw.memory.heap_total_bytes` / `openclaw.memory.external_bytes` / `openclaw.memory.array_buffers_bytes` (histogramlar, öznitelik yok; işlem belleği örnekleri)
- `openclaw.memory.pressure` (sayaç, öznitelikler: `openclaw.memory.level`, `openclaw.memory.reason`)
- `openclaw.diagnostic.async_queue.dropped` (sayaç, öznitelikler: `openclaw.diagnostic.async_queue.drop_class`; iç tanılama kuyruğu geri basıncı nedeniyle bırakılanlar)
- `openclaw.telemetry.exporter.events` (sayaç, öznitelikler: `openclaw.exporter`, `openclaw.signal`, `openclaw.status`, isteğe bağlı `openclaw.reason`, isteğe bağlı `openclaw.errorCategory`; dışa aktarıcı yaşam döngüsü/arızası öz telemetrisi)

## Dışa aktarılan yayılımlar

- `openclaw.model.usage`
  - `openclaw.channel`, `openclaw.provider`, `openclaw.model`
  - `openclaw.tokens.*` (girdi/çıktı/önbellek_okuma/önbellek_yazma/toplam)
  - varsayılan olarak `gen_ai.system` veya en son GenAI anlamsal kuralları tercih edildiğinde `gen_ai.provider.name`
  - `gen_ai.request.model`, `gen_ai.operation.name`, `gen_ai.usage.*`
- `openclaw.run`
  - `openclaw.outcome`, `openclaw.channel`, `openclaw.provider`, `openclaw.model`, `openclaw.errorCategory`
- `openclaw.model.call`
  - varsayılan olarak `gen_ai.system` veya en son GenAI anlamsal kuralları tercih edildiğinde `gen_ai.provider.name`
  - `gen_ai.request.model`, `gen_ai.operation.name`, `openclaw.provider`, `openclaw.model`, `openclaw.api`, `openclaw.transport`, `openclaw.model_call.observation_unit` (`request` veya `turn`)
  - `openclaw.errorCategory`, `error.type` ve hatalarda isteğe bağlı `openclaw.failureKind`
  - `openclaw.model_call.request_bytes`, `openclaw.model_call.response_bytes`, `openclaw.model_call.time_to_first_byte_ms`
  - `openclaw.model_call.prompt.input_messages_count`, `openclaw.model_call.prompt.input_messages_chars`, `openclaw.model_call.prompt.system_prompt_chars`, `openclaw.model_call.prompt.tool_definitions_count`, `openclaw.model_call.prompt.tool_definitions_chars`, `openclaw.model_call.prompt.total_chars` (yalnızca güvenli bileşen boyutları, istem metni yok)
  - sonuç, ilgili istek veya toplu sıra için kullanımı içerdiğinde `openclaw.model_call.usage.*` ve `gen_ai.usage.*`
  - Üst sağlayıcı sonucu bir istek kimliği sunduğunda `openclaw.upstreamRequestIdHash` özniteliğine sahip `openclaw.provider.request` yayılım olayı (sınırlı, karma tabanlı); ham kimlikler hiçbir zaman dışa aktarılmaz
  - `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` ile istek yayılımları en son GenAI çıkarım yayılım adı olan `{gen_ai.operation.name} {gen_ai.request.model}` adını kullanır. OpenClaw, opak CLI sınırından yerel bir aracı adı atfetmediği için sıra yayılımları `invoke_agent` kullanır. Her ikisi de `openclaw.model.call` yerine `CLIENT` yayılım türünü kullanır.
- `openclaw.harness.run`
  - `openclaw.harness.id`, `openclaw.harness.plugin`, `openclaw.outcome`, `openclaw.provider`, `openclaw.model`, `openclaw.channel`
  - Tamamlandığında: `openclaw.harness.result_classification`, `openclaw.harness.yield_detected`, `openclaw.harness.items.started`, `openclaw.harness.items.completed`, `openclaw.harness.items.active`
  - Hata durumunda: `openclaw.harness.phase`, `openclaw.errorCategory`, isteğe bağlı `openclaw.harness.cleanup_failed`
- `openclaw.tool.execution`
  - `gen_ai.tool.name`, `gen_ai.operation.name` (`execute_tool`), `openclaw.toolName`, `openclaw.tool.source`, isteğe bağlı `gen_ai.tool.call.id`, `openclaw.tool.owner`, `openclaw.tool.params.*`
  - Hatalarda isteğe bağlı `openclaw.errorCategory`/`openclaw.errorCode`; politika veya korumalı alan tarafından reddedildiğinde `openclaw.deniedReason` ve `openclaw.outcome=blocked`
- `openclaw.exec`
  - `openclaw.exec.target`, `openclaw.exec.mode`, `openclaw.outcome`, `openclaw.failureKind`, `openclaw.exec.command_length`, `openclaw.exec.exit_code`, `openclaw.exec.exit_signal`, `openclaw.exec.timed_out`
- `openclaw.webhook.processed`
  - `openclaw.channel`, `openclaw.webhook`
- `openclaw.webhook.error`
  - `openclaw.channel`, `openclaw.webhook`, `openclaw.error`
- `openclaw.message.processed`
  - `openclaw.channel`, `openclaw.outcome`, `openclaw.reason`
- `openclaw.message.delivery`
  - `openclaw.channel`, `openclaw.delivery.kind`, `openclaw.outcome`, `openclaw.errorCategory`, `openclaw.delivery.result_count`
- `openclaw.session.stuck`
  - `openclaw.state`, `openclaw.ageMs`, `openclaw.queueDepth`
- `openclaw.context.assembled`
  - `openclaw.prompt.size`, `openclaw.history.size`, `openclaw.context.tokens`, `openclaw.errorCategory` (istem, geçmiş, yanıt veya oturum anahtarı içeriği yok)
- `openclaw.tool.loop`
  - `openclaw.toolName`, `openclaw.loop.level`, `openclaw.loop.action`, `openclaw.loop.detector`, `openclaw.loop.count`, isteğe bağlı `openclaw.loop.paired_tool` (döngü mesajları, parametreler veya araç çıktısı yok)
- `openclaw.memory.pressure`
  - `openclaw.memory.level`, `openclaw.memory.reason`, `openclaw.memory.rss_bytes`, `openclaw.memory.heap_used_bytes`, `openclaw.memory.heap_total_bytes`, `openclaw.memory.external_bytes`, `openclaw.memory.array_buffers_bytes`, isteğe bağlı `openclaw.memory.threshold_bytes`/`openclaw.memory.rss_growth_bytes`/`openclaw.memory.window_ms`

İçerik yakalama açıkça etkinleştirildiğinde, model ve araç yayılımları ayrıca
etkinleştirdiğiniz belirli içerik sınıfları için sınırlı ve redakte edilmiş
`openclaw.content.*` özniteliklerini içerebilir.

## Tanılama olayları kataloğu

Aşağıdaki olaylar, yukarıdaki metrikleri ve yayılımları destekler veya doğrudan
Plugin aboneliği için kullanılabilir. `run.progress` ve `run.execution_phase` yalnızca doğrudan kullanılan
yaşam döngüsü sinyalleridir; diagnostics-otel Plugin bunları
bağımsız OTLP sinyalleri olarak dışa aktarmaz. Olay türleri ve `run.execution_phase.phase` değerleri
eklenebilir niteliktedir. TypeScript tüketicileri, her iki birleşimin de kalıcı olarak
kapsamlı olduğunu varsaymak yerine varsayılan dalları korumalıdır.

**Model kullanımı**

- `model.usage` - belirteçler, maliyet, süre, bağlam, sağlayıcı/model/kanal,
  oturum kimlikleri. `usage`, maliyet ve telemetri için sağlayıcı/sıra muhasebesidir;
  `context.used` geçerli istem/bağlam anlık görüntüsüdür ve önbelleğe alınmış girdi veya araç döngüsü çağrıları söz konusu olduğunda
  sağlayıcının `usage.total` değerinden düşük olabilir.

**Mesaj akışı**

- `webhook.received` / `webhook.processed` / `webhook.error`
- `message.queued` / `message.processed`
- `message.delivery.started` / `message.delivery.completed` / `message.delivery.error`

**Kuyruk ve oturum**

- `queue.lane.enqueue` / `queue.lane.dequeue`
- `session.state` / `session.long_running` / `session.stalled` / `session.stuck`
- `run.attempt` / `run.progress`
- `run.execution_phase` (genel, oturumla ilişkilendirilmiş gömülü çalıştırıcı başlatma aşamaları)
- `diagnostic.heartbeat` (toplu sayaçlar: Webhook'lar/kuyruk/oturum)

**Çalıştırma düzeneği yaşam döngüsü**

- `harness.run.started` / `harness.run.completed` / `harness.run.error` -
  aracı çalıştırma düzeneğinin her çalıştırmaya özgü yaşam döngüsü. `harnessId`, isteğe bağlı
  `pluginId`, sağlayıcı/model/kanal ve çalıştırma kimliğini içerir. Tamamlanma,
  `durationMs`, `outcome`, isteğe bağlı `resultClassification`, `yieldDetected`
  ve `itemLifecycle` sayılarını ekler. Hatalar `phase`
  (`prepare`/`start`/`send`/`resolve`/`cleanup`), `errorCategory` ve
  isteğe bağlı `cleanupFailed` değerini ekler.

**Exec**

- `exec.process.completed` - terminal sonucu, süre, hedef, mod, çıkış
  kodu ve hata türü. Komut metni ve çalışma dizinleri
  dahil edilmez.
- `exec.approval.followup_suppressed` - oturum yeniden bağlandıktan sonra eski onay takibi
  bırakıldı. `approvalId`, `reason`
  (`session_rebound`), `phase` (`direct_delivery` veya `gateway_preflight`)
  ve dağıtıcının zaman damgasını içerir. Oturum anahtarları, rotalar ve komut metni
  dahil edilmez.

## Dışa aktarıcı olmadan

Tanılama olaylarını `diagnostics-otel` çalıştırmadan Plugin'lerin veya özel
hedeflerin kullanımına sunun:

```json5
{
  diagnostics: { enabled: true },
}
```

`logging.level` seviyesini yükseltmeden hedefli hata ayıklama çıktısı almak için tanılama
bayraklarını kullanın. Bayraklar büyük/küçük harfe duyarlı değildir ve joker karakterleri
(`telegram.*` veya `*`) destekler:

```json5
{
  diagnostics: { flags: ["telegram.http"] },
}
```

Veya tek seferlik bir ortam değişkeni geçersiz kılması olarak:

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload openclaw gateway
```

Bayrak çıktısı standart günlük dosyasına (`logging.file`) gider ve yine
`logging.redactSensitive` tarafından hassas bilgilerden arındırılır. Tam kılavuz:
[Tanılama bayrakları](/tr/diagnostics/flags).

## Devre dışı bırakma

```json5
{
  diagnostics: { otel: { enabled: false } },
}
```

Veya `diagnostics-otel` öğesini `plugins.allow` dışında bırakın ya da
`openclaw plugins disable diagnostics-otel` komutunu çalıştırın.

## İlgili

- [Günlük kaydı](/tr/logging) - dosya günlükleri, konsol çıktısı, CLI üzerinden günlük takibi ve Control UI Günlükler sekmesi
- [Gateway günlük kaydı iç işleyişi](/tr/gateway/logging) - WS günlük stilleri, alt sistem ön ekleri ve konsol yakalama
- [Tanılama bayrakları](/tr/diagnostics/flags) - hedefli hata ayıklama günlüğü bayrakları
- [Tanılama dışa aktarımı](/tr/gateway/diagnostics) - operatör destek paketi aracı (OTEL dışa aktarımından ayrıdır)
- [Yapılandırma başvurusu](/tr/gateway/configuration-reference#diagnostics) - tam `diagnostics.*` alan başvurusu
