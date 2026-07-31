---
read_when:
    - می‌خواهید OpenClaw را با یک سرور محلی SGLang اجرا کنید
    - شما endpointهای سازگار با OpenAI در مسیر `/v1` را با مدل‌های خودتان می‌خواهید
summary: اجرای OpenClaw با SGLang (سرور خودمیزبان سازگار با OpenAI)
title: SGLang
x-i18n:
    generated_at: "2026-07-27T14:33:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 54a7805315a7d65fdd2c7c9b6836aa2faccc88db7802cce0ba8c2d4a1aac9d65
    source_path: providers/sglang.md
    workflow: 16
---

SGLang مدل‌های دارای وزن باز را از طریق یک API ‏HTTP سازگار با OpenAI ارائه می‌کند. OpenClaw با استفاده از خانواده ارائه‌دهنده `openai-completions` و کشف خودکار مدل‌های موجود به SGLang متصل می‌شود.

| ویژگی                     | مقدار                                                        |
| ------------------------- | ------------------------------------------------------------ |
| شناسه ارائه‌دهنده         | `sglang`                                                     |
| Plugin                    | همراه، `enabledByDefault: true`                            |
| متغیر محیطی احراز هویت    | `SGLANG_API_KEY` (اگر سرور احراز هویت ندارد، هر مقدار غیرخالی) |
| پرچم راه‌اندازی اولیه     | `--auth-choice sglang`                                       |
| API                       | سازگار با OpenAI ‏(`openai-completions`)                     |
| URL پایه پیش‌فرض          | `http://127.0.0.1:30000/v1`                                  |
| جای‌نگهدار مدل پیش‌فرض    | `sglang/Qwen/Qwen3-8B`                                       |
| میزان استفاده از استریم   | بله (`supportsStreamingUsage: true`)                         |
| قیمت‌گذاری                | به‌عنوان رایگان خارجی علامت‌گذاری شده است (`modelPricing.external: false`)        |

همچنین وقتی با `SGLANG_API_KEY` اعلام مشارکت می‌کنید، OpenClaw مدل‌های موجود را به‌صورت **خودکار کشف می‌کند**. وقتی یک URL پایه سفارشی SGLang را نیز پیکربندی می‌کنید، برای پویا نگه‌داشتن کشف، از `sglang/*` در `agents.defaults.models` استفاده کنید. بخش [کشف مدل (ارائه‌دهنده ضمنی)](#model-discovery-implicit-provider) را در ادامه ببینید.

## شروع به کار

<Steps>
  <Step title="راه‌اندازی SGLang">
    SGLang را با یک سرور سازگار با OpenAI اجرا کنید. URL پایه شما باید نقطه‌های پایانی
    `/v1` را ارائه کند (برای مثال `/v1/models`، `/v1/chat/completions`). SGLang
    معمولاً روی نشانی زیر اجرا می‌شود:

    - `http://127.0.0.1:30000/v1`

  </Step>
  <Step title="تنظیم یک کلید API">
    اگر احراز هویت روی سرور شما پیکربندی نشده باشد، هر مقداری کار می‌کند:

    ```bash
    export SGLANG_API_KEY="sglang-local"
    ```

  </Step>
  <Step title="اجرای راه‌اندازی اولیه یا تنظیم مستقیم مدل">
    ```bash
    openclaw onboard
    ```

    یا مدل را به‌صورت دستی پیکربندی کنید:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "sglang/your-model-id" },
        },
      },
    }
    ```

  </Step>
</Steps>

## کشف مدل (ارائه‌دهنده ضمنی)

وقتی `SGLANG_API_KEY` تنظیم شده باشد (یا یک نمایه احراز هویت وجود داشته باشد) و
`models.providers.sglang` را تعریف **نکرده باشید**، OpenClaw نشانی زیر را پرس‌وجو می‌کند:

- `GET http://127.0.0.1:30000/v1/models`

و شناسه‌های بازگشتی را به ورودی‌های مدل تبدیل می‌کند.

<Note>
اگر `models.providers.sglang` را صریحاً تنظیم کنید، OpenClaw به‌طور پیش‌فرض از
مدل‌های اعلام‌شده شما استفاده می‌کند. وقتی می‌خواهید OpenClaw نقطه پایانی
`/models` آن ارائه‌دهنده پیکربندی‌شده را پرس‌وجو و همه مدل‌های SGLang
اعلام‌شده را شامل کند، `"sglang/*": {}` را به `agents.defaults.models` اضافه کنید.
</Note>

## پیکربندی صریح (مدل‌های دستی)

در موارد زیر از پیکربندی صریح استفاده کنید:

- SGLang روی میزبان/درگاه دیگری اجرا می‌شود.
- می‌خواهید مقادیر `contextWindow`/`maxTokens` را ثابت کنید.
- سرور شما به یک کلید API واقعی نیاز دارد (یا می‌خواهید سرآیندها را کنترل کنید).

```json5
{
  models: {
    providers: {
      sglang: {
        baseUrl: "http://127.0.0.1:30000/v1",
        apiKey: "${SGLANG_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "your-model-id",
            name: "Local SGLang Model",
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

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="رفتار پروکسی‌مانند">
    با SGLang به‌عنوان یک بک‌اند پروکسی‌مانند `/v1` سازگار با OpenAI رفتار می‌شود، نه
    یک نقطه پایانی بومی OpenAI.

    | رفتار | SGLang |
    |----------|--------|
    | شکل‌دهی درخواست مختص OpenAI | اعمال نمی‌شود |
    | `service_tier`، Responses `store`، راهنمایی‌های کش پرامپت | ارسال نمی‌شوند |
    | شکل‌دهی محموله سازگاری استدلال | اعمال نمی‌شود |
    | سرآیندهای انتساب پنهان (`originator`، `version`، `User-Agent`) | در URLهای پایه سفارشی SGLang تزریق نمی‌شوند |

  </Accordion>

  <Accordion title="عیب‌یابی">
    **سرور در دسترس نیست**

    بررسی کنید که سرور در حال اجرا و پاسخ‌گو باشد:

    ```bash
    curl http://127.0.0.1:30000/v1/models
    ```

    **خطاهای احراز هویت**

    اگر درخواست‌ها با خطاهای احراز هویت مواجه می‌شوند، یک `SGLANG_API_KEY` واقعی تنظیم کنید که با
    پیکربندی سرور شما مطابقت داشته باشد، یا ارائه‌دهنده را به‌طور صریح در
    `models.providers.sglang` پیکربندی کنید.

    <Tip>
    اگر SGLang را بدون احراز هویت اجرا می‌کنید، هر مقدار غیرخالی برای
    `SGLANG_API_KEY` جهت اعلام مشارکت در کشف مدل کافی است.
    </Tip>

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/configuration-reference" icon="gear">
    طرح‌واره کامل پیکربندی، شامل ورودی‌های ارائه‌دهنده.
  </Card>
</CardGroup>
