---
read_when:
    - می‌خواهید یک URL را واکشی و محتوای خوانا را استخراج کنید
    - باید web_fetch یا گزینهٔ جایگزین آن، Firecrawl، را پیکربندی کنید
    - می‌خواهید محدودیت‌ها و سازوکار کش‌کردن web_fetch را درک کنید
sidebarTitle: Web Fetch
summary: ابزار web_fetch — واکشی HTTP با استخراج محتوای خوانا
title: واکشی وب
x-i18n:
    generated_at: "2026-07-27T14:46:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ddf312245064672dcf489e8714740fa3e034827e16b33be8fb6a87db04f19ef8
    source_path: tools/web-fetch.md
    workflow: 16
---

`web_fetch` یک درخواست ساده HTTP GET انجام می‌دهد و محتوای خوانا را استخراج می‌کند (HTML به
markdown یا متن). این ابزار JavaScript را اجرا **نمی‌کند**. برای سایت‌های متکی به JS یا
صفحه‌های محافظت‌شده با ورود، به‌جای آن از [مرورگر وب](/fa/tools/browser) استفاده کنید.

## شروع سریع

به‌طور پیش‌فرض فعال است و به پیکربندی نیاز ندارد:

```javascript
await web_fetch({ url: "https://example.com/article" });
```

## پارامترهای ابزار

<ParamField path="url" type="string" required>
URL برای واکشی. فقط `http(s)`.
</ParamField>

<ParamField path="extractMode" type="'markdown' | 'text'" default="markdown">
قالب خروجی پس از استخراج محتوای اصلی.
</ParamField>

<ParamField path="maxChars" type="number">
خروجی را به این تعداد نویسه برش می‌دهد. به `tools.web.fetch.maxCharsCap` محدود می‌شود.
</ParamField>

## نتیجه

`web_fetch` یک نتیجه ساخت‌یافته بسته با این فیلدها برمی‌گرداند:

- فراداده درخواست: `url`، `finalUrl`، `status`، `extractMode` و `extractor`
- فراداده اختیاری پاسخ: `contentType`، `title` و `warning` (در صورت نبودن حذف می‌شوند)
- فراداده محتوای پوشش‌داده‌شده: `externalContent`، `truncated`، `length`، `rawLength`،
  `fetchedAt`، `tookMs` و `text`
- `cached: true` اختیاری هنگام اصابت به کش
- `spill: { path, chars, truncated? }` اختیاری هنگامی که محتوای برش‌خورده در
  یک فایل موقت خصوصی نوشته شده باشد؛ `truncated` فقط زمانی وجود دارد که آن فایل
  حاوی بخشی از محتوای منبع باشد

`length` طول `text` پوشش‌داده‌شده است. `rawLength` طول محتوای استخراج‌شده
پیش از پوشش‌دهی محتوای خارجی است.

## نحوه کار

<Steps>
  <Step title="واکشی">
    یک درخواست HTTP GET با User-Agent مشابه Chrome و هدر `Accept-Language`
    ارسال می‌کند. نام‌های میزبان خصوصی/داخلی را مسدود می‌کند و تغییرمسیرها را دوباره بررسی می‌کند.
  </Step>
  <Step title="استخراج">
    Readability (استخراج محتوای اصلی) را روی پاسخ HTML اجرا می‌کند.
  </Step>
  <Step title="مسیر جایگزین (اختیاری)">
    اگر Readability ناموفق باشد و یک ارائه‌دهنده واکشی در دسترس باشد، درخواست را از طریق
    آن ارائه‌دهنده دوباره امتحان می‌کند (برای مثال، حالت دور زدن ربات Firecrawl).
  </Step>
  <Step title="کش">
    نتایج برای 15 دقیقه (قابل پیکربندی) کش می‌شوند تا واکشی‌های تکراری
    همان URL کاهش یابد.
  </Step>
</Steps>

## به‌روزرسانی‌های پیشرفت

`web_fetch` فقط زمانی یک خط عمومی پیشرفت منتشر می‌کند که واکشی پس از
پنج ثانیه همچنان در انتظار باشد:

```text
در حال واکشی محتوای صفحه...
```

