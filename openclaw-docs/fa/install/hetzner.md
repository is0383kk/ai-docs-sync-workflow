---
read_when:
    - می‌خواهید OpenClaw به‌صورت ۲۴/۷ روی یک VPS ابری (نه لپ‌تاپتان) اجرا شود
    - یک Gateway همیشه‌فعال و مناسب محیط عملیاتی روی VPS خود می‌خواهید
    - می‌خواهید کنترل کاملی بر ماندگاری، فایل‌های اجرایی و رفتار راه‌اندازی مجدد داشته باشید
    - شما OpenClaw را در Docker روی Hetzner یا ارائه‌دهنده‌ای مشابه اجرا می‌کنید
summary: اجرای ۲۴/۷ Gateway ‏OpenClaw روی یک VPS ارزان Hetzner ‏(Docker) با وضعیت پایدار و باینری‌های ازپیش‌تعبیه‌شده
title: Hetzner
x-i18n:
    generated_at: "2026-07-27T15:37:05Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8ffebc0ce725fd219d13d0a556940327e70dab810b8fbee0b365c4870dc7109b
    source_path: install/hetzner.md
    workflow: 16
---

یک Gateway پایدار OpenClaw را با استفاده از Docker روی یک VPS از Hetzner اجرا کنید؛ به‌گونه‌ای که وضعیت ماندگار، فایل‌های اجرایی تعبیه‌شده و رفتار راه‌اندازی مجدد ایمن داشته باشد.

قیمت‌گذاری Hetzner تغییر می‌کند؛ کوچک‌ترین VPS مبتنی بر Debian/Ubuntu را که پاسخ‌گوی نیازهاست انتخاب کنید و در صورت بروز خطاهای OOM، منابع آن را افزایش دهید.

از طریق انتقال پورت SSH از لپ‌تاپ می‌توان به Gateway دسترسی داشت؛ همچنین اگر دیوار آتش و توکن‌ها را خودتان مدیریت می‌کنید، می‌توانید پورت را مستقیماً در دسترس قرار دهید.

یادآوری مدل امنیتی:

- عامل‌های مشترک سازمانی زمانی مناسب‌اند که همه در یک مرز اعتماد یکسان باشند و محیط اجرا فقط برای امور سازمانی استفاده شود.
- جداسازی سخت‌گیرانه را حفظ کنید: VPS/محیط اجرای اختصاصی + حساب‌های اختصاصی؛ هیچ نمایه شخصی Apple/Google/مرورگر/مدیر گذرواژه‌ای روی آن میزبان قرار ندهید.
- اگر کاربران نسبت به یکدیگر متخاصم‌اند، آن‌ها را بر اساس Gateway/میزبان/کاربر سیستم‌عامل تفکیک کنید.

[امنیت](/fa/gateway/security) و [میزبانی VPS](/fa/vps) را ببینید.

این راهنما Ubuntu یا Debian روی Hetzner را فرض می‌کند. در یک VPS لینوکسی دیگر، بسته‌های معادل را نصب کنید. برای روند عمومی Docker، [Docker](/fa/install/docker) را ببینید.

## موارد موردنیاز

- VPS از Hetzner با دسترسی root
- دسترسی SSH از لپ‌تاپ
- Docker و Docker Compose
- اعتبارنامه‌های احراز هویت مدل
- اعتبارنامه‌های اختیاری ارائه‌دهنده (کد QR برای WhatsApp، توکن ربات Telegram، OAuth برای Gmail)
- حدود 20 دقیقه

## مسیر سریع

1. راه‌اندازی VPS در Hetzner
2. نصب Docker
3. همانندسازی مخزن OpenClaw
4. ایجاد پوشه‌های ماندگار روی میزبان
5. پیکربندی `.env` و `docker-compose.yml`
6. تعبیه فایل‌های اجرایی موردنیاز در ایمیج
7. `docker compose up -d`
8. تأیید ماندگاری و دسترسی به Gateway

