---
read_when:
    - برای بسیاری از LLMها یک کلید API واحد می‌خواهید
    - می‌خواهید مدل‌ها را از طریق OpenRouter در OpenClaw اجرا کنید
    - می‌خواهید از OpenRouter برای تولید تصویر استفاده کنید
    - می‌خواهید از OpenRouter برای تولید موسیقی استفاده کنید
    - می‌خواهید از OpenRouter برای تولید ویدئو استفاده کنید
summary: از API یکپارچهٔ OpenRouter برای دسترسی به مدل‌های متعدد در OpenClaw استفاده کنید
title: OpenRouter
x-i18n:
    generated_at: "2026-07-27T14:33:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c0936a10222f44f376dee081b7ee0678cddc3bc4579ac0006321dc1012d59bcf
    source_path: providers/openrouter.md
    workflow: 16
---

OpenRouter درخواست‌ها را با استفاده از یک API و یک کلید به مدل‌های بسیاری هدایت می‌کند. این سرویس
با OpenAI سازگار است، بنابراین OpenClaw از طریق همان انتقال به سبک
`openai-completions` که برای سایر ارائه‌دهندگان پروکسی استفاده می‌شود، با آن ارتباط برقرار می‌کند.

## شروع به کار

<Tabs>
  <Tab title="OAuth">
    <Steps>
      <Step title="اجرای راه‌اندازی اولیه OAuth">
        ```bash
        openclaw onboard --auth-choice openrouter-oauth
        ```

        OpenClaw جریان ورود مرورگری OpenRouter ‏(PKCE) را باز می‌کند، کد را
        با یک کلید API مربوط به OpenRouter مبادله می‌کند و آن را در پروفایل پیش‌فرض
        احراز هویت OpenRouter ذخیره می‌کند. در میزبان‌های راه‌دور یا بدون رابط گرافیکی، OpenClaw نشانی
        ورود را نمایش می‌دهد و پس از ورود از شما می‌خواهد نشانی تغییرمسیر را جای‌گذاری کنید.
      </Step>
      <Step title="(اختیاری) تغییر به یک مدل مشخص">
        راه‌اندازی اولیه به‌طور پیش‌فرض از `openrouter/auto` استفاده می‌کند. بعداً یک مدل مشخص انتخاب کنید:

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
  <Tab title="کلید API">
    <Steps>
      <Step title="دریافت کلید API">
        در [openrouter.ai/keys](https://openrouter.ai/keys) یک کلید API ایجاد کنید.
      </Step>
      <Step title="اجرای راه‌اندازی اولیه با کلید API">
        ```bash
        openclaw onboard --auth-choice openrouter-api-key
        ```
      </Step>
      <Step title="(اختیاری) تغییر به یک مدل مشخص">
        راه‌اندازی اولیه به‌طور پیش‌فرض از `openrouter/auto` استفاده می‌کند. بعداً یک مدل مشخص انتخاب کنید:

        ```bash
        openclaw models set openrouter/<provider>/<model>
        ```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## نمونه پیکربندی

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/auto" },
    },
  },
}
```

## ارجاع‌های مدل

<Note>
ارجاع‌های مدل از الگوی `openrouter/<provider>/<model>` پیروی می‌کنند. برای فهرست کامل
ارائه‌دهندگان و مدل‌های موجود، به [/concepts/model-providers](/fa/concepts/model-providers) مراجعه کنید.
</Note>

مدل‌های جایگزین همراه، که هنگام در دسترس نبودن کشف زنده کاتالوگ استفاده می‌شوند:

| ارجاع مدل                         | توضیحات                        |
| --------------------------------- | ---------------------------- |
| `openrouter/auto`                 | مسیریابی خودکار OpenRouter |
| `openrouter/moonshotai/kimi-k2.6` | Kimi K2.6 از طریق MoonshotAI     |
| `openrouter/moonshotai/kimi-k2.5` | Kimi K2.5 از طریق MoonshotAI     |

هر ارجاع `openrouter/<provider>/<model>` دیگری، از جمله
`openrouter/openrouter/fusion` (به [مسیریاب Fusion](#fusion-router) مراجعه کنید)، به‌صورت
پویا با کاتالوگ زنده مدل‌های OpenRouter تطبیق داده می‌شود.

## تولید تصویر

OpenRouter می‌تواند پشتیبان ابزار `image_generate` باشد. یک مدل تصویر OpenRouter را
در `agents.defaults.mediaModels.image` تنظیم کنید:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openrouter/google/gemini-3.1-flash-image-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

OpenClaw درخواست‌های تصویر را با
`modalities: ["image", "text"]` به API تصویر تکمیل‌های گفت‌وگوی OpenRouter ارسال می‌کند. مدل‌های تصویر Gemini علاوه بر این، راهنمایی‌های
`aspectRatio` و `resolution` را از طریق `image_config` متعلق به OpenRouter دریافت می‌کنند؛ سایر
مدل‌های تصویر چنین راهنمایی‌هایی دریافت نمی‌کنند. برای مدل‌های
کندتر از `agents.defaults.mediaModels.image.timeoutMs` استفاده کنید؛ مقدار `timeoutMs` در هر فراخوانی ابزار `image_generate` همچنان اولویت دارد.

## تولید ویدئو

OpenRouter می‌تواند از طریق API ناهمگام
`/videos` خود، پشتیبان ابزار `video_generate` باشد. یک مدل ویدئوی OpenRouter را در
`agents.defaults.mediaModels.video` تنظیم کنید:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "openrouter/google/veo-3.1-fast",
      },
    },
  },
}
```

