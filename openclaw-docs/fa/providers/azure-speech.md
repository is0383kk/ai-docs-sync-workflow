---
read_when:
    - برای پاسخ‌های خروجی، به سنتز گفتار Azure نیاز دارید
    - به خروجی بومی یادداشت صوتی Ogg Opus از Azure Speech نیاز دارید
summary: تبدیل متن به گفتار Azure AI Speech برای پاسخ‌های OpenClaw
title: گفتار Azure
x-i18n:
    generated_at: "2026-07-27T16:57:19Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cfeeb9daa8d7d6aa24e497d57d64e07efa94c3c0c6b16f793343a450286ab3c1
    source_path: providers/azure-speech.md
    workflow: 16
---

Azure Speech یک ارائه‌دهندهٔ تبدیل متن به گفتار Azure AI Speech است که به‌صورت داخلی ارائه می‌شود. OpenClaw
با استفاده از SSML مستقیماً API ‏REST سرویس Azure Speech را فراخوانی می‌کند و برای
پاسخ‌های استاندارد MP3، برای پیام‌های صوتی Ogg/Opus بومی و برای
کانال‌های تلفنی مانند تماس صوتی، mulaw با نرخ 8 kHz تولید می‌کند. درخواست، قالب خروجی متعلق به
ارائه‌دهنده را از طریق هدر `X-Microsoft-OutputFormat` ارسال می‌کند.

