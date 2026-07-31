---
read_when:
    - بازآرایی رابط کاربری پیام کانال، محموله‌های تعاملی یا رندرکننده‌های بومی کانال
    - تغییر قابلیت‌های ابزار پیام، راهنماهای تحویل یا نشانگرهای میان‌بافتی
    - اشکال‌زدایی از توزیع گسترده import مربوط به Discord Carbon یا بارگذاری تنبل زمان اجرای Plugin کانال
summary: ارائهٔ معنایی پیام را از رندرکننده‌های رابط کاربری بومی کانال مستقل کنید.
title: برنامه بازآرایی نحوه ارائه کانال
x-i18n:
    generated_at: "2026-07-27T15:30:42Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b0f0c4f64e0c503209ac0a5b763b1b5483bf8d55a28ceacffbbcd1337d4371e
    source_path: plan/ui-channels.md
    workflow: 16
---

## وضعیت

برای عامل مشترک، CLI، قابلیت Plugin و سطوح تحویل خروجی پیاده‌سازی شده است:

- `ReplyPayload.presentation` رابط کاربری معنایی پیام را حمل می‌کند.
- `ReplyPayload.delivery.pin` درخواست‌های سنجاق‌کردن پیام ارسال‌شده را حمل می‌کند.
- کنش‌های مشترک پیام، به‌جای `components`،‏ `blocks`،‏ `buttons` یا `card` بومی ارائه‌دهنده، `presentation`،‏ `delivery` و `pin` را ارائه می‌کنند.
- هسته، ارائه را از طریق قابلیت‌های خروجی اعلام‌شده توسط Plugin رندر می‌کند یا به‌طور خودکار تنزل می‌دهد.
- رندرکننده‌های Discord، Slack، Telegram، Mattermost، MS Teams و Feishu قرارداد عمومی را مصرف می‌کنند.
- کد صفحه کنترل کانال Discord دیگر کانتینرهای رابط کاربری مبتنی بر Carbon را وارد نمی‌کند.

مستندات مرجع اکنون در [ارائه پیام](/fa/plugins/message-presentation) قرار دارند.
این طرح را به‌عنوان زمینه تاریخی پیاده‌سازی نگه دارید؛ برای تغییرات قرارداد،
رندرکننده یا رفتار جایگزین، راهنمای مرجع را به‌روزرسانی کنید.

## مشکل

رابط کاربری کانال در حال حاضر میان چند سطح ناسازگار تقسیم شده است:

- هسته از طریق `buildCrossContextComponents` مالک یک هوک رندر میان‌زمینه‌ای با ساختار Discord است.
- `channel.ts` در Discord می‌تواند رابط کاربری بومی Carbon را از طریق `DiscordUiContainer` وارد کند که وابستگی‌های زمان اجرای رابط کاربری را به صفحه کنترل Plugin کانال می‌کشاند.
- عامل و CLI راه‌های گریز محموله بومی مانند `components` در Discord،‏ `blocks` در Slack،‏ `buttons` در Telegram یا Mattermost و `card` در Teams یا Feishu را ارائه می‌کنند.
- `ReplyPayload.channelData` هم راهنمایی‌های انتقال و هم پاکت‌های رابط کاربری بومی را حمل می‌کند.
- مدل عمومی `interactive` وجود دارد، اما از چیدمان‌های غنی‌تری که هم‌اکنون در Discord، Slack، Teams، Feishu، LINE، Telegram و Mattermost استفاده می‌شوند محدودتر است.

این وضعیت هسته را از ساختارهای رابط کاربری بومی آگاه می‌کند، تنبلی زمان اجرای Plugin را تضعیف می‌کند و راه‌های مختص ارائه‌دهنده بیش‌ازحدی برای بیان یک منظور پیام یکسان در اختیار عامل‌ها می‌گذارد.

## اهداف

- هسته بر اساس قابلیت‌های اعلام‌شده، بهترین ارائه معنایی را برای پیام تعیین می‌کند.
- افزونه‌ها قابلیت‌ها را اعلام می‌کنند و ارائه معنایی را به محموله‌های انتقال بومی رندر می‌کنند.
- رابط کاربری کنترل وب از رابط کاربری بومی گفت‌وگو جدا باقی می‌ماند.
- محموله‌های بومی کانال از طریق سطح پیام مشترک عامل یا CLI ارائه نمی‌شوند.
- ویژگی‌های ارائه پشتیبانی‌نشده به‌طور خودکار به بهترین نمایش متنی تنزل می‌یابند.
- رفتار تحویل، مانند سنجاق‌کردن پیام ارسال‌شده، فراداده عمومی تحویل است، نه ارائه.

