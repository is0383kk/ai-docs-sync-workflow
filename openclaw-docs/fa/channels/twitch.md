---
read_when:
    - راه‌اندازی یکپارچه‌سازی چت Twitch برای OpenClaw
sidebarTitle: Twitch
summary: 'ربات چت Twitch: نصب، اطلاعات احراز هویت، کنترل دسترسی، تازه‌سازی توکن'
title: Twitch
x-i18n:
    generated_at: "2026-07-27T13:45:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: d827c742ded5fd0b071443dead27b975e2414419b0facb486d7f9c0c9800b060
    source_path: channels/twitch.md
    workflow: 16
---

پشتیبانی از چت Twitch از طریق رابط چت (IRC) این سرویس و با استفاده از کلاینت Twurple. OpenClaw با یک حساب ربات Twitch وارد می‌شود، برای هر حساب پیکربندی‌شده به یک کانال می‌پیوندد و در همان کانال پاسخ می‌دهد.

## نصب

Twitch به‌عنوان یک Plugin رسمی عرضه می‌شود و بخشی از نصب هسته نیست.

<Tabs>
  <Tab title="رجیستری npm">
    ```bash
    openclaw plugins install @openclaw/twitch
    ```
  </Tab>
  <Tab title="نسخه محلی مخزن">
    ```bash
    openclaw plugins install ./path/to/local/twitch-plugin
    ```
  </Tab>
</Tabs>

`plugins install`، Plugin را ثبت و فعال می‌کند. انتخاب Twitch هنگام `openclaw onboard` یا `openclaw channels add` آن را در صورت نیاز نصب می‌کند. برای دنبال‌کردن نسخه انتشار فعلی، از نام ساده بسته استفاده کنید؛ فقط برای نصب‌های بازتولیدپذیر یک نسخه دقیق را ثابت کنید. به OpenClaw 2026.4.10 یا جدیدتر نیاز دارد.

جزئیات: [Pluginها](/fa/tools/plugin)

## راه‌اندازی سریع

