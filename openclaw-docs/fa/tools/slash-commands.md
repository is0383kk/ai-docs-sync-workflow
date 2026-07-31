---
read_when:
    - استفاده یا پیکربندی فرمان‌های چت
    - اشکال‌زدایی مسیریابی فرمان‌ها یا مجوزها
    - آشنایی با نحوه ثبت فرمان‌های Skills
sidebarTitle: Slash commands
summary: همهٔ فرمان‌های اسلش، دستورالعمل‌ها و میان‌برهای درون‌خطی موجود — پیکربندی، مسیریابی و رفتار ویژهٔ هر سطح.
title: دستورهای اسلش
x-i18n:
    generated_at: "2026-07-27T15:56:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ee5ee5e46d632a54ea92dea7ca61046288bf1998d05b08396107bec90e646fff
    source_path: tools/slash-commands.md
    workflow: 16
---

Gateway فرمان‌هایی را مدیریت می‌کند که به‌صورت پیام‌های مستقل و با `/` شروع می‌شوند.
فرمان‌های bash مختص میزبان از `! <cmd>` استفاده می‌کنند (و `/bash <cmd>` نام مستعار آن است).

وقتی یک مکالمه به نشست ACP متصل است، متن عادی به
سامانه ACP هدایت می‌شود. فرمان‌های مدیریتی Gateway محلی باقی می‌مانند: `/acp ...` همیشه به
مدیریت‌کننده فرمان OpenClaw می‌رسد و هرگاه مدیریت فرمان برای سطح فعال باشد،
`/status` و `/unfocus` نیز محلی باقی می‌مانند.

## سه نوع فرمان

<CardGroup cols={3}>
  <Card title="فرمان‌ها" icon="terminal">
    پیام‌های مستقل `/...` که Gateway آن‌ها را مدیریت می‌کند. باید به‌عنوان
    تنها محتوای پیام ارسال شوند.
  </Card>
  <Card title="دستورالعمل‌ها" icon="sliders">
    `/think`، `/fast`، `/verbose`، `/trace`، `/reasoning`، `/elevated`،
    `/exec`، `/model`، `/queue` — پیش از مشاهده پیام توسط مدل
    از آن حذف می‌شوند. اگر به‌تنهایی ارسال شوند، تنظیمات نشست را ماندگار می‌کنند؛
    اگر همراه متن دیگری ارسال شوند، به‌صورت راهنمای درون‌خطی عمل می‌کنند.
  </Card>
  <Card title="میان‌برهای درون‌خطی" icon="bolt">
    `/help`، `/commands`، `/status`، `/whoami` — بی‌درنگ اجرا می‌شوند و
    پیش از مشاهده متن باقی‌مانده توسط مدل حذف می‌شوند. فقط برای فرستندگان مجاز.
  </Card>
</CardGroup>

<AccordionGroup>
  <Accordion title="جزئیات رفتار دستورالعمل‌ها">
    - دستورالعمل‌ها پیش از مشاهده پیام توسط مدل از آن حذف می‌شوند.
    - در پیام‌های **فقط شامل دستورالعمل** (پیام فقط شامل دستورالعمل‌ها است)، در
      نشست ماندگار می‌شوند و با یک تأیید پاسخ می‌دهند.
    - در پیام‌های **گفت‌وگوی عادی** که متن دیگری نیز دارند، به‌صورت راهنمای درون‌خطی عمل می‌کنند و
      تنظیمات نشست را ماندگار **نمی‌کنند**.
    - دستورالعمل‌ها فقط برای **فرستندگان مجاز** اعمال می‌شوند. اگر `commands.allowFrom`
      تنظیم شده باشد، تنها فهرست مجاز مورداستفاده است؛ در غیر این صورت، مجوز از
      فهرست‌های مجاز کانال، جفت‌سازی و اعمال همیشگی گروه دسترسی به‌دست می‌آید. برای فرستندگان
      غیرمجاز، دستورالعمل‌ها به‌صورت متن ساده در نظر گرفته می‌شوند.
  </Accordion>
</AccordionGroup>

## پیکربندی

```json5
{
  commands: {
    native: "auto",
    nativeSkills: "auto",
    text: true,
    bash: false,
    bashForegroundMs: 2000,
    config: false,
    mcp: false,
    plugins: false,
    debug: false,
    restart: true,
    ownerAllowFrom: ["discord:123456789012345678"],
    ownerDisplay: "raw",
    ownerDisplaySecret: "${OWNER_ID_HASH_SECRET}",
    allowFrom: {
      "*": ["user1"],
      discord: ["user:123"],
    },
    useAccessGroups: true,
  },
}
```

<ParamField path="commands.text" type="boolean" default="true">
  تجزیه `/...` را در پیام‌های گفت‌وگو فعال می‌کند. در سطوح بدون فرمان‌های بومی
  (WhatsApp، WebChat، Signal، iMessage، Google Chat، Microsoft Teams)، فرمان‌های متنی
  حتی با تنظیم روی `false` نیز کار می‌کنند.
