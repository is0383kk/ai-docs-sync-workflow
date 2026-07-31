---
read_when:
    - می‌خواهید web_search را فعال یا پیکربندی کنید
    - می‌خواهید x_search را فعال یا پیکربندی کنید
    - باید یک ارائه‌دهنده جست‌وجو انتخاب کنید
    - می‌خواهید تشخیص خودکار و انتخاب ارائه‌دهنده را درک کنید
sidebarTitle: Web Search
summary: web_search، x_search و web_fetch — جست‌وجوی وب، جست‌وجوی پست‌های X یا واکشی محتوای صفحه
title: جست‌وجوی وب
x-i18n:
    generated_at: "2026-07-27T14:55:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 997e51064b0cd08d0f30987aa038e2f4a98da22f1094974b45f59c18491bd979
    source_path: tools/web.md
    workflow: 16
---

`web_search` با ارائه‌دهنده پیکربندی‌شده شما وب را جست‌وجو می‌کند و
نتایج نرمال‌سازی‌شده را برمی‌گرداند که بر اساس پرس‌وجو به‌مدت 15 دقیقه در حافظه نهان نگه‌داری می‌شوند (قابل پیکربندی). OpenClaw
همچنین `x_search` را برای پست‌های X (که پیش‌تر Twitter نام داشت) و `web_fetch` را برای
واکشی سبک URL ارائه می‌کند. `web_fetch` همیشه به‌صورت محلی اجرا می‌شود؛ `web_search` هنگامی که Grok ارائه‌دهنده باشد
از طریق xAI Responses مسیریابی می‌شود، و `x_search` همیشه از
xAI Responses استفاده می‌کند.

<Info>
  `web_search` یک ابزار سبک HTTP است، نه خودکارسازی مرورگر. برای
  سایت‌های متکی به JS یا ورود به حساب، از [مرورگر وب](/fa/tools/browser) استفاده کنید. برای
  واکشی یک URL مشخص، از [واکشی وب](/fa/tools/web-fetch) استفاده کنید.
</Info>

## شروع سریع

<Steps>
  <Step title="انتخاب ارائه‌دهنده">
    یک ارائه‌دهنده انتخاب کنید و هرگونه راه‌اندازی الزامی را تکمیل کنید. برخی ارائه‌دهندگان
    به کلید نیاز ندارند و برخی دیگر به کلید API نیاز دارند. برای
    جزئیات، صفحات ارائه‌دهندگان در ادامه را ببینید.
  </Step>
  <Step title="پیکربندی">
    ```bash
    openclaw configure --section web
    ```
    این فرمان ارائه‌دهنده و هر اعتبارنامه لازم را ذخیره می‌کند. برای ارائه‌دهندگان
    مبتنی بر API، می‌توانید به‌جای آن متغیر محیطی ارائه‌دهنده را تنظیم کنید (برای نمونه
    `BRAVE_API_KEY`) و از این مرحله بگذرید.
  </Step>
  <Step title="استفاده">
    ```javascript
    await web_search({ query: "OpenClaw plugin SDK" });
    ```

    برای پست‌های X:

    ```javascript
    await x_search({ query: "dinner recipes" });
    ```

  </Step>
</Steps>

## انتخاب ارائه‌دهنده

<CardGroup cols={2}>
  <Card title="Brave Search" icon="shield" href="/fa/tools/brave-search">
    نتایج ساخت‌یافته همراه با قطعه‌های متنی. از حالت `llm-context` و فیلترهای کشور/زبان پشتیبانی می‌کند. سطح رایگان در دسترس است.
  </Card>
  <Card title="جست‌وجوی میزبانی‌شده Codex" icon="search" href="/fa/plugins/codex-harness">
    پاسخ‌های مستند و ترکیب‌شده با هوش مصنوعی از طریق حساب app-server مربوط به Codex شما.
  </Card>
  <Card title="DuckDuckGo" icon="bird" href="/fa/tools/duckduckgo-search">
    ارائه‌دهنده بدون کلید. به کلید API نیاز ندارد. یکپارچه‌سازی غیررسمی مبتنی بر HTML.
  </Card>
  <Card title="Exa" icon="brain" href="/fa/tools/exa-search">
    جست‌وجوی عصبی + کلیدواژه‌ای همراه با استخراج محتوا (بخش‌های برجسته، متن، خلاصه‌ها).
  </Card>
  <Card title="Firecrawl" icon="flame" href="/fa/tools/firecrawl">
    نتایج ساخت‌یافته. برای استخراج عمیق، بهترین عملکرد را در کنار `firecrawl_search` و `firecrawl_scrape` دارد.
  </Card>
  <Card title="Gemini" icon="sparkles" href="/fa/tools/gemini-search">
    پاسخ‌های ترکیب‌شده با هوش مصنوعی همراه با ارجاع، از طریق مستندسازی Google Search.
  </Card>
  <Card title="Grok" icon="zap" href="/fa/tools/grok-search">
    پاسخ‌های ترکیب‌شده با هوش مصنوعی همراه با ارجاع، از طریق مستندسازی وب xAI.
  </Card>
  <Card title="Kimi" icon="moon" href="/fa/tools/kimi-search">
    پاسخ‌های ترکیب‌شده با هوش مصنوعی همراه با ارجاع، از طریق جست‌وجوی وب Moonshot؛ بازگشت‌های چت بدون مستندات صریحاً ناموفق می‌شوند.
  </Card>
  <Card title="جست‌وجوی MiniMax" icon="globe" href="/fa/tools/minimax-search">
    نتایج ساخت‌یافته از طریق API جست‌وجوی MiniMax Token Plan.
  </Card>
  <Card title="جست‌وجوی وب Ollama" icon="globe" href="/fa/tools/ollama-search">
    جست‌وجو از طریق میزبان محلی Ollama که به آن وارد شده‌اید، یا API میزبانی‌شده Ollama.
  </Card>
  <Card title="Parallel" icon="layer-group" href="/fa/tools/parallel-search">
    API پولی Parallel Search ‏(`PARALLEL_API_KEY`)؛ محدودیت نرخ بالاتر و تنظیم هدف.
  </Card>
  <Card title="جست‌وجوی Parallel (رایگان)" icon="layer-group" href="/fa/tools/parallel-search">
    گزینه بدون کلید و نیازمند فعال‌سازی. Search MCP رایگان Parallel، همراه با گزیده‌های متراکم بهینه‌شده برای LLM و بدون کلید API.
  </Card>
  <Card title="Perplexity" icon="search" href="/fa/tools/perplexity-search">
    نتایج ساخت‌یافته همراه با کنترل‌های استخراج محتوا و فیلتر دامنه.
  </Card>
  <Card title="SearXNG" icon="server" href="/fa/tools/searxng-search">
    فراجست‌وجوی خودمیزبان. به کلید API نیاز ندارد. Google، Bing، DuckDuckGo و موارد دیگر را تجمیع می‌کند.
  </Card>
  <Card title="Tavily" icon="globe" href="/fa/tools/tavily">
    نتایج ساخت‌یافته همراه با عمق جست‌وجو، فیلتر موضوع و `tavily_extract` برای استخراج URL.
  </Card>
