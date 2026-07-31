---
read_when:
    - راه‌اندازی اولیه یک نمونهٔ جدید دستیار
    - بازبینی پیامدهای ایمنی و مجوزها
summary: راهنمای سرتاسری اجرای OpenClaw به‌عنوان دستیار شخصی، همراه با هشدارهای ایمنی
title: راه‌اندازی دستیار شخصی
x-i18n:
    generated_at: "2026-07-27T17:09:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ed3e267971fc1ee5c9154194e5b1f98db8c7a7edca8182871a2057a778614217
    source_path: start/openclaw.md
    workflow: 16
---

OpenClaw یک Gateway خودمیزبان است که Discord، Google Chat، iMessage، Matrix، Microsoft Teams، Signal، Slack، Telegram، WhatsApp، Zalo و سرویس‌های دیگر را به عامل‌های هوش مصنوعی متصل می‌کند. این راهنما راه‌اندازی «دستیار شخصی» را پوشش می‌دهد: یک شماره اختصاصی WhatsApp که مانند دستیار هوش مصنوعی همیشه‌فعال شما رفتار می‌کند.

## ابتدا ایمنی

دادن یک کانال به عامل، آن را در موقعیتی قرار می‌دهد که بتواند فرمان‌هایی را روی دستگاه شما اجرا کند (بسته به خط‌مشی ابزار شما)، فایل‌های فضای کاری شما را بخواند یا بنویسد و از طریق هر کانال متصل، پیام ارسال کند. در ابتدا محافظه‌کارانه عمل کنید:

- همیشه `channels.whatsapp.allowFrom` را تنظیم کنید (هرگز آن را روی Mac شخصی خود برای دسترسی عمومی اجرا نکنید).
- برای دستیار از یک شماره اختصاصی WhatsApp استفاده کنید.
- Heartbeatها به‌طور پیش‌فرض هر 30 دقیقه اجرا می‌شوند. تا زمانی که به راه‌اندازی اعتماد نکرده‌اید، با تنظیم `agents.defaults.heartbeat.every: "0m"` آن‌ها را غیرفعال کنید.

## پیش‌نیازها

- OpenClaw نصب و راه‌اندازی اولیه شده باشد — اگر هنوز این کار را انجام نداده‌اید، به [شروع کار](/fa/start/getting-started) مراجعه کنید
- یک شماره تلفن دوم (SIM/eSIM/اعتباری) برای دستیار

## راه‌اندازی با دو تلفن (توصیه‌شده)

چیدمان مطلوب به این صورت است:

```mermaid
flowchart TB
    A["<b>تلفن شما (شخصی)<br></b><br>WhatsApp شما<br>+1-555-YOU"] -- پیام --> B["<b>تلفن دوم (دستیار)<br></b><br>WhatsApp دستیار<br>+1-555-ASSIST"]
    B -- پیوند با کد QR --> C["<b>Mac شما (openclaw)<br></b><br>عامل هوش مصنوعی"]
```

اگر WhatsApp شخصی خود را به OpenClaw پیوند دهید، هر پیامی که برای شما ارسال می‌شود به «ورودی عامل» تبدیل خواهد شد. این معمولاً چیزی نیست که می‌خواهید.

## شروع سریع 5 دقیقه‌ای

1. WhatsApp Web را جفت کنید (کد QR نمایش داده می‌شود؛ آن را با تلفن دستیار اسکن کنید):

```bash
openclaw channels login
```

2. Gateway را راه‌اندازی کنید (در حال اجرا نگه دارید):

```bash
openclaw gateway --port 18789
```

3. یک پیکربندی حداقلی در `~/.openclaw/openclaw.json` قرار دهید:

```json5
{
  gateway: { mode: "local" },
  channels: { whatsapp: { allowFrom: ["+15555550123"] } },
}
```

اکنون از تلفنی که در فهرست مجاز قرار دارد به شماره دستیار پیام دهید.

پس از پایان راه‌اندازی اولیه، OpenClaw داشبورد را به‌طور خودکار باز می‌کند و یک پیوند تمیز (بدون توکن) نمایش می‌دهد. اگر داشبورد احراز هویت درخواست کرد، راز مشترک پیکربندی‌شده را در تنظیمات Control UI جای‌گذاری کنید. راه‌اندازی اولیه به‌طور پیش‌فرض از توکن استفاده می‌کند (`gateway.auth.token`)، اما اگر `gateway.auth.mode` را به `password` تغییر داده‌اید، احراز هویت با گذرواژه نیز کار می‌کند. برای بازکردن دوباره در آینده: `openclaw dashboard`.

