---
read_when:
    - باید خروجی خام مدل را برای نشت استدلال بررسی کنید
    - می‌خواهید هنگام تکرار و اصلاح، Gateway را در حالت پایش اجرا کنید
    - به یک گردش‌کار اشکال‌زدایی تکرارپذیر نیاز دارید
summary: 'ابزارهای اشکال‌زدایی: حالت پایش، جریان‌های خام مدل و ردیابی نشت استدلال'
title: اشکال‌زدایی
x-i18n:
    generated_at: "2026-07-27T15:20:03Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 45a1196c03e4deede3ce47553e1b2b3e1903ee04fe6855d929e0c32bf4e5e686
    source_path: help/debugging.md
    workflow: 16
---

راهکارهای کمکی اشکال‌زدایی برای خروجی جریانی، تکرار Gateway و پروفایل‌سازی راه‌اندازی.

## بازنویسی‌های اشکال‌زدایی زمان اجرا

`/debug` بازنویسی‌های پیکربندی **فقط در زمان اجرا** (در حافظه، نه روی دیسک) را تنظیم می‌کند. به‌طور پیش‌فرض غیرفعال است؛ آن را با `commands.debug: true` فعال کنید.

```text
/debug show
/debug set channels.whatsapp.responsePrefix="[openclaw]"
/debug unset channels.whatsapp.responsePrefix
/debug reset
```

`/debug reset` همه بازنویسی‌ها را پاک می‌کند و به پیکربندی روی دیسک بازمی‌گردد.

## خروجی ردیابی نشست

`/trace` بدون فعال‌کردن حالت کاملاً پرجزئیات، خطوط ردیابی/اشکال‌زدایی متعلق به Plugin را برای یک نشست نمایش می‌دهد. از آن برای عیب‌یابی Plugin، مانند خلاصه‌های اشکال‌زدایی Active Memory، استفاده کنید؛ برای خروجی عادی وضعیت/ابزار از `/verbose` استفاده کنید.

```text
/trace
/trace on
/trace off
```

## ردیابی چرخه حیات Plugin

برای مشاهده تفکیک مرحله‌به‌مرحله فراداده Plugin، کشف، رجیستری، آینه زمان اجرا، تغییر پیکربندی و عملیات نوسازی، `OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1` را تنظیم کنید. خروجی در stderr نوشته می‌شود تا خروجی JSON فرمان همچنان قابل تجزیه بماند.
هنگامی که این ردیابی فعال باشد، شکست‌های بارگذاری Plugin شامل ردیابی پشته آن‌ها نیز می‌شود.

```bash
OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1 openclaw plugins install tokenjuice --force
```

```text
[plugins:lifecycle] phase="config read" ms=6.83 status=ok command="install"
[plugins:lifecycle] phase="slot selection" ms=94.31 status=ok command="install" pluginId="tokenjuice"
[plugins:lifecycle] phase="registry refresh" ms=51.56 status=ok command="install" reason="source-changed"
```

پیش از رجوع به پروفایل‌ساز CPU از این روش استفاده کنید. در یک وارسی منبع، پس از `pnpm build` زمان اجرای ساخته‌شده را با `node dist/entry.js ...` اندازه‌گیری کنید؛ `pnpm openclaw ...` سربار اجراکننده منبع را نیز اندازه‌گیری می‌کند.

برای زمان‌بندی‌های همگام بارگذاری ماژول، به‌جای یک کلید محیطی جداگانه و مختص Plugin، از سطح عیب‌یابی مشترک استفاده کنید:

```bash
OPENCLAW_DIAGNOSTICS=plugin.load-profile openclaw plugins list
```

## پروفایل‌سازی راه‌اندازی CLI و فرمان‌ها

بنچمارک‌های راه‌اندازی ثبت‌شده در مخزن:

```bash
pnpm test:startup:bench:smoke
pnpm tsx scripts/bench-cli-startup.ts --preset real --case status --runs 3
pnpm tsx scripts/bench-cli-startup.ts --preset real --cpu-prof-dir .artifacts/cli-cpu
```

برای پروفایل‌سازی موردی از طریق اجراکننده عادی منبع، `OPENCLAW_RUN_NODE_CPU_PROF_DIR` را تنظیم کنید:

```bash
OPENCLAW_RUN_NODE_CPU_PROF_DIR=.artifacts/cli-cpu pnpm openclaw status
```

اجراکننده منبع پرچم‌های پروفایل CPU مربوط به Node را اضافه می‌کند و یک `.cpuprofile` برای فرمان می‌نویسد. پیش از افزودن ابزارگذاری موقت به کد فرمان، از این روش استفاده کنید.

برای توقف‌های راه‌اندازی که شبیه عملیات همگام فایل‌سیستم یا بارگذار ماژول هستند، پرچم ردیابی ورودی/خروجی همگام Node را از طریق اجراکننده منبع اضافه کنید:

```bash
OPENCLAW_TRACE_SYNC_IO=1 pnpm openclaw gateway --force
```

`pnpm gateway:watch` این پرچم را به‌طور پیش‌فرض برای فرزند Gateway تحت نظارت غیرفعال نگه می‌دارد؛ اگر در حالت نظارت نیز خروجی ردیابی ورودی/خروجی همگام را می‌خواهید، `OPENCLAW_TRACE_SYNC_IO=1` را تنظیم کنید.

## حالت نظارت Gateway

```bash
pnpm gateway:watch
```

این فرمان به‌طور پیش‌فرض یک نشست tmux با نام `openclaw-gateway-watch-<profile>` (برای مثال `openclaw-gateway-watch-main`) را آغاز یا بازراه‌اندازی می‌کند؛ پسوند پورتی مانند `openclaw-gateway-watch-dev-19001` فقط زمانی افزوده می‌شود که `OPENCLAW_GATEWAY_PORT` با پورت پیش‌فرض `18789` متفاوت باشد. از ترمینال‌های تعاملی به‌طور خودکار متصل می‌شود؛ پوسته‌های غیرتعاملی، CI و فراخوانی‌های اجرای عامل جدا می‌مانند و در عوض دستورالعمل اتصال را چاپ می‌کنند:

```bash
tmux attach -t openclaw-gateway-watch-main
# خواندن خروجی اخیر بدون اتصال
tmux capture-pane -ep -t openclaw-gateway-watch-main -S -200
```

پنجره از `remain-on-exit` در tmux استفاده می‌کند، بنابراین شکست‌های راه‌اندازی به‌جای حذف نشست، برای اتصال یا ثبت در دسترس می‌مانند. اجرای دوباره `pnpm gateway:watch` آن پنجره را دوباره ایجاد می‌کند.

پنجره tmux ناظر خام را اجرا می‌کند:

```bash
node scripts/watch-node.mjs gateway --force
```

پیش از نظارت بر پورت پیکربندی‌شده/پیش‌فرض، پوشاننده tmux سرویس Gateway نصب‌شده پروفایل فعال را متوقف می‌کند. این کار پورت را بدون بازایجاد و جایگزینی آن توسط launchd،‏ systemd یا Scheduled Task به ناظر منبع واگذار می‌کند. سرویس نصب‌شده باقی می‌ماند؛ پس از پایان نشست نظارت، آن را با فرمان زیر بازیابی کنید:

```bash
pnpm openclaw gateway start
```

هنگامی که یک `--port` یا `OPENCLAW_GATEWAY_PORT` صریح با پورت مؤثر سرویس نصب‌شده متفاوت باشد، پوشاننده سرویس را در حال اجرا نگه می‌دارد تا هر دو Gateway بتوانند کنار یکدیگر اجرا شوند.

حالت پیش‌زمینه بدون tmux:

```bash
pnpm gateway:watch:raw
# یا
OPENCLAW_GATEWAY_WATCH_TMUX=0 pnpm gateway:watch
```

حالت خام سرویس نصب‌شده را مدیریت نمی‌کند. اگر سرویس از همان پورت استفاده می‌کند، ابتدا `pnpm openclaw gateway stop` را اجرا کنید.

مدیریت tmux را حفظ کنید اما اتصال خودکار را غیرفعال کنید:

```bash
OPENCLAW_GATEWAY_WATCH_ATTACH=0 pnpm gateway:watch
```

