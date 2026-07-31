---
read_when:
    - فعال‌سازی تبدیل متن به گفتار برای پاسخ‌ها
    - پیکربندی ارائه‌دهندهٔ TTS، زنجیرهٔ جایگزین یا پرسونا
    - استفاده از دستورها یا دایرکتیوهای /tts
sidebarTitle: Text to speech (TTS)
summary: تبدیل متن به گفتار برای پاسخ‌های خروجی — ارائه‌دهندگان، شخصیت‌ها، فرمان‌های اسلش و خروجی مختص هر کانال
title: تبدیل متن به گفتار
x-i18n:
    generated_at: "2026-07-27T17:13:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2ae9d0cc6f77c6a8b1b379c3712fd92fbbc22dae694ecdd46a0bb35cec0d29e7
    source_path: tools/tts.md
    workflow: 16
---

OpenClaw پاسخ‌های خروجی را با استفاده از **14 ارائه‌دهنده گفتار** به صوت تبدیل می‌کند:
پیام‌های صوتی بومی در Feishu، Matrix، Telegram و WhatsApp؛ پیوست‌های
صوتی در همه‌جای دیگر؛ و جریان‌های PCM/Ulaw برای تلفن و Talk.

TTS بخش خروجی گفتار در حالت `stt-tts`ِ Talk است (`talk.speak` نیز از همین
مسیر سنتز استفاده می‌کند). نشست‌های Talk بومیِ ارائه‌دهنده با `realtime` گفتار را
درون ارائه‌دهنده بلادرنگ سنتز می‌کنند؛ نشست‌های `transcription` هرگز
پاسخ صوتی دستیار را سنتز نمی‌کنند.

## شروع سریع

