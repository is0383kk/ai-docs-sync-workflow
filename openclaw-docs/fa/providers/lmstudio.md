---
read_when:
    - می‌خواهید OpenClaw را با مدل‌های متن‌باز از طریق LM Studio اجرا کنید
    - می‌خواهید LM Studio را راه‌اندازی و پیکربندی کنید
summary: اجرای OpenClaw با LM Studio
title: LM Studio
x-i18n:
    generated_at: "2026-07-27T14:32:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f43b4d04aad6e5edfdf224747083834ebd441aa7f91ccbf2d61de990443fc414
    source_path: providers/lmstudio.md
    workflow: 16
---

LM Studio مدل‌های llama.cpp (GGUF) یا MLX را به‌صورت محلی، در قالب یک برنامهٔ GUI یا daemon بدون رابط `llmster` اجرا می‌کند. برای راهنمای نصب و مستندات محصول، به [lmstudio.ai](https://lmstudio.ai/) مراجعه کنید.

## شروع سریع

<Steps>
  <Step title="نصب و راه‌اندازی سرور">
    LM Studio (نسخهٔ دسکتاپ) یا `llmster` (بدون رابط) را نصب کنید، سپس سرور را راه‌اندازی کنید:

    ```bash
    lms server start --port 1234
    ```

    یا daemon بدون رابط را اجرا کنید:

    ```bash
    lms daemon up
    ```

    اگر از برنامهٔ دسکتاپ استفاده می‌کنید، برای بارگذاری روان مدل، JIT را فعال کنید؛ به
    [راهنمای JIT و TTL در LM Studio](https://lmstudio.ai/docs/developer/core/ttl-and-auto-evict) مراجعه کنید.

  </Step>
  <Step title="در صورت فعال بودن احراز هویت، کلید API را تنظیم کنید">
    ```bash
    export LM_API_TOKEN="your-lm-studio-api-token"
    ```

    اگر احراز هویت LM Studio غیرفعال است، هنگام راه‌اندازی کلید API را خالی بگذارید. به
    [احراز هویت LM Studio](https://lmstudio.ai/docs/developer/core/authentication) مراجعه کنید.

  </Step>
  <Step title="اجرای راه‌اندازی اولیه">
    ```bash
    openclaw onboard
    ```

    `LM Studio` را انتخاب کنید، سپس در اعلان `Default model` یک مدل برگزینید.

    در یک راه‌اندازی هدایت‌شدهٔ جدید، OpenClaw ابتدا `/api/v1/models` را روی میزبان
    پیش‌فرض یا پیکربندی‌شدهٔ LM Studio واکشی می‌کند. یک LLM موجود فقط زمانی به‌طور خودکار
    پیشنهاد می‌شود که LM Studio آموزش ابزار و دست‌کم 16K زمینهٔ مؤثر را گزارش کند.
    برای مدل‌های بارگذاری‌شده، زمینهٔ نمونهٔ بارگذاری‌شده بر حداکثر بزرگ‌تر اعلام‌شده
    اولویت دارد. همان توالی راه‌اندازی CLI/macOS پیش از ذخیره‌سازی، مسیر را با یک
    تکمیل واقعی اعتبارسنجی می‌کند. بررسی خودکار هرگز مدلی را دانلود نمی‌کند و ورودی‌های
    کاتالوگِ مختص تعبیه را نادیده می‌گیرد.

  </Step>
</Steps>

بعداً مدل پیش‌فرض را تغییر دهید:

```bash
openclaw models set lmstudio/qwen/qwen3.5-9b
```

کلیدهای مدل LM Studio از قالب `author/model-name` استفاده می‌کنند (برای مثال `qwen/qwen3.5-9b`)؛ ارجاع‌های مدل OpenClaw
ارائه‌دهنده را به ابتدای آن می‌افزایند: `lmstudio/qwen/qwen3.5-9b`. برای یافتن کلید دقیق یک مدل، فرمان
زیر را اجرا کنید و فیلد `key` را ببینید:

```bash
curl http://localhost:1234/api/v1/models
```

## راه‌اندازی اولیهٔ غیرتعاملی

```bash
openclaw onboard --non-interactive --accept-risk --auth-choice lmstudio
```

یا نشانی URL پایه، مدل و کلید API را صریحاً مشخص کنید:

```bash
openclaw onboard \
  --non-interactive \
  --accept-risk \
  --auth-choice lmstudio \
  --custom-base-url http://localhost:1234/v1 \
  --lmstudio-api-key "$LM_API_TOKEN" \
  --custom-model-id qwen/qwen3.5-9b
```

`--custom-model-id` کلید مدل را همان‌گونه که LM Studio برمی‌گرداند (برای مثال `qwen/qwen3.5-9b`) و بدون
پیشوند ارائه‌دهندهٔ `lmstudio/` دریافت می‌کند. برای سرورهای دارای احراز هویت، `--lmstudio-api-key` را ارسال کنید (یا `LM_API_TOKEN` را تنظیم کنید)؛
برای سرورهای بدون احراز هویت آن را حذف کنید تا OpenClaw در عوض یک نشانگر محلیِ غیرمحرمانه ذخیره کند.
`--custom-api-key` همچنان برای سازگاری پذیرفته می‌شود، اما `--lmstudio-api-key` ترجیح داده می‌شود.

این کار `models.providers.lmstudio` را می‌نویسد و مدل پیش‌فرض را روی `lmstudio/<custom-model-id>` تنظیم می‌کند.
ارائهٔ کلید API همچنین پروفایل احراز هویت `lmstudio:default` را می‌نویسد.

راه‌اندازی تعاملی می‌تواند علاوه بر این، طول زمینهٔ بارگذاری ترجیحی را بپرسد و آن را روی
مدل‌های کشف‌شده‌ای که در پیکربندی ذخیره می‌کند اعمال کند.

## پیکربندی

### سازگاری مصرف در استریم

LM Studio همیشه یک شیء `usage` با ساختار OpenAI را در پاسخ‌های استریم‌شده منتشر نمی‌کند. OpenClaw
در عوض، تعداد توکن‌ها را از فرادادهٔ سبک llama.cpp یعنی `timings.prompt_n` / `timings.predicted_n`
بازیابی می‌کند. هر نقطهٔ پایانی سازگار با OpenAI که به‌عنوان نقطهٔ پایانی محلی (میزبان loopback) شناسایی شود، همین
سازوکار جایگزین را دریافت می‌کند که بک‌اندهای محلی دیگر مانند vLLM، SGLang، llama.cpp، LocalAI، Jan، TabbyAPI
و text-generation-webui را نیز پوشش می‌دهد.

### سازگاری تفکر

وقتی کشف `/api/v1/models` در LM Studio گزینه‌های استدلال مختص مدل را گزارش می‌کند، OpenClaw
مقادیر متناظر `reasoning_effort` (`none`، `minimal`، `low`، `medium`، `high`، `xhigh`) را در
فرادادهٔ سازگاری مدل ارائه می‌کند. برخی نسخه‌های LM Studio یک گزینهٔ دودویی UI (`allowed_options: ["off",
"on"]`) را اعلام می‌کنند، اما آن مقادیر تحت‌اللفظی را در `/v1/chat/completions` رد می‌کنند؛ OpenClaw پیش از
ارسال درخواست‌ها، این ساختار دودویی را به مقیاس شش‌سطحی عادی‌سازی می‌کند؛ از جمله برای پیکربندی‌های ذخیره‌شدهٔ قدیمی‌تر که
هنوز نگاشت‌های استدلال `off`/`on` را دارند.

### پیکربندی صریح

```json5
{
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        apiKey: "${LM_API_TOKEN}",
        api: "openai-completions",
        models: [
          {
            id: "qwen/qwen3-coder-next",
            name: "Qwen 3 Coder Next",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            maxTokens: 8192,
          },
        ],
      },
    },
  },
}
```

### غیرفعال کردن پیش‌بارگذاری

LM Studio از بارگذاری به‌هنگام (JIT) مدل پشتیبانی می‌کند و مدل‌ها را در نخستین درخواست بارگذاری می‌کند. OpenClaw
به‌طور پیش‌فرض مدل‌ها را از طریق نقطهٔ پایانی بومی بارگذاری LM Studio پیش‌بارگذاری می‌کند که هنگام
غیرفعال بودن JIT مفید است. برای اینکه JIT، TTL بیکاری و رفتار تخلیهٔ خودکار LM Studio چرخهٔ عمر مدل را مدیریت کنند،
مرحلهٔ پیش‌بارگذاری OpenClaw را غیرفعال کنید:

```json5
{
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://localhost:1234/v1",
        api: "openai-completions",
        params: { preload: false },
        models: [{ id: "qwen/qwen3.5-9b" }],
      },
    },
  },
}
```

### میزبان LAN یا tailnet

از نشانی قابل‌دسترسی میزبان LM Studio استفاده کنید، `/v1` را نگه دارید و مطمئن شوید LM Studio روی آن دستگاه
فراتر از loopback متصل شده است:

```json5
{
  models: {
    providers: {
      lmstudio: {
        baseUrl: "http://gpu-box.local:1234/v1",
        apiKey: "lmstudio",
        api: "openai-completions",
        models: [{ id: "qwen/qwen3.5-9b" }],
      },
    },
  },
}
```

`lmstudio` به‌طور خودکار به نقطهٔ پایانی پیکربندی‌شدهٔ خود برای درخواست‌های مدل اعتماد می‌کند؛ از جمله میزبان‌های loopback،
LAN و tailnet (به‌جز مبدأهای فراداده/link-local). هر ورودی سفارشی/محلیِ ارائه‌دهندهٔ سازگار با OpenAI نیز
همین اعتماد به مبدأ دقیق را دریافت می‌کند. درخواست‌ها به میزبان خصوصی یا درگاه دیگری همچنان
به `models.providers.<id>.request.allowPrivateNetwork: true` نیاز دارند؛ برای انصراف از
اعتماد پیش‌فرض، آن را روی `false` تنظیم کنید.

## عیب‌یابی

### LM Studio شناسایی نمی‌شود

مطمئن شوید LM Studio در حال اجرا است:

```bash
lms server start --port 1234
```

اگر احراز هویت فعال است، `LM_API_TOKEN` را نیز تنظیم کنید. در دسترس بودن API را بررسی کنید:

```bash
curl http://localhost:1234/api/v1/models
```

### خطاهای احراز هویت (HTTP 401)

- بررسی کنید که `LM_API_TOKEN` با کلید پیکربندی‌شده در LM Studio مطابقت داشته باشد.
- به [احراز هویت LM Studio](https://lmstudio.ai/docs/developer/core/authentication) مراجعه کنید.
- اگر سرور به احراز هویت نیاز ندارد، هنگام راه‌اندازی کلید را خالی بگذارید.

## مرتبط

- [انتخاب مدل](/fa/concepts/model-providers)
- [Ollama](/fa/providers/ollama)
- [مدل‌های محلی](/fa/gateway/local-models)
