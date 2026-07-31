---
read_when:
    - برای /new، /reset، /stop و رویدادهای چرخهٔ عمر عامل، اتوماسیون رویدادمحور می‌خواهید
    - می‌خواهید هوک‌ها را بسازید، نصب کنید یا اشکال‌زدایی کنید
summary: 'قلاب‌ها: خودکارسازی رویدادمحور برای فرمان‌ها و رویدادهای چرخهٔ حیات'
title: هوک‌ها
x-i18n:
    generated_at: "2026-07-27T14:50:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 039a55cca60e0005d7b9c4d950a86aceb6e7c29d5768108b34011bfc21c85be6
    source_path: automation/hooks.md
    workflow: 16
---

Hookها اسکریپت‌های کوچکی هستند که هنگام رخ‌دادن رویدادهای عامل، داخل Gateway اجرا می‌شوند: فرمان‌هایی مانند `/new`، `/reset`، `/stop`، Compaction نشست، چرخهٔ حیات Gateway و جریان پیام. آن‌ها از دایرکتوری‌ها شناسایی و با `openclaw hooks` مدیریت می‌شوند. Gateway تنها پس از فعال‌کردن Hookها یا پیکربندی حداقل یک ورودی Hook، بستهٔ Hook، کنترل‌گر قدیمی یا دایرکتوری اضافی Hook، Hookهای داخلی را بارگذاری می‌کند.

در OpenClaw دو نوع Hook وجود دارد:

