---
read_when:
    - راه‌اندازی OpenClaw در DigitalOcean
    - در جست‌وجوی یک VPS پولی ساده برای OpenClaw هستید؟
summary: میزبانی OpenClaw روی یک Droplet در DigitalOcean
title: DigitalOcean
x-i18n:
    generated_at: "2026-07-27T16:41:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e124a59c079efda0c8e880018f2657fad784af1489ca3f98ed8ab609249e35bd
    source_path: install/digitalocean.md
    workflow: 16
---

یک Gateway پایدار OpenClaw را روی یک Droplet از DigitalOcean اجرا کنید (حدود ۶ دلار در ماه برای طرح Basic با ۱ گیگابایت حافظه).

DigitalOcean مسیری ساده برای استفاده از VPS پولی است. برای گزینه‌های ارزان‌تر یا رایگان:

- [Hetzner](/fa/install/hetzner) -- هسته‌ها و RAM بیشتر به‌ازای هر دلار.
- [Oracle Cloud](/fa/install/oracle) -- سطح ARM همیشه‌رایگان (تا 4 OCPU و 24 GB RAM)، اما ثبت‌نام ممکن است دردسرساز باشد و فقط از ARM پشتیبانی می‌کند.

## پیش‌نیازها

- حساب DigitalOcean ([ثبت‌نام](https://cloud.digitalocean.com/registrations/new))
- جفت کلید SSH (یا آمادگی برای استفاده از احراز هویت با گذرواژه)
- حدود ۲۰ دقیقه

## راه‌اندازی

<Steps>
  <Step title="ایجاد یک Droplet">
    <Warning>
    از یک ایمیج پایه پاک (Ubuntu 24.04 LTS) استفاده کنید. از ایمیج‌های یک‌کلیکی شخص ثالث Marketplace خودداری کنید، مگر اینکه اسکریپت‌های راه‌اندازی و تنظیمات پیش‌فرض فایروال آن‌ها را بررسی کرده باشید.
    </Warning>

    1. وارد [DigitalOcean](https://cloud.digitalocean.com/) شوید.
    2. روی **Create > Droplets** کلیک کنید.
    3. موارد زیر را انتخاب کنید:
       - **Region:** نزدیک‌ترین گزینه به شما
       - **Image:** Ubuntu 24.04 LTS
       - **Size:** Basic، Regular، 1 vCPU / 1 GB RAM / 25 GB SSD
       - **Authentication:** کلید SSH (توصیه‌شده) یا گذرواژه
    4. روی **Create Droplet** کلیک کنید و نشانی IP را یادداشت کنید.

  </Step>

  <Step title="اتصال و نصب">
    ```bash
    ssh root@YOUR_DROPLET_IP

    apt update && apt upgrade -y

    # نصب Node.js 24
    curl -fsSL https://deb.nodesource.com/setup_24.x | bash -
    apt install -y nodejs

    # نصب OpenClaw
    curl -fsSL https://openclaw.ai/install.sh | bash

    # ایجاد کاربر غیر root که مالک وضعیت و سرویس‌های OpenClaw خواهد بود.
    adduser openclaw
    usermod -aG sudo openclaw
    loginctl enable-linger openclaw

    su - openclaw
    openclaw --version
    ```

    از پوسته root فقط برای راه‌اندازی اولیه سیستم استفاده کنید. فرمان‌های OpenClaw را با کاربر غیر root ‏`openclaw` اجرا کنید تا وضعیت در مسیر `/home/openclaw/.openclaw/` نگهداری شود و Gateway به‌عنوان سرویس systemd ‏`--user` آن کاربر نصب شود.

  </Step>

  <Step title="اجرای فرایند آغاز به کار">
    ```bash
    openclaw onboard --install-daemon
    ```

    راهنما شما را در مراحل احراز هویت مدل، راه‌اندازی کانال، تولید توکن Gateway و نصب daemon (سرویس کاربر systemd) همراهی می‌کند.

  </Step>

  <Step title="افزودن swap (توصیه‌شده برای Dropletهای ۱ گیگابایتی)">
    ```bash
    fallocate -l 2G /swapfile
    chmod 600 /swapfile
    mkswap /swapfile
    swapon /swapfile
    echo '/swapfile none swap sw 0 0' >> /etc/fstab
    ```
  </Step>

  <Step title="بررسی Gateway">
    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```
  </Step>

  <Step title="دسترسی به رابط کنترل">
    Gateway به‌طور پیش‌فرض به loopback متصل می‌شود. یکی از گزینه‌های زیر را انتخاب کنید.

    **گزینه A: تونل SSH (ساده‌ترین)**

    ```bash
    # از دستگاه محلی خود
    ssh -L 18789:localhost:18789 root@YOUR_DROPLET_IP
    ```

    سپس `http://localhost:18789` را باز کنید.

    **گزینه B: Tailscale Serve**

    ```bash
    curl -fsSL https://tailscale.com/install.sh | sudo sh
    sudo tailscale up
    openclaw config set gateway.tailscale.mode serve
    openclaw gateway restart
    ```

    سپس `https://<magicdns>/` را از هر دستگاهی در tailnet خود باز کنید.

    Tailscale Serve ترافیک رابط کنترل و WebSocket را از طریق سرآیندهای هویت tailnet احراز هویت می‌کند؛ این روش فرض می‌کند خود میزبان Gateway قابل اعتماد است. نقطه‌های پایانی HTTP API صرف‌نظر از این موضوع، همچنان از حالت عادی احراز هویت Gateway (توکن/گذرواژه) پیروی می‌کنند. برای الزامی‌کردن اعتبارنامه‌های صریحِ راز مشترک روی Serve، مقدار `gateway.auth.allowTailscale: false` را تنظیم کنید و از `gateway.auth.mode: "token"` یا `"password"` استفاده کنید.

    **گزینه C: اتصال به Tailnet (بدون Serve)**

    ```bash
    openclaw config set gateway.bind tailnet
    openclaw gateway restart
    ```

    سپس `http://<tailscale-ip>:18789` را باز کنید (توکن الزامی است).

  </Step>
</Steps>

## ماندگاری و پشتیبان‌گیری

وضعیت OpenClaw در مسیرهای زیر نگهداری می‌شود:

- `~/.openclaw/` -- ‏`openclaw.json`، اعتبارنامه‌های کانال/ارائه‌دهنده، ‏`auth-profiles.json` هر عامل و داده‌های نشست.
- `~/.openclaw/workspace/` -- فضای کاری عامل (SOUL.md، حافظه و مصنوعات).

این داده‌ها پس از راه‌اندازی مجدد Droplet باقی می‌مانند. برای تهیه یک اسنپ‌شات قابل‌انتقال:

```bash
openclaw backup create
```

اسنپ‌شات‌های DigitalOcean از کل Droplet پشتیبان می‌گیرند؛ `openclaw backup create` میان میزبان‌ها قابل‌انتقال است.

## نکته‌های RAM یک گیگابایتی

Droplet شش‌دلاری فقط 1 GB RAM دارد. برای عملکرد روان:

- مطمئن شوید مرحله swap بالا در `/etc/fstab` ثبت شده است تا پس از راه‌اندازی مجدد باقی بماند.
- مدل‌های مبتنی بر API ‏(Claude، GPT) را به مدل‌های محلی ترجیح دهید -- استنتاج LLM محلی در 1 GB جا نمی‌گیرد.
- اگر در اعلان‌های بزرگ با خطای OOM مواجه شدید، `agents.defaults.model.primary` را روی مدلی کوچک‌تر تنظیم کنید.
- با `free -h` و `htop` نظارت کنید.

## عیب‌یابی

**Gateway راه‌اندازی نمی‌شود** -- ‏`openclaw doctor --non-interactive` را اجرا کنید و گزارش‌ها را با `journalctl --user -u openclaw-gateway.service -n 50` بررسی کنید.

**درگاه از قبل در حال استفاده است** -- برای یافتن فرایند، `lsof -i :18789` را اجرا و سپس آن را متوقف کنید.

**کمبود حافظه** -- با `free -h` فعال‌بودن swap را بررسی کنید. اگر همچنان با OOM مواجه می‌شوید، به‌جای مدل‌های محلی از مدل‌های مبتنی بر API ‏(Claude، GPT) استفاده کنید یا Droplet را به نسخه 2 GB ارتقا دهید.

## مراحل بعدی

- [کانال‌ها](/fa/channels) -- اتصال Telegram، WhatsApp، Discord و موارد دیگر
- [پیکربندی Gateway](/fa/gateway/configuration) -- همه گزینه‌های پیکربندی
- [به‌روزرسانی](/fa/install/updating) -- به‌روز نگه‌داشتن OpenClaw

## مرتبط

- [نمای کلی نصب](/fa/install)
- [Fly.io](/fa/install/fly)
- [Hetzner](/fa/install/hetzner)
- [میزبانی VPS](/fa/vps)
