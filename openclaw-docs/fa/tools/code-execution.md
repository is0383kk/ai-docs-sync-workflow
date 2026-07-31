---
read_when:
    - می‌خواهید code_execution را فعال یا پیکربندی کنید
    - تحلیل از راه دور را بدون دسترسی به پوستهٔ محلی می‌خواهید
    - می‌خواهید x_search یا web_search را با تحلیل از راه دور Python ترکیب کنید
summary: 'code_execution: اجرای تحلیل Python راه‌دور در محیط ایزوله با xAI'
title: اجرای کد
x-i18n:
    generated_at: "2026-07-27T14:40:08Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1ab391daed9154f113535e6d241c45d5c08c22abdc012148a9f0f2ae5ec548b3
    source_path: tools/code-execution.md
    workflow: 16
---

`code_execution` تحلیل راه‌دور و ایزوله‌شدهٔ Python را روی Responses API شرکت xAI اجرا می‌کند
(`https://api.x.ai/v1/responses`، همان نقطهٔ پایانی که `x_search` استفاده می‌کند). این ابزار
توسط Plugin همراه `xai` تحت قرارداد `tools` ثبت می‌شود.

<Warning>
  `code_execution` روی سرورهای xAI اجرا می‌شود. xAI به‌ازای هر ۱٬۰۰۰ فراخوانی ابزار، $5
  به‌علاوهٔ توکن‌های ورودی و خروجی مدل هزینه دریافت می‌کند.
</Warning>

| ویژگی           | مقدار                                                                             |
| ------------------ | --------------------------------------------------------------------------------- |
| نام ابزار          | `code_execution`                                                                  |
| Plugin ارائه‌دهنده    | `xai` (همراه، `enabledByDefault: true`)                                         |
| احراز هویت               | پروفایل احراز هویت xAI، `XAI_API_KEY`، یا `plugins.entries.xai.config.webSearch.apiKey` |
| مدل پیش‌فرض      | `grok-4.3`                                                                        |
| مهلت زمانی پیش‌فرض    | 30 ثانیه                                                                        |
| `maxTurns` پیش‌فرض | تنظیم‌نشده (xAI محدودیت داخلی خودش را اعمال می‌کند)                                        |

