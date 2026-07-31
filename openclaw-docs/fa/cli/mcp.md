---
read_when:
    - اتصال Codex، Claude Code یا یک کلاینت MCP دیگر به کانال‌های مبتنی بر OpenClaw
    - در حال اجرای `openclaw mcp serve`
    - مدیریت تعریف‌های سرور MCP ذخیره‌شده توسط OpenClaw
sidebarTitle: MCP
summary: مکالمات کانال OpenClaw را از طریق MCP در دسترس قرار دهید و تعریف‌های ذخیره‌شدهٔ سرور MCP را مدیریت کنید
title: MCP
x-i18n:
    generated_at: "2026-07-27T14:59:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ee6146bbc0181d10997336094d1bd693d0afb0985f1febef8e8c6b0d6e656cf9
    source_path: cli/mcp.md
    workflow: 16
---

`openclaw mcp` دو وظیفه دارد:

- اجرای OpenClaw به‌عنوان سرور MCP با `openclaw mcp serve`
- مدیریت تعریف‌های سرور MCP خروجی تحت مدیریت OpenClaw با `list`، `show`، `status`، `doctor`، `probe`، `add`، `set`، `configure`، `tools`، `login`، `logout`، `reload` و `unset`

`serve` حالت عمل‌کردن OpenClaw به‌عنوان سرور MCP است. زیرفرمان‌های دیگر، حالت عمل‌کردن OpenClaw به‌عنوان رجیستری سمت کلاینت MCP برای سرورهایی هستند که زمان‌اجرای خود آن ممکن است بعداً مصرف کند.

<Note>
  `list`، `show`، `set` و `unset` فقط ورودی‌های `mcp.servers` تحت مدیریت OpenClaw را در پیکربندی OpenClaw می‌خوانند و می‌نویسند. آن‌ها سرورهای mcporter از `config/mcporter.json` را شامل نمی‌شوند؛ برای آن رجیستری از `mcporter list` استفاده کنید.
</Note>

وقتی OpenClaw باید خودش یک نشست محیط کدنویسی را میزبانی و آن زمان‌اجرا را از طریق ACP مسیریابی کند، از [`openclaw acp`](/fa/cli/acp) استفاده کنید.

## انتخاب مسیر مناسب MCP

| هدف                                                                | استفاده                                                                  | دلیل                                                                                                             |
| ------------------------------------------------------------------- | -------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| اجازه‌دادن به یک کلاینت خارجی MCP برای خواندن/ارسال مکالمات کانال OpenClaw | `openclaw mcp serve`                                                 | OpenClaw سرور MCP است و مکالمات متکی بر Gateway را از طریق stdio ارائه می‌کند.                                 |
| ذخیره‌کردن سرورهای MCP شخص ثالث برای اجراهای عامل تحت مدیریت OpenClaw        | `openclaw mcp add`، `set`، `configure`، `tools`، `login`             | OpenClaw رجیستری سمت کلاینت MCP است و بعداً آن سرورها را در زمان‌اجراهای واجد شرایط منعکس می‌کند.               |
| بررسی یک سرور ذخیره‌شده بدون اجرای نوبت عامل                  | `openclaw mcp status`، `doctor`، `probe`                             | `status` و `doctor` پیکربندی را بررسی می‌کنند؛ `probe` یک اتصال زنده MCP باز می‌کند و قابلیت‌ها را فهرست می‌کند.               |
| ویرایش پیکربندی MCP از مرورگر                                      | رابط کنترل `/settings/mcp` (نام مستعار `/mcp`)                            | این صفحه موجودی، وضعیت فعال‌سازی، خلاصه‌های OAuth/فیلتر، راهنمای فرمان‌ها و یک ویرایشگر محدود به دامنه برای `mcp` را نشان می‌دهد.         |
| ارائه یک سرور بومی MCP با دامنه محدود به app-server کدکس                    | `mcp.servers.<name>.codex`                                           | بلوک `codex` فقط بر انعکاس رشته app-server کدکس اثر می‌گذارد و پیش از تحویل پیکربندی بومی حذف می‌شود. |
| اجرای نشست‌های محیط میزبانی‌شده با ACP                                     | [`openclaw acp`](/fa/cli/acp) و [عامل‌های ACP](/fa/tools/acp-agents-setup) | حالت پل ACP تزریق سرور MCP به‌ازای هر نشست را نمی‌پذیرد؛ به‌جای آن پل‌های Gateway/Plugin را پیکربندی کنید.     |

<Tip>
اگر مطمئن نیستید به کدام مسیر نیاز دارید، با `openclaw mcp status --verbose` شروع کنید. این فرمان بدون راه‌اندازی هیچ سرور MCP، موارد ذخیره‌شده در OpenClaw را نشان می‌دهد.
</Tip>

## OpenClaw به‌عنوان سرور MCP

این مسیر `openclaw mcp serve` است.

### زمان استفاده از serve

در موارد زیر از `openclaw mcp serve` استفاده کنید:

- کدکس، Claude Code یا یک کلاینت MCP دیگر باید مستقیماً با مکالمات کانال متکی بر OpenClaw ارتباط برقرار کند
- از قبل یک Gateway محلی یا راه‌دور OpenClaw با نشست‌های مسیریابی‌شده دارید
- یک سرور MCP می‌خواهید که در همه زیرساخت‌های کانال OpenClaw کار کند، به‌جای آنکه برای هر کانال پل جداگانه‌ای اجرا کنید

وقتی OpenClaw باید خودش زمان‌اجرای کدنویسی را میزبانی کند و نشست عامل را داخل OpenClaw نگه دارد، به‌جای آن از [`openclaw acp`](/fa/cli/acp) استفاده کنید.

### نحوه کار

`openclaw mcp serve` یک سرور MCP مبتنی بر stdio راه‌اندازی می‌کند. کلاینت MCP مالک آن فرایند است. تا زمانی که کلاینت نشست stdio را باز نگه دارد، پل از طریق WebSocket به یک Gateway محلی یا راه‌دور OpenClaw متصل می‌شود و مکالمات کانال مسیریابی‌شده را از طریق MCP ارائه می‌کند.

<Steps>
  <Step title="کلاینت پل را راه‌اندازی می‌کند">
    کلاینت MCP فرایند `openclaw mcp serve` را راه‌اندازی می‌کند.
  </Step>
  <Step title="پل به Gateway متصل می‌شود">
    پل از طریق WebSocket به Gateway متعلق به OpenClaw متصل می‌شود.
  </Step>
  <Step title="نشست‌ها به مکالمات MCP تبدیل می‌شوند">
    نشست‌های مسیریابی‌شده به مکالمات MCP و ابزارهای رونوشت/تاریخچه تبدیل می‌شوند.
  </Step>
  <Step title="رویدادهای زنده در صف قرار می‌گیرند">
    تا زمانی که پل متصل است، رویدادهای زنده در حافظه در صف قرار می‌گیرند.
  </Step>
  <Step title="ارسال اختیاری Claude">
    اگر حالت کانال Claude فعال باشد، همان نشست می‌تواند اعلان‌های ارسالی ویژه Claude را نیز دریافت کند.
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="رفتار مهم">
    - وضعیت صف زنده هنگام اتصال پل آغاز می‌شود
    - تاریخچه رونوشت قدیمی‌تر با `messages_read` خوانده می‌شود
    - اعلان‌های ارسالی Claude فقط تا زمانی وجود دارند که نشست MCP زنده باشد
    - وقتی کلاینت قطع می‌شود، پل خارج می‌شود و صف زنده از بین می‌رود
    - نقاط ورود یک‌باره عامل مانند `openclaw agent` و `openclaw infer model run` هر زمان‌اجرای همراه MCP را که باز می‌کنند، پس از تکمیل پاسخ خاتمه می‌دهند؛ بنابراین اجراهای اسکریپتی تکراری باعث انباشته‌شدن فرایندهای فرزند MCP مبتنی بر stdio نمی‌شوند
    - سرورهای MCP مبتنی بر stdio که OpenClaw راه‌اندازی می‌کند (همراه یا پیکربندی‌شده توسط کاربر)، هنگام خاموش‌شدن به‌صورت یک درخت فرایند خاتمه داده می‌شوند؛ بنابراین زیرفرایندهایی که سرور آغاز کرده است، پس از خروج کلاینت والد stdio باقی نمی‌مانند
    - حذف یا بازنشانی یک نشست، کلاینت‌های MCP آن نشست را از طریق مسیر مشترک پاک‌سازی زمان‌اجرا آزاد می‌کند؛ بنابراین هیچ اتصال stdio باقی‌مانده‌ای به نشست حذف‌شده وابسته نمی‌ماند

  </Accordion>