</CardGroup>

### مقایسه ارائه‌دهندگان

| ارائه‌دهنده                                         | سبک نتیجه                                                   | فیلترها                                          | کلید API                                                                                 |
| ------------------------------------------------ | -------------------------------------------------------------- | ------------------------------------------------ | --------------------------------------------------------------------------------------- |
| [Brave](/fa/tools/brave-search)                     | قطعه‌های متنی ساخت‌یافته                                            | کشور، زبان، زمان، حالت `llm-context`      | `BRAVE_API_KEY`                                                                         |
| [جست‌وجوی میزبانی‌شده Codex](/fa/plugins/codex-harness)    | ترکیب‌شده با هوش مصنوعی + URLهای منبع                                   | دامنه‌ها، اندازه زمینه، موقعیت مکانی کاربر             | ندارد؛ از ورود به حساب Codex/OpenAI استفاده می‌کند                                                         |
| [DuckDuckGo](/fa/tools/duckduckgo-search)           | قطعه‌های متنی ساخت‌یافته                                            | --                                               | ندارد (بدون کلید)                                                                         |
| [Exa](/fa/tools/exa-search)                         | ساخت‌یافته + استخراج‌شده                                         | حالت عصبی/کلیدواژه‌ای، تاریخ، استخراج محتوا    | `EXA_API_KEY`                                                                           |
| [Firecrawl](/fa/tools/firecrawl)                    | قطعه‌های متنی ساخت‌یافته                                            | از طریق ابزار `firecrawl_search`                      | `FIRECRAWL_API_KEY`                                                                     |
| [Gemini](/fa/tools/gemini-search)                   | ترکیب‌شده با هوش مصنوعی + ارجاع‌ها                                     | --                                               | `GEMINI_API_KEY`                                                                        |
| [Grok](/fa/tools/grok-search)                       | ترکیب‌شده با هوش مصنوعی + ارجاع‌ها                                     | --                                               | xAI OAuth، `XAI_API_KEY`، یا `plugins.entries.xai.config.webSearch.apiKey`              |
| [Kimi](/fa/tools/kimi-search)                       | ترکیب‌شده با هوش مصنوعی + ارجاع‌ها؛ در بازگشت‌های چت بدون مستندات ناموفق می‌شود | --                                               | `KIMI_API_KEY` / `MOONSHOT_API_KEY`                                                     |
| [جست‌وجوی MiniMax](/fa/tools/minimax-search)          | قطعه‌های متنی ساخت‌یافته                                            | منطقه (`global` / `cn`)                         | `MINIMAX_CODE_PLAN_KEY` / `MINIMAX_CODING_API_KEY` / `MINIMAX_OAUTH_TOKEN`              |
| [جست‌وجوی وب Ollama](/fa/tools/ollama-search)        | قطعه‌های متنی ساخت‌یافته                                            | --                                               | برای میزبان‌های محلی واردشده ندارد؛ `OLLAMA_API_KEY` برای جست‌وجوی مستقیم `https://ollama.com` |
| [Parallel](/fa/tools/parallel-search)               | گزیده‌های متراکم رتبه‌بندی‌شده برای زمینه LLM                          | --                                               | `PARALLEL_API_KEY` (پولی)                                                               |
| [جست‌وجوی Parallel (رایگان)](/fa/tools/parallel-search) | گزیده‌های متراکم رتبه‌بندی‌شده برای زمینه LLM                          | --                                               | ندارد (Search MCP رایگان)                                                                  |
| [Perplexity](/fa/tools/perplexity-search)           | قطعه‌های متنی ساخت‌یافته                                            | کشور، زبان، زمان، دامنه‌ها، محدودیت‌های محتوا | `PERPLEXITY_API_KEY` / `OPENROUTER_API_KEY`                                             |
| [SearXNG](/fa/tools/searxng-search)                 | قطعه‌های متنی ساخت‌یافته                                            | دسته‌ها، زبان                             | ندارد (خودمیزبان)                                                                      |
| [Tavily](/fa/tools/tavily)                          | قطعه‌های متنی ساخت‌یافته                                            | از طریق ابزار `tavily_search`                         | `TAVILY_API_KEY`                                                                        |

