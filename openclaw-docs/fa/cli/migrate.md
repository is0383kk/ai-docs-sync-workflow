---
read_when:
    - می‌خواهید از Hermes یا یک سیستم عامل دیگر به OpenClaw مهاجرت کنید
    - در حال افزودن یک ارائه‌دهنده مهاجرت تحت مالکیت Plugin هستید
summary: مرجع CLI برای `openclaw migrate` (درون‌ریزی وضعیت از یک سامانه عامل دیگر)
title: مهاجرت کنید
x-i18n:
    generated_at: "2026-07-27T16:19:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f492535019f8a69706ff918462ba74cf5d26e733d2e4e9493b3c76bd77f2584d
    source_path: cli/migrate.md
    workflow: 16
---

# `openclaw migrate`

وضعیت را از یک سامانه عامل دیگر، از طریق ارائه‌دهنده مهاجرت متعلق به Plugin وارد کنید. ارائه‌دهندگان همراه، Claude،‏ Codex CLI و [Hermes](/fa/install/migrating-hermes) را پوشش می‌دهند؛ Pluginها می‌توانند ارائه‌دهندگان دیگری را ثبت کنند.

<Tip>
برای راهنماهای گام‌به‌گام مخصوص کاربران، به [مهاجرت از Claude](/fa/install/migrating-claude) و [مهاجرت از Hermes](/fa/install/migrating-hermes) مراجعه کنید. [مرکز مهاجرت](/fa/install/migrating) همه مسیرها را فهرست می‌کند.
</Tip>

## فرمان‌ها

```bash
openclaw migrate list
openclaw migrate claude --dry-run
openclaw migrate codex --dry-run
openclaw migrate codex --skill gog-vault77-google-workspace
openclaw migrate codex --plugin google-calendar --dry-run
openclaw migrate codex --plugin google-calendar --verify-plugin-apps --dry-run
openclaw migrate hermes --dry-run
openclaw migrate hermes
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --plugin google-calendar
openclaw migrate apply codex --yes
openclaw migrate apply claude --yes
openclaw migrate apply hermes --yes
openclaw migrate apply hermes --include-secrets --yes
openclaw onboard --flow import
openclaw onboard --import-from claude --import-source ~/.claude
openclaw onboard --import-from hermes --import-source ~/.hermes
```

اجرای `openclaw migrate <provider>` بدون هیچ پرچم دیگری، طرح را ایجاد و پیش‌نمایش می‌کند و (در یک TTY) پیش از اعمال، درخواست تأیید می‌دهد. `openclaw migrate plan <provider>` و `openclaw migrate apply <provider>` پیش‌نمایش و اعمال را با پرچم‌های یکسان به زیرفرمان‌های جداگانه تقسیم می‌کنند.

<ParamField path="<provider>" type="string">
  نام یک ارائه‌دهنده مهاجرت ثبت‌شده، برای نمونه `hermes`. برای دیدن ارائه‌دهندگان نصب‌شده، `openclaw migrate list` را اجرا کنید.
</ParamField>
<ParamField path="--dry-run" type="boolean">
  طرح را بسازید و بدون تغییر وضعیت خارج شوید.
</ParamField>
<ParamField path="--from <path>" type="string">
  پوشه وضعیت مبدأ را بازنویسی کنید. Hermes ابتدا از `$HERMES_HOME` و نمایه فعال پیروی می‌کند، سپس از پیش‌فرض پلتفرم (`~/.hermes` یا `%LOCALAPPDATA%\hermes`) استفاده می‌کند. پیش‌فرض Codex،‏ `~/.codex` (یا `$CODEX_HOME`) و پیش‌فرض Claude،‏ `~/.claude` است.
</ParamField>
<ParamField path="--include-secrets" type="boolean">
  اطلاعات اعتبارنامه پشتیبانی‌شده را بدون درخواست تأیید وارد کنید. اعمال تعاملی پیش از واردکردن اعتبارنامه‌های احراز هویت شناسایی‌شده سؤال می‌کند و گزینه بله به‌طور پیش‌فرض انتخاب شده است؛ `--yes` غیرتعاملی برای واردکردن آن‌ها به `--include-secrets` نیاز دارد.