<Steps>
  <Step title="انتخاب ارائه‌دهنده">
    OpenAI و ElevenLabs مطمئن‌ترین گزینه‌های میزبانی‌شده هستند. Microsoft و
    CLI محلی بدون کلید API کار می‌کنند. برای فهرست کامل، [ماتریس ارائه‌دهندگان](#supported-providers)
    را ببینید.
  </Step>
  <Step title="تنظیم کلید API">
    متغیر محیطی ارائه‌دهنده خود را صادر کنید (برای نمونه `OPENAI_API_KEY`،
    `ELEVENLABS_API_KEY`). Microsoft و CLI محلی به کلید نیاز ندارند.
  </Step>
  <Step title="فعال‌سازی در پیکربندی">
    `tts.auto: "always"` و `tts.provider` را تنظیم کنید:

    ```json5
    {
      tts: {
        auto: "always",
        provider: "elevenlabs",
      },
    }
    ```

  </Step>
  <Step title="آزمایش در گفت‌وگو">
    `/tts status` وضعیت فعلی را نشان می‌دهد. `/tts audio Hello from OpenClaw`
    یک پاسخ صوتی یک‌باره ارسال می‌کند.
  </Step>
</Steps>

<Note>
TTS خودکار به‌طور پیش‌فرض **خاموش** است. وقتی `tts.provider` تنظیم نشده باشد،
OpenClaw نخستین ارائه‌دهنده پیکربندی‌شده را بر اساس ترتیب انتخاب خودکار رجیستری برمی‌گزیند.
ابزار داخلی عامل `tts` فقط برای قصد صریح است: گفت‌وگوی عادی
متنی باقی می‌ماند، مگر اینکه کاربر صوت درخواست کند، از `/tts` استفاده کند یا گفتار
TTS خودکار/دستورالعملی را فعال کند.
</Note>

## ارائه‌دهندگان پشتیبانی‌شده

| ارائه‌دهنده          | احراز هویت                                                                                                             | توضیحات                                                                                       |
| ----------------- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| **Azure Speech**  | `AZURE_SPEECH_KEY` + `AZURE_SPEECH_REGION` (همچنین `AZURE_SPEECH_API_KEY`، `SPEECH_KEY`، `SPEECH_REGION`)          | خروجی بومی یادداشت صوتی Ogg/Opus و تلفن.                                            |
| **DeepInfra**     | `DEEPINFRA_API_KEY`                                                                                              | TTS سازگار با OpenAI. مقدار پیش‌فرض `hexgrad/Kokoro-82M` است.                                    |
| **ElevenLabs**    | `ELEVENLABS_API_KEY` یا `XI_API_KEY`                                                                             | شبیه‌سازی صدا، چندزبانه، قطعی با `seed`؛ برای پخش صوتی Discord به‌صورت جریانی ارائه می‌شود. |
| **Google Gemini** | `GEMINI_API_KEY` یا `GOOGLE_API_KEY`                                                                             | TTS دسته‌ای Gemini API؛ با `promptTemplate: "audio-profile-v1"` از پرسونا آگاه می‌شود.               |
| **Gradium**       | `GRADIUM_API_KEY`                                                                                                | خروجی یادداشت صوتی و تلفن.                                                            |
| **Inworld**       | `INWORLD_API_KEY`                                                                                                | API جریانی TTS. یادداشت صوتی بومی Opus و تلفن PCM.                                |
| **CLI محلی**     | هیچ‌کدام                                                                                                             | یک فرمان محلی پیکربندی‌شده TTS را اجرا می‌کند.                                                        |
| **Microsoft**     | هیچ‌کدام                                                                                                             | TTS عصبی عمومی Edge از طریق `node-edge-tts`. به‌صورت بهترین تلاش، بدون SLA.                            |
| **MiniMax**       | `MINIMAX_API_KEY` (یا طرح توکن: `MINIMAX_OAUTH_TOKEN`، `MINIMAX_CODE_PLAN_KEY`، `MINIMAX_CODING_API_KEY`)      | API نسخه 2 T2A. مقدار پیش‌فرض `speech-2.8-hd` است.                                                    |
| **OpenAI**        | `OPENAI_API_KEY`                                                                                                 | برای خلاصه‌سازی خودکار نیز استفاده می‌شود؛ از پرسونای `instructions` پشتیبانی می‌کند.                                |
| **OpenRouter**    | `OPENROUTER_API_KEY` (می‌تواند از `models.providers.openrouter.apiKey` دوباره استفاده کند)                                            | مدل پیش‌فرض `hexgrad/kokoro-82m` است.                                                         |
| **Volcengine**    | `VOLCENGINE_TTS_API_KEY` یا `BYTEPLUS_SEED_SPEECH_API_KEY` (AppID/توکن قدیمی: `VOLCENGINE_TTS_APPID`/`_TOKEN`) | API HTTP گفتار BytePlus Seed.                                                              |
| **Vydra**         | `VYDRA_API_KEY`                                                                                                  | ارائه‌دهنده مشترک تصویر، ویدئو و گفتار.                                                   |
| **xAI**           | `XAI_API_KEY`                                                                                                    | TTS دسته‌ای xAI. یادداشت صوتی بومی Opus پشتیبانی **نمی‌شود**.                                 |
| **Xiaomi MiMo**   | `XIAOMI_API_KEY`                                                                                                 | TTS مدل MiMo از طریق تکمیل‌های گفت‌وگوی Xiaomi.                                                   |

اگر چند ارائه‌دهنده پیکربندی شده باشند، ابتدا از ارائه‌دهنده انتخاب‌شده استفاده می‌شود و
سایرین گزینه‌های جایگزین هستند. خلاصه‌سازی خودکار از `summaryModel` (یا
`agents.defaults.model.primary`) استفاده می‌کند؛ بنابراین اگر خلاصه‌ها را فعال نگه می‌دارید،
آن ارائه‌دهنده نیز باید احراز هویت شده باشد.

<Warning>
ارائه‌دهنده همراه **Microsoft** از سرویس آنلاین TTS عصبی Microsoft Edge
از طریق `node-edge-tts` استفاده می‌کند. این یک سرویس وب عمومی بدون
SLA یا سهمیه منتشرشده است؛ آن را سرویسی بر مبنای بهترین تلاش در نظر بگیرید. شناسه قدیمی ارائه‌دهنده `edge`
به `microsoft` نرمال‌سازی می‌شود و `openclaw doctor --fix` پیکربندی ذخیره‌شده را
بازنویسی می‌کند؛ پیکربندی‌های جدید باید همیشه از `microsoft` استفاده کنند.
</Warning>

## پیکربندی

پیکربندی TTS در `~/.openclaw/openclaw.json` زیر `tts` قرار دارد. یک
پیش‌تنظیم انتخاب کنید و بلوک ارائه‌دهنده را تطبیق دهید. فیلدهای `speakerVoice`/`speakerVoiceId`
که در ادامه آمده‌اند، فیلدهای معیار هستند؛ نام فیلدهای اختصاصی `voice`/`voiceId`/
`voiceName` هر ارائه‌دهنده همچنان به‌عنوان نام‌های مستعار قدیمی کار می‌کنند.

<Tabs>
  <Tab title="Azure Speech">
```json5
{
  tts: {
    auto: "always",
    provider: "azure-speech",
    providers: {
      "azure-speech": {
        apiKey: "${AZURE_SPEECH_KEY}",
        region: "eastus",
        speakerVoice: "en-US-JennyNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
        voiceNoteOutputFormat: "ogg-24khz-16bit-mono-opus",
      },
    },
  },
}
```
  </Tab>
  <Tab title="ElevenLabs">
```json5
{
  tts: {
    auto: "always",
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        model: "eleven_multilingual_v2",
        speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Google Gemini">
```json5
{
  tts: {
    auto: "always",
    provider: "google",
    providers: {
      google: {
        apiKey: "${GEMINI_API_KEY}",
        model: "gemini-3.1-flash-tts-preview",
        speakerVoice: "Kore",
        // درخواست‌های اختیاری سبک به زبان طبیعی:
        // audioProfile: "با لحنی آرام و شبیه میزبان پادکست صحبت کن.",
        // speakerName: "Alex",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Gradium">
```json5
{
  tts: {
    auto: "always",
    provider: "gradium",
    providers: {
      gradium: {
        apiKey: "${GRADIUM_API_KEY}",
        speakerVoiceId: "YTpq7expH9539ERJ",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Inworld">
```json5
{
  tts: {
    auto: "always",
    provider: "inworld",
    providers: {
      inworld: {
        apiKey: "${INWORLD_API_KEY}",
        modelId: "inworld-tts-1.5-max",
        speakerVoiceId: "Sarah",
        temperature: 0.7,
      },
    },
  },
}
```
  </Tab>
  <Tab title="CLI محلی">
```json5
{
  tts: {
    auto: "always",
    provider: "tts-local-cli",
    providers: {
      "tts-local-cli": {
        command: "say",
        args: ["-o", "{{OutputPath}}", "{{Text}}"],
        outputFormat: "wav",
        timeoutMs: 120000,
      },
    },
  },
}
```
  </Tab>
  <Tab title="Microsoft (بدون کلید)">
```json5
{
  tts: {
    auto: "always",
    provider: "microsoft",
    providers: {
      microsoft: {
        enabled: true,
        speakerVoice: "en-US-MichelleNeural",
        lang: "en-US",
        outputFormat: "audio-24khz-48kbitrate-mono-mp3",
        rate: "+0%",
        pitch: "+0%",
      },
    },
  },
}
```
  </Tab>
  <Tab title="MiniMax">
```json5
{
  tts: {
    auto: "always",
    provider: "minimax",
    providers: {
      minimax: {
        apiKey: "${MINIMAX_API_KEY}",
        model: "speech-2.8-hd",
        speakerVoiceId: "English_expressive_narrator",
        speed: 1.0,
        vol: 1.0,
        pitch: 0,
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenAI + ElevenLabs">
```json5
{
  tts: {
    auto: "always",
    provider: "openai",
    summaryModel: "openai/gpt-4.1-mini",
    modelOverrides: { enabled: true },
    providers: {
      openai: {
        apiKey: "${OPENAI_API_KEY}",
        model: "gpt-4o-mini-tts",
        speakerVoice: "alloy",
      },
      elevenlabs: {
        apiKey: "${ELEVENLABS_API_KEY}",
        model: "eleven_multilingual_v2",
        speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
        voiceSettings: { stability: 0.5, similarityBoost: 0.75, style: 0.0, useSpeakerBoost: true, speed: 1.0 },
        applyTextNormalization: "auto",
        languageCode: "en",
      },
    },
  },
}
```
  </Tab>
  <Tab title="OpenRouter">
```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        apiKey: "${OPENROUTER_API_KEY}",
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Volcengine">
```json5
{
  tts: {
    auto: "always",
    provider: "volcengine",
    providers: {
      volcengine: {
        apiKey: "${VOLCENGINE_TTS_API_KEY}",
        resourceId: "seed-tts-1.0",
        speakerVoice: "en_female_anna_mars_bigtts",
      },
    },
  },
}
```
  </Tab>
  <Tab title="xAI">
```json5
{
  tts: {
    auto: "always",
    provider: "xai",
    providers: {
      xai: {
        apiKey: "${XAI_API_KEY}",
        speakerVoiceId: "eve",
        language: "en",
        responseFormat: "mp3",
      },
    },
  },
}
```
  </Tab>
  <Tab title="Xiaomi MiMo">
```json5
{
  tts: {
    auto: "always",
    provider: "xiaomi",
    providers: {
      xiaomi: {
        apiKey: "${XIAOMI_API_KEY}",
        model: "mimo-v2.5-tts",
        speakerVoice: "mimo_default",
        format: "mp3",
      },
    },
  },
}
```
  </Tab>
</Tabs>

برای Xiaomi `mimo-v2.5-tts-voicedesign`، `speakerVoice` را حذف کنید و `style` را روی
درخواست طراحی صدا تنظیم کنید. OpenClaw آن درخواست را به‌عنوان پیام TTS با نقش `user` ارسال می‌کند
و برای مدل voicedesign، `audio.voice` را ارسال نمی‌کند.

### بازنویسی‌های صدا برای هر عامل

وقتی یک عامل باید با ارائه‌دهنده، صدا، مدل، پرسونا یا حالت TTS خودکار متفاوتی صحبت کند،
از `agents.entries.*.tts` استفاده کنید. بلوک عامل به‌صورت ادغام عمیق روی
`tts` اعمال می‌شود؛ بنابراین اطلاعات احراز هویت ارائه‌دهنده می‌توانند در پیکربندی سراسری ارائه‌دهنده باقی بمانند:

```json5
{
  tts: {
    auto: "always",
    provider: "elevenlabs",
    providers: {
      elevenlabs: { apiKey: "${ELEVENLABS_API_KEY}", model: "eleven_multilingual_v2" },
    },
  },
  agents: {
    list: [
      {
        id: "reader",
        tts: {
          providers: {
            elevenlabs: { speakerVoiceId: "EXAVITQu4vr4xnSDxMaL" },
          },
        },
      },
    ],
  },
}
```

برای تثبیت یک پرسونا برای هر عامل، `agents.entries.*.tts.persona` را در کنار پیکربندی ارائه‌دهنده
تنظیم کنید — این مقدار فقط برای همان عامل، `tts.persona` سراسری را لغو می‌کند.

ترتیب تقدم برای پاسخ‌های خودکار، `/tts audio`، `/tts status` و ابزار عامل
`tts`:

1. `tts`
2. `agents.entries.*.tts` فعال
3. بازنویسی کانال، وقتی کانال از `channels.<channel>.tts` پشتیبانی می‌کند
4. بازنویسی حساب، وقتی کانال `channels.<channel>.accounts.<id>.tts` را ارسال می‌کند
5. ترجیحات محلی `/tts` برای این میزبان
6. دستورالعمل‌های درون‌خطی `[[tts:...]]` وقتی [بازنویسی‌های مبتنی بر مدل](#model-driven-directives) فعال باشند

بازنویسی‌های کانال و حساب همان ساختار `tts` را دارند و به‌صورت عمیق
روی لایه‌های پیشین ادغام می‌شوند؛ بنابراین اعتبارنامه‌های مشترک ارائه‌دهنده می‌توانند در
`tts` باقی بمانند، درحالی‌که یک کانال یا حساب ربات فقط صدای گوینده، مدل، پرسونا
یا حالت خودکار را تغییر می‌دهد:

```json5
{
  tts: {
    provider: "openai",
    providers: {
      openai: { apiKey: "${OPENAI_API_KEY}", model: "gpt-4o-mini-tts" },
    },
  },
  channels: {
    feishu: {
      accounts: {
        english: {
          tts: {
            providers: {
              openai: { speakerVoice: "shimmer" },
            },
          },
        },
      },
    },
  },
}
```

## پرسوناها

**پرسونا** یک هویت گفتاری پایدار است که می‌توان آن را به‌صورت قطعی
در میان ارائه‌دهندگان اعمال کرد. پرسونا می‌تواند یک ارائه‌دهنده را ترجیح دهد، مقصود اعلان
مستقل از ارائه‌دهنده را تعریف کند و اتصال‌های ویژهٔ ارائه‌دهنده را برای صداها، مدل‌ها، الگوهای
اعلان، بذرها و تنظیمات صدا در خود نگه دارد.

### پرسونای حداقلی

```json5
{
  tts: {
    auto: "always",
    persona: "narrator",
    personas: {
      narrator: {
        label: "راوی",
        provider: "elevenlabs",
        providers: {
          elevenlabs: {
            speakerVoiceId: "EXAVITQu4vr4xnSDxMaL",
            modelId: "eleven_multilingual_v2",
          },
        },
      },
    },
  },
}
```

### پرسونای کامل (شکل‌دهی ویژهٔ ارائه‌دهنده)

```json5
{
  tts: {
    auto: "always",
    persona: "alfred",
    personas: {
      alfred: {
        label: "آلفرد",
        description: "راوی پیشخدمت بریتانیایی با لحنی خشک و گرم.",
        provider: "google",
        fallbackPolicy: "preserve-persona",
        providers: {
          google: {
            model: "gemini-3.1-flash-tts-preview",
            speakerVoice: "Algieba",
            promptTemplate: "audio-profile-v1",
          },
          openai: { model: "gpt-4o-mini-tts", speakerVoice: "cedar" },
          elevenlabs: {
            speakerVoiceId: "voice_id",
            modelId: "eleven_multilingual_v2",
            seed: 42,
            voiceSettings: {
              stability: 0.65,
              similarityBoost: 0.8,
              style: 0.25,
              useSpeakerBoost: true,
              speed: 0.95,
            },
          },
        },
      },
    },
  },
}
```

### تفکیک پرسونا

پرسونای فعال به‌صورت قطعی انتخاب می‌شود:

1. ترجیح محلی `/tts persona <id>`، اگر تنظیم شده باشد.
2. `tts.persona`، اگر تنظیم شده باشد.
3. بدون پرسونا.

انتخاب ارائه‌دهنده ابتدا موارد صریح را بررسی می‌کند:

1. بازنویسی‌های مستقیم (CLI، Gateway، Talk و دستورالعمل‌های مجاز TTS).
2. ترجیح محلی `/tts provider <id>`.
3. `provider` پرسونای فعال.
4. `tts.provider`.
5. انتخاب خودکار رجیستری.

برای هر تلاش ارائه‌دهنده، OpenClaw پیکربندی‌ها را با این ترتیب ادغام می‌کند:

1. `tts.providers.<id>`
2. `tts.personas.<persona>.providers.<id>`
3. بازنویسی‌های درخواست مورداعتماد
4. بازنویسی‌های مجاز دستورالعمل TTS صادرشده از مدل

### شکل‌دهی سفارشی پرسونا

پیکربندی مستقل از ارائه‌دهندهٔ `personas.<id>.prompt.*` بازنشسته شده است. Doctor آن
فیلدها را حذف می‌کند و به درگاه ارائه‌دهندهٔ گفتار اشاره می‌کند. تنظیمات ارائه‌دهندهٔ
داخلی را زیر `personas.<id>.providers.<provider>` قرار دهید (برای مثال
`personaPrompt` گوگل یا `instructions` OpenAI). برای شکل‌دهی سفارشی، یک
Plugin ارائه‌دهندهٔ گفتار با `prepareSynthesis(ctx)` پیاده‌سازی کنید و پیش از اجرای
`synthesize()`، متن تنظیم‌شده، پیکربندی ارائه‌دهنده یا بازنویسی‌ها را برگردانید. این کار ساخت
اعلان بیانی را در کد ارائه‌دهنده نگه می‌دارد؛ جایی که معناشناسی درخواست مشخص است.

### سیاست بازگشت جایگزین

`fallbackPolicy` رفتار را هنگامی کنترل می‌کند که پرسونا برای ارائه‌دهندهٔ
مورد تلاش **هیچ اتصالی** ندارد:

| سیاست              | رفتار                                                                                                                                         |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `preserve-persona`  | **پیش‌فرض.** فیلدهای اعلان مستقل از ارائه‌دهنده در دسترس می‌مانند؛ ارائه‌دهنده می‌تواند از آن‌ها استفاده کند یا نادیده‌شان بگیرد.                                            |
| `provider-defaults` | پرسونا برای آن تلاش از آماده‌سازی اعلان کنار گذاشته می‌شود؛ ارائه‌دهنده از پیش‌فرض‌های خنثی خود استفاده می‌کند، درحالی‌که بازگشت به ارائه‌دهندگان دیگر ادامه می‌یابد. |
| `fail`              | تلاش آن ارائه‌دهنده را با `reasonCode: "not_configured"` و `personaBinding: "missing"` رد کنید. ارائه‌دهندگان جایگزین همچنان امتحان می‌شوند.              |

کل درخواست TTS فقط زمانی ناموفق می‌شود که **همهٔ** ارائه‌دهندگان مورد تلاش رد شوند
یا شکست بخورند.

انتخاب ارائه‌دهندهٔ نشست Talk در محدودهٔ همان نشست است. کلاینت Talk باید شناسه‌های
ارائه‌دهنده، مدل و صدا و همچنین محلی‌ها را از `talk.catalog` انتخاب کند و آن‌ها را
از طریق درخواست نشست Talk یا تحویل ارسال کند. باز کردن یک نشست صوتی نباید
`tts` یا پیش‌فرض‌های سراسری ارائه‌دهندهٔ Talk را تغییر دهد.

## دستورالعمل‌های مبتنی بر مدل

به‌طور پیش‌فرض، دستیار **می‌تواند** دستورالعمل‌های `[[tts:...]]` را برای بازنویسی
صدا، مدل یا سرعت در یک پاسخ واحد صادر کند؛ همچنین می‌تواند یک بلوک اختیاری
`[[tts:text]]...[[/tts:text]]` برای نشانه‌های بیانی داشته باشد که باید فقط در
صدا ظاهر شوند:

```text
بفرمایید.

[[tts:speakerVoiceId=pMsXgVXv3BLzUgSXRplE model=eleven_v3 speed=1.1]]
[[tts:text]](می‌خندد) ترانه را یک بار دیگر بخوان.[[/tts:text]]
```

وقتی `tts.auto` برابر با `"tagged"` باشد، برای فعال‌کردن
صدا **وجود دستورالعمل‌ها الزامی است**. تحویل جریانی بلوک، دستورالعمل‌ها را پیش از آنکه
کانال متن قابل‌مشاهده را دریافت کند حذف می‌کند، حتی اگر میان بلوک‌های مجاور تقسیم شده باشند.

`provider=...` نادیده گرفته می‌شود، مگر اینکه `modelOverrides.allowProvider: true`. وقتی یک
پاسخ `provider=...` را اعلام می‌کند، کلیدهای دیگر آن دستورالعمل فقط توسط
همان ارائه‌دهنده تجزیه می‌شوند؛ کلیدهای پشتیبانی‌نشده حذف و به‌عنوان هشدارهای
دستورالعمل TTS گزارش می‌شوند.

**کلیدهای دستورالعمل موجود:**

- `provider` (شناسهٔ ارائه‌دهندهٔ ثبت‌شده؛ نیازمند `allowProvider: true`)
- `speakerVoice` / `speakerVoiceId` (نام‌های مستعار قدیمی: `voice`، `voiceName`، `voice_name`، `google_voice`، `voiceId`)
- `model` / `google_model`
- `stability`، `similarityBoost`، `style`، `speed`، `useSpeakerBoost`
- `vol` / `volume` (بلندی صدای MiniMax، `(0, 10]`)
- `pitch` (زیر و بمی عدد صحیح MiniMax، از −12 تا 12؛ مقادیر اعشاری بریده می‌شوند)
- `emotion` (برچسب احساس Volcengine)
- `applyTextNormalization` (`auto|on|off`)
- `languageCode` (ISO 639-1)
- `seed`

**بازنویسی‌های مدل را کاملاً غیرفعال کنید:**

```json5
{ messages: { tts: { modelOverrides: { enabled: false } } } }
```

**تغییر ارائه‌دهنده را مجاز کنید و سایر کنترل‌ها را قابل‌پیکربندی نگه دارید:**

```json5
{ messages: { tts: { modelOverrides: { enabled: true, allowProvider: true, allowSeed: false } } } }
```

## فرمان‌های اسلش

فرمان واحد `/tts`. در Discord،‏ OpenClaw همچنین `/voice` را ثبت می‌کند، زیرا
`/tts` یک فرمان داخلی Discord است — متن `/tts ...` همچنان کار می‌کند.

```text
/tts off | on | status
/tts chat on | off | default
/tts latest
/tts provider <id>
/tts persona <id> | off
/tts limit <chars>
/tts summary off
/tts audio <text>
```

<Note>
فرمان‌ها به فرستندهٔ مجاز نیاز دارند (قواعد فهرست مجاز/مالک اعمال می‌شوند) و باید یا
`commands.text` یا ثبت فرمان بومی فعال باشد.
</Note>

نکات رفتاری:

- `/tts on` ترجیح محلی TTS را در `always` می‌نویسد؛ `/tts off` آن را در `off` می‌نویسد.
- `/tts chat on|off|default` یک بازنویسی خودکار TTS در محدودهٔ نشست برای گفت‌وگوی جاری می‌نویسد.
- `/tts persona <id>` ترجیح محلی پرسونا را می‌نویسد؛ `/tts persona off` آن را پاک می‌کند.
- `/tts latest` آخرین پاسخ دستیار را از رونوشت نشست جاری می‌خواند و یک بار آن را به‌صورت صدا ارسال می‌کند. برای جلوگیری از ارسال‌های صوتی تکراری، فقط هش آن پاسخ را در ورودی نشست ذخیره می‌کند.
- `/tts audio` یک پاسخ صوتی یک‌باره تولید می‌کند (TTS را **فعال نمی‌کند**).
- `/tts limit <chars>` مقادیر **100–4096** را می‌پذیرد (4096 حداکثر زیرنویس/پیام Telegram است)؛ مقادیر خارج از این بازه رد می‌شوند.
- `limit` و `summary` در **ترجیحات محلی** ذخیره می‌شوند، نه در پیکربندی اصلی.
- `/tts status` شامل جزئیات تشخیصی بازگشت جایگزین برای آخرین تلاش است — `Fallback: <primary> -> <used>`، `Attempts: ...` و جزئیات هر تلاش (`provider:outcome(reasonCode) latency`).
- `/status` حالت فعال TTS و نیز ارائه‌دهنده، مدل، صدا و فرادادهٔ پالایش‌شدهٔ نقطهٔ پایانی سفارشی را هنگام فعال‌بودن TTS نمایش می‌دهد.

## ترجیحات هر کاربر

فرمان‌های اسلش، بازنویسی‌های محلی را در مسیر ترجیحات TTS می‌نویسند. مقدار پیش‌فرض
`~/.openclaw/settings/tts.json` است؛ آن را با `OPENCLAW_TTS_PREFS` بازنویسی کنید. Doctor
مقدار سراسری بازنشسته‌شدهٔ `tts.prefsPath` را به وضعیت مشترک دستگاه منتقل می‌کند.
راه‌اندازی‌های پیشرفتهٔ چندعاملی همچنان می‌توانند `agents.entries.<id>.tts.prefsPath` را تنظیم کنند،
وقتی عامل‌ها عمداً از مخازن ترجیحات جداگانه استفاده می‌کنند.

| فیلد ذخیره‌شده | اثر                                                                           |
| ------------ | -------------------------------------------------------------------------------- |
| `auto`       | بازنویسی محلی TTS خودکار (`always`، `off`، …)                                     |
| `provider`   | بازنویسی محلی ارائه‌دهندهٔ اصلی                                                  |
| `persona`    | بازنویسی محلی پرسونا                                                           |
| `maxLength`  | آستانهٔ خلاصه‌سازی/کوتاه‌سازی (پیش‌فرض `1500` نویسه، بازهٔ `/tts limit` برابر با 100–4096) |
| `summarize`  | کلید تغییر وضعیت خلاصه‌سازی (پیش‌فرض `true`)                                                  |

این موارد، پیکربندی مؤثر حاصل از `tts` به‌اضافهٔ بلوک فعال
`agents.entries.*.tts` را برای آن میزبان بازنویسی می‌کنند.

## قالب‌های خروجی

تحویل صدای TTS بر اساس قابلیت‌های کانال انجام می‌شود. Pluginهای کانال اعلام می‌کنند
که آیا TTS با سبک صوتی باید از ارائه‌دهندگان یک هدف بومی `voice-note` درخواست کند یا
سنتز عادی `audio-file` را نگه دارد، و آیا کانال خروجی
غیربومی را پیش از ارسال تبدیل کدگذاری می‌کند.

| مقصد                                  | قالب                                                                                                                                |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Feishu / Matrix / Telegram / WhatsApp | پاسخ‌های پیام صوتی، **Opus** را ترجیح می‌دهند (`opus_48000_64` از ElevenLabs، `opus` از OpenAI). 48 kHz / 64 kbps میان وضوح و اندازه تعادل برقرار می‌کند. |
| کانال‌های دیگر                        | **MP3** (`mp3_44100_128` از ElevenLabs، `mp3` از OpenAI). 44.1 kHz / 128 kbps تعادل پیش‌فرض برای گفتار است.                  |
| مکالمه / تلفن                         | **PCM** بومی ارائه‌دهنده (Inworld با 22050 Hz، Google با 24 kHz)، یا `ulaw_8000` از Gradium برای تلفن.                                 |

نکات مربوط به هر ارائه‌دهنده:

- **تبدیل کدگذاری Feishu / WhatsApp:** وقتی پاسخ پیام صوتی به‌صورت MP3/WebM/WAV/M4A یا فایل صوتی محتمل دیگری دریافت شود، Plugin کانال پیش از ارسال پیام صوتی بومی، آن را با `ffmpeg` (`libopus`، 64 kbps) به Ogg/Opus با نرخ 48 kHz تبدیل می‌کند. WhatsApp نتیجه را از طریق محموله Baileys با نام `audio` همراه با `ptt: true` و `audio/ogg; codecs=opus` ارسال می‌کند. در صورت شکست تبدیل کدگذاری: Feishu خطا را مدیریت می‌کند و به ارسال فایل اصلی به‌صورت پیوست معمولی برمی‌گردد؛ WhatsApp راهکار جایگزینی ندارد، بنابراین خود ارسال ناموفق می‌شود و محموله PTT ناسازگار منتشر نمی‌شود.
- **MiniMax:** برای پیوست‌های صوتی معمولی، MP3 (مدل `speech-2.8-hd`، نرخ نمونه‌برداری 32 kHz)؛ برای مقصدهای پیام صوتی اعلام‌شده توسط کانال، با `ffmpeg` به Opus با نرخ 48 kHz تبدیل می‌شود.
- **Xiaomi MiMo:** به‌طور پیش‌فرض MP3، یا در صورت پیکربندی WAV؛ برای مقصدهای پیام صوتی اعلام‌شده توسط کانال، با `ffmpeg` به Opus با نرخ 48 kHz تبدیل می‌شود.
- **CLI محلی:** از `outputFormat` پیکربندی‌شده استفاده می‌کند. مقصدهای پیام صوتی به Ogg/Opus تبدیل می‌شوند و خروجی تلفن با `ffmpeg` به PCM خام تک‌کاناله 16 kHz تبدیل می‌شود.
- **Google Gemini:** PCM خام 24 kHz برمی‌گرداند. OpenClaw آن را برای پیوست‌های صوتی در قالب WAV قرار می‌دهد، برای مقصدهای پیام صوتی به Opus با نرخ 48 kHz تبدیل می‌کند و برای مکالمه/تلفن، PCM را مستقیماً برمی‌گرداند.
- **Gradium:** WAV برای پیوست‌های صوتی، Opus برای مقصدهای پیام صوتی و `ulaw_8000` با نرخ 8 kHz برای تلفن.
- **Inworld:** MP3 برای پیوست‌های صوتی معمولی، `OGG_OPUS` بومی برای مقصدهای پیام صوتی و `PCM` خام با نرخ 22050 Hz برای مکالمه/تلفن.
- **xAI:** به‌طور پیش‌فرض MP3؛ ساخت فایل صوتی ممکن است برای خروجی بافرشده و جریانی از `mp3`، `wav`، `pcm`، `mulaw` یا `alaw` استفاده کند. مقصدهای پیام صوتی برای حالت جریانی و جایگزین بافرشده از MP3 استفاده می‌کنند، زیرا خروجی‌های `pcm`، `mulaw` و `alaw` در xAI صوت خام بدون سربرگ هستند. ساخت بافرشده از نقطه پایانی دسته‌ای REST در xAI با نام `/v1/tts` استفاده می‌کند؛ `textToSpeechStream` از `wss://api.x.ai/v1/tts` بومی استفاده می‌کند. این قرارداد صوتی بلادرنگ نیست. قالب بومی پیام صوتی Opus پشتیبانی نمی‌شود.
- **Microsoft:** از `microsoft.outputFormat` استفاده می‌کند (پیش‌فرض `audio-24khz-48kbitrate-mono-mp3`).
  - انتقال‌دهنده همراه، یک `outputFormat` را می‌پذیرد، اما همه قالب‌ها از طریق سرویس در دسترس نیستند.
  - مقادیر قالب خروجی از قالب‌های خروجی Microsoft Speech پیروی می‌کنند (از جمله Ogg/WebM Opus).
  - Telegram `sendVoice`، قالب‌های OGG/MP3/M4A را می‌پذیرد؛ اگر به پیام‌های صوتی Opus تضمین‌شده نیاز دارید، از OpenAI/ElevenLabs استفاده کنید.
  - اگر قالب خروجی پیکربندی‌شده Microsoft ناموفق باشد، OpenClaw دوباره با MP3 تلاش می‌کند.
  - وقتی جایگزین صریحی برای صدا تنظیم نشده باشد و صدای پیش‌فرض انگلیسی استفاده شود، اگر متن پاسخ عمدتاً CJK باشد، OpenClaw به‌طور خودکار به یک صدای عصبی چینی (`zh-CN-XiaoxiaoNeural`، منطقه زبانی `zh-CN`) تغییر می‌کند.

قالب‌های خروجی OpenAI و ElevenLabs برای هر کانال مطابق فهرست بالا ثابت هستند.

## رفتار TTS خودکار

وقتی `tts.auto` فعال باشد، OpenClaw:

- اگر پاسخ از قبل شامل رسانه ساخت‌یافته باشد، TTS را نادیده می‌گیرد.
- پاسخ‌های بسیار کوتاه (کمتر از 10 نویسه) را نادیده می‌گیرد.
- هنگامی که خلاصه‌ها فعال باشند، پاسخ‌های طولانی را با استفاده از
  `summaryModel` (یا `agents.defaults.model.primary`) خلاصه می‌کند.
- صدای تولیدشده را به پاسخ پیوست می‌کند.
- در `mode: "final"`، پس از تکمیل جریان متن، همچنان TTS فقط‌صوتی را برای پاسخ‌های نهایی جریانی
  ارسال می‌کند؛ رسانه تولیدشده همان فرایند عادی‌سازی رسانه کانال را طی می‌کند
  که پیوست‌های معمولی پاسخ طی می‌کنند.

اگر پاسخ از `maxLength` فراتر رود، OpenClaw هرگز صدا را به‌طور کامل نادیده نمی‌گیرد:

- **خلاصه‌سازی روشن** (پیش‌فرض) و یک مدل خلاصه‌سازی در دسترس است: متن را
  تقریباً به `maxLength` نویسه خلاصه می‌کند، سپس خلاصه را به گفتار تبدیل می‌کند.
- **خلاصه‌سازی خاموش**، خلاصه‌سازی ناموفق است، یا هیچ کلید API برای
  مدل خلاصه‌سازی در دسترس نیست: متن را به `maxLength` نویسه کوتاه می‌کند و متن
  کوتاه‌شده را به گفتار تبدیل می‌کند.

```text
پاسخ -> TTS فعال است؟
  خیر  -> ارسال متن
  بله -> دارای رسانه است / کوتاه است؟
          بله -> ارسال متن
          خیر  -> طول > محدودیت؟
                   خیر  -> TTS -> پیوست‌کردن صدا
                   بله -> خلاصه‌سازی فعال و در دسترس است؟
                            خیر  -> کوتاه‌کردن -> TTS -> پیوست‌کردن صدا
                            بله -> خلاصه‌کردن -> TTS -> پیوست‌کردن صدا
```

## مرجع فیلدها

<AccordionGroup>
  <Accordion title="tts.* سطح بالا">
    <ParamField path="auto" type='"off" | "always" | "inbound" | "tagged"'>
      حالت خودکار TTS. `inbound` فقط پس از یک پیام صوتی ورودی، صدا ارسال می‌کند؛ `tagged` فقط هنگامی صدا ارسال می‌کند که پاسخ شامل دستورهای `[[tts:...]]` یا یک بلوک `[[tts:text]]` باشد.
    </ParamField>
    <ParamField path="enabled" type="boolean" deprecated>
      کلید تغییر وضعیت قدیمی. `openclaw doctor --fix` آن را به `auto` مهاجرت می‌دهد.
    </ParamField>
    <ParamField path="mode" type='"final" | "all"' default="final">
      `"all"` علاوه بر پاسخ‌های نهایی، پاسخ‌های ابزار/بلوک را نیز شامل می‌شود.
    </ParamField>
    <ParamField path="provider" type="string">
      شناسه ارائه‌دهنده گفتار. اگر تنظیم نشده باشد، OpenClaw نخستین ارائه‌دهنده پیکربندی‌شده را بر اساس ترتیب انتخاب خودکار رجیستری استفاده می‌کند. `provider: "edge"` قدیمی به‌وسیله `openclaw doctor --fix` به `"microsoft"` بازنویسی می‌شود.
    </ParamField>
    <ParamField path="persona" type="string">
      شناسه پرسونای فعال از `personas`. به حروف کوچک نرمال‌سازی می‌شود.
    </ParamField>
    <ParamField path="personas.<id>" type="object">
      هویت گفتاری پایدار. فیلدها: `label`، `description`، `provider`، `fallbackPolicy`، `prompt`، `providers.<provider>`. به [پرسوناها](#personas) مراجعه کنید.
    </ParamField>
    <ParamField path="summaryModel" type="string">
      مدل کم‌هزینه برای خلاصه‌سازی خودکار؛ مقدار پیش‌فرض `agents.defaults.model.primary` است. `provider/model` یا نام مستعار یک مدل پیکربندی‌شده را می‌پذیرد.
    </ParamField>
    <ParamField path="modelOverrides" type="object">
      به مدل اجازه می‌دهد دستورهای TTS را تولید کند. مقدار پیش‌فرض `enabled` برابر `true` است؛ مقدار پیش‌فرض `allowProvider` برابر `false` است.
    </ParamField>
    <ParamField path="providers.<id>" type="object">
      تنظیمات متعلق به ارائه‌دهنده که بر اساس شناسه ارائه‌دهنده گفتار کلیدگذاری شده‌اند. بلوک‌های مستقیم قدیمی (`tts.openai`، `.elevenlabs`، `.microsoft`، `.edge`) به‌وسیله `openclaw doctor --fix` بازنویسی می‌شوند؛ فقط `tts.providers.<id>` را ثبت کنید.
    </ParamField>
    <ParamField path="maxTextLength" type="number" default="4096">
      سقف قطعی نویسه‌های ورودی TTS. `/tts audio`، `tts.convert` و `tts.speak` در صورت عبور از آن ناموفق می‌شوند.
    </ParamField>
    <ParamField path="timeoutMs" type="number" default="30000">
      مهلت زمانی درخواست برحسب میلی‌ثانیه. اگر `timeoutMs` مختص هر فراخوانی (ابزار عامل، Gateway) تنظیم شده باشد، اولویت دارد؛ در غیر این صورت، `tts.timeoutMs` که به‌صراحت پیکربندی شده باشد بر هر مقدار پیش‌فرض ارائه‌دهنده که Plugin تعیین کرده است اولویت دارد.
    </ParamField>
  </Accordion>

فیلدهای `apiKey` ارائه‌دهنده می‌توانند رشته‌های خام یا SecretRef باشند. هنگام راه‌اندازی سرد Gateway،
یک SecretRef ناموجود برای TTS، قابلیت داخلی TTS را به‌جای متوقف‌کردن Gateway
به‌عنوان پیکربندی‌شده-ناموجود علامت‌گذاری می‌کند. سپس `tts.speak`
مقدار `UNAVAILABLE` را با دلیل `SECRET_SURFACE_UNAVAILABLE` برمی‌گرداند و هیچ درخواستی برای
ارائه‌دهنده ارسال نمی‌شود. وضعیت و doctor مالک تنزل‌یافته TTS و مسیرهای پیکربندی آن را فهرست می‌کنند.
ارجاع‌های صریح در اسنپ‌شات زمان اجرا باقی می‌مانند، بنابراین اعتبارنامه‌های محیط یا پروفایل
نمی‌توانند بدون اعلام، حساب دیگری را انتخاب کنند. بارگذاری‌های مجدد و پیش‌بررسی نوشتن پیکربندی،
سیاست تنزل آگاه از مالک را اعمال می‌کنند: یک مالک واجد شرایط و بدون تغییر TTS
می‌تواند آخرین اعتبارنامه‌های سالم خود را به‌صورت کهنه حفظ کند، درحالی‌که یک خرابی جدید یا تغییریافته
بدون مسدودکردن مالکان سالم به حالت سرد درمی‌آید. ارجاع‌های نامعتبر از نظر ساختاری
و مقادیر حل‌شده همچنان باعث شکست راه‌اندازی یا رد به‌روزرسانی می‌شوند.

  <Accordion title="Azure Speech">
    <ParamField path="apiKey" type="string">محیط: `AZURE_SPEECH_KEY`، `AZURE_SPEECH_API_KEY` یا `SPEECH_KEY`.</ParamField>
    <ParamField path="region" type="string">منطقه Azure Speech (برای نمونه `eastus`). محیط: `AZURE_SPEECH_REGION` یا `SPEECH_REGION`.</ParamField>
    <ParamField path="endpoint" type="string">بازنویسی اختیاری نقطه پایانی Azure Speech (نام مستعار `baseUrl`).</ParamField>
    <ParamField path="speakerVoice" type="string">ShortName صدای Azure. مقدار پیش‌فرض `en-US-JennyNeural`. نام مستعار قدیمی: `voice`.</ParamField>
    <ParamField path="lang" type="string">کد زبان SSML. مقدار پیش‌فرض `en-US`.</ParamField>
    <ParamField path="outputFormat" type="string">`X-Microsoft-OutputFormat` در Azure برای صدای استاندارد. مقدار پیش‌فرض `audio-24khz-48kbitrate-mono-mp3`.</ParamField>
    <ParamField path="voiceNoteOutputFormat" type="string">`X-Microsoft-OutputFormat` در Azure برای خروجی یادداشت صوتی. مقدار پیش‌فرض `ogg-24khz-16bit-mono-opus`.</ParamField>
  </Accordion>

  <Accordion title="ElevenLabs">
    <ParamField path="apiKey" type="string">در صورت نیاز از `ELEVENLABS_API_KEY` یا `XI_API_KEY` استفاده می‌کند.</ParamField>
    <ParamField path="model" type="string">شناسه مدل. مقدار پیش‌فرض `eleven_multilingual_v2`. شناسه‌های قدیمی `eleven_turbo_v2_5`/`eleven_turbo_v2` به مدل متناظر `flash` نرمال‌سازی می‌شوند.</ParamField>
    <ParamField path="speakerVoiceId" type="string">شناسه صدای ElevenLabs. مقدار پیش‌فرض `pMsXgVXv3BLzUgSXRplE`. نام مستعار قدیمی: `voiceId`.</ParamField>
    <ParamField path="voiceSettings" type="object">
      `stability`، `similarityBoost`، `style` (هرکدام `0..1`، با مقادیر پیش‌فرض `0.5`/`0.75`/`0`)؛ `useSpeakerBoost` (`true|false`، مقدار پیش‌فرض `true`)؛ `speed` (`0.5..2.0`، مقدار پیش‌فرض `1.0`).
    </ParamField>
    <ParamField path="applyTextNormalization" type='"auto" | "on" | "off"'>حالت نرمال‌سازی متن.</ParamField>
    <ParamField path="languageCode" type="string">کد ۲ حرفی ISO 639-1 (برای نمونه `en`، `de`).</ParamField>
    <ParamField path="seed" type="number">عدد صحیح `0..4294967295` برای قطعی‌بودن در حد بهترین تلاش.</ParamField>
    <ParamField path="baseUrl" type="string">بازنویسی URL پایه API در ElevenLabs.</ParamField>
  </Accordion>

  <Accordion title="Google Gemini">
    <ParamField path="apiKey" type="string">در صورت نبود، از `GEMINI_API_KEY` / `GOOGLE_API_KEY` استفاده می‌شود. اگر حذف شود، TTS می‌تواند پیش از رجوع به متغیر محیطی، از `models.providers.google.apiKey` دوباره استفاده کند.</ParamField>
    <ParamField path="model" type="string">مدل TTS سرویس Gemini. پیش‌فرض `gemini-3.1-flash-tts-preview`.</ParamField>
    <ParamField path="speakerVoice" type="string">نام صدای ازپیش‌ساخته‌شده Gemini. پیش‌فرض `Kore`. نام‌های مستعار قدیمی: `voiceName`، `voice`.</ParamField>
    <ParamField path="audioProfile" type="string">درخواست سبک به زبان طبیعی که پیش از متن گفتاری افزوده می‌شود.</ParamField>
    <ParamField path="speakerName" type="string">برچسب اختیاری گوینده که وقتی درخواست شما از گوینده‌ای نام‌گذاری‌شده استفاده می‌کند، پیش از متن گفتاری افزوده می‌شود.</ParamField>
    <ParamField path="promptTemplate" type='"audio-profile-v1"'>روی `audio-profile-v1` تنظیم کنید تا فیلدهای فعال درخواست شخصیت در یک ساختار قطعی درخواست TTS سرویس Gemini قرار گیرند.</ParamField>
    <ParamField path="personaPrompt" type="string">متن اضافی درخواست شخصیت ویژه Google که به یادداشت‌های کارگردان الگو افزوده می‌شود.</ParamField>
    <ParamField path="baseUrl" type="string">فقط `https://generativelanguage.googleapis.com` پذیرفته می‌شود.</ParamField>
  </Accordion>

  <Accordion title="Gradium">
    <ParamField path="apiKey" type="string">متغیر محیطی: `GRADIUM_API_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">نشانی HTTPS رابط API سرویس Gradium روی `api.gradium.ai`. پیش‌فرض `https://api.gradium.ai`.</ParamField>
    <ParamField path="speakerVoiceId" type="string">پیش‌فرض Emma ‏(`YTpq7expH9539ERJ`). نام مستعار قدیمی: `voiceId`.</ParamField>
  </Accordion>

  <Accordion title="Inworld">
    ### Inworld اصلی

    <ParamField path="apiKey" type="string">متغیر محیطی: `INWORLD_API_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">پیش‌فرض `https://api.inworld.ai`.</ParamField>
    <ParamField path="modelId" type="string">پیش‌فرض `inworld-tts-1.5-max`. همچنین: `inworld-tts-1.5-mini`، `inworld-tts-1-max`، `inworld-tts-1`.</ParamField>
    <ParamField path="speakerVoiceId" type="string">پیش‌فرض `Sarah`. نام مستعار قدیمی: `voiceId`.</ParamField>
    <ParamField path="temperature" type="number">دمای نمونه‌برداری `0..2` (به‌استثنای 0).</ParamField>

  </Accordion>

  <Accordion title="CLI محلی (tts-local-cli)">
    <ParamField path="command" type="string">فایل اجرایی محلی یا رشته فرمان برای TTS مبتنی بر CLI.</ParamField>
    <ParamField path="args" type="string[]">آرگومان‌های فرمان. از جای‌نگهدارهای `{{Text}}`، `{{OutputPath}}`، `{{OutputDir}}`، `{{OutputBase}}` پشتیبانی می‌کند.</ParamField>
    <ParamField path="outputFormat" type='"mp3" | "opus" | "wav"'>قالب خروجی مورد انتظار CLI. پیش‌فرض `mp3` برای پیوست‌های صوتی.</ParamField>
    <ParamField path="timeoutMs" type="number">مهلت زمانی فرمان برحسب میلی‌ثانیه. پیش‌فرض `120000`.</ParamField>
    <ParamField path="cwd" type="string">دایرکتوری کاری اختیاری فرمان.</ParamField>
    <ParamField path="env" type="Record<string, string>">مقادیر بازنویسی اختیاری محیط برای فرمان.</ParamField>

    خروجی استاندارد فرمان و صوت تولیدشده یا تبدیل‌شده به 50 MiB محدود هستند. خروجی خطای تشخیصی به 1 MiB محدود است. اگر هرکدام از این محدودیت‌ها رد شود، OpenClaw فرمان را خاتمه می‌دهد و ترکیب گفتار ناموفق می‌شود.

  </Accordion>

  <Accordion title="Microsoft (بدون کلید API)">
    <ParamField path="enabled" type="boolean" default="true">اجازه استفاده از گفتار Microsoft.</ParamField>
    <ParamField path="speakerVoice" type="string">نام صدای عصبی Microsoft (برای مثال `en-US-MichelleNeural`). نام مستعار قدیمی: `voice`. اگر صدای پیش‌فرض انگلیسی فعال باشد و متن پاسخ عمدتاً CJK باشد، OpenClaw به‌طور خودکار به `zh-CN-XiaoxiaoNeural` تغییر می‌کند.</ParamField>
    <ParamField path="lang" type="string">کد زبان (برای مثال `en-US`).</ParamField>
    <ParamField path="outputFormat" type="string">قالب خروجی Microsoft. پیش‌فرض `audio-24khz-48kbitrate-mono-mp3`. انتقال مبتنی بر Edge همراه از همه قالب‌ها پشتیبانی نمی‌کند.</ParamField>
    <ParamField path="rate / pitch / volume" type="string">رشته‌های درصدی (برای مثال `+10%`، `-5%`).</ParamField>
    <ParamField path="saveSubtitles" type="boolean">زیرنویس‌های JSON را در کنار فایل صوتی بنویسید.</ParamField>
    <ParamField path="proxy" type="string">نشانی پراکسی برای درخواست‌های گفتار Microsoft.</ParamField>
    <ParamField path="timeoutMs" type="number">بازنویسی مهلت زمانی درخواست (ms).</ParamField>
    <ParamField path="edge.*" type="object" deprecated>نام مستعار قدیمی. برای بازنویسی پیکربندی ذخیره‌شده به `providers.microsoft`، دستور `openclaw doctor --fix` را اجرا کنید.</ParamField>
  </Accordion>

  <Accordion title="MiniMax">
    <ParamField path="apiKey" type="string">در صورت نبود، از `MINIMAX_API_KEY` استفاده می‌شود. احراز هویت Token Plan از طریق `MINIMAX_OAUTH_TOKEN`، `MINIMAX_CODE_PLAN_KEY` یا `MINIMAX_CODING_API_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">پیش‌فرض `https://api.minimax.io`. متغیر محیطی: `MINIMAX_API_HOST`.</ParamField>
    <ParamField path="model" type="string">پیش‌فرض `speech-2.8-hd`. متغیر محیطی: `MINIMAX_TTS_MODEL`.</ParamField>
    <ParamField path="speakerVoiceId" type="string">پیش‌فرض `English_expressive_narrator`. متغیر محیطی: `MINIMAX_TTS_VOICE_ID`. نام مستعار قدیمی: `voiceId`.</ParamField>
    <ParamField path="speed" type="number">`0.5..2.0`. پیش‌فرض `1.0`.</ParamField>
    <ParamField path="vol" type="number">`(0, 10]`. پیش‌فرض `1.0`.</ParamField>
    <ParamField path="pitch" type="number">عدد صحیح `-12..12`. پیش‌فرض `0`. مقادیر اعشاری پیش از درخواست بریده می‌شوند.</ParamField>
  </Accordion>

  <Accordion title="OpenAI">
    <ParamField path="apiKey" type="string">در صورت نبود، از `OPENAI_API_KEY` استفاده می‌شود.</ParamField>
    <ParamField path="model" type="string">شناسه مدل TTS سرویس OpenAI. پیش‌فرض `gpt-4o-mini-tts`.</ParamField>
    <ParamField path="speakerVoice" type="string">نام صدا (برای مثال `alloy`، `cedar`). پیش‌فرض `coral`. نام مستعار قدیمی: `voice`.</ParamField>
    <ParamField path="instructions" type="string">فیلد صریح `instructions` سرویس OpenAI. وقتی تنظیم شود، فیلدهای درخواست شخصیت به‌طور خودکار نگاشت **نمی‌شوند**.</ParamField>
    <ParamField path="extraBody / extra_body" type="Record<string, unknown>">فیلدهای اضافی JSON که پس از فیلدهای تولیدشده TTS سرویس OpenAI در بدنه درخواست‌های `/audio/speech` ادغام می‌شوند. از این گزینه برای نقاط پایانی سازگار با OpenAI مانند Kokoro استفاده کنید که به کلیدهای ویژه ارائه‌دهنده مانند `lang` نیاز دارند؛ کلیدهای ناامن prototype نادیده گرفته می‌شوند.</ParamField>
    <ParamField path="baseUrl" type="string">
      نقطه پایانی TTS سرویس OpenAI را بازنویسی کنید. ترتیب تفکیک: پیکربندی ← `OPENAI_TTS_BASE_URL` ← `https://api.openai.com/v1`. مقادیر غیرازپیش‌فرض به‌عنوان نقاط پایانی TTS سازگار با OpenAI در نظر گرفته می‌شوند؛ بنابراین نام‌های سفارشی مدل و صدا پذیرفته می‌شوند و `speed` بررسی بازه `0.25..4.0` خود را از دست می‌دهد.
    </ParamField>
  </Accordion>

  <Accordion title="OpenRouter">
    <ParamField path="apiKey" type="string">متغیر محیطی: `OPENROUTER_API_KEY`. می‌تواند از `models.providers.openrouter.apiKey` دوباره استفاده کند.</ParamField>
    <ParamField path="baseUrl" type="string">پیش‌فرض `https://openrouter.ai/api/v1`. مقدار قدیمی `https://openrouter.ai/v1` نرمال‌سازی می‌شود.</ParamField>
    <ParamField path="model" type="string">پیش‌فرض `hexgrad/kokoro-82m`. نام مستعار: `modelId`.</ParamField>
    <ParamField path="speakerVoice" type="string">پیش‌فرض `af_alloy`. نام‌های مستعار قدیمی: `voice`، `voiceId`.</ParamField>
    <ParamField path="responseFormat" type='"mp3" | "pcm"'>پیش‌فرض `mp3`.</ParamField>
    <ParamField path="speed" type="number">بازنویسی سرعت بومی ارائه‌دهنده.</ParamField>
  </Accordion>

  <Accordion title="Volcengine (BytePlus Seed Speech)">
    <ParamField path="apiKey" type="string">متغیر محیطی: `VOLCENGINE_TTS_API_KEY` یا `BYTEPLUS_SEED_SPEECH_API_KEY`.</ParamField>
    <ParamField path="resourceId" type="string">پیش‌فرض `seed-tts-1.0`. متغیر محیطی: `VOLCENGINE_TTS_RESOURCE_ID`. وقتی پروژه شما مجوز TTS 2.0 دارد، از `seed-tts-2.0` استفاده کنید.</ParamField>
    <ParamField path="appKey" type="string">سرآیند کلید برنامه. پیش‌فرض `aGjiRDfUWi`. متغیر محیطی: `VOLCENGINE_TTS_APP_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">نقطه پایانی HTTP سرویس Seed Speech TTS را بازنویسی کنید. متغیر محیطی: `VOLCENGINE_TTS_BASE_URL`.</ParamField>
    <ParamField path="speakerVoice" type="string">نوع صدا. پیش‌فرض `en_female_anna_mars_bigtts`. متغیر محیطی: `VOLCENGINE_TTS_VOICE`. نام مستعار قدیمی: `voice`.</ParamField>
    <ParamField path="speedRatio" type="number">نسبت سرعت بومی ارائه‌دهنده، `0.2..3`.</ParamField>
    <ParamField path="emotion" type="string">برچسب احساس بومی ارائه‌دهنده.</ParamField>
    <ParamField path="appId / token / cluster" type="string" deprecated>فیلدهای قدیمی Volcengine Speech Console. متغیرهای محیطی: `VOLCENGINE_TTS_APPID`، `VOLCENGINE_TTS_TOKEN`، `VOLCENGINE_TTS_CLUSTER` (پیش‌فرض `volcano_tts`).</ParamField>
  </Accordion>

  <Accordion title="xAI">
    <ParamField path="apiKey" type="string">متغیر محیطی: `XAI_API_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">پیش‌فرض `https://api.x.ai/v1`. متغیر محیطی: `XAI_BASE_URL`.</ParamField>
    <ParamField path="speakerVoiceId" type="string">پیش‌فرض `eve`. با احراز هویت، `openclaw infer tts voices --provider xai` کاتالوگ داخلی فعلی را دریافت می‌کند؛ بدون احراز هویت، جایگزین‌های آفلاین `ara`، `eve`، `leo`، `rex` و `sal` را فهرست می‌کند. شناسه‌های صدای سفارشی حساب حتی در صورت نبودن در فهرست داخلی ارسال می‌شوند. نام مستعار قدیمی: `voiceId`.</ParamField>
    <ParamField path="language" type="string">کد زبان BCP-47 یا `auto`. پیش‌فرض `en`.</ParamField>
    <ParamField path="responseFormat" type='"mp3" | "wav" | "pcm" | "mulaw" | "alaw"'>پیش‌فرض `mp3`.</ParamField>
    <ParamField path="speed" type="number">بازنویسی سرعت بومی ارائه‌دهنده، `0.7..1.5`.</ParamField>
  </Accordion>

  <Accordion title="Xiaomi MiMo">
    <ParamField path="apiKey" type="string">متغیر محیطی: `XIAOMI_API_KEY`.</ParamField>
    <ParamField path="baseUrl" type="string">پیش‌فرض `https://api.xiaomimimo.com/v1`. متغیر محیطی: `XIAOMI_BASE_URL`.</ParamField>
    <ParamField path="model" type="string">پیش‌فرض `mimo-v2.5-tts`. متغیر محیطی: `XIAOMI_TTS_MODEL`. از `mimo-v2.5-tts-voicedesign` نیز پشتیبانی می‌کند.</ParamField>
    <ParamField path="speakerVoice" type="string">پیش‌فرض `mimo_default` برای مدل‌های صدای ازپیش‌تنظیم‌شده. متغیر محیطی: `XIAOMI_TTS_VOICE`. نام مستعار قدیمی: `voice`. برای `mimo-v2.5-tts-voicedesign` ارسال نمی‌شود.</ParamField>
    <ParamField path="format" type='"mp3" | "wav"'>پیش‌فرض `mp3`. متغیر محیطی: `XIAOMI_TTS_FORMAT`.</ParamField>
    <ParamField path="style" type="string">دستور اختیاری سبک به زبان طبیعی که به‌صورت پیام کاربر ارسال می‌شود و خوانده نمی‌شود. برای `mimo-v2.5-tts-voicedesign`، این همان درخواست طراحی صدا است؛ اگر حذف شود، OpenClaw مقداری پیش‌فرض ارائه می‌کند.</ParamField>
  </Accordion>
</AccordionGroup>

## ابزار عامل

ابزار `tts` متن را به گفتار تبدیل می‌کند و برای تحویل پاسخ، یک پیوست صوتی برمی‌گرداند. در Feishu، Matrix، Telegram و WhatsApp، صوت به‌جای پیوست فایل به‌صورت پیام صوتی تحویل داده می‌شود. در این مسیر، وقتی `ffmpeg` در دسترس باشد، Feishu و WhatsApp می‌توانند خروجی TTS غیر Opus را تبدیل قالب کنند.

WhatsApp صوت را از طریق Baileys به‌صورت یادداشت صوتی PTT ‏(`audio` با `ptt: true`) ارسال می‌کند و متن قابل‌مشاهده را **جداگانه** از صوت PTT می‌فرستد، زیرا کلاینت‌ها زیرنویس یادداشت‌های صوتی را به‌طور یکسان نمایش نمی‌دهند.

این ابزار فیلدهای اختیاری `channel` و `timeoutMs` را می‌پذیرد؛ `timeoutMs` مهلت زمانی درخواست ارائه‌دهنده برای هر فراخوانی برحسب میلی‌ثانیه است. مقادیر هر فراخوانی، `tts.timeoutMs` را بازنویسی می‌کنند؛ مهلت‌های زمانی پیکربندی‌شده TTS هر مقدار پیش‌فرض ارائه‌دهنده را که Plugin تعیین کرده باشد، بازنویسی می‌کنند.

## RPC سرویس Gateway

| روش            | هدف                                      |
| ----------------- | -------------------------------------------- |
| `tts.status`      | خواندن وضعیت فعلی TTS و آخرین تلاش.     |
| `tts.enable`      | تنظیم ترجیح خودکار محلی روی `always`.       |
| `tts.disable`     | تنظیم ترجیح خودکار محلی روی `off`.          |
| `tts.convert`     | تبدیل یک‌بارهٔ متن به صدا.                        |
| `tts.setProvider` | تنظیم ترجیح ارائه‌دهندهٔ محلی.               |
| `tts.personas`    | فهرست‌کردن شخصیت‌های پیکربندی‌شده و شخصیت فعال. |
| `tts.setPersona`  | تنظیم ترجیح شخصیت محلی.                |
| `tts.providers`   | فهرست‌کردن ارائه‌دهندگان پیکربندی‌شده و وضعیت آن‌ها.        |

## پیوندهای سرویس

- [راهنمای تبدیل متن به گفتار OpenAI](https://platform.openai.com/docs/guides/text-to-speech)
- [مرجع Audio API در OpenAI](https://platform.openai.com/docs/api-reference/audio)
- [تبدیل متن به گفتار با REST در Azure Speech](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech)
- [ارائه‌دهندهٔ Azure Speech](/fa/providers/azure-speech)
- [تبدیل متن به گفتار ElevenLabs](https://elevenlabs.io/docs/api-reference/text-to-speech)
- [احراز هویت ElevenLabs](https://elevenlabs.io/docs/api-reference/authentication)
- [Gradium](/fa/providers/gradium)
- [API تبدیل متن به گفتار Inworld](https://docs.inworld.ai/tts/tts)
- [API مدل MiniMax T2A v2](https://platform.minimaxi.com/document/T2A%20V2)
- [API مبتنی بر HTTP برای TTS در Volcengine](/fa/providers/volcengine#text-to-speech)
- [ترکیب گفتار Xiaomi MiMo](/fa/providers/xiaomi#text-to-speech)
- [node-edge-tts](https://github.com/SchneeHertz/node-edge-tts)
- [قالب‌های خروجی گفتار Microsoft](https://learn.microsoft.com/azure/ai-services/speech-service/rest-text-to-speech#audio-outputs)
- [تبدیل متن به گفتار xAI](https://docs.x.ai/developers/rest-api-reference/inference/voice#text-to-speech-rest)

## مرتبط

- [نمای کلی رسانه](/fa/tools/media-overview)
- [تولید موسیقی](/fa/tools/music-generation)
- [تولید ویدئو](/fa/tools/video-generation)
- [دستورهای اسلش](/fa/tools/slash-commands)
- [Plugin تماس صوتی](/fa/plugins/voice-call)
