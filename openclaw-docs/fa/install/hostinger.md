---
read_when:
    - راه‌اندازی OpenClaw روی Hostinger
    - در جست‌وجوی یک VPS مدیریت‌شده برای OpenClaw هستید؟
    - استفاده از OpenClaw با نصب یک‌کلیکی Hostinger
summary: میزبانی OpenClaw روی Hostinger
title: Hostinger
x-i18n:
    generated_at: "2026-07-27T16:41:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7dc49e741f8581928553e2426ed91f92df6e7b0c31dd8780c0d6e891a07be263
    source_path: install/hostinger.md
    workflow: 16
---

یک Gateway دائمی OpenClaw را روی [Hostinger](https://www.hostinger.com/openclaw) اجرا کنید؛ یا به‌صورت استقرار مدیریت‌شده با **1-Click**، یا به‌صورت نصب روی **VPS** که خودتان آن را مدیریت می‌کنید.

## پیش‌نیازها

- حساب Hostinger ([ثبت‌نام](https://www.hostinger.com/openclaw))
- حدود 5-10 دقیقه

## گزینه A: OpenClaw با 1-Click

Hostinger زیرساخت، Docker و به‌روزرسانی‌های خودکار را مدیریت می‌کند. سریع‌ترین راه برای راه‌اندازی یک نمونه در حال اجرا است.

<Steps>
  <Step title="خرید و راه‌اندازی">
    1. در [صفحه OpenClaw در Hostinger](https://www.hostinger.com/openclaw)، یک طرح Managed OpenClaw انتخاب کنید و فرایند پرداخت را تکمیل کنید.

    <Note>
    هنگام پرداخت می‌توانید اعتبارهای **Ready-to-Use AI** را انتخاب کنید که از پیش خریداری شده‌اند و بلافاصله در OpenClaw یکپارچه می‌شوند؛ به هیچ حساب خارجی یا کلید API از ارائه‌دهندگان دیگر نیازی نیست. می‌توانید فوراً گفت‌وگو را شروع کنید. در روش دیگر، هنگام راه‌اندازی کلید خود را از Anthropic، OpenAI، Google Gemini یا xAI وارد کنید.
    </Note>

  </Step>

  <Step title="انتخاب کانال پیام‌رسانی">
    یک یا چند کانال را برای اتصال انتخاب کنید:

    - **WhatsApp** -- کد QR نمایش‌داده‌شده در راهنمای راه‌اندازی را اسکن کنید.
    - **Telegram** -- توکن ربات دریافتی از [BotFather](https://t.me/BotFather) را جای‌گذاری کنید.

  </Step>

  <Step title="تکمیل نصب">
    برای استقرار نمونه، روی **Finish** کلیک کنید. پس از آماده‌شدن، از بخش **OpenClaw Overview** در hPanel به داشبورد OpenClaw دسترسی پیدا کنید.
  </Step>

</Steps>

## گزینه B: OpenClaw روی VPS

کنترل بیشتری بر سرور فراهم می‌کند. Hostinger، OpenClaw را از طریق Docker روی VPS شما مستقر می‌کند؛ شما آن را از طریق **Docker Manager** در hPanel مدیریت می‌کنید.

<Steps>
  <Step title="خرید VPS">
    1. در [صفحه OpenClaw در Hostinger](https://www.hostinger.com/openclaw)، یک طرح OpenClaw on VPS انتخاب کنید و فرایند پرداخت را تکمیل کنید.

    <Note>
    هنگام پرداخت می‌توانید اعتبارهای **Ready-to-Use AI** را انتخاب کنید؛ این اعتبارها از پیش خریداری شده‌اند و بلافاصله در OpenClaw یکپارچه می‌شوند، بنابراین می‌توانید بدون هیچ حساب خارجی یا کلید API از ارائه‌دهندگان دیگر، گفت‌وگو را شروع کنید.
    </Note>

  </Step>

  <Step title="پیکربندی OpenClaw">
    پس از آماده‌سازی VPS، فیلدهای پیکربندی را تکمیل کنید:

    - **توکن Gateway** -- به‌طور خودکار تولید می‌شود؛ آن را برای استفاده بعدی ذخیره کنید.
    - **شماره WhatsApp** -- شماره شما همراه با کد کشور (اختیاری).
    - **توکن ربات Telegram** -- از [BotFather](https://t.me/BotFather) دریافت کنید (اختیاری).
    - **کلیدهای API** -- فقط در صورتی لازم است که هنگام پرداخت، اعتبارهای Ready-to-Use AI را انتخاب نکرده باشید.

  </Step>

  <Step title="راه‌اندازی OpenClaw">
    روی **Deploy** کلیک کنید. پس از اجراشدن، با کلیک روی **Open** در hPanel، داشبورد OpenClaw را باز کنید.
  </Step>

</Steps>

گزارش‌ها، راه‌اندازی‌های مجدد و به‌روزرسانی‌ها از رابط Docker Manager در hPanel اجرا می‌شوند. برای به‌روزرسانی، در Docker Manager روی **Update** کلیک کنید تا جدیدترین تصویر دریافت شود.

## تأیید راه‌اندازی

در کانالی که متصل کرده‌اید، پیام «سلام» را برای دستیار خود ارسال کنید. OpenClaw پاسخ می‌دهد و شما را در تنظیم ترجیحات اولیه راهنمایی می‌کند.

## عیب‌یابی

**داشبورد بارگیری نمی‌شود** -- چند دقیقه منتظر بمانید تا آماده‌سازی کانتینر تکمیل شود، سپس گزارش‌های Docker Manager را در hPanel بررسی کنید.

**کانتینر Docker مرتباً راه‌اندازی مجدد می‌شود** -- گزارش‌های Docker Manager را باز کنید و به‌دنبال خطاهای پیکربندی بگردید (توکن‌های موجودنیست، کلیدهای API نامعتبر).

**ربات Telegram پاسخ نمی‌دهد** -- اگر جفت‌سازی پیام خصوصی لازم باشد، فرستنده ناشناس به‌جای پاسخ، یک کد کوتاه جفت‌سازی دریافت می‌کند. آن را از گفت‌وگوی داشبورد OpenClaw یا در صورت داشتن دسترسی پوسته به کانتینر، با `openclaw pairing approve telegram <CODE>` تأیید کنید. به [جفت‌سازی](/fa/channels/pairing) مراجعه کنید.

## گام‌های بعدی

- [کانال‌ها](/fa/channels) -- Telegram، WhatsApp، Discord و موارد دیگر را متصل کنید
- [پیکربندی Gateway](/fa/gateway/configuration) -- همه گزینه‌های پیکربندی

## مرتبط

- [نمای کلی نصب](/fa/install)
- [میزبانی VPS](/fa/vps)
- [DigitalOcean](/fa/install/digitalocean)
