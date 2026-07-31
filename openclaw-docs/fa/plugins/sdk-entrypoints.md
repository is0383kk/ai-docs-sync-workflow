---
read_when:
    - به امضای نوع دقیق `defineToolPlugin`،‏ `definePluginEntry` یا `defineChannelPluginEntry` نیاز دارید
    - می‌خواهید حالت ثبت را درک کنید (کامل در برابر راه‌اندازی در برابر فرادادهٔ CLI)
    - در حال جست‌وجوی گزینه‌های نقطه ورود هستید
sidebarTitle: Entry Points
summary: مرجع defineToolPlugin، definePluginEntry، defineChannelPluginEntry و defineSetupPluginEntry
title: نقاط ورود Plugin
x-i18n:
    generated_at: "2026-07-27T17:00:04Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e64fe1d65531fea8f266aa23b73064daf2ed2c5c43af8bb08ea57e347fe566f4
    source_path: plugins/sdk-entrypoints.md
    workflow: 16
---

هر Plugin یک شیء ورودی پیش‌فرض صادر می‌کند. SDK برای
هر شکل ورودی یک تابع کمکی ارائه می‌دهد: `defineToolPlugin`، `definePluginEntry`،
`defineChannelPluginEntry`، `defineSetupPluginEntry`.

<Tip>
  **به‌دنبال یک راهنمای گام‌به‌گام هستید؟** برای راهنماهای مرحله‌به‌مرحله، به [Pluginهای ابزار](/fa/plugins/tool-plugins)،
  [Pluginهای کانال](/fa/plugins/sdk-channel-plugins) یا
  [Pluginهای ارائه‌دهنده](/fa/plugins/sdk-provider-plugins) مراجعه کنید.
</Tip>

## ورودی‌های بسته

Pluginهای نصب‌شده، فیلدهای `package.json` `openclaw` را هم به ورودی‌های منبع و هم
به ورودی‌های ساخته‌شده ارجاع می‌دهند:

```json
{
  "openclaw": {
    "extensions": ["./src/index.ts"],
    "runtimeExtensions": ["./dist/index.js"],
    "setupEntry": "./src/setup-entry.ts",
    "runtimeSetupEntry": "./dist/setup-entry.js"
  }
}
```

- `extensions` و `setupEntry` ورودی‌های منبع هستند که برای توسعه در فضای کاری و
  checkout گیت استفاده می‌شوند.
- `runtimeExtensions` و `runtimeSetupEntry` برای بسته‌های
  نصب‌شده ترجیح داده می‌شوند: آن‌ها به بسته‌های npm امکان می‌دهند از کامپایل TypeScript در زمان اجرا صرف‌نظر کنند.
- `runtimeExtensions`، در صورت وجود، باید از نظر طول آرایه با `extensions`
  مطابقت داشته باشد (ورودی‌ها بر اساس موقعیت جفت می‌شوند). `runtimeSetupEntry` به `setupEntry` نیاز دارد.
- اگر یک آرتیفکت `runtimeExtensions`/`runtimeSetupEntry` اعلام شده باشد اما
  وجود نداشته باشد، نصب/کشف با خطای بسته‌بندی شکست می‌خورد؛ OpenClaw
  بی‌سروصدا به منبع بازنمی‌گردد. بازگشت به منبع (در ادامه) فقط زمانی اعمال می‌شود که
  هیچ ورودی زمان اجرایی اعلام نشده باشد.
- اگر یک بسته نصب‌شده فقط یک ورودی منبع TypeScript اعلام کند، OpenClaw
  به‌دنبال همتای ساخته‌شده و منطبق `dist/*.js` (یا `.mjs`/`.cjs`) می‌گردد و از آن استفاده می‌کند؛
  در غیر این صورت به منبع TypeScript بازمی‌گردد.
- همه مسیرهای ورودی باید داخل دایرکتوری بسته Plugin باقی بمانند. ورودی‌های زمان اجرا
  و همتاهای ساخته‌شده JS که استنتاج شده‌اند، یک مسیر منبع `extensions` یا
  `setupEntry` خارج‌شونده را معتبر نمی‌کنند.

## `defineToolPlugin`

**واردکردن:** `openclaw/plugin-sdk/tool-plugin`

