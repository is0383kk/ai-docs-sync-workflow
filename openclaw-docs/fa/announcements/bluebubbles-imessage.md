---
read_when:
    - شما از کانال قدیمی BlueBubbles استفاده می‌کردید و باید به iMessage مهاجرت کنید
    - شما در حال انتخاب راه‌اندازی پشتیبانی‌شدهٔ iMessage در OpenClaw هستید
    - به توضیح کوتاهی درباره حذف BlueBubbles نیاز دارید
summary: پشتیبانی از BlueBubbles از OpenClaw حذف شده است. برای راه‌اندازی‌های جدید و مهاجرت‌یافتهٔ iMessage، از Plugin همراه iMessage با imsg استفاده کنید.
title: حذف BlueBubbles و مسیر imsg برای iMessage
x-i18n:
    generated_at: "2026-07-27T14:50:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7dec7d3f27e0df6431494d864b0c7ae7457574797e199f9a2cb6931d28feacd0
    source_path: announcements/bluebubbles-imessage.md
    workflow: 16
---

# حذف BlueBubbles و مسیر iMessage مبتنی بر imsg

OpenClaw دیگر کانال BlueBubbles را ارائه نمی‌کند. پشتیبانی از iMessage از طریق Plugin همراه `imessage` انجام می‌شود: Gateway، [`imsg`](https://github.com/steipete/imsg) را به‌صورت یک فرایند فرزند، به‌شکل محلی یا از طریق یک پوشش SSH، اجرا می‌کند و با استفاده از JSON-RPC روی stdin/stdout با آن ارتباط برقرار می‌کند. بدون سرور، بدون Webhook و بدون پورت.

اگر پیکربندی شما همچنان حاوی `channels.bluebubbles` است، آن را به `channels.imessage` مهاجرت دهید. نشانی قدیمی مستندات `/channels/bluebubbles` به [مهاجرت از BlueBubbles](/fa/channels/imessage-from-bluebubbles) هدایت می‌شود که جدول کامل تبدیل پیکربندی و چک‌لیست تغییر مسیر را در بر دارد.

## چه چیزهایی تغییر کرده است

- مسیر پشتیبانی‌شده iMessage هیچ سرور HTTP، مسیر Webhook، گذرواژه REST یا زمان اجرای Plugin متعلق به BlueBubbles ندارد.
- OpenClaw پیام‌ها را از طریق `imsg` در Macی که Messages.app در آن وارد حساب شده است، می‌خواند و پایش می‌کند.
- ارسال، دریافت، تاریخچه و رسانه‌های پایه از سطوح معمول `imsg` و مجوزهای macOS استفاده می‌کنند.
- کنش‌های پیشرفته (پاسخ‌های رشته‌ای، واکنش‌های tapback، ویرایش، لغو ارسال، جلوه‌ها، رسیدهای خواندن، نشانگرهای تایپ و مدیریت گروه) به پل API خصوصی نیاز دارند: `imsg launch` را اجرا کنید که مستلزم غیرفعال‌بودن SIP است.
- Gatewayهای Linux و Windows همچنان می‌توانند با تنظیم `channels.imessage.cliPath` روی یک پوشش SSH که `imsg` را در Mac واردشده به حساب اجرا می‌کند، از iMessage استفاده کنند.

## چه باید کرد

1. `imsg` را روی Mac میزبان Messages نصب و بررسی کنید:

   ```bash
   brew install steipete/tap/imsg
   imsg --version
   imsg chats --limit 3
   imsg rpc --help
   ```

2. مجوزهای Full Disk Access و Automation را به بستر فرایندی که `imsg` و OpenClaw را اجرا می‌کند، اعطا کنید.

3. پیکربندی قدیمی را تبدیل کنید:

   ```json5
   {
     channels: {
       imessage: {
         enabled: true,
         cliPath: "/opt/homebrew/bin/imsg",
         dmPolicy: "pairing",
         allowFrom: ["+15555550123"],
         groupPolicy: "allowlist",
         groupAllowFrom: ["+15555550123"],
         groups: {
           "*": { requireMention: true },
         },
         includeAttachments: true,
       },
     },
   }
   ```

4. Gateway را دوباره راه‌اندازی و بررسی کنید:

   ```bash
   openclaw channels status --probe
   ```

5. پیش از حذف سرور قدیمی BlueBubbles، پیام‌های مستقیم، گروه‌ها، پیوست‌ها و هر کنش API خصوصی موردنیاز خود را آزمایش کنید.

## نکات مهاجرت

- `channels.bluebubbles.serverUrl` و `channels.bluebubbles.password` هیچ معادلی در iMessage ندارند؛ هیچ سروری برای دسترسی یا احراز هویت وجود ندارد.
- `allowFrom`، `groupAllowFrom`، `groups`، `includeAttachments`، `attachmentRoots`، `mediaMaxMb`، `textChunkLimit` و `actions.*` در `channels.imessage` معنای خود را حفظ می‌کنند.
- `channels.imessage.includeAttachments` همچنان به‌طور پیش‌فرض غیرفعال است. اگر انتظار دارید عکس‌ها، یادداشت‌های صوتی، ویدئوها یا فایل‌های ورودی به عامل برسند، آن را صریحاً تنظیم کنید.
- هنگام استفاده از `groupPolicy: "allowlist"`، بلوک قدیمی `groups` را همراه با هر ورودی نویسه عام `"*"` کپی کنید. فهرست‌های مجاز فرستندگان گروه و رجیستری گروه دروازه‌هایی جداگانه هستند؛ یک بلوک `groups` که دارای ورودی است اما `chat_id` منطبق ندارد (یا `"*"` ندارد)، پیام را هنگام اجرا کنار می‌گذارد و یک بلوک خالی `groups` هنگام راه‌اندازی هشدار ثبت می‌کند، هرچند پالایش فرستنده همچنان به پیام‌ها اجازه عبور می‌دهد.
- اتصال‌های ACP دارای `match.channel: "bluebubbles"` باید به `"imessage"` تغییر کنند.
- کلیدهای نشست قدیمی BlueBubbles به کلیدهای نشست iMessage تبدیل نمی‌شوند. تأییدهای جفت‌سازی بر اساس شناسه‌های فرستنده کلیدگذاری می‌شوند؛ بنابراین ورودی‌های کپی‌شده `allowFrom` همچنان کار می‌کنند، اما تاریخچه مکالمه تحت کلیدهای نشست BlueBubbles منتقل نمی‌شود.

## همچنین ببینید

- [مهاجرت از BlueBubbles](/fa/channels/imessage-from-bluebubbles)
- [iMessage](/fa/channels/imessage)
- [مرجع پیکربندی - iMessage](/fa/gateway/config-channels#imessage)
