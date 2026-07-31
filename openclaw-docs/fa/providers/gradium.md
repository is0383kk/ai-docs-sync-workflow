---
read_when:
    - شما Gradium را برای تبدیل متن به گفتار می‌خواهید
    - به پیکربندی کلید API، صدا یا توکن دستور Gradium نیاز دارید
summary: استفاده از تبدیل متن به گفتار Gradium در OpenClaw
title: Gradium
x-i18n:
    generated_at: "2026-07-27T17:03:25Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5536426eb6d3c8f24c04643b033ebb519a1f2f9df9d97c917ced1c7e23ad180d
    source_path: providers/gradium.md
    workflow: 16
---

[Gradium](https://gradium.ai) یک ارائه‌دهندهٔ تبدیل متن به گفتار برای OpenClaw است. این سرویس پاسخ‌های صوتی استاندارد (WAV)، خروجی Opus سازگار با پیام صوتی و صدای u-law با نرخ 8 kHz را برای بسترهای تلفنی تولید می‌کند.

| ویژگی      | مقدار                                |
| ------------- | ------------------------------------ |
| شناسهٔ ارائه‌دهنده   | `gradium`                            |
| احراز هویت          | `GRADIUM_API_KEY` یا پیکربندی `apiKey` |
| نشانی پایه      | `https://api.gradium.ai` (پیش‌فرض)   |
| صدای پیش‌فرض | `Emma` (`YTpq7expH9539ERJ`)          |

## نصب Plugin

Gradium یک Plugin خارجی رسمی است. آن را نصب کنید، سپس Gateway را راه‌اندازی مجدد کنید:

```bash
openclaw plugins install @openclaw/gradium-speech
openclaw gateway restart
```

## راه‌اندازی

یک کلید API در Gradium ایجاد کنید، سپس آن را از طریق یک متغیر محیطی یا کلید پیکربندی در دسترس قرار دهید. پیکربندی بر متغیر محیطی اولویت دارد.

<Tabs>
  <Tab title="متغیر محیطی">
    ```bash
    export GRADIUM_API_KEY="gsk_..."
    ```
  </Tab>

  <Tab title="کلید پیکربندی">
    ```json5
    {
      tts: {
        auto: "always",
        provider: "gradium",
        providers: {
          gradium: {
            apiKey: "${GRADIUM_API_KEY}",
          },
        },
      },
    }
    ```
  </Tab>
</Tabs>

## پیکربندی

```json5
{
  tts: {
    auto: "always",
    provider: "gradium",
    providers: {
      gradium: {
        speakerVoiceId: "YTpq7expH9539ERJ",
        // apiKey: "${GRADIUM_API_KEY}",
        // baseUrl: "https://api.gradium.ai",
      },
    },
  },
}
```

| کلید                                    | نوع   | توضیحات                                                                                             |
| -------------------------------------- | ------ | ------------------------------------------------------------------------------------------------------- |
| `tts.providers.gradium.apiKey`         | رشته | کلید API نهایی. از `${ENV}` و ارجاع‌های محرمانه پشتیبانی می‌کند.                                                    |
| `tts.providers.gradium.baseUrl`        | رشته | نشانی HTTPS مربوط به API Gradium در `api.gradium.ai`. اسلش‌های انتهایی حذف می‌شوند. پیش‌فرض: `https://api.gradium.ai`. |
| `tts.providers.gradium.speakerVoiceId` | رشته | شناسهٔ صدای پیش‌فرض که در صورت نبود بازنویسی با دستورالعمل استفاده می‌شود.                                            |

قالب خروجی به‌طور خودکار بر اساس بستر مقصد انتخاب می‌شود (به [خروجی](#output) مراجعه کنید) و در `openclaw.json` قابل پیکربندی نیست.

## صداها

| نام               | شناسهٔ صدا           |
| ------------------ | ------------------ |
| Arthur             | `3jUdJyOi9pgbxBTK` |
| Christina          | `2H4HY2CBNyJHBCrP` |
| Emma **(پیش‌فرض)** | `YTpq7expH9539ERJ` |
| John               | `KWJiFWu2O9nMPYcR` |
| Kent               | `LFZvm12tW_z0xfGo` |
| Sydney             | `jtEKaLYNn6iif5PR` |
| Tiffany            | `Eu9iL_CYe8N-Gkx_` |

### بازنویسی صدا برای هر پیام

هنگامی که سیاست فعال گفتار اجازهٔ بازنویسی صدا را می‌دهد، با استفاده از یک توکن دستورالعمل، صدا را درون‌خطی تغییر دهید (همهٔ موارد زیر معادل‌اند و همگی شناسهٔ صدای بومی ارائه‌دهنده را می‌پذیرند):

```text
/voice:LFZvm12tW_z0xfGo
/voice_id:LFZvm12tW_z0xfGo
/voiceid:LFZvm12tW_z0xfGo
/gradium_voice:LFZvm12tW_z0xfGo
/gradiumvoice:LFZvm12tW_z0xfGo
```

اگر سیاست گفتار بازنویسی صدا را غیرفعال کند، دستورالعمل پردازش می‌شود، اما نادیده گرفته می‌شود.

## خروجی

قالب خروجی بر اساس بستر مقصد انتخاب می‌شود؛ ارائه‌دهنده قالب‌های دیگری تولید نمی‌کند.

| مقصد         | قالب      | پسوند فایل | نرخ نمونه‌برداری | پرچم سازگاری با صدا |
| -------------- | ----------- | -------- | ----------- | --------------------- |
| صدای استاندارد | `wav`       | `.wav`   | ارائه‌دهنده    | خیر                    |
| پیام صوتی     | `opus`      | `.opus`  | ارائه‌دهنده    | بله                   |
| تلفن      | `ulaw_8000` | نامرتبط      | 8 kHz       | نامرتبط                   |

## ترتیب انتخاب خودکار

در میان ارائه‌دهندگان پیکربندی‌شدهٔ TTS، ترتیب انتخاب خودکار Gradium برابر با `30` است. برای آگاهی از نحوهٔ انتخاب ارائه‌دهندهٔ فعال توسط OpenClaw هنگامی که `tts.provider` ثابت نشده است، به [تبدیل متن به گفتار](/fa/tools/tts) مراجعه کنید.

## مرتبط

- [تبدیل متن به گفتار](/fa/tools/tts)
- [نمای کلی رسانه](/fa/tools/media-overview)
