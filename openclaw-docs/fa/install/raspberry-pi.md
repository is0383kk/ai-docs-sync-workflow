---
read_when:
    - راه‌اندازی OpenClaw روی Raspberry Pi
    - اجرای OpenClaw روی دستگاه‌های ARM
    - ساخت یک هوش مصنوعی شخصی ارزان و همیشه‌فعال
summary: میزبانی OpenClaw روی Raspberry Pi برای میزبانی شخصی همیشگی
title: Raspberry Pi
x-i18n:
    generated_at: "2026-07-27T16:42:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 60f8f3b23577155658d410993937ebe7c34c21f71c1bd7d9b0c453f15c4aa024
    source_path: install/raspberry-pi.md
    workflow: 16
---

یک Gateway دائمی و همیشه‌روشن OpenClaw را روی Raspberry Pi اجرا کنید. از آنجا که Pi فقط نقش Gateway را دارد (مدل‌ها از طریق API در فضای ابری اجرا می‌شوند)، حتی یک Pi معمولی نیز به‌خوبی از پس این بار کاری برمی‌آید — هزینهٔ معمول سخت‌افزار **$35-80 فقط برای یک‌بار** است و هزینهٔ ماهانه‌ای ندارد.

## سازگاری سخت‌افزاری

| مدل Pi      | RAM    | قابل استفاده؟ | توضیحات                                  |
| ----------- | ------ | ------------- | ---------------------------------------- |
| Pi 5        | 4/8 GB | بهترین       | سریع‌ترین گزینه؛ توصیه می‌شود.           |
| Pi 4        | 4 GB   | خوب           | گزینهٔ متعادل برای بیشتر کاربران.        |
| Pi 4        | 2 GB   | قابل قبول     | فضای swap اضافه کنید.                    |
| Pi 4        | 1 GB   | محدود         | با swap و پیکربندی حداقلی امکان‌پذیر است. |
| Pi 3B+      | 1 GB   | کند           | کار می‌کند، اما کند است.                 |
| Pi Zero 2 W | 512 MB | خیر           | توصیه نمی‌شود.                           |

**حداقل:** 1 GB RAM، یک هسته، 500 MB فضای آزاد دیسک و سیستم‌عامل 64 بیتی.
**توصیه‌شده:** 2 GB+ RAM، کارت SD با ظرفیت 16 GB+ (یا USB SSD) و Ethernet.

## پیش‌نیازها

- Raspberry Pi 4 یا 5 با 2 GB+ RAM (مقدار 4 GB توصیه می‌شود)
- کارت MicroSD با ظرفیت 16 GB+ یا USB SSD (برای عملکرد بهتر)
- منبع تغذیهٔ رسمی Pi
- اتصال شبکه (Ethernet یا WiFi)
- Raspberry Pi OS نسخهٔ 64 بیتی (الزامی — از نسخهٔ 32 بیتی استفاده نکنید)
- حدود 30 دقیقه زمان

## راه‌اندازی

