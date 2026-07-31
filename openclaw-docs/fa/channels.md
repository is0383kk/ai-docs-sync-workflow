---
read_when:
    - می‌خواهید یک کانال چت برای OpenClaw انتخاب کنید
    - به یک مرور سریع از پلتفرم‌های پیام‌رسان پشتیبانی‌شده نیاز دارید
summary: پلتفرم‌های پیام‌رسانی که OpenClaw می‌تواند به آن‌ها متصل شود
title: کانال‌های گفتگو
x-i18n:
    generated_at: "2026-07-27T15:03:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 102ad190f5bdb61fb3610985948e022f03fd54598ed4889da7a443ec0a2bdef3
    source_path: channels/index.md
    workflow: 16
---

OpenClaw می‌تواند در هر برنامهٔ گفت‌وگویی که از قبل استفاده می‌کنید با شما صحبت کند. هر کانال از طریق Gateway متصل می‌شود.
متن در همه‌جا پشتیبانی می‌شود؛ رسانه و واکنش‌ها بسته به کانال متفاوت‌اند.

iMessage، Telegram و رابط کاربری WebChat همراه با نصب هسته ارائه می‌شوند. کانال‌هایی که با
«Plugin رسمی» مشخص شده‌اند، با یک فرمان (`openclaw plugins install @openclaw/<id>`)
یا هنگام نیاز در جریان `openclaw onboard` / `openclaw channels add` نصب می‌شوند و سپس به راه‌اندازی مجدد Gateway
نیاز دارند. کانال‌های «Plugin خارجی» خارج از مخزن OpenClaw نگه‌داری می‌شوند.

## کانال‌های پشتیبانی‌شده

- [Discord](/fa/channels/discord) - API ربات Discord + Gateway؛ از سرورها، کانال‌ها و پیام‌های مستقیم پشتیبانی می‌کند (Plugin رسمی).
- [Feishu](/fa/channels/feishu) - ربات Feishu/Lark از طریق WebSocket (Plugin رسمی).
- [Google Chat](/fa/channels/googlechat) - برنامهٔ Google Chat API از طریق Webhook ‏HTTP (Plugin رسمی).
- [iMessage](/fa/channels/imessage) - در هسته گنجانده شده است. یکپارچه‌سازی بومی macOS از طریق پل `imsg` روی یک Mac واردشده به حساب (یا پوشش SSH هنگامی که Gateway جای دیگری اجرا می‌شود)، شامل عملیات API خصوصی برای پاسخ‌ها، tapbackها، جلوه‌ها، پیوست‌ها و مدیریت گروه.
- [IRC](/fa/channels/irc) - سرورهای کلاسیک IRC؛ کانال‌ها + پیام‌های مستقیم با کنترل‌های جفت‌سازی/فهرست مجاز (Plugin رسمی).
- [LINE](/fa/channels/line) - ربات LINE Messaging API (Plugin رسمی).
- [Matrix](/fa/channels/matrix) - پروتکل Matrix (Plugin رسمی).
- [Mattermost](/fa/channels/mattermost) - API ربات + WebSocket؛ کانال‌ها، گروه‌ها و پیام‌های مستقیم (Plugin رسمی).
- [Microsoft Teams](/fa/channels/msteams) - Bot Framework؛ پشتیبانی سازمانی (Plugin رسمی).
- [Nextcloud Talk](/fa/channels/nextcloud-talk) - گفت‌وگوی خودمیزبان از طریق Nextcloud Talk (Plugin رسمی).
- [Nostr](/fa/channels/nostr) - پیام‌های مستقیم غیرمتمرکز از طریق NIP-04 (Plugin رسمی).
- [QQ Bot](/fa/channels/qqbot) - QQ Bot API؛ گفت‌وگوی خصوصی، گفت‌وگوی گروهی و رسانهٔ غنی (Plugin رسمی).
- [Reef](/channels/reef) - پیام‌رسانی محافظت‌شده و رمزگذاری‌شدهٔ سرتاسری از claw به claw میان عامل‌های OpenClaw متعلق به افراد مختلف (Plugin همراه).
- [Raft](/fa/channels/raft) - پل بیدارسازی CLI ‏Raft برای همکاری انسان و عامل (Plugin رسمی).
- [Signal](/fa/channels/signal) - signal-cli؛ با تمرکز بر حریم خصوصی (Plugin رسمی).
- [Slack](/fa/channels/slack) - Bolt SDK؛ برنامه‌های فضای کاری (Plugin رسمی).
- [SMS](/fa/channels/sms) - پیامک مبتنی بر Twilio از طریق Webhook ‏Gateway (Plugin رسمی).
- [Synology Chat](/fa/channels/synology-chat) - گفت‌وگوی Synology NAS از طریق Webhookهای خروجی و ورودی (Plugin رسمی).
- [Telegram](/fa/channels/telegram) - در هسته گنجانده شده است. API ربات از طریق grammY؛ از گروه‌ها پشتیبانی می‌کند.
- [Tlon](/fa/channels/tlon) - پیام‌رسان مبتنی بر Urbit (Plugin رسمی).
- [Twitch](/fa/channels/twitch) - گفت‌وگوی Twitch از طریق اتصال IRC (Plugin رسمی).
- [تماس صوتی](/fa/plugins/voice-call) - تلفن از طریق Plivo، Telnyx یا Twilio (Plugin رسمی).
- [WebChat](/fa/web/webchat) - در هسته گنجانده شده است. رابط کاربری WebChat ‏Gateway بر بستر WebSocket.
- [WeChat](/fa/channels/wechat) - ربات Tencent iLink از طریق ورود با کد QR؛ فقط گفت‌وگوهای خصوصی (Plugin خارجی).
- [WhatsApp](/fa/channels/whatsapp) - محبوب‌ترین؛ از Baileys استفاده می‌کند و به جفت‌سازی با کد QR نیاز دارد (Plugin رسمی).
- [Yuanbao](/fa/channels/yuanbao) - ربات Tencent Yuanbao (Plugin خارجی).
- [Zalo](/fa/channels/zalo) - Zalo Bot API؛ پیام‌رسان محبوب ویتنام (Plugin رسمی).
- [Zalo ClawBot](/fa/channels/zaloclawbot) - دستیار شخصی Zalo از طریق ورود با کد QR؛ وابسته به مالک (Plugin خارجی).
- [Zalo Personal](/fa/channels/zalouser) - حساب شخصی Zalo از طریق ورود با کد QR (Plugin رسمی).

