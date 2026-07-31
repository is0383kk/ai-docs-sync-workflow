---
read_when:
    - راه‌اندازی Moonshot Kimi K3/K2 (Moonshot Open Platform) در مقایسه با Kimi Coding مدنظر شماست
    - باید endpointها، کلیدها و ارجاع‌های مدلِ جداگانه را درک کنید
    - برای هر یک از ارائه‌دهندگان، پیکربندی آمادهٔ کپی/جای‌گذاری می‌خواهید
summary: پیکربندی مدل‌های Moonshot Kimi در مقایسه با Kimi Coding (ارائه‌دهندگان و کلیدهای جداگانه)
title: Moonshot AI
x-i18n:
    generated_at: "2026-07-27T17:04:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 213379bf88fec26b052184a920e112f0887d6485601bfb47f590cf37ef983e58
    source_path: providers/moonshot.md
    workflow: 16
---

Moonshot، API مربوط به Kimi را با نقاط پایانی سازگار با OpenAI ارائه می‌کند. برای Kimi K3،
`moonshot/kimi-k3` را انتخاب کنید، مقدار پیش‌فرض راه‌اندازی
`moonshot/kimi-k2.6` را نگه دارید، یا برای Kimi Coding از `kimi/kimi-for-coding` استفاده کنید.

<Warning>
Moonshot و Kimi Coding **ارائه‌دهندگان جداگانه‌ای هستند** که هرکدام به‌صورت یک Plugin خارجی مجزا عرضه می‌شوند. کلیدها قابل‌استفاده به‌جای یکدیگر نیستند، نقاط پایانی متفاوت‌اند و ارجاعات مدل نیز تفاوت دارند (`moonshot/...` در برابر `kimi/...`).
</Warning>

## کاتالوگ داخلی مدل‌ها

