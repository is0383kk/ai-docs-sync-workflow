---
read_when:
    - جفت‌سازی Nodeهای iOS/watchOS/Android با یک Gateway
    - استفاده از بوم/دوربین Node برای زمینهٔ عامل
    - افزودن فرمان‌های جدید Node یا ابزارهای کمکی CLI
summary: 'Nodeها: جفت‌سازی، قابلیت‌ها، مجوزها و ابزارهای کمکی CLI برای بوم/دوربین/صفحه‌نمایش/دستگاه/اعلان‌ها/سیستم'
title: Nodeها
x-i18n:
    generated_at: "2026-07-27T15:41:06Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: b4f7c80491d713777e1ba5b8f55c88bd9fa48be602b504e6ac6ba00cd12a4313
    source_path: nodes/index.md
    workflow: 16
---

یک **node** دستگاهی همراه (macOS/iOS/watchOS/Android/بدون رابط گرافیکی) است که با `role: "node"` به Gateway متصل می‌شود و از طریق `node.invoke` یک سطح فرمان (برای مثال `canvas.*`، `camera.*`، `device.*`، `notifications.*`، `system.*`) ارائه می‌کند. بیشتر nodeها از WebSocket مربوط به Gateway روی پورت اپراتور استفاده می‌کنند. node مستقیم و اختیاری Apple Watch از نظرسنجی HTTPS امضاشده روی همان پورت استفاده می‌کند، زیرا watchOS شبکه‌سازی عمومی سطح پایین را برای برنامه‌های عادی مسدود می‌کند. جزئیات پروتکل: [پروتکل Gateway](/fa/gateway/protocol).

انتقال قدیمی: [پروتکل Bridge](/fa/gateway/bridge-protocol) (TCP JSONL؛ فقط برای سابقه تاریخی nodeهای فعلی).

macOS همچنین می‌تواند در **حالت node** اجرا شود: برنامه نوار منو به سرور
WS مربوط به Gateway به‌عنوان یک node متصل می‌شود (بنابراین `openclaw nodes …` روی این Mac کار می‌کند). برنامه
فرمان‌های بومی Canvas، دوربین، صفحه‌نمایش، اعلان و کنترل رایانه را
به همان سطح فرمان میزبان node که `openclaw node run` استفاده می‌کند اضافه می‌کند. یک node دوم
از نوع CLI را روی آن Mac اجرا نکنید؛ برنامه، محیط اجرای متناظر میزبان node در CLI را
به‌عنوان worker داخلی اجرا می‌کند و تنها اتصال Gateway و هویت node باقی می‌ماند.

Nodeها **دستگاه‌های جانبی** هستند، نه Gateway: آن‌ها سرویس Gateway را اجرا نمی‌کنند و پیام‌های کانال (Telegram، WhatsApp و غیره) به Gateway می‌رسند، نه به nodeها.

راهنمای رفع اشکال: [/nodes/troubleshooting](/fa/nodes/troubleshooting)

## جفت‌سازی + وضعیت

Nodeها از **جفت‌سازی دستگاه** استفاده می‌کنند. یک node هنگام اتصال، هویت امضاشده دستگاه را ارائه می‌دهد؛ Gateway برای `role: node` یک درخواست جفت‌سازی دستگاه ایجاد می‌کند. آن را از طریق CLI دستگاه‌ها (یا رابط کاربری) تأیید کنید. راه‌اندازی مستقیم Apple Watch برای تأیید سطح فرمان ثابت و کم‌خطر خود از یک کد راه‌اندازی کوتاه‌عمر و مختص node که مدیر ایجاد کرده است استفاده می‌کند؛ گسترش قابلیت‌ها در آینده همچنان به تأیید عادی نیاز دارد.

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw devices reject <requestId>
openclaw nodes status
openclaw nodes describe --node <idOrNameOrIp>
```

درخواست‌های جفت‌سازی در انتظار، 5 دقیقه پس از آخرین تلاش مجدد دستگاه منقضی می‌شوند — دستگاهی که پیوسته دوباره متصل می‌شود، به‌جای ایجاد یک درخواست جدید هر چند دقیقه، همان یک درخواست در انتظار (و `requestId`) را فعال نگه می‌دارد؛ برای چرخه کامل درخواست/تأیید، [جفت‌سازی Node](/fa/gateway/pairing) را ببینید. اگر یک node با جزئیات احراز هویت تغییریافته (نقش/دامنه‌ها/کلید عمومی) دوباره تلاش کند، درخواست در انتظار قبلی جایگزین و یک `requestId` جدید ایجاد می‌شود — کلاینت‌ها برای درخواست جایگزین‌شده یک رویداد `device.pair.resolved` دریافت می‌کنند و باید پیش از تأیید، `openclaw devices list` را دوباره اجرا کنید.

- `nodes status` هنگامی یک node را **جفت‌شده** علامت‌گذاری می‌کند که نقش جفت‌سازی دستگاه آن شامل `node` باشد.
- یک Mac بومی متصل می‌تواند در
  **Settings -> Permissions -> Active computer detection** تجمیع فعالیت ورودی فیزیکی را فعال کند. دسترسی‌پذیری
  نیز الزامی است. Gateway جدیدترین Mac واجد شرایط را به‌عنوان
  `active` علامت‌گذاری می‌کند، یک راهنمای پایدار شناسه node به عامل می‌دهد و هشدارهای اتصال node را
  پیش از بازگشت تأخیردار به آن هدایت می‌کند. برای راه‌اندازی، حریم خصوصی، زمان‌بندی و
  رفع اشکال، [حضور رایانه فعال](/fa/nodes/presence) را ببینید.
- رکورد جفت‌سازی دستگاه، قرارداد پایدار نقش تأییدشده است. چرخش توکن درون همان قرارداد باقی می‌ماند؛ این کار نمی‌تواند یک node جفت‌شده را به نقشی ارتقا دهد که تأیید جفت‌سازی هرگز اعطا نکرده است.
- `node.pair.*` (CLI: `openclaw nodes pending/approve/reject/remove/rename`) یک مخزن جفت‌سازی node جداگانه و تحت مالکیت Gateway است که سطح فرمان/قابلیت تأییدشده node را در اتصال‌های مجدد ردیابی می‌کند. این مخزن احراز هویت انتقال را کنترل **نمی‌کند** — جفت‌سازی دستگاه این کار را انجام می‌دهد.
- `openclaw nodes remove --node <id|name|ip>` جفت‌سازی یک node را حذف می‌کند. برای یک node مبتنی بر دستگاه، نقش `node` دستگاه را در مخزن دستگاه‌های جفت‌شده لغو و نشست‌های دارای نقش node آن دستگاه را قطع می‌کند: دستگاهی با چند نقش، ردیف خود را حفظ می‌کند و فقط نقش `node` را از دست می‌دهد، درحالی‌که ردیف دستگاهی که فقط node است حذف می‌شود. همچنین هر ورودی منطبق را از مخزن جفت‌سازی جداگانه node پاک می‌کند. `operator.pairing` ممکن است ردیف‌های node غیر اپراتور را روی دستگاه‌های دیگر حذف کند؛ فراخواننده دارای توکن دستگاه که نقش node خود را روی دستگاهی با چند نقش لغو می‌کند، علاوه بر این به `operator.admin` نیاز دارد.
- دامنه تأیید از فرمان‌های اعلام‌شده درخواست در انتظار پیروی می‌کند:
  - درخواست بدون فرمان: `operator.pairing`
  - فرمان‌های غیر اجرایی node: `operator.pairing` + `operator.write`
  - `system.run` / `system.run.prepare` / `system.which`: `operator.pairing` + `operator.admin`

## اختلاف نسخه و ترتیب ارتقا

WebSocket مربوط به Gateway، کلاینت‌های node احراز هویت‌شده را در یک بازه پروتکلی N-1 می‌پذیرد.
بنابراین Gateway فعلی v4، هنگامی nodeهای v3 را می‌پذیرد که اتصال
هر دو `role: "node"` و `client.mode: "node"` را اعلام کند. نشست‌های اپراتور و رابط کاربری
همچنان باید از پروتکل فعلی استفاده کنند.

برای ارتقای مرحله‌ای ناوگان، ابتدا Gateway و سپس هر node را ارتقا دهید.
یک node با نسخه N-1 هنگام ارتقا همچنان قابل مشاهده و مدیریت باقی می‌ماند؛ Gateway
همراه با توصیه ارتقا، `legacy node protocol accepted` را ثبت می‌کند. جفت‌سازی،
احراز هویت دستگاه، فهرست‌های مجاز فرمان و تأییدهای اجرا همچنان اعمال می‌شوند.
قابلیت‌ها و فرمان‌های تحت مالکیت Plugin تا زمانی که node به
پروتکل فعلی ارتقا یابد پنهان می‌مانند. Nodeهای قدیمی‌تر از N-1 پیش از
اتصال مجدد به ارتقای خارج از باند نیاز دارند.

انتقال مستقیم HTTPS در watchOS به نسخه فعلی پروتکل نیاز دارد؛ پیش از فعال‌سازی
حالت مستقیم، برنامه ساعت را همراه Gateway به‌روزرسانی کنید.

## میزبان node راه‌دور (system.run)

هنگامی از **میزبان node** استفاده کنید که Gateway روی یک دستگاه اجرا می‌شود و می‌خواهید فرمان‌ها روی دستگاه دیگری اجرا شوند. مدل همچنان با **Gateway** ارتباط برقرار می‌کند؛ هنگامی که `host=node` انتخاب شده باشد، Gateway فراخوانی‌های `exec` را به **میزبان node** ارسال می‌کند.

| نقش         | مسئولیت                                                   |
| ------------ | ---------------------------------------------------------------- |
| میزبان Gateway | پیام‌ها را دریافت می‌کند، مدل را اجرا می‌کند و فراخوانی‌های ابزار را هدایت می‌کند.            |
| میزبان Node    | `system.run`/`system.which` را روی دستگاه node اجرا می‌کند.        |
| تأییدها    | از طریق `~/.openclaw/exec-approvals.json` روی میزبان node اعمال می‌شوند. |

نکته تأیید:

- اجراهای node مبتنی بر تأیید، زمینه دقیق درخواست را مقید می‌کنند. مسیر اجرا پیش از تأیید یک `systemRunPlan` معیار آماده می‌کند؛ پس از اعطای تأیید، Gateway همان طرح ذخیره‌شده را ارسال می‌کند، نه هیچ‌یک از فیلدهای فرمان/cwd/نشست که فراخواننده بعداً ویرایش کرده باشد، و پیش از اجرا دایرکتوری کاری را دوباره اعتبارسنجی می‌کند.
- برای اجرای مستقیم فایل‌های پوسته/محیط اجرا، OpenClaw همچنین در حد توان یک عملوند مشخص فایل محلی را مقید می‌کند و اگر آن فایل پیش از اجرا تغییر کند، اجرا را رد می‌کند.
- اگر OpenClaw نتواند دقیقاً یک فایل محلی مشخص را برای فرمان مفسر/محیط اجرا شناسایی کند، اجرای مبتنی بر تأیید به‌جای وانمود کردن پوشش کامل محیط اجرا رد می‌شود. برای معناشناسی گسترده‌تر مفسر از sandbox، میزبان‌های جداگانه یا یک فهرست مجاز صریح و قابل‌اعتماد/گردش‌کار کامل استفاده کنید.

### راه‌اندازی میزبان node (پیش‌زمینه)

روی دستگاه node:

```bash
openclaw node run --host <gateway-host> --port 18789 --display-name "Build Node"
```

`node run` همچنین `--context-path` (مسیر زمینه WS مربوط به Gateway)، `--tls`، `--tls-fingerprint <sha256>` و `--node-id` (بازنویسی شناسه قدیمی نمونه کلاینت؛ این کار جفت‌سازی را بازنشانی نمی‌کند) را می‌پذیرد. در macOS برای اعلام `device.apps`، `--share-installed-apps` را ارسال کنید؛ اشتراک‌گذاری به‌طور پیش‌فرض غیرفعال است. برای غیرفعال کردن انتخاب ذخیره‌شده قبلی از `--no-share-installed-apps` استفاده کنید.

### Gateway راه‌دور از طریق تونل SSH (اتصال loopback)

اگر Gateway به loopback متصل شود (`gateway.bind=loopback`، پیش‌فرض در حالت محلی)، میزبان‌های node راه‌دور نمی‌توانند مستقیماً متصل شوند. یک تونل SSH ایجاد کنید و میزبان node را به انتهای محلی تونل هدایت کنید.

نمونه (میزبان node -> میزبان Gateway):

```bash
# ترمینال A (در حال اجرا نگه دارید): هدایت 18790 محلی -> Gateway 127.0.0.1:18789
ssh -N -L 18790:127.0.0.1:18789 user@gateway-host

