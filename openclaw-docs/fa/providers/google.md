---
read_when:
    - می‌خواهید از مدل‌های Google Gemini با OpenClaw استفاده کنید
    - به کلید API یا جریان احراز هویت OAuth نیاز دارید
summary: راه‌اندازی Google Gemini (کلید API و OAuth، تولید تصویر، درک رسانه، TTS، جست‌وجوی وب)
title: Google (Gemini)
x-i18n:
    generated_at: "2026-07-27T17:01:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fdf8db70bcebd425238e5f02ca12bdbcd75fa1c03d285ea127d4e3863892b3aa
    source_path: providers/google.md
    workflow: 16
---

Plugin گوگل دسترسی به مدل‌های Gemini را از طریق Google AI Studio فراهم می‌کند و همچنین شامل تولید تصویر، درک رسانه (تصویر/صدا/ویدئو)، تبدیل متن به گفتار و جست‌وجوی وب از طریق Gemini Grounding است.

- ارائه‌دهنده: `google`
- احراز هویت: `GEMINI_API_KEY` یا `GOOGLE_API_KEY`
- API: Google Gemini API
- گزینه زمان اجرا: `agentRuntime.id: "google-gemini-cli"` از OAuth مربوط به Gemini CLI مجدداً استفاده می‌کند و هم‌زمان ارجاع‌های مدل را به‌شکل استاندارد `google/*` نگه می‌دارد.

## شروع به کار

روش احراز هویت دلخواه خود را انتخاب کنید و مراحل راه‌اندازی را دنبال کنید.

