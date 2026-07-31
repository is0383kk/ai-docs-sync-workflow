---
read_when:
    - در حال نصب، پیکربندی یا ممیزی Plugin ‏opencode هستید
summary: پشتیبانی از ارائه‌دهنده مدل OpenCode را به OpenClaw اضافه می‌کند.
title: Plugin ‏OpenCode
x-i18n:
    generated_at: "2026-07-27T15:56:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aecf396cfc645e4a036b8130ed7f33db9081dffda120c6d06ebe863dd3be3730
    source_path: plugins/reference/opencode.md
    workflow: 16
---

# Plugin OpenCode

پشتیبانی از ارائه‌دهنده مدل OpenCode را به OpenClaw اضافه می‌کند.

## توزیع

- بسته: `@openclaw/opencode-provider`
- مسیر نصب: همراه OpenClaw ارائه می‌شود

## سطح

ارائه‌دهندگان: `opencode`؛ قراردادها: `mediaUnderstandingProviders`

<!-- openclaw-plugin-reference:manual-start -->

## نشست‌های بومی

OpenClaw به‌طور خودکار CLI ‏`opencode` را روی Gateway و Nodeهای جفت‌شده شناسایی می‌کند. سپس نشست‌های ذخیره‌شده در گروه نشست‌های **OpenCode** در نوار کناری ظاهر می‌شوند و امکان مرور فقط‌خواندنی رونوشت‌ها از طریق فرمان‌های رسمی `opencode --pure db ... --format json`
و `opencode --pure export` فراهم می‌شود. محیط محدودشده و حالت `--pure`
مانع از آن می‌شوند که مرور کاتالوگ، Pluginهای پروژه را بارگذاری کند یا اعتبارنامه‌های نامرتبط
Gateway را به ارث ببرد.

برای غیرفعال‌کردن کشف، گزینه **کاتالوگ نشست OpenCode** را در **پیکربندی > Pluginها > OpenCode** خاموش کنید. این گزینه به‌طور پیش‌فرض فعال است.

<!-- openclaw-plugin-reference:manual-end -->

## مستندات مرتبط

- [opencode](/fa/providers/opencode)
