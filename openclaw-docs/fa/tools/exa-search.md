---
read_when:
    - می‌خواهید از Exa برای web_search استفاده کنید
    - به یک EXA_API_KEY نیاز دارید
    - جست‌وجوی عصبی یا استخراج محتوا می‌خواهید
summary: جست‌وجوی Exa AI — جست‌وجوی عصبی و کلیدواژه‌ای با استخراج محتوا
title: جست‌وجوی Exa
x-i18n:
    generated_at: "2026-07-27T17:11:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3ddfd6fb471f92e705facf5a2d02361c1a343b9032fa8e0a7b135af634df65b7
    source_path: tools/exa-search.md
    workflow: 16
---

[Exa AI](https://exa.ai/) یک ارائه‌دهندهٔ `web_search` با حالت‌های جست‌وجوی عصبی، کلیدواژه‌ای و
ترکیبی، به‌همراه استخراج داخلی محتوا (نکات برجسته، متن و
خلاصه‌ها) است.

## نصب Plugin

```bash
openclaw plugins install @openclaw/exa-plugin
openclaw gateway restart
```

## دریافت کلید API

<Steps>
  <Step title="ایجاد حساب">
    در [exa.ai](https://exa.ai/) ثبت‌نام کنید و از داشبورد خود یک کلید API
    ایجاد کنید.
  </Step>
  <Step title="ذخیره کلید">
    `EXA_API_KEY` را در محیط Gateway تنظیم کنید، یا از طریق دستور زیر پیکربندی کنید:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

## پیکربندی

```json5
{
  plugins: {
    entries: {
      exa: {
        config: {
          webSearch: {
            apiKey: "exa-...", // در صورت تنظیم EXA_API_KEY اختیاری است
            baseUrl: "https://api.exa.ai", // اختیاری؛ OpenClaw مقدار /search را می‌افزاید
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "exa",
      },
    },
  },
}
```

**جایگزین محیطی:** `EXA_API_KEY` را در محیط Gateway تنظیم کنید. برای
نصب Gateway، آن را در `~/.openclaw/.env` قرار دهید. به
[متغیرهای محیطی](/fa/help/faq#env-vars-and-env-loading) مراجعه کنید.

## بازنویسی URL پایه

برای هدایت درخواست‌های جست‌وجوی Exa
از طریق یک پراکسی سازگار یا نقطهٔ پایانی جایگزین، `plugins.entries.exa.config.webSearch.baseUrl` را تنظیم کنید. OpenClaw
میزبان‌های بدون طرح را با افزودن `https://` در ابتدا عادی‌سازی می‌کند و `/search` را می‌افزاید، مگر اینکه
مسیر از قبل به آن ختم شود. نقطهٔ پایانی حل‌شده بخشی از کلید کش جست‌وجو
است؛ بنابراین نتایج نقاط پایانی مختلف هرگز با یکدیگر به‌اشتراک گذاشته نمی‌شوند.

## پارامترهای ابزار

<ParamField path="query" type="string" required>
پرس‌وجوی جست‌وجو.
</ParamField>

<ParamField path="count" type="number" default="5">
تعداد نتایج بازگشتی (1-100، مشروط به محدودیت‌های نوع جست‌وجوی Exa).
</ParamField>

<ParamField path="type" type="'auto' | 'neural' | 'fast' | 'deep' | 'deep-reasoning' | 'instant'">
حالت جست‌وجو.
</ParamField>

<ParamField path="freshness" type="'day' | 'week' | 'month' | 'year'">
فیلتر زمانی. نمی‌توان آن را با `date_after`/`date_before` ترکیب کرد.
</ParamField>

<ParamField path="date_after" type="string">
نتایج پس از این تاریخ (`YYYY-MM-DD`).
</ParamField>

<ParamField path="date_before" type="string">
نتایج پیش از این تاریخ (`YYYY-MM-DD`).
</ParamField>

<ParamField path="contents" type="object">
گزینه‌های استخراج محتوا (در ادامه ببینید).
</ParamField>

### استخراج محتوا

برای کنترل محتوای استخراج‌شده در نتایج، یک شیء `contents` ارسال کنید:

```javascript
await web_search({
  query: "توضیح معماری ترنسفورمر",
  type: "neural",
  contents: {
    text: true, // متن کامل صفحه
    highlights: { numSentences: 3 }, // جمله‌های کلیدی
    summary: true, // خلاصه تولیدشده با هوش مصنوعی
  },
});
```

| گزینه محتوا | نوع                                                                  | توضیحات            |
| --------------- | --------------------------------------------------------------------- | ---------------------- |
| `text`          | `boolean \| { maxCharacters }`                                        | استخراج متن کامل صفحه |
| `highlights`    | `boolean \| { maxCharacters, query, numSentences, highlightsPerUrl }` | استخراج جمله‌های کلیدی  |
| `summary`       | `boolean \| { query }`                                                | خلاصه تولیدشده با هوش مصنوعی   |

اگر `contents` حذف شود، مقدار پیش‌فرض Exa برابر `{ highlights: true }` است تا نتایج
شامل گزیده‌هایی از جمله‌های کلیدی باشند. توضیحات نتایج ابتدا از نکات برجسته،
سپس خلاصه و بعد متن کامل تأمین می‌شوند؛ هرکدام که زودتر در دسترس باشد. نتایج
همچنین در صورت موجود بودن، فیلدهای خام `highlightScores` و `summary` را از پاسخ API
‏Exa حفظ می‌کنند.

### حالت‌های جست‌وجو

| حالت             | توضیحات                       |
| ---------------- | --------------------------------- |
| `auto`           | Exa بهترین حالت را انتخاب می‌کند (پیش‌فرض) |
| `neural`         | جست‌وجوی معنایی/مبتنی بر مفهوم     |
| `fast`           | جست‌وجوی سریع کلیدواژه‌ای              |
| `deep`           | جست‌وجوی عمیق و جامع              |
| `deep-reasoning` | جست‌وجوی عمیق همراه با استدلال        |
| `instant`        | سریع‌ترین نتایج                   |

## نکات

- `count` تا سقف 100 را، مشروط به محدودیت‌های نوع جست‌وجوی Exa، می‌پذیرد.
- نتایج به‌طور پیش‌فرض برای 15 دقیقه کش می‌شوند. برای تغییر کش و
  زمان پایان درخواست برای همهٔ ارائه‌دهندگان `web_search`، از جمله Exa،
  مقادیر مشترک `tools.web.search.cacheTtlMinutes` (دقیقه) و
  `tools.web.search.timeoutSeconds` (پیش‌فرض 30 ثانیه) را پیکربندی کنید.

## مرتبط

- [نمای کلی جست‌وجوی وب](/fa/tools/web) -- همهٔ ارائه‌دهندگان و تشخیص خودکار
- [Brave Search](/fa/tools/brave-search) -- نتایج ساختاریافته با فیلترهای کشور/زبان
- [Perplexity Search](/fa/tools/perplexity-search) -- نتایج ساختاریافته با فیلتر دامنه