## اهداف خارج از محدوده

- هیچ شیم سازگاری با گذشته برای `buildCrossContextComponents` وجود ندارد.
- هیچ راه گریز بومی عمومی برای `components`،‏ `blocks`،‏ `buttons` یا `card` وجود ندارد.
- هسته هیچ کتابخانه رابط کاربری بومی کانال را وارد نمی‌کند.
- هیچ درز SDK مختص ارائه‌دهنده‌ای برای کانال‌های همراه وجود ندارد.

## مدل هدف

یک فیلد `presentation` متعلق به هسته به `ReplyPayload` اضافه کنید.

```ts
type MessagePresentationTone = "neutral" | "info" | "success" | "warning" | "danger";

type MessagePresentation = {
  tone?: MessagePresentationTone;
  title?: string;
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] };

type MessagePresentationButton = {
  label: string;
  value?: string;
  url?: string;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  value: string;
};
```

`interactive` هنگام مهاجرت به زیرمجموعه‌ای از `presentation` تبدیل می‌شود:

- بلوک متنی `interactive` به `presentation.blocks[].type = "text"` نگاشت می‌شود.
- بلوک دکمه‌های `interactive` به `presentation.blocks[].type = "buttons"` نگاشت می‌شود.
- بلوک انتخاب `interactive` به `presentation.blocks[].type = "select"` نگاشت می‌شود.

طرح‌واره‌های عامل خارجی و CLI اکنون از `presentation` استفاده می‌کنند؛ `interactive` به‌عنوان یک ابزار کمکی داخلی و قدیمی برای تجزیه/رندر تولیدکنندگان پاسخ موجود باقی می‌ماند.
API عمومی روبه‌تولیدکننده، `interactive` را منسوخ‌شده تلقی می‌کند. پشتیبانی زمان اجرا
باقی می‌ماند تا ابزارهای کمکی تأیید موجود و Plugin‌های قدیمی‌تر همچنان
کار کنند، درحالی‌که کد جدید `presentation` را منتشر می‌کند.

## فراداده تحویل

یک فیلد `delivery` متعلق به هسته برای رفتار ارسالی که رابط کاربری نیست اضافه کنید.

```ts
type ReplyPayloadDelivery = {
  pin?:
    | boolean
    | {
        enabled: boolean;
        notify?: boolean;
        required?: boolean;
      };
};
```

معناشناسی:

- `delivery.pin = true` یعنی نخستین پیام با تحویل موفق سنجاق شود.
- مقدار پیش‌فرض `notify` برابر `false` است.
- مقدار پیش‌فرض `required` برابر `false` است؛ کانال‌های پشتیبانی‌نشده یا شکست در سنجاق‌کردن، با ادامه تحویل به‌طور خودکار تنزل می‌یابند.
- کنش‌های دستی پیام `pin`،‏ `unpin` و `list-pins` برای پیام‌های موجود باقی می‌مانند.

اتصال کنونی موضوع ACP در Telegram باید از `channelData.telegram.pin = true` به `delivery.pin = true` منتقل شود.

## قرارداد قابلیت زمان اجرا

هوک‌های رندر ارائه و تحویل را به آداپتور خروجی زمان اجرا اضافه کنید، نه به Plugin کانال صفحه کنترل.

```ts
type ChannelPresentationCapabilities = {
  supported: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  tones?: MessagePresentationTone[];
  limits?: {
    actions?: {
      maxActions?: number;
      maxActionsPerRow?: number;
      maxRows?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
      supportsStyles?: boolean;
      supportsDisabled?: boolean;
      supportsLayoutHints?: boolean;
    };
    selects?: {
      maxOptions?: number;
      maxLabelLength?: number;
      maxValueBytes?: number;
    };
    text?: {
      maxLength?: number;
      encoding?: "characters" | "utf8-bytes" | "utf16-units";
      markdownDialect?: "plain" | "markdown" | "html" | "slack-mrkdwn" | "discord-markdown";
      supportsEdit?: boolean;
    };
  };
};

type ChannelDeliveryCapabilities = {
  pinSentMessage?: boolean;
};

type ChannelOutboundAdapter = {
  presentationCapabilities?: ChannelPresentationCapabilities;

  renderPresentation?: (params: {
    payload: ReplyPayload;
    presentation: MessagePresentation;
    ctx: ChannelOutboundSendContext;
  }) => ReplyPayload | null;

  deliveryCapabilities?: ChannelDeliveryCapabilities;

  pinDeliveredMessage?: (params: {
    cfg: OpenClawConfig;
    accountId?: string | null;
    to: string;
    threadId?: string | number | null;
    messageId: string;
    notify: boolean;
  }) => Promise<void>;
};
```