<Steps>
  <Step title="نصب Plugin">
    بخش [نصب](#install) در بالا را ببینید.
  </Step>
  <Step title="ایجاد حساب ربات Twitch">
    یک حساب اختصاصی Twitch برای ربات ایجاد کنید (یا از یک حساب موجود استفاده کنید).
  </Step>
  <Step title="ایجاد اطلاعات اعتبارسنجی">
    از [مولد توکن Twitch](https://twitchtokengenerator.com/) استفاده کنید:

    - گزینه **Bot Token** را انتخاب کنید
    - بررسی کنید که دامنه‌های دسترسی `chat:read` و `chat:write` انتخاب شده باشند
    - مقادیر **Client ID** و **Access Token** را کپی کنید

  </Step>
  <Step title="یافتن شناسه کاربری Twitch">
    با استفاده از [https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) یک نام کاربری را به شناسه کاربری Twitch تبدیل کنید.
  </Step>
  <Step title="پیکربندی توکن">
    - متغیر محیطی: `OPENCLAW_TWITCH_ACCESS_TOKEN=...` (فقط حساب پیش‌فرض)
    - یا پیکربندی: `channels.twitch.accessToken`

    اگر هر دو تنظیم شده باشند، پیکربندی اولویت دارد (متغیر محیطی فقط برای حساب پیش‌فرض نقش جایگزین را دارد).

  </Step>
  <Step title="راه‌اندازی Gateway">
    ```bash
    openclaw gateway run
    ```
  </Step>
</Steps>

<Warning>
برای جلوگیری از فعال‌کردن ربات توسط کاربران غیرمجاز، کنترل دسترسی (`allowFrom` یا `allowedRoles`) را اضافه کنید. مقدار پیش‌فرض `requireMention` برابر با `true` است.
</Warning>

پیکربندی حداقلی:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw", // حساب Twitch ربات (برای احراز هویت)
      accessToken: "oauth:abc123...", // توکن دسترسی OAuth (یا استفاده از متغیر محیطی OPENCLAW_TWITCH_ACCESS_TOKEN)
      clientId: "xyz789...", // شناسه کلاینت از Token Generator
      channel: "yourchannel", // چت کدام کانال Twitch باید پیوسته شود (الزامی)
      allowFrom: ["123456789"], // (توصیه‌شده) فقط شناسه کاربری Twitch شما
    },
  },
}
```

## چیستی آن

- یک کانال Twitch تحت مالکیت Gateway.
- مسیریابی قطعی: پاسخ‌ها همیشه به همان کانال Twitch بازمی‌گردند که پیام از آن آمده است.
- هر کانال پیوسته‌شده به یک کلید نشست گروهی مجزا به نام `agent:<agentId>:twitch:group:<channel>` نگاشت می‌شود.
- `username` حساب ربات است (حساب احراز هویت‌کننده) و `channel` اتاق چتی است که باید به آن پیوست. هر ورودی حساب دقیقاً به یک کانال می‌پیوندد.
- توکن‌ها با پیشوند `oauth:` یا بدون آن کار می‌کنند؛ OpenClaw هر دو حالت را یکسان‌سازی می‌کند (دستیار راه‌اندازی قالب `oauth:` را انتظار دارد).

## دوام پیام‌های ورودی

OpenClaw هر پیام پذیرفته‌شده چت Twitch را پیش از ارسال عادی، به‌صورت پایدار در صف قرار می‌دهد. پیام‌های در انتظار یا قابل‌تلاش مجدد پس از راه‌اندازی دوباره Gateway باقی می‌مانند، برای کانال پیکربندی‌شده به‌شکل ترتیبی پردازش می‌شوند و تا زمانی که رکورد تکمیل فعال یا نگه‌داری‌شده وجود دارد، از شناسه پیام Twitch برای جلوگیری از ورودی‌های تکراری صف استفاده می‌کنند.

پس از آنکه کلاینت یک `PRIVMSG` را پذیرفت، چت Twitch آن را دوباره پخش نمی‌کند. این سازوکار از بازه خرابی محلی میان پذیرش و ارسال محافظت می‌کند، اما نمی‌تواند پیام‌هایی را که پیش از پذیرش پایدار از دست رفته‌اند بازیابی کند. اگر خود افزودن به صف ناموفق باشد، OpenClaw خطا را ثبت می‌کند؛ اتصال مجدد از Twitch نمی‌خواهد آن پیام را دوباره ارسال کند.

## تازه‌سازی توکن (اختیاری)

توکن‌های [مولد توکن Twitch](https://twitchtokengenerator.com/) را OpenClaw نمی‌تواند تازه‌سازی کند؛ پس از انقضا آن‌ها را دوباره ایجاد کنید (چند ساعت دوام دارند و نیازی به ثبت برنامه نیست).

برای تازه‌سازی خودکار، برنامه خود را در [کنسول توسعه‌دهندگان Twitch](https://dev.twitch.tv/console) ایجاد کنید و موارد زیر را اضافه کنید:

```json5
{
  channels: {
    twitch: {
      clientSecret: "your_client_secret",
      refreshToken: "your_refresh_token",
    },
  },
}
```

با تنظیم هر دو مقدار، Plugin از یک ارائه‌دهنده احراز هویت با قابلیت تازه‌سازی استفاده می‌کند که توکن‌ها را پیش از انقضا تمدید و هر تازه‌سازی را ثبت می‌کند. بدون `refreshToken`، پیام `token refresh disabled (no refresh token)` را ثبت می‌کند؛ بدون `clientSecret`، به یک توکن ایستا (بدون قابلیت تازه‌سازی) برمی‌گردد.

## پشتیبانی از چند حساب

از `channels.twitch.accounts` همراه با اطلاعات اعتبارسنجی مختص هر حساب استفاده کنید. برای الگوی مشترک، [پیکربندی](/fa/gateway/configuration) را ببینید.

نمونه (یک حساب ربات در دو کانال):

```json5
{
  channels: {
    twitch: {
      accounts: {
        channel1: {
          username: "openclaw",
          accessToken: "oauth:abc123...",
          clientId: "xyz789...",
          channel: "yourchannel",
        },
        channel2: {
          username: "openclaw",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "secondchannel",
        },
      },
    },
  },
}
```

<Note>
هر ورودی حساب به `accessToken` مختص خود نیاز دارد (متغیر محیطی فقط حساب پیش‌فرض را پوشش می‌دهد). هر حساب دقیقاً به یک کانال می‌پیوندد، بنابراین پیوستن به دو کانال به دو حساب نیاز دارد. `channels.twitch.defaultAccount` تعیین می‌کند کدام حساب پیش‌فرض باشد.
</Note>

## کنترل دسترسی

`allowFrom` یک فهرست مجاز سخت‌گیرانه از شناسه‌های کاربری Twitch است. وقتی تنظیم شده باشد، `allowedRoles` نادیده گرفته می‌شود؛ برای استفاده از دسترسی مبتنی بر نقش، `allowFrom` را تنظیم‌نشده باقی بگذارید.

**نقش‌های موجود:** `"moderator"`، `"owner"`، `"vip"`، `"subscriber"`، `"all"`.

<Tabs>
  <Tab title="فهرست مجاز شناسه کاربری (امن‌ترین)">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowFrom: ["123456789", "987654321"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="مبتنی بر نقش">
    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              allowedRoles: ["moderator", "vip"],
            },
          },
        },
      },
    }
    ```
  </Tab>
  <Tab title="غیرفعال‌کردن الزام @mention">
    مقدار پیش‌فرض `requireMention` برابر با `true` است. برای پاسخ‌دادن به همه پیام‌های مجاز:

    ```json5
    {
      channels: {
        twitch: {
          accounts: {
            default: {
              requireMention: false,
            },
          },
        },
      },
    }
    ```

  </Tab>