</ParamField>

<ParamField path="commands.native" type='boolean | "auto"' default='"auto"'>
  فرمان‌های بومی را ثبت می‌کند. حالت خودکار: برای Discord/Telegram روشن؛ برای Slack خاموش؛
  برای ارائه‌دهندگان بدون پشتیبانی بومی نادیده گرفته می‌شود. برای هر کانال با
  `channels.<provider>.commands.native` بازنویسی کنید. در Discord، `false` ثبت
  فرمان اسلش را نادیده می‌گیرد؛ فرمان‌های ثبت‌شده قبلی ممکن است تا زمان حذف قابل‌مشاهده بمانند.
</ParamField>

<ParamField path="commands.nativeSkills" type='boolean | "auto"' default='"auto"'>
  در صورت پشتیبانی، فرمان‌های Skills را به‌صورت بومی ثبت می‌کند. حالت خودکار: برای
  Discord/Telegram روشن؛ برای Slack خاموش. با
  `channels.<provider>.commands.nativeSkills` بازنویسی کنید.
</ParamField>

<ParamField path="commands.bash" type="boolean" default="false">
  `! <cmd>` را برای اجرای فرمان‌های پوسته میزبان فعال می‌کند (نام مستعار `/bash <cmd>`). به
  فهرست‌های مجاز `tools.elevated` نیاز دارد.
</ParamField>

<ParamField path="commands.bashForegroundMs" type="number" default="2000">
  مدت انتظار bash پیش از تغییر به حالت پس‌زمینه را تعیین می‌کند (`0` بی‌درنگ
  به پس‌زمینه می‌رود).
</ParamField>

<ParamField path="commands.config" type="boolean" default="false">
  `/config` را فعال می‌کند (`openclaw.json` را می‌خواند/می‌نویسد). فقط مالک.
</ParamField>

<ParamField path="commands.mcp" type="boolean" default="false">
  `/mcp` را فعال می‌کند (پیکربندی MCP مدیریت‌شده توسط OpenClaw را در `mcp.servers` می‌خواند/می‌نویسد). فقط مالک.
</ParamField>

<ParamField path="commands.plugins" type="boolean" default="false">
  `/plugins` را فعال می‌کند (کشف/وضعیت Plugin به‌همراه نصب و فعال/غیرفعال‌سازی). عملیات نوشتن فقط برای مالک است.
</ParamField>

<ParamField path="commands.debug" type="boolean" default="false">
  `/debug` را فعال می‌کند (بازنویسی‌های پیکربندی فقط در زمان اجرا). فقط مالک.
</ParamField>

<ParamField path="commands.restart" type="boolean" default="true">
  `/restart` و درخواست‌های راه‌اندازی مجدد خارجی `SIGUSR1` را فعال می‌کند.
</ParamField>

<ParamField path="commands.ownerAllowFrom" type="string[]">
  فهرست مجاز صریح مالک برای سطوح فرمان مختص مالک. جدا از
  `commands.allowFrom` و دسترسی جفت‌سازی پیام مستقیم است.
</ParamField>

<ParamField path="channels.<channel>.commands.enforceOwnerForCommands" type="boolean" default="false">
  به‌ازای هر کانال: برای فرمان‌های مختص مالک، هویت مالک را الزامی می‌کند. وقتی `true`،
  فرستنده باید با `commands.ownerAllowFrom` مطابقت داشته باشد یا دامنهٔ داخلی `operator.admin`
  را در اختیار داشته باشد. ورودی عام `allowFrom` به‌تنهایی کافی **نیست**.
</ParamField>

<ParamField path="commands.ownerDisplay" type='"raw" | "hash"'>
  نحوهٔ نمایش شناسه‌های مالک در اعلان سیستم را کنترل می‌کند.
</ParamField>

<ParamField path="commands.ownerDisplaySecret" type="string">
  راز HMAC که هنگام `commands.ownerDisplay: "hash"` استفاده می‌شود.
</ParamField>

<ParamField path="commands.allowFrom" type="object">
  فهرست مجاز به‌ازای هر ارائه‌دهنده برای مجوزدهی فرمان‌ها. وقتی پیکربندی شود،
  **تنها** منبع مجوزدهی برای فرمان‌ها و دستورالعمل‌ها است. از `"*"` برای
  پیش‌فرض سراسری استفاده کنید؛ کلیدهای ویژهٔ ارائه‌دهنده آن را بازنویسی می‌کنند.
</ParamField>

## فهرست فرمان‌ها

فرمان‌ها از سه منبع می‌آیند:

- **فرمان‌های داخلی هسته:** `src/auto-reply/commands-registry.shared.ts`
- **فرمان‌های تولیدشدهٔ داک:** `src/auto-reply/commands-registry.data.ts`
- **فرمان‌های Plugin:** فراخوانی‌های `registerCommand()` در Plugin

