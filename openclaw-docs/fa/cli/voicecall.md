---
read_when:
    - از Plugin تماس صوتی استفاده می‌کنید و همهٔ نقاط ورود CLI را می‌خواهید
    - به جدول‌های پرچم و مقادیر پیش‌فرض برای setup، smoke، call، continue، speak، dtmf، end، status، tail، latency، expose و start نیاز دارید
summary: مرجع CLI برای `openclaw voicecall` (سطح فرمان Plugin تماس صوتی)
title: تماس صوتی
x-i18n:
    generated_at: "2026-07-27T15:20:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aec445886cccb79c9212dd9f1f448ff9634274deb380632be786478c9bb29670
    source_path: cli/voicecall.md
    workflow: 16
---

# `openclaw voicecall`

`voicecall` فرمانی است که یک Plugin ارائه می‌کند. این فرمان فقط زمانی نمایش داده می‌شود که Plugin تماس صوتی نصب و فعال باشد.

وقتی Gateway در حال اجرا است، فرمان‌های عملیاتی (`call`، `start`، `continue`، `speak`، `dtmf`، `end`، `status`) به زمان اجرای تماس صوتی همان Gateway هدایت می‌شوند. اگر هیچ Gateway در دسترس نباشد، از زمان اجرای مستقل CLI به‌عنوان جایگزین استفاده می‌کنند.

## زیرفرمان‌ها

```bash
openclaw voicecall setup    [--json]
openclaw voicecall smoke    [-t <phone>] [--message <text>] [--mode <m>] [--yes] [--json]
openclaw voicecall call     -m <text> [-t <phone>] [--mode <m>]
openclaw voicecall start    --to <phone> [--message <text>] [--mode <m>]
openclaw voicecall continue --call-id <id> --message <text>
openclaw voicecall speak    --call-id <id> --message <text>
openclaw voicecall dtmf     --call-id <id> --digits <digits>
openclaw voicecall end      --call-id <id>
openclaw voicecall status   [--call-id <id>] [--json]
openclaw voicecall tail     [--file <path>] [--since <n>] [--poll <ms>]
openclaw voicecall latency  [--file <path>] [--last <n>]
openclaw voicecall expose   [--mode <m>] [--path <p>] [--port <port>] [--serve-path <p>]
```

| زیرفرمان | توضیحات                                                     |
| ---------- | --------------------------------------------------------------- |
| `setup`    | بررسی‌های آمادگی ارائه‌دهنده و Webhook را نمایش می‌دهد.                     |
| `smoke`    | بررسی‌های آمادگی را اجرا می‌کند؛ تماس آزمایشی واقعی را فقط با `--yes` برقرار می‌کند. |
| `call`     | یک تماس صوتی خروجی را آغاز می‌کند.                                |
| `start`    | نام مستعاری برای `call` است که در آن `--to` الزامی و `--message` اختیاری است. |
| `continue` | پیامی را پخش می‌کند و منتظر پاسخ بعدی می‌ماند.                 |
| `speak`    | پیامی را بدون انتظار برای پاسخ پخش می‌کند.                 |
| `dtmf`     | ارقام DTMF را به یک تماس فعال ارسال می‌کند.                             |
| `end`      | یک تماس فعال را قطع می‌کند.                                         |
| `status`   | تماس‌های فعال را بررسی می‌کند (یا یک تماس را با `--call-id`).                   |
| `tail`     | انتهای `calls.jsonl` را به‌صورت زنده دنبال می‌کند (برای آزمون‌های ارائه‌دهنده مفید است).              |
| `latency`  | معیارهای تأخیر نوبت را از `calls.jsonl` خلاصه می‌کند.              |
| `expose`   | قابلیت serve/funnel در Tailscale را برای نقطه پایانی Webhook فعال یا غیرفعال می‌کند.         |

## راه‌اندازی و آزمون دود

### `setup`

