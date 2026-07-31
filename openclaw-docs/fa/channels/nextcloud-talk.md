---
read_when:
    - کار روی قابلیت‌های کانال Nextcloud Talk
summary: وضعیت پشتیبانی، قابلیت‌ها و پیکربندی Nextcloud Talk
title: Nextcloud Talk
x-i18n:
    generated_at: "2026-07-27T13:44:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 59f4fe51555bcb13d630140866307b1a49ba077059818ec116ee50ef0c877b2b
    source_path: channels/nextcloud-talk.md
    workflow: 16
---

Nextcloud Talk یک Plugin کانال قابل‌دانلود (`@openclaw/nextcloud-talk`) است که OpenClaw را از طریق یک ربات Webhook در Talk به یک نمونه Nextcloud خودمیزبان متصل می‌کند. پیام‌های مستقیم، اتاق‌ها، واکنش‌ها و پیام‌های Markdown پشتیبانی می‌شوند؛ رسانه به‌صورت URL ارسال می‌شود.

## نصب

```bash
openclaw plugins install @openclaw/nextcloud-talk
```

برای دنبال‌کردن برچسب انتشار رسمی فعلی، از مشخصه ساده بسته استفاده کنید. فقط زمانی یک نسخه دقیق را ثابت کنید که به نصب بازتولیدپذیر نیاز دارید.

از یک نسخه محلی کد (گردش‌کارهای توسعه):

```bash
openclaw plugins install ./path/to/local/nextcloud-talk-plugin
```

پس از نصب، Gateway را راه‌اندازی مجدد کنید. جزئیات: [Pluginها](/fa/tools/plugin)

## راه‌اندازی سریع (مبتدی)

1. Plugin را نصب کنید (بالا).
2. در سرور Nextcloud خود، یک ربات ایجاد کنید:

   ```bash
   ./occ talk:bot:install "OpenClaw" "<shared-secret>" "<webhook-url>" --feature webhook --feature response --feature reaction
   ```

   `--feature response` را حفظ کنید: بدون آن، پاسخ‌های خروجی با خطای 401 مواجه می‌شوند. یک ربات موجود را با `./occ talk:bot:state --feature webhook --feature response --feature reaction <botId> 1` تعمیر کنید.

3. ربات را در تنظیمات اتاق هدف فعال کنید.
4. OpenClaw را پیکربندی کنید:
   - پیکربندی: `channels.nextcloud-talk.baseUrl` + `channels.nextcloud-talk.botSecret`
   - یا متغیر محیطی: `NEXTCLOUD_TALK_BOT_SECRET` (فقط حساب پیش‌فرض)

   راه‌اندازی CLI (`--url`/`--token` نام‌های مستعار فیلدهای صریح هستند؛ `nc-talk` و `nc` به‌عنوان نام‌های مستعار کانال کار می‌کنند):

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --url https://cloud.example.com \
     --token "<shared-secret>"
   ```

   فیلدهای صریح معادل:

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --base-url https://cloud.example.com \
     --secret "<shared-secret>"
   ```

   راز مبتنی بر فایل:

   ```bash
   openclaw channels add --channel nextcloud-talk \
     --base-url https://cloud.example.com \
     --secret-file /path/to/nextcloud-talk-secret
   ```

5. Gateway را راه‌اندازی مجدد کنید (یا راه‌اندازی را به پایان برسانید).

پیکربندی حداقلی:

```json5
{
  channels: {
    "nextcloud-talk": {
      enabled: true,
      baseUrl: "https://cloud.example.com",
      botSecret: "shared-secret",
      dmPolicy: "pairing",
    },
  },
}
```

## نکات

