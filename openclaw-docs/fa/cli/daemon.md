---
read_when:
    - شما همچنان در اسکریپت‌ها از `openclaw daemon ...` استفاده می‌کنید
    - به فرمان‌های چرخهٔ عمر سرویس (نصب/راه‌اندازی/توقف/راه‌اندازی مجدد/وضعیت) نیاز دارید
summary: مرجع CLI برای `openclaw daemon` (نام مستعار قدیمی برای مدیریت سرویس Gateway)
title: دیمون
x-i18n:
    generated_at: "2026-07-27T13:57:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 629852ebf3efe86dedc4c84f6ddc9349b25ddde832df5d78521641fe4b137658
    source_path: cli/daemon.md
    workflow: 16
---

# `openclaw daemon`

نام مستعار قدیمی برای مدیریت سرویس Gateway. `openclaw daemon ...` به همان فرمان‌های کنترل سرویسِ `openclaw gateway ...` نگاشت می‌شود. برای مستندات و نمونه‌های فعلی، [`openclaw gateway`](/fa/cli/gateway) را ترجیح دهید.

## نحوه استفاده

```bash
openclaw daemon status
openclaw daemon install
openclaw daemon start
openclaw daemon stop
openclaw daemon restart
openclaw daemon uninstall
```

## زیرفرمان‌ها و گزینه‌ها

| زیرفرمان  | گزینه‌ها                                                                                          |
| ----------- | ------------------------------------------------------------------------------------------------ |
| `status`    | `--url`, `--token`, `--password`, `--timeout`, `--no-probe`, `--require-rpc`, `--deep`, `--json` |
| `install`   | `--port`, `--runtime <node>`, `--token`, `--wrapper <path>`, `--force`, `--json`                 |
| `uninstall` | `--json`                                                                                         |
| `start`     | `--json`                                                                                         |
| `stop`      | `--json`, `--disable` (فقط launchd: تا شروع بعدی، KeepAlive/RunAtLoad را به‌طور پایدار غیرفعال می‌کند) |
| `restart`   | `--force`, `--safe`, `--skip-deferral`, `--wait <duration>`, `--json`                            |

- `status`: وضعیت نصب سرویس (launchd/systemd/schtasks) را نمایش می‌دهد و سلامت Gateway را بررسی می‌کند.
- `install`: سرویس را نصب می‌کند؛ `--force` نصب موجود را دوباره نصب یا بازنویسی می‌کند.
- `restart --safe`: از Gateway در حال اجرا می‌خواهد کارهای فعال را پیش‌بررسی کند و پس از تخلیه کارها، یک راه‌اندازی مجدد ادغام‌شده را با سقف زمانی 5 دقیقه زمان‌بندی کند. با پایان این مهلت، راه‌اندازی مجدد در هر صورت به‌اجبار انجام می‌شود. `restart` ساده مستقیماً از مدیر سرویس استفاده می‌کند؛ `--force` لغو فوری این رفتار است.
- `restart --safe --skip-deferral`: دروازه تعویق ناشی از کار فعال را دور می‌زند تا Gateway حتی هنگام گزارش موانع، فوراً راه‌اندازی مجدد شود. به `--safe` نیاز دارد.

## نکات

- `status` در صورت امکان، SecretRefهای احراز هویت پیکربندی‌شده را برای احراز هویت بررسی حل می‌کند. اگر یک SecretRef ضروری حل‌نشده باشد، `status --json` مقدار `rpc.authWarning` را گزارش می‌کند؛ `--token`/`--password` را صریحاً ارسال کنید یا ابتدا منبع راز را حل کنید. پس از موفقیت بررسی از سایر جهات، هشدارهای احراز هویت حل‌نشده پنهان می‌شوند.
- `status --deep` یک پویش سطح سیستم با تلاش حداکثری برای یافتن سایر سرویس‌های مشابه Gateway اضافه می‌کند (راهنمای پاک‌سازی را چاپ می‌کند؛ همچنان یک Gateway برای هر دستگاه توصیه می‌شود) و اعتبارسنجی پیکربندی را در حالت آگاه از Plugin اجرا می‌کند و هشدارهای مانیفست Plugin را که مسیر پیش‌فرض سریع نادیده می‌گیرد، نمایش می‌دهد.
- در نصب‌های systemd لینوکس، بررسی‌های انحراف توکن هر دو منبع واحد `Environment=` و `EnvironmentFile=` را بازرسی می‌کنند.
- بررسی‌های انحراف توکن، SecretRefهای `gateway.auth.token` را با استفاده از محیط زمان اجرا‌ی ادغام‌شده حل می‌کنند (ابتدا محیط فرمان سرویس، سپس محیط فرایند). اگر احراز هویت توکنی عملاً فعال نباشد (`gateway.auth.mode` از `password`/`none`/`trusted-proxy`، یا تنظیم‌نشده باشد و گذرواژه بتواند اولویت یابد)، حل توکن پیکربندی نادیده گرفته می‌شود.
- `install` اعتبارسنجی می‌کند که `gateway.auth.token` مدیریت‌شده با SecretRef قابل حل باشد، اما هرگز مقدار حل‌شده را در فراداده محیط سرویس ذخیره نمی‌کند؛ اگر قابل حل نباشد، نصب با حالت بسته شکست می‌خورد.
- اگر هر دو `gateway.auth.token` و `gateway.auth.password` پیکربندی شده باشند و `gateway.auth.mode` تنظیم نشده باشد، `install` تا زمانی که حالت را صریحاً تنظیم کنید، مسدود می‌شود.
- در macOS، `install` به‌جای جاسازی رازها در `EnvironmentVariables`، فایل‌های plist مربوط به LaunchAgent و فایل محیط/پوشاننده تولیدشده را فقط در دسترس مالک نگه می‌دارد (حالت `0600`/`0700`).
- برای اجرای چند Gateway روی یک میزبان، درگاه‌ها، پیکربندی/وضعیت و فضاهای کاری را از هم جدا کنید. [چند Gateway](/fa/gateway#multiple-gateways-same-host) را ببینید.

## مرتبط

- [مرجع CLI](/fa/cli)
- [راهنمای عملیاتی Gateway](/fa/gateway)