دسترس‌پذیری به پرچم‌های پیکربندی، سطح کانال و Pluginهای
نصب‌شده/فعال بستگی دارد.

### فرمان‌های هسته

<AccordionGroup>
  <Accordion title="نشست‌ها و اجراها">
    | فرمان | توضیحات |
    | --- | --- |
    | `/new [model]` | نشست فعلی را بایگانی و نشستی تازه آغاز می‌کند |
    | `/reset [soft [message]]` | نشست فعلی را درجا بازنشانی می‌کند. `soft` رونوشت را نگه می‌دارد، شناسه‌های نشستِ استفاده‌شدهٔ مجدد در بک‌اند CLI را حذف می‌کند و راه‌اندازی را دوباره اجرا می‌کند |
    | `/name <title>` | نشست فعلی را نام‌گذاری یا نام آن را تغییر می‌دهد. برای دیدن نام فعلی و یک پیشنهاد، عنوان را وارد نکنید |
    | `/compact [instructions]` | زمینهٔ نشست را فشرده می‌کند. [Compaction](/fa/concepts/compaction) را ببینید |
    | `/stop` | اجرای فعلی را لغو می‌کند |
    | `/session idle <duration\|off>` | انقضای بی‌کاری اتصال رشته را مدیریت می‌کند |
    | `/session max-age <duration\|off>` | انقضای حداکثر عمر اتصال رشته را مدیریت می‌کند |
    | `/export-session [path]` | فقط مالک. نشست فعلی را در فضای کاری به HTML صادر می‌کند. نام مستعار: `/export` |
    | `/export-trajectory [path]` | یک بستهٔ مسیر JSONL برای نشست فعلی صادر می‌کند. نام مستعار: `/trajectory` |

    مسیرهای صریح `/export-session` فایل‌های موجود در
    فضای کاری را جایگزین می‌کنند. برای تولید نام فایلی ایمن در برابر تداخل، مسیر را وارد نکنید.

    <Note>
      Control UI، ورودی تایپ‌شدهٔ `/new` را رهگیری می‌کند تا یک نشست تازهٔ
      داشبورد بسازد و به آن جابه‌جا شود؛ مگر وقتی `session.dmScope: "main"` پیکربندی شده باشد
      و والد فعلی، نشست اصلی عامل باشد — در این حالت `/new`
      نشست اصلی را درجا بازنشانی می‌کند. ورودی تایپ‌شدهٔ `/reset` همچنان
      بازنشانی درجای Gateway را اجرا می‌کند. وقتی می‌خواهید انتخاب مدل سنجاق‌شدهٔ
      نشست را پاک کنید، از `/model default` استفاده کنید.
    </Note>

  </Accordion>

  <Accordion title="کنترل‌های مدل و اجرا">
    | فرمان | توضیحات |
    | --- | --- |
    | `/think <level\|default>` | سطح تفکر را تنظیم یا بازنویسی نشست را پاک می‌کند. نام‌های مستعار: `/thinking`، `/t` |
    | `/verbose on\|off\|full` | خروجی مشروح را تغییر وضعیت می‌دهد. نام مستعار: `/v` |
    | `/trace on\|off` | خروجی ردیابی Plugin را برای نشست فعلی تغییر وضعیت می‌دهد |
    | `/fast [status\|auto\|on\|off\|default]` | حالت سریع را نمایش می‌دهد، تنظیم می‌کند یا پاک می‌کند |
    | `/reasoning [on\|off\|stream]` | نمایش استدلال را تغییر وضعیت می‌دهد. نام مستعار: `/reason` |
    | `/elevated [on\|off\|ask\|full]` | حالت ارتقایافته را تغییر وضعیت می‌دهد. نام مستعار: `/elev` |
    | `/exec host=<auto\|sandbox\|gateway\|node> security=<deny\|allowlist\|full> ask=<off\|on-miss\|always> node=<id>` | پیش‌فرض‌های اجرا را نمایش می‌دهد یا تنظیم می‌کند |
    | `/login [codex\|openai\|openai-codex]` | ورود Codex/OpenAI را از یک گفت‌وگوی خصوصی یا نشست Web UI جفت می‌کند. فقط مالک/مدیر |
    | `/model [name\|#\|status]` | مدل را نمایش می‌دهد یا تنظیم می‌کند |
    | `/models [provider] [page] [limit=<n>\|all]` | ارائه‌دهندگان یا مدل‌های پیکربندی‌شده/دارای احراز هویت را فهرست می‌کند |
    | `/queue <mode>` | رفتار صف اجرای فعال را مدیریت می‌کند. [صف](/fa/concepts/queue) و [هدایت صف](/fa/concepts/queue-steering) را ببینید |
    | `/steer <message>` | راهنمایی را به اجرای فعال تزریق می‌کند. نام مستعار: `/tell`. [هدایت](/fa/tools/steer) را ببینید |

    <AccordionGroup>
      <Accordion title="ایمنی حالت‌های مشروح / ردیابی / سریع / استدلال">
        - `/verbose` برای اشکال‌زدایی است — در استفادهٔ عادی آن را **خاموش** نگه دارید.
        - `/trace` فقط خطوط ردیابی/اشکال‌زدایی متعلق به Plugin را آشکار می‌کند؛ پیام‌های مشروح عادی خاموش می‌مانند.
        - `/fast auto|on|off` بازنویسی نشست را پایدار نگه می‌دارد؛ برای پاک‌کردن آن از گزینهٔ `inherit` در رابط Sessions استفاده کنید.
        - `/fast` ویژهٔ ارائه‌دهنده است: OpenAI/Codex آن را به `service_tier=priority` نگاشت می‌کنند؛ درخواست‌های مستقیم Anthropic آن را به `service_tier=auto` یا `standard_only` نگاشت می‌کنند.
        - `/reasoning`، `/verbose` و `/trace` در محیط‌های گروهی پرخطرند — ممکن است استدلال داخلی یا اطلاعات تشخیصی Plugin را آشکار کنند. آن‌ها را در گفت‌وگوهای گروهی خاموش نگه دارید.

      </Accordion>
      <Accordion title="جزئیات تعویض مدل">
        - `/model` مدل جدید را بلافاصله در نشست پایدار می‌کند.
        - اگر عامل بی‌کار باشد، اجرای بعدی فوراً از آن استفاده می‌کند.
        - اگر اجرایی فعال باشد، تعویض در حالت انتظار علامت‌گذاری می‌شود و در نقطهٔ پاک بعدی برای تلاش مجدد اعمال می‌شود.

      </Accordion>
    </AccordionGroup>

  </Accordion>

  <Accordion title="کشف و وضعیت">
    | فرمان | توضیحات |
    | --- | --- |
    | `/help` | خلاصهٔ کوتاه راهنما را نمایش می‌دهد |
    | `/commands` | کاتالوگ فرمان‌های تولیدشده را نمایش می‌دهد |
    | `/tools [compact\|verbose]` | آنچه عامل فعلی همین حالا می‌تواند استفاده کند نمایش می‌دهد |
    | `/status` | وضعیت اجرا/محیط اجرا، زمان کارکرد Gateway و سیستم، سلامت Plugin، به‌علاوهٔ مصرف/سهمیهٔ ارائه‌دهنده را نمایش می‌دهد |
    | `/status plugins` | جزئیات سلامت Plugin را نمایش می‌دهد: خطاهای بارگذاری، قرنطینه‌ها، خرابی‌های Plugin کانال، مشکلات وابستگی و اعلان‌های سازگاری. به `commands.plugins: true` نیاز دارد |
    | `/goal [status\|start\|edit\|pause\|resume\|complete\|block\|clear] ...` | [هدف](/fa/tools/goal) پایدار نشست فعلی را مدیریت می‌کند |
    | `/diagnostics [note]` | جریان گزارش پشتیبانی مختص مالک. هر بار تأیید اجرا را درخواست می‌کند |
    | `/openclaw <request>` | دستیار راه‌اندازی و تعمیر OpenClaw را از یک پیام خصوصی مالک اجرا می‌کند |
    | `/tasks` | وظایف پس‌زمینهٔ فعال/اخیر نشست فعلی را فهرست می‌کند |
    | `/context [list\|detail\|map\|json]` | نحوهٔ گردآوری زمینه را توضیح می‌دهد |
    | `/whoami` | شناسهٔ فرستندهٔ شما را نمایش می‌دهد. نام مستعار: `/id` |
    | `/usage off\|tokens\|full\|reset\|cost` | پاورقی مصرف هر پاسخ را کنترل می‌کند (`reset`/`inherit`/`clear`/`default` بازنویسی نشست را پاک می‌کند تا پیش‌فرض پیکربندی‌شده دوباره به ارث برسد) یا خلاصهٔ هزینهٔ محلی را چاپ می‌کند |
  </Accordion>

  <Accordion title="Skills، فهرست‌های مجاز، تأییدها">
    | فرمان | توضیحات |
    | --- | --- |
    | `/skill <name> [input]` | اجرای یک skill با نام |
    | `/learn [request]` | پیش‌نویس یک skill قابل‌بازبینی از گفت‌وگوی جاری یا منابع نام‌برده‌شده از طریق [کارگاه Skill](/fa/tools/skill-workshop) |
    | `/allowlist [list\|add\|remove] ...` | مدیریت ورودی‌های فهرست مجاز. فقط متنی |
    | `/approve <id> <decision>` | رسیدگی به درخواست‌های تأیید exec یا plugin |
    | `/btw <question>` | پرسیدن یک پرسش جانبی بدون تغییر زمینهٔ نشست. نام مستعار: `/side`. [BTW](/fa/tools/btw) را ببینید |
  </Accordion>

  <Accordion title="زیرعامل‌ها و ACP">
    | فرمان | توضیحات |
    | --- | --- |
    | `/subagents list\|log\|info` | بررسی اجراهای زیرعامل برای نشست جاری |
    | `/acp spawn\|cancel\|steer\|close\|sessions\|status\|set-mode\|set\|cwd\|permissions\|timeout\|model\|reset-options\|doctor\|install\|help` | مدیریت نشست‌های ACP و گزینه‌های زمان اجرا. کنترل‌های زمان اجرا به هویت مالک خارجی یا مدیر داخلی Gateway نیاز دارند |
    | `/focus <target>` | متصل‌کردن رشتهٔ جاری Discord یا موضوع Telegram به یک مقصد نشست |
    | `/unfocus` | حذف اتصال رشتهٔ جاری |
    | `/agents` | فهرست‌کردن عامل‌های متصل به رشته برای نشست جاری |
  </Accordion>

  <Accordion title="نوشتن و مدیریت ویژهٔ مالک">
    | فرمان | نیازمند | توضیحات |
    | --- | --- | --- |
    | `/config show\|get\|set\|unset` | `commands.config: true` | خواندن یا نوشتن `openclaw.json`. فقط مالک |
    | `/mcp show\|get\|set\|unset` | `commands.mcp: true` | خواندن یا نوشتن پیکربندی سرور MCP تحت مدیریت OpenClaw. فقط مالک |
    | `/plugins list\|inspect\|show\|get\|install\|enable\|disable` | `commands.plugins: true` | بررسی یا تغییر وضعیت plugin. نوشتن فقط برای مالک. نام مستعار: `/plugin` |
    | `/debug show\|set\|unset\|reset` | `commands.debug: true` | بازنویسی‌های پیکربندی فقط برای زمان اجرا. فقط مالک |
    | `/restart` | `commands.restart: true` (پیش‌فرض) | راه‌اندازی مجدد OpenClaw |
    | `/send on\|off\|inherit` | مالک | تنظیم خط‌مشی ارسال |
  </Accordion>

  <Accordion title="صدا، TTS، کنترل کانال">
    | فرمان | توضیحات |
    | --- | --- |
    | `/tts on\|off\|status\|chat\|latest\|provider\|limit\|summary\|audio\|help` | کنترل TTS. [TTS](/fa/tools/tts) را ببینید |
    | `/activation mention\|always` | تنظیم حالت فعال‌سازی گروه |
    | `/bash <command>` | اجرای فرمان پوسته روی میزبان. نام مستعار: `! <command>`. نیازمند `commands.bash: true` |
    | `!poll [sessionId]` | بررسی یک کار پس‌زمینهٔ bash |
    | `!stop [sessionId]` | توقف یک کار پس‌زمینهٔ bash |
  </Accordion>