اصابت‌های سریع به کش و پاسخ‌های سریع شبکه پیش از فعال‌شدن زمان‌سنج پایان می‌یابند، بنابراین
هرگز خط پیشرفت را نشان نمی‌دهند. لغو فراخوانی، زمان‌سنج را پاک می‌کند. خط
پیشرفت فقط وضعیت رابط کاربری کانال است و هرگز محتوای واکشی‌شده صفحه را در بر نمی‌گیرد.

## پیکربندی

```json5
{
  tools: {
    web: {
      fetch: {
        enabled: true, // پیش‌فرض: true
        provider: "firecrawl", // اختیاری؛ برای تشخیص خودکار حذف کنید
        maxChars: 20000, // نویسه‌های خروجی پیش‌فرض؛ محدودشده با maxCharsCap
        maxCharsCap: 20000, // سقف قطعی پارامتر maxChars
        maxResponseBytes: 750000, // حداکثر اندازه دانلود پیش از برش (32000-10000000)
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        useTrustedEnvProxy: false, // اجازه به پروکسی محیطی HTTP(S) مورد اعتماد برای تفکیک DNS
        readability: true, // استفاده از استخراج Readability
        userAgent: "Mozilla/5.0 ...", // بازنویسی User-Agent
        ssrfPolicy: {
          allowRfc2544BenchmarkRange: true, // فعال‌سازی اختیاری برای پروکسی‌های IP جعلی مورد اعتماد با استفاده از 198.18.0.0/15
          allowIpv6UniqueLocalRange: true, // فعال‌سازی اختیاری برای پروکسی‌های IP جعلی مورد اعتماد با استفاده از fc00::/7
        },
      },
    },
  },
}
```

## مسیر جایگزین Firecrawl

اگر استخراج Readability ناموفق باشد، `web_fetch` می‌تواند برای
دور زدن ربات و استخراج بهتر از [Firecrawl](/fa/tools/firecrawl) به‌عنوان مسیر جایگزین استفاده کند:

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // اختیاری؛ برای تشخیص خودکار از اعتبارنامه‌های موجود حذف کنید
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            // apiKey: "fc-...", // اختیاری؛ برای دسترسی آغازین بدون کلید حذف کنید
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000, // مدت کش (2 روز)
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

`plugins.entries.firecrawl.config.webFetch.apiKey` اختیاری است و از اشیای SecretRef پشتیبانی می‌کند.
پیکربندی قدیمی `tools.web.fetch.firecrawl.*` از طریق `openclaw doctor --fix` به‌طور خودکار به
`plugins.entries.firecrawl.config.webFetch` مهاجرت می‌کند.

<Note>
  اگر یک SecretRef کلید API مربوط به Firecrawl را پیکربندی کنید و بدون مسیر جایگزین
  محیطی `FIRECRAWL_API_KEY` قابل تفکیک نباشد، راه‌اندازی Gateway بی‌درنگ ناموفق می‌شود.
</Note>

<Note>
  بازنویسی‌های `baseUrl` در Firecrawl محدود شده‌اند: ترافیک میزبانی‌شده از
  `https://api.firecrawl.dev` استفاده می‌کند؛ بازنویسی‌های خودمیزبان باید نقاط پایانی خصوصی یا
  داخلی را هدف بگیرند و `http://` فقط برای همان اهداف خصوصی پذیرفته می‌شود.
</Note>

رفتار فعلی زمان اجرا:

- `tools.web.fetch.provider` ارائه‌دهنده مسیر جایگزین واکشی را به‌صراحت انتخاب می‌کند.
- اگر `provider` حذف شود، OpenClaw نخستین ارائه‌دهنده آماده واکشی وب را
  به‌طور خودکار از اعتبارنامه‌های پیکربندی‌شده تشخیص می‌دهد. `web_fetch` خارج از sandbox می‌تواند از
  Pluginهای نصب‌شده‌ای استفاده کند که `contracts.webFetchProviders` را اعلام می‌کنند و
  یک ارائه‌دهنده منطبق را هنگام اجرا ثبت می‌کنند. Plugin رسمی Firecrawl در حال حاضر این
  مسیر جایگزین را فراهم می‌کند.
