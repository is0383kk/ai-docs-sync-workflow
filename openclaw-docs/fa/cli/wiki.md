---
read_when:
    - می‌خواهید از CLI مربوط به memory-wiki استفاده کنید
    - شما در حال مستندسازی یا تغییر `openclaw wiki` هستید
summary: مرجع CLI برای `openclaw wiki` (وضعیت مخزن memory-wiki، جست‌وجو، کامپایل، لینت، اعمال، پل ارتباطی، درون‌ریزی ChatGPT و ابزارهای کمکی Obsidian)
title: ویکی
x-i18n:
    generated_at: "2026-07-27T16:25:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1f793d52de270068cf3a06b13f52242bb66738235718639486e090a2de213e73
    source_path: cli/wiki.md
    workflow: 16
---

# `openclaw wiki`

مخزن `memory-wiki` را بررسی و نگهداری کنید. این قابلیت توسط Plugin اختیاری همراهِ `memory-wiki` ارائه می‌شود. پیش از نخستین استفاده، آن را فعال کنید:

```bash
openclaw plugins enable memory-wiki
openclaw gateway restart
```

مرتبط: [Plugin ویکی حافظه](/fa/plugins/memory-wiki)، [نمای کلی حافظه](/fa/concepts/memory)، [CLI: حافظه](/fa/cli/memory)

## فرمان‌های رایج

```bash
openclaw wiki status
openclaw wiki doctor
openclaw wiki init
openclaw wiki ingest ./notes/alpha.md
openclaw wiki okf import ./knowledge-catalog/okf/bundles/ga4
openclaw wiki compile
openclaw wiki lint
openclaw wiki search "alpha"
openclaw wiki search "who should I ask about Teams?" --mode route-question
openclaw wiki get entity.alpha --from 1 --lines 80

openclaw wiki apply synthesis "Alpha Summary" \
  --body "Short synthesis body" \
  --source-id source.alpha

openclaw wiki apply metadata entity.alpha \
  --source-id source.alpha \
  --status review \
  --question "Still active?"

openclaw wiki bridge import
openclaw wiki unsafe-local import
openclaw wiki chatgpt import --export ./chatgpt-export --dry-run
openclaw wiki chatgpt rollback <run-id>

openclaw wiki obsidian status
openclaw wiki obsidian search "alpha"
openclaw wiki obsidian open syntheses/alpha-summary.md
openclaw wiki obsidian command workspace:quick-switcher
openclaw wiki obsidian daily
```

## انتخاب عامل

وقتی `plugins.entries.memory-wiki.config.vault.scope` برابر با `agent` است، مخزن را با گزینهٔ سطح‌بالای `--agent <id>` انتخاب کنید:

```bash
openclaw wiki --agent support status
openclaw wiki --agent support search "refund policy"
openclaw wiki --agent marketing ingest ./campaign-notes.md
```

در راه‌اندازی‌ای با چند عامل پیکربندی‌شده، `--agent` برای عملیات CLI الزامی است تا هیچ فرمانی نتواند مخزن پیش‌فرض دلخواهی را بخواند یا بنویسد. اگر فقط یک عامل پیکربندی شده باشد، همان عامل پیش‌فرض باقی می‌ماند. شناسه‌های ناشناختهٔ عامل پیش از آغاز عملیات مخزن با خطا مواجه می‌شوند. وقتی `vault.scope` برابر با `global` است، این گزینه مسیر انتخاب‌شده را تغییر نمی‌دهد.

کلاینت‌های Gateway نیز از همین قاعده پیروی می‌کنند: در یک راه‌اندازی چندعاملی با محدودهٔ عامل، برای درخواست‌های `wiki.*` مبتنی بر مخزن، `agentId` را ارسال کنید. شناسهٔ مفقود یا ناشناخته خطا محسوب می‌شود. نوبت‌های عامل، ابزارهای ویکی، مکمل‌های پیکرهٔ حافظه و چکیده‌های کامپایل‌شدهٔ پرامپت از قبل زمینهٔ عامل فعال زمان اجرا را با خود حمل می‌کنند.

## فرمان‌ها

### `wiki status`

حالت و محدودهٔ مخزن، عامل حل‌شده، سلامت و دسترس‌پذیری CLI مربوط به Obsidian را نمایش می‌دهد. ابتدا از این فرمان استفاده کنید تا بررسی شود آیا مخزن موردنظر مقداردهی اولیه شده، حالت پل سالم است یا یکپارچه‌سازی Obsidian در دسترس است.

