---
read_when:
    - اشکال‌زدایی احراز هویت مدل یا انقضای OAuth
    - مستندسازی احراز هویت یا ذخیره‌سازی اطلاعات اعتبارنامه‌ای
summary: 'احراز هویت مدل: OAuth، کلیدهای API، استفادهٔ مجدد از Claude CLI و توکن راه‌اندازی Anthropic'
title: احراز هویت
x-i18n:
    generated_at: "2026-07-27T16:26:45Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1fd4bf1c73f41d297638811f568c1b11e920eba3bd1527206cbb760df51531f2
    source_path: gateway/authentication.md
    workflow: 16
---

<Note>
این صفحه احراز هویت **ارائه‌دهنده مدل** (کلیدهای API،‏ OAuth، استفاده مجدد از Claude CLI،‏ setup-token متعلق به Anthropic) را پوشش می‌دهد. برای احراز هویت **اتصال Gateway** (توکن، گذرواژه، trusted-proxy)، به [پیکربندی](/fa/gateway/configuration) و [احراز هویت پراکسی مورد اعتماد](/fa/gateway/trusted-proxy-auth) مراجعه کنید.
</Note>

OpenClaw از OAuth و کلیدهای API برای ارائه‌دهندگان مدل پشتیبانی می‌کند. برای میزبان Gateway همیشه‌روشن، کلید API قابل‌پیش‌بینی‌ترین گزینه است؛ جریان‌های اشتراک/OAuth نیز در صورتی کار می‌کنند که با مدل حساب ارائه‌دهنده شما مطابقت داشته باشند.

- جریان کامل OAuth و چیدمان ذخیره‌سازی: [/concepts/oauth](/fa/concepts/oauth)
- احراز هویت مبتنی بر SecretRef (ارائه‌دهندگان `env`/`file`/`exec`): [مدیریت اسرار](/fa/gateway/secrets)
- کدهای واجد شرایط بودن/دلیل اعتبارنامه که `models status --probe` استفاده می‌کند: [معناشناسی اعتبارنامه احراز هویت](/fa/auth-credential-semantics)

## راه‌اندازی توصیه‌شده: کلید API (هر ارائه‌دهنده‌ای)

1. در کنسول ارائه‌دهنده خود یک کلید API ایجاد کنید.
2. آن را روی **میزبان Gateway** (دستگاهی که `openclaw gateway` را اجرا می‌کند) قرار دهید:

```bash
export <PROVIDER>_API_KEY="..."
openclaw models status
```

3. اگر Gateway تحت systemd/launchd اجرا می‌شود، کلید را در `~/.openclaw/.env` قرار دهید تا دیمن بتواند آن را بخواند:

```bash
cat >> ~/.openclaw/.env <<'EOF'
<PROVIDER>_API_KEY=...
EOF
```

4. فرایند Gateway (یا دیمن) را راه‌اندازی مجدد کنید، سپس دوباره بررسی کنید:

```bash
openclaw models status
openclaw doctor
```

`openclaw onboard` نیز می‌تواند کلیدهای API را برای استفاده دیمن ذخیره کند، اگر نمی‌خواهید متغیرهای محیطی را خودتان مدیریت کنید. برای ترتیب تقدم کامل بارگذاری متغیرهای محیطی (`env.shellEnv`،‏ `~/.openclaw/.env`،‏ systemd/launchd)، به [متغیرهای محیطی](/fa/help/environment) مراجعه کنید.

## Anthropic: استفاده مجدد از Claude CLI

احراز هویت setup-token متعلق به Anthropic همچنان یک مسیر پشتیبانی‌شده است. استفاده مجدد از Claude CLI (کاربرد به سبک `claude -p`) نیز برای این یکپارچه‌سازی مجاز است؛ وقتی ورود Claude CLI روی میزبان موجود باشد، این مسیر برای استفاده محلی/دسکتاپ ترجیح داده می‌شود. برای میزبان‌های Gateway با عمر طولانی، کلید API متعلق به Anthropic همچنان قابل‌پیش‌بینی‌ترین انتخاب است و کنترل صریح صورت‌حساب سمت سرور را فراهم می‌کند.

راه‌اندازی میزبان برای استفاده مجدد از Claude CLI:

```bash
# روی میزبان Gateway اجرا کنید
claude auth login
claude auth status --text
openclaw models auth login --provider anthropic --method cli --set-default
```

این فرایند دو مرحله دارد: Claude Code را روی میزبان وارد Anthropic کنید، سپس به OpenClaw بگویید انتخاب مدل Anthropic را از طریق بک‌اند محلی `claude-cli` مسیریابی کند و پروفایل احراز هویت متناظر OpenClaw را ذخیره کند.

سرویس Gateway باید بتواند `claude` را در `PATH` پیدا کند. اگر استقرار به مسیر اجرایی
غیراستانداردی نیاز دارد، یک پوشش‌دهنده را از طریق
[Plugin بک‌اند CLI](/fa/plugins/cli-backend-plugins) ثبت کنید.

## ورود دستی توکن

برای هر ارائه‌دهنده‌ای کار می‌کند؛ مخزن احراز هویت SQLite مختص هر عامل را می‌نویسد و پیکربندی را به‌روزرسانی می‌کند:

```bash
openclaw models auth paste-token --provider openrouter
```

OpenClaw پروفایل‌های احراز هویت را از `openclaw-agent.sqlite` هر عامل می‌خواند. جزئیات نقطه پایانی (`baseUrl`،‏ `api`، شناسه‌های مدل، سرآیندها، مهلت‌های زمانی) باید زیر `models.providers.<id>` در `openclaw.json` یا `models.json` قرار گیرند، نه در پروفایل‌های احراز هویت.

اگر یک نصب قدیمی هنوز `auth-profiles.json`،‏ `auth-state.json` یا ساختاری تخت مانند `{ "openrouter": { "apiKey": "..." } }` دارد، `openclaw doctor --fix` را اجرا کنید تا آن را به SQLite وارد کند؛ doctor نسخه‌های پشتیبان دارای مُهر زمانی را کنار فایل‌های JSON اصلی نگه می‌دارد.

مسیرهای احراز هویت خارجی مانند `auth: "aws-sdk"` متعلق به Bedrock اعتبارنامه نیستند. برای یک مسیر نام‌گذاری‌شده Bedrock،‏ `auth.profiles.<id>.mode: "aws-sdk"` را در `openclaw.json` تنظیم کنید —‏ `type: "aws-sdk"` را در مخزن پروفایل احراز هویت ننویسید. `openclaw doctor --fix` نشانگرهای قدیمی AWS SDK را از مخزن اعتبارنامه به فراداده پیکربندی منتقل می‌کند.

### اعتبارنامه‌های مبتنی بر SecretRef

- اعتبارنامه‌های `api_key` می‌توانند از `keyRef: { source, provider, id }` استفاده کنند
- اعتبارنامه‌های `token` می‌توانند از `tokenRef: { source, provider, id }` استفاده کنند
- پروفایل‌های حالت OAuth اعتبارنامه‌های SecretRef را رد می‌کنند: اگر `auth.profiles.<id>.mode` برابر `"oauth"` باشد، یک `keyRef`/`tokenRef` مبتنی بر SecretRef برای آن پروفایل رد می‌شود.

## بررسی وضعیت احراز هویت مدل

```bash
openclaw models status
openclaw doctor
```

بررسی مناسب برای خودکارسازی؛ هنگام انقضا/فقدان با کد `1` و هنگام نزدیک بودن انقضا با کد `2` خارج می‌شود:

```bash
openclaw models status --check
```

کاوش‌های زنده احراز هویت (برای محدود کردن دامنه، `--probe-provider`،‏ `--probe-profile`،‏ `--probe-timeout`،‏ `--probe-concurrency` یا `--probe-max-tokens` را اضافه کنید):

```bash
openclaw models status --probe
```

نکات:

- ردیف‌های کاوش می‌توانند از پروفایل‌های احراز هویت، اعتبارنامه‌های محیطی یا `models.json` بیایند.
- اگر `auth.order.<provider>` یک پروفایل ذخیره‌شده را حذف کند، کاوش به‌جای امتحان کردن آن، برای آن پروفایل `excluded_by_auth_order` را گزارش می‌کند.
- اگر احراز هویت موجود باشد اما OpenClaw نتواند مدلی قابل‌کاوش برای آن ارائه‌دهنده پیدا کند، کاوش `status: no_model` را گزارش می‌کند.
- دوره‌های انتظار محدودیت نرخ می‌توانند مختص مدل باشند: پروفایلی که برای یک مدل در دوره انتظار است، همچنان می‌تواند به مدل هم‌خانواده‌ای در همان ارائه‌دهنده سرویس دهد.