رفتار هسته:

- کانال هدف و آداپتور زمان اجرا را تفکیک کنید.
- قابلیت‌های ارائه را درخواست کنید.
- پیش از رندر، بلوک‌های پشتیبانی‌نشده را تنزل دهید و محدودیت‌های عمومی قابلیت را
  اعمال کنید.
- `renderPresentation` را فراخوانی کنید.
- اگر رندرکننده‌ای وجود ندارد، ارائه را به متن جایگزین تبدیل کنید.
- پس از ارسال موفق، هنگامی که `delivery.pin` درخواست شده و پشتیبانی می‌شود، `pinDeliveredMessage` را فراخوانی کنید.

## نگاشت کانال

Discord:

- `presentation` را در ماژول‌های صرفاً زمان اجرا به مؤلفه‌های v2 و کانتینرهای Carbon رندر کنید.
- ابزارهای کمکی رنگ تأکیدی را در ماژول‌های سبک نگه دارید.
- واردکردن‌های `DiscordUiContainer` را از کد صفحه کنترل Plugin کانال حذف کنید.

Slack:

- `presentation` را به Block Kit رندر کنید.
- ورودی `blocks` عامل و CLI را حذف کنید.

Telegram:

- متن، زمینه و جداکننده‌ها را به‌شکل متن رندر کنید.
- هنگامی که برای سطح هدف پیکربندی و مجاز است، کنش‌ها و انتخاب را به‌صورت صفحه‌کلیدهای درون‌خطی رندر کنید.
- هنگامی که دکمه‌های درون‌خطی غیرفعال‌اند، از متن جایگزین استفاده کنید.
- سنجاق‌کردن موضوع ACP را به `delivery.pin` منتقل کنید.

Mattermost:

- در صورت پیکربندی، کنش‌ها را به‌شکل دکمه‌های تعاملی رندر کنید.
- بلوک‌های دیگر را به‌شکل متن جایگزین رندر کنید.

MS Teams:

- `presentation` را به Adaptive Cards رندر کنید.
- کنش‌های دستی سنجاق‌کردن/برداشتن سنجاق/فهرست‌کردن سنجاق‌ها را نگه دارید.
- اگر پشتیبانی Graph برای گفت‌وگوی هدف قابل‌اعتماد است، `pinDeliveredMessage` را به‌صورت اختیاری پیاده‌سازی کنید.

Feishu:

- `presentation` را به کارت‌های تعاملی رندر کنید.
- کنش‌های دستی سنجاق‌کردن/برداشتن سنجاق/فهرست‌کردن سنجاق‌ها را نگه دارید.
- اگر رفتار API قابل‌اعتماد است، `pinDeliveredMessage` را برای سنجاق‌کردن پیام ارسال‌شده به‌صورت اختیاری پیاده‌سازی کنید.

LINE:

- `presentation` را در صورت امکان به پیام‌های Flex یا الگو رندر کنید.
- برای بلوک‌های پشتیبانی‌نشده به متن بازگردید.
- محموله‌های رابط کاربری LINE را از `channelData` حذف کنید.

کانال‌های ساده یا محدود:

- ارائه را با قالب‌بندی محافظه‌کارانه به متن تبدیل کنید.

## مراحل بازآرایی

