---
read_when:
    - می‌خواهید از مدل‌های Mistral در OpenClaw استفاده کنید
    - برای تماس صوتی، رونویسی بلادرنگ Voxtral می‌خواهید
    - به راه‌اندازی اولیه کلید API ‏Mistral و ارجاع‌های مدل نیاز دارید
summary: استفاده از مدل‌های Mistral و رونویسی Voxtral با OpenClaw
title: Mistral
x-i18n:
    generated_at: "2026-07-27T16:06:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 23f0ebb664a37cadefb65b7f531cecd3bdfaa4ff5426cb665e305f8f03f0b0ab
    source_path: providers/mistral.md
    workflow: 16
---

Plugin همراه `mistral` چهار قرارداد را ثبت می‌کند: تکمیل‌های چت، درک رسانه (رونویسی دسته‌ای Voxtral)، تبدیل گفتار به متن بلادرنگ برای تماس صوتی (Voxtral Realtime) و تعبیه‌های حافظه (`mistral-embed`).

| ویژگی            | مقدار                                       |
| ---------------- | ------------------------------------------- |
| شناسه ارائه‌دهنده | `mistral`                                   |
| Plugin           | همراه، به‌طور پیش‌فرض فعال                 |
| متغیر محیطی احراز هویت | `MISTRAL_API_KEY`                           |
| پرچم راه‌اندازی اولیه | `--auth-choice mistral-api-key`             |
| پرچم مستقیم CLI  | `--mistral-api-key <key>`                   |
| API              | سازگار با OpenAI (`openai-completions`)    |
| نشانی پایه       | `https://api.mistral.ai/v1`                 |
| مدل پیش‌فرض      | `mistral/mistral-large-latest`              |
| مدل تعبیه‌سازی   | `mistral-embed`                             |
| پردازش دسته‌ای Voxtral | `voxtral-mini-latest` (رونویسی صوت) |
| بلادرنگ Voxtral  | `voxtral-mini-transcribe-realtime-2602`     |

## شروع به کار

<Steps>
  <Step title="دریافت کلید API">
    یک کلید API در [کنسول Mistral](https://console.mistral.ai/) ایجاد کنید.
  </Step>
  <Step title="اجرای راه‌اندازی اولیه">
    ```bash
    openclaw onboard --auth-choice mistral-api-key
    ```

    یا کلید را مستقیماً ارسال کنید:

    ```bash
    openclaw onboard --mistral-api-key "$MISTRAL_API_KEY"
    ```

  </Step>
  <Step title="تنظیم مدل پیش‌فرض">
    ```json5
    {
      env: { MISTRAL_API_KEY: "sk-..." },
      agents: { defaults: { model: { primary: "mistral/mistral-large-latest" } } },
    }
    ```
  </Step>
  <Step title="بررسی در دسترس بودن مدل">
    ```bash
    openclaw models list --provider mistral
    ```
  </Step>
</Steps>

## کاتالوگ داخلی LLM

| مرجع مدل                        | ورودی       | زمینه | حداکثر خروجی | توضیحات                                                 |
| -------------------------------- | ----------- | ------- | ---------- | ----------------------------------------------------- |
| `mistral/mistral-large-latest`   | متن، تصویر | 262,144 | 16,384     | مدل پیش‌فرض                                         |
| `mistral/mistral-medium-2508`    | متن، تصویر | 262,144 | 8,192      | Mistral Medium 3.1                                    |
| `mistral/mistral-medium-3-5`     | متن، تصویر | 262,144 | 8,192      | Mistral Medium 3.5؛ استدلال قابل تنظیم              |
| `mistral/mistral-small-latest`   | متن، تصویر | 262,144 | 16,384     | جدیدترین Mistral Small 4؛ `reasoning_effort` قابل تنظیم |
| `mistral/mistral-small-2603`     | متن، تصویر | 262,144 | 16,384     | نسخه ثابت Mistral Small 4؛ `reasoning_effort` قابل تنظیم |
| `mistral/pixtral-large-latest`   | متن، تصویر | 128,000 | 32,768     | Pixtral                                               |
| `mistral/codestral-latest`       | متن        | 256,000 | 4,096      | کدنویسی                                                |
| `mistral/devstral-medium-latest` | متن        | 262,144 | 32,768     | Devstral 2                                            |
| `mistral/magistral-small`        | متن        | 128,000 | 40,000     | با قابلیت استدلال                                     |

پیش از تغییر پیکربندی، ردیف کاتالوگ همراه را مرور کنید:

```bash
openclaw models list --all --provider mistral --plain
```

یک مدل را بدون راه‌اندازی Gateway به‌صورت آزمایشی اجرا کنید:

```bash
openclaw infer model run --local \
  --model mistral/mistral-medium-3-5 \
  --prompt "دقیقاً با این عبارت پاسخ دهید: mistral-ok" \
  --json
```

## رونویسی صوت (Voxtral)

برای رونویسی دسته‌ای صوت از طریق پایپ‌لاین درک رسانه، از Voxtral استفاده کنید:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "mistral", model: "voxtral-mini-latest" }],
      },
    },
  },
}
```

<Tip>
مسیر رونویسی رسانه از `/v1/audio/transcriptions` استفاده می‌کند. مدل صوتی پیش‌فرض Mistral برابر با `voxtral-mini-latest` است.
</Tip>

## تبدیل گفتار به متن جریانی تماس صوتی

Plugin همراه `mistral`، Voxtral Realtime را به‌عنوان ارائه‌دهنده تبدیل گفتار به متن جریانی تماس صوتی ثبت می‌کند.

| تنظیم         | مسیر پیکربندی                                                            | پیش‌فرض                                 |
| ------------ | ---------------------------------------------------------------------- | --------------------------------------- |
| کلید API      | `plugins.entries.voice-call.config.streaming.providers.mistral.apiKey` | در صورت نبود، از `MISTRAL_API_KEY` استفاده می‌کند         |
| مدل           | `...mistral.model`                                                     | `voxtral-mini-transcribe-realtime-2602` |
| کدگذاری       | `...mistral.encoding`                                                  | `pcm_mulaw`                             |
| نرخ نمونه‌برداری | `...mistral.sampleRate`                                                | `8000`                                  |
| تأخیر هدف     | `...mistral.targetStreamingDelayMs`                                    | `800`                                   |

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          streaming: {
            enabled: true,
            provider: "mistral",
            providers: {
              mistral: {
                apiKey: "${MISTRAL_API_KEY}",
                targetStreamingDelayMs: 800,
              },
            },
          },
        },
      },
    },
  },
}
```