برای Pluginهایی که فقط ابزارهای عامل را اضافه می‌کنند. منبع را کوچک نگه می‌دارد، نوع‌های پیکربندی
و پارامترهای ابزار را از شِمای TypeBox استنتاج می‌کند، مقادیر بازگشتی ساده را در
قالب نتیجه ابزار OpenClaw می‌پیچد و فراداده ایستایی را در دسترس قرار می‌دهد که
`openclaw plugins build` در مانیفست Plugin می‌نویسد (`contracts.tools`،
`configSchema`).

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "Fetch stock quotes.",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "API key." })),
  }),
  tools: (tool) => [
    tool({
      name: "quote",
      label: "Quote",
      description: "Fetch a quote.",
      parameters: Type.Object({
        symbol: Type.String({ description: "Ticker symbol." }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          hasKey: Type.Boolean(),
        },
        { additionalProperties: false },
      ),
      execute: async ({ symbol }, config) => ({ symbol, hasKey: Boolean(config.apiKey) }),
    }),
  ],
});
```

- `configSchema` اختیاری است؛ حذف آن از یک شِمای سخت‌گیرانه شیء خالی استفاده می‌کند
  (مانیفست تولیدشده همچنان شامل `configSchema` است).
- `execute` یک رشته ساده یا مقدار قابل سریال‌سازی به JSON برمی‌گرداند؛ تابع کمکی
  آن را به‌صورت یک نتیجه ابزار متنی می‌پیچد و `details` را روی مقدار بازگشتی اصلی
  (تبدیل‌نشده به رشته) تنظیم می‌کند.
- `outputSchema` در صورت نیاز آن مقدار اصلی `details` را برای حالت کد
  و جست‌وجوی ابزار توصیف می‌کند. فراخوانی‌های کاتالوگ، شِمای نامعتبر را پیش از اجرا رد می‌کنند
  و مقدار نهایی را پیش از بازگرداندن اعتبارسنجی می‌کنند.
- برای نتایج ابزار سفارشی، `openclaw/plugin-sdk/tool-results`،
  `textResult` و `jsonResult` را صادر می‌کند.
- نام ابزارها ایستا است، بنابراین `openclaw plugins build`،
  `contracts.tools` را بدون تکرار دستی نام‌ها از ابزارهای اعلام‌شده استخراج می‌کند.
- بارگذاری زمان اجرا سخت‌گیرانه باقی می‌ماند: Pluginهای نصب‌شده همچنان به
  `openclaw.plugin.json` و `package.json` `openclaw.extensions` نیاز دارند. OpenClaw
  هرگز کد Plugin را برای استنتاج داده‌های مفقود مانیفست اجرا نمی‌کند.

## `definePluginEntry`

**واردکردن:** `openclaw/plugin-sdk/plugin-entry`

برای Pluginهای ارائه‌دهنده، Pluginهای ابزار پیشرفته، Pluginهای هوک و هر چیزی که
یک کانال پیام‌رسانی **نیست**.

```typescript
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";

