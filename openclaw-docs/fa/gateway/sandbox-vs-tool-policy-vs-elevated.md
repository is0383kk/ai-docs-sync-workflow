---
read_when: You hit 'sandbox jail' or see a tool/elevated refusal and want the exact config key to change.
status: active
summary: 'چرا یک ابزار مسدود می‌شود: محیط اجرای sandbox، خط‌مشی مجاز/غیرمجاز بودن ابزار و گیت‌های اجرای دارای دسترسی ارتقایافته'
title: Sandbox در برابر خط‌مشی ابزار در برابر دسترسی ارتقایافته
x-i18n:
    generated_at: "2026-07-27T15:16:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c4da521215fe55bf2774008a53d896d5c00b8babcbca2005dc4593ebfebc5343
    source_path: gateway/sandbox-vs-tool-policy-vs-elevated.md
    workflow: 16
---

OpenClaw سه کنترل مرتبط اما متفاوت دارد:

1. **محیط ایزوله** (`agents.defaults.sandbox.*` / `agents.entries.*.sandbox.*`) تعیین می‌کند **ابزارها کجا اجرا شوند** (بک‌اند محیط ایزوله یا میزبان).
2. **سیاست ابزار** (`tools.*`، `tools.sandbox.tools.*`، `agents.entries.*.tools.*`) تعیین می‌کند **کدام ابزارها در دسترس/مجاز باشند**.
3. **سطح‌بالا** (`tools.elevated.*`، `agents.entries.*.tools.elevated.*`) یک **راه گریز مخصوص exec** برای اجرای خارج از محیط ایزوله در زمان حضور در محیط ایزوله است (`gateway` به‌طور پیش‌فرض، یا `node` هنگامی که مقصد exec روی `node` تنظیم شده باشد).

## اشکال‌زدایی سریع

برای مشاهده کاری که OpenClaw _واقعاً_ انجام می‌دهد، از بازرس استفاده کنید:

```bash
openclaw sandbox explain
openclaw sandbox explain --session agent:main:main
openclaw sandbox explain --agent work
openclaw sandbox explain --json
```

این موارد را نمایش می‌دهد:

- حالت/دامنه مؤثر محیط ایزوله و دسترسی به فضای کاری
- این‌که نشست در حال حاضر در محیط ایزوله است یا نه (اصلی در برابر غیراصلی)
- اجازه/رد مؤثر ابزارهای محیط ایزوله (و این‌که از عامل/سراسری/پیش‌فرض آمده است)
- دروازه‌های سطح‌بالا و مسیر کلیدهای اصلاح

## محیط ایزوله: ابزارها کجا اجرا می‌شوند

ایزوله‌سازی توسط `agents.defaults.sandbox.mode` کنترل می‌شود:

- `"off"`: همه‌چیز روی میزبان اجرا می‌شود.
- `"non-main"`: فقط نشست‌های غیراصلی ایزوله می‌شوند (یک «غافلگیری» رایج برای گروه‌ها/کانال‌ها).
- `"all"`: همه‌چیز ایزوله می‌شود.

`agents.defaults.sandbox.workspaceAccess` آنچه را محیط ایزوله می‌تواند ببیند کنترل می‌کند: `"none"`، `"ro"` یا `"rw"`.

برای ماتریس کامل (دامنه، سوارکردن فضای کاری، ایمیج‌ها)، به [ایزوله‌سازی](/fa/gateway/sandboxing) مراجعه کنید.

### سوارکردن‌های bind (بررسی سریع امنیتی)

- `docker.binds` سامانه فایل محیط ایزوله را _سوراخ می‌کند_: هرآنچه سوار کنید، با حالتی که تنظیم کرده‌اید (`:ro` یا `:rw`) درون کانتینر قابل‌مشاهده است.
- اگر حالت را حذف کنید، پیش‌فرض خواندن‌ونوشتن است؛ برای کد منبع/اطلاعات محرمانه، `:ro` را ترجیح دهید.
- `scope: "shared"`، bindهای مختص عامل را نادیده می‌گیرد (فقط bindهای سراسری اعمال می‌شوند).
- OpenClaw منابع bind را دو بار اعتبارسنجی می‌کند: ابتدا روی مسیر مبدأ نرمال‌شده و سپس بار دیگر پس از رفع مسیر از طریق عمیق‌ترین جد موجود. گریز از والد پیوند نمادین نمی‌تواند بررسی مسیر مسدودشده یا ریشه مجاز را دور بزند.
- مسیرهای برگِ ناموجود نیز به‌طور ایمن بررسی می‌شوند. اگر `/workspace/alias-out/new-file` از طریق یک والد پیوند نمادین به مسیری مسدودشده یا خارج از ریشه‌های مجاز پیکربندی‌شده رفع شود، bind رد می‌شود.
- متصل‌کردن `/var/run/docker.sock` عملاً کنترل میزبان را به محیط ایزوله می‌سپارد؛ این کار را فقط آگاهانه انجام دهید.
- دسترسی فضای کاری (`workspaceAccess`) مستقل از حالت‌های bind است.

