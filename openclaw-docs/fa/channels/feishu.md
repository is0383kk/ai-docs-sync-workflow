---
read_when:
    - می‌خواهید یک ربات Feishu/Lark را متصل کنید
    - در حال پیکربندی کانال Feishu هستید
summary: نمای کلی، قابلیت‌ها و پیکربندی ربات Feishu
title: فیشو
x-i18n:
    generated_at: "2026-07-27T16:12:18Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5e7c4cbb704ce266b7c7b0f6e160c36c873050fee8d5808965e15b56ad637f28
    source_path: channels/feishu.md
    workflow: 16
---

OpenClaw از طریق Plugin رسمی `@openclaw/feishu` به Feishu/Lark (پلتفرم جامع همکاری) متصل می‌شود: پیام‌های مستقیم ربات، گفت‌وگوهای گروهی، پاسخ‌های جریانی کارت و ابزارهای سند/ویکی/درایو/Bitable در Feishu.

**وضعیت:** آمادهٔ استفاده در محیط عملیاتی برای پیام‌های مستقیم ربات و گفت‌وگوهای گروهی. WebSocket روش پیش‌فرض انتقال رویداد است (به URL عمومی نیاز ندارد)؛ حالت webhook اختیاری است.

## شروع سریع

<Note>
به OpenClaw 2026.5.29 یا بالاتر نیاز دارد. برای بررسی، `openclaw --version` را اجرا کنید. با `openclaw update` ارتقا دهید.
</Note>

<Steps>
  <Step title="اجرای راه‌انداز تنظیم کانال">
  ```bash
  openclaw channels login --channel feishu
  ```
  اگر Plugin ‏`@openclaw/feishu` موجود نباشد، این فرمان آن را نصب می‌کند و سپس مراحل راه‌اندازی را پیش می‌برد:

- **راه‌اندازی دستی**: یک App ID و App Secret از Feishu Open Platform ‏(`https://open.feishu.cn`) یا Lark Developer ‏(`https://open.larksuite.com`) وارد کنید.
- **راه‌اندازی با QR**: برای ایجاد خودکار ربات، یک کد QR را در برنامهٔ Feishu اسکن کنید. این جریان، پیام‌های مستقیم را به حساب خودتان محدود می‌کند (`dmPolicy: "allowlist"` با `open_id` خودتان).

راه‌انداز همچنین دامنهٔ API ‏(Feishu یا Lark) و خط‌مشی گروه را می‌پرسد. اگر برنامهٔ موبایل داخلی Feishu به کد QR واکنش نشان نداد، راه‌اندازی را دوباره اجرا و راه‌اندازی دستی را انتخاب کنید.
</Step>

  <Step title="پس از تکمیل راه‌اندازی، برای اعمال تغییرات Gateway را راه‌اندازی مجدد کنید">
  ```bash
  openclaw gateway restart
  ```
  </Step>
</Steps>

## دوام ورودی‌ها

OpenClaw پیش از ارسال به عامل، پاکت‌های احرازهویت‌شدهٔ `im.message.receive_v1` و `drive.notice.comment_add_v1` را به‌صورت پایدار در صف قرار می‌دهد. رویدادهای در انتظار یا قابل‌تلاش‌مجدد پس از راه‌اندازی مجدد Gateway باقی می‌مانند، برای هر گفت‌وگو یا سند به‌شکل سریالی پردازش می‌شوند و تا زمانی که رکورد تکمیل فعال یا نگه‌داری‌شده وجود دارد، با استفاده از شناسهٔ رویداد Feishu از ورود موارد تکراری به صف جلوگیری می‌کنند.

اگر پس از تعداد محدودی تلاش، رویداد WebSocket قابل ذخیره‌سازی نباشد، OpenClaw آن سوکت را می‌بندد و به‌جای ادامه‌دادن پس از یک نوبت ثبت‌نشده، اتصال احرازهویت‌شدهٔ جدیدی را اجباری می‌کند. سایر انواع رویداد Feishu، از جمله واکنش‌ها و دعوت‌نامه‌های جلسهٔ VC، از مسیرهای عادی رویداد خود استفاده می‌کنند و مشمول این تضمین صف پایدار نیستند.

## کنترل دسترسی

### پیام‌های مستقیم

برای کنترل افرادی که می‌توانند به ربات پیام مستقیم بدهند، `channels.feishu.dmPolicy` (پیش‌فرض: `pairing`) را پیکربندی کنید:

| مقدار         | رفتار                                                                                                      |
| ------------- | ------------------------------------------------------------------------------------------------------------- |
| `"pairing"`   | کاربران ناشناس یک کد جفت‌سازی دریافت می‌کنند؛ آن را از طریق CLI تأیید کنید                                                         |
| `"allowlist"` | فقط کاربران فهرست‌شده در `allowFrom` می‌توانند گفت‌وگو کنند                                                                     |
| `"open"`      | پیام‌های مستقیم عمومی؛ اعتبارسنجی پیکربندی مستلزم آن است که `allowFrom` شامل `"*"` باشد. ورودی‌های غیرعام همچنان دسترسی را محدود می‌کنند |

**تأیید درخواست جفت‌سازی:**

```bash
openclaw pairing list feishu
openclaw pairing approve feishu <CODE>
```

### گفت‌وگوهای گروهی

**خط‌مشی گروه** (`channels.feishu.groupPolicy`، پیش‌فرض: `allowlist`):

| مقدار         | رفتار                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------- |
| `"open"`      | پاسخ‌دادن به همهٔ پیام‌ها در گروه‌ها                                                            |
| `"allowlist"` | فقط پاسخ‌دادن به گروه‌های موجود در `groupAllowFrom` یا گروه‌هایی که صریحاً در `groups.<chat_id>` پیکربندی شده‌اند |
| `"disabled"`  | غیرفعال‌کردن همهٔ پیام‌های گروهی؛ ورودی‌های صریح `groups.<chat_id>` این مورد را لغو نمی‌کنند         |