</AccordionGroup>

### انتخاب حالت کلاینت

<Tabs>
  <Tab title="کلاینت‌های عمومی MCP">
    فقط ابزارهای استاندارد MCP. از `conversations_list`، `messages_read`، `events_poll`، `events_wait`، `messages_send` و ابزارهای تأیید استفاده کنید.
  </Tab>
  <Tab title="Claude Code">
    ابزارهای استاندارد MCP به‌علاوه آداپتور کانال ویژه Claude. `--claude-channel-mode on` را فعال کنید یا مقدار پیش‌فرض `auto` را باقی بگذارید.
  </Tab>
</Tabs>

<Note>
در حال حاضر، `auto` همانند `on` رفتار می‌کند. هنوز تشخیص قابلیت‌های کلاینت وجود ندارد.
</Note>

### مواردی که serve ارائه می‌کند

پل با استفاده از فراداده مسیر نشست موجود در Gateway، مکالمات متکی بر کانال را ارائه می‌کند. یک مکالمه زمانی ظاهر می‌شود که OpenClaw از قبل وضعیت نشستی با مسیری شناخته‌شده مانند موارد زیر داشته باشد:

- `channel`
- فراداده گیرنده یا مقصد
- `accountId` اختیاری
- `threadId` اختیاری

این قابلیت یک مکان واحد برای انجام کارهای زیر در اختیار کلاینت‌های MCP قرار می‌دهد:

- فهرست‌کردن مکالمات مسیریابی‌شده اخیر
- خواندن تاریخچه اخیر رونوشت
- انتظار برای رویدادهای ورودی جدید
- ارسال پاسخ از همان مسیر
- مشاهده درخواست‌های تأییدی که هنگام اتصال پل می‌رسند

### استفاده

<Tabs>
  <Tab title="Gateway محلی">
    ```bash
    openclaw mcp serve
    ```
  </Tab>
  <Tab title="Gateway راه‌دور (توکن)">
    ```bash
    openclaw mcp serve --url wss://gateway-host:18789 --token-file ~/.openclaw/gateway.token
    ```
  </Tab>
  <Tab title="Gateway راه‌دور (گذرواژه)">
    ```bash
    openclaw mcp serve --url wss://gateway-host:18789 --password-file ~/.openclaw/gateway.password
    ```
  </Tab>
  <Tab title="پرحرف / Claude خاموش">
    ```bash
    openclaw mcp serve --verbose
    openclaw mcp serve --claude-channel-mode off
    ```
  </Tab>
</Tabs>

### ابزارهای پل

<AccordionGroup>
  <Accordion title="conversations_list">
    مکالمات اخیر متکی بر نشستی را فهرست می‌کند که از قبل در وضعیت نشست Gateway دارای فراداده مسیر هستند.

    فیلترها: `limit` (حداکثر 500)، `search`، `channel`، `includeDerivedTitles`، `includeLastMessage`.

  </Accordion>
  <Accordion title="conversation_get">
    یک مکالمه را با `session_key` و از طریق جست‌وجوی مستقیم نشست Gateway برمی‌گرداند.
  </Accordion>
  <Accordion title="messages_read">
    پیام‌های اخیر رونوشت را برای یک مکالمه متکی بر نشست می‌خواند. مقدار پیش‌فرض `limit` برابر 20 و حداکثر آن 200 است.
  </Accordion>
  <Accordion title="attachments_fetch">
    بلوک‌های محتوای غیرمتنی را از یک پیام رونوشت استخراج می‌کند. این نمایی از فراداده روی محتوای رونوشت است، نه یک مخزن مستقل و ماندگار برای اشیای پیوست.
  </Accordion>
  <Accordion title="events_poll">
    رویدادهای زنده در صف را از یک مکان‌نمای عددی به بعد می‌خواند. حداکثر `limit` برابر 200 است.
  </Accordion>
  <Accordion title="events_wait">
    تا رسیدن رویداد بعدی در صف که با شرایط مطابقت دارد، یا پایان مهلت زمانی، به‌صورت نظرسنجی طولانی منتظر می‌ماند (پیش‌فرض 30s و حداکثر 300s).

    زمانی از این ابزار استفاده کنید که یک کلاینت عمومی MCP بدون پروتکل ارسال ویژه Claude به تحویل تقریباً بلادرنگ نیاز دارد.

  </Accordion>
  <Accordion title="messages_send">
    متن را از همان مسیری که از قبل در نشست ثبت شده است، ارسال می‌کند.

    رفتار فعلی:

    - به یک مسیر مکالمه موجود نیاز دارد
    - از کانال، گیرنده، شناسه حساب و شناسه رشته نشست استفاده می‌کند
    - فقط متن ارسال می‌کند

  </Accordion>
  <Accordion title="permissions_list_open">
    درخواست‌های در انتظار تأیید اجرای فرمان/Plugin را که پل از زمان اتصال به Gateway مشاهده کرده است، فهرست می‌کند.
  </Accordion>
  <Accordion title="permissions_respond">
    یک درخواست در انتظار تأیید اجرای فرمان/Plugin را با یکی از موارد زیر تعیین تکلیف می‌کند:

    - `allow-once`
    - `allow-always`
    - `deny`

  </Accordion>
</AccordionGroup>

### مدل رویداد

پل تا زمانی که متصل است، یک صف رویداد در حافظه نگه می‌دارد.

انواع فعلی رویداد:

- `message`
- `exec_approval_requested`
- `exec_approval_resolved`
- `plugin_approval_requested`
- `plugin_approval_resolved`
- `claude_permission_request`

<Warning>
- صف فقط زنده است؛ هنگام راه‌اندازی پل MCP آغاز می‌شود
- `events_poll` و `events_wait` به‌خودی‌خود تاریخچه قدیمی‌تر Gateway را بازپخش نمی‌کنند
- صف پایدار عقب‌افتاده باید با `messages_read` خوانده شود

</Warning>

### اعلان‌های کانال Claude

پل همچنین می‌تواند اعلان‌های کانال ویژه Claude را ارائه کند. این قابلیت معادل OpenClaw برای آداپتور کانال Claude Code است: ابزارهای استاندارد MCP همچنان در دسترس می‌مانند، اما پیام‌های ورودی زنده می‌توانند به‌شکل اعلان‌های MCP ویژه Claude نیز برسند.

<Tabs>
  <Tab title="خاموش">
    `--claude-channel-mode off`: فقط ابزارهای استاندارد MCP.
  </Tab>
  <Tab title="روشن">
    `--claude-channel-mode on`: اعلان‌های کانال Claude را فعال می‌کند.
  </Tab>
  <Tab title="خودکار (پیش‌فرض)">
    `--claude-channel-mode auto`: پیش‌فرض فعلی؛ رفتار پل مشابه `on` است.
  </Tab>
</Tabs>

وقتی حالت کانال Claude فعال باشد، سرور قابلیت‌های آزمایشی Claude را اعلام می‌کند و می‌تواند موارد زیر را منتشر کند:

- `notifications/claude/channel`
- `notifications/claude/channel/permission`

رفتار فعلی پل:

- پیام‌های ورودی رونوشت `user` به‌صورت `notifications/claude/channel` هدایت می‌شوند
- درخواست‌های مجوز Claude که از طریق MCP دریافت می‌شوند، در حافظه رهگیری می‌شوند
- اگر مالک فرمان در مکالمه پیوندشده بعداً `yes <id>` یا `no <id>` را ارسال کند (`<id>` شناسه 5 حرفی درخواست، بدون `l` است)، پل آن را به `notifications/claude/channel/permission` تبدیل می‌کند
- این اعلان‌ها فقط مختص نشست زنده هستند؛ اگر کلاینت MCP قطع شود، هیچ مقصد ارسالی وجود نخواهد داشت