[//]: # "moonshot-kimi-k2-ids:start"

| ارجاع مدل                           | نام                     | استدلال  | ورودی       | زمینه   | حداکثر خروجی |
| ----------------------------------- | ------------------------ | ---------- | ----------- | --------- | ---------- |
| `moonshot/kimi-k2.6`                | Kimi K2.6                | خیر         | متن، تصویر | 262,144   | 262,144    |
| `moonshot/kimi-k3`                  | Kimi K3                  | همیشه حداکثر | متن، تصویر | 1,048,576 | 1,048,576  |
| `moonshot/kimi-k2.7-code`           | Kimi K2.7 Code           | همیشه فعال  | متن، تصویر | 262,144   | 262,144    |
| `moonshot/kimi-k2.7-code-highspeed` | Kimi K2.7 Code HighSpeed | همیشه فعال  | متن، تصویر | 262,144   | 262,144    |
| `moonshot/kimi-k2.5`                | Kimi K2.5                | خیر         | متن، تصویر | 262,144   | 262,144    |

[//]: # "moonshot-kimi-k2-ids:end"

برآوردهای هزینه در کاتالوگ از تعرفه‌های پرداخت به‌ازای مصرف منتشرشده توسط Moonshot استفاده می‌کنند. پیش از تصمیم‌گیری درباره هزینه،
صفحات زنده فروشنده را برای [Kimi K3](https://platform.kimi.ai/docs/pricing/chat-k3)،
[Kimi K2.7 Code](https://platform.kimi.ai/docs/pricing/chat-k27-code)،
[Kimi K2.6](https://platform.kimi.ai/docs/pricing/chat-k26) و
[Kimi K2.5](https://platform.kimi.ai/docs/pricing/chat-k25) بررسی کنید.

Kimi K3 همیشه در سطح `reasoning_effort: "max"` استدلال می‌کند. OpenClaw فقط
`/think max` را در دسترس قرار می‌دهد، فیلد مختص K2 یعنی `thinking` را حذف می‌کند و بازنویسی‌های نمونه‌گیری
(`temperature`، `top_p`، `n`، `presence_penalty` و
`frequency_penalty`) را که K3 روی مقادیر پیش‌فرض ارائه‌دهنده ثابت می‌کند، کنار می‌گذارد. Kimi K2.7 Code نیز
همیشه از تفکر بومی استفاده می‌کند، اما لازم است هر دو `thinking` و
`reasoning_effort` حذف شوند؛ گونه HighSpeed نیز از همین قرارداد استفاده می‌کند.
Kimi K2.6 همچنان مقدار پیش‌فرض راه‌اندازی است.
به [شروع سریع Kimi K3 در Moonshot](https://platform.kimi.ai/docs/guide/kimi-k3-quickstart) مراجعه کنید.

## شروع به کار

Moonshot و Kimi Coding هردو Plugin خارجی هستند — پیش از
راه‌اندازی یکی را نصب کنید.

<Tabs>
  <Tab title="API مربوط به Moonshot">
    **مناسب برای:** مدل‌های Kimi K3 و K2 از طریق Moonshot Open Platform.

    <Steps>
      <Step title="نصب Plugin">
        ```bash
        openclaw plugins install @openclaw/moonshot-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="انتخاب منطقه نقطه پایانی">
        | انتخاب احراز هویت            | نقطه پایانی                       | منطقه        |
        | ---------------------- | ------------------------------ | ------------- |
        | `moonshot-api-key`     | `https://api.moonshot.ai/v1`   | بین‌المللی |
        | `moonshot-api-key-cn`  | `https://api.moonshot.cn/v1`   | چین         |
      </Step>
      <Step title="اجرای راه‌اندازی">
        ```bash
        openclaw onboard --auth-choice moonshot-api-key
        ```

        یا برای نقطه پایانی چین:

        ```bash
        openclaw onboard --auth-choice moonshot-api-key-cn
        ```
      </Step>
      <Step title="تنظیم Kimi K3 به‌عنوان مدل پیش‌فرض">
        راه‌اندازی، Kimi K2.6 را به‌عنوان مقدار پیش‌فرض اولیه نگه می‌دارد. هر زمان
        Kimi K3 را می‌خواهید، صریحاً به آن تغییر دهید:

        ```bash
        openclaw models set moonshot/kimi-k3
        ```
      </Step>
      <Step title="تأیید دردسترس‌بودن مدل‌ها">
        ```bash
        openclaw models list --provider moonshot
        ```
      </Step>
      <Step title="اجرای یک آزمون دود زنده">
        هنگامی که می‌خواهید دسترسی به مدل و ردیابی هزینه را بدون
        دست‌زدن به نشست‌های عادی خود تأیید کنید، از یک پوشه وضعیت ایزوله استفاده کنید:

        ```bash
        OPENCLAW_CONFIG_PATH=/tmp/openclaw-kimi/openclaw.json \
        OPENCLAW_STATE_DIR=/tmp/openclaw-kimi \
        openclaw agent --local \
          --session-id live-kimi-cost \
          --message 'دقیقاً پاسخ دهید: KIMI_LIVE_OK' \
          --thinking max \
          --json
        ```

        پاسخ JSON باید `provider: "moonshot"` و
        `model: "kimi-k3"` را گزارش کند. ورودی رونوشت دستیار، میزان مصرف نرمال‌شده
        توکن را به‌همراه هزینه تخمینی در `usage.cost` ذخیره می‌کند، مشروط بر اینکه Moonshot
        فراداده مصرف را برگرداند.
      </Step>
    </Steps>

    ### نمونه پیکربندی

    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: {
            // moonshot-kimi-k2-aliases:start
            "moonshot/kimi-k2.6": { alias: "Kimi K2.6" },
            "moonshot/kimi-k3": { alias: "Kimi K3" },
            "moonshot/kimi-k2.7-code": { alias: "Kimi K2.7 Code" },
            "moonshot/kimi-k2.7-code-highspeed": { alias: "Kimi K2.7 Code HighSpeed" },
            "moonshot/kimi-k2.5": { alias: "Kimi K2.5" },
            // moonshot-kimi-k2-aliases:end
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              // moonshot-kimi-k2-models:start
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k3",
                name: "Kimi K3",
                reasoning: true,
                thinkingLevelMap: {
                  off: null,
                  minimal: null,
                  low: null,
                  medium: null,
                  high: null,
                  xhigh: "max",
                  max: "max",
                },
                input: ["text", "image"],
                cost: { input: 3, output: 15, cacheRead: 0.3, cacheWrite: 0 },
                contextWindow: 1048576,
                maxTokens: 1048576,
              },
              {
                id: "kimi-k2.7-code",
                name: "Kimi K2.7 Code",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.19, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.7-code-highspeed",
                name: "Kimi K2.7 Code HighSpeed",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 1.9, output: 8, cacheRead: 0.38, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              {
                id: "kimi-k2.5",
                name: "Kimi K2.5",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.6, output: 3, cacheRead: 0.1, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
              // moonshot-kimi-k2-models:end
            ],
          },
        },
      },
    }
    ```

  </Tab>

  <Tab title="Kimi Coding">
    **مناسب برای:** وظایف متمرکز بر کدنویسی از طریق نقطه پایانی Kimi Coding.

    <Note>
    Kimi Coding از کلید API و پیشوند ارائه‌دهنده متفاوتی (`kimi/...`) نسبت به Moonshot (`moonshot/...`) استفاده می‌کند. ارجاعات فعلی عبارت‌اند از `kimi/k3` برای زمینه 256K، `kimi/k3[1m]` برای سطح 1M، `kimi/kimi-for-coding` و `kimi/kimi-for-coding-highspeed`. ارجاعات قدیمی `kimi/kimi-code` و `kimi/k2p5` همچنان پذیرفته می‌شوند و به `kimi/kimi-for-coding` نرمال‌سازی می‌شوند.
    </Note>

    سرویس کدنویسی، هم کلاینت‌های سازگار با OpenAI یعنی
    `https://api.kimi.com/coding/v1` و هم کلاینت‌های سازگار با Anthropic یعنی
    `https://api.kimi.com/coding/` را می‌پذیرد. این Plugin از Anthropic Messages استفاده می‌کند.
    کلیدهای عضویت را در
    [کنسول Kimi Code](https://www.kimi.com/code/console) ایجاد کنید؛ قیمت‌گذاری فعلی عضویت
    در [صفحه قیمت‌گذاری Kimi](https://www.kimi.com/membership/pricing) قرار دارد.

    <Steps>
      <Step title="نصب Plugin">
        ```bash
        openclaw plugins install @openclaw/kimi-provider
        openclaw gateway restart
        ```
      </Step>
      <Step title="اجرای راه‌اندازی">
        ```bash
        openclaw onboard --auth-choice kimi-code-api-key
        ```
      </Step>
      <Step title="تنظیم یک مدل پیش‌فرض">
        ```json5
        {
          agents: {
            defaults: {
              model: { primary: "kimi/kimi-for-coding" },
            },
          },
        }
        ```
      </Step>
      <Step title="تأیید دردسترس‌بودن مدل">
        ```bash
        openclaw models list --provider kimi
        ```
      </Step>
    </Steps>

    Kimi Code K3 به‌طور پیش‌فرض از تفکر عمیق در سطح `max` استفاده می‌کند. `/think off`
    مقدار `thinking.type: "disabled"` را ارسال می‌کند؛ `/think max` درخواست تفکر تطبیقی
    K3 را با حداکثر تلاش ارسال می‌کند. سطوح پایین‌تر و منسوخ تفکر به سطح
    پشتیبانی‌شده `max` تبدیل می‌شوند. مدل 1M به عضویت Allegretto یا بالاتر در Kimi
    نیاز دارد؛ در Moderato از `kimi/k3` استفاده کنید.

    برای دردسترس‌بودن فعلی طرح‌ها، به [جدول رسمی مدل‌های Kimi Code](https://www.kimi.com/code/docs/en/kimi-code/models.html) مراجعه کنید.

    ### نمونه پیکربندی

    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: {
            "kimi/kimi-for-coding": { alias: "Kimi" },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

## جست‌وجوی وب Kimi

Plugin مربوط به Moonshot همچنین **Kimi** را به‌عنوان ارائه‌دهنده `web_search` ثبت می‌کند که از جست‌وجوی وب Moonshot پشتیبانی می‌شود.

<Steps>
  <Step title="اجرای راه‌اندازی تعاملی جست‌وجوی وب">
    ```bash
    openclaw configure --section web
    ```

    برای ذخیره
    `plugins.entries.moonshot.config.webSearch.*`، در بخش جست‌وجوی وب **Kimi** را انتخاب کنید.

  </Step>
  <Step title="پیکربندی منطقه و مدل جست‌وجوی وب">
    راه‌اندازی تعاملی موارد زیر را درخواست می‌کند:

    | تنظیم             | گزینه‌ها                                                              |
    | ------------------- | -------------------------------------------------------------------- |
    | منطقه API          | `https://api.moonshot.ai/v1` (بین‌المللی) یا `https://api.moonshot.cn/v1` (چین) |
    | مدل جست‌وجوی وب    | مقدار پیش‌فرض `kimi-k2.6` است                                             |

  </Step>
</Steps>

پیکربندی در `plugins.entries.moonshot.config.webSearch` قرار دارد:

```json5
{
  plugins: {
    entries: {
      moonshot: {
        config: {
          webSearch: {
            apiKey: "sk-...", // یا از KIMI_API_KEY / MOONSHOT_API_KEY استفاده کنید
            baseUrl: "https://api.moonshot.ai/v1",
            model: "kimi-k2.6",
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "kimi",
      },
    },
  },
}
```

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="حالت تفکر بومی">
    Kimi K3 در API مربوط به Moonshot همیشه با حداکثر تلاش استدلال می‌کند. OpenClaw فقط
    `/think max` را در دسترس قرار می‌دهد، `reasoning_effort: "max"` را ارسال می‌کند و تنظیمات منسوخ سطح پایین‌تر یا
    `off` را نادیده می‌گیرد.

    Kimi Code K3 ‏`/think off|max` را ارائه می‌کند. نقطه پایانی سازگار با Anthropic آن
    برای حالت خاموش، `thinking.type: "disabled"` یا برای تفکر تطبیقی با
    `output_config.effort: "max"` برای حداکثر را دریافت می‌کند. این مورد هم برای `kimi/k3` و هم
    `kimi/k3[1m]` اعمال می‌شود.
    API ‏Moonshot K3 از `auto`، ‏`none`، ‏`required` و انتخاب‌های ثابت ابزار پشتیبانی می‌کند،
    بنابراین OpenClaw مقدار درخواستی `tool_choice` را حفظ می‌کند. برای استفاده چندمرحله‌ای از ابزار،
    OpenClaw محتوای استدلال دستیار را که قرارداد بازپخش Moonshot
    به آن نیاز دارد، حفظ می‌کند.

    Kimi K2.7 Code همیشه از تفکر بومی استفاده می‌کند. Moonshot از کلاینت‌ها می‌خواهد
    فیلد `thinking` را برای این مدل حذف کنند؛ بنابراین OpenClaw فقط `on` را ارائه می‌کند و
    تنظیمات منسوخ `off` را نادیده می‌گیرد. K2.7 همچنین `temperature`، ‏`top_p`، ‏`n`،
    `presence_penalty` و `frequency_penalty` را ثابت می‌کند؛ OpenClaw مقادیر بازنویسی پیکربندی‌شده
    آن فیلدها را حذف می‌کند.

    سایر مدل‌های Kimi در Moonshot از تفکر بومی دودویی پشتیبانی می‌کنند:

    - `thinking: { type: "enabled" }`
    - `thinking: { type: "disabled" }`

    آن را برای هر مدل از طریق `agents.defaults.models.<provider/model>.params` پیکربندی کنید:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "disabled" },
              },
            },
          },
        },
      },
    }
    ```

    OpenClaw سطوح زمان اجرای `/think` را برای آن مدل‌ها نگاشت می‌کند:

    | سطح `/think`       | رفتار Moonshot          |
    | -------------------- | -------------------------- |
    | `/think off`         | `thinking.type=disabled`   |
    | هر سطحی غیر از خاموش    | `thinking.type=enabled`    |

    <Warning>
    وقتی تفکر Moonshot K2 فعال است، `tool_choice` باید `auto` یا `none` باشد. انتخاب ثابت ابزار (`type: "tool"` یا `type: "function"`) در عوض تفکر را به `disabled` بازمی‌گرداند تا ابزار درخواستی همچنان اجرا شود؛ `tool_choice: "required"` نیز در عوض به `auto` نرمال‌سازی می‌شود. Kimi K2.7 Code نمی‌تواند تفکر را غیرفعال کند، بنابراین `tool_choice` ناسازگار آن به `auto` نرمال‌سازی می‌شود. Kimi K3 از قرارداد جداگانه تلاش استدلال خود استفاده می‌کند و انتخاب‌های ابزار پشتیبانی‌شده را حفظ می‌کند.
    </Warning>

    Kimi K2.6 همچنین فیلد اختیاری `thinking.keep` را می‌پذیرد که نگهداشت چندمرحله‌ای
    `reasoning_content` را کنترل می‌کند. برای حفظ استدلال کامل در مراحل مختلف، آن را روی `"all"` تنظیم کنید؛ برای استفاده از راهبرد
    پیش‌فرض سرور، آن را حذف کنید (یا روی `null` باقی بگذارید).
    OpenClaw فقط `thinking.keep` را برای
    `moonshot/kimi-k2.6` ارسال می‌کند و آن را از سایر مدل‌ها حذف می‌کند. Kimi K2.7 Code
    به‌طور پیش‌فرض تاریخچه کامل استدلال را حفظ می‌کند، درحالی‌که OpenClaw کل
    فیلد `thinking` را حذف می‌کند.

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "moonshot/kimi-k2.6": {
              params: {
                thinking: { type: "enabled", keep: "all" },
              },
            },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="پاک‌سازی شناسه فراخوانی ابزار">
    Moonshot Kimi شناسه‌های بومی tool_call را با ساختاری مانند `functions.<name>:<index>` ارائه می‌کند. OpenClaw نخستین رخداد هر شناسه بومی Kimi را حفظ می‌کند و موارد تکراری بعدی را به شناسه‌های قطعی و به‌سبک OpenAI، یعنی `call_*`، بازنویسی می‌کند. نتایج ابزار متناظر با همان شناسه دوباره نگاشت می‌شوند تا بازپخش بدون حذف نخستین شناسه بومی Kimi، یکتا باقی بماند. این رفتار در ارائه‌دهنده همراه Moonshot تعبیه شده است و تنظیمی قابل‌پیکربندی برای کاربر نیست.
  </Accordion>

  <Accordion title="سازگاری مصرف در استریم">
    نقاط پایانی بومی Moonshot ‏(`https://api.moonshot.ai/v1` و
    `https://api.moonshot.cn/v1`) سازگاری مصرف در استریم را اعلام می‌کنند.
    OpenClaw این قابلیت را بر اساس میزبان نقطه پایانی، نه شناسه ارائه‌دهنده، تعیین می‌کند؛ بنابراین شناسه ارائه‌دهنده سفارشی که به همان
    میزبان بومی Moonshot اشاره کند، همان رفتار مصرف در استریم را به ارث می‌برد.

    با قیمت‌گذاری K2.6 در کاتالوگ، مصرف استریم‌شده‌ای که شامل توکن‌های ورودی، خروجی
    و خواندن کش باشد، برای
    `/status`، ‏`/usage full`، ‏`/usage cost` و محاسبه نشست مبتنی بر رونوشت
    نیز به هزینه تخمینی محلی بر حسب دلار آمریکا تبدیل می‌شود.

  </Accordion>

  <Accordion title="مرجع نقطه پایانی و ارجاع مدل">
    | ارائه‌دهنده   | پیشوند ارجاع مدل | نقطه پایانی                      | متغیر محیطی احراز هویت        |
    | ---------- | ---------------- | ------------------------------ | ------------------- |
    | Moonshot   | `moonshot/`      | `https://api.moonshot.ai/v1`  | `MOONSHOT_API_KEY`  |
    | Moonshot CN| `moonshot/`      | `https://api.moonshot.cn/v1`  | `MOONSHOT_API_KEY`  |
    | Kimi Coding| `kimi/`          | نقطه پایانی Kimi Coding           | `KIMI_API_KEY`      |
    | جست‌وجوی وب | نامرتبط              | همان منطقه API ‏Moonshot    | `KIMI_API_KEY` یا `MOONSHOT_API_KEY` |

    - جست‌وجوی وب Kimi از `KIMI_API_KEY` یا `MOONSHOT_API_KEY` استفاده می‌کند و مقدار پیش‌فرض آن `https://api.moonshot.ai/v1` با مدل `kimi-k2.6` است.
    - در صورت نیاز، قیمت‌گذاری و فراداده زمینه را در `models.providers` بازنویسی کنید.
    - اگر Moonshot محدودیت‌های زمینه متفاوتی برای یک مدل منتشر کرد، `contextWindow` را متناسب با آن تنظیم کنید.

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="جست‌وجوی وب" href="/fa/tools/web" icon="magnifying-glass">
    پیکربندی ارائه‌دهندگان جست‌وجوی وب، از جمله Kimi.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/configuration-reference" icon="gear">
    طرح‌واره کامل پیکربندی برای ارائه‌دهندگان، مدل‌ها و Pluginها.
  </Card>
  <Card title="پلتفرم باز Moonshot" href="https://platform.moonshot.ai" icon="globe">
    مدیریت کلید API ‏Moonshot و مستندات آن.
  </Card>
</CardGroup>
