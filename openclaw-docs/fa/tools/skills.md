---
read_when:
    - افزودن یا تغییر مهارت‌ها
    - تغییر محدودسازی، فهرست‌های مجاز یا قواعد بارگذاری Skills
    - درک تقدم Skills و رفتار اسنپ‌شات
sidebarTitle: Skills
summary: Skills به عامل شما می‌آموزند چگونه از ابزارها استفاده کند. با نحوهٔ بارگذاری آن‌ها، چگونگی عملکرد اولویت، و روش پیکربندی دروازه‌بندی، فهرست‌های مجاز و تزریق محیط آشنا شوید.
title: Skills
x-i18n:
    generated_at: "2026-07-27T16:06:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 6925add85652023e3dd2f51f607412fd0bf00581923f76ab2aafd2ca5b8d72be
    source_path: tools/skills.md
    workflow: 16
---

Skills فایل‌های دستورالعمل Markdown هستند که به عامل می‌آموزند چگونه و چه زمانی از
ابزارها استفاده کند. هر Skill در یک دایرکتوری قرار دارد که شامل یک فایل `SKILL.md` با
frontmatter از نوع YAML و یک بدنه Markdown است. OpenClaw، Skills همراه را به‌اضافه هرگونه
بازنویسی محلی بارگیری می‌کند و هنگام بارگیری، آن‌ها را بر اساس محیط، پیکربندی و
وجود فایل‌های باینری پالایش می‌کند.

<CardGroup cols={2}>
  <Card title="ایجاد Skills" href="/fa/tools/creating-skills" icon="hammer">
    یک Skill سفارشی را از ابتدا بسازید و آزمایش کنید.
  </Card>
  <Card title="کارگاه Skill" href="/fa/tools/skill-workshop" icon="flask">
    پیشنهادهای Skill پیش‌نویس‌شده توسط عامل را بازبینی و تأیید کنید.
  </Card>
  <Card title="پیکربندی Skills" href="/fa/tools/skills-config" icon="gear">
    طرح‌واره کامل پیکربندی `skills.*` و فهرست‌های مجاز عامل.
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="cloud">
    Skills جامعه را مرور و نصب کنید.
  </Card>
</CardGroup>

## ترتیب بارگیری

OpenClaw از منابع زیر بارگیری می‌کند، **ابتدا با بالاترین اولویت**. هنگامی که نام یک
Skill در چند مکان ظاهر شود، منبع دارای بالاترین اولویت برنده می‌شود.

| اولویت       | منبع                    | مسیر                                    |
| ------------ | ----------------------- | --------------------------------------- |
| 1 — بالاترین | Skills فضای کاری        | `<workspace>/skills`                    |
| 2            | Skills عامل پروژه       | `<workspace>/.agents/skills`            |
| 3            | Skills شخصی عامل        | `~/.agents/skills`                      |
| 4            | Skills مدیریت‌شده / محلی | `~/.openclaw/skills`                    |
| 5            | Skills همراه            | همراه با نصب ارائه می‌شوند              |
| 6 — پایین‌ترین | دایرکتوری‌های اضافی     | `skills.load.extraDirs` + Skills افزونه |

ریشه‌های Skill از چیدمان‌های گروه‌بندی‌شده پشتیبانی می‌کنند. OpenClaw هر زمان که
`SKILL.md` در هر جایی زیر یک ریشه پیکربندی‌شده ظاهر شود، آن Skill را کشف می‌کند (تا عمق 6 سطح):

```text
<workspace>/skills/research/SKILL.md          ✓ با نام "research" یافت شد
<workspace>/skills/personal/research/SKILL.md ✓ این مورد نیز با نام "research" یافت شد
```

مسیر پوشه فقط برای سازمان‌دهی است. نام Skill و فرمان اسلش آن
از فیلد frontmatter با نام `name` می‌آیند (یا اگر `name` وجود نداشته باشد، از نام
دایرکتوری). فهرست‌های مجاز عامل (در ادامه) نیز بر اساس همین `name` تطبیق می‌دهند.

<Note>
  دایرکتوری بومی `$CODEX_HOME/skills` در Codex CLI، ریشه Skill در OpenClaw
  **نیست**. برای فهرست‌برداری از آن Skills از `openclaw migrate plan codex` استفاده کنید، سپس
  برای کپی‌کردن آن‌ها به فضای کاری OpenClaw خود از `openclaw migrate codex` استفاده کنید.
</Note>

## Skills میزبانی‌شده روی Node

