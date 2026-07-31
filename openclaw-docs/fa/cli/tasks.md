---
read_when:
    - می‌خواهید رکوردهای وظایف پس‌زمینه را بررسی، ممیزی یا لغو کنید
    - شما در حال مستندسازی فرمان‌های Task Flow در `openclaw tasks flow` هستید
summary: مرجع CLI برای `openclaw tasks` (دفتر ثبت وظایف پس‌زمینه و وضعیت جریان وظیفه)
title: '`openclaw tasks`'
x-i18n:
    generated_at: "2026-07-27T16:25:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b03a4aa9fab12b6e5773259a76a1e89fd6e6398c73e5b0533a31e5e3a3894f9c
    source_path: cli/tasks.md
    workflow: 16
---

وظایف پس‌زمینهٔ پایدار و وضعیت Task Flow را بررسی کنید. در صورت نبود زیرفرمان،
`openclaw tasks` معادل `openclaw tasks list` است.

برای آشنایی با چرخهٔ حیات و مدل تحویل، به [وظایف پس‌زمینه](/fa/automation/tasks)
و برای توضیحات کامل یافته‌ها به بخش `tasks audit` آن مراجعه کنید.

## نحوهٔ استفاده

```bash
openclaw tasks
openclaw tasks list
openclaw tasks list --runtime acp
openclaw tasks list --status running
openclaw tasks show <lookup>
openclaw tasks notify <lookup> state_changes
openclaw tasks cancel <lookup>
openclaw tasks audit
openclaw tasks maintenance
openclaw tasks maintenance --apply
openclaw tasks flow list
openclaw tasks flow show <lookup>
openclaw tasks flow cancel <lookup>
```

## گزینه‌های ریشه

| پرچم               | توضیحات                                                                                        |
| ------------------ | -------------------------------------------------------------------------------------------------- |
| `--json`           | خروجی JSON.                                                                                       |
| `--runtime <name>` | فیلتر بر اساس نوع: `subagent`، `acp`، `cron` یا `cli`.                                               |
| `--status <name>`  | فیلتر بر اساس وضعیت: `queued`، `running`، `succeeded`، `failed`، `timed_out`، `cancelled` یا `lost`. |

## زیرفرمان‌ها

### `list`

```bash
openclaw tasks list [--runtime <name>] [--status <name>] [--json]
```

وظایف پس‌زمینهٔ رهگیری‌شده را از جدیدترین به قدیمی‌ترین فهرست می‌کند.

### `show`

```bash
openclaw tasks show <lookup> [--json]
```

یک وظیفه را بر اساس شناسهٔ وظیفه، شناسهٔ اجرا یا کلید نشست نمایش می‌دهد.

### `notify`

```bash
openclaw tasks notify <lookup> <done_only|state_changes|silent>
```

خط‌مشی اعلان را برای یک وظیفهٔ در حال اجرا تغییر می‌دهد.

### `cancel`

```bash
openclaw tasks cancel <lookup>
```

یک وظیفهٔ پس‌زمینهٔ در حال اجرا را لغو می‌کند.

### `audit`

```bash
openclaw tasks audit [--severity <warn|error>] [--code <name>] [--limit <n>] [--json]
```

رکوردهای کهنه، مفقود، ناموفق در تحویل یا به‌شکلی دیگر ناسازگارِ وظیفه و
Task Flow را آشکار می‌کند. وظایف مفقودی که تا `cleanupAfter` نگه داشته شده‌اند، هشدار هستند؛
وظایف مفقود منقضی‌شده یا بدون مُهر، خطا هستند.

`--code` کدهای وظیفه (`stale_queued`، `stale_running`، `lost`،
`delivery_failed`، `missing_cleanup`، `inconsistent_timestamps`) و کدهای Task
Flow (`restore_failed`، `stale_waiting`، `stale_blocked`،
`cancel_stuck`، `missing_linked_tasks`، `blocked_task_missing`) را می‌پذیرد. برای
جزئیات شدت و محرک هر کد، به [وظایف پس‌زمینه](/fa/automation/tasks)
مراجعه کنید.

### `maintenance`

```bash
openclaw tasks maintenance [--apply] [--json]
```

پیش‌نمایشی از تطبیق وظیفه و Task Flow، مُهرگذاری پاک‌سازی، هرس و پاک‌سازی
رجیستری نشست اجرای cron کهنه ارائه می‌دهد یا آن‌ها را اعمال می‌کند.

برای وظایف cron، تطبیق پیش از علامت‌گذاری یک وظیفهٔ فعال قدیمی به‌عنوان
`lost`، از گزارش‌های اجرای ماندگار/وضعیت کار استفاده می‌کند؛ بنابراین اجراهای تکمیل‌شدهٔ cron
صرفاً به‌دلیل از بین رفتن وضعیت زمان اجرای درون‌حافظه‌ای Gateway به خطاهای کاذب
ممیزی تبدیل نمی‌شوند. ممیزی آفلاین CLI برای مجموعهٔ محلیِ فرایندِ کارهای فعال cron
در Gateway مرجع قطعی نیست. وظایف CLI دارای شناسهٔ اجرا/شناسهٔ منبع، هنگامی که
بافت اجرای زندهٔ Gateway آن‌ها از بین رفته باشد، با `lost` علامت‌گذاری می‌شوند؛ حتی اگر
یک ردیف قدیمی نشست فرزند همچنان باقی مانده باشد.

هنگام اعمال، نگه‌داری همچنین ردیف‌های رجیستری نشست `cron:<jobId>:run:<uuid>`
قدیمی‌تر از 7 روز را هرس می‌کند، درحالی‌که کارهای cron در حال اجرا را حفظ کرده
و ردیف‌های نشست غیر cron را دست‌نخورده باقی می‌گذارد.

### `flow`

```bash
openclaw tasks flow list [--status <name>] [--json]
openclaw tasks flow show <lookup> [--json]
openclaw tasks flow cancel <lookup>
```

وضعیت پایدار Task Flow را در دفتر وظایف بررسی یا لغو می‌کند.
`flow list --status` مقادیر `queued`، `running`، `waiting`، `blocked`،
`succeeded`، `failed`، `cancelled` یا `lost` را می‌پذیرد.

## مرتبط

- [مرجع CLI](/fa/cli)
- [وظایف پس‌زمینه](/fa/automation/tasks)
