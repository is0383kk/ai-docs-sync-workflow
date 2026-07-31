---
read_when:
    - می‌خواهید تأییدهای exec را از طریق CLI ویرایش کنید
    - باید فهرست‌های مجاز را روی میزبان‌های Gateway یا Node مدیریت کنید
    - باید یک تأیید در انتظار را بدون رابط چت فهرست یا تعیین‌تکلیف کنید
summary: مرجع CLI برای `openclaw approvals` و `openclaw exec-policy`
title: تأییدها
x-i18n:
    generated_at: "2026-07-27T13:56:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8b6f198af718d7b058498dbb960a1eb68ced601e1cd9205070b7199688552d2
    source_path: cli/approvals.md
    workflow: 16
---

# `openclaw approvals`

تأییدهای exec را برای **میزبان محلی**، **میزبان Gateway** یا یک **میزبان Node** مدیریت کنید. اگر پرچم مقصدی مشخص نشود، فرمان‌ها فایل تأییدهای محلی روی دیسک را می‌خوانند/می‌نویسند. برای هدف‌گیری Gateway از `--gateway` و برای هدف‌گیری یک Node مشخص از `--node <id|name|ip>` استفاده کنید.

نام مستعار: `openclaw exec-approvals`

مرتبط: [تأییدهای exec](/fa/tools/exec-approvals)، [Nodeها](/fa/nodes)

## `openclaw exec-policy`

`openclaw exec-policy` فرمانی تسهیل‌کننده و **مختص محیط محلی** است که پیکربندی درخواستی `tools.exec.*` و فایل تأییدهای میزبان محلی را در یک مرحله همگام نگه می‌دارد:

```bash
openclaw exec-policy show
openclaw exec-policy show --json

openclaw exec-policy preset yolo
openclaw exec-policy preset cautious --json

openclaw exec-policy set --host gateway --security full --ask off --ask-fallback full
```

پیش‌تنظیم‌ها (`yolo`، `cautious`، `deny-all`) مقادیر `host`، `security`، `ask` و `askFallback` را با هم اعمال می‌کنند. `set` فقط پرچم‌هایی را که ارسال می‌کنید اعمال می‌کند؛ هر مقدار پذیرفته‌شده اعتبارسنجی می‌شود (`--host auto|sandbox|gateway|node`، `--security deny|allowlist|full`، `--ask off|on-miss|always`، `--ask-fallback deny|allowlist|full`).

دامنه:

- فایل پیکربندی محلی و فایل تأییدهای محلی را با هم به‌روزرسانی می‌کند؛ سیاست را به Gateway یا میزبان Node ارسال نمی‌کند.
- `--host node` رد می‌شود: تأییدهای exec مربوط به Node هنگام اجرا از خود Node دریافت می‌شوند، بنابراین `exec-policy` محلی نمی‌تواند آن‌ها را همگام کند. به‌جای آن از `openclaw approvals set --node <id|name|ip>` استفاده کنید.
- `exec-policy show` دامنه‌های `host=node` را هنگام اجرا به‌عنوان تحت مدیریت Node علامت‌گذاری می‌کند، به‌جای آنکه یک سیاست مؤثر از فایل تأییدهای محلی استخراج کند.

برای تأییدهای میزبان راه‌دور، مستقیماً از `openclaw approvals set --gateway` یا `openclaw approvals set --node <id|name|ip>` استفاده کنید.

## فرمان‌های رایج

```bash
openclaw approvals get
openclaw approvals get --node <id|name|ip>
openclaw approvals get --gateway
openclaw approvals pending
openclaw approvals resolve <id> <allow-once|allow-always|deny>
```

`get` سیاست مؤثر exec را برای مقصد نشان می‌دهد: سیاست درخواستی `tools.exec`، سیاست فایل تأییدهای میزبان و نتیجه مؤثر ادغام‌شده. Nodeهایی که سیاست بومی میزبان دارند، مانند برنامه همراه Windows، آن سیاست را مستقیماً نشان می‌دهند و محاسبات سیاست فایل تأییدهای OpenClaw را اعمال نمی‌کنند.

برای Nodeهای مبتنی بر فایل، نمای ادغام‌شده به یک اسنپ‌شات سیاست تفکیک‌شده توسط میزبان نیاز دارد. Nodeهای قدیمی‌تر، به‌جای فرض اینکه سیاست درخواستی Gateway روی میزبان نیز اعمال می‌شود، سیاست مؤثر را دردسترس‌نبودنی نشان می‌دهند.

<Note>
بازنویسی‌های مختص هر نشست در `/exec` لحاظ نمی‌شوند. برای بررسی پیش‌فرض‌های فعلی آن، `/exec` را در نشست مربوطه اجرا کنید.
</Note>

اولویت:

