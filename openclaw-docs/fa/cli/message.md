---
read_when:
    - افزودن یا تغییر کنش‌های CLI پیام
    - تغییر رفتار کانال خروجی
summary: مرجع CLI برای `openclaw message` (ارسال + کنش‌های کانال)
title: پیام
x-i18n:
    generated_at: "2026-07-27T15:19:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e2d1cca9be7cfa7625cac3e440ecb5847d9fab9c545c9267a41a2f99c26c514b
    source_path: cli/message.md
    workflow: 16
---

# `openclaw message`

فرمان خروجی واحد برای ارسال پیام‌ها و کنش‌های کانال در
Discord، Google Chat، iMessage، Matrix، Mattermost (Plugin)، Microsoft Teams،
Signal، Slack، Telegram و WhatsApp.

```bash
openclaw message <subcommand> [flags]
```

## انتخاب کانال

- `--channel <name>` در صورتی الزامی است که بیش از یک کانال پیکربندی شده باشد؛ اگر
  دقیقاً یک کانال پیکربندی شده باشد، همان کانال پیش‌فرض است.
- مقادیر: `discord|googlechat|imessage|matrix|mattermost|msteams|signal|slack|telegram|whatsapp`
  (Mattermost به Plugin نیاز دارد).
- مقصدهای دارای پیشوند کانال (برای مثال `discord:channel:123`) بدون
  `--channel` صریح، Plugin مالک را تشخیص می‌دهند.

## قالب‌های مقصد (`-t, --target`)

| کانال               | قالب                                                                                                       |
| ------------------- | ---------------------------------------------------------------------------------------------------------- |
| Discord             | `channel:<id>`، `user:<id>`، منشن `<@id>` یا شناسه عددی ساده (به‌عنوان شناسه کانال در نظر گرفته می‌شود)               |
| Google Chat         | `spaces/<spaceId>` یا `users/<userId>`                                                                     |
| iMessage            | شناسه تماس، `chat_id:<id>`، `chat_guid:<guid>` یا `chat_identifier:<id>`                                      |
| Mattermost (Plugin) | `channel:<id>`، `user:<id>`، `@username` یا شناسه ساده (به‌عنوان کانال در نظر گرفته می‌شود)                              |
| Matrix              | `@user:server`، `!room:server` یا `#alias:server`                                                         |
| Microsoft Teams     | `conversation:<id>` ‏(`19:...@thread.tacv2`)، شناسه مکالمه ساده یا `user:<aad-object-id>`             |
| Signal              | `+E.164`، `group:<id>`، `uuid:<id>`، `username:<name>`/`u:<name>` یا هرکدام از این موارد با پیشوند `signal:` |
| Slack               | `channel:<id>` یا `user:<id>` (شناسه ساده به‌عنوان کانال در نظر گرفته می‌شود)                                          |
| Telegram            | شناسه گفتگو، `@username` یا مقصد موضوع انجمن: `<chatId>:topic:<topicId>` (یا `--thread-id <topicId>`)     |
| WhatsApp            | E.164،‏ JID گروه (`...@g.us`) یا JID کانال/خبرنامه (`...@newsletter`)                                |

جست‌وجوی نام کانال: برای ارائه‌دهندگانی که فهرست راهنما دارند (Discord/Slack/و غیره)، نام‌هایی
مانند `Help` یا `#help` از طریق کش فهرست راهنما تشخیص داده می‌شوند و در صورت نبودن مورد در کش،
اگر ارائه‌دهنده پشتیبانی کند، جست‌وجوی زنده در فهرست راهنما انجام می‌شود.

## پرچم‌های مشترک

هر کنش این موارد را می‌پذیرد: `--channel <name>`، `--account <id>`، `--json`،
`--dry-run`، `--verbose`. کنش‌هایی که مقصد می‌گیرند، `-t, --target <dest>` را نیز
می‌پذیرند.

## تفکیک SecretRef

`openclaw message` پیش از اجرای کنش، SecretRefهای کانال را
با محدودترین دامنه ممکن تفکیک می‌کند:

- با دامنه کانال، هنگامی که `--channel` تنظیم شده باشد (یا از مقصد پیشوندداری استنباط شود)
- با دامنه حساب، هنگامی که `--account` نیز تنظیم شده باشد
- همه کانال‌های پیکربندی‌شده، هنگامی که هیچ‌کدام تنظیم نشده باشند

SecretRefهای تفکیک‌نشده در کانال‌های نامرتبط هرگز کنش هدفمند را مسدود نمی‌کنند؛
وجود SecretRef تفکیک‌نشده در کانال/حساب انتخاب‌شده باعث می‌شود کنش به‌صورت بسته شکست بخورد.

## کنش‌ها

### اصلی

| کنش            | کانال‌ها                                                                                                        | الزامی                                                         | توضیحات                                                                                                                                                                                                                                                                                                |
| --------------- | --------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `send`          | Discord، Google Chat، iMessage، Matrix، Mattermost (Plugin)، Microsoft Teams، Signal، Slack، Telegram، WhatsApp | `--target`، به‌علاوه یکی از `--message`/`--media`/`--presentation` | بخش [ارسال](#send) را در ادامه ببینید.                                                                                                                                                                                                                                                                               |
| `poll`          | Discord، Matrix، Microsoft Teams، Telegram، WhatsApp                                                            | `--target`، `--poll-question`، `--poll-option` (تکرار)        | بخش [نظرسنجی](#poll) را در ادامه ببینید.                                                                                                                                                                                                                                                                               |
| `react`         | Discord، Matrix، Nextcloud Talk، Signal، Slack، Telegram، WhatsApp                                              | `--message-id`، `--target`                                     | `--emoji`، `--remove` (به `--emoji` نیاز دارد؛ برای پاک‌کردن واکنش‌های خود در موارد پشتیبانی‌شده، آن را حذف کنید؛ [واکنش‌ها](/fa/tools/reactions) را ببینید). WhatsApp:‏ `--participant`، `--from-me`. واکنش‌های گروهی Signal به `--target-author` یا `--target-author-uuid` نیاز دارند. Nextcloud Talk فقط واکنش اضافه می‌کند؛ `--remove` خطا می‌دهد. |
| `reactions`     | Discord، Matrix، Microsoft Teams، Slack                                                                         | `--message-id`، `--target`                                     | `--limit`.                                                                                                                                                                                                                                                                                             |
| `read`          | Discord، Matrix، Microsoft Teams، Slack                                                                         | `--target`                                                     | `--limit`، `--message-id`، `--before`، `--after`. Discord:‏ `--around`، `--include-thread`. Slack:‏ `--message-id` یک برچسب زمانی مشخص را می‌خواند؛ برای پاسخ دقیق به یک رشته، آن را با `--thread-id` ترکیب کنید.                                                                                                     |
| `edit`          | Discord، Matrix، Microsoft Teams، Slack، Telegram                                                               | `--message-id`، `--message`، `--target`                        | رشته‌های انجمن Telegram از `--thread-id` استفاده می‌کنند.                                                                                                                                                                                                                                                              |
| `delete`        | Discord، Matrix، Microsoft Teams، Slack، Telegram                                                               | `--message-id`، `--target`                                     |                                                                                                                                                                                                                                                                                                        |
| `pin` / `unpin` | Discord، Matrix، Microsoft Teams، Slack                                                                         | `--message-id`، `--target`                                     | `unpin` همچنین `--pinned-message-id` را می‌پذیرد (در Microsoft Teams: شناسه منبع سنجاق/فهرست سنجاق‌ها، نه شناسه پیام گفتگو).                                                                                                                                                                                  |
| `pins` (فهرست)   | Discord، Matrix، Microsoft Teams، Slack                                                                         | `--target`                                                     | `--limit`.                                                                                                                                                                                                                                                                                             |
| `permissions`   | Discord، Matrix                                                                                                 | `--target`                                                     | Matrix: فقط هنگامی در دسترس است که رمزگذاری فعال باشد و کنش‌های تأیید مجاز باشند.                                                                                                                                                                                                                |
| `search`        | Discord                                                                                                         | `--guild-id`، `--query`                                        | `--channel-id`، `--channel-ids` (تکرار)، `--author-id`، `--author-ids` (تکرار)، `--limit`.                                                                                                                                                                                                           |
| `member info`   | Discord، Matrix، Microsoft Teams، Slack                                                                         | `--user-id`                                                    | `--guild-id` ‏(Discord).                                                                                                                                                                                                                                                                                |

### ارسال

```bash
openclaw message send --channel discord \
  --target channel:123 --message "سلام" --reply-to 456
```

- `--media <path-or-url>`: پیوست‌کردن تصویر/صوت/ویدئو/سند (مسیر محلی یا
  URL).
- `--presentation <json>`: بار داده مشترک با بلوک‌های `text`، `context`، `divider`،
  `chart`، `table`، `buttons` و `select` که متناسب با قابلیت هر کانال
  رندر می‌شود. [نحوه نمایش پیام](/fa/plugins/message-presentation) را ببینید.
- `--delivery <json>`: ترجیحات عمومی تحویل، برای مثال `{"pin":
true}`. در کانال‌هایی که پشتیبانی می‌کنند، `--pin` شکل کوتاه‌شده تحویل سنجاق‌شده
  است.
- `--reply-to <id>`، `--thread-id <id>` (موضوع انجمن Telegram؛ برچسب زمانی رشته Slack،
  همان فیلد `--reply-to`).
- `--force-document` ‏(Telegram، WhatsApp): تصاویر/GIFها/ویدئوها را به‌صورت
  سند ارسال می‌کند تا از فشرده‌سازی کانال جلوگیری شود.
- `--silent` ‏(Telegram، Discord): ارسال بدون اعلان.
- `--gif-playback` (فقط WhatsApp): رسانه ویدئویی را به‌صورت پخش GIF در نظر می‌گیرد.

```bash
openclaw message send --channel discord \
  --target channel:123 --message "انتخاب کنید:" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"تأیید","value":"approve","style":"success"},{"label":"رد","value":"decline","style":"danger"}]}]}'
```

```bash
openclaw message send --channel telegram --target @mychat --message "انتخاب کنید:" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"بله","value":"cmd:yes"},{"label":"خیر","value":"cmd:no"}]}]}'
```

Slack بلوک‌های نمودار پشتیبانی‌شده را به‌صورت بومی رندر می‌کند؛ کانال‌های دیگر همان
داده‌ها را به‌شکل متن خوانا دریافت می‌کنند:

```bash
openclaw message send --channel slack --target channel:C123 \
  --presentation '{"blocks":[{"type":"chart","chartType":"bar","title":"درآمد فصلی","categories":["سه‌ماهه اول","سه‌ماهه دوم"],"series":[{"name":"درآمد","values":[120,145]}],"xLabel":"سه‌ماهه"}]}'
```

Slack همچنین بلوک‌های جدول صریح را به‌صورت بومی رندر می‌کند. کانال‌های دیگر
عنوان و همه ردیف‌ها را به‌شکل متن قطعی دریافت می‌کنند:

```bash
openclaw message send --channel slack --target channel:C123 \
  --presentation '{"title":"گزارش پایپ‌لاین","blocks":[{"type":"table","caption":"پایپ‌لاین باز","headers":["حساب","مرحله","ARR"],"rows":[["Acme","Won",125000],["Globex","Review",82000]],"rowHeaderColumnIndex":0}]}'
```

دکمه‌های Mini App در Telegram از `webApp` استفاده می‌کنند (`web_app` همچنان برای JSON قدیمی
تجزیه می‌شود) و فقط در گفت‌وگوهای خصوصی میان کاربر و ربات رندر می‌شوند:

```bash
openclaw message send --channel telegram --target 123456789 --message "باز کردن برنامه:" \
  --presentation '{"blocks":[{"type":"buttons","buttons":[{"label":"اجرا","webApp":{"url":"https://example.com/app"}}]}]}'
```

```bash
openclaw message send --channel telegram --target @mychat \
  --media ./diagram.png --force-document
```

```bash
openclaw message send --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --presentation '{"title":"به‌روزرسانی وضعیت","blocks":[{"type":"text","text":"ساخت تکمیل شد"}]}'
```

### نظرسنجی

```bash
openclaw message poll --channel discord \
  --target channel:123 \
  --poll-question "میان‌وعده؟" \
  --poll-option Pizza --poll-option Sushi \
  --poll-multi --poll-duration-hours 48
```

- `--poll-option <choice>`: 2-12 بار تکرار کنید.
- `--poll-multi`: امکان انتخاب چند گزینه را فراهم می‌کند.
- Discord: `--poll-duration-hours`، `--silent`، `--message`.
- Telegram: `--poll-duration-seconds <n>` (5-600)، `--silent`،
  `--poll-anonymous` / `--poll-public`، `--thread-id`.

```bash
openclaw message poll --channel telegram \
  --target @mychat \
  --poll-question "ناهار؟" \
  --poll-option Pizza --poll-option Sushi \
  --poll-duration-seconds 120 --silent
```

```bash
openclaw message poll --channel msteams \
  --target conversation:19:abc@thread.tacv2 \
  --poll-question "ناهار؟" \
  --poll-option Pizza --poll-option Sushi
```

### رشته‌ها

- `thread create`: کانال‌های Discord. الزامی: `--thread-name`، `--target`
  (شناسه کانال). اختیاری: `--message-id`، `--message`، `--auto-archive-min`.
- `thread list`: کانال‌های Discord. الزامی: `--guild-id`. اختیاری:
  `--channel-id`، `--include-archived`، `--before`، `--limit`.
- `thread reply`: کانال‌های Discord. الزامی: `--target` (شناسه رشته)،
  `--message`. اختیاری: `--media`، `--reply-to`.

### ایموجی‌ها

- `emoji list`: Discord (`--guild-id`)، Slack (بدون پرچم اضافی).
- `emoji upload`: Discord. الزامی: `--guild-id`، `--emoji-name`، `--media`.
  اختیاری: `--role-ids` (تکرارشونده).

### استیکرها

- `sticker send`: Discord. الزامی: `--target`، `--sticker-id` (تکرارشونده).
  اختیاری: `--message`.
- `sticker upload`: Discord. الزامی: `--guild-id`، `--sticker-name`،
  `--sticker-desc`، `--sticker-tags`، `--media`.

### نقش‌ها، کانال‌ها، صوت و رویدادها (Discord)

- `role info`: `--guild-id`.
- `role add` / `role remove`: `--guild-id`، `--user-id`، `--role-id`.
- `channel info`: `--target`.
- `channel list`: `--guild-id`.
- `voice status`: `--guild-id`، `--user-id`.
- `event list`: `--guild-id`.
- `event create`: الزامی `--guild-id`، `--event-name`، `--start-time`؛
  اختیاری `--end-time`، `--desc`، `--channel-id`، `--location`،
  `--event-type`، `--image <url-or-path>`.

### مدیریت محتوا (Discord)

- `timeout`: `--guild-id`، `--user-id`؛ اختیاری `--duration-min` یا
  `--until` (برای پاک‌کردن مهلت زمانی، هر دو را حذف کنید)، `--reason`.
- `kick`: `--guild-id`، `--user-id`، `--reason`.
- `ban`: `--guild-id`، `--user-id`، `--delete-days`، `--reason`.

### پخش همگانی

```bash
openclaw message broadcast --targets <target...> [--channel all] [--message <text>] [--media <url>] [--dry-run]
```

یک محموله را به چندین مقصد ارسال می‌کند. `--targets` فهرستی با جداکننده فاصله
می‌پذیرد. برای هدف‌گیری همه ارائه‌دهندگان پیکربندی‌شده، از `--channel all` استفاده کنید.

## مرتبط

- [مرجع CLI](/fa/cli)
- [ارسال عامل](/fa/tools/agent-send)
- [ارائه پیام](/fa/plugins/message-presentation)
