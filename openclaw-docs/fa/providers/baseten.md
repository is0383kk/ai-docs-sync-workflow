---
read_when:
    - می‌خواهید Inkling متعلق به Thinking Machines Lab را در OpenClaw اجرا کنید
    - یک API سازگار با OpenAI برای مدل‌های میزبانی‌شدهٔ Baseten می‌خواهید
summary: راه‌اندازی Baseten برای Inkling و APIهای میزبانی‌شدهٔ مدل
title: Baseten
x-i18n:
    generated_at: "2026-07-27T17:02:29Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2ccc3b5cf64b01859f9f022d7bc15a69a1cb42c87d4f914c118276c1151020de
    source_path: providers/baseten.md
    workflow: 16
---

[APIهای مدل Baseten](https://docs.baseten.co/inference/model-apis/overview) دسترسی میزبانی‌شده و سازگار با OpenAI به مدل‌های پیشرو را فراهم می‌کنند. Plugin خارجی رسمی از کشف احراز هویت‌شده استفاده می‌کند، بنابراین OpenClaw مجموعه کامل مدل‌های فعال‌شده برای حساب Baseten شما را دنبال می‌کند. جایگزین آفلاین آن شامل تمام APIهای مدلی است که هنگام ساخت این نسخه OpenClaw در دسترس بودند.

| ویژگی            | مقدار                                                    |
| --------------- | -------------------------------------------------------- |
| شناسه ارائه‌دهنده | `baseten`                                                |
| Plugin          | بسته خارجی رسمی (`@openclaw/baseten-provider`) |
| متغیر محیطی احراز هویت | `BASETEN_API_KEY`                                        |
| پرچم راه‌اندازی اولیه | `--auth-choice baseten-api-key`                          |
| پرچم مستقیم CLI | `--baseten-api-key <key>`                                |
| API             | سازگار با OpenAI (`openai-completions`)                 |
| نشانی پایه        | `https://inference.baseten.co/v1`                        |
| مدل پیش‌فرض       | `baseten/thinkingmachines/inkling`                       |

## نصب Plugin

```bash
openclaw plugins install @openclaw/baseten-provider
openclaw gateway restart
```

## شروع به کار

<Steps>
  <Step title="ایجاد حساب Baseten و کلید API">
    طرح Basic در Baseten هزینه ماهانه پلتفرم ندارد؛ هزینه فراخوانی‌های API مدل بر اساس میزان استفاده محاسبه می‌شود. در [تنظیمات کلید API در Baseten](https://app.baseten.co/settings/api_keys) یک کلید ایجاد کنید و نرخ‌های فعلی را در [صفحه قیمت‌گذاری](https://www.baseten.co/pricing) بررسی کنید.
  </Step>
  <Step title="اجرای راه‌اندازی اولیه">
    <CodeGroup>

```bash Onboarding
openclaw onboard --auth-choice baseten-api-key
```

```bash Direct flag
openclaw onboard --non-interactive \
  --auth-choice baseten-api-key \
  --baseten-api-key "$BASETEN_API_KEY"
```

```bash Env only
export BASETEN_API_KEY=...
```

    </CodeGroup>

  </Step>
  <Step title="تأیید کاتالوگ زنده">
    ```bash
    openclaw models list --provider baseten
    ```

    با احراز هویت قابل‌استفاده، Plugin از `GET /v1/models` درخواست می‌کند و همه مدل‌های بازگردانده‌شده برای حساب را فهرست می‌کند. بدون احراز هویت، آفلاین می‌ماند و از جایگزین همراه بسته استفاده می‌کند.

  </Step>
</Steps>

## Inkling

[Inkling از Thinking Machines Lab](https://thinkingmachines.ai/news/introducing-inkling/) مدل پیش‌فرض است. این مدل در OpenClaw از ورودی متن و تصویر، فراخوانی ابزار، شِماهای ساخت‌یافته ابزار، میزان تلاش استدلال قابل‌تنظیم، پنجره زمینه 1.048M توکنی و حداکثر 32k توکن خروجی پشتیبانی می‌کند:

```json5
{
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
}
```

برای تغییر مدل در یک گفت‌وگوی موجود از `/model baseten/thinkingmachines/inkling` استفاده کنید.

## کاتالوگ جایگزین همراه بسته

کاتالوگ زنده احراز هویت‌شده مرجع قطعی است. این ردیف‌ها پیش از موفقیت کشف، راه‌اندازی و انتخاب مدل را کاربردی نگه می‌دارند:

| ارجاع مدل                                          | ورودی       | زمینه | حداکثر خروجی |
| -------------------------------------------------- | ----------- | ------: | ---------: |
| `baseten/deepseek-ai/DeepSeek-V4-Pro`              | متن        |    262k |       262k |
| `baseten/zai-org/GLM-4.7`                          | متن        |    200k |       200k |
| `baseten/zai-org/GLM-5`                            | متن        |    202k |       202k |
| `baseten/zai-org/GLM-5.1`                          | متن        |    202k |       202k |
| `baseten/zai-org/GLM-5.2`                          | متن        |    202k |       202k |
| `baseten/thinkingmachines/inkling`                 | متن، تصویر |  1.048M |        32k |
| `baseten/moonshotai/Kimi-K2.5`                     | متن، تصویر |    262k |       262k |
| `baseten/moonshotai/Kimi-K2.6`                     | متن، تصویر |    262k |       262k |
| `baseten/moonshotai/Kimi-K2.7-Code`                | متن، تصویر |    262k |       262k |
| `baseten/nvidia/Nemotron-120B-A12B`                | متن        |    202k |       202k |
| `baseten/nvidia/NVIDIA-Nemotron-3-Ultra-550B-A55B` | متن        |    202k |       202k |
| `baseten/openai/gpt-oss-120b`                      | متن        |    128k |       128k |

همه مدل‌های همراه بسته از فراخوانی ابزار و استدلال پشتیبانی می‌کنند. OpenClaw سطوح تفکر خود را به مدل‌های دارای `reasoning_effort` بومی نگاشت می‌کند. مدل‌های اختیاری GLM، Kimi و Nemotron در Baseten به‌طور پیش‌فرض تفکر را غیرفعال می‌کنند؛ بیشتر آن‌ها کنترل دودویی خاموش/روشن ارائه می‌دهند، درحالی‌که GLM 5.2 گزینه‌های خاموش، زیاد و حداکثر را ارائه می‌کند. OpenClaw این انتخاب‌ها را از طریق کنترل `chat_template_args.enable_thinking` در Baseten ارسال می‌کند و برای GLM 5.2 نیز پارامتر سطح‌بالای اعتبارسنجی‌شده `reasoning_effort` را می‌فرستد.

<Note>
Baseten می‌تواند مستقل از انتشارهای OpenClaw، APIهای مدل را اضافه، حذف یا تغییر دهد. Plugin شناسه‌های مدل، محدودیت‌های زمینه، محدودیت‌های خروجی و قیمت‌گذاری ورودی، ورودی ذخیره‌شده در حافظه نهان و خروجی را از API احراز هویت‌شده تازه‌سازی می‌کند و در عین حال سیاست انتقال مختص هر مدل در OpenClaw را حفظ می‌کند.
</Note>

## پیکربندی دستی

بیشتر راه‌اندازی‌ها فقط به کلید API نیاز دارند. برای تثبیت صریح ارائه‌دهنده:

```json5
{
  env: { BASETEN_API_KEY: "..." },
  agents: {
    defaults: {
      model: { primary: "baseten/thinkingmachines/inkling" },
    },
  },
  models: {
    mode: "merge",
    providers: {
      baseten: {
        baseUrl: "https://inference.baseten.co/v1",
        apiKey: "${BASETEN_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "thinkingmachines/inkling",
            name: "Inkling",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<Note>
اگر Gateway به‌صورت سرویس پس‌زمینه (launchd، systemd، Docker) اجرا می‌شود، مطمئن شوید `BASETEN_API_KEY` برای آن فرایند در دسترس است. کلیدی که فقط در یک پوسته تعاملی صادر شده باشد، برای سرویس مدیریت‌شده‌ای که از قبل در حال اجرا است قابل‌مشاهده نیست.
</Note>

## مرتبط

<CardGroup cols={2}>
  <Card title="ارائه‌دهندگان مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="حالت‌های تفکر" href="/fa/tools/thinking" icon="brain">
    سطوح تلاش استدلال OpenClaw را انتخاب کنید.
  </Card>
  <Card title="CLI مدل‌ها" href="/fa/cli/models" icon="terminal">
    مدل‌های کشف‌شده را فهرست، بررسی و انتخاب کنید.
  </Card>
  <Card title="پرسش‌های متداول مدل‌ها" href="/fa/help/faq-models" icon="circle-question">
    عیب‌یابی نمایه‌های احراز هویت و انتخاب مدل.
  </Card>
</CardGroup>