<Steps>
  <Step title="راه‌اندازی VPS">
    یک VPS مبتنی بر Ubuntu یا Debian در Hetzner ایجاد کنید، سپس به‌عنوان root متصل شوید:

    ```bash
    ssh root@YOUR_VPS_IP
    ```

    با VPS مانند زیرساخت دارای وضعیت رفتار کنید، نه زیرساختی یک‌بارمصرف.

  </Step>

  <Step title="نصب Docker (روی VPS)">
    ```bash
    apt-get update
    apt-get install -y git curl ca-certificates
    curl -fsSL https://get.docker.com | sh
    ```

    بررسی کنید:

    ```bash
    docker --version
    docker compose version
    ```

  </Step>

  <Step title="همانندسازی مخزن OpenClaw">
    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    ```

    این راهنما یک ایمیج سفارشی می‌سازد تا فایل‌های اجرایی تعبیه‌شده پس از راه‌اندازی مجدد باقی بمانند.

  </Step>

  <Step title="ایجاد پوشه‌های ماندگار روی میزبان">
    کانتینرهای Docker زودگذرند؛ تمام وضعیت‌های بلندمدت باید روی میزبان نگه‌داری شوند.

    ```bash
    mkdir -p /root/.openclaw/workspace

    # مالکیت را روی کاربر کانتینر تنظیم کنید (uid 1000):
    chown -R 1000:1000 /root/.openclaw
    ```

  </Step>

  <Step title="پیکربندی متغیرهای محیطی">
    فایل `.env` را در ریشه مخزن ایجاد کنید:

    ```bash
    OPENCLAW_IMAGE=openclaw:latest
    OPENCLAW_GATEWAY_TOKEN=
    OPENCLAW_GATEWAY_BIND=lan
    OPENCLAW_GATEWAY_PORT=18789

    OPENCLAW_CONFIG_DIR=/root/.openclaw
    OPENCLAW_WORKSPACE_DIR=/root/.openclaw/workspace

    GOG_KEYRING_PASSWORD=
    XDG_CONFIG_HOME=/home/node/.openclaw
    ```

    برای مدیریت توکن پایدار Gateway از طریق
    `.env`، مقدار `OPENCLAW_GATEWAY_TOKEN` را تنظیم کنید؛ در غیر این صورت، پیش از اتکا به کلاینت‌ها
    در راه‌اندازی‌های مجدد، `gateway.auth.token` را پیکربندی کنید. اگر هیچ‌کدام تنظیم نشده باشند، OpenClaw برای
    همان راه‌اندازی از یک توکن مختص محیط اجرا استفاده می‌کند. برای `GOG_KEYRING_PASSWORD` یک گذرواژه دسته‌کلید ایجاد کنید:

    ```bash
    openssl rand -hex 32
    ```

    **این فایل را commit نکنید.** این فایل شامل متغیرهای محیطی کانتینر/محیط اجرا مانند
    `OPENCLAW_GATEWAY_TOKEN` است. احراز هویت ذخیره‌شده OAuth/کلید API ارائه‌دهنده در
    `~/.openclaw/agents/<agentId>/agent/auth-profiles.json` متصل‌شده قرار دارد.

  </Step>

  <Step title="پیکربندی Docker Compose">
    فایل `docker-compose.yml` را ایجاد یا به‌روزرسانی کنید:

    ```yaml
    services:
      openclaw-gateway:
        image: ${OPENCLAW_IMAGE}
        build: .
        restart: unless-stopped
        env_file:
          - .env
        environment:
          - HOME=/home/node
          - NODE_ENV=production
          - TERM=xterm-256color
          - OPENCLAW_GATEWAY_BIND=${OPENCLAW_GATEWAY_BIND}
          - OPENCLAW_GATEWAY_PORT=${OPENCLAW_GATEWAY_PORT}
          - OPENCLAW_GATEWAY_TOKEN=${OPENCLAW_GATEWAY_TOKEN}
          - GOG_KEYRING_PASSWORD=${GOG_KEYRING_PASSWORD}
          - XDG_CONFIG_HOME=${XDG_CONFIG_HOME}
          - PATH=/home/linuxbrew/.linuxbrew/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
        volumes:
          - ${OPENCLAW_CONFIG_DIR}:/home/node/.openclaw
          - ${OPENCLAW_WORKSPACE_DIR}:/home/node/.openclaw/workspace
        ports:
          # توصیه می‌شود: Gateway را روی VPS فقط به loopback محدود کنید؛ از طریق تونل SSH دسترسی داشته باشید.
          # برای در دسترس قراردادن عمومی، پیشوند `127.0.0.1:` را حذف و دیوار آتش را متناسب با آن پیکربندی کنید.
          - "127.0.0.1:${OPENCLAW_GATEWAY_PORT}:18789"
        command:
          [
            "node",
            "dist/index.js",
            "gateway",
            "--bind",
            "${OPENCLAW_GATEWAY_BIND}",
            "--port",
            "${OPENCLAW_GATEWAY_PORT}",
            "--allow-unconfigured",
          ]
    ```

    `--allow-unconfigured` فقط برای سهولت راه‌اندازی اولیه است و جایگزین پیکربندی واقعی Gateway نیست. همچنان احراز هویت (`gateway.auth.token` یا گذرواژه) و حالت اتصال ایمن را برای استقرار خود تنظیم کنید.

  </Step>

  <Step title="مراحل مشترک محیط اجرای ماشین مجازی Docker">
    برای روند متداول میزبان Docker، راهنمای مشترک محیط اجرا را دنبال کنید:

    - [تعبیه فایل‌های اجرایی موردنیاز در ایمیج](/fa/install/docker-vm-runtime#bake-required-binaries-into-the-image)
    - [ساخت و اجرا](/fa/install/docker-vm-runtime#build-and-launch)
    - [موارد ماندگار و محل آن‌ها](/fa/install/docker-vm-runtime#what-persists-where)
    - [به‌روزرسانی‌ها](/fa/install/docker-vm-runtime#updates)

  </Step>

  <Step title="دسترسی ویژه Hetzner">
    پس از مراحل مشترک ساخت و اجرا، تونل را باز کنید.

    **پیش‌نیاز:** مطمئن شوید پیکربندی sshd در VPS اجازه انتقال TCP را می‌دهد. اگر
    پیکربندی SSH را سخت‌گیرانه کرده‌اید، `/etc/ssh/sshd_config` را بررسی و این مقدار را تنظیم کنید:

    ```text
    AllowTcpForwarding local
    ```

    `local` انتقال‌های محلی `ssh -L` از لپ‌تاپ را مجاز و هم‌زمان
    انتقال‌های راه‌دور از سرور را مسدود می‌کند. تنظیم آن روی `no` باعث شکست تونل با این پیام می‌شود:
    `channel 3: open failed: administratively prohibited: open failed`

    پس از تأیید فعال‌بودن انتقال TCP، سرویس SSH را
    (`systemctl restart ssh`) مجدداً راه‌اندازی کنید و تونل را از لپ‌تاپ اجرا کنید:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 root@YOUR_VPS_IP
    ```

    `http://127.0.0.1:18789/` را باز و راز مشترک پیکربندی‌شده را جای‌گذاری کنید.
    این راهنما به‌طور پیش‌فرض از توکن Gateway استفاده می‌کند؛ اگر احراز هویت را به روش گذرواژه تغییر داده‌اید،
    به‌جای آن از گذرواژه پیکربندی‌شده استفاده کنید.

  </Step>
