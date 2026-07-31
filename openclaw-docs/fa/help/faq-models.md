---
read_when:
    - انتخاب یا تغییر مدل‌ها، پیکربندی نام‌های مستعار
    - اشکال‌زدایی از جایگزینی خودکار مدل / «همه مدل‌ها ناموفق بودند»
    - آشنایی با پروفایل‌های احراز هویت و نحوه مدیریت آن‌ها
sidebarTitle: Models FAQ
summary: 'پرسش‌های متداول: پیش‌فرض‌های مدل، انتخاب، نام‌های مستعار، جابه‌جایی، انتقال در خرابی و پروفایل‌های احراز هویت'
title: 'پرسش‌های متداول: مدل‌ها و احراز هویت'
x-i18n:
    generated_at: "2026-07-27T16:38:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0c46d99352c5e51af5917c6f62b897dfa4030cb0201b8235e28f2f81f2954544
    source_path: help/faq-models.md
    workflow: 16
---

پرسش‌وپاسخ درباره مدل‌ها و پروفایل‌های احراز هویت. برای راه‌اندازی، نشست‌ها، Gateway، کانال‌ها و
عیب‌یابی، به [پرسش‌های متداول](/fa/help/faq) اصلی مراجعه کنید.

## مدل‌ها: پیش‌فرض‌ها، انتخاب، نام‌های مستعار و جابه‌جایی

