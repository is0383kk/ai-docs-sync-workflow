---
read_when:
    - می‌خواهید OpenClaw فقط زمانی یک سرور مدل محلی را راه‌اندازی کند که ارائه‌دهنده مدل یا تعبیه‌سازی آن انتخاب شده باشد
    - شما ds4، inferrs، vLLM، llama.cpp، MLX یا سرور محلی دیگری سازگار با OpenAI را اجرا می‌کنید
    - باید شروع سرد، آمادگی و خاموش‌شدن هنگام بی‌کاری را برای ارائه‌دهندگان محلی کنترل کنید
summary: راه‌اندازی سرورهای مدل محلی برحسب تقاضا پیش از درخواست‌های مدل و تعبیه‌سازی OpenClaw
title: سرویس‌های مدل محلی
x-i18n:
    generated_at: "2026-07-27T16:32:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a761113dd591fed0394379b2bad173165efc5e284565c652493e73d1e724529d
    source_path: gateway/local-model-services.md
    workflow: 16
---

`models.providers.<id>.localService` یک سرور مدل محلی متعلق به ارائه‌دهنده را در صورت نیاز راه‌اندازی می‌کند. وقتی یک درخواست مدل یا تعبیه آن ارائه‌دهنده را انتخاب می‌کند، OpenClaw نقطه پایانی سلامت را بررسی می‌کند، اگر فرایند متوقف باشد آن را آغاز می‌کند، تا آماده‌شدن منتظر می‌ماند و سپس درخواست را ارسال می‌کند. از آن برای جلوگیری از روشن نگه‌داشتن سرورهای محلی پرهزینه در تمام طول روز استفاده کنید.

## نحوه کار

1. یک درخواست مدل یا تعبیه به یک ارائه‌دهنده پیکربندی‌شده نگاشت می‌شود.
2. اگر آن ارائه‌دهنده دارای `localService` باشد، OpenClaw نشانی `healthUrl` را بررسی می‌کند.
3. در صورت موفقیت بررسی، OpenClaw از سروری که از قبل در حال اجرا است استفاده می‌کند.
4. در صورت ناموفق‌بودن بررسی، OpenClaw فرایند `command` را با `args` ایجاد می‌کند.
5. OpenClaw نقطه پایانی سلامت را تا پایان مهلت `readyTimeoutMs` به‌طور دوره‌ای بررسی می‌کند.
6. درخواست از مسیر انتقال عادی مدل یا تعبیه عبور می‌کند.
7. اگر OpenClaw فرایند را راه‌اندازی کرده باشد و `idleStopMs` تنظیم شده باشد، پس از آنکه آخرین درخواست در حال اجرا به همان مدت بیکار ماند، فرایند را متوقف می‌کند.

OpenClaw برای این کار launchd،‏ systemd،‏ Docker یا هیچ سرویس پس‌زمینه‌ای را نصب نمی‌کند. سرور یک فرایند فرزند ساده از نخستین فرایند OpenClaw است که به آن نیاز پیدا کرده است.

راه‌اندازی برای هر ارائه‌دهنده پیکربندی‌شده و هر مجموعه فرمان/آرگومان/محیط به‌صورت ترتیبی انجام می‌شود؛ بنابراین درخواست‌های هم‌زمان گفت‌وگو و تعبیه برای یک سرویس، سرورهای تکراری ایجاد نمی‌کنند. هر درخواست تا پایان پردازش پاسخ، اجاره اختصاصی خود را نگه می‌دارد؛ بنابراین خاموش‌سازی هنگام بیکاری برای تمام درخواست‌های مدل و تعبیه در حال اجرا منتظر می‌ماند. نام‌های مستعار پیکربندی‌شده ارائه‌دهندگان متمایز باقی می‌مانند: دو نام مستعار می‌توانند بدون ادغام‌شدن روی یک شناسه آداپتور Ollama،‏ LM Studio یا سازگار با OpenAI، به میزبان‌های GPU متفاوتی اشاره کنند.

اگر فرایند دیگری از OpenClaw از قبل سروری سالم در همان `healthUrl` داشته باشد، این فرایند بدون پذیرفتن مالکیت آن دوباره از آن استفاده می‌کند (هر فرایند فقط فرزند ایجادشده توسط خودش را مدیریت می‌کند). گزارش‌های راه‌اندازی و خروج شامل بخش‌های انتهایی محدود و پالایش‌شده خروجی فرایند فرزند، به‌همراه جزئیات زمان‌بندی و خروج هستند؛ مقادیر محیطی پیکربندی‌شده هرگز منتشر نمی‌شوند.

## ساختار پیکربندی

