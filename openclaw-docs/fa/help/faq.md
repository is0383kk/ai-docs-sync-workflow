---
read_when:
    - پاسخ‌گویی به پرسش‌های رایج پشتیبانی درباره راه‌اندازی، نصب، شروع به کار یا زمان اجرا
    - اولویت‌بندی مشکلات گزارش‌شده توسط کاربران پیش از اشکال‌زدایی عمیق‌تر
summary: پرسش‌های متداول درباره راه‌اندازی، پیکربندی و استفاده از OpenClaw
title: پرسش‌های متداول
x-i18n:
    generated_at: "2026-07-27T16:39:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 7bddbf851a0e25323aa7e7cfc3882b33cc0d33a2aa223cccf00328af477ab4c4
    source_path: help/faq.md
    workflow: 16
---

پاسخ‌های سریع به‌همراه عیب‌یابی عمیق‌تر برای راه‌اندازی‌های واقعی (توسعه محلی، VPS، چندعاملی، کلیدهای OAuth/API، جایگزینی مدل هنگام خرابی). برای عیب‌یابی زمان اجرا، به [عیب‌یابی](/fa/gateway/troubleshooting) مراجعه کنید. برای مرجع کامل پیکربندی، به [پیکربندی](/fa/gateway/configuration) مراجعه کنید.

## ۶۰ ثانیه نخست در صورت بروز مشکل

<Steps>
  <Step title="وضعیت سریع">
    ```bash
    openclaw status
    ```
    خلاصه سریع محلی: سیستم‌عامل + به‌روزرسانی، دسترس‌پذیری Gateway/سرویس، عامل‌ها/نشست‌ها، پیکربندی ارائه‌دهنده + مشکلات زمان اجرا (هنگامی که Gateway در دسترس باشد).
  </Step>
  <Step title="گزارش قابل جای‌گذاری (ایمن برای اشتراک‌گذاری)">
    ```bash
    openclaw status --all
    ```
    عیب‌یابی فقط‌خواندنی همراه با انتهای گزارش رخدادها (توکن‌ها حذف می‌شوند).
  </Step>
  <Step title="وضعیت دیمون + درگاه">
    ```bash
    openclaw gateway status
    ```
    زمان اجرای سرپرست در مقایسه با دسترس‌پذیری RPC، نشانی URL هدف کاوش و پیکربندی‌ای را نشان می‌دهد که احتمالاً سرویس استفاده کرده است.
  </Step>
  <Step title="کاوش‌های عمیق">
    ```bash
    openclaw status --deep
    ```
    کاوش زنده سلامت Gateway، شامل کاوش کانال‌ها در صورت پشتیبانی (به Gateway قابل‌دسترسی نیاز دارد). به [سلامت](/fa/gateway/health) مراجعه کنید.
  </Step>
  <Step title="مشاهده زنده آخرین گزارش رخداد">
    ```bash
    openclaw logs --follow
    ```
    اگر RPC از کار افتاده است، از این روش جایگزین استفاده کنید:
    ```bash
    tail -f "/tmp/openclaw/openclaw-$(date +%F).log"
    # نمونه نمایه نام‌گذاری‌شده:
    tail -f "/tmp/openclaw/openclaw-dev-$(date +%F).log"
    ```
    گزارش‌های رخداد فایل از گزارش‌های رخداد سرویس جدا هستند؛ به [ثبت گزارش رخداد](/fa/logging) و [عیب‌یابی](/fa/gateway/troubleshooting) مراجعه کنید.
  </Step>
  <Step title="اجرای Doctor (ترمیم‌ها)">
    ```bash
    openclaw doctor
    ```
    پیکربندی و وضعیت را ترمیم/مهاجرت می‌دهد و سپس بررسی‌های سلامت را اجرا می‌کند. به [Doctor](/fa/gateway/doctor) مراجعه کنید.
  </Step>
  <Step title="عکس فوری Gateway (فقط WS)">
    ```bash
    openclaw health --json
    openclaw health --verbose   # هنگام خطا، نشانی URL هدف + مسیر پیکربندی را نشان می‌دهد
    ```
    یک عکس فوری کامل از Gateway در حال اجرا درخواست می‌کند. به [سلامت](/fa/gateway/health) مراجعه کنید.
  </Step>
</Steps>

## شروع سریع و راه‌اندازی اجرای نخست

پرسش‌وپاسخ اجرای نخست — نصب، ورود اولیه، مسیرهای احراز هویت، اشتراک‌ها، خطاهای اولیه — در [پرسش‌های متداول اجرای نخست](/fa/help/faq-first-run) قرار دارد.

## OpenClaw چیست؟

