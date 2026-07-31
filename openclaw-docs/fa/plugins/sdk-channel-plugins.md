---
read_when:
    - در حال ساخت یک Plugin جدید برای کانال پیام‌رسانی هستید
    - می‌خواهید OpenClaw را به یک پلتفرم پیام‌رسان متصل کنید
    - باید سطح آداپتور `ChannelPlugin` را درک کنید
sidebarTitle: Channel Plugins
summary: راهنمای گام‌به‌گام ساخت Plugin کانال پیام‌رسانی برای OpenClaw
title: ساخت Pluginهای کانال
x-i18n:
    generated_at: "2026-07-27T17:31:14Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0ff8ad04346babf3eece7e10bd38946ee290947b2e504b6b5ca438865531bf38
    source_path: plugins/sdk-channel-plugins.md
    workflow: 16
---

این راهنما یک Plugin کانال می‌سازد که OpenClaw را به یک پلتفرم پیام‌رسانی متصل می‌کند: امنیت پیام خصوصی، جفت‌سازی، رشته‌بندی پاسخ‌ها و پیام‌رسانی خروجی.

<Info>
  با Plugin‌های OpenClaw تازه آشنا شده‌اید؟ ابتدا برای آشنایی با ساختار بسته و راه‌اندازی مانیفست، [شروع به کار](/fa/plugins/building-plugins)
  را بخوانید.
</Info>

## موارد تحت مالکیت Plugin شما

Plugin‌های کانال ابزارهای ارسال/ویرایش/واکنش را پیاده‌سازی نمی‌کنند؛ هسته یک ابزار
مشترک `message` ارائه می‌کند. Plugin شما مالک موارد زیر است:

- **پیکربندی** - تفکیک حساب و راهنمای راه‌اندازی
- **امنیت** - خط‌مشی پیام خصوصی و فهرست‌های مجاز
- **جفت‌سازی** - جریان تأیید پیام خصوصی
- **دستور زبان نشست** - نحوه نگاشت شناسه‌های مکالمه مختص ارائه‌دهنده به
  گپ‌های پایه، شناسه‌های رشته و بازگشت‌های والد
- **خروجی** - ارسال متن، رسانه و نظرسنجی به پلتفرم
- **رشته‌بندی** - نحوه قرارگیری پاسخ‌ها در رشته
- **نشانگر تایپ Heartbeat** - سیگنال‌های اختیاری تایپ/مشغول برای مقصدهای تحویل
  Heartbeat

هسته مالک ابزار مشترک پیام، سیم‌کشی پرامپت، شکل بیرونی کلید نشست،
ثبت‌ودفتر عمومی `:thread:` و اعزام است.

## آداپتور پیام

یک آداپتور `message` را با `defineChannelMessageAdapter` از
`openclaw/plugin-sdk/channel-outbound` در معرض استفاده قرار دهید. فقط قابلیت‌های پایدار ارسال نهایی
را که انتقال بومی شما واقعاً پشتیبانی می‌کند اعلام کنید و آن‌ها را با یک آزمون قرارداد
پشتیبانی کنید که اثر جانبی بومی و رسید بازگشتی را اثبات کند. ارسال‌های متن/رسانه را
به همان توابع انتقالی هدایت کنید که آداپتور قدیمی `outbound` استفاده می‌کند. برای
قرارداد کامل API، ماتریس قابلیت‌ها، قواعد رسید، نهایی‌سازی پیش‌نمایش زنده،
خط‌مشی تأیید دریافت، آزمون‌ها و جدول مهاجرت، به
[API خروجی کانال](/fa/plugins/sdk-channel-outbound) مراجعه کنید.

اگر آداپتور موجود `outbound` از قبل روش‌های ارسال و
فراداده قابلیت مناسب را دارد، به‌جای نوشتن دستی پلی دیگر،
آداپتور `message` را با `createChannelMessageAdapterFromOutbound(...)`
استخراج کنید. ارسال‌های آداپتور مقادیر `MessageReceipt` را برمی‌گردانند. برای شناسه‌های قدیمی، آن‌ها را
با `listMessageReceiptPlatformIds(...)` یا
`resolveMessageReceiptPrimaryId(...)` استخراج کنید، نه اینکه فیلدهای موازی `messageIds`
را نگه دارید.

قابلیت‌های زنده و نهایی‌ساز را دقیق اعلام کنید - هسته از آن‌ها برای تصمیم‌گیری درباره
کارهایی که یک کانال می‌تواند انجام دهد استفاده می‌کند و اختلاف میان رفتار اعلام‌شده و واقعی،
شکست آزمون قرارداد محسوب می‌شود:

| سطح                                   | مقادیر                                                                                           |
| ------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `message.live.capabilities`           | `draftPreview`، `previewFinalization`، `progressUpdates`، `nativeStreaming`، `quietFinalization` |
| `message.live.finalizer.capabilities` | `finalEdit`، `normalFallback`، `discardPending`، `previewReceipt`، `retainOnAmbiguousFailure`    |

کانال‌هایی که پیش‌نویس پیش‌نمایش را درجا نهایی می‌کنند باید منطق زمان اجرا را
از طریق `defineFinalizableLivePreviewAdapter(...)` به‌همراه
`deliverWithFinalizableLivePreviewAdapter(...)` هدایت کنند و قابلیت‌های اعلام‌شده را
با آزمون‌های `verifyChannelMessageLiveCapabilityAdapterProofs(...)`
و `verifyChannelMessageLiveFinalizerProofs(...)` پشتیبانی کنند تا رفتار پیش‌نمایش بومی،
پیشرفت، ویرایش، بازگشت/نگهداشت، پاک‌سازی و رسید نتواند بی‌سروصدا منحرف شود.

گیرنده‌های ورودی که تأییدهای پلتفرم را به تعویق می‌اندازند باید
`message.receive.defaultAckPolicy` و `supportedAckPolicies` را اعلام کنند، نه اینکه
زمان‌بندی تأیید را در وضعیت محلی پایشگر پنهان کنند. هر خط‌مشی اعلام‌شده را با
`verifyChannelMessageReceiveAckPolicyAdapterProofs(...)` پوشش دهید.

کمک‌تابع‌های قدیمی پاسخ مانند `dispatchInboundReplyWithBase` و
`recordInboundSessionAndDispatchReply` برای اعزام‌کننده‌های سازگاری همچنان در دسترس‌اند.
برای کد کانال جدید از آن‌ها استفاده نکنید؛ در عوض با آداپتور `message`،
رسیدها و کمک‌تابع‌های چرخه عمر دریافت/ارسال در
`openclaw/plugin-sdk/channel-outbound` شروع کنید.

### ورود ورودی (آزمایشی)

کانال‌هایی که مجوزدهی ورودی را مهاجرت می‌دهند می‌توانند از زیرمسیر آزمایشی
`openclaw/plugin-sdk/channel-ingress-runtime` در مسیرهای دریافت زمان اجرا استفاده کنند.
این زیرمسیر واقعیت‌های پلتفرم، فهرست‌های مجاز خام، توصیفگرهای مسیر، واقعیت‌های فرمان
و پیکربندی گروه دسترسی را می‌پذیرد، سپس تصویرهای فرستنده/مسیر/فرمان/فعال‌سازی
به‌همراه گراف مرتب ورود را برمی‌گرداند، درحالی‌که جست‌وجوی پلتفرم و اثرهای جانبی
در Plugin باقی می‌مانند. نرمال‌سازی هویت Plugin را در
توصیفگری که به تفکیک‌کننده می‌دهید نگه دارید؛ مقادیر تطبیق خام را از
وضعیت یا تصمیم تفکیک‌شده سریال‌سازی نکنید. برای طراحی API،
مرز مالکیت و انتظارات آزمون، به
[API ورود کانال](/fa/plugins/sdk-channel-ingress) مراجعه کنید.

### ورود پایدار و حذف تکرار بازپخش