- فراخوانی‌های `web_fetch` در sandbox، ارائه‌دهندگان همراه و نیز ارائه‌دهندگان نصب‌شده‌ای را مجاز می‌کنند
  که منشأ رسمی npm یا ClawHub آن‌ها تأیید شده باشد. در حال حاضر، این وضعیت
  Plugin رسمی Firecrawl را مجاز می‌کند؛ Pluginهای واکشی خارجی شخص ثالث همچنان مستثنا هستند.
- اگر Readability غیرفعال باشد، `web_fetch` مستقیماً به مسیر جایگزین
  ارائه‌دهنده انتخاب‌شده می‌رود. اگر هیچ ارائه‌دهنده‌ای در دسترس نباشد، به‌صورت بسته ناموفق می‌شود.

## پروکسی محیطی مورد اعتماد

اگر استقرار شما مستلزم عبور `web_fetch` از یک پروکسی خروجی
HTTP(S) مورد اعتماد است، `tools.web.fetch.useTrustedEnvProxy: true` را تنظیم کنید.

در این حالت، OpenClaw همچنان پیش از ارسال درخواست بررسی‌های SSRF مبتنی بر نام میزبان را
اعمال می‌کند، اما به‌جای سنجاق‌کردن محلی DNS، به پروکسی اجازه تفکیک DNS می‌دهد.
این گزینه را فقط زمانی فعال کنید که پروکسی تحت کنترل اپراتور باشد و پس از تفکیک DNS،
سیاست خروجی را اعمال کند.

<Note>
  اگر هیچ متغیر محیطی پروکسی HTTP(S) پیکربندی نشده باشد یا میزبان هدف توسط
  `NO_PROXY` مستثنا شده باشد، `web_fetch` به مسیر سخت‌گیرانه عادی با
  سنجاق‌کردن محلی DNS بازمی‌گردد.
</Note>

## محدودیت‌ها و ایمنی

- `maxChars` به `tools.web.fetch.maxCharsCap` (پیش‌فرض `20000`) محدود می‌شود
- بدنه پاسخ پیش از تجزیه به `maxResponseBytes` (پیش‌فرض `750000`، محدودشده به
  32000-10000000) محدود می‌شود؛ پاسخ‌های بیش‌ازحد بزرگ با یک هشدار برش می‌خورند
- نام‌های میزبان خصوصی/داخلی مسدود می‌شوند
- `tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` و
  `tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange` فعال‌سازی‌های اختیاری محدودی
  برای پشته‌های پروکسی IP جعلی مورد اعتماد هستند؛ مگر اینکه پروکسی شما مالک
  آن محدوده‌های مصنوعی باشد و سیاست مقصد خود را اعمال کند، آن‌ها را تنظیم نکنید
- تغییرمسیرها بررسی می‌شوند و به‌وسیله `maxRedirects` (پیش‌فرض `3`) محدود می‌شوند
- `useTrustedEnvProxy` یک فعال‌سازی اختیاری صریح است و فقط باید برای
  پروکسی‌های تحت کنترل اپراتور فعال شود که پس از تفکیک DNS همچنان سیاست خروجی را
  اعمال می‌کنند
- `web_fetch` بر مبنای بهترین تلاش عمل می‌کند -- برخی سایت‌ها به [مرورگر وب](/fa/tools/browser) نیاز دارند

## پروفایل‌های ابزار

اگر از پروفایل‌ها یا فهرست‌های مجاز ابزار استفاده می‌کنید، `web_fetch` یا `group:web` را اضافه کنید:

```json5
{
  tools: {
    allow: ["web_fetch"],
    // یا: allow: ["group:web"]  (شامل web_fetch، web_search و x_search)
  },
}
```

## مرتبط

- [جست‌وجوی وب](/fa/tools/web) -- جست‌وجوی وب با چندین ارائه‌دهنده
- [مرورگر وب](/fa/tools/browser) -- خودکارسازی کامل مرورگر برای سایت‌های متکی به JS
- [Firecrawl](/fa/tools/firecrawl) -- ابزارهای جست‌وجو و استخراج Firecrawl
