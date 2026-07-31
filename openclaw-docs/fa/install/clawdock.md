---
read_when:
    - شما اغلب OpenClaw را با Docker اجرا می‌کنید و برای کارهای روزمره فرمان‌های کوتاه‌تری می‌خواهید
    - یک لایهٔ کمکی برای داشبورد، گزارش‌ها، راه‌اندازی توکن و فرایندهای جفت‌سازی می‌خواهید
summary: توابع کمکی پوستهٔ ClawDock برای نصب‌های مبتنی بر Dockerِ OpenClaw
title: ClawDock
x-i18n:
    generated_at: "2026-07-27T15:20:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5bb829a3301178503f910931e86a39f7befeaf186044f4088a25dc80ea99130d
    source_path: install/clawdock.md
    workflow: 16
---

ClawDock یک لایهٔ کوچک از ابزارهای کمکی پوسته برای نصب‌های مبتنی بر Docker در OpenClaw است.

این لایه به‌جای فراخوانی‌های طولانی‌تر `docker compose ...`، فرمان‌های کوتاهی مانند `clawdock-start`، `clawdock-dashboard` و `clawdock-fix-token` در اختیارتان می‌گذارد.

اگر هنوز Docker را راه‌اندازی نکرده‌اید، از [Docker](/fa/install/docker) شروع کنید.

## نصب

```bash
mkdir -p ~/.clawdock && curl -sL https://raw.githubusercontent.com/openclaw/openclaw/main/scripts/clawdock/clawdock-helpers.sh -o ~/.clawdock/clawdock-helpers.sh
echo 'source ~/.clawdock/clawdock-helpers.sh' >> ~/.zshrc && source ~/.zshrc
```

اگر پیش‌تر ClawDock را از `scripts/shell-helpers/clawdock-helpers.sh` نصب کرده‌اید، آن را از مسیر فعلی `scripts/clawdock/clawdock-helpers.sh` دوباره نصب کنید؛ مسیر خام قدیمی GitHub حذف شده است.

ابزارهای کمکی در نخستین استفاده، محل نسخهٔ دریافت‌شدهٔ OpenClaw را به‌طور خودکار شناسایی می‌کنند (با بررسی مسیرهای رایجی مانند `~/openclaw` و `~/projects/openclaw`) و نتیجه را در `~/.clawdock/config` ذخیره می‌کنند. اگر نسخهٔ دریافت‌شدهٔ شما در جای دیگری قرار دارد، `CLAWDOCK_DIR` را خودتان تنظیم کنید.

## امکانات

### عملیات پایه

| فرمان              | توضیحات                 |
| ------------------ | ----------------------- |
| `clawdock-start` | راه‌اندازی Gateway      |
| `clawdock-stop` | توقف Gateway            |
| `clawdock-restart` | راه‌اندازی مجدد Gateway |
| `clawdock-status` | بررسی وضعیت کانتینر     |
| `clawdock-logs` | دنبال‌کردن لاگ‌های Gateway |

### دسترسی به کانتینر

| فرمان                    | توضیحات                                         |
| ------------------------ | ----------------------------------------------- |
| `clawdock-shell`       | بازکردن پوسته‌ای درون کانتینر Gateway           |
| `clawdock-cli <command>`       | اجرای فرمان‌های CLI مربوط به OpenClaw در Docker |
| `clawdock-exec <command>`       | اجرای یک فرمان دلخواه در کانتینر                |

### رابط کاربری وب و جفت‌سازی

| فرمان                    | توضیحات                           |
| ------------------------ | --------------------------------- |
| `clawdock-dashboard`       | بازکردن نشانی رابط کاربری کنترل   |
| `clawdock-devices`       | فهرست‌کردن جفت‌سازی‌های معلق دستگاه |
| `clawdock-approve <id>`       | تأیید درخواست جفت‌سازی            |

### راه‌اندازی و نگه‌داری

| فرمان                    | توضیحات                                           |
| ------------------------ | ------------------------------------------------- |
| `clawdock-fix-token`       | نوشتن توکن Gateway در پیکربندی کانتینر            |
| `clawdock-update`       | دریافت، ساخت مجدد و راه‌اندازی مجدد               |
| `clawdock-rebuild`       | فقط ساخت مجدد تصویر Docker                        |
| `clawdock-clean`       | حذف کانتینرها و حجم‌ها                            |

### ابزارها

| فرمان                    | توضیحات                                      |
| ------------------------ | -------------------------------------------- |
| `clawdock-health`       | اجرای بررسی سلامت Gateway                    |
| `clawdock-token`       | نمایش توکن Gateway                           |
| `clawdock-cd`       | رفتن به پوشهٔ پروژهٔ OpenClaw                |
| `clawdock-config`       | بازکردن `~/.openclaw`                   |
| `clawdock-show-config`       | نمایش فایل‌های پیکربندی با مقادیر پوشانده‌شده |
| `clawdock-workspace`       | بازکردن پوشهٔ فضای کاری                      |
| `clawdock-help`       | فهرست‌کردن همهٔ فرمان‌های ClawDock           |

## روند نخستین استفاده

```bash
clawdock-start
clawdock-fix-token
clawdock-dashboard
```

اگر مرورگر اعلام کرد که جفت‌سازی لازم است:

```bash
clawdock-devices
clawdock-approve <request-id>
```

## پیکربندی و اسرار

ClawDock دو فایل جداگانهٔ `.env` را مطابق تفکیک شرح‌داده‌شده در [Docker](/fa/install/docker) می‌خواند:

- فایل پروژهٔ `.env` در کنار `docker-compose.yml`: مقادیر ویژهٔ Docker مانند نام تصویر، درگاه‌ها و `OPENCLAW_GATEWAY_TOKEN`. ‏`clawdock-token` توکن را از اینجا می‌خواند.
- `~/.openclaw/.env` (که در کانتینر سوار می‌شود): اسرار مبتنی بر متغیرهای محیطی که خود OpenClaw در کنار `openclaw.json` و `agents/<agentId>/agent/auth-profiles.json` مدیریت می‌کند.

`clawdock-fix-token` توکن را از فایل پروژهٔ `.env` در مقادیر پیکربندی `gateway.remote.token` و `gateway.auth.token` کانتینر کپی می‌کند و Gateway را دوباره راه‌اندازی می‌کند.

برای بررسی سریع `openclaw.json` و هر دو فایل `.env` از `clawdock-show-config` استفاده کنید؛ این فرمان مقادیر `.env` را در خروجی چاپ‌شده می‌پوشاند.

## مرتبط

<CardGroup cols={2}>
  <Card title="Docker" href="/fa/install/docker" icon="docker">
    نصب مرجع Docker برای OpenClaw.
  </Card>
  <Card title="محیط اجرای ماشین مجازی Docker" href="/fa/install/docker-vm-runtime" icon="cube">
    محیط اجرای ماشین مجازی تحت مدیریت Docker برای جداسازی مقاوم‌تر.
  </Card>
  <Card title="به‌روزرسانی" href="/fa/install/updating" icon="arrow-up-right-from-square">
    به‌روزرسانی بستهٔ OpenClaw و سرویس‌های مدیریت‌شده.
  </Card>
</CardGroup>
