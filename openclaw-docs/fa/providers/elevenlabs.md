---
read_when:
    - شما می‌خواهید از تبدیل متن به گفتار ElevenLabs در OpenClaw استفاده کنید
    - شما برای پیوست‌های صوتی، تبدیل گفتار به متن ElevenLabs Scribe را می‌خواهید
    - رونویسی بلادرنگ ElevenLabs را برای تماس صوتی یا Google Meet می‌خواهید
summary: از گفتار ElevenLabs، تبدیل گفتار به متن Scribe و رونویسی بلادرنگ با OpenClaw استفاده کنید
title: ElevenLabs
x-i18n:
    generated_at: "2026-07-27T17:03:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5c570aab5fd3ca00e8ded8e3daa143cb199334d507461800ec0b6c1ab0b65c59
    source_path: providers/elevenlabs.md
    workflow: 16
---

OpenClaw برای تبدیل متن به گفتار، تبدیل دسته‌ای گفتار به متن با Scribe
v2 و STT جریانی با Scribe v2 Realtime از ElevenLabs استفاده می‌کند. Plugin به‌صورت داخلی ارائه شده و
به‌طور پیش‌فرض فعال است؛ هیچ مرحله `plugins install` لازم نیست.

| قابلیت               | سطح OpenClaw                                                     | پیش‌فرض                  |
| ------------------------ | -------------------------------------------------------------------- | ------------------------ |
| تبدیل متن به گفتار           | `tts` / `talk`                                                       | `eleven_multilingual_v2` |
| تبدیل دسته‌ای گفتار به متن     | `tools.media.audio`                                                  | `scribe_v2`              |
| تبدیل جریانی گفتار به متن | استریم Voice Call یا `realtime.transcriptionProvider` در Google Meet | `scribe_v2_realtime`     |

## احراز هویت

`ELEVENLABS_API_KEY` را در محیط تنظیم کنید. برای
سازگاری با ابزارهای موجود ElevenLabs، `XI_API_KEY` نیز پذیرفته می‌شود.

```bash
export ELEVENLABS_API_KEY="..."
```

## تبدیل متن به گفتار

```json5
{
  tts: {
    providers: {
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        voiceId: "pMsXgVXv3BLzUgSXRplE",
        modelId: "eleven_multilingual_v2",
      },
    },
  },
}
```

برای استفاده از TTS نسخه v3 در ElevenLabs، `modelId` را روی `eleven_v3` تنظیم کنید. OpenClaw برای نصب‌های موجود،
`eleven_multilingual_v2` را به‌عنوان پیش‌فرض حفظ می‌کند.

هنگامی که ElevenLabs ارائه‌دهنده انتخاب‌شده `voice.tts`/`tts` باشد، کانال‌های صوتی Discord
از نقطه پایانی TTS جریانی ElevenLabs استفاده می‌کنند: پخش از جریان صوتی
برگردانده‌شده آغاز می‌شود، به‌جای اینکه ابتدا منتظر بماند تا OpenClaw کل
فایل صوتی را بارگیری کند. برای مدل‌هایی که آن را می‌پذیرند، `latencyTier` به پارامتر کوئری `optimize_streaming_latency`
در ElevenLabs نگاشت می‌شود؛ OpenClaw این پارامتر را برای
`eleven_v3` که آن را رد می‌کند، حذف می‌کند.

## تبدیل گفتار به متن

برای پیوست‌های صوتی ورودی و قطعه‌های صوتی کوتاه ضبط‌شده از Scribe v2 استفاده کنید:

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "elevenlabs", model: "scribe_v2" }],
      },
    },
  },
}
```

OpenClaw صوت چندبخشی را با
`model_id: "scribe_v2"` به `/v1/speech-to-text` در ElevenLabs ارسال می‌کند. راهنمایی‌های زبان، در صورت وجود، به `language_code` نگاشت می‌شوند.

## STT جریانی

Plugin داخلی `elevenlabs`، ‏Scribe v2 Realtime را برای رونویسی جریانی Voice Call و
حالت عامل Google Meet ثبت می‌کند.

| تنظیم         | مسیر پیکربندی                                                               | پیش‌فرض                                           |
| --------------- | ------------------------------------------------------------------------- | ------------------------------------------------- |
| کلید API         | `plugins.entries.voice-call.config.streaming.providers.elevenlabs.apiKey` | به `ELEVENLABS_API_KEY` / `XI_API_KEY` برمی‌گردد |
| مدل           | `...elevenlabs.modelId`                                                   | `scribe_v2_realtime`                              |
| قالب صوتی    | `...elevenlabs.audioFormat`                                               | `ulaw_8000`                                       |
| نرخ نمونه‌برداری     | `...elevenlabs.sampleRate`                                                | `8000`                                            |
| راهبرد ثبت | `...elevenlabs.commitStrategy`                                            | `vad`                                             |
| زبان        | `...elevenlabs.languageCode`                                              | (تنظیم‌نشده)                                           |

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          streaming: {
            enabled: true,
            provider: "elevenlabs",
            providers: {
              elevenlabs: {
                apiKey: "${ELEVENLABS_API_KEY}",
                audioFormat: "ulaw_8000",
                commitStrategy: "vad",
                languageCode: "en",
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
Voice Call رسانه Twilio را به‌صورت G.711 u-law با نرخ 8 کیلوهرتز دریافت می‌کند. ارائه‌دهنده بلادرنگ ElevenLabs
به‌طور پیش‌فرض از `ulaw_8000` استفاده می‌کند، بنابراین فریم‌های تلفنی را می‌توان بدون
تبدیل کدک ارسال کرد.
</Note>

برای حالت عامل Google Meet،
`plugins.entries.google-meet.config.realtime.transcriptionProvider` را روی
`"elevenlabs"` تنظیم کنید و همان بلوک ارائه‌دهنده را زیر
`plugins.entries.google-meet.config.realtime.providers.elevenlabs` پیکربندی کنید.

## مرتبط

- [تبدیل متن به گفتار](/fa/tools/tts)
- [Google Meet](/fa/plugins/google-meet)
- [انتخاب مدل](/fa/concepts/model-providers)
