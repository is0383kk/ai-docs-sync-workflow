---
read_when:
    - به معناشناسی دقیق پیکربندی یا مقادیر پیش‌فرض در سطح فیلد نیاز دارید
    - در حال اعتبارسنجی بلوک‌های پیکربندی کانال، مدل، Gateway یا ابزار هستید
summary: مرجع پیکربندی Gateway برای کلیدهای اصلی OpenClaw، مقادیر پیش‌فرض و پیوندها به مراجع اختصاصی زیرسیستم‌ها
title: مرجع پیکربندی
x-i18n:
    generated_at: "2026-07-27T15:11:00Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7135554fda444fd1b8c072af5768c53a165f7be2dcd12a7909fc7fd4bd864428
    source_path: gateway/configuration-reference.md
    workflow: 16
---

مرجع سطح‌فیلد برای `~/.openclaw/openclaw.json`: کلیدها، مقادیر پیش‌فرض و پیوندها به صفحات عمیق‌تر زیرسامانه‌ها. برای راهنمای راه‌اندازی وظیفه‌محور، به [پیکربندی](/fa/gateway/configuration) مراجعه کنید. فهرست فرمان‌های متعلق به کانال‌ها و Pluginها و تنظیمات عمیق حافظه/QMD در صفحات خودشان قرار دارند، نه اینجا.

قالب پیکربندی **JSON5** است (استفاده از توضیحات و ویرگول انتهایی مجاز است). همه فیلدها اختیاری‌اند؛ در صورت حذف، OpenClaw از مقادیر پیش‌فرض امن استفاده می‌کند.

حقیقت کد بر این صفحه اولویت دارد:

- `openclaw config schema` شِمای JSON زنده‌ای را که برای اعتبارسنجی و Control UI استفاده می‌شود، با فراداده ادغام‌شده بسته‌ها/Pluginها/کانال‌ها چاپ می‌کند.
- عامل‌ها باید پیش از ویرایش پیکربندی، کنش ابزار `gateway` یعنی `config.schema.lookup` را برای یک گره دقیق و محدود به مسیر در شِما فراخوانی کنند.
- `pnpm config:docs:check` / `pnpm config:docs:gen` هش مبنای این سند را در برابر سطح فعلی شِما اعتبارسنجی می‌کنند.

شِمای `uiHints` همچنین برای هر مسیر یک مقدار بولی حل‌شده `advanced` دارد.
Control UI از آن برای نمایش ابتدا فیلدهای متداول و جمع‌کردن فیلدهای پیشرفته در هر
بخش استفاده می‌کند؛ جست‌وجو همچنان هر دو سطح را پوشش می‌دهد. فراداده سطح صرفاً نمایشی است.
هنگام افزودن یک کلید، سطح آن را روی برگ اعلام کنید یا اجازه دهید از نزدیک‌ترین
نیای دارای اعلام سطح به ارث ببرد. مسیری که هیچ نیای دارای اعلام سطح ندارد، به‌طور پیش‌فرض پیشرفته است.

مراجع عمیق اختصاصی:

- [مرجع پیکربندی حافظه](/fa/reference/memory-config) برای `memory.search.*`، `memory.qmd.*`، `memory.citations` و پیکربندی Dreaming زیر `plugins.entries.memory-core.config.dreaming`.
- [فرمان‌های اسلش](/fa/tools/slash-commands) برای فهرست فعلی فرمان‌های داخلی و بسته‌شده.
- صفحات مالک کانال/Plugin برای سطوح فرمان مختص هر کانال.

---

## کانال‌ها

کلیدهای پیکربندی هر کانال در [پیکربندی - کانال‌ها](/fa/gateway/config-channels) قرار دارند: `channels.*` برای Slack، Discord، Telegram، WhatsApp، Matrix، iMessage و دیگر کانال‌های بسته‌شده (احراز هویت، کنترل دسترسی، چندحسابی و الزام اشاره).

## پیش‌فرض‌های عامل، چندعاملی، نشست‌ها و پیام‌ها

برای موارد زیر به [پیکربندی - عامل‌ها](/fa/gateway/config-agents) مراجعه کنید:

- `agents.defaults.*` (فضای کاری، مدل، تفکر، Heartbeat، حافظه، رسانه، Skills، محیط ایزوله)
- `multiAgent.*` (مسیریابی و اتصال‌های چندعاملی)
- `session.*` (چرخه عمر نشست، Compaction، هرس)
- `messages.*` (تحویل پیام، تبدیل متن به گفتار، رندر Markdown)
- `talk.*` (حالت گفت‌وگو)
  - `talk.consultThinkingLevel`: بازنویسی سطح تفکر برای اجرای کامل عامل OpenClaw در پشت مشاوره‌های بلادرنگ گفت‌وگوی Control UI
  - `talk.consultFastMode`: بازنویسی یک‌باره حالت سریع برای مشاوره‌های بلادرنگ گفت‌وگوی Control UI
  - `talk.speechLocale`: شناسه محلی اختیاری BCP 47 برای تشخیص گفتار در حالت گفت‌وگو روی Android، iOS و macOS
  - `talk.silenceTimeoutMs`: وقتی تنظیم نشده باشد، حالت گفت‌وگو پیش از ارسال رونوشت، بازه مکث پیش‌فرض پلتفرم را حفظ می‌کند (`700 ms on macOS and Android, 900 ms on iOS`)
  - `talk.realtime.consultRouting`: مسیر جایگزین رله Gateway برای رونوشت‌های نهایی‌شده بلادرنگ حالت گفت‌وگو که `openclaw_agent_consult` را رد می‌کنند

## ابزارها و ارائه‌دهندگان سفارشی

سیاست ابزار، گزینه‌های آزمایشی، پیکربندی ابزارهای متکی به ارائه‌دهنده و راه‌اندازی
ارائه‌دهنده سفارشی / URL پایه در
[پیکربندی - ابزارها و ارائه‌دهندگان سفارشی](/fa/gateway/config-tools) قرار دارند.

## مدل‌ها