**الزام اشاره** (`channels.feishu.requireMention`):

- پیش‌فرض: ‎@mention الزامی است، مگر زمانی که خط‌مشی مؤثر گروه `"open"` باشد؛ در آن حالت مقدار پیش‌فرض `false` است تا پیام‌هایی که نمی‌توانند اشاره داشته باشند (برای مثال تصاویر) همچنان به عامل برسند.
- برای بازنویسی صریح، `true` یا `false` را تنظیم کنید؛ بازنویسی برای هر گروه: `channels.feishu.groups.<chat_id>.requireMention`.
- اشاره‌های صرفاً همگانی `@all` و `@_all` به‌عنوان اشاره به ربات محسوب نمی‌شوند. پیامی که هم به `@all` و هم مستقیماً به ربات اشاره کند، همچنان اشاره به ربات محسوب می‌شود.

## نمونه‌های پیکربندی گروه

### اجازه به همهٔ گروه‌ها، بدون نیاز به ‎@mention

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open", // requireMention defaults to false under "open"
    },
  },
}
```

### اجازه به همهٔ گروه‌ها، همچنان با الزام ‎@mention

```json5
{
  channels: {
    feishu: {
      groupPolicy: "open",
      requireMention: true,
    },
  },
}
```

### اجازه فقط به گروه‌های مشخص

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      // Group IDs look like: oc_xxx
      groupAllowFrom: ["oc_xxx", "oc_yyy"],
    },
  },
}
```

