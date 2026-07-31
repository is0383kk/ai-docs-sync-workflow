---
read_when:
    - به یک مرجع راه‌اندازی مدل برای تک‌تک ارائه‌دهندگان نیاز دارید
    - پیکربندی‌های نمونه یا فرمان‌های راه‌اندازی اولیه CLI برای ارائه‌دهندگان مدل می‌خواهید
sidebarTitle: Model providers
summary: مروری بر ارائه‌دهندگان مدل همراه با نمونه پیکربندی‌ها و جریان‌های CLI
title: ارائه‌دهندگان مدل
x-i18n:
    generated_at: "2026-07-27T15:05:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 51ce1ca5dde28821596ca667619cd860cebda4787993fadb6b0e20fc0e1624ac
    source_path: concepts/model-providers.md
    workflow: 16
---

مرجع **ارائه‌دهندگان LLM/مدل** (نه کانال‌های گفت‌وگو مانند WhatsApp/Telegram). برای قواعد انتخاب مدل، به [مدل‌ها](/fa/concepts/models) مراجعه کنید.

## قواعد سریع

<AccordionGroup>
  <Accordion title="ارجاع‌های مدل و ابزارهای کمکی CLI">
    - ارجاع‌های مدل از `provider/model` استفاده می‌کنند (مثال: `opencode/claude-opus-4-6`).
    - `agents.defaults.models` نام‌های مستعار و تنظیمات هر مدل را ذخیره می‌کند؛ `agents.defaults.modelPolicy.allow` فهرست مجاز اختیاری برای بازنویسی صریح است.
    - ابزارهای کمکی CLI: `openclaw onboard`، `openclaw models list`، `openclaw models set <provider/model>`.
    - `models.providers.*.contextWindow` / `contextTokens` / `maxTokens` پیش‌فرض‌های سطح ارائه‌دهنده را تنظیم می‌کنند؛ `models.providers.*.models[].contextWindow` / `contextTokens` / `maxTokens` آن‌ها را برای هر مدل بازنویسی می‌کنند.
    - قواعد جایگزینی، کاوش‌های دوره انتظار، و ماندگاری بازنویسی نشست: [جایگزینی مدل](/fa/concepts/model-failover).

  </Accordion>
  <Accordion title="افزودن احراز هویت ارائه‌دهنده، مدل اصلی را تغییر نمی‌دهد">
    `openclaw configure` هنگام افزودن یا احراز هویت مجدد یک ارائه‌دهنده، `agents.defaults.model.primary` موجود را حفظ می‌کند. `openclaw models auth login` نیز همین کار را انجام می‌دهد، مگر اینکه `--set-default` را ارسال کنید. Pluginهای ارائه‌دهنده همچنان ممکن است در وصله پیکربندی احراز هویت خود یک مدل پیش‌فرض پیشنهادی برگردانند، اما وقتی مدل اصلی از قبل وجود داشته باشد، OpenClaw آن را به‌معنای «این مدل را در دسترس قرار بده» در نظر می‌گیرد، نه «مدل اصلی فعلی را جایگزین کن».

    برای تغییر عمدی مدل پیش‌فرض، از `openclaw models set <provider/model>` یا `openclaw models auth login --provider <id> --set-default` استفاده کنید.

  </Accordion>
  <Accordion title="تفکیک ارائه‌دهنده/زمان‌اجرای OpenAI">
    ارجاع‌های مدل OpenAI و زمان‌های اجرای عامل از یکدیگر جدا هستند:

    - `openai/<model>` ارائه‌دهنده و مدل معیار OpenAI را انتخاب می‌کند. پیشوند به‌تنهایی هرگز Codex را انتخاب نمی‌کند.
    - وقتی سیاست زمان‌اجرای ارائه‌دهنده/مدل تنظیم نشده باشد یا `auto` باشد، OpenAI فقط برای یک مسیر رسمی و دقیق HTTPS از نوع Platform Responses یا ChatGPT Responses که هیچ بازنویسی تألیفی درخواست ندارد، می‌تواند Codex را به‌طور ضمنی انتخاب کند.
    - سازگارکننده‌های Completions تألیفی، نقاط پایانی سفارشی، و مسیرهای دارای رفتار تألیفی درخواست روی OpenClaw باقی می‌مانند. نقاط پایانی رسمی HTTP متن ساده رد می‌شوند.
    - ارجاع‌های قدیمی مدل Codex پیکربندی منسوخی هستند که doctor آن‌ها را به `openai/<model>` بازنویسی می‌کند.
    - `agentRuntime.id: "openclaw"` ارائه‌دهنده/مدل، مسیری را که در غیر این صورت واجد شرایط است، صریحاً روی OpenClaw نگه می‌دارد. `agentRuntime.id: "codex"` به Codex نیاز دارد و وقتی مسیر مؤثر با Codex سازگار نباشد، به‌صورت بسته شکست می‌خورد.

    به [زمان‌اجرای ضمنی عامل OpenAI](/fa/providers/openai#implicit-agent-runtime) و [چارچوب Codex](/fa/plugins/codex-harness) مراجعه کنید. اگر تفکیک ارائه‌دهنده/زمان‌اجرا گیج‌کننده است، ابتدا [زمان‌های اجرای عامل](/fa/concepts/agent-runtimes) را بخوانید.

    فعال‌سازی خودکار Plugin نیز از همین مرز پیروی می‌کند: یک مسیر مؤثر که به‌طور ضمنی با Codex سازگار است می‌تواند Plugin مربوط به Codex را فعال کند، درحالی‌که `agentRuntime.id: "codex"` صریح ارائه‌دهنده/مدل یا ارجاع‌های قدیمی `codex/<model>` به آن نیاز دارند. پیشوند `openai/*` به‌تنهایی چنین کاری نمی‌کند.

    راه‌اندازی جدید OpenAI از ارجاع GPT-5.6 مختص مسیر استفاده می‌کند: راه‌اندازی با کلید API،
    `openai/gpt-5.6` را انتخاب می‌کند (شناسه ساده API مستقیم به Sol تفکیک می‌شود)، درحالی‌که
    OAuth مربوط به ChatGPT/Codex، مقدار دقیق `openai/gpt-5.6-sol` را برای کاتالوگ بومی Codex
    انتخاب می‌کند. مدل‌های اصلی صریح موجود، از جمله `openai/gpt-5.5`، هنگام افزودن یا
    تازه‌سازی احراز هویت OpenAI حفظ می‌شوند. GPT-5.5 همچنان از طریق هر دو زمان‌اجرا
    به‌عنوان انتخاب بازیابی صریح برای حساب‌های بدون دسترسی به GPT-5.6 در دسترس است.

  </Accordion>
  <Accordion title="زمان‌های اجرای CLI">
    زمان‌های اجرای CLI از همین تفکیک استفاده می‌کنند: ارجاع‌های معیار مدل مانند `anthropic/claude-*` یا `google/gemini-*` را انتخاب کنید، سپس وقتی یک پشتیبان محلی CLI می‌خواهید، سیاست زمان‌اجرای ارائه‌دهنده/مدل را روی `claude-cli` یا `google-gemini-cli` تنظیم کنید.

    ارجاع‌های قدیمی `claude-cli/*` و `google-gemini-cli/*` با ثبت جداگانه زمان‌اجرا، دوباره به ارجاع‌های معیار ارائه‌دهنده مهاجرت می‌کنند. ارجاع‌های قدیمی `codex-cli/*` به `openai/*` مهاجرت می‌کنند و از مسیر سرور برنامه Codex استفاده می‌کنند؛ OpenClaw دیگر پشتیبان CLI بسته‌بندی‌شده Codex را نگه نمی‌دارد.

  </Accordion>
</AccordionGroup>

## پیکربندی ارائه‌دهندگان در رابط کنترل

برای افزودن، جایگزینی یا حذف کلیدهای API ارائه‌دهندگان که در `models.providers.<id>.apiKey` ذخیره شده‌اند، **Settings → Model Providers** را در رابط کنترل باز کنید. این صفحه بدون نمایش اعتبارنامه مشخص می‌کند که هر کلید API از پیکربندی OpenClaw می‌آید یا از یک متغیر محیطی. مدیریت کلیدهای ارائه‌شده از محیط همچنان بر عهده محیط فرایند Gateway است.

برای اجرای یک کاوش زنده ارائه‌دهنده و مشاهده تأخیر یا خطای دسته‌بندی‌شده احراز هویت، محدودیت نرخ، صورت‌حساب، پایان مهلت یا پاسخ، از **Test connection** استفاده کنید. یک کاوش، درخواستی واقعی به ارائه‌دهنده ارسال می‌کند و ممکن است تعداد کمی توکن مصرف کند. همچنین می‌توان از طریق کارت ارائه‌دهنده، از نمایه‌های OAuth و توکن خارج شد.

کارت **Default models** مدل اصلی، جایگزین‌های مرتب‌شده و مدل کاربردی را از کاتالوگ مدل پیکربندی‌شده مدیریت می‌کند. مدل‌ها را انتخاب کنید، سپس آن‌ها را با هم در تنظیمات موجود `agents.defaults.model` و `agents.defaults.utilityModel` ذخیره کنید. برای مدل کاربردی، **Automatic** تنظیم را بدون مقدار باقی می‌گذارد و **Disabled** یک رشته خالی ذخیره می‌کند تا مسیریابی کاربردی خاموش شود.

## رفتار ارائه‌دهنده تحت مالکیت Plugin

بیشتر منطق مختص ارائه‌دهنده در Pluginهای ارائه‌دهنده (`registerProvider(...)`) قرار دارد، درحالی‌که OpenClaw حلقه استنتاج عمومی را نگه می‌دارد. Pluginها مالک فرایند آغاز به کار، کاتالوگ‌های مدل، نگاشت متغیر محیطی احراز هویت، نرمال‌سازی انتقال/پیکربندی، پاک‌سازی شِمای ابزار، دسته‌بندی جایگزینی، تازه‌سازی OAuth، گزارش مصرف، نمایه‌های تفکر/استدلال و موارد دیگر هستند.

فهرست کامل قلاب‌های SDK ارائه‌دهنده و نمونه‌های Plugin بسته‌بندی‌شده در [Pluginهای ارائه‌دهنده](/fa/plugins/sdk-provider-plugins) قرار دارد. ارائه‌دهنده‌ای که به اجراکننده درخواست کاملاً سفارشی نیاز دارد، یک سطح گسترش جداگانه و عمیق‌تر محسوب می‌شود.

<Note>
رفتار اجراکننده تحت مالکیت ارائه‌دهنده روی قلاب‌های صریح ارائه‌دهنده مانند سیاست بازپخش، نرمال‌سازی شِمای ابزار، پوشش‌دهی جریان، و ابزارهای کمکی انتقال/درخواست قرار دارد. مجموعه ایستای قدیمی `ProviderPlugin.capabilities` فقط برای سازگاری است و دیگر توسط منطق اجراکننده مشترک خوانده نمی‌شود.
</Note>

## چرخش کلید API

<AccordionGroup>
  <Accordion title="منابع کلید و اولویت">
    چندین کلید را از طریق موارد زیر پیکربندی کنید:

    - `OPENCLAW_LIVE_<PROVIDER>_KEY` (بازنویسی زنده تکی، بالاترین اولویت)
    - `<PROVIDER>_API_KEYS` (فهرست جداشده با ویرگول یا نقطه‌ویرگول)
    - `<PROVIDER>_API_KEY` (کلید اصلی)
    - `<PROVIDER>_API_KEY_*` (فهرست شماره‌گذاری‌شده، برای مثال `<PROVIDER>_API_KEY_1`)

    برای ارائه‌دهندگان Google، `GOOGLE_API_KEY` نیز به‌عنوان جایگزین گنجانده می‌شود. ترتیب انتخاب کلید، اولویت را حفظ و مقادیر تکراری را حذف می‌کند.

  </Accordion>
  <Accordion title="زمان فعال‌شدن چرخش">
    - درخواست‌ها فقط در پاسخ‌های محدودیت نرخ (برای مثال `429`، `rate_limit`، `quota`، `resource exhausted`، `Too many concurrent requests`، `ThrottlingException`، `concurrency limit reached`، `workers_ai ... quota limit exceeded` یا پیام‌های دوره‌ای محدودیت مصرف) با کلید بعدی دوباره امتحان می‌شوند.
    - خطاهای غیرمرتبط با محدودیت نرخ بلافاصله شکست می‌خورند؛ چرخش کلید انجام نمی‌شود.
    - وقتی همه کلیدهای نامزد شکست بخورند، خطای نهایی از آخرین تلاش بازگردانده می‌شود.

  </Accordion>
</AccordionGroup>

## Pluginهای رسمی ارائه‌دهنده

Pluginهای رسمی ارائه‌دهنده، ردیف‌های کاتالوگ مدل خود را منتشر می‌کنند. این ارائه‌دهندگان به **هیچ** ورودی مدل `models.providers` نیاز ندارند؛ Plugin ارائه‌دهنده را فعال کنید، احراز هویت را تنظیم کنید و یک مدل برگزینید. از `models.providers` فقط برای ارائه‌دهندگان سفارشی صریح یا تنظیمات محدود درخواست مانند پایان مهلت‌ها استفاده کنید.

### OpenAI

- ارائه‌دهنده: `openai`
- احراز هویت: `OPENAI_API_KEY`
- چرخش اختیاری: `OPENAI_API_KEYS`، `OPENAI_API_KEY_1`، `OPENAI_API_KEY_2`، به‌علاوه `OPENCLAW_LIVE_OPENAI_KEY` (بازنویسی تکی)
- پیش‌فرض راه‌اندازی جدید: `openai/gpt-5.6`؛ در API مستقیم، شناسه ساده به Sol تفکیک می‌شود.
- مدل‌های نمونه: `openai/gpt-5.6`، `openai/gpt-5.6-terra`، `openai/gpt-5.6-luna`، `openai/gpt-5.5`
- اگر یک نصب یا کلید API مشخص رفتار متفاوتی دارد، دسترس‌پذیری حساب/مدل را با `openclaw models list --provider openai` بررسی کنید.
- CLI: `openclaw onboard --auth-choice openai-api-key`
- انتقال پیش‌فرض `auto` است؛ OpenClaw انتخاب انتقال را به زمان‌اجرای مشترک مدل می‌فرستد.
- برای هر مدل از طریق `agents.defaults.models["openai/<model>"].params.transport` بازنویسی کنید (`"sse"`، `"websocket"` یا `"auto"`)
- پردازش اولویت‌دار OpenAI را می‌توان از طریق `agents.defaults.models["openai/<model>"].params.serviceTier` فعال کرد
- `/fast` و `params.fastMode` درخواست‌های مستقیم Responses مربوط به `openai/*` را در `api.openai.com` به `service_tier=priority` نگاشت می‌کنند
- وقتی به‌جای کلید مشترک `/fast` یک سطح صریح می‌خواهید، از `params.serviceTier` استفاده کنید
- سرآیندهای پنهان انتساب OpenClaw (`originator`، `version`، `User-Agent`) فقط روی ترافیک بومی OpenAI به `api.openai.com` اعمال می‌شوند، نه پراکسی‌های عمومی سازگار با OpenAI
- مسیرهای بومی OpenAI همچنین `store` مربوط به Responses، راهنمایی‌های کش پرامپت و شکل‌دهی محموله سازگاری استدلال OpenAI را حفظ می‌کنند؛ مسیرهای پراکسی چنین نمی‌کنند
- `openai/gpt-5.3-codex-spark` فقط از طریق OAuth مربوط به ChatGPT/Codex در دسترس است؛ مسیرهای کلید API مستقیم OpenAI و کلید API مربوط به Azure آن را رد می‌کنند

```json5
{
  agents: { defaults: { model: { primary: "openai/gpt-5.6" } } },
}
```

اگر سازمان API، ‏GPT-5.6 را ارائه نمی‌کند،
`openai/gpt-5.5` را صریحاً تنظیم کنید. آغاز به کار و احراز هویت مجدد عادی،
مدل اصلی صریح موجود را حفظ می‌کنند؛ `models auth login --set-default` و
`models set` مسیرهای جایگزینی عمدی هستند.

### Anthropic

- ارائه‌دهنده: `anthropic`
- احراز هویت: `ANTHROPIC_API_KEY`
- چرخش اختیاری: `ANTHROPIC_API_KEYS`، `ANTHROPIC_API_KEY_1`، `ANTHROPIC_API_KEY_2`، به‌علاوه `OPENCLAW_LIVE_ANTHROPIC_KEY` (بازنویسی تکی)
- مدل نمونه: `anthropic/claude-opus-5`
- CLI: `openclaw onboard --auth-choice apiKey`
- درخواست‌های عمومی مستقیم Anthropic از کلید مشترک `/fast` و `params.fastMode` پشتیبانی می‌کنند، از جمله ترافیک احراز هویت‌شده با کلید API و OAuth که به `api.anthropic.com` ارسال می‌شود؛ OpenClaw آن را به `service_tier` مربوط به Anthropic نگاشت می‌کند (`auto` در برابر `standard_only`)
- پیکربندی ترجیحی Claude CLI ارجاع مدل را معیار نگه می‌دارد و پشتیبان CLI را
  جداگانه انتخاب می‌کند: `anthropic/claude-opus-5` همراه با
  `agentRuntime.id: "claude-cli"` در سطح مدل. ارجاع‌های قدیمی
  `claude-cli/claude-opus-4-7` همچنان برای سازگاری کار می‌کنند.

<Note>
استفاده مجدد از Claude CLI‏ (`claude -p`) یکی از مسیرهای یکپارچه‌سازی تأییدشده OpenClaw است. احراز هویت با توکن راه‌اندازی Anthropic همچنان پشتیبانی می‌شود، اما OpenClaw در صورت امکان استفاده مجدد از Claude CLI را ترجیح می‌دهد.
</Note>

```json5
{
  agents: { defaults: { model: { primary: "anthropic/claude-opus-5" } } },
}
```

### OAuth مربوط به OpenAI ChatGPT/Codex

- ارائه‌دهنده: `openai`
- احراز هویت: OAuth (ChatGPT)
- ارجاع تازه به هارنس بومی app-server در Codex: `openai/gpt-5.6-sol`
- مستندات هارنس بومی app-server در Codex: [هارنس Codex](/fa/plugins/codex-harness)
- ارجاع‌های مدل قدیمی: `codex/gpt-*`، `openai-codex/gpt-*`
- مرز Plugin: ‏`openai/*`، Plugin مربوط به OpenAI را بارگذاری می‌کند؛ سیاست صریح زمان اجرا یا مسیر مؤثر تحت مالکیت ارائه‌دهنده تعیین می‌کند که آیا Plugin بومی app-server در Codex انتخاب شود یا نه.
- CLI: ‏`openclaw onboard --auth-choice openai` یا `openclaw models auth login --provider openai`
- انتقال تعبیه‌شده ChatGPT Responses در OpenClaw به‌طور پیش‌فرض از `auto` استفاده می‌کند (ابتدا WebSocket و در صورت شکست، SSE).
- `agents.defaults.models["openai/<model>"].params.transport`، `params.serviceTier` و `params.fastMode` تنظیمات تألیف‌شده درخواست تعبیه‌شده هستند. این تنظیمات، انتخاب ضمنی زمان اجرا را در اختیار OpenClaw نگه می‌دارند؛ Codex بومی مالک انتقال app-server و سطح سرویس خود است.
- سرآیندهای پنهان انتساب OpenClaw ‏(`originator`، `version`، `User-Agent`) فقط به ترافیک Codex بومی به مقصد `chatgpt.com/backend-api` پیوست می‌شوند، نه پراکسی‌های عمومی سازگار با OpenAI
- کلید مشترک `/fast` همچنان به‌عنوان کنترل زمان اجرا در دسترس است؛ این کلید با پارامترهای تألیف‌شده مدل تفاوت دارد.
- کاتالوگ بومی Codex می‌تواند بسته به دسترسی حساب، ارجاع‌های دقیق `openai/gpt-5.6-sol`، `openai/gpt-5.6-terra` و `openai/gpt-5.6-luna` را عرضه کند. این کاتالوگ، نام مستعار ساده `gpt-5.6` در API مستقیم را در سمت کلاینت اعمال نمی‌کند.
- `openai/gpt-5.5` از `contextWindow = 400000` بومی کاتالوگ Codex و زمان اجرای پیش‌فرض `contextTokens = 272000` استفاده می‌کند؛ سقف زمان اجرا را با `models.providers.openai.models[].contextTokens` بازنویسی کنید
- با احراز هویت `openai` وارد شوید و برای راه‌اندازی تازه مبتنی بر اشتراک از `openai/gpt-5.6-sol` استفاده کنید. اگر آن فضای کاری Codex، ‏GPT-5.6 را عرضه نمی‌کند، `openai/gpt-5.5` را به‌صراحت انتخاب کنید.
- برای نگه‌داشتن مسیری که از جهات دیگر واجد شرایط است روی زمان اجرای داخلی، از ارائه‌دهنده/مدل `agentRuntime.id: "openclaw"` استفاده کنید. وقتی زمان اجرا تنظیم نشده یا `auto` است، فقط یک مسیر رسمی و دقیق HTTPS سازگار با Responses/ChatGPT که هیچ بازنویسی تألیف‌شده‌ای برای درخواست ندارد، می‌تواند Codex را به‌صورت ضمنی انتخاب کند.
- ارجاع‌های قدیمی Codex GPT حالت قدیمی هستند، نه یک مسیر زنده ارائه‌دهنده. برای پیکربندی عامل جدید از ارجاع‌های متعارف `openai/*` استفاده کنید و برای مهاجرت ارجاع‌های `codex/*` و `openai-codex/*`، درحالی‌که معناشناسی بومی Codex آن‌ها با `agentRuntime.id: "codex"` محدود به مدل حفظ می‌شود، `openclaw doctor --fix` را اجرا کنید. انتخاب‌های صریح و متعارف موجود `openai/gpt-5.5` ارتقا داده نمی‌شوند.

```json5
{
  plugins: { entries: { codex: { enabled: true } } },
  agents: {
    defaults: {
      model: { primary: "openai/gpt-5.6-sol" },
    },
  },
}
```

```json5
{
  models: {
    providers: {
      openai: {
        models: [{ id: "gpt-5.5", contextTokens: 160000 }],
      },
    },
  },
}
```

### سایر گزینه‌های میزبانی‌شده به سبک اشتراکی

<CardGroup cols={3}>
  <Card title="MiniMax" href="/fa/providers/minimax">
    دسترسی با OAuth طرح کدنویسی MiniMax یا کلید API.
  </Card>
  <Card title="Qwen Cloud" href="/fa/providers/qwen">
    سطح ارائه‌دهنده Qwen Cloud به‌همراه نگاشت نقطه پایانی Alibaba DashScope و طرح کدنویسی.
  </Card>
  <Card title="Z.AI (GLM)" href="/fa/providers/zai">
    نقاط پایانی طرح کدنویسی Z.AI یا API عمومی.
  </Card>
</CardGroup>

### OpenCode

- احراز هویت: `OPENCODE_API_KEY` (یا `OPENCODE_ZEN_API_KEY`)
- ارائه‌دهنده زمان اجرای Zen: ‏`opencode`
- ارائه‌دهنده زمان اجرای Go: ‏`opencode-go`
- مدل‌های نمونه: `opencode/claude-opus-4-6`، `opencode-go/kimi-k2.6`
- CLI: ‏`openclaw onboard --auth-choice opencode-zen` یا `openclaw onboard --auth-choice opencode-go`

```json5
{
  agents: { defaults: { model: { primary: "opencode/claude-opus-4-6" } } },
}
```

### Google Gemini (کلید API)

- ارائه‌دهنده: `google`
- احراز هویت: `GEMINI_API_KEY`
- چرخش اختیاری: `GEMINI_API_KEYS`، `GEMINI_API_KEY_1`، `GEMINI_API_KEY_2`، ‏`GOOGLE_API_KEY` به‌عنوان مسیر جایگزین و `OPENCLAW_LIVE_GEMINI_KEY` (بازنویسی تکی)
- مدل‌های نمونه: `google/gemini-3.1-pro-preview`، `google/gemini-3.5-flash`
- سازگاری: پیکربندی قدیمی OpenClaw که از `google/gemini-3.1-flash-preview` استفاده می‌کند به `google/gemini-3-flash-preview` نرمال‌سازی می‌شود
- نام مستعار: `google/gemini-3.1-pro` پذیرفته و به شناسه زنده Gemini API گوگل، یعنی `google/gemini-3.1-pro-preview`، نرمال‌سازی می‌شود
- CLI: ‏`openclaw onboard --auth-choice gemini-api-key`
- تفکر: `/think adaptive` از تفکر پویای گوگل استفاده می‌کند. Gemini ‏3/3.1 یک `thinkingLevel` ثابت را حذف می‌کنند؛ Gemini 2.5، ‏`thinkingBudget: -1` را ارسال می‌کند.
- اجرای مستقیم Gemini همچنین `agents.defaults.models["google/<model>"].params.cachedContent` (یا `cached_content` قدیمی) را برای ارسال یک دسته بومی ارائه‌دهنده به نام `cachedContents/...` می‌پذیرد؛ اصابت‌های کش Gemini به‌صورت `cacheRead` در OpenClaw نمایان می‌شوند

### Google Vertex و Gemini CLI

- ارائه‌دهندگان: `google-vertex`، `google-gemini-cli`
- احراز هویت: Vertex از gcloud ADC استفاده می‌کند؛ Gemini CLI از جریان OAuth خود استفاده می‌کند

<Warning>
OAuth مربوط به Gemini CLI در OpenClaw یک یکپارچه‌سازی غیررسمی است. برخی کاربران پس از استفاده از کلاینت‌های شخص ثالث، محدودیت‌هایی را برای حساب گوگل خود گزارش کرده‌اند. شرایط گوگل را بررسی کنید و اگر ادامه می‌دهید، از حسابی غیرحیاتی استفاده کنید.
</Warning>

OAuth مربوط به Gemini CLI به‌عنوان بخشی از Plugin همراه `google` عرضه می‌شود.

<Steps>
  <Step title="نصب Gemini CLI">
    <Tabs>
      <Tab title="brew">
        ```bash
        brew install gemini-cli
        ```
      </Tab>
      <Tab title="npm">
        ```bash
        npm install -g @google/gemini-cli
        ```
      </Tab>
    </Tabs>
  </Step>
  <Step title="فعال‌سازی Plugin">
    ```bash
    openclaw plugins enable google
    ```
  </Step>
  <Step title="ورود">
    ```bash
    openclaw models auth login --provider google-gemini-cli --set-default
    ```

    مدل پیش‌فرض: `google-gemini-cli/gemini-3-flash-preview`. شناسه کلاینت یا راز را در `openclaw.json` جای‌گذاری **نکنید**. جریان ورود CLI، توکن‌ها را در نمایه‌های احراز هویت روی میزبان Gateway ذخیره می‌کند.

  </Step>
  <Step title="تنظیم پروژه (در صورت نیاز)">
    اگر درخواست‌ها پس از ورود ناموفق بودند، `GOOGLE_CLOUD_PROJECT` یا `GOOGLE_CLOUD_PROJECT_ID` را روی میزبان Gateway تنظیم کنید.
  </Step>
</Steps>

Gemini CLI به‌طور پیش‌فرض از `stream-json` استفاده می‌کند. OpenClaw پیام‌های جریان
دستیار را می‌خواند و `stats.cached` را به `cacheRead` نرمال‌سازی می‌کند؛ بازنویسی‌های قدیمی
`--output-format json` همچنان متن پاسخ را از `response` می‌خوانند.

### Z.AI (GLM)

- ارائه‌دهنده: `zai`
- احراز هویت: `ZAI_API_KEY`
- مدل نمونه: `zai/glm-5.2`
- CLI: ‏`openclaw onboard --auth-choice zai-api-key`
  - ارجاع‌های مدل از شناسه متعارف ارائه‌دهنده `zai/*` استفاده می‌کنند.
  - `zai-api-key` نقطه پایانی منطبق Z.AI را به‌طور خودکار تشخیص می‌دهد؛ `zai-coding-global`، `zai-coding-cn`، `zai-global` و `zai-cn` یک سطح مشخص را اجبار می‌کنند

### Vercel AI Gateway

- ارائه‌دهنده: `vercel-ai-gateway`
- احراز هویت: `AI_GATEWAY_API_KEY`
- مدل‌های نمونه: `vercel-ai-gateway/anthropic/claude-opus-4.6`، `vercel-ai-gateway/moonshotai/kimi-k2.6`
- CLI: ‏`openclaw onboard --auth-choice ai-gateway-api-key`

### سایر Pluginهای همراه ارائه‌دهنده

| ارائه‌دهنده                              | شناسه                            | متغیر محیطی احراز هویت                              | مدل نمونه                                              |
| --------------------------------------- | -------------------------------- | ---------------------------------------------------- | ------------------------------------------------------ |
| Arcee                                   | `arcee`                          | `ARCEEAI_API_KEY` یا `OPENROUTER_API_KEY`            | `arcee/trinity-large-thinking`                         |
| BytePlus                                | `byteplus` / `byteplus-plan`     | `BYTEPLUS_API_KEY`                                   | `byteplus-plan/ark-code-latest`                        |
| Cerebras                                | `cerebras`                       | `CEREBRAS_API_KEY`                                   | `cerebras/zai-glm-4.7`                                 |
| Chutes                                  | `chutes`                         | `CHUTES_API_KEY` یا `CHUTES_OAUTH_TOKEN`             | `chutes/zai-org/GLM-5-TEE`                             |
| ClawRouter                              | `clawrouter`                     | `CLAWROUTER_API_KEY`                                 | `clawrouter/anthropic/claude-sonnet-4-6`               |
| Cohere                                  | `cohere`                         | `COHERE_API_KEY`                                     | `cohere/command-a-plus-05-2026`                        |
| DeepInfra                               | `deepinfra`                      | `DEEPINFRA_API_KEY`                                  | `deepinfra/deepseek-ai/DeepSeek-V4-Flash`              |
| DeepSeek                                | `deepseek`                       | `DEEPSEEK_API_KEY`                                   | `deepseek/deepseek-v4-flash`                           |
| Featherless AI                          | `featherless`                    | `FEATHERLESS_API_KEY`                                | `featherless/Qwen/Qwen3-32B`                           |
| GitHub Copilot                          | `github-copilot`                 | `COPILOT_GITHUB_TOKEN` / `GH_TOKEN` / `GITHUB_TOKEN` | -                                                      |
| GMI Cloud                               | `gmi`                            | `GMI_API_KEY`                                        | `gmi/google/gemini-3.1-flash-lite`                     |
| Groq                                    | `groq`                           | `GROQ_API_KEY`                                       | `groq/llama-3.3-70b-versatile`                         |
| Hugging Face Inference                  | `huggingface`                    | `HUGGINGFACE_HUB_TOKEN` یا `HF_TOKEN`                | `huggingface/deepseek-ai/DeepSeek-R1`                  |
| MiniMax                                 | `minimax` / `minimax-portal`     | `MINIMAX_API_KEY` / `MINIMAX_OAUTH_TOKEN`            | `minimax/MiniMax-M3`                                   |
| Mistral                                 | `mistral`                        | `MISTRAL_API_KEY`                                    | `mistral/mistral-large-latest`                         |
| Moonshot                                | `moonshot`                       | `MOONSHOT_API_KEY`                                   | `moonshot/kimi-k2.6`                                   |
| NVIDIA                                  | `nvidia`                         | `NVIDIA_API_KEY`                                     | `nvidia/nvidia/nemotron-3-ultra-550b-a55b`             |
| NovitaAI                                | `novita`                         | `NOVITA_API_KEY`                                     | `novita/deepseek/deepseek-v3-0324`                     |
| [Ollama Cloud](/fa/providers/ollama-cloud) | `ollama-cloud`                   | `OLLAMA_API_KEY`                                     | `ollama-cloud/kimi-k2.6`                               |
| OpenRouter                              | `openrouter`                     | OpenRouter OAuth یا `OPENROUTER_API_KEY`             | `openrouter/auto`                                      |
| Qianfan                                 | `qianfan`                        | `QIANFAN_API_KEY`                                    | `qianfan/deepseek-v3.2`                                |
| Tencent TokenHub                        | `tencent-tokenhub`               | `TOKENHUB_API_KEY`                                   | `tencent-tokenhub/hy3-preview`                         |
| Together                                | `together`                       | `TOGETHER_API_KEY`                                   | `together/meta-llama/Llama-3.3-70B-Instruct-Turbo`     |
| Venice                                  | `venice`                         | `VENICE_API_KEY`                                     | -                                                      |
| Vercel AI Gateway                       | `vercel-ai-gateway`              | `AI_GATEWAY_API_KEY`                                 | `vercel-ai-gateway/anthropic/claude-opus-4.6`          |
| Volcano Engine (Doubao)                 | `volcengine` / `volcengine-plan` | `VOLCANO_ENGINE_API_KEY`                             | `volcengine-plan/ark-code-latest`                      |
| xAI                                     | `xai`                            | SuperGrok/X Premium OAuth یا `XAI_API_KEY`           | `xai/grok-4.3`                                         |
| Xiaomi                                  | `xiaomi` / `xiaomi-token-plan`   | `XIAOMI_API_KEY` / `XIAOMI_TOKEN_PLAN_API_KEY`       | `xiaomi/mimo-v2.5` / `xiaomi-token-plan/mimo-v2.5-pro` |

#### نکاتی که دانستنشان مفید است

<AccordionGroup>
  <Accordion title="OpenRouter">
    سرآیندهای انتساب برنامه و نشانگرهای Anthropic `cache_control` را فقط در مسیرهای تأییدشدهٔ `openrouter.ai` اعمال می‌کند. ارجاع‌های DeepSeek، Moonshot و ZAI برای کش‌کردن پرامپت تحت مدیریت OpenRouter واجد شرایط TTL کش هستند، اما نشانگرهای کش Anthropic را دریافت نمی‌کنند. این مسیر به‌عنوان مسیری پراکسی‌مانند و سازگار با OpenAI، شکل‌دهی‌های مختص OpenAI بومی (`serviceTier`، Responses `store`، راهنمایی‌های کش پرامپت و سازگاری استدلال OpenAI) را نادیده می‌گیرد. ارجاع‌های مبتنی بر Gemini فقط پاک‌سازی امضای تفکر proxy-Gemini را حفظ می‌کنند.
  </Accordion>
  <Accordion title="Kilo Gateway">
    ارجاع‌های مبتنی بر Gemini از همان مسیر پاک‌سازی proxy-Gemini پیروی می‌کنند؛ `kilocode/kilo-auto/balanced` و دیگر ارجاع‌هایی که از استدلال پراکسی پشتیبانی نمی‌کنند، تزریق استدلال پراکسی را نادیده می‌گیرند.
  </Accordion>
  <Accordion title="MiniMax">
    راه‌اندازی اولیه با کلید API، تعریف‌های صریح مدل چت M3 و M2.7 را می‌نویسد؛ درک تصویر همچنان بر عهدهٔ ارائه‌دهندهٔ رسانهٔ `MiniMax-VL-01` متعلق به Plugin است.
  </Accordion>
  <Accordion title="NVIDIA">
    شناسه‌های مدل از فضای نام `nvidia/<vendor>/<model>` استفاده می‌کنند (برای مثال `nvidia/nvidia/nemotron-...`)؛ انتخاب‌گرها ترکیب تحت‌اللفظی `<provider>/<model-id>` را حفظ می‌کنند، درحالی‌که کلید معیار ارسال‌شده به API تنها یک پیشوند دارد.
  </Accordion>
  <Accordion title="xAI">
    از مسیر Responses در xAI استفاده می‌کند. مسیر توصیه‌شده SuperGrok/X Premium OAuth است؛ کلیدهای API همچنان از طریق `XAI_API_KEY` یا پیکربندی Plugin کار می‌کنند و Grok `web_search` پیش از بازگشت به کلید API، از همان نمایهٔ احراز هویت استفاده می‌کند. Grok 4.5 در صورت دسترس‌بودن برای چت، کدنویسی و کارهای عامل‌محور قابل انتخاب است؛ `grok-4.3` همچنان پیش‌فرض همراه و ایمن برای مناطق مختلف است. پیکربندی‌های قدیمی‌تر `/fast` و `params.fastMode: true` همچنان از طریق تغییرمسیرهای سازگاری Grok 4.3 در xAI حل می‌شوند، اما پیکربندی‌های جدید باید مستقیماً یک مدل کنونی را انتخاب کنند. `tool_stream` به‌طور پیش‌فرض فعال است؛ آن را از طریق `agents.defaults.models["xai/<model>"].params.tool_stream=false` غیرفعال کنید.
  </Accordion>
</AccordionGroup>

## ارائه‌دهندگان از طریق `models.providers` (نشانی پایه/سفارشی)

برای افزودن ارائه‌دهندگان **سفارشی** یا پراکسی‌های سازگار با OpenAI/Anthropic، از `models.providers` (یا `models.json`) استفاده کنید.

بسیاری از Pluginهای همراه ارائه‌دهندگان در ادامه، از قبل یک کاتالوگ پیش‌فرض منتشر می‌کنند. تنها زمانی از ورودی‌های صریح `models.providers.<id>` استفاده کنید که می‌خواهید نشانی پایه، سرآیندها یا فهرست مدل پیش‌فرض را بازنویسی کنید.

مسیرهای همراه و شناخته‌شده در کاتالوگ، قابلیت‌های `compat` خود را از Plugin ارائه‌دهندهٔ مالک دریافت می‌کنند. بلوک پیکربندی `compat` برای یک ارائه‌دهنده/مدل سفارشی یا مسیر متفاوت `api`/`baseUrl` است که قرارداد نقطهٔ پایانی آن را تأیید کرده‌اید؛ [راهنمای قابلیت‌های ارائه‌دهندهٔ سفارشی](/fa/gateway/config-tools#custom-provider-capability-declarations) را ببینید. Doctor مقادیر قدیمی‌ای را که صرفاً کاتالوگ را تکرار می‌کنند حذف می‌کند و مقادیر متفاوت را برای بازبینی اپراتور قابل مشاهده نگه می‌دارد.

بررسی قابلیت‌های مدل در Gateway، فرادادهٔ صریح `models.providers.<id>.models[]` را نیز می‌خواند. اگر یک مدل سفارشی یا پراکسی تصاویر را می‌پذیرد، `input: ["text", "image"]` را روی آن مدل تنظیم کنید تا مسیرهای پیوست WebChat و منشأگرفته از Node، تصاویر را به‌جای ارجاع‌های رسانه‌ای فقط‌متنی، به‌عنوان ورودی‌های بومی مدل ارسال کنند.

`agents.defaults.models["provider/model"]` نام‌های مستعار و فرادادهٔ مختص هر مدل را برای عامل‌ها کنترل می‌کند. این گزینه نه بازنویسی‌ها را محدود می‌کند و نه به‌تنهایی یک مدل زمان‌اجرای جدید ثبت می‌کند. برای مدل‌های ارائه‌دهندهٔ سفارشی، `models.providers.<provider>.models[]` را نیز دست‌کم با `id` منطبق اضافه کنید؛ وقتی می‌خواهید بازنویسی را محدود کنید، به‌طور جداگانه از `agents.defaults.modelPolicy.allow` استفاده کنید.

### Moonshot AI (Kimi)

پیش از راه‌اندازی اولیه، `@openclaw/moonshot-provider` را نصب کنید. تنها زمانی یک ورودی صریح `models.providers.moonshot` اضافه کنید که لازم است نشانی پایه یا فرادادهٔ مدل را بازنویسی کنید:

- ارائه‌دهنده: `moonshot`
- احراز هویت: `MOONSHOT_API_KEY`
- مدل نمونه: `moonshot/kimi-k3`
- CLI: `openclaw onboard --auth-choice moonshot-api-key` یا `openclaw onboard --auth-choice moonshot-api-key-cn`

شناسه‌های مدل Kimi:

[//]: # "moonshot-kimi-k2-model-refs:start"

- `moonshot/kimi-k2.6`
- `moonshot/kimi-k3`
- `moonshot/kimi-k2.7-code`
- `moonshot/kimi-k2.7-code-highspeed`
- `moonshot/kimi-k2.5`

[//]: # "moonshot-kimi-k2-model-refs:end"

```json5
{
  agents: {
    defaults: { model: { primary: "moonshot/kimi-k2.6" } },
  },
  models: {
    mode: "merge",
    providers: {
      moonshot: {
        baseUrl: "https://api.moonshot.ai/v1",
        apiKey: "${MOONSHOT_API_KEY}",
        api: "openai-completions",
        models: [{ id: "kimi-k2.6", name: "Kimi K2.6" }],
      },
    },
  },
}
```

برای راهنمای کامل راه‌اندازی، [Moonshot AI (Kimi + Kimi Coding)](/fa/providers/moonshot) را ببینید.

### Kimi Coding

Kimi Coding از نقطهٔ پایانی سازگار با Anthropic در Moonshot AI استفاده می‌کند:

- ارائه‌دهنده: `kimi`
- احراز هویت: `KIMI_API_KEY`
- Kimi K3: `kimi/k3` (256K) یا `kimi/k3[1m]` (طرح 1M)
- Kimi Code: `kimi/kimi-for-coding`
- Kimi Code HighSpeed: `kimi/kimi-for-coding-highspeed`

```json5
{
  env: { KIMI_API_KEY: "sk-..." },
  agents: {
    defaults: { model: { primary: "kimi/kimi-for-coding" } },
  },
}
```

`kimi/kimi-code` و `kimi/k2p5` قدیمی همچنان به‌عنوان شناسه‌های مدل سازگاری پذیرفته می‌شوند و به شناسهٔ پایدار مدل API در Kimi نرمال‌سازی می‌شوند.

### Volcano Engine (Doubao)

Volcano Engine (火山引擎) دسترسی به Doubao و مدل‌های دیگر را در چین فراهم می‌کند.

- ارائه‌دهنده: `volcengine` (کدنویسی: `volcengine-plan`)
- احراز هویت: `VOLCANO_ENGINE_API_KEY`
- مدل نمونه: `volcengine-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice volcengine-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "volcengine-plan/ark-code-latest" } },
  },
}
```

راه‌اندازی اولیه به‌طور پیش‌فرض از سطح کدنویسی استفاده می‌کند، اما کاتالوگ عمومی `volcengine/*` هم‌زمان ثبت می‌شود.

در انتخاب‌گرهای مدل راه‌اندازی اولیه/پیکربندی، گزینهٔ احراز هویت Volcengine هر دو ردیف `volcengine/*` و `volcengine-plan/*` را ترجیح می‌دهد. اگر این مدل‌ها هنوز بارگذاری نشده باشند، OpenClaw به‌جای نمایش یک انتخاب‌گر خالی با دامنهٔ ارائه‌دهنده، به کاتالوگ فیلترنشده بازمی‌گردد.

<Tabs>
  <Tab title="مدل‌های استاندارد">
    - `volcengine/doubao-seed-1-8-251228` (Doubao Seed 1.8)
    - `volcengine/doubao-seed-code-preview-251028`
    - `volcengine/kimi-k2-5-260127` (Kimi K2.5)
    - `volcengine/glm-4-7-251222` (GLM 4.7)
    - `volcengine/deepseek-v3-2-251201` (DeepSeek V3.2)

  </Tab>
  <Tab title="مدل‌های کدنویسی (volcengine-plan)">
    - `volcengine-plan/ark-code-latest`
    - `volcengine-plan/doubao-seed-code`

  </Tab>
</Tabs>

### BytePlus (بین‌المللی)

BytePlus ARK دسترسی کاربران بین‌المللی به همان مدل‌های Volcano Engine را فراهم می‌کند.

- ارائه‌دهنده: `byteplus` (کدنویسی: `byteplus-plan`)
- احراز هویت: `BYTEPLUS_API_KEY`
- مدل نمونه: `byteplus-plan/ark-code-latest`
- CLI: `openclaw onboard --auth-choice byteplus-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "byteplus-plan/ark-code-latest" } },
  },
}
```

فرایند راه‌اندازی اولیه به‌طور پیش‌فرض از سطح کدنویسی استفاده می‌کند، اما کاتالوگ عمومی `byteplus/*` نیز هم‌زمان ثبت می‌شود.

در انتخاب‌گرهای مدلِ راه‌اندازی اولیه/پیکربندی، گزینه احراز هویت BytePlus هر دو ردیف `byteplus/*` و `byteplus-plan/*` را ترجیح می‌دهد. اگر این مدل‌ها هنوز بارگیری نشده باشند، OpenClaw به‌جای نمایش انتخاب‌گر خالیِ محدود به ارائه‌دهنده، به کاتالوگ فیلترنشده بازمی‌گردد.

<Tabs>
  <Tab title="مدل‌های استاندارد">
    - `byteplus/seed-1-8-251228` (Seed 1.8)
    - `byteplus/kimi-k2-5-260127` (Kimi K2.5)
    - `byteplus/glm-4-7-251222` (GLM 4.7)

  </Tab>
  <Tab title="مدل‌های کدنویسی (byteplus-plan)">
    - `byteplus-plan/ark-code-latest`
    - `byteplus-plan/kimi-k2.5`
    - `byteplus-plan/glm-4.7`

  </Tab>
</Tabs>

### Synthetic

Synthetic مدل‌های سازگار با Anthropic را از طریق ارائه‌دهنده `synthetic` فراهم می‌کند:

- ارائه‌دهنده: `synthetic`
- احراز هویت: `SYNTHETIC_API_KEY`
- مدل نمونه: `synthetic/hf:MiniMaxAI/MiniMax-M3`
- CLI: `openclaw onboard --auth-choice synthetic-api-key`

```json5
{
  agents: {
    defaults: { model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" } },
  },
  models: {
    mode: "merge",
    providers: {
      synthetic: {
        baseUrl: "https://api.synthetic.new/anthropic",
        apiKey: "${SYNTHETIC_API_KEY}",
        api: "anthropic-messages",
        models: [{ id: "hf:MiniMaxAI/MiniMax-M3", name: "MiniMax M3" }],
      },
    },
  },
}
```

### MiniMax

MiniMax از طریق `models.providers` پیکربندی می‌شود، زیرا از نقاط پایانی سفارشی استفاده می‌کند:

- OAuth‏ MiniMax (جهانی): `--auth-choice minimax-global-oauth`
- OAuth‏ MiniMax (چین): `--auth-choice minimax-cn-oauth`
- کلید API‏ MiniMax (جهانی): `--auth-choice minimax-global-api`
- کلید API‏ MiniMax (چین): `--auth-choice minimax-cn-api`
- احراز هویت: `MINIMAX_API_KEY` برای `minimax`؛ `MINIMAX_OAUTH_TOKEN` یا `MINIMAX_API_KEY` برای `minimax-portal`

برای جزئیات راه‌اندازی، گزینه‌های مدل و قطعه‌پیکربندی‌ها، به [/providers/minimax](/fa/providers/minimax) مراجعه کنید.

<Note>
در مسیر استریم سازگار با Anthropic در MiniMax، ‏OpenClaw قابلیت تفکر را به‌طور پیش‌فرض برای خانواده M2.x غیرفعال می‌کند، مگر اینکه آن را صریحاً تنظیم کنید؛ MiniMax-M3 (و M3.x) به‌طور پیش‌فرض در مسیر تفکر حذف‌شده/تطبیقی ارائه‌دهنده باقی می‌ماند. `/fast on` مقدار `MiniMax-M2.7` را به `MiniMax-M2.7-highspeed` بازنویسی می‌کند.
</Note>

تفکیک قابلیت‌های تحت مالکیت Plugin:

- پیش‌فرض‌های متن/گفت‌وگو روی `minimax/MiniMax-M3` باقی می‌مانند
- تولید تصویر `minimax/image-01` یا `minimax-portal/image-01` است
- درک تصویر در هر دو مسیر احراز هویت MiniMax تحت مالکیت Plugin و برابر با `MiniMax-VL-01` است
- جست‌وجوی وب روی شناسه ارائه‌دهنده `minimax` باقی می‌ماند

### LM Studio

LM Studio به‌صورت یک Plugin ارائه‌دهنده همراه عرضه می‌شود که از API بومی استفاده می‌کند:

- ارائه‌دهنده: `lmstudio`
- احراز هویت: `LM_API_TOKEN`
- نشانی پایه پیش‌فرض استنتاج: `http://localhost:1234/v1`

سپس یک مدل تنظیم کنید (آن را با یکی از شناسه‌های بازگردانده‌شده توسط `http://localhost:1234/api/v1/models` جایگزین کنید):

```json5
{
  agents: {
    defaults: { model: { primary: "lmstudio/openai/gpt-oss-20b" } },
  },
}
```

OpenClaw برای کشف و بارگیری خودکار از `/api/v1/models` و `/api/v1/models/load` بومی LM Studio استفاده می‌کند و به‌طور پیش‌فرض از `/v1/chat/completions` برای استنتاج بهره می‌برد. اگر می‌خواهید بارگیری JIT‏، TTL و حذف خودکار LM Studio چرخه عمر مدل را مدیریت کنند، `models.providers.lmstudio.params.preload: false` را تنظیم کنید. برای راه‌اندازی و عیب‌یابی به [/providers/lmstudio](/fa/providers/lmstudio) مراجعه کنید.

### Ollama

Ollama به‌صورت یک Plugin ارائه‌دهنده همراه عرضه می‌شود و از API بومی Ollama استفاده می‌کند:

- ارائه‌دهنده: `ollama`
- احراز هویت: لازم نیست (سرور محلی)
- مدل نمونه: `ollama/llama3.3`
- نصب: [https://ollama.com/download](https://ollama.com/download)

```bash
# Ollama را نصب کنید، سپس یک مدل دریافت کنید:
ollama pull llama3.3
```

```json5
{
  agents: {
    defaults: { model: { primary: "ollama/llama3.3" } },
  },
}
```

وقتی با `OLLAMA_API_KEY` آن را فعال کنید، Ollama به‌صورت محلی در `http://127.0.0.1:11434` شناسایی می‌شود و Plugin ارائه‌دهنده همراه، Ollama را مستقیماً به `openclaw onboard` و انتخاب‌گر مدل اضافه می‌کند. برای راه‌اندازی اولیه، حالت ابری/محلی و پیکربندی سفارشی به [/providers/ollama](/fa/providers/ollama) مراجعه کنید.

### vLLM

vLLM به‌صورت یک Plugin ارائه‌دهنده همراه برای سرورهای محلی/خودمیزبانِ سازگار با OpenAI عرضه می‌شود:

- ارائه‌دهنده: `vllm`
- احراز هویت: اختیاری (بسته به سرور شما)
- نشانی پایه پیش‌فرض: `http://127.0.0.1:8000/v1`

برای فعال‌سازی کشف خودکار به‌صورت محلی (اگر سرور شما احراز هویت را الزامی نمی‌کند، هر مقداری قابل استفاده است):

```bash
export VLLM_API_KEY="vllm-local"
```

سپس یک مدل تنظیم کنید (آن را با یکی از شناسه‌های بازگردانده‌شده توسط `/v1/models` جایگزین کنید):

```json5
{
  agents: {
    defaults: { model: { primary: "vllm/your-model-id" } },
  },
}
```

برای جزئیات به [/providers/vllm](/fa/providers/vllm) مراجعه کنید.

### SGLang

SGLang به‌صورت یک Plugin ارائه‌دهنده همراه برای سرورهای سریعِ خودمیزبان و سازگار با OpenAI عرضه می‌شود:

- ارائه‌دهنده: `sglang`
- احراز هویت: اختیاری (بسته به سرور شما)
- نشانی پایه پیش‌فرض: `http://127.0.0.1:30000/v1`

برای فعال‌سازی کشف خودکار به‌صورت محلی (اگر سرور شما احراز هویت را الزامی نمی‌کند، هر مقداری قابل استفاده است):

```bash
export SGLANG_API_KEY="sglang-local"
```

سپس یک مدل تنظیم کنید (آن را با یکی از شناسه‌های بازگردانده‌شده توسط `/v1/models` جایگزین کنید):

```json5
{
  agents: {
    defaults: { model: { primary: "sglang/your-model-id" } },
  },
}
```

برای جزئیات به [/providers/sglang](/fa/providers/sglang) مراجعه کنید.

### پراکسی‌های محلی (LM Studio، ‏vLLM، ‏LiteLLM و غیره)

نمونه (سازگار با OpenAI):

```json5
{
  agents: {
    defaults: {
      model: { primary: "lmstudio/my-local-model" },
      models: { "lmstudio/my-local-model": { alias: "Local" } },
    },
  },
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "${LM_API_TOKEN}",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "my-local-model",
            name: "Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 200000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="فیلدهای اختیاری پیش‌فرض">
    برای ارائه‌دهندگان سفارشی، `reasoning`، `input`، `cost`، `contextWindow` و `maxTokens` اختیاری هستند. در صورت حذف، OpenClaw از پیش‌فرض‌های زیر استفاده می‌کند:

    - `reasoning: false`
    - `input: ["text"]`
    - `cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 }`
    - `contextWindow: 200000`
    - `maxTokens: 8192`

    توصیه می‌شود: مقادیر صریحی مطابق با محدودیت‌های پراکسی/مدل خود تنظیم کنید.

  </Accordion>
  <Accordion title="قواعد شکل‌دهی مسیر پراکسی">
    - برای `api: "openai-completions"` روی نقاط پایانی غیربومی (هر `baseUrl` غیرخالی که میزبان آن `api.openai.com` نباشد)، OpenClaw برای جلوگیری از خطاهای 400 ارائه‌دهنده در نقش‌های پشتیبانی‌نشده `developer`، مقدار `compat.supportsDeveloperRole: false` را اجباری می‌کند.
    - مسیرهای پراکسی‌مانندِ سازگار با OpenAI نیز شکل‌دهی درخواست مختص OpenAI بومی را نادیده می‌گیرند: بدون `service_tier`، بدون `store` در Responses، بدون `store` در Completions، بدون راهنمای کش پرامپت، بدون شکل‌دهی بار سازگاری استدلال OpenAI و بدون سرآیندهای پنهان انتساب OpenClaw.
    - برای پراکسی‌های Completions سازگار با OpenAI که به فیلدهای مختص فروشنده نیاز دارند، `agents.defaults.models["provider/model"].params.extra_body` (یا `extraBody`) را تنظیم کنید تا JSON اضافی در بدنه درخواست خروجی ادغام شود.
    - برای کنترل‌های الگوی گفت‌وگوی vLLM، ‏`agents.defaults.models["provider/model"].params.chat_template_kwargs` را تنظیم کنید. Plugin همراه vLLM وقتی سطح تفکر نشست خاموش است، به‌طور خودکار `enable_thinking: false` و `force_nonempty_content: true` را برای `vllm/nemotron-3-*` ارسال می‌کند.
    - برای مدل‌های محلی کند یا میزبان‌های راه دور LAN/tailnet، ‏`models.providers.<id>.timeoutSeconds` را تنظیم کنید. این کار مدیریت درخواست HTTP مدل ارائه‌دهنده، شامل اتصال، سرآیندها، استریم بدنه و لغو کلی guarded-fetch را گسترش می‌دهد، بدون اینکه مهلت زمانی کل زمان اجرای عامل را افزایش دهد. اگر `agents.defaults.timeoutSeconds` یا مهلت زمانی مختص یک اجرا کمتر است، آن سقف را نیز افزایش دهید؛ مهلت‌های زمانی ارائه‌دهنده نمی‌توانند کل اجرا را تمدید کنند.
    - فراخوانی‌های HTTP ارائه‌دهنده مدل، پاسخ‌های DNS‏ fake-IP مربوط به Surge، ‏Clash و sing-box را در `198.18.0.0/15` و `fc00::/7` فقط برای نام میزبان `baseUrl` ارائه‌دهنده پیکربندی‌شده مجاز می‌کنند. نقاط پایانی ارائه‌دهنده سفارشی/محلی نیز برای درخواست‌های محافظت‌شده مدل، دقیقاً به مبدأ `scheme://host:port` پیکربندی‌شده، شامل میزبان‌های loopback، ‏LAN و tailnet، اعتماد می‌کنند. این یک گزینه پیکربندی جدید نیست؛ `baseUrl` که پیکربندی می‌کنید، سیاست درخواست را فقط برای همان مبدأ گسترش می‌دهد. مجازسازی نام میزبان fake-IP و اعتماد به مبدأ دقیق، سازوکارهایی مستقل هستند. سایر مقصدهای خصوصی، loopback، ‏link-local، فراداده و درگاه‌های متفاوت همچنان به فعال‌سازی صریح `models.providers.<id>.request.allowPrivateNetwork: true` نیاز دارند. برای انصراف از اعتماد به مبدأ دقیق، `models.providers.<id>.request.allowPrivateNetwork: false` را تنظیم کنید.
    - اگر `baseUrl` خالی یا حذف‌شده باشد، OpenClaw رفتار پیش‌فرض OpenAI را حفظ می‌کند (که به `api.openai.com` منتهی می‌شود).
    - برای ایمنی، `compat.supportsDeveloperRole: true` صریح همچنان در نقاط پایانی غیربومی `openai-completions` بازنویسی می‌شود.
    - برای `api: "anthropic-messages"` روی نقاط پایانی غیرمستقیم (هر ارائه‌دهنده‌ای جز `anthropic` متعارف، یا `models.providers.anthropic.baseUrl` سفارشی که میزبان آن یک نقطه پایانی عمومی `api.anthropic.com` نباشد)، OpenClaw سرآیندهای بتای ضمنی Anthropic مانند `claude-code-20250219`، `interleaved-thinking-2025-05-14` و نشانگرهای OAuth را سرکوب می‌کند تا پراکسی‌های سفارشی سازگار با Anthropic پرچم‌های بتای پشتیبانی‌نشده را رد نکنند. اگر پراکسی شما به قابلیت‌های بتای مشخصی نیاز دارد، `models.providers.<id>.headers["anthropic-beta"]` را صریحاً تنظیم کنید.

  </Accordion>
</AccordionGroup>

## نمونه‌های CLI

```bash
openclaw onboard --auth-choice opencode-zen
openclaw models set opencode/claude-opus-4-6
openclaw models list
```

همچنین ببینید: [پیکربندی](/fa/gateway/configuration) برای نمونه‌های کامل پیکربندی.

## مرتبط

- [مرجع پیکربندی](/fa/gateway/config-agents#agent-defaults) - کلیدهای پیکربندی مدل
- [جابه‌جایی خودکار مدل](/fa/concepts/model-failover) - زنجیره‌های جایگزین و رفتار تلاش مجدد
- [مدل‌ها](/fa/concepts/models) - پیکربندی مدل و نام‌های مستعار
- [ارائه‌دهندگان](/fa/providers) - راهنماهای راه‌اندازی هر ارائه‌دهنده
