---
read_when:
    - در حال پیکربندی Plugin حافظهٔ `memory-lancedb` هستید
    - حافظهٔ بلندمدت مبتنی بر LanceDB با بازیابی خودکار یا ثبت خودکار می‌خواهید
    - در حال استفاده از تعبیه‌سازی‌های محلی سازگار با OpenAI مانند Ollama هستید
sidebarTitle: Memory LanceDB
summary: Plugin رسمی و خارجی حافظهٔ LanceDB، از جمله embeddingهای محلی سازگار با Ollama را پیکربندی کنید
title: حافظه LanceDB
x-i18n:
    generated_at: "2026-07-27T15:39:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bdb7208925ac6c76430ee36dfcd9733041530e0f2ee175950b3cdb8010d67b24
    source_path: plugins/memory-lancedb.md
    workflow: 16
---

`memory-lancedb` یک Plugin خارجی رسمی است که حافظهٔ بلندمدت را با قابلیت جست‌وجوی برداری در
LanceDB ذخیره می‌کند. این Plugin می‌تواند پیش از نوبت مدل، حافظه‌های مرتبط را به‌طور خودکار
بازیابی کند و پس از پاسخ، واقعیت‌های مهم را به‌طور خودکار ثبت کند.

از آن برای یک پایگاه دادهٔ برداری محلی، یک نقطهٔ پایانی تعبیه‌سازی سازگار با OpenAI، یا
یک مخزن حافظه خارج از بک‌اند حافظهٔ داخلی پیش‌فرض استفاده کنید.

## نصب

```bash
openclaw plugins install @openclaw/memory-lancedb
```

این Plugin در npm منتشر شده است و در تصویر زمان اجرای OpenClaw گنجانده نمی‌شود.
نصب آن ورودی Plugin را می‌نویسد، آن را فعال می‌کند و
`plugins.slots.memory` را به `memory-lancedb` تغییر می‌دهد. اگر در حال حاضر Plugin دیگری
مالک جایگاه حافظه باشد، آن Plugin با یک هشدار غیرفعال می‌شود.

<Note>
Pluginهای همراه مانند `memory-wiki` می‌توانند در کنار `memory-lancedb` اجرا شوند،
اما در هر لحظه فقط یک Plugin مالک جایگاه حافظهٔ فعال است.
</Note>

