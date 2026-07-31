---
read_when:
    - می‌خواهید عامل‌های OpenClaw در حالت Codex از Codex Computer Use استفاده کنند
    - شما در حال انتخاب میان Codex Computer Use، PeekabooBridge و MCP مستقیم cua-driver هستید
    - در حال پیکربندی computerUse برای Plugin همراه Codex هستید
    - در حال عیب‌یابی وضعیت یا نصب استفاده از رایانه در ‎/codex‎ هستید
summary: راه‌اندازی Codex Computer Use برای عامل‌های OpenClaw در حالت Codex
title: استفاده از رایانه با Codex
x-i18n:
    generated_at: "2026-07-27T16:45:28Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6b11d00c74bc2990a4e33b6ffe23209ed76a1e10180ce5950dbb5073ea57ad05
    source_path: plugins/codex-computer-use.md
    workflow: 16
---

Computer Use یک Plugin بومی Codex برای MCP جهت کنترل دسکتاپ محلی است. OpenClaw
برنامه دسکتاپ را در خود جای نمی‌دهد، خودْ اقدامات دسکتاپ را اجرا نمی‌کند و
مجوزهای Codex را دور نمی‌زند. Plugin همراه `codex` فقط app-server مربوط به Codex را آماده می‌کند:
پشتیبانی از Plugin در Codex را فعال می‌کند، Plugin پیکربندی‌شده Computer Use را
پیدا یا نصب می‌کند، در دسترس بودن سرور MCP با نام `computer-use` را بررسی می‌کند و سپس
در نوبت‌های حالت Codex، فراخوانی ابزارهای بومی MCP را به خود Codex می‌سپارد.

وقتی OpenClaw از قبل از هارنس بومی Codex استفاده می‌کند، از این صفحه استفاده کنید. برای
راه‌اندازی خود محیط اجرا، به [هارنس Codex](/fa/plugins/codex-harness) مراجعه کنید.

این با [ابزار رایانه مبتنی بر Node](/fa/nodes/computer-use) داخلی OpenClaw متفاوت است. وقتی قرارداد یکسان عامل باید یک Mac جفت‌شده را کنترل کند، صرف‌نظر از اینکه عامل روی Gateway یا Node دیگری اجرا می‌شود، از ابزار داخلی استفاده کنید. وقتی app-server مربوط به Codex باید مالک نصب محلی MCP، مجوزها و فراخوانی‌های بومی ابزار باشد، از Codex Computer Use استفاده کنید.

## OpenClaw.app و Peekaboo

یکپارچه‌سازی Peekaboo در OpenClaw.app از Codex Computer Use جدا است.
برنامه macOS می‌تواند یک سوکت PeekabooBridge میزبانی کند تا CLI با نام `peekaboo` بتواند از
مجوزهای محلی Accessibility و Screen Recording برنامه برای ابزارهای اتوماسیون خود Peekaboo
دوباره استفاده کند. آن پل Codex Computer Use را نصب یا پراکسی نمی‌کند و
Codex Computer Use نیز از طریق سوکت PeekabooBridge فراخوانی انجام نمی‌دهد.

وقتی می‌خواهید OpenClaw.app میزبانی آگاه از مجوز برای اتوماسیون CLI مربوط به Peekaboo باشد، از
[پل Peekaboo](/fa/platforms/mac/peekaboo) استفاده کنید. وقتی یک عامل OpenClaw در
حالت Codex باید پیش از آغاز نوبت، Plugin بومی MCP با نام `computer-use` متعلق به Codex را
در دسترس داشته باشد، از این صفحه استفاده کنید.

## برنامه iOS

برنامه iOS از Codex Computer Use جدا است. این برنامه سرور MCP مربوط به Codex با نام
`computer-use` را نصب یا پراکسی نمی‌کند و بک‌اند کنترل دسکتاپ نیست.
در عوض، برنامه iOS به‌عنوان یک Node در OpenClaw متصل می‌شود و قابلیت‌های
موبایل را از طریق فرمان‌های Node مانند `canvas.*`، `camera.*`، `screen.*`،
`location.*` و `talk.*` ارائه می‌کند.

