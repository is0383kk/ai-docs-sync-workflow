---
read_when:
    - اجرای QA دسکتاپ Slack برای Mantis از GitHub یا به‌صورت محلی
    - اشکال‌زدایی اجرای کند Mantis در نسخه دسکتاپ Slack
    - انتخاب حالت منبع، ازپیش‌آماده‌شده یا اجارهٔ گرم
    - ارسال شواهد تصویری و ویدئویی در یک PR
summary: 'راهنمای عملیاتی برای QA دسکتاپ Slack در Mantis: اجرای GitHub، CLI محلی، اجاره‌های گرم VNC، حالت‌های hydrate، تفسیر زمان‌بندی، مصنوعات و مدیریت خطا.'
title: راهنمای عملیاتی دسکتاپ Slack برای Mantis
x-i18n:
    generated_at: "2026-07-27T16:23:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b3e956d99fc43a7b6fe65e2e820812b0e0e8b9e32badd25be27c74d302ab30dc
    source_path: concepts/mantis-slack-desktop-runbook.md
    workflow: 16
---

Mantis Slack desktop QA مسیر رابط کاربری واقعی برای باگ‌های هم‌رده Slack است که به
دسکتاپ Linux، بازیابی از طریق VNC، Slack Web، یک Gateway واقعی OpenClaw، اسکرین‌شات‌ها،
ویدئوها و یک نظر حاوی شواهد در PR نیاز دارند. زمانی از آن استفاده کنید که تست‌های واحد یا مسیر زنده
بدون رابط گرافیکی Slack نتوانند باگ را اثبات کنند.

## مدل ذخیره‌سازی

Mantis از سه لایه ذخیره‌سازی استفاده می‌کند:

- **ایمیج ارائه‌دهنده** - متعلق به Crabbox است و در حساب ارائه‌دهنده ابری ذخیره می‌شود.
  قابلیت‌های ماشین (Chrome/Chromium، ffmpeg، scrot،
  Node/corepack/pnpm و ابزارهای بومی ساخت) و دایرکتوری‌های خالی کش را نگه می‌دارد.
- **وضعیت اجاره گرم** - متعلق به نشست اپراتور فعلی است. تا زمانی که اجاره فعال است، می‌تواند
  یک پروفایل مرورگر واردشده، `/var/cache/crabbox/pnpm` و یک checkout آماده از کد منبع
  را نگه دارد.
- **آرتیفکت‌های Mantis** - متعلق به اجرای OpenClaw هستند. در
  `.artifacts/qa-e2e/mantis/...` قرار می‌گیرند؛ GitHub Actions آن‌ها را بارگذاری می‌کند و GitHub App مربوط به Mantis
  شواهد درون‌خطی را در PR نظر می‌دهد.

هرگز اسرار، کوکی‌های مرورگر، وضعیت ورود Slack، checkoutهای مخزن،
`node_modules` یا `dist/` را در ایمیج ارائه‌دهنده تعبیه نکنید.

## اجرای GitHub

گردش‌کار را از `main` اجرا کنید:

```bash
gh workflow run mantis-slack-desktop-smoke.yml \
  --ref main \
  -f candidate_ref=<trusted-ref-or-sha> \
  -f pr_number=<pr-number> \
  -f scenario_id=slack-canary \
  -f crabbox_provider=aws \
  -f keep_vm=false \
  -f hydrate_mode=source
```

`candidate_ref` محدود شده است، زیرا گردش‌کار از اطلاعات هویتی واقعی استفاده می‌کند: این مقدار
باید به ancestry فعلی `main`، یک تگ انتشار یا head یک PR باز در
`openclaw/openclaw` resolve شود.

گردش‌کار موارد زیر را تولید می‌کند:

- آرتیفکت بارگذاری‌شده `mantis-slack-desktop-smoke-<run-id>-<attempt>`
- نظر درون‌خطی PR از GitHub App مربوط به Mantis
- `slack-desktop-smoke.png`، `slack-desktop-smoke.mp4`
- `slack-desktop-smoke-preview.gif`، `slack-desktop-smoke-change.mp4`
- `mantis-slack-desktop-smoke-summary.json`، `mantis-slack-desktop-smoke-report.md`
- لاگ‌های راه دور: `slack-desktop-command.log`، `openclaw-gateway.log`، `chrome.log`، `ffmpeg.log`

