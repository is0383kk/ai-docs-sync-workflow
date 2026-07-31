---
read_when:
    - می‌خواهید OpenClaw را با مدل‌های NovitaAI اجرا کنید
    - به شناسه ارائه‌دهنده، کلید یا نقطه پایانی Novita نیاز دارید
summary: از API سازگار با OpenAI متعلق به NovitaAI با OpenClaw استفاده کنید
title: NovitaAI
x-i18n:
    generated_at: "2026-07-27T14:32:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 83e0e43e68d85d73e790023858a49f971b683129dbbdf6092fbd8bba4d8da331
    source_path: providers/novita.md
    workflow: 16
---

NovitaAI یک ارائه‌دهندهٔ میزبانی‌شدهٔ زیرساخت هوش مصنوعی با API سازگار با OpenAI است.
این ارائه‌دهنده به‌صورت داخلی همراه OpenClaw عرضه می‌شود (نیازی به نصب Plugin جداگانه نیست)، بنابراین
اعتبارنامه‌ها از جریان عادی احراز هویت مدل عبور می‌کنند و ارجاع‌های مدل به‌شکل
`novita/deepseek/deepseek-v3-0324` هستند.

## راه‌اندازی

یک کلید API در [novita.ai/settings/key-management](https://novita.ai/settings/key-management) ایجاد کنید، سپس اجرا کنید:

```bash
openclaw onboard --auth-choice novita-api-key
```

یا تنظیم کنید:

```bash
export NOVITA_API_KEY="<your-novita-api-key>" # pragma: allowlist secret
```

## پیش‌فرض‌ها

| تنظیم          | مقدار                              |
| -------------- | ---------------------------------- |
| شناسهٔ ارائه‌دهنده | `novita`                           |
| نام‌های مستعار | `novita-ai`, `novitaai`            |
| نشانی پایه     | `https://api.novita.ai/openai/v1`  |
| متغیر محیطی    | `NOVITA_API_KEY`                   |
| مدل پیش‌فرض    | `novita/deepseek/deepseek-v3-0324` |

## کاتالوگ مدل‌های داخلی

- `novita/moonshotai/kimi-k2.5`
- `novita/minimax/minimax-m2.7`
- `novita/zai-org/glm-5`
- `novita/deepseek/deepseek-v3-0324`
- `novita/deepseek/deepseek-r1-0528`
- `novita/qwen/qwen3-235b-a22b-fp8`

این یک نقطهٔ شروع است، نه کاتالوگی زنده. حساب، منطقه یا
خدمات فعلی Novita ممکن است مسیرهایی را اضافه، حذف یا محدود کند. پیش از
تنظیم یک پیش‌فرض بلندمدت، بررسی کنید:

```bash
openclaw models list --provider novita
```

## چه زمانی Novita را انتخاب کنید

- دسترسی میزبانی‌شده به مدل‌های دارای وزن باز با API سازگار با OpenAI.
- مسیرهای خانواده‌های DeepSeek، Kimi، MiniMax، GLM یا Qwen از طریق یک حساب
  ارائه‌دهنده.
- یک مسیر جایگزین میزبانی‌شدهٔ دیگر در کنار DeepInfra، GMI، OpenRouter یا APIهای
  مستقیم فروشندگان.
- میزبانی مدل در سمت ارائه‌دهنده به‌جای نگهداری زیرساخت LM Studio، Ollama،
  SGLang یا vLLM.

هنگامی که به پارامترهای درخواست بومی فروشنده یا قراردادهای پشتیبانی نیاز دارید،
یک ارائه‌دهندهٔ مستقیم فروشنده را انتخاب کنید. هنگامی که مدل باید
روی سخت‌افزار یا در محدودهٔ شبکهٔ خودتان اجرا شود، یک ارائه‌دهندهٔ محلی را انتخاب کنید.

## عیب‌یابی

- `401`/`403`: کلید را در صفحهٔ مدیریت کلید Novita بررسی کنید و اگر پروفایل ذخیره‌شده
  قدیمی است، `openclaw onboard --auth-choice novita-api-key` را دوباره اجرا کنید.
- خطاهای مدل ناشناخته: از `novita/<route-id>` دقیق بازگردانده‌شده توسط
  `openclaw models list --provider novita` استفاده کنید.
- مسیرهای کند یا ناموفق: مسیر مدل دیگری از Novita را امتحان کنید، یا برای بارهای کاری که می‌توانند
  تغییرات مختص ارائه‌دهنده را تحمل کنند، Novita را به‌عنوان ارائه‌دهندهٔ
  جایگزین تنظیم کنید.

## مرتبط

- [ارائه‌دهندگان مدل](/fa/concepts/model-providers)
- [فهرست ارائه‌دهندگان](/fa/providers/index)