</AccordionGroup>

### فرمان‌های اتصال

فرمان‌های اتصال، مسیر پاسخ نشست فعال را به کانال پیوندخوردهٔ دیگری تغییر می‌دهند.
برای راه‌اندازی و عیب‌یابی، [اتصال کانال](/fa/concepts/channel-docking) را ببینید.

تولیدشده از pluginهای کانال دارای پشتیبانی فرمان بومی:

- `/dock-discord` (نام مستعار: `/dock_discord`)
- `/dock-mattermost` (نام مستعار: `/dock_mattermost`)
- `/dock-slack` (نام مستعار: `/dock_slack`)
- `/dock-telegram` (نام مستعار: `/dock_telegram`)

فرمان‌های اتصال به `session.identityLinks` نیاز دارند. فرستندهٔ مبدأ و همتای مقصد
باید در یک گروه هویتی باشند.

### فرمان‌های plugin همراه

| فرمان                                                 | توضیحات                                                                                                                                                                                    |
| ------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `/dreaming [on\|off\|status\|help]`                     | تغییر وضعیت Dreaming حافظه (مالک یا مدیر Gateway). [Dreaming](/fa/concepts/dreaming) را ببینید                                                                                                            |
| `/pair [qr\|status\|pending\|approve\|cleanup\|notify]` | مدیریت جفت‌سازی دستگاه. [جفت‌سازی](/fa/channels/pairing) را ببینید                                                                                                                                        |
| `/phone status\|arm ...\|disarm`                        | فعال‌سازی موقت فرمان‌های پرخطر Node (دوربین/صفحه‌نمایش/رایانه/نوشتن). [استفاده از رایانه](/fa/nodes/computer-use) را ببینید                                                                               |
| `/voice status\|list\|set <voiceId>`                    | مدیریت پیکربندی صدای Talk. نام بومی Discord: `/talkvoice`                                                                                                                                    |
| `/card ...`                                             | ارسال پیش‌تنظیم‌های کارت غنی LINE. [LINE](/fa/channels/line) را ببینید                                                                                                                                        |
| `/codex <action> ...`                                   | اتصال، هدایت و بررسی مهار app-server مربوط به Codex (وضعیت، رشته‌ها، ازسرگیری، مدل، حالت سریع، مجوزها، فشرده‌سازی، بازبینی، mcp، skillها و موارد دیگر). [مهار Codex](/fa/plugins/codex-harness) را ببینید |