<AccordionGroup>
  <Accordion title="OpenClaw در یک بند چیست؟">
    OpenClaw یک دستیار هوش مصنوعی شخصی است که روی دستگاه‌های خود اجرا می‌کنید. این دستیار در بسترهای پیام‌رسانی‌ای که از قبل استفاده می‌کنید پاسخ می‌دهد (Discord، Google Chat، iMessage، Mattermost، Signal، Slack، Telegram، WebChat، WhatsApp و Pluginهای کانال همراه مانند QQ Bot) و در پلتفرم‌های پشتیبانی‌شده می‌تواند قابلیت صوتی و یک Canvas زنده نیز ارائه دهد. **Gateway** صفحه کنترل همیشه‌فعال است؛ دستیار همان محصول است.
  </Accordion>

  <Accordion title="ارزش پیشنهادی">
    OpenClaw «فقط یک پوشش برای Claude» نیست. یک **صفحه کنترل محلی‌محور** است که دستیاری توانمند را روی **سخت‌افزار خودتان** اجرا می‌کند و از طریق برنامه‌های گفت‌وگویی که از قبل استفاده می‌کنید در دسترس است؛ با نشست‌های دارای وضعیت، حافظه و ابزارها، بدون واگذاری گردش‌کارهایتان به یک SaaS میزبانی‌شده.

    - **دستگاه‌های شما، داده‌های شما**: Gateway را هر جا می‌خواهید (Mac، Linux، VPS) اجرا کنید و فضای کاری و تاریخچه نشست را محلی نگه دارید.
    - **کانال‌های واقعی، نه محیط آزمایشی وب**: Discord/iMessage/Signal/Slack/Telegram/WhatsApp/و غیره، به‌علاوه قابلیت صوتی موبایل و Canvas در پلتفرم‌های پشتیبانی‌شده.
    - **مستقل از مدل**: از Anthropic، MiniMax، OpenAI، OpenRouter و غیره، همراه با مسیریابی و جایگزینی هنگام خرابی برای هر عامل، استفاده کنید.
    - **گزینه کاملاً محلی**: مدل‌های محلی را اجرا کنید تا همه داده‌ها بتوانند روی دستگاهتان باقی بمانند.
    - **مسیریابی چندعاملی**: عامل‌هایی جداگانه برای هر کانال، حساب یا وظیفه داشته باشید که هرکدام فضای کاری و پیش‌فرض‌های خود را دارند.
    - **متن‌باز و قابل‌تغییر**: بدون وابستگی انحصاری به فروشنده، آن را بررسی و گسترش دهید و خودتان میزبانی کنید.

    مستندات: [Gateway](/fa/gateway)، [کانال‌ها](/fa/channels)، [چندعاملی](/fa/concepts/multi-agent)، [حافظه](/fa/concepts/memory).

  </Accordion>

  <Accordion title="تازه آن را راه‌اندازی کرده‌ام؛ ابتدا چه کاری انجام دهم؟">
    پروژه‌های مناسب برای شروع: ساخت یک وب‌سایت (WordPress، Shopify یا یک وب‌سایت ایستا)؛ نمونه‌سازی یک برنامه موبایل (طرح کلی، صفحه‌ها، برنامه API)؛ سازمان‌دهی فایل‌ها و پوشه‌ها؛ اتصال Gmail و خودکارسازی خلاصه‌ها یا پیگیری‌ها.

    این ابزار می‌تواند وظایف بزرگ را انجام دهد، اما وقتی آن‌ها به چند مرحله تقسیم شوند و برای کار موازی از زیرعامل‌ها استفاده شود، بهترین عملکرد را دارد.

  </Accordion>

  <Accordion title="پنج مورد استفاده روزمره برتر OpenClaw کدام‌اند؟">
    - **گزارش‌های مختصر شخصی**: خلاصه‌هایی از صندوق ورودی، تقویم و اخبار موردعلاقه شما.
    - **پژوهش و پیش‌نویس‌نویسی**: پژوهش سریع، خلاصه‌ها و پیش‌نویس‌های اولیه برای ایمیل‌ها یا اسناد.
    - **یادآوری‌ها و پیگیری‌ها**: تلنگرها و فهرست‌های بررسی مبتنی بر Cron یا Heartbeat.
    - **خودکارسازی مرورگر**: تکمیل فرم‌ها، جمع‌آوری داده و تکرار وظایف وب.
    - **هماهنگی میان دستگاه‌ها**: وظیفه‌ای را از تلفن خود ارسال کنید، اجازه دهید Gateway آن را روی سرور اجرا کند و نتیجه را در گفت‌وگو دریافت کنید.

  </Accordion>

  <Accordion title="آیا OpenClaw می‌تواند برای جذب سرنخ، ارتباط‌گیری، تبلیغات و وبلاگ‌های یک SaaS کمک کند؟">
    بله، برای **پژوهش، ارزیابی صلاحیت و پیش‌نویس‌نویسی**: بررسی وب‌سایت‌ها، تهیه فهرست‌های منتخب، خلاصه‌سازی مشتریان بالقوه و نوشتن پیش‌نویس متن‌های ارتباطی یا تبلیغاتی.

    برای **اجرای ارتباط‌گیری یا تبلیغات**، یک انسان را در چرخه نگه دارید. از هرزنامه‌فرستی خودداری کنید، قوانین محلی و سیاست‌های پلتفرم را رعایت کنید و همه‌چیز را پیش از ارسال بازبینی کنید. اجازه دهید OpenClaw پیش‌نویس را تهیه کند؛ شما تأیید کنید.

    مستندات: [امنیت](/fa/gateway/security).

  </Accordion>

  <Accordion title="مزیت‌های آن در مقایسه با Claude Code برای توسعه وب چیست؟">
    OpenClaw یک **دستیار شخصی** و لایه هماهنگی است، نه جایگزین IDE. برای سریع‌ترین چرخه مستقیم کدنویسی درون یک مخزن، از Claude Code یا Codex استفاده کنید. برای حافظه پایدار، دسترسی میان دستگاه‌ها و هماهنگ‌سازی ابزارها از OpenClaw استفاده کنید.

    - حافظه و فضای کاری پایدار در میان نشست‌ها.
    - دسترسی چندپلتفرمی (Telegram، WhatsApp، TUI، WebChat).
    - هماهنگ‌سازی ابزارها (مرورگر، فایل‌ها، زمان‌بندی، هوک‌ها).
    - Gateway همیشه‌فعال (اجرا روی VPS و تعامل از هر مکان).
    - Nodeها برای مرورگر/صفحه‌نمایش/دوربین/اجرای محلی.

    ویترین: [https://openclaw.ai/showcase](https://openclaw.ai/showcase).

  </Accordion>
</AccordionGroup>

## Skills و خودکارسازی

<AccordionGroup>
  <Accordion title="چگونه بدون کثیف نگه‌داشتن مخزن، Skills را سفارشی کنم؟">
    به‌جای ویرایش نسخه مخزن، از بازنویسی‌های مدیریت‌شده استفاده کنید. تغییرات را در `~/.openclaw/skills/<name>/SKILL.md` قرار دهید (یا پوشه‌ای را از طریق `skills.load.extraDirs` در `~/.openclaw/openclaw.json` اضافه کنید). اولویت: `<workspace>/skills` -> `<workspace>/.agents/skills` -> `~/.agents/skills` -> `~/.openclaw/skills` -> همراه -> `skills.load.extraDirs`؛ بنابراین بازنویسی‌های مدیریت‌شده بدون دست‌زدن به git بر Skills همراه اولویت دارند. برای نصب سراسری همراه با محدودکردن نمایش به برخی عامل‌ها، نسخه مشترک را در `~/.openclaw/skills` نگه دارید و نمایش را با `agents.defaults.skills` / `agents.entries.*.skills` کنترل کنید. فقط ویرایش‌هایی که ارزش ارسال به بالادست را دارند باید به‌صورت PR علیه نسخه مخزن ارسال شوند.
  </Accordion>

  <Accordion title="آیا می‌توانم Skills را از یک پوشه سفارشی بارگذاری کنم؟">
    بله: شاخه‌ها را از طریق `skills.load.extraDirs` در `~/.openclaw/openclaw.json` اضافه کنید (کمترین اولویت در ترتیب بالا). `clawhub` به‌طور پیش‌فرض در `./skills` نصب می‌شود و OpenClaw در نشست بعدی آن را به‌عنوان `<workspace>/skills` در نظر می‌گیرد. برای محدودکردن نمایش به عامل‌های خاص، آن را با `agents.defaults.skills` یا `agents.entries.*.skills` جفت کنید.
  </Accordion>

  <Accordion title="چگونه می‌توانم برای وظایف مختلف از مدل‌ها یا تنظیمات متفاوت استفاده کنم؟">
    الگوهای پشتیبانی‌شده:

    - **کارهای Cron**: کارهای ایزوله می‌توانند برای هر کار یک بازنویسی `model` تنظیم کنند.
    - **عامل‌ها**: وظایف را به عامل‌های جداگانه با مدل‌های پیش‌فرض، سطوح تفکر و پارامترهای پخش متفاوت مسیریابی کنید.
    - **تغییر هنگام نیاز**: `/model` مدل نشست فعلی را در هر زمان تغییر می‌دهد.

    نمونه — مدل یکسان، تنظیمات متفاوت برای هر عامل:

    ```json5
    {
      agents: {
        list: [
          {
            id: "coder",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "high",
            params: { temperature: 0.1 },
          },
          {
            id: "chat",
            model: "xiaomi/mimo-v2.5-pro",
            thinkingDefault: "off",
            params: { temperature: 0.8 },
          },
        ],
      },
    }
    ```

    پیش‌فرض‌های مشترک هر مدل را در `agents.defaults.models["provider/model"].params` و سپس بازنویسی‌های ویژه هر عامل را در `agents.entries.*.params` تخت قرار دهید. همان مدل را زیر `agents.entries.*.models["provider/model"].params` تودرتو تکرار نکنید؛ آن مسیر برای کاتالوگ مدل و بازنویسی‌های زمان اجرای هر عامل است.

    به [کارهای Cron](/fa/automation/cron-jobs)، [مسیریابی چندعاملی](/fa/concepts/multi-agent)، [پیکربندی](/fa/gateway/config-agents)، [دستورهای اسلش](/fa/tools/slash-commands) مراجعه کنید.

  </Accordion>

  <Accordion title="ربات هنگام انجام کار سنگین متوقف می‌شود. چگونه آن را واگذار کنم؟">
    برای وظایف طولانی یا موازی از **زیرعامل‌ها** استفاده کنید: آن‌ها در نشست خود اجرا می‌شوند، خلاصه‌ای برمی‌گردانند و گفت‌وگوی اصلی شما را پاسخ‌گو نگه می‌دارند. از ربات بخواهید «برای این وظیفه یک زیرعامل ایجاد کند» یا از `/subagents` استفاده کنید. برای مشاهده اینکه آیا Gateway هم‌اکنون مشغول است، از `/status` استفاده کنید.

    هم وظایف طولانی و هم زیرعامل‌ها توکن مصرف می‌کنند؛ اگر هزینه مهم است، از طریق `agents.defaults.subagents.model` مدل ارزان‌تری برای زیرعامل‌ها تنظیم کنید.

    مستندات: [زیرعامل‌ها](/fa/tools/subagents)، [وظایف پس‌زمینه](/fa/automation/tasks).

  </Accordion>

  <Accordion title="نشست‌های زیرعامل وابسته به رشته در Discord چگونه کار می‌کنند؟">
    یک رشته Discord را به یک زیرعامل یا هدف نشست متصل کنید تا پیام‌های پیگیری در آن رشته در همان نشست متصل باقی بمانند.

    - با `sessions_spawn` و استفاده از `thread: true` ایجاد کنید (در صورت تمایل، `mode: "session"` را برای پیگیری پایدار به‌کار ببرید).
    - یا با `/focus <target>` به‌صورت دستی متصل کنید.
    - `/agents` وضعیت اتصال را بررسی می‌کند.
    - `/session idle <duration|off>` و `/session max-age <duration|off>` خروج خودکار از تمرکز را کنترل می‌کنند.
    - `/unfocus` رشته را جدا می‌کند.

    پیکربندی: `session.threadBindings.enabled` (کلید سراسری)، `session.threadBindings.idleHours` (پیش‌فرض `24`، مقدار `0` آن را غیرفعال می‌کند)، `session.threadBindings.maxAgeHours` (پیش‌فرض `0` = بدون سقف سخت) و `session.threadBindings.spawnSessions` برای اتصال خودکار هنگام ایجاد (پیش‌فرض `true`).

    مستندات: [زیرعامل‌ها](/fa/tools/subagents)، [Discord](/fa/channels/discord)، [مرجع پیکربندی](/fa/gateway/configuration-reference)، [دستورهای اسلش](/fa/tools/slash-commands).

  </Accordion>

  <Accordion title="یک زیرعامل پایان یافت، اما به‌روزرسانی تکمیل به محل اشتباهی رفت یا هرگز ارسال نشد. چه چیزی را بررسی کنم؟">
    مسیر حل‌شده درخواست‌کننده را بررسی کنید:

    - تحویل زیرعامل در حالت تکمیل، در صورت وجود، رشته متصل یا مسیر مکالمه را ترجیح می‌دهد.
    - اگر مبدأ تکمیل فقط یک کانال داشته باشد، OpenClaw به مسیر ذخیره‌شده نشست درخواست‌کننده (`lastChannel` / `lastTo` / `lastAccountId`) برمی‌گردد تا تحویل مستقیم همچنان بتواند موفق شود.
    - بدون مسیر متصل و بدون مسیر ذخیره‌شده قابل‌استفاده: تحویل مستقیم ممکن است ناموفق باشد و نتیجه به‌جای ارسال فوری، به تحویل صف‌شده نشست برمی‌گردد.
    - هدف‌های نامعتبر یا منقضی نیز می‌توانند بازگشت به صف یا شکست نهایی تحویل را تحمیل کنند.
    - اگر آخرین پاسخ قابل‌مشاهده دستیار فرزند دقیقاً `NO_REPLY` / `no_reply` یا `ANNOUNCE_SKIP` باشد، OpenClaw عمداً به‌جای ارسال پیشرفت قدیمی‌تر، اعلان را سرکوب می‌کند.

    اشکال‌زدایی: `openclaw tasks show <lookup>` که در آن `<lookup>` شناسه وظیفه، شناسه اجرا یا کلید نشست است.

    مستندات: [زیرعامل‌ها](/fa/tools/subagents)، [وظایف پس‌زمینه](/fa/automation/tasks)، [ابزارهای نشست](/fa/concepts/session-tool).

  </Accordion>

  <Accordion title="Cron یا یادآوری‌ها اجرا نمی‌شوند. چه چیزی را بررسی کنم؟">
    Cron درون فرایند Gateway اجرا می‌شود؛ اگر Gateway به‌طور پیوسته در حال اجرا نباشد، فعال نمی‌شود.

    - تأیید کنید که Cron فعال است (`cron.enabled`) و `OPENCLAW_SKIP_CRON` تنظیم نشده است.
    - تأیید کنید که Gateway به‌طور مداوم در حال اجرا است 24/7 (بدون خواب/راه‌اندازی مجدد).
    - منطقه زمانی کار را بررسی کنید (`--tz` در برابر منطقه زمانی میزبان).

    اشکال‌زدایی:
    ```bash
    openclaw cron run <jobId>
    openclaw cron runs --id <jobId> --limit 50
    ```

    مستندات: [کارهای Cron](/fa/automation/cron-jobs)، [خودکارسازی](/fa/automation).

  </Accordion>

  <Accordion title="Cron اجرا شد، اما چرا چیزی به کانال ارسال نشد؟">
    حالت تحویل را بررسی کنید:

    - `--no-deliver` / `delivery.mode: "none"`: انتظار نمی‌رود ارسال جایگزین اجراکننده انجام شود.
    - هدف اعلان وجود ندارد یا نامعتبر است (`channel` / `to`): اجراکننده تحویل خروجی را نادیده گرفت.
    - خطاهای احراز هویت کانال (`unauthorized`، `Forbidden`): اجراکننده برای تحویل تلاش کرد، اما اعتبارنامه‌ها مانع آن شدند.
    - یک نتیجه ساکت و ایزوله (فقط `NO_REPLY` / `no_reply`) عمداً غیرقابل‌تحویل تلقی می‌شود؛ بنابراین تحویل جایگزین صف‌شده نیز سرکوب می‌شود.

    برای کارهای Cron ایزوله، اگر مسیر چت دردسترس باشد، عامل همچنان می‌تواند مستقیماً با ابزار `message` ارسال کند. `--announce` فقط تحویل جایگزین اجراکننده را برای متن نهایی‌ای کنترل می‌کند که عامل پیش‌تر خودش ارسال نکرده است.

    اشکال‌زدایی:
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    openclaw tasks show <lookup>
    ```

    مستندات: [کارهای Cron](/fa/automation/cron-jobs)، [وظایف پس‌زمینه](/fa/automation/tasks).

  </Accordion>

  <Accordion title="چرا یک اجرای Cron ایزوله مدل را تغییر داد یا یک‌بار دوباره تلاش کرد؟">
    این مسیر زنده تغییر مدل است، نه زمان‌بندی تکراری. Cron ایزوله واگذاری مدل در زمان اجرا را ماندگار می‌کند و وقتی اجرای فعال `LiveSessionModelSwitchError` را پرتاب کند، با حفظ ارائه‌دهنده/مدل تغییریافته (و هر بازنویسی تغییریافته نمایه احراز هویت) دوباره تلاش می‌کند.

    اولویت انتخاب مدل: ابتدا بازنویسی مدل قلاب Gmail (`hooks.gmail.model`)، سپس `model` مختص هر کار، بعد هر بازنویسی ذخیره‌شده مدل نشست Cron و در پایان انتخاب عادی مدل عامل/پیش‌فرض.

    حلقه تلاش مجدد به تلاش اولیه به‌علاوه 2 تلاش مجدد برای تغییر محدود است؛ سپس Cron به‌جای تکرار بی‌پایان متوقف می‌شود.

    اشکال‌زدایی:
    ```bash
    openclaw cron runs --id <jobId> --limit 50
    ```

    مستندات: [کارهای Cron](/fa/automation/cron-jobs)، [CLI مربوط به Cron](/fa/cli/cron).

  </Accordion>

  <Accordion title="چگونه Skills را روی Linux نصب کنم؟">
    از فرمان‌های بومی `openclaw skills` استفاده کنید یا Skills را در فضای کاری خود قرار دهید؛ رابط کاربری Skills در macOS روی Linux دردسترس نیست. Skills را در [https://clawhub.ai](https://clawhub.ai) مرور کنید.

    ```bash
    openclaw skills search "calendar"
    openclaw skills search --limit 20
    openclaw skills install @owner/<skill-slug>
    openclaw skills install @owner/<skill-slug> --version <version>
    openclaw skills install @owner/<skill-slug> --force
    openclaw skills install @owner/<skill-slug> --global
    openclaw skills update --all
    openclaw skills update --all --global
    openclaw skills list --eligible
    openclaw skills check
    ```

    `openclaw skills install` بومی به‌طور پیش‌فرض در پوشه `skills/` فضای کاری فعال می‌نویسد. برای نصب در پوشه مشترک و مدیریت‌شده Skills برای همه عامل‌های محلی، `--global` را اضافه کنید. CLI جداگانه `clawhub` را فقط برای انتشار یا همگام‌سازی Skills خودتان نصب کنید. برای محدودکردن عامل‌هایی که Skills مشترک را می‌بینند، از `agents.defaults.skills` یا `agents.entries.*.skills` استفاده کنید.

  </Accordion>

  <Accordion title="آیا OpenClaw می‌تواند وظایف را طبق برنامه یا به‌طور پیوسته در پس‌زمینه اجرا کند؟">
    بله، از طریق زمان‌بند Gateway:

    - **کارهای Cron** برای وظایف زمان‌بندی‌شده یا تکرارشونده (پس از راه‌اندازی مجدد نیز ماندگار می‌مانند).
    - **Heartbeat** برای بررسی‌های دوره‌ای نشست اصلی.
    - **کارهای ایزوله** برای عامل‌های خودمختاری که خلاصه‌ها را منتشر می‌کنند یا به چت‌ها تحویل می‌دهند.

    مستندات: [کارهای Cron](/fa/automation/cron-jobs)، [خودکارسازی](/fa/automation)، [Heartbeat](/fa/gateway/heartbeat).

  </Accordion>

  <Accordion title="آیا می‌توانم Skills مختص Apple macOS را از Linux اجرا کنم؟">
    نه به‌طور مستقیم. Skills مربوط به macOS با `metadata.openclaw.os` به‌همراه فایل‌های اجرایی لازم محدود می‌شوند و فقط وقتی روی **میزبان Gateway** واجد شرایط باشند بارگذاری می‌شوند. در Linux، Skills مختص `darwin` (`apple-notes`، `apple-reminders`، `things-mac`) بارگذاری نمی‌شوند، مگر اینکه این محدودیت را بازنویسی کنید.

    سه الگوی پشتیبانی‌شده:

    **گزینه A - Gateway را روی یک Mac اجرا کنید (ساده‌ترین)**. Gateway را جایی اجرا کنید که فایل‌های اجرایی macOS وجود دارند، سپس از Linux در [حالت راه‌دور](#gateway-ports-already-running-and-remote-mode) یا از طریق Tailscale متصل شوید. Skills به‌طور عادی بارگذاری می‌شوند، زیرا میزبان Gateway از macOS استفاده می‌کند.

    **گزینه B - از یک Node مبتنی بر macOS استفاده کنید (بدون SSH)**. Gateway را روی Linux اجرا کنید، یک Node مبتنی بر macOS (برنامه نوار منو) را جفت کنید و **Node Run Commands** را روی Mac به "Always Ask" یا "Always Allow" تنظیم کنید. وقتی فایل‌های اجرایی لازم روی Node وجود داشته باشند، OpenClaw ‏Skills مختص macOS را واجد شرایط در نظر می‌گیرد؛ عامل آن‌ها را از طریق ابزار `nodes` اجرا می‌کند. با "Always Ask"، تأیید "Always Allow" در اعلان، آن فرمان را به فهرست مجاز اضافه می‌کند.

    **گزینه C - فایل‌های اجرایی macOS را از طریق SSH واسطه‌گری کنید (پیشرفته)**. Gateway را روی Linux نگه دارید، اما کاری کنید فایل‌های اجرایی CLI لازم به پوشش‌های SSH نگاشت شوند که روی یک Mac اجرا می‌شوند؛ سپس Skill را بازنویسی کنید تا Linux را مجاز بداند و واجد شرایط باقی بماند.

    1. یک پوشش SSH برای فایل اجرایی ایجاد کنید (نمونه: `memo` برای Apple Notes):
       ```bash
       #!/usr/bin/env bash
       set -euo pipefail
       exec ssh -T user@mac-host /opt/homebrew/bin/memo "$@"
       ```
    2. پوشش را در `PATH` روی میزبان Linux قرار دهید (برای مثال `~/bin/memo`).
    3. فراداده Skill را (در فضای کاری یا `~/.openclaw/skills`) بازنویسی کنید تا Linux مجاز شود:
       ```markdown
       ---
       name: apple-notes
       description: یادداشت‌های Apple را از طریق CLI مربوط به memo در macOS مدیریت کنید.
       metadata: { "openclaw": { "os": ["darwin", "linux"], "requires": { "bins": ["memo"] } } }
       ---
       ```
    4. یک نشست جدید آغاز کنید تا تصویر لحظه‌ای Skills تازه‌سازی شود.

  </Accordion>

  <Accordion title="آیا ادغامی برای Notion یا HeyGen دارید؟">
    در حال حاضر به‌صورت داخلی وجود ندارد. گزینه‌ها:

    - **Skill / Plugin سفارشی**: بهترین گزینه برای دسترسی مطمئن به API (هر دو API دارند).
    - **خودکارسازی مرورگر**: بدون کدنویسی کار می‌کند، اما کندتر و آسیب‌پذیرتر است.

    برای زمینه مختص هر مشتری به سبک آژانس: برای هر مشتری یک صفحه Notion نگه دارید (زمینه + ترجیحات + کار فعال) و از عامل بخواهید در آغاز نشست آن صفحه را دریافت کند.

    برای یک ادغام بومی، درخواست قابلیت باز کنید یا Skillی بر پایه آن APIها بسازید.

    ```bash
    openclaw skills install @owner/<skill-slug>
    openclaw skills update --all
    ```

    نصب‌های بومی در پوشه `skills/` فضای کاری فعال قرار می‌گیرند؛ برای همه عامل‌های محلی از `--global` استفاده کنید، یا برای محدودکردن دسترسی `agents.defaults.skills` / `agents.entries.*.skills` را پیکربندی کنید. برخی Skills انتظار دارند فایل‌های اجرایی با Homebrew نصب شده باشند؛ در Linux این یعنی Linuxbrew.

    به [Skills](/fa/tools/skills)، [پیکربندی Skills](/fa/tools/skills-config)، [ClawHub](/tools/clawhub) مراجعه کنید.

  </Accordion>

  <Accordion title="چگونه از Chrome موجود خود که در آن وارد حساب شده‌ام با OpenClaw استفاده کنم؟">
    از نمایه مرورگر داخلی `user` استفاده کنید که از طریق Chrome DevTools MCP متصل می‌شود:

    ```bash
    openclaw browser --browser-profile user tabs
    openclaw browser --browser-profile user snapshot
    ```

    برای یک نام سفارشی، یک نمایه صریح MCP ایجاد کنید:

    ```bash
    openclaw browser create-profile --name chrome-live --driver existing-session
    openclaw browser --browser-profile chrome-live tabs
    ```

    این قابلیت می‌تواند از مرورگر میزبان محلی یا یک Node مرورگر متصل استفاده کند. اگر Gateway در جای دیگری اجرا می‌شود، یک میزبان Node روی دستگاه مرورگر اجرا کنید یا به‌جای آن از CDP راه‌دور استفاده کنید.

    محدودیت‌های فعلی نمایه‌های `existing-session` / `user` در مقایسه با نمایه مدیریت‌شده `openclaw`:

    - `click`، `type`، `hover`، `scrollIntoView`، `drag` و `select` به ارجاع‌های تصویر لحظه‌ای نیاز دارند، نه انتخابگرهای CSS.
    - قلاب‌های بارگذاری به `ref` یا `inputRef` نیاز دارند؛ هر بار یک فایل و بدون `element` مربوط به CSS.
    - `responsebody`، خروجی PDF، رهگیری دانلود و عملیات دسته‌ای همچنان به مسیر مرورگر مدیریت‌شده نیاز دارند.

    برای مقایسه کامل به [مرورگر](/fa/tools/browser#existing-session-via-chrome-devtools-mcp) مراجعه کنید.

  </Accordion>
</AccordionGroup>

## سندباکس و حافظه

<AccordionGroup>
  <Accordion title="آیا مستندات اختصاصی برای سندباکس وجود دارد؟">
    بله: [سندباکس](/fa/gateway/sandboxing). برای راه‌اندازی مختص Docker (Gateway کامل در Docker یا تصاویر سندباکس)، به [Docker](/fa/install/docker) مراجعه کنید.
  </Accordion>

  <Accordion title="Docker محدود به نظر می‌رسد؛ چگونه همه قابلیت‌ها را فعال کنم؟">
    تصویر پیش‌فرض ابتدا امنیت را در نظر می‌گیرد و با کاربر `node` اجرا می‌شود؛ بنابراین بسته‌های سیستم، Homebrew و مرورگرهای همراه را شامل نمی‌شود. برای راه‌اندازی کامل‌تر:

    - با `OPENCLAW_HOME_VOLUME`، ‏`/home/node` را ماندگار کنید تا حافظه‌های نهان حفظ شوند.
    - وابستگی‌های سیستم را با `OPENCLAW_IMAGE_APT_PACKAGES` درون تصویر بگنجانید.
    - مرورگرهای Playwright را از طریق CLI همراه نصب کنید: `node /app/node_modules/playwright-core/cli.js install chromium`.
    - `PLAYWRIGHT_BROWSERS_PATH` را تنظیم و آن مسیر را ماندگار کنید.

    مستندات: [Docker](/fa/install/docker)، [مرورگر](/fa/tools/browser).

  </Accordion>

  <Accordion title="آیا می‌توانم پیام‌های خصوصی را شخصی نگه دارم، اما گروه‌ها را با یک عامل عمومی/سندباکس‌شده کنم؟">
    بله، اگر ترافیک خصوصی **پیام‌های خصوصی** و ترافیک عمومی **گروه‌ها** باشند. `agents.defaults.sandbox.mode: "non-main"` را تنظیم کنید تا نشست‌های گروه/کانال (کلیدهای غیر اصلی) در بک‌اند سندباکس پیکربندی‌شده اجرا شوند، درحالی‌که نشست اصلی پیام خصوصی روی میزبان باقی می‌ماند. پس از فعال‌شدن سندباکس، Docker بک‌اند پیش‌فرض است. ابزارهای دردسترس در نشست‌های سندباکس‌شده را از طریق `tools.sandbox.tools` محدود کنید.

    راهنمای راه‌اندازی: [گروه‌ها: پیام‌های خصوصی شخصی + گروه‌های عمومی](/fa/channels/groups#pattern-personal-dms-public-groups-single-agent). مرجع اصلی: [پیکربندی Gateway](/fa/gateway/config-agents#agentsdefaultssandbox).

  </Accordion>

  <Accordion title="چگونه یک پوشه میزبان را به سندباکس متصل کنم؟">
    `agents.defaults.sandbox.docker.binds` را روی `["host:container:mode"]` تنظیم کنید (برای مثال `"/home/user/src:/src:ro"`). اتصال‌های سراسری و مختص عامل با هم ادغام می‌شوند؛ وقتی `scope: "shared"` باشد، اتصال‌های مختص عامل نادیده گرفته می‌شوند. برای هر مورد حساس از `:ro` استفاده کنید؛ اتصال‌ها از دیواره‌های سیستم فایل سندباکس عبور می‌کنند.

    OpenClaw منابع اتصال را هم در برابر مسیر نرمال‌شده و هم مسیر متعارفی که از طریق عمیق‌ترین نیای موجود حل شده است اعتبارسنجی می‌کند؛ بنابراین گریز از طریق والدِ پیوند نمادین حتی زمانی که بخش نهایی مسیر هنوز وجود ندارد نیز به‌صورت بسته و امن شکست می‌خورد.

    به [سندباکس](/fa/gateway/sandboxing#custom-bind-mounts) و [سندباکس در برابر خط‌مشی ابزار در برابر دسترسی ارتقایافته](/fa/gateway/sandbox-vs-tool-policy-vs-elevated#bind-mounts-security-quick-check) مراجعه کنید.

  </Accordion>

  <Accordion title="حافظه چگونه کار می‌کند؟">
    حافظه OpenClaw شامل فایل‌های Markdown در فضای کاری عامل است: یادداشت‌های روزانه در `memory/YYYY-MM-DD.md` و یادداشت‌های بلندمدت گزینش‌شده در `MEMORY.md` (فقط نشست‌های اصلی/خصوصی).

    OpenClaw همچنین پیش از آنکه Compaction مکالمه را خلاصه کند، یک **تخلیه بی‌صدای حافظه پیش از Compaction** اجرا می‌کند و به مدل یادآوری می‌کند که ابتدا یادداشت‌های ماندگار را بنویسد. این فرایند فقط زمانی اجرا می‌شود که فضای کاری قابل‌نوشتن باشد (سندباکس‌های فقط‌خواندنی آن را نادیده می‌گیرند)؛ برای غیرفعال‌کردن از `agents.defaults.compaction.memoryFlush.enabled: false` استفاده کنید. به [حافظه](/fa/concepts/memory) مراجعه کنید.

  </Accordion>

  <Accordion title="حافظه مدام چیزها را فراموش می‌کند. چگونه آن‌ها را ماندگار کنم؟">
    از ربات بخواهید **واقعیت را در حافظه بنویسد**: یادداشت‌های بلندمدت در `MEMORY.md` و زمینه کوتاه‌مدت در `memory/YYYY-MM-DD.md` قرار می‌گیرند. یادآوری به مدل برای ذخیره خاطرات معمولاً مشکل را برطرف می‌کند. اگر همچنان فراموش می‌کند، بررسی کنید که Gateway در هر اجرا از همان فضای کاری استفاده کند.

    مستندات: [حافظه](/fa/concepts/memory)، [فضای کاری عامل](/fa/concepts/agent-workspace).

  </Accordion>

  <Accordion title="آیا حافظه برای همیشه باقی می‌ماند؟ محدودیت‌ها چیست؟">
    فایل‌های حافظه روی دیسک قرار دارند و تا زمانی که حذف نشوند باقی می‌مانند؛ محدودیت، فضای ذخیره‌سازی شماست، نه مدل. **بافت نشست** همچنان به پنجرهٔ بافت مدل محدود است، بنابراین مکالمه‌های طولانی ممکن است فشرده یا بریده شوند؛ به همین دلیل جست‌وجوی حافظه وجود دارد و فقط بخش‌های مرتبط را دوباره وارد بافت می‌کند.

    مستندات: [حافظه](/fa/concepts/memory)، [بافت](/fa/concepts/context).

  </Accordion>

  <Accordion title="آیا جست‌وجوی معنایی حافظه به کلید API شرکت OpenAI نیاز دارد؟">
    فقط در صورتی که از **تعبیه‌های OpenAI** استفاده کنید که ارائه‌دهندهٔ پیش‌فرض است. Codex OAuth چت/تکمیل‌ها را پوشش می‌دهد و دسترسی به تعبیه‌ها را **اعطا نمی‌کند**، بنابراین ورود با Codex (از طریق OAuth یا ورود CLI مربوط به Codex) جست‌وجوی معنایی حافظه را فعال نمی‌کند. تعبیه‌های OpenAI همچنان به یک کلید API واقعی نیاز دارند (`OPENAI_API_KEY` یا `models.providers.openai.apiKey`).

    برای محلی ماندن، `memory.search.provider: "local"` (GGUF/llama.cpp) را تنظیم کنید. سایر ارائه‌دهندگان پشتیبانی‌شده: Bedrock، DeepInfra، Gemini (`GEMINI_API_KEY` یا `memory.search.remote.apiKey`)، GitHub Copilot، LM Studio، Mistral، Ollama، سازگار با OpenAI و Voyage. برای جزئیات راه‌اندازی، [حافظه](/fa/concepts/memory) و [جست‌وجوی حافظه](/fa/concepts/memory-search) را ببینید.

  </Accordion>
</AccordionGroup>

## محل قرارگیری موارد روی دیسک

<AccordionGroup>
  <Accordion title="آیا همهٔ داده‌های استفاده‌شده با OpenClaw به‌صورت محلی ذخیره می‌شوند؟">
    خیر: **وضعیت متعلق به خود OpenClaw محلی است**، اما **سرویس‌های خارجی همچنان آنچه را برایشان ارسال می‌کنید می‌بینند**.

    - **به‌طور پیش‌فرض محلی**: نشست‌ها، فایل‌های حافظه، پیکربندی و فضای کاری روی میزبان Gateway قرار دارند (`~/.openclaw` به‌همراه پوشهٔ فضای کاری شما).
    - **بنا بر ضرورت راه‌دور**: پیام‌های ارسال‌شده به ارائه‌دهندگان مدل (Anthropic/OpenAI/و غیره) به APIهای آن‌ها می‌روند و پلتفرم‌های گفت‌وگو (Slack/Telegram/WhatsApp/و غیره) داده‌های پیام را روی سرورهای خود ذخیره می‌کنند.
    - **ردپای داده را شما کنترل می‌کنید**: مدل‌های محلی درخواست‌ها را روی دستگاه شما نگه می‌دارند، اما ترافیک کانال همچنان از سرورهای کانال عبور می‌کند.

    مرتبط: [فضای کاری عامل](/fa/concepts/agent-workspace)، [حافظه](/fa/concepts/memory).

  </Accordion>

  <Accordion title="OpenClaw داده‌هایش را کجا ذخیره می‌کند؟">
    همه‌چیز در `$OPENCLAW_STATE_DIR` قرار دارد (پیش‌فرض: `~/.openclaw`):

    | مسیر                                                               | هدف                                                            |
    | ------------------------------------------------------------------ | ------------------------------------------------------------------ |
    | `$OPENCLAW_STATE_DIR/openclaw.json`                                 | پیکربندی اصلی (JSON5)                                                 |
    | `$OPENCLAW_STATE_DIR/credentials/oauth.json`                        | درون‌ریزی قدیمی OAuth (در نخستین استفاده در نمایه‌های احراز هویت کپی می‌شود)        |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth-profiles.json`     | نمایه‌های احراز هویت (OAuth، کلیدهای API، `keyRef`/`tokenRef` اختیاری)        |
    | `$OPENCLAW_STATE_DIR/secrets.json`                                  | محتوای محرمانهٔ اختیاریِ مبتنی بر فایل برای ارائه‌دهندگان SecretRef مربوط به `file`   |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/auth.json`              | فایل سازگاری قدیمی (ورودی‌های ایستای `api_key` پاک‌سازی شده‌اند)        |
    | `$OPENCLAW_STATE_DIR/credentials/`                                  | وضعیت ارائه‌دهنده (برای مثال `whatsapp/<accountId>/creds.json`)      |
    | `$OPENCLAW_STATE_DIR/agents/`                                       | وضعیت هر عامل (agentDir به‌همراه آثار قدیمی/بایگانی‌شدهٔ نشست)        |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/agent/openclaw-agent.sqlite`  | وضعیت SQLite هر عامل، شامل ردیف‌های نشست و رونوشت‌ها      |
    | `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/`                    | منابع مهاجرت نشست قدیمی و آثار بایگانی/پشتیبانی      |

    مسیر قدیمی تک‌عاملی `~/.openclaw/agent/*` توسط `openclaw doctor` مهاجرت می‌شود.

    **فضای کاری** شما (AGENTS.md، فایل‌های حافظه، مهارت‌ها و غیره) جداست و از طریق `agents.defaults.workspace` پیکربندی می‌شود (پیش‌فرض: `~/.openclaw/workspace`).

  </Accordion>

  <Accordion title="AGENTS.md / SOUL.md / USER.md / MEMORY.md باید کجا قرار بگیرند؟">
    این فایل‌ها در **فضای کاری عامل** قرار می‌گیرند، نه در `~/.openclaw`.

    - **فضای کاری (برای هر عامل)**: `AGENTS.md`، `SOUL.md`، `IDENTITY.md`، `USER.md`، `MEMORY.md`، `memory/YYYY-MM-DD.md`، و `HEARTBEAT.md` اختیاری. ریشهٔ حروف‌کوچک `memory.md` فقط ورودی تعمیر قدیمی است؛ وقتی هر دو وجود داشته باشند، `openclaw doctor --fix` می‌تواند آن را در `MEMORY.md` ادغام کند.
    - **پوشهٔ وضعیت (`~/.openclaw`)**: پیکربندی، وضعیت کانال/ارائه‌دهنده، نمایه‌های احراز هویت، نشست‌ها، گزارش‌ها و مهارت‌های مشترک (`~/.openclaw/skills`).

    فضای کاری پیش‌فرض `~/.openclaw/workspace` است و قابل پیکربندی است:

    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
    }
    ```

    اگر ربات پس از راه‌اندازی مجدد «فراموش می‌کند»، تأیید کنید که Gateway در هر اجرا از همان فضای کاری استفاده می‌کند (حالت راه‌دور از فضای کاری **میزبان Gateway** استفاده می‌کند، نه لپ‌تاپ محلی شما).

    نکته: برای رفتار یا ترجیح ماندگار، به‌جای تکیه بر تاریخچهٔ گفت‌وگو از ربات بخواهید **آن را در AGENTS.md یا MEMORY.md بنویسد**.

    [فضای کاری عامل](/fa/concepts/agent-workspace) و [حافظه](/fa/concepts/memory) را ببینید.

  </Accordion>

  <Accordion title="آیا می‌توانم SOUL.md را بزرگ‌تر کنم؟">
    بله. `SOUL.md` یکی از فایل‌های راه‌انداز فضای کاری است که به بافت عامل تزریق می‌شود. محدودیت پیش‌فرض تزریق برای هر فایل `20000` نویسه است؛ بودجهٔ کل راه‌اندازی در همهٔ فایل‌ها `60000` نویسه است.

    پیش‌فرض‌های مشترک را تغییر دهید:

    ```json5
    {
      agents: {
        defaults: {
          bootstrapMaxChars: 50000,
          bootstrapTotalMaxChars: 300000,
        },
      },
    }
    ```

    یا تنظیم یک عامل را در `agents.entries.*.bootstrapMaxChars` / `bootstrapTotalMaxChars` بازنویسی کنید.

    برای بررسی اندازه‌های خام در برابر اندازه‌های تزریق‌شده و اینکه آیا برش رخ داده است، از `/context` استفاده کنید. `SOUL.md` را بر صدا، موضع و شخصیت متمرکز نگه دارید؛ قواعد اجرایی را در `AGENTS.md` و واقعیت‌های ماندگار را در حافظه قرار دهید.

    [بافت](/fa/concepts/context) و [پیکربندی عامل](/fa/gateway/config-agents) را ببینید.

  </Accordion>

  <Accordion title="راهبرد پیشنهادی پشتیبان‌گیری">
    **فضای کاری عامل** خود را در یک مخزن git **خصوصی** قرار دهید و در مکانی خصوصی از آن پشتیبان بگیرید (برای مثال GitHub خصوصی). این کار حافظه به‌همراه فایل‌های AGENTS/SOUL/USER را ثبت می‌کند و به شما امکان می‌دهد بعداً «ذهن» دستیار را بازیابی کنید.

    هیچ‌چیز از زیر `~/.openclaw` را ثبت نکنید (اعتبارنامه‌ها، نشست‌ها، توکن‌ها، محتوای رمزگذاری‌شدهٔ اسرار). برای بازیابی کامل، از فضای کاری و پوشهٔ وضعیت به‌صورت جداگانه پشتیبان بگیرید.

    مستندات: [فضای کاری عامل](/fa/concepts/agent-workspace).

  </Accordion>

  <Accordion title="چگونه OpenClaw را به‌طور کامل حذف نصب کنم؟">
    [حذف نصب](/fa/install/uninstall) را ببینید.
  </Accordion>

  <Accordion title="آیا عامل‌ها می‌توانند خارج از فضای کاری فعالیت کنند؟">
    بله. فضای کاری **cwd پیش‌فرض** و لنگر حافظه است، نه یک جعبهٔ شنی سخت‌گیرانه. مسیرهای نسبی درون فضای کاری حل می‌شوند؛ مسیرهای مطلق می‌توانند به سایر مکان‌های میزبان دسترسی داشته باشند، مگر اینکه جعبهٔ شنی فعال باشد. برای جداسازی، از [`agents.defaults.sandbox`](/fa/gateway/sandboxing) یا تنظیمات جعبهٔ شنی هر عامل استفاده کنید. برای اینکه یک مخزن، پوشهٔ کاری پیش‌فرض باشد، `workspace` آن عامل را به ریشهٔ مخزن اشاره دهید؛ خود مخزن OpenClaw فقط کد منبع است، بنابراین فضای کاری را جدا نگه دارید، مگر اینکه عمداً بخواهید عامل درون آن کار کند.

    ```json5
    {
      agents: {
        defaults: {
          workspace: "~/Projects/my-repo",
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="حالت راه‌دور: محل ذخیرهٔ نشست کجاست؟">
    وضعیت نشست متعلق به **میزبان Gateway** است. در حالت راه‌دور، محل ذخیرهٔ نشست موردنظر شما روی دستگاه راه‌دور قرار دارد، نه لپ‌تاپ محلی شما. [مدیریت نشست](/fa/concepts/session) را ببینید.
  </Accordion>
</AccordionGroup>

## مبانی پیکربندی

<AccordionGroup>
  <Accordion title="قالب پیکربندی چیست؟ کجا قرار دارد؟">
    OpenClaw یک پیکربندی اختیاری **JSON5** را از `$OPENCLAW_CONFIG_PATH` می‌خواند (پیش‌فرض: `~/.openclaw/openclaw.json`). اگر فایل وجود نداشته باشد، از پیش‌فرض‌های نسبتاً امن، از جمله فضای کاری پیش‌فرض `~/.openclaw/workspace`، استفاده می‌کند.
  </Accordion>

  <Accordion title='gateway.bind: "lan" (یا "tailnet") را تنظیم کردم و حالا چیزی گوش نمی‌دهد / رابط کاربری می‌گوید مجاز نیست'>
    اتصال‌های غیر-loopback **به یک مسیر معتبر احراز هویت Gateway نیاز دارند**: احراز هویت با راز مشترک (توکن یا گذرواژه)، یا `gateway.auth.mode: "trusted-proxy"` پشت یک پراکسی معکوس آگاه از هویت که به‌درستی پیکربندی شده باشد.

    ```json5
    {
      gateway: {
        bind: "lan",
        auth: {
          mode: "token",
          token: "replace-me",
        },
      },
    }
    ```

    - `gateway.remote.token` / `.password` به‌تنهایی احراز هویت Gateway محلی را فعال **نمی‌کنند**؛ مسیرهای فراخوانی محلی فقط وقتی `gateway.auth.*` تنظیم نشده باشد می‌توانند از `gateway.remote.*` به‌عنوان جایگزین استفاده کنند.
    - برای احراز هویت با گذرواژه، `gateway.auth.mode: "password"` را به‌همراه `gateway.auth.password` (یا `OPENCLAW_GATEWAY_PASSWORD`) تنظیم کنید.
    - اگر `gateway.auth.token` / `.password` به‌صراحت از طریق SecretRef پیکربندی شده باشد و قابل حل نباشد، فرایند حل به‌صورت بسته شکست می‌خورد (هیچ جایگزین راه‌دوری آن را پنهان نمی‌کند).
    - راه‌اندازی‌های Control UI با راز مشترک از طریق `connect.params.auth.token` یا `connect.params.auth.password` احراز هویت می‌کنند (در تنظیمات برنامه/رابط کاربری ذخیره می‌شود). حالت‌های دارای هویت مانند Tailscale Serve یا `trusted-proxy` در عوض از سرآیندهای درخواست استفاده می‌کنند؛ از قرار دادن رازهای مشترک در URLها خودداری کنید.
    - با `gateway.auth.mode: "trusted-proxy"`، پراکسی‌های معکوس loopback روی همان میزبان به `gateway.auth.trustedProxy.allowLoopback = true` صریح و یک ورودی loopback در `gateway.trustedProxies` نیاز دارند.

  </Accordion>

  <Accordion title="چرا اکنون روی localhost به توکن نیاز دارم؟">
    OpenClaw احراز هویت Gateway را به‌طور پیش‌فرض، از جمله برای loopback، اعمال می‌کند. اگر هیچ مسیر احراز هویت صریحی پیکربندی نشده باشد، هنگام راه‌اندازی حالت توکن انتخاب می‌شود و برای همان راه‌اندازی یک توکن صرفاً زمان‌اجرا تولید می‌گردد؛ بنابراین کلاینت‌های WS محلی باید احراز هویت کنند. این کار مانع فراخوانی Gateway توسط سایر فرایندهای محلی می‌شود.

    وقتی کلاینت‌ها به یک راز پایدار میان راه‌اندازی‌های مجدد نیاز دارند، `gateway.auth.token`، `gateway.auth.password`، `OPENCLAW_GATEWAY_TOKEN` یا `OPENCLAW_GATEWAY_PASSWORD` را به‌صراحت پیکربندی کنید. همچنین می‌توانید حالت گذرواژه یا `trusted-proxy` را برای پراکسی‌های معکوس آگاه از هویت انتخاب کنید. برای loopback باز، `gateway.auth.mode: "none"` را به‌صراحت تنظیم کنید. `openclaw doctor --generate-gateway-token` هر زمان یک توکن تولید می‌کند.

  </Accordion>

  <Accordion title="آیا پس از تغییر پیکربندی باید راه‌اندازی مجدد انجام دهم؟">
    Gateway پیکربندی را زیر نظر دارد و از بارگذاری مجدد گرم پشتیبانی می‌کند: `gateway.reload.mode: "hybrid"` (پیش‌فرض) تغییرات امن را به‌صورت گرم اعمال می‌کند و برای تغییرات بحرانی راه‌اندازی مجدد انجام می‌دهد. `hot`، `restart` و `off` نیز پشتیبانی می‌شوند. بیشتر تغییرات `tools.*`، خط‌مشی `agents.*`، `session.*` و `messages.*` بلافاصله و بدون هیچ اقدام بارگذاری مجدد اعمال می‌شوند؛ تغییرات اتصال/درگاه `gateway.*` به راه‌اندازی مجدد نیاز دارند.
  </Accordion>

  <Accordion title="چگونه جست‌وجوی وب (و واکشی وب) را فعال کنم؟">
    `web_fetch` بدون کلید API کار می‌کند. `web_search` به ارائه‌دهندهٔ انتخابی شما بستگی دارد:

    | ارائه‌دهنده | بدون نیاز به کلید | متغیر(های) محیطی |
    | --- | --- | --- |
    | Brave | خیر | `BRAVE_API_KEY` |
    | DuckDuckGo | بله (غیررسمی و مبتنی بر HTML) | - |
    | Exa | خیر | `EXA_API_KEY` |
    | Firecrawl | خیر | `FIRECRAWL_API_KEY` |
    | Gemini | خیر | `GEMINI_API_KEY` |
    | Grok | خیر (OAuth مربوط به xAI یا کلید) | `XAI_API_KEY` |
    | Kimi | خیر | `KIMI_API_KEY` یا `MOONSHOT_API_KEY` |
    | MiniMax Search | خیر | `MINIMAX_CODE_PLAN_KEY`، `MINIMAX_CODING_API_KEY` یا `MINIMAX_API_KEY` |
    | Ollama Web Search | بله (به `ollama signin` نیاز دارد) | - |
    | Perplexity | خیر | `PERPLEXITY_API_KEY` یا `OPENROUTER_API_KEY` |
    | SearXNG | بله (خودمیزبان) | `SEARXNG_BASE_URL` |
    | Tavily | خیر | `TAVILY_API_KEY` |

    Grok همچنین می‌تواند از OAuth مربوط به xAI در احراز هویت مدل (`openclaw onboard --auth-choice xai-oauth`) دوباره استفاده کند.

    **پیشنهادشده**: `openclaw configure --section web` و یک ارائه‌دهنده انتخاب کنید.

    ```json5
    {
      plugins: {
        entries: {
          brave: {
            config: {
              webSearch: {
                apiKey: "BRAVE_API_KEY_HERE",
              },
            },
          },
        },
      },
      tools: {
        web: {
          search: {
            enabled: true,
            provider: "brave",
            maxResults: 5,
          },
          fetch: {
            enabled: true,
            provider: "firecrawl", // اختیاری؛ برای تشخیص خودکار حذف کنید
          },
        },
      },
    }
    ```

    پیکربندی جست‌وجوی وب مختص ارائه‌دهنده در `plugins.entries.<plugin>.config.webSearch.*` قرار دارد. مسیرهای قدیمی ارائه‌دهنده در `tools.web.search.*` همچنان برای سازگاری بارگذاری می‌شوند، اما نباید در پیکربندی‌های جدید استفاده شوند. پیکربندی جایگزین واکشی وب Firecrawl در `plugins.entries.firecrawl.config.webFetch.*` قرار دارد.

    - فهرست‌های مجاز: `web_search`/`web_fetch`/`x_search`، یا `group:web` را برای هر سه مورد اضافه کنید.
    - `web_fetch` به‌طور پیش‌فرض فعال است.
    - اگر `tools.web.fetch.provider` حذف شده باشد، OpenClaw نخستین ارائه‌دهنده جایگزین آماده برای واکشی را به‌طور خودکار از روی اعتبارنامه‌های موجود تشخیص می‌دهد؛ Plugin رسمی Firecrawl این جایگزین را فراهم می‌کند.
    - دیمون‌ها متغیرهای محیطی را از `~/.openclaw/.env` (یا محیط سرویس) می‌خوانند.

    مستندات: [ابزارهای وب](/fa/tools/web).

  </Accordion>

  <Accordion title="config.apply پیکربندی من را پاک کرد. چگونه آن را بازیابی و از تکرار این اتفاق جلوگیری کنم؟">
    `config.apply` **کل پیکربندی** را جایگزین می‌کند؛ یک شیء جزئی هر چیز دیگری را حذف می‌کند.

    نسخه فعلی OpenClaw از بیشتر بازنویسی‌های تصادفی محافظت می‌کند:

    - نوشتن پیکربندی تحت مالکیت OpenClaw، پیش از نوشتن، کل پیکربندی حاصل از تغییر را اعتبارسنجی می‌کند.
    - نوشتن نامعتبر یا مخرب تحت مالکیت OpenClaw رد و با نام `openclaw.json.rejected.*` ذخیره می‌شود.
    - ویرایش مستقیمی که راه‌اندازی یا بارگذاری مجدد داغ را مختل کند، باعث می‌شود Gateway با حالت بسته امن شکست بخورد یا از بارگذاری مجدد صرف‌نظر کند؛ `openclaw.json` را بازنویسی نمی‌کند.
    - `openclaw doctor --fix` مسئول تعمیر است، می‌تواند آخرین نسخه سالم شناخته‌شده را بازیابی کند و فایل ردشده را با نام `openclaw.json.clobbered.*` ذخیره می‌کند.

    بازیابی:

    - `openclaw logs --follow` را برای یافتن `Invalid config at`، `Config write rejected:` یا `config reload skipped (invalid config)` بررسی کنید.
    - جدیدترین `openclaw.json.clobbered.*` یا `openclaw.json.rejected.*` را در کنار پیکربندی فعال بررسی کنید.
    - `openclaw config validate` و `openclaw doctor --fix` را اجرا کنید.
    - فقط کلیدهای موردنظر را با `openclaw config set` یا `config.patch` برگردانید.
    - اگر آخرین نسخه سالم شناخته‌شده یا محتوای ردشده موجود نیست: از نسخه پشتیبان بازیابی کنید، یا `openclaw doctor` را دوباره اجرا و کانال‌ها/مدل‌ها را مجدداً پیکربندی کنید.
    - در صورت فقدان غیرمنتظره: همراه با آخرین پیکربندی شناخته‌شده یا یک نسخه پشتیبان، گزارش اشکال ثبت کنید. یک عامل کدنویسی محلی اغلب می‌تواند پیکربندی کارآمدی را از گزارش‌ها یا تاریخچه بازسازی کند.

    برای جلوگیری از آن: برای تغییرات کوچک از `openclaw config set`، برای ویرایش تعاملی از `openclaw configure`، برای بررسی مسیری ناآشنا از `config.schema.lookup` (یک گره کم‌عمق طرح‌واره به‌همراه خلاصه فرزندان مستقیم را برمی‌گرداند) و برای ویرایش‌های جزئی RPC از `config.patch` استفاده کنید؛ `config.apply` را فقط برای جایگزینی کامل پیکربندی نگه دارید. ابزار زمان اجرای `gateway` که در اختیار عامل است، حتی از طریق نام‌های مستعار قدیمی `tools.bash.*` نیز از بازنویسی `tools.exec.ask` / `tools.exec.security` خودداری می‌کند.

    مستندات: [پیکربندی](/fa/cli/config)، [پیکربندی تعاملی](/fa/cli/configure)، [عیب‌یابی Gateway](/fa/gateway/troubleshooting#gateway-rejected-invalid-config)، [Doctor](/fa/gateway/doctor).

  </Accordion>

  <Accordion title="چگونه یک Gateway مرکزی را با کارگرهای تخصصی در دستگاه‌های مختلف اجرا کنم؟">
    الگوی رایج: **یک Gateway** (برای مثال Raspberry Pi) به‌همراه **Nodeها** و **عامل‌ها**.

    - **Gateway (مرکزی)**: مالک کانال‌ها (Signal/WhatsApp)، مسیریابی و نشست‌ها است.
    - **Nodeها (دستگاه‌ها)**: مک‌ها/iOS/Android به‌عنوان تجهیزات جانبی متصل می‌شوند و ابزارهای محلی (`system.run`، `canvas`، `camera`) را ارائه می‌کنند.
    - **عامل‌ها (کارگرها)**: مغزها/فضاهای کاری جداگانه برای نقش‌های ویژه (برای مثال عملیات در برابر داده‌های شخصی).
    - **زیرعامل‌ها**: برای پردازش موازی، کارهای پس‌زمینه را از یک عامل اصلی ایجاد می‌کنند.
    - **TUI**: به Gateway متصل شوید و میان عامل‌ها/نشست‌ها جابه‌جا شوید.

    مستندات: [Nodeها](/fa/nodes)، [دسترسی از راه دور](/fa/gateway/remote)، [مسیریابی چندعاملی](/fa/concepts/multi-agent)، [زیرعامل‌ها](/fa/tools/subagents)، [TUI](/fa/web/tui).

  </Accordion>

  <Accordion title="آیا مرورگر OpenClaw می‌تواند بدون رابط گرافیکی اجرا شود؟">
    بله:

    ```json5
    {
      browser: { headless: true },
      agents: {
        defaults: {
          sandbox: { browser: { headless: true } },
        },
      },
    }
    ```

    مقدار پیش‌فرض `false` (با رابط گرافیکی) است. حالت بدون رابط گرافیکی در برخی سایت‌ها احتمال بیشتری دارد که بررسی‌های ضدربات را فعال کند (X/Twitter اغلب نشست‌های بدون رابط گرافیکی را مسدود می‌کند). این حالت از همان موتور Chromium استفاده می‌کند و برای بیشتر خودکارسازی‌ها کار می‌کند؛ تفاوت اصلی، نبود پنجره قابل‌مشاهده مرورگر است (برای مشاهده از اسکرین‌شات استفاده کنید). [مرورگر](/fa/tools/browser) را ببینید.

  </Accordion>

  <Accordion title="چگونه از Brave برای کنترل مرورگر استفاده کنم؟">
    `browser.executablePath` را روی فایل اجرایی Brave خود (یا هر مرورگر مبتنی بر Chromium) تنظیم کنید و Gateway را دوباره راه‌اندازی کنید. [مرورگر](/fa/tools/browser#use-brave-or-another-chromium-based-browser) را ببینید.
  </Accordion>
</AccordionGroup>

## Gatewayها و Nodeهای راه دور

<AccordionGroup>
  <Accordion title="فرمان‌ها چگونه میان Telegram، Gateway و Nodeها منتقل می‌شوند؟">
    پیام‌های Telegram توسط **Gateway** مدیریت می‌شوند؛ Gateway عامل را اجرا می‌کند و تنها پس از آن، در صورت نیاز به ابزار Node، از طریق **Gateway WebSocket**، Nodeها را فراخوانی می‌کند:

    Telegram -> Gateway -> عامل -> `node.*` -> Node -> Gateway -> Telegram

    Nodeها ترافیک ورودی ارائه‌دهنده را نمی‌بینند؛ آن‌ها فقط فراخوانی‌های RPC مربوط به Node را دریافت می‌کنند.

  </Accordion>

  <Accordion title="اگر Gateway از راه دور میزبانی شود، عامل من چگونه می‌تواند به رایانه‌ام دسترسی پیدا کند؟">
    رایانه خود را به‌عنوان یک **Node** جفت کنید. Gateway در جای دیگری اجرا می‌شود، اما می‌تواند ابزارهای `node.*` (صفحه‌نمایش، دوربین، سیستم) را از طریق Gateway WebSocket روی دستگاه محلی شما فراخوانی کند.

    1. Gateway را روی میزبان همیشه‌روشن (VPS/سرور خانگی) اجرا کنید.
    2. میزبان Gateway و رایانه خود را روی یک tailnet قرار دهید.
    3. مطمئن شوید Gateway WS قابل‌دسترسی است (اتصال به tailnet یا تونل SSH).
    4. برنامه macOS را به‌صورت محلی باز کنید و در حالت **Remote over SSH** (یا مستقیماً از طریق tailnet) متصل شوید تا به‌عنوان Node ثبت شود.
    5. Node را تأیید کنید:
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    پل TCP جداگانه‌ای لازم نیست؛ Nodeها از طریق Gateway WebSocket متصل می‌شوند.

    یادآوری امنیتی: جفت‌کردن یک Node مبتنی بر macOS، `system.run` را روی آن دستگاه مجاز می‌کند. فقط دستگاه‌های مورداعتماد را جفت کنید؛ [امنیت](/fa/gateway/security) را بررسی کنید.

    مستندات: [Nodeها](/fa/nodes)، [پروتکل Gateway](/fa/gateway/protocol)، [حالت راه دور macOS](/fa/platforms/mac/remote)، [امنیت](/fa/gateway/security).

  </Accordion>

  <Accordion title="Tailscale متصل است، اما پاسخی دریافت نمی‌کنم. اکنون چه کنم؟">
    موارد پایه را بررسی کنید:

    ```bash
    openclaw gateway status
    openclaw status
    openclaw channels status
    ```

    سپس احراز هویت و مسیریابی را بررسی کنید: اگر از Tailscale Serve استفاده می‌کنید، تأیید کنید `gateway.auth.allowTailscale` درست تنظیم شده است؛ اگر از طریق تونل SSH متصل می‌شوید، تأیید کنید تونل فعال است و به درگاه درست اشاره می‌کند؛ همچنین تأیید کنید فهرست‌های مجاز پیام مستقیم/گروه شامل حساب شما هستند.

    مستندات: [Tailscale](/fa/gateway/tailscale)، [دسترسی از راه دور](/fa/gateway/remote)، [کانال‌ها](/fa/channels).

  </Accordion>

  <Accordion title="آیا دو نمونه OpenClaw می‌توانند با یکدیگر ارتباط برقرار کنند (محلی + VPS)؟">
    بله، هرچند پل داخلی ربات‌به‌ربات وجود ندارد.

    **ساده‌ترین روش**: از یک کانال گفت‌وگوی معمولی استفاده کنید که هر دو ربات به آن دسترسی دارند (Slack/Telegram/WhatsApp). از ربات A بخواهید به ربات B پیام دهد، سپس اجازه دهید ربات B طبق معمول پاسخ دهد.

    **پل CLI (عمومی)**: اسکریپتی اجرا کنید که Gateway دیگر را با `openclaw agent --message ... --deliver` فراخوانی کند و گفت‌وگویی را هدف بگیرد که ربات دیگر در آن شنونده است. اگر یکی از ربات‌ها روی یک VPS راه دور قرار دارد، CLI خود را از طریق SSH/Tailscale به آن Gateway راه دور متصل کنید ([دسترسی از راه دور](/fa/gateway/remote) را ببینید):

    ```bash
    openclaw agent --message "سلام از ربات محلی" --deliver --channel telegram --reply-to <chat-id>
    ```

    یک محدودیت حفاظتی اضافه کنید تا دو ربات وارد حلقه بی‌پایان نشوند (فقط با اشاره، فهرست‌های مجاز کانال یا قانون «به پیام‌های ربات پاسخ نده»).

    مستندات: [دسترسی از راه دور](/fa/gateway/remote)، [CLI عامل](/fa/cli/agent)، [ارسال عامل](/fa/tools/agent-send).

  </Accordion>

  <Accordion title="آیا برای چند عامل به VPSهای جداگانه نیاز دارم؟">
    خیر. یک Gateway چند عامل را میزبانی می‌کند که هرکدام فضای کاری، پیش‌فرض‌های مدل و مسیریابی خود را دارند؛ این راه‌اندازی معمول است و نسبت به یک VPS برای هر عامل بسیار ارزان‌تر و ساده‌تر است. فقط برای جداسازی سخت‌گیرانه (مرزهای امنیتی) یا پیکربندی‌های بسیار متفاوتی که نمی‌خواهید به‌اشتراک گذاشته شوند، از VPSهای جداگانه استفاده کنید.
  </Accordion>

  <Accordion title="آیا استفاده از Node روی لپ‌تاپ شخصی‌ام به‌جای SSH از یک VPS مزیتی دارد؟">
    بله: Nodeها روش درجه‌یک برای دسترسی از Gateway راه دور به لپ‌تاپ شما هستند و قابلیت‌هایی فراتر از دسترسی پوسته فراهم می‌کنند. Gateway روی macOS/Linux (و Windows از طریق WSL2) اجرا می‌شود و سبک است (یک VPS کوچک یا دستگاهی هم‌رده Raspberry Pi کافی است؛ 4 GB رم کاملاً کافی است)، بنابراین راه‌اندازی رایج شامل یک میزبان همیشه‌روشن و لپ‌تاپ شما به‌عنوان Node است.

    - **به SSH ورودی نیازی نیست** - Nodeها از طریق جفت‌سازی دستگاه، اتصال خروجی به Gateway WebSocket برقرار می‌کنند.
    - **کنترل‌های اجرای امن‌تر** - `system.run` با فهرست‌های مجاز/تأییدهای Node روی آن لپ‌تاپ محدود می‌شود.
    - **ابزارهای بیشتر دستگاه** - Nodeها افزون بر `system.run`، ابزارهای `canvas`، `camera` و `screen` را ارائه می‌کنند.
    - **خودکارسازی مرورگر محلی** - Gateway را روی یک VPS نگه دارید، اما Chrome را به‌صورت محلی از طریق یک میزبان Node اجرا کنید، یا از طریق Chrome MCP به Chrome محلی متصل شوید.

    SSH برای دسترسی موردی به پوسته مناسب است؛ Nodeها برای گردش‌کارهای مداوم عامل و خودکارسازی دستگاه ساده‌تر هستند.

    مستندات: [Nodeها](/fa/nodes)، [CLI مربوط به Nodeها](/fa/cli/nodes)، [مرورگر](/fa/tools/browser).

  </Accordion>

  <Accordion title="آیا Nodeها یک سرویس Gateway اجرا می‌کنند؟">
    خیر. در هر میزبان فقط باید **یک Gateway** اجرا شود، مگر اینکه عمداً پروفایل‌های جداشده اجرا کنید ([چند Gateway](/fa/gateway/multiple-gateways) را ببینید). Nodeها تجهیزات جانبی متصل‌شونده به Gateway هستند (Nodeهای iOS/Android یا «حالت Node» macOS در برنامه نوار منو). برای میزبان‌های Node بدون رابط گرافیکی و کنترل CLI، [CLI میزبان Node](/fa/cli/node) را ببینید.

    برای تغییرات `gateway`، `discovery` و سطوح Plugin میزبانی‌شده، راه‌اندازی مجدد کامل لازم است.

  </Accordion>

  <Accordion title="آیا روشی مبتنی بر API / RPC برای اعمال پیکربندی وجود دارد؟">
    بله:

    - `config.schema.lookup`: پیش از نوشتن، یک زیردرخت پیکربندی را همراه با گره کم‌عمق طرح‌واره، راهنمای رابط کاربری منطبق و خلاصه فرزندان مستقیم آن بررسی می‌کند.
    - `config.get`: عکس فوری فعلی را به‌همراه هش دریافت می‌کند.
    - `config.patch`: به‌روزرسانی جزئی امن (ترجیحی برای بیشتر ویرایش‌های RPC)؛ در صورت امکان بارگذاری مجدد داغ و در صورت لزوم راه‌اندازی مجدد می‌کند.
    - `config.apply`: کل پیکربندی را اعتبارسنجی و جایگزین می‌کند؛ در صورت امکان بارگذاری مجدد داغ و در صورت لزوم راه‌اندازی مجدد می‌کند.
    - ابزار زمان اجرای `gateway` که در اختیار عامل است، همچنان از بازنویسی `tools.exec.ask` / `tools.exec.security` خودداری می‌کند؛ نام‌های مستعار قدیمی `tools.bash.*` به همان مسیرهای محافظت‌شده نرمال‌سازی می‌شوند.

  </Accordion>

  <Accordion title="حداقل پیکربندی معقول برای نخستین نصب">
    ```json5
    {
      agents: { defaults: { workspace: "~/.openclaw/workspace" } },
      channels: { whatsapp: { allowFrom: ["+15555550123"] } },
    }
    ```

    فضای کاری شما را تنظیم و افرادی را که می‌توانند ربات را فعال کنند محدود می‌کند.

  </Accordion>

  <Accordion title="چگونه Tailscale را روی یک VPS راه‌اندازی کنم و از Mac خود متصل شوم؟">
    1. **نصب و ورود به سیستم روی VPS**:
       ```bash
       curl -fsSL https://tailscale.com/install.sh | sh
       sudo tailscale up
       ```
    2. با استفاده از برنامه Tailscale و همان tailnet، **روی Mac خود نصب و وارد سیستم شوید**.
    3. در کنسول مدیریت Tailscale، **MagicDNS را فعال کنید** تا VPS نامی پایدار داشته باشد.
    4. **از نام میزبان tailnet استفاده کنید**: SSH `ssh user@your-vps.tailnet-xxxx.ts.net`؛ WS مربوط به Gateway‏ `ws://your-vps.tailnet-xxxx.ts.net:18789`.

    برای استفاده از رابط کاربری کنترل بدون SSH، از Tailscale Serve روی VPS استفاده کنید:

    ```bash
    openclaw gateway --tailscale serve
    ```

    این کار Gateway را متصل به loopback نگه می‌دارد و HTTPS را از طریق Tailscale در دسترس قرار می‌دهد. به [Tailscale](/fa/gateway/tailscale) مراجعه کنید.

  </Accordion>

  <Accordion title="چگونه یک Node در Mac را به یک Gateway راه دور متصل کنم (Tailscale Serve)؟">
    Serve، **رابط کاربری کنترل Gateway و WS** را در دسترس قرار می‌دهد؛ Nodeها از طریق همان نقطه پایانی WS مربوط به Gateway متصل می‌شوند.

    1. مطمئن شوید VPS و Mac در یک tailnet قرار دارند.
    2. از برنامه macOS در حالت Remote استفاده کنید (هدف SSH می‌تواند نام میزبان tailnet باشد)؛ این برنامه پورت Gateway را تونل می‌کند و به‌عنوان یک Node متصل می‌شود.
    3. Node را تأیید کنید:
       ```bash
       openclaw devices list
       openclaw devices approve <requestId>
       ```

    مستندات: [پروتکل Gateway](/fa/gateway/protocol)، [کشف](/fa/gateway/discovery)، [حالت راه دور macOS](/fa/platforms/mac/remote).

  </Accordion>

  <Accordion title="آیا باید روی لپ‌تاپ دوم نصب کنم یا فقط یک Node اضافه کنم؟">
    برای استفاده از **فقط ابزارهای محلی** (صفحه‌نمایش/دوربین/exec) روی لپ‌تاپ دوم، آن را به‌عنوان یک **Node** اضافه کنید؛ یک Gateway خواهید داشت و پیکربندی تکراری ایجاد نمی‌شود. ابزارهای محلی Node در حال حاضر فقط در macOS در دسترس‌اند. Gateway دوم را فقط برای **جداسازی کامل** یا دو ربات کاملاً مجزا نصب کنید.

    مستندات: [Nodeها](/fa/nodes)، [CLI مربوط به Nodeها](/fa/cli/nodes)، [چند Gateway](/fa/gateway/multiple-gateways).

  </Accordion>
</AccordionGroup>

## متغیرهای محیطی و بارگذاری .env

<AccordionGroup>
  <Accordion title="OpenClaw چگونه متغیرهای محیطی را بارگذاری می‌کند؟">
    OpenClaw متغیرهای محیطی را از فرایند والد (shell،‏ launchd/systemd،‏ CI و غیره) می‌خواند و علاوه بر آن، موارد زیر را بارگذاری می‌کند:

    - `.env` از پوشه کاری فعلی.
    - یک مقدار بازگشتی سراسری `.env` از `~/.openclaw/.env` (`$OPENCLAW_STATE_DIR/.env`).

    هیچ‌یک از فایل‌های `.env` متغیرهای محیطی موجود را بازنویسی نمی‌کنند. کلیدهای اعتبارنامه ارائه‌دهنده و مسیریابی نقطه پایانی برای `.env` فضای کاری استثنا هستند: کلیدهایی مانند `GEMINI_API_KEY`،‏ `XAI_API_KEY`،‏ `MISTRAL_API_KEY` یا هر کلیدی که به `_ENDPOINT` ختم شود (و سایر متغیرهای محیطی احراز هویت یا نقطه پایانی ارائه‌دهندگان همراه) از `.env` فضای کاری نادیده گرفته می‌شوند و باید در محیط فرایند، `~/.openclaw/.env` یا پیکربندی `env` قرار گیرند.

    متغیرهای محیطی درون‌خطی در پیکربندی فقط در صورت نبودن در محیط فرایند اعمال می‌شوند:

    ```json5
    {
      env: {
        OPENROUTER_API_KEY: "sk-or-...",
        vars: { GROQ_API_KEY: "gsk-..." },
      },
    }
    ```

    برای تقدم کامل و منابع، به [/environment](/fa/help/environment) مراجعه کنید.

  </Accordion>

  <Accordion title="Gateway را از طریق سرویس راه‌اندازی کردم و متغیرهای محیطی من ناپدید شدند. حالا چه کنم؟">
    دو راه‌حل:

    1. کلیدهای مفقود را در `~/.openclaw/.env` قرار دهید تا حتی وقتی سرویس محیط shell شما را به ارث نمی‌برد، بارگذاری شوند.
    2. درون‌ریزی shell را فعال کنید (قابلیت اختیاری برای سهولت):
       ```json5
       {
         env: {
           shellEnv: {
             enabled: true,
             timeoutMs: 15000,
           },
         },
       }
       ```
       این کار shell ورود شما را اجرا می‌کند و فقط کلیدهای مورد انتظارِ مفقود را درون‌ریزی می‌کند (هرگز بازنویسی نمی‌کند). معادل‌های متغیر محیطی: `OPENCLAW_LOAD_SHELL_ENV=1`،‏ `OPENCLAW_SHELL_ENV_TIMEOUT_MS=15000`.

  </Accordion>

  <Accordion title='COPILOT_GITHUB_TOKEN را تنظیم کردم، اما وضعیت مدل‌ها "Shell env: off." را نشان می‌دهد. چرا؟'>
    `openclaw models status` گزارش می‌کند که آیا **درون‌ریزی محیط shell** فعال است یا خیر. "Shell env: off" به این معنا **نیست** که متغیرهای محیطی شما مفقودند؛ فقط یعنی OpenClaw،‏ shell ورود شما را به‌طور خودکار بارگذاری نمی‌کند.

    اگر Gateway به‌عنوان سرویس (launchd/systemd) اجرا شود، محیط shell شما را به ارث نمی‌برد. برای رفع مشکل، توکن را در `~/.openclaw/.env` قرار دهید، `env.shellEnv.enabled: true` را فعال کنید، یا آن را به پیکربندی `env` اضافه کنید (فقط در صورت مفقود بودن اعمال می‌شود)؛ سپس Gateway را مجدداً راه‌اندازی و دوباره بررسی کنید:

    ```bash
    openclaw models status
    ```

    توکن‌های Copilot با این ترتیب تعیین می‌شوند: ابتدا `OPENCLAW_GITHUB_TOKEN`، سپس `COPILOT_GITHUB_TOKEN`، بعد `GH_TOKEN` و در نهایت `GITHUB_TOKEN`.

    به [/concepts/model-providers](/fa/concepts/model-providers) و [/environment](/fa/help/environment) مراجعه کنید.

  </Accordion>
</AccordionGroup>

## نشست‌ها و چند گفت‌وگو

<AccordionGroup>
  <Accordion title="چگونه یک گفت‌وگوی تازه آغاز کنم؟">
    `/new` یا `/reset` را به‌عنوان یک پیام مستقل ارسال کنید. به [مدیریت نشست](/fa/concepts/session) مراجعه کنید.
  </Accordion>

  <Accordion title="اگر هرگز /new را ارسال نکنم، آیا نشست‌ها به‌طور خودکار بازنشانی می‌شوند؟">
    خیر، به‌طور پیش‌فرض این‌طور نیست. نشست‌ها همان `sessionId` را حفظ می‌کنند و با رشد گفت‌وگوها، Compaction زمینه فعال مدل را محدود می‌کند. `/new` و `/reset` همچنان در دسترس‌اند، یا می‌توانید با `mode: "daily"` یا `mode: "idle"` بازنشانی خودکار را فعال کنید. حالت روزانه در ساعت `session.reset.atHour` (پیش‌فرض `4`،‏ 0-23) روی میزبان Gateway تغییر روز می‌دهد؛ حالت بی‌کاری از `session.reset.idleMinutes` پس از آخرین تعامل واقعی استفاده می‌کند، نه رویدادهای سیستمی Heartbeat/Cron/exec.

    ```json5
    {
      session: {
        reset: { mode: "daily", atHour: 4 },
        resetByType: {
          group: { mode: "idle", idleMinutes: 120 },
          thread: { mode: "daily", atHour: 6 },
        },
        resetByChannel: {
          discord: { mode: "idle", idleMinutes: 10080 },
        },
      },
    }
    ```

    `resetByType` از `direct`،‏ `group` و `thread` پشتیبانی می‌کند. Doctor ورودی‌های قدیمی `dm` را به `direct` منتقل می‌کند؛ طرح‌واره `dm` را رد می‌کند. `session.idleMinutes` قدیمی در سطح بالا، هنگامی که هیچ بلوک `session.reset`/`resetByType` تنظیم نشده باشد، همچنان به‌عنوان نام مستعار سازگاری برای پیش‌فرض حالت بی‌کاری کار می‌کند. برای چرخه عمر کامل، به [مدیریت نشست](/fa/concepts/session) مراجعه کنید.

  </Accordion>

  <Accordion title="آیا راهی برای ساخت یک تیم از نمونه‌های OpenClaw وجود دارد (یک مدیرعامل و چندین عامل)؟">
    بله، از طریق **مسیریابی چندعاملی** و **زیرعامل‌ها**: یک عامل هماهنگ‌کننده به‌همراه چندین عامل اجرایی که فضاهای کاری و مدل‌های خود را دارند.

    بهتر است این را یک آزمایش سرگرم‌کننده در نظر بگیرید؛ توکن زیادی مصرف می‌کند و اغلب از یک ربات با نشست‌های جداگانه کم‌بازده‌تر است. الگوی معمول، یک ربات است که با آن گفت‌وگو می‌کنید و برای کارهای موازی نشست‌های متفاوت دارد و در صورت نیاز زیرعامل ایجاد می‌کند.

    مستندات: [مسیریابی چندعاملی](/fa/concepts/multi-agent)، [زیرعامل‌ها](/fa/tools/subagents)، [CLI عامل‌ها](/fa/cli/agents).

  </Accordion>

  <Accordion title="چرا زمینه در میانه کار کوتاه شد؟ چگونه از آن جلوگیری کنم؟">
    زمینه نشست به پنجره مدل محدود است. گفت‌وگوهای طولانی، خروجی‌های بزرگ ابزارها یا فایل‌های زیاد می‌توانند باعث Compaction یا کوتاه‌سازی شوند.

    - از ربات بخواهید وضعیت فعلی را خلاصه کند و آن را در یک فایل بنویسد.
    - پیش از کارهای طولانی از `/compact` و هنگام تغییر موضوع از `/new` استفاده کنید.
    - زمینه مهم را در فضای کاری نگه دارید و از ربات بخواهید آن را دوباره بخواند.
    - برای کارهای طولانی یا موازی از زیرعامل‌ها استفاده کنید تا گفت‌وگوی اصلی کوچک‌تر بماند.
    - اگر این اتفاق زیاد رخ می‌دهد، مدلی با پنجره زمینه بزرگ‌تر انتخاب کنید.

  </Accordion>

  <Accordion title="چگونه OpenClaw را کاملاً بازنشانی کنم، اما نصب‌شده نگه دارم؟">
    ```bash
    openclaw reset
    ```

    بازنشانی کامل غیرتعاملی:

    ```bash
    openclaw reset --scope full --yes --non-interactive
    ```

    سپس راه‌اندازی را دوباره اجرا کنید:

    ```bash
    openclaw onboard --install-daemon
    ```

    اگر فرایند راه‌اندازی اولیه یک پیکربندی موجود را تشخیص دهد، گزینه **بازنشانی** را نیز ارائه می‌دهد؛ به [راه‌اندازی اولیه (CLI)](/fa/start/wizard) مراجعه کنید. اگر از پروفایل‌ها (`--profile` / `OPENCLAW_PROFILE`) استفاده کرده‌اید، هر پوشه وضعیت را بازنشانی کنید (پیش‌فرض `~/.openclaw-<profile>`). بازنشانی ویژه توسعه: `openclaw gateway --dev --reset` پیکربندی توسعه، اعتبارنامه‌ها، نشست‌ها و فضای کاری را پاک می‌کند.

  </Accordion>

  <Accordion title='خطاهای "context too large" دریافت می‌کنم؛ چگونه بازنشانی یا فشرده‌سازی کنم؟'>
    - **Compaction** (گفت‌وگو را حفظ و نوبت‌های قدیمی‌تر را خلاصه می‌کند): `/compact` یا `/compact <instructions>` برای هدایت خلاصه.
    - **بازنشانی** (شناسه نشست تازه برای همان کلید گفت‌وگو): `/new` یا `/reset`.

    اگر این مشکل ادامه یافت، **هرس نشست** (`agents.defaults.contextPruning`) را برای حذف خروجی‌های قدیمی ابزار تنظیم کنید، یا از مدلی با پنجره زمینه بزرگ‌تر استفاده کنید.

    مستندات: [Compaction](/fa/concepts/compaction)، [هرس نشست](/fa/concepts/session-pruning)، [مدیریت نشست](/fa/concepts/session).

  </Accordion>

  <Accordion title='چرا پیام "LLM request rejected: messages.content.tool_use.input field required" را می‌بینم؟'>
    خطای اعتبارسنجی ارائه‌دهنده: مدل یک بلوک `tool_use` بدون `input` الزامی تولید کرده است. معمولاً یعنی تاریخچه نشست قدیمی یا خراب است (اغلب پس از رشته‌گفت‌وگوهای طولانی یا تغییر ابزار/طرح‌واره).

    راه‌حل: با `/new` یک نشست تازه آغاز کنید (پیام مستقل).

  </Accordion>

  <Accordion title="چرا هر 30 دقیقه پیام‌های Heartbeat دریافت می‌کنم؟">
    Heartbeatها به‌طور پیش‌فرض هر **30m** اجرا می‌شوند، یا زمانی که حالت احراز هویت تعیین‌شده، احراز هویت OAuth/توکن Anthropic باشد (از جمله استفاده مجدد از Claude CLI) و `heartbeat.every` تنظیم نشده باشد، هر **1h** اجرا می‌شوند. برای تنظیم یا غیرفعال‌سازی:

    ```json5
    {
      agents: {
        defaults: {
          heartbeat: {
            every: "2h", // یا "0m" برای غیرفعال‌سازی
          },
        },
      },
    }
    ```

    اگر `HEARTBEAT.md` وجود داشته باشد اما عملاً خالی باشد (فقط خطوط خالی، توضیحات Markdown/HTML، عنوان‌های ATX، نشانگرهای fence یا جای‌نگهدارهای خالی آیتم فهرست)، OpenClaw برای صرفه‌جویی در فراخوانی‌های API اجرای Heartbeat را نادیده می‌گیرد. اگر فایل وجود نداشته باشد، Heartbeat همچنان اجرا می‌شود و مدل تصمیم می‌گیرد چه کاری انجام دهد.

    بازنویسی‌های مختص هر عامل از `agents.entries.*.heartbeat` استفاده می‌کنند. مستندات: [Heartbeat](/fa/gateway/heartbeat).

  </Accordion>

  <Accordion title='آیا باید یک "حساب ربات" به گروه WhatsApp اضافه کنم؟'>
    خیر. OpenClaw روی **حساب خودتان** اجرا می‌شود؛ اگر عضو گروه باشید، OpenClaw می‌تواند آن را ببیند. به‌طور پیش‌فرض، پاسخ‌ها در گروه تا زمانی که فرستندگان را مجاز کنید (`groupPolicy: "allowlist"`) مسدود هستند.

    برای محدود کردن پاسخ‌های گروه فقط به خودتان:

    ```json5
    {
      channels: {
        whatsapp: {
          groupPolicy: "allowlist",
          groupAllowFrom: ["+15551234567"],
        },
      },
    }
    ```

  </Accordion>

  <Accordion title="چگونه JID یک گروه WhatsApp را به‌دست آورم؟">
    سریع‌ترین راه: گزارش‌ها را به‌صورت زنده دنبال کنید و یک پیام آزمایشی در گروه بفرستید.

    ```bash
    openclaw logs --follow --json
    ```

    به‌دنبال `chatId` (یا `from`) بگردید که به `@g.us` ختم می‌شود، مانند `1234567890-1234567890@g.us`.

    اگر از قبل پیکربندی یا در فهرست مجاز قرار داده شده است، گروه‌ها را از پیکربندی فهرست کنید:

    ```bash
    openclaw directory groups list --channel whatsapp
    ```

    مستندات: [WhatsApp](/fa/channels/whatsapp)، [دایرکتوری](/fa/cli/directory)، [گزارش‌ها](/fa/cli/logs).

  </Accordion>

  <Accordion title="چرا OpenClaw در یک گروه پاسخ نمی‌دهد؟">
    دو علت رایج: محدودسازی بر اساس اشاره به‌طور پیش‌فرض فعال است (باید ربات را با @ خطاب کنید یا با `mentionPatterns` تطبیق داشته باشید)، یا `channels.whatsapp.groups` را بدون `"*"` پیکربندی کرده‌اید و گروه در فهرست مجاز نیست.

    به [گروه‌ها](/fa/channels/groups) و [پیام‌های گروهی](/fa/channels/group-messages) مراجعه کنید.

  </Accordion>

  <Accordion title="آیا گروه‌ها/رشته‌گفت‌وگوها زمینه را با پیام‌های مستقیم به اشتراک می‌گذارند؟">
    گفت‌وگوهای مستقیم به‌طور پیش‌فرض در نشست اصلی ادغام می‌شوند. گروه‌ها/کانال‌ها کلیدهای نشست خود را دارند و موضوعات Telegram / رشته‌گفت‌وگوهای Discord نشست‌های جداگانه‌ای هستند. به [گروه‌ها](/fa/channels/groups) و [پیام‌های گروهی](/fa/channels/group-messages) مراجعه کنید.
  </Accordion>

  <Accordion title="چند فضای کاری و عامل می‌توانم ایجاد کنم؟">
    هیچ محدودیت قطعی وجود ندارد؛ ده‌ها یا حتی صدها مورد مشکلی ندارند، اما مراقب موارد زیر باشید:

    - **رشد فضای دیسک**: نشست‌های فعال و رونوشت‌ها در پایگاه‌داده SQLite مختص هر عامل نگه‌داری می‌شوند؛ مصنوعات قدیمی/بایگانی همچنان ممکن است در `~/.openclaw/agents/<agentId>/sessions/` انباشته شوند.
    - **هزینه توکن**: عامل‌های بیشتر به‌معنای استفاده هم‌زمان بیشتر از مدل است.
    - **سربار عملیاتی**: پروفایل‌های احراز هویت، فضاهای کاری و مسیریابی کانال مختص هر عامل.

    برای هر عامل یک فضای کاری **فعال** (`agents.defaults.workspace`) نگه دارید، اگر مصرف دیسک افزایش یافت نشست‌های قدیمی را با `openclaw sessions cleanup` پاک‌سازی کنید (وضعیت فعال SQLite را دستی ویرایش نکنید) و برای یافتن فضاهای کاری سرگردان و ناهماهنگی‌های پروفایل از `openclaw doctor` استفاده کنید.

  </Accordion>

  <Accordion title="آیا می‌توانم چند ربات یا گفت‌وگو را هم‌زمان اجرا کنم (Slack) و چگونه باید آن را راه‌اندازی کنم؟">
    بله، از طریق **مسیریابی چندعاملی**: چند عامل مجزا را اجرا کنید و پیام‌های ورودی را بر اساس کانال/حساب/همتا مسیریابی کنید. Slack به‌عنوان کانال پشتیبانی می‌شود و می‌توان آن را به عامل‌های مشخصی متصل کرد.

    دسترسی مرورگر قدرتمند است، اما نمی‌تواند «هر کاری را که انسان می‌تواند انجام دهد» انجام دهد؛ سازوکارهای ضدربات، CAPTCHA و MFA همچنان می‌توانند جلوی خودکارسازی را بگیرند. برای مطمئن‌ترین کنترل، از Chrome MCP محلی روی میزبان یا CDP روی دستگاهی استفاده کنید که مرورگر واقعاً روی آن اجرا می‌شود.

    راه‌اندازی پیشنهادی: میزبان Gateway همیشه‌روشن (VPS/Mac mini)، یک عامل برای هر نقش (اتصال‌ها)، کانال‌های Slack متصل به آن عامل‌ها و در صورت نیاز مرورگر محلی از طریق Chrome MCP یا یک Node.

    مستندات: [مسیریابی چندعاملی](/fa/concepts/multi-agent)، [Slack](/fa/channels/slack)، [مرورگر](/fa/tools/browser)، [Nodeها](/fa/nodes).

  </Accordion>
</AccordionGroup>

## مدل‌ها، انتقال هنگام خرابی و پروفایل‌های احراز هویت

پرسش‌وپاسخ مدل‌ها — پیش‌فرض‌ها، انتخاب، نام‌های مستعار، جابه‌جایی، انتقال هنگام خرابی و پروفایل‌های احراز هویت — در [پرسش‌های متداول مدل‌ها](/fa/help/faq-models) قرار دارد.

## Gateway: درگاه‌ها، «از قبل در حال اجرا» و حالت راه دور

<AccordionGroup>
  <Accordion title="Gateway از چه درگاهی استفاده می‌کند؟">
    `gateway.port` درگاه چندگانه واحد برای WebSocket + HTTP (رابط کاربری کنترل، هوک‌ها و غیره) را کنترل می‌کند. ترتیب اولویت:

    ```text
    --port > OPENCLAW_GATEWAY_PORT > gateway.port > پیش‌فرض 18789
    ```

  </Accordion>

  <Accordion title='چرا openclaw gateway status عبارت "Runtime: running" را نشان می‌دهد، اما "Connectivity probe: failed" است؟'>
    «در حال اجرا» دیدگاه **ناظر** (launchd/systemd/schtasks) است؛ آزمون اتصال، اتصال واقعی CLI به WebSocket در Gateway است. به این سطرها در `openclaw gateway status` اعتماد کنید: `Probe target:` (نشانی اینترنتی استفاده‌شده توسط آزمون)، `Listening:` (چیزی که واقعاً به درگاه متصل است)، `Last gateway error:` (علت اصلی رایج وقتی فرایند زنده است اما درگاه شنونده نیست).
  </Accordion>

  <Accordion title='چرا openclaw gateway status مقادیر متفاوتی برای "Config (cli)" و "Config (service)" نشان می‌دهد؟'>
    شما یک فایل پیکربندی را ویرایش می‌کنید، درحالی‌که سرویس فایل دیگری را اجرا می‌کند (اغلب ناهماهنگی `--profile` / `OPENCLAW_STATE_DIR`).

    برای رفع مشکل، فرمان زیر را از همان `--profile` / محیطی اجرا کنید که می‌خواهید سرویس استفاده کند:

    ```bash
    openclaw gateway install --force
    ```

  </Accordion>

  <Accordion title='عبارت "another gateway instance is already listening" به چه معناست؟'>
    OpenClaw با اتصال فوری شنونده WebSocket هنگام راه‌اندازی (پیش‌فرض `ws://127.0.0.1:18789`) قفل زمان اجرا را اعمال می‌کند. اگر اتصال با `EADDRINUSE` ناموفق شود، خطای `GatewayLockError` («نمونه دیگری از Gateway از قبل در حال شنود است») صادر می‌شود.

    راه‌حل: نمونه دیگر را متوقف کنید، درگاه را آزاد کنید یا با `openclaw gateway --port <port>` اجرا کنید.

  </Accordion>

  <Accordion title="چگونه OpenClaw را در حالت راه دور اجرا کنم (کلاینت به Gateway در مکانی دیگر متصل شود)؟">
    `gateway.mode: "remote"` را تنظیم کنید و نشانی WebSocket راه دور را مشخص کنید؛ اعتبارنامه‌های راه دور مبتنی بر راز مشترک اختیاری هستند:

    ```json5
    {
      gateway: {
        mode: "remote",
        remote: {
          url: "ws://gateway.tailnet:18789",
          token: "your-token",
          password: "your-password",
        },
      },
    }
    ```

    - `openclaw gateway` فقط زمانی شروع می‌شود که `gateway.mode` برابر با `local` باشد (یا یک پرچم بازنویسی ارسال کنید).
    - برنامه macOS فایل پیکربندی را زیر نظر می‌گیرد و هنگام تغییر این مقادیر، حالت‌ها را به‌صورت زنده جابه‌جا می‌کند.
    - `gateway.remote.token` / `.password` فقط اعتبارنامه‌های راه دور سمت کلاینت هستند؛ این مقادیر به‌تنهایی احراز هویت Gateway محلی را فعال نمی‌کنند.

  </Accordion>

  <Accordion title='رابط کاربری کنترل پیام "unauthorized" را نشان می‌دهد (یا پیوسته دوباره متصل می‌شود). اکنون چه کنم؟'>
    مسیر احراز هویت Gateway و روش احراز هویت رابط کاربری با یکدیگر مطابقت ندارند.

    واقعیت‌ها (برگرفته از کد):

    - رابط کاربری کنترل، توکن را در `sessionStorage` نگه می‌دارد و دامنه آن را به زبانه فعلی مرورگر و نشانی Gateway انتخاب‌شده محدود می‌کند؛ بنابراین تازه‌سازی همان زبانه بدون ماندگاری بلندمدت توکن در localStorage به کار خود ادامه می‌دهد.
    - در `AUTH_TOKEN_MISMATCH`، وقتی Gateway راهنمای تلاش مجدد (`canRetryWithDeviceToken=true`، `recommendedNextStep=retry_with_device_token`) برمی‌گرداند، کلاینت‌های مورد اعتماد می‌توانند یک تلاش مجدد محدود را با توکن دستگاه ذخیره‌شده در حافظه نهان انجام دهند.
    - این تلاش مجدد با توکن ذخیره‌شده، دامنه‌های تأییدشده ذخیره‌شده همراه توکن دستگاه را دوباره استفاده می‌کند؛ فراخوان‌های صریح `deviceToken` / صریح `scopes` به‌جای به‌ارث‌بردن دامنه‌های ذخیره‌شده، مجموعه دامنه درخواستی خود را حفظ می‌کنند.
    - خارج از آن مسیر تلاش مجدد، ترتیب اولویت احراز هویت اتصال چنین است: ابتدا توکن/گذرواژه مشترک صریح، سپس `deviceToken` صریح، بعد توکن دستگاه ذخیره‌شده و در پایان توکن راه‌اندازی اولیه.
    - راه‌اندازی اولیه داخلی با کد راه‌اندازی، یک توکن دستگاه Node با `scopes: []` به‌همراه یک توکن محدود تحویل به اپراتور برای آماده‌سازی موبایل مورد اعتماد برمی‌گرداند. تحویل به اپراتور می‌تواند پیکربندی بومی زمان راه‌اندازی را بخواند، اما دامنه‌های تغییر جفت‌سازی یا `operator.admin` را اعطا نمی‌کند.

    راه‌حل:

    - سریع‌ترین روش: `openclaw dashboard` (نشانی داشبورد را چاپ و کپی می‌کند و می‌کوشد آن را باز کند؛ در محیط بدون نمایشگر راهنمای SSH نشان می‌دهد).
    - هنوز توکن ندارید: `openclaw doctor --generate-gateway-token`.
    - راه دور: ابتدا با `ssh -N -L 18789:127.0.0.1:18789 user@host` تونل ایجاد کنید، سپس `http://127.0.0.1:18789/` را باز کنید.
    - حالت راز مشترک: `gateway.auth.token` / `OPENCLAW_GATEWAY_TOKEN` یا `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` را تنظیم کنید، سپس راز متناظر را در تنظیمات رابط کاربری کنترل جای‌گذاری کنید.
    - حالت Tailscale Serve: تأیید کنید `gateway.auth.allowTailscale` فعال است و نشانی Serve را باز می‌کنید، نه یک نشانی خام loopback/tailnet که سرآیندهای هویت Tailscale را دور می‌زند.
    - حالت پراکسی مورد اعتماد: تأیید کنید از پراکسی آگاه از هویت پیکربندی‌شده عبور می‌کنید. پراکسی‌های loopback روی همان میزبان نیز به `gateway.auth.trustedProxy.allowLoopback = true` نیاز دارند.
    - اگر ناهماهنگی پس از یک تلاش مجدد ادامه داشت، توکن دستگاه جفت‌شده را چرخش دهید/دوباره تأیید کنید:
      ```bash
      openclaw devices list
      openclaw devices rotate --device <id> --role operator
      ```
    - چرخش رد شد: نشست‌های دستگاه جفت‌شده فقط می‌توانند دستگاه **خودشان** را چرخش دهند، مگر اینکه `operator.admin` را نیز داشته باشند؛ همچنین مقادیر صریح `--scope` نمی‌توانند از دامنه‌های اپراتوری فعلی فراخواننده فراتر روند.
    - اگر همچنان مشکل باقی است: `openclaw status --all` به‌همراه [عیب‌یابی](/fa/gateway/troubleshooting). برای جزئیات احراز هویت، [داشبورد](/fa/web/dashboard) را ببینید.

  </Accordion>

  <Accordion title="gateway.bind را روی tailnet تنظیم کردم، اما فقط روی loopback شنود می‌کند">
    اتصال `tailnet` یک IP متعلق به Tailscale را از رابط‌های شبکه شما انتخاب می‌کند (100.64.0.0/10). اگر دستگاه عضو Tailscale نباشد (یا رابط از کار افتاده باشد)، Gateway به‌جای در معرض قرار دادن یک رابط شبکه دیگر، به loopback بازمی‌گردد.

    راه‌حل: Tailscale را روی آن میزبان راه‌اندازی و Gateway را بازراه‌اندازی کنید، یا صراحتاً به `gateway.bind: "loopback"` / `"lan"` تغییر دهید.

    `tailnet` صریح است؛ `auto`، loopback را ترجیح می‌دهد. برای محدود کردن دسترسی غیر-loopback به Tailnet، درحالی‌که شنونده الزامی `127.0.0.1` روی همان میزبان حفظ می‌شود، از `gateway.bind: "tailnet"` استفاده کنید.

  </Accordion>

  <Accordion title="آیا می‌توانم چند Gateway را روی یک میزبان اجرا کنم؟">
    معمولاً خیر؛ یک Gateway می‌تواند چند کانال پیام‌رسانی و عامل را اجرا کند. فقط برای افزونگی (برای مثال، یک ربات نجات) یا جداسازی سخت از چند Gateway استفاده کنید و هرکدام را با `OPENCLAW_CONFIG_PATH`، `OPENCLAW_STATE_DIR`، `agents.defaults.workspace` و `gateway.port` منحصربه‌فرد خودش مجزا کنید.

    توصیه‌شده: برای هر نمونه `openclaw --profile <name> ...` (به‌طور خودکار `~/.openclaw-<name>` را می‌سازد)، برای پیکربندی هر پروفایل یک `gateway.port` منحصربه‌فرد (یا `--port` برای اجراهای دستی) و یک سرویس مختص هر پروفایل با `openclaw --profile <name> gateway install`.

    پروفایل‌ها همچنین پسوندی به نام سرویس‌ها اضافه می‌کنند: launchd با `ai.openclaw.<profile>`، systemd با `openclaw-gateway-<profile>.service` و Windows با `OpenClaw Gateway (<profile>)`. واحد systemd بدون پسوند `openclaw-gateway` فقط برای پروفایل پیش‌فرض وجود دارد؛ نام قدیمی واحد systemd پیش از تغییر نام، یعنی `clawdbot-gateway`، به‌طور خودکار مهاجرت داده می‌شود.

    راهنمای کامل: [چند Gateway](/fa/gateway/multiple-gateways).

  </Accordion>

  <Accordion title='عبارت "invalid handshake" / کد 1008 به چه معناست؟'>
    Gateway یک **سرور WebSocket** است و انتظار دارد نخستین پیام یک فریم `connect` باشد. هر چیز دیگری اتصال را با **کد 1008** (نقض سیاست) می‌بندد.

    علت‌های رایج: نشانی **HTTP** را به‌جای کلاینت WS در مرورگر باز کرده‌اید، درگاه/مسیر اشتباه را به‌کار برده‌اید، یا یک پراکسی/تونل سرآیندهای احراز هویت را حذف کرده یا درخواستی غیر از Gateway فرستاده است.

    راه‌حل: از نشانی WS (`ws://<host>:18789`، یا `wss://...` روی HTTPS) استفاده کنید، درگاه WS را در زبانه معمولی مرورگر باز نکنید و وقتی احراز هویت فعال است، توکن/گذرواژه را در فریم `connect` قرار دهید. نمونه CLI/TUI:

    ```bash
    openclaw tui --url ws://<host>:18789 --token <token>
    ```

    جزئیات پروتکل: [پروتکل Gateway](/fa/gateway/protocol).

  </Accordion>
</AccordionGroup>

## ثبت رویداد و اشکال‌زدایی

<AccordionGroup>
  <Accordion title="گزارش‌ها کجا هستند؟">
    گزارش‌های فایل (ساخت‌یافته): `/tmp/openclaw/openclaw-YYYY-MM-DD.log` برای پروفایل پیش‌فرض یا `/tmp/openclaw/openclaw-<profile>-YYYY-MM-DD.log` برای یک پروفایل نام‌گذاری‌شده. یک مسیر پایدار را از طریق `logging.file` تنظیم کنید؛ سطح گزارش فایل را با `logging.level` و میزان جزئیات کنسول را با `--verbose` و `logging.consoleLevel` تنظیم کنید.

    سریع‌ترین روش دنبال‌کردن گزارش:

    ```bash
    openclaw logs --follow
    ```

    گزارش‌های سرویس/ناظر (هنگامی که Gateway از طریق launchd/systemd اجرا می‌شود):

    - خروجی استاندارد launchd در macOS: `~/Library/Logs/openclaw/gateway.log` (پروفایل‌ها از `gateway-<profile>.log` استفاده می‌کنند؛ خطای استاندارد سرکوب می‌شود).
    - Linux: `journalctl --user -u openclaw-gateway[-<profile>].service -n 200 --no-pager`.
    - Windows: `schtasks /Query /TN "OpenClaw Gateway (<profile>)" /V /FO LIST`.

    برای اطلاعات بیشتر [عیب‌یابی](/fa/gateway/troubleshooting) را ببینید.

  </Accordion>

  <Accordion title="چگونه سرویس Gateway را شروع/متوقف/بازراه‌اندازی کنم؟">
    ```bash
    openclaw gateway status
    openclaw gateway restart
    ```

    اگر Gateway را دستی اجرا می‌کنید، `openclaw gateway --force` می‌تواند درگاه را پس بگیرد. [Gateway](/fa/gateway) را ببینید.

  </Accordion>

  <Accordion title="ترمینال خود را در Windows بستم؛ چگونه OpenClaw را دوباره راه‌اندازی کنم؟">
    سه حالت نصب در Windows:

    **1) راه‌اندازی محلی Windows Hub**: برنامه بومی، یک WSL Gateway محلی و متعلق به برنامه را مدیریت می‌کند. **OpenClaw Companion** را از منوی Start یا ناحیه اعلان باز کنید، سپس از **Gateway Setup** یا زبانه Connections استفاده کنید.

    **2) WSL2 Gateway دستی**: Gateway درون Linux اجرا می‌شود.
    ```powershell
    wsl
    openclaw gateway status
    openclaw gateway restart
    ```
    اگر هرگز سرویس را نصب نکرده‌اید، آن را در پیش‌زمینه راه‌اندازی کنید: `openclaw gateway run`.

    **3) CLI/Gateway بومی Windows**: مستقیماً در Windows اجرا می‌شود.
    ```powershell
    openclaw gateway status
    openclaw gateway restart
    ```
    اگر آن را دستی اجرا می‌کنید (بدون سرویس): `openclaw gateway run`.

    مستندات: [Windows](/fa/platforms/windows)، [راهنمای عملیاتی سرویس Gateway](/fa/gateway).

  </Accordion>

  <Accordion title="Gateway فعال است، اما پاسخ‌ها هرگز نمی‌رسند. چه مواردی را بررسی کنم؟">
    بررسی سریع سلامت:

    ```bash
    openclaw status
    openclaw models status
    openclaw channels status
    openclaw logs --follow
    ```

    علت‌های رایج: احراز هویت مدل روی **میزبان Gateway** بارگیری نشده است (`models status` را بررسی کنید)، جفت‌سازی/فهرست مجاز کانال جلوی پاسخ‌ها را می‌گیرد (پیکربندی کانال و گزارش‌ها را بررسی کنید)، یا WebChat/داشبورد بدون توکن صحیح باز شده است. اگر اتصال راه دور است، تأیید کنید اتصال تونل/Tailscale برقرار است و WebSocket مربوط به Gateway دسترس‌پذیر است.

    مستندات: [کانال‌ها](/fa/channels)، [عیب‌یابی](/fa/gateway/troubleshooting)، [دسترسی از راه دور](/fa/gateway/remote).

  </Accordion>

  <Accordion title='"ارتباط با gateway قطع شد: بدون دلیل" — حالا چه باید کرد؟'>
    معمولاً یعنی رابط کاربری اتصال WebSocket را از دست داده است. بررسی کنید: آیا Gateway در حال اجراست (`openclaw gateway status`)؟ آیا سالم است (`openclaw status`)؟ آیا رابط کاربری توکن درست را دارد (`openclaw dashboard`)؟ اگر از راه دور است، آیا پیوند تونل/Tailscale برقرار است؟

    سپس لاگ‌ها را به‌صورت زنده دنبال کنید:

    ```bash
    openclaw logs --follow
    ```

    مستندات: [داشبورد](/fa/web/dashboard)، [دسترسی از راه دور](/fa/gateway/remote)، [عیب‌یابی](/fa/gateway/troubleshooting).

  </Accordion>

  <Accordion title="Telegram setMyCommands ناموفق است. چه چیزهایی را باید بررسی کرد؟">
    ```bash
    openclaw channels status
    openclaw channels logs --channel telegram
    ```

    سپس خطا را تطبیق دهید:

    - `BOT_COMMANDS_TOO_MUCH`: منوی Telegram ورودی‌های بیش‌ازحدی دارد. OpenClaw از قبل تعداد را تا محدودیت Telegram کاهش می‌دهد و با فرمان‌های کمتری دوباره تلاش می‌کند، اما ممکن است برخی ورودی‌های منو همچنان حذف شوند. تعداد فرمان‌های Plugin/skill/سفارشی را کاهش دهید، یا اگر به منو نیاز ندارید `channels.telegram.commands.native` را غیرفعال کنید.
    - `TypeError: fetch failed`، `Network request for 'setMyCommands' failed!` یا خطاهای شبکه‌ای مشابه: در یک VPS یا پشت پراکسی، تأیید کنید HTTPS خروجی مجاز است و DNS برای `api.telegram.org` کار می‌کند.

    اگر Gateway از راه دور است، لاگ‌ها را در میزبان Gateway بررسی کنید.

    مستندات: [Telegram](/fa/channels/telegram)، [عیب‌یابی کانال](/fa/channels/troubleshooting).

  </Accordion>

  <Accordion title="TUI هیچ خروجی‌ای نشان نمی‌دهد. چه چیزهایی را باید بررسی کرد؟">
    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    در TUI، برای دیدن وضعیت فعلی از `/status` استفاده کنید. اگر انتظار پاسخ در یک کانال چت را دارید، تأیید کنید تحویل فعال است (`/deliver on`).

    مستندات: [TUI](/fa/web/tui)، [فرمان‌های اسلش](/fa/tools/slash-commands).

  </Accordion>

  <Accordion title="چگونه Gateway را کاملاً متوقف و سپس راه‌اندازی کنم؟">
    اگر سرویس را نصب کرده‌اید (launchd در macOS،‏ systemd در Linux):

    ```bash
    openclaw gateway stop
    openclaw gateway start
    ```

    در پیش‌زمینه، با Ctrl-C متوقف کنید، سپس `openclaw gateway run`.

    مستندات: [راهنمای عملیاتی سرویس Gateway](/fa/gateway).

  </Accordion>

  <Accordion title="توضیح ساده: تفاوت openclaw gateway restart با openclaw gateway">
    `openclaw gateway restart` **سرویس پس‌زمینه** (launchd/systemd) را راه‌اندازی مجدد می‌کند. `openclaw gateway`، gateway را برای این نشست ترمینال **در پیش‌زمینه** اجرا می‌کند. اگر سرویس را نصب کرده‌اید از زیرفرمان‌های gateway استفاده کنید؛ برای یک اجرای موردی، از اجرای مستقیم در پیش‌زمینه استفاده کنید.
  </Accordion>

  <Accordion title="سریع‌ترین راه برای دریافت جزئیات بیشتر هنگام بروز خطا">
    برای جزئیات بیشتر در کنسول، Gateway را با `--verbose` راه‌اندازی کنید، سپس فایل لاگ را برای خطاهای احراز هویت کانال، مسیریابی مدل و RPC بررسی کنید.
  </Accordion>
</AccordionGroup>

## رسانه و پیوست‌ها

<AccordionGroup>
  <Accordion title="skill من یک تصویر/PDF تولید کرد، اما چیزی ارسال نشد">
    پیوست‌های خروجی عامل باید از فیلدهای ساخت‌یافته رسانه مانند `media`، `mediaUrl`، `path` یا `filePath` استفاده کنند. به [راه‌اندازی دستیار OpenClaw](/fa/start/openclaw) و [ارسال عامل](/fa/tools/agent-send) مراجعه کنید.

    ```bash
    openclaw message send --target +15555550123 --message "بفرمایید" --media /path/to/file.png
    ```

    همچنین بررسی کنید: کانال مقصد از رسانه خروجی پشتیبانی می‌کند و توسط فهرست‌های مجاز مسدود نشده است؛ فایل در محدوده اندازه ارائه‌دهنده قرار دارد (اندازه تصاویر به حداکثر ضلع 2048px تغییر می‌کند)؛ `tools.fs.workspaceOnly=true` ارسال از مسیر محلی را به فایل‌های فضای کاری، موقت/مخزن رسانه و فایل‌های تأییدشده در sandbox محدود می‌کند؛ `tools.fs.workspaceOnly=false` (پیش‌فرض) به ارسال‌های ساخت‌یافته رسانه محلی اجازه می‌دهد از فایل‌های محلی میزبان که عامل از قبل قادر به خواندن آن‌هاست استفاده کنند؛ این شامل رسانه و انواع امن سند است (تصاویر، صوت، ویدئو، PDF، اسناد Office و اسناد متنی تأییدشده مانند Markdown/MD،‏ TXT،‏ JSON،‏ YAML/YML). این یک اسکنر اطلاعات محرمانه نیست — یک `secret.txt` یا `config.json` قابل‌خواندن برای عامل، در صورت تطابق پسوند و اعتبارسنجی محتوا، می‌تواند پیوست شود. فایل‌های حساس را خارج از مسیرهای قابل‌خواندن برای عامل نگه دارید، یا برای سخت‌گیری بیشتر در ارسال از مسیر محلی، `tools.fs.workspaceOnly=true` را حفظ کنید.

    به [تصاویر](/fa/nodes/images) مراجعه کنید.

  </Accordion>
</AccordionGroup>

## امنیت و کنترل دسترسی

<AccordionGroup>
  <Accordion title="آیا قرار دادن OpenClaw در معرض پیام‌های خصوصی ورودی امن است؟">
    پیام‌های خصوصی ورودی را ورودی نامطمئن تلقی کنید. تنظیمات پیش‌فرض خطر را کاهش می‌دهند:

    - رفتار پیش‌فرض در کانال‌های دارای قابلیت پیام خصوصی، **جفت‌سازی** است: فرستندگان ناشناس یک کد جفت‌سازی دریافت می‌کنند و پیامشان پردازش نمی‌شود. با `openclaw pairing approve --channel <channel> [--account <id>] <code>` تأیید کنید. درخواست‌های در انتظار به **3 مورد در هر کانال** محدودند؛ اگر کدی دریافت نشد، `openclaw pairing list --channel <channel> [--account <id>]` را بررسی کنید.
    - باز کردن عمومی پیام‌های خصوصی مستلزم انتخاب صریح (`dmPolicy: "open"` و فهرست مجاز `"*"`) است.

    برای آشکار کردن خط‌مشی‌های پرخطر پیام خصوصی، `openclaw doctor` را اجرا کنید.

  </Accordion>

  <Accordion title="آیا تزریق پرامپت فقط برای بات‌های عمومی نگران‌کننده است؟">
    خیر. تزریق پرامپت به **محتوای نامطمئن** مربوط است، نه صرفاً به اینکه چه کسی می‌تواند به بات پیام خصوصی بدهد. اگر دستیار محتوای خارجی را می‌خواند (جست‌وجو/واکشی وب، صفحات مرورگر، ایمیل‌ها، اسناد، پیوست‌ها، لاگ‌های جای‌گذاری‌شده)، آن محتوا می‌تواند حاوی دستورالعمل‌هایی برای ربودن کنترل مدل باشد — حتی اگر تنها فرستنده خودتان باشید.

    بیشترین خطر زمانی است که ابزارها فعال باشند: ممکن است مدل فریب بخورد و زمینه را استخراج کند یا از طرف شما ابزارها را فراخوانی کند. دامنه آسیب را کاهش دهید:

    - برای خلاصه‌سازی محتوای نامطمئن، از یک عامل «خواننده» فقط‌خواندنی یا بدون ابزار استفاده کنید
    - برای عامل‌های دارای ابزار، `web_search` / `web_fetch` / `browser` را خاموش نگه دارید
    - متن رمزگشایی‌شده فایل/سند را نیز نامطمئن تلقی کنید: هم OpenResponses `input_file` و هم استخراج پیوست رسانه‌ای، به‌جای عبور مستقیم متن خام فایل، متن استخراج‌شده را در نشانگرهای صریح مرزی محتوای خارجی قرار می‌دهند
    - از sandbox و فهرست‌های مجاز سخت‌گیرانه ابزار استفاده کنید

    جزئیات: [امنیت](/fa/gateway/security).

  </Accordion>

  <Accordion title="آیا OpenClaw به‌دلیل استفاده از TypeScript/Node به‌جای Rust/WASM امنیت کمتری دارد؟">
    زبان و محیط اجرا اهمیت دارند، اما خطر اصلی برای یک عامل شخصی نیستند. خطرهای عملی عبارت‌اند از در معرض بودن gateway، اینکه چه کسی می‌تواند به بات پیام بدهد، تزریق پرامپت، دامنه ابزار، مدیریت اطلاعات احراز هویت، دسترسی مرورگر، دسترسی exec و اعتماد به skill/Plugin شخص ثالث.

    Rust و WASM می‌توانند برای برخی دسته‌های کد جداسازی قوی‌تری فراهم کنند، اما تزریق پرامپت، فهرست‌های مجاز نامناسب، در معرض بودن عمومی gateway، ابزارهای بیش‌ازحد گسترده یا نمایه مرورگری را که از قبل وارد حساب‌های حساس شده است حل نمی‌کنند. این موارد را کنترل‌های اصلی در نظر بگیرید: Gateway را خصوصی یا دارای احراز هویت نگه دارید، برای پیام‌های خصوصی/گروه‌ها از جفت‌سازی و فهرست‌های مجاز استفاده کنید، ابزارهای پرخطر را برای ورودی‌های نامطمئن منع کنید یا در sandbox اجرا کنید، فقط Pluginها و skillهای مورداعتماد را نصب کنید و پس از تغییرات پیکربندی `openclaw security audit --deep` را اجرا کنید.

    جزئیات: [امنیت](/fa/gateway/security)، [اجرای ایزوله](/fa/gateway/sandboxing).

  </Accordion>

  <Accordion title="گزارش‌هایی درباره نمونه‌های در معرض دسترس OpenClaw دیدم. چه چیزهایی را باید بررسی کرد؟">
    ```bash
    openclaw security audit --deep
    openclaw gateway status
    ```

    یک خط مبنای امن‌تر: Gateway به `loopback` متصل باشد، یا فقط از طریق دسترسی خصوصی احراز هویت‌شده در معرض قرار گیرد (tailnet، تونل SSH، احراز هویت توکن/گذرواژه یا یک پراکسی مورداعتماد با پیکربندی درست)؛ پیام‌های خصوصی در حالت `pairing` یا `allowlist` باشند؛ گروه‌ها در فهرست مجاز باشند و نیازمند اشاره باشند، مگر اینکه همه اعضا مورداعتماد باشند؛ ابزارهای پرخطر (`exec`، `browser`، `gateway`، `cron`) برای عامل‌هایی که محتوای نامطمئن می‌خوانند منع یا به‌شدت محدود شوند؛ در جاهایی که اجرای ابزار به دامنه آسیب کوچک‌تری نیاز دارد، sandbox فعال باشد.

    اتصال عمومی بدون احراز هویت، پیام‌های خصوصی/گروه‌های باز همراه با ابزارها و کنترل مرورگرِ در معرض دسترس، یافته‌هایی هستند که باید ابتدا برطرف شوند. جزئیات: [openclaw security audit](/fa/gateway/security#openclaw-security-audit).

  </Accordion>

  <Accordion title="آیا نصب skillهای ClawHub و Pluginهای شخص ثالث امن است؟">
    skillها و Pluginهای شخص ثالث را کدی در نظر بگیرید که انتخاب می‌کنید به آن اعتماد کنید. صفحات skill در ClawHub وضعیت اسکن را پیش از نصب نمایش می‌دهند، اما اسکن‌ها یک مرز امنیتی کامل نیستند. OpenClaw هنگام نصب یا به‌روزرسانی Plugin/skill، مسدودسازی محلی داخلی برای کد خطرناک اجرا نمی‌کند؛ برای تصمیم‌های محلی مجاز/مسدودسازی، از `security.installPolicy` تحت کنترل اپراتور استفاده کنید.

    الگوی امن‌تر: نویسندگان مورداعتماد و نسخه‌های ثابت‌شده را ترجیح دهید، پیش از فعال‌سازی skill/Plugin آن را بخوانید، فهرست‌های مجاز Plugin/skill را محدود نگه دارید، گردش‌کارهای دارای ورودی نامطمئن را با حداقل ابزارها در sandbox اجرا کنید و از اعطای دسترسی گسترده به فایل‌سیستم، exec، مرورگر یا اطلاعات محرمانه به کد شخص ثالث خودداری کنید.

    جزئیات: [Skills](/fa/tools/skills)، [Pluginها](/fa/tools/plugin)، [امنیت](/fa/gateway/security).

  </Accordion>

  <Accordion title="آیا بات من باید ایمیل، حساب GitHub یا شماره تلفن خودش را داشته باشد؟">
    بله، برای بیشتر راه‌اندازی‌ها. جداسازی بات با حساب‌ها و شماره‌های تلفن جداگانه، در صورت بروز مشکل دامنه آسیب را کاهش می‌دهد و چرخش اطلاعات احراز هویت یا لغو دسترسی را بدون تأثیر بر حساب‌های شخصی آسان‌تر می‌کند.

    از کم شروع کنید: فقط به ابزارها و حساب‌هایی که واقعاً نیاز دارید دسترسی بدهید و در صورت نیاز بعداً آن را گسترش دهید.

    مستندات: [امنیت](/fa/gateway/security)، [جفت‌سازی](/fa/channels/pairing).

  </Accordion>

  <Accordion title="آیا می‌توانم کنترل مستقلی بر پیام‌های متنی‌ام به آن بدهم و آیا این کار امن است؟">
    ما استقلال کامل بر پیام‌های شخصی را توصیه **نمی‌کنیم**. امن‌ترین الگو: پیام‌های خصوصی را در **حالت جفت‌سازی** یا یک فهرست مجاز محدود نگه دارید، اگر قرار است از طرف شما پیام دهد از یک **شماره یا حساب جداگانه** استفاده کنید و اجازه دهید پیش‌نویس تهیه کند، درحالی‌که شما **پیش از ارسال تأیید می‌کنید**.

    برای آزمایش، این کار را روی یک حساب اختصاصی و ایزوله انجام دهید. به [امنیت](/fa/gateway/security) مراجعه کنید.

  </Accordion>

  <Accordion title="آیا می‌توانم برای وظایف دستیار شخصی از مدل‌های ارزان‌تر استفاده کنم؟">
    بله، **اگر** عامل فقط برای چت باشد و ورودی مورداعتماد باشد. رده‌های کوچک‌تر بیشتر در معرض ربایش دستورالعمل هستند، بنابراین برای عامل‌های دارای ابزار یا هنگام خواندن محتوای نامطمئن از آن‌ها استفاده نکنید. اگر مجبورید از مدل کوچک‌تری استفاده کنید، ابزارها را محدود و آن را داخل sandbox اجرا کنید. به [امنیت](/fa/gateway/security) مراجعه کنید.
  </Accordion>

  <Accordion title="در Telegram دستور /start را اجرا کردم، اما کد جفت‌سازی دریافت نکردم">
    کدهای جفت‌سازی **فقط** زمانی ارسال می‌شوند که یک فرستنده ناشناس به بات پیام دهد و `dmPolicy: "pairing"` فعال باشد؛ `/start` به‌تنهایی کدی تولید نمی‌کند.

    درخواست‌های در انتظار را بررسی کنید:

    ```bash
    openclaw pairing list telegram
    ```

    برای دسترسی فوری، شناسه فرستنده خود را به فهرست مجاز اضافه کنید یا برای آن حساب `dmPolicy: "open"` را تنظیم کنید.

  </Accordion>

  <Accordion title="WhatsApp: آیا به مخاطبان من پیام می‌دهد؟ جفت‌سازی چگونه کار می‌کند؟">
    خیر. خط‌مشی پیش‌فرض پیام خصوصی WhatsApp، **جفت‌سازی** است. فرستندگان ناشناس فقط یک کد جفت‌سازی دریافت می‌کنند؛ پیام آن‌ها **پردازش نمی‌شود**. OpenClaw فقط به چت‌هایی که دریافت می‌کند یا ارسال‌های صریحی که شما آغاز می‌کنید پاسخ می‌دهد.

    ```bash
    openclaw pairing approve whatsapp <code>
    openclaw pairing list whatsapp
    ```

    درخواست شماره تلفن در راهنما، **فهرست مجاز/مالک** شما را تنظیم می‌کند تا پیام‌های خصوصی خودتان مجاز باشند — از آن برای ارسال خودکار استفاده نمی‌شود. روی شماره شخصی WhatsApp خود، از همان شماره استفاده کنید و `channels.whatsapp.selfChatMode` را فعال کنید.

  </Accordion>
</AccordionGroup>

## فرمان‌های چت، لغو وظایف و «متوقف نمی‌شود»

<AccordionGroup>
  <Accordion title="چگونه نمایش پیام‌های داخلی سیستم در چت را متوقف کنم؟">
    بیشتر پیام‌های داخلی/ابزار فقط زمانی ظاهر می‌شوند که **verbose**،‏ **trace** یا **reasoning** برای آن نشست فعال باشد.

    در همان چتی که آن‌ها را می‌بینید اصلاح کنید:

    ```text
    /verbose off
    /trace off
    /reasoning off
    ```

    اگر همچنان شلوغ است: تنظیمات نشست را در رابط کاربری کنترل بررسی و verbose را روی **inherit** تنظیم کنید؛ تأیید کنید از نمایه باتی که در پیکربندی `verboseDefault: "on"` دارد استفاده نمی‌کنید.

    مستندات: [تفکر و خروجی مفصل](/fa/tools/thinking)، [امنیت](/fa/gateway/security/index#reasoning-and-verbose-output-in-groups).

  </Accordion>

  <Accordion title="چگونه یک وظیفه در حال اجرا را متوقف/لغو کنم؟">
    برای لغو اجرا، هریک از این موارد را **به‌صورت یک پیام مستقل** (بدون اسلش) ارسال کنید: `stop`، `stop action`، `stop current action`، `stop run`، `stop current run`، `stop agent`، `stop the agent`، `stop openclaw`، `openclaw stop`، `stop don't do anything`، `stop do not do anything`، `stop doing anything`، `do not do that`، `please stop`، `stop please`، `abort`، `esc`، `exit`، `interrupt`، `halt`. محرک‌های رایج غیراِنگلیسی (فرانسوی، آلمانی، اسپانیایی، چینی، ژاپنی، هندی، عربی و روسی) نیز کار می‌کنند.

    برای فرایندهای پس‌زمینه‌ای که ابزار exec راه‌اندازی کرده است، از عامل بخواهید این دستور را اجرا کند:

    ```text
    process action:kill sessionId:XXX
    ```

    بیشتر فرمان‌های اسلش باید به‌صورت پیامی **مستقل** که با `/` آغاز می‌شود ارسال شوند، اما چند میان‌بر (مانند `/status`) برای فرستندگان موجود در فهرست مجاز به‌صورت درون‌خطی نیز کار می‌کنند. به [فرمان‌های اسلش](/fa/tools/slash-commands) مراجعه کنید.

  </Accordion>

  <Accordion title='چگونه از Telegram به Discord پیام بفرستم؟ («پیام‌رسانی میان‌زمینه‌ای رد شد»)'>
    OpenClaw به‌طور پیش‌فرض پیام‌رسانی **میان ارائه‌دهندگان** را مسدود می‌کند. اگر فراخوانی ابزاری به Telegram مقید باشد، به Discord پیام نمی‌فرستد، مگر اینکه صریحاً آن را مجاز کنید؛ این تغییر بی‌درنگ اعمال می‌شود و نیازی به راه‌اندازی مجدد Gateway نیست:

    ```json5
    {
      tools: {
        message: {
          crossContext: {
            allowAcrossProviders: true,
            marker: { enabled: true, prefix: "[from {channel}] " },
          },
        },
      },
    }
    ```

  </Accordion>

  <Accordion title='چرا به نظر می‌رسد ربات پیام‌های پی‌درپی و سریع را «نادیده می‌گیرد»؟'>
    به‌طور پیش‌فرض، درخواست‌های حین اجرا به اجرای فعال هدایت می‌شوند. برای انتخاب رفتار اجرای فعال از `/queue` استفاده کنید:

    - `steer` (پیش‌فرض) - اجرای فعال را در مرز بعدی مدل هدایت می‌کند.
    - `followup` - پیام‌ها را در صف قرار می‌دهد و پس از پایان اجرای فعلی، آن‌ها را یکی‌یکی اجرا می‌کند.
    - `collect` - پیام‌های سازگار را در صف قرار می‌دهد و پس از پایان اجرای فعلی، یک‌بار پاسخ می‌دهد.
    - `interrupt` - اجرای فعلی را لغو می‌کند و اجرای تازه‌ای را آغاز می‌کند.

    گزینه‌هایی مانند `debounce:0.5s cap:25 drop:summarize` را به حالت‌های صف‌شده اضافه کنید. به [صف فرمان‌ها](/fa/concepts/queue) و [صف هدایت](/fa/concepts/queue-steering) مراجعه کنید.

  </Accordion>
</AccordionGroup>

## متفرقه

<AccordionGroup>
  <Accordion title='مدل پیش‌فرض Anthropic هنگام استفاده از کلید API چیست؟'>
    اعتبارنامه‌ها و انتخاب مدل از یکدیگر جدا هستند. تنظیم `ANTHROPIC_API_KEY` (یا ذخیره‌کردن کلید API متعلق به Anthropic در نمایه‌های احراز هویت) احراز هویت را فعال می‌کند، اما مدل پیش‌فرض واقعی همان مدلی است که در `agents.defaults.model.primary` پیکربندی می‌کنید (برای مثال `anthropic/claude-sonnet-4-6` یا `anthropic/claude-opus-4-6`). `No credentials found for profile "anthropic:default"` یعنی Gateway نتوانسته است اعتبارنامه‌های Anthropic را در `auth-profiles.json` مورد انتظار برای عامل در حال اجرا پیدا کند.
  </Accordion>
</AccordionGroup>

---

هنوز مشکل حل نشده است؟ در [Discord](https://discord.com/invite/clawd) بپرسید یا یک [گفت‌وگوی GitHub](https://github.com/openclaw/openclaw/discussions) باز کنید.

## مرتبط

- [پرسش‌های متداول نخستین اجرا](/fa/help/faq-first-run) - نصب، راه‌اندازی اولیه، احراز هویت، اشتراک‌ها، خطاهای اولیه
- [پرسش‌های متداول مدل‌ها](/fa/help/faq-models) - انتخاب مدل، جایگزینی هنگام خرابی، نمایه‌های احراز هویت
- [عیب‌یابی](/fa/help/troubleshooting) - بررسی و تشخیص بر پایه نشانه‌ها
