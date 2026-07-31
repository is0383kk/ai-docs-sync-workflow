---
read_when:
    - استفاده از CLI ‏ClawHub
    - اشکال‌زدایی نصب، به‌روزرسانی یا انتشار
summary: 'مرجع CLI: فرمان‌ها، پرچم‌ها، پیکربندی و رفتار فایل قفل.'
x-i18n:
    generated_at: "2026-07-27T15:13:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: eba91a83c5542c4b570bd22a526911633e43d0b4e921c013e6fd29451193f2a7
    source_path: clawhub/cli.md
    workflow: 16
---

# CLI

بستهٔ CLI: `clawhub`، فایل اجرایی: `clawhub`.

آن را به‌صورت سراسری با npm یا pnpm نصب کنید:

```bash
npm i -g clawhub
# یا
pnpm add -g clawhub
```

سپس آن را بررسی کنید:

```bash
clawhub --help
clawhub login
clawhub whoami
```

## پرچم‌های سراسری

- `--workdir <dir>`: دایرکتوری کاری (پیش‌فرض: cwd؛ در صورت پیکربندی، به فضای کاری Clawdbot برمی‌گردد)
- `--dir <dir>`: دایرکتوری نصب درون دایرکتوری کاری (پیش‌فرض: `skills`)
- `--site <url>`: نشانی پایه برای ورود از طریق مرورگر (پیش‌فرض: `https://clawhub.ai`)
- `--registry <url>`: نشانی پایهٔ API (پیش‌فرض: شناسایی‌شده؛ در غیر این صورت `https://clawhub.ai`)
- `--no-input`: غیرفعال‌کردن درخواست‌های تعاملی

معادل‌های متغیر محیطی:

- `CLAWHUB_SITE` (قدیمی: `CLAWDHUB_SITE`)
- `CLAWHUB_REGISTRY` (قدیمی: `CLAWDHUB_REGISTRY`)
- `CLAWHUB_WORKDIR` (قدیمی: `CLAWDHUB_WORKDIR`)

### پراکسی HTTP

CLI متغیرهای محیطی استاندارد پراکسی HTTP را برای سامانه‌های پشت
پراکسی‌های سازمانی یا شبکه‌های محدود رعایت می‌کند:

- `HTTPS_PROXY` / `https_proxy`
- `HTTP_PROXY` / `http_proxy`
- `NO_PROXY` / `no_proxy`

وقتی هرکدام از این متغیرها تنظیم شده باشد، CLI درخواست‌های خروجی را از طریق
پراکسی مشخص‌شده هدایت می‌کند. `HTTPS_PROXY` برای درخواست‌های HTTPS و `HTTP_PROXY`
برای HTTP ساده استفاده می‌شود. `NO_PROXY` / `no_proxy` برای عبور نکردن از پراکسی در
میزبان‌ها یا دامنه‌های خاص رعایت می‌شود.

این قابلیت در سامانه‌هایی لازم است که اتصال مستقیم خروجی در آن‌ها مسدود است
(برای مثال، کانتینرهای Docker، سرور مجازی Hetzner با اینترنت صرفاً پراکسی، یا
دیوارهای آتش سازمانی).

مثال:

```bash
export HTTPS_PROXY=http://proxy.example.com:3128
export NO_PROXY=localhost,127.0.0.1
clawhub search "پرس‌وجوی من"
```

وقتی هیچ متغیر پراکسی تنظیم نشده باشد، رفتار تغییری نمی‌کند (اتصال مستقیم).

## فایل پیکربندی

توکن API و نشانی رجیستری ذخیره‌شده در حافظهٔ نهان را نگه می‌دارد.

- macOS: `~/Library/Application Support/clawhub/config.json`
- Linux/XDG: `$XDG_CONFIG_HOME/clawhub/config.json` یا `~/.config/clawhub/config.json`
- Windows: `%APPDATA%\\clawhub\\config.json`
- مسیر جایگزین قدیمی: اگر `clawhub/config.json` هنوز وجود نداشته باشد اما `clawdhub/config.json` وجود داشته باشد، CLI از مسیر قدیمی دوباره استفاده می‌کند
- بازنویسی: `CLAWHUB_CONFIG_PATH` (قدیمی: `CLAWDHUB_CONFIG_PATH`)

## فرمان‌ها

### `login` / `auth login`

- پیش‌فرض: مرورگر را در `<site>/cli/auth` باز می‌کند و فرایند را از طریق فراخوانی بازگشتی حلقهٔ محلی تکمیل می‌کند.
- بدون رابط گرافیکی: `clawhub login --token clh_...`
- تعاملی راه‌دور/بدون رابط گرافیکی: `clawhub login --device` یک کد نمایش می‌دهد و تا زمانی که آن را در `<site>/cli/device` مجاز کنید، منتظر می‌ماند.

### `whoami`

- توکن ذخیره‌شده را از طریق `/api/v1/whoami` اعتبارسنجی می‌کند.

### `token`

- توکن API ذخیره‌شده را در stdout چاپ می‌کند.
- برای انتقال توکن ورود محلی از طریق پایپ به فرمان‌های تنظیم اسرار CI مفید است.

### `star <skill>` / `unstar <skill>`

- یک مهارت را به نشانک‌های شما اضافه یا از آن‌ها حذف می‌کند. نام فرمان‌ها برای سازگاری همچنان `star` و
  `unstar` باقی می‌مانند.
- فراخوانی `POST /api/v1/stars/<slug>` و `DELETE /api/v1/stars/<slug>` را انجام می‌دهد.
- `--yes` تأیید را رد می‌کند.

### `search <query...>`