<Steps>
  <Step title="نوشتن سیستم‌عامل روی حافظه">
    از **Raspberry Pi OS Lite (64-bit)** استفاده کنید — برای یک سرور بدون نمایشگر، نیازی به محیط دسکتاپ نیست.

    1. [Raspberry Pi Imager](https://www.raspberrypi.com/software/) را دانلود کنید.
    2. سیستم‌عامل را انتخاب کنید: **Raspberry Pi OS Lite (64-bit)**.
    3. در پنجرهٔ تنظیمات، موارد زیر را از پیش پیکربندی کنید:
       - Hostname: `gateway-host`
       - Enable SSH
       - نام کاربری و گذرواژه را تعیین کنید
       - WiFi را پیکربندی کنید (اگر از Ethernet استفاده نمی‌کنید)
    4. سیستم‌عامل را روی کارت SD یا درایو USB بنویسید، آن را متصل کنید و Pi را راه‌اندازی کنید.

  </Step>

  <Step title="اتصال از طریق SSH">
    ```bash
    ssh user@gateway-host
    ```
  </Step>

  <Step title="به‌روزرسانی سیستم">
    ```bash
    sudo apt update && sudo apt upgrade -y
    sudo apt install -y git curl build-essential

    # تنظیم منطقهٔ زمانی (برای Cron و یادآوری‌ها مهم است)
    sudo timedatectl set-timezone America/Chicago
    ```

  </Step>

  <Step title="نصب Node.js 24">
    ```bash
    curl -fsSL https://deb.nodesource.com/setup_24.x | sudo -E bash -
    sudo apt install -y nodejs
    node --version
    ```
  </Step>

  <Step title="افزودن swap (برای 2 GB یا کمتر مهم است)">
    ```bash
    sudo fallocate -l 2G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab

    # کاهش میزان استفاده از swap برای دستگاه‌های کم‌حافظه
    echo 'vm.swappiness=10' | sudo tee -a /etc/sysctl.conf
    sudo sysctl -p
    ```

  </Step>

  <Step title="نصب OpenClaw">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    ```
  </Step>

  <Step title="اجرای فرایند آغازین">
    ```bash
    openclaw onboard --install-daemon
    ```

    مراحل راهنما را دنبال کنید. برای دستگاه‌های بدون نمایشگر، استفاده از کلیدهای API به‌جای OAuth توصیه می‌شود. Telegram ساده‌ترین کانال برای شروع است.

  </Step>

  <Step title="تأیید عملکرد">
    ```bash
    openclaw status
    systemctl --user status openclaw-gateway.service
    journalctl --user -u openclaw-gateway.service -f
    ```
  </Step>

  <Step title="دسترسی به رابط کنترل">
    در رایانهٔ خود، یک URL داشبورد از Pi دریافت کنید:

    ```bash
    ssh user@gateway-host 'openclaw dashboard --no-open'
    ```

    سپس در ترمینال دیگری یک تونل SSH ایجاد کنید:

    ```bash
    ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
    ```

    URL نمایش‌داده‌شده را در مرورگر محلی خود باز کنید. برای دسترسی راه دور دائمی، به [یکپارچه‌سازی Tailscale](/fa/gateway/tailscale) مراجعه کنید.

  </Step>
</Steps>

## نکات بهبود عملکرد

**از USB SSD استفاده کنید** — کارت‌های SD کند هستند و فرسوده می‌شوند. یک USB SSD عملکرد را به‌شکل چشمگیری بهبود می‌دهد و چرخه‌های نوشتن بیشتری را تحمل می‌کند؛ اگر سیستم‌عامل را روی SD نگه می‌دارید، از آن برای `OPENCLAW_STATE_DIR` استفاده کنید. به [راهنمای راه‌اندازی Pi از USB](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#usb-mass-storage-boot) مراجعه کنید.

**کش کامپایل ماژول را فعال کنید** — اجرای مکرر CLI را روی میزبان‌های Pi کم‌توان سریع‌تر می‌کند. `OPENCLAW_NO_RESPAWN=1` راه‌اندازی‌های مجدد معمول Gateway را در همان فرایند نگه می‌دارد، از واگذاری‌های اضافی بین فرایندها جلوگیری می‌کند و ردیابی PID را در میزبان‌های کوچک ساده نگه می‌دارد:

```bash
grep -q 'NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache' ~/.bashrc || cat >> ~/.bashrc <<'EOF' # pragma: allowlist secret
export NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
mkdir -p /var/tmp/openclaw-compile-cache
export OPENCLAW_NO_RESPAWN=1
EOF
source ~/.bashrc
```

از `/var/tmp` استفاده کنید، نه `/tmp` — برخی توزیع‌ها `/tmp` را هنگام راه‌اندازی پاک می‌کنند و در نتیجه کش گرم‌شده از بین می‌رود.

**مصرف حافظه را کاهش دهید** — در راه‌اندازی‌های بدون نمایشگر، حافظهٔ GPU را آزاد و سرویس‌های بلااستفاده را غیرفعال کنید:

```bash
echo 'gpu_mem=16' | sudo tee -a /boot/config.txt
sudo systemctl disable bluetooth
```

**پیکربندی تکمیلی systemd برای راه‌اندازی مجدد پایدار** — اگر این Pi عمدتاً OpenClaw را اجرا می‌کند، یک پیکربندی تکمیلی برای سرویس اضافه کنید:

```bash
systemctl --user edit openclaw-gateway.service
```

```ini
[Service]
Environment=OPENCLAW_NO_RESPAWN=1
Environment=NODE_COMPILE_CACHE=/var/tmp/openclaw-compile-cache
Restart=always
RestartSec=2
TimeoutStartSec=90
```

سپس `systemctl --user daemon-reload && systemctl --user restart openclaw-gateway.service`. در یک Pi بدون نمایشگر، قابلیت ماندگاری سرویس کاربر پس از خروج را نیز یک‌بار فعال کنید: `sudo loginctl enable-linger "$(whoami)"`.

## پیکربندی مدل توصیه‌شده

از آنجا که Pi فقط Gateway را اجرا می‌کند، از مدل‌های API میزبانی‌شده در فضای ابری استفاده کنید — مدل‌های LLM محلی را روی Pi اجرا نکنید؛ حتی مدل‌های کوچک نیز آن‌قدر کند هستند که کاربردی نباشند:

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-6",
        "fallbacks": ["openai/gpt-5.4-mini"]
      }
    }
  }
}
```

## نکات باینری ARM

بیشتر قابلیت‌های OpenClaw بدون تغییر روی ARM64 کار می‌کنند (Node.js، Telegram، WhatsApp/Baileys و Chromium). باینری‌هایی که گاهی نسخهٔ ARM ندارند، معمولاً ابزارهای اختیاری CLI نوشته‌شده با Go/Rust هستند که توسط Skills ارائه می‌شوند. معماری را با `uname -m` بررسی کنید (باید `aarch64` را نشان دهد)، سپس پیش از ساخت از کد منبع به‌عنوان راه‌حل جایگزین، صفحهٔ انتشار باینری مفقود را برای فایل‌های `linux-arm64` / `aarch64` بررسی کنید.

## ماندگاری و پشتیبان‌گیری

وضعیت OpenClaw در مسیرهای زیر قرار دارد:

- `~/.openclaw/` — `openclaw.json`، `auth-profiles.json` مختص هر عامل، وضعیت کانال/ارائه‌دهنده و نشست‌ها.
- `~/.openclaw/workspace/` — فضای کاری عامل (SOUL.md، حافظه و مصنوعات).

این داده‌ها پس از راه‌اندازی مجدد باقی می‌مانند و استفاده از SSD به‌جای کارت SD، هم عملکرد و هم طول عمر آن‌ها را بهبود می‌دهد. با فرمان زیر یک اسنپ‌شات قابل‌انتقال تهیه کنید:

```bash
openclaw backup create
```

## عیب‌یابی

**کمبود حافظه** — با `free -h` بررسی کنید که swap فعال باشد. سرویس‌های بلااستفاده را غیرفعال کنید (`sudo systemctl disable cups bluetooth avahi-daemon`). فقط از مدل‌های مبتنی بر API استفاده کنید.

**عملکرد کند** — به‌جای کارت SD از USB SSD استفاده کنید. با `vcgencmd get_throttled` محدودشدن سرعت CPU را بررسی کنید (باید `0x0` را برگرداند).

**سرویس راه‌اندازی نمی‌شود** — گزارش‌ها را با `journalctl --user -u openclaw-gateway.service --no-pager -n 100` بررسی و `openclaw doctor --non-interactive` را اجرا کنید. اگر Pi بدون نمایشگر است، همچنین بررسی کنید که ماندگاری سرویس کاربر فعال باشد: `sudo loginctl enable-linger "$(whoami)"`.

**مشکلات باینری ARM** — اگر یک Skill با خطای "exec format error" ناموفق شد، بررسی کنید آیا باینری نسخهٔ ARM64 دارد. معماری را با `uname -m` بررسی کنید (باید `aarch64` را نشان دهد).

**قطع‌شدن WiFi** — مدیریت مصرف انرژی WiFi را غیرفعال کنید: `sudo iwconfig wlan0 power off`.

## مراحل بعدی

- [کانال‌ها](/fa/channels) — اتصال Telegram، WhatsApp، Discord و موارد دیگر
- [پیکربندی Gateway](/fa/gateway/configuration) — همهٔ گزینه‌های پیکربندی
- [به‌روزرسانی](/fa/install/updating) — به‌روز نگه‌داشتن OpenClaw

## مطالب مرتبط

- [نمای کلی نصب](/fa/install)
- [سرور Linux](/fa/vps)
- [پلتفرم‌ها](/fa/platforms)