نظر PR با استفاده از نشانگر پنهان `<!-- mantis-slack-desktop-smoke -->` در همان محل به‌روزرسانی می‌شود.

## CLI محلی

اثبات سرد از کد منبع:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --credential-source convex \
  --credential-role maintainer \
  --provider-mode live-frontier \
  --model openai/gpt-5.4 \
  --alt-model openai/gpt-5.4 \
  --scenario slack-canary \
  --hydrate-mode source
```

ماشین مجازی را برای بازیابی از طریق VNC نگه دارید:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --gateway-setup \
  --scenario slack-canary \
  --keep-lease
```

VNC را باز کنید:

```bash
crabbox vnc --provider aws --id <cbx_id> --open
```

از یک اجاره گرم دوباره استفاده کنید:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --lease-id <cbx_id-or-slug> \
  --gateway-setup \
  --scenario slack-canary \
  --hydrate-mode source
```

فقط زمانی از `--hydrate-mode prehydrated` استفاده کنید که فضای کاری راه دورِ مورداستفاده مجدد از قبل
دارای `node_modules` و یک `dist/` ساخته‌شده باشد؛ در غیر این صورت، Mantis به‌صورت fail-closed متوقف می‌شود.

رابط کاربری بومی تأیید Slack را اثبات کنید:

```bash
pnpm openclaw qa mantis slack-desktop-smoke \
  --provider aws \
  --class standard \
  --approval-checkpoints \
  --credential-source convex \
  --credential-role maintainer \
  --hydrate-mode source
