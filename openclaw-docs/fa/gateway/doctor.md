---
read_when:
    - افزودن یا تغییر مهاجرت‌های doctor
    - معرفی تغییرات ناسازگار در پیکربندی
sidebarTitle: Doctor
summary: 'دستور Doctor: بررسی‌های سلامت، مهاجرت‌های پیکربندی و مراحل تعمیر'
title: دکتر
x-i18n:
    generated_at: "2026-07-27T16:32:17Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3f599553a2455759cd0fe56bafbc16948f7ab4d381d344b08a496bf19c9dc636
    source_path: gateway/doctor.md
    workflow: 16
---

`openclaw doctor` ابزار تعمیر و مهاجرت OpenClaw است. این ابزار پیکربندی/وضعیت قدیمی را اصلاح می‌کند، سلامت را بررسی می‌کند و مراحل عملی تعمیر را ارائه می‌دهد.

## شروع سریع

```bash
openclaw doctor
```

### حالت‌های بدون رابط و خودکارسازی

<Tabs>
  <Tab title="--yes">
    ```bash
    openclaw doctor --yes
    ```

    پذیرش پیش‌فرض‌ها بدون درخواست تأیید (از جمله مراحل راه‌اندازی مجدد/سرویس/تعمیر سندباکس، در صورت کاربرد).

  </Tab>
  <Tab title="--fix">
    ```bash
    openclaw doctor --fix
    ```

    اعمال تعمیرات توصیه‌شده بدون درخواست تأیید (`--repair` نام مستعار آن است).

  </Tab>
  <Tab title="--lint">
    ```bash
    openclaw doctor --lint
    openclaw doctor --lint --json
    ```

    اجرای بررسی‌های ساخت‌یافته سلامت برای CI یا خودکارسازی پیش‌بررسی. فقط‌خواندنی: بدون
    درخواست تأیید، تعمیر، مهاجرت، راه‌اندازی مجدد یا نوشتن وضعیت.

  </Tab>
  <Tab title="--fix --force">
    ```bash
    openclaw doctor --fix --force
    ```

    اعمال تعمیرات تهاجمی نیز (پیکربندی‌های سفارشی سرپرست را بازنویسی می‌کند).

  </Tab>
  <Tab title="--non-interactive">
    ```bash
    openclaw doctor --non-interactive
    ```

    اجرا بدون درخواست تأیید و اعمال فقط مهاجرت‌های ایمن (نرمال‌سازی پیکربندی +
    جابه‌جایی وضعیت روی دیسک). اقدامات راه‌اندازی مجدد/سرویس/سندباکس را که به تأیید
    انسانی نیاز دارند، نادیده می‌گیرد. مهاجرت‌های وضعیت قدیمی همچنان پس از شناسایی به‌طور خودکار اجرا می‌شوند.

  </Tab>
  <Tab title="--deep">
    ```bash
    openclaw doctor --deep
    ```

    اسکن سرویس‌های سیستم برای نصب‌های اضافی Gateway ‏(launchd/systemd/schtasks).

  </Tab>
</Tabs>

برای بازبینی تغییرات پیش از نوشتن، ابتدا فایل پیکربندی را باز کنید:

```bash
cat ~/.openclaw/openclaw.json
```

## حالت لینت فقط‌خواندنی

`openclaw doctor --lint` همتای مناسب خودکارسازیِ
`openclaw doctor --fix` است. هر دو از یک رجیستری قواعد Doctor استفاده می‌کنند، اما
قواعد را به یک شیوه انتخاب یا اجرا نمی‌کنند:

| حالت                     | درخواست تأیید   | نوشتن پیکربندی/وضعیت     | خروجی                 | کاربرد                      |
| ------------------------ | --------- | ----------------------- | ---------------------- | ------------------------------- |
| `openclaw doctor`        | بله       | خیر                      | گزارش سلامت خوانا | بررسی وضعیت توسط انسان         |
| `openclaw doctor --fix`  | گاهی | بله، با سیاست تعمیر | گزارش خوانای تعمیر    | اعمال تعمیرات تأییدشده       |
| `openclaw doctor --lint` | خیر        | خیر                      | یافته‌های ساخت‌یافته    | CI، پیش‌بررسی و دروازه‌های بازبینی |

اجرای پیش‌فرض `doctor --lint` از پروفایل گسترده و ایمن خودکارسازی استفاده می‌کند: بررسی‌هایی که
ایستا، محلی و در خروجی CI یا پیش‌بررسی مفیدند. بررسی‌های اختیاریِ
توصیه‌ای، حساس به محیط، وابسته به سرویس زنده، موجودی حساب/فضای کاری
یا پاک‌سازی تاریخی را نادیده می‌گیرد. برای ممیزی کامل لینت ثبت‌شده، شامل
این بررسی‌های اختیاری، از `doctor --lint --all` یا برای یک بررسی هدفمند از `--only <id>` استفاده کنید.

`doctor --fix` از پروفایل پیش‌فرض لینت استفاده نمی‌کند و
`--all` را نمی‌پذیرد. این فرمان مسیر مرتب‌شده تعمیر Doctor را اجرا می‌کند: بررسی‌های سلامت مدرن ممکن است
پیاده‌سازی اختیاری `repair()` را ارائه دهند و بخش‌های قدیمی‌تر همچنان از جریان
تعمیر قدیمی Doctor استفاده می‌کنند. برخی یافته‌های لینت عمداً فقط تشخیصی هستند؛ بنابراین ظاهرشدن
یک بررسی در `--lint --all` به این معنا نیست که `--fix` آن بخش را تغییر می‌دهد.
این قرارداد `detect()` (گزارش یافته‌ها) را از `repair()` (گزارش
تغییرات/تفاوت‌ها/اثرات جانبی) جدا می‌کند و بدون تبدیل بررسی‌های لینت به برنامه‌ریز تغییرات،
مسیر را برای `doctor --fix --dry-run` احتمالی در آینده باز نگه می‌دارد.

برخی بررسی‌های داخلی به‌طور پیش‌فرض در داخل غیرفعال‌اند تا برای
`--all`، `--only` و جریان‌های تعمیر Doctor در دسترس بمانند، بدون آنکه بخشی از پروفایل پیش‌فرض
خودکارسازی `doctor --lint` شوند. شدت هر یافته همچنان به‌صورت جداگانه منتشر می‌شود
(`info`، `warning` یا `error`)؛ انتخاب پیش‌فرض یک سطح شدت نیست.

```bash
openclaw doctor --lint
openclaw doctor --lint --severity-min warning
openclaw doctor --lint --json
openclaw doctor --lint --all
openclaw doctor --lint --only core/doctor/gateway-config --json
```

فیلدهای خروجی JSON:

- `ok`: اینکه آیا یافته‌ای به آستانه شدت انتخاب‌شده رسیده است
- `checksRun` / `checksSkipped`: تعدادها (نادیده‌گرفته‌شده به‌دلیل پروفایل، `--only` یا `--skip`)
- `findings`: تشخیص‌های ساخت‌یافته با `checkId`، `severity`، `message` و موارد اختیاری `path`، `line`، `column`، `ocPath`، `source`، `target`، `requirement`، `fixHint`

کدهای خروج:

| کد | معنا                                                  |
| ---- | -------------------------------------------------------- |
| `0`  | هیچ یافته‌ای در آستانه انتخاب‌شده یا بالاتر از آن وجود ندارد           |
| `1`  | یک یا چند یافته به آستانه انتخاب‌شده رسیده‌اند          |
| `2`  | شکست فرمان/زمان اجرا پیش از امکان انتشار یافته‌ها |

پرچم‌ها:

- `--severity-min info|warning|error` (پیش‌فرض `warning`): هم موارد چاپ‌شده و هم عوامل ایجاد خروج غیرصفر را کنترل می‌کند.
- `--all`: همه بررسی‌های لینت ثبت‌شده، از جمله بررسی‌های اختیاری خارج‌شده از مجموعه پیش‌فرض خودکارسازی را اجرا می‌کند.
- `--only <id>` (تکرارپذیر): فقط شناسه‌های بررسی نام‌برده را اجرا می‌کند؛ شناسه ناشناخته به‌صورت یافته خطا گزارش می‌شود.
- `--skip <id>` (تکرارپذیر): یک بررسی را در حالی کنار می‌گذارد که بقیه اجرا فعال می‌ماند.
- `--json`، `--severity-min`، `--all`، `--only` و `--skip` به `--lint` نیاز دارند؛ اجراهای ساده `openclaw doctor` و `--fix` آن‌ها را رد می‌کنند.

## کارکردها (خلاصه)

