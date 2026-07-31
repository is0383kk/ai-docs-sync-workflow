---
read_when:
    - مدل‌های MiniMax را در OpenClaw می‌خواهید
    - به راهنمای راه‌اندازی MiniMax نیاز دارید
summary: استفاده از مدل‌های MiniMax در OpenClaw
title: MiniMax
x-i18n:
    generated_at: "2026-07-27T15:41:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6d3e95cf9836fd0bc30ac91649422a1d0ed8e7b2908a42e241106c1ea783cbbc
    source_path: providers/minimax.md
    workflow: 16
---

Plugin همراه `minimax` دو ارائه‌دهنده و پنج قابلیت را ثبت می‌کند: چت، تولید تصویر، تولید موسیقی، تولید ویدئو، درک تصویر، گفتار (T2A v2) و جست‌وجوی وب.

| شناسه ارائه‌دهنده      | احراز هویت    | قابلیت‌ها                                                                                        |
| ---------------- | ------- | --------------------------------------------------------------------------------------------------- |
| `minimax`        | کلید API | متن، تولید تصویر، تولید موسیقی، تولید ویدئو، درک تصویر، گفتار، جست‌وجوی وب |
| `minimax-portal` | OAuth   | متن، تولید تصویر، تولید موسیقی، تولید ویدئو، درک تصویر، گفتار             |

