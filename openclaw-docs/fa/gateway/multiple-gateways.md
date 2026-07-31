---
read_when:
    - اجرای بیش از یک Gateway روی یک دستگاه
    - برای هر Gateway به پیکربندی/وضعیت/درگاه‌های مجزا نیاز دارید
summary: اجرای چندین Gateway ‏OpenClaw روی یک میزبان (جداسازی، پورت‌ها و پروفایل‌ها)
title: چندین Gateway
x-i18n:
    generated_at: "2026-07-27T15:15:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 655fa865a98064d7c017a7c2eb08ea9a9683002d96a3dbe45a8c16cbd3c86ba1
    source_path: gateway/multiple-gateways.md
    workflow: 16
---

بیشتر راه‌اندازی‌ها به یک Gateway نیاز دارند؛ یک Gateway چندین اتصال پیام‌رسان و عامل را مدیریت می‌کند. تنها زمانی Gatewayهای جداگانه با پروفایل‌ها/پورت‌های ایزوله اجرا کنید که به ایزوله‌سازی قوی‌تر یا افزونگی نیاز دارید (برای مثال، یک ربات نجات).

## شروع سریع ربات نجات

ساده‌ترین راه‌اندازی ربات نجات:

- ربات اصلی را روی پروفایل پیش‌فرض نگه دارید.
- ربات نجات را روی `--profile rescue` و با توکن ربات Telegram مخصوص خودش اجرا کنید.
- ربات نجات را روی یک پورت پایه متفاوت، برای مثال `19789`، قرار دهید.

با این کار، اگر ربات اصلی از کار افتاده باشد، ربات نجات همچنان می‌تواند اشکال‌زدایی کند یا تغییرات پیکربندی را اعمال کند. بین پورت‌های پایه حداقل 20 پورت فاصله بگذارید تا پورت‌های مشتق‌شده مرورگر/CDP هرگز با هم تداخل نکنند.

```bash
# ربات نجات (ربات Telegram جداگانه، پروفایل جداگانه، پورت 19789)
openclaw --profile rescue onboard
openclaw --profile rescue gateway install --port 19789
```

اگر ربات اصلی شما از قبل در حال اجرا است، معمولاً همین تمام چیزی است که نیاز دارید. اگر فرایند راه‌اندازی اولیه از قبل سرویس نجات را نصب کرده است، `gateway install` نهایی را نادیده بگیرید.

در طول `openclaw --profile rescue onboard`:

- از یک توکن ربات Telegram جداگانه و اختصاص‌یافته به حساب نجات استفاده کنید (به‌سادگی می‌توان دسترسی به آن را فقط به اپراتورها محدود کرد، از نصب کانال/برنامه ربات اصلی مستقل است و یک مسیر بازیابی ساده مبتنی بر پیام مستقیم فراهم می‌کند).
- نام پروفایل `rescue` را حفظ کنید.
- از یک پورت پایه استفاده کنید که حداقل 20 واحد از پورت ربات اصلی بیشتر باشد.
- فضای کاری پیش‌فرض نجات را بپذیرید، مگر اینکه از قبل خودتان یکی را مدیریت می‌کنید.

### تغییراتی که `--profile rescue onboard` ایجاد می‌کند

`--profile rescue onboard` جریان عادی راه‌اندازی اولیه را اجرا می‌کند، اما همه‌چیز را در یک پروفایل جداگانه می‌نویسد؛ بنابراین ربات نجات موارد زیر را مخصوص خود خواهد داشت:

- فایل پروفایل/پیکربندی
- دایرکتوری وضعیت
- فضای کاری (پیش‌فرض: `~/.openclaw/workspace-rescue`)
- نام سرویس مدیریت‌شده
- پورت پایه (به‌علاوه پورت‌های مشتق‌شده)
- توکن ربات Telegram

سایر اعلان‌ها با راه‌اندازی اولیه عادی یکسان هستند.

## راه‌اندازی عمومی چند Gateway

همین الگوی ایزوله‌سازی برای هر جفت یا گروهی از Gatewayها روی یک میزبان کار می‌کند؛ به هر Gateway اضافی، پروفایل نام‌گذاری‌شده و پورت پایه مخصوص خودش را اختصاص دهید:

```bash
# اصلی (پروفایل پیش‌فرض)
openclaw setup
openclaw gateway --port 18789

# Gateway اضافی
openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

پروفایل‌های نام‌گذاری‌شده در هر دو طرف نیز کار می‌کنند:

```bash
openclaw --profile main setup
openclaw --profile main gateway --port 18789

