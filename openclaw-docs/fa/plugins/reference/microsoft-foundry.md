---
read_when:
    - در حال نصب، پیکربندی یا ممیزی Plugin ‏microsoft-foundry هستید
summary: پشتیبانی از ارائه‌دهنده مدل Microsoft Foundry را به OpenClaw اضافه می‌کند.
title: Plugin Microsoft Foundry
x-i18n:
    generated_at: "2026-07-27T16:58:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f2ea554ce16cffeb4cc315e53d986d6f07b5e113fbb844c61c6575f19f8ad291
    source_path: plugins/reference/microsoft-foundry.md
    workflow: 16
---

# Plugin ‏Microsoft Foundry

پشتیبانی از ارائه‌دهنده مدل Microsoft Foundry را به OpenClaw اضافه می‌کند.

## توزیع

- بسته: `@openclaw/microsoft-foundry`
- روش نصب: همراه OpenClaw ارائه می‌شود

## سطح

ارائه‌دهندگان: `microsoft-foundry`؛ قراردادها: `imageGenerationProviders`

<!-- openclaw-plugin-reference:manual-start -->

- ارائه‌دهنده تولید تصویر: `microsoft-foundry`

## الزامات

- یک منبع Microsoft Foundry یا Azure AI Foundry دارای استقرارها.
- احراز هویت با کلید API از طریق `AZURE_OPENAI_API_KEY` یا یک کلید API پیکربندی‌شده برای ارائه‌دهنده.
- برای احراز هویت Entra ID،‏ Azure CLI را نصب کنید و پیش از
  راه‌اندازی اولیه، `az login` را اجرا کنید. OpenClaw توکن‌های زمان اجرای Microsoft Foundry را از طریق
  `az account get-access-token` تازه‌سازی می‌کند.

## مدل‌های چت

استقرارهای چت Microsoft Foundry از ارجاع مدل ارائه‌دهنده
`microsoft-foundry/<deployment-name>` استفاده می‌کنند. راه‌اندازی اولیه با Azure CLI منابع
و استقرارهای Foundry را شناسایی می‌کند و سپس نام استقرار انتخاب‌شده را در
پیکربندی مدل می‌نویسد.

OpenClaw برای APIهای چت سازگار با OpenAI که پشتیبانی می‌شوند، از نقطه پایانی
`/openai/v1` در Foundry استفاده می‌کند:

- خانواده‌های مدل GPT،‏ `o*`،‏ `computer-use-preview` و DeepSeek-V4 به‌طور پیش‌فرض از
  `openai-responses` استفاده می‌کنند.
- استقرارهای MAI-DS-R1 و سایر استقرارهای تکمیل چت، مگر آنکه یک API پشتیبانی‌شده صراحتاً پیکربندی شده باشد، از `openai-completions`
  استفاده می‌کنند.
- قابلیت استدلال MAI-DS-R1 از طریق محتوای استدلال ثبت می‌شود، نه
  از طریق `reasoning_effort`. فراداده توکن‌های زمینه و خروجی آن
  163,840 توکن است.

استقرارهای Anthropic Claude در Microsoft Foundry از ساختار Anthropic Messages
API استفاده می‌کنند، نه ساختار سازگار با OpenAI در `/openai/v1`. تا زمانی که Plugin ‏Microsoft Foundry
از زمان اجرای بومی Anthropic پشتیبانی کند، آن‌ها را به‌عنوان یک ارائه‌دهنده سفارشی
`anthropic-messages` پیکربندی کنید. اگر نام استقرار Foundry با شناسه مدل
Claude متفاوت است، `params.canonicalModelId` را در ورودی مدل تنظیم کنید تا OpenClaw
بتواند قراردادهای انتقال مختص مدل را اعمال کند، `/think off` را به‌درستی نگاشت کند و
تفکر امضاشده را به‌شکلی امن حفظ کند.

## تولید تصویر MAI

این Plugin،‏ `microsoft-foundry` را برای `image_generate` با مدل‌های فعلی
تولید تصویر Microsoft AI ثبت می‌کند:

- `MAI-Image-2.5-Flash`
- `MAI-Image-2.5`
- `MAI-Image-2e`
- `MAI-Image-2`

از نام یک استقرار تصویر MAI که مستقر شده است، به‌عنوان ارجاع مدل استفاده کنید. ارائه‌دهنده
مدل تصویر پیش‌فرضی اعلام نمی‌کند، زیرا API ‏MAI به نام استقرار شما
در فیلد `model` درخواست نیاز دارد:

```json5
{
  agents: {
    defaults: {
      imageGenerationModel: {
        primary: "microsoft-foundry/<deployment-name>",
        timeoutMs: 600000,
      },
    },
  },
}
```

تولید صرفاً بر اساس پرامپت، نقطه پایانی تولیدات MAI در Microsoft Foundry را فراخوانی می‌کند:
`/mai/v1/images/generations`. ویرایش‌های مبتنی بر تصویر مرجع،
`/mai/v1/images/edits` را فراخوانی می‌کنند و به استقرارهای `MAI-Image-2.5-Flash` و
`MAI-Image-2.5` محدود هستند.

تولید صرفاً بر اساس پرامپت می‌تواند فقط با پیکربندی نقطه پایانی Foundry از یک نام استقرار
سفارشی استفاده کند. برای ویرایش تصویر با نام استقرار سفارشی، استقرار را
از طریق راه‌اندازی اولیه انتخاب کنید یا فراداده مدل را وارد کنید تا OpenClaw بتواند تأیید کند
که استقرار بر `MAI-Image-2.5-Flash` یا `MAI-Image-2.5` متکی است.

محدودیت‌های تصویر MAI:

- خروجی: یک تصویر PNG در هر درخواست.
- اندازه: مقدار پیش‌فرض `1024x1024`؛ عرض و ارتفاع باید هر دو دست‌کم 768 px باشند.
- مجموع پیکسل‌ها: حاصل‌ضرب عرض × ارتفاع باید حداکثر 1,048,576 باشد.
- ویرایش‌ها: یک تصویر ورودی PNG یا JPEG.
- راهنمایی‌های مشترک پشتیبانی‌نشده مانند `aspectRatio`،‏ `resolution`،‏ `quality`،
  `background` و `outputFormat` غیر PNG به Microsoft Foundry ارسال نمی‌شوند.

## عیب‌یابی

- `az: command not found`:‏ Azure CLI را نصب کنید یا از احراز هویت با کلید API استفاده کنید.
- `Microsoft Foundry endpoint missing for MAI image generation`: یک
  استقرار Foundry را از طریق راه‌اندازی اولیه انتخاب کنید یا `models.providers.microsoft-foundry.baseUrl` را اضافه کنید.
- `supports MAI image deployments only`: مدل تصویر انتخاب‌شده به یک
  استقرار غیر MAI اشاره می‌کند. برای `image_generate` از یک مدل تصویر MAI مستقرشده استفاده کنید.

<!-- openclaw-plugin-reference:manual-end -->
