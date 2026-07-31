---
read_when:
    - به‌روزرسانی رابط کاربری تنظیمات Skills در macOS
    - تغییر محدودسازی یا رفتار نصب Skills
summary: رابط کاربری تنظیمات Skills در macOS و وضعیت مبتنی بر Gateway
title: Skills (macOS)
x-i18n:
    generated_at: "2026-07-27T14:21:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fd9d8f1190320889029335e008c3605bd4bf0194f83398cedd4ae658fd90065c
    source_path: platforms/mac/skills.md
    workflow: 16
---

برنامه macOS مهارت‌های OpenClaw را از طریق Gateway ارائه می‌کند؛ مهارت‌ها را به‌صورت محلی تجزیه نمی‌کند.

## منبع داده

- `skills.status` (Gateway) همه مهارت‌ها را به‌همراه وضعیت واجد شرایط بودن و الزامات برآورده‌نشده، از جمله مسدودسازی‌های فهرست مجاز برای مهارت‌های همراه، برمی‌گرداند.
- الزامات از `metadata.openclaw.requires` در هر `SKILL.md` می‌آیند.

## عملیات نصب

- `metadata.openclaw.install` گزینه‌های نصب (brew/node/go/uv/download) را تعریف می‌کند.
- برنامه برای اجرای نصب‌کننده‌ها روی میزبان Gateway، `skills.install` را فراخوانی می‌کند.
- `security.installPolicy` تحت مالکیت اپراتور (`enabled`، `targets`، `exec`) می‌تواند پیش از اجرای فراداده نصب‌کننده، نصب مهارت‌های متکی بر Gateway را مسدود کند. اسکن داخلی کد خطرناک (که برای نصب Pluginها استفاده می‌شود) به جریان نصب مهارت متصل نیست.
- اگر همه گزینه‌های نصب `download` باشند، Gateway همه گزینه‌های دانلود را ارائه می‌کند.
- در غیر این صورت، Gateway با استفاده از ترجیحات نصب فعلی (`skills.install.preferBrew`، `skills.install.nodeManager`) و فایل‌های اجرایی میزبان، یک نصب‌کننده ترجیحی را انتخاب می‌کند: ابتدا Homebrew، هنگامی که `preferBrew` فعال و `brew` موجود باشد؛ سپس `uv`؛ سپس مدیر node پیکربندی‌شده؛ سپس در صورت دسترس‌بودن، دوباره Homebrew (حتی بدون `preferBrew`)؛ سپس `go`؛ و در نهایت `download`.
- برچسب‌های نصب Node، مدیر node پیکربندی‌شده، از جمله `yarn`، را منعکس می‌کنند.

## کلیدهای محیط/API

- برنامه کلیدها را در `~/.openclaw/openclaw.json`، زیر `skills.entries.<skillKey>` ذخیره می‌کند.
- `skills.update`، `enabled`، `apiKey` و `env` را وصله می‌کند.

## حالت راه‌دور

- نصب و به‌روزرسانی‌های پیکربندی روی میزبان Gateway انجام می‌شوند، نه روی Mac محلی.

## مرتبط

- [Skills](/fa/tools/skills)
- [برنامه macOS](/fa/platforms/macos)