هنگام اشکال‌زدایی نقاط داغ راه‌اندازی/زمان اجرا، زمان CPU مربوط به Gateway تحت نظارت را پروفایل‌سازی کنید:

```bash
pnpm gateway:watch --benchmark
```

پوشاننده نظارت پیش از فراخوانی Gateway،‏ `--benchmark` را مصرف می‌کند و با هر خروج فرزند Gateway، یک `.cpuprofile` مربوط به V8 را در `.artifacts/gateway-watch-profiles/` می‌نویسد. برای تخلیه پروفایل جاری، Gateway تحت نظارت را متوقف یا بازراه‌اندازی کنید؛ سپس آن را با Chrome DevTools یا Speedscope باز کنید:

```bash
npx speedscope .artifacts/gateway-watch-profiles/*.cpuprofile
```

- `--benchmark-dir <path>`: پروفایل‌ها را در محل دیگری بنویسید.
- `--benchmark-no-force`: پاک‌سازی پورت پیش‌فرض `--force` را رد کنید و اگر پورت Gateway از قبل در حال استفاده است، فوراً شکست بخورید.

حالت بنچمارک به‌طور پیش‌فرض پیام‌های پرتکرار ردیابی ورودی/خروجی همگام را سرکوب می‌کند. برای دریافت هم‌زمان پروفایل‌های CPU و ردیابی پشته ورودی/خروجی همگام، `OPENCLAW_TRACE_SYNC_IO=1` را همراه با `--benchmark` تنظیم کنید؛ در حالت بنچمارک، این بلوک‌های ردیابی در `gateway-watch-output.log` زیر پوشه بنچمارک قرار می‌گیرند (و از پنجره ترمینال فیلتر می‌شوند)، درحالی‌که گزارش‌های عادی Gateway همچنان قابل مشاهده‌اند.

پوشاننده tmux انتخابگرهای رایج و غیرمحرمانه زمان اجرا، از جمله `OPENCLAW_PROFILE`،‏ `OPENCLAW_CONFIG_PATH`،‏ `OPENCLAW_STATE_DIR`،‏ `OPENCLAW_GATEWAY_PORT` و `OPENCLAW_SKIP_CHANNELS` را به پنجره منتقل می‌کند. اطلاعات اعتبارسنجی ارائه‌دهنده را در پروفایل/پیکربندی عادی خود قرار دهید، یا برای اسرار موقتی و موردی از حالت پیش‌زمینه خام استفاده کنید.

اگر Gateway تحت نظارت هنگام راه‌اندازی خارج شود، ناظر یک‌بار `openclaw doctor --fix --non-interactive` را اجرا می‌کند و فرزند Gateway را بازراه‌اندازی می‌کند. برای مشاهده شکست اصلی راه‌اندازی بدون مرحله ترمیم مختص توسعه، `OPENCLAW_GATEWAY_WATCH_AUTO_DOCTOR=0` را تنظیم کنید.

پنجره مدیریت‌شده tmux به‌طور پیش‌فرض گزارش‌های رنگی Gateway را نمایش می‌دهد؛ هنگام راه‌اندازی `pnpm gateway:watch`،‏ `FORCE_COLOR=0` را تنظیم کنید تا خروجی ANSI غیرفعال شود.

ناظر با تغییر فایل‌های مرتبط با ساخت در `src/`، فایل‌های منبع افزونه، فراداده `package.json` و `openclaw.plugin.json` افزونه،‏ `tsconfig.json`،‏ `package.json` و `tsdown.config.ts` بازراه‌اندازی می‌شود. تغییرات فراداده افزونه Gateway را بدون اجبار به ساخت مجدد بازراه‌اندازی می‌کنند؛ تغییرات منبع و پیکربندی همچنان ابتدا `dist` را دوباره می‌سازند.

پرچم‌های CLI مربوط به Gateway را پس از `gateway:watch` اضافه کنید تا در هر بازراه‌اندازی منتقل شوند. اجرای دوباره همان فرمان نظارت، پنجره نام‌گذاری‌شده tmux را دوباره ایجاد می‌کند؛ ناظر خام یک قفل تک‌ناظری نگه می‌دارد تا والدهای ناظر تکراری به‌جای انباشته‌شدن جایگزین شوند.