## اختصاص فضای کاری به عامل (AGENTS)

OpenClaw دستورالعمل‌های عملیاتی و «حافظه» را از پوشه فضای کاری خود می‌خواند.

OpenClaw به‌طور پیش‌فرض از `~/.openclaw/workspace` به‌عنوان فضای کاری عامل استفاده می‌کند و آن را (همراه با فایل‌های آغازین `AGENTS.md`، `SOUL.md`، `TOOLS.md`، `IDENTITY.md` و `USER.md`) هنگام راه‌اندازی اولیه یا نخستین اجرای عامل به‌طور خودکار ایجاد می‌کند. `BOOTSTRAP.md` فقط برای یک فضای کاری کاملاً جدید ایجاد می‌شود و پس از حذف نباید دوباره ظاهر شود. `MEMORY.md` اختیاری است و هرگز به‌طور خودکار ایجاد نمی‌شود؛ در صورت وجود، برای نشست‌های عادی بارگذاری می‌شود. نشست‌های زیرعامل فقط `AGENTS.md` و `TOOLS.md` را تزریق می‌کنند.

<Tip>
این پوشه را مانند حافظه OpenClaw در نظر بگیرید و آن را به یک مخزن git تبدیل کنید (ترجیحاً خصوصی) تا از `AGENTS.md` و فایل‌های حافظه شما پشتیبان تهیه شود. اگر git نصب باشد، فضای کاری کاملاً جدید به‌طور خودکار با `git init` مقداردهی اولیه می‌شود.
</Tip>

برای ایجاد پوشه‌های فضای کاری و پیکربندی بدون اجرای کامل راهنمای تعاملی راه‌اندازی اولیه:

```bash
openclaw setup --baseline
```

(`openclaw setup` به‌تنهایی نام مستعار `openclaw onboard` است و راهنمای تعاملی کامل را اجرا می‌کند.)

راهنمای کامل چیدمان فضای کاری و پشتیبان‌گیری: [فضای کاری عامل](/fa/concepts/agent-workspace)
گردش کار حافظه: [حافظه](/fa/concepts/memory)

اختیاری: با `agents.defaults.workspace` فضای کاری دیگری انتخاب کنید (از `~` پشتیبانی می‌کند).

```json5
{
  agents: {
    defaults: {
      workspace: "~/.openclaw/workspace",
    },
  },
}
```

اگر فایل‌های فضای کاری خود را از یک مخزن ارائه می‌کنید، می‌توانید ایجاد فایل‌های راه‌انداز را به‌طور کامل غیرفعال کنید:

```json5
{
  agents: {
    defaults: {
      skipBootstrap: true,
    },
  },
}
```

## پیکربندی‌ای که آن را به «یک دستیار» تبدیل می‌کند

تنظیمات پیش‌فرض OpenClaw برای یک دستیار مناسب است، اما معمولاً بهتر است موارد زیر را تنظیم کنید:

- شخصیت/دستورالعمل‌ها در [`SOUL.md`](/fa/concepts/soul)
- پیش‌فرض‌های تفکر (در صورت تمایل)
- Heartbeatها (پس از آنکه به آن اعتماد کردید)

نمونه:

```json5
{
  logging: { level: "info" },
  agents: {
    defaults: {
      model: { primary: "anthropic/claude-opus-5" },
      workspace: "~/.openclaw/workspace",
      thinkingDefault: "high",
      timeoutSeconds: 1800,
      // ابتدا روی 0 تنظیم کنید؛ بعداً فعال کنید.
      heartbeat: { every: "0m" },
    },
    list: [
      {
        id: "main",
        default: true,
        groupChat: {
          mentionPatterns: ["@openclaw", "openclaw"],
        },
      },
    ],
  },
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: {
        "*": { requireMention: true },
      },
    },
  },
  session: {
    scope: "per-sender",
    resetTriggers: ["/new", "/reset"],
    reset: {
      mode: "daily",
      atHour: 4,
      idleMinutes: 10080,
    },
  },
}
```