فقط QQBot: `/bot-ping`، `/bot-version`، `/bot-help`، `/bot-upgrade`، `/bot-logs`

### فرمان‌های Skill

skillهای قابل‌فراخوانی توسط کاربر به‌صورت فرمان‌های اسلش ارائه می‌شوند:

- `/skill <name> [input]` همیشه به‌عنوان نقطهٔ ورود عمومی کار می‌کند.
- Skills می‌توانند به‌عنوان فرمان‌های مستقیم ثبت شوند (برای مثال `/prose` برای OpenProse).
- ثبت بومی فرمان skill توسط `commands.nativeSkills` و
  `channels.<provider>.commands.nativeSkills` کنترل می‌شود.
- نام‌ها به `a-z0-9_` پاک‌سازی می‌شوند (حداکثر 32 نویسه)؛ تداخل‌ها پسوند عددی می‌گیرند.

<AccordionGroup>
  <Accordion title="ارسال فرمان Skill">
    به‌طور پیش‌فرض، فرمان‌های skill مانند یک درخواست عادی به مدل هدایت می‌شوند.

    Skills می‌توانند `command-dispatch: tool` را تعریف کنند تا مستقیماً به یک ابزار هدایت شوند
    (قطعی و بدون دخالت مدل). نمونه: `/prose` (plugin مربوط به OpenProse)
    — [OpenProse](/fa/prose) را ببینید.

  </Accordion>
  <Accordion title="آرگومان‌های فرمان بومی">
    هنگامی که آرگومان‌های الزامی وارد نشده باشند، Discord برای گزینه‌های پویا از تکمیل خودکار و منوهای دکمه‌ای استفاده می‌کند.
    Telegram و Slack برای فرمان‌های دارای گزینه، یک منوی دکمه‌ای نمایش می‌دهند.
    گزینه‌های پویا بر اساس مدل نشست مقصد تعیین می‌شوند؛ بنابراین گزینه‌های ویژهٔ
    مدل مانند سطوح `/think` از بازنویسی `/model` نشست پیروی می‌کنند.
  </Accordion>