## نکات تحویل

- پاسخ‌های Telegram که شامل نحو تصویر Markdown، مانند `![alt](url)`، هستند،
  در صورت امکان در مسیر نهایی خروجی به پاسخ‌های رسانه‌ای تبدیل می‌شوند.
- پیام‌های مستقیم چندنفرهٔ Slack به‌صورت گفت‌وگوی گروهی مسیریابی می‌شوند؛ بنابراین خط‌مشی گروه، رفتار
  اشاره و قواعد نشست گروهی برای مکالمات MPIM اعمال می‌شوند.
- راه‌اندازی WhatsApp به‌صورت نصب هنگام نیاز است: فرایند آغاز به کار می‌تواند پیش از
  نصب بستهٔ Plugin، جریان راه‌اندازی را نمایش دهد و Gateway فقط هنگامی Plugin خارجی
  ClawHub/npm را بارگیری می‌کند که کانال واقعاً فعال باشد.
- کانال‌هایی که پیام‌های ورودی نوشته‌شده توسط ربات را می‌پذیرند، می‌توانند از
  [محافظت در برابر حلقهٔ ربات](/fa/channels/bot-loop-protection) مشترک استفاده کنند تا از پاسخ‌دادن
  بی‌پایان جفت‌های ربات به یکدیگر جلوگیری شود.
- اتاق‌های همیشه‌فعال پشتیبانی‌شده می‌توانند از [رویدادهای محیطی اتاق](/fa/channels/ambient-room-events)
  استفاده کنند تا گفت‌وگوهای اتاق که در آن‌ها به عامل اشاره نشده است، به زمینه‌ای آرام تبدیل شوند؛ مگر اینکه عامل با
  ابزار `message` پیام ارسال کند.

## یادداشت‌ها

- کانال‌ها می‌توانند هم‌زمان اجرا شوند؛ چند کانال را پیکربندی کنید تا OpenClaw مسیریابی را برای هر گفت‌وگو انجام دهد.
- سریع‌ترین راه‌اندازی معمولاً **Telegram** است (توکن سادهٔ ربات، بدون نصب Plugin). WhatsApp
  به جفت‌سازی با کد QR نیاز دارد و وضعیت بیشتری را روی دیسک ذخیره می‌کند.
- رفتار گروه بسته به کانال متفاوت است؛ [گروه‌ها](/fa/channels/groups) را ببینید.
- جفت‌سازی پیام مستقیم و فهرست‌های مجاز برای ایمنی اعمال می‌شوند؛ [امنیت](/fa/gateway/security) را ببینید.
- عیب‌یابی: [عیب‌یابی کانال](/fa/channels/troubleshooting).
- ارائه‌دهندگان مدل جداگانه مستند شده‌اند؛ [ارائه‌دهندگان مدل](/fa/providers/models) را ببینید.
