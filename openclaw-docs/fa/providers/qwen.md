---
read_when:
    - می‌خواهید از Qwen با OpenClaw استفاده کنید
    - اشتراک Alibaba Cloud Token Plan دارید
summary: استفاده از Qwen Cloud از طریق Plugin آن در OpenClaw
title: Qwen
x-i18n:
    generated_at: "2026-07-27T16:07:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 74f94a35631dcdf8c9afc12e86d7a9d6b51a359411ba36f8820f8b1e7c03a27a
    source_path: providers/qwen.md
    workflow: 16
---

Qwen Cloud یک Plugin رسمی ارائه‌دهنده خارجی برای OpenClaw با شناسه متعارف `qwen` است. این Plugin نقاط پایانی Qwen Cloud / Alibaba DashScope Standard و Coding Plan را هدف قرار می‌دهد، Token Plan را با نام `qwen-token-plan` ارائه می‌کند، `modelstudio` را به‌عنوان نام مستعار سازگاری نگه می‌دارد و مستقلاً مالک شناسه ارائه‌دهنده سفارشی مستندشده Alibaba یعنی `bailian-token-plan` است.

| ویژگی                  | مقدار                                      |
| ---------------------- | ------------------------------------------ |
| ارائه‌دهنده            | `qwen`                                     |
| ارائه‌دهنده Token Plan | `qwen-token-plan`                          |
| متغیر محیطی ترجیحی     | `QWEN_API_KEY`                             |
| متغیر محیطی Token Plan | `QWEN_TOKEN_PLAN_API_KEY`                  |
| موارد پذیرفته‌شده دیگر (سازگاری) | `MODELSTUDIO_API_KEY`, `DASHSCOPE_API_KEY` |
| سبک API                 | سازگار با OpenAI                           |

<Tip>
`qwen3.7-plus` و `qwen3.6-plus` با نقاط پایانی Coding Plan و Standard کار می‌کنند.
برای `qwen3.7-max` یا `qwen3.6-flash`، از یک نقطه پایانی **Standard (پرداخت به‌ازای مصرف)** استفاده کنید.
</Tip>

## نصب Plugin

`qwen` به‌صورت یک Plugin رسمی خارجی عرضه می‌شود و همراه هسته نیست. آن را نصب و Gateway را راه‌اندازی مجدد کنید:

```bash
openclaw plugins install @openclaw/qwen-provider
openclaw gateway restart
```

## شروع به کار

نوع طرح خود را انتخاب کنید و مراحل راه‌اندازی را دنبال کنید.

