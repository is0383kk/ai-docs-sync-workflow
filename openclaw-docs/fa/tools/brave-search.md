---
read_when:
    - می‌خواهید از Brave Search برای web_search استفاده کنید
    - به یک BRAVE_API_KEY یا جزئیات طرح نیاز دارید
summary: راه‌اندازی API جست‌وجوی Brave برای `web_search`
title: جست‌وجوی Brave
x-i18n:
    generated_at: "2026-07-27T16:21:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 52168db93abb564eda5868584261e0530ce3cff57c3463a2fc1eded351df30f2
    source_path: tools/brave-search.md
    workflow: 16
---

OpenClaw از Brave Search API به‌عنوان ارائه‌دهندهٔ `web_search` پشتیبانی می‌کند.

## دریافت کلید API

1. یک حساب Brave Search API در [https://brave.com/search/api/](https://brave.com/search/api/) ایجاد کنید.
2. در داشبورد، طرح **Search** را انتخاب و یک کلید API ایجاد کنید.
3. کلید را در پیکربندی ذخیره کنید یا `BRAVE_API_KEY` را در محیط Gateway تنظیم کنید.

## نمونهٔ پیکربندی

```json5
{
  plugins: {
    entries: {
      brave: {
        config: {
          webSearch: {
            apiKey: "BRAVE_API_KEY_HERE",
            mode: "web", // یا "llm-context"
            baseUrl: "https://api.search.brave.com", // بازنویسی اختیاری پراکسی/نشانی پایه
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "brave",
        maxResults: 5,
        timeoutSeconds: 30,
      },
    },
  },
}
```

تنظیمات جست‌وجوی ویژهٔ ارائه‌دهندهٔ Brave در `plugins.entries.brave.config.webSearch.*` قرار دارند؛ این مسیر متعارف پیکربندی است.

`webSearch.mode` انتقال Brave را کنترل می‌کند:

- `web` (پیش‌فرض): جست‌وجوی عادی وب Brave با عنوان‌ها، نشانی‌های URL و گزیده‌ها
- `llm-context`: رابط Brave LLM Context API با قطعه‌های متنی ازپیش‌استخراج‌شده و منابع برای زمینه‌سازی

`webSearch.baseUrl` می‌تواند درخواست‌های Brave را به یک پراکسی
یا Gateway سازگار با Brave و مورداعتماد هدایت کند. OpenClaw، `/res/v1/web/search` یا `/res/v1/llm/context` را به
نشانی پایهٔ پیکربندی‌شده می‌افزاید و نشانی پایه را در کلید کش نگه می‌دارد. نقاط پایانی
عمومی باید از `https://` استفاده کنند؛ `http://` فقط برای میزبان‌های پراکسی loopback مورداعتماد
یا شبکهٔ خصوصی پذیرفته می‌شود.

## پارامترهای ابزار

<ParamField path="query" type="string" required>
عبارت جست‌وجو.
</ParamField>

<ParamField path="count" type="number" default="5">
تعداد نتایجی که باید بازگردانده شوند (1–10).
</ParamField>

<ParamField path="country" type="string">
کد دوحرفی ISO کشور (برای مثال، `US`، `DE`).
</ParamField>

<ParamField path="language" type="string">
کد زبان ISO 639-1 برای نتایج جست‌وجو (برای مثال، `en`، `de`، `fr`).
</ParamField>

<ParamField path="search_lang" type="string">
کد زبان جست‌وجوی Brave (برای مثال، `en`، `en-gb`، `zh-hans`).
</ParamField>

<ParamField path="ui_lang" type="string">
کد زبان ISO برای عناصر رابط کاربری.
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
فیلتر زمانی — `day` برابر با 24 ساعت است.
</ParamField>

<ParamField path="date_after" type="string">
فقط نتایجی که پس از این تاریخ منتشر شده‌اند (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
فقط نتایجی که پیش از این تاریخ منتشر شده‌اند (`YYYY-MM-DD`).
</ParamField>

**نمونه‌ها:**

```javascript
// جست‌وجوی ویژهٔ کشور و زبان
await web_search({
  query: "renewable energy",
  country: "DE",
  language: "de",
});

// نتایج اخیر (هفتهٔ گذشته)
await web_search({
  query: "AI news",
  freshness: "week",
});

// جست‌وجو در بازهٔ تاریخی
await web_search({
  query: "AI developments",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});
```

## نکات

- OpenClaw از طرح **Search** متعلق به Brave استفاده می‌کند. اگر اشتراک قدیمی دارید (برای مثال، طرح اصلی Free با 2,000 درخواست در ماه)، همچنان معتبر است، اما قابلیت‌های جدیدتر مانند LLM Context یا محدودیت نرخ بالاتر را شامل نمی‌شود.
- هر طرح Brave شامل **اعتبار رایگان ماهانهٔ \$5** (با تمدید ماهانه) است. طرح Search برای هر 1,000 درخواست، \$5 هزینه دارد؛ بنابراین این اعتبار 1,000 درخواست در ماه را پوشش می‌دهد. برای جلوگیری از هزینه‌های غیرمنتظره، محدودیت مصرف خود را در داشبورد Brave تنظیم کنید. برای مشاهدهٔ طرح‌های فعلی، به [پرتال Brave API](https://brave.com/search/api/) مراجعه کنید.
- طرح Search شامل نقطهٔ پایانی LLM Context و حقوق استنتاج هوش مصنوعی است. ذخیرهٔ نتایج برای آموزش یا تنظیم مدل‌ها به طرحی با حقوق صریح ذخیره‌سازی نیاز دارد. [شرایط استفاده](https://api-dashboard.search.brave.com/terms-of-service) Brave را ببینید.
- حالت `llm-context` به‌جای قالب عادی گزیدهٔ جست‌وجوی وب، ورودی‌های منبع زمینه‌سازی‌شده را بازمی‌گرداند.
- حالت `llm-context` از `freshness` و بازه‌های محدودشدهٔ `date_after` + `date_before` پشتیبانی می‌کند. از `ui_lang` پشتیبانی نمی‌کند؛ `date_before` بدون `date_after` رد می‌شود، زیرا Brave ملزم می‌کند بازه‌های تازگی سفارشی شامل هر دو تاریخ آغاز و پایان باشند.
- `ui_lang` باید شامل یک زیرتگ منطقه مانند `en-US` باشد.
- نتایج به‌طور پیش‌فرض برای 15 دقیقه کش می‌شوند (از طریق `cacheTtlMinutes` قابل‌تنظیم است).
- مقادیر سفارشی `webSearch.baseUrl` در شناسهٔ کش Brave گنجانده می‌شوند تا
  پاسخ‌های ویژهٔ پراکسی با یکدیگر تداخل نکنند.
- برای ثبت نشانی‌های URL و پارامترهای درخواست Brave، وضعیت و زمان‌بندی پاسخ، و رویدادهای اصابت/عدم‌اصابت/نوشتن کش جست‌وجو هنگام عیب‌یابی، پرچم تشخیصی `brave.http` را فعال کنید. این پرچم هرگز کلید API یا بدنهٔ پاسخ‌ها را ثبت نمی‌کند، اما عبارت‌های جست‌وجو ممکن است حساس باشند.

## مطالب مرتبط

- [نمای کلی جست‌وجوی وب](/fa/tools/web) -- همهٔ ارائه‌دهندگان و تشخیص خودکار
- [جست‌وجوی Perplexity](/fa/tools/perplexity-search) -- نتایج ساختاریافته با پالایش دامنه
- [جست‌وجوی Exa](/fa/tools/exa-search) -- جست‌وجوی عصبی با استخراج محتوا