export default definePluginEntry({
  id: "my-plugin",
  name: "My Plugin",
  description: "Short summary",
  register(api) {
    api.registerProvider({/* ... */});
    api.registerTool({/* ... */});
  },
});
```

| فیلد                     | نوع                                                             | الزامی | پیش‌فرض             |
| ------------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                      | `string`                                                         | بله      | -                   |
| `name`                    | `string`                                                         | بله      | -                   |
| `description`             | `string`                                                         | بله      | -                   |
| `kind`                    | `string` (منسوخ، ادامه را ببینید)                                 | خیر       | -                   |
| `configSchema`            | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | خیر       | شِمای شیء خالی |
| `reload`                  | `OpenClawPluginReloadRegistration`                               | خیر       | -                   |
| `nodeHostCommands`        | `OpenClawPluginNodeHostCommand[]`                                | خیر       | -                   |
| `securityAuditCollectors` | `OpenClawPluginSecurityAuditCollector[]`                         | خیر       | -                   |
| `register`                | `(api: OpenClawPluginApi) => void`                               | بله      | -                   |

- `id` باید با مانیفست `openclaw.plugin.json` شما مطابقت داشته باشد.
- کاتالوگ‌های نشست خارجی از
  `openclaw/plugin-sdk/session-catalog` و
  `api.registerSessionCatalog({ id, label, list, read, continueSession?, archive? })` استفاده می‌کنند.
  هسته مالک متدهای Gateway در `sessions.catalog.*` است؛ ارائه‌دهندگان، نگاشت‌های میزبان،
  نشست و رونوشت نرمال‌شده را بدون ثبت RPCها برمی‌گردانند. ارائه‌دهنده فهرست باید
  با نهایی‌شدن هر میزبان، callback اختیاری `onHost(host)` را فراخوانی کند؛ آرایه میزبان
  بازگشتی همچنان به‌عنوان تصویر لحظه‌ای نهایی سازگاری الزامی است.
- `kind` منسوخ شده است: به‌جای آن یک جایگاه انحصاری (`"memory"` یا
  `"context-engine"`) را در فیلد `kind` مانیفست `openclaw.plugin.json`
  اعلام کنید. `kind` ورودی زمان اجرا فقط به‌عنوان راهکار بازگشت سازگاری برای
  Pluginهای قدیمی‌تر باقی می‌ماند.
- `configSchema` می‌تواند برای ارزیابی تنبل یک تابع باشد. OpenClaw شِما را
  هنگام نخستین دسترسی حل و ذخیره می‌کند، بنابراین سازنده‌های پرهزینه شِما فقط یک‌بار
  اجرا می‌شوند.
- یک توصیفگر `nodeHostCommands` می‌تواند `isAvailable({ config, env })` را تعریف کند.
  بازگرداندن `false` آن فرمان و قابلیت آن را از اعلان Gateway در Node
  بدون رابط حذف می‌کند. OpenClaw آن را در برابر پیکربندی راه‌اندازی محلی Node
  ارزیابی می‌کند؛ کنترل‌کننده‌های فرمان همچنان باید هنگام فراخوانی، دردسترس‌بودن را
  اعتبارسنجی کنند.

## `defineChannelPluginEntry`

**واردکردن:** `openclaw/plugin-sdk/channel-core`

`definePluginEntry` را با سیم‌کشی ویژه کانال می‌پیچد: به‌طور خودکار
`api.registerChannel({ plugin })` را فراخوانی می‌کند، یک درگاه اختیاری فراداده CLI برای راهنمای ریشه
در دسترس قرار می‌دهد و `registerFull` را بر اساس حالت ثبت محدود می‌کند.

```typescript
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineChannelPluginEntry({
  id: "my-channel",
  name: "My Channel",
  description: "Short summary",
  plugin: myChannelPlugin,
  setRuntime: setMyRuntime,
  registerCliMetadata(api) {
    api.registerCli(/* ... */);
  },
  registerFull(api) {
    api.registerGatewayMethod(/* ... */);
  },
});
```

| فیلد                 | نوع                                                             | الزامی | پیش‌فرض             |
| --------------------- | ---------------------------------------------------------------- | -------- | ------------------- |
| `id`                  | `string`                                                         | بله      | -                   |
| `name`                | `string`                                                         | بله      | -                   |
| `description`         | `string`                                                         | بله      | -                   |
| `plugin`              | `ChannelPlugin`                                                  | بله      | -                   |
| `configSchema`        | `OpenClawPluginConfigSchema \| () => OpenClawPluginConfigSchema` | خیر       | شِمای شیء خالی |
| `setRuntime`          | `(runtime: PluginRuntime) => void`                               | خیر       | -                   |
| `registerCliMetadata` | `(api: OpenClawPluginApi) => void`                               | خیر       | -                   |
| `registerFull`        | `(api: OpenClawPluginApi) => void`                               | خیر       | -                   |

callbackها بر اساس حالت ثبت اجرا می‌شوند (جدول کامل در
[حالت ثبت](#registration-mode)):

- `setRuntime` در همه حالت‌ها به‌جز `"cli-metadata"` و
  `"tool-discovery"` اجرا می‌شود. ارجاع زمان اجرا را در اینجا، معمولاً از طریق
  `createPluginRuntimeStore`، ذخیره کنید.
- `registerCliMetadata` برای `"cli-metadata"`، `"discovery"` و
  `"full"` اجرا می‌شود. از آن به‌عنوان محل مرجع برای توصیفگرهای CLI متعلق به کانال
  استفاده کنید تا راهنمای ریشه بدون فعال‌سازی باقی بماند، تصاویر لحظه‌ای کشف شامل فراداده ایستای
  فرمان باشند و ثبت عادی CLI با بارگذاری کامل Plugin سازگار باقی بماند.
- `registerFull` فقط برای `"full"` و `"tool-discovery"` اجرا می‌شود. برای
  `"tool-discovery"`، این تابع _به‌جای_ ثبت کانال اجرا می‌شود: OpenClaw
  `registerChannel`/`setRuntime` را کاملاً نادیده می‌گیرد و فقط
  `registerFull` را فراخوانی می‌کند؛ بنابراین هرگونه ثبت ارائه‌دهنده/ابزاری که کانال شما برای
  کشف یا اجرای مستقل ابزار نیاز دارد، باید در آنجا قرار گیرد، نه پشت راه‌اندازی عادی
  کانال.
- ثبت کشف، غیرفعال‌کننده است نه بدون واردکردن: OpenClaw ممکن است
  ورودی Plugin مورد اعتماد و ماژول Plugin کانال را برای ساخت تصویر لحظه‌ای
  ارزیابی کند. importهای سطح بالا را بدون اثر جانبی نگه دارید و سوکت‌ها،
  کلاینت‌ها، workerها و سرویس‌ها را پشت مسیرهای مخصوص `"full"` قرار دهید.
- همانند `definePluginEntry`، `configSchema` می‌تواند یک سازنده تنبل باشد؛ OpenClaw
  شِمای حل‌شده را هنگام نخستین دسترسی ذخیره می‌کند.

ثبت CLI:

- از `api.registerCli(..., { descriptors: [...] })` برای فرمان‌های ریشهٔ CLI متعلق به Plugin که می‌خواهید بدون ناپدیدشدن از درخت تجزیهٔ CLI ریشه به‌صورت تنبل بارگذاری شوند، استفاده کنید. نام توصیف‌گرها باید فقط شامل حروف، اعداد، خط تیره و زیرخط باشد و با حرف یا عدد آغاز شود؛ OpenClaw شکل‌های دیگر را رد می‌کند و پیش از نمایش راهنما، توالی‌های کنترل پایانه را از توضیحات حذف می‌کند. همهٔ ریشه‌های فرمان سطح بالایی را که ثبت‌کننده ارائه می‌دهد، پوشش دهید.
  `commands` به‌تنهایی در مسیر سازگاری بارگذاری فوری باقی می‌ماند.
- از `api.registerNodeCliFeature(...)` برای فرمان‌های قابلیت Node جفت‌شده استفاده کنید تا
  زیر `openclaw nodes` قرار گیرند (معادل
  `registerCli(registrar, { parentPath: ["nodes"], ... })`).
- برای دیگر فرمان‌های تودرتوی Plugin، `parentPath` را اضافه کنید و فرمان‌ها را
  روی شیء `program` که به ثبت‌کننده ارسال می‌شود ثبت کنید؛ OpenClaw پیش از
  فراخوانی Plugin، آن را به فرمان والد تبدیل می‌کند.
- برای Pluginهای کانال، توصیف‌گرهای CLI را از `registerCliMetadata`
  ثبت کنید و تمرکز `registerFull` را فقط بر کارهای زمان اجرا نگه دارید.
- اگر `registerFull` متدهای RPC مربوط به Gateway را نیز ثبت می‌کند، آن‌ها را زیر یک
  پیشوند مختص Plugin نگه دارید. فضای نام‌های مدیریتی رزروشدهٔ هسته (`config.*`،
  `exec.approvals.*`، `wizard.*`، `update.*`) همیشه به
  `operator.admin` تبدیل می‌شوند.

## `defineSetupPluginEntry`

**واردسازی:** `openclaw/plugin-sdk/channel-core`

برای فایل سبک‌وزن `setup-entry.ts`. فقط `{ plugin }` را بدون
اتصال زمان اجرا یا CLI برمی‌گرداند.

```typescript
import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";

