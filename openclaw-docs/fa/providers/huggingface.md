---
read_when:
    - می‌خواهید از Hugging Face Inference با OpenClaw استفاده کنید
    - به متغیر محیطی توکن HF یا گزینه احراز هویت CLI نیاز دارید
summary: راه‌اندازی Hugging Face Inference (احراز هویت + انتخاب مدل)
title: Hugging Face (استنتاج)
x-i18n:
    generated_at: "2026-07-27T14:34:52Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 92c400b78c5ad2cc724ad4029560dccc5bc2006fdeae400fc6b58998e727e17c
    source_path: providers/huggingface.md
    workflow: 16
---

[ارائه‌دهندگان استنتاج Hugging Face](https://huggingface.co/docs/inference-providers) یک مسیریاب تکمیل‌های گفت‌وگوی سازگار با OpenAI را برای مدل‌های میزبانی‌شدهٔ متعدد (DeepSeek، Llama و مدل‌های دیگر) با یک توکن ارائه می‌کند. OpenClaw فقط با **نقطهٔ پایانی تکمیل‌های گفت‌وگو** ارتباط برقرار می‌کند؛ برای تبدیل متن به تصویر، تعبیه‌ها یا گفتار، مستقیماً از [کلاینت‌های استنتاج HF](https://huggingface.co/docs/api-inference/quicktour) استفاده کنید.

| ویژگی     | مقدار                                                                                                                       |
| ------------ | --------------------------------------------------------------------------------------------------------------------------- |
| شناسهٔ ارائه‌دهنده  | `huggingface`                                                                                                               |
| Plugin       | همراه برنامه (به‌طور پیش‌فرض فعال، بدون مرحلهٔ نصب)                                                                               |
| متغیر محیطی احراز هویت | `HUGGINGFACE_HUB_TOKEN` یا `HF_TOKEN` (توکن ریزدانه)                                                                  |
| API          | سازگار با OpenAI (`https://router.huggingface.co/v1`)                                                                      |
| صورت‌حساب      | یک توکن HF؛ [قیمت‌گذاری](https://huggingface.co/docs/inference-providers/pricing) از نرخ‌های ارائه‌دهنده با یک سطح رایگان پیروی می‌کند |

## شروع به کار

<Steps>
  <Step title="ایجاد توکن ریزدانه">
    به [Hugging Face Settings Tokens](https://huggingface.co/settings/tokens/new?ownUserPermissions=inference.serverless.write&tokenType=fineGrained) بروید و یک توکن ریزدانهٔ جدید ایجاد کنید.

    <Warning>
    مجوز **Make calls to Inference Providers** باید برای توکن فعال باشد؛ در غیر این صورت، درخواست‌های API رد می‌شوند.
    </Warning>

  </Step>
  <Step title="اجرای راه‌اندازی اولیه">
    در فهرست کشویی ارائه‌دهنده، **Hugging Face** را انتخاب کنید و سپس هنگام درخواست، کلید API خود را وارد کنید:

    ```bash
    openclaw onboard --auth-choice huggingface-api-key
    ```

  </Step>
  <Step title="انتخاب مدل پیش‌فرض">
    در فهرست کشویی **Default Hugging Face model** یک مدل انتخاب کنید. وقتی توکن معتبر باشد، فهرست از Inference API بارگیری می‌شود؛ در غیر این صورت، OpenClaw کاتالوگ داخلی زیر را نمایش می‌دهد. انتخاب شما به‌صورت `agents.defaults.model.primary` ذخیره می‌شود:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
        },
      },
    }
    ```

  </Step>
  <Step title="بررسی دردسترس‌بودن مدل">
    ```bash
    openclaw models list --provider huggingface
    ```
  </Step>
</Steps>

### راه‌اندازی غیرتعاملی

```bash
openclaw onboard --non-interactive \
  --mode local \
  --auth-choice huggingface-api-key \
  --huggingface-api-key "$HF_TOKEN"
