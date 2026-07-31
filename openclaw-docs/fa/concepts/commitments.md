---
read_when:
    - در حال ارتقای پیکربندی‌ای هستید که از تعهدات استنباط‌شده استفاده می‌کرد
    - می‌خواهید سوابق پیگیریِ ذخیره‌شده قبلی را بررسی یا حذف کنید
sidebarTitle: Commitments
summary: راهنمای وضعیت و پاک‌سازی تعهدات استنباط‌شده و منسوخِ پیگیری‌های بعدی
title: تعهدات استنباط‌شده
x-i18n:
    generated_at: "2026-07-27T15:03:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cfaa8c44be4ffb8db48279dba5347d4f598a193bfc4e244aeaed7a93e00ffb79
    source_path: concepts/commitments.md
    workflow: 16
---

آزمایش تعهدات استنباط‌شده بازنشسته شده است. OpenClaw دیگر پیگیری‌های جدید
مکالمه را استخراج نمی‌کند یا آن‌ها را از طریق Heartbeat تحویل نمی‌دهد، و بلوک پیکربندی سابق
`commitments` توسط `openclaw doctor --fix` حذف می‌شود.

یادآوری‌های دقیق و کارهای زمان‌بندی‌شده همچنان از
[وظایف زمان‌بندی‌شده](/fa/automation/cron-jobs) استفاده می‌کنند. واقعیت‌های پایدار مکالمه باید در
[حافظه](/fa/concepts/memory) قرار گیرند.

## رکوردهای موجود

تعهدات ذخیره‌شده قبلی در پایگاه داده مشترک وضعیت SQLite باقی می‌مانند تا
ارتقا، تاریخچه قابل‌مشاهده برای اپراتور را از بین نبرد. برای بررسی یا رد کردن آن ردیف‌ها از
CLI نگه‌داری قدیمی استفاده کنید:

```bash
openclaw commitments --all
openclaw commitments dismiss cm_abc123
```

برای مرجع فرمان نگه‌داری، به [`openclaw commitments`](/fa/cli/commitments)
مراجعه کنید.

## مرتبط

- [وظایف زمان‌بندی‌شده](/fa/automation/cron-jobs)
- [نمای کلی حافظه](/fa/concepts/memory)
- [Heartbeat](/fa/gateway/heartbeat)
