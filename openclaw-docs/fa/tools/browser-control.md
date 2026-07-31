---
read_when:
    - اسکریپت‌نویسی یا اشکال‌زدایی مرورگر عامل از طریق API کنترل محلی
    - در جست‌وجوی مرجع CLI برای `openclaw browser` هستید؟
    - افزودن خودکارسازی سفارشی مرورگر با اسنپ‌شات‌ها و ارجاع‌ها
summary: API کنترل مرورگر OpenClaw، مرجع CLI و کنش‌های اسکریپت‌نویسی
title: API کنترل مرورگر
x-i18n:
    generated_at: "2026-07-27T17:09:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 812358a5ad366e419413b78507d3620ea9f3981224bc8cc62fb512b87eaadd9b
    source_path: tools/browser-control.md
    workflow: 16
---

برای راه‌اندازی، پیکربندی و عیب‌یابی، به [مرورگر](/fa/tools/browser) مراجعه کنید.
این صفحه مرجع API محلی HTTP کنترل، `openclaw browser`
CLI و الگوهای اسکریپت‌نویسی (snapshotها، refها، انتظارها و جریان‌های اشکال‌زدایی) است.

## API کنترل (اختیاری)

فقط برای یکپارچه‌سازی‌های محلی، Gateway یک API کوچک HTTP روی loopback ارائه می‌کند.
این سرور مستقل اختیاری است — متغیر محیطی
`OPENCLAW_EAGER_BROWSER_CONTROL_SERVER=1` را در محیط سرویس Gateway تنظیم کنید
و پیش از دردسترس قرار گرفتن endpointهای HTTP، Gateway را مجدداً راه‌اندازی کنید. بدون
این متغیر، زمان‌اجرای کنترل مرورگر همچنان از طریق CLI و
ابزارهای عامل کار می‌کند، اما هیچ‌چیز روی پورت کنترل loopback گوش نمی‌دهد.

- وضعیت/شروع/توقف: `GET /`, `GET /doctor`, `POST /start`, `POST /stop`, `POST /reset-profile`
- پروفایل‌ها: `GET /profiles`, `POST /profiles/create`, `DELETE /profiles/:name`
- زبانه‌ها: `GET /tabs`, `POST /tabs/open`, `POST /tabs/focus`, `DELETE /tabs/:targetId`, `POST /tabs/action`
- Snapshot/نماگرفت: `GET /snapshot`, `POST /screenshot`
- کنش‌ها: `POST /navigate`, `POST /act`
- قلاب‌ها: `POST /hooks/file-chooser`, `POST /hooks/dialog`
- بارگیری‌ها: `POST /download`, `POST /wait/download`
- مجوزها: `POST /permissions/grant`
- اشکال‌زدایی: `GET /console`, `POST /pdf`
- اشکال‌زدایی: `GET /errors`, `GET /requests`, `GET /dialogs`, `POST /trace/start`, `POST /trace/stop`, `POST /highlight`
- شبکه: `POST /response/body`
- حالت: `GET /cookies`, `POST /cookies/set`, `POST /cookies/clear`
- حالت: `GET /storage/:kind`, `POST /storage/:kind/set`, `POST /storage/:kind/clear`
- تنظیمات: `POST /set/offline`, `POST /set/headers`, `POST /set/credentials`, `POST /set/geolocation`, `POST /set/media`, `POST /set/timezone`, `POST /set/locale`, `POST /set/device`

`POST /tabs/action` شکل دسته‌ای است که CLI به‌صورت داخلی برای
زیرفرمان‌های `browser tab` استفاده می‌کند (`{"action":"new"|"label"|"select"|"close"|"list", ...}`);
هنگام اسکریپت‌نویسی مستقیم، مسیرهای تک‌منظوره زبانه در بالا را ترجیح دهید.

همه endpointها `?profile=<name>` را می‌پذیرند. `POST /start?headless=true` یک
اجرای یک‌باره headless را برای پروفایل‌های مدیریت‌شده محلی، بدون تغییر پیکربندی
ذخیره‌شده مرورگر، درخواست می‌کند؛ پروفایل‌های فقط-اتصال، CDP راه‌دور و نشست موجود
این بازنویسی را رد می‌کنند، زیرا OpenClaw آن فرایندهای مرورگر را اجرا نمی‌کند.

