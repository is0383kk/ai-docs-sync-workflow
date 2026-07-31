---
read_when:
    - می‌خواهید یک ربات Yuanbao متصل کنید
    - در حال پیکربندی کانال Yuanbao هستید
summary: نمای کلی، قابلیت‌ها و پیکربندی ربات Yuanbao
title: Yuanbao
x-i18n:
    generated_at: "2026-07-27T14:58:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 43488834f588530206b290cb0fb185fd1fe2e1f214ab4a4ccccc49b9b549b6ac
    source_path: channels/yuanbao.md
    workflow: 16
---

Tencent Yuanbao پلتفرم دستیار هوش مصنوعی Tencent است. Plugin نگه‌داری‌شده توسط جامعه، `openclaw-plugin-yuanbao`، ربات‌های Yuanbao را از طریق WebSocket برای پیام‌های مستقیم و گفت‌وگوهای گروهی به OpenClaw متصل می‌کند.

**وضعیت:** آماده استفاده در محیط عملیاتی برای پیام‌های مستقیم ربات و گفت‌وگوهای گروهی. WebSocket تنها حالت اتصال پشتیبانی‌شده است. این Plugin را تیم Tencent Yuanbao به‌عنوان یک ورودی کاتالوگ خارجی نگه‌داری می‌کند، نه هسته OpenClaw؛ جزئیات پیکربندی/رفتار زیر (به‌جز نصب و سطح عمومی CLI) از مستندات خود Plugin گرفته شده‌اند و در برابر کد منبع هسته OpenClaw تأیید نشده‌اند.

## شروع سریع

به OpenClaw 2026.4.10 یا بالاتر نیاز دارد. با `openclaw --version` بررسی و با `openclaw update` ارتقا دهید.

<Steps>
  <Step title="کانال Yuanbao را با اطلاعات اعتبارسنجی خود اضافه کنید">
  ```bash
  openclaw channels add --channel yuanbao --token "appKey:appSecret"
  ```
  `--token` از `appKey:appSecret` جداشده با دونقطه استفاده می‌کند. این موارد را با ایجاد یک ربات در تنظیمات برنامه خود، از برنامه Yuanbao دریافت کنید.
  </Step>

  <Step title="برای اعمال تغییر، Gateway را دوباره راه‌اندازی کنید">
  ```bash
  openclaw gateway restart
  ```
  </Step>
</Steps>

### راه‌اندازی تعاملی (جایگزین)

```bash
openclaw channels login --channel yuanbao
```

برای وارد کردن App ID و App Secret، اعلان‌ها را دنبال کنید.

## کنترل دسترسی

### پیام‌های مستقیم

`channels.yuanbao.dm.policy`:

| مقدار            | رفتار                                          |
| ---------------- | ------------------------------------------------- |
| `open` (پیش‌فرض) | اجازه به همه کاربران                                   |
| `pairing`        | کاربران ناشناس یک کد جفت‌سازی دریافت می‌کنند؛ از طریق CLI تأیید کنید |
| `allowlist`      | فقط کاربران موجود در `allowFrom` می‌توانند گفت‌وگو کنند                |
| `disabled`       | غیرفعال‌کردن همه پیام‌های مستقیم                                   |

یک درخواست جفت‌سازی را تأیید کنید:

```bash
openclaw pairing list yuanbao
openclaw pairing approve yuanbao <CODE>
```

### گفت‌وگوهای گروهی

`channels.yuanbao.requireMention` (پیش‌فرض `true`): پیش از پاسخ‌دادن ربات در گروه، به @mention نیاز است. پاسخ‌دادن به پیام خود ربات، اشاره ضمنی تلقی می‌شود.

## نمونه‌های پیکربندی