- فراخوانی `/api/v1/search?q=...` را انجام می‌دهد.
- خروجی شامل نامک مهارت، شناسهٔ مالک، نام نمایشی و امتیاز مرتبط‌بودن است.
- جست‌وجو پیش از محبوبیت دانلود، تطبیق دقیق توکن‌های نامک/نام را ترجیح می‌دهد. یک توکن نامک مستقل مانند `map` با `personal-map` بسیار قوی‌تر از زیررشتهٔ درون `amap` تطبیق پیدا می‌کند.
- محبوبیت تنها یک پیش‌فرض رتبه‌بندی کوچک است، نه تضمینی برای قرارگرفتن در رتبهٔ نخست.
- اگر مهارتی باید ظاهر شود اما نمی‌شود، در حالت واردشده `clawhub inspect @owner/slug` را اجرا کنید تا پیش از تغییر نام فراداده، عیب‌یابی نظارتی قابل‌مشاهده برای مالک را بررسی کنید.

### `explore`

- جدیدترین مهارت‌ها را از طریق `/api/v1/skills?limit=...&sort=createdAt` فهرست می‌کند (مرتب‌شده بر اساس `createdAt` به‌صورت نزولی).
- پرچم‌ها:
  - `--limit <n>` (1-200، پیش‌فرض: 25)
  - `--sort newest|updated|rating|downloads|trending` (پیش‌فرض: جدیدترین). نام‌های مستعار قدیمی مرتب‌سازی نصب همچنان برای سازگاری کار می‌کنند.
  - `--json` (خروجی قابل‌خواندن برای ماشین)
- خروجی: `<slug>  v<version>  <age>  <summary>` (خلاصه به 50 نویسه محدود می‌شود).

### `inspect @owner/slug`

- فرادادهٔ مهارت و فایل‌های نسخه را بدون نصب دریافت می‌کند.
- `--version <version>`: بررسی یک نسخهٔ خاص (پیش‌فرض: جدیدترین).
- `--tag <tag>`: بررسی یک نسخهٔ برچسب‌خورده (برای مثال `latest`).
- `--versions`: فهرست‌کردن تاریخچهٔ نسخه‌ها (صفحهٔ نخست).
- `--limit <n>`: بیشینهٔ نسخه‌های قابل‌فهرست (1-200).
- `--files`: فهرست‌کردن فایل‌های نسخهٔ انتخاب‌شده.
- `--file <path>`: دریافت بایت‌های خام فایل (محدودیت 10MB).
- `--json`: خروجی قابل‌خواندن برای ماشین؛ `--file` در صورت امکان، بایت‌های دقیق را به‌صورت base64 و متن UTF-8 شامل می‌شود.

### `install @owner/slug`

- جدیدترین نسخه را برای مالک و مهارت نام‌برده تعیین می‌کند.
- فایل zip را از طریق `/api/v1/download` دانلود می‌کند.
- محتوا را در `<workdir>/<dir>/<slug>` استخراج می‌کند.
- از بازنویسی مهارت‌های سنجاق‌شده خودداری می‌کند؛ ابتدا `clawhub unpin <skill>` را اجرا کنید.
- موارد زیر را می‌نویسد:
  - `<workdir>/.clawhub/lock.json` (قدیمی: `.clawdhub`)
  - `<skill>/.clawhub/origin.json` (قدیمی: `.clawdhub`)

### `uninstall <skill>`

- `<workdir>/<dir>/<slug>` را حذف می‌کند و ورودی قفل‌فایل را پاک می‌کند.
- در حالت واردشده، تله‌متری را به‌صورت بهترین تلاش ارسال می‌کند تا شمار نصب‌های جاری
  غیرفعال شود.
- تعاملی: تأیید می‌خواهد.
- غیرتعاملی (`--no-input`): به `--yes` نیاز دارد.

### `list`

- `<workdir>/.clawhub/lock.json` را می‌خواند (قدیمی: `.clawdhub`).
- `pinned` را کنار مهارت‌هایی که با `clawhub pin` ثابت شده‌اند، همراه با دلیل اختیاری، نمایش می‌دهد.

### `pin <skill>`

- یک مهارت نصب‌شده را در قفل‌فایل به‌عنوان سنجاق‌شده علامت‌گذاری می‌کند.
- `--reason <text>` دلیل ثابت‌شدن مهارت را ثبت می‌کند.
- مهارت‌های سنجاق‌شده در `update --all` نادیده گرفته می‌شوند و `update <skill>` مستقیم آن‌ها را رد می‌کند.
- مهارت‌های سنجاق‌شده همچنین `install --force` را رد می‌کنند تا بایت‌های محلی به‌طور تصادفی جایگزین نشوند.

### `unpin <skill>`

- سنجاق قفل‌فایل را از یک مهارت نصب‌شده حذف می‌کند تا به‌روزرسانی‌های آینده بتوانند آن را تغییر دهند.

### `update [@owner/slug]` / `update --all`

- اثر انگشت را از فایل‌های محلی محاسبه می‌کند.
- اگر اثر انگشت با نسخه‌ای شناخته‌شده مطابقت داشته باشد: درخواستی نمایش داده نمی‌شود.
- اگر اثر انگشت مطابقت نداشته باشد:
  - به‌طور پیش‌فرض خودداری می‌کند
  - با `--force` بازنویسی می‌کند (یا در حالت تعاملی درخواست تأیید می‌دهد)
- مهارت‌های سنجاق‌شده هرگز با `--force` به‌روزرسانی نمی‌شوند.
- `update <skill>` برای مهارت‌های سنجاق‌شده بلافاصله شکست می‌خورد و اعلام می‌کند ابتدا `clawhub unpin <skill>` را اجرا کنید.
- `update --all` نامک‌های سنجاق‌شده را نادیده می‌گیرد و خلاصه‌ای از مواردی که ثابت باقی مانده‌اند چاپ می‌کند.

### `skill publish <path>`

- اثر انگشت بستهٔ محلی را با ClawHub مقایسه می‌کند و وقتی
  محتوا از قبل منتشر شده باشد، با موفقیت خارج می‌شود.
- مهارت‌های جدید به‌طور پیش‌فرض `1.0.0` هستند؛ مهارت‌های تغییریافته به‌طور پیش‌فرض نسخهٔ
  وصلهٔ بعدی را دریافت می‌کنند.
- `--version <version>` نسخه‌ای را صریحاً انتخاب می‌کند و حتی وقتی
  محتوا با نسخه‌ای موجود مطابقت دارد، آن را منتشر می‌کند.