OpenClaw کارهای تبدیل متن به ویدئو و تصویر به ویدئو را ارسال می‌کند،
`polling_url` بازگشتی را به‌طور دوره‌ای بررسی می‌کند و ویدئوی تکمیل‌شده را از
`unsigned_urls` متعلق به OpenRouter یا نقطه پایانی محتوای کار دانلود می‌کند. تصاویر مرجع به‌طور پیش‌فرض
به‌عنوان تصاویر فریم اول/آخر استفاده می‌شوند؛ در عوض، تصاویر دارای برچسب `reference_image` به‌عنوان
مراجع ورودی ارسال می‌شوند. مقدار پیش‌فرض همراه `google/veo-3.1-fast` از مدت‌زمان‌های 4/6/8
ثانیه‌ای، وضوح‌های `720P`/`1080P` و نسبت‌های ابعاد `16:9`/`9:16` پشتیبانی می‌کند.
تبدیل ویدئو به ویدئو پشتیبانی نمی‌شود: API بالادستی فقط متن و مراجع
تصویری را می‌پذیرد.

## تولید موسیقی

OpenRouter می‌تواند از طریق خروجی صوتی تکمیل‌های گفت‌وگو، پشتیبان ابزار `music_generate`
باشد. یک مدل صوتی OpenRouter را در
`agents.defaults.mediaModels.music` تنظیم کنید:

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "openrouter/google/lyria-3-pro-preview",
        timeoutMs: 180_000,
      },
    },
  },
}
```

ارائه‌دهنده موسیقی همراه OpenRouter به‌طور پیش‌فرض از `google/lyria-3-pro-preview`
استفاده می‌کند و `google/lyria-3-clip-preview` را نیز ارائه می‌دهد. OpenClaw مقدار `modalities:
["text", "audio"]` را ارسال می‌کند، پاسخ را به‌صورت جریانی دریافت می‌کند، قطعه‌های صوتی را گردآوری می‌کند و
نتیجه را به‌عنوان رسانه تولیدشده برای تحویل به کانال ذخیره می‌کند. مدل‌های Lyria یک
تصویر مرجع را از طریق پارامتر مشترک `music_generate image=...` می‌پذیرند.
صوت جریانی، نگه‌داری رونوشت و پوش رویداد SSE مشتق‌شده
به `agents.defaults.mediaMaxMb` محدود می‌شوند (سقف پیش‌فرض صوت 16 MB است).

## تبدیل متن به گفتار

OpenRouter می‌تواند از طریق endpoint سازگار با OpenAI خود با نام
`/audio/speech` به‌عنوان ارائه‌دهنده TTS عمل کند.

```json5
{
  tts: {
    auto: "always",
    provider: "openrouter",
    providers: {
      openrouter: {
        model: "hexgrad/kokoro-82m",
        speakerVoice: "af_alloy",
        responseFormat: "mp3",
      },
    },
  },
}
```

اگر `tts.providers.openrouter.apiKey` حذف شود، TTS ابتدا به
`models.providers.openrouter.apiKey` و سپس به `OPENROUTER_API_KEY` برمی‌گردد.

## تبدیل گفتار به متن (صدای ورودی)

OpenRouter می‌تواند پیوست‌های صوتی/صدای ورودی را از طریق مسیر مشترک
`tools.media.audio` و با استفاده از endpoint تبدیل گفتار به متن خود (`/audio/transcriptions`)
رونویسی کند. این قابلیت برای هر Plugin کانالی اعمال می‌شود که صدای ورودی را برای
پیش‌بررسی درک رسانه ارسال می‌کند.

```json5
{
  tools: {
    media: {
      audio: {
        enabled: true,
        models: [{ provider: "openrouter", model: "openai/whisper-large-v3-turbo" }],
      },
    },
  },
}
```

OpenClaw درخواست‌های تبدیل گفتار به متن OpenRouter را به‌صورت JSON و با صدای
base64 در `input_audio` (قرارداد تبدیل گفتار به متن OpenRouter) ارسال می‌کند،
نه به‌صورت بارگذاری فرم چندبخشی OpenAI.

## مسیریاب Fusion

OpenRouter Fusion یک ارجاع مدل OpenClaw را به‌صورت موازی به چند مدل OpenRouter
ارسال می‌کند، از OpenRouter می‌خواهد پاسخ‌های آن‌ها را داوری کند و یک پاسخ نهایی
را از طریق endpoint معمول OpenRouter بازمی‌گرداند. شناسه مدل بالادستی
`openrouter/fusion` است؛ بنابراین ارجاع مدل OpenClaw هم پیشوند ارائه‌دهنده OpenClaw
و هم فضای نام بالادستی OpenRouter را در خود دارد:

```bash
openclaw models set openrouter/openrouter/fusion
```

پنل و مدل داور Fusion را از طریق `params.extraBody` مدل پیکربندی کنید؛
این فیلدها مستقیماً به بدنه درخواست تکمیل‌های چت OpenRouter منتقل می‌شوند.
Fusion هم با راه‌اندازی اولیه OAuth و هم کلید API کار می‌کند؛ اگر از OAuth
استفاده می‌کنید، خط `env.OPENROUTER_API_KEY` زیر را حذف کنید.

```json5
{
  env: { OPENROUTER_API_KEY: "sk-or-..." },
  agents: {
    defaults: {
      model: { primary: "openrouter/openrouter/fusion" },
      models: {
        "openrouter/openrouter/fusion": {
          params: {
            extraBody: {
              plugins: [
                {
                  id: "fusion",
                  analysis_models: [
                    "google/gemini-3.5-flash",
                    "moonshotai/kimi-k2.6",
                    "deepseek/deepseek-v4-pro",
                  ],
                  model: "google/gemini-3.5-flash",
                },
              ],
            },
          },
        },
      },
    },
  },
}
```

`analysis_models` پنل موازی است؛ `model` در پیکربندی Plugin مربوط به
Fusion، مدل داور است. برای وادارکردن Fusion، در نوبت‌های عادی عامل/چت
`tool_choice` سطح بالا را روی `"required"` تنظیم نکنید: نوبت‌های
OpenClaw می‌توانند شامل تعریف ابزارهای خود باشند و انتخاب اجباری ابزار در سطح
بالا ممکن است به‌جای مسیریاب Fusion یکی از آن‌ها را انتخاب کند. وقتی این
پیکربندی Plugin مربوط به Fusion وجود داشته باشد، OpenClaw یادداشتی پالایش‌شده
به اعلان سیستم اضافه می‌کند که مدل‌های تحلیل پیکربندی‌شده و مدل داور را فهرست
می‌کند تا عامل بتواند به پرسش‌های مربوط به پنل Fusion خودش پاسخ دهد. سایر
فیلدهای `extraBody` در اعلان کپی نمی‌شوند.

Fusion بنا به طراحی کندتر است: OpenRouter اعلان را به چند مدل تحلیل ارسال می‌کند
و سپس مرحله داوری/ترکیب را اجرا می‌کند؛ بنابراین تأخیر از یک درخواست مستقیم
تک‌مدلی بیشتر است. از آن برای پاسخ‌های سنجیده و باکیفیت یا مسیرهای ارجاع استفاده
کنید، نه به‌عنوان پیش‌فرضی حساس به تأخیر. برای دریافت پاسخ‌های سریع‌تر، پنل را
کوچک نگه دارید و مدل‌های تحلیل/داور سریع‌تری انتخاب کنید.

یک ارجاع پیکربندی‌شده را با یک فراخوانی محلی یک‌باره آزمایش کنید:

```bash
openclaw infer model run --local \
  --model openrouter/openrouter/fusion \
  --prompt "دقیقاً با این عبارت پاسخ دهید: FUSION_OK" \
  --json
