---
read_when:
    - پیکربندی یک Plugin کانال (احراز هویت، کنترل دسترسی، چندحسابی)
    - عیب‌یابی کلیدهای پیکربندی هر کانال
    - ممیزی سیاست پیام مستقیم، سیاست گروه یا محدودسازی بر اساس اشاره‌کردن
summary: 'پیکربندی کانال: کنترل دسترسی، جفت‌سازی و کلیدهای مختص هر کانال در Slack، Discord، Telegram، WhatsApp، Matrix، iMessage و موارد دیگر'
title: پیکربندی — کانال‌ها
x-i18n:
    generated_at: "2026-07-27T16:31:47Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e346648287d275d84a9c082a3bb13edaee751d53546d8231dcf1525bf9adafc2
    source_path: gateway/config-channels.md
    workflow: 16
---

کلیدهای پیکربندی مختص هر کانال زیر `channels.*`: دسترسی پیام خصوصی و گروه، راه‌اندازی‌های چندحسابی، دروازه‌گذاری اشاره، و کلیدهای مختص هر کانال برای Slack، Discord، Telegram، WhatsApp، Matrix، iMessage و دیگر Pluginهای کانال.

برای عامل‌ها، ابزارها، زمان اجرای Gateway و دیگر کلیدهای سطح‌بالا، به [مرجع پیکربندی](/fa/gateway/configuration-reference) مراجعه کنید.

## کانال‌ها

هر کانال با وجود بخش پیکربندی‌اش به‌طور خودکار راه‌اندازی می‌شود (مگر اینکه `enabled: false`). Telegram و iMessage درون بستهٔ اصلی `openclaw` ارائه می‌شوند. دیگر کانال‌های رسمی (Discord، Slack، WhatsApp، Matrix، Microsoft Teams، IRC، Google Chat، Signal، Mattermost و موارد بیشتر) به‌صورت Pluginهای جداگانه با `openclaw plugins install <spec>` نصب می‌شوند؛ برای فهرست کامل و مشخصات نصب، به [کانال‌ها](/fa/channels) مراجعه کنید.

### دسترسی پیام خصوصی و گروه

همهٔ کانال‌ها از خط‌مشی‌های پیام خصوصی و گروه پشتیبانی می‌کنند:

| خط‌مشی پیام خصوصی           | رفتار                                                        |
| ------------------- | --------------------------------------------------------------- |
| `pairing` (پیش‌فرض) | فرستندگان ناشناس یک کد جفت‌سازی یک‌بارمصرف دریافت می‌کنند؛ مالک باید تأیید کند |
| `allowlist`         | فقط فرستندگان موجود در `allowFrom` (یا مخزن مجاز جفت‌شده)             |
| `open`              | اجازه به همهٔ پیام‌های خصوصی ورودی (نیازمند `allowFrom: ["*"]`)             |
| `disabled`          | نادیده‌گرفتن همهٔ پیام‌های خصوصی ورودی                                          |

| خط‌مشی گروه          | رفتار                                               |
| --------------------- | ------------------------------------------------------ |
| `allowlist` (پیش‌فرض) | فقط گروه‌های منطبق با فهرست مجاز پیکربندی‌شده          |
| `open`                | نادیده‌گرفتن فهرست‌های مجاز گروه (دروازه‌گذاری اشاره همچنان اعمال می‌شود) |
| `disabled`            | مسدودکردن همهٔ پیام‌های گروه/اتاق                          |

<Note>
وقتی `groupPolicy` یک ارائه‌دهنده تنظیم نشده باشد، `channels.defaults.groupPolicy` مقدار پیش‌فرض را تعیین می‌کند.
کدهای جفت‌سازی پس از 1 ساعت منقضی می‌شوند. درخواست‌های جفت‌سازی در انتظار به **3 مورد برای هر حساب** محدود می‌شوند (با دامنه‌بندی بر اساس کانال و شناسهٔ حساب).
اگر بلوک یک ارائه‌دهنده کاملاً موجود نباشد (`channels.<provider>` وجود نداشته باشد)، خط‌مشی گروه در زمان اجرا با یک هشدار هنگام راه‌اندازی به `allowlist` (بسته در حالت خطا) بازمی‌گردد.
</Note>

### بازنویسی مدل کانال

از `channels.modelByChannel` برای سنجاق‌کردن شناسه‌های کانال یا همتایان پیام خصوصی مشخص به یک مدل استفاده کنید. مقادیر، `provider/model` یا نام‌های مستعار مدل پیکربندی‌شده را می‌پذیرند. نگاشت کانال فقط زمانی اعمال می‌شود که نشست از قبل بازنویسی مدل فعالی نداشته باشد (برای مثال، موردی که از طریق `/model` تنظیم شده است).

برای گفت‌وگوهای گروهی/رشته‌ای، کلیدها شناسه‌های گروه، شناسه‌های موضوع یا نام‌های کانال مختص همان کانال هستند. برای گفت‌وگوهای پیام خصوصی (DM)، کلیدها شناسه‌های همتا هستند که از هویت فرستندهٔ کانال (`nativeDirectUserId`، `origin.from`، `origin.to`، `OriginatingTo`، `From` یا `SenderId`) به‌دست می‌آیند. شکل دقیق کلید به کانال بستگی دارد:

| کانال  | شکل کلید پیام خصوصی         | نمونه                                      |
| -------- | ------------------- | -------------------------------------------- |
| Discord  | شناسهٔ خام کاربر         | `987654321`                                  |
| Feishu   | `feishu:ou_...`     | `feishu:ou_a8b6cab7e945387de5f253775d9b4d85` |
| Matrix   | شناسهٔ کاربر Matrix      | `@user:matrix.org`                           |
| Slack    | `user:U...`         | `user:U12345`                                |
| Telegram | شناسهٔ خام کاربر         | `123456789`                                  |
| WhatsApp | شماره تلفن یا JID | `15551234567`                                |

```json5
{
  channels: {
    modelByChannel: {
      discord: {
        "123456789012345678": "anthropic/claude-opus-4-6",
      },
      slack: {
        C1234567890: "openai/gpt-5.6-sol",
        "user:U12345": "openai/gpt-5.4-mini",
      },
      telegram: {
        "-1001234567890": "openai/gpt-5.4-mini",
        "-1001234567890:topic:99": "anthropic/claude-sonnet-4-6",
        "123456789": "openai/gpt-4.1",
      },
    },
  },
}
```

کلیدهای مختص پیام خصوصی فقط در گفت‌وگوهای پیام خصوصی تطبیق داده می‌شوند؛ آن‌ها بر مسیریابی گروه/رشته تأثیری ندارند.

### پیش‌فرض‌های کانال و Heartbeat

از `channels.defaults` برای رفتار مشترک خط‌مشی گروه، اشارهٔ ضمنی و Heartbeat میان ارائه‌دهندگان استفاده کنید:

```json5
{
  channels: {
    defaults: {
      groupPolicy: "allowlist", // open | allowlist | disabled
      contextVisibility: "all", // all | allowlist | allowlist_quote
      implicitMentions: {
        replyToBot: true,
        quotedBot: true,
        threadParticipation: true,
      },
      heartbeat: {
        showOk: false,
        showAlerts: true,
        useIndicator: true,
      },
    },
  },
}
```

