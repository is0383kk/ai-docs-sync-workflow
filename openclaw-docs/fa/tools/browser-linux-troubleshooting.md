---
read_when: Browser control fails on Linux, especially with snap Chromium
summary: رفع مشکلات راه‌اندازی CDP در Chrome/Brave/Edge/Chromium برای کنترل مرورگر OpenClaw در Linux
title: عیب‌یابی مرورگر
x-i18n:
    generated_at: "2026-07-27T17:11:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8e5db2da2d43129862f0c005213df828f6eae81f5561e57d41795ea90787822a
    source_path: tools/browser-linux-troubleshooting.md
    workflow: 16
---

## مشکل: راه‌اندازی Chrome CDP روی پورت 18800 ناموفق بود

```json
{ "error": "خطا: راه‌اندازی Chrome CDP روی پورت 18800 برای پروفایل \"openclaw\" ناموفق بود." }
```

### علت اصلی

در Ubuntu و بیشتر توزیع‌های Linux، `apt install chromium` یک پوشش snap نصب
می‌کند، نه یک مرورگر واقعی:

```text
توجه، به‌جای 'chromium'، 'chromium-browser' انتخاب می‌شود
chromium-browser از قبل جدیدترین نسخه است (2:1snap1-0ubuntu2).
```

محدودسازی AppArmor در Snap با نحوه راه‌اندازی و پایش فرایند
مرورگر توسط OpenClaw تداخل دارد.

دیگر خطاهای متداول راه‌اندازی در Linux:

- `The profile appears to be in use by another Chromium process`: فایل‌های قفل قدیمی
  `Singleton*` در پوشه پروفایل مدیریت‌شده. OpenClaw این قفل‌ها را حذف
  می‌کند و وقتی قفل به فرایندی متوقف‌شده یا متعلق به میزبانی
  دیگر اشاره کند، یک‌بار دیگر تلاش می‌کند.
- `Missing X server or $DISPLAY`: مرورگر قابل‌مشاهده به‌صراحت روی
  میزبانی بدون نشست دسکتاپ درخواست شده است. پروفایل‌های مدیریت‌شده محلی در Linux، وقتی هر دو `DISPLAY` و `WAYLAND_DISPLAY` تنظیم نشده باشند،
  به حالت بدون رابط گرافیکی برمی‌گردند.
  اگر `OPENCLAW_BROWSER_HEADLESS=0`، `browser.headless: false` یا
  `browser.profiles.<name>.headless: false` را تنظیم کرده‌اید، آن بازنویسی حالت دارای رابط گرافیکی را حذف کنید،
  `OPENCLAW_BROWSER_HEADLESS=1` را تنظیم کنید، `Xvfb` را راه‌اندازی کنید،
  برای یک راه‌اندازی مدیریت‌شده یک‌باره `openclaw browser start --headless` را اجرا کنید، یا
  OpenClaw را در یک نشست دسکتاپ واقعی اجرا کنید.

### راه‌حل 1: نصب Google Chrome (توصیه‌شده)

```bash
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y  # اگر خطاهای وابستگی وجود دارد
```

`~/.openclaw/openclaw.json` را به‌روزرسانی کنید:

```json
{
  "browser": {
    "enabled": true,
    "executablePath": "/usr/bin/google-chrome-stable",
    "headless": true,
    "noSandbox": true
  }
}
```

### راه‌حل 2: استفاده از snap Chromium در حالت فقط اتصال

اگر باید snap Chromium را نگه دارید، OpenClaw را پیکربندی کنید تا به‌جای
راه‌اندازی مرورگر، به مرورگری که به‌صورت دستی راه‌اندازی شده است متصل شود:

```json
{
  "browser": {
    "enabled": true,
    "attachOnly": true,
    "headless": true,
    "noSandbox": true
  }
}
```

Chromium را به‌صورت دستی راه‌اندازی کنید:

```bash
chromium-browser --headless --no-sandbox --disable-gpu \
  --remote-debugging-port=18800 \
  --user-data-dir=$HOME/.openclaw/browser/openclaw/user-data \
  about:blank &
```

در صورت تمایل، آن را با یک سرویس کاربری systemd به‌طور خودکار راه‌اندازی کنید:

