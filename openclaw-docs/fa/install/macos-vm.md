---
read_when:
    - می‌خواهید OpenClaw از محیط اصلی macOS شما جدا باشد
    - می‌خواهید iMessage را در یک محیط ایزوله یکپارچه کنید
    - یک محیط macOS قابل بازنشانی می‌خواهید که بتوانید آن را کلون کنید
    - می‌خواهید گزینه‌های ماشین مجازی macOS محلی را با گزینه‌های میزبانی‌شده مقایسه کنید
summary: هنگامی که به جداسازی یا iMessage نیاز دارید، OpenClaw را در یک ماشین مجازی macOS سندباکس‌شده (محلی یا میزبانی‌شده) اجرا کنید
title: ماشین‌های مجازی macOS
x-i18n:
    generated_at: "2026-07-27T15:26:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7e6b963faaf40f65adce1081715bc295059b8bed278a8c71a05a86e04ad7a7a5
    source_path: install/macos-vm.md
    workflow: 16
---

## پیش‌فرض پیشنهادی (برای بیشتر کاربران)

- **VPS کوچک لینوکسی** برای Gateway همیشه‌روشن و هزینهٔ کم. به [میزبانی VPS](/fa/vps) مراجعه کنید.
- **سخت‌افزار اختصاصی** (Mac mini یا دستگاه لینوکسی) اگر کنترل کامل و یک **IP خانگی** برای خودکارسازی مرورگر می‌خواهید. بسیاری از وب‌سایت‌ها IPهای مرکز داده را مسدود می‌کنند، بنابراین مرور محلی اغلب بهتر کار می‌کند.
- **ترکیبی**: Gateway را روی یک VPS ارزان نگه دارید و هرگاه به خودکارسازی مرورگر/رابط کاربری نیاز داشتید، Mac خود را به‌عنوان **Node** متصل کنید. به [Nodeها](/fa/nodes) و [Gateway راه‌دور](/fa/gateway/remote) مراجعه کنید.

فقط زمانی از ماشین مجازی macOS استفاده کنید که مشخصاً به قابلیت‌های مختص macOS مانند iMessage نیاز دارید، یا می‌خواهید جداسازی سخت‌گیرانه‌ای از Mac روزمرهٔ خود داشته باشید.

## گزینه‌های ماشین مجازی macOS

### ماشین مجازی محلی روی Mac مجهز به Apple Silicon شما (Lume)

