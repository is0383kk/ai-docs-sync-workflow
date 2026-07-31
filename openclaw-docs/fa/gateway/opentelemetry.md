---
read_when:
    - می‌خواهید معیارهای استفاده از مدل، جریان پیام یا نشست OpenClaw را به یک گردآورنده OpenTelemetry ارسال کنید
    - در حال اتصال ردیابی‌ها، متریک‌ها یا گزارش‌ها به Grafana، Datadog، Honeycomb، New Relic، Tempo یا یک بک‌اند OTLP دیگر هستید
    - برای ساخت داشبوردها یا هشدارها، به نام‌های دقیق سنجه‌ها، نام‌های span یا ساختار ویژگی‌ها نیاز دارید
summary: خروجی‌گرفتن از داده‌های تشخیصی OpenClaw به گردآورنده‌های OpenTelemetry یا JSONL در stdout از طریق Plugin diagnostics-otel
title: صدور OpenTelemetry
x-i18n:
    generated_at: "2026-07-27T14:09:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6ed37f094c6c151379d8e0aaa2633b3ebebdb08b7dcbc9403c4bdeb6e5b8cf76
    source_path: gateway/opentelemetry.md
    workflow: 16
---

OpenClaw داده‌های تشخیصی را از طریق Plugin رسمی `diagnostics-otel`
با استفاده از **OTLP/HTTP (protobuf)** صادر می‌کند. گزارش‌ها را همچنین می‌توان به‌صورت JSONL در stdout برای
پایپ‌لاین‌های گزارش‌گیری کانتینر و سندباکس نوشت. هر گردآورنده یا بک‌اندی که
OTLP/HTTP را بپذیرد، بدون تغییر کد کار می‌کند. برای گزارش‌های فایل محلی، به
[گزارش‌گیری](/fa/logging) مراجعه کنید.

- **رویدادهای تشخیصی** رکوردهای ساخت‌یافته و درون‌پردازه‌ای هستند که توسط
  Gateway و Pluginهای همراه برای اجرای مدل، جریان پیام، نشست‌ها، صف‌ها
  و exec منتشر می‌شوند.
- **`diagnostics-otel`** مشترک آن رویدادها می‌شود و آن‌ها را به‌صورت
  **سنجه‌ها**، **ردیابی‌ها** و **گزارش‌ها** از طریق OTLP/HTTP صادر می‌کند و می‌تواند
  رکوردهای گزارش را در JSONL خروجی استاندارد نیز بازتاب دهد.
- **فراخوانی‌های ارائه‌دهنده** یک هدر W3C `traceparent` را از زمینه span
  قابل‌اعتماد فراخوانی مدل OpenClaw دریافت می‌کنند، مشروط بر اینکه انتقال ارائه‌دهنده
  هدرهای سفارشی را بپذیرد. زمینه ردیابی منتشرشده توسط Plugin منتقل نمی‌شود.
- صادرکننده‌ها تنها زمانی متصل می‌شوند که هم سطح تشخیص و هم Plugin
  فعال باشند؛ بنابراین هزینه درون‌پردازه‌ای به‌طور پیش‌فرض نزدیک به صفر باقی می‌ماند.

## شروع سریع

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

یا Plugin را از CLI فعال کنید: `openclaw plugins enable diagnostics-otel`.

<Note>
`protocol` فقط از `http/protobuf` پشتیبانی می‌کند. از آنجا که `traces` و `metrics` به‌طور پیش‌فرض فعال هستند، هر مقدار دیگری (از جمله `grpc`) کل اشتراک diagnostics-otel را با هشدار `unsupported protocol` متوقف می‌کند؛ این کار صدور گزارش به stdout را نیز متوقف می‌کند. اگر فقط `logsExporter: "stdout"` را با یک مقدار پروتکل غیر OTLP می‌خواهید، `traces: false` و `metrics: false` را صریحاً تنظیم کنید.
</Note>

## سیگنال‌های صادرشده

| سیگنال      | محتوای آن                                                                                                                                                                                              |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **سنجه‌ها** | شمارنده‌ها/هیستوگرام‌ها برای مصرف توکن، هزینه، مدت اجرا، تغییر مسیر هنگام خرابی، استفاده از skill، جریان پیام، رویدادهای Talk، مسیرهای صف، وضعیت/بازیابی نشست، اجرای ابزار، exec، حافظه، زنده‌بودن و سلامت صادرکننده. |
| **ردیابی‌ها**  | spanهای مربوط به استفاده از مدل، فراخوانی‌های مدل، چرخه عمر harness، استفاده از skill، اجرای ابزار، exec، پردازش webhook/پیام، سرهم‌بندی زمینه و حلقه‌های ابزار.                                                      |
| **گزارش‌ها**    | رکوردهای ساخت‌یافته `logging.file` که هنگام فعال بودن `diagnostics.otel.logs` از طریق OTLP یا JSONL در stdout صادر می‌شوند؛ بدنه گزارش‌ها ارائه نمی‌شود، مگر اینکه ثبت محتوا صریحاً فعال شده باشد.                          |

`traces`، `metrics` و `logs` را مستقل از یکدیگر تغییر دهید. هنگامی که `diagnostics.otel.enabled` برابر true باشد، ردیابی‌ها و سنجه‌ها
به‌طور پیش‌فرض روشن هستند؛ گزارش‌ها به‌طور پیش‌فرض خاموش‌اند
و فقط زمانی صادر می‌شوند که `diagnostics.otel.logs` صریحاً `true` باشد. صدور گزارش
به‌طور پیش‌فرض از OTLP استفاده می‌کند؛ برای JSONL در
stdout، `diagnostics.otel.logsExporter` را روی `stdout` تنظیم کنید، یا برای هر دو از `both` استفاده کنید.

## مرجع پیکربندی

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
      protocol: "http/protobuf", // grpc صدور OTLP را غیرفعال می‌کند
      serviceName: "openclaw-gateway", // در صورت تنظیم‌نبودن، ابتدا از OTEL_SERVICE_NAME و سپس "openclaw" استفاده می‌شود
      headers: { "x-collector-token": "..." },
      traces: true,
      metrics: true,
      logs: true,
      logsExporter: "otlp", // otlp | stdout | both
      sampleRate: 0.2, // نمونه‌بردار span ریشه، 0.0..1.0
      flushIntervalMs: 60000, // بازه صدور سنجه (حداقل 1000ms)
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

### متغیرهای محیطی