وقتی می‌خواهید یک عامل از طریق Gateway یک Node آیفون را هدایت کند، از
[iOS](/fa/platforms/ios) استفاده کنید. وقتی یک عامل در حالت Codex باید دسکتاپ
محلی macOS را از طریق Plugin بومی Computer Use متعلق به Codex کنترل کند، از این صفحه استفاده کنید.

## MCP مستقیم cua-driver

Codex Computer Use تنها راه ارائه کنترل دسکتاپ نیست. اگر می‌خواهید
محیط‌های اجرای مدیریت‌شده توسط OpenClaw مستقیماً درایور TryCua را فراخوانی کنند، به‌جای
جریان بازار اختصاصی Codex، از سرور بالادستی `cua-driver mcp` از طریق رجیستری
MCP متعلق به OpenClaw استفاده کنید.

پس از نصب `cua-driver`، یا فرمان OpenClaw را از آن درخواست کنید:

```bash
cua-driver mcp-config --client openclaw
```

یا سرور stdio را مستقیماً ثبت کنید:

```bash
openclaw mcp set cua-driver '{"command":"cua-driver","args":["mcp"]}'
```

این مسیر سطح ابزار MCP بالادستی، از جمله شِماهای درایور
و پاسخ‌های ساخت‌یافته MCP، را دست‌نخورده نگه می‌دارد. وقتی می‌خواهید درایور CUA
به‌عنوان یک سرور عادی MCP در OpenClaw در دسترس باشد، از آن استفاده کنید. وقتی
app-server مربوط به Codex باید مالک نصب Plugin، بارگذاری مجدد MCP
و فراخوانی‌های بومی ابزار در نوبت‌های حالت Codex باشد، از راه‌اندازی Codex Computer Use در
این صفحه استفاده کنید.

درایور CUA بیلدهای پیش‌انتشار را برای macOS، Windows (x64 و ARM64) و
Linux (x64 و ARM64، رده پیش‌نمایش) ارائه می‌کند. همچنان به مجوزهای محلی
سیستم‌عامل که برنامه درخواست می‌کند نیاز دارد، مانند Accessibility و Screen Recording در
macOS. OpenClaw برنامه `cua-driver` را نصب نمی‌کند، آن مجوزها را اعطا نمی‌کند و
مدل ایمنی درایور بالادستی را دور نمی‌زند.

## راه‌اندازی سریع