openclaw --profile ops setup
openclaw --profile ops gateway --port 19789
```

سرویس‌ها نیز از همین الگو پیروی می‌کنند:

```bash
openclaw gateway install
openclaw --profile ops gateway install --port 19789
```

برای یک مسیر اپراتوری پشتیبان از شروع سریع ربات نجات استفاده کنید؛ برای چند Gateway با عمر طولانی در کانال‌ها، مستأجرها، فضاهای کاری یا نقش‌های عملیاتی مختلف، از الگوی عمومی پروفایل استفاده کنید.

## چک‌لیست ایزوله‌سازی

این موارد را برای هر نمونه Gateway منحصربه‌فرد نگه دارید:

| تنظیم                      | هدف                              |
| ---------------------------- | ------------------------------------ |
| `OPENCLAW_CONFIG_PATH`       | فایل پیکربندی مخصوص هر نمونه             |
| `OPENCLAW_STATE_DIR`         | نشست‌ها، اطلاعات اعتبارسنجی و حافظه‌های نهان مخصوص هر نمونه |
| `agents.defaults.workspace`  | ریشه فضای کاری مخصوص هر نمونه          |
| `gateway.port` (یا `--port`) | منحصربه‌فرد برای هر نمونه                  |
| پورت‌های مشتق‌شده مرورگر/CDP    | بخش زیر را ببینید                            |

اشتراک‌گذاری هر یک از این موارد باعث تداخل پیکربندی، وضعیت یا پورت می‌شود. راه‌اندازی Gateway
مالکیت منحصربه‌فرد دایرکتوری وضعیت را حتی زمانی که
`OPENCLAW_ALLOW_MULTI_GATEWAY=1` تک‌نمونه‌ای‌بودن هر پیکربندی را نادیده می‌گیرد، اعمال می‌کند.

## نگاشت پورت (مشتق‌شده)

پورت پایه = `gateway.port` (یا `OPENCLAW_GATEWAY_PORT` / `--port`).

- پورت سرویس کنترل مرورگر = پورت پایه + 2 (فقط loopback).
- میزبان Canvas روی خود سرور HTTP مربوط به Gateway ارائه می‌شود (همان پورت `gateway.port`).
- پورت‌های CDP پروفایل مرورگر به‌طور خودکار از `browser control port + 9` تا `+ 108` تخصیص می‌یابند.

اگر هر یک از این موارد را در پیکربندی یا متغیر محیطی بازنویسی می‌کنید، باید آن‌ها را برای هر نمونه منحصربه‌فرد نگه دارید.

## نکات مرورگر/CDP (اشتباه رایج)

- `browser.cdpUrl` را در چند نمونه روی یک مقدار یکسان **ثابت نکنید**.
- هر نمونه به پورت کنترل مرورگر و بازه CDP مخصوص خودش نیاز دارد (که از پورت Gateway آن مشتق می‌شوند).
- برای پورت‌های صریح CDP، مقدار `browser.profiles.<name>.cdpPort` را برای هر نمونه تنظیم کنید.
- برای Chrome راه‌دور، از `browser.profiles.<name>.cdpUrl` استفاده کنید (برای هر پروفایل و هر نمونه).

## نمونه دستی متغیرهای محیطی

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/main.json \
OPENCLAW_STATE_DIR=~/.openclaw \
openclaw gateway --port 18789

OPENCLAW_CONFIG_PATH=~/.openclaw/rescue.json \
OPENCLAW_STATE_DIR=~/.openclaw-rescue \
openclaw gateway --port 19789
```

## بررسی‌های سریع

```bash
openclaw gateway status --deep
openclaw --profile rescue gateway status --deep
openclaw --profile rescue gateway probe
openclaw status
openclaw --profile rescue status
openclaw --profile rescue browser status
```

- `gateway status --deep` سرویس‌های قدیمی launchd/systemd/schtasks باقی‌مانده از نصب‌های پیشین را شناسایی می‌کند.
- متن هشدار `gateway probe`، مانند `multiple reachable gateway identities detected`، فقط زمانی مورد انتظار است که عمداً بیش از یک Gateway ایزوله را اجرا می‌کنید، یا زمانی که OpenClaw نمی‌تواند اثبات کند اهداف قابل‌دسترسیِ پروب همان Gateway هستند. یک تونل SSH، نشانی پروکسی یا نشانی راه‌دور پیکربندی‌شده به همان Gateway، حتی اگر پورت‌های انتقال متفاوت باشند، یک Gateway با چند انتقال محسوب می‌شود.

## مرتبط

- [راهنمای عملیاتی Gateway](/fa/gateway)
- [قفل Gateway](/fa/gateway/gateway-lock)
- [پیکربندی](/fa/gateway/configuration)
