---
read_when:
    - راه‌اندازی Synology Chat با OpenClaw
    - اشکال‌زدایی مسیریابی Webhook در Synology Chat
summary: راه‌اندازی Webhook چت Synology و پیکربندی OpenClaw
title: گفت‌وگوی Synology
x-i18n:
    generated_at: "2026-07-27T14:57:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3c03379944ee4187260a7287f6d2aed1ad8fdd1c22b5581c8a5d55515bbb6ad5
    source_path: channels/synology-chat.md
    workflow: 16
---

Synology Chat از طریق یک جفت Webhook به OpenClaw متصل می‌شود: یک Webhook خروجی Synology Chat پیام‌های مستقیم ورودی را به Gateway ارسال می‌کند و پاسخ‌ها از طریق یک Webhook ورودی Synology Chat بازگردانده می‌شوند.

وضعیت: Plugin رسمی، با نصب جداگانه. فقط پیام‌های مستقیم؛ ارسال متن و فایل مبتنی بر URL پشتیبانی می‌شود.

## نصب

```bash
openclaw plugins install @openclaw/synology-chat
```

نسخهٔ محلی (هنگام اجرا از یک مخزن git):

```bash
openclaw plugins install ./path/to/local/synology-chat-plugin
```

جزئیات: [Pluginها](/fa/tools/plugin)

## راه‌اندازی سریع

1. Plugin را نصب کنید (در بالا).
2. در یکپارچه‌سازی‌های Synology Chat:
   - یک Webhook ورودی ایجاد و URL آن را کپی کنید.
   - یک Webhook خروجی با توکن محرمانهٔ خود ایجاد کنید.
3. URL وب‌هوک خروجی را به Gateway مربوط به OpenClaw هدایت کنید:
   - به‌طور پیش‌فرض `https://gateway-host/webhook/synology`.
   - یا `channels.synology-chat.webhookPath` سفارشی خودتان.
4. راه‌اندازی را در OpenClaw تکمیل کنید. Synology Chat در هر دو جریان در همان فهرست راه‌اندازی کانال نمایش داده می‌شود:
   - هدایت‌شده: `openclaw onboard` یا `openclaw channels add`
   - مستقیم: `openclaw channels add --channel synology-chat --token <token> --url <incoming-webhook-url>`
5. Gateway را دوباره راه‌اندازی کنید و یک پیام مستقیم برای ربات Synology Chat بفرستید.

جزئیات احراز هویت Webhook:

- OpenClaw توکن Webhook خروجی را ابتدا از `body.token`، سپس
  `?token=...` و پس از آن از سرآیندها می‌پذیرد.
- قالب‌های پذیرفته‌شدهٔ سرآیند:
  - `x-synology-token`
  - `x-webhook-token`
  - `x-openclaw-token`
  - `Authorization: Bearer <token>`
- توکن‌های خالی یا مفقود با رد ایمن مواجه می‌شوند.
- محموله‌ها می‌توانند `application/x-www-form-urlencoded` یا `application/json` باشند؛ `token`، `user_id` و `text` الزامی هستند.

## ماندگاری ورودی

پس از موفقیت بررسی‌های توکن، سیاست فرستنده و محدودیت نرخ، OpenClaw توکن Webhook را از پاکت ذخیره‌شده حذف می‌کند و پیش از تأیید دریافت، رویداد را به‌صورت پایدار در صف قرار می‌دهد. مسیر فقط پس از موفقیت این الحاق، `204` را برمی‌گرداند؛ خرابی ماندگارسازی `503` را برمی‌گرداند تا Synology Chat بتواند به‌جای ازدست‌دادن بی‌سروصدای پیام، دوباره تلاش کند.

رویدادهای در انتظار یا قابل تلاش مجدد پس از راه‌اندازی مجدد Gateway باقی می‌مانند. `post_id` پایدار Synology تا زمانی که رکورد تکمیل فعال یا نگه‌داری‌شدهٔ متناظر وجود داشته باشد، از ایجاد ورودی‌های تکراری در صف جلوگیری می‌کند. تحویل در انتقال از صف به عامل همچنان حداقل یک‌بار انجام می‌شود، بنابراین خرابی در این مرز ممکن است همچنان باعث بازپخش یک نوبت شود.

