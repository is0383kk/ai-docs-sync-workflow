---
read_when:
    - می‌خواهید از مدل‌های Grok در OpenClaw استفاده کنید
    - در حال پیکربندی احراز هویت xAI یا شناسه‌های مدل هستید
summary: استفاده از مدل‌های xAI Grok در OpenClaw
title: xAI
x-i18n:
    generated_at: "2026-07-27T16:07:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71ae7b049649b08b6508b8331714fec3464628814629256ad23b584f0f8ca8b7
    source_path: providers/xai.md
    workflow: 16
---

OpenClaw یک Plugin ارائه‌دهندهٔ داخلی `xai` برای مدل‌های Grok عرضه می‌کند. مسیر
توصیه‌شده، استفاده از OAuthِ Grok با اشتراک واجد شرایط SuperGrok یا X Premium
است. Gateway، پیکربندی، مسیریابی و ابزارها محلی باقی می‌مانند؛ فقط درخواست‌های
Grok به APIِ xAI ارسال می‌شوند.

OAuth به کلید APIِ xAI یا برنامهٔ Grok Build نیاز ندارد. ممکن است xAI همچنان
Grok Build را در صفحهٔ رضایت نمایش دهد، زیرا OpenClaw از کلاینت مشترک
OAuthِ xAI استفاده می‌کند.

## راه‌اندازی

<Steps>
  <Step title="نصب جدید">
    فرایند راه‌اندازی اولیه را همراه با نصب دیمن اجرا کنید، سپس در مرحلهٔ
    مدل/احراز هویت، OAuthِ xAI/Grok را انتخاب کنید:

    ```bash
    openclaw onboard --install-daemon
    ```

    در VPS یا از طریق SSH، مستقیماً xAI OAuth را انتخاب کنید؛ این روش از
    تأیید با کد دستگاه استفاده می‌کند و به فراخوانی برگشتی localhost نیاز ندارد:

    ```bash
    openclaw onboard --install-daemon --auth-choice xai-oauth
    ```

  </Step>
  <Step title="نصب موجود">
    فقط وارد xAI شوید؛ صرفاً برای اتصال Grok، فرایند کامل راه‌اندازی اولیه را دوباره اجرا نکنید:

    ```bash
    openclaw models auth login --provider xai --method oauth
    ```

    Grok را جداگانه به‌عنوان مدل پیش‌فرض اعمال کنید:

    ```bash
    openclaw models set xai/grok-4.3
    ```

    فرایند کامل راه‌اندازی اولیه را فقط زمانی دوباره اجرا کنید که عمداً می‌خواهید Gateway،
    دیمن، کانال، فضای کاری یا دیگر گزینه‌های راه‌اندازی را تغییر دهید.

  </Step>
  <Step title="مسیر کلید API">
    راه‌اندازی با کلید API همچنان برای کلیدهای xAI Console و سطوح رسانه‌ای
    نیازمند پیکربندی ارائه‌دهندهٔ متکی بر کلید کار می‌کند:

    ```bash
    openclaw models auth login --provider xai --method api-key
    export XAI_API_KEY=xai-...
    ```

  </Step>
  <Step title="انتخاب مدل">
    ```json5
    {
      agents: { defaults: { model: { primary: "xai/grok-4.3" } } },
    }
    ```
  </Step>
</Steps>

<Note>
OpenClaw از APIِ Responsesِ xAI به‌عنوان انتقال داخلی xAI استفاده می‌کند. همان
اعتبارنامهٔ `openclaw models auth login --provider xai --method oauth` یا
`--method api-key`، قابلیت‌های `web_search` (شناسهٔ ارائه‌دهنده `grok`)، `x_search`،
`code_execution`، گفتار/رونویسی و تولید تصویر/ویدیوی xAI را نیز فعال می‌کند. اگر
یک کلید xAI را زیر `plugins.entries.xai.config.webSearch.apiKey` ذخیره کنید،
ارائه‌دهندهٔ داخلی مدل xAI آن را نیز به‌عنوان گزینهٔ جایگزین استفاده می‌کند.
</Note>

## عیب‌یابی OAuth

- برای SSH، Docker، VPS یا دیگر راه‌اندازی‌های راه دور، از
  `openclaw models auth login --provider xai --method oauth` استفاده کنید؛ این روش از
  تأیید با کد دستگاه استفاده می‌کند، نه فراخوانی برگشتی localhost.
- اگر ورود موفق است اما Grok مدل پیش‌فرض نیست، دستور
  `openclaw models set xai/grok-4.3` را اجرا کنید.
- پروفایل‌های ذخیره‌شدهٔ احراز هویت xAI را بررسی کنید:

  ```bash
  openclaw models auth list --provider xai
  openclaw models status
  ```

- xAI تعیین می‌کند کدام حساب‌ها می‌توانند توکن‌های APIِ OAuth دریافت کنند. اگر حسابی
  واجد شرایط نیست، از مسیر کلید API استفاده کنید یا اشتراک را در سمت xAI بررسی کنید.

