---
read_when:
    - می‌خواهید OpenClaw را به LINE متصل کنید
    - به راه‌اندازی Webhook و اعتبارنامه‌های LINE نیاز دارید
    - گزینه‌های پیام مخصوص LINE را می‌خواهید
summary: راه‌اندازی، پیکربندی و استفاده از Plugin پیام‌رسانی LINE
title: LINE
x-i18n:
    generated_at: "2026-07-27T16:14:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: aa160970278e0899637307136139f7d2fc83bf57defc30771d77649060f77274
    source_path: channels/line.md
    workflow: 16
---

LINE از طریق LINE Messaging API به OpenClaw متصل می‌شود. Plugin به‌عنوان گیرنده Webhook
روی Gateway اجرا می‌شود و برای احراز هویت از توکن دسترسی کانال + راز کانال شما
استفاده می‌کند.

وضعیت: Plugin رسمی که جداگانه نصب می‌شود. پیام‌های مستقیم، گفت‌وگوهای گروهی، رسانه،
موقعیت‌ها، پیام‌های Flex، پیام‌های قالبی و پاسخ‌های سریع پشتیبانی می‌شوند.
واکنش‌ها و رشته‌ها پشتیبانی نمی‌شوند.

## نصب

پیش از پیکربندی کانال، LINE را نصب کنید:

```bash
openclaw plugins install @openclaw/line
```

نسخه محلی (هنگام اجرا از یک مخزن git):

```bash
openclaw plugins install ./path/to/local/line-plugin
```

## راه‌اندازی