وقتی در نوبت‌های حالت Codex باید Computer Use
پیش از آغاز یک رشته در دسترس باشد، `plugins.entries.codex.config.computerUse` را تنظیم کنید. `autoInstall: true`
Computer Use را فعال می‌کند و به OpenClaw اجازه می‌دهد پیش از نوبت آن را نصب یا دوباره فعال کند:

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          computerUse: {
            autoInstall: true,
          },
        },
      },
    },
  },
  agents: {
    defaults: {
      model: "openai/gpt-5.6-sol",
    },
  },
}
```

با این پیکربندی، OpenClaw پیش از هر نوبت حالت Codex،
app-server مربوط به Codex را بررسی می‌کند. اگر Computer Use موجود نباشد اما app-server مربوط به Codex از قبل
یک بازار قابل‌نصب را کشف کرده باشد، OpenClaw از app-server مربوط به Codex می‌خواهد Plugin را
نصب یا دوباره فعال کند و سرورهای MCP را دوباره بارگذاری کند. پیش از راه‌اندازی یک
app-server ایزوله Codex در macOS، نصب خودکار همچنین برنامه سرویس رسمی و امضاشده
Computer Use را از بسته برنامه دسکتاپ انتخاب‌شده به پوشه
`computer-use` در خانه Codex کپی می‌کند، مشروط بر اینکه کلاینت بومی موجود نباشد.
در macOS، وقتی هیچ
بازار منطبقی ثبت نشده و یک بسته استاندارد برنامه دسکتاپ وجود دارد، OpenClaw
همچنین تلاش می‌کند بازار همراه Codex را از
`/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled` ثبت کند و
`/Applications/Codex.app/Contents/Resources/plugins/openai-bundled` را
به‌عنوان مسیر جایگزین برای نصب‌های مستقل قدیمی نگه می‌دارد. اگر راه‌اندازی همچنان نتواند
سرور MCP را در دسترس قرار دهد، نوبت پیش از آغاز رشته شکست می‌خورد.
شکست‌های سخت‌گیرانه آمادگی، شکست‌های پیش‌بررسی هارنس هستند؛ بنابراین بازگشت به مدل جایگزین،
توالی یکسان آمادگی محلی را برای هر نامزد مدل تکرار نمی‌کند.

پس از تغییر پیکربندی Computer Use، اگر یک رشته موجود Codex از قبل آغاز شده است، پیش از آزمایش در
گفت‌وگوی تحت‌تأثیر از `/new` یا `/reset` استفاده کنید.

در macOS، راه‌اندازی مدیریت‌شده Computer Use ابتدا باینری برنامه دسکتاپ در
`/Applications/ChatGPT.app/Contents/Resources/codex` را ترجیح می‌دهد و سپس برای نصب‌های مستقل
قدیمی به `/Applications/Codex.app/Contents/Resources/codex` بازمی‌گردد.
این موضوع برای فرمان‌های یک‌باره وضعیت و نصب Computer Use که کلاینت خود را
راه‌اندازی می‌کنند نیز صدق می‌کند. این کار کنترل دسکتاپ را زیر نظر بسته برنامه‌ای نگه می‌دارد
که مالک مجوزهای محلی macOS است. اگر برنامه دسکتاپ نصب نشده باشد،
OpenClaw به باینری مدیریت‌شده Codex که کنار Plugin نصب شده است بازمی‌گردد.
نوبت‌های مدیریت‌شده عادی Codex با خانه ایزوله پیش‌فرض عامل، ابتدا آن بسته سنجاق‌شده را ترجیح می‌دهند
تا یک برنامه دسکتاپ قدیمی نتواند پشتیبانی مدل فعلی را تحت‌الشعاع قرار دهد.
خانه‌های در محدوده کاربر همچنان دسکتاپ را در اولویت می‌گذارند، زیرا می‌توانند وضعیت بومی
Computer Use را بارگذاری کنند. یک خانه ایزوله عامل که پیکربندی مؤثر Codex در آن
Computer Use را فعال می‌کند نیز همچنان دسکتاپ را در اولویت می‌گذارد. پیکربندی صریح
`appServer.command` یا `OPENCLAW_CODEX_APP_SERVER_BIN` همچنان
این انتخاب مدیریت‌شده را لغو می‌کند.

OpenClaw خواندن پیکربندی بومی Codex و نصب Computer Use را
درون یک Gateway در حال اجرا به‌صورت ترتیبی انجام می‌دهد. یک فرایند جداگانه Codex یا Gateway دیگری
در محدوده آن حصار نیست. پس از تغییر پیکربندی بومی Plugin مربوط به Codex خارج از
Gateway، پیش از اتکا به انتخاب جدید، Gateway را دوباره راه‌اندازی کنید و یک گفت‌وگوی جدید آغاز کنید.

## فرمان‌ها

از فرمان‌های `/codex computer-use` در هر سطح گفت‌وگویی که
سطح فرمان Plugin با نام `codex` در دسترس است استفاده کنید. این‌ها فرمان‌های گفت‌وگو/محیط اجرای
OpenClaw هستند، نه زیرفرمان‌های CLI با نام `openclaw codex ...`:

```text
/codex computer-use status
/codex computer-use install
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
/codex computer-use install --marketplace <name>
```

`status` اقدام پیش‌فرض و فقط‌خواندنی است: منبع بازار
اضافه نمی‌کند، Plugin نصب نمی‌کند و پشتیبانی از Plugin در Codex را فعال نمی‌کند. اگر هیچ پیکربندی‌ای
Computer Use را فعال نکند، `status` حتی پس از یک فرمان نصب یک‌باره نیز می‌تواند
وضعیت غیرفعال را گزارش کند.

`install` پشتیبانی از Plugin در app-server مربوط به Codex را فعال می‌کند، در صورت نیاز یک
منبع بازار پیکربندی‌شده اضافه می‌کند، Plugin پیکربندی‌شده را
از طریق app-server مربوط به Codex نصب یا دوباره فعال می‌کند، سرورهای MCP را دوباره بارگذاری می‌کند و بررسی می‌کند که سرور MCP
ابزارها را ارائه می‌دهد. چون نصب منابع قابل‌اعتماد میزبان را تغییر می‌دهد،
فقط مالک یا یک کلاینت Gateway با نام `operator.admin` می‌تواند `install` را اجرا کند. دیگر
فرستندگان مجاز می‌توانند همچنان از فرمان فقط‌خواندنی `status`
استفاده کنند، از جمله همراه با بازنویسی‌ها.

نسخه‌های قدیمی بازنویسی‌های یک‌باره هویت با نام‌های `--plugin`، `--server` و `--mcp-server`
را می‌پذیرفتند. در عوض، `computerUse.pluginName` و
`computerUse.mcpServerName` را به‌صورت پایدار پیکربندی کنید. وقتی از یک پرچم قدیمی هویت
استفاده شود، فرمان تنظیم دقیق قابل‌ذخیره را مشخص می‌کند و
اقدام درخواستی به‌همراه هر پرچم بازار پشتیبانی‌شده را در راهنمای مهاجرت خود تکرار می‌کند.

## گزینه‌های بازار

OpenClaw از همان API مربوط به app-server استفاده می‌کند که خود Codex ارائه می‌دهد.
فیلدهای بازار تعیین می‌کنند Codex باید `computer-use` را کجا پیدا کند.

| فیلد                 | زمان استفاده                                                     | پشتیبانی نصب                                                |
| -------------------- | --------------------------------------------------------------- | -------------------------------------------------------- |
| بدون فیلد بازار | می‌خواهید app-server مربوط به Codex از بازارهایی استفاده کند که از قبل می‌شناسد. | بله، وقتی app-server یک بازار محلی برمی‌گرداند.        |
| `marketplaceSource`  | یک منبع بازار Codex دارید که app-server می‌تواند اضافه کند.         | بله، برای `/codex computer-use install` صریح.         |
| `marketplacePath`    | از قبل مسیر فایل بازار محلی روی میزبان را می‌دانید.   | بله، برای نصب صریح و نصب خودکار هنگام آغاز نوبت.   |
| `marketplaceName`    | می‌خواهید یکی از بازارهای از قبل ثبت‌شده را با نام انتخاب کنید.  | فقط وقتی بازار انتخاب‌شده مسیر محلی داشته باشد. |

خانه‌های تازه Codex ممکن است برای آماده‌سازی بازارهای رسمی خود به کمی
زمان نیاز داشته باشند. هنگام نصب، OpenClaw تا
`marketplaceDiscoveryTimeoutMs` میلی‌ثانیه (پیش‌فرض 60 ثانیه) `plugin/list` را پیمایش می‌کند.

اگر چند بازار شناخته‌شده شامل Computer Use باشند، OpenClaw ابتدا
`openai-bundled`، سپس `openai-curated` و بعد `local` را ترجیح می‌دهد. تطابق‌های ناشناخته و
مبهم به‌صورت بسته شکست می‌خورند و از شما می‌خواهند `marketplaceName` یا
`marketplacePath` را تنظیم کنید.

## بازار همراه macOS

بیلدهای فعلی دسکتاپ ChatGPT، Computer Use را در اینجا همراه دارند؛ بیلدهای مستقل قدیمی
دسکتاپ Codex از همان چیدمان زیر `Codex.app` استفاده می‌کنند:

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled/plugins/computer-use
```