## ساختار نتیجه

`web_search` همه ارائه‌دهندگان Plugin داخلی و خارجی را در مرز ابزار هسته
نرمال‌سازی می‌کند. فراخواننده‌ها دقیقاً یکی از این ساختارهای بسته را دریافت می‌کنند:

```typescript
type WebSearchOutput =
  | {
      kind: "error";
      provider: string;
      error: "provider_error";
      message: string;
      docs?: string;
    }
  | {
      kind: "results";
      provider: string;
      query: string;
      count: number;
      tookMs?: number;
      results: Array<{
        title: string;
        url: string;
        snippet?: string;
        published?: string;
        siteName?: string;
      }>;
      externalContent: {
        untrusted: true;
        source: "web_search";
        wrapped: true;
        provider: string;
      };
      cached?: true;
    }
  | {
      kind: "answer";
      provider: string;
      query: string;
      tookMs?: number;
      content: string;
      citations?: Array<{ url: string; title?: string }>;
      externalContent: {
        untrusted: true;
        source: "web_search";
        wrapped: true;
        provider: string;
      };
      cached?: true;
    }
  | {
      kind: "raw";
      provider: string;
      data: unknown;
    };
```

ارائه‌دهندگان ساخت‌یافته از `kind: "results"` استفاده می‌کنند؛ ارائه‌دهندگان ترکیبی از
`kind: "answer"` استفاده می‌کنند. ارائه‌دهندگان Plugin خارجی که بار داده آن‌ها با هیچ‌یک از این ساختارها
مطابقت ندارد، برای سازگاری بدون تغییر به‌شکل `kind: "raw"` عبور داده می‌شوند. فیلدهای مختص ارائه‌دهنده،
مانند امتیازهای خام، گزیده‌ها، جست‌وجوهای مرتبط، آفست‌های
ارجاع درون‌خطی، شناسه‌های مدل یا فراداده نشست، در شاخه‌های نرمال‌سازی‌شده
عبور داده نمی‌شوند. هنگامی که پاسخ غنی‌تر یک ارائه‌دهنده بخشی از
گردش‌کار شماست، از ابزار اختصاصی آن ارائه‌دهنده استفاده کنید.

`externalContent.wrapped: true` یک نشانگر اعتماد است که خود مرز آن را
درست می‌کند: نثر ارائه‌دهنده (`title`، `snippet`، `siteName`، `content`، عنوان‌های
ارجاع، `message` خطا) از هرگونه خط پوشش ازپیش‌موجود پاک می‌شود و
دقیقاً یک‌بار در مرز هسته دوباره پوشش داده می‌شود، بنابراین هیچ فراداده ارائه‌دهنده‌ای نمی‌تواند
این نشانگر را جعل کند. `query` همیشه همان پرس‌وجوی درخواست‌شده است، URLهای ارجاع و نتیجه
باید به‌صورت http(s) تجزیه‌پذیر باشند، `published` باید ساختار تاریخ ISO داشته باشد، URLها به‌شکل کانونی‌شده منتشر می‌شوند، و
بار داده‌ای که کلید `error` دارد همیشه به‌صورت `kind: "error"` گزارش می‌شود و
کد خام ارائه‌دهنده درون پیام پوشش‌داده‌شده حفظ می‌شود. بارهای داده‌ای که به‌صورت خام عبور داده می‌شوند
هر نشانگری را که ارائه‌دهنده تنظیم کرده است حفظ می‌کنند.

## تشخیص خودکار

فهرست ارائه‌دهندگان در مستندات و جریان‌های راه‌اندازی به‌ترتیب الفبایی است. تشخیص خودکار از یک
ترتیب تقدم ثابت و جداگانه استفاده می‌کند و تنها زمانی ارائه‌دهنده‌ای را انتخاب می‌کند که به
اعتبارنامه (`requiresCredential !== false`) نیاز دارد که آن را پیکربندی‌شده بیابد. اگر
هیچ `provider` تنظیم نشده باشد، OpenClaw ارائه‌دهندگان را به این ترتیب بررسی می‌کند و از
نخستین مورد آماده استفاده می‌کند:

ابتدا ارائه‌دهندگان مبتنی بر API:

1. **Brave** -- `BRAVE_API_KEY` یا `plugins.entries.brave.config.webSearch.apiKey` (ترتیب 10)
2. **MiniMax Search** -- `MINIMAX_CODE_PLAN_KEY` / `MINIMAX_CODING_API_KEY` / `MINIMAX_OAUTH_TOKEN` / `MINIMAX_API_KEY` یا `plugins.entries.minimax.config.webSearch.apiKey` (ترتیب 15)
3. **Gemini** -- `plugins.entries.google.config.webSearch.apiKey`، `GEMINI_API_KEY` یا `models.providers.google.apiKey` (ترتیب 20)
4. **Grok** -- OAuth ‏xAI، ‏`XAI_API_KEY` یا `plugins.entries.xai.config.webSearch.apiKey` (ترتیب 30)
5. **Kimi** -- `KIMI_API_KEY` / `MOONSHOT_API_KEY` یا `plugins.entries.moonshot.config.webSearch.apiKey` (ترتیب 40)
6. **Perplexity** -- `PERPLEXITY_API_KEY` / `OPENROUTER_API_KEY` یا `plugins.entries.perplexity.config.webSearch.apiKey` (ترتیب 50)
7. **Firecrawl** -- `FIRECRAWL_API_KEY` یا `plugins.entries.firecrawl.config.webSearch.apiKey` (ترتیب 60)
8. **Exa** -- `EXA_API_KEY` یا `plugins.entries.exa.config.webSearch.apiKey`؛ مقدار اختیاری `plugins.entries.exa.config.webSearch.baseUrl` نقطه پایانی Exa را بازنویسی می‌کند (ترتیب 65)
9. **Tavily** -- `TAVILY_API_KEY` یا `plugins.entries.tavily.config.webSearch.apiKey` (ترتیب 70)
10. **Parallel** -- ‏Parallel Search API پولی از طریق `PARALLEL_API_KEY` یا `plugins.entries.parallel.config.webSearch.apiKey`؛ مقدار اختیاری `plugins.entries.parallel.config.webSearch.baseUrl` نقطه پایانی را بازنویسی می‌کند (ترتیب 75)

