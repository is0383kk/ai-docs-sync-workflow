---
read_when:
    - می‌خواهید از تولید ویدئوی PixVerse در OpenClaw استفاده کنید
    - به تنظیم کلید API و متغیر محیطی PixVerse نیاز دارید
    - می‌خواهید PixVerse را ارائه‌دهنده پیش‌فرض ویدئو قرار دهید
summary: راه‌اندازی تولید ویدیو با PixVerse در OpenClaw
title: PixVerse
x-i18n:
    generated_at: "2026-07-27T15:52:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3dba881e877e3da4677a40dff736cb46de114337a1e0338ef8220dcd8e616f46
    source_path: providers/pixverse.md
    workflow: 16
---

OpenClaw، ‏`pixverse` را به‌عنوان یک Plugin خارجی رسمی برای تولید ویدیوی میزبانی‌شده PixVerse ارائه می‌کند. این Plugin، ارائه‌دهنده `pixverse` را مطابق قرارداد `videoGenerationProviders` ثبت می‌کند.

| ویژگی             | مقدار                                                                |
| ------------------ | -------------------------------------------------------------------- |
| شناسه ارائه‌دهنده  | `pixverse`                                                           |
| بسته Plugin        | `@openclaw/pixverse-provider`                                        |
| متغیر محیطی احراز هویت | `PIXVERSE_API_KEY`                                                   |
| پرچم راه‌اندازی اولیه | `--auth-choice pixverse-api-key`                                     |
| پرچم مستقیم CLI    | `--pixverse-api-key <key>`                                           |
| API                | PixVerse Platform API v2 (ارسال `video_id` به‌همراه پایش نتیجه) |
| مدل پیش‌فرض        | `pixverse/v6`                                                        |
| منطقه پیش‌فرض API  | بین‌المللی                                                        |

## شروع به کار

<Steps>
  <Step title="نصب Plugin">
    ```bash
    openclaw plugins install @openclaw/pixverse-provider
    openclaw gateway restart
    ```
  </Step>
  <Step title="تنظیم کلید API">
    ```bash
    openclaw onboard --auth-choice pixverse-api-key
    ```

    جادوگر پیش از نوشتن `region` و `baseUrl` در پیکربندی ارائه‌دهنده، درباره نقطه پایانی International یا CN درخواست انتخاب می‌کند (بخش منطقه API را
    در ادامه ببینید). اجراهای غیرتعاملی (کلید از `--pixverse-api-key` یا `PIXVERSE_API_KEY`)
    به‌طور پیش‌فرض از International استفاده می‌کنند.

    راه‌اندازی اولیه همچنین در صورتی که هنوز هیچ مدل ویدیویی پیش‌فرضی پیکربندی نشده باشد، `agents.defaults.mediaModels.video.primary` را روی
    `pixverse/v6` تنظیم می‌کند.

  </Step>
  <Step title="تغییر ارائه‌دهنده پیش‌فرض ویدیوی موجود (اختیاری)">
    ```bash
    openclaw config set agents.defaults.mediaModels.video.primary "pixverse/v6"
    ```
  </Step>
  <Step title="تولید ویدیو">
    از عامل بخواهید یک ویدیو تولید کند. PixVerse به‌طور خودکار استفاده خواهد شد.
  </Step>
</Steps>

## حالت‌ها و مدل‌های پشتیبانی‌شده

ارائه‌دهنده، مدل‌های تولید PixVerse را از طریق ابزار ویدیوی مشترک OpenClaw در دسترس قرار می‌دهد.

| حالت           | مدل‌ها               | ورودی مرجع         |
| -------------- | -------------------- | ----------------------- |
| متن به ویدیو  | `v6` (پیش‌فرض)، `c1` | هیچ‌کدام                    |
| تصویر به ویدیو | `v6` (پیش‌فرض)، `c1` | 1 تصویر محلی یا راه‌دور |

ارجاع‌های تصویر محلی پیش از درخواست تصویر به ویدیو در PixVerse بارگذاری می‌شوند. نشانی‌های URL تصویر راه‌دور به‌صورت `image_url` از طریق نقطه پایانی بارگذاری تصویر PixVerse ارسال می‌شوند.

