---
read_when:
    - جست‌وجوی وب با پشتیبانی Tavily می‌خواهید
    - به یک کلید API برای Tavily نیاز دارید
    - می‌خواهید از Tavily به‌عنوان ارائه‌دهندهٔ web_search استفاده کنید
    - می‌خواهید محتوا را از URLها استخراج کنید
summary: ابزارهای جست‌وجو و استخراج Tavily
title: Tavily
x-i18n:
    generated_at: "2026-07-27T14:46:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9a61351872eb8aecb0b3ada9b573ee8d3db1dcec3d7bd74074446fbe9dc1f274
    source_path: tools/tavily.md
    workflow: 16
---

[Tavily](https://tavily.com) یک API جست‌وجو است که برای برنامه‌های هوش مصنوعی طراحی شده است. OpenClaw آن را به دو روش ارائه می‌کند:

- به‌عنوان ارائه‌دهنده `web_search` برای ابزار جست‌وجوی عمومی
- به‌عنوان ابزارهای صریح Plugin:‏ `tavily_search` و `tavily_extract`

Tavily نتایج ساختاریافته و بهینه‌شده برای استفاده LLMها را برمی‌گرداند و عمق جست‌وجوی قابل‌پیکربندی، پالایش موضوعی، پالایش دامنه، خلاصه پاسخ‌های تولیدشده با هوش مصنوعی و استخراج محتوا از URLها (از جمله صفحه‌های رندرشده با JavaScript) را ارائه می‌دهد.

| ویژگی  | مقدار                                                                                         |
| --------- | --------------------------------------------------------------------------------------------- |
| شناسه Plugin | `tavily`                                                                                      |
| بسته   | `@openclaw/tavily-plugin`                                                                     |
| احراز هویت      | متغیر محیطی `TAVILY_API_KEY` یا پیکربندی `apiKey`                                                   |
| URL پایه  | `https://api.tavily.com` (پیش‌فرض)؛ متغیر محیطی `TAVILY_BASE_URL` یا پیکربندی `baseUrl` برای بازنویسی |
| مهلت‌های زمانی  | 30s برای جست‌وجو، 60s برای استخراج (پیش‌فرض)                                                             |
| ابزارها     | `tavily_search`، `tavily_extract`                                                             |

## شروع به کار

<Steps>
  <Step title="نصب Plugin">
    ```bash
    openclaw plugins install @openclaw/tavily-plugin
    ```
  </Step>
  <Step title="دریافت کلید API">
    در [tavily.com](https://tavily.com) یک حساب Tavily ایجاد کنید، سپس در داشبورد یک کلید API بسازید.
  </Step>
  <Step title="پیکربندی Plugin و ارائه‌دهنده">
    ```json5
    {
      plugins: {
        entries: {
          tavily: {
            enabled: true,
            config: {
              webSearch: {
                apiKey: "tvly-...", // در صورت تنظیم TAVILY_API_KEY اختیاری است
                baseUrl: "https://api.tavily.com",
              },
            },
          },
        },
      },
      tools: {
        web: {
          search: {
            provider: "tavily",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="تأیید اجرای جست‌وجو">
    یک `web_search` را از هر عاملی راه‌اندازی کنید یا مستقیماً `tavily_search` را فراخوانی کنید.
  </Step>
</Steps>

<Tip>
انتخاب Tavily در راه‌اندازی اولیه یا `openclaw configure --section web`، در صورت نیاز Plugin رسمی Tavily را نصب و فعال می‌کند.
</Tip>

## مرجع ابزار

### `tavily_search`

زمانی از این ابزار استفاده کنید که به‌جای `web_search` عمومی، کنترل‌های جست‌وجوی ویژه Tavily را می‌خواهید.

| پارامتر         | نوع         | محدودیت‌ها / پیش‌فرض                  | توضیحات                                   |
| ----------------- | ------------ | -------------------------------------- | --------------------------------------------- |
| `query`           | رشته       | الزامی                               | رشته پرس‌وجوی جست‌وجو.                          |
| `search_depth`    | شمارشی         | `basic` (پیش‌فرض)، `advanced`          | `advanced` کندتر است، اما ارتباط بیشتری دارد.    |
| `topic`           | شمارشی         | `general` (پیش‌فرض)، `news`، `finance` | پالایش بر اساس خانواده موضوعی.                       |
| `max_results`     | عدد صحیح      | 1-20، پیش‌فرض `5`                      | تعداد نتایج.                            |
| `include_answer`  | بولی      | پیش‌فرض `false`                        | شامل‌کردن خلاصه پاسخ تولیدشده با هوش مصنوعی Tavily. |
| `time_range`      | شمارشی         | `day`، `week`، `month`، `year`         | پالایش نتایج بر اساس تازگی.                    |
| `include_domains` | آرایه رشته | (هیچ‌کدام)                                 | فقط نتایج این دامنه‌ها را شامل می‌شود.      |
| `exclude_domains` | آرایه رشته | (هیچ‌کدام)                                 | نتایج این دامنه‌ها را حذف می‌کند.           |

موازنه عمق جست‌وجو:

| عمق      | سرعت  | میزان ارتباط | مناسب برای                             |
| ---------- | ------ | --------- | ------------------------------------ |
| `basic`    | سریع‌تر | زیاد      | پرس‌وجوهای عمومی (پیش‌فرض).   |
| `advanced` | کندتر | بیشترین   | پژوهش دقیق و یافتن واقعیت‌ها. |

### `tavily_extract`

از این ابزار برای استخراج محتوای پاک از یک یا چند URL استفاده کنید. صفحه‌های رندرشده با JavaScript را مدیریت می‌کند و برای استخراج هدفمند، قطعه‌بندی متمرکز بر پرس‌وجو را پشتیبانی می‌کند.

| پارامتر           | نوع         | محدودیت‌ها / پیش‌فرض         | توضیحات                                                 |
| ------------------- | ------------ | ----------------------------- | ----------------------------------------------------------- |
| `urls`              | آرایه رشته | الزامی، 1-20                | URLهایی که محتوا باید از آن‌ها استخراج شود.                               |
| `query`             | رشته       | (اختیاری)                    | رتبه‌بندی مجدد قطعه‌های استخراج‌شده بر اساس ارتباط با این پرس‌وجو.         |
| `extract_depth`     | شمارشی         | `basic` (پیش‌فرض)، `advanced` | برای صفحه‌های سنگین از نظر JS، برنامه‌های تک‌صفحه‌ای یا جدول‌های پویا از `advanced` استفاده کنید. |
| `chunks_per_source` | عدد صحیح      | 1-5؛ **به `query` نیاز دارد**     | تعداد قطعه‌های بازگردانده‌شده برای هر URL. اگر بدون `query` تنظیم شود، خطا می‌دهد.     |
| `include_images`    | بولی      | پیش‌فرض `false`               | شامل‌کردن URL تصاویر در نتایج.                              |

موازنه عمق استخراج:

| عمق      | زمان استفاده                                |
| ---------- | ------------------------------------------ |
| `basic`    | صفحه‌های ساده. ابتدا این گزینه را امتحان کنید.              |
| `advanced` | برنامه‌های تک‌صفحه‌ای رندرشده با JS، محتوای پویا و جدول‌ها. |

<Tip>
فهرست‌های بزرگ‌تر URL را میان چند فراخوانی `tavily_extract` تقسیم کنید (حداکثر 20 مورد در هر درخواست). برای دریافت فقط محتوای مرتبط به‌جای صفحه‌های کامل، از `query` همراه با `chunks_per_source` استفاده کنید.
</Tip>

## انتخاب ابزار مناسب

| نیاز                                 | ابزار             |
| ------------------------------------ | ---------------- |
| جست‌وجوی سریع وب، بدون گزینه‌های ویژه | `web_search`     |
| جست‌وجو با عمق، موضوع و پاسخ‌های هوش مصنوعی | `tavily_search`  |
| استخراج محتوا از URLهای مشخص   | `tavily_extract` |

<Note>
ابزار عمومی `web_search` با Tavily به‌عنوان ارائه‌دهنده، از `query` و `count` (تا 20 نتیجه) پشتیبانی می‌کند. برای کنترل‌های ویژه Tavily ‏(`search_depth`، `topic`، `include_answer`، پالایش دامنه و بازه زمانی)، به‌جای آن از `tavily_search` استفاده کنید.
</Note>

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="ترتیب تفکیک کلید API">
    کلاینت Tavily کلید API خود را به این ترتیب جست‌وجو می‌کند:

    1. `plugins.entries.tavily.config.webSearch.apiKey` (تفکیک‌شده از طریق SecretRefs).
    2. `TAVILY_API_KEY` از محیط Gateway.

    اگر هیچ‌کدام موجود نباشند، هر دو `tavily_search` و `tavily_extract` خطای راه‌اندازی ایجاد می‌کنند.

  </Accordion>

  <Accordion title="URL پایه سفارشی">
    اگر Tavily را از طریق پراکسی ارائه می‌کنید، `plugins.entries.tavily.config.webSearch.baseUrl` را بازنویسی کنید یا `TAVILY_BASE_URL` را تنظیم کنید. پیکربندی بر متغیر محیطی اولویت دارد. مقدار پیش‌فرض `https://api.tavily.com` است.
  </Accordion>

  <Accordion title="`chunks_per_source` به `query` نیاز دارد">
    `tavily_extract` فراخوانی‌هایی را که `chunks_per_source` را بدون `query` ارسال کنند رد می‌کند. Tavily قطعه‌ها را بر اساس ارتباط با پرس‌وجو رتبه‌بندی می‌کند، بنابراین این پارامتر بدون پرس‌وجو بی‌معناست.
  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="نمای کلی جست‌وجوی وب" href="/fa/tools/web" icon="magnifying-glass">
    همه ارائه‌دهندگان و قواعد تشخیص خودکار.
  </Card>
  <Card title="Firecrawl" href="/fa/tools/firecrawl" icon="fire">
    جست‌وجو به‌همراه خزش و استخراج محتوا.
  </Card>
  <Card title="جست‌وجوی Exa" href="/fa/tools/exa-search" icon="binoculars">
    جست‌وجوی عصبی همراه با استخراج محتوا.
  </Card>
  <Card title="پیکربندی" href="/fa/gateway/configuration" icon="gear">
    طرح‌واره کامل پیکربندی برای ورودی‌های Plugin و مسیریابی ابزار.
  </Card>
</CardGroup>