ارائه‌دهندگان نقطه پایانی پیکربندی‌شده پس از آن:

11. **SearXNG** -- `SEARXNG_BASE_URL` یا `plugins.entries.searxng.config.webSearch.baseUrl` (ترتیب 200)

ارائه‌دهندگان بدون کلید مانند **Parallel Search (رایگان)**، ‏**DuckDuckGo**،
**Ollama Web Search** و **Codex Hosted Search** هرگز در تشخیص خودکار انتخاب نمی‌شوند،
حتی با وجود اینکه مقدار ترتیب داخلی دارند. آن‌ها فقط زمانی استفاده می‌شوند که
به‌صراحت با `tools.web.search.provider` یا از طریق
`openclaw configure --section web` انتخابشان کنید. OpenClaw صرفاً به‌دلیل پیکربندی‌نشدن
یک ارائه‌دهنده مبتنی بر API، پرس‌وجوهای مدیریت‌شده
`web_search` را به ارائه‌دهنده‌ای بدون کلید ارسال نمی‌کند.

مدل‌های OpenAI Responses یک استثنا هستند: تا زمانی که `tools.web.search.provider`
تنظیم نشده باشد، به‌جای ارائه‌دهندگان مدیریت‌شده بالا از جست‌وجوی وب بومی OpenAI
استفاده می‌کنند (پایین را ببینید). `tools.web.search.provider` را روی
`parallel-free` (یا ارائه‌دهنده‌ای دیگر) تنظیم کنید تا در عوض از مسیر مدیریت‌شده
هدایت شوند.

<Note>
  همه فیلدهای کلید ارائه‌دهندگان از اشیای SecretRef پشتیبانی می‌کنند. SecretRefهای در محدوده Plugin
  زیر `plugins.entries.<plugin>.config.webSearch.apiKey` برای
  ارائه‌دهندگان نصب‌شده جست‌وجوی وب مبتنی بر API، از جمله Brave، ‏Exa، ‏Firecrawl،
  ‏Gemini، ‏Grok، ‏Kimi، ‏MiniMax، ‏Parallel، ‏Perplexity و Tavily،
  صرف‌نظر از اینکه ارائه‌دهنده به‌صراحت از طریق `tools.web.search.provider` انتخاب شده
  یا با تشخیص خودکار برگزیده شده باشد، برطرف می‌شوند. در حالت تشخیص خودکار، OpenClaw فقط
  کلید ارائه‌دهنده انتخاب‌شده را برطرف می‌کند؛ SecretRefهای انتخاب‌نشده غیرفعال می‌مانند، بنابراین می‌توانید
  چندین ارائه‌دهنده را بدون پرداخت هزینه برطرف‌سازی برای مواردی که استفاده نمی‌کنید،
  پیکربندی‌شده نگه دارید.
</Note>

## جست‌وجوی وب بومی OpenAI

مدل‌های مستقیم OpenAI Responses‏ (`api: "openai-responses"`، ارائه‌دهنده `openai`،
بدون URL پایه یا با URL پایه رسمی OpenAI API) هنگامی که جست‌وجوی وب OpenClaw فعال است و هیچ
ارائه‌دهنده مدیریت‌شده‌ای تثبیت نشده، به‌طور خودکار از ابزار میزبانی‌شده
`web_search` متعلق به OpenAI استفاده می‌کنند. این رفتار متعلق به ارائه‌دهنده در Plugin همراه
OpenAI است و برای URLهای پایه پراکسی سازگار با OpenAI یا مسیرهای Azure
اعمال نمی‌شود. برای حفظ ابزار مدیریت‌شده `web_search` برای مدل‌های OpenAI،
‏`tools.web.search.provider` را روی ارائه‌دهنده دیگری مانند `brave` تنظیم کنید، یا
برای غیرفعال‌کردن جست‌وجوی مدیریت‌شده و جست‌وجوی بومی OpenAI،
‏`tools.web.search.enabled: false` را تنظیم کنید.

## جست‌وجوی وب بومی Codex

زمان‌اجرای app-server در Codex هنگامی که جست‌وجوی وب فعال است و هیچ ارائه‌دهنده مدیریت‌شده‌ای
انتخاب نشده، به‌طور خودکار از ابزار میزبانی‌شده `web_search` متعلق به Codex استفاده می‌کند.
جست‌وجوی میزبانی‌شده بومی و ابزار پویای مدیریت‌شده `web_search` در OpenClaw
به‌طور متقابل انحصاری هستند، بنابراین جست‌وجوی مدیریت‌شده نمی‌تواند محدودیت‌های دامنه بومی را دور بزند.
وقتی جست‌وجوی میزبانی‌شده در دسترس نباشد، به‌صراحت غیرفعال شده باشد یا
با یک ارائه‌دهنده مدیریت‌شده انتخابی جایگزین شده باشد، OpenClaw از ابزار مدیریت‌شده استفاده می‌کند.
OpenClaw افزونه مستقل `web.run` در Codex را غیرفعال نگه می‌دارد
(`features.standalone_web_search: false`)، زیرا ترافیک app-server در محیط عملیاتی فضای نام
`web` تعریف‌شده توسط کاربر آن را رد می‌کند.

