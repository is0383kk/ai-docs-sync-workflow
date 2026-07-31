---
read_when:
    - استقرار OpenClaw در Railway
    - استقرار ابری با یک کلیک و رابط کنترل مبتنی بر مرورگر می‌خواهید
summary: OpenClaw را با قالب یک‌کلیکی روی Railway مستقر کنید
title: Railway
x-i18n:
    generated_at: "2026-07-27T15:40:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cbef00b8de61545e9971b18164472c2f47fe607f69ec36f83a27a11b65ea863f
    source_path: install/railway.mdx
    workflow: 16
---

OpenClaw را با یک قالب تک‌کلیکی روی Railway مستقر کنید و از طریق Control UI وب به آن دسترسی داشته باشید. این ساده‌ترین مسیر «بدون ترمینال روی سرور» است: Railway، Gateway را برای شما اجرا می‌کند.

## استقرار تک‌کلیکی

<a href="https://railway.com/deploy/clawdbot-railway-template" target="_blank" rel="noreferrer">
  استقرار روی Railway
</a>

<Steps>
  <Step title="استقرار قالب">
    روی **Deploy on Railway** در بالا کلیک کنید.
  </Step>

<Step title="افزودن یک Volume">
  یک Volume را با نقطهٔ اتصال `/data` پیوست کنید (برای ماندگاری وضعیت الزامی است).
</Step>

  <Step title="تنظیم متغیرها">
    **Variables** الزامی را در سرویس تنظیم کنید:

    - `OPENCLAW_GATEWAY_PORT=8080` (الزامی -- باید با پورت Public Networking مطابقت داشته باشد)
    - `OPENCLAW_GATEWAY_TOKEN` (الزامی؛ آن را مانند یک راز مدیریتی نگهداری کنید)
    - `OPENCLAW_STATE_DIR=/data/.openclaw` (توصیه‌شده)
    - `OPENCLAW_WORKSPACE_DIR=/data/workspace` (توصیه‌شده)

  </Step>

<Step title="فعال‌سازی شبکهٔ عمومی">
  در بخش **Public Networking**، گزینهٔ **HTTP Proxy** را برای سرویس روی پورت `8080` فعال کنید.
</Step>

  <Step title="اتصال">
    نشانی عمومی خود را در **Railway -> your service -> Settings -> Domains** پیدا کنید -- یا یک دامنهٔ تولیدشده (اغلب `https://<something>.up.railway.app`) یا دامنهٔ سفارشی متصل‌شدهٔ خودتان.

    `https://<your-railway-domain>/openclaw` را باز کنید و با استفاده از راز مشترک پیکربندی‌شده متصل شوید. قالب به‌طور پیش‌فرض از `OPENCLAW_GATEWAY_TOKEN` استفاده می‌کند؛ اگر آن را با احراز هویت مبتنی بر گذرواژه جایگزین کردید، به‌جای آن از همان گذرواژه استفاده کنید.

  </Step>
</Steps>

## آنچه دریافت می‌کنید

- Gateway میزبانی‌شدهٔ OpenClaw + Control UI
- ذخیره‌سازی پایدار از طریق Railway Volume ‏(`/data`)؛ بنابراین `openclaw.json`، ‏`auth-profiles.json` مختص هر عامل، وضعیت کانال/ارائه‌دهنده، نشست‌ها و فضای کاری پس از استقرارهای مجدد حفظ می‌شوند

## اتصال یک کانال

برای دستورالعمل‌های راه‌اندازی کانال، از Control UI در `/openclaw` استفاده کنید یا `openclaw onboard` را از طریق پوستهٔ Railway اجرا کنید:

- [Discord](/fa/channels/discord)
- [Telegram](/fa/channels/telegram) (سریع‌ترین گزینه -- فقط یک توکن ربات)
- [همهٔ کانال‌ها](/fa/channels)

## پشتیبان‌گیری و مهاجرت

از وضعیت، پیکربندی، پروفایل‌های احراز هویت و فضای کاری خود خروجی بگیرید:

```bash
openclaw backup create
```

این فرمان یک بایگانی پشتیبان قابل‌انتقال شامل وضعیت OpenClaw و هر فضای کاری پیکربندی‌شده ایجاد می‌کند. برای جزئیات، [پشتیبان‌گیری](/fa/cli/backup) را ببینید.

## گام‌های بعدی

- کانال‌های پیام‌رسانی را راه‌اندازی کنید: [کانال‌ها](/fa/channels)
- Gateway را پیکربندی کنید: [پیکربندی Gateway](/fa/gateway/configuration)
- OpenClaw را به‌روز نگه دارید: [به‌روزرسانی](/fa/install/updating)
