---
read_when:
    - می‌خواهید یک Plugin ساده برای OpenClaw بسازید که فقط ابزارهای عامل را اضافه کند
    - می‌خواهید به‌جای نوشتن دستی فراداده‌های مانیفست Plugin، از defineToolPlugin استفاده کنید
    - باید یک Plugin صرفاً ابزاری را چارچوب‌بندی، تولید، اعتبارسنجی، آزمایش یا منتشر کنید
sidebarTitle: Tool Plugins
summary: ابزارهای ساده و نوع‌دار عامل را با defineToolPlugin و openclaw plugins init/build/validate بسازید
title: Pluginهای ابزار
x-i18n:
    generated_at: "2026-07-27T14:31:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ac23d15ba79cbdd1d8b8eab7c87007b44af16361b2866b14123e18f816bf4075
    source_path: plugins/tool-plugins.md
    workflow: 16
---

`defineToolPlugin` یک Plugin می‌سازد که فقط ابزارهای قابل فراخوانی توسط عامل را اضافه می‌کند: بدون
کانال، ارائه‌دهنده مدل، هوک، سرویس یا بک‌اند راه‌اندازی. این دستور فراداده مانیفست موردنیاز
OpenClaw را تولید می‌کند تا ابزارها را بدون بارگذاری کد زمان اجرای Plugin
کشف کند.

برای Pluginهای ارائه‌دهنده، کانال، هوک، سرویس یا دارای قابلیت‌های ترکیبی، به‌جای آن با
[ساخت Pluginها](/fa/plugins/building-plugins)، [Pluginهای کانال](/fa/plugins/sdk-channel-plugins)،
یا [Pluginهای ارائه‌دهنده](/fa/plugins/sdk-provider-plugins) شروع کنید.

## الزامات

- Node 22.22.3+، Node 24.15+ یا Node 25.9+.
- خروجی بسته TypeScript ESM.
- `typebox` در `dependencies` (نه فقط `devDependencies` — Plugin تولیدشده
  آن را در زمان اجرا وارد می‌کند).
- `openclaw >=2026.5.17`، نخستین نسخه‌ای که
  `openclaw/plugin-sdk/tool-plugin` را صادر می‌کند.
- ریشه بسته‌ای که `dist/`، `openclaw.plugin.json` و
  `package.json` را ارائه می‌کند.

## شروع سریع

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm install
npm run plugin:build
npm run plugin:validate
npm test
```

`plugins init` موارد زیر را داربست‌بندی می‌کند:

| فایل                   | هدف                                                           |
| ---------------------- | ----------------------------------------------------------------- |
| `src/index.ts`         | ورودی `defineToolPlugin` با یک ابزار `echo`                     |
| `src/index.test.ts`    | آزمون فراداده برای تأیید فهرست ابزار                             |
| `tsconfig.json`        | خروجی TypeScript از نوع NodeNext در `dist/`                             |
| `vitest.config.ts`     | پیکربندی Vitest برای `src/**/*.test.ts`                              |
| `package.json`         | اسکریپت‌ها، وابستگی‌های زمان اجرا، `openclaw.extensions: ["./dist/index.js"]` |
| `openclaw.plugin.json` | فراداده مانیفست تولیدشده برای ابزار اولیه                  |

`npm run plugin:build` ابتدا `npm run build` (tsc) و سپس
`openclaw plugins build --entry ./dist/index.js` را اجرا می‌کند. `npm run plugin:validate`
دوباره می‌سازد و `openclaw plugins validate --entry ./dist/index.js` را اجرا می‌کند.
اعتبارسنجی موفق پیام زیر را چاپ می‌کند:

```text
Plugin stock-quotes معتبر است.
```

گزینه‌های `openclaw plugins init <id>`:

| پرچم                 | پیش‌فرض            | اثر                                 |
| -------------------- | ------------------ | -------------------------------------- |
| `--directory <path>` | `<id>`             | دایرکتوری خروجی                       |
| `--name <name>`      | `<id>` با حروف عنوانی | نام نمایشی                           |
| `--type <type>`      | `tool`             | نوع داربست: `tool` یا `provider`    |
| `--force`            | غیرفعال                | بازنویسی دایرکتوری خروجی موجود |

## نوشتن یک ابزار

`defineToolPlugin` هویت Plugin، یک شِمای پیکربندی اختیاری و فهرستی
ایستا از ابزارها را دریافت می‌کند. نوع‌های پارامتر و پیکربندی از شِماهای
TypeBox استنتاج می‌شوند.

```typescript
import { Type } from "typebox";
import { defineToolPlugin } from "openclaw/plugin-sdk/tool-plugin";