برای endpointهای زبانه، `targetId` نام فیلد سازگاری است. ارسال
`suggestedTargetId` از `GET /tabs` یا `POST /tabs/open` را ترجیح دهید؛ برچسب‌ها و شناسه‌های `tabId`
مانند `t1` نیز پذیرفته می‌شوند. شناسه‌های خام هدف CDP و پیشوندهای یکتای خام
شناسه هدف همچنان کار می‌کنند، اما شناسه‌های تشخیصی ناپایداری هستند.

اگر احراز هویت Gateway با secret مشترک پیکربندی شده باشد، مسیرهای HTTP مرورگر نیز به احراز هویت نیاز دارند:

- `Authorization: Bearer <gateway token>`
- `x-openclaw-password: <gateway password>` یا احراز هویت HTTP Basic با آن گذرواژه

نکته‌ها:

- این API مستقل مرورگر روی loopback، هدرهای هویتی trusted-proxy یا
  Tailscale Serve را مصرف **نمی‌کند**.
- اگر `gateway.auth.mode` برابر `none` یا `trusted-proxy` باشد، این مسیرهای مرورگر روی loopback
  آن حالت‌های حامل هویت را به ارث نمی‌برند؛ آن‌ها را فقط روی loopback نگه دارید.

### قرارداد خطای `/act`

`POST /act` برای اعتبارسنجی در سطح مسیر و
شکست‌های سیاست، از پاسخ خطای ساخت‌یافته استفاده می‌کند:

```json
{ "error": "<message>", "code": "ACT_*" }
```

مقادیر فعلی `code`:

- `ACT_KIND_REQUIRED` (HTTP 400): `kind` وجود ندارد یا شناخته‌شده نیست.
- `ACT_INVALID_REQUEST` (HTTP 400): نرمال‌سازی یا اعتبارسنجی payload کنش ناموفق بود.
- `ACT_SELECTOR_UNSUPPORTED` (HTTP 400): `selector` با نوع کنشی پشتیبانی‌نشده استفاده شد.
- `ACT_EVALUATE_DISABLED` (HTTP 403): `evaluate` (یا `wait --fn`) در پیکربندی غیرفعال است.
- `ACT_TARGET_ID_MISMATCH` (HTTP 403): مقدار سطح‌بالا یا دسته‌ای `targetId` با هدف درخواست تعارض دارد.
- `ACT_EXISTING_SESSION_UNSUPPORTED` (HTTP 501): این کنش برای پروفایل‌های نشست موجود پشتیبانی نمی‌شود.

سایر شکست‌های زمان‌اجرا ممکن است همچنان `{ "error": "<message>" }` را بدون
فیلد `code` برگردانند.

### نیازمندی Playwright

برخی قابلیت‌ها (پیمایش/کنش/snapshot هوش مصنوعی/snapshot نقش، نماگرفت عناصر،
PDF) به Playwright نیاز دارند. اگر Playwright نصب نباشد، آن endpointها
خطای واضح 501 برمی‌گردانند.

مواردی که بدون Playwright همچنان کار می‌کنند:

- snapshotهای ARIA
- snapshotهای دسترس‌پذیری به‌سبک نقش (`--interactive`, `--compact`,
  `--depth`, `--efficient`) هنگامی که WebSocket مربوط به CDP هر زبانه در دسترس باشد. این
  یک مسیر جایگزین برای بازرسی و کشف ref است؛ Playwright همچنان موتور اصلی
  کنش باقی می‌ماند.
- نماگرفت‌های صفحه برای مرورگر مدیریت‌شده `openclaw`، هنگامی که WebSocket مربوط به CDP
  هر زبانه در دسترس باشد
- نماگرفت‌های صفحه برای پروفایل‌های `existing-session` / Chrome MCP
- نماگرفت‌های مبتنی بر ref در `existing-session` (`--ref`) از خروجی snapshot

مواردی که همچنان به Playwright نیاز دارند:

- `navigate`
- `act`
- snapshotهای هوش مصنوعی که به قالب بومی snapshot هوش مصنوعی Playwright وابسته‌اند
- نماگرفت عناصر با انتخابگر CSS (`--element`)
- خروجی کامل PDF مرورگر

نماگرفت عناصر همچنین `--full-page` را رد می‌کند؛ مسیر `fullPage is
not supported for element screenshots` را برمی‌گرداند.

اگر `Playwright is not available in this gateway build` را مشاهده کردید، بسته
Gateway فاقد وابستگی اصلی زمان‌اجرای مرورگر است. OpenClaw را دوباره نصب یا به‌روزرسانی کنید،
سپس Gateway را مجدداً راه‌اندازی کنید. برای Docker، فایل‌های اجرایی مرورگر Chromium را نیز
مطابق زیر نصب کنید.

