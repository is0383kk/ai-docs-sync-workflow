---
read_when:
    - تولید یا ویرایش تصاویر از طریق عامل
    - پیکربندی ارائه‌دهندگان و مدل‌های تولید تصویر
    - آشنایی با پارامترهای ابزار image_generate
sidebarTitle: Image generation
summary: تولید و ویرایش تصاویر از طریق image_generate در OpenAI، Google، fal، Microsoft Foundry، MiniMax، ComfyUI، DeepInfra، OpenRouter، LiteLLM، xAI و Vydra
title: تولید تصویر
x-i18n:
    generated_at: "2026-07-27T14:41:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9688b1bc649713d8ed345a69a28d20b36ecd768b6a6d28a2d6c022d65b081862
    source_path: tools/image-generation.md
    workflow: 16
---

ابزار `image_generate` تصاویر را از طریق ارائه‌دهندگان پیکربندی‌شدهٔ شما ایجاد و ویرایش می‌کند.
در نشست‌های چت، به‌صورت ناهمگام اجرا می‌شود: OpenClaw یک وظیفهٔ
پس‌زمینه ثبت می‌کند، شناسهٔ وظیفه را بی‌درنگ برمی‌گرداند و پس از پایان کار
ارائه‌دهنده، عامل را بیدار می‌کند. عامل تکمیل از حالت عادی پاسخِ قابل‌مشاهدهٔ
نشست پیروی می‌کند: در صورت پیکربندی، پاسخ نهایی به‌طور خودکار تحویل داده
می‌شود؛ یا وقتی نشست به ابزار پیام نیاز دارد، از `message(action="send")` استفاده می‌شود. اگر
نشست درخواست‌کننده غیرفعال باشد یا بیدارسازی فعال آن ناموفق شود، OpenClaw یک
ارسال جایگزین مستقیم و هم‌توان با تصاویر تولیدشده انجام می‌دهد تا نتیجه
از دست نرود.

<Note>
این ابزار تنها زمانی نمایش داده می‌شود که دست‌کم یک ارائه‌دهندهٔ تولید تصویر
در دسترس باشد. اگر `image_generate` را در ابزارهای عامل خود نمی‌بینید،
`agents.defaults.mediaModels.image` را پیکربندی کنید، یک کلید API ارائه‌دهنده تنظیم کنید،
یا با OpenAI ChatGPT/Codex OAuth وارد شوید.
</Note>

## شروع سریع