export default defineSetupPluginEntry(myChannelPlugin);
```

هنگامی که کانالی غیرفعال یا پیکربندی‌نشده است، یا بارگذاری با تأخیر فعال باشد، OpenClaw
این ورودی را به‌جای ورودی کامل بارگذاری می‌کند. برای اطلاع از موارد اهمیت این موضوع، به
[راه‌اندازی و پیکربندی](/fa/plugins/sdk-setup#setup-entry) مراجعه کنید.

`defineSetupPluginEntry(...)` را با خانواده‌های محدودِ دستیار راه‌اندازی همراه کنید:

| واردسازی                              | کاربرد                                                                                                                                                                            |
| ----------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `openclaw/plugin-sdk/setup-runtime` | دستیارهای راه‌اندازی ایمن برای زمان اجرا: `createSetupTranslator`، آداپتورهای وصلهٔ راه‌اندازی ایمن برای واردسازی، خروجی یادداشت جست‌وجو، `promptResolvedAllowFrom`، `splitSetupEntries`، پراکسی‌های راه‌اندازی تفویض‌شده |
| `openclaw/plugin-sdk/channel-setup` | سطوح راه‌اندازی نصب اختیاری                                                                                                                                                    |
| `openclaw/plugin-sdk/setup-tools`   | دستیارهای CLI راه‌اندازی/نصب، بایگانی و مستندات                                                                                                                                       |

SDKهای سنگین، ثبت CLI و سرویس‌های زمان اجرای بلندمدت را در
ورودی کامل نگه دارید.

کانال‌های همراهِ فضای کاری که سطوح راه‌اندازی و زمان اجرا را جدا می‌کنند، می‌توانند به‌جای آن از
`defineBundledChannelSetupEntry(...)` در
`openclaw/plugin-sdk/channel-entry-contract` استفاده کنند. این امکان را می‌دهد که ورودی راه‌اندازی،
خروجی‌های Plugin/اسرارِ ایمن برای راه‌اندازی را نگه دارد و در عین حال یک تنظیم‌کنندهٔ زمان اجرا
ارائه کند:

```typescript
import { defineBundledChannelSetupEntry } from "openclaw/plugin-sdk/channel-entry-contract";

