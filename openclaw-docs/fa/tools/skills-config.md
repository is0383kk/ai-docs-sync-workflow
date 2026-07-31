---
read_when:
    - پیکربندی رفتار بارگذاری، نصب یا محدودسازی Skills
    - تنظیم قابلیت مشاهده Skills برای هر عامل
    - تنظیم محدودیت‌ها یا سیاست تأیید Skill Workshop
sidebarTitle: Skills config
summary: مرجع کامل طرح‌وارهٔ پیکربندی `skills.*`، فهرست‌های مجاز عامل، تنظیمات کارگاه و مدیریت متغیرهای محیطی سندباکس.
title: پیکربندی Skills
x-i18n:
    generated_at: "2026-07-27T14:52:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: bc154bdf8a8537095a4d39bc6e86ebfd716e6beacd45def9c8a1c15fcdc93698
    source_path: tools/skills-config.md
    workflow: 16
---

بیشتر پیکربندی Skills زیر `skills` در
`~/.openclaw/openclaw.json` قرار دارد. قابلیت مشاهدهٔ مختص هر عامل زیر
`agents.defaults.skills` و `agents.entries.*.skills` قرار دارد.

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
      watch: true,
    },
    install: {
      preferBrew: true,
      nodeManager: "npm",
      allowUploadedArchives: false,
    },
    workshop: {
      autonomous: { enabled: false },
      allowSymlinkTargetWrites: false,
      approvalPolicy: "auto",
      maxPending: 50,
      maxSkillBytes: 40000,
    },
    entries: {
      "image-lab": {
        enabled: true,
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

<Note>
  برای تولید تصویر داخلی، به‌جای `skills.entries` از `agents.defaults.mediaModels.image`
  به‌همراه ابزار اصلی `image_generate` استفاده کنید. ورودی‌های Skill
  فقط برای گردش‌کارهای سفارشی یا شخص ثالث Skill هستند.
</Note>

## بارگذاری (`skills.load`)

<ParamField path="skills.load.extraDirs" type="string[]">
  دایرکتوری‌های اضافی Skill برای اسکن، با کمترین اولویت (پایین‌تر از
  Skills همراه و Plugin). مسیرها با پشتیبانی از `~` گسترش می‌یابند.
</ParamField>

<ParamField path="skills.load.allowSymlinkTargets" type="string[]">
  دایرکتوری‌های مقصد واقعی و مورد اعتماد که پوشه‌های Skill دارای پیوند نمادین می‌توانند
  به آن‌ها تفکیک شوند، حتی وقتی پیوند نمادین خارج از ریشهٔ پیکربندی‌شده قرار دارد. از این گزینه برای
  چیدمان‌های عمدی مخزن‌های هم‌سطح، مانند
  `<workspace>/skills/manager -> ~/Projects/manager/skills`، استفاده کنید. این فهرست را
  محدود نگه دارید — به ریشه‌های گسترده‌ای مانند `~` یا `~/Projects` اشاره نکنید.
</ParamField>

<ParamField path="skills.load.watch" type="boolean" default="true">
  پوشه‌های Skill را پایش کنید و هنگام تغییر فایل‌های `SKILL.md`
  تصویر لحظه‌ای Skills را تازه‌سازی کنید. فایل‌های تودرتو زیر ریشه‌های گروه‌بندی‌شدهٔ Skill را نیز پوشش می‌دهد.
</ParamField>

## نصب (`skills.install`)

<ParamField path="skills.install.preferBrew" type="boolean" default="true">
  در صورت موجود بودن `brew`، نصب‌کننده‌های Homebrew را ترجیح دهید.
</ParamField>

<ParamField path="skills.install.nodeManager" type='"npm" | "pnpm" | "yarn" | "bun"' default='"npm"'>
  ترجیح مدیر بستهٔ Node برای نصب Skills. این گزینه فقط بر نصب
  Skills اثر می‌گذارد — CLI و محیط اجرای Gateway در OpenClaw به Node نیاز دارند، زیرا
  مخزن وضعیت استاندارد از `node:sqlite` استفاده می‌کند. `openclaw setup --node-manager` و
  `openclaw onboard --node-manager` مقادیر `npm`، `pnpm` یا `bun` را می‌پذیرند؛ برای نصب
  Skills مبتنی بر Yarn، مقدار `"yarn"` را مستقیماً در پیکربندی تنظیم کنید.
</ParamField>

<ParamField path="skills.install.allowUploadedArchives" type="boolean" default="false">
  به کلاینت‌های مورد اعتماد `operator.admin` در Gateway اجازه دهید بایگانی‌های zip خصوصی
  آماده‌شده از طریق `skills.upload.*` را نصب کنند. نصب‌های معمول ClawHub به
  این تنظیم نیاز ندارند.
</ParamField>

## خط‌مشی نصب اپراتور (`security.installPolicy`)

هنگامی که اپراتورها به یک فرمان محلی مورد اعتماد برای
تأیید یا مسدودسازی نصب Skills و Plugins با خط‌مشی مختص میزبان نیاز دارند، از `security.installPolicy` استفاده کنید.
خط‌مشی پس از آن اجرا می‌شود که OpenClaw محتوای منبع را آماده کرده و پیش از آن‌که نصب
یا به‌روزرسانی ادامه یابد. این خط‌مشی بر Skills مربوط به ClawHub، Skills بارگذاری‌شده، Skills مربوط به Git/محلی،
نصب‌کننده‌های وابستگی Skill و منابع نصب/به‌روزرسانی Plugin اعمال می‌شود.

```json5
{
  security: {
    installPolicy: {
      enabled: true,
      // برای پوشش دادن همهٔ مقصدهای پشتیبانی‌شده، targets را حذف کنید.
      targets: ["skill", "plugin"],
      exec: {
        source: "exec",
        command: "/usr/local/bin/openclaw-install-policy",
        args: ["--json"],
        timeoutMs: 10000,
        noOutputTimeoutMs: 10000,
        maxOutputBytes: 1048576,
        passEnv: ["OPENCLAW_STATE_DIR", "PATH"],
        env: { POLICY_MODE: "strict" },
        trustedDirs: ["/usr/local/bin"],
      },
    },
  },
}
```

<ParamField path="security.installPolicy.enabled" type="boolean" default="false">
  خط‌مشی نصب تحت مالکیت اپراتور را فعال می‌کند. وقتی بدون یک فرمان معتبر `exec`
  فعال شود، نصب‌ها به‌صورت بسته شکست می‌خورند.
</ParamField>

<ParamField path="security.installPolicy.targets" type='("skill" | "plugin")[]'>
  فیلتر اختیاری مقصد. وقتی حذف شود، خط‌مشی بر همهٔ
  مقصدهای پشتیبانی‌شده اعمال می‌شود تا نصب‌های جدید به‌طور غیرمنتظره به‌صورت باز شکست نخورند.
</ParamField>

<ParamField path="security.installPolicy.exec.command" type="string">
  مسیر مطلق فایل اجرایی مورد اعتماد خط‌مشی. OpenClaw آن را بدون
  پوسته اجرا می‌کند و مسیر را پیش از استفاده اعتبارسنجی می‌کند.
</ParamField>

<ParamField path="security.installPolicy.exec.args" type="string[]">
  آرگومان‌های ثابتی که پس از `command` ارسال می‌شوند.
</ParamField>

<ParamField path="security.installPolicy.exec.timeoutMs" type="number" default="10000">
  حداکثر زمان اجرا بر اساس ساعت دیواری برای یک تصمیم خط‌مشی.
</ParamField>

<ParamField path="security.installPolicy.exec.noOutputTimeoutMs" type="number" default="timeoutMs">
  حداکثر زمان بدون خروجی stdout یا stderr پیش از آن‌که خط‌مشی
  به‌صورت بسته شکست بخورد.
</ParamField>

<ParamField path="security.installPolicy.exec.maxOutputBytes" type="number" default="1048576">
  حداکثر مجموع بایت‌های stdout و stderr پذیرفته‌شده از فرایند خط‌مشی.
</ParamField>

<ParamField path="security.installPolicy.exec.env" type="Record<string, string>">
  متغیرهای محیطی لفظی ارائه‌شده به فرایند خط‌مشی.
</ParamField>

<ParamField path="security.installPolicy.exec.passEnv" type="string[]">
  نام متغیرهای محیطی که از فرایند OpenClaw به
  فرایند خط‌مشی کپی می‌شوند. فقط متغیرهای نام‌برده‌شده ارسال می‌شوند.
</ParamField>

<ParamField path="security.installPolicy.exec.trustedDirs" type="string[]">
  فهرست مجاز اختیاری دایرکتوری‌هایی که می‌توانند فایل اجرایی خط‌مشی را در خود داشته باشند.
</ParamField>

<ParamField path="security.installPolicy.exec.allowInsecurePath" type="boolean" default="false">
  بررسی‌های مالکیت و مجوز مسیر فرمان را دور می‌زند. فقط هنگامی استفاده کنید که
  مسیر با سازوکار دیگری محافظت می‌شود.
</ParamField>

<ParamField path="security.installPolicy.exec.allowSymlinkCommand" type="boolean" default="false">
  اجازه می‌دهد مسیر فرمان پیکربندی‌شده یک پیوند نمادین باشد. مقصد تفکیک‌شده
  همچنان باید سایر بررسی‌های مسیر را برآورده کند. آرگومان‌های اسکریپت مفسر باید
  فایل‌های عادی مستقیم باشند، نه پیوند نمادین.
</ParamField>

خط‌مشی یک شیء JSON را در stdin دریافت می‌کند که شامل `protocolVersion: 1`،
`openclawVersion`، `targetType`، `targetName`، `sourcePath`، `sourcePathKind`،
`source` ساختاریافتهٔ اختیاری، `origin` ساختاریافته و `request` است. باید
یک شیء JSON را در stdout بنویسد: `{ "protocolVersion": 1, "decision": "allow" }`
یا `{ "protocolVersion": 1, "decision": "block", "reason": "..." }`. خروج با کد غیرصفر،
پایان مهلت، JSON نادرست، فیلدهای مفقود یا نسخه‌های پروتکل پشتیبانی‌نشده
به‌صورت بسته شکست می‌خورند.

OpenClaw خط‌مشی نصب را هنگام راه‌اندازی عادی Gateway اجرا نمی‌کند.
وقتی خط‌مشی فعال اما در دسترس نباشد، نصب‌ها و به‌روزرسانی‌ها به‌صورت بسته شکست می‌خورند.
`openclaw doctor` اعتبارسنجی ایستا را انجام می‌دهد؛ `openclaw doctor --deep`
یک کاوش نصب مصنوعی را در برابر فرمان پیکربندی‌شده اجرا می‌کند.

به‌روزرسانی‌های انبوه خط‌مشی را برای هر مقصد اعمال می‌کنند: به‌روزرسانی مسدودشدهٔ یک Skill یا Plugin
برای همان مقصد شکست می‌خورد، بدون آن‌که خط‌مشی غیرفعال شود یا مقصدهای بعدی در
دسته نادیده گرفته شوند.

نمونهٔ stdin:

```json
{
  "protocolVersion": 1,
  "openclawVersion": "2026.6.1",
  "targetType": "skill",
  "targetName": "weather",
  "sourcePath": "/var/folders/.../openclaw-skill-clawhub/root",
  "sourcePathKind": "directory",
  "source": {
    "kind": "clawhub",
    "authority": "openclaw",
    "mutable": false,
    "network": true
  },
  "origin": {
    "type": "clawhub",
    "registry": "https://clawhub.openclaw.ai",
    "slug": "weather",
    "version": "1.0.0"
  },
  "request": {
    "kind": "skill-install",
    "mode": "install",
    "requestedSpecifier": "clawhub:weather@1.0.0"
  },
  "skill": {
    "installId": "clawhub"
  }
}
```

فرمان حداقلی خط‌مشی:

```js
#!/usr/bin/env node

let input = "";
process.stdin.setEncoding("utf8");
process.stdin.on("data", (chunk) => {
  input += chunk;
});
process.stdin.on("end", () => {
  const request = JSON.parse(input);
  if (request.targetType === "plugin" && request.source?.kind === "local-path") {
    process.stdout.write(
      JSON.stringify({
        protocolVersion: 1,
        decision: "block",
        reason: "مسیرهای محلی Plugin در این میزبان تأیید نشده‌اند",
      }),
    );
    return;
  }
  process.stdout.write(JSON.stringify({ protocolVersion: 1, decision: "allow" }));
});
```

## فهرست مجاز Skills همراه

<ParamField path="skills.allowBundled" type="string[]">
  فهرست مجاز اختیاری فقط برای Skills **همراه**. وقتی تنظیم شود، فقط Skills همراه
  موجود در فهرست واجد شرایط هستند. Skills مدیریت‌شده، سطح عامل و فضای کاری
  تحت تأثیر قرار نمی‌گیرند.
</ParamField>

## ورودی‌های هر Skill (`skills.entries`)

کلیدهای زیر `entries` به‌طور پیش‌فرض با `name` مربوط به Skill مطابقت دارند. اگر یک Skill
مقدار `metadata.openclaw.skillKey` را تعریف می‌کند، به‌جای آن از همان کلید استفاده کنید. نام‌های دارای خط تیره را
درون نقل‌قول قرار دهید (JSON5 کلیدهای نقل‌قول‌شده را می‌پذیرد).

<ParamField path="skills.entries.<key>.enabled" type="boolean">
  `false` حتی در صورت همراه یا نصب‌شده بودن Skill، آن را غیرفعال می‌کند.
  Skill همراه `coding-agent` نیازمند فعال‌سازی صریح است — آن را روی `true` تنظیم کنید و مطمئن شوید یکی از
  `claude`، `codex`، `opencode` یا یک CLI پشتیبانی‌شدهٔ دیگر نصب و
  احراز هویت شده است.
</ParamField>

<ParamField path="skills.entries.<key>.apiKey" type='string | { source, provider, id }'>
  فیلد کمکی برای Skills که `metadata.openclaw.primaryEnv` را اعلام می‌کنند.
  از یک رشتهٔ متن ساده یا SecretRef پشتیبانی می‌کند: `{ source: "env", provider: "default", id: "VAR_NAME" }`.
</ParamField>

<ParamField path="skills.entries.<key>.env" type="Record<string, string>">
  متغیرهای محیطی تزریق‌شده برای اجرای عامل. فقط زمانی تزریق می‌شوند که
  متغیر از قبل در فرایند تنظیم نشده باشد.
</ParamField>

<ParamField path="skills.entries.<key>.config" type="object">
  مجموعهٔ اختیاری فیلدهای پیکربندی سفارشی هر Skill.
</ParamField>

## فهرست‌های مجاز عامل (`agents`)

هنگامی از پیکربندی عامل استفاده کنید که می‌خواهید ریشه‌های Skill ماشین/فضای کاری یکسان باشند، اما
مجموعهٔ Skills قابل مشاهده برای هر عامل متفاوت باشد.

```json5
{
  agents: {
    defaults: {
      skills: ["github", "weather"], // خط پایهٔ مشترک
    },
    list: [
      { id: "writer" }, // github و weather را به ارث می‌برد
      { id: "docs", skills: ["docs-search"] }, // پیش‌فرض‌ها را کاملاً جایگزین می‌کند
      { id: "locked-down", skills: [] }, // بدون Skill
    ],
  },
}
```

<ParamField path="agents.defaults.skills" type="string[]">
  فهرست مجاز خط پایهٔ مشترک که عامل‌های فاقد
  `agents.entries.*.skills` آن را به ارث می‌برند. برای آن‌که Skills به‌طور
  پیش‌فرض نامحدود باقی بمانند، این گزینه را کاملاً حذف کنید.
</ParamField>

<ParamField path="agents.entries.*.skills" type="string[]">
  مجموعهٔ نهایی و صریح Skills برای آن عامل. فهرست‌های صریح، پیش‌فرض‌های
  به‌ارث‌رسیده را **جایگزین** می‌کنند — با آن‌ها ادغام نمی‌شوند. برای آن‌که هیچ Skill برای
  آن عامل نمایان نشود، مقدار را روی `[]` تنظیم کنید.
</ParamField>

<Warning>
  فهرست‌های مجاز Skills عامل، فیلتری برای قابلیت مشاهده و بارگذاری در کشف
  Skills توسط OpenClaw، اعلان‌ها، کشف فرمان‌های اسلش، همگام‌سازی sandbox و تصاویر لحظه‌ای
  Skills هستند. آن‌ها مرز مجوزدهی هنگام اجرای پوسته نیستند. اگر یک عامل
  بتواند `exec` میزبان را اجرا کند، آن پوسته همچنان می‌تواند کلاینت‌های خارجی را اجرا کند یا
  فایل‌های میزبان قابل مشاهده برای کاربر اجراکننده را بخواند، از جمله رجیستری‌های کلاینت
  MCP مانند `~/.openclaw/skills/config/mcporter.json`. برای
  جداسازی MCP در سطح هر عامل، فهرست‌های مجاز Skills را با جداسازی sandbox/کاربر سیستم‌عامل
  ترکیب کنید، اجرای میزبان را ممنوع یا به‌شدت محدود کنید و اعتبارنامه‌های مختص هر عامل
  را در سرور MCP ترجیح دهید.
</Warning>

## Workshop (`skills.workshop`)

<ParamField path="skills.workshop.autonomous.enabled" type="boolean" default="false">
  وقتی `true` باشد، OpenClaw می‌تواند از اصلاحات ماندگار پیشنهادهای در انتظار ایجاد کند
  و پس از بی‌کار شدن سیستم، کارهای تکمیل‌شدهٔ موفق، قابل‌توجه و اساسی را بازبینی کند.
  این قابلیت می‌تواند پس از نوبت‌های واجد شرایط، یک اجرای پس‌زمینه‌ای مدل اضافه کند. ایجاد Skill
  با درخواست کاربر و `/learn` هنگامی که تنظیم `false` است همچنان کار می‌کنند.
</ParamField>

برای معیارهای واجد شرایط بودن، حریم خصوصی، هزینه، مجوزهای صرفاً پیشنهادی
و عیب‌یابی، به [خودآموزی](/fa/tools/self-learning) مراجعه کنید.

<ParamField path="skills.workshop.approvalPolicy" type='"pending" | "auto"' default='"auto"'>
  `auto` اجازه می‌دهد عامل بدون درخواست تأیید اضافی، اعمال، رد یا قرنطینه را آغاز کند.
  `pending` به تأیید اپراتور نیاز دارد.
</ParamField>

<ParamField path="skills.workshop.allowSymlinkTargetWrites" type="boolean" default="false">
  به اعمال Skill Workshop اجازه می‌دهد از طریق پیوندهای نمادین Skill در فضای کاری بنویسد که
  هدف واقعی آن‌ها از قبل توسط `skills.load.allowSymlinkTargets` مورد اعتماد است. این گزینه را
  غیرفعال نگه دارید، مگر اینکه اعمال پیشنهادهای تولیدشده باید آن ریشهٔ مشترک
  Skill را تغییر دهد.
</ParamField>

<ParamField path="skills.workshop.maxPending" type="number" default="50">
  حداکثر تعداد پیشنهادهای در انتظار و قرنطینه‌شده که در هر فضای کاری نگه‌داری
  می‌شوند (محدودهٔ مجاز: 1-200).
</ParamField>

<ParamField path="skills.workshop.maxSkillBytes" type="number" default="40000">
  حداکثر اندازهٔ بدنهٔ پیشنهاد بر حسب بایت (محدودهٔ مجاز: 1024-200000). توضیحات
  پیشنهاد به‌طور جداگانه به‌صورت سخت‌گیرانه به 160 بایت محدود می‌شوند، زیرا
  در خروجی کشف و فهرست‌سازی ظاهر می‌شوند.
</ParamField>

برای چرخهٔ عمر پیشنهاد، فرمان‌های CLI، پارامترهای ابزار عامل و روش‌های Gateway
که این پیکربندی کنترل می‌کند، به [Skill Workshop](/fa/tools/skill-workshop) مراجعه کنید.

## ریشه‌های Skill دارای پیوند نمادین

به‌طور پیش‌فرض، ریشه‌های Skill مربوط به فضای کاری، عامل پروژه، دایرکتوری اضافی
و Skillهای همراه، مرزهای محصورسازی هستند. پوشهٔ Skill دارای پیوند نمادین زیر
`<workspace>/skills` که به بیرون از ریشه منتهی شود، همراه با یک پیام گزارش نادیده گرفته می‌شود.

برای مجاز کردن یک چیدمان عمدی پیوند نمادین، هدف مورد اعتماد را اعلام کنید:

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

با این پیکربندی، `<workspace>/skills/manager -> ~/Projects/manager/skills`
پس از تفکیک realpath پذیرفته می‌شود. `extraDirs` مخزن هم‌سطح را
مستقیماً اسکن می‌کند؛ `allowSymlinkTargets` مسیر دارای پیوند نمادین را برای
چیدمان‌های موجود حفظ می‌کند.

اعمال Skill Workshop به‌طور پیش‌فرض از طریق این پیوندهای نمادین نمی‌نویسد.
برای اینکه اعمال Workshop بتواند Skillهای زیر اهداف پیوند نمادینِ از قبل
مورد اعتماد را تغییر دهد، جداگانه آن را فعال کنید:

```json5
{
  skills: {
    load: {
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    workshop: {
      allowSymlinkTargetWrites: true,
    },
  },
}
```

دایرکتوری‌های مدیریت‌شدهٔ `~/.openclaw/skills` و شخصیِ `~/.agents/skills`
از قبل پیوندهای نمادین دایرکتوری Skill را بدون شرط می‌پذیرند (محصورسازی
`SKILL.md` برای هر Skill همچنان اعمال می‌شود) — `allowSymlinkTargets` فقط برای
ریشه‌های فضای کاری، دایرکتوری اضافی و عامل پروژه (`<workspace>/.agents/skills`)
لازم است.

## Skillهای سندباکس‌شده و متغیرهای محیطی

<Warning>
  `skills.entries.<skill>.env` و `apiKey` فقط برای اجراهای **میزبان** اعمال می‌شوند.
  درون سندباکس هیچ اثری ندارند — Skillی که به
  `GEMINI_API_KEY` وابسته است، با `apiKey not configured` شکست می‌خورد، مگر اینکه
  متغیر به‌طور جداگانه در اختیار سندباکس قرار گیرد.
</Warning>

اطلاعات محرمانه را به این شکل به یک سندباکس Docker منتقل کنید:

```json5
{
  agents: {
    defaults: {
      sandbox: {
        docker: {
          env: { GEMINI_API_KEY: "your-key-here" },
        },
      },
    },
  },
}
```

<Note>
  کاربرانی که به دیمن Docker دسترسی دارند، می‌توانند مقادیر `sandbox.docker.env`
  را از طریق فرادادهٔ Docker بررسی کنند. اگر این افشا قابل‌قبول نیست، از یک فایل
  محرمانهٔ سوارشده، یک ایمیج سفارشی یا مسیر تحویل دیگری استفاده کنید.
</Note>

## یادآوری ترتیب بارگذاری

```text
workspace/skills      (بالاترین)
workspace/.agents/skills
~/.agents/skills
~/.openclaw/skills
Skillهای همراه
skills.load.extraDirs (پایین‌ترین)
```

وقتی ناظر فعال باشد، تغییرات Skillها و پیکربندی در نشست جدید بعدی اعمال
می‌شوند؛ یا وقتی ناظر تغییری را تشخیص دهد، در نوبت بعدی عامل اعمال می‌شوند.

## مرتبط

<CardGroup cols={2}>
  <Card title="مرجع Skillها" href="/fa/tools/skills" icon="puzzle-piece">
    چیستی Skillها، ترتیب بارگذاری، کنترل دسترسی و قالب SKILL.md.
  </Card>
  <Card title="ایجاد Skillها" href="/fa/tools/creating-skills" icon="hammer">
    تألیف Skillهای سفارشی فضای کاری.
  </Card>
  <Card title="Skill Workshop" href="/fa/tools/skill-workshop" icon="flask">
    صف پیشنهاد برای Skillهای پیش‌نویس‌شده توسط عامل.
  </Card>
  <Card title="خودآموزی" href="/fa/tools/self-learning" icon="brain">
    پیشنهادهای محافظه‌کارانه و انتخابی از کارهای تکمیل‌شده.
  </Card>
  <Card title="فرمان‌های اسلش" href="/fa/tools/slash-commands" icon="terminal">
    کاتالوگ بومی فرمان‌های اسلش و دستورالعمل‌های چت.
  </Card>
</CardGroup>