راه‌اندازی پایه، سیاست باز پیام مستقیم:

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "open",
      },
    },
  },
}
```

محدودکردن پیام‌های مستقیم به کاربران مشخص:

```json5
{
  channels: {
    yuanbao: {
      appKey: "your_app_key",
      appSecret: "your_app_secret",
      dm: {
        policy: "allowlist",
        allowFrom: ["user_id_1", "user_id_2"],
      },
    },
  },
}
```

غیرفعال‌کردن الزام @mention در گروه‌ها:

```json5
{
  channels: {
    yuanbao: {
      requireMention: false,
    },
  },
}
```

تنظیم تحویل خروجی:

```json5
{
  channels: {
    yuanbao: {
      outboundQueueStrategy: "merge-text",
      minChars: 2800, // تا رسیدن به این تعداد نویسه در بافر نگه دارید
      maxChars: 3000, // بالاتر از این حد، تقسیم را اجباری کنید
      idleMs: 5000, // پس از پایان مهلت بی‌کاری، به‌طور خودکار تخلیه کنید (میلی‌ثانیه)
    },
  },
}
```

برای ارسال هر قطعه بدون بافرکردن، `outboundQueueStrategy: "immediate"` را تنظیم کنید.

## فرمان‌های رایج

| فرمان    | توضیح                 |
| ---------- | --------------------------- |
| `/help`    | نمایش فرمان‌های موجود     |
| `/status`  | نمایش وضعیت ربات             |
| `/new`     | شروع یک نشست جدید         |
| `/stop`    | توقف اجرای فعلی        |
| `/restart` | راه‌اندازی دوباره OpenClaw            |
| `/compact` | فشرده‌سازی زمینه نشست |

Yuanbao از منوهای بومی فرمان‌های اسلش پشتیبانی می‌کند؛ هنگام شروع Gateway، فرمان‌ها به‌طور خودکار با پلتفرم همگام می‌شوند.

## عیب‌یابی

**ربات در گفت‌وگوهای گروهی پاسخ نمی‌دهد:**

1. تأیید کنید ربات به گروه اضافه شده است
2. تأیید کنید که ربات را @mention می‌کنید (به‌طور پیش‌فرض الزامی است)
3. گزارش‌ها را بررسی کنید: `openclaw logs --follow`

**ربات پیام‌ها را دریافت نمی‌کند:**

1. تأیید کنید ربات در برنامه Yuanbao ایجاد و تأیید شده است
2. تأیید کنید `appKey` و `appSecret` به‌درستی پیکربندی شده‌اند
3. تأیید کنید Gateway در حال اجرا است: `openclaw gateway status`
4. گزارش‌ها را بررسی کنید: `openclaw logs --follow`

**ربات پاسخ‌های خالی یا جایگزین ارسال می‌کند:**

1. بررسی کنید آیا مدل هوش مصنوعی محتوای معتبری برمی‌گرداند
2. پاسخ جایگزین پیش‌فرض: "فعلاً نمی‌توانم پاسخ دهم؛ می‌توانید سؤال دیگری از من بپرسید"
3. با `channels.yuanbao.fallbackReply` سفارشی‌سازی کنید

**App Secret افشا شده است:**

1. App Secret را در برنامه Yuanbao بازنشانی کنید
2. مقدار را در پیکربندی خود به‌روزرسانی کنید
3. Gateway را دوباره راه‌اندازی کنید: `openclaw gateway restart`

## پیکربندی پیشرفته

### چند حساب

```json5
{
  channels: {
    yuanbao: {
      defaultAccount: "main",
      accounts: {
        main: {
          appKey: "key_xxx",
          appSecret: "secret_xxx",
          name: "ربات اصلی",
        },
        backup: {
          appKey: "key_yyy",
          appSecret: "secret_yyy",
          name: "ربات پشتیبان",
          enabled: false,
        },
      },
    },
  },
}
```

`defaultAccount` کنترل می‌کند وقتی APIهای خروجی یک `accountId` را مشخص نمی‌کنند، از کدام حساب استفاده شود.

### محدودیت‌های پیام

- `maxChars`: حداکثر تعداد نویسه یک پیام (پیش‌فرض `3000`)
- `mediaMaxMb`: محدودیت بارگذاری/دریافت رسانه (پیش‌فرض `20` مگابایت)
- `overflowPolicy`: رفتار هنگام عبور پیام از محدودیت، `"split"` (پیش‌فرض) یا `"stop"`

### پخش جریانی

Yuanbao از خروجی جریانی در سطح بلوک پشتیبانی می‌کند؛ ربات هم‌زمان با تولید متن، آن را به‌صورت قطعه‌ای ارسال می‌کند.

```json5
{
  channels: {
    yuanbao: {
      disableBlockStreaming: false, // پخش جریانی بلوکی فعال است (پیش‌فرض)
    },
  },
}
```

برای ارسال پاسخ کامل در یک پیام، `disableBlockStreaming: true` را تنظیم کنید.

### زمینه تاریخچه گفت‌وگوی گروهی

```json5
{
  channels: {
    yuanbao: {
      historyLimit: 100, // پیش‌فرض: 100، برای غیرفعال‌کردن روی 0 تنظیم کنید
    },
  },
}
```

تعداد پیام‌های تاریخی گنجانده‌شده در زمینه هوش مصنوعی برای گفت‌وگوهای گروهی را کنترل می‌کند.

### حالت پاسخ‌به

```json5
{
  channels: {
    yuanbao: {
      replyToMode: "first", // "off" | "first" | "all" (پیش‌فرض: "first")
    },
  },
}
```

| مقدار   | رفتار                                                 |
| ------- | -------------------------------------------------------- |
| `off`   | بدون پاسخ نقل‌قولی                                           |
| `first` | فقط نخستین پاسخ به هر پیام ورودی را نقل‌قول کنید (پیش‌فرض) |
| `all`   | هر پاسخ را نقل‌قول کنید                                        |

### تزریق راهنمای Markdown

به‌طور پیش‌فرض، ربات یک دستور اعلان سیستمی تزریق می‌کند تا از قراردادن کل پاسخ توسط مدل در یک بلوک کد Markdown جلوگیری شود.

```json5
{
  channels: {
    yuanbao: {
      markdownHintEnabled: true, // پیش‌فرض: true
    },
  },
}
```

### حالت اشکال‌زدایی

```json5
{
  channels: {
    yuanbao: {
      debugBotIds: ["bot_user_id_1", "bot_user_id_2"],
    },
  },
}
```

خروجی گزارش پالایش‌نشده را برای شناسه‌های ربات فهرست‌شده فعال می‌کند.

### مسیریابی چندعاملی

برای مسیریابی پیام‌های مستقیم یا گروه‌های Yuanbao به عامل‌های مختلف، از `bindings` استفاده کنید:

```json5
{
  agents: {
    list: [
      { id: "main" },
      { id: "agent-a", workspace: "/home/user/agent-a" },
      { id: "agent-b", workspace: "/home/user/agent-b" },
    ],
  },
  bindings: [
    {
      agentId: "agent-a",
      match: {
        channel: "yuanbao",
        peer: { kind: "direct", id: "user_xxx" },
      },
    },
    {
      agentId: "agent-b",
      match: {
        channel: "yuanbao",
        peer: { kind: "group", id: "group_zzz" },
      },
    },
  ],
}
```

- `match.channel`: `"yuanbao"`
- `match.peer.kind`: `"direct"` (پیام مستقیم) یا `"group"` (گفت‌وگوی گروهی)
- `match.peer.id`: شناسه کاربر یا کد گروه

## مرجع پیکربندی

پیکربندی کامل: [پیکربندی Gateway](/fa/gateway/configuration)

| تنظیم                                    | توضیح                                       | پیش‌فرض                                |
| ------------------------------------------ | ------------------------------------------------- | -------------------------------------- |
| `channels.yuanbao.enabled`                 | فعال/غیرفعال‌کردن کانال                        | `true`                                 |
| `channels.yuanbao.defaultAccount`          | حساب پیش‌فرض برای مسیریابی خروجی              | `default`                              |
| `channels.yuanbao.accounts.<id>.appKey`    | App Key (امضاکردن + تولید بلیت)             | -                                      |
| `channels.yuanbao.accounts.<id>.appSecret` | App Secret (امضاکردن)                              | -                                      |
| `channels.yuanbao.accounts.<id>.token`     | توکن ازپیش‌امضاشده (از امضای خودکار بلیت صرف‌نظر می‌کند) | -                                      |
| `channels.yuanbao.accounts.<id>.name`      | نام نمایشی حساب                              | -                                      |
| `channels.yuanbao.accounts.<id>.enabled`   | فعال/غیرفعال‌کردن یک حساب مشخص                 | `true`                                 |
| `channels.yuanbao.dm.policy`               | سیاست پیام مستقیم                                         | `open`                                 |
| `channels.yuanbao.dm.allowFrom`            | فهرست مجاز پیام مستقیم (فهرست شناسه کاربران)                       | -                                      |
| `channels.yuanbao.requireMention`          | الزام @mention در گروه‌ها                        | `true`                                 |
| `channels.yuanbao.overflowPolicy`          | مدیریت پیام طولانی (`split` یا `stop`)         | `split`                                |
| `channels.yuanbao.replyToMode`             | راهبرد پاسخ‌به گروه (`off`، `first`، `all`)   | `first`                                |
| `channels.yuanbao.outboundQueueStrategy`   | راهبرد خروجی (`merge-text` یا `immediate`)   | `merge-text`                           |
| `channels.yuanbao.minChars`                | ادغام متن: حداقل نویسه برای آغاز ارسال             | `2800`                                 |
| `channels.yuanbao.maxChars`                | ادغام متن: حداکثر نویسه در هر پیام                 | `3000`                                 |
| `channels.yuanbao.idleMs`                  | ادغام متن: مهلت بی‌کاری پیش از تخلیه خودکار (میلی‌ثانیه)   | `5000`                                 |
| `channels.yuanbao.mediaMaxMb`              | محدودیت اندازه رسانه (مگابایت)                             | `20`                                   |
| `channels.yuanbao.historyLimit`            | ورودی‌های زمینه تاریخچه گفت‌وگوی گروهی                | `100`                                  |
| `channels.yuanbao.disableBlockStreaming`   | غیرفعال‌کردن خروجی جریانی در سطح بلوک              | `false`                                |
| `channels.yuanbao.fallbackReply`           | پاسخ جایگزین وقتی مدل هیچ محتوایی برنمی‌گرداند  | `暂时无法解答，你可以换个问题问问我哦` |
| `channels.yuanbao.markdownHintEnabled`     | تزریق دستورهای جلوگیری از پوشاندن پاسخ با Markdown        | `true`                                 |
| `channels.yuanbao.debugBotIds`             | شناسه‌های ربات فهرست مجاز اشکال‌زدایی (گزارش‌های پالایش‌نشده)        | `[]`                                   |

## انواع پیام پشتیبانی‌شده

**دریافت:** متن، تصویر، فایل، صدا/پیام صوتی، ویدئو، برچسب/ایموجی سفارشی، عناصر سفارشی (کارت‌های پیوند).

**ارسال:** متن (Markdown)، تصویر، فایل، صدا، ویدئو، برچسب.

**رشته‌ها و پاسخ‌ها:** پاسخ‌های نقل‌قولی (قابل پیکربندی از طریق `replyToMode`)؛ پاسخ‌های رشته‌ای توسط پلتفرم پشتیبانی نمی‌شوند.

## مرتبط

- [نمای کلی کانال‌ها](/fa/channels) - همه کانال‌های پشتیبانی‌شده
- [جفت‌سازی](/fa/channels/pairing) - احراز هویت پیام مستقیم و جریان جفت‌سازی
- [گروه‌ها](/fa/channels/groups) - رفتار گفت‌وگوی گروهی و محدودسازی بر اساس اشاره
- [مسیریابی کانال](/fa/channels/channel-routing) - مسیریابی نشست برای پیام‌ها
- [امنیت](/fa/gateway/security) - مدل دسترسی و مقاوم‌سازی
