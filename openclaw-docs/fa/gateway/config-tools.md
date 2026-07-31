---
read_when:
    - پیکربندی خط‌مشی، فهرست‌های مجاز یا قابلیت‌های آزمایشی `tools.*`
    - ثبت ارائه‌دهندگان سفارشی یا بازنویسی URLهای پایه
    - راه‌اندازی نقاط پایانی خودمیزبانِ سازگار با OpenAI
sidebarTitle: Tools and custom providers
summary: پیکربندی ابزارها (سیاست، گزینه‌های آزمایشی و ابزارهای مبتنی بر ارائه‌دهنده) و راه‌اندازی ارائه‌دهنده سفارشی/نشانی پایه URL
title: پیکربندی — ابزارها و ارائه‌دهندگان سفارشی
x-i18n:
    generated_at: "2026-07-27T14:04:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 2010a2e48e8f4c8d0049e5c707bb8286e291a92312baac94301a7b5a674583c1
    source_path: gateway/config-tools.md
    workflow: 16
---

کلیدهای پیکربندی `tools.*` و راه‌اندازی ارائه‌دهنده سفارشی / URL پایه. برای عامل‌ها، کانال‌ها و دیگر کلیدهای پیکربندی سطح‌بالا، [مرجع پیکربندی](/fa/gateway/configuration-reference) را ببینید.

## ابزارها

### پروفایل‌های ابزار

`tools.profile` پیش از `tools.allow`/`tools.deny` یک فهرست مجاز پایه تعیین می‌کند:

<Note>
فرایند راه‌اندازی اولیه محلی، در صورت تنظیم‌نبودن، پیکربندی‌های محلی جدید را به‌طور پیش‌فرض روی `tools.profile: "coding"` قرار می‌دهد (پروفایل‌های صریح موجود حفظ می‌شوند).
</Note>

| پروفایل     | شامل                                                                                                                                                                                                                                                |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `minimal`   | فقط `session_status`                                                                                                                                                                                                                                   |
| `coding`    | `group:fs`، `group:runtime`، `group:web`، `group:sessions`، `group:memory`، `cron`، `get_goal`، `create_goal`، `update_goal`، `update_plan`، `ask_user`، `skill_workshop`، `image`، `image_generate`، `music_generate`، `video_generate`                |
| `messaging` | `group:messaging`، `sessions`، `sessions_list`، `sessions_history`، `sessions_search`، `conversations_list`، `conversations_send`، `conversations_turn`، `sessions_send`، `sessions_spawn`، `sessions_yield`، `subagents`، `session_status`، `ask_user` |
| `full`      | بدون محدودیت (همانند حالت تنظیم‌نشده)                                                                                                                                                                                                                          |

`coding` و `messaging` همچنین به‌طور ضمنی `bundle-mcp` (سرورهای MCP پیکربندی‌شده) را مجاز می‌کنند.

### گروه‌های ابزار

| گروه              | ابزارها                                                                                                                                                                                                                                                  |
| ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `group:runtime`    | `exec`، `process`، `code_execution` (`bash` به‌عنوان نام مستعار `exec` پذیرفته می‌شود)                                                                                                                                                                        |
| `group:fs`         | `read`، `write`، `edit`، `apply_patch`                                                                                                                                                                                                                 |
| `group:sessions`   | `sessions`، `sessions_list`، `sessions_history`، `sessions_search`، `conversations_list`، `conversations_send`، `conversations_turn`، `sessions_send`، `sessions_spawn`، `sessions_yield`، `subagents`، `session_status`، `spawn_task`، `dismiss_task` |
| `group:memory`     | `memory_search`، `memory_get`                                                                                                                                                                                                                          |
| `group:web`        | `web_search`، `x_search`، `web_fetch`                                                                                                                                                                                                                  |
| `group:ui`         | `browser`، `screen`، `terminal`، `canvas`، `show_widget`                                                                                                                                                                                               |
| `group:automation` | `heartbeat_respond`، `cron`، `gateway`                                                                                                                                                                                                                 |
| `group:messaging`  | `message`                                                                                                                                                                                                                                              |
| `group:nodes`      | `nodes`، `computer`                                                                                                                                                                                                                                    |
| `group:agents`     | `agents_list`، `get_goal`، `create_goal`، `update_goal`، `update_plan`، `ask_user`، `skill_workshop`                                                                                                                                                   |
| `group:media`      | `image`، `image_generate`، `music_generate`، `video_generate`، `tts`                                                                                                                                                                                   |
| `group:openclaw`   | همه ابزارهای داخلی بالا به‌جز `read`/`write`/`edit`/`apply_patch`/`exec`/`process`/`canvas` (ابزارهای Plugin را مستثنا می‌کند)                                                                                                                                  |
| `group:plugins`    | ابزارهای تحت مالکیت Pluginهای بارگذاری‌شده، شامل سرورهای MCP پیکربندی‌شده که از طریق `bundle-mcp` ارائه می‌شوند                                                                                                                                                           |

`spawn_task` به یک عامل کدنویسی اجازه می‌دهد بدون آغاز کار، کار پیگیریِ تأییدشده‌ای را پیشنهاد کند. رابط کنترل، عنوان و خلاصه را به‌صورت یک تراشه قابل‌اقدام نمایش می‌دهد؛ یک TUI متکی بر Gateway نیز یک درخواست تعاملی معادل نشان می‌دهد. پذیرش هرکدام، یک نشست تازه در درخت‌کاری مدیریت‌شده ایجاد می‌کند و درحالی‌که نوبت فعلی ادامه دارد، درخواست کامل را به آنجا می‌فرستد. `dismiss_task` پیشنهادی را که همچنان در انتظار است، با استفاده از `task_id` موقتیِ بازگردانده‌شده از `spawn_task` پس می‌گیرد.

این ابزارها فقط زمانی ارائه می‌شوند که سطح اپراتوری آغازکننده بتواند رویدادهای پیشنهاد وظیفه Gateway را دریافت و اجرا کند. نشست‌های کانال و نشست‌های TUI محلی/تعبیه‌شده این رویدادها را دریافت نمی‌کنند؛ انتقال‌دهنده‌های کانال پیش از آنکه بتوانند این جریان را با ایمنی ارائه دهند، به یک اقدام وظیفه نوع‌دار و قابل‌حمل نیاز دارند. پیشنهادها محلیِ فرایند هستند و با راه‌اندازی مجدد Gateway ناپدید می‌شوند. هر دو ابزار در پروفایل `coding` و `group:sessions` باقی می‌مانند، بنابراین خط‌مشی عادی `tools.allow` و `tools.deny` هنگامی که سطح از آن‌ها پشتیبانی کند، آن‌ها را به‌طور خودکار پیکربندی می‌کند.

### ابزارهای MCP و Plugin درون خط‌مشی ابزار سندباکس

سرورهای MCP پیکربندی‌شده، به‌عنوان ابزارهای تحت مالکیت Plugin و با شناسه Plugin برابر با `bundle-mcp` ارائه می‌شوند. پروفایل‌های عادی ابزار می‌توانند آن‌ها را مجاز کنند، اما `tools.sandbox.tools` یک دروازه اضافی برای نشست‌های سندباکس‌شده است. اگر حالت سندباکس `"all"` یا `"non-main"` است، هنگامی که ابزارهای MCP/Plugin باید قابل‌مشاهده باشند، یکی از ورودی‌های زیر را در فهرست مجاز ابزارهای سندباکس قرار دهید:

- `bundle-mcp` برای سرورهای MCP مدیریت‌شده توسط OpenClaw از `mcp.servers`
- شناسه Plugin برای یک Plugin بومی مشخص
- `group:plugins` برای همه ابزارهای تحت مالکیت Plugin بارگذاری‌شده
- نام دقیق ابزارهای سرور MCP یا الگوهای فراگیر سرور مانند `outlook__send_mail` یا `outlook__*`، هنگامی که فقط یک سرور را می‌خواهید

الگوهای فراگیر سرور از پیشوند سرور MCP ایمن برای ارائه‌دهنده استفاده می‌کنند، نه لزوماً کلید خام `mcp.servers`. نویسه‌های غیر `[A-Za-z0-9_-]` به `-` تبدیل می‌شوند، نام‌هایی که با حرف شروع نمی‌شوند پیشوند `mcp-` می‌گیرند، و پیشوندهای بلند یا تکراری ممکن است کوتاه شوند یا پسوند بگیرند؛ برای مثال، `mcp.servers["Outlook Graph"]` از الگویی مانند `outlook-graph__*` استفاده می‌کند.

```json5
{
  agents: { defaults: { sandbox: { mode: "all" } } },
  mcp: {
    servers: {
      outlook: { command: "node", args: ["./outlook-mcp.js"] },
    },
  },
  tools: {
    sandbox: {
      tools: {
        alsoAllow: ["web_search", "web_fetch", "memory_search", "memory_get", "bundle-mcp"],
      },
    },
  },
}
```

بدون آن ورودی لایه سندباکس، سرور MCP همچنان می‌تواند با موفقیت بارگذاری شود، درحالی‌که ابزارهایش پیش از درخواست ارائه‌دهنده فیلتر می‌شوند. برای شناسایی این وضعیت در سرورهای مدیریت‌شده توسط OpenClaw در `mcp.servers`، از `openclaw doctor` استفاده کنید. سرورهای MCP بارگذاری‌شده از مانیفست‌های Plugin همراه یا `.mcp.json` مربوط به Claude از همان دروازه سندباکس استفاده می‌کنند، اما این ابزار تشخیصی هنوز آن منابع را فهرست نمی‌کند؛ اگر ابزارهای آن‌ها در نوبت‌های سندباکس‌شده ناپدید شدند، از همان ورودی‌های فهرست مجاز استفاده کنید.

### `tools.codeMode`

`tools.codeMode` سطح عمومی حالت کد OpenClaw را فعال می‌کند. هنگام فعال‌بودن
برای اجرایی دارای ابزار، ابزارهای عادی OpenClaw پشت پل کاتالوگ درون‌سندباکس `tools.*`
قرار می‌گیرند و ابزارهای MCP از طریق فضای نام تولیدشده `MCP`
در دسترس هستند. مدل معمولاً `exec` و `wait` را می‌بیند؛ ابزارهایی مانند `computer`
که نتایج ساختاریافته آن‌ها نمی‌تواند از پل صرفاً JSON عبور کند، مستقیم باقی می‌مانند.

```json5
{
  tools: {
    codeMode: {
      enabled: true,
    },
  },
}
```

شکل کوتاه نیز پذیرفته می‌شود:

```json5
{
  tools: { codeMode: true },
}
```

اعلان‌های MCP در حالت کد از طریق سطح فایل API مجازیِ فقط‌خواندنی ارائه می‌شوند.
کد مهمان می‌تواند برای بررسی امضاهای به‌سبک TypeScript پیش از
فراخوانی `MCP.<server>.<tool>()`، توابع `API.list("mcp")` و
`API.read("mcp/<server>.d.ts")` را فراخوانی کند. برای قرارداد زمان اجرا، محدودیت‌ها و
مراحل اشکال‌زدایی، [حالت کد](/tools/code-mode) را ببینید.

### `tools.allow` / `tools.deny`

خط‌مشی سراسری مجاز/ممنوع‌سازی ابزار (ممنوع‌سازی اولویت دارد). به بزرگی و کوچکی حروف حساس نیست و از نویسه‌های عام `*` پشتیبانی می‌کند. حتی هنگامی که سندباکس Docker خاموش است اعمال می‌شود.

```json5
{
  tools: { deny: ["browser", "canvas"] },
}
```

`write` و `apply_patch` شناسه‌های ابزار جداگانه‌ای هستند. `allow: ["write"]` همچنین `apply_patch` را برای مدل‌های سازگار فعال می‌کند، اما `deny: ["write"]`، `apply_patch` را ممنوع نمی‌کند. برای مسدودکردن همه تغییرات فایل، `group:fs` را ممنوع کنید یا هر ابزار تغییردهنده را صریحاً فهرست کنید:

```json5
{
  tools: { deny: ["write", "edit", "apply_patch"] },
}
```

<Note>
`allow` و `alsoAllow` را نمی‌توان هم‌زمان در یک دامنه (`tools`، `tools.byProvider.<id>`، `agents.entries.*.tools`) تنظیم کرد — اعتبارسنجی پیکربندی آن را رد می‌کند. ورودی‌های `alsoAllow` را در `allow` ادغام کنید، یا `allow` را حذف کنید و به‌جای آن از `profile` + `alsoAllow` استفاده کنید.
</Note>

### `tools.byProvider`

ابزارها را برای ارائه‌دهندگان یا مدل‌های مشخص بیشتر محدود می‌کند. ترتیب: پروفایل پایه ← پروفایل ارائه‌دهنده ← مجاز/ممنوع.

```json5
{
  tools: {
    profile: "coding",
    byProvider: {
      "google-antigravity": { profile: "minimal" },
      "openai/gpt-5.4": { allow: ["group:fs", "sessions_list"] },
    },
  },
}
```

### `tools.toolsBySender`

