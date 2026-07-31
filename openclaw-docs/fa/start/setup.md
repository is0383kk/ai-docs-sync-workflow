---
read_when:
    - راه‌اندازی یک دستگاه جدید
    - جدیدترین و بهترین‌ها را می‌خواهید، بدون اینکه تنظیمات شخصی‌تان به‌هم بخورد
summary: گردش‌کارهای پیشرفته راه‌اندازی و توسعه برای OpenClaw
title: راه‌اندازی
x-i18n:
    generated_at: "2026-07-27T14:42:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c40d6d2bf2814465f3cc49c65d4c1498671420af728ce8012d13af3fba67025a
    source_path: start/setup.md
    workflow: 16
---

<Note>
اگر برای نخستین بار راه‌اندازی می‌کنید، از [شروع به کار](/fa/start/getting-started) آغاز کنید.
برای جزئیات راه‌اندازی اولیه، به [راه‌اندازی اولیه (CLI)](/fa/start/wizard) مراجعه کنید.
</Note>

## خلاصه

بر اساس دفعاتی که به‌روزرسانی می‌خواهید و اینکه آیا می‌خواهید Gateway را خودتان اجرا کنید، یک گردش‌کار راه‌اندازی انتخاب کنید:

- **شخصی‌سازی خارج از مخزن انجام می‌شود:** پیکربندی و فضای کاری خود را در `~/.openclaw/openclaw.json` و `~/.openclaw/workspace/` نگه دارید تا به‌روزرسانی‌های مخزن به آن‌ها دست نزنند.
- **گردش‌کار پایدار (پیشنهادی برای اکثر کاربران):** برنامه macOS را نصب کنید و اجازه دهید Gateway همراه آن را اجرا کند.
- **گردش‌کار پیشرو (توسعه):** Gateway را خودتان از طریق `pnpm gateway:watch` اجرا کنید، سپس اجازه دهید برنامه macOS در حالت Local به آن متصل شود.

## پیش‌نیازها (از کد منبع)

- Node 24.15+ پیشنهاد می‌شود (Node 22 LTS که در حال حاضر `22.22.3+` است، همچنان پشتیبانی می‌شود)
- `pnpm` برای دریافت‌های کد منبع الزامی است. OpenClaw در حالت توسعه، Pluginهای همراه را از بسته‌های فضای کاری pnpm در
  `extensions/*` بارگیری می‌کند؛ بنابراین `npm install` در ریشه،
  کل درخت کد منبع را آماده نمی‌کند.
- Docker (اختیاری؛ فقط برای راه‌اندازی کانتینری/e2e — به [Docker](/fa/install/docker) مراجعه کنید)

## راهبرد شخصی‌سازی (تا به‌روزرسانی‌ها آسیبی نزنند)

اگر هم «۱۰۰٪ متناسب با من» و هم به‌روزرسانی آسان می‌خواهید، سفارشی‌سازی‌های خود را در این موارد نگه دارید:

- **پیکربندی:** `~/.openclaw/openclaw.json` (JSON/تقریباً JSON5)
- **فضای کاری:** `~/.openclaw/workspace` (Skills، پرامپت‌ها، حافظه‌ها؛ آن را به یک مخزن خصوصی git تبدیل کنید)

پوشه‌های پیکربندی/فضای کاری را یک بار، بدون اجرای کامل راهنمای تعاملی راه‌اندازی اولیه، آماده کنید:

```bash
openclaw setup --baseline
```

هنوز نصب سراسری ندارید؟ در عوض، آن را از همین مخزن اجرا کنید:

```bash
pnpm openclaw setup --baseline
```

(`openclaw setup` به‌تنهایی و بدون `--baseline`، نام مستعار `openclaw onboard` است و راهنمای تعاملی کامل را اجرا می‌کند.)

## اجرای Gateway از این مخزن

پس از `pnpm build`، می‌توانید CLI بسته‌بندی‌شده را مستقیماً اجرا کنید:

```bash
node openclaw.mjs gateway --port 18789 --verbose
```

## گردش‌کار پایدار (ابتدا برنامه macOS)

