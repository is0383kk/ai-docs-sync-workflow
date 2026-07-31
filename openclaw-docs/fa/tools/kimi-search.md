---
read_when:
    - می‌خواهید از Kimi برای web_search استفاده کنید
    - به یک KIMI_API_KEY یا MOONSHOT_API_KEY نیاز دارید
summary: جست‌وجوی وب Kimi از طریق جست‌وجوی وب Moonshot
title: جست‌وجوی Kimi
x-i18n:
    generated_at: "2026-07-27T16:03:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 65e5f8c9f3b607dbcc3256c51a6a083864e31f65ed2a751d2d500abeb35ba844
    source_path: tools/kimi-search.md
    workflow: 16
---

Kimi یک ارائه‌دهندهٔ `web_search` است که از جست‌وجوی وب بومی Moonshot بهره می‌برد. Moonshot
به‌جای بازگرداندن فهرستی رتبه‌بندی‌شده از نتایج، مشابه ارائه‌دهندگان پاسخ مستند
Gemini و Grok، یک پاسخ واحد همراه با ارجاع‌های درون‌متنی تولید می‌کند.

## راه‌اندازی

<Steps>
  <Step title="ایجاد کلید">
    یک کلید API از [Moonshot AI](https://platform.moonshot.cn/) دریافت کنید.
  </Step>
  <Step title="ذخیره کلید">
    `KIMI_API_KEY` یا `MOONSHOT_API_KEY` را در محیط Gateway تنظیم کنید (برای نصب
    Gateway، آن را به `~/.openclaw/.env` اضافه کنید)، یا از طریق دستور زیر پیکربندی کنید:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

انتخاب **Kimi** هنگام `openclaw onboard` یا `openclaw configure --section web`
موارد زیر را نیز درخواست می‌کند:

- منطقه API متعلق به Moonshot:‏ `https://api.moonshot.ai/v1` یا `https://api.moonshot.cn/v1`
- مدل جست‌وجوی وب (پیش‌فرض `kimi-k2.6`)

## پیکربندی

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // در صورت تنظیم KIMI_API_KEY یا MOONSHOT_API_KEY اختیاری است
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.6",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

در صورت حذف `tools.web.search.provider`، مقدار آن به‌طور خودکار از روی کلیدهای API موجود تشخیص داده می‌شود؛
اگر چند اعتبارنامه جست‌وجو پیکربندی شده‌اند، آن را صراحتاً روی `kimi` تنظیم کنید.

مقادیر مختص Kimi برای `apiKey`،‏ `baseUrl` و `model` را زیر
`plugins.entries.moonshot.config.webSearch` پیکربندی کنید.

مقادیر پیش‌فرض: در صورت حذف `baseUrl`، مقدار پیش‌فرض آن `https://api.moonshot.ai/v1` است و مقدار پیش‌فرض
`model` نیز `kimi-k2.6` است.

اگر ترافیک گفت‌وگو از میزبان چین استفاده کند (`models.providers.moonshot.baseUrl`:
`https://api.moonshot.cn/v1`) و مقدار `baseUrl` خود Kimi تنظیم نشده باشد، `web_search` متعلق به Kimi
به‌طور خودکار همان میزبان را دوباره استفاده می‌کند تا کلیدهای `.cn` به‌اشتباه به نقطه پایانی
بین‌المللی ارسال نشوند (آن نقطه پایانی برای این کلیدها HTTP 401 برمی‌گرداند). برای نادیده گرفتن این
وراثت، `baseUrl` متعلق به Kimi را صراحتاً تنظیم کنید.

## الزام مستندسازی

OpenClaw تنها زمانی نتیجهٔ `web_search` متعلق به Kimi را برمی‌گرداند که پاسخ Moonshot
شامل شواهد بومی مستندسازی جست‌وجوی وب، مانند بازپخش فراخوانی ابزار `$web_search`،
`search_results` یا URLهای ارجاع باشد. اگر Kimi بدون هیچ مستندسازی مستقیماً پاسخ دهد
(برای مثال «نمی‌توانم در اینترنت مرور کنم»)، OpenClaw به‌جای درنظرگرفتن آن متن به‌عنوان نتیجهٔ
جست‌وجو، خطای `kimi_web_search_ungrounded` را برمی‌گرداند. پرس‌وجو را دوباره امتحان کنید، به یک ارائه‌دهندهٔ
ساخت‌یافته مانند Brave تغییر دهید، یا هنگامی که از قبل URL مقصد را دارید از
`web_fetch` / ابزار مرورگر استفاده کنید.

## پارامترهای ابزار

| پارامتر                                                       | پشتیبانی                                                                                                                |
| --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `query`                                                         | بله                                                                                                                      |
| `count`                                                         | برای سازگاری میان ارائه‌دهندگان پذیرفته می‌شود، اما نادیده گرفته می‌شود: Kimi همیشه یک پاسخ ترکیبی بازمی‌گرداند، نه فهرستی از N نتیجه |
| `country`, `language`, `freshness`, `date_after`, `date_before` | خیر                                                                                                                       |

## مرتبط

- [نمای کلی جست‌وجوی وب](/fa/tools/web) - همهٔ ارائه‌دهندگان و تشخیص خودکار
- [Moonshot AI](/fa/providers/moonshot) - مستندات مدل Moonshot و ارائه‌دهندهٔ Kimi Coding
- [جست‌وجوی Gemini](/fa/tools/gemini-search) - پاسخ‌های تولیدشده با هوش مصنوعی از طریق مستندسازی Google
- [جست‌وجوی Grok](/fa/tools/grok-search) - پاسخ‌های تولیدشده با هوش مصنوعی از طریق مستندسازی xAI