- **Hookهای داخلی** (این صفحه): هنگام رخ‌دادن رویدادهای عامل، داخل Gateway اجرا می‌شوند.
- **Webhookها**: نقاط پایانی HTTP خارجی که به سامانه‌های دیگر امکان می‌دهند کاری را در OpenClaw آغاز کنند. به [Webhookها](/fa/automation/cron-jobs#webhooks) مراجعه کنید.

Hookها همچنین می‌توانند داخل Pluginها بسته‌بندی شوند. `openclaw hooks list` هم Hookهای مستقل و هم Hookهای مدیریت‌شده توسط Plugin را نشان می‌دهد (که به‌شکل `plugin:<id>` نمایش داده می‌شوند).

## سطح مناسب را انتخاب کنید

OpenClaw چندین سطح توسعه‌پذیری دارد که مشابه به نظر می‌رسند، اما مسائل متفاوتی را حل می‌کنند:

| اگر می‌خواهید...                                                                                                     | استفاده کنید از...                                | دلیل                                                                                           |
| --------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | --------------------------------------------------------------------------------------------- |
| در `/new` یک اسنپ‌شات ذخیره کنید، `/reset` را ثبت کنید، پس از `message:sent` یک API خارجی را فراخوانی کنید یا خودکارسازی کلی اپراتوری بیفزایید | Hookهای داخلی (`HOOK.md`، این صفحه) | Hookهای مبتنی بر فایل برای عوارض جانبی مدیریت‌شده توسط اپراتور و خودکارسازی فرمان/چرخهٔ حیات طراحی شده‌اند |
| پرامپت‌ها را بازنویسی کنید، ابزارها را مسدود کنید، پیام‌های خروجی را لغو کنید یا میان‌افزار/سیاست ترتیبی بیفزایید                              | Hookهای نوع‌دار Plugin از طریق `api.on(...)`  | Hookهای نوع‌دار قراردادهای صریح، اولویت‌ها، قواعد ادغام و معناشناسی مسدودسازی/لغو دارند      |
| خروجی صرفاً تله‌متری یا مشاهده‌پذیری بیفزایید                                                                            | رویدادهای تشخیصی                     | مشاهده‌پذیری یک گذرگاه رویداد جداگانه است، نه سطح Hook سیاستی                              |

هنگامی از Hookهای داخلی استفاده کنید که خودکارسازی‌ای می‌خواهید که مانند یک یکپارچه‌سازی کوچک نصب‌شده عمل کند. هنگامی از Hookهای نوع‌دار Plugin استفاده کنید که به کنترل چرخهٔ حیات زمان اجرا نیاز دارید.

## شروع سریع

```bash
# فهرست Hookهای موجود
openclaw hooks list

# فعال‌کردن یک Hook
openclaw hooks enable session-memory

# بررسی وضعیت Hook
openclaw hooks check

# دریافت اطلاعات تفصیلی
openclaw hooks info session-memory
```

## انواع رویداد

Hookها برای دریافت هر کنش در یک خانواده، در یک کلید مشخص از این جدول یا در نام خام خانواده
(`command`، `session`، `agent`، `gateway`، `message`) مشترک می‌شوند.
هستهٔ OpenClaw هیچ رویداد دیگری منتشر نمی‌کند؛ بنابراین هر نام دیگری تقریباً
همیشه یک غلط تایپی است که Hook را بی‌سروصدا غیرفعال باقی می‌گذارد (تنها Pluginی که یک
رویداد سفارشی منتشر کند می‌تواند آن را فعال کند). بارگذار Hook برای چنین نام‌هایی
هشدار ثبت می‌کند (برای مثال `command:nwe`) و `openclaw hooks info <name>` آن‌ها را علامت‌گذاری می‌کند؛ بنابراین
Hookی که هرگز اجرا نمی‌شود، قابل عیب‌یابی است.

| رویداد                    | زمان فعال‌شدن                                              |
| ------------------------ | ---------------------------------------------------------- |
| `command:new`            | صدور فرمان `/new`                                      |
| `command:reset`          | صدور فرمان `/reset`                                    |
| `command:stop`           | صدور فرمان `/stop`                                     |
| `command`                | هر رویداد فرمانی (شنوندهٔ عمومی)                       |
| `session:compact:before` | پیش از آنکه Compaction تاریخچه را خلاصه کند                       |
| `session:compact:after`  | پس از تکمیل Compaction                                 |
| `session:patch`          | هنگام تغییر ویژگی‌های نشست                       |
| `agent:bootstrap`        | پیش از تزریق فایل‌های راه‌اندازی فضای کاری              |
| `gateway:startup`        | پس از شروع کانال‌ها و بارگذاری Hookها                  |
| `gateway:shutdown`       | هنگام آغاز خاموش‌شدن Gateway                               |
| `gateway:pre-restart`    | پیش از راه‌اندازی مجدد مورد انتظار Gateway                         |
| `message:received`       | پیام ورودی از هر کانال                           |
| `message:transcribed`    | پس از تکمیل رونویسی صوتی                        |
| `message:preprocessed`   | پس از تکمیل یا ردشدن پیش‌پردازش رسانه و پیوند |
| `message:sent`           | تلاش برای ارسال خروجی (`context.success` نتیجه را دربردارد) |

## نوشتن Hookها

### ساختار Hook

هر Hook یک دایرکتوری شامل دو فایل است:

```text
my-hook/
├── HOOK.md          # فراداده + مستندات
└── handler.ts       # پیاده‌سازی کنترل‌گر
```

فایل کنترل‌گر می‌تواند `handler.ts`، `handler.js`، `index.ts` یا `index.js` باشد.

### قالب HOOK.md

```markdown
---
name: my-hook
description: "توضیحی کوتاه دربارهٔ کاری که این Hook انجام می‌دهد"
metadata:
  { "openclaw": { "emoji": "🔗", "events": ["command:new"], "requires": { "bins": ["node"] } } }
---

# Hook من

مستندات تفصیلی در اینجا قرار می‌گیرد.
```

**فیلدهای فراداده** (`metadata.openclaw`):

| فیلد      | توضیحات                                          |
| ---------- | ---------------------------------------------------- |
| `emoji`    | ایموجی نمایشی برای CLI                                |
| `events`   | آرایهٔ رویدادهایی که باید شنیده شوند                        |
| `export`   | خروجی نام‌داری که باید استفاده شود (پیش‌فرض `"default"`)        |
| `os`       | پلتفرم‌های الزامی (برای مثال `["darwin", "linux"]`)     |
| `requires` | مسیرهای الزامی `bins`، `anyBins`، `env` یا `config` |
| `always`   | دورزدن بررسی‌های واجدشرایط‌بودن (بولی)                  |
| `hookKey`  | بازنویسی کلید پیکربندی (پیش‌فرض نام Hook)      |
| `homepage` | نشانی URL مستندات که توسط `openclaw hooks info` نمایش داده می‌شود              |
| `install`  | روش‌های نصب                                 |

### پیاده‌سازی کنترل‌گر

```typescript
const handler = async (event) => {
  if (event.type !== "command" || event.action !== "new") {
    return;
  }

  console.log(`[my-hook] فرمان جدید فعال شد`);
  // منطق شما در اینجا

  // در صورت تمایل، در سطوح پاسخ‌پذیر پاسخی ارسال کنید
  event.messages.push("Hook اجرا شد!");
};

export default handler;
```

هر رویداد شامل این موارد است: `type`، `action`، `sessionKey`، `timestamp`، `messages` و `context` (داده‌های مختص رویداد). زمینه‌های Hook نوع‌دار Plugin برای Hookهای عامل و ابزار می‌توانند `trace` را نیز شامل شوند؛ یک زمینهٔ ردیابی تشخیصی فقط‌خواندنی و سازگار با W3C که Pluginها می‌توانند برای هم‌بستگی OTEL به گزارش‌های ساخت‌یافته منتقل کنند.

رشته‌هایی که به `event.messages` افزوده می‌شوند، تنها برای
`command:new` و `command:reset` به چت بازگردانده می‌شوند (به‌عنوان پاسخ به گفت‌وگوی
مبدأ مسیریابی می‌شوند) و برای `session:compact:before` / `session:compact:after`
(به‌عنوان اعلان‌های وضعیت Compaction ارسال می‌شوند). همهٔ رویدادهای دیگر، از جمله
`command:stop`، `message:*`، `agent:bootstrap`، `session:patch` و
`gateway:*`، پیام‌های افزوده‌شده را نادیده می‌گیرند.

### نکات برجستهٔ زمینهٔ رویداد

**رویدادهای فرمان** (`command:new`، `command:reset`): `context.sessionEntry`، `context.previousSessionEntry`، `context.commandSource`، `context.senderId`، `context.workspaceDir`، `context.cfg`.

**رویدادهای فرمان** (`command:stop`): `context.sessionEntry`، `context.sessionId`، `context.commandSource`، `context.senderId`.

**رویدادهای پیام** (`message:received`): `context.from`، `context.content`، `context.channelId`، `context.media` (اطلاعات مرتب‌شدهٔ پیوست‌های مرحله‌بندی‌شده)، `context.originalMedia` به‌همراه `context.mediaStagingPending` هنگامی که رسانهٔ راه‌دور هنوز به‌صورت محلی مرحله‌بندی نشده است، و `context.metadata` (داده‌های مختص ارائه‌دهنده شامل `senderId`، `senderName`، `guildId`). `context.content` برای پیام‌های فرمان‌مانند، بدنهٔ فرمان غیرخالی را ترجیح می‌دهد، سپس به بدنهٔ خام ورودی و بدنهٔ عمومی بازمی‌گردد؛ این مورد شامل غنی‌سازی‌های مختص عامل مانند تاریخچهٔ رشته یا خلاصه‌های پیوند نمی‌شود. نام‌های مستعار قدیمی رسانه در `metadata` منسوخ شده‌اند.

**رویدادهای پیام** (`message:sent`): `context.to`، `context.content`، `context.success`، `context.channelId`، به‌همراه `context.error` هنگام ناموفق‌بودن ارسال.

**رویدادهای پیام** (`message:transcribed`): `context.transcript`، `context.from`، `context.channelId` و `context.media`. `context.mediaPath` و `context.mediaType` همچنان نام‌های مستعار منسوخ‌شده برای نخستین مورد هستند.

**رویدادهای پیام** (`message:preprocessed`): `context.bodyForAgent` (بدنهٔ نهایی غنی‌شده)، `context.from`، `context.channelId`.

**رویدادهای راه‌اندازی** (`agent:bootstrap`): `context.bootstrapFiles` (آرایهٔ تغییرپذیر)، `context.agentId`.

**رویدادهای وصلهٔ نشست** (`session:patch`): `context.sessionEntry`، `context.patch` (فقط فیلدهای تغییرکرده)، `context.cfg`. فقط کلاینت‌های دارای امتیاز می‌توانند رویدادهای وصله را فعال کنند؛ زمینه یک کپی است، بنابراین کنترل‌گرها نمی‌توانند ورودی زندهٔ نشست را تغییر دهند.

**رویدادهای Compaction**: `session:compact:before` شامل `messageCount`، `tokenCount` است. `session:compact:after` موارد `compactedCount`، `summaryLength`، `tokensBefore`، `tokensAfter` را اضافه می‌کند.

`command:stop` صدور `/stop` توسط کاربر را مشاهده می‌کند؛ این مربوط به چرخهٔ حیات
لغو/فرمان است، نه یک دروازهٔ نهایی‌سازی عامل. Pluginهایی که باید یک
پاسخ نهایی طبیعی را بررسی کنند و از عامل یک دور دیگر بخواهند، باید در عوض از Hook نوع‌دار
Plugin به نام `before_agent_finalize` استفاده کنند. به [Hookهای Plugin](/fa/plugins/hooks) مراجعه کنید.

**رویدادهای چرخهٔ حیات Gateway**: `gateway:shutdown` شامل `reason` و `restartExpectedMs` است و هنگام آغاز خاموش‌شدن Gateway فعال می‌شود. `gateway:pre-restart` همان زمینه را شامل می‌شود، اما تنها زمانی فعال می‌شود که خاموش‌شدن بخشی از یک راه‌اندازی مجدد مورد انتظار باشد و مقدار متناهی `restartExpectedMs` ارائه شود. هنگام خاموش‌شدن، انتظار برای هر Hook چرخهٔ حیات به‌صورت بهترین تلاش و محدود انجام می‌شود تا اگر کنترل‌گری متوقف شد، خاموش‌شدن ادامه یابد. بودجهٔ انتظار پیش‌فرض برای `gateway:shutdown` برابر 5 ثانیه و برای `gateway:pre-restart` برابر 10 ثانیه است.

برای اعلان‌های کوتاه راه‌اندازی مجدد، درحالی‌که کانال‌ها همچنان در دسترس‌اند، از `gateway:pre-restart` استفاده کنید:

```typescript
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const execFileAsync = promisify(execFile);

export default async function handler(event) {
  if (event.type !== "gateway" || event.action !== "pre-restart") {
    return;
  }

  const restartInSeconds = Math.ceil(event.context.restartExpectedMs / 1000);
  await execFileAsync("openclaw", [
    "system",
    "event",
    "--mode",
    "now",
    "--text",
    `Gateway تقریباً تا ${restartInSeconds} ثانیهٔ دیگر راه‌اندازی مجدد می‌شود (${event.context.reason}). اکنون نقطهٔ بازرسی ایجاد کنید.`,
  ]);
}
```

بین رویداد `gateway:shutdown` (یا `gateway:pre-restart`) و ادامهٔ توالی خاموش‌شدن، Gateway همچنین برای هر نشستی که هنگام توقف فرایند همچنان فعال بوده است، یک Hook نوع‌دار Plugin به نام `session_end` فعال می‌کند. مقدار `reason` این رویداد برای توقف ساده با SIGTERM/SIGINT برابر `shutdown` و هنگامی که بسته‌شدن به‌عنوان بخشی از یک راه‌اندازی مجدد مورد انتظار زمان‌بندی شده باشد برابر `restart` است. این تخلیه محدود است تا یک کنترل‌گر کند `session_end` نتواند خروج فرایند را مسدود کند، و نشست‌هایی که پیش‌تر از طریق جایگزینی / بازنشانی / حذف / Compaction نهایی شده‌اند، برای جلوگیری از فعال‌شدن دوباره نادیده گرفته می‌شوند.

## شناسایی Hook

Hookها از چهار منبع شناسایی می‌شوند:

1. **هوک‌های همراه**: همراه OpenClaw عرضه می‌شوند
2. **هوک‌های Plugin**: درون Pluginهای نصب‌شده قرار دارند؛ می‌توانند هوک‌های همراهِ هم‌نام را بازنویسی کنند
3. **هوک‌های مدیریت‌شده**: `~/.openclaw/hooks/` (نصب‌شده توسط کاربر و مشترک میان فضای‌کارها)؛ می‌توانند هوک‌های همراه و هوک‌های Plugin را بازنویسی کنند. دایرکتوری‌های اضافی از `hooks.internal.load.extraDirs` نیز همین اولویت را دارند.
4. **هوک‌های فضای‌کار**: `<workspace>/hooks/` (مختص هر عامل، به‌طور پیش‌فرض غیرفعال تا زمانی که صریحاً فعال شود)

هوک‌های فضای‌کار می‌توانند نام‌های هوک جدیدی اضافه کنند، اما نمی‌توانند هوک‌های هم‌نامِ همراه، مدیریت‌شده یا ارائه‌شده توسط Plugin را بازنویسی کنند.

Gateway هنگام راه‌اندازی، تا زمانی که هوک‌های داخلی پیکربندی نشده باشند، از کشف هوک‌های داخلی صرف‌نظر می‌کند. یک هوک همراه یا مدیریت‌شده را با `openclaw hooks enable <name>` فعال کنید، یک بسته هوک نصب کنید، یا برای اعلام موافقت، `hooks.internal.enabled=true` را تنظیم کنید. وقتی یک هوک نام‌دار را فعال می‌کنید، Gateway فقط کنترل‌گر همان هوک را بارگذاری می‌کند؛ `hooks.internal.enabled=true`، دایرکتوری‌های هوک اضافی و کنترل‌گرهای قدیمی، کشف گسترده را فعال می‌کنند.

### بسته‌های هوک

بسته‌های هوک، بسته‌های npm هستند که هوک‌ها را از طریق `openclaw.hooks` در `package.json` صادر می‌کنند. برای نصب:

```bash
openclaw plugins install <path-or-spec>
```

مشخصات Npm فقط به رجیستری محدود می‌شوند (نام بسته + نسخه دقیق اختیاری یا dist-tag). مشخصات Git/URL/file و بازه‌های semver رد می‌شوند. فرمان‌های قدیمی‌تر `openclaw hooks install` و `openclaw hooks update` نام‌های مستعار منسوخ‌شده برای `openclaw plugins install` / `openclaw plugins update` هستند.

## هوک‌های همراه

| هوک                  | رویدادها                                            | عملکرد                                                   |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------- |
| session-memory        | `command:new`، `command:reset`                    | زمینه نشست را در `<workspace>/memory/` ذخیره می‌کند                 |
| bootstrap-extra-files | `agent:bootstrap`                                 | فایل‌های راه‌اندازی اولیه اضافی را از الگوهای glob تزریق می‌کند          |
| command-logger        | `command`                                         | همه فرمان‌ها را در `~/.openclaw/logs/commands.log` ثبت می‌کند           |
| compaction-notifier   | `session:compact:before`، `session:compact:after` | هنگام شروع/پایان Compaction نشست، اعلان‌های قابل‌مشاهده در گفت‌وگو ارسال می‌کند |
| boot-md               | `gateway:startup`                                 | هنگام شروع Gateway، `BOOT.md` را اجرا می‌کند                         |

برای فعال‌کردن هر هوک همراه:

```bash
openclaw hooks enable <hook-name>
```

<a id="session-memory"></a>

### جزئیات session-memory

آخرین پیام‌های کاربر/دستیار را استخراج می‌کند (پیش‌فرض 15، قابل‌پیکربندی با `hooks.internal.entries.session-memory.messages`) و با استفاده از تاریخ محلی میزبان در `<workspace>/memory/YYYY-MM-DD-HHMM.md` ذخیره می‌کند. ثبت حافظه در پس‌زمینه اجرا می‌شود تا تأییدهای `/new` و `/reset` به‌دلیل خواندن رونوشت یا تولید اختیاری نامک به تأخیر نیفتند. برای تولید نامک‌های توصیفی نام فایل، `hooks.internal.entries.session-memory.llmSlug: true` را تنظیم کنید و در صورت تمایل، `hooks.internal.entries.session-memory.model` را روی یک نام مستعار پیکربندی‌شده مانند `sonnet`، یک شناسه مدل ساده در ارائه‌دهنده پیش‌فرض عامل، یا یک ارجاع `provider/model` تنظیم کنید. وقتی `model` حذف شده باشد، تولید نامک از مدل پیش‌فرض عامل استفاده می‌کند و در صورت دردسترس‌نبودن، به نامک‌های برچسب زمانی برمی‌گردد. مستلزم پیکربندی `workspace.dir` است.

<a id="bootstrap-extra-files"></a>

### پیکربندی bootstrap-extra-files

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "bootstrap-extra-files": {
          "enabled": true,
          "paths": ["packages/*/AGENTS.md", "packages/*/TOOLS.md"]
        }
      }
    }
  }
}
```

`patterns` و `files` به‌عنوان نام‌های مستعار `paths` پذیرفته می‌شوند. مسیرها نسبت به فضای‌کار تفکیک می‌شوند و باید درون آن باقی بمانند. فقط نام‌های پایه راه‌اندازی اولیه شناخته‌شده بارگذاری می‌شوند (`AGENTS.md`، `SOUL.md`، `TOOLS.md`، `IDENTITY.md`، `USER.md`، `HEARTBEAT.md`، `BOOTSTRAP.md`، `MEMORY.md`).

<a id="command-logger"></a>

### جزئیات command-logger

هر فرمان اسلش را به‌صورت یک خط JSON (برچسب زمانی، کنش، کلید نشست، شناسه فرستنده، منبع) در `~/.openclaw/logs/commands.log` ثبت می‌کند.

<a id="compaction-notifier"></a>

### جزئیات compaction-notifier

هنگامی که OpenClaw فشرده‌سازی رونوشت نشست را شروع و تمام می‌کند، پیام‌های وضعیت کوتاهی به گفت‌وگوی جاری ارسال می‌کند. این کار نوبت‌های طولانی را در محیط‌های گفت‌وگو کمتر گیج‌کننده می‌کند، زیرا کاربر می‌تواند ببیند که دستیار در حال خلاصه‌سازی زمینه است و پس از Compaction ادامه خواهد داد.

<a id="boot-md"></a>

### جزئیات boot-md

در زمان راه‌اندازی Gateway، برای هر محدوده عامل پیکربندی‌شده، اگر فایل در فضای‌کار تفکیک‌شده آن عامل وجود داشته باشد، `BOOT.md` را اجرا می‌کند.

## هوک‌های Plugin

Pluginها می‌توانند برای یکپارچه‌سازی عمیق‌تر، هوک‌های نوع‌دار را از طریق SDK ‏Plugin ثبت کنند:
رهگیری فراخوانی ابزارها، تغییر اعلان‌ها، کنترل جریان پیام و موارد دیگر.
زمانی از هوک‌های Plugin استفاده کنید که به `before_tool_call`، `before_agent_reply`،
`before_install` یا سایر هوک‌های چرخه‌عمر درون‌فرایندی نیاز دارید.

هوک‌های داخلی مدیریت‌شده توسط Plugin متفاوت‌اند: آن‌ها در سامانه کلی
رویدادهای فرمان/چرخه‌عمر این صفحه مشارکت دارند و در `openclaw hooks list` به‌صورت
`plugin:<id>` نمایش داده می‌شوند. از آن‌ها برای اثرات جانبی و سازگاری با بسته‌های هوک استفاده کنید، نه
برای میان‌افزار مرتب‌شده یا دروازه‌های خط‌مشی.

برای مرجع کامل هوک‌های Plugin، به [هوک‌های Plugin](/fa/plugins/hooks) مراجعه کنید.

## پیکربندی

```json
{
  "hooks": {
    "internal": {
      "enabled": true,
      "entries": {
        "session-memory": { "enabled": true },
        "command-logger": { "enabled": false }
      }
    }
  }
}
```

مقادیر محیطی مختص هر هوک، بررسی‌های صلاحیت `requires.env` هوک را برآورده می‌کنند (در کنار محیط فرایند) و کنترل‌گرها می‌توانند آن‌ها را از ورودی پیکربندی هوک خود بخوانند:

```json
{
  "hooks": {
    "internal": {
      "entries": {
        "my-hook": {
          "enabled": true,
          "env": { "MY_CUSTOM_VAR": "value" }
        }
      }
    }
  }
}
```

دایرکتوری‌های هوک اضافی:

```json
{
  "hooks": {
    "internal": {
      "load": {
        "extraDirs": ["/path/to/more/hooks"]
      }
    }
  }
}
```

<Note>
قالب قدیمی پیکربندی آرایه `hooks.internal.handlers` همچنان برای سازگاری با نسخه‌های پیشین پشتیبانی می‌شود، اما هوک‌های جدید باید از سامانه مبتنی بر کشف استفاده کنند.
</Note>

## مرجع CLI

```bash
# فهرست‌کردن همه هوک‌ها (--eligible، --verbose یا --json را اضافه کنید)
openclaw hooks list