```ini
# ~/.config/systemd/user/openclaw-browser.service
[Unit]
Description=مرورگر OpenClaw (Chrome CDP)
After=network.target

[Service]
ExecStart=/snap/bin/chromium --headless --no-sandbox --disable-gpu --remote-debugging-port=18800 --user-data-dir=%h/.openclaw/browser/openclaw/user-data about:blank
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
```

```bash
systemctl --user enable --now openclaw-browser.service
```

### بررسی عملکرد مرورگر

```bash
curl -s http://127.0.0.1:18791/ | jq '{running, pid, chosenBrowser}'
curl -s -X POST http://127.0.0.1:18791/start
curl -s http://127.0.0.1:18791/tabs
```

### مرجع پیکربندی

| گزینه                      | توضیحات                                                          | پیش‌فرض                                                            |
| --------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `browser.enabled`           | فعال‌سازی کنترل مرورگر                                               | `true`                                                             |
| `browser.executablePath`    | مسیر فایل اجرایی مرورگر مبتنی بر Chromium ‏(Chrome/Brave/Edge/Chromium) | شناسایی خودکار (اگر مرورگر پیش‌فرض سیستم‌عامل مبتنی بر Chromium باشد، در اولویت است) |
| `browser.headless`          | اجرا بدون رابط گرافیکی                                                      | `false`                                                            |
| `OPENCLAW_BROWSER_HEADLESS` | بازنویسی به‌ازای هر فرایند برای حالت بدون رابط گرافیکی مرورگر مدیریت‌شده محلی         | تنظیم‌نشده                                                              |
| `browser.noSandbox`         | افزودن پرچم `--no-sandbox` (برای برخی تنظیمات Linux ضروری است)               | `false`                                                            |
| `browser.attachOnly`        | مرورگر را راه‌اندازی نکنید؛ فقط به مرورگر موجود متصل شوید              | `false`                                                            |

در Raspberry Pi، میزبان‌های VPS قدیمی‌تر یا فضای ذخیره‌سازی کند، وقتی Chrome برای در دسترس قرار دادن نقطه پایانی HTTP مربوط به CDP
یا آماده‌شدن به زمانی بیش از مهلت مرورگر مدیریت‌شده نیاز دارد، از مرورگری که به‌صورت دستی راه‌اندازی شده
و `attachOnly` استفاده کنید.

### مشکل: هیچ زبانه Chrome برای profile="user" یافت نشد

از پروفایل `user` ‏(`existing-session` / Chrome MCP) استفاده می‌کنید و هیچ
زبانه‌ای برای اتصال باز نیست.

گزینه‌های رفع مشکل:

1. به‌جای آن از مرورگر مدیریت‌شده استفاده کنید:
   `openclaw browser --browser-profile openclaw start` (یا
   `browser.defaultProfile: "openclaw"` را تنظیم کنید).
2. Chrome محلی را با دست‌کم یک زبانه باز در حال اجرا نگه دارید، سپس با
   `--browser-profile user` دوباره تلاش کنید.

نکته‌ها:

- `user` فقط مختص میزبان است. در سرورهای Linux، کانتینرها یا میزبان‌های راه دور،
  پروفایل‌های CDP را ترجیح دهید.
- `user` و دیگر پروفایل‌های `existing-session` محدودیت‌های فعلی Chrome MCP
  را به‌اشتراک می‌گذارند: فقط کنش‌های مبتنی بر ارجاع، یک فایل در هر بارگذاری، بدون بازنویسی `timeoutMs`
  برای پنجره‌های محاوره‌ای، بدون `wait --load networkidle` و بدون `responsebody`، خروجی PDF،
  رهگیری دانلود یا کنش‌های دسته‌ای.
- پروفایل‌های درایور محلی `openclaw`، مقادیر `cdpPort`/`cdpUrl` را به‌طور خودکار اختصاص می‌دهند؛ این موارد را فقط
  برای CDP راه دور به‌صورت دستی تنظیم کنید.
- پروفایل‌های CDP راه دور، `http://`، `https://`، `ws://` و `wss://` را می‌پذیرند.
  برای کشف `/json/version` از HTTP(S) استفاده کنید، یا وقتی سرویس مرورگر
  نشانی مستقیم سوکت DevTools را ارائه می‌دهد، از WS(S) استفاده کنید.

## مرتبط

- [مرورگر](/fa/tools/browser)
- [ورود به مرورگر](/fa/tools/browser-login)
- [عیب‌یابی WSL2 مرورگر](/fa/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
