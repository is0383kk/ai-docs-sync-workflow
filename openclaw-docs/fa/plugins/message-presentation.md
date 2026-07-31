---
read_when:
    - افزودن یا تغییر رندر کارت پیام، نمودار، جدول، دکمه یا انتخاب‌گر
    - ساخت Plugin کانال با پشتیبانی از پیام‌های خروجی غنی
    - تغییر نحوه نمایش ابزار پیام یا قابلیت‌های تحویل آن
    - اشکال‌زدایی پس‌رفت‌های رندر کارت/بلوک/کامپوننت مختص ارائه‌دهنده
summary: کارت‌های معنایی پیام، نمودارها، جدول‌ها، کنترل‌ها، متن جایگزین و راهنمایی‌های تحویل برای Pluginهای کانال
title: ارائهٔ پیام
x-i18n:
    generated_at: "2026-07-27T15:53:32Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fce3874c99627eb87ceb83aebe381b8a8466722703ec6322c609f187d15d9ae
    source_path: plugins/message-presentation.md
    workflow: 16
---

ارائهٔ پیام، قرارداد مشترک OpenClaw برای رابط کاربری غنیِ چت خروجی است.
این قرارداد به عامل‌ها، فرمان‌های CLI، جریان‌های تأیید و Pluginها اجازه می‌دهد قصد پیام را
یک‌بار توصیف کنند، درحالی‌که هر Plugin کانال، بهترین قالب بومی ممکن را رندر می‌کند.

برای رابط کاربری قابل‌حمل پیام از ارائه استفاده کنید: بخش‌های متنی، متن کوتاه زمینه/پاورقی،
جداکننده‌ها، نمودارها، جدول‌ها، دکمه‌ها، منوهای انتخاب و عنوان/لحن کارت.

فیلدهای بومیِ ارائه‌دهندهٔ جدید مانند Discord `components`، Slack
`blocks`، Telegram `buttons`، Teams `card` یا Feishu `card` را به ابزار مشترک
پیام اضافه نکنید. این‌ها خروجی‌های رندرکننده‌اند که مالکیتشان با Plugin کانال است.

## قرارداد

نویسندگان Plugin قرارداد عمومی را از این مسیر وارد می‌کنند:

```ts
import type {
  MessagePresentation,
  ReplyPayloadDelivery,
} from "openclaw/plugin-sdk/interactive-runtime";
```

ساختار:

