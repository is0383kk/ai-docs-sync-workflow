---
read_when:
    - می‌خواهید OpenClaw را از طریق Twilio به SMS متصل کنید
    - به راه‌اندازی Webhook پیامک یا فهرست مجاز نیاز دارید
summary: راه‌اندازی کانال پیامک Twilio، کنترل‌های دسترسی و پیکربندی Webhook
title: پیامک
x-i18n:
    generated_at: "2026-07-27T14:54:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 99a76b2f2d66858f8eb699939084104e620af9bc024053bbe1c1d7350530bff0
    source_path: channels/sms.md
    workflow: 16
---

OpenClaw از طریق یک شماره تلفن یا Messaging Service متعلق به Twilio پیامک دریافت و ارسال می‌کند. Gateway یک مسیر Webhook ورودی (به‌طور پیش‌فرض `/webhooks/sms`) ثبت می‌کند، به‌طور پیش‌فرض امضاهای درخواست Twilio را اعتبارسنجی می‌کند و پاسخ‌ها را از طریق Messages API متعلق به Twilio بازمی‌فرستد.

وضعیت: Plugin رسمی، با نصب جداگانه. فقط متن: بدون MMS/رسانه و فقط پیام‌های مستقیم.

<CardGroup cols={3}>
  <Card title="جفت‌سازی" icon="link" href="/fa/channels/pairing">
    خط‌مشی پیش‌فرض پیام مستقیم برای پیامک، جفت‌سازی است.
  </Card>
  <Card title="امنیت Gateway" icon="shield" href="/fa/gateway/security">
    نحوه در معرض دسترس قرار گرفتن Webhook و کنترل‌های دسترسی فرستنده را بررسی کنید.
  </Card>
  <Card title="عیب‌یابی کانال" icon="wrench" href="/fa/channels/troubleshooting">
    راهکارهای تشخیص و تعمیر مشترک میان کانال‌ها.
  </Card>
</CardGroup>

## پیش از شروع

موارد زیر لازم است:

- Plugin رسمی پیامک که با `openclaw plugins install @openclaw/sms` نصب شده باشد.
- یک حساب Twilio با شماره تلفنی دارای قابلیت پیامک، یا یک Twilio Messaging Service.
- Account SID و Auth Token متعلق به Twilio.
- یک نشانی HTTPS عمومی که به OpenClaw Gateway شما دسترسی داشته باشد.
- انتخاب خط‌مشی فرستنده: `pairing` (پیش‌فرض) برای استفاده خصوصی، `allowlist` برای شماره‌تلفن‌های ازپیش‌تأییدشده، یا `open` فقط برای دسترسی عمومی و عامدانه به پیامک.

یک شماره Twilio می‌تواند هم‌زمان برای پیامک و [تماس صوتی](/fa/plugins/voice-call) استفاده شود، مشروط بر اینکه هر دو قابلیت را داشته باشد. Webhook پیامک و Webhook صوتی به‌طور جداگانه در Twilio پیکربندی می‌شوند و از مسیرهای جداگانه Gateway استفاده می‌کنند؛ این صفحه فقط Webhook پیامک را پوشش می‌دهد.

## راه‌اندازی سریع