وقتی حالت پل فعال است و برای خواندن مصنوعات حافظه پیکربندی شده، این فرمان Gateway در حال اجرا را پرس‌وجو می‌کند تا همان زمینهٔ Plugin حافظهٔ فعال در حافظهٔ عامل/زمان اجرا را ببیند.

### `wiki doctor`

بررسی‌های سلامت ویکی را اجرا و اصلاحات عملی را گزارش می‌کند. در صورت ناسالم‌بودن با کد غیرصفر خارج می‌شود.

وقتی حالت پل فعال است و برای خواندن مصنوعات حافظه پیکربندی شده، این فرمان پیش از ساخت گزارش Gateway در حال اجرا را پرس‌وجو می‌کند. واردسازی‌های غیرفعال پل و پیکربندی‌های پلی که مصنوعات حافظه را نمی‌خوانند، محلی/آفلاین باقی می‌مانند.

مشکلات معمول:

- حالت پل بدون مصنوعات عمومی حافظه فعال شده است
- چیدمان مخزن نامعتبر یا مفقود است
- وقتی حالت Obsidian مورد انتظار است، CLI خارجی Obsidian وجود ندارد

### `wiki init`

چیدمان مخزن ویکی و صفحه‌های آغازین، از جمله نمایه‌های سطح‌بالا و دایرکتوری‌های کش را ایجاد می‌کند.

### `wiki ingest <path>`

یک فایل محلی Markdown یا متنی را به‌عنوان صفحهٔ منبع در پوشهٔ `sources/` ویکی وارد می‌کند. `<path>` باید مسیر یک فایل محلی باشد؛ در حال حاضر واردسازی از URL وجود ندارد. فایل‌های دودویی رد می‌شوند.

صفحه‌های منبع واردشده فرادادهٔ منشأ (`sourceType: local-file`، `sourcePath`، `ingestedAt`) را حمل می‌کنند. پس از واردسازی، مخزن همیشه دوباره کامپایل می‌شود.

پرچم‌ها: `--title <title>` عنوان منبع را بازنویسی می‌کند (پیش‌فرض: برگرفته از نام فایل).

### `wiki okf import <path>`

یک بستهٔ استخراج‌شدهٔ Open Knowledge Format را به صفحه‌های مفهومی ویکی وارد می‌کند.

واردکننده همهٔ سندهای مفهومی غیررزروشدهٔ `.md` را در درخت دایرکتوری OKF می‌خواند، وجود یک فیلد غیرخالی `type` را الزامی می‌داند و مقادیر ناشناختهٔ `type` در OKF را به‌عنوان مفاهیم عمومی در نظر می‌گیرد. فایل‌های رزروشدهٔ `index.md` و `log.md` در OKF به‌عنوان مفهوم وارد نمی‌شوند.

صفحه‌های واردشده زیر `concepts/` مسطح می‌شوند تا جریان‌های فعلی کامپایل، جست‌وجو، دریافت، چکیده و داشبورد ویکی فوراً آن‌ها را ببینند. شناسهٔ اصلی مفهوم OKF، `type`، `resource`، `tags`، مُهر زمانی، مسیر منبع و فرادادهٔ کامل در فرادادهٔ صفحه حفظ می‌شوند. پیوندهای داخلی Markdown در OKF به صفحه‌های ویکی تولیدشده بازنویسی می‌شوند؛ پیوندهای خراب یا خارجی بدون تغییر باقی می‌مانند. پس از واردسازی، مخزن همیشه دوباره کامپایل می‌شود.

مثال‌ها:

```bash
openclaw wiki okf import ./bundles/ga4
openclaw wiki okf import ./bundles/ga4 --json
openclaw wiki search "BigQuery Table" --mode source-evidence --json
openclaw wiki get <path-from-json-result>
```

### `wiki compile`

نمایه‌ها، بلوک‌های مرتبط، داشبوردها و عکس فوری کامپایل‌شدهٔ پرس‌وجو/پرامپت را بازسازی می‌کند. عکس فوری در وضعیت SQLite اشتراکی Plugin متعلق به OpenClaw ذخیره و برای تصویرسازی همگام پرامپت در حافظه نگه داشته می‌شود؛ این فرایند در مخزن فایل کش ایجاد نمی‌کند.

اگر `render.createDashboards` فعال باشد، کامپایل صفحه‌های گزارش را نیز تازه‌سازی می‌کند.

### `wiki lint`

مخزن را لینت می‌کند و گزارشی شامل موارد زیر می‌نویسد:

- مشکلات ساختاری (پیوندهای خراب، شناسه‌های مفقود/تکراری، نوع یا عنوان مفقود صفحه، فرادادهٔ نامعتبر)
- شکاف‌های منشأ (شناسه‌های مفقود منبع، منشأ مفقود واردسازی)
- تناقض‌ها (تناقض‌های علامت‌گذاری‌شده، ادعاهای متعارض)
- پرسش‌های باز
- صفحه‌ها و ادعاهای کم‌اطمینان
- صفحه‌ها و ادعاهای قدیمی

این فرمان را پس از به‌روزرسانی‌های معنادار ویکی اجرا کنید.

### `wiki search <query>`

محتوای ویکی را جست‌وجو می‌کند. رفتار به پیکربندی بستگی دارد:

- `search.backend`: `shared` یا `local`
- `search.corpus`: `wiki`، `memory` یا `all`
- `--mode`: `auto`، `find-person`، `route-question`، `source-evidence` یا `raw-claim`

برای رتبه‌بندی و منشأ ویژهٔ ویکی از `wiki search` استفاده کنید. برای یک گذر گستردهٔ مشترک بازیابی، وقتی Plugin حافظهٔ فعال جست‌وجوی اشتراکی ارائه می‌کند، `openclaw memory search` را ترجیح دهید.

حالت‌های جست‌وجو:

- `find-person`: نام‌های مستعار، شناسه‌های کاربری، شبکه‌های اجتماعی، شناسه‌های متعارف و صفحه‌های اشخاص
- `route-question`: راهنمایی‌های «از چه کسی بپرسید»/«بهترین کاربرد» و زمینهٔ رابطه
- `source-evidence`: صفحه‌های منبع و فیلدهای ساختاریافتهٔ شواهد
- `raw-claim`: متن ساختاریافتهٔ ادعا همراه با فرادادهٔ ادعا/شواهد

مثال‌ها:

```bash
openclaw wiki search "bgroux" --mode find-person
openclaw wiki search "who knows Teams rollout?" --mode route-question
openclaw wiki search "maintainer-whois" --mode source-evidence
openclaw wiki search "strong route Teams" --mode raw-claim --json
```

وقتی نتیجه‌ای با یک ادعای ساختاریافته مطابقت داشته باشد، خروجی متنی شامل خطوط `Claim:` و `Evidence:` است. خروجی JSON افزون بر این، `matchedClaimId`، `matchedClaimStatus`، `matchedClaimConfidence`، `evidenceKinds` و `evidenceSourceIds` را برای واکاوی بیشتر در سمت عامل ارائه می‌کند.

### `wiki get <lookup>`

یک صفحهٔ ویکی را با شناسه یا مسیر نسبی می‌خواند.

```bash
openclaw wiki get entity.alpha
openclaw wiki get syntheses/alpha-summary.md --from 1 --lines 80
```

### `wiki apply`

تغییرات محدود را بدون دست‌کاری آزادانهٔ صفحه اعمال می‌کند:

- `apply synthesis <title>`: ایجاد یا تازه‌سازی یک صفحهٔ جمع‌بندی با بدنهٔ خلاصهٔ مدیریت‌شده
- `apply metadata <lookup>`: به‌روزرسانی فرادادهٔ یک صفحهٔ موجود

هر دو `--source-id`، `--contradiction`، `--question` (هرکدام قابل تکرار)، `--confidence <n>` (0-1) و `--status <status>` را می‌پذیرند. `apply metadata` همچنین `--clear-confidence` را برای حذف مقدار اطمینان ذخیره‌شده می‌پذیرد. این روش پشتیبانی‌شده برای تکامل صفحه‌های ویکی است تا بلوک‌های تولیدشدهٔ مدیریت‌شده دست‌نخورده باقی بمانند.

### `wiki bridge import`

مصنوعات عمومی حافظه را از Plugin حافظهٔ فعال به صفحه‌های منبع مبتنی بر پل وارد می‌کند. در حالت `bridge` از این فرمان برای کشیدن جدیدترین مصنوعات صادرشدهٔ حافظه به مخزن ویکی استفاده کنید.

برای خواندن مصنوعات فعال پل، CLI واردسازی را از طریق RPC مربوط به Gateway مسیریابی می‌کند تا از زمینهٔ Plugin حافظه در زمان اجرا استفاده شود. اگر واردسازی‌های پل غیرفعال باشند یا خواندن مصنوعات خاموش باشد، فرمان رفتار محلی/آفلاین با صفر واردسازی را حفظ می‌کند. تازه‌سازی نمایه پس از واردسازی توسط `ingest.autoCompile` کنترل می‌شود.

### `wiki unsafe-local import`