این رفتار عمداً مختص کلاینت است. کلاینت‌های عمومی MCP باید به ابزارهای استاندارد نظرسنجی متکی باشند.

### پیکربندی کلاینت MCP

نمونه پیکربندی کلاینت stdio:

```json
{
  "mcpServers": {
    "openclaw": {
      "command": "openclaw",
      "args": [
        "mcp",
        "serve",
        "--url",
        "wss://gateway-host:18789",
        "--token-file",
        "/path/to/gateway.token"
      ]
    }
  }
}
```

برای بیشتر کلاینت‌های عمومی MCP، با سطح استاندارد ابزار شروع کنید و حالت Claude را نادیده بگیرید. حالت Claude را فقط برای کلاینت‌هایی فعال کنید که واقعاً متدهای اعلان مختص Claude را درک می‌کنند.

### گزینه‌ها

`openclaw mcp serve` از موارد زیر پشتیبانی می‌کند:

<ParamField path="--url" type="string">
  نشانی WebSocket مربوط به Gateway. در صورت پیکربندی، مقدار پیش‌فرض `gateway.remote.url` است.
</ParamField>
<ParamField path="--token" type="string">
  توکن Gateway.
</ParamField>
<ParamField path="--token-file" type="string">
  خواندن توکن از فایل.
</ParamField>
<ParamField path="--password" type="string">
  گذرواژه Gateway.
</ParamField>
<ParamField path="--password-file" type="string">
  خواندن گذرواژه از فایل.
</ParamField>
<ParamField path="--claude-channel-mode" type='"auto" | "on" | "off"'>
  حالت اعلان Claude. مقدار پیش‌فرض `auto` است.
</ParamField>
<ParamField path="-v, --verbose" type="boolean">
  گزارش‌های مفصل در stderr.
</ParamField>

<Tip>
در صورت امکان، `--token-file` یا `--password-file` را به اسرار درون‌خطی ترجیح دهید.
</Tip>

### مرز امنیت و اعتماد

پل، مسیریابی را ابداع نمی‌کند. فقط مکالماتی را در دسترس قرار می‌دهد که Gateway از قبل نحوه مسیریابی آن‌ها را می‌داند.

این یعنی:

- فهرست‌های مجاز فرستندگان، جفت‌سازی و اعتماد در سطح کانال همچنان به پیکربندی کانال زیربنایی OpenClaw تعلق دارند
- `messages_send` فقط می‌تواند از طریق یک مسیر ذخیره‌شده موجود پاسخ دهد
- وضعیت تأیید فقط برای نشست فعلی پل، زنده و درون حافظه است
- احراز هویت پل باید از همان کنترل‌های توکن یا گذرواژه Gateway استفاده کند که برای هر کلاینت راه‌دور دیگر Gateway به آن‌ها اعتماد می‌کنید

اگر مکالمه‌ای در `conversations_list` وجود ندارد، علت معمول پیکربندی MCP نیست. علت، نبودن یا ناقص‌بودن فراداده مسیر در نشست زیربنایی Gateway است.

### آزمایش

OpenClaw یک آزمون دود قطعی Docker برای این پل ارائه می‌کند:

```bash
pnpm test:docker:mcp-channels
```

این آزمون دود یک کانتینر واحد را اجرا می‌کند: وضعیت مکالمه را مقداردهی اولیه می‌کند، Gateway را راه‌اندازی می‌کند، سپس `openclaw mcp serve` را به‌صورت یک فرایند فرزند stdio اجرا کرده و آن را به‌عنوان کلاینت MCP هدایت می‌کند. این آزمون کشف مکالمه، خواندن رونوشت، خواندن فراداده پیوست، رفتار صف رویداد زنده و اعلان‌های کانال و مجوز به سبک Claude را از طریق پل واقعی stdio MCP تأیید می‌کند. مسیریابی ارسال خروجی (`messages_send` با استفاده مجدد از مسیر ذخیره‌شده مکالمه) به‌طور جداگانه با آزمون‌های واحد در `src/mcp/channel-server.test.ts` پوشش داده می‌شود.

این سریع‌ترین راه برای اثبات عملکرد پل است، بدون آنکه یک حساب واقعی Telegram، Discord یا iMessage را به اجرای آزمایش متصل کنید.

برای زمینه گسترده‌تر آزمایش، به [آزمایش](/fa/help/testing) مراجعه کنید.

### عیب‌یابی

<AccordionGroup>
  <Accordion title="هیچ مکالمه‌ای برگردانده نمی‌شود">
    معمولاً به این معناست که نشست Gateway از قبل قابل مسیریابی نیست. تأیید کنید که نشست زیربنایی، فراداده ذخیره‌شده کانال/ارائه‌دهنده، گیرنده و مسیر اختیاری حساب/رشته را دارد.
  </Accordion>
  <Accordion title="events_poll یا events_wait پیام‌های قدیمی‌تر را از دست می‌دهد">
    مورد انتظار است. صف زنده هنگام اتصال پل آغاز می‌شود. تاریخچه قدیمی‌تر رونوشت را با `messages_read` بخوانید.
  </Accordion>
  <Accordion title="اعلان‌های Claude نمایش داده نمی‌شوند">
    همه موارد زیر را بررسی کنید:

    - کلاینت نشست stdio MCP را باز نگه داشته است
    - `--claude-channel-mode` برابر با `on` یا `auto` است
    - کلاینت واقعاً متدهای اعلان مختص Claude را درک می‌کند
    - پیام ورودی پس از اتصال پل دریافت شده است

  </Accordion>
  <Accordion title="تأییدها وجود ندارند">
    `permissions_list_open` فقط درخواست‌های تأییدی را نمایش می‌دهد که هنگام اتصال پل مشاهده شده‌اند. این یک API پایدار برای تاریخچه تأییدها نیست.
  </Accordion>
</AccordionGroup>

## OpenClaw به‌عنوان رجیستری کلاینت MCP

این مسیر `openclaw mcp list`، `show`، `status`، `doctor`، `probe`، `add`، `set`،
`configure`، `tools`، `login`، `logout`، `reload` و `unset` است.

این فرمان‌ها OpenClaw را از طریق MCP در دسترس قرار نمی‌دهند. آن‌ها تعریف‌های سرور MCP مدیریت‌شده توسط OpenClaw را در `mcp.servers` در پیکربندی OpenClaw مدیریت می‌کنند. آن‌ها سرورهای mcporter را از `config/mcporter.json` نمی‌خوانند.

این تعریف‌های ذخیره‌شده برای محیط‌های اجرایی‌ای هستند که OpenClaw بعداً راه‌اندازی یا پیکربندی می‌کند، مانند OpenClaw تعبیه‌شده و دیگر آداپتورهای محیط اجرایی. OpenClaw تعریف‌ها را به‌صورت متمرکز ذخیره می‌کند تا آن محیط‌های اجرایی مجبور نباشند فهرست‌های تکراری سرور MCP خود را نگه دارند.