- فایل تأییدهای میزبان، منبع حقیقت قابل‌اعمال است.
- سیاست درخواستی `tools.exec` می‌تواند دامنه قصد را محدودتر یا گسترده‌تر کند، اما نتیجه مؤثر از قواعد میزبان استخراج می‌شود.
- `--node` فایل تأییدهای میزبان Node را با سیاست `tools.exec` مربوط به Gateway ترکیب می‌کند (هر دو هنگام اجرا اعمال می‌شوند).
- اگر پیکربندی Gateway دردسترس نباشد، CLI به اسنپ‌شات تأییدهای Node بازمی‌گردد و یادآوری می‌کند که سیاست نهایی زمان اجرا قابل محاسبه نبوده است.

## تأییدهای در انتظار

تأییدهای در انتظار exec، Plugin و عامل سیستمی OpenClaw را از Gateway فهرست کنید:

```bash
openclaw approvals pending
openclaw approvals pending --json
```

فهرست‌سازی کامل و جریان متناظر `resolve` در سطح همه اپراتورها از `operator.admin` استفاده می‌کنند، زیرا در غیر این صورت رکوردهای تأیید، فیلتر درخواست‌کننده/بازبین را حفظ می‌کنند. فرایند حل‌وفصل همچنین دامنه اختصاصی `operator.approvals` را درخواست می‌کند. مجوز استاندارد اپراتور CLI هر دو دامنه را شامل می‌شود؛ یک کلاینت شخص ثالث محدود نباید صرفاً برای شبیه‌سازی این فرمان، دسترسی مدیر درخواست کند.

خروجی قابل‌خواندن برای انسان، نوع تأیید، انتساب عامل/نشست، عمر درخواست، زمان باقی‌مانده تا انقضا، یک فرمان یا خلاصه کوتاه‌شده و یک توکن شناسه `id64_<base64url>` مستقل از پوسته را نشان می‌دهد. پس از جدول فشرده، همیشه یک بلوک `Full request text` شامل همه توکن‌های کامل و درخواستِ بدون‌اتلاف escapeشده نمایش داده می‌شود تا کوتاه‌سازی متناسب با عرض ترمینال نتواند پسوند یا توکن لازم برای حل‌وفصل را پنهان کند. توکن کامل را در `resolve` کپی کنید. نویسه‌های ناامن ترمینال در فیلدهای دیگر به‌صورت escapeهای قابل‌مشاهده Unicode نمایش داده می‌شوند. خروجی JSON ورودی‌های نرمال‌شده را زیر `approvals` برمی‌گرداند و مقادیر خام اصلی `id`، `summary`، `createdAtMs` و `expiresAtMs` را برای اسکریپت‌ها حفظ می‌کند؛ شناسه‌های خام همچنان توسط `resolve` پذیرفته می‌شوند، مگر آنکه از پیشوند رزروشده توکن نمایشی `id64_` استفاده کنند.

اگر مقدار ارائه‌شده `id64_` هم با یک شناسه خام عینی و هم با توکن نمایشی رمزگشایی‌شده تأییدی دیگر مطابقت داشته باشد، CLI به‌جای به‌خطرانداختن حل‌وفصل درخواست اشتباه، آن را مبهم تشخیص داده و رد می‌کند.

یک تأیید را با شناسه کامل آن حل‌وفصل کنید:

```bash
openclaw approvals resolve <id> allow-once
openclaw approvals resolve <id> allow-always
openclaw approvals resolve <id> deny --reason "در زمان نگهداری انتظار نمی‌رود"
```

CLI رکورد یکپارچه تأیید را می‌خواند تا نوع آن را انتخاب کند، تصمیم درخواستی را با تصمیم‌های مجاز رکورد تطبیق می‌دهد و سپس حل‌کننده یکپارچه را فراخوانی می‌کند. نخستین تصمیم موفق با `0` خارج می‌شود. تکرار تصمیم ثبت‌شده نیز با `0` خارج می‌شود و `already resolved (same decision)` را گزارش می‌کند. تصمیم متناقض، تأیید مفقود، تأیید منقضی‌شده یا تصمیمی که برای آن نوع تأیید دردسترس نیست، خطایی روشن چاپ می‌کند و با کد غیرصفر خارج می‌شود.

`--reason` یک یادداشت محلی به تأییدیه CLI اضافه می‌کند. رکورد فعلی تأیید Gateway فیلد آزاد برای دلیل حل‌وفصل ندارد، بنابراین این یادداشت ذخیره نمی‌شود یا به سطوح تأیید دیگر ارسال نمی‌شود.

## جایگزینی تأییدها از یک فایل

```bash
openclaw approvals set --file ./exec-approvals.json
openclaw approvals set --stdin <<'EOF'
{ version: 1, defaults: { security: "full", ask: "off", askFallback: "full" } }
EOF
openclaw approvals set --node <id|name|ip> --file ./exec-approvals.json
openclaw approvals set --gateway --file ./exec-approvals.json
```

`set` علاوه بر JSON سخت‌گیرانه، JSON5 را نیز می‌پذیرد. از `--file` یا `--stdin` استفاده کنید، نه هر دو.

