---
read_when:
    - استخراج وب با پشتیبانی Firecrawl می‌خواهید
    - شما Firecrawl Search بدون کلید (رایگان) یا web_fetch بدون کلید می‌خواهید
    - برای جست‌وجو یا محدودیت‌های بالاتر، به یک کلید API از Firecrawl نیاز دارید
    - شما Firecrawl را به‌عنوان ارائه‌دهندهٔ web_search می‌خواهید
    - برای web_fetch استخراج مقاوم در برابر ربات‌ها می‌خواهید
summary: جست‌وجو و خزش Firecrawl و جایگزین `web_fetch`
title: Firecrawl
x-i18n:
    generated_at: "2026-07-27T15:53:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 98b8af0839b1759e3be9393879a6d9a92fa0c505bf475bafd73c3f32d20fa106
    source_path: tools/firecrawl.md
    workflow: 16
---

OpenClaw می‌تواند از **Firecrawl** به سه روش استفاده کند:

- به‌عنوان ارائه‌دهندهٔ `web_search`
- به‌عنوان ابزارهای صریح Plugin:‏ `firecrawl_search` و `firecrawl_scrape`
- به‌عنوان استخراج‌کنندهٔ جایگزین برای `web_fetch`

این یک سرویس میزبانی‌شده برای استخراج/جست‌وجو است که از دور زدن ربات‌ها و ذخیره‌سازی در حافظهٔ نهان پشتیبانی می‌کند و برای سایت‌های متکی به JS یا صفحه‌هایی که واکشی سادهٔ HTTP را مسدود می‌کنند، مفید است.

## نصب Plugin

Plugin رسمی را نصب کنید، سپس Gateway را راه‌اندازی مجدد کنید:

```bash
openclaw plugins install @openclaw/firecrawl-plugin
openclaw gateway restart
```

## دسترسی بدون کلید و کلیدهای API

Firecrawl دو ارائه‌دهندهٔ `web_search` ثبت می‌کند:

- **جست‌وجوی Firecrawl** (`firecrawl`) — از API میزبانی‌شدهٔ `/v2/search` با کلید شما استفاده می‌کند؛
  در صورت وجود کلید، به‌طور خودکار شناسایی می‌شود.
- **جست‌وجوی Firecrawl (رایگان)** (`firecrawl-free`) — از سطح آغازین میزبانی‌شده و بدون کلید استفاده می‌کند
  و به کلید API نیاز ندارد. این گزینه **فقط با انتخاب صریح فعال می‌شود** و هرگز به‌طور خودکار انتخاب نمی‌شود، زیرا
  انتخاب آن عبارت‌های جست‌وجوی شما را به سطح رایگان Firecrawl ارسال می‌کند.

گزینهٔ جایگزین `web_fetch` در Firecrawl که به‌صراحت انتخاب شده نیز بدون کلید است. ابزارهای صریح
`firecrawl_search` و `firecrawl_scrape` به کلید API نیاز دارند. برای محدودیت‌های بالاتر،
`FIRECRAWL_API_KEY` را به محیط Gateway اضافه یا پیکربندی کنید.

## پیکربندی جست‌وجوی Firecrawl

```json5
{
  tools: {
    web: {
      search: {
        provider: "firecrawl",
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webSearch: {
            apiKey: "FIRECRAWL_API_KEY_HERE",
            baseUrl: "https://api.firecrawl.dev",
          },
        },
      },
    },
  },
}
```

نکته‌ها:

- انتخاب Firecrawl در فرایند راه‌اندازی اولیه یا `openclaw configure --section web`، Plugin نصب‌شدهٔ Firecrawl را به‌طور خودکار فعال می‌کند.
- برای اجرای بدون کلید و بدون نیاز به کلید API، در فرایند راه‌اندازی اولیه **جست‌وجوی Firecrawl (رایگان)** را انتخاب کنید (یا `provider: "firecrawl-free"` را تنظیم کنید). ارائه‌دهندهٔ کلیددار **جست‌وجوی Firecrawl** مقدار `plugins.entries.firecrawl.config.webSearch.apiKey` یا `FIRECRAWL_API_KEY` را ارسال می‌کند.
- `web_search` با Firecrawl از `query` و `count` پشتیبانی می‌کند.
- برای کنترل‌های ویژهٔ Firecrawl مانند `sources`، ‏`categories` یا استخراج نتایج، از `firecrawl_search` استفاده کنید.
- `baseUrl` به‌طور پیش‌فرض از Firecrawl میزبانی‌شده در `https://api.firecrawl.dev` استفاده می‌کند. جایگزین‌های خودمیزبانی‌شده فقط برای نقاط پایانی خصوصی/داخلی مجازند؛ HTTP فقط برای همین مقصدهای خصوصی پذیرفته می‌شود.
- `FIRECRAWL_BASE_URL` متغیر محیطی جایگزین مشترک برای URLهای پایهٔ جست‌وجو و استخراج Firecrawl است.
- مهلت زمانی پیش‌فرض درخواست‌های جست‌وجوی Firecrawl برابر با 30 ثانیه است؛ پارامتر `timeoutSeconds` در `firecrawl_search` آن را برای هر فراخوانی بازنویسی می‌کند.

## پیکربندی گزینهٔ جایگزین Firecrawl برای web_fetch

```json5
{
  tools: {
    web: {
      fetch: {
        provider: "firecrawl", // انتخاب صریح، گزینهٔ جایگزین بدون کلید را فعال می‌کند
      },
    },
  },
  plugins: {
    entries: {
      firecrawl: {
        enabled: true,
        config: {
          webFetch: {
            baseUrl: "https://api.firecrawl.dev",
            onlyMainContent: true,
            maxAgeMs: 172800000,
            timeoutSeconds: 60,
          },
        },
      },
    },
  },
}
```

نکته‌ها:

- گزینهٔ جایگزین `web_fetch` در Firecrawl که به‌صراحت انتخاب شده، بدون کلید API کار می‌کند. در صورت پیکربندی، OpenClaw برای محدودیت‌های بالاتر `plugins.entries.firecrawl.config.webFetch.apiKey` یا `FIRECRAWL_API_KEY` را ارسال می‌کند.
- انتخاب Firecrawl در فرایند راه‌اندازی اولیه یا `openclaw configure --section web`، Plugin را فعال می‌کند و Firecrawl را برای `web_fetch` انتخاب می‌کند، مگر آنکه ارائه‌دهندهٔ واکشی دیگری از قبل پیکربندی شده باشد.
- `firecrawl_scrape` به کلید API نیاز دارد.
- `maxAgeMs` حداکثر عمر نتایج ذخیره‌شده در حافظهٔ نهان را کنترل می‌کند (میلی‌ثانیه). مقدار پیش‌فرض 172,800,000 میلی‌ثانیه (2 روز) است.
- مقدار پیش‌فرض `onlyMainContent` برابر `true` است؛ مقدار پیش‌فرض `timeoutSeconds` برابر 60 است.
- پیکربندی قدیمی `tools.web.fetch.firecrawl.*` و `tools.web.search.firecrawl.*` به‌طور خودکار توسط `openclaw doctor --fix` مهاجرت داده می‌شود.
- بازنویسی‌های URL پایه/استخراج Firecrawl از همان قاعدهٔ میزبانی‌شده/خصوصی جست‌وجو پیروی می‌کنند: ترافیک عمومی میزبانی‌شده از `https://api.firecrawl.dev` استفاده می‌کند؛ بازنویسی‌های خودمیزبانی‌شده باید به نقاط پایانی خصوصی/داخلی منتهی شوند.
- `firecrawl_scrape` پیش از ارسال URLها به Firecrawl، URLهای مقصد آشکارا خصوصی، loopback، فراداده و غیر HTTP(S) را رد می‌کند؛ این رفتار با قرارداد ایمنی مقصد `web_fetch` برای فراخوانی‌های صریح استخراج Firecrawl مطابقت دارد.

`firecrawl_scrape` از همان تنظیمات `plugins.entries.firecrawl.config.webFetch.*` و متغیرهای محیطی، از جمله کلید API الزامی آن، استفاده می‌کند.

### Firecrawl خودمیزبانی‌شده