تعریف ارائه‌دهندگان، فهرست‌های مجاز مدل و راه‌اندازی ارائه‌دهنده سفارشی در
[پیکربندی - ابزارها و ارائه‌دهندگان سفارشی](/fa/gateway/config-tools#custom-providers-and-base-urls) قرار دارند.
ریشه `models` همچنین رفتار سراسری فهرست مدل‌ها را مدیریت می‌کند.

```json5
{
  models: {
    // اختیاری. پیش‌فرض: true. پس از تغییر به راه‌اندازی مجدد Gateway نیاز دارد.
    pricing: { enabled: false },
  },
}
```

- `models.mode`: رفتار فهرست ارائه‌دهنده (`merge` یا `replace`).
- `models.providers`: نگاشت ارائه‌دهنده سفارشی با کلید شناسه ارائه‌دهنده.
- `models.providers.*.localService`: مدیر فرایند اختیاری و درخواستی برای
  سرورهای مدل محلی. OpenClaw نقطه پایانی سلامت پیکربندی‌شده را بررسی می‌کند، در صورت
  نیاز `command` مطلق را راه‌اندازی می‌کند، منتظر آماده‌شدن می‌ماند و سپس درخواست
  مدل را ارسال می‌کند. به [سرویس‌های مدل محلی](/fa/gateway/local-model-services) مراجعه کنید.
- `models.pricing.enabled`: راه‌اندازی اولیه قیمت‌گذاری پس‌زمینه را کنترل می‌کند که
  پس از رسیدن فرایندهای جانبی و کانال‌ها به مسیر آماده Gateway آغاز می‌شود. وقتی `false` باشد،
  Gateway واکشی فهرست قیمت OpenRouter و LiteLLM را رد می‌کند؛ مقادیر پیکربندی‌شده
  `models.providers.*.models[].cost` همچنان برای برآورد هزینه محلی کار می‌کنند.

## MCP

تعریف سرورهای MCP مدیریت‌شده توسط OpenClaw زیر `mcp.servers` قرار دارند و
OpenClaw تعبیه‌شده و دیگر سازگارکننده‌های زمان اجرا آن‌ها را مصرف می‌کنند. فرمان‌های `openclaw mcp list`،
`show`، `set` و `unset` این بلوک را بدون اتصال به
سرور مقصد هنگام ویرایش پیکربندی مدیریت می‌کنند.

```json5
{
  mcp: {
    servers: {
      docs: {
        command: "npx",
        args: ["-y", "@modelcontextprotocol/server-fetch"],
      },
      remote: {
        url: "https://example.com/mcp",
        transport: "streamable-http", // streamable-http | sse
        requestTimeoutMs: 20000,
        connectionTimeoutMs: 5000,
        supportsParallelToolCalls: true,
        headers: {
          Authorization: "Bearer ${MCP_REMOTE_TOKEN}",
        },
        auth: "oauth",
        oauth: {
          scope: "docs.read",
        },
        sslVerify: true,
        clientCert: "/path/to/client.crt",
        clientKey: "/path/to/client.key",
        toolFilter: {
          include: ["search_*"],
          exclude: ["admin_*"],
        },
        // کنترل‌های اختیاری نگاشت app-server در Codex.
        codex: {
          agents: ["main"],
          defaultToolsApprovalMode: "approve", // auto | prompt | approve
        },
      },
    },
  },
}
```

- `mcp.servers`: تعریف سرورهای نام‌گذاری‌شده stdio یا MCP راه‌دور برای زمان‌های اجرایی که
  ابزارهای MCP پیکربندی‌شده را ارائه می‌کنند.
  ورودی‌های راه‌دور از `transport: "streamable-http"` یا `transport: "sse"` استفاده می‌کنند؛
  `type: "http"` نام مستعار بومی CLI است که `openclaw mcp set` و
  `openclaw doctor --fix` آن را به فیلد معیار `transport` نرمال‌سازی می‌کنند.
- `mcp.servers.<name>.enabled`: برای حفظ تعریف ذخیره‌شده سرور و درعین‌حال
  حذف آن از کشف MCP و نگاشت ابزار در OpenClaw تعبیه‌شده، `false` را تنظیم کنید.
- `mcp.servers.<name>.requestTimeoutMs`: مهلت زمانی درخواست MCP برای هر سرور برحسب میلی‌ثانیه.
- `mcp.servers.<name>.connectionTimeoutMs`: مهلت زمانی اتصال برای هر سرور برحسب میلی‌ثانیه.
- `mcp.servers.<name>.supportsParallelToolCalls`: راهنمای اختیاری هم‌زمانی برای
  سازگارکننده‌هایی که می‌توانند درباره صدور موازی فراخوانی‌های ابزار MCP تصمیم بگیرند.
- `mcp.servers.<name>.auth`: برای سرورهای HTTP MCP که به
  OAuth نیاز دارند، `"oauth"` را تنظیم کنید. برای ذخیره توکن‌ها در وضعیت OpenClaw، `openclaw mcp login <name>` را اجرا کنید.
- `mcp.servers.<name>.oauth`: بازنویسی اختیاری دامنه OAuth، نشانی هدایت مجدد و URL
  فراداده کارخواه.
- `mcp.servers.<name>.sslVerify`، `clientCert`، `clientKey`: کنترل‌های TLS در HTTP
  برای نقاط پایانی خصوصی و TLS متقابل.
- `mcp.servers.<name>.toolFilter`: انتخاب اختیاری ابزار برای هر سرور. `include`
  ابزارهای MCP کشف‌شده را به نام‌های منطبق محدود می‌کند؛ `exclude` نام‌های منطبق
  را پنهان می‌کند. ورودی‌ها نام دقیق ابزارهای MCP یا الگوهای ساده `*` هستند. سرورهای دارای
  منابع یا اعلان‌ها همچنین نام ابزارهای کمکی (`resources_list`،
  `resources_read`، `prompts_list`، `prompts_get`) را تولید می‌کنند و همان
  پالایه بر این نام‌ها نیز اعمال می‌شود.
- `mcp.servers.<name>.codex`: کنترل‌های اختیاری نگاشت app-server در Codex.
  این بلوک فقط فراداده OpenClaw برای رشته‌های app-server در Codex است و بر
  نشست‌های ACP، پیکربندی عمومی چارچوب Codex یا دیگر سازگارکننده‌های زمان اجرا
  اثری ندارد.
  `codex.agents` غیرخالی، سرور را به شناسه‌های عامل OpenClaw فهرست‌شده محدود می‌کند.
  فهرست‌های عامل محدودشده خالی، سفید یا نامعتبر توسط اعتبارسنجی پیکربندی رد می‌شوند
  و مسیر نگاشت زمان اجرا به‌جای سراسری‌کردن آن‌ها، حذفشان می‌کند.
  `codex.defaultToolsApprovalMode` مقدار بومی Codex یعنی
  `default_tools_approval_mode` را برای آن سرور تولید می‌کند. OpenClaw پیش از ارسال پیکربندی بومی
  `mcp_servers` به Codex، بلوک `codex` را حذف می‌کند. برای حفظ
  نگاشت سرور برای همه عامل‌های app-server در Codex با رفتار پیش‌فرض تأیید MCP
  در Codex، این بلوک را حذف کنید.
- زمان‌های اجرای MCP بسته‌شده و محدود به نشست، از TTL داخلی 10 دقیقه‌ای برای بیکاری استفاده می‌کنند.
  اجراهای تعبیه‌شده یک‌باره در پایان اجرا درخواست پاک‌سازی می‌کنند؛ TTL پشتیبان نشست‌های طولانی‌مدت و فراخوان‌های آینده است.
- تغییرات زیر `mcp.*` با دورریختن زمان‌های اجرای MCP ذخیره‌شده نشست، بی‌درنگ اعمال می‌شوند.
  کشف/استفاده بعدی ابزار، آن‌ها را از پیکربندی جدید دوباره ایجاد می‌کند؛ بنابراین ورودی‌های حذف‌شده
  `mcp.servers` به‌جای انتظار برای TTL بیکاری، فوراً جمع‌آوری می‌شوند.
- کشف زمان اجرا همچنین با حذف فهرست ذخیره‌شده آن نشست، اعلان‌های تغییر فهرست ابزار MCP
  را رعایت می‌کند. سرورهایی که منابع یا اعلان‌ها را معرفی می‌کنند، ابزارهای کمکی
  برای فهرست‌کردن/خواندن منابع و فهرست‌کردن/واکشی اعلان‌ها دریافت می‌کنند.
  شکست‌های مکرر فراخوانی ابزار، سرور درگیر را پیش از تلاش فراخوانی بعدی برای مدت کوتاهی متوقف می‌کند.

برای رفتار زمان اجرا به [MCP](/fa/cli/mcp#openclaw-as-an-mcp-client-registry) و
[پشتانه‌های CLI](/fa/gateway/cli-backends#bundle-mcp-overlays) مراجعه کنید.

## Skills

```json5
{
  skills: {
    allowBundled: ["gemini", "peekaboo"],
    load: {
      extraDirs: ["~/Projects/agent-scripts/skills"],
      allowSymlinkTargets: ["~/Projects/manager/skills"],
    },
    install: {
      preferBrew: true,
      nodeManager: "npm", // npm | pnpm | yarn | bun
      allowUploadedArchives: false,
    },
    workshop: {
      allowSymlinkTargetWrites: false,
    },
    entries: {
      "image-lab": {
        apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" }, // یا رشته متن ساده
        env: { GEMINI_API_KEY: "GEMINI_KEY_HERE" },
      },
      peekaboo: { enabled: true },
      sag: { enabled: false },
    },
  },
}
```

- `allowBundled`: فهرست مجاز اختیاری فقط برای Skills بسته‌شده (Skills مدیریت‌شده/فضای کاری بی‌تأثیرند).
- `load.extraDirs`: ریشه‌های مشترک اضافی Skills (کمترین اولویت).
- `load.allowSymlinkTargets`: ریشه‌های واقعی و مورداعتماد مقصد که پیوندهای نمادین Skills می‌توانند
  وقتی پیوند خارج از ریشه منبع پیکربندی‌شده قرار دارد، به آن‌ها ختم شوند.
- `workshop.allowSymlinkTargetWrites`: به اعمال Skill Workshop اجازه می‌دهد از طریق
  مقصدهای ازپیش‌مورداعتماد پیوندهای نمادین بنویسد (پیش‌فرض: false).
- `install.preferBrew`: وقتی true باشد و `brew` در دسترس باشد، پیش از
  بازگشت به دیگر انواع نصب‌کننده، نصب‌کننده‌های Homebrew ترجیح داده می‌شوند.
- `install.nodeManager`: ترجیح نصب‌کننده Node برای مشخصات `metadata.openclaw.install`
  (`npm` | `pnpm` | `yarn` | `bun`).
- `install.allowUploadedArchives`: به کارخواه‌های مورداعتماد `operator.admin` در Gateway
  اجازه می‌دهد بایگانی‌های zip خصوصی آماده‌شده از طریق `skills.upload.*` را نصب کنند
  (پیش‌فرض: false). این فقط مسیر بایگانی بارگذاری‌شده را فعال می‌کند؛ نصب‌های عادی ClawHub
  به آن نیازی ندارند.
- `entries.<skillKey>.enabled: false` یک Skill را حتی اگر بسته‌شده/نصب‌شده باشد غیرفعال می‌کند.
- `entries.<skillKey>.apiKey`: میان‌بری برای Skills که یک متغیر محیطی اصلی اعلام می‌کنند (رشته متن ساده یا شیء SecretRef).
- `limits.maxCandidatesPerRoot`، `limits.maxSkillsLoadedPerSource`، `limits.maxSkillsInPrompt`، `limits.maxSkillsPromptChars`، `limits.maxSkillFileBytes`: کشف Skills و اعلان Skills روبه‌مدل را محدود می‌کنند.
- تنظیمات خودمختاری/تأیید Skill Workshop (`workshop.autonomous.enabled`، `workshop.approvalPolicy`، `workshop.maxPending`، `workshop.maxSkillBytes`) در [پیکربندی Skills](/fa/tools/skills-config) مستند شده‌اند.

---

## Pluginها

```json5
{
  plugins: {
    enabled: true,
    allow: ["voice-call"],
    deny: [],
    load: {
      paths: ["~/Projects/oss/voice-call-plugin"],
    },
    entries: {
      "voice-call": {
        enabled: true,
        hooks: {
          allowPromptInjection: false,
        },
        config: { provider: "twilio" },
      },
    },
  },
}
```

- از دایرکتوری‌های بسته یا باندل زیر `~/.openclaw/extensions` و `<workspace>/.openclaw/extensions`، به‌علاوه فایل‌ها یا دایرکتوری‌های فهرست‌شده در `plugins.load.paths` بارگذاری می‌شود.
- فایل‌های مستقل plugin را در `plugins.load.paths` قرار دهید؛ ریشه‌های افزونه‌ای که به‌طور خودکار کشف می‌شوند، فایل‌های سطح‌بالای `.js`، `.mjs` و `.ts` را نادیده می‌گیرند تا اسکریپت‌های کمکی در آن ریشه‌ها مانع راه‌اندازی نشوند.
- فرایند کشف، pluginهای بومی OpenClaw و نیز باندل‌های سازگار Codex و Claude، از جمله باندل‌های بدون مانیفست Claude با چیدمان پیش‌فرض را می‌پذیرد.
- **تغییرات پیکربندی به راه‌اندازی مجدد Gateway نیاز دارند.**
- `allow`: فهرست مجاز اختیاری (فقط pluginهای فهرست‌شده بارگذاری می‌شوند). `deny` اولویت دارد.
- `plugins.entries.<id>.apiKey`: فیلد کمکی کلید API در سطح plugin (هنگامی که plugin از آن پشتیبانی کند).
- `plugins.entries.<id>.env`: نگاشت متغیرهای محیطی با دامنه plugin.
- `plugins.entries.<id>.hooks.allowPromptInjection`: هنگامی که `false` باشد، هسته هوک‌های تغییردهنده پرامپت مانند `before_prompt_build` را مسدود می‌کند. این مورد بر هوک‌های بومی plugin و دایرکتوری‌های هوک ارائه‌شده توسط باندل‌های پشتیبانی‌شده اعمال می‌شود.
- `plugins.entries.<id>.hooks.allowConversationAccess`: هنگامی که `true` باشد، pluginهای مورداعتماد و غیرباندل‌شده می‌توانند محتوای خام مکالمه را از هوک‌های نوع‌دار مانند `llm_input`، `llm_output`، `before_model_resolve`، `before_agent_reply`، `before_agent_run`، `before_agent_finalize` و `agent_end` بخوانند.
- `plugins.entries.<id>.subagent.allowModelOverride`: به‌طور صریح به این plugin اعتماد کنید تا برای اجرای پس‌زمینه زیرعامل‌ها، بازنویسی‌های مختص هر اجرا برای `provider` و `model` درخواست کند.
- `plugins.entries.<id>.subagent.allowedModels`: فهرست مجاز اختیاری از مقصدهای متعارف `provider/model` برای بازنویسی‌های مورداعتماد زیرعامل. فقط زمانی از `"*"` استفاده کنید که عمداً می‌خواهید هر مدلی مجاز باشد.
- `plugins.entries.<id>.llm.allowModelOverride`: به‌طور صریح به این plugin اعتماد کنید تا برای `api.runtime.llm.complete` بازنویسی مدل درخواست کند.
- `plugins.entries.<id>.llm.allowedModels`: فهرست مجاز اختیاری از مقصدهای متعارف `provider/model` برای بازنویسی‌های مورداعتماد تکمیل LLM توسط plugin. فقط زمانی از `"*"` استفاده کنید که عمداً می‌خواهید هر مدلی مجاز باشد.
- `plugins.entries.<id>.llm.allowAgentIdOverride`: به‌طور صریح به این plugin اعتماد کنید تا `api.runtime.llm.complete` را برای شناسه عاملی غیر از عامل پیش‌فرض اجرا کند.
- `plugins.entries.<id>.config`: شیء پیکربندی تعریف‌شده توسط plugin (در صورت وجود، با طرح‌واره بومی plugin در OpenClaw اعتبارسنجی می‌شود).
- تنظیمات حساب و زمان اجرای plugin کانال در `channels.<id>` قرار دارند و باید با فراداده `channelConfigs` مانیفست plugin مالک توصیف شوند، نه با رجیستری مرکزی گزینه‌های OpenClaw.

### پیکربندی plugin هارنس Codex

plugin باندل‌شده `codex` مالک تنظیمات هارنس بومی app-server در Codex تحت
`plugins.entries.codex.config` است. برای سطح کامل پیکربندی به
[مرجع هارنس Codex](/fa/plugins/codex-harness-reference) و برای مدل زمان اجرا به
[هارنس Codex](/fa/plugins/codex-harness) مراجعه کنید.

`codexPlugins` فقط برای نشست‌هایی اعمال می‌شود که هارنس بومی Codex را انتخاب می‌کنند.
این گزینه pluginهای Codex را برای اجراهای ارائه‌دهنده OpenClaw، اتصال‌های مکالمه
ACP یا هیچ هارنس غیر Codex دیگری فعال نمی‌کند.

```json5
{
  plugins: {
    entries: {
      codex: {
        enabled: true,
        config: {
          codexPlugins: {
            enabled: true,
            allow_all_plugins: true,
            allow_destructive_actions: "auto",
            plugins: {
              "google-calendar": {
                enabled: true,
                marketplaceName: "openai-curated",
                pluginName: "google-calendar",
                allow_destructive_actions: false,
              },
            },
          },
        },
      },
    },
  },
}
```

- `plugins.entries.codex.config.codexPlugins.enabled`: پشتیبانی بومی
  plugin/برنامه Codex را برای هارنس Codex فعال می‌کند. پیش‌فرض: `false`.
- `plugins.entries.codex.config.codexPlugins.allow_all_plugins`: تمام
  برنامه‌های در حال حاضر قابل‌دسترسی و متصل به حساب احرازهویت‌شده Codex را در
  هر رشته بومی جدید Codex در دسترس قرار می‌دهد. پیش‌فرض: `false`.
- `plugins.entries.codex.config.codexPlugins.allow_destructive_actions`:
  سیاست پیش‌فرض اقدامات مخرب برای درخواست‌های تعاملی برنامه‌های plugin پیکربندی‌شده.
  از `true` برای پذیرش طرح‌واره‌های ایمن تأیید Codex بدون نمایش درخواست، از `false`
  برای رد آن‌ها، از `"auto"` برای هدایت تأییدهای موردنیاز Codex از طریق تأییدهای
  plugin در OpenClaw، یا از `"ask"` برای نمایش درخواست در هر اقدام نوشتنی/مخرب
  plugin بدون تأیید ماندگار استفاده کنید. حالت `"ask"` بازنویسی‌های ماندگار
  تأیید مختص هر ابزار Codex را برای برنامه مربوط پاک می‌کند و پیش از آغاز رشته Codex،
  بازبین انسانی تأییدها را برای آن برنامه انتخاب می‌کند.
  پیش‌فرض: `true`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.enabled`: هنگامی که
  `codexPlugins.enabled` سراسری نیز true باشد، یک ورودی plugin پیکربندی‌شده را فعال می‌کند.
  پیش‌فرض برای ورودی‌های صریح: `true`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.marketplaceName`:
  هویت پایدار بازار، که همراه با `pluginName` برای هر ورودی تفکیک‌شده الزامی است.
  از `"openai-curated"` و `"workspace-directory"` پشتیبانی می‌کند. ورودی‌هایی که
  یکی از این دو فیلد هویت را نداشته باشند، نادیده گرفته می‌شوند.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.pluginName`: هویت پایدار
  plugin در Codex، که همراه با `marketplaceName` الزامی است. یک
  ورودی `workspace-directory` باید دقیقاً از `summary.id` واجد نام بازار
  که `plugin/list` برمی‌گرداند استفاده کند؛ برای مثال
  `"example-plugin@workspace-directory"`.
- `plugins.entries.codex.config.codexPlugins.plugins.<key>.allow_destructive_actions`:
  بازنویسی اقدام مخرب مختص هر plugin. در صورت حذف آن، مقدار سراسری
  `allow_destructive_actions` استفاده می‌شود. مقدار مختص هر plugin همان سیاست‌های
  `true`، `false`، `"auto"` یا `"ask"` را می‌پذیرد.

هر برنامه plugin پذیرفته‌شده‌ای که از `"ask"` استفاده می‌کند، درخواست‌های تأیید آن برنامه را
به بازبین انسانی هدایت می‌کند. سایر برنامه‌ها و تأییدهای رشته‌ای غیرمرتبط با برنامه، بازبین
پیکربندی‌شده خود را حفظ می‌کنند؛ بنابراین سیاست‌های ترکیبی plugin، رفتار `"ask"` را به ارث نمی‌برند.

`codexPlugins.enabled` دستور فعال‌سازی سراسری است. ورودی‌های صریح plugin
که مهاجرت ایجاد می‌کند، مجموعه ماندگار واجد شرایط برای نصب گزینش‌شده و تعمیر هستند.
ورودی‌های `workspace-directory` که به‌صورت دستی پیکربندی شده‌اند باید از قبل
نصب و فعال باشند و برنامه‌های تحت مالکیتشان نیز باید قابل‌دسترسی باشند؛ OpenClaw
آن‌ها را نصب یا احراز هویت نمی‌کند. اگر Codex درخواست صریح کاتالوگ فضای کاری را
رد کند، ورودی‌های فعال فضای کاری با `marketplace_missing` به‌صورت بسته شکست می‌خورند،
درحالی‌که ورودی‌های گزینش‌شده از کاتالوگ پیش‌فرض همچنان در دسترس می‌مانند.
`plugins["*"]` پشتیبانی نمی‌شود، هیچ کلید `install` وجود ندارد و
مقادیر محلی `marketplacePath` عمداً فیلد پیکربندی نیستند، زیرا به میزبان وابسته‌اند. برای
نیازمندی‌های نسخه app-server و آمادگی، به
[pluginهای بومی Codex](/fa/plugins/codex-native-plugins) مراجعه کنید.

بررسی‌های آمادگی `app/list` به‌مدت یک ساعت در حافظه نهان نگه‌داری می‌شوند و
پس از کهنه‌شدن به‌صورت ناهمگام تازه‌سازی می‌شوند. پیکربندی برنامه رشته Codex هنگام
برقراری نشست هارنس Codex محاسبه می‌شود، نه در هر نوبت؛ پس از تغییر پیکربندی plugin بومی،
از `/new`، `/reset` یا راه‌اندازی مجدد Gateway استفاده کنید.

`codexPlugins.allow_all_plugins` از همه برنامه‌های حساب که در حال حاضر قابل‌دسترسی‌اند
در هر رشته بومی جدید Codex یک تصویر لحظه‌ای ثبت می‌کند. این گزینه plugin یا برنامه‌ای نصب نمی‌کند و
برنامه‌های غیرقابل‌دسترسی همچنان کنار گذاشته می‌شوند. برنامه‌های حساب از سیاست سراسری
`codexPlugins.allow_destructive_actions` استفاده می‌کنند. اگر یک برنامه در هر دو مسیر موجود باشد،
ورودی‌های صریح plugin اولویت دارند. اگر `app/list` قابل خواندن نباشد،
دسترسی سراسری حساب به‌صورت بسته شکست می‌خورد.

- `plugins.entries.firecrawl.config.webFetch`: تنظیمات ارائه‌دهنده واکشی وب Firecrawl.
  - `apiKey`: کلید API اختیاری Firecrawl برای محدودیت‌های بالاتر (SecretRef را می‌پذیرد). در صورت نبود، از متغیر محیطی `plugins.entries.firecrawl.config.webSearch.apiKey` یا `FIRECRAWL_API_KEY` استفاده می‌کند.
  - `baseUrl`: نشانی URL پایه API در Firecrawl (پیش‌فرض: `https://api.firecrawl.dev`؛ بازنویسی‌های خودمیزبان باید به نقاط پایانی خصوصی/داخلی اشاره کنند).
  - `onlyMainContent`: فقط محتوای اصلی صفحه‌ها را استخراج می‌کند (پیش‌فرض: `true`).
  - `maxAgeMs`: حداکثر عمر حافظه نهان برحسب میلی‌ثانیه (پیش‌فرض: `172800000` / 2 روز).
  - `timeoutSeconds`: مهلت زمانی درخواست خزش برحسب ثانیه (پیش‌فرض: `60`).
- `plugins.entries.xai.config.xSearch`: تنظیمات X Search در xAI (جست‌وجوی وب Grok).
  - `enabled`: ارائه‌دهنده X Search را فعال می‌کند.
  - `model`: مدل Grok مورد استفاده برای جست‌وجو (برای مثال `"grok-4.3"`).
- `plugins.entries.memory-core.config.dreaming`: تنظیمات Dreaming حافظه. برای مراحل و آستانه‌ها به [Dreaming](/fa/concepts/dreaming) مراجعه کنید.
  - `enabled`: کلید اصلی Dreaming (پیش‌فرض `false`).
  - `frequency`: تناوب Cron برای هر پیمایش کامل Dreaming (به‌طور پیش‌فرض `"0 3 * * *"`).
  - `model`: بازنویسی اختیاری مدل زیرعامل Dream Diary. به `plugins.entries.memory-core.subagent.allowModelOverride: true` نیاز دارد؛ برای محدودکردن مقصدها آن را با `allowedModels` همراه کنید. خطاهای دردسترس‌نبودن مدل یک‌بار دیگر با مدل پیش‌فرض نشست تلاش می‌شوند؛ خطاهای اعتماد یا فهرست مجاز بدون اعلام به مسیر جایگزین نمی‌روند.
  - سیاست مراحل و آستانه‌ها جزئیات پیاده‌سازی هستند (نه کلیدهای پیکربندی قابل‌مشاهده برای کاربر).
- پیکربندی کامل حافظه در [مرجع پیکربندی حافظه](/fa/reference/memory-config) قرار دارد:
  - `memory.search.*`
  - `agents.entries.*.memory.search.*` برای بازنویسی‌های مختص هر عامل
  - `memory.backend`
  - `memory.citations`
  - `memory.qmd.*`
  - `plugins.entries.memory-core.config.dreaming`
- pluginهای فعال باندل Claude همچنین می‌توانند پیش‌فرض‌های تعبیه‌شده OpenClaw را از `settings.json` ارائه کنند؛ OpenClaw آن‌ها را به‌عنوان تنظیمات پاک‌سازی‌شده عامل اعمال می‌کند، نه وصله‌های خام پیکربندی OpenClaw.
- `plugins.slots.memory`: شناسه plugin فعال حافظه را انتخاب کنید، یا برای غیرفعال‌کردن pluginهای حافظه از `"none"` استفاده کنید.
- `plugins.slots.contextEngine`: شناسه plugin فعال موتور زمینه را انتخاب کنید؛ مگر اینکه موتور دیگری نصب و انتخاب شود، مقدار پیش‌فرض `"legacy"` است.

به [Pluginها](/fa/tools/plugin) مراجعه کنید.

---

## مرورگر

```json5
{
  browser: {
    enabled: true,
    evaluateEnabled: true,
    defaultProfile: "user",
    ssrfPolicy: {
      // dangerouslyAllowPrivateNetwork: true, // فقط برای دسترسی مورداعتماد به شبکه خصوصی، آن را به‌صورت صریح فعال کنید
      // allowPrivateNetwork: true, // نام مستعار قدیمی
      // hostnameAllowlist: ["*.example.com", "example.com"],
      // allowedHostnames: ["localhost"],
    },
    tabCleanup: {
      enabled: true,
      idleMinutes: 120,
      maxTabsPerSession: 8,
      sweepMinutes: 5,
    },
    profiles: {
      openclaw: { cdpPort: 18800, color: "#FF4500" },
      work: {
        cdpPort: 18801,
        color: "#0066CC",
        executablePath: "/Applications/Google Chrome.app/Contents/MacOS/Google Chrome",
      },
      user: { driver: "existing-session", attachOnly: true, color: "#00AA00" },
      brave: {
        driver: "existing-session",
        attachOnly: true,
        userDataDir: "~/Library/Application Support/BraveSoftware/Brave-Browser",
        color: "#FB542B",
      },
      remote: { cdpUrl: "http://10.0.0.42:9222", color: "#00AA00" },
    },
    color: "#FF4500",
    // headless: false,
    // noSandbox: false,
    // extraArgs: [],
    // executablePath: "/Applications/Brave Browser.app/Contents/MacOS/Brave Browser",
    // attachOnly: false,
  },
}
```

- `evaluateEnabled: false`، `act:evaluate` و `wait --fn` را غیرفعال می‌کند.
- `tabCleanup` پاک‌سازی دوره‌ایِ مبتنی بر بهترین تلاش را برای زبانه‌های عامل اصلیِ ردیابی‌شده،
  پس از زمان بی‌کاری یا زمانی که یک نشست از سقف خود فراتر می‌رود، کنترل می‌کند. ردیابی فقط
  برای زبانه‌هایی اعمال می‌شود که توسط ابزار مرورگر `action: "open"` ایجاد شده‌اند؛ زبانه‌هایی که کاربر باز کرده
  یا مالکیت نامشخصی دارند هرگز تحت مالکیت گرفته نمی‌شوند. غیرفعال‌کردن `tabCleanup`، پاک‌سازی صریح چرخهٔ عمر نشست را غیرفعال نمی‌کند.
- بازشدن‌های محلیِ میزبان با هدف بومی و پایدار CDP و هویت مرورگر،
  در وضعیت مشترک SQLite ذخیره می‌شوند و پس از راه‌اندازی مجدد Gateway همچنان برای
  `/new` و پاک‌سازی چرخهٔ عمر نشست واجد شرایط می‌مانند. هدف‌های بومی CDP که در معرض ابزار قرار دارند نیز
  پس از راه‌اندازی مجدد همچنان برای پاک‌سازی بر اساس بی‌کاری و سقف واجد شرایط می‌مانند. Chrome MCP از
  هندل‌های هدفِ محلیِ فرایند استفاده می‌کند، بنابراین رکوردهای سردِ نشست‌های موجود به‌جای
  به‌خطرانداختن جاروب بی‌کاری در برابر فعالیت غیرقابل‌انتساب پس از راه‌اندازی مجدد،
  منتظر پاک‌سازی چرخهٔ عمر می‌مانند. OpenClaw پیش از بستن، پروفایل و نمونهٔ مرورگر را
  تأیید می‌کند. اتصال خودکار Chrome MCP، نبود هویت مرورگر `/json/version`،
  و هدف‌های بومیِ حل‌نشده کاملاً محلیِ فرایند باقی می‌مانند، بنابراین
  پس از راه‌اندازی مجدد به‌طور خودکار بسته نمی‌شوند. زبانه‌های قدیمی‌ترِ ردیابی‌نشده به
  بسته‌شدن دستی نیاز دارند. خطاهای موقت برای تلاش مجدد در آینده در حالت انتظار می‌مانند. به
  [مالکیت پاک‌سازی زبانه](/fa/tools/browser#tab-cleanup-ownership) مراجعه کنید.
- `ssrfPolicy.dangerouslyAllowPrivateNetwork` در صورت تنظیم‌نشدن غیرفعال است، بنابراین پیمایش مرورگر به‌طور پیش‌فرض سخت‌گیرانه باقی می‌ماند.
- تنها زمانی `ssrfPolicy.dangerouslyAllowPrivateNetwork: true` را تنظیم کنید که عمداً به پیمایش مرورگر در شبکهٔ خصوصی اعتماد دارید.
- در حالت سخت‌گیرانه، نقاط پایانی پروفایل CDP راه‌دور (`profiles.*.cdpUrl`) هنگام بررسی‌های دسترسی‌پذیری/کشف، مشمول همان مسدودسازی شبکهٔ خصوصی هستند.
- `ssrfPolicy.allowPrivateNetwork` همچنان به‌عنوان نام مستعار قدیمی پشتیبانی می‌شود.
- در حالت سخت‌گیرانه، برای استثناهای صریح از `ssrfPolicy.hostnameAllowlist` و `ssrfPolicy.allowedHostnames` استفاده کنید.
- پروفایل‌های راه‌دور فقط قابلیت اتصال دارند (شروع/توقف/بازنشانی غیرفعال است).
- `profiles.*.cdpUrl`، مقادیر `http://`، `https://`، `ws://` و `wss://` را می‌پذیرد.
  هنگامی که می‌خواهید OpenClaw، `/json/version` را کشف کند از HTTP(S) استفاده کنید؛ هنگامی که
  ارائه‌دهنده یک URL مستقیم WebSocket برای DevTools در اختیارتان می‌گذارد، از WS(S) استفاده کنید.
- اگر یک سرویس CDP با مدیریت خارجی از طریق loopback قابل‌دسترسی است،
  `attachOnly: true` آن پروفایل را تنظیم کنید؛ در غیر این صورت OpenClaw پورت loopback را یک
  پروفایل مرورگر مدیریت‌شدهٔ محلی در نظر می‌گیرد و ممکن است خطاهای مالکیت پورت محلی را گزارش کند.
- پروفایل‌های `existing-session` به‌جای CDP از Chrome MCP استفاده می‌کنند و می‌توانند
  روی میزبان انتخاب‌شده یا از طریق یک Node مرورگر متصل شوند.
- پروفایل‌های `existing-session` می‌توانند `userDataDir` را برای هدف‌گیری یک
  پروفایل مرورگر خاص مبتنی بر Chromium، مانند Brave یا Edge، تنظیم کنند.
- پروفایل‌های `existing-session` می‌توانند زمانی که Chrome از قبل
  پشت یک نقطهٔ پایانی کشف HTTP(S) برای DevTools یا یک نقطهٔ پایانی مستقیم WS(S) در حال اجرا است، `cdpUrl` را تنظیم کنند. در آن
  حالت، OpenClaw به‌جای استفاده از اتصال خودکار، نقطهٔ پایانی را به Chrome MCP می‌دهد؛
  `userDataDir` برای آرگومان‌های راه‌اندازی Chrome MCP نادیده گرفته می‌شود.
- پروفایل‌های `existing-session` محدودیت‌های فعلی مسیر Chrome MCP را حفظ می‌کنند:
  کنش‌های مبتنی بر snapshot/ref به‌جای هدف‌گیری با انتخابگر CSS، هوک‌های بارگذاری
  تک‌فایلی، بدون بازنویسی مهلت زمانی گفت‌وگو، بدون `wait --load networkidle`، و بدون
  `responsebody`، خروجی PDF، رهگیری دانلود یا کنش‌های دسته‌ای.
- پروفایل‌های مدیریت‌شدهٔ محلی `openclaw`، مقادیر `cdpPort` و `cdpUrl` را به‌طور خودکار اختصاص می‌دهند؛
  `cdpUrl` را فقط برای پروفایل‌های CDP راه‌دور یا اتصال به نقطهٔ پایانی
  نشست موجود به‌صراحت تنظیم کنید.
- پروفایل‌های مدیریت‌شدهٔ محلی می‌توانند `executablePath` را برای بازنویسی
  `browser.executablePath` سراسری در همان پروفایل تنظیم کنند. از این قابلیت برای اجرای یک پروفایل در
  Chrome و پروفایلی دیگر در Brave استفاده کنید.
- ترتیب تشخیص خودکار: مرورگر پیش‌فرض در صورت مبتنی‌بودن بر Chromium → Chrome → Brave → Edge → Chromium → Chrome Canary.
- هر دو `browser.executablePath` و `browser.profiles.<name>.executablePath`،
  مقادیر `~` و `~/...` را برای دایرکتوری خانگی سیستم‌عامل شما پیش از راه‌اندازی Chromium می‌پذیرند.
  مقدار `userDataDir` مختص هر پروفایل در پروفایل‌های `existing-session` نیز با جایگزینی تیلدا گسترش می‌یابد.
- سرویس کنترل: فقط loopback (پورت از `gateway.port` مشتق می‌شود، مقدار پیش‌فرض `18791` است).
- `extraArgs` پرچم‌های راه‌اندازی اضافی را به شروع محلی Chromium می‌افزاید (برای مثال
  `--disable-gpu`، اندازه‌گذاری پنجره یا پرچم‌های اشکال‌زدایی).

---

## رابط کاربری

```json5
{
  ui: {
    seamColor: "#FF4500",
    assistant: {
      name: "OpenClaw",
      avatar: "CB", // ایموجی، متن کوتاه، URL تصویر یا URI داده
    },
    prefs: {
      theme: "claw", // claw | knot | dash | custom
      themeMode: "system", // light | dark | system
      locale: "en",
      chatShowThinking: true,
      chatShowToolCalls: true,
      chatPersistCommentary: true, // توضیحات را پس از اجراها در رابط کاربری کنترل نگه می‌دارد؛ آن‌ها را به کانال‌ها تحویل نمی‌دهد
      chatSendShortcut: "enter", // enter | modifier-enter
      chatFollowUpMode: "steer", // steer | queue؛ برای استفاده از حالت صف سرور حذفش کنید
      showAdvancedSettings: false, // همهٔ گروه‌های پیشرفته را در تنظیمات باز می‌کند
    },
  },
}
```

- `seamColor`: رنگ تأکیدی برای پوستهٔ رابط کاربری بومی برنامه (رنگ حباب حالت مکالمه و موارد دیگر).
- `assistant`: بازنویسی هویت رابط کاربری کنترل. در صورت نبود، از هویت عامل فعال استفاده می‌شود.
- `prefs`: ترجیحات اپراتور میان‌دستگاهی. این محل، خانهٔ مرجع است تا عامل‌ها بتوانند
  آن‌ها را از طریق دروازهٔ تأیید تغییر دهند و همهٔ کلاینت‌های رابط کاربری کنترل همگام
  بمانند؛ مرورگرها برای راه‌اندازی فوری، مقادیر را در فضای ذخیره‌سازی محلی بازتاب می‌دهند و هنگامی که
  نمی‌توانند پیکربندی را بنویسند (دامنهٔ مشاهده‌گر، آفلاین)، یک نسخهٔ محلیِ دستگاه نگه می‌دارند.
  مقدار پیش‌فرض `chatPersistCommentary`، `true` است. تنظیم آن روی `false`،
  توضیحات زنده را هنگام اجرا قابل‌مشاهده نگه می‌دارد، اما در پایان آن‌ها را حذف می‌کند و مانع ورود
  توضیحات جدید Codex به آینهٔ پایدار رونوشت می‌شود. تحویل به کانال‌های
  پیام‌رسان جداگانه و بدون تغییر باقی می‌ماند.
  مقدار پیش‌فرض `showAdvancedSettings`، `false` است؛ جست‌وجوی تنظیمات ممکن است موقتاً
  یک گروه پیشرفتهٔ منطبق را بدون تغییر این ترجیح باز کند.
  ترجیحات صرفاً نمایشی، مانند مقیاس متن، عرض گفت‌وگو و فعالیت زندهٔ
  نوار کناری، در مرورگر محلی باقی می‌مانند و در تنظیمات پیکربندی می‌شوند.
  کلاینت‌های متصل تغییرات سمت سرور را به‌صورت زنده اعمال می‌کنند: Gateway پس از هر
  نوشتن پایدار پیکربندی، یک رویداد فقط-هش `config.changed` پخش می‌کند و
  کلاینت‌ها snapshot خود را تازه می‌کنند (هنگامی که پیش‌نویس محلی تنظیمات دارای
  ویرایش‌های ذخیره‌نشده باشد، این کار انجام نمی‌شود). کلاینت‌های در حال اتصال مجدد هنگام اتصال تطبیق داده می‌شوند.

---

## Gateway

```json5
{
  gateway: {
    mode: "local", // local | remote
    port: 18789,
    bind: "loopback",
    auth: {
      mode: "token", // none | token | password | trusted-proxy
      token: "your-token",
      // password: "your-password", // یا OPENCLAW_GATEWAY_PASSWORD
      // trustedProxy: { userHeader: "x-forwarded-user" }, // برای mode=trusted-proxy؛ به /gateway/trusted-proxy-auth مراجعه کنید
      allowTailscale: true,
      rateLimit: {
        maxAttempts: 10,
        windowMs: 60000,
        lockoutMs: 300000,
        exemptLoopback: true,
      },
    },
    tailscale: {
      mode: "off", // off | serve | funnel
      resetOnExit: false,
    },
    controlUi: {
      enabled: true,
      basePath: "/openclaw",
      // root: "dist/control-ui",
      // toolTitles: false, // فعال‌سازی اختیاری عنوان‌های هدف تولیدشده با هوش مصنوعی برای فراخوانی ابزارها (توکن‌های مدل کمکی را مصرف می‌کند)
      // embedSandbox: "scripts", // strict | scripts | trusted
      // allowExternalEmbedUrls: false, // خطرناک: اجازه‌دادن به URLهای جاسازی خارجی و مطلق http(s)
      // allowedOrigins: ["https://control.example.com"], // برای رابط کاربری کنترل غیر-loopback الزامی است
      // dangerouslyAllowHostHeaderOriginFallback: false, // حالت خطرناک بازگشت به مبدأ مبتنی بر سرآیند Host
    },
    terminal: {
      enabled: false,
      // shell: "/bin/zsh",
    },
    remote: {
      url: "ws://127.0.0.1:18789",
      transport: "ssh", // ssh | direct
      token: "your-token",
      // password: "your-password",
    },
    trustedProxies: ["10.0.0.1"],
    // اختیاری. مقدار پیش‌فرض false است.
    allowRealIpFallback: false,
    nodes: {
      pairing: {
        // اختیاری. مقدار پیش‌فرض تنظیم‌نشده/غیرفعال است.
        autoApproveCidrs: ["192.168.1.0/24", "fd00:1234:5678::/64"],
        // تأیید خودکارِ اعتبارسنجی‌شده با SSH. پیش‌فرض: فعال (true).
        // برای غیرفعال‌کردن فقط تأیید SSH، مقدار false را تنظیم کنید؛ این کار بر
        // autoApproveCidrs بالا اثری ندارد. برای جفت‌سازی کاملاً دستی Node، مقدار false را تنظیم کنید و
        // autoApproveCidrs را نیز تنظیم نکنید. برای تنظیم دقیق، یک شیء ارسال کنید: { user, identity,
        // timeoutMs, cidrs }.
        sshVerify: true,
      },
      commands: {
        allow: ["canvas.navigate"],
        deny: ["system.run"],
      },
    },
    tools: {
      // موارد منع اضافی HTTP برای /tools/invoke
      deny: ["browser"],
      // حذف ابزارها از فهرست منع پیش‌فرض HTTP برای فراخوانندگان مالک/مدیر
      allow: ["gateway"],
    },
    push: {
      apns: {
        relay: {
          baseUrl: "https://relay.example.com",
          timeoutMs: 10000,
        },
      },
    },
  },
}
```

<Accordion title="جزئیات فیلدهای Gateway">

- `mode`: `local` (اجرای Gateway) یا `remote` (اتصال به Gateway راه‌دور). Gateway آغاز به کار نمی‌کند مگر اینکه `local`.
- `port`: یک درگاه چندگانه برای WS و HTTP. اولویت: `--port` > `OPENCLAW_GATEWAY_PORT` > `gateway.port` > `18789`.
- `bind`: `auto`، `loopback` (پیش‌فرض)، `lan` (`0.0.0.0`)، `tailnet` (IPv4 مربوط به Tailscale در صورت دسترس‌بودن، وگرنه loopback)، یا `custom` (یک نشانی IPv4). یک نشانی تفکیک‌شدهٔ `tailnet` و هر نشانی `custom` به‌جز `127.0.0.1` یا `0.0.0.0`، برای سرویس‌گیرندگان همان میزبان به `127.0.0.1` روی همان درگاه نیاز دارد؛ اگر هرکدام از شنونده‌ها نتواند متصل شود، راه‌اندازی ناموفق خواهد بود. دسترسی غیر-loopback همچنان به رابط انتخاب‌شده محدود است.
- **نام‌های مستعار قدیمی اتصال**: از مقادیر حالت اتصال در `gateway.bind` (`auto`، `loopback`، `lan`، `tailnet`، `custom`) استفاده کنید، نه نام‌های مستعار میزبان (`0.0.0.0`، `127.0.0.1`، `localhost`، `::`، `::1`).
- **نکتهٔ Docker**: اتصال پیش‌فرض `loopback` درون کانتینر روی `127.0.0.1` گوش می‌دهد. با شبکه‌سازی پل Docker (`-p 18789:18789`)، ترافیک روی `eth0` وارد می‌شود، بنابراین Gateway در دسترس نیست. از `--network host` استفاده کنید، یا `bind: "lan"` (یا `bind: "custom"` همراه با `customBindHost: "0.0.0.0"`) را تنظیم کنید تا روی همهٔ رابط‌ها گوش دهد.
- **احراز هویت**: به‌طور پیش‌فرض الزامی است. اتصال‌های غیر-loopback به احراز هویت Gateway نیاز دارند. در عمل، این به‌معنای یک توکن/گذرواژهٔ مشترک یا پراکسی معکوس آگاه از هویت با `gateway.auth.mode: "trusted-proxy"` است. راهنمای آغاز به کار به‌طور پیش‌فرض یک توکن تولید می‌کند.
- اگر هر دو `gateway.auth.token` و `gateway.auth.password` پیکربندی شده‌اند (از جمله SecretRefها)، `gateway.auth.mode` را صریحاً روی `token` یا `password` تنظیم کنید. هنگامی که هر دو پیکربندی شده باشند و حالت تنظیم نشده باشد، راه‌اندازی و جریان‌های نصب/ترمیم سرویس ناموفق خواهند بود.
- `gateway.auth.mode: "none"`: حالت صریح بدون احراز هویت. فقط برای پیکربندی‌های loopback محلی و مورداعتماد استفاده کنید؛ این گزینه عمداً در پیام‌های آغاز به کار ارائه نمی‌شود.
- `gateway.auth.mode: "trusted-proxy"`: احراز هویت مرورگر/کاربر را به یک پراکسی معکوس آگاه از هویت واگذار کنید و به سرآیندهای هویت از `gateway.trustedProxies` اعتماد کنید (نگاه کنید به [احراز هویت پراکسی مورداعتماد](/fa/gateway/trusted-proxy-auth)). این حالت به‌طور پیش‌فرض انتظار یک منبع پراکسی **غیر-loopback** را دارد؛ پراکسی‌های معکوس loopback روی همان میزبان به `gateway.auth.trustedProxy.allowLoopback = true` صریح نیاز دارند. فراخوان‌های داخلی همان میزبان می‌توانند از `gateway.auth.password` به‌عنوان جایگزین مستقیم محلی استفاده کنند؛ `gateway.auth.token` همچنان با حالت پراکسی مورداعتماد ناسازگار است.
- `gateway.auth.allowTailscale`: هنگامی که `true` است، سرآیندهای هویت Tailscale Serve می‌توانند احراز هویت رابط کنترل/WebSocket را برآورده کنند (تأییدشده از طریق `tailscale whois`). نقاط پایانی API مربوط به HTTP از آن احراز هویت سرآیند Tailscale استفاده **نمی‌کنند**؛ در عوض از حالت عادی احراز هویت HTTP مربوط به Gateway پیروی می‌کنند. این جریان بدون توکن فرض می‌کند میزبان Gateway مورداعتماد است. هنگامی که `tailscale.mode = "serve"`، مقدار پیش‌فرض `true` است.
- `gateway.auth.rateLimit`: محدودکنندهٔ اختیاری احراز هویت ناموفق. به‌ازای هر IP سرویس‌گیرنده و هر دامنهٔ احراز هویت اعمال می‌شود (راز مشترک و توکن دستگاه جداگانه ردیابی می‌شوند). تلاش‌های مسدودشده `429` + `Retry-After` را برمی‌گردانند.
  - در مسیر ناهمگام رابط کنترل Tailscale Serve، تلاش‌های ناموفق برای `{scope, clientIp}` یکسان پیش از ثبت شکست به‌صورت سریالی انجام می‌شوند. بنابراین تلاش‌های بد هم‌زمان از یک سرویس‌گیرنده می‌توانند در درخواست دوم محدودکننده را فعال کنند، به‌جای آنکه هر دو به‌صورت رقابتی صرفاً به‌عنوان عدم تطابق عبور کنند.
  - `gateway.auth.rateLimit.exemptLoopback` به‌طور پیش‌فرض `true` است؛ هنگامی که عمداً می‌خواهید ترافیک localhost نیز محدود شود (برای پیکربندی‌های آزمایشی یا استقرارهای سخت‌گیرانهٔ پراکسی)، `false` را تنظیم کنید.
- تلاش‌های احراز هویت WS با مبدأ مرورگر همیشه با غیرفعال‌بودن معافیت loopback محدود می‌شوند (دفاع چندلایه در برابر حملهٔ جست‌وجوی فراگیر به localhost از طریق مرورگر).
- در loopback، آن قفل‌شدن‌های با مبدأ مرورگر به‌ازای هر مقدار نرمال‌شدهٔ `Origin`
  جدا هستند، بنابراین شکست‌های مکرر از یک مبدأ localhost به‌طور خودکار
  مبدأ دیگری را قفل نمی‌کنند.
- `tailscale.mode`: `serve` (فقط tailnet، اتصال loopback) یا `funnel` (عمومی، نیازمند احراز هویت).
- `tailscale.serviceName`: نام اختیاری سرویس Tailscale برای حالت Serve، مانند
  `svc:openclaw`. هنگامی که تنظیم شود، OpenClaw آن را به `tailscale serve
--service` می‌دهد تا رابط کنترل به‌جای
  نام میزبان دستگاه از طریق یک سرویس نام‌گذاری‌شده در دسترس قرار گیرد. مقدار باید از قالب نام سرویس
  `svc:<dns-label>` مربوط به Tailscale استفاده کند؛ راه‌اندازی URL مشتق‌شدهٔ سرویس را گزارش می‌کند.
- `tailscale.preserveFunnel`: هنگامی که `true` و `tailscale.mode = "serve"`، OpenClaw
  پیش از اعمال دوبارهٔ Serve هنگام راه‌اندازی، `tailscale funnel status` را بررسی می‌کند و اگر
  یک مسیر Funnel با پیکربندی خارجی از قبل درگاه Gateway را پوشش دهد، از آن صرف‌نظر می‌کند.
  پیش‌فرض `false`.
- `controlUi.allowedOrigins`: فهرست مجاز صریح مبدأهای مرورگر برای اتصال‌های WebSocket مربوط به Gateway. برای مبدأهای عمومی و غیر-loopback مرورگر الزامی است. بارگذاری‌های خصوصی و هم‌مبدأ رابط کاربری LAN/Tailnet از میزبان‌های loopback، RFC1918/link-local، `.local`، `.ts.net` یا Tailscale CGNAT بدون فعال‌کردن جایگزین سرآیند Host پذیرفته می‌شوند.
- `controlUi.toolTitles`: استفاده از عنوان‌های هدف تولیدشده با هوش مصنوعی برای فراخوانی ابزارها در گفت‌وگوی رابط کنترل را فعال کنید. پیش‌فرض: `false` (نمایش ابزار کاملاً قطعی و بدون فراخوانی پس‌زمینهٔ مدل باقی می‌ماند). وقتی فعال باشد، روش `chat.toolTitles` فراخوانی‌های پیچیده را از طریق مسیریابی استاندارد مدل کاربردی برچسب‌گذاری می‌کند — `utilityModel` عامل (تصمیمی از سوی اپراتور که ممکن است آرگومان‌های محدود ابزار را مانند هر وظیفهٔ کاربردی به ارائه‌دهندهٔ انتخاب‌شده ارسال کند)، یا مدل کوچک پیش‌فرض اعلام‌شده توسط ارائه‌دهندهٔ نشست (OpenAI ← `gpt-5.6-luna`، Anthropic ← `claude-haiku-4-5`) — و نتایج را در پایگاه دادهٔ وضعیت هر عامل ذخیره می‌کند تا مشاهده‌های تکراری هرگز دوباره هزینه ایجاد نکنند. `utilityModel: \"\"` مانند هر وظیفهٔ کاربردی دیگر عنوان‌ها را غیرفعال می‌کند؛ عنوان‌ها هرگز به مدل اصلی بازنمی‌گردند.
- `controlUi.dangerouslyAllowHostHeaderOriginFallback`: حالتی خطرناک که جایگزین مبدأ مبتنی بر سرآیند Host را برای استقرارهایی فعال می‌کند که عمداً به سیاست مبدأ سرآیند Host متکی هستند.
- `terminal.enabled`: استفاده از پایانهٔ اپراتور با دامنهٔ مدیر را فعال کنید. پیش‌فرض: `false`. پایانه یک PTY میزبان را در فضای کاری عامل انتخاب‌شده آغاز می‌کند، محیط فرایند Gateway را به ارث می‌برد و برای عامل‌های دارای `sandbox.mode: "all"` رد می‌شود. آن را فقط برای استقرارهای اپراتوری مورداعتماد فعال کنید؛ تغییر آن Gateway را بازراه‌اندازی می‌کند و سیاست امنیت محتوای رابط کنترل را به‌روزرسانی می‌کند.
- `terminal.shell`: فایل اجرایی اختیاری پوسته. هنگامی که تنظیم نشده باشد، OpenClaw در Unix از `$SHELL` و در Windows از `%ComSpec%` استفاده می‌کند.
- `terminal.detachedSessionTimeoutSeconds`: مدت‌زمانی که نشست پایانه پس از قطع اتصالش (بازبارگذاری صفحه، خواب لپ‌تاپ) زنده می‌ماند و از طریق `terminal.attach` با بازپخش خروجی اخیرش قابل اتصال مجدد است. پیش‌فرض: `300`. برای پایان‌دادن به نشست‌ها در لحظهٔ قطع اتصال، `0` را تنظیم کنید. نشست‌های جداشده همچنان فرمان‌های خود را اجرا می‌کنند، بنابراین این مدت را در میزبان‌های اشتراکی یا در معرض دسترسی کاهش دهید.
- `remote.transport`: `ssh` (پیش‌فرض) یا `direct` (ws/wss). برای `direct`، در میزبان‌های عمومی `remote.url` باید `wss://` باشد؛ `ws://` متن ساده فقط برای میزبان‌های loopback، LAN، link-local، `.local`، `.ts.net` و Tailscale CGNAT پذیرفته می‌شود.
- `remote.remotePort`: درگاه Gateway روی میزبان SSH راه‌دور. مقدار پیش‌فرض `18789` است؛ هنگامی که درگاه تونل محلی با درگاه Gateway راه‌دور متفاوت است، از این گزینه استفاده کنید.
- `remote.tlsFingerprint`: اثرانگشت موردانتظار گواهی SHA-256 برای یک Gateway راه‌دور `wss://`. برنامهٔ macOS آن را هم برای اتصال‌های اپراتور/کنترل و هم برای اتصال‌های Node همراه اعمال می‌کند. بدون مقدار صریح، macOS فقط پس از موفقیت اعتماد عادی سیستم، پین نخستین استفاده را ثبت می‌کند.
- `remote.sshHostKeyPolicy`: سیاست کلید میزبان تونل SSH در macOS. `strict` مقدار پیش‌فرض است و به کلیدی نیاز دارد که از قبل مورداعتماد باشد. `openssh` یک رضایت صریح برای پیکربندی مؤثر OpenSSH در نام‌های مستعار مدیریت‌شده است؛ پیش از استفاده، تنظیمات منطبق SSH کاربر و سیستم را بررسی کنید. برنامهٔ macOS و `configure-remote` هنگام تغییر مقصدها این سیاست را به `strict` بازنشانی می‌کنند، مگر اینکه دوباره صریحاً فعال شود.
- `gateway.remote.token` / `.password` فیلدهای اعتبارنامهٔ سرویس‌گیرندهٔ راه‌دور هستند. این فیلدها به‌تنهایی احراز هویت Gateway را پیکربندی نمی‌کنند.
- `gateway.push.apns.relay.baseUrl`: URL پایهٔ HTTPS برای رلهٔ خارجی APNs که پس از انتشار ثبت‌ها در Gateway توسط ساخت‌های iOS متکی بر رله استفاده می‌شود. ساخت‌های عمومی App Store از رلهٔ میزبانی‌شدهٔ OpenClaw استفاده می‌کنند. URLهای سفارشی رله باید با یک مسیر ساخت/استقرار عمداً جداگانهٔ iOS منطبق باشند که URL رلهٔ آن به همان رله اشاره دارد.
- `gateway.push.apns.relay.timeoutMs`: مهلت ارسال از Gateway به رله بر حسب میلی‌ثانیه. مقدار پیش‌فرض `10000` است.
- ثبت‌های متکی بر رله به یک هویت مشخص Gateway واگذار می‌شوند. برنامهٔ جفت‌شدهٔ iOS، `gateway.identity.get` را دریافت می‌کند، آن هویت را در ثبت رله می‌گنجاند و یک مجوز ارسال با دامنهٔ ثبت را به Gateway می‌فرستد. Gateway دیگری نمی‌تواند از آن ثبت ذخیره‌شده دوباره استفاده کند.
- `OPENCLAW_APNS_RELAY_BASE_URL` / `OPENCLAW_APNS_RELAY_TIMEOUT_MS`: نادیده‌گیری‌های موقت محیطی برای پیکربندی رلهٔ بالا.
- `OPENCLAW_APNS_RELAY_ALLOW_HTTP=true`: راه فرار صرفاً توسعه‌ای برای URLهای رلهٔ HTTP در loopback. URLهای رلهٔ محیط تولید باید روی HTTPS باقی بمانند.
- `OPENCLAW_HANDSHAKE_TIMEOUT_MS`: نادیده‌گیری اختیاری محیط برای مهلت داخلی دست‌دهی WebSocket پیش از احراز هویت Gateway.
- `channels.<provider>.healthMonitor.enabled`: انصراف به‌ازای هر کانال از بازراه‌اندازی‌های پایشگر سلامت، درحالی‌که پایشگر سراسری فعال می‌ماند.
- `channels.<provider>.accounts.<accountId>.healthMonitor.enabled`: نادیده‌گیری به‌ازای هر حساب برای کانال‌های چندحسابی. هنگامی که تنظیم شود، بر نادیده‌گیری سطح کانال اولویت دارد.
- مسیرهای فراخوانی Gateway محلی فقط هنگامی می‌توانند از `gateway.remote.*` به‌عنوان جایگزین استفاده کنند که `gateway.auth.*` تنظیم نشده باشد.
- اگر `gateway.auth.token` / `gateway.auth.password` صریحاً از طریق SecretRef پیکربندی شده و تفکیک‌نشده باشد، تفکیک به‌صورت بسته و امن ناموفق می‌شود (بدون پنهان‌سازی با جایگزین راه‌دور).
- `trustedProxies`: IPهای پراکسی معکوس که TLS را خاتمه می‌دهند یا سرآیندهای ارسال‌شدهٔ سرویس‌گیرنده را تزریق می‌کنند. فقط پراکسی‌هایی را فهرست کنید که تحت کنترل شما هستند. ورودی‌های loopback همچنان برای پیکربندی‌های تشخیص محلی/پراکسی همان میزبان معتبرند (برای نمونه Tailscale Serve یا یک پراکسی معکوس محلی)، اما درخواست‌های loopback را واجد شرایط `gateway.auth.mode: "trusted-proxy"` **نمی‌کنند**.
- `allowRealIpFallback`: هنگامی که `true`، Gateway در صورت نبود `X-Forwarded-For`، `X-Real-IP` را می‌پذیرد. برای رفتار بسته و امن، مقدار پیش‌فرض `false` است.
- `gateway.nodes.pairing.autoApproveCidrs`: فهرست مجاز اختیاری CIDR/IP برای تأیید خودکار نخستین جفت‌سازی دستگاه Node بدون دامنه‌های درخواستی. در صورت تنظیم‌نشدن غیرفعال است. این گزینه جفت‌سازی اپراتور/مرورگر/رابط کنترل/WebChat را خودکار تأیید نمی‌کند و ارتقای نقش، دامنه، فراداده یا کلید عمومی را نیز خودکار تأیید نمی‌کند.
- `gateway.nodes.pairing.sshVerify`: تأیید خودکار مبتنی بر اعتبارسنجی SSH برای نخستین جفت‌سازی دستگاه Node (پیش‌فرض: فعال). Gateway از طریق SSH به میزبان جفت‌سازی متصل می‌شود (BatchMode، کلیدهای میزبان سخت‌گیرانه) و فقط در صورت تطابق دقیق کلید دستگاه `openclaw node identity` تأیید می‌کند. حداقل شرایط احراز همان `autoApproveCidrs` است؛ کاوش‌ها به نشانی‌های مبدأ خصوصی/CGNAT محدودند، مگر اینکه `cidrs` آن‌ها را نادیده بگیرد. برای غیرفعال‌کردن `false` را تنظیم کنید، یا برای تنظیم دقیق از `{ user, identity, timeoutMs, cidrs }` استفاده کنید. نگاه کنید به [جفت‌سازی Node](/fa/gateway/pairing#ssh-verified-device-auto-approval-default).
- `gateway.nodes.commands.allow` / `gateway.nodes.commands.deny`: شکل‌دهی سراسری مجاز/غیرمجاز برای فرمان‌های اعلام‌شده Node پس از ارزیابی جفت‌سازی و فهرست مجاز پلتفرم. برای فعال‌سازی فرمان‌های خطرناک Node مانند `camera.snap`، `camera.clip`، `screen.record`، `health.summary`، `sms.search` و `sms.send` از `commands.allow` استفاده کنید؛ `commands.deny` یک فرمان را حذف می‌کند، حتی اگر در حالت عادی پیش‌فرض پلتفرم یا مجوز صریح آن را شامل شود. مجوز Health در iOS، مجوز SMS در Android و مجوزدهی فرمان Gateway مستقل از یکدیگرند. پس از تغییر فهرست فرمان‌های اعلام‌شده یک Node، جفت‌سازی آن دستگاه را رد و دوباره تأیید کنید تا Gateway تصویر لحظه‌ای به‌روزشده فرمان‌ها را ذخیره کند.
- `gateway.tools.deny`: نام ابزارهای اضافی مسدودشده برای HTTP `POST /tools/invoke` (فهرست پیش‌فرض موارد غیرمجاز را گسترش می‌دهد).
- `gateway.tools.allow`: حذف نام ابزارها از فهرست پیش‌فرض موارد غیرمجاز HTTP برای
  فراخوانندگان مالک/مدیر. این کار فراخوانندگان هویت‌دار `operator.write` را به دسترسی
  مالک/مدیر ارتقا نمی‌دهد؛ `cron`، `gateway` و `nodes` حتی در صورت قرارگرفتن در فهرست مجاز نیز
  برای فراخوانندگان غیرمالک در دسترس نمی‌مانند.

</Accordion>

### نقاط پایانی سازگار با OpenAI

- RPC مدیریتی HTTP: مانند Plugin‏ `admin-http-rpc` به‌طور پیش‌فرض خاموش است. برای ثبت `POST /api/v1/admin/rpc`، Plugin را فعال کنید. [RPC مدیریتی HTTP](/fa/plugins/admin-http-rpc) را ببینید.
- Chat Completions: به‌طور پیش‌فرض غیرفعال است. با `gateway.http.endpoints.chatCompletions.enabled: true` فعال کنید.
- Responses API: `gateway.http.endpoints.responses.enabled`.
- سخت‌سازی ورودی URL در Responses:
  - `gateway.http.endpoints.responses.maxUrlParts`
  - `gateway.http.endpoints.responses.files.urlAllowlist`
  - `gateway.http.endpoints.responses.images.urlAllowlist`
    فهرست‌های مجاز خالی، تنظیم‌نشده در نظر گرفته می‌شوند؛ برای غیرفعال‌کردن واکشی URL از `gateway.http.endpoints.responses.files.allowUrl=false`
    و/یا `gateway.http.endpoints.responses.images.allowUrl=false` استفاده کنید.
- هدر اختیاری سخت‌سازی پاسخ:
  - `gateway.http.securityHeaders.strictTransportSecurity` (فقط برای مبدأهای HTTPS تحت کنترل خود تنظیم کنید؛ [احراز هویت پراکسی مورد اعتماد](/fa/gateway/trusted-proxy-auth#tls-termination-and-hsts) را ببینید)

### جداسازی چند نمونه

چند Gateway را با پورت‌ها و دایرکتوری‌های وضعیت یکتا روی یک میزبان اجرا کنید:

```bash
OPENCLAW_CONFIG_PATH=~/.openclaw/a.json \
OPENCLAW_STATE_DIR=~/.openclaw-a \
openclaw gateway --port 19001
```

پرچم‌های تسهیل‌کننده: `--dev` (از `~/.openclaw-dev` + پورت `19001` استفاده می‌کند)، `--profile <name>` (از `~/.openclaw-<name>` استفاده می‌کند).

[چند Gateway](/fa/gateway/multiple-gateways) را ببینید.

### `gateway.tls`

```json5
{
  gateway: {
    tls: {
      enabled: false,
      autoGenerate: false,
      certPath: "/etc/openclaw/tls/server.crt",
      keyPath: "/etc/openclaw/tls/server.key",
      caPath: "/etc/openclaw/tls/ca-bundle.crt",
    },
  },
}
```

- `enabled`: خاتمه TLS را در شنونده Gateway‏ (HTTPS/WSS) فعال می‌کند (پیش‌فرض: `false`).
- `autoGenerate`: وقتی فایل‌های صریح پیکربندی نشده‌اند، یک جفت گواهی/کلید خودامضای محلی تولید می‌کند؛ فقط برای استفاده محلی/توسعه.
- `certPath`: مسیر سیستم فایل به فایل گواهی TLS.
- `keyPath`: مسیر سیستم فایل به فایل کلید خصوصی TLS؛ دسترسی آن را محدود نگه دارید.
- `caPath`: مسیر اختیاری بسته CA برای تأیید کلاینت یا زنجیره‌های اعتماد سفارشی.

### `gateway.reload`

```json5
{
  gateway: {
    reload: {
      mode: "hybrid", // off | restart | hot | hybrid
      debounceMs: 500,
      deferralTimeoutMs: 300000,
    },
  },
}
```

- `mode`: نحوه اعمال ویرایش‌های پیکربندی در زمان اجرا را کنترل می‌کند.
  - `"off"`: ویرایش‌های زنده را نادیده می‌گیرد؛ تغییرات به راه‌اندازی مجدد صریح نیاز دارند.
  - `"restart"`: هنگام تغییر پیکربندی، همیشه فرایند Gateway را راه‌اندازی مجدد می‌کند.
  - `"hot"`: تغییرات را بدون راه‌اندازی مجدد درون فرایند اعمال می‌کند.
  - `"hybrid"` (پیش‌فرض): ابتدا بارگذاری مجدد داغ را امتحان می‌کند؛ در صورت نیاز به راه‌اندازی مجدد برمی‌گردد.
- `debounceMs`: بازه رفع نوسان بر حسب میلی‌ثانیه پیش از اعمال تغییرات پیکربندی (عدد صحیح نامنفی؛ پیش‌فرض: `300`).
- `deferralTimeoutMs`: حداکثر زمان اختیاری بر حسب میلی‌ثانیه برای انتظار عملیات در حال انجام، پیش از اجبار به راه‌اندازی مجدد یا بارگذاری مجدد داغ کانال. برای استفاده از انتظار محدود پیش‌فرض (`300000`) آن را حذف کنید؛ برای انتظار نامحدود و ثبت دوره‌ای هشدارهای همچنان در انتظار، `0` را تنظیم کنید.

---

## محیط‌های کارگر ابری

کارگرهای ابری اختیاری هستند. اگر `cloudWorkers` وجود نداشته باشد یا `profiles` خالی باشد، OpenClaw ایجاد هیچ کارگر جدیدی را نمی‌پذیرد. رکوردهای ماندگاری که پیش‌تر ایجاد شده‌اند همچنان تطبیق داده می‌شوند و قابل مشاهده می‌مانند؛ تصویر موجود Gateway/Node بدون تغییر است.

هر ارائه‌دهنده کارگر باید یک `hostKey` مربوط به SSH را از خروجی تأمین مورد اعتماد، دقیقاً به‌شکل `algorithm base64` و بدون نام میزبان یا توضیح بازگرداند. راه‌انداز اولیه آن کلید را در یک فایل مجزای `known_hosts` می‌نویسد، از `StrictHostKeyChecking=yes` استفاده می‌کند و اگر ارائه‌دهنده آن را ارائه نکند، پیش از بازکردن اتصال شکست می‌خورد. هیچ سازوکار جایگزینی برای اعتماد در نخستین استفاده وجود ندارد.

راه‌اندازی تونل بنا به درخواست انجام می‌شود و بخشی از تأمین نیست. هنگام شروع، Gateway یک سوکت Unix محلی کارگر را به نقطه پایانی WebSocket روی loopback خود به‌صورت معکوس فوروارد می‌کند. سوکت در یک دایرکتوری راه دور با تخصیص تصادفی و دسترسی صرفاً مالک قرار دارد؛ برخلاف پورت TCP روی loopback، حساب‌های دیگر روی یک کارگر چندکاربره نمی‌توانند به آن دسترسی داشته باشند و با پورت محیط دیگری تداخل نمی‌کند. پیام‌های زنده‌نگه‌داشتن SSH و عقب‌نشینی محدود اتصال مجدد، فقط تا زمانی اجرا می‌شوند که مالک تونل همچنان جاری باشد. توقف تونل، پیش از بستن فرایند SSH، اتصال‌های مجدد را مسدود می‌کند.

ترافیک کنترلی و انتقال فضای کاری از اتصال‌های SSH جداگانه استفاده می‌کنند. هر دو از همان هویت حل‌شده و فایل مجزای سنجاق‌شده `known_hosts` استفاده می‌کنند، اما انتقال فضای کاری، چندگانه‌سازی اتصال SSH را با تونل بلندمدت به اشتراک نمی‌گذارد؛ بنابراین rsync نمی‌تواند ترافیک کنترلی را مسدود کند.

### نمایه Crabbox

ارائه‌دهنده همراه `crabbox` از طریق CLI محلی Crabbox یک اجاره دارای قابلیت SSH تأمین می‌کند. `settings.provider` داخلی، بک‌اند Crabbox را انتخاب می‌کند؛ این مورد از شناسه ارائه‌دهنده بیرونی OpenClaw جدا است.

```json5
{
  cloudWorkers: {
    profiles: {
      production: {
        provider: "crabbox",
        install: "bundle", // Default; use "npm" only for a released gateway version.
        settings: {
          provider: "aws",
          class: "standard",
          ttl: "24h",
          idleTimeout: "60m",
          // Optional absolute path. Default: sibling ../crabbox/bin/crabbox, then PATH.
          binary: "/usr/local/bin/crabbox",
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `settings.provider` (الزامی): بک‌اند Crabbox که از طریق `--provider` ارسال می‌شود. از بک‌اندی استفاده کنید که خروجی بازرسی آن شامل یک نقطه پایانی SSH باشد؛ `aws` بک‌اند مستقیم AWS را انتخاب می‌کند.
- `settings.class` (الزامی): کلاس ماشین Crabbox که به `--class` ارسال می‌شود.
- `settings.ttl` و `settings.idleTimeout` (الزامی): رشته‌های مدت‌زمان مثبت Go که به `--ttl` و `--idle-timeout` ارسال می‌شوند. این سازوکارهای ایمنی سمت ارائه‌دهنده با خط‌مشی ذخیره‌شده `lifetime` در OpenClaw که در ادامه آمده، متفاوت‌اند.
- `settings.binary`: مسیر مطلق اختیاری فایل اجرایی Crabbox. بدون آن، OpenClaw ابتدا checkout هم‌سطح Crabbox و سپس ورودی‌های اجرایی در `PATH` را بررسی می‌کند و در نهایت `crabbox` را فراخوانی می‌کند تا نبود CLI همچنان به‌صورت خطای قابل مشاهده ارائه‌دهنده باقی بماند.

تنظیمات ناشناخته رد می‌شوند. اعتبارنامه‌های Crabbox و پیکربندی حساب مختص بک‌اند همچنان در مالکیت Crabbox باقی می‌مانند؛ آن‌ها را در `settings` قرار ندهید. OpenClaw فقط CLI محلی را فراخوانی می‌کند و این Plugin هیچ فراخوانی شبکه‌ای به ارائه‌دهنده انجام نمی‌دهد. تأمین همیشه `--keep=true` را ارسال می‌کند؛ OpenClaw مالک چرخه حیات خارجی است و اجاره را با `crabbox stop` نابود می‌کند.

<Note>
  OpenClaw مسیر محلی اجاره `sshKey` متعلق به Crabbox را از طریق حل‌کننده راز متعلق به ارائه‌دهنده حل می‌کند و `sshHostKey` معتبر بازگردانده‌شده توسط `crabbox inspect --json` را سنجاق می‌کند. پذیرش AWS همچنین به `providerMetadata.instanceProfileAttached` نیاز دارد. برای این قرارداد بازرسی بسته، Crabbox 0.38.1 یا جدیدتر را نصب کنید.
</Note>

### نمایه توسعه SSH ایستا

```json5
{
  cloudWorkers: {
    profiles: {
      development: {
        provider: "static-ssh",
        settings: {
          host: "worker.example.test",
          port: 22,
          user: "openclaw",
          hostKey: "ssh-ed25519 <base64-public-host-key>",
          keyRef: {
            source: "env",
            provider: "default",
            id: "OPENCLAW_WORKER_SSH_KEY",
          },
        },
        lifetime: {
          idleTimeoutMinutes: 60,
          maxLifetimeMinutes: 1440,
        },
      },
    },
  },
}
```

- `profiles`: نمایه‌های نام‌گذاری‌شده کارگر با شناسه‌های غیرخالی و بدون فاصله سفید ابتدا و انتها. هر نمایه، ارائه‌دهنده‌ای را انتخاب می‌کند که توسط یک Plugin ثبت شده است.
- `provider`: شناسه غیرخالی ارائه‌دهنده کارگر. نمونه‌ها از ارائه‌دهنده همراه `crabbox` و ارائه‌دهنده QA Lab‏ `static-ssh` استفاده می‌کنند.
- `install`: روش نصب کارگر. `"bundle"` (پیش‌فرض) یک بسته با هش محتوا از بیلد نصب‌شده Gateway را منتقل می‌کند و از نسخه‌های منتشرشده، توسعه و منتشرنشده پشتیبانی می‌کند. `"npm"` یک بهینه‌سازی اختیاری برای انتشار بسته‌بندی‌شده و بدون تغییر است؛ `openclaw@<exact gateway version>` را از رجیستری عمومی npm نصب می‌کند و هرگز `latest` را نصب نمی‌کند.
- Pluginهای ارائه‌دهنده همراه هنگام پیکربندی به‌طور خودکار انتخاب می‌شوند، اما غیرفعال‌سازی‌های صریح و `plugins.allow` همچنان اعمال می‌شوند. وقتی فهرست مجاز پیکربندی شده است، شناسه ارائه‌دهنده را (برای نمونه، `crabbox`) بگنجانید. Pluginهای ارائه‌دهنده خارجی نیز باید نصب و به‌صراحت فعال شوند.
- `settings`: JSON محدود متعلق به ارائه‌دهنده. Plugin انتخاب‌شده کلیدهای آن را تعریف و اعتبارسنجی می‌کند؛ برای مقادیر حاوی راز از [اشیای SecretRef](/fa/gateway/secrets) استفاده کنید. ارائه‌دهنده SSH ایستا به `host`، `user`، `hostKey` و `keyRef` نیاز دارد؛ مقدار پیش‌فرض `port` برابر `22` است. `hostKey` باید یک خط کلید عمومی میزبان OpenSSH‏ (`algorithm base64`) باشد که از میزبان شناخته‌شده یا کانال مورد اعتماد دیگری دریافت شده و پیشوند گزینه نداشته باشد.
- `lifetime.idleTimeoutMinutes`: تعداد صحیح مثبت دقیقه که برای خط‌مشی بازیابی بعدی هنگام بی‌کاری ذخیره می‌شود.
- `lifetime.maxLifetimeMinutes`: تعداد صحیح مثبت دقیقه که برای خط‌مشی بعدی چرخه حیات ذخیره می‌شود.

یک محیط اجرای Node پشتیبانی‌شده (22.22.3+، 24.15+ یا 25.9+) با SQLite ایمن برای بازنشانی WAL باید از قبل روی کارگر نصب شده باشد. روش اختیاری `"npm"` همچنین به `npm` و دسترسی خروجی HTTPS به رجیستری عمومی npm نیاز دارد. راه‌اندازی زنجیره ابزار شبکه‌ای، خط‌مشی ارائه‌دهنده است؛ راه‌انداز اولیه به‌جای نصب زنجیره‌های ابزار، خطایی قابل اقدام گزارش می‌کند.

این زیرساخت، بیلد Gateway را نصب و تأیید می‌کند و چرخه حیات شروع/توقف تونل را فراهم می‌سازد، اما CLI عمومی OpenClaw را اجرا نمی‌کند. نقطه ورود و حلقه خودبسنده کارگر در مرحله بعدی کارگر ابری ارائه می‌شوند.

هر رکورد ماندگار محیط، تنظیمات اعتبارسنجی‌شده ارائه‌دهنده، روش نصب حل‌شده و خط‌مشی طول عمر خود را در یک snapshot نمایه هنگام ایجاد نگه می‌دارد. تغییر یا حذف یک نمایه نام‌گذاری‌شده بر ایجادهای جدید اثر می‌گذارد؛ رکوردهای موجود، به‌شرط در دسترس‌بودن Plugin مالک، تطبیق چرخه حیات را با همان snapshot ادامه می‌دهند.

مقادیر طول عمر در نخستین انتشار کارگر ابری فقط داده هستند؛ اعمال خودکار آن‌ها با کارهای بعدی چرخه حیات ارائه می‌شود. تغییرات نمایه به راه‌اندازی مجدد Gateway نیاز دارند.

<Warning>
  ارائه‌دهنده `static-ssh` یک زیرساخت توسعه QA Lab مبتنی بر درخت منبع است و در توزیع‌های بسته‌بندی‌شده وجود ندارد. کارگری که روی میزبان مشترک آن اجرا می‌شود می‌تواند داده‌های نامرتبط میزبان را بخواند؛ بنابراین از این ارائه‌دهنده به‌عنوان مرز جداسازی تولید استفاده نکنید.
  اپراتور آن باید `hostKey` مورد انتظار را ارائه کند؛ OpenClaw در نخستین اتصال کلیدی را یاد نمی‌گیرد یا نمی‌پذیرد.
  نابودکردن اجاره آن فقط رکورد منطقی OpenClaw را آزاد می‌کند؛ میزبان را متوقف یا پاک‌سازی نمی‌کند.
</Warning>

---

## هوک‌ها

```json5
{
  hooks: {
    enabled: true,
    token: "shared-secret",
    path: "/hooks",
    defaultSessionKey: "hook:ingress",
    allowRequestSessionKey: true,
    allowedSessionKeyPrefixes: ["hook:", "hook:gmail:"],
    allowedAgentIds: ["hooks", "main"],
    presets: ["gmail"],
    transformsDir: "~/.openclaw/hooks/transforms",
    mappings: [
      {
        match: { path: "gmail" },
        action: "agent",
        agentId: "hooks",
        wakeMode: "now",
        name: "Gmail",
        sessionKey: "hook:gmail:{{messages[0].id}}",
        messageTemplate: "From: {{messages[0].from}}\nSubject: {{messages[0].subject}}\n{{messages[0].snippet}}",
        deliver: true,
        channel: "last",
        model: "openai/gpt-5.6-sol",
      },
    ],
  },
}
```

احراز هویت: `Authorization: Bearer <token>` یا `x-openclaw-token: <token>`.
توکن‌های هوک در رشته پرس‌وجو رد می‌شوند.

نکات اعتبارسنجی و ایمنی:

- `hooks.enabled=true` باید یک `hooks.token` غیرخالی داشته باشد.
- `hooks.token` باید از احراز هویت فعال Gateway با راز مشترک (`gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` یا `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD`) متمایز باشد؛ هنگام شناسایی استفادهٔ مجدد، در زمان راه‌اندازی یک هشدار امنیتی غیرکشنده ثبت می‌شود.
- `openclaw security audit` استفادهٔ مجدد از احراز هویت هوک/Gateway را، از جمله احراز هویت Gateway با گذرواژه که فقط هنگام ممیزی ارائه شده است (`--auth password --password <password>`)، به‌عنوان یک یافتهٔ بحرانی علامت‌گذاری می‌کند. برای تعویض یک `hooks.token` ذخیره‌شده که مجدداً استفاده شده است، `openclaw doctor --fix` را اجرا کنید، سپس فرستنده‌های خارجی هوک را به‌روزرسانی کنید تا از توکن جدید هوک استفاده کنند.
- `hooks.path` نمی‌تواند `/` باشد؛ از یک زیرمسیر اختصاصی مانند `/hooks` استفاده کنید.
- اگر `hooks.allowRequestSessionKey=true`، `hooks.allowedSessionKeyPrefixes` را محدود کنید (برای مثال `["hook:"]`).
- اگر یک نگاشت یا پیش‌تنظیم از `sessionKey` قالب‌دار استفاده می‌کند، `hooks.allowedSessionKeyPrefixes` و `hooks.allowRequestSessionKey=true` را تنظیم کنید. کلیدهای نگاشت ایستا به این پذیرش صریح نیاز ندارند.

**نقاط پایانی:**

- `POST /hooks/wake` → `{ text, mode?: "now"|"next-heartbeat" }`
- `POST /hooks/agent` → `{ message, name?, agentId?, sessionKey?, wakeMode?, deliver?, channel?, to?, model?, thinking?, timeoutSeconds? }`
  - `sessionKey` از محتوای درخواست فقط زمانی پذیرفته می‌شود که `hooks.allowRequestSessionKey=true` (پیش‌فرض: `false`).
- `POST /hooks/<name>` → از طریق `hooks.mappings` تفکیک می‌شود
  - مقادیر `sessionKey` نگاشت که با قالب رندر شده‌اند، ورودی خارجی تلقی می‌شوند و به `hooks.allowRequestSessionKey=true` نیز نیاز دارند.

<Accordion title="جزئیات نگاشت">

- `match.path` با زیرمسیر پس از `/hooks` مطابقت دارد (برای مثال `/hooks/gmail` → `gmail`).
- `match.source` برای مسیرهای عمومی با یک فیلد محتوا مطابقت دارد.
- قالب‌هایی مانند `{{messages[0].subject}}` از محتوا می‌خوانند.
- `transform` می‌تواند به یک ماژول JS/TS اشاره کند که یک کنش هوک برمی‌گرداند.
  - `transform.module` باید مسیری نسبی باشد و در محدودهٔ `hooks.transformsDir` باقی بماند (مسیرهای مطلق و پیمایش مسیر رد می‌شوند).
  - `hooks.transformsDir` را زیر `~/.openclaw/hooks/transforms` نگه دارید؛ دایرکتوری‌های Skills فضای کاری رد می‌شوند. اگر `openclaw doctor` این مسیر را نامعتبر گزارش می‌کند، ماژول تبدیل را به دایرکتوری تبدیل‌های هوک منتقل کنید یا `hooks.transformsDir` را حذف کنید.
- `agentId` به یک عامل مشخص مسیریابی می‌کند؛ شناسه‌های ناشناخته به عامل پیش‌فرض بازمی‌گردند.
- `allowedAgentIds`: مسیریابی مؤثر عامل را محدود می‌کند، از جمله مسیر عامل پیش‌فرض هنگامی که `agentId` حذف شده است (`*` یا حذف‌شده = اجازه به همه، `[]` = رد همه).
- `defaultSessionKey`: کلید نشست ثابت اختیاری برای اجرای عامل هوک بدون `sessionKey` صریح.
- `allowRequestSessionKey`: به فراخوانندگان `/hooks/agent` و کلیدهای نشست نگاشت مبتنی بر قالب اجازه می‌دهد `sessionKey` را تنظیم کنند (پیش‌فرض: `false`).
- `allowedSessionKeyPrefixes`: فهرست مجاز پیشوند اختیاری برای مقادیر صریح `sessionKey` (درخواست + نگاشت)، برای مثال `["hook:"]`. هنگامی که هر نگاشت یا پیش‌تنظیمی از `sessionKey` قالب‌دار استفاده کند، این مورد الزامی می‌شود.
- `deliver: true` پاسخ نهایی را به یک کانال می‌فرستد؛ مقدار پیش‌فرض `channel` برابر با `last` است.
- `model` مدل زبانی بزرگ را برای این اجرای هوک بازنویسی می‌کند (اگر کاتالوگ مدل تنظیم شده باشد، باید مجاز باشد).

</Accordion>

### یکپارچه‌سازی Gmail

- پیش‌تنظیم داخلی Gmail از `sessionKey: "hook:gmail:{{messages[0].id}}"` استفاده می‌کند.
- این کلید به‌ازای هر پیام، زمینهٔ مکالمه را ایزوله می‌کند، نه ابزارها یا دسترسی به فضای کاری را. بدون نگاشت سفارشی که `agentId` را تنظیم کند، پیش‌تنظیم از عامل پیش‌فرض استفاده می‌کند.
- برای صندوق‌های ورودی غیرقابل‌اعتماد، Gmail را به یک عامل خوانندهٔ اختصاصی مسیریابی کنید و آن عامل را با [سندباکس و خط‌مشی ابزار به‌ازای هر عامل](/fa/tools/multi-agent-sandbox-tools) محدود کنید. اگر خواننده باید عامل اصلی را مطلع کند، تحویل را با [`tools.agentToAgent`](/fa/gateway/config-tools#toolsagenttoagent) محدود کنید. برای مدل تهدید و ردهٔ مدل پیشنهادی، [تزریق پرامپت](/fa/gateway/security#prompt-injection) را ببینید.
- اگر این مسیریابی به‌ازای هر پیام را حفظ می‌کنید، `hooks.allowRequestSessionKey: true` را تنظیم و `hooks.allowedSessionKeyPrefixes` را طوری محدود کنید که با فضای نام Gmail مطابقت داشته باشد، برای مثال `["hook:", "hook:gmail:"]`.
- اگر به `hooks.allowRequestSessionKey: false` نیاز دارید، به‌جای پیش‌فرض قالب‌دار، پیش‌تنظیم را با یک `sessionKey` ایستا بازنویسی کنید.

```json5
{
  hooks: {
    gmail: {
      account: "openclaw@gmail.com",
      topic: "projects/<project-id>/topics/gog-gmail-watch",
      subscription: "gog-gmail-watch-push",
      pushToken: "shared-push-token",
      hookUrl: "http://127.0.0.1:18789/hooks/gmail",
      includeBody: true,
      maxBytes: 20000,
      renewEveryMinutes: 720,
      serve: { bind: "127.0.0.1", port: 8788, path: "/" },
      tailscale: { mode: "funnel", path: "/gmail-pubsub" },
      model: "openai/gpt-5.6-sol",
      thinking: "high",
    },
  },
}
```

- در صورت پیکربندی، Gateway هنگام راه‌اندازی به‌طور خودکار `gog gmail watch serve` را اجرا می‌کند. برای غیرفعال‌سازی، `OPENCLAW_SKIP_GMAIL_WATCHER=1` را تنظیم کنید.
- یک `gog gmail watch serve` جداگانه را هم‌زمان با Gateway اجرا نکنید.

---

## میزبان Plugin بوم

```json5
{
  plugins: {
    entries: {
      canvas: {
        config: {
          host: {
            root: "~/.openclaw/workspace/canvas",
            liveReload: true,
            // enabled: false, // or OPENCLAW_SKIP_CANVAS_HOST=1
          },
        },
      },
    },
  },
}
```

- محتوای HTML/CSS/JS قابل‌ویرایش توسط عامل و A2UI را از طریق HTTP زیر پورت Gateway ارائه می‌کند:
  - `http://<gateway-host>:<gateway.port>/__openclaw__/canvas/`
  - `http://<gateway-host>:<gateway.port>/__openclaw__/a2ui/`
- فقط محلی: `gateway.bind: "loopback"` را حفظ کنید (پیش‌فرض).
- اتصال‌های غیرـloopback: مسیرهای بوم مانند سایر سطوح HTTP در Gateway به احراز هویت Gateway (توکن/گذرواژه/پراکسی قابل‌اعتماد) نیاز دارند.
- WebViewهای Node معمولاً سرآیندهای احراز هویت را ارسال نمی‌کنند؛ پس از جفت و متصل‌شدن یک Node، Gateway نشانی‌های قابلیت با دامنهٔ Node را برای دسترسی به بوم/A2UI اعلام می‌کند.
- نشانی‌های قابلیت به نشست فعال WS در Node متصل‌اند و به‌سرعت منقضی می‌شوند. از بازگشت مبتنی بر IP استفاده نمی‌شود.
- کلاینت بارگذاری مجدد زنده را به HTML ارائه‌شده تزریق می‌کند.
- در صورت خالی‌بودن، `index.html` آغازین را به‌طور خودکار ایجاد می‌کند.
- همچنین A2UI را در `/__openclaw__/a2ui/` ارائه می‌کند.
- تغییرات به راه‌اندازی مجدد Gateway نیاز دارند.
- برای دایرکتوری‌های بزرگ یا خطاهای `EMFILE`، بارگذاری مجدد زنده را غیرفعال کنید.

---

## کشف

### mDNS (Bonjour)

```json5
{
  discovery: {
    mdns: {
      mode: "minimal", // minimal | full | off
    },
  },
}
```

- `minimal` (پیش‌فرض): `cliPath` + `sshPort` را از رکوردهای TXT حذف می‌کند.
- `full`: شامل `cliPath` + `sshPort` است؛ تبلیغ چندپخشی LAN همچنان مستلزم فعال‌بودن Plugin همراه `bonjour` است.
- `off`: بدون تغییر فعال‌بودن Plugin، تبلیغ چندپخشی LAN را متوقف می‌کند.
- Plugin همراه `bonjour` در میزبان‌های macOS به‌طور خودکار راه‌اندازی می‌شود و در استقرارهای Gateway روی Linux، Windows و کانتینرها نیازمند فعال‌سازی صریح است.
- اگر نام میزبان سیستم یک برچسب DNS معتبر باشد، به‌طور پیش‌فرض از آن استفاده می‌شود و در غیر این صورت به `openclaw` بازمی‌گردد. با `OPENCLAW_MDNS_HOSTNAME` بازنویسی کنید.
- `OPENCLAW_DISABLE_BONJOUR=1` تبلیغ mDNS را کاملاً غیرفعال می‌کند و `discovery.mdns.mode` را بازنویسی می‌کند.

### گسترهٔ وسیع (DNS-SD)

```json5
{
  discovery: {
    wideArea: { enabled: true },
  },
}
```

یک ناحیهٔ DNS-SD تک‌پخشی زیر `~/.openclaw/dns/` می‌نویسد. برای کشف میان‌شبکه‌ای، آن را با یک سرور DNS (CoreDNS پیشنهاد می‌شود) + DNS تفکیک‌شدهٔ Tailscale همراه کنید.

راه‌اندازی: `openclaw dns setup --apply`.

---

## محیط

### `env` (متغیرهای محیطی درون‌خطی)

```json5
{
  env: {
    OPENROUTER_API_KEY: "sk-or-...",
    vars: {
      GROQ_API_KEY: "gsk-...",
    },
    shellEnv: {
      enabled: true,
      timeoutMs: 15000,
    },
  },
}
```

- متغیرهای محیطی درون‌خطی فقط زمانی اعمال می‌شوند که کلید در محیط فرایند وجود نداشته باشد.
- فایل‌های `.env`: `.env` در CWD + `~/.openclaw/.env` (هیچ‌کدام متغیرهای موجود را بازنویسی نمی‌کنند).
- `shellEnv`: کلیدهای موردانتظارِ موجودنبوده را از نمایهٔ پوستهٔ ورود شما وارد می‌کند.
- برای تقدم کامل، [محیط](/fa/help/environment) را ببینید.

### جای‌گذاری متغیر محیطی

با `${VAR_NAME}` در هر رشتهٔ پیکربندی به متغیرهای محیطی ارجاع دهید:

```json5
{
  gateway: {
    auth: { token: "${OPENCLAW_GATEWAY_TOKEN}" },
  },
}
```

- فقط نام‌های با حروف بزرگ مطابقت داده می‌شوند: `[A-Z_][A-Z0-9_]*`.
- متغیرهای موجودنبوده/خالی هنگام بارگذاری پیکربندی خطا ایجاد می‌کنند.
- برای `${VAR}` تحت‌اللفظی، با `$${VAR}` از آن گریز کنید.
- با `$include` کار می‌کند.

---

## رازها

ارجاع‌های راز افزایشی هستند: مقادیر متن ساده همچنان کار می‌کنند.

### `SecretRef`

از یک شکل شیء استفاده کنید:

```json5
{ source: "env" | "file" | "exec", provider: "default", id: "..." }
```

اعتبارسنجی:

- الگوی `provider`: `^[a-z][a-z0-9_-]{0,63}$`
- الگوی شناسهٔ `source: "env"`: `^[A-Z][A-Z0-9_]{0,127}$`
- شناسهٔ `source: "file"`: اشاره‌گر مطلق JSON (برای مثال `"/providers/openai/apiKey"`)
- الگوی شناسهٔ `source: "exec"`: `^[A-Za-z0-9][A-Za-z0-9._:/#-]{0,255}$` (از انتخابگرهای `secret#json_key` به‌سبک AWS پشتیبانی می‌کند)
- شناسه‌های `source: "exec"` نباید شامل بخش‌های مسیر جداشده با اسلشِ `.` یا `..` باشند (برای مثال `a/../b` رد می‌شود)

### سطح پشتیبانی‌شدهٔ اعتبارنامه

- ماتریس معیار: [سطح اعتبارنامهٔ SecretRef](/fa/reference/secretref-credential-surface)
- `secrets apply` مسیرهای اعتبارنامهٔ پشتیبانی‌شدهٔ `openclaw.json` را هدف می‌گیرد.
- ارجاع‌های `auth-profiles.json` در تفکیک زمان اجرا و پوشش ممیزی گنجانده می‌شوند.

### پیکربندی ارائه‌دهندگان راز

```json5
{
  secrets: {
    providers: {
      default: { source: "env" }, // optional explicit env provider
      filemain: {
        source: "file",
        path: "~/.openclaw/secrets.json",
        mode: "json",
        timeoutMs: 5000,
      },
      vault: {
        source: "exec",
        command: "/usr/local/bin/openclaw-vault-resolver",
        passEnv: ["PATH", "VAULT_ADDR"],
      },
    },
    defaults: {
      env: "default",
      file: "filemain",
      exec: "vault",
    },
  },
}
```

نکات:

- ارائه‌دهندهٔ `file` از `mode: "json"` و `mode: "singleValue"` پشتیبانی می‌کند (`id` باید در حالت singleValue برابر با `"value"` باشد).
- مسیرهای ارائه‌دهندهٔ فایل و exec هنگامی که تأیید ACL در Windows در دسترس نباشد، به‌صورت بسته شکست می‌خورند. `allowInsecurePath: true` را فقط برای مسیرهای قابل‌اعتمادی تنظیم کنید که قابل‌تأیید نیستند.
- ارائه‌دهندهٔ `exec` به مسیر مطلق `command` نیاز دارد و از محتوای پروتکل روی stdin/stdout استفاده می‌کند.
- به‌طور پیش‌فرض، مسیرهای فرمان پیوند نمادین رد می‌شوند. برای اجازه‌دادن به مسیرهای پیوند نمادین همراه با اعتبارسنجی مسیر مقصد تفکیک‌شده، `allowSymlinkCommand: true` را تنظیم کنید.
- اگر `trustedDirs` پیکربندی شده باشد، بررسی دایرکتوری قابل‌اعتماد روی مسیر مقصد تفکیک‌شده اعمال می‌شود.
- محیط فرزند `exec` به‌طور پیش‌فرض حداقلی است؛ متغیرهای موردنیاز را با `passEnv` به‌صراحت ارسال کنید.
- ارجاع‌های راز هنگام فعال‌سازی در یک عکس فوری درون‌حافظه‌ای تفکیک می‌شوند و سپس مسیرهای درخواست فقط آن عکس فوری را می‌خوانند.
- فیلترکردن سطح فعال هنگام فعال‌سازی اعمال می‌شود: ارجاع‌های تفکیک‌نشده در سطوح فعال، راه‌اندازی/بارگذاری مجدد را ناموفق می‌کنند، درحالی‌که سطوح غیرفعال همراه با اطلاعات تشخیصی نادیده گرفته می‌شوند.

---

## ذخیره‌سازی احراز هویت

```json5
{
  auth: {
    profiles: {
      "anthropic:default": { provider: "anthropic", mode: "api_key" },
      "anthropic:work": { provider: "anthropic", mode: "api_key" },
      "openai:personal": { provider: "openai", mode: "oauth" },
    },
    order: {
      anthropic: ["anthropic:default", "anthropic:work"],
      openai: ["openai:personal"],
    },
  },
}
```

- پروفایل‌های هر عامل در `<agentDir>/auth-profiles.json` ذخیره می‌شوند.
- `auth-profiles.json` برای حالت‌های ایستای اعتبارنامه از ارجاع‌های سطح‌مقدار (`keyRef` برای `api_key`، `tokenRef` برای `token`) پشتیبانی می‌کند.
- نگاشت‌های تخت قدیمی `auth-profiles.json` مانند `{ "provider": { "apiKey": "..." } }` قالب زمان اجرا نیستند؛ `openclaw doctor --fix` آن‌ها را به پروفایل‌های متعارف کلید API در `provider:default` بازنویسی می‌کند و یک نسخهٔ پشتیبان `.legacy-flat.*.bak` می‌سازد.
- پروفایل‌های حالت OAuth (`auth.profiles.<id>.mode = "oauth"`) از اعتبارنامه‌های پروفایل احراز هویت مبتنی بر SecretRef پشتیبانی نمی‌کنند.
- اعتبارنامه‌های ایستای زمان اجرا از اسنپ‌شات‌های حل‌شدهٔ درون‌حافظه‌ای تأمین می‌شوند؛ ورودی‌های ایستای قدیمی `auth.json` هنگام شناسایی پاک‌سازی می‌شوند.
- درون‌ریزی‌های قدیمی OAuth از `~/.openclaw/credentials/oauth.json` انجام می‌شوند.
- [OAuth](/fa/concepts/oauth) را ببینید.
- رفتار اسرار در زمان اجرا و ابزارهای `audit/configure/apply`: [مدیریت اسرار](/fa/gateway/secrets).

---

## ممیزی

```json5
{
  audit: {
    enabled: true,
    messages: "off", // off | direct | all
  },
}
```

Gateway رویدادهای ممیزی **فقط شامل فراداده** را برای اجرای عامل‌ها و
اقدام‌های ابزار در پایگاه‌دادهٔ وضعیت مشترک ثبت می‌کند. فرادادهٔ چرخهٔ عمر پیام
یک قابلیت انتخابی جداگانه است. دفترکل، هویت، زمان‌بندی، نام ابزارها و
نتایج نرمال‌شده را ذخیره می‌کند، اما هرگز اعلان‌ها، بدنهٔ پیام‌ها، آرگومان‌های ابزار،
نتایج یا متن خام خطا را ذخیره نمی‌کند. ردیف‌های پیام شناسه‌های خام حساب پلتفرم،
گفت‌وگو، پیام و مقصد را ذخیره نمی‌کنند. کلیدهای نشست اجرا/ابزار برای هم‌بستگی
در دسترس می‌مانند و ممکن است خودشان حاوی شناسه‌های حساب پلتفرم یا همتا باشند.
سوابق پس از 30 روز منقضی می‌شوند و دفترکل به 100,000 ردیف محدود است. آن‌ها را با
[`openclaw audit`](/fa/cli/audit) یا
RPC مربوط به Gateway در [`audit.activity.list`](/fa/gateway/protocol#audit-ledger-rpc) جست‌وجو کنید. برای
مدل کامل داده، مفاهیم حریم خصوصی و محدودیت‌های پوشش، [تاریخچهٔ ممیزی](/fa/gateway/audit)
را ببینید.

- `enabled`: ثبت رویدادهای ممیزی جدید (پیش‌فرض: `true`). دفترکل به‌طور
  پیش‌فرض فعال است، زیرا ردپای ممیزی‌ای که فقط پس از یک رخداد فعال شود نمی‌تواند
  آن رخداد را توضیح دهد. تنظیم `false` پس از راه‌اندازی مجدد Gateway، درج رویدادهای جدید را متوقف می‌کند؛
  سوابق موجود تا زمان انقضا خواندنی می‌مانند. فعال‌سازی دوباره، ثبت را
  از همان نقطه از سر می‌گیرد — فاصلهٔ ایجادشده به‌صورت پس‌نگر پر نمی‌شود.
- `messages`: دامنهٔ فرادادهٔ پیام (پیش‌فرض: `"off"`). `"direct"` فقط
  گفت‌وگوهای مستقیم شناخته‌شده را ثبت می‌کند. `"all"` همچنین گروه‌ها، کانال‌ها و
  انواع ناشناختهٔ گفت‌وگو را ثبت می‌کند. هر دو حالت بدون محتوا باقی می‌مانند و در مواردی
  که هم‌بستگی ممکن باشد، شناسه‌های خام را با نام‌های مستعار کلیددار محلیِ نصب
  جایگزین می‌کنند. این‌ها ابزار کمک به هم‌بستگی هستند، نه ناشناس‌سازی؛ پایگاه‌دادهٔ
  وضعیت کلید استخراج را ذخیره می‌کند، اما خروجی‌های RPC و CLI آن را ذخیره نمی‌کنند.

Gateway در حال اجرا، `audit.enabled` و `audit.messages` را هنگام راه‌اندازی دریافت می‌کند؛
پس از تغییر هرکدام از تنظیمات، آن را راه‌اندازی مجدد کنید. پوشش پیام در حال حاضر
شامل پیام‌های ورودی پذیرفته‌شده‌ای است که به توزیع مرکزی می‌رسند و برای هر
بار پاسخ خروجی منطقی اصلی که به تحویل پایدار مشترک می‌رسد، یک ردیف پایانی ثبت می‌شود.
مسیرهای محلی Plugin و ارسال مستقیم که این مرزهای مشترک را دور می‌زنند هنوز
پوشش داده نمی‌شوند. نویسندهٔ پس‌زمینهٔ محدودشده بر مبنای بهترین تلاش عمل می‌کند،
نه به‌عنوان یک بایگانی انطباقی بدون اتلاف.

---

## ثبت گزارش

```json5
{
  logging: {
    level: "info",
    file: "/tmp/openclaw/openclaw.log",
    consoleLevel: "info",
    consoleStyle: "pretty", // pretty | compact | json
    redactSensitive: "tools", // off | tools
    redactPatterns: ["\\bTOKEN\\b\\s*[=:]\\s*([\"']?)([^\\s\"']+)\\1"],
  },
}
```

- فایل گزارش پیش‌فرض: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`؛ پروفایل‌های نام‌گذاری‌شده از `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log` استفاده می‌کنند.
- برای داشتن مسیری پایدار، `logging.file` را تنظیم کنید.
- `consoleLevel` هنگامی که `--verbose` باشد، به `debug` افزایش می‌یابد.
- `maxFileBytes`: حداکثر اندازهٔ فایل گزارش فعال برحسب بایت پیش از چرخش (عدد صحیح مثبت؛ پیش‌فرض: `104857600` = 100 MB). OpenClaw حداکثر پنج بایگانی شماره‌گذاری‌شده را کنار فایل فعال نگه می‌دارد.
- `redactSensitive` / `redactPatterns`: پوشاندن بر مبنای بهترین تلاش برای خروجی کنسول، گزارش‌های فایل، رکوردهای گزارش OTLP و متن ذخیره‌شدهٔ رونوشت نشست. `redactSensitive: "off"` فقط این سیاست عمومی گزارش/رونوشت را غیرفعال می‌کند؛ سطوح ایمنی رابط کاربری/ابزار/عیب‌یابی همچنان اسرار را پیش از انتشار حذف می‌کنند.

---

## عیب‌یابی

```json5
{
  diagnostics: {
    enabled: true,
    flags: ["telegram.*"],

    otel: {
      enabled: false,
      endpoint: "https://otel-collector.example.com:4318",
      tracesEndpoint: "https://traces.example.com/v1/traces",
      metricsEndpoint: "https://metrics.example.com/v1/metrics",
      logsEndpoint: "https://logs.example.com/v1/logs",
      protocol: "http/protobuf", // http/protobuf | grpc
      headers: { "x-tenant-id": "my-org" },
      serviceName: "openclaw-gateway",
      traces: true,
      metrics: true,
      logs: false,
      logsExporter: "otlp",
      sampleRate: 1.0,
      flushIntervalMs: 5000,
      captureContent: {
        enabled: false,
        inputMessages: false,
        outputMessages: false,
        toolInputs: false,
        toolOutputs: false,
        systemPrompt: false,
        toolDefinitions: false,
      },
    },

    cacheTrace: {
      enabled: false,
      filePath: "~/.openclaw/logs/cache-trace.jsonl",
      includeMessages: true,
      includePrompt: true,
      includeSystem: true,
    },
  },
}
```

- `enabled`: کلید اصلی خروجی ابزاربندی (پیش‌فرض: `true`).
- `flags`: آرایه‌ای از رشته‌های پرچم برای فعال‌سازی خروجی گزارش هدفمند (از نویسه‌های عام مانند `"telegram.*"` یا `"*"` پشتیبانی می‌کند).
- `otel.enabled`: پایپ‌لاین خروجی OpenTelemetry را فعال می‌کند (پیش‌فرض: `false`). برای پیکربندی کامل، فهرست سیگنال‌ها و مدل حریم خصوصی، [خروجی OpenTelemetry](/fa/gateway/opentelemetry) را ببینید.
- `otel.endpoint`: نشانی URL گردآورنده برای خروجی OTel.
- `otel.tracesEndpoint` / `otel.metricsEndpoint` / `otel.logsEndpoint`: نقاط پایانی اختیاری OTLP ویژهٔ هر سیگنال. در صورت تنظیم، فقط برای همان سیگنال جایگزین `otel.endpoint` می‌شوند.
- `otel.protocol`: `"http/protobuf"` (پیش‌فرض) یا `"grpc"`.
- `otel.headers`: سرآیندهای فرادادهٔ اضافی HTTP/gRPC که همراه درخواست‌های خروجی OTel ارسال می‌شوند.
- `otel.serviceName`: نام سرویس برای ویژگی‌های منبع.
- `otel.traces` / `otel.metrics` / `otel.logs`: فعال‌سازی خروجی ردیابی، سنجه‌ها یا گزارش.
- `otel.logsExporter`: مقصد خروجی گزارش: `"otlp"` (پیش‌فرض)، `"stdout"` برای یک شیء JSON در هر خط stdout، یا `"both"`.
- `otel.sampleRate`: نرخ نمونه‌برداری ردیابی `0`-`1`.
- `otel.flushIntervalMs`: بازهٔ تخلیهٔ دوره‌ای دورسنجی برحسب ms.
- `otel.captureContent`: دریافت انتخابی محتوای خام برای ویژگی‌های span در OTEL. به‌طور پیش‌فرض غیرفعال است. مقدار بولی `true` محتوای غیرسیستمی پیام/ابزار را دریافت می‌کند؛ قالب شیء امکان می‌دهد `inputMessages`، `outputMessages`، `toolInputs`، `toolOutputs`، `systemPrompt` و `toolDefinitions` را صریحاً فعال کنید.
- `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental`: کلید محیطی برای جدیدترین قالب آزمایشی span استنتاج GenAI، شامل نام‌های span در `{gen_ai.operation.name} {gen_ai.request.model}`، نوع span در `CLIENT` و `gen_ai.provider.name` به‌جای `gen_ai.system` قدیمی. به‌طور پیش‌فرض، spanها برای سازگاری `openclaw.model.call` و `gen_ai.system` را حفظ می‌کنند؛ سنجه‌های GenAI از ویژگی‌های معنایی محدودشده استفاده می‌کنند.
- `OPENCLAW_OTEL_PRELOADED=1`: کلید محیطی برای میزبان‌هایی که قبلاً یک SDK سراسری OpenTelemetry ثبت کرده‌اند. در این حالت OpenClaw راه‌اندازی/خاموش‌سازی SDK متعلق به Plugin را نادیده می‌گیرد، درحالی‌که شنونده‌های عیب‌یابی فعال می‌مانند.
- `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`، `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` و `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT`: متغیرهای محیطی نقطهٔ پایانی ویژهٔ هر سیگنال که وقتی کلید پیکربندی متناظر تنظیم نشده باشد استفاده می‌شوند.
- `cacheTrace.enabled`: ثبت اسنپ‌شات‌های ردیابی کش برای اجراهای جاسازی‌شده (پیش‌فرض: `false`).
- `cacheTrace.filePath`: مسیر خروجی JSONL ردیابی کش (پیش‌فرض: `$OPENCLAW_STATE_DIR/logs/cache-trace.jsonl`).
- `cacheTrace.includeMessages` / `includePrompt` / `includeSystem`: کنترل محتوای گنجانده‌شده در خروجی ردیابی کش (پیش‌فرض همه: `true`).

---

## به‌روزرسانی

```json5
{
  update: {
    channel: "stable", // stable | extended-stable | beta | dev
    checkOnStart: true,

    auto: {
      enabled: false,
    },
  },
}
```

- `channel`: کانال انتشار - `"stable"`، `"extended-stable"`، `"beta"` یا `"dev"`. نسخهٔ پایدار توسعه‌یافته فقط مختص بسته است: فرمان‌های پیش‌زمینه نصب را انجام می‌دهند، درحالی‌که Gateway ممکن است راهنمایی‌های فقط‌خواندنی به‌روزرسانی را منتشر کند.
- `checkOnStart`: بررسی به‌روزرسانی‌های npm هنگام راه‌اندازی Gateway (پیش‌فرض: `true`). انتخاب‌های ذخیره‌شدهٔ نسخهٔ پایدار توسعه‌یافته از همان راهنمای فقط‌خواندنی و برنامهٔ راهنمای 24 ساعته استفاده می‌کنند.
- `auto.enabled`: فعال‌سازی به‌روزرسانی خودکار پس‌زمینه برای نصب بسته‌های پایدار و بتا (پیش‌فرض: `false`). نسخهٔ پایدار توسعه‌یافته هرگز به‌طور خودکار اعمال نمی‌شود.

---

## ACP

```json5
{
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    fallbacks: ["acpx-secondary"],
    defaultAgent: "main",
    allowedAgents: ["main", "ops"],
    stream: {
      repeatSuppression: true,
      deliveryMode: "live", // live | final_only
    },
  },
}
```

- `enabled`: دروازهٔ سراسری قابلیت ACP (پیش‌فرض: `true`؛ برای پنهان‌کردن امکانات توزیع و ایجاد ACP، `false` را تنظیم کنید).
- `dispatch.enabled`: دروازهٔ مستقل برای توزیع نوبت نشست ACP (پیش‌فرض: `true`). برای در دسترس نگه‌داشتن فرمان‌های ACP درحالی‌که اجرا مسدود است، `false` را تنظیم کنید.
- `backend`: شناسهٔ پیش‌فرض بخش پشتی زمان اجرای ACP (باید با یک Plugin ثبت‌شدهٔ زمان اجرای ACP مطابقت داشته باشد).
  ابتدا Plugin بخش پشتی را نصب کنید و اگر `plugins.allow` تنظیم شده است، شناسهٔ Plugin بخش پشتی (برای مثال `acpx`) را در آن بگنجانید؛ در غیر این صورت بخش پشتی ACP بارگذاری نخواهد شد.
- `fallbacks`: فهرست مرتب‌شدهٔ شناسه‌های بخش پشتی جایگزین ACP که وقتی بخش پشتی اصلی، پیش از تولید هرگونه خروجی، با خطایی موقت‌نما (در دسترس نبودن، محدودیت نرخ، اتمام سهمیه یا بار بیش‌ازحد) زودهنگام شکست می‌خورد، امتحان می‌شوند. هر ورودی باید با بخش پشتی یک Plugin ثبت‌شدهٔ زمان اجرای ACP مطابقت داشته باشد.
- `defaultAgent`: شناسهٔ عامل مقصد جایگزین ACP هنگامی که ایجادها مقصد صریحی مشخص نمی‌کنند.
- `allowedAgents`: فهرست مجاز شناسه‌های عامل برای نشست‌های زمان اجرای ACP؛ خالی‌بودن یعنی محدودیت اضافی وجود ندارد.
- `stream.repeatSuppression`: سرکوب خطوط تکراری وضعیت/ابزار در هر نوبت (پیش‌فرض: `true`).
- `stream.deliveryMode`: `"live"` به‌صورت افزایشی جریان می‌یابد؛ `"final_only"` تا رخدادهای پایانی نوبت بافر می‌کند.
- `stream.tagVisibility`: رکوردی از نام برچسب‌ها تا مقادیر بولی بازنویسی قابلیت مشاهده برای رخدادهای جریانی.
- `runtime.installCommand`: فرمان نصب اختیاری که هنگام راه‌اندازی اولیهٔ محیط زمان اجرای ACP اجرا می‌شود.

---

## راه‌انداز

رفتار و فراداده برای جریان‌های راه‌اندازی هدایت‌شدهٔ CLI (`onboard`، `configure`، `doctor`):

```json5
{
  wizard: {
    accessMode: "full",
    appRecommendations: true,
    lastRunAt: "2026-01-01T00:00:00.000Z",
    lastRunVersion: "2026.1.4",
    lastRunCommit: "abc1234",
    lastRunCommand: "configure",
    lastRunMode: "local",
    securityAcknowledgedAt: "2026-01-01T00:00:00.000Z",
  },
}
```

- `wizard.accessMode`: رضایت برای کشف که در آغاز راه‌اندازی هدایت‌شده انتخاب می‌شود. `"full"` (توصیه‌شده) به راه‌اندازی اجازه می‌دهد برنامه‌های هوش مصنوعی، کلیدها و محیط‌های اجرای محلی را به‌طور خودکار جست‌وجو کند؛ `"guarded"` باعث می‌شود راه‌اندازی پیش از جست‌وجو یک‌بار پرسش کند و به‌جای آن پیکربندی دستی را ارائه دهد.

- `wizard.appRecommendations` به‌طور پیش‌فرض `true` است. برای غیرفعال‌کردن توصیه‌های برنامه‌های نصب‌شده در راه‌اندازی هدایت‌شده یا کلاسیک و مسدودکردن دسترسی `device.apps` در Gateway، آن را روی `false` تنظیم کنید. میزبان‌های Node همچنان پیش از اعلام این فرمان، به پرچم جداگانهٔ اشتراک‌گذاری برنامه‌های نصب‌شده نیاز دارند که به‌طور پیش‌فرض غیرفعال است.

---

## هویت

فیلدهای هویت `agents.entries` را در [پیش‌فرض‌های عامل](/fa/gateway/config-agents#agent-defaults) ببینید.

---

## پل (قدیمی، حذف‌شده)

ساخت‌های فعلی دیگر پل TCP را شامل نمی‌شوند. Nodeها از طریق WebSocket مربوط به Gateway متصل می‌شوند. کلیدهای `bridge.*` دیگر بخشی از شِمای پیکربندی نیستند (اعتبارسنجی تا زمان حذف آن‌ها شکست می‌خورد؛ `openclaw doctor --fix` می‌تواند کلیدهای ناشناخته را حذف کند).

<Accordion title="پیکربندی پل قدیمی (مرجع تاریخی)">

```json
{
  "bridge": {
    "enabled": true,
    "port": 18790,
    "bind": "tailnet",
    "tls": {
      "enabled": true,
      "autoGenerate": true
    }
  }
}
```

</Accordion>

---

## Cron

```json5
{
  cron: {
    enabled: true,
    webhook: "https://example.invalid/legacy", // جایگزین منسوخ برای کارهای ذخیره‌شدهٔ notify:true
    webhookToken: "replace-with-dedicated-token", // توکن bearer اختیاری برای احراز هویت webhook خروجی
    sessionRetention: "24h", // رشتهٔ مدت‌زمان یا false
  },
}
```

- `sessionRetention`: مدت نگهداری نشست‌های اجرای ایزوله‌شده و تکمیل‌شدهٔ Cron پیش از پاک‌سازی ردیف‌های نشست SQLite. پاک‌سازی رونوشت‌های بایگانی‌شدهٔ Cronهای حذف‌شده را نیز کنترل می‌کند. پیش‌فرض: `24h`؛ برای غیرفعال‌کردن، `false` را تنظیم کنید.
- تاریخچهٔ اجرا به‌طور خودکار جدیدترین 2000 ردیف پایانی هر کار را نگه می‌دارد. ردیف‌های ازدست‌رفته بازهٔ پاک‌سازی 24 ساعتهٔ خود را حفظ می‌کنند.
- `webhookToken`: توکن bearer مورداستفاده برای تحویل POST به webhook در Cron (`delivery.mode = "webhook"`)؛ اگر حذف شود، هیچ هدر احراز هویتی ارسال نمی‌شود.
- `webhook`: نشانی URL جایگزین قدیمی و منسوخ webhook ‏(http/https) که `openclaw doctor --fix` برای مهاجرت کارهای ذخیره‌شده‌ای استفاده می‌کند که هنوز `notify: true` دارند؛ تحویل در زمان اجرا از `delivery.mode="webhook"` مختص هر کار به‌همراه `delivery.to`، یا هنگام حفظ تحویل اعلامی از `delivery.completionDestination` استفاده می‌کند.

### `cron.failureAlert`

```json5
{
  cron: {
    failureAlert: {
      enabled: false,
      after: 3,
      cooldownMs: 3600000,
      includeSkipped: false,
      mode: "announce",
      accountId: "main",
    },
  },
}
```

- `enabled`: هشدارهای شکست را برای کارهای Cron فعال می‌کند (پیش‌فرض: `false`).
- `after`: تعداد شکست‌های متوالی پیش از فعال‌شدن هشدار (عدد صحیح مثبت، حداقل: `1`).
- `cooldownMs`: حداقل زمان برحسب میلی‌ثانیه میان هشدارهای تکراری برای یک کار یکسان (عدد صحیح نامنفی).
- `includeSkipped`: اجراهای ردشدهٔ متوالی را در آستانهٔ هشدار محاسبه می‌کند (پیش‌فرض: `false`). اجراهای ردشده جداگانه ردیابی می‌شوند و بر پس‌روی خطای اجرا تأثیر نمی‌گذارند.
- `mode`: حالت تحویل — `"announce"` از طریق پیام کانال ارسال می‌کند؛ `"webhook"` به webhook پیکربندی‌شده POST می‌فرستد.
- `accountId`: شناسهٔ اختیاری حساب یا کانال برای محدودکردن دامنهٔ تحویل هشدار.

### `cron.failureDestination`

```json5
{
  cron: {
    failureDestination: {
      mode: "announce",
      channel: "last",
      to: "channel:C1234567890",
      accountId: "main",
    },
  },
}
```

- مقصد پیش‌فرض اعلان‌های شکست Cron برای همهٔ کارها.
- `mode`: ‏`"announce"` یا `"webhook"`؛ وقتی دادهٔ مقصد کافی وجود داشته باشد، به‌طور پیش‌فرض `"announce"` است.
- `channel`: بازنویسی کانال برای تحویل اعلامی. `"last"` آخرین کانال تحویل شناخته‌شده را دوباره استفاده می‌کند.
- `to`: مقصد صریح اعلام یا نشانی URL ‏webhook. برای حالت webhook الزامی است.
- `accountId`: بازنویسی اختیاری حساب برای تحویل.
- `delivery.failureDestination` مختص هر کار، این پیش‌فرض سراسری را بازنویسی می‌کند.
- وقتی نه مقصد شکست سراسری و نه مقصد مختص کار تنظیم شده باشد، کارهایی که از قبل از طریق `announce` تحویل می‌شوند، هنگام شکست به همان مقصد اصلی اعلام بازمی‌گردند.
- `delivery.failureDestination` فقط برای کارهای `sessionTarget="isolated"` پشتیبانی می‌شود، مگر اینکه `delivery.mode` اصلی کار، `"webhook"` باشد.

[کارهای Cron](/fa/automation/cron-jobs) را ببینید. اجراهای ایزوله‌شدهٔ Cron به‌عنوان [وظایف پس‌زمینه](/fa/automation/tasks) ردیابی می‌شوند.

## متغیرهای قالب مدل رسانه

جای‌نگهدارهای قالب که در `tools.media.models[].args` بسط داده می‌شوند:

| متغیر                    | توضیحات                                       |
| --------------------------- | ------------------------------------------------- |
| `{{Body}}`                  | متن کامل پیام ورودی                         |
| `{{RawBody}}`               | متن خام (بدون پوشش‌های تاریخچه/فرستنده)             |
| `{{BodyStripped}}`          | متن با اشاره‌های گروهی حذف‌شده                 |
| `{{From}}`                  | شناسهٔ فرستنده                                 |
| `{{To}}`                    | شناسهٔ مقصد                            |
| `{{MessageSid}}`            | شناسهٔ پیام کانال                                |
| `{{SessionId}}`             | UUID نشست فعلی                              |
| `{{IsNewSession}}`          | هنگام ایجاد نشست جدید، `"true"`                 |
| `{{AttachmentUrl}}`         | نشانی URL پیوست فعلی یا مرجع ارائه‌دهنده      |
| `{{AttachmentPath}}`        | مسیر محلی پیوست فعلی                     |
| `{{AttachmentContentType}}` | نوع محتوای MIME پیوست فعلی              |
| `{{AttachmentDir}}`         | پوشهٔ حاوی `AttachmentPath`             |
| `{{AttachmentIndex}}`       | نمایهٔ مبتنی بر صفرِ واقعیت منبع                      |
| `{{Transcript}}`            | رونوشت صوتی                                  |
| `{{Prompt}}`                | پرامپت رسانهٔ حل‌شده برای ورودی‌های CLI             |
| `{{MaxChars}}`              | حداکثر نویسه‌های خروجی حل‌شده برای ورودی‌های CLI         |
| `{{ChatType}}`              | `"direct"` یا `"group"`                           |
| `{{GroupSubject}}`          | موضوع گروه (در حد بهترین تلاش)                       |
| `{{GroupMembers}}`          | پیش‌نمایش اعضای گروه (در حد بهترین تلاش)               |
| `{{SenderName}}`            | نام نمایشی فرستنده (در حد بهترین تلاش)                 |
| `{{SenderE164}}`            | شماره تلفن فرستنده (در حد بهترین تلاش)                 |
| `{{Provider}}`              | راهنمای ارائه‌دهنده (whatsapp، telegram، discord و غیره) |

نام‌های قدیمی `{{MediaPath}}`، `{{MediaUrl}}`، `{{MediaType}}` و `{{MediaDir}}`
در طول بازهٔ سازگاری SDK افزونه همچنان در دسترس‌اند، اما
منسوخ شده‌اند. پیکربندی جدید باید از متغیرهای `Attachment*` استفاده کند.

---

## گنجاندن پیکربندی (`$include`)

پیکربندی را به چند فایل تقسیم کنید:

```json5
// ~/.openclaw/openclaw.json
{
  gateway: { port: 18789 },
  agents: { $include: "./agents.json5" },
  broadcast: {
    $include: ["./clients/mueller.json5", "./clients/schmidt.json5"],
  },
}
```

**رفتار ادغام:**

- یک فایل: شیء دربرگیرنده را جایگزین می‌کند.
- آرایه‌ای از فایل‌ها: به‌ترتیب به‌صورت عمیق ادغام می‌شوند (موارد بعدی، موارد قبلی را بازنویسی می‌کنند).
- کلیدهای هم‌سطح: پس از گنجاندن‌ها ادغام می‌شوند (مقادیر گنجانده‌شده را بازنویسی می‌کنند).
- گنجاندن‌های تودرتو: تا عمق 10 سطح.
- مسیرها: نسبت به فایل گنجاننده حل می‌شوند، اما باید درون پوشهٔ پیکربندی سطح بالا باقی بمانند (`dirname` مربوط به `openclaw.json`). شکل‌های مطلق/`../` فقط هنگامی مجازند که همچنان درون آن محدوده حل شوند. برای مجازکردن ریشه‌های اضافی خارج از پوشهٔ پیکربندی، `OPENCLAW_INCLUDE_ROOTS` (مسیرهای مطلق) را تنظیم کنید.
- محدودیت‌ها: مسیرها نباید شامل بایت تهی باشند و پیش و پس از حل‌شدن باید اکیداً کوتاه‌تر از 4096 نویسه باشند؛ اندازهٔ هر فایل گنجانده‌شده حداکثر 2 MB است.
- نوشتن‌های تحت مالکیت OpenClaw که فقط یک بخش سطح بالا با پشتوانهٔ گنجاندن تک‌فایلی را تغییر می‌دهند، مستقیماً در همان فایل گنجانده‌شده نوشته می‌شوند. برای نمونه، `plugins install` مقدار `plugins: { $include: "./plugins.json5" }` را در `plugins.json5` به‌روزرسانی می‌کند و `openclaw.json` را دست‌نخورده باقی می‌گذارد.
- گنجاندن‌های ریشه، آرایه‌های گنجاندن و گنجاندن‌های دارای بازنویسی هم‌سطح، برای نوشتن‌های تحت مالکیت OpenClaw فقط‌خواندنی‌اند؛ این نوشتن‌ها به‌جای مسطح‌کردن پیکربندی، با حالت بسته شکست می‌خورند.
- خطاها: پیام‌های واضح برای فایل‌های مفقود، خطاهای تجزیه، گنجاندن‌های حلقوی، قالب نامعتبر مسیر و طول بیش‌ازحد.

---

## مرتبط

- [پیکربندی](/fa/gateway/configuration)
- [نمونه‌های پیکربندی](/fa/gateway/configuration-examples)
- [Doctor](/fa/gateway/doctor)
