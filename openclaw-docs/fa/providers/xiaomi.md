---
read_when:
    - شما مدل‌های Xiaomi MiMo را در OpenClaw می‌خواهید
    - به احراز هویت Xiaomi MiMo یا راه‌اندازی Token Plan نیاز دارید
summary: استفاده از مدل‌های پرداخت به‌ازای‌مصرف و طرح توکن Xiaomi MiMo با OpenClaw
title: Xiaomi MiMo
x-i18n:
    generated_at: "2026-07-27T16:08:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ef79dea8332903c726f076c91b3b458e2d98534d402a412e7c156c06b2912a69
    source_path: providers/xiaomi.md
    workflow: 16
---

Xiaomi MiMo پلتفرم API برای مدل‌های **MiMo** است. Plugin همراه `xiaomi`
(`enabledByDefault: true`، بدون نیاز به نصب) دو ارائه‌دهندهٔ متن
به‌همراه یک ارائه‌دهندهٔ گفتار (TTS) ثبت می‌کند:

- `xiaomi` - کلیدهای پرداخت به‌ازای مصرف (`sk-...`)
- `xiaomi-token-plan` - کلیدهای طرح توکن (`tp-...`) با پیش‌تنظیم‌های نقطهٔ پایانی منطقه‌ای

| ویژگی         | مقدار                                                                                                                                              |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| شناسه‌های ارائه‌دهنده     | `xiaomi` (پرداخت به‌ازای مصرف)، `xiaomi-token-plan` (طرح توکن)                                                                                         |
| متغیرهای محیطی احراز هویت    | `XIAOMI_API_KEY`، `XIAOMI_TOKEN_PLAN_API_KEY`                                                                                                      |
| پرچم‌های راه‌اندازی اولیه | `--auth-choice xiaomi-api-key`، `--auth-choice xiaomi-token-plan-cn`، `--auth-choice xiaomi-token-plan-sgp`، `--auth-choice xiaomi-token-plan-ams` |
| پرچم‌های مستقیم CLI | `--xiaomi-api-key <key>`، `--xiaomi-token-plan-api-key <key>`                                                                                      |
| API              | تکمیل‌های گفت‌وگوی سازگار با OpenAI (`openai-completions`)                                                                                          |
| قرارداد گفتار  | `speechProviders: ["xiaomi"]`                                                                                                                      |
| URLهای پایه        | پرداخت به‌ازای مصرف: `https://api.xiaomimimo.com/v1`؛ طرح توکن: `token-plan-{cn,sgp,ams}.xiaomimimo.com/v1`                                            |
| مدل‌های پیش‌فرض   | `xiaomi/mimo-v2.5`، `xiaomi-token-plan/mimo-v2.5-pro`                                                                                              |
| پیش‌فرض TTS      | `mimo-v2.5-tts`، صدا `mimo_default`؛ مدل طراحی صدا `mimo-v2.5-tts-voicedesign`                                                               |

## شروع به کار