هنگامی که Firecrawl را خودتان اجرا می‌کنید، `plugins.entries.firecrawl.config.webSearch.baseUrl`، ‏`plugins.entries.firecrawl.config.webFetch.baseUrl` یا `FIRECRAWL_BASE_URL` را تنظیم کنید. OpenClaw مقدار `http://` را فقط برای مقصدهای loopback، شبکهٔ خصوصی، `.local`، ‏`.internal` یا `.localhost` می‌پذیرد. میزبان‌های عمومی سفارشی رد می‌شوند تا کلیدهای API مربوط به Firecrawl به‌طور تصادفی به نقاط پایانی دلخواه ارسال نشوند.

## ابزارهای Plugin مربوط به Firecrawl

### `firecrawl_search`

وقتی به‌جای `web_search` عمومی، کنترل‌های جست‌وجوی ویژهٔ Firecrawl را می‌خواهید، از این ابزار استفاده کنید. به کلید API نیاز دارد.

پارامترها:

- `query`
- `count` (1-100)
- `sources`
- `categories`
- `includeDomains` / `excludeDomains` (فقط نام میزبان؛ ناسازگار با یکدیگر)
- `tbs` (فیلتر زمانی، برای مثال `qdr:d`، ‏`qdr:w`، ‏`sbd:1`)
- `location` و `country` (هدف‌گیری جغرافیایی)
- `scrapeResults`
- `timeoutSeconds`

### `firecrawl_scrape`

برای صفحه‌های متکی به JS یا محافظت‌شده در برابر ربات که `web_fetch` ساده عملکرد ضعیفی دارد، از این ابزار استفاده کنید.

پارامترها:

- `url`
- `extractMode`
- `maxChars`
- `onlyMainContent`
- `maxAgeMs`
- `proxy`
- `storeInCache`
- `timeoutSeconds`

## حالت پنهان‌کارانه / دور زدن ربات

`firecrawl_scrape` و گزینهٔ جایگزین Firecrawl برای `web_fetch`، به‌طور پیش‌فرض از `proxy: "auto"` به‌همراه `storeInCache: true` استفاده می‌کنند، مگر آنکه فراخواننده آن پارامترها را بازنویسی کند. `firecrawl_search` و ارائه‌دهندهٔ Firecrawl برای `web_search` هیچ کنترلی برای `proxy`/`storeInCache` ندارند؛ حالت پروکسی پنهان‌کارانه فقط برای درخواست‌های استخراج/واکشی اعمال می‌شود.

حالت `proxy` در Firecrawl دور زدن ربات را کنترل می‌کند (`basic`، ‏`stealth` یا `auto`). ‏`auto` در صورت ناموفق بودن تلاش پایه، با پروکسی‌های پنهان‌کارانه دوباره تلاش می‌کند که ممکن است نسبت به استخراج صرفاً پایه، اعتبار بیشتری مصرف کند.

## نحوهٔ استفادهٔ `web_fetch` از Firecrawl

ترتیب استخراج `web_fetch`:

1. Readability (محلی)
2. ارائه‌دهندهٔ واکشی پیکربندی‌شده، مانند Firecrawl (هنگامی که انتخاب شده یا به‌طور خودکار از اعتبارنامه‌های پیکربندی‌شده شناسایی شده باشد)
3. پاک‌سازی پایهٔ HTML (آخرین گزینهٔ جایگزین)

کنترل انتخاب، `tools.web.fetch.provider` است. اگر آن را حذف کنید، OpenClaw نخستین ارائه‌دهندهٔ آمادهٔ واکشی وب را از میان اعتبارنامه‌های موجود به‌طور خودکار شناسایی می‌کند. Plugin رسمی Firecrawl این گزینهٔ جایگزین را فراهم می‌کند.

## مرتبط

- [نمای کلی جست‌وجوی وب](/fa/tools/web) -- همهٔ ارائه‌دهندگان و شناسایی خودکار
- [واکشی وب](/fa/tools/web-fetch) -- ابزار web_fetch با گزینهٔ جایگزین Firecrawl
- [Tavily](/fa/tools/tavily) -- ابزارهای جست‌وجو + استخراج