<AccordionGroup>
  <Accordion title="رفتار مهم">
    - این فرمان‌ها فقط پیکربندی OpenClaw را می‌خوانند یا می‌نویسند
    - `status`، `list`، `show`، `doctor` بدون `--probe`، `set`، `configure`، `tools`، `logout`، `reload` و `unset` به سرور MCP مقصد متصل نمی‌شوند
    - `login` جریان شبکه OAuth مربوط به MCP را برای سرور HTTP پیکربندی‌شده انجام می‌دهد و اعتبارنامه‌های محلی حاصل را ذخیره می‌کند
    - `status --verbose` بدون اتصال، نکات مربوط به انتقال، احراز هویت، مهلت زمانی، فیلتر و فراخوانی موازی ابزار را پس از رفع مقادیر چاپ می‌کند
    - `doctor` تعریف‌های ذخیره‌شده را برای مشکلات راه‌اندازی محلی مانند نبود فرمان‌های stdio، نامعتبر بودن دایرکتوری‌های کاری، نبود فایل‌های TLS، سرورهای غیرفعال، مقادیر حساس صریح در سرآیند/محیط و مجوزدهی ناقص OAuth بررسی می‌کند
    - `doctor --probe` پس از موفقیت بررسی‌های ایستا، همان اثبات اتصال زنده `probe` را اضافه می‌کند
    - `probe` به سرور انتخاب‌شده یا همه سرورهای پیکربندی‌شده متصل می‌شود، ابزارها را فهرست می‌کند و قابلیت‌ها/اطلاعات تشخیصی را گزارش می‌دهد
    - `add` تعریفی را از پرچم‌ها می‌سازد و پیش از ذخیره آن را بررسی می‌کند، مگر اینکه `--no-probe` تنظیم شده باشد یا ابتدا مجوزدهی OAuth لازم باشد
    - آداپتورهای محیط اجرایی در زمان اجرا تصمیم می‌گیرند که واقعاً از کدام شکل‌های انتقال پشتیبانی کنند
    - `enabled: false` سرور را ذخیره‌شده نگه می‌دارد، اما آن را از کشف محیط اجرایی تعبیه‌شده کنار می‌گذارد
    - `requestTimeoutMs` و `connectionTimeoutMs` مهلت‌های زمانی درخواست و اتصال هر سرور را برحسب میلی‌ثانیه تنظیم می‌کنند
    - `supportsParallelToolCalls: true` سرورهایی را مشخص می‌کند که آداپتورها می‌توانند به‌طور هم‌زمان فراخوانی کنند
    - سرورهای HTTP می‌توانند از سرآیندهای ایستا، ورود OAuth، کنترل اعتبارسنجی TLS و مسیرهای گواهی/کلید mTLS استفاده کنند
    - OpenClaw تعبیه‌شده ابزارهای MCP پیکربندی‌شده را در پروفایل‌های عادی ابزار `coding` و `messaging` در دسترس قرار می‌دهد؛ `minimal` همچنان آن‌ها را پنهان می‌کند و `tools.deny: ["bundle-mcp"]` صراحتاً آن‌ها را غیرفعال می‌کند
    - `toolFilter.include` و `toolFilter.exclude` مختص هر سرور، ابزارهای MCP کشف‌شده را پیش از تبدیل‌شدن به ابزارهای OpenClaw فیلتر می‌کنند
    - سرورهایی که منابع یا پرامپت‌ها را اعلام می‌کنند، ابزارهای کمکی برای فهرست‌کردن/خواندن منابع و فهرست‌کردن/دریافت پرامپت‌ها نیز در دسترس قرار می‌دهند؛ نام‌های کمکی تولیدشده (`resources_list`، `resources_read`، `prompts_list`، `prompts_get`) از همان فیلتر شمول/حذف استفاده می‌کنند
    - تغییرات پویای فهرست ابزار MCP، کاتالوگ کش‌شده آن نشست را نامعتبر می‌کند؛ کشف/استفاده بعدی آن را از سرور تازه‌سازی می‌کند
    - خطاهای مکرر درخواست ابزار/پروتکل MCP آن سرور را برای مدت کوتاهی متوقف می‌کنند تا یک سرور خراب تمام نوبت را مصرف نکند
    - محیط‌های اجرایی همراه MCP با دامنه نشست، پس از 10 دقیقه بی‌کاری جمع‌آوری می‌شوند و اجراهای یک‌باره تعبیه‌شده آن‌ها را در پایان اجرا پاک‌سازی می‌کنند

  </Accordion>
</AccordionGroup>

آداپتورهای محیط اجرایی ممکن است این رجیستری مشترک را به شکلی تبدیل کنند که کلاینت پایین‌دستی آن‌ها انتظار دارد. برای نمونه، OpenClaw تعبیه‌شده مقادیر `transport` مربوط به OpenClaw را مستقیماً مصرف می‌کند، درحالی‌که Claude Code و Gemini مقادیر بومی CLI در `type` مانند `http`، `sse` یا `stdio` را دریافت می‌کنند.

Codex app-server همچنین یک بلوک اختیاری `codex` را برای هر سرور رعایت می‌کند. این
فراداده نگاشت OpenClaw فقط برای رشته‌های Codex app-server است؛ این فراداده
نشست‌های ACP، پیکربندی عمومی چارچوب Codex یا دیگر آداپتورهای محیط اجرایی را
تغییر نمی‌دهد. از `codex.agents` غیرخالی استفاده کنید تا یک سرور فقط به شناسه‌های مشخص عامل
OpenClaw نگاشت شود. فهرست‌های خالی، سفید یا نامعتبر عامل توسط اعتبارسنجی
پیکربندی رد می‌شوند و به‌جای جهانی‌شدن، در مسیر نگاشت محیط اجرایی حذف
می‌شوند. از `codex.defaultToolsApprovalMode` (`auto`، `prompt` یا `approve`)
برای تولید `default_tools_approval_mode` بومی Codex برای یک سرور مورداعتماد استفاده کنید.
OpenClaw پیش از تحویل پیکربندی بومی `mcp_servers` به Codex، فراداده `codex`
را حذف می‌کند.

### تعریف‌های ذخیره‌شده سرور MCP

فرمان‌ها:

- `openclaw mcp list`
- `openclaw mcp show [name]`
- `openclaw mcp status [--verbose]`
- `openclaw mcp doctor [name] [--probe]`
- `openclaw mcp probe [name]`
- `openclaw mcp add <name> [flags]`
- `openclaw mcp set <name> <json>`
- `openclaw mcp configure <name> [flags]`
- `openclaw mcp tools <name> [--include csv] [--exclude csv] [--clear]`
- `openclaw mcp login <name> [--code code]`
- `openclaw mcp logout <name>`
- `openclaw mcp reload`
- `openclaw mcp unset <name>`

نکات:

- `list` نام سرورها را مرتب می‌کند.
- `show` بدون نام، شیء کامل سرور MCP پیکربندی‌شده را چاپ می‌کند.
- `status` انتقال‌های پیکربندی‌شده را بدون اتصال دسته‌بندی می‌کند. `--verbose` جزئیات رفع‌شده راه‌اندازی، مهلت زمانی، OAuth، فیلتر و فراخوانی موازی را شامل می‌شود، از جمله زمانی که توکن‌های ذخیره‌شده OAuth به مجوزدهی بیشتری نیاز دارند. آرگومان‌های stdio حاوی اعتبارنامه در خروجی متنی و JSON پوشانده می‌شوند.
- `doctor` بررسی‌های ایستا را بدون اتصال انجام می‌دهد. زمانی که فرمان باید اتصال سرورهای فعال را نیز تأیید کند، `--probe` را اضافه کنید.
- `probe` متصل می‌شود و تعداد ابزارها، پشتیبانی از منابع/پرامپت‌ها، پشتیبانی از تغییر فهرست و اطلاعات تشخیصی را گزارش می‌دهد.
- `add` پرچم‌های stdio مانند `--command`، `--arg`، `--env` و `--cwd` یا پرچم‌های HTTP مانند `--url`، `--transport`، `--header`، `--auth oauth`، TLS، مهلت زمانی و پرچم‌های انتخاب ابزار را می‌پذیرد.
- `set` انتظار دارد یک مقدار شیء JSON در خط فرمان دریافت کند.
- `configure` وضعیت فعال‌بودن، فیلترهای ابزار، مهلت‌های زمانی، OAuth، TLS و نکات فراخوانی موازی ابزار را بدون جایگزینی کل تعریف سرور به‌روزرسانی می‌کند. برای تأیید سرور به‌روزشده پیش از ذخیره، `--probe` را اضافه کنید.
- `tools` فیلترهای ابزار هر سرور را به‌روزرسانی می‌کند. ورودی‌های شمول/حذف، نام ابزارهای MCP و الگوهای ساده `*` هستند.
- `login` جریان OAuth را برای سرورهای HTTP پیکربندی‌شده با `auth: "oauth"` اجرا می‌کند. اجرای نخست یک نشانی مجوزدهی چاپ می‌کند؛ پس از تأیید، آن را دوباره با `--code` اجرا کنید.
- `logout` اعتبارنامه‌های ذخیره‌شده OAuth را برای سرور نام‌برده پاک می‌کند، بدون اینکه تعریف ذخیره‌شده سرور را حذف کند.
- `reload` محیط‌های اجرایی MCP کش‌شده درون‌فرایندی را فقط برای فرایند فعلی CLI آزاد می‌کند. فرایندهای Gateway یا عامل در فرایندی دیگر همچنان به مسیر بارگذاری مجدد یا راه‌اندازی مجدد خود نیاز دارند.
- برای سرورهای Streamable HTTP MCP از `transport: "streamable-http"` استفاده کنید. `openclaw mcp set` همچنین برای سازگاری، `type: "http"` بومی CLI را به همان شکل متعارف پیکربندی تبدیل می‌کند.
- اگر سرور نام‌برده وجود نداشته باشد، `unset` با شکست مواجه می‌شود.

