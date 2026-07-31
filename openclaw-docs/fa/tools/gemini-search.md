---
read_when:
    - می‌خواهید از Gemini برای web_search استفاده کنید
    - به GEMINI_API_KEY یا models.providers.google.apiKey نیاز دارید
    - می‌خواهید پاسخ‌ها بر جست‌وجوی Google مبتنی باشند
summary: جست‌وجوی وب Gemini با اتکا به Google Search
title: جست‌وجوی Gemini
x-i18n:
    generated_at: "2026-07-27T16:23:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4c7cb55fb185adfda01ab6b3c6434ab6e3ee31162733c752d4c81328bce9a6cd
    source_path: tools/gemini-search.md
    workflow: 16
---

OpenClaw از مدل‌های Gemini با قابلیت داخلی
[اتصال به Google Search](https://ai.google.dev/gemini-api/docs/grounding)
پشتیبانی می‌کند که پاسخ‌های ترکیب‌شده توسط هوش مصنوعی و مبتنی بر نتایج زندهٔ Google Search را همراه با
ارجاعات برمی‌گرداند.

## دریافت کلید API

<Steps>
  <Step title="ایجاد کلید">
    به [Google AI Studio](https://aistudio.google.com/apikey) بروید و یک
    کلید API ایجاد کنید.
  </Step>
  <Step title="ذخیره‌سازی کلید">
    `GEMINI_API_KEY` را در محیط Gateway تنظیم کنید، از
    `models.providers.google.apiKey` دوباره استفاده کنید، یا یک کلید اختصاصی جست‌وجوی وب را از این طریق پیکربندی کنید:

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
      google: {
        config: {
          webSearch: {
            apiKey: "AIza...", // در صورت تنظیم GEMINI_API_KEY یا models.providers.google.apiKey اختیاری است
            baseUrl: "https://generativelanguage.googleapis.com/v1beta", // اختیاری؛ در صورت نبود از models.providers.google.baseUrl استفاده می‌کند
            model: "gemini-2.5-flash", // پیش‌فرض
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "gemini",
      },
    },
  },
}
```

**اولویت اعتبارنامه‌ها:** جست‌وجوی وب Gemini ابتدا از
`plugins.entries.google.config.webSearch.apiKey`، سپس `GEMINI_API_KEY`،
و بعد `models.providers.google.apiKey` استفاده می‌کند. برای نشانی‌های پایه،
`plugins.entries.google.config.webSearch.baseUrl` اختصاصی بر
`models.providers.google.baseUrl` اولویت دارد.

برای نصب Gateway، کلیدهای محیطی را در `~/.openclaw/.env` قرار دهید.

## نحوهٔ کار

برخلاف ارائه‌دهندگان سنتی جست‌وجو که فهرستی از پیوندها و قطعه‌متن‌ها را برمی‌گردانند،
Gemini با استفاده از اتصال به Google Search، پاسخ‌های ترکیب‌شده توسط هوش مصنوعی را همراه با
ارجاعات درون‌متنی تولید می‌کند. نتایج هم پاسخ ترکیب‌شده و هم نشانی‌های URL منبع
را شامل می‌شوند.

- نشانی‌های URL ارجاع از اتصال Gemini، از طریق یک درخواست HEAD در مسیر واکشی
  محافظت‌شده در برابر SSRF متعلق به OpenClaw، به‌طور خودکار از نشانی‌های تغییرمسیر Google
  به نشانی‌های مستقیم تبدیل می‌شوند (دنبال‌کردن تغییرمسیر، اعتبارسنجی http/https).
- حل تغییرمسیر از پیش‌فرض‌های سخت‌گیرانهٔ SSRF استفاده می‌کند؛ بنابراین تغییرمسیر به
  مقصدهای خصوصی/داخلی مسدود می‌شود.

## پارامترهای پشتیبانی‌شده

جست‌وجوی Gemini از `query`، `freshness`، `date_after` و `date_before` پشتیبانی می‌کند.

`count` برای سازگاری مشترک با `web_search` پذیرفته می‌شود، اما اتصال Gemini
همچنان به‌جای فهرستی با N نتیجه، یک پاسخ ترکیب‌شده همراه با ارجاعات
برمی‌گرداند.

`freshness` مقادیر `day`، `week`، `month`، `year` و میان‌برهای مشترک
`pd`، `pw`، `pm` و `py` را می‌پذیرد. `day`/`pd` به‌جای یک بازهٔ سخت‌گیرانهٔ 24 ساعته، دستور تازگی را به پرس‌وجوی Gemini
اضافه می‌کند. `week`، `month`، `year` و بازه‌های صریح
`date_after`/`date_before` مقدار
`timeRangeFilter` در اتصال Google Search متعلق به Gemini را تنظیم می‌کنند. `country`، `language` و `domain_filter` پشتیبانی نمی‌شوند.

## انتخاب مدل

مدل پیش‌فرض `gemini-2.5-flash` است (سریع و مقرون‌به‌صرفه). هر مدل Gemini
که از اتصال پشتیبانی کند، از طریق
`plugins.entries.google.config.webSearch.model` قابل استفاده است.

## بازنویسی نشانی پایه

وقتی جست‌وجوی وب Gemini باید از طریق پراکسی اپراتور یا نقطهٔ پایانی سفارشیِ سازگار با Gemini
مسیریابی شود، `plugins.entries.google.config.webSearch.baseUrl` را تنظیم کنید. اگر
تنظیم نشده باشد، جست‌وجوی وب Gemini دوباره از `models.providers.google.baseUrl` استفاده می‌کند. مقدار سادهٔ
`https://generativelanguage.googleapis.com` به
`https://generativelanguage.googleapis.com/v1beta` نرمال‌سازی می‌شود؛ مسیرهای پراکسی سفارشی
پس از حذف اسلش‌های انتهایی، همان‌گونه که ارائه شده‌اند حفظ می‌شوند.

## مرتبط

- [نمای کلی جست‌وجوی وب](/fa/tools/web) -- همهٔ ارائه‌دهندگان و تشخیص خودکار
- [جست‌وجوی Brave](/fa/tools/brave-search) -- نتایج ساخت‌یافته همراه با قطعه‌متن‌ها
- [جست‌وجوی Perplexity](/fa/tools/perplexity-search) -- نتایج ساخت‌یافته + استخراج محتوا