<Steps>
  <Step title="نصب Plugin">
    ```bash
    openclaw plugins install @openclaw/sms
    ```
  </Step>
  <Step title="ایجاد یا انتخاب فرستنده Twilio">
    در Twilio، بخش **Phone Numbers > Manage > Active numbers** را باز کنید و شماره‌ای دارای قابلیت پیامک انتخاب کنید. موارد زیر را ذخیره کنید:

    - Account SID، برای نمونه `ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
    - Auth Token
    - شماره تلفن فرستنده، برای نمونه `+15551234567`

    اگر به‌جای شماره فرستنده ثابت از Messaging Service استفاده می‌کنید، Messaging Service SID را ذخیره کنید؛ برای نمونه `MGxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`.

  </Step>

  <Step title="پیکربندی کانال پیامک">

این محتوا را با نام `sms.patch.json5` ذخیره کنید و جای‌نگهدارها را تغییر دهید:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

آن را اعمال کنید:

```bash
openclaw config patch --file ./sms.patch.json5 --dry-run
openclaw config patch --file ./sms.patch.json5
```

  </Step>

  <Step title="هدایت Twilio به Webhook مربوط به Gateway">
    در تنظیمات شماره تلفن Twilio، بخش **Messaging** را باز کنید و **A message comes in** را روی مقدار زیر تنظیم کنید:

```text
https://gateway.example.com/webhooks/sms
```

    از HTTP `POST` استفاده کنید. مسیر محلی پیش‌فرض `/webhooks/sms` است؛ اگر مسیر دیگری نیاز دارید، `channels.sms.webhookPath` را تغییر دهید.

  </Step>

  <Step title="در معرض دسترس قرار دادن مسیر دقیق Webhook پیامک">
    نشانی عمومی شما باید مسیر پیامک را به فرایند Gateway (درگاه پیش‌فرض `18789`) هدایت کند. اگر برای آزمایش محلی از Tailscale Funnel استفاده می‌کنید، `/webhooks/sms` را صریحاً در معرض دسترس قرار دهید:

```bash
tailscale funnel --bg --set-path /webhooks/sms http://127.0.0.1:<gateway-port>/webhooks/sms
tailscale funnel status
```

    تماس صوتی و پیامک از مسیرهای Webhook جداگانه استفاده می‌کنند. اگر یک شماره Twilio هر دو را مدیریت می‌کند، هر دو مسیر را در Twilio و تونل خود پیکربندی‌شده نگه دارید.

  </Step>

  <Step title="راه‌اندازی Gateway و تأیید نخستین فرستنده">

```bash
openclaw gateway
```

یک پیام متنی به شماره Twilio ارسال کنید. نخستین پیام یک درخواست جفت‌سازی ایجاد می‌کند. آن را تأیید کنید:

```bash
openclaw pairing list sms
openclaw pairing approve sms <CODE>
```

    کدهای جفت‌سازی پس از 1 ساعت منقضی می‌شوند.

  </Step>
</Steps>

## نمونه‌های پیکربندی

همه کلیدها زیر `channels.sms` قرار دارند (و برای هر حساب زیر `channels.sms.accounts.<id>`):

| کلید                                     | پیش‌فرض         | کاربرد                                                             |
| --------------------------------------- | --------------- | ------------------------------------------------------------------- |
| `enabled`                               | `true`          | فعال یا غیرفعال کردن کانال/حساب.                              |
| `accountSid`                            | —               | Twilio Account SID ‏(`AC...`).                                       |
| `authToken`                             | —               | Twilio Auth Token؛ رشته متن ساده یا SecretRef.                   |
| `fromNumber`                            | —               | شماره فرستنده با قالب E.164.                                                |
| `messagingServiceSid`                   | —               | Messaging Service SID ‏(`MG...`) که وقتی هیچ `fromNumber`‌ای حل نشود استفاده می‌شود. |
| `defaultTo`                             | —               | مقصد پیش‌فرض وقتی جریان ارسال، هدف صریحی را مشخص نمی‌کند.      |
| `webhookPath`                           | `/webhooks/sms` | مسیر HTTP در Gateway برای Webhookهای ورودی Twilio.                      |
| `publicWebhookUrl`                      | —               | نشانی عمومی پیکربندی‌شده در Twilio؛ برای اعتبارسنجی امضا الزامی است. |
| `dangerouslyDisableSignatureValidation` | `false`         | نادیده گرفتن بررسی‌های `X-Twilio-Signature`؛ فقط برای آزمایش تونل محلی.        |
| `dmPolicy`                              | `"pairing"`     | `pairing`، `allowlist`، `open` یا `disabled`.                      |
| `allowFrom`                             | `[]`            | شماره‌های مجاز فرستنده با قالب E.164، یا `"*"` همراه با `dmPolicy: "open"`.  |
| `textChunkLimit`                        | `1500`          | حداکثر تعداد نویسه در هر بخش پیامک خروجی.                          |
| `accounts`، `defaultAccount`            | —               | نگاشت چندحسابی و شناسه حساب پیش‌فرض.                           |

### فایل پیکربندی

وقتی می‌خواهید تعریف کانال همراه با پیکربندی Gateway منتقل شود، از راه‌اندازی مبتنی بر فایل پیکربندی استفاده کنید:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

### متغیرهای محیطی

متغیرهای محیطی فقط روی حساب پیش‌فرض اعمال می‌شوند؛ مقادیر پیکربندی بر مقادیر محیطی اولویت دارند.

| متغیر                                        | نگاشت به                                            |
| ----------------------------------------------- | -------------------------------------------------- |
| `TWILIO_ACCOUNT_SID`                            | `accountSid`                                       |
| `TWILIO_AUTH_TOKEN`                             | `authToken`                                        |
| `TWILIO_PHONE_NUMBER` (نام مستعار `TWILIO_SMS_FROM`) | `fromNumber`                                       |
| `TWILIO_MESSAGING_SERVICE_SID`                  | `messagingServiceSid`                              |
| `SMS_PUBLIC_WEBHOOK_URL`                        | `publicWebhookUrl`                                 |
| `SMS_WEBHOOK_PATH`                              | `webhookPath`                                      |
| `SMS_ALLOWED_USERS`                             | `allowFrom` (جداشده با ویرگول)                      |
| `SMS_TEXT_CHUNK_LIMIT`                          | `textChunkLimit`                                   |
| `SMS_DANGEROUSLY_DISABLE_SIGNATURE_VALIDATION`  | `dangerouslyDisableSignatureValidation` ‏(`"true"`) |

```bash
export TWILIO_ACCOUNT_SID="ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
export TWILIO_AUTH_TOKEN="<twilio-auth-token>"
export TWILIO_PHONE_NUMBER="+15551234567"
export SMS_PUBLIC_WEBHOOK_URL="https://gateway.example.com/webhooks/sms"
```

سپس کانال را در پیکربندی فعال کنید:

```json5
{
  channels: {
    sms: {
      enabled: true,
      dmPolicy: "pairing",
    },
  },
}
```

### توکن احراز هویت SecretRef

`authToken` می‌تواند یک SecretRef ‏(`source: "env" | "file" | "exec"`) باشد. زمانی از این روش استفاده کنید که Gateway باید به‌جای ذخیره پیکربندی به‌صورت متن ساده، Twilio Auth Token را از زمان‌اجرای اسرار OpenClaw دریافت کند:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: { source: "env", provider: "default", id: "TWILIO_AUTH_TOKEN" },
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

متغیر محیطی یا ارائه‌دهنده اسرار ارجاع‌شده باید برای زمان‌اجرای Gateway قابل مشاهده باشد. پس از تغییر متغیرهای محیطی میزبان، فرایندهای مدیریت‌شده Gateway را دوباره راه‌اندازی کنید.

### فرستنده Messaging Service

وقتی Twilio باید فرستنده را از طریق یک Messaging Service انتخاب کند، به‌جای `fromNumber` از `messagingServiceSid` استفاده کنید:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      messagingServiceSid: "MGxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "pairing",
    },
  },
}
```