```

`--approval-checkpoints` با `--gateway-setup` ناسازگار است و نمی‌توان هر دو را هم‌زمان استفاده کرد. این گزینه
سناریوهای opt-in یعنی `slack-approval-exec-native` و `slack-approval-plugin-native`
را اجرا می‌کند، مگر اینکه یک `--scenario` صریح برای نقطه بررسی تأیید ارسال کنید؛ سایر
سناریوهای Slack پیش از شروع ماشین مجازی رد می‌شوند. اجراکننده QA مربوط به Slack
هر فایل JSON نقطه بررسی را از پیام واقعی Slack API که مشاهده کرده است می‌نویسد، سپس
ناظر راه دور آن پیام را در
`approval-checkpoints/<scenario>-pending.png` و
`approval-checkpoints/<scenario>-resolved.png` رندر می‌کند. اگر هر
فایل JSON نقطه بررسی، شواهد پیام، JSON تأیید دریافت یا اسکرین‌شات رندرشده وجود نداشته باشد
یا خالی باشد، اجرا ناموفق خواهد بود.

اجاره‌های سرد GitHub Actions کوکی‌های Slack Web را ندارند، بنابراین ثبت مرورگر آن‌ها
ممکن است به صفحه ورود Slack برسد. برای اثبات نقطه بررسی تأیید، به
تصاویر رندرشده نقطه بررسی و آرتیفکت‌های QA مربوط به Slack اعتماد کنید، نه
`slack-desktop-smoke.png`. تنها زمانی از یک اجاره گرم نگه‌داشته‌شده با پروفایل Slack Web که
به‌صورت دستی وارد شده است استفاده کنید که خود اسکرین‌شات مرورگر باید
Slack Web را نشان دهد.

## حالت‌های آماده‌سازی

| حالت          | زمان استفاده                                  | رفتار راه دور                                                                       | موازنه                                                 |
| ------------- | ----------------------------------------- | ------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `source`      | اثبات عادی PR، ماشین‌های سرد، CI        | `pnpm install --frozen-lockfile --prefer-offline` و `pnpm build` را داخل ماشین مجازی اجرا می‌کند | کندترین حالت، قوی‌ترین اثبات checkout کد منبع                 |
| `prehydrated` | عمداً یک اجاره مورداستفاده مجدد را آماده کرده‌اید | به `node_modules` و `dist/` موجود نیاز دارد؛ نصب/ساخت را رد می‌کند                     | سریع است، اما فقط برای اجاره‌های گرم تحت کنترل اپراتور معتبر است |

GitHub Actions همیشه checkout کاندید را پیش از اجرای ماشین مجازی آماده می‌کند. مخزن
pnpm آن بر اساس سیستم‌عامل، نسخه Node و lockfile کش می‌شود. اجرای `source` در ماشین مجازی
نیز در صورت وجود، از `/var/cache/crabbox/pnpm` دوباره استفاده می‌کند.

## تفسیر زمان‌بندی

`mantis-slack-desktop-smoke-report.md` شامل زمان‌بندی فازها است:

- `crabbox.warmup` - راه‌اندازی ارائه‌دهنده ابری، آماده‌شدن دسکتاپ/مرورگر و SSH.
- `crabbox.inspect` - جست‌وجوی فراداده اجاره.
- `credentials.prepare` - دریافت اجاره اطلاعات هویتی Convex.
- `crabbox.remote_run` - همگام‌سازی، اجرای مرورگر، نصب/ساخت OpenClaw یا
  اعتبارسنجی آماده‌سازی، راه‌اندازی Gateway، ثبت اسکرین‌شات و ضبط ویدئو.
- `artifacts.copy` - بازگرداندن داده‌ها از ماشین مجازی با rsync.

هنگامی که Crabbox یک وضعیت راه دور غیرصفر برمی‌گرداند، `crabbox.remote_run` ممکن است
`accepted` را نشان دهد، اما Mantis فراداده‌ای را کپی کرده باشد که ثابت می‌کند یا راه‌اندازی
Gateway مربوط به OpenClaw کامل شده یا خود فرمان QA مربوط به Slack با موفقیت خاتمه یافته است.
`accepted` را موفق با توضیح در نظر بگیرید، نه یک سناریوی ناموفق.

اگر اجرا کند است:

- مرحله آماده‌سازی غالب است: یک ایمیج بهتر برای ارائه‌دهنده Crabbox از پیش بسازید یا ارتقا دهید.
- `remote_run` در `source` غالب است: از یک اجاره گرم استفاده کنید، استفاده مجدد از مخزن
  pnpm را بهبود دهید یا پیش‌نیازهای ماشین را به ایمیج ارائه‌دهنده منتقل کنید.
- `remote_run` در `prehydrated` غالب است: فضای کاری راه دور در واقع
  آماده نبوده است، یا راه‌اندازی Gateway/مرورگر/Slack کند است.
- کپی آرتیفکت غالب است: اندازه ویدئو و محتوای دایرکتوری آرتیفکت را بررسی کنید.

## چک‌لیست شواهد

یک نظر خوب در PR موارد زیر را نشان می‌دهد:

- شناسه سناریو و SHA کاندید
- نشانی اجرای GitHub Actions و نشانی آرتیفکت
- اسکرین‌شات درون‌خطی نقطه بررسی تأیید، یا اسکرین‌شات Slack Web از یک
  اجاره گرم واردشده
- پیش‌نمایش متحرک درون‌خطی در صورت وجود
- پیوندهای MP4 کامل و MP4 برش‌خورده
- وضعیت موفق/ناموفق و خلاصه زمان‌بندی گزارش

اسکرین‌شات‌ها یا ویدئوها را در مخزن commit نکنید. آن‌ها را در آرتیفکت‌های GitHub
Actions یا نظر PR نگه دارید.

## مدیریت خطا

اگر گردش‌کار پیش از اجرای ماشین مجازی ناموفق شد، ابتدا job مربوط به Actions را بررسی کنید.
علت‌های معمول: `candidate_ref` غیرقابل‌اعتماد، نبود اسرار محیط یا
ناموفق‌بودن نصب/ساخت کاندید.

اگر اجرای ماشین مجازی ناموفق شد اما اسکرین‌شات‌ها بازگردانده شدند، موارد زیر را بررسی کنید:

```bash
cat mantis-slack-desktop-smoke-report.md
cat mantis-slack-desktop-smoke-summary.json
cat slack-desktop-command.log
cat openclaw-gateway.log
cat chrome.log
cat ffmpeg.log
```

اگر اجرا اجاره را نگه داشت، VNC را با فرمان `crabbox vnc ...`
موجود در گزارش باز کنید و پس از پایان کار، اجاره را متوقف کنید:

```bash
crabbox stop --provider aws <cbx_id-or-slug>
```

اگر ورود Slack منقضی شده است، آن را از طریق VNC روی یک اجاره نگه‌داشته‌شده اصلاح کنید و با
`--lease-id` دوباره اجرا کنید. آن پروفایل مرورگر را در ایمیج ارائه‌دهنده تعبیه نکنید.

## مرتبط

- [نمای کلی QA](/fa/concepts/qa-e2e-automation)
- [کانال Slack](/fa/channels/slack)
- [تست](/fa/help/testing)