1. یک حساب LINE Developers ایجاد و Console را باز کنید:
   [https://developers.line.biz/console/](https://developers.line.biz/console/)
2. یک Provider ایجاد (یا انتخاب) کنید و یک کانال **Messaging API** بیفزایید.
3. مقادیر **Channel access token** و **Channel secret** را از تنظیمات کانال کپی کنید.
4. گزینه **Use webhook** را در تنظیمات Messaging API فعال کنید.
5. نشانی Webhook را روی نقطه پایانی Gateway خود تنظیم کنید (HTTPS الزامی است):

```text
https://gateway-host/line/webhook
```

Gateway به تأیید Webhook مربوط به LINE ‏(GET) پاسخ می‌دهد. برای رویدادهای ورودی امضاشده
(POST)، پیش از بازگرداندن `200` هر رویداد را در صف ورودی پایدار می‌نویسد؛
پردازش عامل به‌صورت ناهمگام ادامه می‌یابد. تحویل ناموفق از صف دوباره امتحان می‌شود،
از جمله پس از راه‌اندازی مجدد Gateway، و رویدادهای مسموم پس از تعداد محدودی تلاش مجدد
به رکوردهای ناموفق صف تبدیل می‌شوند. اگر ماندگاری پایدار شکست بخورد، درخواست به‌جای
تأیید رویدادی که ممکن است از دست برود، `500` را بازمی‌گرداند.
تحویل در مرز صف به عامل حداقل یک‌بار انجام می‌شود: خاموشی یا خرابی Gateway هنگام
تحویل فعال ممکن است نوبت را دوباره اجرا کند. رویدادهای پیام بر اساس شناسه پیام
LINE رفع تکرار می‌شوند؛ انواع دیگر رویداد از `webhookEventId` استفاده می‌کنند. رکوردهای تکمیل نگه‌داری‌شده
Webhookهای تکراری معمول را سرکوب می‌کنند، اما کنترل‌گرهایی که عوارض جانبی خارجی دارند
همچنان باید هم‌توان باشند.
اگر به مسیر سفارشی نیاز دارید، `channels.line.webhookPath` یا
`channels.line.accounts.<id>.webhookPath` را تنظیم و نشانی را متناسب با آن به‌روزرسانی کنید.

نکات امنیتی:

- تأیید امضای LINE به بدنه وابسته است (HMAC روی بدنه خام)، بنابراین OpenClaw پیش از تأیید، محدودیت سخت‌گیرانه بدنه پیش از احراز هویت (64 KB) و مهلت خواندن اعمال می‌کند.
- OpenClaw رویدادهای Webhook را از بایت‌های خام تأییدشده درخواست پردازش می‌کند. مقادیر `req.body` که توسط میان‌افزار بالادستی تغییر یافته‌اند، برای حفظ یکپارچگی امضا نادیده گرفته می‌شوند.

## پیکربندی

پیکربندی حداقلی:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "pairing",
    },
  },
}
```

پیکربندی پیام مستقیم عمومی:

```json5
{
  channels: {
    line: {
      enabled: true,
      channelAccessToken: "LINE_CHANNEL_ACCESS_TOKEN",
      channelSecret: "LINE_CHANNEL_SECRET",
      dmPolicy: "open",
      allowFrom: ["*"],
    },
  },
}
```

متغیرهای محیطی (فقط حساب پیش‌فرض):

- `LINE_CHANNEL_ACCESS_TOKEN`
- `LINE_CHANNEL_SECRET`

فایل‌های توکن/راز:

```json5
{
  channels: {
    line: {
      tokenFile: "/path/to/line-token.txt",
      secretFile: "/path/to/line-secret.txt",
    },
  },
}
```

`tokenFile` و `secretFile` باید به فایل‌های عادی اشاره کنند. پیوندهای نمادین رد می‌شوند.
مقادیر درون‌خطی پیکربندی بر فایل‌ها اولویت دارند؛ متغیرهای محیطی آخرین گزینه جایگزین برای حساب پیش‌فرض هستند.

چند حساب:

```json5
{
  channels: {
    line: {
      accounts: {
        marketing: {
          channelAccessToken: "...",
          channelSecret: "...",
          webhookPath: "/line/marketing",
        },
      },
    },
  },
}
```

## کنترل دسترسی

پیام‌های مستقیم به‌طور پیش‌فرض از جفت‌سازی استفاده می‌کنند. فرستندگان ناشناس یک کد
جفت‌سازی دریافت می‌کنند و پیام‌هایشان تا زمان تأیید نادیده گرفته می‌شود:

```bash
openclaw pairing list line
openclaw pairing approve line <CODE>
```

فهرست‌های مجاز و سیاست‌ها:

- `channels.line.dmPolicy`: `pairing | allowlist | open | disabled` (پیش‌فرض `pairing`)
- `channels.line.allowFrom`: شناسه‌های کاربری مجاز LINE برای پیام‌های مستقیم؛ `dmPolicy: "open"` به `["*"]` نیاز دارد
- `channels.line.groupPolicy`: `allowlist | open | disabled` (پیش‌فرض `allowlist`)
- `channels.line.groupAllowFrom`: شناسه‌های کاربری مجاز LINE برای گروه‌ها؛ ورودی‌های پیام مستقیم `allowFrom` به فرستندگان گروه اجازه ورود نمی‌دهند
- بازنویسی‌های مختص هر گروه: `channels.line.groups.<groupId>.allowFrom` (به‌همراه `enabled`، `requireMention`، `systemPrompt`، `skills`). با
  `groupPolicy: "allowlist"`، ‏`groupAllowFrom` یا `allowFrom` مختص هر گروه را تنظیم کنید؛ فهرست مجاز خالی گروه، حتی زمانی که پیام‌های مستقیم باز هستند، پیام‌های گروه را مسدود می‌کند.
- گروه‌های ایستای دسترسی فرستنده را می‌توان از `allowFrom`، `groupAllowFrom` و `allowFrom` مختص هر گروه با `accessGroup:<name>` ارجاع داد؛ [گروه‌های دسترسی](/fa/channels/access-groups) را ببینید.
- نکته زمان اجرا: اگر `channels.line` کاملاً وجود نداشته باشد، زمان اجرا برای بررسی‌های گروه به `groupPolicy="allowlist"` برمی‌گردد (حتی اگر `channels.defaults.groupPolicy` تنظیم شده باشد).

شناسه‌های LINE به بزرگی و کوچکی حروف حساس هستند. شناسه‌های معتبر به این شکل‌اند:

- کاربر: `U` + 32 نویسه هگزادسیمال
- گروه: `C` + 32 نویسه هگزادسیمال
- اتاق: `R` + 32 نویسه هگزادسیمال

## رفتار پیام

- متن در 5000 نویسه قطعه‌بندی می‌شود.
- قالب‌بندی Markdown حذف می‌شود؛ بلوک‌های کد و جدول‌ها در صورت امکان به کارت‌های Flex
  تبدیل می‌شوند.
- پاسخ‌های جریانی بافر می‌شوند؛ هنگام کار عامل، LINE قطعه‌های کامل را همراه با پویانمایی
  بارگذاری دریافت می‌کند.
- حجم دانلود رسانه با `channels.line.mediaMaxMb` محدود می‌شود (پیش‌فرض 10).
- رسانه ورودی پیش از ارسال به عامل در `~/.openclaw/media/inbound/` ذخیره می‌شود
  و با مخزن رسانه مشترک مورداستفاده سایر Pluginهای کانال مطابقت دارد.

## داده‌های کانال (پیام‌های غنی)

برای ارسال پاسخ‌های سریع، موقعیت‌ها، کارت‌های Flex یا پیام‌های قالبی از
`channelData.line` استفاده کنید.

```json5
{
  text: "بفرمایید",
  channelData: {
    line: {
      quickReplies: ["وضعیت", "راهنما"],
      location: {
        title: "دفتر",
        address: "خیابان اصلی، پلاک 123",
        latitude: 35.681236,
        longitude: 139.767125,
      },
      flexMessage: {
        altText: "کارت وضعیت",
        contents: {/* محتوای Flex */},
      },
      templateMessage: {
        type: "confirm",
        text: "ادامه داده شود؟",
        confirmLabel: "بله",
        confirmData: "yes",
        cancelLabel: "خیر",
        cancelData: "no",
      },
    },
  },
}
```

Plugin مربوط به LINE همچنین یک فرمان `/card` برای پیش‌تنظیم‌های پیام Flex ارائه می‌کند:

```text
/card info "خوش آمدید" "از پیوستن شما سپاسگزاریم!"
```

## پشتیبانی از ACP

LINE از اتصال مکالمه‌های ACP ‏(پروتکل ارتباط عامل) پشتیبانی می‌کند:

- `/acp spawn <agent> --bind here` گفت‌وگوی فعلی LINE را بدون ایجاد رشته فرزند به یک نشست ACP متصل می‌کند.
- اتصال‌های پیکربندی‌شده ACP و نشست‌های فعال ACP متصل به مکالمه، در LINE مانند سایر کانال‌های مکالمه کار می‌کنند.

برای جزئیات، [عامل‌های ACP](/fa/tools/acp-agents) را ببینید.

## رسانه خروجی

Plugin مربوط به LINE تصاویر، ویدئوها و صداها را از طریق ابزار پیام عامل ارسال می‌کند:

- **تصاویر**: به‌صورت پیام تصویری LINE ارسال می‌شوند؛ تصویر پیش‌نمایش به‌طور پیش‌فرض همان نشانی رسانه است.
- **ویدئوها**: به تصویر پیش‌نمایش نیاز دارند؛ `channelData.line.previewImageUrl` را روی نشانی یک تصویر تنظیم کنید.
- **صدا**: به‌صورت پیام صوتی LINE ارسال می‌شود؛ مدت‌زمان به‌طور پیش‌فرض 60 ثانیه است، مگر اینکه `channelData.line.durationMs` تنظیم شده باشد.

نوع رسانه در صورت تنظیم از `channelData.line.mediaKind` گرفته می‌شود؛ در غیر این صورت
از سایر گزینه‌های LINE یا پسوند فایل نشانی استنباط می‌شود و تصویر گزینه جایگزین است.

نشانی‌های رسانه خروجی باید نشانی‌های عمومی HTTPS با حداکثر 2000 نویسه باشند. OpenClaw
نام میزبان مقصد را پیش از تحویل نشانی به LINE اعتبارسنجی می‌کند و مقصدهای loopback،
link-local و شبکه خصوصی را رد می‌کند.

ارسال‌های عمومی رسانه بدون گزینه‌های مختص LINE از مسیر تصویر استفاده می‌کنند.

## عیب‌یابی

- **تأیید Webhook شکست می‌خورد:** مطمئن شوید نشانی Webhook از HTTPS استفاده می‌کند و
  `channelSecret` با LINE console مطابقت دارد.
- **هیچ رویداد ورودی دریافت نمی‌شود:** تأیید کنید مسیر Webhook با `channels.line.webhookPath`
  مطابقت دارد و Gateway از LINE قابل دسترسی است.
- **خطاهای دانلود رسانه:** اگر حجم رسانه از محدودیت پیش‌فرض بیشتر است، `channels.line.mediaMaxMb` را
  افزایش دهید.

## مطالب مرتبط

- [نمای کلی کانال‌ها](/fa/channels) — همه کانال‌های پشتیبانی‌شده
- [جفت‌سازی](/fa/channels/pairing) — احراز هویت پیام مستقیم و جریان جفت‌سازی
- [گروه‌ها](/fa/channels/groups) — رفتار گفت‌وگوی گروهی و کنترل اشاره
- [مسیریابی کانال](/fa/channels/channel-routing) — مسیریابی نشست برای پیام‌ها
- [امنیت](/fa/gateway/security) — مدل دسترسی و مقاوم‌سازی
