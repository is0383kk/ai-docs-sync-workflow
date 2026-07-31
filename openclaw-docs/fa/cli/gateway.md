---
read_when:
    - اجرای Gateway از طریق CLI (محیط توسعه یا سرورها)
    - اشکال‌زدایی احراز هویت Gateway، حالت‌های اتصال و ارتباط‌پذیری
    - کشف Gatewayها از طریق Bonjour (محلی + DNS-SD گسترده)
    - یکپارچه‌سازی ناظر خارجی فرایند Gateway
sidebarTitle: Gateway
summary: CLI ‏Gateway ‏OpenClaw ‏(`openclaw gateway`) — اجرا، پرس‌وجو و کشف Gatewayها
title: Gateway
x-i18n:
    generated_at: "2026-07-27T15:01:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0188d7c79571ebf8f350295775625533a83cb2eb909bcc8763e8ce81806d2214
    source_path: cli/gateway.md
    workflow: 16
---

Gateway سرور WebSocket متعلق به OpenClaw است (کانال‌ها، Nodeها، نشست‌ها، هوک‌ها). همهٔ زیرفرمان‌های زیر ذیل `openclaw gateway ...` قرار دارند.

<CardGroup cols={3}>
  <Card title="کشف Bonjour" href="/fa/gateway/bonjour">
    راه‌اندازی mDNS محلی + DNS-SD گسترده.
  </Card>
  <Card title="نمای کلی کشف" href="/fa/gateway/discovery">
    نحوهٔ اعلام حضور و یافتن Gatewayها توسط OpenClaw.
  </Card>
  <Card title="پیکربندی" href="/fa/gateway/configuration">
    کلیدهای سطح‌بالای پیکربندی Gateway.
  </Card>
</CardGroup>

## اجرای Gateway

```bash
openclaw gateway
openclaw gateway run   # معادل، شکل صریح
```

<AccordionGroup>
  <Accordion title="رفتار هنگام راه‌اندازی">
    - تا زمانی که `gateway.mode=local` در `~/.openclaw/openclaw.json` تنظیم نشده باشد، از شروع خودداری می‌کند. برای اجراهای موقت/توسعه از `--allow-unconfigured` استفاده کنید؛ این گزینه بدون نوشتن یا ترمیم پیکربندی، کنترل محافظتی را دور می‌زند.
    - وقتی هنگام راه‌اندازی یک پیکربندی نامعتبرِ قابل‌ترمیم پیدا شود، ترمینال تعاملی پیشنهاد اجرای `openclaw doctor --fix` را می‌دهد و پس از تأیید، راه‌اندازی را یک‌بار دیگر امتحان می‌کند. اجراهای غیرتعاملی هرگز به‌طور خودکار ترمیم نمی‌کنند؛ در عوض فرمان را نمایش می‌دهند. اگر پیکربندی ترمیم‌شده همچنان نامعتبر باشد، راه‌اندازی متوقف می‌ماند.
    - `openclaw onboard --mode local` و `openclaw setup` مقدار `gateway.mode=local` را می‌نویسند. اگر فایل پیکربندی وجود داشته باشد اما `gateway.mode` موجود نباشد، این وضعیت به‌عنوان پیکربندی آسیب‌دیده/بازنویسی‌شده تلقی می‌شود و Gateway از حدس‌زدن `local` برای شما خودداری می‌کند — فرایند آغازین را دوباره اجرا کنید، کلید را دستی تنظیم کنید، یا `--allow-unconfigured` را ارسال کنید.
    - اتصال به فراتر از loopback بدون احراز هویت مسدود است.
    - مقادیر `--bind` شامل `lan`، `tailnet` و `custom` در حال حاضر تنها از مسیرهای IPv4 تفکیک می‌شوند؛ راه‌اندازی‌های میزبان شخصیِ فقط IPv6 به یک sidecar یا پراکسی IPv4 در جلوی Gateway نیاز دارند.
    - `SIGUSR1` در صورت مجازبودن، راه‌اندازی مجدد درون‌پردازه‌ای را فعال می‌کند. `commands.restart` (پیش‌فرض: فعال) اجرای `SIGUSR1`های ارسال‌شده از بیرون را کنترل می‌کند؛ برای مسدودکردن راه‌اندازی‌های مجدد دستی با سیگنال سیستم‌عامل، آن را روی `false` تنظیم کنید. ابزار روبه‌عاملِ `gateway` فقط‌خواندنی است؛ عامل‌ها از طریق ابزار واگذاری `openclaw` که نیازمند تأیید انسان است، درخواست راه‌اندازی مجدد می‌دهند.
    - `SIGINT`/`SIGTERM` پردازه را متوقف می‌کنند، اما وضعیت سفارشی ترمینال را بازیابی نمی‌کنند — اگر CLI را درون TUI یا ورودی حالت خام قرار داده‌اید، پیش از خروج خودتان ترمینال را بازیابی کنید.

  </Accordion>
</AccordionGroup>

### گزینه‌ها

<ParamField path="--port <port>" type="number">
  پورت WebSocket (پیش‌فرض از پیکربندی/محیط؛ معمولاً `18789`).
</ParamField>
<ParamField path="--bind <mode>" type="string">
  حالت اتصال: `loopback` (پیش‌فرض)، `lan`، `tailnet`، `auto`، `custom`.
</ParamField>
<ParamField path="--token <token>" type="string">
  توکن مشترک برای `connect.params.auth.token`. در صورت تنظیم، مقدار پیش‌فرض آن `OPENCLAW_GATEWAY_TOKEN` است.
</ParamField>
<ParamField path="--auth <mode>" type="string">
  حالت احراز هویت: `none`، `token`، `password`، `trusted-proxy`.
</ParamField>
<ParamField path="--password <password>" type="string">
  گذرواژه برای `--auth password`.
</ParamField>
<ParamField path="--password-file <path>" type="string">
  خواندن گذرواژهٔ Gateway از یک فایل.
</ParamField>
<ParamField path="--tailscale <mode>" type="string">
  نحوهٔ در دسترس قرارگرفتن از طریق Tailscale: `off`، `serve`، `funnel`.
</ParamField>
<ParamField path="--tailscale-reset-on-exit" type="boolean">
  بازنشانی پیکربندی serve/funnel مربوط به Tailscale هنگام خاموش‌شدن.
</ParamField>
<ParamField path="--allow-unconfigured" type="boolean">
  شروع بدون اعمال الزام `gateway.mode=local`. فقط برای راه‌اندازی موقت/توسعه؛ پیکربندی را ماندگار یا ترمیم نمی‌کند.
</ParamField>
<ParamField path="--dev" type="boolean">
  در صورت نبود، پیکربندی توسعه + فضای کاری ایجاد می‌کند (`BOOTSTRAP.md` را نادیده می‌گیرد).
