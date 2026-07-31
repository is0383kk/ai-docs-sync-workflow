---
read_when:
    - تنظیم پیش‌فرض‌های عامل (مدل‌ها، تفکر، فضای کاری، Heartbeat، رسانه، Skills)
    - پیکربندی مسیریابی و پیوندهای چندعاملی
    - تنظیم رفتار نشست، تحویل پیام و حالت مکالمه
summary: پیش‌فرض‌های عامل، مسیریابی چندعاملی، نشست، پیام‌ها و پیکربندی گفت‌وگو
title: پیکربندی — عامل‌ها
x-i18n:
    generated_at: "2026-07-27T16:27:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7a161d65b02e3333c15a2d998421419ee37d36be4d02ebb3a86e66282df06adb
    source_path: gateway/config-agents.md
    workflow: 16
---

کلیدهای پیکربندی مختص عامل در زیر `agents.*`، `multiAgent.*`، `session.*`،
`messages.*` و `talk.*`. برای کانال‌ها، ابزارها، زمان اجرای Gateway و دیگر
کلیدهای سطح‌بالا، به [مرجع پیکربندی](/fa/gateway/configuration-reference) مراجعه کنید.

## پیش‌فرض‌های عامل

### `agents.defaults.workspace`

پیش‌فرض: در صورت تنظیم‌بودن `OPENCLAW_WORKSPACE_DIR`، در غیر این صورت `~/.openclaw/workspace` (یا هنگامی که `OPENCLAW_PROFILE` روی نمایه‌ای غیراز پیش‌فرض تنظیم شده باشد، `~/.openclaw/workspace-<profile>`).

```json5
{
  agents: { defaults: { workspace: "~/.openclaw/workspace" } },
}
```

مقدار صریح `agents.defaults.workspace` بر
`OPENCLAW_WORKSPACE_DIR` اولویت دارد. هنگامی که نمی‌خواهید مسیر را در پیکربندی بنویسید، از متغیر محیطی استفاده کنید تا عامل‌های پیش‌فرض
را به یک فضای کاری سوارشده هدایت کنید.

### `agents.defaults.repoRoot`

ریشه اختیاری مخزن که در خط Runtime اعلان سیستم نمایش داده می‌شود. اگر تنظیم نشده باشد، OpenClaw با پیمایش رو به بالا از فضای کاری، آن را به‌طور خودکار شناسایی می‌کند.

```json5
{
  agents: { defaults: { repoRoot: "~/Projects/openclaw" } },
}
```

### `agents.defaults.skills`

فهرست مجاز پیش‌فرض و اختیاری Skills برای عامل‌هایی که
`agents.entries.*.skills` را تنظیم نمی‌کنند.

```json5
{
  agents: {
    defaults: { skills: ["github", "weather"] },
    list: [
      { id: "writer" }, // github و weather را به ارث می‌برد
      { id: "docs", skills: ["docs-search"] }, // جایگزین پیش‌فرض‌ها می‌شود
      { id: "locked-down", skills: [] }, // بدون Skills
    ],
  },
}
```

- برای نامحدودبودن Skills به‌صورت پیش‌فرض، `agents.defaults.skills` را حذف کنید.
- برای به‌ارث‌بردن پیش‌فرض‌ها، `agents.entries.*.skills` را حذف کنید.
- برای نداشتن هیچ Skills، `agents.entries.*.skills: []` را تنظیم کنید.
- یک فهرست غیرخالی `agents.entries.*.skills`، مجموعه نهایی آن عامل است؛ این فهرست
  با پیش‌فرض‌ها ادغام نمی‌شود.

### `agents.defaults.skipBootstrap`

ایجاد خودکار فایل‌های راه‌اندازی فضای کاری (`AGENTS.md`، `SOUL.md`، `TOOLS.md`، `IDENTITY.md`، `USER.md`، `BOOTSTRAP.md`) را غیرفعال می‌کند.

```json5
{
  agents: { defaults: { skipBootstrap: true } },
}
```

### `agents.defaults.skipOptionalBootstrapFiles`

از ایجاد فایل‌های اختیاری انتخاب‌شده فضای کاری صرف‌نظر می‌کند، در حالی که فایل‌های راه‌اندازی الزامی (`AGENTS.md`، `TOOLS.md`، `BOOTSTRAP.md`) همچنان نوشته می‌شوند. مقادیر معتبر: `SOUL.md`، `USER.md` و `IDENTITY.md` هستند (`HEARTBEAT.md` پذیرفته می‌شود، اما چون زمینه Heartbeat به فضای موقت پایشگر Cron منتقل شده، هیچ اثری ندارد).

```json5
{
  agents: {
    defaults: {
      skipOptionalBootstrapFiles: ["SOUL.md", "USER.md"],
    },
  },
}
```

### `agents.defaults.contextInjection`

زمان تزریق فایل‌های راه‌اندازی فضای کاری به اعلان سیستم را کنترل می‌کند. پیش‌فرض: `"always"`.

- `"continuation-skip"`: نوبت‌های ادامه امن (پس از پاسخ تکمیل‌شده دستیار) از تزریق مجدد راه‌اندازی فضای کاری صرف‌نظر می‌کنند و اندازه اعلان را کاهش می‌دهند. اجراهای Heartbeat و تلاش‌های مجدد پس از Compaction همچنان زمینه را بازسازی می‌کنند.
- `"never"`: تزریق راه‌اندازی فضای کاری و فایل‌های زمینه را در همه نوبت‌ها غیرفعال می‌کند. این گزینه را فقط برای عامل‌هایی استفاده کنید که چرخه عمر اعلان خود را کاملاً مدیریت می‌کنند (موتورهای زمینه سفارشی، زمان‌های اجرای بومی که زمینه خود را می‌سازند، یا گردش‌کارهای تخصصی بدون راه‌اندازی). نوبت‌های Heartbeat و بازیابی از Compaction نیز از تزریق صرف‌نظر می‌کنند.

```json5
{
  agents: { defaults: { contextInjection: "continuation-skip" } },
}
```

بازنویسی مختص هر عامل: `agents.entries.*.contextInjection`. مقادیر حذف‌شده
`agents.defaults.contextInjection` را به ارث می‌برند.

### `agents.defaults.bootstrapMaxChars`

حداکثر تعداد نویسه برای هر فایل راه‌اندازی فضای کاری پیش از کوتاه‌سازی. پیش‌فرض: `20000`.

```json5
{
  agents: { defaults: { bootstrapMaxChars: 20000 } },
}
```

بازنویسی مختص هر عامل: `agents.entries.*.bootstrapMaxChars`. مقادیر حذف‌شده
`agents.defaults.bootstrapMaxChars` را به ارث می‌برند.

### `agents.defaults.bootstrapTotalMaxChars`

حداکثر مجموع نویسه‌های تزریق‌شده از همه فایل‌های راه‌اندازی فضای کاری. پیش‌فرض: `60000`.

```json5
{
  agents: { defaults: { bootstrapTotalMaxChars: 60000 } },
}
```

بازنویسی مختص هر عامل: `agents.entries.*.bootstrapTotalMaxChars`. مقادیر حذف‌شده
`agents.defaults.bootstrapTotalMaxChars` را به ارث می‌برند.

### بازنویسی‌های نمایه راه‌اندازی مختص هر عامل

هنگامی که یک عامل به رفتار تزریق اعلان متفاوتی نسبت به پیش‌فرض‌های مشترک نیاز دارد، از بازنویسی‌های نمایه راه‌اندازی مختص هر عامل استفاده کنید. فیلدهای حذف‌شده از
`agents.defaults` به ارث می‌رسند.

```json5
{
  agents: {
    defaults: {
      contextInjection: "continuation-skip",
      bootstrapMaxChars: 20000,
      bootstrapTotalMaxChars: 60000,
    },
    list: [
      {
        id: "strict-worker",
        contextInjection: "always",
        bootstrapMaxChars: 50000,
        bootstrapTotalMaxChars: 300000,
      },
    ],
  },
}
```

### `agents.defaults.bootstrapPromptTruncationWarning`

اعلان قابل‌مشاهده برای عامل را در اعلان سیستم، هنگام کوتاه‌شدن زمینه راه‌اندازی کنترل می‌کند.
پیش‌فرض: `"always"`.

- `"off"`: هرگز متن اعلان کوتاه‌سازی را به اعلان سیستم تزریق نکن.
- `"once"`: برای هر امضای کوتاه‌سازی منحصربه‌فرد، یک‌بار یک اعلان مختصر تزریق کن.
- `"always"`: هر زمان کوتاه‌سازی وجود دارد، در هر اجرا یک اعلان مختصر تزریق کن (توصیه‌شده).

تعدادهای خام/تزریق‌شده تفصیلی و فیلدهای تنظیم پیکربندی در بخش‌های تشخیصی مانند
گزارش‌های زمینه/وضعیت و گزارش‌های ثبت‌شده باقی می‌مانند؛ زمینه معمول کاربر/زمان اجرای WebChat فقط
اعلان مختصر بازیابی را دریافت می‌کند.

```json5
{
  agents: { defaults: { bootstrapPromptTruncationWarning: "always" } }, // off | once | always
}
```

### نقشه مالکیت بودجه زمینه

OpenClaw چندین بودجه پرحجم اعلان/زمینه دارد که عمداً
به‌جای عبور همگی از یک گزینه عمومی، بر اساس زیرسامانه تفکیک شده‌اند.

