---
read_when:
    - می‌خواهید شناسه‌های مخاطبان/گروه‌ها/خودتان را برای یک کانال پیدا کنید
    - در حال توسعه یک آداپتور فهرست کانال هستید
summary: مرجع CLI برای `openclaw directory` (خود، همتاها، گروه‌ها)
title: دایرکتوری
x-i18n:
    generated_at: "2026-07-27T15:18:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 33f1cabd0954f2e6e6affbfbff9f8e1f543bffebc54baff7c1ffaa21778744a0
    source_path: cli/directory.md
    workflow: 16
---

# `openclaw directory`

جست‌وجوهای دایرکتوری برای کانال‌هایی که از آن‌ها پشتیبانی می‌کنند: مخاطبان/همتایان، گروه‌ها و «من» (خود).

نتایج برای جای‌گذاری در فرمان‌های دیگر، به‌ویژه `openclaw message send --target ...`، در نظر گرفته شده‌اند.

## پرچم‌های مشترک

- `--channel <name>`: شناسه/نام مستعار کانال (هنگامی‌که چند کانال پیکربندی شده باشند الزامی است؛ اگر فقط یک کانال پیکربندی شده باشد، به‌طور خودکار انتخاب می‌شود)
- `--account <id>`: شناسه حساب (پیش‌فرض: مقدار پیش‌فرض کانال)
- `--json`: خروجی JSON

خروجی پیش‌فرض (غیر JSON) به‌صورت `id` (و گاهی `name`) است که با یک تب از هم جدا شده‌اند.

## نکات

- برای بسیاری از کانال‌ها، نتایج به‌جای دایرکتوری زنده ارائه‌دهنده، مبتنی بر پیکربندی هستند (فهرست‌های مجاز / گروه‌های پیکربندی‌شده).
- فهرست گروه‌های WhatsApp زنده است. جست‌وجوهای Gateway از اتصال تحت مالکیت آن دوباره استفاده می‌کنند؛ یک فرمان مستقل فقط زمانی نشست پیوندشده را باز می‌کند که هیچ فرایند دیگری مالک آن حساب نباشد و در غیر این صورت گزارش می‌دهد که گروه‌های زنده در دسترس نیستند.
- ممکن است یک Plugin کانال که از قبل نصب شده است، از دایرکتوری پشتیبانی نکند. در این حالت، فرمان عملیات پشتیبانی‌نشده را گزارش می‌دهد؛ برای افزودن پشتیبانی، تلاشی برای نصب مجدد یا ارتقای Plugin نمی‌کند.

## استفاده از نتایج با `message send`

```bash
openclaw directory peers list --channel slack --query "U0"
openclaw message send --channel slack --target user:U012ABCDEF --message "hello"
```

## قالب شناسه بر اساس کانال

| کانال                             | قالب شناسه مقصد                                                                                                            |
| ----------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| WhatsApp                            | `+15551234567` (پیام مستقیم)، `1234567890-1234567890@g.us` (گروه)، `120363123456789@newsletter` (کانال/خبرنامه، فقط خروجی) |
| Signal                              | نام‌های مستعار پیکربندی‌شده به مقصدهای پیام مستقیم E.164/UUID یا مقصدهای گروه `group:<id>` نگاشت می‌شوند                                           |
| Telegram                            | `@username` یا شناسه عددی گفت‌وگو؛ گروه‌ها از شناسه‌های عددی استفاده می‌کنند                                                                      |
| Slack                               | `user:U…` و `channel:C…`                                                                                                  |
| Discord                             | `user:<id>` و `channel:<id>`                                                                                              |
| Matrix (Plugin)                     | `user:@user:server`، `room:!roomId:server` یا `#alias:server`                                                              |
| Microsoft Teams (Plugin)            | `user:<id>` و `conversation:<id>`                                                                                         |
| Zalo (Plugin)                       | شناسه کاربر (Bot API)                                                                                                           |
| Zalo Personal / `zalouser` (Plugin) | شناسه رشته (پیام مستقیم/گروه)، از `zca` (`me`، `friend list`، `group list`)                                                        |

## خود («من»)

```bash
openclaw directory self --channel zalouser
```

## همتایان (مخاطبان/کاربران)

```bash
openclaw directory peers list --channel zalouser
openclaw directory peers list --channel zalouser --query "name"
openclaw directory peers list --channel zalouser --limit 50
```

## گروه‌ها

```bash
openclaw directory groups list --channel zalouser
openclaw directory groups list --channel zalouser --query "work"
openclaw directory groups members --channel zalouser --group-id <id>
```

## مرتبط

- [مرجع CLI](/fa/cli)
