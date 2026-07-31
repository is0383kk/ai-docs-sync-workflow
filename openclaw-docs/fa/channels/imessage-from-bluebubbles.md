---
read_when:
    - برنامه‌ریزی برای مهاجرت از BlueBubbles به Plugin همراه iMessage
    - ترجمه کلیدهای پیکربندی BlueBubbles به معادل‌های iMessage
    - تأیید imsg پیش از فعال‌کردن Plugin ‏iMessage
summary: 'پیکربندی‌های قدیمی BlueBubbles را به Plugin همراه iMessage منتقل کنید: نگاشت کلیدها، دروازه‌های فهرست مجاز گروه‌ها و راستی‌آزمایی انتقال.'
title: مهاجرت از BlueBubbles
x-i18n:
    generated_at: "2026-07-27T14:51:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5984ad1319b4bb3060496666bea6de663eba0105a89f82d13030c015c5df159d
    source_path: channels/imessage-from-bluebubbles.md
    workflow: 16
---

پشتیبانی از BlueBubbles حذف شد. OpenClaw فقط از طریق Plugin همراه `imessage` از iMessage پشتیبانی می‌کند؛ این Plugin، [`steipete/imsg`](https://github.com/steipete/imsg) را روی JSON-RPC راه‌اندازی می‌کند و به همان سطح API خصوصی دسترسی دارد که BlueBubbles داشت (`react`، `edit`، `unsend`، `reply`، `sendWithEffect`، نظرسنجی‌های بومی، مدیریت گروه و پیوست‌ها). یک فایل اجرایی CLI جایگزین سرور BlueBubbles، برنامهٔ کلاینت و سازوکار Webhook می‌شود: بدون نقطهٔ پایانی REST و بدون احراز هویت Webhook.

این راهنما پیکربندی‌های قدیمی `channels.bluebubbles` را به `channels.imessage` مهاجرت می‌دهد. هیچ مسیر مهاجرت پشتیبانی‌شدهٔ دیگری وجود ندارد. در نسخهٔ فعلی OpenClaw، بلوک باقی‌ماندهٔ `channels.bluebubbles` غیرفعال است — هیچ محیط اجرایی آن را نمی‌خواند.

<Note>
برای اطلاعیهٔ کوتاه و خلاصهٔ ویژهٔ اپراتورها، به [حذف BlueBubbles و مسیر imsg برای iMessage](/fa/announcements/bluebubbles-imessage) مراجعه کنید.
</Note>

## چک‌لیست مهاجرت

اگر از قبل پیکربندی قدیمی BlueBubbles خود را می‌شناسید، کوتاه‌ترین مسیر امن این است:

1. خود `imsg` را مستقیماً روی Mac اجراکنندهٔ Messages.app بررسی کنید (`imsg chats`، `imsg history`، `imsg send`، `imsg rpc --help`).
2. کلیدهای رفتاری را از `channels.bluebubbles` به `channels.imessage` کپی کنید: `dmPolicy`، `allowFrom`، `groupPolicy`، `groupAllowFrom`، `groups`، `includeAttachments`، `attachmentRoots`، `mediaMaxMb`، `textChunkLimit` و `actions`.
3. کلیدهای انتقالی را که دیگر وجود ندارند حذف کنید: `serverUrl`، `password`، نشانی‌های Webhook و راه‌اندازی سرور BlueBubbles.
4. اگر Gateway روی Mac میزبان Messages اجرا نمی‌شود، `channels.imessage.cliPath` را روی یک لفاف SSH تنظیم کنید و برای واکشی راه‌دور پیوست‌ها، `remoteHost` را تنظیم کنید.
5. `channels.imessage` را فعال و Gateway را دوباره راه‌اندازی کنید، سپس `openclaw channels status --probe --channel imessage` را اجرا کنید.
6. یک پیام مستقیم، یک گروه مجاز، در صورت فعال‌بودن پیوست‌ها، و تمام کنش‌های API خصوصی مورد انتظار برای استفادهٔ عامل را آزمایش کنید.
7. پس از تأیید مسیر iMessage، سرور BlueBubbles و پیکربندی قدیمی `channels.bluebubbles` را حذف کنید.

## کارکرد imsg

`imsg` یک CLI محلی macOS برای Messages است. OpenClaw، ‏`imsg rpc` را به‌عنوان فرایند فرزند آغاز می‌کند و از طریق ورودی/خروجی استاندارد با JSON-RPC ارتباط برقرار می‌کند. هیچ سرور HTTP، نشانی Webhook، دیمن پس‌زمینه، عامل راه‌انداز یا درگاهی برای در معرض دسترس قراردادن وجود ندارد.

- خواندن از `~/Library/Messages/chat.db` با استفاده از یک دستگیرهٔ فقط‌خواندنی SQLite انجام می‌شود.
- پیام‌های ورودی زنده از `imsg watch` / `watch.subscribe` می‌آیند که رویدادهای سیستم فایل `chat.db` را با روش جایگزین نظرسنجی دنبال می‌کند.
- ارسال متن و فایل معمولی از خودکارسازی Messages.app استفاده می‌کند.
- کنش‌های پیشرفته از `imsg launch` برای تزریق ابزار کمکی `imsg` به Messages.app استفاده می‌کنند. این همان چیزی است که رسید خواندن، نشانگرهای تایپ، ارسال‌های غنی، ویرایش، لغو ارسال، پاسخ رشته‌ای، واکنش‌ها، نظرسنجی‌ها و مدیریت گروه را فعال می‌کند.
- ساخت‌های Linux می‌توانند یک نسخهٔ کپی‌شدهٔ `chat.db` را بررسی کنند، اما نمی‌توانند ارسال کنند، پایگاه دادهٔ زندهٔ Mac را زیر نظر بگیرند یا Messages.app را کنترل کنند. برای iMessage در OpenClaw، ‏`imsg` را روی Mac واردشده به حساب یا از طریق یک لفاف SSH متصل به آن Mac اجرا کنید.

## پیش از شروع

1. `imsg` را روی Mac اجراکنندهٔ Messages.app نصب کنید:

   ```bash
   brew install steipete/tap/imsg
   brew update && brew upgrade imsg
   imsg --version
   imsg chats --limit 3
   ```

   در راه‌اندازی معمول محلی، فرایند راه‌اندازی OpenClaw می‌تواند نصب یا به‌روزرسانی تأییدشده توسط کاربر با Homebrew را برای `imsg` روی Mac واردشده به حساب Messages پیشنهاد دهد. راه‌اندازی دستی و توپولوژی‌های لفاف SSH همچنان تحت مدیریت اپراتور هستند: به‌روزرسانی Homebrew را در همان زمینهٔ کاربری محلی یا راه‌دوری تکرار کنید که `imsg` را اجرا خواهد کرد. اگر `imsg chats` با `unable to open database file`، خروجی خالی یا `authorization denied` ناموفق شد، دسترسی کامل دیسک را به ترمینال، ویرایشگر، فرایند Node، سرویس Gateway یا فرایند والد SSH که `imsg` را اجرا می‌کند اعطا کنید، سپس آن فرایند والد را دوباره باز کنید.

2. پیش از تغییر پیکربندی OpenClaw، سطوح خواندن، نظارت، ارسال و RPC را بررسی کنید:

   ```bash
   imsg chats --limit 10 --json | jq -s
   imsg history --chat-id 42 --limit 10 --attachments --json | jq -s
   imsg watch --chat-id 42 --reactions --json
   imsg send --chat-id 42 --text "OpenClaw imsg test"
   imsg rpc --help
   ```

   `42` را با یک شناسهٔ واقعی گفت‌وگو از `imsg chats` جایگزین کنید. ارسال به مجوز Automation برای Messages.app نیاز دارد. اگر OpenClaw از طریق SSH اجرا خواهد شد، این فرمان‌ها را با همان لفاف SSH یا زمینهٔ کاربری اجرا کنید که OpenClaw استفاده خواهد کرد. اگر خواندن کار می‌کند اما ارسال با AppleEvents ‏`-1743` ناموفق می‌شود، بررسی کنید که آیا Automation به `/usr/libexec/sshd-keygen-wrapper` اختصاص یافته است؛ به [شکست ارسال از طریق لفاف SSH با AppleEvents -1743](/fa/channels/imessage#requirements-and-permissions-macos) مراجعه کنید.

3. پل API خصوصی را فعال کنید. این کار برای iMessage در OpenClaw اکیداً توصیه می‌شود، زیرا پاسخ‌ها، واکنش‌ها، جلوه‌ها، نظرسنجی‌ها، پاسخ به پیوست‌ها و کنش‌های گروهی به آن وابسته‌اند:

   ```bash
   imsg launch
   imsg status --json
   ```

   `imsg launch` مستلزم غیرفعال‌بودن SIP است (و در نسخه‌های جدید macOS، اعتبارسنجی کتابخانه نیز باید تسهیل شود — به [فعال‌کردن API خصوصی imsg](/fa/channels/imessage#enabling-the-imsg-private-api) مراجعه کنید). ارسال پایه، تاریخچه و نظارت بدون `imsg launch` کار می‌کنند؛ اما سطح کامل کنش‌های iMessage در OpenClaw کار نمی‌کند.

4. پس از فعال‌کردن `channels.imessage` و راه‌اندازی Gateway، پل را از طریق OpenClaw بررسی کنید:

   ```bash
   openclaw channels status --probe
   ```

   حساب iMessage باید `works` را گزارش کند؛ با `--json`، محمولهٔ بررسی شامل `privateApi.available: true` است. اگر `false` را گزارش کرد، ابتدا آن را برطرف کنید — به [تشخیص قابلیت](/fa/channels/imessage#private-api-actions) مراجعه کنید. بررسی به Gateway قابل‌دسترسی نیاز دارد (در غیر این صورت CLI به خروجی صرفاً مبتنی بر پیکربندی بازمی‌گردد) و فقط حساب‌های پیکربندی‌شده و فعال را بررسی می‌کند.

5. از پیکربندی خود نسخهٔ لحظه‌ای تهیه کنید:

   ```bash
   cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
   ```

## تبدیل پیکربندی

iMessage و BlueBubbles بیشتر کلیدهای رفتاری سطح کانال را به‌اشتراک می‌گذارند. آنچه تغییر می‌کند، روش انتقال (سرور REST در برابر CLI محلی) و قالب کلید دفتر ثبت گروه است.

| BlueBubbles                                                | iMessage همراه                          | یادداشت‌ها                                                                                                                                                                                                                                                                            |
| ---------------------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `channels.bluebubbles.enabled`                             | `channels.imessage.enabled`               | معنای یکسان (پس از وجود بلوک، مقدار پیش‌فرض `true` است).                                                                                                                                                                                                                           |
| `channels.bluebubbles.serverUrl`                           | _(حذف‌شده)_                               | سرور REST وجود ندارد — Plugin، `imsg rpc` را از طریق stdio اجرا می‌کند.                                                                                                                                                                                                                        |
| `channels.bluebubbles.password`                            | _(حذف‌شده)_                               | نیازی به احراز هویت Webhook نیست.                                                                                                                                                                                                                                                |
| _(ضمنی)_                                               | `channels.imessage.cliPath`               | مسیر `imsg` (پیش‌فرض `imsg`)؛ برای SSH از یک اسکریپت پوشاننده استفاده کنید.                                                                                                                                                                                                                   |
| _(ضمنی)_                                               | `channels.imessage.dbPath`                | بازنویسی اختیاری `chat.db` برای Messages.app؛ در صورت حذف، به‌طور خودکار شناسایی می‌شود.                                                                                                                                                                                                            |
| _(ضمنی)_                                               | `channels.imessage.remoteHost`            | `host` یا `user@host` — تنها زمانی لازم است که `cliPath` یک پوشاننده SSH باشد و بخواهید پیوست‌ها از طریق SCP واکشی شوند.                                                                                                                                                                        |
| `channels.bluebubbles.dmPolicy`                            | `channels.imessage.dmPolicy`              | مقادیر یکسان (`pairing` / `allowlist` / `open` / `disabled`)؛ پیش‌فرض `pairing`.                                                                                                                                                                                                  |
| `channels.bluebubbles.allowFrom`                           | `channels.imessage.allowFrom`             | قالب‌های شناسه یکسان (`+15555550123`، `user@example.com`). تأییدهای ذخیره‌گاه جفت‌سازی منتقل نمی‌شوند — ادامه را ببینید.                                                                                                                                                                   |
| `channels.bluebubbles.groupPolicy`                         | `channels.imessage.groupPolicy`           | مقادیر یکسان (`allowlist` / `open` / `disabled`)؛ پیش‌فرض `allowlist`.                                                                                                                                                                                                            |
| `channels.bluebubbles.groupAllowFrom`                      | `channels.imessage.groupAllowFrom`        | یکسان است. وقتی تنظیم نشده باشد، iMessage به `allowFrom` برمی‌گردد؛ `groupAllowFrom: []` که صراحتاً خالی باشد، همه گروه‌ها را تحت `groupPolicy: "allowlist"` مسدود می‌کند.                                                                                                                               |
| `channels.bluebubbles.groups`                              | `channels.imessage.groups`                | ورودی عام `"*"` را عیناً کپی کنید؛ کلید ورودی‌های هر گروه را بر اساس `chat_id` عددی iMessage تغییر دهید — «دام رجیستری گروه» را ببینید. `requireMention`، `tools`، `toolsBySender`، `systemPrompt` منتقل می‌شوند.                                                                            |
| `channels.bluebubbles.sendReadReceipts`                    | `channels.imessage.sendReadReceipts`      | پیش‌فرض `true`. در Plugin همراه، این مورد فقط زمانی فعال می‌شود که کاوش API خصوصی برقرار باشد.                                                                                                                                                                                        |
| `channels.bluebubbles.includeAttachments`                  | `channels.imessage.includeAttachments`    | ساختار یکسان و همچنان به‌طور پیش‌فرض غیرفعال است. اگر پیوست‌ها در BlueBubbles منتقل می‌شدند، این مورد را صراحتاً تنظیم کنید — تا آن زمان، عکس‌ها/رسانه‌های ورودی بی‌صدا کنار گذاشته می‌شوند (بدون خط گزارش `Inbound message`).                                                                                             |
| `channels.bluebubbles.attachmentRoots`                     | `channels.imessage.attachmentRoots`       | ریشه‌های محلی؛ قواعد عام یکسان.                                                                                                                                                                                                                                                |
| _(نامربوط)_                                                    | `channels.imessage.remoteAttachmentRoots` | فقط وقتی استفاده می‌شود که `remoteHost` برای واکشی‌های SCP تنظیم شده باشد.                                                                                                                                                                                                                              |
| `channels.bluebubbles.mediaMaxMb`                          | `channels.imessage.mediaMaxMb`            | پیش‌فرض در iMessage برابر 16 MB است (پیش‌فرض BlueBubbles برابر 8 MB بود). برای حفظ سقف پایین‌تر، آن را صراحتاً تنظیم کنید.                                                                                                                                                                                  |
| `channels.bluebubbles.textChunkLimit`                      | `channels.imessage.textChunkLimit`        | پیش‌فرض در هر دو 4000 است.                                                                                                                                                                                                                                                            |
| `channels.bluebubbles.coalesceSameSenderDms`               | _(حذف‌شده)_                               | این کلید را مهاجرت ندهید. `imsg` نسخه 0.13.1 و جدیدتر، ارسال‌های تقسیم‌شده پیش‌نمایش URL اپل را پیش از دریافت آن‌ها توسط OpenClaw ادغام می‌کند؛ `openclaw doctor --fix` یک کلید منسوخ iMessage را حذف می‌کند.                                                                                                    |
| `channels.bluebubbles.enrichGroupParticipantsFromContacts` | _(نامربوط)_                                   | `imsg` از قبل نام نمایشی فرستنده را از `chat.db` ارائه می‌کند.                                                                                                                                                                                                                     |
| `channels.bluebubbles.actions.*`                           | `channels.imessage.actions.*`             | کلیدهای تغییر وضعیت یکسان برای هر کنش (`reactions`، `edit`، `unsend`، `reply`، `sendWithEffect`، `renameGroup`، `setGroupIcon`، `addParticipant`، `removeParticipant`، `leaveGroup`، `sendAttachment`) به‌علاوه `polls` جدید. همه به‌طور پیش‌فرض فعال‌اند؛ کنش‌های API خصوصی همچنان به پل نیاز دارند. |

پیکربندی‌های چندحسابی (`channels.bluebubbles.accounts.*`) به‌صورت یک‌به‌یک به `channels.imessage.accounts.*` تبدیل می‌شوند.

## دام رجیستری گروه

Plugin همراه iMessage دو دروازه گروه را پشت‌سرهم اجرا می‌کند. پیام گروهی برای رسیدن به عامل باید از هر دو عبور کند:

1. **فهرست مجاز فرستنده / مقصد گپ** (`channels.imessage.groupAllowFrom`) — با شناسه فرستنده یا مقصد گپ (ورودی‌های `chat_id:`، `chat_guid:`، `chat_identifier:`) مطابقت داده می‌شود. وقتی `groupAllowFrom` تنظیم نشده باشد، این دروازه به `allowFrom` برمی‌گردد؛ `groupAllowFrom: []` صریح، این بازگشت را غیرفعال می‌کند و همه پیام‌های گروهی را تحت `groupPolicy: "allowlist"` کنار می‌گذارد.
2. **رجیستری گروه** (`channels.imessage.groups`) — با `chat_id` عددی iMessage کلیدگذاری می‌شود:
   - بدون بلوک `groups` (یا با بلوک خالی): گروه‌ها تا زمانی از این دروازه عبور می‌کنند که دروازه 1 یک فهرست مجاز مؤثر و غیرخالی برای فرستندگان داشته باشد؛ پالایش فرستنده دسترسی را کنترل می‌کند و هیچ هشدار آغازبه‌کار برای کنارگذاری همه صادر نمی‌شود.
   - `groups` دارای ورودی اما بدون `"*"`: فقط کلیدهای `chat_id` فهرست‌شده عبور می‌کنند. فهرست‌کردن هر گروه، حتی تحت `groupPolicy: "open"`، رجیستری را به فهرست مجاز تبدیل می‌کند.
   - `groups: { "*": { ... } }`: همه گروه‌ها از این دروازه عبور می‌کنند.

دام مهاجرت: BlueBubbles ورودی‌های `groups` را با GUID گپ / شناسه گپ کلیدگذاری می‌کرد، اما رجیستری iMessage با `chat_id` عددی کلیدگذاری می‌شود. کپی عینی ورودی‌های هر گروه، رجیستری غیرخالی‌ای می‌سازد که کلیدهایش هرگز مطابقت ندارند؛ در نتیجه همه پیام‌های گروهی در دروازه 2 کنار گذاشته می‌شوند. ورودی عام `"*"` را عیناً کپی کنید؛ کلید ورودی‌های گروه مشخص را با مقادیر `chat_id` از `imsg chats` تغییر دهید.

هر دو مسیر کنارگذاری در سطح پیش‌فرض گزارش‌گیری، از طریق خطوط `warn` قابل مشاهده‌اند:

- یک‌بار برای هر حساب هنگام آغازبه‌کار، وقتی `groupPolicy: "allowlist"` تنظیم شده و فهرست مجاز مؤثر فرستندگان گروه خالی است: `imessage: groupPolicy="allowlist" for account "<id>" but no group sender allowlist is configured ...`. برای پذیرش فرستندگان، `groupAllowFrom` (یا `allowFrom`) را تنظیم کنید؛ افزودن `groups` به‌تنهایی دروازه فرستنده را برآورده نمی‌کند.
- یک‌بار برای هر `chat_id` هنگام اجرا، وقتی رجیستری گروهی را کنار می‌گذارد: `imessage: dropping group message from chat_id=<id> ... not in channels.imessage.groups allowlist` که کلید دقیق موردنیاز برای افزودن را نام می‌برد.

پیام‌های مستقیم در هر دو حالت همچنان کار می‌کنند — آن‌ها مسیر کد متفاوتی دارند، بنابراین موفقیت پیام مستقیم، مسیریابی گروهی را اثبات نمی‌کند.

حداقل پیکربندی محدودشده به فرستنده با `groupPolicy: "allowlist"`:

```json5
{
  channels: {
    imessage: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15555550123", "chat_guid:any;-;..."],
    },
  },
}
```

این پیکربندی فرستندگان تنظیم‌شده را در هر گروهی می‌پذیرد. برای محدودکردن گپ‌های مجاز یا تنظیم گزینه‌های هر گپ مانند `requireMention`، ورودی‌های `groups` را اضافه کنید؛ ورودی `"*"` مربوط به BlueBubbles را عیناً کپی کنید، اما کلید ورودی‌های مشخص را با مقادیر عددی `chat_id` در iMessage تغییر دهید.

## گام‌به‌گام

1. پیکربندی را تبدیل کنید. هنگام ویرایش، بلوک جدید را غیرفعال نگه دارید؛ بلوک قدیمی `channels.bluebubbles` در OpenClaw کنونی نادیده گرفته می‌شود و می‌تواند به‌عنوان مرجع در کنار آن باقی بماند:

   ```json5
   {
     channels: {
       imessage: {
         enabled: false, // وقتی برای انتقال آماده شد، به true تغییر دهید
         cliPath: "/opt/homebrew/bin/imsg",
         dmPolicy: "pairing",
         allowFrom: ["+15555550123"], // از bluebubbles.allowFrom کپی کنید
         groupPolicy: "allowlist",
         groupAllowFrom: [], // از bluebubbles.groupAllowFrom کپی کنید
         groups: { "*": { requireMention: true } }, // مقدار عام عیناً کپی می‌شود؛ کلید ورودی‌های هر گپ را بر اساس chat_id تغییر دهید
         // کنش‌ها به‌طور پیش‌فرض فعال‌اند؛ برای غیرفعال‌کردن، کلیدهای تغییر وضعیت هر مورد را روی false تنظیم کنید
       },
     },
   }
   ```

2. **انتقال و کاوش را انجام دهید.** `channels.imessage.enabled: true` را تنظیم کنید، Gateway را دوباره راه‌اندازی کنید و تأیید کنید که کانال وضعیت سالم گزارش می‌کند:

   ```bash
   openclaw gateway restart
   openclaw channels status --probe --channel imessage   # انتظار می‌رود "works"؛ --json مقدار privateApi.available: true را نشان می‌دهد
   ```

   کاوش به یک Gateway دردسترس نیاز دارد و فقط حساب‌های پیکربندی‌شده و فعال را بررسی می‌کند. برای اعتبارسنجی خود Mac، از فرمان‌های مستقیم `imsg` در [پیش از شروع](#before-you-start) استفاده کنید.

3. **پیام‌های مستقیم را تأیید کنید.** یک پیام مستقیم برای عامل بفرستید؛ تأیید کنید که پاسخ دریافت می‌شود.

4. **گروه‌ها را جداگانه تأیید کنید.** پیام‌های مستقیم و گروه‌ها از مسیرهای کد متفاوتی عبور می‌کنند — موفقیت پیام مستقیم، مسیریابی گروه‌ها را اثبات نمی‌کند. در یک گفت‌وگوی گروهی مجاز پیامی بفرستید و تأیید کنید که پاسخ دریافت می‌شود. اگر گروه بی‌پاسخ ماند (نه پاسخی از عامل و نه خطایی)، گزارش Gateway را برای دو خط `warn` از بخش «دام رجیستری گروه» در بالا بررسی کنید. هشدار هنگام راه‌اندازی یعنی فهرست مجاز فرستندگان مؤثر خالی است؛ هشدار مختص هر `chat_id` یعنی رجیستری پرشدهٔ `groups` آن گفت‌وگو را دربر نمی‌گیرد.

5. **سطح کنش‌ها را تأیید کنید.** از یک پیام مستقیم جفت‌شده، از عامل بخواهید واکنش نشان دهد، ویرایش کند، ارسال را لغو کند، پاسخ دهد، عکس بفرستد و (در یک گروه) نام گروه را تغییر دهد یا شرکت‌کننده‌ای را اضافه/حذف کند. هر کنش باید به‌صورت بومی در Messages.app انجام شود. اگر کنشی خطای `iMessage <action> requires the imsg private API bridge` ایجاد کرد، دوباره `imsg launch` را اجرا کنید و با `openclaw channels status --probe` تازه‌سازی کنید.

6. **پس از تأیید پیام‌های مستقیم، گروه‌ها و کنش‌های iMessage، سرور BlueBubbles و بلوک `channels.bluebubbles` را حذف کنید.** OpenClaw مقدار `channels.bluebubbles` را نمی‌خواند.

## مقایسهٔ اجمالی کنش‌ها

| کنش                                                | BlueBubbles قدیمی | iMessage همراه                                                              |
| --------------------------------------------------- | ------------------ | ----------------------------------------------------------------------------- |
| ارسال متن / بازگشت به SMS                           | ✅                 | ✅                                                                            |
| ارسال رسانه (عکس، ویدئو، فایل، صدا)                | ✅                 | ✅                                                                            |
| پاسخ رشته‌ای (`reply_to_guid`)                    | ✅                 | ✅ ([#51892](https://github.com/openclaw/openclaw/issues/51892) را می‌بندد)       |
| Tapback (`react`)                                   | ✅                 | ✅                                                                            |
| ویرایش / لغو ارسال (گیرندگان macOS 13+)             | ✅                 | ✅                                                                            |
| ارسال با جلوهٔ صفحه‌نمایش                           | ✅                 | ✅ (بخشی از [#9394](https://github.com/openclaw/openclaw/issues/9394) را می‌بندد) |
| متن غنی پررنگ / مورب / زیرخط‌دار / خط‌خورده         | ✅                 | ✅ (قالب‌بندی typed-run از طریق attributedBody)                                  |
| نظرسنجی‌های بومی Messages (ایجاد و رأی‌دادن)         | ❌                 | ✅ (`actions.polls`؛ گیرندگان برای نمایش بومی به iOS/macOS 26+ نیاز دارند)      |
| تغییر نام گروه / تنظیم نماد گروه                    | ✅                 | ✅                                                                            |
| افزودن / حذف شرکت‌کننده، ترک گروه                   | ✅                 | ✅                                                                            |
| رسید خواندن و نشانگر تایپ                           | ✅                 | ✅ (مشروط به کاوش API خصوصی)                                               |
| یکپارچه‌سازی ارسال تفکیک‌شدهٔ پیش‌نمایش URL اپل     | ✅                 | ✅ (در بالادست توسط `imsg` 0.13.1 و جدیدتر مدیریت می‌شود؛ بدون تنظیم OpenClaw)         |
| بازیابی ورودی پس از راه‌اندازی مجدد                 | ✅                 | ✅ (خودکار: بازپخش `since_rowid` + حذف تکرار GUID؛ پنجرهٔ گسترده‌تر در حالت محلی)     |

iMessage پیام‌هایی را که هنگام توقف Gateway از دست رفته‌اند بازیابی می‌کند: هنگام راه‌اندازی، از آخرین rowid ارسال‌شده از طریق `imsg watch.subscribe` `since_rowid` بازپخش می‌کند، موارد تکراری را بر اساس GUID حذف می‌کند و یک مرز سنی برای انباشت قدیمی، «بمب انباشت» Push-flush را مهار می‌کند. این فرایند روی اتصال RPC ‏`imsg` اجرا می‌شود؛ بنابراین برای راه‌اندازی‌های SSH راه دور `cliPath` نیز کار می‌کند. راه‌اندازی‌های محلی پنجرهٔ بازیابی گسترده‌تری دارند، زیرا می‌توانند `chat.db` را بخوانند. به [بازیابی ورودی پس از راه‌اندازی مجدد پل یا Gateway](/fa/channels/imessage#inbound-recovery-after-a-bridge-or-gateway-restart) مراجعه کنید.

## جفت‌سازی، نشست‌ها و اتصال‌های ACP

- **فهرست‌های مجاز بر اساس شناسه منتقل می‌شوند.** `channels.imessage.allowFrom` همان رشته‌های `+15555550123` / `user@example.com` مورداستفادهٔ BlueBubbles را می‌شناسد — آن‌ها را عیناً کپی کنید.
- **تأییدهای مخزن جفت‌سازی منتقل نمی‌شوند.** مخزن جفت‌سازی برای هر کانال جدا است و هیچ‌چیز مخزن قدیمی BlueBubbles را مهاجرت نمی‌دهد. فرستندگانی که فقط از طریق جفت‌سازی تأیید شده بودند باید یک‌بار دیگر در iMessage جفت شوند، یا شناسه‌های آن‌ها را به `allowFrom` اضافه کنید.
- **نشست‌ها** همچنان به هر عامل + گفت‌وگو محدود می‌مانند. با مقدار پیش‌فرض `session.dmScope=main`، پیام‌های مستقیم در نشست اصلی عامل ادغام می‌شوند؛ نشست‌های گروهی به‌ازای هر `chat_id` (`agent:<agentId>:imessage:group:<chat_id>`) جدا باقی می‌مانند. تاریخچهٔ قدیمی گفت‌وگو تحت کلیدهای نشست BlueBubbles به نشست‌های iMessage منتقل نمی‌شود.
- **اتصال‌های ACP** که به `match.channel: "bluebubbles"` ارجاع می‌دهند باید به `"imessage"` تغییر کنند. قالب‌های `match.peer.id` (`chat_id:`، `chat_guid:`، `chat_identifier:`، شناسهٔ بدون پیشوند) یکسان هستند.

## بدون کانال بازگشت

هیچ زمان‌اجرای پشتیبانی‌شده‌ای از BlueBubbles برای بازگشت به آن وجود ندارد. اگر تأیید iMessage ناموفق بود، `channels.imessage.enabled: false` را تنظیم کنید، Gateway را راه‌اندازی مجدد کنید، مانع `imsg` را برطرف کنید و گذار را دوباره امتحان کنید.

کش پاسخ در وضعیت SQLite ‏Plugin قرار دارد. در صورت وجود، `openclaw doctor --fix` فایل جانبی قدیمی `imessage/reply-cache.jsonl` را وارد و بایگانی می‌کند.

## مرتبط

- [حذف BlueBubbles و مسیر imsg در iMessage](/fa/announcements/bluebubbles-imessage) — اطلاعیه‌ای کوتاه و خلاصه‌ای برای راهبر.
- [iMessage](/fa/channels/imessage) — مرجع کامل کانال iMessage، شامل راه‌اندازی `imsg launch` و تشخیص قابلیت‌ها.
- `/channels/bluebubbles` — نشانی قدیمی که به این راهنمای مهاجرت هدایت می‌شود.
- [جفت‌سازی](/fa/channels/pairing) — احراز هویت پیام مستقیم و جریان جفت‌سازی.
- [مسیریابی کانال](/fa/channels/channel-routing) — نحوهٔ انتخاب کانال توسط Gateway برای پاسخ‌های خروجی.