## پروفایل توسعه + Gateway توسعه (--dev)

دو پرچم `--dev` **جداگانه**:

- **`--dev` سراسری (پروفایل):** وضعیت را در `~/.openclaw-dev` ایزوله می‌کند و پورت پیش‌فرض Gateway را روی `19001` قرار می‌دهد (پورت‌های مشتق‌شده نیز همراه آن جابه‌جا می‌شوند).
- **`gateway --dev`:** به Gateway می‌گوید در صورت نبودن پیکربندی و فضای کاری، آن‌ها را به‌طور خودکار با مقادیر پیش‌فرض ایجاد کند (و راه‌اندازی اولیه را رد کند).

روند پیشنهادی (پروفایل توسعه + راه‌اندازی اولیه توسعه):

```bash
pnpm gateway:dev
OPENCLAW_PROFILE=dev openclaw tui
```

بدون نصب سراسری، CLI را از طریق `pnpm openclaw ...` اجرا کنید.

کارکرد این فرمان:

1. **ایزوله‌سازی پروفایل** (`--dev` سراسری)
   - `OPENCLAW_PROFILE=dev`
   - `OPENCLAW_STATE_DIR=~/.openclaw-dev`
   - `OPENCLAW_CONFIG_PATH=~/.openclaw-dev/openclaw.json`
   - `OPENCLAW_GATEWAY_PORT=19001` (پورت‌های مرورگر/canvas نیز متناسب با آن جابه‌جا می‌شوند)

2. **راه‌اندازی اولیه توسعه** (`gateway --dev`)
   - اگر پیکربندی وجود نداشته باشد، یک پیکربندی حداقلی می‌نویسد (`gateway.mode=local`، اتصال به loopback).
   - `agents.defaults.workspace` را روی فضای کاری توسعه و `agents.defaults.skipBootstrap=true` تنظیم می‌کند.
   - اگر فایل‌های فضای کاری وجود نداشته باشند، آن‌ها را ایجاد می‌کند: `AGENTS.md`،‏ `SOUL.md`،‏ `TOOLS.md`،‏ `IDENTITY.md`،‏ `USER.md`.
   - هویت پیش‌فرض: **C3-PO** (دروید پروتکل).
   - `pnpm gateway:dev` همچنین برای ردکردن ارائه‌دهندگان کانال، `OPENCLAW_SKIP_CHANNELS=1` را تنظیم می‌کند.

Gatewayهای توسعه به‌طور پیش‌فرض محرک‌های محیطی کانال را نادیده می‌گیرند، بنابراین اطلاعات اعتبارسنجی به‌ارث‌رسیده از پوسته، نمونه توسعه را به سرویس‌های واقعی کانال متصل نمی‌کنند. پیکربندی صریح `channels.<id>` همچنان کار می‌کند. برای بازیابی پیکربندی خودکار کانال از محیط در همان اجرا، `--dev-ambient-channels` را همراه با `--dev` ارسال کنید.

روند بازنشانی (شروع تازه):

```bash
pnpm gateway:dev:reset
```

<Note>
`--dev` یک پرچم پروفایل **سراسری** است و برخی اجراکننده‌ها آن را مصرف می‌کنند. اگر لازم است آن را به‌صراحت مشخص کنید، از شکل متغیر محیطی استفاده کنید:

```bash
OPENCLAW_PROFILE=dev openclaw gateway --dev --reset
```

</Note>

`--reset` پیکربندی، اطلاعات اعتبارسنجی، نشست‌ها و فضای کاری توسعه را پاک می‌کند (به زباله‌دان منتقل می‌شوند، نه اینکه حذف شوند) و سپس تنظیمات پیش‌فرض توسعه را دوباره ایجاد می‌کند.

<Tip>
اگر یک Gateway غیرتوسعه از قبل در حال اجرا است (launchd یا systemd)، ابتدا آن را متوقف کنید:

```bash
openclaw gateway stop
```

</Tip>

## ثبت جریان خام