</ParamField>
<ParamField path="--dev-ambient-channels" type="boolean">
  به Gateway توسعه اجازه می‌دهد کانال‌ها را به‌طور خودکار از متغیرهای محیطی موجود پیکربندی کند. به `--dev` نیاز دارد.
</ParamField>
<ParamField path="--reset" type="boolean">
  پیکربندی توسعه، اطلاعات اعتبارسنجی، نشست‌ها و فضای کاری را بازنشانی می‌کند. به `--dev` نیاز دارد.
</ParamField>
<ParamField path="--force" type="boolean">
  پیش از شروع، هر شنوندهٔ موجود روی پورت مقصد را خاتمه می‌دهد. در پوستهٔ غیرتعاملی، این گزینه از خاتمه‌دادن شنوندهٔ تأییدشدهٔ Gateway خودداری می‌کند؛ به‌جای آن از `--dev` یا یک `--profile` ایزوله با پورتی آزاد استفاده کنید.
</ParamField>
<ParamField path="--verbose" type="boolean">
  ثبت گزارش مفصل در stdout/stderr.
</ParamField>
<ParamField path="--cli-backend-logs" type="boolean">
  فقط گزارش‌های بک‌اند CLI را در کنسول نمایش می‌دهد (stdout/stderr را نیز فعال می‌کند).
</ParamField>
<ParamField path="--ws-log <style>" type="string" default="auto">
  سبک گزارش WebSocket: `auto`، `full`، `compact`.
</ParamField>
<ParamField path="--compact" type="boolean">
  نام مستعار `--ws-log compact`.
</ParamField>
<ParamField path="--raw-stream" type="boolean">
  رویدادهای خام جریان مدل را در JSONL ثبت می‌کند.
</ParamField>
<ParamField path="--raw-stream-path <path>" type="string">
  مسیر JSONL جریان خام.
</ParamField>

`--claude-cli-logs` نام مستعار منسوخ‌شدهٔ `--cli-backend-logs` است.

برای `--bind custom`، مقدار `gateway.customBindHost` را روی یک نشانی IPv4 تنظیم کنید. هر نشانی به‌جز `127.0.0.1` یا `0.0.0.0` برای کلاینت‌های همان میزبان، به `127.0.0.1` روی همان پورت نیز نیاز دارد؛ اگر هرکدام از شنونده‌ها نتواند متصل شود، راه‌اندازی شکست می‌خورد. مقدار wildcard یعنی `0.0.0.0` یک نام مستعار الزامی جداگانه اضافه نمی‌کند. راه‌اندازی‌های میزبان شخصیِ فقط IPv6 به یک sidecar یا پراکسی IPv4 در جلوی Gateway نیاز دارند.

## راه‌اندازی مجدد Gateway

```bash
openclaw gateway restart
openclaw gateway restart --safe
openclaw gateway restart --safe --skip-deferral
openclaw gateway restart --force
openclaw gateway restart --wait 30s
```

`--safe` از Gateway در حال اجرا می‌خواهد کارهای فعال را پیش‌بررسی کند و پس از تخلیهٔ آن کارها، یک راه‌اندازی مجدد تجمیع‌شده را زمان‌بندی کند. انتظار به 5 دقیقه محدود است؛ با پایان بودجهٔ زمانی، راه‌اندازی مجدد به‌اجبار انجام می‌شود. `--safe` را نمی‌توان با `--force` یا `--wait` ترکیب کرد.

`--skip-deferral` در یک راه‌اندازی مجدد امن، مانع تعویق به‌دلیل کار فعال را دور می‌زند؛ بنابراین Gateway حتی با وجود مسدودکننده‌های گزارش‌شده، بلافاصله راه‌اندازی مجدد می‌شود. این گزینه به `--safe` نیاز دارد — وقتی تعویق روی وظیفه‌ای مهارنشده گیر کرده است از آن استفاده کنید.

`--wait <duration>` بودجهٔ تخلیه را برای یک راه‌اندازی مجدد ساده (غیرامن) بازنویسی می‌کند. میلی‌ثانیهٔ بدون پسوند یا پسوندهای واحد `ms`، `s`، `m`، `h`، `d` را می‌پذیرد (برای نمونه `30s`، `5m`، `1h30m`)؛ `--wait 0` به‌طور نامحدود منتظر می‌ماند. با `--force` یا `--safe` سازگار نیست.

`--force` تخلیهٔ کار فعال را نادیده می‌گیرد و بلافاصله راه‌اندازی مجدد می‌کند. `restart` ساده (بدون پرچم) رفتار فعلی راه‌اندازی مجدد مدیر سرویس را حفظ می‌کند.

<Warning>
مقدار درون‌خطی `--password` ممکن است در فهرست پردازه‌های محلی نمایان شود. `--password-file`، محیط، یا `gateway.auth.password` مبتنی بر SecretRef را ترجیح دهید.
</Warning>

### ناظران خارجی

تنها زمانی `OPENCLAW_SUPERVISOR_MODE=external` را تنظیم کنید که مدیر پردازهٔ دیگری مالک چرخهٔ عمر Gateway باشد. در این حالت:

- `openclaw gateway restart` رفتارهای موجودِ امن، اجباری و انتظار محدود را حفظ می‌کند، اما به‌جای launchd،‏ systemd یا Task Scheduler، Gateway تأییدشدهٔ در حال اجرا را هدف می‌گیرد.
- عملیات بومی نصب، شروع، توقف و حذف سرویس رد می‌شوند و راهنمای استفاده از ناظر خارجی ارائه می‌شود.
- به‌روزرسانی خودکار OpenClaw رد می‌شود تا ناظر بتواند Gateway را متوقف کند، محیط اجرا را جایگزین و نهایی کند و سپس آن را به‌صورت امن دوباره راه‌اندازی کند.
- راه‌اندازی مجدد با پردازه‌ای تازه، پیش از خروج پاک یک تحویل محدودشده را در SQLite می‌نویسد. اگر ماندگارکردن شکست بخورد، Gateway به‌جای خروج بدون تحویل قابل‌مصرف، به راه‌اندازی مجدد درون‌پردازه‌ای بازمی‌گردد.

`OPENCLAW_SERVICE_REPAIR_POLICY=external` همچنان یک سیاست ترمیم جداگانهٔ Doctor است. این متغیر مالکیت محیط اجرا را اعلام نمی‌کند؛ ناظرانی که به هر دو رفتار نیاز دارند باید هر دو متغیر را تنظیم کنند.

ناظران خارجی می‌توانند از طریق قرارداد ماشینی پنهان، دربارهٔ تحویل‌های راه‌اندازی مجدد مذاکره کنند و آن‌ها را مصرف کنند:

```bash
openclaw gateway restart-handoff capabilities --json
openclaw gateway restart-handoff consume --expected-pid <pid> --json
```

