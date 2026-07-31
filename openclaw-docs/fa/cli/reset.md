---
read_when:
    - می‌خواهید وضعیت محلی را پاک کنید و در عین حال CLI را نصب‌شده نگه دارید
    - می‌خواهید یک اجرای آزمایشی از مواردی که حذف خواهند شد داشته باشید
summary: مرجع CLI برای `openclaw reset` (بازنشانی وضعیت/پیکربندی محلی)
title: بازنشانی
x-i18n:
    generated_at: "2026-07-27T15:19:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54f1d320ee368dae4a4bfb32dea73d19eb35f9f30edd12d9c2580ab7e6a26fa6
    source_path: cli/reset.md
    workflow: 16
---

# `openclaw reset`

پیکربندی/وضعیت محلی را بازنشانی کنید (CLI نصب‌شده باقی می‌ماند).

```bash
openclaw reset
openclaw reset --dry-run
openclaw reset --scope config --yes --non-interactive
openclaw reset --scope config+creds+sessions --yes --non-interactive
openclaw reset --scope full --yes --non-interactive
```

## گزینه‌ها

- `--scope <scope>`: `config`، `config+creds+sessions` یا `full`
- `--yes`: از اعلان‌های تأیید صرف‌نظر می‌کند
- `--non-interactive`: اعلان‌ها را غیرفعال می‌کند؛ به `--scope` و `--yes` نیاز دارد
- `--dry-run`: بدون حذف فایل‌ها، عملیات را نمایش می‌دهد

## دامنه‌ها

| دامنه                   | مواردی که حذف می‌شوند                                                                     | ابتدا Gateway متوقف می‌شود |
| ----------------------- | --------------------------------------------------------------------------- | ------------------- |
| `config`                | فقط فایل پیکربندی                                                            | خیر                  |
| `config+creds+sessions` | فایل پیکربندی، پوشهٔ OAuth/اعتبارنامه‌ها و پوشه‌های نشست هر عامل           | بله                 |
| `full`                  | پوشهٔ وضعیت (شامل پایگاه دادهٔ SQLite مشترک) به‌همراه پوشه‌های فضای کاری | بله                 |

`config+creds+sessions` و `full` پیش از حذف وضعیت، سرویس Gateway مدیریت‌شدهٔ در حال اجرا را متوقف می‌کنند.

## نکته‌ها

- پیش از حذف وضعیت محلی، ابتدا `openclaw backup create` را برای ایجاد یک اسنپ‌شات قابل‌بازیابی اجرا کنید.
- وضعیت راه‌اندازی فضای کاری و گواهی‌ها به‌صورت ردیف‌هایی در پایگاه دادهٔ SQLite مشترک قرار دارند؛ بنابراین `full` آن‌ها را همراه با پوشهٔ وضعیت حذف می‌کند و در حال حاضر هیچ فایل جانبی گواهی وجود ندارد که جداگانه حذف شود.
- بدون `--scope`، دستور `openclaw reset` به‌صورت تعاملی دامنهٔ موردنظر برای حذف را درخواست می‌کند.
- `--non-interactive` فقط زمانی معتبر است که هر دو `--scope` و `--yes` تنظیم شده باشند.
- `config+creds+sessions` و `full` پس از پایان، `Next: openclaw onboard --install-daemon` را نمایش می‌دهند.

## مرتبط

- [مرجع CLI](/fa/cli)