در حالت `allowlist`، با افزودن یک ورودی صریح `groups.<chat_id>` نیز می‌توانید گروهی را بپذیرید. ورودی‌های صریح، `groupPolicy: "disabled"` را لغو نمی‌کنند. پیش‌فرض‌های عام زیر `groups.*` گروه‌های منطبق را پیکربندی می‌کنند، اما به‌تنهایی باعث پذیرش گروه‌ها نمی‌شوند.

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groups: {
        oc_xxx: {
          requireMention: false,
        },
      },
    },
  },
}
```

### محدودکردن فرستندگان درون یک گروه

```json5
{
  channels: {
    feishu: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx"],
      groups: {
        oc_xxx: {
          // User open_ids look like: ou_xxx
          allowFrom: ["ou_user1", "ou_user2"],
        },
      },
    },
  },
}
```

`channels.feishu.groupSenderAllowFrom` فهرست مجاز فرستندگان یکسانی را برای همهٔ گروه‌ها تنظیم می‌کند؛ `allowFrom` مربوط به هر گروه اولویت دارد.

### پیام‌های نوشته‌شده توسط ربات

Feishu به‌طور پیش‌فرض پیام‌های نوشته‌شده توسط ربات‌های دیگر را نادیده می‌گیرد. برای اجازه‌دادن به گفت‌وگوهای گروهی ربات‌به‌ربات، دامنه‌های دسترسی `im:message.group_at_msg.include_bot:readonly` و `im:message:readonly` را به برنامه اعطا کنید، سپس `allowBots` را تنظیم کنید:

```json5
{
  channels: {
    feishu: {
      allowBots: true,
    },
  },
}
```

Feishu فقط زمانی رویدادهای گروهی نوشته‌شده توسط ربات را تحویل می‌دهد که ربات دیگری به این ربات اشاره کند. خط‌مشی گروه، فهرست‌های مجاز فرستنده و الزامات اشارهٔ موجود همچنان اعمال می‌شوند. OpenClaw پیام‌های نوشته‌شده توسط خودش را حذف می‌کند، در هر پاسخ متنی یا کارتی به ربات همتا اشاره می‌کند و محافظ مشترک [`channels.defaults.botLoopProtection`](/fa/channels/bot-loop-protection) را اعمال می‌کند.

<a id="get-groupuser-ids"></a>

## دریافت شناسه‌های گروه/کاربر

### شناسه‌های گروه (`chat_id`، قالب: `oc_xxx`)

گروه را در Feishu/Lark باز کنید، روی نماد منو در گوشهٔ بالا-راست کلیک کنید و به **Settings** بروید. شناسهٔ گروه (`chat_id`) در صفحهٔ تنظیمات فهرست شده است.

![دریافت شناسهٔ گروه](/images/feishu-get-group-id.png)

### شناسه‌های کاربر (`open_id`، قالب: `ou_xxx`)

Gateway را راه‌اندازی کنید، یک پیام مستقیم به ربات بفرستید، سپس گزارش‌ها را بررسی کنید:

```bash
openclaw logs --follow
```

در خروجی گزارش به‌دنبال `open_id` بگردید. همچنین می‌توانید درخواست‌های جفت‌سازی در انتظار را بررسی کنید:

```bash
openclaw pairing list feishu
```

## فرمان‌های متداول

| فرمان   | توضیح                 |
| --------- | --------------------------- |
| `/status` | نمایش وضعیت ربات             |
| `/reset`  | بازنشانی نشست فعلی   |
| `/model`  | نمایش یا تغییر مدل هوش مصنوعی |

<Note>
Feishu/Lark از منوهای بومی فرمان اسلش پشتیبانی نمی‌کند؛ بنابراین این موارد را به‌صورت پیام‌های متنی ساده ارسال کنید.
</Note>

## عیب‌یابی

### ربات در گفت‌وگوهای گروهی پاسخ نمی‌دهد

1. مطمئن شوید ربات به گروه اضافه شده است
2. مطمئن شوید با ‎@mention به ربات اشاره می‌کنید (به‌طور پیش‌فرض الزامی است)
3. بررسی کنید `groupPolicy` برابر با `"disabled"` نباشد
4. گزارش‌ها را بررسی کنید: `openclaw logs --follow`

### ربات پیام‌ها را دریافت نمی‌کند

1. مطمئن شوید ربات در Feishu Open Platform / Lark Developer منتشر و تأیید شده است
2. مطمئن شوید اشتراک رویداد شامل `im.message.receive_v1` است
3. برای پیوستن خودکار به دعوت جلسه، `vc.bot.meeting_invited_v1` را نیز مشترک شوید
4. مطمئن شوید **persistent connection** ‏(WebSocket) انتخاب شده است
5. مطمئن شوید همهٔ دامنه‌های دسترسی لازم اعطا شده‌اند
6. مطمئن شوید Gateway در حال اجرا است: `openclaw gateway status`
7. گزارش‌ها را بررسی کنید: `openclaw logs --follow`

اشتراک در `vc.bot.meeting_invited_v1` فقط رویداد را تحویل می‌دهد. پیوستن خودکار به‌طور
پیش‌فرض غیرفعال است. برای فعال‌کردن سراسری آن:

```json5
{
  channels: {
    feishu: {
      vcAutoJoin: true,
    },
  },
}
```

برای فعال‌کردن فقط یک حساب، کلید سطح بالا را حذف کنید و بازنویسی حساب را تنظیم کنید:

```json5
{
  channels: {
    feishu: {
      accounts: {
        meetings: { vcAutoJoin: true },
      },
    },
  },
}
```

پیش از آنکه عامل نوبت پیوستن را دریافت کند، دعوت‌کنندگان همچنان از خط‌مشی عادی پیام مستقیم Feishu، فهرست مجاز/جفت‌سازی، نشست و مسیریابی
پاسخ عبور می‌کنند. پیوستن همچنین به یک ابزار در دسترس پیوستن به VC در Feishu نیاز دارد
که برای هویت برنامه با دامنهٔ دسترسی
`vc:meeting.bot.join:write` پیکربندی شده باشد. برای مثال، Skills رسمی
[عامل VC ‏`lark-cli`](https://github.com/larksuite/cli/tree/main/skills/lark-vc-agent)
مقدار `vc +meeting-join` را فراهم می‌کند.

<Warning>
Skills رسمی عامل VC ‏`lark-cli` در حال حاضر اقدامات ربات جلسه را به‌عنوان نسخهٔ بتای محدود مشخص می‌کند. اگر ابزار `ErrNotInGray` یا کد خطای `20017` را برگرداند، برنامه یا مستأجر برای آن نسخهٔ بتا فعال نشده است؛ پیش از عیب‌یابی اعطای عادی دامنه‌های دسترسی، از راهنمای دسترسی زودهنگام در Skills پیوندشده استفاده کنید.
</Warning>

### راه‌اندازی با QR در برنامهٔ موبایل Feishu واکنش نشان نمی‌دهد

1. راه‌اندازی را دوباره اجرا کنید: `openclaw channels login --channel feishu`
2. راه‌اندازی دستی را انتخاب کنید
3. در Feishu Open Platform، یک برنامهٔ خودساخته ایجاد و App ID و App Secret آن را کپی کنید
4. آن اطلاعات ورود را در راه‌انداز تنظیم وارد کنید

### App Secret افشا شده است

1. App Secret را در Feishu Open Platform / Lark Developer بازنشانی کنید
2. مقدار را در پیکربندی خود به‌روزرسانی کنید
3. Gateway را راه‌اندازی مجدد کنید: `openclaw gateway restart`

## پیکربندی پیشرفته

### چند حساب

```json5
{
  channels: {
    feishu: {
      defaultAccount: "main",
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "xxx",
          name: "Primary bot",
          tts: {
            providers: {
              openai: { voice: "shimmer" },
            },
          },
        },
        backup: {
          appId: "cli_yyy",
          appSecret: "yyy",
          name: "Backup bot",
          enabled: false,
        },
      },
    },
  },
}
```

`defaultAccount` تعیین می‌کند وقتی APIهای خروجی یک `accountId` را مشخص نمی‌کنند، از کدام حساب استفاده شود. ورودی‌های حساب، تنظیمات سطح بالا را به ارث می‌برند؛ بیشتر کلیدهای سطح بالا را می‌توان برای هر حساب بازنویسی کرد.
`accounts.<id>.tts` از همان ساختار `tts` استفاده می‌کند و با ادغام عمیق روی پیکربندی سراسری TTS اعمال می‌شود؛ بنابراین راه‌اندازی‌های چندرباتهٔ Feishu می‌توانند اطلاعات ورود مشترک ارائه‌دهنده را به‌صورت سراسری نگه دارند و فقط صدا، مدل، شخصیت یا حالت خودکار را برای هر حساب بازنویسی کنند.

### محدودیت‌های پیام

- `textChunkLimit` - اندازهٔ بخش متن خروجی (پیش‌فرض: `4000` نویسه)
- `streaming.chunkMode` - `"length"` (پیش‌فرض) در حد تعیین‌شده تقسیم می‌کند؛ `"newline"` مرزهای خط جدید را ترجیح می‌دهد
- `mediaMaxMb` - محدودیت بارگذاری/دریافت رسانه (پیش‌فرض: `30` مگابایت)

### جریان‌سازی

Feishu/Lark از پاسخ‌های جریانی از طریق کارت‌های تعاملی (API جریان‌سازی Card Kit) پشتیبانی می‌کند. وقتی فعال باشد، ربات هم‌زمان با تولید متن، کارت را در لحظه به‌روزرسانی می‌کند.

```json5
{
  channels: {
    feishu: {
      streaming: {
        mode: "partial", // streaming card output (default: "partial")
        block: { enabled: true }, // opt into completed-block streaming
      },
    },
  },
}
```

برای ارسال پاسخ کامل در یک پیام، `streaming.mode: "off"` را تنظیم کنید؛ `renderMode: "raw"` (متن ساده به‌جای کارت‌ها) نیز کارت‌های جریانی را غیرفعال می‌کند. `streaming.block.enabled` به‌طور پیش‌فرض غیرفعال است؛ آن را فقط زمانی فعال کنید که می‌خواهید بلوک‌های تکمیل‌شدهٔ دستیار پیش از پاسخ نهایی ارسال شوند. مقدار بولی قدیمی `streaming` و کلیدهای مسطح `blockStreaming` / `blockStreamingCoalesce` / `chunkMode` از طریق `openclaw doctor --fix` به این ساختار تودرتو مهاجرت می‌کنند.

### بهینه‌سازی سهمیه

با دو پرچم اختیاری، تعداد فراخوانی‌های API مربوط به Feishu/Lark را کاهش دهید:

- `typingIndicator` (پیش‌فرض `true`): برای صرف‌نظر کردن از فراخوانی‌های واکنش تایپ، `false` را تنظیم کنید
- `resolveSenderNames` (پیش‌فرض `true`): برای صرف‌نظر کردن از جست‌وجوی پروفایل فرستنده، `false` را تنظیم کنید

```json5
{
  channels: {
    feishu: {
      typingIndicator: false,
      resolveSenderNames: false,
    },
  },
}
```

### دامنهٔ نشست گروه و رشته‌های موضوعی

`channels.feishu.groupSessionScope` (در سطح بالا، برای هر حساب یا برای هر گروه) نحوهٔ نگاشت پیام‌های گروه به نشست‌های عامل را کنترل می‌کند:

| مقدار                  | نشست                                                          |
| ---------------------- | ---------------------------------------------------------------- |
| `"group"` (پیش‌فرض)    | یک نشست برای هر گفت‌وگوی گروهی                                       |
| `"group_sender"`       | یک نشست برای هر (گروه + فرستنده)                                 |
| `"group_topic"`        | یک نشست برای هر رشتهٔ موضوعی؛ در صورت نبود، از نشست گروه استفاده می‌کند    |
| `"group_topic_sender"` | یک نشست برای هر (موضوع + فرستنده)؛ در صورت نبود، از (گروه + فرستنده) استفاده می‌کند |

برای دامنه‌های موضوعی، گروه‌های موضوعی بومی Feishu/Lark از رویداد `thread_id` (`omt_*`) به‌عنوان کلید متعارف نشست موضوع استفاده می‌کنند. اگر رویداد آغازگر یک موضوع بومی فاقد `thread_id` باشد، OpenClaw پیش از مسیریابی نوبت، آن را از Feishu دریافت و تکمیل می‌کند. پاسخ‌های عادی گروه که OpenClaw آن‌ها را به رشته تبدیل می‌کند، همچنان از شناسهٔ پیام ریشهٔ پاسخ (`om_*`) استفاده می‌کنند تا نوبت نخست و نوبت‌های بعدی در همان نشست باقی بمانند.

`replyInThread: "enabled"` را (در سطح بالا یا برای هر گروه) تنظیم کنید تا پاسخ‌های ربات به‌جای پاسخ درون‌خطی، یک رشتهٔ موضوعی Feishu ایجاد کنند یا ادامه دهند. `topicSessionMode` نسخهٔ منسوخ‌شدهٔ پیشین `groupSessionScope` است؛ `groupSessionScope` را ترجیح دهید.

### ابزارهای فضای کاری Feishu

Plugin ابزارهای عامل را برای اسناد، گفت‌وگوها، پایگاه دانش، فضای ذخیره‌سازی ابری، مجوزها و Bitable در Feishu، به‌همراه Skills متناظر (`feishu-doc`، `feishu-drive`، `feishu-perm`، `feishu-wiki`) ارائه می‌کند. خانواده‌های ابزار با `channels.feishu.tools` کنترل می‌شوند:

| کلید             | ابزارها                                         | پیش‌فرض             |
| --------------- | --------------------------------------------- | ------------------- |
| `tools.doc`     | عملیات اسناد `feishu_doc`              | `true`              |
| `tools.chat`    | اطلاعات گفت‌وگو + پرس‌وجوهای اعضای `feishu_chat`      | `true`              |
| `tools.wiki`    | پایگاه دانش `feishu_wiki` (نیازمند `doc`) | `true`              |
| `tools.drive`   | فضای ذخیره‌سازی ابری `feishu_drive`                  | `true`              |
| `tools.perm`    | مدیریت مجوز `feishu_perm`           | `false` (حساس) |
| `tools.scopes`  | عیب‌یابی دامنهٔ برنامه `feishu_app_scopes`     | `true`              |
| `tools.bitable` | عملیات Bitable/Base مربوط به `feishu_bitable_*`    | `true`              |

`tools.base` نام مستعار `tools.bitable` است؛ وقتی هر دو تنظیم شده باشند، مقدار صریح `bitable` اولویت دارد. کنترل‌های هر حساب زیر `accounts.<id>.tools` قرار دارند.

برای جست‌وجوهای مستقیم `feishu_drive info` خارج از پوشهٔ ریشه، مجوز `drive:drive.metadata:readonly` را اعطا کنید؛ مگر اینکه برنامه از قبل دامنهٔ کامل `drive:drive` را داشته باشد. بدون هیچ‌یک از این دامنه‌ها، `info`
جست‌وجوی قدیمی پوشهٔ ریشه را از طریق `drive:drive:readonly` در دسترس نگه می‌دارد.

### نشست‌های ACP

Feishu/Lark از ACP برای پیام‌های مستقیم و پیام‌های رشته‌ای گروه پشتیبانی می‌کند. ACP در Feishu/Lark مبتنی بر فرمان متنی است؛ منوی بومی فرمان اسلش وجود ندارد، بنابراین پیام‌های `/acp ...` را مستقیماً در مکالمه استفاده کنید.

#### اتصال پایدار ACP

```json5
{
  agents: {
    list: [
      {
        id: "codex",
        runtime: {
          type: "acp",
          acp: {
            agent: "codex",
            backend: "acpx",
            mode: "persistent",
            cwd: "/workspace/openclaw",
          },
        },
      },
    ],
  },
  bindings: [
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "direct", id: "ou_1234567890" },
      },
    },
    {
      type: "acp",
      agentId: "codex",
      match: {
        channel: "feishu",
        accountId: "default",
        peer: { kind: "group", id: "oc_group_chat:topic:om_topic_root" },
      },
      acp: { label: "codex-feishu-topic" },
    },
  ],
}
```

#### ایجاد ACP از گفت‌وگو

در یک پیام مستقیم یا رشتهٔ Feishu/Lark:

```text
/acp spawn codex --thread here
```

`--thread here` برای پیام‌های مستقیم و پیام‌های رشته‌ای Feishu/Lark کار می‌کند. پیام‌های بعدی در مکالمهٔ متصل‌شده مستقیماً به همان نشست ACP مسیریابی می‌شوند.

### مسیریابی چندعاملی

برای مسیریابی پیام‌های مستقیم یا گروه‌های Feishu/Lark به عامل‌های مختلف، از `bindings` استفاده کنید.

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
        channel: "feishu",
        peer: { kind: "direct", id: "ou_xxx" },
      },
    },
    {
      agentId: "agent-b",
      match: {
        channel: "feishu",
        peer: { kind: "group", id: "oc_zzz" },
      },
    },
  ],
}
```

