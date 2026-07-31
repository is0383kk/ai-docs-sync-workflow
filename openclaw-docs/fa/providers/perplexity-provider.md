---
read_when:
    - می‌خواهید Perplexity را به‌عنوان ارائه‌دهندهٔ جست‌وجوی وب پیکربندی کنید
    - به کلید API ‏Perplexity یا راه‌اندازی پراکسی OpenRouter نیاز دارید
summary: راه‌اندازی ارائه‌دهنده جست‌وجوی وب Perplexity (کلید API، حالت‌های جست‌وجو، فیلترسازی)
title: پرپلکسیتی
x-i18n:
    generated_at: "2026-07-27T14:36:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea76a5cb7befce95756e9bcc8f9c1637fac87711d02d8a486ec2a1b9f51b73dc
    source_path: providers/perplexity-provider.md
    workflow: 16
---

Plugin ‏Perplexity یک ارائه‌دهندهٔ `web_search` را با دو روش انتقال ثبت می‌کند: API بومی جست‌وجوی Perplexity (نتایج ساختاریافته با فیلترها) و تکمیل‌های چت Sonar ‏Perplexity، به‌صورت مستقیم یا از طریق OpenRouter (پاسخ‌های ترکیب‌شده با هوش مصنوعی همراه با ارجاعات).

<Note>
این صفحه راه‌اندازی **ارائه‌دهندهٔ** Perplexity را پوشش می‌دهد. برای **ابزار** Perplexity (نحوهٔ استفادهٔ عامل از آن)، به [جست‌وجوی Perplexity](/fa/tools/perplexity-search) مراجعه کنید.
</Note>