- `channels.defaults.groupPolicy`: خط‌مشی گروه جایگزین هنگامی که `groupPolicy` در سطح ارائه‌دهنده تنظیم نشده باشد.
- `channels.defaults.contextVisibility`: حالت پیش‌فرض مشاهده‌پذیری زمینهٔ تکمیلی برای همهٔ کانال‌ها. مقادیر: `all` (پیش‌فرض، شامل همهٔ زمینه‌های نقل‌قول/رشته/تاریخچه)، `allowlist` (فقط شامل زمینهٔ فرستندگان موجود در فهرست مجاز)، `allowlist_quote` (همانند فهرست مجاز، اما با حفظ زمینهٔ صریح نقل‌قول/پاسخ). بازنویسی مختص هر کانال: `channels.<channel>.contextVisibility`.
- `channels.defaults.implicitMentions`: کنترل می‌کند کدام واقعیت‌های ورودیِ پشتیبانی‌شده به‌عنوان اشاره محسوب شوند. هر یک از `replyToBot`، `quotedBot` و `threadParticipation` به‌طور پیش‌فرض `true` هستند و رفتار فعلی را حفظ می‌کنند. برای هر کانال با `channels.<channel>.implicitMentions` یا برای هر حساب با `channels.<channel>.accounts.<id>.implicitMentions` بازنویسی کنید؛ هر پرچم به‌طور مستقل با ترتیب حساب -> کانال -> پیش‌فرض‌ها حل می‌شود. نام‌ها مثبت هستند: برای جلوگیری از اینکه آن واقعیت دروازه‌گذاری اشاره را دور بزند، پرچم را روی `false` تنظیم کنید. اشاره‌های صریح بومی همیشه مجازند و اگر کانال آن واقعیت را تولید نکند، پرچم اثری ندارد. برای ماتریس فعلی تولیدکنندگان، به [دروازه‌گذاری اشاره](/fa/channels/groups#mention-gating-default) مراجعه کنید. این تنظیمات حالت‌های پاسخ/رشتهٔ خروجی یا رسیدگی به فرمان‌های مجاز را تغییر نمی‌دهند.
- `channels.defaults.heartbeat.showOk`: وضعیت‌های سالم کانال را در خروجی Heartbeat لحاظ می‌کند (پیش‌فرض `false`).
- `channels.defaults.heartbeat.showAlerts`: وضعیت‌های افت‌کرده/خطا را در خروجی Heartbeat لحاظ می‌کند (پیش‌فرض `true`).
- `channels.defaults.heartbeat.useIndicator`: خروجی فشردهٔ Heartbeat به‌سبک نشانگر را رندر می‌کند (پیش‌فرض `true`).

### WhatsApp

WhatsApp از طریق کانال وب Gateway (Baileys Web) اجرا می‌شود. وقتی یک نشست پیوندخورده وجود داشته باشد، به‌طور خودکار راه‌اندازی می‌شود.

```json5
{
  web: {
    enabled: true,
  },
  channels: {
    whatsapp: {
      dmPolicy: "pairing", // pairing | allowlist | open | disabled
      allowFrom: ["+15555550123", "+447700900123"],
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" }, // length | newline
      mediaMaxMb: 50,
      sendReadReceipts: true, // blue ticks (false in self-chat mode)
      groups: {
        "*": { requireMention: true },
      },
      groupPolicy: "allowlist",
      groupAllowFrom: ["+15551234567"],
    },
  },
}
```

- ورودی‌های سطح‌بالای `bindings[]` همراه با `type: "acp"` اتصال‌های پایدار ACP را برای پیام‌های خصوصی و گروه‌های WhatsApp پیکربندی می‌کنند. در `match.peer.id` از یک شمارهٔ مستقیم E.164 یا JID گروه WhatsApp استفاده کنید. معنای فیلدها در [عامل‌های ACP](/fa/tools/acp-agents#persistent-channel-bindings) مشترک است.

<Accordion title="WhatsApp چندحسابی">

```json5
{
  channels: {
    whatsapp: {
      accounts: {
        default: {},
        personal: {},
        biz: {
          // authDir: "~/.openclaw/credentials/whatsapp/biz",
        },
      },
    },
  },
}
```

- فرمان‌های خروجی در صورت وجود، به‌طور پیش‌فرض از حساب `default` استفاده می‌کنند؛ در غیر این صورت، نخستین شناسهٔ حساب پیکربندی‌شده (مرتب‌شده).
- `channels.whatsapp.defaultAccount` اختیاری، هنگامی که با یک شناسهٔ حساب پیکربندی‌شده مطابقت داشته باشد، انتخاب حساب پیش‌فرض جایگزین را بازنویسی می‌کند.
- دایرکتوری احراز هویت قدیمی Baileys برای تک‌حساب، توسط `openclaw doctor` به `whatsapp/default` مهاجرت می‌کند.
- بازنویسی‌های مختص هر حساب: `channels.whatsapp.accounts.<id>.sendReadReceipts`، `channels.whatsapp.accounts.<id>.dmPolicy`، `channels.whatsapp.accounts.<id>.allowFrom`.

</Accordion>

### Telegram

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "your-bot-token",
      dmPolicy: "pairing",
      allowFrom: ["tg:123456789"],
      groups: {
        "*": { requireMention: true },
        "-1001234567890": {
          allowFrom: ["@admin"],
          systemPrompt: "Keep answers brief.",
          topics: {
            "99": {
              requireMention: false,
              skills: ["search"],
              systemPrompt: "Stay on topic.",
            },
          },
        },
      },
      customCommands: [
        { command: "backup", description: "Git backup" },
        { command: "generate", description: "Create an image" },
      ],
      historyLimit: 50,
      replyToMode: "first", // off | first | all | batched
      linkPreview: true,
      streaming: { mode: "partial" }, // off | partial | block | progress (default: partial)
      actions: { reactions: true, sendMessage: true },
      reactionNotifications: "own", // off | own | all
      mediaMaxMb: 100,
      retry: {
        attempts: 3,
        minDelayMs: 400,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
      network: {
        autoSelectFamily: true,
        dnsResultOrder: "ipv4first",
      },
      apiRoot: "https://api.telegram.org",
      trustedLocalFileRoots: ["/srv/telegram-bot-api-data"],
      proxy: "socks5://localhost:9050",
      webhookUrl: "https://example.com/telegram-webhook",
      webhookSecret: "secret",
      webhookPath: "/telegram-webhook",
    },
  },
}
```

- توکن ربات: `channels.telegram.botToken` یا `channels.telegram.tokenFile` (فقط فایل عادی؛ پیوندهای نمادین رد می‌شوند)، با `TELEGRAM_BOT_TOKEN` به‌عنوان جایگزین برای حساب پیش‌فرض.
- `apiRoot` فقط ریشهٔ Telegram Bot API است. از `https://api.telegram.org` یا ریشهٔ خودمیزبان/پروکسی خود استفاده کنید، نه `https://api.telegram.org/bot<TOKEN>`؛ `openclaw doctor --fix` پسوند انتهایی ناخواستهٔ `/bot<TOKEN>` را حذف می‌کند.
- برای سرور Bot API خودمیزبان در حالت `--local`، `trustedLocalFileRoots` مسیرهای میزبان قابل خواندن توسط OpenClaw را فهرست می‌کند. حجم دادهٔ سرور را روی میزبان OpenClaw سوار کنید و ریشهٔ داده یا دایرکتوری مختص هر توکن آن را پیکربندی کنید؛ مسیرهای کانتینر زیر `/var/lib/telegram-bot-api` به آن ریشه‌ها نگاشت می‌شوند. دیگر مسیرهای مطلق همچنان رد می‌شوند.
- `channels.telegram.defaultAccount` اختیاری، هنگامی که با یک شناسهٔ حساب پیکربندی‌شده مطابقت داشته باشد، انتخاب حساب پیش‌فرض را بازنویسی می‌کند.
- در راه‌اندازی‌های چندحسابی (2+ شناسهٔ حساب)، برای جلوگیری از مسیریابی جایگزین یک پیش‌فرض صریح (`channels.telegram.defaultAccount` یا `channels.telegram.accounts.default`) تنظیم کنید؛ `openclaw doctor` در صورت نبودن یا نامعتبر بودن آن هشدار می‌دهد.
- `configWrites: false` نوشتن پیکربندی آغازشده از Telegram را مسدود می‌کند (مهاجرت‌های شناسهٔ ابرگروه، `/config set|unset`).
- ورودی‌های سطح‌بالای `bindings[]` همراه با `type: "acp"` اتصال‌های پایدار ACP را برای موضوع‌های انجمن پیکربندی می‌کنند (از `chatId:topic:topicId` متعارف در `match.peer.id` استفاده کنید). معنای فیلدها در [عامل‌های ACP](/fa/tools/acp-agents#persistent-channel-bindings) مشترک است.
- پیش‌نمایش‌های جریان Telegram از `sendMessage` + `editMessageText` استفاده می‌کنند (در گفت‌وگوهای خصوصی و گروهی کار می‌کند).
- `network.dnsResultOrder` برای جلوگیری از خطاهای رایج واکشی IPv6 به‌طور پیش‌فرض `"ipv4first"` است.
- خط‌مشی تلاش مجدد: به [خط‌مشی تلاش مجدد](/fa/concepts/retry) مراجعه کنید.

### Discord

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: "your-bot-token",
      mediaMaxMb: 100,
      allowBots: false,
      actions: {
        reactions: true,
        stickers: true,
        polls: true,
        permissions: true,
        messages: true,
        threads: true,
        pins: true,
        search: true,
        memberInfo: true,
        roleInfo: true,
        roles: false,
        channelInfo: true,
        voiceStatus: true,
        events: true,
        moderation: false,
      },
      replyToMode: "off", // off | first | all | batched
      dmPolicy: "pairing",
      allowFrom: ["1234567890", "123456789012345678"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["openclaw-dm"] },
      guilds: {
        "123456789012345678": {
          slug: "friends-of-openclaw",
          requireMention: false,
          ignoreOtherMentions: true,
          reactionNotifications: "own",
          users: ["987654321098765432"],
          channels: {
            general: { allow: true },
            help: {
              allow: true,
              requireMention: true,
              users: ["987654321098765432"],
              skills: ["docs"],
              systemPrompt: "Short answers only.",
            },
          },
        },
      },
      historyLimit: 20,
      textChunkLimit: 2000,
      suppressEmbeds: true,
      streaming: {
        mode: "progress", // off | partial | block | progress (پیش‌فرض Discord: progress)
        chunkMode: "length", // length | newline
        progress: {
          label: "auto",
          maxLines: 8,
          maxLineChars: 120,
          toolProgress: true,
        },
      },
      maxLinesPerMessage: 17,
      ui: {
        components: {
          accentColor: "#5865F2",
        },
      },
      threadBindings: {
        enabled: true,
        idleHours: 24,
        maxAgeHours: 0,
        spawnSessions: true,
        defaultSpawnContext: "fork",
      },
      voice: {
        enabled: true,
        autoJoin: [
          {
            guildId: "123456789012345678",
            channelId: "234567890123456789",
          },
        ],
        daveEncryption: true,
        decryptionFailureTolerance: 24,
        connectTimeoutMs: 30000,
        reconnectGraceMs: 15000,
        tts: {
          provider: "openai",
          openai: { voice: "alloy" },
        },
      },
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["987654321098765432"],
        agentFilter: ["default"],
        sessionFilter: ["discord:"],
        target: "dm", // dm | channel | both
        cleanupAfterResolve: false,
      },
      retry: {
        attempts: 3,
        minDelayMs: 500,
        maxDelayMs: 30000,
        jitter: 0.1,
      },
    },
  },
}
```

- توکن: `channels.discord.token`، با `DISCORD_BOT_TOKEN` به‌عنوان گزینهٔ جایگزین برای حساب پیش‌فرض.
- فراخوانی‌های خروجی مستقیم که یک `token` صریح Discord ارائه می‌کنند، از همان توکن برای فراخوانی استفاده می‌کنند؛ تنظیمات تلاش مجدد/سیاست حساب همچنان از حساب انتخاب‌شده در تصویر لحظه‌ای زمان اجرای فعال دریافت می‌شوند.
- `channels.discord.defaultAccount` اختیاری، هنگامی که با شناسهٔ یک حساب پیکربندی‌شده مطابقت داشته باشد، انتخاب حساب پیش‌فرض را لغو می‌کند.
- برای مقصدهای تحویل از `user:<id>` (پیام مستقیم) یا `channel:<id>` (کانال انجمن) استفاده کنید؛ شناسه‌های صرفاً عددی رد می‌شوند.
- نامک‌های انجمن با حروف کوچک هستند و فاصله‌ها در آن‌ها با `-` جایگزین می‌شوند؛ کلیدهای کانال از نام نامک‌شده استفاده می‌کنند (بدون `#`). شناسه‌های انجمن را ترجیح دهید.
- پیام‌های ایجادشده توسط ربات به‌طور پیش‌فرض نادیده گرفته می‌شوند. `allowBots: true` آن‌ها را فعال می‌کند؛ برای پذیرش فقط پیام‌های رباتی که ربات را منشن می‌کنند از `allowBots: "mentions"` استفاده کنید (پیام‌های خود ربات همچنان فیلتر می‌شوند).
- کانال‌هایی که از پیام‌های ورودی ایجادشده توسط ربات پشتیبانی می‌کنند، می‌توانند از [محافظت مشترک در برابر حلقهٔ ربات](/fa/channels/bot-loop-protection) استفاده کنند. `channels.defaults.botLoopProtection` را برای بودجه‌های پایهٔ جفت تنظیم کنید، سپس فقط هنگامی کانال یا حساب را بازنویسی کنید که یک سطح به محدودیت‌های متفاوتی نیاز دارد.
- `channels.discord.guilds.<id>.ignoreOtherMentions` (و بازنویسی‌های کانال) پیام‌هایی را حذف می‌کند که کاربر یا نقش دیگری را منشن می‌کنند اما ربات را منشن نمی‌کنند (به‌استثنای @everyone/@here).
- `channels.discord.mentionAliases` پیش از ارسال، متن پایدار خروجی `@handle` را به شناسه‌های کاربری Discord نگاشت می‌کند تا هم‌تیمی‌های شناخته‌شده حتی زمانی که حافظهٔ نهان گذرای فهرست خالی است، به‌شکلی قطعی منشن شوند. بازنویسی‌های مختص هر حساب زیر `channels.discord.accounts.<accountId>.mentionAliases` قرار دارند.
- `maxLinesPerMessage` (پیش‌فرض `17`) پیام‌های بلند را حتی در صورت کمتر بودن از 2000 نویسه تقسیم می‌کند.
- `channels.discord.suppressEmbeds` به‌طور پیش‌فرض `true` است؛ بنابراین URLهای خروجی، مگر اینکه این گزینه غیرفعال شود، به پیش‌نمایش پیوند Discord گسترش نمی‌یابند. بارهای صریح `embeds` همچنان به‌طور عادی ارسال می‌شوند؛ فراخوانی‌های ابزار برای هر پیام می‌توانند این رفتار را با `suppressEmbeds` بازنویسی کنند.
- `channels.discord.threadBindings` مسیریابی متصل به رشتهٔ Discord را کنترل می‌کند:
  - `enabled`: بازنویسی Discord برای قابلیت‌های نشست متصل به رشته (`/focus`، `/unfocus`، `/agents`، `/session idle`، `/session max-age` و تحویل/مسیریابی متصل)
  - `idleHours`: بازنویسی Discord برای لغو تمرکز خودکار بر اثر عدم فعالیت، برحسب ساعت (`0` آن را غیرفعال می‌کند)
  - `maxAgeHours`: بازنویسی Discord برای حداکثر سن قطعی، برحسب ساعت (`0` آن را غیرفعال می‌کند)
  - `spawnSessions`: کلید ایجاد/اتصال خودکار رشته برای `sessions_spawn({ thread: true })` و ایجاد رشتهٔ ACP (پیش‌فرض: `true`)
  - `defaultSpawnContext`: زمینهٔ بومی زیرعامل برای ایجادهای متصل به رشته (به‌طور پیش‌فرض `"fork"`)