- جست‌وجوی بومی را زیر `tools.web.search.openaiCodex` پیکربندی کنید
- `tools.web.search.provider: "codex"` را تنظیم کنید تا Codex Hosted Search به‌عنوان
  ارائه‌دهنده مدیریت‌شده `web_search` برای هر مدل والد فراهم شود. هر فراخوانی یک
  نوبت موقت و محدود app-server در Codex را اجرا می‌کند و اگر Codex یک مورد
  میزبانی‌شده `webSearch` تولید نکند، با شکست مواجه می‌شود.
- `mode: "cached"` ترجیح پیش‌فرض است، اما Codex آن را برای نوبت‌های
  بدون محدودیت app-server به دسترسی خارجی زنده تبدیل می‌کند؛ برای درخواست صریح
  دسترسی زنده، `"live"` را تنظیم کنید
- `tools.web.search.provider` را روی ارائه‌دهنده‌ای مدیریت‌شده مانند `brave`
  تنظیم کنید تا به‌جای آن از `web_search` مدیریت‌شده OpenClaw استفاده شود
- برای انصراف از جست‌وجوی میزبانی‌شده Codex،
  ‏`tools.web.search.openaiCodex.enabled: false` را تنظیم کنید؛ سایر ارائه‌دهندگان مدیریت‌شده همچنان در دسترس می‌مانند
- محدودکردن سطح ابزار بومی Codex نیز `web_search`
  مدیریت‌شده را در دسترس نگه می‌دارد
- وقتی `allowedDomains` تنظیم شده باشد، اگر جست‌وجوی میزبانی‌شده
  در دسترس نباشد، بازگشت خودکار مدیریت‌شده به‌صورت بسته و ناموفق عمل می‌کند تا فهرست مجاز بومی دور زده نشود
- اجراهای فقط-LLM با ابزارهای غیرفعال، جست‌وجوی بومی و مدیریت‌شده را غیرفعال می‌کنند
- `tools.web.search.enabled: false` جست‌وجوی مدیریت‌شده و بومی را غیرفعال می‌کند

تغییرات پایدار در سیاست مؤثر جست‌وجوی Codex یک رشته مقید تازه را آغاز می‌کنند تا
رشته app-server که از قبل بارگذاری شده نتواند دسترسی قدیمی به جست‌وجوی میزبانی‌شده را حفظ کند.
محدودیت‌های موقت هر نوبت از یک رشته محدودشده موقت استفاده می‌کنند و اتصال
موجود را برای ادامه بعدی حفظ می‌کنند.