نسخهٔ پروتکل `1` از عملیات `consume` پشتیبانی می‌کند. مصرف، PID مورد انتظار و فیلدهای محدودشدهٔ تحویل را درون یک تراکنش فوری SQLite اعتبارسنجی می‌کند. تحویل پذیرفته‌شده پیش از بازگرداندن موفقیت حذف می‌شود؛ بنابراین مصرف‌کنندگان هم‌زمان یا تکراری نمی‌توانند هر دو آن را بپذیرند. عدم تطابق PID برای مالک منطبق نگه داشته می‌شود؛ ردیف‌های مفقود، منقضی و نامعتبر اجازهٔ راه‌اندازی مجدد نمی‌دهند.

درخواست‌های ماشینی معتبر، JSON را با کد خروج `0` بازمی‌گردانند؛ نتایج بدون راه‌اندازی مجدد نیز شامل آن هستند. آرگومان‌های نامعتبر `reason: "invalid-expected-pid"` را با کد خروج `2` بازمی‌گردانند؛ خرابی‌های ذخیره‌گاه وضعیت `reason: "store-unavailable"` را با کد خروج `1` بازمی‌گردانند. ناظران باید `capabilities` را دقیقاً روی همان محیط اجرا یا راه‌انداز مورد استفاده بررسی کنند، نه اینکه پشتیبانی را از رشتهٔ نسخهٔ OpenClaw استنباط کنند یا طرح‌وارهٔ خصوصی SQLite را مستقیماً بخوانند.

### پروفایل‌گیری Gateway

- `OPENCLAW_GATEWAY_STARTUP_TRACE=1` زمان‌بندی مرحله‌ها را هنگام راه‌اندازی ثبت می‌کند؛ از جمله تأخیر `eventLoopMax` برای هر مرحله و زمان‌بندی جدول جست‌وجوی Plugin (فهرست نصب‌شده، رجیستری manifest، برنامه‌ریزی راه‌اندازی، کارهای نگاشت مالک).
- `OPENCLAW_GATEWAY_RESTART_TRACE=1` خطوط `restart trace:` مختص راه‌اندازی مجدد را ثبت می‌کند: مدیریت سیگنال، تخلیهٔ کار فعال، مراحل خاموش‌شدن، شروع بعدی، زمان آماده‌شدن و معیارهای حافظه.
- `OPENCLAW_DIAGNOSTICS=timeline` همراه با `OPENCLAW_DIAGNOSTICS_TIMELINE_PATH=<path>` یک خط زمانی تشخیصی JSONL را به‌صورت best-effort برای هارنس‌های QA خارجی می‌نویسد (معادل پیکربندی `diagnostics.flags: ["timeline"]`؛ مسیر همچنان فقط از طریق محیط تعیین می‌شود). برای گنجاندن نمونه‌های حلقهٔ رویداد، `OPENCLAW_DIAGNOSTICS_EVENT_LOOP=1` را اضافه کنید.
- `pnpm build` و سپس `pnpm test:startup:gateway -- --runs 5 --warmup 1`، راه‌اندازی Gateway را در برابر ورودی CLI ساخته‌شده بنچمارک می‌کند: نخستین خروجی پردازه، `/healthz`، `/readyz`، زمان‌بندی ردگیری راه‌اندازی، تأخیر حلقهٔ رویداد و زمان‌بندی جدول جست‌وجوی Plugin.
- `pnpm build` و سپس `pnpm test:restart:gateway -- --case skipChannels --runs 1 --restarts 5`، راه‌اندازی مجدد درون‌پردازه‌ای را در macOS یا Linux بنچمارک می‌کند (در Windows پشتیبانی نمی‌شود؛ راه‌اندازی مجدد به `SIGUSR1` نیاز دارد). از `SIGUSR1` استفاده می‌کند، هر دو ردگیری را در پردازهٔ فرزند فعال می‌کند و `/healthz` بعدی، `/readyz` بعدی، زمان قطعی، زمان آماده‌شدن، CPU،‏ RSS و معیارهای ردگیری راه‌اندازی مجدد را ثبت می‌کند.
- `/healthz` نشان‌دهندهٔ زنده‌بودن است؛ `/readyz` نشان‌دهندهٔ آمادگی قابل‌استفاده است. خطوط ردگیری و خروجی بنچمارک را نشانه‌ای برای انتساب به مالک در نظر بگیرید، نه نتیجه‌گیری کامل عملکردی بر پایهٔ یک بازه یا نمونه.

## پرس‌وجو از Gateway در حال اجرا

همهٔ فرمان‌های پرس‌وجو از RPC مبتنی بر WebSocket استفاده می‌کنند.

<Tabs>
  <Tab title="حالت‌های خروجی">
    - پیش‌فرض: خوانا برای انسان (رنگی در TTY).
    - `--json`:‏ JSON خوانا برای ماشین (بدون سبک‌دهی/نشانگر چرخان).
    - `--no-color` (یا `NO_COLOR=1`): غیرفعال‌کردن ANSI با حفظ چیدمان انسانی.

  </Tab>
  <Tab title="گزینه‌های مشترک">
    - `--url <url>`: نشانی WebSocket متعلق به Gateway.
    - `--token <token>`: توکن Gateway.
    - `--password <password>`: گذرواژهٔ Gateway.
    - `--timeout <ms>`: مهلت زمانی/بودجه (پیش‌فرض برای هر فرمان متفاوت است؛ هر فرمان را در ادامه ببینید).
    - `--expect-final`: انتظار برای پاسخ «نهایی» (فراخوانی‌های عامل).

  </Tab>
</Tabs>

<Note>
وقتی `--url` را تنظیم می‌کنید، CLI به اطلاعات اعتبارسنجی موجود در پیکربندی یا محیط بازنمی‌گردد. `--token` یا `--password` را صریحاً ارسال کنید. نبود اطلاعات اعتبارسنجی صریح یک خطا است.
</Note>

### `gateway health`

```bash
openclaw gateway health --url ws://127.0.0.1:18789
openclaw gateway health --port 18789
```

`/healthz` یک کاوشگر زنده‌بودن است: به‌محض اینکه سرور بتواند به HTTP پاسخ دهد، برمی‌گردد. `/readyz` سخت‌گیرانه‌تر است و تا زمانی که پردازه‌های جانبی Plugin هنگام راه‌اندازی، کانال‌ها یا هوک‌های پیکربندی‌شده همچنان در حال پایدارشدن باشند، قرمز می‌ماند. پاسخ‌های تفصیلی محلی یا احرازهویت‌شدهٔ `/readyz` شامل یک بلوک تشخیصی `eventLoop` هستند (تأخیر، میزان استفاده، نسبت هستهٔ CPU، پرچم `degraded`).