export default defineBundledChannelSetupEntry({
  importMetaUrl: import.meta.url,
  plugin: {
    specifier: "./channel-plugin-api.js",
    exportName: "myChannelPlugin",
  },
  runtime: {
    specifier: "./runtime-api.js",
    exportName: "setMyChannelRuntime",
  },
  registerSetupRuntime(api) {
    api.registerHttpRoute({
      path: "/my-channel/events",
      auth: "plugin",
      handler: async (req, res) => {
        /* مسیر ایمن برای راه‌اندازی */
      },
    });
  },
});
```

فقط زمانی از این استفاده کنید که جریان راه‌اندازی واقعاً پیش از بارگذاری ورودی کامل کانال،
به یک تنظیم‌کنندهٔ سبک‌وزن زمان اجرا یا سطح Gateway ایمن برای راه‌اندازی نیاز داشته باشد.
`registerSetupRuntime` فقط برای بارگذاری‌های `"setup-runtime"` اجرا می‌شود؛ آن را
به مسیرها یا متدهای صرفاً پیکربندی که باید پیش از فعال‌سازی کامل با تأخیر وجود داشته باشند
محدود کنید.

## حالت ثبت

`api.registrationMode` به Plugin شما می‌گوید چگونه بارگذاری شده است:

| حالت               | زمان                                               | موارد قابل ثبت                                                                                                        |
| ------------------ | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `"full"`           | راه‌اندازی عادی Gateway                             | همه‌چیز                                                                                                              |
| `"discovery"`      | کشف قابلیت فقط‌خواندنی                     | ثبت کانال به‌همراه توصیف‌گرهای ایستای CLI؛ کد ورودی ممکن است بارگذاری شود، اما سوکت‌ها، کارکنان، کلاینت‌ها و سرویس‌ها را رد کنید |
| `"tool-discovery"` | بارگذاری محدود برای فهرست‌کردن یا اجرای ابزارهای Pluginهای مشخص | فقط ثبت قابلیت/ابزار؛ بدون فعال‌سازی کانال                                                                |
| `"setup-only"`     | کانال غیرفعال/پیکربندی‌نشده                      | فقط ثبت کانال                                                                                               |
| `"setup-runtime"`  | جریان راه‌اندازی با زمان اجرای در دسترس                  | ثبت کانال به‌همراه فقط زمان اجرای سبک‌وزن موردنیاز پیش از بارگذاری ورودی کامل                               |
| `"cli-metadata"`   | راهنمای ریشه / ثبت فرادادهٔ CLI                   | فقط توصیف‌گرهای CLI                                                                                                    |

`defineChannelPluginEntry` این تفکیک را به‌طور خودکار مدیریت می‌کند. اگر برای یک کانال مستقیماً از
`definePluginEntry` استفاده می‌کنید، حالت را خودتان بررسی کنید و به یاد داشته باشید که
`"tool-discovery"` ثبت کانال را رد می‌کند:

```typescript
register(api) {
  if (
    api.registrationMode === "cli-metadata" ||
    api.registrationMode === "discovery" ||
    api.registrationMode === "full"
  ) {
    api.registerCli(/* ... */);
    if (api.registrationMode === "cli-metadata") return;
  }

  if (api.registrationMode === "tool-discovery") {
    // فقط سطوح قابلیت (ارائه‌دهندگان/ابزارها) را ثبت کنید، نه کانال را.
    return;
  }

  api.registerChannel({ plugin: myPlugin });
  if (api.registrationMode !== "full") return;

  // ثبت‌های سنگینِ مختص زمان اجرا
  api.registerService(/* ... */);
}
```

سرویس‌های بلندمدت می‌توانند رویدادهای کوچک ابطال یا چرخهٔ عمر را از طریق
زمینهٔ سرویس خود منتشر کنند:

```typescript
api.registerService({
  id: "index-events",
  start(ctx) {
    ctx.gatewayEvents?.emit("changed", { revision: 1 }, { scope: "operator.read" });
  },
});
```

OpenClaw این را با فضای نام `plugin.<plugin-id>.changed` ارائه می‌کند. نام رویدادها از یک
بخش با حروف کوچک تشکیل می‌شوند، بارهای داده باید JSON با اندازهٔ محدود باشند و دامنه باید
`operator.read`، `operator.write` یا `operator.admin` باشد. منتشرکننده فقط
در طول عمر سرویس وجود دارد و پس از توقف یا راه‌اندازی ناموفق لغو می‌شود. بارهای دادهٔ
نسخه یا ابطال را به رکوردهای کامل ترجیح دهید تا کلاینت‌های مجاز، وضعیت مرجع را از طریق
متدهای محدود Gateway متعلق به Plugin دوباره بخوانند.

حالت کشف یک تصویر لحظه‌ای غیرفعال‌کننده از رجیستری می‌سازد. این حالت ممکن است همچنان
ورودی Plugin و شیء Plugin کانال را ارزیابی کند تا OpenClaw بتواند
قابلیت‌های کانال و توصیف‌گرهای ایستای CLI را ثبت کند. ارزیابی ماژول
در حالت کشف را قابل‌اعتماد اما سبک‌وزن در نظر بگیرید: بدون کلاینت‌های شبکه،
زیرفرایندها، شنونده‌ها، اتصال‌های پایگاه داده، کارکنان پس‌زمینه،
خواندن اعتبارنامه‌ها یا دیگر عوارض جانبی زندهٔ زمان اجرا در سطح بالایی.

`"setup-runtime"` را بازه‌ای در نظر بگیرید که سطوح راه‌اندازی مختص راه‌اندازی باید
بدون ورود دوباره به زمان اجرای کامل کانال همراه وجود داشته باشند. گزینه‌های مناسب شامل
ثبت کانال، مسیرهای HTTP ایمن برای راه‌اندازی، متدهای Gateway ایمن برای راه‌اندازی
و دستیارهای راه‌اندازی تفویض‌شده هستند. سرویس‌های سنگین پس‌زمینه، ثبت‌کننده‌های CLI و
راه‌اندازی اولیهٔ SDK ارائه‌دهنده/کلاینت همچنان به `"full"` تعلق دارند.

## شکل‌های Plugin

OpenClaw، Pluginهای بارگذاری‌شده را بر اساس رفتار ثبت آن‌ها دسته‌بندی می‌کند:

| شکل                 | توضیح                                        |
| --------------------- | -------------------------------------------------- |
| **قابلیت ساده**  | یک نوع قابلیت (برای مثال، فقط ارائه‌دهنده)           |
| **قابلیت ترکیبی** | چند نوع قابلیت (برای مثال، ارائه‌دهنده + گفتار) |
| **فقط هوک**         | فقط هوک‌ها، بدون قابلیت                        |
| **بدون قابلیت**    | ابزارها/فرمان‌ها/سرویس‌ها، اما بدون قابلیت        |

برای مشاهدهٔ شکل یک Plugin از `openclaw plugins inspect <id>` استفاده کنید.

## مرتبط

- [نمای کلی SDK](/fa/plugins/sdk-overview) - API ثبت و مرجع زیرمسیر
- [دستیارهای زمان اجرا](/fa/plugins/sdk-runtime) - `api.runtime` و `createPluginRuntimeStore`
- [راه‌اندازی و پیکربندی](/fa/plugins/sdk-setup) - مانیفست، ورودی راه‌اندازی، بارگذاری با تأخیر
- [Pluginهای کانال](/fa/plugins/sdk-channel-plugins) - ساخت شیء `ChannelPlugin`
- [Pluginهای ارائه‌دهنده](/fa/plugins/sdk-provider-plugins) - ثبت ارائه‌دهنده و هوک‌ها