ترافیک مستقیم OpenAI ChatGPT Responses نیز می‌تواند از ابزار میزبانی‌شده
`web_search` متعلق به OpenAI استفاده کند. آن مسیر جداگانه از طریق
`tools.web.search.openaiCodex.enabled: true` اختیاری باقی می‌ماند و فقط برای مدل‌های واجد شرایط
`openai/*` که از `api: "openai-chatgpt-responses"` استفاده می‌کنند، اعمال می‌شود.

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        // اختیاری: از Codex Hosted Search برای مدل‌های والد غیر Codex نیز استفاده کنید.
        provider: "codex",
        openaiCodex: {
          enabled: true,
          mode: "cached",
          allowedDomains: ["example.com"],
          contextSize: "high",
          userLocation: {
            country: "US",
            city: "New York",
            timezone: "America/New_York",
          },
        },
      },
    },
  },
}
```

برای زمان‌های اجرا و ارائه‌دهندگانی که از جست‌وجوی بومی Codex پشتیبانی نمی‌کنند، Codex می‌تواند
از جایگزین مدیریت‌شده `web_search` از طریق فضای نام ابزار پویای OpenClaw استفاده کند.
هنگامی که به کنترل‌های شبکه ویژه ارائه‌دهنده OpenClaw به‌جای جست‌وجوی میزبانی‌شده Codex
نیاز دارید، از یک ارائه‌دهنده مدیریت‌شده صریح استفاده کنید.

انتخاب `provider: "codex"`، ‏Plugin همراه `codex` را فعال می‌کند و از همان
محدودیت‌های `tools.web.search.openaiCodex` نشان‌داده‌شده در بالا استفاده می‌کند. ابتدا app-server
در Codex را با `openclaw models auth login --provider openai` احراز هویت کنید.
عامل والد می‌تواند از هر مدل یا زمان‌اجرایی استفاده کند؛ فقط کارگر جست‌وجوی محدودشده
از طریق Codex اجرا می‌شود.

## ایمنی شبکه

فراخوانی‌های ارائه‌دهنده HTTP مدیریت‌شده `web_search` از مسیر واکشی محافظت‌شده OpenClaw
استفاده می‌کنند که به نام میزبان خود ارائه‌دهنده فعلی محدود شده است. OpenClaw فقط برای آن نام میزبان،
پاسخ‌های DNS مربوط به IP جعلی Surge، ‏Clash و sing-box را در
`198.18.0.0/15` و `fc00::/7` مجاز می‌داند. سایر مقصدهای خصوصی، loopback، ‏link-local و
فراداده همچنان مسدود می‌مانند. Codex Hosted Search استثنا است:
کارگر محدودشده آن دسترسی شبکه را به ابزار میزبانی‌شده
`web_search` در app-server متعلق به Codex واگذار می‌کند.

این مجوز خودکار برای URLهای دلخواه `web_fetch` اعمال نمی‌شود. برای
`web_fetch`، تنها زمانی که پراکسی مورداعتماد شما مالک آن محدوده‌های ساختگی است،
‏`tools.web.fetch.ssrfPolicy.allowRfc2544BenchmarkRange` و `tools.web.fetch.ssrfPolicy.allowIpv6UniqueLocalRange` را به‌صراحت فعال کنید.

## پیکربندی

```json5
{
  tools: {
    web: {
      search: {
        enabled: true, // پیش‌فرض: true
        provider: "brave", // یا برای تشخیص خودکار حذف کنید
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
    },
  },
}
```

پیکربندی ویژه ارائه‌دهنده (کلیدهای API، ‏URLهای پایه، حالت‌ها) زیر
`plugins.entries.<plugin>.config.webSearch.*` قرار دارد. Gemini همچنین می‌تواند پس از پیکربندی اختصاصی
جست‌وجوی وب خود و `GEMINI_API_KEY`، از `models.providers.google.apiKey` و
`models.providers.google.baseUrl` به‌عنوان جایگزین‌هایی با اولویت پایین‌تر استفاده کند. برای نمونه‌ها
به صفحات ارائه‌دهندگان مراجعه کنید.
Grok همچنین می‌تواند از نمایه احراز هویت OAuth ‏xAI در `openclaw models auth login
--provider xai --method oauth` دوباره استفاده کند؛ پیکربندی کلید API همچنان گزینه جایگزین است.

`tools.web.search.provider` در برابر شناسه‌های ارائه‌دهنده جست‌وجوی وب
اعلام‌شده در مانیفست Pluginهای همراه و نصب‌شده اعتبارسنجی می‌شود. یک اشتباه تایپی مانند
`"brvae"` به‌جای بازگشت بی‌صدا به تشخیص خودکار، اعتبارسنجی پیکربندی را
ناموفق می‌کند. اگر یک ارائه‌دهنده پیکربندی‌شده فقط شواهد قدیمی Plugin داشته باشد، مانند
بلوک باقی‌مانده `plugins.entries.<plugin>` پس از حذف نصب یک Plugin شخص ثالث،
OpenClaw راه‌اندازی را تاب‌آور نگه می‌دارد و هشداری گزارش می‌کند تا بتوانید
Plugin را دوباره نصب کنید یا برای پاک‌سازی پیکربندی قدیمی، `openclaw doctor --fix` را اجرا کنید.

انتخاب ارائه‌دهنده جایگزین `web_fetch` جداگانه است:

- آن را با `tools.web.fetch.provider` انتخاب کنید
- یا آن فیلد را حذف کنید و اجازه دهید OpenClaw نخستین ارائه‌دهنده آماده واکشی وب
  را از اعتبارنامه‌های پیکربندی‌شده به‌طور خودکار تشخیص دهد
- `web_fetch` بدون sandbox می‌تواند از ارائه‌دهندگان Plugin نصب‌شده‌ای استفاده کند که
  `contracts.webFetchProviders` را اعلام می‌کنند؛ واکشی‌های sandboxشده ارائه‌دهندگان همراه و
  نصب‌های تأییدشده Pluginهای رسمی را مجاز می‌دانند، اما Pluginهای خارجی شخص ثالث را کنار می‌گذارند
- ‏Plugin رسمی Firecrawl تنها مشارکت‌کننده همراه `webFetchProviders`
  در حال حاضر است که زیر
  `plugins.entries.firecrawl.config.webFetch.*` پیکربندی می‌شود

وقتی در جریان `openclaw onboard` یا
`openclaw configure --section web`، ‏**Kimi** را انتخاب می‌کنید، OpenClaw می‌تواند این موارد را نیز درخواست کند:

- منطقه Moonshot API‏ (`https://api.moonshot.ai/v1` یا `https://api.moonshot.cn/v1`)
- مدل پیش‌فرض جست‌وجوی وب Kimi (پیش‌فرض `kimi-k2.6`)

برای `x_search`، ‏`plugins.entries.xai.config.xSearch.*` را پیکربندی کنید. این مورد از
همان نمایه احراز هویت xAI مورد استفاده در گفت‌وگو یا اعتبارنامه
`XAI_API_KEY` / جست‌وجوی وب Plugin مورد استفاده جست‌وجوی وب Grok استفاده می‌کند.
پیکربندی قدیمی `tools.web.x_search.*` به‌طور خودکار توسط `openclaw doctor --fix` مهاجرت داده می‌شود.
وقتی در جریان `openclaw onboard` یا `openclaw configure --section web`، ‏Grok را انتخاب می‌کنید،
OpenClaw درست پس از تکمیل راه‌اندازی Grok، راه‌اندازی اختیاری `x_search` را نیز با همان
اعتبارنامه ارائه می‌دهد. این یک مرحله پیگیری جداگانه درون مسیر Grok است،
نه یک انتخاب جداگانه ارائه‌دهنده جست‌وجوی وب در سطح بالا. اگر ارائه‌دهنده دیگری را انتخاب کنید،
OpenClaw اعلان `x_search` را نمایش نمی‌دهد.

### ذخیره‌سازی کلیدهای API

<Tabs>
  <Tab title="فایل پیکربندی">
    ‏`openclaw configure --section web` را اجرا کنید یا کلید را مستقیماً تنظیم کنید:

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "YOUR_KEY", // pragma: allowlist secret
              },
            },
          },
        },
      },
    }
    ```

  </Tab>
  <Tab title="متغیر محیطی">
    متغیر محیطی ارائه‌دهنده را در محیط فرایند Gateway تنظیم کنید:

    ```bash
    export BRAVE_API_KEY="YOUR_KEY"
    ```

    برای نصب Gateway، آن را در `~/.openclaw/.env` قرار دهید.
    [متغیرهای محیطی](/fa/help/faq#env-vars-and-env-loading) را ببینید.

  </Tab>
</Tabs>

## پارامترهای ابزار

| پارامتر             | توضیحات                                                        |
| --------------------- | ------------------------------------------------------------------ |
| `query`               | عبارت جست‌وجو (الزامی)                                            |
| `count`               | تعداد نتایج بازگشتی (1-10، پیش‌فرض: 5)                               |
| `country`             | کد دوحرفی کشور ISO (برای مثال، "US"، "DE")                        |
| `language`            | کد زبان ISO 639-1 (برای مثال، "en"، "de")                          |
| `search_lang`         | کد زبان جست‌وجو (فقط Brave)                                  |
| `freshness`           | فیلتر زمانی: `day`، `week`، `month`، یا `year`                     |
| `date_after`          | نتایج پس از این تاریخ (YYYY-MM-DD)                               |
| `date_before`         | نتایج پیش از این تاریخ (YYYY-MM-DD)                              |
| `ui_lang`             | کد زبان رابط کاربری (فقط Brave)                                      |
| `domain_filter`       | آرایهٔ فهرست مجاز/مسدود دامنه‌ها (فقط Perplexity)                  |
| `max_tokens`          | بودجهٔ کل توکن محتوا، فقط API بومی Perplexity Search      |
| `max_tokens_per_page` | محدودیت توکن استخراج در هر صفحه، فقط API بومی Perplexity Search |

<Warning>
  همهٔ پارامترها با تمام ارائه‌دهندگان کار نمی‌کنند. حالت `llm-context` در Brave
  پارامتر `ui_lang` را رد می‌کند؛ `date_before` نیز به `date_after` نیاز دارد، زیرا بازه‌های سفارشی
  تازگی در Brave به هر دو تاریخ شروع و پایان نیاز دارند.
  Gemini، Grok و Kimi یک پاسخ ترکیبی همراه با ارجاعات برمی‌گردانند. آن‌ها
  برای سازگاری با ابزار مشترک، `count` را می‌پذیرند، اما این پارامتر شکل
  پاسخ مبتنی بر منابع را تغییر نمی‌دهد. Gemini تازگی `day` را به‌عنوان راهنمای جدیدبودن در نظر می‌گیرد؛ مقادیر
  گسترده‌تر تازگی و تاریخ‌های صریح، بازه‌های زمانی اتکای Google Search به منابع را تنظیم می‌کنند.
  Perplexity هنگام استفاده از مسیر سازگاری Sonar/OpenRouter
  (`plugins.entries.perplexity.config.webSearch.baseUrl` /
  `model` یا `OPENROUTER_API_KEY`) به همین شیوه عمل می‌کند؛ آن مسیر همچنین پشتیبانی از `max_tokens` و
  `max_tokens_per_page` را حذف می‌کند.
  SearXNG فقط برای میزبان‌های قابل‌اعتماد شبکهٔ خصوصی یا loopback، `http://` را می‌پذیرد؛
  نقاط پایانی عمومی SearXNG باید از `https://` استفاده کنند.
  Firecrawl و Tavily فقط از `query` و `count` از طریق `web_search`
  پشتیبانی می‌کنند -- برای گزینه‌های پیشرفته از ابزارهای اختصاصی آن‌ها استفاده کنید.