| بودجه                                                         | پوشش                                                                                                                                                          |
| -------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agents.defaults.bootstrapMaxChars` / `bootstrapTotalMaxChars` | تزریق عادی راه‌اندازی فضای کاری                                                                                                                            |
| `agents.defaults.startupContext.*`                             | پیش‌درآمد یک‌باره اجرای مدل در بازنشانی/راه‌اندازی، شامل فایل‌های روزانه اخیر `memory/*.md`. فرمان‌های ساده گفت‌وگو `/new` و `/reset` بدون فراخوانی مدل تأیید می‌شوند |
| `skills.limits.*`                                              | فهرست فشرده Skills که به اعلان سیستم تزریق می‌شود                                                                                                         |
| `agents.defaults.contextLimits.*`                              | گزیده‌های محدودشده زمان اجرا و بلوک‌های تزریق‌شده تحت مالکیت زمان اجرا                                                                                                      |
| `memory.qmd.limits.*`                                          | اندازه‌گذاری قطعه جست‌وجوی حافظه نمایه‌شده و تزریق                                                                                                              |

بازنویسی‌های متناظر مختص هر عامل:

- `agents.entries.*.skillsLimits.maxSkillsPromptChars`
- `agents.entries.*.contextInjection`
- `agents.entries.*.bootstrapMaxChars`
- `agents.entries.*.bootstrapTotalMaxChars`
- `agents.entries.*.contextLimits.*`

#### `agents.defaults.startupContext`

پیش‌درآمد راه‌اندازی نخستین نوبت را که در اجراهای مدلِ بازنشانی/راه‌اندازی تزریق می‌شود، کنترل می‌کند.
فرمان‌های ساده گفت‌وگو `/new` و `/reset` بازنشانی را بدون فراخوانی
مدل تأیید می‌کنند، بنابراین این پیش‌درآمد را بارگذاری نمی‌کنند.

```json5
{
  agents: {
    defaults: {
      startupContext: {
        enabled: true,
        applyOn: ["new", "reset"],
        dailyMemoryDays: 2,
        maxFileBytes: 16384,
        maxFileChars: 1200,
        maxTotalChars: 2800,
      },
    },
  },
}
```

#### `agents.defaults.contextLimits`

پیش‌فرض‌های مشترک برای سطوح محدودشده زمینه زمان اجرا.

```json5
{
  agents: {
    defaults: {
      contextLimits: {
        memoryGetMaxChars: 12000,
        postCompactionMaxChars: 1800,
      },
    },
  },
}
```

- `memoryGetMaxChars`: سقف پیش‌فرض گزیده `memory_get` پیش از افزوده‌شدن
  فراداده کوتاه‌سازی و اعلان ادامه.
- هنگامی که `memory_get` شامل `lines` نباشد، OpenClaw از یک پنجره داخلی 120خطی استفاده می‌کند و
  سپس `memoryGetMaxChars` را اعمال می‌کند.
- نتایج زنده ابزار از سقف خودکار زمینه مدل استفاده می‌کنند: `16000` نویسه زیر 100K
  توکن، `32000` نویسه در 100K+ توکن و `64000` نویسه در 200K+ توکن.
- `postCompactionMaxChars`: سقف گزیده AGENTS.md که هنگام تزریق تازه‌سازی
  پس از Compaction استفاده می‌شود.

#### `agents.entries.*.contextLimits`

بازنویسی مختص هر عامل برای گزینه‌های مشترک `contextLimits`. فیلدهای حذف‌شده از
`agents.defaults.contextLimits` به ارث می‌رسند.

```json5
{
  agents: {
    defaults: {
      contextLimits: { memoryGetMaxChars: 12000 },
    },
    list: [
      {
        id: "tiny-local",
        contextLimits: {
          memoryGetMaxChars: 6000,
        },
      },
    ],
  },
}
```

#### `skills.limits.maxSkillsPromptChars`

سقف سراسری فهرست فشرده Skills که به اعلان سیستم تزریق می‌شود. این مورد
بر خواندن فایل‌های `SKILL.md` هنگام درخواست تأثیری ندارد.

```json5
{
  skills: { limits: { maxSkillsPromptChars: 18000 } },
}
```

#### `agents.entries.*.skillsLimits.maxSkillsPromptChars`

بازنویسی مختص هر عامل برای بودجه اعلان Skills.

```json5
{
  agents: {
    list: [{ id: "tiny-local", skillsLimits: { maxSkillsPromptChars: 6000 } }],
  },
}
```

### `agents.defaults.imageMaxDimensionPx`

حداکثر اندازه پیکسلی طولانی‌ترین ضلع تصویر در بلوک‌های تصویر رونوشت/ابزار پیش از فراخوانی ارائه‌دهنده.
پیش‌فرض: `1200`.

مقادیر کمتر معمولاً مصرف توکن بینایی و اندازه محموله درخواست را برای اجراهای دارای اسکرین‌شات فراوان کاهش می‌دهند.
مقادیر بیشتر جزئیات بصری بیشتری را حفظ می‌کنند.

```json5
{
  agents: { defaults: { imageMaxDimensionPx: 1200 } },
}
```

### `agents.defaults.imageQuality`

ترجیح فشرده‌سازی/جزئیات ابزار تصویر برای تصاویر بارگذاری‌شده از مسیر فایل، URL و ارجاع‌های رسانه‌ای.
پیش‌فرض: `auto`.

OpenClaw نردبان تغییر اندازه را با مدل تصویر انتخاب‌شده تطبیق می‌دهد. برای نمونه، Claude Opus 4.8، OpenAI GPT-5.6 Sol، Qwen VL و مدل‌های میزبانی‌شده بینایی Llama 4 می‌توانند نسبت به مسیرهای قدیمی‌تر/پیش‌فرض بینایی با جزئیات بالا از تصاویر بزرگ‌تری استفاده کنند، در حالی که نوبت‌های چندتصویری در حالت `auto` با شدت بیشتری فشرده می‌شوند تا هزینه توکن و تأخیر کنترل شود.

مقادیر:

- `auto`: با محدودیت‌های مدل و تعداد تصاویر تطبیق بده.
- `efficient`: برای مصرف کمتر توکن و بایت، تصاویر کوچک‌تر را ترجیح بده.
- `balanced`: از نردبان استاندارد و متعادل استفاده کن.
- `high`: جزئیات بیشتری را برای اسکرین‌شات‌ها، نمودارها و تصاویر اسناد حفظ کن.

```json5
{
  agents: { defaults: { imageQuality: "auto" } },
}
```

### `agents.defaults.userTimezone`

منطقه زمانی برای زمینه اعلان سیستم (نه مُهرهای زمانی پیام). در صورت نبود، از منطقه زمانی میزبان استفاده می‌شود.

```json5
{
  agents: { defaults: { userTimezone: "America/Chicago" } },
}
```

### `agents.defaults.timeFormat`

قالب زمان در اعلان سیستم. پیش‌فرض: `auto` (ترجیح سیستم‌عامل).

```json5
{
  agents: { defaults: { timeFormat: "auto" } }, // auto | 12 | 24
}
```

### `agents.defaults.model`

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": { alias: "opus" },
        "minimax/MiniMax-M2.7": { alias: "minimax" },
      },
      model: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["minimax/MiniMax-M2.7"],
      },
      utilityModel: "openai/gpt-5.4-mini",
      imageModel: {
        primary: "openrouter/qwen/qwen-2.5-vl-72b-instruct:free",
        fallbacks: ["openrouter/google/gemini-2.0-flash-vision:free"],
      },
      mediaModels: {
        image: {
          primary: "openai/gpt-image-2",
          fallbacks: ["google/gemini-3.1-flash-image"],
        },
        video: {
          primary: "qwen/wan2.6-t2v",
          fallbacks: ["qwen/wan2.6-i2v"],
        },
      },
      pdfModel: {
        primary: "anthropic/claude-opus-4-6",
        fallbacks: ["openai/gpt-5.4-mini"],
      },
      params: { cacheRetention: "long" }, // پارامترهای پیش‌فرض سراسری ارائه‌دهنده
      pdfMaxMb: 10,
      pdfMaxPages: 20,
      thinkingDefault: "low",
      verboseDefault: "off",
      toolProgressDetail: "explain",
      reasoningDefault: "off",
      elevatedDefault: "on",
      timeoutSeconds: 600,
      mediaMaxMb: 5,
      contextTokens: 200000,
      maxConcurrent: 4,
    },
  },
}
```

- `model`: یک رشته (`"provider/model"`) یا یک شیء (`{ primary, fallbacks }`) را می‌پذیرد.
  - حالت رشته‌ای فقط مدل اصلی را تنظیم می‌کند.
  - حالت شیء، مدل اصلی را به‌همراه مدل‌های جایگزین مرتب‌شده تنظیم می‌کند.
- `utilityModel`: ارجاع یا نام مستعار اختیاری `provider/model` برای وظایف داخلی کوتاه. درحال‌حاضر برای عنوان‌های تولیدشده نشست در رابط کاربری کنترل، عنوان‌های موضوع پیام خصوصی Telegram، عنوان‌های خودکار رشته‌های Discord و [روایت پیش‌نویس پیشرفت](/fa/concepts/progress-drafts#narrated-status) استفاده می‌شود. وقتی تنظیم نشده باشد، OpenClaw در صورت وجود، مقدار پیش‌فرض اعلام‌شده مدل کوچکِ ارائه‌دهنده اصلی را استخراج می‌کند (OpenAI ← `gpt-5.6-luna`، Anthropic ← `claude-haiku-4-5`)؛ در غیر این صورت، وظایف عنوان از مدل اصلی عامل استفاده می‌کنند و روایت غیرفعال می‌ماند. اگر یک مدل کاربردی مجزا نتواند عنوان تولیدشده‌ای را آماده یا تکمیل کند، OpenClaw آن عنوان را یک‌بار با مدل اصلی دوباره امتحان می‌کند. برای عنوان‌های داشبورد، استخراج خودکار مدل کاربردی و جایگزین معمول از ارائه‌دهنده مؤثر نشست و نمایه احراز هویت آن استفاده می‌کنند؛ مدل کاربردی صریح، ارائه‌دهنده و احراز هویت پیکربندی‌شده خود را حفظ می‌کند. برای نادیده‌گرفتن مسیر جایگزین مدل کاربردی، `utilityModel: ""` را تنظیم کنید؛ تولید عنوان داشبورد همچنان مستقیماً با مدل معمول نشست ادامه می‌یابد. `agents.entries.*.utilityModel` مقدار پیش‌فرض را بازنویسی می‌کند و بازنویسی مدل مختص عملیات بر هر دو اولویت دارد. وظایف کاربردی فراخوانی‌های مدل جداگانه انجام می‌دهند و محتوای مختص وظیفه را به ارائه‌دهنده مدل انتخاب‌شده می‌فرستند. تولید عنوان داشبورد حداکثر 1,000 نویسه نخستِ اولین پیام غیر‌دستوری را می‌فرستد؛ روایت، درخواست ورودی را به‌همراه خلاصه‌های فشرده و پالایش‌شده ابزارها می‌فرستد. ارائه‌دهنده‌ای را انتخاب کنید که با الزامات هزینه و پردازش داده شما مطابقت داشته باشد.
- `imageModel`: یک رشته (`"provider/model"`) یا یک شیء (`{ primary, fallbacks }`) را می‌پذیرد.
  - وقتی مدل فعال نتواند تصویر بپذیرد، مسیر ابزار `image` از آن به‌عنوان پیکربندی مدل بینایی خود استفاده می‌کند. در عوض، مدل‌های دارای قابلیت بینایی بومی، بایت‌های تصویر بارگذاری‌شده را مستقیماً دریافت می‌کنند.
  - همچنین وقتی مدل انتخاب‌شده یا پیش‌فرض نتواند ورودی تصویر بپذیرد، برای مسیریابی جایگزین استفاده می‌شود.
  - ارجاع‌های صریح `provider/model` را ترجیح دهید. شناسه‌های بدون پیشوند برای سازگاری پذیرفته می‌شوند؛ اگر شناسه‌ای بدون پیشوند به‌طور یکتا با یک ورودی پیکربندی‌شده دارای قابلیت تصویر در `models.providers.*.models` مطابقت داشته باشد، OpenClaw آن را با ارائه‌دهنده مربوطه کامل می‌کند. تطابق‌های پیکربندی‌شده مبهم به پیشوند صریح ارائه‌دهنده نیاز دارند.
- `mediaModels.image`: یک رشته (`"provider/model"`) یا یک شیء (`{ primary, fallbacks }`) را می‌پذیرد.
  - قابلیت مشترک تولید تصویر و هر سطح ابزار/Plugin آینده که تصویر تولید کند، از آن استفاده می‌کنند.
  - مقادیر معمول: `google/gemini-3.1-flash-image` برای تولید تصویر بومی Gemini، `fal/fal-ai/flux/dev` برای fal، `openai/gpt-image-2` برای OpenAI Images یا `openai/gpt-image-1.5` برای خروجی PNG/WebP با پس‌زمینه شفاف OpenAI.
  - اگر ارائه‌دهنده/مدلی را مستقیماً انتخاب می‌کنید، احراز هویت منطبق ارائه‌دهنده را نیز پیکربندی کنید (برای نمونه، `GEMINI_API_KEY` یا `GOOGLE_API_KEY` برای `google/*`، `OPENAI_API_KEY` یا OpenAI Codex OAuth برای `openai/gpt-image-2` / `openai/gpt-image-1.5`، و `FAL_KEY` برای `fal/*`).
  - اگر حذف شود، `image_generate` همچنان می‌تواند یک ارائه‌دهنده پیش‌فرضِ دارای پشتوانه احراز هویت را استنباط کند. ابتدا ارائه‌دهنده پیش‌فرض فعلی و سپس سایر ارائه‌دهندگان ثبت‌شده تولید تصویر را به‌ترتیب شناسه ارائه‌دهنده امتحان می‌کند.
- `mediaModels.music`: یک رشته (`"provider/model"`) یا یک شیء (`{ primary, fallbacks }`) را می‌پذیرد.
  - قابلیت مشترک تولید موسیقی و ابزار داخلی `music_generate` از آن استفاده می‌کنند.
  - مقادیر معمول: `google/lyria-3-clip-preview`، `google/lyria-3-pro-preview` یا `minimax/music-2.6`.
  - اگر حذف شود، `music_generate` همچنان می‌تواند یک ارائه‌دهنده پیش‌فرضِ دارای پشتوانه احراز هویت را استنباط کند. ابتدا ارائه‌دهنده پیش‌فرض فعلی و سپس سایر ارائه‌دهندگان ثبت‌شده تولید موسیقی را به‌ترتیب شناسه ارائه‌دهنده امتحان می‌کند.
  - اگر ارائه‌دهنده/مدلی را مستقیماً انتخاب می‌کنید، احراز هویت/کلید API منطبق ارائه‌دهنده را نیز پیکربندی کنید.
- `mediaModels.video`: یک رشته (`"provider/model"`) یا یک شیء (`{ primary, fallbacks }`) را می‌پذیرد.
  - قابلیت مشترک تولید ویدئو و ابزار داخلی `video_generate` از آن استفاده می‌کنند.
  - مقادیر معمول: `qwen/wan2.6-t2v`، `qwen/wan2.6-i2v`، `qwen/wan2.6-r2v`، `qwen/wan2.6-r2v-flash` یا `qwen/wan2.7-r2v`.
  - اگر حذف شود، `video_generate` همچنان می‌تواند یک ارائه‌دهنده پیش‌فرضِ دارای پشتوانه احراز هویت را استنباط کند. ابتدا ارائه‌دهنده پیش‌فرض فعلی و سپس سایر ارائه‌دهندگان ثبت‌شده تولید ویدئو را به‌ترتیب شناسه ارائه‌دهنده امتحان می‌کند.
  - اگر ارائه‌دهنده/مدلی را مستقیماً انتخاب می‌کنید، احراز هویت/کلید API منطبق ارائه‌دهنده را نیز پیکربندی کنید.
  - Plugin رسمی تولید ویدئوی Qwen حداکثر از 1 ویدئوی خروجی، 1 تصویر ورودی، 4 ویدئوی ورودی، مدت‌زمان 10 ثانیه و گزینه‌های سطح ارائه‌دهنده `size`، `aspectRatio`، `resolution`، `audio` و `watermark` پشتیبانی می‌کند.
- `pdfModel`: یک رشته (`"provider/model"`) یا یک شیء (`{ primary, fallbacks }`) را می‌پذیرد.
  - ابزار `pdf` از آن برای مسیریابی مدل استفاده می‌کند.
  - اگر حذف شود، ابزار PDF ابتدا به `imageModel` و سپس به مدل حل‌شده نشست/پیش‌فرض بازمی‌گردد.
- `pdfMaxMb`: محدودیت پیش‌فرض اندازه PDF برای ابزار `pdf`، هنگامی که `maxBytesMb` در زمان فراخوانی ارسال نشده باشد.
- `pdfMaxPages`: حداکثر تعداد پیش‌فرض صفحاتی که حالت جایگزین استخراج در ابزار `pdf` بررسی می‌کند.
- `verboseDefault`: سطح پیش‌فرض پرگویی عامل‌ها. مقادیر: `"off"`، `"on"`، `"full"`. پیش‌فرض: `"off"`.
- `toolProgressDetail`: حالت جزئیات برای خلاصه‌های ابزار `/verbose` و خطوط ابزار در پیش‌نویس پیشرفت. مقادیر: `"explain"` (پیش‌فرض، برچسب‌های انسانی فشرده) یا `"raw"` (افزودن دستور/جزئیات خام در صورت وجود). `agents.entries.*.toolProgressDetail` مختص هر عامل، این مقدار پیش‌فرض را بازنویسی می‌کند.
- `reasoningDefault`: نمایانی پیش‌فرض استدلال برای عامل‌ها. مقادیر: `"off"`، `"on"`، `"stream"`. `agents.entries.*.reasoningDefault` مختص هر عامل، این مقدار پیش‌فرض را بازنویسی می‌کند. مقادیر پیش‌فرض پیکربندی‌شده استدلال فقط برای مالکان، فرستندگان مجاز یا زمینه‌های Gateway با مدیر اپراتور اعمال می‌شوند، آن هم وقتی هیچ بازنویسی استدلال مختص پیام یا نشست تنظیم نشده باشد.
- `elevatedDefault`: سطح پیش‌فرض خروجی ارتقایافته برای عامل‌ها. مقادیر: `"off"`، `"on"`، `"ask"`، `"full"`. پیش‌فرض: `"on"`.
- `model.primary`: قالب `provider/model` (برای مثال، `openai/gpt-5.6-sol` برای دسترسی Codex OAuth). اگر ارائه‌دهنده را حذف کنید، OpenClaw ابتدا یک نام مستعار، سپس یک تطابق یکتای ارائه‌دهنده پیکربندی‌شده برای دقیقاً همان شناسه مدل را امتحان می‌کند و تنها پس از آن به ارائه‌دهنده پیش‌فرض پیکربندی‌شده بازمی‌گردد (رفتار سازگاری منسوخ‌شده است، بنابراین `provider/model` صریح را ترجیح دهید). اگر آن ارائه‌دهنده دیگر مدل پیش‌فرض پیکربندی‌شده را ارائه نکند، OpenClaw به‌جای نمایش یک پیش‌فرض قدیمیِ ارائه‌دهنده حذف‌شده، به اولین ارائه‌دهنده/مدل پیکربندی‌شده بازمی‌گردد.
- `contextTokens`: سقف اختیاری سراسری عامل. می‌تواند بودجه مؤثر یک مدل بزرگ‌تر را کاهش دهد، اما نمی‌تواند مدل را بالاتر از `contextTokens` پیکربندی‌شده یا کشف‌شده آن ببرد. برای فعال‌کردن پنجره بومی بزرگ‌تر یک مدل مستقیم OpenAI، `models.providers.openai.models[].contextWindow` و `contextTokens` را برای آن مدل تنظیم کنید؛ [مقادیر پیش‌فرض پنجره زمینه OpenAI](/fa/providers/openai#context-window-defaults-and-long-context-opt-in) را ببینید.
- `models`: نام‌های مستعار پیکربندی‌شده و تنظیمات مختص هر مدل. هر ورودی می‌تواند شامل `alias` (میان‌بر) و `params` (مختص ارائه‌دهنده، برای مثال `temperature`، `maxTokens`، `cacheRetention`، `context1m`، `responsesServerCompaction`، `responsesCompactThreshold`، مسیریابی `provider` در OpenRouter، `chat_template_kwargs`، `extra_body`/`extraBody`) باشد. افزودن ورودی‌ها، بازنویسی مدل‌ها را محدود نمی‌کند.
  - برای نمایش همه مدل‌های کشف‌شده ارائه‌دهندگان انتخابی، بدون فهرست‌کردن دستی تک‌تک شناسه‌های مدل، از ورودی‌های `provider/*` مانند `"openai/*": {}` یا `"vllm/*": {}` استفاده کنید.
  - وقتی همه مدل‌های پویای کشف‌شده برای یک ارائه‌دهنده باید از زمان‌اجرای یکسانی استفاده کنند، `agentRuntime` را به ورودی `provider/*` اضافه کنید. سیاست دقیق زمان‌اجرای `provider/model` همچنان بر نویسه عام اولویت دارد.
  - ویرایش‌های امن فراداده: برای افزودن ورودی‌ها از `openclaw config set agents.defaults.models '<json>' --strict-json --merge` استفاده کنید. `config set` از جایگزینی‌هایی که ورودی‌های موجود را حذف می‌کنند خودداری می‌کند، مگر اینکه `--replace` را ارسال کنید.
- `modelPolicy.allow`: فهرست مجاز صریح بازنویسی‌ها. نام‌های مستعار، ارجاع‌های دقیق `provider/model` و نویسه‌های عام انتهای پیشوند مانند `openai/*` یا `clawrouter/anthropic/*` را می‌پذیرد. برای مجازکردن هر مدلی، آن را حذف کنید یا از `[]` استفاده کنید. `agents.entries.*.modelPolicy.allow` سیاست پیش‌فرض آن عامل را جایگزین می‌کند؛ یک فهرست خالی صریح، آن عامل را به حالت مجازبودن همه مدل‌ها وارد می‌کند.
  - جریان‌های پیکربندی/راه‌اندازی اولیه مختص ارائه‌دهنده، مدل‌های انتخاب‌شده ارائه‌دهنده را در این نگاشت ادغام می‌کنند و ارائه‌دهندگان نامرتبطی را که از قبل پیکربندی شده‌اند حفظ می‌کنند.
  - برای مدل‌های مستقیم OpenAI Responses، Compaction سمت سرور به‌طور خودکار فعال می‌شود. برای توقف تزریق `context_management` از `params.responsesServerCompaction: false` یا برای بازنویسی آستانه از `params.responsesCompactThreshold` استفاده کنید. [Compaction سمت سرور OpenAI](/fa/providers/openai#advanced-configuration) را ببینید.
- `params`: پارامترهای پیش‌فرض سراسری ارائه‌دهنده که بر همه مدل‌ها اعمال می‌شوند. در `agents.defaults.params` تنظیم کنید (برای مثال `{ cacheRetention: "long" }`).
- تقدم ادغام `params` (پیکربندی): `agents.defaults.models["provider/model"].params` (مختص مدل) مقدار `agents.defaults.params` (پایه سراسری) را بازنویسی می‌کند، سپس `agents.entries.*.params` (شناسه عامل منطبق) بر اساس کلید بازنویسی می‌کند. برای جزئیات، [ذخیره‌سازی موقت پرامپت](/fa/reference/prompt-caching) را ببینید.
- `models.providers.openrouter.params.provider`: سیاست پیش‌فرض مسیریابی ارائه‌دهنده در سراسر OpenRouter. OpenClaw آن را به شیء `provider` درخواست OpenRouter ارسال می‌کند؛ `agents.defaults.models["openrouter/<model>"].params.provider` مختص مدل و پارامترهای عامل بر اساس کلید بازنویسی می‌کنند. [مسیریابی ارائه‌دهنده OpenRouter](/fa/providers/openrouter#advanced-configuration) را ببینید.
- `params.extra_body`/`params.extraBody`: JSON عبوری پیشرفته که در بدنه درخواست‌های `api: "openai-completions"` برای پراکسی‌های سازگار با OpenAI ادغام می‌شود. اگر با کلیدهای تولیدشده درخواست تداخل داشته باشد، بدنه اضافی اولویت دارد؛ مسیرهای تکمیل غیربومی همچنان پس از آن `store` مختص OpenAI را حذف می‌کنند.
- `params.chat_template_kwargs`: آرگومان‌های الگوی گفت‌وگوی سازگار با vLLM/OpenAI که در بدنه سطح‌بالای درخواست‌های `api: "openai-completions"` ادغام می‌شوند. برای `vllm/nemotron-3-*` با تفکر غیرفعال، Plugin همراه vLLM به‌طور خودکار `enable_thinking: false` و `force_nonempty_content: true` را ارسال می‌کند؛ `chat_template_kwargs` صریح، مقادیر پیش‌فرض تولیدشده را بازنویسی می‌کند و `extra_body.chat_template_kwargs` همچنان اولویت نهایی را دارد. مدل‌های تفکر Qwen و Nemotron پیکربندی‌شده در vLLM، به‌جای نردبان چندسطحی تلاش، گزینه‌های دودویی `/think` (`off`، `on`) را ارائه می‌کنند.
- `compat.thinkingFormat`: سبک محموله تفکر سازگار با OpenAI. از `"together"` برای `reasoning.enabled` به‌سبک Together، از `"qwen"` برای `enable_thinking` سطح‌بالا به‌سبک Qwen، یا از `"qwen-chat-template"` برای `chat_template_kwargs.enable_thinking` در بک‌اندهای خانواده Qwen که از آرگومان‌های کلیدی الگوی گفت‌وگو در سطح درخواست پشتیبانی می‌کنند، مانند vLLM، استفاده کنید. OpenClaw تفکر غیرفعال را به `false` و تفکر فعال را به `true` نگاشت می‌کند و مدل‌های Qwen پیکربندی‌شده در vLLM برای این قالب‌ها گزینه‌های دودویی `/think` را ارائه می‌کنند.
- `compat.supportedReasoningEfforts`: فهرست میزان تلاش استدلالی سازگار با OpenAI برای هر مدل. برای نقاط پایانی سفارشی که واقعاً `"xhigh"` را می‌پذیرند، آن را درج کنید؛ سپس OpenClaw اعتبارسنجی `/think xhigh` را در منوهای فرمان، ردیف‌های نشست Gateway، اعتبارسنجی وصله نشست، اعتبارسنجی CLI عامل و اعتبارسنجی `llm-task` برای ارائه‌دهنده/مدل پیکربندی‌شده ارائه می‌کند. هنگامی که بک‌اند برای یک سطح استاندارد به مقداری ویژه ارائه‌دهنده نیاز دارد، از `compat.reasoningEffortMap` استفاده کنید.
- `params.preserveThinking`: گزینه فعال‌سازی مختص Z.AI برای حفظ تفکر. هنگامی که فعال باشد و تفکر روشن باشد، OpenClaw مقدار `thinking.clear_thinking: false` را ارسال و `reasoning_content` قبلی را بازپخش می‌کند؛ [تفکر و تفکر حفظ‌شده در Z.AI](/fa/providers/zai#advanced-configuration) را ببینید.
- `localService`: مدیر پردازش اختیاری در سطح ارائه‌دهنده برای سرورهای مدل محلی/خودمیزبان. هنگامی که مدل انتخاب‌شده متعلق به آن ارائه‌دهنده باشد، OpenClaw نشانی `healthUrl` (یا `baseUrl + "/models"`) را بررسی می‌کند، اگر نقطه پایانی از دسترس خارج باشد `command` را با `args` راه‌اندازی می‌کند، تا `readyTimeoutMs` منتظر می‌ماند و سپس درخواست مدل را ارسال می‌کند. `command` باید یک مسیر مطلق باشد. `idleStopMs: 0` پردازش را تا خروج OpenClaw زنده نگه می‌دارد؛ یک مقدار مثبت، پردازش راه‌اندازی‌شده توسط OpenClaw را پس از آن تعداد میلی‌ثانیه بیکاری متوقف می‌کند. [سرویس‌های مدل محلی](/fa/gateway/local-model-services) را ببینید.
- سیاست زمان اجرا باید روی ارائه‌دهندگان یا مدل‌ها قرار گیرد، نه روی `agents.defaults`. برای قواعد سراسری ارائه‌دهنده از `models.providers.<provider>.agentRuntime` و برای قواعد مختص مدل از `agents.defaults.models["provider/model"].agentRuntime` / `agents.entries.*.models["provider/model"].agentRuntime` استفاده کنید. پیشوند ارائه‌دهنده/مدل به‌تنهایی هرگز یک هارنس را انتخاب نمی‌کند. اگر زمان اجرا تنظیم نشده باشد یا `auto` باشد، OpenAI فقط برای یک مسیر رسمی و دقیق HTTPS از نوع Platform Responses یا ChatGPT Responses، بدون بازنویسی درخواست توسط نویسنده، ممکن است Codex را به‌طور ضمنی انتخاب کند. [زمان اجرای ضمنی عامل OpenAI](/fa/providers/openai#implicit-agent-runtime) را ببینید.
- نویسندگان پیکربندی که این فیلدها را تغییر می‌دهند (برای مثال `/models set`، `/models set-image` و فرمان‌های افزودن/حذف جایگزین) فرم شیء استاندارد را ذخیره می‌کنند و در صورت امکان فهرست‌های جایگزین موجود را حفظ می‌کنند.
- `maxConcurrent`: حداکثر اجرای موازی عامل در میان نشست‌ها (هر نشست همچنان به‌صورت ترتیبی اجرا می‌شود). پیش‌فرض: `4`.

### سیاست زمان اجرا

```json5
{
  models: {
    providers: {
      openai: {
        agentRuntime: { id: "codex" },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
      models: {
        "anthropic/claude-opus-5": {
          agentRuntime: { id: "claude-cli" },
        },
        "vllm/*": {
          agentRuntime: { id: "openclaw" },
        },
      },
    },
  },
}
```

- `id`: `"auto"`، `"openclaw"`، شناسهٔ ثبت‌شدهٔ مهار Plugin، یا نام مستعار پشتیبانی‌شدهٔ بک‌اند CLI. Plugin همراه Codex، `codex` را ثبت می‌کند؛ Plugin همراه Anthropic بک‌اند CLI با نام `claude-cli` را فراهم می‌کند.
- `id: "auto"` به مهارهای ثبت‌شدهٔ Plugin اجازه می‌دهد مسیرهای مؤثری را که قرارداد پشتیبانی خود را اعلام می‌کنند یا به‌نحوی برآورده می‌سازند در اختیار بگیرند، و اگر هیچ مهاری منطبق نباشد از OpenClaw استفاده می‌کند. یک زمان اجرای صریح Plugin مانند `id: "codex"` به آن مهار و یک مسیر مؤثر سازگار نیاز دارد؛ اگر هرکدام در دسترس نباشد یا اجرا ناموفق باشد، به‌صورت بسته شکست می‌خورد.
- `id: "pi"` فقط به‌عنوان نام مستعار منسوخ‌شدهٔ `openclaw` پذیرفته می‌شود تا پیکربندی‌های منتشرشده در v2026.5.22 و نسخه‌های پیشین حفظ شوند. پیکربندی جدید باید از `openclaw` استفاده کند.
- ترتیب تقدم زمان اجرا ابتدا سیاست دقیق مدل (`agents.entries.*.models["provider/model"]`، `agents.defaults.models["provider/model"]`، یا `models.providers.<provider>.models[]`) است، سپس `agents.entries.*` / `agents.defaults.models["provider/*"]`، و پس از آن سیاست سراسری ارائه‌دهنده در `models.providers.<provider>.agentRuntime`.
- کلیدهای زمان اجرای کل عامل قدیمی هستند. `agents.defaults.agentRuntime`، `agents.entries.*.agentRuntime`، پین‌های زمان اجرای نشست، و `OPENCLAW_AGENT_RUNTIME` در انتخاب زمان اجرا نادیده گرفته می‌شوند. برای حذف مقادیر منسوخ، `openclaw doctor --fix` را اجرا کنید.
- مسیرهای رسمی HTTPS دقیق و واجد شرایط OpenAI Responses/ChatGPT که بازنویسی تألیفی درخواست ندارند، ممکن است به‌طور ضمنی از مهار Codex استفاده کنند. `agentRuntime.id: "codex"` در سطح ارائه‌دهنده/مدل، Codex را به الزامی با شکست بسته تبدیل می‌کند، اما یک مسیر ناسازگار را سازگار نمی‌کند.
- برای استقرارهای Claude CLI، استفاده از `model: "anthropic/claude-opus-5"` به‌همراه `agentRuntime.id: "claude-cli"` در محدودهٔ مدل ترجیح داده می‌شود. ارجاع‌های قدیمی `claude-cli/<model>` همچنان برای سازگاری کار می‌کنند، اما پیکربندی جدید باید انتخاب ارائه‌دهنده/مدل را کانونی نگه دارد و بک‌اند اجرا را در سیاست زمان اجرای ارائه‌دهنده/مدل قرار دهد.
- این فقط اجرای نوبت عامل متنی را کنترل می‌کند. تولید رسانه، بینایی، PDF، موسیقی، ویدئو و TTS همچنان از تنظیمات ارائه‌دهنده/مدل خود استفاده می‌کنند.

**صورت‌های کوتاه نام مستعار داخلی** (فقط زمانی اعمال می‌شوند که مدل در `agents.defaults.models` باشد):

| نام مستعار               | مدل                           |
| ------------------- | ------------------------------- |
| `opus`              | `anthropic/claude-opus-5`       |
| `sonnet`            | `anthropic/claude-sonnet-5`     |
| `gpt`               | `openai/gpt-5.4`                |
| `gpt-mini`          | `openai/gpt-5.4-mini`           |
| `gpt-nano`          | `openai/gpt-5.4-nano`           |
| `gemini`            | `google/gemini-3.1-pro-preview` |
| `gemini-flash`      | `google/gemini-3-flash-preview` |
| `gemini-flash-lite` | `google/gemini-3.1-flash-lite`  |

نام‌های مستعار پیکربندی‌شدهٔ شما همیشه بر پیش‌فرض‌ها تقدم دارند.

مدل‌های Z.AI GLM-4.x به‌طور خودکار حالت تفکر را فعال می‌کنند، مگر اینکه `--thinking off` را تنظیم کنید یا خودتان `agents.defaults.models["zai/<model>"].params.thinking` را تعریف کنید.
مدل‌های Z.AI برای پخش جریانی فراخوانی ابزار، `tool_stream` را به‌طور پیش‌فرض فعال می‌کنند. برای غیرفعال‌کردن آن، `agents.defaults.models["zai/<model>"].params.tool_stream` را روی `false` تنظیم کنید.
در OpenClaw، تفکر Anthropic Claude Opus 4.8 به‌طور پیش‌فرض غیرفعال می‌ماند؛ وقتی تفکر تطبیقی صریحاً فعال شود، پیش‌فرض تلاش متعلق به ارائه‌دهندهٔ Anthropic برابر `high` است. اگر سطح تفکر صریحی تنظیم نشده باشد، مدل‌های Claude 4.6 به‌طور پیش‌فرض از `adaptive` استفاده می‌کنند.

### انتخاب بک‌اند CLI

سازوکارهای آداپتور CLI توسط Pluginها ثبت می‌شوند و زیر پیش‌فرض‌های عامل
پیکربندی نمی‌شوند. همان‌طور که در بالا نشان داده شد، یک بک‌اند CLI ثبت‌شده را با
`agentRuntime.id` در محدودهٔ مدل انتخاب کنید. برای عملیات به [بک‌اندهای CLI](/fa/gateway/cli-backends)
و برای ثبت فرمان، نشست، تصویر و تجزیه‌گر به [ساخت Pluginهای بک‌اند CLI](/fa/plugins/cli-backend-plugins)
مراجعه کنید.

### `agents.defaults.promptOverlays`

هم‌پوشانی‌های اعلان مستقل از ارائه‌دهنده که بر اساس خانوادهٔ مدل روی سطوح اعلان ساخته‌شده توسط OpenClaw اعمال می‌شوند. شناسه‌های مدل خانوادهٔ GPT-5 قرارداد رفتاری مشترک را در مسیرهای OpenClaw/ارائه‌دهنده دریافت می‌کنند؛ `personality` فقط لایهٔ سبک تعامل دوستانه را کنترل می‌کند. مسیرهای بومی سرور برنامهٔ Codex به‌جای این هم‌پوشانی GPT-5 متعلق به OpenClaw، دستورالعمل‌های پایه/مدل متعلق به Codex را حفظ می‌کنند، و OpenClaw شخصیت داخلی Codex را برای رشته‌های بومی غیرفعال می‌کند.

```json5
{
  agents: {
    defaults: {
      promptOverlays: {
        gpt5: {
          personality: "friendly", // friendly | on | off
        },
      },
    },
  },
}
```

- `"friendly"` (پیش‌فرض) و `"on"` لایهٔ سبک تعامل دوستانه را فعال می‌کنند.
- `"off"` فقط لایهٔ دوستانه را غیرفعال می‌کند؛ قرارداد رفتاری برچسب‌خوردهٔ GPT-5 فعال می‌ماند.
- `plugins.entries.openai.config.personality` قدیمی همچنان زمانی خوانده می‌شود که این تنظیم مشترک تعیین نشده باشد.

### `agents.defaults.heartbeat`

اجرای دوره‌ای Heartbeat.

```json5
{
  agents: {
    defaults: {
      heartbeat: {
        every: "30m", // 0m disables
        model: "openai/gpt-5.4-mini",
        includeReasoning: false,
        includeSystemPromptSection: true, // default: true; false omits the Heartbeat section from the system prompt
        lightContext: false, // default: false; true skips workspace bootstrap files for heartbeat runs
        isolatedSession: false, // default: false; true runs each heartbeat in a fresh session (no conversation history)
        skipWhenBusy: false, // default: false; true also waits for this agent's subagent/nested lanes
        session: "main",
        to: "+15555550123",
        directPolicy: "allow", // allow (default) | block
        target: "none", // default: none | options: last | whatsapp | telegram | discord | ...
        prompt: "Follow the heartbeat monitor scratch context...",
        ackMaxChars: 300,
        suppressToolErrorWarnings: false,
        timeoutSeconds: 45,
      },
    },
  },
}
```

- `every`: رشتهٔ مدت‌زمان (ms/s/m/h). پیش‌فرض: `30m` (احراز هویت با کلید API) یا `1h` (احراز هویت OAuth). برای غیرفعال‌کردن، آن را روی `0m` تنظیم کنید.
- تناوب در یک ردیف پایش Cron متعلق به سیستم نوشته می‌شود. برای ایجاد یک ردیف مفقود یا منسوخ، `openclaw doctor --fix` را اجرا کنید. اگر Cron غیرفعال باشد، Heartbeatهای زمان‌بندی‌شده اجرا نمی‌شوند و Gateway یک هشدار راه‌اندازی ثبت می‌کند.
- `includeSystemPromptSection`: وقتی false باشد، بخش Heartbeat را از اعلان سیستم حذف می‌کند. پیش‌فرض: `true`.
- `suppressToolErrorWarnings`: وقتی true باشد، محموله‌های هشدار خطای ابزار را هنگام اجرای Heartbeat سرکوب می‌کند.
- `timeoutSeconds`: حداکثر زمان مجاز بر حسب ثانیه برای یک نوبت عامل Heartbeat، پیش از لغو آن. اگر تنظیم نشود، در صورت تنظیم‌بودن از `agents.defaults.timeoutSeconds` استفاده می‌شود؛ در غیر این صورت، تناوب Heartbeat با سقف 600 ثانیه به‌کار می‌رود.
- `directPolicy`: سیاست تحویل مستقیم/DM. `allow` (پیش‌فرض) تحویل به مقصد مستقیم را مجاز می‌کند. `block` تحویل به مقصد مستقیم را سرکوب و `reason=dm-blocked` را منتشر می‌کند.
- `lightContext`: وقتی true باشد، اجرای Heartbeat از زمینهٔ راه‌اندازی سبک استفاده می‌کند و فایل‌های راه‌اندازی فضای کاری را نادیده می‌گیرد. زمینهٔ موقت پایش در هر دو حالت توسط اجراکنندهٔ Heartbeat تزریق می‌شود.
- `isolatedSession`: وقتی true باشد، هر Heartbeat در نشستی تازه و بدون سابقهٔ مکالمهٔ قبلی اجرا می‌شود. همان الگوی جداسازی Cron `sessionTarget: "isolated"`. هزینهٔ توکن هر Heartbeat را از حدود ~100K به حدود ~2-5K توکن کاهش می‌دهد.
- `skipWhenBusy`: وقتی true باشد، اجرای Heartbeat در مسیرهای مشغول اضافی آن عامل به تعویق می‌افتد: کار زیرعامل مبتنی بر کلید نشست خودش یا کار فرمان تو‌در‌تو. مسیرهای Cron حتی بدون این پرچم همیشه Heartbeatها را به تعویق می‌اندازند.
- برای هر عامل: `agents.entries.*.heartbeat` را تنظیم کنید. وقتی هر عاملی `heartbeat` را تعریف کند، **فقط همان عامل‌ها** Heartbeat اجرا می‌کنند.
- Heartbeatها نوبت‌های کامل عامل را اجرا می‌کنند — فاصله‌های کوتاه‌تر توکن بیشتری مصرف می‌کنند.

### `agents.defaults.compaction`

```json5
{
  agents: {
    defaults: {
      compaction: {
        mode: "safeguard", // default | safeguard
        provider: "my-provider", // id of a registered compaction provider plugin (optional)
        thinkingLevel: "low", // optional compaction-only thinking override
        timeoutSeconds: 180,
        keepRecentTokens: 50000,
        recentTurnsPreserve: 3,
        identifierPolicy: "strict", // strict | off
        qualityGuard: { enabled: true, maxRetries: 1 },
        midTurnPrecheck: { enabled: false }, // optional tool-loop pressure check
        postIndexSync: "async", // off | async | await
        postCompactionSections: ["Session Startup", "Red Lines"],
        model: "openrouter/anthropic/claude-sonnet-4-6", // optional compaction-only model override
        truncateAfterCompaction: true, // rotate to a smaller successor JSONL after compaction
        maxActiveTranscriptBytes: "20mb", // optional preflight local compaction trigger
        notifyUser: true, // notices when compaction starts/completes and on memory-flush degradation (default: false)
        memoryFlush: {
          enabled: true,
          model: "ollama/qwen3:8b", // optional memory-flush-only model override
          softThresholdTokens: 6000,
          forceFlushTranscriptBytes: "2mb",
        },
      },
    },
  },
}
```

- `mode`: `default` یا `safeguard` (خلاصه‌سازی قطعه‌ای برای تاریخچه‌های طولانی). به [Compaction](/fa/concepts/compaction) مراجعه کنید.
- `provider`: شناسهٔ Plugin ارائه‌دهندهٔ Compaction ثبت‌شده. وقتی تنظیم شود، به‌جای خلاصه‌سازی داخلی LLM، `summarize()` ارائه‌دهنده فراخوانی می‌شود. در صورت شکست، به روش داخلی بازمی‌گردد. تنظیم ارائه‌دهنده، `mode: "safeguard"` را اجباری می‌کند. به [Compaction](/fa/concepts/compaction) مراجعه کنید.
- `thinkingLevel`: سطح تفکر اختیاری که فقط برای خلاصه‌های Compaction تعبیه‌شدهٔ OpenClaw استفاده می‌شود (`off`، `minimal`، `low`، `medium`، `high`، `xhigh`، `adaptive`، `max` یا `ultra`). این مقدار سطح تفکر فعلی نشست را بازنویسی می‌کند و به مدل/زمان‌اجرای Compaction انتخاب‌شده محدود می‌شود. برای به‌ارث‌بردن سطح نشست، آن را تنظیم‌نشده باقی بگذارید. Compaction بومی app-server در Codex این تنظیم را نادیده می‌گیرد، زیرا درخواست compact بومی فاقد بازنویسی تفکر برای هر عملیات است؛ OpenClaw هنگام پیکربندی آن، هشداری ثبت می‌کند.
- `timeoutSeconds`: حداکثر تعداد ثانیهٔ مجاز برای یک عملیات Compaction پیش از آنکه OpenClaw آن را لغو کند. پیش‌فرض: `180`.
- `keepRecentTokens`: بودجهٔ نقطهٔ برش عامل برای حفظ عیناً آخرین دنبالهٔ رونوشت. `/compact` دستی، وقتی صریحاً تنظیم شده باشد، آن را رعایت می‌کند؛ در غیر این صورت، Compaction دستی یک نقطهٔ بازرسی سخت است.
- `recentTurnsPreserve`: تعداد آخرین نوبت‌های کاربر/دستیار که بیرون از خلاصه‌سازی حفاظتی عیناً حفظ می‌شوند. پیش‌فرض: `3`.
- `identifierPolicy`: `strict` (پیش‌فرض) یا `off`. `strict` هنگام خلاصه‌سازی Compaction، راهنمای داخلی حفظ شناسه‌های مات را در ابتدا اضافه می‌کند.
- `qualityGuard`: بررسی‌های تلاش مجدد در صورت خروجی بدشکل برای خلاصه‌های حفاظتی. در حالت حفاظتی به‌طور پیش‌فرض فعال است؛ برای ردکردن ممیزی، `enabled: false` را تنظیم کنید.
- `midTurnPrecheck`: بررسی اختیاری فشار حلقهٔ ابزار. وقتی `enabled: true` باشد، OpenClaw پس از افزوده‌شدن نتایج ابزار و پیش از فراخوانی بعدی مدل، فشار زمینه را بررسی می‌کند. اگر زمینه دیگر نگنجد، تلاش جاری را پیش از ارسال پرامپت لغو می‌کند و برای کوتاه‌کردن نتایج ابزار یا انجام Compaction و تلاش مجدد، از مسیر بازیابی موجودِ پیش‌بررسی دوباره استفاده می‌کند. با هر دو حالت Compaction یعنی `default` و `safeguard` کار می‌کند. پیش‌فرض: غیرفعال.
- `postIndexSync`: حالت نمایه‌سازی مجدد حافظهٔ نشست پس از Compaction. پیش‌فرض: `"async"`. برای بیشترین تازگی از `"await"`، برای تأخیر کمتر Compaction از `"async"`، یا فقط هنگامی که همگام‌سازی حافظهٔ نشست در جای دیگری انجام می‌شود از `"off"` استفاده کنید.
- `postCompactionSections`: نام‌های اختیاری بخش‌های H2/H3 در AGENTS.md که پس از Compaction دوباره تزریق می‌شوند. برای غیرفعال‌سازی، آن را تنظیم‌نشده بگذارید یا از `[]` استفاده کنید.
- `model`: `provider/model-id` اختیاری یا نام مستعار ساده از `agents.defaults.models` فقط برای خلاصه‌سازی Compaction. نام‌های مستعار ساده پیش از ارسال حل می‌شوند؛ در صورت تداخل، شناسه‌های لفظی مدلِ پیکربندی‌شده اولویت خود را حفظ می‌کنند. زمانی از این گزینه استفاده کنید که نشست اصلی باید یک مدل را حفظ کند، اما خلاصه‌های Compaction باید روی مدل دیگری اجرا شوند؛ وقتی تنظیم نشده باشد، Compaction از مدل اصلی نشست استفاده می‌کند.
- `truncateAfterCompaction`: پس از Compaction رونوشت نشست فعال را چرخش می‌دهد تا نوبت‌های آینده فقط خلاصه و دنبالهٔ خلاصه‌نشده را بارگذاری کنند، درحالی‌که رونوشت کامل قبلی بایگانی‌شده باقی می‌ماند. از رشد نامحدود رونوشت فعال در نشست‌های طولانی‌مدت جلوگیری می‌کند. پیش‌فرض: `false`.
- `maxActiveTranscriptBytes`: آستانهٔ اختیاری بایت (`number` یا رشته‌هایی مانند `"20mb"`) که وقتی تاریخچهٔ رونوشت از آستانه فراتر رود، پیش از اجرا Compaction محلی عادی را فعال می‌کند. به `truncateAfterCompaction` نیاز دارد تا Compaction موفق بتواند به رونوشت جانشین کوچک‌تری چرخش کند. وقتی تنظیم نشده باشد یا `0` باشد، غیرفعال است.
- `notifyUser`: وقتی `true` باشد، اعلان‌های کوتاه نگهداشت زمینه را برای کاربر می‌فرستد: هنگام شروع و تکمیل Compaction (برای مثال، «در حال فشرده‌سازی زمینه...» و «Compaction کامل شد»)، و هنگامی که تخلیهٔ حافظهٔ پیش از Compaction به پایان ظرفیت خود می‌رسد و پاسخ در وضعیت تنزل‌یافته ادامه می‌یابد (برای مثال، «نگهداشت حافظه موقتاً ناموفق بود؛ پاسخ شما ادامه می‌یابد.»). برای بی‌صدا نگه‌داشتن این اعلان‌ها، به‌طور پیش‌فرض غیرفعال است.
- `memoryFlush`: نوبت عاملی بی‌صدا پیش از Compaction خودکار برای ذخیرهٔ حافظه‌های پایدار. وقتی این نوبت نگهداشت باید روی مدل محلی باقی بماند، `model` را روی ارائه‌دهنده/مدل دقیقی مانند `ollama/qwen3:8b` تنظیم کنید؛ این بازنویسی زنجیرهٔ جایگزین نشست فعال را به ارث نمی‌برد. `forceFlushTranscriptBytes` وقتی اندازهٔ رونوشت به آستانه برسد، حتی اگر شمارنده‌های توکن کهنه باشند، تخلیه را اجباری می‌کند. وقتی فضای کاری فقط‌خواندنی باشد، رد می‌شود.

دستورالعمل‌های سفارشی Compaction تحت مالکیت کد هستند. برای ساخت سفارشی خلاصه، یک
Plugin ارائه‌دهندهٔ Compaction با `summarize()` پیاده‌سازی کنید و هنگامی که
زمینهٔ پس از Compaction باید به پرامپت‌های بعدی مدل تزریق شود، از
`before_prompt_build` استفاده کنید. Doctor فیلدهای دستورالعمل منسوخ‌شده را حذف می‌کند و به این
درزها اشاره می‌کند.

### `agents.defaults.contextPruning`

**نتایج قدیمی ابزار** را پیش از ارسال به LLM از زمینهٔ درون‌حافظه‌ای هرس می‌کند. تاریخچهٔ نشست روی دیسک را تغییر **نمی‌دهد**. به‌طور پیش‌فرض غیرفعال است؛ برای فعال‌سازی، `mode: "cache-ttl"` را تنظیم کنید.

```json5
{
  agents: {
    defaults: {
      contextPruning: {
        mode: "cache-ttl", // خاموش (پیش‌فرض) | cache-ttl
      },
    },
  },
}
```

<Accordion title="رفتار حالت cache-ttl">

- `mode: "cache-ttl"` گذرهای هرس را فعال می‌کند.
- هرس ابتدا نتایج بیش‌ازحد بزرگ ابزار را به‌صورت نرم کوتاه می‌کند، سپس در صورت نیاز نتایج قدیمی‌تر ابزار را کاملاً پاک می‌کند.

**کوتاه‌سازی نرم** ابتدا + انتها را حفظ می‌کند و `...` را در میانه درج می‌کند.

**پاک‌سازی کامل** کل نتیجهٔ ابزار را با جای‌نگهدار جایگزین می‌کند.

نکته‌ها:

- بلوک‌های تصویر هرگز کوتاه/پاک نمی‌شوند.
- نسبت‌ها بر پایهٔ نویسه هستند (تقریبی)، نه شمارش دقیق توکن.
- جدیدترین پیام‌های دستیار حفظ می‌شوند.

</Accordion>

برای جزئیات رفتار، به [هرس نشست](/fa/concepts/session-pruning) مراجعه کنید.

### استریم‌سازی بلوکی

```json5
{
  agents: {
    defaults: {
      blockStreamingDefault: "off", // on | off
      blockStreamingBreak: "text_end", // text_end | message_end
      blockStreamingChunk: { minChars: 800, maxChars: 1200, breakPreference: "paragraph" },
      blockStreamingCoalesce: { idleMs: 1000 },
      humanDelay: { mode: "natural" }, // off (پیش‌فرض) | natural | custom (استفاده از minMs/maxMs)
    },
  },
}
```

- کانال‌های غیر از Telegram برای فعال‌سازی پاسخ‌های بلوکی به `*.streaming.block.enabled: true` صریح نیاز دارند. QQ Bot استثناست: هیچ کلید `streaming.block` ندارد و پاسخ‌های بلوکی را استریم می‌کند، مگر اینکه `channels.qqbot.streaming.mode` برابر `"off"` باشد.
- بازنویسی‌های کانال: `channels.<channel>.streaming.block.coalesce` (و گونه‌های مختص هر حساب). Discord، Google Chat، Mattermost، MS Teams، Signal و Slack به‌طور پیش‌فرض `minChars: 1500` / `idleMs: 1000` هستند.
- `blockStreamingChunk.breakPreference`: مرز ترجیحی قطعه (`"paragraph" | "newline" | "sentence"`).
- `humanDelay`: مکث تصادفی میان پاسخ‌های بلوکی. پیش‌فرض: `off`. `natural` = 800-2500ms. `custom` از `minMs`/`maxMs` استفاده می‌کند (برای هر کران تنظیم‌نشده، به بازهٔ طبیعی بازمی‌گردد). بازنویسی مختص عامل: `agents.entries.*.humanDelay`.

برای جزئیات رفتار + قطعه‌بندی، به [استریم‌سازی](/fa/concepts/streaming) مراجعه کنید.

### نشانگرهای تایپ

```json5
{
  agents: {
    defaults: {
      typingMode: "instant", // never | instant | thinking | message
      typingIntervalSeconds: 6,
    },
  },
}
```

- پیش‌فرض‌ها: `instant` برای گفت‌وگوهای مستقیم/اشاره‌ها، `message` برای گفت‌وگوهای گروهی بدون اشاره.
- پیش‌فرض `typingIntervalSeconds`: `6`.
- بازنویسی مختص عامل: `agents.entries.*.typingMode`.

به [نشانگرهای تایپ](/fa/concepts/typing-indicators) مراجعه کنید.

<a id="agentsdefaultssandbox"></a>

### `agents.defaults.sandbox`

سندباکس اختیاری برای عامل تعبیه‌شده. برای راهنمای کامل، به [سندباکس](/fa/gateway/sandboxing) مراجعه کنید.

```json5
{
  agents: {
    defaults: {
      sandbox: {
        mode: "non-main", // off (پیش‌فرض) | non-main | all
        backend: "docker", // docker (پیش‌فرض) | ssh | openshell
        scope: "agent", // session | agent (پیش‌فرض) | shared
        workspaceAccess: "none", // none (پیش‌فرض) | ro | rw
        workspaceRoot: "~/.openclaw/sandboxes",
        docker: {
          image: "openclaw-sandbox:bookworm-slim",
          containerPrefix: "openclaw-sbx-",
          workdir: "/workspace",
          readOnlyRoot: true,
          tmpfs: ["/tmp", "/var/tmp", "/run"],
          network: "none",
          user: "1000:1000",
          capDrop: ["ALL"],
          env: { LANG: "C.UTF-8" },
          setupCommand: "apt-get update && apt-get install -y git curl jq",
          pidsLimit: 256,
          memory: "1g",
          memorySwap: "2g",
          cpus: 1,
          gpus: "all",
          ulimits: {
            nofile: { soft: 1024, hard: 2048 },
            nproc: 256,
          },
          seccompProfile: "/path/to/seccomp.json",
          apparmorProfile: "openclaw-sandbox",
          dns: ["1.1.1.1", "8.8.8.8"],
          extraHosts: ["internal.service:10.0.0.5"],
          binds: ["/home/user/source:/source:rw"],
        },
        ssh: {
          target: "user@gateway-host:22",
          command: "ssh",
          workspaceRoot: "/tmp/openclaw-sandboxes",
          strictHostKeyChecking: true,
          updateHostKeys: true,
          identityFile: "~/.ssh/id_ed25519",
          certificateFile: "~/.ssh/id_ed25519-cert.pub",
          knownHostsFile: "~/.ssh/known_hosts",
          // SecretRefها / محتوای درون‌خطی نیز پشتیبانی می‌شوند:
          // identityData: { source: "env", provider: "default", id: "SSH_IDENTITY" },
          // certificateData: { source: "env", provider: "default", id: "SSH_CERTIFICATE" },
          // knownHostsData: { source: "env", provider: "default", id: "SSH_KNOWN_HOSTS" },
        },
        browser: {
          enabled: false,
          image: "openclaw-sandbox-browser:bookworm-slim",
          network: "openclaw-sandbox-browser",
          cdpPort: 9222,
          cdpSourceRange: "172.21.0.1/32",
          vncPort: 5900,
          noVncPort: 6080,
          headless: false,
          enableNoVnc: true,
          allowHostControl: false,
          autoStart: true,
          autoStartTimeoutMs: 12000,
        },
        prune: {
          idleHours: 24,
          maxAgeDays: 7,
        },
      },
    },
  },
  tools: {
    sandbox: {
      tools: {
        allow: [
          "exec",
          "process",
          "read",
          "write",
          "edit",
          "apply_patch",
          "sessions_list",
          "sessions_history",
          "sessions_send",
          "sessions_spawn",
          "session_status",
        ],
        deny: ["browser", "canvas", "nodes", "cron", "discord", "gateway"],
      },
    },
  },
}
```

پیش‌فرض‌های نشان‌داده‌شده در بالا (تصویر `off`/`docker`/`agent`/`none`/`bookworm-slim`، شبکهٔ `none` و غیره) پیش‌فرض‌های واقعی OpenClaw هستند، نه صرفاً مقادیر نمونه.

<Accordion title="جزئیات سندباکس">

**بک‌اند:**

- `docker`: زمان‌اجرای محلی Docker (پیش‌فرض)
- `ssh`: زمان‌اجرای راه‌دور عمومی مبتنی بر SSH
- `openshell`: زمان‌اجرای OpenShell

وقتی `backend: "openshell"` انتخاب شود، تنظیمات مختص زمان‌اجرا به
`plugins.entries.openshell.config` منتقل می‌شوند.

**پیکربندی بک‌اند SSH:**

- `target`: مقصد SSH با قالب `user@host[:port]`
- `command`: فرمان کلاینت SSH (پیش‌فرض: `ssh`)
- `workspaceRoot`: ریشهٔ مطلق راه‌دور برای فضاهای کاری هر محدوده (پیش‌فرض: `/tmp/openclaw-sandboxes`)
- `identityFile` / `certificateFile` / `knownHostsFile`: فایل‌های محلی موجود که به OpenSSH داده می‌شوند
- `identityData` / `certificateData` / `knownHostsData`: محتوای درون‌خطی یا SecretRefهایی که OpenClaw هنگام اجرا در فایل‌های موقت قرار می‌دهد
- `strictHostKeyChecking` / `updateHostKeys`: گزینه‌های سیاست کلید میزبان OpenSSH (مقدار پیش‌فرض هر دو `true` است)

**اولویت احراز هویت SSH:**

- `identityData` بر `identityFile` اولویت دارد
- `certificateData` بر `certificateFile` اولویت دارد
- `knownHostsData` بر `knownHostsFile` اولویت دارد
- مقادیر `*Data` مبتنی بر SecretRef پیش از آغاز نشست سندباکس، از اسنپ‌شات فعال زمان‌اجرای اسرار حل می‌شوند

**رفتار بک‌اند SSH:**

- پس از ایجاد یا ایجاد مجدد، فضای کاری راه‌دور را یک‌بار مقداردهی اولیه می‌کند
- سپس فضای کاری SSH راه‌دور را به‌عنوان مرجع اصلی نگه می‌دارد
- `exec`، ابزارهای فایل و مسیرهای رسانه را از طریق SSH مسیریابی می‌کند
- تغییرات راه‌دور را به‌طور خودکار با میزبان همگام نمی‌کند
- از کانتینرهای مرورگر سندباکس پشتیبانی نمی‌کند

**دسترسی به فضای کاری:**

- `none`: فضای کاری سندباکس هر محدوده زیر `~/.openclaw/sandboxes` (پیش‌فرض)
- `ro`: فضای کاری سندباکس در `/workspace`، فضای کاری عامل به‌صورت فقط‌خواندنی در `/agent` سوار می‌شود
- `rw`: فضای کاری عامل به‌صورت خواندنی/نوشتنی در `/workspace` سوار می‌شود

**محدوده:**

- `session`: کانتینر + فضای کاری برای هر نشست
- `agent`: یک کانتینر + فضای کاری برای هر عامل (پیش‌فرض)
- `shared`: کانتینر و فضای کاری مشترک (بدون جداسازی میان نشست‌ها)

**پیکربندی Plugin مربوط به OpenShell:**

```json5
{
  plugins: {
    entries: {
      openshell: {
        enabled: true,
        config: {
          mode: "mirror", // آینه‌ای (پیش‌فرض) | راه‌دور
          command: "openshell",
          from: "openclaw",
          remoteWorkspaceDir: "/sandbox",
          remoteAgentWorkspaceDir: "/agent",
          gateway: "lab", // اختیاری
          gatewayEndpoint: "https://lab.example", // اختیاری
          policy: "strict", // شناسهٔ اختیاری سیاست OpenShell
          providers: ["openai"], // اختیاری
          autoProviders: true,
          timeoutSeconds: 120,
        },
      },
    },
  },
}
```

**حالت OpenShell:**

- `mirror`: پیش از اجرا، راه‌دور را از محلی مقداردهی اولیه می‌کند و پس از اجرا همگام‌سازی معکوس انجام می‌دهد؛ فضای کاری محلی مرجع اصلی باقی می‌ماند
- `remote`: هنگام ایجاد سندباکس، راه‌دور را یک‌بار مقداردهی اولیه می‌کند و سپس فضای کاری راه‌دور را به‌عنوان مرجع اصلی نگه می‌دارد

در حالت `remote`، ویرایش‌های محلی میزبان که خارج از OpenClaw انجام شوند، پس از مرحلهٔ مقداردهی اولیه به‌طور خودکار با سندباکس همگام نمی‌شوند.
انتقال از طریق SSH به سندباکس OpenShell انجام می‌شود، اما Plugin چرخهٔ عمر سندباکس و همگام‌سازی آینه‌ای اختیاری را مدیریت می‌کند.

**`setupCommand`** پس از ایجاد کانتینر یک‌بار اجرا می‌شود (از طریق `sh -lc`). به خروجی شبکه، ریشهٔ قابل‌نوشتن و کاربر root نیاز دارد.

**کانتینرها به‌طور پیش‌فرض از `network: "none"` استفاده می‌کنند** — اگر عامل به دسترسی خروجی نیاز دارد، آن را روی `"bridge"` (یا یک شبکهٔ bridge سفارشی) تنظیم کنید.
`"host"` مسدود است. `"container:<id>"` نیز به‌طور پیش‌فرض مسدود است، مگر اینکه
`sandbox.docker.dangerouslyAllowContainerNamespaceJoin: true` را صریحاً تنظیم کنید (راهکار اضطراری).
نوبت‌های app-server مربوط به Codex در یک سندباکس فعال OpenClaw، برای دسترسی شبکهٔ بومی حالت کد خود از همین تنظیم خروجی استفاده می‌کنند.

**پیوست‌های ورودی** در `media/inbound/*` داخل فضای کاری فعال آماده‌سازی می‌شوند.

**`docker.binds`** دایرکتوری‌های میزبان بیشتری را سوار می‌کند؛ اتصال‌های سراسری و مخصوص هر عامل با هم ادغام می‌شوند.

**مرورگر سندباکس‌شده** (`sandbox.browser.enabled`، پیش‌فرض `false`): Chromium + CDP در یک کانتینر. نشانی noVNC به پرامپت سیستم تزریق می‌شود. در `openclaw.json` به `browser.enabled` نیاز ندارد.
دسترسی ناظر noVNC به‌طور پیش‌فرض از احراز هویت VNC استفاده می‌کند و OpenClaw یک نشانی توکن کوتاه‌عمر صادر می‌کند (به‌جای افشای گذرواژه در نشانی مشترک).

- `allowHostControl: false` (پیش‌فرض) مانع هدف‌گیری مرورگر میزبان توسط نشست‌های سندباکس‌شده می‌شود.
- مقدار پیش‌فرض `network` برابر `openclaw-sandbox-browser` است (شبکهٔ bridge اختصاصی). فقط زمانی آن را روی `bridge` تنظیم کنید که صریحاً اتصال سراسری bridge را می‌خواهید. `"host"` در اینجا نیز مسدود است.
- `cdpSourceRange` می‌تواند ورود CDP در لبهٔ کانتینر را به یک محدودهٔ CIDR محدود کند (برای مثال `172.21.0.1/32`).
- `sandbox.browser.binds` دایرکتوری‌های میزبان بیشتری را فقط در کانتینر مرورگر سندباکس سوار می‌کند. وقتی تنظیم شود (از جمله `[]`)، برای کانتینر مرورگر جایگزین `docker.binds` می‌شود.
- Chromium کانتینر مرورگر سندباکس همیشه با `--no-sandbox --disable-setuid-sandbox` راه‌اندازی می‌شود (کانتینرها سازوکارهای هسته‌ای موردنیاز سندباکس داخلی Chrome را ندارند)؛ هیچ گزینهٔ پیکربندی برای تغییر آن وجود ندارد.
- پیش‌فرض‌های راه‌اندازی در `scripts/sandbox-browser-entrypoint.sh` تعریف شده‌اند و برای میزبان‌های کانتینری تنظیم شده‌اند:
  - `--remote-debugging-address=127.0.0.1`
  - `--remote-debugging-port=<derived from OPENCLAW_BROWSER_CDP_PORT>`
  - `--user-data-dir=${HOME}/.chrome`
  - `--no-first-run`
  - `--no-default-browser-check`
  - `--disable-dev-shm-usage`
  - `--disable-background-networking`
  - `--disable-breakpad`
  - `--disable-crash-reporter`
  - `--no-zygote`
  - `--metrics-recording-only`
  - `--password-store=basic`
  - `--use-mock-keychain`
  - `--disable-3d-apis`، `--disable-gpu` و `--disable-software-rasterizer`
    به‌طور پیش‌فرض فعال هستند و اگر استفاده از WebGL/3D به آن نیاز داشته باشد،
    می‌توان آن‌ها را با `OPENCLAW_BROWSER_DISABLE_GRAPHICS_FLAGS=0` غیرفعال کرد.
  - `--disable-extensions` (به‌طور پیش‌فرض فعال)؛ اگر گردش کار به افزونه‌ها وابسته است،
    `OPENCLAW_BROWSER_DISABLE_EXTENSIONS=0` آن‌ها را دوباره فعال می‌کند.
  - به‌طور پیش‌فرض `--renderer-process-limit=2`؛ با
    `OPENCLAW_BROWSER_RENDERER_PROCESS_LIMIT=<N>` تغییر دهید، برای استفاده از محدودیت پیش‌فرض فرایندهای Chromium،
    `0` را تنظیم کنید.
  - `--headless=new` فقط وقتی `headless` فعال باشد.
  - پیش‌فرض‌ها خط مبنای ایمیج کانتینر هستند؛ برای تغییر پیش‌فرض‌های کانتینر، از یک ایمیج مرورگر سفارشی با
    نقطهٔ ورود سفارشی استفاده کنید.

</Accordion>

سندباکس مرورگر و `sandbox.docker.binds` فقط در Docker قابل استفاده‌اند.

ساخت ایمیج‌ها (از یک نسخهٔ کاری کد منبع):

```bash
scripts/sandbox-setup.sh           # ایمیج اصلی سندباکس
scripts/sandbox-browser-setup.sh   # ایمیج اختیاری مرورگر
```

برای نصب‌های npm بدون نسخهٔ کاری کد منبع، برای فرمان‌های درون‌خطی `docker build` به [سندباکس‌سازی § ایمیج‌ها و راه‌اندازی](/fa/gateway/sandboxing#images-and-setup) مراجعه کنید.

### `agents.entries` (بازنویسی‌های مخصوص هر عامل)

از `agents.entries.*.tts` استفاده کنید تا ارائه‌دهنده، صدا، مدل،
سبک یا حالت TTS خودکار اختصاصی به یک عامل بدهید. بلوک عامل به‌صورت عمیق روی
`tts` سراسری ادغام می‌شود؛ بنابراین اعتبارنامه‌های مشترک می‌توانند در یک محل باقی بمانند و هر
عامل فقط فیلدهای صدا یا ارائه‌دهندهٔ موردنیاز خود را بازنویسی کند. بازنویسی عامل فعال
برای پاسخ‌های گفتاری خودکار، `/tts audio`، `/tts status` و
ابزار عامل `tts` اعمال می‌شود. برای نمونه‌های ارائه‌دهندگان و ترتیب اولویت به
[تبدیل متن به گفتار](/fa/tools/tts#per-agent-voice-overrides) مراجعه کنید.

```json5
{
  agents: {
    list: [
      {
        id: "main",
        default: true,
        name: "عامل اصلی",
        workspace: "~/.openclaw/workspace",
        agentDir: "~/.openclaw/agents/main/agent",
        model: "anthropic/claude-opus-4-6", // یا { primary, fallbacks }
        utilityModel: "openai/gpt-5.4-mini",
        thinkingDefault: "high", // بازنویسی سطح تفکر برای هر عامل
        reasoningDefault: "on", // بازنویسی قابلیت مشاهدهٔ استدلال برای هر عامل
        fastModeDefault: false, // بازنویسی حالت سریع برای هر عامل
        params: { cacheRetention: "none" }, // پارامترهای مطابق defaults.models را بر اساس کلید بازنویسی می‌کند
        tts: {
          providers: {
            elevenlabs: { speakerVoiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
        skills: ["docs-search"], // در صورت تنظیم، جایگزین agents.defaults.skills می‌شود
        identity: {
          name: "Samantha",
          theme: "تنبلِ یاری‌رسان",
          emoji: "🦥",
          avatar: "avatars/samantha.png",
        },
        groupChat: { mentionPatterns: ["@openclaw"] },
        sandbox: { mode: "off" },
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent", // پایدار | یک‌باره
            cwd: "/workspace/openclaw",
          },
        },
        subagents: { allowAgents: ["*"] },
        tools: {
          profile: "coding",
          allow: ["browser"],
          deny: ["canvas"],
          elevated: { enabled: true },
        },
      },
    ],
  },
}
```

- `id`: شناسهٔ پایدار عامل (الزامی).
- `default`: وقتی چند مورد تنظیم شده باشند، نخستین مورد برنده می‌شود (هشدار ثبت می‌شود). اگر هیچ‌کدام تنظیم نشده باشند، نخستین ورودی فهرست پیش‌فرض است.
- `model`: قالب رشته‌ای یک مدل اصلی سخت‌گیرانه برای هر عامل، بدون مدل جایگزین، تنظیم می‌کند؛ قالب شیء `{ primary }` نیز سخت‌گیرانه است، مگر اینکه `fallbacks` را اضافه کنید. برای فعال‌کردن جایگزینی برای آن عامل از `{ primary, fallbacks: [...] }` استفاده کنید، یا برای صریح‌کردن رفتار سخت‌گیرانه از `{ primary, fallbacks: [] }` استفاده کنید. کارهای Cron که فقط `primary` را بازنویسی می‌کنند، همچنان مدل‌های جایگزین پیش‌فرض را به ارث می‌برند، مگر اینکه `fallbacks: []` را تنظیم کنید.
- `utilityModel`: بازنویسی اختیاری برای هر عامل جهت کارهای داخلی کوتاه، مانند عنوان‌های تولیدشده برای نشست و رشته. ابتدا به `agents.defaults.utilityModel` و سپس به مدل کوچک پیش‌فرض اعلام‌شده توسط ارائه‌دهندهٔ مؤثر نشست برمی‌گردد. عنوان‌های داشبورد یک‌بار دیگر با مدل عادی مؤثر نشست تلاش می‌شوند. رشتهٔ خالی، مسیر ابزار کمکی جایگزین را برای این عامل نادیده می‌گیرد، بدون اینکه تولید عنوان داشبورد را غیرفعال کند.
- `params`: پارامترهای جریان برای هر عامل که روی ورودی مدل انتخاب‌شده در `agents.defaults.models` ادغام می‌شوند. از این مورد برای بازنویسی‌های مختص عامل، مانند `cacheRetention`، `temperature` یا `maxTokens`، بدون تکرار کل کاتالوگ مدل استفاده کنید.
- `tts`: بازنویسی‌های اختیاری تبدیل متن به گفتار برای هر عامل. این بلوک به‌صورت عمیق روی `tts` ادغام می‌شود؛ بنابراین اعتبارنامه‌های مشترک ارائه‌دهنده و سیاست جایگزینی را در `tts` نگه دارید و اینجا فقط مقادیر مختص شخصیت، مانند ارائه‌دهنده، صدا، مدل، سبک یا حالت خودکار را تنظیم کنید.
- `skills`: فهرست مجاز اختیاری Skills برای هر عامل. اگر حذف شود، عامل در صورت تنظیم‌بودن `agents.defaults.skills` آن را به ارث می‌برد؛ یک فهرست صریح، به‌جای ادغام، مقادیر پیش‌فرض را جایگزین می‌کند و `[]` به‌معنای نبود Skills است.
- `thinkingDefault`: سطح پیش‌فرض اختیاری تفکر برای هر عامل (`off | minimal | low | medium | high | xhigh | adaptive | max`). وقتی هیچ بازنویسی در سطح پیام یا نشست تنظیم نشده باشد، `agents.defaults.thinkingDefault` را برای این عامل بازنویسی می‌کند. نمایهٔ ارائه‌دهنده/مدل انتخاب‌شده تعیین می‌کند کدام مقادیر معتبرند؛ برای Google Gemini، مقدار `adaptive` تفکر پویای تحت مالکیت ارائه‌دهنده را حفظ می‌کند (`thinkingLevel` در Gemini 3/3.1 حذف می‌شود، `thinkingBudget: -1` در Gemini 2.5).
- `reasoningDefault`: میزان نمایش پیش‌فرض اختیاری استدلال برای هر عامل (`on | off | stream`). وقتی هیچ بازنویسی استدلال در سطح پیام یا نشست تنظیم نشده باشد، `agents.defaults.reasoningDefault` را برای این عامل بازنویسی می‌کند.
- `fastModeDefault`: پیش‌فرض اختیاری حالت سریع برای هر عامل (`"auto" | true | false`). وقتی هیچ بازنویسی حالت سریع در سطح پیام یا نشست تنظیم نشده باشد، اعمال می‌شود.
- `models`: بازنویسی‌های اختیاری کاتالوگ مدل/زمان اجرا برای هر عامل که با شناسه‌های کامل `provider/model` کلیدگذاری می‌شوند. برای استثناهای زمان اجرای مختص عامل از `models["provider/model"].agentRuntime` استفاده کنید.
- `runtime`: توصیف‌گر اختیاری زمان اجرا برای هر عامل. وقتی عامل باید به‌طور پیش‌فرض از نشست‌های چارچوب ACP استفاده کند، از `type: "acp"` با پیش‌فرض‌های `runtime.acp` (`agent`، `backend`، `mode`، `cwd`) استفاده کنید.
- `identity.avatar`: مسیر نسبی به فضای کاری، نشانی وب `http(s)` یا URI نوع `data:`.
- فایل‌های تصویر محلی `identity.avatar` با مسیر نسبی به فضای کاری به 2 MB محدودند. نشانی‌های وب `http(s)` و URIهای `data:` با محدودیت اندازهٔ فایل محلی بررسی نمی‌شوند.
- `identity` مقادیر پیش‌فرض را استخراج می‌کند: `ackReaction` از `emoji`، و `mentionPatterns` از `name`/`emoji`.
- `subagents.allowAgents`: فهرست مجاز شناسه‌های عامل پیکربندی‌شده برای مقصدهای صریح `sessions_spawn.agentId` (`["*"]` = هر مقصد پیکربندی‌شده؛ پیش‌فرض: فقط همان عامل). وقتی فراخوانی‌های خودمقصد `agentId` باید مجاز باشند، شناسهٔ درخواست‌کننده را وارد کنید. ورودی‌های منقضی‌شده‌ای که پیکربندی عاملشان حذف شده است، توسط `sessions_spawn` رد و از `agents_list` حذف می‌شوند؛ برای پاک‌سازی آن‌ها `openclaw doctor --fix` را اجرا کنید، یا اگر آن مقصد باید ضمن به‌ارث‌بردن مقادیر پیش‌فرض همچنان قابل ایجاد باشد، یک ورودی حداقلی `agents.entries.*` اضافه کنید.
- محافظ وراثت سندباکس: اگر نشست درخواست‌کننده در سندباکس باشد، `sessions_spawn` مقصدهایی را که بدون سندباکس اجرا می‌شوند رد می‌کند.
- `subagents.requireAgentId`: وقتی true باشد، فراخوانی‌های `sessions_spawn` که `agentId` را حذف کرده‌اند مسدود می‌شوند (انتخاب صریح نمایه را اجباری می‌کند؛ پیش‌فرض: false).
- `subagents.maxConcurrent`: بیشینهٔ اجرای هم‌زمان عامل‌های فرزند در سراسر اجرای زیرعامل‌ها. پیش‌فرض: `8`.
- `subagents.maxChildrenPerAgent`: بیشینهٔ فرزندان فعالی که یک نشست عامل می‌تواند ایجاد کند. پیش‌فرض: `5`.
- `subagents.maxSpawnDepth`: بیشینهٔ عمق تودرتویی برای ایجاد زیرعامل (`1`-`5`). پیش‌فرض: `1` (بدون تودرتویی).
- `subagents.archiveAfterMinutes`: مدت‌زمانی که پس از آن وضعیت زیرعامل تکمیل‌شده بایگانی می‌شود. پیش‌فرض: `60`.

---

## مسیریابی چندعاملی

چند عامل مجزا را درون یک Gateway اجرا کنید. [چندعاملی](/fa/concepts/multi-agent) را ببینید.

```json5
{
  agents: {
    list: [
      { id: "home", default: true, workspace: "~/.openclaw/workspace-home" },
      { id: "work", workspace: "~/.openclaw/workspace-work" },
    ],
  },
  bindings: [
    { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
    { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },
  ],
}
```

### فیلدهای تطبیق اتصال

- `type` (اختیاری): `route` برای مسیریابی عادی (نوع حذف‌شده به‌طور پیش‌فرض route است)، `acp` برای اتصال‌های پایدار مکالمهٔ ACP.
- `match.channel` (الزامی)
- `match.accountId` (اختیاری؛ `*` = هر حساب؛ حذف‌شده = حساب پیش‌فرض)
- `match.peer` (اختیاری؛ `{ kind: direct|group|channel, id }`)
- `match.guildId` / `match.teamId` (اختیاری؛ مختص کانال)
- `acp` (اختیاری؛ فقط برای `type: "acp"`): `{ mode, label, cwd, backend }`

**ترتیب قطعی تطبیق:**

1. `match.peer`
2. `match.guildId`
3. `match.teamId`
4. `match.accountId` (دقیق، بدون همتا/انجمن/تیم)
5. `match.accountId: "*"` (در سراسر کانال)
6. عامل پیش‌فرض

در هر سطح، نخستین ورودی منطبق `bindings` برنده می‌شود.

برای ورودی‌های `type: "acp"`، OpenClaw بر اساس هویت دقیق مکالمه (`match.channel` + حساب + `match.peer.id`) تصمیم می‌گیرد و از ترتیب سطوح اتصال مسیریابی بالا استفاده نمی‌کند.

### نمایه‌های دسترسی برای هر عامل

<Accordion title="دسترسی کامل (بدون سندباکس)">

```json5
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: { mode: "off" },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="ابزارهای فقط‌خواندنی + فضای کاری">

```json5
{
  agents: {
    list: [
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "ro" },
        tools: {
          allow: [
            "read",
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
          ],
          deny: ["write", "edit", "apply_patch", "exec", "process", "browser"],
        },
      },
    ],
  },
}
```

</Accordion>

<Accordion title="بدون دسترسی به سیستم فایل (فقط پیام‌رسانی)">

```json5
{
  agents: {
    list: [
      {
        id: "public",
        workspace: "~/.openclaw/workspace-public",
        sandbox: { mode: "all", scope: "agent", workspaceAccess: "none" },
        tools: {
          allow: [
            "sessions_list",
            "sessions_history",
            "sessions_send",
            "sessions_spawn",
            "session_status",
            "whatsapp",
            "telegram",
            "slack",
            "discord",
            "gateway",
          ],
          deny: [
            "read",
            "write",
            "edit",
            "apply_patch",
            "exec",
            "process",
            "browser",
            "canvas",
            "nodes",
            "cron",
            "gateway",
            "image",
          ],
        },
      },
    ],
  },
}
```

</Accordion>

برای جزئیات تقدم، [سندباکس و ابزارهای چندعاملی](/fa/tools/multi-agent-sandbox-tools) را ببینید.

---

## نشست

```json5
{
  session: {
    scope: "per-sender",
    dmScope: "main", // main | per-peer | per-channel-peer | per-account-channel-peer
    identityLinks: {
      alice: ["telegram:123456789", "discord:987654321012345678"],
    },
    reset: {
      mode: "daily", // daily | idle
      atHour: 4,
      idleMinutes: 60,
    },
    resetByType: {
      thread: { mode: "daily", atHour: 4 },
      direct: { mode: "idle", idleMinutes: 240 },
      group: { mode: "idle", idleMinutes: 120 },
    },
    resetByChannel: {
      discord: { mode: "idle", idleMinutes: 30 },
    },
    resetTriggers: ["/new", "/reset"],
    store: "~/.openclaw/agents/{agentId}/sessions/sessions.json",
    maintenance: {
      mode: "enforce", // enforce (default) | warn
      pruneAfter: "30d",
      maxEntries: 500,
      resetArchiveRetention: "30d", // duration or false
      maxDiskBytes: "500mb", // optional hard budget
      highWaterBytes: "400mb", // optional cleanup target
    },
    threadBindings: {
      enabled: true,
      idleHours: 24, // default inactivity auto-unfocus in hours (`0` disables)
      maxAgeHours: 0, // default hard max age in hours (`0` disables)
    },
    sharing: {
      readOnly: true,
      suggest: true,
      drafts: true,
    },
    mainKey: "main", // legacy (runtime always uses "main")
    sendPolicy: {
      rules: [{ action: "deny", match: { channel: "discord", chatType: "group" } }],
      default: "allow",
    },
  },
}
```

<Accordion title="جزئیات فیلدهای نشست">

- **`scope`**: راهبرد پایهٔ گروه‌بندی نشست برای زمینه‌های گفت‌وگوی گروهی.
  - `per-sender` (پیش‌فرض): هر فرستنده در یک زمینهٔ کانال، نشست مجزایی دریافت می‌کند.
  - `global`: همهٔ شرکت‌کنندگان در یک زمینهٔ کانال، یک نشست واحد را به‌اشتراک می‌گذارند (فقط زمانی استفاده شود که زمینهٔ مشترک مدنظر است).
- **`dmScope`**: نحوهٔ گروه‌بندی پیام‌های مستقیم.
  - `main`: همهٔ پیام‌های مستقیم، نشست اصلی را به‌اشتراک می‌گذارند.
  - `per-peer`: جداسازی بر اساس شناسهٔ فرستنده در کانال‌های مختلف.
  - `per-channel-peer`: جداسازی بر اساس کانال + فرستنده (برای صندوق‌های ورودی چندکاربره توصیه می‌شود).
  - `per-account-channel-peer`: جداسازی بر اساس حساب + کانال + فرستنده (برای حالت چندحسابی توصیه می‌شود).
- **`identityLinks`**: نگاشت شناسه‌های کانونی به همتایان دارای پیشوند ارائه‌دهنده برای اشتراک‌گذاری نشست میان کانال‌ها. فرمان‌های اتصال مانند `/dock_discord` از همین نگاشت استفاده می‌کنند تا مسیر پاسخ نشست فعال را به همتای کانال پیوندخوردهٔ دیگری تغییر دهند؛ به [اتصال کانال](/fa/concepts/channel-docking) مراجعه کنید.
- **`reset`**: سیاست اصلی بازنشانی. `none` بازنشانی خودکار را غیرفعال می‌کند و حالت پیش‌فرض است؛ در عوض Compaction زمینهٔ فعال را محدود می‌کند. `daily` در ساعت محلی `atHour` بازنشانی می‌کند؛ `idle` پس از `idleMinutes` بازنشانی می‌کند. اگر هر دو پیکربندی شده باشند، هرکدام زودتر منقضی شود اعمال می‌شود. `/new` و `/reset` در همهٔ حالت‌ها در دسترس می‌مانند. تازگی بازنشانی روزانه از `sessionStartedAt` ردیف نشست استفاده می‌کند؛ تازگی بازنشانی بر اثر بی‌کاری از `lastInteractionAt` استفاده می‌کند. نوشتن‌های پس‌زمینه/رویداد سیستمی مانند Heartbeat، بیدارباش‌های Cron، اعلان‌های اجرا و ثبت امور Gateway می‌توانند `updatedAt` را به‌روزرسانی کنند، اما نشست‌های روزانه/بی‌کار را تازه نگه نمی‌دارند.
  - **`resetByType`**: بازنویسی‌های مختص هر نوع (`direct`، `group`، `thread`). Doctor ورودی‌های قدیمی `dm` را به `direct` مهاجرت می‌دهد؛ طرح‌واره `dm` را رد می‌کند.
- **`resetByChannel`**: بازنویسی‌های بازنشانی مختص هر کانال که با شناسهٔ ارائه‌دهنده/کانال کلیدگذاری شده‌اند. وقتی کانال نشست ورودی منطبق داشته باشد، برای آن نشست بدون قیدوشرط بر `resetByType`/`reset` اولویت می‌یابد. فقط زمانی استفاده شود که یک کانال به رفتار بازنشانی متفاوتی از سیاست سطح نوع نیاز دارد.
- **`mainKey`**: فیلد قدیمی. زمان اجرا همیشه از `"main"` برای سطل اصلی گفت‌وگوی مستقیم استفاده می‌کند.
- **`sendPolicy`**: تطبیق بر اساس `channel`، `chatType` (`direct|group|channel`، با نام مستعار قدیمی `dm`)، `keyPrefix` یا `rawKeyPrefix`. نخستین منع، اعمال می‌شود.
- **`maintenance`**: کنترل‌های پاک‌سازی + نگهداشت مخزن نشست.
  - `mode`: `enforce` پاک‌سازی را اعمال می‌کند و حالت پیش‌فرض است؛ `warn` فقط هشدار صادر می‌کند.
  - `pruneAfter`: حد آستانهٔ سنی برای ورودی‌های کهنه (پیش‌فرض `30d`).
  - `maxEntries`: حداکثر تعداد ورودی‌های نشست SQLite (پیش‌فرض `500`). نوشتن‌های زمان اجرا، پاک‌سازی دسته‌ای را با یک حاشیهٔ کوچک سقف بالا برای محدودیت‌های در مقیاس محیط عملیاتی انجام می‌دهند؛ `openclaw sessions cleanup --enforce` محدودیت را بلافاصله اعمال می‌کند.
  - نشست‌های کوتاه‌عمر بررسی اجرای مدل Gateway از نگهداشت ثابت `24h` استفاده می‌کنند، اما پاک‌سازی وابسته به فشار است: تنها زمانی ردیف‌های کهنهٔ بررسی صریح اجرای مدل را حذف می‌کند که فشار نگهداشت/محدودیت ورودی‌های نشست ایجاد شده باشد. فقط کلیدهای بررسی صریح و دقیق منطبق با `agent:*:explicit:model-run-<uuid>` واجد شرایط‌اند؛ نشست‌های عادی مستقیم، گروهی، رشته‌ای، Cron، قلاب، Heartbeat، ACP و عامل فرعی این نگهداشت 24h را به ارث نمی‌برند. وقتی پاک‌سازی اجرای مدل انجام می‌شود، پیش از پاک‌سازی گسترده‌تر ورودی‌های کهنهٔ `pruneAfter` و محدودیت `maxEntries` اجرا می‌شود.
  - `rotateBytes` قدیمی توسط طرح‌وارهٔ فعلی رد می‌شود؛ `openclaw doctor --fix` آن را از پیکربندی‌های قدیمی‌تر حذف می‌کند.
  - `resetArchiveRetention`: نگهداشت مبتنی بر سن برای بایگانی‌های رونوشت بازنشانی‌شده/حذف‌شده. به‌طور پیش‌فرض، بایگانی‌ها تا زمان بیرون‌رانی بر اساس بودجهٔ دیسک باقی می‌مانند؛ برای فعال‌سازی حذف بر اساس زمان واقعی یک مدت تعیین کنید، یا برای غیرفعال‌سازی صریح آن `false` را تنظیم کنید.
  - `maxDiskBytes`: بودجهٔ اختیاری دیسک برای شاخهٔ نشست‌ها. در حالت `warn` هشدارها را ثبت می‌کند؛ در حالت `enforce` ابتدا قدیمی‌ترین مصنوعات/نشست‌ها را حذف می‌کند.
  - `highWaterBytes`: هدف اختیاری پس از پاک‌سازی بودجه. مقدار پیش‌فرض، `80%` از `maxDiskBytes` است.
- **`threadBindings`**: پیش‌فرض‌های سراسری برای قابلیت‌های نشست مقید به رشته.
  - `enabled`: کلید اصلی اتصال نشست به رشته در کانال‌های پشتیبانی‌شده
  - `idleHours`: لغو تمرکز خودکار پیش‌فرض پس از عدم فعالیت، بر حسب ساعت (`0` غیرفعال می‌کند؛ ارائه‌دهندگان می‌توانند بازنویسی کنند)
  - `maxAgeHours`: حداکثر سن قطعی پیش‌فرض بر حسب ساعت (`0` غیرفعال می‌کند؛ ارائه‌دهندگان می‌توانند بازنویسی کنند)
  - `spawnSessions`: دروازهٔ پیش‌فرض برای ایجاد نشست‌های کاری مقید به رشته از `sessions_spawn` و ایجاد رشته‌های ACP. وقتی اتصال رشته‌ها فعال باشد، مقدار پیش‌فرض `true` است؛ ارائه‌دهندگان/حساب‌ها می‌توانند بازنویسی کنند.
  - `defaultSpawnContext`: زمینهٔ بومی پیش‌فرض عامل فرعی برای ایجادهای مقید به رشته (`"fork"` یا `"isolated"`). مقدار پیش‌فرض `"fork"` است.
- **`sharing`**: کنترل می‌کند مالکان و اتصال‌های `operator.admin` کدام حالت‌های همکاری مختص نشست را می‌توانند انتخاب کنند. مقدار پیش‌فرض همهٔ پرچم‌ها `true` است؛ تنظیم یکی از آن‌ها روی `false` آن گزینه را از رابط کاربری کنترل حذف می‌کند و باعث می‌شود نمایانی هنگام ایجاد یا `session.visibility.set` آن را رد کند. نشست‌های جدید با `shared` آغاز می‌شوند، مگر اینکه رابط کاربری کنترل یکی را به‌صورت پیش‌نویس آغاز کند.
  - `readOnly`: اجازهٔ `read-only`، که در آن افراد غیرعضو می‌توانند تماشا کنند اما نمی‌توانند ارسال کنند، هدایت کنند، متوقف کنند، تأیید کنند یا وضعیت نشست را تغییر دهند.
  - `suggest`: اجازهٔ `suggest`. در این مرحله همان رفتار پذیرش `read-only` را اعمال می‌کند؛ صف پیشنهاد قابلیتی برای آینده است.
  - `drafts`: اجازهٔ `draft`، که نشست را از فهرست نشست‌ها و انتشار رویدادها برای افراد غیرمدیر و غیرمالک پنهان می‌کند.

تغییرات عضویت و نمایانی به‌صورت یادداشت‌های سیستمی در رونوشت نشست نوشته می‌شوند. این کنترل‌ها اپراتورهایی را هماهنگ می‌کنند که یک عامل را به‌اشتراک می‌گذارند؛ این‌ها مرز امنیتی میان مستأجران نیستند. وقتی کار به جداسازی نیاز دارد، از Gatewayها یا عامل‌های جداگانه استفاده کنید.

</Accordion>

---

## پیام‌ها

```json5
{
  messages: {
    responsePrefix: "🦞", // یا "auto"
    ackReaction: "👀",
    ackReactionScope: "group-mentions", // group-mentions | group-all | direct | all | off | none
    queue: {
      mode: "steer", // steer (پیش‌فرض) | followup | collect | interrupt
      debounceMs: 500,
      cap: 20,
      drop: "summarize", // old | new | summarize (پیش‌فرض)
      byChannel: {
        whatsapp: "followup",
        telegram: "followup",
      },
    },
    inbound: {
      debounceMs: 2000, // 0 غیرفعال می‌کند
      byChannel: {
        whatsapp: 5000,
        slack: 1500,
      },
    },
  },
}
```

### پیشوند پاسخ

بازنویسی‌های مختص کانال/حساب: `channels.<channel>.responsePrefix`، `channels.<channel>.accounts.<id>.responsePrefix`.

ترتیب حل (مشخص‌ترین مورد اولویت دارد): حساب ← کانال ← سراسری. `""` غیرفعال می‌کند و زنجیره را متوقف می‌سازد. `"auto"`، `[{identity.name}]` را استخراج می‌کند.

**متغیرهای الگو:**

| متغیر          | توضیح            | نمونه                     |
| ----------------- | ---------------------- | --------------------------- |
| `{model}`         | نام کوتاه مدل       | `claude-opus-4-6`           |
| `{modelFull}`     | شناسهٔ کامل مدل  | `anthropic/claude-opus-4-6` |
| `{provider}`      | نام ارائه‌دهنده          | `anthropic`                 |
| `{thinkingLevel}` | سطح فعلی تفکر | `high`، `low`، `off`        |
| `{identity.name}` | نام هویت عامل    | (همان `"auto"`)          |

متغیرها به بزرگی و کوچکی حروف حساس نیستند. `{think}` نام مستعار `{thinkingLevel}` است.

### واکنش تأیید دریافت

- مقدار پیش‌فرض، `identity.emoji` عامل فعال و در غیر این صورت `"👀"` است. برای غیرفعال‌سازی، `""` را تنظیم کنید.
- بازنویسی‌های مختص کانال: `channels.<channel>.ackReaction`، `channels.<channel>.accounts.<id>.ackReaction`.
- ترتیب حل: حساب ← کانال ← `messages.ackReaction` ← بازگشت به هویت.
- دامنه: `group-mentions` (پیش‌فرض)، `group-all`، `direct`، `all` یا `off`/`none` (واکنش‌های تأیید دریافت را کاملاً غیرفعال می‌کند).
- `messages.statusReactions.enabled`: واکنش‌های وضعیت چرخهٔ عمر را در Slack، Discord، Signal، Telegram و WhatsApp فعال می‌کند.
  در Discord، تنظیم‌نشدن این گزینه باعث می‌شود وقتی واکنش‌های تأیید دریافت فعال‌اند، واکنش‌های وضعیت نیز فعال بمانند.
  در Slack، Signal، Telegram و WhatsApp، برای فعال‌سازی واکنش‌های وضعیت چرخهٔ عمر، آن را صریحاً روی `true` تنظیم کنید.
  Slack به‌طور پیش‌فرض برای نمایش پیشرفت از وضعیت بومی رشتهٔ دستیار و پیام‌های بارگذاری چرخشی استفاده می‌کند، درحالی‌که واکنش تأیید دریافت پیکربندی‌شده را ثابت نگه می‌دارد.

### صف

- `mode`: راهبرد صف برای پیام‌های ورودی که هنگام فعال‌بودن اجرای نشست می‌رسند. پیش‌فرض: `"steer"`.
  - `steer`: درخواست جدید را به اجرای فعال تزریق می‌کند.
  - `followup`: درخواست جدید را پس از پایان اجرای فعال اجرا می‌کند.
  - `collect`: پیام‌های سازگار را دسته‌بندی می‌کند و بعداً با هم اجرا می‌کند.
  - `interrupt`: پیش از آغاز جدیدترین درخواست، اجرای فعال را متوقف می‌کند.
- `debounceMs`: تأخیر پیش از ارسال پیام صف‌شده/هدایت‌شده. پیش‌فرض: `500`.
- `cap`: حداکثر پیام‌های صف‌شده پیش از اعمال سیاست حذف. پیش‌فرض: `20`.
- `drop`: راهبرد هنگام عبور از سقف. `"summarize"` (پیش‌فرض) قدیمی‌ترین ورودی‌ها را حذف می‌کند اما خلاصه‌های فشرده را نگه می‌دارد؛ `"old"` قدیمی‌ترین‌ها را بدون خلاصه حذف می‌کند؛ `"new"` جدیدترین مورد را رد می‌کند.
- `byChannel`: بازنویسی‌های مختص کانال `mode` که با شناسهٔ ارائه‌دهنده کلیدگذاری شده‌اند.
- `debounceMsByChannel`: بازنویسی‌های مختص کانال `debounceMs` که با شناسهٔ ارائه‌دهنده کلیدگذاری شده‌اند.

### حذف پرش ورودی

پیام‌های سریع و صرفاً متنی از یک فرستنده را در یک نوبت واحد عامل دسته‌بندی می‌کند. رسانه/پیوست‌ها بلافاصله دسته را ارسال می‌کنند. فرمان‌های کنترلی از حذف پرش عبور می‌کنند. مقدار پیش‌فرض `debounceMs`: `2000`.

### سایر کلیدهای پیام

- `channels.whatsapp.responsePrefix`: پیشوند پاسخ خروجی WhatsApp. Doctor تنها زمانی مقدار بازنشستهٔ ورودی `messagePrefix` را به اینجا منتقل می‌کند که این مقدار کانونی تنظیم نشده باشد.
- `messages.visibleReplies`: پاسخ‌های قابل‌مشاهدهٔ منبع را در مکالمه‌های مستقیم، گروهی و کانالی کنترل می‌کند (`"message_tool"` برای خروجی قابل‌مشاهده به `message(action=send)` نیاز دارد؛ `"automatic"` مانند گذشته پاسخ‌های عادی را ارسال می‌کند).
- `messages.usageTemplate` / `messages.responseUsage`: الگوی سفارشی پانوشت `/usage` و حالت پیش‌فرض استفاده در هر پاسخ (`off | tokens | full`، به‌علاوهٔ نام مستعار قدیمی `on` برای `tokens`).
- `messages.groupChat.mentionPatterns` / `historyLimit`: محرک‌های اشاره در پیام‌های گروهی و اندازهٔ پنجرهٔ تاریخچه.
- `messages.suppressToolErrors`: وقتی `true` باشد، هشدارهای خطای ابزار `⚠️` را که به کاربر نمایش داده می‌شوند پنهان می‌کند (عامل همچنان خطاها را در زمینه می‌بیند و می‌تواند دوباره تلاش کند). پیش‌فرض: `false`.

### TTS (تبدیل متن به گفتار)

```json5
{
  tts: {
    auto: "off", // off (default) | always | inbound | tagged
    mode: "final", // final | all
    provider: "elevenlabs",
    summaryModel: "openai/gpt-5.4-mini",
    modelOverrides: { enabled: true },
    maxTextLength: 4000,
    timeoutMs: 30000,
    providers: {
      elevenlabs: {
        apiKey: "example-elevenlabs-api-key",
        baseUrl: "https://api.elevenlabs.io",
        speakerVoiceId: "voice_id",
        modelId: "eleven_multilingual_v2",
        seed: 42,
        applyTextNormalization: "auto",
        languageCode: "en",
        voiceSettings: {
          stability: 0.5,
          similarityBoost: 0.75,
          style: 0.0,
          useSpeakerBoost: true,
          speed: 1.0,
        },
      },
      microsoft: {
        speakerVoice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
      },
      openai: {
        apiKey: "example-openai-api-key",
        baseUrl: "https://api.openai.com/v1",
        model: "gpt-4o-mini-tts",
        speakerVoice: "coral",
      },
    },
  },
}
```

مسیر سراسری تنظیمات ترجیحی بخشی از وضعیت ماشین است (مقدار پیش‌فرض
`~/.openclaw/settings/tts.json`؛ با `OPENCLAW_TTS_PREFS` بازنویسی می‌شود). پیکربندی‌های پیشرفته
چندعاملی می‌توانند برای مخزن‌های مجزای تنظیمات ترجیحی
هر عامل، `agents.entries.<id>.tts.prefsPath` را تنظیم کنند.

- `auto` حالت پیش‌فرض تبدیل خودکار متن به گفتار را کنترل می‌کند: `off`، `always`، `inbound` یا `tagged`. ‏`/tts on|off` می‌تواند تنظیمات ترجیحی محلی را بازنویسی کند و `/tts status` وضعیت مؤثر را نشان می‌دهد.
- `summaryModel` برای خلاصه‌سازی خودکار، `agents.defaults.model.primary` را بازنویسی می‌کند.
- `modelOverrides` به‌طور پیش‌فرض فعال است (`enabled !== false`)؛ استفاده از `modelOverrides.allowProvider` اختیاری است و باید صریحاً فعال شود.
- کلیدهای API در صورت نبود مقدار، از `ELEVENLABS_API_KEY`/`XI_API_KEY` و `OPENAI_API_KEY` استفاده می‌کنند.
- ارائه‌دهندگان گفتار همراه، تحت مالکیت Plugin هستند. اگر `plugins.allow` تنظیم شده است، هر Plugin ارائه‌دهنده TTS را که می‌خواهید استفاده کنید در آن بگنجانید؛ برای مثال، `microsoft` برای Edge TTS. شناسه قدیمی ارائه‌دهنده، یعنی `edge`، به‌عنوان نام مستعار `microsoft` پذیرفته می‌شود.
- `providers.openai.baseUrl` نقطه پایانی TTS متعلق به OpenAI را بازنویسی می‌کند. ترتیب تفکیک ابتدا پیکربندی، سپس `OPENAI_TTS_BASE_URL` و پس از آن `https://api.openai.com/v1` است.
- وقتی `providers.openai.baseUrl` به یک نقطه پایانی غیر OpenAI اشاره می‌کند، OpenClaw آن را سرور TTS سازگار با OpenAI در نظر می‌گیرد و اعتبارسنجی مدل/صدا را آسان‌گیرانه‌تر می‌کند.

---

## مکالمه

مقادیر پیش‌فرض حالت مکالمه (macOS/iOS/Android و رابط کنترل مرورگر).

```json5
{
  talk: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        speakerVoiceId: "elevenlabs_voice_id",
        voiceAliases: {
          Clawd: "EXAVITQu4vr4xnSDxMaL",
          Roger: "CwhRBWXzGAHq8TQ4Fs17",
        },
        modelId: "eleven_multilingual_v2",
        outputFormat: "mp3_44100_128",
        apiKey: "elevenlabs_api_key",
      },
      mlx: {
        modelId: "mlx-community/Soprano-80M-bf16",
      },
      system: {},
    },
    consultThinkingLevel: "low",
    consultFastMode: true,
    speechLocale: "ru-RU",
    silenceTimeoutMs: 1500,
    interruptOnSpeech: true,
    realtime: {
      provider: "openai",
      providers: {
        openai: {
          model: "gpt-realtime-2.1",
          speakerVoice: "cedar",
        },
      },
      instructions: "Speak warmly and keep answers brief.",
      mode: "realtime", // realtime | stt-tts | transcription
      transport: "webrtc", // webrtc | provider-websocket | gateway-relay | managed-room
      vadThreshold: 0.5,
      silenceDurationMs: 500,
      prefixPaddingMs: 300,
      reasoningEffort: "medium",
      brain: "agent-consult", // agent-consult | direct-tools | none
    },
  },
}
```

- وقتی چند ارائه‌دهنده مکالمه پیکربندی شده‌اند، `talk.provider` باید با یکی از کلیدهای `talk.providers` مطابقت داشته باشد.
- کلیدهای مسطح قدیمی مکالمه (`talk.voiceId`، `talk.voiceAliases`، `talk.modelId`، `talk.outputFormat`، `talk.apiKey`) فقط برای سازگاری هستند. برای بازنویسی پیکربندی ذخیره‌شده به `talk.providers.<provider>`، دستور `openclaw doctor --fix` را اجرا کنید.
- شناسه‌های صدا در صورت نبود مقدار، از `ELEVENLABS_VOICE_ID` یا `SAG_VOICE_ID` استفاده می‌کنند (رفتار کلاینت مکالمه macOS).
- `providers.*.apiKey` رشته‌های متن ساده یا اشیای SecretRef را می‌پذیرد.
- استفاده جایگزین از `ELEVENLABS_API_KEY` فقط زمانی اعمال می‌شود که هیچ کلید API مکالمه‌ای پیکربندی نشده باشد.
- `providers.*.voiceAliases` به دستورالعمل‌های مکالمه اجازه می‌دهد از نام‌های کاربرپسند استفاده کنند.
- `providers.mlx.modelId` مخزن Hugging Face مورد استفاده کمک‌کننده محلی MLX در macOS را انتخاب می‌کند. اگر مشخص نشود، macOS از `mlx-community/Soprano-80M-bf16` استفاده می‌کند.
- پخش MLX در macOS، در صورت وجود، از طریق کمک‌کننده همراه `openclaw-mlx-tts` یا از طریق یک فایل اجرایی موجود در `PATH` انجام می‌شود؛ `OPENCLAW_MLX_TTS_BIN` مسیر کمک‌کننده را برای توسعه بازنویسی می‌کند.
- `consultThinkingLevel` سطح تفکر را برای اجرای کامل عامل OpenClaw در پس‌زمینه فراخوانی‌های `openclaw_agent_consult` مکالمه بلادرنگ رابط کنترل، تعیین می‌کند. برای حفظ رفتار عادی نشست/مدل، آن را تنظیم‌نشده بگذارید.
- `consultFastMode` یک بازنویسی یک‌باره حالت سریع را برای مشورت‌های بلادرنگ مکالمه رابط کنترل تنظیم می‌کند، بدون آنکه تنظیم عادی حالت سریع نشست را تغییر دهد.
- `speechLocale` شناسه محلی BCP 47 مورد استفاده تشخیص گفتار مکالمه در Android، iOS و macOS را تنظیم می‌کند. Android همچنین از مؤلفه زبان آن برای هدایت رونویسی ورودی بلادرنگ استفاده می‌کند. برای استفاده از مقدار پیش‌فرض دستگاه، آن را تنظیم‌نشده بگذارید.
- `silenceTimeoutMs` مدت انتظار حالت مکالمه پس از سکوت کاربر، پیش از ارسال رونوشت را کنترل می‌کند. تنظیم‌نکردن آن، بازه مکث پیش‌فرض پلتفرم (`700 ms on macOS and Android, 900 ms on iOS`) را حفظ می‌کند.
- `realtime.instructions` دستورالعمل‌های سیستمی مختص ارائه‌دهنده را به اعلان بلادرنگ داخلی OpenClaw می‌افزاید تا سبک صدا بدون از دست دادن راهنمایی پیش‌فرض `openclaw_agent_consult` قابل پیکربندی باشد.
- `realtime.vadThreshold` آستانه فعالیت صوتی ارائه‌دهنده را از `0` (حساس‌ترین) تا `1` (کم‌حساس‌ترین) تنظیم می‌کند. تنظیم‌نکردن آن، مقدار پیش‌فرض ارائه‌دهنده را حفظ می‌کند.
- `realtime.silenceDurationMs` بازه سکوت با عدد صحیح مثبت را پیش از ثبت نوبت بلادرنگ کاربر توسط ارائه‌دهنده تنظیم می‌کند. تنظیم‌نکردن آن، مقدار پیش‌فرض ارائه‌دهنده را حفظ می‌کند.
- `realtime.prefixPaddingMs` مقدار صوت نگه‌داری‌شده پیش از آغاز گفتار تشخیص‌داده‌شده را به‌صورت عدد صحیح نامنفی تنظیم می‌کند. تنظیم‌نکردن آن، مقدار پیش‌فرض ارائه‌دهنده را حفظ می‌کند.
- `realtime.reasoningEffort` سطح استدلال مختص ارائه‌دهنده را برای نشست‌های بلادرنگ تنظیم می‌کند. تنظیم‌نکردن آن، مقدار پیش‌فرض ارائه‌دهنده را حفظ می‌کند.
- `realtime.consultRouting`: ‏`"provider-direct"` (پیش‌فرض) پاسخ‌های مستقیم ارائه‌دهنده را زمانی حفظ می‌کند که ارائه‌دهنده بلادرنگ، رونوشت نهایی کاربر را بدون `openclaw_agent_consult` تولید کند. در عوض، `"force-agent-consult"` درخواست نهایی‌شده را از طریق OpenClaw مسیریابی می‌کند.

---

## مرتبط

- [مرجع پیکربندی](/fa/gateway/configuration-reference) — همه کلیدهای پیکربندی دیگر
- [پیکربندی](/fa/gateway/configuration) — کارهای رایج و راه‌اندازی سریع
- [نمونه‌های پیکربندی](/fa/gateway/configuration-examples)