OpenClaw می‌تواند **جریان خام دستیار** را پیش از هرگونه فیلترکردن/قالب‌بندی ثبت کند. این بهترین روش برای مشاهده این است که آیا استدلال به‌شکل دلتاهای متن ساده می‌رسد (یا به‌شکل بلوک‌های تفکر جداگانه).

آن را از طریق CLI فعال کنید:

```bash
pnpm gateway:watch --raw-stream
```

بازنویسی اختیاری مسیر:

```bash
pnpm gateway:watch --raw-stream --raw-stream-path ~/.openclaw/logs/raw-stream.jsonl
```

متغیرهای محیطی معادل:

```bash
OPENCLAW_RAW_STREAM=1
OPENCLAW_RAW_STREAM_PATH=~/.openclaw/logs/raw-stream.jsonl
```

فایل پیش‌فرض: `~/.openclaw/logs/raw-stream.jsonl`

## نکات ایمنی

- گزارش‌های جریان خام ممکن است شامل اعلان‌های کامل، خروجی ابزار و داده‌های کاربر باشند.
- گزارش‌ها را محلی نگه دارید و پس از اشکال‌زدایی حذف کنید.
- اگر گزارش‌ها را به اشتراک می‌گذارید، ابتدا اسرار و اطلاعات هویتی شخصی را از آن‌ها پاک کنید.

## اشکال‌زدایی در VSCode

نقشه‌های منبع ضروری‌اند، زیرا فرایند ساخت نام فایل‌های تولیدشده را هش می‌کند. `launch.json` موجود، سرویس Gateway را هدف قرار می‌دهد:

1. **بازسازی و اشکال‌زدایی Gateway** - پیش از راه‌اندازی Gateway،‏ `/dist` را حذف می‌کند و با فعال‌بودن اشکال‌زدایی دوباره می‌سازد.
2. **اشکال‌زدایی Gateway** - بدون تغییر `/dist`، یک ساخت موجود را اشکال‌زدایی می‌کند.

### راه‌اندازی

1. **Run and Debug** را باز کنید (Activity Bar یا `Ctrl`+`Shift`+`D`).
2. **Rebuild and Debug Gateway** را انتخاب کنید و **Start Debugging** را فشار دهید.

برای مدیریت دستی چرخه ساخت/اشکال‌زدایی:

1. نقشه‌های منبع را در یک ترمینال فعال کنید:
   - **Linux/macOS**: `export OUTPUT_SOURCE_MAPS=1`
   - **Windows (PowerShell)**: `$env:OUTPUT_SOURCE_MAPS="1"`
   - **Windows (CMD)**: `set OUTPUT_SOURCE_MAPS=1`
2. بازسازی: `pnpm clean:dist && pnpm build`
3. **Debug Gateway** را انتخاب کنید و **Start Debugging** را فشار دهید.

نقاط توقف را در فایل‌های TypeScript مربوط به `src/` تنظیم کنید؛ اشکال‌زدا با استفاده از نقشه‌های منبع، آن‌ها را به JavaScript کامپایل‌شده نگاشت می‌کند.

### نکات

- **Rebuild and Debug Gateway**،‏ `/dist` را حذف می‌کند و در هر اجرا، یک `pnpm build` کامل را با نقشه‌های منبع اجرا می‌کند.
- **Debug Gateway** می‌تواند بدون تأثیر بر `/dist` آغاز/متوقف شود، اما باید چرخه ساخت را در یک ترمینال جداگانه مدیریت کنید.
- برای اشکال‌زدایی زیرفرمان‌های دیگر CLI،‏ `args` مربوط به `launch.json` را ویرایش کنید.
- برای استفاده از CLI ساخته‌شده در کارهای دیگر (برای مثال `dashboard --no-open`، اگر نشست اشکال‌زدایی شما یک توکن احراز هویت جدید ایجاد می‌کند)، آن را از ترمینالی دیگر اجرا کنید: `node ./openclaw.mjs` یا نام مستعاری مانند `alias openclaw-build="node $(pwd)/openclaw.mjs"`.

## مرتبط

- [رفع اشکال](/fa/help/troubleshooting)
- [پرسش‌های متداول](/fa/help/faq)
