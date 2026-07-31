---
read_when:
    - می‌خواهید از تولید تصویر fal در OpenClaw استفاده کنید
    - به جریان احراز هویت FAL_KEY نیاز دارید
    - برای image_generate، video_generate یا music_generate پیش‌فرض‌های fal را می‌خواهید
summary: راه‌اندازی تولید تصویر، ویدئو و موسیقی با fal در OpenClaw
title: Fal
x-i18n:
    generated_at: "2026-07-27T14:30:55Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9bd868aaf6771f6fa38bb8e2a83133460d150e2a5aa9e5b888e221c07f29e0ad
    source_path: providers/fal.md
    workflow: 16
---

OpenClaw یک ارائه‌دهندهٔ داخلی `fal` برای تولید میزبانی‌شدهٔ تصویر، ویدئو و موسیقی
ارائه می‌کند.

| ویژگی | مقدار                                                                           |
| -------- | ------------------------------------------------------------------------------- |
| ارائه‌دهنده | `fal`                                                                           |
| احراز هویت     | `FAL_KEY` (متعارف؛ `FAL_API_KEY` به‌عنوان مسیر جایگزین نیز کار می‌کند)                   |
| API      | نقاط پایانی مدل fal (`https://fal.run`؛ کارهای ویدئویی از `https://queue.fal.run` استفاده می‌کنند) |
| نشانی پایه | با `models.providers.fal.baseUrl` بازنویسی کنید                                    |

## شروع به کار

<Steps>
  <Step title="تنظیم کلید API">
    ```bash
    openclaw onboard --auth-choice fal-api-key
    ```

    راه‌اندازی‌های غیرتعاملی می‌توانند `--fal-api-key <key>` را ارسال یا `FAL_KEY` را صادر کنند.
    فرایند راه‌اندازی، در صورت پیکربندی‌نشدن هیچ مدلی، `fal/fal-ai/flux/dev` را نیز به‌عنوان
    مدل پیش‌فرض تصویر تنظیم می‌کند.

  </Step>
  <Step title="تنظیم مدل پیش‌فرض تصویر">
    ```json5
    {
      agents: {
        defaults: {
          imageGenerationModel: {
            primary: "fal/fal-ai/flux/dev",
          },
        },
      },
    }
    ```
  </Step>
</Steps>

## تولید تصویر

ارائه‌دهندهٔ داخلی `fal` برای تولید تصویر به‌طور پیش‌فرض از
`fal/fal-ai/flux/dev` استفاده می‌کند.

| قابلیت     | مقدار                                                              |
| -------------- | ------------------------------------------------------------------ |
| حداکثر تصاویر     | 4 در هر درخواست؛ Krea 2:‏ 1 در هر درخواست                               |
| بازنویسی‌های اندازه | `1024x1024`، `1024x1536`، `1536x1024`، `1024x1792`، `1792x1024`    |
| نسبت ابعاد   | همه‌جا به‌جز تبدیل تصویر به تصویر Flux پشتیبانی می‌شود                    |
| وضوح     | `1K`، `2K`، `4K` (محدودیت‌های هر مدل در ادامه آمده است)                          |
| قالب خروجی  | `png` (پیش‌فرض) یا `jpeg`؛ Krea 2 بازنویسی‌های `outputFormat` را رد می‌کند |

درخواست‌های ویرایش (تصاویر مرجع از طریق پارامترهای مشترک `image` / `images`)
به یک نقطهٔ پایانی ویرایش مختص هر مدل، با محدودیت‌های مرجع مختص همان مدل هدایت می‌شوند:

| خانوادهٔ مدل              | مرجع مدل پس از `fal/`                 | نقطهٔ پایانی ویرایش     | حداکثر تصاویر مرجع |
| ------------------------- | -------------------------------------- | ----------------- | -------------------- |
| Flux و دیگر مدل‌های fal | `fal-ai/flux/dev` (پیش‌فرض)            | `/image-to-image` | 1                    |
| GPT Image                 | `openai/gpt-image-*`                   | `/edit`           | 10                   |
| Grok Imagine              | `xai/grok-imagine-image`               | `/edit`           | 3                    |
| Nano Banana (قدیمی)      | `fal-ai/nano-banana`                   | `/edit`           | 3                    |
| Nano Banana 2             | `fal-ai/nano-banana-*`                 | `/edit`           | 14                   |
| Nano Banana 2 Lite        | `google/nano-banana-2-lite`            | `/edit`           | 14                   |
| Krea 2                    | `krea/v2/{medium,large}/text-to-image` | ندارد (مراجع سبک) | 10 مرجع سبک  |

