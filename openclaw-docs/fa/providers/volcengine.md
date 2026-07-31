---
read_when:
    - می‌خواهید از مدل‌های Volcano Engine یا Doubao با OpenClaw استفاده کنید
    - باید کلید API مربوط به Volcengine را تنظیم کنید
    - می‌خواهید از تبدیل متن به گفتار Volcengine Speech استفاده کنید
summary: راه‌اندازی Volcano Engine (مدل‌های Doubao، نقاط پایانی کدنویسی و تبدیل متن به گفتار Seed Speech)
title: Volcengine (Doubao)
x-i18n:
    generated_at: "2026-07-27T14:37:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 89538772b704499547ecf0274c5bb9bf8f68cc267dc7f484d3236921a9c89681
    source_path: providers/volcengine.md
    workflow: 16
---

ارائه‌دهنده Volcengine دسترسی به مدل‌های Doubao و مدل‌های شخص ثالث میزبانی‌شده روی Volcano Engine را فراهم می‌کند و برای بارهای کاری عمومی و کدنویسی، نقطه‌های پایانی جداگانه دارد. همان Plugin همراه، Volcengine Speech را نیز به‌عنوان ارائه‌دهنده TTS ثبت می‌کند.

| جزئیات     | مقدار                                                      |
| ---------- | ---------------------------------------------------------- |
| ارائه‌دهندگان  | `volcengine` (عمومی + TTS)، `volcengine-plan` (کدنویسی)   |
| احراز هویت مدل | `VOLCANO_ENGINE_API_KEY`                                   |
| احراز هویت TTS   | `VOLCENGINE_TTS_API_KEY` یا `BYTEPLUS_SEED_SPEECH_API_KEY` |
| API        | مدل‌های سازگار با OpenAI، TTS از BytePlus Seed Speech         |

## شروع به کار

<Steps>
  <Step title="تنظیم کلید API">
    فرایند راه‌اندازی تعاملی را اجرا کنید:

    ```bash
    openclaw onboard --auth-choice volcengine-api-key
    ```

    با این کار، هر دو ارائه‌دهنده عمومی (`volcengine`) و کدنویسی (`volcengine-plan`) با یک کلید API ثبت می‌شوند.

  </Step>
  <Step title="تنظیم مدل پیش‌فرض">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "volcengine-plan/ark-code-latest" },
        },
      },
    }
    ```
  </Step>
  <Step title="بررسی دردسترس‌بودن مدل">
    ```bash
    openclaw models list --provider volcengine
    openclaw models list --provider volcengine-plan
    ```
  </Step>
</Steps>

<Tip>
برای راه‌اندازی غیرتعاملی (CI، اسکریپت‌نویسی)، کلید را مستقیماً ارسال کنید:

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice volcengine-api-key \
  --volcengine-api-key "$VOLCANO_ENGINE_API_KEY"
```

</Tip>

## ارائه‌دهندگان و نقطه‌های پایانی

| ارائه‌دهنده          | نقطه پایانی                                  | مورد استفاده       |
| ----------------- | ----------------------------------------- | -------------- |
| `volcengine`      | `ark.cn-beijing.volces.com/api/v3`        | مدل‌های عمومی |
| `volcengine-plan` | `ark.cn-beijing.volces.com/api/coding/v3` | مدل‌های کدنویسی  |

<Note>
هر دو ارائه‌دهنده با یک کلید API پیکربندی می‌شوند. راه‌اندازی، هر دو را به‌طور خودکار ثبت می‌کند و انتخاب‌گر مدل ارائه‌دهنده کدنویسی نیز از احراز هویت ارائه‌دهنده عمومی استفاده مجدد می‌کند (`volcengine-plan` نام مستعار احراز هویت `volcengine` است).
</Note>

## کاتالوگ داخلی

<Tabs>
  <Tab title="عمومی (volcengine)">
    | مرجع مدل                                    | نام                            | ورودی       | زمینه |
    | -------------------------------------------- | ------------------------------- | ----------- | ------- |
    | `volcengine/deepseek-v3-2-251201`            | DeepSeek V3.2                   | متن، تصویر | 128,000 |
    | `volcengine/doubao-seed-1-8-251228`          | Doubao Seed 1.8                 | متن، تصویر | 256,000 |
    | `volcengine/doubao-seed-code-preview-251028` | doubao-seed-code-preview-251028 | متن، تصویر | 256,000 |
    | `volcengine/glm-4-7-251222`                  | GLM 4.7                         | متن، تصویر | 200,000 |
    | `volcengine/kimi-k2-5-260127`                | Kimi K2.5                       | متن، تصویر | 256,000 |
  </Tab>
  <Tab title="کدنویسی (volcengine-plan)">
    | مرجع مدل                                         | نام                     | ورودی | زمینه |
    | ------------------------------------------------- | ------------------------ | ----- | ------- |
    | `volcengine-plan/ark-code-latest`                 | Ark Coding Plan          | متن  | 256,000 |
    | `volcengine-plan/doubao-seed-code`                | Doubao Seed Code         | متن  | 256,000 |
  </Tab>
</Tabs>

هر دو کاتالوگ ثابت هستند (بدون فراخوانی کشف `/models`) و از محاسبه میزان مصرف به‌صورت جریانی و سازگار با OpenAI پشتیبانی می‌کنند. طرح‌واره‌های ابزار برای هر دو ارائه‌دهنده، کلیدواژه‌های `minLength`، `maxLength`، `minItems`، `maxItems`، `minContains` و `maxContains` را به‌طور خودکار حذف می‌کنند، زیرا API فراخوانی ابزار Volcengine آن‌ها را رد می‌کند.

## تبدیل متن به گفتار