- `--dry-run` انتشار را بدون بارگذاری تعیین می‌کند؛ `--json` نتیجه‌ای
  قابل‌خواندن برای ماشین چاپ می‌کند.
- `--owner <handle>` هنگامی که عامل دسترسی ناشر داشته باشد، با شناسهٔ ناشر سازمان/کاربر
  منتشر می‌کند.
- `--migrate-owner` هنگام انتشار نسخه‌ای جدید، یک مهارت موجود را به `--owner` منتقل
  می‌کند. به دسترسی مدیر/مالک در هر دو ناشر نیاز دارد.
- رفتار مالک و بازبینی در `docs/publishing.md` توضیح داده شده است.
- انتشار یک مهارت یعنی آن مهارت تحت `MIT-0` در ClawHub عرضه می‌شود.
- استفاده، تغییر و بازتوزیع مهارت‌های منتشرشده بدون ذکر منبع آزاد است.
- ClawHub از مهارت‌های پولی یا قیمت‌گذاری به‌ازای هر مهارت پشتیبانی نمی‌کند.
- نام مستعار قدیمی: `publish <path>`.

```bash
clawhub skill publish ./my-skill --dry-run
clawhub skill publish ./my-skill
clawhub skill publish ./my-skill --version 2.0.0
```

#### GitHub Actions