<Tip>
پیوند ارجاع برای MiniMax Coding Plan (با 10% تخفیف): [MiniMax Coding Plan](https://platform.minimax.io/subscribe/coding-plan?code=DbXJTRClnb&source=link)
</Tip>

## کاتالوگ داخلی

| مدل                    | نوع             | توضیحات                              |
| ------------------------ | ---------------- | ---------------------------------------- |
| `MiniMax-M3`             | چت (استدلالی) | مدل استدلالی میزبانی‌شده پیش‌فرض           |
| `MiniMax-M2.7`           | چت (استدلالی) | مدل استدلالی میزبانی‌شده قبلی          |
| `MiniMax-M2.7-highspeed` | چت (استدلالی) | سطح استدلالی سریع‌تر M2.7               |
| `MiniMax-VL-01`          | بینایی           | مدل درک تصویر                |
| `image-01`               | تولید تصویر | تبدیل متن به تصویر و ویرایش تصویر به تصویر |
| `music-2.6`              | تولید موسیقی | مدل موسیقی پیش‌فرض                      |
| `MiniMax-Hailuo-2.3`     | تولید ویدئو | جریان‌های متن به ویدئو و تصویر به ویدئو   |

ارجاع‌های مدل از مسیر احراز هویت پیروی می‌کنند: `minimax/<model>` برای راه‌اندازی‌های مبتنی بر کلید API و `minimax-portal/<model>` برای راه‌اندازی‌های OAuth.

## شروع به کار

<Tabs>
  <Tab title="OAuth (Coding Plan)">
    **مناسب برای:** راه‌اندازی سریع MiniMax Coding Plan از طریق OAuth، بدون نیاز به کلید API.

    <Tabs>
      <Tab title="بین‌المللی">
        <Steps>
          <Step title="اجرای راه‌اندازی اولیه">
            ```bash
            openclaw onboard --auth-choice minimax-global-oauth
            ```

            نشانی URL پایه ارائه‌دهنده حاصل: `api.minimax.io`.
          </Step>
          <Step title="بررسی دردسترس‌بودن مدل">
            ```bash
            openclaw models list --provider minimax-portal
            ```
          </Step>
        </Steps>
      </Tab>
      <Tab title="چین">
        <Steps>
          <Step title="اجرای راه‌اندازی اولیه">
            ```bash
            openclaw onboard --auth-choice minimax-cn-oauth
            ```

            نشانی URL پایه ارائه‌دهنده حاصل: `api.minimaxi.com`.
          </Step>
          <Step title="بررسی دردسترس‌بودن مدل">
            ```bash
            openclaw models list --provider minimax-portal
            ```
          </Step>
        </Steps>
      </Tab>
    </Tabs>

    <Note>
    راه‌اندازی‌های OAuth از شناسه ارائه‌دهنده `minimax-portal` استفاده می‌کنند. ارجاع‌های مدل از قالب `minimax-portal/MiniMax-M3` پیروی می‌کنند.
    </Note>

  </Tab>

  <Tab title="کلید API">
    **مناسب برای:** MiniMax میزبانی‌شده با API سازگار با Anthropic.

    <Tabs>
      <Tab title="بین‌المللی">
        <Steps>
          <Step title="اجرای راه‌اندازی اولیه">
            ```bash
            openclaw onboard --auth-choice minimax-global-api
            ```

            این کار `api.minimax.io` را به‌عنوان URL پایه پیکربندی می‌کند.
          </Step>
          <Step title="بررسی دردسترس‌بودن مدل">
            ```bash
            openclaw models list --provider minimax
            ```
          </Step>
        </Steps>
      </Tab>
      <Tab title="چین">
        <Steps>
          <Step title="اجرای راه‌اندازی اولیه">
            ```bash
            openclaw onboard --auth-choice minimax-cn-api
            ```

            این کار `api.minimaxi.com` را به‌عنوان URL پایه پیکربندی می‌کند.
          </Step>
          <Step title="بررسی دردسترس‌بودن مدل">
            ```bash
            openclaw models list --provider minimax
            ```
          </Step>
        </Steps>
      </Tab>
    </Tabs>

    ### نمونه پیکربندی

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-..." },
      agents: { defaults: { model: { primary: "minimax/MiniMax-M3" } } },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.12, cacheWrite: 0 },
                contextWindow: 1000000,
                maxTokens: 131072,
              },
              {
                id: "MiniMax-M2.7",
                name: "MiniMax M2.7",
                reasoning: true,
                input: ["text"],
                cost: { input: 0.3, output: 1.2, cacheRead: 0.06, cacheWrite: 0.375 },
                contextWindow: 204800,
                maxTokens: 131072,
              },
              {
                id: "MiniMax-M2.7-highspeed",
                name: "MiniMax M2.7 Highspeed",
                reasoning: true,
                input: ["text"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.06, cacheWrite: 0.375 },
                contextWindow: 204800,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    <Warning>
    نقطه پایانی استریم سازگار با Anthropic در MiniMax-M2.x، به‌جای بلوک‌های تفکر بومی Anthropic، `reasoning_content` را در قطعه‌های دلتا به سبک OpenAI منتشر می‌کند؛ در نتیجه، اگر تفکر به‌طور ضمنی فعال باقی بماند، استدلال داخلی در خروجی قابل‌مشاهده افشا می‌شود. OpenClaw تفکر M2.x را به‌طور پیش‌فرض غیرفعال می‌کند، مگر اینکه خودتان صراحتاً `thinking` را تنظیم کنید. MiniMax-M3 (و M3.x سازگار با نسخه‌های آینده) مستثنا است: M3 بلوک‌های تفکر صحیح Anthropic را منتشر می‌کند و برای تولید محتوای قابل‌مشاهده به تفکر فعال نیاز دارد؛ بنابراین OpenClaw، ‏M3 را در مسیر تفکر تطبیقی ارائه‌دهنده نگه می‌دارد. بخش پیش‌فرض‌های تفکر را در قسمت پیکربندی پیشرفته در ادامه ببینید.
    </Warning>

    <Note>
    راه‌اندازی‌های مبتنی بر کلید API از شناسه ارائه‌دهنده `minimax` استفاده می‌کنند. ارجاع‌های مدل از قالب `minimax/MiniMax-M3` پیروی می‌کنند.
    </Note>

  </Tab>
</Tabs>

## پیکربندی از طریق `openclaw configure`

<Steps>
  <Step title="اجرای راهنمای گام‌به‌گام">
    ```bash
    openclaw configure
    ```
  </Step>
  <Step title="انتخاب Model/auth">
    **Model/auth** را از منو انتخاب کنید.
  </Step>
  <Step title="انتخاب یک گزینه احراز هویت MiniMax">
    | گزینه احراز هویت            | توضیحات                        |
    | ----------------------- | ----------------------------------- |
    | `minimax-global-oauth` | OAuth بین‌المللی (Coding Plan)  |
    | `minimax-cn-oauth`     | OAuth چین (Coding Plan)          |
    | `minimax-global-api`   | کلید API بین‌المللی              |
    | `minimax-cn-api`       | کلید API چین                      |
  </Step>
  <Step title="انتخاب مدل پیش‌فرض">
    هنگام نمایش درخواست، مدل پیش‌فرض را انتخاب کنید.
  </Step>
</Steps>

## قابلیت‌ها

### تولید تصویر

Plugin ‏MiniMax، مدل `image-01` را برای ابزار `image_generate` روی هر دو `minimax` و `minimax-portal` ثبت می‌کند و از همان `MINIMAX_API_KEY` یا احراز هویت OAuth مدل‌های متنی دوباره استفاده می‌کند.

- تولید متن به تصویر و ویرایش تصویر به تصویر (ارجاع سوژه)، هر دو با کنترل نسبت ابعاد
- حداکثر 9 تصویر خروجی در هر درخواست و 1 تصویر مرجع در هر درخواست ویرایش
- نسبت‌های ابعاد پشتیبانی‌شده: `1:1`، `16:9`، `4:3`، `3:2`، `2:3`، `3:4`، `9:16`، `21:9`

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: { primary: "minimax/image-01" },
    },
  },
}
```

تولید تصویر همیشه از نقطه پایانی اختصاصی تصویر MiniMax (`/v1/image_generation`) استفاده می‌کند و `models.providers.minimax.baseUrl` را نادیده می‌گیرد، زیرا آن فیلد URL پایه چت/سازگار با Anthropic را پیکربندی می‌کند. برای هدایت تولید تصویر از طریق نقطه پایانی چین، `MINIMAX_API_HOST=https://api.minimaxi.com` را تنظیم کنید؛ نقطه پایانی جهانی پیش‌فرض `https://api.minimax.io` است.