</ParamField>
<ParamField path="--no-auth-credentials" type="boolean">
  از واردکردن اعتبارنامه‌های احراز هویت، از جمله درخواست تأیید تعاملی، صرف‌نظر کنید.
</ParamField>
<ParamField path="--overwrite" type="boolean">
  هنگامی که طرح تداخل گزارش می‌کند، به عملیات اعمال اجازه دهید مقصدهای موجود را جایگزین کند.
</ParamField>
<ParamField path="--yes" type="boolean">
  از درخواست تأیید صرف‌نظر کنید. در حالت غیرتعاملی الزامی است.
</ParamField>
<ParamField path="--skill <name>" type="string">
  یک مورد کپی مهارت را بر اساس نام مهارت یا شناسه مورد انتخاب کنید. برای مهاجرت چند مهارت، پرچم را تکرار کنید. در صورت حذف، مهاجرت‌های تعاملی Codex یک انتخاب‌گر کادر انتخاب نشان می‌دهند و مهاجرت‌های غیرتعاملی همه مهارت‌های برنامه‌ریزی‌شده را نگه می‌دارند.
</ParamField>
<ParamField path="--plugin <name>" type="string">
  یک مورد نصب Plugin مربوط به Codex را بر اساس نام Plugin یا شناسه مورد انتخاب کنید. برای مهاجرت چند Plugin مربوط به Codex، پرچم را تکرار کنید. در صورت حذف، مهاجرت‌های تعاملی Codex یک انتخاب‌گر کادر انتخاب بومی Plugin مربوط به Codex نشان می‌دهند و مهاجرت‌های غیرتعاملی همه Pluginهای برنامه‌ریزی‌شده را نگه می‌دارند. فقط برای Pluginهای Codex نصب‌شده از مبدأ `openai-curated` که موجودی app-server مربوط به Codex آن‌ها را کشف کرده است، اعمال می‌شود.
</ParamField>
<ParamField path="--verify-plugin-apps" type="boolean">
  فقط Codex. پیش از برنامه‌ریزی فعال‌سازی Plugin بومی، پیمایش تازه `app/list` را در app-server مبدأ Codex اجباری می‌کند. برای سریع نگه‌داشتن برنامه‌ریزی مهاجرت، به‌طور پیش‌فرض خاموش است.
</ParamField>
<ParamField path="--backup-output <path>" type="string">
  مسیر یا پوشه بایگانی پشتیبان پیش از مهاجرت. بدون تغییر به `openclaw backup create` ارسال می‌شود.
</ParamField>
<ParamField path="--no-backup" type="boolean">
  از پشتیبان‌گیری پیش از اعمال صرف‌نظر کنید. هنگامی که وضعیت محلی OpenClaw موجود است، به `--force` نیاز دارد.
</ParamField>
<ParamField path="--force" type="boolean">
  هنگامی که عملیات اعمال در غیر این صورت از صرف‌نظرکردن از پشتیبان‌گیری خودداری می‌کند، در کنار `--no-backup` الزامی است.
</ParamField>
<ParamField path="--json" type="boolean">
  طرح یا نتیجه اعمال را به‌صورت JSON چاپ کنید. با `--json` و بدون `--yes`، عملیات اعمال طرح را چاپ می‌کند و وضعیت را تغییر نمی‌دهد.
</ParamField>

## مدل ایمنی

`openclaw migrate` ابتدا پیش‌نمایش را انجام می‌دهد.