<AccordionGroup>
  <Accordion title="سلامت، رابط کاربری و به‌روزرسانی‌ها">
    - پیش‌به‌روزرسانی اختیاری برای نصب‌های git (فقط تعاملی).
    - بررسی تازگی پروتکل رابط کاربری (وقتی شِمای پروتکل جدیدتر باشد، Control UI را دوباره می‌سازد).
    - بررسی سلامت + درخواست راه‌اندازی مجدد.
    - یادداشت‌های Skills و Plugin فقط برای مشکلات؛ موجودی سالم در `openclaw skills check` و `openclaw plugins list` باقی می‌ماند.

  </Accordion>
  <Accordion title="پیکربندی و مهاجرت‌ها">
    - نرمال‌سازی پیکربندی برای شکل‌های قدیمی مقادیر.
    - مهاجرت پیکربندی گفتگو از فیلدهای مسطح قدیمی `talk.*` به `talk.provider` + `talk.providers.<provider>`.
    - بررسی‌های مهاجرت مرورگر برای پیکربندی‌های قدیمی افزونه Chrome و آمادگی Chrome MCP.
    - هشدارهای بازنویسی ارائه‌دهنده OpenCode ‏(`models.providers.opencode` / `opencode-zen` / `opencode-go`).
    - مهاجرت ارائه‌دهنده/پروفایل قدیمی OpenAI Codex ‏(`openai-codex` → `openai`) و هشدارهای تحت‌الشعاع قرارگرفتن برای `models.providers.openai-codex` قدیمی.
    - بررسی پیش‌نیازهای TLS مربوط به OAuth برای پروفایل‌های OAuth در OpenAI Codex.
    - هشدارهای فهرست مجاز Plugin/ابزار، هنگامی که `plugins.allow` محدودکننده است اما سیاست ابزار همچنان ابزارهای دارای نویسه عام یا متعلق به Plugin را درخواست می‌کند.
    - مهاجرت وضعیت قدیمی روی دیسک (نشست‌ها/دایرکتوری عامل/احراز هویت WhatsApp).
    - مهاجرت کلیدهای قدیمی قرارداد مانیفست Plugin ‏(`speechProviders`، `realtimeTranscriptionProviders`، `realtimeVoiceProviders`، `mediaUnderstandingProviders`، `imageGenerationProviders`، `videoGenerationProviders`، `webFetchProviders`، `webSearchProviders` → `contracts`).
    - مهاجرت مخزن قدیمی Cron ‏(`jobId`، `schedule.cron`، فیلدهای سطح‌بالای تحویل/بار داده، `provider` بار داده، کارهای Webhook جایگزین `notify: true`).
    - تعمیر پین زمان اجرای Codex CLI ‏(`agentRuntime.id: "codex-cli"` → `"codex"`) در `agents.defaults`، `agents.entries.*` و `models.providers.*` (شامل ورودی‌های هر مدل).
    - پاک‌سازی پیکربندی قدیمی Plugin هنگامی که Pluginها فعال‌اند؛ در حالت `plugins.enabled=false`، ارجاعات قدیمی Plugin به‌صورت پیکربندی مهار غیرفعال حفظ می‌شوند.

  </Accordion>
  <Accordion title="وضعیت و یکپارچگی">
    - بازرسی فایل قفل نشست و پاک‌سازی قفل‌های قدیمی.
    - تعمیر رونوشت نشست برای شاخه‌های تکراری بازنویسی پرامپت که توسط بیلدهای تحت‌تأثیر 2026.4.24 ایجاد شده‌اند.
    - شناسایی سنگ‌قبر بازیابی پس از راه‌اندازی مجدد برای نشست اصلی و زیرعامل‌های گیرکرده. Doctor نشست‌های مسدود را گزارش می‌کند و فقط پرچم‌های قدیمی لغوشده‌ای را تعمیر می‌کند که با سنگ‌قبر موجود در تعارض‌اند؛ بازیابی خودکار را دوباره فعال نمی‌کند.
    - بررسی‌های یکپارچگی وضعیت و مجوزها (نشست‌ها، رونوشت‌ها، دایرکتوری وضعیت).
    - بررسی مجوزهای فایل پیکربندی (chmod 600) هنگام اجرای محلی.
    - سلامت احراز هویت مدل: انقضای OAuth را بررسی می‌کند، می‌تواند توکن‌های نزدیک به انقضا را تازه‌سازی کند و وضعیت‌های دوره انتظار/غیرفعال پروفایل احراز هویت را گزارش می‌کند.

  </Accordion>
  <Accordion title="Gateway، سرویس‌ها و سرپرست‌ها">
    - تعمیر تصویر سندباکس هنگامی که سندباکس فعال است.
    - مهاجرت سرویس قدیمی و شناسایی Gateway اضافی.
    - مهاجرت وضعیت قدیمی کانال Matrix (در حالت `--fix` / `--repair`).
    - بررسی‌های زمان اجرای Gateway (سرویس نصب شده اما در حال اجرا نیست؛ برچسب launchd ذخیره‌شده در کش).
    - هشدارهای وضعیت کانال (بررسی‌شده از Gateway در حال اجرا).
    - بررسی‌های مجوز مختص کانال در `openclaw channels capabilities` قرار دارند؛ برای مثال، مجوزهای کانال صوتی Discord با `openclaw channels capabilities --channel discord --target channel:<channel-id>` ممیزی می‌شوند.
    - بررسی‌های پاسخ‌گویی WhatsApp برای افت سلامت حلقه رویداد Gateway در حالی که کلاینت‌های محلی TUI همچنان در حال اجرا هستند؛ `--fix` فقط کلاینت‌های محلی TUI تأییدشده را متوقف می‌کند.
    - تعمیر مسیر Codex برای ارجاعات قدیمی مدل `openai-codex/*` در مدل‌های اصلی، جایگزین‌ها، مدل‌های تولید تصویر/ویدئو، بازنویسی‌های Heartbeat/زیرعامل/Compaction، هوک‌ها، بازنویسی مدل کانال و پین‌های مسیر نشست؛ `--fix` آن‌ها را به `openai/*` بازنویسی می‌کند، پروفایل‌ها/ترتیب احراز هویت `openai-codex:*` را به `openai:*` مهاجرت می‌دهد، پین‌های قدیمی زمان اجرای نشست/کل عامل را حذف می‌کند و به مسیر مؤثر تعمیرشده اجازه می‌دهد سازگاری Codex را تعیین کند.
    - ممیزی پیکربندی سرپرست (launchd/systemd/schtasks) با تعمیر اختیاری.
    - پاک‌سازی محیط پراکسی تعبیه‌شده برای سرویس‌های Gateway که هنگام نصب یا به‌روزرسانی مقادیر `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` پوسته را ثبت کرده‌اند.
    - بررسی‌های زمان اجرای Gateway (سرویس‌های قدیمی و پشتیبانی‌نشده Bun، مسیرهای مدیر نسخه).
    - تشخیص تداخل پورت Gateway (پیش‌فرض `18789`).

  </Accordion>
  <Accordion title="احراز هویت، امنیت و جفت‌سازی">
    - هشدارهای امنیتی برای سیاست‌های DM باز.
    - بررسی‌های احراز هویت Gateway برای حالت توکن محلی (وقتی هیچ منبع توکنی وجود ندارد، تولید توکن را پیشنهاد می‌دهد؛ پیکربندی‌های SecretRef توکن را بازنویسی نمی‌کند).
    - شناسایی مشکل جفت‌سازی دستگاه (درخواست‌های معلق جفت‌سازی بار نخست، ارتقای معلق نقش/دامنه، انحراف کش قدیمی توکن دستگاه محلی و انحراف احراز هویت رکورد جفت‌شده).

  </Accordion>
  <Accordion title="فضای کاری و پوسته">
    - بررسی linger در systemd روی Linux.
    - بررسی اندازه فایل راه‌اندازی فضای کاری (هشدارهای برش/نزدیک‌شدن به محدودیت برای فایل‌های زمینه).
    - بررسی آمادگی Skills برای عامل پیش‌فرض؛ Skills مجاز با باینری‌ها، محیط، پیکربندی یا نیازمندی‌های سیستم‌عامل مفقود را گزارش می‌کند و `--fix` می‌تواند Skills دردسترس‌نبودنی را در `skills.entries` غیرفعال کند.
    - بررسی وضعیت تکمیل خودکار پوسته و نصب/ارتقای خودکار.
    - بررسی آمادگی ارائه‌دهنده تعبیه‌سازی جست‌وجوی حافظه (مدل محلی، کلید API راه‌دور یا باینری QMD).
    - بررسی‌های نصب از منبع (ناهماهنگی فضای کاری pnpm، نبود دارایی‌های رابط کاربری، نبود باینری tsx).
    - نوشتن پیکربندی به‌روزشده + فراداده راهنما.

  </Accordion>
</AccordionGroup>