<Tip>
هنگام ورود از طریق SSH، Docker یا VPS از `xai-oauth` استفاده کنید. OpenClaw یک
URL و کد کوتاه نمایش می‌دهد؛ در حالی که فرایند راه دور برای تکمیل تبادل توکن
از xAI نظرسنجی می‌کند، ورود را در هر مرورگر محلی تکمیل کنید.
</Tip>

## کاتالوگ داخلی

شناسه‌های قابل انتخاب در انتخابگرهای مدل. Plugin همچنان شناسه‌های قدیمی Grok 3،
Grok 4، Grok 4 Fast، Grok 4.1 Fast و Grok Code را برای پیکربندی‌های موجود تفکیک می‌کند؛
[سازگاری قدیمی و نام‌های مستعار متغیر](#legacy-compatibility-and-moving-aliases) را ببینید.

| خانواده         | شناسه‌های مدل                                                    |
| -------------- | ------------------------------------------------------------ |
| Grok 4.5       | `grok-4.5` (نام‌های مستعار: `grok-4.5-latest`، `grok-build-latest`) |
| Grok Build 0.1 | `grok-build-0.1`                                             |
| Grok 4.3       | `grok-4.3` (نام‌های مستعار: `grok-4.3-latest`، `grok-latest`)       |
| Grok 4.20      | `grok-4.20-0309-reasoning`، `grok-4.20-0309-non-reasoning`   |

<Tip>
برای گفت‌وگوی عمومی، کدنویسی و کارهای عامل‌محور، هرجا در دسترس است از `grok-4.5` استفاده کنید.
Grok 4.3 همچنان پیش‌فرض راه‌اندازی ایمن از نظر منطقه‌ای است؛ `grok-build-0.1` و هر دو
گونهٔ تاریخ‌دار Grok 4.20 قابل انتخاب باقی می‌مانند.
</Tip>

فرادادهٔ زمینهٔ کاتالوگ و هزینهٔ توکن از
[صفحه‌های مدل](https://docs.x.ai/developers/models) و
[صفحهٔ قیمت‌گذاری](https://docs.x.ai/developers/pricing) زندهٔ xAI پیروی می‌کند. هنگامی که
درخواستی از آستانهٔ مستندشدهٔ زمینهٔ طولانی عبور کند، xAI نرخ‌های بالاتری اعمال می‌کند؛
فیلدهای ثابت هزینهٔ کاتالوگ OpenClaw نرخ‌های زمینهٔ کوتاه را ثبت می‌کنند. Grok Build،
CLIِ عامل کدنویسی مجزای xAI، در [x.ai/cli](https://x.ai/cli) در دسترس است و در حال حاضر
از Grok 4.5 استفاده می‌کند.

## پوشش قابلیت‌ها

Plugin داخلی، APIهای پشتیبانی‌شدهٔ xAI را به قراردادهای مشترک ارائه‌دهنده و
ابزار OpenClaw نگاشت می‌کند. قابلیت‌هایی که با قرارداد مشترک سازگار نیستند
در ادامه یا زیر محدودیت‌های شناخته‌شده فهرست شده‌اند.

| قابلیت xAI             | سطح OpenClaw                        | وضعیت                                               |
| -------------------------- | --------------------------------------- | ---------------------------------------------------- |
| گفت‌وگو / Responses           | ارائه‌دهندهٔ مدل `xai/<model>`            | بله                                                  |
| جست‌وجوی وب سمت سرور     | ارائه‌دهندهٔ `web_search` با `grok`            | بله                                                  |
| جست‌وجوی X سمت سرور       | ابزار `x_search`                         | بله                                                  |
| اجرای کد سمت سرور | ابزار `code_execution`                   | بله                                                  |
| تصاویر                     | `image_generate`                        | بله                                                  |
| ویدیوها                     | `video_generate`                        | بله                                                  |
| تبدیل دسته‌ای متن به گفتار       | `tts.provider: "xai"` / `tts`           | بله                                                  |
| TTS جریانی              | `textToSpeechStream`                    | بله، از طریق `wss://api.x.ai/v1/tts` (نه صدای بلادرنگ) |
| تبدیل دسته‌ای گفتار به متن       | درک رسانه‌ای `tools.media.audio` | بله                                                  |
| تبدیل جریانی گفتار به متن   | `streaming.provider: "xai"` تماس صوتی  | بله                                                  |
| صدای بلادرنگ             | `talk.realtime.provider: "xai"` گفت‌وگو    | بله؛ رلهٔ Gateway برای Nodeهای بومی Talk             |
| فایل‌ها / دسته‌ها            | فقط سازگاری عمومی API مدل    | ابزار درجه‌یک OpenClaw نیست                      |

<Note>
OpenClaw برای تولید رسانه و رونویسی دسته‌ای از APIهای RESTِ تصویر/ویدیو/TTS/STTِ xAI،
برای رونویسی زندهٔ تماس صوتی از WebSocket جریانی STTِ xAI،
برای نشست‌های بلادرنگ Talk از WebSocket عامل صوتی Grokِ xAI
و برای گفت‌وگو، جست‌وجو و ابزارهای اجرای کد از APIِ Responses استفاده می‌کند.
</Note>

### سازگاری قدیمی حالت سریع

`/fast on` یا `agents.defaults.models["xai/<model>"].params.fastMode: true`
همچنان پیکربندی‌های قدیمی xAI را به‌شکل زیر بازنویسی می‌کند. این شناسه‌های مقصد
فقط برای سازگاری نگه‌داری می‌شوند؛ برای پیکربندی‌های جدید از مدل‌های قابل انتخاب
فعلی استفاده کنید.

| مدل مبدأ  | مقصد حالت سریع   |
| ------------- | ------------------ |
| `grok-3`      | `grok-3-fast`      |
| `grok-3-mini` | `grok-3-mini-fast` |
| `grok-4`      | `grok-4-fast`      |
| `grok-4-0709` | `grok-4-fast`      |

### سازگاری قدیمی و نام‌های مستعار متغیر

نام‌های مستعار قدیمی به‌شکل زیر نرمال‌سازی می‌شوند:

| نام مستعار قدیمی                                                  | شناسهٔ نرمال‌شده    |
| ------------------------------------------------------------- | ---------------- |
| `grok-code-fast-1`، `grok-code-fast`، `grok-code-fast-1-0825` | `grok-build-0.1` |

شناسه‌های تاریخ‌دار 0309 ورودی‌های قابل انتخاب کاتالوگ هستند. OpenClaw همهٔ نام‌های
مستعار فعلی دیگر Grok 4.20 را بدون تغییر ارسال می‌کند تا xAI کنترل معنای نام‌های
مستعار پایدار، latest، beta، experimental و تاریخ‌دار را حفظ کند. نام مستعار سراسری
`grok-latest` نیز بدون تغییر حفظ می‌شود.

xAI شناسه‌های دقیق زیر را بازنشسته کرده است. OpenClaw آن‌ها را به‌صورت ردیف‌های
سازگاری پنهان برای پیکربندی‌های منتشرشده، با محدودیت‌ها و قیمت‌گذاری مقصدهای
تغییرمسیر فعلی‌شان، نگه می‌دارد:

| شناسه‌های بازنشسته                                                          | رفتار فعلی                 |
| -------------------------------------------------------------------- | -------------------------------- |
| `grok-4-1-fast-reasoning`، `grok-4-fast-reasoning`، `grok-4-0709`    | Grok 4.3 با استدلال `low`    |
| `grok-4-1-fast-non-reasoning`، `grok-4-fast-non-reasoning`، `grok-3` | Grok 4.3 با استدلال غیرفعال |
| `grok-code-fast-1`                                                   | Grok Build 0.1                   |
| `grok-imagine-image-pro`                                             | کیفیت تصویر Grok Imagine       |

`openclaw doctor --fix` پیش‌فرض‌های ماندگار ابزارهای سرور xAI و
نامک بازنشستهٔ تصویر باکیفیت را به‌روزرسانی می‌کند، ردیف‌های قدیمی تولیدشدهٔ
کاتالوگ را حذف می‌کند و فرادادهٔ قدیمی زمینه را در ردیف‌های فعال 4.20 ترمیم
می‌کند. این کار نام‌های مستعار فعال 4.20 با `beta-latest` را به یک تصویر
لحظه‌ای تاریخ‌دار مقید نمی‌کند.

## قابلیت‌ها

<Warning>
  `x_search` و `code_execution` روی سرورهای xAI اجرا می‌شوند. xAI برای هر 1,000
  فراخوانی ابزار $5، به‌علاوهٔ توکن‌های ورودی و خروجی مدل، هزینه دریافت می‌کند. اگر
  تنظیم `enabled` هر ابزار مشخص نشده باشد، OpenClaw آن را فقط برای یک مدل فعال xAI ارائه می‌کند.
  یک ارائه‌دهندهٔ مدل شناخته‌شدهٔ غیر xAI به `enabled: true` صریح برای هر ابزار نیاز دارد؛
  ارائه‌دهندهٔ مفقود یا تفکیک‌نشده به‌صورت بسته شکست می‌خورد. احراز هویت xAI همیشه الزامی است
  و `enabled: false` ابزار را برای همهٔ ارائه‌دهندگان غیرفعال می‌کند.
</Warning>

<AccordionGroup>
  <Accordion title="جست‌وجوی وب">
    ارائه‌دهندهٔ داخلی جست‌وجوی وب `grok`، OAuthِ xAI را ترجیح می‌دهد و سپس
    به `XAI_API_KEY` یا کلید جست‌وجوی وب Plugin بازمی‌گردد:

    ```bash
    openclaw models auth login --provider xai --method oauth
    openclaw config set tools.web.search.provider grok
    ```

  </Accordion>

  <Accordion title="تولید ویدیو">
    Plugin داخلی `xai` تولید ویدیو را از طریق ابزار مشترک
    `video_generate` ثبت می‌کند.

    - مدل پیش‌فرض: `xai/grok-imagine-video`
    - مدل اضافی: `xai/grok-imagine-video-1.5`
    - حالت‌های کلاسیک: متن به ویدیو، تصویر به ویدیو، تولید با تصویر مرجع،
      ویرایش ویدیوی راه دور و گسترش ویدیوی راه دور
    - حالت Video 1.5: فقط تصویر به ویدیو، با دقیقاً یک تصویر فریم نخست
    - نسبت‌های ابعاد: `1:1`، `16:9`، `9:16`، `4:3`، `3:4`، `3:2`، `2:3`؛
      در صورت حذف، حالت کلاسیک و تصویر به ویدیوی Video 1.5 نسبت تصویر منبع را
      به ارث می‌برند
    - وضوح‌ها: حالت کلاسیک `480P`/`720P`؛ Video 1.5 از `1080P` نیز پشتیبانی می‌کند؛
      پیش‌فرض همهٔ حالت‌های تولید `480P` است
    - مدت: 1-15 ثانیه برای تولید/تصویر به ویدیو، 1-10 ثانیه هنگام
      استفاده از نقش‌های کلاسیک `reference_image`، و 2-10 ثانیه برای گسترش کلاسیک
    - تولید با تصویر مرجع: برای هر تصویر ارائه‌شده `imageRoles` را روی
      `reference_image` تنظیم کنید؛ xAI حداکثر 7 تصویر از این نوع می‌پذیرد
    - ویرایش/گسترش ویدیو نسبت ابعاد و وضوح ویدیوی ورودی را به ارث می‌برد؛
      این عملیات بازنویسی هندسه را نمی‌پذیرند
    - مهلت پیش‌فرض عملیات: 600 ثانیه، مگر اینکه `video_generate.timeoutMs`
      یا `agents.defaults.mediaModels.video.timeoutMs` تنظیم شده باشد

    <Warning>
    بافرهای ویدیویی محلی پذیرفته نمی‌شوند. برای ورودی ویرایش/گسترش ویدیو از URLهای
    راه دور `http(s)` استفاده کنید. تصویر به ویدیو بافرهای تصویر محلی را
    می‌پذیرد، زیرا OpenClaw آن‌ها را برای xAI به‌صورت URL داده کدگذاری می‌کند.
    </Warning>

    Video 1.5 همچنین شناسه‌های `grok-imagine-video-1.5-preview` و
    `grok-imagine-video-1.5-2026-05-30` متعلق به xAI را تشخیص می‌دهد. OpenClaw شناسهٔ
    انتخاب‌شده را بدون تغییر ارسال می‌کند، اما همان اعتبارسنجی مختص تصویر را اعمال می‌کند.

    برای استفاده از xAI به‌عنوان ارائه‌دهندهٔ پیش‌فرض ویدیو:

    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "xai/grok-imagine-video",
          },
        },
      },
    }
    ```

    <Note>
    برای پارامترهای مشترک ابزار، انتخاب ارائه‌دهنده و رفتار جایگزینی در صورت خرابی، به
    [تولید ویدئو](/fa/tools/video-generation) مراجعه کنید.
    </Note>

  </Accordion>

  <Accordion title="تولید تصویر">
    Plugin همراه `xai`، تولید تصویر را از طریق ابزار مشترک
    `image_generate` ثبت می‌کند.

    - مدل پیش‌فرض تصویر: `xai/grok-imagine-image`
    - مدل اضافی: `xai/grok-imagine-image-quality`
    - حالت‌ها: تبدیل متن به تصویر و ویرایش تصویر مرجع
    - ورودی‌های مرجع: یک `image` یا حداکثر سه `images`
    - نسبت‌های ابعاد: `1:1`، `16:9`، `9:16`، `4:3`، `3:4`، `3:2`، `2:3`، `2:1`،
      `1:2`، `19.5:9`، `9:19.5`، `20:9`، `9:20`
    - وضوح‌ها: `1K`، `2K`
    - تعداد: حداکثر 4 تصویر
    - مهلت پیش‌فرض عملیات: 600 ثانیه، مگر اینکه `image_generate.timeoutMs`
      یا `agents.defaults.mediaModels.image.timeoutMs` تنظیم شده باشد

    OpenClaw از xAI پاسخ‌های تصویری `b64_json` را درخواست می‌کند تا رسانه تولیدشده
    از طریق مسیر عادی پیوست کانال ذخیره و تحویل داده شود. تصاویر مرجع
    محلی به URLهای داده تبدیل می‌شوند؛ ارجاع‌های راه دور `http(s)`
    بدون تغییر عبور داده می‌شوند.

    برای استفاده از xAI به‌عنوان ارائه‌دهنده پیش‌فرض تصویر:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "xai/grok-imagine-image",
          },
        },
      },
    }
    ```

    <Note>
    xAI همچنین `quality`، `mask`، `user` و نسبت ابعاد `auto` را مستند کرده است.
    OpenClaw در حال حاضر فقط کنترل‌های مشترک تصویر میان ارائه‌دهندگان را ارسال می‌کند؛
    این گزینه‌های مختص ارائه‌دهنده از طریق `image_generate` در دسترس نیستند.
    </Note>

  </Accordion>

  <Accordion title="تبدیل متن به گفتار">
    Plugin همراه `xai`، تبدیل متن به گفتار را از طریق سطح مشترک ارائه‌دهنده
    `tts` ثبت می‌کند.

    - صداها: کاتالوگ زنده و احراز هویت‌شده از xAI؛ آن را با
      `openclaw infer tts voices --provider xai` فهرست کنید
    - صداهای جایگزین آفلاین: `ara`، `eve`، `leo`، `rex`، `sal`
    - صدای پیش‌فرض: `eve`
    - شناسه‌های صدای سفارشی حساب، حتی در صورت نبودن در پاسخ کاتالوگ
      داخلی، ارسال می‌شوند
    - قالب‌ها: `mp3`، `wav`، `pcm`، `mulaw`، `alaw`
    - زبان: کد BCP-47 یا `auto`
    - سرعت: بازنویسی سرعت بومی ارائه‌دهنده
    - قالب بومی یادداشت صوتی Opus پشتیبانی نمی‌شود

    برای استفاده از xAI به‌عنوان ارائه‌دهنده پیش‌فرض TTS:

    ```json5
    {
      tts: {
        provider: "xai",
        providers: {
          xai: {
            voiceId: "eve",
          },
        },
      },
    }
    ```

    <Note>
    OpenClaw برای سنتز بافرشده از نقطه پایانی دسته‌ای `/v1/tts` متعلق به xAI،
    برای کشف کاتالوگ احراز هویت‌شده از `/v1/tts/voices` و برای سنتز جریانی از
    `wss://api.x.ai/v1/tts` بومی استفاده می‌کند. جریان به میزبان بومی
    `api.x.ai` محدود است؛ بنابراین مقادیر سفارشی `baseUrl` در این
    مسیر رد می‌شوند. این مسیر از کنترل‌های موجود زبان، صدا، کدک و سرعت استفاده می‌کند؛
    پیش‌فرض‌های xAI برای نرخ نمونه‌برداری و نرخ بیت اعمال می‌شوند. سنتز فایل صوتی از همه
    کدک‌های پیکربندی‌شده پیروی می‌کند. مقصدهای یادداشت صوتی برای جریان و حالت جایگزین
    بافرشده از MP3 استفاده می‌کنند، زیرا کدک‌های خام xAI فراداده کدک/نرخ را حمل نمی‌کنند.
    جریان ابتدا `text.delta` و سپس
    `text.done` را ارسال می‌کند، `audio.delta`، `audio.done` یا `error` را دریافت می‌کند و
    یک `timeoutMs` بی‌کاری اعمال می‌کند که با هر قطعه صوتی تازه می‌شود. این قابلیت از
    نشست‌های صوتی بلادرنگ جدا است. قرارداد [API جریانی TTS](https://docs.x.ai/developers/rest-api-reference/inference/voice) متعلق به xAI را ببینید.
    </Note>

  </Accordion>

  <Accordion title="تبدیل گفتار به متن">
    Plugin همراه `xai`، تبدیل دسته‌ای گفتار به متن را از طریق سطح رونویسی
    درک رسانه OpenClaw ثبت می‌کند.

    - نقطه پایانی: REST متعلق به xAI، `/v1/stt`
    - مسیر ورودی: بارگذاری فایل صوتی چندبخشی
    - انتخاب مدل: xAI مدل رونویسی را به‌صورت داخلی انتخاب می‌کند؛ این
      نقطه پایانی انتخابگر مدل ندارد
    - در هر جایی استفاده می‌شود که رونویسی صوت ورودی `tools.media.audio` را می‌خواند،
      از جمله بخش‌های کانال صوتی Discord و پیوست‌های صوتی کانال

    برای اجبار استفاده از xAI جهت رونویسی صوت ورودی:

    ```json5
    {
      tools: {
        media: {
          audio: {
            models: [
              {
                type: "provider",
                provider: "xai",
              },
            ],
          },
        },
      },
    }
    ```

    زبان را می‌توان از طریق پیکربندی مشترک رسانه صوتی یا درخواست رونویسی
    هر فراخوانی ارائه کرد. سطح مشترک OpenClaw راهنمایی‌های اعلان را می‌پذیرد،
    اما یکپارچه‌سازی REST STT متعلق به xAI فقط فایل و زبان را ارسال می‌کند،
    زیرا تنها این موارد با نقطه پایانی عمومی فعلی xAI مطابقت دارند.

  </Accordion>

  <Accordion title="تبدیل جریانی گفتار به متن">
    Plugin همراه `xai` همچنین یک ارائه‌دهنده رونویسی بلادرنگ
    برای صدای زنده تماس صوتی ثبت می‌کند.

    - نقطه پایانی: WebSocket متعلق به xAI، `wss://api.x.ai/v1/stt`
    - کدگذاری پیش‌فرض: `mulaw`
    - نرخ نمونه‌برداری پیش‌فرض: `8000`
    - نقطه‌گذاری پایانی پیش‌فرض: `800ms`
    - رونویسی‌های موقت: به‌طور پیش‌فرض فعال

    جریان رسانه Twilio متعلق به Voice Call، فریم‌های صوتی G.711 mu-law را ارسال می‌کند؛
    بنابراین ارائه‌دهنده xAI این فریم‌ها را مستقیماً و بدون تبدیل کدک ارسال می‌کند:

    ```json5
    {
      plugins: {
        entries: {
          "voice-call": {
            config: {
              streaming: {
                enabled: true,
                provider: "xai",
                providers: {
                  xai: {
                    apiKey: "${XAI_API_KEY}",
                    endpointingMs: 800,
                    language: "en",
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

    پیکربندی متعلق به ارائه‌دهنده در
    `plugins.entries.voice-call.config.streaming.providers.xai` قرار دارد. کلیدهای
    پشتیبانی‌شده عبارت‌اند از `apiKey`، `baseUrl`، `sampleRate`، `encoding` ‏(`pcm`، `mulaw` یا
    `alaw`)، `interimResults`، `endpointingMs` و `language`.

    <Note>
    این ارائه‌دهنده جریانی برای مسیر رونویسی بلادرنگ Voice Call است.
    صدای Discord بخش‌های کوتاه را ضبط می‌کند و در عوض از مسیر رونویسی دسته‌ای
    `tools.media.audio` استفاده می‌کند.
    </Note>

  </Accordion>

  <Accordion title="صدای بلادرنگ (Talk)">
    Plugin همراه `xai`، نشست‌های بلادرنگ Grok Voice Agent را برای
    حالت Talk از طریق قرارداد مشترک `registerRealtimeVoiceProvider` ثبت می‌کند.

    - نقطه پایانی: `wss://api.x.ai/v1/realtime?model=<voice-model>`
    - مدل پیش‌فرض: `grok-voice-latest`
    - صدای پیش‌فرض: `eve`
    - انتقال: `gateway-relay` (مسیرهای رله iOS، Android و Control UI)
    - صوت: PCM16 با 24 kHz یا G.711 µ-law با 8 kHz
    - مداخله در حین گفتار: VAD سرور xAI پاسخ را قطع می‌کند؛ OpenClaw پخش صف‌شده را پاک
      و تاریخچه پخش‌نشده ارائه‌دهنده را کوتاه می‌کند

    Talk را در Gateway پیکربندی کنید:

    ```json5
    {
      talk: {
        realtime: {
          provider: "xai",
          mode: "realtime",
          transport: "gateway-relay",
          brain: "agent-consult",
          providers: {
            xai: {
              model: "grok-voice-latest",
              voice: "eve",
              // فقط در صورتی فعال کنید که بازپخش نشست در سمت ارائه‌دهنده پذیرفتنی باشد.
              sessionResumption: false,
            },
          },
        },
      },
      env: { XAI_API_KEY: "xai-..." },
    }
    ```

    پیکربندی متعلق به ارائه‌دهنده همچنین از
    `plugins.entries.voice-call.config.realtime.providers.xai` تفکیک می‌شود، هنگامی که Voice Call
    یا انتخابگرهای مشترک بلادرنگ از همان نگاشت ارائه‌دهنده استفاده مجدد می‌کنند. کلیدهای پشتیبانی‌شده
    عبارت‌اند از `apiKey`، `baseUrl`، `model`، `voice`، `vadThreshold`، `silenceDurationMs`،
    `prefixPaddingMs`، `reasoningEffort` و `sessionResumption`.
    `reasoningEffort` فقط `high` یا `none` را می‌پذیرد که با API عامل صوتی xAI مطابقت دارد.

    VAD سرور xAI همیشه پاسخ‌ها را ایجاد و وقفه صوتی را مدیریت می‌کند.
    از `consultRouting: "provider-direct"` استفاده کنید؛ مسیریابی اجباری رونویسی و غیرفعال‌کردن
    وقفه صوت ورودی در پروتکل عامل صوتی xAI پشتیبانی نمی‌شوند.

    <Note>
    OAuth متعلق به xAI یا `XAI_API_KEY` می‌تواند صدای بلادرنگ را احراز هویت کند. WebRTC
    تحت مالکیت مرورگر هنوز بخشی از این سطح ارائه‌دهنده نیست؛ از Talk با gateway-relay روی
    Nodeهای بومی یا مسیر رله Control UI استفاده کنید.
    </Note>

    <Note>
    مقدار پیش‌فرض `sessionResumption` برابر `false` است. وقتی روی `true` تنظیم شود، OpenClaw از
    xAI می‌خواهد وضعیت کافی از نشست را نگه دارد تا پس از اتصال مجدد همان گفتگو از سر گرفته شود
    و سپس با شناسه گفتگوی بازگردانده‌شده دوباره متصل می‌شود. اگر بازپخش/نگه‌داری
    در سمت ارائه‌دهنده پذیرفتنی نیست، آن را غیرفعال نگه دارید؛ در این صورت سوکت‌های
    قطع‌شده به‌شکل بسته و امن شکست می‌خورند، به‌جای اینکه بی‌سروصدا گفتگوی تازه‌ای را آغاز کنند.
    </Note>

  </Accordion>

  <Accordion title="پیکربندی x_search">
    Plugin همراه xAI، `x_search` را به‌عنوان ابزار OpenClaw برای
    جست‌وجوی محتوای X (که پیش‌تر Twitter نام داشت) از طریق Grok ارائه می‌کند.

    مسیر پیکربندی: `plugins.entries.xai.config.xSearch`

    | کلید               | نوع    | پیش‌فرض                   | توضیحات                                      |
    | ----------------- | ------- | ------------------------- | ------------------------------------------------ |
    | `enabled`         | بولی | خودکار برای مدل‌های xAI  | غیرفعال کنید، یا برای یک ارائه‌دهنده شناخته‌شده غیر xAI فعال کنید |
    | `model`           | رشته  | `grok-4.3`                | مدل مورداستفاده برای درخواست‌های x_search                 |
    | `baseUrl`         | رشته  | -                         | بازنویسی URL پایه Responses متعلق به xAI                  |
    | `inlineCitations` | بولی | -                         | گنجاندن ارجاع‌های درون‌خطی در نتایج              |
    | `maxTurns`        | عدد  | -                         | حداکثر نوبت‌های گفتگو                       |
    | `timeoutSeconds`  | عدد  | `30`                      | مهلت درخواست بر حسب ثانیه                       |
    | `cacheTtlMinutes` | عدد  | `15`                      | زمان ماندگاری حافظه نهان بر حسب دقیقه                    |

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              xSearch: {
                enabled: true,
                model: "grok-4.3",
                baseUrl: "https://api.x.ai/v1",
                inlineCitations: true,
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="پیکربندی اجرای کد">
    Plugin همراه xAI، `code_execution` را به‌عنوان ابزار OpenClaw برای
    اجرای کد از راه دور در محیط سندباکس xAI ارائه می‌کند.

    مسیر پیکربندی: `plugins.entries.xai.config.codeExecution`

    | کلید              | نوع    | پیش‌فرض                  | توضیحات                                      |
    | ---------------- | ------- | ------------------------ | ------------------------------------------------ |
    | `enabled`        | بولی | خودکار برای مدل‌های xAI | غیرفعال کنید، یا برای یک ارائه‌دهنده شناخته‌شده غیر xAI فعال کنید |
    | `model`          | رشته  | `grok-4.3`               | مدل مورداستفاده برای درخواست‌های اجرای کد           |
    | `maxTurns`       | عدد  | -                        | حداکثر نوبت‌های مکالمه                       |
    | `timeoutSeconds` | عدد  | `30`                     | مهلت زمانی درخواست برحسب ثانیه                       |

    <Note>
    این اجرای راه دور در سندباکس xAI است، نه [`exec`](/fa/tools/exec) محلی.
    </Note>

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              codeExecution: {
                enabled: true,
                model: "grok-4.3",
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="محدودیت‌های شناخته‌شده">
    - احراز هویت xAI می‌تواند از کلید API، متغیر محیطی، پیکربندی Plugin به‌عنوان
      راهکار جایگزین، یا OAuth با یک حساب واجد شرایط xAI استفاده کند. OAuth بدون
      فراخوانی برگشتی localhost از تأیید با کد دستگاه استفاده می‌کند. xAI تعیین می‌کند
      کدام حساب‌ها می‌توانند توکن‌های API مبتنی بر OAuth دریافت کنند و ممکن است صفحه
      رضایت، Grok Build را نمایش دهد، هرچند OpenClaw به برنامه Grok Build نیاز ندارد.
    - OpenClaw درحال‌حاضر خانواده مدل چندعاملی xAI را ارائه نمی‌کند. xAI
      این مدل‌ها را از طریق Responses API ارائه می‌دهد، اما آن‌ها ابزارهای سمت
      کلاینت یا سفارشی مورداستفاده در حلقه عامل مشترک OpenClaw را نمی‌پذیرند.
      به
      [محدودیت‌های چندعاملی xAI](https://docs.x.ai/developers/model-capabilities/text/multi-agent#limitations)
      مراجعه کنید.
    - صدای بلادرنگ xAI درحال‌حاضر فقط انتقال Talk از طریق رله Gateway را ارائه می‌دهد.
      نشست‌های WebSocket ارائه‌دهنده که تحت مالکیت مرورگر هستند، هنوز در Control UI
      متصل نشده‌اند.
    - `quality` تصویر xAI، `mask` تصویر و نسبت‌های ابعاد اضافی مختص
      حالت بومی تا زمانی که ابزار مشترک `image_generate` کنترل‌های متناظر
      میان‌ارائه‌دهنده‌ای نداشته باشد، ارائه نمی‌شوند.
  </Accordion>

  <Accordion title="نکات پیشرفته">
    - OpenClaw اصلاحات سازگاری ویژه xAI برای شِمای ابزار و فراخوانی ابزار را
      به‌طور خودکار در مسیر اجراکننده مشترک اعمال می‌کند.
    - درخواست‌های بومی xAI به‌طور پیش‌فرض `tool_stream: true` هستند. برای
      غیرفعال‌کردن آن، `agents.defaults.models["xai/<model>"].params.tool_stream` را روی `false`
      تنظیم کنید.
    - پوشش‌دهنده همراه xAI پیش از ارسال درخواست‌های بومی xAI، کران‌های
      پشتیبانی‌نشده شمارش contains و کلیدهای پشتیبانی‌نشده *میزان* استدلال را از
      بار داده شِما حذف می‌کند. Grok 4.5 از میزان کم، متوسط و زیاد
      (پیش‌فرض زیاد) پشتیبانی می‌کند. Grok 4.3 از میزان بدون تلاش، کم، متوسط و زیاد
      (پیش‌فرض کم) پشتیبانی می‌کند. سایر مدل‌های xAI با قابلیت استدلال، کنترل میزان
      قابل‌تنظیمی ارائه نمی‌کنند، اما همچنان `include: ["reasoning.encrypted_content"]` را
      درخواست می‌کنند تا استدلال رمزگذاری‌شده قبلی در نوبت‌های بعدی بازپخش شود.
    - `web_search`، `x_search` و `code_execution` به‌عنوان ابزارهای OpenClaw
      ارائه می‌شوند. OpenClaw فقط قابلیت داخلی مشخص xAI را که هر ابزار نیاز دارد
      به درخواست همان ابزار متصل می‌کند، به‌جای آنکه همه ابزارهای بومی را به هر
      نوبت چت متصل کند.
    - Grok `web_search`، `plugins.entries.xai.config.webSearch.baseUrl` را می‌خواند.
      `x_search`، `plugins.entries.xai.config.xSearch.baseUrl` را می‌خواند و سپس
      به URL پایه جست‌وجوی وب Grok بازمی‌گردد.
    - `x_search` و `code_execution` به‌جای آنکه در زمان اجرای مدل هسته
      به‌صورت ثابت کدنویسی شوند، تحت مالکیت Plugin همراه xAI هستند.
    - `code_execution` اجرای راه دور در سندباکس xAI است، نه
      [`exec`](/fa/tools/exec) محلی.
  </Accordion>
</AccordionGroup>

## آزمایش زنده

مسیرهای رسانه‌ای xAI با آزمون‌های واحد و مجموعه‌آزمون‌های زنده اختیاری پوشش داده
شده‌اند. پیش از اجرای بررسی‌های زنده، `XAI_API_KEY` را در محیط فرایند صادر کنید.

```bash
pnpm test extensions/xai
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 pnpm test:live -- extensions/xai/xai.live.test.ts
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "classic Grok Imagine"
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_XAI_VIDEO=1 pnpm test:live -- extensions/xai/xai.live.test.ts -t "Grok Imagine Video 1.5"
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 pnpm test:live -- extensions/xai/x-search.live.test.ts
OPENCLAW_LIVE_GATEWAY_MODELS="xai/grok-4.5,xai/grok-build-0.1,xai/grok-4.3,xai/grok-4.20-0309-reasoning,xai/grok-4.20-0309-non-reasoning" OPENCLAW_LIVE_GATEWAY_MAX_MODELS=0 OPENCLAW_LIVE_GATEWAY_SMOKE=0 pnpm test:live -- src/gateway/gateway-models.profiles.live.test.ts
OPENCLAW_LIVE_TEST=1 OPENCLAW_LIVE_TEST_QUIET=1 OPENCLAW_LIVE_IMAGE_GENERATION_PROVIDERS=xai pnpm test:live -- test/image-generation.runtime.live.test.ts
```

فایل زنده ویژه ارائه‌دهنده، TTS عادی و TTS مبتنی بر PCM مناسب تلفن را تولید می‌کند،
صدا را از طریق STT دسته‌ای xAI رونویسی می‌کند، همان PCM را از طریق STT بلادرنگ xAI
جریان می‌دهد، خروجی متن‌به‌تصویر تولید می‌کند و یک تصویر مرجع را ویرایش می‌کند.
فایل زنده مشترک تصویر، همان ارائه‌دهنده xAI را از طریق انتخاب زمان اجرای OpenClaw،
راهکار جایگزین، نرمال‌سازی و مسیر پیوست رسانه‌ای تأیید می‌کند. مورد اختیاری Video 1.5
یک تصویر تولیدشده برای فریم نخست را با وضوح 1080P ارسال و بارگیری ویدیوی تکمیل‌شده را
تأیید می‌کند.

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="تولید ویدیو" href="/fa/tools/video-generation" icon="video">
    پارامترهای ابزار مشترک ویدیو و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="همه ارائه‌دهندگان" href="/fa/providers/index" icon="grid-2">
    نمای کلی گسترده‌تر ارائه‌دهندگان.
  </Card>
  <Card title="عیب‌یابی" href="/fa/help/troubleshooting" icon="wrench">
    مشکلات رایج و راه‌حل‌ها.
  </Card>
</CardGroup>
