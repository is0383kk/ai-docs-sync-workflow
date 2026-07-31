---
read_when:
    - استقرار OpenClaw روی EasyRunner
    - اجرای Gateway پشت پراکسی Caddy متعلق به EasyRunner
    - انتخاب حجم‌های ذخیره‌سازی پایدار و احراز هویت برای Gateway میزبانی‌شده
summary: Gatewayِ OpenClaw را در EasyRunner با Podman و Caddy اجرا کنید
title: EasyRunner
x-i18n:
    generated_at: "2026-07-27T16:43:39Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 80cbde016a8bf7662d4b4a056a3d122a423264179daf70b5705e8f10b0dad5cb
    source_path: platforms/easyrunner.md
    workflow: 16
---

EasyRunner، ‏Gateway‏ متعلق به OpenClaw را به‌صورت یک برنامه کوچک کانتینری پشت
پروکسی Caddy خود میزبانی می‌کند. این راهنما یک میزبان EasyRunner را فرض می‌کند که برنامه‌های
Compose سازگار با Podman را اجرا می‌کند و HTTPS را از طریق Caddy خاتمه می‌دهد.

## پیش از شروع

- یک سرور EasyRunner با دامنه‌ای که به آن هدایت شده است.
- ایمیج رسمی OpenClaw‏ (`ghcr.io/openclaw/openclaw`) یا بیلد خودتان.
- یک والیوم پایدار پیکربندی برای `/home/node/.openclaw`.
- یک والیوم پایدار فضای کاری برای `/home/node/.openclaw/workspace`.
- یک توکن یا گذرواژه قوی برای Gateway.

در صورت امکان، احراز هویت دستگاه را فعال نگه دارید. اگر پروکسی معکوس نمی‌تواند
هویت دستگاه را به‌درستی منتقل کند، ابتدا تنظیمات پروکسی مورداعتماد را اصلاح کنید (به
[احراز هویت پروکسی مورداعتماد](/fa/gateway/trusted-proxy-auth) مراجعه کنید)؛ تنها در یک شبکه کاملاً خصوصی و تحت کنترل
اپراتور از روش‌های دور زدن خطرناک احراز هویت استفاده کنید.

## برنامه Compose

یک برنامه EasyRunner با فایل Compose به‌شکل زیر ایجاد کنید:

```yaml
services:
  openclaw:
    image: ghcr.io/openclaw/openclaw:latest
    restart: unless-stopped
    environment:
      OPENCLAW_GATEWAY_TOKEN: ${OPENCLAW_GATEWAY_TOKEN}
      OPENCLAW_HOME: /home/node
      OPENCLAW_STATE_DIR: /home/node/.openclaw
      OPENCLAW_CONFIG_PATH: /home/node/.openclaw/openclaw.json
      OPENCLAW_WORKSPACE_DIR: /home/node/.openclaw/workspace
    volumes:
      - openclaw-config:/home/node/.openclaw
      - openclaw-workspace:/home/node/.openclaw/workspace
    labels:
      caddy: openclaw.example.com
      caddy.reverse_proxy: "{{upstreams 1455}}"
    command: ["node", "openclaw.mjs", "gateway", "--bind", "lan", "--port", "1455"]

volumes:
  openclaw-config:
  openclaw-workspace:
```

مقدار `openclaw.example.com` را با نام میزبان Gateway خود جایگزین کنید. به‌جای
ثبت `OPENCLAW_GATEWAY_TOKEN` در تعریف برنامه، آن را در مدیر رازها/محیط EasyRunner
ذخیره کنید. ایمیج به‌طور پیش‌فرض به رابط loopback متصل می‌شود،
بنابراین `--bind lan --port 1455` صریح در `command` لازم است تا Caddy بتواند
به کانتینر دسترسی پیدا کند.

## پیکربندی OpenClaw

درون والیوم پایدار پیکربندی، Gateway را فقط از طریق
پروکسی قابل‌دسترسی نگه دارید و احراز هویت را الزامی کنید:

```json5
{
  gateway: {
    bind: "lan",
    port: 1455,
    auth: {
      token: "${OPENCLAW_GATEWAY_TOKEN}",
    },
  },
}
```

اگر Caddy اتصال TLS را برای Gateway خاتمه می‌دهد، به‌جای غیرفعال‌کردن سراسری بررسی‌های
احراز هویت، تنظیمات پروکسی مورداعتماد را برای مسیر دقیق پروکسی پیکربندی کنید. به
[احراز هویت پروکسی مورداعتماد](/fa/gateway/trusted-proxy-auth) مراجعه کنید.

## تأیید

از ایستگاه کاری خود:

```bash
openclaw gateway probe --url https://openclaw.example.com --token <token>
openclaw gateway status --url https://openclaw.example.com --token <token>
```

از میزبان EasyRunner، ‏`GET /healthz` (زنده‌بودن) و `GET /readyz`
(آمادگی) به احراز هویت نیاز ندارند و مبنای بررسی سلامت داخلی کانتینر
در ایمیج هستند. همچنین گزارش‌های برنامه را برای وجود یک Gateway در حال گوش‌دادن و نبود خطاهای هنگام راه‌اندازی
مربوط به SecretRef، ‏plugin یا احراز هویت کانال بررسی کنید.

## به‌روزرسانی‌ها و پشتیبان‌گیری

- ایمیج جدید OpenClaw را دریافت یا بیلد کنید، سپس برنامه EasyRunner را دوباره مستقر کنید.
- پیش از به‌روزرسانی‌ها از والیوم `openclaw-config` پشتیبان بگیرید. این والیوم
  شامل `openclaw.json`، ‏`agents/<agentId>/agent/auth-profiles.json` و وضعیت بسته‌های
  plugin نصب‌شده است.
- اگر عامل‌ها داده‌های پایدار پروژه را در `openclaw-workspace` می‌نویسند، از آن پشتیبان بگیرید.
- پس از به‌روزرسانی‌های عمده، `openclaw doctor` را اجرا کنید تا مهاجرت‌های پیکربندی و
  هشدارهای سرویس شناسایی شوند.

## عیب‌یابی

- `gateway probe` نمی‌تواند متصل شود: تأیید کنید که نام میزبان Caddy به برنامه اشاره می‌کند
  و کانتینر روی `0.0.0.0:1455` گوش می‌دهد.
- احراز هویت ناموفق است: توکن را هم‌زمان در رازهای EasyRunner و فرمان کلاینت محلی
  تعویض کنید.
- پس از بازیابی، فایل‌ها متعلق به root هستند: ایمیج با کاربر `node` (uid 1000) اجرا می‌شود؛
  والیوم‌های متصل‌شده را اصلاح کنید تا آن کاربر بتواند در
  `/home/node/.openclaw` و `/home/node/.openclaw/workspace` بنویسد.
- pluginهای مرورگر یا کانال ناموفق هستند: بررسی کنید که باینری‌های خارجی
  موردنیاز، خروجی شبکه و اعتبارنامه‌های متصل‌شده درون
  کانتینر در دسترس باشند.
