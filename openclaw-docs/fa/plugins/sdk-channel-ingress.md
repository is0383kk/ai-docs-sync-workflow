---
read_when:
    - ساخت یا مهاجرت Plugin کانال پیام‌رسانی
    - تغییر فهرست‌های مجاز پیام خصوصی یا گروه، دروازه‌های مسیریابی، احراز هویت فرمان‌ها، احراز هویت رویدادها یا فعال‌سازی با اشاره‌کردن
    - بازبینی حذف اطلاعات حساس در ورودی کانال یا مرزهای سازگاری SDK
sidebarTitle: Channel Ingress
summary: API آزمایشی ورودی کانال برای مجوزدهی پیام‌های ورودی
title: API ورودی کانال
x-i18n:
    generated_at: "2026-07-27T14:30:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 60feecb7bcf203cf37d2543a7855e89b5bfb2eb9d8263d804219e140facb8fc6
    source_path: plugins/sdk-channel-ingress.md
    workflow: 16
---

ورودی کانال، مرز آزمایشی کنترل دسترسی برای رویدادهای ورودی
کانال است. Pluginها مالک واقعیت‌های پلتفرم و اثرات جانبی هستند؛ هسته مالک
سیاست عمومی است: فهرست‌های مجاز پیام خصوصی/گروه، ورودی‌های پیام خصوصی در ذخیره‌گاه جفت‌سازی، دروازه‌های مسیر،
دروازه‌های فرمان، احراز هویت رویداد، فعال‌سازی با اشاره، عیب‌یابی‌های پوشیده‌شده و
پذیرش.

برای مسیرهای دریافت از `openclaw/plugin-sdk/channel-ingress-runtime` استفاده کنید.

## تفکیک‌کننده زمان اجرا

```ts
import {
  defineStableChannelIngressIdentity,
  resolveChannelMessageIngress,
} from "openclaw/plugin-sdk/channel-ingress-runtime";

const identity = defineStableChannelIngressIdentity({
  key: "platform-user-id",
  normalize: normalizePlatformUserId,
  sensitivity: "pii",
});

const result = await resolveChannelMessageIngress({
  channelId: "my-channel",
  accountId,
  identity,
  subject: { stableId: platformUserId },
  conversation: { kind: isGroup ? "group" : "direct", id: conversationId },
  event: { kind: "message", authMode: "inbound", mayPair: !isGroup },
  policy: {
    dmPolicy: config.dmPolicy,
    groupPolicy: config.groupPolicy,
    groupAllowFromFallbackToAllowFrom: true,
  },
  allowFrom: config.allowFrom,
  groupAllowFrom: config.groupAllowFrom,
  accessGroups: cfg.accessGroups,
  route,
  readStoreAllowFrom,
  command: hasControlCommand ? { allowTextCommands: true, hasControlCommand } : undefined,
});
```

فهرست‌های مجاز مؤثر، مالکان فرمان یا گروه‌های فرمان را از پیش محاسبه نکنید.
تفکیک‌کننده آن‌ها را از فهرست‌های مجاز خام، فراخوان‌های ذخیره‌گاه، توصیف‌گرهای
مسیر، گروه‌های دسترسی، سیاست و نوع مکالمه استخراج می‌کند.

## نتیجه

Pluginهای همراه باید تصویرسازی‌های مدرن را مستقیماً مصرف کنند:

| فیلد              | معنا                                                            |
| ------------------ | ------------------------------------------------------------------ |
| `ingress`          | تصمیم مرتب‌شده دروازه‌ها و پذیرش                                |
| `senderAccess`     | فقط مجوز فرستنده/مکالمه                             |
| `routeAccess`      | تصویرسازی مسیر و فرستنده مسیر                                  |
| `commandAccess`    | مجوز فرمان؛ `requested: false` وقتی هیچ دروازه فرمانی اجرا نشده باشد |
| `activationAccess` | نتیجه اشاره/فعال‌سازی                                          |

مجوز رویداد در `ingress.graph` مرتب‌شده و
`ingress.reasonCode` تعیین‌کننده در دسترس می‌ماند؛ تصویرسازی جداگانه‌ای برای رویداد تولید نمی‌شود.