TTS در Volcengine از API مبتنی بر HTTP سرویس BytePlus Seed Speech (`voice.ap-southeast-1.bytepluses.com`) استفاده می‌کند و جدا از کلید API مدل Doubao سازگار با OpenAI پیکربندی می‌شود. در کنسول BytePlus، Seed Speech > Settings > API Keys را باز کنید، کلید API را کپی کنید و سپس موارد زیر را تنظیم کنید:

```bash
export VOLCENGINE_TTS_API_KEY="byteplus_seed_speech_api_key"
export VOLCENGINE_TTS_RESOURCE_ID="seed-tts-1.0"
```

سپس آن را در `openclaw.json` فعال کنید:

```json5
{
  tts: {
    auto: "always",
    provider: "volcengine",
    providers: {
      volcengine: {
        apiKey: "byteplus_seed_speech_api_key",
        voice: "en_female_anna_mars_bigtts",
        speedRatio: 1.0,
      },
    },
  },
}
```

فیلدهای موجود در `tts.providers.volcengine` عبارت‌اند از: `apiKey`، `voice`، `speedRatio` (0.2-3.0)، `emotion`، `cluster`، `resourceId`، `appKey` و `baseUrl`. وقتی بازنویسی تنظیم صدا مجاز باشد، `!emotion=<value>` نیز به‌عنوان دستور درون‌خطی صدا کار می‌کند.

برای مقصدهای پیام صوتی، OpenClaw قالب بومی ارائه‌دهنده یعنی `ogg_opus` را درخواست می‌کند. برای پیوست‌های صوتی معمولی، `mp3` را درخواست می‌کند. نام‌های مستعار ارائه‌دهنده یعنی `bytedance` و `doubao` نیز به همین ارائه‌دهنده گفتار نگاشت می‌شوند.

شناسه منبع پیش‌فرض `seed-tts-1.0` است؛ این همان مجوزی است که BytePlus به‌طور پیش‌فرض به کلیدهای API تازه‌ساخته‌شده Seed Speech اعطا می‌کند. اگر پروژه شما مجوز TTS 2.0 دارد، `VOLCENGINE_TTS_RESOURCE_ID=seed-tts-2.0` را تنظیم کنید.

<Warning>
`VOLCANO_ENGINE_API_KEY` برای نقطه‌های پایانی مدل ModelArk/Doubao است و کلید API سرویس Seed Speech نیست. TTS به یک کلید API سرویس Seed Speech از BytePlus Speech Console یا یک جفت AppID/توکن قدیمی Speech Console نیاز دارد.
</Warning>

احراز هویت قدیمی AppID/توکن برای برنامه‌های قدیمی‌تر Speech Console همچنان پشتیبانی می‌شود:

```bash
export VOLCENGINE_TTS_APPID="speech_app_id"
export VOLCENGINE_TTS_TOKEN="speech_access_token"
export VOLCENGINE_TTS_CLUSTER="volcano_tts"
```

دیگر متغیرهای محیطی اختیاری TTS: در صورت تنظیم، `VOLCENGINE_TTS_VOICE`، `VOLCENGINE_TTS_APP_KEY` و `VOLCENGINE_TTS_BASE_URL` فیلدهای پیکربندی متناظر `tts.providers.volcengine` را بازنویسی می‌کنند.

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="مدل پیش‌فرض پس از راه‌اندازی">
    `openclaw onboard --auth-choice volcengine-api-key` ضمن ثبت کاتالوگ عمومی `volcengine`، مدل `volcengine-plan/ark-code-latest` را به‌عنوان مدل پیش‌فرض تنظیم می‌کند.
  </Accordion>

  <Accordion title="رفتار جایگزین انتخاب‌گر مدل">
    هنگام انتخاب مدل در فرایند راه‌اندازی/پیکربندی، گزینه احراز هویت Volcengine هر دو ردیف `volcengine/*` و `volcengine-plan/*` را ترجیح می‌دهد. اگر این مدل‌ها هنوز بارگذاری نشده باشند، OpenClaw به‌جای نمایش انتخاب‌گر خالی و محدودشده به ارائه‌دهنده، به کاتالوگ فیلترنشده بازمی‌گردد.
  </Accordion>

  <Accordion title="متغیرهای محیطی برای فرایندهای پس‌زمینه">
    اگر Gateway به‌صورت فرایند پس‌زمینه (launchd/systemd) اجرا می‌شود، مطمئن شوید متغیرهای محیطی مدل و TTS مانند `VOLCANO_ENGINE_API_KEY`، `VOLCENGINE_TTS_API_KEY`، `BYTEPLUS_SEED_SPEECH_API_KEY`، `VOLCENGINE_TTS_APPID` و `VOLCENGINE_TTS_TOKEN` در دسترس آن فرایند هستند (برای نمونه، در `~/.openclaw/.env` یا از طریق `env.shellEnv`).
  </Accordion>
</AccordionGroup>

<Warning>
هنگام اجرای OpenClaw به‌عنوان سرویس پس‌زمینه، متغیرهای محیطی تنظیم‌شده در پوسته تعاملی به‌طور خودکار به ارث برده نمی‌شوند. یادداشت مربوط به فرایند پس‌زمینه در بالا را ببینید.
</Warning>

## مطالب مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، مراجع مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="پیکربندی" href="/fa/gateway/configuration" icon="gear">
    مرجع کامل پیکربندی عامل‌ها، مدل‌ها و ارائه‌دهندگان.
  </Card>
  <Card title="عیب‌یابی" href="/fa/help/troubleshooting" icon="wrench">
    مشکلات رایج و مراحل اشکال‌زدایی.
  </Card>
  <Card title="پرسش‌های متداول" href="/fa/help/faq" icon="circle-question">
    پرسش‌های پرتکرار درباره راه‌اندازی OpenClaw.
  </Card>
</CardGroup>
