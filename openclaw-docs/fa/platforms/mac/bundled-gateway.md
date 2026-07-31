---
read_when:
    - بسته‌بندی OpenClaw.app
    - اشکال‌زدایی سرویس launchd مربوط به Gateway در macOS
    - نصب CLI مربوط به Gateway برای macOS
summary: زمان اجرای Gateway در macOS (سرویس خارجی launchd)
title: Gateway در macOS
x-i18n:
    generated_at: "2026-07-27T15:24:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 30c1ae14d8f8eaab73d0e2b725292d7411c2c8b5e0e0c32ad13989c01340d054
    source_path: platforms/mac/bundled-gateway.md
    workflow: 16
---

OpenClaw.app شامل Node یا زمان‌اجرای Gateway نیست. برنامه macOS
انتظار دارد CLI `openclaw` به‌صورت **خارجی** نصب شده باشد، Gateway را به‌عنوان
فرایند فرزند اجرا نمی‌کند و برای روشن نگه‌داشتن Gateway یک سرویس launchd مختص هر کاربر
را مدیریت می‌کند (یا به یک Gateway محلی که از قبل در حال اجراست متصل می‌شود).

## راه‌اندازی خودکار

در یک Mac تازه، هنگام شروع‌به‌کار **This Mac** را انتخاب کنید. برنامه پیش از جادوگر
Gateway، اسکریپت نصب‌کننده امضاشده و همراه خود را اجرا می‌کند: یک زمان‌اجرای Node
در فضای کاربر و CLI متناظر `openclaw` را زیر `~/.openclaw` نصب می‌کند،
سپس سرویس launchd مختص هر کاربر را نصب و راه‌اندازی می‌کند. این مسیر به
Terminal، Homebrew یا دسترسی مدیر سیستم نیاز ندارد.

برنامه فقط اسکریپت نصب‌کننده را همراه دارد، نه محتوای Node یا Gateway را؛
راه‌اندازی برای بارگیری زمان‌اجرا و بسته متناظر OpenClaw به اتصال اینترنت نیاز دارد.

## بازیابی دستی

برای نصب دستی، Node 24.15+ توصیه می‌شود؛ Node 22.22.3+ نیز کار می‌کند. بسته
`openclaw` را به‌صورت سراسری نصب کنید:

```bash
npm install -g openclaw@<version>
```

پس از ناموفق بودن راه‌اندازی خودکار، از **Retry setup** استفاده کنید. اگر باز هم ناموفق بود،
CLI را به‌صورت دستی با فرمان بالا نصب کنید، سپس در شروع‌به‌کار **Check again**
را انتخاب کنید.

## Launchd ‏(Gateway به‌عنوان LaunchAgent)

برچسب: `ai.openclaw.gateway` (پروفایل پیش‌فرض)، یا `ai.openclaw.<profile>`
برای یک پروفایل نام‌گذاری‌شده.

محل Plist (مختص هر کاربر): `~/Library/LaunchAgents/ai.openclaw.gateway.plist`
(یا `ai.openclaw.<profile>.plist`).

برنامه macOS در حالت Local مالک نصب/به‌روزرسانی LaunchAgent برای پروفایل پیش‌فرض است.
CLI نیز می‌تواند آن را مستقیماً نصب کند: `openclaw gateway install`
(پروفایل‌های نام‌گذاری‌شده از طریق متغیر محیطی `OPENCLAW_PROFILE` انتخاب می‌شوند).

رفتار:

- «OpenClaw Active»‏ LaunchAgent را فعال/غیرفعال می‌کند.
- خروج از برنامه Gateway را متوقف **نمی‌کند** (launchd آن را فعال نگه می‌دارد).
- اگر Gateway از قبل روی درگاه پیکربندی‌شده در حال اجرا باشد، برنامه به‌جای
  راه‌اندازی یک نمونه جدید به آن متصل می‌شود.

ثبت گزارش:

- خروجی استاندارد launchd:‏ `~/Library/Logs/openclaw/gateway.log` (پروفایل‌ها از
  `gateway-<profile>.log` استفاده می‌کنند)
- خطای استاندارد launchd: سرکوب می‌شود
- اگر میزبان با تکرار `EADDRINUSE` یا راه‌اندازی‌های مجدد سریع وارد حلقه شد،
  LaunchAgentهای تکراری `ai.openclaw.gateway` / `ai.openclaw.node` و راهکار موقت
  نشانگر launchd را در
  [عیب‌یابی Gateway](/fa/gateway/troubleshooting#macos-launchd-supervisor-loop-with-duplicate-gatewaynode-launchagents) بررسی کنید.

## سازگاری نسخه

برنامه macOS نسخه Gateway را با نسخه خودش مقایسه می‌کند. اگر CLI موجود نباشد یا
ناسازگار باشد، شروع‌به‌کار به‌طور خودکار راه‌اندازی مدیریت‌شده را اجرا می‌کند.
برای تکرار نصب از **Retry setup** یا پس از تعمیر یک CLI خارجی از **Check again**
استفاده کنید.

## پوشه وضعیت در macOS

وضعیت OpenClaw را روی یک دیسک محلی و همگام‌نشده نگه دارید. از iCloud Drive و دیگر
پوشه‌های همگام‌شده با فضای ابری دوری کنید؛ تأخیر همگام‌سازی و قفل فایل‌ها می‌تواند بر نشست‌ها،
اعتبارنامه‌ها و وضعیت Gateway اثر بگذارد.

فقط زمانی که نیاز به بازنویسی دارید، `OPENCLAW_STATE_DIR` را روی یک مسیر محلی تنظیم کنید.
`openclaw doctor` درباره مسیرهای رایج وضعیت همگام‌شده با فضای ابری هشدار می‌دهد و
بازگشت به ذخیره‌سازی محلی را توصیه می‌کند. به
[متغیرهای محیطی](/fa/help/environment#path-related-env-vars) و
[Doctor](/fa/gateway/doctor) مراجعه کنید.

## اشکال‌زدایی اتصال برنامه

برای آزمایش همان دست‌دهی WebSocket و منطق کشف Gateway که برنامه استفاده می‌کند،
از CLI اشکال‌زدایی macOS در یک checkout منبع استفاده کنید:

```bash
cd apps/macos
swift run openclaw-mac connect --json
swift run openclaw-mac discover --timeout 3000 --json
```

`connect` گزینه‌های `--url`، `--token`، `--timeout`، `--probe` و `--json`
را می‌پذیرد (به‌علاوه بازنویسی‌های هویت کلاینت؛ برای فهرست کامل با `--help` اجرا کنید).
`discover` گزینه‌های `--timeout`، `--json` و `--include-local` را می‌پذیرد. هنگامی که نیاز دارید
کشف CLI را از مشکلات اتصال سمت برنامه تفکیک کنید، خروجی کشف را با `openclaw gateway discover --json`
مقایسه کنید.

## بررسی سریع

```bash
openclaw --version

OPENCLAW_SKIP_CHANNELS=1 \
OPENCLAW_SKIP_CANVAS_HOST=1 \
openclaw gateway --port 18999 --bind loopback
```

سپس:

```bash
openclaw gateway call health --url ws://127.0.0.1:18999 --timeout 3000
```

## مرتبط

- [برنامه macOS](/fa/platforms/macos)
- [راهنمای عملیاتی Gateway](/fa/gateway)