<Note>
OpenClaw به‌طور پیش‌فرض تبدیل گفتار به متن بلادرنگ Mistral را روی `pcm_mulaw` با نرخ 8 kHz تنظیم می‌کند تا تماس صوتی بتواند فریم‌های رسانه‌ای Twilio را مستقیماً ارسال کند. تنها در صورتی از `encoding: "pcm_s16le"` و `sampleRate` منطبق استفاده کنید که جریان بالادستی از قبل PCM خام باشد.
</Note>

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="استدلال قابل تنظیم">
    `mistral/mistral-small-latest`، `mistral/mistral-small-2603` و `mistral/mistral-medium-3-5` از طریق `reasoning_effort` از [استدلال قابل تنظیم](https://docs.mistral.ai/studio-api/conversations/reasoning/adjustable) در API تکمیل‌های چت پشتیبانی می‌کنند (`none` تفکر اضافی در خروجی را به حداقل می‌رساند؛ `high` رد کامل تفکر را پیش از پاسخ نهایی نمایش می‌دهد).

    OpenClaw سطح **تفکر** نشست را به API متعلق به Mistral نگاشت می‌کند:

    | سطح تفکر OpenClaw                                              | `reasoning_effort` در Mistral |
    | ----------------------------------------------------------------------- | --------------------------- |
    | **خاموش** / **حداقلی**                                                 | `none`                      |
    | **کم** / **متوسط** / **زیاد** / **بسیار زیاد** / **تطبیقی** / **حداکثر** | `high`                       |

    <Warning>
    حالت استدلال Medium 3.5 را با `temperature: 0` ترکیب نکنید؛ گزارش شده است که API ‏HTTP متعلق به Mistral ترکیب `reasoning_effort="high"` و `temperature: 0` را با پاسخ 400 رد می‌کند. دما را تنظیم‌نشده باقی بگذارید، یا تفکر را خاموش/حداقلی کنید تا OpenClaw پیش از تنظیم دمای پایین، `reasoning_effort: "none"` را ارسال کند.
    </Warning>

    نمونه پیکربندی محدود به مدل برای استدلال Medium 3.5:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "mistral/mistral-medium-3-5" },
          models: {
            "mistral/mistral-medium-3-5": {
              params: { thinking: "high" },
            },
          },
        },
      },
    }
    ```

    <Note>
    سایر مدل‌های کاتالوگ همراه Mistral از این پارامتر استفاده نمی‌کنند. هنگامی که رفتار بومی و استدلال‌محور Mistral را می‌خواهید، همچنان از مدل‌های `magistral-*` استفاده کنید.
    </Note>

  </Accordion>

  <Accordion title="تعبیه‌های حافظه">
    Mistral می‌تواند تعبیه‌های حافظه را از طریق `/v1/embeddings` ارائه دهد (مدل پیش‌فرض: `mistral-embed`):

    ```json5
    {
      memory: {
        search: { provider: "mistral" },
      },
    }
    ```

  </Accordion>

  <Accordion title="احراز هویت و نشانی پایه">
    - احراز هویت Mistral از `MISTRAL_API_KEY` (سرآیند Bearer) استفاده می‌کند.
    - نشانی پایه ارائه‌دهنده به‌طور پیش‌فرض `https://api.mistral.ai/v1` است و قالب استاندارد درخواست تکمیل چت سازگار با OpenAI را می‌پذیرد.
    - مدل پیش‌فرض راه‌اندازی اولیه `mistral/mistral-large-latest` است.
    - تنها زمانی نشانی پایه را در `models.providers.mistral.baseUrl` بازنویسی کنید که Mistral صراحتاً نقطه پایانی منطقه‌ای مورد نیازتان را منتشر کرده باشد.

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، مراجع مدل و رفتار تغییر مسیر هنگام خرابی.
  </Card>
  <Card title="درک رسانه" href="/fa/nodes/media-understanding" icon="microphone">
    راه‌اندازی رونویسی صوت و انتخاب ارائه‌دهنده.
  </Card>
</CardGroup>
