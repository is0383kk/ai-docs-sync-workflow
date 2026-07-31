---
read_when:
    - می‌خواهید هوک‌های عامل را مدیریت کنید
    - می‌خواهید در دسترس بودن هوک‌ها را بررسی کنید یا هوک‌های فضای کاری را فعال کنید
summary: مرجع CLI برای `openclaw hooks` (هوک‌های عامل)
title: هوک‌ها
x-i18n:
    generated_at: "2026-07-27T16:19:41Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d4d58ea2270cf5122018f7be2943401229929f48f448b15fdd126d1cc99e1e56
    source_path: cli/hooks.md
    workflow: 16
---

# `openclaw hooks`

قلاب‌های عامل را مدیریت کنید (خودکارسازی‌های رویدادمحور برای فرمان‌هایی مانند `/new`، `/reset` و راه‌اندازی Gateway). استفادهٔ تنها از `openclaw hooks` معادل `openclaw hooks list` است.

مرتبط: [قلاب‌ها](/fa/automation/hooks) - [قلاب‌های Plugin](/fa/plugins/hooks)

## فهرست قلاب‌ها

```bash
openclaw hooks list [--eligible] [--json] [-v|--verbose]
```

قلاب‌های کشف‌شده در دایرکتوری‌های فضای کاری، مدیریت‌شده، اضافی و همراه را فهرست می‌کند.

- `--eligible`: فقط قلاب‌هایی که الزاماتشان برآورده شده است.
- `--json`: خروجی ساخت‌یافته.
- `-v, --verbose`: یک ستون Missing شامل الزامات برآورده‌نشده اضافه می‌کند.

```
قلاب‌ها (4/5 آماده)

آماده:
  🚀 boot-md ✓ - اجرای BOOT.md هنگام راه‌اندازی Gateway
  📎 bootstrap-extra-files ✓ - تزریق فایل‌های راه‌انداز اضافی فضای کاری هنگام راه‌اندازی عامل
  📝 command-logger ✓ - ثبت همهٔ رویدادهای فرمان در یک فایل ممیزی متمرکز
  💾 session-memory ✓ - ذخیرهٔ زمینهٔ نشست در حافظه هنگام صدور فرمان /new یا /reset
```

## دریافت اطلاعات قلاب

```bash
openclaw hooks info <name> [--json]
```

`<name>` نام یا کلید قلاب است (برای مثال `session-memory`). منبع، مسیرهای فایل/مدیریت‌کننده، صفحهٔ اصلی، رویدادها و وضعیت هر الزام (فایل‌های اجرایی، محیط، پیکربندی، سیستم‌عامل) را نمایش می‌دهد.

## بررسی واجد شرایط بودن

```bash
openclaw hooks check [--json]
```

خلاصه‌ای از تعداد آماده/ناآماده را چاپ می‌کند؛ اگر قلاب‌هایی آماده نباشند، هرکدام را با دلیل بازدارندهٔ آن فهرست می‌کند.

## فعال‌کردن یک قلاب

```bash
openclaw hooks enable <name>
```

`hooks.internal.entries.<name>.enabled = true` را در پیکربندی اضافه/به‌روزرسانی می‌کند و کلید اصلی `hooks.internal.enabled` را نیز روشن می‌کند (Gateway تا زمانی که دست‌کم یک قلاب پیکربندی نشده باشد، هیچ مدیریت‌کنندهٔ داخلی قلاب را بارگذاری نمی‌کند). اگر قلاب وجود نداشته باشد، تحت مدیریت Plugin باشد یا واجد شرایط نباشد (الزامات مفقود باشند)، عملیات ناموفق می‌شود.

قلاب‌های تحت مدیریت Plugin در `hooks list` مقدار `plugin:<id>` را نشان می‌دهند و از اینجا نمی‌توان آن‌ها را فعال/غیرفعال کرد؛ در عوض Plugin مالک را فعال یا غیرفعال کنید.

پس از فعال‌سازی، Gateway را بازراه‌اندازی کنید (بازراه‌اندازی برنامهٔ نوار منوی macOS یا بازراه‌اندازی فرایند Gateway در محیط توسعه) تا قلاب‌ها را دوباره بارگذاری کند.

## غیرفعال‌کردن یک قلاب

```bash
openclaw hooks disable <name>
```

`hooks.internal.entries.<name>.enabled = false` را تنظیم می‌کند. سپس Gateway را بازراه‌اندازی کنید.

## نصب و به‌روزرسانی بسته‌های قلاب

