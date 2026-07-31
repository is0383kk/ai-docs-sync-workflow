---
read_when:
    - شما در حال استقرار OpenClaw روی یک ماشین مجازی ابری با Docker هستید
    - به فرایند مشترک ساخت باینری، ماندگاری و به‌روزرسانی نیاز دارید
summary: مراحل مشترک زمان‌اجرای ماشین مجازی Docker برای میزبان‌های دیرپای Gateway‏ OpenClaw
title: زمان‌اجرای ماشین مجازی Docker
x-i18n:
    generated_at: "2026-07-27T16:40:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d1c474b1f826077ac03c7aaa1e334ed2f38d2de2770f32f2cc907846ecc8bb19
    source_path: install/docker-vm-runtime.md
    workflow: 16
---

مراحل مشترک زمان اجرا برای نصب‌های Docker مبتنی بر ماشین مجازی، مانند GCP، Hetzner و ارائه‌دهندگان مشابه VPS.

## قراردادن باینری‌های موردنیاز در ایمیج

نصب باینری‌ها درون کانتینر در حال اجرا یک دام است: هر چیزی که
در زمان اجرا نصب شود، با راه‌اندازی مجدد از بین می‌رود. هر باینری خارجی موردنیاز یک مهارت را
هنگام ساخت در ایمیج قرار دهید.

مثال‌های زیر فقط سه باینری را به‌ترتیب الفبایی پوشش می‌دهند:

- `gog` (از `gogcli`) برای دسترسی به Gmail
- `goplaces` برای Google Places
- `wacli` برای WhatsApp

این‌ها مثال هستند، نه فهرستی کامل. با استفاده از همین الگو، هر تعداد باینری که
مهارت‌هایتان نیاز دارند نصب کنید. وقتی بعداً مهارتی اضافه می‌کنید که به یک
باینری جدید نیاز دارد:

1. Dockerfile را به‌روزرسانی کنید.
2. ایمیج را دوباره بسازید.
3. کانتینرها را راه‌اندازی مجدد کنید.

**نمونه Dockerfile**

```dockerfile
FROM node:24-bookworm

RUN apt-get update && apt-get install -y socat && rm -rf /var/lib/apt/lists/*

# باینری نمونه 1: CLI مربوط به Gmail ‏(gogcli — با نام `gog` نصب می‌شود)
# نشانی فعلی دارایی Linux را از https://github.com/steipete/gogcli/releases کپی کنید
RUN curl -L https://github.com/steipete/gogcli/releases/latest/download/gogcli_linux_amd64.tar.gz \
  | tar -xzO gog > /usr/local/bin/gog; \
  chmod +x /usr/local/bin/gog

# باینری نمونه 2: CLI مربوط به Google Places
# نشانی فعلی دارایی Linux را از https://github.com/steipete/goplaces/releases کپی کنید
RUN curl -L https://github.com/steipete/goplaces/releases/latest/download/goplaces_linux_amd64.tar.gz \
  | tar -xzO goplaces > /usr/local/bin/goplaces; \
  chmod +x /usr/local/bin/goplaces

# باینری نمونه 3: CLI مربوط به WhatsApp
# نشانی فعلی دارایی Linux را از https://github.com/steipete/wacli/releases کپی کنید
RUN curl -L https://github.com/steipete/wacli/releases/latest/download/wacli-linux-amd64.tar.gz \
  | tar -xzO wacli > /usr/local/bin/wacli; \
  chmod +x /usr/local/bin/wacli

# با استفاده از همین الگو، باینری‌های بیشتری را در ادامه اضافه کنید

WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml .npmrc ./
COPY ui/package.json ./ui/package.json
COPY scripts ./scripts

RUN corepack enable
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build
RUN pnpm ui:install
RUN pnpm ui:build

ENV NODE_ENV=production

CMD ["node","dist/index.js"]
```

<Note>
نشانی‌های بالا نمونه هستند. برای ماشین‌های مجازی مبتنی بر ARM، دارایی‌های `arm64` را انتخاب کنید. برای ساخت‌های تکرارپذیر، نشانی‌های انتشار نسخه‌دار را ثابت کنید.
</Note>

## ساخت و راه‌اندازی

```bash
docker compose build
docker compose up -d openclaw-gateway
```

اگر ساخت هنگام `pnpm install --frozen-lockfile` با `Killed` یا کد خروج 137 شکست خورد، حافظه ماشین مجازی تمام شده است. پیش از تلاش دوباره، از کلاس ماشین بزرگ‌تری استفاده کنید.