```

مدل `huggingface/deepseek-ai/DeepSeek-R1` را به‌عنوان مدل پیش‌فرض تنظیم می‌کند.

## شناسه‌های مدل

ارجاع‌های مدل از قالب `huggingface/<org>/<model>` (شناسه‌های سبک Hub) استفاده می‌کنند. کاتالوگ داخلی OpenClaw:

| مدل         | ارجاع (با پیشوند `huggingface/`) |
| ------------- | -------------------------------- |
| DeepSeek R1   | `deepseek-ai/DeepSeek-R1`        |
| DeepSeek V3.1 | `deepseek-ai/DeepSeek-V3.1`      |
| GPT-OSS 120B  | `openai/gpt-oss-120b`            |

<Tip>
وقتی توکن معتبر باشد، OpenClaw هنگام راه‌اندازی اولیه و شروع Gateway، هر مدل دیگری را نیز از **GET** `https://router.huggingface.co/v1/models` کشف می‌کند؛ بنابراین کاتالوگ شما می‌تواند بسیار بیشتر از سه مدل بالا را در بر بگیرد. می‌توانید `:fastest` یا `:cheapest` را به هر شناسهٔ مدل بیفزایید؛ مسیریاب HF درخواست را به ارائه‌دهندهٔ استنتاج منطبق هدایت می‌کند. ترتیب پیش‌فرض ارائه‌دهندگان را در [تنظیمات ارائه‌دهندهٔ استنتاج](https://hf.co/settings/inference-providers) تنظیم کنید.
</Tip>

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="کشف مدل و فهرست کشویی راه‌اندازی اولیه">
    OpenClaw مدل‌ها را به‌شکل زیر کشف می‌کند:

    ```bash
    GET https://router.huggingface.co/v1/models
    Authorization: Bearer $HUGGINGFACE_HUB_TOKEN   # یا $HF_TOKEN
    ```

    پاسخ به سبک OpenAI است: `{ "object": "list", "data": [ { "id": "Qwen/Qwen3-8B", "owned_by": "Qwen", ... }, ... ] }`.

    با یک کلید پیکربندی‌شده (راه‌اندازی اولیه، `HUGGINGFACE_HUB_TOKEN` یا `HF_TOKEN`) فهرست کشویی **Default Hugging Face model** هنگام راه‌اندازی تعاملی از این نقطهٔ پایانی پر می‌شود. هنگام شروع Gateway، همان فراخوانی برای تازه‌سازی کاتالوگ تکرار می‌شود. مدل‌های کشف‌شده با کاتالوگ داخلی بالا ادغام می‌شوند (که هنگام تطابق شناسه برای فراداده‌هایی مانند پنجرهٔ زمینه و هزینه استفاده می‌شود). اگر درخواست ناموفق باشد، داده‌ای برنگرداند یا هیچ کلیدی تنظیم نشده باشد، OpenClaw فقط به کاتالوگ داخلی بازمی‌گردد.

    غیرفعال‌کردن کشف بدون حذف ارائه‌دهنده:

    ```bash
    openclaw config set plugins.entries.huggingface.config.discovery.enabled false
    ```

  </Accordion>

  <Accordion title="نام‌های مدل، نام‌های مستعار و پسوندهای سیاست">
    - **نام از API:** مدل‌های کشف‌شده در صورت وجود، از `name`، `title` یا `display_name` متعلق به API استفاده می‌کنند؛ در غیر این صورت، OpenClaw نامی را از شناسهٔ مدل استخراج می‌کند (برای نمونه، `deepseek-ai/DeepSeek-R1` به «DeepSeek R1» تبدیل می‌شود).
    - **بازنویسی نام نمایشی:** برای هر مدل یک برچسب سفارشی در پیکربندی تنظیم کنید:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1 (fast)" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheap)" },
          },
        },
      },
    }
    ```

    - **پسوندهای سیاست:** `:fastest` و `:cheapest` قراردادهای مسیریاب HF هستند و OpenClaw آن‌ها را بازنویسی نمی‌کند: پسوند عیناً به‌عنوان بخشی از شناسهٔ مدل ارسال می‌شود و مسیریاب HF ارائه‌دهندهٔ استنتاج منطبق را انتخاب می‌کند. اگر برای هر پسوند نام مستعار متمایزی می‌خواهید، هر گونه را به‌عنوان ورودی جداگانه‌ای زیر `models.providers.huggingface.models` (یا در `model.primary`) اضافه کنید.
    - **ادغام پیکربندی:** ورودی‌های موجود در `models.providers.huggingface.models` (برای نمونه، در `models.json`) هنگام ادغام پیکربندی حفظ می‌شوند؛ بنابراین هر `name`، `alias` یا گزینهٔ مدلی که در آنجا تنظیم کرده‌اید، پس از راه‌اندازی‌های مجدد باقی می‌ماند.

  </Accordion>

  <Accordion title="راه‌اندازی محیط و سرویس پس‌زمینه">
    اگر Gateway به‌صورت سرویس پس‌زمینه (launchd/systemd) اجرا می‌شود، مطمئن شوید `HUGGINGFACE_HUB_TOKEN` یا `HF_TOKEN` برای آن فرایند دردسترس است (برای نمونه، در `~/.openclaw/.env` یا از طریق `env.shellEnv`).

    <Note>
    OpenClaw هر دو `HUGGINGFACE_HUB_TOKEN` و `HF_TOKEN` را می‌پذیرد. اگر هر دو تنظیم شده باشند، `HUGGINGFACE_HUB_TOKEN` اولویت دارد.
    </Note>

  </Accordion>

  <Accordion title="پیکربندی: DeepSeek R1 با مدل جایگزین">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-R1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="پیکربندی: DeepSeek با گونه‌های ارزان‌ترین و سریع‌ترین">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "huggingface/deepseek-ai/DeepSeek-R1" },
          models: {
            "huggingface/deepseek-ai/DeepSeek-R1": { alias: "DeepSeek R1" },
            "huggingface/deepseek-ai/DeepSeek-R1:cheapest": { alias: "DeepSeek R1 (cheapest)" },
            "huggingface/deepseek-ai/DeepSeek-R1:fastest": { alias: "DeepSeek R1 (fastest)" },
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="پیکربندی: DeepSeek و GPT-OSS با نام‌های مستعار">
    ```json5
    {
      agents: {
        defaults: {
          model: {
            primary: "huggingface/deepseek-ai/DeepSeek-V3.1",
            fallbacks: ["huggingface/openai/gpt-oss-120b"],
          },
          models: {
            "huggingface/deepseek-ai/DeepSeek-V3.1": { alias: "DeepSeek V3.1" },
            "huggingface/openai/gpt-oss-120b": { alias: "GPT-OSS 120B" },
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    نمای کلی همهٔ ارائه‌دهندگان، ارجاع‌های مدل و رفتار جابه‌جایی در زمان خرابی.
  </Card>
  <Card title="انتخاب مدل" href="/fa/concepts/models" icon="brain">
    نحوهٔ انتخاب و پیکربندی مدل‌ها.
  </Card>
  <Card title="مستندات ارائه‌دهندگان استنتاج" href="https://huggingface.co/docs/inference-providers" icon="book">
    مستندات رسمی ارائه‌دهندگان استنتاج Hugging Face.
  </Card>
  <Card title="پیکربندی" href="/fa/gateway/configuration" icon="gear">
    مرجع کامل پیکربندی.
  </Card>
</CardGroup>
