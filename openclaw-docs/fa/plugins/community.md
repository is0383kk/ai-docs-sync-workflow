---
doc-schema-version: 1
read_when:
    - می‌خواهید Pluginهای شخص ثالث OpenClaw را پیدا کنید
    - می‌خواهید Plugin خود را در ClawHub منتشر یا فهرست کنید
summary: یافتن و انتشار Pluginهای OpenClaw که توسط جامعه نگهداری می‌شوند
title: Plugin‌های جامعه کاربری
x-i18n:
    generated_at: "2026-07-27T15:51:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a9eb477f20da8171a35c22ea6b112d77ff4afe0878f60314c052746aef4e0ac
    source_path: plugins/community.md
    workflow: 16
---

Pluginهای جامعه بسته‌های شخص ثالثی هستند که OpenClaw را با
کانال‌ها، ابزارها، ارائه‌دهندگان، هوک‌ها یا قابلیت‌های دیگر گسترش می‌دهند. از
[ClawHub](/clawhub) به‌عنوان بستر اصلی کشف Pluginهای عمومی جامعه
استفاده کنید.

## یافتن Pluginها

ClawHub را از CLI جست‌وجو کنید:

```bash
openclaw plugins search "calendar"
```

یک Plugin از ClawHub را با پیشوند صریح منبع نصب کنید:

```bash
openclaw plugins install clawhub:<package-name>
```

در دوره گذار راه‌اندازی، npm همچنان یک مسیر پشتیبانی‌شده برای نصب مستقیم است:

```bash
openclaw plugins install npm:<package-name>
```

برای نمونه‌های متداول نصب، به‌روزرسانی، بررسی و حذف نصب، از
[مدیریت Pluginها](/fa/plugins/manage-plugins) استفاده کنید. برای مرجع کامل فرمان‌ها و
قواعد انتخاب منبع، از [`openclaw plugins`](/fa/cli/plugins) استفاده کنید.

## انتشار Pluginها

Pluginهای عمومی جامعه را در ClawHub منتشر کنید تا کاربران OpenClaw بتوانند آن‌ها را پیدا
و نصب کنند. فهرست زنده بسته‌ها، تاریخچه انتشار، وضعیت اسکن
و راهنمای نصب تحت مدیریت ClawHub است؛ مستندات یک کاتالوگ ثابت
از Pluginهای شخص ثالث نگهداری نمی‌کنند.

```bash
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

پیش از انتشار، مطمئن شوید Plugin دارای فراداده بسته، یک مانیفست Plugin،
مستندات راه‌اندازی و یک مسئول نگهداری مشخص است. ClawHub پیش از
ایجاد نسخه انتشار، محدوده مالک، نام بسته، نسخه، محدودیت‌های فایل و فراداده منبع را
اعتبارسنجی می‌کند، سپس نسخه‌های جدید را تا پایان بازبینی و راستی‌آزمایی از
بسترهای عادی نصب و دانلود پنهان نگه می‌دارد.

چک‌لیست پیش از انتشار:

| الزام                 | دلیل                                                |
| -------------------- | --------------------------------------------------- |
| انتشار در ClawHub     | کاربران برای کارکرد راهنمای `openclaw plugins install` به آن نیاز دارند |
| مخزن عمومی GitHub     | بازبینی منبع، پیگیری مشکلات، شفافیت                 |
| مستندات راه‌اندازی و استفاده | کاربران باید بدانند چگونه آن را پیکربندی کنند       |
| نگهداری فعال          | به‌روزرسانی‌های اخیر یا رسیدگی پاسخ‌گو به مشکلات    |

قرارداد کامل انتشار:

- [انتشار در ClawHub](/fa/clawhub/publishing) - مالکان، محدوده‌ها، نسخه‌های انتشار،
  بازبینی، اعتبارسنجی بسته و انتقال بسته
- [ساخت Pluginها](/fa/plugins/building-plugins) - ساختار بسته Plugin
  و گردش‌کار نخستین انتشار
- [مانیفست Plugin](/fa/plugins/manifest) - فیلدهای مانیفست بومی Plugin

## مرتبط

- [Pluginها](/fa/tools/plugin) - نصب، پیکربندی، راه‌اندازی مجدد و عیب‌یابی
- [مدیریت Pluginها](/fa/plugins/manage-plugins) - نمونه‌های فرمان
- [انتشار در ClawHub](/fa/clawhub/publishing) - قواعد انتشار و نسخه‌دهی
