---
read_when:
    - یک ربات دستیار شخصی Zalo با ورود از طریق کد QR می‌خواهید
    - در حال نصب یا عیب‌یابی Plugin کانال openclaw-zaloclawbot هستید
summary: راه‌اندازی کانال Zalo ClawBot از طریق Plugin خارجی openclaw-zaloclawbot
title: ClawBot زالو
x-i18n:
    generated_at: "2026-07-27T16:16:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 76c9f79d114856b86026a5e4b98a43f451b0d3f16dd41a67e9226da4f8b37b33
    source_path: channels/zaloclawbot.md
    workflow: 16
---

OpenClaw از طریق Plugin خارجی `@zalo-platforms/openclaw-zaloclawbot` که در کاتالوگ فهرست شده است، به Zalo ClawBot متصل می‌شود. ورود با کد QR یک Zalo Mini App انجام می‌شود؛ شناسه Plugin در پیکربندی `openclaw-zaloclawbot` است.

## سازگاری

| نسخه Plugin | نسخه OpenClaw | dist-tag در npm | وضعیت        |
| -------------- | ---------------- | ------------ | ------------- |
| 0.1.4          | >=2026.4.10      | `latest`     | فعال / بتا |

## پیش‌نیازها

- Node.js >= 22
- [OpenClaw](https://docs.openclaw.ai/install) نصب‌شده (`openclaw` CLI در دسترس باشد)
- یک حساب Zalo روی دستگاه همراه برای اسکن کد QR ورود

## نصب با onboard (توصیه‌شده)

```bash
openclaw onboard
```

از منوی کانال، **Zalo ClawBot** را انتخاب کنید. راه‌انداز، Plugin را از کاتالوگ رسمی نصب می‌کند (با صحت‌سنجی یکپارچگی)، کد QR ورود را در ترمینال نمایش می‌دهد و پس از اسکن آن با برنامه Zalo، راه‌اندازی کانال را تکمیل می‌کند.

## نصب دستی

برای افزودن کانال به Gateway که قبلاً راه‌اندازی شده است:

### 1. نصب Plugin

```bash
openclaw plugins install "@zalo-platforms/openclaw-zaloclawbot@0.1.4"
```

از همین نسخه دقیق و ثابت استفاده کنید تا OpenClaw هنگام نصب، بسته را با هش یکپارچگی کاتالوگ اعتبارسنجی کند.

### 2. فعال‌کردن Plugin در پیکربندی

```bash
openclaw config set plugins.entries.openclaw-zaloclawbot.enabled true
```

### 3. ایجاد کد QR و ورود

```bash
openclaw channels login --channel openclaw-zaloclawbot
```

کد QR نمایش‌داده‌شده در ترمینال را با برنامه موبایل Zalo اسکن کنید، شرایط استفاده را در Zalo Mini App بپذیرید و نشست را مجاز کنید.

### 4. راه‌اندازی مجدد Gateway

```bash
openclaw gateway restart
```

## نحوه کار

برخلاف کانال استاندارد Zalo که نیازمند ثبت Zalo Official Account (OA) اختصاصی و پیکربندی اطلاعات اعتبارسنجی ایستای توسعه‌دهنده است، Zalo ClawBot یک **دستیار شخصی وابسته به مالک** روی زیرساخت رسمی مشترک است:

1. **راه‌اندازی اولیه:** کد QR به یک Zalo Mini App منتهی می‌شود که ربات خصوصی و تازه تأمین‌شده‌ای را زیر یک OA رسمی مشترک، مستقیماً به شناسه کاربری Zalo شما متصل می‌کند.
2. **حریم خصوصی وابسته به مالک:** ربات فقط با مالک خود ارتباط برقرار می‌کند. پیام‌های کاربران دیگر در سطح پلتفرم حذف می‌شوند.
3. **مسیر API رسمی:** Plugin از APIهای Zalo Bot Platform استفاده می‌کند، نه خودکارسازی مرورگر یا نشست وب.

## جزئیات داخلی

Plugin از طریق یک حلقه long-polling پایدار (`getUpdates`) با Zalo ارتباط برقرار می‌کند. Webhookها به‌طور پیش‌فرض برای اجرای محلی Gateway در دسکتاپ یا ترمینال غیرفعال هستند. پیام‌ها در سمت کلاینت پردازش و به محیط اجرای عامل محلی شما نگاشت می‌شوند.

Plugin اطلاعات اعتبارسنجی ربات را در دایرکتوری وضعیت OpenClaw مدیریت می‌کند. این دایرکتوری را حساس در نظر بگیرید و همان سیاست کنترل دسترسی و پشتیبان‌گیری سایر وضعیت OpenClaw را برای آن نیز اعمال کنید.

محیط اجرای این Plugin کاملاً در بسته خارجی `@zalo-platforms/openclaw-zaloclawbot` قرار دارد؛ جزئیات رفتاری زیر، فراتر از نصب و پیکربندی، مطابق گزارش نگه‌دارندگان Plugin است و با کد منبع هسته OpenClaw اعتبارسنجی نشده است.

## عیب‌یابی

- **پایان مهلت ورود با QR:** توکن ورود (`zbsk`) برای حفظ امنیت پس از 5 دقیقه منقضی می‌شود. اگر کد QR پیش از اسکن منقضی شد، فرمان ورود را دوباره اجرا کنید تا کد جدیدی ایجاد شود.
- **Gateway بارگذاری نمی‌شود:** تأیید کنید نسخه میزبان OpenClaw شما `2026.4.10` یا بالاتر است. نسخه‌های قدیمی‌تر از دفتر ثبت نصب Plugin خارجی npm که این شناسه نیاز دارد، پشتیبانی نمی‌کنند.

## مرتبط

- [نمای کلی کانال‌ها](/fa/channels) - همه کانال‌های پشتیبانی‌شده
- [Zalo](/fa/channels/zalo) - کانال همراه Zalo Bot Creator / Marketplace
- [جفت‌سازی](/fa/channels/pairing) - احراز هویت پیام مستقیم و جریان جفت‌سازی
- [Pluginها](/fa/tools/plugin) - نصب و مدیریت Pluginها