# ترمینال B: توکن Gateway را صادر کنید و از طریق تونل متصل شوید
export OPENCLAW_GATEWAY_TOKEN="<gateway-token>"
openclaw node run --host 127.0.0.1 --port 18790 --display-name "Build Node"
```

نکات:

- `openclaw node run` از احراز هویت با توکن یا رمز عبور پشتیبانی می‌کند.
- متغیرهای محیطی ترجیح داده می‌شوند: `OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`.
- تنظیمات جایگزین عبارت‌اند از `gateway.auth.token` / `gateway.auth.password`.
- در حالت محلی، میزبان node عمداً `gateway.remote.token` / `gateway.remote.password` را نادیده می‌گیرد.
- در حالت راه‌دور، `gateway.remote.token` / `gateway.remote.password` طبق قواعد تقدم راه‌دور واجد شرایط هستند.
- اگر SecretRefهای فعال محلی `gateway.auth.*` پیکربندی شده اما حل‌نشده باشند، احراز هویت میزبان node به‌صورت بسته شکست می‌خورد.
- حل احراز هویت میزبان node فقط متغیرهای محیطی `OPENCLAW_GATEWAY_*` را می‌پذیرد.

### راه‌اندازی میزبان node (سرویس)

```bash
openclaw node install --host <gateway-host> --port 18789 --display-name "Build Node"
openclaw node start
openclaw node restart
```

`node install` همچنین `--context-path`، `--tls`، `--tls-fingerprint`، `--node-id` (فقط شناسه قدیمی نمونه کلاینت)، `--share-installed-apps` / `--no-share-installed-apps`، `--runtime <node>` (پیش‌فرض: node) و `--force` برای نصب مجدد را می‌پذیرد. `node status`، `node stop` و `node uninstall` نیز در دسترس هستند.

### جفت‌سازی + نام‌گذاری

روی میزبان Gateway:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

اگر node با جزئیات احراز هویت تغییریافته دوباره تلاش کرد، `openclaw devices list` را دوباره اجرا و `requestId` فعلی را تأیید کنید.

گزینه‌های نام‌گذاری:

- `--display-name` روی `openclaw node run` / `openclaw node install` (در ردیف مشترک SQLite مربوط به `node_host_config`، کنار شناسه نمونه کلاینت و فراداده اتصال Gateway نگهداری می‌شود).
- `openclaw nodes rename --node <id|name|ip> --name "Build Node"` (بازنویسی Gateway).

### سرورهای MCP میزبانی‌شده روی node

سرورهای MCP را در `openclaw.json` روی دستگاه node پیکربندی کنید، نه روی
Gateway:

```json5
{
  nodeHost: {
    mcp: {
      servers: {
        localDocs: {
          command: "npx",
          args: ["-y", "@modelcontextprotocol/server-filesystem", "/srv/docs"],
          toolFilter: {
            include: ["read_*", "search"],
          },
        },
        internalApi: {
          url: "https://mcp.internal.example/mcp",
          transport: "streamable-http",
          headers: {
            Authorization: "Bearer ${INTERNAL_MCP_TOKEN}",
          },
        },
      },
    },
  },
}
```

میزبان node بدون رابط گرافیکی این سرورها را راه‌اندازی می‌کند، ابزارهایشان را فهرست می‌کند و
پس از اتصال، توصیفگرها را منتشر می‌کند. فراخوانی‌های ابزار از طریق
`mcp.tools.call.v1` به همان node بازمی‌گردند؛ Gateway به پیکربندی منطبق MCP یا یک
Plugin جاوااسکریپت نیاز ندارد. سرورهای OAuth MCP در این مسیر v1 میزبانی‌شده روی node پشتیبانی نمی‌شوند.

میزبان‌های فعلی node در جفت‌سازی اولیه خود، خانواده فرمان داخلی `mcp.tools.call.v1` را حتی
هنگامی که هیچ سرور MCP پیکربندی نشده باشد اعلام می‌کنند. یک node که روی نسخه قدیمی‌تر
OpenClaw جفت شده است، ممکن است پس از به‌روزرسانی میزبان node یک ارتقای یک‌باره سطح فرمان درخواست کند.
افزودن، حذف یا فیلتر کردن سرورها پس از آن نیازی به جفت‌سازی مجدد ندارد،
زیرا خانواده فرمان تأییدشده بدون تغییر باقی می‌ماند. برای اعمال تغییرات پیکربندی MCP مربوط به node،
`openclaw node run` یا `openclaw node restart` را دوباره راه‌اندازی کنید؛
میزبان node این پیکربندی را پایش نمی‌کند.

اپراتورهای Gateway می‌توانند همه ابزارهای قابل‌مشاهده برای عامل را که nodeهای جفت‌شده منتشر می‌کنند،
از جمله ابزارهای MCP میزبانی‌شده روی node، با
`gateway.nodes.pluginTools.enabled: false` نادیده بگیرند. منع دقیق فرمان، مانند
`gateway.nodes.commands.deny: ["mcp.tools.call.v1"]`، نیز اجرا را مسدود می‌کند.

### Skills میزبانی‌شده روی node

مهارت‌ها را در پوشهٔ فعال مهارت‌های OpenClaw روی ماشین Node نصب کنید که به‌طور پیش‌فرض
`~/.openclaw/skills` است. `OPENCLAW_HOME`، `OPENCLAW_STATE_DIR` و
`OPENCLAW_CONFIG_PATH` این پروفایل فعال را جابه‌جا می‌کنند. برای مهارت‌ها، `OPENCLAW_STATE_DIR`
اولویت دارد؛ در غیر این صورت، `skills/` در کنار مسیری قرار دارد که
`openclaw config file` چاپ می‌کند. میزبان Node بدون رابط، پس از اتصال، فایل‌های معتبر
`SKILL.md` را منتشر می‌کند و Gateway فقط تا زمانی که آن Node متصل بماند،
آن‌ها را به اسنپ‌شات‌های مهارت عامل اضافه می‌کند. نام هر پوشهٔ مهارت باید با فیلد
frontmatter یعنی `name` مطابقت داشته باشد تا مکان‌یاب انتزاعی Node،
بدون افزودن فیلد پروتکل دیگری، به یک ورودی نگاشت شود.

جفت‌سازی اولیهٔ نقش Node، انتشار مهارت را تأیید می‌کند. افزودن، حذف یا
تغییر مهارت‌ها به جفت‌سازی مجدد یا تغییر پیکربندی Gateway نیاز ندارد.
پس از تغییر فایل‌های مهارت Node، `openclaw node run` یا `openclaw node restart`
را راه‌اندازی مجدد کنید؛ میزبان Node پوشهٔ مهارت‌ها را پایش نمی‌کند.

ورودی‌های مهارت میزبانی‌شده روی Node، Node خود را مشخص می‌کنند و محل اجرای
خود را همراه دارند. فایل‌های مهارت، مسیرهای نسبی ارجاع‌شده و فایل‌های اجرایی
روی همان Node باقی می‌مانند. عامل، محل اعلام‌شدهٔ `node://.../SKILL.md` را با
ابزار معمول `read` می‌خواند. `file_fetch` مسیرهای مطلق Node
را که اپراتور تأیید کرده است می‌پذیرد، نه مکان‌یاب‌های مهارت Node؛ زمان‌های اجرایی
فاقد ابزار معمول خواندن می‌توانند در عوض `cat SKILL.md` را از طریق
`exec host=node node=<node-id>` و با پوشهٔ اعلام‌شدهٔ `node://.../skills/<name>` به‌عنوان
`workdir` اجرا کنند. فایل‌ها و فایل‌های اجرایی ارجاع‌شده از همان مقصد
اجرا و پوشهٔ کاری استفاده می‌کنند. میزبان Node آن مکان‌یاب را نسبت به پوشهٔ
فعال وضعیت OpenClaw خود حل می‌کند؛ بنابراین مسیرهای نسبی روی Node حل می‌شوند،
نه روی ماشین Gateway. Node منتشرکننده باید `system.run` تأییدشده داشته
باشد و سیاست اجرای عامل باید `host=node` را مجاز بداند؛ در غیر این صورت،
مهارت وارد اسنپ‌شات آن عامل نمی‌شود.

