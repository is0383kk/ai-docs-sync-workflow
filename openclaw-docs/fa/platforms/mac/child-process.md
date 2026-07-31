---
read_when:
    - یکپارچه‌سازی اپ مک با چرخهٔ عمر Gateway
summary: چرخهٔ حیات Gateway در macOS (launchd)
title: چرخه عمر Gateway در macOS
x-i18n:
    generated_at: "2026-07-27T16:44:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89a27334afcecb322feb2732cf6282b4c286ef27828a1b57157f9d4fc161aed6
    source_path: platforms/mac/child-process.md
    workflow: 16
---

برنامه macOS به‌طور پیش‌فرض Gateway را از طریق **launchd** مدیریت می‌کند و
Gateway را به‌عنوان فرایند فرزند اجرا نمی‌کند. برنامه ابتدا تلاش می‌کند به
Gateway در حال اجرایی روی پورت پیکربندی‌شده متصل شود؛ اگر هیچ‌کدام در دسترس نباشد،
سرویس launchd را از طریق CLI خارجی `openclaw` فعال می‌کند (بدون runtime
تعبیه‌شده). این کار شروع خودکار قابل‌اعتماد هنگام ورود و راه‌اندازی مجدد پس از خرابی را فراهم می‌کند.

حالت فرایند فرزند (اجرای مستقیم Gateway توسط برنامه) امروزه **استفاده نمی‌شود**.
اگر به اتصال نزدیک‌تری با رابط کاربری نیاز است، Gateway را به‌صورت دستی در یک
ترمینال اجرا کنید.

## رفتار پیش‌فرض (launchd)

- برنامه یک LaunchAgent مخصوص هر کاربر با برچسب `ai.openclaw.gateway` نصب می‌کند (یا
  هنگام استفاده از `--profile`/`OPENCLAW_PROFILE`، برچسب `ai.openclaw.<profile>`).
- وقتی حالت محلی فعال است، برنامه از بارگذاری‌شدن LaunchAgent اطمینان حاصل می‌کند و
  در صورت نیاز Gateway را راه‌اندازی می‌کند.
- لاگ‌ها در مسیر لاگ Gateway مربوط به launchd نوشته می‌شوند (در تنظیمات اشکال‌زدایی قابل مشاهده است).

دستورهای رایج:

```bash
launchctl kickstart -k gui/$UID/ai.openclaw.gateway
launchctl bootout gui/$UID/ai.openclaw.gateway
```

هنگام اجرای یک پروفایل نام‌گذاری‌شده، برچسب را با `ai.openclaw.<profile>` جایگزین کنید.

## بیلدهای توسعه امضانشده

`scripts/restart-mac.sh --no-sign` برای بیلدهای محلی سریع بدون کلیدهای امضا است.
برای جلوگیری از اشاره launchd به یک باینری relay امضانشده، این فرایند
`~/.openclaw/disable-launchagent` را می‌نویسد.

اجراهای امضاشده `scripts/restart-mac.sh` در صورت وجود نشانگر، این بازنویسی را
پاک می‌کنند. برای بازنشانی دستی:

```bash
rm ~/.openclaw/disable-launchagent
```

## حالت فقط اتصال

برای اینکه برنامه macOS هرگز launchd را نصب یا مدیریت نکند، آن را با
`--attach-only` (یا `--no-launchd`) اجرا کنید. این کار
`~/.openclaw/disable-launchagent` را تنظیم می‌کند تا برنامه فقط به Gateway از پیش
در حال اجرا متصل شود. همین رفتار را می‌توان در تنظیمات اشکال‌زدایی تغییر داد.

## حالت راه دور

حالت راه دور هرگز Gateway محلی را راه‌اندازی نمی‌کند. برنامه از یک تونل SSH به
میزبان راه دور استفاده می‌کند و از طریق آن تونل متصل می‌شود.

## چرا launchd را ترجیح می‌دهیم

- شروع خودکار هنگام ورود.
- معانی داخلی راه‌اندازی مجدد/KeepAlive.
- لاگ‌ها و نظارت پیش‌بینی‌پذیر.

اگر دوباره به یک حالت واقعی فرایند فرزند نیاز شود، باید به‌عنوان
حالتی جداگانه، صریح و فقط مخصوص توسعه مستند شود.

## مرتبط

- [برنامه macOS](/fa/platforms/macos)
- [راهنمای عملیاتی Gateway](/fa/gateway)
