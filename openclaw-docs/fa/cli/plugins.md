---
read_when:
    - می‌خواهید Pluginهای Gateway یا بسته‌های سازگار را نصب یا مدیریت کنید
    - می‌خواهید یک Plugin ابزار ساده را چارچوب‌بندی یا اعتبارسنجی کنید
    - می‌خواهید خطاهای بارگذاری Plugin را اشکال‌زدایی کنید
sidebarTitle: Plugins
summary: مرجع CLI برای `openclaw plugins` (مقداردهی اولیه، ساخت، اعتبارسنجی، فهرست‌کردن، نصب، بازار، حذف نصب، فعال/غیرفعال‌کردن، عیب‌یابی)
title: Pluginها
x-i18n:
    generated_at: "2026-07-27T16:22:46Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a1acba76fb1bc0ddae75e51fe573d3c2ac8f694607836e0c072ec7ca8fc0e262
    source_path: cli/plugins.md
    workflow: 16
---

مدیریت Pluginهای Gateway، بسته‌های هوک و باندل‌های سازگار.

<CardGroup cols={2}>
  <Card title="سیستم Plugin" href="/fa/tools/plugin">
    راهنمای کاربر نهایی برای نصب، فعال‌سازی و عیب‌یابی Pluginها.
  </Card>
  <Card title="مدیریت Pluginها" href="/fa/plugins/manage-plugins">
    نمونه‌های سریع برای نصب، فهرست‌کردن، به‌روزرسانی، حذف نصب و انتشار.
  </Card>
  <Card title="باندل‌های Plugin" href="/fa/plugins/bundles">
    مدل سازگاری باندل.
  </Card>
  <Card title="مانیفست Plugin" href="/fa/plugins/manifest">
    فیلدهای مانیفست و شِمای پیکربندی.
  </Card>
  <Card title="امنیت" href="/fa/gateway/security">
    مقاوم‌سازی امنیتی نصب‌های Plugin.
  </Card>
</CardGroup>

## فرمان‌ها

```bash
openclaw plugins list [--enabled] [--verbose] [--json]
openclaw plugins search <query> [--limit <n>] [--json]
openclaw plugins install <path-or-spec> [--link] [--force] [--pin] [--marketplace <source>]
openclaw plugins inspect <id> [--runtime] [--json]
openclaw plugins inspect --all [--runtime] [--json]
openclaw plugins info <id>                    # نام مستعار inspect
openclaw plugins enable <id>
openclaw plugins disable <id>
openclaw plugins uninstall <id> [--dry-run] [--keep-files] [--force]
openclaw plugins update <id-or-npm-spec> | --all [--dry-run]
openclaw plugins registry [--refresh] [--json]
openclaw plugins doctor
openclaw plugins init <id> [--name <name>] [--type tool|provider] [--directory <path>]
openclaw plugins build [--entry <path>] [--check]
openclaw plugins validate [--entry <path>]
openclaw plugins marketplace entries [--offline] [--feed-profile <name>] [--json]
openclaw plugins marketplace list <source> [--json]
openclaw plugins marketplace refresh [--feed-profile <name>] [--expected-sha256 <sha256>] [--json]
```