## بازپُرکردن و بازنشانی رابط کاربری رؤیاها

  صحنه Dreams در رابط کاربری کنترل شامل کنش‌های **Backfill**، **Reset** و **Clear Grounded** برای گردش‌کار Dreaming مبتنی بر داده‌های واقعی است. این کنش‌ها از متدهای RPC به سبک doctor در Gateway استفاده می‌کنند، اما بخشی از تعمیر/مهاجرت CLI در `openclaw doctor` **نیستند**.

  | کنش           | کاری که انجام می‌دهد                                                                                                                                                      |
  | -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | Backfill       | فایل‌های تاریخی `memory/YYYY-MM-DD.md` را در فضای کاری فعال پویش می‌کند، گذر دفترچه REM مبتنی بر داده‌های واقعی را اجرا می‌کند و ورودی‌های Backfill برگشت‌پذیر را در `DREAMS.md` می‌نویسد. |
  | Reset          | فقط ورودی‌های علامت‌گذاری‌شده دفترچه Backfill را از `DREAMS.md` حذف می‌کند.                                                                                                  |
  | Clear Grounded | فقط ورودی‌های کوتاه‌مدت مرحله‌بندی‌شده و مختص داده‌های واقعی را که از بازپخش تاریخی ایجاد شده‌اند و هنوز یادآوری زنده یا پشتیبانی روزانه انباشته نکرده‌اند، حذف می‌کند.                           |

  هیچ‌یک از این کنش‌ها `MEMORY.md` را ویرایش نمی‌کنند، مهاجرت‌های کامل doctor را اجرا نمی‌کنند یا به‌تنهایی نامزدهای مبتنی بر داده‌های واقعی را در مخزن زنده ارتقای کوتاه‌مدت مرحله‌بندی نمی‌کنند. برای واردکردن بازپخش تاریخی مبتنی بر داده‌های واقعی به مسیر عادی ارتقای عمیق، در عوض از جریان CLI استفاده کنید:

  ```bash
  openclaw memory rem-backfill --path ./memory --stage-short-term
  ```

  این فرمان نامزدهای ماندگار مبتنی بر داده‌های واقعی را در مخزن Dreaming کوتاه‌مدت مرحله‌بندی می‌کند، درحالی‌که `DREAMS.md` همچنان سطح بازبینی باقی می‌ماند.

  ## رفتار و منطق تفصیلی

  <AccordionGroup>
  <Accordion title="0. به‌روزرسانی اختیاری (نصب‌های git)">
    اگر این یک checkout از git باشد و doctor به‌صورت تعاملی اجرا شود، پیش از اجرای doctor پیشنهاد به‌روزرسانی (fetch/rebase/build) می‌دهد.
  </Accordion>
  <Accordion title="1. نرمال‌سازی پیکربندی">
    Doctor شکل‌های قدیمی مقادیر را به شِمای فعلی نرمال‌سازی می‌کند. پیکربندی فعلی گفتار Talk شامل `talk.provider` + `talk.providers.<provider>` است و پیکربندی صدای بلادرنگ در `talk.realtime.*` قرار دارد. Doctor شکل‌های قدیمی `talk.voiceId` / `talk.voiceAliases` / `talk.modelId` / `talk.outputFormat` / `talk.apiKey` را در نگاشت ارائه‌دهنده بازنویسی می‌کند و انتخاب‌گرهای بلادرنگ قدیمی سطح‌بالا (`talk.mode`، `talk.transport`، `talk.brain`، `talk.model`، `talk.voice`) را در `talk.realtime` بازنویسی می‌کند.

    Doctor همچنین هنگامی هشدار می‌دهد که `plugins.allow` خالی نباشد و سیاست ابزار از نویسه عام یا ورودی‌های ابزار متعلق به Plugin استفاده کند. `tools.allow: ["*"]` فقط با ابزارهای Pluginهایی مطابقت دارد که واقعاً بارگذاری می‌شوند؛ این مورد فهرست مجاز انحصاری Plugin را دور نمی‌زند.

  </Accordion>
  <Accordion title="2. مهاجرت کلیدهای پیکربندی قدیمی">
    وقتی پیکربندی شامل کلیدی منسوخ با مهاجرت فعال باشد، فرمان‌های دیگر از اجرا خودداری می‌کنند و از شما می‌خواهند `openclaw doctor` را اجرا کنید. Doctor توضیح می‌دهد کدام کلیدهای قدیمی پیدا شده‌اند، مهاجرت اعمال‌شده را نشان می‌دهد و `~/.openclaw/openclaw.json` را با شِمای به‌روزشده بازنویسی می‌کند. راه‌اندازی Gateway قالب‌های قدیمی پیکربندی را نمی‌پذیرد و از شما می‌خواهد `openclaw doctor --fix` را اجرا کنید؛ هنگام راه‌اندازی، `openclaw.json` را بازنویسی نمی‌کند. مهاجرت‌های مخزن کار Cron نیز توسط `openclaw doctor --fix` مدیریت می‌شوند.

    <Note>
      Doctor مهاجرت‌های خودکار را فقط تا حدود دو ماه پس از بازنشسته‌شدن یک
      کلید نگه می‌دارد. کلیدهای قدیمی‌تر (برای مثال
      `routing.queue`، `routing.bindings`، `routing.agents`/`defaultAgentId`،
      `routing.transcribeAudio`، `agent.*` سطح‌بالا یا `identity` سطح‌بالا
      از شکل پیکربندی پیش از چندعاملی) دیگر مسیر مهاجرتی ندارند؛
      پیکربندی‌ای که از آن‌ها استفاده کند، اکنون به‌جای بازنویسی در اعتبارسنجی
      شکست می‌خورد. پیش از آنکه doctor بتواند ادامه دهد، آن کلیدها را با مراجعه
      به مرجع فعلی پیکربندی به‌صورت دستی اصلاح کنید.
    </Note>

    مهاجرت‌های فعال:

    | کلید قدیمی                                                                                    | کلید فعلی                                                                 |
    | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------- |
    | `routing.allowFrom`                                                                              | `channels.whatsapp.allowFrom`                                                |
    | `routing.groupChat.requireMention`                                                               | `channels.whatsapp/telegram/imessage.groups."*".requireMention`             |
    | `routing.groupChat.historyLimit`                                                                 | `messages.groupChat.historyLimit`                                            |
    | `routing.groupChat.mentionPatterns`                                                              | `messages.groupChat.mentionPatterns`                                         |
    | `channels.telegram.requireMention`                                                               | `channels.telegram.groups."*".requireMention`                               |
    | `channels.webchat`، `gateway.webchat`                                                            | حذف‌شده (WebChat بازنشسته شده است)                                                 |
    | `channels.feishu.accounts.<accountId>.botName`                                                   | `channels.feishu.accounts.<accountId>.name`                                 |
    | `session.threadBindings.ttlHours`، `channels.<id>.threadBindings.ttlHours` (و برای هر حساب)      | `...threadBindings.idleHours`                                               |
    | `talk.voiceId`/`talk.voiceAliases`/`talk.modelId`/`talk.outputFormat`/`talk.apiKey` قدیمی        | `talk.provider` + `talk.providers.<provider>`                               |
    | انتخاب‌گرهای بلادرنگ Talk سطح‌بالای قدیمی (`talk.mode`/`talk.transport`/`talk.brain`/`talk.model`/`talk.voice`) | `talk.realtime`                                                              |
    | `messages.tts`                                                                                  | `tts` سطح‌بالا                                                              |
    | `messages.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)                             | `tts.providers.<provider>`                                                   |
    | `messages.tts.provider: "edge"` / `messages.tts.providers.edge`                                  | `tts.provider: "microsoft"` / `tts.providers.microsoft`                    |
    | `tools.exec.security` + `tools.exec.ask`                                                         | `tools.exec.mode`                                                            |
    | `session.idleMinutes`                                                                            | `session.reset.idleMinutes`                                                  |
    | `messages.responsePrefix` با بلوک‌های صریح کانال                                           | در `responsePrefix` کانال/حساب پیکربندی‌شده کپی می‌شود؛ بازگشت سراسری برای کانال‌های ضمنی/سفارشی حفظ می‌شود |
    | `web.enabled`                                                                                    | `channels.whatsapp.enabled`                                                  |
    | `meta.lastTouchedAt`، نصب‌های هوک، مخزن Cron، کشف بسته‌بندی‌شده، مسیر تنظیمات ترجیحی سراسری TTS            | وضعیت مشترک SQLite                                                       |
    | فیلدهای گوینده TTS ‏`voice`/`voiceName`/`voiceId`                                                 | `speakerVoice`/`speakerVoiceId`                                              |
    | `channels.<id>.tts.<provider>` / `channels.<id>.accounts.<accountId>.tts.<provider>` (همه کانال‌ها به‌جز Discord)                                          | `...tts.providers.<provider>`                                                |
    | `channels.<id>.voice.tts.<provider>` / `channels.<id>.accounts.<accountId>.voice.tts.<provider>` (همه کانال‌ها، ازجمله Discord)                          | `...voice.tts.providers.<provider>`                                          |
    | `plugins.entries.voice-call.config.tts.<provider>` (`openai`/`elevenlabs`/`microsoft`/`edge`)     | `plugins.entries.voice-call.config.tts.providers.<provider>`                |
    | `plugins.entries.voice-call.config.tts.provider: "edge"` / `...tts.providers.edge`                | `provider: "microsoft"` / `...tts.providers.microsoft`                      |
    | `plugins.entries.voice-call.config.provider: "log"`                                              | `"mock"`                                                                      |
    | `plugins.entries.voice-call.config.twilio.from`                                                  | `plugins.entries.voice-call.config.fromNumber`                              |
    | `plugins.entries.voice-call.config.streaming.sttProvider`                                        | `plugins.entries.voice-call.config.streaming.provider`                      |
    | `plugins.entries.voice-call.config.streaming.openaiApiKey`/`sttModel`/`silenceDurationMs`/`vadThreshold` | `plugins.entries.voice-call.config.streaming.providers.openai.*`             |
    | `models.providers.*.api: "openai"`                                                               | `"openai-completions"` (هنگام راه‌اندازی Gateway، ارائه‌دهندگانی که `api` آن‌ها یک مقدار enum آینده/ناشناخته است نیز به‌جای توقف ایمن، نادیده گرفته می‌شوند) |
    | `browser.ssrfPolicy.allowPrivateNetwork`                                                         | `browser.ssrfPolicy.dangerouslyAllowPrivateNetwork`                          |
    | `browser.profiles.*.driver: "extension"`                                                         | `"existing-session"`                                                          |
    | `browser.relayBindHost`                                                                          | حذف‌شده (تنظیم قدیمی رله افزونه Chrome)                             |
    | `mcp.servers.*.type` (نام‌های مستعار بومی CLI)                                                        | `mcp.servers.*.transport`                                                    |
    | `mcp.servers.*.disabled`                                                                         | `mcp.servers.*.enabled` معکوس                                              |
    | نام‌های مستعار مهلت زمانی MCP ‏`connectTimeout`/`connect_timeout`/`timeout`                                 | `connectionTimeoutMs`/`requestTimeoutMs`                                    |
    | فیلدهای snake-case سرور MCP                                                                     | فیلدهای camelCase سرور MCP                                                   |
    | `tools.media.image/audio/video.models`                                                           | `tools.media.models` برچسب‌گذاری‌شده با قابلیت                                        |
    | `tools.media.asyncCompletion`                                                                    | حذف‌شده                                                                       |
    | `tools.message.allowCrossContextSend`                                                            | `tools.message.crossContext`                                                  |
    | گزینه‌های `deepgram` مدل رسانه                                                                   | `providerOptions.deepgram`                                                    |
    | `talk.realtime.voice`، `voice` بلادرنگ Discord                                                 | `speakerVoice`                                                                |
    | `agents.defaults.pdfMaxBytesMb`                                                                  | `agents.defaults.pdfMaxMb`                                                    |
    | `tools.exec.timeoutSec`                                                                          | `tools.exec.timeoutSeconds`                                                   |
    | `browser.ssrfPolicy.hostnameAllowlist`                                                           | `browser.ssrfPolicy.allowedHostnames` آگاه از نویسه عام                          |
    | `enableNoVnc` مرورگر sandbox                                                                    | `noVncEnabled`                                                                |
    | `media` ریشه                                                                                     | `attachments`                                                                |
    | بلوک‌های نمایش‌پذیری `heartbeat` کانال/حساب                                                   | `heartbeatVisibility`                                                         |
    | `channels.slack.identity`                                                                        | `channels.slack.postAs`                                                       |
    | `audit` ریشه                                                                                     | `logging.audit`                                                               |
    | `gateway.nodes.skills.enabled`                                                                   | `gateway.nodes.allowSkills`                                                   |
    | `gateway.nodes.allowCommands`/`denyCommands`                                                    | `gateway.nodes.commands.allow`/`deny`                                         |
    | پیش‌فرض‌های مدل تولید                                                                       | `agents.defaults.mediaModels.{image,video,music}`                              |
    | کنترل‌های تنظیم چیدمان نهایی بازنشسته‌شده                                                               | رفتار پیش‌فرض داخلی                                                     |
    | `channels.whatsapp.messagePrefix` و `messages.messagePrefix` قدیمی                            | `channels.whatsapp.responsePrefix`                                            |
    | `channels.whatsapp.ackReaction`                                                                  | `messages.ackReaction` سراسری و `ackReactionScope` در موارد قابل‌ترجمه        |
    | `cron.failureDestination`                                                                        | فیلدهای مقصد در `cron.failureAlert`                                     |
    | `gateway.controlUi.chatMessageMaxWidth`، کلیدهای صرفاً نمایشی `ui.prefs`                       | حذف‌شده (مقیاس متن، عرض گفت‌وگو و فعالیت زنده نوار کناری، محلیِ مرورگر هستند) |
    | `agents.list`                                                                                    | `agents.entries` کلیددار                                                        |
    | `defaultModel` سطح‌بالا                                                                         | `agents.defaults.model`                                                      |
    | `messages.messagePrefix`                                                                         | `channels.whatsapp.responsePrefix`                                            |
    | `session.maintenance.pruneDays`، `session.resetByType.dm`                                        | `session.maintenance.pruneAfter`، `session.resetByType.direct`               |
    | `tui` سطح‌بالا                                                                                  | حذف‌شده (پابرگ TUI از پیش‌فرض فشرده استفاده می‌کند)                            |
    | `plugins.entries.codex.config.codexDynamicToolsProfile`                                          | حذف‌شده (app-server مربوط به Codex همیشه ابزارهای فضای کاری بومی Codex را بومی نگه می‌دارد) |
    | `commands.modelsWrite`                                                                           | حذف‌شده (`/models add` منسوخ شده است)                                       |
    | `agents.defaults/list[].silentReplyRewrite`، `surfaces.*.silentReplyRewrite`                     | حذف‌شده (`NO_REPLY` دقیق دیگر به متن بازگشتی قابل‌مشاهده بازنویسی نمی‌شود)  |
    | `agents.defaults/list[].systemPromptOverride`                                                    | حذف‌شده (OpenClaw مالک پرامپت سیستمی تولیدشده است)                        |
    | `agents.defaults/list[].embeddedPi`                                                              | `embeddedAgent`                                                              |
    | `agents.defaults/list[].sandbox.perSession`                                                      | `sandbox.scope`                                                              |
    | `agents.defaults.llm`                                                                             | حذف‌شده (برای مهلت‌های زمانی مدل/ارائه‌دهنده کند از `models.providers.<id>.timeoutSeconds` استفاده کنید که پایین‌تر از سقف مهلت زمانی عامل/اجرا نگه داشته می‌شود) |
    | سطح‌بالا `memorySearch`، `agents.defaults.memorySearch`                                         | `memory.search`                                                             |
    | `agents.entries.*.memorySearch`                                                                     | `agents.entries.*.memory.search`                                               |
    | `memorySearch.provider: "auto"`                                                                  | `"openai"`                                                                    |
    | `memorySearch.store.path` (در هر سطحی)                                                            | حذف شد (نمایه‌های حافظه در پایگاه داده هر عامل قرار دارند)                       |
    | سطح‌بالا `heartbeat`                                                                            | `agents.defaults.heartbeat` / `channels.defaults.heartbeat`                 |
    | شناسه‌های سیاست `plugins.openai-codex`                                                                | `plugins.openai`                                                             |
    | `tools.web.x_search.apiKey`                                                                      | `plugins.entries.xai.config.webSearch.apiKey`                               |
    | `session.maintenance.rotateBytes`، `session.parentForkMaxTokens`                                 | حذف شد (منسوخ)                                                        |
    | گزینه‌های تنظیم Runtime و کانال که در 2026.7 کنار گذاشته شدند                                               | حذف شد (پیش‌فرض‌های داخلی محیط عملیاتی اعمال می‌شوند)                               |

    <Note>
      ردیف‌های `plugins.entries.voice-call.config.*` بالا در هر بار بارگذاری پیکربندی توسط
      خود Plugin تماس صوتی عادی‌سازی می‌شوند، نه توسط `openclaw
      doctor`. این Plugin همچنین هنگام راه‌اندازی هشداری ثبت می‌کند که به `openclaw
      doctor --fix` اشاره دارد، اما doctor در حال حاضر
      `openclaw.json` را برای این کلیدها بازنویسی نمی‌کند؛ این عادی‌سازی خود Plugin است که
      تغییر را هنگام اجرا اعمال می‌کند.
    </Note>

    راهنمای پیش‌فرض حساب برای کانال‌های چندحسابی:

    - اگر دو یا چند ورودی `channels.<channel>.accounts` بدون `channels.<channel>.defaultAccount` یا `accounts.default` پیکربندی شده باشند، doctor هشدار می‌دهد که مسیریابی جایگزین ممکن است حسابی غیرمنتظره را انتخاب کند.
    - اگر `channels.<channel>.defaultAccount` روی شناسه حسابی ناشناخته تنظیم شده باشد، doctor هشدار می‌دهد و شناسه‌های حساب پیکربندی‌شده را فهرست می‌کند.

  </Accordion>
  <Accordion title="2b. بازنویسی‌های ارائه‌دهنده OpenCode">
    اگر `models.providers.opencode`، `opencode-zen` یا `opencode-go` را به‌صورت دستی افزوده باشید، کاتالوگ داخلی OpenCode از `openclaw/plugin-sdk/llm` را بازنویسی می‌کند. این کار ممکن است مدل‌ها را وادار کند از API نادرست استفاده کنند یا هزینه‌ها را صفر کند. Doctor هشدار می‌دهد تا بتوانید بازنویسی را حذف کنید و مسیریابی API و هزینه‌های مختص هر مدل را بازیابی کنید.
  </Accordion>
  <Accordion title="2c. مهاجرت مرورگر و آمادگی Chrome MCP">
    اگر پیکربندی مرورگر شما همچنان به مسیر حذف‌شده افزونه Chrome اشاره کند، doctor آن را به مدل اتصال فعلی Chrome MCP محلیِ میزبان عادی‌سازی می‌کند (`browser.profiles.*.driver: "extension"` → `"existing-session"`؛ `browser.relayBindHost` حذف می‌شود).

    Doctor همچنین هنگام استفاده از `defaultProfile: "user"` یا نمایه پیکربندی‌شده `existing-session`، مسیر Chrome MCP محلیِ میزبان را بررسی می‌کند:

    - برای نمایه‌های اتصال خودکار پیش‌فرض، بررسی می‌کند که Google Chrome روی همان میزبان نصب شده باشد
    - نسخه شناسایی‌شده Chrome را بررسی می‌کند و اگر پایین‌تر از Chrome 144 باشد هشدار می‌دهد
    - یادآوری می‌کند که اشکال‌زدایی از راه دور را در صفحه بازرسی مرورگر فعال کنید (برای مثال `chrome://inspect/#remote-debugging`، `brave://inspect/#remote-debugging` یا `edge://inspect/#remote-debugging`)

    Doctor نمی‌تواند تنظیم سمت Chrome را برای شما فعال کند. Chrome MCP محلیِ میزبان همچنان به مرورگری مبتنی بر Chromium با نسخه 144+ روی میزبان gateway/node نیاز دارد که به‌صورت محلی اجرا شود، اشکال‌زدایی از راه دور در آن فعال باشد و درخواست اولیه رضایت برای اتصال در مرورگر تأیید شده باشد.

    آمادگی در اینجا فقط پیش‌نیازهای اتصال محلی را پوشش می‌دهد. Existing-session محدودیت‌های فعلی مسیر Chrome MCP را حفظ می‌کند؛ مسیرهای پیشرفته‌ای مانند `responsebody`، خروجی PDF، رهگیری دانلود و عملیات دسته‌ای همچنان به مرورگر مدیریت‌شده یا نمایه خام CDP نیاز دارند. این بررسی شامل Docker، sandbox، مرورگر راه‌دور یا دیگر جریان‌های headless نمی‌شود که همچنان از CDP خام استفاده می‌کنند.

  </Accordion>
  <Accordion title="2d. پیش‌نیازهای TLS برای OAuth">
    وقتی نمایه OAuth مربوط به OpenAI Codex پیکربندی شده باشد، doctor نقطه پایانی مجوزدهی OpenAI را بررسی می‌کند تا اطمینان یابد پشته محلی TLS در Node/OpenSSL می‌تواند زنجیره گواهی را اعتبارسنجی کند. اگر بررسی به‌دلیل خطای گواهی ناموفق باشد (برای مثال `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`، گواهی منقضی‌شده یا گواهی خودامضا)، doctor راهنمای رفع مشکل مختص پلتفرم را نمایش می‌دهد. در macOS با Node نصب‌شده از Homebrew، راه‌حل معمولاً `brew postinstall ca-certificates` است. با `--deep`، حتی اگر Gateway سالم باشد نیز بررسی اجرا می‌شود.
  </Accordion>
  <Accordion title="2e. بازنویسی‌های ارائه‌دهنده OAuth مربوط به Codex">
    اگر پیش‌تر تنظیمات قدیمی انتقال OpenAI را زیر `models.providers.openai-codex` افزوده باشید، ممکن است مسیر داخلی ارائه‌دهنده OAuth مربوط به Codex را تحت‌الشعاع قرار دهند. وقتی doctor این تنظیمات قدیمی انتقال را در کنار OAuth مربوط به Codex ببیند، هشدار می‌دهد تا بتوانید بازنویسی انتقال منسوخ را حذف یا بازنویسی کنید و رفتار فعلی مسیریابی را بازیابی کنید. پراکسی‌های سفارشی و بازنویسی‌های صرفاً مبتنی بر سرآیند همچنان پشتیبانی می‌شوند و این هشدار را فعال نمی‌کنند، اما مسیرهای درخواست تعریف‌شده توسط کاربر واجد شرایط انتخاب ضمنی Codex نیستند.
  </Accordion>
  <Accordion title="2f. ترمیم مسیر Codex">
    Doctor ارجاع‌های قدیمی مدل `openai-codex/*` را بررسی می‌کند. مسیریابی بومی هارنس Codex از ارجاع‌های متعارف مدل `openai/*` استفاده می‌کند، اما پیشوند به‌تنهایی هرگز Codex را انتخاب نمی‌کند. وقتی خط‌مشی زمان اجرا تنظیم نشده باشد یا `auto` باشد، فقط یک مسیر رسمی و دقیق HTTPS مربوط به Platform Responses یا ChatGPT Responses، بدون بازنویسی درخواست تعریف‌شده توسط کاربر، واجد شرایط است. به [زمان اجرای ضمنی عامل OpenAI](/fa/providers/openai#implicit-agent-runtime) مراجعه کنید.

    در حالت `--fix` / `--repair`، doctor ارجاع‌های پیش‌فرض عامل و هر عامل را بازنویسی می‌کند؛ از جمله مدل‌های اصلی، جایگزین‌ها، مدل‌های تولید تصویر/ویدئو، بازنویسی‌های heartbeat/زیرعامل/compaction، هوک‌ها، بازنویسی مدل کانال و وضعیت منسوخ و ماندگار مسیر نشست:

    - `openai-codex/gpt-*` به `openai/gpt-*` تبدیل می‌شود.
    - قصد استفاده از Codex برای ارجاع‌های ترمیم‌شده مدل عامل به ورودی‌های `agentRuntime.id: "codex"` با دامنه ارائه‌دهنده/مدل منتقل می‌شود.
    - پیکربندی منسوخ زمان اجرای کل عامل و تثبیت‌های ماندگار زمان اجرای نشست حذف می‌شوند، زیرا انتخاب زمان اجرا دارای دامنه ارائه‌دهنده/مدل است.
    - خط‌مشی موجود زمان اجرای ارائه‌دهنده/مدل حفظ می‌شود، مگر اینکه ارجاع ترمیم‌شده مدل قدیمی برای حفظ مسیر احراز هویت قبلی به مسیریابی Codex نیاز داشته باشد.
    - فهرست‌های موجود جایگزین مدل حفظ می‌شوند و ورودی‌های قدیمی آن‌ها بازنویسی می‌شود؛ تنظیمات کپی‌شده هر مدل از کلید قدیمی به کلید متعارف `openai/*` منتقل می‌شوند.
    - `modelProvider`/`providerOverride`، `model`/`modelOverride`، اعلان‌های جایگزین و تثبیت‌های نمایه احراز هویت ماندگار نشست، در همه مخازن نشست عاملِ کشف‌شده ترمیم می‌شوند.
    - Doctor تثبیت‌های منسوخ `agentRuntime.id: "codex-cli"` (یک شناسه قدیمی و متمایز زمان اجرا) را نیز جداگانه در ورودی‌های مدل `agents.defaults`، `agents.entries.*` و `models.providers.*` به `"codex"` ترمیم می‌کند.
    - `/codex ...` یعنی «یک مکالمه بومی Codex را از طریق چت کنترل یا متصل کنید.»
    - `/acp ...` یا `runtime: "acp"` یعنی «از آداپتور خارجی ACP/acpx استفاده کنید.»

  </Accordion>
  <Accordion title="2g. پاک‌سازی مسیر نشست">
    Doctor همچنین مخازن نشست عاملِ کشف‌شده را برای وضعیت منسوخ و خودکار ایجادشده مسیر، پس از انتقال مدل‌های پیکربندی‌شده یا زمان اجرا از مسیری متعلق به یک Plugin مانند Codex، اسکن می‌کند.

    `openclaw doctor --fix` می‌تواند وضعیت منسوخ و خودکار ایجادشده‌ای مانند تثبیت‌های مدل `modelOverrideSource: "auto"`، فراداده مدل زمان اجرا، شناسه‌های تثبیت‌شده هارنس، اتصال‌های نشست CLI و بازنویسی‌های خودکار نمایه احراز هویت را، وقتی مسیر مالک آن‌ها دیگر پیکربندی نشده است، پاک کند. انتخاب‌های صریح کاربر یا مدل قدیمی نشست برای بازبینی دستی گزارش می‌شوند و دست‌نخورده باقی می‌مانند؛ وقتی دیگر استفاده از آن مسیر مدنظر نیست، آن‌ها را با `/model ...`، `/new` تغییر دهید یا نشست را بازنشانی کنید.

  </Accordion>
  <Accordion title="3. مهاجرت‌های وضعیت قدیمی (چیدمان دیسک)">
    Doctor می‌تواند چیدمان‌های قدیمی روی دیسک را به ساختار فعلی مهاجرت دهد:

    - مخزن نشست‌ها + رونوشت‌ها: از `~/.openclaw/sessions/` به `~/.openclaw/agents/<agentId>/sessions/`
    - دایرکتوری عامل: از `~/.openclaw/agent/` به `~/.openclaw/agents/<agentId>/agent/`
    - وضعیت احراز هویت WhatsApp (Baileys): از `~/.openclaw/credentials/*.json` قدیمی (به‌جز `oauth.json`) به `~/.openclaw/credentials/whatsapp/<accountId>/...` (شناسه حساب پیش‌فرض: `default`)
    - هویت امضاشده دستگاه: از `~/.openclaw/identity/device.json` به ردیف `device_identities` مربوط به `primary` در `state/openclaw.sqlite`؛ فایل جداگانه احراز هویت دستگاه دست‌نخورده باقی می‌ماند

    این مهاجرت‌ها بر مبنای بیشترین تلاش و هم‌توان هستند؛ وقتی doctor هر پوشه قدیمی را به‌عنوان پشتیبان باقی بگذارد، هشدار صادر می‌کند. Gateway/CLI نیز هنگام راه‌اندازی، نشست‌های قدیمی + دایرکتوری عامل را خودکار مهاجرت می‌کند تا تاریخچه/احراز هویت/مدل‌ها بدون اجرای دستی doctor در مسیر مختص عامل قرار گیرند. احراز هویت WhatsApp عمداً فقط از طریق `openclaw doctor` مهاجرت داده می‌شود. عادی‌سازی ارائه‌دهنده Talk/نگاشت ارائه‌دهنده بر اساس برابری ساختاری مقایسه می‌شود، بنابراین تفاوت‌هایی که صرفاً ناشی از ترتیب کلیدها هستند دیگر تغییرات تکراری و بی‌اثر `doctor --fix` را فعال نمی‌کنند.

  </Accordion>
  <Accordion title="3a. مهاجرت‌های مانیفست Plugin قدیمی">
    Doctor همه مانیفست‌های Plugin نصب‌شده را برای کلیدهای قابلیت منسوخ سطح بالا (`speechProviders`، `realtimeTranscriptionProviders`، `realtimeVoiceProviders`، `mediaUnderstandingProviders`، `imageGenerationProviders`، `videoGenerationProviders`، `webFetchProviders`، `webSearchProviders`) اسکن می‌کند. در صورت یافتن، پیشنهاد می‌دهد آن‌ها را به شیء `contracts` منتقل کرده و فایل مانیفست را درجا بازنویسی کند. این مهاجرت هم‌توان است؛ اگر `contracts` از قبل همان مقادیر را داشته باشد، کلید قدیمی بدون تکرار داده حذف می‌شود.
  </Accordion>
  <Accordion title="3b. مهاجرت‌های مخزن Cron قدیمی">
    Doctor همچنین پیش از وارد کردن ردیف‌های متعارف به SQLite، مخزن قدیمی کارهای cron (`~/.openclaw/cron/jobs.json`) را برای ساختارهای قدیمی کار بررسی می‌کند.

    پاک‌سازی‌های فعلی cron شامل موارد زیر است:

    - `jobId` → `id`
    - `schedule.cron` → `schedule.expr`
    - فیلدهای payload سطح بالا (`message`، `model`، `thinking`، ...) → `payload`
    - فیلدهای تحویل سطح بالا (`deliver`، `channel`، `to`، `provider`، ...) → `delivery`
    - نام‌های مستعار تحویل `provider` در payload → `delivery.channel` صریح
    - کارهای قدیمی Webhook جایگزین `notify: true` → تحویل صریح Webhook از مقدار خام و بازنشسته‌شده `cron.webhook`، در صورت معتبر بودن؛ کارهای اعلان، تحویل چت خود را حفظ می‌کنند و `delivery.completionDestination` را دریافت می‌کنند. سپس Doctor کلید پیکربندی قدیمی را حذف می‌کند. بدون یک Webhook قدیمی قابل‌استفاده، نشانگر غیرفعال سطح بالای `notify` برای کارهای بدون مقصد حذف می‌شود (تحویل موجود، از جمله اعلان، حفظ می‌شود)، زیرا تحویل هنگام اجرا هرگز آن را نمی‌خواند.

    Gateway همچنین هنگام بارگذاری، ردیف‌های cron بدساخت را پاک‌سازی می‌کند تا کارهای معتبر همچنان اجرا شوند. ردیف‌های خام بدساخت پیش از حذف از `jobs.json`، در `jobs-quarantine.json` کنار مخزن فعال کپی می‌شوند؛ doctor ردیف‌های قرنطینه‌شده را گزارش می‌کند تا بتوانید آن‌ها را به‌صورت دستی بازبینی یا ترمیم کنید.

    راه‌اندازی Gateway تصویر زمان اجرا را عادی‌سازی می‌کند و نشانگر سطح بالای `notify` را نادیده می‌گیرد، اما وضعیت ماندگار cron را برای ترمیم توسط doctor باقی می‌گذارد. Doctor نشانگرهای غیرفعال را برای کارهایی که مقصد مهاجرت ندارند حذف می‌کند (`delivery.mode` فاقد مقدار/غایب، مقصد Webhook قدیمی غیرقابل‌استفاده، یا تحویل اعلان/چت موجود) و تحویل موجود را دست‌نخورده باقی می‌گذارد؛ بنابراین اجراهای تکراری `doctor --fix` دیگر درباره همان کار دوباره هشدار نمی‌دهند.

    در Linux، doctor همچنین وقتی crontab کاربر همچنان `~/.openclaw/bin/ensure-whatsapp.sh` قدیمی را فراخوانی می‌کند هشدار می‌دهد. این اسکریپت محلیِ میزبان در OpenClaw فعلی نگه‌داری نمی‌شود و وقتی cron نتواند به گذرگاه کاربر systemd دسترسی پیدا کند، ممکن است پیام‌های نادرست `Gateway inactive` را در `~/.openclaw/logs/whatsapp-health.log` بنویسد. ورودی منسوخ crontab را با `crontab -e` حذف کنید؛ برای بررسی‌های سلامت فعلی از `openclaw channels status --probe`، `openclaw doctor` و `openclaw gateway status` استفاده کنید.

  </Accordion>
  <Accordion title="3c. پاک‌سازی قفل نشست">
    Doctor همه دایرکتوری‌های نشست عامل را برای یافتن فایل‌های قفل نوشتنِ باقی‌مانده از نشست‌هایی که به‌طور غیرعادی خاتمه یافته‌اند، اسکن می‌کند. برای هر فایل قفل یافت‌شده، این موارد را گزارش می‌دهد: مسیر، PID، اینکه آیا PID همچنان فعال است، عمر قفل و اینکه آیا قفل منقضی‌شده تلقی می‌شود یا نه (PID مرده، فراداده مالک بدشکل، قدیمی‌تر از 30 دقیقه، یا PID فعالی که ثابت شده متعلق به فرایندی غیر از OpenClaw است). در حالت `--fix` / `--repair`، قفل‌هایی را که مالک مرده، یتیم، بازیافت‌شده، قدیمی و بدشکل، یا غیر OpenClaw دارند، به‌طور خودکار حذف می‌کند. قفل‌های قدیمی که همچنان متعلق به یک فرایند فعال OpenClaw هستند گزارش می‌شوند، اما در جای خود باقی می‌مانند تا Doctor نویسنده فعال رونوشت را قطع نکند.
  </Accordion>
  <Accordion title="3d. ترمیم شاخه رونوشت نشست">
    Doctor فایل‌های JSONL نشست عامل را برای یافتن ساختار شاخه تکراری ایجادشده بر اثر باگ بازنویسی رونوشت پرامپت در 2026.4.24 اسکن می‌کند: یک نوبت کاربر رهاشده همراه با زمینه زمان‌اجرای داخلی OpenClaw، به‌علاوه یک شاخه هم‌سطح فعال که همان پرامپت قابل‌مشاهده کاربر را در بر دارد. در حالت `--fix` / `--repair`، Doctor از هر فایل متأثر در کنار فایل اصلی نسخه پشتیبان می‌گیرد و رونوشت را به شاخه فعال بازنویسی می‌کند تا تاریخچه Gateway و خوانشگرهای حافظه دیگر نوبت‌های تکراری را نبینند.
  </Accordion>
  <Accordion title="4. بررسی‌های یکپارچگی وضعیت (ماندگاری نشست، مسیریابی و ایمنی)">
    دایرکتوری وضعیت ساقه مغز عملیاتی است. اگر ناپدید شود، نشست‌ها، اعتبارنامه‌ها، گزارش‌ها و پیکربندی را از دست می‌دهید، مگر اینکه در جایی دیگر نسخه پشتیبان داشته باشید.

    Doctor موارد زیر را بررسی می‌کند:

    - **نبودن دایرکتوری وضعیت**: درباره از دست رفتن فاجعه‌بار وضعیت هشدار می‌دهد، برای ایجاد مجدد دایرکتوری درخواست تأیید می‌کند و یادآوری می‌کند که نمی‌تواند داده‌های ازدست‌رفته را بازیابی کند.
    - **مجوزهای دایرکتوری وضعیت**: نوشتنی‌بودن را بررسی می‌کند؛ پیشنهاد ترمیم مجوزها را می‌دهد (و هنگام تشخیص ناهماهنگی مالک/گروه، راهنمای `chown` را نمایش می‌دهد).
    - **دایرکتوری وضعیت همگام‌شده با فضای ابری macOS**: هنگامی که مسیر وضعیت زیر iCloud Drive ‏(`~/Library/Mobile Documents/com~apple~CloudDocs/...`) یا `~/Library/CloudStorage/...` قرار گرفته باشد هشدار می‌دهد، زیرا مسیرهای متکی بر همگام‌سازی می‌توانند باعث کندی ورودی/خروجی و رقابت‌های قفل/همگام‌سازی شوند.
    - **دایرکتوری وضعیت روی SD یا eMMC در Linux**: هنگامی که مسیر وضعیت به یک منبع اتصال `mmcblk*` منتهی شود هشدار می‌دهد، زیرا ورودی/خروجی تصادفی متکی بر SD/eMMC ممکن است هنگام نوشتن نشست و اعتبارنامه کندتر باشد و سریع‌تر فرسوده شود.
    - **دایرکتوری وضعیت فرّار در Linux**: هنگامی که مسیر وضعیت به `tmpfs` یا `ramfs` منتهی شود هشدار می‌دهد، زیرا نشست‌ها، اعتبارنامه‌ها، پیکربندی و وضعیت SQLite (همراه با فایل‌های جانبی WAL/journal) با راه‌اندازی مجدد ناپدید می‌شوند. اتصال‌های `overlay` در Docker عمداً علامت‌گذاری نمی‌شوند، زیرا تا زمانی که کانتینر باقی بماند، لایه‌های نوشتنی آن‌ها در راه‌اندازی مجدد میزبان ماندگارند.
    - **نبودن دایرکتوری‌های نشست**: `sessions/` و دایرکتوری ذخیره‌گاه نشست برای ماندگارکردن تاریخچه و جلوگیری از ازکارافتادگی‌های `ENOENT` ضروری‌اند.
    - **ناهماهنگی رونوشت**: هنگامی که ورودی‌های اخیر نشست فایل رونوشت ندارند هشدار می‌دهد.
    - **نشست اصلی «JSONL یک‌خطی»**: هنگامی که رونوشت اصلی فقط یک خط دارد علامت‌گذاری می‌کند (تاریخچه انباشته نمی‌شود).
    - **چندین دایرکتوری وضعیت**: هنگامی که چند پوشه `~/.openclaw` در دایرکتوری‌های خانه وجود داشته باشد، یا `OPENCLAW_STATE_DIR` به جای دیگری اشاره کند، هشدار می‌دهد (تاریخچه ممکن است میان نصب‌ها تقسیم شود).
    - **یادآوری حالت راه‌دور**: اگر `gateway.mode=remote`، Doctor یادآوری می‌کند که آن را روی میزبان راه‌دور اجرا کنید (وضعیت در آنجا قرار دارد).
    - **مجوزهای فایل پیکربندی**: اگر `~/.openclaw/openclaw.json` برای گروه/همگان خواندنی باشد هشدار می‌دهد و پیشنهاد می‌کند مجوز به `600` محدود شود.

  </Accordion>
  <Accordion title="5. سلامت احراز هویت مدل (انقضای OAuth)">
    Doctor پروفایل‌های OAuth را در ذخیره‌گاه احراز هویت بررسی می‌کند، هنگام نزدیک‌بودن انقضا یا منقضی‌شدن توکن‌ها هشدار می‌دهد و در صورت ایمن‌بودن می‌تواند آن‌ها را تازه‌سازی کند. اگر پروفایل OAuth/توکن Anthropic منقضی باشد، یک کلید API متعلق به Anthropic یا مسیر توکن راه‌اندازی Anthropic را پیشنهاد می‌کند. درخواست‌های تازه‌سازی فقط هنگام اجرای تعاملی (TTY) ظاهر می‌شوند؛ `--non-interactive` تلاش‌های تازه‌سازی را نادیده می‌گیرد.

    هنگامی که تازه‌سازی OAuth به‌طور دائمی ناموفق باشد (برای مثال `refresh_token_reused`، `invalid_grant`، یا ارائه‌دهنده‌ای که می‌گوید دوباره وارد شوید)، Doctor گزارش می‌دهد که احراز هویت مجدد لازم است و فرمان دقیق `openclaw models auth login --provider ...` را برای اجرا نمایش می‌دهد.

    Doctor همچنین پروفایل‌های احراز هویتی را گزارش می‌کند که به‌دلیل دوره‌های انتظار کوتاه (محدودیت نرخ/مهلت زمانی/شکست احراز هویت) یا غیرفعال‌سازی‌های طولانی‌تر (شکست صورت‌حساب/اعتبار) موقتاً قابل‌استفاده نیستند.

    پروفایل‌های قدیمی Codex OAuth که توکن‌هایشان در Keychain در macOS قرار دارد (فرایندهای راه‌اندازی قدیمی‌تر، پیش از چیدمان فایل جانبی مبتنی بر فایل) فقط توسط Doctor ترمیم می‌شوند. برای انتقال درجا و یک‌باره توکن‌های قدیمی متکی بر Keychain به `auth-profiles.json`، فرمان `openclaw doctor --fix` را از یک ترمینال تعاملی اجرا کنید؛ پس از آن، نوبت‌های تعبیه‌شده (Telegram، cron، اعزام زیرعامل) آن‌ها را به‌عنوان پروفایل‌های متعارف OpenAI OAuth شناسایی می‌کنند.

  </Accordion>
  <Accordion title="6. اعتبارسنجی مدل هوک‌ها">
    اگر `hooks.gmail.model` تنظیم شده باشد، Doctor ارجاع مدل را در برابر کاتالوگ و فهرست مجاز اعتبارسنجی می‌کند و هنگامی که قابل‌شناسایی نباشد یا مجاز نباشد هشدار می‌دهد.
  </Accordion>
  <Accordion title="7. ترمیم تصویر سندباکس">
    هنگامی که سندباکس فعال باشد، Doctor تصویرهای Docker را بررسی می‌کند و اگر تصویر فعلی موجود نباشد، پیشنهاد ساختن آن یا تغییر به نام‌های قدیمی را می‌دهد.
  </Accordion>
  <Accordion title="7b. پاک‌سازی نصب Plugin">
    Doctor در حالت `openclaw doctor --fix` / `openclaw doctor --repair` وضعیت قدیمی مرحله‌بندی وابستگی Plugin را که OpenClaw ایجاد کرده است حذف می‌کند: ریشه‌های قدیمی وابستگی تولیدشده، دایرکتوری‌های قدیمی مرحله نصب، بقایای محلی بسته از کدهای قدیمی ترمیم وابستگی Plugin داخلی و نسخه‌های مدیریت‌شده npm یتیم یا بازیابی‌شده از Pluginهای داخلی `@openclaw/*` که می‌توانند مانیفست داخلی فعلی را تحت‌الشعاع قرار دهند. Doctor همچنین بسته میزبان `openclaw` را دوباره به Pluginهای مدیریت‌شده npm که `peerDependencies.openclaw` را اعلام می‌کنند پیوند می‌دهد تا ایمپورت‌های زمان‌اجرای محلی بسته، مانند `openclaw/plugin-sdk/*`، پس از به‌روزرسانی‌ها یا ترمیم‌های npm همچنان قابل‌شناسایی باشند.

    Doctor همچنین می‌تواند Pluginهای قابل‌بارگیریِ مفقود را هنگامی دوباره نصب کند که پیکربندی به آن‌ها ارجاع می‌دهد، اما رجیستری محلی Plugin نمی‌تواند آن‌ها را پیدا کند (موارد مهم `plugins.entries`، تنظیمات پیکربندی‌شده کانال/ارائه‌دهنده/جست‌وجو، زمان‌های اجرای پیکربندی‌شده عامل). هنگام به‌روزرسانی بسته‌ها، Doctor تا وقتی بسته اصلی در حال جایگزینی است از نصب مجدد بسته‌های Plugin خودداری می‌کند؛ اگر یک Plugin پیکربندی‌شده همچنان به بازیابی نیاز دارد، پس از به‌روزرسانی دوباره `openclaw doctor --fix` را اجرا کنید. خارج از استثنای راه‌اندازی تصویر کانتینر در ادامه، راه‌اندازی Gateway و بارگذاری مجدد پیکربندی، ترمیم بسته را اجرا نمی‌کنند؛ نصب Pluginها همچنان کار صریح Doctor/نصب/به‌روزرسانی است.

    راه‌اندازی Gateway کانتینری یک استثنای محدود برای ارتقا دارد: هنگامی که `openclaw gateway run` روی نسخه جدید OpenClaw راه‌اندازی می‌شود، پیش از آماده‌شدن، مهاجرت‌های ایمن وضعیت و همگرایی موجود Plugin پس از بخش اصلی را اجرا می‌کند و سپس یک نقطه وارسی برای هر نسخه ثبت می‌کند. این گذر راه‌اندازی می‌تواند رکوردهای قدیمی Pluginهای داخلی را پاک کند، پیوندهای محلی Plugin را ترمیم کند، بسته‌های Plugin پیکربندی‌شده را هنگامی که مسیر همگرایی به آن نیاز دارد دوباره نصب کند و بارهای فعال Plugin را بررسی کند. اگر راه‌اندازی نتواند ترمیم را به‌صورت ایمن انجام دهد، همان تصویر را یک‌بار با `openclaw doctor --fix` و همان وضعیت/پیکربندی متصل‌شده اجرا کنید، سپس کانتینر را به‌صورت عادی دوباره راه‌اندازی کنید.

  </Accordion>
  <Accordion title="8. مهاجرت‌های سرویس Gateway و راهنمای پاک‌سازی">
    Doctor سرویس‌های قدیمی Gateway ‏(launchd/systemd/schtasks) را شناسایی می‌کند و پیشنهاد حذف آن‌ها و نصب سرویس OpenClaw با استفاده از پورت فعلی Gateway را می‌دهد. همچنین می‌تواند سرویس‌های اضافی شبیه Gateway را اسکن کند و راهنمای پاک‌سازی نمایش دهد. سرویس‌های Gateway متعلق به OpenClaw که با نام پروفایل نام‌گذاری شده‌اند، درجه‌یک محسوب می‌شوند و به‌عنوان «اضافی» علامت‌گذاری نمی‌شوند.

    در Linux، اگر سرویس Gateway در سطح کاربر وجود نداشته باشد، اما یک سرویس Gateway متعلق به OpenClaw در سطح سیستم وجود داشته باشد، Doctor سرویس دومی را در سطح کاربر به‌طور خودکار نصب نمی‌کند. با `openclaw gateway status --deep` یا `openclaw doctor --deep` بررسی کنید، سپس سرویس تکراری را حذف کنید یا هنگامی که یک ناظر سیستم مالک چرخه عمر Gateway است، `OPENCLAW_SERVICE_REPAIR_POLICY=external` را تنظیم کنید.

  </Accordion>
  <Accordion title="8b. مهاجرت Matrix هنگام راه‌اندازی">
    هنگامی که حساب کانال Matrix یک مهاجرت وضعیت قدیمیِ معلق یا قابل‌اقدام دارد، Doctor (در حالت `--fix` / `--repair`) یک اسنپ‌شات پیش از مهاجرت ایجاد می‌کند و سپس مراحل مهاجرت را به‌صورت بهترین تلاش اجرا می‌کند: مهاجرت وضعیت قدیمی Matrix و آماده‌سازی وضعیت رمزگذاری‌شده قدیمی. هیچ‌یک از این دو مرحله کشنده نیستند؛ خطاها ثبت می‌شوند و راه‌اندازی ادامه می‌یابد. در حالت فقط‌خواندنی (`openclaw doctor` بدون `--fix`) این بررسی به‌طور کامل نادیده گرفته می‌شود.
  </Accordion>
  <Accordion title="8c. جفت‌سازی دستگاه و انحراف احراز هویت">
    Doctor وضعیت جفت‌سازی دستگاه را به‌عنوان بخشی از بررسی عادی سلامت وارسی می‌کند و موارد زیر را گزارش می‌دهد:

    - درخواست‌های معلق جفت‌سازی برای نخستین بار
    - ارتقاهای معلق نقش یا دامنه برای دستگاه‌هایی که از قبل جفت شده‌اند
    - ترمیم‌های ناهماهنگی کلید عمومی، در مواردی که شناسه دستگاه همچنان مطابقت دارد اما هویت دستگاه دیگر با رکورد تأییدشده مطابقت ندارد
    - رکوردهای جفت‌شده‌ای که برای یک نقش تأییدشده توکن فعال ندارند
    - توکن‌های جفت‌شده‌ای که دامنه‌هایشان از خط مبنای تأییدشده جفت‌سازی منحرف شده‌اند
    - ورودی‌های محلی ذخیره‌شده توکن دستگاه برای ماشین فعلی که پیش از چرخش توکن در سمت Gateway ایجاد شده‌اند یا فراداده دامنه منقضی دارند

    Doctor درخواست‌های جفت‌سازی را به‌طور خودکار تأیید نمی‌کند و توکن‌های دستگاه را نیز به‌طور خودکار نمی‌چرخاند. مراحل بعدی دقیق را نمایش می‌دهد:

    - درخواست‌های معلق را با `openclaw devices list` بررسی کنید
    - درخواست دقیق را با `openclaw devices approve <requestId>` تأیید کنید
    - یک توکن تازه را با `openclaw devices rotate --device <deviceId> --role <role>` بچرخانید
    - یک رکورد منقضی را با `openclaw devices remove <deviceId>` حذف و دوباره تأیید کنید

    این کار جفت‌سازی نخستین بار را از ارتقاهای معلق نقش/دامنه و از انحراف توکن منقضی/هویت دستگاه متمایز می‌کند و رخنه رایج «از قبل جفت شده، اما همچنان پیام نیاز به جفت‌سازی دریافت می‌شود» را می‌بندد.

  </Accordion>
  <Accordion title="9. هشدارهای امنیتی">
    Doctor تنها هنگامی یادداشت امنیتی نمایش می‌دهد که هشداری پیدا کند؛ مانند ارائه‌دهنده‌ای که بدون فهرست مجاز برای پیام‌های مستقیم باز است یا خط‌مشی‌ای که به‌شکل خطرناکی پیکربندی شده است. برای فهرست کامل امنیتی از `openclaw security audit` استفاده کنید.
  </Accordion>
  <Accordion title="10. ماندگاری systemd ‏(Linux)">
    اگر به‌عنوان سرویس کاربر systemd اجرا شود، Doctor اطمینان حاصل می‌کند که ماندگاری فعال است تا Gateway پس از خروج کاربر فعال بماند.
  </Accordion>
  <Accordion title="11. وضعیت فضای کاری (Skills، Pluginها و TaskFlowها)">
    Doctor مشکلات و اقدامات مربوط به عامل پیش‌فرض را نمایش می‌دهد، نه فهرست وضعیت سالم:

    - **Skills**: نام Skills مجاز اما غیرقابل‌استفاده را فهرست می‌کند؛ برای جزئیات الزامات و شمارش کامل از `openclaw skills check` استفاده کنید.
    - **Pluginها**: فقط شناسه Pluginهای خطادار را گزارش می‌دهد؛ برای فهرست Pluginهای بارگذاری‌شده، ایمپورت‌شده، غیرفعال و بسته‌ای از `openclaw plugins list` استفاده کنید.
    - **هشدارهای سازگاری Plugin**: Pluginهایی را که با زمان‌اجرای فعلی مشکلات سازگاری دارند علامت‌گذاری می‌کند.
    - **عیب‌یابی Plugin**: هرگونه هشدار یا خطای زمان بارگذاری را که رجیستری Plugin ایجاد کرده است نمایش می‌دهد.
    - **بازیابی TaskFlow**: TaskFlowهای مدیریت‌شده مشکوکی را که نیاز به بررسی دستی یا لغو دارند نمایش می‌دهد.
    - **Claude CLI**: فقط مشکلات فایل اجرایی، احراز هویت، پروفایل، فضای کاری یا دایرکتوری پروژه را گزارش می‌دهد؛ جزئیات کاوش سالم حذف می‌شوند.

  </Accordion>
  <Accordion title="11b. اندازه فایل راه‌اندازی اولیه">
    Doctor بررسی می‌کند که آیا فایل‌های راه‌اندازی اولیه فضای کاری (برای مثال `AGENTS.md`، `CLAUDE.md` یا سایر فایل‌های زمینه تزریق‌شده) نزدیک به بودجه نویسه پیکربندی‌شده یا بیشتر از آن هستند. برای هر فایل، تعداد نویسه‌های خام در برابر تزریق‌شده، درصد کوتاه‌سازی، علت کوتاه‌سازی (`max/file` یا `max/total`) و مجموع نویسه‌های تزریق‌شده را به‌عنوان کسری از بودجه کل گزارش می‌دهد. هنگامی که فایل‌ها کوتاه شده‌اند یا نزدیک به محدودیت هستند، Doctor نکاتی را برای تنظیم `agents.defaults.bootstrapMaxChars` و `agents.defaults.bootstrapTotalMaxChars` نمایش می‌دهد.
  </Accordion>
  <Accordion title="11c. تکمیل خودکار پوسته">
    Doctor بررسی می‌کند که آیا تکمیل خودکار با کلید Tab برای پوسته فعلی (zsh، bash، fish یا PowerShell) نصب شده است:

    - اگر پروفایل پوسته از الگوی تکمیل پویای کندی استفاده کند (`source <(openclaw completion ...)`)، doctor آن را به نوع سریع‌ترِ فایلِ کش‌شده ارتقا می‌دهد.
    - اگر تکمیل در پروفایل پیکربندی شده باشد اما فایل کش وجود نداشته باشد، doctor کش را به‌طور خودکار بازتولید می‌کند.
    - اگر تکمیل اصلاً پیکربندی نشده باشد، doctor برای نصب آن درخواست تأیید می‌کند (فقط در حالت تعاملی؛ با `--non-interactive` نادیده گرفته می‌شود).

    برای بازتولید دستی کش، `openclaw completion --write-state` را اجرا کنید.

  </Accordion>
  <Accordion title="11d. پاک‌سازی Plugin قدیمی کانال">
    وقتی `openclaw doctor --fix` یک Plugin کانال مفقود را حذف می‌کند، پیکربندی معلقِ مختص کانال را که به آن Plugin ارجاع می‌داد نیز حذف می‌کند: ورودی‌های `channels.<id>`، مقصدهای Heartbeat که نام کانال را مشخص کرده‌اند، و بازنویسی‌های `agents.*.models["<channel>/*"]`. این کار از حلقه‌های راه‌اندازی Gateway جلوگیری می‌کند که در آن‌ها زمان‌اجرای کانال حذف شده است، اما پیکربندی همچنان از Gateway می‌خواهد به آن متصل شود.
  </Accordion>
  <Accordion title="12. بررسی‌های احراز هویت Gateway (توکن محلی)">
    Doctor آمادگی احراز هویت توکن محلی Gateway را بررسی می‌کند.

    - اگر حالت توکن به توکن نیاز داشته باشد و هیچ منبع توکنی وجود نداشته باشد، doctor پیشنهاد می‌دهد یکی تولید شود.
    - اگر `gateway.auth.token` با SecretRef مدیریت شود اما در دسترس نباشد، doctor هشدار می‌دهد و آن را با متن ساده بازنویسی نمی‌کند.
    - `openclaw doctor --generate-gateway-token` فقط زمانی تولید را اجباری می‌کند که هیچ SecretRef توکنی پیکربندی نشده باشد.

  </Accordion>
  <Accordion title="12b. تعمیرات فقط‌خواندنیِ آگاه از SecretRef">
    برخی جریان‌های تعمیر باید اعتبارنامه‌های پیکربندی‌شده را بدون تضعیف رفتار توقف سریع زمان‌اجرا بررسی کنند.

    - `openclaw doctor --fix` برای تعمیرات هدفمند پیکربندی، از همان مدل خلاصه فقط‌خواندنی SecretRef استفاده می‌کند که فرمان‌های خانواده وضعیت به‌کار می‌برند.
    - مثال: تعمیر `@username` مربوط به `allowFrom` / `groupAllowFrom` در Telegram تلاش می‌کند در صورت دسترس‌بودن، از اعتبارنامه‌های پیکربندی‌شده ربات استفاده کند.
    - اگر توکن ربات Telegram از طریق SecretRef پیکربندی شده باشد اما در مسیر فرمان فعلی در دسترس نباشد، doctor گزارش می‌دهد که اعتبارنامه پیکربندی‌شده اما در دسترس نیست و به‌جای ازکارافتادن یا گزارش نادرست توکن به‌عنوان مفقود، رفع خودکار را نادیده می‌گیرد.

  </Accordion>
  <Accordion title="13. بررسی سلامت Gateway و راه‌اندازی مجدد">
    Doctor یک بررسی سلامت اجرا می‌کند و وقتی Gateway ناسالم به‌نظر برسد، پیشنهاد راه‌اندازی مجدد آن را می‌دهد.
  </Accordion>
  <Accordion title="13b. آمادگی جست‌وجوی حافظه">
    Doctor بررسی می‌کند که آیا ارائه‌دهنده تعبیه‌سازیِ پیکربندی‌شده برای جست‌وجوی حافظه، برای عامل پیش‌فرض آماده است یا نه. رفتار به بک‌اند و ارائه‌دهنده پیکربندی‌شده بستگی دارد:

    - **بک‌اند QMD**: بررسی می‌کند که آیا باینری `qmd` در دسترس و قابل راه‌اندازی است یا نه. در غیر این صورت، راهنمای رفع مشکل شامل `npm install -g @tobilu/qmd` (یا معادل Bun) و گزینه مسیر دستی باینری را نمایش می‌دهد.
    - **ارائه‌دهنده محلی صریح**: وجود فایل مدل محلی یا URL شناخته‌شده مدل راه‌دور/قابل‌دانلود را بررسی می‌کند. اگر وجود نداشته باشد، تغییر به یک ارائه‌دهنده راه‌دور را پیشنهاد می‌دهد.
    - **ارائه‌دهنده راه‌دور صریح** (`openai`، `voyage` و غیره): تأیید می‌کند که یک کلید API در محیط یا مخزن احراز هویت وجود دارد. اگر وجود نداشته باشد، راهنمای عملی رفع مشکل را نمایش می‌دهد.
    - **ارائه‌دهنده خودکار قدیمی**: `memorySearch.provider: "auto"` را OpenAI در نظر می‌گیرد، آمادگی OpenAI را بررسی می‌کند و `doctor --fix` آن را به `provider: "openai"` بازنویسی می‌کند.

    وقتی نتیجه کش‌شده بررسی Gateway در دسترس باشد (Gateway هنگام بررسی سالم بوده است)، doctor نتیجه آن را با پیکربندی قابل‌مشاهده در CLI تطبیق می‌دهد و هرگونه مغایرت را اعلام می‌کند. Doctor در مسیر پیش‌فرض، پینگ تعبیه‌سازی جدیدی آغاز نمی‌کند؛ برای بررسی زنده ارائه‌دهنده از فرمان وضعیت عمیق حافظه استفاده کنید.

    برای تأیید آمادگی تعبیه‌سازی در زمان‌اجرا، از `openclaw memory status --deep` استفاده کنید.

  </Accordion>
  <Accordion title="14. هشدارهای وضعیت کانال">
    اگر Gateway سالم باشد، doctor بررسی وضعیت کانال را اجرا می‌کند و هشدارها را همراه با راه‌حل‌های پیشنهادی گزارش می‌دهد.
  </Accordion>
  <Accordion title="15. ممیزی و تعمیر پیکربندی ناظر">
    Doctor پیکربندی ناظر نصب‌شده (launchd/systemd/schtasks) را برای پیش‌فرض‌های مفقود یا قدیمی بررسی می‌کند (برای مثال وابستگی‌های network-online و تأخیر راه‌اندازی مجدد در systemd). وقتی مغایرتی پیدا کند، به‌روزرسانی را توصیه می‌کند و می‌تواند فایل سرویس/وظیفه را مطابق پیش‌فرض‌های فعلی بازنویسی کند.

    نکته‌ها:

    - `openclaw doctor` پیش از بازنویسی پیکربندی ناظر درخواست تأیید می‌کند.
    - `openclaw doctor --yes` درخواست‌های پیش‌فرض تعمیر را می‌پذیرد.
    - `openclaw doctor --fix` اصلاحات توصیه‌شده را بدون درخواست تأیید اعمال می‌کند (`--repair` یک نام مستعار است).
    - `openclaw doctor --fix --force` پیکربندی‌های سفارشی ناظر را بازنویسی می‌کند.
    - `OPENCLAW_SERVICE_REPAIR_POLICY=external` doctor را برای چرخه عمر سرویس Gateway در حالت فقط‌خواندنی نگه می‌دارد. همچنان سلامت سرویس را گزارش می‌دهد و تعمیرات غیرسرویسی را اجرا می‌کند، اما نصب/شروع/راه‌اندازی مجدد/راه‌اندازی اولیه سرویس، بازنویسی پیکربندی ناظر و پاک‌سازی سرویس قدیمی را نادیده می‌گیرد، زیرا یک ناظر خارجی مالک آن چرخه عمر است.
    - در Linux، هنگامی که واحد منطبق systemd مربوط به Gateway فعال است، doctor فراداده فرمان/نقطه ورود را بازنویسی نمی‌کند. همچنین هنگام اسکن سرویس‌های تکراری، واحدهای اضافی غیرفعال، غیرقدیمی و شبیه Gateway را نادیده می‌گیرد تا فایل‌های سرویس همراه موجب پیام‌های زائد پاک‌سازی نشوند.
    - اگر احراز هویت توکنی به توکن نیاز داشته باشد و `gateway.auth.token` با SecretRef مدیریت شود، نصب/تعمیر سرویس توسط doctor، SecretRef را اعتبارسنجی می‌کند اما مقادیر حل‌شده توکن به‌صورت متن ساده را در فراداده محیط سرویس ناظر ذخیره نمی‌کند.
    - Doctor مقادیر محیط سرویسِ مدیریت‌شده با `.env`/مبتنی بر SecretRef را که نصب‌های قدیمی‌تر LaunchAgent،‏ systemd یا Windows Scheduled Task به‌صورت درون‌خطی جاسازی کرده‌اند، شناسایی می‌کند و فراداده سرویس را بازنویسی می‌کند تا آن مقادیر به‌جای تعریف ناظر، از منبع زمان‌اجرا بارگیری شوند.
    - Doctor تشخیص می‌دهد که فرمان سرویس پس از تغییرات `gateway.port` همچنان یک `--port` قدیمی را ثابت نگه داشته است و فراداده سرویس را با درگاه فعلی بازنویسی می‌کند.
    - اگر احراز هویت توکنی به توکن نیاز داشته باشد و SecretRef توکن پیکربندی‌شده حل‌نشده باشد، doctor مسیر نصب/تعمیر را مسدود می‌کند و راهنمای عملی ارائه می‌دهد.
    - اگر هر دو `gateway.auth.token` و `gateway.auth.password` پیکربندی شده باشند و `gateway.auth.mode` تنظیم نشده باشد، doctor نصب/تعمیر را تا زمان تنظیم صریح حالت مسدود می‌کند.
    - برای واحدهای user-systemd در Linux، بررسی‌های اختلاف توکن توسط doctor هنگام مقایسه فراداده احراز هویت سرویس، هر دو منبع `Environment=` و `EnvironmentFile=` را دربر می‌گیرد.
    - تعمیرات سرویس توسط Doctor از بازنویسی، توقف یا راه‌اندازی مجدد سرویس Gateway توسط یک باینری قدیمی‌تر OpenClaw خودداری می‌کند، وقتی پیکربندی آخرین‌بار توسط نسخه‌ای جدیدتر نوشته شده باشد. [عیب‌یابی Gateway](/fa/gateway/troubleshooting#split-brain-installs-and-newer-config-guard) را ببینید.
    - همیشه می‌توانید با `openclaw gateway install --force` بازنویسی کامل را اجباری کنید.

  </Accordion>
  <Accordion title="16. عیب‌یابی زمان‌اجرا و درگاه Gateway">
    Doctor زمان‌اجرای سرویس (PID، آخرین وضعیت خروج) را بررسی می‌کند و هنگامی که سرویس نصب شده اما واقعاً در حال اجرا نیست، هشدار می‌دهد. همچنین تداخل‌های درگاه Gateway (پیش‌فرض `18789`) را بررسی می‌کند و علت‌های محتمل را گزارش می‌دهد (Gateway از قبل در حال اجرا است، تونل SSH).
  </Accordion>
  <Accordion title="17. بهترین شیوه‌های زمان‌اجرای Gateway">
    Doctor هنگامی هشدار می‌دهد که سرویس Gateway روی Bun یا یک مسیر Node مدیریت‌شده با نسخه (`nvm`، `fnm`، `volta`، `asdf` و غیره) اجرا شود. Bun نمی‌تواند مخزن وضعیت `node:sqlite` متعلق به OpenClaw را باز کند، بنابراین تعمیرات، سرویس‌های قدیمی Bun را به Node مهاجرت می‌دهند. مسیرهای مدیر نسخه ممکن است پس از ارتقا از کار بیفتند، زیرا سرویس فایل آغازین پوسته را بارگیری نمی‌کند. Doctor در صورت وجود نصب سیستمی Node، پیشنهاد مهاجرت به آن را می‌دهد (Homebrew/apt/choco).

    LaunchAgentهای macOS که به‌تازگی نصب یا تعمیر شده‌اند، به‌جای کپی‌کردن PATH پوسته تعاملی، از یک PATH سیستمی استاندارد (`/opt/homebrew/bin:/opt/homebrew/sbin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin`) استفاده می‌کنند؛ بنابراین باینری‌های سیستمی مدیریت‌شده با Homebrew در دسترس باقی می‌مانند، درحالی‌که دایرکتوری‌های Volta،‏ asdf،‏ fnm،‏ pnpm و دیگر مدیرهای نسخه، Node مورداستفاده فرایندهای فرزند را تغییر نمی‌دهند. سرویس‌های Linux همچنان ریشه‌های محیطی صریح (`NVM_DIR`، `FNM_DIR`، `VOLTA_HOME`، `ASDF_DATA_DIR`، `BUN_INSTALL`، `PNPM_HOME`) و دایرکتوری‌های پایدار باینری کاربر را حفظ می‌کنند، اما دایرکتوری‌های جایگزین حدس‌زده‌شده مدیر نسخه فقط زمانی در PATH سرویس نوشته می‌شوند که آن دایرکتوری‌ها روی دیسک وجود داشته باشند.

  </Accordion>
  <Accordion title="18. نوشتن پیکربندی و فراداده راهنما">
    Doctor همه تغییرات پیکربندی را ذخیره می‌کند و برای ثبت اجرای doctor، فراداده راهنما را مهر زمانی می‌زند.
  </Accordion>
  <Accordion title="19. نکته‌های فضای کاری (پشتیبان‌گیری و سامانه حافظه)">
    Doctor در صورت نبود سامانه حافظه فضای کاری، آن را پیشنهاد می‌دهد و اگر فضای کاری از قبل تحت git نباشد، نکته‌ای برای پشتیبان‌گیری نمایش می‌دهد.

    برای راهنمای کامل ساختار فضای کاری و پشتیبان‌گیری با git (GitHub یا GitLab خصوصی توصیه می‌شود)، به [/concepts/agent-workspace](/fa/concepts/agent-workspace) مراجعه کنید.

  </Accordion>
</AccordionGroup>

## مرتبط

- [راهنمای عملیاتی Gateway](/fa/gateway)
- [عیب‌یابی Gateway](/fa/gateway/troubleshooting)