نمونه‌ها:

```bash
openclaw mcp list
openclaw mcp show context7 --json
openclaw mcp status --verbose
openclaw mcp doctor --probe
openclaw mcp probe context7 --json
openclaw mcp add memory --command npx --arg -y --arg @modelcontextprotocol/server-memory
openclaw mcp set context7 '{"command":"uvx","args":["context7-mcp"]}'
openclaw mcp tools context7 --include 'resolve-library-id,get-library-docs'
openclaw mcp set docs '{"url":"https://mcp.example.com","transport":"streamable-http"}'
openclaw mcp configure docs --timeout 20 --connect-timeout 5 --include 'search,read_*'
openclaw mcp configure docs --auth oauth --oauth-scope 'docs.read'
openclaw mcp login docs
openclaw mcp logout docs
openclaw mcp unset context7
```

### دستورالعمل‌های رایج سرور

این نمونه‌ها فقط تعریف‌های سرور را ذخیره می‌کنند. پس از آن، `openclaw mcp doctor --probe` را اجرا کنید تا ثابت شود سرور راه‌اندازی می‌شود و ابزارها را ارائه می‌دهد.

<Tabs>
  <Tab title="سیستم فایل">
    ```bash
    openclaw mcp add files \
      --command npx \
      --arg -y \
      --arg @modelcontextprotocol/server-filesystem \
      --arg "$HOME/Documents" \
      --include 'read_file,list_directory,search_files'
    openclaw mcp doctor files --probe
    ```

    دامنه سرورهای سیستم فایل را به کوچک‌ترین درخت دایرکتوری که عامل باید بخواند یا ویرایش کند محدود کنید.

  </Tab>
  <Tab title="حافظه">
    ```bash
    openclaw mcp add memory \
      --command npx \
      --arg -y \
      --arg @modelcontextprotocol/server-memory
    openclaw mcp probe memory --json
    ```

    اگر سرور ابزارهای نوشتنی ارائه می‌دهد که نباید در دسترس عامل‌های عادی باشند، از فیلتر ابزار استفاده کنید.

  </Tab>
  <Tab title="اسکریپت محلی">
    ```bash
    openclaw mcp add local-tools \
      --command node \
      --arg ./dist/mcp-server.js \
      --cwd /srv/openclaw-tools \
      --env API_BASE=https://internal.example
    openclaw mcp status --verbose
    ```

    `doctor` بررسی می‌کند که `cwd` وجود داشته باشد و فرمان از محیط پیکربندی‌شده قابل یافتن باشد.

  </Tab>
  <Tab title="HTTP راه دور">
    ```bash
    openclaw mcp add docs \
      --url https://mcp.example.com/mcp \
      --transport streamable-http \
      --auth oauth \
      --oauth-scope docs.read \
      --timeout 20 \
      --connect-timeout 5 \
      --include 'search,read_*'
    openclaw mcp doctor docs --probe
    ```

    وقتی سرور راه دور از OAuth پشتیبانی می‌کند، از آن استفاده کنید. اگر سرور به سرآیندهای ثابت نیاز دارد، از ثبت توکن‌های حامل صریح در مخزن خودداری کنید.

  </Tab>
  <Tab title="دسکتاپ/CUA">
    ```bash
    openclaw mcp set cua-driver '{"command":"cua-driver","args":["mcp"]}'
    openclaw mcp tools cua-driver --include 'list_apps,get_window_state,click,type_text'
    openclaw mcp doctor cua-driver --probe
    ```

    سرورهای کنترل مستقیم دسکتاپ، مجوزهای فرایندی را که راه‌اندازی می‌کنند به ارث می‌برند. از فیلترهای محدود ابزار و درخواست‌های مجوز در سطح سیستم‌عامل استفاده کنید.

  </Tab>
</Tabs>

### ساختارهای خروجی JSON

برای اسکریپت‌ها و داشبوردها از `--json` استفاده کنید. مجموعه فیلدها ممکن است به‌مرور زمان گسترش یابد، بنابراین مصرف‌کنندگان باید کلیدهای ناشناخته را نادیده بگیرند.

<AccordionGroup>
  <Accordion title="status --json">
    ```json
    {
      "path": "/home/user/.openclaw/openclaw.json",
      "servers": [
        {
          "name": "docs",
          "configured": true,
          "enabled": true,
          "ok": true,
          "transport": "streamable-http",
          "launch": "streamable-http https://mcp.example.com/mcp",
          "auth": "oauth",
          "authStatus": {
            "hasTokens": true,
            "requiresAuthorization": false,
            "hasClientInformation": true,
            "hasCodeVerifier": false,
            "hasDiscoveryState": true,
            "hasLastAuthorizationUrl": false
          },
          "requestTimeoutMs": 20000,
          "connectionTimeoutMs": 5000,
          "toolFilter": {
            "include": ["search", "read_*"],
            "exclude": []
          },
          "supportsParallelToolCalls": true
        }
      ]
    }
    ```
  </Accordion>
  <Accordion title="doctor --json">
    ```json
    {
      "ok": true,
      "path": "/home/user/.openclaw/openclaw.json",
      "servers": [
        {
          "name": "docs",
          "ok": true,
          "issues": [
            {
              "level": "warning",
              "message": "اعتبارنامه‌های OAuth مجاز نشده‌اند؛ openclaw mcp login docs را اجرا کنید"
            }
          ]
        }
      ]
    }
    ```

    هنگامی که هر سرور فعالِ بررسی‌شده مشکلی در سطح `error` داشته باشد، `doctor --json` با کد خروج غیرصفر پایان می‌یابد. مشکلات `warning` و `info` گزارش می‌شوند، اما به‌تنهایی باعث شکست فرمان نمی‌شوند.

  </Accordion>
  <Accordion title="probe --json">
    ```json
    {
      "generatedAt": "2026-05-31T09:00:00.000Z",
      "servers": {
        "docs": {
          "launch": "streamable-http https://mcp.example.com/mcp",
          "tools": 2,
          "resources": true,
          "listChanged": {
            "tools": true,
            "resources": false,
            "prompts": false
          }
        }
      },
      "tools": ["docs__read_page", "docs__search"],
      "diagnostics": []
    }
    ```

    `probe --json` یک نشست زنده کلاینت MCP باز می‌کند و نتیجه آن را مستقیماً چاپ می‌کند؛ برخلاف `status`/`doctor`، خروجی هیچ فیلد سطح‌بالای `path` ندارد. کلیدهای `resources` و `prompts` فقط زمانی وجود دارند که سرور واقعاً آن قابلیت را اعلام کند (سروری بدون اعلان‌ها، به‌جای گزارش `false`، کلید `prompts` را حذف می‌کند). از `probe` برای اثبات دسترس‌پذیری و قابلیت‌ها استفاده کنید، نه برای ممیزی پیکربندی ایستا.

  </Accordion>
</AccordionGroup>

نمونه ساختار پیکربندی:

```json
{
  "mcp": {
    "servers": {
      "context7": {
        "command": "uvx",
        "args": ["context7-mcp"]
      },
      "docs": {
        "url": "https://mcp.example.com",
        "transport": "streamable-http",
        "requestTimeoutMs": 20000,
        "connectionTimeoutMs": 5000,
        "supportsParallelToolCalls": true,
        "auth": "oauth",
        "oauth": {
          "scope": "docs.read"
        },
        "sslVerify": true,
        "clientCert": "/path/to/client.crt",
        "clientKey": "/path/to/client.key",
        "toolFilter": {
          "include": ["search_*"],
          "exclude": ["admin_*"]
        }
      }
    }
  }
}
```