export default defineToolPlugin({
  id: "stock-quotes",
  name: "Stock Quotes",
  description: "Fetch stock quote snapshots.",
  configSchema: Type.Object({
    apiKey: Type.Optional(Type.String({ description: "Quote API key." })),
    baseUrl: Type.Optional(Type.String({ description: "Quote API base URL." })),
  }),
  tools: (tool) => [
    tool({
      name: "stock_quote",
      label: "Stock Quote",
      description: "Fetch a stock quote snapshot.",
      parameters: Type.Object({
        symbol: Type.String({ description: "Ticker symbol, for example OPEN." }),
      }),
      outputSchema: Type.Object(
        {
          symbol: Type.String(),
          configured: Type.Boolean(),
          baseUrl: Type.String(),
        },
        { additionalProperties: false },
      ),
      async execute({ symbol }, config, context) {
        context.signal?.throwIfAborted();
        return {
          symbol: symbol.toUpperCase(),
          configured: Boolean(config.apiKey),
          baseUrl: config.baseUrl ?? "https://api.example.com",
        };
      },
    }),
  ],
});
```

نام ابزارها API پایدار هستند. نام‌هایی یکتا، با حروف کوچک و به‌اندازه کافی
مشخص انتخاب کنید تا با ابزارهای هسته یا Pluginهای دیگر تداخل نداشته باشند.

## ابزارهای اختیاری و کارخانه‌ای

وقتی کاربران باید ابزار را پیش از ارسال به مدل صریحاً در فهرست مجاز قرار دهند،
`optional: true` را تنظیم کنید. `openclaw plugins build` ورودی مانیفست
`toolMetadata.<tool>.optional` متناظر را می‌نویسد تا OpenClaw بتواند بدون بارگذاری کد
زمان اجرای Plugin تشخیص دهد که ابزار اختیاری است.

```typescript
tool({
  name: "workflow_run",
  description: "Run an external workflow.",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  execute: ({ goal }) => ({ queued: true, goal }),
});
```

وقتی ابزاری پیش از ساخته‌شدن به زمینه ابزار زمان اجرا نیاز دارد، از
`factory` استفاده کنید — برای انصراف در یک اجرای خاص، بررسی وضعیت
سندباکس یا اتصال کمک‌تابع‌های زمان اجرا. حتی با وجود ساخته‌شدن ابزار مشخص در
زمان اجرا، فراداده ایستا باقی می‌ماند.

```typescript
tool({
  name: "local_workflow",
  description: "Run a local workflow outside sandboxed sessions.",
  parameters: Type.Object({ goal: Type.String() }),
  optional: true,
  factory({ api, toolContext }) {
    if (toolContext.sandboxed) {
      return null;
    }
    return createLocalWorkflowTool(api);
  },
});
```

کارخانه‌ها همچنان نام ثابت ابزار را از پیش اعلام می‌کنند. وقتی Plugin نام
ابزارها را به‌صورت پویا محاسبه می‌کند یا ابزارها را با هوک‌ها، سرویس‌ها،
ارائه‌دهندگان یا فرمان‌ها ترکیب می‌کند، مستقیماً از `definePluginEntry`
استفاده کنید.

## مقادیر بازگشتی

`defineToolPlugin` مقادیر بازگشتی ساده را در قالب نتیجه ابزار OpenClaw
قرار می‌دهد:

- وقتی مدل باید دقیقاً همان متن را ببیند، یک رشته برگردانید.
- وقتی می‌خواهید مدل JSON قالب‌بندی‌شده را ببیند و OpenClaw مقدار اصلی
  را در `details` نگه دارد، مقداری سازگار با JSON برگردانید.

```typescript
tool({
  name: "echo_text",
  description: "Echo input text.",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => input,
});
```

```typescript
tool({
  name: "echo_json",
  description: "Echo input as structured JSON.",
  parameters: Type.Object({
    input: Type.String(),
  }),
  execute: ({ input }) => ({ input, length: input.length }),
});
```

وقتی به `AgentToolResult` سفارشی نیاز دارید یا می‌خواهید از پیاده‌سازی
`api.registerTool` موجود دوباره استفاده کنید، از ابزار کارخانه‌ای استفاده کنید.

## قراردادهای خروجی

وقتی ابزار داده‌های پایدار سازگار با JSON برمی‌گرداند، `outputSchema` را
اضافه کنید. این مورد مقدار اصلی ذخیره‌شده در `AgentToolResult.details` را توصیف
می‌کند، نه متن قالب‌بندی‌شده در `content`:

```typescript
tool({
  name: "shipment_list",
  description: "List shipments.",
  parameters: Type.Object({
    buyer: Type.Optional(Type.String()),
  }),
  outputSchema: Type.Array(
    Type.Object(
      {
        id: Type.String(),
        buyer: Type.String(),
        paid: Type.Boolean(),
        tons: Type.Number(),
      },
      { additionalProperties: false },
    ),
  ),
  execute: ({ buyer }) => listShipments(buyer),
});
```

[حالت کد](/fa/tools/code-mode) و [جست‌وجوی ابزار](/fa/tools/tool-search) این
شِما را به یک راهنمای خروجی محدود به سبک TypeScript تبدیل می‌کنند. این امکان
به مدل اجازه می‌دهد نتیجه‌ای شناخته‌شده را در یک برنامه فراخوانی و تبدیل کند،
به‌جای آنکه یک نوبت دیگر مدل را صرف مشاهده ساختار آن کند.

OpenClaw شِما را پیش از اجرای فراخوانی کاتالوگ کامپایل می‌کند، سپس مقدار نهایی
`details` را پس از هوک‌های ابزار و پیش از بازگرداندن آن از طریق پل
اعتبارسنجی می‌کند. شِمای نامعتبر نمی‌تواند ابزار را اجرا کند؛ عدم تطابق نتیجه
باعث شکست فراخوانی تکمیل‌شده می‌شود. همه گونه‌های نتیجه‌ای را که استثنا ایجاد
نمی‌کنند، از جمله گونه‌های خطای ساخت‌یافته، درج کنید؛ یا وقتی نتیجه پایدار
نیست، شِما را حذف کنید. رازها یا مقادیر حساس را در توضیحات شِما قرار ندهید،
زیرا فراداده خروجی قابل‌اعتماد ممکن است برای مدل قابل‌مشاهده شود.
وقتی یک راهنمای خروجی فشرده و کامل می‌خواهید، در لایه‌های شیء از
`{ additionalProperties: false }` استفاده کنید؛ شِماهای باز یا کوتاه‌شده همچنان از طریق
`tools.describe(...)` در دسترس‌اند، اما به‌عنوان قراردادهای کامل نمایه سریع
معرفی نمی‌شوند.

ابزارهای کارخانه‌ای `outputSchema` را روی `AnyAgentTool` مشخصی
که برمی‌گردانند اعلام می‌کنند. اعلان ایستای `tool({ factory })` شِمای خروجی
جداگانه‌ای نمی‌پذیرد، زیرا ممکن است با ابزار زمان اجرا ناسازگار شود.

## پیکربندی

`configSchema` اختیاری است. آن را حذف کنید تا OpenClaw یک شِمای سخت‌گیرانه
شیء خالی اعمال کند؛ مانیفست تولیدشده همچنان شامل `configSchema` خواهد بود.

```typescript
export default defineToolPlugin({
  id: "no-config-tools",
  name: "No Config Tools",
  description: "Adds tools that do not need configuration.",
  tools: () => [],
});
```

با یک `configSchema`، نوع آرگومان دوم `execute` از آن
استنتاج می‌شود:

```typescript
const configSchema = Type.Object({
  apiKey: Type.String(),
});