</AccordionGroup>

## `/tools`: عامل اکنون از چه چیزهایی می‌تواند استفاده کند

`/tools` به یک پرسش زمان اجرا پاسخ می‌دهد: **این عامل همین حالا در این
گفت‌وگو از چه چیزهایی می‌تواند استفاده کند** — نه یک کاتالوگ ایستای پیکربندی.

```text
/tools         # نمای فشرده
/tools verbose # همراه با توضیحات کوتاه
```

نتایج محدود به نشست هستند. تغییر عامل، کانال، رشته، مجوز
فرستنده یا مدل می‌تواند خروجی را تغییر دهد. برای ویرایش نمایه و بازنویسی‌ها،
از پنل Tools در Control UI یا سطوح پیکربندی استفاده کنید.

## `/model`: انتخاب مدل

```text
/model             # نمایش انتخابگر مدل
/model list        # همان مورد
/model 3           # انتخاب با شماره از انتخابگر
/model openai/gpt-5.4
/model opus@anthropic:default
/model default     # پاک‌کردن انتخاب مدل نشست
/model status      # نمای تفصیلی همراه با نقطهٔ پایانی و حالت API
```

در Discord، `/model` و `/models` یک انتخابگر تعاملی با فهرست‌های کشویی ارائه‌دهنده و
مدل باز می‌کنند. انتخابگر، `agents.defaults.modelPolicy.allow` را رعایت می‌کند،
از جمله ورودی‌های `provider/*`. بدون فهرست مجاز صریح، ورودی‌های مدل و
نام‌های مستعار انتخاب را محدود نمی‌کنند.

## `/config`: نوشتن پیکربندی روی دیسک

<Note>
  فقط مالک. به‌طور پیش‌فرض غیرفعال است — با `commands.config: true` فعال کنید.
</Note>

```text
/config show
/config show channels.whatsapp.responsePrefix
/config get channels.whatsapp.responsePrefix
/config set channels.whatsapp.responsePrefix="[openclaw]"
/config unset channels.whatsapp.responsePrefix
```

پیکربندی پیش از نوشتن اعتبارسنجی می‌شود. تغییرات نامعتبر رد می‌شوند. به‌روزرسانی‌های `/config`
پس از راه‌اندازی مجدد نیز باقی می‌مانند.

## `/mcp`: پیکربندی سرور MCP

<Note>
  فقط مالک. به‌طور پیش‌فرض غیرفعال است — با `commands.mcp: true` فعال کنید.
</Note>

```text
/mcp show
/mcp show context7
/mcp set context7={"command":"uvx","args":["context7-mcp"]}
/mcp unset context7
```