<Steps>
  <Step title="پیکربندی احراز هویت">
    برای دست‌کم یک ارائه‌دهنده یک کلید API تنظیم کنید (برای مثال `OPENAI_API_KEY`،
    `GEMINI_API_KEY`، `OPENROUTER_API_KEY`) یا با OpenAI Codex OAuth وارد شوید.
  </Step>
  <Step title="انتخاب مدل پیش‌فرض (اختیاری)">
    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "openai/gpt-image-2",
            timeoutMs: 180_000,
          },
        },
      },
    }
    ```

    ChatGPT/Codex OAuth از همان ارجاع مدل `openai/gpt-image-2` استفاده می‌کند. وقتی یک
    نمایهٔ OAuth با `openai` پیکربندی شده باشد، OpenClaw درخواست‌های تصویر را
    به‌جای آنکه ابتدا `OPENAI_API_KEY` را امتحان کند، از طریق آن نمایهٔ OAuth
    مسیریابی می‌کند. پیکربندی صریح `models.providers.openai` (کلید API، نشانی پایهٔ
    سفارشی/Azure) دوباره مسیر مستقیم OpenAI Images API را فعال می‌کند.

  </Step>
  <Step title="درخواست از عامل">
    _«تصویری از یک ربات خوش‌برخورد به‌عنوان نماد خوش‌یمنی تولید کن.»_

    عامل به‌طور خودکار `image_generate` را فراخوانی می‌کند. نیازی به افزودن ابزار
    به فهرست مجاز نیست؛ وقتی ارائه‌دهنده‌ای در دسترس باشد، ابزار به‌طور پیش‌فرض
    فعال است. ابزار یک شناسهٔ وظیفهٔ پس‌زمینه برمی‌گرداند؛ سپس عامل تکمیل، پس از
    آماده‌شدن پیوست تولیدشده، آن را از طریق ابزار `message` ارسال می‌کند.

  </Step>
</Steps>

<Warning>
برای نقاط پایانی LAN سازگار با OpenAI مانند LocalAI، مقدار سفارشی
`models.providers.openai.baseUrl` را نگه دارید و با
`browser.ssrfPolicy.dangerouslyAllowPrivateNetwork: true` صراحتاً آن را فعال کنید. نقاط پایانی
خصوصی و داخلی تصویر همچنان به‌طور پیش‌فرض مسدود هستند.
</Warning>

## مسیرهای رایج

| هدف                                                 | ارجاع مدل                                          | احراز هویت                                   |
| ---------------------------------------------------- | -------------------------------------------------- | -------------------------------------- |
| تولید تصویر OpenAI با صورت‌حساب API             | `openai/gpt-image-2`                               | `OPENAI_API_KEY`                       |
| تولید تصویر OpenAI با احراز هویت اشتراک Codex | `openai/gpt-image-2`                               | OpenAI ChatGPT/Codex OAuth             |
| PNG/WebP با پس‌زمینهٔ شفاف در OpenAI               | `openai/gpt-image-1.5`                             | `OPENAI_API_KEY` یا OpenAI Codex OAuth |
| تولید تصویر DeepInfra                           | `deepinfra/black-forest-labs/FLUX-1-schnell`       | `DEEPINFRA_API_KEY`                    |
| تولید بیان‌محور/سبک‌محور fal Krea 2      | `fal/krea/v2/medium/text-to-image`                 | `FAL_KEY`                              |
| تولید تصویر OpenRouter                          | `openrouter/google/gemini-3.1-flash-image-preview` | `OPENROUTER_API_KEY`                   |
| تولید تصویر LiteLLM                             | `litellm/gpt-image-2`                              | `LITELLM_API_KEY`                      |
| تولید تصویر Microsoft Foundry MAI               | `microsoft-foundry/<deployment-name>`              | `AZURE_OPENAI_API_KEY` یا Entra ID     |
| تولید تصویر Google Gemini                       | `google/gemini-3.1-flash-image`                    | `GEMINI_API_KEY` یا `GOOGLE_API_KEY`   |

همین ابزار تبدیل متن به تصویر و ویرایش تصویر مرجع را مدیریت می‌کند. برای یک
مرجع از `image` و برای چند مرجع از `images` استفاده کنید. برای مدل‌های
Krea 2 در fal، این مراجع به‌جای ورودی‌های ویرایش به‌عنوان مراجع سبک ارسال
می‌شوند. راهنمایی‌های خروجی پشتیبانی‌شده توسط ارائه‌دهنده، مانند
`quality`، `outputFormat` و `background`، در صورت امکان منتقل می‌شوند؛ و
اگر ارائه‌دهنده پشتیبانی از آن‌ها را اعلام نکرده باشد، نادیده‌گرفته‌شدنشان
گزارش می‌شود. پشتیبانی داخلی از پس‌زمینهٔ شفاف مختص OpenAI است؛ سایر
ارائه‌دهندگان نیز ممکن است در صورت تولید آن توسط سامانهٔ پشتیبانشان، آلفای
PNG را حفظ کنند.

## ارائه‌دهندگان پشتیبانی‌شده

| ارائه‌دهنده          | مدل پیش‌فرض                           | پشتیبانی از ویرایش                       | احراز هویت                                                  |
| ----------------- | --------------------------------------- | ---------------------------------- | ----------------------------------------------------- |
| ComfyUI           | `workflow`                              | بله (1 تصویر، پیکربندی‌شده در گردش‌کار) | `COMFY_API_KEY` یا `COMFY_CLOUD_API_KEY` برای فضای ابری    |
| DeepInfra         | `black-forest-labs/FLUX-1-schnell`      | بله (1 تصویر)                      | `DEEPINFRA_API_KEY`                                   |
| fal               | `fal-ai/flux/dev`                       | بله (محدودیت‌های خاص مدل)        | `FAL_KEY`                                             |
| Google            | `gemini-3.1-flash-image`                | بله (تا 5 تصویر)               | `GEMINI_API_KEY` یا `GOOGLE_API_KEY`                  |
| LiteLLM           | `gpt-image-2`                           | بله (تا 5 تصویر ورودی)         | `LITELLM_API_KEY`                                     |
| Microsoft Foundry | `<deployment-name>`                     | بله (فقط مدل‌های MAI-Image-2.5)    | `AZURE_OPENAI_API_KEY` یا Entra ID (`az login`)       |
| MiniMax           | `image-01`                              | بله (مرجع سوژه)            | `MINIMAX_API_KEY` یا MiniMax OAuth (`minimax-portal`) |
| OpenAI            | `gpt-image-2`                           | بله (تا 5 تصویر)               | `OPENAI_API_KEY` یا OpenAI ChatGPT/Codex OAuth        |
| OpenRouter        | `google/gemini-3.1-flash-image-preview` | بله (تا 5 تصویر ورودی)         | `OPENROUTER_API_KEY`                                  |
| Vydra             | `grok-imagine`                          | خیر                                 | `VYDRA_API_KEY`                                       |
| xAI               | `grok-imagine-image`                    | بله (تا 3 تصویر)               | `XAI_API_KEY`                                         |

برای بررسی ارائه‌دهندگان و مدل‌های موجود در زمان اجرا، از `action: "list"` استفاده کنید:

```text
/tool image_generate action=list
```

برای بررسی وظیفهٔ فعال تولید تصویر در نشست کنونی، از `action: "status"` استفاده کنید:

```text
/tool image_generate action=status
```

## قابلیت‌های ارائه‌دهندگان

| قابلیت            | ComfyUI            | DeepInfra | fal                                            | Google         | Microsoft Foundry | MiniMax               | OpenAI         | Vydra | xAI            |
| --------------------- | ------------------ | --------- | ---------------------------------------------- | -------------- | ----------------- | --------------------- | -------------- | ----- | -------------- |
| تولید (حداکثر تعداد)  | 1                  | 4         | 4                                              | 4              | 1                 | 9                     | 4              | 1     | 4              |
| ویرایش / مرجع      | 1 تصویر (گردش‌کار) | 1 تصویر   | Flux: 1؛ GPT: 10؛ مراجع سبک Krea: 10؛ NB2: 14 | تا 5 تصویر | 1 تصویر           | 1 تصویر (مرجع سوژه) | تا 5 تصویر | -     | تا 3 تصویر |
| کنترل اندازه          | -                  | ✓         | ✓                                              | ✓              | ✓                 | -                     | تا 4K       | -     | -              |
| نسبت ابعاد          | -                  | -         | ✓                                              | ✓              | -                 | ✓                     | -              | -     | ✓              |
| وضوح (1K/2K/4K) | -                  | -         | ✓                                              | ✓              | -                 | -                     | -              | -     | 1K, 2K         |

## پارامترهای ابزار

<ParamField path="prompt" type="string" required>
  درخواست تولید تصویر. برای `action: "generate"` الزامی است.
</ParamField>
<ParamField path="action" type='"generate" | "status" | "list"' default="generate">
  برای بررسی وظیفهٔ فعال نشست از `"status"` یا برای بررسی
  ارائه‌دهندگان و مدل‌های موجود در زمان اجرا از `"list"` استفاده کنید.
</ParamField>
<ParamField path="model" type="string">
  بازنویسی ارائه‌دهنده/مدل (برای مثال `openai/gpt-image-2`). برای پس‌زمینه‌های
  شفاف OpenAI از `openai/gpt-image-1.5` استفاده کنید.
</ParamField>
<ParamField path="image" type="string">
  مسیر یا URL یک تصویر مرجع برای حالت ویرایش.
</ParamField>
<ParamField path="images" type="string[]">
  چند تصویر مرجع برای حالت ویرایش یا مدل‌های مرجع سبک (تا 14 تصویر
  از طریق ابزار مشترک؛ محدودیت‌های خاص ارائه‌دهنده همچنان اعمال می‌شوند).
</ParamField>
<ParamField path="size" type="string">
  راهنمای اندازه: `1024x1024`، `1536x1024`، `1024x1536`، `2048x2048`، `3840x2160`.
</ParamField>
<ParamField path="aspectRatio" type="string">
  نسبت ابعاد: `1:1`، `2:1`، `20:9`، `19.5:9`، `2:3`، `3:2`، `2.35:1`، `3:4`،
  `4:3`، `4:5`، `5:4`، `9:16`، `9:19.5`، `9:20`، `16:9`، `21:9`، `1:2`، `4:1`،
  `1:4`، `8:1`، `1:8`. ارائه‌دهندگان زیرمجموعهٔ خاص مدل خود را اعتبارسنجی می‌کنند.
</ParamField>
<ParamField path="resolution" type='"1K" | "2K" | "4K"'>راهنمای وضوح.</ParamField>
<ParamField path="quality" type='"low" | "medium" | "high" | "auto"'>
  راهنمای کیفیت، در صورتی که ارائه‌دهنده از آن پشتیبانی کند.
</ParamField>
<ParamField path="outputFormat" type='"png" | "jpeg" | "webp"'>
  راهنمای قالب خروجی، در صورتی که ارائه‌دهنده از آن پشتیبانی کند.
</ParamField>
<ParamField path="background" type='"transparent" | "opaque" | "auto"'>
  راهنمای پس‌زمینه، در صورتی که ارائه‌دهنده از آن پشتیبانی کند. برای
  ارائه‌دهندگان دارای قابلیت شفافیت، از `transparent` همراه با
  `outputFormat: "png"` یا `"webp"` استفاده کنید.
</ParamField>
<ParamField path="count" type="number">تعداد تصاویر برای تولید (1-4).</ParamField>
<ParamField path="timeoutMs" type="number">
  مهلت اختیاری درخواست از ارائه‌دهنده بر حسب میلی‌ثانیه. وقتی Codex
  `image_generate` را از طریق ابزارهای پویا فراخوانی می‌کند، این مقدار مختص
  هر فراخوانی همچنان مقدار پیش‌فرض پیکربندی‌شده را بازنویسی می‌کند و حداکثر
  آن 600000 ms است.
</ParamField>
<ParamField path="filename" type="string">راهنمای نام فایل خروجی.</ParamField>
<ParamField path="openai" type="object">
  راهنمایی‌های مختص OpenAI:‏ `background`، `moderation`، `outputCompression` و `user`.
</ParamField>
<ParamField path="fal.creativity" type='"raw" | "low" | "medium" | "high"'>
  کنترل خلاقیت fal Krea 2. مقدار پیش‌فرض `medium` است.
</ParamField>

<Note>
همهٔ ارائه‌دهندگان از همهٔ پارامترها پشتیبانی نمی‌کنند. وقتی یک ارائه‌دهندهٔ
جایگزین به‌جای گزینهٔ هندسی دقیقِ درخواست‌شده از گزینه‌ای نزدیک پشتیبانی
می‌کند، OpenClaw پیش از ارسال، آن را به نزدیک‌ترین اندازه، نسبت ابعاد یا
وضوح پشتیبانی‌شده نگاشت می‌کند. راهنمایی‌های خروجی پشتیبانی‌نشده برای
ارائه‌دهندگانی که پشتیبانی از آن‌ها را اعلام نمی‌کنند حذف می‌شوند و این
موضوع در نتیجهٔ ابزار گزارش می‌شود. نتایج ابزار تنظیمات اعمال‌شده را گزارش
می‌کنند؛ `details.normalization` هرگونه تبدیل از مقدار درخواست‌شده به مقدار
اعمال‌شده را ثبت می‌کند.
</Note>

## پیکربندی

### انتخاب مدل

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "openai/gpt-image-2",
        timeoutMs: 180_000,
        fallbacks: [
          "openrouter/google/gemini-3.1-flash-image-preview",
          "google/gemini-3.1-flash-image",
          "fal/fal-ai/flux/dev",
        ],
      },
    },
  },
}
```