#### نصب Playwright در Docker

اگر Gateway در Docker اجرا می‌شود، از `npx playwright` اجتناب کنید (تعارض‌های بازنویسی npm).
برای imageهای سفارشی، Chromium را در image بگنجانید:

```bash
OPENCLAW_INSTALL_BROWSER=1 ./scripts/docker/setup.sh
```

برای یک image موجود، در عوض از طریق CLI همراه بسته نصب کنید:

```bash
docker compose run --rm openclaw-cli \
  node /app/node_modules/playwright-core/cli.js install chromium
```

برای ماندگار کردن بارگیری‌های مرورگر، `PLAYWRIGHT_BROWSERS_PATH` را تنظیم کنید (برای مثال،
`/home/node/.cache/ms-playwright`) و مطمئن شوید `/home/node` از طریق
`OPENCLAW_HOME_VOLUME` یا یک bind mount ماندگار می‌شود. OpenClaw به‌طور خودکار
Chromium ماندگارشده را در Linux شناسایی می‌کند. به [Docker](/fa/install/docker) مراجعه کنید.

## نحوه کار (داخلی)

یک سرور کوچک کنترل روی loopback درخواست‌های HTTP را می‌پذیرد و از طریق CDP به مرورگرهای مبتنی بر Chromium متصل می‌شود. کنش‌های پیشرفته (کلیک/تایپ/snapshot/PDF) از طریق Playwright روی CDP انجام می‌شوند؛ وقتی Playwright وجود ندارد، فقط عملیات غیر Playwright در دسترس‌اند. عامل یک رابط پایدار می‌بیند، درحالی‌که مرورگرها و پروفایل‌های محلی/راه‌دور در لایه زیرین آزادانه جابه‌جا می‌شوند.

## مرجع سریع CLI

همه فرمان‌ها `--browser-profile <name>` را برای هدف‌گیری یک پروفایل مشخص و `--json` را برای خروجی قابل‌خواندن توسط ماشین می‌پذیرند.

<AccordionGroup>

<Accordion title="مبانی: وضعیت، زبانه‌ها، باز کردن/فوکوس/بستن">

```bash
openclaw browser status
openclaw browser doctor
openclaw browser doctor --deep    # یک کاوش زنده snapshot اضافه می‌کند
openclaw browser start
openclaw browser start --headless # اجرای یک‌باره headless مدیریت‌شده محلی
openclaw browser stop            # شبیه‌سازی را در CDP فقط-اتصال/راه‌دور نیز پاک می‌کند
openclaw browser reset-profile   # داده‌های مرورگر پروفایل را به Trash منتقل می‌کند
openclaw browser tabs
openclaw browser tab             # میان‌بر زبانه فعلی
openclaw browser tab new
openclaw browser tab new --label research
openclaw browser tab label abcd1234 research
openclaw browser tab select 2
openclaw browser tab close 2
openclaw browser open https://example.com
openclaw browser focus abcd1234
openclaw browser close abcd1234
```

</Accordion>

<Accordion title="پروفایل‌ها: فهرست، ایجاد، حذف">

```bash
openclaw browser profiles
openclaw browser create-profile --name research --color "#0066CC"
openclaw browser create-profile --name attach --driver existing-session --cdp-url http://127.0.0.1:9222
openclaw browser delete-profile --name research
```

</Accordion>

<Accordion title="بازرسی: نماگرفت، snapshot، کنسول، خطاها، درخواست‌ها">

```bash
openclaw browser screenshot
openclaw browser screenshot --full-page
openclaw browser screenshot --ref 12        # یا --ref e12
openclaw browser screenshot --labels
openclaw browser snapshot
openclaw browser snapshot --format aria --limit 200
openclaw browser snapshot --interactive --compact --depth 6
openclaw browser snapshot --efficient
openclaw browser snapshot --labels
openclaw browser snapshot --urls
openclaw browser snapshot --selector "#main" --interactive
openclaw browser snapshot --frame "iframe#main" --interactive
openclaw browser snapshot --out snapshot.txt
openclaw browser console --level error
openclaw browser errors --clear
openclaw browser requests --filter api --clear
openclaw browser pdf
openclaw browser responsebody "**/api" --max-chars 5000
```

</Accordion>

