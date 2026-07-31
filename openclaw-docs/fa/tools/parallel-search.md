---
read_when:
    - می‌خواهید بدون کلید API در وب جست‌وجو کنید
    - شما API جست‌وجوی پولی Parallel را می‌خواهید
    - می‌خواهید گزیده‌های فشرده بر اساس کارایی زمینه برای LLM رتبه‌بندی شوند
summary: جست‌وجوی موازی -- گزیده‌های فشرده و بهینه‌شده برای LLM از منابع وب
title: جست‌وجوی موازی
x-i18n:
    generated_at: "2026-07-27T16:24:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eff693f286015b287bbdacf44f11ff6f07f2f7d2605ef6f09259e7402b40515e
    source_path: tools/parallel-search.md
    workflow: 16
---

Plugin مربوط به Parallel دو ارائه‌دهندهٔ `web_search` از [Parallel](https://parallel.ai/) فراهم می‌کند که هر دو گزیده‌های رتبه‌بندی‌شده و بهینه‌شده برای LLM را از یک نمایهٔ وب ساخته‌شده برای عامل‌های هوش مصنوعی برمی‌گردانند:

| ارائه‌دهنده               | شناسه              | احراز هویت                                                                                       |
| ---------------------- | --------------- | ------------------------------------------------------------------------------------------ |
| جست‌وجوی Parallel (رایگان) | `parallel-free` | هیچ‌کدام -- [Search MCP](https://docs.parallel.ai/integrations/mcp/search-mcp) رایگان Parallel |
| جست‌وجوی Parallel        | `parallel`      | `PARALLEL_API_KEY` -- API جست‌وجوی پولی، محدودیت نرخ بالاتر و تنظیم هدف             |

برای انتخاب صریح یکی از آن‌ها، `tools.web.search.provider` را روی `parallel-free` یا `parallel` تنظیم کنید؛ هیچ‌کدام به‌طور خودکار شناسایی نمی‌شوند.

<Note>
  مدل‌های مستقیم OpenAI Responses (`api: "openai-responses"`، ارائه‌دهندهٔ
  `openai`، نشانی پایهٔ رسمی API) هنگامی که `tools.web.search.provider` تنظیم نشده، خالی، `"auto"`،
  یا `"openai"` باشد، به‌طور خودکار از جست‌وجوی وب بومی میزبانی‌شدهٔ OpenAI استفاده می‌کنند؛
  بنابراین به‌طور پیش‌فرض Parallel را دور می‌زنند. برای هدایت آن‌ها از طریق Parallel، به‌جای آن
  `tools.web.search.provider` را روی `parallel-free` یا `parallel` تنظیم کنید. به [نمای کلی جست‌وجوی وب](/fa/tools/web) مراجعه کنید.
</Note>

## نصب Plugin

```bash
openclaw plugins install @openclaw/parallel-plugin
openclaw gateway restart
```

## کلید API (ارائه‌دهندهٔ پولی)

`parallel-free` به کلید نیاز ندارد، اما همچنان باید به‌طور صریح انتخاب شود. ارائه‌دهندهٔ پولی
`parallel` به یک کلید API نیاز دارد:

<Steps>
  <Step title="ایجاد حساب">
    در [platform.parallel.ai](https://platform.parallel.ai) ثبت‌نام کنید و
    از داشبورد خود یک کلید API بسازید.
  </Step>
  <Step title="ذخیره‌سازی کلید">
    `PARALLEL_API_KEY` را در محیط Gateway تنظیم کنید، یا از این طریق پیکربندی کنید:

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
      parallel: {
        config: {
          webSearch: {
            apiKey: "par-...", // در صورت تنظیم PARALLEL_API_KEY اختیاری است
            baseUrl: "https://api.parallel.ai", // اختیاری؛ OpenClaw ‏/v1/search را می‌افزاید
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        // ‏"parallel-free" برای Search MCP رایگان، یا "parallel" برای
        // ارائه‌دهندهٔ متکی به API پولی که اینجا نشان داده شده است.
        provider: "parallel",
      },
    },
  },
}
```

**جایگزین محیطی:** `PARALLEL_API_KEY` را در محیط Gateway تنظیم کنید. برای نصب Gateway، آن را در `~/.openclaw/.env` قرار دهید.

## بازنویسی نشانی پایه

فقط برای ارائه‌دهندهٔ پولی `parallel` اعمال می‌شود؛ `parallel-free` همیشه از
`https://search.parallel.ai/mcp` استفاده می‌کند و این تنظیم را نادیده می‌گیرد.

برای هدایت درخواست‌های پولی از طریق یک پراکسی سازگار یا نقطهٔ پایانی جایگزین
(برای مثال، Cloudflare AI Gateway)، `plugins.entries.parallel.config.webSearch.baseUrl` را تنظیم کنید.
OpenClaw میزبان‌های فاقد طرح را با افزودن `https://` در ابتدا عادی‌سازی می‌کند
و `/v1/search` را می‌افزاید، مگر اینکه مسیر از قبل به آن ختم شود. نقطهٔ پایانی
حل‌شده بخشی از کلید کش جست‌وجو است، بنابراین نتایج نقطه‌های پایانی مختلف هرگز
به‌اشتراک گذاشته نمی‌شوند.

## پارامترهای ابزار

هر دو ارائه‌دهنده ساختار بومی جست‌وجوی Parallel را ارائه می‌کنند تا مدل یک
هدف به زبان طبیعی را همراه با چند پرس‌وجوی کلیدواژه‌ای کوتاه تکمیل کند؛ ترکیبی
که Parallel برای دستیابی به بهترین نتایج [توصیه می‌کند](https://docs.parallel.ai/search/best-practices).

<ParamField path="objective" type="string" required>
شرحی به زبان طبیعی از پرسش یا هدف زیربنایی (حداکثر 5000 نویسه).
باید خودبسنده باشد.
</ParamField>

<ParamField path="search_queries" type="string[]" required>
پرس‌وجوهای جست‌وجوی کلیدواژه‌ای مختصر، هرکدام 3-6 واژه (1-5 ورودی، حداکثر 200 نویسه
برای هرکدام). برای بهترین نتایج، 2-3 پرس‌وجوی متنوع ارائه کنید.
</ParamField>

<ParamField path="count" type="number">
تعداد نتایجی که باید برگردانده شوند (1-40).
</ParamField>

<ParamField path="session_id" type="string">
شناسهٔ نشست اختیاری Parallel از `sessionId` نتیجهٔ قبلی. آن را در
جست‌وجوهای پیگیری همان وظیفه ارسال کنید تا Parallel فراخوانی‌های مرتبط را گروه‌بندی
و نتایج بعدی را بهبود دهد. حداکثر 1000 نویسه در `parallel`؛ Search MCP رایگان
`parallel-free` آن را به 100 محدود می‌کند. شناسه‌ای فراتر از حد مجاز حذف می‌شود
(پولی) یا یک شناسهٔ تازه ساخته می‌شود (رایگان).
</ParamField>

<ParamField path="client_model" type="string">
شناسهٔ اختیاری مدلی که فراخوانی را انجام می‌دهد (برای مثال `claude-opus-4-7`،
`gpt-5.6-sol`)، حداکثر 100 نویسه. به Parallel امکان می‌دهد تنظیمات پیش‌فرض را
برای قابلیت‌های مدل شما متناسب کند. نامک دقیق مدل فعال را ارسال کنید؛ آن را به
نام مستعار خانواده کوتاه نکنید.
</ParamField>

## نکات

- Parallel نتایج را برای سودمندی در استدلال LLM رتبه‌بندی و فشرده می‌کند، نه برای
  کلیک‌کردن انسان؛ بنابراین به‌جای محتوای کامل صفحه، انتظار گزیده‌های متراکم برای هر نتیجه را داشته باشید.
- گزیده‌های نتیجه به‌شکل آرایهٔ `excerpts` برمی‌گردند و برای سازگاری با قرارداد عمومی
  `web_search`، در `description` نیز به‌هم پیوسته می‌شوند.
- هر دو ارائه‌دهنده یک `session_id` برمی‌گردانند؛ OpenClaw آن را در بار ابزار
  به‌شکل `sessionId` ارائه می‌کند تا فراخوان‌ها بتوانند جست‌وجوهای پیگیری را گروه‌بندی کنند. شناسهٔ
  نشستی که Parallel تولید کرده است (شناسه‌ای که فراخوان ارائه نکرده) از ورودی کش کنار گذاشته می‌شود،
  زیرا وظایف نامرتبط با پرس‌وجوهای یکسان نباید آن را به ارث ببرند.
- `searchId`، `warnings` و `usage` از Parallel، در صورت وجود، بدون تغییر عبور داده می‌شوند.
- OpenClaw همیشه تعداد نتیجهٔ حل‌شده را به‌صورت
  `advanced_settings.max_results` (`parallel`) به Parallel ارسال می‌کند یا پس از پاسخ با اندازهٔ ثابت Parallel،
  `count` را در سمت کارخواه اعمال می‌کند (`parallel-free`). ابتدا آرگومان
  `count` فراخوان، سپس `tools.web.search.maxResults` و در غیر این صورت مقدار پیش‌فرض عمومی
  `web_search` در OpenClaw (5) اولویت دارد؛ مقدار پیش‌فرض API خود Parallel برابر با 10 است.
- نتایج به‌طور پیش‌فرض برای 15 دقیقه کش می‌شوند (`cacheTtlMinutes`).
- `parallel-free` هنگامی که فراخوان شناسه‌ای ارائه نکند، در هر فراخوان از طریق دست‌دهی MCP خود
  یک `session_id` تازه می‌سازد؛ `parallel` در آن حالت آن را تنظیم‌نشده باقی می‌گذارد.

## مرتبط

- [نمای کلی جست‌وجوی وب](/fa/tools/web) -- همهٔ ارائه‌دهندگان و شناسایی خودکار
- [جست‌وجوی Exa](/fa/tools/exa-search) -- جست‌وجوی عصبی همراه با استخراج محتوا
- [جست‌وجوی Perplexity](/fa/tools/perplexity-search) -- نتایج ساختاریافته با پالایش دامنه