### ترتیب انتخاب ارائه‌دهنده

OpenClaw ارائه‌دهندگان را به این ترتیب امتحان می‌کند:

1. پارامتر **`model`** از فراخوانی ابزار (اگر عامل آن را مشخص کند).
2. **`imageGenerationModel.primary`** از پیکربندی.
3. **`imageGenerationModel.fallbacks`** به‌ترتیب.
4. **تشخیص خودکار** - فقط پیش‌فرض‌های ارائه‌دهنده‌ای که احراز هویت از آن‌ها پشتیبانی می‌کند:
   - ابتدا ارائه‌دهنده پیش‌فرض فعلی؛
   - سپس سایر ارائه‌دهندگان ثبت‌شده تولید تصویر، به‌ترتیب شناسه ارائه‌دهنده.

اگر ارائه‌دهنده‌ای ناموفق باشد (خطای احراز هویت، محدودیت نرخ و غیره)، نامزد
پیکربندی‌شده بعدی به‌طور خودکار امتحان می‌شود. اگر همه ناموفق باشند، خطا شامل جزئیات
هر تلاش خواهد بود.

<AccordionGroup>
  <Accordion title="بازنویسی مدل در هر فراخوانی دقیق است">
    بازنویسی `model` در هر فراخوانی فقط همان ارائه‌دهنده/مدل را امتحان می‌کند و
    به ارائه‌دهنده اصلی/جایگزین پیکربندی‌شده یا ارائه‌دهندگان تشخیص‌داده‌شده خودکار ادامه نمی‌دهد.
  </Accordion>
  <Accordion title="تشخیص خودکار از احراز هویت آگاه است">
    پیش‌فرض یک ارائه‌دهنده فقط زمانی وارد فهرست نامزدها می‌شود که OpenClaw بتواند
    واقعاً در آن ارائه‌دهنده احراز هویت کند. جایگزینی خودکار میان ارائه‌دهندگان
    احرازشده همیشه فعال است؛ `model` در هر فراخوانی همچنان مرجع نهایی است.
  </Accordion>
  <Accordion title="مهلت‌های زمانی">
    برای بک‌اندهای کند تولید تصویر، `agents.defaults.mediaModels.image.timeoutMs` را تنظیم کنید.
    پارامتر ابزار `timeoutMs` در هر فراخوانی، پیش‌فرض پیکربندی‌شده را بازنویسی می‌کند
    و پیش‌فرض‌های پیکربندی‌شده نیز پیش‌فرض‌های ارائه‌دهنده تعریف‌شده توسط Plugin را
    بازنویسی می‌کنند. ارائه‌دهندگان میزبانی‌شده تصویر در Google و OpenRouter از
    پیش‌فرض 180 ثانیه استفاده می‌کنند؛ تولید تصویر Microsoft Foundry MAI، ‏xAI و
    Azure OpenAI از 600 ثانیه استفاده می‌کند. فراخوانی‌های ابزار پویا در Codex از
    پیش‌فرض پل `image_generate` برابر با 120 ثانیه استفاده می‌کنند و در صورت
    پیکربندی، همان بودجه زمانی را رعایت می‌کنند؛ این مقدار به حداکثر 600000 ms
    پل ابزار پویای OpenClaw محدود است.
  </Accordion>
  <Accordion title="بازرسی هنگام اجرا">
    برای بررسی ارائه‌دهندگان ثبت‌شده فعلی، مدل‌های پیش‌فرض آن‌ها و راهنمای
    متغیرهای محیطی احراز هویت، از `action: "list"` استفاده کنید.
  </Accordion>