export default defineToolPlugin({
  id: "configured-tools",
  name: "Configured Tools",
  description: "Adds configured tools.",
  configSchema,
  tools: (tool) => [
    tool({
      name: "configured_ping",
      description: "Check whether configuration is available.",
      parameters: Type.Object({}),
      execute: (_params, config) => ({ hasKey: config.apiKey.length > 0 }),
    }),
  ],
});
```

OpenClaw پیکربندی Plugin را از ورودی آن Plugin در پیکربندی Gateway می‌خواند.
رازها را در کد منبع یا نمونه‌های مستندات به‌صورت ثابت ننویسید؛ مطابق مدل
امنیتی Plugin از پیکربندی، متغیرهای محیطی یا SecretRefها استفاده کنید.

## فراداده تولیدشده

OpenClaw باید پیش از واردکردن کد زمان اجرای Plugin، مانیفست آن را بخواند.
`defineToolPlugin` فراداده ایستا را برای این کار ارائه می‌کند و
`openclaw plugins build` آن را در بسته می‌نویسد. پس از تغییر شناسه، نام، توضیحات،
شِمای پیکربندی، فعال‌سازی یا نام ابزارهای Plugin، مولد را دوباره اجرا کنید:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

مانیفست تولیدشده برای یک Plugin تک‌ابزاری:

```json
{
  "id": "stock-quotes",
  "name": "Stock Quotes",
  "description": "Fetch stock quote snapshots.",
  "version": "0.1.0",
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {}
  },
  "activation": {
    "onStartup": true
  },
  "contracts": {
    "tools": ["stock_quote"]
  }
}
```

`contracts.tools` قرارداد مهم کشف است: این قرارداد بدون بارگذاری زمان اجرای
همه Pluginهای نصب‌شده به OpenClaw می‌گوید مالک هر ابزار کدام Plugin است.
مانیفست منسوخ ممکن است باعث شود ابزاری در کشف پیدا نشود یا خطای ثبت به
Plugin اشتباهی نسبت داده شود.

## فراداده بسته

`openclaw plugins build` همچنین `package.json` را با ورودی زمان اجرای
انتخاب‌شده هم‌تراز می‌کند:

```json
{
  "type": "module",
  "files": ["dist", "openclaw.plugin.json", "README.md"],
  "dependencies": {
    "typebox": "^1.1.38"
  },
  "peerDependencies": {
    "openclaw": ">=2026.5.17"
  },
  "openclaw": {
    "extensions": ["./dist/index.js"]
  }
}
```

JavaScript ساخته‌شده (`./dist/index.js`) را منتشر کنید، نه یک ورودی منبع
TypeScript. ورودی‌های منبع فقط برای توسعه محلی در فضای کاری کار می‌کنند.

## اعتبارسنجی در CI

اگر فراداده تولیدشده منسوخ باشد، `plugins build --check` بدون بازنویسی فایل‌ها
شکست می‌خورد:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js --check
openclaw plugins validate --entry ./dist/index.js
npm test
```