</Warning>

## x_search

`x_search` با استفاده از xAI پست‌های X (نام پیشین: Twitter) را جست‌وجو می‌کند و
پاسخ‌های ترکیب‌شده با هوش مصنوعی را همراه با ارجاعات برمی‌گرداند. این ابزار عبارت‌های زبان طبیعی و
فیلترهای ساختاریافتهٔ اختیاری را می‌پذیرد. OpenClaw ابزار داخلی `x_search` متعلق به xAI را
برای هر درخواست می‌سازد و آن را به‌طور دائمی ثبت‌شده نگه نمی‌دارد؛ بنابراین ابزار فقط
در نوبتی فعال است که واقعاً آن را فراخوانی می‌کند.

<Warning>
  `x_search` روی سرورهای xAI اجرا می‌شود. xAI به‌ازای هر 1,000 فراخوانی ابزار، $5 به‌علاوهٔ
  توکن‌های ورودی و خروجی مدل هزینه دریافت می‌کند.
</Warning>

<Note>
  طبق مستندات xAI، `x_search` از جست‌وجوی کلیدواژه‌ای، جست‌وجوی معنایی، جست‌وجوی کاربر
  و دریافت رشته‌گفت‌وگو پشتیبانی می‌کند. برای آمار تعامل هر پست، مانند بازنشرها،
  پاسخ‌ها، نشانک‌ها یا بازدیدها، جست‌وجوی هدفمند URL دقیق پست
  یا شناسهٔ وضعیت را ترجیح دهید. جست‌وجوهای گستردهٔ کلیدواژه‌ای ممکن است پست درست را پیدا کنند، اما
  فرادادهٔ ناقص‌تری برای هر پست برگردانند. الگوی مناسب این است: ابتدا پست را پیدا کنید، سپس
  یک عبارت `x_search` دوم را با تمرکز بر همان پست دقیق اجرا کنید.