در حالت `unsafe-local` از مسیرهای محلی صریحاً پیکربندی‌شده (`unsafeLocal.paths`) وارد می‌کند. عمداً آزمایشی است و فقط روی همان دستگاه کار می‌کند. تازه‌سازی نمایه پس از واردسازی توسط `ingest.autoCompile` کنترل می‌شود.

### `wiki chatgpt import`

یک خروجی ChatGPT را به صفحه‌های منبع پیش‌نویس ویکی وارد می‌کند.

```bash
openclaw wiki chatgpt import --export ./chatgpt-export
openclaw wiki chatgpt import --export ./conversations.json --dry-run
```

| پرچم              | پیش‌فرض    | توضیحات                                                   |
| ----------------- | ---------- | ------------------------------------------------------------- |
| `--export <path>` | (الزامی) | دایرکتوری خروجی ChatGPT یا مسیر `conversations.json`.        |
| `--dry-run`       | `false`    | پیش‌نمایش تعداد موارد ایجادشده/به‌روزشده/ردشده بدون نوشتن صفحه‌ها. |

یک واردسازی غیرآزمایشی که صفحه‌ای را تغییر دهد، شناسهٔ اجرای واردسازی را ثبت می‌کند؛ این شناسه در خلاصه چاپ می‌شود و برای بازگردانی لازم است.

### `wiki chatgpt rollback <run-id>`

یک اجرای واردسازی ChatGPT را که قبلاً اعمال شده بازمی‌گرداند؛ صفحه‌های ایجادشدهٔ آن را حذف و صفحه‌هایی را که بازنویسی کرده بود بازیابی می‌کند. اگر اجرا قبلاً بازگردانده شده باشد، هیچ عملی انجام نمی‌دهد (و `alreadyRolledBack` را گزارش می‌کند).

### `wiki obsidian ...`

فرمان‌های کمکی Obsidian برای مخزن‌هایی که در حالت سازگار با Obsidian اجرا می‌شوند: `status`، `search`، `open`، `command`، `daily`. وقتی `obsidian.useOfficialCli` فعال باشد، این فرمان‌ها به CLI رسمی `obsidian` در `PATH` نیاز دارند.

اعتبارسنجی پیکربندی، `obsidian.useOfficialCli: true` را وقتی `vault.scope` برابر با `agent` باشد رد می‌کند، زیرا `obsidian.vaultName` یک تنظیم سراسری است، نه نگاشت به‌ازای هر عامل. رندر Markdown سازگار با Obsidian همچنان در دسترس است.

## راهنمای کاربرد عملی

- وقتی منشأ و هویت صفحه اهمیت دارند، از `wiki search` + `wiki get` استفاده کنید.
- به‌جای ویرایش دستی بخش‌های تولیدشدهٔ مدیریت‌شده، از `wiki apply` استفاده کنید.
- پیش از اعتماد به محتوای متناقض یا کم‌اطمینان، از `wiki lint` استفاده کنید.
- پس از واردسازی انبوه یا تغییر منابع، وقتی داشبوردهای تازه و چکیده‌های کامپایل‌شده را فوراً می‌خواهید، از `wiki compile` استفاده کنید.
- وقتی پایپ‌لاین کاتالوگ داده، خروجی مستندات یا غنی‌سازی عامل از قبل بسته‌های Markdown در قالب OKF تولید می‌کند، از `wiki okf import` استفاده کنید.
- وقتی حالت پل به مصنوعات حافظهٔ تازه صادرشده وابسته است، از `wiki bridge import` استفاده کنید.

## ارتباط با پیکربندی

رفتار `openclaw wiki` تحت تأثیر موارد زیر است:

- `plugins.entries.memory-wiki.config.vaultMode`
- `plugins.entries.memory-wiki.config.vault.scope`
- `plugins.entries.memory-wiki.config.vault.path`
- `plugins.entries.memory-wiki.config.search.backend`
- `plugins.entries.memory-wiki.config.search.corpus`
- `plugins.entries.memory-wiki.config.bridge.*`
- `plugins.entries.memory-wiki.config.obsidian.*`
- `plugins.entries.memory-wiki.config.ingest.autoCompile`
- `plugins.entries.memory-wiki.config.render.*`
- `plugins.entries.memory-wiki.config.context.includeCompiledDigestPrompt`

برای مدل کامل پیکربندی، [Plugin ویکی حافظه](/fa/plugins/memory-wiki) را ببینید.

## مرتبط

- [مرجع CLI](/fa/cli)
- [ویکی حافظه](/fa/plugins/memory-wiki)