<Note>
`memory_recall` متعلق به LanceDB مجوز محافظت‌شدهٔ رونوشت خصوصی
مورد استفادهٔ `memory.search.rememberAcrossConversations` را دریافت نمی‌کند. از
`autoRecall` متعلق به LanceDB یا ابزار `memory_recall` آن از طریق
[Active Memory پیشرفته](/fa/concepts/active-memory#lancedb-memory) استفاده کنید.
`openclaw doctor` گزارش می‌دهد که چه زمانی «به‌خاطرسپاری در میان مکالمه‌ها» با
ارائه‌دهندهٔ حافظهٔ فعلی در دسترس نیست.
</Note>

## شروع سریع

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb",
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "openai",
            model: "text-embedding-3-small",
          },
          autoRecall: true,
          autoCapture: false,
        },
      },
    },
  },
}
```

پس از تغییر پیکربندی Plugin، Gateway را راه‌اندازی مجدد کنید، سپس بررسی کنید که بارگذاری شده است:

```bash
openclaw gateway restart
openclaw plugins list
```

## پیکربندی تعبیه‌سازی

`embedding` الزامی است و باید دست‌کم یک فیلد داشته باشد. مقدار پیش‌فرض `provider`
برابر `openai` و مقدار پیش‌فرض `model` برابر `text-embedding-3-small` است.

| فیلد                  | نوع          | توضیحات                                                                    |
| ---------------------- | ------------- | ------------------------------------------------------------------------ |
| `embedding.provider`   | رشته        | شناسهٔ آداپتور، برای مثال `openai`، `github-copilot`، `ollama`. پیش‌فرض: `openai`. |
| `embedding.model`      | رشته        | پیش‌فرض: `text-embedding-3-small`.                                        |
| `embedding.apiKey`     | رشته        | اختیاری؛ از بسط `${ENV_VAR}` پشتیبانی می‌کند.                               |
| `embedding.baseUrl`    | رشته        | اختیاری؛ از بسط `${ENV_VAR}` پشتیبانی می‌کند.                               |
| `embedding.dimensions` | عدد صحیح (>=1) | برای مدل‌هایی که در جدول داخلی نیستند الزامی است (پایین را ببینید).               |

دو مسیر درخواست وجود دارد:

- **مسیر آداپتور ارائه‌دهنده** (پیش‌فرض): `embedding.provider` را تنظیم و
  `embedding.apiKey`/`embedding.baseUrl` را حذف کنید. Plugin، پروفایل احراز هویت
  پیکربندی‌شدهٔ ارائه‌دهنده، متغیر محیطی، یا
  `models.providers.<provider>.apiKey` را از طریق همان آداپتورهای تعبیه‌سازی حافظه‌ای
  که `memory-core` استفاده می‌کند، تفکیک می‌کند. این مسیر برای `github-copilot`، `ollama`
  و هر ارائه‌دهندهٔ همراه دیگری است که از تعبیه‌سازی پشتیبانی می‌کند.
- **مسیر مستقیم کلاینت سازگار با OpenAI**: `embedding.provider` را تنظیم‌نشده
  بگذارید (یا `"openai"`) و `embedding.apiKey` را همراه با `embedding.baseUrl` تنظیم کنید. از این مسیر
  برای یک نقطهٔ پایانی خام تعبیه‌سازی سازگار با OpenAI استفاده کنید که آداپتور
  ارائه‌دهندهٔ همراه ندارد.

OAuth مربوط به OpenAI Codex / ChatGPT یک اعتبارنامهٔ تعبیه‌سازی OpenAI Platform نیست.
برای تعبیه‌سازی‌های OpenAI از یک پروفایل احراز هویت کلید API در OpenAI، `OPENAI_API_KEY`، یا
`models.providers.openai.apiKey` استفاده کنید. کاربرانی که فقط OAuth دارند باید ارائه‌دهندهٔ دیگری
با قابلیت تعبیه‌سازی، مانند `github-copilot` یا `ollama`، انتخاب کنند.

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "github-copilot",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

برخی نقاط پایانی تعبیه‌سازی سازگار با OpenAI پارامتر `encoding_format` را
رد می‌کنند؛ برخی دیگر آن را نادیده می‌گیرند و همیشه `number[]` را برمی‌گردانند. `memory-lancedb`
در درخواست‌ها `encoding_format` را حذف می‌کند و پاسخ‌های آرایهٔ اعشاری یا
float32 کدگذاری‌شده با base64 را می‌پذیرد؛ بنابراین هر دو شکل پاسخ بدون پیکربندی کار می‌کنند.

### ابعاد

OpenClaw فقط برای `text-embedding-3-small` (1536) و
`text-embedding-3-large` (3072) بُعد داخلی دارد. هر مدل دیگری به
`embedding.dimensions` صریح نیاز دارد تا LanceDB بتواند ستون برداری را ایجاد کند؛ برای مثال
`embedding-3` از ZhiPu با 2048 بُعد:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            apiKey: "${ZHIPU_API_KEY}",
            baseUrl: "https://open.bigmodel.cn/api/paas/v4",
            model: "embedding-3",
            dimensions: 2048,
          },
        },
      },
    },
  },
}
```

## تعبیه‌سازی‌های Ollama

از مسیر آداپتور ارائه‌دهندهٔ همراه Ollama استفاده کنید (`embedding.provider: "ollama"`).
این مسیر نقطهٔ پایانی بومی `/api/embed` متعلق به Ollama را فراخوانی می‌کند و از همان قواعد احراز هویت/نشانی
پایهٔ ارائه‌دهندهٔ [Ollama](/fa/providers/ollama) پیروی می‌کند.

```json5
{
  plugins: {
    slots: {
      memory: "memory-lancedb",
    },
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          embedding: {
            provider: "ollama",
            baseUrl: "http://127.0.0.1:11434",
            model: "mxbai-embed-large",
            dimensions: 1024,
          },
          recallMaxChars: 400,
          autoRecall: true,
          autoCapture: false,
        },
      },
    },
  },
}
```

