---
read_when:
    - می‌خواهید از تولید ویدئو با Runway در OpenClaw استفاده کنید
    - به تنظیم کلید API/متغیر محیطی Runway نیاز دارید
    - می‌خواهید Runway را به ارائه‌دهنده پیش‌فرض ویدئو تبدیل کنید
summary: راه‌اندازی تولید ویدئو با Runway در OpenClaw
title: Runway
x-i18n:
    generated_at: "2026-07-27T17:03:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6a56e768893e327b56d70e8b8c2d426123a861b3cf05c0107d98104e2cee856c
    source_path: providers/runway.md
    workflow: 16
---

OpenClaw یک ارائه‌دهندهٔ داخلی `runway` برای تولید ویدیوی میزبانی‌شده عرضه می‌کند که به‌طور پیش‌فرض فعال است و برای قرارداد `videoGenerationProviders` ثبت شده است.

| ویژگی           | مقدار                                                             |
| --------------- | ----------------------------------------------------------------- |
| شناسهٔ ارائه‌دهنده | `runway`                                                          |
| Plugin          | داخلی، `enabledByDefault: true`                                 |
| متغیرهای محیطی احراز هویت | `RUNWAYML_API_SECRET` (استاندارد) یا `RUNWAY_API_KEY`             |
| پرچم راه‌اندازی اولیه | `--auth-choice runway-api-key`                                    |
| پرچم مستقیم CLI | `--runway-api-key <key>`                                          |
| API             | تولید ویدیوی مبتنی بر وظیفهٔ Runway (نظرسنجی `GET /v1/tasks/{id}`) |
| مدل پیش‌فرض     | `runway/gen4.5`                                                   |

## شروع به کار

<Steps>
  <Step title="تنظیم کلید API">
    ```bash
    openclaw onboard --auth-choice runway-api-key
    ```
  </Step>
  <Step title="تنظیم Runway به‌عنوان ارائه‌دهندهٔ پیش‌فرض ویدیو">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "runway/gen4.5"
    ```
  </Step>
  <Step title="تولید یک ویدیو">
    از عامل بخواهید یک ویدیو تولید کند. Runway به‌طور خودکار استفاده خواهد شد.
  </Step>
</Steps>

## حالت‌ها و مدل‌های پشتیبانی‌شده

این ارائه‌دهنده هفت مدل Runway را در سه حالت ارائه می‌کند. یک شناسهٔ مدل می‌تواند برای بیش از یک حالت به‌کار رود (برای مثال، `gen4.5` هم برای تبدیل متن به ویدیو و هم تصویر به ویدیو کار می‌کند).

| حالت           | مدل‌ها                                                                 | ورودی مرجع         |
| -------------- | ---------------------------------------------------------------------- | ----------------------- |
| متن به ویدیو  | `gen4.5` (پیش‌فرض)، `veo3.1`، `veo3.1_fast`، `veo3`                    | ندارد                    |
| تصویر به ویدیو | `gen4.5`، `gen4_turbo`، `gen3a_turbo`، `veo3.1`، `veo3.1_fast`، `veo3` | 1 تصویر محلی یا راه‌دور |
| ویدیو به ویدیو | `gen4_aleph`                                                           | 1 ویدیوی محلی یا راه‌دور |

ارجاع‌های محلی تصویر و ویدیو از طریق URIهای داده پشتیبانی می‌شوند.

| نسبت‌های تصویر         | مقادیر مجاز                              |
| --------------------- | ------------------------------------------- |
| متن به ویدیو         | `16:9`، `9:16`                              |
| ویرایش تصویر و ویدیو | `1:1`، `16:9`، `9:16`، `3:4`، `4:3`، `21:9` |

<Warning>
  تبدیل ویدیو به ویدیو در حال حاضر به `runway/gen4_aleph` نیاز دارد. سایر شناسه‌های مدل Runway ورودی‌های مرجع ویدیو را رد می‌کنند.
</Warning>

<Note>
  انتخاب شناسهٔ مدل Runway از ستون نادرست، پیش از خروج درخواست API از OpenClaw خطایی صریح ایجاد می‌کند. ارائه‌دهنده در `extensions/runway/video-generation-provider.ts`، `model` را با فهرست مجاز حالت (`TEXT_ONLY_MODELS`، `IMAGE_MODELS`، `VIDEO_MODELS`) اعتبارسنجی می‌کند.
</Note>

## پیکربندی

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "runway/gen4.5",
      },
    },
  },
}
```

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="نام‌های مستعار متغیر محیطی">
    OpenClaw هر دو `RUNWAYML_API_SECRET` (استاندارد) و `RUNWAY_API_KEY` را می‌شناسد.
    هر یک از این متغیرها ارائه‌دهندهٔ Runway را احراز هویت می‌کند.
  </Accordion>

  <Accordion title="نظرسنجی وظیفه">
    Runway از یک API مبتنی بر وظیفه استفاده می‌کند. پس از ارسال درخواست تولید، OpenClaw
    تا آماده‌شدن ویدیو، `GET /v1/tasks/{id}` را نظرسنجی می‌کند. برای رفتار نظرسنجی به
    پیکربندی دیگری نیاز نیست.
  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="تولید ویدیو" href="/fa/tools/video-generation" icon="video">
    پارامترهای مشترک ابزار، انتخاب ارائه‌دهنده و رفتار ناهمگام.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/config-agents#agent-defaults" icon="gear">
    تنظیمات پیش‌فرض عامل، از جمله مدل تولید ویدیو.
  </Card>
</CardGroup>
