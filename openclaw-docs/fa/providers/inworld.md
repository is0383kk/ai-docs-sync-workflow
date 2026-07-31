---
read_when:
    - برای پاسخ‌های خروجی، سنتز گفتار Inworld می‌خواهید
    - به خروجی تلفنی PCM یا یادداشت صوتی OGG_OPUS از Inworld نیاز دارید
summary: تبدیل متن به گفتار جریانی Inworld برای پاسخ‌های OpenClaw
title: Inworld
x-i18n:
    generated_at: "2026-07-27T16:05:22Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 09560f5beda3b40d9c67f9408d34446f28ecddb8235fc0725c4265c813302946
    source_path: providers/inworld.md
    workflow: 16
---

Inworld یک ارائه‌دهندهٔ تبدیل متن به گفتار (TTS) به‌صورت جریانی است. در OpenClaw، صدای پاسخ‌های خروجی (به‌طور پیش‌فرض MP3 و برای پیام‌های صوتی OGG_OPUS) و صدای PCM خام را برای کانال‌های تلفنی مانند تماس صوتی تولید می‌کند.

OpenClaw درخواست را به نقطهٔ پایانی TTS جریانی Inworld ارسال می‌کند، قطعه‌های صوتی base64 بازگشتی را در یک بافر واحد به‌هم می‌پیوندد و نتیجه را به پایپ‌لاین استاندارد صدای پاسخ تحویل می‌دهد.

| ویژگی      | مقدار                                                           |
| ------------- | --------------------------------------------------------------- |
| شناسهٔ ارائه‌دهنده   | `inworld`                                                       |
| Plugin        | بستهٔ خارجی رسمی (`@openclaw/inworld-speech`)          |
| قرارداد      | `speechProviders` (فقط TTS)                                    |
| متغیر محیطی احراز هویت  | `INWORLD_API_KEY` (HTTP Basic، اعتبارنامهٔ Base64 داشبورد)     |
| نشانی پایه      | `https://api.inworld.ai`                                        |
| صدای پیش‌فرض | `Sarah`                                                         |
| مدل پیش‌فرض | `inworld-tts-1.5-max`                                           |
| خروجی        | MP3 (پیش‌فرض)، OGG_OPUS (پیام‌های صوتی)، PCM با فرکانس 22050 Hz (تلفنی) |
| وب‌سایت       | [inworld.ai](https://inworld.ai)                                |
| مستندات          | [docs.inworld.ai/tts/tts](https://docs.inworld.ai/tts/tts)      |

## نصب Plugin

```bash
openclaw plugins install @openclaw/inworld-speech
openclaw gateway restart
```

## شروع به کار

<Steps>
  <Step title="کلید API خود را تنظیم کنید">
    اعتبارنامه را از داشبورد Inworld خود (Workspace > API Keys) کپی و به‌عنوان متغیر محیطی تنظیم کنید. مقدار بدون تغییر به‌عنوان اعتبارنامهٔ HTTP Basic ارسال می‌شود؛ بنابراین آن را دوباره با Base64 کدگذاری نکنید یا به توکن حامل تبدیل نکنید.

    ```bash
    INWORLD_API_KEY=<base64-credential-from-dashboard>
    ```

  </Step>
  <Step title="Inworld را در tts انتخاب کنید">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "inworld",
        providers: {
          inworld: {
            voiceId: "Sarah",
            modelId: "inworld-tts-1.5-max",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="یک پیام ارسال کنید">
    پاسخی را از طریق هر کانال متصل ارسال کنید. OpenClaw صدا را با Inworld تولید و آن را به‌صورت MP3 تحویل می‌دهد (یا هنگامی که کانال انتظار پیام صوتی دارد، به‌صورت OGG_OPUS).
  </Step>
</Steps>

## گزینه‌های پیکربندی

| گزینه        | مسیر                                | توضیحات                                                         |
| ------------- | ----------------------------------- | ------------------------------------------------------------------- |
| `apiKey`      | `tts.providers.inworld.apiKey`      | اعتبارنامهٔ Base64 داشبورد. در صورت نبود، از `INWORLD_API_KEY` استفاده می‌کند.       |
| `baseUrl`     | `tts.providers.inworld.baseUrl`     | بازنویسی نشانی پایهٔ API‏ Inworld (پیش‌فرض `https://api.inworld.ai`).   |
| `voiceId`     | `tts.providers.inworld.voiceId`     | شناسهٔ صدا (پیش‌فرض `Sarah`). نام مستعار قدیمی: `speakerVoiceId`. |
| `modelId`     | `tts.providers.inworld.modelId`     | شناسهٔ مدل TTS (پیش‌فرض `inworld-tts-1.5-max`).                       |
| `temperature` | `tts.providers.inworld.temperature` | دمای نمونه‌برداری، از `0` (غیرشامل) تا `2` (اختیاری).            |

## نکته‌ها

<AccordionGroup>
  <Accordion title="احراز هویت">
    Inworld از احراز هویت HTTP Basic با یک رشتهٔ اعتبارنامهٔ کدگذاری‌شده با Base64 استفاده می‌کند. آن را بدون تغییر از داشبورد Inworld کپی کنید. ارائه‌دهنده آن را بدون هیچ‌گونه کدگذاری بیشتر به‌صورت `Authorization: Basic <apiKey>` ارسال می‌کند؛ بنابراین خودتان آن را با Base64 کدگذاری نکنید و توکنی از نوع حامل نیز ارائه ندهید. برای همین هشدار، به [نکته‌های احراز هویت TTS](/fa/tools/tts#inworld-primary) مراجعه کنید.
  </Accordion>
  <Accordion title="مدل‌ها">
    شناسه‌های مدل پشتیبانی‌شده: `inworld-tts-1.5-max` (پیش‌فرض)، `inworld-tts-1.5-mini`، `inworld-tts-1-max`، `inworld-tts-1`.
  </Accordion>
  <Accordion title="خروجی‌های صوتی">
    پاسخ‌ها به‌طور پیش‌فرض از MP3 استفاده می‌کنند. هنگامی که مقصد کانال `voice-note` باشد، OpenClaw از Inworld قالب `OGG_OPUS` را درخواست می‌کند تا صدا به‌شکل حباب صوتی بومی پخش شود. تولید گفتار تلفنی برای تغذیهٔ پل تلفنی از `PCM` خام با فرکانس 22050 Hz استفاده می‌کند.
  </Accordion>
  <Accordion title="نقطه‌های پایانی سفارشی">
    میزبان API را با `tts.providers.inworld.baseUrl` بازنویسی کنید. پیش از ارسال درخواست‌ها، اسلش‌های انتهایی حذف می‌شوند.
  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="تبدیل متن به گفتار" href="/fa/tools/tts" icon="waveform-lines">
    نمای کلی TTS، ارائه‌دهندگان و پیکربندی `tts`.
  </Card>
  <Card title="پیکربندی" href="/fa/gateway/configuration" icon="gear">
    مرجع کامل پیکربندی، شامل تنظیمات `tts`.
  </Card>
  <Card title="ارائه‌دهندگان" href="/fa/providers" icon="grid">
    همهٔ ارائه‌دهندگان پشتیبانی‌شدهٔ OpenClaw.
  </Card>
  <Card title="عیب‌یابی" href="/fa/help/troubleshooting" icon="wrench">
    مشکلات رایج و مراحل اشکال‌زدایی.
  </Card>
</CardGroup>