`mxbai-embed-large` در جدول داخلی ابعاد نیست؛ بنابراین `dimensions`
الزامی است. برای مدل‌های تعبیه‌سازی محلی کوچک، اگر سرور محلی خطاهای طول زمینه
برمی‌گرداند، `recallMaxChars` را کاهش دهید.

## محدودیت‌های بازیابی و ثبت

| تنظیم           | پیش‌فرض | محدوده                        | کاربرد                                                 |
| ----------------- | ------- | ---------------------------- | ---------------------------------------------------------- |
| `recallMaxChars`  | `1000`  | 100-10000                    | متنی که برای بازیابی به API تعبیه‌سازی ارسال می‌شود.                 |
| `captureMaxChars` | `500`   | 100-10000                    | طول پیام واجد شرایط ثبت خودکار.                  |
| `customTriggers`  | `[]`    | 0-50 مورد، هرکدام <=100 نویسه | عبارت‌های تحت‌اللفظی که باعث می‌شوند ثبت خودکار یک پیام را بررسی کند. |

`recallMaxChars` اندازهٔ پرس‌وجوی بازیابی خودکار `before_prompt_build`،
ابزار `memory_recall`، مسیر پرس‌وجوی `memory_forget` و `openclaw ltm
search` را محدود می‌کند. بازیابی خودکار، آخرین پیام کاربر در نوبت را تعبیه می‌کند و
فقط وقتی هیچ پیام کاربری وجود ندارد به پرامپت کامل بازمی‌گردد؛ در نتیجه فرادادهٔ کانال
و بلوک‌های بزرگ پرامپت از درخواست تعبیه‌سازی کنار گذاشته می‌شوند.

`captureMaxChars` تعیین می‌کند که آیا پیام کاربر از رویداد `agent_end`
نوبت به‌اندازهٔ کافی کوتاه است تا برای ثبت خودکار در نظر گرفته شود؛ این تنظیم بر
پرس‌وجوهای بازیابی اثری ندارد.

`customTriggers` عبارت‌های تحت‌اللفظی ثبت خودکار را بدون عبارت منظم اضافه می‌کند. محرک‌های
داخلی، عبارت‌های رایج حافظه در زبان‌های انگلیسی، چکی، چینی، ژاپنی و کره‌ای را
پوشش می‌دهند (`remember`، `prefer`، `记住`، `覚えて`، `기억해` و موارد مشابه).

ثبت خودکار همچنین متن‌هایی را که شبیه فرادادهٔ پوش/انتقال،
محموله‌های تزریق پرامپت، یا زمینهٔ `<relevant-memories>` ازپیش‌تزریق‌شده هستند رد می‌کند
و در هر نوبت عامل حداکثر 3 حافظه ثبت می‌کند.

هر حافظه متعلق به یک عامل است. بازیابی، تشخیص موارد تکراری، ثبت،
فهرست‌کردن، پرس‌وجوهای خام و حذف، همگی پیش از بازگرداندن یا
تغییر ردیف‌ها آن مالک را اعمال می‌کنند. عاملی که در ورودی `agents.entries.*`
خود دارای `memory.search.enabled: false` است، یا جست‌وجوی سطح‌بالای غیرفعال را به ارث می‌برد، هیچ‌یک از ابزارهای `memory_recall`، `memory_store`
یا `memory_forget` را نیز دریافت نمی‌کند و در بازیابی یا
ثبت خودکار مشارکت ندارد، حتی وقتی پرچم‌های سطح Plugin یعنی `autoRecall`/`autoCapture` روشن باشند.

## فرمان‌ها

`memory-lancedb` هر زمان که نصب باشد فضای نام CLI یعنی `ltm` را ثبت می‌کند
(نه فقط زمانی که مالک جایگاه حافظهٔ فعال است):

```bash
openclaw ltm list [--agent <id>] [--limit <n>] [--order-by-created-at]
openclaw ltm search <query> [--agent <id>] [--limit <n>]
openclaw ltm stats [--agent <id>]
```

