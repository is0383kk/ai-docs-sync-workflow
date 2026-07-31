---
read_when:
    - می‌خواهید تعهدات استنباط‌شده برای پیگیری را بررسی کنید
    - می‌خواهید اعلام حضورهای در انتظار را رد کنید
    - در حال ممیزی مواردی هستید که Heartbeat ممکن است تحویل دهد
summary: مرجع CLI برای `openclaw commitments` (بررسی و رد پیگیری‌های استنباط‌شده)
title: '`openclaw commitments`'
x-i18n:
    generated_at: "2026-07-27T14:58:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 4a7c573daad6a9bc6ce4532514c8cc22b3c510b4fc0cf9d1a79048413f08c1a2
    source_path: cli/commitments.md
    workflow: 16
---

رکوردهای باقی‌مانده از آزمایش بازنشسته‌شدهٔ تعهدات استنباطی را بررسی و حذف کنید.
OpenClaw دیگر تعهد جدیدی ایجاد یا تحویل نمی‌دهد، اما فرمان نگه‌داری را حفظ می‌کند
تا ارتقاها بتوانند ردیف‌های موجود SQLite را ممیزی و پاک‌سازی کنند.

اگر هیچ زیرفرمانی داده نشود، `openclaw commitments` تعهدات در انتظار را فهرست می‌کند.

## استفاده

```bash
openclaw commitments [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments list [--all] [--agent <id>] [--status <status>] [--json]
openclaw commitments dismiss <id...> [--json]
```

## گزینه‌ها

- `--all`: همهٔ وضعیت‌ها را به‌جای فقط تعهدات در انتظار نمایش می‌دهد.
- `--agent <id>`: نتایج را به یک شناسهٔ عامل محدود می‌کند.
- `--status <status>`: نتایج را بر اساس وضعیت فیلتر می‌کند. مقادیر: `pending`، `sent`،
  `dismissed`، `snoozed` یا `expired`. مقادیر ناشناخته با خطا خاتمه می‌یابند.
- `--json`: خروجی JSON قابل‌خواندن برای ماشین تولید می‌کند.

`dismiss` شناسه‌های تعهد داده‌شده را با وضعیت `dismissed` علامت‌گذاری می‌کند.

## مثال‌ها

فهرست‌کردن تعهدات در انتظار:

```bash
openclaw commitments
```

فهرست‌کردن همهٔ تعهدات ذخیره‌شده:

```bash
openclaw commitments --all
```

فیلترکردن برای یک عامل:

```bash
openclaw commitments --agent main
```

یافتن تعهدات به‌تعویق‌افتاده:

```bash
openclaw commitments --status snoozed
```

حذف یک یا چند تعهد:

```bash
openclaw commitments dismiss cm_abc123 cm_def456
```

برون‌بری به‌صورت JSON:

```bash
openclaw commitments --all --json
```

## خروجی

خروجی متنی تعداد تعهدات، مسیر پایگاه دادهٔ مشترک SQLite، همهٔ فیلترهای فعال
و برای هر تعهد یک ردیف چاپ می‌کند:

- شناسهٔ تعهد
- وضعیت
- نوع (`event_check_in`، `deadline_check`، `care_check_in` یا `open_loop`)
- زودترین زمان سررسید
- دامنه (عامل/کانال/هدف)
- متن پیشنهادی برای پیگیری

خروجی JSON شامل تعداد، فیلترهای فعال وضعیت و عامل،
مسیر پایگاه دادهٔ مشترک SQLite و رکوردهای کامل ذخیره‌شده است.

## مرتبط

- [تعهدات استنباطی](/fa/concepts/commitments)
- [نمای کلی حافظه](/fa/concepts/memory)
- [Heartbeat](/fa/gateway/heartbeat)
- [وظایف زمان‌بندی‌شده](/fa/automation/cron-jobs)