کانال‌هایی که ورود پایدار را به کار می‌گیرند، مگر آنکه به قرارداد پذیرش یا پمپاژ
اساساً متفاوتی نیاز داشته باشند، باید از `createChannelIngressMonitor`
در `openclaw/plugin-sdk/channel-outbound` استفاده کنند. پاکت خام انتقال را در یک
گلوگاه دریافت واحد در صف قرار دهید (بدون نرمال‌سازی هنگام دریافت)، برای انتقال‌های Webhook
تأیید انتقال را به افزودن پایدار مشروط کنید، برای هر مکالمه یک مسیر سریال‌شده
بسازید و رویداد را هنگام پذیرش اعزام تکمیل‌شده علامت بزنید.
کلید اصلی صف `(queue_name, event_id)` است و تکمیل، به‌جای حذف، ردیف را
به سنگ‌قبر تبدیل می‌کند؛ بنابراین تحویل مجدد دیرهنگام همان
`event_id` از سوی پلتفرم، در پنجره نگهداشت سنگ‌قبر به‌طور پایدار رد می‌شود.
برای API پایشگر و قرارداد خاموش‌سازی، به
[API خروجی کانال](/fa/plugins/sdk-channel-outbound#durable-ingress-monitors)
مراجعه کنید.

این سنگ‌قبر، قاعده لایه‌بندی محافظ‌های بازپخش
(`openclaw/plugin-sdk/persistent-dedupe`) است: یک کانال تخلیه‌شده فقط زمانی محافظ
بازپخش جداگانه‌ای نگه می‌دارد که هویت یا نگهداشت محافظ از صف بیشتر باشد
— کلید منطقی پیامی که با شناسه تحویل انتقال متفاوت است (Telegram
`chat_id:message_id` را تکرارزدایی می‌کند، زیرا ادغام‌های رفع جهش می‌توانند پیامی را
با `update_id` تازه دوباره ظاهر کنند)، یا پنجره‌ای طولانی‌تر از نگهداشت
سنگ‌قبر کانال. اگر کلید محافظ شما با `event_id` تخلیه برابر است، هنگام
به‌کارگیری تخلیه، محافظ را حذف کنید و به‌جای آن `completedTtlMs`/`completedMaxEntries`
را طوری اندازه‌گذاری کنید که پنجره قدیمی محافظ را پوشش دهند. محافظت‌های غیرمرتبط با تکرارزدایی،
مانند حصارهای سنی، مشمول این قاعده نیستند. شناسه‌های پایدار پیام خروجی به‌جای
حافظه نهان TTL محلی کانال، از رجیستری مشترک پژواک خروجی در
`openclaw/plugin-sdk/channel-outbound` استفاده می‌کنند.

#### کلاس‌های انتقال و نگهداشت

یک انتقال را بر اساس تضمین بازیابی در مرز دریافت آن طبقه‌بندی کنید:

- **تحویل Webhook یا رویداد مشروط به تأیید:** فقط پس از
  افزودن پایدار، تأیید کنید یا موفقیت برگردانید. شکست افزودن باید تحویل را واجد شرایط
  تلاش مجدد نگه دارد یا مرز دریافت را با شکست مواجه کند. این کلاس شامل Slack، SMS، Zalo،
  Microsoft Teams، Google Chat، LINE و Synology Chat است.
- **تحویل نظرسنجی یا جریان انتظارشده:** مکان‌نما را از راه دور جلو ببرید یا
  تأیید انتقال را فقط پس از افزودن ارسال کنید. وقتی مکان‌نمای صریحی وجود ندارد،
  فراخوان دریافت را سریال‌شده و انتظارشده نگه دارید تا شکست افزودن نتواند باعث شود
  حلقه دریافت جلو بیفتد. نظرسنجی Telegram، Signal و Tlon از این کلاس استفاده می‌کنند؛
  تحویل Webhook در Telegram از قاعده مشروط به تأیید بالا پیروی می‌کند.
- **سوکت‌های بدون بازپخش:** IRC، Mattermost، Twitch و Zalo Personal نمی‌توانند از
  پلتفرم بخواهند رویداد پذیرفته‌شده را دوباره تحویل دهد. صف پایدار آن‌ها از
  پنجره خرابی فرایند محافظت می‌کند و بازیابی پس از راه‌اندازی مجدد محلی را پشتیبانی می‌کند؛ سنگ‌قبرهای
  تکمیل در برابر بازپخش پلتفرم تقریباً بی‌اثرند.

از 30 روز به‌عنوان قرارداد TTL سنگ‌قبر در کل ناوگان استفاده کنید، نه به‌عنوان پیش‌فرض SDK. یک
پنجره تحویل مجدد با حجم بالا معمولاً از سقف 20,000 ورودی تکمیل‌شده استفاده می‌کند؛
انتقال‌های انتظارشده و بدون بازپخش با حجم کمتر معمولاً از 1,000-2,000 استفاده می‌کنند.
استثناهای کنونی شامل سقف‌های 4,096 ورودی LINE، TTL تکمیل‌شده 24 ساعته SMS
و نگهداشت تکمیل‌شده فقط مبتنی بر سقف در Tlon است. سقف ردیف‌های شکست‌خورده نیز ممکن است از
سقف تکمیل‌شده کمتر باشد. TTL و سقف هر دو ردیف‌ها را هرس می‌کنند، بنابراین نگهداشت مؤثر
با رسیدن نخستین حد پایان می‌یابد. فقط برای افق تلاش مجدد مستند پلتفرم،
پنجره حفظ‌شده محافظ بازپخش منتشرشده، حجم مورد انتظار یا بودجه دیسک،
یا انتقال بدون بازپخش انحراف ایجاد کنید و قرارداد نگهداشت را با آزمون‌ها پوشش دهید.

#### اثرهای جانبی حداقل یک‌بار

اعزام تخلیه، اثرهای جانبی فرمان را پیش از آنکه ردیف ورود به سنگ‌قبر
تکمیل برسد اجرا می‌کند. خرابی فرایند میان این مراحل، ردیف را بازپخش می‌کند و
می‌تواند اثر جانبی را دوباره اجرا کند. این پنجره خرابی حداقل یک‌بار،
قرارداد پیش‌فرض است. برای کارهای غیرهم‌توان مانند نوشتن پیکربندی، پاک‌سازی
فضای ذخیره‌سازی یا تأییدهای قابل‌مشاهده خارج از مسیر پاسخ، از
`createIngressEffectOnce(...)` در
`openclaw/plugin-sdk/ingress-effect-once` استفاده کنید. به هر فراخوانی
`eventId` پایدار ورود را به‌همراه نام اثر بدهید. برای هر صف ورود/حساب یک کمک‌تابع
بسازید و برای آن دامنه از `namespacePrefix` پایدار و یکتا استفاده کنید، زیرا شناسه‌های رویداد
انتقال ممکن است محلی صف باشند. کمک‌تابع ادعای پایدار خود را فقط پس از
موفقیت اثر ثبت می‌کند؛ اثر پرتاب‌شده ادعا را آزاد می‌کند تا تلاش مجدد تخلیه بتواند
آن را دوباره اجرا کند، درحالی‌که فراخوان‌های هم‌زمان منتظر ادعای فعال می‌مانند. خطاهای
وضعیت پایدار در صورت ارائه، `onDiskError` را فراخوانی می‌کنند و به‌جای بازگشت
به حافظه فرایند، رد می‌شوند.

مقدار `ttlMs` کمک‌تابع را دست‌کم برابر با نگهداشت سنگ‌قبر ورود کانال
به‌علاوه بیشینه تأخیر میان ثبت اثر و تکمیل ردیف، شامل
زمان توقف محدود و تلاش‌های مجدد تخلیه، تنظیم کنید. TTL رکورد اثر از هنگام ثبت آغاز می‌شود،
درحالی‌که نگهداشت سنگ‌قبر بعدتر و هنگام تکمیل شروع می‌شود؛ اگر طول عمر ردیف در انتظار
نامحدود باشد، هیچ TTL محدودی زمان توقف دلخواه را پوشش نمی‌دهد. پس از آنکه سنگ‌قبر دیگر
نتواند ردیف را بازپخش کند، رکوردهای اثر قدیمی سربار بی‌فایده‌اند. مقدار
`stateMaxEntries` را برای هر کلید متمایز رویداد/اثر که می‌تواند در آن
پنجره نگهداشت وجود داشته باشد اندازه‌گذاری کنید و سقف ورودی‌های تکمیل‌شده صف و
بیشینه اثرها در هر رویداد را در نظر بگیرید. سقف پایین‌تر، قدیمی‌ترین رکورد را پیش از TTL آن
بیرون می‌اندازد و اجازه می‌دهد آن اثر دوباره اجرا شود. اگر فرایند پس از موفقیت اثر اما پیش از
ثبت ادعا از کار بیفتد، یا ماندگاری شکست بخورد، یا رکورد زمانی منقضی شود که ردیف ورود آن هنوز
در انتظار است، پنجره‌های باقی‌مانده حداقل یک‌بار همچنان وجود خواهند داشت.

#### قرارداد راه‌اندازی مجدد در دامنه حساب

تغییرات پیکربندی کانال به‌طور پیش‌فرض کل کانال را راه‌اندازی مجدد می‌کنند. یک کانال چندحسابی
فقط زمانی می‌تواند `reload.accountScopedRestart: true` را تنظیم کند که تفکیک پیکربندی،
فیلدهای مشترک سراسری کانال به‌علاوه حساب انتخاب‌شده را بخواند و هرگز حساب هم‌سطح را
نخواند، و Gateway بتواند یک زمان اجرای `(channel, accountId)` را بدون
جایگزینی زمان‌های اجرای هم‌سطح متوقف و راه‌اندازی کند.

مسیر دامنه‌دار فقط بر تغییرات زیر
`channels.<channel>.accounts.<non-default-id>.*` اعمال می‌شود. تغییرات در فیلدهای مشترک کانال،
`accounts.default`، حساب‌های حذف‌شده یا تفکیک‌ناپذیر و تغییرات ترکیبی
که می‌توانند بر وراثت اثر بگذارند، به راه‌اندازی مجدد کل کانال ارتقا داده می‌شوند. Plugin‌هایی
که این قابلیت را فعال نمی‌کنند، همیشه از مسیر کل کانال استفاده می‌کنند.

برای کانال‌هایی که از تخلیه پایدار ورود استفاده می‌کنند، مسیر توقف پایشگر حساب
باید ابتدا همه پذیرش‌های پذیرفته‌شده انتقال را به پایان برساند، سپس تخلیه خود را دفع کند و در انتظار آن بماند.
راه‌اندازی حساب، همان صف کلیدگذاری‌شده با حساب را باز می‌کند که تخلیه اولیه آن،
ردیف‌های پایدار اعزام‌نشده را بازیابی می‌کند. گذر بازپخش دوم و مخصوص بارگذاری مجدد اضافه نکنید؛
بازیابی صف، مسیر متعارف راه‌اندازی مجدد است.

این پرچم را یک ادعای قابلیت بدانید، نه ترجیح عملکرد. آزمون‌های قرارداد
باید اثبات کنند که افزودن و ویرایش یک حساب نام‌گذاری‌شده، پیکربندی تفکیک‌شده حساب هم‌سطح را
بدون تغییر باقی می‌گذارد، توقف یک حساب فقط پایشگر و تخلیه همان حساب را به پایان می‌رساند،
و پایشگر تازه ردیف‌های آن حساب را دقیقاً یک‌بار بازیابی می‌کند.
اگر هر تضمینی قابل اثبات نیست، پرچم را حذف کنید.

### نشانگرهای تایپ

اگر کانال شما از نشانگرهای تایپ خارج از پاسخ‌های ورودی پشتیبانی می‌کند،
`heartbeat.sendTyping(...)` را در Plugin کانال در معرض استفاده قرار دهید. هسته آن را
با مقصد تفکیک‌شده تحویل Heartbeat پیش از شروع اجرای مدل Heartbeat فراخوانی می‌کند و
از چرخه عمر مشترک زنده‌نگه‌داشتن/پاک‌سازی تایپ استفاده می‌کند. وقتی پلتفرم به
سیگنال توقف صریح نیاز دارد، `heartbeat.clearTyping(...)` را اضافه کنید.

### پارامترهای منبع رسانه

اگر کانال شما پارامترهایی به ابزار پیام اضافه می‌کند که حامل منابع رسانه‌اند،
نام آن پارامترها را از طریق `plugin.actions.describeMessageTool(...).mediaSourceParams` در معرض استفاده قرار دهید.
هسته از آن فهرست صریح برای نرمال‌سازی مسیر جعبه شنی و خط‌مشی دسترسی رسانه خروجی
استفاده می‌کند؛ بنابراین Plugin‌ها برای پارامترهای آواتار، پیوست یا تصویر جلد
مختص ارائه‌دهنده به موارد خاص در هسته مشترک نیاز ندارند.

نقشه‌ای مبتنی بر کنش مانند `{ "set-profile": ["avatarUrl", "avatarPath"] }` را ترجیح دهید
تا کنش‌های نامرتبط آرگومان‌های رسانه‌ای کنش دیگری را به ارث نبرند. آرایهٔ تخت
همچنان برای پارامترهایی که عمداً میان همهٔ کنش‌های ارائه‌شده مشترک‌اند، کار می‌کند.

کانال‌هایی که باید یک URL عمومی موقت برای واکشی رسانه در سمت پلتفرم
ارائه کنند، می‌توانند از `createHostedOutboundMediaStore(...)` در
`openclaw/plugin-sdk/outbound-media` همراه با مخازن وضعیت Plugin استفاده کنند. تجزیهٔ مسیر پلتفرم
و اعمال توکن را در Plugin کانال نگه دارید؛ راهکار کمکی مشترک
فقط مالک بارگذاری رسانه، فرادادهٔ انقضا، ردیف‌های قطعه و پاک‌سازی است.

پیوست‌های ورودی از واقعیت‌های مرتب‌شده استفاده می‌کنند، نه فیلدهای موازی `Media*`. رکوردهای
کانال را با `toInboundMediaFacts(...)` از
`openclaw/plugin-sdk/channel-inbound` نرمال‌سازی کنید و هنگام ساخت
زمینهٔ ورودی، آن‌ها را به‌صورت `media` ارسال کنید. وقتی یک Plugin باید خواندن رسانهٔ محلی را مجاز کند،
`getAgentScopedMediaLocalRoots(...)` یا
`getAgentScopedMediaLocalRootsForSources(...)` را از زیرمسیر متمرکز
`openclaw/plugin-sdk/media-local-roots` وارد کنید. سازنده/نمای ریشهٔ قدیمی
`agent-media-payload` سازگاری منسوخ‌شده است.

### شکل‌دهی بار بومی

اگر کانال شما برای `message(action="send")` به شکل‌دهی ویژهٔ ارائه‌دهنده نیاز دارد،
`actions.prepareSendPayload(...)` را ترجیح دهید. کارت‌ها، بلوک‌ها، جاسازی‌ها یا
سایر داده‌های پایدار بومی را زیر `payload.channelData.<channel>` قرار دهید و اجازه دهید هسته
از طریق آداپتور خروجی/پیام ارسال کند. از `actions.handleAction(...)` برای ارسال
فقط به‌عنوان گزینهٔ بازگشت سازگاری برای بارهایی استفاده کنید که نمی‌توان آن‌ها را سریال‌سازی و
دوباره تلاش کرد.

### دستور زبان مکالمهٔ نشست

اگر پلتفرم شما دامنهٔ اضافی را درون شناسه‌های مکالمه ذخیره می‌کند، تجزیهٔ آن را
با `messaging.resolveSessionConversation(...)` در Plugin نگه دارید. این
قلاب استاندارد برای نگاشت `rawId` به شناسهٔ پایهٔ مکالمه، شناسهٔ اختیاری
رشته، `baseConversationId` صریح و هر
`parentConversationCandidates` است. وقتی `parentConversationCandidates` را برمی‌گردانید،
آن‌ها را از محدودترین والد تا گسترده‌ترین/پایه‌ترین مکالمه مرتب کنید.

`messaging.resolveParentConversationCandidates(...)` یک گزینهٔ بازگشت
سازگاری منسوخ‌شده برای Pluginهایی است که فقط روی شناسهٔ عمومی/خام به گزینه‌های بازگشت والد نیاز دارند.
اگر هر دو قلاب وجود داشته باشند، هسته ابتدا از
`resolveSessionConversation(...).parentConversationCandidates` استفاده می‌کند و فقط هنگامی
به `resolveParentConversationCandidates(...)` بازمی‌گردد که قلاب استاندارد آن‌ها را
حذف کرده باشد.

Pluginهای همراهی که پیش از راه‌اندازی رجیستری کانال به همین تجزیه نیاز دارند،
می‌توانند یک فایل سطح‌بالای `session-key-api.ts` با خروجی
`resolveSessionConversation(...)` منطبق ارائه کنند (Pluginهای Feishu و Telegram
را ببینید). هسته فقط زمانی از آن سطح امن برای راه‌اندازی اولیه استفاده می‌کند که رجیستری Plugin
زمان اجرا هنوز در دسترس نباشد.

وقتی کد Plugin باید فیلدهای مسیرمانند را نرمال‌سازی کند،
رشتهٔ فرزند را با مسیر والدش مقایسه کند یا از `{ channel, to, accountId, threadId }` یک
کلید پایدار حذف تکرار بسازد، از `openclaw/plugin-sdk/channel-route` استفاده کنید. این راهکار کمکی
شناسه‌های عددی رشته را همانند هسته نرمال‌سازی می‌کند، بنابراین آن را به مقایسه‌های موقتی
`String(threadId)` ترجیح دهید. Pluginهایی با دستور زبان هدف ویژهٔ ارائه‌دهنده
باید `messaging.resolveOutboundSessionRoute(...)` را ارائه کنند تا هسته
هویت نشست و رشتهٔ بومی ارائه‌دهنده را بدون واسطه‌های تجزیه‌گر دریافت کند.

### پشتیبانی از اتصال مکالمه در دامنهٔ حساب

وقتی کانال از اتصال‌های عمومی مکالمهٔ جاری
پشتیبانی می‌کند، `conversationBindings.supportsCurrentConversationBinding` را تنظیم کنید. `createChatChannelPlugin(...)`
به‌طور پیش‌فرض این قابلیت ایستا را روی `true` تنظیم می‌کند.

اگر پشتیبانی بر اساس حساب پیکربندی‌شده متفاوت است،
`conversationBindings.isCurrentConversationBindingSupported({ accountId })` را نیز پیاده‌سازی کنید.
هسته این قلاب همگام را فقط پس از فعال‌شدن قابلیت ایستا ارزیابی می‌کند.
برگرداندن `false` عملیات قابلیت عمومی مکالمهٔ جاری،
اتصال، جست‌وجو، فهرست‌کردن، به‌روزرسانی زمان و لغو اتصال را برای آن حساب از دسترس خارج می‌کند.
حذف قلاب باعث می‌شود قابلیت ایستا برای همهٔ حساب‌ها اعمال شود.

پاسخ را از پیکربندی ازپیش‌بارگذاری‌شدهٔ حساب یا وضعیت زمان اجرا تعیین کنید. این
قلاب فقط اتصال‌های عمومی مکالمهٔ جاری را کنترل می‌کند؛ جایگزین
قواعد اتصال پیکربندی‌شده یا مسیریابی نشست تحت مالکیت Plugin نمی‌شود. آزمون‌های قرارداد
باید حداقل یک حساب پشتیبانی‌شده و یک حساب پشتیبانی‌نشده را از طریق قرارداد
`ChannelPlugin["conversationBindings"]` خروجی‌گرفته‌شده از
`openclaw/plugin-sdk/channel-core` پوشش دهند.

## تأییدها و قابلیت‌های کانال

بیشتر Pluginهای کانال به کد ویژهٔ تأیید نیاز ندارند. هسته مالک
`/approve` در همان گفت‌وگو، بارهای دکمهٔ تأیید مشترک و تحویل عمومی جایگزین است.
`ChannelPlugin.approvals` حذف شده است؛ در عوض، واقعیت‌های تحویل/بومی/رندر/احراز هویت
تأیید را روی یک شیء `approvalCapability` قرار دهید. `plugin.auth` فقط
برای ورود/خروج است — هسته دیگر قلاب‌های احراز هویت تأیید را از آن شیء نمی‌خواند.

از `approvalCapability.delivery` فقط برای مسیریابی بومی تأیید یا جلوگیری از
گزینهٔ بازگشت، و از `approvalCapability.render` فقط زمانی استفاده کنید که کانال واقعاً به
بارهای تأیید سفارشی به‌جای رندرکنندهٔ مشترک نیاز دارد.

### احراز هویت تأیید

- `approvalCapability.authorizeActorAction` و
  `approvalCapability.getActionAvailabilityState` رابط استاندارد
  احراز هویت تأیید هستند.
- از `getActionAvailabilityState` برای دسترس‌بودن احراز هویت تأیید در همان گفت‌وگو استفاده کنید.
  تأییدکنندگان پیکربندی‌شده را حتی هنگامی که تحویل بومی
  غیرفعال است، برای `/approve` در دسترس نگه دارید؛ در عوض، برای راهنمایی تحویل/راه‌اندازی
  از وضعیت سطح آغازگر بومی استفاده کنید.
- اگر کانال شما تأییدهای اجرای بومی را ارائه می‌کند، هنگامی که
  وضعیت سطح آغازگر/کارخواه بومی با احراز هویت تأیید
  در همان گفت‌وگو متفاوت است، از `approvalCapability.getExecInitiatingSurfaceState` برای آن
  استفاده کنید. هسته از این قلاب ویژهٔ اجرا استفاده می‌کند تا `enabled` را از
  `disabled` متمایز کند، تصمیم بگیرد آیا کانال آغازگر از تأییدهای اجرای بومی
  پشتیبانی می‌کند و کانال را در راهنمایی گزینهٔ بازگشت کارخواه بومی بگنجاند.
  `createApproverRestrictedNativeApprovalCapability(...)` این مورد را برای
  حالت رایج تکمیل می‌کند.
- اگر کانالی بتواند هویت‌های پایدار پیام خصوصی شبیه مالک را از پیکربندی موجود استنباط کند،
  از `createResolvedApproverActionAuthAdapter` در
  `openclaw/plugin-sdk/approval-runtime` برای محدودکردن `/approve` در همان گفت‌وگو
  بدون افزودن منطق ویژهٔ تأیید به هسته استفاده کنید.
- اگر احراز هویت تأیید سفارشی عمداً فقط گزینهٔ بازگشت در همان گفت‌وگو را مجاز می‌کند،
  `markImplicitSameChatApprovalAuthorization({ authorized: true })` را از
  `openclaw/plugin-sdk/approval-auth-runtime` برگردانید؛ در غیر این صورت، هسته نتیجه را
  مجوز صریح تأییدکننده تلقی می‌کند.
- اگر یک فراخوان برگشتی بومی تحت مالکیت کانال تأییدها را مستقیماً حل‌وفصل می‌کند،
  پیش از حل‌وفصل از `isImplicitSameChatApprovalAuthorization(...)` استفاده کنید تا گزینهٔ بازگشت
  ضمنی همچنان از مجوزدهی عادی کنشگر کانال عبور کند.

### چرخهٔ عمر بار و راهنمای راه‌اندازی

- برای رفتار چرخهٔ عمر بار ویژهٔ کانال،
  مانند پنهان‌کردن اعلان‌های تکراری تأیید محلی یا ارسال نشانگرهای تایپ
  پیش از تحویل، از `outbound.shouldSuppressLocalPayloadPrompt` یا
  `outbound.beforeDeliverPayload` استفاده کنید.
- وقتی کانال می‌خواهد پاسخ مسیر غیرفعال
  گزینه‌های دقیق پیکربندی لازم برای فعال‌کردن تأییدهای اجرای بومی را توضیح دهد، از `approvalCapability.describeExecApprovalSetup` استفاده کنید.
  این قلاب `{ channel, channelLabel, accountId }` را دریافت می‌کند؛
  کانال‌های دارای حساب نام‌گذاری‌شده باید به‌جای پیش‌فرض‌های سطح‌بالا،
  مسیرهای در دامنهٔ حساب مانند
  `channels.<channel>.accounts.<id>.execApprovals.*` را رندر کنند.
- وقتی نمایش راهنمای شکست تأیید Plugin برای
  شکست‌های بدون مسیر و مهلت‌گذشتهٔ تأیید Plugin امن است، از `approvalCapability.describePluginApprovalSetup` استفاده کنید.
  `createApproverRestrictedNativeApprovalCapability(...)` این مورد را
  از `describeExecApprovalSetup` استنباط نمی‌کند؛ تنها زمانی همان راهکار کمکی را صریحاً
  ارسال کنید که تأییدهای Plugin و اجرا واقعاً از راه‌اندازی بومی یکسانی استفاده می‌کنند.

### تحویل بومی تأیید

اگر کانالی به تحویل بومی تأیید نیاز دارد، کد کانال را بر
نرمال‌سازی هدف به‌همراه واقعیت‌های انتقال/ارائه متمرکز نگه دارید. از
`createChannelExecApprovalProfile`، `createChannelNativeOriginTargetResolver`،
`createChannelApproverDmTargetResolver` و
`createApproverRestrictedNativeApprovalCapability` از
`openclaw/plugin-sdk/approval-runtime` استفاده کنید. واقعیت‌های ویژهٔ کانال را پشت
`approvalCapability.nativeRuntime`، ترجیحاً از طریق
`createChannelApprovalNativeRuntimeAdapter(...)` یا
`createLazyChannelApprovalNativeRuntimeAdapter(...)` قرار دهید تا هسته بتواند
گرداننده را مونتاژ کند و مالک پالایش درخواست، مسیریابی، حذف تکرار، انقضا، اشتراک Gateway
و اعلان‌های مسیریابی‌شده به محل دیگر باشد.

`nativeRuntime` به چند رابط کوچک‌تر تقسیم شده است:

- `availability` - اینکه آیا حساب پیکربندی شده و آیا یک درخواست
  باید پردازش شود
- `presentation` - نگاشت مدل نمای مشترک تأیید به
  بارهای بومی در انتظار/حل‌شده/منقضی‌شده یا کنش‌های نهایی
- `transport` - آماده‌سازی هدف‌ها و ارسال/به‌روزرسانی/حذف پیام‌های بومی
  تأیید
- `interactions` - قلاب‌های اختیاری اتصال/لغو اتصال/پاک‌کردن کنش برای دکمه‌ها
  یا واکنش‌های بومی، به‌همراه قلاب اختیاری `cancelDelivered`. وقتی
  `deliverPending` وضعیت درون‌پردازه‌ای یا پایدار
  (مانند مخزن هدف واکنش) را ثبت می‌کند، `cancelDelivered` را پیاده‌سازی کنید تا اگر
  توقف گرداننده تحویل را پیش از اجرای `bindPending` لغو کرد، یا هنگامی که
  `bindPending` هیچ دستگیره‌ای برنگرداند، آن وضعیت آزاد شود
- `observe` - قلاب‌های اختیاری عیب‌یابی تحویل

سایر راهکارهای کمکی تأیید:

- وقتی یک کانال هم از تحویل بومی مبدأ نشست و هم از هدف‌های صریح
  هدایت تأیید پشتیبانی می‌کند، از `createNativeApprovalChannelRouteGates` در
  `openclaw/plugin-sdk/approval-native-runtime` استفاده کنید. این
  راهکار کمکی انتخاب پیکربندی تأیید، مدیریت `mode`، فیلترهای عامل/نشست،
  اتصال حساب، تطبیق هدف نشست و تطبیق فهرست هدف را متمرکز می‌کند،
  درحالی‌که فراخوان‌ها همچنان مالک شناسهٔ کانال، حالت پیش‌فرض هدایت، جست‌وجوی حساب،
  بررسی فعال‌بودن انتقال، نرمال‌سازی هدف و تعیین هدف
  مبدأ نوبت هستند. از آن برای ایجاد پیش‌فرض‌های سیاست کانال تحت مالکیت هسته
  استفاده نکنید؛ حالت پیش‌فرض مستندشدهٔ کانال را صریحاً ارسال کنید.
- `createChannelNativeOriginTargetResolver` به‌طور پیش‌فرض از تطبیق‌دهندهٔ مشترک مسیر کانال
  برای هدف‌های `{ to, accountId, threadId }` استفاده می‌کند. فقط هنگامی
  `targetsMatch` را ارسال کنید که کانال قواعد هم‌ارزی ویژهٔ ارائه‌دهنده داشته باشد،
  مانند تطبیق پیشوند برچسب زمانی Slack. وقتی کانال باید
  شناسه‌های ارائه‌دهنده را پیش از اجرای تطبیق‌دهندهٔ پیش‌فرض مسیر
  یا فراخوان برگشتی سفارشی `targetsMatch` استاندارد کند و درعین‌حال هدف
  اصلی را برای تحویل حفظ کند، `normalizeTargetForMatch` را ارسال کنید. فقط هنگامی از `normalizeTarget` استفاده کنید
  که خود هدف تحویل تعیین‌شده باید استاندارد شود.
- اگر کانال به اشیای تحت مالکیت زمان اجرا مانند کارخواه، توکن، برنامهٔ Bolt
  یا گیرندهٔ Webhook نیاز دارد، آن‌ها را از طریق
  `openclaw/plugin-sdk/channel-runtime-context` ثبت کنید. رجیستری عمومی زمینهٔ زمان اجرا
  به هسته اجازه می‌دهد گرداننده‌های قابلیت‌محور را از وضعیت
  راه‌اندازی کانال، بدون افزودن کد واسط ویژهٔ تأیید، راه‌اندازی اولیه کند.
- فقط زمانی به سراغ `createChannelApprovalHandler` یا
  `createChannelNativeApprovalRuntime` سطح پایین‌تر بروید که رابط قابلیت‌محور
  هنوز به‌اندازهٔ کافی گویا نباشد.
- کانال‌های تأیید بومی باید هم `accountId` و هم `approvalKind`
  را از طریق آن راهکارهای کمکی مسیریابی کنند. `accountId` سیاست تأیید چندحسابی را
  در دامنهٔ حساب ربات درست نگه می‌دارد و `approvalKind` رفتار تأیید
  اجرا در برابر Plugin را بدون شاخه‌های سخت‌کدشده در هسته برای کانال
  در دسترس نگه می‌دارد.
- هسته مالک اعلان‌های تغییر مسیر تأیید نیز هست. Pluginهای کانال نباید
  پیام‌های پیگیری «تأیید به پیام‌های خصوصی / کانال دیگری رفت» خود را از
  `createChannelNativeApprovalRuntime` ارسال کنند؛ در عوض، مسیریابی دقیق مبدأ +
  پیام خصوصی تأییدکننده را از طریق راهکارهای کمکی مشترک قابلیت تأیید ارائه کنند و اجازه دهند
  هسته پیش از ارسال هر اعلان به گفت‌وگوی آغازگر، تحویل‌های واقعی را تجمیع کند.
- نوع شناسهٔ تأیید تحویل‌شده را از ابتدا تا انتها حفظ کنید. کارخواه‌های بومی نباید
  مسیریابی تأیید اجرا در برابر Plugin را از وضعیت محلی کانال حدس بزنند یا بازنویسی کنند.
- آن `approvalKind` صریح را به `resolveApprovalOverGateway` ارسال کنید. این کار از
  سرویس استاندارد `approval.resolve` استفاده می‌کند و هنگامی که سطح دیگری ابتدا پاسخ دهد،
  برندهٔ ثبت‌شده را برمی‌گرداند. ورودی صریح قدیمی‌تر `resolveMethod`
  برای کنترل‌های مبتنی بر فرمان باقی مانده است؛ کنش‌های بومی جدید نباید از آن استفاده کنند یا
  نوع را از یک شناسه استنباط کنند.
- انواع مختلف تأیید می‌توانند عمداً سطوح بومی متفاوتی ارائه کنند.
  نمونه‌های همراه فعلی: Matrix همان مسیریابی بومی پیام خصوصی/کانال و
  تجربهٔ کاربری واکنش را برای تأییدهای اجرا و Plugin حفظ می‌کند، درحالی‌که همچنان اجازه می‌دهد
  احراز هویت بر اساس نوع تأیید متفاوت باشد؛ Slack مسیریابی بومی تأیید را
  برای شناسه‌های اجرا و Plugin در دسترس نگه می‌دارد.
- `createApproverRestrictedNativeApprovalAdapter` همچنان به‌عنوان یک
  پوشش سازگاری وجود دارد، اما کد جدید باید سازندهٔ قابلیت را ترجیح دهد
  و `approvalCapability` را روی Plugin ارائه کند.

### زیرمسیرهای محدودتر زمان اجرای تأیید

برای نقاط ورود پرترافیک کانال، وقتی فقط به یک بخش از این خانواده نیاز دارید،
این زیرمسیرهای محدودتر را به مجموعهٔ گسترده‌تر
`approval-runtime` ترجیح دهید:

- `openclaw/plugin-sdk/approval-auth-runtime`
- `openclaw/plugin-sdk/approval-client-runtime`
- `openclaw/plugin-sdk/approval-delivery-runtime`
- `openclaw/plugin-sdk/approval-gateway-runtime`
- `openclaw/plugin-sdk/approval-reference-runtime`
- `openclaw/plugin-sdk/approval-handler-adapter-runtime`
- `openclaw/plugin-sdk/approval-handler-runtime`
- `openclaw/plugin-sdk/approval-native-runtime`
- `openclaw/plugin-sdk/approval-reply-runtime`
- `openclaw/plugin-sdk/channel-runtime-context`

به همین ترتیب، هنگامی که به همه آن‌ها نیاز ندارید، `openclaw/plugin-sdk/reply-runtime`،
`openclaw/plugin-sdk/reply-dispatch-runtime`،
`openclaw/plugin-sdk/reply-reference` و
`openclaw/plugin-sdk/reply-chunking` را به سطوح فراگیرتر ترجیح دهید.

### زیرمسیرهای راه‌اندازی

- `openclaw/plugin-sdk/setup-runtime` شامل ابزارهای کمکی راه‌اندازی ایمن برای زمان اجرا است:
  `createSetupTranslator`، آداپتورهای وصله راه‌اندازی ایمن برای import
  (`createPatchedAccountSetupAdapter`، `createEnvPatchedAccountSetupAdapter`،
  `createSetupInputPresenceValidator`)، خروجی یادداشت جست‌وجو،
  `promptResolvedAllowFrom`، `splitSetupEntries` و سازنده‌های تفویض‌شده
  پراکسی راه‌اندازی.
- `openclaw/plugin-sdk/channel-setup` شامل سازنده‌های راه‌اندازی نصب اختیاری
  به‌همراه چند سازه اولیه ایمن برای راه‌اندازی است: `createOptionalChannelSetupSurface`،
  `createOptionalChannelSetupAdapter`، `createOptionalChannelSetupWizard`،
  `DEFAULT_ACCOUNT_ID`، `createTopLevelChannelDmPolicy`،
  `setSetupChannelEnabled` و `splitSetupEntries`.
- تنها زمانی از مرز گسترده‌تر `openclaw/plugin-sdk/setup` استفاده کنید که به
  ابزارهای کمکی سنگین‌تر و مشترک راه‌اندازی/پیکربندی، مانند
  `moveSingleAccountChannelSectionToDefaultAccount(...)` نیز نیاز دارید.

اگر کانال شما فقط می‌خواهد در سطوح راه‌اندازی اعلام کند «ابتدا این Plugin را نصب کنید»،
`createOptionalChannelSetupSurface(...)` را ترجیح دهید. آداپتور/ویزارد تولیدشده
هنگام نوشتن پیکربندی و نهایی‌سازی به‌صورت بسته شکست می‌خورد و همان پیام
الزام نصب را در اعتبارسنجی، نهایی‌سازی و متن پیوند مستندات دوباره استفاده
می‌کند.

اگر کانال شما از راه‌اندازی یا احراز هویت مبتنی بر متغیرهای محیطی پشتیبانی می‌کند، آن را از طریق
شمای پیکربندی کانال و توصیفگرهای راه‌اندازی ارائه دهید. `envVars` زمان اجرای کانال یا
ثابت‌های محلی را فقط برای متن ویژه اپراتور نگه دارید.

اگر کانال شما می‌تواند پیش از آغاز زمان اجرای Plugin در `status`، `channels list`، `channels status` یا
اسکن‌های SecretRef ظاهر شود، `openclaw.setupEntry` را در
`package.json` اضافه کنید. import این نقطه ورود باید در مسیرهای فرمان
فقط‌خواندنی ایمن باشد و فراداده کانال، آداپتور پیکربندی ایمن برای راه‌اندازی،
آداپتور وضعیت و فراداده مقصد اسرار کانال موردنیاز برای آن
خلاصه‌ها را برگرداند. کلاینت‌ها، شنونده‌ها یا زمان‌های اجرای انتقال را از ورودی
راه‌اندازی شروع نکنید.

مسیر import ورودی اصلی کانال را نیز محدود نگه دارید. کشف می‌تواند
ورودی و ماژول Plugin کانال را برای ثبت قابلیت‌ها ارزیابی کند، بدون آنکه
کانال را فعال کند. فایل‌هایی مانند `channel-plugin-api.ts` باید
شیء Plugin کانال را بدون import کردن ویزاردهای راه‌اندازی، کلاینت‌های
انتقال، شنونده‌های سوکت، اجراکننده‌های زیرفرایند یا ماژول‌های راه‌اندازی سرویس صادر کنند.
آن بخش‌های زمان اجرا را در ماژول‌هایی قرار دهید که از `registerFull(...)`، تنظیم‌کننده‌های
زمان اجرا یا آداپتورهای قابلیت با بارگذاری تنبل بارگذاری می‌شوند.

### دیگر زیرمسیرهای محدود کانال

برای دیگر مسیرهای داغ کانال، ابزارهای کمکی محدود را به سطوح قدیمی
گسترده‌تر ترجیح دهید:

- `openclaw/plugin-sdk/account-core`، `openclaw/plugin-sdk/account-id`،
  `openclaw/plugin-sdk/account-resolution` و
  `openclaw/plugin-sdk/account-helpers` برای پیکربندی چندحسابی و
  بازگشت به حساب پیش‌فرض
- `openclaw/plugin-sdk/inbound-envelope` و
  `openclaw/plugin-sdk/channel-inbound` برای مسیر/پاکت ورودی و
  سیم‌کشی ثبت و ارسال
- `openclaw/plugin-sdk/channel-targets` برای ابزارهای کمکی تجزیه مقصد
- `openclaw/plugin-sdk/channel-outbound` برای نماینده‌های هویت/ارسال خروجی
  و برنامه‌ریزی محموله نوع‌دار
- `buildThreadAwareOutboundSessionRoute(...)` از
  `openclaw/plugin-sdk/channel-core` هنگامی که یک مسیر خروجی باید یک `replyToId`/`threadId` صریح را حفظ کند یا نشست فعلی `:thread:` را
  پس از آنکه کلید نشست پایه همچنان مطابقت دارد بازیابی کند. Pluginهای ارائه‌دهنده می‌توانند
  تقدم، رفتار پسوند و عادی‌سازی شناسه رشته را هنگامی که
  پلتفرمشان معنای بومی تحویل رشته‌ای دارد، بازنویسی کنند.
- `openclaw/plugin-sdk/thread-bindings-runtime` برای چرخه عمر اتصال رشته
  و ثبت آداپتور

کانال‌های صرفاً احراز هویت معمولاً می‌توانند به مسیر پیش‌فرض بسنده کنند: هسته
تأییدها را مدیریت می‌کند و Plugin فقط قابلیت‌های خروجی/احراز هویت را ارائه می‌دهد. کانال‌های
تأیید بومی مانند Matrix، Slack، Telegram و انتقال‌های گفت‌وگوی سفارشی
باید به‌جای ساخت چرخه عمر تأیید اختصاصی خود، از ابزارهای کمکی بومی مشترک استفاده کنند.

## سیاست اشاره ورودی

مدیریت اشاره ورودی را در دو لایه جدا نگه دارید:

- گردآوری شواهد تحت مالکیت Plugin
- ارزیابی سیاست مشترک

برای تصمیم‌های سیاست اشاره از `openclaw/plugin-sdk/channel-mention-gating` استفاده کنید.
تنها هنگامی از `openclaw/plugin-sdk/channel-inbound` استفاده کنید که به barrel گسترده‌تر
ابزارهای کمکی ورودی نیاز دارید.

مناسب برای منطق محلی Plugin:

- تشخیص پاسخ به ربات
- تشخیص نقل‌قول از ربات
- بررسی مشارکت در رشته
- استثناهای پیام سرویس/سیستم
- حافظه‌های نهان بومی پلتفرم که برای اثبات مشارکت ربات لازم‌اند

مناسب برای ابزار کمکی مشترک:

- `requireMention`
- نتیجه اشاره صریح
- فهرست مجاز اشاره ضمنی
- دور زدن فرمان
- تصمیم نهایی برای رد کردن

جریان ترجیحی:

1. واقعیت‌های محلی اشاره را محاسبه کنید.
2. آن واقعیت‌ها را به `resolveInboundMentionDecision({ facts, policy })` بدهید.
3. از `decision.effectiveWasMentioned`، `decision.shouldBypassMention` و
   `decision.shouldSkip` در دروازه ورودی خود استفاده کنید.

```typescript
import {
  implicitMentionKindWhen,
  matchesMentionWithExplicit,
  resolveInboundMentionDecision,
} from "openclaw/plugin-sdk/channel-inbound";
import { resolveChannelImplicitMentions } from "openclaw/plugin-sdk/channel-ingress-runtime";

const wasMentioned = matchesMentionWithExplicit({
  text,
  mentionRegexes,
  explicit: {
    hasAnyMention,
    isExplicitlyMentioned,
    canResolveExplicit,
  },
});

const facts = {
  canDetectMention: true,
  wasMentioned,
  hasAnyMention,
  implicitMentionKinds: [
    ...implicitMentionKindWhen("reply_to_bot", isReplyToBot),
    ...implicitMentionKindWhen("quoted_bot", isQuoteOfBot),
  ],
};

const implicitMentions = resolveChannelImplicitMentions({
  cfg,
  channel: channelId,
  accountId,
});

const decision = resolveInboundMentionDecision({
  facts,
  policy: {
    isGroup,
    requireMention,
    implicitMentions,
    allowTextCommands,
    hasControlCommand,
    commandAuthorized,
  },
});

if (decision.shouldSkip) return;
```

`matchesMentionWithExplicit(...)` یک مقدار بولی برمی‌گرداند. `hasAnyMention`،
`isExplicitlyMentioned` و `canResolveExplicit` از فراداده بومی اشاره خود کانال
(موجودیت‌های پیام، پرچم‌های پاسخ به ربات و موارد مشابه) می‌آیند؛
وقتی پلتفرم شما نمی‌تواند آن‌ها را تشخیص دهد، مقادیر `false`/`undefined` را ارائه دهید.

`api.runtime.channel.mentions` همان ابزارهای کمکی مشترک اشاره را برای
Pluginهای کانال همراهی ارائه می‌کند که از قبل به تزریق زمان اجرا وابسته‌اند:
`buildMentionRegexes`، `matchesMentionPatterns`، `matchesMentionWithExplicit`،
`implicitMentionKindWhen`، `resolveInboundMentionDecision`.

اگر فقط به `implicitMentionKindWhen` و `resolveInboundMentionDecision` نیاز دارید،
برای جلوگیری از بارگذاری ابزارهای کمکی نامرتبط زمان اجرای ورودی، از
`openclaw/plugin-sdk/channel-mention-gating` import کنید.

## راهنمای گام‌به‌گام

<Steps>
  <a id="step-1-package-and-manifest"></a>
  <Step title="بسته و مانیفست">
    فایل‌های استاندارد Plugin را ایجاد کنید. فیلد `channels` در
    `openclaw.plugin.json` (نه فیلد `kind`) مشخص می‌کند که یک مانیفست
    مالک یک کانال است. برای سطح کامل فراداده بسته، به
    [راه‌اندازی و پیکربندی Plugin](/fa/plugins/sdk-setup#openclaw-channel) مراجعه کنید:

    <CodeGroup>
    ```json package.json
    {
      "name": "@myorg/openclaw-acme-chat",
      "version": "1.0.0",
      "type": "module",
      "openclaw": {
        "extensions": ["./index.ts"],
        "setupEntry": "./setup-entry.ts",
        "channel": {
          "id": "acme-chat",
          "label": "گفت‌وگوی Acme",
          "blurb": "OpenClaw را به گفت‌وگوی Acme متصل کنید."
        }
      }
    }
    ```

    ```json openclaw.plugin.json
    {
      "id": "acme-chat",
      "channels": ["acme-chat"],
      "name": "گفت‌وگوی Acme",
      "description": "Plugin کانال گفت‌وگوی Acme",
      "configSchema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {}
      },
      "channelConfigs": {
        "acme-chat": {
          "schema": {
            "type": "object",
            "additionalProperties": false,
            "properties": {
              "token": { "type": "string" },
              "allowFrom": {
                "type": "array",
                "items": { "type": "string" }
              }
            }
          },
          "uiHints": {
            "token": {
              "label": "توکن ربات",
              "sensitive": true
            }
          }
        }
      }
    }
    ```
    </CodeGroup>

    `configSchema`، `plugins.entries.acme-chat.config` را اعتبارسنجی می‌کند. از آن برای
    تنظیمات تحت مالکیت Plugin که پیکربندی حساب کانال نیستند استفاده کنید.
    `channelConfigs.acme-chat.schema`، `channels.acme-chat` را اعتبارسنجی می‌کند و
    منبع مسیر سردی است که پیش از بارگذاری زمان اجرای Plugin توسط شمای پیکربندی،
    راه‌اندازی و سطوح UI استفاده می‌شود. برای مرجع کامل فیلدهای
    سطح بالا، به [مانیفست Plugin](/fa/plugins/manifest) مراجعه کنید.

  </Step>

  <Step title="ساخت شیء Plugin کانال">
    رابط `ChannelPlugin` سطوح آداپتور اختیاری بسیاری دارد. با
    حداقل موارد — `id`، `config` و `setup` — شروع کنید و آداپتورها را هر زمان به آن‌ها
    نیاز داشتید اضافه کنید.

    `src/channel.ts` را ایجاد کنید:

    ```typescript src/channel.ts
    import {
      createChatChannelPlugin,
      createChannelPluginBase,
    } from "openclaw/plugin-sdk/channel-core";
    import type { OpenClawConfig } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatApi } from "./client.js"; // your platform API client

    type ResolvedAccount = {
      accountId: string | null;
      token: string;
      allowFrom: string[];
      dmPolicy: string | undefined;
    };

    function resolveAccount(
      cfg: OpenClawConfig,
      accountId?: string | null,
    ): ResolvedAccount {
      const section = (cfg.channels as Record<string, any>)?.["acme-chat"];
      const token = section?.token;
      if (!token) throw new Error("acme-chat: token is required");
      return {
        accountId: accountId ?? null,
        token,
        allowFrom: section?.allowFrom ?? [],
        dmPolicy: section?.dmSecurity,
      };
    }

    export const acmeChatPlugin = createChatChannelPlugin<ResolvedAccount>({
      base: createChannelPluginBase({
        id: "acme-chat",
        // Account resolution/inspection belongs on `config`, not `setup`.
        // `setup` covers onboarding writes (applyAccountConfig, validateInput).
        config: {
          listAccountIds: () => ["default"],
          resolveAccount,
          inspectAccount(cfg, accountId) {
            const section =
              (cfg.channels as Record<string, any>)?.["acme-chat"];
            return {
              enabled: Boolean(section?.token),
              configured: Boolean(section?.token),
              tokenStatus: section?.token ? "available" : "missing",
            };
          },
        },
        setup: {
          applyAccountConfig: ({ cfg, input }) => ({
            ...cfg,
            channels: {
              ...cfg.channels,
              "acme-chat": { ...(cfg.channels as any)?.["acme-chat"], ...input },
            },
          }),
        },
      }),

      // DM security: who can message the bot
      security: {
        dm: {
          channelKey: "acme-chat",
          resolvePolicy: (account) => account.dmPolicy,
          resolveAllowFrom: (account) => account.allowFrom,
          defaultPolicy: "allowlist",
        },
      },

      // Pairing: approval flow for new DM contacts
      pairing: {
        text: {
          idLabel: "Acme Chat username",
          message: "Send this code to verify your identity:",
          notify: async ({ target, code }) => {
            await acmeChatApi.sendDm(target, `Pairing code: ${code}`);
          },
        },
      },

      // Threading: how replies are delivered
      threading: { topLevelReplyToMode: "reply" },

      // Outbound: send messages to the platform
      outbound: {
        attachedResults: {
          channel: "acme-chat",
          sendText: async (params) => {
            const result = await acmeChatApi.sendMessage(
              params.to,
              params.text,
            );
            return { messageId: result.id };
          },
        },
        base: {
          sendMedia: async (params) => {
            await acmeChatApi.sendFile(params.to, params.filePath);
          },
        },
      },
    });
    ```

    برای کانال‌هایی که هم کلیدهای متعارف سطح‌بالای DM و هم کلیدهای تودرتوی قدیمی را می‌پذیرند، از راهنماهای `plugin-sdk/channel-config-helpers` استفاده کنید: `resolveChannelDmAccess`، `resolveChannelDmPolicy`، `resolveChannelDmAllowFrom` و `normalizeChannelDmPolicy` مقادیر محلی حساب را مقدم بر مقادیر ارث‌رسیده از ریشه نگه می‌دارند. همان تفکیک‌گر را از طریق `normalizeLegacyDmAliases` با ترمیم doctor جفت کنید تا زمان اجرا و مهاجرت قرارداد یکسانی را بخوانند.

    <Accordion title="آنچه createChatChannelPlugin برای شما انجام می‌دهد">
      به‌جای پیاده‌سازی دستی رابط‌های آداپتور سطح‌پایین، گزینه‌های
      اعلانی را ارائه می‌کنید و سازنده آن‌ها را ترکیب می‌کند:

      | گزینه | آنچه متصل می‌کند |
      | --- | --- |
      | `security.dm` | تفکیک‌گر امنیتی DM با محدوده‌بندی بر اساس فیلدهای پیکربندی |
      | `pairing.text` | جریان جفت‌سازی متنی DM با تبادل کد |
      | `threading` | تفکیک‌گر حالت پاسخ (ثابت، مختص حساب یا سفارشی) |
      | `outbound.attachedResults` | توابع ارسالی که فراداده نتیجه (شناسه‌های پیام) را برمی‌گردانند؛ به یک شناسه هم‌سطح `channel` نیاز دارد تا هسته بتواند نتیجه تحویل بازگشتی را نشانه‌گذاری کند |

      اگر به کنترل کامل نیاز دارید، می‌توانید به‌جای گزینه‌های اعلانی،
      اشیای خام آداپتور را نیز ارائه کنید.

      آداپتورهای خام خروجی می‌توانند تابع `chunker(text, limit, ctx)` را تعریف کنند.
      `ctx.formatting` اختیاری تصمیم‌های قالب‌بندی زمان تحویل،
      مانند `maxLinesPerMessage`، را حمل می‌کند؛ پیش از ارسال آن را اعمال کنید تا
      رشته‌بندی پاسخ و مرزهای قطعه‌ها فقط یک‌بار توسط تحویل خروجی مشترک
      تفکیک شوند. زمینه‌های ارسال همچنین، هنگامی که یک مقصد پاسخ بومی تفکیک شده باشد،
      شامل `replyToIdSource` (`implicit` یا `explicit`)
      هستند تا راهنماهای محموله بتوانند برچسب‌های صریح پاسخ را بدون مصرف یک جایگاه
      ضمنی و یک‌بارمصرف پاسخ حفظ کنند.
    </Accordion>

    ### آداپتورهای خط‌مشی ابزار گروه

    کانالی که `group.resolveToolPolicy` را پیاده‌سازی می‌کند و از
    `toolsBySender` پشتیبانی می‌کند، باید `ChannelGroupContext` کامل را به
    تفکیک‌گر خط‌مشی مشترک خود ارسال کند. به‌طور خاص، با نادیده‌گرفتن هم‌پوشانی‌های
    مختص فرستنده در محدوده‌های گروه منطبق و نویسه عام، به `senderPolicyMode: "never"`
    پایبند باشید و در عین حال خط‌مشی پایه `tools` را همچنان اعمال کنید.

    OpenClaw این حالت را فقط برای اجرای قابل‌اعتماد و غیرورودی تنظیم می‌کند که اختیار
    فرستنده آن از قبل در یک پوشش تحت مالکیت سرور ثبت شده باشد؛ مانند یک اجرای
    زمان‌بندی‌شده با سقف صریح. Pluginها نباید این حالت را از فراداده ورودی استخراج کنند،
    آن را به‌عنوان وضعیت کانال پایدار کنند یا به‌صورت پیکربندی در معرض دسترس قرار دهند.
    یک آزمون آداپتور اضافه کنید که ثابت کند این حالت یک ورودی نویسه عام
    `toolsBySender` را بدون حذف محدودیت پایه منطبق `tools`
    نادیده می‌گیرد.

  </Step>

  <Step title="نقطه ورود را متصل کنید">
    `index.ts` را ایجاد کنید:

    ```typescript index.ts
    import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineChannelPluginEntry({
      id: "acme-chat",
      name: "Acme Chat",
      description: "Acme Chat channel plugin",
      plugin: acmeChatPlugin,
      registerCliMetadata(api) {
        api.registerCli(
          ({ program }) => {
            program
              .command("acme-chat")
              .description("Acme Chat management");
          },
          {
            descriptors: [
              {
                name: "acme-chat",
                description: "Acme Chat management",
                hasSubcommands: false,
              },
            ],
          },
        );
      },
      registerFull(api) {
        api.registerGatewayMethod(/* ... */);
      },
    });
    ```

    توصیف‌گرهای CLI متعلق به کانال را در `registerCliMetadata(...)` قرار دهید تا OpenClaw
    بتواند آن‌ها را در راهنمای ریشه بدون فعال‌کردن کامل زمان اجرای کانال نمایش دهد،
    در حالی که بارگذاری‌های کامل عادی همچنان همان توصیف‌گرها را برای ثبت واقعی فرمان
    دریافت می‌کنند. `registerFull(...)` را برای کارهای صرفاً زمان اجرا نگه دارید.
    `defineChannelPluginEntry` جداسازی حالت ثبت را به‌طور خودکار مدیریت می‌کند.
    اگر `registerFull(...)` متدهای RPC در Gateway را ثبت می‌کند، از یک
    پیشوند مختص Plugin استفاده کنید. فضاهای نام مدیریتی هسته (`config.*`،
    `exec.approvals.*`، `wizard.*`، `update.*`) رزروشده باقی می‌مانند و همیشه
    به `operator.admin` تفکیک می‌شوند. برای مشاهده همه گزینه‌ها به
    [نقاط ورود](/fa/plugins/sdk-entrypoints#definechannelpluginentry) مراجعه کنید.

  </Step>

  <Step title="یک ورودی راه‌اندازی اضافه کنید">
    برای بارگذاری سبک هنگام فرایند آغازین، `setup-entry.ts` را ایجاد کنید:

    ```typescript setup-entry.ts
    import { defineSetupPluginEntry } from "openclaw/plugin-sdk/channel-core";
    import { acmeChatPlugin } from "./src/channel.js";

    export default defineSetupPluginEntry(acmeChatPlugin);
    ```

    وقتی کانال غیرفعال یا پیکربندی‌نشده باشد، OpenClaw این ورودی را به‌جای ورودی کامل بارگذاری می‌کند.
    این کار از بارگذاری کد سنگین زمان اجرا در جریان‌های راه‌اندازی جلوگیری می‌کند.
    برای جزئیات، [راه‌اندازی و پیکربندی](/fa/plugins/sdk-setup#setup-entry) را ببینید.

    کانال‌های همراه فضای کاری که خروجی‌های امن برای راه‌اندازی را در ماژول‌های جانبی
    تفکیک می‌کنند، هنگامی که به یک تنظیم‌کننده صریح زمان اجرا در زمان راه‌اندازی
    نیز نیاز دارند، می‌توانند از `defineBundledChannelSetupEntry(...)` در
    `openclaw/plugin-sdk/channel-entry-contract` استفاده کنند.

  </Step>

  <Step title="مدیریت پیام‌های ورودی">
    Plugin شما باید پیام‌ها را از پلتفرم دریافت و آن‌ها را به
    OpenClaw ارسال کند. الگوی معمول، Webhookی است که درخواست را تأیید می‌کند و
    آن را از طریق مدیریت‌کننده ورودی کانال شما هدایت می‌کند:

    ```typescript
    registerFull(api) {
      api.registerHttpRoute({
        path: "/acme-chat/webhook",
        auth: "plugin", // احراز هویت مدیریت‌شده توسط Plugin (امضاها را خودتان تأیید کنید)
        handler: async (req, res) => {
          const event = parseWebhookPayload(req);

          // مدیریت‌کننده ورودی شما پیام را به OpenClaw هدایت می‌کند.
          // اتصال دقیق به SDK پلتفرم شما بستگی دارد -
          // یک نمونه واقعی را در بسته Plugin همراه Microsoft Teams یا Google Chat ببینید.
          await handleAcmeChatInbound(api, event);

          res.statusCode = 200;
          res.end("ok");
          return true;
        },
      });
    }
    ```

    <Note>
      مدیریت پیام‌های ورودی مختص هر کانال است. هر Plugin کانال
      پایپ‌لاین ورودی خودش را در اختیار دارد. برای مشاهده الگوهای واقعی، Pluginهای کانال همراه
      (برای مثال، بسته Plugin مربوط به Microsoft Teams یا Google Chat) را بررسی کنید.
    </Note>

  </Step>

<a id="step-6-test"></a>
<Step title="آزمایش">
آزمایش‌های هم‌مکان را در `src/channel.test.ts` بنویسید:

    ```typescript src/channel.test.ts
    import { describe, it, expect } from "vitest";
    import { acmeChatPlugin } from "./channel.js";

    describe("Plugin acme-chat", () => {
      it("حساب را از پیکربندی استخراج می‌کند", () => {
        const cfg = {
          channels: {
            "acme-chat": { token: "test-token", allowFrom: ["user1"] },
          },
        } as any;
        const account = acmeChatPlugin.config.resolveAccount(cfg, undefined);
        expect(account.token).toBe("test-token");
      });

      it("حساب را بدون ایجاد عینی اسرار بررسی می‌کند", () => {
        const cfg = {
          channels: { "acme-chat": { token: "test-token" } },
        } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(true);
        expect(result.tokenStatus).toBe("available");
      });

      it("نبود پیکربندی را گزارش می‌کند", () => {
        const cfg = { channels: {} } as any;
        const result = acmeChatPlugin.config.inspectAccount!(cfg, undefined);
        expect(result.configured).toBe(false);
      });
    });
    ```

    ```bash
    pnpm test <bundled-plugin-root>/acme-chat/
    ```

    برای راهنماهای مشترک آزمایش، [آزمایش](/fa/plugins/sdk-testing) را ببینید.

</Step>
</Steps>

## ساختار فایل

```text
<bundled-plugin-root>/acme-chat/
├── package.json              # فراداده openclaw.channel
├── openclaw.plugin.json      # مانیفست دارای شِمای پیکربندی
├── index.ts                  # defineChannelPluginEntry
├── setup-entry.ts            # defineSetupPluginEntry
├── api.ts                    # خروجی‌های عمومی (اختیاری)
├── runtime-api.ts            # خروجی‌های داخلی زمان اجرا (اختیاری)
└── src/
    ├── channel.ts            # ChannelPlugin از طریق createChatChannelPlugin
    ├── channel.test.ts       # آزمایش‌ها
    ├── client.ts             # کلاینت API پلتفرم
    └── runtime.ts            # مخزن زمان اجرا (در صورت نیاز)
```

## موضوعات پیشرفته

<CardGroup cols={2}>
  <Card title="گزینه‌های رشته‌بندی" icon="git-branch" href="/fa/plugins/sdk-entrypoints#registration-mode">
    حالت‌های پاسخ ثابت، محدود به حساب یا سفارشی
  </Card>
  <Card title="یکپارچه‌سازی ابزار پیام" icon="puzzle" href="/fa/plugins/architecture#channel-plugins-and-the-shared-message-tool">
    describeMessageTool و کشف کنش‌ها
  </Card>
  <Card title="تفکیک مقصد" icon="crosshair" href="/fa/plugins/architecture-internals#channel-target-resolution">
    inferTargetChatType، looksLikeId، reservedLiterals، resolveTarget
  </Card>
  <Card title="کمک‌تابع‌های زمان اجرا" icon="settings" href="/fa/plugins/sdk-runtime">
    TTS، STT، رسانه و زیرعامل از طریق api.runtime
  </Card>
  <Card title="API ورودی کانال" icon="bolt" href="/fa/plugins/sdk-channel-inbound">
    چرخهٔ عمر مشترک رویداد ورودی: دریافت، تفکیک، ثبت، ارسال و نهایی‌سازی
  </Card>
</CardGroup>

<Note>
برخی نقاط اتصال کمک‌تابعِ همراه همچنان برای نگهداشت Pluginهای همراه و
سازگاری وجود دارند. آن‌ها الگوی توصیه‌شده برای Pluginهای کانال جدید نیستند؛
مگر اینکه مستقیماً در حال نگهداشت آن خانواده از Pluginهای همراه باشید، زی مسیرهای فرعی عمومی
کانال/راه‌اندازی/پاسخ/زمان اجرا در سطح مشترک SDK استفاده کنید.
</Note>

## گام‌های بعدی

- [Pluginهای ارائه‌دهنده](/fa/plugins/sdk-provider-plugins) - اگر Plugin شما مدل‌ها را نیز ارائه می‌کند
- [نمای کلی SDK](/fa/plugins/sdk-overview) - مرجع کامل واردکردن مسیرهای فرعی
- [آزمایش SDK](/fa/plugins/sdk-testing) - ابزارهای کمکی آزمایش و آزمایش‌های قرارداد
- [مانیفست Plugin](/fa/plugins/manifest) - شِمای کامل مانیفست

## مرتبط

- [راه‌اندازی SDK برای Plugin](/fa/plugins/sdk-setup)
- [ساخت Pluginها](/fa/plugins/building-plugins)
- [Pluginهای چارچوب عامل](/fa/plugins/sdk-agent-harness)
