---
read_when:
    - نصب‌های تکرارپذیر و قابل بازگردانی می‌خواهید
    - شما از قبل از Nix/NixOS/Home Manager استفاده می‌کنید
    - می‌خواهید همه‌چیز به‌صورت اعلانی تثبیت و مدیریت شود
summary: نصب اعلانی OpenClaw با Nix
title: Nix
x-i18n:
    generated_at: "2026-07-27T16:41:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6f74e259ec3d909c73d9184db24d236135db04c29c2e7fab9be9e6fa7f98ba91
    source_path: install/nix.md
    workflow: 16
---

OpenClaw را به‌صورت اعلانی با **[nix-openclaw](https://github.com/openclaw/nix-openclaw)**، ماژول رسمی و کامل Home Manager، نصب کنید.

<Info>
مخزن [nix-openclaw](https://github.com/openclaw/nix-openclaw) مرجع اصلی نصب Nix است. این صفحه مروری سریع ارائه می‌دهد.
</Info>

## چه امکاناتی دریافت می‌کنید

- Gateway + برنامه macOS + ابزارها (whisper، spotify، دوربین‌ها)، همگی با نسخه‌های ثابت
- سرویس Launchd که پس از راه‌اندازی مجدد نیز فعال می‌ماند
- سامانه Plugin با پیکربندی اعلانی
- بازگردانی فوری: `home-manager switch --rollback`

## شروع سریع

<Steps>
  <Step title="نصب Determinate Nix">
    اگر Nix از قبل نصب نشده است، دستورالعمل‌های [نصب‌کننده Determinate Nix](https://github.com/DeterminateSystems/nix-installer) را دنبال کنید.
  </Step>
  <Step title="ایجاد یک flake محلی">
    از قالب agent-first در مخزن nix-openclaw استفاده کنید:
    ```bash
    mkdir -p ~/code/openclaw-local
    # فایل templates/agent-first/flake.nix را از مخزن nix-openclaw کپی کنید
    ```
  </Step>
  <Step title="پیکربندی اسرار">
    توکن ربات پیام‌رسان و کلید API ارائه‌دهنده مدل را تنظیم کنید. فایل‌های متنی ساده در `~/.secrets/` به‌خوبی کار می‌کنند.
  </Step>
  <Step title="تکمیل جای‌نگهدارهای قالب و اعمال پیکربندی">
    ```bash
    home-manager switch
    ```
  </Step>
  <Step title="تأیید">
    اطمینان حاصل کنید که سرویس launchd در حال اجرا است و ربات به پیام‌ها پاسخ می‌دهد.
  </Step>
</Steps>

برای مشاهده تمام گزینه‌های ماژول و نمونه‌ها، به [README مربوط به nix-openclaw](https://github.com/openclaw/nix-openclaw) مراجعه کنید.

## رفتار زمان اجرا در حالت Nix

وقتی `OPENCLAW_NIX_MODE=1` تنظیم شده باشد (در nix-openclaw به‌صورت خودکار)، OpenClaw برای نصب‌های مدیریت‌شده با Nix وارد حالت قطعی می‌شود. سایر بسته‌های Nix نیز می‌توانند همین حالت را تنظیم کنند؛ nix-openclaw مرجع رسمی است.

همچنین می‌توانید آن را به‌صورت دستی تنظیم کنید:

```bash
export OPENCLAW_NIX_MODE=1
```

در macOS، برنامه رابط گرافیکی متغیرهای محیطی پوسته را به ارث نمی‌برد. به‌جای آن، حالت Nix را از طریق `defaults` فعال کنید:

```bash
defaults write ai.openclaw.mac openclaw.nixMode -bool true
```

### تغییرات حالت Nix

- فرایندهای نصب خودکار و تغییر خودکار غیرفعال می‌شوند.
- `openclaw.json` تغییرناپذیر در نظر گرفته می‌شود. مقادیر پیش‌فرض مشتق‌شده هنگام راه‌اندازی فقط در زمان اجرا باقی می‌مانند و نویسنده‌های پیکربندی (راه‌اندازی، ورود اولیه، `openclaw update` تغییردهنده، نصب/به‌روزرسانی/حذف/فعال‌سازی Plugin، `doctor --fix`، `doctor --generate-gateway-token`، `openclaw config set`) از ویرایش فایل خودداری می‌کنند.
- به‌جای آن، منبع Nix را ویرایش کنید. برای nix-openclaw، از [شروع سریع](https://github.com/openclaw/nix-openclaw#quick-start) مبتنی بر agent-first استفاده کنید و پیکربندی را زیر `programs.openclaw.config` یا `instances.<name>.config` تنظیم کنید.
- وابستگی‌های مفقود، پیام‌های اصلاحی ویژه Nix را نمایش می‌دهند.
- رابط کاربری نوار حالت فقط‌خواندنی Nix را نمایش می‌دهد.

### مسیرهای پیکربندی و وضعیت

OpenClaw پیکربندی JSON5 را از `OPENCLAW_CONFIG_PATH` می‌خواند و داده‌های تغییرپذیر را در `OPENCLAW_STATE_DIR` ذخیره می‌کند. در Nix، این مسیرها را صریحاً روی مکان‌های مدیریت‌شده با Nix تنظیم کنید تا وضعیت زمان اجرا و پیکربندی خارج از مخزن تغییرناپذیر باقی بمانند.

| متغیر               | پیش‌فرض                                 |
| ---------------------- | --------------------------------------- |
| `OPENCLAW_HOME`        | `HOME` / `USERPROFILE` / `os.homedir()` |
| `OPENCLAW_STATE_DIR`   | `~/.openclaw`                           |
| `OPENCLAW_CONFIG_PATH` | `$OPENCLAW_STATE_DIR/openclaw.json`     |

### کشف PATH سرویس

سرویس Gateway در launchd/systemd فایل‌های اجرایی پروفایل Nix را به‌طور خودکار شناسایی می‌کند تا Pluginها و ابزارهایی که فایل‌های اجرایی نصب‌شده با `nix` را از طریق پوسته اجرا می‌کنند، بدون تنظیم دستی PATH کار کنند:

- وقتی `NIX_PROFILES` تنظیم شده باشد، هر ورودی با اولویت راست‌به‌چپ به PATH سرویس افزوده می‌شود (مطابق اولویت پوسته Nix: راست‌ترین مورد برنده است).
- وقتی `NIX_PROFILES` تنظیم نشده باشد، `~/.nix-profile/bin` به‌عنوان مسیر جایگزین افزوده می‌شود.

این رفتار برای محیط سرویس launchd در macOS و systemd در Linux اعمال می‌شود.

## مرتبط

<CardGroup cols={2}>
  <Card title="nix-openclaw" href="https://github.com/openclaw/nix-openclaw" icon="arrow-up-right-from-square">
    ماژول مرجع Home Manager و راهنمای کامل راه‌اندازی.
  </Card>
  <Card title="جادوگر راه‌اندازی" href="/fa/start/wizard" icon="wand-magic-sparkles">
    راهنمای گام‌به‌گام راه‌اندازی CLI بدون Nix.
  </Card>
  <Card title="Docker" href="/fa/install/docker" icon="docker">
    راه‌اندازی کانتینری به‌عنوان جایگزینی بدون Nix.
  </Card>
  <Card title="به‌روزرسانی" href="/fa/install/updating" icon="arrow-up-right-from-square">
    به‌روزرسانی نصب‌های مدیریت‌شده با Home Manager همراه با بسته.
  </Card>
</CardGroup>