اسکریپت‌های عملیاتی اختیاری (systemd/Termux): [اسکریپت‌های پایش احراز هویت](/fa/help/scripts#auth-monitoring-scripts).

## چرخش کلید API ‏(Gateway)

برخی ارائه‌دهندگان وقتی فراخوانی با محدودیت نرخ ارائه‌دهنده مواجه می‌شود، درخواست را با یک کلید جایگزین پیکربندی‌شده دوباره امتحان می‌کنند.

ترتیب اولویت کلید برای هر ارائه‌دهنده:

1. `OPENCLAW_LIVE_<PROVIDER>_KEY` (بازنویسی تکی؛ یک کلید را ثابت می‌کند)
2. `<PROVIDER>_API_KEYS` (فهرست جداشده با ویرگول/فاصله/نقطه‌ویرگول)
3. `<PROVIDER>_API_KEY`
4. `<PROVIDER>_API_KEY_*` (هر متغیر محیطی با این پیشوند)

ارائه‌دهندگان Google ‏(`google`،‏ `google-vertex`) علاوه بر این، در صورت نیاز به `GOOGLE_API_KEY` بازمی‌گردند. فهرست ترکیبی پیش از استفاده از موارد تکراری پاک می‌شود.

OpenClaw فقط زمانی به کلید بعدی می‌چرخد که پیام خطا با یکی از این موارد مطابقت داشته باشد: `rate_limit`،‏ `rate limit`،‏ `429`،‏ `quota exceeded`/`quota_exceeded`،‏ `resource exhausted`/`resource_exhausted` یا `too many requests`. خطاهای دیگر با کلیدهای جایگزین دوباره امتحان نمی‌شوند. اگر همه کلیدها ناموفق باشند، خطای نهایی آخرین تلاش بازگردانده می‌شود.

<Note>
عبارت‌های مختص ارائه‌دهنده مانند `ThrottlingException`،‏ `concurrency limit reached` یا `workers_ai ... quota limit exceeded`، **طبقه‌بندی failover/تلاش مجدد** (تعویض مدل‌ها یا ارائه‌دهندگان پس از شکست مکرر) را هدایت می‌کنند که سازوکاری جدا از چرخش کلید API در بالا است.
</Note>

حذف احراز هویت ذخیره‌شده، کلید را نزد ارائه‌دهنده باطل نمی‌کند — هر زمان به بی‌اعتبارسازی سمت ارائه‌دهنده نیاز دارید، آن را در داشبورد ارائه‌دهنده بچرخانید یا باطل کنید.

## حذف احراز هویت ارائه‌دهنده هنگام اجرای Gateway

وقتی احراز هویت ارائه‌دهنده را از طریق صفحه کنترل Gateway حذف می‌کنید، OpenClaw پروفایل‌های احراز هویت ذخیره‌شده آن ارائه‌دهنده را حذف می‌کند و اجراهای فعال گفت‌وگو/عامل را که ارائه‌دهنده مدل انتخابی‌شان با مورد حذف‌شده مطابقت دارد، متوقف می‌کند. اجراهای متوقف‌شده رویدادهای عادی لغو/چرخه‌عمر را با `stopReason: "auth-revoked"` منتشر می‌کنند تا کلاینت‌های متصل بتوانند نشان دهند اجرا به‌دلیل حذف اعتبارنامه‌ها متوقف شده است.

## کنترل اعتبارنامه مورداستفاده

### OpenAI و شناسه‌های قدیمی `openai-codex`

پروفایل‌های کلید API متعلق به OpenAI و پروفایل‌های OAuth متعلق به ChatGPT/Codex هر دو از شناسه ارائه‌دهنده استاندارد `openai` استفاده می‌کنند. برای پیکربندی جدید از شناسه‌های پروفایل `openai:*` و `auth.order.openai` استفاده کنید.

اگر `openai-codex` را در پیکربندی قدیمی، شناسه‌های پروفایل احراز هویت یا `auth.order.openai-codex` مشاهده کردید، آن را ورودی مهاجرت قدیمی در نظر بگیرید — پروفایل‌های جدید `openai-codex` ایجاد نکنید. اجرا کنید:

```bash
openclaw doctor --fix
openclaw models auth list --provider openai
```

Doctor شناسه‌های پروفایل قدیمی `openai-codex:*` و ورودی‌های `auth.order.openai-codex` را به مسیر استاندارد `openai` بازنویسی می‌کند. برای مسیریابی مدل/زمان‌اجرای مختص OpenAI، به [OpenAI](/fa/providers/openai) مراجعه کنید.

### هنگام ورود (CLI)

```bash
openclaw models auth login --provider openai --profile-id openai:ritsuko
openclaw models auth login --provider openai --profile-id openai:lain
```

`--profile-id` ورودهای OAuth متعدد برای یک ارائه‌دهنده را درون یک عامل از هم جدا نگه می‌دارد.

`--force` پروفایل‌های احراز هویت ذخیره‌شده آن ارائه‌دهنده را در پوشه عامل انتخابی حذف می‌کند و سپس همان جریان احراز هویت را دوباره اجرا می‌کند. وقتی پروفایل ذخیره‌شده گیر کرده، منقضی شده یا به حساب اشتباهی متصل است، از آن استفاده کنید. این کار اعتبارنامه‌ها را نزد ارائه‌دهنده باطل نمی‌کند.

```bash
openclaw models auth login --provider anthropic --force
```

### برای هر نشست (فرمان گفت‌وگو)

- `/model <alias-or-id>@<profileId>` یک اعتبارنامه ارائه‌دهنده مشخص را برای نشست جاری ثابت می‌کند (نمونه شناسه‌های پروفایل: `anthropic:default`،‏ `anthropic:work`).
- `/model` (یا `/model list`) یک انتخاب‌گر فشرده نشان می‌دهد؛ `/model status` نمای کامل را نشان می‌دهد (گزینه‌ها + پروفایل احراز هویت بعدی، به‌همراه جزئیات نقطه پایانی ارائه‌دهنده در صورت پیکربندی).

اگر ترتیب احراز هویت یا ثابت‌سازی پروفایل را برای گفت‌وگویی که از قبل در حال اجرا است تغییر می‌دهید، `/new` یا `/reset` را ارسال کنید تا نشست تازه‌ای آغاز شود — نشست‌های موجود تا زمان بازنشانی، انتخاب مدل/پروفایل فعلی خود را حفظ می‌کنند.

### برای هر عامل (بازنویسی CLI)

بازنویسی‌های ترتیب احراز هویت در وضعیت احراز هویت SQLite همان عامل ذخیره می‌شوند:

```bash
openclaw models auth order get --provider anthropic
openclaw models auth order set --provider anthropic anthropic:default
openclaw models auth order clear --provider anthropic
```

برای هدف قرار دادن یک عامل مشخص از `--agent <id>` استفاده کنید؛ برای استفاده از عامل پیش‌فرض پیکربندی‌شده، آن را حذف کنید. `openclaw models status --probe` پروفایل‌های ذخیره‌شده حذف‌شده را به‌جای رد کردن بی‌سروصدای آن‌ها، به‌صورت `excluded_by_auth_order` نشان می‌دهد.

## عیب‌یابی

### «هیچ اعتبارنامه‌ای یافت نشد»

یک کلید API متعلق به Anthropic را روی **میزبان Gateway** پیکربندی کنید، یا مسیر setup-token متعلق به Anthropic را راه‌اندازی کنید، سپس دوباره بررسی کنید:

```bash
openclaw models status
```

### توکن در حال انقضا/منقضی‌شده

`openclaw models status` را اجرا کنید تا ببینید کدام پروفایل در حال انقضا است. اگر پروفایل توکن Anthropic موجود نیست یا منقضی شده است، آن را از طریق setup-token تازه‌سازی کنید یا به کلید API متعلق به Anthropic مهاجرت کنید.

## مرتبط

- [مدیریت اسرار](/fa/gateway/secrets)
- [دسترسی از راه دور](/fa/gateway/remote)
- [ذخیره‌سازی احراز هویت](/fa/concepts/oauth)
