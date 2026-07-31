---
read_when: You are managing sandbox runtimes or debugging sandbox/tool-policy behavior.
status: active
summary: زمان‌اجراهای سندباکس را مدیریت و سیاست مؤثر سندباکس را بررسی کنید
title: CLI سندباکس
x-i18n:
    generated_at: "2026-07-27T16:20:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea8311de7702222295f3ba8753304e30f6ed21958e2843f62db5d064f06e24ae
    source_path: cli/sandbox.md
    workflow: 16
---

مدیریت محیط‌های اجرایی sandbox برای اجرای ایزوله عامل: کانتینرهای Docker، مقصدهای SSH یا بک‌اندهای OpenShell.

## فرمان‌ها

### `openclaw sandbox list`

فهرست‌کردن محیط‌های اجرایی sandbox همراه با وضعیت، بک‌اند، تطابق پیکربندی، عمر، مدت بی‌کاری و نشست/عامل مرتبط.

```bash
openclaw sandbox list
openclaw sandbox list --browser  # فقط کانتینرهای مرورگر
openclaw sandbox list --json
```

### `openclaw sandbox recreate`

حذف محیط‌های اجرایی sandbox برای اجبار به ایجاد مجدد با پیکربندی فعلی. محیط‌های اجرایی دفعه بعد که عامل استفاده شود، به‌طور خودکار دوباره ایجاد می‌شوند.

```bash
openclaw sandbox recreate --all
openclaw sandbox recreate --agent mybot        # شامل زیرنشست‌های agent:mybot:*
openclaw sandbox recreate --session "agent:main:main"
openclaw sandbox recreate --browser --all      # فقط کانتینرهای مرورگر
openclaw sandbox recreate --all --force        # ردکردن تأیید
```

گزینه‌ها:

- `--all`: ایجاد مجدد همه کانتینرهای sandbox
- `--session <key>`: ایجاد مجدد محیط اجرایی با همین کلید دامنه دقیق (همان‌طور که در `sandbox list` نشان داده شده است)؛ بدون بسط نام کوتاه
- `--agent <id>`: ایجاد مجدد محیط‌های اجرایی برای یک عامل (مطابق با `agent:<id>` و `agent:<id>:*`)
- `--browser`: تأثیرگذاری فقط بر کانتینرهای مرورگر
- `--force`: ردکردن اعلان تأیید

دقیقاً یکی از `--all`، `--session` یا `--agent` را ارسال کنید.

برای `ssh` و `remote` در OpenShell، ایجاد مجدد نسبت به Docker اهمیت بیشتری دارد: پس از مقداردهی اولیه، فضای کاری راه‌دور مرجع اصلی است، `recreate` آن فضای کاری راه‌دور مرجع را برای دامنه انتخاب‌شده حذف می‌کند و اجرای بعدی آن را از فضای کاری محلی فعلی دوباره مقداردهی می‌کند.

### `openclaw sandbox explain`

بررسی حالت/دامنه مؤثر sandbox، دسترسی فضای کاری، خط‌مشی ابزار sandbox و دروازه‌های ابزارهای دارای سطح دسترسی بالاتر (همراه با مسیر کلیدهای پیکربندی برای رفع مشکل).

گزارش، `workspaceRoot` را به‌عنوان ریشه پیکربندی‌شده sandbox نگه می‌دارد و فضای کاری مؤثر میزبان، دایرکتوری کاری محیط اجرایی بک‌اند و جدول اتصال‌های Docker را جداگانه نمایش می‌دهد. برای `workspaceAccess: "rw"`، فضای کاری مؤثر میزبان همان فضای کاری عامل است، نه دایرکتوری‌ای زیر `workspaceRoot`.

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

برخلاف `recreate --session`، این فرمان نام‌های کوتاه نشست را می‌پذیرد (برای مثال `main`) و آن‌ها را بر اساس عامل تعیین‌شده بسط می‌دهد.

## چرا ایجاد مجدد لازم است

