---
read_when:
    - می‌خواهید استنتاج متنی را به‌صورت محلی و بدون کلید API یا سرور مدل انجام دهید
    - می‌خواهید تعبیه‌های جست‌وجوی حافظه از یک مدل محلی GGUF باشند
    - شما در حال پیکربندی `memory.search.provider = "local"` هستید
    - به Plugin مربوط به OpenClaw نیاز دارید که مالک زمان‌اجرای node-llama-cpp است
sidebarTitle: llama.cpp Provider
summary: استنتاج محلی متن GGUF و تعبیه‌سازی‌های حافظه را در OpenClaw با llama.cpp اجرا کنید
title: ارائه‌دهنده llama.cpp
x-i18n:
    generated_at: "2026-07-27T15:52:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 88e6d66943adcbc602421b8cc00359b3ed87357194c3ffaa845c1db7fbcd9c38
    source_path: plugins/llama-cpp.md
    workflow: 16
---

`llama-cpp`، Plugin رسمی ارائه‌دهنده خارجی برای استنتاج متن و تعبیه‌سازی محلی و درون‌پردازه‌ای GGUF است. این افزونه ارائه‌دهنده متن `llama-cpp` و ارائه‌دهنده تعبیه‌سازی `local` را ثبت می‌کند و مالک زمان‌اجرای بومی `node-llama-cpp` است.

پیش از استفاده از استنتاج محلی یا تعبیه‌سازی‌های حافظه محلی، آن را نصب کنید:

```bash
openclaw plugins install @openclaw/llama-cpp-provider
```

بسته اصلی npm با نام `openclaw` شامل `node-llama-cpp` نیست. نگه‌داشتن وابستگی بومی در این Plugin مانع از آن می‌شود که به‌روزرسانی‌های عادی npm در OpenClaw، زمان‌اجرای نصب‌شده به‌صورت دستی درون پوشه بسته OpenClaw را حذف کنند.

## استنتاج متن محلی

هنگام راه‌اندازی تعاملی، **مدل محلی (llama.cpp)** را انتخاب کنید. OpenClaw پیش از دانلود مدل پیش‌فرض سؤال می‌کند:

`hf:bartowski/Qwen_Qwen3-4B-Instruct-2507-GGUF/Qwen_Qwen3-4B-Instruct-2507-Q4_K_M.gguf`

حجم فایل Qwen3 4B Instruct 2507 Q4_K_M حدود 2.5 GB است. تقریباً 3 GB حافظه RAM برای وزن‌های مدل، به‌علاوه فضای لازم برای زمینه و سربار زمان‌اجرای OpenClaw در نظر بگیرید. اندازه زمینه پیش‌فرض به‌طور خودکار با سقف 8,192 توکن تنظیم می‌شود تا استفاده از آن روی دستگاه‌های دارای 8 GB حافظه عملی باقی بماند. فقط هنگامی زمینه بزرگ‌تری پیکربندی کنید که دستگاه حافظه کافی داشته باشد.

بررسی شناسایی هنگام راه‌اندازی فقط خواندنی است. llama.cpp فقط زمانی به‌طور خودکار پیشنهاد می‌شود که فایل GGUF پیش‌فرض یا پیکربندی‌شده از قبل در حافظه نهان مدل موجود باشد؛ در فرایند شناسایی هرگز چیزی دانلود نمی‌شود. Ollama و LM Studio همچنان گزینه‌های جداگانه سرویس محلی هستند و جریان‌های شناسایی مختص خود را حفظ می‌کنند. انتخاب دستی llama.cpp مسیری است که برای دانلود مدل پیش‌فرض درخواست تأیید می‌کند.

ارائه‌دهنده از الگوی گفت‌وگوی تعبیه‌شده مدل GGUF و فراخوانی بومی توابع node-llama-cpp استفاده می‌کند. متن توکن‌به‌توکن جریان می‌یابد. فراخوانی‌های ابزار برای اجرا به OpenClaw بازگردانده می‌شوند و درون node-llama-cpp اجرا نمی‌شوند.

### استفاده از یک مدل GGUF دیگر

مدلی به `models.providers.llama-cpp` اضافه کنید. یک مسیر محلی یا URI کامل فایل `hf:` را در `params.modelPath` قرار دهید:

```json5
{
  models: {
    mode: "merge",
    providers: {
      "llama-cpp": {
        baseUrl: "local://llama-cpp",
        api: "openai-completions",
        params: {
          modelCacheDir: "~/.node-llama-cpp/models",
        },
        models: [
          {
            id: "my-local-model",
            name: "My local GGUF",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 8192,
            maxTokens: 2048,
            params: {
              modelPath: "~/Models/my-model.Q4_K_M.gguf",
              contextSize: 8192,
            },
            compat: { supportsTools: true },
          },
        ],
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "llama-cpp/my-local-model" },
    },
  },
}
```