برای توقف انتشار، `nodeHost.skills.enabled: false` را روی Node تنظیم کنید. اپراتورهای Gateway
می‌توانند با `gateway.nodes.allowSkills: false` مهارت‌های همهٔ Nodeهای جفت‌شده را نادیده بگیرند.

### وضعیت هویت بدون رابط

Node بدون رابط سه رکورد وضعیت جداگانه را در SQLite مشترک نگه می‌دارد:

- `~/.openclaw/state/openclaw.sqlite` ‏(`node_host_config`): شناسهٔ نمونهٔ کلاینت، نام نمایشی و فرادادهٔ اتصال Gateway.
- `~/.openclaw/state/openclaw.sqlite` ‏(`device_identities`، کلید `primary`): جفت‌کلید امضاشدهٔ دستگاه و شناسهٔ رمزنگاری‌شدهٔ دستگاه که از آن مشتق شده است.
- `~/.openclaw/state/openclaw.sqlite` ‏(`device_auth_tokens`): توکن‌های احراز هویت دستگاه جفت‌شده که بر اساس شناسهٔ رمزنگاری‌شدهٔ دستگاه و نقش کلیدگذاری شده‌اند.

برای یک Node امضاشده، Gateway از شناسهٔ رمزنگاری‌شدهٔ دستگاه برای جفت‌سازی و
مسیریابی Node استفاده می‌کند. شناسهٔ نمونهٔ کلاینت فقط فرادادهٔ اتصال است.
بنابراین تغییر `--node-id` یا مهاجرت یک `node.json` بازنشسته،
جفت‌سازی را بازنشانی نمی‌کند. برای روند پشتیبانی‌شدهٔ لغو و جفت‌سازی مجدد و
یادداشت‌های ارتقا، به [وضعیت هویت و جفت‌سازی](/fa/cli/node#identity-and-pairing-state)
مراجعه کنید.

فایل‌های بازنشستهٔ `identity/device.json` و `identity/device-auth.json` ورودی‌های مهاجرت
تحت مالکیت Doctor هستند. میزبان Node را متوقف و `openclaw doctor --fix` را اجرا کنید؛
Doctor پیش از حذف فایل‌های قدیمی، ردیف‌های آن‌ها را در SQLite وارد و تأیید می‌کند.

### افزودن فرمان‌ها به فهرست مجاز

تأییدهای اجرا **برای هر میزبان Node جداگانه** هستند. ورودی‌های فهرست مجاز را از Gateway اضافه کنید:

```bash
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/uname"
openclaw approvals allowlist add --node <id|name|ip> "/usr/bin/sw_vers"
```

تأییدها در `~/.openclaw/exec-approvals.json` روی میزبان Node نگه‌داری می‌شوند.

### هدایت اجرا به Node

پیش‌فرض‌ها را پیکربندی کنید (پیکربندی Gateway):

```bash
openclaw config set tools.exec.host node
openclaw config set tools.exec.mode allowlist
openclaw config set tools.exec.node "<id-or-name>"
```

یا برای هر نشست:

```text
/exec host=node security=allowlist node=<id-or-name>
```

پس از تنظیم، هر فراخوانی `exec` با `host=node` روی میزبان Node اجرا می‌شود (مشروط به فهرست مجاز/تأییدهای Node).

`host=auto` به‌طور ضمنی و خودکار Node را انتخاب نمی‌کند، اما درخواست صریح
`host=node` برای هر فراخوانی از `auto` مجاز است. اگر می‌خواهید
اجرای Node پیش‌فرض نشست باشد، `tools.exec.host=node` یا `/exec host=node ...` را صریحاً تنظیم کنید.

مرتبط:

- [CLI میزبان Node](/fa/cli/node)
- [ابزار اجرا](/fa/tools/exec)
- [تأییدهای اجرا](/fa/tools/exec-approvals)

### استنتاج مدل محلی

یک Node دسکتاپ یا سرور می‌تواند مدل‌های دارای قابلیت گفتگو را از یک سرور Ollama
که روی همان Node اجرا می‌شود ارائه کند. عامل‌ها برای کشف مدل‌های نصب‌شده و اجرای
راه‌دور یک پرامپت محدود، از ابزار `node_inference` در Plugin مربوط به Ollama
استفاده می‌کنند؛ Gateway به دسترسی مستقیم شبکه‌ای به Ollama نیاز ندارد. برای
راه‌اندازی، پالایش مدل و فرمان‌های تأیید مستقیم، به
[استنتاج محلی Ollama روی Node](/fa/providers/ollama#node-local-inference) مراجعه کنید.

### نشست‌ها و رونوشت‌های Codex

Plugin رسمی `codex` می‌تواند نشست‌های بایگانی‌نشدهٔ Codex را روی یک
میزبان Node بدون رابط یا Node بومی macOS ارائه کند. ثبت کاتالوگ دیگر به
`supervision.enabled` وابسته نیست؛ آن گزینه ابزارهای نظارتی در دسترس عامل را کنترل
می‌کند. برای غیرفعال‌کردن کاتالوگ اپراتور و فرمان‌های کاتالوگ Node جفت‌شده، بدون
غیرفعال‌کردن ارائه‌دهنده یا چارچوب اجرا، `sessionCatalog.enabled: false` را در پیکربندی Plugin
مربوط به Codex تنظیم کنید.
Plugin همچنان باید روی هر دو رایانه فعال باشد و تنظیم Node رضایت محلی باقی می‌ماند:
فعال‌سازی فقط روی Gateway امکان خواندن وضعیت Codex رایانه‌ای دیگر را فراهم نمی‌کند.

Node فرمان‌های فقط‌خواندنی و نسخه‌بندی‌شدهٔ
`codex.appServer.threads.list.v1` و
`codex.appServer.thread.turns.list.v1` را اعلام می‌کند. میزبان بومی Node که CLI مربوط به Codex
روی آن در دسترس باشد، `codex.terminal.resume.v1` را نیز اعلام می‌کند. هنگامی که این
فرمان‌ها نخستین بار ظاهر می‌شوند، ارتقای جفت‌سازی Node را تأیید کنید. Gateway
آن‌ها را از طریق سیاست معمول Node در Plugin فراخوانی می‌کند و خرابی‌ها را بر اساس
میزبان ایزوله می‌کند.

ردیف‌های Node جفت‌شده به‌صورت یک گروه **Codex** در نوار کناری معمول نشست‌ها ظاهر
می‌شوند. در هر میزبان، ردیف‌ها به‌طور پیش‌فرض بر اساس پوشهٔ پروژه گروه‌بندی
می‌شوند؛ یک پوشهٔ کاری زیر `.claude/worktrees/<name>` در مخزن مبدأ خود ادغام می‌شود و
گروه‌های پروژه مانند سایر بخش‌های نوار کناری جمع می‌شوند. برای مسطح‌کردن یا
بازگرداندن گروه‌های پروژه، از نماد پوشه در سربرگ کاتالوگ استفاده کنید. همین
گروه‌بندی برای کاتالوگ نشست‌های Claude نیز اعمال می‌شود.
به‌طور پیش‌فرض، انتخاب یک ردیف، پنل معمول گفتگو را باز می‌کند و رونوشت پایدار آن
را از طریق فراخوانی‌های محدود و صفحه‌بندی‌شده با مکان‌نما به
`thread/turns/list`، همراه با نگاشت کامل موارد، می‌خواند. برای آغاز
`codex resume <thread-id>` در پایانهٔ اپراتور روی رایانهٔ مالک نشست، از منوی ردیف،
سربرگ نمایشگر یا ترجیح **بازکردن نشست‌های Codex/Claude در** استفاده کنید. مسیر
پایانهٔ Node جفت‌شده، یک رلهٔ PTY در فهرست مجاز و تحت مالکیت Plugin مربوط به
Codex است، نه اجرای دلخواه فرمان روی Node.

این رله قراردادهای کامل ادامهٔ چارچوب OpenClaw و مالکیت بایگانی را ارائه نمی‌دهد.
بنابراین **ادامه** و **بایگانی** برای ردیف‌های راه‌دور در دسترس نیستند. در رایانهٔ
Gateway، ردیف‌های ذخیره‌شده و غیرفعال می‌توانند یک شاخهٔ گفتگوی مجزا و قفل‌شده
به مدل را آغاز کنند. هرکدام فقط پس از آن قابل بایگانی هستند که اپراتور تأیید کند
هیچ کلاینت Codex دیگری از آن استفاده نمی‌کند؛ فعالیت زندهٔ یک ردیف ذخیره‌شده
همچنان نامشخص است. ردیف‌های فعال نمی‌توانند شاخه ایجاد یا بایگانی شوند.

برای راه‌اندازی، صفحه‌بندی، ادامهٔ محلی و مرز امنیتی فراداده، به
[نظارت بر نشست‌های Codex](/plugins/codex-supervision) مراجعه کنید.

### نشست‌ها و رونوشت‌های Claude

Plugin همراه `anthropic` به‌طور پیش‌فرض نشست‌های بایگانی‌نشدهٔ Claude CLI
و Claude Desktop را روی Gateway و Nodeهای جفت‌شده کشف می‌کند. برای غیرفعال‌کردن
کاتالوگ اپراتور و فرمان‌های کاتالوگ Node جفت‌شده، بدون غیرفعال‌کردن مدل‌های
Anthropic یا بک‌اند Claude CLI، `plugins.entries.anthropic.config.sessionCatalog.enabled: false` را تنظیم کنید.
یک Node راه‌دور برنامهٔ macOS، هنگامی که Plugin مربوط به Anthropic فعال باشد و
`~/.claude/projects/` وجود داشته باشد، `anthropic.claude.sessions.list.v1` و
`anthropic.claude.sessions.read.v1` را اعلام می‌کند. هنگامی که این فرمان‌ها نخستین بار ظاهر
می‌شوند، ارتقای جفت‌سازی Node را تأیید کنید.

میزبان بومی Node که Claude CLI روی آن در دسترس باشد، `anthropic.claude.terminal.resume.v1` را نیز
اعلام می‌کند. ردیف‌های واجد شرایط CLI و Desktop می‌توانند `claude --resume <session-id>`
را در پایانهٔ اپراتور روی میزبان مالک خود باز کنند. این کار تصاحب نشست بومی است؛
برخلاف پذیرش توسط OpenClaw، ابتدا نشست Claude را منشعب نمی‌کند.

کاتالوگ، رکوردهای معتبر نمایهٔ پروژهٔ Claude CLI را با یک مسیر جایگزین محدود
فراداده‌ای برای رونوشت‌های JSONL نمایه‌نشده ترکیب می‌کند. این مسیر جایگزین،
نشست‌های هم‌زمان و تعاملیِ غیرزنجیرهٔ جانبی (`cli`) و نشست‌های
CLI بدون رابط Agent SDK ‏(`sdk-cli`) را تشخیص می‌دهد. فرادادهٔ محلی
Claude Desktop عنوان‌های Desktop و وضعیت بایگانی را فراهم می‌کند. هنگامی که هر
دو منبع به شناسهٔ نشست Claude Code یکسانی اشاره کنند، فرادادهٔ Desktop اولویت
دارد؛ رونوشت‌های صرفاً CLI همچنان قابل مشاهده می‌مانند، زیرا CLI پرچم بایگانی
ندارد. خواندن رونوشت از مکان‌نماهای کدر مبتنی بر جابه‌جایی بایت و خواندن محدود
و رو‌به‌عقب فایل استفاده می‌کند؛ بنابراین انتخاب یک نشست بزرگ یا بارگذاری صفحه‌ای
قدیمی‌تر، کل تاریخچهٔ JSONL را در یک پاسخ Gateway نمی‌خواند.

فرمان‌های فهرست و خواندن فقط‌خواندنی هستند. آن‌ها فرادادهٔ کاتالوگ و محتوای
رونوشت را فقط از طریق روش‌های عمومی `sessions.catalog.list` و
`sessions.catalog.read`، به یک اتصال اپراتور احراز هویت‌شده دارای
`operator.write` ارائه می‌کنند. یک ردیف Claude CLI محلی Gateway را می‌توان
از کادر نوشتن معمول گفتگو پذیرفت: OpenClaw تاریخچهٔ قابل‌مشاهده و محدود را وارد
می‌کند، در نوبت نخست با `--fork-session` ادامه می‌دهد و رونوشت مبدأ را
دست‌نخورده باقی می‌گذارد.

یک میزبان Node بدون رابط می‌تواند همین جریان ادامه را فعال کند:

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

Node فقط زمانی `agent.cli.claude.run.v1` را اعلام می‌کند که این تنظیم محلی Node فعال
باشد و فایل اجرایی `claude` روی همان Node حل شود. Gateway نمی‌تواند
آن را از راه دور فعال کند. فرمان همچنین از سیاست تأیید اجرای موجود Node عبور
می‌کند. هنگامی که هر سه فرمان Claude اعلام شوند و سیاست فرمان Node در Gateway
آن‌ها را مجاز بداند، یک ردیف Claude CLI روی آن Node قابل ادامه می‌شود: OpenClaw
تاریخچهٔ محدود را وارد می‌کند، نشست پذیرفته‌شده را به Node و پوشهٔ کاری گزارش‌شده
توسط کاتالوگ آن متصل می‌کند و هر نوبت یک‌بارهٔ `claude -p` را در آنجا
اجرا می‌کند. نوبت نخست همچنان از `--fork-session` استفاده می‌کند و رونوشت
مبدأ را حفظ می‌کند.

نوبت‌های مستقر روی Node از پیش‌فرض‌های Claude همان Node استفاده می‌کنند. در v1
آن‌ها پیکربندی MCP حلقهٔ برگشتی Gateway یا Plugin مهارت‌های Gateway را دریافت
نمی‌کنند، نمی‌توانند از رونوشت Gateway دوباره مقداردهی اولیه شوند و پیوست‌ها و
تصاویر را رد می‌کنند. ردیف‌های Claude Desktop و Nodeهایی که فرمان اجرا را اعلام
نمی‌کنند، فقط قابل مشاهده باقی می‌مانند. Node برنامهٔ macOS هنوز این فرمان را
اعلام نمی‌کند؛ بنابراین ردیف‌های آن فقط قابل مشاهده باقی می‌مانند.

برای رفتار Control UI و منابع ذخیره‌سازی، به
[Anthropic: نشست‌های Claude در چند رایانه](/fa/providers/anthropic#claude-sessions-across-computers)
مراجعه کنید.

### نشست‌های OpenCode و Pi

Pluginهای همراه OpenCode و ACPX نیز کاتالوگ‌های بومی و فقط‌خواندنی نشست را روی
Gateway و Nodeهای جفت‌شده کشف می‌کنند. یک Node، هنگامی که CLI مربوط به
`opencode` نصب باشد، `opencode.sessions.list.v1` / `opencode.sessions.read.v1` را
اعلام می‌کند و هنگامی که پوشهٔ نشست Pi وجود داشته باشد،
`acpx.pi.sessions.list.v1` / `acpx.pi.sessions.read.v1` را اعلام می‌کند. هنگامی که فرمان‌های
جدید نخستین بار ظاهر می‌شوند، ارتقای جفت‌سازی Node را تأیید کنید. وقتی CLI متناظر
نیز در دسترس باشد، Node فرمان `opencode.terminal.resume.v1` یا `acpx.pi.terminal.resume.v1` را
اضافه می‌کند؛ سپس منوی موجود ردیف و سربرگ نمایشگر می‌توانند نشست انتخاب‌شده را
با `opencode --session <id>` یا `pi --session <id>` در پایانهٔ مالک آن دوباره باز کنند.

OpenCode از طریق سطح رسمی JSON/خروجی CLI خود می‌خواند. Pi مخزن مستندشدهٔ نشست
JSONL خود را می‌خواند که شامل پوشه‌های نشست پروژه و سراسری `settings.json`
و همچنین جایگزین‌های `PI_CODING_AGENT_DIR` و `PI_CODING_AGENT_SESSION_DIR` است. هر دو
کاتالوگ به‌طور پیش‌فرض فعال هستند؛ آن‌ها را در Web UI و زیر **Config > Plugins**
خاموش کنید.

ازسرگیری در پایانه از پوشهٔ کاری ذخیره‌شدهٔ نشست و همان رلهٔ PTY دوطرفه و موجود
در فهرست مجاز که Codex و Claude استفاده می‌کنند، بهره می‌گیرد. این قابلیت اجرای
دلخواه فرمان روی Node را ارائه نمی‌دهد.

### بارگذاری فایل در پایانه

Control UI می‌تواند فایل‌ها را به یک ترمینال بازِ node جفت‌شده بکشد. میزبان بومی node فرمان مختص مدیر `terminal.upload` را اعلام می‌کند؛ هنگام نخستین نمایش آن، ارتقای جفت‌سازی را تأیید کنید. هر فایل به 16 MiB محدود است، در یک پوشه موقت خصوصی روی همان node قرار می‌گیرد و بدون اجراشدن، به‌صورت یک مسیر shell-quoted به ترمینال بازگردانده می‌شود.

درج مسیر از PowerShell، `cmd.exe` و پوسته‌های POSIX شناخته‌شده (`sh`، Bash، Dash، Ash، Ksh، Zsh و Fish)، از جمله Git Bash در Windows، پشتیبانی می‌کند. سایر جایگزینی‌های پوسته رد می‌شوند، زیرا قواعد نقل‌قول‌گذاری آن‌ها را نمی‌توان با اطمینان استنباط کرد؛ برای مسیرهای بومی WSL، میزبان node را داخل WSL اجرا کنید. مسیرهای `cmd.exe` که شامل `%` یا `!` باشند نیز رد می‌شوند، زیرا آن پوسته این نویسه‌ها را حتی داخل نقل‌قول دوتایی بسط می‌دهد.

## فراخوانی فرمان‌ها

سطح پایین (RPC خام):

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command canvas.eval --params '{"javaScript":"location.href"}'
```

`nodes invoke`، `system.run` و `system.run.prepare` را مسدود می‌کند؛ این فرمان‌ها فقط از طریق ابزار `exec` با `host=node` اجرا می‌شوند (بالا را ببینید). برای گردش‌کارهای رایج «دادن یک پیوست MEDIA به عامل»، راهنماهای سطح بالاتری وجود دارد (Canvas، دوربین، صفحه‌نمایش، موقعیت مکانی؛ در ادامه).

فرمان‌های جریانی و طولانی‌مدت node از رویدادهای افزایشی `node.invoke.progress`
استفاده می‌کنند. هر رویداد شامل شناسه فراخوانی، یک شماره توالی با مبدأ صفر و یک
قطعه متن UTF-8 با اندازه محدود است؛ Gateway پیش از تحویل قطعه‌ها به
فراخواننده، آن‌ها را مرتب می‌کند. `node.invoke.result` موجود همچنان تنها پاسخ
نهایی است. فراخواننده‌های جریانی می‌توانند یک مهلت عدم‌فعالیت تنظیم کنند که با
اولین رویداد پیشرفت آغاز می‌شود و پس از پیشرفت‌های بعدی بازنشانی می‌شود، درحالی‌که
مهلت سخت جداگانه فراخوانی در طول تأیید و اجرا حفظ می‌شود. نتیجه، مهلت سخت،
مهلت عدم‌فعالیت و قطع اتصال node همگی وضعیت جریان در انتظار را کنار می‌گذارند.
لغو از سوی فراخواننده، `node.invoke.cancel` را منتشر می‌کند؛ سپس میزبان node
درخت فرایند متناظر را خاتمه می‌دهد. فرمان‌های درخواست/پاسخ موجود بدون تغییر هستند.

## خط‌مشی فرمان‌ها

فرمان‌های node پیش از فراخوانی باید از دو دروازه عبور کنند:

1. Node باید فرمان را در فراداده اتصال احراز هویت‌شده خود اعلام کند (`connect.commands`).
2. فهرست مجاز Gateway که از پلتفرم و تأیید مشتق شده است باید فرمان اعلام‌شده را شامل شود.

فهرست‌های مجاز پیش‌فرض بر اساس پلتفرم (پیش از پیش‌فرض‌های Plugin و جایگزینی‌های `commands.allow`/`commands.deny`):

| پلتفرم | فرمان‌های مجاز به‌صورت پیش‌فرض                                                                                                                                                                                                                                                                                           |
| -------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| iOS      | `camera.list`, `location.get`, `device.info`, `device.status`, `contacts.search`, `calendar.events`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer`, `system.notify`                                                                                                                        |
| watchOS  | `device.info`, `device.status`, `system.notify`                                                                                                                                                                                                                                                                       |
| Android  | `camera.list`, `location.get`, `notifications.list`, `notifications.actions`, `system.notify`, `device.info`, `device.status`, `device.permissions`, `device.health`, `device.apps`, `contacts.search`, `calendar.events`, `callLog.search`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer` |
| macOS    | `camera.list`, `location.get`, `device.info`, `device.status`, `device.apps`, `contacts.search`, `calendar.events`, `reminders.list`, `photos.latest`, `motion.activity`, `motion.pedometer`, `system.notify`                                                                                                         |
| Windows  | `camera.list`, `location.get`, `device.info`, `device.status`, `system.notify`                                                                                                                                                                                                                                        |
| Linux    | `system.notify` (فرمان‌های میزبان node مانند `system.run` نیازمند تأیید هستند؛ ادامه را ببینید)                                                                                                                                                                                                                                  |

این ردیف‌ها سقف خط‌مشی Gateway را توصیف می‌کنند، نه فرمان‌هایی که هر برنامه node پیاده‌سازی کرده است. یک فرمان فقط زمانی قابل استفاده است که node متصل نیز آن را اعلام کند. به‌طور خاص، برنامه فعلی macOS خانواده‌های دستگاه و داده‌های شخصی فهرست‌شده در ردیف خط‌مشی macOS را اعلام نمی‌کند.

فرمان‌های `canvas.*` (`canvas.present`، `canvas.hide`، `canvas.navigate`، `canvas.eval`، `canvas.snapshot`، `canvas.a2ui.*`) یک پیش‌فرض Plugin در iOS، Android، macOS، Windows، Linux و پلتفرم‌های ناشناخته هستند. Nodeهای Linux فقط زمانی آن‌ها را اعلام می‌کنند که سوکت محلی Canvas برنامه دسکتاپ موجود باشد. همه فرمان‌های Canvas در iOS به پیش‌زمینه محدود هستند.

`talk.ptt.start`، `talk.ptt.stop`، `talk.ptt.cancel` و `talk.ptt.once` به‌صورت پیش‌فرض برای هر nodeای که قابلیت `talk` را اعلام کند یا فرمان‌های `talk.*` را اظهار کند، مستقل از برچسب پلتفرم مجاز هستند.

فرمان‌های میزبان دسکتاپ (`system.run`، `system.run.prepare`، `system.which`، `browser.proxy`، `mcp.tools.call.v1` و `screen.snapshot` در macOS/Windows/Linux) بخشی از جدول پیش‌فرض ایستای پلتفرم در بالا نیستند. پس از اینکه اپراتور یک درخواست جفت‌سازی را که این فرمان‌ها را اعلام می‌کند تأیید کند، آن‌ها در دسترس قرار می‌گیرند و پس از آن مجموعه فرمان‌های تأییدشده node در اتصال مجدد نیز آن‌ها را حفظ می‌کند.

فرمان‌های خطرناک یا دارای حساسیت بالای حریم خصوصی، حتی اگر node آن‌ها را اعلام کند، همچنان به پذیرش صریح با `gateway.nodes.commands.allow` نیاز دارند: `camera.snap`، `camera.clip`، `screen.record`، `computer.act`، `contacts.add`، `calendar.add`، `reminders.add`، `health.summary`، `sms.send`، `sms.search`. `gateway.nodes.commands.deny` همیشه بر پیش‌فرض‌ها و ورودی‌های اضافی فهرست مجاز مقدم است. برای دروازه رضایت iPhone، [خلاصه‌های HealthKit](/fa/platforms/ios-healthkit) و برای دروازه‌های اضافی قابلیت، خط‌مشی ابزار، مسلح‌سازی و اجراکننده پلتفرم پیرامون ورودی دسکتاپ، [استفاده از رایانه](/fa/nodes/computer-use) را ببینید.

فرمان‌های node متعلق به Plugin می‌توانند یک خط‌مشی فراخوانی node در Gateway اضافه کنند. این خط‌مشی پس از بررسی فهرست مجاز و پیش از ارسال به node اجرا می‌شود، بنابراین `node.invoke` خام، راهنماهای CLI و ابزارهای اختصاصی عامل همگی مرز مجوز یکسان Plugin را به اشتراک می‌گذارند. فرمان‌های خطرناک node متعلق به Plugin همچنان به پذیرش صریح `gateway.nodes.commands.allow` نیاز دارند.

پس از تغییر فهرست فرمان‌های اعلام‌شده یک node، جفت‌سازی قدیمی دستگاه را رد و درخواست جدید را تأیید کنید تا Gateway تصویر لحظه‌ای به‌روزشده فرمان‌ها را ذخیره کند.

## پیکربندی (`openclaw.json`)

تنظیمات مرتبط با node زیر `gateway.nodes` و `tools.exec` قرار دارند:

```json5
{
  gateway: {
    nodes: {
      // تأیید خودکار جفت‌سازی اولیه node از شبکه‌های مورد اعتماد (فهرست CIDR).
      // در صورت تنظیم‌نشدن غیرفعال است. فقط برای درخواست‌های اولیه role:node
      // بدون حوزه‌های درخواستی اعمال می‌شود؛ ارتقاها را خودکار تأیید نمی‌کند.
      pairing: {
        autoApproveCidrs: ["192.168.1.0/24"],
        // تأیید خودکارِ راستی‌آزمایی‌شده با SSH (پیش‌فرض: فعال). جفت‌سازی اولیه
        // node را با تطبیق دقیق کلید دستگاه که از طریق SSH بازخوانی شده تأیید می‌کند.
        sshVerify: true,
      },
      // اعتماد به ابزارهای Plugin قابل مشاهده برای عامل که nodeهای جفت‌شده منتشر می‌کنند (پیش‌فرض: true).
      pluginTools: {
        enabled: true,
      },
      // پذیرش فرمان‌های خطرناک/دارای حساسیت بالای حریم خصوصی node (camera.snap و غیره).
      commands: {
        allow: ["camera.snap", "screen.record"],
        // مسدودکردن نام دقیق فرمان‌ها، حتی اگر پیش‌فرض‌ها یا commands.allow شامل آن‌ها باشند.
        deny: ["camera.clip"],
      },
    },
  },
  tools: {
    exec: {
      // میزبان پیش‌فرض exec: مقدار "node" همه فراخوانی‌های exec را به یک node جفت‌شده هدایت می‌کند.
      host: "node",
      // حالت امنیتی exec در node: فقط فرمان‌های تأییدشده/موجود در فهرست مجاز را اجازه می‌دهد.
      security: "allowlist",
      // مقیدکردن exec به یک node مشخص (شناسه یا نام). برای اجازه‌دادن به هر node حذف شود.
      node: "build-node",
    },
  },
}
```

از نام دقیق فرمان‌های node استفاده کنید. `commands.deny` یک فرمان را حتی زمانی حذف می‌کند که پیش‌فرض پلتفرم یا ورودی `commands.allow` در غیر این صورت آن را مجاز کند. Nodeهای جفت‌شده می‌توانند به‌صورت پیش‌فرض توصیفگرهای ابزار Plugin قابل مشاهده برای عامل را منتشر کنند، اما فرمان هر توصیفگر همچنان باید در سطح فرمان‌های تأییدشده node باشد. برای نادیده‌گرفتن همه این توصیفگرها، `gateway.nodes.pluginTools.enabled: false` را تنظیم کنید. برای جزئیات فیلدهای جفت‌سازی node و خط‌مشی فرمان Gateway، [مرجع پیکربندی Gateway](/fa/gateway/configuration-reference#gateway) را ببینید.

جایگزینی node برای exec به‌ازای هر عامل:

```json5
{
  agents: {
    list: [
      {
        id: "main",
        tools: { exec: { node: "build-node" } },
      },
    ],
  },
}
```

## نماگرفت‌ها (تصاویر لحظه‌ای Canvas)

اگر node در حال نمایش Canvas (WebView) باشد، `canvas.snapshot` مقدار `{ format, base64 }` را بازمی‌گرداند.

راهنمای CLI (در یک فایل موقت می‌نویسد و مسیر ذخیره‌شده را چاپ می‌کند):

```bash
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format png
openclaw nodes canvas snapshot --node <idOrNameOrIp> --format jpg --max-width 1200 --quality 0.9
```

### کنترل‌های Canvas

```bash
openclaw nodes canvas present --node <idOrNameOrIp> --target https://example.com
openclaw nodes canvas hide --node <idOrNameOrIp>
openclaw nodes canvas navigate https://example.com --node <idOrNameOrIp>
openclaw nodes canvas eval --node <idOrNameOrIp> --js "document.title"
```

نکته‌ها:

- `canvas present` در nodeهایی که از مسیرهای محلی پشتیبانی می‌کنند، URLها یا مسیرهای فایل محلی (`--target`) و همچنین `--x/--y/--width/--height` اختیاری برای موقعیت‌دهی را می‌پذیرد. Canvas در Linux، URLهای HTTP(S) یا رندرکننده A2UI همراه خود را می‌پذیرد.
- `canvas eval`، JS درون‌خطی (`--js`) یا یک آرگومان موقعیتی را می‌پذیرد.

### A2UI (Canvas)

```bash
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --text "Hello"
openclaw nodes canvas a2ui push --node <idOrNameOrIp> --jsonl ./payload.jsonl
openclaw nodes canvas a2ui reset --node <idOrNameOrIp>
```

نکته‌ها:

- Nodeهای موبایل و دسکتاپ Linux برای رندر دارای قابلیت کنش از یک صفحه A2UI همراه و متعلق به برنامه استفاده می‌کنند.
- فقط JSONL نسخه A2UI v0.8 پشتیبانی می‌شود (v0.9/createSurface رد می‌شود).
- iOS و Android صفحات Canvas راه‌دور Gateway را رندر می‌کنند، اما کنش‌های دکمه A2UI فقط از صفحه A2UI همراه و متعلق به برنامه ارسال می‌شوند. صفحات HTTP/HTTPS A2UI میزبانی‌شده در Gateway در آن کلاینت‌های موبایل فقط قابل رندر هستند.
- macOS می‌تواند کنش‌ها را از صفحه دقیق A2UI در Gateway که محدود به قابلیت است و برنامه آن را انتخاب کرده، ارسال کند. سایر صفحات HTTP/HTTPS فقط قابل رندر باقی می‌مانند.
- Linux کنش‌ها را فقط از صفحه A2UI همراه ارسال می‌کند. سایر صفحات HTTP/HTTPS فقط قابل رندر باقی می‌مانند و یک node بدون رابط گرافیکی Linux که برنامه دسکتاپ را ندارد، Canvas را اعلام نمی‌کند.

## عکس‌ها و ویدئوها (دوربین node)

عکس‌ها (`jpg`):

```bash
openclaw nodes camera list --node <idOrNameOrIp>
openclaw nodes camera snap --node <idOrNameOrIp>            # پیش‌فرض: هر دو جهت دوربین (۲ خط MEDIA)
openclaw nodes camera snap --node <idOrNameOrIp> --facing front
openclaw nodes camera snap --node <idOrNameOrIp> --device-id <id> --max-width 1200 --quality 0.9 --delay-ms 2000
```

کلیپ‌های ویدیویی (`mp4`):

```bash
openclaw nodes camera clip --node <idOrNameOrIp> --duration 10s
openclaw nodes camera clip --node <idOrNameOrIp> --duration 3000 --no-audio
```

نکته‌ها:

- برای `canvas.*` و `camera.*`، Node باید **در پیش‌زمینه** باشد (فراخوانی‌های پس‌زمینه `NODE_BACKGROUND_UNAVAILABLE` را برمی‌گردانند).
- Nodeها مدت کلیپ را محدود می‌کنند تا بار base64 قابل‌مدیریت بماند (برای محدودیت‌های دقیق هر پلتفرم، [ضبط دوربین](/fa/nodes/camera) را ببینید). ابزار عامل `nodes` نیز پیش از ارسال فراخوانی، `durationMs` درخواستی را به 300000 (۵ دقیقه) محدود می‌کند؛ خود Node محدودیت سخت‌گیرانه‌تر را اعمال می‌کند.
- Android در صورت امکان برای مجوزهای `CAMERA`/`RECORD_AUDIO` درخواست می‌دهد؛ رد مجوزها با `*_PERMISSION_REQUIRED` ناموفق می‌شود.

## ضبط‌های صفحه‌نمایش (Nodeها)

Nodeهای پشتیبانی‌شده `screen.record` (mp4) را ارائه می‌کنند. نمونه:

```bash
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10
openclaw nodes screen record --node <idOrNameOrIp> --duration 10s --fps 10 --no-audio
```

نکته‌ها:

- دردسترس‌بودن `screen.record` به پلتفرم Node بستگی دارد.
- ابزار عامل `nodes`، `durationMs` درخواستی را به 300000 (۵ دقیقه) محدود می‌کند؛ Node ممکن است برای محدودکردن اندازه بار بازگشتی، محدودیت سخت‌گیرانه‌تری اعمال کند.
- `--no-audio` ضبط میکروفن را در پلتفرم‌های پشتیبانی‌شده غیرفعال می‌کند.
- وقتی چند صفحه‌نمایش در دسترس است، برای انتخاب نمایشگر از `--screen <index>` استفاده کنید (0 = اصلی).

## موقعیت مکانی (Nodeها)

وقتی موقعیت مکانی در تنظیمات فعال باشد، Nodeها `location.get` را ارائه می‌کنند.

ابزار کمکی CLI:

```bash
openclaw nodes location get --node <idOrNameOrIp>
openclaw nodes location get --node <idOrNameOrIp> --accuracy precise --max-age 15000 --location-timeout 10000
```

نکته‌ها:

- موقعیت مکانی **به‌طور پیش‌فرض غیرفعال است**.
- «همیشه» به مجوز سیستم نیاز دارد؛ واکشی در پس‌زمینه به‌صورت بهترین تلاش انجام می‌شود.
- پاسخ شامل عرض/طول جغرافیایی، دقت (متر) و برچسب زمانی است.
- شکل کامل پارامتر/پاسخ و کدهای خطا: [فرمان موقعیت مکانی](/fa/nodes/location-command).

## SMS (Nodeهای Android)

Nodeهای Android می‌توانند هنگامی که کاربر مجوز **SMS** را اعطا کرده و دستگاه از تلفن پشتیبانی می‌کند، `sms.send` و `sms.search` را ارائه کنند. هر دو فرمان به‌طور پیش‌فرض خطرناک هستند: اپراتور Gateway باید پیش از امکان فراخوانی، آن‌ها را نیز به `gateway.nodes.commands.allow` اضافه کند ([سیاست فرمان](#command-policy) را ببینید).

برای جست‌وجوی فقط‌خواندنی SMS، صریحاً در `openclaw.json` شرکت کنید:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["sms.search"] },
    },
  },
}
```

تنها زمانی `sms.send` را جداگانه اضافه کنید که Node باید قادر به ارسال پیام نیز باشد. مجوز Android و مجوزدهی فرمان Gateway مستقل هستند؛ اعطای مجوز تلفن، سیاست Gateway را ویرایش نمی‌کند.

فراخوانی سطح پایین:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command sms.send --params '{"to":"+15555550123","message":"Hello from OpenClaw"}'
```

نکته‌ها:

- ممکن است `sms.search` پیش از اعطای `READ_SMS` اعلام شود تا یک فراخوانی بتواند پیام تشخیصی مجوز را برگرداند؛ خواندن پیام‌ها همچنان به آن مجوز Android نیاز دارد.
- دستگاه‌های صرفاً Wi-Fi و فاقد تلفن، `sms.send` را اعلام نمی‌کنند.
- خطای `requires explicit gateway.nodes.commands.allow opt-in` به این معناست که تلفن فرمان را اعلام کرده، اما اپراتور Gateway آن را مجاز نکرده است.

## فرمان‌های دستگاه و داده‌های شخصی

Nodeهای iOS و Android به‌طور پیش‌فرض چندین فرمان داده فقط‌خواندنی را اعلام می‌کنند (جدول [سیاست فرمان](#command-policy) را ببینید)؛ Android علاوه بر آن، خانواده بزرگ‌تری را ارائه می‌کند که تنظیمات درون‌برنامه‌ای خودش دسترسی به آن را کنترل می‌کند. میزبان Node TypeScript در macOS یا مک بدون رابط، تنها پس از آنکه اپراتور اشتراک‌گذاری برنامه‌های نصب‌شده را با `--share-installed-apps` فعال کند، `device.apps` را اعلام می‌کند.

خانواده‌های موجود:

- `device.status`، `device.info` — iOS، Android، Windows.
- `device.permissions`، `device.health` — فقط Android.
- `device.apps` — Nodeهای Android، macOS و مک بدون رابط. Android به اشتراک‌گذاری برنامه‌های نصب‌شده در Settings نیاز دارد و به‌طور پیش‌فرض برنامه‌های قابل‌مشاهده در راه‌انداز را برمی‌گرداند. میزبان‌های Node TypeScript اشتراک‌گذاری را به‌طور پیش‌فرض غیرفعال نگه می‌دارند و `query`، `limit` و `includeSystem` را می‌پذیرند؛ نتایج macOS شامل `label`، `bundleId`، `path` و `system` هستند.
- `notifications.list`، `notifications.actions` — فقط Android.
- `photos.latest` — iOS، Android.
- `contacts.search` — iOS، Android (پیش‌فرض فقط‌خواندنی)؛ `contacts.add` خطرناک است و به `gateway.nodes.commands.allow` نیاز دارد.
- `calendar.events` — iOS، Android (پیش‌فرض فقط‌خواندنی)؛ `calendar.add` خطرناک است و به `gateway.nodes.commands.allow` نیاز دارد.
- `reminders.list` — iOS، Android (پیش‌فرض فقط‌خواندنی)؛ `reminders.add` خطرناک است و به `gateway.nodes.commands.allow` نیاز دارد.
- `callLog.search` — فقط Android.
- `motion.activity`، `motion.pedometer` — iOS، Android؛ دسترسی بر اساس حسگرهای موجود کنترل می‌شود.

نمونه فراخوانی‌ها:

```bash
openclaw nodes invoke --node <idOrNameOrIp> --command device.status --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command device.apps --params '{"limit":10}'
openclaw nodes invoke --node <idOrNameOrIp> --command notifications.list --params '{}'
openclaw nodes invoke --node <idOrNameOrIp> --command photos.latest --params '{"limit":1}'
```

## فرمان‌های سیستم (میزبان Node / Node مک)

Node در macOS، `system.run`، `system.which`، `system.notify` و `system.execApprovals.get/set` را ارائه می‌کند. میزبان Node بدون رابط، `system.run.prepare`، `system.run`، `system.which` و `system.execApprovals.get/set` را ارائه می‌کند.

نمونه‌ها:

```bash
openclaw nodes notify --node <idOrNameOrIp> --title "Ping" --body "Gateway ready"
openclaw nodes invoke --node <idOrNameOrIp> --command system.which --params '{"bins":["git"]}'
```

نکته‌ها:

- `system.run` خروجی استاندارد/خطای استاندارد/کد خروج را در بار برمی‌گرداند.
- اجرای پوسته اکنون از طریق ابزار `exec` با `host=node` انجام می‌شود؛ `nodes` سطح RPC مستقیم برای فرمان‌های صریح Node باقی می‌ماند.
- `nodes invoke`، `system.run` یا `system.run.prepare` را ارائه نمی‌کند؛ آن‌ها فقط در مسیر اجرا باقی می‌مانند.
- مسیر اجرا پیش از تأیید، یک `systemRunPlan` متعارف آماده می‌کند. پس از اعطای تأیید، Gateway همان طرح ذخیره‌شده را ارسال می‌کند، نه فیلدهای فرمان/cwd/نشست که فراخواننده بعداً ویرایش کرده باشد.
- `system.notify` وضعیت مجوز اعلان در برنامه macOS را رعایت می‌کند؛ از `--priority <passive|active|timeSensitive>` و `--delivery <system|overlay|auto>` پشتیبانی می‌کند.
- فراداده `platform` / `deviceFamily` ناشناخته Node از فهرست مجاز پیش‌فرض محافظه‌کارانه‌ای استفاده می‌کند که `system.run` و `system.which` را مستثنا می‌کند. اگر عمداً برای یک پلتفرم ناشناخته به این فرمان‌ها نیاز دارید، آن‌ها را صریحاً از طریق `gateway.nodes.commands.allow` اضافه کنید.
- `system.run` از `--cwd`، `--env KEY=VAL`، `--command-timeout` و `--needs-screen-recording` پشتیبانی می‌کند.
- برای پوشش‌دهنده‌های پوسته (`bash|sh|zsh ... -c/-lc`)، مقادیر `--env` محدود به درخواست به یک فهرست مجاز صریح (`TERM`، `LANG`، `LC_*`، `COLORTERM`، `NO_COLOR`، `FORCE_COLOR`) کاهش می‌یابند.
- برای تصمیم‌های همیشه‌مجاز در حالت فهرست مجاز، پوشش‌دهنده‌های توزیع شناخته‌شده (`env`، `flock`، `nice`، `nohup`، `stdbuf`، `timeout`) به‌جای مسیرهای پوشش‌دهنده، مسیرهای فایل اجرایی داخلی را ماندگار می‌کنند. اگر بازکردن پوشش ایمن نباشد، هیچ ورودی فهرست مجازی به‌طور خودکار ماندگار نمی‌شود.
- در میزبان‌های Node ویندوز در حالت فهرست مجاز، اجراهای پوشش‌دهنده پوسته از طریق `cmd.exe /c` به تأیید نیاز دارند (صرف وجود ورودی در فهرست مجاز، شکل پوشش‌دهنده را خودکار مجاز نمی‌کند).
- میزبان‌های Node جایگزینی‌های `PATH` در `--env` را نادیده می‌گیرند و پیش از اجرای فرمان، مجموعه بزرگ و نگه‌داری‌شده‌ای از متغیرهای راه‌اندازی مفسر/پوسته (برای نمونه `NODE_OPTIONS`، `PYTHONPATH`، `BASH_ENV`، `DYLD_*`، `LD_*`) را حذف می‌کنند. اگر به ورودی‌های PATH بیشتری نیاز دارید، به‌جای ارسال `PATH` از طریق `--env`، محیط سرویس میزبان Node را پیکربندی کنید (یا ابزارها را در مکان‌های استاندارد نصب کنید).
- در حالت Node در macOS، دسترسی به `system.run` با تأییدهای اجرا در برنامه macOS کنترل می‌شود (Settings → Exec approvals). حالت‌های پرسش/فهرست مجاز/کامل همانند میزبان Node بدون رابط رفتار می‌کنند؛ درخواست‌های ردشده `SYSTEM_RUN_DENIED` را برمی‌گردانند.
- در میزبان Node بدون رابط، دسترسی به `system.run` با تأییدهای اجرا (`~/.openclaw/exec-approvals.json`) کنترل می‌شود؛ مشخصاً در macOS، متغیرهای محیطی مسیریابی میزبان اجرا را در بخش [میزبان Node بدون رابط](#headless-node-host-cross-platform) در ادامه ببینید.

## اتصال Node اجرا

وقتی چند Node در دسترس است، می‌توانید اجرا را به Node مشخصی متصل کنید. این کار Node پیش‌فرض را برای `exec host=node` تعیین می‌کند (و می‌توان آن را برای هر عامل جایگزین کرد).

پیش‌فرض سراسری:

```bash
openclaw config set tools.exec.node "node-id-or-name"
```

جایگزینی برای هر عامل:

```bash
openclaw config get agents.entries
openclaw config set 'agents.entries.main.tools.exec.node' "node-id-or-name"
```

برای مجازکردن هر Node، تنظیم را حذف کنید:

```bash
openclaw config unset tools.exec.node
openclaw config unset 'agents.entries.main.tools.exec.node'
```

## نگاشت مجوزها

Nodeها ممکن است در `node.list` / `node.describe` یک نگاشت `permissions` داشته باشند که با نام مجوز (برای نمونه `screenRecording`، `accessibility`، `location`) کلیدگذاری شده و مقادیر بولی دارد (`true` = اعطاشده).

## میزبان Node بدون رابط (چندسکویی)

OpenClaw می‌تواند یک **میزبان Node بدون رابط** (بدون رابط کاربری) اجرا کند که به WebSocket در Gateway متصل می‌شود و `system.run` / `system.which` را ارائه می‌کند. این قابلیت در Linux/Windows یا برای اجرای یک Node حداقلی در کنار سرور مفید است.

آن را راه‌اندازی کنید:

```bash
openclaw node run --host <gateway-host> --port 18789
```

نکته‌ها:

- جفت‌سازی همچنان الزامی است (Gateway درخواست جفت‌سازی دستگاه را نمایش می‌دهد).
- فراداده نمونه کلاینت، هویت امضاشده دستگاه و احراز هویت جفت‌سازی از رکوردهای وضعیت جداگانه استفاده می‌کنند؛ [وضعیت هویت بدون رابط](#headless-identity-state) را ببینید.
- تأییدهای اجرا به‌صورت محلی از طریق `~/.openclaw/exec-approvals.json` اعمال می‌شوند ([تأییدهای اجرا](/fa/tools/exec-approvals) را ببینید).
- در macOS، میزبان Node بدون رابط به‌طور پیش‌فرض `system.run` را محلی اجرا می‌کند. برای مسیریابی `system.run` از طریق میزبان اجرای برنامه همراه، `OPENCLAW_NODE_EXEC_HOST=app` را تنظیم کنید؛ `OPENCLAW_NODE_EXEC_FALLBACK=0` را اضافه کنید تا میزبان برنامه الزامی شود و در صورت دردسترس‌نبودن آن، عملیات به‌شکل بسته ناموفق شود.
- وقتی WebSocket در Gateway از TLS استفاده می‌کند، `--tls` / `--tls-fingerprint` را اضافه کنید.

## حالت Node مک

- برنامه نوار منوی macOS به‌عنوان یک Node به سرور WebSocket در Gateway متصل می‌شود (بنابراین `openclaw nodes …` روی این مک کار می‌کند).
- در حالت راه‌دور، برنامه یک تونل SSH برای درگاه Gateway باز می‌کند و به `localhost` متصل می‌شود.