Nodeهای Windows با سیاست بومی میزبان از قالب سیاست خودشان استفاده می‌کنند:

```bash
openclaw approvals set --node <id|name|ip> --stdin <<'EOF'
{
  defaultAction: "deny",
  rules: [{ pattern: "hostname", action: "allow" }]
}
EOF
```

CLI ابتدا هش فعلی Node را می‌خواند و آن را همراه به‌روزرسانی ارسال می‌کند تا ویرایش‌های محلی هم‌زمان به‌جای بازنویسی‌شدن، رد شوند. `rules` الزامی است، زیرا این عملیات فهرست کامل قواعد Node را جایگزین می‌کند؛ `defaultAction` اختیاری است. Nodeای که سیاست بومی خود را غیرفعال گزارش می‌کند، از راه دور قابل پیکربندی نیست؛ ابتدا سیاست را روی آن میزبان فعال یا پیکربندی کنید. سیاست‌های بومی میزبان از ابزارهای کمکی `allowlist add|remove` پشتیبانی نمی‌کنند.

## نمونه «هرگز درخواست نکن» / YOLO

پیش‌فرض‌های تأیید میزبان را برای میزبانی که هرگز نباید به‌دلیل تأییدهای exec متوقف شود، روی `full` + `off` تنظیم کنید:

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "full",
    ask: "off",
    askFallback: "full"
  }
}
EOF
```

برای Nodeهایی که یک فایل تأیید OpenClaw ارائه می‌کنند، همین بدنه را با `openclaw approvals set --node <id|name|ip> --stdin` استفاده کنید. Nodeهای بومی میزبان به قالب مختص مالک خود که در بالا نشان داده شده است نیاز دارند.

این کار فقط **فایل تأییدهای میزبان** را تغییر می‌دهد. برای هم‌تراز نگه‌داشتن سیاست درخواستی OpenClaw، این موارد را نیز تنظیم کنید:

```bash
openclaw config set tools.exec.host gateway
openclaw config set tools.exec.mode full
```

`tools.exec.host=gateway` در اینجا صریح است، زیرا `host=auto` همچنان به‌معنای «در صورت امکان sandbox، در غیر این صورت Gateway» است: YOLO درباره تأییدهاست، نه مسیریابی. هنگامی که حتی با وجود sandbox پیکربندی‌شده، exec روی میزبان را می‌خواهید، از `gateway` (یا `/exec host=gateway`) استفاده کنید.

مقدار حذف‌شده `askFallback` به‌طور پیش‌فرض `deny` است. هنگام ارتقای میزبانی بدون رابط کاربری که باید رفتار بدون درخواست را حفظ کند، `askFallback: "full"` را صریحاً تنظیم کنید.

میان‌بر محلی برای همین منظور، فقط روی دستگاه محلی:

```bash
openclaw exec-policy preset yolo
```

## ابزارهای کمکی فهرست مجاز

```bash
openclaw approvals allowlist add "~/Projects/**/bin/rg"
openclaw approvals allowlist add --agent main --node <id|name|ip> "/usr/bin/uptime"
openclaw approvals allowlist add --agent "*" "/usr/bin/uname"

openclaw approvals allowlist remove "~/Projects/**/bin/rg"
```

## گزینه‌های رایج

`get`، `set` و `allowlist add|remove` همگی از موارد زیر پشتیبانی می‌کنند:

- `--node <id|name|ip>` (شناسه، نام، IP یا پیشوند شناسه را تفکیک می‌کند؛ همان تفکیک‌کننده `openclaw nodes`)
- `--gateway`
- گزینه‌های مشترک RPC مربوط به Node: `--url`، `--token`، `--timeout`، `--json`

نبود پرچم مقصد به‌معنای فایل تأییدهای محلی روی دیسک است.

`allowlist add|remove` همچنین از `--agent <id>` پشتیبانی می‌کند (مقدار پیش‌فرض `"*"` است و برای همه عامل‌ها اعمال می‌شود).

`pending` و `resolve` همیشه از Gateway استفاده می‌کنند، زیرا درخواست‌های در انتظار جزو وضعیت زنده Gateway هستند. آن‌ها از گزینه‌های مشترک اتصال Gateway یعنی `--url`، `--token` و `--timeout` پشتیبانی می‌کنند؛ `pending` همچنین از `--json` پشتیبانی می‌کند.

## یادداشت‌ها

- میزبان Node باید `system.execApprovals.get/set` را اعلام کند (برنامه macOS، میزبان Node بدون رابط گرافیکی یا برنامه همراه Windows).
- فایل‌های تأیید به‌ازای هر میزبان در دایرکتوری وضعیت OpenClaw ذخیره می‌شوند: `$OPENCLAW_STATE_DIR/exec-approvals.json`، یا زمانی که متغیر تنظیم نشده باشد `~/.openclaw/exec-approvals.json`.

## مرتبط

- [مرجع CLI](/fa/cli)
- [تأییدهای exec](/fa/tools/exec-approvals)