به‌طور پیش‌فرض، بررسی‌های آمادگی را در قالبی خوانا برای انسان چاپ می‌کند. برای اسکریپت‌ها `--json` را ارسال کنید.

```bash
openclaw voicecall setup
openclaw voicecall setup --json
```

### `smoke`

همان بررسی‌های آمادگی را اجرا می‌کند. فقط زمانی یک تماس تلفنی واقعی برقرار می‌کند که هر دو `--to` و `--yes` موجود باشند.

| پرچم               | پیش‌فرض                           | توضیحات                             |
| ------------------ | --------------------------------- | --------------------------------------- |
| `-t, --to <phone>` | (هیچ‌کدام)                            | شماره تلفن برای برقراری آزمون دود واقعی.  |
| `--message <text>` | `OpenClaw voice call smoke test.` | پیامی که هنگام تماس آزمون دود پخش می‌شود. |
| `--mode <mode>`    | `notify`                          | حالت تماس: `notify` یا `conversation`.  |
| `--yes`            | `false`                           | تماس خروجی واقعی را عملاً برقرار می‌کند.  |
| `--json`           | `false`                           | JSON قابل‌خواندن توسط ماشین را چاپ می‌کند.            |

```bash
openclaw voicecall smoke
openclaw voicecall smoke --to "+15555550123"        # اجرای آزمایشی
openclaw voicecall smoke --to "+15555550123" --yes  # تماس اعلان واقعی
```

<Note>
برای ارائه‌دهندگان خارجی (`plivo`، `telnyx`، `twilio`)، `setup` و `smoke` به یک URL عمومی Webhook از `publicUrl`، یک تونل یا دسترسی Tailscale نیاز دارند. جایگزین loopback یا serve خصوصی رد می‌شود، زیرا اپراتورها نمی‌توانند به آن دسترسی پیدا کنند.
</Note>

## چرخه عمر تماس

### `call`

یک تماس صوتی خروجی را آغاز می‌کند.

| پرچم                   | الزامی | پیش‌فرض           | توضیحات                                                                |
| ---------------------- | -------- | ----------------- | -------------------------------------------------------------------------- |
| `-m, --message <text>` | بله      | (هیچ‌کدام)            | پیامی که هنگام برقراری تماس پخش می‌شود.                                   |
| `-t, --to <phone>`     | خیر       | پیکربندی `toNumber` | شماره تلفن E.164 برای تماس.                                                |
| `--mode <mode>`        | خیر       | `conversation`    | حالت تماس: `notify` (قطع تماس پس از پیام) یا `conversation` (باز نگه‌داشتن تماس). |

```bash
openclaw voicecall call --to "+15555550123" --message "Hello"
openclaw voicecall call -m "Heads up" --mode notify
```

### `start`

نام مستعاری برای `call` با شکل متفاوتی از پرچم‌های پیش‌فرض است.

| پرچم               | الزامی | پیش‌فرض        | توضیحات                              |
| ------------------ | -------- | -------------- | ---------------------------------------- |
| `--to <phone>`     | بله      | (هیچ‌کدام)         | شماره تلفن برای تماس.                    |
| `--message <text>` | خیر       | (هیچ‌کدام)         | پیامی که هنگام برقراری تماس پخش می‌شود. |
| `--mode <mode>`    | خیر       | `conversation` | حالت تماس: `notify` یا `conversation`.   |

### `continue`

پیامی را پخش می‌کند و منتظر پاسخ می‌ماند.

| پرچم               | الزامی | توضیحات       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | بله      | شناسه تماس.          |
| `--message <text>` | بله      | پیام برای پخش. |

### `speak`

پیامی را بدون انتظار برای پاسخ پخش می‌کند.

| پرچم               | الزامی | توضیحات       |
| ------------------ | -------- | ----------------- |
| `--call-id <id>`   | بله      | شناسه تماس.          |
| `--message <text>` | بله      | پیام برای پخش. |

### `dtmf`