```ts
type MessagePresentation = {
  title?: string;
  tone?: "neutral" | "info" | "success" | "warning" | "danger";
  blocks: MessagePresentationBlock[];
};

type MessagePresentationBlock =
  | { type: "text"; text: string }
  | { type: "context"; text: string }
  | { type: "divider" }
  | { type: "buttons"; buttons: MessagePresentationButton[] }
  | { type: "select"; placeholder?: string; options: MessagePresentationOption[] }
  | {
      type: "chart";
      chartType: "pie";
      title: string;
      segments: Array<{ label: string; value: number }>;
    }
  | {
      type: "chart";
      chartType: "bar" | "area" | "line";
      title: string;
      categories: string[];
      series: Array<{ name: string; values: number[] }>;
      xLabel?: string;
      yLabel?: string;
    }
  | {
      type: "table";
      caption: string;
      headers: string[];
      rows: Array<Array<string | number>>;
      rowHeaderColumnIndex?: number;
    };

type MessagePresentationAction =
  | { type: "command"; command: string }
  | { type: "callback"; value: string }
  | {
      type: "approval";
      approvalId: string;
      approvalKind: "exec" | "plugin";
      decision: "allow-once" | "allow-always" | "deny";
    }
  | {
      type: "question";
      questionId: string;
      optionValue: string;
    }
  | { type: "url"; url: string }
  | {
      type: "web-app";
      url: string;
      widgetId?: string;
    }
  | {
      type: "web-app";
      url?: string;
      widgetId: string;
    };

type MessagePresentationButton = {
  label: string;
  action?: MessagePresentationAction;
  /** مقدار callback قدیمی. برای کنترل‌های جدید action را ترجیح دهید. */
  value?: string;
  /** @deprecated از action با type برابر "url" استفاده کنید. */
  url?: string;
  /** @deprecated از action با type برابر "web-app" استفاده کنید. */
  webApp?: { url: string };
  /** @deprecated از action با type برابر "web-app" استفاده کنید. */
  web_app?: { url: string };
  priority?: number;
  disabled?: boolean;
  reusable?: boolean;
  style?: "primary" | "secondary" | "success" | "danger";
};

type MessagePresentationOption = {
  label: string;
  action?: Extract<MessagePresentationAction, { type: "command" | "callback" }>;
  /** مقدار callback قدیمی. برای کنترل‌های جدید action را ترجیح دهید. */
  value?: string;
};

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

معنای دکمه‌ها:

- `action.type: "command"` یک فرمان slash بومی را از طریق مسیر فرمان هسته
  اجرا می‌کند. از این مورد برای دکمه‌ها و منوهای فرمان داخلی استفاده کنید.
- `action.type: "callback"` دادهٔ مات Plugin را از طریق مسیر تعامل کانال
  حمل می‌کند. Pluginهای کانال نباید دادهٔ callback را دوباره به‌عنوان فرمان‌های slash
  تفسیر کنند.
- `action.type: "approval"` یک تأیید پایدار اپراتور، نوع صریح
  `exec` یا `plugin` آن و تصمیم درخواستی را مشخص می‌کند. Pluginهای کانال
  این اقدام را در یک callback خصوصیِ انتقال کدگذاری می‌کنند و آن را از طریق
  سرویس تأیید حل می‌کنند؛ آن‌ها نباید متن فرمان `/approve` را تجزیه کنند یا
  نوع را از شناسه استنباط کنند.
- `action.type: "question"` یک گزینه را برای پرسش زندهٔ `ask_user` که در زمان اجرا ایجاد شده است،
  مشخص می‌کند. همانند `approval`، این یک اقدام زمان اجرای OpenClaw است؛
  عامل‌ها و Pluginها نباید شناسه‌های پرسش را تولید کنند. Telegram، Discord و
  Slack آن را به callbackهای بومی و خصوصیِ انتقال نگاشت می‌کنند و گزینه را
  از طریق Gateway حل می‌کنند. وقتی پرسش پاسخ داده شود، منقضی شود یا
  لغو شود، این کانال‌ها پیام تحویل‌شده را ویرایش می‌کنند، اقدام‌های آن را حذف می‌کنند
  و وضعیت پایانی را می‌افزایند. WhatsApp، Signal و iMessage حداکثر
  چهار گزینهٔ تک‌انتخابی را به‌صورت واکنش‌های `1️⃣` تا `4️⃣` رندر می‌کنند. دیگر ساختارهای
  پرسش به متن برچسب تنزل می‌یابند و کاربر می‌تواند با یک پاسخ
  متنی ساده جواب دهد.
- `action.type: "url"` یک پیوند معمولی را باز می‌کند.
- `action.type: "web-app"` یک برنامهٔ وب بومیِ کانال را اجرا می‌کند. برای یک
  برنامهٔ مبتنی بر URL، `url` را تنظیم کنید یا برای ویجتی که OpenClaw میزبانی می‌کند و سازوکار
  اجرای آن در مالکیت کانال است، `widgetId` را تنظیم کنید؛ حداقل یکی الزامی است. وقتی هر دو
  وجود دارند، کانال می‌تواند اجرای بومیِ ویجت میزبانی‌شدهٔ خود را ترجیح دهد و در جایی
  که آن سازوکار در دسترس نیست از URL استفاده کند.
- `value` مقدار مات callback قدیمی است. کنترل‌های جدید باید از `action`
  استفاده کنند تا Pluginهای کانال بتوانند بدون حدس‌زدن از روی متن، فرمان‌ها و callbackها را نگاشت کنند.
- `url`، `webApp` و `web_app` همچنان به‌عنوان ورودی‌های منسوخ‌شدهٔ مرزی پذیرفته می‌شوند.
  نرمال‌سازها این فیلدها را حفظ می‌کنند تا رندرکننده‌ها بتوانند معنای قدیمیِ
  منتشرشده را از اقدام‌های صریحِ نوع‌دار تشخیص دهند. تولیدکنندگان جدید باید از `action` استفاده کنند.
- `label` الزامی است و در جایگزین متنی نیز استفاده می‌شود.
- `style` جنبهٔ توصیه‌ای دارد. رندرکننده‌ها باید سبک‌های پشتیبانی‌نشده را به یک
  پیش‌فرض امن نگاشت کنند، نه اینکه ارسال را ناموفق کنند.
- `priority` اختیاری است. وقتی کانالی محدودیت‌های اقدام را اعلام می‌کند و لازم است
  کنترل‌ها حذف شوند، هسته ابتدا دکمه‌های دارای اولویت بالاتر را نگه می‌دارد و
  ترتیب اصلی را میان دکمه‌های هم‌اولویت حفظ می‌کند. وقتی همهٔ کنترل‌ها جا شوند،
  ترتیب تألیف‌شده حفظ می‌شود.
- `disabled` اختیاری است. کانال‌ها باید با `supportsDisabled` صراحتاً آن را فعال کنند؛ در غیر این صورت
  هسته کنترل غیرفعال را به متن جایگزین غیرتعاملی تنزل می‌دهد. یک
  دکمهٔ غیرفعال همیشه در متن جایگزین فقط با برچسب رندر می‌شود، حتی وقتی
  یک اقدام `command` داشته باشد.
- `reusable` اختیاری است. کانال‌هایی که از callbackهای بومیِ قابل‌استفادهٔ مجدد پشتیبانی می‌کنند، می‌توانند
  اقدام را پس از یک تعامل موفق همچنان در دسترس نگه دارند. از آن برای
  اقدام‌های تکرارپذیر یا ایدمپوتنت مانند تازه‌سازی، بازرسی یا جزئیات بیشتر استفاده کنید؛
  برای تأییدهای معمولیِ یک‌باره و اقدام‌های مخرب، آن را تنظیم‌نشده باقی بگذارید.

معنای انتخاب:

- `options[].action` فقط `command` یا `callback` را می‌پذیرد؛ اقدام‌های تأیید و پیوند فقط مخصوص دکمه هستند.
- `options[].value` مقدار برنامهٔ انتخاب‌شدهٔ قدیمی است.
- `placeholder` جنبهٔ توصیه‌ای دارد و ممکن است کانال‌هایی که پشتیبانی بومی از
  انتخاب ندارند، آن را نادیده بگیرند.
- اگر کانالی از انتخاب‌ها پشتیبانی نکند، متن جایگزین برچسب‌ها را فهرست می‌کند.

معنای نمودار:

- `pie` به مقادیر مثبت برای بخش‌ها نیاز دارد.
- `bar`، `area` و `line` از یک آرایهٔ مرتب `categories` استفاده می‌کنند. هر سری
  دقیقاً یک مقدار متناهی برای هر دسته، با همان ترتیب، فراهم می‌کند.
- برچسب‌های دسته و نام‌های سری باید یکتا باشند. بلوک‌های نمودار نامعتبر یا ناقص
  هنگام نرمال‌سازی حذف می‌شوند، نه اینکه داده‌ها بی‌سروصدا تغییر کنند.
- رندر بومی نمودار از طریق `presentationCapabilities.charts` به‌صورت صریح فعال می‌شود.
  دیگر کانال‌ها عنوان نمودار، محورها، دسته‌ها، سری‌ها و مقادیر را
  به‌صورت متن قطعی دریافت می‌کنند. این همچنین جایگزین دسترس‌پذیری است.

معنای جدول:

- `caption` یک عنوان کوتاه الزامی است. `headers` باید حداقل یک
  برچسب ستون یکتا و غیرخالی داشته باشد.
- `rows` باید حداقل یک ردیف داشته باشد. هر ردیف باید دقیقاً برای هر
  سرستون یک سلول داشته باشد و هر سلول باید رشته‌ای غیرخالی یا عددی متناهی باشد.
- `rowHeaderColumnIndex` یک شاخص اختیاری با مبدأ صفر است که ستونی را مشخص می‌کند
  که سلول‌های آن باید توسط رندرکننده‌های بومی به‌عنوان سرستون ردیف ارائه شوند.
- نرمال‌سازی جدول اتمی است. عنوان، سرستون، عرض ردیف، سلول
  یا شاخص سرستون ردیف نامعتبر باعث حذف بلوک جدول می‌شود، نه کوتاه‌سازی یا ترمیم
  داده‌های آن.
- رندر بومی جدول از طریق `presentationCapabilities.tables` به‌صورت صریح فعال می‌شود.
  دیگر کانال‌ها عنوان و همهٔ ردیف‌ها را به‌صورت متن خطی قطعی
  دریافت می‌کنند و فاصله‌های خالی داخلی ادغام می‌شوند:

  ```text
  پایپ‌لاین باز (جدول)
  - حساب: Acme؛ مرحله: برنده؛ ARR: 125000
  - حساب: Globex؛ مرحله: بازبینی؛ ARR: 82000
  ```

هیچ متمایزکنندهٔ جداگانهٔ `report` وجود ندارد. یک گزارش را از `title`،
`tone`، `text`، `context`، `chart`، `table` و بلوک‌های اقدام ترکیب کنید. این کار هر
بلوک را مستقل از دیگران قابل‌رندر نگه می‌دارد و همان جایگزین متنی
قطعی را برای گزارش کامل فراهم می‌کند.

## نمونه‌های تولیدکننده

کارت ساده:

```json
{
  "title": "تأیید استقرار",
  "tone": "warning",
  "blocks": [
    { "type": "text", "text": "Canary آمادهٔ ارتقا است." },
    { "type": "context", "text": "ساخت 1234، مرحلهٔ staging با موفقیت گذشت." },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "تأیید",
          "action": { "type": "callback", "value": "deploy:approve" },
          "style": "success"
        },
        {
          "label": "رد",
          "action": { "type": "callback", "value": "deploy:decline" },
          "style": "danger"
        }
      ]
    }
  ]
}
```

دکمهٔ پیوند فقط-URL:

```json
{
  "blocks": [
    { "type": "text", "text": "یادداشت‌های انتشار آماده‌اند." },
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "بازکردن یادداشت‌ها",
          "action": { "type": "url", "url": "https://example.com/release" }
        }
      ]
    }
  ]
}
```

دکمهٔ برنامهٔ کوچک Telegram:

```json
{
  "blocks": [
    {
      "type": "buttons",
      "buttons": [
        {
          "label": "اجرا",
          "action": { "type": "web-app", "url": "https://example.com/app" }
        }
      ]
    }
  ]
}
```

منوی انتخاب:

```json
{
  "title": "انتخاب محیط",
  "blocks": [
    {
      "type": "select",
      "placeholder": "محیط",
      "options": [
        { "label": "Canary", "value": "env:canary" },
        { "label": "محیط تولید", "value": "env:prod" }
      ]
    }
  ]
}
```

نمودار:

```json
{
  "blocks": [
    {
      "type": "chart",
      "chartType": "line",
      "title": "درآمد فصلی",
      "categories": ["Q1", "Q2", "Q3"],
      "series": [
        { "name": "محصول", "values": [120, 145, 138] },
        { "name": "خدمات", "values": [80, 95, 104] }
      ],
      "xLabel": "فصل",
      "yLabel": "درآمد"
    }
  ]
}
```

گزارش جدولی:

```json
{
  "title": "گزارش پایپ‌لاین",
  "tone": "info",
  "blocks": [
    { "type": "text", "text": "فرصت‌های فعلی بر اساس مرحله." },
    {
      "type": "table",
      "caption": "پایپ‌لاین باز",
      "headers": ["حساب", "مرحله", "ARR"],
      "rows": [
        ["Acme", "برنده", 125000],
        ["Globex", "بازبینی", 82000]
      ],
      "rowHeaderColumnIndex": 0
    },
    { "type": "context", "text": "از اسنپ‌شات CRM به‌روزرسانی شده است." }
  ]
}
```

ارسال با CLI:

```bash
openclaw message send --channel slack \
  --target channel:C123 \
  --message "تأیید استقرار" \
  --presentation '{"title":"تأیید استقرار","tone":"warning","blocks":[{"type":"text","text":"Canary آماده است."},{"type":"buttons","buttons":[{"label":"تأیید","value":"deploy:approve","style":"success"},{"label":"رد","value":"deploy:decline","style":"danger"}]}]}'