<Note>
برای پارامترهای مشترک ابزار، انتخاب ارائه‌دهنده و رفتار جایگزینی هنگام خرابی، به [تولید تصویر](/fa/tools/image-generation) مراجعه کنید.
</Note>

### تبدیل متن به گفتار

Plugin همراه `minimax`، ‏MiniMax T2A v2 را به‌عنوان ارائه‌دهنده گفتار برای `tts` ثبت می‌کند.

- مدل پیش‌فرض TTS: ‏`speech-2.8-hd`
- صدای پیش‌فرض: `English_expressive_narrator`
- شناسه‌های مدل همراه: `speech-2.8-hd`، `speech-2.8-turbo`، `speech-2.6-hd`، `speech-2.6-turbo`، `speech-02-hd`، `speech-02-turbo`، `speech-01-hd`، `speech-01-turbo`
- ترتیب تفکیک احراز هویت: ابتدا `tts.providers.minimax.apiKey`، سپس پروفایل‌های احراز هویت OAuth/توکن `minimax-portal`، سپس کلیدهای محیطی Token Plan ‏(`MINIMAX_OAUTH_TOKEN`، `MINIMAX_CODE_PLAN_KEY`، `MINIMAX_CODING_API_KEY`) و در پایان `MINIMAX_API_KEY`
- اگر هیچ میزبان TTS پیکربندی نشده باشد، OpenClaw از میزبان OAuth پیکربندی‌شده `minimax-portal` دوباره استفاده می‌کند و پسوندهای مسیر سازگار با Anthropic مانند `/anthropic` را حذف می‌کند
- پیوست‌های صوتی عادی به‌صورت MP3 باقی می‌مانند. مقصدهای یادداشت صوتی (Feishu، ‏Telegram و کانال‌های دیگری که پیوست سازگار با یادداشت صوتی درخواست می‌کنند) با `ffmpeg` از MP3 ‏MiniMax به Opus با نرخ 48kHz تبدیل می‌شوند، زیرا برای مثال API فایل Feishu/Lark برای پیام‌های صوتی بومی فقط `file_type: "opus"` را می‌پذیرد
- ‏MiniMax T2A مقادیر اعشاری `speed` و `vol` را می‌پذیرد، اما `pitch` به‌صورت عدد صحیح ارسال می‌شود؛ OpenClaw پیش از درخواست API بخش اعشاری مقادیر `pitch` را حذف می‌کند