<ParamField path="--port <port>" type="number">
  یک Gateway محلی روی رابط loopback را در این پورت هدف قرار دهید. برای این فراخوانی، `OPENCLAW_GATEWAY_URL` و `OPENCLAW_GATEWAY_PORT` را نادیده می‌گیرد.
</ParamField>

### `gateway usage-cost`

خلاصه‌های هزینهٔ استفاده را از گزارش‌های نشست دریافت می‌کند.

```bash
openclaw gateway usage-cost
openclaw gateway usage-cost --days 7
openclaw gateway usage-cost --agent work --json
openclaw gateway usage-cost --all-agents
openclaw gateway usage-cost --json
```

<ParamField path="--days <days>" type="number" default="30">
  تعداد روزهایی که باید لحاظ شوند.
</ParamField>
<ParamField path="--agent <id>" type="string">
  دامنهٔ خلاصه را به شناسهٔ یک عامل پیکربندی‌شده محدود می‌کند.
</ParamField>
<ParamField path="--all-agents" type="boolean">
  داده‌های همهٔ عامل‌های پیکربندی‌شده را تجمیع می‌کند. نمی‌توان آن را با `--agent` ترکیب کرد.
</ParamField>

### `gateway stability`

ثبت‌کنندهٔ تشخیصی پایداری اخیر را از یک Gateway در حال اجرا دریافت می‌کند.

```bash
openclaw gateway stability
openclaw gateway stability --type payload.large
openclaw gateway stability --bundle latest
openclaw gateway stability --bundle latest --export
openclaw gateway stability --json
```

<ParamField path="--limit <limit>" type="number" default="25">
  حداکثر تعداد رویدادهای اخیر برای درج (حداکثر `1000`).
</ParamField>
<ParamField path="--type <type>" type="string">
  بر اساس نوع رویداد تشخیصی فیلتر می‌کند؛ برای مثال `payload.large` یا `diagnostic.memory.pressure`.
</ParamField>
<ParamField path="--since-seq <seq>" type="number">
  فقط رویدادهای پس از یک شمارهٔ توالی تشخیصی را درج می‌کند.
</ParamField>
<ParamField path="--bundle [path]" type="string">
  به‌جای فراخوانی Gateway در حال اجرا، یک بستهٔ پایداری ذخیره‌شده را می‌خواند. `--bundle latest` (یا صرفاً `--bundle`) جدیدترین بسته را در پوشهٔ وضعیت انتخاب می‌کند؛ همچنین می‌توانید مسیر JSON یک بسته را مستقیماً ارائه کنید.
</ParamField>
<ParamField path="--export" type="boolean">
  به‌جای چاپ جزئیات پایداری، یک فایل فشردهٔ تشخیصی قابل‌اشتراک برای پشتیبانی می‌نویسد.
</ParamField>
<ParamField path="--output <path>" type="string">
  مسیر خروجی برای `--export`.
</ParamField>

<AccordionGroup>
  <Accordion title="حریم خصوصی و رفتار بسته">
    - رکوردها فراداده‌های عملیاتی را نگه می‌دارند: نام رویدادها، تعدادها، اندازه‌های بایتی، مقادیر حافظه، وضعیت صف/نشست، شناسه‌های تأیید، نام کانال‌ها/Pluginها و خلاصه‌های ویرایش‌شدهٔ نشست. آن‌ها متن گفت‌وگو، بدنه‌های Webhook، خروجی ابزارها، بدنه‌های خام درخواست/پاسخ، توکن‌ها، کوکی‌ها، مقادیر محرمانه، نام‌های میزبان و شناسه‌های خام نشست را مستثنا می‌کنند. برای غیرفعال‌کردن کامل ثبت‌کننده، `diagnostics.enabled: false` را تنظیم کنید.
    - خروج‌های مرگبار Gateway، مهلت‌های پایان‌یافتهٔ خاموش‌سازی و شکست‌های راه‌اندازی مجدد، هنگامی که ثبت‌کننده رویدادهایی داشته باشد، همان عکس فوری تشخیصی را در `~/.openclaw/logs/stability/openclaw-stability-*.json` می‌نویسند. جدیدترین بسته را با `openclaw gateway stability --bundle latest` بررسی کنید؛ `--limit`، `--type` و `--since-seq` برای خروجی بسته نیز اعمال می‌شوند.

  </Accordion>
</AccordionGroup>

### `gateway diagnostics export`

یک فایل فشردهٔ تشخیصی محلی برای گزارش‌های اشکال می‌نویسد. برای مدل حریم خصوصی و محتوای بسته، به [برون‌بری اطلاعات تشخیصی](/fa/gateway/diagnostics) مراجعه کنید.

```bash
openclaw gateway diagnostics export
openclaw gateway diagnostics export --output openclaw-diagnostics.zip
openclaw gateway diagnostics export --json
```

<ParamField path="--output <path>" type="string">
  مسیر فایل فشردهٔ خروجی. پیش‌فرض، یک برون‌بری پشتیبانی در پوشهٔ وضعیت است.
</ParamField>
<ParamField path="--log-lines <count>" type="number" default="5000">
  حداکثر تعداد خطوط پاک‌سازی‌شدهٔ گزارش برای درج.
</ParamField>
<ParamField path="--log-bytes <bytes>" type="number" default="1000000">
  حداکثر تعداد بایت‌های گزارش برای بررسی.
</ParamField>
<ParamField path="--url <url>" type="string">
  نشانی WebSocket مربوط به Gateway برای عکس فوری سلامت.
</ParamField>
<ParamField path="--token <token>" type="string">
  توکن Gateway برای عکس فوری سلامت.
</ParamField>
<ParamField path="--password <password>" type="string">
  گذرواژهٔ Gateway برای عکس فوری سلامت.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="3000">
  مهلت عکس فوری وضعیت/سلامت.
</ParamField>
<ParamField path="--no-stability-bundle" type="boolean">
  جست‌وجوی بستهٔ پایداری ذخیره‌شده را نادیده می‌گیرد.
</ParamField>
<ParamField path="--json" type="boolean">
  مسیر نوشته‌شده، اندازه و مانیفست را در قالب JSON چاپ می‌کند.
</ParamField>

بستهٔ برون‌بری شامل این موارد است: `manifest.json` (فهرست فایل‌ها)، `summary.md` (خلاصهٔ Markdown)، `diagnostics.json` (خلاصهٔ سطح بالای پیکربندی/گزارش‌ها/کشف/پایداری/وضعیت/سلامت)، `config/sanitized.json`، `status/gateway-status.json`، `health/gateway-health.json`، `logs/openclaw-sanitized.jsonl` و در صورت وجود بسته، `stability/latest.json`.

