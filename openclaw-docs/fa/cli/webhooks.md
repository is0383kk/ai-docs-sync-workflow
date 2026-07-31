---
read_when:
    - می‌خواهید رویدادهای Pub/Sub جیمیل را به OpenClaw متصل کنید
    - به فهرست کامل پرچم‌ها و مقادیر پیش‌فرض نیاز دارید
summary: مرجع CLI برای `openclaw webhooks` (راه‌اندازی و اجراکننده Gmail Pub/Sub)
title: Webhookها
x-i18n:
    generated_at: "2026-07-27T16:22:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83fff0ac2ce247402f45523eda0b5cdd551bd65212636118698e45cb8740236c
    source_path: cli/webhooks.md
    workflow: 16
---

# `openclaw webhooks`

کمک‌ابزارها و یکپارچه‌سازی‌های Webhook. در حال حاضر، دامنه این سطح به جریان‌های Gmail Pub/Sub مبتنی بر ناظر همراه `gog` محدود است.

## زیرفرمان‌ها

```bash
openclaw webhooks gmail setup --account <email> [...]
openclaw webhooks gmail run   [--account <email>] [...]
```

| زیرفرمان    | توضیحات                                                                           |
| ------------- | ------------------------------------------------------------------------------------- |
| `gmail setup` | راه‌انداز یک‌باره: نظارت Gmail، موضوع/اشتراک Pub/Sub و تحویل هوک OpenClaw. |
| `gmail run`   | اجرای `gog watch serve` به‌همراه حلقه تمدید خودکار نظارت در پیش‌زمینه.               |