```json5
{
  models: {
    providers: {
      local: {
        baseUrl: "http://127.0.0.1:8000/v1",
        apiKey: "local-model",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/absolute/path/to/server",
          args: ["--host", "127.0.0.1", "--port", "8000"],
          cwd: "/absolute/path/to/working-dir",
          env: { LOCAL_MODEL_CACHE: "/absolute/path/to/cache" },
          healthUrl: "http://127.0.0.1:8000/v1/models",
          readyTimeoutMs: 180000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "my-local-model",
            name: "My Local Model",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

مقدار `timeoutSeconds` را در ورودی ارائه‌دهنده (نه `localService`) تنظیم کنید تا راه‌اندازی‌های سرد کند و تولیدهای طولانی با مهلت پیش‌فرض درخواست مدل مواجه نشوند. هرگاه سرور شما آمادگی را در مکانی غیر از `/models` در نشانی پایه ارائه می‌کند، یک `healthUrl` صریح تنظیم کنید.

## فیلدها

| فیلد            | الزامی | توضیحات                                                                                                                          |
| ---------------- | -------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `command`        | بله      | مسیر مطلق فایل اجرایی. جست‌وجویی در PATH پوسته انجام نمی‌شود.                                                                                      |
| `args`           | خیر       | آرگومان‌های فرایند. بدون گسترش پوسته، پایپ، تطبیق الگو یا نقل‌قول.                                                                  |
| `cwd`            | خیر       | پوشه کاری فرایند.                                                                                                   |
| `env`            | خیر       | متغیرهای محیطی که روی محیط فرایند OpenClaw ادغام می‌شوند.                                                                  |
| `healthUrl`      | خیر       | نشانی آمادگی. مقدار پیش‌فرض `baseUrl` است که `/models` به آن افزوده می‌شود (`http://127.0.0.1:8000/v1` به `http://127.0.0.1:8000/v1/models` تبدیل می‌شود). |
| `readyTimeoutMs` | خیر       | مهلت آمادگی هنگام راه‌اندازی. پیش‌فرض: `120000`.                                                                                       |
| `idleStopMs`     | خیر       | تأخیر خاموش‌سازی هنگام بیکاری برای فرایندی که OpenClaw راه‌اندازی کرده است. مقدار `0` یا حذف این فیلد، آن را تا زمان خروج OpenClaw زنده نگه می‌دارد.                             |

## نمونه Inferrs

Inferrs یک بک‌اند سفارشی `/v1` سازگار با OpenAI است؛ بنابراین همان API ‏`localService` با یک ورودی ارائه‌دهنده `inferrs` کار می‌کند:

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/google/gemma-4-E2B-it" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "/opt/homebrew/bin/inferrs",
          args: [
            "serve",
            "google/gemma-4-E2B-it",
            "--host",
            "127.0.0.1",
            "--port",
            "8080",
            "--device",
            "metal",
          ],
          healthUrl: "http://127.0.0.1:8080/v1/models",
          readyTimeoutMs: 180000,
          idleStopMs: 0,
        },
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: { requiresStringContent: true },
          },
        ],
      },
    },
  },
}
```

مقدار `command` را با نتیجه `which inferrs` در دستگاهی که OpenClaw را اجرا می‌کند جایگزین کنید. راه‌اندازی کامل Inferrs:‏ [Inferrs](/fa/providers/inferrs).

## نمونه ds4

```json5
{
  models: {
    providers: {
      ds4: {
        baseUrl: "http://127.0.0.1:18000/v1",
        apiKey: "ds4-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        localService: {
          command: "<DS4_DIR>/ds4-server",
          args: [
            "--model",
            "<DS4_DIR>/ds4flash.gguf",
            "--host",
            "127.0.0.1",
            "--port",
            "18000",
            "--ctx",
            "32768",
            "--tokens",
            "128",
          ],
          cwd: "<DS4_DIR>",
          healthUrl: "http://127.0.0.1:18000/v1/models",
          readyTimeoutMs: 300000,
          idleStopMs: 0,
        },
        models: [],
      },
    },
  },
}
```

دستورهای راه‌اندازی کامل، اندازه‌بندی زمینه و راستی‌آزمایی: [ds4](/fa/providers/ds4).

## مرتبط

<CardGroup cols={2}>
  <Card title="مدل‌های محلی" href="/fa/gateway/local-models" icon="server">
    راه‌اندازی مدل محلی، گزینه‌های ارائه‌دهنده و راهنمای ایمنی.
  </Card>
  <Card title="Inferrs" href="/fa/providers/inferrs" icon="cpu">
    OpenClaw را از طریق سرور محلی سازگار با OpenAI در Inferrs اجرا کنید.
  </Card>
</CardGroup>
