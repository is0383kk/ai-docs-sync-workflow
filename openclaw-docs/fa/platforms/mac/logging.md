---
read_when:
    - ثبت لاگ‌های macOS یا بررسی ثبت داده‌های خصوصی
    - اشکال‌زدایی مشکلات چرخهٔ حیات بیدارباش صوتی/نشست
summary: 'گزارش‌گیری OpenClaw: گزارش فایل تشخیصی چرخشی + پرچم‌های یکپارچهٔ حریم خصوصی گزارش‌ها'
title: ثبت رویدادهای macOS
x-i18n:
    generated_at: "2026-07-27T15:32:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ef0fd91bd7fc0a8b5f598cfe8f5de551795a4badd0f6634c5bcbd4f3916bfc64
    source_path: platforms/mac/logging.md
    workflow: 16
---

# گزارش‌گیری (macOS)

## گزارش فایل تشخیصی چرخشی (پنل اشکال‌زدایی)

برنامه macOS از طریق swift-log گزارش ثبت می‌کند (به‌طور پیش‌فرض با گزارش‌گیری یکپارچه) و همچنین می‌تواند برای ثبت پایدار، گزارش فایل محلی چرخشی بنویسد (`DiagnosticsFileLog`).

- فعال‌سازی: **Debug pane -> Logs -> App logging -> "Write rolling diagnostics log (JSONL)"** (به‌طور پیش‌فرض غیرفعال است).
- میزان جزئیات: انتخاب‌گر **Debug pane -> Logs -> App logging -> Verbosity**.
- مکان: `~/Library/Logs/OpenClaw/diagnostics.jsonl`.
- چرخش: در 5 MB می‌چرخد؛ حداکثر 5 نسخه پشتیبان با پسوندهای `.1`...`.5` نگه‌داری می‌شود (قدیمی‌ترین حذف می‌شود).
- پاک‌سازی: گزینه **Debug pane -> Logs -> App logging -> "Clear"** فایل فعال و همه نسخه‌های پشتیبان را حذف می‌کند.

این فایل را حساس تلقی کنید؛ بدون بازبینی آن را به اشتراک نگذارید.

## داده‌های خصوصی گزارش‌گیری یکپارچه در macOS

گزارش‌گیری یکپارچه بیشتر محتوای بارها را پنهان می‌کند، مگر اینکه یک زیرسیستم `privacy -off` را فعال کند. این رفتار با یک plist در `/Library/Preferences/Logging/Subsystems/` که بر اساس نام زیرسیستم کلیدگذاری شده است کنترل می‌شود. فقط ورودی‌های گزارش جدید این پرچم را اعمال می‌کنند، بنابراین پیش از بازتولید مشکل آن را فعال کنید. پیش‌زمینه: [ترفندهای حریم خصوصی گزارش‌گیری macOS](https://steipete.me/posts/2025/logging-privacy-shenanigans).

## فعال‌سازی برای OpenClaw (`ai.openclaw`)

ابتدا plist را در یک فایل موقت بنویسید، سپس آن را به‌صورت اتمی و با کاربر root نصب کنید:

```bash
cat <<'EOF' >/tmp/ai.openclaw.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>DEFAULT-OPTIONS</key>
    <dict>
        <key>Enable-Private-Data</key>
        <true/>
    </dict>
</dict>
</plist>
EOF
sudo install -m 644 -o root -g wheel /tmp/ai.openclaw.plist /Library/Preferences/Logging/Subsystems/ai.openclaw.plist
```

نیازی به راه‌اندازی مجدد نیست؛ logd فایل را سریع شناسایی می‌کند، اما فقط خطوط گزارش جدید شامل محتوای خصوصی خواهند بود. خروجی غنی‌تر را با `./scripts/clawlog.sh --category WebChat --last 5m` مشاهده کنید (`--last`/`-l` بازه زمانی را تعیین می‌کند؛ مقدار پیش‌فرض `5m` است؛ `--category`/`-c` بر اساس دسته فیلتر می‌کند).

## غیرفعال‌سازی پس از اشکال‌زدایی

- بازنویسی را حذف کنید: `sudo rm /Library/Preferences/Logging/Subsystems/ai.openclaw.plist`.
- در صورت تمایل، `sudo log config --reload` را اجرا کنید تا logd فوراً بازنویسی را کنار بگذارد.
- این سطح ممکن است شامل شماره تلفن‌ها و متن پیام‌ها باشد؛ plist را فقط تا زمانی نگه دارید که فعالانه به آن نیاز دارید.

## مرتبط

- [برنامه macOS](/fa/platforms/macos)
- [گزارش‌گیری Gateway](/fa/gateway/logging)
