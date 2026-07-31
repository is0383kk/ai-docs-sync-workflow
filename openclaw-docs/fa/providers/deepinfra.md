---
read_when:
    - برای برترین LLMهای متن‌باز، یک کلید API واحد می‌خواهید
    - می‌خواهید مدل‌ها را از طریق API شرکت DeepInfra در OpenClaw اجرا کنید
summary: از API یکپارچهٔ DeepInfra برای دسترسی به محبوب‌ترین مدل‌های متن‌باز و پیشرو در OpenClaw استفاده کنید
title: DeepInfra
x-i18n:
    generated_at: "2026-07-27T15:39:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a63bdd4ffd2189cde50f0ee601fd7ee32ca86c943a9899072f0c140823608004
    source_path: providers/deepinfra.md
    workflow: 16
---

DeepInfra درخواست‌ها را از طریق یک نقطه پایانی سازگار با OpenAI و یک کلید API به مدل‌های متن‌باز محبوب و مدل‌های پیشرو هدایت می‌کند. بیشتر SDKهای OpenAI با تغییر URL پایه با آن کار می‌کنند.

## نصب Plugin

```bash
openclaw plugins install @openclaw/deepinfra-provider
openclaw gateway restart
```

## دریافت کلید API

1. در [deepinfra.com](https://deepinfra.com/) وارد شوید
2. به Dashboard / Keys بروید و یک کلید ایجاد کنید، یا از کلیدی که به‌طور خودکار ایجاد شده است استفاده کنید

## راه‌اندازی CLI

```bash
openclaw onboard --deepinfra-api-key <key>
```

یا متغیر محیطی را تنظیم کنید:

```bash
export DEEPINFRA_API_KEY="<your-deepinfra-api-key>" # pragma: allowlist secret
```

## قطعه پیکربندی

```json5
{
  env: { DEEPINFRA_API_KEY: "<your-deepinfra-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "deepinfra/deepseek-ai/DeepSeek-V4-Flash" },
    },
  },
}
```

## سطوح پشتیبانی‌شده

کاتالوگ مدل‌های گفت‌وگو، تولید تصویر و تولید ویدئو پس از پیکربندی `DEEPINFRA_API_KEY`، به‌صورت زنده از `https://api.deepinfra.com/v1/openai/models?sort_by=openclaw&filter=with_meta` به‌روزرسانی می‌شود. کشف زنده فهرست مدل‌های قابل‌انتخاب را گسترش می‌دهد؛ مدل پیش‌فرض هر سطح همان مقدار ایستای زیر باقی می‌ماند. سطوح دیگر تا زمانی که به همین کاتالوگ زنده منتقل شوند، از کاتالوگ‌های ایستا استفاده می‌کنند.

| سطح                      | مدل پیش‌فرض                                                                    | پیکربندی/ابزار OpenClaw                               |
| ------------------------ | ------------------------------------------------------------------------------ | ----------------------------------------------------- |
| گفت‌وگو / ارائه‌دهنده مدل | `deepseek-ai/DeepSeek-V4-Flash` (کاتالوگ زنده مدل‌های گفت‌وگوی بیشتری اضافه می‌کند)         | `agents.defaults.model`                                    |
| تولید/ویرایش تصویر       | `black-forest-labs/FLUX-1-schnell` (کاتالوگ زنده مدل‌های `image-gen` بیشتری اضافه می‌کند) | `image_generate`، `agents.defaults.mediaModels.image`                |
| درک رسانه                | `moonshotai/Kimi-K2.5` برای تصاویر                                                 | درک تصاویر ورودی                                     |
| تبدیل گفتار به متن       | `openai/whisper-large-v3-turbo`                                                            | رونویسی صوت ورودی                                     |
| تبدیل متن به گفتار       | `hexgrad/Kokoro-82M`                                                            | `tts.provider: "deepinfra"`                                    |
| تولید ویدئو              | `Pixverse/Pixverse-T2V` (کاتالوگ زنده مدل‌های `video-gen` بیشتری اضافه می‌کند) | `video_generate`، `agents.defaults.mediaModels.video`                |
| تعبیه‌های حافظه          | `BAAI/bge-m3`                                                            | `memory.search.provider: "deepinfra"`                                    |

DeepInfra همچنین بازرتبه‌بندی، طبقه‌بندی، تشخیص اشیا و دیگر انواع مدل بومی را ارائه می‌دهد. OpenClaw هنوز برای این دسته‌ها قرارداد ارائه‌دهنده‌ای ندارد، بنابراین این Plugin آن‌ها را ثبت نمی‌کند.

## مدل‌های موجود

OpenClaw پس از پیکربندی یک کلید، مدل‌های DeepInfra را به‌صورت پویا کشف می‌کند. برای مشاهده فهرست فعلی از `/models deepinfra` یا `openclaw models list --provider deepinfra` استفاده کنید.

هر مدلی در [deepinfra.com](https://deepinfra.com/) با پیشوند `deepinfra/` کار می‌کند:

```text
deepinfra/deepseek-ai/DeepSeek-V4-Flash
deepinfra/deepseek-ai/DeepSeek-V3.2
deepinfra/MiniMaxAI/MiniMax-M2.5
deepinfra/moonshotai/Kimi-K2.5
deepinfra/nvidia/NVIDIA-Nemotron-3-Super-120B-A12B
deepinfra/zai-org/GLM-5.1
...و بسیاری مدل دیگر
```

## نکات

- ارجاع‌های مدل به‌شکل `deepinfra/<provider>/<model>` هستند (برای مثال `deepinfra/Qwen/Qwen3-Max`).
- مدل پیش‌فرض گفت‌وگو: `deepinfra/deepseek-ai/DeepSeek-V4-Flash`
- URL پایه: `https://api.deepinfra.com/v1/openai`
- تولید ویدئو از نقطه پایانی ناهمگام سازگار با OpenAI یعنی `https://api.deepinfra.com/v1/openai/videos` استفاده می‌کند (ارسال، سپس نظرسنجی). `baseUrl` پیکربندی‌شده رعایت می‌شود. `openclaw doctor --fix` مقادیر قدیمی `nativeBaseUrl` یا `/v1/inference` را در `api.deepinfra.com` به‌طور خودکار به `baseUrl` منتقل می‌کند؛ نقاط پایانی بومی سفارشی با یک اعلان doctor بازنشسته می‌شوند و به `baseUrl` سازگار با OpenAI که به‌صورت دستی پیکربندی شده باشد نیاز دارند. تا زمانی که `baseUrl` همچنان سطح بازنشسته‌شده `/v1/inference` را هدف قرار دهد، تولید ویدئو پیش از ارسال هرگونه درخواست با خطایی همراه با راهکار عملی ناموفق می‌شود.

## مرتبط

- [ارائه‌دهندگان مدل](/fa/concepts/model-providers)
- [همه ارائه‌دهندگان](/fa/providers/index)