</Note>

### پیکربندی x_search

در صورت حذف `enabled`، تنها زمانی `x_search` در دسترس قرار می‌گیرد که ارائه‌دهندهٔ مدل فعال
`xai` باشد و اعتبارنامه‌های xAI قابل‌دستیابی باشند. برای مدل فعالی با ارائه‌دهندهٔ شناخته‌شدهٔ
غیر xAI، جهت موافقت با استفادهٔ بین‌ارائه‌دهنده‌ای، `plugins.entries.xai.config.xSearch.enabled` را روی `true`
تنظیم کنید. اگر ارائه‌دهندهٔ مدل فعال مشخص نباشد یا
قابل‌شناسایی نباشد، ابزار پنهان می‌ماند. برای غیرفعال‌کردن آن نزد
همهٔ ارائه‌دهندگان، `enabled` را روی `false` تنظیم کنید. اعتبارنامه‌های xAI همیشه الزامی هستند.

```json5
{
  plugins: {
    entries: {
      xai: {
        config: {
          xSearch: {
            enabled: true, // برای یک ارائه‌دهندهٔ مدل شناخته‌شدهٔ غیر xAI الزامی است
            model: "grok-4.3",
            baseUrl: "https://api.x.ai/v1", // اختیاری، webSearch.baseUrl را بازنویسی می‌کند
            inlineCitations: false,
            maxTurns: 2,
            timeoutSeconds: 30,
            cacheTtlMinutes: 15,
          },
          webSearch: {
            apiKey: "xai-...", // در صورت تنظیم پروفایل احراز هویت xAI یا XAI_API_KEY اختیاری است
            baseUrl: "https://api.x.ai/v1", // نشانی پایهٔ مشترک و اختیاری xAI Responses
          },
        },
      },
    },
  },
}
```

`x_search` در صورت تنظیم
`plugins.entries.xai.config.xSearch.baseUrl` به `<baseUrl>/responses` پست می‌کند. اگر آن فیلد حذف شود،
ابتدا به `plugins.entries.xai.config.webSearch.baseUrl` و سپس به
نقطهٔ پایانی عمومی xAI (`https://api.x.ai/v1`) برمی‌گردد.

### پارامترهای x_search

| پارامتر                    | توضیحات                                            |
| ---------------------------- | ------------------------------------------------------ |
| `query`                      | عبارت جست‌وجو (الزامی)                                |
| `allowed_x_handles`          | محدودکردن نتایج به حداکثر 20 نام کاربری X               |
| `excluded_x_handles`         | مستثناکردن حداکثر 20 نام کاربری X                           |
| `from_date`                  | فقط پست‌های این تاریخ یا پس از آن را شامل شود (YYYY-MM-DD)  |
| `to_date`                    | فقط پست‌های این تاریخ یا پیش از آن را شامل شود (YYYY-MM-DD) |
| `enable_image_understanding` | اجازه به xAI برای بررسی تصاویر پیوست‌شده به پست‌های منطبق      |
| `enable_video_understanding` | اجازه به xAI برای بررسی ویدئوهای پیوست‌شده به پست‌های منطبق      |

`allowed_x_handles` و `excluded_x_handles` ناسازگار با یکدیگرند.

### نمونهٔ x_search

```javascript
await x_search({
  query: "دستورهای تهیهٔ شام",
  allowed_x_handles: ["nytfood"],
  from_date: "2026-03-01",
});
```

```javascript
// آمار هر پست: در صورت امکان از URL دقیق وضعیت یا شناسهٔ وضعیت استفاده کنید
await x_search({
  query: "https://x.com/huntharo/status/1905678901234567890",
});
```

## نمونه‌ها

```javascript
// جست‌وجوی پایه
await web_search({ query: "OpenClaw plugin SDK" });

// جست‌وجوی مختص آلمانی
await web_search({ query: "تماشای آنلاین تلویزیون", country: "DE", language: "de" });

// نتایج اخیر (هفتهٔ گذشته)
await web_search({ query: "پیشرفت‌های هوش مصنوعی", freshness: "week" });

// بازهٔ تاریخی
await web_search({
  query: "پژوهش اقلیمی",
  date_after: "2024-01-01",
  date_before: "2024-06-30",
});

// فیلتر دامنه (فقط Perplexity)
await web_search({
  query: "نقدهای محصول",
  domain_filter: ["-reddit.com", "-pinterest.com"],
});
```

## پروفایل‌های ابزار

اگر از پروفایل‌های ابزار یا فهرست‌های مجاز استفاده می‌کنید، `web_search`، `x_search` یا `group:web` را اضافه کنید:

```json5
{
  tools: {
    allow: ["web_search", "x_search"],
    // یا: allow: ["group:web"]  (شامل web_search، x_search و web_fetch)
  },
}
```

## مرتبط

- [واکشی وب](/fa/tools/web-fetch) -- واکشی یک URL و استخراج محتوای خوانا
- [مرورگر وب](/fa/tools/browser) -- خودکارسازی کامل مرورگر برای سایت‌های متکی بر JS
- [جست‌وجوی Grok](/fa/tools/grok-search) -- استفاده از Grok به‌عنوان ارائه‌دهندهٔ `web_search`
- [جست‌وجوی وب Ollama](/fa/tools/ollama-search) -- جست‌وجوی وب بدون کلید از طریق میزبان Ollama شما
