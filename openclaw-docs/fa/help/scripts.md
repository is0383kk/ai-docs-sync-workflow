---
read_when:
    - اجرای اسکریپت‌ها از مخزن کد
    - افزودن یا تغییر اسکریپت‌ها در `./scripts`
summary: 'اسکریپت‌های مخزن: هدف، دامنه و نکات ایمنی'
title: اسکریپت‌ها
x-i18n:
    generated_at: "2026-07-27T14:14:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 323069190ea6647101ee7120e06f6b2a018833d0904a11787fa1b610f5b3d9e1
    source_path: help/scripts.md
    workflow: 16
---

`scripts/` شامل اسکریپت‌های کمکی برای گردش‌کارهای محلی و وظایف عملیاتی است. وقتی یک وظیفه به‌وضوح به اسکریپتی مرتبط است، از این موارد استفاده کنید؛ در غیر این صورت، CLI را ترجیح دهید.

## قراردادها

- اسکریپت‌ها **اختیاری** هستند، مگر اینکه در مستندات یا چک‌لیست‌های انتشار به آن‌ها ارجاع شده باشد.
- هرگاه سطوح CLI وجود دارند، آن‌ها را ترجیح دهید (مثال: `openclaw models status --check`).
- فرض کنید اسکریپت‌ها مختص میزبان هستند؛ پیش از اجرای آن‌ها روی دستگاهی جدید، آن‌ها را بخوانید.

## اسکریپت‌های پایش احراز هویت

احراز هویت عمومی مدل در [احراز هویت](/fa/gateway/authentication) توضیح داده شده است. اسکریپت‌های زیر سامانه‌ای جداگانه و اختیاری برای پایش **توکن اشتراک Claude Code CLI** روی میزبانی راه‌دور یا بدون رابط گرافیکی و احراز هویت مجدد از طریق تلفن هستند:

- `scripts/setup-auth-system.sh` - راه‌اندازی یک‌باره: احراز هویت فعلی را بررسی می‌کند، به ایجاد یک `claude setup-token` با طول عمر زیاد کمک می‌کند و مراحل نصب systemd/Termux را نمایش می‌دهد.
- `scripts/claude-auth-status.sh [full|json|simple]` - وضعیت احراز هویت Claude Code و OpenClaw را بررسی می‌کند.
- `scripts/auth-monitor.sh` - وضعیت را به‌طور دوره‌ای بررسی می‌کند و هنگامی که توکن به انقضا نزدیک می‌شود، اعلانی ارسال می‌کند (از طریق ارسال OpenClaw و/یا ntfy.sh). متغیرهای محیطی: `WARN_HOURS` (پیش‌فرض `2`)، `NOTIFY_PHONE`، `NOTIFY_NTFY`. آن را طبق برنامه با `scripts/systemd/openclaw-auth-monitor.{service,timer}` همراه اجرا کنید (هر 30 دقیقه).
- `scripts/mobile-reauth.sh` - `claude setup-token` را دوباره اجرا می‌کند و URLهایی را برای باز کردن در تلفن نمایش می‌دهد تا از طریق SSH در Termux استفاده شوند.
- `scripts/termux-quick-auth.sh`، `scripts/termux-auth-widget.sh`، `scripts/termux-sync-widget.sh` - اسکریپت‌های Termux:Widget که از طریق SSH به میزبان متصل می‌شوند، یک پیام کوتاه وضعیت نمایش می‌دهند و در صورت منقضی‌شدن احراز هویت، کنسول/دستورالعمل‌های احراز هویت مجدد را باز می‌کنند.

## ابزار کمکی خواندن GitHub

وقتی می‌خواهید `gh` برای فراخوانی‌های خواندن محدود به مخزن از توکن نصب GitHub App استفاده کند و در عین حال `gh` عادی برای عملیات نوشتن با ورود شخصی شما باقی بماند، از `scripts/gh-read` استفاده کنید.

متغیرهای محیطی الزامی:

- `OPENCLAW_GH_READ_APP_ID`
- `OPENCLAW_GH_READ_PRIVATE_KEY_FILE`

متغیرهای محیطی اختیاری:

- `OPENCLAW_GH_READ_INSTALLATION_ID` برای زمانی که می‌خواهید جست‌وجوی نصب مبتنی بر مخزن را نادیده بگیرید
- `OPENCLAW_GH_READ_PERMISSIONS` به‌عنوان جایگزینی با مقادیر جداشده با ویرگول برای زیرمجموعه مجوزهای خواندن درخواستی

ترتیب تشخیص مخزن:

- `gh ... -R owner/repo`
- `GH_REPO`
- `git remote origin`

مثال‌ها:

- `scripts/gh-read pr view 123`
- `scripts/gh-read run list -R openclaw/openclaw`
- `scripts/gh-read api repos/openclaw/openclaw/pulls/123`

## هنگام افزودن اسکریپت‌ها

- اسکریپت‌ها را متمرکز و مستند نگه دارید.
- یک مدخل کوتاه به سند مرتبط اضافه کنید (یا اگر وجود ندارد، یکی ایجاد کنید).

## مرتبط

- [آزمایش](/fa/help/testing)
- [آزمایش زنده](/fa/help/testing-live)
