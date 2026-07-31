---
read_when:
    - پیکربندی فهرست مجاز یکسان در چندین کانال پیام‌رسانی
    - اشتراک‌گذاری قواعد دسترسی فرستنده در پیام‌های مستقیم و گروه‌ها
    - بازبینی کنترل دسترسی کانال پیام‌رسانی
summary: فهرست‌های مجاز فرستندگانِ قابل‌استفادهٔ مجدد برای کانال‌های پیام‌رسانی
title: گروه‌های دسترسی
x-i18n:
    generated_at: "2026-07-27T15:03:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 099abc95e90d9a7b7006d19062c46b4ffdb2aecb1e8e714454a3182131a786d0
    source_path: channels/access-groups.md
    workflow: 16
---

گروه‌های دسترسی، فهرست‌های نام‌گذاری‌شده‌ای از فرستندگان هستند که یک‌بار در `accessGroups` تعریف می‌کنید و با `accessGroup:<name>` از فهرست‌های مجاز کانال به آن‌ها ارجاع می‌دهید.

از آن‌ها زمانی استفاده کنید که افراد یکسانی باید در چند کانال پیام‌رسان مجاز باشند، یا یک مجموعه مورد اعتماد باید هم برای مجوزدهی پیام‌های مستقیم و هم فرستندگان گروه اعمال شود.

یک گروه به‌تنهایی هیچ مجوزی اعطا نمی‌کند. فقط زمانی اثر دارد که یک فیلد فهرست مجاز به آن ارجاع دهد.

## گروه‌های ایستای فرستندگان پیام

گروه‌های ایستای فرستندگان از `type: "message.senders"` استفاده می‌کنند. کلیدهای `members` بر اساس شناسه کانال پیام‌رسان تنظیم می‌شوند و `"*"` برای ورودی‌های مشترک میان همه کانال‌ها است:

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        "*": ["global-owner-id"],
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
        whatsapp: ["+15551234567"],
      },
    },
  },
}
```

| کلید                        | معنا                                                                     |
| -------------------------- | --------------------------------------------------------------------------- |
| `"*"`                      | ورودی‌های مشترکی که برای هر کانال پیام‌رسان ارجاع‌دهنده به گروه بررسی می‌شوند. |
| `discord`، `telegram`، ... | ورودی‌هایی که فقط برای تطبیق فهرست مجاز همان کانال بررسی می‌شوند.                 |

ورودی‌ها با استفاده از قواعد عادی `allowFrom` کانال مقصد تطبیق داده می‌شوند. OpenClaw شناسه‌های فرستندگان را میان کانال‌ها تبدیل نمی‌کند: اگر Alice یک شناسه Telegram و یک شناسه Discord دارد، هر دو شناسه را زیر کلید کانال مربوطه فهرست کنید.

## ارجاع به گروه‌ها از فهرست‌های مجاز

در هر جایی از مسیر کانال پیام‌رسان که از فهرست‌های مجاز فرستندگان پشتیبانی می‌کند، با `accessGroup:<name>` به یک گروه ارجاع دهید.

نمونه فهرست مجاز پیام مستقیم:

```json5
{
  accessGroups: {
    operators: {
      type: "message.senders",
      members: {
        discord: ["discord:123456789012345678"],
        telegram: ["987654321"],
      },
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
    telegram: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators"],
    },
  },
}
```

نمونه فهرست مجاز فرستندگان گروه:

```json5
{
  accessGroups: {
    oncall: {
      type: "message.senders",
      members: {
        whatsapp: ["+15551234567"],
        googlechat: ["users/1234567890"],
      },
    },
  },
  channels: {
    whatsapp: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["accessGroup:oncall"],
    },
    googlechat: {
      groups: {
        "spaces/AAA": {
          users: ["accessGroup:oncall"],
        },
      },
    },
  },
}
```

می‌توانید گروه‌ها و ورودی‌های مستقیم را با هم ترکیب کنید:

```json5
{
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:operators", "discord:123456789012345678"],
    },
  },
}
```

## مسیرهای پشتیبانی‌شده کانال پیام‌رسان

گروه‌های دسترسی در مسیرهای مشترک مجوزدهی کانال‌های پیام‌رسان کار می‌کنند:

- فهرست‌های مجاز فرستندگان پیام مستقیم، مانند `channels.<channel>.allowFrom`
- فهرست‌های مجاز فرستندگان گروه، مانند `channels.<channel>.groupAllowFrom`
- فهرست‌های مجاز مختص هر کانال برای فرستندگان هر اتاق که از همان قواعد تطبیق فرستنده استفاده می‌کنند (برای مثال `groups.<space>.users` در Google Chat)
- مسیرهای مجوزدهی فرمان که از فهرست‌های مجاز فرستندگان کانال پیام‌رسان دوباره استفاده می‌کنند

پشتیبانی هر کانال به این بستگی دارد که آیا آن کانال به کمک‌کننده‌های مشترک مجوزدهی فرستنده در OpenClaw متصل شده است یا خیر. پشتیبانی فعلی همراه محصول شامل ClickClack، Discord، Feishu، Google Chat، iMessage، IRC، LINE، Mattermost، Microsoft Teams، Nextcloud Talk، Nostr، QQ Bot، Signal، Slack، SMS، Telegram، WhatsApp، Zalo و Zalo Personal است. گروه‌های ایستای `message.senders` مستقل از کانال هستند؛ بنابراین کانال‌های پیام‌رسان جدید با استفاده از کمک‌کننده‌های ورودی SDK مشترک Plugin به‌جای گسترش سفارشی فهرست مجاز، آن‌ها را دریافت می‌کنند.

## مخاطبان کانال Discord

Discord همچنین از یک نوع پویای گروه دسترسی پشتیبانی می‌کند:

```json5
{
  accessGroups: {
    maintainers: {
      type: "discord.channelAudience",
      guildId: "1456350064065904867",
      channelId: "1456744319972282449",
      membership: "canViewChannel",
    },
  },
  channels: {
    discord: {
      dmPolicy: "allowlist",
      allowFrom: ["accessGroup:maintainers"],
    },
  },
}
```

`discord.channelAudience` یعنی «فرستندگان پیام مستقیم Discord را که در حال حاضر می‌توانند این کانال guild را مشاهده کنند، مجاز بدان». OpenClaw هنگام مجوزدهی، فرستنده را از طریق Discord شناسایی می‌کند و قواعد مجوز `ViewChannel` در Discord را اعمال می‌کند. `membership` اختیاری است و مقدار پیش‌فرض آن `canViewChannel` است.

از این قابلیت زمانی استفاده کنید که یک کانال Discord از قبل منبع حقیقت یک تیم است، مانند `#maintainers` یا `#on-call`.