| تنظیم                         | متغیر محیطی                | پیش‌فرض                       | توضیحات                      |
| ------------------------------- | ---------------------- | ----------------------------- | -------------------------------- |
| `tts.providers.minimax.baseUrl` | `MINIMAX_API_HOST`     | `https://api.minimax.io`      | میزبان API ‏MiniMax T2A.            |
| `tts.providers.minimax.model`   | `MINIMAX_TTS_MODEL`    | `speech-2.8-hd`               | شناسه مدل TTS.                    |
| `tts.providers.minimax.voiceId` | `MINIMAX_TTS_VOICE_ID` | `English_expressive_narrator` | شناسه صدای مورداستفاده برای خروجی گفتار. |
| `tts.providers.minimax.speed`   |                        | `1.0`                         | سرعت پخش، `0.5..2.0`.      |
| `tts.providers.minimax.vol`     |                        | `1.0`                         | حجم صدا، `(0, 10]`.               |
| `tts.providers.minimax.pitch`   |                        | `0`                           | تغییر زیر و بمی به‌صورت عدد صحیح، `-12..12`.  |

### تولید موسیقی

Plugin همراه MiniMax، تولید موسیقی را از طریق ابزار مشترک `music_generate` برای هر دو `minimax` و `minimax-portal` ثبت می‌کند.

- مدل موسیقی پیش‌فرض: `minimax/music-2.6` ‏(OAuth: ‏`minimax-portal/music-2.6`)
- همچنین از `music-2.6-free`، `music-cover` و `music-cover-free` پشتیبانی می‌کند
- کنترل‌های پرامپت: `lyrics`، `instrumental`
- قالب خروجی: `mp3`
- اجراهای مبتنی بر نشست از طریق جریان مشترک وظیفه/وضعیت، از جمله `action: "status"`، جدا می‌شوند

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: { primary: "minimax/music-2.6" },
    },
  },
}
```

<Note>
برای پارامترهای مشترک ابزار، انتخاب ارائه‌دهنده و رفتار جایگزینی هنگام خرابی، به [تولید موسیقی](/fa/tools/music-generation) مراجعه کنید.
</Note>

### تولید ویدئو

Plugin همراه MiniMax، تولید ویدئو را از طریق ابزار مشترک `video_generate` برای هر دو `minimax` و `minimax-portal` ثبت می‌کند.

- مدل پیش‌فرض ویدیو: `minimax/MiniMax-Hailuo-2.3` (OAuth: `minimax-portal/MiniMax-Hailuo-2.3`)
- همچنین از `MiniMax-Hailuo-2.3-Fast`، `MiniMax-Hailuo-02`، `I2V-01-Director`، `I2V-01-live` و `I2V-01` پشتیبانی می‌کند
- حالت‌ها: جریان‌های متن‌به‌ویدیو و ارجاع تک‌تصویری
- از `resolution` پشتیبانی می‌کند (`768P` یا `1080P` در مدل‌های Hailuo 2.3/02)؛ `aspectRatio` پشتیبانی نمی‌شود و نادیده گرفته می‌شود

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "minimax/MiniMax-Hailuo-2.3" },
    },
  },
}
```

<Note>
برای پارامترهای مشترک ابزار، انتخاب ارائه‌دهنده و رفتار جایگزینی هنگام خرابی، به [تولید ویدیو](/fa/tools/video-generation) مراجعه کنید.
</Note>

### درک تصویر

Plugin مربوط به MiniMax، درک تصویر را جدا از کاتالوگ متنی ثبت می‌کند:

| شناسه ارائه‌دهنده      | مدل پیش‌فرض تصویر | استخراج متن PDF |
| ---------------- | ------------------- | ------------------- |
| `minimax`        | `MiniMax-VL-01`     | `MiniMax-M2.7`      |
| `minimax-portal` | `MiniMax-VL-01`     | `MiniMax-M2.7`      |