- مدخل‌های سطح‌بالای `bindings[]` دارای `type: "acp"`، اتصال‌های پایدار ACP را برای کانال‌ها و رشته‌ها پیکربندی می‌کنند (از شناسهٔ کانال/رشته در `match.peer.id` استفاده کنید). معنای فیلدها در [عامل‌های ACP](/fa/tools/acp-agents#persistent-channel-bindings) مشترک است.
- `channels.discord.ui.components.accentColor` رنگ تأکیدی محفظه‌های مؤلفه‌های نسخهٔ 2 Discord را تنظیم می‌کند.
- `channels.discord.agentComponents.ttlMs` مدت ثبت‌ماندن فراخوانی‌های بازگشتی مؤلفه‌های ارسال‌شدهٔ Discord را کنترل می‌کند. پیش‌فرض `1800000` (30 دقیقه)، حداکثر `86400000` (24 ساعت). بازنویسی‌های مختص هر حساب زیر `channels.discord.accounts.<accountId>.agentComponents.ttlMs` قرار دارند. کوتاه‌ترین TTL متناسب با گردش کار را ترجیح دهید.
- `channels.discord.voice` مکالمات کانال صوتی Discord و بازنویسی‌های اختیاری پیوستن خودکار + LLM + TTS را فعال می‌کند. پیکربندی‌های صرفاً متنی Discord به‌طور پیش‌فرض صدا را غیرفعال نگه می‌دارند؛ برای فعال‌سازی آن، `channels.discord.voice.enabled=true` را تنظیم کنید.
- `channels.discord.voice.model` به‌صورت اختیاری مدل LLM مورداستفاده برای پاسخ‌های کانال صوتی Discord را بازنویسی می‌کند.
- `channels.discord.voice.daveEncryption` (پیش‌فرض `true`) و `channels.discord.voice.decryptionFailureTolerance` (پیش‌فرض `24`) مستقیماً به گزینه‌های DAVE در `@discordjs/voice` منتقل می‌شوند.
- `channels.discord.voice.connectTimeoutMs` انتظار اولیه برای Ready در `@discordjs/voice` را برای `/vc join` و تلاش‌های پیوستن خودکار کنترل می‌کند (پیش‌فرض `30000`).
- `channels.discord.voice.reconnectGraceMs` کنترل می‌کند یک نشست صوتی قطع‌شده چه مدت فرصت دارد پیش از آنکه OpenClaw آن را نابود کند، وارد سیگنال‌دهی اتصال مجدد شود (پیش‌فرض `15000`).
- پخش صوتی Discord با رویداد آغاز صحبت کاربر دیگری قطع نمی‌شود. برای جلوگیری از حلقه‌های بازخورد، OpenClaw هنگام پخش TTS دریافت صدای جدید را نادیده می‌گیرد.
- OpenClaw همچنین پس از شکست‌های مکرر رمزگشایی، با ترک و پیوستن مجدد به نشست صوتی برای بازیابی دریافت صدا تلاش می‌کند.
- `channels.discord.streaming` کلید متعارف حالت جریان است. پیش‌فرض Discord برابر `streaming.mode: "progress"` است تا پیشرفت ابزار/کار در یک پیام پیش‌نمایش ویرایش‌شونده نمایش داده شود؛ برای غیرفعال‌سازی آن، `streaming.mode: "off"` را تنظیم کنید. کلیدهای تخت قدیمی (`streamMode`، `chunkMode`، `blockStreaming`، `draftChunk`، `blockStreamingCoalesce`) دیگر هنگام اجرا خوانده نمی‌شوند؛ برای مهاجرت پیکربندی ذخیره‌شده، `openclaw doctor --fix` را اجرا کنید.
- `channels.discord.autoPresence` دسترس‌پذیری زمان اجرا را به وضعیت حضور ربات نگاشت می‌کند (سالم => آنلاین، افت‌کرده => بیکار، تمام‌شده => مزاحم نشوید) و بازنویسی اختیاری متن وضعیت را امکان‌پذیر می‌کند.
- `channels.discord.guilds.<id>.presenceEvents` ورودهای دسترس‌پذیری انسان‌ها را به‌صورت رویدادهای سیستمی عامل به یک کانال پیکربندی‌شدهٔ Discord هدایت می‌کند. اعضای واجد شرایط باید بتوانند `channelId` را مشاهده کنند؛ رشته‌های عمومی قابلیت مشاهده را از والد به ارث می‌برند، درحالی‌که رشته‌های خصوصی علاوه‌بر آن به عضویت یا Manage Threads نیاز دارند. `users` می‌تواند این مخاطبان را محدودتر کند. این قابلیت اعضای آنلاین فعلی را از تصاویر لحظه‌ای کامل `GUILD_CREATE` مقداردهی اولیه می‌کند، گذارهای مشاهده‌شده از آفلاین به آنلاین را هدایت می‌کند و نخستین سیگنال آنلاین بعدی برای عضوی دیده‌نشده را به‌عنوان دسترس‌پذیرشدن جدید در نظر می‌گیرد، بدون اینکه ادعا کند آن عضو آنلاین شده یا پس از تصویر لحظه‌ای پیوسته است. انجمن‌های فراتر از محدودیت تصویر لحظه‌ای 75,000 عضوی Discord ابتدا به یک به‌روزرسانی صریح آفلاین نیاز دارند. کنترل‌های محدودسازی: `reconnectSuppressSeconds` (پنجرهٔ سکوت پس از یک نشست جدید Gateway، هنگام بازسازی وضعیت حضور انجمن؛ پیش‌فرض 300، `0` آن را غیرفعال می‌کند) و `burstLimit`/`burstWindowSeconds` (محدودیت نرخ رویدادهای با موفقیت در صف قرارگرفته برای هر انجمن؛ پیش‌فرض 8 رویداد در هر پنجرهٔ لغزان 60s). نشست‌های ازسرگرفته‌شده پنجرهٔ سرکوب اتصال مجدد را آغاز نمی‌کنند. دورهٔ انتظار موجود برای خوشامدگویی مجدد به هر کاربر همچنان هشت ساعت است. این قابلیت به `channels.discord.intents.presence=true`، مجوز ممتاز Presence Intent در Developer Portal متعلق به Discord و Heartbeat فعال عامل نیاز دارد.
- `channels.discord.dangerouslyAllowNameMatching` تطبیق تغییرپذیر نام/برچسب را دوباره فعال می‌کند (حالت سازگاری اضطراری).
- `channels.discord.execApprovals`: تحویل بومی تأیید اجرای Discord و مجوزدهی تأییدکننده.
  - `enabled`: `true`، `false` یا `"auto"` (پیش‌فرض). در حالت خودکار، هنگامی که تأییدکنندگان از `approvers` یا `commands.ownerAllowFrom` قابل شناسایی باشند، تأییدهای اجرا فعال می‌شوند.
  - `approvers`: شناسه‌های کاربری Discord که مجاز به تأیید درخواست‌های اجرا هستند. در صورت حذف، به `commands.ownerAllowFrom` بازمی‌گردد.
  - `agentFilter`: فهرست مجاز اختیاری شناسه‌های عامل. برای ارسال تأییدها برای همهٔ عامل‌ها، آن را حذف کنید.
  - `sessionFilter`: الگوهای اختیاری کلید نشست (زیررشته یا عبارت منظم).
  - `target`: محل ارسال درخواست‌های تأیید. `"dm"` (پیش‌فرض) به پیام‌های مستقیم تأییدکنندگان ارسال می‌کند، `"channel"` به کانال مبدأ ارسال می‌کند و `"both"` به هر دو ارسال می‌کند. وقتی مقصد شامل `"channel"` باشد، دکمه‌ها فقط برای تأییدکنندگان شناسایی‌شده قابل استفاده‌اند.
  - `cleanupAfterResolve`: هنگامی که `true` باشد، پیام‌های مستقیم تأیید را پس از تأیید، رد یا پایان مهلت حذف می‌کند.

**حالت‌های اعلان واکنش:** `off` (هیچ‌کدام)، `own` (پیام‌های ربات، پیش‌فرض)، `all` (همهٔ پیام‌ها)، `allowlist` (از `guilds.<id>.users` روی همهٔ پیام‌ها).

### Google Chat

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      audienceType: "app-url", // app-url | project-number
      audience: "https://gateway.example.com/googlechat",
      webhookPath: "/googlechat",
      botUser: "users/1234567890",
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": { allow: true, requireMention: true },
      },
      actions: { reactions: true },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