این برون‌بری برای اشتراک‌گذاری طراحی شده است. جزئیات عملیاتی مفید برای اشکال‌زدایی — فیلدهای امن گزارش، نام زیرسامانه‌ها، کدهای وضعیت، مدت‌زمان‌ها، حالت‌های پیکربندی‌شده، پورت‌ها، شناسه‌های Plugin/ارائه‌دهنده، تنظیمات غیرمحرمانهٔ قابلیت‌ها و پیام‌های عملیاتی ویرایش‌شدهٔ گزارش — را نگه می‌دارد و متن گفت‌وگو، بدنه‌های Webhook، خروجی ابزارها، اعتبارنامه‌ها، کوکی‌ها، شناسه‌های حساب/پیام، متن اعلان/دستورالعمل، نام‌های میزبان و مقادیر محرمانه را حذف یا ویرایش می‌کند. هنگامی که یک پیام گزارش شبیه متن بار دادهٔ کاربر/گفت‌وگو/ابزار باشد (برای مثال "کاربر گفت"، "متن گفت‌وگو"، "خروجی ابزار"، "بدنهٔ Webhook")، برون‌بری فقط این واقعیت را نگه می‌دارد که پیامی حذف شده است، همراه با تعداد بایت‌های آن.

### `gateway status`

سرویس Gateway ‏(launchd/systemd/schtasks) را به‌همراه یک کاوش اختیاری اتصال/احراز هویت نمایش می‌دهد.

```bash
openclaw gateway status
openclaw gateway status --json
openclaw gateway status --require-rpc
```

<ParamField path="--url <url>" type="string">
  یک هدف کاوش صریح اضافه می‌کند. مقصد راه دور پیکربندی‌شده و localhost همچنان کاوش می‌شوند.
</ParamField>
<ParamField path="--token <token>" type="string">
  احراز هویت با توکن برای کاوش.
</ParamField>
<ParamField path="--password <password>" type="string">
  احراز هویت با گذرواژه برای کاوش.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  مهلت کاوش.
</ParamField>
<ParamField path="--no-probe" type="boolean">
  کاوش اتصال را نادیده می‌گیرد (نمای فقط سرویس).
</ParamField>
<ParamField path="--deep" type="boolean">
  سرویس‌های سطح سیستم را نیز اسکن می‌کند.
</ParamField>
<ParamField path="--require-rpc" type="boolean">
  کاوش اتصال را به کاوش خواندن ارتقا می‌دهد و در صورت شکست، با کد غیرصفر خارج می‌شود. نمی‌توان آن را با `--no-probe` ترکیب کرد.
</ParamField>

<AccordionGroup>
  <Accordion title="معنای وضعیت">
    - حتی هنگامی که پیکربندی محلی CLI وجود ندارد یا نامعتبر است، برای اطلاعات تشخیصی در دسترس می‌ماند.
    - خروجی پیش‌فرض، وضعیت سرویس، اتصال WebSocket و قابلیت احراز هویت قابل‌مشاهده در زمان دست‌دادن را اثبات می‌کند — نه عملیات خواندن/نوشتن/مدیریتی را.
    - کاوش‌ها برای احراز هویت نخستین‌بارهٔ دستگاه، بدون تغییر هستند: در صورت وجود توکن دستگاه ذخیره‌شده، از آن دوباره استفاده می‌کنند، اما هرگز صرفاً برای بررسی وضعیت، هویت جدیدی برای دستگاه CLI یا رکورد جفت‌سازی فقط‌خواندنی ایجاد نمی‌کنند.
    - در صورت امکان، SecretRefهای احراز هویت پیکربندی‌شده را برای احراز هویت کاوش برطرف می‌کند. اگر SecretRef موردنیازی برطرف نشده باشد، هنگام شکست اتصال/احراز هویت کاوش، `--json` مقدار `rpc.authWarning` را گزارش می‌کند؛ `--token`/`--password` را صریحاً ارائه کنید یا منبع راز را اصلاح کنید. پس از موفقیت کاوش، هشدارهای احراز هویت برطرف‌نشده نمایش داده نمی‌شوند.
    - خروجی JSON هنگامی که Gateway در حال اجرا آن را گزارش کند، شامل `gateway.version` است؛ اگر کاوش دست‌دادن نتواند فرادادهٔ نسخه را فراهم کند، `--require-rpc` می‌تواند به بار دادهٔ RPC مربوط به `status.runtimeVersion` بازگردد.
    - هنگامی که وجود یک سرویس در حال گوش‌دادن کافی نیست و سالم‌بودن RPC با دامنهٔ خواندن نیز لازم است، در اسکریپت‌ها/اتوماسیون از `--require-rpc` استفاده کنید.
    - `--deep` نصب‌های اضافی launchd/systemd/schtasks را اسکن می‌کند؛ هنگامی که چند سرویس شبیه Gateway پیدا شود، خروجی قابل‌خواندن برای انسان نکات پاک‌سازی را چاپ می‌کند (معمولاً در هر دستگاه یک Gateway اجرا کنید) و در صورت مرتبط‌بودن، واگذاری اخیر راه‌اندازی مجدد ناظر را گزارش می‌دهد.
    - `--deep` همچنین اعتبارسنجی پیکربندی را در حالت آگاه از Plugin ‏(`pluginValidation: "full"`) اجرا می‌کند و هشدارهای مانیفست Plugin را نمایش می‌دهد (برای مثال نبود فرادادهٔ پیکربندی کانال). `gateway status` پیش‌فرض، مسیر سریع فقط‌خواندنی را حفظ می‌کند که اعتبارسنجی Plugin را نادیده می‌گیرد.
    - خروجی قابل‌خواندن برای انسان، مسیر فایل گزارش برطرف‌شده را به‌همراه مسیرها/اعتبار پیکربندی CLI در مقایسه با سرویس دربر می‌گیرد تا به تشخیص انحراف پروفایل یا پوشهٔ وضعیت کمک کند.
    - خروجی قابل‌خواندن برای انسان شامل `Gateway heap:` با محدودیت اعمال‌شده و نحوهٔ استخراج تطبیقی آن است. خروجی JSON همان گزارش را به‌شکل `service.gatewayHeap` ارائه می‌کند.

  </Accordion>
  <Accordion title="بررسی‌های انحراف احراز هویت systemd در Linux">
    - بررسی‌های انحراف احراز هویت سرویس، هر دو `Environment=` و `EnvironmentFile=` را از واحد می‌خوانند (از جمله `%h`، مسیرهای نقل‌قول‌شده، چند فایل و فایل‌های اختیاری `-`).
    - ‏SecretRefهای `gateway.auth.token` را با استفاده از محیط ادغام‌شدهٔ زمان اجرا برطرف می‌کند (ابتدا محیط فرمان سرویس، سپس محیط فرایند به‌عنوان مسیر جایگزین).
    - هنگامی که احراز هویت با توکن عملاً فعال نباشد (تنظیم صریح `gateway.auth.mode` به `password`/`none`/`trusted-proxy`، یا تنظیم‌نبودن حالت در شرایطی که گذرواژه می‌تواند اولویت پیدا کند و هیچ توکن نامزدی نمی‌تواند اولویت پیدا کند)، بررسی‌های انحراف توکن از برطرف‌سازی توکن پیکربندی صرف‌نظر می‌کنند.

  </Accordion>