</AccordionGroup>

### ویرایش تصویر

OpenAI، ‏OpenRouter، ‏Google، ‏DeepInfra، ‏fal، ‏Microsoft Foundry، ‏MiniMax،
‏ComfyUI و xAI از ویرایش تصاویر مرجع پشتیبانی می‌کنند. مدل‌های Krea 2 در fal
به‌جای ورودی ویرایش، از همان فیلدهای `image` / `images`
به‌عنوان مراجع سبک استفاده می‌کنند. مسیر یا URL یک تصویر مرجع را ارسال کنید:

```text
"نسخه‌ای آبرنگی از این عکس تولید کن" + image: "/path/to/photo.jpg"
```

OpenAI، ‏OpenRouter و Google از طریق پارامتر `images` تا 5 تصویر مرجع
را پشتیبانی می‌کنند؛ xAI تا 3 تصویر را پشتیبانی می‌کند. fal برای تبدیل تصویر به
تصویر در Flux از 1 تصویر مرجع، برای ویرایش‌های GPT Image 2 تا 10 تصویر، برای
Krea 2 تا 10 مرجع سبک و برای ویرایش‌های Nano Banana 2 تا 14 تصویر پشتیبانی
می‌کند. Microsoft Foundry، ‏MiniMax و ComfyUI از 1 تصویر پشتیبانی می‌کنند.