### انتقال Stdio

یک فرایند فرزند محلی را راه‌اندازی می‌کند و از طریق stdin/stdout ارتباط برقرار می‌کند.

| فیلد                      | توضیحات                       |
| -------------------------- | --------------------------------- |
| `command`                  | فایل اجرایی برای راه‌اندازی (الزامی)    |
| `args`                     | آرایه آرگومان‌های خط فرمان   |
| `env`                      | متغیرهای محیطی اضافی       |
| `cwd` / `workingDirectory` | دایرکتوری کاری فرایند |

<Warning>
**فیلتر ایمنی محیط Stdio**

OpenClaw پیش از راه‌اندازی یک سرور stdio MCP، کلیدهای محیطی مربوط به آغاز به کار مفسر، ربایش بارگذار و مقداردهی اولیه پوسته را رد می‌کند، حتی اگر در بلوک `env` سرور ظاهر شوند. این سازوکار از همان خط‌مشی امنیتی محیط میزبان استفاده می‌کند که برای دیگر فرایندهای راه‌اندازی‌شده به‌دست OpenClaw به‌کار می‌رود: قلاب‌های شناخته‌شده آغاز به کار مفسر (برای مثال `NODE_OPTIONS`، `PYTHONSTARTUP`، `PERL5OPT`، `RUBYOPT`، `BASHOPTS`، `KSH_ENV`)؛ پیشوندهای تزریق کتابخانه اشتراکی و تابع (`DYLD_*`، `LD_*`، `BASH_FUNC_*`)؛ و متغیرهای مشابه کنترل زمان اجرا را مسدود می‌کند. هنگام راه‌اندازی، این موارد بی‌صدا حذف و یک هشدار ثبت می‌شود تا نتوانند مقدمه‌ای ضمنی تزریق کنند، مفسر را جابه‌جا کنند، اشکال‌زدا را فعال کنند یا پیونددهنده پویا را علیه فرایند stdio بربایند. یک فهرست مجاز صریح، متغیرهای محیطی معمول اعتبارنامه MCP را قابل‌استفاده نگه می‌دارد (`GITHUB_TOKEN`، `GH_TOKEN`، `GITLAB_TOKEN`، `NPM_TOKEN`، `NODE_AUTH_TOKEN`، `DATABASE_URL`، `MONGODB_URI`، `REDIS_URL`، `AMQP_URL`، `AWS_ACCESS_KEY_ID`، `AWS_SECRET_ACCESS_KEY`، `AWS_SESSION_TOKEN`، `AZURE_CLIENT_ID`، `AZURE_CLIENT_SECRET`)؛ همچنین متغیرهای محیطی معمول پراکسی و مختص سرور (`HTTP_PROXY`، `*_API_KEY` سفارشی و غیره) را. سایر کلیدهای `AWS_*` مانند `AWS_CONFIG_FILE` و `AWS_SHARED_CREDENTIALS_FILE` مسدود باقی می‌مانند، زیرا به‌جای حمل مستقیم مقدار اعتبارنامه، به فایل‌های اعتبارنامه اشاره می‌کنند.

اگر سرور MCP واقعاً به یکی از متغیرهای مسدودشده نیاز دارد، آن را به‌جای بخش `env` سرور stdio، روی فرایند میزبان Gateway تنظیم کنید.
</Warning>

### انتقال SSE / HTTP

از طریق HTTP Server-Sent Events به یک سرور MCP راه دور متصل می‌شود.

| فیلد                       | توضیحات                                                      |
| --------------------------- | ---------------------------------------------------------------- |
| `url`                       | نشانی HTTP یا HTTPS سرور راه دور (الزامی)                |
| `headers`                   | نگاشت اختیاری کلید-مقدار سرآیندهای HTTP (برای مثال توکن‌های احراز هویت) |
| `connectionTimeoutMs`       | مهلت زمانی اتصال هر سرور برحسب ms (اختیاری)                   |
| `requestTimeoutMs`          | مهلت زمانی درخواست MCP هر سرور برحسب میلی‌ثانیه                   |
| `auth: "oauth"`             | استفاده از اعتبارنامه‌های OAuth مربوط به MCP که با `openclaw mcp login` ذخیره شده‌اند          |
| `sslVerify`                 | فقط برای نقاط پایانی خصوصی و صراحتاً مورداعتماد HTTPS روی false تنظیم شود    |
| `clientCert` / `clientKey`  | مسیرهای گواهی و کلید کلاینت mTLS                            |
| `supportsParallelToolCalls` | نشان می‌دهد فراخوانی‌های هم‌زمان برای این سرور ایمن هستند              |

نمونه:

```json
{
  "mcp": {
    "servers": {
      "remote-tools": {
        "url": "https://mcp.example.com",
        "auth": "oauth",
        "requestTimeoutMs": 20000,
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

مقادیر حساس در `url` (اطلاعات کاربر) و `headers` در گزارش‌ها و خروجی وضعیت پوشانده می‌شوند. هنگامی که ورودی‌های ظاهراً حساس `headers` یا `env` حاوی مقادیر صریح باشند، `openclaw mcp doctor` هشدار می‌دهد تا اپراتورها بتوانند آن مقادیر را از پیکربندی ثبت‌شده در مخزن خارج کنند.

### گردش‌کار OAuth

OAuth برای سرورهای HTTP MCP است که گردش‌کار OAuth مربوط به MCP را اعلام می‌کنند. هنگامی که `auth: "oauth"` فعال است، سرآیندهای ثابت `Authorization` برای آن سرور نادیده گرفته می‌شوند. اعتبارنامه‌های ذخیره‌شده با `openclaw mcp login` با MCP تعبیه‌شده، اجراکننده‌های CLI و app-server محلی Codex کار می‌کنند.

نشست‌های بومی OAuth مربوط به MCP در پایگاه‌داده SQLite مشترک و فقط در اختیار مالک در `<state-dir>/state/openclaw.sqlite` (`mcp_oauth_stores`) قرار دارند. ردیف می‌تواند حاوی توکن‌های دسترسی و نوسازی، اسرار ثبت پویای کلاینت، فراداده اکتشاف و تأییدکننده موقت PKCE باشد. نوسازی، ورود و خروج از همان اجاره SQLite استفاده می‌کنند، بنابراین فرایندهای موازی OpenClaw نمی‌توانند یک توکن نوسازی را مصرف کنند یا نشست خارج‌شده را دوباره زنده کنند.

ارتقا از مخزن منسوخ‌شده `<state-dir>/mcp-oauth/*.json` فقط توسط `openclaw doctor --fix` انجام می‌شود. کد زمان اجرا هرگز آن فایل‌ها را نمی‌خواند، در آن‌ها نمی‌نویسد یا به آن‌ها بازنمی‌گردد.

تا زمانی که اعتبارنامه‌ها در دسترس باشند، OpenClaw به‌جای شکست نوبت عامل، فقط همان سرور MCP را از زمان اجرای عامل حذف می‌کند. سپس اپراتور یا عاملی با دسترسی پوسته می‌تواند `openclaw mcp login <name>` را اجرا کند و در نوبت بعدی از سرور استفاده کند.

اگر سروری یک توکن را با `insufficient_scope` رد کند، OpenClaw دامنه درخواستی را حفظ می‌کند و به‌جای تکرار نوسازی‌ای که نمی‌تواند دامنه جدیدی اعطا کند، `openclaw mcp login <name>` را درخواست می‌کند. آن ورود، یک درخواست مجوزدهی جدید آغاز می‌کند و توکن پیشین را تا زمان ذخیره اعتبارنامه‌های جایگزین نگه می‌دارد.

هنگامی که یک سرویس MCP راه دور از قبل بر یک پروفایل احراز هویت جداگانه OpenClaw با قابلیت نوسازی تکیه دارد، می‌توانید به‌صورت اختیاری `oauth.authProfileId` را تنظیم کنید. OpenClaw پیش از نگاشت زمان اجرا، هرکدام از منابع اعتبارنامه را نوسازی می‌کند و فقط توکن دسترسی فعلی را به کلاینت پایین‌دستی MCP می‌دهد.

<Steps>
  <Step title="ذخیره سرور">
    سرور را با `auth: "oauth"` و هرگونه فراداده اختیاری OAuth اضافه یا به‌روزرسانی کنید.

    ```bash
    openclaw mcp set docs '{"url":"https://mcp.example.com/mcp","transport":"streamable-http","auth":"oauth","oauth":{"scope":"docs.read"}}'
    ```

    برای توکن حامل مبتنی بر پروفایل احراز هویت، اتصال پروفایل را ذخیره کنید:

    ```bash
    openclaw mcp set docs '{"url":"https://mcp.example.com/mcp","transport":"streamable-http","auth":"oauth","oauth":{"authProfileId":"docs:mcp"}}'
    ```

  </Step>
  <Step title="شروع ورود">
    برای ایجاد درخواست مجوز، فرمان ورود را اجرا کنید.

    ```bash
    openclaw mcp login docs
    ```

    OpenClaw نشانی URL مجوز را نمایش می‌دهد و وضعیت موقت تأییدکننده OAuth را در SQLite مشترک ذخیره می‌کند.

  </Step>
  <Step title="تکمیل با کد">
    پس از تأیید در مرورگر، کد بازگردانده‌شده را به OpenClaw بدهید.

    ```bash
    openclaw mcp login docs --code abc123
    ```

  </Step>
  <Step title="بررسی مجوز">
    برای تأیید وجود توکن‌ها و اینکه به مجوز اضافی نیاز ندارند، از وضعیت یا doctor استفاده کنید. اگر وضعیت `authorization-required` را گزارش کرد یا doctor مجوز اضافی درخواست کرد، `openclaw mcp login <name>` را دوباره اجرا کنید.

    ```bash
    openclaw mcp status --verbose
    openclaw mcp doctor docs --probe
    ```

  </Step>
  <Step title="پاک‌کردن اطلاعات اعتبارسنجی">
    خروج، اطلاعات اعتبارسنجی ذخیره‌شده OAuth را حذف می‌کند، اما تعریف ذخیره‌شده سرور را نگه می‌دارد.

    ```bash
    openclaw mcp logout docs
    ```

  </Step>
</Steps>

اگر ارائه‌دهنده توکن‌ها را تعویض کرد یا وضعیت مجوز گیر کرد، `openclaw mcp logout <name>` را اجرا کنید، سپس `login` را تکرار کنید. `logout` می‌تواند اطلاعات اعتبارسنجی یک سرور HTTP ذخیره‌شده را حتی پس از حذف `auth: "oauth"` از پیکربندی پاک کند، به‌شرط آنکه نام و URL سرور همچنان ورودی مخزن اطلاعات اعتبارسنجی را مشخص کنند.

### انتقال HTTP جریانی

`streamable-http` در کنار `sse` و `stdio` یک گزینه انتقال اضافی است. این گزینه برای ارتباط دوطرفه با سرورهای MCP راه‌دور از جریان‌سازی HTTP استفاده می‌کند.

| فیلد                        | توضیحات                                                                                 |
| --------------------------- | -------------------------------------------------------------------------------------- |
| `url`                       | نشانی URL مبتنی بر HTTP یا HTTPS سرور راه‌دور (الزامی)                                 |
| `transport`                 | برای انتخاب این انتقال روی `"streamable-http"` تنظیم کنید؛ در صورت حذف، OpenClaw از `sse` استفاده می‌کند |
| `headers`                   | نگاشت اختیاری کلید-مقدار سرآیندهای HTTP (برای مثال توکن‌های احراز هویت)                |
| `connectionTimeoutMs`       | مهلت اتصال هر سرور بر حسب ms (اختیاری)                                                  |
| `requestTimeoutMs`          | مهلت درخواست MCP هر سرور بر حسب میلی‌ثانیه                                              |
| `auth: "oauth"`             | استفاده از اطلاعات اعتبارسنجی OAuth مربوط به MCP که توسط `openclaw mcp login` ذخیره شده‌اند |
| `sslVerify`                 | فقط برای نقاط پایانی خصوصی HTTPS که صریحاً مورد اعتمادند، روی false تنظیم کنید          |
| `clientCert` / `clientKey`  | مسیرهای گواهی و کلید کلاینت mTLS                                                         |
| `supportsParallelToolCalls` | نشان می‌دهد فراخوانی‌های هم‌زمان برای این سرور ایمن هستند                               |

پیکربندی OpenClaw از `transport: "streamable-http"` به‌عنوان املای معیار استفاده می‌کند. مقادیر `type: "http"` بومی MCP در CLI، هنگام ذخیره از طریق `openclaw mcp set` پذیرفته می‌شوند و در پیکربندی موجود توسط `openclaw doctor --fix` اصلاح می‌شوند، اما `transport` مقداری است که OpenClaw تعبیه‌شده مستقیماً مصرف می‌کند.

مثال:

```json
{
  "mcp": {
    "servers": {
      "streaming-tools": {
        "url": "https://mcp.example.com/stream",
        "transport": "streamable-http",
        "connectionTimeoutMs": 10000,
        "requestTimeoutMs": 30000,
        "headers": {
          "Authorization": "Bearer <token>"
        }
      }
    }
  }
}
```

<Note>
فرمان‌های رجیستری پل کانال را راه‌اندازی نمی‌کنند. فقط `probe` و `doctor --probe` یک نشست زنده کلاینت MCP را باز می‌کنند تا قابل‌دسترسی‌بودن سرور مقصد را اثبات کنند.
</Note>

## رابط کاربری کنترل

رابط کاربری کنترل در مرورگر، صفحه تنظیمات اختصاصی MCP را در `/settings/mcp` شامل می‌شود؛ مسیر پیشین `/mcp` همچنان به‌عنوان نام مستعار باقی می‌ماند. این صفحه تعداد سرورهای پیکربندی‌شده، خلاصه‌های فعال‌بودن/OAuth/فیلتر، ردیف انتقال هر سرور، کنترل‌های فعال‌سازی/غیرفعال‌سازی، فرمان‌های متداول CLI و ویرایشگری با دامنه محدود برای بخش پیکربندی `mcp` را نمایش می‌دهد.

از این صفحه برای ویرایش‌های اپراتور و فهرست‌برداری سریع استفاده کنید. هنگامی که به اثبات زنده سرور نیاز دارید، از `openclaw mcp doctor --probe` یا `openclaw mcp probe` استفاده کنید.

گردش‌کار اپراتور:

1. رابط کاربری کنترل را باز کنید و **MCP** را انتخاب کنید.
2. کارت‌های خلاصه را برای تعداد کل، سرورهای فعال، OAuth و سرورهای فیلترشده بررسی کنید.
3. از ردیف هر سرور برای مشاهده راهنمای انتقال، احراز هویت، فیلتر، مهلت و فرمان استفاده کنید.
4. هنگامی که می‌خواهید تعریفی را نگه دارید اما آن را از کشف زمان اجرا کنار بگذارید، فعال‌بودن آن را تغییر دهید.
5. برای تغییرات ساختاری مانند سرورهای جدید، سرآیندها، TLS، فراداده OAuth یا فیلترهای ابزار، بخش پیکربندی محدودشده `mcp` را ویرایش کنید.
6. برای فقط ذخیره‌کردن پیکربندی، **Save** را انتخاب کنید یا برای اعمال آن از طریق مسیر پیکربندی Gateway، **Save & Publish** را انتخاب کنید.
7. هنگامی که به اثبات زنده نیاز دارید که سرور ویرایش‌شده راه‌اندازی می‌شود و ابزارها را فهرست می‌کند، `openclaw mcp doctor --probe` را اجرا کنید.

نکات:

- قطعه‌فرمان‌ها نام سرورها را درون نقل‌قول قرار می‌دهند تا نام‌های نامتعارف نیز در پوسته قابل کپی باشند
- مقادیر نمایش‌داده‌شده شبیه URL، اگر حاوی اطلاعات اعتبارسنجی تعبیه‌شده باشند، پیش از رندر پوشانده می‌شوند
- این صفحه به‌تنهایی انتقال‌های MCP را راه‌اندازی نمی‌کند
- بسته به اینکه کدام فرایند مالک کلاینت‌های MCP است، زمان‌های اجرای فعال ممکن است به `openclaw mcp reload`، انتشار پیکربندی Gateway یا راه‌اندازی مجدد فرایند نیاز داشته باشند

## برنامه‌های MCP

OpenClaw می‌تواند ابزارهایی را رندر کند که [افزونه برنامه‌های MCP](https://modelcontextprotocol.io/extensions/apps) پایدار را پیاده‌سازی می‌کنند. برنامه‌ها اختیاری هستند، زیرا HTML آن‌ها از سرور MCP پیکربندی‌شده می‌آید و می‌تواند ابزارها یا منابع قابل‌مشاهده برای برنامه را از همان سرور درخواست کند.

پل میزبان را فعال کنید:

```bash
openclaw config set mcp.apps.enabled true --strict-json
```

پس از تغییر این تنظیم، Gateway را راه‌اندازی مجدد کنید. وقتی فعال باشد، OpenClaw یک شنونده HTTP(S) مختص محیط ایزوله را روی پورت Gateway به‌علاوه یک راه‌اندازی می‌کند (برای Gateway پیش‌فرض، `18790`). رابط کاربری کنترل برنامه‌ها را از آن مبدأ جداگانه بارگذاری می‌کند؛ این شنونده هرگز رابط کاربری کنترل، مسیرهای احرازهویت‌شده Gateway یا داده‌های کاربر را ارائه نمی‌کند.

اتصال‌های مستقیم Gateway به دسترسی به هر دو پورت نیاز دارند. اگر یک پراکسی معکوس یا خاتمه‌دهنده TLS رابط کاربری کنترل را در دسترس قرار می‌دهد، برای برنامه‌ها یک مبدأ عمومی اختصاصی فراهم کنید و فقط همان مبدأ را به شنونده محیط ایزوله پراکسی کنید:

```json5
{
  mcp: {
    apps: {
      enabled: true,
      sandboxOrigin: "https://mcp-apps.example.com",
      sandboxPort: 18790,
    },
  },
}
```

مبدأ محیط ایزوله باید با مبدأ رابط کاربری کنترل متفاوت باشد. محتوای احرازهویت‌شده یا حساس دیگری را روی آن میزبانی نکنید.

برای مثال، دموی رسمی و پایه React را می‌توان به این صورت پیکربندی کرد:

```json5
{
  mcp: {
    apps: { enabled: true },
    servers: {
      "basic-react": {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-basic-react", "--stdio"],
      },
    },
  },
}
```

مرزهای رفتاری و امنیتی:

- OpenClaw فقط هنگامی افزونه `io.modelcontextprotocol/ui` را اعلام می‌کند که برنامه‌ها فعال باشند.
- فقط منابع `ui://` با نوع MIME دقیق `text/html;profile=mcp-app` رندر می‌شوند.
- منابع رابط کاربری به 2 MiB محدود می‌شوند، پشت یک پراکسی دو-iframe در یک مبدأ بیرونی اختصاصی قرار می‌گیرند، در یک مبدأ داخلی ماتِ برنامه بارگذاری می‌شوند و با CSP استخراج‌شده از فراداده منبع محدود می‌شوند.
- ابزارهای مختص برنامه (`_meta.ui.visibility: ["app"]`) در فهرست ابزارهای مدل قرار نمی‌گیرند. برنامه‌ها فقط می‌توانند ابزارهای قابل‌مشاهده برای برنامه را در سرور مالک خود فراخوانی کنند که از سیاست مؤثر ابزار OpenClaw برای اجرایی که نما را ایجاد کرده نیز عبور کنند.
- مجوزهای برنامه مقید به مبدأ، مانند دوربین، میکروفون و موقعیت جغرافیایی، تا زمانی که اسناد داخلی برنامه برای جداسازی میان برنامه‌ها از مبدأهای مات استفاده می‌کنند، اعطا نمی‌شوند.
- HTML برنامه، آرگومان‌های کامل ابزار و نتایج خام در یک اجاره نمای درون‌حافظه‌ای و محدود ده‌دقیقه‌ای نگهداری می‌شوند و روی دیسک نوشته یا در فراداده پیش‌نمایش رونوشت کپی نمی‌شوند. رونوشت فقط یک توصیفگر محدود سرور/ابزار/منبع را که به شناسه فراخوانی اصلی ابزار متصل است ذخیره می‌کند. پس از راه‌اندازی مجدد Gateway، رابط کاربری کنترل می‌تواند آن توصیفگر را در برابر رونوشت نشست احرازهویت‌شده تأیید و منبع `ui://` را دوباره دریافت کند؛ نماهای بازسازی‌شده تا زمانی که یک اجرای تازه مجوزهای فعلی ابزار را برقرار کند، فقط خواندنی هستند.
- در مکالمات کانال، آخرین نمای موفق برنامه در یک نوبت، یک کنش به سبک **بازکردن برنامه** به پاسخ نهایی دستیار اضافه می‌کند. پیام‌های مستقیم Telegram از یک دکمه بومی Mini App استفاده می‌کنند؛ Slack و Discord همان کنش قابل‌حمل را به‌شکل پیوند رندر می‌کنند. کانال‌های دیگر متن اصلی پاسخ را نگه می‌دارند و یک پیوند قابل‌فهم HTTPS به آن می‌افزایند.
- پیوندهای راه‌اندازی کانال فقط زمانی در دسترس‌اند که ارائه Tailscale در Gateway یک مبدأ HTTPS منتشرشده آماده کرده باشد. `gateway.tailscale.mode: "serve"` فقط از tailnet قابل‌دسترسی است؛ `"funnel"` از اینترنت عمومی قابل‌دسترسی است. یک Funnel مدیریت‌شده خارجی که توسط `gateway.tailscale.preserveFunnel` حفظ شده نیز قابل‌دسترسی از اینترنت در نظر گرفته می‌شود. [Tailscale](/fa/gateway/tailscale) را ببینید.
- بلیت‌های راه‌اندازی مات هستند، فقط هنگام ساخت پاسخ نهایی کانال صادر می‌شوند و حداکثر پس از دو دقیقه یا هنگام انقضای اجاره نمای زیربنایی، هرکدام زودتر رخ دهد، منقضی می‌شوند. URL حاوی اطلاعات اعتبارسنجی حامل Gateway، کلیدهای نشست، فراداده نما، HTML برنامه، ورودی ابزار یا نتایج ابزار نیست.
- اگر هیچ مبدأ منتشرشده یا ظرفیت بلیتی در دسترس نباشد، نما یا بلیت منقضی شده باشد، یا انتقال نتواند کنترل‌های بومی را رندر کند، متن اصلی دستیار همچنان در دسترس می‌ماند. رابط کاربری کنترل بوم درون‌خطی موجود برنامه را نگه می‌دارد و کنش راه‌اندازی تکراری دریافت نمی‌کند.
- `openclaw security audit` هنگام فعال‌بودن پل هشدار می‌دهد. وقتی به آن نیاز نیست، با `openclaw config set mcp.apps.enabled false --strict-json` غیرفعالش کنید.

## محدودیت‌های کنونی

این صفحه پل را همان‌گونه که امروز عرضه شده مستند می‌کند.

محدودیت‌های کنونی:

- کشف مکالمه به فراداده موجود مسیر نشست Gateway وابسته است
- هیچ پروتکل push عمومی فراتر از آداپتور ویژه Claude وجود ندارد
- هنوز هیچ ابزار ویرایش پیام یا واکنش وجود ندارد
- انتقال HTTP/SSE/streamable-http به یک سرور راه‌دور واحد متصل می‌شود؛ هنوز upstream چندگانه‌ای وجود ندارد
- `permissions_list_open` فقط تأییدهای مشاهده‌شده در زمان اتصال پل را شامل می‌شود

## مرتبط

- [مرجع CLI](/fa/cli)
- [Pluginها](/fa/cli/plugins)
