---
read_when:
    - راه‌اندازی OpenClaw در Oracle Cloud
    - به‌دنبال میزبانی رایگان VPS برای OpenClaw هستید
    - OpenClaw شبانه‌روزی روی یک سرور کوچک می‌خواهید
summary: میزبانی OpenClaw در سطح ARM همیشه رایگان Oracle Cloud
title: ابر اوراکل
x-i18n:
    generated_at: "2026-07-27T15:27:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e1eb95b6bc8ad73e1492a03d8ebe32d89c80e58347614e6ae12d2d3d926d577
    source_path: install/oracle.md
    workflow: 16
---

یک Gateway پایدار OpenClaw را بدون هیچ هزینه‌ای روی ردهٔ ARM سرویس **Always Free** در Oracle Cloud (تا 4 OCPU، ‏24 GB رم و ‏200 GB فضای ذخیره‌سازی) اجرا کنید.

## پیش‌نیازها

- حساب Oracle Cloud ([ثبت‌نام](https://www.oracle.com/cloud/free/)) -- اگر با مشکلی روبه‌رو شدید، [راهنمای ثبت‌نام انجمن](https://gist.github.com/rssnyder/51e3cfedd730e7dd5f4a816143b25dbd) را ببینید
- حساب Tailscale (رایگان در [tailscale.com](https://tailscale.com))
- یک جفت کلید SSH
- حدود 30 دقیقه زمان

## راه‌اندازی

<Steps>
  <Step title="ایجاد یک نمونهٔ OCI">
    1. وارد [Oracle Cloud Console](https://cloud.oracle.com/) شوید.
    2. به **Compute > Instances > Create Instance** بروید.
    3. پیکربندی کنید:
       - **نام:** `openclaw`
       - **ایمیج:** Ubuntu 24.04 (aarch64)
       - **شکل:** `VM.Standard.A1.Flex` (Ampere ARM)
       - **OCPUها:** 2 (یا تا 4)
       - **حافظه:** 12 GB (یا تا 24 GB)
       - **حجم راه‌اندازی:** 50 GB (تا 200 GB رایگان)
       - **کلید SSH:** کلید عمومی خود را اضافه کنید
    4. روی **Create** کلیک کنید و نشانی IP عمومی را یادداشت کنید.

    <Tip>
    اگر ایجاد نمونه با پیام «Out of capacity» ناموفق بود، یک دامنهٔ دسترس‌پذیری دیگر را امتحان کنید یا بعداً دوباره تلاش کنید. ظرفیت ردهٔ رایگان محدود است.
    </Tip>

  </Step>

  <Step title="اتصال و به‌روزرسانی سیستم">
    ```bash
    ssh ubuntu@YOUR_PUBLIC_IP

    sudo apt update && sudo apt upgrade -y
    sudo apt install -y build-essential
    ```

    `build-essential` برای کامپایل ARM برخی وابستگی‌ها لازم است.

  </Step>

  <Step title="پیکربندی کاربر و نام میزبان">
    ```bash
    sudo hostnamectl set-hostname openclaw
    sudo passwd ubuntu
    sudo loginctl enable-linger ubuntu
    ```

    فعال‌کردن linger باعث می‌شود سرویس‌های کاربر پس از خروج نیز در حال اجرا بمانند.

  </Step>

  <Step title="نصب Tailscale">
    ```bash
    curl -fsSL https://tailscale.com/install.sh | sh
    sudo tailscale up --ssh --hostname=openclaw
    ```

    از این پس، از طریق Tailscale متصل شوید: `ssh ubuntu@openclaw`.

  </Step>

  <Step title="نصب OpenClaw">
    ```bash
    curl -fsSL https://openclaw.ai/install.sh | bash
    source ~/.bashrc
    ```

    وقتی پیام «How do you want to hatch your bot?» نمایش داده شد، **Do this later** را انتخاب کنید.

  </Step>

  <Step title="پیکربندی Gateway">
    برای دسترسی امن از راه دور، از احراز هویت توکنی همراه با Tailscale Serve استفاده کنید.

    ```bash
    openclaw config set gateway.bind loopback
    openclaw config set gateway.auth.mode token
    openclaw doctor --generate-gateway-token
    openclaw config set gateway.tailscale.mode serve
    openclaw config set gateway.trustedProxies '["127.0.0.1"]'

    systemctl --user restart openclaw-gateway.service
    ```

    `gateway.trustedProxies=["127.0.0.1"]` در اینجا فقط برای مدیریت IP ارسال‌شده/کلاینت محلی توسط پراکسی محلی Tailscale Serve است. این مورد `gateway.auth.mode: "trusted-proxy"` **نیست**. مسیرهای نمایشگر تفاوت در این راه‌اندازی همچنان رفتار بسته در صورت خطا را حفظ می‌کنند: درخواست‌های خام نمایشگر `127.0.0.1` بدون هدرهای ارسال‌شدهٔ پراکسی، `Diff not found` را برمی‌گردانند. برای پیوست‌ها از `mode=file` / `mode=both` استفاده کنید، یا اگر به پیوندهای قابل‌اشتراک‌گذاری نمایشگر نیاز دارید، نمایشگرهای راه دور را آگاهانه فعال کنید و `plugins.entries.diffs.config.viewerBaseUrl` را تنظیم کنید (یا یک `baseUrl` پراکسی ارسال کنید).

  </Step>

  <Step title="محدودسازی امنیت VCN">
    همهٔ ترافیک به‌جز Tailscale را در لبهٔ شبکه مسدود کنید:

    1. در OCI Console به **Networking > Virtual Cloud Networks** بروید.
    2. روی VCN خود و سپس **Security Lists > Default Security List** کلیک کنید.
    3. همهٔ قواعد ورودی به‌جز `0.0.0.0/0 UDP 41641` (Tailscale) را **حذف کنید**.
    4. قواعد خروجی پیش‌فرض را حفظ کنید (اجازهٔ همهٔ ترافیک خروجی).

    این کار SSH روی درگاه 22، ‏HTTP، ‏HTTPS و هر ترافیک دیگری را در لبهٔ شبکه مسدود می‌کند. از این مرحله به بعد، فقط از طریق Tailscale می‌توانید متصل شوید.

  </Step>

  <Step title="بررسی">
    ```bash
    openclaw --version
    systemctl --user status openclaw-gateway.service
    tailscale serve status
    curl http://localhost:18789
    ```

    از هر دستگاهی در tailnet خود به رابط کنترل دسترسی پیدا کنید:

    ```
    https://openclaw.<tailnet-name>.ts.net/
    ```

    `<tailnet-name>` را با نام tailnet خود جایگزین کنید (در `tailscale status` قابل‌مشاهده است).

  </Step>
</Steps>

## بررسی وضعیت امنیتی

با محدودسازی VCN (تنها UDP 41641 باز است) و اتصال Gateway به loopback، ترافیک عمومی در لبهٔ شبکه مسدود می‌شود و دسترسی مدیریتی فقط از طریق tailnet امکان‌پذیر است. این وضعیت نیاز به چندین مرحلهٔ سنتی ایمن‌سازی VPS را برطرف می‌کند:

| مرحلهٔ سنتی                 | لازم است؟       | دلیل                                                                       |
| --------------------------- | --------------- | -------------------------------------------------------------------------- |
| دیوار آتش UFW               | خیر             | VCN ترافیک را پیش از رسیدن به نمونه مسدود می‌کند.                          |
| fail2ban                    | خیر             | درگاه 22 در VCN مسدود است؛ سطحی برای حملهٔ جست‌وجوی فراگیر وجود ندارد.     |
| ایمن‌سازی sshd              | خیر             | SSH در Tailscale از sshd استفاده نمی‌کند.                                  |
| غیرفعال‌کردن ورود root      | خیر             | Tailscale بر اساس هویت tailnet احراز هویت می‌کند، نه کاربران سیستم.        |
| احراز هویت فقط با کلید SSH  | خیر             | به همین دلیل -- هویت tailnet جایگزین کلیدهای SSH سیستم می‌شود.             |
| ایمن‌سازی IPv6              | معمولاً خیر     | به تنظیمات VCN/زیرشبکه بستگی دارد؛ موارد واقعاً تخصیص‌یافته/درمعرض را بررسی کنید. |

همچنان توصیه می‌شود:

- `chmod 700 ~/.openclaw` برای محدودکردن مجوزهای فایل اعتبارنامه.
- `openclaw security audit` برای بررسی وضعیت مختص OpenClaw.
- اجرای منظم `sudo apt update && sudo apt upgrade` برای وصله‌های سیستم‌عامل.
- دستگاه‌ها را به‌طور دوره‌ای در [کنسول مدیریت Tailscale](https://login.tailscale.com/admin) بازبینی کنید.

فرمان‌های بررسی سریع:

```bash
# تأیید کنید هیچ درگاه عمومی در حال گوش‌دادن نیست
sudo ss -tlnp | grep -v '127.0.0.1\|::1'

# بررسی کنید SSH در Tailscale فعال است
tailscale status | grep -q 'offers: ssh' && echo "SSH در Tailscale فعال است"

# اختیاری: پس از اطمینان از عملکرد SSH در Tailscale، ‏sshd را کاملاً غیرفعال کنید
sudo systemctl disable --now ssh
```

## نکات ARM

ردهٔ Always Free مبتنی بر ARM است (`aarch64`). بیشتر قابلیت‌های OpenClaw بدون مشکل کار می‌کنند؛ تعداد کمی از فایل‌های اجرایی بومی به بیلد ARM نیاز دارند:

- Node.js، ‏Telegram، ‏WhatsApp (Baileys): جاوااسکریپت خالص، بدون مشکل.
- بیشتر بسته‌های npm دارای کد بومی: مصنوعات ازپیش‌ساخته‌شدهٔ `linux-arm64` در دسترس‌اند.
- ابزارهای کمکی اختیاری CLI (برای مثال، فایل‌های اجرایی Go/Rust ارائه‌شده توسط Skills): پیش از نصب، وجود نسخهٔ `aarch64` / `linux-arm64` را بررسی کنید.

معماری را با `uname -m` بررسی کنید (باید `aarch64` را چاپ کند). برای فایل‌های اجرایی فاقد بیلد ARM، آن‌ها را از کد منبع نصب کنید یا از آن‌ها صرف‌نظر کنید.

## ماندگاری و پشتیبان‌گیری

وضعیت OpenClaw در مسیرهای زیر قرار دارد:

- `~/.openclaw/` -- `openclaw.json`، ‏`auth-profiles.json` مختص هر عامل، وضعیت کانال/ارائه‌دهنده و داده‌های نشست.
- `~/.openclaw/workspace/` -- فضای کاری عامل (SOUL.md، حافظه، مصنوعات).

این داده‌ها پس از راه‌اندازی مجدد باقی می‌مانند. برای گرفتن یک اسنپ‌شات قابل‌انتقال:

```bash
openclaw backup create
```

## روش جایگزین: تونل SSH

اگر Tailscale Serve کار نمی‌کند، از دستگاه محلی خود یک تونل SSH ایجاد کنید:

```bash
ssh -L 18789:127.0.0.1:18789 ubuntu@openclaw
```

سپس `http://localhost:18789` را باز کنید.

## عیب‌یابی

**ایجاد نمونه ناموفق است («Out of capacity»)** -- نمونه‌های ARM ردهٔ رایگان پرطرفدار هستند. یک دامنهٔ دسترس‌پذیری دیگر را امتحان کنید یا در ساعات کم‌ترافیک دوباره تلاش کنید.

**Tailscale متصل نمی‌شود** -- برای احراز هویت مجدد، `sudo tailscale up --ssh --hostname=openclaw --reset` را اجرا کنید.

**Gateway راه‌اندازی نمی‌شود** -- ‏`openclaw doctor --non-interactive` را اجرا کنید و گزارش‌ها را با `journalctl --user -u openclaw-gateway.service -n 50` بررسی کنید.

**مشکلات فایل اجرایی ARM** -- بیشتر بسته‌های npm روی ARM64 کار می‌کنند. برای فایل‌های اجرایی بومی، به‌دنبال نسخه‌های `linux-arm64` یا `aarch64` بگردید. معماری را با `uname -m` بررسی کنید.

## مراحل بعدی

- [کانال‌ها](/fa/channels) -- اتصال Telegram، ‏WhatsApp، ‏Discord و موارد دیگر
- [پیکربندی Gateway](/fa/gateway/configuration) -- همهٔ گزینه‌های پیکربندی
- [به‌روزرسانی](/fa/install/updating) -- به‌روز نگه‌داشتن OpenClaw

## مرتبط

- [نمای کلی نصب](/fa/install)
- [GCP](/fa/install/gcp)
- [میزبانی VPS](/fa/vps)