- JSON حساب سرویس: درون‌خطی (`serviceAccount`) یا مبتنی بر فایل (`serviceAccountFile`).
- `serviceAccount` یک SecretRef را مستقیماً می‌پذیرد.
- گزینه‌های جایگزین محیطی: `GOOGLE_CHAT_SERVICE_ACCOUNT` یا `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE` (فقط حساب پیش‌فرض).
- برای مقصدهای تحویل از `spaces/<spaceId>` یا `users/<userId>` استفاده کنید.
- `channels.googlechat.dangerouslyAllowNameMatching` تطبیق تغییرپذیر اصل ایمیل را دوباره فعال می‌کند (حالت سازگاری اضطراری).

### Slack

```json5
{
  channels: {
    slack: {
      enabled: true,
      botToken: "xoxb-...",
      appToken: "xapp-...",
      socketMode: {
        clientPingTimeout: 15000,
        serverPingTimeout: 30000,
        pingPongLoggingEnabled: false,
      },
      dmPolicy: "pairing",
      allowFrom: ["U123", "U456", "*"],
      dm: { enabled: true, groupEnabled: false, groupChannels: ["G123"] },
      channels: {
        C123: { enabled: true, requireMention: true, allowBots: false },
        "#general": {
          enabled: true,
          requireMention: true,
          allowBots: false,
          users: ["U123"],
          skills: ["docs"],
          systemPrompt: "فقط پاسخ‌های کوتاه.",
        },
      },
      historyLimit: 50,
      allowBots: false,
      reactionNotifications: "own",
      reactionAllowlist: ["U123"],
      replyToMode: "off", // off | first | all | batched
      thread: {
        historyScope: "thread", // thread | channel
        inheritParent: false,
        initialHistoryLimit: 20,
      },
      actions: {
        reactions: true,
        messages: true,
        pins: true,
        memberInfo: true,
        emojiList: true,
      },
      slashCommand: {
        enabled: true,
        name: "openclaw",
        sessionPrefix: "slack:slash",
        ephemeral: true,
      },
      typingReaction: "hourglass_flowing_sand",
      unfurlLinks: false,
      unfurlMedia: false,
      textChunkLimit: 4000,
      streaming: {
        mode: "partial", // off | partial | block | progress
        chunkMode: "length", // length | newline
        nativeTransport: true, // وقتی mode=partial است، از API استریم بومی Slack استفاده می‌کند
      },
      mediaMaxMb: 20,
      execApprovals: {
        enabled: "auto", // true | false | "auto"
        approvers: ["U123"],
        agentFilter: ["default"],
        sessionFilter: ["slack:"],
        target: "dm", // dm | channel | both
      },
    },
  },
}
```