<AccordionGroup>
  <Accordion title="پیش‌نمایش پیش از اعمال">
    پیش از هر تغییری، ارائه‌دهنده یک طرح جزءبه‌جزء شامل تداخل‌ها، موارد ردشده و موارد حساس برمی‌گرداند. طرح‌های JSON، خروجی اعمال و گزارش‌های مهاجرت، کلیدهای تودرتویی را که شبیه اسرار هستند، مانند کلیدهای API، توکن‌ها، سرآیندهای مجوز، کوکی‌ها و گذرواژه‌ها، پنهان می‌کنند.

    `openclaw migrate apply <provider>` طرح را پیش‌نمایش می‌کند و، مگر اینکه `--yes` تنظیم شده باشد، پیش از تغییر وضعیت درخواست تأیید می‌دهد. در حالت غیرتعاملی، عملیات اعمال به `--yes` نیاز دارد.

  </Accordion>
  <Accordion title="پشتیبان‌ها">
    عملیات اعمال پیش از اعمال مهاجرت، یک نسخه پشتیبان OpenClaw ایجاد و تأیید می‌کند. اگر هنوز هیچ وضعیت محلی OpenClaw وجود نداشته باشد، مرحله پشتیبان‌گیری رد می‌شود و مهاجرت ادامه می‌یابد. برای صرف‌نظرکردن از پشتیبان‌گیری هنگام وجود وضعیت، هم `--no-backup` و هم `--force` را ارسال کنید.
  </Accordion>
  <Accordion title="تداخل‌ها">
    هنگامی که طرح تداخل دارد، عملیات اعمال از ادامه خودداری می‌کند. طرح را بررسی کنید، سپس اگر جایگزینی مقصدهای موجود عمدی است، دوباره با `--overwrite` اجرا کنید. ارائه‌دهندگان ممکن است همچنان برای فایل‌های بازنویسی‌شده، پشتیبان‌های سطح مورد را در پوشه گزارش مهاجرت بنویسند.
  </Accordion>
  <Accordion title="اسرار">
    عملیات اعمال تعاملی می‌پرسد آیا اعتبارنامه‌های احراز هویت شناسایی‌شده وارد شوند یا نه و گزینه بله به‌طور پیش‌فرض انتخاب شده است. برای صرف‌نظرکردن از آن‌ها از `--no-auth-credentials` استفاده کنید، یا برای واردکردن اعتبارنامه بدون نظارت همراه با `--yes` از `--include-secrets` استفاده کنید.
  </Accordion>
</AccordionGroup>

## ارائه‌دهنده Claude

ارائه‌دهنده همراه Claude به‌طور پیش‌فرض وضعیت Claude Code را در `~/.claude` شناسایی می‌کند. برای واردکردن یک خانه یا ریشه پروژه مشخص Claude Code از `--from <path>` استفاده کنید.

<Tip>
برای راهنمای گام‌به‌گام مخصوص کاربران، به [مهاجرت از Claude](/fa/install/migrating-claude) مراجعه کنید.
</Tip>

### آنچه Claude وارد می‌کند

- Markdown حافظه خودکار Claude Code از `~/.claude/projects/*/memory` و یک
  `autoMemoryDirectory` پیکربندی‌شده توسط کاربر، که برای بازیابی نمایه‌سازی‌شده زیر
  `memory/imports/claude-code/` کپی می‌شود.
- `CLAUDE.md` و `.claude/CLAUDE.md` پروژه به فضای کاری عامل OpenClaw ‏(`AGENTS.md`).
- `~/.claude/CLAUDE.md` کاربر که به `USER.md` فضای کاری افزوده می‌شود.
- تعریف‌های سرور MCP از `.mcp.json` پروژه، `~/.claude.json` مربوط به Claude Code (شامل ورودی‌های هر پروژه آن) و `claude_desktop_config.json` مربوط به Claude Desktop.
- پوشه‌های مهارت Claude که شامل `SKILL.md` هستند (`~/.claude/skills` کاربر و `.claude/skills` پروژه).
- فایل‌های Markdown فرمان Claude (`~/.claude/commands` کاربر و `.claude/commands` پروژه) که به مهارت‌های OpenClaw فقط با فراخوانی دستی تبدیل می‌شوند.

### وضعیت بایگانی و بازبینی دستی

هوک‌های Claude، مجوزها، پیش‌فرض‌های محیط، `CLAUDE.local.md` پروژه، `.claude/rules`، پوشه‌های `agents/` کاربر و پروژه، و تاریخچه پروژه (`projects`، `cache`، `plans` زیر `~/.claude`) در گزارش مهاجرت حفظ می‌شوند یا به‌عنوان موارد نیازمند بازبینی دستی گزارش می‌شوند. OpenClaw هوک‌ها را اجرا نمی‌کند، فهرست‌های مجاز گسترده را کپی نمی‌کند و وضعیت اعتبارنامه OAuth/Desktop را به‌طور خودکار وارد نمی‌کند.

