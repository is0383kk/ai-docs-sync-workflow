---
read_when:
    - می‌خواهید OpenClaw را با یک سرور محلی Inferrs اجرا کنید
    - شما Gemma یا مدل دیگری را از طریق Inferrs ارائه می‌کنید
    - به فلگ‌های دقیق سازگاری OpenClaw برای Inferrs نیاز دارید
summary: اجرای OpenClaw از طریق Inferrs (سرور محلی سازگار با OpenAI)
title: Inferrs
x-i18n:
    generated_at: "2026-07-27T15:40:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8b9b6fe337a2ec6536332dd62840052fd802fad0a5f3d885ce137523266ff3c9
    source_path: providers/inferrs.md
    workflow: 16
---

[inferrs](https://github.com/ericcurtin/inferrs) مدل‌های محلی را پشت یک API سازگار با OpenAI به نشانی `/v1` ارائه می‌کند. OpenClaw از طریق آداپتور عمومی `openai-completions` با آن ارتباط برقرار می‌کند.

| ویژگی              | مقدار                                                                |
| ------------------ | -------------------------------------------------------------------- |
| شناسه ارائه‌دهنده  | `inferrs` (سفارشی؛ در `models.providers.inferrs` پیکربندی کنید)       |
| Plugin             | ندارد — یک Plugin ارائه‌دهنده همراه OpenClaw نیست                    |
| متغیر محیطی احراز هویت | لازم نیست؛ اگر سرور inferrs احراز هویت نداشته باشد، هر مقداری کار می‌کند |
| API                | سازگار با OpenAI (`openai-completions`)                             |
| نشانی پایه پیشنهادی | `http://127.0.0.1:8080/v1` (یا هر جایی که سرور inferrs شما گوش می‌دهد) |

<Note>
  `inferrs` یک بک‌اند سفارشی، خودمیزبان و سازگار با OpenAI است، نه یک Plugin اختصاصی ارائه‌دهنده OpenClaw: آن را به‌جای انتخاب یک گزینه احراز هویت هنگام راه‌اندازی اولیه، در `models.providers.inferrs` پیکربندی می‌کنید. برای یک Plugin همراه با کشف خودکار، به [SGLang](/fa/providers/sglang) یا [vLLM](/fa/providers/vllm) مراجعه کنید.
</Note>

## شروع به کار

<Steps>
  <Step title="راه‌اندازی inferrs با یک مدل">
    ```bash
    inferrs serve google/gemma-4-E2B-it \
      --host 127.0.0.1 \
      --port 8080 \
      --device metal
    ```
  </Step>
  <Step title="بررسی دسترسی‌پذیری سرور">
    ```bash
    curl http://127.0.0.1:8080/health
    curl http://127.0.0.1:8080/v1/models
    ```
  </Step>
  <Step title="افزودن ورودی ارائه‌دهنده OpenClaw">
    یک ورودی صریح برای ارائه‌دهنده اضافه کنید و مدل پیش‌فرض خود را به آن ارجاع دهید. نمونه پیکربندی زیر را ببینید.
  </Step>
</Steps>

## نمونه پیکربندی کامل

Gemma 4 روی یک سرور محلی `inferrs`:

```json5
{
  agents: {
    defaults: {
      model: { primary: "inferrs/google/gemma-4-E2B-it" },
      models: {
        "inferrs/google/gemma-4-E2B-it": {
          alias: "Gemma 4 (inferrs)",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      inferrs: {
        baseUrl: "http://127.0.0.1:8080/v1",
        apiKey: "inferrs-local",
        api: "openai-completions",
        models: [
          {
            id: "google/gemma-4-E2B-it",
            name: "Gemma 4 E2B (inferrs)",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 131072,
            maxTokens: 4096,
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

## راه‌اندازی در صورت نیاز

OpenClaw فقط زمانی می‌تواند `inferrs` را خودش راه‌اندازی کند که یک مدل `inferrs/...` انتخاب شده باشد. `localService` را به همان ورودی ارائه‌دهنده اضافه کنید:

```json5
{
  models: {
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
            compat: {
              requiresStringContent: true,
            },
          },
        ],
      },
    },
  },
}
```

`command` باید یک مسیر مطلق باشد. `which inferrs` را روی میزبان Gateway اجرا کنید و از همان مسیر استفاده کنید. مرجع کامل فیلدها: [سرویس‌های مدل محلی](/fa/gateway/local-model-services).

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="چرا requiresStringContent اهمیت دارد">
    برخی مسیرهای Chat Completions در `inferrs` فقط `messages[].content` رشته‌ای را می‌پذیرند، نه آرایه‌های ساختاریافته اجزای محتوا.

    <Warning>
    اگر اجرای OpenClaw با خطای زیر ناموفق بود:

    ```text
    messages[1].content: نوع نامعتبر: دنباله؛ یک رشته انتظار می‌رفت
    ```

    مقدار `compat.requiresStringContent: true` را در ورودی مدل تنظیم کنید. سپس OpenClaw پیش از ارسال درخواست، اجزای محتوای صرفاً متنی را به رشته‌های ساده تبدیل می‌کند.
    </Warning>

  </Accordion>

  <Accordion title="نکته احتیاطی Gemma و شِمای ابزار">
    برخی ترکیب‌های `inferrs` و Gemma درخواست‌های مستقیم و کوچک `/v1/chat/completions` را می‌پذیرند، اما در نوبت‌های کامل زمان‌اجرای عامل OpenClaw ناموفق می‌شوند. ابتدا سطح شِمای ابزار را غیرفعال کنید:

    ```json5
    compat: {
      requiresStringContent: true,
      supportsTools: false
    }
    ```

    این کار فشار پرامپت را روی بک‌اندهای محلی سخت‌گیرتر کاهش می‌دهد. اگر درخواست‌های مستقیم کوچک همچنان کار می‌کنند، اما نوبت‌های عادی عامل OpenClaw درون `inferrs` پیوسته از کار می‌افتند، آن را محدودیت مدل یا سرور بالادستی در نظر بگیرید، نه مشکل انتقال OpenClaw.

  </Accordion>

  <Accordion title="آزمون دودستی">
    پس از پیکربندی، هر دو لایه را آزمایش کنید:

    ```bash
    curl http://127.0.0.1:8080/v1/chat/completions \
      -H 'content-type: application/json' \
      -d '{"model":"google/gemma-4-E2B-it","messages":[{"role":"user","content":"۲ + ۲ چند می‌شود؟"}],"stream":false}'
    ```

    ```bash
    openclaw infer model run \
      --model inferrs/google/gemma-4-E2B-it \
      --prompt "۲ + ۲ چند می‌شود؟ با یک جمله کوتاه پاسخ بده." \
      --json
    ```

    اگر فرمان نخست کار می‌کند اما فرمان دوم ناموفق است، بخش عیب‌یابی زیر را ببینید.

  </Accordion>

  <Accordion title="رفتار به‌سبک پراکسی">
    چون `inferrs` از آداپتور عمومی `openai-completions` استفاده می‌کند (نه `openai-responses`)، شکل‌دهی درخواست مختص OpenAI بومی هرگز اعمال نمی‌شود: هیچ `service_tier`، هیچ `store` مربوط به Responses، هیچ راهنمای کش پرامپت و هیچ شکل‌دهی محموله سازگاری استدلال OpenAI ارسال نمی‌شود.
  </Accordion>
</AccordionGroup>

## عیب‌یابی

<AccordionGroup>
  <Accordion title="curl /v1/models ناموفق است">
    `inferrs` اجرا نشده، در دسترس نیست یا به میزبان/پورتی که پیکربندی کرده‌اید متصل نشده است. بررسی کنید که سرور راه‌اندازی شده و روی آن نشانی در حال گوش‌دادن است.
  </Accordion>

  <Accordion title="messages[].content باید رشته باشد">
    مقدار `compat.requiresStringContent: true` را در ورودی مدل تنظیم کنید (بخش بالا را ببینید).
  </Accordion>

  <Accordion title="فراخوانی‌های مستقیم /v1/chat/completions موفق‌اند، اما openclaw infer model run ناموفق است">
    برای غیرفعال‌کردن سطح شِمای ابزار، `compat.supportsTools: false` را تنظیم کنید (نکته احتیاطی Gemma در بالا را ببینید).
  </Accordion>

  <Accordion title="inferrs همچنان در نوبت‌های بزرگ‌تر عامل از کار می‌افتد">
    اگر خطاهای شِما برطرف شده‌اند، اما `inferrs` همچنان در نوبت‌های بزرگ‌تر عامل از کار می‌افتد، آن را محدودیت بالادستی `inferrs` یا مدل در نظر بگیرید. فشار پرامپت را کاهش دهید یا بک‌اند/مدل را تغییر دهید.
  </Accordion>
</AccordionGroup>

<Tip>
برای راهنمایی عمومی، [عیب‌یابی](/fa/help/troubleshooting) و [پرسش‌های متداول](/fa/help/faq) را ببینید.
</Tip>

## مرتبط

<CardGroup cols={2}>
  <Card title="مدل‌های محلی" href="/fa/gateway/local-models" icon="server">
    اجرای OpenClaw با سرورهای مدل محلی.
  </Card>
  <Card title="سرویس‌های مدل محلی" href="/fa/gateway/local-model-services" icon="play">
    راه‌اندازی سرورهای مدل محلی در صورت نیاز برای ارائه‌دهندگان پیکربندی‌شده.
  </Card>
  <Card title="عیب‌یابی Gateway" href="/fa/gateway/troubleshooting#local-openai-compatible-backend-passes-direct-probes-but-agent-runs-fail" icon="wrench">
    اشکال‌زدایی بک‌اندهای محلی سازگار با OpenAI که آزمون‌های بررسی را با موفقیت می‌گذرانند، اما اجرای عامل در آن‌ها ناموفق است.
  </Card>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    مروری بر همه ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
</CardGroup>
