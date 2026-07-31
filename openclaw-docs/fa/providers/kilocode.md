---
read_when:
    - برای چندین LLM یک کلید API واحد می‌خواهید
    - می‌خواهید مدل‌ها را از طریق Kilo Gateway در OpenClaw اجرا کنید
summary: از API یکپارچه Kilo Gateway برای دسترسی به مدل‌های متعدد در OpenClaw استفاده کنید
title: Gateway کیلو
x-i18n:
    generated_at: "2026-07-27T17:02:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0246a1a77f4265168b213e0167360e1cd89dc2ca864997f08cae5331037f9e89
    source_path: providers/kilocode.md
    workflow: 16
---

Kilo Gateway درخواست‌ها را از طریق یک نقطه پایانی سازگار با OpenAI و یک کلید API به مدل‌های متعددی هدایت می‌کند.

| ویژگی | مقدار                              |
| -------- | ---------------------------------- |
| ارائه‌دهنده | `kilocode`                         |
| احراز هویت     | `KILOCODE_API_KEY`                 |
| API      | سازگار با OpenAI                  |
| نشانی پایه | `https://api.kilo.ai/api/gateway/` |

## نصب Plugin

```bash
openclaw plugins install @openclaw/kilocode-provider
openclaw gateway restart
```

## راه‌اندازی

<Steps>
  <Step title="ایجاد حساب">
    به [app.kilo.ai](https://app.kilo.ai) بروید، وارد شوید یا حسابی ایجاد کنید و سپس یک کلید API بسازید.
  </Step>
  <Step title="اجرای فرایند آغاز به کار">
    ```bash
    openclaw onboard --auth-choice kilocode-api-key
    ```

    یا متغیر محیطی را مستقیماً تنظیم کنید:

    ```bash
    export KILOCODE_API_KEY="<your-kilocode-api-key>" # pragma: allowlist secret
    ```

  </Step>
  <Step title="بررسی دردسترس‌بودن مدل">
    ```bash
    openclaw models list --provider kilocode
    ```
  </Step>
</Steps>

## مدل پیش‌فرض و کاتالوگ

مدل پیش‌فرض `kilocode/kilo-auto/balanced`، رده مسیریابی هوشمند و متعادل Kilo Gateway است.
OpenClaw نگاشت وظیفه به مدل بالادستی را برای آن منتشر نمی‌کند؛ مسیریابی پشت
`kilo-auto/balanced` در اختیار Kilo Gateway است.

هنگام راه‌اندازی، OpenClaw از `GET https://api.kilo.ai/api/gateway/models` پرس‌وجو می‌کند و مدل‌های کشف‌شده را
پیش از کاتالوگ جایگزین ایستا ادغام می‌کند. کاتالوگ جایگزین ایستا فقط شامل
`kilocode/kilo-auto/balanced` (`Auto Balanced`، `input: ["text", "image"]`، `reasoning: true`،
`contextWindow: 1000000`، `maxTokens: 65536`) است.

هر مدل موجود در Gateway با `kilocode/<upstream-id>` قابل دسترسی است (برای مثال
`kilocode/anthropic/claude-sonnet-4`، `kilocode/openai/gpt-5.5`). برای مشاهده فهرست کامل کشف‌شده، `/models kilocode` یا
`openclaw models list --provider kilocode` را اجرا کنید.

## نمونه پیکربندی

```json5
{
  env: { KILOCODE_API_KEY: "<your-kilocode-api-key>" }, // pragma: allowlist secret
  agents: {
    defaults: {
      model: { primary: "kilocode/kilo-auto/balanced" },
    },
  },
}
```

## نکات رفتاری

<AccordionGroup>
  <Accordion title="انتقال و سازگاری">
    Kilo Gateway با OpenRouter سازگار است، بنابراین به‌جای شکل‌دهی بومی درخواست OpenAI، از مسیر درخواست
    پروکسی‌مانند و سازگار با OpenAI استفاده می‌کند (بدون `store` و بدون محموله تلاش استدلال OpenAI).

    - ارجاع‌های Kilo مبتنی بر Gemini در مسیر پروکسی Gemini باقی می‌مانند: OpenClaw امضاهای تفکر Gemini را
      در آنجا پاک‌سازی می‌کند، اما اعتبارسنجی بازپخش بومی Gemini یا بازنویسی‌های راه‌اندازی اولیه را فعال نمی‌کند.
    - درخواست‌ها از یک توکن Bearer ساخته‌شده از کلید API شما استفاده می‌کنند.

  </Accordion>

  <Accordion title="پوشش جریان و استدلال">
    پوشش جریان Kilo یک سرآیند درخواست `X-KILOCODE-FEATURE` اضافه می‌کند (پیش‌فرض `openclaw`،
    قابل بازنویسی با متغیر محیطی `KILOCODE_FEATURE`) و محموله‌های تلاش استدلال را برای
    مدل‌هایی که از آن پشتیبانی می‌کنند، عادی‌سازی می‌کند.

    <Warning>
    ارجاع‌های `kilocode/kilo-auto/balanced` و `x-ai/*` تزریق تلاش استدلال را نادیده می‌گیرند. اگر به پشتیبانی
    از استدلال نیاز دارید، از یک ارجاع مدل مشخص مانند `kilocode/anthropic/claude-sonnet-4` استفاده کنید.
    </Warning>

  </Accordion>

  <Accordion title="عیب‌یابی">
    - اگر کشف مدل هنگام راه‌اندازی ناموفق باشد، OpenClaw به کاتالوگ ایستای شامل `kilocode/kilo-auto/balanced` بازمی‌گردد.
    - معتبر بودن کلید API و فعال بودن مدل‌های موردنظر در حساب Kilo خود را تأیید کنید.
    - هنگامی که Gateway به‌صورت سرویس پس‌زمینه اجرا می‌شود، مطمئن شوید `KILOCODE_API_KEY` برای آن فرایند در دسترس است (برای مثال در `~/.openclaw/.env` یا از طریق `env.shellEnv`).

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/configuration-reference" icon="gear">
    مرجع کامل پیکربندی OpenClaw.
  </Card>
  <Card title="Kilo Gateway" href="https://app.kilo.ai" icon="arrow-up-right-from-square">
    داشبورد Kilo Gateway، کلیدهای API و مدیریت حساب.
  </Card>
</CardGroup>