به‌روزرسانی پیکربندی sandbox بر کانتینرهای در حال اجرا تأثیری ندارد: محیط‌های اجرایی موجود تنظیمات قدیمی خود را حفظ می‌کنند و محیط‌های اجرایی بی‌کار تنها پس از `prune.idleHours` (پیش‌فرض 24h) پاک‌سازی می‌شوند. عامل‌هایی که مرتباً استفاده می‌شوند، ممکن است محیط‌های اجرایی منسوخ را برای همیشه فعال نگه دارند. `openclaw sandbox recreate` محیط اجرایی قدیمی را حذف می‌کند تا استفاده بعدی آن را بر اساس پیکربندی فعلی دوباره بسازد.

<Tip>
به‌جای پاک‌سازی دستی و مختص هر بک‌اند، `openclaw sandbox recreate` را ترجیح دهید. این روش از رجیستری محیط اجرایی Gateway استفاده می‌کند و هنگام تغییر دامنه یا کلیدهای نشست، از عدم تطابق جلوگیری می‌کند.
</Tip>

## محرک‌های رایج

| تغییر                                                                                                                                                         | فرمان                                                             |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| به‌روزرسانی ایمیج Docker (`agents.defaults.sandbox.docker.image`)                                                                                                   | `openclaw sandbox recreate --all`                                   |
| پیکربندی sandbox (`agents.defaults.sandbox.*`)                                                                                                                   | `openclaw sandbox recreate --all`                                   |
| مقصد/احراز هویت SSH (`agents.defaults.sandbox.ssh.{target,workspaceRoot,identityFile,certificateFile,knownHostsFile,identityData,certificateData,knownHostsData}`) | `openclaw sandbox recreate --all`                                   |
| منبع/خط‌مشی/حالت OpenShell (`plugins.entries.openshell.config.{from,mode,policy}`)                                                                           | `openclaw sandbox recreate --all`                                   |
| `setupCommand`                                                                                                                                                 | `openclaw sandbox recreate --all` (یا `--agent <id>` برای یک عامل) |

<Note>
محیط‌های اجرایی دفعه بعد که عامل استفاده شود، به‌طور خودکار دوباره ایجاد می‌شوند.
</Note>

## مهاجرت رجیستری

فراداده محیط اجرایی sandbox در پایگاه داده مشترک وضعیت SQLite نگهداری می‌شود. نصب‌های قدیمی‌تر ممکن است فایل‌های رجیستری قدیمی‌ای داشته باشند که خواندن‌های معمول دیگر آن‌ها را بازنویسی نمی‌کنند:

- `~/.openclaw/sandbox/containers.json`
- `~/.openclaw/sandbox/browsers.json`
- یک قطعه JSON برای هر کانتینر/مرورگر در `~/.openclaw/sandbox/containers/` یا `~/.openclaw/sandbox/browsers/`

برای انتقال ورودی‌های قدیمی معتبر به SQLite، `openclaw doctor --fix` را اجرا کنید. فایل‌های قدیمی نامعتبر قرنطینه می‌شوند تا یک رجیستری قدیمی خراب نتواند ورودی‌های فعلی محیط اجرایی را پنهان کند.

## پیکربندی

تنظیمات sandbox در `~/.openclaw/openclaw.json` زیر `agents.defaults.sandbox` قرار دارند (بازنویسی‌های مختص هر عامل در `agents.entries.*.sandbox` قرار می‌گیرند):

```jsonc
{
  "agents": {
    "defaults": {
      "sandbox": {
        "mode": "all", // off، non-main، all
        "backend": "docker", // docker، ssh، openshell (ارائه‌شده توسط Plugin)
        "scope": "agent", // session، agent، shared
        "docker": {
          "image": "openclaw-sandbox:bookworm-slim",
          "containerPrefix": "openclaw-sbx-",
          // ... گزینه‌های بیشتر Docker
        },
        "prune": {
          "idleHours": 24, // پاک‌سازی خودکار پس از 24h بی‌کاری
          "maxAgeDays": 7, // پاک‌سازی خودکار پس از 7 روز
        },
      },
    },
  },
}
```

## مرتبط

- [مرجع CLI](/fa/cli)
- [Sandboxing](/fa/gateway/sandboxing)
- [فضای کاری عامل](/fa/concepts/agent-workspace)
- [Doctor](/fa/gateway/doctor): راه‌اندازی sandbox را بررسی می‌کند.