به همین دلیل، مسیریابی خودکار رسانه می‌تواند از درک تصویر MiniMax استفاده کند، حتی زمانی که کاتالوگ همراهِ ارائه‌دهندگان متنی شامل ارجاع‌های چت M3 با قابلیت پردازش تصویر نیز باشد. درک PDF فقط برای استخراج متن از `MiniMax-M2.7` استفاده می‌کند؛ MiniMax هیچ مسیر تبدیل PDF به تصویر ثبت نمی‌کند.

### جست‌وجوی وب

Plugin مربوط به MiniMax همچنین `web_search` را از طریق API جست‌وجوی MiniMax Token Plan (`/v1/coding_plan/search`) ثبت می‌کند.

- شناسه ارائه‌دهنده: `minimax`
- نتایج ساختاریافته: عنوان‌ها، URLها، قطعه‌متن‌ها، پرس‌وجوهای مرتبط
- متغیر محیطی ترجیحی: `MINIMAX_CODE_PLAN_KEY`
- نام‌های مستعار پذیرفته‌شده برای متغیر محیطی: `MINIMAX_CODING_API_KEY`، `MINIMAX_OAUTH_TOKEN`
- جایگزین سازگاری: `MINIMAX_API_KEY`، هنگامی که از قبل به اعتبارنامه Token Plan اشاره می‌کند
- استفادهٔ مجدد از منطقه: ابتدا `plugins.entries.minimax.config.webSearch.region`، سپس `MINIMAX_API_HOST` و بعد URLهای پایهٔ ارائه‌دهنده MiniMax
- جست‌وجو روی شناسهٔ ارائه‌دهنده `minimax` باقی می‌ماند؛ راه‌اندازی OAuth چین/جهانی می‌تواند از طریق `models.providers.minimax-portal.baseUrl` به‌طور غیرمستقیم منطقه را هدایت کند و از طریق `MINIMAX_OAUTH_TOKEN` احراز هویت bearer را فراهم کند

پیکربندی در `plugins.entries.minimax.config.webSearch.*` قرار دارد.

