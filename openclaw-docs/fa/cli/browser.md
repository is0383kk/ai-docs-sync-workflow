---
read_when:
    - از `openclaw browser` استفاده می‌کنید و برای کارهای رایج مثال می‌خواهید
    - می‌خواهید مرورگری را که روی دستگاه دیگری اجرا می‌شود، از طریق یک میزبان Node کنترل کنید
    - می‌خواهید از طریق Chrome MCP به Chrome محلی خود که در آن وارد حساب شده‌اید متصل شوید
summary: مرجع CLI برای `openclaw browser` (چرخهٔ عمر، پروفایل‌ها، زبانه‌ها، کنش‌ها، وضعیت و اشکال‌زدایی)
title: مرورگر
x-i18n:
    generated_at: "2026-07-27T16:18:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 62eb41248cda87cef96be7b0dfe3e0d36a9d3e1ee55c165bd8e3efd68d1e9a5e
    source_path: cli/browser.md
    workflow: 16
---

# `openclaw browser`

سطح کنترل مرورگر OpenClaw را مدیریت کنید و عملیات مرورگر را اجرا کنید: چرخه حیات، پروفایل‌ها، برگه‌ها، عکس‌های فوری، نماگرفت‌ها، پیمایش، ورودی، شبیه‌سازی وضعیت و اشکال‌زدایی.

مرتبط: [ابزار مرورگر](/fa/tools/browser)

## پرچم‌های متداول

- `--url <gatewayWsUrl>`: نشانی WebSocket مربوط به Gateway (پیش‌فرض از پیکربندی گرفته می‌شود).
- `--token <token>`: توکن Gateway (در صورت نیاز).
- `--timeout <ms>`: مهلت زمانی درخواست برحسب میلی‌ثانیه (پیش‌فرض: `30000`).
- `--expect-final`: منتظر پاسخ نهایی Gateway بمانید.
- `--browser-profile <name>`: یک پروفایل مرورگر انتخاب کنید (پیش‌فرض: `openclaw`، یا `browser.defaultProfile`).
- `--json`: خروجی قابل‌خواندن برای ماشین (در موارد پشتیبانی‌شده). این گزینه در سطح مرورگر است، بنابراین
  برای جلوگیری از ابهام، آن را پیش از زیرفرمان قرار دهید، مانند
  `openclaw browser --json status`. قرار دادن آن در انتها، مانند
  `openclaw browser status --json`، نیز هنگامی کار می‌کند که فرمان فرزند انتخاب‌شده
  گزینه `--json` مخصوص خود را تعریف نکرده باشد.

## شروع سریع (محلی)

```bash
openclaw browser profiles
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw open https://example.com
openclaw browser --browser-profile openclaw snapshot
```

عامل‌ها می‌توانند همان بررسی آمادگی را با `browser({ action: "doctor" })` اجرا کنند.

## عیب‌یابی سریع

اگر `start` با `not reachable after start` ناموفق شد، ابتدا آمادگی CDP را عیب‌یابی کنید. اگر `start` و `tabs` موفق شدند، اما `open` یا `navigate` ناموفق شد، صفحه کنترل مرورگر سالم است و علت خرابی معمولاً مسدودسازی پیمایش توسط سیاست SSRF است.

توالی حداقلی:

```bash
openclaw browser --browser-profile openclaw doctor
openclaw browser --browser-profile openclaw start
openclaw browser --browser-profile openclaw tabs
openclaw browser --browser-profile openclaw open https://example.com
```