فیلدهای مسیریابی:

- `match.channel`: `"feishu"`
- `match.peer.kind`: `"direct"` (پیام مستقیم) یا `"group"` (گفت‌وگوی گروهی)
- `match.peer.id`: Open ID کاربر (`ou_xxx`) یا شناسهٔ گروه (`oc_xxx`)

برای نکات جست‌وجو، به [دریافت شناسه‌های گروه/کاربر](#get-groupuser-ids) مراجعه کنید.

## جداسازی عامل برای هر کاربر (ایجاد پویای عامل)

`dynamicAgentCreation` را فعال کنید تا برای هر کاربر پیام مستقیم، **نمونه‌های عامل ایزوله** به‌طور خودکار ایجاد شوند. هر کاربر موارد زیر را به‌صورت اختصاصی دریافت می‌کند:

- پوشهٔ فضای کاری مستقل
- `USER.md` / `SOUL.md` / `MEMORY.md` مجزا
- تاریخچهٔ خصوصی مکالمه
- Skills و وضعیت ایزوله

این قابلیت برای ربات‌های عمومی که می‌خواهید هر کاربر در آن‌ها تجربهٔ دستیار هوش مصنوعی خصوصی خود را داشته باشد، ضروری است.

<Note>
اتصال‌های پویا شامل `accountId` نرمال‌شدهٔ Feishu هستند، بنابراین حساب‌های پیش‌فرض و نام‌گذاری‌شده هر فرستنده را به عامل پویای صحیح مسیریابی می‌کنند.

اگر یک حساب نام‌گذاری‌شده در نسخه‌ای قدیمی‌تر عامل پویای بدون دامنه ایجاد کرده باشد، آن عامل قدیمی همچنان در `maxAgents` محاسبه می‌شود. پیش از حذف آن، مطمئن شوید که حساب پیش‌فرض از آن استفاده نمی‌کند؛ یا `maxAgents` را موقتاً افزایش دهید. OpenClaw نمی‌تواند با اطمینان تشخیص دهد که وضعیت مبهم قدیمی متعلق به کدام حساب است.
</Note>

### راه‌اندازی سریع

```json5
{
  channels: {
    feishu: {
      dmPolicy: "open",
      allowFrom: ["*"],
      dynamicAgentCreation: {
        enabled: true,
        workspaceTemplate: "~/.openclaw/workspace-{agentId}",
        agentDirTemplate: "~/.openclaw/agents/{agentId}/agent",
      },
    },
  },
  session: {
    // حیاتی: پیام مستقیم هر کاربر را به «نشست اصلی» او تبدیل می‌کند
    // USER.md / SOUL.md / MEMORY.md را به‌طور خودکار بارگیری می‌کند
    // برای جداسازی قوی‌تر، به‌جای آن از "per-channel-peer" استفاده کنید
    dmScope: "main",
  },
}
```

### نحوهٔ کار

وقتی کاربر جدید نخستین پیام مستقیم خود را ارسال می‌کند:

1. کانال یک `agentId` یکتا تولید می‌کند: `feishu-{user_open_id}` برای حساب پیش‌فرض، یا یک چکیدهٔ هویت محدودشده با پیشوند حساب برای حساب نام‌گذاری‌شده
2. یک فضای کاری جدید در مسیر `workspaceTemplate` ایجاد می‌کند
3. عامل را ثبت می‌کند و برای این کاربر یک اتصال می‌سازد
4. کمک‌ابزار فضای کاری در نخستین دسترسی، وجود فایل‌های راه‌اندازی اولیه (`AGENTS.md`، `SOUL.md`، `USER.md` و غیره) را تضمین می‌کند
5. همهٔ پیام‌های آیندهٔ این کاربر را به عامل اختصاصی او مسیریابی می‌کند

### گزینه‌های پیکربندی

| تنظیم                                                  | توضیحات                                | پیش‌فرض                              |
| -------------------------------------------------------- | ------------------------------------------ | ------------------------------------ |
| `channels.feishu.dynamicAgentCreation.enabled`           | فعال‌سازی ایجاد خودکار عامل برای هر کاربر   | `false`                              |
| `channels.feishu.dynamicAgentCreation.workspaceTemplate` | الگوی مسیر فضاهای کاری عامل پویا | `~/.openclaw/workspace-{agentId}`    |
| `channels.feishu.dynamicAgentCreation.agentDirTemplate`  | الگوی نام پوشهٔ عامل              | `~/.openclaw/agents/{agentId}/agent` |
| `channels.feishu.dynamicAgentCreation.maxAgents`         | حداکثر تعداد عامل‌های پویای قابل ایجاد | نامحدود                            |

متغیرهای الگو:

- `{agentId}` - شناسهٔ عامل تولیدشده (برای مثال، `feishu-ou_xxxxxx` یا `feishu-support-<identity_digest>`)
- `{userId}` - شناسهٔ open_id فرستنده در Feishu (برای مثال، `ou_xxxxxx`)

### دامنهٔ نشست

`session.dmScope` نحوهٔ نگاشت پیام‌های مستقیم به نشست‌های عامل را کنترل می‌کند. این یک **تنظیم سراسری** است که بر همهٔ کانال‌ها اثر می‌گذارد.

| مقدار                        | رفتار                                                            | مناسب برای                                                           |
| ---------------------------- | ------------------------------------------------------------------- | ------------------------------------------------------------------ |
| `"main"`                     | پیام مستقیم هر کاربر به نشست اصلی عامل او نگاشت می‌شود                   | ربات‌های تک‌کاربره‌ای که می‌خواهید `USER.md` / `SOUL.md` در آن‌ها به‌طور خودکار بارگیری شود |
| `"per-peer"`                 | هر همتا یک نشست جداگانه دریافت می‌کند (صرف‌نظر از کانال)           | جداسازی صرفاً بر اساس هویت فرستنده                            |
| `"per-channel-peer"`         | هر ترکیب (کانال + کاربر) یک نشست جداگانه دریافت می‌کند           | ربات‌های عمومی چندکاربره‌ای که به جداسازی قوی‌تر نیاز دارند                  |
| `"per-account-channel-peer"` | هر ترکیب (حساب + کانال + کاربر) یک نشست جداگانه دریافت می‌کند | ربات‌های چندحسابی که به جداسازی نشست در سطح حساب نیاز دارند         |

**موازنه**: استفاده از `"main"` بارگیری خودکار فایل‌های راه‌اندازی اولیه (`USER.md`، `SOUL.md`، `MEMORY.md`) را فعال می‌کند، اما به این معناست که همهٔ پیام‌های مستقیم در تمام کانال‌ها الگوی کلید نشست یکسانی دارند. برای ربات‌های عمومی چندکاربره که جداسازی در آن‌ها مهم‌تر از بارگیری خودکار فایل‌های راه‌اندازی اولیه است، `"per-channel-peer"` را در نظر بگیرید و فایل‌های راه‌اندازی اولیه را به‌صورت دستی مدیریت کنید.

<Note>
وقتی حساب‌های نام‌گذاری‌شدهٔ Feishu باید برای یک فرستنده نشست‌های جداگانه نگه دارند، از `"per-account-channel-peer"` استفاده کنید. اتصال‌های پویا دامنهٔ حساب را حفظ می‌کنند.
</Note>

### استقرار معمول چندکاربره

```json5
{
  channels: {
    feishu: {
      appId: "cli_xxx",
      appSecret: "xxx",
      dmPolicy: "open",
      allowFrom: ["*"],
      groupPolicy: "open",
      requireMention: true,
      dynamicAgentCreation: {
        enabled: true,
        workspaceTemplate: "~/.openclaw/workspace-{agentId}",
        agentDirTemplate: "~/.openclaw/agents/{agentId}/agent",
      },
    },
  },
  session: {
    // dmScope را بر اساس نیازهای جداسازی خود انتخاب کنید:
    // "main" برای بارگیری خودکار فایل‌های راه‌اندازی اولیه، و "per-channel-peer" برای جداسازی قوی‌تر
    dmScope: "main",
  },
  bindings: [], // خالی — عامل‌های پویا به‌طور خودکار متصل می‌شوند
}
```

### تأیید

گزارش‌های Gateway را بررسی کنید تا مطمئن شوید ایجاد پویا به‌درستی کار می‌کند:

```text
feishu: در حال ایجاد عامل پویای "feishu-ou_xxxxxx" برای کاربر ou_xxxxxx
  فضای کاری: /home/user/.openclaw/workspace-feishu-ou_xxxxxx
  پوشهٔ عامل: /home/user/.openclaw/agents/feishu-ou_xxxxxx/agent
```

فهرست همهٔ فضاهای کاری ایجادشده:

```bash
ls -la ~/.openclaw/workspace-*
```

### نکات

- **جداسازی فضای کاری**: هر کاربر دایرکتوری فضای کاری و نمونهٔ عامل مخصوص خود را دارد. کاربران در جریان عادی پیام‌رسانی نمی‌توانند تاریخچهٔ مکالمه یا فایل‌های یکدیگر را ببینند.
- **مرز امنیتی**: این سازوکاری برای جداسازی زمینهٔ پیام‌رسانی است، نه یک مرز امنیتی در برابر هم‌مستأجر متخاصم. فرایند عامل و محیط میزبان مشترک هستند.
- **نوشتن پیکربندی باید فعال بماند**: ایجاد پویای عامل، عامل‌ها و اتصال‌ها را در پیکربندی می‌نویسد؛ وقتی `channels.feishu.configWrites` برابر با `false` باشد، این کار انجام نمی‌شود (پیش‌فرض: فعال).
- **`bindings` باید خالی باشد**: عامل‌های پویا اتصال‌های خود را به‌طور خودکار ثبت می‌کنند
- **مسیر ارتقا**: اتصال‌های دستی موجود در کنار عامل‌های پویا همچنان کار می‌کنند
- **`session.dmScope` سراسری است**: این تنظیم بر همهٔ کانال‌ها اثر می‌گذارد، نه فقط Feishu

## مرجع پیکربندی

پیکربندی کامل: [پیکربندی Gateway](/fa/gateway/configuration)

| تنظیم                                                  | توضیحات                                                                          | پیش‌فرض                              |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------ |
| `channels.feishu.enabled`                                | فعال/غیرفعال‌کردن کانال                                                           | `true`                               |
| `channels.feishu.domain`                                 | دامنهٔ API (`feishu`، `lark` یا یک URL پایهٔ `https://`)                             | `feishu`                             |
| `channels.feishu.connectionMode`                         | انتقال رویداد (`websocket` یا `webhook`)                                           | `websocket`                          |
| `channels.feishu.defaultAccount`                         | حساب پیش‌فرض برای مسیریابی خروجی                                                 | `default`                            |
| `channels.feishu.verificationToken`                      | برای حالت Webhook الزامی است                                                            | -                                    |
| `channels.feishu.encryptKey`                             | برای حالت Webhook الزامی است                                                            | -                                    |
| `channels.feishu.webhookPath`                            | مسیر مسیریابی Webhook                                                                   | `/feishu/events`                     |
| `channels.feishu.webhookHost`                            | میزبان اتصال Webhook                                                                    | `127.0.0.1`                          |
| `channels.feishu.webhookPort`                            | درگاه اتصال Webhook                                                                    | `3000`                               |
| `channels.feishu.accounts.<id>.appId`                    | شناسهٔ برنامه                                                                               | -                                    |
| `channels.feishu.accounts.<id>.appSecret`                | راز برنامه                                                                           | -                                    |
| `channels.feishu.accounts.<id>.domain`                   | بازنویسی دامنه برای هر حساب                                                          | `feishu`                             |
| `channels.feishu.accounts.<id>.tts`                      | بازنویسی TTS برای هر حساب                                                             | `tts`                                |
| `channels.feishu.dmPolicy`                               | سیاست پیام مستقیم (`pairing`، `allowlist`، `open`)                                           | `pairing`                            |
| `channels.feishu.allowFrom`                              | فهرست مجاز پیام مستقیم (فهرست open_id)                                                          | -                                    |
| `channels.feishu.groupPolicy`                            | سیاست گروه (`open`، `allowlist`، `disabled`)                                       | `allowlist`                          |
| `channels.feishu.groupAllowFrom`                         | فهرست مجاز گروه‌ها                                                                      | -                                    |
| `channels.feishu.groupSenderAllowFrom`                   | فهرست مجاز فرستندگان که بر همهٔ گروه‌ها اعمال می‌شود                                               | -                                    |
| `channels.feishu.requireMention`                         | الزام @mention در گروه‌ها                                                           | `true` (`false` وقتی سیاست `open` است)  |
| `channels.feishu.allowBots`                              | پذیرش ربات‌های دیگری که این ربات را منشن می‌کنند، همراه با محافظت در برابر حلقهٔ ربات‌ها                    | `false`                              |
| `channels.feishu.groups.<chat_id>.requireMention`        | بازنویسی @mention برای هر گروه؛ شناسه‌های صریح در حالت فهرست مجاز، گروه را نیز می‌پذیرند     | ارث‌بری‌شده                            |
| `channels.feishu.groups.<chat_id>.enabled`               | فعال/غیرفعال‌کردن یک گروه مشخص                                                      | `true`                               |
| `channels.feishu.groups.<chat_id>.allowFrom`             | فهرست مجاز فرستندگان برای هر گروه (`groupSenderAllowFrom` را بازنویسی می‌کند)                        | -                                    |
| `channels.feishu.groupSessionScope`                      | نگاشت نشست گروه (`group`، `group_sender`، `group_topic`، `group_topic_sender`) | `group`                              |
| `channels.feishu.replyInThread`                          | پاسخ‌های ربات رشته‌های موضوعی را ایجاد/ادامه می‌دهند (`disabled`، `enabled`)                    | `disabled`                           |
| `channels.feishu.reactionNotifications`                  | رویدادهای واکنش ورودی (`off`، `own`، `all`)                                        | `own`                                |
| `channels.feishu.vcAutoJoin`                             | پیوستن به جلسه‌های VC دعوت‌شده پس از مجوزدهی عادی پیام مستقیم                               | `false`                              |
| `channels.feishu.dynamicAgentCreation.enabled`           | فعال‌کردن ایجاد خودکار عامل برای هر کاربر                                             | `false`                              |
| `channels.feishu.dynamicAgentCreation.workspaceTemplate` | الگوی مسیر فضاهای کاری عامل‌های پویا                                           | `~/.openclaw/workspace-{agentId}`    |
| `channels.feishu.dynamicAgentCreation.agentDirTemplate`  | الگوی نام دایرکتوری عامل                                                        | `~/.openclaw/agents/{agentId}/agent` |
| `channels.feishu.dynamicAgentCreation.maxAgents`         | حداکثر تعداد عامل‌های پویا برای ایجاد                                           | نامحدود                            |
| `channels.feishu.textChunkLimit`                         | اندازهٔ قطعهٔ پیام                                                                   | `4000`                               |
| `channels.feishu.streaming.chunkMode`                    | تقسیم قطعه‌ها (`length` یا `newline`)                                              | `length`                             |
| `channels.feishu.mediaMaxMb`                             | محدودیت اندازهٔ رسانه                                                                     | `30`                                 |
| `channels.feishu.renderMode`                             | رندر پاسخ (`auto`، `raw`، `card`)                                              | `auto`                               |
| `channels.feishu.streaming.mode`                         | خروجی کارت جریانی (`partial` یا `off`)                                           | `partial`                            |
| `channels.feishu.streaming.block.enabled`                | پخش جریانی پاسخ برای بلوک‌های تکمیل‌شده                                                      | `false`                              |
| `channels.feishu.typingIndicator`                        | ارسال واکنش‌های در حال تایپ                                                                | `true`                               |
| `channels.feishu.resolveSenderNames`                     | تفکیک نام‌های نمایشی فرستنده                                                         | `true`                               |
| `channels.feishu.configWrites`                           | اجازهٔ نوشتن پیکربندی آغازشده از کانال (موردنیاز عامل‌های پویا)                     | `true`                               |
| `channels.feishu.tools.doc`                              | فعال‌کردن ابزارهای سند                                                                | `true`                               |
| `channels.feishu.tools.chat`                             | فعال‌کردن ابزارهای اطلاعات گفت‌وگو                                                               | `true`                               |
| `channels.feishu.tools.wiki`                             | فعال‌کردن ابزارهای پایگاه دانش (نیازمند `doc`)                                         | `true`                               |
| `channels.feishu.tools.drive`                            | فعال‌کردن ابزارهای فضای ذخیره‌سازی ابری                                                           | `true`                               |
| `channels.feishu.tools.perm`                             | فعال‌کردن ابزارهای مدیریت مجوزها                                                   | `false`                              |
| `channels.feishu.tools.scopes`                           | فعال‌کردن ابزار تشخیص دامنه‌های دسترسی برنامه                                                    | `true`                               |
| `channels.feishu.tools.bitable`                          | فعال‌کردن ابزارهای Bitable/Base                                                            | `true`                               |
| `channels.feishu.tools.base`                             | نام مستعار برای `channels.feishu.tools.bitable`؛ اگر هر دو تنظیم شوند، `bitable` صریح اولویت دارد     | `true`                               |
| `channels.feishu.accounts.<id>.tools.bitable`            | محدودکنندهٔ ابزار Bitable/Base برای هر حساب                                                   | ارث‌بری‌شده                            |
| `channels.feishu.accounts.<id>.tools.base`               | نام مستعار برای هر حساب برای `tools.bitable`                                                | ارث‌بری‌شده                            |

## انواع پیام پشتیبانی‌شده

### دریافت

- ✅ متن
- ✅ متن غنی (پست)
- ✅ تصاویر
- ✅ فایل‌ها
- ✅ صدا
- ✅ ویدئو/رسانه
- ✅ استیکرها

پیام‌های صوتی ورودی Feishu/Lark به‌جای JSON خام `file_key`،
به‌صورت جای‌نگهدار رسانه عادی‌سازی می‌شوند. وقتی `tools.media.audio` پیکربندی شده باشد، OpenClaw
منبع یادداشت صوتی را بارگیری می‌کند و پیش از نوبت عامل، رونویسی صوتی مشترک را اجرا می‌کند،
تا عامل رونوشت گفتار را دریافت کند. اگر Feishu متن رونوشت را مستقیماً
در بار دادهٔ صوتی قرار دهد، آن متن بدون فراخوانی دوبارهٔ ASR استفاده می‌شود.
بدون ارائه‌دهندهٔ رونویسی صوتی، عامل همچنان یک جای‌نگهدار
`<media:audio>` به‌همراه پیوست ذخیره‌شده دریافت می‌کند، نه بار دادهٔ خام
منبع Feishu.

### ارسال

- ✅ متن
- ✅ تصاویر
- ✅ فایل‌ها
- ✅ صدا
- ✅ ویدئو/رسانه
- ✅ کارت‌های تعاملی (شامل به‌روزرسانی‌های جریانی)
- ⚠️ متن غنی (قالب‌بندی به‌سبک پست؛ از همهٔ قابلیت‌های نگارش Feishu/Lark پشتیبانی نمی‌کند)

حباب‌های صوتی بومی Feishu/Lark از نوع پیام Feishu با نام `audio` استفاده می‌کنند و به رسانهٔ بارگذاری‌شده با قالب Ogg/Opus (`file_type: "opus"`) نیاز دارند. رسانه‌های موجود `.opus` و `.ogg`
مستقیماً به‌صورت صوت بومی ارسال می‌شوند. MP3/WAV/M4A و سایر قالب‌هایی که احتمالاً صوتی هستند،
تنها زمانی با استفاده از `ffmpeg` به Ogg/Opus با نرخ 48kHz تبدیل می‌شوند که پاسخ، تحویل صوتی
را درخواست کند (`audioAsVoice` / ابزار پیام `asVoice`، شامل پاسخ‌های یادداشت صوتی
TTS). پیوست‌های معمولی MP3 همچنان به‌صورت فایل عادی باقی می‌مانند. اگر `ffmpeg` موجود نباشد یا
تبدیل ناموفق باشد، OpenClaw به پیوست فایل بازمی‌گردد و دلیل را ثبت می‌کند.

### رشته‌ها و پاسخ‌ها

- ✅ پاسخ‌های درون‌خطی
- ✅ پاسخ‌های رشته‌ای
- ✅ هنگام پاسخ به پیام یک رشته، پاسخ‌های رسانه‌ای همچنان از رشته آگاه می‌مانند

مسیریابی نشست گروه موضوعی در بخش
[دامنهٔ نشست گروهی و رشته‌های موضوعی](#group-session-scope-and-topic-threads) توضیح داده شده است.

## مرتبط

- [نمای کلی کانال‌ها](/fa/channels) - همهٔ کانال‌های پشتیبانی‌شده
- [جفت‌سازی](/fa/channels/pairing) - احراز هویت پیام مستقیم و جریان جفت‌سازی
- [گروه‌ها](/fa/channels/groups) - رفتار گفت‌وگوی گروهی و محدودسازی بر اساس منشن
- [مسیریابی کانال](/fa/channels/channel-routing) - مسیریابی نشست برای پیام‌ها
- [امنیت](/fa/gateway/security) - مدل دسترسی و مقاوم‌سازی