## نشست‌ها و حافظه

- ردیف‌های نشست، ردیف‌های رونوشت و فراداده (مصرف توکن، آخرین مسیر و غیره): `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
- آثار رونوشت قدیمی/بایگانی‌شده: `~/.openclaw/agents/<agentId>/sessions/`
- منبع مهاجرت ردیف‌های قدیمی: `~/.openclaw/agents/<agentId>/sessions/sessions.json`
- `/new` یا `/reset` یک نشست تازه برای آن گفت‌وگو آغاز می‌کند (از طریق `session.resetTriggers` قابل پیکربندی است). اگر به‌تنهایی ارسال شود، OpenClaw بدون فراخوانی مدل، بازنشانی را تأیید می‌کند.
- `/compact [instructions]` زمینه نشست را Compaction می‌کند و بودجه باقی‌مانده زمینه را گزارش می‌دهد.

## Heartbeatها (حالت فعالانه)

OpenClaw به‌طور پیش‌فرض هر 30 دقیقه یک Heartbeat با این اعلان اجرا می‌کند:
`Follow the heartbeat monitor scratch context when provided. Recurring tasks are cron jobs; create or change their schedules with cron tools or the openclaw cron CLI, not heartbeat scratch. Do not infer or repeat old tasks from prior chats. If nothing needs attention, reply HEARTBEAT_OK.`
برای غیرفعال‌کردن، `agents.defaults.heartbeat.every: "0m"` را تنظیم کنید. چک‌لیست‌های Heartbeat در فضای موقت Cron ناظر قرار دارند (به [Heartbeat](/fa/gateway/heartbeat) مراجعه کنید)؛ `openclaw doctor --fix` فایل قدیمی `HEARTBEAT.md` فضای کاری را به آن منتقل می‌کند.

- اگر فضای موقت ناظر وجود داشته باشد اما عملاً خالی باشد (فقط شامل خطوط خالی، توضیحات Markdown/HTML، عنوان‌های Markdown مانند `# Heading`، نشانگرهای حصار یا نمونه‌های خالی چک‌لیست باشد)، OpenClaw برای صرفه‌جویی در فراخوانی‌های API اجرای Heartbeat را نادیده می‌گیرد.
- اگر فضای موقتی وجود نداشته باشد، Heartbeat همچنان اجرا می‌شود و مدل تصمیم می‌گیرد چه کاری انجام دهد.
- اگر عامل با `HEARTBEAT_OK` پاسخ دهد (در صورت تمایل همراه با حاشیه کوتاه؛ به `agents.defaults.heartbeat.ackMaxChars` مراجعه کنید)، OpenClaw ارسال خروجی آن Heartbeat را متوقف می‌کند.
- ارسال Heartbeat به مقصدهای پیام مستقیم‌مانند `user:<id>` به‌طور پیش‌فرض مجاز است. برای جلوگیری از ارسال به مقصدهای مستقیم، درحالی‌که اجرای Heartbeat فعال باقی می‌ماند، `agents.defaults.heartbeat.directPolicy: "block"` را تنظیم کنید.
- Heartbeatها نوبت‌های کامل عامل را اجرا می‌کنند — فاصله‌های کوتاه‌تر توکن بیشتری مصرف می‌کنند.

```json5
{
  agents: {
    defaults: {
      heartbeat: { every: "30m" },
    },
  },
}
```

## رسانه ورودی و خروجی

پیوست‌های ورودی (تصویر/صدا/سند) را می‌توان از طریق الگوها در اختیار فرمان قرار داد:

- `{{AttachmentPath}}` (مسیر فایل موقت محلی)
- `{{AttachmentUrl}}` (نشانی اینترنتی اصلی یا ارجاع ارائه‌دهنده)
- `{{AttachmentContentType}}` (نوع محتوای MIME)
- `{{AttachmentDir}}` (پوشه حاوی مسیر محلی)
- `{{AttachmentIndex}}` (شاخص مبتنی بر صفرِ داده منبع)
- `{{Transcript}}` (اگر رونویسی صدا فعال باشد)

نام‌های قدیمی‌تر `{{MediaPath}}`، `{{MediaUrl}}`، `{{MediaType}}` و `{{MediaDir}}`
همچنان به‌عنوان نام‌های مستعار سازگاری منسوخ‌شده در دسترس هستند.

