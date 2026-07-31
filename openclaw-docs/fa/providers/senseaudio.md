---
read_when:
    - برای پیوست‌های صوتی، تبدیل گفتار به متن SenseAudio را می‌خواهید
    - به متغیر محیطی کلید API مربوط به SenseAudio یا مسیر پیکربندی صوتی نیاز دارید
summary: تبدیل دسته‌ای گفتار به متن با SenseAudio برای یادداشت‌های صوتی ورودی
title: SenseAudio
x-i18n:
    generated_at: "2026-07-27T17:04:44Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0ca4a31a32eed85c1d9dcd13ebc2eaea94be370d2b1013ae8b4677949bea91d
    source_path: providers/senseaudio.md
    workflow: 16
---

SenseAudio پیوست‌های صوتی و یادداشت‌های صوتی ورودی را از طریق پایپ‌لاین مشترک `tools.media.audio` در OpenClaw رونویسی می‌کند. OpenClaw فایل صوتی چندبخشی را به نقطه پایانی رونویسی سازگار با OpenAI ارسال می‌کند و متن بازگشتی را به‌صورت `{{Transcript}}` به‌همراه یک بلوک `[Audio]` درج می‌کند.

| ویژگی      | مقدار                                            |
| ------------- | ------------------------------------------------ |
| شناسه ارائه‌دهنده   | `senseaudio`                                     |
| Plugin        | داخلی، `enabledByDefault: true`                |
| قرارداد      | `mediaUnderstandingProviders` (صوت)            |
| متغیر محیطی احراز هویت  | `SENSEAUDIO_API_KEY`                             |
| مدل پیش‌فرض | `senseaudio-asr-pro-1.5-260319`                  |
| URL پیش‌فرض   | `https://api.senseaudio.cn/v1`                   |
| وب‌سایت       | [senseaudio.cn](https://senseaudio.cn)           |
| مستندات          | [senseaudio.cn/docs](https://senseaudio.cn/docs) |

## شروع به کار

<Steps>
  <Step title="کلید API خود را تنظیم کنید">
    ```bash
    export SENSEAUDIO_API_KEY="..."
    ```
  </Step>
  <Step title="ارائه‌دهنده صوت را فعال کنید">
    ```json5
    {
      tools: {
        media: {
          audio: {
            enabled: true,
            models: [{ provider: "senseaudio", model: "senseaudio-asr-pro-1.5-260319" }],
          },
        },
      },
    }
    ```
  </Step>
  <Step title="یک یادداشت صوتی ارسال کنید">
    یک پیام صوتی را از طریق هر کانال متصل ارسال کنید. OpenClaw فایل
    صوتی را در SenseAudio بارگذاری می‌کند و از متن رونویسی‌شده در پایپ‌لاین پاسخ استفاده می‌کند.
  </Step>
</Steps>

## گزینه‌ها

| گزینه     | مسیر                            | توضیحات                         |
| ---------- | ------------------------------- | ----------------------------------- |
| `model`    | `tools.media.models[].model`    | شناسه مدل ASR در SenseAudio             |
| `language` | `tools.media.models[].language` | راهنمای اختیاری زبان              |
| `prompt`   | `tools.media.models[].prompt`   | پرامپت اختیاری رونویسی       |
| `baseUrl`  | `tools.media.models[].baseUrl`  | بازنویسی نشانی پایه سازگار با OpenAI |
| `headers`  | `tools.media.models[].headers`  | سرآیندهای اضافی درخواست               |

<Note>
SenseAudio در OpenClaw فقط از STT دسته‌ای پشتیبانی می‌کند. رونویسی بلادرنگ تماس صوتی
همچنان از ارائه‌دهندگانی استفاده می‌کند که از STT جریانی پشتیبانی می‌کنند.
</Note>

## مرتبط

- [درک رسانه (صوت)](/fa/nodes/audio)
- [ارائه‌دهندگان مدل](/fa/concepts/model-providers)