- **حالت Socket** به هر دو `botToken` و `appToken` نیاز دارد (`SLACK_BOT_TOKEN` + `SLACK_APP_TOKEN` برای بازگشت پیش‌فرض به متغیرهای محیطی حساب).
- **حالت HTTP** به `botToken` به‌همراه `signingSecret` نیاز دارد (در ریشه یا برای هر حساب).
- **هویت کاربر** (`identity: "user"`) به‌عنوان انسانِ مجوزدهنده مطلب ارسال می‌کند و می‌خواند. این قابلیت در حالت Socket به `userToken` به‌همراه `appToken`، یا در حالت HTTP به `userToken` به‌همراه `signingSecret` نیاز دارد. هیچ توکن ربات یا کاربر رباتی لازم نیست. برای محدوده‌های دسترسی کاربر و اشتراک‌های رویداد، به [هویت کاربر](/fa/channels/slack#user-identity-post-as-a-real-person) مراجعه کنید.
- `enterpriseOrgInstall: true` یک حساب را در مسیر رویداد سراسری سازمانی Slack Enterprise Grid
  وارد می‌کند. هنگام راه‌اندازی، توکن ربات با `auth.test` بررسی می‌شود و
  اگر حالت پیکربندی‌شده با هویت نصب Slack مطابقت نداشته باشد، راه‌اندازی شکست می‌خورد.
  پیام‌های مستقیم سازمانی باید غیرفعال باشند یا از `dmPolicy: "open"` با یک
  `allowFrom: ["*"]` مؤثر استفاده کنند. سیاست‌های کانال و کاربر باید از شناسه‌های پایدار Slack استفاده کنند؛
  نام‌های تغییرپذیر و پیشوندهای کانال پشتیبانی‌نشده باعث شکست راه‌اندازی می‌شوند. V1 فقط
  رویدادهای مستقیم Socket Mode یا HTTP `message` و `app_mention` را با پاسخ‌های
  فوری مدیریت می‌کند؛ رله، فرمان‌ها، تعاملات، App Home، شنونده‌های رویداد واکنش،
  سنجاق‌ها، ابزارهای کنش، تأییدهای بومی، اتصال‌ها، تحویل با تأخیر و
  ارسال‌های پیش‌دستانه در دسترس نیستند. تأیید دریافت، نشانگر تایپ و
  واکنش‌های وضعیتِ متعلق به شنونده با `reactions:write` همچنان در دسترس‌اند؛ اعلان‌های
  واکنش ورودی و ابزارهای کنش واکنش در دسترس نیستند. برای مانیفست با کمترین
  سطح دسترسی، گردش‌کار راه‌اندازی و محدودیت‌های کامل، به
  [نصب‌های سراسری سازمانی Enterprise Grid](/fa/channels/slack#enterprise-grid-org-wide-installs)
  مراجعه کنید.
- `socketMode` تنظیمات انتقال Socket Mode در Slack SDK را به API عمومی گیرنده Bolt منتقل می‌کند. فقط هنگام بررسی مهلت زمانی ping/pong یا رفتار websocket منقضی از آن استفاده کنید. مقدار پیش‌فرض `clientPingTimeout` برابر `15000` است؛ `serverPingTimeout` و `pingPongLoggingEnabled` فقط در صورت پیکربندی منتقل می‌شوند.
- `botToken`، `appToken`، `signingSecret` و `userToken` رشته‌های
  متن ساده یا اشیای SecretRef را می‌پذیرند.
- تصاویر لحظه‌ای حساب Slack، فیلدهای منبع/وضعیت هر اعتبارنامه مانند
  `botTokenSource`، `botTokenStatus`، `userTokenSource`، `userTokenStatus`،
  `appTokenStatus` و در حالت HTTP، `signingSecretStatus` را ارائه می‌کنند.
  `configured_unavailable` یعنی حساب
  از طریق SecretRef پیکربندی شده، اما مسیر فعلی فرمان/زمان اجرا نتوانسته است
  مقدار محرمانه را رفع کند.
- `configWrites: false` نوشتن پیکربندی آغازشده از سوی Slack را مسدود می‌کند.
- `channels.slack.defaultAccount` اختیاری، وقتی با شناسه یک حساب پیکربندی‌شده مطابقت داشته باشد، انتخاب حساب پیش‌فرض را بازنویسی می‌کند.
- `channels.slack.streaming.mode` کلید متعارف حالت استریم Slack است (پیش‌فرض `"partial"`). `channels.slack.streaming.nativeTransport` انتقال استریم بومی Slack را کنترل می‌کند (پیش‌فرض `true`). مقادیر قدیمی `streamMode`، مقدار بولی `streaming`، `chunkMode`، `blockStreaming`، `blockStreamingCoalesce` و `nativeStreaming` دیگر هنگام اجرا خوانده نمی‌شوند؛ برای مهاجرت پیکربندی ذخیره‌شده به `streaming.{mode,chunkMode,block.enabled,block.coalesce,nativeTransport}`، فرمان `openclaw doctor --fix` را اجرا کنید.
- `unfurlLinks` و `unfurlMedia` مقادیر بولی بازکردن پیوند و رسانه `chat.postMessage` در Slack را برای پاسخ‌های ربات منتقل می‌کنند. مقدار پیش‌فرض `unfurlLinks` برابر `false` است تا پیوندهای خروجی ربات، مگر در صورت فعال‌سازی، به‌صورت درون‌خطی باز نشوند؛ `unfurlMedia` مگر در صورت پیکربندی حذف می‌شود. برای بازنویسی مقدار سطح بالا برای یک حساب، هر یک از مقادیر را در `channels.slack.accounts.<accountId>` تنظیم کنید.
- برای مقصدهای تحویل از `user:<id>` (پیام مستقیم) یا `channel:<id>` استفاده کنید.

**حالت‌های اعلان واکنش:** `off`، `own` (پیش‌فرض)، `all`، `allowlist` (از `reactionAllowlist`).

**جداسازی نشست رشته:** `thread.historyScope` برای هر رشته جداگانه (پیش‌فرض) یا در سراسر کانال مشترک است. `thread.inheritParent` رونوشت کانال والد را در رشته‌های جدید کپی می‌کند. `thread.initialHistoryLimit` (پیش‌فرض `20`) حداکثر تعداد پیام‌های موجود رشته را که هنگام آغاز یک نشست رشته جدید دریافت می‌شوند محدود می‌کند؛ `0` دریافت تاریخچه رشته را غیرفعال می‌کند.

- استریم بومی Slack به‌همراه وضعیت رشته به سبک دستیار Slack یعنی «در حال تایپ...» به یک مقصد پاسخ در رشته نیاز دارد. پیام‌های مستقیم سطح بالا به‌طور پیش‌فرض خارج از رشته باقی می‌مانند، بنابراین همچنان می‌توانند به‌جای نمایش پیش‌نمایش استریم/وضعیت بومی به سبک رشته، از طریق پیش‌نمایش‌های پیش‌نویسِ ارسال و ویرایش Slack استریم شوند.
- `typingReaction` هنگام اجرای پاسخ، یک واکنش موقت به پیام ورودی Slack اضافه می‌کند و پس از تکمیل آن را حذف می‌کند. از یک کد کوتاه ایموجی Slack مانند `"hourglass_flowing_sand"` استفاده کنید.
- `channels.slack.execApprovals`: تحویل کلاینت تأیید بومی Slack و مجوزدهی تأییدکننده اجرای فرمان. طرح‌واره همان Discord است: `enabled` (`true`/`false`/`"auto"`)، `approvers` (شناسه‌های کاربر Slack)، `agentFilter`، `sessionFilter` و `target` (`"dm"`، `"channel"` یا `"both"`). وقتی تأییدکنندگان Plugin در Slack رفع شوند، تأییدهای Plugin می‌توانند برای درخواست‌های منشأگرفته از Slack از این مسیر کلاینت بومی استفاده کنند؛ تحویل تأیید بومی Plugin در Slack نیز می‌تواند از طریق `approvals.plugin` برای نشست‌های منشأگرفته از Slack یا مقصدهای Slack فعال شود. تأییدهای Plugin از تأییدکنندگان Plugin در Slack از `allowFrom` و مسیریابی پیش‌فرض استفاده می‌کنند، نه تأییدکنندگان اجرا.

| گروه کنش | پیش‌فرض | یادداشت‌ها                  |
| ------------ | ------- | ---------------------- |
| واکنش‌ها    | فعال | افزودن واکنش + فهرست واکنش‌ها |
| پیام‌ها     | فعال | خواندن/ارسال/ویرایش/حذف  |
| سنجاق‌ها         | فعال | سنجاق‌کردن/برداشتن سنجاق/فهرست         |
| اطلاعات عضو   | فعال | اطلاعات عضو            |
| فهرست ایموجی    | فعال | فهرست ایموجی‌های سفارشی      |

### Mattermost

Mattermost همانند Discord، Slack و WhatsApp به‌صورت یک Plugin جداگانه نصب می‌شود:

```bash
openclaw plugins install @openclaw/mattermost
```

پیش از ثابت‌کردن یک نسخه، برچسب‌های توزیع فعلی را در [npmjs.com/package/@openclaw/mattermost](https://www.npmjs.com/package/@openclaw/mattermost) بررسی کنید.

```json5
{
  channels: {
    mattermost: {
      enabled: true,
      botToken: "mm-token",
      baseUrl: "https://chat.example.com",
      dmPolicy: "pairing",
      chatmode: "oncall", // oncall | onmessage | onchar
      oncharPrefixes: [">", "!"],
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
      commands: {
        native: true, // نیازمند فعال‌سازی
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // نشانی اینترنتی صریح و اختیاری برای استقرارهای پروکسی معکوس/عمومی
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
      textChunkLimit: 4000,
      streaming: { chunkMode: "length" },
    },
  },
}
```

حالت‌های گفت‌وگو: `oncall` (پاسخ در صورت اشاره با @، پیش‌فرض)، `onmessage` (هر پیام)، `onchar` (پیام‌هایی که با پیشوند محرک آغاز می‌شوند).

وقتی فرمان‌های بومی Mattermost فعال باشند:

- `commands.callbackPath` باید یک مسیر باشد (برای نمونه `/api/channels/mattermost/command`)، نه یک URL کامل.
- `commands.callbackUrl` باید به نقطه پایانی Gateway در OpenClaw رفع شود و از سرور Mattermost قابل دسترسی باشد.
- فراخوانی‌های بازگشتی اسلش بومی با توکن‌های مختص هر فرمان که
  Mattermost هنگام ثبت فرمان اسلش بازمی‌گرداند احراز هویت می‌شوند. اگر ثبت شکست بخورد یا هیچ
  فرمانی فعال نشود، OpenClaw فراخوانی‌های بازگشتی را با
  `Unauthorized: invalid command token.` رد می‌کند.
- برای میزبان‌های فراخوانی بازگشتی خصوصی/tailnet/داخلی، ممکن است Mattermost نیاز داشته باشد که
  `ServiceSettings.AllowedUntrustedInternalConnections` شامل میزبان/دامنه فراخوانی بازگشتی باشد.
  از مقادیر میزبان/دامنه استفاده کنید، نه URLهای کامل.
- `channels.mattermost.configWrites`: نوشتن پیکربندی آغازشده از سوی Mattermost را مجاز یا ممنوع می‌کند.
- `channels.mattermost.requireMention`: پیش از پاسخ‌دادن در کانال‌ها، `@mention` را الزامی می‌کند.
- `channels.mattermost.groups.<channelId>.requireMention`: بازنویسی الزام اشاره برای هر کانال (`"*"` برای پیش‌فرض).
- `channels.mattermost.defaultAccount` اختیاری، وقتی با شناسه یک حساب پیکربندی‌شده مطابقت داشته باشد، انتخاب حساب پیش‌فرض را بازنویسی می‌کند.

### Signal

```json5
{
  channels: {
    signal: {
      enabled: true,
      account: "+15555550123", // اتصال اختیاری حساب
      dmPolicy: "pairing",
      allowFrom: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      configWrites: true,
      reactionNotifications: "own", // off | own | all | allowlist
      reactionAllowlist: ["+15551234567", "uuid:123e4567-e89b-12d3-a456-426614174000"],
      historyLimit: 50,
    },
  },
}
```

**حالت‌های اعلان واکنش:** `off`، `own` (پیش‌فرض)، `all`، `allowlist` (از `reactionAllowlist`).

- `channels.signal.account`: راه‌اندازی کانال را به هویت یک حساب مشخص Signal مقید می‌کند.
- `channels.signal.configWrites`: نوشتن پیکربندی آغازشده از سوی Signal را مجاز یا ممنوع می‌کند.
- `channels.signal.defaultAccount` اختیاری، وقتی با شناسه یک حساب پیکربندی‌شده مطابقت داشته باشد، انتخاب حساب پیش‌فرض را بازنویسی می‌کند.

### iMessage

OpenClaw فرایند `imsg rpc` را اجرا می‌کند (JSON-RPC از طریق stdio). هیچ دیمون یا پورتی لازم نیست. وقتی میزبان بتواند مجوزهای پایگاه داده Messages و Automation را اعطا کند، این مسیر ترجیحی برای راه‌اندازی‌های جدید iMessage در OpenClaw است.

پشتیبانی BlueBubbles حذف شده است. `channels.bluebubbles` در نسخه فعلی OpenClaw یک سطح پیکربندی زمان اجرای پشتیبانی‌شده نیست. پیکربندی‌های قدیمی را به `channels.imessage` مهاجرت دهید؛ برای نسخه کوتاه از [حذف BlueBubbles و مسیر imsg برای iMessage](/fa/announcements/bluebubbles-imessage) و برای جدول تبدیل کامل از [مهاجرت از BlueBubbles](/fa/channels/imessage-from-bluebubbles) استفاده کنید.

اگر Gateway روی Mac واردشده به Messages اجرا نمی‌شود، `channels.imessage.enabled=true` را حفظ کنید و `channels.imessage.cliPath` را روی یک پوشش SSH تنظیم کنید که `imsg "$@"` را روی آن Mac اجرا کند. مسیر محلی پیش‌فرض `imsg` فقط مخصوص macOS است.

پیش از تکیه بر یک پوشش SSH برای ارسال‌های محیط عملیاتی، یک `imsg send` خروجی را از طریق همان پوشش دقیق بررسی کنید. برخی وضعیت‌های TCC در macOS، خودکارسازی Messages را به `/usr/libexec/sshd-keygen-wrapper` اختصاص می‌دهند؛ در نتیجه ممکن است خواندن‌ها و کاوش‌ها کار کنند، اما ارسال‌ها با AppleEvents `-1743` ناموفق شوند؛ بخش عیب‌یابی پوشش SSH در [iMessage](/fa/channels/imessage) را ببینید.

```json5
{
  channels: {
    imessage: {
      enabled: true,
      cliPath: "imsg",
      dbPath: "~/Library/Messages/chat.db",
      remoteHost: "user@gateway-host",
      dmPolicy: "pairing",
      allowFrom: ["+15555550123", "user@example.com", "chat_id:123"],
      historyLimit: 50,
      includeAttachments: false,
      attachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      remoteAttachmentRoots: ["/Users/*/Library/Messages/Attachments"],
      mediaMaxMb: 16,
      service: "auto",
      sendTransport: "auto",
      region: "US",
      actions: {
        reactions: true,
        edit: true,
        unsend: true,
        reply: true,
        sendWithEffect: true,
        sendAttachment: true,
      },
    },
  },
}
```

- `channels.imessage.defaultAccount` اختیاری، هنگامی‌که با شناسه یک حساب پیکربندی‌شده مطابقت داشته باشد، انتخاب پیش‌فرض حساب را بازنویسی می‌کند.
- به دسترسی کامل به دیسک برای پایگاه داده Messages نیاز دارد.
- مقصدهای `chat_id:<id>` را ترجیح دهید. برای فهرست‌کردن گفت‌وگوها از `imsg chats --limit 20` استفاده کنید.
- `cliPath` می‌تواند به یک پوشش SSH اشاره کند؛ برای دریافت پیوست‌ها با SCP، `remoteHost` را روی `host` یا `user@host` تنظیم کنید.
- `attachmentRoots` و `remoteAttachmentRoots` مسیرهای پیوست ورودی را محدود می‌کنند (پیش‌فرض: `/Users/*/Library/Messages/Attachments`).
- SCP از بررسی سخت‌گیرانه کلید میزبان استفاده می‌کند؛ بنابراین مطمئن شوید کلید میزبان رله از قبل در `~/.ssh/known_hosts` وجود دارد.
- `channels.imessage.configWrites`: نوشتن پیکربندی آغازشده از iMessage را مجاز یا رد می‌کند.
- `channels.imessage.sendTransport`: روش انتقال ترجیحی ارسال RPC در `imsg` برای پاسخ‌های خروجی عادی. `auto` (پیش‌فرض)، هنگامی‌که پل IMCore در حال اجرا باشد، برای گفت‌وگوهای موجود از آن استفاده می‌کند و سپس به AppleScript بازمی‌گردد؛ `bridge` به تحویل از طریق API خصوصی نیاز دارد؛ `applescript` مسیر عمومی خودکارسازی Messages را اجباری می‌کند.
- `channels.imessage.actions.*`: کنش‌های API خصوصی را فعال می‌کند که علاوه بر آن با `imsg status` / `openclaw channels status --probe` نیز محدود می‌شوند.
- `channels.imessage.includeAttachments` به‌طور پیش‌فرض خاموش است؛ پیش از انتظار رسانه ورودی در نوبت‌های عامل، آن را روی `true` تنظیم کنید.
- بازیابی ورودی پس از راه‌اندازی مجدد پل/Gateway خودکار است (حذف موارد تکراری بر اساس GUID، به‌علاوه محدودیت سنی برای صف عقب‌افتاده قدیمی). پیکربندی‌های موجود `channels.imessage.catchup.enabled: true` همچنان به‌عنوان نمایه سازگاری منسوخ‌شده رعایت می‌شوند؛ `catchup` به‌طور پیش‌فرض غیرفعال است.
- `channels.imessage.groups`: دفتر ثبت گروه و تنظیمات هر گروه. با `groupPolicy: "allowlist"`، کلیدهای صریح `chat_id` یا یک ورودی عام `"*"` را پیکربندی کنید تا پیام‌های گروهی بتوانند از دروازه دفتر ثبت عبور کنند.
- ورودی‌های سطح‌بالای `bindings[]` با `type: "acp"` می‌توانند مکالمه‌های iMessage را به نشست‌های پایدار ACP متصل کنند. در `match.peer.id` از یک شناسه تماس نرمال‌شده یا مقصد صریح گفت‌وگو (`chat_id:*`، `chat_guid:*`، `chat_identifier:*`) استفاده کنید. معنای فیلدهای مشترک: [عامل‌های ACP](/fa/tools/acp-agents#persistent-channel-bindings).

<Accordion title="نمونه پوشش SSH برای iMessage">

```bash
#!/usr/bin/env bash
exec ssh -T gateway-host imsg "$@"
```

</Accordion>

### Matrix

Matrix مبتنی بر Plugin است و در `channels.matrix` پیکربندی می‌شود.

```json5
{
  channels: {
    matrix: {
      enabled: true,
      homeserver: "https://matrix.example.org",
      accessToken: "syt_bot_xxx",
      proxy: "http://127.0.0.1:7890",
      encryption: true,
      initialSyncLimit: 20,
      defaultAccount: "ops",
      accounts: {
        ops: {
          name: "Ops",
          userId: "@ops:example.org",
          accessToken: "syt_ops_xxx",
        },
        alerts: {
          userId: "@alerts:example.org",
          password: "secret",
          proxy: "http://127.0.0.1:7891",
        },
      },
    },
  },
}
```

- احراز هویت توکنی از `accessToken` استفاده می‌کند؛ احراز هویت با گذرواژه از `userId` + `password` استفاده می‌کند.
- `channels.matrix.proxy` ترافیک HTTP مربوط به Matrix را از طریق یک پراکسی صریح HTTP(S) هدایت می‌کند. حساب‌های نام‌گذاری‌شده می‌توانند آن را با `channels.matrix.accounts.<id>.proxy` بازنویسی کنند.
- `channels.matrix.network.dangerouslyAllowPrivateNetwork` سرورهای خانگی خصوصی/داخلی را مجاز می‌کند. `proxy` و این پذیرش صریح شبکه، کنترل‌هایی مستقل هستند.
- `channels.matrix.defaultAccount` حساب ترجیحی را در پیکربندی‌های چندحسابی انتخاب می‌کند.
- `channels.matrix.autoJoin` به‌طور پیش‌فرض `"off"` است؛ بنابراین اتاق‌های دعوت‌شده و دعوت‌های تازه شبیه پیام مستقیم نادیده گرفته می‌شوند تا زمانی که `autoJoin: "allowlist"` را با `autoJoinAllowlist` یا `autoJoin: "always"` تنظیم کنید.
- `channels.matrix.execApprovals`: تحویل تأیید اجرای بومی Matrix و مجوزدهی تأییدکننده.
  - `enabled`: `true`، `false` یا `"auto"` (پیش‌فرض). در حالت خودکار، تأییدهای اجرا هنگامی فعال می‌شوند که تأییدکنندگان از `approvers` یا `commands.ownerAllowFrom` قابل شناسایی باشند.
  - `approvers`: شناسه‌های کاربری Matrix (برای مثال `@owner:example.org`) که مجاز به تأیید درخواست‌های اجرا هستند.
  - `agentFilter`: فهرست مجاز اختیاری شناسه‌های عامل. برای ارسال تأییدها برای همه عامل‌ها، آن را حذف کنید.
  - `sessionFilter`: الگوهای اختیاری کلید نشست (زیررشته یا عبارت منظم).
  - `target`: محل ارسال درخواست‌های تأیید. `"dm"` (پیش‌فرض)، `"channel"` (اتاق مبدأ) یا `"both"`.
  - بازنویسی‌های هر حساب: `channels.matrix.accounts.<id>.execApprovals`.
- `channels.matrix.dm.sessionScope` نحوه گروه‌بندی پیام‌های مستقیم Matrix در نشست‌ها را کنترل می‌کند: `per-user` (پیش‌فرض) نشست را بر اساس همتای مسیردهی‌شده مشترک می‌کند، درحالی‌که `per-room` هر اتاق پیام مستقیم را جدا می‌کند.
- کاوش‌های وضعیت Matrix و جست‌وجوهای زنده فهرست راهنما از همان سیاست پراکسی ترافیک زمان اجرا استفاده می‌کنند.
- پیکربندی کامل Matrix، قواعد مقصدگیری و نمونه‌های راه‌اندازی در [Matrix](/fa/channels/matrix) مستند شده‌اند.

### Microsoft Teams

Microsoft Teams مبتنی بر Plugin است و در `channels.msteams` پیکربندی می‌شود.

```json5
{
  channels: {
    msteams: {
      enabled: true,
      configWrites: true,
      // appId, appPassword, tenantId, webhook, team/channel policies:
      // see /channels/msteams
    },
  },
}
```

- مسیرهای کلید اصلی پوشش‌داده‌شده در اینجا: `channels.msteams`، `channels.msteams.configWrites`.
- پیکربندی کامل Teams (اعتبارنامه‌ها، Webhook، سیاست پیام مستقیم/گروه و بازنویسی‌های هر تیم/هر کانال) در [Microsoft Teams](/fa/channels/msteams) مستند شده است.

### IRC

IRC مبتنی بر Plugin است و در `channels.irc` پیکربندی می‌شود.

```json5
{
  channels: {
    irc: {
      enabled: true,
      dmPolicy: "pairing",
      configWrites: true,
      nickserv: {
        enabled: true,
        service: "NickServ",
        password: "${IRC_NICKSERV_PASSWORD}",
        register: false,
        registerEmail: "bot@example.com",
      },
    },
  },
}
```

- مسیرهای کلید اصلی پوشش‌داده‌شده در اینجا: `channels.irc`، `channels.irc.dmPolicy`، `channels.irc.configWrites`، `channels.irc.nickserv.*`.
- `channels.irc.defaultAccount` اختیاری، هنگامی‌که با شناسه یک حساب پیکربندی‌شده مطابقت داشته باشد، انتخاب پیش‌فرض حساب را بازنویسی می‌کند.
- پیکربندی کامل کانال IRC (میزبان/درگاه/TLS/کانال‌ها/فهرست‌های مجاز/محدودسازی بر اساس اشاره) در [IRC](/fa/channels/irc) مستند شده است.

### چندحسابی (همه کانال‌ها)

چند حساب را در هر کانال اجرا کنید (هرکدام با `accountId` اختصاصی خود):

```json5
{
  channels: {
    telegram: {
      accounts: {
        default: {
          name: "Primary bot",
          botToken: "123456:ABC...",
        },
        alerts: {
          name: "Alerts bot",
          botToken: "987654:XYZ...",
        },
      },
    },
  },
}
```

- هنگامی‌که `accountId` حذف شده باشد، از `default` استفاده می‌شود (CLI + مسیردهی).
- توکن‌های محیطی فقط برای حساب **پیش‌فرض** اعمال می‌شوند.
- تنظیمات پایه کانال برای همه حساب‌ها اعمال می‌شوند، مگر اینکه برای هر حساب بازنویسی شوند.
- برای هدایت هر حساب به یک عامل متفاوت، از `bindings[].match.accountId` استفاده کنید.
- اگر درحالی‌که هنوز از پیکربندی سطح‌بالای تک‌حسابی کانال استفاده می‌کنید، حسابی غیرپیش‌فرض را از طریق `openclaw channels add` (یا راه‌اندازی اولیه کانال) اضافه کنید، OpenClaw ابتدا مقادیر تک‌حسابی سطح‌بالای مختص حساب را به نگاشت حساب‌های کانال منتقل می‌کند تا حساب اصلی همچنان کار کند. بیشتر کانال‌ها آن‌ها را به `channels.<channel>.accounts.default` منتقل می‌کنند؛ Matrix می‌تواند در عوض مقصد نام‌گذاری‌شده/پیش‌فرض موجود و منطبق را حفظ کند.
- اتصال‌های موجودِ فقط مختص کانال (بدون `accountId`) همچنان با حساب پیش‌فرض مطابقت دارند؛ اتصال‌های مختص حساب اختیاری باقی می‌مانند.
- `openclaw doctor --fix` همچنین با انتقال مقادیر تک‌حسابی سطح‌بالای مختص حساب به حساب ارتقایافته‌ای که برای آن کانال انتخاب شده است، شکل‌های ترکیبی را ترمیم می‌کند. بیشتر کانال‌ها از `accounts.default` استفاده می‌کنند؛ Matrix می‌تواند در عوض مقصد نام‌گذاری‌شده/پیش‌فرض موجود و منطبق را حفظ کند.

### سایر کانال‌های Plugin

بسیاری از کانال‌های Plugin به‌شکل `channels.<id>` پیکربندی می‌شوند و در صفحه‌های اختصاصی کانال خود مستند شده‌اند (برای مثال Feishu، LINE، Nextcloud Talk، Nostr، QQ Bot، Synology Chat، Twitch و Zalo).
نمایه کامل کانال‌ها را ببینید: [کانال‌ها](/fa/channels).

### محدودسازی گفت‌وگوی گروهی بر اساس اشاره

پیام‌های گروهی به‌طور پیش‌فرض **به اشاره نیاز دارند** (اشاره در فراداده یا الگوهای امن عبارت منظم). این مورد برای گفت‌وگوهای گروهی WhatsApp، Telegram، Discord، Google Chat و iMessage اعمال می‌شود.

پاسخ‌های قابل‌مشاهده جداگانه کنترل می‌شوند. درخواست‌های مستقیم عادی در گروه، کانال و WebChat داخلی به‌طور پیش‌فرض تحویل نهایی خودکار دارند: متن نهایی دستیار از طریق مسیر قدیمی پاسخ قابل‌مشاهده ارسال می‌شود. هنگامی‌که پاسخ‌های مبدأ نوشته‌شده توسط مدل باید تنها پس از فراخوانی `message(action=send)` توسط عامل ارسال شوند، `messages.visibleReplies: "message_tool"` یا `messages.groupChat.visibleReplies: "message_tool"` را فعال کنید. اگر مدل در حالت ابزارمحور فعال‌شده، بدون فراخوانی ابزار پیام، پاسخی نهایی و دارای محتوای معنادار برگرداند، آن متن نهایی خصوصی باقی می‌ماند، گزارش تفصیلی Gateway فراداده محموله سرکوب‌شده را ثبت می‌کند و OpenClaw یک تلاش مجدد بازیابی را در صف قرار می‌دهد و از مدل می‌خواهد همان پاسخ را از طریق `message(action=send)` تحویل دهد.

سیاست ابزارمحور بر پاسخ‌های مبدأ دستیار و رسانه عمومی ابزار حاکم است. این سیاست خروجی پایانی متعلق به زمان اجرا، مانند پاسخ‌های فرمان مجاز، اعلان‌های پایدار تکمیل یا مصنوعات بومی ارائه‌دهنده را که مهارکننده مالک صراحتاً متعلق به میزبان طبقه‌بندی می‌کند، سرکوب نمی‌کند. مصنوعات متعلق به میزبان از طریق مسیر عادی ارسال کانال تحویل داده می‌شوند و همچنان رد خروجی `sendPolicy` را رعایت می‌کنند. نوبت‌های محیطی `room_event`، حتی هنگامی‌که خروجی زمان اجرا متعلق به میزبان علامت‌گذاری شده باشد، مگر اینکه فرمان‌هایی صریح باشند، بی‌صدا باقی می‌مانند.

پاسخ‌های قابل‌مشاهده ابزارمحور به مدل/زمان اجرایی نیاز دارند که ابزارها را با اطمینان فراخوانی کند و برای اتاق‌های محیطی مشترک روی مدل‌های نسل جدید مانند GPT-5.6 Sol توصیه می‌شوند. برخی مدل‌های ضعیف‌تر می‌توانند متن نهایی را پاسخ دهند، اما در درک اینکه خروجی قابل‌مشاهده در مبدأ باید با `message(action=send)` ارسال شود، ناموفق‌اند. OpenClaw به‌طور پیش‌فرض مورد رایج نهاییِ تحویل‌نشده را تنها هنگامی بازیابی می‌کند که پاسخ نهایی دارای محتوای معنادار باشد، نوبت مبدأ رویداد اتاق نباشد، سیاست ارسال تحویل را رد نکرده باشد و قبلاً هیچ پاسخ مبدئی ارسال نشده باشد. بازیابی به یک تلاش مجدد محدود است؛ ماندگاری را برای درخواست مصنوعی تلاش مجدد سرکوب می‌کند و آن تلاش مجدد را از دسته‌بندی گردآوری خارج نگه می‌دارد تا نتواند با درخواست‌های نامرتبط موجود در صف ادغام شود. اگر تلاش مجدد نیز تحویل‌نشده بماند یا نتوان آن را در صف قرار داد، OpenClaw فقط یک پیام تشخیصی پاک‌سازی‌شده مانند «پاسخی تولید کردم، اما نتوانستم آن را به این گفت‌وگو تحویل دهم. لطفاً دوباره تلاش کنید.» تحویل می‌دهد. متن نهایی خصوصی اصلی هرگز برای تحویل خودکار به مبدأ علامت‌گذاری نمی‌شود. برای مدل‌هایی که مکرراً پاسخ‌ها را تحویل‌نشده باقی می‌گذارند، از `"automatic"` استفاده کنید تا نوبت نهایی دستیار مسیر پاسخ قابل‌مشاهده باشد، به یک مدل قوی‌تر در فراخوانی ابزار تغییر دهید، گزارش تفصیلی Gateway را برای خلاصه محموله سرکوب‌شده بررسی کنید، یا `messages.groupChat.visibleReplies: "automatic"` را تنظیم کنید تا برای هر درخواست گروهی/کانالی از پاسخ‌های نهایی قابل‌مشاهده استفاده شود.

اگر ابزار پیام تحت خط‌مشی فعال ابزار در دسترس نباشد، OpenClaw به‌جای سرکوب بی‌سروصدای پاسخ، از پاسخ‌های قابل‌مشاهده خودکار استفاده می‌کند. `openclaw doctor` درباره این ناهماهنگی هشدار می‌دهد.

این قاعده درباره متن نهایی عادی عامل اعمال می‌شود. اتصال‌های مکالمه متعلق به Plugin، برای نوبت‌های ادعاشده رشته متصل، پاسخ بازگردانده‌شده Plugin مالک را به‌عنوان پاسخ قابل‌مشاهده استفاده می‌کنند؛ Plugin برای این پاسخ‌های اتصال نیازی به فراخوانی `message(action=send)` ندارد.

**عیب‌یابی: @اشاره در گروه، نشانگر تایپ را فعال می‌کند و سپس سکوت رخ می‌دهد (بدون خطا)**

نشانه: یک @اشاره در گروه/کانال، نشانگر تایپ را نمایش می‌دهد و گزارش Gateway حاوی `dispatch complete (queuedFinal=false, replies=0)` است، اما هیچ پیامی به اتاق نمی‌رسد. پیام‌های مستقیم به همان عامل به‌طور عادی پاسخ می‌گیرند.

علت: حالت پاسخ قابل‌مشاهده گروه/کانال به `"message_tool"` تبدیل می‌شود؛ بنابراین OpenClaw نوبت را اجرا می‌کند، اما متن نهایی دستیار را سرکوب می‌کند، مگر اینکه عامل `message(action=send)` را فراخوانی کند. در این حالت هیچ قرارداد `NO_REPLY` وجود ندارد؛ نبود فراخوانی ابزار پیام یعنی متن نهایی اصلی خصوصی است. اکنون OpenClaw برای نوبت‌های منبعِ دارای محتوای قابل‌توجه، یک تلاش مجدد بازیابیِ محافظت‌شده انجام می‌دهد؛ یادداشت‌های کوتاه، سکوت صریح، رویدادهای اتاق، نوبت‌های ردشده توسط خط‌مشی ارسال و نوبت‌هایی که قبلاً تحویل شده‌اند دوباره امتحان نمی‌شوند. نوبت‌های عادی گروه و کانال به‌طور پیش‌فرض از `"automatic"` استفاده می‌کنند، بنابراین این نشانه فقط زمانی ظاهر می‌شود که `messages.groupChat.visibleReplies` (یا `messages.visibleReplies` سراسری) صریحاً روی `"message_tool"` تنظیم شده باشد. `defaultVisibleReplies` در هارنس اینجا اعمال نمی‌شود — تفکیک‌کننده گروه/کانال آن را نادیده می‌گیرد؛ این گزینه فقط بر گفت‌وگوهای مستقیم/منبع اثر می‌گذارد (هارنس Codex متن‌های نهایی گفت‌وگوی مستقیم را به این روش سرکوب می‌کند).

راه‌حل: یا مدلی با توانایی قوی‌تر در فراخوانی ابزار انتخاب کنید، یا بازنویسی صریح `"message_tool"` را حذف کنید تا از مقدار پیش‌فرض `"automatic"` استفاده شود، یا `messages.groupChat.visibleReplies: "automatic"` را تنظیم کنید تا برای هر درخواست گروه/کانال، پاسخ قابل‌مشاهده اجباری شود. یک متن نهاییِ قابل‌توجه که تحویل نشده است، دیگر نباید با موفقیت خاموش پایان یابد؛ باید یا با یک تلاش مجدد `message(action=send)` بازیابی شود یا پیام تشخیصی پاک‌سازی‌شده خطای تحویل را نشان دهد. Gateway پس از ذخیره فایل، پیکربندی `messages` را به‌صورت گرم بارگذاری مجدد می‌کند؛ فقط زمانی Gateway را راه‌اندازی مجدد کنید که پایش فایل یا بارگذاری مجدد پیکربندی در استقرار غیرفعال باشد.

**انواع اشاره:**

- **اشاره‌های فراداده‌ای**: @اشاره‌های بومی پلتفرم. در حالت گفت‌وگوی شخصی WhatsApp نادیده گرفته می‌شوند.
- **الگوهای متنی**: الگوهای عبارت منظم ایمن در `agents.entries.*.groupChat.mentionPatterns`. الگوهای نامعتبر و تکرار تودرتوی ناامن نادیده گرفته می‌شوند.
- محدودسازی بر اساس اشاره فقط زمانی اعمال می‌شود که تشخیص ممکن باشد (اشاره‌های بومی یا دست‌کم یک الگو).

```json5
{
  messages: {
    visibleReplies: "automatic", // پاسخ‌های نهایی خودکار قدیمی را برای گفت‌وگوهای مستقیم/منبع اجباری می‌کند
    groupChat: {
      historyLimit: 50,
      unmentionedInbound: "room_event", // گفت‌وگوی همیشگی و بدون اشاره اتاق را به زمینه‌ای آرام تبدیل می‌کند
      visibleReplies: "message_tool", // انتخابی؛ برای پاسخ‌های قابل‌مشاهده اتاق به message(action=send) نیاز دارد
    },
  },
  agents: {
    list: [{ id: "main", groupChat: { mentionPatterns: ["@openclaw", "openclaw"] } }],
  },
}
```

`messages.groupChat.historyLimit` مقدار پیش‌فرض سراسری را تنظیم می‌کند. کانال‌ها می‌توانند آن را با `channels.<channel>.historyLimit` (یا به‌ازای هر حساب) بازنویسی کنند. برای غیرفعال‌کردن، `0` را تنظیم کنید.

`messages.groupChat.unmentionedInbound: "room_event"` پیام‌های همیشگی و بدون اشاره گروه/کانال را در کانال‌های پشتیبانی‌شده، به‌عنوان زمینه آرام اتاق ارسال می‌کند. پیام‌های دارای اشاره، فرمان‌ها و پیام‌های مستقیم همچنان درخواست کاربر باقی می‌مانند. برای نمونه‌های کامل Discord، Slack و Telegram، به [رویدادهای محیطی اتاق](/fa/channels/ambient-room-events) مراجعه کنید.

`messages.visibleReplies` مقدار پیش‌فرض سراسری رویداد منبع است؛ `messages.groupChat.visibleReplies` آن را برای رویدادهای منبع گروه/کانال بازنویسی می‌کند. وقتی `messages.visibleReplies` تنظیم نشده باشد، گفت‌وگوهای مستقیم/منبع از مقدار پیش‌فرض زمان‌اجرا یا هارنس انتخاب‌شده استفاده می‌کنند، اما نوبت‌های مستقیم داخلی WebChat برای هم‌ترازی پرامپت Pi/Codex از تحویل خودکار متن نهایی استفاده می‌کنند. برای اینکه خروجی قابل‌مشاهده عمداً به `message(action=send)` نیاز داشته باشد، `messages.visibleReplies: "message_tool"` را تنظیم کنید. فهرست‌های مجاز کانال و محدودسازی بر اساس اشاره همچنان تعیین می‌کنند که آیا یک رویداد پردازش شود یا نه.

#### محدودیت‌های تاریخچه پیام مستقیم

```json5
{
  channels: {
    telegram: {
      dmHistoryLimit: 30,
      dms: {
        "123456789": { historyLimit: 50 },
      },
    },
  },
}
```

ترتیب تفکیک: بازنویسی به‌ازای هر پیام مستقیم ← مقدار پیش‌فرض ارائه‌دهنده ← بدون محدودیت (همه حفظ می‌شوند).

این تفکیک‌کننده `channels.<provider>.dmHistoryLimit` و `channels.<provider>.dms.<id>.historyLimit` را برای هر کانالی می‌خواند که کلید نشست آن از قالب استاندارد `provider:direct:<id>` (یا قالب قدیمی `provider:dm:<id>`) پیروی کند؛ بنابراین در کانال‌های همراه و کانال‌های Plugin به یکسان کار می‌کند و به یک فهرست ثابت محدود نیست.

#### حالت گفت‌وگوی شخصی

برای فعال‌کردن حالت گفت‌وگوی شخصی، شماره خود را در `allowFrom` قرار دهید (در این حالت @اشاره‌های بومی نادیده گرفته می‌شوند و فقط به الگوهای متنی پاسخ داده می‌شود):

```json5
{
  channels: {
    whatsapp: {
      allowFrom: ["+15555550123"],
      groups: { "*": { requireMention: true } },
    },
  },
  agents: {
    list: [
      {
        id: "main",
        groupChat: { mentionPatterns: ["reisponde", "@openclaw"] },
      },
    ],
  },
}
```

### فرمان‌ها (مدیریت فرمان‌های گفت‌وگو)

```json5
{
  commands: {
    native: "auto", // در صورت پشتیبانی، فرمان‌های بومی را ثبت می‌کند
    nativeSkills: "auto", // در صورت پشتیبانی، فرمان‌های بومی Skills را ثبت می‌کند
    text: true, // ‏/commands را در پیام‌های گفت‌وگو تجزیه می‌کند
    bash: false, // اجازه استفاده از ! (نام مستعار: /bash)
    bashForegroundMs: 2000,
    config: false, // اجازه استفاده از /config
    mcp: false, // اجازه استفاده از /mcp
    plugins: false, // اجازه استفاده از /plugins
    debug: false, // اجازه استفاده از /debug
    restart: true, // اجازه استفاده از /restart و درخواست‌های خارجی راه‌اندازی مجدد SIGUSR1
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw", // raw | hash
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<Accordion title="جزئیات فرمان">

- این بلوک سطوح فرمان را پیکربندی می‌کند. برای فهرست فعلی فرمان‌های داخلی و همراه، به [فرمان‌های اسلش](/fa/tools/slash-commands) مراجعه کنید.
- این صفحه **مرجع کلیدهای پیکربندی** است، نه فهرست کامل فرمان‌ها. فرمان‌های متعلق به کانال/Plugin، مانند QQ Bot `/bot-ping` `/bot-help` `/bot-logs`، ‏LINE `/card`، جفت‌سازی دستگاه `/pair`، حافظه `/dreaming`، کنترل تلفن `/phone` و Talk `/voice`، در صفحات کانال/Plugin مربوطه و نیز [فرمان‌های اسلش](/fa/tools/slash-commands) مستند شده‌اند.
- فرمان‌های متنی باید پیام‌های **مستقل** با `/` در ابتدای خود باشند.
- `native: "auto"` فرمان‌های بومی را برای Discord/Telegram فعال و برای Slack غیرفعال نگه می‌دارد.
- `nativeSkills: "auto"` فرمان‌های بومی Skills را برای Discord/Telegram فعال و برای Slack غیرفعال نگه می‌دارد.
- بازنویسی به‌ازای هر کانال: `channels.discord.commands.native` (مقدار بولی یا `"auto"`). برای Discord، ‏`false` ثبت و پاک‌سازی فرمان‌های بومی را هنگام راه‌اندازی رد می‌کند.
- ثبت بومی Skills را به‌ازای هر کانال با `channels.<provider>.commands.nativeSkills` بازنویسی کنید.
- `channels.telegram.customCommands` ورودی‌های بیشتری به منوی ربات Telegram اضافه می‌کند.
- `bash: true`، ‏`! <cmd>` را برای پوسته میزبان فعال می‌کند. به `tools.elevated.enabled` و حضور فرستنده در `tools.elevated.allowFrom.<channel>` نیاز دارد.
- `config: true`، ‏`/config` را فعال می‌کند (`openclaw.json` را می‌خواند/می‌نویسد). برای کلاینت‌های `chat.send` در Gateway، نوشتن پایدار `/config set|unset` به `operator.admin` نیز نیاز دارد؛ `/config show` فقط‌خواندنی برای کلاینت‌های عادی اپراتور با دامنه نوشتن همچنان در دسترس است.
- `mcp: true`، ‏`/mcp` را برای پیکربندی سرور MCP مدیریت‌شده توسط OpenClaw در `mcp.servers` فعال می‌کند.
- `plugins: true`، ‏`/plugins` را برای کشف و نصب Plugin و کنترل‌های فعال‌سازی/غیرفعال‌سازی فعال می‌کند.
- `channels.<provider>.configWrites` تغییرات پیکربندی را به‌ازای هر کانال محدود می‌کند (پیش‌فرض: true).
- برای کانال‌های چندحسابی، `channels.<provider>.accounts.<id>.configWrites` نوشتن‌هایی را نیز محدود می‌کند که آن حساب را هدف قرار می‌دهند (برای مثال `/allowlist --config --account <id>` یا `/config set channels.<provider>.accounts.<id>...`).
- `restart: false`، ‏`/restart` و درخواست‌های خارجی راه‌اندازی مجدد `SIGUSR1` را غیرفعال می‌کند. پیش‌فرض: `true`.
- `ownerAllowFrom` فهرست مجاز صریح مالک برای فرمان‌های مختص مالک و کنش‌های کانالی محدودشده به مالک است. این فهرست از `allowFrom` جداست.
- `ownerDisplay: "hash"` شناسه‌های مالک را در پرامپت سیستم هش می‌کند. برای کنترل هش‌کردن، `ownerDisplaySecret` را تنظیم کنید.
- `allowFrom` به‌ازای هر ارائه‌دهنده است. وقتی تنظیم شود، **تنها** منبع مجوزدهی است (فهرست‌های مجاز کانال/جفت‌سازی و `useAccessGroups` نادیده گرفته می‌شوند).
- `useAccessGroups: false` به فرمان‌ها اجازه می‌دهد وقتی `allowFrom` تنظیم نشده است، خط‌مشی‌های گروه دسترسی را دور بزنند.
- نقشه مستندات فرمان‌ها:
  - فهرست داخلی و همراه: [فرمان‌های اسلش](/fa/tools/slash-commands)
  - سطوح فرمان مختص کانال: [کانال‌ها](/fa/channels)
  - فرمان‌های QQ Bot:‏ [QQ Bot](/fa/channels/qqbot)
  - فرمان‌های جفت‌سازی: [جفت‌سازی](/fa/channels/pairing)
  - فرمان کارت LINE:‏ [LINE](/fa/channels/line)
  - Dreaming حافظه: [Dreaming](/fa/concepts/dreaming)

</Accordion>

---

## مرتبط

- [مرجع پیکربندی](/fa/gateway/configuration-reference) — کلیدهای سطح بالا
- [پیکربندی — عامل‌ها](/fa/gateway/config-agents)
- [نمای کلی کانال‌ها](/fa/channels)
