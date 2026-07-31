---
read_when:
    - راه‌اندازی Slack یا اشکال‌زدایی حالت سوکت، HTTP یا رله Slack
summary: راه‌اندازی و رفتار زمان اجرای Slack (Socket Mode، نشانی‌های URL درخواست HTTP و حالت رله)
title: Slack
x-i18n:
    generated_at: "2026-07-27T13:54:11Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e0f974ddf8e6965b09cede6a16f171434915a994fa3c1fc744d2350399941bee
    source_path: channels/slack.md
    workflow: 16
---

پشتیبانی Slack پیام‌های خصوصی و کانال‌ها را از طریق یکپارچه‌سازی‌های اپ Slack پوشش می‌دهد. انتقال پیش‌فرض Socket Mode است؛ URLهای درخواست HTTP نیز پشتیبانی می‌شوند. حالت رله برای استقرارهای مدیریت‌شده‌ای است که در آن‌ها یک مسیریاب مورد اعتماد ورودی Slack را در اختیار دارد.

<CardGroup cols={3}>
  <Card title="جفت‌سازی" icon="link" href="/fa/channels/pairing">
    پیام‌های خصوصی Slack به‌طور پیش‌فرض از حالت جفت‌سازی استفاده می‌کنند.
  </Card>
  <Card title="دستورهای اسلش" icon="terminal" href="/fa/tools/slash-commands">
    رفتار بومی دستورها و فهرست دستورها.
  </Card>
  <Card title="عیب‌یابی کانال" icon="wrench" href="/fa/channels/troubleshooting">
    روش‌های تشخیص و دستورالعمل‌های رفع مشکل میان کانال‌ها.
  </Card>
</CardGroup>

## انتخاب روش انتقال

Socket Mode و URLهای درخواست HTTP برای پیام‌رسانی، دستورهای اسلش، App Home و تعامل‌پذیری از نظر قابلیت‌ها هم‌سطح‌اند. انتخاب را بر اساس شکل استقرار انجام دهید، نه قابلیت‌ها.

| ملاحظه                      | Socket Mode (پیش‌فرض)                                                                                                                                | URLهای درخواست HTTP                                                                                              |
| ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| URL عمومی Gateway           | لازم نیست                                                                                                                                         | الزامی است (DNS، TLS، پراکسی معکوس یا تونل)                                                                   |
| شبکه خروجی             | WSS خروجی به `wss-primary.slack.com` باید قابل دسترسی باشد                                                                                            | بدون WS خروجی؛ فقط HTTPS ورودی                                                                             |
| توکن‌های مورد نیاز                | هویت ربات: توکن ربات + توکن سطح اپ با `connections:write`؛ هویت کاربر: توکن کاربر + توکن سطح اپ                                      | هویت ربات: توکن ربات + راز امضا؛ هویت کاربر: توکن کاربر + راز امضا                           |
| لپ‌تاپ توسعه / پشت دیوار آتش | بدون تغییر کار می‌کند                                                                                                                                          | به تونل عمومی (ngrok، Cloudflare Tunnel، Tailscale Funnel) یا Gateway آزمایشی نیاز دارد                          |
| مقیاس‌پذیری افقی           | یک نشست Socket Mode برای هر اپ در هر میزبان؛ چند Gateway به اپ‌های Slack جداگانه نیاز دارند                                                                 | کنترل‌کننده POST بدون حالت؛ چند نمونه Gateway می‌توانند یک اپ را پشت متعادل‌کننده بار به‌اشتراک بگذارند                     |
| چند حساب در یک Gateway | پشتیبانی می‌شود؛ هر حساب WS خود را باز می‌کند                                                                                                             | پشتیبانی می‌شود؛ هر حساب به یک `webhookPath` یکتا (پیش‌فرض `/slack/events`) نیاز دارد تا ثبت‌ها تداخل نکنند |
| انتقال دستور اسلش      | از طریق اتصال WS تحویل می‌شود؛ `slash_commands[].url` نادیده گرفته می‌شود                                                                                  | Slack درخواست POST را به `slash_commands[].url` می‌فرستد؛ این فیلد برای ارسال دستور الزامی است                           |
| امضای درخواست              | استفاده نمی‌شود (احراز هویت با توکن سطح اپ انجام می‌شود)                                                                                                               | Slack هر درخواست را امضا می‌کند؛ OpenClaw آن را با `signingSecret` اعتبارسنجی می‌کند                                              |
| بازیابی پس از قطع اتصال  | اتصال مجدد خودکار Slack SDK فعال است؛ OpenClaw نیز نشست‌های ناموفق Socket Mode را با تأخیر افزایشی محدود دوباره راه‌اندازی می‌کند. تنظیم انتقال مربوط به مهلت پایان pong اعمال می‌شود. | اتصال پایداری وجود ندارد که قطع شود؛ تلاش‌های مجدد برای هر درخواست از سوی Slack انجام می‌شوند                                           |

<Note>
  **برای میزبان‌های تک‌Gateway، لپ‌تاپ‌های توسعه و شبکه‌های درون‌سازمانی که می‌توانند به `*.slack.com` به‌صورت خروجی دسترسی داشته باشند اما نمی‌توانند HTTPS ورودی بپذیرند، Socket Mode را انتخاب کنید.**

**هنگام اجرای چند نمونه Gateway پشت متعادل‌کننده بار، زمانی که WSS خروجی مسدود اما HTTPS ورودی مجاز است، یا زمانی که Webhookهای Slack را از قبل در یک پراکسی معکوس خاتمه می‌دهید، URLهای درخواست HTTP را انتخاب کنید.**
</Note>

