---
read_when:
    - راه‌اندازی دستی یک فضای کاری
summary: قالب فضای کاری برای HEARTBEAT.md
title: قالب HEARTBEAT.md
x-i18n:
    generated_at: "2026-07-27T16:10:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d5b02cd62708a87515c4ae59bd2ffab3e4c8ebf81f4126fdd43ced756241b151
    source_path: reference/templates/HEARTBEAT.md
    workflow: 16
---

# قالب HEARTBEAT.md

`HEARTBEAT.md` در فضای کاری عامل قرار دارد و شامل چک‌لیست دوره‌ای Heartbeat است. آن را خالی یا فقط شامل فاصله‌های خالی، توضیحات Markdown، عنوان‌های ATX، فهرست‌های خالی (`- `، `* [ ]`) یا نشانگرهای حصار نگه دارید تا OpenClaw فراخوانی مدل Heartbeat را به‌طور کامل نادیده بگیرد (`reason=empty-heartbeat-file`).

محتوای پیش‌فرض ارائه‌شده:

```markdown
<!-- Heartbeat template; comments-only content prevents scheduled heartbeat API calls. -->

# برای نادیده گرفتن فراخوانی‌های API مربوط به Heartbeat، این فایل را خالی (یا فقط شامل توضیحات) نگه دارید.

# هنگامی که Heartbeat باید زمینه مشترک را بررسی کند، یک چک‌لیست کوتاه در زیر اضافه کنید.
```

تنها زمانی یک چک‌لیست کوتاه زیر خطوط توضیح اضافه کنید که یک نوبت Heartbeat باید موارد را با هم بررسی کند. آن را مختصر نگه دارید: اجرای Heartbeat در هر تیک این فایل را می‌خواند (به‌طور پیش‌فرض هر 30 دقیقه)، بنابراین دستورالعمل‌های حجیم در هر بار بیدارشدن توکن مصرف می‌کنند.

برای بررسی‌هایی که به‌طور مستقل زمان‌بندی شده‌اند یا فقط در موعد مقرر انجام می‌شوند، [کارهای Cron](/fa/automation/cron-jobs) ایجاد کنید. پیش‌نویس Heartbeat دیگر از نحو زمان‌بندی پشتیبانی نمی‌کند. برای تبدیل بلوک‌های قدیمی `tasks:`، دستور `openclaw doctor --fix` را اجرا کنید.

## مرتبط

- [Heartbeat](/fa/gateway/heartbeat)
- [پیکربندی Heartbeat](/fa/gateway/config-agents)
