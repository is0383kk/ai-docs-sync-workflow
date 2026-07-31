---
read_when:
    - لازم است لاگ‌های Gateway را از راه دور دنبال کنید (بدون SSH)
    - برای ابزارها به خطوط گزارش JSON نیاز دارید
summary: مرجع CLI برای `openclaw logs` (دنبال‌کردن لاگ‌های Gateway از طریق RPC)
title: گزارش‌ها
x-i18n:
    generated_at: "2026-07-27T13:58:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7c8dc40e70f2eb4f8d6ba8b75b91a33337786a146abbe401079ee374daa5a0c6
    source_path: cli/logs.md
    workflow: 16
---

# `openclaw logs`

لاگ‌های فایل Gateway را از طریق RPC به‌صورت زنده دنبال کنید. در حالت راه دور کار می‌کند.

## گزینه‌ها

- `--limit <n>`: حداکثر تعداد خطوط لاگ برای بازگرداندن (پیش‌فرض `200`)
- `--max-bytes <n>`: حداکثر تعداد بایت برای خواندن از فایل لاگ (پیش‌فرض `250000`)
- `--follow`: دنبال‌کردن جریان لاگ
- `--interval <ms>`: فاصلهٔ زمانی نظرسنجی هنگام دنبال‌کردن (پیش‌فرض `1000`)
- `--json`: انتشار رویدادهای JSON با هر رویداد در یک خط
- `--plain`: خروجی متن ساده بدون قالب‌بندی سبک‌دار
- `--no-color`: غیرفعال‌کردن رنگ‌های ANSI
- `--local-time`: نمایش مُهرهای زمانی در منطقهٔ زمانی محلی شما (پیش‌فرض)
- `--utc`: نمایش مُهرهای زمانی در UTC

## گزینه‌های مشترک RPC در Gateway

- `--url <url>`: نشانی WebSocket مربوط به Gateway
- `--token <token>`: توکن Gateway
- `--timeout <ms>`: مهلت زمانی بر حسب میلی‌ثانیه (پیش‌فرض `30000`)
- `--expect-final`: وقتی فراخوانی Gateway متکی به عامل است، منتظر پاسخ نهایی بمانید

ارسال `--url` اعتبارنامه‌های پیکربندی را که به‌طور خودکار اعمال می‌شوند نادیده می‌گیرد؛ اگر Gateway مقصد به احراز هویت نیاز دارد، `--token` را به‌صراحت وارد کنید.

## نمونه‌ها

```bash
openclaw logs
openclaw logs --follow
openclaw --dev logs --follow
openclaw --profile work logs --follow
openclaw logs --follow --interval 2000
openclaw logs --limit 500 --max-bytes 500000
openclaw logs --json
openclaw logs --plain
openclaw logs --no-color
openclaw logs --utc
openclaw logs --follow --local-time
openclaw logs --url ws://127.0.0.1:18789 --token "$OPENCLAW_GATEWAY_TOKEN"
```

پروفایل ریشهٔ انتخاب‌شده با فایل چرخشی Gateway مطابقت دارد: پروفایل
پیش‌فرض از `openclaw-YYYY-MM-DD.log` استفاده می‌کند، درحالی‌که پروفایل‌های نام‌دار از
`openclaw-<profile>-YYYY-MM-DD.log` استفاده می‌کنند (برای مثال،
`openclaw-dev-YYYY-MM-DD.log`).

## رفتار بازگشت و بازیابی

- اگر Gateway ضمنی روی لوپ‌بک محلی درخواست جفت‌سازی کند، هنگام اتصال بسته شود، یا پیش از پاسخ‌دادن `logs.tail` مهلتش به پایان برسد، `openclaw logs` به‌طور خودکار به لاگ فایل پیکربندی‌شدهٔ Gateway بازمی‌گردد. مقصدهای صریح `--url` هرگز از این بازگشت استفاده نمی‌کنند.
- `--follow` پس از شکست RPC در Gateway محلی ضمنی، به آن فایل پیکربندی‌شده بازنمی‌گردد — یک فایل قدیمی در کنار آن ممکن است در دنبال‌کردن زنده گمراه‌کننده باشد. در Linux، در صورت دسترس‌بودن، به‌جای آن از ژورنال Gateway مربوط به user-systemd فعال بر اساس PID استفاده می‌کند (منبع انتخاب‌شده را نمایش می‌دهد)؛ در غیر این صورت، تلاش برای اتصال مجدد به Gateway زنده را ادامه می‌دهد.
- در طول `--follow`، قطع اتصال‌های گذرا (بسته‌شدن WebSocket، پایان مهلت زمانی، قطع ارتباط) باعث اتصال مجدد خودکار با پس‌روی نمایی می‌شوند: حداکثر 8 تلاش مجدد، با سقف فاصلهٔ 30s بین تلاش‌ها. در هر تلاش مجدد یک هشدار در stderr چاپ می‌شود و پس از موفقیت یک نظرسنجی، اعلان `[logs] gateway reconnected` چاپ می‌شود. در حالت `--json`، هر دو به‌صورت رکوردهای `{"type":"notice"}` در stderr منتشر می‌شوند. خطاهای بازیابی‌ناپذیر (شکست احراز هویت، پیکربندی نامعتبر) همچنان بلافاصله باعث خروج می‌شوند.
- در حالت `--follow --json`، انتقال‌های منبع لاگ به‌صورت رکوردهای `{"type":"meta"}` منتشر می‌شوند. مکان‌نماها را به‌ازای هر `sourceKind` پیگیری کنید: یک جریان می‌تواند از خروجی فایل Gateway (`sourceKind: "file"`) به بازگشت ژورنال محلی (`sourceKind: "journal"`، `localFallback: true`، همراه با `service.pid`/`service.unit`) منتقل شود و پس از بازیابی دوباره به خروجی فایل Gateway بازگردد. برای کل نشست، یک منبع یا مکان‌نمای ثابت را فرض نکنید و هنگام بازپخش مکان‌نمای فایل Gateway در فرایند بازیابی، خطوط هم‌پوشان را بپذیرید.

## مرتبط

- [نمای کلی لاگ‌گیری](/fa/logging)
- [CLI مربوط به Gateway](/fa/cli/gateway)
- [مرجع CLI](/fa/cli)
- [لاگ‌گیری Gateway](/fa/gateway/logging)
