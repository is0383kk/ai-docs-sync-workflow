---
read_when:
    - می‌خواهید از MiniMax برای web_search استفاده کنید
    - به یک کلید MiniMax Token Plan یا توکن OAuth نیاز دارید
    - راهنمای میزبان جست‌وجوی MiniMax چین/جهانی را می‌خواهید
summary: جست‌وجوی MiniMax از طریق API جست‌وجوی Token Plan
title: جست‌وجوی MiniMax
x-i18n:
    generated_at: "2026-07-27T14:44:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cb851614bbe43f011e07fe3e80d5390f1ba515f3e00ba749c91999617ad2d1e2
    source_path: tools/minimax-search.md
    workflow: 16
---

OpenClaw از MiniMax به‌عنوان ارائه‌دهندهٔ `web_search` از طریق API جست‌وجوی MiniMax
Token Plan پشتیبانی می‌کند. این API نتایج جست‌وجوی ساختاریافته‌ای شامل عنوان‌ها، URLها،
خلاصه‌ها و پرس‌وجوهای مرتبط برمی‌گرداند.

## دریافت اعتبارنامهٔ Token Plan

<Steps>
  <Step title="ایجاد کلید">
    یک کلید MiniMax Token Plan را در
    [پلتفرم MiniMax](https://platform.minimax.io/user-center/basic-information/interface-key) ایجاد یا کپی کنید.
    در راه‌اندازی‌های OAuth می‌توان به‌جای آن از `MINIMAX_OAUTH_TOKEN` استفاده کرد.
  </Step>
  <Step title="ذخیرهٔ کلید">
    `MINIMAX_CODE_PLAN_KEY` را در محیط Gateway تنظیم کنید یا از طریق دستور زیر پیکربندی کنید:

    ```bash
    openclaw configure --section web
    ```

  </Step>
</Steps>

OpenClaw همچنین `MINIMAX_CODING_API_KEY`، `MINIMAX_OAUTH_TOKEN` و
`MINIMAX_API_KEY` را به‌عنوان نام‌های مستعار متغیر محیطی می‌پذیرد که پس از
`MINIMAX_CODE_PLAN_KEY` به همین ترتیب بررسی می‌شوند. `MINIMAX_API_KEY` باید به یک
اعتبارنامهٔ Token Plan دارای قابلیت جست‌وجو اشاره کند؛ ممکن است کلیدهای معمولی API مدل
MiniMax از سوی نقطهٔ پایانی جست‌وجوی Token Plan پذیرفته نشوند.

## پیکربندی

```json5
{
  plugins: {
    entries: {
      minimax: {
        config: {
          webSearch: {
            apiKey: "sk-cp-...", // اختیاری است، اگر متغیر محیطی MiniMax Token Plan تنظیم شده باشد
            region: "global", // یا "cn"
          },
        },
      },
    },
  },
  tools: {
    web: {
      search: {
        provider: "minimax",
      },
    },
  },
}
```

**روش جایگزین با متغیر محیطی:** `MINIMAX_CODE_PLAN_KEY`، `MINIMAX_CODING_API_KEY`،
`MINIMAX_OAUTH_TOKEN` یا `MINIMAX_API_KEY` را در محیط Gateway تنظیم کنید.
برای نصب Gateway، آن را در `~/.openclaw/.env` قرار دهید.

## انتخاب منطقه

جست‌وجوی MiniMax از این نقاط پایانی استفاده می‌کند:

- جهانی: `https://api.minimax.io/v1/coding_plan/search`
- چین: `https://api.minimaxi.com/v1/coding_plan/search`

اگر `plugins.entries.minimax.config.webSearch.region` تنظیم نشده باشد، OpenClaw
منطقه را به ترتیب زیر تعیین می‌کند:

1. `webSearch.region` متعلق به Plugin
2. `MINIMAX_API_HOST`
3. `models.providers.minimax.baseUrl`
4. `models.providers.minimax-portal.baseUrl`

این یعنی فرایند راه‌اندازی چین یا `MINIMAX_API_HOST=https://api.minimaxi.com/...`
به‌طور خودکار جست‌وجوی MiniMax را نیز روی میزبان چین نگه می‌دارد.

حتی اگر احراز هویت MiniMax را از طریق مسیر OAuth ‏`minimax-portal` انجام داده باشید،
جست‌وجوی وب همچنان با شناسهٔ ارائه‌دهندهٔ `minimax` ثبت می‌شود؛ URL پایهٔ ارائه‌دهندهٔ OAuth
به‌عنوان راهنمای منطقه برای انتخاب میزبان چین/جهانی استفاده می‌شود و `MINIMAX_OAUTH_TOKEN`
می‌تواند اعتبارنامهٔ bearer جست‌وجوی MiniMax را تأمین کند.

## پارامترهای پشتیبانی‌شده

| پارامتر | نوع     | محدودیت‌ها     | توضیحات                                                                        |
| --------- | ------- | --------------- | --------------------------------------------------------------------------- |
| `query`   | رشته  | الزامی        | رشتهٔ پرس‌وجوی جست‌وجو.                                                        |
| `count`   | عدد صحیح | 1-10، پیش‌فرض 5 | تعداد نتایجی که برگردانده می‌شوند. OpenClaw فهرست بازگشتی را به این اندازه محدود می‌کند. |

در حال حاضر فیلترهای مختص ارائه‌دهنده پشتیبانی نمی‌شوند.

## مرتبط

- [نمای کلی جست‌وجوی وب](/fa/tools/web) -- همهٔ ارائه‌دهندگان و تشخیص خودکار
- [MiniMax](/fa/providers/minimax) -- راه‌اندازی مدل، تصویر، گفتار و احراز هویت
