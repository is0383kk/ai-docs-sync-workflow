---
read_when:
    - آزمایش جریان‌های آغاز به کار یا راه‌اندازی با یک Plugin بسته‌بندی‌شده به‌صورت محلی
    - اعتبارسنجی بستهٔ Plugin پیش از انتشار آن
    - جایگزینی نصب خودکار Plugin با یک مصنوع آزمایشی
sidebarTitle: Install overrides
summary: نادیده‌گیری‌های Plugin بسته‌بندی‌شده را با جریان‌های نصب در زمان راه‌اندازی آزمایش کنید
title: لغو تنظیمات نصب Plugin
x-i18n:
    generated_at: "2026-07-27T15:28:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: adc823f49ea9f8fa86e6a89933e43fdc309d808ac24397770495dbe81cb4b0d7
    source_path: plugins/install-overrides.md
    workflow: 16
---

بازنویسی‌های نصب Plugin به نگه‌دارندگان اجازه می‌دهند نصب‌های Plugin در زمان راه‌اندازی را به
یک بسته npm مشخص یا تاربال محلی npm-pack هدایت کنند، به‌جای آنکه از منبع کاتالوگ،
باندل‌شده یا پیش‌فرض npm استفاده شود. این قابلیت‌ها فقط برای E2E و اعتبارسنجی بسته
وجود دارند؛ کاربران عادی Pluginها را با
[`openclaw plugins install`](/fa/cli/plugins) نصب می‌کنند.

<Warning>
بازنویسی‌ها کد Plugin را از منبعی که ارائه می‌کنید اجرا می‌کنند. از آن‌ها فقط در یک
دایرکتوری وضعیت ایزوله یا ماشین آزمایشی یک‌بارمصرف استفاده کنید.
</Warning>

## محیط

بازنویسی‌ها غیرفعال‌اند، مگر اینکه هر دو متغیر تنظیم شده باشند:

```bash
export OPENCLAW_ALLOW_PLUGIN_INSTALL_OVERRIDES=1
export OPENCLAW_PLUGIN_INSTALL_OVERRIDES='{
  "codex": "npm-pack:/tmp/openclaw-codex-2026.5.8.tgz",
  "openclaw-web-search": "npm:@openclaw/web-search@2026.5.8"
}'
```

نگاشت بازنویسی یک JSON با کلید شناسه Plugin است. مقادیر از موارد زیر پشتیبانی می‌کنند:

| پیشوند                | منبع                                                                                           |
| --------------------- | ------------------------------------------------------------------------------------------------ |
| `npm:<registry-spec>` | بسته‌های رجیستری، نسخه‌های دقیق یا برچسب‌ها                                                       |
| `npm-pack:<path.tgz>` | تاربال‌های محلی تولیدشده توسط `npm pack`؛ مسیرهای نسبی از دایرکتوری کاری فعلی تفکیک می‌شوند |

## رفتار

وقتی یک جریان زمان راه‌اندازی Pluginی را نصب می‌کند که شناسه‌اش در نگاشت وجود دارد، OpenClaw
به‌جای منبع کاتالوگ، باندل‌شده یا پیش‌فرض npm از منبع بازنویسی استفاده می‌کند.
این رفتار برای فرایند آغازبه‌کار و هر جریان دیگری که از نصب‌کننده مشترک Plugin
در زمان راه‌اندازی استفاده می‌کند، اعمال می‌شود.

- بازنویسی‌ها همچنان شناسه مورد انتظار Plugin را اعمال می‌کنند: تار‌بالی که به `codex`
  نگاشت شده است، باید Pluginی را نصب کند که شناسه مانیفست آن `codex` است.
- بازنویسی‌ها وضعیت منبع رسمی و مورداعتماد را به ارث نمی‌برند. حتی زمانی که
  ورودی کاتالوگ معمولاً نمایانگر یک بسته متعلق به OpenClaw است، بازنویسی به‌عنوان
  ورودی آزمایشی ارائه‌شده توسط اپراتور در نظر گرفته می‌شود.
- فایل‌های `.env` فضای کاری نمی‌توانند بازنویسی‌های نصب را فعال کنند؛ هر دو متغیر محیطی در
  فهرست مسدودشده dotenv فضای کاری قرار دارند. آن‌ها را در پوسته مورداعتماد، کار CI یا
  فرمان آزمایش راه‌دوری که OpenClaw را راه‌اندازی می‌کند، تنظیم کنید.

## E2E بسته

از یک دایرکتوری وضعیت ایزوله استفاده کنید تا نصب بسته‌ها و سوابق نصب به
وضعیت عادی OpenClaw شما دست نزنند:

```bash
npm pack extensions/codex --pack-destination /tmp

OPENCLAW_STATE_DIR="$(mktemp -d)" \
OPENCLAW_ALLOW_PLUGIN_INSTALL_OVERRIDES=1 \
OPENCLAW_PLUGIN_INSTALL_OVERRIDES='{"codex":"npm-pack:/tmp/openclaw-codex-2026.5.8.tgz"}' \
pnpm openclaw onboard --mode local
```

بسته نصب‌شده را در دایرکتوری وضعیت تأیید کنید:

```bash
find "$OPENCLAW_STATE_DIR/npm/projects" -path '*/node_modules/@openclaw/codex/package.json' -print
grep -R '"@openclaw/codex"' "$OPENCLAW_STATE_DIR/npm/projects"/*/package-lock.json
```

برای E2E ارائه‌دهنده زنده، پیش از اجرای فرمان آزمایش، کلید API واقعی را از یک پوسته
مورداعتماد یا راز CI بارگذاری کنید. کلیدها را چاپ نکنید؛ فقط منبع و موجودبودن
کلید را گزارش دهید.