`/mcp` پیکربندی را در پیکربندی OpenClaw ذخیره می‌کند، نه در تنظیمات پروژهٔ عامل تعبیه‌شده.
`/mcp show` فیلدهای حاوی اطلاعات اعتبارنامه، مقادیر شناخته‌شدهٔ پرچم‌های اعتبارنامه
و آرگومان‌هایی با الگوی شناخته‌شدهٔ اطلاعات محرمانه را پنهان می‌کند. هنگام اجرا از یک گروه،
پیکربندی به‌صورت خصوصی برای مالک ارسال می‌شود؛ اگر مسیر خصوصی برای مالک
در دسترس نباشد، فرمان به‌شکل بسته شکست می‌خورد و از مالک می‌خواهد از یک
گفت‌وگوی مستقیم دوباره تلاش کند.

## `/debug`: بازنویسی‌های فقط زمان اجرا

<Note>
  فقط مالک. به‌طور پیش‌فرض غیرفعال است — با `commands.debug: true` فعال کنید.
  بازنویسی‌ها فوراً روی خواندن‌های جدید پیکربندی اعمال می‌شوند، اما روی دیسک **نوشته نمی‌شوند**.
</Note>

```text
/debug show
/debug set channels.whatsapp.responsePrefix="[openclaw]"
/debug set channels.whatsapp.allowFrom=["+1555","+4477"]
/debug unset channels.whatsapp.responsePrefix
/debug reset
```

## `/plugins`: مدیریت plugin

<Note>
  نوشتن فقط برای مالک. به‌طور پیش‌فرض غیرفعال است — با `commands.plugins: true` فعال کنید.
</Note>

```text
/plugins
/plugins list
/plugin show context7
/plugins enable context7
/plugins disable context7
/plugins install clawhub:<package>
/plugins install npm:@openclaw/<official-package>
/plugins install npm:<package> --force
/plugins install git:<repository>@<ref> --force
```

`/plugins enable|disable` پیکربندی plugin را به‌روزرسانی می‌کند و زمان اجرای
plugin در Gateway را برای نوبت‌های جدید عامل بدون توقف بارگذاری مجدد می‌کند. `/plugins install`، Gatewayهای مدیریت‌شده را
به‌طور خودکار راه‌اندازی مجدد می‌کند، زیرا ماژول‌های منبع plugin تغییر کرده‌اند. نصب‌های معتبر ClawHub
و کاتالوگ رسمی به تأیید اضافی نیاز ندارند. منابع دلخواه npm،
git، بایگانی، `npm-pack:` و مسیر محلی، هشدار منشأ نمایش می‌دهند و
پس از بازبینی منبع، به `--force` پایانی نیاز دارند. این پرچم منبع را تأیید
و جایگزینی نصب موجود را مجاز می‌کند؛ اما `security.installPolicy` یا بررسی‌های امنیتی
نصب‌کننده را دور نمی‌زند. نسخه‌های ClawHub دارای
هشدارهای خطر همچنان به پرچم جداگانهٔ فقط پوستهٔ
`--acknowledge-clawhub-risk` نیاز دارند. نصب‌های فروشگاهی، پیوندی و سنجاق‌شده نیز
همچنان فقط از طریق پوسته ممکن هستند.

## `/trace`: خروجی ردیابی plugin

```text
/trace          # نمایش وضعیت فعلی ردیابی
/trace on
/trace off
```

`/trace` خطوط ردیابی/اشکال‌زدایی plugin محدود به نشست را بدون حالت verbose کامل
نمایان می‌کند. این مورد جایگزین `/debug` (بازنویسی‌های زمان اجرا) یا `/verbose` (خروجی عادی
ابزار) نیست.

## `/btw`: پرسش‌های جانبی

`/btw` یک پرسش جانبی سریع دربارهٔ زمینهٔ نشست جاری است. نام مستعار: `/side`.

```text
/btw همین حالا چه کاری انجام می‌دهیم؟
/side هنگام ادامهٔ اجرای اصلی چه چیزی تغییر کرد؟
```

برخلاف یک پیام عادی:

- از نشست جاری به‌عنوان زمینهٔ پس‌زمینه استفاده می‌کند.
- در نشست‌های مهار Codex، به‌صورت یک رشتهٔ جانبی موقت Codex اجرا می‌شود.
- زمینهٔ نشست آینده را تغییر **نمی‌دهد**.
- در تاریخچهٔ رونوشت نوشته نمی‌شود.

برای رفتار کامل، [پرسش‌های جانبی BTW](/fa/tools/btw) را ببینید.

## نکات سطوح

