---
read_when:
    - شما در حال نصب، پیکربندی یا ممیزی Plugin انتقال فایل هستید
summary: فایل‌ها را در Nodeهای جفت‌شده، از طریق فرمان‌های اختصاصی Node دریافت، فهرست و ذخیره کنید. با استفاده از base64 روی `node.invoke` برای فایل‌های باینری تا 16 MB، محدودیت کوتاه‌شدن خروجی استاندارد bash را دور می‌زند.
title: Plugin انتقال فایل
x-i18n:
    generated_at: "2026-07-27T15:43:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f76e92a821be53e988011e2fd9dd53b107b43a8191bf4cdf41baaf918a9c5412
    source_path: plugins/reference/file-transfer.md
    workflow: 16
---

# Plugin انتقال فایل

فایل‌ها را در Nodeهای جفت‌شده از طریق فرمان‌های اختصاصی Node دریافت، فهرست و ذخیره کنید. با استفاده از base64 روی node.invoke برای فایل‌های دودویی تا 16 MB، محدودیت خروجی stdout در bash را دور می‌زند.

## توزیع

- بسته: `@openclaw/file-transfer`
- مسیر نصب: همراه OpenClaw ارائه می‌شود

## سطح

قراردادها: `tools`
