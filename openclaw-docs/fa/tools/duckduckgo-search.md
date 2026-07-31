---
read_when:
    - یک ارائه‌دهندهٔ جست‌وجوی وب می‌خواهید که به کلید API نیاز نداشته باشد
    - می‌خواهید برای `web_search` از DuckDuckGo استفاده کنید
    - یک ارائه‌دهندهٔ جست‌وجوی بدون کلید می‌خواهید که صراحتاً انتخاب شده باشد
summary: جست‌وجوی وب DuckDuckGo -- ارائه‌دهنده بدون نیاز به کلید (آزمایشی، مبتنی بر HTML)
title: جست‌وجوی DuckDuckGo
x-i18n:
    generated_at: "2026-07-27T16:22:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 84e90532de276dcb3f73c67015dffe5f5a62be673e44a19053b2b1dfcb0986ac
    source_path: tools/duckduckgo-search.md
    workflow: 16
---

OpenClaw از DuckDuckGo به‌عنوان ارائه‌دهندهٔ `web_search` **بدون نیاز به کلید** پشتیبانی می‌کند. هیچ کلید API یا حسابی لازم نیست.

<Warning>
  DuckDuckGo یک یکپارچه‌سازی **آزمایشی و غیررسمی** است که صفحات جست‌وجوی HTML بدون JavaScript در DuckDuckGo را استخراج می‌کند و API رسمی نیست. احتمال اختلال‌های گاه‌به‌گاه ناشی از صفحات چالش ربات یا تغییرات HTML وجود دارد.
</Warning>

## راه‌اندازی

DuckDuckGo هرگز به‌طور خودکار انتخاب نمی‌شود، زیرا تشخیص خودکار فقط ارائه‌دهندگانی را در نظر می‌گیرد که اعتبارنامه‌های قابل‌استفاده دارند. آن را صراحتاً تنظیم کنید:

<Steps>
  <Step title="پیکربندی">
    ```bash
    openclaw configure --section web
    # ارائه‌دهنده "duckduckgo" را انتخاب کنید
    ```
  </Step>
</Steps>

## پیکربندی

ارائه‌دهنده را مستقیماً در پیکربندی تنظیم کنید:

```json5
{
  tools: {
    web: {
      search: {
        provider: "duckduckgo",
      },
    },
  },
}
```

تنظیمات اختیاری در سطح Plugin برای منطقه و SafeSearch:

```json5
{
  plugins: {
    entries: {
      duckduckgo: {
        config: {
          webSearch: {
            region: "us-en", // کد منطقه DuckDuckGo
            safeSearch: "moderate", // "strict"، "moderate" یا "off"
          },
        },
      },
    },
  },
}
```

## پارامترهای ابزار

<ParamField path="query" type="string" required>
عبارت جست‌وجو.
</ParamField>

<ParamField path="count" type="number" default="5">
تعداد نتایج بازگردانده‌شده (1-10).
</ParamField>

<ParamField path="region" type="string">
کد منطقه DuckDuckGo (برای مثال `us-en`، `uk-en`، `de-de`).
</ParamField>

<ParamField path="safeSearch" type="'strict' | 'moderate' | 'off'" default="moderate">
سطح SafeSearch.
</ParamField>

پارامترهای ابزار `region` و `safeSearch` در هر جست‌وجو، مقادیر پیکربندی Plugin در بالا را لغو می‌کنند.

## نکات

- **بدون کلید API** -- پس از انتخاب DuckDuckGo به‌عنوان ارائه‌دهندهٔ `web_search` کار می‌کند.
- **آزمایشی** -- صفحات جست‌وجوی HTML بدون JavaScript در DuckDuckGo را استخراج می‌کند و API یا SDK رسمی نیست. نتایج به ساختار صفحه وابسته‌اند که ممکن است بدون اطلاع تغییر کند.
- **خطر چالش ربات** -- DuckDuckGo ممکن است هنگام استفادهٔ سنگین یا خودکار، CAPTCHA نمایش دهد یا درخواست‌ها را مسدود کند.
- **فقط انتخاب صریح** -- تشخیص خودکار OpenClaw فقط ارائه‌دهندگانی را در نظر می‌گیرد که اعتبارنامه‌های قابل‌استفاده دارند؛ بنابراین ارائه‌دهنده‌ای بدون نیاز به کلید مانند DuckDuckGo هرگز به‌طور خودکار انتخاب نمی‌شود و باید `provider: "duckduckgo"` را تنظیم کنید.
- **در صورت پیکربندی‌نشدن، مقدار پیش‌فرض SafeSearch برابر `moderate` است**.

<Tip>
  برای استفاده در محیط عملیاتی، [Brave Search](/fa/tools/brave-search) (با سطح رایگان) یا ارائه‌دهندهٔ دیگری مبتنی بر API را در نظر بگیرید.
</Tip>

## مرتبط

- [نمای کلی جست‌وجوی وب](/fa/tools/web) -- همهٔ ارائه‌دهندگان و تشخیص خودکار
- [Brave Search](/fa/tools/brave-search) -- نتایج ساختاریافته با سطح رایگان
- [جست‌وجوی Exa](/fa/tools/exa-search) -- جست‌وجوی عصبی با استخراج محتوا