ابزارها را برای درخواست‌کننده‌ای که نوبت جاری را آغاز کرده است محدود می‌کند. این سازوکار، یک دفاع چندلایه افزون بر کنترل دسترسی کانال است؛ مقادیر فرستنده باید از آداپتور کانال دریافت شوند، نه از متن پیام. این سازوکار سایر محتوای موجود در پرامپت مدل را احراز هویت نمی‌کند؛ به [کنترل‌های محدود به درخواست‌کننده و زمینه پرامپت](/fa/gateway/security#requester-scoped-controls-and-prompt-context) مراجعه کنید.

```json5
{
  tools: {
    toolsBySender: {
      "channel:discord:1234567890123": { alsoAllow: ["group:fs"] },
      "id:guest-user-id": { deny: ["group:runtime", "group:fs"] },
      "*": { deny: ["exec", "process", "write", "edit", "apply_patch"] },
    },
  },
}
```

کلیدها از پیشوندهای صریح استفاده می‌کنند: `channel:<channelId>:<senderId>`، `id:<senderId>`، `e164:<phone>`، `username:<handle>`، `name:<displayName>` یا `"*"`. شناسه‌های کانال، شناسه‌های متعارف OpenClaw هستند؛ نام‌های مستعاری مانند `teams` به `msteams` نرمال‌سازی می‌شوند. کلیدهای قدیمیِ بدون پیشوند فقط به‌عنوان `id:` پذیرفته می‌شوند. ترتیب تطبیق عبارت است از کانال+شناسه، شناسه، e164، نام کاربری، نام و سپس نویسه عام.

تنظیم هر عامل در `agents.entries.*.tools.toolsBySender`، در صورت تطبیق، حتی با یک سیاست خالی `{}`، تطبیق سراسری فرستنده را لغو می‌کند.

### `tools.elevated`

دسترسی اجرای ارتقایافته در خارج از محیط ایزوله را کنترل می‌کند:

```json5
{
  tools: {
    elevated: {
      enabled: true,
      allowFrom: {
        whatsapp: ["+15555550123"],
        discord: ["1234567890123", "987654321098765432"],
      },
    },
  },
}
```

- تنظیم مختص هر عامل (`agents.entries.*.tools.elevated`) فقط می‌تواند محدودیت بیشتری اعمال کند.
- `/elevated on|off|ask|full` وضعیت را برای هر نشست ذخیره می‌کند؛ دستورالعمل‌های درون‌خطی فقط بر یک پیام اعمال می‌شوند.
- `exec` ارتقایافته، محیط ایزوله را دور می‌زند و از مسیر خروج پیکربندی‌شده استفاده می‌کند (به‌طور پیش‌فرض `gateway`، یا هنگامی که هدف اجرا `node` باشد، `node`).

### `tools.exec`

```json5
{
  tools: {
    exec: {
      backgroundMs: 10000,
      timeoutSec: 1800,
      cleanupMs: 1800000,
      approvalRunningNoticeMs: 10000,
      notifyOnExit: true,
      notifyOnExitEmptySuccess: false,
      commandHighlighting: false,
      applyPatch: {
        enabled: true,
        allowModels: ["gpt-5.6-sol"],
      },
    },
  },
}
```

مقادیر نمایش‌داده‌شده به‌جز `applyPatch.allowModels` پیش‌فرض هستند (که به‌طور پیش‌فرض خالی/تنظیم‌نشده است و یعنی هر مدل سازگاری می‌تواند از `apply_patch` استفاده کند). هنگامی که اجرای مبتنی بر تأیید طولانی شود، `approvalRunningNoticeMs` اعلان در حال اجرا منتشر می‌کند؛ `0` آن را غیرفعال می‌کند.

### `tools.loopDetection`

بررسی‌های ایمنی حلقه ابزار **به‌طور پیش‌فرض غیرفعال هستند**. برای فعال‌کردن تشخیص، `enabled: true` را تنظیم کنید. تنظیمات را می‌توان به‌صورت سراسری در `tools.loopDetection` تعریف کرد و برای هر عامل در `agents.entries.*.tools.loopDetection` لغو کرد.

```json5
{
  tools: {
    loopDetection: {
      enabled: true,
    },
  },
}
```

### `tools.web`

```json5
{
  tools: {
    web: {
      search: {
        enabled: true,
        apiKey: "brave_api_key", // یا متغیر محیطی BRAVE_API_KEY (ارائه‌دهنده Brave)
        maxResults: 5,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
      },
      fetch: {
        enabled: true,
        provider: "firecrawl", // اختیاری؛ برای تشخیص خودکار حذف کنید
        maxChars: 20000,
        maxCharsCap: 20000,
        maxResponseBytes: 750000,
        timeoutSeconds: 30,
        cacheTtlMinutes: 15,
        maxRedirects: 3,
        readability: true,
        userAgent: "custom-ua",
      },
    },
  },
}
```

مقادیر نمایش‌داده‌شده به‌جز `provider` و `userAgent` پیش‌فرض هستند. `maxResponseBytes` به بازه 32000–10000000 محدود می‌شود؛ `maxChars` به `maxCharsCap` محدود می‌شود (برای مجازکردن پاسخ‌های بزرگ‌تر، `maxCharsCap` را افزایش دهید).

### `tools.media`

درک رسانه‌های ورودی (تصویر/صدا/ویدئو) را پیکربندی می‌کند:

```json5
{
  tools: {
    media: {
      concurrency: 2,
      models: [
        { provider: "openai", model: "gpt-4o-mini-transcribe", capabilities: ["audio"] },
        {
          type: "cli",
          command: "whisper",
          args: ["--model", "base", "{{AttachmentPath}}"],
          capabilities: ["audio"],
        },
        { provider: "ollama", model: "gemma4:26b", capabilities: ["image"] },
        { provider: "google", model: "gemini-3-flash-preview", capabilities: ["video"] },
      ],
      audio: { enabled: true, preferredModel: "openai/gpt-4o-mini-transcribe" },
      image: { enabled: true, preferredModel: "ollama/gemma4:26b" },
      video: { enabled: true },
    },
  },
}
```

`tools.media.models` تنها فهرست مدل پیکربندی‌شده است. هر ورودی قابلیت‌هایی را که مدیریت می‌کند مشخص می‌سازد. گزینشگر اختیاری `preferredModel`، مقادیر `provider/model`، یک شناسه مدل، `provider:<id>` برای ورودی‌های پیش‌فرض ارائه‌دهنده، یا `cli:command` را می‌پذیرد؛ ورودی‌های منطبق به ابتدای ترتیب جایگزین‌های آن قابلیت منتقل می‌شوند. پرامپت‌ها، محدودیت‌ها، تنظیمات درخواست، دامنه، سیاست پیوست و بازتاب رونوشت صوتیِ مختص هر قابلیت، برای مدل‌های پیکربندی‌شده و مدل‌های شناسایی‌شده به‌صورت خودکار در حالت پیش‌فرض باقی می‌مانند؛ یک ورودی مدل می‌تواند فیلدهای مختص مدل را لغو کند.

<AccordionGroup>
  <Accordion title="فیلدهای ورودی مدل رسانه">
    **ورودی ارائه‌دهنده** (`type: "provider"` یا حذف‌شده):

    - `provider`: شناسه ارائه‌دهنده API (`openai`، `anthropic`، `google`/`gemini`، `groq` و غیره)
    - `model`: لغو شناسه مدل
    - `profile` / `preferredProfile`: انتخاب پروفایل `auth-profiles.json`

    **ورودی CLI** (`type: "cli"`):

    - `command`: فایل اجرایی برای اجرا
    - `args`: آرگومان‌های قالب‌بندی‌شده (از `{{AttachmentPath}}`، `{{AttachmentUrl}}`، `{{AttachmentContentType}}`، `{{AttachmentDir}}`، `{{AttachmentIndex}}`، `{{Prompt}}`، `{{MaxChars}}` و غیره پشتیبانی می‌کند؛ `openclaw doctor --fix` جای‌نگهدارهای منسوخ‌شدهٔ `{input}` را به `{{AttachmentPath}}` مهاجرت می‌دهد). نام‌های مستعار قدیمی‌تر `{{MediaPath}}`، `{{MediaUrl}}`، `{{MediaType}}` و `{{MediaDir}}` در طول بازهٔ سازگاری خود همچنان در دسترس‌اند، اما منسوخ شده‌اند.

    **فیلدهای مشترک:**

    - `capabilities`: فهرستی شامل یک یا چند مورد از `image`، `audio` و `video`.
    - `prompt`، `maxChars`، `maxBytes`، `timeoutSeconds`، `language`: بازنویسی‌های مختص هر ورودی.
    - ورودی‌های منطبق `timeoutSeconds` برای مدل تصویر، هنگام فراخوانی ابزار صریح `image` توسط عامل نیز اعمال می‌شوند. برای درک تصویر، این مهلت زمانی بر خود درخواست اعمال می‌شود و با کارهای آماده‌سازی پیشین کاهش نمی‌یابد.
    - در صورت شکست، ورودی بعدی به‌عنوان جایگزین استفاده می‌شود.

    احراز هویت ارائه‌دهنده از ترتیب استاندارد پیروی می‌کند: `auth-profiles.json` → متغیرهای محیطی → `models.providers.*.apiKey`.

  </Accordion>
</AccordionGroup>

### `tools.agentToAgent`

```json5
{
  tools: {
    agentToAgent: {
      enabled: false,
      allow: ["home", "work"],
    },
  },
}
```

### `tools.sessions`

کنترل می‌کند ابزارهای نشست (`sessions_list`، `sessions_history`، `sessions_send`) کدام نشست‌ها را می‌توانند هدف قرار دهند.

پیش‌فرض: `tree` (نشست فعلی + نشست‌های ایجادشده توسط آن، مانند عامل‌های فرعی، به‌علاوهٔ نشست‌های گروهی تحت نظارت محیطی برای همان عامل).

```json5
{
  tools: {
    sessions: {
      // "self" | "tree" | "agent" | "all"
      visibility: "tree",
    },
  },
}
```

<AccordionGroup>
  <Accordion title="دامنه‌های مشاهده‌پذیری">
    - `self`: فقط کلید نشست فعلی.
    - `tree`: نشست فعلی + نشست‌های ایجادشده توسط نشست فعلی (عامل‌های فرعی). برای عملیات خواندن، نشست‌های گروهی همان عامل را نیز شامل می‌شود که نشست فعلی از طریق آگاهی محیطی گروه بر آن‌ها نظارت دارد.
    - `agent`: هر نشستی که به شناسهٔ عامل فعلی تعلق دارد (اگر نشست‌های مجزای هر فرستنده را تحت همان شناسهٔ عامل اجرا کنید، ممکن است کاربران دیگر را نیز شامل شود).
    - `all`: هر نشست. هدف‌گیری بین‌عاملی همچنان به `tools.agentToAgent` نیاز دارد.
    - محدودیت سندباکس: وقتی نشست فعلی در سندباکس است و `agents.defaults.sandbox.sessionToolsVisibility="spawned"` (حالت پیش‌فرض)، مشاهده‌پذیری به‌اجبار روی `tree` تنظیم می‌شود، حتی اگر `tools.sessions.visibility="all"`.
    - وقتی `all` نباشد، `sessions_list` شامل یک فیلد فشردهٔ `visibility` است
      که حالت مؤثر را توصیف می‌کند و هشدار می‌دهد ممکن است برخی نشست‌های خارج
      از دامنهٔ فعلی حذف شوند.

  </Accordion>
</AccordionGroup>

با `session.dmScope: "main"` پیش‌فرض، فعالیت انسانی در یک گروه، نشست گروهی همان عامل را
به‌صورت محیطی برای نشست اصلی عامل قابل مشاهده می‌کند. در پیکربندی چندکاربره، `"main"` همچنین
یک نشست پیام مستقیم را میان کاربران به اشتراک می‌گذارد؛ بنابراین هر کاربری که به آن هدایت شود می‌تواند گروه‌های تحت نظارت محیطی را بخواند،
از جمله از طریق `memory_search` حافظهٔ نشست. برای جداسازی پیام‌های مستقیم، از `dmScope` مختص هر همتا استفاده کنید، یا
`tools.sessions.visibility: "self"` را تنظیم کنید تا خواندن نشست‌های تحت نظارت محیطی غیرفعال شود.

### `tools.sessions_spawn`

پشتیبانی از پیوست درون‌خطی را برای `sessions_spawn` کنترل می‌کند.

```json5
{
  tools: {
    sessions_spawn: {
      attachments: {
        enabled: false, // انتخابی: برای مجاز کردن پیوست‌های فایل درون‌خطی روی true تنظیم کنید
        maxTotalBytes: 5242880, // مجموعاً 5 MB در همهٔ فایل‌ها
        maxFiles: 50,
        maxFileBytes: 1048576, // برای هر فایل 1 MB
        retainOnSessionKeep: false, // وقتی cleanup="keep" است، پیوست‌ها را نگه دارید
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="نکات پیوست">
    - پیوست‌ها به `enabled: true` نیاز دارند.
    - پیوست‌های عامل فرعی با یک `.manifest.json` در `.openclaw/attachments/<uuid>/` داخل فضای کاری فرزند قرار می‌گیرند.
    - پیوست‌های ACP فقط می‌توانند تصویر باشند و پس از رعایت همان محدودیت‌های تعداد فایل، بایت هر فایل و مجموع بایت‌ها، به‌صورت درون‌خطی به زمان اجرای ACP ارسال می‌شوند.
    - محتوای پیوست به‌طور خودکار از ماندگارسازی رونوشت حذف می‌شود.
    - ورودی‌های Base64 با بررسی سخت‌گیرانهٔ الفبا/پدینگ و محافظ اندازه پیش از رمزگشایی اعتبارسنجی می‌شوند.
    - مجوزهای فایل پیوست عامل فرعی برای پوشه‌ها `0700` و برای فایل‌ها `0600` است.
    - پاک‌سازی عامل فرعی از سیاست `cleanup` پیروی می‌کند: `delete` همیشه پیوست‌ها را حذف می‌کند؛ `keep` تنها وقتی آن‌ها را نگه می‌دارد که `retainOnSessionKeep: true`.

  </Accordion>
</AccordionGroup>

<a id="toolsexperimental"></a>

### `tools.experimental`

پرچم‌های آزمایشی ابزارهای داخلی. به‌طور پیش‌فرض خاموش‌اند، مگر اینکه قاعدهٔ فعال‌سازی خودکار GPT-5 با عاملیت سخت‌گیرانه اعمال شود.

```json5
{
  tools: {
    experimental: {
      planTool: true, // فعال‌سازی update_plan آزمایشی
    },
  },
}
```

- `planTool`: ابزار ساخت‌یافتهٔ `update_plan` را برای پیگیری کارهای چندمرحله‌ای غیرساده فعال می‌کند.
- پیش‌فرض: `false`، مگر اینکه `agents.defaults.embeddedAgent.executionContract` (یا بازنویسی مختص هر عامل) برای اجرای ارائه‌دهندهٔ `openai` در برابر شناسهٔ مدل خانوادهٔ GPT-5 روی `"strict-agentic"` تنظیم شده باشد (این مورد اجرای OpenAI Codex CLI را نیز پوشش می‌دهد، زیرا مسیریابی احراز هویت/مدل Codex زیر ارائه‌دهندهٔ `openai` قرار دارد). برای اجبار به فعال‌سازی ابزار خارج از این دامنه، `true` را تنظیم کنید؛ یا برای خاموش نگه‌داشتن آن حتی در اجراهای GPT-5 با عاملیت سخت‌گیرانه، `false` را تنظیم کنید.
- وقتی فعال باشد، اعلان سیستم راهنمای استفاده را نیز اضافه می‌کند تا مدل فقط برای کارهای قابل‌توجه از آن استفاده کند و حداکثر یک مرحله را در وضعیت `in_progress` نگه دارد.

### `agents.defaults.subagents`

```json5
{
  agents: {
    defaults: {
      subagents: {
        allowAgents: ["research"],
        model: "minimax/MiniMax-M2.7",
        maxConcurrent: 8,
        runTimeoutSeconds: 900,
        announceTimeoutMs: 120000,
        archiveAfterMinutes: 60,
      },
    },
  },
}
```

- `model`: مدل پیش‌فرض برای زیرعامل‌های ایجادشده. در صورت حذف، زیرعامل‌ها مدل فراخواننده را به ارث می‌برند.
- `allowAgents`: فهرست مجاز پیش‌فرض شناسه‌های عامل مقصد پیکربندی‌شده برای `sessions_spawn`، هنگامی که عامل درخواست‌کننده `subagents.allowAgents` خودش را تنظیم نکرده باشد (`["*"]` = هر مقصد پیکربندی‌شده؛ پیش‌فرض: فقط همان عامل). ورودی‌های منسوخی که پیکربندی عاملشان حذف شده است، از سوی `sessions_spawn` رد و از `agents_list` حذف می‌شوند؛ برای پاک‌سازی آن‌ها `openclaw doctor --fix` را اجرا کنید.
- `maxConcurrent`: حداکثر اجرای هم‌زمان زیرعامل‌ها. پیش‌فرض: `8`.
- `runTimeoutSeconds`: مهلت زمانی (ثانیه) برای `sessions_spawn`، هنگامی که فراخواننده مقدار جایگزین خودش را ارسال نمی‌کند. پیش‌فرض: `0` (بدون مهلت زمانی)؛ `900` نمایش‌داده‌شده در بالا یک مقدار رایجِ اختیاری است، نه پیش‌فرض داخلی.
- `announceTimeoutMs`: مهلت زمانی هر فراخوانی (میلی‌ثانیه) برای تلاش‌های تحویل اعلان `agent` در Gateway. پیش‌فرض: `120000`. تلاش‌های مجدد گذرا ممکن است زمان انتظار کلی اعلان را از یک مهلت زمانی پیکربندی‌شده بیشتر کنند.
- `archiveAfterMinutes`: تعداد دقایق پس از تکمیل نشست زیرعامل تا بایگانی خودکار آن. پیش‌فرض: `60`؛ `0` بایگانی خودکار را غیرفعال می‌کند.
- سیاست ابزار برای هر زیرعامل: `tools.subagents.tools.allow` / `tools.subagents.tools.deny`.

---

## ارائه‌دهندگان سفارشی و URLهای پایه

Pluginهای ارائه‌دهنده، ردیف‌های کاتالوگ مدل خود را منتشر می‌کنند. ارائه‌دهندگان سفارشی را از طریق `models.providers` در پیکربندی یا `~/.openclaw/agents/<agentId>/agent/models.json` اضافه کنید.

پیکربندی `baseUrl` برای یک ارائه‌دهنده سفارشی/محلی، تصمیم محدود اعتماد شبکه برای درخواست‌های HTTP مدل نیز هست: OpenClaw همان مبدأ دقیق `scheme://host:port` را از مسیر واکشی محافظت‌شده مجاز می‌کند، بدون افزودن گزینه پیکربندی جداگانه یا اعتماد به دیگر مبدأهای خصوصی.

```json5
{
  models: {
    mode: "merge", // ادغام (پیش‌فرض) | جایگزینی
    providers: {
      "custom-proxy": {
        baseUrl: "http://localhost:4000/v1",
        apiKey: "LITELLM_KEY",
        api: "openai-completions", // openai-completions | openai-responses | anthropic-messages | google-generative-ai | غیره
        models: [
          {
            id: "llama-3.1-8b",
            name: "Llama 3.1 8B",
            reasoning: false,
            input: ["text"],
            cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
            contextWindow: 128000,
            contextTokens: 96000,
            maxTokens: 32000,
          },
        ],
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="احراز هویت و تقدم ادغام">
    - برای نیازهای احراز هویت سفارشی از `authHeader: true` + `headers` استفاده کنید.
    - ریشه پیکربندی عامل را با `OPENCLAW_AGENT_DIR` جایگزین کنید.
    - تقدم ادغام برای شناسه‌های ارائه‌دهنده منطبق:
      - مقادیر غیرخالی `models.json` `baseUrl` عامل اولویت دارند.
      - مقادیر غیرخالی `apiKey` عامل فقط هنگامی اولویت دارند که آن ارائه‌دهنده در زمینه فعلی پیکربندی/نمایه احراز هویت تحت مدیریت SecretRef نباشد.
      - مقادیر `apiKey` ارائه‌دهنده تحت مدیریت SecretRef به‌جای ذخیره‌سازی رازهای حل‌شده، از نشانگرهای منبع (`ENV_VAR_NAME` برای ارجاع‌های محیطی، `secretref-managed` برای ارجاع‌های فایل/اجرا) تازه‌سازی می‌شوند.
      - مقادیر سرآیند ارائه‌دهنده تحت مدیریت SecretRef از نشانگرهای منبع (`secretref-env:ENV_VAR_NAME` برای ارجاع‌های محیطی، `secretref-managed` برای ارجاع‌های فایل/اجرا) تازه‌سازی می‌شوند.
      - `apiKey`/`baseUrl` خالی یا مفقود عامل به `models.providers` در پیکربندی بازمی‌گردند.
      - `contextWindow`/`maxTokens` مدل منطبق: در صورت وجود و معتبر بودن مقدار صریح پیکربندی (یک عدد متناهی مثبت)، همان مقدار اولویت دارد؛ در غیر این صورت مقدار ضمنی/تولیدشده کاتالوگ استفاده می‌شود.
      - `contextTokens` مدل منطبق از همان قاعده «صریح اولویت دارد، وگرنه ضمنی» پیروی می‌کند؛ برای محدود کردن زمینه مؤثر بدون تغییر فراداده بومی مدل از آن استفاده کنید.
      - کاتالوگ‌های Plugin ارائه‌دهنده به‌صورت قطعه‌های کاتالوگ تولیدشده و متعلق به Plugin در وضعیت Plugin عامل ذخیره می‌شوند.
      - هنگامی که می‌خواهید پیکربندی، `models.json` را کاملاً بازنویسی و از ادغام قطعه‌های کاتالوگ متعلق به Plugin صرف‌نظر کند، از `models.mode: "replace"` استفاده کنید.
      - ماندگاری نشانگر بر مرجعیت منبع استوار است: نشانگرها از تصویر لحظه‌ای پیکربندی منبع فعال (پیش از حل‌سازی) نوشته می‌شوند، نه از مقادیر راز حل‌شده زمان اجرا.

  </Accordion>
</AccordionGroup>

### جزئیات فیلدهای ارائه‌دهنده

<AccordionGroup>
  <Accordion title="کاتالوگ سطح بالا">
    - `models.mode`: رفتار کاتالوگ ارائه‌دهنده (`merge` یا `replace`).
    - `models.providers`: نگاشت ارائه‌دهندگان سفارشی با کلید شناسه ارائه‌دهنده.
      - ویرایش‌های ایمن: برای به‌روزرسانی‌های افزایشی از `openclaw config set models.providers.<id> '<json>' --strict-json --merge` یا `openclaw config set models.providers.<id>.models '<json-array>' --strict-json --merge` استفاده کنید. `config set` جایگزینی‌های مخرب را رد می‌کند، مگر آنکه `--replace` را ارسال کنید.

  </Accordion>
  <Accordion title="اتصال و احراز هویت ارائه‌دهنده">
    - `models.providers.*.api`: سازگارکننده درخواست (`openai-completions`، `openai-responses`، `openai-chatgpt-responses`، `anthropic-messages`، `google-generative-ai`، `google-vertex`، `github-copilot`، `bedrock-converse-stream`، `ollama`، `azure-openai-responses`). برای بک‌اندهای خودمیزبان `/v1/chat/completions` مانند MLX، vLLM، SGLang و بیشتر سرورهای محلی سازگار با OpenAI، از `openai-completions` استفاده کنید. ارائه‌دهنده سفارشی دارای `baseUrl` اما بدون `api` به‌طور پیش‌فرض از `openai-completions` استفاده می‌کند؛ `openai-responses` را فقط هنگامی تنظیم کنید که بک‌اند از `/v1/responses` پشتیبانی کند.
    - `models.providers.*.apiKey`: اعتبارنامه ارائه‌دهنده (جایگزینی SecretRef/محیط ترجیح داده می‌شود).
    - `models.providers.*.auth`: راهبرد احراز هویت (`api-key`، `token`، `oauth`، `aws-sdk`).
    - `models.providers.*.contextWindow`: پنجره زمینه بومی پیش‌فرض برای مدل‌های این ارائه‌دهنده، هنگامی که ورودی مدل `contextWindow` را تنظیم نمی‌کند.
    - `models.providers.*.contextTokens`: سقف پیش‌فرض زمینه مؤثر زمان اجرا برای مدل‌های این ارائه‌دهنده، هنگامی که ورودی مدل `contextTokens` را تنظیم نمی‌کند.
    - `models.providers.*.maxTokens`: سقف پیش‌فرض توکن خروجی برای مدل‌های این ارائه‌دهنده، هنگامی که ورودی مدل `maxTokens` را تنظیم نمی‌کند.
    - `models.providers.*.timeoutSeconds`: مهلت زمانی اختیاری درخواست HTTP مدل برای هر ارائه‌دهنده بر حسب ثانیه، شامل اتصال، سرآیندها، بدنه و مدیریت لغو کل درخواست.
    - `models.providers.*.injectNumCtxForOpenAICompat`: برای Ollama + `openai-completions`، مقدار `options.num_ctx` را به درخواست‌ها تزریق می‌کند (پیش‌فرض: `true`).
    - `models.providers.*.authHeader`: در صورت نیاز، انتقال اعتبارنامه را در سرآیند `Authorization` اجباری می‌کند.
    - `models.providers.*.baseUrl`: URL پایه API بالادستی.
    - `models.providers.*.headers`: سرآیندهای ایستای اضافی برای مسیریابی پراکسی/مستأجر.

  </Accordion>
  <Accordion title="جایگزینی‌های انتقال درخواست">
    `models.providers.*.request`: جایگزینی‌های انتقال برای درخواست‌های HTTP ارائه‌دهنده مدل.

    - `request.headers`: سرآیندهای اضافی (با پیش‌فرض‌های ارائه‌دهنده ادغام می‌شوند). مقادیر SecretRef را می‌پذیرند.
    - `request.auth`: جایگزینی راهبرد احراز هویت. حالت‌ها: `"provider-default"` (استفاده از احراز هویت داخلی ارائه‌دهنده)، `"authorization-bearer"` (با `token`)، `"header"` (با `headerName`، `value` و `prefix` اختیاری).
    - `request.proxy`: جایگزینی پراکسی HTTP. حالت‌ها: `"env-proxy"` (استفاده از متغیرهای محیطی `HTTP_PROXY`/`HTTPS_PROXY`) و `"explicit-proxy"` (با `url`). هر دو حالت یک زیربخش اختیاری `tls` را می‌پذیرند.
    - `request.tls`: جایگزینی TLS برای اتصال‌های مستقیم. فیلدها: `ca`، `cert`، `key`، `passphrase` (همگی SecretRef را می‌پذیرند)، `serverName`، `insecureSkipVerify`.
    - `request.allowPrivateNetwork`: هنگامی که `true` است، درخواست‌های HTTP ارائه‌دهنده مدل را از طریق محافظ واکشی HTTP ارائه‌دهنده به محدوده‌های خصوصی، CGNAT یا مشابه مجاز می‌کند. URLهای پایه ارائه‌دهنده سفارشی/محلی از قبل به مبدأ دقیق پیکربندی‌شده اعتماد دارند، به‌جز مبدأهای فراداده/پیوند-محلی که بدون فعال‌سازی صریح همچنان مسدود می‌مانند. برای انصراف از اعتماد به مبدأ دقیق، این مقدار را روی `false` تنظیم کنید. WebSocket برای سرآیندها/TLS از همان `request` استفاده می‌کند، اما مشمول آن دروازه SSRF واکشی نیست. پیش‌فرض `false`.

  </Accordion>
  <Accordion title="ورودی‌های کاتالوگ مدل">
    - `models.providers.*.models`: ورودی‌های صریح کاتالوگ مدل ارائه‌دهنده.
    - `models.providers.*.models.*.input`: شیوه‌های ورودی مدل. برای مدل‌های فقط‌متنی از `["text"]` و برای مدل‌های بومی تصویر/بینایی از `["text", "image"]` استفاده کنید. پیوست‌های تصویر فقط هنگامی به نوبت‌های عامل تزریق می‌شوند که مدل انتخاب‌شده دارای قابلیت تصویر علامت‌گذاری شده باشد.
    - `models.providers.*.models.*.contextWindow`: فراداده پنجره زمینه بومی مدل. این مقدار برای آن مدل جایگزین `contextWindow` سطح ارائه‌دهنده می‌شود.
    - `models.providers.*.models.*.contextTokens`: سقف اختیاری زمینه زمان اجرا. این مقدار جایگزین `contextTokens` سطح ارائه‌دهنده می‌شود؛ هنگامی از آن استفاده کنید که بودجه زمینه مؤثری کوچک‌تر از `contextWindow` بومی مدل می‌خواهید؛ `openclaw models list` در صورت تفاوت، هر دو مقدار را نمایش می‌دهد.

    #### اعلان قابلیت‌های ارائه‌دهنده سفارشی

    کاتالوگ‌های ارائه‌دهنده مالک `compat` برای مسیرهای مدل همراه و شناخته‌شده در کاتالوگ هستند. آن پرچم‌ها را در پیکربندی کپی نکنید: وقتی `api` و `baseUrl` پیکربندی‌شده همچنان آن مسیر را شناسایی می‌کنند، OpenClaw از ردیف کاتالوگ استفاده می‌کند. `openclaw doctor --fix` جایگزینی‌های قدیمی منطبق را حذف و مقادیر متفاوت را برای بازبینی گزارش می‌کند.

    یک بلوک `compat` برای یک ارائه‌دهنده واقعاً سفارشی، مدل سفارشی یا مدل کاتالوگ که به نقطه پایانی دیگری هدایت شده است، همچنان پشتیبانی می‌شود. فقط قابلیت‌هایی را تنظیم کنید که در برابر آن نقطه پایانی تأیید شده‌اند:

    | کلید مسیر سفارشی | قرارداد زمان اجرا |
    | --- | --- |
    | `supportsStore` | فیلد درخواست `store` متعلق به OpenAI را می‌پذیرد. |
    | `supportsPromptCacheKey` | کلیدهای وابستگی نشست/حافظه نهان پرامپت OpenAI را می‌پذیرد. |
    | `supportsDeveloperRole` | پیام‌های `developer` را به‌جای الزام `system` می‌پذیرد. |
    | `supportsReasoningEffort` | کنترل میزان تلاش استدلال را می‌پذیرد. |
    | `supportsTemperature` | مقدار `temperature` را برای این مدل و سازگارکننده می‌پذیرد. |
    | `supportsUsageInStreaming` | فراداده مصرف را در پاسخ‌های جریانی منتشر می‌کند. |
    | `supportsTools` | از فراخوانی ساخت‌یافته ابزار/تابع پشتیبانی می‌کند. برای غیرفعال کردن ابزارها، `false` را تنظیم کنید. |
    | `supportsStrictMode` | شِمای سخت‌گیرانه ابزار را می‌پذیرد. |
    | `requiresStringContent` | به محتوای پیام Chat Completions به‌شکل رشته ساده نیاز دارد. |
    | `strictMessageKeys` | الزام می‌کند پیام‌های خروجی فقط شامل کلیدهای پذیرفته‌شده باشند. |
    | `visibleReasoningDetailTypes` | انواع بلوک جزئیات استدلال را که نمایششان در رونوشت‌ها ایمن است، نام‌گذاری می‌کند. |
    | `supportedReasoningEfforts` | برچسب‌های استدلال پذیرفته‌شده نقطه پایانی را فهرست می‌کند. |
    | `reasoningEffortMap` | برچسب‌های تفکر OpenClaw را به برچسب‌های ویژه نقطه پایانی نگاشت می‌کند. |
    | `maxTokensField` | `max_tokens` یا `max_completion_tokens` را انتخاب می‌کند. |
    | `thinkingFormat` | گویش محموله استدلال نقطه پایانی را انتخاب می‌کند. |
    | `requiresToolResultName` | وجود نام ابزار در پیام‌های نتیجه ابزار را الزامی می‌کند. |
    | `requiresAssistantAfterToolResult` | وجود پیام دستیار پس از نتایج ابزار را الزامی می‌کند. |
    | `requiresThinkingAsText` | استدلال را به‌جای محتوای ساخت‌یافته، به‌صورت متن بازپخش می‌کند. |
    | `requiresReasoningContentOnAssistantMessages` | هنگام بازپخش، `reasoning_content` به‌سبک DeepSeek را حفظ می‌کند. |
    | `toolSchemaProfile` | یک نمایه عادی‌سازی شِمای ابزار تعریف‌شده توسط ارائه‌دهنده را انتخاب می‌کند. |
    | `unsupportedToolSchemaKeywords` | کلیدواژه‌های نام‌برده JSON Schema را که نقطه پایانی رد می‌کند، حذف می‌کند. |
    | `toolCallArgumentsEncoding` | کدگذاری آرگومان فراخوانی ابزار نقطه پایانی را انتخاب می‌کند. |
    | `requiresOpenAiAnthropicToolPayload` | فراخوانی‌های ابزار با قالب OpenAI را به محموله‌های خانواده Anthropic تبدیل می‌کند. |

  </Accordion>
  <Accordion title="کشف Amazon Bedrock">
    - `plugins.entries.amazon-bedrock.config.discovery`: ریشه تنظیمات کشف خودکار Bedrock.
    - `plugins.entries.amazon-bedrock.config.discovery.enabled`: فعال/غیرفعال‌کردن کشف ضمنی.
    - `plugins.entries.amazon-bedrock.config.discovery.region`: منطقه AWS برای کشف.
    - `plugins.entries.amazon-bedrock.config.discovery.providerFilter`: فیلتر اختیاری شناسه ارائه‌دهنده برای کشف هدفمند.
    - `plugins.entries.amazon-bedrock.config.discovery.refreshInterval`: فاصله زمانی نظرسنجی برای تازه‌سازی کشف.
    - `plugins.entries.amazon-bedrock.config.discovery.defaultContextWindow`: پنجره زمینه جایگزین برای مدل‌های کشف‌شده.
    - `plugins.entries.amazon-bedrock.config.discovery.defaultMaxTokens`: حداکثر توکن‌های خروجی جایگزین برای مدل‌های کشف‌شده.

  </Accordion>
</AccordionGroup>

راه‌اندازی تعاملی ارائه‌دهنده سفارشی، ورودی تصویر را برای الگوهای شناخته‌شده شناسه مدل بینایی استنباط می‌کند؛ از جمله GPT-4o/GPT-4.1/GPT-5+، خانواده‌های استدلالی `o1`/`o3`/`o4`، Claude، Gemini، هر شناسه دارای پسوند `-vl` (مانند Qwen-VL و موارد مشابه) و خانواده‌های نام‌گذاری‌شده‌ای مانند LLaVA، Pixtral، InternVL، Mllama، MiniCPM-V و GLM-4V؛ همچنین برای خانواده‌های شناخته‌شده صرفاً متنی (Llama، DeepSeek، Mistral/Mixtral، Kimi/Moonshot، Codestral، Devstral، Phi، QwQ، CodeLlama و شناسه‌های ساده Qwen بدون پسوند vl/vision) پرسش اضافی را نادیده می‌گیرد. برای شناسه‌های مدل ناشناخته همچنان درباره پشتیبانی از تصویر پرسیده می‌شود. راه‌اندازی غیرتعاملی نیز از همین استنباط استفاده می‌کند؛ برای اجبار فراداده دارای قابلیت تصویر، `--custom-image-input` و برای اجبار فراداده صرفاً متنی، `--custom-text-input` را ارسال کنید.

### نمونه‌های ارائه‌دهنده

<AccordionGroup>
  <Accordion title="Cerebras (GLM 4.7 / GPT OSS)">
    Plugin رسمی ارائه‌دهنده خارجی `cerebras` می‌تواند این مورد را از طریق `openclaw onboard --auth-choice cerebras-api-key` پیکربندی کند. فقط هنگام بازنویسی پیش‌فرض‌ها از پیکربندی صریح ارائه‌دهنده استفاده کنید.

    ```json5
    {
      env: { CEREBRAS_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: {
            primary: "cerebras/zai-glm-4.7",
            fallbacks: ["cerebras/gpt-oss-120b"],
          },
          models: {
            "cerebras/zai-glm-4.7": { alias: "GLM 4.7 (Cerebras)" },
            "cerebras/gpt-oss-120b": { alias: "GPT OSS 120B (Cerebras)" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          cerebras: {
            baseUrl: "https://api.cerebras.ai/v1",
            apiKey: "${CEREBRAS_API_KEY}",
            api: "openai-completions",
            models: [
              { id: "zai-glm-4.7", name: "GLM 4.7 (Cerebras)" },
              { id: "gpt-oss-120b", name: "GPT OSS 120B (Cerebras)" },
            ],
          },
        },
      },
    }
    ```

    برای Cerebras از `cerebras/zai-glm-4.7` و برای اتصال مستقیم به Z.AI از `zai/glm-4.7` استفاده کنید.

  </Accordion>
  <Accordion title="Kimi Coding">
    ```json5
    {
      env: { KIMI_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "kimi/kimi-for-coding" },
          models: { "kimi/kimi-for-coding": { alias: "Kimi Code" } },
        },
      },
    }
    ```

    ارائه‌دهنده داخلی سازگار با Anthropic. میان‌بر: `openclaw onboard --auth-choice kimi-code-api-key`.

  </Accordion>
  <Accordion title="مدل‌های محلی (LM Studio)">
    [مدل‌های محلی](/fa/gateway/local-models) را ببینید. خلاصه: یک مدل محلی بزرگ را از طریق Responses API در LM Studio و روی سخت‌افزار قدرتمند اجرا کنید؛ مدل‌های میزبانی‌شده را برای حالت جایگزین ادغام‌شده نگه دارید.
  </Accordion>
  <Accordion title="MiniMax M3 (مستقیم)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "minimax/MiniMax-M3" },
          models: {
            "minimax/MiniMax-M3": { alias: "Minimax" },
          },
        },
      },
      models: {
        mode: "merge",
        providers: {
          minimax: {
            baseUrl: "https://api.minimax.io/anthropic",
            apiKey: "${MINIMAX_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0.6, output: 2.4, cacheRead: 0.12, cacheWrite: 0 },
                contextWindow: 1000000,
                maxTokens: 131072,
              },
            ],
          },
        },
      },
    }
    ```

    `MINIMAX_API_KEY` را تنظیم کنید. میان‌برها: `openclaw onboard --auth-choice minimax-global-api` یا `openclaw onboard --auth-choice minimax-cn-api`. کاتالوگ مدل به‌طور پیش‌فرض از M3 استفاده می‌کند و گونه‌های M2.7 را نیز در بر می‌گیرد. در مسیر استریم سازگار با Anthropic، OpenClaw قابلیت تفکر MiniMax M2.x را به‌طور پیش‌فرض غیرفعال می‌کند، مگر اینکه خودتان `thinking` را صریحاً تنظیم کنید؛ MiniMax-M3 (و M3.x) به‌طور پیش‌فرض در مسیر تفکر حذف‌شده/تطبیقی ارائه‌دهنده باقی می‌ماند. `/fast on` یا `params.fastMode: true`، مقدار `MiniMax-M2.7` را به `MiniMax-M2.7-highspeed` بازنویسی می‌کند.

  </Accordion>
  <Accordion title="Moonshot AI (Kimi)">
    ```json5
    {
      env: { MOONSHOT_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "moonshot/kimi-k2.6" },
          models: { "moonshot/kimi-k2.6": { alias: "Kimi K2.6" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          moonshot: {
            baseUrl: "https://api.moonshot.ai/v1",
            apiKey: "${MOONSHOT_API_KEY}",
            api: "openai-completions",
            models: [
              {
                id: "kimi-k2.6",
                name: "Kimi K2.6",
                reasoning: false,
                input: ["text", "image"],
                cost: { input: 0.95, output: 4, cacheRead: 0.16, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 262144,
              },
            ],
          },
        },
      },
    }
    ```

    برای نقطه پایانی چین: `baseUrl: "https://api.moonshot.cn/v1"` یا `openclaw onboard --auth-choice moonshot-api-key-cn`.

    نقاط پایانی بومی Moonshot سازگاری مصرف در حالت استریم را روی انتقال مشترک `openai-completions` اعلام می‌کنند و OpenClaw این قابلیت را بر اساس ویژگی‌های نقطه پایانی، نه صرفاً شناسه ارائه‌دهنده داخلی، فعال می‌کند.

  </Accordion>
  <Accordion title="OpenCode">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "opencode/claude-opus-4-6" },
          models: { "opencode/claude-opus-4-6": { alias: "Opus" } },
        },
      },
    }
    ```

    `OPENCODE_API_KEY` (یا `OPENCODE_ZEN_API_KEY`) را تنظیم کنید. برای کاتالوگ Zen از ارجاع‌های `opencode/...` و برای کاتالوگ Go از ارجاع‌های `opencode-go/...` استفاده کنید. میان‌بر: `openclaw onboard --auth-choice opencode-zen` یا `openclaw onboard --auth-choice opencode-go`.

  </Accordion>
  <Accordion title="Synthetic (سازگار با Anthropic)">
    ```json5
    {
      env: { SYNTHETIC_API_KEY: "sk-..." },
      agents: {
        defaults: {
          model: { primary: "synthetic/hf:MiniMaxAI/MiniMax-M3" },
          models: { "synthetic/hf:MiniMaxAI/MiniMax-M3": { alias: "MiniMax M3" } },
        },
      },
      models: {
        mode: "merge",
        providers: {
          synthetic: {
            baseUrl: "https://api.synthetic.new/anthropic",
            apiKey: "${SYNTHETIC_API_KEY}",
            api: "anthropic-messages",
            models: [
              {
                id: "hf:MiniMaxAI/MiniMax-M3",
                name: "MiniMax M3",
                reasoning: true,
                input: ["text", "image"],
                cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
                contextWindow: 262144,
                maxTokens: 65536,
              },
            ],
          },
        },
      },
    }
    ```

    نشانی پایه باید فاقد `/v1` باشد (کلاینت Anthropic آن را اضافه می‌کند). میان‌بر: `openclaw onboard --auth-choice synthetic-api-key`.

  </Accordion>
  <Accordion title="Z.AI (GLM-4.7)">
    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "zai/glm-4.7" },
          models: { "zai/glm-4.7": {} },
        },
      },
    }
    ```

    `ZAI_API_KEY` را تنظیم کنید. ارجاع‌های مدل از شناسه متعارف ارائه‌دهنده `zai/*` استفاده می‌کنند. میان‌بر: `openclaw onboard --auth-choice zai-api-key`.

    - نقطه پایانی عمومی: `https://api.z.ai/api/paas/v4`
    - نقطه پایانی کدنویسی: `https://api.z.ai/api/coding/paas/v4`
    - گزینه احراز هویت پیش‌فرض `zai-api-key`، کلید شما را بررسی می‌کند و به‌طور خودکار تشخیص می‌دهد که به کدام نقطه پایانی تعلق دارد (اگر تشخیص قطعی نباشد، پرسشی نمایش می‌دهد که مقدار پیش‌فرض آن Global است). گزینه‌های اختصاصی احراز هویت CN و Coding-Plan نیز برای انتخاب صریح در دسترس‌اند.
    - برای نقطه پایانی عمومی، یک ارائه‌دهنده سفارشی با بازنویسی نشانی پایه تعریف کنید.

  </Accordion>
</AccordionGroup>

---

## مرتبط

- [پیکربندی — عامل‌ها](/fa/gateway/config-agents)
- [پیکربندی — کانال‌ها](/fa/gateway/config-channels)
- [مرجع پیکربندی](/fa/gateway/configuration-reference) — سایر کلیدهای سطح بالا
- [ابزارها و Pluginها](/fa/tools)