از آن برای محاسبات، جدول‌بندی، آمار سریع و تحلیل به‌سبک نمودار،
از جمله تحلیل داده‌های برگردانده‌شده توسط `x_search` یا `web_search` استفاده کنید. این ابزار به
فایل‌های محلی، پوسته، مخزن یا دستگاه‌های جفت‌شدهٔ شما دسترسی ندارد و
وضعیت را میان فراخوانی‌ها نگه نمی‌دارد؛ بنابراین هر فراخوانی را تحلیلی موقتی بدانید، نه
یک نشست دفترچه. برای داده‌های تازهٔ X، ابتدا [`x_search`](/fa/tools/web#x_search)
را اجرا کنید و نتیجه را به آن انتقال دهید.

برای اجرای محلی، به‌جای آن از [`exec`](/fa/tools/exec) استفاده کنید.

## راه‌اندازی

<Steps>
  <Step title="ارائهٔ اطلاعات احراز هویت xAI">
    OAuth به اشتراک واجد شرایط SuperGrok یا X Premium نیاز دارد
    (با تأیید کد دستگاه، بنابراین از میزبان‌های راه‌دور بدون
    فراخوانی برگشتی localhost نیز کار می‌کند):

    ```bash
    openclaw models auth login --provider xai --method oauth
    ```

    هنگام نصب تازه، همین گزینه در فرایند راه‌اندازی اولیه در دسترس است:

    ```bash
    openclaw onboard --install-daemon --auth-choice xai-oauth
    ```

    یا با یک کلید API:

    ```bash
    openclaw models auth login --provider xai --method api-key
    export XAI_API_KEY=xai-...
    ```

    یا از طریق پیکربندی:

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              webSearch: {
                apiKey: "xai-...",
              },
            },
          },
        },
      },
    }
    ```

    هر یک از این سه روش، `x_search` و `web_search` مربوط به Grok را نیز فعال می‌کند.

  </Step>

  <Step title="فعال‌سازی و تنظیم code_execution">
    وقتی `enabled` حذف شده باشد، `code_execution` فقط زمانی در دسترس قرار می‌گیرد که ارائه‌دهندهٔ
    مدل فعال `xai` باشد و اطلاعات احراز هویت xAI با موفقیت یافت شود. برای یک مدل فعال
    با ارائه‌دهندهٔ شناخته‌شدهٔ غیر xAI،
    `plugins.entries.xai.config.codeExecution.enabled` را روی `true` تنظیم کنید تا استفادهٔ
    میان‌ارائه‌دهنده‌ای فعال شود. اگر ارائه‌دهندهٔ مدل فعال مشخص نباشد یا یافت نشود،
    ابزار پنهان می‌ماند. برای غیرفعال‌کردن آن برای همهٔ
    ارائه‌دهندگان، `enabled` را روی `false` تنظیم کنید. اطلاعات احراز هویت xAI همیشه الزامی است.

    برای تغییر مدل، سقف نوبت‌ها یا مهلت زمانی نیز از همین بلوک استفاده کنید:

    ```json5
    {
      plugins: {
        entries: {
          xai: {
            config: {
              codeExecution: {
                enabled: true, // برای یک ارائه‌دهندهٔ مدل شناخته‌شدهٔ غیر xAI الزامی است
                model: "grok-4.3", // جایگزینی مدل پیش‌فرض اجرای کد xAI
                maxTurns: 2,            // سقف اختیاری نوبت‌های داخلی ابزار
                timeoutSeconds: 30,     // مهلت زمانی درخواست (پیش‌فرض: 30)
              },
            },
          },
        },
      },
    }
    ```

  </Step>

  <Step title="راه‌اندازی مجدد Gateway">
    ```bash
    openclaw gateway restart
    ```

    پس از ثبت مجدد Plugin ‏xAI و موفقیت بررسی‌های ارائه‌دهنده،
    فعال‌سازی و احراز هویت بالا، `code_execution` در فهرست ابزارهای عامل ظاهر می‌شود.

  </Step>
</Steps>

## نحوهٔ استفاده

هدف تحلیل را صریح بیان کنید؛ ابزار تنها یک پارامتر `task` می‌گیرد،
بنابراین درخواست کامل و هرگونه دادهٔ درون‌خطی را در یک پرامپت ارسال کنید:

```text
از code_execution برای محاسبهٔ میانگین متحرک 7روزهٔ این اعداد استفاده کن: ...
```

```text
از x_search برای یافتن پست‌هایی که این هفته به OpenClaw اشاره کرده‌اند استفاده کن، سپس با code_execution آن‌ها را بر اساس روز بشمار.
```

```text
از web_search برای گردآوری تازه‌ترین اعداد بنچمارک هوش مصنوعی استفاده کن، سپس با code_execution تغییرات درصدی را مقایسه کن.
```

## خطاها

بدون احراز هویت، ابزار یک خطای JSON ساختاریافته برمی‌گرداند (نه یک
استثنای پرتاب‌شده)، بنابراین عامل می‌تواند خودش مشکل را اصلاح کند:

```json
{
  "error": "missing_xai_api_key",
  "message": "code_execution به اطلاعات احراز هویت xAI نیاز دارد. برای ورود با Grok، `openclaw onboard --auth-choice xai-oauth` را اجرا کنید؛ `openclaw onboard --auth-choice xai-api-key` را اجرا کنید؛ `XAI_API_KEY` را در محیط Gateway تنظیم کنید؛ یا `plugins.entries.xai.config.webSearch.apiKey` را پیکربندی کنید.",
  "docs": "https://docs.openclaw.ai/tools/code-execution"
}
```

## مرتبط

<CardGroup cols={2}>
  <Card title="ابزار Exec" href="/fa/tools/exec" icon="terminal">
    اجرای پوستهٔ محلی روی دستگاه یا Node جفت‌شدهٔ شما.
  </Card>
  <Card title="تأییدهای Exec" href="/fa/tools/exec-approvals" icon="shield">
    خط‌مشی اجازه/رد برای اجرای پوسته.
  </Card>
  <Card title="ابزارهای وب" href="/fa/tools/web" icon="globe">
    `web_search`، `x_search` و `web_fetch`.
  </Card>
  <Card title="ارائه‌دهندهٔ xAI" href="/fa/providers/xai" icon="microchip">
    مدل‌های Grok، جست‌وجوی وب/X و پیکربندی اجرای کد.
  </Card>
</CardGroup>