| متغیر                                                                                                          | هدف                                                                                                                                                                                                                                                                                                        |
| ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `OTEL_EXPORTER_OTLP_ENDPOINT`                                                                                     | مقدار جایگزین برای `diagnostics.otel.endpoint` هنگامی که کلید پیکربندی تنظیم نشده باشد.                                                                                                                                                                                                                                         |
| `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT` / `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` / `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | مقادیر جایگزین نقطه پایانی مختص سیگنال که هنگام تنظیم‌نبودن کلید پیکربندی منطبق `diagnostics.otel.*Endpoint` استفاده می‌شوند. پیکربندی مختص سیگنال بر محیط مختص سیگنال اولویت دارد و محیط مختص سیگنال نیز بر نقطه پایانی مشترک اولویت دارد.                                                                                                         |
| `OTEL_SERVICE_NAME`                                                                                               | مقدار جایگزین برای `diagnostics.otel.serviceName` هنگامی که کلید پیکربندی تنظیم نشده باشد. نام پیش‌فرض سرویس `openclaw` است.                                                                                                                                                                                                  |
| `OTEL_EXPORTER_OTLP_PROTOCOL`                                                                                     | مقدار جایگزین برای پروتکل سیمی هنگامی که `diagnostics.otel.protocol` تنظیم نشده باشد. فقط `http/protobuf` صدور را فعال می‌کند.                                                                                                                                                                                                 |
| `OTEL_SEMCONV_STABILITY_OPT_IN`                                                                                   | برای انتشار جدیدترین شکل span استنتاج GenAI، آن را روی `gen_ai_latest_experimental` تنظیم کنید: نام‌های span از نوع `{gen_ai.operation.name} {gen_ai.request.model}`، نوع span برابر `CLIENT` و `gen_ai.provider.name` به‌جای `gen_ai.system` قدیمی. سنجه‌های GenAI صرف‌نظر از این تنظیم، همیشه از ویژگی‌های محدود و کم‌کاردینالیتی استفاده می‌کنند. |
| `OPENCLAW_OTEL_PRELOADED`                                                                                         | هنگامی که یک پیش‌بارگذاری یا پردازش میزبان دیگر قبلاً SDK سراسری OpenTelemetry را ثبت کرده است، آن را روی `1` تنظیم کنید. سپس Plugin چرخه عمر NodeSDK خود را نادیده می‌گیرد، اما همچنان شنونده‌های تشخیصی را متصل می‌کند و `traces`/`metrics`/`logs` را رعایت می‌کند.                                                                                    |

## حریم خصوصی و ثبت محتوا

محتوای خام مدل/ابزار به‌طور پیش‌فرض صادر **نمی‌شود**. spanها شناسه‌های
محدود (کانال، ارائه‌دهنده، مدل، دسته خطا، شناسه‌های درخواست صرفاً هش‌شده،
منبع ابزار، مالک ابزار، نام/منبع skill) را حمل می‌کنند و هرگز شامل متن prompt،
متن پاسخ، ورودی‌های ابزار، خروجی‌های ابزار، مسیر فایل‌های skill یا کلیدهای نشست نیستند.
مقادیر شبیه کلیدهای نشست عامل با دامنه محدود (برای مثال، آن‌هایی که با
`agent:` شروع می‌شوند) در ویژگی‌های کم‌کاردینالیتی با `unknown` جایگزین می‌شوند. رکوردهای گزارش OTLP
به‌طور پیش‌فرض شدت، گزارش‌گر، محل کد، زمینه ردیابی قابل‌اعتماد و
ویژگی‌های پاک‌سازی‌شده را حفظ می‌کنند؛ بدنه خام پیام گزارش فقط
زمانی صادر می‌شود که `diagnostics.otel.captureContent` مقدار بولی `true` داشته باشد. زیرکلیدهای جزئی
`captureContent.*` هرگز بدنه گزارش‌ها را فعال نمی‌کنند. سنجه‌های Talk فقط
فراداده محدود رویداد (حالت، انتقال، ارائه‌دهنده، نوع رویداد) را صادر می‌کنند و شامل
رونوشت‌ها، محتوای صوتی، شناسه‌های نشست، شناسه‌های نوبت، شناسه‌های تماس، شناسه‌های اتاق یا
توکن‌های تحویل نیستند.

درخواست‌های خروجی مدل ممکن است شامل یک هدر W3C `traceparent` باشند که فقط
از زمینه ردیابی تشخیصی متعلق به OpenClaw برای فراخوانی فعال مدل تولید شده است.
هدرهای `traceparent` موجود که توسط فراخواننده ارائه شده‌اند جایگزین می‌شوند؛ بنابراین Pluginها یا
گزینه‌های سفارشی ارائه‌دهنده نمی‌توانند تبار ردیابی میان‌سرویسی را جعل کنند.

فقط زمانی `diagnostics.otel.captureContent.*` را روی `true` تنظیم کنید که گردآورنده
و سیاست نگه‌داری شما برای متن prompt، پاسخ، ابزار یا
prompt سیستمی تأیید شده باشند. هر زیرکلید مستقل است:

- `inputMessages` - محتوای prompt کاربر.
- `outputMessages` - محتوای پاسخ مدل.
- `toolInputs` - محتوای آرگومان‌های ابزار.
- `toolOutputs` - محتوای نتیجه ابزار.
- `systemPrompt` - prompt سرهم‌شده سیستم/توسعه‌دهنده.
- `toolDefinitions` - نام‌ها، توضیحات و طرح‌واره‌های ابزار مدل.

وقتی هر زیرکلیدی فعال شود، spanهای مدل و ابزار فقط برای همان دسته
ویژگی‌های محدود و ویرایش‌شده `openclaw.content.*` را دریافت می‌کنند.

<Note>
مقدار بولی `captureContent: true`، گزینه‌های `inputMessages`، `outputMessages`، `toolInputs`، `toolOutputs`، `toolDefinitions` و بدنه گزارش‌های OTLP را با هم فعال می‌کند، اما `systemPrompt` را فعال **نمی‌کند**؛ اگر به prompt سیستمی سرهم‌شده نیز نیاز دارید، `captureContent.systemPrompt: true` را صریحاً تنظیم کنید.
</Note>

محتوای `toolInputs`/`toolOutputs` برای اجرای ابزارهای runtime داخلی عامل
ثبت می‌شود (`openclaw.content.tool_input` و
`gen_ai.tool.call.arguments` در spanهای تکمیل‌شده/خطا؛
`openclaw.content.tool_output` و `gen_ai.tool.call.result` در spanهای
تکمیل‌شده). نام‌های `openclaw.content.*` همچنان نام‌های ویژگی پایدار OpenClaw
باقی می‌مانند؛ نسخه‌های `gen_ai.tool.call.*` آن‌ها را برای نمایشگرهای بومی semconv بازتاب می‌دهند.
فراخوانی‌های ابزار harness خارجی (Codex، Claude CLI)
spanهای `tool.execution.*` را بدون محتوای همراه منتشر می‌کنند. محتوای ثبت‌شده از طریق یک
کانال قابل‌اعتماد و مختص شنونده منتقل می‌شود و هرگز روی گذرگاه عمومی رویدادهای
تشخیصی قرار نمی‌گیرد.

## نمونه‌برداری و تخلیه

- **ردیابی‌ها:** `diagnostics.otel.sampleRate` فقط روی بازهٔ ریشه یک `TraceIdRatioBasedSampler`
  تنظیم می‌کند (`0.0` همه را حذف می‌کند، `1.0` همه را نگه می‌دارد). در صورت تنظیم‌نبودن، از پیش‌فرض
  SDK ‏OpenTelemetry استفاده می‌شود (همیشه فعال).
- **سنجه‌ها:** `diagnostics.otel.flushIntervalMs` (با حداقل
  `1000` محدود می‌شود)؛ در صورت تنظیم‌نبودن، از پیش‌فرض صدور دوره‌ای SDK استفاده می‌شود.
- **گزارش‌ها:** گزارش‌های OTLP از `logging.level` (سطح گزارش فایل) پیروی می‌کنند و به‌جای
  قالب‌بندی کنسول، از مسیر حذف اطلاعات حساس رکورد گزارش تشخیصی استفاده می‌کنند. نصب‌های
  پرترافیک باید نمونه‌برداری/فیلترکردن گردآورندهٔ OTLP را به نمونه‌برداری
  محلی ترجیح دهند. وقتی پلتفرم شما از قبل stdout/stderr را به یک پردازشگر گزارش
  ارسال می‌کند و گردآورندهٔ گزارش OTLP ندارید، `diagnostics.otel.logsExporter: "stdout"` را تنظیم کنید.
  رکوردهای stdout در هر خط یک شیء JSON هستند و شامل `ts`، `signal`،
  `service.name`، شدت، بدنه، ویژگی‌های با اطلاعات حساس حذف‌شده و در صورت وجود، فیلدهای
  ردیابی مورداعتماد هستند.
- **هم‌بستگی گزارش فایل:** گزارش‌های فایل JSONL هنگامی که فراخوانی گزارش دارای
  زمینهٔ ردیابی تشخیصی معتبر باشد، `traceId`، `spanId`، `parentSpanId` و
  `traceFlags` را در سطح بالا دربر می‌گیرند؛ در نتیجه پردازشگرهای گزارش می‌توانند خطوط
  گزارش محلی را به بازه‌های صادرشده مرتبط کنند.
- **هم‌بستگی درخواست:** درخواست‌های HTTP و فریم‌های WebSocket در Gateway
  یک محدودهٔ ردیابی داخلی درخواست ایجاد می‌کنند. گزارش‌ها و رویدادهای تشخیصی درون آن
  محدوده به‌طور پیش‌فرض ردیابی درخواست را به ارث می‌برند، درحالی‌که بازه‌های اجرای عامل
  و فراخوانی مدل به‌عنوان فرزند ایجاد می‌شوند تا سرآیندهای `traceparent` ارائه‌دهنده
  در همان ردیابی باقی بمانند.
- **هم‌بستگی فراخوانی مدل:** بازه‌های `openclaw.model.call` به‌طور پیش‌فرض شامل
  اندازه‌های امن مؤلفه‌های اعلان و، در صورت ارائهٔ میزان مصرف در نتیجهٔ ارائه‌دهنده،
  ویژگی‌های توکن هر فراخوانی هستند. `openclaw.model.usage` همچنان بازهٔ
  حسابداری سطح اجرا برای داشبوردهای هزینهٔ تجمیعی، زمینه و کانال است و وقتی
  زمان‌اجرای صادرکننده زمینهٔ ردیابی مورداعتماد داشته باشد، در همان ردیابی تشخیصی
  باقی می‌ماند.

### واحدهای مشاهدهٔ فراخوانی مدل

هر بازهٔ `openclaw.model.call` از طریق `openclaw.model_call.observation_unit` مشخص می‌کند چرخهٔ عمر آن
چه چیزی را اندازه‌گیری می‌کند:

- `request` - یک درخواست قابل‌مشاهدهٔ مدل/ارائه‌دهنده. فراخوانی‌های
  بومی مدل تعبیه‌شده از این واحد استفاده می‌کنند و صادرکننده‌ها برای سازگاری با
  صادرکننده‌های قدیمی‌تر یا خارجی، مقدار موجودنبودن را `request` در نظر می‌گیرند.
- `turn` - یک نوبت مبهم CLI عامل که ممکن است شامل درخواست‌های
  پنهان مدل، تلاش‌های مجدد، کار ابزار یا کار پس‌زمینه باشد. فراخوانی‌های Claude Code CLI
  و کارساز برنامهٔ Codex از این واحد استفاده می‌کنند.

هر دو واحد به‌صورت بازه‌های فراخوانی مدل باقی می‌مانند تا سامانه‌های پشتیبان ردیابی
بتوانند ورودی، خروجی، میزان مصرف و سلسله‌مراتب مدل را نمایش دهند. بازه‌های درخواست از
عملیات GenAI برگرفته از API (`chat`، `generate_content` یا `text_completion`)
استفاده می‌کنند، درحالی‌که بازه‌های نوبت از `gen_ai.operation.name = invoke_agent` استفاده می‌کنند. هر دو در
`gen_ai.client.operation.duration` مشارکت دارند؛ در آنجا نام عملیات، تأخیر
درخواست مستقیم را از تأخیر کل نوبت جدا نگه می‌دارد. سنجه‌های فراخوانی مدل OTEL در
OpenClaw همچنین شامل `openclaw.model_call.observation_unit` هستند؛ سنجه‌های
فراخوانی مدل Prometheus برچسب معادل `observation_unit` را ارائه می‌کنند.

### صحت فراخوانی مدل Claude Code CLI

نوبت‌های Claude Code CLI یک بازهٔ مصنوعی `openclaw.model.call` در سطح نوبت
صادر می‌کنند. این‌ها بازه‌های درخواست HTTP ‏Anthropic نیستند. آن‌ها از
`openclaw.api =
claude-code` و `openclaw.model_call.observation_unit = turn` استفاده می‌کنند و عملیات را
`gen_ai.operation.name = invoke_agent` معرفی می‌کنند. آن‌ها مرز CLI ‏OpenClaw را از طریق
`openclaw.transport` مشخص می‌کنند:

- `stdio` - یک فرایند محلی یک‌بارهٔ Claude Code.
- `stdio-live` - یک نوبت در نشست پایدار مدیریت‌شدهٔ Claude stdio.
- `paired-node-cli` - اجرای یک‌بارهٔ Claude Code که به یک Node جفت‌شده
  واگذار شده است.

تشخیص‌های Claude CLI تنها زمانی نمونه‌سازی می‌شوند که توزیع‌کنندهٔ تشخیصی فرایند
فعال باشد و یک شنوندهٔ رویداد داخلی یا مورداعتماد متصل شده باشد. وقتی هیچ Plugin
مشاهده‌پذیری یا شنوندهٔ دیگری فعال نباشد، نوبت‌های Claude CLI از سلسله‌مراتب ردیابی
مصنوعی، میان‌گیرهای محتوا و حسابداری بایت‌های جریان تشخیصی صرف‌نظر می‌کنند. وقتی
ثبت محتوا فعال باشد، فیلدهای اعلان و اعلان سامانه هرکدام به 128 KiB محدود می‌شوند؛
خروجی دستیار در حداکثر 200 پوش، در مجموع به 128 KiB محدود می‌شود و 16 KiB و یک
مورد برای پاسخ جایگزین نهایی و قابل‌مشاهده رزرو می‌شود. هنگام رسیدن به حد، یک
نشانگر کوتاه‌شدن را ثبت می‌کند.

OpenClaw برای نوبت‌های Claude CLI همان سلسله‌مراتب مالکیتی را در نظر می‌گیرد که
سایر زمان‌های اجرای عامل استفاده می‌کنند: `openclaw.harness.run` (`openclaw.harness.id = claude-cli`)
شامل `openclaw.run` است که خود بازهٔ `openclaw.model.call` مربوط به Claude را
شامل می‌شود. بازه‌های مهار و اجرا، مرزهای مصنوعی نوبت OpenClaw هستند، نه
مرحله‌های داخلی Claude Code. نوبت‌های یک‌باره و stdio مدیریت‌شده از یک
سلسله‌مراتب استفاده می‌کنند؛ یک تلاش مجدد واقعی با نشست تازه، فرزند فراخوانی مدل
دیگری را در همان اجرای OpenClaw ایجاد می‌کند.

بازه هنگامی آغاز می‌شود که OpenClaw نوبت آماده‌شدهٔ CLI را می‌پذیرد و تنها پس از
موفقیت یا شکست آن نوبت پایان می‌یابد. برای نشست‌های مدیریت‌شده، تا وقتی Claude
عامل‌های پس‌زمینه یا جریان‌های کاری نگه‌دارندهٔ نتیجه را گزارش می‌کند، نتیجهٔ موفقیت
میانی بازه را پایان نمی‌دهد؛ نتیجهٔ نهایی پس از تخلیه آن را پایان می‌دهد. لغو، پایان
مهلت، شکست فرایند، شکست خروجی/تجزیه و سایر شکست‌های نوبت، همان بازه را با خطا
پایان می‌دهند.

Claude Code میزان مصرف هر پیام دستیار را گزارش می‌کند و ممکن است میزان مصرف انباشته
را نیز در نتیجهٔ پایانی خود گزارش کند. حسابداری پاسخ OpenClaw همچنان از آخرین پیام
دستیار استفاده می‌کند تا معناشناسی هزینهٔ موجود تغییر نکند؛ بازهٔ فراخوانی مدل در
سطح نوبت، در صورت وجود، از میزان مصرف انباشتهٔ پایانی، شامل توکن‌های خواندن کش و
ایجاد کش، استفاده می‌کند.

برای این بازه‌های CLI، فیلدهای بایت و زمان‌بندی، مرز قابل‌مشاهدهٔ CLI ‏OpenClaw را
توصیف می‌کنند:

- `openclaw.model_call.request_bytes` اندازهٔ UTF-8 مقدار اعلانی است که از طریق
  stdin/argv یک‌باره یا پوش کاربر JSONL در stdio مدیریت‌شده ارسال می‌شود. این
  اندازهٔ درخواست پنهان مدل Claude Code نیست.
- `openclaw.model_call.response_bytes` اندازهٔ UTF-8 خروجی stdout ‏Claude CLI است
  که طی نوبت مشاهده می‌شود. این اندازهٔ پاسخ HTTP ‏Anthropic نیست.
- `openclaw.model_call.time_to_first_byte_ms` زمان تا نخستین خروجی قابل‌مشاهدهٔ stdout یا
  stderr ‏Claude CLI است. این TTFB شبکه نیست.

با فعال‌بودن فیلدهای ریزدانهٔ منطبق `captureContent`، بازه اعلان مؤثری را که
OpenClaw به Claude Code می‌فرستد، اعلان سامانهٔ افزوده‌شدهٔ OpenClaw و متن/استدلال/
هویت فراخوانی ابزار قابل‌مشاهدهٔ دستیار را از طریق `gen_ai.input.messages`،
`gen_ai.output.messages` و `gen_ai.system_instructions` صادر می‌کند. آرگومان‌های ابزار،
امضاهای مبهم تفکر و نتایج ابزار از پوش دستیار Claude حذف می‌شوند. OpenClaw ادعا
نمی‌کند به اعلان سامانهٔ خصوصی Claude Code، بار درخواست پنهان ازسرگرفته‌شده یا
Compaction‌یافته، طرح‌واره‌های بومی ابزار داخلی، درخواست خام HTTP ‏Anthropic،
تلاش‌های مجدد داخلی، شناسهٔ درخواست بالادستی یا TTFB واقعی شبکه دسترسی دارد. چون
Claude Code تعریف‌های مؤثر ابزار بومی خود را با دقت ارائه نمی‌کند، این بازه‌ها
`gen_ai.tool.definitions` را پر نمی‌کنند.

بازه‌های ابزار مهار خارجی Claude حتی با فعال‌بودن ثبت محتوای ابزار، فقط شامل
فراداده باقی می‌مانند. همانند هر بازهٔ مدل، محتوای ثبت‌شدهٔ Claude CLI از مسیر
مختص شنوندهٔ مورداعتماد و حدود موجود حذف اطلاعات حساس و اندازه در صادرکننده استفاده
می‌کند؛ محتوا به‌طور پیش‌فرض غیرفعال می‌ماند.

## سنجه‌های صادرشده

### مصرف مدل

- `openclaw.tokens` (شمارنده، ویژگی‌ها: `openclaw.token`، `openclaw.channel`، `openclaw.provider`، `openclaw.model`، `openclaw.agent`)
- `openclaw.cost.usd` (شمارنده، ویژگی‌ها: `openclaw.channel`، `openclaw.provider`، `openclaw.model`)
- `openclaw.run.duration_ms` (هیستوگرام، ویژگی‌ها: `openclaw.channel`، `openclaw.provider`، `openclaw.model`)
- `openclaw.context.tokens` (هیستوگرام، ویژگی‌ها: `openclaw.context`، `openclaw.channel`، `openclaw.provider`، `openclaw.model`)
- `gen_ai.client.token.usage` (هیستوگرام، سنجهٔ قراردادهای معنایی GenAI، ویژگی‌ها: `gen_ai.token.type` = `input`/`output`، `gen_ai.provider.name`، `gen_ai.operation.name`، `gen_ai.request.model`)
- `gen_ai.client.operation.duration` (هیستوگرام، ثانیه، سنجهٔ قراردادهای معنایی GenAI برای درخواست‌های مدل و نوبت‌های مصنوعی عامل؛ ویژگی‌ها: `gen_ai.provider.name`، `gen_ai.operation.name`، `gen_ai.request.model`، `error.type` اختیاری؛ مشاهده‌های نوبت از `gen_ai.operation.name = invoke_agent` استفاده می‌کنند)
- `openclaw.model_call.duration_ms` (هیستوگرام، ویژگی‌ها: `openclaw.provider`، `openclaw.model`، `openclaw.api`، `openclaw.transport`، `openclaw.model_call.observation_unit`، به‌علاوهٔ `openclaw.errorCategory` و `openclaw.failureKind` برای خطاهای طبقه‌بندی‌شده)
- `openclaw.model_call.request_bytes` (هیستوگرام، اندازهٔ بایتی UTF-8 بار نهایی درخواست مدل؛ برای Claude Code CLI، ورودی/پوش اعلان قابل‌مشاهدهٔ شرح‌داده‌شده در بالا؛ بدون محتوای خام بار)
- `openclaw.model_call.response_bytes` (هیستوگرام، اندازهٔ بایتی UTF-8 بار قطعه‌های پاسخ جریانی؛ دلتاهای پرتکرار متن، تفکر و فراخوانی ابزار فقط بایت‌های افزایشی `delta` را می‌شمارند؛ برای Claude Code CLI، بایت‌های stdout مشاهده‌شده؛ بدون محتوای خام پاسخ)
- `openclaw.model_call.time_to_first_byte_ms` (هیستوگرام، زمان سپری‌شده پیش از نخستین رویداد پاسخ جریانی؛ برای Claude Code CLI، نخستین خروجی قابل‌مشاهدهٔ CLI به‌جای TTFB شبکه)
- `openclaw.model.failover` (شمارنده، ویژگی‌ها: `openclaw.provider`، `openclaw.model`، `openclaw.failover.to_provider`، `openclaw.failover.to_model`، `openclaw.failover.reason`، `openclaw.failover.suspended`، `openclaw.lane`)
- `openclaw.skill.used` (شمارنده، ویژگی‌ها: `openclaw.skill.name`، `openclaw.skill.source`، `openclaw.skill.activation`، `openclaw.agent` اختیاری، `openclaw.toolName` اختیاری)

### جریان پیام

- `openclaw.webhook.received` (شمارنده، ویژگی‌ها: `openclaw.channel`، `openclaw.webhook`)
- `openclaw.webhook.error` (شمارنده، ویژگی‌ها: `openclaw.channel`، `openclaw.webhook`)
- `openclaw.webhook.duration_ms` (هیستوگرام، ویژگی‌ها: `openclaw.channel`، `openclaw.webhook`)
- `openclaw.message.queued` (شمارنده، ویژگی‌ها: `openclaw.channel`، `openclaw.source`)
- `openclaw.message.received` (شمارنده، ویژگی‌ها: `openclaw.channel`، `openclaw.source`)
- `openclaw.message.dispatch.started` (شمارنده، ویژگی‌ها: `openclaw.channel`، `openclaw.source`)
- `openclaw.message.dispatch.completed` (شمارنده، ویژگی‌ها: `openclaw.channel`، `openclaw.outcome`، `openclaw.reason`، `openclaw.source`)
- `openclaw.message.dispatch.duration_ms` (هیستوگرام، ویژگی‌ها: `openclaw.channel`، `openclaw.outcome`، `openclaw.reason`، `openclaw.source`)
- `openclaw.message.processed` (شمارنده، ویژگی‌ها: `openclaw.channel`، `openclaw.outcome`)
- `openclaw.message.duration_ms` (هیستوگرام، ویژگی‌ها: `openclaw.channel`، `openclaw.outcome`)
- `openclaw.message.delivery.started` (شمارنده، ویژگی‌ها: `openclaw.channel`، `openclaw.delivery.kind`)
- `openclaw.message.delivery.duration_ms` (هیستوگرام، ویژگی‌ها: `openclaw.channel`، `openclaw.delivery.kind`، `openclaw.outcome`، `openclaw.errorCategory`)

### گفت‌وگو

- `openclaw.talk.event` (شمارنده، ویژگی‌ها: `openclaw.talk.event_type`، `openclaw.talk.mode`، `openclaw.talk.transport`، `openclaw.talk.brain`، `openclaw.talk.provider`)
- `openclaw.talk.event.duration_ms` (هیستوگرام، ویژگی‌ها: همانند `openclaw.talk.event`؛ هنگامی صادر می‌شود که یک رویداد گفت‌وگو مدت‌زمان را گزارش کند)
- `openclaw.talk.audio.bytes` (هیستوگرام، ویژگی‌ها: همانند `openclaw.talk.event`؛ برای رویدادهای فریم صوتی گفت‌وگو که طول بایت را گزارش می‌کنند صادر می‌شود)

### صف‌ها و نشست‌ها

- `openclaw.queue.lane.enqueue` (شمارنده، ویژگی‌ها: `openclaw.lane`)
- `openclaw.queue.lane.dequeue` (شمارنده، ویژگی‌ها: `openclaw.lane`)
- `openclaw.queue.depth` (هیستوگرام، ویژگی‌ها: `openclaw.lane` یا `openclaw.channel=heartbeat`)
- `openclaw.queue.wait_ms` (هیستوگرام، ویژگی‌ها: `openclaw.lane`)
- `openclaw.session.state` (شمارنده، ویژگی‌ها: `openclaw.state`، `openclaw.reason`)
- `openclaw.session.stuck` (شمارنده، ویژگی‌ها: `openclaw.state`؛ برای ثبت منقضی‌شدهٔ نشست که قابل بازیابی است منتشر می‌شود)
- `openclaw.session.stuck_age_ms` (هیستوگرام، ویژگی‌ها: `openclaw.state`؛ برای ثبت منقضی‌شدهٔ نشست که قابل بازیابی است منتشر می‌شود)
- `openclaw.session.turn.created` (شمارنده، ویژگی‌ها: `openclaw.agent`، `openclaw.channel`، `openclaw.trigger`)
- `openclaw.session.recovery.requested` (شمارنده، ویژگی‌ها: `openclaw.state`، `openclaw.action`، `openclaw.active_work_kind`، `openclaw.reason`)
- `openclaw.session.recovery.completed` (شمارنده، ویژگی‌ها: `openclaw.state`، `openclaw.action`، `openclaw.status`، `openclaw.active_work_kind`، `openclaw.reason`)
- `openclaw.session.recovery.age_ms` (هیستوگرام، ویژگی‌ها: همانند شمارندهٔ بازیابی متناظر)
- `openclaw.run.attempt` (شمارنده، ویژگی‌ها: `openclaw.attempt`)

### تله‌متری زنده‌بودن نشست

تا زمانی که OpenClaw پیشرفت پاسخ، ابزار، وضعیت، بلوک یا زمان اجرای ACP را مشاهده می‌کند، یک نشست `processing` به‌سوی آستانهٔ داخلی زنده‌بودن پیر نمی‌شود. سیگنال‌های زنده‌نگه‌داشتن تایپ به‌عنوان پیشرفت محسوب نمی‌شوند؛ بنابراین همچنان می‌توان مدل یا مهارِ بی‌صدا را تشخیص داد.

OpenClaw نشست‌ها را بر اساس کاری که همچنان می‌تواند مشاهده کند دسته‌بندی می‌کند:

- `session.long_running`: کار جاسازی‌شدهٔ فعال، فراخوانی‌های مدل یا فراخوانی‌های ابزار
  همچنان در حال پیشرفت‌اند. فراخوانی‌های بی‌صدای مدل که مالک دارند نیز پیش از آستانهٔ داخلی لغو به‌عنوان طولانی‌مدت گزارش می‌شوند؛ بنابراین ارائه‌دهندگان مدل کند یا بدون پخش جریانی، تا زمانی که امکان مشاهدهٔ لغو وجود دارد، شبیه نشست‌های متوقف‌شدهٔ Gateway به نظر نمی‌رسند.
- `session.stalled`: کار فعال وجود دارد، اما اجرای فعال
  پیشرفت اخیر را گزارش نکرده است. فراخوانی‌های مدل دارای مالک، در آستانهٔ داخلی لغو یا پس از آن، از `session.long_running` به
  `session.stalled` تغییر می‌کنند؛ فعالیت منقضی‌شدهٔ مدل/ابزارِ بدون مالک
  به‌عنوان کار طولانی‌مدت بی‌ضرر تلقی نمی‌شود.
  اجراهای جاسازی‌شدهٔ متوقف‌شده ابتدا فقط در حالت مشاهده باقی می‌مانند و سپس پس از
  آستانهٔ لغو و در نبود پیشرفت، لغو و تخلیه می‌شوند تا نوبت‌های صف‌شده در پشت آن مسیر بتوانند از سر گرفته شوند.
- `session.stuck`: ثبت منقضی‌شدهٔ نشست بدون کار فعال، یا یک نشست
  صف‌شدهٔ بیکار با فعالیت منقضی‌شدهٔ مدل/ابزارِ بدون مالک. پس از عبور از دروازه‌های بازیابی،
  مسیر نشست تحت‌تأثیر را فوراً آزاد می‌کند.

بازیابی، رویدادهای ساخت‌یافتهٔ `session.recovery.requested` و
`session.recovery.completed` را منتشر می‌کند. وضعیت تشخیصی نشست تنها
پس از یک نتیجهٔ بازیابی تغییردهنده (`aborted` یا `released`) و فقط در صورتی بیکار علامت‌گذاری می‌شود
که همان نسل پردازش همچنان جاری باشد.

فقط `session.stuck` شمارندهٔ `openclaw.session.stuck`،
هیستوگرام `openclaw.session.stuck_age_ms` و گسترهٔ `openclaw.session.stuck` را منتشر می‌کند.
تشخیص‌های تکراری `session.stuck` تا زمانی که نشست بدون تغییر باقی بماند با تأخیر فزاینده تکرار می‌شوند؛
بنابراین داشبوردها باید به‌جای هر تیک Heartbeat، دربارهٔ افزایش‌های پایدار
هشدار دهند. برای گزینهٔ پیکربندی و مقادیر پیش‌فرض، به
[مرجع پیکربندی](/fa/gateway/configuration-reference#diagnostics) مراجعه کنید.

هشدارهای زنده‌بودن همچنین موارد زیر را منتشر می‌کنند:

- `openclaw.liveness.warning` (شمارنده، ویژگی‌ها: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_delay_p99_ms` (هیستوگرام، ویژگی‌ها: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_delay_max_ms` (هیستوگرام، ویژگی‌ها: `openclaw.liveness.reason`)
- `openclaw.liveness.event_loop_utilization` (هیستوگرام، ویژگی‌ها: `openclaw.liveness.reason`)
- `openclaw.liveness.cpu_core_ratio` (هیستوگرام، ویژگی‌ها: `openclaw.liveness.reason`)

### چرخهٔ حیات مهار

- `openclaw.harness.duration_ms` (هیستوگرام، ویژگی‌ها: `openclaw.harness.id`، `openclaw.harness.plugin`، `openclaw.outcome`، و `openclaw.harness.phase` هنگام خطا)

### اجرای ابزار و تشخیص حلقه

- `openclaw.tool.execution.duration_ms` (هیستوگرام، ویژگی‌ها: `gen_ai.tool.name`، `openclaw.toolName`، `openclaw.tool.source`، `openclaw.tool.owner`، `openclaw.tool.params.kind`، به‌علاوهٔ `openclaw.errorCategory` هنگام خطا)
- `openclaw.tool.execution.blocked` (شمارنده، ویژگی‌ها: `gen_ai.tool.name`، `openclaw.toolName`، `openclaw.tool.source`، `openclaw.tool.owner`، `openclaw.tool.params.kind`، `openclaw.deniedReason`)
- `openclaw.tool.loop` (شمارنده، ویژگی‌ها: `openclaw.toolName`، `openclaw.loop.level`، `openclaw.loop.action`، `openclaw.loop.detector`، `openclaw.loop.count`، `openclaw.loop.paired_tool` اختیاری؛ هنگام تشخیص یک حلقهٔ تکراری فراخوانی ابزار منتشر می‌شود)

### اجرا

- `openclaw.exec.duration_ms` (هیستوگرام، ویژگی‌ها: `openclaw.exec.target`، `openclaw.exec.mode`، `openclaw.outcome`، `openclaw.failureKind`)

### سازوکارهای داخلی تشخیص (حافظه، محموله‌ها، سلامت صادرکننده)

- `openclaw.payload.large` (شمارنده، ویژگی‌ها: `openclaw.payload.surface`، `openclaw.payload.action`، `openclaw.channel`، `openclaw.plugin`، `openclaw.reason`)
- `openclaw.payload.large_bytes` (هیستوگرام، ویژگی‌ها: همانند `openclaw.payload.large`)
- `openclaw.memory.rss_bytes` / `openclaw.memory.heap_used_bytes` / `openclaw.memory.heap_total_bytes` / `openclaw.memory.external_bytes` / `openclaw.memory.array_buffers_bytes` (هیستوگرام‌ها، بدون ویژگی؛ نمونه‌های حافظهٔ فرایند)
- `openclaw.memory.pressure` (شمارنده، ویژگی‌ها: `openclaw.memory.level`، `openclaw.memory.reason`)
- `openclaw.diagnostic.async_queue.dropped` (شمارنده، ویژگی‌ها: `openclaw.diagnostic.async_queue.drop_class`؛ حذف‌های ناشی از فشار برگشتی صف تشخیصی داخلی)
- `openclaw.telemetry.exporter.events` (شمارنده، ویژگی‌ها: `openclaw.exporter`، `openclaw.signal`، `openclaw.status`، `openclaw.reason` اختیاری، `openclaw.errorCategory` اختیاری؛ خودتله‌متری چرخهٔ حیات/خرابی صادرکننده)

## گستره‌های صادرشده

- `openclaw.model.usage`
  - `openclaw.channel`، `openclaw.provider`، `openclaw.model`
  - `openclaw.tokens.*` (ورودی/خروجی/خواندن از حافظهٔ نهان/نوشتن در حافظهٔ نهان/مجموع)
  - `gen_ai.system` به‌طور پیش‌فرض، یا `gen_ai.provider.name` هنگامی که جدیدترین قراردادهای معنایی GenAI به‌صورت اختیاری فعال شوند
  - `gen_ai.request.model`، `gen_ai.operation.name`، `gen_ai.usage.*`
- `openclaw.run`
  - `openclaw.outcome`، `openclaw.channel`، `openclaw.provider`، `openclaw.model`، `openclaw.errorCategory`
- `openclaw.model.call`
  - `gen_ai.system` به‌طور پیش‌فرض، یا `gen_ai.provider.name` هنگامی که جدیدترین قراردادهای معنایی GenAI به‌صورت اختیاری فعال شوند
  - `gen_ai.request.model`، `gen_ai.operation.name`، `openclaw.provider`، `openclaw.model`، `openclaw.api`، `openclaw.transport`، `openclaw.model_call.observation_unit` (`request` یا `turn`)
  - `openclaw.errorCategory`، `error.type`، و `openclaw.failureKind` اختیاری هنگام خطا
  - `openclaw.model_call.request_bytes`، `openclaw.model_call.response_bytes`، `openclaw.model_call.time_to_first_byte_ms`
  - `openclaw.model_call.prompt.input_messages_count`، `openclaw.model_call.prompt.input_messages_chars`، `openclaw.model_call.prompt.system_prompt_chars`، `openclaw.model_call.prompt.tool_definitions_count`، `openclaw.model_call.prompt.tool_definitions_chars`، `openclaw.model_call.prompt.total_chars` (فقط اندازه‌های امن مؤلفه‌ها، بدون متن پرامپت)
  - `openclaw.model_call.usage.*` و `gen_ai.usage.*` هنگامی که نتیجه شامل میزان استفاده برای آن درخواست یا نوبت تجمیعی باشد
  - رویداد گسترهٔ `openclaw.provider.request` با ویژگی `openclaw.upstreamRequestIdHash` (محدود و مبتنی بر هش) هنگامی که نتیجهٔ ارائه‌دهندهٔ بالادستی یک شناسهٔ درخواست ارائه کند؛ شناسه‌های خام هرگز صادر نمی‌شوند
  - با `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`، گستره‌های درخواست از جدیدترین نام گسترهٔ استنتاج GenAI یعنی `{gen_ai.operation.name} {gen_ai.request.model}` استفاده می‌کنند. گستره‌های نوبت از `invoke_agent` استفاده می‌کنند، زیرا OpenClaw از مرز مبهم CLI ادعای نام عامل بومی نمی‌کند. هر دو به‌جای `openclaw.model.call` از نوع گسترهٔ `CLIENT` استفاده می‌کنند.
- `openclaw.harness.run`
  - `openclaw.harness.id`، `openclaw.harness.plugin`، `openclaw.outcome`، `openclaw.provider`، `openclaw.model`، `openclaw.channel`
  - هنگام تکمیل: `openclaw.harness.result_classification`، `openclaw.harness.yield_detected`، `openclaw.harness.items.started`، `openclaw.harness.items.completed`، `openclaw.harness.items.active`
  - هنگام خطا: `openclaw.harness.phase`، `openclaw.errorCategory`، `openclaw.harness.cleanup_failed` اختیاری
- `openclaw.tool.execution`
  - `gen_ai.tool.name`، `gen_ai.operation.name` (`execute_tool`)، `openclaw.toolName`، `openclaw.tool.source`، `gen_ai.tool.call.id` اختیاری، `openclaw.tool.owner`، `openclaw.tool.params.*`
  - `openclaw.errorCategory`/`openclaw.errorCode` اختیاری هنگام خطا، و `openclaw.deniedReason` و `openclaw.outcome=blocked` هنگام ردشدن توسط خط‌مشی یا محیط ایزوله
- `openclaw.exec`
  - `openclaw.exec.target`، `openclaw.exec.mode`، `openclaw.outcome`، `openclaw.failureKind`، `openclaw.exec.command_length`، `openclaw.exec.exit_code`، `openclaw.exec.exit_signal`، `openclaw.exec.timed_out`
- `openclaw.webhook.processed`
  - `openclaw.channel`، `openclaw.webhook`
- `openclaw.webhook.error`
  - `openclaw.channel`، `openclaw.webhook`، `openclaw.error`
- `openclaw.message.processed`
  - `openclaw.channel`، `openclaw.outcome`، `openclaw.reason`
- `openclaw.message.delivery`
  - `openclaw.channel`، `openclaw.delivery.kind`، `openclaw.outcome`، `openclaw.errorCategory`، `openclaw.delivery.result_count`
- `openclaw.session.stuck`
  - `openclaw.state`، `openclaw.ageMs`، `openclaw.queueDepth`
- `openclaw.context.assembled`
  - `openclaw.prompt.size`، `openclaw.history.size`، `openclaw.context.tokens`، `openclaw.errorCategory` (بدون محتوای پرامپت، تاریخچه، پاسخ یا کلید نشست)
- `openclaw.tool.loop`
  - `openclaw.toolName`، `openclaw.loop.level`، `openclaw.loop.action`، `openclaw.loop.detector`، `openclaw.loop.count`، `openclaw.loop.paired_tool` اختیاری (بدون پیام‌های حلقه، پارامترها یا خروجی ابزار)
- `openclaw.memory.pressure`
  - `openclaw.memory.level`، `openclaw.memory.reason`، `openclaw.memory.rss_bytes`، `openclaw.memory.heap_used_bytes`، `openclaw.memory.heap_total_bytes`، `openclaw.memory.external_bytes`، `openclaw.memory.array_buffers_bytes`، `openclaw.memory.threshold_bytes`/`openclaw.memory.rss_growth_bytes`/`openclaw.memory.window_ms` اختیاری

هنگامی که ثبت محتوا صراحتاً فعال شده باشد، گستره‌های مدل و ابزار می‌توانند
ویژگی‌های محدود و ویرایش‌شدهٔ `openclaw.content.*` را نیز برای رده‌های مشخص
محتوایی که فعال کرده‌اید شامل شوند.

## فهرست رویدادهای تشخیصی

رویدادهای زیر از معیارها و گستره‌های بالا پشتیبانی می‌کنند یا برای اشتراک مستقیم
Plugin در دسترس‌اند. `run.progress` و `run.execution_phase` سیگنال‌های چرخهٔ حیات
صرفاً مستقیم هستند؛ Plugin ‏diagnostics-otel آن‌ها را به‌عنوان
سیگنال‌های مستقل OTLP صادر نمی‌کند. انواع رویداد و مقادیر `run.execution_phase.phase`
افزایشی هستند. مصرف‌کنندگان TypeScript باید به‌جای فرض دائماً جامع‌بودن
هر یک از unionها، شاخه‌های پیش‌فرض را حفظ کنند.

**میزان استفاده از مدل**

- `model.usage` - توکن‌ها، هزینه، مدت، زمینه، ارائه‌دهنده/مدل/کانال،
  شناسه‌های نشست. `usage` حسابداری ارائه‌دهنده/نوبت برای هزینه و تله‌متری است؛
  `context.used` تصویر لحظه‌ای پرامپت/زمینهٔ جاری است و هنگام دخیل‌بودن ورودی حافظهٔ نهان
  یا فراخوانی‌های حلقهٔ ابزار می‌تواند از `usage.total` ارائه‌دهنده کمتر باشد.

**جریان پیام**

- `webhook.received` / `webhook.processed` / `webhook.error`
- `message.queued` / `message.processed`
- `message.delivery.started` / `message.delivery.completed` / `message.delivery.error`

**صف و نشست**

- `queue.lane.enqueue` / `queue.lane.dequeue`
- `session.state` / `session.long_running` / `session.stalled` / `session.stuck`
- `run.attempt` / `run.progress`
- `run.execution_phase` (نقاط عطف عمومی و مرتبط با نشستِ راه‌اندازی اجراکنندهٔ جاسازی‌شده)
- `diagnostic.heartbeat` (شمارنده‌های تجمیعی: Webhookها/صف/نشست)

**چرخهٔ حیات مهار**

- `harness.run.started` / `harness.run.completed` / `harness.run.error` -
  چرخهٔ حیات به‌ازای هر اجرا برای مهار عامل. شامل `harnessId`، مقدار اختیاری
  `pluginId`، ارائه‌دهنده/مدل/کانال و شناسهٔ اجرا است. تکمیل، شمارش‌های
  `durationMs`، `outcome`، مقدار اختیاری `resultClassification`، `yieldDetected`
  و `itemLifecycle` را می‌افزاید. خطاها، `phase`
  (`prepare`/`start`/`send`/`resolve`/`cleanup`)، `errorCategory` و
  مقدار اختیاری `cleanupFailed` را می‌افزایند.

**اجرا**

- `exec.process.completed` - نتیجهٔ ترمینال، مدت‌زمان، مقصد، حالت، کد
  خروج و نوع خرابی. متن فرمان و دایرکتوری‌های کاری
  درج نمی‌شوند.
- `exec.approval.followup_suppressed` - پیگیری تأیید منقضی‌شده که
  پس از اتصال مجدد نشست حذف شد. شامل `approvalId`، `reason`
  (`session_rebound`)، `phase` (`direct_delivery` یا `gateway_preflight`)
  و مهر زمانی توزیع‌کننده است. کلیدهای نشست، مسیرها و متن فرمان
  درج نمی‌شوند.

## بدون صادرکننده

رویدادهای عیب‌یابی را بدون اجرای
`diagnostics-otel` برای Pluginها یا مقصدهای سفارشی در دسترس نگه دارید:

```json5
{
  diagnostics: { enabled: true },
}
```

برای خروجی اشکال‌زدایی هدفمند بدون افزایش `logging.level`، از پرچم‌های
عیب‌یابی استفاده کنید. پرچم‌ها به بزرگی و کوچکی حروف حساس نیستند و از نویسه‌های عام (`telegram.*` یا
`*`) پشتیبانی می‌کنند:

```json5
{
  diagnostics: { flags: ["telegram.http"] },
}
```

یا به‌صورت بازنویسی یک‌بارهٔ متغیر محیطی:

```bash
OPENCLAW_DIAGNOSTICS=telegram.http,telegram.payload openclaw gateway
```

خروجی پرچم در فایل گزارش استاندارد (`logging.file`) ثبت می‌شود و همچنان
توسط `logging.redactSensitive` پوشانده می‌شود. راهنمای کامل:
[پرچم‌های عیب‌یابی](/fa/diagnostics/flags).

## غیرفعال‌سازی

```json5
{
  diagnostics: { otel: { enabled: false } },
}
```

یا `diagnostics-otel` را در `plugins.allow` قرار ندهید، یا
`openclaw plugins disable diagnostics-otel` را اجرا کنید.

## مرتبط

- [گزارش‌گیری](/fa/logging) - گزارش‌های فایل، خروجی کنسول، دنبال‌کردن از طریق CLI و زبانهٔ گزارش‌های رابط کاربری کنترل
- [جزئیات داخلی گزارش‌گیری Gateway](/fa/gateway/logging) - سبک‌های گزارش WS، پیشوندهای زیرسامانه و ضبط کنسول
- [پرچم‌های عیب‌یابی](/fa/diagnostics/flags) - پرچم‌های هدفمند گزارش اشکال‌زدایی
- [صدور عیب‌یابی](/fa/gateway/diagnostics) - ابزار بستهٔ پشتیبانی اپراتور (جدا از صدور OTEL)
- [مرجع پیکربندی](/fa/gateway/configuration-reference#diagnostics) - مرجع کامل فیلد `diagnostics.*`