<Tabs>
  <Tab title="Coding Plan (اشتراکی)">
    **مناسب برای:** دسترسی مبتنی بر اشتراک از طریق Qwen Coding Plan.

    <Steps>
      <Step title="دریافت کلید API">
        یک کلید API را در [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) ایجاد یا کپی کنید.
      </Step>
      <Step title="اجرای راه‌اندازی اولیه">
        برای نقطه پایانی **سراسری**:

        ```bash
        openclaw onboard --auth-choice qwen-api-key
        ```

        برای نقطه پایانی **چین**:

        ```bash
        openclaw onboard --auth-choice qwen-api-key-cn
        ```
      </Step>
      <Step title="تنظیم مدل پیش‌فرض">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```
      </Step>
      <Step title="تأیید دردسترس‌بودن مدل">
        ```bash
        openclaw models list --provider qwen
        ```
      </Step>
    </Steps>

    <Note>
    شناسه‌های قدیمی انتخاب احراز هویت `modelstudio-*` و ارجاع‌های مدل `modelstudio/...` همچنان
    به‌عنوان نام‌های مستعار سازگاری کار می‌کنند، اما جریان‌های راه‌اندازی جدید باید شناسه‌های متعارف
    انتخاب احراز هویت `qwen-*` و ارجاع‌های مدل `qwen/...` را ترجیح دهند. اگر یک ورودی سفارشی
    دقیق `models.providers.modelstudio` با مقدار `api` دیگری تعریف کنید، آن
    ارائه‌دهنده سفارشی به‌جای نام مستعار سازگاری Qwen مالک ارجاع‌های
    `modelstudio/...` خواهد بود.
    </Note>

  </Tab>

  <Tab title="Standard (پرداخت به‌ازای مصرف)">
    **مناسب برای:** دسترسی پرداخت به‌ازای مصرف از طریق نقطه پایانی Standard Model Studio، شامل `qwen3.7-max` و `qwen3.6-flash` که در Coding Plan در دسترس نیستند.

    <Steps>
      <Step title="دریافت کلید API">
        یک کلید API را در [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) ایجاد یا کپی کنید.
      </Step>
      <Step title="اجرای راه‌اندازی اولیه">
        برای نقطه پایانی **سراسری**:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key
        ```

        برای نقطه پایانی **چین**:

        ```bash
        openclaw onboard --auth-choice qwen-standard-api-key-cn
        ```
      </Step>
      <Step title="تنظیم مدل پیش‌فرض">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "qwen/qwen3.5-plus" },
            },
          },
        }
        ```
      </Step>
      <Step title="تأیید دردسترس‌بودن مدل">
        ```bash
        openclaw models list --provider qwen
        ```
      </Step>
    </Steps>

    <Note>
    شناسه‌های قدیمی انتخاب احراز هویت `modelstudio-*` و ارجاع‌های مدل `modelstudio/...` همچنان
    به‌عنوان نام‌های مستعار سازگاری کار می‌کنند، اما جریان‌های راه‌اندازی جدید باید شناسه‌های متعارف
    انتخاب احراز هویت `qwen-*` و ارجاع‌های مدل `qwen/...` را ترجیح دهند. اگر یک ورودی سفارشی
    دقیق `models.providers.modelstudio` با مقدار `api` دیگری تعریف کنید، آن
    ارائه‌دهنده سفارشی به‌جای نام مستعار سازگاری Qwen مالک ارجاع‌های
    `modelstudio/...` خواهد بود.
    </Note>

  </Tab>

  <Tab title="Token Plan (نسخه تیمی)">
    **مناسب برای:** دسترسی اشتراکی تیمی مبتنی بر اعتبار به Qwen و مدل‌های شخص ثالث پشتیبانی‌شده از طریق Alibaba Cloud Model Studio.

    <Steps>
      <Step title="دریافت کلید اختصاصی">
        یک جایگاه Token Plan اختصاص دهید و کلید اختصاصی `sk-sp-...` آن را ایجاد کنید. کلیدهای Token Plan، Coding Plan و پرداخت به‌ازای مصرف قابل‌جایگزینی با یکدیگر نیستند. به [نمای کلی Token Plan سراسری](https://www.alibabacloud.com/help/en/model-studio/token-plan-overview) یا [نمای کلی Token Plan چین](https://help.aliyun.com/zh/model-studio/token-plan-overview) مراجعه کنید.
      </Step>
      <Step title="اجرای راه‌اندازی اولیه">
        برای نقطه پایانی **سراسری / بین‌المللی** در سنگاپور:

        ```bash
        openclaw onboard --auth-choice qwen-token-plan
        ```

        برای نقطه پایانی **چین** در پکن:

        ```bash
        openclaw onboard --auth-choice qwen-token-plan-cn
        ```
      </Step>
      <Step title="تأیید ارائه‌دهنده">
        ```bash
        openclaw models list --provider qwen-token-plan
        openclaw agent --model qwen-token-plan/qwen3.7-plus --message "Reply with: token plan ready"
        ```
      </Step>
    </Steps>

    <Note>
    راهنمای OpenClaw متعلق به Alibaba از `bailian-token-plan` برای یک ارائه‌دهنده سفارشی
    دستی استفاده می‌کند. Plugin آن شناسه را به‌عنوان مالک سازگاری ثبت می‌کند، اما پیکربندی‌های
    جدید باید از `qwen-token-plan` استفاده کنند. یک ورودی سفارشی دقیق
    `models.providers.bailian-token-plan` مالکیت انتقال و کاتالوگ پیکربندی‌شده خود را
    حفظ می‌کند؛ این ورودی هرگز با کاتالوگ متعارف OpenAI ادغام نمی‌شود.
    </Note>

    <Warning>
    از Token Plan فقط برای نشست‌های تعاملی OpenClaw استفاده کنید. آن را برای
    کارهای Cron، اسکریپت‌های بدون نظارت یا بک‌اندهای برنامه انتخاب نکنید. Alibaba اعلام می‌کند که
    استفاده غیرتعاملی می‌تواند اشتراک را تعلیق یا کلید API آن را لغو کند.
    </Warning>

  </Tab>

</Tabs>

## انواع طرح و نقاط پایانی

| طرح                        | منطقه | انتخاب احراز هویت          | نقطه پایانی                                                     |
| -------------------------- | ------ | -------------------------- | ---------------------------------------------------------------- |
| Coding Plan (اشتراکی)      | چین    | `qwen-api-key-cn`          | `coding.dashscope.aliyuncs.com/v1`                               |
| Coding Plan (اشتراکی)      | سراسری | `qwen-api-key`             | `coding-intl.dashscope.aliyuncs.com/v1`                          |
| Standard (پرداخت به‌ازای مصرف) | چین    | `qwen-standard-api-key-cn` | `dashscope.aliyuncs.com/compatible-mode/v1`                      |
| Standard (پرداخت به‌ازای مصرف) | سراسری | `qwen-standard-api-key`    | `dashscope-intl.aliyuncs.com/compatible-mode/v1`                 |
| Token Plan (نسخه تیمی)     | چین    | `qwen-token-plan-cn`       | `token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`     |
| Token Plan (نسخه تیمی)     | سراسری | `qwen-token-plan`          | `token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1` |

ارائه‌دهنده نقطه پایانی را بر اساس انتخاب احراز هویت شما به‌طور خودکار انتخاب می‌کند. انتخاب‌های
متعارف از خانواده `qwen-*` استفاده می‌کنند؛ `modelstudio-*` فقط برای سازگاری باقی می‌ماند.
با یک `baseUrl` سفارشی در پیکربندی آن را بازنویسی کنید.

<Tip>
**مدیریت کلیدها:** [home.qwencloud.com/api-keys](https://home.qwencloud.com/api-keys) |
**مستندات:** [docs.qwencloud.com](https://docs.qwencloud.com/developer-guides/getting-started/introduction)
</Tip>

## کاتالوگ داخلی

OpenClaw این کاتالوگ ایستای Qwen را عرضه می‌کند. کاتالوگ از نقطه پایانی آگاه است: پیکربندی‌های Coding
Plan مدل‌هایی را که فقط روی نقطه پایانی Standard کار می‌کنند حذف می‌کنند.

| ارجاع مدل                   | ورودی       | زمینه     | توضیحات                 |
| --------------------------- | ----------- | --------- | ----------------------- |
| `qwen/qwen3.5-plus`         | متن، تصویر | 1,000,000 | مدل پیش‌فرض             |
| `qwen/qwen3.6-flash`        | متن، تصویر | 1,000,000 | فقط نقاط پایانی Standard |
| `qwen/qwen3.6-plus`         | متن، تصویر | 1,000,000 | Coding Plan + Standard  |
| `qwen/qwen3.7-max`          | متن         | 1,000,000 | فقط نقاط پایانی Standard |
| `qwen/qwen3.7-plus`         | متن، تصویر | 1,000,000 | Coding Plan + Standard  |
| `qwen/qwen3-max-2026-01-23` | متن         | 262,144   | خانواده Qwen Max        |
| `qwen/qwen3-coder-next`     | متن         | 262,144   | کدنویسی                 |
| `qwen/qwen3-coder-plus`     | متن         | 1,000,000 | کدنویسی                 |
| `qwen/MiniMax-M2.5`         | متن         | 1,000,000 | استدلال فعال            |
| `qwen/glm-5`                | متن         | 202,752   | GLM                     |
| `qwen/glm-4.7`              | متن         | 202,752   | GLM                     |
| `qwen/kimi-k2.5`            | متن، تصویر | 262,144   | Moonshot AI از طریق Alibaba |

<Note>
حتی وقتی مدلی در کاتالوگ ایستا وجود دارد، دردسترس‌بودن آن ممکن است بسته به نقطه پایانی و طرح
صورت‌حساب متفاوت باشد.
</Note>

### کاتالوگ Token Plan

Token Plan از یک فهرست مجاز جداگانه با تطبیق دقیق رشته استفاده می‌کند. مدل‌های طرحی که فقط برای
تولید تصویر هستند، چون از APIهای متفاوتی استفاده می‌کنند، در اینجا گنجانده نشده‌اند.

| ارجاع مدل                           | ورودی       | زمینه     |
| ----------------------------------- | ----------- | --------- |
| `qwen-token-plan/qwen3.7-max`       | متن         | 1,000,000 |
| `qwen-token-plan/qwen3.7-plus`      | متن، تصویر | 1,000,000 |
| `qwen-token-plan/qwen3.6-plus`      | متن، تصویر | 1,000,000 |
| `qwen-token-plan/qwen3.6-flash`     | متن، تصویر | 1,000,000 |
| `qwen-token-plan/deepseek-v4-pro`   | متن         | 1,000,000 |
| `qwen-token-plan/deepseek-v4-flash` | متن         | 1,000,000 |
| `qwen-token-plan/deepseek-v3.2`     | متن         | 131,072   |
| `qwen-token-plan/kimi-k2.7-code`    | متن، تصویر | 262,144   |
| `qwen-token-plan/kimi-k2.6`         | متن، تصویر | 262,144   |
| `qwen-token-plan/kimi-k2.5`         | متن، تصویر | 262,144   |
| `qwen-token-plan/glm-5.2`           | متن         | 1,000,000 |
| `qwen-token-plan/glm-5.1`           | متن         | 202,752   |
| `qwen-token-plan/glm-5`             | متن         | 202,752   |
| `qwen-token-plan/MiniMax-M2.5`      | متن         | 196,608   |

## کنترل‌های تفکر

`qwen3.7-max`، `qwen3.7-plus`، `qwen3.6-flash` و `qwen3.6-plus` در
کاتالوگ داخلی قابلیت استدلال دارند. برای مدل‌های استدلالی خانواده `qwen`،
ارائه‌دهنده سطوح تفکر OpenClaw را به پرچم درخواست سطح‌بالای
`enable_thinking` در DashScope نگاشت می‌کند: تفکر غیرفعال `enable_thinking: false` را ارسال می‌کند،
و هر سطح دیگری `enable_thinking: true` را می‌فرستد. مدل‌های سفارشی می‌توانند با تنظیم
`compat.thinkingFormat: "qwen-chat-template"` در ورودی مدل، محموله تفکر جایگزین مبتنی بر الگوی چت را
فعال کنند.

مدل‌های Token Plan نیز دارای قابلیت استدلال علامت‌گذاری شده‌اند. `kimi-k2.7-code` و
`MiniMax-M2.5` فقط در حالت تفکر کار می‌کنند، بنابراین OpenClaw حتی وقتی نشست
`/think off` را درخواست می‌کند، تفکر را فعال نگه می‌دارد. DeepSeek V4 مقادیر `minimal` تا `high` را به
میزان تلاش `high` سرویس نگاشت می‌کند و `xhigh` یا `max` را به `max` نگاشت می‌کند. GLM 5.2
بازه کامل `minimal` تا `max` را می‌پذیرد؛ GLM 5.1 و GLM 5 تا
`xhigh` را می‌پذیرند و مقدار پیش‌فرض هر سه `high` است. سایر مدل‌های ترکیبی از
حالت روشن/خاموش درخواستی پیروی می‌کنند.

## افزونه‌های چندوجهی

Plugin‏ `qwen` قابلیت‌های چندوجهی را فقط در نقاط پایانی **Standard** مربوط به DashScope
ارائه می‌کند، نه نقاط پایانی Coding Plan:

- **درک تصویر و ویدئو** از طریق `qwen3.6-plus`
- **تولید ویدئوی Wan** از طریق `wan2.6-t2v` (پیش‌فرض)، `wan2.6-i2v`، `wan2.6-r2v`، `wan2.6-r2v-flash`، `wan2.7-r2v`

درک رسانه به‌طور خودکار از احراز هویت پیکربندی‌شده Qwen تعیین می‌شود و به
پیکربندی اضافی نیاز ندارد. برای کارکرد درک رسانه، مطمئن شوید که از یک نقطه پایانی
Standard (پرداخت به‌ازای مصرف) استفاده می‌کنید.

برای تبدیل Qwen به ارائه‌دهنده پیش‌فرض ویدئو:

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: { primary: "qwen/wan2.6-t2v" },
    },
  },
}
```