فیلدهای سازگاری SDK در OpenClaw دارای حاشیه‌نویسی‌های TypeScript
`@deprecated` هستند که ویرایشگرها آن‌ها را به‌صورت هشدار مهاجرت نمایش
می‌دهند. برای اعمال آن‌ها در CI، یک قاعده آگاه از نوع مانند
[`@typescript-eslint/no-deprecated`](https://typescript-eslint.io/rules/no-deprecated/)
را فعال کنید.
Oxlint از نوع‌ها آگاه نیست، بنابراین نمی‌تواند این حاشیه‌نویسی‌ها را اعمال کند.
ازاین‌رو داربست تولیدشده `plugins init` پیکربندی لینت برای موارد منسوخ
اضافه نمی‌کند.

`plugins validate` بررسی می‌کند که:

- `openclaw.plugin.json` وجود دارد و از بارگذار عادی مانیفست عبور می‌کند.
- ورودی فعلی فرادادهٔ `defineToolPlugin` را صادر می‌کند.
- فیلدهای مانیفست تولیدشده با فرادادهٔ ورودی مطابقت دارند.
- `contracts.tools` با نام‌های ابزار اعلام‌شده مطابقت دارد.
- `package.json`، `openclaw.extensions` را به ورودی زمان اجرای انتخاب‌شده هدایت می‌کند.

## نصب و بررسی محلی

از یک checkout جداگانهٔ OpenClaw یا CLI نصب‌شده، مسیر بسته را نصب کنید:

```bash
openclaw plugins install ./stock-quotes
openclaw plugins inspect stock-quotes --runtime
```

برای یک آزمون دود بسته‌بندی‌شده، ابتدا بسته را بسازید و سپس tarball را نصب کنید:

```bash
npm pack
openclaw plugins install npm-pack:./openclaw-plugin-stock-quotes-0.1.0.tgz
openclaw plugins inspect stock-quotes --runtime --json
```

پس از نصب، Gateway را راه‌اندازی مجدد یا بازبارگذاری کنید و از عامل بخواهید از
ابزار استفاده کند. اگر ابزار قابل مشاهده نیست، پیش از تغییر کد، زمان اجرای Plugin و کاتالوگ
مؤثر ابزار را بررسی کنید (به [عیب‌یابی](#troubleshooting) مراجعه کنید).

## انتشار

پس از آماده‌شدن بسته، آن را از طریق ClawHub منتشر کنید. `clawhub package publish`
یک منبع می‌پذیرد: یک پوشهٔ محلی، یک مخزن GitHub (`owner/repo[@ref]`) یا یک
نشانی URL مربوط به tarball.

```bash
clawhub package publish ./stock-quotes --dry-run
clawhub package publish ./stock-quotes
```

با یک مکان‌یاب صریح ClawHub نصب کنید:

```bash
openclaw plugins install clawhub:your-org/stock-quotes
```

مشخصات سادهٔ بسته‌های npm در دورهٔ گذار راه‌اندازی همچنان از npm نصب می‌شوند، اما
ClawHub بستر ترجیحی کشف و توزیع Pluginهای OpenClaw است.
برای محدودهٔ مالک و بازبینی انتشار، به [انتشار در ClawHub](/fa/clawhub/publishing) مراجعه کنید.

## عیب‌یابی

### `plugin entry not found: ./dist/index.js`

فایل ورودی انتخاب‌شده وجود ندارد. `npm run build` را اجرا کنید، سپس
`openclaw plugins build --entry ./dist/index.js` یا
`openclaw plugins validate --entry ./dist/index.js` را دوباره اجرا کنید.

### `plugin entry does not expose defineToolPlugin metadata`

ورودی، مقداری را که توسط `defineToolPlugin` ایجاد شده باشد صادر نکرد. تأیید کنید که
خروجی پیش‌فرض ماژول، نتیجهٔ `defineToolPlugin(...)` است، یا ورودی
صحیح را با `--entry` ارسال کنید.

### `openclaw.plugin.json generated metadata is stale`

مانیفست دیگر با فرادادهٔ ورودی مطابقت ندارد. اجرا کنید:

```bash
npm run build
openclaw plugins build --entry ./dist/index.js
```

تغییرات هر دو `openclaw.plugin.json` و `package.json` را commit کنید.

### `package.json openclaw.extensions must include ./dist/index.js`

فرادادهٔ بسته به ورودی زمان اجرای دیگری اشاره می‌کند. `openclaw plugins build --entry ./dist/index.js` را اجرا کنید
تا تولیدکننده، فرادادهٔ بسته را با ورودی‌ای که قصد انتشارش را دارید هم‌راستا کند.

### `Cannot find package 'typebox'`

Plugin ساخته‌شده در زمان اجرا `typebox` را import می‌کند. آن را در `dependencies`
نگه دارید، دوباره نصب و build کنید و اعتبارسنجی را مجدداً اجرا کنید.

### ابزار پس از نصب ظاهر نمی‌شود

موارد زیر را به‌ترتیب بررسی کنید:

1. `openclaw plugins inspect <plugin-id> --runtime`
2. `openclaw plugins validate --root <plugin-root> --entry ./dist/index.js`
3. `openclaw.plugin.json` دارای `contracts.tools` با نام‌های مورد انتظار ابزار است.
4. `package.json` دارای `openclaw.extensions: ["./dist/index.js"]` است.
5. Gateway پس از نصب Plugin راه‌اندازی مجدد یا بازبارگذاری شده است.

## همچنین ببینید

- [ساخت Pluginها](/fa/plugins/building-plugins)
- [نقاط ورود Plugin](/fa/plugins/sdk-entrypoints)
- [زیرمسیرهای SDK مربوط به Plugin](/fa/plugins/sdk-subpaths)
- [مانیفست Plugin](/fa/plugins/manifest)
- [CLI مربوط به Pluginها](/fa/cli/plugins)
- [انتشار در ClawHub](/fa/clawhub/publishing)
