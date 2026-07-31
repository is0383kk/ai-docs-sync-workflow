---
read_when:
    - می‌خواهید از Cohere با OpenClaw استفاده کنید
    - به متغیر محیطی کلید API سرویس Cohere یا گزینه احراز هویت CLI نیاز دارید
summary: راه‌اندازی Cohere (احراز هویت + انتخاب مدل)
title: Cohere
x-i18n:
    generated_at: "2026-07-27T14:30:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fee46bf80609bd5e8211d6be507713f4de178653941effb81ebae48d8bb6528a
    source_path: providers/cohere.md
    workflow: 16
---

[Cohere](https://cohere.com) استنتاج سازگار با OpenAI را از طریق API سازگاری خود ارائه می‌دهد. OpenClaw در دورهٔ انتقال Cohere به حالت خارجی، ارائه‌دهندهٔ Cohere را به‌صورت همراه عرضه می‌کند و آن را به‌عنوان Plugin خارجی رسمی نیز منتشر می‌کند.

| ویژگی        | مقدار                                                |
| --------------- | ---------------------------------------------------- |
| شناسهٔ ارائه‌دهنده     | `cohere`                                             |
| Plugin          | در دورهٔ انتقال به‌صورت همراه؛ بستهٔ خارجی رسمی |
| متغیر محیطی احراز هویت    | `COHERE_API_KEY`                                     |
| پرچم راه‌اندازی اولیه | `--auth-choice cohere-api-key`                       |
| پرچم مستقیم CLI | `--cohere-api-key <key>`                             |
| API             | سازگار با OpenAI (`openai-completions`)             |
| نشانی پایه        | `https://api.cohere.ai/compatibility/v1`             |
| مدل پیش‌فرض   | `cohere/command-a-plus-05-2026`                      |
| پنجرهٔ زمینه  | 128,000 توکن                                       |

## کاتالوگ داخلی

| ارجاع مدل                            | ورودی       | زمینه | حداکثر خروجی | یادداشت‌ها                                         |
| ------------------------------------ | ----------- | ------- | ---------- | --------------------------------------------- |
| `cohere/command-a-plus-05-2026`      | متن، تصویر | 128,000 | 64,000     | پیش‌فرض؛ مدل شاخص عامل‌محور و استدلالی |
| `cohere/command-a-03-2025`           | متن        | 256,000 | 8,000      | مدل پیشین Command A                      |
| `cohere/command-a-reasoning-08-2025` | متن        | 256,000 | 32,000     | استدلال عامل‌محور و استفاده از ابزار                |
| `cohere/command-a-vision-07-2025`    | متن، تصویر | 128,000 | 8,000      | تحلیل تصویر و سند؛ بدون استفاده از ابزار     |
| `cohere/north-mini-code-1-0`         | متن، تصویر | 256,000 | 64,000     | کدنویسی عامل‌محور؛ استدلال؛ محدودیت‌های رایگان        |

مدل‌های Cohere دارای قابلیت استدلال از دو حالت استدلال API سازگاری پشتیبانی می‌کنند. OpenClaw گزینهٔ **خاموش** را به `none` و هر سطح فعال تفکر را به `high` نگاشت می‌کند. Command A Vision از استفاده از ابزار پشتیبانی نمی‌کند؛ بنابراین OpenClaw ابزارهای عامل را برای آن مدل غیرفعال نگه می‌دارد.

## شروع به کار

1. Cohere همراه بسته‌های فعلی OpenClaw عرضه می‌شود. اگر وجود ندارد، بستهٔ خارجی را نصب و Gateway را راه‌اندازی مجدد کنید:

```bash
openclaw plugins install @openclaw/cohere-provider
openclaw gateway restart
```

2. یک کلید API برای Cohere ایجاد کنید.
3. راه‌اندازی اولیه را اجرا کنید:

```bash
openclaw onboard --non-interactive \
  --auth-choice cohere-api-key \
  --cohere-api-key "$COHERE_API_KEY"
```

4. در دسترس بودن کاتالوگ را تأیید کنید:

```bash
openclaw models list --provider cohere
```

راه‌اندازی اولیه تنها زمانی Cohere را به‌عنوان مدل اصلی تنظیم می‌کند که از قبل هیچ مدل اصلی پیکربندی نشده باشد.

## راه‌اندازی فقط با محیط

`COHERE_API_KEY` را در دسترس فرایند Gateway قرار دهید، سپس مدل Cohere را انتخاب کنید:

```json5
{
  agents: {
    defaults: {
      model: { primary: "cohere/command-a-plus-05-2026" },
    },
  },
}
```

<Note>
اگر Gateway به‌صورت سرویس پس‌زمینه یا در Docker اجرا می‌شود، `COHERE_API_KEY` را برای آن سرویس تنظیم کنید. صادر کردن آن فقط در یک پوستهٔ تعاملی، متغیر را در دسترس Gateway در حال اجرا قرار نمی‌دهد.
</Note>

## مرتبط

- [ارائه‌دهندگان مدل](/fa/concepts/model-providers)
- [CLI مدل‌ها](/fa/cli/models)
- [فهرست ارائه‌دهندگان](/fa/providers/index)