</Steps>

نقشه مشترک ماندگاری در [محیط اجرای ماشین مجازی Docker](/fa/install/docker-vm-runtime#what-persists-where) قرار دارد.

## زیرساخت به‌عنوان کد (Terraform)

برای تیم‌هایی که گردش‌کارهای زیرساخت به‌عنوان کد را ترجیح می‌دهند، یک پیکربندی Terraform نگه‌داری‌شده توسط جامعه این موارد را ارائه می‌دهد:

- پیکربندی ماژولار Terraform با مدیریت وضعیت راه‌دور
- راه‌اندازی خودکار از طریق cloud-init
- اسکریپت‌های استقرار (راه‌اندازی اولیه، استقرار، پشتیبان‌گیری/بازیابی)
- سخت‌سازی امنیتی (دیوار آتش، UFW، دسترسی فقط از طریق SSH)
- پیکربندی تونل SSH برای دسترسی به Gateway

**مخزن‌ها:**

- زیرساخت: [openclaw-terraform-hetzner](https://github.com/andreesg/openclaw-terraform-hetzner)
- پیکربندی Docker: [openclaw-docker-config](https://github.com/andreesg/openclaw-docker-config)

این رویکرد با افزودن استقرارهای بازتولیدپذیر، زیرساخت دارای کنترل نسخه و بازیابی خودکار پس از فاجعه، پیکربندی Docker بالا را تکمیل می‌کند.

<Note>
توسط جامعه نگه‌داری می‌شود. برای گزارش مشکلات یا مشارکت، پیوندهای مخزن بالا را ببینید.
</Note>

## مراحل بعدی

- راه‌اندازی کانال‌های پیام‌رسانی: [کانال‌ها](/fa/channels)
- پیکربندی Gateway: [پیکربندی Gateway](/fa/gateway/configuration)
- به‌روز نگه‌داشتن OpenClaw: [به‌روزرسانی](/fa/install/updating)

## مرتبط

- [نمای کلی نصب](/fa/install)
- [Fly.io](/fa/install/fly)
- [Docker](/fa/install/docker)
- [میزبانی VPS](/fa/vps)