<Warning>
درخواست‌های تبدیل تصویر به تصویر Flux از بازنویسی‌های `aspectRatio` پشتیبانی **نمی‌کنند**. درخواست‌های
ویرایش GPT Image و Nano Banana 2 از نقطهٔ پایانی `/edit` متعلق به fal استفاده می‌کنند و
راهنماهای نسبت ابعاد را می‌پذیرند. Nano Banana 2 نسبت‌های عریض/بلند فراتر از محدودهٔ بومی
مانند `4:1`، `1:4`، `8:1` و `1:8` را نیز می‌پذیرد؛ Krea 2 مجموعهٔ کوچک‌تر
نسبت‌های ابعاد خود را اعتبارسنجی می‌کند. Grok Imagine فهرست نسبت‌های مختص خود را دارد (شامل `2:1`،
`20:9`، `19.5:9` و معکوس‌های آن‌ها) و فقط وضوح‌های `1K`/`2K` را می‌پذیرد؛
Nano Banana قدیمی و Nano Banana 2 Lite بازنویسی‌های `resolution` را رد می‌کنند.
</Warning>

مدل‌های Krea 2 از طرح‌وارهٔ بومی بار دادهٔ Krea در fal استفاده می‌کنند. OpenClaw به‌جای
بار دادهٔ عمومی `image_size` / نقطهٔ پایانی ویرایش که Flux استفاده می‌کند،
`aspect_ratio`، `creativity` و `image_style_references` را ارسال می‌کند. مراجع مدل عبارت‌اند از:

- `fal/krea/v2/medium/text-to-image`
- `fal/krea/v2/large/text-to-image`

برای تصویرسازی بیانگر، انیمه، نقاشی و سبک‌های هنری سریع‌تر از Medium
استفاده کنید. برای ظاهرهای واقع‌گرایانه، بافت خام، دانه‌بندی فیلم و پرجزئیات که کندتر تولید می‌شوند، از Large
استفاده کنید. مقدار پیش‌فرض Krea برابر `fal.creativity: "medium"` است؛ مقادیر پشتیبانی‌شده
`raw`، `low`، `medium` و `high` هستند.

Krea 2 در طرح‌وارهٔ درخواست fal نسبت ابعاد را ارائه می‌کند، نه `image_size`. ترجیحاً از
`aspectRatio` استفاده کنید؛ OpenClaw مقدار `size` را به نزدیک‌ترین نسبت ابعاد پشتیبانی‌شدهٔ Krea نگاشت
می‌کند و به‌جای حذف `resolution`، آن را برای Krea رد می‌کند.

هنگامی که خروجی PNG را از مدل‌های fal ارائه‌کنندهٔ
`output_format` می‌خواهید، از `outputFormat: "png"` استفاده کنید. fal در OpenClaw کنترل صریحی
برای پس‌زمینهٔ شفاف اعلام نمی‌کند؛ بنابراین `background: "transparent"` برای مدل‌های fal
به‌عنوان بازنویسی نادیده‌گرفته‌شده گزارش می‌شود.
نقاط پایانی Krea 2 فیلد درخواست `output_format` را از طریق fal ارائه نمی‌کنند؛ بنابراین
OpenClaw بازنویسی‌های `outputFormat` را برای درخواست‌های Krea رد می‌کند.

برای استفاده از Krea 2 Medium:

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

## تولید ویدئو

ارائه‌دهندهٔ داخلی `fal` برای تولید ویدئو به‌طور پیش‌فرض از
`fal/fal-ai/minimax/video-01-live` استفاده می‌کند.

| قابلیت | مقدار                                                              |
| ---------- | ------------------------------------------------------------------ |
| حالت‌ها      | متن به ویدئو، مرجع تک‌تصویری، مرجع به ویدئوی Seedance |
| زمان اجرا    | جریان ارسال/وضعیت/نتیجهٔ مبتنی بر صف برای کارهای طولانی‌مدت       |
| مهلت زمانی    | به‌طور پیش‌فرض 20 دقیقه برای هر کار؛ وضعیت هر 5 ثانیه بررسی می‌شود       |