</AccordionGroup>

### `gateway probe`

فرمان «اشکال‌زدایی همه‌چیز». این فرمان همیشه موارد زیر را کاوش می‌کند:

- ‏Gateway راه دور پیکربندی‌شدهٔ شما (در صورت تنظیم)، و
- ‏localhost ‏(loopback)، **حتی اگر مقصد راه دور پیکربندی شده باشد**.

ارائهٔ `--url` آن هدف صریح را پیش از هر دو مورد اضافه می‌کند. خروجی قابل‌خواندن برای انسان، هدف‌ها را با `URL (explicit)`، `Remote (configured)` / `Remote (configured, inactive)` و `Local loopback` برچسب‌گذاری می‌کند.

<Note>
اگر چند هدف کاوش در دسترس باشند، همه چاپ می‌شوند. یک تونل SSH، نشانی TLS/proxy و نشانی راه دور پیکربندی‌شده می‌توانند حتی با پورت‌های انتقال متفاوت به یک Gateway اشاره کنند؛ `multiple_gateways` برای Gatewayهای در دسترسِ متمایز یا دارای هویت مبهم رزرو شده است. اجرای چند Gateway برای پروفایل‌های ایزوله پشتیبانی می‌شود (برای مثال یک ربات نجات)، اما بیشتر نصب‌ها یک Gateway واحد اجرا می‌کنند.
</Note>

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --port 18789
```

<ParamField path="--port <port>" type="number">
  از این پورت برای هدف کاوش loopback محلی و پورت راه دور تونل SSH استفاده می‌کند. بدون `--url`، این گزینه فقط هدف loopback محلی را به‌جای نشانی محیطی Gateway پیکربندی‌شده، پورت محیطی یا هدف‌های راه دور انتخاب می‌کند.
</ParamField>

<AccordionGroup>
  <Accordion title="تفسیر">
    - `Reachable: yes` یعنی دست‌کم یک هدف، اتصال WebSocket را پذیرفته است.
    - `Capability: read-only|write-capable|admin-capable|pairing-pending|connect-only` آنچه کاوش توانسته دربارهٔ احراز هویت اثبات کند، جدا از دسترس‌پذیری، گزارش می‌دهد.
    - `Read probe: ok` یعنی فراخوانی‌های تفصیلی RPC با دامنهٔ خواندن (`health`/`status`/`system-presence`/`config.get`) نیز موفق بوده‌اند.
    - `Read probe: limited - missing scope: operator.read` یعنی اتصال موفق بوده، اما RPC با دامنهٔ خواندن محدود است. این وضعیت به‌عنوان دسترس‌پذیری **تنزل‌یافته** گزارش می‌شود، نه شکست کامل.
    - `Read probe: failed` پس از `Connect: ok` یعنی WebSocket متصل شده، اما اطلاعات تشخیصی خواندن بعدی مهلتشان پایان یافته یا ناموفق بوده‌اند — این وضعیت نیز **تنزل‌یافته** است، نه غیرقابل‌دسترسی.
    - همانند `gateway status`، کاوش از احراز هویت ذخیره‌شدهٔ موجود دستگاه دوباره استفاده می‌کند، اما هویت نخستین‌بارهٔ دستگاه یا وضعیت جفت‌سازی ایجاد نمی‌کند.
    - کد خروج تنها زمانی غیرصفر است که هیچ‌یک از هدف‌های کاوش‌شده در دسترس نباشند.

  </Accordion>
  <Accordion title="خروجی JSON">
    سطح بالا:

    - `ok`: دست‌کم یک مقصد قابل دسترسی است.
    - `degraded`: دست‌کم یک مقصد اتصال را پذیرفت، اما عیب‌یابی کامل RPC جزئیات را به پایان نرساند.
    - `capability`: بهترین قابلیت مشاهده‌شده در میان مقصدهای قابل دسترسی (`read_only`، `write_capable`، `admin_capable`، `pairing_pending`، `connected_no_operator_scope` یا `unknown`).
    - `primaryTargetId`: بهترین مقصد برای در نظر گرفتن به‌عنوان برندهٔ فعال، به‌ترتیب: URL صریح، تونل SSH، مقصد راه‌دور پیکربندی‌شده، حلقهٔ بازگشتی محلی.
    - `warnings[]`: رکوردهای هشدار با تلاش حداکثری، شامل `code`، `message` و `targetIds` اختیاری.
    - `network`: راهنمای URL حلقهٔ بازگشتی محلی/tailnet که از پیکربندی فعلی و شبکهٔ میزبان استخراج شده است.
    - `discovery.timeoutMs` / `discovery.count`: بودجهٔ واقعی کشف/تعداد نتایج استفاده‌شده برای این نوبت وارسی.

    برای هر مقصد (`targets[].connect`): `ok` (دسترسی‌پذیری + طبقه‌بندی تنزل‌یافته)، `rpcOk` (موفقیت کامل RPC جزئیات)، `scopeLimited` (شکست RPC جزئیات به‌دلیل نبود دامنهٔ اپراتور).

    برای هر مقصد (`targets[].auth`): در صورت موجود بودن، `role` و `scopes` در `hello-ok` گزارش می‌شوند، به‌همراه طبقه‌بندی نمایش‌داده‌شدهٔ `capability`.

  </Accordion>
  <Accordion title="کدهای هشدار رایج">
    - `ssh_tunnel_failed`: راه‌اندازی تونل SSH ناموفق بود؛ فرمان به وارسی‌های مستقیم بازگشت.
    - `multiple_gateways`: هویت‌های متمایز Gateway قابل دسترسی بودند، یا OpenClaw نتوانست اثبات کند که مقصدهای قابل دسترسی همان Gateway هستند. تونل SSH، ‏URL پروکسی یا URL راه‌دور پیکربندی‌شده به همان Gateway باعث فعال‌شدن این هشدار نمی‌شود.
    - `auth_secretref_unresolved`: یک SecretRef احراز هویت پیکربندی‌شده برای مقصد ناموفق قابل حل نبود.
    - `probe_scope_limited`: اتصال WebSocket موفق بود، اما وارسی خواندن به‌دلیل نبود `operator.read` محدود شد.
    - `local_tls_runtime_unavailable`: ‏TLS محلی Gateway فعال است، اما OpenClaw نتوانست اثر انگشت گواهی محلی را بارگیری کند.

  </Accordion>
</AccordionGroup>

#### راه‌دور از طریق SSH (هم‌ترازی با برنامهٔ Mac)

حالت «Remote over SSH» برنامهٔ macOS از انتقال محلی پورت استفاده می‌کند تا Gateway راه‌دوری که فقط روی حلقهٔ بازگشتی در دسترس است، در `ws://127.0.0.1:<port>` قابل دسترسی شود.