<Note>
پس از تنظیم `hooks.enabled=true` و `hooks.gmail.account` (توسط `gmail setup`) Gateway نیز هنگام راه‌اندازی، `gog gmail watch serve` را به‌طور خودکار اجرا می‌کند. `gmail run` همان منطق را در پیش‌زمینه اجرا می‌کند و برای اشکال‌زدایی یا زمانی که ناظر Gateway غیرفعال است مفید است. برای جزئیات شروع خودکار و انصراف با `OPENCLAW_SKIP_GMAIL_WATCHER`، به [یکپارچه‌سازی Gmail Pub/Sub](/fa/automation/cron-jobs#gmail-pubsub-integration) مراجعه کنید.
</Note>

## `webhooks gmail setup`

```bash
openclaw webhooks gmail setup --account you@example.com
openclaw webhooks gmail setup --account you@example.com --project my-gcp-project --json
openclaw webhooks gmail setup --account you@example.com --hook-url https://gateway.example.com/hooks/gmail
```

در صورت فقدان، `gcloud` و `gog` را نصب می‌کند، `gcloud` را احراز هویت می‌کند، موضوع و اشتراک Pub/Sub را می‌سازد، نظارت Gmail را آغاز می‌کند و پیکربندی `hooks.gmail` را با `hooks.enabled=true` می‌نویسد. `Next: openclaw webhooks gmail run` را چاپ می‌کند.

### الزامی

| پرچم                | توضیحات             |
| ------------------- | ----------------------- |
| `--account <email>` | حساب Gmail برای نظارت. |

### گزینه‌های Pub/Sub

| پرچم                    | پیش‌فرض                | توضیحات                                                                                                                             |
| ----------------------- | ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `--project <id>`        | (هیچ‌کدام)                 | شناسه پروژه GCP (مالک کلاینت OAuth). ابتدا از شناسه پروژه خود موضوع و سپس از پروژه تعیین‌شده با اعتبارنامه‌های `gog` استفاده می‌کند. |
| `--topic <name>`        | `gog-gmail-watch`      | نام موضوع Pub/Sub.                                                                                                                     |
| `--subscription <name>` | `gog-gmail-watch-push` | نام اشتراک Pub/Sub.                                                                                                              |
| `--label <label>`       | `INBOX`                | برچسب Gmail برای نظارت.                                                                                                                   |
| `--push-endpoint <url>` | (هیچ‌کدام)                 | نقطه پایانی صریح push برای Pub/Sub. بر Tailscale اولویت دارد.                                                                                    |

### گزینه‌های تحویل OpenClaw

| پرچم                   | پیش‌فرض                                      | توضیحات                                |
| ---------------------- | -------------------------------------------- | ------------------------------------------ |
| `--hook-url <url>`     | ساخته‌شده از `hooks.path` و درگاه Gateway | نشانی Webhook ‏OpenClaw.                      |
| `--hook-token <token>` | `hooks.token` یا یک توکن تولیدشده          | توکن Webhook ‏OpenClaw.                    |
| `--push-token <token>` | توکن تولیدشده                              | توکن push که به `gog watch serve` ارسال می‌شود. |

### گزینه‌های `gog watch serve`

| پرچم                  | پیش‌فرض         | توضیحات                                                                                                                                  |
| --------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `--bind <host>`       | `127.0.0.1`     | میزبان اتصال `gog watch serve`.                                                                                                                 |
| `--port <port>`       | `8788`          | درگاه `gog watch serve`.                                                                                                                      |
| `--path <path>`       | `/gmail-pubsub` | مسیر `gog watch serve`. وقتی Tailscale بدون مقصد صریح فعال باشد، به‌اجبار `/` می‌شود، زیرا Tailscale پیش از پروکسی‌کردن مسیر را حذف می‌کند. |
| `--include-body`      | `true`          | قطعه‌هایی از بدنه ایمیل را شامل می‌شود. هیچ پرچم CLI برای غیرفعال‌کردن آن وجود ندارد؛ به‌جای آن `hooks.gmail.includeBody: false` را در پیکربندی تنظیم کنید.                  |
| `--max-bytes <n>`     | `20000`         | حداکثر تعداد بایت برای هر قطعه بدنه.                                                                                                                  |
| `--renew-minutes <n>` | `720` (12h)     | نظارت Gmail را هر N دقیقه تمدید می‌کند.                                                                                                           |

### در معرض دسترس قراردادن با Tailscale

| پرچم                      | پیش‌فرض  | توضیحات                                                      |
| ------------------------- | -------- | ---------------------------------------------------------------- |
| `--tailscale <mode>`      | `funnel` | نقطه پایانی push را از طریق tailscale در معرض دسترس قرار می‌دهد: `funnel`، `serve` یا `off`. |
| `--tailscale-path <path>` | (هیچ‌کدام)   | مسیر برای tailscale serve/funnel.                                 |
| `--tailscale-target <t>`  | (هیچ‌کدام)   | مقصد serve/funnel در Tailscale (درگاه، `host:port` یا URL).       |

### خروجی

| پرچم     | توضیحات                                       |
| -------- | ------------------------------------------------- |
| `--json` | به‌جای متن، خلاصه‌ای ماشین‌خوان چاپ می‌کند. |

## `webhooks gmail run`

```bash
openclaw webhooks gmail run --account you@example.com
```

`gog watch serve` را به‌همراه حلقه تمدید خودکار نظارت در پیش‌زمینه اجرا می‌کند و اگر `gog watch serve` به‌طور غیرمنتظره خارج شود، آن را پس از تأخیر 2s دوباره راه‌اندازی می‌کند.

`run` همان پرچم‌های Pub/Sub، تحویل OpenClaw، `gog watch serve` و Tailscale در `setup` را می‌پذیرد، به‌جز موارد زیر:

- `--account` در `run` **اختیاری** است؛ مقدار جایگزین آن `hooks.gmail.account` است.
- `run`، مقادیر `--project`، `--push-endpoint` یا `--json` را نمی‌پذیرد.
- هر پرچم ابتدا از مقدار پیکربندی متناظر `hooks.gmail.*` (نوشته‌شده توسط `setup`) و سپس از همان پیش‌فرض داخلی مورد استفاده `setup` استفاده می‌کند، با یک استثنا: اگر نه پرچم و نه `hooks.gmail.tailscale.mode` تنظیم شده باشد، مقدار پیش‌فرض `--tailscale` در `run` برابر `off` است (نه `funnel`).

| دسته          | پرچم‌ها                                                                            |
| ----------------- | -------------------------------------------------------------------------------- |
| Pub/Sub           | `--account`، `--topic`، `--subscription`، `--label`                              |
| تحویل OpenClaw | `--hook-url`، `--hook-token`، `--push-token`                                     |
| `gog watch serve` | `--bind`، `--port`، `--path`، `--include-body`، `--max-bytes`، `--renew-minutes` |
| Tailscale         | `--tailscale`، `--tailscale-path`، `--tailscale-target`                          |

<Note>
برای `run`، مقدار `--topic` مسیر کامل موضوع Pub/Sub است (`projects/.../topics/...`)، نه فقط نام کوتاه موضوع.
</Note>

## مرتبط

- [مرجع CLI](/fa/cli)
- [خودکارسازی Webhook](/fa/automation/cron-jobs)
- [یکپارچه‌سازی Gmail Pub/Sub](/fa/automation/cron-jobs#gmail-pubsub-integration)