## بررسی عمیق ارائه‌دهندگان

<AccordionGroup>
  <Accordion title="OpenAI gpt-image-2 (و gpt-image-1.5)">
    تولید تصویر OpenAI به‌طور پیش‌فرض از `openai/gpt-image-2` استفاده می‌کند. اگر
    نمایه OAuth مربوط به `openai` پیکربندی شده باشد، OpenClaw همان
    نمایه OAuth استفاده‌شده توسط مدل‌های گفت‌وگوی اشتراکی Codex را دوباره
    استفاده می‌کند و درخواست تصویر را از طریق بک‌اند Codex Responses می‌فرستد.
    URLهای پایه قدیمی Codex مانند `https://chatgpt.com/backend-api` برای درخواست‌های تصویر به
    `https://chatgpt.com/backend-api/codex` استانداردسازی می‌شوند. OpenClaw برای آن درخواست
    **به‌طور پنهانی** به `OPENAI_API_KEY` بازنمی‌گردد - برای اجبار مسیریابی
    مستقیم از طریق OpenAI Images API، ‏`models.providers.openai` را صریحاً با کلید API،
    ‏URL پایه سفارشی یا نقطه پایانی Azure پیکربندی کنید.

    مدل‌های `openai/gpt-image-1.5`، ‏`openai/gpt-image-1` و
    `openai/gpt-image-1-mini` همچنان می‌توانند صریحاً انتخاب شوند. برای خروجی PNG/WebP
    با پس‌زمینه شفاف از `gpt-image-1.5` استفاده کنید؛ API فعلی
    `gpt-image-2`، ‏`background: "transparent"` را رد می‌کند.

    `gpt-image-2` هم از تولید متن‌به‌تصویر و هم از ویرایش تصویر مرجع از
    طریق همان ابزار `image_generate` پشتیبانی می‌کند. OpenClaw مقادیر
    `prompt`، ‏`count`، ‏`size`، ‏`quality`، ‏`outputFormat`
    و تصاویر مرجع را به OpenAI ارسال می‌کند. OpenAI مقادیر
    `aspectRatio` یا `resolution` را مستقیماً دریافت **نمی‌کند**؛
    OpenClaw در صورت امکان آن‌ها را به یک `size` پشتیبانی‌شده نگاشت
    می‌کند و در غیر این صورت ابزار آن‌ها را به‌عنوان بازنویسی‌های نادیده‌گرفته‌شده
    گزارش می‌دهد.

    گزینه‌های مختص OpenAI در شیء `openai` قرار دارند:

    ```json
    {
      "quality": "low",
      "outputFormat": "jpeg",
      "openai": {
        "background": "opaque",
        "moderation": "low",
        "outputCompression": 60,
        "user": "end-user-42"
      }
    }
    ```

    `openai.background` مقادیر `transparent`، ‏`opaque` یا `auto`
    را می‌پذیرد؛ خروجی‌های شفاف به `outputFormat` ‏`png` یا
    `webp` و یک مدل تصویر OpenAI با قابلیت شفافیت نیاز دارند.
    OpenClaw درخواست‌های پیش‌فرض `gpt-image-2` با پس‌زمینه شفاف را به
    `gpt-image-1.5` هدایت می‌کند. `openai.outputCompression` برای خروجی‌های JPEG/WebP
    اعمال می‌شود و برای خروجی‌های PNG نادیده گرفته می‌شود.

    راهنمای سطح بالای `background` مستقل از ارائه‌دهنده است و در حال حاضر،
    هنگام انتخاب ارائه‌دهنده OpenAI، به همان فیلد درخواست `background`
    در OpenAI نگاشت می‌شود. ارائه‌دهندگانی که پشتیبانی از پس‌زمینه را اعلام
    نمی‌کنند، به‌جای دریافت پارامتر پشتیبانی‌نشده آن را در
    `ignoredOverrides` بازمی‌گردانند.

    برای هدایت تولید تصویر OpenAI از طریق یک استقرار Azure OpenAI به‌جای
    `api.openai.com`، به
    [نقاط پایانی Azure OpenAI](/fa/providers/openai#azure-openai-endpoints) مراجعه کنید.

  </Accordion>
  <Accordion title="مدل‌های تصویر Microsoft Foundry MAI">
    تولید تصویر Microsoft Foundry از نام استقرار مدل‌های تصویر MAI مستقرشده
    زیر پیشوند ارائه‌دهنده `microsoft-foundry/` استفاده می‌کند. مدل پیش‌فرضی در
    سطح ارائه‌دهنده وجود ندارد، زیرا API مربوط به MAI انتظار دارد نام استقرار
    خود را در فیلد `model` وارد کنید:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "microsoft-foundry/<deployment-name>",
            timeoutMs: 600_000,
          },
        },
      },
    }
    ```

    این ارائه‌دهنده از API مربوط به MAI در Microsoft Foundry استفاده می‌کند، نه OpenAI Images API:

    - نقطه پایانی تولید: `/mai/v1/images/generations`
    - نقطه پایانی ویرایش: `/mai/v1/images/edits`
    - احراز هویت: `AZURE_OPENAI_API_KEY` / کلید API ارائه‌دهنده، یا Entra ID از طریق `az login`
    - خروجی: یک تصویر PNG
    - اندازه: پیش‌فرض `1024x1024`؛ عرض و ارتفاع باید هرکدام حداقل 768 px باشند،
      و مجموع پیکسل‌ها باید حداکثر 1,048,576 باشد
    - ویرایش‌ها: یک تصویر مرجع PNG یا JPEG که فقط توسط استقرارهای
      `MAI-Image-2.5-Flash` و `MAI-Image-2.5` پشتیبانی می‌شود

    تولید صرفاً مبتنی بر پرامپت می‌تواند با پیکربندی فقط نقطه پایانی Foundry،
    از یک نام استقرار سفارشی استفاده کند. ویرایش با نام‌های استقرار سفارشی به
    فراداده راه‌اندازی اولیه/مدل نیاز دارد تا OpenClaw بتواند تأیید کند که
    استقرار بر پایه `MAI-Image-2.5-Flash` یا `MAI-Image-2.5` است.

    مدل‌های تصویر فعلی MAI عبارت‌اند از `MAI-Image-2.5-Flash`، ‏`MAI-Image-2.5`،
    ‏`MAI-Image-2e` و `MAI-Image-2`. برای راه‌اندازی و رفتار مدل گفت‌وگو،
    به [Plugin مربوط به Microsoft Foundry](/fa/plugins/reference/microsoft-foundry)
    مراجعه کنید.

  </Accordion>
  <Accordion title="مدل‌های تصویر OpenRouter">
    تولید تصویر OpenRouter از همان `OPENROUTER_API_KEY` استفاده می‌کند و
    از طریق API تصویر تکمیل گفت‌وگوی OpenRouter مسیریابی می‌شود. مدل‌های تصویر
    OpenRouter را با پیشوند `openrouter/` انتخاب کنید:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "openrouter/google/gemini-3.1-flash-image-preview",
          },
        },
      },
    }
    ```

    OpenClaw مقادیر `prompt`، ‏`count`، تصاویر مرجع و
    راهنماهای سازگار با Gemini یعنی `aspectRatio` / `resolution` را
    به OpenRouter ارسال می‌کند. میان‌برهای داخلی فعلی برای مدل‌های تصویر
    OpenRouter شامل `google/gemini-3.1-flash-image`،
    ‏`google/gemini-3-pro-image` و `openai/gpt-5.4-image-2` هستند. برای مشاهده مواردی که
    Plugin پیکربندی‌شده شما ارائه می‌کند، از `action: "list"` استفاده کنید.

  </Accordion>
  <Accordion title="fal Krea 2">
    مدل‌های Krea 2 در fal به‌جای شِمای عمومی `image_size` که Flux از آن
    استفاده می‌کند، از شِمای بومی Krea در fal استفاده می‌کنند. OpenClaw موارد
    زیر را ارسال می‌کند:

    - `aspect_ratio` برای راهنمای نسبت ابعاد
    - `creativity`، با مقدار پیش‌فرض `medium`
    - `image_style_references` هنگامی که `image` یا `images` ارائه شده باشد

    برای تصویرسازی سریع‌تر و پرحالت، Krea 2 Medium و برای ظاهرهای واقع‌گرایانه‌تر،
    بافت‌دارتر، کندتر و پرجزئیات‌تر، Krea 2 Large را انتخاب کنید:

    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "fal/krea/v2/medium/text-to-image",
          },
        },
      },
    }
    ```

    Krea 2 در حال حاضر برای هر درخواست یک تصویر بازمی‌گرداند. برای Krea،
    ‏`aspectRatio` را ترجیح دهید؛ OpenClaw مقدار `size` را به
    نزدیک‌ترین نسبت ابعاد پشتیبانی‌شده Krea نگاشت می‌کند و به‌جای حذف
    `resolution`، آن را برای Krea رد می‌کند. هنگامی که سطح خلاقیت بومی
    Krea را می‌خواهید، از `fal.creativity` استفاده کنید:

    ```json
    {
      "model": "fal/krea/v2/medium/text-to-image",
      "prompt": "A cyber zine portrait with risograph texture",
      "aspectRatio": "9:16",
      "fal": {
        "creativity": "high"
      }
    }
    ```

  </Accordion>
  <Accordion title="احراز هویت دوگانه MiniMax">
    تولید تصویر MiniMax از طریق هر دو مسیر احراز هویت همراه MiniMax در دسترس است:

    - `minimax/image-01` برای راه‌اندازی‌های مبتنی بر کلید API
    - `minimax-portal/image-01` برای راه‌اندازی‌های مبتنی بر OAuth

  </Accordion>
  <Accordion title="xAI grok-imagine-image">
    ارائه‌دهنده همراه xAI برای درخواست‌های صرفاً مبتنی بر پرامپت از
    `/v1/images/generations` و در صورت وجود `image` یا `images`
    از `/v1/images/edits` استفاده می‌کند.

    - مدل‌ها: `xai/grok-imagine-image`، ‏`xai/grok-imagine-image-quality`
    - تعداد: تا 4
    - مراجع: یک `image` یا حداکثر سه `images`
    - نسبت‌های ابعاد: `1:1`، ‏`16:9`، ‏`9:16`، ‏`4:3`، ‏`3:4`، ‏`3:2`، ‏`2:3`، ‏`2:1`،
      ‏`1:2`، ‏`19.5:9`، ‏`9:19.5`، ‏`20:9`، ‏`9:20`
    - وضوح‌ها: `1K`، ‏`2K`
    - خروجی‌ها: به‌صورت پیوست‌های تصویری مدیریت‌شده توسط OpenClaw بازگردانده می‌شوند

    OpenClaw عمداً `quality`، ‏`mask`،
    ‏`user` بومی xAI یا نسبت ابعاد `auto` را تا زمانی
    که این کنترل‌ها در قرارداد مشترک و میان‌ارائه‌دهنده‌ای `image_generate`
    وجود نداشته باشند، ارائه نمی‌کند.

  </Accordion>
</AccordionGroup>

## مثال‌ها

<Tabs>
  <Tab title="تولید (منظره 4K)">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="یک پوستر تحریریه‌ای تمیز برای تولید تصویر OpenClaw" size=3840x2160 count=1
```
  </Tab>
  <Tab title="تولید (PNG شفاف)">
