---
read_when:
    - می‌خواهید با حفظ کش، هزینهٔ توکن‌های پرامپت را کاهش دهید
    - در پیکربندی‌های چندعاملی، به رفتار کش به‌ازای هر عامل نیاز دارید
    - شما در حال تنظیم هم‌زمان Heartbeat و پاک‌سازی بر اساس cache-ttl هستید
summary: تنظیمات کش‌کردن پرامپت، ترتیب ادغام، رفتار ارائه‌دهنده و الگوهای تنظیم‌سازی
title: کش‌کردن پرامپت
x-i18n:
    generated_at: "2026-07-27T17:05:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 99dfd3d226d37014110adf16818051236114dcb0277e9b4d13eaced0f1fc03aa
    source_path: reference/prompt-caching.md
    workflow: 16
---

کش‌کردن پرامپت به ارائه‌دهنده مدل امکان می‌دهد پیشوند بدون‌تغییر پرامپت (دستورالعمل‌های system/developer، تعریف ابزارها و سایر زمینه‌های پایدار) را در نوبت‌های مختلف دوباره استفاده کند، به‌جای آنکه در هر درخواست دوباره آن را پردازش کند. این کار هزینه توکن و تأخیر را در نشست‌های طولانی‌مدت با زمینه تکراری کاهش می‌دهد.

OpenClaw در هر جایی که API بالادستی این شمارنده‌ها را ارائه دهد، میزان استفاده ارائه‌دهنده را به `cacheRead` و `cacheWrite` نرمال‌سازی می‌کند. خلاصه‌های استفاده (`/status` و موارد مشابه) هنگامی که نمای لحظه‌ای نشست زنده فاقد شمارنده‌های کش باشد، از آخرین ورودی استفاده در رونوشت استفاده می‌کنند؛ مقدار زنده غیرصفر همیشه بر مقدار جایگزین اولویت دارد.

مراجع ارائه‌دهندگان:

- [کش‌کردن پرامپت Anthropic](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- [کش‌کردن پرامپت OpenAI](https://developers.openai.com/api/docs/guides/prompt-caching)

## تنظیمات اصلی

### `cacheRetention`

مقادیر: `"none" | "short" | "long"`. به‌عنوان پیش‌فرض سراسری، برای هر مدل و برای هر عامل قابل پیکربندی است.
`"standard"` نام مستعار نیست؛ برای پنجره کش پیش‌فرض ارائه‌دهنده از `"short"` استفاده کنید. مقادیر نامعتبر با یک هشدار نادیده گرفته می‌شوند.

```yaml
agents:
  defaults:
    params:
      cacheRetention: "long" # none | short | long
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "short" # پیش‌فرض سراسری را برای این مدل لغو می‌کند
  list:
    - id: "alerts"
      params:
        cacheRetention: "none" # هر دو پیش‌فرض را برای این عامل لغو می‌کند
```

ترتیب ادغام (مورد بعدی اولویت دارد):

1. `agents.defaults.params` - پیش‌فرض سراسری برای همه مدل‌ها
2. `agents.defaults.models["provider/model"].params` - لغو برای هر مدل
3. `agents.entries.*.params` - لغو برای هر عامل، با تطبیق شناسه عامل

منبع: `src/agents/embedded-agent-runner/extra-params.ts` (`resolveExtraParams`).

### `contextPruning.mode: "cache-ttl"`

پس از سپری‌شدن پنجره TTL کش، زمینه قدیمی نتایج ابزار را هرس می‌کند تا درخواست پس از دوره بیکاری، تاریخچه بیش‌ازحد بزرگ را دوباره کش نکند.

```yaml
agents:
  defaults:
    contextPruning:
      mode: "cache-ttl"
      ttl: "1h"
```

برای رفتار کامل، [هرس نشست](/fa/concepts/session-pruning) را ببینید.

### گرم نگه‌داشتن با Heartbeat

Heartbeat می‌تواند پنجره‌های کش را گرم نگه دارد و نوشتن مکرر کش پس از وقفه‌های بیکاری را کاهش دهد. به‌صورت سراسری (`agents.defaults.heartbeat`) یا برای هر عامل (`agents.entries.*.heartbeat`) قابل پیکربندی است.

```yaml
agents:
  defaults:
    heartbeat:
      every: "55m"
```

## رفتار ارائه‌دهندگان

### Anthropic (API مستقیم و Vertex AI)

- `cacheRetention` برای ارائه‌دهندگان `anthropic` و `anthropic-vertex` و نیز برای مدل‌های Claude در `amazon-bedrock` و نقاط پایانی سفارشی سازگار با `anthropic-messages`، در صورت تنظیم صریح `cacheRetention`، پشتیبانی می‌شود.
- در صورت تنظیم‌نشدن، OpenClaw مقدار `cacheRetention: "short"` را برای Anthropic مستقیم مقداردهی اولیه می‌کند (فقط ارائه‌دهندگان `anthropic` و `anthropic-vertex`؛ سایر مسیرهای خانواده Anthropic به مقدار صریح نیاز دارند).
- پاسخ‌های بومی Anthropic Messages، مقادیر `cache_read_input_tokens` و `cache_creation_input_tokens` را ارائه می‌کنند که به `cacheRead` و `cacheWrite` نگاشت می‌شوند.
- `cacheRetention: "short"` به کش موقت 5 دقیقه‌ای پیش‌فرض نگاشت می‌شود. در صورت تنظیم صریح، `cacheRetention: "long"` مقدار TTL یک‌ساعته (`cache_control: { type: "ephemeral", ttl: "1h" }`) را درخواست می‌کند. نگهداشت طولانی ضمنی/مبتنی بر متغیر محیطی (`OPENCLAW_CACHE_RETENTION=long` بدون `cacheRetention` صریح) فقط در میزبان‌های `api.anthropic.com` یا Vertex AI (`aiplatform.googleapis.com` / `*-aiplatform.googleapis.com`) به TTL یک‌ساعته ارتقا می‌یابد؛ سایر میزبان‌ها کش 5 دقیقه‌ای را حفظ می‌کنند.

منبع: `packages/ai/src/transports/anthropic-payload-policy.ts` (`resolveAnthropicEphemeralCacheControl`، `isLongTtlEligibleEndpoint`).

### OpenAI (API مستقیم)

- کش‌کردن پرامپت در مدل‌های جدید پشتیبانی‌شده خودکار است؛ OpenClaw نشانگرهای کش در سطح بلوک تزریق نمی‌کند.
- OpenClaw مقدار `prompt_cache_key` را ارسال می‌کند تا مسیریابی کش در نوبت‌های مختلف پایدار بماند. میزبان‌های مستقیم `api.openai.com` این مقدار را به‌صورت خودکار دریافت می‌کنند. پراکسی‌های سازگار با OpenAI (oMLX، llama.cpp و نقاط پایانی سفارشی) برای فعال‌سازی به `compat.supportsPromptCacheKey: true` در پیکربندی مدل نیاز دارند؛ این مورد هرگز برای پراکسی به‌صورت خودکار تشخیص داده نمی‌شود.
- `prompt_cache_retention: "24h"` فقط زمانی افزوده می‌شود که `cacheRetention: "long"` انتخاب شده باشد و نقطه پایانی حل‌شده هم از کلید کش و هم نگهداشت طولانی پشتیبانی کند (`compat.supportsLongCacheRetention`، به‌طور پیش‌فرض true؛ پروفایل‌های سازگاری Together AI و Cloudflare آن را غیرفعال می‌کنند). `cacheRetention: "none"` هر دو فیلد را حذف می‌کند.
- موفقیت‌های کش از طریق `usage.prompt_tokens_details.cached_tokens` (Chat Completions) یا `input_tokens_details.cached_tokens` (Responses API) نمایان می‌شوند و به `cacheRead` نگاشت می‌شوند.
- بارهای Responses API همچنین می‌توانند `input_tokens_details.cache_write_tokens` را ارائه کنند که به `cacheWrite` نگاشت می‌شود و با نرخ نوشتن کش مدل قیمت‌گذاری می‌شود؛ بارهای Responses که این فیلد را ندارند، `cacheWrite` را در `0` نگه می‌دارند. API ‏Chat Completions متعلق به OpenAI شمارنده `cache_write_tokens` را مستند یا منتشر نمی‌کند، اما OpenClaw همچنان `prompt_tokens_details.cache_write_tokens` را در آنجا برای پراکسی‌های سازگار با OpenRouter و پراکسی‌های سبک DeepSeek که شمارش نوشتن جداگانه گزارش می‌کنند، می‌خواند.
- در عمل، رفتار OpenAI بیشتر شبیه کش پیشوند اولیه است تا استفاده مجدد Anthropic از کل تاریخچه متحرک؛ [انتظارات زنده OpenAI](#openai-live-expectations) را در ادامه ببینید.

### Amazon Bedrock

- مراجع مدل Anthropic Claude (`amazon-bedrock/*anthropic.claude*`، به‌همراه پیشوندهای پروفایل استنتاج سیستمی AWS یعنی `us.`/`eu.`/`global.anthropic.claude*`) از عبور صریح `cacheRetention` پشتیبانی می‌کنند.
- مدل‌های غیر Anthropic در Bedrock (برای مثال `amazon.nova-*`) در زمان اجرا، صرف‌نظر از هر مقدار پیکربندی‌شده `cacheRetention`، بدون نگهداشت کش حل می‌شوند.
- ARNهای مبهم پروفایل استنتاج برنامه Bedrock (شناسه‌های پروفایلی که شامل `claude` نیستند) نیز، مگر اینکه `cacheRetention` صریحاً تنظیم شده باشد، بدون نگهداشت کش حل می‌شوند؛ زیرا خانواده مدل را نمی‌توان تنها از ARN استنباط کرد.

### OpenRouter

برای مراجع مدل `openrouter/anthropic/*`، OpenClaw نشانگرهای `cache_control` مربوط به Anthropic را در بلوک‌های پرامپت system/developer تزریق می‌کند، اما فقط زمانی که درخواست همچنان یک مسیر تأییدشده OpenRouter را هدف قرار دهد (`openrouter` در نقطه پایانی پیش‌فرض آن، یا هر ارائه‌دهنده/URL پایه‌ای که به `openrouter.ai` حل شود). تغییر مقصد مدل به یک URL پراکسی دلخواه سازگار با OpenAI این تزریق را متوقف می‌کند.

`contextPruning.mode: "cache-ttl"` برای مراجع مدل `openrouter/anthropic/*`، `openrouter/deepseek/*`، `openrouter/moonshot/*`، `openrouter/moonshotai/*` و `openrouter/zai/*` مجاز است، زیرا این مسیرها کش‌کردن پرامپت در سمت ارائه‌دهنده را بدون نیاز به نشانگرهای تزریق‌شده OpenClaw انجام می‌دهند.

منبع: `extensions/openrouter/index.ts` (`OPENROUTER_CACHE_TTL_MODEL_PREFIXES`).

ساخت کش DeepSeek در OpenRouter به‌صورت بهترین تلاش انجام می‌شود و ممکن است چند ثانیه طول بکشد؛ یک درخواست پیگیری فوری ممکن است همچنان `cached_tokens: 0` را نشان دهد. پس از یک تأخیر کوتاه، با یک درخواست تکراری دارای همان پیشوند و با استفاده از `usage.prompt_tokens_details.cached_tokens` به‌عنوان نشانه موفقیت کش، آن را تأیید کنید.

### Google Gemini (API مستقیم)

- انتقال مستقیم Gemini (`api: "google-generative-ai"`) موفقیت‌های کش را از طریق `cachedContentTokenCount` بالادستی گزارش می‌کند که به `cacheRead` نگاشت می‌شود.
- خانواده‌های مدل واجد شرایط: `gemini-2.5*` و `gemini-3*` (گونه‌های Live/preview خارج از تطبیق این پیشوند، برای مثال `gemini-live-2.5-flash-preview`، مستثنا هستند).
- وقتی `cacheRetention` روی یک مدل واجد شرایط تنظیم شود، OpenClaw به‌صورت خودکار یک منبع `cachedContents` برای پرامپت system ایجاد، دوباره استفاده و تازه‌سازی می‌کند؛ نیازی به دستگیره دستی محتوای کش‌شده نیست. TTL برای `cacheRetention: "short"` برابر `300s` و برای `"long"` برابر `3600s` است.
- همچنان می‌توانید یک دستگیره ازپیش‌موجود محتوای کش‌شده Gemini را به‌صورت `params.cachedContent` (یا `params.cached_content` قدیمی) عبور دهید؛ دستگیره صریح، مسیر مدیریت خودکار کش را کاملاً نادیده می‌گیرد.
- این مورد از کش پیشوند پرامپت Anthropic/OpenAI جدا است: OpenClaw به‌جای تزریق نشانگرهای درون‌خطی کش، یک منبع بومی ارائه‌دهنده `cachedContents` را برای Gemini مدیریت می‌کند.

منبع: `src/agents/embedded-agent-runner/google-prompt-cache.ts`.

### ارائه‌دهندگان مبتنی بر مهار CLI (Claude Code، Gemini CLI)

بک‌اندهای CLI که رویدادهای استفاده JSONL (`jsonlDialect: "claude-stream-json"` یا `"gemini-stream-json"`) منتشر می‌کنند، از یک تجزیه‌گر مشترک استفاده عبور می‌کنند که چندین گونه نام فیلد، از جمله شمارنده ساده `cached` نگاشت‌شده به `cacheRead`، را تشخیص می‌دهد. وقتی بار JSON متعلق به CLI فیلد مستقیم توکن ورودی را نداشته باشد، OpenClaw آن را به‌صورت `input_tokens - cached` محاسبه می‌کند. این فقط نرمال‌سازی استفاده است و برای این مدل‌های هدایت‌شده با CLI نشانگرهای کش پرامپت به سبک Anthropic/OpenAI ایجاد نمی‌کند.

منبع: `src/agents/cli-output.ts` (`toCliUsage`).

### سایر ارائه‌دهندگان

اگر ارائه‌دهنده‌ای از هیچ‌یک از حالت‌های کش بالا پشتیبانی نکند، `cacheRetention` اثری ندارد.

## مرز کش پرامپت system

OpenClaw پرامپت system را در یک مرز داخلی پیشوند کش به یک **پیشوند پایدار** و یک **پسوند متغیر** تقسیم می‌کند. محتوای بالای مرز (تعریف ابزارها، فراداده Skills و فایل‌های فضای کاری) طوری مرتب می‌شود که در نوبت‌های مختلف از نظر بایتی یکسان بماند. محتوای پایین مرز (برای مثال `HEARTBEAT.md`، مُهرهای زمانی زمان اجرا و سایر فراداده‌های مختص هر نوبت) می‌تواند بدون نامعتبرکردن پیشوند کش‌شده تغییر کند.

گزینه‌های کلیدی طراحی:

- فایل‌های پایدار زمینه پروژه در فضای کاری پیش از `HEARTBEAT.md` مرتب می‌شوند تا تغییرات Heartbeat پیشوند پایدار را باطل نکند.
- این مرز در شکل‌دهی انتقال خانواده Anthropic، خانواده OpenAI، Google و CLI اعمال می‌شود تا همه ارائه‌دهندگان پشتیبانی‌شده از همان پایداری پیشوند بهره‌مند شوند.
- درخواست‌های Codex Responses و Anthropic Vertex از طریق شکل‌دهی کش آگاه از مرز مسیریابی می‌شوند تا استفاده مجدد از کش با آنچه ارائه‌دهندگان واقعاً دریافت می‌کنند هم‌راستا بماند.
- اثر انگشت پرامپت‌های system نرمال‌سازی می‌شود (فاصله‌ها، پایان خطوط، زمینه افزوده‌شده با هوک و ترتیب قابلیت‌های زمان اجرا) تا پرامپت‌هایی که از نظر معنایی تغییری نکرده‌اند، در نوبت‌های مختلف کش مشترک داشته باشند.

اگر پس از تغییر پیکربندی یا فضای کاری جهش‌های غیرمنتظره `cacheWrite` مشاهده کردید، بررسی کنید که تغییر در بالا یا پایین مرز کش قرار می‌گیرد. انتقال محتوای متغیر به پایین مرز (یا پایدارسازی آن) معمولاً مشکل را برطرف می‌کند.

## محافظ‌های پایداری کش OpenClaw

- کاتالوگ‌های ابزار MCP همراه، پیش از ثبت ابزار به‌صورت قطعی مرتب می‌شوند (ابتدا بر اساس نام سرور، سپس نام ابزار) تا تغییر ترتیب `listTools()` موجب تغییر مداوم بلوک ابزارها و باطل‌شدن پیشوندهای کش پرامپت نشود.
- نشست‌های قدیمی دارای بلوک‌های تصویر ماندگار، **3 نوبت تکمیل‌شده اخیر** را دست‌نخورده نگه می‌دارند (با شمارش همه نوبت‌های تکمیل‌شده، نه فقط نوبت‌های دارای تصویر). بلوک‌های تصویر قدیمی‌تر که قبلاً پردازش شده‌اند با یک نشانگر متنی جایگزین می‌شوند تا پیگیری‌های سنگین از نظر تصویر، بارهای قدیمی و بزرگ را مکرراً ارسال نکنند.

## الگوهای تنظیم

### ترافیک ترکیبی (پیش‌فرض توصیه‌شده)

یک خط پایه بلندمدت را برای عامل اصلی خود حفظ کنید و کش را برای عامل‌های اعلان‌دهنده با فعالیت جهشی غیرفعال کنید:

```yaml
agents:
  defaults:
    model:
      primary: "anthropic/claude-opus-4-6"
    models:
      "anthropic/claude-opus-4-6":
        params:
          cacheRetention: "long"
  list:
    - id: "research"
      default: true
      heartbeat:
        every: "55m"
    - id: "alerts"
      params:
        cacheRetention: "none"
```

### خط پایه با اولویت هزینه

- مقدار پایه `cacheRetention: "short"` را تنظیم کنید.
- `contextPruning.mode: "cache-ttl"` را فعال کنید.
- فقط برای عامل‌هایی که از کش گرم بهره می‌برند، Heartbeat را پایین‌تر از TTL نگه دارید.

## آزمون‌های زنده رگرسیون

OpenClaw یک دروازه ترکیبی رگرسیون زنده کش را اجرا می‌کند که پیشوندهای تکراری، نوبت‌های ابزار، نوبت‌های تصویر، رونوشت‌های ابزار به سبک MCP و یک کنترل بدون کش Anthropic را پوشش می‌دهد.

- `src/agents/live-cache-regression.live.test.ts`
- `src/agents/live-cache-regression-runner.ts`
- `src/agents/live-cache-regression-baseline.ts`

آن را با دستور زیر اجرا کنید:

```sh
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_CACHE_TEST=1 pnpm test:live:cache
```

فایل مبنا جدیدترین اعداد مشاهده‌شده در محیط زنده را همراه با کف‌های رگرسیون ویژه هر ارائه‌دهنده که آزمون آن‌ها را بررسی می‌کند، ذخیره می‌کند. هر اجرا از شناسه‌های نشست و فضاهای نام پرامپت تازه و مختص همان اجرا استفاده می‌کند تا وضعیت کش قبلی نمونه فعلی را آلوده نکند. Anthropic و OpenAI سازوکارهای اعمال متفاوتی دارند: نرسیدن Anthropic به کف یک رگرسیون قطعی است (آزمون ناموفق می‌شود)، درحالی‌که نرسیدن OpenAI به کف صرفاً تحت نظارت است (به‌صورت هشدار ثبت می‌شود و اجرا را ناموفق نمی‌کند). آن‌ها آستانه واحدی را میان ارائه‌دهندگان به اشتراک نمی‌گذارند.

### انتظارات محیط زنده Anthropic

- انتظار نوشتن‌های صریح گرم‌سازی از طریق `cacheWrite` را داشته باشید.
- در نوبت‌های تکراری انتظار استفاده مجدد از تقریباً تمام تاریخچه را داشته باشید، زیرا کنترل کش Anthropic نقطه شکست کش را در طول مکالمه جلو می‌برد.
- کف‌های مبنا برای مسیرهای پایدار، ابزار، تصویر و سبک MCP دروازه‌های قطعی رگرسیون هستند.

### انتظارات محیط زنده OpenAI

- فقط انتظار `cacheRead` را داشته باشید؛ `cacheWrite` در Chat Completions به‌صورت `0` باقی می‌ماند.
- استفاده مجدد از کش در نوبت‌های تکراری را یک سطح ثابت ویژه ارائه‌دهنده در نظر بگیرید، نه استفاده مجدد متحرک از تمام تاریخچه به سبک Anthropic.
- کف‌ها صرفاً تحت نظارت هستند (نرسیدن به کف به‌صورت هشدار ثبت می‌شود، نه شکست آزمون) و از رفتار مشاهده‌شده در محیط زنده روی `gpt-5.4-mini` استخراج شده‌اند:

| سناریو               | کف `cacheRead` | کف نرخ اصابت |
| -------------------- | ----------------: | -------------: |
| پیشوند پایدار        |             4,608 |           0.90 |
| رونوشت ابزار         |             4,096 |           0.85 |
| رونوشت تصویر         |             3,840 |           0.82 |
| رونوشت سبک MCP       |             4,096 |           0.85 |

جدیدترین اعداد مبنای مشاهده‌شده (از `live-cache-regression-baseline.ts`) به این مقادیر رسیدند: پیشوند پایدار `cacheRead=4864`، نرخ اصابت `0.966`؛ رونوشت ابزار `cacheRead=4608`، نرخ اصابت `0.896`؛ رونوشت تصویر `cacheRead=4864`، نرخ اصابت `0.954`؛ رونوشت سبک MCP `cacheRead=4608`، نرخ اصابت `0.891`.

دلیل تفاوت بررسی‌ها: Anthropic نقاط شکست صریح کش و استفاده مجدد متحرک از تاریخچه مکالمه را ارائه می‌کند، درحالی‌که پیشوند عملاً قابل‌استفاده مجدد OpenAI در ترافیک زنده ممکن است پیش از رسیدن به کل پرامپت در سطح ثابتی متوقف شود. مقایسه این دو ارائه‌دهنده با یک آستانه درصدی واحد میان ارائه‌دهندگان باعث رگرسیون‌های کاذب می‌شود.

## پیکربندی `diagnostics.cacheTrace`

```yaml
diagnostics:
  cacheTrace:
    enabled: true
    filePath: "~/.openclaw/logs/cache-trace.jsonl" # اختیاری
    includeMessages: false # پیش‌فرض true
    includePrompt: false # پیش‌فرض true
    includeSystem: false # پیش‌فرض true
```

مقادیر پیش‌فرض:

| کلید               | پیش‌فرض                                      |
| ----------------- | -------------------------------------------- |
| `filePath`        | `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl` |
| `includeMessages` | `true`                                       |
| `includePrompt`   | `true`                                       |
| `includeSystem`   | `true`                                       |

### کلیدهای تغییر محیطی (اشکال‌زدایی موردی)

| متغیر                             | اثر                               |
| ------------------------------------ | ------------------------------------ |
| `OPENCLAW_CACHE_TRACE=1`             | ردیابی کش را فعال می‌کند                |
| `OPENCLAW_CACHE_TRACE_FILE=path`     | مسیر خروجی را بازنویسی می‌کند                |
| `OPENCLAW_CACHE_TRACE_MESSAGES=0\|1` | ثبت کامل محتوای پیام را تغییر می‌دهد |
| `OPENCLAW_CACHE_TRACE_PROMPT=0\|1`   | ثبت متن پرامپت را تغییر می‌دهد          |
| `OPENCLAW_CACHE_TRACE_SYSTEM=0\|1`   | ثبت پرامپت سیستم را تغییر می‌دهد        |

### چه مواردی را بررسی کنید

- رویدادهای ردیابی کش به‌صورت JSONL و دارای تصویرهای لحظه‌ای مرحله‌بندی‌شده‌ای مانند `session:loaded`، `prompt:before`، `stream:context` و `session:after` هستند.
- اثر توکن کش در هر نوبت در سطوح عادی مصرف قابل‌مشاهده است: `cacheRead` و `cacheWrite` در `/usage tokens`، `/status`، خلاصه‌های مصرف نشست و چیدمان‌های سفارشی `messages.usageTemplate` نمایش داده می‌شوند.
- برای Anthropic، هنگام فعال بودن کش انتظار هر دو مورد `cacheRead` و `cacheWrite` را داشته باشید.
- برای OpenAI، هنگام اصابت کش انتظار `cacheRead` را داشته باشید؛ `cacheWrite` فقط در محتوای Responses API که آن را شامل می‌شود مقداردهی می‌شود (بخش [OpenAI](#openai-direct-api) در بالا را ببینید).
- OpenAI همچنین سرآیندهای ردیابی و محدودیت نرخ مانند `x-request-id`، `openai-processing-ms` و `x-ratelimit-*` را برمی‌گرداند؛ از آن‌ها برای ردیابی درخواست استفاده کنید، اما محاسبه اصابت کش همچنان باید از محتوای مصرف انجام شود، نه از سرآیندها.

## عیب‌یابی سریع

- **مقدار بالای `cacheWrite` در بیشتر نوبت‌ها**: ورودی‌های ناپایدار پرامپت سیستم را بررسی کنید؛ مطمئن شوید مدل/ارائه‌دهنده از تنظیمات کش شما پشتیبانی می‌کند.
- **مقدار بالای `cacheWrite` در Anthropic**: اغلب به این معناست که نقطه شکست کش روی محتوایی قرار می‌گیرد که با هر درخواست تغییر می‌کند.
- **مقدار پایین `cacheRead` در OpenAI**: مطمئن شوید پیشوند پایدار در ابتدا قرار دارد، پیشوند تکراری حداقل 1024 توکن است و برای نوبت‌هایی که باید کش مشترک داشته باشند، همان `prompt_cache_key` دوباره استفاده می‌شود.
- **بی‌اثر بودن `cacheRetention`**: تأیید کنید کلید مدل با `agents.defaults.models["provider/model"]` مطابقت دارد.
- **درخواست‌های Bedrock Nova با تنظیمات کش**: مورد انتظار است — این درخواست‌ها در زمان اجرا بدون نگهداشت کش حل می‌شوند.

مستندات مرتبط:

- [Anthropic](/fa/providers/anthropic)
- [مصرف توکن و هزینه‌ها](/fa/reference/token-use)
- [هرس نشست](/fa/concepts/session-pruning)
- [مرجع پیکربندی Gateway](/fa/gateway/configuration-reference)

## مرتبط

- [مصرف توکن و هزینه‌ها](/fa/reference/token-use)
- [مصرف API و هزینه‌ها](/fa/reference/api-usage-costs)