باینری‌ها را بررسی کنید:

```bash
docker compose exec openclaw-gateway which gog
docker compose exec openclaw-gateway which goplaces
docker compose exec openclaw-gateway which wacli
```

خروجی مورد انتظار:

```text
/usr/local/bin/gog
/usr/local/bin/goplaces
/usr/local/bin/wacli
```

بررسی کنید که Gateway در حال اجرا است:

```bash
docker compose logs -f openclaw-gateway
curl -fsS http://127.0.0.1:18789/healthz
```

بازگرداندن پاسخ 200 توسط `/healthz` تأیید می‌کند که فرایند Gateway در حال گوش‌دادن و سالم است؛ `HEALTHCHECK` داخلی ایمیج نیز همین نقطه پایانی را پایش می‌کند.

## چه چیزی کجا پایدار می‌ماند

OpenClaw در Docker اجرا می‌شود، اما Docker منبع حقیقت نیست. تمام وضعیت‌های بلندمدت باید پس از راه‌اندازی‌های مجدد، ساخت‌های مجدد و بازراه‌اندازی‌ها باقی بمانند.

| مؤلفه                  | مکان                                                   | سازوکار ماندگاری       | یادداشت‌ها                                                                                                           |
| ---------------------- | ------------------------------------------------------ | ---------------------- | ------------------------------------------------------------------------------------------------------------------- |
| پیکربندی Gateway       | `/home/node/.openclaw/`                                | اتصال ولوم میزبان      | شامل `openclaw.json` است                                                                                            |
| اعتبارنامه‌های کانال/ارائه‌دهنده | `/home/node/.openclaw/credentials/`                    | اتصال ولوم میزبان      | اطلاعات اعتبارنامه کانال و ارائه‌دهنده                                                                               |
| پروفایل‌های احراز هویت مدل | `/home/node/.openclaw/agents/`                         | اتصال ولوم میزبان      | `agents/<agentId>/agent/auth-profiles.json` (OAuth، کلیدهای API)                                                       |
| فایل کلید قدیمی OAuth  | `/home/node/.config/openclaw/`                         | اتصال ولوم میزبان      | سازگاری فقط‌خواندنی برای فایل‌های جانبی OAuth پیش از مهاجرت؛ `openclaw doctor --fix` آن‌ها را به `auth-profiles.json` مهاجرت می‌دهد |
| پیکربندی‌های مهارت     | `/home/node/.openclaw/skills/`                         | اتصال ولوم میزبان      | وضعیت در سطح مهارت                                                                                                  |
| فضای کاری عامل         | `/home/node/.openclaw/workspace/`                      | اتصال ولوم میزبان      | کد و مصنوعات عامل                                                                                                   |
| نشست WhatsApp          | `/home/node/.openclaw/`                                | اتصال ولوم میزبان      | ورود با کد QR را حفظ می‌کند                                                                                          |
| جاکلیدی Gmail          | `/home/node/.openclaw/`                                | ولوم میزبان + گذرواژه  | به `GOG_KEYRING_PASSWORD` نیاز دارد                                                                                     |
| بسته‌های Plugin        | `/home/node/.openclaw/npm`, `/home/node/.openclaw/git` | اتصال ولوم میزبان      | ریشه‌های بسته Plugin قابل‌بارگیری                                                                                    |
| باینری‌های خارجی       | `/usr/local/bin/`                                      | ایمیج Docker            | باید هنگام ساخت در ایمیج قرار گیرند                                                                                  |
| زمان اجرای Node        | فایل‌سیستم کانتینر                                    | ایمیج Docker            | با هر بار ساخت ایمیج، دوباره ساخته می‌شود                                                                            |
| بسته‌های سیستم‌عامل    | فایل‌سیستم کانتینر                                    | ایمیج Docker            | در زمان اجرا نصب نکنید                                                                                               |
| کانتینر Docker         | موقتی                                                   | قابل راه‌اندازی مجدد   | حذف آن بی‌خطر است                                                                                                    |

## به‌روزرسانی‌ها

برای به‌روزرسانی OpenClaw روی ماشین مجازی:

```bash
git pull
docker compose build
docker compose up -d
```

## مرتبط

- [Docker](/fa/install/docker)
- [Podman](/fa/install/podman)
- [ClawDock](/fa/install/clawdock)
