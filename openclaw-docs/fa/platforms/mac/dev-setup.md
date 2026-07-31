---
read_when:
    - راه‌اندازی محیط توسعه macOS
summary: راهنمای راه‌اندازی برای توسعه‌دهندگانی که روی برنامه macOS ‏OpenClaw کار می‌کنند
title: راه‌اندازی محیط توسعه در macOS
x-i18n:
    generated_at: "2026-07-27T16:46:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ff72bb449e70b94b8a13504414955ab7fe411a674b65e670939484a5863b5f48
    source_path: platforms/mac/dev-setup.md
    workflow: 16
---

# راه‌اندازی توسعه در macOS

برنامهٔ macOS متعلق به OpenClaw را از کد منبع بسازید و اجرا کنید.

## پیش‌نیازها

- **Xcode 26.2+** (زنجیره‌ابزار Swift 6.2)، روی جدیدترین نسخهٔ macOS موجود در
  Software Update.
- **Node.js 24.15+ و pnpm** برای Gateway، CLI و اسکریپت‌های بسته‌بندی. Node
  22.22.3+ نیز کار می‌کند.

## 1. نصب وابستگی‌ها

```bash
pnpm install
```

## 2. ساخت و بسته‌بندی برنامه

```bash
./scripts/package-mac-app.sh
```

خروجی در `dist/OpenClaw.app` قرار می‌گیرد. بدون گواهی Apple Developer ID،
اسکریپت از امضای موقت استفاده می‌کند.

برای حالت‌های اجرای توسعه، پرچم‌های امضا و عیب‌یابی Team ID، به
[apps/macos/README.md](https://github.com/openclaw/openclaw/blob/main/apps/macos/README.md) مراجعه کنید.
چرخهٔ سریع توسعه از ریشهٔ مخزن: `scripts/restart-mac.sh` (برای امضای موقت،
`--no-sign` را اضافه کنید؛ مجوزهای TCC با `--no-sign` حفظ نمی‌شوند).

<Note>
برنامه‌های دارای امضای موقت ممکن است پیام‌های امنیتی نمایش دهند. اگر برنامه
بلافاصله با خطای "Abort trap 6" از کار می‌افتد، به [عیب‌یابی](#troubleshooting) مراجعه کنید.
</Note>

## 3. نصب CLI و Gateway

برنامهٔ بسته‌بندی‌شده، نصب‌کنندهٔ استاندارد `scripts/install-cli.sh` را در خود دارد. در یک
پروفایل تازه، هنگام راه‌اندازی اولیه **This Mac** را انتخاب کنید؛ برنامه پیش از شروع
جادوگر Gateway، CLI و محیط اجرای همسان را در فضای کاربر نصب می‌کند.

برای بازیابی دستی محیط توسعه، CLI همسان را خودتان نصب کنید:

```bash
npm install -g openclaw@<version>
```

`pnpm add -g openclaw@<version>` و `bun add -g openclaw@<version>` نیز
کار می‌کنند. Node همچنان محیط اجرای پیشنهادی برای خود Gateway است.

## عیب‌یابی

### شکست ساخت: ناهماهنگی زنجیره‌ابزار یا SDK

ساخت برنامهٔ macOS به جدیدترین SDK مربوط به macOS و زنجیره‌ابزار Swift 6.2
(Xcode 26.2+) نیاز دارد.

```bash
xcodebuild -version
xcrun swift --version
```

اگر نسخه‌ها مطابقت ندارند، macOS/Xcode را به‌روزرسانی و ساخت را دوباره اجرا کنید.

### از کار افتادن برنامه هنگام اعطای مجوز

اگر برنامه هنگام تلاش برای اجازه‌دادن دسترسی **Speech Recognition** یا
**Microphone** از کار می‌افتد، ممکن است حافظهٔ نهان TCC خراب یا امضا ناهماهنگ باشد.

1. مجوزهای TCC را برای شناسهٔ بستهٔ اشکال‌زدایی بازنشانی کنید:

   ```bash
   tccutil reset All ai.openclaw.mac.debug
   ```

2. اگر این روش ناموفق بود، `BUNDLE_ID` را به‌طور موقت در
   [`scripts/package-mac-app.sh`](https://github.com/openclaw/openclaw/blob/main/scripts/package-mac-app.sh)
   تغییر دهید تا macOS از یک وضعیت پاک شروع کند.

### باقی‌ماندن Gateway روی "Starting..." برای مدتی نامحدود

بررسی کنید آیا فرایندی زامبی پورت را در اختیار دارد:

```bash
openclaw gateway status
openclaw gateway stop

# اگر از LaunchAgent استفاده نمی‌کنید (حالت توسعه / اجرای دستی)، فرایند شنونده را پیدا کنید:
lsof -nP -iTCP:18789 -sTCP:LISTEN
```

اگر یک اجرای دستی پورت را در اختیار دارد، آن را متوقف کنید (Ctrl+C)، یا در
آخرین راه‌حل، PID یافت‌شده در بالا را خاتمه دهید.

## مرتبط

- [برنامهٔ macOS](/fa/platforms/macos)
- [نمای کلی نصب](/fa/install)