معادل CLI:

```bash
openclaw gateway probe --ssh user@gateway-host
```

<ParamField path="--ssh <target>" type="string">
  `user@host` یا `user@host:port` (پورت پیش‌فرض `22` است).
</ParamField>
<ParamField path="--ssh-identity <path>" type="string">
  فایل هویت.
</ParamField>
<ParamField path="--ssh-auto" type="boolean">
  نخستین میزبان Gateway کشف‌شده را از نقطهٔ پایانی کشفِ حل‌شده (`local.` به‌همراه دامنهٔ گستردهٔ پیکربندی‌شده، در صورت وجود) به‌عنوان مقصد SSH انتخاب می‌کند. راهنماهای فقط-TXT نادیده گرفته می‌شوند.
</ParamField>

پیش‌فرض‌های پیکربندی (اختیاری): `gateway.remote.sshTarget`، `gateway.remote.sshIdentity`.

### `gateway call <method>`

ابزار کمکی سطح‌پایین RPC.

```bash
openclaw gateway call status
openclaw gateway call logs.tail --params '{"limit": 200}'
```

<ParamField path="--params <json>" type="string" default="{}">
  رشتهٔ شیء JSON برای پارامترها.
</ParamField>
<ParamField path="--url <url>" type="string">
  ‏URL وب‌سوکت Gateway.
</ParamField>
<ParamField path="--token <token>" type="string">
  توکن Gateway.
</ParamField>
<ParamField path="--password <password>" type="string">
  گذرواژهٔ Gateway.
</ParamField>
<ParamField path="--timeout <ms>" type="number" default="10000">
  بودجهٔ مهلت زمانی.
</ParamField>
<ParamField path="--expect-final" type="boolean">
  عمدتاً برای RPCهای سبک عامل که پیش از بار نهایی، رویدادهای میانی را به‌صورت جریانی ارسال می‌کنند.
</ParamField>
<ParamField path="--json" type="boolean">
  خروجی JSON قابل خواندن توسط ماشین.
</ParamField>

<Note>
`--params` باید JSON معتبر باشد و هر متد شکل پارامترهای خود را اعتبارسنجی می‌کند (فیلدهای اضافی یا با نام نادرست رد می‌شوند).
</Note>

## مدیریت سرویس Gateway

```bash
openclaw gateway install
openclaw gateway start
openclaw gateway stop
openclaw gateway restart
openclaw gateway uninstall
```

### نصب با یک پوشاننده

هنگامی از `--wrapper` استفاده کنید که سرویس مدیریت‌شده باید از طریق فایل اجرایی دیگری آغاز شود؛ برای مثال یک لایهٔ واسط مدیر اسرار یا ابزار اجرا با هویت دیگر. پوشاننده آرگومان‌های معمول Gateway را دریافت می‌کند و مسئول است در نهایت `openclaw` یا Node را با همان آرگومان‌ها اجرا کند.

```bash
cat > ~/.local/bin/openclaw-doppler <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
exec doppler run --project my-project --config production -- openclaw "$@"
EOF
chmod +x ~/.local/bin/openclaw-doppler

openclaw gateway install --wrapper ~/.local/bin/openclaw-doppler --force
openclaw gateway restart
```

همچنین می‌توانید پوشاننده را از طریق محیط تنظیم کنید. `gateway install` اعتبارسنجی می‌کند که مسیر یک فایل اجرایی باشد، پوشاننده را در `ProgramArguments` سرویس می‌نویسد و `OPENCLAW_WRAPPER` را برای نصب‌های مجدد اجباری، به‌روزرسانی‌ها و تعمیرات doctor بعدی در محیط سرویس پایدار می‌کند.

```bash
OPENCLAW_WRAPPER="$HOME/.local/bin/openclaw-doppler" openclaw gateway install --force
openclaw doctor
```

برای حذف پوشانندهٔ پایدارشده، هنگام نصب مجدد `OPENCLAW_WRAPPER` را پاک کنید:

```bash
OPENCLAW_WRAPPER= openclaw gateway install --force
openclaw gateway restart
```