ارقام DTMF را به یک تماس فعال ارسال می‌کند.

| پرچم                | الزامی | توضیحات                                      |
| ------------------- | -------- | ------------------------------------------------ |
| `--call-id <id>`    | بله      | شناسه تماس.                                         |
| `--digits <digits>` | بله      | ارقام DTMF (برای مثال `ww123456#` برای انتظارها). |

### `end`

یک تماس فعال را قطع می‌کند.

| پرچم             | الزامی | توضیحات |
| ---------------- | -------- | ----------- |
| `--call-id <id>` | بله      | شناسه تماس.    |

### `status`

تماس‌های فعال را بررسی می‌کند.

| پرچم             | پیش‌فرض | توضیحات                  |
| ---------------- | ------- | ---------------------------- |
| `--call-id <id>` | (هیچ‌کدام)  | خروجی را به یک تماس محدود می‌کند. |
| `--json`         | `false` | JSON قابل‌خواندن توسط ماشین را چاپ می‌کند. |

```bash
openclaw voicecall status
openclaw voicecall status --json
openclaw voicecall status --call-id <id>
```

## گزارش‌ها و معیارها

### `tail`

انتهای گزارش JSONL تماس صوتی را به‌صورت زنده دنبال می‌کند. هنگام شروع، آخرین `--since` خط را چاپ می‌کند و سپس خطوط جدید را هم‌زمان با نوشته‌شدنشان پخش می‌کند.

| پرچم            | پیش‌فرض                    | توضیحات                    |
| --------------- | -------------------------- | ------------------------------ |
| `--file <path>` | از مخزن Plugin تعیین می‌شود | مسیر `calls.jsonl`.         |
| `--since <n>`   | `25`                       | تعداد خطوطی که پیش از دنبال‌کردن چاپ می‌شوند. |
| `--poll <ms>`   | `250` (حداقل 50)         | فاصله نمونه‌برداری برحسب میلی‌ثانیه. |

### `latency`

معیارهای تأخیر نوبت و انتظار برای شنیدن را از `calls.jsonl` خلاصه می‌کند. خروجی JSON شامل خلاصه‌های `recordsScanned`، `turnLatency` و `listenWait` است.

| پرچم            | پیش‌فرض                    | توضیحات                          |
| --------------- | -------------------------- | ------------------------------------ |
| `--file <path>` | از مخزن Plugin تعیین می‌شود | مسیر `calls.jsonl`.               |
| `--last <n>`    | `200` (حداقل 1)          | تعداد رکوردهای اخیر برای تحلیل. |

## در دسترس قرار دادن Webhookها

### `expose`

پیکربندی serve/funnel در Tailscale را برای Webhook صوتی فعال، غیرفعال یا تغییر می‌دهد.

| پرچم                  | پیش‌فرض                                   | توضیحات                                     |
| --------------------- | ----------------------------------------- | ----------------------------------------------- |
| `--mode <mode>`       | `funnel`                                  | `off`، `serve` (tailnet) یا `funnel` (عمومی). |
| `--path <path>`       | پیکربندی `tailscale.path` یا `--serve-path` | مسیر Tailscale برای در دسترس قرار دادن.                       |
| `--port <port>`       | پیکربندی `serve.port` یا `3334`             | پورت Webhook محلی.                             |
| `--serve-path <path>` | پیکربندی `serve.path` یا `/voice/webhook`   | مسیر Webhook محلی.                             |

```bash
openclaw voicecall expose --mode serve
openclaw voicecall expose --mode funnel
openclaw voicecall expose --mode off
```

<Warning>
نقطه پایانی Webhook را فقط در دسترس شبکه‌هایی قرار دهید که به آن‌ها اعتماد دارید. در صورت امکان، Tailscale Serve را به Funnel ترجیح دهید.
</Warning>

## مطالب مرتبط

- [مرجع CLI](/fa/cli)
- [Plugin تماس صوتی](/fa/plugins/voice-call)
