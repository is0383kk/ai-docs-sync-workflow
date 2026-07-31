---
read_when:
    - می‌خواهید از Chutes با OpenClaw استفاده کنید
    - به مسیر راه‌اندازی OAuth یا کلید API نیاز دارید
    - مدل پیش‌فرض، نام‌های مستعار یا رفتار کشف را می‌خواهید
summary: راه‌اندازی Chutes (OAuth یا کلید API، کشف مدل، نام‌های مستعار)
title: Chutes
x-i18n:
    generated_at: "2026-07-27T15:39:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 57ea5112105f19028c1a348b4d7fec4cf7ef12de00b1b2de9c152057bf5033a9
    source_path: providers/chutes.md
    workflow: 16
---

[Chutes](https://chutes.ai) کاتالوگ‌های مدل متن‌باز را از طریق یک API سازگار با OpenAI ارائه می‌کند. OpenClaw هم از OAuth مرورگر و هم از احراز هویت با کلید API پشتیبانی می‌کند.

| ویژگی            | مقدار                                                   |
| ---------------- | ------------------------------------------------------- |
| ارائه‌دهنده      | `chutes`                                                |
| Plugin           | بسته خارجی رسمی (`@openclaw/chutes-provider`) |
| API              | سازگار با OpenAI                                       |
| نشانی پایه       | `https://llm.chutes.ai/v1`                              |
| احراز هویت       | OAuth یا کلید API (پایین را ببینید)                            |
| متغیرهای محیطی زمان اجرا | `CHUTES_API_KEY`، `CHUTES_OAUTH_TOKEN`                  |

`CHUTES_OAUTH_TOKEN` یک توکن دسترسی OAuth را که از قبل دریافت شده است، مستقیماً ارائه می‌کند
(برای مثال در CI) و جریان تعاملی مرورگر در ادامه را دور می‌زند.

## نصب Plugin

```bash
openclaw plugins install @openclaw/chutes-provider
openclaw gateway restart
```

## شروع به کار

هر دو مسیر، مدل پیش‌فرض را روی `chutes/zai-org/GLM-5-TEE` تنظیم و
کاتالوگ Chutes را ثبت می‌کنند.

<Tabs>
  <Tab title="OAuth">
    <Steps>
      <Step title="اجرای جریان راه‌اندازی OAuth">
        ```bash
        openclaw onboard --auth-choice chutes
        ```
        OpenClaw جریان مرورگر را به‌صورت محلی اجرا می‌کند یا در میزبان‌های راه‌دور/بدون رابط گرافیکی،
        یک نشانی URL و جریان چسباندن تغییرمسیر را نمایش می‌دهد. توکن‌های OAuth از طریق پروفایل‌های
        احراز هویت OpenClaw به‌طور خودکار تازه‌سازی می‌شوند.
      </Step>
    </Steps>
  </Tab>
  <Tab title="کلید API">
    <Steps>
      <Step title="دریافت کلید API">
        یک کلید در
        [chutes.ai/settings/api-keys](https://chutes.ai/settings/api-keys) ایجاد کنید.
      </Step>
      <Step title="اجرای جریان راه‌اندازی کلید API">
        ```bash
        openclaw onboard --auth-choice chutes-api-key
        ```
      </Step>
    </Steps>
  </Tab>
</Tabs>

## رفتار کشف

وقتی احراز هویت Chutes در دسترس باشد، OpenClaw با آن اعتبارنامه از `GET /v1/models`
پرس‌وجو می‌کند و از مدل‌های کشف‌شده استفاده می‌کند که به‌ازای هر
اعتبارنامه به‌مدت 5 دقیقه در حافظه نهان نگه‌داری می‌شوند. در صورت منقضی یا غیرمجاز بودن کلید (HTTP 401)، OpenClaw یک‌بار دیگر
بدون اعتبارنامه تلاش می‌کند. اگر کشف همچنان هیچ ردیفی برنگرداند، ناموفق شود یا هر
وضعیت غیر 2xx دیگری برگرداند، سامانه به کاتالوگ ایستای همراه بازمی‌گردد (کشف با کلید API
و OAuth هر دو از همین مسیر استفاده می‌کنند). اگر کشف هنگام راه‌اندازی ناموفق باشد،
کاتالوگ ایستا به‌طور خودکار استفاده می‌شود.

## نام‌های مستعار پیش‌فرض

OpenClaw دو نام مستعار کاربردی برای کاتالوگ Chutes ثبت می‌کند:

| نام مستعار           | مدل مقصد                           |
| --------------- | -------------------------------------- |
| `chutes-pro`    | `chutes/deepseek-ai/DeepSeek-V3.2-TEE` |
| `chutes-vision` | `chutes/moonshotai/Kimi-K2.5-TEE`      |

## کاتالوگ آغازین داخلی

کاتالوگ جایگزین همراه شامل این پنج مدل است که در حال حاضر ارائه می‌شوند:

| ارجاع مدل                              |
| -------------------------------------- |
| `chutes/zai-org/GLM-5-TEE`             |
| `chutes/deepseek-ai/DeepSeek-V3.2-TEE` |
| `chutes/moonshotai/Kimi-K2.5-TEE`      |
| `chutes/MiniMaxAI/MiniMax-M2.5-TEE`    |
| `chutes/Qwen/Qwen3.5-397B-A17B-TEE`    |

برای فهرست کامل، `openclaw models list --all --provider chutes` را اجرا کنید.

## نمونه پیکربندی

```json5
{
  agents: {
    defaults: {
      model: { primary: "chutes/zai-org/GLM-5-TEE" },
      models: {
        "chutes/zai-org/GLM-5-TEE": { alias: "Chutes GLM 5" },
        "chutes/deepseek-ai/DeepSeek-V3.2-TEE": { alias: "Chutes DeepSeek V3.2" },
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="جایگزین‌های OAuth">
    جریان OAuth را با متغیرهای محیطی اختیاری سفارشی کنید:

    | متغیر | کاربرد |
    | -------- | ------- |
    | `CHUTES_CLIENT_ID` | شناسه کلاینت OAuth (اگر تنظیم نشده باشد، درخواست می‌شود) |
    | `CHUTES_CLIENT_SECRET` | رمز کلاینت OAuth |
    | `CHUTES_OAUTH_REDIRECT_URI` | URI تغییرمسیر (پیش‌فرض `http://127.0.0.1:1456/oauth-callback`) |
    | `CHUTES_OAUTH_SCOPES` | دامنه‌های جداشده با فاصله (پیش‌فرض `openid profile chutes:invoke`) |

    برای الزامات برنامه تغییرمسیر و دریافت راهنمایی، [مستندات OAuth مربوط به Chutes](https://chutes.ai/docs/sign-in-with-chutes/overview)
    را ببینید.

  </Accordion>

  <Accordion title="نکته‌ها">
    - مدل‌های Chutes به‌صورت `chutes/<model-id>` ثبت می‌شوند.
    - Chutes هنگام پخش جریانی، میزان استفاده از توکن را گزارش نمی‌کند (`supportsUsageInStreaming: false`)؛ پس از تکمیل جریان، مجموع استفاده همچنان نمایش داده می‌شود.

  </Accordion>
</AccordionGroup>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    قواعد ارائه‌دهنده، ارجاع‌های مدل و رفتار تغییر مسیر در صورت خرابی.
  </Card>
  <Card title="مرجع پیکربندی" href="/fa/gateway/configuration-reference" icon="gear">
    طرح‌واره کامل پیکربندی، شامل تنظیمات ارائه‌دهنده.
  </Card>
  <Card title="Chutes" href="https://chutes.ai" icon="arrow-up-right-from-square">
    داشبورد و مستندات API مربوط به Chutes.
  </Card>
  <Card title="کلیدهای API مربوط به Chutes" href="https://chutes.ai/settings/api-keys" icon="key">
    کلیدهای API مربوط به Chutes را ایجاد و مدیریت کنید.
  </Card>
</CardGroup>