<Steps>
  <Step title="دریافت کلید مناسب">
    یک کلید پرداخت به‌ازای مصرف در [کنسول Xiaomi MiMo](https://platform.xiaomimimo.com/#/console/api-keys) ایجاد کنید، یا صفحهٔ اشتراک طرح توکن خود را باز کنید و URL پایهٔ منطقه‌ای سازگار با OpenAI را به‌همراه کلید `tp-...` متناظر کپی کنید.
  </Step>

  <Step title="اجرای راه‌اندازی اولیه">
    پرداخت به‌ازای مصرف:

    ```bash
    openclaw onboard --auth-choice xiaomi-api-key
    ```

    طرح توکن:

    ```bash
    openclaw onboard --auth-choice xiaomi-token-plan-sgp
    ```

    یا کلیدها را مستقیماً وارد کنید:

    ```bash
    openclaw onboard --auth-choice xiaomi-api-key --xiaomi-api-key "$XIAOMI_API_KEY"
    openclaw onboard --auth-choice xiaomi-token-plan-sgp --xiaomi-token-plan-api-key "$XIAOMI_TOKEN_PLAN_API_KEY"
    ```

  </Step>
  <Step title="بررسی دردسترس‌بودن مدل">
    ```bash
    openclaw models list --provider xiaomi
    openclaw models list --provider xiaomi-token-plan
    ```
  </Step>
</Steps>

<Tip>
راه‌اندازی اولیه قالب کلید را اعتبارسنجی می‌کند و هنگامی هشدار می‌دهد که کلید `tp-...` در مسیر پرداخت به‌ازای مصرف یا کلید `sk-...` در مسیر طرح توکن وارد شود.
</Tip>

## فهرست مدل‌های پرداخت به‌ازای مصرف

| ارجاع مدل              | ورودی       | زمینه   | حداکثر خروجی | استدلال | یادداشت‌ها         |
| ---------------------- | ----------- | --------- | ---------- | --------- | ------------- |
| `xiaomi/mimo-v2.5`     | متن، تصویر | 1,048,576 | 131,072    | بله       | مدل پیش‌فرض |
| `xiaomi/mimo-v2.5-pro` | متن        | 1,048,576 | 131,072    | بله       | پرچم‌دار      |

## فهرست مدل‌های طرح توکن

گزینهٔ احراز هویت طرح توکن را انتخاب کنید که با URL پایهٔ منطقه‌ای نمایش‌داده‌شده در رابط اشتراک Xiaomi مطابقت دارد:

| گزینهٔ احراز هویت             | URL پایه                                   |
| ----------------------- | ------------------------------------------ |
| `xiaomi-token-plan-cn`  | `https://token-plan-cn.xiaomimimo.com/v1`  |
| `xiaomi-token-plan-sgp` | `https://token-plan-sgp.xiaomimimo.com/v1` |
| `xiaomi-token-plan-ams` | `https://token-plan-ams.xiaomimimo.com/v1` |

| ارجاع مدل                         | ورودی       | زمینه   | حداکثر خروجی | استدلال | یادداشت‌ها         |
| --------------------------------- | ----------- | --------- | ---------- | --------- | ------------- |
| `xiaomi-token-plan/mimo-v2.5-pro` | متن        | 1,048,576 | 131,072    | بله       | مدل پیش‌فرض |
| `xiaomi-token-plan/mimo-v2.5`     | متن، تصویر | 1,048,576 | 131,072    | بله       | چندوجهی    |

`xiaomi-token-plan` برای تفکیک به یک URL پایهٔ منطقه‌ای نیاز دارد. مسیر پشتیبانی‌شده
استفاده از یکی از گزینه‌های راه‌اندازی اولیهٔ طرح توکن همراه یا یک بلوک پیکربندی صریح
`models.providers.xiaomi-token-plan` با تنظیم `baseUrl` است؛
بدون یکی از این موارد، ارائه‌دهنده عرضه نمی‌شود.

## مدل‌های استدلالی

`mimo-v2.5` و `mimo-v2.5-pro` از
[دستورالعمل `/think` در OpenClaw](/fa/tools/thinking) با سطوح `off`،
`minimal`، `low`، `medium`، `high`، `xhigh` و `max` (پیش‌فرض `high`) پشتیبانی می‌کنند.

## تبدیل متن به گفتار

Plugin همراه `xiaomi` همچنین Xiaomi MiMo را به‌عنوان ارائه‌دهندهٔ گفتار
برای `tts` ثبت می‌کند. این Plugin قرارداد TTS تکمیل گفت‌وگوی Xiaomi را با
متن در قالب یک پیام `assistant` و راهنمای اختیاری سبک در قالب یک پیام `user`
فراخوانی می‌کند.

| ویژگی | مقدار                                    |
| -------- | ---------------------------------------- |
| شناسهٔ TTS   | `xiaomi` (نام مستعار `mimo`)                  |
| احراز هویت     | `XIAOMI_API_KEY`                         |
| API      | `POST /v1/chat/completions` با `audio` |
| پیش‌فرض  | `mimo-v2.5-tts`، صدا `mimo_default`    |
| خروجی   | به‌طور پیش‌فرض MP3؛ هنگام پیکربندی WAV      |

```json5
{
  tts: {
    auto: "always",
    provider: "xiaomi",
    providers: {
      xiaomi: {
        apiKey: "xiaomi_api_key",
        model: "mimo-v2.5-tts",
        speakerVoice: "mimo_default",
        format: "mp3",
        style: "Bright, natural, conversational tone.",
      },
    },
  },
}
```

صداهای داخلی: `mimo_default`، `default_zh`، `default_en`، `Mia`، `Chloe`،
`Milo`، `Dean`. مدل صدای ازپیش‌تنظیم‌شدهٔ `mimo-v2.5-tts` از `audio.voice` استفاده می‌کند، بنابراین
OpenClaw برای آن مدل `speakerVoice` را ارسال می‌کند.

مدل طراحی صدای `mimo-v2.5-tts-voicedesign` به‌جای شناسهٔ صدای ازپیش‌تنظیم‌شده، صدا را از یک
پرامپت سبک به زبان طبیعی تولید می‌کند. `style` را روی
توصیف صدای موردنظر تنظیم کنید؛ OpenClaw آن را در قالب پیام `user` ارسال می‌کند،
متن گفتاری را در قالب پیام `assistant` می‌فرستد و `audio.voice` را برای این
مدل حذف می‌کند.

```json5
{
  tts: {
    provider: "xiaomi",
    providers: {
      xiaomi: {
        model: "mimo-v2.5-tts-voicedesign",
        format: "wav",
        style: "Warm, natural female voice with clear pronunciation.",
      },
    },
  },
}
```

برای کانال‌هایی که هدف سنتز یادداشت صوتی را درخواست می‌کنند (Discord، Feishu،
Matrix، Telegram و WhatsApp)، OpenClaw پیش از تحویل، خروجی Xiaomi را با
`ffmpeg` به Opus تک‌کانالهٔ 48kHz تبدیل می‌کند.

## نمونهٔ پیکربندی

```json5
{
  env: { XIAOMI_API_KEY: "your-key" },
  agents: { defaults: { model: { primary: "xiaomi/mimo-v2.5" } } },
  models: {
    mode: "merge",
    providers: {
      xiaomi: {
        baseUrl: "https://api.xiaomimimo.com/v1",
        api: "openai-completions",
        apiKey: "XIAOMI_API_KEY",
        models: [
          {
            id: "mimo-v2.5",
            name: "Xiaomi MiMo V2.5",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
          {
            id: "mimo-v2.5-pro",
            name: "Xiaomi MiMo V2.5 Pro",
            reasoning: true,
            input: ["text"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

قیمت‌گذاری و پرچم‌های سازگاری از مانیفست Plugin همراه می‌آیند، بنابراین نمونهٔ پیکربندی برای جلوگیری از تفاوت با رفتار زمان اجرا، `cost` و `compat` را حذف می‌کند.

طرح توکن:

```json5
{
  env: { XIAOMI_TOKEN_PLAN_API_KEY: "tp-your-key" },
  agents: { defaults: { model: { primary: "xiaomi-token-plan/mimo-v2.5-pro" } } },
  models: {
    mode: "merge",
    providers: {
      "xiaomi-token-plan": {
        baseUrl: "https://token-plan-sgp.xiaomimimo.com/v1",
        api: "openai-completions",
        apiKey: "XIAOMI_TOKEN_PLAN_API_KEY",
        models: [
          {
            id: "mimo-v2.5-pro",
            name: "Xiaomi MiMo V2.5 Pro",
            reasoning: true,
            input: ["text"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
          {
            id: "mimo-v2.5",
            name: "Xiaomi MiMo V2.5",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048576,
            maxTokens: 131072,
          },
        ],
      },
    },
  },
}
```

قیمت‌گذاری از مانیفست همراه می‌آید (مدل‌های طرح توکن شامل قیمت‌گذاری سطح‌بندی‌شده برای خواندن کش هستند)، بنابراین نمونهٔ پیکربندی `cost` را حذف می‌کند.

<AccordionGroup>
  <Accordion title="رفتار تزریق خودکار">
    ارائه‌دهندهٔ `xiaomi` هنگامی‌که `XIAOMI_API_KEY` در محیط تنظیم شده باشد یا یک نمایهٔ احراز هویت وجود داشته باشد، به‌طور خودکار فعال می‌شود. `xiaomi-token-plan` به یک URL پایهٔ منطقه‌ای نیاز دارد، بنابراین مسیر پشتیبانی‌شده استفاده از گزینهٔ راه‌اندازی اولیهٔ طرح توکن همراه یا یک بلوک پیکربندی صریح `models.providers.xiaomi-token-plan` است.
  </Accordion>

  <Accordion title="جزئیات مدل">
    - **mimo-v2.5** - مسیر پیش‌فرض پرداخت به‌ازای مصرف و مسیر چندوجهی V2.5 طرح توکن.
    - **mimo-v2.5-pro** - مدل استدلالی پرچم‌دار و پیش‌فرض طرح توکن.

    <Note>
    مدل‌های پرداخت به‌ازای مصرف از پیشوند `xiaomi/` استفاده می‌کنند. مدل‌های طرح توکن از پیشوند `xiaomi-token-plan/` استفاده می‌کنند.
    </Note>

  </Accordion>

  <Accordion title="عیب‌یابی">
    - اگر مدل‌ها ظاهر نمی‌شوند، تأیید کنید که متغیر محیطی کلید مرتبط یا نمایهٔ احراز هویت موجود و معتبر است.
    - برای طرح توکن، تأیید کنید که منطقهٔ انتخاب‌شده در راه‌اندازی اولیه با URL پایهٔ صفحهٔ اشتراک مطابقت دارد و کلید با `tp-` آغاز می‌شود.
    - هنگامی‌که Gateway به‌صورت سرویس پس‌زمینه اجرا می‌شود، مطمئن شوید کلید برای آن فرایند در دسترس است (برای مثال در `~/.openclaw/.env` یا از طریق `env.shellEnv`).

    <Warning>
    کلیدهایی که فقط در پوستهٔ تعاملی تنظیم شده‌اند، برای فرایندهای Gateway مدیریت‌شده به‌صورت سرویس پس‌زمینه قابل مشاهده نیستند. برای دسترسی پایدار از پیکربندی `~/.openclaw/.env` یا `env.shellEnv` استفاده کنید.
    </Warning>

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="سطوح تفکر" href="/fa/tools/thinking" icon="brain">
    نحو دستورالعمل `/think` و نگاشت سطوح.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/configuration-reference" icon="gear">
    مرجع کامل پیکربندی OpenClaw.
  </Card>
  <Card title="کنسول Xiaomi MiMo" href="https://platform.xiaomimimo.com" icon="arrow-up-right-from-square">
    داشبورد Xiaomi MiMo و مدیریت کلید API.
  </Card>
</CardGroup>