<AccordionGroup>
  <Accordion title='«مدل پیش‌فرض» چیست؟'>
    با این گزینه تنظیم می‌شود:

    ```text
    agents.defaults.model.primary
    ```

    مدل‌ها ارجاع‌های `provider/model` هستند (برای نمونه: `openai/gpt-5.5`،
    `anthropic/claude-sonnet-4-6`). همیشه `provider/model` را صریحاً تنظیم کنید. اگر
    ارائه‌دهنده را حذف کنید، OpenClaw ابتدا تطبیق نام مستعار، سپس تطبیق یکتای
    ارائه‌دهنده پیکربندی‌شده برای آن شناسه مدل را امتحان می‌کند و در نهایت به
    ارائه‌دهنده پیش‌فرض پیکربندی‌شده بازمی‌گردد (مسیر سازگاری منسوخ‌شده). اگر آن
    ارائه‌دهنده دیگر مدل پیش‌فرض پیکربندی‌شده را نداشته باشد، OpenClaw به‌جای
    یک پیش‌فرض منسوخ، به نخستین ارائه‌دهنده/مدل پیکربندی‌شده بازمی‌گردد.

  </Accordion>

  <Accordion title="چه مدلی را پیشنهاد می‌کنید؟">
    از قدرتمندترین مدل نسل جدیدی که مجموعه ارائه‌دهندگان شما عرضه می‌کند استفاده کنید،
    به‌ویژه برای عامل‌های مجهز به ابزار یا عامل‌هایی که ورودی نامطمئن دریافت می‌کنند؛ مدل‌های
    ضعیف‌تر یا بیش‌ازحد کوانتیزه‌شده در برابر تزریق پرامپت و رفتار ناامن آسیب‌پذیرترند
    (به [امنیت](/fa/gateway/security) مراجعه کنید). مدل‌های ارزان‌تر را بر اساس نقش عامل به
    گفت‌وگوهای معمول و کم‌خطر اختصاص دهید.

    مدل‌ها را برای هر عامل مسیریابی کنید و برای موازی‌سازی وظایف طولانی از عامل‌های فرعی
    استفاده کنید (هر عامل فرعی توکن‌های خودش را مصرف می‌کند). به [مدل‌ها](/fa/concepts/models)،
    [عامل‌های فرعی](/fa/tools/subagents)، [MiniMax](/fa/providers/minimax) و
    [مدل‌های محلی](/fa/gateway/local-models) مراجعه کنید.

  </Accordion>

  <Accordion title="چگونه بدون پاک‌کردن پیکربندی، مدل‌ها را عوض کنم؟">
    فقط فیلدهای مدل را تغییر دهید؛ از جایگزینی کامل پیکربندی خودداری کنید.

    - `/model` در گفت‌وگو (برای هر نشست؛ به [دستورهای اسلش](/fa/tools/slash-commands) مراجعه کنید)
    - `openclaw models set ...` (فقط پیکربندی مدل را به‌روزرسانی می‌کند)
    - `openclaw configure --section model` (تعاملی)
    - ویرایش مستقیم `agents.defaults.model` در `~/.openclaw/openclaw.json`

    برای ویرایش‌های RPC، ابتدا با `config.schema.lookup` بررسی کنید (مسیر
    نرمال‌شده، مستندات سطحی طرح‌واره و خلاصه فرزندان)، سپس استفاده از `config.patch`
    را همراه با یک شیء جزئی به `config.apply` ترجیح دهید. اگر پیکربندی را
    بازنویسی کرده‌اید، آن را از نسخه پشتیبان بازیابی کنید یا برای تعمیر
    `openclaw doctor` را اجرا کنید.

    مستندات: [مدل‌ها](/fa/concepts/models)، [پیکربندی](/fa/cli/configure)،
    [پیکربندی](/fa/cli/config)، [Doctor](/fa/gateway/doctor).

  </Accordion>

  <Accordion title="آیا می‌توانم از مدل‌های خودمیزبان (llama.cpp، vLLM و Ollama) استفاده کنم؟">
    بله؛ Ollama ساده‌ترین مسیر است. راه‌اندازی سریع:

    1. Ollama را از `https://ollama.com/download` نصب کنید
    2. یک مدل محلی دریافت کنید، برای نمونه `ollama pull gemma4`
    3. برای مدل‌های ابری نیز `ollama signin` را اجرا کنید
    4. `openclaw onboard` را اجرا کنید، `Ollama` و سپس `Local` یا `Cloud + Local` را انتخاب کنید

    `Cloud + Local` مدل‌های ابری را در کنار مدل‌های محلی Ollama در اختیارتان
    قرار می‌دهد؛ مدل‌های ابری مانند `kimi-k2.5:cloud` به دریافت محلی نیاز ندارند.
    برای جابه‌جایی دستی: `openclaw models list` و سپس `openclaw models set ollama/<model>`.

    مدل‌های کوچک‌تر یا به‌شدت کوانتیزه‌شده در برابر تزریق پرامپت آسیب‌پذیرترند.
    برای هر رباتی که به ابزارها دسترسی دارد از مدل‌های بزرگ استفاده کنید؛ اگر بااین‌حال
    از مدل‌های کوچک استفاده می‌کنید، محیط ایزوله و فهرست‌های مجاز سخت‌گیرانه ابزارها را
    فعال کنید.

    مستندات: [Ollama](/fa/providers/ollama)، [مدل‌های محلی](/fa/gateway/local-models)،
    [ارائه‌دهندگان مدل](/fa/concepts/model-providers)، [امنیت](/fa/gateway/security)،
    [محیط ایزوله](/fa/gateway/sandboxing).

  </Accordion>

  <Accordion title="چگونه مدل‌ها را حین اجرا و بدون راه‌اندازی مجدد عوض کنم؟">
    `/model <name>` را به‌صورت پیامی مستقل ارسال کنید. برای فهرست کامل
    دستورها، از جمله انتخاب‌گر شماره‌دار (`/model`، `/model
    list`، `/model 3`)، گزینه
    `/model default` برای پاک‌کردن بازنویسی نشست و `/model status` برای جزئیات
    نقطه پایانی/حالت API، به [دستورهای اسلش](/fa/tools/slash-commands) مراجعه کنید.

    با `@profile` یک پروفایل احراز هویت مشخص را برای هر نشست اجبار کنید:

    ```text
    /model opus@anthropic:default
    /model opus@anthropic:work
    ```

    برای برداشتن سنجاق پروفایلی که با `@profile` تنظیم شده است،
    `/model` را بدون پسوند دوباره اجرا کنید (برای نمونه `/model anthropic/claude-opus-4-6`) یا
    مقدار پیش‌فرض را از `/model` انتخاب کنید. برای تأیید پروفایل احراز هویت
    فعال از `/model status` استفاده کنید.

  </Accordion>

  <Accordion title="اگر دو ارائه‌دهنده شناسه مدل یکسانی عرضه کنند، ‎/model از کدام‌یک استفاده می‌کند؟">
    `/model provider/model` دقیقاً همان مسیر ارائه‌دهنده را انتخاب می‌کند. برای نمونه،
    `qianfan/deepseek-v4-flash` و `deepseek/deepseek-v4-flash` با وجود یکسان‌بودن شناسه مدل، ارجاع‌های
    متفاوتی هستند؛ OpenClaw صرفاً با تطبیق یک شناسه بدون ارائه‌دهنده، ارائه‌دهندگان
    را بی‌سروصدا عوض نمی‌کند.

    ارجاع `/model` که کاربر انتخاب کرده است، برای بازگشت جایگزین سخت‌گیرانه
    عمل می‌کند: اگر آن ارائه‌دهنده/مدل از دسترس خارج شود، پاسخ به‌صورت آشکار شکست
    می‌خورد و به `agents.defaults.model.fallbacks` بازنمی‌گردد. زنجیره‌های بازگشت جایگزین
    پیکربندی‌شده همچنان برای پیش‌فرض‌های پیکربندی‌شده، مدل‌های اصلی کارهای Cron و
    وضعیت بازگشت جایگزین خودکار اعمال می‌شوند. هنگامی که اجرای فاقد بازنویسی نشست
    اجازه استفاده از بازگشت جایگزین را دارد، OpenClaw ابتدا ارائه‌دهنده/مدل
    درخواست‌شده، سپس گزینه‌های بازگشت پیکربندی‌شده و در پایان مدل اصلی
    پیکربندی‌شده را امتحان می‌کند؛ بنابراین شناسه‌های مدل یکسان و بدون ارائه‌دهنده
    هرگز مستقیماً به ارائه‌دهنده پیش‌فرض بازنمی‌گردند.

    به [مدل‌ها](/fa/concepts/models) و [جابه‌جایی خودکار مدل](/fa/concepts/model-failover) مراجعه کنید.

  </Accordion>

  <Accordion title="آیا می‌توانم برای وظایف روزانه از GPT 5.5 و برای برنامه‌نویسی از Codex 5.5 استفاده کنم؟">
    بله؛ انتخاب مدل و انتخاب محیط اجرا از یکدیگر جدا هستند:

    - **عامل برنامه‌نویسی بومی Codex:** مقدار `agents.defaults.model.primary` را روی
      `openai/gpt-5.5` تنظیم کنید. برای احراز هویت اشتراک ChatGPT/Codex با
      `openclaw models auth login --provider
      openai` وارد شوید.
    - **وظایف مستقیم OpenAI API خارج از حلقه عامل:** برای تصاویر،
      جاسازی‌ها، گفتار، بلادرنگ و دیگر سطوح غیرعاملی OpenAI API،
      `OPENAI_API_KEY` را پیکربندی کنید.
    - **احراز هویت عامل OpenAI با کلید API:** `/model openai/gpt-5.5` همراه با
      یک پروفایل مرتب‌شده کلید API در `openai`.
    - **عامل‌های فرعی:** وظایف برنامه‌نویسی را به عاملی متمرکز بر Codex با
      مدل `openai/gpt-5.5` خودش مسیریابی کنید.

    به [مدل‌ها](/fa/concepts/models) و [دستورهای اسلش](/fa/tools/slash-commands) مراجعه کنید.

  </Accordion>

  <Accordion title="چگونه حالت سریع را برای GPT 5.5 پیکربندی کنم؟">
    - **برای هر نشست:** هنگام استفاده از `openai/gpt-5.5`، مقدار `/fast on` را ارسال کنید.
    - **پیش‌فرض هر مدل:** مقدار
      `agents.defaults.models["openai/gpt-5.5"].params.fastMode` را روی `true` تنظیم کنید.
    - **حد توقف خودکار:** `/fast auto` یا `params.fastMode: "auto"` فراخوانی‌های جدید
      مدل را تا زمان حد توقف در حالت سریع اجرا می‌کند؛ سپس فراخوانی‌های بعدیِ تلاش مجدد،
      بازگشت جایگزین، نتیجه ابزار یا ادامه را بدون حالت سریع اجرا می‌کند. حد توقف به‌طور
      پیش‌فرض 60 ثانیه است؛ آن را با `params.fastAutoOnSeconds` روی مدل بازنویسی کنید.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openai/gpt-5.5": {
              params: {
                fastMode: "auto",
                fastAutoOnSeconds: 30,
              },
            },
          },
        },
      },
    }
    ```

    حالت سریع در درخواست‌های بومی OpenAI Responses به `service_tier = "priority"` نگاشت
    می‌شود؛ مقادیر موجود `service_tier` حفظ می‌شوند و حالت سریع
    `reasoning` یا `text.verbosity` را بازنویسی نمی‌کند. بازنویسی‌های نشست
    در `/fast` بر پیش‌فرض‌های پیکربندی اولویت دارند.

    به [تفکر و حالت سریع](/fa/tools/thinking) و بخش حالت سریع در قسمت پیکربندی پیشرفته
    صفحه ارائه‌دهنده [OpenAI](/fa/providers/openai) مراجعه کنید.

  </Accordion>

  <Accordion title='چرا پیام «Model ... is not allowed» را می‌بینم و سپس پاسخی دریافت نمی‌کنم؟'>
    اگر `agents.defaults.modelPolicy.allow` خالی نباشد، به **فهرست مجاز**
    برای `/model`، بازنویسی‌های نشست و `--model` تبدیل می‌شود. انتخاب مدلی خارج از آن فهرست،
    به‌جای پاسخ عادی، این پیام را برمی‌گرداند:

    ```text
    بازنویسی مدل "provider/model" توسط agents.defaults.modelPolicy.allow مجاز نیست.
    ```

    راه‌حل: مدل دقیق یا یک نویسه عام ارائه‌دهنده مانند `"provider/*"` را به
    فهرست نام‌گذاری‌شده `modelPolicy.allow` اضافه کنید، آن فهرست را حذف/خالی کنید یا
    مدلی را از `/model list` انتخاب کنید. اگر دستور شامل
    `--runtime codex` نیز بود، ابتدا فهرست مجاز را به‌روزرسانی کنید و سپس همان
    دستور `/model provider/model --runtime codex` را دوباره امتحان کنید.

  </Accordion>

  <Accordion title='چرا پیام «Unknown model: minimax/MiniMax-M3» را می‌بینم؟'>
    اگر از نسخه قدیمی‌تر OpenClaw استفاده می‌کنید، ابتدا ارتقا دهید (یا
    `main` را از منبع اجرا کنید) و Gateway را دوباره راه‌اندازی کنید؛
    ممکن است `MiniMax-M3` هنوز در کاتالوگ نسخه نصب‌شده شما نباشد. در غیر این
    صورت، ارائه‌دهنده MiniMax پیکربندی نشده است (هیچ ورودی ارائه‌دهنده یا پروفایل
    احراز هویتی پیدا نشده است)، بنابراین مدل قابل تفکیک نیست. برای فهرست بررسی کامل
    راه‌حل‌ها، جدول شناسه ارائه‌دهنده/مدل و نمونه بلوک پیکربندی، به بخش عیب‌یابی
    صفحه ارائه‌دهنده [MiniMax](/fa/providers/minimax) مراجعه کنید.

  </Accordion>

  <Accordion title="آیا می‌توانم MiniMax را پیش‌فرض و OpenAI را برای وظایف پیچیده استفاده کنم؟">
    بله. MiniMax را به‌عنوان پیش‌فرض استفاده کنید و مدل‌ها را برای هر نشست تغییر دهید؛
    بازگشت‌های جایگزین برای خطاها هستند، نه «وظایف دشوار»، بنابراین از
    `/model` یا یک عامل جداگانه استفاده کنید.

    **گزینه الف: جابه‌جایی برای هر نشست**

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-...", OPENAI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "minimax" },
            "openai/gpt-5.5": { alias: "gpt" },
          },
        },
      },
    }
    ```

    سپس `/model gpt`.

    **گزینه ب: عامل‌های جداگانه** — پیش‌فرض عامل الف MiniMax و پیش‌فرض عامل ب
    OpenAI است؛ بر اساس عامل مسیریابی کنید یا برای جابه‌جایی از `/agent` استفاده کنید.

    مستندات: [مدل‌ها](/fa/concepts/models)، [مسیریابی چندعاملی](/fa/concepts/multi-agent)،
    [MiniMax](/fa/providers/minimax)، [OpenAI](/fa/providers/openai).

  </Accordion>

  <Accordion title="آیا opus / sonnet / gpt میان‌برهای داخلی هستند؟">
    بله؛ این صورت‌های کوتاه داخلی فقط زمانی اعمال می‌شوند که مدل مقصد در
    `agents.defaults.models` وجود داشته باشد:

    | نام مستعار | تفکیک می‌شود به |
    | --- | --- |
    | `opus` | `anthropic/claude-opus-5` |
    | `sonnet` | `anthropic/claude-sonnet-5` |
    | `gpt` | `openai/gpt-5.4` |
    | `gpt-mini` | `openai/gpt-5.4-mini` |
    | `gpt-nano` | `openai/gpt-5.4-nano` |
    | `gemini` | `google/gemini-3.1-pro-preview` |
    | `gemini-flash` | `google/gemini-3-flash-preview` |
    | `gemini-flash-lite` | `google/gemini-3.1-flash-lite` |

    نام مستعار تعریف‌شده توسط شما با همان نام، نام مستعار داخلی را بازنویسی می‌کند.

  </Accordion>

  <Accordion title="چگونه میان‌برهای مدل (نام‌های مستعار) را تعریف/بازنویسی کنم؟">
    نام‌های مستعار در `agents.defaults.models.<modelId>.alias` قرار دارند:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": { alias: "opus" },
            "anthropic/claude-sonnet-4-6": { alias: "sonnet" },
          },
        },
      },
    }
    ```

    سپس `/model sonnet` (یا در صورت پشتیبانی، `/<alias>`) به آن
    شناسه مدل تفکیک می‌شود.

  </Accordion>

  <Accordion title="چگونه مدل‌هایی از ارائه‌دهندگان دیگر مانند OpenRouter یا Z.AI اضافه کنم؟">
    OpenRouter (پرداخت به‌ازای هر توکن؛ مدل‌های فراوان):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "openrouter/anthropic/claude-sonnet-4-6" },
          models: { "openrouter/anthropic/claude-sonnet-4-6": {} },
        },
      },
      env: { OPENROUTER_API_KEY: "sk-or-..." },
    }
    ```

    Z.AI (مدل‌های GLM):

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-5.1" },
          models: { "zai/glm-5.1": {} },
        },
      },
      env: { ZAI_API_KEY: "..." },
    }
    ```

    نبود کلید ارائه‌دهنده برای یک ارائه‌دهنده/مدل ارجاع‌شده، هنگام اجرا
    خطای احراز هویت ایجاد می‌کند (برای نمونه `No API key found for provider "zai"`).

    **پس از افزودن عامل جدید، هیچ کلید API برای ارائه‌دهنده پیدا نشد**

    یک عامل جدید مخزن احراز هویت خالی دارد؛ احراز هویت برای هر عامل جداگانه است و
    در این مسیر ذخیره می‌شود:

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    رفع مشکل: `openclaw agents add <id>` را اجرا کنید و احراز هویت را در راه‌انداز پیکربندی کنید، یا
    فقط پروفایل‌های ایستای قابل‌انتقال `api_key`/`token` را از مخزن عامل اصلی
    کپی کنید. برای OAuth، هنگامی که عامل جدید به حساب
    اختصاصی خود نیاز دارد، از همان عامل وارد شوید. برای مشاهده قواعد کامل استفادهٔ مجدد از
    `agentDir` و اشتراک‌گذاری اعتبارنامه‌ها، به [مسیریابی چندعاملی](/fa/concepts/multi-agent) مراجعه کنید — هرگز
    `agentDir` را میان عامل‌ها دوباره استفاده نکنید.

  </Accordion>
</AccordionGroup>

## جایگزینی مدل هنگام خرابی و «همهٔ مدل‌ها ناموفق بودند»

<AccordionGroup>
  <Accordion title="جایگزینی هنگام خرابی چگونه کار می‌کند؟">
    دو مرحله دارد:

    1. **چرخش پروفایل احراز هویت** در همان ارائه‌دهنده.
    2. **مدل جایگزین** به مدل بعدی در `agents.defaults.model.fallbacks`.

    برای پروفایل‌های ناموفق دوره‌های انتظار اعمال می‌شود (پس‌روی نمایی)، بنابراین OpenClaw
    هنگام محدودیت نرخ ارائه‌دهنده یا خرابی موقت آن، به پاسخ‌گویی ادامه می‌دهد.

    سبد محدودیت نرخ فقط `429` ساده را پوشش نمی‌دهد: `Too many concurrent
    requests`، `ThrottlingException`، `concurrency limit reached`، `workers_ai
    ... quota limit exceeded`، `resource exhausted` و محدودیت‌های دوره‌ای
    پنجرهٔ مصرف (`weekly/monthly limit reached`) همگی محدودیت نرخی محسوب می‌شوند
    که استفاده از جایگزین را توجیه می‌کنند.

    پاسخ‌های مربوط به صورت‌حساب همیشه `402` نیستند و برخی `402`ها به‌جای
    مسیر صورت‌حساب، در سبد خطاهای گذرا/محدودیت نرخ باقی می‌مانند. متن صریح
    صورت‌حساب در `401`/`403` همچنان می‌تواند به مسیر صورت‌حساب هدایت شود؛ تطبیق‌دهنده‌های متنی
    ویژهٔ ارائه‌دهنده (برای مثال `Key limit exceeded` در OpenRouter) فقط به
    ارائه‌دهندهٔ خود محدود می‌مانند. یک `402` که شبیه محدودیت قابل‌تلاش‌مجدد پنجرهٔ مصرف یا
    سقف هزینهٔ سازمان/فضای کاری باشد (`daily limit reached, resets tomorrow`،
    `organization spending limit exceeded`) به‌عنوان `rate_limit` در نظر گرفته می‌شود، نه
    غیرفعال‌سازی طولانی‌مدت به‌دلیل صورت‌حساب.

    خطاهای سرریز زمینه به‌طور کامل از مسیر جایگزینی کنار می‌مانند — امضاهایی
    مانند `request_too_large`، `input exceeds the maximum number of tokens`،
    `input token count exceeds the maximum number of input tokens`، `input is
    too long for the model` یا `ollama error: context length exceeded` به‌جای رفتن به
    مدل جایگزین، به Compaction/تلاش مجدد هدایت می‌شوند.

    دامنهٔ متن عمومی خطای سرور محدودتر از «هر چیزی است که در آن unknown/error
    وجود دارد». قالب‌های گذرای مختص ارائه‌دهنده که سیگنال استفاده از جایگزین
    محسوب می‌شوند عبارت‌اند از: `An unknown error occurred` بدون جزئیات از Anthropic، `Provider returned error`
    بدون جزئیات از OpenRouter، خطاهای دلیل توقف مانند `Unhandled stop reason:
    error`، محتوای JSON
    از نوع `api_error` با متن گذرای سرور (`internal
    server error`، `unknown error, 520`، `upstream error`، `backend error`)
    و خطاهای مشغول‌بودن ارائه‌دهنده مانند `ModelNotReadyException`، مشروط بر اینکه زمینهٔ
    ارائه‌دهنده مطابقت داشته باشد. متن عمومی جایگزین داخلی مانند `LLM request failed
    with an unknown error.` محافظه‌کارانه باقی می‌ماند و به‌تنهایی باعث
    استفاده از جایگزین نمی‌شود.

  </Accordion>

  <Accordion title='پیام «No credentials found for profile anthropic:default» به چه معناست؟'>
    شناسهٔ پروفایل احراز هویت `anthropic:default` در مخزن احراز هویت
    مورد انتظار هیچ اعتبارنامه‌ای ندارد.

    **چک‌لیست رفع مشکل:**

    - محل نگهداری پروفایل‌ها را تأیید کنید — محل فعلی:
      `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`؛ محل قدیمی:
      `~/.openclaw/agent/*` (توسط `openclaw doctor` مهاجرت داده می‌شود).
    - تأیید کنید Gateway متغیر محیطی شما را بارگذاری می‌کند. `ANTHROPIC_API_KEY` که فقط در
      پوستهٔ شما تنظیم شده باشد، به Gateway اجراشده از طریق systemd/launchd نمی‌رسد — آن را در
      `~/.openclaw/.env` قرار دهید یا `env.shellEnv` را فعال کنید.
    - تأیید کنید عامل درست را ویرایش می‌کنید — پیکربندی‌های چندعاملی
      چندین فایل `auth-profiles.json` دارند.
    - برای مشاهدهٔ مدل‌های پیکربندی‌شده و وضعیت احراز هویت
      ارائه‌دهنده، `openclaw models status` را اجرا کنید.

    **برای «No credentials found for profile anthropic» (بدون پسوند ایمیل):**

    اجرا به یک پروفایل Anthropic متصل شده است که Gateway نمی‌تواند آن را پیدا کند.

    - از Claude CLI استفاده کنید: `openclaw models auth login --provider anthropic
      --method cli --set-default` را روی میزبان Gateway اجرا کنید.
    - اگر کلید API را ترجیح می‌دهید: `ANTHROPIC_API_KEY` را در
      `~/.openclaw/.env` روی میزبان Gateway قرار دهید، سپس هر ترتیب تثبیت‌شده‌ای را که
      پروفایل مفقود را تحمیل می‌کند پاک کنید:

      ```bash
      openclaw models auth order clear --provider anthropic
      ```

    - حالت راه‌دور: پروفایل‌های احراز هویت روی دستگاه Gateway قرار دارند، نه روی
      لپ‌تاپ شما — تأیید کنید فرمان‌ها را همان‌جا اجرا می‌کنید.

  </Accordion>

  <Accordion title="چرا Google Gemini را نیز امتحان کرد و ناموفق بود؟">
    اگر پیکربندی مدل شما Google Gemini را به‌عنوان مدل جایگزین شامل شود (یا به
    یک نام کوتاه Gemini تغییر داده باشید)، OpenClaw هنگام جایگزینی آن را امتحان می‌کند. نبود
    اعتبارنامهٔ پیکربندی‌شده برای Google باعث `No API key found for provider
    "google"` می‌شود. رفع مشکل: احراز هویت Google را اضافه کنید یا مدل‌های Google را از
    `agents.defaults.model.fallbacks`/نام‌های مستعار حذف کنید.

    **درخواست LLM رد شد: امضای تفکر الزامی است (Google Antigravity)**

    علت: تاریخچهٔ نشست دارای بلوک‌های تفکر بدون امضا است (اغلب
    ناشی از جریان لغوشده/ناتمام)؛ Google Antigravity برای بلوک‌های تفکر به امضا
    نیاز دارد. OpenClaw بلوک‌های تفکر بدون امضا را برای Google
    Antigravity Claude حذف می‌کند؛ اگر همچنان ظاهر شد، نشست جدیدی آغاز کنید یا
    `/thinking off` را برای آن عامل تنظیم کنید.

  </Accordion>
</AccordionGroup>

## پروفایل‌های احراز هویت: چیستی و نحوهٔ مدیریت آن‌ها

مرتبط: [/concepts/oauth](/fa/concepts/oauth) (جریان‌های OAuth، ذخیره‌سازی توکن، الگوهای چندحسابی)

<AccordionGroup>
  <Accordion title="پروفایل احراز هویت چیست؟">
    یک رکورد نام‌گذاری‌شدهٔ اعتبارنامه (OAuth یا کلید API) که به یک ارائه‌دهنده متصل است و
    در محل زیر ذخیره می‌شود:

    ```text
    ~/.openclaw/agents/<agentId>/agent/auth-profiles.json
    ```

    پروفایل‌های ذخیره‌شده را بدون نمایش اسرار بررسی کنید: `openclaw models auth
    list` (در صورت نیاز `--provider <id>` یا `--json`). به
    [CLI مدل‌ها](/fa/cli/models#auth-profiles) مراجعه کنید.

  </Accordion>

  <Accordion title="شناسه‌های متداول پروفایل کدام‌اند؟">
    دارای پیشوند ارائه‌دهنده: `anthropic:default` (هنگامی که هویت ایمیلی
    وجود ندارد متداول است)، `anthropic:<email>` برای هویت‌های OAuth، یا یک شناسهٔ سفارشی که
    انتخاب می‌کنید (برای مثال `anthropic:work`).

  </Accordion>

  <Accordion title="آیا می‌توانم تعیین کنم کدام پروفایل احراز هویت نخست امتحان شود؟">
    بله. پیکربندی `auth.order.<provider>` ترتیب چرخش را برای هر ارائه‌دهنده تعیین می‌کند
    (فقط فراداده — هیچ رازی ذخیره نمی‌شود).

    OpenClaw ممکن است پروفایلی را که در وضعیت کوتاه‌مدت **دورهٔ انتظار** (محدودیت نرخ،
    وقفهٔ زمانی، خطاهای احراز هویت) یا وضعیت طولانی‌تر **غیرفعال**
    (صورت‌حساب/اعتبار ناکافی) است، نادیده بگیرد. با `openclaw models status
    --json` بررسی کنید و `auth.unusableProfiles` را ببینید. دوره‌های انتظار محدودیت نرخ می‌توانند
    مختص مدل باشند — پروفایلی که برای یک مدل در دورهٔ انتظار است، همچنان می‌تواند به
    مدل هم‌خانواده‌ای در همان ارائه‌دهنده خدمت‌رسانی کند؛ پنجره‌های صورت‌حساب/غیرفعال‌شدن
    کل پروفایل را مسدود می‌کنند.

    یک ترتیب سفارشی برای هر عامل تنظیم کنید (در `auth-state.json` همان عامل ذخیره می‌شود):

    ```bash
    # Defaults to the configured default agent (omit --agent)
    openclaw models auth order get --provider anthropic

    # Lock rotation to a single profile
    openclaw models auth order set --provider anthropic anthropic:default

    # Or set an explicit order (fallback within provider)
    openclaw models auth order set --provider anthropic anthropic:work anthropic:default

    # Clear override (fall back to config auth.order / round-robin)
    openclaw models auth order clear --provider anthropic

    # Target a specific agent
    openclaw models auth order set --provider anthropic --agent main anthropic:default
    ```

    آنچه واقعاً امتحان خواهد شد را بررسی کنید: `openclaw models status --probe`. یک
    پروفایل ذخیره‌شده که از ترتیب صریح حذف شده باشد، به‌جای اینکه بی‌سروصدا امتحان شود،
    `excluded_by_auth_order` را گزارش می‌کند.

  </Accordion>

  <Accordion title="تفاوت OAuth و کلید API چیست؟">
    - **ورود OAuth / CLI** اغلب در ارائه‌دهندگانی که از آن پشتیبانی می‌کنند، از دسترسی اشتراکی
      استفاده می‌کند. برای Anthropic، بک‌اند Claude CLI در OpenClaw
      از `claude -p` در Claude Code استفاده می‌کند که Anthropic در حال حاضر آن را
      استفادهٔ Agent SDK/برنامه‌نویسی با کسر از محدودیت‌های مصرف اشتراک در نظر می‌گیرد —
      برای وضعیت فعلی توقف صورت‌حساب و پیوندهای منبع به [Anthropic](/fa/providers/anthropic)
      مراجعه کنید.
    - **کلیدهای API** از صورت‌حساب بر مبنای پرداخت به‌ازای هر توکن استفاده می‌کنند.

    راه‌انداز از Anthropic Claude CLI، ‏OAuth در OpenAI Codex و کلیدهای API
    پشتیبانی می‌کند.

  </Accordion>
</AccordionGroup>

## مطالب مرتبط

- [پرسش‌های متداول](/fa/help/faq) — پرسش‌های متداول اصلی
- [پرسش‌های متداول — شروع سریع و راه‌اندازی اجرای نخست](/fa/help/faq-first-run)
- [انتخاب مدل](/fa/concepts/model-providers)
- [جایگزینی مدل هنگام خرابی](/fa/concepts/model-failover)