```text
/tool image_generate action=generate model=openai/gpt-image-1.5 prompt="یک برچسب دایره قرمز ساده روی پس‌زمینه شفاف" outputFormat=png background=transparent
```

CLI معادل:

```bash
openclaw infer image generate \
  --model openai/gpt-image-1.5 \
  --output-format png \
  --background transparent \
  --prompt "یک برچسب دایره قرمز ساده روی پس‌زمینه شفاف" \
  --json
```

  </Tab>
  <Tab title="تولید (کیفیت پایین OpenAI)">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="پیش‌نویس کم‌هزینه پوستر برای یک برنامه آرام بهره‌وری" quality=low openai='{"moderation":"low"}'
```

CLI معادل:

```bash
openclaw infer image generate \
  --model openai/gpt-image-2 \
  --quality low \
  --openai-moderation low \
  --prompt "پیش‌نویس کم‌هزینه پوستر برای یک برنامه آرام بهره‌وری" \
  --json
```

  </Tab>
  <Tab title="تولید (دو تصویر مربعی)">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="دو جهت‌گیری بصری برای آیکون یک برنامه آرام بهره‌وری" size=1024x1024 count=2
```
  </Tab>
  <Tab title="ویرایش (یک مرجع)">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="سوژه را حفظ کن و پس‌زمینه را با یک چیدمان روشن استودیویی جایگزین کن" image=/path/to/reference.png size=1024x1536
