---
read_when:
    - راه‌اندازی Mattermost
    - اشکال‌زدایی مسیریابی Mattermost
sidebarTitle: Mattermost
summary: راه‌اندازی ربات Mattermost و پیکربندی OpenClaw
title: Mattermost
x-i18n:
    generated_at: "2026-07-27T16:14:12Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ea41fb9a7e4e9ea6bd8d04a4f2c6d2d7f2e43cf71830e445f1e28e2e8737f3cb
    source_path: channels/mattermost.md
    workflow: 16
---

وضعیت: Plugin قابل‌دانلود (توکن ربات + رویدادهای WebSocket). کانال‌ها، کانال‌های خصوصی، پیام‌های مستقیم گروهی و پیام‌های مستقیم پشتیبانی می‌شوند. Mattermost یک پلتفرم پیام‌رسانی تیمی با قابلیت میزبانی شخصی است ([mattermost.com](https://mattermost.com)).

## نصب

<Tabs>
  <Tab title="رجیستری npm">
    ```bash
    openclaw plugins install @openclaw/mattermost
    ```
  </Tab>
  <Tab title="نسخهٔ محلی">
    ```bash
    openclaw plugins install ./path/to/local/mattermost-plugin
    ```
  </Tab>
</Tabs>

جزئیات: [Pluginها](/fa/tools/plugin)

## راه‌اندازی سریع

<Steps>
  <Step title="از در دسترس بودن Plugin مطمئن شوید">
    `@openclaw/mattermost` را با دستور بالا نصب کنید، سپس اگر Gateway از قبل در حال اجرا است، آن را راه‌اندازی مجدد کنید.
  </Step>
  <Step title="یک ربات Mattermost ایجاد کنید">
    یک حساب ربات Mattermost ایجاد کنید، **توکن ربات** را کپی کنید و ربات را به تیم‌ها و کانال‌هایی که باید بخواند اضافه کنید.
  </Step>
  <Step title="نشانی URL پایه را کپی کنید">
    **نشانی URL پایه** Mattermost را کپی کنید (برای مثال، `https://chat.example.com`). نویسهٔ `/api/v4` در انتهای آن به‌طور خودکار حذف می‌شود.
  </Step>
  <Step title="OpenClaw را پیکربندی و Gateway را راه‌اندازی کنید">
    پیکربندی حداقلی:

    ```json5
    {
      channels: {
        mattermost: {
          enabled: true,
          botToken: "mm-token",
          baseUrl: "https://chat.example.com",
          dmPolicy: "pairing",
        },
      },
    }
    ```

    جایگزین غیرتعاملی:

    ```bash
    openclaw channels add --channel mattermost --bot-token <token> --http-url https://chat.example.com
    ```

  </Step>
</Steps>

<Note>
برای Mattermost خودمیزبان روی یک نشانی خصوصی/LAN/tailnet: درخواست‌های خروجی API مربوط به Mattermost از یک محافظ SSRF عبور می‌کنند که به‌طور پیش‌فرض IPهای خصوصی و داخلی را مسدود می‌کند. با `channels.mattermost.network.dangerouslyAllowPrivateNetwork: true` آن را فعال کنید (برای هر حساب: `channels.mattermost.accounts.<id>.network.dangerouslyAllowPrivateNetwork`).
</Note>

## دستورهای اسلش بومی

دستورهای اسلش بومی اختیاری هستند. وقتی فعال شوند، OpenClaw دستورهای اسلش `oc_*` را در هر تیمی که ربات عضو آن است ثبت می‌کند و POSTهای بازخوانی را روی سرور HTTP مربوط به Gateway دریافت می‌کند.

```json5
{
  channels: {
    mattermost: {
      commands: {
        native: true,
        nativeSkills: true,
        callbackPath: "/api/channels/mattermost/command",
        // وقتی Mattermost نمی‌تواند مستقیماً به Gateway دسترسی پیدا کند، استفاده کنید (پروکسی معکوس/نشانی URL عمومی).
        callbackUrl: "https://gateway.example.com/api/channels/mattermost/command",
      },
    },
  },
}
```

دستورهای ثبت‌شده: `/oc_status`، `/oc_model`، `/oc_models`، `/oc_new`، `/oc_help`، `/oc_think`، `/oc_reasoning`، `/oc_verbose`، `/oc_queue`. با `nativeSkills: true`، دستورهای Skills نیز به‌صورت `/oc_<skill>` ثبت می‌شوند.

<AccordionGroup>
  <Accordion title="نکات رفتاری">
    - `native` و `nativeSkills` به‌طور پیش‌فرض `"auto"` هستند که برای Mattermost به حالت غیرفعال تبدیل می‌شود. آن‌ها را صراحتاً روی `true` تنظیم کنید.
    - `callbackPath` به‌طور پیش‌فرض `/api/channels/mattermost/command` است.
    - اگر `callbackUrl` حذف شود، OpenClaw مقدار `http://<gateway.customBindHost or localhost>:<gateway.port, default 18789><callbackPath>` را استخراج می‌کند. میزبان‌های اتصال عام (`0.0.0.0`، `::`) به `localhost` بازمی‌گردند.
    - برای راه‌اندازی‌های چندحسابی، `commands` را می‌توان در سطح بالا یا زیر `channels.mattermost.accounts.<id>.commands` تنظیم کرد (مقادیر حساب بر فیلدهای سطح بالا اولویت دارند).
    - دستورهای اسلش موجود با همان محرک که توسط یکپارچه‌سازی‌های دیگر ایجاد شده‌اند، دست‌نخورده باقی می‌مانند (ثبت از آن‌ها صرف‌نظر می‌کند)؛ دستورهایی که ربات ایجاد کرده است، هنگام تغییر نشانی URL بازخوانی به‌روزرسانی یا دوباره ایجاد می‌شوند.
    - بازخوانی‌های دستور با توکن‌های مختص هر دستور که Mattermost هنگام ثبت دستورهای `oc_*` توسط OpenClaw برمی‌گرداند، اعتبارسنجی می‌شوند.
    - OpenClaw پیش از پذیرش هر بازخوانی، ثبت فعلی دستور Mattermost را تازه‌سازی می‌کند؛ بنابراین توکن‌های قدیمی دستورهای اسلش حذف‌شده یا بازتولیدشده بدون راه‌اندازی مجدد Gateway دیگر پذیرفته نمی‌شوند.
    - اگر API مربوط به Mattermost نتواند تأیید کند که دستور همچنان فعلی است، اعتبارسنجی به‌صورت بسته شکست می‌خورد؛ اعتبارسنجی‌های ناموفق برای مدت کوتاهی در حافظهٔ نهان ذخیره می‌شوند، جست‌وجوهای هم‌زمان ادغام می‌شوند و شروع جست‌وجوهای تازه برای هر دستور با محدودیت نرخ انجام می‌شود تا فشار بازپخش محدود بماند.
    - هنگامی که ثبت ناموفق باشد، راه‌اندازی ناقص بوده باشد یا توکن بازخوانی با توکن ثبت‌شدهٔ دستور حل‌شده مطابقت نداشته باشد، بازخوانی‌های اسلش به‌صورت بسته شکست می‌خورند (توکنی که برای یک دستور معتبر است، نمی‌تواند برای دستور دیگری به اعتبارسنجی بالادستی برسد).
    - بازخوانی‌های پذیرفته‌شده با یک پاسخ موقت «در حال پردازش...» تأیید می‌شوند؛ پاسخ واقعی به‌صورت یک پیام عادی می‌رسد.

  </Accordion>
  <Accordion title="الزام دسترسی‌پذیری">
    نقطهٔ پایانی بازخوانی باید از سرور Mattermost قابل‌دسترسی باشد.

    - مگر اینکه Mattermost در همان میزبان/فضای نام شبکهٔ OpenClaw اجرا شود، `callbackUrl` را روی `localhost` تنظیم نکنید.
    - مگر اینکه نشانی URL پایهٔ Mattermost مسیر `/api/channels/mattermost/command` را با پروکسی معکوس به OpenClaw هدایت کند، `callbackUrl` را روی آن نشانی تنظیم نکنید.
    - یک بررسی سریع `curl https://<gateway-host>/api/channels/mattermost/command` است؛ یک GET باید `405 Method Not Allowed` را از OpenClaw برگرداند، نه `404`.

  </Accordion>
  <Accordion title="فهرست مجاز خروجی Mattermost">
    اگر بازخوانی شما نشانی‌های خصوصی/tailnet/داخلی را هدف می‌گیرد، `ServiceSettings.AllowedUntrustedInternalConnections` مربوط به Mattermost را طوری تنظیم کنید که میزبان/دامنهٔ بازخوانی را شامل شود.

    از ورودی‌های میزبان/دامنه استفاده کنید، نه نشانی‌های URL کامل.

    - درست: `gateway.tailnet-name.ts.net`
    - نادرست: `https://gateway.tailnet-name.ts.net`

  </Accordion>
</AccordionGroup>

## متغیرهای محیطی (حساب پیش‌فرض)

اگر متغیرهای محیطی را ترجیح می‌دهید، این موارد را روی میزبان Gateway تنظیم کنید:

- `MATTERMOST_BOT_TOKEN=...`
- `MATTERMOST_URL=https://chat.example.com`

<Note>
متغیرهای محیطی فقط روی حساب **پیش‌فرض** (`default`) اعمال می‌شوند. حساب‌های دیگر باید از مقادیر پیکربندی استفاده کنند.

`MATTERMOST_URL` را نمی‌توان از یک `.env` فضای کاری تنظیم کرد؛ [فایل‌های ‎.env فضای کاری](/fa/gateway/security) را ببینید.
</Note>

## حالت‌های گفت‌وگو

Mattermost به‌طور خودکار به پیام‌های مستقیم پاسخ می‌دهد. رفتار کانال توسط `chatmode` کنترل می‌شود:

<Tabs>
  <Tab title="oncall (پیش‌فرض)">
    در کانال‌ها فقط هنگام @اشاره پاسخ دهید.
  </Tab>
  <Tab title="onmessage">
    به هر پیام کانال پاسخ دهید.
  </Tab>
  <Tab title="onchar">
    وقتی پیام با یک پیشوند محرک آغاز می‌شود، پاسخ دهید.
  </Tab>
</Tabs>

نمونهٔ پیکربندی:

```json5
{
  channels: {
    mattermost: {
      chatmode: "onchar",
      oncharPrefixes: [">", "!"], // پیش‌فرض
    },
  },
}
```

نکات:

- `onchar` همچنان به @اشاره‌های صریح پاسخ می‌دهد.
- `channels.mattermost.requireMention` همچنان رعایت می‌شود، اما `chatmode` ترجیح داده می‌شود. تنظیمات `groups.<channelId>.requireMention` هر کانال بر هر دو اولویت دارند.
- پس از اینکه ربات یک پاسخ قابل‌مشاهده در رشتهٔ یک کانال ارسال کند، پیام‌های بعدی همان رشته بدون @اشارهٔ جدید یا پیشوند `onchar` پاسخ داده می‌شوند تا گفت‌وگوهای چندمرحله‌ای رشته بدون وقفه ادامه پیدا کنند. مشارکت تا 7 روز پس از آخرین پاسخ ربات در آن رشته به خاطر سپرده می‌شود و پس از راه‌اندازی‌های مجدد Gateway نیز باقی می‌ماند. رشته‌هایی که ربات فقط مشاهده کرده است تحت‌تأثیر قرار نمی‌گیرند؛ برای الزام دوبارهٔ اشارهٔ صریح، یک پیام جدید در سطح بالا آغاز کنید.
- برای جلوگیری از دور زدن محدودیت اشاره توسط پیگیری‌های رشته‌ای که ربات در آن‌ها مشارکت کرده است، `channels.mattermost.implicitMentions.threadParticipation: false` را تنظیم کنید. بازنویسی‌های حساب از `channels.mattermost.accounts.<id>.implicitMentions` استفاده می‌کنند. Mattermost در حال حاضر واقعیت‌های `replyToBot` یا `quotedBot` را تولید نمی‌کند، بنابراین این پرچم‌ها در اینجا اثری ندارند.

## رشته‌ها و نشست‌ها

برای کنترل اینکه پاسخ‌های کانال و گروه در کانال اصلی بمانند یا زیر پست محرک یک رشته آغاز کنند، از `channels.mattermost.replyToMode` استفاده کنید.

- `off` (پیش‌فرض): فقط زمانی در یک رشته پاسخ دهید که پست ورودی از قبل در یک رشته باشد.
- `first`: برای پست‌های سطح بالای کانال/گروه، زیر آن پست یک رشته آغاز کنید و گفت‌وگو را به یک نشست مختص رشته هدایت کنید.
- `all` و `batched`: امروزه برای Mattermost رفتاری مشابه `first` دارند، زیرا پس از ایجاد ریشهٔ رشته در Mattermost، بخش‌های بعدی و رسانه‌ها در همان رشته ادامه پیدا می‌کنند.
- پیام‌های مستقیم حتی وقتی `replyToMode` تنظیم شده باشد، به‌طور پیش‌فرض از `off` استفاده می‌کنند.

برای بازنویسی حالت گفت‌وگوهای `direct`، `group` یا `channel` از `channels.mattermost.replyToModeByChatType` استفاده کنید. برای وارد کردن پیام‌های مستقیم به رشته‌بندی، `direct` را تنظیم کنید:

- `off` (پیش‌فرض): پیام‌های مستقیم بدون رشته‌بندی و در یک نشست پیوسته باقی می‌مانند.
- `first`، `all` یا `batched`: هر پیام مستقیم سطح بالا یک رشتهٔ Mattermost را آغاز می‌کند که یک نشست تازه و مستقل پشتیبان آن است.

```json5
{
  channels: {
    mattermost: {
      replyToMode: "all",
      replyToModeByChatType: {
        direct: "first",
      },
    },
  },
}
```

نکات:

- نشست‌های مختص رشته از شناسهٔ پست محرک به‌عنوان ریشهٔ رشته استفاده می‌کنند.
- `first` و `all` در حال حاضر معادل‌اند، زیرا پس از ایجاد ریشهٔ رشته در Mattermost، بخش‌های بعدی و رسانه‌ها در همان رشته ادامه پیدا می‌کنند.
- بازنویسی‌های مختص نوع گفت‌وگو بر `replyToMode` اولویت دارند. بدون بازنویسی `direct`، استقرارهای موجود پیام‌های مستقیم را تخت و بدون رشته‌بندی نگه می‌دارند.

## کنترل دسترسی (پیام‌های مستقیم)

- پیش‌فرض: `channels.mattermost.dmPolicy = "pairing"` (فرستندگان ناشناس یک کد جفت‌سازی دریافت می‌کنند). مقادیر دیگر: `allowlist`، `open`، `disabled`.
- تأیید از طریق:
  - `openclaw pairing list mattermost`
  - `openclaw pairing approve mattermost <CODE>`
- پیام‌های مستقیم عمومی: `channels.mattermost.dmPolicy="open"` به‌همراه `channels.mattermost.allowFrom=["*"]` (شِمای پیکربندی نویسهٔ عام را الزامی می‌کند).
- `channels.mattermost.allowFrom` شناسه‌های کاربر (توصیه‌شده) و ورودی‌های `accessGroup:<name>` را می‌پذیرد. [گروه‌های دسترسی](/fa/channels/access-groups) را ببینید.

## کانال‌ها (گروه‌ها)

- پیش‌فرض: `channels.mattermost.groupPolicy = "allowlist"` (محدود به اشاره).
- فرستندگان را با `channels.mattermost.groupAllowFrom` در فهرست مجاز قرار دهید (شناسه‌های کاربر توصیه می‌شوند).
- `channels.mattermost.groupAllowFrom` ورودی‌های `accessGroup:<name>` را می‌پذیرد. [گروه‌های دسترسی](/fa/channels/access-groups) را ببینید.
- بازنویسی‌های اشاره برای هر کانال زیر `channels.mattermost.groups.<channelId>.requireMention` یا برای مقدار پیش‌فرض زیر `channels.mattermost.groups["*"].requireMention` قرار می‌گیرند.
- تطبیق `@username` تغییرپذیر است و فقط زمانی فعال می‌شود که `channels.mattermost.dangerouslyAllowNameMatching: true`.
- کانال‌های باز: `channels.mattermost.groupPolicy="open"` (محدود به اشاره).
- ترتیب حل: `channels.mattermost.groupPolicy`، سپس `channels.defaults.groupPolicy` و سپس `"allowlist"`.
- نکتهٔ زمان اجرا: اگر بخش `channels.mattermost` کاملاً وجود نداشته باشد، زمان اجرا برای بررسی‌های گروه به‌صورت بسته به `groupPolicy="allowlist"` شکست می‌خورد (حتی اگر `channels.defaults.groupPolicy` تنظیم شده باشد) و یک هشدار یک‌باره ثبت می‌کند.

مثال:

```json5
{
  channels: {
    mattermost: {
      groupPolicy: "open",
      groups: {
        "*": { requireMention: true },
        "team-channel-id": { requireMention: false },
      },
    },
  },
}
```

## مقصدهای تحویل خروجی

این قالب‌های مقصد را با `openclaw message send` یا cron/webhookها استفاده کنید:

| مقصد                                | تحویل به                                                       |
| ----------------------------------- | ------------------------------------------------------------- |
| `channel:<id>`                      | کانال بر اساس شناسه                                            |
| `channel:<name>` یا `#channel-name` | کانال بر اساس نام، با جست‌وجو در تیم‌هایی که ربات عضو آن‌هاست |
| `user:<id>` یا `mattermost:<id>`    | پیام مستقیم با آن کاربر                                       |
| `@username`                         | پیام مستقیم (نام کاربری از طریق API مربوط به Mattermost حل می‌شود) |

ارسال‌های خروجی حداکثر از یک پیوست در هر پیام پشتیبانی می‌کنند؛ چند فایل را به ارسال‌های جداگانه تقسیم کنید.

<Warning>
شناسه‌های مبهم بدون پیشوند (مانند `64ifufp...`) در Mattermost **مبهم** هستند (شناسهٔ کاربر در برابر شناسهٔ کانال).

OpenClaw آن‌ها را **ابتدا به‌عنوان کاربر** حل می‌کند:

- اگر شناسه به‌عنوان کاربر وجود داشته باشد (`GET /api/v4/users/<id>` موفق شود)، OpenClaw با یافتن کانال مستقیم از طریق `/api/v4/channels/direct` یک **پیام مستقیم** ارسال می‌کند.
- در غیر این صورت، شناسه به‌عنوان **شناسه کانال** در نظر گرفته می‌شود.

اگر به رفتار قطعی نیاز دارید، همیشه از پیشوندهای صریح (`user:<id>` / `channel:<id>`) استفاده کنید.
</Warning>

## تلاش مجدد برای کانال پیام مستقیم

وقتی OpenClaw به یک مقصد پیام مستقیم Mattermost ارسال می‌کند و ابتدا باید کانال مستقیم را پیدا کند، به‌طور پیش‌فرض خطاهای موقت ایجاد کانال مستقیم را دوباره امتحان می‌کند.

برای تنظیم سراسری این رفتار در Plugin مربوط به Mattermost از `channels.mattermost.dmChannelRetry` و برای یک حساب از `channels.mattermost.accounts.<id>.dmChannelRetry` استفاده کنید. مقادیر پیش‌فرض:

```json5
{
  channels: {
    mattermost: {
      dmChannelRetry: {
        maxRetries: 3,
        initialDelayMs: 1000,
        maxDelayMs: 10000,
        timeoutMs: 30000,
      },
    },
  },
}
```

نکته‌ها:

- این تنظیم فقط برای ایجاد کانال پیام مستقیم (`/api/v4/channels/direct`) اعمال می‌شود، نه برای تمام فراخوانی‌های API در Mattermost.
- تلاش‌های مجدد از عقب‌نشینی نمایی همراه با نوسان تصادفی استفاده می‌کنند و برای خطاهای موقتی مانند محدودیت نرخ، پاسخ‌های 5xx و خطاهای شبکه یا پایان مهلت اعمال می‌شوند.
- خطاهای سمت کارخواه 4xx به‌جز `429` دائمی در نظر گرفته می‌شوند و دوباره امتحان نمی‌شوند.

## استریم پیش‌نمایش

Mattermost فرایند تفکر، فعالیت ابزار و متن جزئی پاسخ را در یک **پست پیش‌نویس پیش‌نمایش** استریم می‌کند که وقتی ارسال پاسخ نهایی ایمن باشد، در همان محل نهایی می‌شود. در حالت `partial`، پیش‌نمایش به‌جای پر کردن کانال با پیام‌های جداگانه برای هر قطعه، روی همان شناسه پست به‌روزرسانی می‌شود. در حالت `block`، پیش‌نمایش بین متن تکمیل‌شده و بلوک‌های فعالیت ابزار جابه‌جا می‌شود؛ بنابراین بلوک‌های قبلی به‌جای بازنویسی‌شدن با بلوک بعدی، به‌صورت پست‌های مستقل قابل‌مشاهده باقی می‌مانند. خروجی‌های نهایی رسانه‌ای یا خطا، ویرایش‌های در انتظار پیش‌نمایش را لغو می‌کنند و به‌جای نهایی‌کردن یک پست پیش‌نمایش بلااستفاده، از تحویل عادی استفاده می‌کنند.

استریم پیش‌نمایش در حالت `partial` **به‌طور پیش‌فرض فعال است**. آن را از طریق `channels.mattermost.streaming.mode` پیکربندی کنید (مقادیر اسکالر/بولی قدیمی `streaming` توسط `openclaw doctor --fix` مهاجرت داده می‌شوند):

```json5
{
  channels: {
    mattermost: {
      streaming: { mode: "partial" }, // off | partial | block | progress
    },
  },
}
```

<AccordionGroup>
  <Accordion title="حالت‌های استریم">
    - `partial` (پیش‌فرض): یک پست پیش‌نمایش که هم‌زمان با گسترش پاسخ ویرایش می‌شود و سپس با پاسخ کامل نهایی می‌شود.
    - `block` پیش‌نمایش را بین متن تکمیل‌شده و بلوک‌های فعالیت ابزار جابه‌جا می‌کند؛ بنابراین هر بلوک به‌جای بازنویسی‌شدن در همان محل، به‌صورت پست مستقل قابل‌مشاهده باقی می‌ماند. به‌روزرسانی‌های موازی و متوالی ابزار از پست فعلی فعالیت ابزار به‌طور مشترک استفاده می‌کنند.
    - `progress` هنگام تولید، یک پیش‌نمایش وضعیت نمایش می‌دهد و پاسخ نهایی را فقط پس از تکمیل ارسال می‌کند.
    - `off` استریم پیش‌نمایش را غیرفعال می‌کند. با `streaming.block.enabled: true`، بلوک‌های تکمیل‌شده دستیار همچنان به‌صورت پاسخ‌های بلوکی عادی (پست‌های جداگانه) تحویل داده می‌شوند، نه یک پست نهایی یکپارچه.

  </Accordion>
  <Accordion title="نکته‌های رفتار استریم">
    - اگر استریم را نتوان در همان محل نهایی کرد (برای مثال، پست در میانه استریم حذف شده باشد)، OpenClaw با ارسال یک پست نهایی جدید ادامه می‌دهد تا پاسخ هرگز از دست نرود.
    - محموله‌هایی که فقط شامل تفکر هستند، از پست‌های کانال حذف می‌شوند؛ از جمله متنی که به‌صورت یک نقل‌قول بلوکی `> Thinking` دریافت می‌شود. برای مشاهده تفکر در سطوح دیگر، `/reasoning on` را تنظیم کنید؛ پست نهایی Mattermost فقط پاسخ را نگه می‌دارد.
    - برای ماتریس نگاشت کانال، به [استریم](/fa/concepts/streaming#preview-streaming-modes) مراجعه کنید.

  </Accordion>
</AccordionGroup>

## واکنش‌ها (ابزار پیام)

- از `message action=react` همراه با `channel=mattermost` استفاده کنید.
- `messageId` شناسه پست Mattermost است.
- `emoji` نام‌هایی مانند `thumbsup` یا `:+1:` را می‌پذیرد (دونقطه‌ها اختیاری هستند).
- برای حذف یک واکنش، `remove=true` (بولی) را تنظیم کنید.
- رویدادهای افزودن/حذف واکنش به‌صورت رویدادهای سیستمی به نشست عامل مسیریابی‌شده فرستاده می‌شوند و مشمول همان بررسی‌های سیاست پیام مستقیم/گروه هستند که برای پیام‌ها اعمال می‌شوند.

نمونه‌ها:

```text
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup
message action=react channel=mattermost target=channel:<channelId> messageId=<postId> emoji=thumbsup remove=true
```

پیکربندی:

- `channels.mattermost.actions.reactions`: فعال/غیرفعال‌کردن کنش‌های واکنش (پیش‌فرض true).
- بازنویسی برای هر حساب: `channels.mattermost.accounts.<id>.actions.reactions`.

## دکمه‌های تعاملی (ابزار پیام)

پیام‌هایی با دکمه‌های قابل‌کلیک ارسال کنید. وقتی کاربری روی دکمه‌ای کلیک می‌کند، عامل گزینه انتخاب‌شده را دریافت می‌کند و می‌تواند پاسخ دهد.

دکمه‌ها از محموله معنایی `presentation` می‌آیند (در پاسخ‌های عادی عامل و در `message action=send`). OpenClaw دکمه‌های مقداری را به‌شکل دکمه‌های تعاملی Mattermost نمایش می‌دهد، دکمه‌های URL را در متن پیام قابل‌مشاهده نگه می‌دارد و منوهای انتخاب را به متن خوانا تبدیل می‌کند.

```text
message action=send channel=mattermost target=channel:<channelId> presentation={"blocks":[{"type":"buttons","buttons":[{"label":"Yes","value":"yes"},{"label":"No","value":"no"}]}]}
```

فیلدهای دکمه ارائه:

<ParamField path="label" type="string" required>
  برچسب نمایشی (نام مستعار: `text`).
</ParamField>
<ParamField path="value" type="string">
  مقداری که هنگام کلیک بازگردانده می‌شود و به‌عنوان شناسه کنش استفاده می‌شود (نام‌های مستعار: `callback_data`، `callbackData`). برای یک دکمه قابل‌کلیک الزامی است، مگر اینکه `url` تنظیم شده باشد.
</ParamField>
<ParamField path="url" type="string">
  دکمه پیوند؛ به‌جای یک دکمه تعاملی، به‌صورت متن `label: url` در بدنه پیام نمایش داده می‌شود.
</ParamField>
<ParamField path="style" type='"primary" | "secondary" | "success" | "danger"'>
  سبک دکمه. Mattermost برای مقادیری که پشتیبانی نمی‌کند، سبک پیش‌فرض را اعمال می‌کند.
</ParamField>

برای اعلام پشتیبانی از دکمه‌ها در اعلان سیستمی عامل، `inlineButtons` را به قابلیت‌های کانال اضافه کنید:

```json5
{
  channels: {
    mattermost: {
      capabilities: ["inlineButtons"],
    },
  },
}
```

وقتی کاربری روی یک دکمه کلیک می‌کند:

<Steps>
  <Step title="بررسی دسترسی">
    کلیک‌کننده باید همان بررسی‌های سیاست پیام مستقیم/گروه را که برای فرستنده پیام اعمال می‌شود با موفقیت پشت سر بگذارد؛ کلیک‌های غیرمجاز یک اعلان موقت دریافت می‌کنند و نادیده گرفته می‌شوند.
  </Step>
  <Step title="جایگزینی دکمه‌ها با تأییدیه">
    همه دکمه‌ها با یک خط تأیید جایگزین می‌شوند (برای مثال، "✓ **Yes** selected by @user").
  </Step>
  <Step title="عامل گزینه انتخاب‌شده را دریافت می‌کند">
    عامل گزینه انتخاب‌شده را به‌صورت یک پیام ورودی (به‌همراه یک رویداد سیستمی) دریافت و پاسخ می‌دهد.
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="نکته‌های پیاده‌سازی">
    - فراخوان‌های بازگشتی دکمه از اعتبارسنجی HMAC-SHA256 استفاده می‌کنند (خودکار است و به پیکربندی نیاز ندارد).
    - هنگام کلیک، کل بلوک پیوست جایگزین می‌شود؛ بنابراین همه دکمه‌ها با هم حذف می‌شوند و حذف جزئی ممکن نیست.
    - شناسه‌های کنش حاوی خط تیره یا زیرخط به‌طور خودکار پاک‌سازی می‌شوند (محدودیت مسیریابی Mattermost).
    - کلیک‌هایی که `action_id` آن‌ها با کنشی در پست اصلی مطابقت ندارد، با `403` ("کنش ناشناخته") رد می‌شوند.

  </Accordion>
  <Accordion title="پیکربندی و دسترس‌پذیری">
    - `channels.mattermost.capabilities`: آرایه‌ای از رشته‌های قابلیت. برای فعال‌کردن توضیح ابزار دکمه‌ها در اعلان سیستمی عامل، `"inlineButtons"` را اضافه کنید.
    - `channels.mattermost.interactions.callbackBaseUrl`: نشانی پایه خارجی اختیاری برای فراخوان‌های بازگشتی دکمه (برای مثال `https://gateway.example.com`). وقتی Mattermost نمی‌تواند مستقیماً در میزبان اتصال Gateway به آن دسترسی داشته باشد، از این گزینه استفاده کنید.
    - در پیکربندی‌های چندحسابی، می‌توانید همین فیلد را در `channels.mattermost.accounts.<id>.interactions.callbackBaseUrl` نیز تنظیم کنید.
    - اگر `interactions.callbackBaseUrl` حذف شده باشد، OpenClaw نشانی فراخوان بازگشتی را از `gateway.customBindHost` + `gateway.port` (پیش‌فرض 18789) استخراج می‌کند و سپس به `http://localhost:<port>` بازمی‌گردد. مسیر فراخوان بازگشتی `/mattermost/interactions/<accountId>` است.
    - قاعده دسترس‌پذیری: نشانی فراخوان بازگشتی دکمه باید از سرور Mattermost قابل‌دسترسی باشد. `localhost` فقط زمانی کار می‌کند که Mattermost و OpenClaw روی یک میزبان/فضای نام شبکه اجرا شوند.
    - `channels.mattermost.interactions.allowedSourceIps`: فهرست مجاز IP مبدأ برای فراخوان‌های بازگشتی دکمه. بدون آن، فقط مبدأهای حلقه محلی (`127.0.0.1`، `::1`) پذیرفته می‌شوند؛ بنابراین سرور راه‌دور Mattermost باید در اینجا مجاز شود، وگرنه کلیک‌های آن با `403` رد می‌شوند. پشت یک پراکسی معکوس، `gateway.trustedProxies` را نیز تنظیم کنید تا IP واقعی کارخواه از سرآیندهای هدایت‌شده استخراج شود.
    - اگر مقصد فراخوان بازگشتی شما خصوصی/tailnet/داخلی است، میزبان/دامنه آن را به `ServiceSettings.AllowedUntrustedInternalConnections` در Mattermost اضافه کنید.

  </Accordion>
</AccordionGroup>

### یکپارچه‌سازی مستقیم API (اسکریپت‌های خارجی)

اسکریپت‌های خارجی و Webhookها می‌توانند به‌جای عبور از ابزار `message` عامل، دکمه‌ها را مستقیماً از طریق REST API مربوط به Mattermost ارسال کنند. ابزار `message` در OpenClaw را ترجیح دهید. برای یکپارچه‌سازی مستقیم، `buildButtonAttachments` را از `@openclaw/mattermost/api.js` وارد کنید؛ اگر JSON خام ارسال می‌کنید، این قواعد را رعایت کنید:

**ساختار محموله:**

```json5
{
  channel_id: "<channelId>",
  message: "Choose an option:",
  props: {
    attachments: [
      {
        actions: [
          {
            id: "mybutton01", // alphanumeric only - see below
            type: "button", // required, or clicks are silently ignored
            name: "Approve", // display label
            style: "primary", // optional: "default", "primary", "danger"
            integration: {
              url: "https://gateway.example.com/mattermost/interactions/default",
              context: {
                action_id: "mybutton01", // must match button id
                action: "approve",
                // ... any custom fields ...
                _token: "<hmac>", // see HMAC section below
              },
            },
          },
        ],
      },
    ],
  },
}
```

<Warning>
**قواعد حیاتی**

1. پیوست‌ها باید در `props.attachments` قرار گیرند، نه در `attachments` سطح بالا (بی‌سروصدا نادیده گرفته می‌شوند).
2. هر کنش به `type: "button"` نیاز دارد؛ بدون آن، کلیک‌ها بی‌سروصدا نادیده گرفته می‌شوند.
3. هر کنش به یک فیلد `id` نیاز دارد؛ Mattermost کنش‌های بدون شناسه را نادیده می‌گیرد.
4. `id` کنش باید **فقط شامل نویسه‌های حرفی‌عددی** باشد (`[a-zA-Z0-9]`). خط تیره و زیرخط مسیریابی سمت سرور کنش در Mattermost را مختل می‌کنند (پاسخ 404 بازمی‌گردد). پیش از استفاده آن‌ها را حذف کنید.
5. `context.action_id` باید با `id` دکمه مطابقت داشته باشد؛ Gateway کلیک‌هایی را که `action_id` آن‌ها در پست وجود ندارد رد می‌کند.
6. `context.action_id` الزامی است؛ کنترل‌کننده تعامل بدون آن پاسخ 400 بازمی‌گرداند.
7. IP مبدأ فراخوان بازگشتی باید مجاز باشد (به `interactions.allowedSourceIps` در بالا مراجعه کنید).

</Warning>

**تولید توکن HMAC**

Gateway کلیک‌های دکمه را با HMAC-SHA256 اعتبارسنجی می‌کند. اسکریپت‌های خارجی باید توکن‌هایی تولید کنند که با منطق اعتبارسنجی Gateway مطابقت داشته باشند:

<Steps>
  <Step title="استخراج راز از توکن ربات">
    `HMAC-SHA256(key="openclaw-mattermost-interactions", data=botToken)`، با کدگذاری هگزادسیمال.
  </Step>
  <Step title="ساخت شیء زمینه">
    شیء زمینه را با همه فیلدها **به‌جز** `_token` بسازید.
  </Step>
  <Step title="سریال‌سازی با کلیدهای مرتب‌شده">
    با **کلیدهای مرتب‌شده به‌صورت بازگشتی** و **بدون فاصله** سریال‌سازی کنید (Gateway اشیای تو‌در‌تو را نیز به‌شکل معیار درمی‌آورد و JSON فشرده تولید می‌کند).
  </Step>
  <Step title="امضای محموله">
    `HMAC-SHA256(key=secret, data=serializedContext)`
  </Step>
  <Step title="افزودن توکن">
    چکیده هگزادسیمال حاصل را به‌عنوان `_token` به زمینه اضافه کنید.
  </Step>
</Steps>

نمونه Python:

```python
import hmac, hashlib, json

secret = hmac.new(
    b"openclaw-mattermost-interactions",
    bot_token.encode(), hashlib.sha256
).hexdigest()

ctx = {"action_id": "mybutton01", "action": "approve"}
payload = json.dumps(ctx, sort_keys=True, separators=(",", ":"))
token = hmac.new(secret.encode(), payload.encode(), hashlib.sha256).hexdigest()

context = {**ctx, "_token": token}
```

<AccordionGroup>
  <Accordion title="خطاهای رایج HMAC">
    - `json.dumps` در Python به‌طور پیش‌فرض فاصله اضافه می‌کند (`{"key": "val"}`). برای مطابقت با خروجی فشردهٔ JavaScript (`{"key":"val"}`) از `separators=(",", ":")` استفاده کنید.
    - همیشه **همهٔ** فیلدهای context را (به‌جز `_token`) امضا کنید. Gateway ابتدا `_token` را حذف می‌کند و سپس تمام موارد باقی‌مانده را امضا می‌کند. امضای زیرمجموعه‌ای از فیلدها باعث شکست بی‌صدای تأیید می‌شود.
    - از `sort_keys=True` استفاده کنید؛ Gateway پیش از امضا کلیدها را مرتب می‌کند و Mattermost ممکن است هنگام ذخیره‌سازی payload ترتیب فیلدهای context را تغییر دهد.
    - secret را از bot token به‌صورت قطعی استخراج کنید، نه از بایت‌های تصادفی. secret باید در فرایندی که دکمه‌ها را ایجاد می‌کند و Gatewayای که تأیید را انجام می‌دهد یکسان باشد.

  </Accordion>
</AccordionGroup>

## آداپتور دایرکتوری

Plugin مربوط به Mattermost شامل یک آداپتور دایرکتوری است که نام کانال‌ها و کاربران را از طریق API مربوط به Mattermost تفکیک می‌کند. این قابلیت، استفاده از مقصدهای `#channel-name` و `@username` را در تحویل‌های `openclaw message send` و Cron/Webhook ممکن می‌سازد.

هیچ پیکربندی‌ای لازم نیست؛ آداپتور از bot token موجود در پیکربندی حساب استفاده می‌کند.

## چندحسابی

Mattermost از چند حساب در `channels.mattermost.accounts` پشتیبانی می‌کند:

```json5
{
  channels: {
    mattermost: {
      accounts: {
        default: { name: "Primary", botToken: "mm-token", baseUrl: "https://chat.example.com" },
        alerts: { name: "Alerts", botToken: "mm-token-2", baseUrl: "https://alerts.example.com" },
      },
    },
  },
}
```

مقادیر حساب، فیلدهای سطح بالا را بازنویسی می‌کنند؛ `channels.mattermost.defaultAccount` تعیین می‌کند وقتی حسابی مشخص نشده است، از کدام حساب استفاده شود.

## عیب‌یابی

<AccordionGroup>
  <Accordion title="پاسخی در کانال‌ها دریافت نمی‌شود">
    مطمئن شوید bot در کانال حضور دارد و به آن اشاره کنید (oncall)، از یک پیشوند راه‌انداز استفاده کنید (onchar)، یا `chatmode: "onmessage"` را تنظیم کنید.
  </Accordion>
  <Accordion title="خطاهای احراز هویت یا چندحسابی">
    - bot token، نشانی پایه و فعال‌بودن حساب را بررسی کنید.
    - مشکلات چندحسابی: متغیرهای محیطی فقط برای حساب `default` اعمال می‌شوند.
    - میزبان‌های خصوصی/LAN مربوط به Mattermost به `network.dangerouslyAllowPrivateNetwork: true` نیاز دارند (محافظ SSRF به‌طور پیش‌فرض IPهای خصوصی را مسدود می‌کند).

  </Accordion>
  <Accordion title="دستورهای بومی اسلش ناموفق‌اند">
    - `Unauthorized: invalid command token.`: OpenClaw توکن callback را نپذیرفت. دلایل معمول:
      - ثبت دستور اسلش هنگام راه‌اندازی ناموفق بوده یا فقط بخشی از آن تکمیل شده است
      - callback به Gateway یا حساب نادرست می‌رسد
      - Mattermost همچنان دستورهای قدیمی‌ای دارد که به مقصد callback قبلی اشاره می‌کنند
      - Gateway بدون فعال‌سازی دوبارهٔ دستورهای اسلش راه‌اندازی مجدد شده است
    - اگر دستورهای بومی اسلش از کار افتادند، گزارش‌ها را برای `mattermost: failed to register slash commands` یا `mattermost: native slash commands enabled but no commands could be registered` بررسی کنید.
    - اگر `callbackUrl` حذف شده باشد و گزارش‌ها هشدار دهند که callback به یک URL حلقهٔ محلی مانند `http://localhost:18789/...` تفکیک شده است، احتمالاً آن URL فقط زمانی قابل دسترسی است که Mattermost در همان میزبان/فضای نام شبکهٔ OpenClaw اجرا شود. به‌جای آن، یک `commands.callbackUrl` صریح و قابل دسترسی از بیرون تنظیم کنید.

  </Accordion>
  <Accordion title="مشکلات دکمه‌ها">
    - دکمه‌ها به‌شکل کادرهای سفید نمایش داده می‌شوند یا اصلاً نمایش داده نمی‌شوند: دادهٔ دکمه نادرست است. هر دکمهٔ نمایشی به یک `label` و یک `value` نیاز دارد (دکمه‌هایی که یکی از این دو را نداشته باشند حذف می‌شوند).
    - دکمه‌ها نمایش داده می‌شوند، اما کلیک‌ها کاری انجام نمی‌دهند: بررسی کنید Gateway از سرور Mattermost قابل دسترسی باشد، IP سرور Mattermost در `channels.mattermost.interactions.allowedSourceIps` گنجانده شده باشد (بدون آن فقط حلقهٔ محلی پذیرفته می‌شود) و `ServiceSettings.AllowedUntrustedInternalConnections` برای مقصدهای خصوصی شامل میزبان callback باشد.
    - دکمه‌ها هنگام کلیک خطای 404 برمی‌گردانند: احتمالاً `id` دکمه حاوی خط تیره یا زیرخط است. مسیریاب action در Mattermost با شناسه‌های غیرالفبایی‌عددی دچار اختلال می‌شود. فقط از `[a-zA-Z0-9]` استفاده کنید.
    - Gateway پیام `rejected callback source` را ثبت می‌کند: کلیک از IPای خارج از `interactions.allowedSourceIps` آمده است. سرور Mattermost یا ingress خود را به فهرست مجاز اضافه کنید و در پشت reverse proxy، `gateway.trustedProxies` را تنظیم کنید.
    - Gateway پیام `invalid _token` را ثبت می‌کند: HMAC مطابقت ندارد. بررسی کنید که همهٔ فیلدهای context را امضا می‌کنید (نه زیرمجموعه‌ای از آن‌ها)، کلیدهای مرتب‌شده را به‌کار می‌برید و از JSON فشرده (بدون فاصله) استفاده می‌کنید. بخش HMAC در بالا را ببینید.
    - Gateway پیام `missing _token in context` را ثبت می‌کند: فیلد `_token` در context دکمه وجود ندارد. هنگام ساخت payload یکپارچه‌سازی، مطمئن شوید این فیلد گنجانده شده است.
    - Gateway کلیک را با `Unknown action` رد می‌کند: `context.action_id` با هیچ action با `id` در پست مطابقت ندارد. هر دو را روی یک مقدار پاک‌سازی‌شدهٔ یکسان تنظیم کنید.
    - Agent دکمه‌ها را ارائه نمی‌کند: `capabilities: ["inlineButtons"]` را به پیکربندی کانال Mattermost اضافه کنید.

  </Accordion>
</AccordionGroup>

## مرتبط

- [مسیریابی کانال](/fa/channels/channel-routing) - مسیریابی نشست برای پیام‌ها
- [نمای کلی کانال‌ها](/fa/channels) - همهٔ کانال‌های پشتیبانی‌شده
- [گروه‌ها](/fa/channels/groups) - رفتار گفت‌وگوی گروهی و محدودسازی بر اساس اشاره
- [جفت‌سازی](/fa/channels/pairing) - احراز هویت پیام مستقیم و جریان جفت‌سازی
- [امنیت](/fa/gateway/security) - مدل دسترسی و مقاوم‌سازی