| گزینه           | مقادیر پشتیبانی‌شده                                                                                                                 |
| --------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| مدت‌زمان        | 1-15 ثانیه (پیش‌فرض 5)                                                                                                         |
| وضوح            | `360P`، `540P`، `720P`، `1080P` (پیش‌فرض `540P`؛ درخواست‌های `480P` به `540P` نگاشت می‌شوند)                                                  |
| نسبت تصویر      | `16:9` (پیش‌فرض)، `4:3`، `1:1`، `3:4`، `9:16`، `2:3`، `3:2`، `21:9`؛ فقط برای متن به ویدیو، تصویر به ویدیو از تصویر منبع پیروی می‌کند |
| صدای تولیدشده   | `audio: true`                                                                                                                    |

<Note>
تولید الگوی تصویر PixVerse هنوز از طریق `image_generate` در دسترس نیست. آن API بر اساس شناسه الگو عمل می‌کند، درحالی‌که قرارداد مشترک تولید تصویر OpenClaw در حال حاضر مجموعه گزینه‌های نوع‌دار ویژه PixVerse ندارد.
</Note>

## گزینه‌های ارائه‌دهنده

ارائه‌دهنده ویدیو این کلیدهای اختیاری ویژه ارائه‌دهنده را می‌پذیرد:

| گزینه                                | نوع    | اثر                                           |
| ------------------------------------ | ------ | --------------------------------------------- |
| `seed`                               | عدد | بذر قطعی، از 0 تا 2147483647           |
| `negativePrompt` / `negative_prompt` | رشته | پرامپت منفی                               |
| `quality`                            | رشته | کیفیت PixVerse مانند `720p`               |
| `motionMode` / `motion_mode`         | رشته | حالت حرکت تصویر به ویدیو (پیش‌فرض `normal`) |
| `cameraMovement` / `camera_movement` | رشته | تنظیم از پیش تعیین‌شده حرکت دوربین PixVerse               |
| `templateId` / `template_id`         | عدد | شناسه الگوی فعال‌شده PixVerse                |

## پیکربندی

```json5
{
  agents: {
    defaults: {
      videoGenerationModel: {
        primary: "pixverse/v6",
      },
    },
  },
}
```

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="منطقه API">
    | مقدار منطقه    | نشانی پایه API مربوط به PixVerse                         |
    | --------------- | --------------------------------------------- |
    | `international` | `https://app-api.pixverse.ai/openapi/v2`      |
    | `cn`            | `https://app-api.pixverseai.cn/openapi/v2`    |

    هنگامی که کلید شما به منطقه مشخصی از پلتفرم PixVerse تعلق دارد، `models.providers.pixverse.region` را به‌صورت دستی تنظیم کنید، یا
    `openclaw onboard --auth-choice pixverse-api-key` را اجرا کنید تا یکی را در
    جادوگر راه‌اندازی انتخاب کنید:

    ```json5
    {
      models: {
        providers: {
          pixverse: {
            region: "cn", // "international" or "cn"
            baseUrl: "https://app-api.pixverseai.cn/openapi/v2",
            models: [],
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="نشانی پایه سفارشی">
    فقط هنگام مسیریابی از طریق یک پراکسی سازگار و مورد اعتماد، `models.providers.pixverse.baseUrl` را تنظیم کنید.
    `baseUrl` بر `region` اولویت دارد.

    ```json5
    {
      models: {
        providers: {
          pixverse: {
            baseUrl: "https://app-api.pixverse.ai/openapi/v2",
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="پایش وظیفه">
    PixVerse در پاسخ به درخواست تولید، یک `video_id` برمی‌گرداند. OpenClaw هر 5 ثانیه
    `/openapi/v2/video/result/{video_id}` را پایش می‌کند تا وظیفه
    موفق شود، شکست بخورد یا به مهلت زمانی برسد (پیش‌فرض 5 دقیقه؛ با
    `agents.defaults.mediaModels.video.timeoutMs` بازنویسی کنید).
  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="تولید ویدیو" href="/fa/tools/video-generation" icon="video">
    پارامترهای ابزار مشترک، انتخاب ارائه‌دهنده و رفتار ناهمگام.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/config-agents#agent-defaults" icon="gear">
    تنظیمات پیش‌فرض عامل، از جمله مدل تولید ویدیو.
  </Card>
</CardGroup>