```
  </Tab>
  <Tab title="ویرایش (چند مرجع)">
```text
/tool image_generate action=generate model=openai/gpt-image-2 prompt="هویت شخصیت از تصویر اول را با پالت رنگی تصویر دوم ترکیب کن" images='["/path/to/character.png","/path/to/palette.jpg"]' size=1536x1024
```
  </Tab>
  <Tab title="مراجع سبک Krea">
```text
/tool image_generate action=generate model=fal/krea/v2/medium/text-to-image prompt="یک پرتره تحریریه‌ای پرحالت با استفاده از این پالت رنگی و بافت چاپی" images='["/path/to/palette.png","/path/to/texture.jpg"]' aspectRatio=9:16 fal='{"creativity":"high"}'
```
  </Tab>
</Tabs>

همان پرچم‌های `--output-format`، `--background`، `--quality` و
`--openai-moderation` در `openclaw infer image edit` نیز در دسترس‌اند؛
`--openai-background` همچنان به‌عنوان نام مستعار ویژه OpenAI باقی می‌ماند. ارائه‌دهندگان همراه
به‌جز OpenAI در حال حاضر کنترل صریح پس‌زمینه را اعلام نمی‌کنند، بنابراین
`background: "transparent"` برای آن‌ها نادیده‌گرفته‌شده گزارش می‌شود.

## مرتبط

- [نمای کلی ابزارها](/fa/tools) - همه ابزارهای عامل موجود
- [ComfyUI](/fa/providers/comfy) - راه‌اندازی گردش‌کار محلی ComfyUI و Comfy Cloud
- [fal](/fa/providers/fal) - راه‌اندازی ارائه‌دهنده تصویر و ویدیوی fal
- [Google (Gemini)](/fa/providers/google) - راه‌اندازی ارائه‌دهنده تصویر Gemini
- [Plugin ‏Microsoft Foundry](/fa/plugins/reference/microsoft-foundry) - راه‌اندازی گفت‌وگوی Microsoft Foundry و تصویر MAI
- [MiniMax](/fa/providers/minimax) - راه‌اندازی ارائه‌دهنده تصویر MiniMax
- [OpenAI](/fa/providers/openai) - راه‌اندازی ارائه‌دهنده OpenAI Images
- [Vydra](/fa/providers/vydra) - راه‌اندازی تصویر، ویدیو و گفتار Vydra
- [xAI](/fa/providers/xai) - راه‌اندازی تصویر، ویدیو، جست‌وجو، اجرای کد و TTS در Grok
- [مرجع پیکربندی](/fa/gateway/config-agents#agent-defaults) - پیکربندی `imageGenerationModel`
- [مدل‌ها](/fa/concepts/models) - پیکربندی مدل و انتقال خودکار در زمان خرابی