<Warning>
  Slack می‌تواند چند اتصال Socket Mode را برای یک اپ حفظ کند و ممکن است هر بار داده را به هرکدام از اتصال‌ها تحویل دهد. بنابراین، Gatewayهای جداگانه OpenClaw که یک اپ Slack را به‌اشتراک می‌گذارند، به پیکربندی مسیریابی و مجوزدهی یکسان نیاز دارند. در غیر این صورت، برای هر Gateway از یک اپ Slack جداگانه، یک ورودی رله واحد، یا URLهای درخواست HTTP پشت متعادل‌کننده بار استفاده کنید. به [استفاده از Socket Mode](https://docs.slack.dev/apis/events-api/using-socket-mode#using-multiple-connections) مراجعه کنید.
</Warning>

### حالت رله

حالت رله ورودی Slack را از Gateway ‏OpenClaw جدا می‌کند. یک مسیریاب مورد اعتماد، اتصال واحد Slack Socket Mode را در اختیار دارد، Gateway مقصد را انتخاب می‌کند و یک رویداد نوع‌دار را از طریق وب‌سوکت احرازهویت‌شده ارسال می‌کند. Gateway همچنان از توکن ربات خود برای فراخوانی‌های خروجی Slack Web API استفاده می‌کند.

```json5
{
  channels: {
    slack: {
      mode: "relay",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      relay: {
        url: "wss://router.example.com/gateway/ws",
        authToken: { source: "env", provider: "default", id: "SLACK_RELAY_AUTH_TOKEN" },
        gatewayId: "team-gateway",
      },
    },
  },
}
```

URL رله باید از `wss://` استفاده کند، مگر اینکه localhost را هدف قرار دهد. توکن حامل و جدول مسیر مسیریاب را بخشی از مرز مجوزدهی Slack در نظر بگیرید: رویدادهای مسیریابی‌شده به‌عنوان فعال‌سازی‌های مجاز وارد کنترل‌کننده عادی پیام Slack می‌شوند. یک `slack_identity` ارائه‌شده توسط مسیریاب در فریم `hello` وب‌سوکت می‌تواند نام کاربری و نماد خروجی پیش‌فرض را تنظیم کند؛ هویتی که فراخواننده به‌صراحت ارائه کند همچنان اولویت دارد. اتصال رله با همان زمان‌بندی تأخیر افزایشی محدود Socket Mode دوباره متصل می‌شود و هنگام هر قطع اتصال، هویت ارائه‌شده توسط مسیریاب را پاک می‌کند.

### نصب‌های سراسری سازمانی Enterprise Grid

یک حساب Slack می‌تواند از تمام فضاهای کاری تحت پوشش نصب سراسری سازمانی
Enterprise Grid پیام دریافت کند. Socket Mode مستقیم یا URLهای درخواست HTTP
را انتخاب کنید؛ حالت رله برای حساب‌های سازمانی پشتیبانی نمی‌شود. هر دو
مانیفست با حداقل دسترسی زیر فقط مسیر رویداد V1 ‏`message` و `app_mention`،
پاسخ‌های فوری و واکنش‌های وضعیت تحت مالکیت شنونده را فعال می‌کنند.

#### Socket Mode

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack connector for OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

از یک مدیر سازمان یا مالک سازمان Enterprise Grid بخواهید اپ را تأیید کند، آن را در
سطح سازمان نصب کند و فضاهای کاری تحت پوشش نصب را انتخاب کند.
پیش از راه‌اندازی OpenClaw، تأیید کنید که اپ در تمام فضاهای کاری مورد نظر
در دسترس است. برای Socket Mode یک توکن سطح اپ با `connections:write` ایجاد کنید،
سپس توکن ربات را از نصب سازمان کپی کنید. حسابی را که
از توکن ربات نصب‌شده در سازمان استفاده می‌کند پیکربندی کنید:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      enterpriseOrgInstall: true,
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

#### URLهای درخواست HTTP

هنگامی از حالت HTTP استفاده کنید که Gateway نقطه پایانی عمومی HTTPS دارد و اتصال
Socket Mode باز نمی‌کند. URL نمونه را با URL عمومی `webhookPath`
Gateway (پیش‌فرض `/slack/events`) جایگزین کنید:

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "Slack connector for OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true }
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "channels:history",
        "channels:read",
        "chat:write",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "mpim:history",
        "mpim:read",
        "reactions:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "org_deploy_enabled": true,
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_mention",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim"
      ]
    }
  }
}
```

از یک مدیر سازمان یا مالک سازمان Enterprise Grid بخواهید اپ را تأیید کند، آن را در
سطح سازمان نصب کند و فضاهای کاری تحت پوشش نصب را انتخاب کند.
پس از اینکه Slack نشانی Request URL را اعتبارسنجی کرد، توکن ربات نصب سازمان و
**Basic Information -> App Credentials -> Signing Secret** اپ را کپی کنید. حساب
سازمانی را با همان مسیر Request URL پیکربندی کنید:

```json5
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      enterpriseOrgInstall: true,
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: {
        source: "env",
        provider: "default",
        id: "SLACK_SIGNING_SECRET",
      },
      webhookPath: "/slack/events",
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "allowlist",
      channels: {
        C0123456789: { requireMention: true },
      },
    },
  },
}
```

هنگام راه‌اندازی، OpenClaw مقدار `enterpriseOrgInstall` را با `auth.test` ‏Slack اعتبارسنجی می‌کند.
توکن نصب‌شده در سازمان بدون این پرچم، یا توکن فضای کاری دارای این پرچم،
باعث شکست راه‌اندازی می‌شود. Slack منبع حقیقت درباره فضاهای کاری‌ای باقی می‌ماند که
مجوز نصب را داده‌اند؛ سپس OpenClaw سیاست‌های پیکربندی‌شده کانال، کاربر،
پیام خصوصی و اشاره را بر هر رویداد تحویل‌شده اعمال می‌کند. Enterprise V1 همه
رویدادهای `message` و `app_mention` نوشته‌شده توسط ربات را بدون توجه به
`allowBots` پیش از ارسال رد می‌کند، زیرا نصب‌های سازمانی هویت پایدار و
مقید به فضای کاری ربات را برای جلوگیری از حلقه فراهم نمی‌کنند.

پشتیبانی سازمانی عمداً به Socket Mode مستقیم یا رویدادهای HTTP
‏`message` و `app_mention` و پاسخ‌های فوری آن‌ها محدود شده است. حالت رله،
دستورهای اسلش، تعاملات، App Home، شنونده‌های رویداد واکنش، پین‌ها، ابزارهای
عمل Slack، تأییدهای بومی Slack، اتصال‌ها، تحویل صف‌بندی‌شده یا زمان‌بندی‌شده
و ارسال‌های پیش‌دستانه برای حساب سازمانی در دسترس نیستند. واکنش‌های خروجی
تأیید دریافت، در حال تایپ و وضعیت از طریق کلاینت Slack تحت مالکیت شنونده
پشتیبانی می‌شوند و به `reactions:write` نیاز دارند؛ اعلان‌های واکنش ورودی
و ابزارهای عمل واکنش همچنان در دسترس نیستند.

پاسخ‌های فوری از رفتار استاندارد تحویل Slack برای قطعه‌ها،
رسانه، فراداده، جایگزین هویت، بازکردن پیش‌نمایش پیوندها و رسیدها استفاده می‌کنند، اما فقط تا زمانی که
کلاینت اعتبارسنجی‌شده و متعلق به شنونده در نوبت رویداد فعال باقی بماند. صف ارسال
درون‌حافظه‌ای و رکوردهای مشارکت در رشته بر اساس فضای کاری آن
رویداد تفکیک می‌شوند؛ خود کلاینت هرگز سریال‌سازی یا پایدارسازی نمی‌شود.

کلیدهای سیاست کانال و ورودی‌های `dm.groupChannels` باید از شناسه‌های خام و پایدار کانال Slack یا
فرم `channel:<id>` استفاده کنند. OpenClaw هر دو فرم را برای
تطبیق زمان اجرا به شناسه خام کانال نرمال‌سازی می‌کند؛ پیشوندهای `slack:`،‏ `group:` و `mpim:` باعث شکست راه‌اندازی می‌شوند.
ورودی‌های سیاست کاربر باید از شناسه‌های پایدار کاربر Slack استفاده کنند؛ نام‌ها، نامک‌ها، نام‌های نمایشی
و نشانی‌های ایمیل باعث شکست راه‌اندازی می‌شوند. شناسه‌ها باید از پیشوند و بدنه استاندارد و بزرگ‌نویسی‌شده
Slack استفاده کنند (برای مثال، `C0123456789` یا `U0123456789`)؛ موارد کوچک‌نویسی‌شده و
نمونه‌های کوتاه و مشابه باعث شکست راه‌اندازی می‌شوند. حساب‌های سازمانی نمی‌توانند
`dangerouslyAllowNameMatching` را فعال کنند. حساب‌های سازمانی می‌توانند مقدار سراسری
`mentionPatterns.mode` را تنظیم کنند، اما `mentionPatterns.allowIn` و
`mentionPatterns.denyIn` باعث شکست راه‌اندازی می‌شوند، زیرا شناسه‌های ساده کانال Slack به
فضای کاری مقید نیستند و ممکن است در چند فضای کاری دوباره استفاده شوند. نصب‌های فضای کاری
رفتار موجود الگوی اشاره با محدوده مشخص را حفظ می‌کنند. هر فضای کاری پذیرفته‌شده
هویت‌های جداگانه‌ای برای مسیریابی، نشست، رونوشت، حذف تکرار، تاریخچه و حافظه نهان
دریافت می‌کند، حتی اگر شناسه‌های Slack هم‌پوشانی داشته باشند. در جریان `message`، پیام‌های عادی کاربران
و رویدادهای `file_share` نوشته‌شده توسط کاربر پشتیبانی می‌شوند؛ سایر زیرنوع‌های پیام
پیش از مجوزدهی یا مدیریت رویداد سیستمی رد می‌شوند.

پیام‌های مستقیم سازمانی باید یا غیرفعال باشند (`dm.enabled=false` یا
`dmPolicy="disabled"`) یا صراحتاً با `dmPolicy="open"` و
یک `allowFrom` مؤثر برای حساب که شامل مقدار لفظی `"*"` است، باز شوند. فهرست مجاز خالی
یا شناسه‌های مختص کاربر بدون `"*"` باعث شکست راه‌اندازی می‌شوند. جفت‌سازی و
فهرست‌های مجاز پیام مستقیم به‌ازای هر کاربر رد می‌شوند، زیرا شناسه‌های کاربر Slack در آن
مخازن مجوزدهی به فضای کاری مقید نیستند. سیاست کانال و فرستنده همچنان
برای پیام‌های کانال اعمال می‌شود.

## نصب

```bash
openclaw plugins install @openclaw/slack
```

`plugins install` افزونه را ثبت و فعال می‌کند. تا زمانی که برنامه Slack و تنظیمات کانال زیر را پیکربندی نکنید، هیچ کاری انجام نمی‌دهد. برای قواعد عمومی نصب افزونه، [افزونه‌ها](/fa/tools/plugin) را ببینید.

## راه‌اندازی سریع

مانیفست‌های این بخش یک نصب با محدوده فضای کاری ایجاد می‌کنند. برای نصب
در سطح سازمان Enterprise Grid، به‌جای آن از
[مانیفست و گردش‌کار اختصاصی سراسر سازمان](#enterprise-grid-org-wide-installs) استفاده کنید.

<Tabs>
  <Tab title="حالت سوکت (پیش‌فرض)">
    <Steps>
      <Step title="ایجاد یک برنامه جدید Slack">
        [api.slack.com/apps](https://api.slack.com/apps/new) را باز کنید ← **Create New App** ← **From a manifest** ← فضای کاری خود را انتخاب کنید ← یکی از مانیفست‌های زیر را جای‌گذاری کنید ← **Next** ← **Create**.

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "رابط Slack برای OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw مکالمات نمای عامل Slack را به عامل‌های OpenClaw متصل می‌کند.",
      "suggested_prompts": [
        { "title": "چه کارهایی می‌توانید انجام دهید؟", "message": "در چه زمینه‌ای می‌توانید به من کمک کنید؟" },
        {
          "title": "خلاصه‌سازی این کانال",
          "message": "فعالیت‌های اخیر این کانال را خلاصه کنید."
        },
        { "title": "تهیه پیش‌نویس پاسخ", "message": "برای تهیه پیش‌نویس یک پاسخ به من کمک کنید." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "ارسال پیام به OpenClaw",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "رابط Slack برای OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw مکالمات نمای عامل Slack را به عامل‌های OpenClaw متصل می‌کند.",
      "suggested_prompts": [
        { "title": "چه کارهایی می‌توانید انجام دهید؟", "message": "در چه زمینه‌ای می‌توانید به من کمک کنید؟" },
        {
          "title": "خلاصه‌سازی این کانال",
          "message": "فعالیت‌های اخیر این کانال را خلاصه کنید."
        },
        { "title": "تهیه پیش‌نویس پاسخ", "message": "برای تهیه پیش‌نویس یک پاسخ به من کمک کنید." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "ارسال پیام به OpenClaw",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    }
  }
}
```

        </CodeGroup>

        <Note>
          **توصیه‌شده** با مجموعه کامل قابلیت‌های افزونه Slack مطابقت دارد: صفحه اصلی برنامه، فرمان‌های اسلش، فایل‌ها، واکنش‌ها، سنجاق‌ها، پیام‌های مستقیم گروهی و خواندن ایموجی/گروه کاربری. وقتی سیاست فضای کاری دامنه‌ها را محدود می‌کند، **حداقلی** را انتخاب کنید — این گزینه پیام‌های مستقیم، تاریخچه کانال/گروه، اشاره‌ها و فرمان‌های اسلش را پوشش می‌دهد، اما فایل‌ها، واکنش‌ها، سنجاق‌ها، پیام مستقیم گروهی (`mpim:*`)،‏ `emoji:read` و `usergroups:read` را حذف می‌کند. برای منطق هر دامنه و گزینه‌های افزودنی مانند فرمان‌های اسلش بیشتر، [چک‌لیست مانیفست و دامنه‌ها](#manifest-and-scope-checklist) را ببینید.
        </Note>

        پس از اینکه Slack برنامه را ایجاد کرد:

        - **Basic Information -> App-Level Tokens -> Generate Token and Scopes**:‏ `connections:write` را اضافه کنید، ذخیره کنید و App-Level Token را کپی کنید.
        - **Install App -> Install to Workspace**:‏ Bot User OAuth Token را کپی کنید.

      </Step>

      <Step title="پیکربندی OpenClaw">

        راه‌اندازی توصیه‌شده SecretRef:

```bash
export SLACK_APP_TOKEN=slack-app-token-example
export SLACK_BOT_TOKEN=slack-bot-token-example
cat > slack.socket.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "socket",
      appToken: { source: "env", provider: "default", id: "SLACK_APP_TOKEN" },
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
    },
  },
}
JSON5
openclaw config patch --file ./slack.socket.patch.json5 --dry-run
openclaw config patch --file ./slack.socket.patch.json5
```

        جایگزین متغیر محیطی (فقط حساب پیش‌فرض):

```bash
SLACK_APP_TOKEN=slack-app-token-example
SLACK_BOT_TOKEN=slack-bot-token-example
```

      </Step>

      <Step title="راه‌اندازی Gateway">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>

  <Tab title="نشانی‌های درخواست HTTP">
    <Steps>
      <Step title="ایجاد یک برنامه جدید Slack">
        [api.slack.com/apps](https://api.slack.com/apps/new) را باز کنید ← **Create New App** ← **From a manifest** ← فضای کاری خود را انتخاب کنید ← یکی از مانیفست‌های زیر را جای‌گذاری کنید ← `https://gateway-host.example.com/slack/events` را با نشانی عمومی Gateway خود جایگزین کنید ← **Next** ← **Create**.

        <CodeGroup>

```json Recommended
{
  "display_information": {
    "name": "OpenClaw",
    "description": "رابط Slack برای OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw مکالمات نمای عامل Slack را به عامل‌های OpenClaw متصل می‌کند.",
      "suggested_prompts": [
        { "title": "چه کارهایی می‌توانید انجام دهید؟", "message": "در چه زمینه‌ای می‌توانید به من کمک کنید؟" },
        {
          "title": "خلاصه‌سازی این کانال",
          "message": "فعالیت‌های اخیر این کانال را خلاصه کنید."
        },
        { "title": "تهیه پیش‌نویس پاسخ", "message": "برای تهیه پیش‌نویس یک پاسخ به من کمک کنید." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "ارسال پیام به OpenClaw",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

```json Minimal
{
  "display_information": {
    "name": "OpenClaw",
    "description": "رابط Slack برای OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw مکالمات Slack Agent View را به عامل‌های OpenClaw متصل می‌کند.",
      "suggested_prompts": [
        { "title": "چه کارهایی می‌توانید انجام دهید؟", "message": "در چه زمینه‌ای می‌توانید به من کمک کنید؟" },
        {
          "title": "خلاصه‌کردن این کانال",
          "message": "فعالیت‌های اخیر این کانال را خلاصه کنید."
        },
        { "title": "نوشتن پیش‌نویس پاسخ", "message": "برای نوشتن پیش‌نویس پاسخ به من کمک کنید." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "ارسال پیام به OpenClaw",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "users:read"
      ]
    }
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "message.channels",
        "message.groups",
        "message.im"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

        </CodeGroup>

        <Note>
          **توصیه‌شده** با مجموعه کامل قابلیت‌های Plugin مربوط به Slack مطابقت دارد؛ **حداقلی** فایل‌ها، واکنش‌ها، سنجاق‌ها، پیام خصوصی گروهی (`mpim:*`)، `emoji:read` و `usergroups:read` را برای فضاهای کاری محدودکننده حذف می‌کند. برای منطق مربوط به هر دامنه، [چک‌لیست مانیفست و دامنه‌ها](#manifest-and-scope-checklist) را ببینید.
        </Note>

        <Info>
          هر سه فیلد URL ‏(`slash_commands[].url`، `event_subscriptions.request_url` و `interactivity.request_url` / `message_menu_options_url`) به یک نقطه پایانی OpenClaw اشاره می‌کنند. طرح‌واره مانیفست Slack ایجاب می‌کند که نام‌های جداگانه‌ای داشته باشند، اما OpenClaw مسیریابی را بر اساس نوع بار داده انجام می‌دهد؛ بنابراین یک `webhookPath` (با مقدار پیش‌فرض `/slack/events`) کافی است. فرمان‌های اسلش بدون `slash_commands[].url` در حالت HTTP بدون هیچ هشداری عملی انجام نمی‌دهند.
        </Info>

        پس از آنکه Slack برنامه را ایجاد کرد:

        - **Basic Information → App Credentials**: برای تأیید درخواست، **Signing Secret** را کپی کنید.
        - **Install App -> Install to Workspace**: ‏Bot User OAuth Token را کپی کنید.

      </Step>

      <Step title="پیکربندی OpenClaw">

        راه‌اندازی توصیه‌شده SecretRef:

```bash
export SLACK_BOT_TOKEN=slack-bot-token-example
export SLACK_SIGNING_SECRET=...
cat > slack.http.patch.json5 <<'JSON5'
{
  channels: {
    slack: {
      enabled: true,
      mode: "http",
      botToken: { source: "env", provider: "default", id: "SLACK_BOT_TOKEN" },
      signingSecret: { source: "env", provider: "default", id: "SLACK_SIGNING_SECRET" },
      webhookPath: "/slack/events",
    },
  },
}
JSON5
openclaw config patch --file ./slack.http.patch.json5 --dry-run
openclaw config patch --file ./slack.http.patch.json5
```

        <Note>
        برای HTTP چندحسابی از مسیرهای Webhook منحصربه‌فرد استفاده کنید

        به هر حساب یک `webhookPath` متمایز (با مقدار پیش‌فرض `/slack/events`) اختصاص دهید تا ثبت‌ها با یکدیگر تداخل نکنند.
        </Note>

      </Step>

      <Step title="راه‌اندازی Gateway">

```bash
openclaw gateway
```

      </Step>
    </Steps>

  </Tab>
</Tabs>

## هویت کاربر (ارسال به‌عنوان یک شخص واقعی)

هویت کاربر به OpenClaw امکان می‌دهد به‌عنوان انسانی که برنامه Slack را مجاز کرده است، پیام‌ها را بخواند و ارسال کند. `userToken` هویت عامل است؛ یک برنامه همراه Slack ترافیک Events API را از طریق Socket Mode یا یک HTTP Request URL منتقل می‌کند. برنامه همراه به کاربر ربات یا توکن ربات نیاز ندارد.

برنامه همراه را به‌شکل زیر راه‌اندازی کنید:

1. در بخش **OAuth & Permissions -> User Token Scopes**، این مجوزهای دارای دامنه کاربر را اضافه کنید:

   - تاریخچه: `channels:history`، `groups:history`، `im:history`، `mpim:history`
   - جست‌وجوی مکالمه: `channels:read`، `groups:read`، `im:read`، `mpim:read`
   - افراد: `users:read`
   - ارسال: `chat:write` (پیام‌ها به نام کاربر مجوزدهنده ارسال می‌شوند)
   - بازکردن پیام‌های خصوصی: `im:write`، `mpim:write`

2. در بخش **Event Subscriptions -> Subscribe to events on behalf of users**، این رویدادهای کاربر را اضافه کنید. آن‌ها را فقط به فهرست رویدادهای ربات اضافه نکنید:

   - `message.channels`
   - `message.groups`
   - `message.im`
   - `message.mpim`

3. یکی از روش‌های انتقال رویداد را انتخاب کنید:

   - **Socket Mode:** ‏Socket Mode را فعال کنید و یک توکن در سطح برنامه با `connections:write` بسازید. آن را به‌عنوان `appToken` پیکربندی کنید.
   - **HTTP Request URL:** ‏Event Subscriptions را به نقطه پایانی عمومی Slack در OpenClaw هدایت کنید و **Basic Information -> App Credentials -> Signing Secret** را کپی کنید. آن را به‌عنوان `signingSecret` پیکربندی کنید.

4. برنامه را نصب یا دوباره نصب کنید، آن را به نام انسان موردنظر مجاز کنید و توکن OAuth کاربر حاصل را در `userToken` کپی کنید.

پیکربندی Socket Mode:

```json5
{
  channels: {
    slack: {
      identity: "user",
      userToken: "<xoxp>",
      appToken: "<xapp>",
    },
  },
}
```

پیکربندی HTTP Request URL:

```json5
{
  channels: {
    slack: {
      identity: "user",
      mode: "http",
      userToken: "<xoxp>",
      signingSecret: "<signing-secret>",
      webhookPath: "/slack/events",
    },
  },
}
```

<Warning>
  پیام‌های خصوصی و پیام‌های خصوصی گروهی فقط از طریق اشتراک رویداد دارای دامنه کاربر که در بالا آمده است کار می‌کنند. یک ربات نمی‌تواند به پیام خصوصی 1:1 یک انسان بپیوندد یا به یک پیام خصوصی گروهی موجود افزوده شود. برنامه همراه زیرساختی نامرئی است: سایر اعضای Slack پیام‌ها را از طرف انسان مجوزدهنده می‌بینند، نه از طرف یک ربات OpenClaw.
</Warning>

OpenClaw رویدادهای پیام دارای دامنه کاربر را که نویسنده آن‌ها هویت انسانی تشخیص‌داده‌شده است، به‌طور خودکار حذف می‌کند؛ بنابراین پیام‌های ارسالی آن باعث پاسخ‌دادن به خود نمی‌شوند.

## تنظیم انتقال Socket Mode

OpenClaw به‌طور پیش‌فرض مهلت انتظار pong کلاینت Slack SDK را برای Socket Mode روی 15 ثانیه تنظیم می‌کند. تنظیمات انتقال را فقط زمانی تغییر دهید که به تنظیم مختص فضای کاری یا میزبان نیاز دارید:

```json5
{
  channels: {
    slack: {
      mode: "socket",
      socketMode: {
        clientPingTimeout: 20000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
    },
  },
}
```

این تنظیم را فقط برای فضاهای کاری Socket Mode به‌کار ببرید که خطاهای پایان مهلت pong وب‌سوکت/پینگ سرور Slack را ثبت می‌کنند یا روی میزبان‌هایی اجرا می‌شوند که گرسنگی حلقه رویداد در آن‌ها شناخته‌شده است. `clientPingTimeout` مدت انتظار برای pong پس از ارسال پینگ کلاینت توسط SDK است؛ `serverPingTimeout` مدت انتظار برای پینگ‌های سرور Slack است. پیام‌ها و رویدادهای برنامه همچنان وضعیت برنامه محسوب می‌شوند، نه سیگنال‌های زنده‌بودن انتقال.

نکته‌ها:

- `socketMode` در حالت HTTP Request URL نادیده گرفته می‌شود.
- تنظیمات پایه `channels.slack.socketMode` برای همه حساب‌های Slack اعمال می‌شوند، مگر اینکه بازنویسی شده باشند. بازنویسی‌های هر حساب از `channels.slack.accounts.<accountId>.socketMode` استفاده می‌کنند؛ چون این یک بازنویسی شیء است، همه فیلدهای تنظیم سوکت موردنیاز برای آن حساب را درج کنید.
- فقط `clientPingTimeout` دارای مقدار پیش‌فرض OpenClaw ‏(`15000`) است. `serverPingTimeout` و `pingPongLoggingEnabled` فقط در صورت پیکربندی به Slack SDK ارسال می‌شوند.
- تأخیر تلاش مجدد برای راه‌اندازی Socket Mode از حدود 2 ثانیه آغاز می‌شود و حداکثر به حدود 30 ثانیه می‌رسد. خطاهای قابل‌بازیابی در آغاز، انتظار آغاز و قطع اتصال تا زمان توقف کانال دوباره امتحان می‌شوند. خطاهای دائمی حساب و اعتبارنامه، مانند احراز هویت نامعتبر، توکن‌های لغوشده یا دامنه‌های مفقود، به‌جای تلاش مجدد همیشگی، سریعاً شکست می‌خورند.

## چک‌لیست مانیفست و دامنه‌ها

مانیفست پایه برنامه Slack برای Socket Mode و HTTP Request URL یکسان است. فقط بلوک `settings` (و `url` فرمان اسلش) متفاوت است.

مانیفست پایه (پیش‌فرض Socket Mode):

```json
{
  "display_information": {
    "name": "OpenClaw",
    "description": "رابط Slack برای OpenClaw"
  },
  "features": {
    "bot_user": { "display_name": "OpenClaw", "always_online": true },
    "app_home": {
      "home_tab_enabled": true,
      "messages_tab_enabled": true,
      "messages_tab_read_only_enabled": false
    },
    "agent_view": {
      "agent_description": "OpenClaw مکالمات Slack Agent View را به عامل‌های OpenClaw متصل می‌کند.",
      "suggested_prompts": [
        { "title": "چه کارهایی می‌توانید انجام دهید؟", "message": "در چه زمینه‌ای می‌توانید به من کمک کنید؟" },
        {
          "title": "خلاصه‌کردن این کانال",
          "message": "فعالیت‌های اخیر این کانال را خلاصه کنید."
        },
        { "title": "نوشتن پیش‌نویس پاسخ", "message": "برای نوشتن پیش‌نویس پاسخ به من کمک کنید." }
      ]
    },
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "ارسال پیام به OpenClaw",
        "should_escape": false
      }
    ]
  },
  "oauth_config": {
    "scopes": {
      "bot": [
        "app_mentions:read",
        "assistant:write",
        "channels:history",
        "channels:read",
        "chat:write",
        "commands",
        "emoji:read",
        "files:read",
        "files:write",
        "groups:history",
        "groups:read",
        "im:history",
        "im:read",
        "im:write",
        "mpim:history",
        "mpim:read",
        "mpim:write",
        "pins:read",
        "pins:write",
        "reactions:read",
        "reactions:write",
        "usergroups:read",
        "users:read"
      ]
    }
  },
  "settings": {
    "socket_mode_enabled": true,
    "event_subscriptions": {
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    }
  }
}
```

برای **حالت HTTP Request URLs**، ‏`settings` را با نوع HTTP جایگزین کنید و `url` را به هر فرمان اسلش بیفزایید. URL عمومی الزامی است:

```json
{
  "features": {
    "slash_commands": [
      {
        "command": "/openclaw",
        "description": "ارسال پیام به OpenClaw",
        "should_escape": false,
        "url": "https://gateway-host.example.com/slack/events"
      }
    ]
  },
  "settings": {
    "event_subscriptions": {
      "request_url": "https://gateway-host.example.com/slack/events",
      "bot_events": [
        "app_home_opened",
        "app_mention",
        "app_context_changed",
        "channel_rename",
        "member_joined_channel",
        "member_left_channel",
        "message.channels",
        "message.groups",
        "message.im",
        "message.mpim",
        "pin_added",
        "pin_removed",
        "reaction_added",
        "reaction_removed"
      ]
    },
    "interactivity": {
      "is_enabled": true,
      "request_url": "https://gateway-host.example.com/slack/events",
      "message_menu_options_url": "https://gateway-host.example.com/slack/events"
    }
  }
}
```

### تنظیمات تکمیلی مانیفست

قابلیت‌های متفاوتی را ارائه کنید که پیش‌فرض‌های بالا را گسترش می‌دهند.

مانیفست پیش‌فرض، زبانهٔ **Home** در Slack App Home را فعال می‌کند و در `app_home_opened` مشترک می‌شود. وقتی یکی از اعضای فضای کاری زبانهٔ Home را باز می‌کند، OpenClaw یک نمای Home پیش‌فرض و امن را با `views.publish` منتشر می‌کند؛ هیچ محتوای مکالمه یا پیکربندی خصوصی در آن گنجانده نمی‌شود. وقتی حالت تک‌فرمان اسلش فعال باشد، راهنمای فرمان از `channels.slack.slashCommand.name` استفاده می‌کند؛ نصب‌هایی که از فرمان‌های بومی استفاده می‌کنند یا هیچ فرمان اسلشی ندارند، این راهنما را نمایش نمی‌دهند. زبانهٔ **Messages** برای پیام‌های مستقیم Slack فعال باقی می‌ماند. برنامه‌های جدید از طریق `features.agent_view`، `assistant:write` و `app_context_changed` از Slack Agent View استفاده می‌کنند. هر ریشهٔ قابل‌مشاهدهٔ Agent View به نشست رشتهٔ OpenClaw مختص خود هدایت می‌شود و موجودیت‌های مرتب‌شدهٔ نمای فعال Slack فقط به‌عنوان زمینهٔ غیرقابل‌اعتماد به عامل می‌رسند.

برنامه‌های موجودی که از قبل از `features.assistant_view` استفاده می‌کنند، می‌توانند مانیفست فعلی خود را حفظ کنند. OpenClaw همچنان `assistant_thread_started` و `assistant_thread_context_changed` را برای آن نصب‌ها مدیریت می‌کند. Slack مهاجرت از Assistant View به Agent View را برگشت‌ناپذیر می‌کند و از کاربران می‌خواهد پس از آن بازآوری کامل انجام دهند؛ بنابراین تا زمانی که قصد مهاجرت کل فضای کاری را ندارید، `assistant_view` را در یک برنامهٔ موجود جایگزین نکنید.

<AccordionGroup>
  <Accordion title="فرمان‌های بومی اسلش اختیاری">

    می‌توان با ملاحظاتی، چند [فرمان بومی اسلش](#commands-and-slash-behavior) را به‌جای یک فرمان پیکربندی‌شده به‌کار برد:

    - به‌جای `/status` از `/agentstatus` استفاده کنید، زیرا فرمان `/status` رزرو شده است.
    - در هر لحظه نمی‌توان بیش از 25 فرمان اسلش را در یک برنامهٔ Slack ثبت کرد (محدودیت پلتفرم Slack).

    OpenClaw برای فرمان‌های بومی فعال، مدیریت‌کننده ثبت می‌کند؛ اما ورودی‌های مانیفست Slack همچنان تحت مدیریت مدیر هستند و هنگام اجرا همگام‌سازی نمی‌شوند. `/login` را به‌صورت دستی به مانیفست اضافه کنید؛ برای باقی‌ماندن در سقف 25 فرمان، نمونهٔ زیر آن را به‌جای نام مستعار اختیاری `/side` دربر می‌گیرد. `/login` را می‌توان در هر جایی نمایش داد، اما فقط در گفت‌وگوهای خصوصی یا رابط وب کدهای جفت‌سازی صادر می‌کند.

    بخش `features.slash_commands` موجود را با زیرمجموعه‌ای از [فرمان‌های موجود](/fa/tools/slash-commands#command-list) جایگزین کنید:

    <Tabs>
      <Tab title="Socket Mode (پیش‌فرض)">

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "آغاز یک نشست جدید",
      "usage_hint": "[model]"
    },
    {
      "command": "/reset",
      "description": "بازنشانی نشست فعلی"
    },
    {
      "command": "/compact",
      "description": "فشرده‌سازی زمینهٔ نشست",
      "usage_hint": "[instructions]"
    },
    {
      "command": "/stop",
      "description": "توقف اجرای فعلی"
    },
    {
      "command": "/session",
      "description": "مدیریت انقضای اتصال رشته",
      "usage_hint": "بیکاری <duration|off> یا حداکثر سن <duration|off>"
    },
    {
      "command": "/think",
      "description": "تنظیم سطح تفکر",
      "usage_hint": "<level>"
    },
    {
      "command": "/verbose",
      "description": "تغییر وضعیت خروجی مشروح",
      "usage_hint": "on|off|full"
    },
    {
      "command": "/fast",
      "description": "نمایش یا تنظیم حالت سریع",
      "usage_hint": "[status|on|off]"
    },
    {
      "command": "/reasoning",
      "description": "تغییر وضعیت نمایش استدلال",
      "usage_hint": "[on|off|stream]"
    },
    {
      "command": "/elevated",
      "description": "تغییر وضعیت حالت ارتقایافته",
      "usage_hint": "[on|off|ask|full]"
    },
    {
      "command": "/exec",
      "description": "نمایش یا تنظیم پیش‌فرض‌های اجرا",
      "usage_hint": "host=<auto|sandbox|gateway|node> security=<deny|allowlist|full> ask=<off|on-miss|always> node=<id>"
    },
    {
      "command": "/approve",
      "description": "تأیید یا رد درخواست‌های تأیید معلق",
      "usage_hint": "<id> <decision>"
    },
    {
      "command": "/model",
      "description": "نمایش یا تنظیم مدل",
      "usage_hint": "[name|#|status]"
    },
    {
      "command": "/models",
      "description": "فهرست ارائه‌دهندگان/مدل‌ها",
      "usage_hint": "[provider] [page] [limit=<n>|size=<n>|all]"
    },
    {
      "command": "/help",
      "description": "نمایش خلاصهٔ کوتاه راهنما"
    },
    {
      "command": "/commands",
      "description": "نمایش کاتالوگ فرمان تولیدشده"
    },
    {
      "command": "/tools",
      "description": "نمایش ابزارهایی که عامل فعلی همین حالا می‌تواند استفاده کند",
      "usage_hint": "[compact|verbose]"
    },
    {
      "command": "/agentstatus",
      "description": "نمایش وضعیت زمان اجرا، از جمله میزان استفاده/سهمیهٔ ارائه‌دهنده در صورت دسترس‌بودن"
    },
    {
      "command": "/tasks",
      "description": "فهرست وظایف پس‌زمینهٔ فعال/اخیر برای نشست فعلی"
    },
    {
      "command": "/context",
      "description": "توضیح نحوهٔ سرهم‌بندی زمینه",
      "usage_hint": "[list|detail|json]"
    },
    {
      "command": "/whoami",
      "description": "نمایش هویت فرستندهٔ شما"
    },
    {
      "command": "/skill",
      "description": "اجرای یک مهارت با نام آن",
      "usage_hint": "<name> [input]"
    },
    {
      "command": "/btw",
      "description": "پرسیدن یک سؤال جانبی بدون تغییر زمینهٔ نشست",
      "usage_hint": "<question>"
    },
    {
      "command": "/login",
      "description": "جفت‌سازی ورود Codex",
      "usage_hint": "[codex|openai]"
    },
    {
      "command": "/usage",
      "description": "کنترل پاصفحهٔ مصرف یا نمایش خلاصهٔ هزینه",
      "usage_hint": "off|tokens|full|cost"
    }
  ]
}
```

      </Tab>
      <Tab title="نشانی‌های درخواست HTTP">
        از همان فهرست `slash_commands` در Socket Mode بالا استفاده کنید و `"url": "https://gateway-host.example.com/slack/events"` را به هر ورودی بیفزایید. نمونه:

```json
{
  "slash_commands": [
    {
      "command": "/new",
      "description": "آغاز یک نشست جدید",
      "usage_hint": "[model]",
      "url": "https://gateway-host.example.com/slack/events"
    },
    {
      "command": "/help",
      "description": "نمایش خلاصهٔ کوتاه راهنما",
      "url": "https://gateway-host.example.com/slack/events"
    }
  ]
}
```

        آن مقدار `url` را برای هر فرمان فهرست تکرار کنید.

      </Tab>
    </Tabs>

  </Accordion>
  <Accordion title="دامنه‌های اختیاری نویسندگی (عملیات نوشتن)">
    اگر می‌خواهید پیام‌های خروجی به‌جای هویت پیش‌فرض برنامهٔ Slack از هویت عامل فعال (نام کاربری و نماد سفارشی) استفاده کنند، دامنهٔ ربات `chat:write.customize` را اضافه کنید.

    اگر از نماد ایموجی استفاده می‌کنید، Slack انتظار دارد نحو `:emoji_name:` به‌کار رود.

  </Accordion>
  <Accordion title="دامنه‌های اختیاری توکن کاربر (عملیات خواندن)">
    اگر `channels.slack.userToken` را پیکربندی می‌کنید، دامنه‌های معمول خواندن عبارت‌اند از:

    - `channels:history`، `groups:history`، `im:history`، `mpim:history`
    - `channels:read`، `groups:read`، `im:read`، `mpim:read`
    - `users:read`
    - `reactions:read`
    - `pins:read`
    - `emoji:read`
    - `search:read` (اگر به خواندن‌های جست‌وجوی Slack وابسته هستید)

  </Accordion>
</AccordionGroup>

## مدل توکن

- هویت ربات (پیش‌فرض) برای Socket Mode به `botToken` + `appToken` و برای حالت HTTP به `botToken` + `signingSecret` نیاز دارد.
- هویت کاربر برای Socket Mode به `userToken` + `appToken` و برای حالت HTTP به `userToken` + `signingSecret` نیاز دارد. این هویت از توکن ربات استفاده نمی‌کند.
- حالت رله به `botToken` به‌همراه `relay.url`، `relay.authToken` و `relay.gatewayId` نیاز دارد؛ این حالت از توکن برنامه یا راز امضا استفاده نمی‌کند.
- `botToken`، `appToken`، `signingSecret`، `relay.authToken` و `userToken` رشته‌های متن ساده
  یا اشیای SecretRef را می‌پذیرند.
- توکن‌های پیکربندی بر بازگشت جایگزین محیط اولویت دارند.
- بازگشت جایگزین محیط برای `SLACK_BOT_TOKEN`، `SLACK_APP_TOKEN` و `SLACK_USER_TOKEN`، هرکدام فقط بر حساب پیش‌فرض اعمال می‌شود.
- `userToken` به‌طور پیش‌فرض رفتاری فقط‌خواندنی دارد (`userTokenReadOnly: true`).

رفتار تصویر لحظه‌ای وضعیت:

- بازرسی حساب Slack، فیلدهای `*Source` و `*Status` را برای هر اعتبارنامه پیگیری می‌کند
  (`botToken`، `appToken`، `signingSecret`، `userToken`).
- وضعیت یکی از `available`، `configured_unavailable` یا `missing` است.
- `configured_unavailable` یعنی حساب از طریق SecretRef
  یا منبع راز غیرخطی دیگری پیکربندی شده است، اما مسیر فعلی فرمان/زمان اجرا
  نتوانسته مقدار واقعی را تفکیک کند.
- در حالت HTTP، `signingSecretStatus` گنجانده می‌شود. Socket Mode برای هویت ربات از
  `botTokenStatus` + `appTokenStatus` و برای هویت کاربر از
  `userTokenStatus` + `appTokenStatus` استفاده می‌کند.

<Tip>
برای هویت ربات، عملیات و خواندن‌های فهرست راهنما می‌توانند یک توکن کاربر اختیاری را ترجیح دهند؛ عملیات نوشتن همچنان از توکن ربات استفاده می‌کنند، مگر اینکه `userTokenReadOnly: false` بازگشت جایگزین را مجاز کند. برای `identity: "user"`، عملیات خواندن و نوشتن همیشه از `userToken` استفاده می‌کنند.
</Tip>

## عملیات و دروازه‌ها

عملیات Slack با `channels.slack.actions.*` کنترل می‌شوند.

گروه‌های عملیات موجود در ابزارهای فعلی Slack:

| گروه       | پیش‌فرض |
| ---------- | ------- |
| messages   | فعال |
| reactions  | فعال |
| pins       | فعال |
| memberInfo | فعال |
| emojiList  | فعال |

عملیات فعلی پیام Slack شامل `send`، `upload-file`، `download-file`، `read`، `edit`، `delete`، `pin`، `unpin`، `list-pins`، `member-info` و `emoji-list` است. `download-file` شناسه‌های فایل Slack نمایش‌داده‌شده در جای‌نگهدارهای فایل ورودی را می‌پذیرد و برای تصاویر پیش‌نمایش تصویر یا برای انواع دیگر فایل، فرادادهٔ فایل محلی را برمی‌گرداند.

## کنترل دسترسی و مسیریابی

<Tabs>
  <Tab title="سیاست پیام مستقیم">
    `channels.slack.dmPolicy` دسترسی پیام مستقیم را کنترل می‌کند. `channels.slack.allowFrom` فهرست مجاز معیار برای پیام مستقیم است.

    - `pairing` (پیش‌فرض)
    - `allowlist`
    - `open` (نیاز دارد `channels.slack.allowFrom` شامل `"*"` باشد)
    - `disabled`

    پرچم‌های پیام مستقیم:

    - `dm.enabled` (پیش‌فرض true)
    - `channels.slack.allowFrom`
    - `dm.allowFrom` (قدیمی)
    - `dm.groupEnabled` (پیش‌فرض پیام‌های مستقیم گروهی false است)
    - `dm.groupChannels` (فهرست مجاز اختیاری MPIM)

    تقدم چندحسابی:

    - `channels.slack.accounts.default.allowFrom` فقط بر حساب `default` اعمال می‌شود.
    - حساب‌های نام‌گذاری‌شده، وقتی `allowFrom` خودشان تنظیم نشده باشد، `channels.slack.allowFrom` را به ارث می‌برند.
    - حساب‌های نام‌گذاری‌شده `channels.slack.accounts.default.allowFrom` را به ارث نمی‌برند.

    `channels.slack.dm.policy` و `channels.slack.dm.allowFrom` قدیمی همچنان برای سازگاری خوانده می‌شوند. `openclaw doctor --fix` آن‌ها را هنگامی که بتواند بدون تغییر دسترسی این کار را انجام دهد، به `dmPolicy` و `allowFrom` مهاجرت می‌دهد.

    جفت‌سازی در پیام‌های مستقیم از `openclaw pairing approve slack <code>` استفاده می‌کند.

  </Tab>

  <Tab title="سیاست کانال">
    `channels.slack.groupPolicy` مدیریت کانال را کنترل می‌کند:

    - `open`
    - `allowlist`
    - `disabled`

    فهرست مجاز کانال زیر `channels.slack.channels` قرار دارد و **باید از شناسه‌های پایدار کانال Slack** (برای مثال `C12345678`) به‌عنوان کلیدهای پیکربندی استفاده کند.

    نکتهٔ زمان اجرا: اگر `channels.slack` کاملاً وجود نداشته باشد (راه‌اندازی فقط با محیط)، زمان اجرا به `groupPolicy="allowlist"` بازمی‌گردد و هشداری ثبت می‌کند (حتی اگر `channels.defaults.groupPolicy` تنظیم شده باشد).

    تفکیک نام/شناسه:

    - ورودی‌های فهرست مجاز کانال و ورودی‌های فهرست مجاز پیام مستقیم هنگام راه‌اندازی، در صورتی که دسترسی توکن اجازه دهد، تفکیک می‌شوند
    - ورودی‌های تفکیک‌نشدهٔ نام کانال همان‌گونه که پیکربندی شده‌اند حفظ می‌شوند، اما به‌طور پیش‌فرض برای مسیریابی نادیده گرفته می‌شوند
    - مجوزدهی ورودی و مسیریابی کانال به‌طور پیش‌فرض ابتدا بر اساس شناسه انجام می‌شوند؛ تطبیق مستقیم نام کاربری/نامک به `channels.slack.dangerouslyAllowNameMatching: true` نیاز دارد

    <Warning>
    کلیدهای مبتنی بر نام (`#channel-name` یا `channel-name`) تحت `groupPolicy: "allowlist"` مطابقت **ندارند**. جست‌وجوی کانال به‌طور پیش‌فرض ابتدا بر اساس شناسه انجام می‌شود، بنابراین یک کلید مبتنی بر نام هرگز با موفقیت مسیریابی نخواهد شد و همهٔ پیام‌های آن کانال بی‌سروصدا مسدود می‌شوند. این رفتار با `groupPolicy: "open"` متفاوت است؛ در آنجا کلید کانال برای مسیریابی الزامی نیست و به نظر می‌رسد کلید مبتنی بر نام کار می‌کند.

    همیشه از شناسهٔ کانال Slack به‌عنوان کلید استفاده کنید. برای یافتن آن: روی کانال در Slack راست‌کلیک کنید ← **Copy link** — شناسه (`C...`) در انتهای URL ظاهر می‌شود.

    صحیح:

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            C12345678: { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```

    نادرست (تحت `groupPolicy: "allowlist"` بی‌سروصدا مسدود می‌شود):

    ```json5
    {
      channels: {
        slack: {
          groupPolicy: "allowlist",
          channels: {
            "#eng-my-channel": { enabled: true, requireMention: true },
          },
        },
      },
    }
    ```
    </Warning>

  </Tab>

  <Tab title="منشن‌ها و کاربران کانال">
    پیام‌های کانال به‌طور پیش‌فرض به منشن مشروط هستند.

    منابع منشن:

    - منشن صریح برنامه (`<@botId>`)
    - منشن گروه کاربری Slack (`<!subteam^S...>`)، هنگامی که کاربر ربات عضو آن گروه کاربری باشد؛ به `usergroups:read` نیاز دارد
    - الگوهای عبارت منظم منشن (`agents.entries.*.groupChat.mentionPatterns`، با بازگشت به `messages.groupChat.mentionPatterns`)
    - پاسخ‌ها به پیام خود ربات در Slack (`implicitMentions.replyToBot`)
    - پیگیری‌ها در رشته‌هایی که ربات در آن‌ها مشارکت داشته است (`implicitMentions.threadParticipation`)

    کنترل‌های هر کانال (`channels.slack.channels.<id>`؛ نام‌ها فقط از طریق تفکیک هنگام راه‌اندازی یا `dangerouslyAllowNameMatching`):

    - `requireMention`
    - `ignoreOtherMentions`
    - `replyToMode` (`off|first|all|batched`؛ حالت پاسخ در سطح حساب/نوع چت را برای این کانال بازنویسی می‌کند)
    - `users` (فهرست مجاز)
    - `allowBots`
    - `skills`
    - `systemPrompt`
    - `tools`، `toolsBySender`
    - قالب کلید `toolsBySender`: `channel:`، `id:`، `e164:`، `username:`، `name:` یا نویسهٔ عام `"*"`
      (کلیدهای قدیمی بدون پیشوند همچنان فقط به `id:` نگاشت می‌شوند)

    `ignoreOtherMentions` (پیش‌فرض `false`) پیام‌های کانالی را که کاربر یا گروه کاربری دیگری را منشن می‌کنند، اما این ربات را منشن نمی‌کنند، حذف می‌کند. پیام‌های مستقیم و پیام‌های مستقیم گروهی (MPIM) تحت‌تأثیر قرار نمی‌گیرند. این فیلتر به شناسهٔ تفکیک‌شدهٔ کاربر ربات از `auth.test` نیاز دارد؛ اگر آن هویت در دسترس نباشد (برای مثال، هویتی که فقط توکن کاربر دارد)، دروازه به‌صورت باز شکست می‌خورد و پیام‌ها بدون تغییر عبور می‌کنند.

    `allowBots` برای کانال‌ها و کانال‌های خصوصی محافظه‌کارانه عمل می‌کند: پیام‌های اتاق که ربات فرستاده است فقط زمانی پذیرفته می‌شوند که ربات فرستنده صراحتاً در فهرست مجاز `users` آن اتاق درج شده باشد، یا دست‌کم یک شناسهٔ صریح مالک Slack از `channels.slack.allowFrom` در حال حاضر عضو اتاق باشد. نویسه‌های عام و ورودی‌های مالک بر اساس نام نمایشی، شرط حضور مالک را برآورده نمی‌کنند. حضور مالک از `conversations.members` در Slack استفاده می‌کند؛ مطمئن شوید برنامه مجوز خواندن متناظر با نوع اتاق را دارد (`channels:read` برای کانال‌های عمومی و `groups:read` برای کانال‌های خصوصی). اگر جست‌وجوی اعضا ناموفق باشد، OpenClaw پیام اتاقِ ارسال‌شده توسط ربات را حذف می‌کند.

    پیام‌های پذیرفته‌شدهٔ Slack که ربات فرستاده است، از [محافظت مشترک در برابر حلقهٔ ربات](/fa/channels/bot-loop-protection) استفاده می‌کنند. `channels.defaults.botLoopProtection` را برای بودجهٔ پیش‌فرض پیکربندی کنید، سپس هرگاه فضای کاری یا کانالی به محدودیت متفاوتی نیاز داشت، آن را با `channels.slack.botLoopProtection` یا `channels.slack.channels.<id>.botLoopProtection` بازنویسی کنید.

  </Tab>
</Tabs>

## رشته‌ها، نشست‌ها و برچسب‌های پاسخ

- پیام‌های مستقیم به‌صورت `direct` مسیریابی می‌شوند؛ کانال‌ها به‌صورت `channel`؛ و MPIMها به‌صورت `group`.
- اتصال‌های مسیر Slack، شناسه‌های خام همتا را به‌همراه قالب‌های مقصد Slack مانند `channel:C12345678`، `user:U12345678` و `<@U12345678>` می‌پذیرند.
- با `session.dmScope=main` پیش‌فرض، پیام‌های مستقیم عادی Slack در نشست اصلی عامل ادغام می‌شوند. ریشه‌های Agent View و رشته‌های موجود Assistant View به‌صورت نشست‌های `:thread:<threadTs>` مجزا باقی می‌مانند.
- نشست‌های کانال: `agent:<agentId>:slack:channel:<channelId>`.
- پیام‌های عادی سطح‌بالای کانال، حتی زمانی که `replyToMode` مقداری غیر از `off` دارد، در نشست مختص همان کانال باقی می‌مانند.
- پاسخ‌های رشته‌ای کانال Slack، MPIM، Agent View و Assistant View برای پسوندهای نشست (`:thread:<threadTs>`) از `thread_ts` والد در Slack استفاده می‌کنند. رشته‌های پاسخ در پیام‌های مستقیم عادی صرفاً یک قابلیت رابط کاربری روی نشست پایهٔ پیام مستقیم باقی می‌مانند.
- OpenClaw یک ریشهٔ واجد شرایط و سطح‌بالای کانال را در `agent:<agentId>:slack:channel:<channelId>:thread:<rootTs>` مقداردهی اولیه می‌کند، هرگاه انتظار رود آن ریشه یک رشتهٔ قابل‌مشاهده در Slack آغاز کند؛ در نتیجه، ریشه و پاسخ‌های بعدی رشته یک نشست OpenClaw را به‌اشتراک می‌گذارند. این موضوع برای رویدادهای `app_mention`، تطابق‌های صریح منشن ربات یا الگوهای منشن پیکربندی‌شده و کانال‌های `requireMention: false` با `replyToMode` غیر از `off` اعمال می‌شود.
- مقدار پیش‌فرض `channels.slack.thread.historyScope` برابر `thread` است؛ مقدار پیش‌فرض `thread.inheritParent` برابر `false` است.
- `channels.slack.thread.initialHistoryLimit` تعیین می‌کند هنگام آغاز نشست رشته‌ای جدید، چه تعداد از پیام‌های موجود رشته واکشی شوند (پیش‌فرض `20`؛ برای غیرفعال‌سازی روی `0` تنظیم کنید).
- `channels.slack.implicitMentions.replyToBot` تعیین می‌کند آیا پاسخ به پیام خود ربات، شرط منشن را دور می‌زند یا نه (پیش‌فرض `true`).
- `channels.slack.implicitMentions.threadParticipation` تعیین می‌کند آیا پیگیری‌ها در رشته‌ای که ربات در آن پاسخ داده است، شرط منشن را دور می‌زنند یا نه (پیش‌فرض `true`). برای الزام یک منشن صریح جدید در این پیگیری‌ها، آن را روی `false` تنظیم کنید. `openclaw doctor --fix` کلید سابق `channels.slack.thread.requireExplicitMention` را به این پرچم مثبت و متعارف مهاجرت می‌دهد.
- بازنویسی‌های حساب در `channels.slack.accounts.<id>.implicitMentions` قرار دارند؛ پیش‌فرض‌های مشترک در `channels.defaults.implicitMentions` قرار دارند.

کنترل‌های رشته‌بندی پاسخ:

- `channels.slack.channels.<id>.replyToMode`: بازنویسی مختص هر کانال برای پیام‌های کانال/کانال خصوصی Slack
- `channels.slack.replyToMode`: `off|first|all|batched` (پیش‌فرض `off`)
- `channels.slack.replyToModeByChatType`: به‌ازای هر `direct|group|channel`
- بازگشت قدیمی برای چت‌های مستقیم: `channels.slack.dm.replyToMode`

برچسب‌های دستی پاسخ پشتیبانی می‌شوند:

- `[[reply_to_current]]`
- `[[reply_to:<id>]]`

برای پاسخ‌های صریح به رشتهٔ Slack از ابزار `message`، مقدار `replyBroadcast: true` را همراه با `action: "send"` و `threadId` یا `replyTo` تنظیم کنید تا از Slack خواسته شود پاسخ رشته را در کانال والد نیز پخش کند. این تنظیم به پرچم `reply_broadcast` در `chat.postMessage` مربوط به Slack نگاشت می‌شود و فقط برای ارسال متن یا Block Kit پشتیبانی می‌شود، نه بارگذاری رسانه.

هنگامی که فراخوانی ابزار `message` درون یک رشتهٔ Slack اجرا می‌شود و همان کانال را هدف می‌گیرد، OpenClaw معمولاً رشتهٔ فعلی Slack را مطابق `replyToMode` مؤثر در سطح حساب، نوع چت یا هر کانال به ارث می‌برد. پاسخ‌های خودکار و فراخوانی‌های `send` یا `upload-file` در همان کانال، از همان بازنویسی مختص کانال استفاده می‌کنند. برای اجبار به ایجاد یک پیام جدید در کانال والد، `topLevel: true` را روی `action: "send"` یا `action: "upload-file"` تنظیم کنید. `threadId: null` نیز به‌عنوان همان انصراف در سطح بالا پذیرفته می‌شود.

<Note>
`replyToMode="off"` رشته‌بندی اختیاری پاسخ‌های خروجی Slack، از جمله برچسب‌های صریح `[[reply_to_*]]` را غیرفعال می‌کند. Agent View و Assistant View تجربه‌های رشته‌ای مدیریت‌شده توسط Slack هستند، بنابراین پاسخ‌ها و وضعیت آن‌ها، صرف‌نظر از این تنظیم، روی ریشهٔ قابل‌مشاهده باقی می‌مانند. این تنظیم نشست‌های دیگر رشته‌های ورودی Slack را تخت نمی‌کند. این رفتار با Telegram متفاوت است؛ در آنجا برچسب‌های صریح همچنان در حالت `"off"` رعایت می‌شوند. رشته‌های Slack پیام‌ها را از کانال پنهان می‌کنند، درحالی‌که پاسخ‌های Telegram در همان جریان قابل‌مشاهده باقی می‌مانند.
</Note>

## واکنش‌های تأیید دریافت

`ackReaction` هنگامی که OpenClaw در حال پردازش یک پیام ورودی است، یک ایموجی تأیید دریافت ارسال می‌کند. `ackReactionScope` تعیین می‌کند آن ایموجی دقیقاً _چه زمانی_ ارسال شود.

به‌طور پیش‌فرض، واکنش تأیید دریافت ثابت باقی می‌ماند، درحالی‌که وضعیت بومی رشتهٔ عامل/دستیار Slack با پیام‌های بارگذاری چرخشی، پیشرفت را نشان می‌دهد. برای استفاده از چرخهٔ واکنش صف/تفکر/ابزار/انجام‌شده/خطا، `messages.statusReactions.enabled: true` را تنظیم کنید.

### ایموجی (`ackReaction`)

ترتیب تفکیک:

- `channels.slack.accounts.<accountId>.ackReaction`
- `channels.slack.ackReaction`
- `messages.ackReaction`
- ایموجی جایگزین هویت عامل (`agents.entries.*.identity.emoji`، در غیر این صورت `"eyes"` / 👀)

نکته‌ها:

- Slack انتظار کد کوتاه دارد (برای مثال `"eyes"`).
- برای غیرفعال‌کردن واکنش در حساب Slack یا به‌صورت سراسری، از `""` استفاده کنید.

### دامنه (`messages.ackReactionScope`)

ارائه‌دهندهٔ Slack دامنه را از `messages.ackReactionScope` می‌خواند (پیش‌فرض `"group-mentions"`). در حال حاضر هیچ بازنویسی در سطح حساب Slack یا کانال Slack وجود ندارد؛ مقدار برای Gateway سراسری است.

مقادیر:

- `"all"`: در پیام‌های مستقیم و گروه‌ها، از جمله رویدادهای محیطی اتاق، واکنش نشان دهید.
- `"direct"`: فقط در پیام‌های مستقیم واکنش نشان دهید.
- `"group-all"`: به همهٔ پیام‌های گروهی به‌جز رویدادهای محیطی اتاق واکنش نشان دهید (بدون پیام مستقیم).
- `"group-mentions"` (پیش‌فرض): در گروه‌ها واکنش نشان دهید، اما فقط هنگامی که ربات منشن شده باشد (یا در موارد قابل‌منشن گروهی که این قابلیت را فعال کرده‌اند). **پیام‌های مستقیم مستثنا هستند.**
- `"off"` / `"none"`: هرگز واکنش نشان ندهید.

<Note>
دامنهٔ پیش‌فرض (`"group-mentions"`) در پیام‌های مستقیم یا رویدادهای محیطی اتاق، واکنش تأیید دریافت را فعال نمی‌کند. برای مشاهدهٔ `ackReaction` پیکربندی‌شده (برای مثال `"eyes"`) در پیام‌های مستقیم ورودی Slack و رویدادهای بی‌سروصدای اتاق، `messages.ackReactionScope` را روی `"all"` تنظیم کنید. `messages.ackReactionScope` هنگام راه‌اندازی ارائه‌دهندهٔ Slack خوانده می‌شود، بنابراین برای اعمال تغییر باید Gateway راه‌اندازی مجدد شود.
</Note>

```json5
{
  messages: {
    ackReaction: "eyes",
    ackReactionScope: "all", // واکنش در پیام‌های مستقیم و گروه‌ها
  },
}
```

## پخش جریانی متن

`channels.slack.streaming` رفتار پیش‌نمایش زنده را کنترل می‌کند:

- `off`: پخش جریانی پیش‌نمایش زنده را غیرفعال کنید.
- `partial` (پیش‌فرض): متن پیش‌نمایش را با جدیدترین خروجی ناقص جایگزین کنید.
- `block`: به‌روزرسانی‌های قطعه‌ای پیش‌نمایش را به انتهای آن بیفزایید.
- `progress`: هنگام تولید، متن وضعیت پیشرفت را نمایش دهید و سپس متن نهایی را ارسال کنید.
- `streaming.preview.toolProgress`: هنگامی که پیش‌نمایش پیش‌نویس فعال است، به‌روزرسانی‌های ابزار/پیشرفت را به همان پیام پیش‌نمایش ویرایش‌شده هدایت کنید (پیش‌فرض: `true`). برای نگه‌داشتن پیام‌های ابزار/پیشرفت به‌صورت جداگانه، `false` را تنظیم کنید.
- `streaming.preview.commandText` / `streaming.progress.commandText`: برای حفظ خطوط فشردهٔ پیشرفت ابزار و در عین حال پنهان‌کردن متن خام فرمان/اجرا، روی `status` تنظیم کنید (پیش‌فرض: `raw`).

پنهان‌کردن متن خام فرمان/اجرا و حفظ خطوط فشردهٔ پیشرفت:

```json
{
  "channels": {
    "slack": {
      "streaming": {
        "mode": "progress",
        "progress": {
          "toolProgress": true,
          "commandText": "status"
        }
      }
    }
  }
}
```

`channels.slack.streaming.nativeTransport` پخش جریانی بومی متن Slack را هنگامی کنترل می‌کند که `channels.slack.streaming.mode` برابر `partial` باشد (پیش‌فرض: `true`).

کارت‌های بومی وظیفهٔ پیشرفت Slack برای حالت پیشرفت اختیاری هستند. برای ارسال کارت بومی طرح/وظیفهٔ Slack هنگام اجرای کار و سپس به‌روزرسانی همان کارت وظیفه در زمان تکمیل، `channels.slack.streaming.progress.nativeTaskCards` را همراه با `channels.slack.streaming.mode="progress"` روی `true` تنظیم کنید. بدون این پرچم، حالت پیشرفت رفتار قابل‌حمل پیش‌نمایش پیش‌نویس را حفظ می‌کند.

- برای نمایش جریان بومی متن و وضعیت رشتهٔ دستیار Slack، باید یک رشتهٔ پاسخ در دسترس باشد. انتخاب رشته همچنان از `replyToMode` پیروی می‌کند.
- ریشه‌های کانال، گفت‌وگوی گروهی و DM سطح‌بالا، هنگامی که جریان بومی در دسترس نیست یا رشتهٔ پاسخی وجود ندارد، همچنان می‌توانند از پیش‌نمایش پیش‌نویس عادی استفاده کنند.
- DMهای سطح‌بالای Slack به‌طور پیش‌فرض خارج از رشته باقی می‌مانند، بنابراین پیش‌نمایش جریان/وضعیت بومیِ رشته‌مانند Slack را نشان نمی‌دهند؛ در عوض، OpenClaw یک پیش‌نمایش پیش‌نویس را در DM ارسال و ویرایش می‌کند.
- رسانه و محموله‌های غیرمتنی به تحویل عادی بازمی‌گردند.
- نتیجه‌های نهایی رسانه/خطا، ویرایش‌های در انتظار پیش‌نمایش را لغو می‌کنند؛ نتیجه‌های نهایی متن/بلوک واجد شرایط تنها زمانی تخلیه می‌شوند که بتوانند پیش‌نمایش را درجا ویرایش کنند.
- اگر جریان در میانهٔ پاسخ ناموفق شود، OpenClaw برای محموله‌های باقی‌مانده به تحویل عادی بازمی‌گردد.

استفاده از پیش‌نمایش پیش‌نویس به‌جای جریان بومی متن Slack:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "partial",
        nativeTransport: false,
      },
    },
  },
}
```

فعال‌سازی اختیاری کارت‌های بومی وظیفهٔ پیشرفت Slack:

```json5
{
  channels: {
    slack: {
      streaming: {
        mode: "progress",
        progress: {
          nativeTaskCards: true,
          render: "rich",
        },
      },
    },
  },
}
```

کلیدهای قدیمی:

- `channels.slack.streamMode` (`replace | status_final | append`) یک نام مستعار قدیمی برای `channels.slack.streaming.mode` است.
- مقدار بولی `channels.slack.streaming` یک نام مستعار قدیمی برای `channels.slack.streaming.mode` و `channels.slack.streaming.nativeTransport` است.
- `channels.slack.chunkMode` و `channels.slack.nativeStreaming` سطح‌بالا، نام‌های مستعار قدیمی برای `channels.slack.streaming.chunkMode` و `channels.slack.streaming.nativeTransport` هستند.
- نام‌های مستعار قدیمی هنگام اجرا خوانده نمی‌شوند؛ برای بازنویسی پیکربندی ذخیره‌شدهٔ جریان Slack به کلیدهای متعارف، `openclaw doctor --fix` را اجرا کنید.

## واکنش جایگزین هنگام تایپ

`typingReaction` هنگام پردازش پاسخ توسط OpenClaw، یک واکنش موقت به پیام ورودی Slack اضافه می‌کند و پس از پایان اجرا آن را برمی‌دارد. این قابلیت بیشتر در خارج از پاسخ‌های رشته‌ای مفید است، زیرا پاسخ‌های رشته‌ای از نشانگر وضعیت پیش‌فرض «در حال تایپ...» استفاده می‌کنند.

ترتیب حل:

- `channels.slack.accounts.<accountId>.typingReaction`
- `channels.slack.typingReaction`

نکات:

- Slack انتظار کد کوتاه دارد (برای مثال `"hourglass_flowing_sand"`).
- واکنش به‌صورت بهترین تلاش اعمال می‌شود و پس از تکمیل مسیر پاسخ یا شکست، پاک‌سازی آن به‌طور خودکار انجام می‌شود.

## ورودی صوتی

برای صحبت با OpenClaw در Slack، در حال حاضر یک کلیپ صوتی Slack به برنامهٔ OpenClaw بفرستید. میکروفون دیکتهٔ Slackbot قابلیتی جداگانه و متعلق به Slack است، نه یک API برنامه.

- **[دیکتهٔ صوتی Slackbot](https://slack.com/help/articles/202026038-How-to-use-Slackbot)** در گفت‌وگوی خصوصی Slackbot کاربر قرار دارد. Slack صدای ضبط‌شده را به یک پرامپت Slackbot تبدیل می‌کند، اما از طریق Events API هیچ فایل صوتی، رویداد دیکته، پرامپت یا نشانگر منبع ورودی برای برنامه‌های شخص ثالث Slack منتشر نمی‌کند. Plugin مربوط به Slack در OpenClaw نمی‌تواند آن را فعال یا دریافت کند.
- **[کلیپ‌های صوتی Slack](https://slack.com/help/articles/4406235165587-Record-audio-and-video-clips-in-Slack)** فایل‌های ذخیره‌شدهٔ Slack هستند که می‌توان آن‌ها را در DM، کانال یا رشتهٔ OpenClaw ارسال کرد. OpenClaw کلیپ قابل‌دسترسی را با توکن بات بارگیری می‌کند، فرادادهٔ MIME کلیپ Slack را نرمال‌سازی می‌کند و آن را از طریق [پایپ‌لاین مشترک رونویسی صوتی](/fa/nodes/audio) می‌فرستد. مانیفست پیشنهادی برنامه، دامنهٔ دسترسی الزامی `files:read` را شامل می‌شود.

کلیپ‌های صوتی و دیکتهٔ Slackbot معناشناسی حریم خصوصی متفاوتی دارند: کلیپ‌ها از سیاست نگه‌داری فایل Slack پیروی می‌کنند و OpenClaw آن‌ها را برای رونویسی بارگیری می‌کند، درحالی‌که Slack می‌گوید صدای دیکته ذخیره نمی‌شود.

در کانالی با `requireMention: true`، یک کلیپ صوتی بدون زیرنویس می‌تواند با گفتن یک الگوی اشارهٔ پیکربندی‌شده (`agents.entries.*.groupChat.mentionPatterns`، با بازگشت به `messages.groupChat.mentionPatterns`) شرط ورودی را برآورده کند. OpenClaw پیش از بارگیری یا رونویسی کلیپ، فرستنده را مجاز می‌کند و سپس تنها در صورت تطابق رونویسی، آن را می‌پذیرد. رونویسی گمانه‌ای ناموفق یا نامطابق، همراه با کلیپ بارگیری‌شده دور ریخته می‌شود و در تاریخچهٔ کانال نگه‌داری نمی‌شود. هویت بومی `@bot` در Slack را نمی‌توان از گفتار استنباط کرد؛ بنابراین یک الگوی نام گفتاری پیکربندی کنید یا یک اشارهٔ تایپ‌شده بگنجانید. اگر بازتاب رونویسی فعال باشد، بازتاب تنها پس از پذیرش ارسال می‌شود.

## رسانه، قطعه‌بندی و تحویل

<AccordionGroup>
  <Accordion title="پیوست‌های ورودی">
    پیوست‌های فایل Slack از URLهای خصوصی میزبانی‌شده در Slack بارگیری می‌شوند (جریان درخواست احراز هویت‌شده با توکن) و در صورت موفقیت دریافت و اجازه‌دادن محدودیت‌های اندازه، در مخزن رسانه نوشته می‌شوند. جای‌نگهدارهای فایل شامل `fileId` مربوط به Slack هستند تا عامل‌ها بتوانند فایل اصلی را با `download-file` دریافت کنند.

    بارگیری‌ها از مهلت‌های محدود بی‌کاری و کل استفاده می‌کنند. اگر بازیابی فایل Slack متوقف یا ناموفق شود، OpenClaw پردازش پیام را ادامه می‌دهد و به جای‌نگهدار فایل بازمی‌گردد.

    سقف اندازهٔ ورودی هنگام اجرا به‌طور پیش‌فرض `20MB` است، مگر اینکه با `channels.slack.mediaMaxMb` بازنویسی شود.

  </Accordion>

  <Accordion title="متن و فایل‌های خروجی">
    - قطعه‌های متن از `channels.slack.textChunkLimit` استفاده می‌کنند (پیش‌فرض `8000`، محدود به سقف طول پیام خود Slack)
    - `channels.slack.streaming.chunkMode="newline"` تقسیم‌بندی با اولویت پاراگراف را فعال می‌کند
    - ارسال فایل از APIهای بارگذاری Slack استفاده می‌کند و می‌تواند پاسخ‌های رشته‌ای را شامل شود (`thread_ts`)
    - زیرنویس‌های طولانی فایل، نخستین قطعهٔ متن سازگار با Slack را به‌عنوان نظر بارگذاری استفاده می‌کنند و قطعه‌های باقی‌مانده را به‌صورت پیام‌های پیگیری می‌فرستند
    - سقف رسانهٔ خروجی، در صورت پیکربندی از `channels.slack.mediaMaxMb` پیروی می‌کند؛ در غیر این صورت، ارسال‌های کانال از پیش‌فرض‌های نوع MIME در پایپ‌لاین رسانه استفاده می‌کنند

  </Accordion>

  <Accordion title="مقصدهای تحویل">
    مقصدهای صریح ترجیحی:

    - `user:<id>` برای DMها
    - `channel:<id>` برای کانال‌ها

    DMهای Slack که فقط شامل متن/بلوک هستند می‌توانند مستقیماً به شناسه‌های کاربر ارسال شوند؛ بارگذاری فایل و ارسال رشته‌ای ابتدا DM را از طریق APIهای گفت‌وگوی Slack باز می‌کنند، زیرا این مسیرها به یک شناسهٔ مشخص گفت‌وگو نیاز دارند.

  </Accordion>
</AccordionGroup>

## فرمان‌ها و رفتار اسلش

فرمان‌های اسلش در Slack یا به‌صورت یک فرمان پیکربندی‌شده یا چند فرمان بومی ظاهر می‌شوند. برای تغییر پیش‌فرض‌های فرمان، `channels.slack.slashCommand` را پیکربندی کنید:

- `enabled: false`
- `name: "openclaw"`
- `sessionPrefix: "slack:slash"`
- `ephemeral: true`

```txt
/openclaw /help
```

فرمان‌های بومی به [تنظیمات اضافی مانیفست](#additional-manifest-settings) در برنامهٔ Slack شما نیاز دارند و در عوض با `channels.slack.commands.native: true` یا `commands.native: true` در پیکربندی‌های سراسری فعال می‌شوند.

- حالت خودکار فرمان بومی برای Slack **خاموش** است؛ بنابراین `commands.native: "auto"` فرمان‌های بومی Slack را فعال نمی‌کند.

```txt
/help
```

منوهای آرگومان بومی به‌ترتیب اولویت به یکی از شکل‌های زیر رندر می‌شوند:

- 3-5 گزینهٔ به‌اندازهٔ کافی کوتاه: منوی سرریز ("...")
- بیش از 100 گزینه، با امکان پالایش ناهمگام گزینه‌ها: انتخاب‌گر خارجی
- 1-2 گزینه، یا هر گزینه‌ای که مقدار رمزگذاری‌شدهٔ آن برای انتخاب‌گر بیش از حد طولانی باشد: بلوک‌های دکمه
- در غیر این صورت (6-100 گزینه، یا بیش از 100 گزینه بدون پالایش ناهمگام): منوی انتخاب ایستا، قطعه‌بندی‌شده با 100 گزینه در هر منو

```txt
/think
```

نشست‌های اسلش از کلیدهای ایزوله‌ای مانند `agent:<agentId>:slack:slash:<userId>` استفاده می‌کنند و همچنان اجرای فرمان‌ها را با استفاده از `CommandTargetSessionKey` به نشست گفت‌وگوی مقصد هدایت می‌کنند.

## نمودارهای بومی

بلوک عمومی Block Kit با نام [`data_visualization`](https://docs.slack.dev/reference/block-kit/blocks/data-visualization-block/)
در Slack، نمودارهای خطی، میله‌ای، ناحیه‌ای و دایره‌ای را در پیام‌ها رندر می‌کند. OpenClaw بلوک قابل‌حمل
`presentation` `chart` را به آن شکل بومی نگاشت می‌کند؛ افزون بر دسترسی عادی پیام
`chat:write`، هیچ دامنهٔ OAuth، بارگذاری فایل، رندرکنندهٔ تصویر یا پیکربندی اضافی Slack
لازم نیست.

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "bar",
      "title": "Quarterly revenue",
      "categories": ["Q1", "Q2"],
      "series": [{ "name": "Revenue", "values": [120, 145] }],
      "xLabel": "Quarter"
    }
  ]
}
```

محدودیت‌های Slack پیش از رندر بومی اعمال می‌شوند:

- عنوان و برچسب‌های اختیاری محورها: 50 نویسه
- دایره‌ای: 1-12 بخش مثبت
- خطی/میله‌ای/ناحیه‌ای: 1-12 سری با نام یکتا و 1-20 دستهٔ مشترک
- برچسب‌های بخش، دسته و سری: 20 نویسه
- هر سری باید برای هر دسته یک مقدار متناهی داشته باشد؛ مقادیر غیردایره‌ای
  می‌توانند منفی باشند

هر نمودار بومی همچنین یک بازنمایی متنی سطح‌بالا برای صفحه‌خوان‌ها،
اعلان‌ها، آینه‌سازی نشست و کارخواه‌هایی دارد که نمی‌توانند بلوک را رندر کنند.
ارسال‌های ارائهٔ استاندارد به دیگر کانال‌های OpenClaw، همان دادهٔ قطعی نمودار
را به‌صورت متن دریافت می‌کنند، مگر اینکه پشتیبانی بومی نمودار را اعلام کنند. اگر
Slack در جریان عرضهٔ مرحله‌ای نمودار را با `invalid_blocks` رد کند، OpenClaw
بلوک‌های دادهٔ بومی ردشده را حذف می‌کند، کنترل‌های هم‌تراز را نگه می‌دارد و
بازنمایی کامل نمودار را به‌صورت متن قابل‌مشاهده می‌فرستد.

Slack در حال حاضر حداکثر دو بلوک `data_visualization` را در هر پیام می‌پذیرد. هنگامی که
یک ارائه بیش از دو نمودار معتبر داشته باشد، OpenClaw ترتیب آن‌ها را حفظ می‌کند
و رندر بومی را در پیام‌های پیگیری ادامه می‌دهد، به‌گونه‌ای که هر پیام بیش از دو
نمودار نداشته باشد.

[عرضه برای توسعه‌دهندگان](https://docs.slack.dev/changelog/2026/06/16/block-kit-data-visualization-block/)
در Slack این بلوک را به‌عنوان قابلیتی از Block Kit برای برنامه‌ها مستند می‌کند و هیچ محدودیت
طرح پولی منتشر نمی‌کند. عبارت مربوط به واجد شرایط بودن Business+/Enterprise دربارهٔ
تولید خودکار نمودار با هوش مصنوعی توسط Slackbot است که از ارسال یک نمودار
ازپیش‌ساختاریافتهٔ Block Kit توسط برنامه جداست. نمودارها فقط بلوک پیام هستند، نه محتوای App
Home، مودال یا Canvas.

## جدول‌های بومی

بلوک کنونی Block Kit با نام [`data_table`](https://docs.slack.dev/reference/block-kit/blocks/data-table-block/)
در Slack، سطرها و ستون‌های ساختاریافته را در پیام‌ها رندر می‌کند. OpenClaw بلوک صریح و قابل‌حمل
`presentation` `table` را به `data_table` نگاشت می‌کند؛ از بلوک قدیمی
[`table`](https://docs.slack.dev/reference/block-kit/blocks/table-block/) در Slack استفاده نمی‌کند.
افزون بر دسترسی عادی پیام `chat:write`، هیچ دامنهٔ OAuth یا پیکربندی اضافی Slack
لازم نیست.

```json
{
  "blocks": [
    {
      "type": "table",
      "caption": "Open pipeline",
      "headers": ["Account", "Stage", "ARR"],
      "rows": [
        ["Acme", "Won", 125000],
        ["Globex", "Review", 82000]
      ],
      "rowHeaderColumnIndex": 0
    }
  ]
}
```

OpenClaw سلول‌های سرآیند و رشته‌ای را به سلول‌های `raw_text` در Slack نگاشت می‌کند. سلول‌های عددی
به `raw_number` نگاشت می‌شوند و مقدار عددی متناهی برای مرتب‌سازی و پالایش بومی حفظ
می‌شود. `rowHeaderColumnIndex`، در صورت وجود، آن ستون با مبنای صفر
را به‌عنوان سرآیند سطر Slack مشخص می‌کند.

محدودیت‌های منتشرشدهٔ `data_table` در Slack پیش از رندر بومی اعمال می‌شوند:

- 1-20 ستون
- 1-100 سطر داده، به‌اضافهٔ سطر سرآیند
- تعداد یکسان سلول در هر سطر
- حداکثر 10,000 نویسه در مجموع همهٔ سلول‌های جدول در یک پیام

تا زمانی که پیام در محدودهٔ کل نویسه‌ها باقی بماند، چندین بلوک جدول معتبر
می‌توانند به‌صورت بومی رندر شوند. جدولی که نتواند در محدودهٔ بومی رندر شود،
به‌جای ازدست‌دادن سطرها یا سلول‌ها به متن کامل و قطعی تبدیل می‌شود. اگر
آن متن از یک پیام Slack فراتر رود، ارسال‌ها و پاسخ‌های اسلش از
قطعه‌های متنی مرتب استفاده می‌کنند. ویرایش جدول به‌جای کوتاه‌کردن بی‌سروصدای
سطرهای پیام موجود، با خطای صریح اندازه ناموفق می‌شود.

هر جدول بومی تولیدشده از ارائه قابل‌حمل، یک بازنمایی متنی سطح‌بالا نیز برای صفحه‌خوان‌ها، اعلان‌ها، آینه‌سازی نشست و
کلاینت‌هایی که نمی‌توانند بلوک را رندر کنند، همراه دارد. مقادیر خام نمودار و جدول در حالت جایگزین دست‌نخورده
می‌مانند، بنابراین داده سلولی مانند `<@U123>` به منشن Slack تبدیل نمی‌شود.
اگر Slack بلوک‌های بومی نمودار یا جدول را با `invalid_blocks` رد کند، OpenClaw
در یک مرحله بازیابی محدود همه بلوک‌های داده بومی را حذف می‌کند، بلوک‌های هم‌سطح معتبر
مانند دکمه‌ها و گزینه‌های انتخاب را نگه می‌دارد و متن کامل و قابل‌مشاهده نمودار
و جدول را با قالب‌بندی Slack غیرفعال ارسال می‌کند. تحویل دستور اسلش
بودجه پنج‌فراخوانی `response_url` در سراسر دستور را پیگیری می‌کند. پیش از هر
دسته پاسخ، طرح کاملی را انتخاب می‌کند که در تعداد فراخوانی‌های باقی‌مانده بگنجد، یا
پیش از ارسال آن دسته با شکست مواجه می‌شود.

فقط بلوک‌های جدول صریح `presentation` به جدول‌های بومی ارتقا می‌یابند.
جدول‌های پایپی Markdown به‌صورت متن تألیف‌شده باقی می‌مانند؛ OpenClaw ساختار جدول
یا نوع سلول‌ها را حدس نمی‌زند. تولیدکنندگان بومی و مورداعتماد فعلی Slack می‌توانند همچنان
بلوک‌های خام را از طریق `channelData.slack.blocks` عبور دهند؛ OpenClaw متن جایگزین را
از سلول‌های خام معتبر `data_table` استخراج می‌کند، درحالی‌که بلوک‌های سفارشی ناقص ممکن است
به زیرنویس خود یا حالت جایگزین عمومی Block Kit تنزل یابند. خروجی قابل‌حمل عامل، CLI
و Plugin باید از `presentation` استفاده کند.

## پاسخ‌های تعاملی

Slack می‌تواند کنترل‌های تعاملی پاسخِ تألیف‌شده توسط عامل را رندر کند، اما این قابلیت به‌طور پیش‌فرض غیرفعال است.
برای خروجی جدید عامل، CLI و Plugin، دکمه‌ها یا بلوک‌های انتخاب مشترک
`presentation` را ترجیح دهید. آن‌ها از همان مسیر تعامل Slack استفاده می‌کنند
و در کانال‌های دیگر نیز به‌درستی تنزل می‌یابند.

فعال‌سازی سراسری:

```json5
{
  channels: {
    slack: {
      capabilities: {
        interactiveReplies: true,
      },
    },
  },
}
```

یا فقط برای یک حساب Slack فعال کنید:

```json5
{
  channels: {
    slack: {
      accounts: {
        ops: {
          capabilities: {
            interactiveReplies: true,
          },
        },
      },
    },
  },
}
```

پس از فعال‌سازی، عامل‌ها همچنان می‌توانند دستورالعمل‌های پاسخ منسوخ و مختص Slack را تولید کنند:

- `[[slack_buttons: Approve:approve, Reject:reject]]`
- `[[slack_select: Choose a target | Canary:canary, Production:production]]`

این دستورالعمل‌ها به Slack Block Kit کامپایل می‌شوند و کلیک‌ها یا انتخاب‌ها را
از طریق مسیر فعلی رویداد تعامل Slack بازمی‌گردانند. آن‌ها را برای پرامپت‌های قدیمی
و راه‌های گریز مختص Slack نگه دارید؛ برای کنترل‌های قابل‌حمل جدید از ارائه مشترک
استفاده کنید.

APIهای کامپایلر دستورالعمل نیز برای کد تولیدکننده جدید منسوخ شده‌اند:

- `compileSlackInteractiveReplies(...)`
- `parseSlackOptionsLine(...)`
- `isSlackInteractiveRepliesEnabled(...)`
- `buildSlackInteractiveBlocks(...)`

برای کنترل‌های جدیدی که در Slack رندر می‌شوند، از محموله‌های `presentation` و `buildSlackPresentationBlocks(...)`
استفاده کنید.

نکته‌ها:

- این یک رابط کاربری قدیمی مختص Slack است. کانال‌های دیگر دستورالعمل‌های Slack Block
  Kit را به سامانه‌های دکمه خود ترجمه نمی‌کنند.
- مقادیر بازخوانی تعاملی، توکن‌های مبهم تولیدشده توسط OpenClaw هستند، نه مقادیر خام تألیف‌شده توسط عامل.
- اگر بلوک‌های تعاملی تولیدشده از محدودیت‌های Slack Block Kit فراتر بروند، OpenClaw به‌جای ارسال محموله بلوک نامعتبر، از پاسخ متنی اصلی به‌عنوان جایگزین استفاده می‌کند.

### ارسال فرم‌های مودال تحت مالکیت Plugin

Pluginهای Slack که یک کنترل‌گر تعاملی ثبت می‌کنند، می‌توانند رویدادهای چرخه‌عمر مودال
`view_submission` و `view_closed` را نیز پیش از آنکه OpenClaw
محموله را برای رویداد سیستمی قابل‌مشاهده برای عامل فشرده کند، دریافت کنند. هنگام بازکردن یک مودال Slack از یکی از این الگوهای
مسیریابی استفاده کنید:

- مقدار `callback_id` را روی `openclaw:<namespace>:<payload>` تنظیم کنید.
- یا `callback_id` موجود را نگه دارید و `pluginInteractiveData:
"<namespace>:<payload>"` را در `private_metadata` مودال قرار دهید.

کنترل‌گر، `ctx.interaction.kind` را به‌صورت `view_submission` یا
`view_closed`، مقدار نرمال‌شده `inputs` و شیء خام کامل `stateValues` را از
Slack دریافت می‌کند. مسیریابی صرفاً بر اساس شناسه بازخوانی برای فراخوانی کنترل‌گر Plugin کافی است؛ هنگامی‌که
مودال باید یک رویداد سیستمی قابل‌مشاهده برای عامل نیز تولید کند، فیلدهای مسیریابی کاربر/نشست
`private_metadata` مودال موجود را اضافه کنید. عامل یک رویداد سیستمی
`Slack interaction: ...` فشرده و ویرایش‌شده دریافت می‌کند. اگر کنترل‌گر
`systemEvent.summary`، `systemEvent.reference` یا `systemEvent.data` را برگرداند، این
فیلدها در آن رویداد فشرده گنجانده می‌شوند تا عامل بتواند بدون مشاهده محموله کامل فرم،
به فضای ذخیره‌سازی تحت مالکیت Plugin ارجاع دهد.

## تأییدهای بومی در Slack

Slack می‌تواند به‌جای استفاده از رابط کاربری وب یا ترمینال به‌عنوان حالت جایگزین، با دکمه‌ها و تعاملات تعاملی به‌عنوان یک کلاینت بومی تأیید عمل کند.

- تأییدهای اجرا و Plugin می‌توانند به‌صورت پرامپت‌های بومی Slack در Block Kit رندر شوند.
- `channels.slack.execApprovals.*` همچنان پیکربندی فعال‌سازی کلاینت بومی تأیید اجرا و مسیریابی پیام خصوصی/کانال است.
- پیام‌های خصوصی تأیید اجرا از `channels.slack.execApprovals.approvers` یا `commands.ownerAllowFrom` استفاده می‌کنند.
- تأییدهای Plugin هنگامی از دکمه‌های بومی Slack استفاده می‌کنند که Slack به‌عنوان کلاینت بومی تأیید برای نشست مبدأ فعال باشد، یا هنگامی‌که `approvals.plugin` به نشست Slack مبدأ یا یک مقصد Slack مسیریابی شود.
- پیام‌های خصوصی تأیید Plugin از تأییدکنندگان Plugin مربوط به Slack در `channels.slack.allowFrom`، مقدار حساب نام‌گذاری‌شده `allowFrom` یا مسیر پیش‌فرض حساب استفاده می‌کنند.
- مجوز تأییدکننده همچنان اعمال می‌شود: تأییدکنندگان صرفاً اجرا نمی‌توانند درخواست‌های Plugin را تأیید کنند، مگر اینکه تأییدکننده Plugin نیز باشند.

این قابلیت از همان سطح مشترک دکمه تأیید در کانال‌های دیگر استفاده می‌کند. هنگامی‌که `interactivity` در تنظیمات برنامه Slack فعال باشد، پرامپت‌های تأیید مستقیماً به‌صورت دکمه‌های Block Kit در گفتگو رندر می‌شوند.
هنگامی‌که این دکمه‌ها وجود دارند، تجربه کاربری اصلی تأیید هستند؛ OpenClaw
فقط زمانی باید دستور دستی `/approve` را درج کند که نتیجه ابزار نشان دهد تأییدهای
گفتگویی در دسترس نیستند یا تأیید دستی تنها مسیر موجود است.

مسیر پیکربندی:

- `channels.slack.execApprovals.enabled`
- `channels.slack.execApprovals.approvers` (اختیاری؛ در صورت امکان از `commands.ownerAllowFrom` به‌عنوان جایگزین استفاده می‌شود)
- `channels.slack.execApprovals.target` (`dm` | `channel` | `both`، پیش‌فرض: `dm`)
- `agentFilter`، `sessionFilter`

Slack هنگامی تأییدهای بومی اجرا را به‌طور خودکار فعال می‌کند که `enabled` تنظیم نشده باشد یا `"auto"` باشد و دست‌کم یک
تأییدکننده اجرا تعیین شود. Slack همچنین می‌تواند تأییدهای بومی Plugin را از طریق این مسیر کلاینت بومی
مدیریت کند، به‌شرط آنکه تأییدکنندگان Plugin مربوط به Slack تعیین شوند و درخواست با فیلترهای کلاینت بومی مطابقت داشته باشد. برای
غیرفعال‌کردن صریح Slack به‌عنوان کلاینت بومی تأیید، `enabled: false` را تنظیم کنید. برای
اجبار تأییدهای بومی در صورت تعیین‌شدن تأییدکنندگان، `enabled: true` را تنظیم کنید. غیرفعال‌کردن تأییدهای اجرای Slack،
تحویل بومی تأیید Plugin در Slack را که از طریق `approvals.plugin` فعال شده است غیرفعال نمی‌کند؛ تحویل تأیید
Plugin به‌جای آن از تأییدکنندگان Plugin مربوط به Slack استفاده می‌کند.

رفتار پیش‌فرض بدون پیکربندی صریح تأیید اجرای Slack:

```json5
{
  commands: {
    ownerAllowFrom: ["slack:U12345678"],
  },
}
```

پیکربندی صریح بومی Slack فقط زمانی لازم است که بخواهید تأییدکنندگان را بازنویسی کنید، فیلتر اضافه کنید یا
تحویل به گفتگوی مبدأ را فعال کنید:

```json5
{
  channels: {
    slack: {
      execApprovals: {
        enabled: true,
        approvers: ["U12345678"],
        target: "both",
      },
    },
  },
}
```

هدایت مشترک `approvals.exec` مستقل است. فقط زمانی از آن استفاده کنید که پرامپت‌های تأیید اجرا باید
به گفتگوهای دیگر یا مقصدهای صریح خارج از باند نیز مسیریابی شوند. هدایت مشترک `approvals.plugin` نیز
مستقل است؛ تحویل بومی Slack فقط زمانی آن حالت جایگزین را سرکوب می‌کند که Slack بتواند درخواست تأیید
Plugin را به‌صورت بومی مدیریت کند.

`/approve` در همان گفتگو، در کانال‌ها و پیام‌های خصوصی Slack که از قبل از دستورها پشتیبانی می‌کنند نیز کار می‌کند. برای مدل کامل هدایت تأیید، به [تأییدهای اجرا](/fa/tools/exec-approvals) مراجعه کنید.

## رویدادها و رفتار عملیاتی

- ویرایش‌ها/حذف‌های پیام به رویدادهای سیستمی نگاشت می‌شوند.
- انتشارهای رشته‌ای (پاسخ‌های رشته با گزینه "Also send to channel") به‌عنوان پیام‌های عادی کاربر پردازش می‌شوند.
- رویدادهای افزودن/حذف واکنش به رویدادهای سیستمی نگاشت می‌شوند.
- رویدادهای پیوستن/ترک عضو، ایجاد/تغییر نام کانال و افزودن/حذف سنجاق به رویدادهای سیستمی نگاشت می‌شوند.
- نظرسنجی اختیاری وضعیت حضور می‌تواند گذار `away` به `active` یک مشارکت‌کننده انسانی مشاهده‌شده را به تازه‌ترین نشست واجد شرایط و فعال Slack آن مشارکت‌کننده نگاشت کند. این قابلیت به‌طور پیش‌فرض غیرفعال است.
- `channel_id_changed` می‌تواند هنگامی‌که `configWrites` فعال است، کلیدهای پیکربندی کانال را مهاجرت دهد.
- فراداده موضوع/هدف کانال به‌عنوان زمینه نامطمئن در نظر گرفته می‌شود و می‌تواند به زمینه مسیریابی تزریق شود.
- موجودیت‌های `app_context` در Agent View به‌ترتیب ارتباط Slack اعتبارسنجی می‌شوند و فقط به‌صورت زمینه ساختاریافته نامطمئن ارائه می‌شوند؛ حذف زمینه به‌جای استفاده مجدد از موجودیت‌های کهنه، نوبت را پاک می‌کند.
- آغازگر رشته و بذرگذاری زمینه اولیه تاریخچه رشته، در صورت کاربرد بر اساس فهرست‌های مجاز فرستنده پیکربندی‌شده فیلتر می‌شوند.
- کنش‌های بلوک، میان‌برها و تعاملات مودال، رویدادهای سیستمی ساختاریافته `Slack interaction: ...` را با فیلدهای غنی محموله تولید می‌کنند:
  - کنش‌های بلوک: مقادیر انتخاب‌شده، برچسب‌ها، مقادیر انتخاب‌گر و فراداده `workflow_*`
  - میان‌برهای سراسری: فراداده بازخوانی و کنشگر، با مسیریابی به نشست مستقیم کنشگر
  - میان‌برهای پیام: زمینه بازخوانی، کنشگر، کانال، رشته و پیام انتخاب‌شده
  - رویدادهای مودال `view_submission` و `view_closed` با فراداده کانال مسیریابی‌شده و ورودی‌های فرم

میان‌برهای سراسری یا پیام را در پیکربندی برنامه Slack تعریف کنید و از هر شناسه بازخوانی غیرخالی استفاده کنید. OpenClaw محموله‌های میان‌بر منطبق را تأیید دریافت می‌کند، همان سیاست فرستنده پیام خصوصی/کانال را که برای سایر تعاملات Slack به‌کار می‌رود اعمال می‌کند و رویداد پاک‌سازی‌شده را برای نشست عامل مسیریابی‌شده در صف قرار می‌دهد. شناسه‌های محرک و URLهای پاسخ از زمینه عامل حذف می‌شوند.

### رویدادهای حضور

Slack تغییرات حضور را از طریق Events API یا Socket Mode ارسال نمی‌کند. در عوض، OpenClaw می‌تواند برای مشارکت‌کنندگان انسانی که پیام‌هایشان بررسی‌های عادی دسترسی و مسیریابی Slack را گذرانده است، [`users.getPresence`](https://docs.slack.dev/reference/methods/users.getPresence/) را نظرسنجی کند.

```json5
{
  channels: {
    slack: {
      presenceEvents: { mode: "auto" },
      channels: {
        C0123456789: { presenceEvents: { mode: "on" } },
        C0987654321: { presenceEvents: { mode: "off" } },
      },
    },
  },
}
```

- `off` (پیش‌فرض): بدون زمان‌سنج حضور یا فراخوانی‌های API مربوط به Slack.
- `auto`: پیام‌های خصوصی، MPIMها و رشته‌های Slack فعال در 24 ساعت گذشته را با حداکثر 8 مشارکت‌کننده انسانی مشاهده‌شده پایش می‌کند. نشست‌های سطح‌بالای کانال مستثنا هستند.
- `on`: همان گفتگوها را بدون سقف مشارکت‌کننده پایش می‌کند و نشست‌های سطح‌بالای کانال را نیز دربر می‌گیرد. برای اجبار یا سرکوب یک کانال، از بازنویسی مختص آن کانال استفاده کنید.

OpenClaw در هر حساب Slack حداکثر 45 کاربر یکتا را در دقیقه نظرسنجی می‌کند، نتیجه نخست را بدون بیدارکردن عامل به‌عنوان مقدار اولیه ثبت می‌کند و فقط در صورت مشاهده گذار `away` به `active` عامل را بیدار می‌کند. یک دوره انتظار پایدار 8 ساعته برای هر حساب Slack و کاربر اعمال می‌شود، حتی اگر آن شخص در چند رشته مشارکت داشته باشد. رویداد فقط به تازه‌ترین گفتگوی واجد شرایط و فعال آن شخص مسیریابی می‌شود و به عامل می‌گوید پیش از تصمیم‌گیری درباره ارسال یک احوالپرسی کوتاه، حافظه/ویکی و زمینه منطقه زمانی شناخته‌شده را بررسی کند. عامل می‌تواند ساکت بماند.

توکن ربات به `users:read` نیاز دارد که از قبل در مانیفست پیشنهادی گنجانده شده است. رویدادهای حضور برای نصب‌های سراسری سازمانی Enterprise Grid در دسترس نیستند.

## مرجع پیکربندی

مرجع اصلی: [مرجع پیکربندی - Slack](/fa/gateway/config-channels#slack).

<Accordion title="فیلدهای پراهمیت Slack">

- حالت/احراز هویت: `identity`، `mode`، `enterpriseOrgInstall`، `botToken`، `appToken`، `userToken`، `signingSecret`، `webhookPath`، `accounts.*`
- دسترسی پیام مستقیم: `dm.enabled`، `dmPolicy`، `allowFrom` (قدیمی: `dm.policy`، `dm.allowFrom`)، `dm.groupEnabled`، `dm.groupChannels`
- کلید سازگاری: `dangerouslyAllowNameMatching` (برای شرایط اضطراری؛ مگر در صورت نیاز خاموش نگه دارید)
- دسترسی کانال: `groupPolicy`، `channels.*`، `channels.*.users`، `channels.*.requireMention`، `implicitMentions.*`
- رشته‌ها/تاریخچه: `replyToMode`، `replyToModeByChatType`، `thread.*`، `historyLimit`، `dmHistoryLimit`، `dms.*.historyLimit`
- بیدارباش‌های حضور: `presenceEvents.mode`، `channels.*.presenceEvents.mode` (`off|auto|on`؛ پیش‌فرض `off`)
- تحویل: `textChunkLimit`، `streaming.chunkMode`، `mediaMaxMb`، `streaming`، `streaming.nativeTransport`، `streaming.preview.toolProgress`
- پیش‌نمایش‌ها: `unfurlLinks` (پیش‌فرض: `false`)، `unfurlMedia` برای کنترل پیش‌نمایش پیوند/رسانهٔ `chat.postMessage`؛ برای فعال‌سازی دوبارهٔ پیش‌نمایش پیوندها، `unfurlLinks: true` را تنظیم کنید
- عملیات/قابلیت‌ها: `configWrites`، `commands.native`، `slashCommand.*`، `actions.*`، `userToken`، `userTokenReadOnly`

</Accordion>

## عیب‌یابی

<AccordionGroup>
  <Accordion title="در کانال‌ها پاسخی دریافت نمی‌شود">
    به‌ترتیب بررسی کنید:

    - `groupPolicy`
    - فهرست مجاز کانال (`channels.slack.channels`) — **کلیدها باید شناسهٔ کانال باشند** (`C12345678`)، نه نام‌ها (`#channel-name`). کلیدهای مبتنی بر نام تحت `groupPolicy: "allowlist"` بی‌سروصدا ناموفق می‌شوند، زیرا مسیریابی کانال به‌طور پیش‌فرض ابتدا بر اساس شناسه انجام می‌شود. برای یافتن شناسه: در Slack روی کانال راست‌کلیک کنید ← **Copy link** — مقدار `C...` در انتهای URL، شناسهٔ کانال است.
    - `requireMention`
    - فهرست مجاز `users` برای هر کانال
    - `messages.groupChat.visibleReplies`: درخواست‌های عادی گروه/کانال به‌طور پیش‌فرض از `"automatic"` استفاده می‌کنند. اگر `"message_tool"` را فعال کرده‌اید و گزارش‌ها متن دستیار را بدون فراخوانی `message(action=send)` نشان می‌دهند، مدل مسیر قابل‌مشاهدهٔ ابزار پیام را از دست داده است. در این حالت متن نهایی خصوصی می‌ماند؛ گزارش تفصیلی Gateway را برای فرادادهٔ محمولهٔ سرکوب‌شده بررسی کنید، یا اگر می‌خواهید هر پاسخ نهایی عادی دستیار از طریق مسیر قدیمی ارسال شود، آن را روی `"automatic"` تنظیم کنید.
    - `messages.groupChat.unmentionedInbound`: اگر مقدار آن `"room_event"` باشد، گفت‌وگوهای بدون اشاره در کانال مجاز، زمینهٔ محیطی محسوب می‌شوند و مگر آنکه عامل ابزار `message` را فراخوانی کند، بی‌پاسخ می‌مانند. به [رویدادهای محیطی اتاق](/fa/channels/ambient-room-events) مراجعه کنید.

```json5
{
  messages: {
    groupChat: {
      visibleReplies: "automatic",
    },
  },
}
```

    فرمان‌های مفید:

```bash
openclaw channels status --probe
openclaw logs --follow
openclaw doctor
```

  </Accordion>

  <Accordion title="پیام‌های مستقیم نادیده گرفته می‌شوند">
    بررسی کنید:

    - `channels.slack.dm.enabled`
    - `channels.slack.dmPolicy` (یا `channels.slack.dm.policy` قدیمی)
    - تأییدهای جفت‌سازی / ورودی‌های فهرست مجاز (`dmPolicy: "open"` همچنان به `channels.slack.allowFrom: ["*"]` نیاز دارد)
    - پیام‌های مستقیم گروهی از مدیریت MPIM استفاده می‌کنند؛ `channels.slack.dm.groupEnabled` را فعال کنید و در صورت پیکربندی، MPIM را در `channels.slack.dm.groupChannels` بگنجانید
    - رویدادهای پیام مستقیم Slack Assistant: گزارش‌های تفصیلی که به `drop message_changed` اشاره می‌کنند
      معمولاً به این معنا هستند که Slack یک رویداد ویرایش‌شدهٔ رشتهٔ Assistant را بدون
      فرستندهٔ انسانی قابل‌بازیابی در فرادادهٔ پیام ارسال کرده است

```bash
openclaw pairing list slack
```

  </Accordion>

  <Accordion title="حالت Socket متصل نمی‌شود">
    توکن‌های ربات و برنامه و فعال‌بودن Socket Mode را در تنظیمات برنامهٔ Slack اعتبارسنجی کنید.
    App-Level Token به `connections:write` نیاز دارد و توکن ربات Bot User OAuth Token
    باید متعلق به همان برنامه/فضای کاری Slack باشد که توکن برنامه به آن تعلق دارد.

    اگر `openclaw channels status --probe --json` مقدار `botTokenStatus` یا
    `appTokenStatus: "configured_unavailable"` را نشان دهد، حساب Slack
    پیکربندی شده است، اما محیط اجرای کنونی نتوانسته مقدار پشتیبانی‌شده با SecretRef را
    برطرف کند.

    گزارش‌هایی مانند `slack socket mode failed to start; retry ...` خطاهای
    قابل‌بازیابی هنگام راه‌اندازی هستند. در مقابل، محدوده‌های دسترسی مفقود، توکن‌های لغوشده و احراز هویت نامعتبر
    فوراً شکست می‌خورند. گزارش `slack token mismatch ...` یعنی ظاهراً توکن ربات و توکن برنامه
    به برنامه‌های متفاوت Slack تعلق دارند؛ اعتبارنامه‌های برنامهٔ Slack را اصلاح کنید.

  </Accordion>

  <Accordion title="حالت HTTP رویدادها را دریافت نمی‌کند">
    اعتبارسنجی کنید:

    - راز امضا
    - مسیر Webhook
    - نشانی‌های درخواست Slack (رویدادها + تعامل‌پذیری + فرمان‌های اسلش)
    - `webhookPath` یکتا برای هر حساب HTTP
    - نشانی عمومی، TLS را خاتمه می‌دهد و درخواست‌ها را به مسیر Gateway هدایت می‌کند
    - مسیر `request_url` برنامهٔ Slack دقیقاً با `channels.slack.webhookPath` مطابقت دارد (پیش‌فرض `/slack/events`)

    اگر `signingSecretStatus: "configured_unavailable"` در تصویرهای لحظه‌ای حساب
    ظاهر شود، حساب HTTP پیکربندی شده است، اما محیط اجرای کنونی نتوانسته راز امضای
    پشتیبانی‌شده با SecretRef را برطرف کند.

    تکرار گزارش `slack: webhook path ... already registered` یعنی دو حساب HTTP
    از `webhookPath` یکسان استفاده می‌کنند؛ به هر حساب مسیری مجزا بدهید.

  </Accordion>

  <Accordion title="فرمان‌های بومی/اسلش اجرا نمی‌شوند">
    بررسی کنید که کدام مورد مدنظرتان بوده است:

    - حالت فرمان بومی (`channels.slack.commands.native: true`) با فرمان‌های اسلش منطبق ثبت‌شده در Slack
    - یا حالت تک‌فرمان اسلش (`channels.slack.slashCommand.enabled: true`)

    Slack فرمان‌های اسلش را به‌طور خودکار ایجاد یا حذف نمی‌کند. `commands.native: "auto"` فرمان‌های بومی Slack را فعال نمی‌کند؛ از `true` استفاده کنید و فرمان‌های منطبق را در برنامهٔ Slack بسازید. در حالت HTTP، هر فرمان اسلش Slack باید URL مربوط به Gateway را شامل شود. در Socket Mode، محموله‌های فرمان از طریق websocket می‌رسند و Slack، `slash_commands[].url` را نادیده می‌گیرد.

    همچنین `commands.useAccessGroups`، مجوز پیام مستقیم، فهرست‌های مجاز کانال،
    و فهرست‌های مجاز `users` برای هر کانال را بررسی کنید. Slack برای
    فرستندگان مسدودشدهٔ فرمان اسلش، خطاهای موقت برمی‌گرداند، از جمله:

    - `This channel is not allowed.`
    - `You are not authorized to use this command here.`

  </Accordion>
</AccordionGroup>

## مرجع رسانهٔ پیوست

هنگامی که دانلود فایل‌های Slack موفق باشد و محدودیت‌های اندازه اجازه دهند، Slack می‌تواند رسانهٔ دانلودشده را به نوبت عامل پیوست کند. کلیپ‌های صوتی را می‌توان رونویسی کرد، فایل‌های تصویری می‌توانند از مسیر درک رسانه عبور کنند یا مستقیماً به مدل پاسخ‌دهی دارای قابلیت بینایی ارسال شوند، و سایر فایل‌ها به‌عنوان زمینهٔ فایل قابل‌دانلود در دسترس می‌مانند.

### انواع رسانهٔ پشتیبانی‌شده

| نوع رسانه                     | منبع               | رفتار کنونی                                                                  | یادداشت‌ها                                                                     |
| ------------------------------ | -------------------- | --------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| کلیپ‌های صوتی Slack              | URL فایل Slack       | دانلود و از طریق رونویسی صوتی مشترک مسیریابی می‌شوند                          | به `files:read` و یک مدل یا CLI کارآمد `tools.media.audio` نیاز دارد      |
| تصاویر JPEG / PNG / GIF / WebP | URL فایل Slack       | برای پردازش دارای قابلیت بینایی دانلود و به نوبت پیوست می‌شوند                   | سقف هر فایل: `channels.slack.mediaMaxMb` (پیش‌فرض 20 MB)                 |
| فایل‌های PDF                      | URL فایل Slack       | دانلود و به‌عنوان زمینهٔ فایل برای ابزارهایی مانند `download-file` یا `pdf` ارائه می‌شوند | ورودی Slack، فایل‌های PDF را به‌طور خودکار به ورودی بینایی تصویری تبدیل نمی‌کند |
| سایر فایل‌ها                    | URL فایل Slack       | در صورت امکان دانلود و به‌عنوان زمینهٔ فایل ارائه می‌شوند                              | فایل‌های دودویی به‌عنوان ورودی تصویر در نظر گرفته نمی‌شوند                               |
| پاسخ‌های رشته                   | فایل‌های آغازگر رشته | وقتی پاسخ رسانهٔ مستقیمی ندارد، فایل‌های پیام ریشه را می‌توان به‌عنوان زمینه بازیابی کرد  | آغازگرهای فقط‌فایلی از جای‌نگهدار پیوست استفاده می‌کنند                          |
| پیام‌های چندفایلی            | چند فایل Slack | هر فایل به‌طور مستقل ارزیابی می‌شود                                              | پردازش Slack به هشت فایل در هر پیام محدود است                     |

### پایپ‌لاین ورودی

وقتی یک پیام Slack دارای فایل‌های پیوست می‌رسد:

1. OpenClaw فایل را با استفاده از توکن ربات از URL خصوصی Slack دانلود می‌کند.
2. در صورت موفقیت، فایل در مخزن رسانه نوشته می‌شود.
3. مسیرهای رسانهٔ دانلودشده و انواع محتوا به زمینهٔ ورودی افزوده می‌شوند.
4. کلیپ‌های صوتی به پایپ‌لاین رونویسی مشترک هدایت می‌شوند؛ مسیرهای مدل/ابزار دارای قابلیت تصویر می‌توانند از پیوست‌های تصویری همان زمینه استفاده کنند.
5. سایر فایل‌ها به‌صورت فرادادهٔ فایل یا ارجاع رسانه برای ابزارهایی که توانایی پردازش آن‌ها را دارند، در دسترس می‌مانند.

### به‌ارث‌بردن پیوست آغازگر رشته

هنگامی که پیامی در یک رشته می‌رسد (والد `thread_ts` دارد):

- اگر خود پاسخ رسانهٔ مستقیمی نداشته باشد و پیام ریشهٔ گنجانده‌شده فایل داشته باشد، Slack می‌تواند فایل‌های ریشه را به‌عنوان زمینهٔ آغازگر رشته بازیابی کند.
- فایل‌های ریشه فقط هنگام مقداردهی اولیهٔ یک نشست رشتهٔ جدید یا بازنشانی‌شده بازیابی می‌شوند. پاسخ‌های متنی بعدی از زمینهٔ نشست موجود استفاده می‌کنند و فایل‌های ریشه را دوباره به‌عنوان رسانهٔ تازه پیوست نمی‌کنند.
- پیوست‌های مستقیم پاسخ بر پیوست‌های پیام ریشه اولویت دارند.
- پیام ریشه‌ای که فقط فایل دارد و فاقد متن است، با یک جای‌نگهدار پیوست نمایش داده می‌شود تا سازوکار جایگزین همچنان بتواند فایل‌های آن را شامل شود.

### مدیریت چند پیوست

هنگامی که یک پیام Slack شامل چند فایل پیوست است:

- هر پیوست به‌طور مستقل از طریق پایپ‌لاین رسانه پردازش می‌شود.
- ارجاعات رسانهٔ دانلودشده در زمینهٔ پیام تجمیع می‌شوند.
- ترتیب پردازش از ترتیب فایل‌های Slack در محمولهٔ رویداد پیروی می‌کند.
- شکست دانلود یک پیوست، مانع پردازش سایر پیوست‌ها نمی‌شود.

### محدودیت‌های اندازه، دانلود و مدل

- **سقف اندازه**: پیش‌فرض 20 MB برای هر فایل. از طریق `channels.slack.mediaMaxMb` قابل‌پیکربندی است.
- **سقف رونویسی صوتی**: مقدار `maxBytes` در ورودی انتخاب‌شدهٔ دارای قابلیت صوتی `tools.media.models[]`، هنگامی که فایل دانلودشده به ارائه‌دهندهٔ رونویسی یا CLI ارسال می‌شود نیز اعمال می‌شود.
- **شکست‌های دانلود**: فایل‌هایی که Slack نمی‌تواند ارائه کند، URLهای منقضی‌شده، فایل‌های غیرقابل‌دسترسی، فایل‌های بیش‌ازحد بزرگ و پاسخ‌های HTML احراز هویت/ورود Slack به‌جای آنکه به‌عنوان قالب‌های پشتیبانی‌نشده گزارش شوند، نادیده گرفته می‌شوند.
- **مدل بینایی**: تحلیل تصویر از مدل پاسخ‌دهی فعال، در صورت پشتیبانی آن از بینایی، یا از مدل تصویر پیکربندی‌شده در `agents.defaults.imageModel` استفاده می‌کند.

### محدودیت‌های شناخته‌شده

| سناریو                                      | رفتار فعلی                                                                   | راه‌حل موقت                                                                    |
| --------------------------------------------- | ---------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
| URL منقضی‌شدهٔ فایل Slack                        | فایل نادیده گرفته می‌شود؛ هیچ خطایی نمایش داده نمی‌شود                                                       | فایل را دوباره در Slack بارگذاری کنید                                                   |
| رونویسی صوتی در دسترس نیست               | کلیپ پیوست باقی می‌ماند، اما هیچ رونوشتی تولید نمی‌شود                                | `tools.media.audio` را پیکربندی کنید یا یک CLI محلی پشتیبانی‌شده برای رونویسی نصب کنید  |
| کلیپ بدون زیرنویس از دروازهٔ اشاره عبور نمی‌کند | پس از رونویسی گمانه‌زنانهٔ خصوصی حذف می‌شود؛ رونوشت و فایل دانلودشده کنار گذاشته می‌شوند | یک الگوی اشاره با نام گفتاری پیکربندی کنید، یک اشارهٔ نوشتاری به بات اضافه کنید، یا از پیام خصوصی استفاده کنید |
| مدل بینایی پیکربندی نشده است                   | پیوست‌های تصویر به‌عنوان ارجاع‌های رسانه‌ای ذخیره می‌شوند، اما به‌عنوان تصویر تحلیل نمی‌شوند       | `agents.defaults.imageModel` را پیکربندی کنید یا از یک مدل پاسخ‌گویی دارای قابلیت بینایی استفاده کنید    |
| تصاویر بسیار بزرگ (> 20 MB به‌طور پیش‌فرض)        | بر اساس سقف اندازه نادیده گرفته می‌شوند                                                               | اگر Slack اجازه می‌دهد، `channels.slack.mediaMaxMb` را افزایش دهید                          |
| پیوست‌های بازارسال‌شده/اشتراک‌گذاری‌شده                  | متن و رسانه‌های تصویری/فایلی میزبانی‌شده در Slack به‌صورت بهترین تلاش پردازش می‌شوند                             | مستقیماً در رشتهٔ OpenClaw دوباره به اشتراک بگذارید                                      |
| پیوست‌های PDF                               | به‌عنوان زمینهٔ فایل/رسانه ذخیره می‌شوند و به‌طور خودکار برای بینایی تصویری مسیریابی نمی‌شوند        | برای فرادادهٔ فایل از `download-file` یا برای تحلیل PDF از ابزار `pdf` استفاده کنید      |

### مستندات مرتبط

- [پایپ‌لاین درک رسانه](/fa/nodes/media-understanding)
- [یادداشت‌های صوتی و صوت](/fa/nodes/audio)
- [ابزار PDF](/fa/tools/pdf)

## مرتبط

<CardGroup cols={2}>
  <Card title="جفت‌سازی" icon="link" href="/fa/channels/pairing">
    یک کاربر Slack را با Gateway جفت کنید.
  </Card>
  <Card title="گروه‌ها" icon="users" href="/fa/channels/groups">
    رفتار کانال و پیام خصوصی گروهی.
  </Card>
  <Card title="مسیریابی کانال" icon="route" href="/fa/channels/channel-routing">
    پیام‌های ورودی را به عامل‌ها مسیریابی کنید.
  </Card>
  <Card title="امنیت" icon="shield" href="/fa/gateway/security">
    مدل تهدید و مقاوم‌سازی.
  </Card>
  <Card title="پیکربندی" icon="sliders" href="/fa/gateway/configuration">
    چیدمان پیکربندی و ترتیب تقدم.
  </Card>
  <Card title="دستورهای اسلش" icon="terminal" href="/fa/tools/slash-commands">
    فهرست دستورها و رفتار آن‌ها.
  </Card>
</CardGroup>