استنتاج هرگز مدل گم‌شده‌ای را به‌طور ضمنی دانلود نمی‌کند. برای یک URI سفارشی `hf:`، ابتدا فایل GGUF را در `modelCacheDir` دانلود کنید. شناسایی از تحلیل‌گر فقط خواندنی حافظه نهان خود node-llama-cpp استفاده می‌کند که نام‌گذاری مخزن، شاخه و فایل‌های چندبخشی را نیز در بر می‌گیرد.

## پیکربندی تعبیه‌سازی حافظه

مقدار `memory.search.provider` را روی `local` تنظیم کنید:

```json5
{
  memory: {
    search: {
      provider: "local",
      local: {
        modelPath: "hf:ggml-org/embeddinggemma-300m-qat-q8_0-GGUF/embeddinggemma-300m-qat-Q8_0.gguf",
      },
    },
  },
}
```

مقدار پیش‌فرض `local.modelPath`، URI مربوط به `hf:` است که در بالا نشان داده شد (`embeddinggemma-300m-qat-Q8_0.gguf`). برای استفاده از مدلی دیگر، آن را به یک URI متفاوت `hf:` یا فایل محلی `.gguf` اشاره دهید. `local.modelCacheDir` محل ذخیره مدل‌های دانلودشده در حافظه نهان را بازنویسی می‌کند (پیش‌فرض: `~/.node-llama-cpp/models`) و `local.contextSize` یک عدد صحیح یا `"auto"` را می‌پذیرد.

هنگامی که `local.contextSize` عددی باشد، ارائه‌دهنده این الزام را نیز به سازوکار جای‌دهی خودکار لایه‌های GPU در node-llama-cpp می‌دهد. به این ترتیب، node-llama-cpp می‌تواند مدل و زمینه تعبیه‌سازی را با هم جای دهد و در عین حال بررسی‌های ایمنی حافظه خود را حفظ کند. با `"auto"`، سازوکار جای‌دهی خودکار عادی node-llama-cpp حفظ می‌شود.

## زمان‌اجرای بومی

برای روان‌ترین مسیر نصب بومی از Node 24 استفاده کنید. در نسخه‌های منبعی که از pnpm استفاده می‌کنند، ممکن است لازم باشد وابستگی بومی را تأیید و دوباره بسازید:

```bash
pnpm approve-builds
pnpm rebuild node-llama-cpp
```

## عیب‌یابی زمان‌اجرای حافظه

پس از بارگذاری ارائه‌دهنده، `openclaw memory status --deep` را اجرا کنید تا بک‌اند و بیلد انتخاب‌شده، نام دستگاه‌ها، لایه‌های واگذارشده به GPU، اندازه زمینه درخواستی و آخرین تصویر مشاهده‌شده از VRAM یا حافظه یکپارچه را بررسی کنید. مقادیر VRAM شامل برچسب زمانی مشاهده هستند، زیرا خواندن غیرفعال وضعیت باعث بارگذاری مجدد مدل یا نظرسنجی از دستگاه نمی‌شود.

همین اطلاعات آخرین وضعیت شناخته‌شده ممکن است در `openclaw doctor` نیز نمایش داده شوند، مشروط بر اینکه Gateway در حال اجرا قبلاً از ارائه‌دهنده محلی استفاده کرده باشد. فرمان عادی وضعیت یا doctor صرفاً برای جمع‌آوری اطلاعات عیب‌یابی، مدلی را بارگذاری نمی‌کند.

## رفع اشکال

اگر `node-llama-cpp` موجود نباشد یا بارگذاری آن ناموفق باشد، OpenClaw خطا را همراه با موارد زیر گزارش می‌کند:

1. Plugin را نصب کنید: `openclaw plugins install @openclaw/llama-cpp-provider`.
2. برای نصب‌ها و به‌روزرسانی‌های بومی از Node 24 استفاده کنید.
3. از یک نسخه منبع pnpm: ابتدا `pnpm approve-builds` و سپس `pnpm rebuild node-llama-cpp` را اجرا کنید.

برای استنتاج محلی بدون وابستگی بومی درون‌پردازه‌ای، به‌جای آن از ارائه‌دهنده Ollama یا LM Studio استفاده کنید. برای تعبیه‌سازی محلی کم‌دردسرتر، در عوض `memory.search.provider` را روی یک ارائه‌دهنده تعبیه‌سازی راه‌دور مانند `lmstudio`، `ollama`، `openai` یا `voyage` تنظیم کنید.
