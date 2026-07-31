---
read_when:
    - می‌خواهید از Perplexity Search برای جست‌وجوی وب استفاده کنید
    - باید PERPLEXITY_API_KEY یا OPENROUTER_API_KEY را تنظیم کنید
summary: سازگاری API جست‌وجوی Perplexity و Sonar/OpenRouter برای web_search
title: جست‌وجوی Perplexity
x-i18n:
    generated_at: "2026-07-27T14:45:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a7ca97355110e70a05f1d57acab475dda8dec89393804df40c6e9be5e30780e8
    source_path: tools/perplexity-search.md
    workflow: 16
---

OpenClaw از API جست‌وجوی Perplexity به‌عنوان ارائه‌دهنده `web_search` پشتیبانی می‌کند. این API نتایج ساختاریافته را با فیلدهای `title`، `url` و `snippet` برمی‌گرداند.

برای سازگاری، OpenClaw از پیکربندی‌های قدیمی Perplexity Sonar/OpenRouter نیز پشتیبانی می‌کند. اگر از `OPENROUTER_API_KEY`، کلید `sk-or-...` در `plugins.entries.perplexity.config.webSearch.apiKey` استفاده کنید یا `plugins.entries.perplexity.config.webSearch.baseUrl` / `model` را تنظیم کنید، ارائه‌دهنده به مسیر تکمیل‌های چت تغییر می‌کند و به‌جای نتایج ساختاریافته API جست‌وجو، پاسخ‌های ترکیب‌شده توسط هوش مصنوعی را همراه با ارجاعات برمی‌گرداند.

## نصب Plugin

Plugin رسمی را نصب کنید، سپس Gateway را راه‌اندازی مجدد کنید:

```bash
openclaw plugins install @openclaw/perplexity-plugin
openclaw gateway restart
```

## دریافت کلید API پرپلکسیتی

1. یک حساب Perplexity در [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api) ایجاد کنید.
2. یک کلید API در داشبورد ایجاد کنید.
3. کلید را در پیکربندی ذخیره کنید یا `PERPLEXITY_API_KEY` را در محیط Gateway تنظیم کنید.

## سازگاری با OpenRouter

اگر از قبل برای Perplexity Sonar از OpenRouter استفاده می‌کردید، `provider: "perplexity"` را نگه دارید و `OPENROUTER_API_KEY` را در محیط Gateway تنظیم کنید، یا کلید `sk-or-...` را در `plugins.entries.perplexity.config.webSearch.apiKey` ذخیره کنید.

کنترل‌های اختیاری سازگاری:

- `plugins.entries.perplexity.config.webSearch.baseUrl`
- `plugins.entries.perplexity.config.webSearch.model`

## نمونه‌های پیکربندی

### API جست‌وجوی بومی Perplexity

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "pplx-...",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

### سازگاری با OpenRouter / Sonar

```json5
{
  plugins: {
    entries: {
      perplexity: {
        config: {
          webSearch: {
            apiKey: "<openrouter-api-key>",
            baseUrl: "https://openrouter.ai/api/v1",
            model: "perplexity/sonar-pro",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "perplexity",
      },
    },
  },
}
```

## محل تنظیم کلید

**از طریق پیکربندی:** دستور `openclaw configure --section web` را اجرا کنید. این دستور کلید را در `~/.openclaw/openclaw.json` زیر `plugins.entries.perplexity.config.webSearch.apiKey` ذخیره می‌کند. این فیلد اشیای SecretRef را نیز می‌پذیرد.

