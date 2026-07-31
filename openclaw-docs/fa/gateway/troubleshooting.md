---
read_when:
    - مرکز عیب‌یابی برای تشخیص عمیق‌تر شما را به اینجا هدایت کرده است
    - به بخش‌های پایدار راهنمای عملیاتی مبتنی بر نشانه‌ها با فرمان‌های دقیق نیاز دارید
sidebarTitle: Troubleshooting
summary: راهنمای جامع عیب‌یابی برای Gateway، کانال‌ها، خودکارسازی، Nodeها و مرورگر
title: عیب‌یابی
x-i18n:
    generated_at: "2026-07-27T14:10:15Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4bb1e061dbf2767118c24ad1ca2d2d1f7eeeff88e18ed0e6111aebe1cc99a26
    source_path: gateway/troubleshooting.md
    workflow: 16
---

این راهنمای عملیاتی عمیق است. برای جریان عیب‌یابی سریع، ابتدا از [/help/troubleshooting](/fa/help/troubleshooting) شروع کنید.

## نردبان فرمان‌ها

به این ترتیب اجرا کنید:

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
openclaw doctor
openclaw channels status --probe
```

نشانه‌های سلامت:

- `openclaw gateway status`، `Runtime: running`، `Connectivity probe: ok` و یک خط `Capability: ...` را نشان می‌دهد.
- `openclaw doctor` هیچ مشکل مسدودکننده‌ای در پیکربندی/سرویس گزارش نمی‌کند.
- `openclaw channels status --probe` وضعیت زنده انتقال را برای هر حساب و، در موارد پشتیبانی‌شده، `works` یا `audit ok` نشان می‌دهد.

## پس از به‌روزرسانی

زمانی استفاده کنید که به‌روزرسانی تمام شده است، اما Gateway از کار افتاده، کانال‌ها خالی‌اند یا فراخوانی‌های مدل با خطاهای 401 شکست می‌خورند.

```bash
openclaw status --all
openclaw update status --json
openclaw gateway status --deep
openclaw doctor --fix
openclaw gateway restart
```

موارد زیر را بررسی کنید:

- `Update restart` در `openclaw status` / `openclaw status --all`. واگذاری‌های در انتظار یا ناموفق شامل فرمان بعدی برای اجرا هستند.
- `plugin load failed: dependency tree corrupted; run openclaw doctor --fix` زیر Channels: پیکربندی کانال همچنان وجود دارد، اما ثبت Plugin پیش از بارگذاری کانال ناموفق بوده است.
- خطاهای 401 ارائه‌دهنده پس از احراز هویت مجدد: `openclaw doctor --fix` سایه‌های منسوخ احراز هویت OAuth مختص هر عامل را بررسی و نسخه‌های قدیمی را حذف می‌کند تا همه عامل‌ها نمایه مشترک فعلی را پیدا کنند.

## نصب‌های چندپاره و محافظ پیکربندی جدیدتر

زمانی استفاده کنید که سرویس Gateway پس از به‌روزرسانی به‌طور غیرمنتظره متوقف می‌شود، یا گزارش‌ها نشان می‌دهند یک فایل اجرایی `openclaw` از نسخه‌ای که آخرین بار `openclaw.json` را نوشته قدیمی‌تر است.

OpenClaw نوشتن پیکربندی را با `meta.lastTouchedVersion` مهرگذاری می‌کند. فرمان‌های فقط‌خواندنی می‌توانند پیکربندی نوشته‌شده توسط OpenClaw جدیدتر را بررسی کنند، اما تغییرات فرایند و سرویس از طریق فایل اجرایی قدیمی‌تر اجرا نمی‌شوند. اقدامات مسدودشده عبارت‌اند از: شروع/توقف/راه‌اندازی مجدد/حذف نصب سرویس Gateway، نصب مجدد اجباری سرویس، راه‌اندازی Gateway در حالت سرویس و پاک‌سازی پورت `gateway --force`.

```bash
which openclaw
openclaw --version
openclaw gateway status --deep
openclaw config get meta.lastTouchedVersion
```

<Steps>
  <Step title="اصلاح PATH">
    `PATH` را اصلاح کنید تا `openclaw` به نصب جدیدتر منتهی شود، سپس اقدام را دوباره اجرا کنید.
  </Step>
  <Step title="نصب مجدد سرویس Gateway">
    سرویس Gateway موردنظر را از نصب جدیدتر دوباره نصب کنید:

    ```bash
    openclaw gateway install --force
    openclaw gateway restart
    ```

  </Step>
  <Step title="حذف لفاف‌های منسوخ">
    ورودی‌های منسوخ بسته سیستمی یا لفاف‌های قدیمی را که همچنان به فایل اجرایی قدیمی `openclaw` اشاره می‌کنند حذف کنید.
  </Step>
</Steps>

<Warning>
فقط برای تنزل عمدی نسخه یا بازیابی اضطراری، `OPENCLAW_ALLOW_OLDER_BINARY_DESTRUCTIVE_ACTIONS=1` را برای همان یک فرمان تنظیم کنید. برای عملکرد عادی آن را تنظیم‌نشده نگه دارید.
</Warning>

## عدم تطابق پروتکل پس از بازگردانی

زمانی استفاده کنید که گزارش‌ها پس از تنزل نسخه یا بازگردانی همچنان `protocol mismatch` را چاپ می‌کنند. یک Gateway قدیمی‌تر در حال اجراست، اما یک فرایند سرویس‌گیرنده محلی جدیدتر همچنان با محدوده پروتکلی که Gateway قدیمی‌تر پشتیبانی نمی‌کند دوباره متصل می‌شود.

```bash
openclaw --version
which -a openclaw
openclaw gateway status --deep
openclaw doctor --deep
openclaw logs --follow
```

موارد زیر را بررسی کنید:

- `protocol mismatch ... client=... v<version> min=<n> max=<n> expected=<n>` در گزارش‌های Gateway.
- `Established clients:` در `openclaw gateway status --deep` یا `Gateway clients` در `openclaw doctor --deep`: سرویس‌گیرنده‌های فعال TCP متصل به پورت Gateway، همراه با PIDها و خطوط فرمان در صورت اجازه سیستم‌عامل.
- فرایند سرویس‌گیرنده‌ای که خط فرمان آن به نصب یا لفاف جدیدتر OpenClaw اشاره می‌کند که از آن بازگردانی کرده‌اید.

راه‌حل:

1. فرایند منسوخ سرویس‌گیرنده OpenClaw را که `gateway status --deep` نشان می‌دهد متوقف یا دوباره راه‌اندازی کنید.
2. برنامه‌ها یا لفاف‌هایی را که OpenClaw را در خود جای داده‌اند دوباره راه‌اندازی کنید: داشبوردهای محلی، ویرایشگرها، ابزارهای کمکی سرور برنامه یا پوسته‌های طولانی‌مدت `openclaw logs --follow`.
3. `openclaw gateway status --deep` یا `openclaw doctor --deep` را دوباره اجرا و تأیید کنید که PID سرویس‌گیرنده منسوخ حذف شده است.

یک Gateway قدیمی‌تر را وادار نکنید پروتکل جدیدتر و ناسازگار را بپذیرد. افزایش نسخه پروتکل از قرارداد ارتباطی محافظت می‌کند؛ بازیابی پس از بازگردانی، مسئله پاک‌سازی فرایند/نسخه است.

## رد شدن پیوند نمادین Skill به‌عنوان خروج از مسیر

زمانی استفاده کنید که گزارش‌ها شامل این مورد هستند:

```text
نادیده گرفتن مسیر Skill خارج‌شده از ریشه پیکربندی‌شده آن: ... reason=symlink-escape
```

هر ریشه Skill یک مرز مهار است. پیوند نمادینی زیر `~/.agents/skills`، `<workspace>/.agents/skills`، `<workspace>/skills` یا `~/.openclaw/skills` زمانی نادیده گرفته می‌شود که مقصد واقعی آن بیرون از آن ریشه قرار گیرد، مگر اینکه مقصد صراحتاً مورد اعتماد باشد.

پیوند را بررسی کنید:

```bash
ls -l ~/.agents/skills/<name>
realpath ~/.agents/skills/<name>
openclaw config get skills.load
```

اگر مقصد عمدی است، هم ریشه مستقیم Skill و هم مقصد مجاز پیوند نمادین را پیکربندی کنید:

```json5
{
  skills: {
    load: {
      extraDirs: ["~/Projects/manager/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
  },
}
```

سپس یک نشست جدید آغاز کنید یا منتظر تازه‌سازی ناظر Skills بمانید. اگر فرایند در حال اجرا پیش از تغییر پیکربندی آغاز شده است، Gateway را دوباره راه‌اندازی کنید.

از مقصدهای گسترده‌ای مانند `~`، `/` یا کل پوشه همگام‌شده پروژه استفاده نکنید. دامنه `allowSymlinkTargets` را به ریشه واقعی Skill که دایرکتوری‌های مورد اعتماد `SKILL.md` را در بر می‌گیرد محدود کنید.

اگر اعمال Skill Workshop باید از طریق آن مسیرهای مورد اعتماد و پیوندشده Skill در فضای کاری نیز بنویسد، `skills.workshop.allowSymlinkTargetWrites` را فعال کنید. برای ریشه‌های اشتراکی و فقط‌خواندنی Skill آن را غیرفعال نگه دارید.

مرتبط:

- [پیکربندی Skills](/fa/tools/skills-config#symlinked-skill-roots)
- [نمونه‌های پیکربندی](/fa/gateway/configuration-examples#symlinked-sibling-skill-repo)

## نیاز Anthropic 429 به مصرف اضافی برای زمینه طولانی

زمانی استفاده کنید که گزارش‌ها/خطاها شامل این مورد هستند: `HTTP 429: rate_limit_error: Extra usage is required for long context requests`.

```bash
openclaw logs --follow
openclaw models status
openclaw config get agents.defaults.models
```

موارد زیر را بررسی کنید:

- مدل انتخاب‌شده Anthropic یک مدل 1M Claude 4.x دارای قابلیت GA است (Opus 4.6/4.7/4.8، Sonnet 4.6)، یا پیکربندی مدل همچنان شامل `params.context1m: true` قدیمی است.
- اعتبارنامه فعلی Anthropic واجد شرایط استفاده از زمینه طولانی نیست.
- درخواست‌ها فقط در نشست‌های طولانی/اجرای مدل‌هایی که به مسیر زمینه 1M نیاز دارند شکست می‌خورند.

گزینه‌های رفع مشکل:

<Steps>
  <Step title="استفاده از پنجره زمینه استاندارد">
    به مدلی با پنجره استاندارد تغییر دهید، یا `context1m` قدیمی را از پیکربندی
    مدل قدیمی‌تری که قابلیت GA برای زمینه 1M ندارد حذف کنید.
  </Step>
  <Step title="استفاده از اعتبارنامه واجد شرایط">
    از اعتبارنامه Anthropic واجد شرایط درخواست‌های زمینه طولانی استفاده کنید، یا به کلید API Anthropic تغییر دهید.
  </Step>
  <Step title="پیکربندی مدل‌های جایگزین">
    مدل‌های جایگزین را پیکربندی کنید تا هنگام رد شدن درخواست‌های زمینه طولانی Anthropic، اجراها ادامه پیدا کنند.
  </Step>
</Steps>

مرتبط:

- [Anthropic](/fa/providers/anthropic)
- [مصرف توکن و هزینه‌ها](/fa/reference/token-use)
- [چرا خطای HTTP 429 از Anthropic مشاهده می‌کنم؟](/fa/help/faq-first-run#why-am-i-seeing-http-429-ratelimiterror-from-anthropic)

## پاسخ‌های مسدودشده 403 از بالادست

زمانی استفاده کنید که ارائه‌دهنده بالادستی LLM یک `403` عمومی مانند `Your request was blocked` برمی‌گرداند.

فرض نکنید که این مورد همیشه مشکل پیکربندی OpenClaw است. پاسخ می‌تواند از لایه امنیتی بالادستی مانند CDN، WAF، قاعده مدیریت ربات یا پراکسی معکوس در جلوی نقطه پایانی سازگار با OpenAI بیاید.

```bash
openclaw status
openclaw gateway status
openclaw logs --follow
```

موارد زیر را بررسی کنید:

- چند مدل زیر یک ارائه‌دهنده به شیوه یکسانی شکست می‌خورند.
- به‌جای خطای معمول API ارائه‌دهنده، HTML یا متن امنیتی عمومی نمایش داده می‌شود.
- رویدادهای امنیتی سمت ارائه‌دهنده برای همان زمان درخواست وجود دارند.
- یک کاوش مستقیم و کوچک `curl` موفق می‌شود، اما درخواست‌هایی با ساختار عادی SDK شکست می‌خورند.

وقتی شواهد به مسدودسازی WAF/CDN اشاره دارند، ابتدا فیلتر سمت ارائه‌دهنده را اصلاح کنید. یک قاعده مجازسازی یا رد شدن با دامنه محدود برای مسیر API مورد استفاده OpenClaw ترجیح دارد؛ از غیرفعال کردن حفاظت برای کل سایت خودداری کنید.

<Warning>
موفقیت حداقل درخواست `curl` تضمین نمی‌کند که درخواست‌های واقعی به سبک SDK از همان لایه امنیتی بالادستی عبور کنند.
</Warning>

مرتبط:

- [نقاط پایانی سازگار با OpenAI](/fa/gateway/configuration-reference#openai-compatible-endpoints)
- [پیکربندی ارائه‌دهنده](/fa/providers)
- [گزارش‌ها](/fa/logging)

## کاوش‌های مستقیم پشتیبان محلی سازگار با OpenAI موفق‌اند، اما اجرای عامل شکست می‌خورد

زمانی استفاده کنید که:

- `curl ... /v1/models` کار می‌کند.
- فراخوانی‌های کوچک و مستقیم `/v1/chat/completions` کار می‌کنند.
- اجرای مدل OpenClaw فقط در نوبت‌های عادی عامل شکست می‌خورد.

```bash
curl http://127.0.0.1:1234/v1/models
curl http://127.0.0.1:1234/v1/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"<id>","messages":[{"role":"user","content":"سلام"}],"stream":false}'
openclaw infer model run --model <provider/model> --prompt "سلام" --json
openclaw logs --follow
```

موارد زیر را بررسی کنید:

- فراخوانی‌های مستقیم کوچک موفق‌اند، اما اجرای OpenClaw فقط برای پرامپت‌های بزرگ‌تر شکست می‌خورد.
- خطاهای `model_not_found` یا 404 رخ می‌دهند، حتی با اینکه `/v1/chat/completions` مستقیم با همان شناسه ساده مدل کار می‌کند.
- خطاهای پشتیبان درباره اینکه `messages[].content` باید رشته باشد.
- هشدارهای متناوب `incomplete turn detected ... stopReason=stop payloads=0` با یک پشتیبان محلی سازگار با OpenAI.
- خرابی‌های پشتیبان که فقط با تعداد بیشتر توکن‌های پرامپت یا پرامپت‌های کامل زمان‌اجرای عامل رخ می‌دهند.

<AccordionGroup>
  <Accordion title="نشانه‌های رایج">
    - `model_not_found` با سرور محلی به سبک MLX/vLLM: تأیید کنید `baseUrl` شامل `/v1` است، `api` برای پشتیبان‌های `/v1/chat/completions` برابر `"openai-completions"` است و `models.providers.<provider>.models[].id` شناسه ساده محلی ارائه‌دهنده است. آن را یک بار با پیشوند ارائه‌دهنده انتخاب کنید، برای مثال `mlx/mlx-community/Qwen3-30B-A3B-6bit`؛ ورودی کاتالوگ را به‌صورت `mlx-community/Qwen3-30B-A3B-6bit` نگه دارید.
    - `messages[...].content: invalid type: sequence, expected a string`: پشتیبان بخش‌های ساختاریافته محتوای Chat Completions را رد می‌کند. راه‌حل: `models.providers.<provider>.models[].compat.requiresStringContent: true` را تنظیم کنید.
    - `validation.keys` یا کلیدهای مجاز پیام مانند `["role","content"]`: پشتیبان فراداده بازپخش به سبک OpenAI را در پیام‌های Chat Completions رد می‌کند. راه‌حل: `models.providers.<provider>.models[].compat.strictMessageKeys: true` را تنظیم کنید.
    - `incomplete turn detected ... stopReason=stop payloads=0`: پشتیبان درخواست Chat Completions را تکمیل کرده، اما برای آن نوبت هیچ متن قابل‌مشاهده‌ای از دستیار برنگردانده است. OpenClaw نوبت‌های خالی سازگار با OpenAI و ایمن برای بازپخش را یک بار دوباره امتحان می‌کند؛ شکست‌های مداوم معمولاً به این معنا هستند که پشتیبان محتوای خالی/غیرمتنی تولید می‌کند یا متن پاسخ نهایی را سرکوب می‌کند.
    - درخواست‌های مستقیم کوچک موفق‌اند، اما اجرای عامل OpenClaw با خرابی پشتیبان/مدل شکست می‌خورد (برای مثال Gemma روی برخی ساخت‌های `inferrs`): انتقال OpenClaw احتمالاً از پیش درست است؛ پشتیبان در ساختار بزرگ‌تر پرامپت زمان‌اجرای عامل شکست می‌خورد.
    - شکست‌ها پس از غیرفعال کردن ابزارها کاهش می‌یابند، اما ناپدید نمی‌شوند: طرح‌واره‌های ابزار بخشی از فشار بوده‌اند، اما مشکل باقی‌مانده همچنان ظرفیت مدل/سرور بالادستی یا باگ پشتیبان است.

  </Accordion>
  <Accordion title="گزینه‌های رفع مشکل">
    1. برای پشتیبان‌های Chat Completions فقط‌رشته‌ای، `compat.requiresStringContent: true` را تنظیم کنید.
    2. برای پشتیبان‌های سخت‌گیر Chat Completions که در هر پیام فقط `role` و `content` را می‌پذیرند، `compat.strictMessageKeys: true` را تنظیم کنید.
    3. برای مدل‌ها/پشتیبان‌هایی که نمی‌توانند سطح طرح‌واره ابزار OpenClaw را به‌طور قابل‌اعتماد مدیریت کنند، `compat.supportsTools: false` را تنظیم کنید.
    4. در صورت امکان فشار پرامپت را کاهش دهید: راه‌اندازی اولیه کوچک‌تر فضای کاری، تاریخچه کوتاه‌تر نشست، مدل محلی سبک‌تر یا پشتیبانی با توانایی بهتر در زمینه طولانی.
    5. اگر درخواست‌های مستقیم کوچک همچنان موفق‌اند، اما نوبت‌های عامل OpenClaw هنوز درون پشتیبان خراب می‌شوند، آن را محدودیت سرور/مدل بالادستی در نظر بگیرید و یک نمونه بازتولید با ساختار محموله پذیرفته‌شده در همان‌جا ثبت کنید.
  </Accordion>
</AccordionGroup>

مرتبط:

- [پیکربندی](/fa/gateway/configuration)
- [مدل‌های محلی](/fa/gateway/local-models)
- [نقاط پایانی سازگار با OpenAI](/fa/gateway/configuration-reference#openai-compatible-endpoints)

## بدون پاسخ

اگر کانال‌ها فعال‌اند اما پاسخی دریافت نمی‌شود، پیش از اتصال مجدد هر چیزی، مسیریابی و خط‌مشی را بررسی کنید.

```bash
openclaw status
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw config get channels
openclaw logs --follow
```

موارد زیر را جست‌وجو کنید:

- در انتظار جفت‌سازی برای فرستندگان پیام خصوصی.
- محدودسازی بر اساس اشاره در گروه (`requireMention`، `mentionPatterns`).
- ناهماهنگی فهرست مجاز کانال/گروه.

نشانه‌های رایج:

- `drop guild message (mention required` ← پیام گروه تا زمان اشاره نادیده گرفته می‌شود.
- `pairing request` ← فرستنده به تأیید نیاز دارد.
- `blocked` / `allowlist` ← فرستنده/کانال توسط خط‌مشی فیلتر شده است.

مطالب مرتبط:

- [عیب‌یابی کانال](/fa/channels/troubleshooting)
- [گروه‌ها](/fa/channels/groups)
- [جفت‌سازی](/fa/channels/pairing)

## اتصال رابط کاربری کنترل داشبورد

وقتی داشبورد/رابط کاربری کنترل متصل نمی‌شود، URL، حالت احراز هویت و فرضیات مربوط به بستر امن را اعتبارسنجی کنید.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --json
```

موارد زیر را جست‌وجو کنید:

- درستی URL کاوش و URL داشبورد.
- ناهماهنگی حالت احراز هویت/توکن میان کلاینت و Gateway.
- استفاده از HTTP در جایی که هویت دستگاه الزامی است.

اگر مرورگر محلی پس از به‌روزرسانی نمی‌تواند به `127.0.0.1:18789` متصل شود، ابتدا سرویس محلی Gateway را بازیابی کنید و مطمئن شوید که داشبورد را ارائه می‌دهد:

```bash
openclaw gateway restart
lsof -i :18789
curl http://127.0.0.1:18789
```

اگر `curl` کد HTML مربوط به OpenClaw را برمی‌گرداند، Gateway کار می‌کند و مشکل باقی‌مانده احتمالاً حافظهٔ نهان مرورگر، یک پیوند عمیق قدیمی یا وضعیت منقضی‌شدهٔ زبانه است. `http://127.0.0.1:18789` را مستقیماً باز کنید و از داشبورد پیمایش کنید. اگر پس از راه‌اندازی مجدد سرویس در حال اجرا نمی‌ماند، `openclaw gateway start` را اجرا و `openclaw gateway status` را دوباره بررسی کنید.

<AccordionGroup>
  <Accordion title="نشانه‌های اتصال / احراز هویت">
    - `device identity required` ← بستر ناامن یا نبود احراز هویت دستگاه.
    - `origin not allowed` ← `Origin` مرورگر در `gateway.controlUi.allowedOrigins` نیست (یا از یک مبدأ مرورگر غیر-loopback بدون فهرست مجاز صریح متصل می‌شوید).
    - `device nonce required` / `device nonce mismatch` ← کلاینت جریان احراز هویت دستگاه مبتنی بر چالش را تکمیل نمی‌کند (`connect.challenge` + `device.nonce`).
    - `device signature invalid` / `device signature expired` ← کلاینت محتوای اشتباه (یا برچسب زمانی منقضی‌شده) را برای دست‌دهی فعلی امضا کرده است.
    - `AUTH_TOKEN_MISMATCH` همراه با `canRetryWithDeviceToken=true` ← کلاینت می‌تواند یک بار با توکن دستگاه ذخیره‌شده در حافظهٔ نهان، تلاش مجدد مورداعتماد انجام دهد.
    - آن تلاش مجدد با توکن ذخیره‌شده در حافظهٔ نهان، مجموعهٔ دامنه‌های ذخیره‌شده همراه با توکن دستگاه جفت‌شده را دوباره استفاده می‌کند. فراخوان‌های صریح `deviceToken` / صریح `scopes` در عوض مجموعهٔ دامنه‌های درخواستی خود را حفظ می‌کنند.
    - `AUTH_SCOPE_MISMATCH` ← توکن دستگاه شناسایی شده است، اما دامنه‌های تأییدشدهٔ آن این درخواست اتصال را پوشش نمی‌دهند؛ به‌جای تعویض توکن مشترک Gateway، دستگاه را دوباره جفت‌سازی یا قرارداد دامنهٔ درخواستی را تأیید کنید.
    - خارج از آن مسیر تلاش مجدد، اولویت احراز هویت اتصال به‌ترتیب عبارت است از توکن مشترک/گذرواژهٔ صریح، سپس `deviceToken` صریح، سپس توکن دستگاه ذخیره‌شده و در پایان توکن راه‌اندازی اولیه.
    - در مسیر ناهمگام رابط کاربری کنترل Tailscale Serve، تلاش‌های ناموفق برای `{scope, ip}` یکسان پیش از ثبت شکست توسط محدودکننده به‌صورت ترتیبی اجرا می‌شوند. بنابراین دو تلاش مجدد نامعتبر و هم‌زمان از یک کلاینت ممکن است در تلاش دوم به‌جای دو عدم تطابق ساده، `retry later` را نشان دهند.
    - `too many failed authentication attempts (retry later)` از یک کلاینت loopback با مبدأ مرورگر ← شکست‌های تکراری از همان `Origin` نرمال‌شده، موقتاً مسدود می‌شوند؛ یک مبدأ localhost دیگر از سهمیه‌ای جداگانه استفاده می‌کند.
    - تکرار `unauthorized` پس از آن تلاش مجدد ← ناهماهنگی توکن مشترک/توکن دستگاه؛ پیکربندی توکن را تازه‌سازی کنید و در صورت نیاز توکن دستگاه را دوباره تأیید یا تعویض کنید.
    - `gateway connect failed:` ← مقصد میزبان/درگاه/URL اشتباه است.

  </Accordion>
</AccordionGroup>

### نگاشت سریع کدهای جزئیات احراز هویت

برای انتخاب اقدام بعدی، از `error.details.code` موجود در پاسخ ناموفق `connect` استفاده کنید:

| کد جزئیات                  | معنی                                                                                                                                                                                      | اقدام پیشنهادی                                                                                                                                                                                                                                                                       |
| ---------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_TOKEN_MISSING`         | کلاینت توکن مشترک الزامی را ارسال نکرده است.                                                                                                                                                 | توکن را در کلاینت جای‌گذاری/تنظیم و دوباره تلاش کنید. برای مسیرهای داشبورد: `openclaw config get gateway.auth.token`، سپس آن را در تنظیمات رابط کاربری کنترل جای‌گذاری کنید.                                                                                                                                              |
| `AUTH_TOKEN_MISMATCH`        | توکن مشترک با توکن احراز هویت Gateway مطابقت نداشت.                                                                                                                                               | اگر `canRetryWithDeviceToken=true` است، یک تلاش مجدد مورداعتماد را مجاز کنید. تلاش‌های مجدد با توکن ذخیره‌شده، دامنه‌های تأییدشدهٔ ذخیره‌شده را دوباره استفاده می‌کنند؛ فراخوان‌های صریح `deviceToken` / `scopes` دامنه‌های درخواستی را حفظ می‌کنند. اگر همچنان ناموفق بود، [چک‌لیست بازیابی ناهماهنگی توکن](/fa/cli/devices#token-drift-recovery-checklist) را اجرا کنید. |
| `AUTH_DEVICE_TOKEN_MISMATCH` | توکن ذخیره‌شدهٔ هر دستگاه منقضی یا لغو شده است.                                                                                                                                                 | با استفاده از [CLI دستگاه‌ها](/fa/cli/devices)، توکن دستگاه را تعویض/دوباره تأیید کنید، سپس دوباره متصل شوید.                                                                                                                                                                                                        |
| `AUTH_SCOPE_MISMATCH`        | توکن دستگاه معتبر است، اما نقش/دامنه‌های تأییدشدهٔ آن این درخواست اتصال را پوشش نمی‌دهند.                                                                                                       | دستگاه را دوباره جفت‌سازی یا قرارداد دامنهٔ درخواستی را تأیید کنید؛ این مورد را ناهماهنگی توکن مشترک تلقی نکنید.                                                                                                                                                                                     |
| `PAIRING_REQUIRED`           | هویت دستگاه به تأیید نیاز دارد. `error.details.reason` را برای `not-paired`، `scope-upgrade`، `role-upgrade` یا `metadata-upgrade` بررسی کنید و در صورت وجود از `requestId` / `remediationHint` استفاده کنید. | درخواست در انتظار را تأیید کنید: `openclaw devices list` و سپس `openclaw devices approve <requestId>`. ارتقای دامنه/نقش پس از بازبینی دسترسی درخواستی، از همین جریان استفاده می‌کند.                                                                                                               |

<Note>
فراخوانی‌های مستقیم RPC بک‌اند loopback که با توکن مشترک/گذرواژهٔ Gateway احراز هویت شده‌اند، نباید به خط پایهٔ دامنه‌های دستگاه جفت‌شدهٔ CLI وابسته باشند. اگر زیرعامل‌ها یا دیگر فراخوانی‌های داخلی همچنان با `scope-upgrade` ناموفق‌اند، بررسی کنید که فراخوان از `client.id: "gateway-client"` و `client.mode: "backend"` استفاده می‌کند و یک `deviceIdentity` صریح یا توکن دستگاه را تحمیل نمی‌کند.
</Note>

بررسی مهاجرت احراز هویت دستگاه v2:

```bash
openclaw --version
openclaw doctor
openclaw gateway status
```

اگر گزارش‌ها خطاهای nonce/امضا را نشان می‌دهند، کلاینت متصل‌شونده را به‌روزرسانی و آن را بررسی کنید:

<Steps>
  <Step title="منتظر connect.challenge بمانید">
    کلاینت منتظر `connect.challenge` صادرشده توسط Gateway می‌ماند.
  </Step>
  <Step title="محتوا را امضا کنید">
    کلاینت محتوای مقید به چالش را امضا می‌کند.
  </Step>
  <Step title="nonce دستگاه را ارسال کنید">
    کلاینت `connect.params.device.nonce` را با همان nonce چالش ارسال می‌کند.
  </Step>
</Steps>

اگر `openclaw devices rotate` / `revoke` / `remove` به‌طور غیرمنتظره رد شد:

- نشست‌های توکن دستگاه جفت‌شده فقط می‌توانند دستگاه **خودشان** را مدیریت کنند، مگر اینکه فراخوان همچنین `operator.admin` را داشته باشد.
- `openclaw devices rotate --scope ...` فقط می‌تواند دامنه‌های اپراتوری را درخواست کند که نشست فراخوان از قبل در اختیار دارد.

مطالب مرتبط:

- [پیکربندی](/fa/gateway/configuration) (حالت‌های احراز هویت Gateway)
- [رابط کاربری کنترل](/fa/web/control-ui)
- [دستگاه‌ها](/fa/cli/devices)
- [دسترسی از راه دور](/fa/gateway/remote)
- [احراز هویت پراکسی مورداعتماد](/fa/gateway/trusted-proxy-auth)

## سرویس Gateway در حال اجرا نیست

زمانی استفاده کنید که سرویس نصب شده است، اما فرایند فعال نمی‌ماند.

```bash
openclaw gateway status
openclaw status
openclaw logs --follow
openclaw doctor
openclaw gateway status --deep   # سرویس‌های سطح سیستم را نیز اسکن کنید
```

موارد زیر را جست‌وجو کنید:

- `Runtime: stopped` همراه با سرنخ‌های خروج.
- ناهماهنگی پیکربندی سرویس (`Config (cli)` در برابر `Config (service)`).
- تداخل درگاه/شنونده.
- نصب‌های اضافی launchd/systemd/schtasks هنگام استفاده از `--deep`.
- راهنمای پاک‌سازی `Other gateway-like services detected (best effort)`.

<AccordionGroup>
  <Accordion title="نشانه‌های رایج">
    - `Gateway start blocked: set gateway.mode=local` یا `existing config is missing gateway.mode` ← حالت Gateway محلی فعال نیست، یا فایل پیکربندی بازنویسی شده و `gateway.mode` را از دست داده است. راه‌حل: `gateway.mode="local"` را در پیکربندی خود تنظیم کنید، یا `openclaw onboard --mode local` / `openclaw setup` را دوباره اجرا کنید تا پیکربندی مورد انتظار حالت محلی مجدداً ثبت شود. اگر OpenClaw را از طریق Podman اجرا می‌کنید، مسیر پیش‌فرض پیکربندی `~/.openclaw/openclaw.json` است.
    - `refusing to bind gateway ... without auth` ← اتصال غیر-loopback بدون مسیر معتبر احراز هویت Gateway (توکن/گذرواژه، یا trusted-proxy در صورت پیکربندی).
    - `another gateway instance is already listening` / `EADDRINUSE` ← تداخل درگاه.
    - `Other gateway-like services detected (best effort)` ← واحدهای منقضی یا موازی launchd/systemd/schtasks وجود دارند. در بیشتر راه‌اندازی‌ها باید در هر دستگاه یک Gateway نگه داشته شود؛ اگر به بیش از یکی نیاز دارید، درگاه‌ها + پیکربندی/وضعیت/فضای کاری را مجزا کنید. به [/gateway#multiple-gateways-same-host](/fa/gateway#multiple-gateways-same-host) مراجعه کنید.
    - `System-level OpenClaw gateway service detected` از doctor ← یک واحد سیستمی systemd وجود دارد، درحالی‌که سرویس سطح کاربر موجود نیست. پیش از اینکه به doctor اجازه دهید سرویس کاربر را نصب کند، مورد تکراری را حذف یا غیرفعال کنید؛ یا اگر واحد سیستمی سرپرست موردنظر است، `OPENCLAW_SERVICE_REPAIR_POLICY=external` را تنظیم کنید.
    - `Gateway service port does not match current gateway config` ← سرپرست نصب‌شده همچنان `--port` قدیمی را ثابت نگه داشته است. `openclaw doctor --fix` یا `openclaw gateway install --force` را اجرا کنید، سپس سرویس Gateway را راه‌اندازی مجدد کنید.

  </Accordion>
</AccordionGroup>

مطالب مرتبط:

- [اجرای پس‌زمینه و ابزار فرایند](/fa/gateway/background-process)
- [پیکربندی](/fa/gateway/configuration)
- [Doctor](/fa/gateway/doctor)

## Gateway در macOS بی‌صدا از پاسخ‌گویی بازمی‌ایستد و با لمس داشبورد دوباره ادامه می‌دهد

برای زمانی استفاده کنید که کانال‌ها (Telegram، WhatsApp و غیره) روی یک میزبان macOS هر بار از چند دقیقه تا چند ساعت بی‌صدا می‌شوند و به‌نظر می‌رسد Gateway درست در لحظه‌ای که Control UI را باز می‌کنید، از طریق SSH متصل می‌شوید یا به‌شکل دیگری با میزبان تعامل می‌کنید، دوباره فعال می‌شود. معمولاً در `openclaw status` نشانه آشکاری وجود ندارد، زیرا تا زمانی که آن را بررسی کنید Gateway دوباره فعال شده است.

```bash
ls ~/.openclaw/logs/stability/ | tail -5
openclaw gateway stability --bundle latest
pmset -g log | grep -iE "sleep|wake|maintenance" | tail -50
launchctl print gui/$UID/ai.openclaw.gateway | grep -E "state|last exit|runs"
```

به‌دنبال موارد زیر باشید:

- یک یا چند بسته `*-uncaught_exception.json` در `~/.openclaw/logs/stability/` که در آن‌ها `error.code` روی یک کد گذرای شبکه مانند `ENETDOWN`، `ENETUNREACH`، `EHOSTUNREACH` یا `ECONNREFUSED` تنظیم شده است.
- خطوط `pmset -g log` مانند `Entering Sleep state due to 'Maintenance Sleep'` یا `en0 driver is slow (msg: WillChangeState to 0)` که با مُهرهای زمانی خرابی هم‌زمان هستند. Power Nap / Maintenance Sleep درایور Wi-Fi را برای مدت کوتاهی وارد وضعیت 0 می‌کند؛ هر `connect()` خروجی که در این بازه رخ دهد، ممکن است با `ENETDOWN` ناموفق شود، حتی روی میزبانی که در حالت عادی اتصال کامل شبکه دارد.
- خروجی `launchctl print` که `state = not running` را همراه با چندین `runs` اخیر و یک کد خروج نشان می‌دهد، به‌ویژه زمانی که فاصله میان خرابی و اجرای بعدی حدود یک ساعت است، نه چند ثانیه. launchd در macOS پس از وقوع متوالی چند خرابی، یک سازوکار حفاظتی مستندنشدۀ اجرای مجدد اعمال می‌کند که ممکن است رعایت `KeepAlive=true` را متوقف کند تا زمانی که یک محرک خارجی مانند ورود تعاملی، اتصال داشبورد یا `launchctl kickstart` آن را دوباره فعال کند.

نشانه‌های رایج:

- یک بسته پایداری که `error.code` آن `ENETDOWN` یا کدی هم‌خانواده است و پشته فراخوانی به `net` در Node، یعنی `lookupAndConnect` / `Socket.connect` اشاره می‌کند. OpenClaw `2026.5.26` و نسخه‌های جدیدتر این موارد را خطاهای گذرای بی‌خطر شبکه طبقه‌بندی می‌کنند تا دیگر به کنترل‌گر سطح‌بالای استثناهای مدیریت‌نشده منتقل نشوند؛ اگر از نسخه‌ای قدیمی‌تر استفاده می‌کنید، ابتدا ارتقا دهید.
- دوره‌های طولانی سکوت که درست در لحظه اتصال به Control UI یا ورود از طریق SSH به میزبان پایان می‌یابند: این فعالیت قابل‌مشاهده برای کاربر است که سازوکار اجرای مجدد launchd را دوباره فعال می‌کند، نه کاری که داشبورد با Gateway انجام می‌دهد.
- افزایش شمار `runs` در طول روز بدون وجود خط متناظر `received SIG*; shutting down` در `~/Library/Logs/openclaw/gateway.log`: خاموش‌شدن‌های سالم یک سیگنال ثبت می‌کنند؛ خرابی‌های گذرا چنین نمی‌کنند.

اقدامات لازم:

1. اگر نسخه‌ای پیش از `2026.5.26` را اجرا می‌کنید، **Gateway را ارتقا دهید**. پس از ارتقا، خطاهای آینده `ENETDOWN` به‌جای خاتمه‌دادن به فرایند، به‌عنوان هشدار ثبت می‌شوند.
2. در میزبان‌های Mac mini / دسکتاپی که قرار است به‌عنوان سرورهای همیشه‌روشن کار کنند، **فعالیت خواب نگه‌داری را کاهش دهید**:

   ```bash
   sudo pmset -a sleep 0 disksleep 0 standby 0 powernap 0
   ```

   این کار ناپایداری زیربنایی درایور را به‌شکل چشمگیری کاهش می‌دهد، اما آن را کاملاً از بین نمی‌برد. سیستم ممکن است صرف‌نظر از این پرچم‌ها همچنان برخی خواب‌های نگه‌داری را برای حفظ TCP keepalive و mDNS انجام دهد.

3. یک **ناظر زنده‌بودن اضافه کنید** تا اگر در آینده launchd پس از وقوع متوالی خرابی‌ها فرایند را متوقف نگه داشت، مشکل به‌سرعت شناسایی شود:

   ```bash
   # نمونه بررسی زنده‌بودن آگاه از launchd، مناسب برای Cron پنج‌دقیقه‌ای یا LaunchAgent
   state=$(launchctl print gui/$UID/ai.openclaw.gateway 2>/dev/null | awk -F'= ' '/state =/ {print $2; exit}')
   if [ "$state" != "running" ]; then
     launchctl kickstart -k gui/$UID/ai.openclaw.gateway
   fi
   ```

   هدف، فعال‌کردن دوباره سازوکار اجرای مجدد از بیرون است؛ پس از وقوع متوالی خرابی‌ها در macOS، `KeepAlive=true` به‌تنهایی کافی نیست.

مرتبط:

- [نکات پلتفرم macOS](/fa/platforms/macos)
- [ثبت گزارش](/fa/logging)
- [Doctor](/fa/gateway/doctor)

## حلقه نظارتی launchd در macOS با LaunchAgentهای تکراری Gateway/node

این بخش را زمانی استفاده کنید که یک نصب macOS هر چند ثانیه یک‌بار پیوسته راه‌اندازی مجدد می‌شود، بررسی‌های سلامت `openclaw`
میان وضعیت سالم و دردسترس‌نبودن نوسان می‌کنند و ارسال کانال متوقف می‌شود،
با اینکه به‌نظر می‌رسد سرویس در حال اجرا است.

این وضعیت در نصب‌های قدیمی‌تری مشاهده شده است که در آن‌ها هر دو LaunchAgent
یعنی `ai.openclaw.gateway` و `ai.openclaw.node` فعال بودند و هرکدام
`OPENCLAW_LAUNCHD_LABEL` را تزریق می‌کردند. در این حالت OpenClaw می‌تواند نظارت
launchd را تشخیص دهد، تلاش کند کنترل راه‌اندازی مجدد را به launchd بازگرداند و به‌جای
یک فرایند پایدار Gateway، وارد حلقه سریع `EADDRINUSE`/اجرای مجدد شود.

```bash
for i in 1 2 3 4; do
  ps aux | grep 'openclaw.*index.js' | grep -v grep | awk '{print $2}'
  sleep 10
done

openclaw gateway status --deep
openclaw node status
launchctl print gui/$UID/ai.openclaw.gateway | grep -E 'state|last exit|runs'
tail -n 80 ~/Library/Logs/openclaw/gateway.log
```

به‌دنبال موارد زیر باشید:

- وجود بیش از یک PID برای Gateway در نمونه 30ثانیه‌ای، به‌جای یک
  فرایند پایدار.
- `EADDRINUSE`، `another gateway instance is already listening` یا خطوط تکراری
  راه‌اندازی مجدد/واگذاری کنترل در `gateway.log`.
- بارگذاری هم‌زمان هر دو `~/Library/LaunchAgents/ai.openclaw.gateway.plist` و
  `~/Library/LaunchAgents/ai.openclaw.node.plist` روی میزبانی که باید فقط یک سرویس مدیریت‌شده
  Gateway را اجرا کند.

اقدامات لازم:

1. اگر این میزبان باید فقط سرویس Gateway را اجرا کند، سرویس مدیریت‌شده node را
   از طریق OpenClaw حذف کنید. اگر برای قابلیت‌های node راه‌دور فعالانه به سرویس node
   متکی هستید، **این مرحله را رد کنید**؛ حذف آن این قابلیت‌ها را روی
   این میزبان متوقف می‌کند:

   ```bash
   openclaw node uninstall
   ```

2. یک پوشش‌دهنده پایدار Gateway نصب کنید که پیش از اجرای OpenClaw، نشانگرهای
   به‌ارث‌رسیده launchd را پاک کند. از گزینه پشتیبانی‌شده `--wrapper` استفاده کنید؛
   فایل تولیدشده در `~/.openclaw/service-env/` را ویرایش نکنید، زیرا نصب مجدد سرویس،
   به‌روزرسانی و تعمیر Doctor آن فایل را دوباره تولید می‌کنند:

   ```bash
   mkdir -p ~/.local/bin
   cat >~/.local/bin/openclaw-launchd-workaround <<'EOF'
   #!/bin/sh
   set -eu
   unset OPENCLAW_LAUNCHD_LABEL LAUNCH_JOB_LABEL LAUNCH_JOB_NAME XPC_SERVICE_NAME || true
   exec openclaw "$@"
   EOF
   chmod 700 ~/.local/bin/openclaw-launchd-workaround

   openclaw gateway install \
     --wrapper ~/.local/bin/openclaw-launchd-workaround \
     --force
   ```

   `gateway install` مسیر پوشش را در نصب‌های مجدد اجباری،
   به‌روزرسانی‌ها و تعمیرات doctor حفظ می‌کند.

3. بررسی کنید که Gateway پایدار است و RPC ارائه می‌دهد، نه اینکه صرفاً در حال گوش‌دادن باشد:

   ```bash
   openclaw gateway status --deep --require-rpc

   for i in 1 2 3 4; do
     ps aux | grep 'openclaw.*index.js' | grep -v grep | awk '{print $2}'
     sleep 10
   done
   ```

   نمونه PID باید به‌جای مجموعه‌ای از PIDهای در حال تغییر، یک فرایند پایدار را نشان دهد
   و توزیع کانال ورودی باید از سر گرفته شود.

4. پس از ارتقا به نسخه‌ای که حلقه زیربنایی دوگانه LaunchAgent در آن
   رفع شده است، راه‌حل موقت را حذف و سرویس مدیریت‌شده عادی را دوباره نصب کنید:

   ```bash
   OPENCLAW_WRAPPER= openclaw gateway install --force
   rm ~/.local/bin/openclaw-launchd-workaround
   ```

مرتبط:

- [نکات پلتفرم macOS](/fa/platforms/mac/bundled-gateway)
- [Doctor](/fa/gateway/doctor)
- [CLI مربوط به Gateway](/fa/cli/gateway)

## خروج Gateway هنگام مصرف زیاد حافظه

زمانی استفاده کنید که Gateway زیر بار ناپدید می‌شود، ناظر راه‌اندازی مجددی شبیه OOM گزارش می‌کند، یا گزارش‌ها به `critical memory pressure bundle written` اشاره دارند.

```bash
openclaw gateway status --deep
openclaw logs --follow
openclaw gateway stability --bundle latest
openclaw gateway diagnostics export
```

به‌دنبال این موارد بگردید:

- `Reason: diagnostic.memory.pressure.critical` در جدیدترین بسته پایداری.
- `Memory pressure:` همراه با `critical/rss_threshold`، `critical/heap_threshold`، یا `critical/rss_growth`.
- مقادیر `V8 heap:` نزدیک به محدودیت heap.
- ورودی‌های `Largest session files:` مانند `agents/<agent>/sessions/<session>.jsonl` یا `sessions/<session>.jsonl`.
- شمارنده‌های حافظه cgroup در Linux، هنگامی که Gateway درون کانتینر یا سرویسی با حافظه محدود اجرا می‌شود.

نشانه‌های رایج:

- `critical memory pressure bundle written` اندکی پیش از راه‌اندازی مجدد ظاهر می‌شود ← OpenClaw یک بسته پایداری پیش از OOM ثبت کرده است. آن را با `openclaw gateway stability --bundle latest` بررسی کنید.
- `memory pressure: level=critical` در گزارش‌های Gateway ظاهر می‌شود ← OpenClaw فشار بحرانی حافظه را تشخیص داده و داده‌های حافظه درون‌فرایندی موجود را ثبت کرده است.
- `Largest session files:` به مسیر رونوشت پالایش‌شده بسیار بزرگی اشاره می‌کند ← تاریخچه نشست نگه‌داری‌شده را کاهش دهید، رشد نشست را بررسی کنید، یا پیش از راه‌اندازی مجدد، رونوشت‌های قدیمی را از مخزن فعال خارج کنید.
- بایت‌های مصرف‌شده `V8 heap:` به محدودیت heap نزدیک‌اند ← ابتدا فشار پرامپت/نشست یا حجم کار هم‌زمان را کاهش دهید. برای سرویس مدیریت‌شده، `Gateway heap:` را در `openclaw gateway status` بررسی کنید؛ اگر مقدار آن `not set` است، فراداده قدیمی سرویس را با `openclaw gateway install --force` دوباره تولید کنید. متغیر `NODE_OPTIONS` در پوسته محیطی عمداً نادیده گرفته می‌شود. فقط پس از تأیید حجم کار پایدار و باقی‌گذاشتن فضای کافی برای حافظه بومی، از بازنویسی صریح heap در سطح ناظر استفاده کنید.
- `Memory pressure: critical/rss_growth` ← حافظه درون یک بازه نمونه‌برداری به‌سرعت رشد کرده است. جدیدترین گزارش‌ها را برای واردسازی بزرگ، خروجی مهارنشده ابزار، تلاش‌های مجدد تکراری، یا دسته‌ای از کارهای صف‌شده عامل بررسی کنید.
- فشار بحرانی حافظه در گزارش‌ها ظاهر می‌شود، اما بسته‌ای وجود ندارد ← پس از رویداد، `openclaw gateway diagnostics export` را برای شواهد عملیاتی موجود ثبت کنید.

بسته پایداری فاقد payload است. این بسته شامل شواهد عملیاتی حافظه و مسیرهای نسبی پالایش‌شده فایل است، نه متن پیام، بدنه‌های Webhook، اطلاعات اعتبارسنجی، توکن‌ها، کوکی‌ها یا شناسه‌های خام نشست. به‌جای کپی‌کردن گزارش‌های خام، خروجی عیب‌یابی را به گزارش‌های اشکال پیوست کنید.

مرتبط:

- [سلامت Gateway](/fa/gateway/health)
- [خروجی عیب‌یابی](/fa/gateway/diagnostics)
- [نشست‌ها](/fa/cli/sessions)

## Gateway پیکربندی نامعتبر را رد کرد

زمانی استفاده کنید که راه‌اندازی Gateway با `Invalid config` شکست می‌خورد یا گزارش‌های بارگذاری مجدد فوری اعلام می‌کنند که ویرایش نامعتبری نادیده گرفته شده است.

```bash
openclaw logs --follow
openclaw config file
openclaw config validate
openclaw doctor
```

به‌دنبال این موارد بگردید:

- `Invalid config at ...`
- `config reload skipped (invalid config): ...`
- `Config write rejected: ...`
- یک فایل `openclaw.json.rejected.*` دارای برچسب زمانی در کنار پیکربندی فعال.
- یک فایل `openclaw.json.clobbered.*` دارای برچسب زمانی، اگر `doctor --fix` یک ویرایش مستقیم خراب را تعمیر کرده باشد.
- OpenClaw جدیدترین 32 فایل `.clobbered.*` را برای هر مسیر پیکربندی نگه می‌دارد و فایل‌های قدیمی‌تر را چرخش می‌دهد.

<AccordionGroup>
  <Accordion title="چه اتفاقی افتاد">
    - پیکربندی هنگام راه‌اندازی، بارگذاری مجدد فوری، یا نوشتن تحت مالکیت OpenClaw اعتبارسنجی نشد.
    - راه‌اندازی Gateway به‌صورت بسته شکست می‌خورد و `openclaw.json` را بازنویسی نمی‌کند.
    - بارگذاری مجدد فوری، ویرایش‌های خارجی نامعتبر را نادیده می‌گیرد و پیکربندی زمان اجرای فعلی را فعال نگه می‌دارد.
    - نوشتن‌های تحت مالکیت OpenClaw، payloadهای نامعتبر/مخرب را پیش از ثبت رد می‌کنند و `.rejected.*` را ذخیره می‌کنند.
    - تعمیر بر عهده `openclaw doctor --fix` است. این مؤلفه می‌تواند پیشوندهای غیر JSON را حذف کند یا آخرین نسخه سالم شناخته‌شده را بازیابی کند، درحالی‌که payload ردشده را به‌شکل `.clobbered.*` حفظ می‌کند.
    - هنگامی که تعمیرات زیادی برای یک مسیر پیکربندی انجام می‌شود، OpenClaw فایل‌های قدیمی‌تر `.clobbered.*` را چرخش می‌دهد تا جدیدترین payload تعمیرشده همچنان در دسترس باشد.

  </Accordion>
  <Accordion title="بررسی و تعمیر">
    ```bash
    CONFIG="$(openclaw config file)"
    ls -lt "$CONFIG".clobbered.* "$CONFIG".rejected.* 2>/dev/null | head
    diff -u "$CONFIG" "$(ls -t "$CONFIG".clobbered.* 2>/dev/null | head -n 1)"
    openclaw config validate
    openclaw doctor
    ```
  </Accordion>
  <Accordion title="نشانه‌های رایج">
    - `.clobbered.*` وجود دارد ← doctor هنگام تعمیر پیکربندی فعال، یک ویرایش خارجی خراب را حفظ کرده است.
    - `.rejected.*` وجود دارد ← نوشتن پیکربندی تحت مالکیت OpenClaw پیش از ثبت، در بررسی‌های طرح‌واره یا بازنویسی مخرب شکست خورده است.
    - `Config write rejected:` ← عملیات نوشتن تلاش کرده ساختار الزامی را حذف کند، اندازهٔ فایل را به‌شدت کاهش دهد، یا پیکربندی نامعتبر را ذخیره کند.
    - `config reload skipped (invalid config):` ← یک ویرایش مستقیم در اعتبارسنجی شکست خورده و Gateway در حال اجرا آن را نادیده گرفته است.
    - `Invalid config at ...` ← راه‌اندازی پیش از آغاز به کار سرویس‌های Gateway شکست خورده است.
    - `missing-meta-vs-last-good`، `gateway-mode-missing-vs-last-good`، یا `size-drop-vs-last-good:*` ← نوشتن تحت مالکیت OpenClaw رد شده، زیرا در مقایسه با آخرین پشتیبان سالم شناخته‌شده، فیلدها یا اندازه را از دست داده است.
    - `Config last-known-good promotion skipped` ← گزینهٔ پیشنهادی حاوی جای‌نگهدارهای محرمانهٔ پوشانده‌شده مانند `***` بوده است.

  </Accordion>
  <Accordion title="گزینه‌های رفع مشکل">
    1. `openclaw doctor --fix` را اجرا کنید تا doctor پیکربندی پیشونددار/بازنویسی‌شده را تعمیر کند یا آخرین نسخهٔ سالم شناخته‌شده را بازیابی کند.
    2. فقط کلیدهای موردنظر را از `.clobbered.*` یا `.rejected.*` کپی کنید، سپس آن‌ها را با `openclaw config set` یا `config.patch` اعمال کنید.
    3. پیش از راه‌اندازی مجدد، `openclaw config validate` را اجرا کنید.
    4. اگر به‌صورت دستی ویرایش می‌کنید، پیکربندی کامل JSON5 را نگه دارید، نه فقط شیء جزئی‌ای را که قصد تغییرش را داشتید.
  </Accordion>
</AccordionGroup>

مرتبط:

- [پیکربندی](/fa/cli/config)
- [پیکربندی: بارگذاری مجدد فوری](/fa/gateway/configuration#config-hot-reload)
- [پیکربندی: اعتبارسنجی سخت‌گیرانه](/fa/gateway/configuration#strict-validation)
- [Doctor](/fa/gateway/doctor)

## هشدارهای کاوش Gateway

زمانی استفاده کنید که `openclaw gateway probe` به چیزی دسترسی پیدا می‌کند، اما همچنان یک بلوک هشدار چاپ می‌کند.

```bash
openclaw gateway probe
openclaw gateway probe --json
openclaw gateway probe --ssh user@gateway-host
```

به‌دنبال موارد زیر باشید:

- `warnings[].code` و `primaryTargetId` در خروجی JSON.
- اینکه هشدار دربارهٔ بازگشت جایگزین SSH، چند Gateway، محدوده‌های دسترسی مفقود، یا ارجاع‌های احراز هویت حل‌نشده است.

نشانه‌های رایج:

- `SSH tunnel failed to start; falling back to direct probes.` ← راه‌اندازی SSH شکست خورده، اما فرمان همچنان اهداف مستقیم پیکربندی‌شده/حلقهٔ محلی را امتحان کرده است.
- `multiple reachable gateway identities detected` ← Gatewayهای متمایز پاسخ داده‌اند، یا OpenClaw نتوانسته ثابت کند اهداف در دسترس همان Gateway هستند. تونل SSH، نشانی پروکسی، یا نشانی راه دور پیکربندی‌شده به همان Gateway، حتی در صورت تفاوت پورت‌های انتقال، یک Gateway با چند روش انتقال در نظر گرفته می‌شود.
- `Read-probe diagnostics are limited by gateway scopes (missing operator.read)` ← اتصال برقرار شده، اما RPC جزئیات به محدودهٔ دسترسی محدود است؛ هویت دستگاه را جفت کنید یا از اعتبارنامه‌های دارای `operator.read` استفاده کنید.
- `Gateway accepted the WebSocket connection, but follow-up read diagnostics failed` ← اتصال برقرار شده، اما مجموعهٔ کامل RPCهای تشخیصی با پایان مهلت یا شکست مواجه شده است. این وضعیت را یک Gateway در دسترس با قابلیت‌های تشخیصی تضعیف‌شده در نظر بگیرید؛ `connect.ok` و `connect.rpcOk` را در خروجی `--json` مقایسه کنید.
- `Capability: pairing-pending` یا `gateway closed (1008): pairing required` ← Gateway پاسخ داده، اما این کارخواه همچنان پیش از دسترسی عادی اپراتور به جفت‌سازی/تأیید نیاز دارد.
- متن هشدار SecretRef حل‌نشدهٔ `gateway.auth.*` / `gateway.remote.*` ← اطلاعات احراز هویت در این مسیر فرمان برای هدف ناموفق در دسترس نبوده است.

مرتبط:

- [Gateway](/fa/cli/gateway)
- [چند Gateway روی یک میزبان](/fa/gateway#multiple-gateways-same-host)
- [دسترسی راه دور](/fa/gateway/remote)

## کانال متصل است، اما پیام‌ها جریان ندارند

اگر وضعیت کانال متصل است اما جریان پیام متوقف شده، روی خط‌مشی، مجوزها و قواعد تحویل مختص کانال تمرکز کنید.

```bash
openclaw channels status --probe
openclaw pairing list --channel <channel> [--account <id>]
openclaw status --deep
openclaw logs --follow
openclaw config get channels
```

به‌دنبال موارد زیر باشید:

- خط‌مشی پیام مستقیم (`pairing`، `allowlist`، `open`، `disabled`).
- فهرست مجاز گروه و الزامات اشاره.
- مجوزها/محدوده‌های دسترسی API کانال که مفقودند.

نشانه‌های رایج:

- `mention required` ← پیام به‌دلیل خط‌مشی اشاره در گروه نادیده گرفته شده است.
- `pairing` / ردپاهای تأیید در انتظار ← فرستنده تأیید نشده است.
- `missing_scope`، `not_in_channel`، `Forbidden`، `401/403` ← مشکل احراز هویت/مجوزهای کانال.

مرتبط:

- [عیب‌یابی کانال](/fa/channels/troubleshooting)
- [Discord](/fa/channels/discord)
- [Telegram](/fa/channels/telegram)
- [WhatsApp](/fa/channels/whatsapp)

## تحویل Cron و Heartbeat

اگر Cron یا Heartbeat اجرا یا تحویل نشد، ابتدا وضعیت زمان‌بند و سپس هدف تحویل را بررسی کنید.

```bash
openclaw cron status
openclaw cron list
openclaw cron runs --id <jobId> --limit 20
openclaw system heartbeat last
openclaw logs --follow
```

به‌دنبال موارد زیر باشید:

- فعال بودن Cron و وجود بیدارباش بعدی.
- وضعیت تاریخچهٔ اجرای کار (`ok`، `skipped`، `error`).
- دلایل رد شدن Heartbeat (`quiet-hours`، `requests-in-flight`، `cron-in-progress`، `lanes-busy`، `alerts-disabled`، `empty-heartbeat-file`).

<AccordionGroup>
  <Accordion title="نشانه‌های رایج">
    - `cron: scheduler disabled; jobs will not run automatically` ← Cron غیرفعال است.
    - `cron: timer tick failed` ← تیک زمان‌بند شکست خورده است؛ خطاهای فایل/گزارش/زمان اجرا را بررسی کنید.
    - `heartbeat skipped` همراه با `reason=quiet-hours` ← خارج از بازهٔ ساعات فعال.
    - `heartbeat skipped` همراه با `reason=empty-heartbeat-file` ← پیش‌نویس پایشگر Heartbeat فقط شامل داربست خالی، نظر، سرصفحه، حصار یا فهرست بررسی خالی است، بنابراین OpenClaw فراخوانی مدل را رد می‌کند.
    - `heartbeat: unknown accountId` ← شناسهٔ حساب برای هدف تحویل Heartbeat نامعتبر است.
    - `heartbeat skipped` همراه با `reason=dm-blocked` ← هدف Heartbeat به مقصدی از نوع پیام مستقیم حل شده، درحالی‌که `agents.defaults.heartbeat.directPolicy` (یا بازنویسی مختص عامل) روی `block` تنظیم شده است.

  </Accordion>
</AccordionGroup>

مرتبط:

- [Heartbeat](/fa/gateway/heartbeat)
- [وظایف زمان‌بندی‌شده](/fa/automation/cron-jobs)
- [وظایف زمان‌بندی‌شده: عیب‌یابی](/fa/automation/cron-jobs#troubleshooting)

## Node جفت شده، اما ابزار شکست می‌خورد

اگر یک Node جفت شده اما ابزارها شکست می‌خورند، وضعیت پیش‌زمینه، مجوز و تأیید را به‌صورت جداگانه بررسی کنید.

```bash
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
openclaw approvals get --node <idOrNameOrIp>
openclaw logs --follow
openclaw status
```

به‌دنبال موارد زیر باشید:

- آنلاین بودن Node با قابلیت‌های مورد انتظار.
- اعطای مجوزهای سیستم‌عامل برای دوربین/میکروفن/موقعیت مکانی/صفحه‌نمایش.
- وضعیت تأییدهای اجرا و فهرست مجاز.

نشانه‌های رایج:

- `NODE_BACKGROUND_UNAVAILABLE` ← برنامهٔ Node باید در پیش‌زمینه باشد.
- `*_PERMISSION_REQUIRED` / `LOCATION_PERMISSION_REQUIRED` ← مجوز سیستم‌عامل مفقود است.
- `SYSTEM_RUN_DENIED: approval required` ← تأیید اجرا در انتظار است.
- `SYSTEM_RUN_DENIED: allowlist miss` ← فرمان توسط فهرست مجاز مسدود شده است.

مرتبط:

- [تأییدهای اجرا](/fa/tools/exec-approvals)
- [عیب‌یابی Node](/fa/nodes/troubleshooting)
- [Nodeها](/fa/nodes/index)

## ابزار مرورگر شکست می‌خورد

زمانی استفاده کنید که عملیات ابزار مرورگر شکست می‌خورند، حتی با اینکه خود Gateway سالم است.

```bash
openclaw browser status
openclaw browser start --browser-profile openclaw
openclaw browser profiles
openclaw logs --follow
openclaw doctor
```

به‌دنبال موارد زیر باشید:

- اینکه آیا `plugins.allow` تنظیم شده و شامل `browser` است.
- مسیر معتبر فایل اجرایی مرورگر.
- دسترس‌پذیری نمایهٔ CDP.
- در دسترس بودن Chrome محلی برای نمایه‌های `existing-session` / `user`.

<AccordionGroup>
  <Accordion title="نشانه‌های Plugin / فایل اجرایی">
    - `unknown command "browser"` یا `unknown command 'browser'` ← Plugin مرورگر همراه توسط `plugins.allow` مستثنا شده است.
    - ابزار مرورگر مفقود / در دسترس نیست درحالی‌که `browser.enabled=true` ← `plugins.allow`، `browser` را مستثنا می‌کند، بنابراین Plugin هرگز بارگذاری نشده است.
    - `Failed to start Chrome CDP on port` ← فرایند مرورگر نتوانسته اجرا شود.
    - `browser.executablePath not found` ← مسیر پیکربندی‌شده نامعتبر است.
    - `browser.cdpUrl must be http(s) or ws(s)` ← نشانی CDP پیکربندی‌شده از طرح پشتیبانی‌نشده‌ای مانند `file:` یا `ftp:` استفاده می‌کند.
    - `browser.cdpUrl has invalid port` ← نشانی CDP پیکربندی‌شده دارای پورت نامعتبر یا خارج از محدوده است.
    - `Playwright is not available in this gateway build; '<feature>' is unsupported.` ← نصب فعلی Gateway وابستگی اصلی زمان اجرای مرورگر را ندارد؛ OpenClaw را دوباره نصب یا به‌روزرسانی کنید، سپس Gateway را راه‌اندازی مجدد کنید. تصویرهای فوری ARIA و نماگرفت‌های سادهٔ صفحه همچنان می‌توانند کار کنند، اما پیمایش، تصویرهای فوری هوش مصنوعی، نماگرفت عناصر با انتخابگر CSS و برون‌بری PDF در دسترس نمی‌مانند.

  </Accordion>
  <Accordion title="نشانه‌های Chrome MCP / نشست موجود">
    - `Could not find DevToolsActivePort for chrome` ← نشست موجود Chrome MCP هنوز نتوانسته به پوشهٔ دادهٔ مرورگر انتخاب‌شده متصل شود. صفحهٔ بازرسی مرورگر را باز کنید، اشکال‌زدایی راه دور را فعال کنید، مرورگر را باز نگه دارید، نخستین درخواست اتصال را تأیید کنید و سپس دوباره تلاش کنید. اگر وضعیت ورود لازم نیست، نمایهٔ مدیریت‌شدهٔ `openclaw` را ترجیح دهید.
    - `No browser tabs found for profile="user"` ← نمایهٔ اتصال Chrome MCP هیچ زبانهٔ محلی باز Chrome ندارد.
    - `Remote CDP for profile "<name>" is not reachable` ← نقطهٔ پایانی CDP راه دور پیکربندی‌شده از میزبان Gateway در دسترس نیست.
    - `Browser attachOnly is enabled ... not reachable` یا `Browser attachOnly is enabled and CDP websocket ... is not reachable` ← نمایهٔ فقط اتصال هیچ هدف در دسترسی ندارد، یا نقطهٔ پایانی HTTP پاسخ داده اما WebSocket مربوط به CDP همچنان باز نشده است.

  </Accordion>
  <Accordion title="نشانه‌های عنصر / نماگرفت / بارگذاری">
    - `fullPage is not supported for element screenshots` ← درخواست نماگرفت، `--full-page` را با `--ref` یا `--element` ترکیب کرده است.
    - `element screenshots are not supported for existing-session profiles; use ref from snapshot.` ← فراخوانی‌های نماگرفت Chrome MCP / `existing-session` باید از ثبت صفحه یا `--ref` تصویر فوری استفاده کنند، نه `--element` از نوع CSS.
    - `existing-session file uploads do not support element selectors; use ref/inputRef.` ← قلاب‌های بارگذاری Chrome MCP به ارجاع‌های تصویر فوری نیاز دارند، نه انتخابگرهای CSS.
    - `existing-session file uploads currently support one file at a time.` ← در نمایه‌های Chrome MCP برای هر فراخوانی فقط یک بارگذاری ارسال کنید.
    - `existing-session dialog handling does not support timeoutMs.` ← قلاب‌های گفت‌وگو در نمایه‌های Chrome MCP از بازنویسی مهلت زمانی پشتیبانی نمی‌کنند.
    - `existing-session type does not support timeoutMs overrides.` ← برای `act:type` در نمایه‌های نشست موجود `profile="user"` / Chrome MCP، `timeoutMs` را حذف کنید؛ یا هنگامی که مهلت زمانی سفارشی لازم است، از نمایهٔ مرورگر مدیریت‌شده/CDP استفاده کنید.
    - `response body is not supported for existing-session profiles yet.` ← `responsebody` همچنان به مرورگر مدیریت‌شده یا نمایهٔ خام CDP نیاز دارد.
    - بازنویسی‌های قدیمیِ محدودهٔ دید / حالت تیره / زبان‌محلی / آفلاین در نمایه‌های فقط اتصال یا CDP راه دور ← `openclaw browser stop --browser-profile <name>` را اجرا کنید تا نشست کنترل فعال بسته شود و وضعیت شبیه‌سازی Playwright/CDP بدون راه‌اندازی مجدد کل Gateway آزاد شود.

  </Accordion>
</AccordionGroup>

مرتبط:

- [مرورگر (مدیریت‌شده توسط OpenClaw)](/fa/tools/browser)
- [عیب‌یابی مرورگر](/fa/tools/browser-linux-troubleshooting)

## اگر ارتقا دادید و ناگهان چیزی خراب شد

بیشتر خرابی‌های پس از ارتقا ناشی از تغییر تدریجی پیکربندی یا اعمال شدن پیش‌فرض‌های سخت‌گیرانه‌تر است.

<AccordionGroup>
  <Accordion title="1. رفتار احراز هویت و بازنویسی نشانی تغییر کرده است">
    ```bash
    openclaw gateway status
    openclaw config get gateway.mode
    openclaw config get gateway.remote.url
    openclaw config get gateway.auth.mode
    ```

    مواردی که باید بررسی شوند:

    - اگر `gateway.mode=remote`، ممکن است فراخوانی‌های CLI سرویس راه‌دور را هدف بگیرند، درحالی‌که سرویس محلی به‌درستی کار می‌کند.
    - فراخوانی‌های صریح `--url` به اعتبارنامه‌های ذخیره‌شده بازنمی‌گردند.

    نشانه‌های رایج:

    - `gateway connect failed:` ← نشانی URL هدف اشتباه است.
    - `unauthorized` ← نقطه پایانی در دسترس است، اما احراز هویت اشتباه است.

  </Accordion>
  <Accordion title="2. محدودیت‌های اتصال و احراز هویت سخت‌گیرانه‌تر هستند">
    ```bash
    openclaw config get gateway.bind
    openclaw config get gateway.auth.mode
    openclaw config get gateway.auth.token
    openclaw gateway status
    openclaw logs --follow
    ```

    مواردی که باید بررسی شوند:

    - اتصال‌های غیرحلقه‌بازگشتی (`lan`، `tailnet`، `custom`) به یک مسیر معتبر احراز هویت Gateway نیاز دارند: احراز هویت با توکن/گذرواژه مشترک، یا استقرار غیرحلقه‌بازگشتی `trusted-proxy` که به‌درستی پیکربندی شده باشد.
    - کلیدهای قدیمی مانند `gateway.token` جایگزین `gateway.auth.token` نمی‌شوند.

    نشانه‌های رایج:

    - `refusing to bind gateway ... without auth` ← اتصال غیرحلقه‌بازگشتی بدون مسیر معتبر احراز هویت Gateway.
    - `Connectivity probe: failed` درحالی‌که زمان‌اجرا فعال است ← Gateway فعال است، اما با احراز هویت/نشانی URL فعلی نمی‌توان به آن دسترسی یافت.

  </Accordion>
  <Accordion title="3. وضعیت جفت‌سازی و هویت دستگاه تغییر کرده است">
    ```bash
    openclaw devices list
    openclaw pairing list --channel <channel> [--account <id>]
    openclaw logs --follow
    openclaw doctor
    ```

    مواردی که باید بررسی شوند:

    - تأییدهای در انتظار دستگاه برای داشبورد/نودها.
    - تأییدهای در انتظار جفت‌سازی پیام خصوصی پس از تغییرات خط‌مشی یا هویت.

    نشانه‌های رایج:

    - `device identity required` ← احراز هویت دستگاه انجام نشده است.
    - `pairing required` ← فرستنده/دستگاه باید تأیید شود.

  </Accordion>
</AccordionGroup>

اگر پس از بررسی‌ها همچنان پیکربندی سرویس و زمان‌اجرا با یکدیگر ناسازگارند، فراداده سرویس را از همان پوشه پروفایل/وضعیت دوباره نصب کنید:

```bash
openclaw gateway install --force
openclaw gateway restart
```

مطالب مرتبط:

- [احراز هویت](/fa/gateway/authentication)
- [اجرای پس‌زمینه و ابزار فرایند](/fa/gateway/background-process)
- [جفت‌سازی Node](/fa/gateway/pairing)

## مطالب مرتبط

- [Doctor](/fa/gateway/doctor)
- [پرسش‌های متداول](/fa/help/faq)
- [راهنمای عملیاتی Gateway](/fa/gateway)
