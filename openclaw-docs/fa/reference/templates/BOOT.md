---
read_when:
    - افزودن چک‌لیست BOOT.md
summary: الگوی فضای کاری برای BOOT.md
title: قالب BOOT.md
x-i18n:
    generated_at: "2026-07-27T15:55:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1adfb4d71f1f03716a1ddc4774a4cb6ead4b8be65bd9bb34066a9e1929a36b21
    source_path: reference/templates/BOOT.md
    workflow: 16
---

# BOOT.md

دستورالعمل‌های کوتاه و صریح راه‌اندازی را اینجا اضافه کنید. هوک همراه `boot-md`، در صورت وجود این فایل و داشتن محتوای غیرسفید، هر بار که Gateway راه‌اندازی می‌شود، آن را یک‌بار برای هر فضای کاری عامل اجرا می‌کند. چند عامل که فضای کاری مشترکی دارند، تنها یک اجرا را فعال می‌کنند.

این هوک به‌صورت غیرفعال عرضه می‌شود. ابتدا آن را فعال کنید:

```bash
openclaw hooks enable boot-md
```

اگر یکی از موارد چک‌لیست پیامی ارسال می‌کند، از ابزار پیام استفاده کنید، سپس دقیقاً با توکن سکوت `NO_REPLY` پاسخ دهید (بدون حساسیت به بزرگی و کوچکی حروف).

## مرتبط

- [فضای کاری عامل](/fa/concepts/agent-workspace)
- [هوک‌ها](/fa/automation/hooks#boot-md)