برای پیکربندی مختص عامل با چند پوشه میزبان، حالت‌های دسترسی و اعلام پذیرش ایمنی منبع خارجی، به [چند پوشه برای یک عامل](/fa/gateway/sandboxing#multiple-folders-for-one-agent) مراجعه کنید.

## سیاست ابزار: کدام ابزارها وجود دارند/قابل‌فراخوانی هستند

دو لایه اهمیت دارند:

- **نمایه ابزار**: `tools.profile` و `agents.entries.*.tools.profile` (فهرست مجاز پایه)
- **نمایه ابزار ارائه‌دهنده**: `tools.byProvider[provider].profile` و `agents.entries.*.tools.byProvider[provider].profile`
- **سیاست ابزار سراسری/مختص عامل**: `tools.allow`/`tools.deny` و `agents.entries.*.tools.allow`/`agents.entries.*.tools.deny`
- **سیاست ابزار ارائه‌دهنده**: `tools.byProvider[provider].allow/deny` و `agents.entries.*.tools.byProvider[provider].allow/deny`
- **سیاست ابزار محیط ایزوله** (فقط هنگام ایزوله‌بودن اعمال می‌شود): `tools.sandbox.tools.allow`/`tools.sandbox.tools.deny` و `agents.entries.*.tools.sandbox.tools.*`

قواعد کلی:

- `deny` همیشه اولویت دارد.
- اگر `allow` خالی نباشد، هر چیز دیگری مسدود تلقی می‌شود.
- سیاست ابزار توقف قطعی است: `/exec` نمی‌تواند ابزار `exec` ردشده را نادیده بگیرد.
- سیاست ابزار، دسترس‌پذیری ابزار را بر اساس نام فیلتر می‌کند؛ اثرات جانبی درون `exec` را بررسی نمی‌کند. اگر `exec` مجاز باشد، ردکردن `write`، `edit` یا `apply_patch` فرمان‌های پوسته را فقط‌خواندنی نمی‌کند.
- `/exec` فقط پیش‌فرض‌های نشست را برای فرستندگان مجاز تغییر می‌دهد؛ دسترسی به ابزار اعطا نمی‌کند.
- کلیدهای ابزار ارائه‌دهنده، هم `provider` (برای مثال `google-antigravity`) و هم `provider/model` (برای مثال `openai/gpt-5.4`) را می‌پذیرند.
- گزارش‌های Gateway هنگامی که یک مرحله سیاست ابزار، ابزارها را حذف کند یا سیاست ابزار محیط ایزوله فراخوانی‌ای را مسدود کند، ورودی‌های ممیزی `agents/tool-policy` را شامل می‌شوند. برای مشاهده برچسب قاعده، کلید پیکربندی و نام ابزارهای متأثر، از `openclaw logs` استفاده کنید.

### گروه‌های ابزار (صورت‌های کوتاه)

سیاست‌های ابزار (سراسری، عامل، محیط ایزوله) از ورودی‌های `group:*` پشتیبانی می‌کنند که به چند ابزار گسترش می‌یابند:

```json5
{
  tools: {
    sandbox: {
      tools: {
        allow: ["group:runtime", "group:fs", "group:sessions", "group:memory"],
      },
    },
  },
}
```

گروه‌های موجود:

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
| `group:openclaw`   | بیشتر ابزارهای داخلی OpenClaw (شامل اجزای اولیه سامانه فایل و زمان اجرای `read`/`write`/`edit`/`apply_patch`/`exec`/`process`، همچنین `canvas` و Pluginهای ارائه‌دهنده نمی‌شود)                                                                                             |
| `group:plugins`    | همه ابزارهای بارگذاری‌شده متعلق به Plugin، از جمله سرورهای MCP پیکربندی‌شده‌ای که از طریق `bundle-mcp` ارائه می‌شوند                                                                                                                                                           |

برای عامل‌های فقط‌خواندنی، علاوه بر ابزارهای تغییردهنده سامانه فایل، `group:runtime` را نیز رد کنید؛ مگر این‌که سیاست سامانه فایل محیط ایزوله یا یک مرز جداگانه میزبان، محدودیت فقط‌خواندنی را اعمال کند.

برای سرورهای MCP ایزوله‌شده، سیاست ابزار محیط ایزوله دومین دروازه اجازه است. اگر `mcp.servers` پیکربندی شده است، اما نوبت‌های ایزوله‌شده فقط ابزارهای داخلی را نشان می‌دهند، `bundle-mcp`، `group:plugins` یا یک نام/الگوی کلی ابزار MCP با پیشوند سرور مانند `outlook__send_mail` یا `outlook__*` را به `tools.sandbox.tools.alsoAllow` اضافه کنید؛ سپس Gateway را بازراه‌اندازی/بارگذاری مجدد کنید و فهرست ابزارها را دوباره ثبت کنید. الگوهای کلی سرور از پیشوند سرور MCP ایمن برای ارائه‌دهنده استفاده می‌کنند: نویسه‌های غیر `[A-Za-z0-9_-]` به `-` تبدیل می‌شوند، نام‌هایی که با حرف آغاز نمی‌شوند پیشوند `mcp-` می‌گیرند و پیشوندهای طولانی یا تکراری ممکن است کوتاه شوند یا پسوند بگیرند.

`openclaw doctor` در حال حاضر این ساختار را برای سرورهای مدیریت‌شده توسط OpenClaw در `mcp.servers` بررسی می‌کند. سرورهای MCP بارگذاری‌شده از مانیفست Pluginهای همراه یا `.mcp.json` متعلق به Claude از همان دروازه محیط ایزوله استفاده می‌کنند، اما این ابزار تشخیصی هنوز آن منابع را فهرست نمی‌کند؛ اگر ابزارهای آن‌ها در نوبت‌های ایزوله‌شده ناپدید شدند، از همان ورودی‌های فهرست مجاز استفاده کنید.

## سطح‌بالا: «اجرا روی میزبان» فقط برای exec

سطح‌بالا ابزارهای بیشتری اعطا **نمی‌کند**؛ فقط بر `exec` اثر می‌گذارد.

- اگر در محیط ایزوله هستید، `/elevated on` (یا `exec` با `elevated: true`) خارج از محیط ایزوله اجرا می‌شود (ممکن است تأییدها همچنان اعمال شوند).
- برای ردکردن تأییدهای exec در نشست، از `/elevated full` استفاده کنید.
- اگر از قبل به‌صورت مستقیم اجرا می‌شوید، سطح‌بالا عملاً بی‌اثر است (اما همچنان مشمول دروازه‌هاست).
- سطح‌بالا محدود به Skill نیست و اجازه/رد ابزار را **نادیده نمی‌گیرد**.
- سطح‌بالا، بازنویسی‌های دلخواه میان‌میزبانی از `host=auto` را اعطا نمی‌کند؛ از قواعد عادی مقصد exec پیروی می‌کند و فقط زمانی `node` را حفظ می‌کند که مقصد پیکربندی‌شده/نشست از قبل `node` باشد.
- `/exec` از سطح‌بالا جداست. این فقط پیش‌فرض‌های exec مختص نشست را برای فرستندگان مجاز تنظیم می‌کند.

دروازه‌ها:

- فعال‌سازی: `tools.elevated.enabled` (و در صورت نیاز `agents.entries.*.tools.elevated.enabled`)
- فهرست‌های مجاز فرستنده: `tools.elevated.allowFrom.<provider>` (و در صورت نیاز `agents.entries.*.tools.elevated.allowFrom.<provider>`)

به [حالت سطح‌بالا](/fa/tools/elevated) مراجعه کنید.

## رفع مشکلات رایج «حبس در محیط ایزوله»

### «ابزار X توسط سیاست ابزار محیط ایزوله مسدود شده است»

کلیدهای اصلاح (یکی را انتخاب کنید):

- غیرفعال‌کردن سندباکس: `agents.defaults.sandbox.mode=off` (یا برای هر عامل `agents.entries.*.sandbox.mode=off`)
- اجازه‌دادن به ابزار درون سندباکس:
  - آن را از `tools.sandbox.tools.deny` حذف کنید (یا برای هر عامل از `agents.entries.*.tools.sandbox.tools.deny`)
  - یا آن را به `tools.sandbox.tools.allow` اضافه کنید (یا مجوز برای هر عامل)
- ورودی `agents/tool-policy` را در `openclaw logs` بررسی کنید. این ورودی حالت سندباکس و اینکه آیا قاعدهٔ مجاز یا ممنوع‌سازی ابزار را مسدود کرده است ثبت می‌کند.

### «فکر می‌کردم این main است، چرا در سندباکس قرار دارد؟»

در حالت `"non-main"`، کلیدهای گروه/کانال main _نیستند_. از کلید نشست main (که `sandbox explain` نشان می‌دهد) استفاده کنید یا حالت را به `"off"` تغییر دهید.

## مرتبط

- [سندباکس‌سازی](/fa/gateway/sandboxing) -- مرجع کامل سندباکس (حالت‌ها، دامنه‌ها، بک‌اندها، ایمیج‌ها)
- [سندباکس و ابزارهای چندعاملی](/fa/tools/multi-agent-sandbox-tools) -- بازنویسی‌های مختص هر عامل و تقدم آن‌ها
- [حالت ارتقایافته](/fa/tools/elevated)