الزامات و رفتار هنگام شکست:

- ربات باید به guild و کانال دسترسی داشته باشد.
- ربات به **Server Members Intent** در Discord Developer Portal نیاز دارد.
- اگر Discord مقدار `Missing Access` را بازگرداند، فرستنده به‌عنوان عضو guild قابل شناسایی نباشد، یا کانال متعلق به guild دیگری باشد، گروه دسترسی به‌صورت بسته شکست می‌خورد.

نمونه‌های بیشتر مختص Discord: [کنترل دسترسی Discord](/fa/channels/discord#access-control-and-routing)

## عیب‌یابی Plugin

نویسندگان Plugin می‌توانند وضعیت ساخت‌یافته گروه دسترسی را بدون گسترش دوباره آن به یک فهرست مجاز تخت بررسی کنند:

```typescript
import { resolveAccessGroupAllowFromState } from "openclaw/plugin-sdk/access-groups";

const state = await resolveAccessGroupAllowFromState({
  accessGroups: cfg.accessGroups,
  allowFrom: channelConfig.allowFrom,
  channel: "my-channel",
  accountId: "default",
  senderId,
  isSenderAllowed,
});
```

نتیجه، گروه‌های ارجاع‌شده، تطبیق‌یافته، مفقود، پشتیبانی‌نشده و ناموفق را گزارش می‌کند. از آن برای عیب‌یابی یا آزمون‌های انطباق استفاده کنید. فقط برای مسیرهای سازگاری که همچنان انتظار یک آرایه تخت `allowFrom` را دارند، از `expandAllowFromWithAccessGroups(...)` استفاده کنید.

## نکات امنیتی

- گروه‌های دسترسی نام‌های مستعار فهرست مجاز هستند، نه نقش‌ها. آن‌ها به‌تنهایی مالک ایجاد نمی‌کنند، درخواست‌های جفت‌سازی را تأیید نمی‌کنند و مجوز ابزار اعطا نمی‌کنند.
- `dmPolicy: "open"` همچنان در فهرست مجاز مؤثر پیام مستقیم به `"*"` نیاز دارد. ارجاع به یک گروه دسترسی با دسترسی عمومی یکسان نیست.
- نام‌های مفقود گروه به‌صورت بسته شکست می‌خورند. اگر `allowFrom` شامل `accessGroup:operators` باشد و `accessGroups.operators` وجود نداشته باشد، آن ورودی به هیچ‌کس مجوز نمی‌دهد.
- شناسه‌های کانال را پایدار نگه دارید. وقتی کانال از هر دو پشتیبانی می‌کند، شناسه‌های عددی/کاربر را به نام‌های نمایشی ترجیح دهید.

## رفع اشکال

اگر یک فرستنده باید تطبیق داده شود اما مسدود است:

1. تأیید کنید که فیلد فهرست مجاز شامل ارجاع دقیق `accessGroup:<name>` است.
2. تأیید کنید که `accessGroups.<name>.type` درست است.
3. تأیید کنید که شناسه فرستنده زیر کلید کانال مربوطه یا زیر `"*"` فهرست شده است.
4. تأیید کنید که ورودی از نحو عادی فهرست مجاز همان کانال استفاده می‌کند.
5. برای مخاطبان کانال Discord، تأیید کنید که ربات می‌تواند کانال guild را ببیند و Server Members Intent فعال است.

پس از ویرایش پیکربندی کنترل دسترسی، `openclaw doctor` را اجرا کنید. این فرمان بسیاری از ترکیب‌های نامعتبر فهرست مجاز و خط‌مشی را پیش از زمان اجرا شناسایی می‌کند.