اگر پس از حل پیکربندی و متغیرهای محیطی، هر دو `fromNumber` و `messagingServiceSid` وجود داشته باشند، `fromNumber` استفاده می‌شود.

### هدف خروجی پیش‌فرض

وقتی در صورت مشخص نشدن هدف صریح در جریان ارسال، خودکارسازی یا تحویل آغازشده توسط عامل باید مقصدی پیش‌فرض داشته باشد، `defaultTo` را تنظیم کنید:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      defaultTo: "+15557654321",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
    },
  },
}
```

## کنترل دسترسی

`channels.sms.dmPolicy` دسترسی مستقیم به پیامک را کنترل می‌کند:

- `pairing` (پیش‌فرض): فرستندگان ناشناس یک کد جفت‌سازی دریافت می‌کنند؛ با `openclaw pairing approve sms <CODE>` تأیید کنید.
- `allowlist`: فقط فرستندگان موجود در `allowFrom` پردازش می‌شوند. `allowFrom` خالی همه فرستندگان را رد می‌کند (Gateway هنگام راه‌اندازی یک هشدار ثبت می‌کند).
- `open`: اعتبارسنجی پیکربندی الزام می‌کند که `allowFrom` شامل `"*"` باشد. بدون نویسه عام، فقط شماره‌های فهرست‌شده می‌توانند گفتگو کنند.
- `disabled`: همه پیام‌های مستقیم ورودی کنار گذاشته می‌شوند.

ورودی‌های `allowFrom` باید شماره‌تلفن‌هایی با قالب E.164 مانند `+15551234567` باشند. پیشوندهای `sms:` و `twilio-sms:` پذیرفته و عادی‌سازی می‌شوند. برای یک دستیار خصوصی، `dmPolicy: "allowlist"` را با شماره‌تلفن‌های صریح ترجیح دهید:

```json5
{
  channels: {
    sms: {
      enabled: true,
      accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      authToken: "twilio-auth-token",
      fromNumber: "+15551234567",
      publicWebhookUrl: "https://gateway.example.com/webhooks/sms",
      dmPolicy: "allowlist",
      allowFrom: ["+15557654321"],
    },
  },
}
```

## ارسال پیامک

وقتی کانال پیامک انتخاب شده باشد، هدف‌ها شماره‌های E.164 بدون پیشوند یا دارای پیشوند `sms:` را می‌پذیرند:

```bash
openclaw message send --channel sms --target sms:+15551234567 --message "hello"
```

وقتی انتخاب کانال ضمنی است، پیشوند `twilio-sms:` این کانال را انتخاب می‌کند، بدون اینکه پیشوند سرویس `sms:` را تصاحب کند؛ iMessage از آن پیشوند برای انتخاب تحویل پیامک اپراتور برای هدف‌های خود استفاده می‌کند:

```bash
openclaw message send --target twilio-sms:+15551234567 --message "hello"
```

CLI به یک `--target` صریح نیاز دارد. `defaultTo` برای مسیرهای خودکارسازی و تحویل آغازشده توسط عامل است که در آن‌ها می‌توان هدف را از پیکربندی کانال حل کرد.

پاسخ‌های عامل به مکالمات SMS ورودی، به‌طور خودکار از طریق فرستنده Twilio پیکربندی‌شده برای فرستنده بازگردانده می‌شوند.

خروجی SMS متن ساده است. OpenClaw نشانه‌گذاری Markdown را حذف می‌کند، بلوک‌های کد حصاردار را مسطح می‌کند، پیوندها را به‌صورت `label (url)` بازنویسی می‌کند و پیش از ارسال پاسخ‌های طولانی از طریق Twilio، آن‌ها را به قطعه‌هایی با حداکثر `textChunkLimit` نویسه (پیش‌فرض 1500) تقسیم می‌کند.

## تأیید راه‌اندازی

پس از راه‌اندازی Gateway:

1. تأیید کنید که گزارش Gateway مسیر Webhook مربوط به SMS را نشان می‌دهد.
2. یک بررسی آزمایشی در سمت Twilio اجرا کنید (URL/روش Webhook پیکربندی‌شده Twilio و خطاهای ورودی اخیر را بررسی می‌کند):

```bash
openclaw channels capabilities --channel sms
openclaw channels status --channel sms --probe --json
```

3. از تلفن خود یک SMS به شماره Twilio ارسال کنید.
4. `openclaw pairing list sms` را اجرا کنید.
5. کد جفت‌سازی را با `openclaw pairing approve sms <CODE>` تأیید کنید.
6. یک SMS دیگر ارسال کنید و تأیید کنید که عامل پاسخ می‌دهد.

برای آزمایش صرفاً خروجی، از دستور زیر استفاده کنید:

```bash
openclaw message send --channel sms --target sms:+15557654321 --message "OpenClaw SMS test"
```

### آزمایش سرتاسری از macOS iMessage/SMS

در یک Mac که می‌تواند از طریق Messages پیامک اپراتوری ارسال کند، می‌توانید با استفاده از `imsg` سمت فرستنده را بدون دست‌زدن به تلفن خود راه‌اندازی کنید:

```bash
imsg send --to "+15551234567" --service sms --text "OpenClaw SMS E2E $(date -u +%Y%m%dT%H%M%SZ)" --json
openclaw pairing list sms
openclaw pairing approve sms <CODE>
imsg send --to "+15551234567" --service sms --text "reply exactly SMS pong" --json
```

پیام نخست باید یک درخواست جفت‌سازی ایجاد کند. پیام دوم باید پاسخ عامل را از طریق Twilio دریافت کند.

## امنیت Webhook

OpenClaw به‌طور پیش‌فرض `X-Twilio-Signature` را با استفاده از `publicWebhookUrl` و `authToken` اعتبارسنجی می‌کند. بخش نقطه پایانی `publicWebhookUrl` را، شامل طرح، میزبان، مسیر و رشته پرس‌وجو، به‌صورت بایت‌به‌بایت با URL پیکربندی‌شده در Twilio یکسان نگه دارید. همان‌طور که Twilio الزام می‌کند، OpenClaw قطعه‌های [لغو تنظیمات اتصال](https://www.twilio.com/docs/usage/webhooks/webhooks-connection-overrides) (`#...`) را از محاسبه امضا کنار می‌گذارد.