## ارائه‌دهنده Codex

ارائه‌دهنده همراه Codex به‌طور پیش‌فرض وضعیت Codex CLI را در `~/.codex`، یا هنگامی که آن متغیر محیطی تنظیم شده باشد در `CODEX_HOME`، شناسایی می‌کند. برای فهرست‌برداری از یک خانه مشخص Codex از `--from <path>` استفاده کنید.

هنگام انتقال به چارچوب Codex مربوط به OpenClaw و زمانی که می‌خواهید دارایی‌های شخصی مفید Codex CLI را آگاهانه ارتقا دهید، از این ارائه‌دهنده استفاده کنید. اجرای محلی app-server مربوط به Codex از یک `CODEX_HOME` مختص هر عامل استفاده می‌کند، بنابراین به‌طور پیش‌فرض `~/.codex` شخصی شما را نمی‌خواند. `HOME` عادی فرایند همچنان به ارث می‌رسد، بنابراین Codex می‌تواند مهارت‌ها/ورودی‌های بازار Plugin مشترک `$HOME/.agents/*` را ببیند و زیرفرایندها می‌توانند پیکربندی و توکن‌های خانه کاربر را پیدا کنند.

اجرای `openclaw migrate codex` در یک پایانه تعاملی، طرح کامل را پیش‌نمایش می‌کند و سپس پیش از تأیید نهایی اعمال، انتخاب‌گرهای کادر انتخاب را باز می‌کند. ابتدا برای موارد کپی مهارت درخواست انتخاب می‌شود. برای انتخاب انبوه از `Toggle all on` یا `Toggle all off` استفاده کنید. برای تغییر وضعیت ردیف‌ها Space را فشار دهید، یا برای فعال‌کردن ردیف برجسته و ادامه Enter را فشار دهید. مهارت‌های برنامه‌ریزی‌شده از ابتدا علامت‌خورده‌اند، مهارت‌های دارای تداخل از ابتدا بدون علامت‌اند، و `Skip for now` کپی مهارت‌ها را برای این اجرا رد می‌کند، درحالی‌که همچنان به انتخاب Plugin ادامه می‌دهد. هنگامی که Pluginهای گزینش‌شده Codex که از مبدأ نصب شده‌اند قابل مهاجرت باشند و `--plugin` ارائه نشده باشد، مهاجرت سپس برای فعال‌سازی Plugin بومی Codex بر اساس نام Plugin درخواست انتخاب می‌کند. موارد Plugin از ابتدا علامت‌خورده‌اند، مگر اینکه پیکربندی مقصد Plugin مربوط به OpenClaw Codex از قبل آن Plugin را داشته باشد. Pluginهای موجود در مقصد از ابتدا بدون علامت‌اند و یک راهنمای تداخل مانند `conflict: plugin exists` نشان می‌دهند؛ برای مهاجرت‌ندادن هیچ Plugin بومی Codex در آن اجرا، `Toggle all off` را انتخاب کنید، یا برای توقف پیش از اعمال، `Skip for now` را انتخاب کنید.

برای اجراهای اسکریپتی یا دقیق، یک یا چند مهارت یا Plugin را صریحاً انتخاب کنید:

```bash
openclaw migrate codex --dry-run --skill gog-vault77-google-workspace
openclaw migrate apply codex --yes --skill gog-vault77-google-workspace
openclaw migrate codex --dry-run --plugin google-calendar
openclaw migrate apply codex --yes --plugin google-calendar
```

### آنچه Codex وارد می‌کند

- `MEMORY.md` و `memory_summary.md` یکپارچه‌شده Codex از
  `$CODEX_HOME/memories`، که برای بازیابی نمایه‌سازی‌شده زیر `memory/imports/codex/`
  کپی می‌شوند. حافظه خام rollout وارد نمی‌شود.