| ویژگی      | مقدار                                                                  |
| ----------- | ---------------------------------------------------------------------- |
| نوع         | ارائه‌دهندهٔ جست‌وجوی وب (نه ارائه‌دهندهٔ مدل)                         |
| احراز هویت | `PERPLEXITY_API_KEY` (بومی) یا `OPENROUTER_API_KEY` (از طریق OpenRouter) |
| مسیر پیکربندی | `plugins.entries.perplexity.config.webSearch.apiKey`                   |
| بازنویسی‌ها | `plugins.entries.perplexity.config.webSearch.baseUrl` / `.model`       |
| دریافت کلید | [perplexity.ai/settings/api](https://www.perplexity.ai/settings/api)   |

## نصب Plugin

```bash
openclaw plugins install @openclaw/perplexity-plugin
openclaw gateway restart
```

## شروع کار

<Steps>
  <Step title="تنظیم کلید API">
    ```bash
    openclaw configure --section web
    ```

    یا کلید را مستقیماً تنظیم کنید:

    ```bash
    openclaw config set plugins.entries.perplexity.config.webSearch.apiKey "pplx-xxxxxxxxxxxx"
    ```

    کلیدی که با نام `PERPLEXITY_API_KEY` یا `OPENROUTER_API_KEY` در محیط Gateway
    صادر شده باشد نیز کار می‌کند.

  </Step>
  <Step title="شروع جست‌وجو">
    وقتی کلید Perplexity به‌عنوان اعتبارنامهٔ جست‌وجوی موجود باشد، `web_search` آن را به‌طور خودکار شناسایی می‌کند؛ راه‌اندازی دیگری لازم نیست. برای تثبیت صریح ارائه‌دهنده:

    ```bash
    openclaw config set tools.web.search.provider perplexity
    ```

  </Step>
</Steps>

## حالت‌های جست‌وجو

Plugin روش انتقال را به‌ترتیب زیر تعیین می‌کند:

1. `webSearch.baseUrl` یا `webSearch.model` تنظیم شده باشد: صرف‌نظر از نوع کلید، همیشه تکمیل‌های چت Sonar را از طریق آن نقطهٔ پایانی مسیریابی می‌کند.
2. در غیر این صورت، منبع کلید نقطهٔ پایانی را تعیین می‌کند: پیشوند کلید پیکربندی‌شده روش انتقال را انتخاب می‌کند (پیکربندی بر متغیرهای محیطی اولویت دارد)؛ کلید محیطی مستقیماً از نقطهٔ پایانی متناظر خود استفاده می‌کند.

| پیشوند کلید | روش انتقال                                               | قابلیت‌ها                                             |
| ---------- | ---------------------------------------------------------- | ------------------------------------------------ |
| `pplx-`    | API بومی جست‌وجوی Perplexity ‏(`https://api.perplexity.ai`) | نتایج ساختاریافته، فیلترهای دامنه/زبان/تاریخ |
| `sk-or-`   | OpenRouter ‏(`https://openrouter.ai/api/v1`)، مدل Sonar   | پاسخ‌های ترکیب‌شده با هوش مصنوعی همراه با ارجاعات |

کلید پیکربندی‌شده با هر پیشوند دیگری نیز از API بومی جست‌وجو استفاده می‌کند. مسیر تکمیل‌های چت به‌طور پیش‌فرض از مدل `perplexity/sonar-pro` استفاده می‌کند؛ آن را با `plugins.entries.perplexity.config.webSearch.model` بازنویسی کنید.

## فیلترگذاری API بومی

| فیلتر                               | توضیحات                                                     | روش انتقال   |
| ------------------------------------ | --------------------------------------------------------------- | ----------- |
| `count`                              | تعداد نتایج در هر جست‌وجو، 1-10 (پیش‌فرض 5)                            | فقط بومی |
| `freshness`                          | بازهٔ تازگی: `day`، `week`، `month`، `year`                  | هر دو        |
| `country`                            | کد دوحرفی کشور (`us`، `de`، `jp`)                        | فقط بومی |
| `language`                           | کد زبان ISO 639-1 ‏(`en`، `fr`، `zh`)                      | فقط بومی |
| `date_after` / `date_before`         | بازهٔ تاریخ انتشار در `YYYY-MM-DD`                            | فقط بومی |
| `domain_filter`                      | حداکثر 20 دامنه؛ فهرست مجاز یا فهرست مسدود با پیشوند `-`، هرگز به‌صورت ترکیبی | فقط بومی |
| `max_tokens` / `max_tokens_per_page` | بودجهٔ محتوا برای همهٔ نتایج / برای هر صفحه                    | فقط بومی |

فیلترهای مختص حالت بومی در مسیر تکمیل‌های چت، خطایی توصیفی برمی‌گردانند.
`freshness` را نمی‌توان با `date_after`/`date_before` ترکیب کرد.

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="متغیر محیطی برای فرایندهای daemon">
    <Warning>
    کلیدی که فقط در یک پوستهٔ تعاملی صادر شده باشد، برای daemon ‏Gateway مبتنی بر
    launchd/systemd قابل مشاهده نیست، مگر اینکه آن محیط به‌صراحت وارد شود.
    کلید را در `~/.openclaw/.env` یا از طریق `env.shellEnv` تنظیم کنید تا
    فرایند Gateway بتواند آن را بخواند. برای ترتیب کامل اولویت‌ها، به
    [متغیرهای محیطی](/fa/help/environment) مراجعه کنید.
    </Warning>
  </Accordion>

  <Accordion title="راه‌اندازی پراکسی OpenRouter">
    برای مسیریابی جست‌وجوهای Perplexity از طریق OpenRouter، به‌جای کلید بومی Perplexity یک `OPENROUTER_API_KEY`
    (با پیشوند `sk-or-`) تنظیم کنید. OpenClaw کلید را شناسایی می‌کند و
    به‌طور خودکار به روش انتقال Sonar تغییر می‌دهد. این گزینه زمانی مفید است که
    صورت‌حساب OpenRouter را از قبل راه‌اندازی کرده‌اید و می‌خواهید ارائه‌دهندگان را در آنجا یکپارچه کنید.
  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="ابزار جست‌وجوی Perplexity" href="/fa/tools/perplexity-search" icon="magnifying-glass">
    نحوهٔ فراخوانی جست‌وجوهای Perplexity و تفسیر نتایج توسط عامل.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/configuration-reference" icon="gear">
    مرجع کامل پیکربندی، شامل ورودی‌های Plugin.
  </Card>
</CardGroup>