با استفاده از [Lume](https://cua.ai/docs/lume)، OpenClaw را در یک ماشین مجازی macOS سندباکس‌شده روی Mac فعلی مجهز به Apple Silicon خود اجرا کنید. این روش مزایای زیر را فراهم می‌کند:

- محیط کامل macOS به‌صورت ایزوله (میزبان شما پاک می‌ماند)
- پشتیبانی از iMessage از طریق `imsg`؛ مسیر محلی پیش‌فرض در Linux/Windows امکان‌پذیر نیست
- بازنشانی فوری با کلون‌کردن ماشین‌های مجازی
- بدون هزینهٔ اضافی سخت‌افزار یا فضای ابری

### ارائه‌دهندگان Mac میزبانی‌شده (ابری)

اگر macOS را در فضای ابری می‌خواهید، ارائه‌دهندگان Mac میزبانی‌شده نیز مناسب‌اند:

- [MacStadium](https://www.macstadium.com/) (Macهای میزبانی‌شده)
- سایر فروشندگان Mac میزبانی‌شده نیز قابل‌استفاده‌اند؛ مستندات ماشین مجازی و SSH آن‌ها را دنبال کنید

پس از دسترسی SSH به یک ماشین مجازی macOS، از بخش [نصب OpenClaw](#6-install-openclaw) در ادامه پیش بروید.

## مسیر سریع (Lume، کاربران باتجربه)

1. Lume را نصب کنید.
2. `lume create openclaw --os macos --ipsw latest`
3. Setup Assistant را تکمیل و Remote Login (SSH) را فعال کنید.
4. `lume run openclaw --no-display`
5. با SSH وارد شوید، OpenClaw را نصب و کانال‌ها را پیکربندی کنید.
6. تمام.

## موارد موردنیاز (Lume)

- Mac مجهز به Apple Silicon (M1/M2/M3/M4)
- macOS Sequoia یا جدیدتر روی میزبان
- حدود 60 GB فضای خالی دیسک برای هر ماشین مجازی
- حدود 20 دقیقه

## 1) نصب Lume

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/trycua/cua/main/libs/lume/scripts/install.sh)"
```

اگر `~/.local/bin` در PATH شما نیست:

```bash
echo 'export PATH="$PATH:$HOME/.local/bin"' >> ~/.zshrc && source ~/.zshrc
```

بررسی کنید:

```bash
lume --version
```

مستندات: [نصب Lume](https://cua.ai/docs/lume/guide/getting-started/installation)

## 2) ایجاد ماشین مجازی macOS

```bash
lume create openclaw --os macos --ipsw latest
```

این فرمان macOS را بارگیری و ماشین مجازی را ایجاد می‌کند. یک پنجرهٔ VNC به‌طور خودکار باز می‌شود.

<Note>
بارگیری بسته به اتصال شما ممکن است مدتی طول بکشد.
</Note>

## 3) تکمیل Setup Assistant

در پنجرهٔ VNC:

1. زبان و منطقه را انتخاب کنید.
2. از Apple ID عبور کنید (یا اگر بعداً iMessage می‌خواهید، وارد شوید).
3. یک حساب کاربری ایجاد کنید (نام کاربری و گذرواژه را به خاطر بسپارید).
4. از همهٔ قابلیت‌های اختیاری عبور کنید.

پس از تکمیل راه‌اندازی:

1. SSH را فعال کنید: System Settings -> General -> Sharing، سپس "Remote Login" را فعال کنید.
2. برای استفادهٔ بدون نمایشگر از ماشین مجازی، ورود خودکار را فعال کنید: System Settings -> Users & Groups، گزینهٔ "Automatically log in as:" را انتخاب و کاربر ماشین مجازی را برگزینید.

## 4) دریافت نشانی IP ماشین مجازی

```bash
lume get openclaw
```

نشانی IP را پیدا کنید (معمولاً `192.168.64.x`).

## 5) ورود به ماشین مجازی با SSH

```bash
ssh youruser@192.168.64.X
```

`youruser` را با حسابی که ایجاد کرده‌اید و IP را با نشانی IP ماشین مجازی خود جایگزین کنید.

## 6) نصب OpenClaw

داخل ماشین مجازی:

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

برای راه‌اندازی ارائه‌دهندهٔ مدل خود (Anthropic، OpenAI و غیره)، اعلان‌های فرایند آغازبه‌کار را دنبال کنید.

## 7) پیکربندی کانال‌ها

فایل پیکربندی را ویرایش کنید:

```bash
nano ~/.openclaw/openclaw.json
```

کانال‌های خود را اضافه کنید:

```json5
{
  channels: {
    telegram: {
      botToken: "YOUR_BOT_TOKEN",
    },
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551234567"],
    },
  },
}
```

سپس وارد WhatsApp شوید (کد QR را اسکن کنید):

```bash
openclaw channels login
```

## 8) اجرای ماشین مجازی بدون نمایشگر

ماشین مجازی را متوقف و بدون نمایشگر دوباره راه‌اندازی کنید:

```bash
lume stop openclaw
lume run openclaw --no-display
```

ماشین مجازی در پس‌زمینه اجرا می‌شود؛ دیمن OpenClaw، Gateway را در حال اجرا نگه می‌دارد. برای بررسی وضعیت:

```bash
ssh youruser@192.168.64.X "openclaw status"
```

## امتیاز ویژه: یکپارچه‌سازی iMessage

این قابلیت برجستهٔ اجرا روی macOS است. برای افزودن Messages به OpenClaw، از [iMessage](/fa/channels/imessage) همراه با `imsg` استفاده کنید.

داخل ماشین مجازی:

1. وارد Messages شوید.
2. `imsg` را نصب کنید.
3. برای فرایندی که OpenClaw/`imsg` را اجرا می‌کند، مجوزهای Full Disk Access و Automation را اعطا کنید.
4. پشتیبانی از RPC را با `imsg rpc --help` بررسی کنید.

موارد زیر را به پیکربندی OpenClaw خود اضافه کنید:

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
    },
  },
}
```

Gateway را دوباره راه‌اندازی کنید. عامل شما اکنون می‌تواند iMessageها را ارسال و دریافت کند. جزئیات کامل راه‌اندازی: [کانال iMessage](/fa/channels/imessage).

## ذخیرهٔ یک ایمیج طلایی

پیش از سفارشی‌سازی بیشتر، از وضعیت پاک خود اسنپ‌شات بگیرید:

```bash
lume stop openclaw
lume clone openclaw openclaw-golden
```

هر زمان خواستید بازنشانی کنید:

```bash
lume stop openclaw && lume delete openclaw
lume clone openclaw-golden openclaw
lume run openclaw --no-display
```

## اجرای 24/7

ماشین مجازی را با این روش‌ها در حال اجرا نگه دارید:

- Mac خود را به برق متصل نگه دارید
- حالت خواب را در System Settings -> Energy Saver غیرفعال کنید
- در صورت نیاز از `caffeinate` استفاده کنید

برای اجرای واقعاً همیشگی، یک Mac mini اختصاصی یا VPS کوچک را در نظر بگیرید. به [میزبانی VPS](/fa/vps) مراجعه کنید.

## عیب‌یابی

| مشکل                  | راه‌حل                                                                            |
| ------------------------ | ----------------------------------------------------------------------------------- |
| امکان اتصال SSH به ماشین مجازی وجود ندارد       | بررسی کنید "Remote Login" در System Settings ماشین مجازی فعال باشد                         |
| IP ماشین مجازی نمایش داده نمی‌شود        | منتظر بمانید ماشین مجازی کاملاً راه‌اندازی شود، سپس `lume get openclaw` را دوباره اجرا کنید                            |
| فرمان Lume پیدا نشد   | `~/.local/bin` را به PATH خود اضافه کنید                                                     |
| کد QR واتس‌اپ اسکن نمی‌شود | هنگام اجرای `openclaw channels login` مطمئن شوید وارد ماشین مجازی هستید (نه میزبان) |

## مستندات مرتبط

- [میزبانی VPS](/fa/vps)
- [Nodeها](/fa/nodes)
- [Gateway راه‌دور](/fa/gateway/remote)
- [کانال iMessage](/fa/channels/imessage)
- [شروع سریع Lume](https://cua.ai/docs/lume/guide/getting-started/quickstart)
- [مرجع CLI مربوط به Lume](https://cua.ai/docs/lume/reference/cli-reference)
- [راه‌اندازی ماشین مجازی بدون نظارت](https://cua.ai/docs/lume/guide/fundamentals/unattended-setup) (پیشرفته)
- [سندباکس‌کردن با Docker](/fa/install/docker) (رویکرد جایگزین برای جداسازی)