- پوشه‌های مهارت Codex CLI زیر `$CODEX_HOME/skills`، به‌جز حافظه نهان `.system` مربوط به Codex.
- AgentSkills شخصی زیر `$HOME/.agents/skills`، که برای مالکیت مختص هر عامل به فضای کاری عامل کنونی OpenClaw کپی می‌شوند.
- Pluginهای Codex نصب‌شده از مبدأ `openai-curated` که از طریق `plugin/list` مربوط به app-server متعلق به Codex کشف شده‌اند. برنامه‌ریزی برای هر Plugin نصب‌شده فعال، `plugin/read` را می‌خواند.

مهاجرت Pluginهای متکی بر برنامه، محدودیت‌های اضافی دارد:

- Pluginهای متکی بر برنامه مستلزم آن‌اند که حساب app-server مبدأ Codex یک حساب اشتراکی ChatGPT باشد. پاسخ‌های حساب غیر ChatGPT یا نبود پاسخ حساب با `codex_subscription_required` رد می‌شوند.
- به‌طور پیش‌فرض، مهاجرت `app/list` مبدأ را فراخوانی نمی‌کند؛ بنابراین Pluginهای متکی بر برنامه که از محدودیت حساب عبور می‌کنند، بدون تأیید دسترس‌پذیری برنامه مبدأ برنامه‌ریزی می‌شوند و خطاهای انتقال در جست‌وجوی حساب با `codex_account_unavailable` رد می‌شوند.
- برای اجباری‌کردن یک تصویر لحظه‌ای تازه `app/list` از مبدأ و الزام حضور، فعال‌بودن و دسترس‌پذیری همه برنامه‌های تحت مالکیت پیش از برنامه‌ریزی فعال‌سازی بومی، `--verify-plugin-apps` را ارسال کنید. در آن حالت، خطاهای انتقال در جست‌وجوی حساب به تأیید موجودی برنامه مبدأ منتقل می‌شوند. تصویر لحظه‌ای فقط برای فرایند کنونی در حافظه نگه داشته می‌شود؛ هرگز در خروجی مهاجرت یا پیکربندی مقصد نوشته نمی‌شود.

Pluginهای غیرفعال، جزئیات ناخوانای Plugin، حساب‌های مبدأ محدودشده به اشتراک و (هنگامی که `--verify-plugin-apps` تنظیم شده است) برنامه‌های مفقود، غیرفعال یا دسترس‌ناپذیر، به‌جای ورودی‌های پیکربندی مقصد به موارد دستی ردشده با دلایل نوع‌دار تبدیل می‌شوند. عملیات اعمال برای هر Plugin واجد شرایط انتخاب‌شده، `plugin/install` مربوط به app-server را فراخوانی می‌کند، حتی اگر app-server مقصد از قبل آن Plugin را نصب‌شده و فعال گزارش کند. Pluginهای Codex مهاجرت‌داده‌شده فقط در نشست‌هایی قابل استفاده‌اند که چارچوب بومی Codex را انتخاب می‌کنند؛ آن‌ها در اختیار اجراهای ارائه‌دهنده OpenClaw، اتصال‌های گفت‌وگوی ACP یا چارچوب‌های دیگر قرار نمی‌گیرند.

### وضعیت Codex نیازمند بازبینی دستی

Codex `config.toml`، `hooks/hooks.json` بومی، بازارهای غیرگزینش‌شده، بسته‌های Plugin ذخیره‌شده در حافظهٔ نهان که Pluginهای گزینش‌شدهٔ نصب‌شده از منبع نیستند، و Pluginهای نصب‌شده از منبع که در دروازهٔ اشتراک منبع رد می‌شوند، به‌طور خودکار فعال نمی‌شوند. وقتی `--verify-plugin-apps` تنظیم شده باشد، Pluginهایی که در دروازهٔ موجودی برنامهٔ منبع رد می‌شوند نیز نادیده گرفته می‌شوند. همهٔ این موارد برای بازبینی دستی در گزارش مهاجرت کپی یا گزارش می‌شوند.

برای Pluginهای گزینش‌شدهٔ مهاجرت‌یافته که از منبع نصب شده‌اند، این نوشتن‌ها اعمال می‌شوند:

- `plugins.entries.codex.enabled: true`
- `plugins.entries.codex.config.codexPlugins.enabled: true`
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions: true`
- یک ورودی صریح Plugin با `marketplaceName: "openai-curated"` و `pluginName` برای هر Plugin انتخاب‌شده

مهاجرت هرگز `plugins["*"]` را نمی‌نویسد و هرگز مسیرهای محلی حافظهٔ نهان بازار را ذخیره نمی‌کند.

Pluginهای نادیده‌گرفته‌شده در پیکربندی مقصد نوشته نمی‌شوند. خطاهای اشتراک سمت منبع در موارد دستی با دلایل نوع‌دار گزارش می‌شوند: `codex_subscription_required`، `codex_account_unavailable`، `plugin_disabled` یا `plugin_read_unavailable`. با `--verify-plugin-apps`، خطاهای موجودی برنامهٔ منبع نیز ممکن است به‌صورت `app_inaccessible`، `app_disabled`، `app_missing` یا `app_inventory_unavailable` ظاهر شوند. نصب‌های سمت مقصد که به احراز هویت نیاز دارند، در مورد Plugin متأثر با `status: "skipped"`، `reason: "auth_required"` و شناسه‌های پاک‌سازی‌شدهٔ برنامه گزارش می‌شوند؛ ورودی‌های صریح پیکربندی آن‌ها تا زمان صدور مجدد مجوز و فعال‌سازی، به‌صورت غیرفعال نوشته می‌شوند. سایر خطاهای نصب، نتایج `error` محدود به همان مورد هستند.

اگر موجودی Pluginهای app-server در Codex هنگام برنامه‌ریزی در دسترس نباشد، مهاجرت به‌جای شکست کل فرایند، از موارد راهنمای بسته‌های ذخیره‌شده در حافظهٔ نهان استفاده می‌کند.

## ارائه‌دهندهٔ Hermes

ارائه‌دهندهٔ همراه Hermes از `$HERMES_HOME` و نمایهٔ فعال پیروی می‌کند و سپس از پیش‌فرض پلتفرم (`~/.hermes` یا `%LOCALAPPDATA%\hermes`) استفاده می‌کند. برای بازنویسی کشف، از `--from <path>` استفاده کنید.

### مواردی که Hermes وارد می‌کند

- پیکربندی مدل پیش‌فرض از `config.yaml`.
- ارائه‌دهندگان مدل پیکربندی‌شده و نقطه‌های پایانی سفارشی سازگار با OpenAI از `model`، `providers` و `custom_providers`.
- تعریف‌های سرور MCP از `mcp_servers` یا `mcp.servers`. نگاشت‌های دقیق OpenClaw شامل مسیریابی پیش‌فرض Streamable HTTP، دامنهٔ OAuth، راستی‌آزمایی بولی TLS، مسیرهای جداگانهٔ گواهی و کلید کارخواه، و سیاست ابزار بومی/منبع/پرامپت Hermes می‌شوند. فیلدهای زمان اجرا یا اعتبارنامهٔ مختص Hermes که پشتیبانی نمی‌شوند، برای بازبینی دستی گزارش می‌شوند.
- `SOUL.md` و `AGENTS.md` به فضای کاری عامل OpenClaw.
- `memories/MEMORY.md` و `memories/USER.md` که به فایل‌های حافظهٔ فضای کاری افزوده می‌شوند.
  در عوض، سطوح صرفاً حافظه‌ای (صفحهٔ حافظهٔ راه‌اندازی اولیه و صفحهٔ واردکردن حافظه در رابط کنترل)
  این فایل‌ها را برای یادآوری نمایه‌شده، بدون دست‌زدن به حافظهٔ موجود فضای کاری،
  زیر `memory/imports/hermes/` کپی می‌کنند.
- پیش‌فرض‌های پیکربندی حافظه برای حافظهٔ فایلی OpenClaw، به‌همراه موارد بایگانی یا بازبینی دستی برای ارائه‌دهندگان حافظهٔ خارجی مانند Honcho.
- Skillsهایی که در هر نقطه زیر `skills/` شامل فایل `SKILL.md` هستند؛ Skillsهای تودرتو در پوشهٔ Skills فضای کاری مسطح می‌شوند.
- مقادیر پیکربندی مختص هر Skill از `skills.config`.
- اعتبارنامه‌های کنونی OAuth مربوط به OpenAI Codex در Hermes و اعتبارنامه‌های OAuth مربوط به OpenAI در OpenCode، هنگامی که مهاجرت تعاملی اعتبارنامه پذیرفته شود یا `--include-secrets` تنظیم شده باشد. اجازه ندهید Hermes و OpenClaw از یک مجوز نوسازی واردشدهٔ مشترک استفاده کنند.
- کلیدها و توکن‌های API پشتیبانی‌شده از `.env` در Hermes و `auth.json` در OpenCode، هنگامی که مهاجرت تعاملی اعتبارنامه پذیرفته شود یا `--include-secrets` تنظیم شده باشد.

### کلیدهای پشتیبانی‌شدهٔ `.env`

`AI_GATEWAY_API_KEY`، `ALIBABA_API_KEY`، `ANTHROPIC_API_KEY`، `ARCEEAI_API_KEY`، `CEREBRAS_API_KEY`، `CHUTES_API_KEY`، `CLOUDFLARE_AI_GATEWAY_API_KEY`، `COPILOT_GITHUB_TOKEN`، `DASHSCOPE_API_KEY`، `DEEPINFRA_API_KEY`، `DEEPSEEK_API_KEY`، `FIREWORKS_API_KEY`، `GEMINI_API_KEY`، `GH_TOKEN`، `GITHUB_TOKEN`، `GLM_API_KEY`، `GOOGLE_API_KEY`، `GROQ_API_KEY`، `HF_TOKEN`، `HUGGINGFACE_HUB_TOKEN`، `KILOCODE_API_KEY`، `KIMICODE_API_KEY`، `KIMI_API_KEY`، `KIMI_CODING_API_KEY`، `MINIMAX_API_KEY`، `MINIMAX_CODING_API_KEY`، `MISTRAL_API_KEY`، `MODELSTUDIO_API_KEY`، `MOONSHOT_API_KEY`، `NVIDIA_API_KEY`، `OPENAI_API_KEY`، `OPENCODE_API_KEY`، `OPENCODE_GO_API_KEY`، `OPENCODE_ZEN_API_KEY`، `OPENROUTER_API_KEY`، `QIANFAN_API_KEY`، `QWEN_API_KEY`، `TOGETHER_API_KEY`، `VENICE_API_KEY`، `XAI_API_KEY`، `XIAOMI_API_KEY`، `ZAI_API_KEY`، `Z_AI_API_KEY`.

### وضعیت فقط‌بایگانی

وضعیت Hermes که OpenClaw نمی‌تواند آن را با ایمنی تفسیر کند، برای بازبینی دستی در گزارش مهاجرت کپی می‌شود، اما در پیکربندی یا اعتبارنامه‌های فعال OpenClaw بارگذاری نمی‌شود. این موارد شامل `plugins/`، `sessions/`، `logs/`، `cron/`، `mcp-tokens/`، `plans/`، `workspace/`، `skins/`، `kanban/`، وضعیت جفت‌سازی/پلتفرم، وضعیت مسیریابی/فرایند Gateway و پایگاه‌های دادهٔ SQLite شناسایی‌شدهٔ Hermes هستند.

### پس از اعمال

```bash
openclaw doctor
```

## قرارداد Plugin

منابع مهاجرت، Plugin هستند. هر Plugin شناسه‌های ارائه‌دهندهٔ خود را در `openclaw.plugin.json` اعلام می‌کند:

```json
{
  "contracts": {
    "migrationProviders": ["hermes"]
  }
}
```

هنگام اجرا، Plugin تابع `api.registerMigrationProvider(...)` را فراخوانی می‌کند. ارائه‌دهنده، `detect`، `plan` و `apply` را پیاده‌سازی می‌کند. هسته مالک هماهنگ‌سازی CLI، سیاست پشتیبان‌گیری، پرامپت‌ها، خروجی JSON و پیش‌بررسی تعارض است. هسته برنامهٔ بازبینی‌شده را به `apply(ctx, plan)` می‌فرستد و ارائه‌دهندگان فقط زمانی می‌توانند برای سازگاری برنامه را بازسازی کنند که آن آرگومان وجود نداشته باشد. موارد مهاجرت می‌توانند برای اثرهای فعال‌سازی خارجی که راه‌اندازی اولیه باید تا زمان انتشار پایدار داده‌های محلی مرحله‌بندی‌شده به تعویق بیندازد، `applyPhase: "after-promotion"` را تنظیم کنند. آن ارائه‌دهندگان باید `deferredApply: { retrySafe: true }` را اعلام کنند و هر اثر معوق را برای اجرای مجدد پس از وقفه در فرایند ایمن سازند؛ راه‌اندازی اولیه اثرهای معوق اعلام‌نشده را رد می‌کند. یک عملیات بدون اثرِ هم‌توان باید موردی غیرجهش‌دهنده با `deferredCompletion: true` برگرداند تا بازیابی بتواند آن را کامل‌شده ثبت کند. `openclaw migrate` مستقل همچنان برنامهٔ کامل را از طریق جریان عادیِ متکی بر پشتیبان خود اعمال می‌کند.

Pluginهای ارائه‌دهنده می‌توانند برای ساخت موارد و شمارش‌های خلاصه از `openclaw/plugin-sdk/migration` و برای کپی فایل با آگاهی از تعارض، کپی‌های گزارش صرفاً بایگانی، پوشش‌دهنده‌های زمان اجرای پیکربندی ذخیره‌شده در حافظهٔ نهان و گزارش‌های مهاجرت از `openclaw/plugin-sdk/migration-runtime` استفاده کنند.

## یکپارچه‌سازی راه‌اندازی اولیه

وقتی ارائه‌دهنده‌ای یک منبع شناخته‌شده را شناسایی کند، راه‌اندازی اولیه می‌تواند مهاجرت را پیشنهاد دهد. هر دو `openclaw onboard --flow import` و `openclaw setup --wizard --import-from hermes` از همان ارائه‌دهندهٔ مهاجرت Plugin استفاده می‌کنند و همچنان پیش از اعمال، پیش‌نمایش را نشان می‌دهند. برخلاف مهاجرت مستقل، مسیر راه‌اندازی اولیه با مقصد تازه، مصنوعات محلی و اعتبارنامه‌های واردشده را مرحله‌بندی می‌کند، استنتاج واردشده را در مرحله‌بندی راستی‌آزمایی یا ترمیم می‌کند و سپس پیش از ثبت پیکربندی، فضای کاری و وضعیت عامل را ارتقا می‌دهد. یک دفتر ثبت ارتقای حالت-`0600` به اجرای بعدی امکان می‌دهد انتشار قطع‌شده، از جمله هرگونه فعال‌سازی خارجی معوق، را بدون اجرای مجدد داده‌های محلی واردشده تکمیل کند یا به عقب برگرداند.

<Note>
واردکردن از طریق راه‌اندازی اولیه به یک راه‌اندازی تازهٔ OpenClaw نیاز دارد. اگر از قبل وضعیت محلی دارید، ابتدا پیکربندی، اعتبارنامه‌ها، نشست‌ها و فضای کاری را بازنشانی کنید. واردکردن با پشتیبان‌گیری و بازنویسی یا ادغام برای راه‌اندازی‌های موجود پشت پرچم قابلیت قرار دارد.
</Note>

## مرتبط

- [مهاجرت از Hermes](/fa/install/migrating-hermes): راهنمای گام‌به‌گام کاربر.
- [مهاجرت از Claude](/fa/install/migrating-claude): راهنمای گام‌به‌گام کاربر.
- [مهاجرت](/fa/install/migrating): انتقال OpenClaw به دستگاهی جدید.
- [Doctor](/fa/gateway/doctor): بررسی سلامت پس از اعمال مهاجرت.
- [Pluginها](/fa/tools/plugin): نصب و ثبت Plugin.