<Tabs>
  <Tab title="کلید API">
    **مناسب برای:** دسترسی استاندارد به Gemini API از طریق Google AI Studio.

    <Steps>
      <Step title="دریافت کلید API">
        یک کلید رایگان در [Google AI Studio](https://aistudio.google.com/apikey) ایجاد کنید.
      </Step>
      <Step title="اجرای راه‌اندازی اولیه">
        ```bash
        openclaw onboard --auth-choice gemini-api-key
        ```

        یا کلید را مستقیماً وارد کنید:

        ```bash
        openclaw onboard --non-interactive \
          --mode local \
          --auth-choice gemini-api-key \
          --gemini-api-key "$GEMINI_API_KEY"
        ```
      </Step>
      <Step title="تنظیم مدل پیش‌فرض">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "google/gemini-3.1-pro-preview" },
            },
          },
        }
        ```
      </Step>
      <Step title="بررسی در دسترس بودن مدل">
        ```bash
        openclaw models list --provider google
        ```
      </Step>
    </Steps>

    <Tip>
    هر دو `GEMINI_API_KEY` و `GOOGLE_API_KEY` پذیرفته می‌شوند. از هرکدام که از قبل پیکربندی کرده‌اید استفاده کنید.
    </Tip>

    با پیکربندی کلید API، ‏OpenClaw فهرست مدل‌های متنی Google AI Studio را
    از API مربوط به Gemini `models.list` به‌روزرسانی می‌کند. بنابراین گونه‌های تازه‌منتشرشده Gemini 3 Pro، ‏Flash
    و Flash-Lite بدون انتظار برای انتشار نسخه‌ای از OpenClaw در
    `openclaw models list --provider google` ظاهر می‌شوند. اگر کشف مدل‌ها در دسترس نباشد، OpenClaw فهرست
    جایگزین همراه بسته را حفظ می‌کند.

  </Tab>

  <Tab title="Gemini CLI (OAuth)">
    **مناسب برای:** ورود با حساب Google از طریق OAuth مربوط به Gemini CLI، به‌جای استفاده از یک کلید API جداگانه.

    <Warning>
    ارائه‌دهنده `google-gemini-cli` یک یکپارچه‌سازی غیررسمی است. برخی کاربران
    هنگام استفاده از OAuth به این روش، محدودیت‌هایی را برای حساب گزارش کرده‌اند. با مسئولیت خود استفاده کنید.
    </Warning>

    <Steps>
      <Step title="نصب Gemini CLI">
        فرمان محلی `gemini` باید در `PATH` در دسترس باشد.

        ```bash
        # Homebrew
        brew install gemini-cli

        # یا npm
        npm install -g @google/gemini-cli
        ```

        OpenClaw هم نصب‌های Homebrew و هم نصب‌های سراسری npm را، از جمله
        چیدمان‌های متداول Windows/npm، پشتیبانی می‌کند.
      </Step>
      <Step title="ورود از طریق OAuth">
        ```bash
        openclaw models auth login --provider google-gemini-cli --set-default
        ```
      </Step>
      <Step title="بررسی در دسترس بودن مدل">
        ```bash
        openclaw models list --provider google
        ```
      </Step>
    </Steps>

    - مدل پیش‌فرض: `google/gemini-3.1-pro-preview`
    - زمان اجرا: `google-gemini-cli`
    - نام مستعار: `gemini-cli`

    شناسه مدل Gemini API برای Gemini 3.1 Pro برابر با `gemini-3.1-pro-preview` است. OpenClaw برای سهولت، `google/gemini-3.1-pro` کوتاه‌تر را به‌عنوان نام مستعار می‌پذیرد و پیش از فراخوانی ارائه‌دهنده آن را استانداردسازی می‌کند.

    **متغیرهای محیطی:**

    - `OPENCLAW_GEMINI_OAUTH_CLIENT_ID` / `GEMINI_CLI_OAUTH_CLIENT_ID`
    - `OPENCLAW_GEMINI_OAUTH_CLIENT_SECRET` / `GEMINI_CLI_OAUTH_CLIENT_SECRET`

    <Note>
    اگر درخواست‌های OAuth مربوط به Gemini CLI پس از ورود ناموفق بودند، `GOOGLE_CLOUD_PROJECT` یا
    `GOOGLE_CLOUD_PROJECT_ID` را روی میزبان Gateway تنظیم و دوباره تلاش کنید.
    </Note>

    <Note>
    اگر ورود پیش از شروع جریان مرورگر ناموفق بود، مطمئن شوید فرمان محلی `gemini`
    نصب شده و در `PATH` قرار دارد.
    </Note>

    تشخیص خودکار راه‌اندازی اولیه، ورود موجود Gemini CLI را فهرست می‌کند اما هرگز
    آن را به‌طور خودکار آزمایش نمی‌کند، زیرا Gemini CLI کاوشگر بدون ابزار ندارد. برای ادامه، OAuth مربوط به Gemini CLI
    یا یک کلید Gemini API را انتخاب کنید.

    ارجاع‌های مدل `google-gemini-cli/*` نام‌های مستعار سازگاری قدیمی هستند. پیکربندی‌های
    جدید برای اجرای محلی Gemini CLI باید از ارجاع‌های مدل `google/*` به‌همراه
    زمان اجرای `google-gemini-cli` استفاده کنند.

  </Tab>
</Tabs>

<Note>
`google/gemini-3-pro-preview` در 2026-03-09 بازنشسته شد؛ به‌جای آن از `google/gemini-3.1-pro-preview` استفاده کنید. اجرای دوباره راه‌اندازی کلید Gemini API ‏(`openclaw onboard --auth-choice gemini-api-key` یا `openclaw models auth login --provider google`) یک مدل پیش‌فرض پیکربندی‌شده قدیمی را با مدل فعلی جایگزین می‌کند.
</Note>

## قابلیت‌ها

| قابلیت                    | پشتیبانی                       |
| ------------------------- | ------------------------------ |
| تکمیل‌های گفت‌وگو         | بله                            |
| تولید تصویر               | بله                            |
| تولید موسیقی              | بله                            |
| تبدیل متن به گفتار        | بله                            |
| صدای بلادرنگ              | بله (Google Live API)          |
| درک تصویر                 | بله                            |
| رونویسی صدا               | بله                            |
| درک ویدئو                 | بله                            |
| جست‌وجوی وب (Grounding)   | بله                            |
| تفکر/استدلال              | بله (Gemini 2.5+ / Gemini 3+)  |
| مدل‌های Gemma 4           | بله                            |

## جست‌وجوی وب

ارائه‌دهنده جست‌وجوی وب همراه بسته، یعنی `gemini`، از اتصال جست‌وجوی Google در Gemini استفاده می‌کند.
یک کلید جست‌وجوی اختصاصی را در `plugins.entries.google.config.webSearch` پیکربندی کنید،
یا اجازه دهید پس از `GEMINI_API_KEY` از `models.providers.google.apiKey` مجدداً استفاده کند:

```json5
{
  plugins: {
    entries: {
      google: {
        config: {
          webSearch: {
            apiKey: "AIza...", // در صورت تنظیم بودن GEMINI_API_KEY یا models.providers.google.apiKey اختیاری است
            baseUrl: "https://generativelanguage.googleapis.com/v1beta", // به models.providers.google.baseUrl بازمی‌گردد
            model: "gemini-2.5-flash",
          },
        },
      },
    },
  },
}
```

ترتیب اولویت اعتبارنامه‌ها ابتدا `webSearch.apiKey`، سپس `GEMINI_API_KEY`
و بعد `models.providers.google.apiKey` است. `webSearch.baseUrl` اختیاری است و
برای پراکسی‌های اپراتور یا نقاط پایانی سازگار Gemini API وجود دارد؛ در صورت حذف،
جست‌وجوی وب Gemini از `models.providers.google.baseUrl` مجدداً استفاده می‌کند. برای رفتار ابزار ویژه ارائه‌دهنده،
[جست‌وجوی Gemini](/fa/tools/gemini-search) را ببینید.

<Tip>
مدل‌های Gemini 3 به‌جای `thinkingBudget` از `thinkingLevel` استفاده می‌کنند. OpenClaw کنترل‌های
استدلال Gemini 3، ‏Gemini 3.1 و نام مستعار `gemini-*-latest` را به
`thinkingLevel` نگاشت می‌کند تا اجراهای پیش‌فرض/کم‌تأخیر مقادیر غیرفعال
`thinkingBudget` را ارسال نکنند.

`/think adaptive` به‌جای انتخاب یک سطح ثابت OpenClaw، معناشناسی تفکر پویای Google را حفظ می‌کند.
Gemini 3 و Gemini 3.1 مقدار ثابت `thinkingLevel` را حذف می‌کنند تا
Google بتواند سطح را انتخاب کند؛ Gemini 2.5 مقدار نشانه پویای Google یعنی
`thinkingBudget: -1` را ارسال می‌کند.

مدل‌های Gemma 4 (برای مثال `gemma-4-26b-a4b-it`) از حالت تفکر پشتیبانی می‌کنند. OpenClaw
برای Gemma 4، ‏`thinkingBudget` را به یک `thinkingLevel` پشتیبانی‌شده Google بازنویسی می‌کند.
تنظیم تفکر روی `off` به‌جای نگاشت به
`MINIMAL`، غیرفعال بودن تفکر را حفظ می‌کند.

Gemini 2.5 Pro فقط در حالت تفکر کار می‌کند و
`thinkingBudget: 0` صریح را رد می‌کند؛ OpenClaw به‌جای ارسال آن، این مقدار را از درخواست‌های
Gemini 2.5 Pro حذف می‌کند.
</Tip>

## تولید تصویر

ارائه‌دهنده تولید تصویر همراه بسته، یعنی `google`، به‌طور پیش‌فرض از
`google/gemini-3.1-flash-image` استفاده می‌کند.

- همچنین از `google/gemini-3-pro-image` پشتیبانی می‌کند
- تولید: حداکثر 4 تصویر در هر درخواست
- حالت ویرایش: فعال، با حداکثر 5 تصویر ورودی
- کنترل‌های هندسی: `size`، `aspectRatio` و `resolution`

برای استفاده از Google به‌عنوان ارائه‌دهنده پیش‌فرض تصویر:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "google/gemini-3.1-flash-image",
      },
    },
  },
}
```

<Note>
برای پارامترهای مشترک ابزار، انتخاب ارائه‌دهنده و رفتار جایگزینی هنگام خطا، [تولید تصویر](/fa/tools/image-generation) را ببینید.
</Note>

## تولید ویدئو

Plugin همراه بسته `google` همچنین تولید ویدئو را از طریق ابزار مشترک
`video_generate` ثبت می‌کند.

- مدل پیش‌فرض ویدئو: `google/veo-3.1-fast-generate-preview`
- حالت‌ها: متن‌به‌ویدئو، تصویر‌به‌ویدئو و جریان‌های ارجاع تک‌ویدئویی
- از `aspectRatio` ‏(`16:9`، `9:16`) و `resolution` ‏(`720P`، `1080P`) پشتیبانی می‌کند؛ Veo در حال حاضر از خروجی صدا پشتیبانی نمی‌کند
- مدت‌زمان‌های پشتیبانی‌شده: **4، 6 یا 8 ثانیه** (مقادیر دیگر به نزدیک‌ترین مقدار مجاز تبدیل می‌شوند)

برای استفاده از Google به‌عنوان ارائه‌دهنده پیش‌فرض ویدئو:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "google/veo-3.1-fast-generate-preview",
      },
    },
  },
}
```

<Note>
برای پارامترهای مشترک ابزار، انتخاب ارائه‌دهنده و رفتار جایگزینی هنگام خطا، [تولید ویدئو](/fa/tools/video-generation) را ببینید.
</Note>

## تولید موسیقی

Plugin همراه بسته `google` همچنین تولید موسیقی را از طریق ابزار مشترک
`music_generate` ثبت می‌کند.

- مدل پیش‌فرض موسیقی: `google/lyria-3-clip-preview`
- همچنین از `google/lyria-3-pro-preview` پشتیبانی می‌کند
- کنترل‌های اعلان: `lyrics` و `instrumental`
- قالب خروجی: به‌طور پیش‌فرض `mp3`، به‌علاوه `wav` در `google/lyria-3-pro-preview`
- ورودی‌های مرجع: حداکثر 10 تصویر
- اجراهای متکی به نشست از طریق جریان مشترک وظیفه/وضعیت، شامل `action: "status"`، جدا می‌شوند

برای استفاده از Google به‌عنوان ارائه‌دهنده پیش‌فرض موسیقی:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "google/lyria-3-clip-preview",
      },
    },
  },
}
```

<Note>
برای پارامترهای مشترک ابزار، انتخاب ارائه‌دهنده و رفتار جایگزینی هنگام خطا، [تولید موسیقی](/fa/tools/music-generation) را ببینید.
</Note>

## تبدیل متن به گفتار

ارائه‌دهنده گفتار همراه بسته، یعنی `google`، از مسیر TTS در Gemini API با
`gemini-3.1-flash-tts-preview` استفاده می‌کند.

- صدای پیش‌فرض: `Kore`
- احراز هویت: `tts.providers.google.apiKey`، `models.providers.google.apiKey`، `GEMINI_API_KEY` یا `GOOGLE_API_KEY`
- خروجی: WAV برای پیوست‌های عادی TTS، ‏Opus برای مقصدهای پیام صوتی و PCM برای مکالمه/تلفن
- خروجی پیام صوتی: PCM مربوط به Google در قالب WAV بسته‌بندی و با `ffmpeg` به Opus با نرخ 48 kHz تبدیل می‌شود

مسیر دسته‌ای Gemini TTS در Google، صدای تولیدشده را در پاسخ تکمیل‌شده
`generateContent` بازمی‌گرداند. برای مکالمات گفتاری با کمترین تأخیر، به‌جای TTS
دسته‌ای از ارائه‌دهنده صدای بلادرنگ Google مبتنی بر Gemini Live API استفاده کنید.

برای استفاده از Google به‌عنوان ارائه‌دهنده پیش‌فرض TTS:

```json5
{
  tts: {
    auto: "always",
    provider: "google",
    providers: {
      google: {
        model: "gemini-3.1-flash-tts-preview",
        speakerVoice: "Kore",
        audioProfile: "حرفه‌ای و با لحنی آرام صحبت کن.",
      },
    },
  },
}
```

Gemini API TTS برای کنترل سبک از اعلان زبان طبیعی استفاده می‌کند. برای افزودن یک اعلان سبک
قابل‌استفاده مجدد پیش از متن گفتاری، `audioProfile` را تنظیم کنید. هنگامی که متن اعلان
به گوینده‌ای نام‌گذاری‌شده اشاره می‌کند، `speakerName` را تنظیم کنید.

Gemini API TTS همچنین برچسب‌های صوتی بیانی داخل کروشه را در متن می‌پذیرد،
مانند `[whispers]` یا `[laughs]`. برای اینکه برچسب‌ها در پاسخ قابل‌مشاهده گفت‌وگو نمایش داده نشوند
اما به TTS ارسال شوند، آن‌ها را داخل یک بلوک `[[tts:text]]...[[/tts:text]]`
قرار دهید:

```text
این متن پاکیزه پاسخ است.

[[tts:text]][whispers] این نسخه گفتاری است.[[/tts:text]]
```

<Note>
یک کلید API در Google Cloud Console که به Gemini API محدود شده باشد، برای این
ارائه‌دهنده معتبر است. این مسیر جداگانه Cloud Text-to-Speech API نیست.
</Note>

## صدای بلادرنگ

Plugin همراه بسته `google` یک ارائه‌دهنده صدای بلادرنگ مبتنی بر
Gemini Live API را برای پل‌های صوتی بک‌اند مانند Voice Call و Google Meet ثبت می‌کند.

| تنظیم                 | مسیر پیکربندی                                                       | پیش‌فرض                                                                               |
| --------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| مدل                   | `plugins.entries.voice-call.config.realtime.providers.google.model` | `gemini-3.1-flash-live-preview`                                                       |
| صدا                   | `...google.voice`                                                   | `Kore`                                                                                |
| دما                   | `...google.temperature`                                             | (تنظیم‌نشده)                                                                               |
| حساسیت شروع VAD       | `...google.startSensitivity`                                        | (تنظیم‌نشده)                                                                               |
| حساسیت پایان VAD      | `...google.endSensitivity`                                          | (تنظیم‌نشده)                                                                               |
| مدت سکوت              | `...google.silenceDurationMs`                                       | (تنظیم‌نشده)                                                                               |
| مدیریت فعالیت         | `...google.activityHandling`                                        | پیش‌فرض Google، `start-of-activity-interrupts`                                        |
| پوشش نوبت             | `...google.turnCoverage`                                            | پیش‌فرض Google، `audio-activity-and-all-video`                                        |
| غیرفعال‌کردن VAD خودکار | `...google.automaticActivityDetectionDisabled`                      | `false`                                                                               |
| ازسرگیری نشست         | `...google.sessionResumption`                                       | `true`                                                                                |
| فشرده‌سازی زمینه      | `...google.contextWindowCompression`                                | `true`                                                                                |
| کلید API              | `...google.apiKey`                                                  | در صورت نبود، از `models.providers.google.apiKey`، `GEMINI_API_KEY` یا `GOOGLE_API_KEY` استفاده می‌کند |

نمونه پیکربندی بلادرنگ تماس صوتی:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          realtime: {
            enabled: true,
            provider: "google",
            providers: {
              google: {
                model: "gemini-3.1-flash-live-preview",
                speakerVoice: "Kore",
                activityHandling: "start-of-activity-interrupts",
                turnCoverage: "audio-activity-and-all-video",
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
Google Live API برای صدا و فراخوانی تابع از طریق WebSocket از ارتباط دوسویه استفاده می‌کند.
OpenClaw صدای پل تلفنی/Meet را با جریان PCM Live API در Gemini سازگار می‌کند و
فراخوانی ابزارها را در قرارداد مشترک صدای بلادرنگ نگه می‌دارد. مگر اینکه به تغییرات نمونه‌برداری نیاز دارید، `temperature`
را تنظیم نکنید؛ OpenClaw مقادیر غیرمثبت را حذف می‌کند،
زیرا Google Live ممکن است برای `temperature: 0` رونوشت را بدون صدا بازگرداند.
رونویسی Gemini API بدون `languageCodes` فعال است؛ SDK فعلی Google
راهنمایی‌های کد زبان را در این مسیر API رد می‌کند.
</Note>

<Note>
Gemini 3.1 Live متن محاوره‌ای را از طریق ورودی بلادرنگ می‌پذیرد و از
فراخوانی ترتیبی تابع استفاده می‌کند. OpenClaw برای این مدل، `NON_BLOCKING` قدیمی، زمان‌بندی
پاسخ تابع و فیلدهای گفت‌وگوی عاطفی را حذف می‌کند. `thinkingLevel` را
ترجیح دهید؛ مقادیر مثبت پیکربندی‌شده `thinkingBudget` به نزدیک‌ترین
سطح پشتیبانی‌شده نگاشت می‌شوند، درحالی‌که `-1` پیش‌فرض Google را حفظ می‌کند. [مقایسه قابلیت‌های Gemini Live](https://ai.google.dev/gemini-api/docs/live-api/capabilities) را ببینید.
</Note>

<Note>
گفت‌وگوی Control UI از نشست‌های مرورگری Google Live با توکن‌های
محدود و یک‌بارمصرف پشتیبانی می‌کند. در گفت‌وگوی ویدیویی، مرورگر فریم‌های JPEG محدود را مستقیماً و با
حداکثر نرخ ارائه‌دهنده، یعنی یک فریم در ثانیه، به Google Live ارسال می‌کند. تابع
`describe_view` گزارش می‌دهد که آیا آن جریان دوربین فعال است یا خیر.
فریم‌های دوربین از Gateway عبور نمی‌کنند. ارائه‌دهندگان صدای بلادرنگِ صرفاً بک‌اند
نیز می‌توانند از طریق انتقال رله عمومی Gateway اجرا شوند که
اعتبارنامه‌های ارائه‌دهنده را روی Gateway نگه می‌دارد.
</Note>

برای راستی‌آزمایی زنده توسط نگه‌دارنده، فرمان
`OPENAI_API_KEY=... GEMINI_API_KEY=... node --import tsx scripts/dev/realtime-talk-live-smoke.ts` را اجرا کنید.
آزمون دود همچنین مسیرهای بک‌اند/WebRTC در OpenAI را پوشش می‌دهد؛ بخش Google همان
قالب توکن محدود Live API مورداستفاده گفت‌وگوی Control UI را صادر می‌کند، نقطه پایانی
WebSocket مرورگر را باز می‌کند، بار راه‌اندازی اولیه را همراه با یک فریم JPEG می‌فرستد و
یک پاسخ متنی و رفت‌وبرگشت تابع `describe_view` را راستی‌آزمایی می‌کند.

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="استفاده مجدد مستقیم از کش Gemini">
    برای اجراهای مستقیم Gemini API ‏(`api: "google-generative-ai"`)، OpenClaw
    یک هندل پیکربندی‌شده `cachedContent` را به درخواست‌های Gemini منتقل می‌کند.

    - پارامترهای سراسری یا مختص هر مدل را با `cachedContent`
      یا `cached_content` قدیمی پیکربندی کنید
    - پارامترهای محدوده خاص‌تر (سطح مدل نسبت به سراسری) همیشه اولویت دارند.
      اگر هر دو کلید در یک محدوده تنظیم شده باشند، `cached_content` اولویت دارد.
      برای جلوگیری از نتایج غیرمنتظره، در هر محدوده فقط از یک کلید استفاده کنید.
    - مقدار نمونه: `cachedContents/prebuilt-context`
    - میزان استفاده ناشی از اصابت کش Gemini، از
      `cachedContentTokenCount` بالادستی به `cacheRead` در OpenClaw نرمال‌سازی می‌شود

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "google/gemini-2.5-pro": {
              params: {
                cachedContent: "cachedContents/prebuilt-context",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="نکات استفاده از Gemini CLI">
    هنگام استفاده از ارائه‌دهنده OAuth با نام `google-gemini-cli`، OpenClaw به‌طور پیش‌فرض از
    خروجی `stream-json` در Gemini CLI استفاده می‌کند و میزان استفاده را از بار نهایی
    `stats` نرمال‌سازی می‌کند. بازنویسی‌های قدیمی `--output-format json` همچنان از
    تجزیه‌گر JSON استفاده می‌کنند.

    - متن پاسخ جریانی از رویدادهای `message` دستیار می‌آید.
    - برای خروجی قدیمی JSON، متن پاسخ از فیلد `response` در JSON ابزار CLI می‌آید.
    - اگر ابزار CLI مقدار `usage` را خالی بگذارد، میزان استفاده به `stats` برمی‌گردد.
    - `stats.cached` به `cacheRead` در OpenClaw نرمال‌سازی می‌شود.
    - اگر `stats.input` وجود نداشته باشد، OpenClaw توکن‌های ورودی را از
      `stats.input_tokens - stats.cached` استخراج می‌کند.

  </Accordion>

  <Accordion title="راه‌اندازی محیط و سرویس پس‌زمینه">
    اگر Gateway به‌صورت سرویس پس‌زمینه (launchd/systemd) اجرا می‌شود، مطمئن شوید `GEMINI_API_KEY`
    برای آن فرایند در دسترس است (برای مثال، در `~/.openclaw/.env` یا از طریق
    `env.shellEnv`).
  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="تولید تصویر" href="/fa/tools/image-generation" icon="image">
    پارامترهای مشترک ابزار تصویر و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="تولید ویدیو" href="/fa/tools/video-generation" icon="video">
    پارامترهای مشترک ابزار ویدیو و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="تولید موسیقی" href="/fa/tools/music-generation" icon="music">
    پارامترهای مشترک ابزار موسیقی و انتخاب ارائه‌دهنده.
  </Card>
</CardGroup>