<AccordionGroup>
  <Accordion title="گزینه‌های فرمان">
    - `gateway status`: `--url`، `--token`، `--password`، `--timeout`، `--no-probe`، `--require-rpc`، `--deep`، `--json`
    - `gateway install`: `--port`، `--runtime <node>` (پیش‌فرض: `node`)، `--token`، `--wrapper <path>`، `--force`، `--json`
    - `gateway restart`: `--safe`، `--skip-deferral`، `--force`، `--wait <duration>`، `--json`
    - `gateway uninstall|start`: `--json`
    - `gateway stop`: `--disable`، `--force`، `--json`

  </Accordion>
  <Accordion title="رفتار چرخهٔ عمر">
    - `gateway start` ایدمپوتنت است: وقتی سرویس مدیریت‌شده از قبل در حال اجرا باشد، فرایند در حال اجرا را گزارش می‌کند و آن را دست‌نخورده باقی می‌گذارد. سرویس بارگیری‌شده اما متوقف، مانند قبل آغاز می‌شود.
    - برای راه‌اندازی مجدد سرویس مدیریت‌شده از `gateway restart` استفاده کنید. `gateway stop` و `gateway start` را به‌عنوان جایگزین راه‌اندازی مجدد زنجیره نکنید.
    - در پوستهٔ غیرتعاملی، `gateway stop` به `--force` نیاز دارد. پایانه‌های تعاملی رفتار فعلی بدون اعلان را حفظ می‌کنند. برای خودکارسازی و آزمون‌ها، `gateway run --dev` یا یک `--profile` ایزوله با پورتی آزاد را ترجیح دهید.
    - در macOS، ‏`gateway stop` به‌طور پیش‌فرض از `launchctl bootout` استفاده می‌کند که LaunchAgent را بدون پایدارسازی غیرفعال‌سازی از نشست راه‌اندازی فعلی حذف می‌کند — بازیابی خودکار KeepAlive برای خرابی‌های آینده فعال می‌ماند و `gateway start` بدون نیاز به `launchctl enable` دستی، دوباره به‌درستی فعال می‌شود. برای سرکوب پایدار KeepAlive و RunAtLoad، ‏`--disable` را ارسال کنید تا Gateway تا `gateway start` صریح بعدی دوباره ایجاد نشود؛ وقتی توقف دستی باید پس از راه‌اندازی مجدد سیستم نیز باقی بماند، از این گزینه استفاده کنید.
    - تغییرات چرخهٔ عمر Gateway رکوردهای ممیزی کلید-مقدار را با تلاش حداکثری به `<state-dir>/logs/gateway-restart.log` می‌افزایند، از جمله عملیات آغاز، توقف و راه‌اندازی مجدد CLI، درخواست‌های راه‌اندازی مجدد ایمن، راه‌اندازی‌های مجدد ناظر و واگذاری‌های جداشده.
    - فرمان‌های چرخهٔ عمر برای اسکریپت‌نویسی `--json` را می‌پذیرند.

  </Accordion>
  <Accordion title="اندازه‌گذاری heap برای Gateway مدیریت‌شده">
    - `gateway install` یک مقدار `NODE_OPTIONS` مختص heap برای سرویس Gateway مدیریت‌شده می‌نویسد. وقتی Node محدودیت کانتینر یا سرویس را گزارش کند، 50% حافظهٔ محدودشده و در غیر این صورت 50% حافظهٔ فیزیکی را هدف قرار می‌دهد.
    - بازهٔ هدف اسمی 2048–8192 MiB است، با سقف اضافی 75% برای فضای آزاد حافظهٔ بومی. در میزبان‌های کوچک، این سقف فضای آزاد می‌تواند حد اعمال‌شده را به کمتر از کف اسمی 2048 MiB برساند.
    - یک `--max-old-space-size` صریح و معتبر که از قبل در سرویس نصب‌شده ذخیره شده باشد، در نصب‌های مجدد اجباری و تعمیرات doctor حفظ می‌شود. دیگر پرچم‌های `NODE_OPTIONS` به سرویس مدیریت‌شده منتقل نمی‌شوند.
    - `NODE_OPTIONS` محیطی پوسته این خط‌مشی را بازنویسی نمی‌کند. برای بررسی مقدار نصب‌شده از `gateway status` یا `doctor` استفاده کنید؛ برای بازتولید فرادادهٔ سرویس‌های قدیمی که تنظیم heap مدیریت‌شده ندارند، `openclaw gateway install --force` را اجرا کنید.
    - این خط‌مشی فقط برای سرویس Gateway مدیریت‌شده اعمال می‌شود. `gateway run` پیش‌زمینه، سرویس‌های Node و واحدهای ناظر دست‌نویس، پیکربندی زمان اجرای خود را حفظ می‌کنند.

  </Accordion>
  <Accordion title="احراز هویت و SecretRefها هنگام نصب">
    - وقتی احراز هویت توکنی به توکن نیاز دارد و `gateway.auth.token` توسط SecretRef مدیریت می‌شود، `gateway install` قابل حل بودن SecretRef را اعتبارسنجی می‌کند، اما توکن حل‌شده را در فرادادهٔ محیط سرویس پایدار نمی‌کند.
    - اگر احراز هویت توکنی به توکن نیاز داشته باشد و SecretRef توکن پیکربندی‌شده حل‌نشده باشد، نصب به‌صورت بسته شکست می‌خورد و متن سادهٔ جایگزین را پایدار نمی‌کند.
    - برای احراز هویت با گذرواژه روی `gateway run`، ‏`OPENCLAW_GATEWAY_PASSWORD`، ‏`--password-file` یا `gateway.auth.password` مبتنی بر SecretRef را به `--password` درون‌خطی ترجیح دهید.
    - در حالت احراز هویت استنباط‌شده، `OPENCLAW_GATEWAY_PASSWORD` مختص پوسته الزامات توکن نصب را تسهیل نمی‌کند؛ هنگام نصب سرویس مدیریت‌شده از پیکربندی پایدار (`gateway.auth.password` یا `env` پیکربندی) استفاده کنید.
    - اگر هم `gateway.auth.token` و هم `gateway.auth.password` پیکربندی شده باشند و `gateway.auth.mode` تنظیم نشده باشد، نصب تا زمانی که حالت به‌صراحت تنظیم شود مسدود می‌ماند.

  </Accordion>
</AccordionGroup>

## کشف Gatewayها (Bonjour)

`gateway discover` برای بیکن‌های Gateway پویش می‌کند (`_openclaw-gw._tcp`).

- ‏DNS-SD چندپخشی: `local.`
- ‏DNS-SD تک‌پخشی (Bonjour گسترده): یک دامنه انتخاب کنید (مثال: `openclaw.internal.`) و DNS تقسیم‌شده + یک سرور DNS را راه‌اندازی کنید؛ [Bonjour](/fa/gateway/bonjour) را ببینید.

فقط Gatewayهایی که کشف Bonjour در آن‌ها فعال است (پیش‌فرض)، بیکن را تبلیغ می‌کنند.

راهنماهای TXT روی هر بیکن: `role` (راهنمای نقش Gateway)، `transport` (راهنمای انتقال، برای مثال `gateway`)، `gatewayPort` (پورت WebSocket، معمولاً `18789`)، `tailnetDns` (نام میزبان MagicDNS، در صورت موجود بودن)، `gatewayTls` / `gatewayTlsSha256` (فعال بودن TLS + اثر انگشت گواهی). `sshPort` و `cliPath` فقط در حالت کشف کامل منتشر می‌شوند (`discovery.mdns.mode: "full"`؛ پیش‌فرض `"minimal"` است که آن‌ها را حذف می‌کند — در این صورت، کلاینت‌ها مقصدهای SSH را به‌طور پیش‌فرض روی پورت `22` قرار می‌دهند).

### `gateway discover`

```bash
openclaw gateway discover
```

<ParamField path="--timeout <ms>" type="number" default="2000">
  مهلت زمانی هر فرمان (مرور/حل).
</ParamField>
<ParamField path="--json" type="boolean">
  خروجی قابل خواندن توسط ماشین (همچنین سبک‌دهی/نشانگر چرخان را غیرفعال می‌کند).
</ParamField>

مثال‌ها:

```bash
openclaw gateway discover --timeout 4000
openclaw gateway discover --json | jq '.beacons[].wsUrl'
```

<Note>
- ‏`local.` را به‌همراه دامنهٔ گستردهٔ پیکربندی‌شده، در صورت فعال بودن، پویش می‌کند.
- `wsUrl` در خروجی JSON از نقطهٔ پایانی سرویس حل‌شده استخراج می‌شود، نه از راهنماهای فقط-TXT مانند `lanHost` یا `tailnetDns`.
- `discovery.mdns.mode` انتشار `sshPort`/`cliPath` را هم در mDNS ‏`local.` و هم در DNS-SD گسترده کنترل می‌کند (بالا را ببینید).

</Note>

## مرتبط

- [مرجع CLI](/fa/cli)
- [راهنمای عملیاتی Gateway](/fa/gateway)