یک Node بدون رابط متصل می‌تواند Skills نصب‌شده در دایرکتوری فعال
Skills در OpenClaw را منتشر کند (به‌طور پیش‌فرض `~/.openclaw/skills`؛ بازنویسی‌های محیطی پروفایل
اعمال می‌شوند). تا زمانی که Node متصل است، آن‌ها در فهرست عادی Skills عامل ظاهر می‌شوند
و هنگام قطع اتصال ناپدید می‌شوند. در صورت تداخل، یک Skill محلی یا Gateway نام خود را حفظ
می‌کند؛ Skill مربوط به Node نامی قطعی با پیشوند Node دریافت می‌کند.
نسخه v1 میزبانی‌شده روی Node مستلزم آن است که نام دایرکتوری با فیلد frontmatter
با نام `name` در Skill مطابقت داشته باشد.

ورودی Skill شامل مکان‌یاب Node است. فایل‌ها، ارجاع‌های نسبی و
فایل‌های باینری آن روی Node قرار دارند؛ بنابراین آن را با
`exec host=node node=<node-id>` بارگیری و اجرا کنید. پس از تغییر فایل‌های
Skill، میزبان Node را راه‌اندازی مجدد کنید. برای جفت‌سازی و کلیدهای غیرفعال‌سازی به [Nodeها](/fa/nodes#node-hosted-skills) مراجعه کنید.

## Skills مختص هر عامل در برابر Skills مشترک

در راه‌اندازی‌های چندعاملی، هر عامل فضای کاری خودش را دارد. از مسیری استفاده کنید که
با سطح مشاهده‌پذیری موردنظر شما مطابقت دارد:

| دامنه          | مسیر                         | قابل‌مشاهده برای                  |
| -------------- | ---------------------------- | --------------------------------- |
| مختص هر عامل   | `<workspace>/skills`         | فقط همان عامل                     |
| عامل پروژه     | `<workspace>/.agents/skills` | فقط عامل همان فضای کاری           |
| عامل شخصی      | `~/.agents/skills`           | همه عامل‌های این دستگاه           |
| مدیریت‌شده مشترک | `~/.openclaw/skills`         | همه عامل‌های این دستگاه           |
| دایرکتوری‌های اضافی | `skills.load.extraDirs`      | همه عامل‌های این دستگاه           |

## فهرست‌های مجاز عامل

**مکان** Skill (اولویت) و **قابلیت مشاهده** Skill (اینکه کدام عامل می‌تواند از
آن استفاده کند) کنترل‌هایی جداگانه هستند. برای محدودکردن Skills قابل‌مشاهده برای هر عامل،
صرف‌نظر از محل بارگیری آن‌ها، از فهرست‌های مجاز استفاده کنید.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // خط مبنای مشترک
    },
    list: [
      { id: "writer" }, // github و weather را به ارث می‌برد
      { id: "docs", skills: ["docs-search"] }, // مقادیر پیش‌فرض را کاملاً جایگزین می‌کند
      { id: "locked-down", skills: [] }, // بدون Skill
    ],
  },
}
```

<AccordionGroup>
  <Accordion title="قواعد فهرست مجاز">
    - برای اینکه همه Skills به‌طور پیش‌فرض بدون محدودیت باقی بمانند، `agents.defaults.skills` را حذف کنید.
    - برای به‌ارث‌بردن `agents.defaults.skills`، `agents.entries.*.skills` را حذف کنید.
    - برای آنکه هیچ Skill برای آن عامل ارائه نشود، `agents.entries.*.skills: []` را تنظیم کنید.
    - یک فهرست غیرخالی `agents.entries.*.skills` مجموعه **نهایی** است — با مقادیر
      پیش‌فرض ادغام نمی‌شود.
    - فهرست مجاز مؤثر در ساخت پرامپت، کشف فرمان‌های اسلش،
      همگام‌سازی sandbox و اسنپ‌شات‌های Skill اعمال می‌شود.
    - این یک مرز مجوزدهی پوسته میزبان نیست. اگر همان عامل بتواند
      از `exec` استفاده کند، آن پوسته را جداگانه با sandboxing، جداسازی
      کاربر سیستم‌عامل، فهرست‌های ممنوع/مجاز اجرا و اعتبارنامه‌های مختص هر منبع محدود کنید.
  </Accordion>
</AccordionGroup>

## افزونه‌ها و Skills

افزونه‌ها می‌توانند با فهرست‌کردن دایرکتوری‌های `skills` در
`openclaw.plugin.json`، Skills خود را ارائه کنند (مسیرها نسبت به ریشه افزونه هستند). Skills افزونه
هنگامی بارگیری می‌شوند که افزونه فعال باشد — برای مثال، افزونه مرورگر یک
Skill با نام `browser-automation` برای کنترل چندمرحله‌ای مرورگر ارائه می‌کند.

دایرکتوری‌های Skill افزونه در همان سطح کم‌اولویت
`skills.load.extraDirs` ادغام می‌شوند؛ بنابراین یک Skill همراه، مدیریت‌شده، عامل یا فضای کاری
با همان نام، آن‌ها را بازنویسی می‌کند. واجدشرایط‌بودن خود یک Skill افزونه را از طریق
`metadata.openclaw.requires` در frontmatter آن، مانند هر Skill دیگری، کنترل کنید.

برای آشنایی با سامانه کامل افزونه به [افزونه‌ها](/fa/tools/plugin) و [ابزارها](/fa/tools) مراجعه کنید.

## کارگاه Skill

[کارگاه Skill](/fa/tools/skill-workshop) یک صف پیشنهاد میان عامل
و فایل‌های فعال Skill شماست. هنگامی که عامل کار قابل‌استفاده مجددی را شناسایی می‌کند، به‌جای
نوشتن مستقیم در `SKILL.md` یک پیشنهاد پیش‌نویس می‌کند. پیش از هرگونه تغییر،
آن را بازبینی و تأیید می‌کنید.

```bash
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