| جزئیات                  | مقدار                                                                                                          |
| ----------------------- | -------------------------------------------------------------------------------------------------------------- |
| شناسهٔ ارائه‌دهنده             | `azure-speech` (نام مستعار: `azure`)                                                                                |
| وب‌سایت                 | [Azure AI Speech](https://azure.microsoft.com/products/ai-services/ai-speech)                                  |
| مستندات                    | [تبدیل متن به گفتار با REST سرویس Speech](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech) |
| احراز هویت                    | `AZURE_SPEECH_KEY` به‌همراه `AZURE_SPEECH_REGION`                                                                  |
| صدای پیش‌فرض           | `en-US-JennyNeural`                                                                                            |
| خروجی پیش‌فرض فایل     | `audio-24khz-48kbitrate-mono-mp3`                                                                              |
| فایل پیش‌فرض پیام صوتی | `ogg-24khz-16bit-mono-opus`                                                                                    |

## شروع به کار

<Steps>
  <Step title="ایجاد یک منبع Azure Speech">
    در پورتال Azure، یک منبع Speech ایجاد کنید. **KEY 1** را از
    Resource Management > Keys and Endpoint کپی کنید و موقعیت منبع
    مانند `eastus` را نیز کپی کنید.

    ```
    AZURE_SPEECH_KEY=<speech-resource-key>
    AZURE_SPEECH_REGION=eastus
    ```

  </Step>
  <Step title="انتخاب Azure Speech در tts">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "azure-speech",
        providers: {
          "azure-speech": {
            voice: "en-US-JennyNeural",
            lang: "en-US",
          },
        },
      },
    }
    ```
  </Step>
  <Step title="ارسال پیام">
    پاسخی را از طریق هر کانال متصل ارسال کنید. OpenClaw صدا را
    با Azure Speech تولید می‌کند و برای صدای استاندارد MP3، یا هنگامی که
    کانال انتظار پیام صوتی دارد Ogg/Opus تحویل می‌دهد.
  </Step>
</Steps>

## گزینه‌های پیکربندی

همهٔ گزینه‌ها زیر `tts.providers["azure-speech"]` قرار دارند.

| گزینه                  | توضیحات                                                                                           |
| ----------------------- | ----------------------------------------------------------------------------------------------------- |
| `apiKey`                | کلید منبع Azure Speech. به `AZURE_SPEECH_KEY`، `AZURE_SPEECH_API_KEY` یا `SPEECH_KEY` بازمی‌گردد. |
| `region`                | منطقهٔ منبع Azure Speech. به `AZURE_SPEECH_REGION` یا `SPEECH_REGION` بازمی‌گردد.                 |
| `endpoint`              | جایگزین اختیاری نقطهٔ پایانی Azure Speech. به `AZURE_SPEECH_ENDPOINT` مورداعتماد بازمی‌گردد.               |
| `baseUrl`               | جایگزین اختیاری URL پایهٔ Azure Speech.                                                              |
| `voice`                 | ShortName صدای Azure (پیش‌فرض `en-US-JennyNeural`). نام مستعار قدیمی: `voiceId`.                         |
| `lang`                  | کد زبان SSML (پیش‌فرض `en-US`).                                                                 |
| `outputFormat`          | قالب خروجی فایل صوتی (پیش‌فرض `audio-24khz-48kbitrate-mono-mp3`).                                 |
| `voiceNoteOutputFormat` | قالب خروجی پیام صوتی (پیش‌فرض `ogg-24khz-16bit-mono-opus`).                                       |
| `timeoutMs`             | جایگزین مهلت زمانی درخواست بر حسب میلی‌ثانیه. به `tts.timeoutMs` سراسری بازمی‌گردد.                   |

پس از تنظیم `apiKey` به‌همراه یکی از
`region`، `endpoint` یا `baseUrl`، ارائه‌دهنده پیکربندی‌شده در نظر گرفته می‌شود. متغیرهای محیطی فقط به‌عنوان گزینهٔ بازگشتی
برای کلیدهای پیکربندی تنظیم‌نشده بررسی می‌شوند. فایل‌های `.env` فضای کاری نمی‌توانند
`AZURE_SPEECH_ENDPOINT` را تنظیم کنند؛ برای مسیریابی نقطهٔ پایانی از محیط فرایند، dotenv زمان اجرای سراسری
یا پیکربندی صریح استفاده کنید.

## نکات

<AccordionGroup>
  <Accordion title="احراز هویت">
    Azure Speech از کلید منبع Speech استفاده می‌کند، نه کلید Azure OpenAI. کلید
    به‌صورت `Ocp-Apim-Subscription-Key` ارسال می‌شود؛ مگر اینکه
    `endpoint` یا `baseUrl` را ارائه دهید، OpenClaw مقدار
    `https://<region>.tts.speech.microsoft.com` را از `region` به دست می‌آورد.
  </Accordion>
  <Accordion title="نام‌های صدا">
    از مقدار `ShortName` صدای Azure Speech استفاده کنید، برای نمونه
    `en-US-JennyNeural`. ارائه‌دهندهٔ داخلی می‌تواند صداها را از طریق
    همان منبع Speech فهرست کند و صداهایی را که منسوخ، بازنشسته
    یا غیرفعال علامت‌گذاری شده‌اند، حذف می‌کند.
  </Accordion>
  <Accordion title="خروجی‌های صوتی">
    Azure قالب‌های خروجی مانند `audio-24khz-48kbitrate-mono-mp3`،
    `ogg-24khz-16bit-mono-opus` و `riff-24khz-16bit-mono-pcm` را می‌پذیرد. OpenClaw
    برای مقصدهای `voice-note`، Ogg/Opus درخواست می‌کند تا کانال‌ها بتوانند حباب‌های صوتی بومی را
    بدون تبدیل اضافی MP3 ارسال کنند و برای مقصدهای تلفنی
    `raw-8khz-8bit-mono-mulaw` را اجباری می‌کند.
  </Accordion>
  <Accordion title="نام مستعار">
    `azure` به‌عنوان نام مستعار ارائه‌دهنده برای پیکربندی موجود پذیرفته می‌شود، اما پیکربندی
    جدید باید برای جلوگیری از اشتباه با ارائه‌دهندگان مدل Azure OpenAI از
    `azure-speech` استفاده کند.
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
    همهٔ ارائه‌دهندگان داخلی OpenClaw.
  </Card>
  <Card title="عیب‌یابی" href="/fa/help/troubleshooting" icon="wrench">
    مشکلات رایج و مراحل اشکال‌زدایی.
  </Card>
</CardGroup>
