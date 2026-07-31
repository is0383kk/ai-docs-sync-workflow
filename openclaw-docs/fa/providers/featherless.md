---
read_when:
    - می‌خواهید از Featherless AI با OpenClaw استفاده کنید
    - به متغیر محیطی کلید API مربوط به Featherless یا قالب ارجاع مدل نیاز دارید
summary: راه‌اندازی Featherless AI، انتخاب مدل و فراخوانی ابزارها
title: Featherless AI
x-i18n:
    generated_at: "2026-07-27T14:34:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9112f7e65b4089bf96933c632d0b62f7fb87d42998d985ca85eb92dc392636b6
    source_path: providers/featherless.md
    workflow: 16
---

[Featherless AI](https://featherless.ai) مدل‌های باز را از طریق یک API
سازگار با OpenAI ارائه می‌دهد. OpenClaw،‏ Featherless را به‌عنوان Plugin رسمی
ارائه‌دهنده خارجی نصب می‌کند و ضمن کوچک نگه‌داشتن کاتالوگ داخلی، در زمان اجرا
شناسه‌های دقیق مدل را از Featherless می‌پذیرد.

| ویژگی        | مقدار                                    |
| --------------- | ---------------------------------------- |
| شناسه ارائه‌دهنده     | `featherless`                            |
| بسته         | `@openclaw/featherless-provider`         |
| متغیر محیطی احراز هویت    | `FEATHERLESS_API_KEY`                    |
| پرچم راه‌اندازی اولیه | `--auth-choice featherless-api-key`      |
| پرچم مستقیم CLI | `--featherless-api-key <key>`            |
| API             | سازگار با OpenAI (`openai-completions`) |
| نشانی پایه        | `https://api.featherless.ai/v1`          |
| مدل پیش‌فرض   | `featherless/Qwen/Qwen3-32B`             |

## راه‌اندازی

Plugin را نصب و Gateway را بازراه‌اندازی کنید:

```bash
openclaw plugins install @openclaw/featherless-provider
openclaw gateway restart
```

راه‌اندازی اولیه را اجرا کنید:

```bash
openclaw onboard --auth-choice featherless-api-key
```

برای راه‌اندازی غیرتعاملی:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice featherless-api-key \
  --featherless-api-key "$FEATHERLESS_API_KEY"
```

یا کلید را در دسترس فرایند Gateway قرار دهید:

```bash
export FEATHERLESS_API_KEY="<your-featherless-api-key>" # pragma: allowlist secret
```

ارائه‌دهنده را تأیید کنید:

```bash
openclaw models list --provider featherless
```

## مدل پیش‌فرض

Plugin از `Qwen/Qwen3-32B` به‌عنوان پیش‌فرض راه‌اندازی استفاده می‌کند، زیرا Featherless
فراخوانی بومی ابزار را برای خانواده Qwen 3 مستند کرده است. OpenClaw پنجره زمینه
32,768 توکنی، محدودیت محافظه‌کارانه خروجی 4,096 توکنی و
کنترل‌های تفکر الگوی گفت‌وگوی Qwen را پیکربندی می‌کند.

فیلدهای هزینه کاتالوگ صفر هستند، زیرا Featherless از چندین حالت صورت‌حساب
پشتیبانی می‌کند و OpenClaw نرخ‌های مختص طرح حساب یا قیمت‌گذاری هر درخواست را
درون خود جای نمی‌دهد.

## سایر مدل‌های Featherless

پس از پیشوند ارائه‌دهنده `featherless/`، از شناسه دقیق مدل Featherless استفاده کنید:

```json5
{
  agents: {
    defaults: {
      model: {
        primary: "featherless/moonshotai/Kimi-K2-Instruct",
      },
    },
  },
}
```

OpenClaw عمداً فهرست عمومی کامل مدل‌های Featherless را در
انتخاب‌گر کپی نمی‌کند. این فهرست بزرگ است و فراداده قابلیت ساختاریافته کافی
برای دسته‌بندی ایمن همه مدل‌های متنی، بینایی، تعبیه‌سازی و استدلالی ارائه
نمی‌دهد. بنابراین، شناسه‌های ناشناخته با پیش‌فرض‌های محافظه‌کارانه صرفاً متنی و
غیراستدلالی تفکیک می‌شوند: پنجره زمینه 4,096 توکنی و محدودیت خروجی 1,024 توکنی.

وقتی مدلی به فراداده متفاوتی نیاز دارد، یک ورودی صریح مدل ارائه‌دهنده اضافه کنید:

```json5
{
  models: {
    mode: "merge",
    providers: {
      featherless: {
        baseUrl: "https://api.featherless.ai/v1",
        apiKey: "${FEATHERLESS_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "google/gemma-3-27b-it",
            name: "Gemma 3 27B",
            input: ["text", "image"],
            reasoning: false,
            contextWindow: 32768,
            maxTokens: 4096,
          },
        ],
      },
    },
  },
}
```

پیش از افزودن فراداده سفارشی، کاتالوگ مدل Featherless را برای بررسی موجود بودن
فعلی مدل و برچسب‌های قابلیت آن بررسی کنید.

## عیب‌یابی

- `401` یا `403`: تأیید کنید `FEATHERLESS_API_KEY` برای فرایند Gateway قابل مشاهده است،
  یا راه‌اندازی اولیه را دوباره اجرا کنید.
- مدل ناشناخته: پس از پیشوند
  `featherless/`، از شناسه دقیق و حساس به بزرگی و کوچکی حروف Featherless استفاده کنید.
- فراخوانی‌های ابزار به‌صورت متن بازگردانده شدند: خانواده مدلی را انتخاب کنید که Featherless برای
  فراخوانی بومی تابع مستند کرده است، مانند Qwen 3.
- Gateway مدیریت‌شده نمی‌تواند کلید را ببیند: آن را در `~/.openclaw/.env` یا منبع محیطی دیگری
  که سرویس بارگذاری می‌کند قرار دهید، سپس Gateway را بازراه‌اندازی کنید.

## مرتبط

- [ارائه‌دهندگان مدل](/fa/concepts/model-providers)
- [همه ارائه‌دهندگان](/fa/providers/index)
- [حالت‌های تفکر](/fa/tools/thinking)