برای چرخه عمر کامل، مرجع CLI و پیکربندی به [کارگاه Skill](/fa/tools/skill-workshop) مراجعه کنید.

## نصب از ClawHub

[ClawHub](https://clawhub.ai) رجیستری عمومی Skills است. برای نصب و به‌روزرسانی از
فرمان‌های `openclaw skills` یا برای انتشار و همگام‌سازی از CLI با نام `clawhub`
استفاده کنید.

| اقدام                                  | فرمان                                                  |
| -------------------------------------- | ------------------------------------------------------ |
| نصب یک Skill در فضای کاری             | `openclaw skills install @owner/<slug>`                |
| نصب از یک مخزن Git                    | `openclaw skills install git:owner/repo@ref`           |
| نصب یک دایرکتوری محلی Skill           | `openclaw skills install ./path/to/skill --as my-tool` |
| نصب برای همه عامل‌های محلی            | `openclaw skills install @owner/<slug> --global`       |
| به‌روزرسانی همه Skills فضای کاری      | `openclaw skills update --all`                         |
| به‌روزرسانی یک Skill مدیریت‌شده مشترک | `openclaw skills update @owner/<slug> --global`        |
| به‌روزرسانی همه Skills مدیریت‌شده مشترک | `openclaw skills update --all --global`                |
| تأیید محدوده اعتماد یک Skill          | `openclaw skills verify @owner/<slug>`                 |
| چاپ کارت Skill تولیدشده               | `openclaw skills verify @owner/<slug> --card`          |
| انتشار / همگام‌سازی از طریق CLI مربوط به ClawHub | `clawhub sync --all`                                   |

<AccordionGroup>
  <Accordion title="جزئیات نصب">
    `openclaw skills install` به‌طور پیش‌فرض در دایرکتوری فعال `skills/`
    فضای کاری نصب می‌کند. برای نصب در دایرکتوری مشترک
    `~/.openclaw/skills`، گزینه `--global` را اضافه کنید؛ این دایرکتوری برای همه عامل‌های محلی
    قابل‌مشاهده است، مگر آنکه فهرست‌های مجاز عامل آن را محدود کنند.

    نصب‌های Git و محلی انتظار دارند `SKILL.md` در ریشه منبع قرار داشته باشد. نامک
    در صورت معتبر بودن از `name` در frontmatter مربوط به `SKILL.md` می‌آید و سپس به نام
    دایرکتوری یا مخزن بازمی‌گردد. برای بازنویسی از `--as <slug>` استفاده کنید.
    `openclaw skills update` فقط نصب‌های ClawHub را ردیابی می‌کند — برای تازه‌سازی منابع Git یا
    محلی، آن‌ها را دوباره نصب کنید.

  </Accordion>
  <Accordion title="تأیید و اسکن امنیتی">
    `openclaw skills verify @owner/<slug>` محدوده اعتماد `clawhub.skill.verify.v1`
    مربوط به Skill را از ClawHub درخواست می‌کند. Skills نصب‌شده از ClawHub در برابر
    نسخه و رجیستری ثبت‌شده در `.clawhub/origin.json` تأیید می‌شوند.
    نامک‌های بدون مالک همچنان برای Skills موجودِ نصب‌شده یا بدون ابهام پذیرفته می‌شوند، اما
    ارجاع‌های واجد نام مالک از ابهام ناشر جلوگیری می‌کنند.

    صفحات Skill در ClawHub آخرین وضعیت اسکن امنیتی را پیش از نصب نمایش می‌دهند
    و برای VirusTotal، ClawScan و تحلیل ایستا صفحات جزئیات دارند. هنگامی که ClawHub
    تأیید را ناموفق علامت‌گذاری کند، فرمان با کد غیرصفر خارج می‌شود. ناشران
    نتایج مثبت کاذب را از طریق داشبورد ClawHub یا
    `clawhub skill rescan @owner/<slug>` رفع می‌کنند.

  </Accordion>
  <Accordion title="نصب بایگانی خصوصی">
    کلاینت‌های Gateway که به تحویل غیر ClawHub نیاز دارند، می‌توانند یک بایگانی zip از Skill را
    با `skills.upload.begin`، `skills.upload.chunk` و `skills.upload.commit`
    آماده کنند، سپس آن را با `skills.install({ source: "upload", ... })` نصب کنند. این مسیر
    به‌طور پیش‌فرض غیرفعال است و به `skills.install.allowUploadedArchives: true` در
    `openclaw.json` نیاز دارد. نصب‌های عادی ClawHub هرگز به آن تنظیم نیاز ندارند.
  </Accordion>
</AccordionGroup>

## امنیت

<Warning>
  Skills شخص ثالث را **کد نامطمئن** در نظر بگیرید. پیش از فعال‌سازی، آن‌ها را بخوانید.
  برای ورودی‌های نامطمئن و ابزارهای پرخطر، اجراهای sandbox‌شده را ترجیح دهید. برای کنترل‌های
  سمت عامل به [Sandboxing](/fa/gateway/sandboxing) مراجعه کنید.
</Warning>

<AccordionGroup>
  <Accordion title="محدودسازی مسیر">
    کشف Skill در فضای کاری، عامل پروژه و دایرکتوری اضافی فقط ریشه‌های
    Skill را می‌پذیرد که realpath حل‌شده آن‌ها داخل ریشه پیکربندی‌شده باقی بماند، مگر آنکه
    `skills.load.allowSymlinkTargets` صریحاً یک ریشه مقصد را قابل‌اعتماد بداند.
    کارگاه Skill فقط هنگامی از طریق آن مقصدهای قابل‌اعتماد می‌نویسد که
    `skills.workshop.allowSymlinkTargetWrites` فعال باشد.
    `~/.openclaw/skills` مدیریت‌شده و `~/.agents/skills` شخصی ممکن است شامل
    پوشه‌های Skill دارای پیوند نمادین باشند، اما realpath هر `SKILL.md` همچنان باید
    داخل دایرکتوری حل‌شده همان Skill باقی بماند.
  </Accordion>
  <Accordion title="سیاست نصب اپراتور">
    `security.installPolicy` را پیکربندی کنید تا پیش از ادامه نصب Skills،
    یک فرمان سیاست محلی قابل‌اعتماد اجرا شود. این سیاست فراداده و مسیر منبع آماده‌شده را
    دریافت می‌کند، بر مسیرهای ClawHub، بارگذاری‌شده، Git، محلی، به‌روزرسانی و
    نصب‌کننده وابستگی اعمال می‌شود و اگر فرمان نتواند تصمیم معتبری برگرداند، به‌صورت بسته شکست می‌خورد.
  </Accordion>
  <Accordion title="دامنه تزریق اطلاعات محرمانه">
    `skills.entries.*.env` و `skills.entries.*.apiKey` اطلاعات محرمانه را فقط برای همان نوبت عامل
    به فرایند **میزبان** تزریق می‌کنند — نه به sandbox. اطلاعات محرمانه را
    از پرامپت‌ها و گزارش‌ها دور نگه دارید.
  </Accordion>
</AccordionGroup>

برای مدل تهدید گسترده‌تر و چک‌لیست‌های امنیتی به
[امنیت](/fa/gateway/security) مراجعه کنید.

## قالب SKILL.md

هر Skill دست‌کم به یک `name` و `description` در frontmatter نیاز دارد:

```markdown
---
name: image-lab
description: تولید یا ویرایش تصاویر از طریق یک گردش‌کار تصویر متکی بر ارائه‌دهنده
---

هنگامی که کاربر درخواست تولید تصویر می‌کند، از ابزار `image_generate` استفاده کنید...
```

<Note>
  OpenClaw از مشخصات [AgentSkills](https://agentskills.io) پیروی می‌کند. frontmatter
  ابتدا به‌صورت YAML تجزیه می‌شود؛ اگر ناموفق باشد، به یک تجزیه‌گر محدود به تک‌خط
  بازمی‌گردد. بلوک‌های تودرتوی `metadata` (از جمله نگاشت‌های چندخطی YAML)
  به یک رشته JSON تخت می‌شوند و دوباره به‌صورت JSON5 تجزیه می‌شوند؛ بنابراین قالب بلوکی نمایش‌داده‌شده
  در بخش [کنترل دسترسی](#gating) کار می‌کند. برای ارجاع به مسیر پوشه
  Skill در بدنه از `{baseDir}` استفاده کنید.
</Note>

### کلیدهای اختیاری frontmatter

<ParamField path="homepage" type="string">
  نشانی اینترنتی که با عنوان "Website" در رابط کاربری Skills در macOS نمایش داده می‌شود. همچنین از طریق
  `metadata.openclaw.homepage` پشتیبانی می‌شود.
</ParamField>

<ParamField path="user-invocable" type="boolean" default="true">
  وقتی `true` باشد، Skills به‌صورت یک فرمان اسلش قابل‌فراخوانی توسط کاربر در دسترس قرار می‌گیرد.
</ParamField>

<ParamField path="disable-model-invocation" type="boolean" default="false">
  وقتی `true` باشد، OpenClaw دستورالعمل‌های Skills را از پرامپت عادی
  عامل خارج نگه می‌دارد. اگر `user-invocable`
  نیز `true` باشد، Skills همچنان به‌صورت فرمان اسلش در دسترس است.
</ParamField>

<ParamField path="command-dispatch" type='"tool"'>
  وقتی روی `tool` تنظیم شود، فرمان اسلش مدل را دور می‌زند و
  مستقیماً به یک ابزار ثبت‌شده ارسال می‌شود.
</ParamField>

<ParamField path="command-tool" type="string">
  نام ابزاری که هنگام تنظیم بودن `command-dispatch: tool` فراخوانی می‌شود.
</ParamField>

<ParamField path="command-arg-mode" type='"raw"' default="raw">
  برای ارسال به ابزار، رشته آرگومان‌های خام را بدون هیچ‌گونه
  تجزیه در هسته به ابزار منتقل می‌کند. ابزار
  `{ command: "<raw args>", commandName: "<slash command>", skillName: "<skill name>" }` را دریافت می‌کند.
</ParamField>

## دروازه‌بندی

OpenClaw هنگام بارگذاری، Skills را با استفاده از `metadata.openclaw` پالایش می‌کند (یک شیء JSON5
که در frontmatter جاسازی شده است؛ یادداشت تجزیه در بالا را ببینید). Skills بدون
بلوک `metadata.openclaw` همیشه واجد شرایط است، مگر اینکه صراحتاً غیرفعال شده باشد.

```markdown
---
name: image-lab
description: تولید یا ویرایش تصاویر از طریق گردش‌کار تصویر متکی بر ارائه‌دهنده
metadata:
  {
    "openclaw":
      {
        "requires": { "bins": ["uv"], "env": ["GEMINI_API_KEY"], "config": ["browser.enabled"] },
        "primaryEnv": "GEMINI_API_KEY",
      },
  }
---
```

<ParamField path="always" type="boolean">
  وقتی `true` باشد، همیشه Skills را شامل کرده و همه دروازه‌های دیگر را نادیده بگیرید.
</ParamField>

<ParamField path="emoji" type="string">
  ایموجی اختیاری که در رابط کاربری Skills در macOS نمایش داده می‌شود.
</ParamField>

<ParamField path="homepage" type="string">
  نشانی اینترنتی اختیاری که با عنوان "Website" در رابط کاربری Skills در macOS نمایش داده می‌شود.
</ParamField>

<ParamField path="os" type='("darwin" | "linux" | "win32")[]'>
  پالایه پلتفرم. در صورت تنظیم، Skills فقط در سیستم‌عامل‌های فهرست‌شده واجد شرایط است.
</ParamField>

<ParamField path="requires.bins" type="string[]">
  هر فایل اجرایی باید در `PATH` وجود داشته باشد.
</ParamField>

<ParamField path="requires.anyBins" type="string[]">
  دست‌کم یک فایل اجرایی باید در `PATH` وجود داشته باشد.
</ParamField>

<ParamField path="requires.env" type="string[]">
  هر متغیر محیطی باید در فرایند وجود داشته باشد یا از طریق پیکربندی ارائه شود.
</ParamField>

<ParamField path="requires.config" type="string[]">
  هر مسیر `openclaw.json` باید مقدار truthy داشته باشد.
</ParamField>

<ParamField path="primaryEnv" type="string">
  نام متغیر محیطی مرتبط با `skills.entries.<name>.apiKey`.
</ParamField>

<ParamField path="install" type="object[]">
  مشخصات اختیاری نصب‌کننده که رابط کاربری Skills در macOS از آن‌ها استفاده می‌کند (brew / node / go / uv / download).
</ParamField>

<Note>
  وقتی `metadata.openclaw` وجود نداشته باشد، بلوک‌های قدیمی `metadata.clawdbot` همچنان پذیرفته می‌شوند
  تا Skills نصب‌شده قدیمی، دروازه‌های وابستگی و راهنمایی‌های
  نصب‌کننده خود را حفظ کنند. Skills جدید باید از
  `metadata.openclaw` استفاده کند.
</Note>

### مشخصات نصب‌کننده

مشخصات نصب‌کننده به رابط کاربری Skills در macOS می‌گوید چگونه یک وابستگی را نصب کند:

```markdown
---
name: gemini
description: استفاده از Gemini CLI برای کمک در کدنویسی و جست‌وجوهای Google.
metadata:
  {
    "openclaw":
      {
        "emoji": "♊️",
        "requires": { "bins": ["gemini"] },
        "install":
          [
            {
              "id": "brew",
              "kind": "brew",
              "formula": "gemini-cli",
              "bins": ["gemini"],
              "label": "نصب Gemini CLI ‏(brew)",
            },
          ],
      },
  }
---
```

<AccordionGroup>
  <Accordion title="قواعد انتخاب نصب‌کننده">
    - وقتی چند نصب‌کننده فهرست شده باشند، Gateway یک گزینه ترجیحی را انتخاب می‌کند
      (در صورت دسترس‌بودن brew، وگرنه node).
    - اگر همه نصب‌کننده‌ها `download` باشند، OpenClaw هر ورودی را فهرست می‌کند تا بتوان
      همه مصنوعات موجود را دید.
    - مشخصات می‌توانند برای پالایش بر اساس پلتفرم شامل `os: ["darwin"|"linux"|"win32"]` باشند.
    - نصب‌های Node از `skills.install.nodeManager` در `openclaw.json` پیروی می‌کنند
      (پیش‌فرض: npm؛ گزینه‌ها: npm / pnpm / yarn / bun). این فقط بر نصب
      Skills اثر می‌گذارد؛ زمان اجرای Gateway همچنان باید Node باشد.
    - ترتیب ترجیح نصب‌کننده Gateway: Homebrew ← uv ← مدیر node پیکربندی‌شده ←
      go ← download.
  </Accordion>
  <Accordion title="جزئیات هر نصب‌کننده">
    - **Homebrew:** OpenClaw، Homebrew را به‌طور خودکار نصب نمی‌کند و فرمول‌های brew را
      به فرمان‌های بسته سیستم تبدیل نمی‌کند. در کانتینرهای Linux فاقد
      `brew`، نصب‌کننده‌های صرفاً brew پنهان می‌شوند؛ از یک تصویر سفارشی استفاده کنید یا
      وابستگی را به‌صورت دستی نصب کنید.
    - **Go:** OpenClaw برای نصب خودکار Skills به Go 1.21 یا جدیدتر نیاز دارد.
      اگر `go` وجود نداشته باشد و Homebrew در دسترس باشد، OpenClaw ابتدا Go را از طریق
      Homebrew نصب می‌کند؛ در Linux بدون Homebrew، اگر گزینه تازه‌سازی‌شده `golang-go`
      حداقل نسخه را تأمین کند، می‌تواند به‌جای آن از `apt-get`
      به‌عنوان root یا از طریق `sudo` بدون گذرواژه استفاده کند. مقدار واقعی `go install` برای
      وابستگی همیشه یک پوشه bin اختصاصی تحت مدیریت OpenClaw را هدف می‌گیرد
      (`bin` متعلق به Homebrew در یک نصب تازه، وگرنه `~/.local/bin`) و نه
      `GOBIN` پیکربندی‌شده؛ متغیرهای محیطی `GOBIN`، `GOPATH` و `GOTOOLCHAIN`
      متعلق به خودتان خوانده می‌شوند، اما هرگز بازنویسی نمی‌شوند.
    - **دانلود:** `url` (الزامی)، `archive` ‏(`tar.gz` | `tar.bz2` | `zip`)،
      `extract` (پیش‌فرض: تشخیص خودکار هنگام شناسایی بایگانی)، `stripComponents`،
      `targetDir` (پیش‌فرض: `~/.openclaw/tools/<skillKey>`).
  </Accordion>
  <Accordion title="یادداشت‌های سندباکس">
    `requires.bins` هنگام بارگذاری Skills روی **میزبان** بررسی می‌شود. اگر عاملی
    در سندباکس اجرا شود، فایل اجرایی باید **داخل کانتینر** نیز وجود داشته باشد.
    آن را از طریق `agents.defaults.sandbox.docker.setupCommand` یا یک
    تصویر سفارشی نصب کنید. `setupCommand` پس از ایجاد کانتینر یک‌بار اجرا می‌شود و به
    خروجی شبکه، فایل‌سیستم ریشه قابل‌نوشتن و کاربر root در سندباکس نیاز دارد.
  </Accordion>
</AccordionGroup>

## بازنویسی‌های پیکربندی

Skills همراه یا مدیریت‌شده را زیر `skills.entries` در
`~/.openclaw/openclaw.json` فعال یا غیرفعال و پیکربندی کنید:

```json5
{
  skills: {
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
        config: {
          endpoint: "https://example.invalid",
          model: "nano-pro",
        },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<ParamField path="enabled" type="boolean">
  `false` حتی در صورت همراه یا نصب‌شده بودن، Skills را غیرفعال می‌کند. Skills همراه `coding-agent`
  نیازمند فعال‌سازی صریح است — `skills.entries.coding-agent.enabled: true` را تنظیم کنید
  و مطمئن شوید یکی از `claude`، `codex`، `opencode` یا یک CLI پشتیبانی‌شده دیگر
  نصب شده و احراز هویت شده است.
</ParamField>

<ParamField path="apiKey" type='string | { source, provider, id }'>
  فیلدی کمکی برای Skillsهایی که `metadata.openclaw.primaryEnv` را اعلام می‌کنند.
  از یک رشته متن ساده یا شیء SecretRef پشتیبانی می‌کند.
</ParamField>

<ParamField path="env" type="Record<string, string>">
  متغیرهای محیطی تزریق‌شده برای اجرای عامل. فقط وقتی تزریق می‌شوند که
  متغیر از قبل در فرایند تنظیم نشده باشد.
</ParamField>

<ParamField path="config" type="object">
  مجموعه‌ای اختیاری برای فیلدهای پیکربندی سفارشی هر Skills.
</ParamField>

<ParamField path="allowBundled" type="string[]">
  فهرست مجاز اختیاری فقط برای Skills **همراه**. در صورت تنظیم، تنها Skills همراه
  موجود در فهرست واجد شرایط است. Skills مدیریت‌شده و فضای کاری تحت تأثیر قرار نمی‌گیرند.
</ParamField>

<Note>
  کلیدهای پیکربندی به‌طور پیش‌فرض با **نام Skills** مطابقت دارند. اگر Skills
  `metadata.openclaw.skillKey` را تعریف کرده باشد، به‌جای آن از همان کلید زیر `skills.entries` استفاده کنید.
  نام‌های دارای خط تیره را در گیومه قرار دهید: JSON5 کلیدهای نقل‌قول‌شده را مجاز می‌داند.
</Note>

## تزریق محیط

هنگام آغاز اجرای عامل، OpenClaw:

<Steps>
  <Step title="فراداده Skills را می‌خواند">
    OpenClaw فهرست مؤثر Skills را برای عامل تعیین می‌کند و قواعد دروازه‌بندی،
    فهرست‌های مجاز و بازنویسی‌های پیکربندی را اعمال می‌کند.
  </Step>
  <Step title="محیط و کلیدهای API را تزریق می‌کند">
    `skills.entries.<key>.env` و `skills.entries.<key>.apiKey` برای مدت اجرای عامل روی
    `process.env` اعمال می‌شوند.
  </Step>
  <Step title="پرامپت سیستم را می‌سازد">
    Skills واجد شرایط در یک بلوک XML فشرده گردآوری و در
    پرامپت سیستم تزریق می‌شوند.
  </Step>
  <Step title="محیط را بازیابی می‌کند">
    پس از پایان اجرا، محیط اصلی بازیابی می‌شود.
  </Step>
</Steps>

<Warning>
  تزریق متغیرهای محیطی به اجرای عامل روی **میزبان** محدود است، نه سندباکس. داخل
  سندباکس، `env` و `apiKey` اثری ندارند. برای شیوه
  انتقال اسرار به اجراهای سندباکس‌شده، [پیکربندی Skills](/fa/tools/skills-config#sandboxed-skills-and-env-vars) را ببینید.
</Warning>

برای بک‌اند همراه `claude-cli`، OpenClaw همچنین همان تصویر لحظه‌ای
Skills واجد شرایط را به‌صورت یک Plugin موقت Claude Code ایجاد کرده و آن را از طریق
`--plugin-dir` منتقل می‌کند. سایر بک‌اندهای CLI فقط از کاتالوگ پرامپت استفاده می‌کنند.

## تصاویر لحظه‌ای و تازه‌سازی

OpenClaw از Skills واجد شرایط **هنگام آغاز نشست** تصویر لحظه‌ای می‌گیرد و همان
فهرست را برای تمام نوبت‌های بعدی نشست دوباره استفاده می‌کند. تغییرات Skills یا پیکربندی
در نشست جدید بعدی اعمال می‌شوند.

Skills در دو حالت میان نشست تازه‌سازی می‌شود:

- ناظر Skills تغییر `SKILL.md` را تشخیص دهد.
- یک Node راه‌دور واجد شرایط جدید متصل شود.

فهرست تازه‌شده در نوبت بعدی عامل استفاده می‌شود. اگر فهرست مجاز مؤثر عامل
تغییر کند، OpenClaw تصویر لحظه‌ای را تازه‌سازی می‌کند تا Skills قابل‌مشاهده
هم‌تراز بماند.

<AccordionGroup>
  <Accordion title="ناظر Skills">
    OpenClaw به‌طور پیش‌فرض پوشه‌های Skills را پایش می‌کند و هنگام تغییر
    فایل‌های `SKILL.md` تصویر لحظه‌ای را به‌روزرسانی می‌کند. آن را زیر `skills.load` پیکربندی کنید:

    ```json5
    {
      skills: {
        load: {
          extraDirs: ["~/Projects/agent-scripts/skills"],
          allowSymlinkTargets: ["~/Projects/manager/skills"],
          watch: true, // پیش‌فرض
        },
      },
    }
    ```

    رویدادهای ناظر از یک حذف لرزش داخلی 250 ms استفاده می‌کنند. برای چیدمان‌های
    نمادین عمدی که در آن‌ها یک پیوند نمادین ریشه Skills به خارج از ریشه پیکربندی‌شده اشاره می‌کند،
    از `allowSymlinkTargets` استفاده کنید؛ برای مثال
    `<workspace>/skills/manager -> ~/Projects/manager/skills`.
    `skills.workshop.allowSymlinkTargetWrites` را فقط زمانی فعال کنید که Skill Workshop
    باید پیشنهادها را نیز از طریق آن مسیرهای نمادین مورد اعتماد اعمال کند.

  </Accordion>
  <Accordion title="Nodeهای راه‌دور macOS (Gateway لینوکسی)">
    اگر Gateway روی Linux اجرا شود اما یک **Node در macOS** با مجوز
    `system.run` متصل باشد، OpenClaw می‌تواند Skills مخصوص macOS را واجد شرایط بداند، به‌شرط آنکه
    فایل‌های اجرایی مورد نیاز در آن Node موجود باشند. عامل باید آن
    Skills را از طریق ابزار `exec` با `host=node` اجرا کند.

    Nodeهای آفلاین، Skills صرفاً راه‌دور را قابل‌مشاهده **نمی‌کنند**. اگر یک Node دیگر
    به کاوش‌های فایل اجرایی پاسخ ندهد، OpenClaw تطابق‌های فایل اجرایی ذخیره‌شده آن را پاک می‌کند.

  </Accordion>
</AccordionGroup>

## تأثیر بر توکن

وقتی Skills واجد شرایط باشد، OpenClaw یک بلوک XML فشرده را در پرامپت
سیستم تزریق می‌کند. هزینه قطعی است و به‌صورت خطی برای هر Skills افزایش می‌یابد:

- **سربار پایه** (فقط وقتی 1 یا چند Skills واجد شرایط باشد): یک بلوک ثابت از متن
  مقدماتی به‌همراه پوشش `<available_skills>`.
- **برای هر Skills:** حدود 97 نویسه + طول فیلدهای `name`، `description` و `location`
  شما.
- گریزدهی XML، ‏`& < > " '` را به موجودیت‌ها بسط می‌دهد و برای هر
  رخداد چند نویسه می‌افزاید.
- با حدود 4 نویسه به‌ازای هر توکن، 97 نویسه ≈ 24 توکن برای هر Skills پیش از احتساب طول فیلدها.

اگر بلوک رندرشده از بودجه پیکربندی‌شده پرامپت
(`skills.limits.maxSkillsPromptChars`) فراتر رود، OpenClaw ابتدا تا جای ممکن هویت‌های Skills
(نام، مکان و نسخه) را با قالب فشرده و بدون توضیحات حفظ می‌کند.
سپس بودجه باقی‌مانده را برای توضیحات کوتاه‌شده به‌کار می‌گیرد. اگر هیچ
بودجه‌ای برای توضیحات باقی نماند، توضیحات حذف می‌شوند. هرگاه قالب‌بندی فشرده یا
کوتاه‌سازی فهرست لازم باشد، پرامپت شامل یادداشتی است که به `openclaw skills check`
اشاره می‌کند.

برای به حداقل رساندن سربار پرامپت، توضیحات را کوتاه و گویا نگه دارید.

## مرتبط

<CardGroup cols={2}>
  <Card title="ایجاد Skills" href="/fa/tools/creating-skills" icon="hammer">
    راهنمای گام‌به‌گام نگارش یک skill سفارشی.
  </Card>
  <Card title="کارگاه Skills" href="/fa/tools/skill-workshop" icon="flask">
    صف پیشنهادها برای Skills پیش‌نویس‌شده توسط عامل.
  </Card>
  <Card title="پیکربندی Skills" href="/fa/tools/skills-config" icon="gear">
    طرح‌واره کامل پیکربندی `skills.*` و فهرست‌های مجاز عامل.
  </Card>
  <Card title="دستورهای اسلش" href="/fa/tools/slash-commands" icon="terminal">
    نحوه ثبت و مسیریابی دستورهای اسلش Skills.
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="cloud">
    Skills را در رجیستری عمومی مرور و منتشر کنید.
  </Card>
  <Card title="Pluginها" href="/fa/tools/plugin" icon="plug">
    Pluginها می‌توانند Skills را همراه ابزارهایی که مستندسازی می‌کنند ارائه دهند.
  </Card>
</CardGroup>
