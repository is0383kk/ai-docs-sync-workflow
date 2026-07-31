---
read_when:
    - می‌خواهید بدون سرور محلی Ollama از مدل‌های میزبانی‌شدهٔ Ollama استفاده کنید
    - به شناسه ارائه‌دهنده، کلید یا نقطه پایانی ollama-cloud نیاز دارید
summary: استفاده مستقیم از Ollama Cloud با OpenClaw
title: Ollama Cloud
x-i18n:
    generated_at: "2026-07-27T15:52:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 966e5237e37134cef109979079db390e9844714001e921e7976dc8ca7f58bcc4
    source_path: providers/ollama-cloud.md
    workflow: 16
---

Ollama Cloud، API میزبانی‌شدهٔ مدل‌های Ollama است. ارائه‌دهندهٔ `ollama-cloud` آن را
مستقیماً در `https://ollama.com` و از طریق API بومی `/api/chat` متعلق به Ollama فراخوانی می‌کند، بدون
سرور محلی Ollama و بدون برنامهٔ محلی Ollama که به حالت ابری وارد شده باشد. از ارجاع‌های مدل
مانند `ollama-cloud/kimi-k2.6` استفاده کنید.

OpenClaw، `ollama-cloud` را به‌عنوان شناسهٔ ارائه‌دهندهٔ مستقل خود ثبت می‌کند تا
اعتبارنامه‌های مختص فضای ابری، کشف زندهٔ کاتالوگ و انتخاب مدل با
میزبان محلی `ollama` آمیخته نشوند. برای Ollama محلی، مسیریابی ترکیبی ابری و محلی،
تعبیه‌ها و جزئیات میزبان سفارشی، به [Ollama](/fa/providers/ollama) مراجعه کنید.

## راه‌اندازی