مسیر Webhook همچنین، مستقل از اعتبارسنجی امضا، موارد زیر را اعمال می‌کند:

- فقط `POST`.
- سهمیه درخواست ناموفق برابر با 300 درخواست در دقیقه به‌ازای هر حساب SMS، مسیر Webhook و نشانی کلاینت تعیین‌شده است. همه درخواست‌ها در این سهمیه محاسبه می‌شوند، اما HTTP 429 فقط پس از شکست درخواست در تجزیه بدنه، اعتبارسنجی Twilio یا تطبیق AccountSid اعمال می‌شود.
- پس از موفقیت آن بررسی‌ها، محدودیت نرخ فراخوان‌های قابل‌ارسال برابر با 30 فراخوان پذیرفته‌شده در دقیقه به‌ازای هر حساب SMS، مسیر Webhook و نشانی کلاینت تعیین‌شده است (بالاتر از آن HTTP 429). اگر اعتبارسنجی امضا غیرفعال باشد، این محدودیت 30 مورد در دقیقه سقف ارسال بدون احراز هویت است.
- نشانی‌های کلاینت از طریق قواعد مشترک پراکسی مورداعتماد Gateway تعیین می‌شوند. اگر `gateway.trustedProxies` شامل پراکسی معکوسی باشد که فراخوان‌های Twilio را هدایت می‌کند، OpenClaw کلید این محدودیت‌ها را از نشانی کلاینت هدایت‌شده می‌سازد؛ در غیر این صورت، از نشانی مستقیم سوکت استفاده می‌کند.
- `AccountSid` در بار داده باید با `accountSid` پیکربندی‌شده مطابقت داشته باشد (در غیر این صورت HTTP 403).
- مقادیر بازپخش‌شده `MessageSid` به‌مدت 10 دقیقه تکرارزدایی می‌شوند.
- حافظه نهان بازپخش هر حساب SMS حداکثر 10,000 شناسه پیام زنده را نگه می‌دارد. وقتی همه جایگاه‌ها فعال باشند، Webhookهای جدید آن حساب تا زمان انقضای قدیمی‌ترین جایگاه، با HTTP 429 و سرآیند `Retry-After` به‌صورت بسته ایمن شکست می‌خورند.
- بدنه درخواست‌های بزرگ‌تر از 32 KB رد می‌شود.