```

تحویل سنجاق‌شده:

```bash
openclaw message send --channel telegram \
  --target -1001234567890 \
  --message "موضوع باز شد" \
  --pin
```

تحویل سنجاق‌شده با JSON صریح:

```json
{
  "pin": {
    "enabled": true,
    "notify": true,
    "required": false
  }
}
```

## قرارداد رندرکننده

Pluginهای کانال، پشتیبانی از رندر را در آداپتور خروجی خود اعلام می‌کنند:

```ts
const adapter: ChannelOutboundAdapter = {
  deliveryMode: "direct",
  presentationCapabilities: {
    supported: true,
    buttons: true,
    selects: true,
    context: true,
    divider: true,
    charts: false,
    tables: false,
    limits: {
      actions: {
        maxActions: 25,
        maxActionsPerRow: 5,
        maxRows: 5,
        maxLabelLength: 80,
        maxValueBytes: 100,
        supportsStyles: true,
        supportsDisabled: false,
      },
      selects: {
        maxOptions: 25,
        maxLabelLength: 100,
        maxValueBytes: 100,
      },
      text: {
        maxLength: 2000,
        encoding: "characters",
        markdownDialect: "discord-markdown",
      },
    },
  },
  deliveryCapabilities: {
    pin: true,
  },
  renderPresentation({ payload, presentation, ctx }) {
    return renderNativePayload(payload, presentation, ctx);
  },
  async pinDeliveredMessage({ target, messageId, pin }) {
    await pinNativeMessage(target, messageId, { notify: pin.notify === true });
  },
};
```

مقادیر بولی قابلیت‌ها مشخص می‌کنند که رندرکننده چه مواردی را می‌تواند تعاملی کند. مقادیر اختیاری
`limits` پوشش عمومی‌ای را توصیف می‌کنند که هسته می‌تواند پیش از فراخوانی
رندرکننده تطبیق دهد:

```ts
type ChannelPresentationCapabilities = {
  supported?: boolean;
  buttons?: boolean;
  selects?: boolean;
  context?: boolean;
  divider?: boolean;
  charts?: boolean;
  tables?: boolean;
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
```

هسته پیش از رندر، محدودیت‌های عمومی را روی کنترل‌های معنایی اعمال می‌کند. رندرکننده‌ها
همچنان مسئول اعتبارسنجی نهایی مختص ارائه‌دهنده و کوتاه‌سازی برای تعداد بلوک‌های بومی،
اندازه کارت، محدودیت‌های URL و ویژگی‌های خاص ارائه‌دهنده‌ای هستند که در
قرارداد عمومی قابل بیان نیستند. اگر محدودیت‌ها همه کنترل‌های یک بلوک را حذف کنند، هسته
برچسب‌ها را به‌صورت متن زمینه‌ای غیرتعاملی نگه می‌دارد تا پیام تحویل‌شده همچنان
یک جایگزین قابل‌مشاهده داشته باشد.

## جریان رندر هسته

در مسیر خروجی معیار که CLI و کنش‌های استاندارد پیام از آن استفاده می‌کنند، هسته:

1. بار ارائه را نرمال‌سازی می‌کند.
2. آداپتور خروجی کانال مقصد را تفکیک می‌کند.
3. `presentationCapabilities` را می‌خواند.
4. هنگامی که آداپتور محدودیت‌ها را اعلام می‌کند، محدودیت‌های عمومی قابلیت مانند تعداد کنش‌ها، طول برچسب و
   تعداد گزینه‌های انتخاب را اعمال می‌کند. بلوک‌های نمودار و جدول به متن قطعی
   تبدیل می‌شوند، مگر آنکه آداپتور به‌ترتیب به‌صراحت
   `charts: true` یا `tables: true` را اعلام کند.
5. وقتی آداپتور بتواند بار را رندر کند، `renderPresentation` را فراخوانی می‌کند.
6. وقتی آداپتور موجود نباشد یا نتواند رندر کند، به متن محافظه‌کارانه برمی‌گردد.
7. بار حاصل را از مسیر عادی تحویل کانال ارسال می‌کند.
8. فراداده تحویل مانند `delivery.pin` را پس از نخستین
   پیام ارسالی موفق اعمال می‌کند.

قیف‌های محلی پاسخ یا پیش‌نمایش کانال که مستقیماً `ReplyPayload` را مصرف می‌کنند،
باید یا وارد آن مسیر معیار شوند یا پیش از تبدیل بار به متن ساده/رسانه،
همان جایگزین ارائه را ایجاد کنند.

هسته مالک رفتار جایگزین است تا تولیدکنندگان بتوانند مستقل از کانال باقی بمانند. Pluginهای
کانال مالک رندر بومی و مدیریت تعامل هستند.

## قواعد تنزل

ارسال ارائه باید در کانال‌های محدود ایمن باشد.

متن جایگزین شامل موارد زیر است:

- `title` به‌عنوان خط نخست
- بلوک‌های `text` به‌صورت بندهای عادی
- بلوک‌های `context` به‌صورت خطوط زمینه‌ای فشرده
- بلوک‌های `divider` به‌صورت جداکننده بصری
- برچسب‌های دکمه، شامل URLها برای دکمه‌های پیوند
- برچسب‌های گزینه‌های انتخاب
- عنوان، نوع، محورها، دسته‌ها، سری‌ها و مقادیر نمودار
- عنوان جدول، سرستون‌ها و مقدار هر ردیف

### نمایان‌بودن مقدار جایگزین دکمه

وقتی کانالی نتواند کنترل‌های تعاملی را رندر کند، مقادیر دکمه و انتخاب
به متن ساده تبدیل می‌شوند. رفتار جایگزین ضمن
خصوصی نگه‌داشتن داده‌های مبهم فراخوان، کاربردپذیری را حفظ می‌کند:

- **کنش‌های دارای نوع `command`** به‌صورت `` label: `command` `` رندر می‌شوند تا کاربران بتوانند
  فرمان را کپی کرده و به‌صورت دستی در ورودی کانال اجرا کنند.
- **کنش‌های دارای نوع `callback`** و فیلدهای قدیمی **`value`** فقط با
  برچسب رندر می‌شوند. مقدار مبهم فراخوان در متن جایگزین افشا نمی‌شود.
- **کنش‌های دارای نوع `approval`** فقط با برچسب رندر می‌شوند. شناسه‌ها و تصمیم‌های تأیید
  داده‌های انتقال هستند و از طریق ابزارهای کمکی اسکالر عمومی یا متن جایگزین
  افشا نمی‌شوند.
- **کنش‌های `url`**، **کنش‌های `web-app` دارای URL** و ورودی‌های منسوخ **`url` /
  `webApp` / `web_app`**، متن URL را در کنار برچسب دکمه رندر می‌کنند،
  زیرا URL برای کاربر قابل‌مشاهده است. کنش‌هایی که فقط برای ویجت میزبانی‌شده هستند، در
  کانال‌های فاقد راه‌اندازی بومی ویجت فقط با برچسب رندر می‌شوند.
- **گزینه‌های انتخاب** فقط با برچسب رندر می‌شوند. مقدار زیربنایی گزینه در
  متن جایگزین افشا نمی‌شود.

آداپتورهای کانالی که راهنمای فرمان دستی را به رابط کاربری جایگزین خود اضافه می‌کنند (برای مثال
دستورالعمل‌های نظر سند Feishu)، باید بررسی وجود فرمان را
از همان بلوک‌های ارائه‌ای استخراج کنند که رندرکننده جایگزین استفاده می‌کند، تا
متن راهنما فقط زمانی ظاهر شود که واقعاً یک فرمان دستی نمایش داده می‌شود.

کنترل‌های بومی پشتیبانی‌نشده باید تنزل یابند، نه اینکه کل ارسال را ناموفق کنند.
مثال‌ها:

- Telegram با دکمه‌های درون‌خطی غیرفعال، متن جایگزین ارسال می‌کند.
- کانالی بدون پشتیبانی از انتخاب، گزینه‌های انتخاب را به‌صورت متن فهرست می‌کند.
- کانالی بدون پشتیبانی بومی از نمودار، داده‌های نمودار را به‌صورت متن فهرست می‌کند.
- کانالی بدون پشتیبانی بومی از جدول، همه ردیف‌های جدول را به‌صورت متن فهرست می‌کند.
- دکمه‌ای که فقط URL دارد، به یک دکمه پیوند بومی یا یک خط URL جایگزین تبدیل می‌شود.
- شکست‌های اختیاری سنجاق‌کردن باعث شکست پیام تحویل‌شده نمی‌شوند.

استثنای اصلی `delivery.pin.required: true` است؛ اگر سنجاق‌کردن به‌عنوان
الزامی درخواست شود و کانال نتواند پیام ارسال‌شده را سنجاق کند، تحویل شکست را گزارش می‌کند.

## نگاشت ارائه‌دهنده

رندرکننده‌های بسته‌بندی‌شده فعلی:

| کانال           | مقصد رندر بومی                           | یادداشت‌ها                                                                                                                                                                                                          |
| --------------- | ----------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Discord         | مؤلفه‌ها و محفظه‌های مؤلفه                | برای تولیدکنندگان موجود بار بومی ارائه‌دهنده، `channelData.discord.components` قدیمی را حفظ می‌کند، اما ارسال‌های اشتراکی جدید باید از `presentation` استفاده کنند.                                                                 |
| Feishu          | کارت‌های تعاملی                           | سربرگ کارت می‌تواند از `title` استفاده کند؛ بدنه از تکرار آن عنوان جلوگیری می‌کند.                                                                                                                                                  |
| Matrix          | متن جایگزین به‌همراه فیلد ساخت‌یافته رویداد | دکمه‌ها/انتخاب‌ها به‌عنوان پشتیبانی‌شده اعلام می‌شوند، اما در حال حاضر هر بلوک به‌صورت خروجی `renderMessagePresentationFallbackText` که در فیلد رویداد `com.openclaw.presentation` حمل می‌شود رندر می‌گردد، نه ویجت‌های تعاملی بومی. |
| Mattermost      | متن به‌همراه ویژگی‌های تعاملی             | انتخاب‌ها و جداکننده‌ها پشتیبانی نمی‌شوند؛ آن بلوک‌ها به متن تنزل می‌یابند.                                                                                                                                             |
| Microsoft Teams | کارت‌های تطبیقی                           | وقتی هر دو ارائه شوند، متن ساده `message` همراه کارت گنجانده می‌شود. انتخاب‌ها، سبک‌ها و وضعیت غیرفعال پشتیبانی نمی‌شوند.                                                                                     |
| Slack           | Block Kit                                 | `chart` را به‌صورت `data_visualization` بومی و `table` را به‌صورت `data_table` بومی رندر می‌کند؛ `channelData.slack.blocks` قدیمی را حفظ می‌کند، اما ارسال‌های اشتراکی جدید باید از `presentation` استفاده کنند.                                   |
| Telegram        | متن به‌همراه صفحه‌کلیدهای درون‌خطی        | دکمه‌ها/انتخاب‌ها برای سطح مقصد به قابلیت دکمه درون‌خطی نیاز دارند؛ در غیر این صورت از متن جایگزین استفاده می‌شود.                                                                                                         |
| کانال‌های ساده  | متن جایگزین                               | کانال‌های بدون رندرکننده همچنان خروجی خوانا دریافت می‌کنند.                                                                                                                                                            |

سازگاری بار بومی ارائه‌دهنده، یک امکان گذار برای تولیدکنندگان موجود
پاسخ است. این دلیلی برای افزودن فیلدهای بومی اشتراکی جدید نیست.

## ارائه در برابر InteractiveReply

`InteractiveReply` زیرمجموعه داخلی قدیمی‌تری است که ابزارهای کمکی تأیید و تعامل
از آن استفاده می‌کنند. این موارد را پشتیبانی می‌کند:

- متن
- دکمه‌ها
- انتخاب‌ها

`MessagePresentation` قرارداد معیار ارسال اشتراکی است. این موارد را اضافه می‌کند:

- عنوان
- لحن
- زمینه
- جداکننده
- نمودار
- جدول
- دکمه‌هایی که فقط URL دارند
- فراداده عمومی تحویل از طریق `ReplyPayload.delivery`

هنگام اتصال کد قدیمی‌تر، از ابزارهای کمکی
`openclaw/plugin-sdk/interactive-runtime` استفاده کنید:

```ts
import {
  adaptMessagePresentationForChannel,
  applyPresentationActionLimits,
  hasMessagePresentationBlocks,
  interactiveReplyToPresentation,
  isMessagePresentationInteractiveBlock,
  normalizeMessagePresentation,
  presentationPageSize,
  presentationToInteractiveControlsReply,
  presentationToInteractiveReply,
  renderMessagePresentationChartFallbackText,
  renderMessagePresentationFallbackText,
  renderMessagePresentationTableFallbackText,
  resolveMessagePresentationActionValue,
  resolveMessagePresentationButtonAction,
  resolveMessagePresentationControlValue,
  resolveMessagePresentationOptionAction,
} from "openclaw/plugin-sdk/interactive-runtime";
```

کد جدید باید مستقیماً `MessagePresentation` را بپذیرد یا تولید کند. بارهای موجود
`interactive` زیرمجموعه‌ای منسوخ از `presentation` هستند؛ پشتیبانی زمان اجرا
برای تولیدکنندگان قدیمی‌تر باقی می‌ماند.

ابزارهای کمکی منسوخ‌نشده‌ای که دانستنشان مفید است:

- `normalizeMessagePresentation(raw)` / `hasMessagePresentationBlocks(value)`
  یک payload بدون نوع را (برای مثال، JSON دریافتی از پرچم
  `--presentation` در CLI) اعتبارسنجی و به `MessagePresentation` تبدیل می‌کنند.
- `isMessagePresentationInteractiveBlock(block)` یک بلوک را به اجتماع
  `buttons` | `select` محدود می‌کند.
- `resolveMessagePresentationButtonAction(button)` و
  `resolveMessagePresentationOptionAction(option)` ضمن پذیرش فیلدهای مرزی منسوخ‌شده،
  کنش نوع‌دار متعارف را برمی‌گردانند. `action` صریح
  همیشه اولویت دارد.
- `resolveMessagePresentationActionValue(action)` /
  `resolveMessagePresentationControlValue(control)` فقط مقادیر اسکالر فرمان/فراخوانی بازگشتی را
  می‌خوانند. یک کنش متعارف غیراسکالر هرگز به `value` سایه‌ای قدیمی
  منتقل نمی‌شود؛ بنابراین شناسه‌های تأیید و مقصدهای پیوند نوع‌دار باقی می‌مانند.
- `renderMessagePresentationChartFallbackText(block)` /
  `renderMessagePresentationTableFallbackText(block)` یک بلوک دادهٔ ساخت‌یافته را برای
  مسیرهای جایگزین مختص کانال، به‌صورت متن قطعی رندر می‌کنند.

نوع‌های قدیمی `InteractiveReply*` و توابع کمکی تبدیل در SDK با
`@deprecated` علامت‌گذاری شده‌اند:

- `InteractiveReply`، `InteractiveReplyBlock`، `InteractiveReplyButton` و
  `InteractiveReplyOption`
- `normalizeInteractiveReply(...)`
- `hasInteractiveReplyBlocks(...)`
- `interactiveReplyToPresentation(...)`
- `presentationToInteractiveReply(...)`
- `presentationToInteractiveControlsReply(...)`
- `resolveInteractiveTextFallback(...)`
- `reduceInteractiveReply(...)`

`presentationToInteractiveReply(...)` و
`presentationToInteractiveControlsReply(...)` همچنان به‌عنوان پل‌های رندرکننده
برای پیاده‌سازی‌های قدیمی کانال در دسترس هستند. کد تولیدکنندهٔ جدید نباید آن‌ها را
فراخوانی کند؛ `presentation` را ارسال کنید و اجازه دهید سازگارسازی هسته/کانال رندر را مدیریت کند.

توابع کمکی تأیید نیز جایگزین‌هایی با اولویت ارائه دارند:

- به‌جای
  `buildApprovalInteractiveReply(...)` از `buildApprovalPresentation(...)` استفاده کنید
- به‌جای
  `buildExecApprovalInteractiveReply(...)` از `buildExecApprovalPresentation(...)` استفاده کنید

این سازنده‌های منتشرشده برای سازگاری Plugin همچنان مبتنی بر فرمان باقی می‌مانند. کد Gateway
و کانال‌های همراه که مالک یک نوع تأیید ماندگار است، باید از
`buildTypedApprovalPresentation(...)`،
`buildTypedExecApprovalPendingReplyPayload(...)` یا
`buildTypedPluginApprovalPendingReplyPayload(...)` استفاده کند تا انتقال‌دهنده‌ها به‌جای استنباط معنا از متن `/approve`،
یک کنش صریح `approval` دریافت کنند.

`renderMessagePresentationFallbackText(...)` برای بلوک‌های ارائه‌ای
که جایگزین متنی ندارند، مانند ارائه‌ای که فقط شامل جداکننده است، رشته‌ای خالی برمی‌گرداند.
انتقال‌دهنده‌هایی که به بدنهٔ ارسال غیرخالی نیاز دارند می‌توانند
`emptyFallback` را برای استفاده از بدنه‌ای حداقلی ارسال کنند، بدون اینکه قرارداد پیش‌فرض
جایگزین را تغییر دهند.

## سنجاق‌کردن تحویل

سنجاق‌کردن یک رفتار تحویل است، نه ارائه. به‌جای فیلدهای بومی ارائه‌دهنده مانند
`channelData.telegram.pin` از `delivery.pin` استفاده کنید.

معناشناسی:

- `pin: true` نخستین پیام با تحویل موفق را سنجاق می‌کند.
- مقدار پیش‌فرض `pin.notify` برابر `false` است.
- مقدار پیش‌فرض `pin.required` برابر `false` است.
- شکست‌های اختیاری سنجاق‌کردن با افت عملکرد ادامه می‌یابند و پیام ارسال‌شده را دست‌نخورده باقی می‌گذارند.
- شکست‌های الزامی سنجاق‌کردن باعث شکست تحویل می‌شوند.
- در پیام‌های تکه‌بندی‌شده، نخستین تکهٔ تحویل‌شده سنجاق می‌شود، نه تکهٔ انتهایی.

کنش‌های دستی پیام `pin`، `unpin` و `pins` همچنان برای پیام‌های
موجودی که ارائه‌دهنده از این عملیات پشتیبانی می‌کند، وجود دارند.

## چک‌لیست نویسندهٔ Plugin

- وقتی کانال می‌تواند ارائهٔ معنایی را رندر کند یا به‌شکلی امن تنزل دهد، `presentation` را از `describeMessageTool(...)`
  اعلام کنید.
- `presentationCapabilities` را به آداپتور خروجی زمان اجرا اضافه کنید.
- `renderPresentation` را در کد زمان اجرا پیاده‌سازی کنید، نه در کد
  راه‌اندازی Plugin صفحهٔ کنترل.
- کتابخانه‌های رابط کاربری بومی را از مسیرهای داغ راه‌اندازی/کاتالوگ خارج نگه دارید.
- وقتی محدودیت‌های عمومی قابلیت مشخص هستند، آن‌ها را در `presentationCapabilities.limits`
  اعلام کنید.
- محدودیت‌های نهایی پلتفرم را در رندرکننده و آزمون‌ها حفظ کنید.
- برای نمودارها، جدول‌ها، دکمه‌ها، گزینه‌های انتخاب، دکمه‌های URL،
  تکرار عنوان/متن و ارسال‌های ترکیبی `message` به‌همراه `presentation`
  که پشتیبانی نمی‌شوند، آزمون‌های جایگزین اضافه کنید.
- فقط زمانی پشتیبانی از سنجاق‌کردن تحویل را از طریق `deliveryCapabilities.pin` و
  `pinDeliveredMessage` اضافه کنید که ارائه‌دهنده بتواند شناسهٔ پیام ارسال‌شده را سنجاق کند.
- فیلدهای جدید کارت/بلوک/مؤلفه/دکمهٔ بومی ارائه‌دهنده را از طریق
  طرح‌وارهٔ مشترک کنش پیام در معرض دسترس قرار ندهید.

## مستندات مرتبط

- [CLI پیام](/fa/cli/message)
- [نمای کلی SDK افزونه](/fa/plugins/sdk-overview)
- [معماری افزونه](/fa/plugins/architecture-internals#message-tool-schemas)
- [طرح بازآرایی ارائهٔ کانال](/fa/plan/ui-channels)