<AccordionGroup>
  <Accordion title="مدل‌های ویدئویی موجود">
    **MiniMax (پیش‌فرض):**

    - `fal/fal-ai/minimax/video-01-live`

    **عامل ویدئویی HeyGen:**

    - `fal/fal-ai/heygen/v2/video-agent`

    **Kling و Wan:**

    - `fal/fal-ai/kling-video/v2.1/master/text-to-video`
    - `fal/fal-ai/wan/v2.2-a14b/text-to-video`
    - `fal/fal-ai/wan/v2.2-a14b/image-to-video`

    **Seedance 2.0:**

    - `fal/bytedance/seedance-2.0/fast/text-to-video`
    - `fal/bytedance/seedance-2.0/fast/image-to-video`
    - `fal/bytedance/seedance-2.0/fast/reference-to-video`
    - `fal/bytedance/seedance-2.0/text-to-video`
    - `fal/bytedance/seedance-2.0/image-to-video`
    - `fal/bytedance/seedance-2.0/reference-to-video`

    درخواست‌های MiniMax Live و HeyGen فقط اعلان را به‌همراه یک
    تصویر مرجع اختیاری ارسال می‌کنند؛ دیگر بازنویسی‌ها ارسال نمی‌شوند. مدل‌های Seedance
    مقادیر `aspectRatio`، `size`، `resolution`، مدت‌زمان‌های 4 تا 15 ثانیه و
    یک کلید تغییر وضعیت صدا را می‌پذیرند.

  </Accordion>

  <Accordion title="نمونهٔ پیکربندی Seedance 2.0">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/bytedance/seedance-2.0/fast/text-to-video",
          },
        },
      },
    }
    ```
  </Accordion>

  <Accordion title="نمونهٔ پیکربندی مرجع به ویدئوی Seedance 2.0">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/bytedance/seedance-2.0/fast/reference-to-video",
          },
        },
      },
    }
    ```

    مرجع به ویدئو از طریق پارامترهای مشترک `video_generate`، `images`، `videos` و
    `audioRefs` حداکثر 9 تصویر، 3 ویدئو و 3 مرجع صوتی را می‌پذیرد و حداکثر
    12 فایل مرجع در مجموع مجاز است. مراجع صوتی مستلزم وجود
    حداقل یک مرجع تصویری یا ویدئویی در همان درخواست هستند.

  </Accordion>

  <Accordion title="نمونهٔ پیکربندی عامل ویدئویی HeyGen">
    ```json5
    {
      agents: {
        defaults: {
          videoGenerationModel: {
            primary: "fal/fal-ai/heygen/v2/video-agent",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## تولید موسیقی

Plugin داخلی `fal` یک ارائه‌دهندهٔ تولید موسیقی نیز برای
ابزار مشترک `music_generate` ثبت می‌کند.

| قابلیت    | مقدار                                                                                                                    |
| ------------- | ------------------------------------------------------------------------------------------------------------------------ |
| مدل پیش‌فرض | `fal/fal-ai/minimax-music/v2.6`                                                                                          |
| مدل‌ها        | `fal-ai/minimax-music/v2.6` (mp3)، `fal-ai/ace-step/prompt-to-audio` (wav)، `fal-ai/stable-audio-25/text-to-audio` (wav) |
| حداکثر مدت‌زمان  | 240 ثانیه                                                                                                              |
| زمان اجرا       | درخواست همگام به‌همراه بارگیری صدای تولیدشده                                                                        |

از fal به‌عنوان ارائه‌دهندهٔ پیش‌فرض موسیقی استفاده کنید:

```json5
{
  agents: {
    defaults: {
      musicGenerationModel: {
        primary: "fal/fal-ai/minimax-music/v2.6",
      },
    },
  },
}
```

`fal-ai/minimax-music/v2.6` از متن ترانهٔ صریح و حالت بی‌کلام پشتیبانی می‌کند،
اما نه هر دو در یک درخواست. ACE-Step و Stable Audio
نقاط پایانی اعلان به صدا هستند؛ هنگامی که آن خانواده‌های مدل را می‌خواهید، آن‌ها را با بازنویسی
`model` انتخاب کنید. ACE-Step متن ترانهٔ صریح را رد می‌کند؛ Stable Audio
هم متن ترانه و هم حالت بی‌کلام را رد می‌کند.

<Tip>
جدول‌ها و بخش‌های بازشوندهٔ بالا خانواده‌های مدلی را پوشش می‌دهند که ارائه‌دهندهٔ داخلی fal
به‌طور ویژه پردازش می‌کند. شناسه‌های دیگر نقاط پایانی تصویر fal را همچنان می‌توان به‌عنوان
مدل تصویر انتخاب کرد؛ با آن‌ها مانند Flux رفتار می‌شود (بار دادهٔ عمومی `image_size`، یک
تصویر مرجع از طریق `/image-to-image`).
</Tip>

## مرتبط

<CardGroup cols={2}>
  <Card title="تولید تصویر" href="/fa/tools/image-generation" icon="image">
    پارامترهای ابزار مشترک تصویر و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="تولید ویدئو" href="/fa/tools/video-generation" icon="video">
    پارامترهای ابزار مشترک ویدئو و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="تولید موسیقی" href="/fa/tools/music-generation" icon="music">
    پارامترهای ابزار مشترک موسیقی و انتخاب ارائه‌دهنده.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/config-agents#agent-defaults" icon="gear">
    پیش‌فرض‌های عامل، شامل انتخاب مدل تصویر، ویدئو و موسیقی.
  </Card>
</CardGroup>