Twilio به‌طور پیش‌فرض HTTP 429 را دوباره امتحان نمی‌کند و پشتیبانی از `Retry-After` را مستند نکرده است. لغو تنظیمات اتصال `#rp=4xx` و `#rp=all` تلاش مجدد برای خطاهای 4xx را فعال می‌کنند، اما Twilio کل تراکنش تلاش مجدد را به 15 ثانیه محدود می‌کند؛ بنابراین تلاش‌های مجدد ممکن است همچنان پیش از انقضای جایگاه حافظه نهان بازپخش پایان یابند. هنگامی که کنترل‌کننده دیگری باید تحویل‌های ناموفق را دریافت کند، یک URL جایگزین پیکربندی کنید؛ پاسخ 429 را رد بسته ایمن در نظر بگیرید، نه پس‌فشار قابل‌اعتماد.

فقط برای آزمایش تونل محلی می‌توانید تنظیم زیر را اعمال کنید:

```json5
{
  channels: {
    sms: {
      dangerouslyDisableSignatureValidation: true,
    },
  },
}
```

در یک Gateway عمومی، اعتبارسنجی امضای غیرفعال‌شده را به‌کار نبرید.

## پیکربندی چندحسابی

هنگامی که بیش از یک شماره Twilio را مدیریت می‌کنید، از `accounts` استفاده کنید:

```json5
{
  channels: {
    sms: {
      accounts: {
        support: {
          enabled: true,
          accountSid: "ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
          authToken: "twilio-auth-token",
          fromNumber: "+15551234567",
          publicWebhookUrl: "https://gateway.example.com/webhooks/sms/support",
          webhookPath: "/webhooks/sms/support",
          dmPolicy: "allowlist",
          allowFrom: ["+15557654321"],
        },
      },
    },
  },
}
```

هر حساب باید از یک `webhookPath` متمایز استفاده کند؛ Gateway از ثبت مسیر Webhook که مسیر آن از قبل در اختیار حساب دیگری است خودداری می‌کند. مقادیر جایگزین محیطی `TWILIO_*`/`SMS_*` فقط برای حساب پیش‌فرض اعمال می‌شوند؛ برای تغییر حساب پیش‌فرض، `defaultAccount` را تنظیم کنید.

## عیب‌یابی

### Twilio پاسخ 403 می‌دهد یا OpenClaw وب‌هوک را رد می‌کند

بررسی کنید که `publicWebhookUrl` دقیقاً با URL پیکربندی‌شده در Twilio، شامل طرح، میزبان، مسیر و رشته پرس‌وجو، مطابقت داشته باشد. Twilio رشته URL عمومی را امضا می‌کند؛ بنابراین بازنویسی‌های پراکسی و نام‌های میزبان جایگزین می‌توانند اعتبارسنجی امضا را مختل کنند.

پاسخ 403 همراه با `Invalid account` به این معناست که `AccountSid` در بار داده ورودی با `accountSid` پیکربندی‌شده مطابقت ندارد؛ بررسی کنید که Webhook به حساب مالک شماره اشاره می‌کند.

### هیچ درخواست جفت‌سازی ظاهر نمی‌شود

URL و روش Webhook بخش **Messaging** شماره Twilio را بررسی کنید. این مورد باید به URL وب‌هوک SMS اشاره کند و از `POST` استفاده کند. همچنین تأیید کنید که Gateway از اینترنت عمومی یا از طریق تونل شما قابل‌دسترسی است.

اگر گزارش پیام Twilio خطای `11200` را نشان می‌دهد، Twilio پیامک ورودی را پذیرفته است اما نتوانسته به Webhook شما دسترسی پیدا کند. موارد زیر را بررسی کنید:

- گزینه **Messaging > A message comes in** در Twilio به `publicWebhookUrl` اشاره می‌کند.
- روش، `POST` است.
- تونل یا پراکسی معکوس، `webhookPath` دقیق را در معرض دسترسی قرار می‌دهد؛ برای Tailscale Funnel، دستور `tailscale funnel status` را اجرا و تأیید کنید که `/webhooks/sms` فهرست شده است.
- `publicWebhookUrl` از همان طرح، میزبان، مسیر و رشته پرس‌وجویی استفاده می‌کند که Twilio ارسال می‌کند تا اعتبارسنجی امضا بتواند URL امضاشده را بازسازی کند.

`openclaw channels status --channel sms --probe` هم تنظیمات ناهماهنگ Webhook در Twilio و هم خطاهای اخیر `11200` را نمایش می‌دهد.

### ارسال‌های خروجی ناموفق هستند

تأیید کنید که `accountSid`، `authToken` و یکی از `fromNumber` یا `messagingServiceSid` تعیین شده‌اند. اگر از حساب آزمایشی Twilio استفاده می‌کنید، ممکن است پیش از ارسال SMS خروجی لازم باشد شماره مقصد در Twilio تأیید شود.

### پیام‌ها می‌رسند اما عامل پاسخ نمی‌دهد

`dmPolicy` و `allowFrom` را بررسی کنید. با خط‌مشی پیش‌فرض `pairing`، فرستنده باید پیش از پردازش نوبت‌های عادی عامل تأیید شده باشد.