محدودیت‌های تولید ویدئو: 1 ویدئوی خروجی در هر درخواست، حداکثر 1 تصویر ورودی
(تبدیل تصویر به ویدئو)، حداکثر 4 ویدئوی ورودی (تبدیل ویدئو به ویدئو)، و حداکثر مدت
10 ثانیه. از `size`، `aspectRatio`، `resolution`، `audio` و
`watermark` پشتیبانی می‌کند. ورودی‌های تصویر/ویدئوی مرجع به URLهای راه‌دور http(s) نیاز دارند؛ مسیرهای
فایل محلی از همان ابتدا رد می‌شوند، زیرا نقطهٔ پایانی ویدئوی DashScope بافرهای محلی بارگذاری‌شده
را برای این مراجع نمی‌پذیرد.

<Note>
برای پارامترهای مشترک ابزار، انتخاب ارائه‌دهنده و رفتار جایگزینی هنگام خرابی، به [تولید ویدئو](/fa/tools/video-generation) مراجعه کنید.
</Note>

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="دسترس‌پذیری Qwen 3.6 و 3.7">
    `qwen3.7-plus` و `qwen3.6-plus` در نقاط پایانی Coding Plan و Standard در دسترس هستند. `qwen3.7-max` و `qwen3.6-flash` فقط در Standard در دسترس‌اند. نقاط پایانی Standard (پرداخت به‌ازای مصرف) عبارت‌اند از:

    - چین: `dashscope.aliyuncs.com/compatible-mode/v1`
    - جهانی: `dashscope-intl.aliyuncs.com/compatible-mode/v1`

    OpenClaw، `qwen3.7-max` و `qwen3.6-flash` را از کاتالوگ‌های Coding Plan حذف می‌کند.
    اگر یک نقطهٔ پایانی Coding Plan برای هرکدام خطای «مدل پشتیبانی‌نشده» برگرداند،
    به نقطهٔ پایانی و کلید متناظر Standard تغییر دهید.

  </Accordion>

  <Accordion title="مسیریابی منطقه‌ای تولید ویدئو">
    OpenClaw پیش از ارسال یک کار ویدئویی، منطقهٔ پیکربندی‌شدهٔ Qwen را
    به میزبان متناظر DashScope AIGC نگاشت می‌کند:

    - جهانی/بین‌المللی: `https://dashscope-intl.aliyuncs.com`
    - چین: `https://dashscope.aliyuncs.com`

    یک `models.providers.qwen.baseUrl` معمولی که به میزبان‌های Qwen مربوط به Coding Plan
    یا Standard اشاره دارد، همچنان تولید ویدئو را به نقطهٔ پایانی منطقه‌ای
    متناظر ویدئوی DashScope هدایت می‌کند.

  </Accordion>

  <Accordion title="سازگاری مصرف در حالت جریانی">
    نقاط پایانی بومی Qwen سازگاری مصرف در حالت جریانی را روی انتقال مشترک
    `openai-completions` اعلام می‌کنند؛ بنابراین شناسه‌های سفارشی ارائه‌دهندهٔ سازگار با DashScope
    که همان میزبان‌های بومی را هدف قرار می‌دهند، بدون نیاز مشخص به
    شناسهٔ ارائه‌دهندهٔ داخلی `qwen`، همان رفتار را به ارث می‌برند. این مورد برای نقاط پایانی Coding Plan،
    Standard و Token Plan اعمال می‌شود:

    - `https://coding.dashscope.aliyuncs.com/v1`
    - `https://coding-intl.dashscope.aliyuncs.com/v1`
    - `https://dashscope.aliyuncs.com/compatible-mode/v1`
    - `https://dashscope-intl.aliyuncs.com/compatible-mode/v1`
    - `https://token-plan.ap-southeast-1.maas.aliyuncs.com/compatible-mode/v1`
    - `https://token-plan.cn-beijing.maas.aliyuncs.com/compatible-mode/v1`

  </Accordion>

  <Accordion title="برنامهٔ قابلیت‌ها">
    Plugin ‏`qwen` به‌عنوان جایگاه اصلی فروشنده برای تمام قابلیت‌های Qwen
    Cloud، نه فقط مدل‌های کدنویسی/متنی، در حال آماده‌سازی است.

    - **مدل‌های متن/گفت‌وگو:** از طریق Plugin در دسترس‌اند
    - **فراخوانی ابزار، خروجی ساختاریافته، تفکر:** از انتقال سازگار با OpenAI به ارث می‌رسند
    - **تولید تصویر:** در لایهٔ Plugin ارائه‌دهنده برنامه‌ریزی شده است
    - **درک تصویر/ویدئو:** از طریق Plugin در نقطهٔ پایانی Standard در دسترس است
    - **گفتار/صوت:** در لایهٔ Plugin ارائه‌دهنده برنامه‌ریزی شده است
    - **تعبیه‌ها/بازرتبه‌بندی حافظه:** از طریق سطح آداپتور تعبیه برنامه‌ریزی شده است
    - **تولید ویدئو:** از طریق Plugin و قابلیت مشترک تولید ویدئو در دسترس است

  </Accordion>

  <Accordion title="راه‌اندازی محیط و دیمن">
    اگر Gateway به‌صورت دیمن (launchd/systemd) اجرا می‌شود، مطمئن شوید `QWEN_API_KEY`
    یا `QWEN_TOKEN_PLAN_API_KEY` برای آن فرایند در دسترس است (برای مثال، در
    `~/.openclaw/.env` یا از طریق `env.shellEnv`).
  </Accordion>
</AccordionGroup>

## مطالب مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="تولید ویدئو" href="/fa/tools/video-generation" icon="video">
    پارامترهای مشترک ابزار ویدئو و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="Alibaba Model Studio" href="/fa/providers/alibaba" icon="cloud">
    ارائه‌دهندهٔ همراه تولید ویدئوی Wan روی همان پلتفرم DashScope.
  </Card>
  <Card title="عیب‌یابی" href="/fa/help/troubleshooting" icon="wrench">
    عیب‌یابی عمومی و پرسش‌های متداول.
  </Card>
</CardGroup>