`ltm query` یک پرس‌وجوی غیربرداری را مستقیماً روی جدول LanceDB اجرا می‌کند:

```bash
openclaw ltm query --agent research --cols id,text,createdAt --limit 20
openclaw ltm query --filter "category = 'preference'" --order-by createdAt:desc
```

| پرچم                              | پیش‌فرض                                 | توضیحات                                                                                                                                     |
| --------------------------------- | --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| `--agent <id>`                    | عامل پیش‌فرض پیکربندی‌شده                | فضای نام خصوصی عامل را انتخاب می‌کند. در `list`، `search`، `query` و `stats` در دسترس است.                                                 |
| `--cols <columns>`                | `id,text,importance,category,createdAt` | فهرست مجاز ستون‌ها با جداسازی ویرگول.                                                                                                         |
| `--filter <condition>`            | هیچ‌کدام                                    | یک مقایسه روی یک ستون خروجی، مانند `category = 'preference'` یا `importance >= 0.8`. مقادیر رشته‌ای باید داخل علامت نقل‌قول باشند.             |
| `--limit <n>`                     | `10`                                    | عدد صحیح مثبت.                                                                                                                         |
| `--order-by <column>:<asc\|desc>` | هیچ‌کدام                                    | پس از اجرای فیلتر در حافظه مرتب می‌شود؛ ستون مرتب‌سازی به‌طور خودکار به نگاشت افزوده می‌شود و اگر درخواست نشده باشد از خروجی حذف می‌شود. |

عامل‌ها سه ابزار از Plugin حافظهٔ فعال دریافت می‌کنند:

- `memory_recall`: جست‌وجوی برداری در حافظه‌های ذخیره‌شده.
- `memory_store`: ذخیرهٔ یک واقعیت، ترجیح، تصمیم یا موجودیت (متنی را
  که شبیه محمولهٔ تزریق پرامپت است رد می‌کند؛ ذخیره‌سازی موارد تقریباً تکراری را نادیده می‌گیرد).
- `memory_forget`: حذف بر اساس `memoryId`، یا بر اساس `query` (یک
  تطابق منفرد با امتیاز بالاتر از 90% را به‌طور خودکار حذف می‌کند؛ در غیر این صورت شناسه‌های نامزد را برای رفع ابهام فهرست می‌کند).

## ذخیره‌سازی