پیکربندی حداقلی:

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      token: "synology-outgoing-token",
      incomingUrl: "https://nas.example.com/webapi/entry.cgi?api=SYNO.Chat.External&method=incoming&version=2&token=...",
      webhookPath: "/webhook/synology",
      dmPolicy: "allowlist",
      allowedUserIds: ["123456"],
      rateLimitPerMinute: 30,
      allowInsecureSsl: false,
    },
  },
}
```

## متغیرهای محیطی

برای حساب پیش‌فرض، می‌توانید از متغیرهای محیطی استفاده کنید:

- `SYNOLOGY_CHAT_TOKEN`
- `SYNOLOGY_CHAT_INCOMING_URL`
- `SYNOLOGY_NAS_HOST`
- `SYNOLOGY_ALLOWED_USER_IDS` (جداشده با ویرگول)
- `SYNOLOGY_RATE_LIMIT`
- `OPENCLAW_BOT_NAME`

مقادیر پیکربندی بر متغیرهای محیطی اولویت دارند.

`SYNOLOGY_CHAT_INCOMING_URL` و `SYNOLOGY_NAS_HOST` را نمی‌توان از `.env` فضای کاری تنظیم کرد؛ [فایل‌های `.env` فضای کاری](/fa/gateway/security#workspace-env-files) را ببینید.

## سیاست پیام مستقیم و کنترل دسترسی

- مقادیر پشتیبانی‌شدهٔ `dmPolicy`: ‏`allowlist` (پیش‌فرض)، `open` و `disabled`. ‏Synology Chat جریان جفت‌سازی ندارد؛ فرستندگان را با افزودن شناسه‌های عددی کاربری Synology آن‌ها به `allowedUserIds` تأیید کنید.
- `allowedUserIds` فهرستی (یا رشته‌ای جداشده با ویرگول) از شناسه‌های کاربری Synology را می‌پذیرد.
- در حالت `allowlist`، فهرست خالی `allowedUserIds` به‌عنوان پیکربندی نادرست تلقی می‌شود و مسیر Webhook شروع نخواهد شد.
- `dmPolicy: "open"` تنها زمانی پیام‌های مستقیم عمومی را مجاز می‌کند که `allowedUserIds` شامل `"*"` باشد؛ با ورودی‌های محدودکننده، فقط کاربران منطبق می‌توانند گفت‌وگو کنند. `open` با فهرست خالی `allowedUserIds` نیز از شروع مسیر خودداری می‌کند.
- `dmPolicy: "disabled"` پیام‌های مستقیم را مسدود می‌کند.
- اتصال گیرندهٔ پاسخ به‌طور پیش‌فرض بر `user_id` عددی پایدار باقی می‌ماند. `channels.synology-chat.dangerouslyAllowNameMatching: true` حالت سازگاری اضطراری است که جست‌وجوی نام کاربری/نام مستعار تغییرپذیر را برای تحویل پاسخ دوباره فعال می‌کند.

## تحویل خروجی

از شناسه‌های عددی کاربران Synology Chat به‌عنوان مقصد استفاده کنید. پیشوندهای `synology-chat:`، `synology_chat:` و `synology:` پذیرفته می‌شوند.

نمونه‌ها:

```bash
openclaw message send --channel synology-chat --target 123456 --message "Hello from OpenClaw"
openclaw message send --channel synology-chat --target synology-chat:123456 --message "Hello again"
openclaw message send --channel synology-chat --target synology:123456 --message "Short prefix"
```

متن خروجی در قطعه‌های 2000 نویسه‌ای تقسیم می‌شود. ارسال رسانه از طریق تحویل فایل مبتنی بر URL پشتیبانی می‌شود: NAS فایل را بارگیری و پیوست می‌کند (حداکثر 32 MB). ‏URLهای فایل خروجی باید از `http` یا `https` استفاده کنند و مقصدهای شبکهٔ خصوصی یا مسدودشده به هر نحو دیگر، پیش از آنکه OpenClaw ‏URL را به Webhook ‏NAS ارسال کند، رد می‌شوند.

## چندحسابی

چندین حساب Synology Chat زیر `channels.synology-chat.accounts` پشتیبانی می‌شوند.
هر حساب می‌تواند توکن، URL ورودی، مسیر Webhook، سیاست پیام مستقیم و محدودیت‌ها را بازنویسی کند.
نشست‌های پیام مستقیم به‌ازای هر حساب و کاربر ایزوله می‌شوند، بنابراین `user_id` عددی یکسان
در دو حساب متفاوت Synology وضعیت رونوشت مشترکی ندارد.
برای هر حساب فعال یک `webhookPath` متمایز تعیین کنید. OpenClaw مسیرهای دقیق تکراری را رد می‌کند
و در راه‌اندازی‌های چندحسابی از شروع حساب‌های نام‌داری که فقط مسیر Webhook مشترکی را به ارث می‌برند خودداری می‌کند.
اگر عمداً به ارث‌بری قدیمی برای یک حساب نام‌دار نیاز دارید،
`dangerouslyAllowInheritedWebhookPath: true` را در آن حساب یا در `channels.synology-chat` تنظیم کنید،
اما مسیرهای دقیق تکراری همچنان با رد ایمن مواجه می‌شوند. مسیرهای صریح برای هر حساب را ترجیح دهید.

```json5
{
  channels: {
    "synology-chat": {
      enabled: true,
      accounts: {
        default: {
          token: "token-a",
          incomingUrl: "https://nas-a.example.com/...token=...",
        },
        alerts: {
          token: "token-b",
          incomingUrl: "https://nas-b.example.com/...token=...",
          webhookPath: "/webhook/synology-alerts",
          dmPolicy: "allowlist",
          allowedUserIds: ["987654"],
        },
      },
    },
  },
}
```

## نکات امنیتی

- `token` را محرمانه نگه دارید و در صورت افشا آن را تغییر دهید.
- مگر اینکه صراحتاً به گواهی خودامضاشدهٔ NAS محلی اعتماد دارید، `allowInsecureSsl: false` را حفظ کنید.
- درخواست‌های Webhook ورودی از نظر توکن تأیید می‌شوند و برای هر فرستنده محدودیت نرخ دارند (`rateLimitPerMinute`، پیش‌فرض 30).
- بررسی توکن نامعتبر از مقایسهٔ زمان‌ثابت اسرار استفاده می‌کند و با رد ایمن مواجه می‌شود؛ تلاش‌های مکرر با توکن نامعتبر، نشانی IP مبدأ را موقتاً مسدود می‌کند.
- متن پیام ورودی در برابر الگوهای شناخته‌شدهٔ تزریق پرامپت پاک‌سازی و در 4000 نویسه کوتاه می‌شود.
- برای محیط عملیاتی، `dmPolicy: "allowlist"` را ترجیح دهید.
- مگر اینکه صراحتاً به تحویل پاسخ قدیمی مبتنی بر نام کاربری نیاز دارید، `dangerouslyAllowNameMatching` را خاموش نگه دارید.
- مگر اینکه صراحتاً خطر مسیریابی با مسیر مشترک را در یک راه‌اندازی چندحسابی می‌پذیرید، `dangerouslyAllowInheritedWebhookPath` را خاموش نگه دارید.

## عیب‌یابی

- `Missing required fields (token, user_id, text)`:
  - یکی از فیلدهای الزامی در محمولهٔ Webhook خروجی وجود ندارد
  - اگر Synology توکن را در سرآیندها ارسال می‌کند، مطمئن شوید Gateway/پراکسی آن سرآیندها را حفظ می‌کند
- `Invalid token`:
  - رمز Webhook خروجی با `channels.synology-chat.token` مطابقت ندارد
  - درخواست به حساب/مسیر Webhook اشتباه ارسال می‌شود
  - یک پراکسی معکوس پیش از رسیدن درخواست به OpenClaw، سرآیند توکن را حذف کرده است
- `Rate limit exceeded`:
  - تلاش‌های بیش‌ازحد با توکن نامعتبر از یک مبدأ می‌تواند آن مبدأ را موقتاً مسدود کند
  - فرستندگان احرازشده نیز محدودیت نرخ پیام جداگانه‌ای برای هر کاربر دارند
- `Allowlist is empty. Configure allowedUserIds or use dmPolicy=open with allowedUserIds=["*"].`:
  - `dmPolicy="allowlist"` فعال است اما هیچ کاربری پیکربندی نشده است
- `User not authorized`:
  - `user_id` عددی فرستنده در `allowedUserIds` نیست

## مرتبط

- [نمای کلی کانال‌ها](/fa/channels) — همهٔ کانال‌های پشتیبانی‌شده
- [گروه‌ها](/fa/channels/groups) — رفتار گفت‌وگوی گروهی و کنترل اشاره
- [مسیریابی کانال](/fa/channels/channel-routing) — مسیریابی نشست برای پیام‌ها
- [امنیت](/fa/gateway/security) — مدل دسترسی و مقاوم‌سازی
