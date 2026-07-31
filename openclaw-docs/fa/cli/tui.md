---
read_when:
    - یک رابط کاربری ترمینالی برای Gateway می‌خواهید (مناسب برای دسترسی از راه دور)
    - می‌خواهید url/token/session را از اسکریپت‌ها ارسال کنید
    - می‌خواهید TUI را در حالت تعبیه‌شدهٔ محلی و بدون Gateway اجرا کنید
    - می‌خواهید از openclaw chat یا openclaw tui --local استفاده کنید
summary: مرجع CLI برای `openclaw tui` (رابط کاربری ترمینال مبتنی بر Gateway یا تعبیه‌شده محلی)
title: TUI
x-i18n:
    generated_at: "2026-07-27T14:01:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5406f25bbd22c64867296c15112fafcaf8e1580c759e5fdc81fccfb62ae1e318
    source_path: cli/tui.md
    workflow: 16
---

# `openclaw tui`

رابط کاربری ترمینال متصل به Gateway را باز کنید، یا آن را در حالت تعبیه‌شدهٔ محلی
اجرا کنید.

راهنمای مرتبط: [TUI](/fa/web/tui)

## گزینه‌ها

| پرچم                         | پیش‌فرض                                   | توضیحات                                                                        |
| ---------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------------- |
| `--local`                    | `false`                                   | به‌جای Gateway، با زمان‌اجرای تعبیه‌شدهٔ محلی عامل اجرا شود.                 |
| `--url <url>`                | `gateway.remote.url` از پیکربندی          | نشانی WebSocket مربوط به Gateway.                                                             |
| `--token <token>`            | (هیچ‌کدام)                                    | توکن Gateway، در صورت نیاز.                                                         |
| `--password <pass>`          | (هیچ‌کدام)                                    | گذرواژهٔ Gateway، در صورت نیاز.                                                      |
| `--tls-fingerprint <sha256>` | `gateway.remote.tlsFingerprint`           | اثر انگشت مورد انتظار گواهی TLS برای یک Gateway سنجاق‌شدهٔ `wss://`.                |
| `--session <key>`            | `main` (یا `global` هنگامی که دامنه سراسری است) | کلید نشست. در فضای کاری عامل، مگر اینکه پیشوند داشته باشد، آن عامل را به‌طور خودکار انتخاب می‌کند. |
| `--deliver`                  | `false`                                   | پاسخ‌های دستیار از طریق کانال‌های پیکربندی‌شده تحویل داده شوند.                             |
| `--thinking <level>`         | (پیش‌فرض مدل)                           | بازنویسی سطح تفکر.                                                           |
| `--message <text>`           | (هیچ‌کدام)                                    | پس از اتصال، یک پیام اولیه ارسال شود.                                          |
| `--timeout-ms <ms>`          | `agents.defaults.timeoutSeconds`          | مهلت زمانی عامل. مقادیر نامعتبر یک هشدار ثبت می‌کنند و نادیده گرفته می‌شوند.                       |
| `--history-limit <n>`        | `200`                                     | تعداد ورودی‌های تاریخچه که هنگام پیوست‌شدن بارگیری می‌شوند.                                                 |

نام‌های مستعار `openclaw chat` و `openclaw terminal` این فرمان را با
`--local` ضمنی فراخوانی می‌کنند.

## نکات

- `--local` را نمی‌توان با `--url`، `--token`، `--password` یا `--tls-fingerprint` ترکیب کرد.
- `tui` در صورت امکان، SecretRefهای احراز هویت پیکربندی‌شدهٔ Gateway را برای احراز هویت با توکن/گذرواژه
  برطرف می‌کند (ارائه‌دهندگان `env`/`file`/`exec`).
- در نبود نشانی یا درگاه صریح، `tui` از درگاه فعال Gateway محلی
  که توسط Gateway در حال اجرا ثبت شده است پیروی می‌کند. `--url`، `OPENCLAW_GATEWAY_URL`،
  `OPENCLAW_GATEWAY_PORT` و پیکربندی Gateway راه‌دورِ صریح همچنان اولویت دارند.
- هنگامی که از داخل پوشهٔ فضای کاری یک عامل پیکربندی‌شده اجرا شود، TUI به‌طور خودکار
  آن عامل را برای مقدار پیش‌فرض کلید نشست انتخاب می‌کند (مگر اینکه `--session` به‌صراحت
  `agent:<id>:...` باشد).
- حالت محلی مستقیماً از زمان‌اجرای تعبیه‌شدهٔ عامل استفاده می‌کند. بیشتر ابزارهای محلی کار می‌کنند،
  اما قابلیت‌های مختص Gateway در دسترس نیستند.
- حالت محلی `/auth [provider]` را به سطح فرمان TUI اضافه می‌کند.
- دروازه‌های تأیید Plugin همچنان در حالت محلی اعمال می‌شوند: ابزارهایی که به تأیید نیاز دارند
  در ترمینال درخواست تصمیم می‌کنند و هیچ‌چیز بدون اطلاع به‌طور خودکار تأیید نمی‌شود.
- [اهداف](/fa/tools/goal) نشست در پاورقی نمایش داده می‌شوند و می‌توان آن‌ها را با
  `/goal` مدیریت کرد.

## مثال‌ها

```bash
openclaw chat
openclaw tui --local
openclaw tui
openclaw tui --url ws://127.0.0.1:18789 --token <token>
openclaw tui --session main --deliver
openclaw chat --message "پیکربندی من را با مستندات مقایسه کن و بگو چه چیزهایی را باید اصلاح کنم"
# هنگام اجرا در فضای کاری عامل، آن عامل را به‌طور خودکار تشخیص می‌دهد
openclaw tui --session bugfix
```

## چرخهٔ ترمیم پیکربندی

از حالت محلی استفاده کنید تا عامل تعبیه‌شده پیکربندی فعلی را بررسی کند، آن را
با مستندات مقایسه کند و از همان ترمینال به ترمیم آن کمک کند.

اگر `openclaw config validate` از قبل ناموفق است، ابتدا `openclaw configure` یا
`openclaw doctor --fix` را اجرا کنید؛ `openclaw chat` محافظ
پیکربندی نامعتبر را دور نمی‌زند.

```bash
openclaw chat
```

سپس درون TUI:

```text
!openclaw config file
!openclaw docs gateway auth token secretref
!openclaw config validate
!openclaw doctor
```

اصلاحات هدفمند را با `openclaw config set` یا `openclaw configure` اعمال کنید، سپس
`openclaw config validate` را دوباره اجرا کنید. به [TUI](/fa/web/tui) و
[پیکربندی](/fa/cli/config) مراجعه کنید.

## مرتبط

- [مرجع CLI](/fa/cli)
- [TUI](/fa/web/tui)
- [هدف](/fa/tools/goal)