یک کلید API برای Ollama Cloud در [ollama.com/settings/keys](https://ollama.com/settings/keys) ایجاد کنید، سپس دستور زیر را اجرا کنید:

```bash
openclaw onboard --auth-choice ollama-cloud
```

یا تنظیم کنید:

```bash
export OLLAMA_API_KEY="<your-ollama-cloud-api-key>" # pragma: allowlist secret
```

راه‌اندازی غیرتعاملی کلید را مستقیماً می‌پذیرد:

```bash
openclaw onboard --auth-choice ollama-cloud --ollama-cloud-api-key "<key>"
```

راه‌اندازی، مدل پیش‌فرض را روی `ollama-cloud/kimi-k2.5:cloud` تنظیم می‌کند.

## پیش‌فرض‌ها

- ارائه‌دهنده: `ollama-cloud`
- نشانی پایه: `https://ollama.com`
- متغیر محیطی: `OLLAMA_API_KEY`
- سبک API: API بومی `/api/chat` متعلق به Ollama
- مدل پیش‌فرض راه‌اندازی: `ollama-cloud/kimi-k2.5:cloud`

## چه زمانی Ollama Cloud را انتخاب کنید

- مدل‌های میزبانی‌شدهٔ Ollama را بدون اجرای محلی `ollama serve` می‌خواهید.
- همان ساختار API بومی گفت‌وگوی Ollama را می‌خواهید که OpenClaw برای Ollama محلی استفاده می‌کند،
  اما به `https://ollama.com` اشاره داشته باشد.
- برای مدل‌هایی که از قبل در کاتالوگ میزبانی‌شدهٔ Ollama قرار دارند، یک مسیر ابری ساده
  می‌خواهید.
- به دریافت محلی مدل، کنترل GPU محلی یا استنتاج فقط در LAN نیاز ندارید.

وقتی مسیریابی فقط محلی یا ترکیبی ابری و محلی را از طریق یک میزبان Ollama واردشده می‌خواهید،
به‌جای آن از [Ollama](/fa/providers/ollama) استفاده کنید. وقتی به معناشناسی
`/v1/chat/completions` یا قابلیت‌های مختص ارائه‌دهنده به سبک OpenAI نیاز دارید، از یک
ارائه‌دهندهٔ سازگار با OpenAI استفاده کنید.

## مدل‌ها

ارائه‌دهنده به یک کلید API نیاز دارد؛ بدون آن غیرفعال می‌ماند. در صورت وجود کلید،
OpenClaw مدل‌های Ollama Cloud را به‌صورت زنده از کاتالوگ میزبانی‌شده کشف می‌کند:

```bash
openclaw models list --provider ollama-cloud
openclaw models set ollama-cloud/kimi-k2.6
```

شناسه‌های میزبانی‌شده در کاتالوگ زنده شامل `deepseek-v4-flash`، `glm-5`،
`gpt-oss:20b`، `kimi-k2.6` و `minimax-m2.7` هستند. وقتی کشف زنده
چیزی برنگرداند، OpenClaw به ردیف‌های همراه `kimi-k2.5:cloud`،
`minimax-m2.7:cloud`، `glm-5.1:cloud` و `glm-5.2:cloud` بازمی‌گردد.

شناسه‌های مدل، شناسه‌های کاتالوگ ابری هستند، نه نام‌های دریافت محلی. اگر نام مدلی در
یک میزبان محلی Ollama کار می‌کند اما در کاتالوگ میزبانی‌شده وجود ندارد، به‌جای آن از ارائه‌دهندهٔ `ollama`
با همان میزبان محلی استفاده کنید.

## آزمون زنده

برای آزمون‌های دود کلید API مربوط به Ollama Cloud، آزمون زندهٔ Ollama را به نقطهٔ پایانی
میزبانی‌شده هدایت کنید و مدلی را از کاتالوگ فعلی خود انتخاب کنید:

```bash
export OLLAMA_API_KEY="<your-ollama-cloud-api-key>" # pragma: allowlist secret

OPENCLAW_LIVE_TEST=1 \
OPENCLAW_LIVE_OLLAMA=1 \
OPENCLAW_LIVE_OLLAMA_BASE_URL=https://ollama.com \
OPENCLAW_LIVE_OLLAMA_MODEL=kimi-k2.6 \
pnpm test:live -- extensions/ollama/ollama.live.test.ts
```

آزمون دود ابری، متن، جریان بومی و جست‌وجوی وب را اجرا می‌کند؛ برای نادیده‌گرفتن
جست‌وجوی وب، `OPENCLAW_LIVE_OLLAMA_WEB_SEARCH=0` را تنظیم کنید. این آزمون به‌طور پیش‌فرض تعبیه‌ها را برای
`https://ollama.com` نادیده می‌گیرد، زیرا ممکن است کلیدهای API مربوط به Ollama Cloud
مجوز `/api/embed` را نداشته باشند؛ برای اجرای اجباری آن‌ها، `OPENCLAW_LIVE_OLLAMA_EMBEDDINGS=1` را تنظیم کنید.

## عیب‌یابی

- خطاهای `Ollama Cloud requires an API key` / `Set OLLAMA_API_KEY`: یک
  کلید API واقعی فضای ابری ارائه کنید. نشانگر محلی `ollama-local` فقط برای میزبان‌های
  محلی یا خصوصی Ollama است.
- خطاهای مدل ناشناخته: `openclaw models list --provider ollama-cloud` را اجرا کنید و
  شناسهٔ مدل میزبانی‌شده را دقیقاً کپی کنید.
- مشکلات فراخوانی ابزار یا JSON خام در میزبان‌های سفارشی Ollama: بررسی کنید که آیا
  تصادفاً از نشانی `/v1` سازگار با OpenAI استفاده می‌کنید. مسیرهای Ollama باید
  از نشانی پایهٔ بومی و بدون پسوند `/v1` استفاده کنند.

## مرتبط

- [Ollama](/fa/providers/ollama)
- [ارائه‌دهندگان مدل](/fa/concepts/model-providers)
- [همهٔ ارائه‌دهندگان](/fa/providers/index)