```

## احراز هویت و سرآیندها

OpenRouter از یک توکن Bearer برگرفته از کلید API شما استفاده می‌کند. OAuth در OpenRouter یک
جریان ورود PKCE است که یک کلید API متعلق به OpenRouter صادر می‌کند؛ بنابراین OpenClaw نتیجه را در
همان پروفایل احراز هویت کلید API با نام `openrouter:default` ذخیره می‌کند که در راه‌اندازی
دستی کلید API استفاده می‌شود.

برای ورود یا تعویض کلید ذخیره‌شده در یک نصب موجود، بدون اجرای دوباره
فرایند کامل راه‌اندازی اولیه:

```bash
openclaw models auth login --provider openrouter --method oauth
openclaw models auth login --provider openrouter --method api-key
```

در درخواست‌های تأییدشده OpenRouter (`https://openrouter.ai/api/v1`)،‏ OpenClaw
سرآیندهای مستندشده OpenRouter برای انتساب برنامه را اضافه می‌کند:

| سرآیند                    | مقدار                                                                                                  |
| ------------------------- | ------------------------------------------------------------------------------------------------------ |
| `HTTP-Referer`            | `https://openclaw.ai`                                                                                  |
| `X-OpenRouter-Title`      | `OpenClaw`                                                                                             |
| `X-OpenRouter-Categories` | `cli-agent,cloud-agent,programming-app,creative-writing,writing-assistant,general-chat,personal-agent` |

