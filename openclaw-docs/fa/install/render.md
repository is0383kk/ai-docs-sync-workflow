---
read_when:
    - استقرار OpenClaw در Render
    - می‌خواهید با Render Blueprints یک استقرار ابری اعلانی داشته باشید
summary: استقرار OpenClaw روی Render با زیرساخت به‌عنوان کد
title: Render
x-i18n:
    generated_at: "2026-07-27T16:42:26Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a5fbb3c6df04e186df958a62a6130da4e3e485acfeecc7e85fee0d5b69a0438f
    source_path: install/render.mdx
    workflow: 16
---

OpenClaw را با استفاده از Blueprint مخزن در `render.yaml` روی [Render](https://render.com) مستقر کنید. این Blueprint سرویس، دیسک و متغیرهای محیطی را در یک فایل تعریف می‌کند.

## پیش‌نیازها

- یک [حساب Render](https://render.com) (سطح رایگان موجود است)
- یک کلید API از [ارائه‌دهنده مدل](/fa/providers) دلخواهتان

## استقرار

[استقرار در Render](https://render.com/deploy?repo=https://github.com/openclaw/openclaw)

این کار یک سرویس Render را از `render.yaml` ایجاد می‌کند، ایمیج Docker را می‌سازد و آن را مستقر می‌کند. URL سرویس شما از الگوی `https://<service-name>.onrender.com` پیروی می‌کند.

## Blueprint

```yaml
services:
  - type: web
    name: openclaw
    runtime: docker
    plan: starter
    healthCheckPath: /health
    envVars:
      - key: OPENCLAW_GATEWAY_PORT
        value: "8080"
      - key: OPENCLAW_STATE_DIR
        value: /data/.openclaw
      - key: OPENCLAW_WORKSPACE_DIR
        value: /data/workspace
      - key: OPENCLAW_GATEWAY_TOKEN
        generateValue: true # یک توکن امن را به‌طور خودکار تولید می‌کند
    disk:
      name: openclaw-data
      mountPath: /data
      sizeGB: 1
```

| قابلیت               | هدف                                                    |
| --------------------- | ---------------------------------------------------------- |
| `runtime: docker`     | از Dockerfile مخزن می‌سازد                          |
| `healthCheckPath`     | Render بر `/health` نظارت می‌کند و نمونه‌های ناسالم را مجدداً راه‌اندازی می‌کند |
| `generateValue: true` | یک مقدار امن از نظر رمزنگاری را به‌طور خودکار تولید می‌کند            |
| `disk`                | فضای ذخیره‌سازی پایداری که پس از استقرار مجدد نیز باقی می‌ماند                 |

## انتخاب پلن

| پلن      | توقف خودکار         | دیسک          | مناسب برای                      |
| --------- | ----------------- | ------------- | ----------------------------- |
| رایگان      | پس از 15 دقیقه بی‌کاری | در دسترس نیست | آزمایش، دموها                |
| Starter   | هرگز             | 1GB+          | استفاده شخصی، تیم‌های کوچک     |
| Standard+ | هرگز             | 1GB+          | محیط عملیاتی، چند کانال |

Blueprint به‌طور پیش‌فرض از `starter` استفاده می‌کند. برای استفاده از سطح رایگان، `plan: free` را در `render.yaml` فورک خود تغییر دهید — توجه داشته باشید که بدون دیسک پایدار، وضعیت OpenClaw با هر استقرار بازنشانی می‌شود.

## پس از استقرار

### دسترسی به رابط کاربری کنترل

داشبورد وب در `https://<your-service>.onrender.com/` در دسترس است. با استفاده از راز مشترک متصل شوید: `OPENCLAW_GATEWAY_TOKEN` که به‌طور خودکار تولید شده است (آن را در **Dashboard → your service → Environment** بیابید)، یا اگر به احراز هویت با گذرواژه تغییر داده‌اید، از گذرواژه خود استفاده کنید.

### گزارش‌ها

**Dashboard → your service → Logs** گزارش‌های ساخت (ایجاد تصویر Docker)، گزارش‌های استقرار (راه‌اندازی سرویس) و گزارش‌های زمان اجرا (خروجی برنامه) را نمایش می‌دهد.

### دسترسی به پوسته

**Dashboard → your service → Shell** یک نشست پوسته باز می‌کند. دیسک پایدار در `/data` سوار شده است.

### متغیرهای محیطی

متغیرها را در **Dashboard → your service → Environment** ویرایش کنید. تغییرات باعث استقرار مجدد خودکار می‌شوند.

### استقرار خودکار

Render هرگاه commit جدیدی به شاخهٔ مخزن متصل اضافه شود، به‌طور خودکار دوباره استقرار می‌یابد. اگر به‌جای fork خودتان، مستقیماً از `openclaw/openclaw` استقرار داده‌اید، دسترسی push برای فعال‌کردن آن ندارید؛ بنابراین با اجرای همگام‌سازی دستی Blueprint از Dashboard به‌روزرسانی کنید، یا سرویس را به fork خودتان متصل کنید.

## دامنهٔ سفارشی

1. **Dashboard → your service → Settings → Custom Domains**
2. دامنهٔ خود را اضافه کنید
3. DNS را طبق دستورالعمل پیکربندی کنید (CNAME به `*.onrender.com`)
4. Render به‌طور خودکار گواهی TLS صادر می‌کند

## مقیاس‌پذیری

- **عمودی**: برای CPU/RAM بیشتر، طرح را تغییر دهید. معمولاً برای OpenClaw کافی است.
- **افقی**: تعداد نمونه‌ها را افزایش دهید (طرح Standard و بالاتر). ازآنجاکه OpenClaw وضعیت زمان اجرا را روی دیسک محلی نگه می‌دارد، این کار به نشست‌های چسبنده یا مدیریت وضعیت خارجی نیاز دارد.

## پشتیبان‌گیری و مهاجرت

از پوستهٔ Render Dashboard، در هر زمان وضعیت، پیکربندی، پروفایل‌های احراز هویت و فضای کاری را صادر کنید:

```bash
openclaw backup create
```

این فرمان یک بایگانی پشتیبان قابل‌انتقال ایجاد می‌کند. به [پشتیبان‌گیری](/fa/cli/backup) مراجعه کنید.

## عیب‌یابی

### سرویس راه‌اندازی نمی‌شود

گزارش‌های استقرار را در Render Dashboard بررسی کنید. مشکلات رایج:

- نبودن `OPENCLAW_GATEWAY_TOKEN` — بررسی کنید که در **Dashboard → Environment** تنظیم شده باشد
- ناهماهنگی درگاه — مطمئن شوید `OPENCLAW_GATEWAY_PORT=8080` تا Gateway به درگاهی که Render انتظار دارد متصل شود

### شروع سرد کند (سطح رایگان)

سرویس‌های سطح رایگان پس از 15 دقیقه عدم فعالیت متوقف می‌شوند؛ نخستین درخواست پس از توقف، هنگام راه‌اندازی کانتینر چند ثانیه طول می‌کشد. برای فعالیت دائمی، به Starter ارتقا دهید.

### ازدست‌رفتن داده‌ها پس از استقرار مجدد

این اتفاق در سطح رایگان رخ می‌دهد (فاقد دیسک پایدار). به یک طرح پولی ارتقا دهید، یا به‌طور منظم با `openclaw backup create` از پوستهٔ Render یک نسخهٔ پشتیبان صادر کنید.

### شکست بررسی سلامت

اگر ساخت‌ها موفق‌اند اما استقرارها شکست می‌خورند، ممکن است راه‌اندازی سرویس بیش‌ازحد طول بکشد یا `/health` در دسترس نباشد. موارد زیر را بررسی کنید:

- گزارش‌های ساخت برای یافتن خطاها
- این‌که آیا کانتینر با `docker build && docker run` به‌صورت محلی اجرا می‌شود

## گام‌های بعدی

- کانال‌های پیام‌رسانی را راه‌اندازی کنید: [کانال‌ها](/fa/channels)
- Gateway را پیکربندی کنید: [پیکربندی Gateway](/fa/gateway/configuration)
- OpenClaw را به‌روز نگه دارید: [به‌روزرسانی](/fa/install/updating)