وقتی `computerUse.autoInstall` برابر true باشد و هیچ بازار شامل
`computer-use` ثبت نشده باشد، OpenClaw تلاش می‌کند نخستین ریشه استاندارد
بازار همراه موجود را اضافه کند:

```text
/Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
/Applications/Codex.app/Contents/Resources/plugins/openai-bundled
```

همچنین می‌توانید آن را به‌طور صریح از طریق پوسته با Codex ثبت کنید:

```bash
codex plugin marketplace add /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

اگر از مسیر غیراستاندارد برنامه Codex استفاده می‌کنید، `/codex computer-use install
--source <marketplace-root>` را یک‌بار اجرا کنید یا `computerUse.marketplacePath` را روی یک
مسیر فایل بازار محلی تنظیم کنید. فقط وقتی مسیر فایل JSON بازار را دارید از `--marketplace-path` استفاده کنید،
نه ریشه بازار همراه را.

### کش مشترک Plugin

مقدار پیش‌فرض `pluginCacheMode: "independent"` هر خانه Codex و
کش Plugin آن را مدیریت‌نشده باقی می‌گذارد. `pluginCacheMode: "shared"` را تنظیم کنید تا Plugin همراه
Computer Use پیش از راه‌اندازی app-server در کش قابل‌کشف Plugin مربوط به خانه فعال Codex
کپی شود. حالت مشترک نسخه‌های قدیمی‌تر کش‌شده را حفظ می‌کند، زیرا
کلاینت‌های در حال اجرای Codex ممکن است همچنان به پوشه‌های نسخه‌بندی‌شده Plugin خود ارجاع دهند؛
یک کپی جایگزین ناموفق نیز کش فعال را حفظ می‌کند. پیکربندی صریح
`marketplaceName` یا `marketplacePath` این
همگام‌سازی را غیرفعال می‌کند تا OpenClaw آن انتخاب را لغو نکند.

## محدودیت کاتالوگ راه دور

app-server مربوط به Codex می‌تواند ورودی‌های کاتالوگ فقط‌راه‌دور را فهرست و بخواند، اما در حال حاضر
از `plugin/install` راه دور پشتیبانی نمی‌کند. این یعنی `marketplaceName`
می‌تواند برای بررسی وضعیت یک بازار فقط‌راه‌دور را انتخاب کند، اما نصب و
فعال‌سازی مجدد همچنان به یک بازار محلی از طریق `marketplaceSource` یا
`marketplacePath` نیاز دارند.

اگر وضعیت اعلام می‌کند Plugin در یک بازار راه دور Codex موجود است اما
نصب راه دور پشتیبانی نمی‌شود، نصب را با یک منبع یا مسیر محلی اجرا کنید:

```text
/codex computer-use install --source <marketplace-source>
/codex computer-use install --marketplace-path <path>
```

## مرجع پیکربندی

| فیلد                           | پیش‌فرض        | مفهوم                                                                        |
| ------------------------------- | -------------- | ------------------------------------------------------------------------------ |
| `enabled`                       | استنباط‌شده       | Computer Use را الزامی می‌کند. وقتی فیلد دیگری از Computer Use تنظیم شده باشد، مقدار پیش‌فرض true است. |
| `autoInstall`                   | false          | کلاینت بومی را آماده می‌کند و در آغاز نوبت، Plugin را نصب یا دوباره فعال می‌کند. |
| `marketplaceDiscoveryTimeoutMs` | 60000          | مدت‌زمانی که نصب برای کشف مارکت‌پلیس توسط app-server برنامه Codex منتظر می‌ماند.             |
| `liveTestTimeoutMs`             | 60000          | مهلت زمانی رشته موقت آمادگی و درخواست‌های پاک‌سازی آن.           |
| `toolCallTimeoutMs`             | 60000          | مهلت زمانی فراخوانی ابزار آمادگی `list_apps` در Computer Use.                  |
| `healthCheckEnabled`            | false          | هنگامی که کلاینت app-server مالک فعال است، کاوش‌های دوره‌ای آمادگی را اجرا می‌کند.    |
| `healthCheckIntervalMinutes`    | 60             | تناوب کاوش؛ مقادیر پذیرفته‌شده 30، 60، 120 یا 240 دقیقه هستند.                |
| `pluginCacheMode`               | `independent`  | برای تازه‌سازی کش Codex-home از Plugin دسکتاپ همراه، از `shared` استفاده می‌کند.  |
| `strictReadiness`               | false          | در صورت شکست کاوش زنده، به‌جای ادامه همراه با هشدار، راه‌اندازی را متوقف می‌کند.      |
| `autoRepair`                    | false          | فرایندهای فرزند MCP قدیمی و محدود به دامنه Computer Use را می‌بندد و کاوش ناموفق را یک‌بار دیگر امتحان می‌کند.     |
| `marketplaceSource`             | تنظیم‌نشده          | رشته منبعی که به `marketplace/add` در app-server برنامه Codex ارسال می‌شود.                    |
| `marketplacePath`               | تنظیم‌نشده          | مسیر فایل محلی مارکت‌پلیس Codex که Plugin را در بر دارد.                       |
| `marketplaceName`               | تنظیم‌نشده          | نام ثبت‌شده مارکت‌پلیس Codex برای انتخاب.                                   |
| `pluginName`                    | `computer-use` | نام Plugin در مارکت‌پلیس Codex.                                                 |
| `mcpServerName`                 | `computer-use` | نام سرور MCP که Plugin نصب‌شده ارائه می‌کند.                               |

نصب خودکار در آغاز نوبت عمداً مقادیر پیکربندی‌شده `marketplaceSource`
را نمی‌پذیرد. افزودن منبع جدید یک عملیات صریح راه‌اندازی است؛ بنابراین یک‌بار از
`/codex computer-use install --source <marketplace-source>` استفاده کنید، سپس اجازه دهید
`autoInstall` فعال‌سازی‌های مجدد آینده را از مارکت‌پلیس‌های محلی کشف‌شده انجام دهد.
نصب خودکار در آغاز نوبت می‌تواند از `marketplacePath` پیکربندی‌شده استفاده کند، زیرا آن
از قبل مسیری محلی روی میزبان است.

هر فیلد یک جایگزین متغیر محیطی نیز می‌پذیرد که وقتی کلید پیکربندی
متناظر تنظیم‌نشده باشد، بررسی می‌شود:

| فیلد                           | متغیر محیطی                                                        |
| ------------------------------- | -------------------------------------------------------------- |
| `enabled`                       | `OPENCLAW_CODEX_COMPUTER_USE`                                  |
| `autoInstall`                   | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_INSTALL`                     |
| `marketplaceDiscoveryTimeoutMs` | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_DISCOVERY_TIMEOUT_MS` |
| `liveTestTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_LIVE_TEST_TIMEOUT_MS`             |
| `toolCallTimeoutMs`             | `OPENCLAW_CODEX_COMPUTER_USE_TOOL_CALL_TIMEOUT_MS`             |
| `healthCheckEnabled`            | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_ENABLED`             |
| `healthCheckIntervalMinutes`    | `OPENCLAW_CODEX_COMPUTER_USE_HEALTH_CHECK_INTERVAL_MINUTES`    |
| `pluginCacheMode`               | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_CACHE_MODE`                |
| `strictReadiness`               | `OPENCLAW_CODEX_COMPUTER_USE_STRICT_READINESS`                 |
| `autoRepair`                    | `OPENCLAW_CODEX_COMPUTER_USE_AUTO_REPAIR`                      |
| `marketplaceSource`             | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_SOURCE`               |
| `marketplacePath`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_PATH`                 |
| `marketplaceName`               | `OPENCLAW_CODEX_COMPUTER_USE_MARKETPLACE_NAME`                 |
| `pluginName`                    | `OPENCLAW_CODEX_COMPUTER_USE_PLUGIN_NAME`                      |
| `mcpServerName`                 | `OPENCLAW_CODEX_COMPUTER_USE_MCP_SERVER_NAME`                  |

## مواردی که OpenClaw بررسی می‌کند

OpenClaw یک دلیل پایدار راه‌اندازی را به‌صورت داخلی گزارش می‌کند و وضعیت
قابل‌مشاهده برای کاربر را برای گفت‌وگو قالب‌بندی می‌کند:

| دلیل                       | مفهوم                                                | گام بعدی                                     |
| ---------------------------- | ------------------------------------------------------ | --------------------------------------------- |
| `disabled`                   | `computerUse.enabled` به false تبدیل شده است.               | `enabled` یا فیلد دیگری از Computer Use را تنظیم کنید.  |
| `marketplace_missing`        | هیچ مارکت‌پلیس منطبقی در دسترس نبود.                 | منبع، مسیر یا نام مارکت‌پلیس را پیکربندی کنید.  |
| `plugin_not_installed`       | مارکت‌پلیس وجود دارد، اما Plugin نصب نشده است.   | نصب را اجرا کنید یا `autoInstall` را فعال کنید.          |
| `plugin_disabled`            | Plugin نصب شده، اما در پیکربندی Codex غیرفعال است.      | برای فعال‌سازی مجدد آن، نصب را اجرا کنید.                  |
| `remote_install_unsupported` | مارکت‌پلیس انتخاب‌شده فقط راه‌دور است.                   | از `marketplaceSource` یا `marketplacePath` استفاده کنید. |
| `mcp_missing`                | Plugin فعال است، اما سرور MCP در دسترس نیست.  | Computer Use در Codex و مجوزهای سیستم‌عامل را بررسی کنید.  |
| `ready`                      | Plugin و ابزارهای MCP در دسترس هستند.                    | نوبت حالت Codex را آغاز کنید.                    |
| `check_failed`               | هنگام بررسی وضعیت، یک درخواست app-server برنامه Codex ناموفق بود. | اتصال و گزارش‌های app-server را بررسی کنید.       |
| `auto_install_blocked`       | راه‌اندازی آغاز نوبت مستلزم افزودن منبع جدید است.       | ابتدا نصب صریح را اجرا کنید.                   |

خروجی گفت‌وگو شامل وضعیت Plugin، وضعیت سرور MCP، مارکت‌پلیس،
ابزارهای موجود و پیام مشخص مربوط به گام ناموفق راه‌اندازی است.

## مجوزهای macOS

این مسیر Computer Use متعلق به Codex روی macOS اجرا می‌شود؛ جایی که ممکن است سرور MCP پیش از بررسی یا کنترل برنامه‌ها
به مجوزهای محلی سیستم‌عامل نیاز داشته باشد. (برای کنترل دسکتاپ چندسکویی روی میزبان‌های Node در Windows و Linux، به
[اجراکننده cua-computer](/fa/nodes/computer-use#windows-and-linux-experimental-via-cua-driver) مراجعه کنید.)
اگر OpenClaw اعلام کرد Computer Use نصب شده اما سرور MCP در دسترس نیست،
ابتدا راه‌اندازی Computer Use در سمت Codex را بررسی کنید:

- app-server برنامه Codex روی همان میزبانی اجرا می‌شود که کنترل دسکتاپ باید
  در آن انجام شود.
- Plugin مربوط به Computer Use در پیکربندی Codex فعال است.
- سرور MCP با نام `computer-use` در وضعیت MCP برنامه app-server در Codex نمایش داده می‌شود.
- macOS مجوزهای لازم را به برنامه کنترل دسکتاپ اعطا کرده است.
- نشست فعلی میزبان می‌تواند به دسکتاپ تحت کنترل دسترسی پیدا کند.

وقتی `computerUse.enabled` مقدار true دارد، OpenClaw عمداً با شکست بسته می‌شود. یک
نوبت حالت Codex نباید بدون ابزارهای بومی دسکتاپی که پیکربندی الزامی کرده است، بی‌سروصدا ادامه یابد.

## عیب‌یابی

**وضعیت می‌گوید نصب نشده است.** `/codex computer-use install` را اجرا کنید. اگر
مارکت‌پلیس کشف نشد، `--source` یا `--marketplace-path` را ارسال کنید.

**وضعیت می‌گوید نصب شده اما غیرفعال است.** دوباره `/codex computer-use install`
را اجرا کنید. نصب app-server برنامه Codex، پیکربندی Plugin را دوباره با وضعیت فعال می‌نویسد.

**وضعیت می‌گوید نصب راه‌دور پشتیبانی نمی‌شود.** از یک منبع یا مسیر
مارکت‌پلیس محلی استفاده کنید. ورودی‌های کاتالوگ که فقط راه‌دور هستند را می‌توان بررسی کرد، اما
نمی‌توان آن‌ها را از طریق API فعلی app-server نصب کرد.

**وضعیت می‌گوید سرور MCP در دسترس نیست.** نصب را یک‌بار دیگر اجرا کنید تا سرورهای MCP
دوباره بارگذاری شوند. اگر همچنان در دسترس نبود، برنامه Computer Use در Codex،
وضعیت MCP در app-server برنامه Codex یا مجوزهای macOS را اصلاح کنید.

**وضعیت یا یک کاوش روی `computer-use.list_apps` به مهلت زمانی می‌رسد.** Plugin و
سرور MCP موجودند، اما پل محلی Computer Use پاسخ نداده است.
از Codex Computer Use خارج شوید یا آن را راه‌اندازی مجدد کنید؛ در صورت نیاز Codex Desktop را دوباره اجرا کنید، سپس
در یک نشست تازه OpenClaw دوباره تلاش کنید. اگر میزبان پیش‌تر Computer Use را
از طریق app-server مدیریت‌شده قدیمی Codex اجرا کرده است، Plugin نصب‌شده را از
مارکت‌پلیس همراه دسکتاپ تازه‌سازی کنید (برای نصب‌های مستقل
دسکتاپ Codex از مسیر `Codex.app` استفاده کنید):

```text
/codex computer-use install --source /Applications/ChatGPT.app/Contents/Resources/plugins/openai-bundled
```

**یک ابزار Computer Use می‌گوید `Native hook relay unavailable`.** هوک
ابزار بومی Codex نتوانسته از طریق پل محلی یا مسیر جایگزین Gateway به یک رله فعال OpenClaw دسترسی پیدا کند.
یک نشست تازه OpenClaw را با `/new`
یا `/reset` آغاز کنید. اگر یک‌بار کار کرد و سپس در فراخوانی بعدی ابزار دوباره شکست خورد،
`/new` فقط تلاش فعلی را پاک می‌کند؛ app-server برنامه Codex یا
OpenClaw Gateway را راه‌اندازی مجدد کنید تا رشته‌های قدیمی و ثبت‌های هوک حذف شوند، سپس
در یک نشست تازه دوباره تلاش کنید.

**نصب خودکار در آغاز نوبت یک منبع را نمی‌پذیرد.** این رفتار عمدی است. ابتدا منبع را
با `/codex computer-use install --source
<marketplace-source>` صریح اضافه کنید، سپس نصب خودکار در آغاز نوبت در آینده می‌تواند از
مارکت‌پلیس محلی کشف‌شده استفاده کند.

## مرتبط

- [مهار Codex](/fa/plugins/codex-harness)
- [پل Peekaboo](/fa/platforms/mac/peekaboo)
- [برنامه iOS](/fa/platforms/ios)