<Accordion title="کنش‌ها: پیمایش، کلیک، تایپ، کشیدن، انتظار، ارزیابی">

```bash
openclaw browser navigate https://example.com
openclaw browser resize 1280 720
openclaw browser click 12 --double           # یا e12 برای refهای نقش
openclaw browser click-coords 120 340        # مختصات viewport
openclaw browser type 23 "hello" --submit
openclaw browser press Enter
openclaw browser hover 44
openclaw browser scrollintoview e12
openclaw browser drag 10 11
openclaw browser select 9 OptionA OptionB
openclaw browser download e12 report.pdf
openclaw browser waitfordownload report.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf
openclaw browser upload /tmp/openclaw/uploads/file.pdf --ref e12
openclaw browser upload media://inbound/file.pdf
openclaw browser fill --fields '[{"ref":"1","type":"text","value":"Ada"}]'
openclaw browser dialog --accept
openclaw browser dialog --dismiss --dialog-id d1
openclaw browser wait --text "Done"
openclaw browser wait "#main" --url "**/dash" --load networkidle --fn "window.ready===true"
openclaw browser evaluate --fn '(el) => el.textContent' --ref 7
openclaw browser evaluate --fn 'const title = document.title; return title;'
openclaw browser evaluate --timeout-ms 30000 --fn 'async () => { await window.ready; return true; }'
openclaw browser highlight e12
openclaw browser trace start
openclaw browser trace stop
```

</Accordion>

<Accordion title="حالت: کوکی‌ها، فضای ذخیره‌سازی، آفلاین، هدرها، موقعیت جغرافیایی، دستگاه">

```bash
openclaw browser cookies
openclaw browser cookies set session abc123 --url "https://example.com"
openclaw browser cookies clear
openclaw browser storage local get
openclaw browser storage local set theme dark
openclaw browser storage session clear
openclaw browser set offline on
openclaw browser set headers --headers-json '{"X-Debug":"1"}'
openclaw browser set credentials user pass            # برای حذف از --clear استفاده کنید
openclaw browser set geo 37.7749 -122.4194 --origin "https://example.com"
openclaw browser set media dark
openclaw browser set timezone America/New_York
openclaw browser set locale en-US
openclaw browser set device "iPhone 14"
```

</Accordion>

</AccordionGroup>

نکته‌ها:

- ابزار `browser` ویژه عامل، `action=download` (با `ref` و
  `path` الزامی) و `action=waitfordownload` (با `path` اختیاری) را ارائه می‌کند. هر دو، URL بارگیری ذخیره‌شده، نام فایل پیشنهادی و مسیر محلی محافظت‌شده را برمی‌گردانند. رهگیری صریح بارگیری
  برای پروفایل‌های مدیریت‌شده Playwright در دسترس است؛ پروفایل‌های نشست موجود
  خطای عملیات پشتیبانی‌نشده برمی‌گردانند.
- بارگذاری اتمی از انتخاب‌گر را ترجیح دهید: `--ref` راه‌انداز را همراه بارگذاری ارسال کنید تا OpenClaw در یک درخواست آماده‌سازی و کلیک کند. `upload` فقط شامل مسیرها، هنگامی که راه‌اندازی بعدی عمدی است، همچنان پشتیبانی می‌شود. برای تنظیم مستقیم ورودی فایل از `--input-ref` یا `--element` استفاده کنید. `dialog` فراخوانی آماده‌سازی است؛ آن را پیش از کلیک/فشردنی اجرا کنید که گفت‌وگو را راه‌اندازی می‌کند. اگر عملی یک پنجره مودال باز کند، پاسخ عمل شامل `blockedByDialog` و `browserState.dialogs.pending` است؛ برای پاسخ مستقیم، آن `dialogId` را ارسال کنید. گفت‌وگوهایی که خارج از OpenClaw مدیریت می‌شوند، زیر `browserState.dialogs.recent` ظاهر می‌شوند.
- `click`/`type`/و غیره، به یک `ref` از `snapshot` نیاز دارند (`12` عددی، ارجاع نقش `e12` یا ارجاع ARIA قابل‌عمل `ax12`). انتخاب‌گرهای CSS عمداً برای عملیات پشتیبانی نمی‌شوند. هنگامی که موقعیت در نمای قابل‌مشاهده تنها هدف قابل‌اعتماد است، از `click-coords` استفاده کنید.
- مسیرهای بارگیری و ردیابی به ریشه‌های موقت OpenClaw محدود می‌شوند: `/tmp/openclaw{,/downloads}` (مسیر جایگزین: `${os.tmpdir()}/openclaw/...`).
- `upload` فایل‌ها را از ریشه بارگذاری‌های موقت OpenClaw و
  رسانه ورودی مدیریت‌شده توسط OpenClaw می‌پذیرد. رسانه ورودی مدیریت‌شده را می‌توان به‌شکل
  `media://inbound/<id>`، `media/inbound/<id>` نسبی به sandbox یا یک
  مسیر حل‌شده درون پوشه رسانه ورودی مدیریت‌شده ارجاع داد. ارجاع‌های رسانه‌ای تودرتو،
  پیمایش مسیر، پیوندهای نمادین، پیوندهای سخت و مسیرهای محلی دلخواه همچنان رد می‌شوند.
