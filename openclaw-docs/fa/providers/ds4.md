---
read_when:
    - می‌خواهید OpenClaw را با antirez/ds4 اجرا کنید
    - یک بک‌اند محلی DeepSeek V4 Flash با فراخوانی ابزار می‌خواهید
    - به پیکربندی OpenClaw برای ds4-server نیاز دارید
summary: OpenClaw را از طریق ds4، یک سرور محلی سازگار با OpenAI برای DeepSeek V4 Flash، اجرا کنید
title: ds4
x-i18n:
    generated_at: "2026-07-27T17:00:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: be449813295648694625ef8003b3f4b12903535b74816916ca5af0695174fbf4
    source_path: providers/ds4.md
    workflow: 16
---

[ds4](https://github.com/antirez/ds4) مدل DeepSeek V4 Flash را از طریق یک بک‌اند محلی
Metal با API سازگار با OpenAI به نشانی `/v1` ارائه می‌کند. OpenClaw از طریق
خانواده ارائه‌دهنده عمومی `openai-completions` به ds4 متصل می‌شود.

ds4 یک Plugin ارائه‌دهنده همراه OpenClaw نیست. آن را زیر
`models.providers.ds4` پیکربندی کنید، سپس `ds4/deepseek-v4-flash` را انتخاب کنید.

| ویژگی       | مقدار                                                     |
| ----------- | --------------------------------------------------------- |
| شناسه ارائه‌دهنده | `ds4`                                                     |
| Plugin      | ندارد (فقط پیکربندی)                                        |
| API         | Chat Completions سازگار با OpenAI ‏(`openai-completions`) |
| نشانی پایه    | `http://127.0.0.1:18000/v1` (پیشنهادی)                   |
| شناسه مدل    | `deepseek-v4-flash`                                       |
| فراخوانی ابزارها  | به سبک OpenAI، ‏`tools` / `tool_calls`                       |
| استدلال   | به سبک DeepSeek، ‏`thinking` و `reasoning_effort`          |

## الزامات

- macOS با پشتیبانی از Metal.
- یک checkout عملیاتی از ds4 به‌همراه `ds4-server` و فایل GGUF مدل DeepSeek V4 Flash.
- حافظه کافی برای بافت انتخابی؛ مقادیر بزرگ‌تر `--ctx` هنگام راه‌اندازی سرور
  حافظه KV بیشتری تخصیص می‌دهند.

<Warning>
نوبت‌های عامل OpenClaw شامل طرح‌واره‌های ابزار و بافت فضای کاری هستند. بافت کوچکی
مانند `--ctx 4096` ممکن است آزمون‌های مستقیم curl را با موفقیت بگذراند، اما اجرای کامل عامل با
`500 prompt exceeds context` ناموفق شود. برای آزمون‌های دود عامل و ابزار، دست‌کم از
`--ctx 32768` استفاده کنید. تنها در صورت وجود حافظه کافی و برای فعال‌کردن
Think Max در ds4 از `--ctx 393216` استفاده کنید.
</Warning>

## شروع سریع

<Steps>
  <Step title="راه‌اندازی ds4-server">
    `<DS4_DIR>` را با مسیر checkout مدل ds4 خود جایگزین کنید.

    ```bash
    <DS4_DIR>/ds4-server \
      --model <DS4_DIR>/ds4flash.gguf \
      --host 127.0.0.1 \
      --port 18000 \
      --ctx 32768 \
      --tokens 128
    ```

  </Step>
  <Step title="بررسی نقطه پایانی سازگار با OpenAI">
    ```bash
    curl http://127.0.0.1:18000/v1/models
    ```

    پاسخ باید شامل `deepseek-v4-flash` باشد.

  </Step>
  <Step title="افزودن پیکربندی ارائه‌دهنده OpenClaw">
    پیکربندی بخش [پیکربندی کامل](#full-config) را اضافه کنید، سپس یک بررسی یک‌باره مدل
    اجرا کنید:

    ```bash
    openclaw infer model run \
      --local \
      --model ds4/deepseek-v4-flash \
      --thinking off \
      --prompt "Reply with exactly: openclaw-ds4-ok" \
      --json
    ```

  </Step>
</Steps>

## پیکربندی کامل

وقتی ds4 از قبل روی `127.0.0.1:18000` در حال اجرا است، از این پیکربندی استفاده کنید.

```json5
{
  agents: {
    defaults: {
      model: { primary: "ds4/deepseek-v4-flash" },
      models: {
        "ds4/deepseek-v4-flash": {
          alias: "DS4 local",
        },
      },
    },
  },
  models: {
    mode: "merge",
    providers: {
      ds4: {
        baseUrl: "http://127.0.0.1:18000/v1",
        apiKey: "ds4-local",
        api: "openai-completions",
        timeoutSeconds: 300,
        models: [
          {
            id: "deepseek-v4-flash",
            name: "DeepSeek V4 Flash (ds4)",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32768,
            maxTokens: 128,
            compat: {
              supportsUsageInStreaming: true,
              supportsReasoningEffort: true,
              maxTokensField: "max_tokens",
              supportsStrictMode: false,
              thinkingFormat: "deepseek",
              supportedReasoningEfforts: ["low", "medium", "high", "xhigh"],
            },
          },
        ],
      },
    },
  },
}
```

`contextWindow` را با `ds4-server --ctx` هم‌تراز نگه دارید. `maxTokens` را با
`--tokens` هم‌تراز نگه دارید، مگر اینکه عمداً بخواهید OpenClaw خروجی کمتری
از مقدار پیش‌فرض سرور درخواست کند.

## راه‌اندازی برحسب تقاضا

OpenClaw می‌تواند ds4 را تنها زمانی راه‌اندازی کند که یک مدل `ds4/...` انتخاب شده باشد.
`localService` را به همان ورودی ارائه‌دهنده اضافه کنید:

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
        models: [
          {
            id: "deepseek-v4-flash",
            name: "DeepSeek V4 Flash (ds4)",
            reasoning: true,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 32768,
            maxTokens: 128,
            compat: {
              supportsUsageInStreaming: true,
              supportsReasoningEffort: true,
              maxTokensField: "max_tokens",
              supportsStrictMode: false,
              thinkingFormat: "deepseek",
              supportedReasoningEfforts: ["low", "medium", "high", "xhigh"],
            },
          },
        ],
      },
    },
  },
}
```

`command` باید یک مسیر مطلق فایل اجرایی باشد. جست‌وجوی پوسته و گسترش `~`
استفاده نمی‌شوند. برای مشاهده تمام فیلدهای `localService`، به
[سرویس‌های مدل محلی](/fa/gateway/local-model-services) مراجعه کنید.

## Think Max

ds4 تنها زمانی Think Max را اعمال می‌کند که هر دو شرط برقرار باشند:

- `ds4-server` با `--ctx 393216` یا مقداری بالاتر آغاز شود.
- درخواست از `reasoning_effort: "max"` (یا فیلد تلاش معادل ds4) استفاده کند.

اگر آن بافت بزرگ را اجرا می‌کنید، هم پرچم‌های سرور و هم فراداده مدل OpenClaw را
به‌روزرسانی کنید:

```json5
{
  contextWindow: 393216,
  maxTokens: 384000,
  compat: {
    supportsUsageInStreaming: true,
    supportsReasoningEffort: true,
    maxTokensField: "max_tokens",
    supportsStrictMode: false,
    thinkingFormat: "deepseek",
    supportedReasoningEfforts: ["low", "medium", "high", "xhigh", "max"],
  },
}
```

## آزمون

بررسی مستقیم HTTP با دورزدن OpenClaw:

```bash
curl http://127.0.0.1:18000/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"Reply with exactly: ds4-ok"}],"max_tokens":16,"stream":false,"thinking":{"type":"disabled"}}'
```

مسیریابی مدل OpenClaw (همانند بررسی شروع سریع):

```bash
openclaw infer model run \
  --local \
  --model ds4/deepseek-v4-flash \
  --thinking off \
  --prompt "Reply with exactly: openclaw-ds4-ok" \
  --json