گردش‌کار قابل‌استفادهٔ مجدد ClawHub با نام
[`skill-publish.yml`](https://github.com/openclaw/clawhub/blob/main/.github/workflows/skill-publish.yml)
برای یک `skill_path` یا برای هر پوشهٔ مستقیم مهارت
درون `root` (پیش‌فرض: `skills`)، `skill publish` را فراخوانی می‌کند. مهارت‌های بدون تغییر را نادیده می‌گیرد و از
همان رفتار خودکار نسخهٔ وصله استفاده می‌کند.

برای پیش‌نمایش بدون توکن، `dry_run: true` را تنظیم کنید. انتشار واقعی به
راز `clawhub_token` نیاز دارد.

### `sync`

- دایرکتوری کاری جاری، دایرکتوری مهارت‌های پیکربندی‌شده و هر
  پوشهٔ `--root <dir>` را برای یافتن پوشه‌های مهارت محلی حاوی `SKILL.md` یا
  `skill.md` پویش می‌کند.
- اثر انگشت هر مهارت محلی را با ClawHub مقایسه می‌کند و فقط مهارت‌های جدید یا
  تغییریافته را منتشر می‌کند.
- مهارت‌های جدید با نسخهٔ `1.0.0` منتشر می‌شوند؛ مهارت‌های تغییریافته به‌طور پیش‌فرض نسخهٔ وصلهٔ بعدی را
  منتشر می‌کنند. برای دسته‌های به‌روزرسانی که باید با یک گام بزرگ‌تر semver
  پیش بروند، از `--bump minor|major` استفاده کنید.
- `--dry-run` طرح انتشار را بدون بارگذاری نمایش می‌دهد؛ `--json` طرحی
  قابل‌خواندن برای ماشین چاپ می‌کند.
- `--all` هر مهارت جدید یا تغییریافته را بدون درخواست تأیید منتشر می‌کند. بدون
  `--all`، پایانه‌های تعاملی اجازه می‌دهند مهارت‌های موردنظر برای انتشار را انتخاب کنید.
- `--owner <handle>` هنگامی که عامل دسترسی ناشر داشته باشد، با شناسهٔ ناشر سازمان/کاربر
  منتشر می‌کند.
- `sync` فقط انتشار یک‌طرفه است. این فرمان نصب، به‌روزرسانی، دانلود یا
  گزارش تله‌متری نصب/دانلود را انجام نمی‌دهد.

```bash
clawhub sync --all --dry-run
clawhub sync --all
clawhub sync --root ./skills --owner openclaw --bump minor
```

### `scan --slug <slug>`

- به `clawhub login` نیاز دارد.
- ClawScan متعلق به ClawHub را از طریق `POST /api/v1/skills/-/scan` اجرا می‌کند، سپس تا نهایی‌شدن پویش نظرسنجی می‌کند.
- پویش‌ها ناهمگام هستند و ممکن است تکمیلشان زمان ببرد. هنگام قرارداشتن در صف، چرخانک پایانه موقعیت اولویت‌بندی‌شدهٔ جاری پویش و تعداد پویش‌های جلوتر را نمایش می‌دهد.
- پویش‌های منتشرشده به مالکیت یا دسترسی مدیریت ناشر نیاز دارند. ناظران/مدیران می‌توانند از طریق `clawhub-admin` از همان بک‌اند استفاده کنند.
- `--update` فقط همراه با `--slug` معتبر است؛ نتایج موفق پویش منتشرشده را در نسخهٔ انتخاب‌شده می‌نویسد.
- `--output <file.zip>` بایگانی کامل گزارش را همراه با `manifest.json`، `clawscan.json`، `skillspector.json`، `static-analysis.json`، `virustotal.json` و `README.md` دانلود می‌کند.
- `--json` پاسخ کامل نظرسنجی را برای خودکارسازی چاپ می‌کند.
- پویش مسیرهای محلی دیگر پشتیبانی نمی‌شود. نسخه‌ای جدید بارگذاری کنید، سپس از `scan download` برای بازیابی نتایج پویش ذخیره‌شدهٔ همان نسخهٔ ارسال‌شده استفاده کنید.

```bash
clawhub scan --slug gifgrep
clawhub scan --slug gifgrep --version 1.2.3
clawhub scan --slug gifgrep --update --output report.zip
```

### `scan download <name>`

- به `clawhub login` نیاز دارد.
- فایل ZIP گزارش اسکن ذخیره‌شده را برای نسخه ارسالی یک skill یا plugin دانلود می‌کند؛ از جمله نسخه‌هایی که بررسی‌های امنیتی ClawHub آن‌ها را مسدود یا پنهان کرده‌اند.
- دانلودهای skill از slug آن استفاده می‌کنند و مقدار پیش‌فرضشان `--kind skill` است.
- دانلودهای plugin از نام بسته استفاده می‌کنند و به `--kind plugin` نیاز دارند.
- `--version` الزامی است تا نویسندگان دقیقاً همان نسخه ارسالی را که ClawHub مسدود کرده است بررسی کنند.
- `--output <file.zip>` مسیر مقصد را انتخاب می‌کند.

```bash
clawhub scan download gifgrep --version 1.2.3
clawhub scan download @scope/demo --version 2.0.0 --kind plugin --output report.zip
```

#### GitHub Actions

ClawHub یک گردش‌کار رسمی و قابل‌استفاده مجدد را در
[`/.github/workflows/skill-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/skill-publish.yml)
برای مخزن‌های skill و مخزن‌های کاتالوگ ارائه می‌کند.

پیکربندی معمول کاتالوگ:

```yaml
name: Skill Publish

on:
  pull_request:
  workflow_dispatch:

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch'
    uses: openclaw/clawhub/.github/workflows/skill-publish.yml@v1
    with:
      owner: nvidia
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

نکات:

- `root` برای مخزن‌های کاتالوگ به‌طور پیش‌فرض `skills` است.
- برای پردازش یک پوشه skill، `skill_path: skills/review-helper` را ارسال کنید.
- `owner` به پرچم `--owner` در CLI نگاشت می‌شود؛ برای انتشار به‌عنوان کاربر احراز هویت‌شده، آن را حذف کنید.
- انتشار skill در V1 از `clawhub_token` استفاده می‌کند؛ انتشار قابل‌اعتماد با GitHub OIDC در حال حاضر فقط برای بسته‌ها است.

### `delete <skill>`

- بدون `--version`، یک skill را به‌صورت حذف نرم حذف می‌کند (مالک، ناظر یا مدیر).
- `DELETE /api/v1/skills/{slug}` را فراخوانی می‌کند.
- حذف‌های نرم آغازشده توسط مالک، slug را برای 30 روز رزرو می‌کنند؛ فرمان زمان انقضا را چاپ می‌کند.
- `--version <version>` یک نسخه غیرجدید متعلق به مالک را از طریق مسیری fail-closed و
  مختص نسخه پس می‌گیرد. شماره نسخه رزرو می‌ماند و نمی‌توان آن را با محتوایی
  متفاوت دوباره منتشر کرد. پیش از حذف نسخه فعلیِ جدیدتر، یک جایگزین منتشر کنید. کارکنان
  پلتفرم در این جریان مختص نسخه، مالکیت را دور نمی‌زنند.
- `--reason <text>` یک یادداشت نظارتی را در حذف نرم کل skill و گزارش ممیزی ثبت می‌کند.
- `--note <text>` نام مستعار `--reason` است.
- `--yes` تأیید را رد می‌کند.

### `undelete <skill>`

- یک skill پنهان را بازیابی می‌کند (مالک، ناظر یا مدیر).
- `POST /api/v1/skills/{slug}/undelete` را فراخوانی می‌کند.
- `--version <version>` فقط همان artifact نگه‌داری‌شده‌ای را بازیابی می‌کند که قبلاً توسط همان
  کنشگر مالک پس گرفته شده بود. نسخه بازیابی‌شده را به جدیدترین نسخه تبدیل نمی‌کند و برچسب‌های حذف‌شده را دوباره نمی‌سازد.
- بازیابی نسخه، `POST /api/v1/skills/{slug}/versions/{version}/restore` را فراخوانی می‌کند.
- `--reason <text>` یک یادداشت نظارتی را در skill و گزارش ممیزی ثبت می‌کند.
- `--note <text>` نام مستعار `--reason` است.
- `--yes` تأیید را رد می‌کند.

### `hide <skill>`

- یک skill را پنهان می‌کند (مالک، ناظر یا مدیر).
- نام مستعار `delete` است.

### `unhide <skill>`

- یک skill را از حالت پنهان خارج می‌کند (مالک، ناظر یا مدیر).
- نام مستعار `undelete` است.

### `skill rename <skill> <new-name>`

- نام یک skill متعلق به مالک را تغییر می‌دهد و slug قبلی را به‌عنوان نام مستعار تغییرمسیر نگه می‌دارد.
- `POST /api/v1/skills/{slug}/rename` را فراخوانی می‌کند.
- `--yes` تأیید را رد می‌کند.

### `skill merge <source> <target>`

- یک skill متعلق به مالک را در skill دیگری متعلق به همان مالک ادغام می‌کند.
- slug مبدأ دیگر به‌صورت عمومی فهرست نمی‌شود و به نام مستعار تغییرمسیر به مقصد تبدیل می‌شود.
- `POST /api/v1/skills/{sourceSlug}/merge` را فراخوانی می‌کند.
- `--yes` تأیید را رد می‌کند.

### `transfer`

- گردش‌کار انتقال مالکیت.
- انتقال به شناسه‌های کاربری، درخواستی در انتظار ایجاد می‌کند که گیرنده آن را می‌پذیرد.
- انتقال به شناسه‌های سازمان/ناشر تنها زمانی بلافاصله اعمال می‌شود که کنشگر
  به مالک فعلی و ناشر مقصد، دسترسی مدیریتی داشته باشد.
- زیرفرمان‌ها:
  - `transfer request <skill> <handle> [--message "..."] [--yes]`
  - `transfer list [--outgoing]`
  - `transfer accept <skill> [--yes]`
  - `transfer reject <skill> [--yes]`
  - `transfer cancel <skill> [--yes]`
- نقاط پایانی:
  - `POST /api/v1/skills/{slug}/transfer`
  - `POST /api/v1/skills/{slug}/transfer/accept`
  - `POST /api/v1/skills/{slug}/transfer/reject`
  - `POST /api/v1/skills/{slug}/transfer/cancel`
  - `GET /api/v1/transfers/incoming`
  - `GET /api/v1/transfers/outgoing`

### `package explore [query...]`

- کاتالوگ یکپارچه بسته‌ها را از طریق `GET /api/v1/packages` و `GET /api/v1/packages/search` مرور یا جست‌وجو می‌کند.
- از این مورد برای pluginها و دیگر ورودی‌های خانواده بسته استفاده کنید؛ `search` سطح بالایی همچنان رابط جست‌وجوی skill است.
- پرچم‌ها:
  - `--family skill|code-plugin|bundle-plugin`
  - `--official`
  - `--executes-code`
  - `--target <target>`، `--os <os>`، `--arch <arch>`، `--libc <libc>`
  - `--requires-browser`، `--requires-desktop`، `--requires-native-deps`
  - `--requires-external-service`، `--external-service <name>`
  - `--binary <name>`، `--os-permission <name>`
  - `--artifact-kind legacy-zip|npm-pack`
  - `--npm-mirror`
  - `--limit <n>` (1-100، پیش‌فرض: 25)
  - `--json`

مثال‌ها:

```bash
clawhub package explore --family code-plugin
clawhub package explore --family code-plugin --os darwin --requires-desktop
clawhub package explore --family code-plugin --artifact-kind npm-pack
clawhub package explore --npm-mirror
clawhub package explore episodic-claw --family code-plugin
```

### `package inspect <name>`

- فراداده بسته را بدون نصب دریافت می‌کند.
- از این مورد برای بررسی فراداده، سازگاری، تأیید، منبع و نسخه/فایل plugin استفاده کنید.
- `--version <version>`: یک نسخه مشخص را بررسی می‌کند (پیش‌فرض: جدیدترین).
- `--tag <tag>`: یک نسخه برچسب‌خورده را بررسی می‌کند (برای مثال `latest`).
- `--versions`: تاریخچه نسخه‌ها را فهرست می‌کند (صفحه نخست).
- `--limit <n>`: حداکثر تعداد نسخه‌ها برای فهرست‌کردن (1-100).
- `--files`: فایل‌های نسخه انتخاب‌شده را فهرست می‌کند.
- `--file <path>`: یک پیش‌نمایش متنی محدودشده UTF-8 را دریافت می‌کند (محدودیت 200KB).
- `--json`: خروجی قابل‌خواندن برای ماشین.

### `package download <name>`

- نسخه یک بسته را از طریق
  `GET /api/v1/packages/{name}/versions/{version}/artifact` تفکیک می‌کند.
- artifact را از `downloadUrl` تفکیک‌کننده دانلود می‌کند.
- مقدار SHA-256 متعلق به ClawHub را برای همه artifactها تأیید می‌کند.
- برای artifactهای npm-pack در ClawPack، یکپارچگی `sha512` در npm،
  shasum در npm و نام/نسخه `package.json` در tarball را نیز تأیید می‌کند.
- نسخه‌های ZIP قدیمی از طریق مسیر قدیمی ZIP دانلود می‌شوند.
- پرچم‌ها:
  - `--version <version>`: یک نسخه مشخص را دانلود می‌کند.
  - `--tag <tag>`: یک نسخه برچسب‌خورده را دانلود می‌کند (پیش‌فرض: `latest`).
  - `-o, --output <path>`: فایل یا پوشه خروجی.
  - `--force`: یک فایل خروجی موجود را بازنویسی می‌کند.
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال‌ها:

```bash
clawhub package download @openclaw/example-plugin --tag latest
clawhub package download @openclaw/example-plugin --version 1.2.3 -o artifacts/
```

### `package verify <file>`

- مقدار SHA-256 متعلق به ClawHub، یکپارچگی `sha512` در npm و shasum در npm را برای یک
  artifact محلی محاسبه می‌کند.
- با `--package`، فراداده مورد انتظار را از ClawHub تفکیک و فایل
  محلی را با فراداده artifact منتشرشده مقایسه می‌کند.
- با پرچم‌های مستقیم digest، بدون جست‌وجوی شبکه‌ای تأیید می‌کند.
- پرچم‌ها:
  - `--package <name>`: نام بسته برای تفکیک فراداده مورد انتظار artifact.
  - `--version <version>` یا `--tag <tag>`: نسخه مورد انتظار بسته.
  - `--sha256 <hex>`: مقدار SHA-256 مورد انتظار ClawHub.
  - `--npm-integrity <sri>`: یکپارچگی مورد انتظار npm.
  - `--npm-shasum <sha1>`: shasum مورد انتظار npm.
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال‌ها:

```bash
clawhub package verify ./example-plugin-1.2.3.tgz --package @openclaw/example-plugin --version 1.2.3
clawhub package verify ./example-plugin-1.2.3.tgz --sha256 <hex>
```

### `package validate <source>`

- Plugin Inspector همراه CLI در ClawHub را روی پوشه محلی بسته plugin
  اجرا می‌کند.
- مقدار پیش‌فرض، اعتبارسنجی آفلاین/ایستا است؛ بدون یافتن یا واردکردن یک checkout محلی
  از OpenClaw.
- خطاهای قطعی سازگاری با کد غیرصفر خارج می‌شوند. یافته‌هایی که فقط هشدار هستند چاپ می‌شوند، اما
  با کد صفر خارج می‌شوند.
- پرچم‌ها:
  - `--out <dir>`: گزارش‌های Plugin Inspector را در این پوشه می‌نویسد.
  - `--openclaw <path>`: در برابر یک checkout محلی و صریح از OpenClaw بررسی می‌کند.
  - `--runtime`: ثبت زمان اجرا را فعال می‌کند؛ کد plugin را وارد می‌کند.
  - `--allow-execute`: ثبت زمان اجرا را در یک فضای کاری ایزوله مجاز می‌کند.
  - `--no-mock-sdk`: SDK شبیه‌سازی‌شده OpenClaw را هنگام ثبت زمان اجرا غیرفعال می‌کند.
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال:

```bash
clawhub package validate ./example-plugin
```

اگر اعتبارسنجی یافته‌ای درباره بسته، مانیفست، واردکردن SDK یا artifact گزارش کرد، به
[رفع مشکلات اعتبارسنجی Plugin](/fa/clawhub/plugin-validation-fixes) مراجعه کنید و سپس فرمان را دوباره اجرا کنید.

### `package delete <name>`

- بدون `--version`، یک بسته و همه انتشارهای آن را به‌صورت حذف نرم حذف می‌کند.
- `--version <version>` یک انتشار غیرجدید متعلق به مالک را از طریق مسیری fail-closed و
  مختص نسخه پس می‌گیرد. شماره نسخه رزرو می‌ماند و نمی‌توان آن را با
  محتوایی متفاوت دوباره منتشر کرد. پیش از حذف نسخه فعلیِ جدیدتر، یک جایگزین منتشر کنید. این
  جریان مختص نسخه به مالک بسته یا مدیر ناشر سازمانی نیاز دارد؛ کارکنان پلتفرم
  مالکیت بسته را دور نمی‌زنند.
- حذف نرم کل بسته به مالک بسته، مالک/مدیر ناشر سازمانی، ناظر
  پلتفرم یا مدیر پلتفرم نیاز دارد.
- پرچم‌ها:
  - `--version <version>`: یک نسخه غیرجدید را پس می‌گیرد.
  - `--yes`: تأیید را رد می‌کند.
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال:

```bash
clawhub package delete @openclaw/example-plugin --yes
clawhub package delete @openclaw/example-plugin --version 1.2.3 --yes
```

### `package undelete <name>`

- یک بسته حذف‌شده به‌صورت نرم و انتشارهای آن را بازیابی می‌کند.
- به مالک بسته، مالک/مدیر ناشر سازمانی، ناظر پلتفرم
  یا مدیر پلتفرم نیاز دارد.
- `POST /api/v1/packages/{name}/undelete` را فراخوانی می‌کند.
- `--version <version>` فقط همان انتشار نگه‌داری‌شده‌ای را بازیابی می‌کند که قبلاً توسط همان
  کنشگر مالک پس گرفته شده بود. انتشار بازیابی‌شده را به جدیدترین انتشار تبدیل نمی‌کند و برچسب‌های بسته/dist-tagهای حذف‌شده را دوباره نمی‌سازد.
- بازیابی نسخه، `POST /api/v1/packages/{name}/versions/{version}/restore` را فراخوانی می‌کند.
- پرچم‌ها:
  - `--version <version>`: یک انتشار پس‌گرفته‌شده توسط مالک را بازیابی می‌کند.
  - `--yes`: تأیید را رد می‌کند.
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال:

```bash
clawhub package undelete @openclaw/example-plugin --yes
```

### `package transfer <name>`

- یک بسته را به ناشر دیگری منتقل می‌کند.
- به دسترسی مدیر برای مالک فعلی بسته و ناشر مقصد نیاز دارد،
  مگر اینکه مدیر پلتفرم آن را انجام دهد.
- نام بسته‌های دارای دامنه باید به مالک دامنهٔ منطبق منتقل شوند.
- `POST /api/v1/packages/{name}/transfer` را فراخوانی می‌کند.
- پرچم‌ها:
  - `--to <owner>`: شناسهٔ ناشر مقصد.
  - `--reason <text>`: دلیل اختیاری ممیزی.
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال:

```bash
clawhub package transfer @openclaw/example-plugin --to openclaw
```

### `package report`

- فرمان احراز هویت‌شده برای گزارش یک بسته به ناظران.
- `POST /api/v1/packages/{name}/report` را فراخوانی می‌کند.
- گزارش‌ها در سطح بسته هستند، می‌توانند به‌صورت اختیاری به یک نسخه مرتبط شوند و برای
  بازبینی در معرض دید ناظران قرار می‌گیرند.
- گزارش‌ها به‌خودی‌خود بسته‌ها را پنهان نمی‌کنند یا جلوی بارگیری را نمی‌گیرند.
- پرچم‌ها:
  - `--version <version>`: نسخهٔ اختیاری بسته برای پیوست‌کردن به گزارش.
  - `--reason <text>`: دلیل الزامی گزارش.
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال:

```bash
clawhub package report @openclaw/example-plugin --version 1.2.3 --reason "suspicious native payload"
```

### `package moderation-status`

- فرمان مالک برای بررسی وضعیت نمایش بسته در نظارت.
- `GET /api/v1/packages/{name}/moderation` را فراخوانی می‌کند.
- وضعیت فعلی اسکن بسته، تعداد گزارش‌های باز، وضعیت نظارت دستی آخرین انتشار،
  وضعیت مسدودبودن بارگیری و دلایل نظارت را نمایش می‌دهد.
- پرچم‌ها:
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال:

```bash
clawhub package moderation-status @openclaw/example-plugin
```

### `package readiness <name>`

- بررسی می‌کند که آیا یک بسته برای استفادهٔ آتی OpenClaw آماده است.
- `GET /api/v1/packages/{name}/readiness` را فراخوانی می‌کند.
- موانع مربوط به وضعیت رسمی، دسترس‌پذیری ClawPack، چکیدهٔ دست‌ساخته،
  منشأ منبع، سازگاری OpenClaw، اهداف میزبان، فرادادهٔ محیط
  و وضعیت اسکن را گزارش می‌کند.
- پرچم‌ها:
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال:

```bash
clawhub package readiness @openclaw/example-plugin
```

### `package migration-status <name>`

- وضعیت مهاجرت معطوف به اپراتور را برای بسته‌ای نمایش می‌دهد که ممکن است جایگزین
  یک plugin همراه OpenClaw شود.
- همان نقطهٔ پایانی آمادگی محاسبه‌شدهٔ `package readiness` را فراخوانی می‌کند، اما
  وضعیت متمرکز بر مهاجرت، آخرین نسخه، وضعیت بستهٔ رسمی، بررسی‌ها و
  موانع را چاپ می‌کند.
- پرچم‌ها:
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال:

```bash
clawhub package migration-status @openclaw/example-plugin
```

### `publisher create <handle>`

- یک ناشر سازمانی متعلق به کاربر احراز هویت‌شده ایجاد می‌کند.
- شناسه به حروف کوچک نرمال‌سازی می‌شود و می‌توان آن را با یا بدون `@` وارد کرد.
- ناشران سازمانی تازه‌ایجادشده به‌طور پیش‌فرض مورد اعتماد یا رسمی نیستند.
- اگر شناسه از قبل توسط ناشر یا کاربری موجود یا یک مسیر رزروشده استفاده شده باشد، ناموفق می‌شود.

```bash
clawhub publisher create opik --display-name "Opik"
```

### `package publish <source>`

- یک plugin کد یا plugin بسته‌ای را از طریق `POST /api/v1/packages` منتشر می‌کند.
- `<source>` موارد زیر را می‌پذیرد:
  - مسیر پوشهٔ محلی: `./my-plugin`
  - تربال محلی npm-pack مربوط به ClawPack: `./my-plugin-1.2.3.tgz`
  - مخزن GitHub: `owner/repo` یا `owner/repo@ref`
  - نشانی GitHub: `https://github.com/owner/repo`
- فراداده به‌طور خودکار از `package.json`، `openclaw.plugin.json` و
  نشانگرهای واقعی بستهٔ OpenClaw مانند `.codex-plugin/plugin.json`،
  `.claude-plugin/plugin.json` و `.cursor-plugin/plugin.json` شناسایی می‌شود.
- منابع `.tgz` به‌عنوان ClawPack در نظر گرفته می‌شوند. CLI بایت‌های دقیق npm-pack
  را بارگذاری می‌کند و از محتوای استخراج‌شدهٔ `package/` فقط برای اعتبارسنجی و
  تکمیل اولیهٔ فراداده استفاده می‌کند.
- پوشه‌های plugin کد پیش از بارگذاری در یک تربال npm مربوط به ClawPack بسته‌بندی می‌شوند تا
  نصب‌های OpenClaw بتوانند دست‌ساختهٔ دقیق را تأیید کنند. پوشه‌های plugin بسته‌ای همچنان
  از مسیر انتشار فایل‌های استخراج‌شده استفاده می‌کنند.
- برای منابع GitHub، انتساب منبع به‌طور خودکار از مخزن، کامیت حل‌شده، ارجاع و زیرمسیر تکمیل می‌شود.
- برای پوشه‌های محلی، هنگامی که ریموت مبدأ به GitHub اشاره کند، انتساب منبع به‌طور خودکار از git محلی شناسایی می‌شود.
- pluginهای کد خارجی باید `openclaw.compat.pluginApi` و
  `openclaw.build.openclawVersion` را صریحاً اعلام کنند.
  `package.json.version` سطح بالا به‌عنوان جایگزین اعتبارسنجی انتشار استفاده نمی‌شود.
- `--dry-run` محمولهٔ انتشار حل‌شده را بدون بارگذاری پیش‌نمایش می‌کند.
- `--json` خروجی قابل‌خواندن برای ماشین را برای CI تولید می‌کند.
- `--owner <handle>` هنگامی که عامل به ناشر دسترسی دارد، انتشار را با شناسهٔ ناشر کاربری یا سازمانی انجام می‌دهد.
- نام بسته‌های دارای دامنه باید با مالک انتخاب‌شده منطبق باشند. `docs/publishing.md` را ببینید.
- پرچم‌های موجود (`--family`، `--name`، `--version`، `--source-repo`، `--source-commit`، `--source-ref`، `--source-path`) همچنان به‌عنوان بازنویسی کار می‌کنند.
- مخزن‌های خصوصی GitHub به `GITHUB_TOKEN` نیاز دارند.

```bash
clawhub package publish ./plugin.tgz --owner openclaw
```

#### جریان محلی پیشنهادی

ابتدا از `--dry-run` استفاده کنید تا بتوانید پیش از ایجاد یک انتشار زنده، فرادادهٔ حل‌شدهٔ بسته و
انتساب منبع را تأیید کنید:

```bash
npm pack
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin --dry-run
clawhub package publish ./my-plugin-1.2.3.tgz --family code-plugin
```

#### جریان پوشهٔ محلی

برای pluginهای کد، انتشار پوشه یک دست‌ساختهٔ ClawPack را از
پوشهٔ بسته می‌سازد و بارگذاری می‌کند:

```bash
clawhub package publish ./my-plugin --family code-plugin --dry-run
clawhub package publish ./my-plugin --family code-plugin
```

#### `package.json` حداقلی برای `--family code-plugin`

pluginهای کد خارجی به مقدار کمی فرادادهٔ OpenClaw در
`package.json` نیاز دارند. این مانیفست حداقلی برای یک انتشار موفق کافی است:

```json
{
  "name": "@myorg/openclaw-my-plugin",
  "version": "1.0.0",
  "type": "module",
  "openclaw": {
    "extensions": ["./index.ts"],
    "compat": {
      "pluginApi": ">=2026.3.24-beta.2"
    },
    "build": {
      "openclawVersion": "2026.3.24-beta.2"
    }
  }
}
```

فیلدهای الزامی:

- `openclaw.compat.pluginApi`
- `openclaw.build.openclawVersion`

نکته‌ها:

- `package.json.version` نسخهٔ انتشار بستهٔ شما است، اما به‌عنوان
  جایگزین اعتبارسنجی سازگاری/ساخت OpenClaw استفاده نمی‌شود.
- `openclaw.hostTargets` و `openclaw.environment` فرادادهٔ اختیاری هستند.
  ClawHub ممکن است در صورت وجود آن‌ها را نمایش دهد، اما برای انتشار الزامی نیستند.
- `openclaw.compat.minGatewayVersion` و
  `openclaw.build.pluginSdkVersion` افزوده‌های اختیاری برای زمانی هستند که بخواهید
  فرادادهٔ سازگاری دقیق‌تری منتشر کنید.
- اگر از نسخهٔ قدیمی‌تر CLI مربوط به `clawhub` استفاده می‌کنید، پیش از انتشار آن را ارتقا دهید تا
  بررسی‌های اولیهٔ محلی پیش از بارگذاری اجرا شوند.
- اگر اعتبارسنجی یک کد اصلاحی گزارش کرد،
  [اصلاحات اعتبارسنجی plugin](/fa/clawhub/plugin-validation-fixes) را ببینید.

#### GitHub Actions

ClawHub همچنین یک گردش‌کار رسمی و قابل‌استفادهٔ مجدد در
[`/.github/workflows/package-publish.yml`](https://github.com/openclaw/clawhub/blob/62a697ef1e1b623afd71cf8813b545487a17354f/.github/workflows/package-publish.yml)
برای مخزن‌های plugin ارائه می‌کند.

پیکربندی معمول فراخوان:

```yaml
name: Package Publish

on:
  pull_request:
  workflow_dispatch:
  push:
    tags:
      - "v*"

jobs:
  dry-run:
    if: github.event_name == 'pull_request'
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: true

  publish:
    if: github.event_name == 'workflow_dispatch' || startsWith(github.ref, 'refs/tags/')
    permissions:
      contents: read
      id-token: write
    uses: openclaw/clawhub/.github/workflows/package-publish.yml@v0.12.0
    with:
      dry_run: false
    secrets:
      clawhub_token: ${{ secrets.CLAWHUB_TOKEN }}
```

نکته‌ها:

- مقدار پیش‌فرض `source` در گردش‌کار قابل‌استفادهٔ مجدد، مخزن فراخوان است.
- برای تک‌مخزن‌ها، `source_path` را وارد کنید تا گردش‌کار پوشهٔ بستهٔ plugin
  را منتشر کند؛ برای مثال `source_path: extensions/codex`.
- گردش‌کار قابل‌استفادهٔ مجدد را به یک برچسب پایدار یا SHA کامل کامیت سنجاق کنید. انتشار نسخه را از `@main` اجرا نکنید.
- `pull_request` باید از `dry_run: true` استفاده کند تا CI اثری بر محیط انتشار نگذارد.
- انتشارهای واقعی باید به رویدادهای قابل‌اعتماد مانند `workflow_dispatch` یا پوش‌کردن برچسب محدود شوند.
- انتشار قابل‌اعتماد بدون راز فقط روی `workflow_dispatch` کار می‌کند؛ پوش‌کردن برچسب همچنان به `clawhub_token` نیاز دارد.
- `clawhub_token` را برای نخستین انتشار، بسته‌های غیرقابل‌اعتماد یا انتشارهای اضطراری در دسترس نگه دارید.
- گردش‌کار نتیجهٔ JSON را به‌عنوان دست‌ساخته بارگذاری می‌کند و آن را به‌صورت خروجی‌های گردش‌کار در دسترس قرار می‌دهد.

### `package trusted-publisher get <name>`

- پیکربندی ناشر قابل‌اعتماد GitHub Actions را برای یک بسته نمایش می‌دهد.
- پس از تنظیم پیکربندی، از این فرمان برای تأیید مخزن، نام فایل گردش‌کار
  و سنجاق اختیاری محیط استفاده کنید.
- پرچم‌ها:
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال:

```bash
clawhub package trusted-publisher get @openclaw/example-plugin
```

### `package trusted-publisher set <name>`

- پیکربندی ناشر قابل‌اعتماد GitHub Actions را به یک بستهٔ موجود متصل یا جایگزین می‌کند.
- بسته ابتدا باید از طریق `clawhub package publish` عادی با احراز هویت دستی یا مبتنی بر توکن
  ایجاد شود.
- پس از تنظیم پیکربندی، انتشارهای پشتیبانی‌شدهٔ آتی GitHub Actions می‌توانند
  بدون توکن بلندمدت ClawHub از انتشار OIDC/قابل‌اعتماد استفاده کنند.
- `--repository <repo>` باید `owner/repo` باشد.
- `--workflow-filename <file>` باید با نام فایل گردش‌کار در
  `.github/workflows/` مطابقت داشته باشد.
- `--environment <name>` اختیاری است. در صورت پیکربندی، محیط GitHub Actions
  در ادعای OIDC باید دقیقاً مطابقت داشته باشد.
- ClawHub هنگام اجرای این فرمان، مخزن پیکربندی‌شدهٔ GitHub را تأیید می‌کند.
  مخزن‌های عمومی را می‌توان از طریق فرادادهٔ عمومی GitHub تأیید کرد. برای مخزن‌های
  خصوصی، ClawHub باید به آن مخزن دسترسی GitHub داشته باشد؛ برای
  مثال، از طریق نصب آتی GitHub App مربوط به ClawHub یا یک یکپارچه‌سازی مجاز دیگر
  با GitHub.
- پرچم‌ها:
  - `--repository <repo>`: مخزن GitHub، برای مثال `openclaw/example-plugin`.
  - `--workflow-filename <file>`: نام فایل گردش‌کار، برای مثال `package-publish.yml`.
  - `--environment <name>`: محیط اختیاری GitHub Actions با تطابق دقیق.
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال:

```bash
clawhub package trusted-publisher set @openclaw/example-plugin \
  --repository openclaw/example-plugin \
  --workflow-filename package-publish.yml \
  --environment release
```

### `package trusted-publisher delete <name>`

- پیکربندی ناشر قابل‌اعتماد را از یک بسته حذف می‌کند.
- اگر لازم است سنجاق گردش‌کار، مخزن یا محیط غیرفعال یا دوباره ایجاد شود،
  از این فرمان برای بازگردانی استفاده کنید.
- تا زمانی که پیکربندی دوباره تنظیم شود، انتشارهای واقعی آتی باید از انتشار عادی احراز هویت‌شده
  استفاده کنند.
- پرچم‌ها:
  - `--json`: خروجی قابل‌خواندن برای ماشین.

مثال:

```bash
clawhub package trusted-publisher delete @openclaw/example-plugin
```

### تله‌متری نصب

- هنگام ورود به سیستم، پس از `clawhub install <slug>` ارسال می‌شود، مگر اینکه
  `CLAWHUB_DISABLE_TELEMETRY=1` تنظیم شده باشد.
- گزارش‌دهی بر مبنای بهترین تلاش است. اگر تله‌متری در دسترس نباشد،
  فرمان‌های نصب ناموفق نمی‌شوند.
- جزئیات: `docs/telemetry.md`.
