---
read_when:
    - توضیح ClawHub چیست
    - جست‌وجو، نصب یا به‌روزرسانی Skills یا Pluginها
    - انتشار Skills یا Pluginها در رجیستری
    - انتخاب بین جریان‌های CLI در openclaw و clawhub
sidebarTitle: ClawHub
summary: مروری عمومی بر ClawHub برای کشف، نصب، انتشار، امنیت و CLI مربوط به clawhub.
title: ClawHub
x-i18n:
    generated_at: "2026-07-27T13:55:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: fde96ccb410b84dc4d3a48d42bbdbc0a80ac11dfb053afac2ee9e7e9d1605a5b
    source_path: clawhub/index.md
    workflow: 16
---

# ClawHub

ClawHub رجیستری عمومی Skills و Pluginهای OpenClaw است.

- از فرمان‌های بومی `openclaw` برای جست‌وجو، نصب و به‌روزرسانی Skills و نصب Pluginها از ClawHub استفاده کنید.
- برای احراز هویت رجیستری، انتشار و جریان‌های کاری حذف/بازیابی، از CLI جداگانهٔ `clawhub` استفاده کنید.

سایت: [clawhub.ai](https://clawhub.ai)

## شروع سریع

Skills را با OpenClaw جست‌وجو و نصب کنید:

```bash
openclaw skills search "calendar"
openclaw skills install @openclaw/demo
openclaw skills update --all
```

Pluginها را با OpenClaw جست‌وجو و نصب کنید:

```bash
openclaw plugins search "calendar"
openclaw plugins install clawhub:<package>
openclaw plugins update --all
```

هنگامی که به جریان‌های کاری دارای احراز هویت رجیستری، مانند انتشار یا حذف/بازیابی، نیاز دارید، CLI مربوط به ClawHub را نصب کنید:

```bash
npm i -g clawhub
# یا
pnpm add -g clawhub
```

## موارد میزبانی‌شده در ClawHub

| سطح           | محتوای ذخیره‌شده                                               | فرمان متداول                                  |
| -------------- | ------------------------------------------------------------ | -------------------------------------------- |
| Skills         | بسته‌های متنی نسخه‌بندی‌شده دارای `SKILL.md` به‌همراه فایل‌های پشتیبان | `openclaw skills install @openclaw/demo`     |
| Pluginهای کد   | بسته‌های Plugin در OpenClaw با فرادادهٔ سازگاری         | `openclaw plugins install clawhub:<package>` |
| Pluginهای بسته‌ای | بسته‌های Plugin بسته‌بندی‌شده برای توزیع OpenClaw            | `clawhub package publish <source>`           |

ClawHub نسخه‌های semver، برچسب‌هایی مانند `latest`، گزارش‌های تغییرات، فایل‌ها،
دانلودها، ستاره‌ها و خلاصه‌های اسکن امنیتی را ردیابی می‌کند. صفحه‌های عمومی وضعیت فعلی رجیستری
را نمایش می‌دهند تا کاربران بتوانند پیش از نصب، یک Skill یا Plugin را بررسی کنند.

## جریان‌های بومی OpenClaw

فرمان‌های بومی OpenClaw در فضای کاری فعال OpenClaw نصب می‌کنند و
فرادادهٔ منبع را ماندگار می‌سازند تا فرمان‌های به‌روزرسانی بعدی بتوانند همچنان از ClawHub استفاده کنند.

هنگامی که نصب یک Plugin باید از طریق ClawHub رفع شود، از `clawhub:<package>` استفاده کنید.
مشخصات سادهٔ Plugin که برای npm ایمن هستند، ممکن است هنگام انتقال‌های انتشار از طریق npm رفع شوند و
هنگامی که منبع باید صریح باشد، `npm:<package>` فقط مختص npm باقی می‌ماند.

نصب Pluginها پیش از اجرای نصب بایگانی، سازگاری اعلام‌شدهٔ `pluginApi` و `minGatewayVersion`
را اعتبارسنجی می‌کند. هنگامی که یک نسخهٔ بسته، مصنوع ClawPack را منتشر می‌کند، OpenClaw فایل npm-pack دقیقِ بارگذاری‌شدهٔ `.tgz` را ترجیح می‌دهد، سرآیند چکیدهٔ ClawHub و بایت‌های دانلودشده را تأیید می‌کند و فرادادهٔ مصنوع را برای
به‌روزرسانی‌های بعدی ثبت می‌کند.

## CLI مربوط به ClawHub

CLI مربوط به ClawHub برای کارهای دارای احراز هویت رجیستری است:

```bash
clawhub login
clawhub whoami
clawhub search "postgres backups"
clawhub skill publish ./my-skill --slug my-skill --name "My Skill" --version 1.0.0
clawhub package explore --family code-plugin
clawhub package inspect episodic-claw
clawhub package publish your-org/your-plugin --dry-run
clawhub package publish your-org/your-plugin
```

این CLI همچنین برای جریان‌های کاری مستقیم رجیستری، فرمان‌های نصب/به‌روزرسانی Skill را دارد:

```bash
clawhub install @openclaw/demo
clawhub update @openclaw/demo
clawhub update --all
clawhub list
```

این فرمان‌ها Skills را در `./skills` زیر شاخهٔ کاری فعلی نصب
و نسخه‌های نصب‌شده را در `.clawhub/lock.json` ثبت می‌کنند.

## انتشار

Skills را از یک پوشهٔ محلی حاوی `SKILL.md` منتشر کنید:

```bash
clawhub skill publish <path>
```

گزینه‌های متداول انتشار:

- `--slug <slug>`: نام URL مهارت منتشرشده.
- `--name <name>`: نام نمایشی.
- `--version <version>`: نسخهٔ semver.
- `--changelog <text>`: متن گزارش تغییرات.
- `--tags <tags>`: برچسب‌های جداشده با ویرگول، با مقدار پیش‌فرض `latest`.

Pluginها را از یک پوشهٔ محلی، `owner/repo`، `owner/repo@ref` یا یک URL در GitHub
منتشر کنید:

```bash
clawhub package publish <source>
```

برای ساخت طرح دقیق انتشار بدون بارگذاری، از `--dry-run` و برای خروجی مناسب CI
از `--json` استفاده کنید.

Pluginهای کد باید فرادادهٔ الزامی سازگاری OpenClaw را در
`package.json`، از جمله `openclaw.compat.pluginApi` و
`openclaw.build.openclawVersion`، داشته باشند. برای مرجع کامل فرمان‌ها، به [CLI](/fa/clawhub/cli)
و برای فرادادهٔ Skill به [قالب Skill](/clawhub/skill-format) مراجعه کنید.

## امنیت و مدیریت محتوا

ClawHub به‌طور پیش‌فرض باز است: همه می‌توانند بارگذاری کنند، اما انتشار به حساب GitHub
با قدمت کافی برای عبور از دروازهٔ بارگذاری نیاز دارد. صفحه‌های عمومی جزئیات، پیش از نصب یا دانلود،
آخرین وضعیت اسکن را خلاصه می‌کنند.

ClawHub بررسی‌های خودکار را روی Skills و نسخه‌های Plugin منتشرشده اجرا می‌کند. نسخه‌های نگه‌داشته‌شده برای اسکن
یا مسدودشده ممکن است از کاتالوگ عمومی و سطوح نصب ناپدید شوند، درحالی‌که برای مالکشان در `/dashboard`
قابل مشاهده باقی می‌مانند.

کاربران واردشده می‌توانند Skills و بسته‌ها را گزارش کنند. مدیران محتوا می‌توانند گزارش‌ها را بررسی کنند،
محتوا را پنهان یا بازیابی کنند و حساب‌های سوءاستفاده‌گر را مسدود کنند. برای جزئیات سیاست‌ها و اعمال آن‌ها، به
[امنیت](/fa/clawhub/security)،
[ممیزی‌های امنیتی](/clawhub/security-audits)،
[مدیریت محتوا و ایمنی حساب](/clawhub/moderation) و
[استفادهٔ قابل‌قبول](/fa/clawhub/acceptable-usage) مراجعه کنید.

## تله‌متری و محیط

هنگامی که در حالت واردشده `clawhub install` را اجرا می‌کنید، CLI ممکن است یک رویداد نصب
به‌صورت best-effort ارسال کند تا ClawHub بتواند تعداد کل نصب‌ها را محاسبه کند. با دستور زیر آن را غیرفعال کنید:

```bash
export CLAWHUB_DISABLE_TELEMETRY=1
```

بازنویسی‌های محیطی مفید:

| متغیر                         | اثر                                               |
| ----------------------------- | ------------------------------------------------- |
| `CLAWHUB_SITE`                | URL سایت مورد استفاده برای ورود با مرورگر را بازنویسی می‌کند.     |
| `CLAWHUB_REGISTRY`            | URL مربوط به API رجیستری را بازنویسی می‌کند.                    |
| `CLAWHUB_CONFIG_PATH`         | محل ذخیرهٔ وضعیت توکن/پیکربندی توسط CLI را بازنویسی می‌کند. |
| `CLAWHUB_WORKDIR`             | شاخهٔ کاری پیش‌فرض را بازنویسی می‌کند.           |
| `CLAWHUB_DISABLE_TELEMETRY=1` | تله‌متری نصب را غیرفعال می‌کند.                        |

برای مطالب مرجع عمیق‌تر، به [تله‌متری](/clawhub/telemetry)، [HTTP API](/clawhub/http-api) و
[عیب‌یابی](/clawhub/troubleshooting) مراجعه کنید.