```

آزمون دود کامل عامل و فراخوانی ابزار، با بافتی دست‌کم برابر با 32768:

```bash
openclaw agent \
  --local \
  --session-id ds4-tool-smoke \
  --model ds4/deepseek-v4-flash \
  --thinking off \
  --message "Use the shell command pwd once, then reply exactly: tool-ok <output>" \
  --json \
  --timeout 240
```

نتیجه مورد انتظار:

- `executionTrace.winnerProvider` برابر با `ds4` است
- `executionTrace.winnerModel` برابر با `deepseek-v4-flash` است
- `toolSummary.calls` دست‌کم `1` است
- `finalAssistantVisibleText` با `tool-ok` آغاز می‌شود

## عیب‌یابی

<AccordionGroup>
  <Accordion title="curl نمی‌تواند به ‎/v1/models متصل شود">
    ds4 در حال اجرا نیست یا به میزبان/درگاه موجود در `baseUrl` متصل نشده است.
    `ds4-server` را راه‌اندازی کنید، سپس دوباره تلاش کنید:

    ```bash
    curl http://127.0.0.1:18000/v1/models
    ```

  </Accordion>

  <Accordion title="خطای 500: پرامپت از بافت فراتر می‌رود">
    `--ctx` پیکربندی‌شده برای نوبت OpenClaw بیش از حد کوچک است.
    `ds4-server --ctx` را افزایش دهید، سپس `models.providers.ds4.models[].contextWindow` را
    مطابق آن به‌روزرسانی کنید. نوبت‌های کامل عامل همراه با ابزارها نسبت به یک
    درخواست مستقیم curl با یک پیام، به بافت بسیار بیشتری نیاز دارند.
  </Accordion>

  <Accordion title="Think Max فعال نمی‌شود">
    ds4 تنها زمانی از Think Max استفاده می‌کند که `--ctx` دست‌کم `393216` باشد و درخواست
    `reasoning_effort: "max"` را بخواهد. بافت‌های کوچک‌تر به استدلال
    بالا بازمی‌گردند.
  </Accordion>

  <Accordion title="نخستین درخواست کند است">
    ds4 یک مرحله استقرار سرد در Metal و گرم‌سازی مدل دارد. وقتی OpenClaw سرور را
    برحسب تقاضا راه‌اندازی می‌کند، `localService.readyTimeoutMs: 300000` را تنظیم کنید.
  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="سرویس‌های مدل محلی" href="/fa/gateway/local-model-services" icon="play">
    سرورهای مدل محلی را پیش از درخواست‌های مدل، برحسب تقاضا راه‌اندازی کنید.
  </Card>
  <Card title="مدل‌های محلی" href="/fa/gateway/local-models" icon="server">
    بک‌اندهای مدل محلی را انتخاب و اداره کنید.
  </Card>
  <Card title="ارائه‌دهندگان مدل" href="/fa/concepts/model-providers" icon="layers">
    ارجاعات ارائه‌دهنده، احراز هویت و انتقال هنگام خرابی را پیکربندی کنید.
  </Card>
  <Card title="DeepSeek" href="/fa/providers/deepseek" icon="brain">
    رفتار بومی ارائه‌دهنده DeepSeek و کنترل‌های تفکر.
  </Card>
</CardGroup>