<Warning>
اگر ارائه‌دهنده OpenRouter را به پراکسی یا نشانی پایه دیگری هدایت کنید، OpenClaw
آن سرآیندهای مختص OpenRouter یا نشانگرهای کش Anthropic را تزریق **نمی‌کند**.
</Warning>

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="کش‌کردن پاسخ">
    کش‌کردن پاسخ OpenRouter اختیاری است. آن را برای هر مدل فعال کنید:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/auto": {
              params: {
                responseCache: true,
                responseCacheTtlSeconds: 300,
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw مقدار `X-OpenRouter-Cache: true` و در صورت پیکربندی،
    مقدار `X-OpenRouter-Cache-TTL` را ارسال می‌کند. `responseCacheClear: true` برای
    درخواست جاری نوسازی را اجباری می‌کند و پاسخ جایگزین را ذخیره می‌کند. نام‌های مستعار
    snake_case ‏(`response_cache`، `response_cache_ttl_seconds`،
    `response_cache_clear`) و نیز `responseCacheTtl` /
    `response_cache_ttl` بدون پسوند `Seconds` پذیرفته می‌شوند.

    این قابلیت از کش‌کردن پرامپت ارائه‌دهنده و نشانگرهای
    Anthropic ‏`cache_control` در OpenRouter مجزا است. این قابلیت فقط در مسیرهای
    تأییدشده `openrouter.ai` اعمال می‌شود، نه نشانی‌های پایه پراکسی سفارشی.

  </Accordion>

  <Accordion title="نشانگرهای کش Anthropic">
    در مسیرهای تأییدشده OpenRouter، ارجاع‌های مدل Anthropic نشانگرهای
    Anthropic ‏`cache_control` در OpenRouter را برای استفاده مجدد بهتر از کش پرامپت در
    بلوک‌های پرامپت system/developer حفظ می‌کنند.
  </Accordion>

  <Accordion title="پیش‌پرکردن استدلال Anthropic">
    در مسیرهای تأییدشده OpenRouter، ارجاع‌های مدل Anthropic که استدلال در آن‌ها فعال است،
    نوبت‌های پیش‌پرشده پایانی assistant را پیش از رسیدن درخواست به
    OpenRouter حذف می‌کنند تا با الزام Anthropic مبنی بر پایان‌یافتن گفتگوهای استدلالی
    با یک نوبت user مطابقت داشته باشند.
  </Accordion>

  <Accordion title="تزریق تفکر / استدلال">
    در مسیرهای پشتیبانی‌شدهٔ غیر `auto`، OpenClaw سطح تفکر انتخاب‌شده را
    به محموله‌های استدلال پراکسی OpenRouter نگاشت می‌کند. `openrouter/auto` و راهنمایی‌های
    پشتیبانی‌نشدهٔ مدل این تزریق را نادیده می‌گیرند. ارجاع‌های منسوخ `openrouter/hunter-alpha` نیز
    آن را نادیده می‌گیرند، زیرا OpenRouter ممکن است در آن مسیر بازنشسته، متن پاسخ نهایی را
    در فیلدهای استدلال برگرداند.
  </Accordion>

  <Accordion title="بازپخش استدلال DeepSeek V4">
    در مسیرهای تأییدشدهٔ OpenRouter، `openrouter/deepseek/deepseek-v4-flash` و
    `openrouter/deepseek/deepseek-v4-pro` مقدار `reasoning_content` مفقود را در
    نوبت‌های بازپخش‌شدهٔ دستیار تکمیل می‌کنند و گفت‌وگوهای تفکر/ابزار را در قالب پیگیری
    الزامی DeepSeek V4 نگه می‌دارند. OpenClaw برای این مسیرها مقادیر
    `reasoning.effort` مورد پشتیبانی OpenRouter را ارسال می‌کند: `xhigh`/`max` به `xhigh` نگاشت می‌شوند،
    و هر سطح غیرفعال‌نشدهٔ دیگری به `high` نگاشت می‌شود.
  </Accordion>

  <Accordion title="شکل‌دهی درخواست مختص OpenAI">
    OpenRouter از مسیر سازگار با OpenAI به‌سبک پراکسی اجرا می‌شود، بنابراین
    شکل‌دهی بومی درخواست مختص OpenAI، مانند `serviceTier`، مقدار `store` در Responses،
    محموله‌های سازگاری استدلال OpenAI و راهنمایی‌های کش پرامپت، ارسال نمی‌شود.
  </Accordion>

  <Accordion title="مسیرهای مبتنی بر Gemini">
    ارجاع‌های OpenRouter مبتنی بر Gemini در مسیر پراکسی-Gemini باقی می‌مانند: OpenClaw
    پاک‌سازی امضای تفکر Gemini را در آنجا حفظ می‌کند، اما اعتبارسنجی بومی
    بازپخش Gemini یا بازنویسی‌های راه‌اندازی اولیه را فعال نمی‌کند.
  </Accordion>

  <Accordion title="فرادادهٔ مسیریابی ارائه‌دهنده">
    OpenRouter برای مسیریابی ارائه‌دهندهٔ زیربنایی، از یک شیء درخواست `provider`
    پشتیبانی می‌کند. با `models.providers.openrouter.params.provider` یک خط‌مشی پیش‌فرض برای همهٔ درخواست‌های
    مدل متنی OpenRouter پیکربندی کنید:

    ```json5
    {
      models: {
        providers: {
          openrouter: {
            params: {
              provider: {
                sort: "latency",
                require_parameters: true,
                data_collection: "deny",
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw آن شیء را به‌عنوان محمولهٔ `provider` درخواست به OpenRouter
    ارسال می‌کند. از فیلدهای snake_case مستندشدهٔ OpenRouter استفاده کنید، از جمله `sort`،
    `only`، `ignore`، `order`، `allow_fallbacks`، `require_parameters`،
    `data_collection`، `quantizations`، `max_price`، `preferred_max_latency`،
    `preferred_min_throughput`، `zdr` و `enforce_distillable_text`.

    پارامترهای هر مدل، شیء مسیریابی سراسری ارائه‌دهنده را بازنویسی می‌کنند:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "openrouter/anthropic/claude-sonnet-4-6": {
              params: {
                provider: {
                  order: ["anthropic"],
                  allow_fallbacks: false,
                },
              },
            },
          },
        },
      },
    }
    ```

    این مورد فقط در مسیرهای chat-completions مربوط به OpenRouter اعمال می‌شود. مسیرهای مستقیم
    Anthropic، Google، OpenAI یا ارائه‌دهندگان سفارشی، پارامترهای مسیریابی OpenRouter را نادیده می‌گیرند.

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/configuration-reference" icon="gear">
    مرجع کامل پیکربندی عامل‌ها، مدل‌ها و ارائه‌دهندگان.
  </Card>
</CardGroup>