- ربات‌ها نمی‌توانند پیام مستقیم را آغاز کنند. کاربر باید ابتدا به ربات پیام دهد.
- URL مربوط به Webhook باید از سرور Nextcloud قابل‌دسترسی باشد؛ هنگامی که Gateway پشت پراکسی قرار دارد، `webhookPublicUrl` را تنظیم کنید. درخواست‌های Webhook با استفاده از راز ربات و HMAC-SHA256 امضا می‌شوند؛ امضاهای نامعتبر رد می‌شوند و مشمول محدودیت نرخ خواهند بود.
- بارگذاری رسانه توسط API ربات پشتیبانی نمی‌شود؛ رسانه خروجی به‌صورت یک خط `Attachment: <url>` افزوده می‌شود.
- بار مفید Webhook میان پیام‌های مستقیم و اتاق‌ها تمایز قائل نمی‌شود؛ برای فعال‌کردن جست‌وجوی نوع اتاق، `apiUser` + `apiPassword` را تنظیم کنید (حدود 5 دقیقه در حافظه نهان نگه داشته می‌شود). بدون آن‌ها، هر مکالمه به‌عنوان اتاق در نظر گرفته می‌شود.
- درخواست‌های خروجی از محافظ SSRF عبور می‌کنند. برای یک میزبان Nextcloud در شبکه خصوصی/داخلی مورداعتماد، با `channels.nextcloud-talk.network.dangerouslyAllowPrivateNetwork: true` آن را فعال کنید.
- با تنظیم `apiUser`/`apiPassword` و `webhookPublicUrl`، فرمان `openclaw channels status` ربات را بررسی می‌کند و هنگام نبود قابلیت `response` هشدار می‌دهد.

## کنترل دسترسی (پیام‌های مستقیم)

- پیش‌فرض: `channels.nextcloud-talk.dmPolicy = "pairing"`. فرستندگان ناشناس یک کد جفت‌سازی دریافت می‌کنند.
- تأیید از طریق:
  - `openclaw pairing list nextcloud-talk`
  - `openclaw pairing approve nextcloud-talk <CODE>`
- پیام‌های مستقیم عمومی: `channels.nextcloud-talk.dmPolicy="open"` به‌همراه `channels.nextcloud-talk.allowFrom=["*"]`.
- `allowFrom` فقط با شناسه‌های کاربری Nextcloud مطابقت دارد (با حروف کوچک)؛ نام‌های نمایشی نادیده گرفته می‌شوند.

## اتاق‌ها (گروه‌ها)

- پیش‌فرض: `channels.nextcloud-talk.groupPolicy = "allowlist"` (مشروط به اشاره).
- اتاق‌ها را با `channels.nextcloud-talk.rooms` و بر اساس توکن اتاق در فهرست مجاز قرار دهید؛ `"*"` یک پیش‌فرض عام را تنظیم می‌کند:

```json5
{
  channels: {
    "nextcloud-talk": {
      rooms: {
        "room-token": { requireMention: true },
      },
    },
  },
}
```

- کلیدهای هر اتاق: `requireMention` (پیش‌فرض true)، `enabled` (false اتاق را غیرفعال می‌کند)، `allowFrom` (فهرست مجاز فرستندگان هر اتاق)، `tools` (بازنویسی مجوز/منع ابزارها)، `skills` (محدودکردن Skills بارگذاری‌شده)، `systemPrompt`.
- برای مجازنبودن هیچ اتاقی، فهرست مجاز را خالی نگه دارید یا `channels.nextcloud-talk.groupPolicy="disabled"` را تنظیم کنید.

## قابلیت‌ها

| قابلیت           | وضعیت             |
| ---------------- | ----------------- |
| پیام‌های مستقیم  | پشتیبانی می‌شود   |
| اتاق‌ها           | پشتیبانی می‌شود   |
| رشته‌ها           | پشتیبانی نمی‌شود  |
| رسانه             | فقط URL           |
| واکنش‌ها          | پشتیبانی می‌شود   |
| فرمان‌های بومی    | پشتیبانی نمی‌شود  |

## مرجع پیکربندی (Nextcloud Talk)

پیکربندی کامل: [پیکربندی](/fa/gateway/configuration)

گزینه‌های ارائه‌دهنده:

- `channels.nextcloud-talk.enabled`: فعال/غیرفعال‌کردن راه‌اندازی کانال.
- `channels.nextcloud-talk.baseUrl`: URL نمونه Nextcloud.
- `channels.nextcloud-talk.botSecret`: راز مشترک ربات (رشته یا ارجاع راز).
- `channels.nextcloud-talk.botSecretFile`: مسیر راز در فایل معمولی. پیوندهای نمادین رد می‌شوند.
- `channels.nextcloud-talk.apiUser`: کاربر API برای جست‌وجوی اتاق‌ها (تشخیص پیام مستقیم) و بررسی وضعیت.
- `channels.nextcloud-talk.apiPassword`: گذرواژه API/برنامه برای جست‌وجوی اتاق‌ها.
- `channels.nextcloud-talk.apiPasswordFile`: مسیر فایل گذرواژه API.
- `channels.nextcloud-talk.webhookPort`: درگاه شنونده Webhook (پیش‌فرض: 8788).
- `channels.nextcloud-talk.webhookHost`: میزبان Webhook (پیش‌فرض: 0.0.0.0).
- `channels.nextcloud-talk.webhookPath`: مسیر Webhook (پیش‌فرض: /nextcloud-talk-webhook).
- `channels.nextcloud-talk.webhookPublicUrl`: URL مربوط به Webhook که از بیرون قابل‌دسترسی است.
- `channels.nextcloud-talk.dmPolicy`: `pairing | allowlist | open | disabled` (پیش‌فرض: pairing). `open` به `allowFrom=["*"]` نیاز دارد.
- `channels.nextcloud-talk.allowFrom`: فهرست مجاز پیام‌های مستقیم (شناسه‌های کاربری).
- `channels.nextcloud-talk.groupPolicy`: `allowlist | open | disabled` (پیش‌فرض: allowlist).
- `channels.nextcloud-talk.groupAllowFrom`: فهرست مجاز فرستندگان اتاق (شناسه‌های کاربری)؛ در صورت تنظیم‌نبودن، از `allowFrom` استفاده می‌کند.
- `channels.nextcloud-talk.rooms`: تنظیمات و فهرست مجاز هر اتاق (بالا را ببینید).
- می‌توان از گروه‌های دسترسی ایستای فرستندگان با `accessGroup:<name>` در `allowFrom` و `groupAllowFrom` ارجاع داد.
- `channels.nextcloud-talk.historyLimit`: محدودیت تاریخچه گروه (0 غیرفعال می‌کند).
- `channels.nextcloud-talk.dmHistoryLimit`: محدودیت تاریخچه پیام مستقیم (0 غیرفعال می‌کند).
- `channels.nextcloud-talk.dms`: بازنویسی‌های هر پیام مستقیم بر اساس شناسه کاربر (`historyLimit`).
- `channels.nextcloud-talk.textChunkLimit`: اندازه قطعه متن خروجی بر حسب نویسه (پیش‌فرض: 4000).
- `channels.nextcloud-talk.streaming.chunkMode`: `length` (پیش‌فرض) یا `newline` برای تقسیم در خطوط خالی (مرز بندها) پیش از قطعه‌بندی بر اساس طول.
- `channels.nextcloud-talk.streaming.block.enabled`: فعال یا غیرفعال‌کردن پخش جریانی بلوکی برای این کانال.
- `channels.nextcloud-talk.streaming.block.coalesce`: تنظیم ادغام پخش جریانی بلوکی.
- `channels.nextcloud-talk.responsePrefix`: پیشوند پاسخ خروجی.
- `channels.nextcloud-talk.markdown.tables`: حالت رندر جدول Markdown (`off | bullets | code | block`).
- `channels.nextcloud-talk.mediaMaxMb`: سقف رسانه ورودی (MB).
- `channels.nextcloud-talk.network.dangerouslyAllowPrivateNetwork`: اجازه عبور میزبان‌های خصوصی/داخلی Nextcloud از محافظ SSRF.
- `channels.nextcloud-talk.accounts.<id>`: بازنویسی‌های هر حساب (همان کلیدها)؛ `defaultAccount` حساب پیش‌فرض را انتخاب می‌کند. متغیرهای محیطی `NEXTCLOUD_TALK_BOT_SECRET` / `NEXTCLOUD_TALK_API_PASSWORD` فقط روی حساب پیش‌فرض اعمال می‌شوند.

## مرتبط

- [نمای کلی کانال‌ها](/fa/channels) — همه کانال‌های پشتیبانی‌شده
- [جفت‌سازی](/fa/channels/pairing) — احراز هویت پیام مستقیم و جریان جفت‌سازی
- [گروه‌ها](/fa/channels/groups) — رفتار گفت‌وگوی گروهی و مشروط‌سازی به اشاره
- [مسیریابی کانال](/fa/channels/channel-routing) — مسیریابی نشست برای پیام‌ها
- [امنیت](/fa/gateway/security) — مدل دسترسی و مقاوم‌سازی