راهنمای تفصیلی: [عیب‌یابی مرورگر](/fa/tools/browser#cdp-startup-failure-vs-navigation-ssrf-block)

## چرخه حیات

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep
openclaw browser start
openclaw browser start --headless
openclaw browser stop
openclaw browser --browser-profile openclaw reset-profile
```

- `doctor --deep` یک کاوش زنده عکس فوری اضافه می‌کند: زمانی مفید است که آمادگی پایه CDP سبز است، اما می‌خواهید ثابت کنید برگه فعلی قابل بررسی است.
- برای یک پروفایل محلی مدیریت‌شده در حال اجرا، `status` و `doctor` اطلاعات تشخیصی گرافیکی ذخیره‌شده در حافظه نهان را
  از Chrome گزارش می‌کنند: دسته‌بندی سخت‌افزاری/نرم‌افزاری، رندرکننده،
  بک‌اند، دستگاه/درایور، جزئیات قابلیت‌ها و وضعیت غیرفعال‌بودن و قابلیت‌های
  ویدیویی شتاب‌یافته. `openclaw browser --json status` محتوای ساختاریافته کامل را برمی‌گرداند.
  وضعیت غیرفعال هرگز صرفاً برای جمع‌آوری این اطلاعات Chrome را اجرا نمی‌کند.
- `stop` نشست کنترل فعال را می‌بندد و جایگزین‌های موقت شبیه‌سازی را حتی برای `attachOnly` و پروفایل‌های CDP راه دور که OpenClaw فرایند مرورگرشان را اجرا نکرده است، پاک می‌کند. برای پروفایل‌های محلی مدیریت‌شده، `stop` فرایند مرورگر ایجادشده را نیز متوقف می‌کند.
- `start --headless` فقط بر همان درخواست شروع اعمال می‌شود و تنها زمانی که OpenClaw یک مرورگر محلی مدیریت‌شده را اجرا کند. این گزینه `browser.headless` یا پیکربندی پروفایل را بازنویسی نمی‌کند و برای مرورگری که از قبل در حال اجرا است، هیچ اثری ندارد.
- در میزبان‌های Linux فاقد `DISPLAY` یا `WAYLAND_DISPLAY`، پروفایل‌های محلی مدیریت‌شده به‌طور خودکار بدون رابط گرافیکی اجرا می‌شوند، مگر اینکه `OPENCLAW_BROWSER_HEADLESS=0`، `browser.headless=false` یا `browser.profiles.<name>.headless=false` صراحتاً مرورگر قابل‌مشاهده‌ای را درخواست کند.

## اگر فرمان وجود ندارد

اگر `openclaw browser` فرمانی ناشناخته است، `plugins.allow` را در `~/.openclaw/openclaw.json` بررسی کنید. هنگامی که `plugins.allow` وجود دارد، Plugin مرورگر همراه را صراحتاً فهرست کنید، مگر اینکه پیکربندی از قبل دارای بلوک ریشه `browser` باشد:

```json5
{
  plugins: {
    allow: ["telegram", "browser"],
  },
}
```

یک بلوک ریشه صریح `browser` (برای مثال `browser.enabled=true` یا `browser.profiles.<name>`) نیز Plugin مرورگر همراه را تحت فهرست مجاز محدودکننده Plugin فعال می‌کند.

مرتبط: [ابزار مرورگر](/fa/tools/browser#missing-browser-command-or-tool)

## پروفایل‌ها

پروفایل‌ها پیکربندی‌های نام‌گذاری‌شده مسیریابی مرورگر هستند:

- `openclaw` (پیش‌فرض): یک نمونه اختصاصی Chrome تحت مدیریت OpenClaw را اجرا می‌کند یا به آن متصل می‌شود (دایرکتوری داده کاربر مجزا).
- `user`: نشست فعلی Chrome شما را که در آن وارد حساب شده‌اید، از طریق Chrome DevTools MCP کنترل می‌کند.
- پروفایل‌های سفارشی CDP: به یک نقطه پایانی CDP محلی یا راه دور اشاره می‌کنند.

```bash
openclaw browser profiles
openclaw browser system-profiles
openclaw browser system-profiles --browser brave
openclaw browser import-profile --browser chrome --system Default --into imported
openclaw browser import-profile --system "Profile 1" --into work --domains google.com,youtube.com
openclaw browser create-profile --name work --color "#FF5A36"
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name remote --cdp-url https://browser-host.example.com
openclaw browser delete-profile --name work
```

برای استفاده از یک پروفایل مشخص در هر زیرفرمان از `--browser-profile <name>` استفاده کنید؛ برای مثال `openclaw browser --browser-profile work tabs`.

در macOS، فرمان `system-profiles` پروفایل‌های واقعی Chrome، Brave، Edge یا Chromium موجود در میزبان را فهرست می‌کند. فرمان `import-profile` پس از یک درخواست رضایت macOS Keychain/Touch ID، کوکی‌های آن‌ها را رمزگشایی می‌کند و در یک پروفایل تازه تحت مدیریت OpenClaw قرار می‌دهد. این فرمان فقط کوکی‌ها را وارد می‌کند؛ ذخیره‌سازی محلی و IndexedDB بدون تغییر می‌مانند. برخی نشست‌های Google از اطلاعات اعتبار نشست وابسته به دستگاه (DBSC) استفاده می‌کنند و ممکن است پس از واردکردن نیز به احراز هویت مجدد نیاز داشته باشند.

هنگامی که برنامه macOS از Gateway محلی استفاده می‌کند، می‌تواند این واردکردن را یک بار پیشنهاد دهد و پروفایل واردشده مجزا را به پیش‌فرض مرور عامل تبدیل کند. واردکردن همیشه به کلیک صریح نیاز دارد؛ واردکردن موفق یا ردکردن، نمایش خودکار درخواست‌های بعدی را متوقف می‌کند و **Settings → General → Browser login** برای واردکردن مجدد در دسترس می‌ماند.

واردکردن پروفایل سیستم به‌طور پیش‌فرض فعال است. برای غیرفعال‌کردن واردکردن‌های راه‌اندازی‌شده از طریق CLI و عامل، `browser.allowSystemProfileImport=false` را تنظیم کنید. واردکردن مختص میزبان محلی است و نمی‌تواند از طریق پروکسی Node مرورگر اجرا شود.

## برگه‌ها

```bash
openclaw browser tabs
openclaw browser tab new --label docs
openclaw browser tab label t1 docs
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://docs.openclaw.ai --label docs
openclaw browser focus docs
openclaw browser close t1
```

`tabs` ابتدا `suggestedTargetId`، سپس `tabId` پایدار (مانند `t1`)، برچسب اختیاری و `targetId` خام را برمی‌گرداند. `suggestedTargetId` را دوباره به `focus`، `close`، عکس‌های فوری و عملیات بدهید. با `open --label`، `tab new --label` یا `tab label` یک برچسب تعیین کنید؛ برچسب‌ها، شناسه‌های برگه، شناسه‌های خام مقصد و پیشوندهای یکتای شناسه مقصد همگی پذیرفته می‌شوند. برای سازگاری، نام فیلد درخواست همچنان `targetId` است، اما هرکدام از این ارجاعات برگه را می‌پذیرد.

شناسه‌های خام مقصد، دستگیره‌های تشخیصی ناپایدار هستند، نه حافظه پایدار عامل: هنگامی که Chromium مقصد خام زیربنایی را در جریان پیمایش یا ارسال فرم جایگزین می‌کند، OpenClaw در صورتی که بتواند تطابق را اثبات کند، `tabId`/برچسب پایدار را به برگه جایگزین متصل نگه می‌دارد. `suggestedTargetId` را ترجیح دهید.

## عکس فوری / نماگرفت / عملیات

عکس فوری:

```bash
openclaw browser snapshot
openclaw browser snapshot --urls
```

نماگرفت:

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref e12
openclaw browser screenshot --labels
```

- `--full-page` فقط برای ثبت صفحه است؛ نمی‌توان آن را با `--ref` یا `--element` ترکیب کرد.
- پروفایل‌های `existing-session` / `user` از نماگرفت صفحه و نماگرفت‌های `--ref` از خروجی عکس فوری پشتیبانی می‌کنند، اما از نماگرفت‌های CSS با `--element` پشتیبانی نمی‌کنند.
- `--labels` ارجاعات عکس فوری فعلی را روی نماگرفت می‌اندازد. در پروفایل‌های مبتنی بر Playwright، این گزینه با `--full-page` (هم‌پوشانی تمام‌صفحه)، `--ref` (هم‌پوشانی برش عنصر بر اساس ارجاع ARIA) و `--element` (هم‌پوشانی برش عنصر بر اساس انتخابگر CSS) کار می‌کند؛ در حالت‌های برش عنصر، برچسب‌ها نسبت به عنصر نگاشت می‌شوند. پاسخ همچنین شامل آرایه `annotations` است (وقتی خالی باشد حذف می‌شود) که کادر مرزی هر ارجاع را دربر دارد: `ref`، `number`، `role`، `name` اختیاری و `box: {x, y, width, height}` در فضای مختصات تصویر ثبت‌شده (ناحیه دید / تمام‌صفحه / نسبی به عنصر).
  پروفایل‌های `existing-session` روی نماگرفت‌های صفحه یک هم‌پوشانی chrome-mcp رندر می‌کنند، اما از یاریگر نگاشت Playwright استفاده نمی‌کنند و شامل `annotations` نیستند؛ نماگرفت‌های CSS با `--element` در آنجا پشتیبانی نمی‌شوند. بدون Playwright یا chrome-mcp، نماگرفت‌های برچسب‌دار در دسترس نیستند.
- `snapshot --urls` مقصد پیوندهای کشف‌شده را به عکس‌های فوری هوش مصنوعی اضافه می‌کند تا عامل‌ها بتوانند به‌جای حدس‌زدن صرفاً از روی متن پیوند، مقصدهای پیمایش مستقیم را انتخاب کنند.

پیمایش/کلیک/تایپ (خودکارسازی رابط کاربری مبتنی بر ارجاع):

```bash
openclaw browser navigate https://example.com
openclaw browser click <ref>
openclaw browser click-coords 120 340
openclaw browser type <ref> "hello"
openclaw browser press Enter
openclaw browser hover <ref>
openclaw browser scrollintoview <ref>
openclaw browser drag <startRef> <endRef>
openclaw browser select <ref> OptionA OptionB
openclaw browser fill --fields '[{"ref":"1","value":"Ada"}]'
openclaw browser wait --text "Done"
openclaw browser evaluate --fn '(el) => el.textContent' --ref <ref>
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
```

`evaluate --fn` منبع تابع، عبارت یا بدنه دستور را می‌پذیرد. بدنه‌های دستور به‌صورت تابع‌های ناهمگام بسته‌بندی می‌شوند، بنابراین برای مقداری که می‌خواهید برگردانده شود از `return` استفاده کنید. هنگامی که تابع سمت صفحه ممکن است به زمانی بیش از مهلت پیش‌فرض ارزیابی نیاز داشته باشد، از `--timeout-ms` استفاده کنید. `browser.evaluateEnabled=false` (پیش‌فرض: `true`) هر دو `evaluate` و `wait --fn` را غیرفعال می‌کند.

هنگامی که OpenClaw بتواند برگه جایگزین را اثبات کند، پاسخ عملیات پس از جایگزینی صفحه ناشی از عملیات، `targetId` خام فعلی را برمی‌گرداند. اسکریپت‌ها همچنان باید برای جریان‌های کاری طولانی‌مدت، `suggestedTargetId`/برچسب‌ها را ذخیره و ارسال کنند.

یاریگرهای فایل و کادر گفت‌وگو:

```bash
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref <ref>
openclaw browser upload media://inbound/file.pdf --ref <ref>
openclaw browser waitfordownload
openclaw browser download <ref> report.pdf
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
```

پروفایل‌های مدیریت‌شده Chrome، دانلودهای معمولی ناشی از کلیک را در دایرکتوری دانلودهای OpenClaw ذخیره می‌کنند (به‌طور پیش‌فرض `/tmp/openclaw/downloads`، یا ریشه موقت پیکربندی‌شده). هنگامی که عامل باید منتظر فایل مشخصی بماند و مسیر آن را برگرداند، از `waitfordownload` یا `download` استفاده کنید؛ آن منتظرهای صریح، مالک دانلود بعدی هستند. بارگذاری‌ها فایل‌های موجود در ریشه موقت بارگذاری‌های OpenClaw و رسانه ورودی تحت مدیریت OpenClaw، از جمله ارجاعات `media://inbound/<id>` و `media/inbound/<id>` نسبی به سندباکس را می‌پذیرند. ارجاعات رسانه تودرتو، پیمایش مسیر و مسیرهای محلی دلخواه رد می‌شوند.

هنگامی که عملی یک کادر گفت‌وگوی معین باز می‌کند، پاسخ عملیات `blockedByDialog` را با `browserState.dialogs.pending` برمی‌گرداند؛ برای پاسخ مستقیم، `--dialog-id` را ارسال کنید. کادرهای گفت‌وگویی که خارج از OpenClaw مدیریت شده‌اند، زیر `browserState.dialogs.recent` ظاهر می‌شوند.

عملیات دسته‌ای:

```bash
openclaw browser batch --actions '[{"kind":"wait","timeMs":500},{"kind":"click","ref":"12"},{"kind":"type","ref":"23","text":"hello"}]'
openclaw browser batch --actions-file plan.json
openclaw browser batch --actions-file - --continue
```

`openclaw browser batch` یک درخواست `kind="batch"` `/act` با کنش‌های تودرتوی `BrowserActRequest` (`wait`، `click`، `type`، `evaluate`، ...) ارسال می‌کند — نه `open`/`navigate`/`snapshot`/`screenshot` که زیرفرمان‌های CLI هستند، نه انواع `/act`. ‏`--continue` مقدار `stopOnError=false` را تنظیم می‌کند (حالت پیش‌فرض با نخستین خطا متوقف می‌شود)؛ `--target-id` کل دسته را به یک زبانه محدود می‌کند. شکست یک کنش تودرتو باعث می‌شود فرمان با کد غیرصفر خارج شود؛ برای حفظ پاسخ مرتب‌شدهٔ `results` از `--json` استفاده کنید. برای قرارداد کامل (چرخهٔ عمر ارجاع، تداخل شناسهٔ هدف و خلاصهٔ خطا)، [CLI دسته‌ای مرورگر](/fa/tools/browser-control#browser-batch-cli) را ببینید. `batch` در نمایه‌های `profile="user"` / نشست موجود پشتیبانی نمی‌شود.

## وضعیت و ذخیره‌سازی

ناحیهٔ دید + شبیه‌سازی:

```bash
openclaw browser resize 1280 720
openclaw browser set viewport 1280 720
openclaw browser set offline on
openclaw browser set media dark
openclaw browser set timezone Europe/London
openclaw browser set locale en-GB
openclaw browser set geo 51.5074 -0.1278 --accuracy 25
openclaw browser set device "iPhone 14"
openclaw browser set headers '{"x-test":"1"}'
openclaw browser set credentials myuser mypass
```

کوکی‌ها + فضای ذخیره‌سازی:

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url https://example.com
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set token abc123
openclaw browser storage session clear
```

## اشکال‌زدایی

```bash
openclaw browser console --level error
openclaw browser pdf
openclaw browser responsebody "**/api"
openclaw browser highlight <ref>
openclaw browser errors --clear
openclaw browser requests --filter api
openclaw browser trace start
openclaw browser trace stop --out trace.zip
```

## ‏Chrome موجود از طریق MCP

از نمایهٔ داخلی `user` استفاده کنید یا نمایهٔ `existing-session` خود را بسازید:

```bash
openclaw browser --browser-profile user tabs
openclaw browser create-profile --name chrome-live --driver existing-session
openclaw browser create-profile --name brave-live --driver existing-session --user-data-dir "~/Library/Application Support/BraveSoftware/Brave-Browser"
openclaw browser create-profile --name chrome-port --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser --browser-profile chrome-live tabs
```

مسیر پیش‌فرض نشست موجود، اتصال خودکار Chrome MCP فقط روی میزبان است. اگر مرورگر از قبل با یک نقطهٔ پایانی DevTools در حال اجراست، `--cdp-url` را ارسال کنید تا Chrome MCP در عوض به آن نقطهٔ پایانی متصل شود. برای Docker، ‏Browserless یا سایر راه‌اندازی‌های راه‌دور که به معناشناسی Chrome MCP نیازی ندارند، به‌جای آن از یک نمایهٔ CDP استفاده کنید.

محدودیت‌های کنونی نشست موجود:

- کنش‌های مبتنی بر عکس فوری از ارجاع‌ها استفاده می‌کنند، نه انتخابگرهای CSS.
- درخواست‌های پشتیبانی‌شدهٔ `act`، هنگامی که فراخواننده‌ها `timeoutMs` را حذف کنند، از مقدار پیش‌فرض داخلی 60000 ms استفاده می‌کنند؛ `timeoutMs` در هر فراخوانی همچنان اولویت دارد.
- `click` فقط از کلیک چپ پشتیبانی می‌کند.
- `type` از `slowly=true` پشتیبانی نمی‌کند.
- `press` از `delayMs` پشتیبانی نمی‌کند.
- `hover`، `scrollintoview`، `drag`، `select` و `fill` بازنویسی مهلت زمانی در هر فراخوانی را رد می‌کنند؛ `evaluate` مقدار `--timeout-ms` را می‌پذیرد.
- `select` فقط از یک مقدار پشتیبانی می‌کند.
- `wait --load networkidle` پشتیبانی نمی‌شود (در نمایه‌های مدیریت‌شده و CDP خام/راه‌دور کار می‌کند).
- بارگذاری فایل به `--ref` / `--input-ref` نیاز دارد، از `--element` مربوط به CSS پشتیبانی نمی‌کند و هر بار از یک فایل پشتیبانی می‌کند.
- قلاب‌های کادر گفت‌وگو از `--timeout` پشتیبانی نمی‌کنند.
- عکس‌های صفحه از ثبت صفحه و `--ref` پشتیبانی می‌کنند، اما از `--element` مربوط به CSS پشتیبانی نمی‌کنند.
- `responsebody`، رهگیری بارگیری، برون‌بری PDF و کنش‌های دسته‌ای همچنان به یک مرورگر مدیریت‌شده یا نمایهٔ CDP خام نیاز دارند.

## کنترل مرورگر راه‌دور (پراکسی میزبان Node)

اگر Gateway روی دستگاهی متفاوت از مرورگر اجرا می‌شود، یک **میزبان Node** را روی دستگاهی اجرا کنید که Chrome/Brave/Edge/Chromium روی آن قرار دارد. Gateway کنش‌های مرورگر را به آن Node پراکسی می‌کند؛ به سرور جداگانه‌ای برای کنترل مرورگر نیازی نیست.

برای کنترل مسیریابی خودکار از `gateway.nodes.browser.mode` و در صورت اتصال چند Node، برای تثبیت یک Node مشخص از `gateway.nodes.browser.node` استفاده کنید.

امنیت + راه‌اندازی راه‌دور: [ابزار مرورگر](/fa/tools/browser)، [دسترسی راه‌دور](/fa/gateway/remote)، [Tailscale](/fa/gateway/tailscale)، [امنیت](/fa/gateway/security)

## مرتبط

- [مرجع CLI](/fa/cli)
- [مرورگر](/fa/tools/browser)