<Note>
برای پیکربندی کامل جست‌وجوی وب و نحوهٔ استفاده، به [جست‌وجوی MiniMax](/fa/tools/minimax-search) مراجعه کنید.
</Note>

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="گزینه‌های پیکربندی">
    | گزینه | توضیحات |
    | --- | --- |
    | `models.providers.minimax.baseUrl` | `https://api.minimax.io/anthropic` (سازگار با Anthropic) را ترجیح دهید؛ `https://api.minimax.io/v1` برای payloadهای سازگار با OpenAI اختیاری است |
    | `models.providers.minimax.api` | `anthropic-messages` را ترجیح دهید؛ `openai-completions` برای payloadهای سازگار با OpenAI اختیاری است |
    | `models.providers.minimax.apiKey` | کلید API مربوط به MiniMax (`MINIMAX_API_KEY`) |
    | `models.providers.minimax.models` | `id`، `name`، `reasoning`، `contextWindow`، `maxTokens` و `cost` را تعریف کنید |
    | `agents.defaults.models` | نام‌های مستعار، پارامترها و فراداده‌های مختص هر مدل |
    | `agents.defaults.modelPolicy.allow` | فهرست مجاز صریح و اختیاری مدل‌ها |
    | `models.mode` | اگر می‌خواهید MiniMax را در کنار موارد داخلی اضافه کنید، `merge` را حفظ کنید |
  </Accordion>

  <Accordion title="پیش‌فرض‌های تفکر">
    در `api: "anthropic-messages"`، OpenClaw برای مدل‌های MiniMax M2.x مقدار `thinking: { type: "disabled" }` را تزریق می‌کند، مگر اینکه یک wrapper قبلی از پیش فیلد `thinking` را در payload تنظیم کرده باشد. این کار مانع می‌شود endpoint استریم M2.x مقدار `reasoning_content` را در قطعه‌های delta به سبک OpenAI منتشر کند؛ رخدادی که استدلال داخلی را در خروجی قابل‌مشاهده افشا می‌کرد.

    MiniMax-M3 (و M3.x) مستثنا است: وقتی تفکر غیرفعال باشد، M3 یک آرایهٔ خالی `content` همراه با `stop_reason: "end_turn"` برمی‌گرداند؛ بنابراین OpenClaw پیش‌فرض ضمنیِ غیرفعال را برای M3 حذف می‌کند و هنگامی که سطح تفکر تنظیم شده باشد، به‌جای آن `thinking: { type: "adaptive" }` را اجباری می‌کند.

    سطوح تفکر موجود برای هر خانوادهٔ مدل:

    | خانوادهٔ مدل   | سطوح                                   | پیش‌فرض    |
    | -------------- | ----------------------------------------- | ---------- |
    | `MiniMax-M3`   | `off`، `adaptive`                        | `adaptive` |
    | `MiniMax-M2.x` | `off`، `minimal`، `low`، `medium`، `high` | `off`      |

  </Accordion>

  <Accordion title="حالت سریع">
    `/fast on` یا `params.fastMode: true`، مقدار `MiniMax-M2.7` را در مسیر استریم سازگار با Anthropic (`api: "anthropic-messages"`، ارائه‌دهنده `minimax` یا `minimax-portal`) به `MiniMax-M2.7-highspeed` بازنویسی می‌کند.
  </Accordion>

  <Accordion title="نمونهٔ جایگزینی هنگام خرابی">
    **مناسب برای:** قوی‌ترین مدل نسل جدید خود را به‌عنوان مدل اصلی حفظ کنید و هنگام خرابی به MiniMax M2.7 تغییر مسیر دهید. نمونهٔ زیر Opus را به‌عنوان یک مدل اصلی مشخص به‌کار می‌برد؛ آن را با مدل اصلی نسل جدید موردنظر خود جایگزین کنید.

    ```json5
    {
      env: { MINIMAX_API_KEY: "sk-..." },
      agents: {
        defaults: {
          models: {
            "anthropic/claude-opus-4-6": { alias: "primary" },
            "minimax/MiniMax-M2.7": { alias: "minimax" },
          },
          model: {
            primary: "anthropic/claude-opus-4-6",
            fallbacks: ["minimax/MiniMax-M2.7"],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="جزئیات استفاده از Coding Plan">
    - API مصرف Coding Plan: `https://api.minimaxi.com/v1/token_plan/remains` یا `https://api.minimax.io/v1/token_plan/remains` (به کلید Coding Plan نیاز دارد).
    - در صورت پیکربندی، نظرسنجی مصرف، میزبان را از `models.providers.minimax-portal.baseUrl` یا `models.providers.minimax.baseUrl` استخراج می‌کند؛ بنابراین راه‌اندازی‌های جهانی که از `https://api.minimax.io/anthropic` استفاده می‌کنند، `api.minimax.io` را نظرسنجی می‌کنند. URLهای پایهٔ مفقود یا نادرست برای سازگاری، جایگزین چین را حفظ می‌کنند.
    - OpenClaw مصرف Coding Plan مربوط به MiniMax را به همان نمایش `% left` که سایر ارائه‌دهندگان استفاده می‌کنند، عادی‌سازی می‌کند. فیلدهای خام `usage_percent` / `usagePercent` در MiniMax سهمیهٔ باقی‌مانده هستند، نه سهمیهٔ مصرف‌شده؛ بنابراین OpenClaw آن‌ها را معکوس می‌کند. در صورت وجود، فیلدهای مبتنی بر تعداد اولویت دارند.
    - وقتی API مقدار `model_remains` را برمی‌گرداند، OpenClaw ورودی مدل چت را ترجیح می‌دهد، در صورت نیاز برچسب بازه را از `start_time` / `end_time` استخراج می‌کند و نام مدل انتخاب‌شده را در برچسب طرح می‌گنجاند تا تشخیص بازه‌های Coding Plan آسان‌تر شود.
    - تصاویر لحظه‌ای مصرف، `minimax`، `minimax-cn`، `minimax-portal` و `minimax-portal-cn` را یک سطح سهمیهٔ یکسان MiniMax در نظر می‌گیرند و پیش از استفادهٔ جایگزین از متغیرهای محیطی کلید Coding Plan، OAuth ذخیره‌شدهٔ MiniMax را ترجیح می‌دهند.

  </Accordion>