1. اصلاح انتشار Discord را که `ui-colors.ts` را از رابط کاربری مبتنی بر Carbon جدا می‌کند و `DiscordUiContainer` را از `extensions/discord/src/channel.ts` حذف می‌کند، دوباره اعمال کنید.
2. `presentation` و `delivery` را به `ReplyPayload`، عادی‌سازی محموله خروجی، خلاصه‌های تحویل و محموله‌های هوک اضافه کنید.
3. طرح‌واره `MessagePresentation` و ابزارهای کمکی تجزیه را در یک زیرمسیر محدود SDK/زمان اجرا اضافه کنید.
4. قابلیت‌های پیام `buttons`،‏ `cards`،‏ `components` و `blocks` را با قابلیت‌های ارائه معنایی جایگزین کنید.
5. هوک‌های آداپتور خروجی زمان اجرا را برای رندر ارائه و سنجاق‌کردن تحویل اضافه کنید.
6. ساخت مؤلفه میان‌زمینه‌ای را با `buildCrossContextPresentation` جایگزین کنید.
7. `src/infra/outbound/channel-adapters.ts` را حذف کنید و `buildCrossContextComponents` را از نوع‌های Plugin کانال بردارید.
8. `maybeApplyCrossContextMarker` را تغییر دهید تا به‌جای پارامترهای بومی، `presentation` را پیوست کند.
9. مسیرهای ارسال توزیع Plugin را به‌روزرسانی کنید تا فقط ارائه معنایی و فراداده تحویل را مصرف کنند.
10. پارامترهای محموله بومی عامل و CLI را حذف کنید: `components`،‏ `blocks`،‏ `buttons` و `card`.
11. ابزارهای کمکی SDK را که طرح‌واره‌های ابزار پیام بومی ایجاد می‌کنند حذف کنید و آن‌ها را با ابزارهای کمکی طرح‌واره ارائه جایگزین کنید.
12. پاکت‌های رابط کاربری/بومی را از `channelData` حذف کنید؛ تا زمان بازبینی هر فیلد باقی‌مانده، فقط فراداده انتقال را نگه دارید.
13. رندرکننده‌های Discord، Slack، Telegram، Mattermost، MS Teams، Feishu و LINE را مهاجرت دهید.
14. مستندات CLI پیام، صفحات کانال، SDK ‏Plugin و کتاب آشپزی قابلیت‌ها را به‌روزرسانی کنید.
15. برای Discord و نقاط ورود کانال‌های متأثر، پروفایل‌گیری گسترش واردکردن را اجرا کنید.

مراحل 1-11 و 13-14 در این بازآرایی برای قراردادهای عامل مشترک، CLI، قابلیت Plugin و آداپتور خروجی پیاده‌سازی شده‌اند. مرحله 12 به‌عنوان یک دور پاک‌سازی داخلی عمیق‌تر برای پاکت‌های انتقال خصوصی ارائه‌دهنده `channelData` باقی می‌ماند. مرحله 15 نیز در صورتی که اعداد کمّی گسترش واردکردن فراتر از دروازه نوع/آزمون لازم باشند، برای اعتبارسنجی بعدی باقی می‌ماند.

## آزمون‌ها

موارد زیر را اضافه یا به‌روزرسانی کنید:

- آزمون‌های عادی‌سازی ارائه.
- آزمون‌های تنزل خودکار ارائه برای بلوک‌های پشتیبانی‌نشده.
- آزمون‌های نشانگر میان‌زمینه‌ای برای توزیع Plugin و مسیرهای تحویل هسته.
- آزمون‌های ماتریس رندر کانال برای Discord، Slack، Telegram، Mattermost، MS Teams، Feishu، LINE و متن جایگزین.
- آزمون‌های طرح‌واره ابزار پیام که نبودن فیلدهای بومی را اثبات می‌کنند.
- آزمون‌های CLI که نبودن پرچم‌های بومی را اثبات می‌کنند.
- رگرسیون تنبلی واردکردن نقطه ورود Discord که Carbon را پوشش می‌دهد.
- آزمون‌های سنجاق تحویل که Telegram و جایگزین عمومی را پوشش می‌دهند.

## پرسش‌های باز

- آیا `delivery.pin` باید در مرحلهٔ نخست برای Discord، Slack، MS Teams و Feishu پیاده‌سازی شود یا ابتدا فقط برای Telegram؟
- آیا `delivery` در نهایت باید فیلدهای موجودی مانند `replyToId`، `replyToCurrent`، `silent` و `audioAsVoice` را نیز دربر بگیرد یا همچنان بر رفتارهای پس از ارسال متمرکز بماند؟
- آیا بخش ارائه باید مستقیماً از تصاویر یا ارجاع‌های فایل پشتیبانی کند یا فعلاً رسانه از چیدمان رابط کاربری جدا بماند؟

## مرتبط

- [نمای کلی کانال‌ها](/fa/channels)
- [ارائهٔ پیام](/fa/plugins/message-presentation)