- `upload` همچنین می‌تواند ورودی‌های فایل را مستقیماً از طریق `--input-ref` یا `--element` تنظیم کند.

شناسه‌ها و برچسب‌های پایدار زبانه هنگام جایگزینی هدف خام Chromium حفظ می‌شوند، مشروط بر اینکه OpenClaw
بتواند زبانه جایگزین را اثبات کند؛ مانند یک جفت قدیمی/جدید یکتا برای همان URL یا
تبدیل یک زبانه قدیمی به یک زبانه جدید پس از ارسال فرم. جایگزینی‌های مبهم با
URL تکراری، دستگیره‌های تازه دریافت می‌کنند. شناسه‌های خام هدف همچنان
ناپایدارند؛ در اسکریپت‌ها `suggestedTargetId` از `tabs` را ترجیح دهید.

نگاهی اجمالی به پرچم‌های snapshot:

- `--format ai` (پیش‌فرض با Playwright): snapshot هوش مصنوعی با ارجاع‌های عددی (`aria-ref="<n>"`).
- `--format aria`: درخت دسترس‌پذیری با ارجاع‌های `axN`. هنگامی که Playwright در دسترس باشد، OpenClaw ارجاع‌های دارای شناسه‌های DOM سمت backend را به صفحه زنده متصل می‌کند تا عملیات بعدی بتوانند از آن‌ها استفاده کنند؛ در غیر این صورت، خروجی را فقط برای بازرسی در نظر بگیرید.
- `--efficient` (یا `--mode efficient`): پیش‌تنظیم فشرده snapshot نقش. برای قراردادن این مورد به‌عنوان پیش‌فرض، `browser.snapshotDefaults.mode: "efficient"` را تنظیم کنید (به [پیکربندی Gateway](/fa/gateway/configuration-reference#browser) مراجعه کنید).
- `--interactive`، `--compact`، `--depth` و `--selector`، snapshot نقش را با ارجاع‌های `ref=e12` تحمیل می‌کنند. `--frame "<iframe>"` محدوده snapshotهای نقش را به یک iframe محدود می‌کند.
- با Playwright، ‏`--labels` یک تصویر صفحه با برچسب‌های ارجاع هم‌پوشان اضافه می‌کند
  (`MEDIA:<path>` را چاپ می‌کند)، به‌علاوه یک آرایه `annotations` شامل کادر
  مرزی هر ارجاع. در `screenshot`، برچسب‌های مبتنی بر Playwright با `--full-page`،
  `--ref` و `--element` کار می‌کنند؛ در `snapshot`، تصویر همراه همچنان
  فقط نمای قابل‌مشاهده را پوشش می‌دهد. پروفایل‌های نشست موجود/chrome-mcp برچسب‌های هم‌پوشان را روی
  تصاویر صفحه رندر می‌کنند، اما `annotations` را برنمی‌گردانند و از ابزار کمکی نگاشت
  تمام‌صفحه/ارجاع/عنصر Playwright استفاده نمی‌کنند. بدون Playwright یا chrome-mcp،
  تصاویر دارای برچسب در دسترس نیستند.
- `--urls` مقصد پیوندهای کشف‌شده را به snapshotهای هوش مصنوعی می‌افزاید.

## Snapshotها و ارجاع‌ها

OpenClaw از دو سبک «snapshot» پشتیبانی می‌کند:

- **snapshot هوش مصنوعی (ارجاع‌های عددی)**: `openclaw browser snapshot` (پیش‌فرض؛ `--format ai`)
  - خروجی: یک snapshot متنی شامل ارجاع‌های عددی.
  - عملیات: `openclaw browser click 12`، `openclaw browser type 23 "hello"`.
  - در داخل، ارجاع از طریق `aria-ref` در Playwright حل می‌شود.

- **snapshot نقش (ارجاع‌های نقش مانند `e12`)**: `openclaw browser snapshot --interactive` (یا `--compact`، `--depth`، `--selector`، `--frame`)
  - خروجی: فهرست/درختی مبتنی بر نقش با `[ref=e12]` (و `[nth=1]` اختیاری).
  - عملیات: `openclaw browser click e12`، `openclaw browser highlight e12`.
  - در داخل، ارجاع از طریق `getByRole(...)` (به‌همراه `nth()` برای موارد تکراری) حل می‌شود.
  - برای افزودن یک تصویر صفحه با برچسب‌های `e12` هم‌پوشان، `--labels` را اضافه کنید. در
    پروفایل‌های مبتنی بر Playwright، این گزینه فراداده کادر مرزی هر ارجاع را نیز
    برمی‌گرداند (`annotations[]`).
  - هنگامی که متن پیوند مبهم است و عامل به اهداف پیمایش مشخص نیاز دارد،
    `--urls` را اضافه کنید.

- **snapshot ‏ARIA (ارجاع‌های ARIA مانند `ax12`)**: `openclaw browser snapshot --format aria`
  - خروجی: درخت دسترس‌پذیری به‌صورت گره‌های ساختاریافته.
  - عملیات: وقتی مسیر snapshot بتواند ارجاع را
    از طریق Playwright و شناسه‌های DOM سمت backend کروم متصل کند، `openclaw browser click ax12` کار می‌کند.
- اگر Playwright در دسترس نباشد، snapshotهای ARIA همچنان می‌توانند برای
  بازرسی مفید باشند، اما ممکن است ارجاع‌ها قابل‌عمل نباشند. هنگامی که به ارجاع‌های عملیاتی نیاز دارید،
  دوباره با `--format ai` یا `--interactive` snapshot بگیرید.
- اثبات Docker برای مسیر جایگزین raw-CDP: ‏`pnpm test:docker:browser-cdp-snapshot`
  ‏Chromium را با CDP راه‌اندازی می‌کند، `browser doctor --deep` را اجرا می‌کند و تأیید می‌کند که snapshotهای نقش
  شامل URL پیوندها، عناصر قابل‌کلیک ارتقایافته با نشانگر و فراداده iframe هستند.

رفتار ارجاع‌ها:

- ارجاع‌ها **در پیمایش‌ها پایدار نیستند**؛ اگر چیزی ناموفق بود، `snapshot` را دوباره اجرا کنید و از یک ارجاع تازه استفاده کنید.
- `/act` پس از جایگزینی ناشی از یک عمل، وقتی بتواند زبانه جایگزین را اثبات کند،
  `targetId` خام کنونی را برمی‌گرداند. برای فرمان‌های
  بعدی همچنان از شناسه‌ها/برچسب‌های پایدار زبانه استفاده کنید.
- اگر snapshot نقش با `--frame` گرفته شده باشد، ارجاع‌های نقش تا snapshot نقش بعدی به همان iframe محدود می‌شوند.
- ارجاع‌های `axN` ناشناخته یا منقضی، به‌جای افتادن در مسیر انتخاب‌گر
  `aria-ref` در Playwright، سریعاً ناموفق می‌شوند. در این حالت،
  روی همان زبانه یک snapshot تازه بگیرید.

## CLI دسته‌ای مرورگر

`openclaw browser batch` آرایه‌ای از عملیات تودرتوی `/act` را در یک فراخوانی `/act`
اجرا می‌کند (همان runtime ‏`kind="batch"` که از طریق ابزار عامل قابل‌دسترسی است)، بنابراین کاربران CLI
و اسکریپت‌ها می‌توانند عملیاتی مانند `wait`، `click`، `type` و
`evaluate` را بدون رفت‌وبرگشت جداگانه برای هر عمل، در یک برنامه قابل‌بازپخش ترکیب کنند. هر
ورودی در `actions[]` یک `BrowserActRequest` است — اجتماع بسته‌ای که مسیر `/act`
می‌پذیرد (`click`، `clickCoords`، `type`، `press`، `hover`،
`scrollIntoView`، `drag`، `select`، `fill`، `resize`، `wait`، `evaluate`،
`close`، `batch`) — نه زیرفرمان‌های دلخواه `openclaw browser`. ‏`batch`
در `profile="user"` و دیگر پروفایل‌های نشست موجود (chrome-mcp)
پشتیبانی نمی‌شود؛ در آن‌ها عملیات را جداگانه ارسال کنید.

- CLI: ‏`openclaw browser batch --actions '<json>'`، `openclaw browser batch
--actions-file plan.json` یا `openclaw browser batch --actions-file -` برای
  خواندن آرایه JSON از ورودی استاندارد. `--continue`، ‏`stopOnError=false` را تنظیم می‌کند؛
  پیش‌فرض، توقف در نخستین خطاست. `--target-id` کل دسته را به
  یک زبانه محدود می‌کند.
- چرخه عمر ارجاع: ارجاع‌ها از اجرای `snapshot` پیش از دسته می‌آیند (snapshot یک
  عمل تودرتو نیست). یک عمل تودرتو که وضعیت صفحه را تغییر می‌دهد — مانند
  `click` که پیمایش را راه‌اندازی می‌کند یا `evaluate` که DOM را تغییر می‌دهد — می‌تواند
  ارجاع‌های پیشین را برای ادامه دسته نامعتبر کند. عملیات تغییردهنده وضعیت را
  ابتدا قرار دهید یا پس از snapshotگیری دوباره، آن‌ها را به یک دسته بعدی تقسیم کنید. پیمایش و
  snapshotگیری دوباره خارج از دسته انجام می‌شوند (`openclaw browser navigate` /
  `snapshot`)؛ زیرا `open`، `navigate` و `snapshot` از انواع `/act` نیستند.
- تداخل شناسه هدف: یک عمل تودرتو می‌تواند `targetId` را حذف کند یا
  `targetId` سطح درخواست را تکرار کند؛ یک `targetId` صریح تودرتو که به
  زبانه‌ای متفاوت حل شود، پیش از اجرای هر عملی با `ACT_TARGET_ID_MISMATCH`
  رد می‌شود. عملیات دسته‌ای عمداً زبانه درخواست را به‌اشتراک می‌گذارند.
- خلاصه خطا: پاسخ `{ "results": [{ "ok": true }, { "ok": false,
"error": "<message>" }, ...] }` است، با یک ورودی برای هر عمل به‌ترتیب. هنگامی که
  `stopOnError` پیش‌فرض باشد، آرایه در نخستین شکست پایان می‌یابد؛ با
  `--continue` همه عملیات را پوشش می‌دهد. هر ورودی ناموفق باعث می‌شود CLI با
  کد غیرصفر خارج شود؛ برای حفظ پاسخ کامل و مرتب‌شده برای اسکریپت‌ها، `--json` را ارسال کنید.

## قابلیت‌های تقویت‌شده انتظار

می‌توانید برای مواردی بیش از صرفاً زمان/متن منتظر بمانید:

- انتظار برای URL (الگوهای glob توسط Playwright پشتیبانی می‌شوند):
  - `openclaw browser wait --url "**/dash"`
- انتظار برای وضعیت بارگذاری:
  - `openclaw browser wait --load networkidle`
  - در پروفایل‌های مدیریت‌شده `openclaw` و پروفایل‌های خام/راه‌دور CDP پشتیبانی می‌شود. پروفایل‌هایی که از درایور `existing-session` استفاده می‌کنند (از جمله پروفایل پیش‌فرض `user`) ‏`networkidle` را رد می‌کنند؛ در آن‌ها از انتظارهای `--url`، `--text`، یک انتخاب‌گر یا `--fn` استفاده کنید.
- انتظار برای یک گزاره JS:
  - `openclaw browser wait --fn "window.ready===true"`
- انتظار برای قابل‌مشاهده‌شدن یک انتخاب‌گر:
  - `openclaw browser wait "#main"`

این موارد را می‌توان ترکیب کرد:

```bash
openclaw browser wait "#main" \
  --url "**/dash" \
  --load networkidle \
  --fn "window.ready===true" \
  --timeout-ms 15000
```

## گردش‌کارهای اشکال‌زدایی

هنگامی که عملی ناموفق می‌شود (برای مثال «قابل‌مشاهده نیست»، «نقض حالت سخت‌گیرانه»، «پوشانده شده»):

1. `openclaw browser snapshot --interactive`
2. از `click <ref>` / `type <ref>` استفاده کنید (در حالت تعاملی، ارجاع‌های نقش را ترجیح دهید)
3. اگر همچنان ناموفق بود: `openclaw browser highlight <ref>` برای مشاهده هدف Playwright
4. اگر صفحه رفتار عجیبی دارد:
   - `openclaw browser errors --clear`
   - `openclaw browser requests --filter api --clear`
5. برای اشکال‌زدایی عمیق: یک ردگیری ضبط کنید:
   - `openclaw browser trace start`
   - مشکل را بازتولید کنید
   - `openclaw browser trace stop` (‏`TRACE:<path>` را چاپ می‌کند)

## خروجی JSON

`--json` برای اسکریپت‌نویسی و ابزارهای ساختاریافته است.

نمونه‌ها:

```bash
openclaw browser --json status
openclaw browser --json snapshot --interactive
openclaw browser --json requests --filter api
openclaw browser --json cookies
```

snapshotهای نقش در JSON شامل `refs` به‌همراه یک بلوک کوچک `stats` (خطوط/نویسه‌ها/ارجاع‌ها/تعاملی) هستند تا ابزارها بتوانند درباره اندازه و تراکم محموله استدلال کنند.

## تنظیمات وضعیت و محیط

این موارد برای گردش‌کارهای «کاری کن سایت مانند X رفتار کند» مفیدند:

- کوکی‌ها: `cookies`، `cookies set`، `cookies clear`
- فضای ذخیره‌سازی: `storage local|session get|set|clear`
- آفلاین: `set offline on|off`
- سرآیندها: `set headers --headers-json '{"X-Debug":"1"}'` (یا فرم موقعیتی `set headers '{"X-Debug":"1"}'`)
- احراز هویت پایه HTTP: ‏`set credentials user pass` (یا `--clear`)
- موقعیت جغرافیایی: `set geo <lat> <lon> --origin "https://example.com"` (یا `--clear`)
- رسانه: `set media dark|light|no-preference|none`
- منطقه زمانی / locale: ‏`set timezone ...`، `set locale ...`
- دستگاه / نمای قابل‌مشاهده:
  - `set device "iPhone 14"` (پیش‌تنظیم‌های دستگاه Playwright)
  - `set viewport 1280 720`

## امنیت و حریم خصوصی

- پروفایل مرورگر openclaw ممکن است شامل نشست‌های واردشده باشد؛ آن را حساس تلقی کنید.
- `browser act kind=evaluate` / `openclaw browser evaluate` و `wait --fn`
  جاوااسکریپت دلخواه را در زمینهٔ صفحه اجرا می‌کنند. تزریق پرامپت می‌تواند
  این رفتار را هدایت کند. اگر به آن نیاز ندارید، با `browser.evaluateEnabled=false` غیرفعالش کنید.
- `openclaw browser evaluate --fn` منبع یک تابع، یک عبارت، یا
  بدنهٔ یک دستور را می‌پذیرد. بدنه‌های دستور در قالب توابع async قرار می‌گیرند، بنابراین برای
  مقداری که می‌خواهید بازگردانده شود از `return` استفاده کنید. وقتی تابع سمت
  صفحه ممکن است به زمانی بیشتر از مهلت پیش‌فرض ارزیابی نیاز داشته باشد، از `--timeout-ms <ms>` استفاده کنید.
- برای ورودها و نکات مربوط به مقابله با ربات‌ها (X/Twitter و غیره)، به [ورود در مرورگر + ارسال پست در X/Twitter](/fa/tools/browser-login) مراجعه کنید.
- میزبان Gateway/node را خصوصی نگه دارید (فقط loopback یا tailnet).
- نقاط پایانی CDP راه‌دور قدرتمند هستند؛ آن‌ها را از طریق تونل متصل و محافظت کنید.

نمونهٔ حالت سخت‌گیرانه (مسدودسازی پیش‌فرض مقصدهای خصوصی/داخلی):

```json5
{
  browser: {
    ssrfPolicy: {
      dangerouslyAllowPrivateNetwork: false,
      hostnameAllowlist: ["*.example.com", "example.com"],
      allowedHostnames: ["localhost"], // اجازهٔ دقیق اختیاری
    },
  },
}
```

## مرتبط

- [مرورگر](/fa/tools/browser) - نمای کلی، پیکربندی، پروفایل‌ها، امنیت
- [ورود در مرورگر](/fa/tools/browser-login) - ورود به سایت‌ها
- [عیب‌یابی مرورگر در Linux](/fa/tools/browser-linux-troubleshooting)
- [عیب‌یابی مرورگر در WSL2](/fa/tools/browser-wsl2-windows-remote-cdp-troubleshooting)