<AccordionGroup>
  <Accordion title="محدوده‌بندی نشست در هر سطح">
    - **فرمان‌های متنی:** در نشست عادی گفت‌وگو اجرا می‌شوند (پیام‌های مستقیم `main` را به‌اشتراک می‌گذارند، گروه‌ها نشست خود را دارند).
    - **فرمان‌های بومی Discord:** `agent:<agentId>:discord:slash:<userId>`
    - **فرمان‌های بومی Slack:** `agent:<agentId>:slack:slash:<userId>` (پیشوند از طریق `channels.slack.slashCommand.sessionPrefix` قابل‌پیکربندی است)
    - **فرمان‌های بومی Telegram:** `telegram:slash:<userId>` (نشست گفت‌وگو را از طریق `CommandTargetSessionKey` هدف قرار می‌دهد)
    - **`/login codex`** کدهای جفت‌سازی دستگاه را فقط از طریق گفت‌وگوی خصوصی یا مسیرهای پاسخ Web UI ارسال می‌کند. فراخوانی‌ها در گروه/موضوع Telegram از مالک می‌خواهند به ربات پیام مستقیم بدهد.
    - **`/stop`** نشست گفت‌وگوی فعال را برای لغو اجرای جاری هدف قرار می‌دهد.

  </Accordion>
  <Accordion title="جزئیات Slack">
    `channels.slack.slashCommand` از یک فرمان به سبک `/openclaw` پشتیبانی می‌کند.
    با `commands.native: true`، برای هر فرمان داخلی یک فرمان اسلش Slack ایجاد کنید.
    `/agentstatus` را ثبت کنید (نه `/status`)، زیرا Slack
    `/status` را رزرو کرده است. متن `/status` همچنان در پیام‌های Slack کار می‌کند.
  </Accordion>
  <Accordion title="مسیر سریع و میان‌برهای درون‌خطی">
    - پیام‌هایی که فقط شامل فرمان هستند و از فرستندگان فهرست مجاز ارسال می‌شوند، بلافاصله پردازش می‌شوند (صف + مدل را دور می‌زنند).
    - میان‌برهای درون‌خطی (`/help`، `/commands`، `/status`، `/whoami`) درون پیام‌های عادی نیز کار می‌کنند و پیش از آنکه مدل متن باقی‌مانده را ببیند، حذف می‌شوند.
    - پیام‌های غیرمجازِ صرفاً حاوی فرمان، بی‌سروصدا نادیده گرفته می‌شوند؛ توکن‌های درون‌خطی `/...` به‌عنوان متن ساده در نظر گرفته می‌شوند.

  </Accordion>
  <Accordion title="نکته‌های آرگومان‌ها">
    - فرمان‌ها یک `:` اختیاری را بین فرمان و آرگومان‌ها می‌پذیرند (`/think: high`، `/send: on`).
    - `/new <model>` یک نام مستعار مدل، `provider/model` یا نام ارائه‌دهنده را می‌پذیرد (تطبیق تقریبی)؛ اگر تطبیقی پیدا نشود، متن به‌عنوان بدنهٔ پیام در نظر گرفته می‌شود.
    - `/allowlist add|remove` به `commands.config: true` نیاز دارد و `configWrites` کانال را رعایت می‌کند.

  </Accordion>
</AccordionGroup>

## میزان استفاده و وضعیت ارائه‌دهنده

- **میزان استفاده/سهمیهٔ ارائه‌دهنده** (برای مثال، «Claude 80% باقی‌مانده») هنگامی که ردیابی میزان استفاده فعال باشد، در `/status` برای ارائه‌دهندهٔ مدل فعلی نمایش داده می‌شود.
- **خطوط توکن/کش** در `/status`، هنگامی که عکس فوری نشست زنده اطلاعات کمی داشته باشد، می‌توانند از آخرین ورودی میزان استفاده در رونوشت استفاده کنند.
- **اجرا در برابر محیط اجرا:** `/status`، `Execution` را برای مسیر مؤثر محیط ایزوله و `Runtime` را برای مشخص‌کردن اجراکنندهٔ نشست گزارش می‌کند: `OpenClaw Default`، `OpenAI Codex`، یک بک‌اند CLI یا یک بک‌اند ACP.
- **توکن‌ها/هزینهٔ هر پاسخ:** با `/usage off|tokens|full` کنترل می‌شود.
- `/model status` مربوط به مدل‌ها/احراز هویت/نقاط پایانی است، نه میزان استفاده.

## مرتبط

<CardGroup cols={2}>
  <Card title="Skills" href="/fa/tools/skills" icon="puzzle-piece">
    نحوهٔ ثبت و محدودسازی فرمان‌های اسلش مهارت‌ها.
  </Card>
  <Card title="ایجاد مهارت‌ها" href="/fa/tools/creating-skills" icon="hammer">
    مهارتی بسازید که فرمان اسلش خودش را ثبت کند.
  </Card>
  <Card title="BTW" href="/fa/tools/btw" icon="comments">
    پرسش‌های جانبی بدون تغییر زمینهٔ نشست.
  </Card>
  <Card title="هدایت" href="/fa/tools/steer" icon="compass">
    عامل را در میانهٔ اجرا با `/steer` هدایت کنید.
  </Card>
</CardGroup>