**از طریق محیط:** `PERPLEXITY_API_KEY` یا `OPENROUTER_API_KEY` را در محیط فرایند Gateway تنظیم کنید. برای نصب Gateway، آن را در `~/.openclaw/.env` (یا محیط سرویس خود) قرار دهید. به [متغیرهای محیطی](/fa/help/faq#env-vars-and-env-loading) مراجعه کنید.

اگر `provider: "perplexity"` پیکربندی شده باشد و SecretRef کلید Perplexity بدون جایگزین محیطی حل‌نشده باقی بماند، راه‌اندازی یا بارگذاری مجدد فوراً با شکست مواجه می‌شود.

## پارامترهای ابزار

این پارامترها برای مسیر API جست‌وجوی بومی Perplexity کاربرد دارند.

<ParamField path="query" type="string" required>
پرس‌وجوی جست‌وجو.
</ParamField>

<ParamField path="count" type="number" default="5">
تعداد نتایجی که باید برگردانده شوند (1-10).
</ParamField>

<ParamField path="country" type="string">
کد کشور ISO دوحرفی (برای مثال `US`، `DE`).
</ParamField>

<ParamField path="language" type="string">
کد زبان ISO 639-1 (برای مثال `en`، `de`، `fr`).
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
فیلتر زمانی ـ `day` برابر با 24 ساعت است.
</ParamField>

<ParamField path="date_after" type="string">
فقط نتایجی که پس از این تاریخ منتشر شده‌اند (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
فقط نتایجی که پیش از این تاریخ منتشر شده‌اند (`YYYY-MM-DD`).
</ParamField>

<ParamField path="domain_filter" type="string[]">
آرایه فهرست مجاز/فهرست مسدود دامنه‌ها (حداکثر 20).
</ParamField>

<ParamField path="max_tokens" type="number" default="25000">
بودجه کل محتوا (حداکثر 1000000).
</ParamField>

<ParamField path="max_tokens_per_page" type="number" default="2048">
محدودیت توکن برای هر صفحه.
</ParamField>

برای مسیر سازگاری قدیمی Sonar/OpenRouter:

- `query`، `count` و `freshness` پذیرفته می‌شوند.
- `count` در آن مسیر فقط برای سازگاری است؛ پاسخ همچنان یک پاسخ ترکیب‌شده همراه با ارجاعات است، نه فهرستی با N نتیجه.
- فیلترهای مختص API جست‌وجو (`country`، `language`، `date_after`، `date_before`، `domain_filter`، `max_tokens`، `max_tokens_per_page`) خطاهای صریح برمی‌گردانند.

**نمونه‌ها:**

```javascript
// جست‌وجوی ویژه کشور و زبان
await web_search({
  query: "انرژی تجدیدپذیر",
  country: "DE",
  language: "de",
});

// نتایج اخیر (هفته گذشته)
await web_search({
  query: "اخبار هوش مصنوعی",
  freshness: "week",
});

// جست‌وجو در بازه زمانی
await web_search({
  query: "تحولات هوش مصنوعی",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});

// فیلتر دامنه (فهرست مجاز)
await web_search({
  query: "پژوهش اقلیمی",
  domain_filter: ["nature.com", "science.org", ".edu"],
});

// فیلتر دامنه (فهرست مسدود ـ با - پیشوندگذاری کنید)
await web_search({
  query: "بررسی محصولات",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});

// استخراج محتوای بیشتر
await web_search({
  query: "پژوهش تفصیلی هوش مصنوعی",
  max_tokens: 50000,
  max_tokens_per_page: 4096,
});
```

### قواعد فیلتر دامنه

- حداکثر 20 دامنه در هر فیلتر.
- نمی‌توان ورودی‌های فهرست مجاز و فهرست مسدود را در یک درخواست ترکیب کرد.
- برای ورودی‌های فهرست مسدود از پیشوند `-` استفاده کنید (برای مثال `["-reddit.com"]`).

## نکات

- API جست‌وجوی Perplexity نتایج ساختاریافته جست‌وجوی وب را برمی‌گرداند (`title`، `url`، `snippet`).
- OpenRouter یا `plugins.entries.perplexity.config.webSearch.baseUrl` / `model` صریح، برای سازگاری Perplexity را دوباره به تکمیل‌های چت Sonar تغییر می‌دهد.
- سازگاری Sonar/OpenRouter یک پاسخ ترکیب‌شده همراه با ارجاعات برمی‌گرداند، نه ردیف‌های نتایج ساختاریافته.
- نتایج به‌طور پیش‌فرض برای 15 دقیقه در حافظه نهان نگه‌داری می‌شوند (از طریق `cacheTtlMinutes` قابل پیکربندی است).

## مرتبط

<CardGroup cols={2}>
  <Card title="نمای کلی جست‌وجوی وب" href="/fa/tools/web" icon="globe">
    همه ارائه‌دهندگان و قواعد تشخیص خودکار.
  </Card>
  <Card title="جست‌وجوی Brave" href="/fa/tools/brave-search" icon="shield">
    نتایج ساختاریافته با فیلترهای کشور و زبان.
  </Card>
  <Card title="جست‌وجوی Exa" href="/fa/tools/exa-search" icon="magnifying-glass">
    جست‌وجوی عصبی همراه با استخراج محتوا.
  </Card>
  <Card title="مستندات API جست‌وجوی Perplexity" href="https://docs.perplexity.ai/docs/search/quickstart" icon="arrow-up-right-from-square">
    راهنمای شروع سریع و مرجع رسمی API جست‌وجوی Perplexity.
  </Card>
</CardGroup>