مسیر پیش‌فرض داده‌های LanceDB برابر `~/.openclaw/memory/lancedb` است. آن را با `dbPath` بازنویسی کنید:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          dbPath: "~/.openclaw/memory/lancedb",
          embedding: {
            apiKey: "${OPENAI_API_KEY}",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

Plugin یک جدول LanceDB نگه می‌دارد و یک مالک عامل نرمال‌شده را در هر
ردیف ذخیره می‌کند. این یک مرز ذخیره‌سازی است، نه فیلتری پس از جست‌وجو: مالکیت عامل
پیش از رتبه‌بندی برداری اعمال می‌شود و در گزاره‌های فهرست، پرس‌وجو، شمارش و حذف
گنجانده می‌شود. `ltm query --filter` یک مقایسهٔ اعتبارسنجی‌شده روی
ستون‌های خروجی عمومی می‌پذیرد. مخزن آن مقایسه را جدا از
گزارهٔ اجباری مالک می‌سازد؛ بنابراین یک فیلتر نمی‌تواند پرس‌وجو را به عامل دیگری
گسترش دهد.

پایگاه‌های داده‌ای که پیش از مالکیت به‌ازای هر عامل ایجاد شده‌اند، منشأ قابل‌اعتمادی برای ردیف‌ها ندارند.
هنگام ارتقا، `openclaw doctor --fix` آن ردیف‌های قدیمی را یک‌بار به
عامل پیش‌فرض پیکربندی‌شده اختصاص می‌دهد. دسترسی زمان اجرا تا تکمیل آن مهاجرت
به‌صورت بسته ناموفق می‌شود؛ عامل‌های دیگر هرگز ردیف‌های اشتراکی قدیمی را به ارث نمی‌برند.

`storageOptions` جفت‌های کلید/مقدار رشته‌ای را برای بک‌اندهای ذخیره‌سازی LanceDB
(برای مثال، ذخیره‌سازی شیء سازگار با S3) می‌پذیرد و از بسط `${ENV_VAR}` پشتیبانی می‌کند:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        enabled: true,
        config: {
          dbPath: "s3://memory-bucket/openclaw",
          storageOptions: {
            access_key: "${AWS_ACCESS_KEY_ID}",
            secret_key: "${AWS_SECRET_ACCESS_KEY}",
            endpoint: "${AWS_ENDPOINT_URL}",
          },
          embedding: {
            apiKey: "${OPENAI_API_KEY}",
            model: "text-embedding-3-small",
          },
        },
      },
    },
  },
}
```

## وابستگی‌های زمان اجرا و پشتیبانی از پلتفرم‌ها

`memory-lancedb` به بسته بومی `@lancedb/lancedb` وابسته است که مالکیت آن با
بسته Plugin است (نه توزیع اصلی OpenClaw). راه‌اندازی Gateway وابستگی‌های
Plugin را ترمیم نمی‌کند؛ اگر وابستگی بومی موجود نباشد یا بارگذاری آن ناموفق باشد،
بسته Plugin را دوباره نصب یا به‌روزرسانی کنید و Gateway را مجدداً راه‌اندازی کنید.

`@lancedb/lancedb` برای `darwin-x64` (مک Intel) بیلد بومی منتشر نمی‌کند.
در آن پلتفرم، Plugin هنگام بارگذاری ثبت می‌کند که LanceDB در دسترس نیست؛
از بک‌اند حافظه پیش‌فرض استفاده کنید، Gateway را روی یک پلتفرم/معماری
پشتیبانی‌شده اجرا کنید، یا `memory-lancedb` را غیرفعال کنید.

## عیب‌یابی

### طول ورودی از طول زمینه فراتر می‌رود

مدل تعبیه‌سازی، پرس‌وجوی بازیابی را رد کرد:

```text
memory-lancedb: بازیابی ناموفق بود: خطا: 400 طول ورودی از طول زمینه فراتر می‌رود
```

`recallMaxChars` را کاهش دهید، سپس Gateway را مجدداً راه‌اندازی کنید:

```json5
{
  plugins: {
    entries: {
      "memory-lancedb": {
        config: {
          recallMaxChars: 400,
        },
      },
    },
  },
}
```

برای Ollama، همچنین با استفاده از نقطه پایانی بومی تعبیه‌سازی آن بررسی کنید که سرور
تعبیه‌سازی از میزبان Gateway قابل دسترسی باشد:

```bash
curl http://127.0.0.1:11434/api/embed \
  -H "Content-Type: application/json" \
  -d '{"model":"mxbai-embed-large","input":"hello"}'
```

### مدل تعبیه‌سازی پشتیبانی‌نشده

بدون `embedding.dimensions`، فقط ابعاد داخلی تعبیه‌سازی OpenAI
شناخته‌شده هستند (`text-embedding-3-small`، `text-embedding-3-large`). برای هر مدل
دیگر، `embedding.dimensions` را روی اندازه برداری که آن مدل گزارش می‌کند تنظیم کنید.

### Plugin بارگذاری می‌شود، اما هیچ حافظه‌ای نمایش داده نمی‌شود

تأیید کنید که `plugins.slots.memory` به `memory-lancedb` اشاره می‌کند، سپس اجرا کنید:

```bash
openclaw ltm stats
openclaw ltm search "recent preference"
```

اگر `autoCapture` غیرفعال باشد، Plugin همچنان حافظه‌های موجود را بازیابی می‌کند،
اما حافظه‌های جدید را به‌طور خودکار ذخیره نمی‌کند. از ابزار `memory_store` استفاده
کنید یا `autoCapture` را فعال کنید.

## مرتبط

- [نمای کلی حافظه](/fa/concepts/memory)
- [Active Memory](/fa/concepts/active-memory)
- [جست‌وجوی حافظه](/fa/concepts/memory-search)
- [ویکی حافظه](/fa/plugins/memory-wiki)
- [Ollama](/fa/providers/ollama)
