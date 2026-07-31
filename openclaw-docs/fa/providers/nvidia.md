---
read_when:
    - می‌خواهید از مدل‌های متن‌باز در OpenClaw به‌صورت رایگان استفاده کنید
    - باید NVIDIA_API_KEY را تنظیم کنید
    - می‌خواهید از Nemotron 3 Ultra از طریق NVIDIA استفاده کنید
summary: استفاده از API سازگار با OpenAI شرکت NVIDIA در OpenClaw
title: NVIDIA
x-i18n:
    generated_at: "2026-07-27T14:35:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b5ac7bcc19400a661b2f2861a1dd4d2306c94e445783929e342e9184003314e9
    source_path: providers/nvidia.md
    workflow: 16
---

NVIDIA مدل‌های باز را به‌صورت رایگان از طریق یک API سازگار با OpenAI در
`https://integrate.api.nvidia.com/v1` ارائه می‌کند که احراز هویت آن با یک کلید API از
[build.nvidia.com](https://build.nvidia.com/settings/api-keys) انجام می‌شود. OpenClaw
ارائه‌دهنده NVIDIA را به‌طور پیش‌فرض روی Nemotron 3 Ultra تنظیم می‌کند؛ مدل استدلالی NVIDIA با مجموع 550B پارامتر / 55B پارامتر
فعال برای کارهای عامل‌محور با زمینهٔ طولانی.

## شروع به کار

<Steps>
  <Step title="دریافت کلید API">
    یک کلید API در [build.nvidia.com](https://build.nvidia.com/settings/api-keys) ایجاد کنید.
  </Step>
  <Step title="صدور کلید و اجرای راه‌اندازی اولیه">
    ```bash
    export NVIDIA_API_KEY="nvapi-..."
    openclaw onboard --auth-choice nvidia-api-key
    ```
  </Step>
  <Step title="تنظیم یک مدل NVIDIA">
    ```bash
    openclaw models set nvidia/nvidia/nemotron-3-ultra-550b-a55b
    ```
  </Step>
</Steps>

برای راه‌اندازی غیرتعاملی، کلید را مستقیماً ارسال کنید:

```bash
openclaw onboard --auth-choice nvidia-api-key --nvidia-api-key "nvapi-..."
```

<Warning>
`--nvidia-api-key` کلید را در تاریخچهٔ پوسته و خروجی `ps` ثبت می‌کند. در صورت امکان، متغیر محیطی
`NVIDIA_API_KEY` را ترجیح دهید.
</Warning>

## نمونهٔ پیکربندی

```json5
{
  env: { NVIDIA_API_KEY: "nvapi-..." },
  models: {
    providers: {
      nvidia: {
        baseUrl: "https://integrate.api.nvidia.com/v1",
        api: "openai-completions",
      },
    },
  },
  agents: {
    defaults: {
      model: { primary: "nvidia/nvidia/nemotron-3-ultra-550b-a55b" },
    },
  },
}
```

## کاتالوگ منتخب

وقتی یک کلید API برای NVIDIA پیکربندی شده باشد، مسیرهای راه‌اندازی و انتخاب مدل،
کاتالوگ عمومی مدل‌های منتخب NVIDIA را از
`https://assets.ngc.nvidia.com/products/api-catalog/featured-models.json` دریافت می‌کنند و
نتیجه را برای 24 ساعت در حافظهٔ نهان نگه می‌دارند (32 ورودی نخست که به‌صورت ردیف‌های ورودی متنی رایگان
وارد می‌شوند). بنابراین مدل‌های منتخب جدید از build.nvidia.com بدون انتظار برای انتشار نسخه‌ای از OpenClaw، در سطوح راه‌اندازی و
انتخاب مدل ظاهر می‌شوند. هنگامی که خوراک زنده در دسترس باشد، نخستین مدل بازگردانده‌شده
گزینهٔ ازپیش‌انتخاب‌شده هنگام راه‌اندازی NVIDIA است.

دریافت داده برای `assets.ngc.nvidia.com` از یک سیاست ثابت میزبان HTTPS استفاده می‌کند. اگر هیچ
کلید API برای NVIDIA پیکربندی نشده باشد، یا خوراک در دسترس نباشد یا ساختاری نامعتبر داشته باشد،
OpenClaw از کاتالوگ همراه و پیش‌فرض همراه زیر استفاده می‌کند.

## Nemotron 3 Ultra

Nemotron 3 Ultra مدل پیش‌فرض NVIDIA در OpenClaw است. صفحهٔ ساخت NVIDIA برای
[`nvidia/nemotron-3-ultra-550b-a55b`](https://build.nvidia.com/nvidia/nemotron-3-ultra-550b-a55b)
آن را به‌عنوان یک نقطهٔ پایانی رایگان و در دسترس با مشخصات زمینهٔ 1M توکنی فهرست می‌کند.

ردیف همراه Ultra به‌طور پیش‌فرض
`chat_template_kwargs: { enable_thinking: false, force_nonempty_content: true }`
را ارسال می‌کند تا خروجی عادی گفت‌وگو به‌جای نمایش متن استدلال،
در پاسخ قابل‌مشاهده باقی بماند.

برای استفاده از توانمندترین گزینهٔ پیش‌فرض NVIDIA، از Ultra استفاده کنید. هنگامی که
گزینهٔ کوچک‌تر Nemotron 3 را می‌خواهید، Super را انتخاب‌شده نگه دارید؛ یا یکی از مدل‌های شخص ثالث
میزبانی‌شده در کاتالوگ NVIDIA را انتخاب کنید، اگر زمینه، تأخیر یا رفتار آن‌ها مناسب‌تر است.

## کاتالوگ جایگزین همراه

ردیف‌های همراه قابل‌انتخاب، تصویری لحظه‌ای از کاتالوگ مدل‌های منتخب NVIDIA هستند. ردیف‌های سازگاری
منسوخ‌شده با ارجاع دقیق همچنان قابل‌شناسایی‌اند، اما در انتخابگرهای مدل
نمایش داده نمی‌شوند.

| ارجاع مدل                                  | نام                  | زمینه   | حداکثر خروجی |
| ------------------------------------------ | --------------------- | --------- | ---------- |
| `nvidia/nvidia/nemotron-3-ultra-550b-a55b` | Nemotron 3 Ultra 550B | 1,048,576 | 8,192      |
| `nvidia/nvidia/nemotron-3-super-120b-a12b` | Nemotron 3 Super 120B | 1,000,000 | 8,192      |
| `nvidia/z-ai/glm-5.2`                      | GLM 5.2               | 202,752   | 8,192      |
| `nvidia/moonshotai/kimi-k2.6`              | Kimi K2.6             | 262,144   | 8,192      |
| `nvidia/minimaxai/minimax-m3`              | Minimax M3            | 196,608   | 8,192      |
| `nvidia/deepseek-ai/deepseek-v4-pro`       | DeepSeek V4 Pro       | 262,144   | 16,384     |
| `nvidia/qwen/qwen3.5-397b-a17b`            | Qwen3.5 397B A17B     | 262,144   | 16,384     |

کاتالوگ کامل سازگاری همچنین این ارجاع‌های منتشرشده را برای پیکربندی‌های موجود
حفظ می‌کند: `nvidia/moonshotai/kimi-k2.5`، `nvidia/z-ai/glm-5.1`،
`nvidia/minimaxai/minimax-m2.5`، `nvidia/z-ai/glm5` و
`nvidia/minimaxai/minimax-m2.7`. آن‌ها با ارجاع دقیق همچنان در دسترس‌اند، اما
هرگز در راه‌اندازی اولیه یا انتخابگرهای مدل ظاهر نمی‌شوند.

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="رفتار فعال‌سازی خودکار">
    هنگامی که متغیر محیطی `NVIDIA_API_KEY` تنظیم شده باشد
    یا کلیدی هنگام راه‌اندازی اولیه ذخیره شده باشد، ارائه‌دهنده به‌طور خودکار فعال می‌شود. به‌جز کلید،
    هیچ پیکربندی صریحی برای ارائه‌دهنده لازم نیست.
  </Accordion>

  <Accordion title="کاتالوگ و قیمت‌گذاری">
    هنگامی که احراز هویت NVIDIA پیکربندی شده باشد، OpenClaw کاتالوگ عمومی مدل‌های منتخب NVIDIA را
    ترجیح می‌دهد و آن را برای 24 ساعت در حافظهٔ نهان نگه می‌دارد. جایگزین همراه و قابل‌انتخاب،
    تصویری ایستا از کاتالوگ مدل‌های منتخب NVIDIA است؛ ردیف‌های سازگاری منسوخ‌شده با ارجاع دقیق
    از انتخابگرهای مدل پنهان‌اند. هزینه‌ها در منبع به‌طور پیش‌فرض `0` هستند،
    زیرا NVIDIA در حال حاضر دسترسی رایگان API را برای مدل‌های فهرست‌شده ارائه می‌کند.
  </Accordion>

  <Accordion title="نقطهٔ پایانی سازگار با OpenAI">
    OpenClaw از طریق آداپتور `openai-completions` و مسیر استاندارد تکمیل گفت‌وگوی
    `/v1` با NVIDIA ارتباط برقرار می‌کند. هر ابزار سازگار با OpenAI باید
    با URL پایهٔ NVIDIA بدون نیاز به پیکربندی اضافی کار کند.
  </Accordion>

  <Accordion title="پارامترهای استدلال Nemotron 3 Ultra">
    درخواست نمونهٔ Ultra از NVIDIA برای خروجی استدلال از `chat_template_kwargs.enable_thinking`
    و `reasoning_budget` استفاده می‌کند. ردیف همراه Ultra در OpenClaw
    تفکر مبتنی بر الگو را به‌طور پیش‌فرض برای استفادهٔ عادی در گفت‌وگو غیرفعال می‌کند. اگر لازم است
    خروجی استدلال NVIDIA را فعال کنید یا فیلدهای دیگر مخصوص درخواست NVIDIA را
    اجباری کنید، پارامترها را برای هر مدل تنظیم کنید و بازنویسی‌های مخصوص ارائه‌دهنده را به
    مدل NVIDIA محدود نگه دارید:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "nvidia/nvidia/nemotron-3-ultra-550b-a55b": {
              params: {
                chat_template_kwargs: { enable_thinking: true },
                extra_body: { reasoning_budget: 16384 },
              },
            },
          },
        },
      },
    }
    ```

    `params.chat_template_kwargs` به هر `chat_template_kwargs`
    موجود در درخواست ادغام می‌شود، نه اینکه کل شیء را جایگزین کند.
    `params.extra_body` بازنویسی نهایی بدنهٔ درخواست سازگار با OpenAI است
    و کلیدهای متداخل محموله را بازنویسی می‌کند؛ بنابراین فقط برای فیلدهایی از آن استفاده کنید که NVIDIA
    برای نقطهٔ پایانی انتخاب‌شده مستند کرده است.

  </Accordion>

  <Accordion title="پاسخ‌های کند ارائه‌دهندهٔ سفارشی">
    برخی مدل‌های سفارشی میزبانی‌شده توسط NVIDIA ممکن است پیش از انتشار نخستین قطعهٔ پاسخ،
    بیشتر از زمان پیش‌فرض حدود 120s نگهبان بی‌کاری مدل طول بکشند. برای ورودی‌های سفارشی
    ارائه‌دهندهٔ NVIDIA، به‌جای مهلت زمانی کل زمان اجرای عامل، مهلت زمانی ارائه‌دهنده را افزایش دهید؛
    `timeoutSeconds` درخواست‌های HTTP ارائه‌دهنده را پوشش می‌دهد و
    سقف نگهبان بی‌کاری/جریان را برای آن ارائه‌دهنده افزایش می‌دهد:

    ```json5
    {
      models: {
        providers: {
          "custom-integrate-api-nvidia-com": {
            baseUrl: "https://integrate.api.nvidia.com/v1",
            api: "openai-completions",
            apiKey: "NVIDIA_API_KEY",
            timeoutSeconds: 300,
          },
        },
      },
      agents: {
        defaults: {
          models: {
            "custom-integrate-api-nvidia-com/meta/llama-3.1-70b-instruct": {
              params: { thinking: "off" },
            },
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

<Tip>
استفاده از مدل‌های NVIDIA در حال حاضر رایگان است. برای اطلاع از آخرین وضعیت دسترس‌پذیری و
جزئیات محدودیت نرخ، [build.nvidia.com](https://build.nvidia.com/) را بررسی کنید.
</Tip>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار تغییر مسیر هنگام خرابی.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/configuration-reference" icon="gear">
    مرجع کامل پیکربندی عامل‌ها، مدل‌ها و ارائه‌دهندگان.
  </Card>
</CardGroup>