کمک‌تابع‌های منسوخ SDK شخص ثالث می‌توانند شکل‌های قدیمی‌تر را به‌صورت داخلی بازسازی کنند. مسیرهای
دریافت همراه جدید نباید نتایج مدرن را دوباره به DTOهای محلی
تبدیل کنند.

## گروه‌های دسترسی

ورودی‌های `accessGroup:<name>` پوشیده‌شده باقی می‌مانند. هسته گروه‌های ایستای
`message.senders` را خودش تفکیک می‌کند و `resolveAccessGroupMembership` را فقط
برای گروه‌های پویایی فراخوانی می‌کند که به جست‌وجوی پلتفرم نیاز دارند. گروه‌های مفقود، پشتیبانی‌نشده و
ناموفق به‌صورت بسته رد می‌شوند.

## حالت‌های رویداد

| `authMode`       | معنا                                          |
| ---------------- | ------------------------------------------------ |
| `inbound`        | دروازه‌های عادی فرستنده ورودی                      |
| `command`        | دروازه‌های فرمان برای فراخوان‌های برگشتی یا دکمه‌های محدوده‌دار    |
| `origin-subject` | عامل باید با موضوع پیام اصلی مطابقت داشته باشد    |
| `route-only`     | فقط دروازه‌های مسیر برای رویدادهای قابل‌اعتماد محدود به مسیر |
| `none`           | رویدادهای داخلی تحت مالکیت Plugin از احراز هویت مشترک عبور می‌کنند  |

برای واکنش‌ها، دکمه‌ها، فراخوان‌های برگشتی و فرمان‌های بومی از `mayPair: false` استفاده کنید.

## مسیرها و فعال‌سازی

برای سیاست اتاق، موضوع، انجمن، رشته یا مسیر تودرتو از توصیف‌گرهای مسیر استفاده کنید:

```ts
route: {
  id: "room",
  allowed: roomAllowed,
  enabled: roomEnabled,
  senderPolicy: "replace",
  senderAllowFrom: roomAllowFrom,
  blockReason: "room_sender_not_allowlisted",
}
```

وقتی یک Plugin چند توصیف‌گر مسیر اختیاری دارد، از `channelIngressRoutes(...)`
استفاده کنید؛ این مورد ضمن عمومی نگه‌داشتن واقعیت‌های مسیر و حفظ ترتیب آن‌ها بر اساس
`precedence` هر توصیف‌گر، شاخه‌های غیرفعال را فیلتر می‌کند.

دروازه‌بندی اشاره، یک دروازه فعال‌سازی است. نبود اشاره
`admission: "skip"` را برمی‌گرداند تا هسته نوبت، نوبتی صرفاً نظارتی را پردازش نکند.
بیشتر کانال‌ها باید فعال‌سازی را پس از دروازه‌های فرستنده و فرمان نگه دارند. سطوح
گفت‌وگوی عمومی که باید ترافیک بدون اشاره را پیش از ایجاد نویز فهرست مجاز فرستنده
ساکت کنند، در صورت غیرفعال بودن عبور فرمان متنی می‌توانند `activation.order: "before-sender"`
را انتخاب کنند. کانال‌های دارای فعال‌سازی ضمنی، مانند پاسخ‌ها در رشته‌های
بات، `channels.defaults.implicitMentions` را همراه با بازنویسی‌های کانال و حساب
با `resolveChannelImplicitMentions(...)` تفکیک می‌کنند و سپس نتیجه را به‌عنوان
`activation.implicitMentions` ارسال می‌کنند. تصویرسازی
`activationAccess.shouldBypassMention` گزارش می‌دهد که چه زمانی فرمان یا فعال‌سازی
ضمنی، الزام اشاره صریح را دور زده است.

## پوشاندن داده‌ها

مقادیر خام فرستنده و ورودی‌های خام فهرست مجاز فقط ورودی تفکیک‌کننده هستند. آن‌ها
نباید در وضعیت تفکیک‌شده، تصمیم‌ها، عیب‌یابی‌ها، عکس‌های فوری یا
واقعیت‌های سازگاری ظاهر شوند. از شناسه‌های مبهم موضوع، ورودی، مسیر و
عیب‌یابی استفاده کنید.

## راستی‌آزمایی

```bash
pnpm test src/channels/message-access/message-access.test.ts src/plugin-sdk/channel-ingress-runtime.test.ts
pnpm plugin-sdk:api:check
```