# نمایش اطلاعات تفصیلی درباره یک هوک
openclaw hooks info <hook-name>

# نمایش خلاصه صلاحیت
openclaw hooks check

# فعال/غیرفعال‌کردن
openclaw hooks enable <hook-name>
openclaw hooks disable <hook-name>
```

## بهترین روش‌ها

- **کنترل‌گرها را سریع نگه دارید.** هوک‌ها هنگام پردازش فرمان اجرا می‌شوند. کارهای سنگین را با `void processInBackground(event)` به‌صورت اجرا و عدم انتظار انجام دهید.
- **خطاها را به‌درستی مدیریت کنید.** عملیات پرریسک را در try/catch بپیچید؛ خطا پرتاب نکنید تا سایر کنترل‌گرها بتوانند اجرا شوند.
- **رویدادها را زود فیلتر کنید.** اگر نوع/کنش رویداد مرتبط نیست، فوراً برگردید.
- **از کلیدهای رویداد مشخص استفاده کنید.** برای کاهش سربار، `"events": ["command:new"]` را به `"events": ["command"]` ترجیح دهید.

## عیب‌یابی

### هوک کشف نمی‌شود

```bash
# بررسی ساختار دایرکتوری
ls -la ~/.openclaw/hooks/my-hook/
# باید نمایش دهد: HOOK.md، handler.ts

# فهرست‌کردن همه هوک‌های کشف‌شده
openclaw hooks list
```

### هوک واجد شرایط نیست

```bash
openclaw hooks info my-hook
```

وجود فایل‌های اجرایی مفقود (PATH)، متغیرهای محیطی، مقادیر پیکربندی یا سازگاری سیستم‌عامل را بررسی کنید.

### هوک اجرا نمی‌شود

1. بررسی کنید هوک فعال است: `openclaw hooks list`
2. فرایند Gateway را مجدداً راه‌اندازی کنید تا هوک‌ها دوباره بارگذاری شوند.
3. لاگ‌های Gateway را بررسی کنید: `openclaw logs --follow | grep -i hook`

## مرتبط

- [مرجع CLI: هوک‌ها](/fa/cli/hooks)
- [Webhookها](/fa/automation/cron-jobs#webhooks)
- [هوک‌های Plugin](/fa/plugins/hooks) — هوک‌های چرخه‌عمر درون‌فرایندی Plugin
- [پیکربندی](/fa/gateway/configuration-reference#hooks)