```bash
openclaw plugins install <package>        # npm به‌طور پیش‌فرض
openclaw plugins install npm:<package>    # فقط npm
openclaw plugins install <package> --pin  # سنجاق‌کردن نسخهٔ تعیین‌شده
openclaw plugins install <path>           # دایرکتوری یا بایگانی محلی
openclaw plugins install -l <path>        # پیوند به یک دایرکتوری محلی به‌جای کپی‌کردن

openclaw plugins update <id>
openclaw plugins update --all
openclaw plugins update --dry-run
```

بسته‌های قلاب از طریق نصب‌کننده/به‌روزرسان یکپارچهٔ Pluginها نصب می‌شوند؛ `openclaw hooks install` / `openclaw hooks update` همچنان به‌عنوان نام‌های مستعار منسوخ‌شده کار می‌کنند که هشداری چاپ می‌کنند و به فرمان‌های `plugins` هدایت می‌شوند.

- مشخصات Npm فقط به رجیستری محدودند: نام بسته به‌همراه یک نسخهٔ دقیق یا dist-tag اختیاری. مشخصات Git/URL/file و بازه‌های semver رد می‌شوند. وابستگی‌ها به‌صورت محلی در پروژه با `--ignore-scripts` نصب می‌شوند.
- مشخصات بدون پیشوند و `@latest` در مسیر پایدار باقی می‌مانند؛ اگر npm یک نسخهٔ پیش‌انتشار را تعیین کند، OpenClaw متوقف می‌شود و از شما می‌خواهد صریحاً آن را بپذیرید (`@beta`، `@rc` یا یک نسخهٔ دقیق پیش‌انتشار).
- بایگانی‌های پشتیبانی‌شده: `.zip`، `.tgz`، `.tar.gz`، `.tar`.
- `-l, --link` به‌جای کپی‌کردن، یک دایرکتوری محلی را پیوند می‌دهد (آن را به `hooks.internal.load.extraDirs` اضافه می‌کند)؛ بسته‌های قلاب پیوندشده، قلاب‌های مدیریت‌شده از یک دایرکتوری پیکربندی‌شده توسط اپراتور هستند، نه قلاب‌های فضای کاری.
- `--pin` نصب‌های npm را به‌صورت `name@version` دقیق و تعیین‌شده در وضعیت SQLite مشترک ثبت می‌کند.
- نصب، بسته را در `~/.openclaw/hooks/<id>` کپی می‌کند، قلاب‌های آن را زیر `hooks.internal.entries.*` فعال می‌کند و منشأ نصب را در وضعیت SQLite مشترک ثبت می‌کند.
- اگر هش یکپارچگی ذخیره‌شده دیگر با دست‌ساختهٔ دریافت‌شده مطابقت نداشته باشد، OpenClaw هشدار می‌دهد و پیش از ادامه تأیید می‌خواهد؛ برای دورزدن این درخواست، `--yes` سراسری را ارسال کنید (برای مثال در CI).

## قلاب‌های همراه

| قلاب                  | رویدادها                                            | کاری که انجام می‌دهد                                                                                       |
| --------------------- | ------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| boot-md               | `gateway:startup`                                 | برای هر محدودهٔ عامل پیکربندی‌شده، `BOOT.md` را هنگام راه‌اندازی Gateway اجرا می‌کند                                  |
| bootstrap-extra-files | `agent:bootstrap`                                 | فایل‌های راه‌انداز اضافی (برای مثال `AGENTS.md`/`TOOLS.md` در monorepo) را هنگام راه‌اندازی عامل تزریق می‌کند |
| command-logger        | `command`                                         | رویدادهای فرمان را در `~/.openclaw/logs/commands.log` ثبت می‌کند                                             |
| compaction-notifier   | `session:compact:before`, `session:compact:after` | هنگام شروع و پایان Compaction نشست، اعلان‌های قابل‌مشاهده‌ای در گفت‌وگو ارسال می‌کند                             |
| session-memory        | `command:new`, `command:reset`                    | زمینهٔ نشست را هنگام `/new` یا `/reset` در حافظه ذخیره می‌کند                                              |

هر قلاب همراه را با `openclaw hooks enable <hook-name>` فعال کنید. جزئیات کامل، کلیدهای پیکربندی و پیش‌فرض‌ها: [قلاب‌های همراه](/fa/automation/hooks#bundled-hooks).

### فایل گزارش command-logger

```bash
tail -n 20 ~/.openclaw/logs/commands.log        # فرمان‌های اخیر
cat ~/.openclaw/logs/commands.log | jq .          # چاپ خوانا
grep '"action":"new"' ~/.openclaw/logs/commands.log | jq .   # پالایش بر اساس کنش
```

## نکات

- `hooks list --json`، `info --json` و `check --json`، JSON ساخت‌یافته را مستقیماً در stdout می‌نویسند.

## مرتبط

- [مرجع CLI](/fa/cli)
- [قلاب‌های خودکارسازی](/fa/automation/hooks)