1. **OpenClaw.app** را نصب و اجرا کنید (نوار منو).
2. چک‌لیست راه‌اندازی اولیه/مجوزها را تکمیل کنید (اعلان‌های TCC).
3. مطمئن شوید Gateway روی **Local** تنظیم شده و در حال اجراست (برنامه آن را مدیریت می‌کند).
4. سطوح ارتباطی را متصل کنید (برای نمونه: WhatsApp):

```bash
openclaw channels login
```

5. بررسی سلامت اولیه:

```bash
openclaw health
```

اگر راه‌اندازی اولیه در بیلد شما موجود نیست:

- `openclaw setup` و سپس `openclaw channels login` را اجرا کنید و بعد Gateway را به‌صورت دستی راه‌اندازی کنید (`openclaw gateway`).

## گردش‌کار پیشرو (Gateway در ترمینال)

هدف: کار روی Gateway مبتنی بر TypeScript، برخورداری از بارگذاری مجدد فوری و متصل نگه‌داشتن رابط کاربری برنامه macOS.

### 0) (اختیاری) برنامه macOS را نیز از کد منبع اجرا کنید

اگر می‌خواهید برنامه macOS نیز روی نسخه پیشرو باشد:

```bash
./scripts/restart-mac.sh
```

### 1) Gateway توسعه را راه‌اندازی کنید

```bash
pnpm install
# فقط در نخستین اجرا (یا پس از بازنشانی پیکربندی/فضای کاری محلی OpenClaw)
pnpm openclaw setup
pnpm gateway:watch
```

`gateway:watch` فرایند پایش Gateway را در یک نشست نام‌گذاری‌شده tmux
(`openclaw-gateway-watch-main`) راه‌اندازی یا بازراه‌اندازی می‌کند و از ترمینال‌های تعاملی
به‌طور خودکار به آن متصل می‌شود. پوسته‌های غیرتعاملی جدا باقی می‌مانند و
`tmux attach -t openclaw-gateway-watch-main` را نمایش می‌دهند؛ برای جدا نگه‌داشتن یک اجرای تعاملی
از `OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch` یا برای حالت پایش در پیش‌زمینه از
`pnpm gateway:watch:raw` استفاده کنید. پایشگر پیش از در اختیار گرفتن پورت
پیکربندی‌شده/پیش‌فرض، سرویس Gateway نصب‌شده مربوط به پروفایل فعال را
متوقف می‌کند تا ناظر سرویس، فرایند کد منبع را جایگزین نکند. سرویس نصب‌شده
باقی می‌ماند؛ پس از پایان پایش، `pnpm openclaw gateway start` را اجرا کنید. پنل tmux
پس از شکست راه‌اندازی نیز در دسترس می‌ماند تا ترمینال یا عامل دیگری بتواند
به آن متصل شود یا گزارش‌هایش را ثبت کند. پایشگر با تغییرات مرتبط در کد منبع،
پیکربندی و فراداده Pluginهای همراه، بارگذاری مجدد می‌شود. اگر Gateway تحت
پایش هنگام راه‌اندازی خارج شود، `gateway:watch` یک بار
`openclaw doctor --fix --non-interactive` را اجرا کرده و دوباره تلاش می‌کند؛ برای غیرفعال‌کردن
این مرحله ترمیمی مخصوص توسعه، `OPENCLAW_GATEWAY_WATCH_AUTO_DOCTOR=0` را تنظیم کنید.
`pnpm gateway:watch`، `dist/control-ui` را دوباره نمی‌سازد؛ بنابراین پس از
تغییرات `ui/`، `pnpm ui:build` را دوباره اجرا کنید یا هنگام
توسعه رابط کاربری کنترل از `pnpm ui:dev` استفاده کنید.

### 2) برنامه macOS را به Gateway در حال اجرای خود متصل کنید

در **OpenClaw.app**:

- Connection Mode: **Local**
  برنامه به Gateway در حال اجرا روی پورت پیکربندی‌شده متصل می‌شود.

### 3) تأیید

- وضعیت Gateway درون برنامه باید **"Using existing gateway …"** را نشان دهد
- یا از طریق CLI:

```bash
openclaw health
```

### اشتباهات رایج

- **پورت اشتباه:** پورت پیش‌فرض WS مربوط به Gateway، `ws://127.0.0.1:18789` است؛ برنامه و CLI را روی یک پورت نگه دارید.
- **محل نگه‌داری وضعیت:**
  - وضعیت کانال/ارائه‌دهنده: `~/.openclaw/credentials/`
  - پروفایل‌های احراز هویت مدل: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
  - نشست‌ها و رونوشت‌ها: `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`
  - آثار نشست قدیمی/بایگانی‌شده: `~/.openclaw/agents/<agentId>/sessions/`
  - گزارش‌ها: `/tmp/openclaw/`

## نقشه ذخیره‌سازی اطلاعات احراز هویت

هنگام اشکال‌زدایی احراز هویت یا تصمیم‌گیری درباره مواردی که باید پشتیبان‌گیری شوند، از این بخش استفاده کنید:

- **WhatsApp**: `~/.openclaw/credentials/whatsapp/<accountId>/creds.json`
- **توکن ربات Telegram**: پیکربندی/متغیر محیطی یا `channels.telegram.tokenFile` (فقط فایل عادی؛ پیوندهای نمادین رد می‌شوند)
- **توکن ربات Discord**: پیکربندی/متغیر محیطی یا SecretRef (ارائه‌دهندگان env/file/exec)
- **توکن‌های Slack**: پیکربندی/متغیر محیطی (`channels.slack.*`)
- **فهرست‌های مجاز جفت‌سازی**:
  - `~/.openclaw/credentials/<channel>-allowFrom.json` (حساب پیش‌فرض)
  - `~/.openclaw/credentials/<channel>-<accountId>-allowFrom.json` (حساب‌های غیراصلی)
- **پروفایل‌های احراز هویت مدل**: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`
- **محموله اسرار مبتنی بر فایل (اختیاری)**: `~/.openclaw/secrets.json`
- **درون‌ریزی OAuth قدیمی**: `~/.openclaw/credentials/oauth.json`
  جزئیات بیشتر: [امنیت](/fa/gateway/security#credential-storage-map).

## به‌روزرسانی (بدون خراب‌کردن راه‌اندازی)

- `~/.openclaw/workspace` و `~/.openclaw/` را به‌عنوان «موارد شخصی خود» نگه دارید؛ پرامپت‌ها/پیکربندی شخصی را در مخزن `openclaw` قرار ندهید.
- به‌روزرسانی کد منبع: `git pull` + `pnpm install` + ادامه استفاده از `pnpm gateway:watch`.

## Linux (سرویس کاربری systemd)

نصب‌های Linux از یک سرویس **کاربری** systemd استفاده می‌کنند. systemd به‌طور
پیش‌فرض سرویس‌های کاربر را هنگام خروج/بی‌کاری متوقف می‌کند و در نتیجه Gateway
از کار می‌افتد. راه‌اندازی اولیه تلاش می‌کند ماندگاری را برای شما فعال کند
(ممکن است برای sudo درخواست دهد). اگر همچنان غیرفعال است، اجرا کنید:

```bash
sudo loginctl enable-linger $USER
```

برای سرورهای همیشه‌روشن یا چندکاربره، به‌جای سرویس کاربری از یک سرویس
**سیستمی** استفاده کنید (نیازی به ماندگاری نیست). برای نکات systemd به
[راهنمای عملیاتی Gateway](/fa/gateway) مراجعه کنید.

## مستندات مرتبط

- [راهنمای عملیاتی Gateway](/fa/gateway) (پرچم‌ها، نظارت، پورت‌ها)
- [پیکربندی Gateway](/fa/gateway/configuration) (شِمای پیکربندی + نمونه‌ها)
- [Discord](/fa/channels/discord) و [Telegram](/fa/channels/telegram) (برچسب‌های پاسخ + تنظیمات replyToMode)
- [راه‌اندازی دستیار OpenClaw](/fa/start/openclaw)
- [برنامه macOS](/fa/platforms/macos) (چرخه عمر Gateway)