</Tabs>

<Note>
**چرا شناسه کاربری؟** نام‌های کاربری می‌توانند تغییر کنند و امکان جعل هویت را فراهم کنند. شناسه‌های کاربری دائمی هستند.

شناسه خود را با [مبدل نام کاربری به شناسه](https://www.streamweasels.com/tools/convert-twitch-username-to-user-id/) پیدا کنید.
</Note>

## عیب‌یابی

ابتدا فرمان‌های تشخیصی را اجرا کنید:

```bash
openclaw doctor
openclaw channels status --probe
```

<AccordionGroup>
  <Accordion title="ربات به پیام‌ها پاسخ نمی‌دهد">
    - **کنترل دسترسی را بررسی کنید:** مطمئن شوید شناسه کاربری شما در `allowFrom` قرار دارد؛ یا برای آزمایش، موقتاً `allowFrom` را حذف و `allowedRoles: ["all"]` را تنظیم کنید.
    - **محدودیت منشن را بررسی کنید:** با `requireMention: true` (مقدار پیش‌فرض)، پیام‌ها باید نام کاربری ربات را @mention کنند.
    - **حضور ربات در کانال را بررسی کنید:** ربات فقط به کانالی می‌پیوندد که در `channel` نام‌گذاری شده است.

  </Accordion>
  <Accordion title="مشکلات توکن">
    خطای «اتصال ناموفق بود» یا خطاهای احراز هویت:

    - بررسی کنید `accessToken` مقدار توکن دسترسی OAuth باشد (پیشوند `oauth:` اختیاری است)
    - بررسی کنید توکن دامنه‌های دسترسی `chat:read` و `chat:write` را داشته باشد
    - اگر از تازه‌سازی توکن استفاده می‌کنید، بررسی کنید `clientSecret` و `refreshToken` تنظیم شده باشند

  </Accordion>
  <Accordion title="تازه‌سازی توکن کار نمی‌کند">
    گزارش‌ها را برای رویدادهای تازه‌سازی بررسی کنید:

    ```text
    استفاده از منبع توکن محیطی برای mybot
    توکن دسترسی برای کاربر 123456 تازه‌سازی شد (انقضا پس از 14400s)
    ```

    اگر `token refresh disabled (no refresh token)` را می‌بینید:

    - مطمئن شوید `clientSecret` ارائه شده است
    - مطمئن شوید `refreshToken` ارائه شده است

  </Accordion>
</AccordionGroup>

## پیکربندی

### پیکربندی حساب

<ParamField path="username" type="string" required>
  نام کاربری ربات (حساب احراز هویت‌کننده).
</ParamField>
<ParamField path="accessToken" type="string" required>
  توکن دسترسی OAuth با `chat:read` و `chat:write` (پیکربندی یا متغیر محیطی برای حساب پیش‌فرض).
</ParamField>
<ParamField path="clientId" type="string" required>
  شناسه کلاینت Twitch (از Token Generator یا برنامه شما). در طرح‌واره اختیاری است، اما برای اتصال الزامی است.
</ParamField>
<ParamField path="channel" type="string" required>
  کانالی که باید به آن پیوست.
</ParamField>
<ParamField path="enabled" type="boolean" default="true">
  این حساب را فعال می‌کند.
</ParamField>
<ParamField path="clientSecret" type="string">
  اختیاری: برای تازه‌سازی خودکار توکن.
</ParamField>
<ParamField path="refreshToken" type="string">
  اختیاری: برای تازه‌سازی خودکار توکن.
</ParamField>
<ParamField path="expiresIn" type="number">
  زمان انقضای توکن برحسب ثانیه (ردیابی تازه‌سازی).
</ParamField>
<ParamField path="obtainmentTimestamp" type="number">
  برچسب زمانی دریافت توکن (ردیابی تازه‌سازی).
</ParamField>
<ParamField path="allowFrom" type="string[]">
  فهرست مجاز شناسه‌های کاربری. وقتی تنظیم شود، نقش‌ها نادیده گرفته می‌شوند.
</ParamField>
<ParamField path="allowedRoles" type='Array<"moderator" | "owner" | "vip" | "subscriber" | "all">'>
  کنترل دسترسی مبتنی بر نقش.
</ParamField>
<ParamField path="requireMention" type="boolean" default="true">
  برای فعال‌شدن ربات به @mention نیاز دارد.
</ParamField>
<ParamField path="responsePrefix" type="string">
  جایگزینی پیشوند پاسخ خروجی برای این حساب.
</ParamField>

### گزینه‌های ارائه‌دهنده

- `channels.twitch.enabled` - فعال/غیرفعال‌کردن راه‌اندازی کانال
- `channels.twitch.username` / `accessToken` / `clientId` / `channel` - پیکربندی ساده‌شده تک‌حسابی (حساب ضمنی `default`؛ بر `accounts.default` اولویت دارد)
- `channels.twitch.accounts.<accountName>` - پیکربندی چندحسابی (همه فیلدهای حساب در بالا)
- `channels.twitch.defaultAccount` - نام حسابی که پیش‌فرض است
- `channels.twitch.markdown.tables` - حالت رندر جدول Markdown (`off` | `bullets` | `code` | `block`)

نمونه کامل:

```json5
{
  channels: {
    twitch: {
      enabled: true,
      username: "openclaw",
      accessToken: "oauth:abc123...",
      clientId: "xyz789...",
      channel: "yourchannel",
      clientSecret: "secret123...",
      refreshToken: "refresh456...",
      allowFrom: ["123456789"],
      accounts: {
        second: {
          username: "mybot",
          accessToken: "oauth:def456...",
          clientId: "uvw012...",
          channel: "your_channel",
          enabled: true,
          expiresIn: 14400,
          obtainmentTimestamp: 1706092800000,
          allowedRoles: ["moderator"],
        },
      },
    },
  },
}
```

## کنش‌های ابزار

عامل می‌تواند از طریق کنش `send` ابزار پیام، پیام‌های Twitch را ارسال کند:

```json5
{
  channel: "twitch",
  action: "send",
  to: "#mychannel",
  message: "سلام Twitch!",
}
```

`to` اختیاری است و مقدار پیش‌فرض آن `channel` پیکربندی‌شده حساب است.

## ایمنی و عملیات

- **با توکن‌ها مانند گذرواژه‌ها رفتار کنید** - هرگز توکن‌ها را در git ثبت نکنید.
- برای ربات‌های طولانی‌مدت **از تازه‌سازی خودکار توکن استفاده کنید**.
- برای کنترل دسترسی، به‌جای نام‌های کاربری **از فهرست‌های مجاز شناسهٔ کاربر استفاده کنید**.
- برای رویدادهای تازه‌سازی توکن و وضعیت اتصال، **لاگ‌ها را پایش کنید**.
- **دامنهٔ دسترسی توکن‌ها را به حداقل برسانید** - فقط `chat:read` و `chat:write` را درخواست کنید.
- **اگر به مشکل برخوردید**: پس از اطمینان از اینکه هیچ فرایند دیگری مالک نشست نیست، Gateway را راه‌اندازی مجدد کنید.

## محدودیت‌ها

- **500 نویسه** برای هر پیام؛ پاسخ‌های طولانی‌تر در مرز واژه‌ها به بخش‌های کوچک‌تر تقسیم می‌شوند.
- Markdown پیش از ارسال حذف می‌شود (گفت‌وگوی Twitch متن ساده است؛ خط‌های جدید به فاصله تبدیل می‌شوند).
- OpenClaw هیچ محدودیت نرخی از جانب خود اعمال نمی‌کند؛ کلاینت گفت‌وگوی Twurple محدودیت‌های نرخ Twitch را مدیریت می‌کند.

## مرتبط

- [مسیریابی کانال](/fa/channels/channel-routing) — مسیریابی نشست برای پیام‌ها
- [نمای کلی کانال‌ها](/fa/channels) — همهٔ کانال‌های پشتیبانی‌شده
- [گروه‌ها](/fa/channels/groups) — رفتار گفت‌وگوی گروهی و محدودسازی بر اساس اشاره
- [جفت‌سازی](/fa/channels/pairing) — احراز هویت پیام مستقیم و جریان جفت‌سازی
- [امنیت](/fa/gateway/security) — مدل دسترسی و مقاوم‌سازی