پیوست‌های خروجی عامل از فیلدهای رسانه‌ای ساخت‌یافته در ابزار پیام یا محموله پاسخ استفاده می‌کنند؛ مانند `media`، `mediaUrl`، `mediaUrls`، `path` یا `filePath`. نمونه آرگومان‌های ابزار پیام:

```json
{
  "message": "این هم تصویر صفحه.",
  "mediaUrl": "https://example.com/screenshot.png"
}
```

OpenClaw رسانه ساخت‌یافته را همراه متن ارسال می‌کند. پاسخ‌های نهایی قدیمی دستیار ممکن است همچنان برای سازگاری عادی‌سازی شوند، اما خروجی ابزار، خروجی مرورگر، بلوک‌های جریانی و کنش‌های پیام، متن را به‌عنوان فرمان پیوست تفسیر نمی‌کنند.

رفتار مسیر محلی از همان مدل اعتماد خواندن فایل عامل پیروی می‌کند:

- اگر `tools.fs.workspaceOnly` برابر با `true` باشد، مسیرهای رسانه محلی خروجی همچنان به ریشه موقت OpenClaw، حافظه نهان رسانه، مسیرهای فضای کاری عامل و فایل‌های تولیدشده در محیط ایزوله محدود می‌مانند.
- اگر `tools.fs.workspaceOnly` برابر با `false` باشد، رسانه محلی خروجی می‌تواند از فایل‌های محلی میزبان که عامل از قبل مجاز به خواندن آن‌هاست استفاده کند.
- مسیرهای محلی می‌توانند مطلق، نسبت به فضای کاری یا نسبت به پوشه خانگی با `~/` باشند.
- ارسال‌های محلی میزبان همچنان فقط رسانه‌ها و انواع امن سند را مجاز می‌دانند (تصاویر، صدا، ویدئو، PDF، اسناد Office و اسناد متنی اعتبارسنجی‌شده مانند Markdown/MD، TXT، JSON، YAML و YML). این گسترشی از مرز اعتماد موجود برای خواندن میزبان است، نه یک اسکنر اسرار: اگر عامل بتواند یک `secret.txt` یا `config.json` محلی میزبان را بخواند، هنگامی که پسوند و اعتبارسنجی محتوا مطابقت داشته باشند می‌تواند آن فایل را پیوست کند.

فایل‌های حساس را خارج از سیستم فایل قابل‌خواندن برای عامل نگه دارید، یا برای ارسال‌های مسیر محلی سخت‌گیرانه‌تر، `tools.fs.workspaceOnly: true` را حفظ کنید.

## چک‌لیست عملیات

```bash
openclaw status          # وضعیت محلی (اعتبارنامه‌ها، نشست‌ها، رویدادهای در صف)
openclaw status --all    # تشخیص کامل (فقط‌خواندنی، قابل جای‌گذاری)
openclaw status --deep   # بررسی کانال‌ها (WhatsApp Web + Telegram + Discord + Slack + Signal)
openclaw health --json   # تصویر لحظه‌ای سلامت Gateway از طریق اتصال WS
```

گزارش‌ها در `/tmp/openclaw/` قرار دارند: `openclaw-YYYY-MM-DD.log` برای نمایه پیش‌فرض
و `openclaw-<profile>-YYYY-MM-DD.log` برای نمایه‌های نام‌گذاری‌شده.

## گام‌های بعدی

- WebChat: [WebChat](/fa/web/webchat)
- عملیات Gateway: [راهنمای عملیاتی Gateway](/fa/gateway)
- Cron و بیدارسازی‌ها: [کارهای Cron](/fa/automation/cron-jobs)
- همراه نوار منوی macOS: [برنامه macOS ‏OpenClaw](/fa/platforms/macos)
- برنامه Node برای iOS: [برنامه iOS](/fa/platforms/ios)
- برنامه Node برای Android: [برنامه Android](/fa/platforms/android)
- مرکز Windows: [Windows](/fa/platforms/windows)
- وضعیت Linux: [برنامه Linux](/fa/platforms/linux)
- امنیت: [امنیت](/fa/gateway/security)

## مرتبط

- [شروع کار](/fa/start/getting-started)
- [راه‌اندازی](/fa/start/setup)
- [نمای کلی کانال‌ها](/fa/channels)