برای بررسی نصب، بازرسی، حذف نصب یا تازه‌سازی رجیستری که کند است، فرمان را با
`OPENCLAW_PLUGIN_LIFECYCLE_TRACE=1` اجرا کنید. ردگیری، زمان‌بندی مرحله‌ها را در stderr می‌نویسد
و خروجی JSON را قابل‌تجزیه نگه می‌دارد. به [عیب‌یابی](/fa/help/debugging#plugin-lifecycle-trace) مراجعه کنید.

<Note>
در حالت Nix ‏(`OPENCLAW_NIX_MODE=1`)، ‏`openclaw.json` تغییرناپذیر است. همهٔ `install`، ‏`update`، ‏`uninstall`، ‏`enable` و `disable` از اجرا خودداری می‌کنند. در عوض، منبع Nix این نصب را ویرایش کنید (`programs.openclaw.config` یا `instances.<name>.config` برای nix-openclaw)، سپس دوباره بسازید. به [شروع سریع](https://github.com/openclaw/nix-openclaw#quick-start) مبتنی بر عامل مراجعه کنید.
</Note>

<Note>
Pluginهای همراه با OpenClaw عرضه می‌شوند. برخی به‌طور پیش‌فرض فعال‌اند (برای مثال، ارائه‌دهندگان مدل همراه، ارائه‌دهندگان گفتار همراه و Plugin مرورگر همراه)؛ بقیه به `plugins enable` نیاز دارند.

Pluginهای بومی OpenClaw، ‏`openclaw.plugin.json` را با یک شِمای JSON درون‌خطی (`configSchema`، حتی اگر خالی باشد) عرضه می‌کنند. باندل‌های سازگار در عوض از مانیفست‌های باندل خود استفاده می‌کنند.

`plugins list`، ‏`Format: openclaw` یا `Format: bundle` را نمایش می‌دهد. خروجی تفصیلی فهرست/اطلاعات همچنین زیرنوع باندل (`codex`، ‏`claude` یا `cursor`) و قابلیت‌های شناسایی‌شدهٔ باندل را نمایش می‌دهد.
</Note>

## تألیف

```bash
openclaw plugins init stock-quotes --name "Stock Quotes"
cd stock-quotes
npm run plugin:build
npm run plugin:validate
```

`plugins init` به‌طور پیش‌فرض یک Plugin ابزار TypeScript حداقلی ایجاد می‌کند. آرگومان نخست
شناسهٔ Plugin است؛ `--name` نام نمایشی را تنظیم می‌کند. OpenClaw از
شناسه برای دایرکتوری خروجی پیش‌فرض و نام‌گذاری بسته استفاده می‌کند. داربست‌های ابزار از
`defineToolPlugin` استفاده می‌کنند و اسکریپت‌های `package.json` یعنی `plugin:build` و
`plugin:validate` را تولید می‌کنند که ابتدا می‌سازند و سپس `openclaw plugins build`/`validate` را فراخوانی می‌کنند.

`plugins build` ورودی ساخته‌شده را وارد می‌کند، فرادادهٔ ایستای ابزار آن را می‌خواند،
`openclaw.plugin.json` را می‌نویسد و `openclaw.extensions` متعلق به `package.json` را همگام نگه می‌دارد.
`plugins validate` بررسی می‌کند که مانیفست تولیدشده، فرادادهٔ بسته و
خروجی فعلی ورودی همچنان با یکدیگر مطابقت داشته باشند. برای گردش‌کار کامل
تألیف، به [Pluginهای ابزار](/fa/plugins/tool-plugins) مراجعه کنید.

داربست، کد منبع TypeScript را می‌نویسد اما فراداده را از ورودی ساخته‌شدهٔ
`./dist/index.js` تولید می‌کند؛ بنابراین گردش‌کار با CLI منتشرشده نیز کار می‌کند. هنگامی‌که
ورودی، ورودی پیش‌فرض بسته نیست، از `--entry <path>` استفاده کنید. در CI از
`plugins build --check` استفاده کنید تا در صورت قدیمی‌بودن فرادادهٔ تولیدشده، بدون
بازنویسی فایل‌ها شکست بخورد.

### داربست ارائه‌دهنده

```bash
openclaw plugins init acme-models --name "Acme Models" --type provider
cd acme-models
npm install
npm run build
npm test
npm run validate
```

داربست‌های ارائه‌دهنده، یک Plugin عمومی ارائه‌دهندهٔ مدل سازگار با OpenAI
با زیرساخت احراز هویت کلید API، اسکریپت `npm run validate` که
`clawhub package validate` را اجرا می‌کند، فرادادهٔ بستهٔ ClawHub و یک
گردش‌کار GitHub Actions با اجرای دستی برای انتشار مورداعتماد آینده از طریق GitHub
OIDC ایجاد می‌کنند. داربست‌های ارائه‌دهنده Skills تولید نمی‌کنند و از
`openclaw plugins build`/`validate` استفاده نمی‌کنند؛ این فرمان‌ها برای مسیر
فرادادهٔ تولیدشدهٔ داربست ابزار هستند.

پیش از انتشار، URL پایهٔ API جای‌نگهدار، کاتالوگ مدل، مسیر مستندات،
متن اعتبارنامه و متن README را با جزئیات واقعی ارائه‌دهنده جایگزین کنید. برای
نخستین انتشار در ClawHub و راه‌اندازی ناشر مورداعتماد، از README تولیدشده استفاده کنید.

## نصب

```bash
openclaw plugins search "calendar"                      # جست‌وجوی Pluginهای ClawHub
openclaw plugins install @openclaw/<package>            # کاتالوگ رسمی مورداعتماد
openclaw plugins install <package>                       # بستهٔ دلخواه npm
openclaw plugins install clawhub:<package>                # فقط ClawHub
openclaw plugins install npm:<package>                    # فقط npm
openclaw plugins install npm-pack:<path.tgz>               # آرشیو محلی npm-pack
openclaw plugins install git:github.com/<owner>/<repo>     # مخزن git
openclaw plugins install git:github.com/<owner>/<repo>@<ref>
openclaw plugins install <path>                            # مسیر یا آرشیو محلی
openclaw plugins install -l <path>                         # پیوند به‌جای کپی
openclaw plugins install <plugin>@<marketplace>             # صورت کوتاه بازار
openclaw plugins install <plugin> --marketplace <name>      # بازار (صریح)
openclaw plugins install <package> --force                  # تأیید منبع / بازنویسی موجود
openclaw plugins install <package> --pin                    # سنجاق‌کردن نسخهٔ حل‌شدهٔ npm
openclaw plugins install clawhub:<package> --acknowledge-clawhub-risk
openclaw plugins install <package> --dangerously-force-unsafe-install
```

نگه‌دارندگانی که نصب‌های زمان راه‌اندازی را آزمایش می‌کنند، می‌توانند منابع نصب خودکار
Plugin را با متغیرهای محیطی محافظت‌شده بازنویسی کنند. به
[بازنویسی‌های نصب Plugin](/fa/plugins/install-overrides) مراجعه کنید.

<Warning>
در دورهٔ گذار راه‌اندازی، نام‌های سادهٔ بسته به‌طور پیش‌فرض از npm نصب می‌شوند، مگر اینکه با شناسهٔ یک Plugin همراه یا رسمی مطابقت داشته باشند؛ در آن صورت OpenClaw به‌جای مراجعه به رجیستری npm از همان نسخهٔ محلی/رسمی استفاده می‌کند. هنگامی‌که عمداً یک بستهٔ خارجی npm می‌خواهید، از `npm:<package>` استفاده کنید. برای ClawHub از `clawhub:<package>` استفاده کنید. نصب Pluginها را مانند اجرای کد در نظر بگیرید؛ نسخه‌های سنجاق‌شده را ترجیح دهید.
</Warning>

<Warning>
بسته‌های ClawHub و کاتالوگ همراه/رسمی OpenClaw منابع نصب
مورداعتماد هستند. یک منبع جدید و دلخواه npm، ‏`npm-pack:`، ‏git، مسیر/آرشیو محلی یا
بازار، پیش از ادامه هشدار می‌دهد و تأیید می‌خواهد. نصب‌های غیرتعاملی دلخواه
پس از بازبینی و اعتماد به منبع باید `--force` را ارائه کنند. همین
پرچم در صورت نیاز مقصد نصب موجود را بازنویسی می‌کند. به‌روزرسانی عادی
یک نصب ازپیش‌ردیابی‌شده به آن نیاز ندارد. این تأیید جدا از
`--acknowledge-clawhub-risk` است که فقط برای هشدارهای پرریسک اعتماد به انتشار
ClawHub کاربرد دارد. `--force`، ‏`security.installPolicy` یا دیگر
بررسی‌های ایمنی نصب را دور نمی‌زند.
</Warning>

`plugins search` برای بسته‌های قابل‌نصب `code-plugin` و
`bundle-plugin` از ClawHub پرس‌وجو می‌کند (نه Skills؛ برای آن‌ها از `openclaw skills search` استفاده کنید).
مقدار پیش‌فرض `--limit` برابر 20 است و سقف آن 100 است. این فرمان فقط کاتالوگ دوردست را می‌خواند: هیچ
بازرسی وضعیت محلی، تغییر پیکربندی، نصب بسته یا بارگذاری زمان اجرای
Plugin انجام نمی‌شود. نتایج شامل نام بستهٔ ClawHub، خانواده، کانال، نسخه،
خلاصه و یک راهنمای نصب مانند `openclaw plugins install clawhub:<package>` هستند.

<Note>
ClawHub سطح اصلی توزیع و کشف برای بیشتر Pluginها است. Npm
همچنان به‌عنوان مسیر جایگزین پشتیبانی‌شده و نصب مستقیم باقی می‌ماند. بسته‌های Plugin
`@openclaw/*` متعلق به OpenClaw دوباره در npm منتشر می‌شوند؛ فهرست فعلی را در
[npmjs.com/org/openclaw](https://www.npmjs.com/org/openclaw) یا
[موجودی Pluginها](/fa/plugins/plugin-inventory) ببینید. نصب‌های پایدار از `latest` استفاده می‌کنند.
نصب‌ها و به‌روزرسانی‌های کانال بتا، در صورت موجودبودن dist-tag ‏`beta` مربوط به npm را ترجیح می‌دهند
و در غیر این صورت به `latest` برمی‌گردند. در کانال پایدارِ توسعه‌یافته، Pluginهای رسمی npm
با قصد ساده/پیش‌فرض یا `latest` دقیقاً به نسخهٔ هستهٔ نصب‌شده
حل می‌شوند. سنجاق‌های دقیق و تگ‌های صریح غیرِ `latest`، بسته‌های شخص ثالث و
منابع غیرِ npm بازنویسی نمی‌شوند.
</Note>

<AccordionGroup>
  <Accordion title="شامل‌کردن پیکربندی و ترمیم پیکربندی نامعتبر">
    اگر بخش `plugins` شما بر یک `$include` تک‌فایلی تکیه دارد، `plugins install/update/enable/disable/uninstall` مستقیماً در همان فایل شامل‌شده می‌نویسد و `openclaw.json` را دست‌نخورده باقی می‌گذارد. شامل‌کردن در ریشه، آرایه‌های شامل و شامل‌هایی با بازنویسی‌های هم‌سطح، به‌جای مسطح‌سازی به‌شکل بسته شکست می‌خورند. برای شکل‌های پشتیبانی‌شده به [شامل‌کردن پیکربندی](/fa/gateway/configuration) مراجعه کنید.

    اگر پیکربندی پیش از نصب نامعتبر باشد، `plugins install` معمولاً به‌شکل بسته شکست می‌خورد و از شما می‌خواهد ابتدا `openclaw doctor --fix` را اجرا کنید. هنگام راه‌اندازی Gateway و بارگذاری مجدد داغ، پیکربندی نامعتبر Plugin مانند هر پیکربندی نامعتبر دیگری به‌شکل بسته شکست می‌خورد؛ `openclaw doctor --fix` می‌تواند ورودی نامعتبر Plugin را قرنطینه کند. تنها استثنای پیکربندی ازپیش‌موجود، یک مسیر بازیابی محدود برای Pluginهای همراهی است که صریحاً `openclaw.install.allowInvalidConfigRecovery` را می‌پذیرند.

    هنگامی‌که پیکربندی میزبان موجود معتبر است اما پیکربندی خود Plugin تازه‌نصب‌شده وجود ندارد، OpenClaw به‌جای نوشتن یک ورودی فعالِ نامعتبر، نصب را غیرفعال ثبت می‌کند. `plugins.entries.<id>.config` را پیکربندی کنید، سپس `openclaw plugins enable <id>` را اجرا کنید. اگر ورودی پیکربندی Plugin از قبل موجود اما نامعتبر باشد، نصب بدون بازنویسی آن شکست می‌خورد.

  </Accordion>
  <Accordion title="تأیید --force و نصب مجدد در برابر به‌روزرسانی">
    `--force` یک منبع غیرِ ClawHub را بدون نمایش درخواست تأیید می‌کند. این گزینه `security.installPolicy` یا دیگر بررسی‌های ایمنی نصب را دور نمی‌زند. هنگامی‌که Plugin یا بستهٔ هوک از قبل نصب شده باشد، مقصد موجود را نیز دوباره استفاده و درجا بازنویسی می‌کند. پس از بازبینی یک منبع دلخواه npm، محلی، آرشیو، git یا بازار، یا هنگامی‌که عمداً همان شناسه را دوباره نصب می‌کنید، از آن استفاده کنید. برای ارتقاهای معمول یک Plugin ازپیش‌ردیابی‌شدهٔ npm، ‏`openclaw plugins update <id-or-npm-spec>` را ترجیح دهید.

    اگر `plugins install` را برای شناسهٔ Pluginای که از قبل نصب شده است اجرا کنید، OpenClaw متوقف می‌شود و برای ارتقای عادی شما را به `plugins update <id-or-npm-spec>`، یا هنگامی‌که واقعاً می‌خواهید نصب فعلی را از منبعی متفاوت بازنویسی کنید به `plugins install <package> --force` هدایت می‌کند. منابع دلخواه همچنان هشدار تعاملی منشأ را نمایش می‌دهند؛ نصب‌های غیرتعاملی پس از بازبینی باید `--force` را ارائه کنند. منابع مورداعتماد ClawHub و کاتالوگ OpenClaw به آن نیاز ندارند. با `--link`، ‏`--force` منبع را تأیید می‌کند اما حالت نصب مسیر پیوندشده را تغییر نمی‌دهد.

  </Accordion>
  <Accordion title="دامنه --pin">
    `--pin` فقط برای نصب‌های npm اعمال می‌شود و `<name>@<version>` دقیقِ حل‌شده را ثبت می‌کند. این گزینه با نصب‌های `git:` پشتیبانی نمی‌شود (در عوض، ref را در مشخصه پین کنید؛ برای مثال `git:github.com/acme/plugin@v1.2.3`) و با `--marketplace` نیز پشتیبانی نمی‌شود (نصب‌های marketplace به‌جای مشخصه npm، فراداده منبع marketplace را نگه می‌دارند).
  </Accordion>
  <Accordion title="--dangerously-force-unsafe-install">
    `--dangerously-force-unsafe-install` منسوخ شده و اکنون هیچ عملی انجام نمی‌دهد. OpenClaw دیگر مسدودسازی داخلی کد خطرناک هنگام نصب را برای نصب Pluginها اجرا نمی‌کند.

    هنگامی که خط‌مشی نصب مختص میزبان لازم است، از سطح `security.installPolicy` تحت مالکیت اپراتور استفاده کنید. هوک‌های `before_install` در Plugin، هوک‌های چرخه‌عمر زمان اجرای Plugin هستند، نه مرز اصلی خط‌مشی برای نصب‌های CLI.

    اگر Plugin منتشرشده شما در ClawHub به‌دلیل اسکن رجیستری پنهان یا مسدود شده است، از مراحل ناشر در [انتشار در ClawHub](/fa/clawhub/publishing) استفاده کنید. `--dangerously-force-unsafe-install` از ClawHub نمی‌خواهد Plugin را دوباره اسکن کند یا یک انتشار مسدودشده را عمومی سازد.

  </Accordion>
  <Accordion title="--acknowledge-clawhub-risk">
    نصب‌های عمومی ClawHub پیش از دانلود، سابقه اعتماد انتشار انتخاب‌شده را بررسی می‌کنند. اگر ClawHub دانلود انتشار را غیرفعال کند، یافته‌های مخرب اسکن را گزارش دهد، یا انتشار را در وضعیت تعدیل مسدودکننده (قرنطینه‌شده، لغوشده) قرار دهد، OpenClaw صرف‌نظر از این پرچم، آن را کاملاً رد می‌کند. برای وضعیت‌های اسکن پرخطر یا وضعیت‌های تعدیل غیرمسدودکننده، OpenClaw جزئیات اعتماد را نمایش می‌دهد و پیش از ادامه تأیید می‌خواهد.

    فقط پس از بررسی هشدار ClawHub و تصمیم به ادامه بدون اعلان تعاملی، از `--acknowledge-clawhub-risk` استفاده کنید. نتایج اسکن در انتظار یا منقضی‌شده (هنوز پاک نیستند) هشدار می‌دهند، اما به تأیید نیاز ندارند. بسته‌های رسمی ClawHub و منابع Plugin همراه OpenClaw این بررسی اعتماد انتشار را به‌طور کامل دور می‌زنند.

  </Accordion>
  <Accordion title="بسته‌های هوک و مشخصه‌های npm">
    `plugins install` همچنین سطح نصب برای بسته‌های هوکی است که `openclaw.hooks` را در `package.json` ارائه می‌کنند. برای مشاهده‌پذیری فیلترشده هوک و فعال‌سازی هر هوک از `openclaw hooks` استفاده کنید، نه برای نصب بسته.

    مشخصه‌های npm **فقط محدود به رجیستری** هستند (نام بسته به‌همراه **نسخه دقیق** یا **dist-tag** اختیاری). مشخصه‌های Git/URL/file و بازه‌های semver رد می‌شوند. برای ایمنی، نصب وابستگی‌ها در یک پروژه مدیریت‌شده npm به‌ازای هر Plugin با `--ignore-scripts` اجرا می‌شود، حتی اگر پوسته شما تنظیمات سراسری نصب npm داشته باشد. پروژه‌های مدیریت‌شده npm مربوط به Plugin، `overrides` سطح بسته npm در OpenClaw را به ارث می‌برند؛ بنابراین پین‌های امنیتی میزبان بر وابستگی‌های بالاکشیده‌شده Plugin نیز اعمال می‌شوند.

    برای صریح‌کردن وضوح‌دهی npm از `npm:<package>` استفاده کنید. مشخصه‌های ساده بسته نیز هنگام گذار راه‌اندازی مستقیماً از npm نصب می‌شوند، مگر اینکه با شناسه رسمی Plugin مطابقت داشته باشند.

    مشخصه‌های خام `@openclaw/*` که با Pluginهای همراه مطابقت دارند، پیش از بازگشت به npm به نسخه همراه تحت مالکیت ایمیج حل می‌شوند. برای مثال، `openclaw plugins install @openclaw/discord@2026.5.20 --pin` به‌جای ایجاد یک جایگزین مدیریت‌شده npm، از Plugin همراه Discord در بیلد فعلی OpenClaw استفاده می‌کند. برای اجبار استفاده از بسته خارجی npm، از `openclaw plugins install npm:@openclaw/discord@2026.5.20 --pin` استفاده کنید.

    مشخصه‌های ساده و `@latest` در مسیر پایدار باقی می‌مانند. نسخه‌های اصلاحی تاریخ‌دار OpenClaw مانند `2026.5.3-1` برای این بررسی پایدار محسوب می‌شوند. اگر npm هر یک از این دو شکل را به یک نسخه پیش‌انتشار حل کند، OpenClaw متوقف می‌شود و از شما می‌خواهد با یک برچسب پیش‌انتشار (`@beta`/`@rc`) یا یک نسخه دقیق پیش‌انتشار (`@1.2.3-beta.4`) صریحاً آن را بپذیرید.

    برای نصب‌های npm بدون نسخه دقیق (`npm:<package>` یا `npm:<package>@latest`)، OpenClaw پیش از نصب فراداده بسته حل‌شده را بررسی می‌کند. اگر جدیدترین بسته پایدار به API جدیدتر Plugin در OpenClaw یا حداقل نسخه جدیدتری از میزبان نیاز داشته باشد، OpenClaw نسخه‌های پایدار قدیمی‌تر را بررسی می‌کند و در عوض جدیدترین انتشار سازگار را نصب می‌کند. نسخه‌های دقیق و dist-tagهای صریح سخت‌گیرانه باقی می‌مانند: انتخاب ناسازگار ناموفق می‌شود و از شما می‌خواهد OpenClaw را ارتقا دهید یا نسخه‌ای سازگار انتخاب کنید.

    اگر یک مشخصه نصب ساده با شناسه رسمی Plugin مطابقت داشته باشد (برای مثال `diffs`)، OpenClaw ورودی کاتالوگ را مستقیماً نصب می‌کند. برای نصب بسته npm با همان نام، از یک مشخصه scopeدار صریح استفاده کنید (برای مثال `@scope/diffs`).

  </Accordion>
  <Accordion title="مخزن‌های Git">
    برای نصب مستقیم از یک مخزن git از `git:<repo>` استفاده کنید. شکل‌های پشتیبانی‌شده: `git:github.com/owner/repo`، `git:owner/repo`، `https://` کامل، `ssh://`، `git://`، `file://` و URLهای clone از نوع `git@host:owner/repo.git`. برای checkout کردن یک شاخه، برچسب یا commit پیش از نصب، `@<ref>` یا `#<ref>` را اضافه کنید.

    نصب‌های Git مخزن را در یک دایرکتوری موقت clone می‌کنند، در صورت وجود ref درخواستی آن را checkout می‌کنند و سپس از نصب‌کننده معمول دایرکتوری Plugin استفاده می‌کنند؛ بنابراین اعتبارسنجی manifest، خط‌مشی نصب اپراتور، عملیات نصب package manager و سوابق نصب مانند نصب‌های npm رفتار می‌کنند. نصب‌های ثبت‌شده git شامل URL/ref منبع به‌همراه commit حل‌شده هستند تا `openclaw plugins update` بتواند بعداً منبع را دوباره حل کند.

    پس از نصب از git، از `openclaw plugins inspect <id> --runtime --json` برای تأیید ثبت‌های زمان اجرا مانند متدهای Gateway و فرمان‌های CLI استفاده کنید. اگر Plugin یک ریشه CLI را با `api.registerCli` ثبت کرده است، آن فرمان را مستقیماً از طریق CLI ریشه OpenClaw اجرا کنید؛ برای مثال `openclaw demo-plugin ping`.

  </Accordion>
  <Accordion title="آرشیوها">
    آرشیوهای پشتیبانی‌شده: `.zip`، `.tgz`، `.tar.gz`، `.tar`. آرشیوهای بومی Plugin در OpenClaw باید در ریشه استخراج‌شده Plugin دارای یک `openclaw.plugin.json` معتبر باشند؛ آرشیوهایی که فقط شامل `package.json` هستند، پیش از آنکه OpenClaw سوابق نصب را بنویسد رد می‌شوند.

    هنگامی که فایل یک tarball حاصل از npm-pack است و می‌خواهید
    از همان مسیر پروژه مدیریت‌شده npm به‌ازای هر Plugin که نصب‌های رجیستری استفاده می‌کنند بهره ببرید، از `npm-pack:<path.tgz>` استفاده کنید؛
    از جمله تأیید `package-lock.json`، اسکن وابستگی‌های بالاکشیده‌شده
    و سوابق نصب npm. مسیرهای ساده آرشیو همچنان به‌عنوان آرشیوهای محلی
    در ریشه افزونه‌های Plugin نصب می‌شوند.

    نصب‌های marketplace متعلق به Claude نیز پشتیبانی می‌شوند.

  </Accordion>
</AccordionGroup>

نصب‌های ClawHub از یک مکان‌یاب صریح `clawhub:<package>` استفاده می‌کنند:

```bash
openclaw plugins install clawhub:openclaw-codex-app-server
openclaw plugins install clawhub:openclaw-codex-app-server@1.2.3
```

مشخصه‌های ساده و معتبر npm برای Plugin، هنگام گذار راه‌اندازی به‌طور پیش‌فرض از npm نصب می‌شوند، مگر اینکه با شناسه رسمی Plugin مطابقت داشته باشند:

```bash
openclaw plugins install openclaw-codex-app-server
```

برای صریح‌کردن وضوح‌دهی صرفاً از طریق npm، از `npm:` استفاده کنید:

```bash
openclaw plugins install npm:openclaw-codex-app-server
openclaw plugins install npm:@openclaw/discord@2026.5.20
openclaw plugins install npm:@scope/plugin-name@1.0.1
```

OpenClaw پیش از نصب، سازگاری اعلام‌شده API مربوط به Plugin / حداقل Gateway را بررسی می‌کند. هنگامی که نسخه انتخاب‌شده ClawHub یک مصنوع ClawPack منتشر کرده باشد، OpenClaw فایل نسخه‌دار npm-pack با نام `.tgz` را دانلود می‌کند، هدر digest مربوط به ClawHub و digest مصنوع را تأیید می‌کند و سپس آن را از طریق مسیر معمول آرشیو نصب می‌کند. نسخه‌های قدیمی‌تر ClawHub بدون فراداده ClawPack همچنان از طریق مسیر قدیمی تأیید آرشیو بسته نصب می‌شوند. نصب‌های ثبت‌شده، فراداده منبع ClawHub، نوع مصنوع، یکپارچگی npm، shasum مربوط به npm، نام tarball و اطلاعات digest مربوط به ClawPack را برای به‌روزرسانی‌های بعدی نگه می‌دارند.
نصب‌های ClawHub بدون نسخه، یک مشخصه ثبت‌شده بدون نسخه نگه می‌دارند تا `openclaw plugins update` بتواند انتشارهای جدیدتر ClawHub را دنبال کند؛ انتخاب‌گرهای صریح نسخه یا برچسب مانند `clawhub:pkg@1.2.3` و `clawhub:pkg@beta` روی همان انتخاب‌گر پین باقی می‌مانند.

### شکل کوتاه marketplace

هنگامی که نام marketplace در کش محلی رجیستری Claude در `~/.claude/plugins/known_marketplaces.json` وجود دارد، از شکل کوتاه `plugin@marketplace` استفاده کنید:

```bash
openclaw plugins marketplace list <marketplace-name>
openclaw plugins install <plugin-name>@<marketplace-name>
```

برای ارسال صریح منبع marketplace از `--marketplace` استفاده کنید:

```bash
openclaw plugins install <plugin-name> --marketplace <marketplace-name>
openclaw plugins install <plugin-name> --marketplace <owner/repo>
openclaw plugins install <plugin-name> --marketplace https://github.com/<owner>/<repo>
openclaw plugins install <plugin-name> --marketplace ./my-marketplace
```

<Tabs>
  <Tab title="منابع marketplace">
    - یک نام marketplace شناخته‌شده Claude از `~/.claude/plugins/known_marketplaces.json`
    - یک ریشه marketplace محلی یا مسیر `marketplace.json`
    - شکل کوتاه مخزن GitHub مانند `owner/repo`
    - URL مخزن GitHub مانند `https://github.com/owner/repo`
    - یک URL مربوط به git

  </Tab>
  <Tab title="قواعد marketplace راه‌دور">
    برای marketplaceهای راه‌دور که از GitHub یا git بارگذاری می‌شوند، ورودی‌های Plugin باید درون مخزن cloneشده marketplace باقی بمانند. OpenClaw منابع مسیر نسبی را از آن مخزن می‌پذیرد و منابع HTTP(S)، مسیر مطلق، git، GitHub و سایر منابع غیرمسیر Plugin را از manifestهای راه‌دور رد می‌کند.
  </Tab>
</Tabs>

برای مسیرها و آرشیوهای محلی، OpenClaw موارد زیر را به‌طور خودکار تشخیص می‌دهد:

- Pluginهای بومی OpenClaw (`openclaw.plugin.json`)
- بسته‌های سازگار با Codex (`.codex-plugin/plugin.json`)
- بسته‌های سازگار با Claude (`.claude-plugin/plugin.json`، یا چیدمان پیش‌فرض مؤلفه Claude هنگامی که آن فایل manifest وجود ندارد)
- بسته‌های سازگار با Cursor (`.cursor-plugin/plugin.json`)

نصب‌های محلی مدیریت‌شده باید دایرکتوری‌ها یا آرشیوهای Plugin باشند. فایل‌های مستقل Plugin از نوع `.js`،
`.mjs`، `.cjs` و `.ts` توسط `plugins install` در ریشه مدیریت‌شده Plugin
کپی نمی‌شوند و با قرارگرفتن مستقیم در
`~/.openclaw/extensions` یا `<workspace>/.openclaw/extensions` نیز بارگذاری نمی‌شوند؛ آن
ریشه‌های کشف خودکار، دایرکتوری‌های بسته یا bundle مربوط به Plugin را بارگذاری می‌کنند و
از فایل‌های اسکریپت سطح بالا به‌عنوان ابزارهای کمکی محلی صرف‌نظر می‌کنند. در عوض، فایل‌های مستقل را صریحاً در
`plugins.load.paths` فهرست کنید.

<Note>
بسته‌های سازگار در ریشه معمول Plugin نصب می‌شوند و در همان جریان فهرست/اطلاعات/فعال‌سازی/غیرفعال‌سازی شرکت می‌کنند. در حال حاضر، Skills موجود در bundle، command-skillهای Claude، پیش‌فرض‌های `settings.json` متعلق به Claude، پیش‌فرض‌های `.lsp.json` متعلق به Claude / `lspServers` اعلام‌شده در manifest، command-skillهای Cursor و دایرکتوری‌های هوک سازگار با Codex پشتیبانی می‌شوند؛ سایر قابلیت‌های bundle تشخیص‌داده‌شده در عیب‌یابی/اطلاعات نمایش داده می‌شوند، اما هنوز به اجرای زمان اجرا متصل نشده‌اند.
</Note>

برای اشاره به یک دایرکتوری محلی Plugin بدون کپی‌کردن آن، از `-l`/`--link` استفاده کنید (آن را
به `plugins.load.paths` اضافه می‌کند):

```bash
openclaw plugins install -l ./my-plugin
```

`--link` با نصب‌های `--marketplace` یا `git:` پشتیبانی نمی‌شود و
به یک مسیر محلی ازپیش‌موجود نیاز دارد. برای یک پیوند محلی غیرتعاملی،
پس از بررسی منبع، `--force` را ارسال کنید؛ این گزینه منشأ را تأیید می‌کند، اما
دایرکتوری پیوندشده را کپی یا بازنویسی نمی‌کند.

<Note>
Pluginهای با منشأ workspace که از ریشه افزونه‌های workspace کشف می‌شوند، تا زمانی که
صریحاً فعال نشوند وارد یا اجرا نمی‌شوند. برای توسعه محلی،
`openclaw plugins enable <plugin-id>` را اجرا کنید یا
`plugins.entries.<plugin-id>.enabled: true` را تنظیم کنید؛ اگر پیکربندی شما از
`plugins.allow` استفاده می‌کند، همان شناسه Plugin را نیز در آن بگنجانید. این قاعده fail-closed
همچنین هنگامی اعمال می‌شود که راه‌اندازی کانال، صریحاً یک Plugin با منشأ workspace را برای
بارگذاری صرفاً جهت راه‌اندازی هدف قرار دهد؛ بنابراین تا زمانی که آن
Plugin فضای کاری غیرفعال یا از allowlist کنار گذاشته شده باشد، کد راه‌اندازی Plugin کانال محلی اجرا نخواهد شد. نصب‌های پیوندشده
و ورودی‌های صریح `plugins.load.paths` از خط‌مشی معمول برای
منشأ حل‌شده Plugin خود پیروی می‌کنند. به
[پیکربندی خط‌مشی Plugin](/fa/tools/plugin#configure-plugin-policy)
و [مرجع پیکربندی](/fa/gateway/configuration-reference#plugins) مراجعه کنید.

برای ذخیره مشخصه دقیق حل‌شده (`name@version`) در نمایه مدیریت‌شده Plugin، در حالی که رفتار پیش‌فرض بدون پین باقی می‌ماند، در نصب‌های npm از `--pin` استفاده کنید.
</Note>

## فهرست

```bash
openclaw plugins list
openclaw plugins list --enabled
openclaw plugins list --verbose
openclaw plugins list --json
```

<ParamField path="--enabled" type="boolean">
  فقط Pluginهای فعال را نمایش می‌دهد.
</ParamField>
<ParamField path="--verbose" type="boolean">
  نمای جدول را به خطوط جزئیات جداگانه برای هر Plugin با فرادادهٔ قالب/منبع/خاستگاه/نسخه/فعال‌سازی تغییر می‌دهد.
</ParamField>
<ParamField path="--json" type="boolean">
  فهرست قابل‌خواندن برای ماشین، به‌همراه عیب‌یابی رجیستری و وضعیت نصب وابستگی‌های بسته.
</ParamField>

<Note>
`plugins list` ابتدا رجیستری محلی و ماندگار Plugin را می‌خواند و اگر رجیستری وجود نداشته باشد یا نامعتبر باشد، از یک جایگزین مشتق‌شده صرفاً از مانیفست استفاده می‌کند. این فرمان برای بررسی نصب‌بودن، فعال‌بودن و قابل‌مشاهده‌بودن یک Plugin در برنامه‌ریزی راه‌اندازی سرد مفید است، اما کاوش زندهٔ زمان‌اجرای یک فرایند Gateway درحال‌اجرا نیست. پس از تغییر کد Plugin، وضعیت فعال‌سازی، خط‌مشی هوک یا `plugins.load.paths`، پیش از انتظار برای اجرای کد یا هوک‌های جدید `register(api)`، Gateway ارائه‌دهندهٔ کانال را بازراه‌اندازی کنید. در استقرارهای راه‌دور/کانتینری، مطمئن شوید فرزند واقعی `openclaw gateway run` را بازراه‌اندازی می‌کنید، نه فقط یک فرایند پوشاننده را.

`plugins list --json` شامل `dependencyStatus` هر Plugin از `package.json`
`dependencies` و `optionalDependencies` است. OpenClaw بررسی می‌کند که آیا نام آن بسته‌ها
در مسیر معمول جست‌وجوی Node برای `node_modules` مربوط به Plugin وجود دارند یا نه؛
کد زمان‌اجرای Plugin را وارد نمی‌کند، مدیر بسته‌ای را اجرا نمی‌کند و وابستگی‌های
مفقود را ترمیم نمی‌کند.
</Note>

اگر گزارش‌های راه‌اندازی `plugins.allow is empty; discovered non-bundled plugins may auto-load: ...` را ثبت کردند،
`openclaw plugins list --enabled --verbose` یا
`openclaw plugins inspect <id>` را با شناسهٔ یکی از Pluginهای فهرست‌شده اجرا کنید تا شناسه‌های Plugin
تأیید شوند و شناسه‌های مورداعتماد را در `plugins.allow` در `openclaw.json` کپی کنید. هنگامی که
هشدار بتواند همهٔ Pluginهای کشف‌شده را فهرست کند، یک قطعهٔ آمادهٔ جای‌گذاری
`plugins.allow` را چاپ می‌کند که از قبل شامل آن شناسه‌ها است. اگر یک Plugin
بدون منشأ نصب/مسیر بارگذاری، بارگذاری شد، آن شناسهٔ Plugin را بررسی کنید و سپس یا
شناسهٔ مورداعتماد را در `plugins.allow` تثبیت کنید یا Plugin را از یک منبع مورداعتماد
دوباره نصب کنید تا OpenClaw منشأ نصب را ثبت کند.

برای کار روی Plugin همراه درون یک تصویر بسته‌بندی‌شدهٔ Docker، دایرکتوری
منبع Plugin را به‌صورت bind mount روی مسیر منبع بسته‌بندی‌شدهٔ متناظر سوار کنید، مانند
`/app/extensions/synology-chat`. OpenClaw این هم‌پوشانی منبعِ سوارشده را
پیش از `/app/dist/extensions/synology-chat` کشف می‌کند؛ یک دایرکتوری منبع که صرفاً کپی شده باشد
غیرفعال می‌ماند، بنابراین نصب‌های بسته‌بندی‌شدهٔ عادی همچنان از dist کامپایل‌شده استفاده می‌کنند.

برای اشکال‌زدایی هوک‌های زمان‌اجرا:

- `openclaw plugins inspect <id> --runtime --json` هوک‌های ثبت‌شده و عیب‌یابی‌ها را از یک گذر بازرسیِ همراه با بارگذاری ماژول نمایش می‌دهد. بازرسی زمان‌اجرا هرگز وابستگی نصب نمی‌کند؛ برای پاک‌سازی وضعیت وابستگی قدیمی یا بازیابی Pluginهای قابل‌دریافتِ مفقودی که پیکربندی به آن‌ها ارجاع می‌دهد، از `openclaw doctor --fix` استفاده کنید.
- `openclaw gateway status --deep --require-rpc` نشانی اینترنتی/پروفایل قابل‌دسترسی Gateway، راهنمایی‌های سرویس/فرایند، مسیر پیکربندی و سلامت RPC را تأیید می‌کند.
- هوک‌های مکالمهٔ غیرهمراه (`llm_input`، `llm_output`، `before_model_resolve`، `before_agent_reply`، `before_agent_run`، `before_agent_finalize`، `agent_end`) به `plugins.entries.<id>.hooks.allowConversationAccess=true` نیاز دارند.

### نمایهٔ Plugin

فرادادهٔ نصب Plugin وضعیتی تحت مدیریت ماشین است، نه پیکربندی کاربر. نصب‌ها و به‌روزرسانی‌ها آن را در پایگاه‌دادهٔ مشترک SQLite زیر دایرکتوری وضعیت فعال OpenClaw می‌نویسند. ردیف `installed_plugin_index` فرادادهٔ ماندگار `installRecords`، از جمله رکوردهای مانیفست‌های خراب یا مفقود Plugin، و همچنین یک کش رجیستری سرد مشتق‌شده از مانیفست را ذخیره می‌کند که `openclaw plugins update`، حذف نصب، عیب‌یابی‌ها و رجیستری سرد Plugin از آن استفاده می‌کنند.

`plugins.installs` یک سطح پیکربندی تألیفی بازنشسته است. زمان‌اجرا و فرمان‌های به‌روزرسانی فقط نمایهٔ Pluginهای نصب‌شده در SQLite را می‌خوانند. پیش از استفادهٔ عادی زمان‌اجرا، `openclaw doctor --fix` را اجرا کنید تا رکوردهای پیکربندی قدیمی به نمایه وارد و کلید بازنشسته حذف شود.

## حذف نصب

```bash
openclaw plugins uninstall <id>
openclaw plugins uninstall <id> --dry-run
openclaw plugins uninstall <id> --keep-files
openclaw plugins uninstall <id> --force
```

`uninstall` رکوردهای Plugin را از `plugins.entries`، نمایهٔ ماندگار Plugin، ورودی‌های فهرست مجاز/غیرمجاز Plugin و در صورت کاربرد، ورودی‌های پیوندخوردهٔ `plugins.load.paths` حذف می‌کند. مگر اینکه `--keep-files` تنظیم شده باشد، حذف نصب دایرکتوری نصب مدیریت‌شدهٔ ردیابی‌شده را نیز حذف می‌کند، اما فقط زمانی که مسیر نهایی آن داخل ریشهٔ افزونه‌های Plugin در OpenClaw باشد. اگر Plugin درحال‌حاضر مالک جایگاه `memory` یا `contextEngine` باشد، آن جایگاه به مقدار پیش‌فرض خود بازنشانی می‌شود (`memory-core` برای حافظه، `legacy` برای موتور زمینه).

`uninstall` پیش‌نمایشی از مواردی که حذف خواهند شد چاپ می‌کند و سپس پیش از اعمال تغییرات، `Uninstall plugin "<id>"?` را درخواست می‌کند. برای ردکردن درخواست تأیید، `--force` را ارسال کنید (برای اسکریپت‌ها و اجراهای غیرتعاملی مفید است)؛ بدون آن، حذف نصب به یک TTY تعاملی نیاز دارد. `--dry-run` همان پیش‌نمایش را چاپ می‌کند و بدون درخواست تأیید یا اعمال هیچ تغییری خارج می‌شود.

<Note>
`--keep-config` به‌عنوان نام مستعار منسوخ‌شدهٔ `--keep-files` پشتیبانی می‌شود.
</Note>

## به‌روزرسانی

```bash
openclaw plugins update <id-or-npm-spec>
openclaw plugins update --all
openclaw plugins update <id-or-npm-spec> --dry-run
openclaw plugins update @openclaw/voice-call
openclaw plugins update @acme/demo
openclaw plugins update openclaw-codex-app-server --acknowledge-clawhub-risk
openclaw plugins update openclaw-codex-app-server --dangerously-force-unsafe-install
```

به‌روزرسانی‌ها روی نصب‌های ردیابی‌شدهٔ Plugin در نمایهٔ مدیریت‌شدهٔ Plugin و نصب‌های ردیابی‌شدهٔ بستهٔ هوک در وضعیت مشترک SQLite اعمال می‌شوند. آن‌ها از همان منبعی استفاده می‌کنند که کاربر هنگام نصب Plugin انتخاب کرده است، بنابراین به تأیید دوبارهٔ منبع نیاز ندارند.

<AccordionGroup>
  <Accordion title="تفکیک شناسهٔ Plugin از مشخصهٔ npm">
    وقتی یک شناسهٔ Plugin ارائه می‌کنید، OpenClaw از مشخصهٔ نصب ثبت‌شده برای آن Plugin دوباره استفاده می‌کند. یعنی dist-tagهای ازپیش ذخیره‌شده مانند `@beta` و نسخه‌های دقیق تثبیت‌شده همچنان در اجراهای بعدی `update <id>` استفاده می‌شوند.

    هنگام `update <id> --dry-run`، نصب‌های npm با نسخهٔ دقیق تثبیت‌شده، تثبیت‌شده باقی می‌مانند. اگر OpenClaw بتواند خط پیش‌فرض رجیستری بسته را نیز تفکیک کند و آن خط پیش‌فرض از نسخهٔ تثبیت‌شدهٔ نصب‌شده جدیدتر باشد، اجرای آزمایشی تثبیت را گزارش می‌کند و فرمان صریح به‌روزرسانی بستهٔ `@latest` را برای پیروی از خط پیش‌فرض رجیستری چاپ می‌کند.

    این قاعدهٔ به‌روزرسانی هدفمند با مسیر نگه‌داری انبوه `openclaw plugins update --all` متفاوت است. به‌روزرسانی‌های انبوه همچنان مشخصه‌های عادی نصبِ ردیابی‌شده را رعایت می‌کنند، اما رکوردهای مورداعتماد Plugin رسمی OpenClaw می‌توانند به‌جای ماندن روی یک بستهٔ رسمی دقیق و قدیمی، با هدف فعلی کاتالوگ رسمی همگام شوند. وقتی عمداً می‌خواهید یک مشخصهٔ رسمی دقیق یا برچسب‌خورده دست‌نخورده بماند، از `update <id>` هدفمند استفاده کنید.

    برای نصب‌های npm، می‌توانید یک مشخصهٔ صریح بستهٔ npm همراه با dist-tag یا نسخهٔ دقیق نیز ارائه کنید. OpenClaw نام آن بسته را به رکورد ردیابی‌شدهٔ Plugin مرتبط می‌کند، آن Plugin نصب‌شده را به‌روزرسانی می‌کند و مشخصهٔ جدید npm را برای به‌روزرسانی‌های شناسه‌محور آینده ثبت می‌کند.

    ارائهٔ نام بستهٔ npm بدون نسخه یا برچسب نیز آن را به رکورد ردیابی‌شدهٔ Plugin مرتبط می‌کند. هنگامی از این روش استفاده کنید که یک Plugin روی نسخه‌ای دقیق تثبیت شده است و می‌خواهید آن را به خط انتشار پیش‌فرض رجیستری بازگردانید.

  </Accordion>
  <Accordion title="به‌روزرسانی‌های کانال بتا">
    `openclaw plugins update <id-or-npm-spec>` هدفمند از مشخصهٔ ردیابی‌شدهٔ Plugin دوباره استفاده می‌کند، مگر اینکه مشخصهٔ جدیدی ارائه کنید. `openclaw plugins update --all` انبوه هنگام همگام‌سازی رکوردهای مورداعتماد Plugin رسمی با هدف کاتالوگ رسمی، از `update.channel` پیکربندی‌شده استفاده می‌کند؛ بنابراین نصب‌های کانال بتا می‌توانند به‌جای عادی‌سازی بی‌سروصدا به stable/latest، در خط انتشار بتا باقی بمانند.

    `openclaw update` کانال به‌روزرسانی فعال OpenClaw را نیز می‌شناسد: در کانال بتا، رکوردهای Plugin مربوط به npm خط پیش‌فرض و ClawHub ابتدا `@beta` را امتحان می‌کنند. اگر انتشار بتایی برای Plugin وجود نداشته باشد، به مشخصهٔ ثبت‌شدهٔ default/latest بازمی‌گردند؛ Pluginهای npm همچنین زمانی بازمی‌گردند که بستهٔ بتا وجود دارد اما اعتبارسنجی نصب آن ناموفق است. این بازگشت به‌صورت هشدار گزارش می‌شود و به‌روزرسانی هسته را ناموفق نمی‌کند. نسخه‌های دقیق و برچسب‌های صریح برای به‌روزرسانی‌های هدفمند روی همان انتخابگر تثبیت می‌مانند.

  </Accordion>
  <Accordion title="بررسی نسخه و انحراف یکپارچگی">
    پیش از به‌روزرسانی زندهٔ npm، OpenClaw نسخهٔ بستهٔ نصب‌شده را با فرادادهٔ رجیستری npm مقایسه می‌کند. اگر نسخهٔ نصب‌شده و هویت ثبت‌شدهٔ مصنوع از قبل با هدف تفکیک‌شده مطابقت داشته باشند، به‌روزرسانی بدون دریافت، نصب دوباره یا بازنویسی `openclaw.json` نادیده گرفته می‌شود.

    وقتی هش یکپارچگی ذخیره‌شده وجود داشته باشد و هش مصنوع دریافت‌شده تغییر کند، OpenClaw آن را انحراف مصنوع npm تلقی می‌کند. فرمان تعاملی `openclaw plugins update` هش‌های موردانتظار و واقعی را چاپ می‌کند و پیش از ادامه تأیید می‌خواهد. ابزارهای کمکی به‌روزرسانی غیرتعاملی، مگر اینکه فراخواننده خط‌مشی صریحی برای ادامه ارائه کند، به‌صورت بسته و همراه با خطا متوقف می‌شوند.

  </Accordion>
  <Accordion title="--dangerously-force-unsafe-install در به‌روزرسانی">
    برای سازگاری، `--dangerously-force-unsafe-install` در `plugins update` نیز پذیرفته می‌شود، اما منسوخ شده و دیگر رفتار به‌روزرسانی Plugin را تغییر نمی‌دهد. `security.installPolicy` اپراتور همچنان می‌تواند به‌روزرسانی‌ها را مسدود کند؛ هوک‌های `before_install` مربوط به Plugin فقط در فرایندهایی اعمال می‌شوند که هوک‌های Plugin در آن‌ها بارگذاری شده باشند.
  </Accordion>
  <Accordion title="--acknowledge-clawhub-risk در به‌روزرسانی">
    به‌روزرسانی Pluginهای جامعه که از ClawHub پشتیبانی می‌شوند، پیش از دریافت بستهٔ جایگزین همان بررسی اعتماد به انتشار دقیق را که برای نصب انجام می‌شود اجرا می‌کنند. برای خودکارسازی بازبینی‌شده‌ای که باید هنگام وجود هشدار اعتماد پرخطر در انتشار انتخاب‌شدهٔ ClawHub ادامه یابد، از `--acknowledge-clawhub-risk` استفاده کنید. بسته‌های رسمی ClawHub و منابع همراه Plugin در OpenClaw این درخواست تأیید اعتماد به انتشار را رد می‌کنند.
  </Accordion>
</AccordionGroup>

## بازرسی

```bash
openclaw plugins inspect <id>
openclaw plugins inspect <id> --runtime
openclaw plugins inspect <id> --json
openclaw plugins inspect --all
```

بازرسی، هویت، وضعیت بارگذاری، منبع، قابلیت‌های مانیفست، پرچم‌های خط‌مشی، عیب‌یابی‌ها، فرادادهٔ نصب، قابلیت‌های بسته و هرگونه پشتیبانی شناسایی‌شده از سرور MCP یا LSP را بدون واردکردن زمان‌اجرای Plugin به‌صورت پیش‌فرض نمایش می‌دهد. خروجی JSON شامل قراردادهای مانیفست Plugin مانند `contracts.agentToolResultMiddleware` و `contracts.trustedToolPolicies` است تا اپراتورها بتوانند اعلان‌های سطح مورداعتماد را پیش از فعال‌سازی یا بازراه‌اندازی یک Plugin ممیزی کنند. برای بارگذاری ماژول Plugin و گنجاندن هوک‌ها، ابزارها، فرمان‌ها، سرویس‌ها، روش‌های Gateway و مسیرهای HTTP ثبت‌شده، `--runtime` را اضافه کنید. بازرسی زمان‌اجرا وابستگی‌های مفقود Plugin را مستقیماً گزارش می‌کند؛ نصب‌ها و ترمیم‌ها در `openclaw plugins install`، `openclaw plugins update` و `openclaw doctor --fix` باقی می‌مانند.

فرمان‌های CLI متعلق به Plugin معمولاً به‌صورت گروه‌های فرمان ریشهٔ `openclaw` نصب می‌شوند، اما Pluginها می‌توانند فرمان‌های تودرتو را نیز زیر یک والد هسته مانند `openclaw nodes` ثبت کنند. پس از اینکه `inspect --runtime` فرمانی را زیر `cliCommands` نمایش داد، آن را در مسیر فهرست‌شده اجرا کنید؛ برای نمونه، Pluginی که `demo-git` را ثبت می‌کند با `openclaw demo-git ping` قابل‌تأیید است.

هر Plugin بر اساس آنچه واقعاً در زمان‌اجرا ثبت می‌کند طبقه‌بندی می‌شود:

| ساختار               | معنا                                                           |
| ------------------- | ----------------------------------------------------------------- |
| `plain-capability`  | دقیقاً یک نوع قابلیت (برای نمونه، Plugin صرفاً ارائه‌دهنده)         |
| `hybrid-capability` | بیش از یک نوع قابلیت (برای نمونه، متن + گفتار + تصویر)       |
| `hook-only`         | فقط هوک، بدون قابلیت، ابزار، فرمان، سرویس یا مسیر |
| `non-capability`    | ابزار/فرمان/سرویس، اما بدون قابلیت                       |

برای اطلاعات بیشتر دربارهٔ مدل قابلیت، به [ساختارهای Plugin](/fa/plugins/architecture#plugin-shapes) مراجعه کنید.

<Note>
پرچم `--json` گزارشی قابل‌خواندن برای ماشین و مناسب اسکریپت‌نویسی و ممیزی تولید می‌کند. `inspect --all` جدولی در سطح کل ناوگان با ستون‌های ساختار، انواع قابلیت، اعلان‌های سازگاری، قابلیت‌های بسته و خلاصهٔ هوک رندر می‌کند. `info` نام مستعاری برای `inspect` است.
</Note>

## Doctor

```bash
openclaw plugins doctor
```

`doctor` خطاهای بارگذاری Plugin، عیب‌یابی‌های مانیفست/کشف، اعلان‌های سازگاری و ارجاع‌های منسوخ پیکربندی Plugin، مانند جایگاه‌های مفقود Plugin، را گزارش می‌کند. هنگامی که درخت نصب و پیکربندی Plugin پاک باشند، `No plugin issues detected.` را چاپ می‌کند. اگر پیکربندی منسوخ باقی مانده باشد اما درخت نصب از جهات دیگر سالم باشد، خلاصه به‌جای القای سلامت کامل Plugin، این موضوع را بیان می‌کند.

اگر یک Plugin پیکربندی‌شده روی دیسک موجود باشد اما بررسی‌های ایمنی مسیر بارگذار آن را مسدود کنند، اعتبارسنجی پیکربندی ورودی Plugin را حفظ می‌کند و آن را به‌صورت `present but blocked` گزارش می‌دهد. به‌جای حذف پیکربندی `plugins.entries.<id>` یا `plugins.allow`، عیب‌یابی Plugin مسدودشده پیشین، مانند مالکیت مسیر یا مجوزهای قابل‌نوشتن برای همه، را برطرف کنید.

برای خرابی‌های شکل ماژول، مانند نبودن خروجی‌های `register`/`activate`، دوباره با `OPENCLAW_PLUGIN_LOAD_DEBUG=1` اجرا کنید تا خلاصه‌ای فشرده از شکل خروجی‌ها در خروجی عیب‌یابی گنجانده شود.

## رجیستری

```bash
openclaw plugins registry
openclaw plugins registry --refresh
openclaw plugins registry --json
```

رجیستری محلی Plugin، مدل خواندن سرد و ماندگار OpenClaw برای هویت Pluginهای نصب‌شده، فعال‌سازی، فراداده مبدأ و مالکیت مشارکت‌ها است. راه‌اندازی عادی، جست‌وجوی مالک ارائه‌دهنده، دسته‌بندی راه‌اندازی کانال و موجودی Plugin می‌توانند بدون واردکردن ماژول‌های زمان اجرای Plugin آن را بخوانند.

از `plugins registry` برای بررسی موجود، به‌روز یا منسوخ بودن رجیستری ماندگار استفاده کنید. از `--refresh` برای بازسازی آن از نمایه ماندگار Plugin، خط‌مشی پیکربندی و فراداده مانیفست/بسته استفاده کنید. این یک مسیر ترمیم است، نه مسیر فعال‌سازی زمان اجرا.

`openclaw doctor --fix` همچنین ناهماهنگی مدیریت‌شده npm در مجاورت رجیستری را ترمیم می‌کند. اگر یک بسته `@openclaw/*` رهاشده یا بازیابی‌شده در یک پروژه مدیریت‌شده npm مربوط به Plugin یا ریشه تخت و قدیمی npm مدیریت‌شده، یک Plugin همراه را تحت‌الشعاع قرار دهد، Doctor آن بسته منسوخ را حذف و رجیستری را بازسازی می‌کند تا راه‌اندازی بر اساس مانیفست همراه اعتبارسنجی شود. هنگامی که یک رکورد نصب معتبر یکی از نسل‌های مدیریت‌شده را انتخاب می‌کند اما دایرکتوری‌های تخت یا نسلی قدیمی‌تر باقی می‌مانند، Doctor آن درخت‌های منسوخ را برای پاک‌سازی پس از راه‌اندازی مجدد Gateway بازنشسته می‌کند. Doctor همچنین بسته میزبان `openclaw` را دوباره به Pluginهای مدیریت‌شده npm که `peerDependencies.openclaw` را اعلام می‌کنند پیوند می‌دهد تا واردکردن‌های زمان اجرای محلی بسته، مانند `openclaw/plugin-sdk/*`، پس از به‌روزرسانی‌ها یا ترمیم‌های npm قابل تفکیک باشند.

## بازارچه

```bash
openclaw plugins marketplace entries
openclaw plugins marketplace entries --offline
openclaw plugins marketplace entries --json
openclaw plugins marketplace entries --feed-profile <name>
openclaw plugins marketplace entries --feed-url <url>
openclaw plugins marketplace list <source>
openclaw plugins marketplace list <source> --json
openclaw plugins marketplace refresh
openclaw plugins marketplace refresh --feed-profile <name>
openclaw plugins marketplace refresh --feed-url <url>
openclaw plugins marketplace refresh --expected-sha256 <sha256> --json
```

`plugins marketplace entries` ورودی‌های خوراک پیکربندی‌شده بازارچه OpenClaw را فهرست می‌کند. به‌طور پیش‌فرض، خوراک میزبانی‌شده را امتحان می‌کند و در صورت ناموفق بودن، از جدیدترین تصویر لحظه‌ای پذیرفته‌شده یا داده‌های همراه استفاده می‌کند. از `--feed-profile <name>` برای خواندن یک نمایه پیکربندی‌شده مشخص، از `--feed-url <url>` برای خواندن یک URL صریح خوراک میزبانی‌شده و از `--offline` برای خواندن جدیدترین تصویر لحظه‌ای پذیرفته‌شده بدون واکشی خوراک استفاده کنید.

`plugins marketplace refresh` تصویر لحظه‌ای خوراک میزبانی‌شده پیکربندی‌شده را تازه‌سازی می‌کند و گزارش می‌دهد که OpenClaw داده‌های میزبانی‌شده، یک تصویر لحظه‌ای میزبانی‌شده یا داده‌های جایگزین همراه را پذیرفته است. هنگامی که فراخواننده می‌خواهد فرمان شکست بخورد، مگر اینکه یک محتوای میزبانی‌شده تازه با جمع‌آزمایی سنجاق‌شده مطابقت داشته باشد، از `--expected-sha256` استفاده کنید.

`list` بازارچه یک مسیر محلی بازارچه، یک مسیر `marketplace.json`، یک اختصار GitHub مانند `owner/repo`، یک URL مخزن GitHub یا یک URL گیت را می‌پذیرد. `--json` برچسب مبدأ تفکیک‌شده را به‌همراه مانیفست تجزیه‌شده بازارچه و ورودی‌های Plugin چاپ می‌کند.

تازه‌سازی بازارچه یک خوراک میزبانی‌شده بازارچه OpenClaw را بارگذاری می‌کند و پاسخ
اعتبارسنجی‌شده را به‌عنوان تصویر لحظه‌ای محلی خوراک میزبانی‌شده ماندگار می‌کند. بدون گزینه، از
نمایه خوراک پیش‌فرض پیکربندی‌شده استفاده می‌کند. از `--feed-profile <name>` برای تازه‌سازی یک
نمایه پیکربندی‌شده مشخص، از `--feed-url <url>` برای تازه‌سازی یک URL صریح خوراک
میزبانی‌شده، از `--expected-sha256 <sha256>` برای الزام به تطبیق جمع‌آزمایی محتوا
(`sha256:<hex>` یا یک چکیده هگزادسیمال ساده 64 نویسه‌ای) و از `--json` برای
خروجی قابل‌خواندن توسط ماشین استفاده کنید. URLهای صریح خوراک میزبانی‌شده نباید شامل
اطلاعات اصالت‌سنجی، رشته‌های پرس‌وجو یا قطعه‌ها باشند. تازه‌سازی‌های سنجاق‌نشده می‌توانند بدون
شکست فرمان، نتیجه یک تصویر لحظه‌ای میزبانی‌شده یا جایگزین همراه را گزارش کنند. تازه‌سازی‌های
سنجاق‌شده شکست می‌خورند، مگر اینکه یک محتوای میزبانی‌شده تازه را بپذیرند، و تازه‌سازی‌های موفق
میزبانی‌شده نیز اگر OpenClaw نتواند تصویر لحظه‌ای اعتبارسنجی‌شده را ماندگار کند، شکست می‌خورند.

نمایه داخلی `clawhub-public` انتظار هویت محتوای
`clawhub-official` را دارد. پس از آنکه ClawHub کلید عمومی عملیاتی خود را تولید و تحویل
داد، OpenClaw آن را همراه خواهد کرد. تا آن زمان، نمایه داخلی
اختیار نصب از خوراک امضاشده را اعطا نمی‌کند. کلیدهای عمومی باید از یک
نسخه انتشار یا کانال اپراتوری مورداعتماد دریافت شوند، نه از نقطه پایانی کلید در میزبان خوراک.

OpenClaw پوش DSSE را تأیید می‌کند و هنگامی که یک نمایه `feedId` را اعلام کند،
الزام می‌کند شناسه محتوای رمزگشایی‌شده با آن مطابقت داشته باشد. نمایه داخلی `clawhub-public`
همیشه هویت خود را اعلام می‌کند و مانع از آن می‌شود که یک سند معتبر متعلق به خوراکی دیگر
از طریق آن نمایه بازپخش شود.

در طول انتشار مرحله‌ای، نمایه‌های امضاشده سفارشی موجود که `feedId` را حذف کرده‌اند،
تأیید امضا را بدون مقیدسازی هویت محتوا حفظ می‌کنند. نمایه‌های سفارشی
جدید باید `feedId` را اعلام کنند. سطح پیکربندی نمایه خوراک همراه با
فراداده نمایشی موردنیاز Control UI به‌طور جداگانه در حال ارائه است؛ عیب‌یابی
Doctor آن باید از اپراتور بخواهد هویت مفقود را ارائه کند و نباید
آن را از URL خوراک استنتاج کند. این مقیدسازی اعتماد، کلید ریشه بازنشسته‌شده
`marketplaces` را بازنمی‌گرداند.

## مرتبط

- [ساخت Pluginها](/fa/plugins/building-plugins)
- [مرجع CLI](/fa/cli)
- [ClawHub](/clawhub)