</AccordionGroup>

## نکته‌ها

- مدل پیش‌فرض چت: `MiniMax-M3`. مدل‌های جایگزین چت: `MiniMax-M2.7`، `MiniMax-M2.7-highspeed`
- راه‌اندازی اولیه و راه‌اندازی مستقیم با کلید API، تعریف مدل‌های M3 و هر دو گونهٔ M2.7 را می‌نویسند
- درک تصویر از ارائه‌دهندهٔ رسانه‌ای `MiniMax-VL-01` متعلق به Plugin استفاده می‌کند
- اگر به ردیابی دقیق هزینه نیاز دارید، مقادیر قیمت‌گذاری را در `models.json` به‌روزرسانی کنید
- برای تأیید شناسهٔ فعلی ارائه‌دهنده از `openclaw models list` استفاده کنید، سپس با `openclaw models set minimax/MiniMax-M3` یا `openclaw models set minimax-portal/MiniMax-M3` تغییر دهید

<Note>
برای قواعد ارائه‌دهنده، به [ارائه‌دهندگان مدل](/fa/concepts/model-providers) مراجعه کنید.
</Note>

## عیب‌یابی

<AccordionGroup>
  <Accordion title='"مدل ناشناخته: minimax/MiniMax-M3"'>
    این معمولاً به این معنا است که **ارائه‌دهنده MiniMax پیکربندی نشده است** (هیچ ورودی منطبق ارائه‌دهنده و هیچ پروفایل احراز هویت یا کلید محیطی MiniMax یافت نشده است). برای رفع مشکل:

    - `openclaw configure` را اجرا و یک گزینهٔ احراز هویت **MiniMax** را انتخاب کنید، یا
    - بلوک منطبق `models.providers.minimax` یا `models.providers.minimax-portal` را به‌صورت دستی اضافه کنید، یا
    - `MINIMAX_API_KEY`، `MINIMAX_OAUTH_TOKEN` یا یک پروفایل احراز هویت MiniMax را تنظیم کنید تا ارائه‌دهندهٔ منطبق تزریق شود.

    مطمئن شوید شناسهٔ مدل **به بزرگی و کوچکی حروف حساس** است:

    - مسیر کلید API: `minimax/MiniMax-M3`، `minimax/MiniMax-M2.7` یا `minimax/MiniMax-M2.7-highspeed`
    - مسیر OAuth: `minimax-portal/MiniMax-M3`، `minimax-portal/MiniMax-M2.7` یا `minimax-portal/MiniMax-M2.7-highspeed`

    سپس با دستور زیر دوباره بررسی کنید:

    ```bash
    openclaw models list
    ```

  </Accordion>
</AccordionGroup>

<Note>
راهنمای بیشتر: [عیب‌یابی](/fa/help/troubleshooting) و [پرسش‌های متداول](/fa/help/faq).
</Note>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="تولید تصویر" href="/fa/tools/image-generation" icon="image">
    پارامترهای مشترک ابزار تصویر و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="تولید موسیقی" href="/fa/tools/music-generation" icon="music">
    پارامترهای مشترک ابزار موسیقی و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="تولید ویدیو" href="/fa/tools/video-generation" icon="video">
    پارامترهای مشترک ابزار ویدیو و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="جست‌وجوی MiniMax" href="/fa/tools/minimax-search" icon="magnifying-glass">
    پیکربندی جست‌وجوی وب از طریق MiniMax Token Plan.
  </Card>
  <Card title="عیب‌یابی" href="/fa/help/troubleshooting" icon="wrench">
    عیب‌یابی عمومی و پرسش‌های متداول.
  </Card>
</CardGroup>
