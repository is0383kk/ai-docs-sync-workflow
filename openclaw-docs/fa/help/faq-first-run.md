---
read_when:
    - نصب جدید، گیرکردن در راه‌اندازی اولیه یا خطاهای اجرای نخست
    - انتخاب اشتراک‌های احراز هویت و ارائه‌دهنده
    - نمی‌توان به docs.openclaw.ai دسترسی یافت، داشبورد باز نمی‌شود، نصب متوقف شده است
sidebarTitle: First-run FAQ
summary: 'پرسش‌های متداول: راه‌اندازی سریع و تنظیمات اجرای نخست — نصب، آغاز به کار، احراز هویت، اشتراک‌ها، خطاهای اولیه'
title: 'پرسش‌های متداول: راه‌اندازی اجرای نخست'
x-i18n:
    generated_at: "2026-07-27T15:34:54Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e1c93b89da625ae5f092db854c9b74adc005be75dd913af4bf89ed1a4f35396a
    source_path: help/faq-first-run.md
    workflow: 16
---

پرسش‌وپاسخ شروع سریع و نخستین اجرا. برای عملیات روزمره، مدل‌ها، احراز هویت، نشست‌ها
و عیب‌یابی، [پرسش‌های متداول](/fa/help/faq) اصلی را ببینید.

## شروع سریع و راه‌اندازی نخستین اجرا

<AccordionGroup>
  <Accordion title="گیر کرده‌ام؛ سریع‌ترین راه برای رفع مشکل چیست؟">
    از یک عامل هوش مصنوعی محلی استفاده کنید که بتواند **دستگاه شما را ببیند**. بیشتر موارد «گیر کرده‌ام»
    ناشی از **مشکلات پیکربندی یا محیط محلی** هستند که یک کمک‌کننده راه‌دور نمی‌تواند بررسی کند؛ بنابراین این روش از
    پرسیدن در Discord بهتر است.

    - **Claude Code**: [https://www.anthropic.com/claude-code/](https://www.anthropic.com/claude-code/)
    - **OpenAI Codex**: [https://openai.com/codex/](https://openai.com/codex/)

    از طریق نصب قابل‌تغییر (git)، نسخه کامل کد منبع را در اختیار عامل قرار دهید تا بتواند
    کد و مستندات را بخواند و دقیقاً درباره نسخه‌ای که اجرا می‌کنید استدلال کند:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    از عامل بخواهید رفع مشکل را گام‌به‌گام برنامه‌ریزی و نظارت کند و سپس فقط
    فرمان‌های ضروری را اجرا کند؛ بررسی تفاوت‌های کوچک‌تر آسان‌تر است.

    هنگام درخواست کمک (در Discord یا یک مسئله GitHub)، این خروجی‌ها را به‌اشتراک بگذارید:

    | فرمان | نمایش می‌دهد |
    | --- | --- |
    | `openclaw status` | سلامت Gateway/عامل و نمای کلی پیکربندی پایه |
    | `openclaw status --all` | تشخیص کامل فقط‌خواندنی و قابل‌جای‌گذاری |
    | `openclaw models status` | احراز هویت ارائه‌دهنده و دسترس‌پذیری مدل |
    | `openclaw doctor` | مشکلات رایج پیکربندی/وضعیت را اعتبارسنجی و اصلاح می‌کند |
    | `openclaw logs --follow` | دنباله زنده گزارش |
    | `openclaw gateway status --deep` | بررسی عمیق سلامت Gateway/پیکربندی/Plugin |
    | `openclaw health --verbose` | گزارش تفصیلی سلامت |

    یک باگ یا اصلاح واقعی پیدا کرده‌اید؟ یک مسئله ثبت کنید یا PR بفرستید:
    [مسئله‌ها](https://github.com/openclaw/openclaw/issues) /
    [Pull requestها](https://github.com/openclaw/openclaw/pulls).

    چرخه سریع اشکال‌زدایی: [۶۰ ثانیه نخست وقتی چیزی خراب است](/fa/help/faq#first-60-seconds-if-something-is-broken).
    مستندات نصب: [نصب](/fa/install)، [پرچم‌های نصب‌کننده](/fa/install/installer)، [به‌روزرسانی](/fa/install/updating).

  </Accordion>

  <Accordion title="Heartbeat مدام رد می‌شود. دلایل رد شدن چه معنایی دارند؟">
    | دلیل رد شدن | معنا |
    | --- | --- |
    | `quiet-hours` | خارج از بازه ساعات فعال پیکربندی‌شده |
    | `empty-heartbeat-file` | پیش‌نویس پایش Heartbeat وجود دارد، اما فقط شامل محتوای خالی، نظر، سرآیند، حصار یا چارچوب چک‌لیست خالی است |
    | `alerts-disabled` | همه قابلیت‌های نمایش Heartbeat خاموش‌اند (`showOk`، `showAlerts` و `useIndicator` همگی غیرفعال‌اند) |

    بلوک‌های قدیمی Heartbeat با `tasks:` به کارهای Cron با زمان‌بندی مستقل و `openclaw doctor --fix` مهاجرت می‌کنند.

    مستندات: [Heartbeat](/fa/gateway/heartbeat)، [خودکارسازی](/fa/automation).

  </Accordion>

  <Accordion title="روش پیشنهادی نصب و راه‌اندازی OpenClaw">
    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash
    openclaw onboard --install-daemon
    ```

    از کد منبع (مشارکت‌کنندگان/توسعه‌دهندگان):

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    pnpm ui:build
    openclaw onboard
    ```

    هنوز نصب سراسری ندارید؟ در عوض `pnpm openclaw onboard` را اجرا کنید. اگر دارایی‌های رابط کنترل
    موجود نباشند، فرایند پذیرش سعی می‌کند آن‌ها را بسازد و در صورت شکست به `pnpm ui:build` بازمی‌گردد.

  </Accordion>

  <Accordion title="پس از پذیرش، چگونه داشبورد را باز کنم؟">
    فرایند پذیرش بلافاصله پس از راه‌اندازی، مرورگر را با یک نشانی تمیز داشبورد (بدون توکن)
    باز می‌کند و پیوند را در خلاصه چاپ می‌کند. آن زبانه را باز نگه دارید؛ اگر اجرا نشد،
    نشانی چاپ‌شده را در همان دستگاه کپی و جای‌گذاری کنید.
  </Accordion>

  <Accordion title="چگونه داشبورد را در localhost و حالت راه‌دور احراز هویت کنم؟">
    **Localhost (همان دستگاه):**

    - `http://127.0.0.1:18789/` را باز کنید.
    - اگر احراز هویت با راز مشترک درخواست شد، توکن یا گذرواژه پیکربندی‌شده را در تنظیمات رابط کنترل جای‌گذاری کنید.
    - منبع توکن: `gateway.auth.token` (یا `OPENCLAW_GATEWAY_TOKEN`).
    - منبع گذرواژه: `gateway.auth.password` (یا `OPENCLAW_GATEWAY_PASSWORD`).
    - هنوز راز مشترکی پیکربندی نشده است؟ `openclaw doctor --generate-gateway-token` (یا `openclaw doctor --fix --generate-gateway-token`) را اجرا کنید.

    **خارج از localhost:**

    - **Tailscale Serve** (پیشنهادی): اتصال را روی loopback نگه دارید، `openclaw gateway --tailscale serve` را اجرا کنید و `https://<magicdns>/` را باز کنید. با `gateway.auth.allowTailscale: true`، سرآیندهای هویت احراز هویت رابط کنترل/WebSocket را برآورده می‌کنند (بدون جای‌گذاری راز مشترک و با فرض مورداعتماد بودن میزبان Gateway)؛ APIهای HTTP همچنان به احراز هویت با راز مشترک نیاز دارند، مگر اینکه عمداً از ورودی خصوصی `none` یا احراز هویت HTTP پراکسی مورداعتماد استفاده کنید.
      تلاش‌های هم‌زمان Serve با احراز هویت نامعتبر از یک کلاینت، پیش از ثبت‌شدن در محدودکننده احراز هویت ناموفق به‌ترتیب اجرا می‌شوند؛ بنابراین تلاش نامعتبر دوم ممکن است از همان ابتدا `retry later` را نشان دهد.
    - **اتصال Tailnet**: `openclaw gateway --bind tailnet --token "<token>"` را اجرا کنید (یا احراز هویت با گذرواژه را پیکربندی کنید)، `http://<tailscale-ip>:18789/` را باز کنید و راز مشترک منطبق را در تنظیمات داشبورد جای‌گذاری کنید.
    - **پراکسی معکوس آگاه از هویت**: Gateway را پشت یک پراکسی مورداعتماد نگه دارید، `gateway.auth.mode: "trusted-proxy"` را تنظیم کنید و نشانی پراکسی را باز کنید. پراکسی‌های loopback روی همان میزبان به `gateway.auth.trustedProxy.allowLoopback: true` صریح نیاز دارند.
    - **تونل SSH**: `ssh -N -L 18789:127.0.0.1:18789 user@gateway-host`، سپس `http://127.0.0.1:18789/` را باز کنید. احراز هویت با راز مشترک همچنان در تونل اعمال می‌شود؛ در صورت درخواست، توکن یا گذرواژه پیکربندی‌شده را جای‌گذاری کنید.

    برای حالت‌های اتصال و جزئیات احراز هویت، [داشبورد](/fa/web/dashboard) و [سطوح وب](/fa/web) را ببینید.

  </Accordion>

  <Accordion title="چرا برای تأییدهای گفت‌وگویی دو پیکربندی تأیید exec وجود دارد؟">
    آن‌ها لایه‌های متفاوتی را کنترل می‌کنند:

    - `approvals.exec` - درخواست‌های تأیید را به مقصدهای گفت‌وگو هدایت می‌کند.
    - `channels.<channel>.execApprovals` - آن کانال را به یک کلاینت بومی تأیید برای تأییدهای exec تبدیل می‌کند.

    سیاست exec میزبان همچنان درگاه واقعی تأیید است؛ پیکربندی گفت‌وگو فقط محل
    نمایش درخواست‌ها و نحوه پاسخ‌دادن افراد را کنترل می‌کند.

    به‌ندرت به هر دو نیاز دارید:

    - اگر گفت‌وگو از قبل از فرمان‌ها و پاسخ‌ها پشتیبانی می‌کند، `/approve` در همان گفت‌وگو از مسیر مشترک کار می‌کند.
    - وقتی یک کانال بومی پشتیبانی‌شده بتواند تأییدکنندگان را با اطمینان تشخیص دهد، اگر `channels.<channel>.execApprovals.enabled` تنظیم نشده یا `"auto"` باشد، OpenClaw تأییدهای بومی با اولویت پیام مستقیم را به‌طور خودکار فعال می‌کند.
    - وقتی کارت‌ها/دکمه‌های بومی تأیید موجود باشند، آن رابط کاربری در اولویت است؛ فقط زمانی به فرمان دستی `/approve` اشاره کنید که نتیجه ابزار اعلام کند تأییدهای گفت‌وگویی در دسترس نیستند.
    - از `approvals.exec` فقط زمانی استفاده کنید که درخواست‌ها باید به گفت‌وگوهای دیگر یا اتاق‌های عملیاتی مشخص نیز برسند.
    - از `channels.<channel>.execApprovals.target: "channel"` یا `"both"` فقط زمانی استفاده کنید که می‌خواهید درخواست‌های تأیید دوباره در اتاق/موضوع مبدأ ارسال شوند.
    - تأییدهای Plugin جدا هستند: به‌طور پیش‌فرض `/approve` در همان گفت‌وگو، هدایت اختیاری `approvals.plugin`، و فقط برخی کانال‌های بومی، رسیدگی بومی را برای آن‌ها نیز حفظ می‌کنند.

    خلاصه: هدایت برای مسیریابی است و پیکربندی کلاینت بومی برای تجربه کاربری غنی‌تر و مختص کانال.
    [تأییدهای Exec](/fa/tools/exec-approvals) را ببینید.

  </Accordion>

  <Accordion title="به چه محیط اجرایی نیاز دارم؟">
    Node **22.22.3+**، **24.15+** یا **25.9+** الزامی است (Node 24 پیشنهاد می‌شود). `pnpm` مدیر بسته مخزن است.
    Bun می‌تواند وابستگی‌ها را نصب و اسکریپت‌های بسته را اجرا کند، اما نمی‌تواند CLI یا Gateway مربوط به OpenClaw را اجرا کند، زیرا `node:sqlite` را ندارد.
  </Accordion>

  <Accordion title="آیا روی Raspberry Pi اجرا می‌شود؟">
    بله، اما ابتدا RAM را بررسی کنید: Pi 5 و Pi 4 (2 GB+) بهترین گزینه‌اند؛ Pi 3B+ (1 GB) کار می‌کند، اما کند است؛ Pi Zero 2 W (512 MB) پیشنهاد نمی‌شود.

    | مدل | RAM | تناسب |
    | --- | --- | --- |
    | Pi 5 | 4/8 GB | بهترین |
    | Pi 4 | 4 GB | خوب |
    | Pi 4 | 2 GB | قابل‌قبول، swap اضافه کنید |
    | Pi 4 | 1 GB | محدود |
    | Pi 3B+ | 1 GB | کند |
    | Pi Zero 2 W | 512 MB | پیشنهاد نمی‌شود |

    حداقل مطلق: 1 GB RAM، یک هسته، 500 MB فضای آزاد دیسک و سیستم‌عامل 64‌بیتی. ازآنجاکه Pi فقط
    Gateway را اجرا می‌کند (مدل‌ها APIهای ابری را فراخوانی می‌کنند)، حتی یک Pi متوسط نیز از پس بار برمی‌آید.

    یک Pi/VPS کوچک همچنین می‌تواند فقط میزبان Gateway باشد، درحالی‌که **Nodeها** را روی
    لپ‌تاپ/تلفن خود برای صفحه‌نمایش/دوربین/بوم محلی یا اجرای فرمان جفت می‌کنید. [Nodeها](/fa/nodes) را ببینید.

    راهنمای کامل راه‌اندازی: [Raspberry Pi](/fa/install/raspberry-pi).

  </Accordion>

  <Accordion title="نکته‌ای برای نصب روی Raspberry Pi دارید؟">
    - از سیستم‌عامل **64‌بیتی** استفاده کنید؛ از Raspberry Pi OS ‏32‌بیتی استفاده نکنید.
    - در بردهای 2 GB یا کوچک‌تر، swap اضافه کنید.
    - برای عملکرد و طول عمر بهتر، **USB SSD** را به کارت SD ترجیح دهید.
    - نصب قابل‌تغییر (git) را ترجیح دهید تا بتوانید گزارش‌ها را ببینید و سریع به‌روزرسانی کنید.
    - بدون کانال‌ها/Skills شروع کنید و آن‌ها را یکی‌یکی اضافه کنید.
    - خرابی‌های عجیب فایل اجرایی («exec format error») معمولاً ناشی از نبود ساخت ARM64 برای یک ابزار اختیاری Skill است.

    راهنمای کامل: [Raspberry Pi](/fa/install/raspberry-pi). همچنین [Linux](/fa/platforms/linux) را ببینید.

  </Accordion>

  <Accordion title="روی wake up my friend گیر کرده است / فرایند پذیرش باز نمی‌شود. چه‌کار کنم؟">
    آن صفحه به دردسترس و احراز هویت‌شده بودن Gateway وابسته است. وقتی یک ارائه‌دهنده مدل پیکربندی شده باشد، TUI نیز
    هنگام نخستین بازشدن به‌طور خودکار «بیدار شو، دوست من!» را می‌فرستد. اگر
    راه‌اندازی مدل/احراز هویت را رد کرده باشید، فرایند پذیرش یادداشت «احراز هویت مدل موجود نیست» را نشان می‌دهد و
    TUI را بدون ارسال چیزی باز می‌کند — با `openclaw configure --section model` یک ارائه‌دهنده اضافه کنید.
    اگر خط بیدارباش را **بدون پاسخ** می‌بینید و تعداد توکن‌ها روی 0 می‌ماند، عامل هرگز اجرا نشده است.

    1. Gateway را راه‌اندازی مجدد کنید:

    ```bash
    openclaw gateway restart
    ```

    2. وضعیت و احراز هویت را بررسی کنید:

    ```bash
    openclaw status
    openclaw models status
    openclaw logs --follow
    ```

    3. هنوز متوقف است؟ اجرا کنید:

    ```bash
    openclaw doctor
    ```

    اگر Gateway راه‌دور است، برقراری اتصال تونل/Tailscale و اشاره رابط کاربری به
    Gateway درست را تأیید کنید. [دسترسی راه‌دور](/fa/gateway/remote) را ببینید.

  </Accordion>

  <Accordion title="آیا می‌توانم بدون تکرار فرایند پذیرش، راه‌اندازی خود را به دستگاه جدیدی منتقل کنم؟">
    بله. **دایرکتوری وضعیت** و **فضای کاری** را کپی کنید، سپس Doctor را یک‌بار اجرا کنید:

    1. OpenClaw را روی دستگاه جدید نصب کنید.
    2. `$OPENCLAW_STATE_DIR` (پیش‌فرض: `~/.openclaw`) را از دستگاه قدیمی کپی کنید.
    3. فضای کاری خود (پیش‌فرض: `~/.openclaw/workspace`) را کپی کنید.
    4. `openclaw doctor` را اجرا و سرویس Gateway را راه‌اندازی مجدد کنید.

    این کار پیکربندی، پروفایل‌های احراز هویت، اطلاعات ورود WhatsApp، نشست‌ها و حافظه را حفظ می‌کند؛ به‌شرط کپی‌کردن
    **هر دو** مکان، ربات شما دقیقاً یکسان باقی می‌ماند. در حالت راه‌دور،
    میزبان Gateway مالک ذخیره‌گاه نشست و فضای کاری است.

    **مهم:** اگر فقط فضای کاری خود را در GitHub ثبت/ارسال کنید،
    از **حافظه و فایل‌های راه‌اندازی اولیه** پشتیبان می‌گیرید، اما نه از تاریخچه نشست یا احراز هویت. آن‌ها در
    `~/.openclaw/` قرار دارند (برای مثال `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`).

    مرتبط: [مهاجرت](/fa/install/migrating)، [محل قرارگیری موارد روی دیسک](/fa/help/faq#where-things-live-on-disk)،
    [فضای کاری عامل](/fa/concepts/agent-workspace)، [Doctor](/fa/gateway/doctor)،
    [حالت راه‌دور](/fa/gateway/remote).

  </Accordion>

  <Accordion title="تغییرات جدید آخرین نسخه را کجا ببینم؟">
    تغییرنامه GitHub را بررسی کنید:
    [https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md)

    جدیدترین ورودی‌ها در بالا قرار دارند. اگر بخش بالایی **منتشرنشده** باشد، بخش تاریخ‌دار
    بعدی آخرین نسخه منتشرشده است. ورودی‌ها زیر **نکات برجسته**، **تغییرات**
    و **اصلاحات** گروه‌بندی می‌شوند (و در صورت نیاز بخش‌های مستندات/سایر موارد نیز وجود دارند).

  </Accordion>

  <Accordion title="دسترسی به docs.openclaw.ai ممکن نیست (خطای SSL)">
    برخی اتصال‌های Comcast/Xfinity به‌اشتباه `docs.openclaw.ai` را از طریق Xfinity
    Advanced Security مسدود می‌کنند. آن را غیرفعال کنید یا `docs.openclaw.ai` را در فهرست مجاز قرار دهید، سپس دوباره تلاش کنید. برای
    رفع مسدودی به ما کمک کنید: [https://spa.xfinity.com/check_url_status](https://spa.xfinity.com/check_url_status).

    هنوز مسدود هستید؟ مستندات در GitHub آینه شده‌اند:
    [https://github.com/openclaw/openclaw/tree/main/docs](https://github.com/openclaw/openclaw/tree/main/docs)

  </Accordion>

  <Accordion title="تفاوت نسخه پایدار و بتا">
    **پایدار** و **بتا**، **dist-tagهای npm** هستند، نه خطوط کد جداگانه:

    - `latest` = پایدار
    - `beta` = بیلد اولیه برای آزمایش (وقتی نسخه بتا وجود نداشته باشد یا از انتشار پایدار فعلی قدیمی‌تر باشد، به `latest` برمی‌گردد)

    یک انتشار پایدار معمولاً ابتدا روی **بتا** قرار می‌گیرد، سپس یک مرحله ارتقای صریح
    همان نسخه را بدون تغییر شماره نسخه به `latest` منتقل می‌کند. نگه‌دارندگان
    همچنین می‌توانند مستقیماً روی `latest` منتشر کنند. به همین دلیل، پس از ارتقا
    بتا و پایدار می‌توانند به **نسخه یکسانی** اشاره کنند.

    تغییرات را ببینید: [CHANGELOG.md](https://github.com/openclaw/openclaw/blob/main/CHANGELOG.md).

    برای دستورهای یک‌خطی نصب و تفاوت میان بتا و dev، بخش بازشونده بعدی را ببینید.

  </Accordion>

  <Accordion title="چگونه نسخه بتا را نصب کنم و تفاوت بتا و dev چیست؟">
    **بتا** همان dist-tag npm با نام `beta` است (ممکن است پس از ارتقا با `latest` یکسان باشد).
    **Dev** نوک متغیر `main` در git است؛ هنگام انتشار در npm از dist-tag با نام `dev` استفاده می‌کند.

    دستورهای یک‌خطی (macOS/Linux):

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta
    ```

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    نصب‌کننده Windows (PowerShell): `iwr -useb https://openclaw.ai/install.ps1 | iex`

    جزئیات بیشتر: [کانال‌های توسعه](/fa/install/development-channels) و [پرچم‌های نصب‌کننده](/fa/install/installer).

  </Accordion>

  <Accordion title="چگونه جدیدترین تغییرات را امتحان کنم؟">
    دو گزینه وجود دارد:

    1. **کانال Dev (نصب موجود):**

    ```bash
    openclaw update --channel dev
    ```

    این کار به یک checkout گیت از `main` جابه‌جا می‌شود، آن را روی upstream بازپایه‌گذاری می‌کند، می‌سازد و
    CLI را از همان checkout نصب می‌کند.

    2. **نصب قابل‌تغییر (git) روی دستگاه تازه:**

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    کلون دستی ترجیح داده می‌شود:

    ```bash
    git clone https://github.com/openclaw/openclaw.git
    cd openclaw
    pnpm install
    pnpm build
    ```

    مستندات: [به‌روزرسانی](/fa/cli/update)، [کانال‌های توسعه](/fa/install/development-channels)، [نصب](/fa/install).

  </Accordion>

  <Accordion title="نصب و راه‌اندازی اولیه معمولاً چقدر طول می‌کشد؟">
    راهنمای تقریبی:

    - **نصب:** 2-5 دقیقه.
    - **راه‌اندازی اولیه QuickStart:** چند دقیقه (Gateway حلقه بازگشتی، توکن خودکار، فضای کاری پیش‌فرض).
    - **راه‌اندازی اولیه پیشرفته/کامل:** وقتی ورود به ارائه‌دهنده، جفت‌سازی کانال، نصب daemon، دانلودهای شبکه یا Skills به تنظیمات بیشتری نیاز داشته باشند، طولانی‌تر است.

    ویزارد این زمان‌بندی را از ابتدا نمایش می‌دهد. مراحل اختیاری را رد کنید و بعداً با
    `openclaw configure` بازگردید.

    متوقف شده است؟ بخش [گیر کرده‌ام](#quick-start-and-first-run-setup) را در بالا ببینید.

  </Accordion>

  <Accordion title="نصب‌کننده گیر کرده است؟ چگونه بازخورد بیشتری دریافت کنم؟">
    دوباره با `--verbose` اجرا کنید:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --beta --verbose
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git --verbose
    ```

    `install.ps1` کلید verbose اختصاصی ندارد؛ در عوض آن را در `Set-PSDebug -Trace 1` /
    `-Trace 0` بپیچید. مرجع کامل پرچم‌ها: [پرچم‌های نصب‌کننده](/fa/install/installer).

  </Accordion>

  <Accordion title="نصب Windows می‌گوید git پیدا نشد یا openclaw شناخته نمی‌شود">
    دو مشکل رایج در Windows:

    **1) خطای npm با پیام spawn git / پیدا نشدن git**

    - **Git for Windows** را نصب کنید و مطمئن شوید `git` در PATH قرار دارد.
    - PowerShell را ببندید و دوباره باز کنید، سپس نصب‌کننده را مجدداً اجرا کنید.

    **2) پس از نصب، openclaw شناخته نمی‌شود**

    - پوشه bin سراسری npm شما در PATH قرار ندارد.
    - آن را بررسی کنید: `npm config get prefix`.
    - آن پوشه را به PATH کاربر خود اضافه کنید (پسوند `\bin` لازم نیست؛ در بیشتر سیستم‌ها مسیر آن `%AppData%\npm` است).
    - PowerShell را ببندید و دوباره باز کنید.

    برنامه دسکتاپ را ترجیح می‌دهید؟ از **Windows Hub** استفاده کنید. برای راه‌اندازی صرفاً با ترمینال،
    هم نصب‌کننده PowerShell و هم مسیرهای Gateway در WSL2 پشتیبانی می‌شوند. مستندات: [Windows](/fa/platforms/windows).

  </Accordion>

  <Accordion title="خروجی exec در Windows متن چینی را به‌هم‌ریخته نشان می‌دهد؛ چه‌کار کنم؟">
    معمولاً علت آن ناهماهنگی صفحه کد کنسول در پوسته‌های بومی Windows است.

    نشانه‌ها: خروجی `system.run`/`exec` متن چینی را به‌صورت نویسه‌های مخدوش نمایش می‌دهد؛ همان فرمان
    در نمایه ترمینال دیگری درست به‌نظر می‌رسد.

    راه‌حل موقت در PowerShell:

    ```powershell
    chcp 65001
    [Console]::InputEncoding = [System.Text.UTF8Encoding]::new($false)
    [Console]::OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    $OutputEncoding = [System.Text.UTF8Encoding]::new($false)
    ```

    سپس Gateway را راه‌اندازی مجدد کنید و دوباره امتحان کنید:

    ```powershell
    openclaw gateway restart
    ```

    آیا همچنان در جدیدترین OpenClaw تکرار می‌شود؟ آن را پیگیری/گزارش کنید: [Issue شماره 30640](https://github.com/openclaw/openclaw/issues/30640).

  </Accordion>

  <Accordion title="مستندات به پرسشم پاسخ ندادند؛ چگونه پاسخ بهتری بگیرم؟">
    از نصب قابل‌تغییر (git) استفاده کنید تا کل کد منبع و مستندات را به‌صورت محلی داشته باشید، سپس
    **از داخل همان پوشه** از ربات خود (یا Claude/Codex) بپرسید تا بتواند مخزن را بخواند و دقیق پاسخ دهد.

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    ```

    جزئیات بیشتر: [نصب](/fa/install) و [پرچم‌های نصب‌کننده](/fa/install/installer).

  </Accordion>

  <Accordion title="چگونه OpenClaw را روی Linux نصب کنم؟">
    - مسیر سریع Linux و نصب سرویس: [Linux](/fa/platforms/linux).
    - راهنمای کامل گام‌به‌گام: [شروع به کار](/fa/start/getting-started).
    - نصب‌کننده و به‌روزرسانی‌ها: [نصب و به‌روزرسانی‌ها](/fa/install/updating).

  </Accordion>

  <Accordion title="چگونه OpenClaw را روی یک VPS نصب کنم؟">
    هر VPS مبتنی بر Linux مناسب است. روی سرور نصب کنید، سپس از طریق SSH/Tailscale به Gateway دسترسی پیدا کنید.

    راهنماها: [exe.dev](/fa/install/exe-dev)، [Hetzner](/fa/install/hetzner)، [Fly.io](/fa/install/fly).
    دسترسی از راه دور: [Gateway از راه دور](/fa/gateway/remote).

  </Accordion>

  <Accordion title="راهنماهای نصب ابری/VPS کجا هستند؟">
    مرکز میزبانی برای ارائه‌دهندگان رایج:

    - [میزبانی VPS](/fa/vps) (همه ارائه‌دهندگان در یک مکان)
    - [Fly.io](/fa/install/fly)
    - [Hetzner](/fa/install/hetzner)
    - [exe.dev](/fa/install/exe-dev)

    در فضای ابری، **Gateway روی سرور اجرا می‌شود** و از لپ‌تاپ/تلفن خود
    از طریق رابط کنترل (یا Tailscale/SSH) به آن دسترسی پیدا می‌کنید. وضعیت و فضای کاری شما روی سرور قرار دارند، بنابراین
    میزبان را منبع حقیقت در نظر بگیرید و از آن پشتیبان تهیه کنید.

    **Nodeها** (Mac/iOS/Android/بدون رابط) را با آن Gateway ابری جفت کنید تا
    صفحه‌نمایش/دوربین/canvas محلی یا اجرای فرمان روی لپ‌تاپ ممکن باشد، درحالی‌که Gateway در
    فضای ابری باقی می‌ماند.

    مرکز: [پلتفرم‌ها](/fa/platforms). دسترسی از راه دور: [Gateway از راه دور](/fa/gateway/remote).
    Nodeها: [Nodeها](/fa/nodes)، [CLI مربوط به Nodeها](/fa/cli/nodes).

  </Accordion>

  <Accordion title="آیا می‌توانم از OpenClaw بخواهم خودش را به‌روزرسانی کند؟">
    ممکن است، اما توصیه نمی‌شود. فرایند به‌روزرسانی ممکن است Gateway را راه‌اندازی مجدد کند (و
    نشست فعال را قطع کند)، ممکن است به یک checkout تمیز گیت نیاز داشته باشد و شاید تأیید بخواهد.
    اجرای به‌روزرسانی‌ها از پوسته توسط اپراتور امن‌تر است.

    ```bash
    openclaw update
    openclaw update status
    openclaw update --channel stable|extended-stable|beta|dev
    openclaw update --tag <dist-tag|version>
    openclaw update --no-restart
    ```

    خودکارسازی از طریق یک عامل:

    ```bash
    openclaw update --yes --no-restart
    openclaw gateway restart
    ```

    مستندات: [به‌روزرسانی](/fa/cli/update)، [به‌روزرسانی](/fa/install/updating).

  </Accordion>

  <Accordion title="راه‌اندازی اولیه دقیقاً چه‌کار می‌کند؟">
    `openclaw onboard` مسیر پیشنهادی راه‌اندازی است. در **حالت محلی** مراحل زیر را طی می‌کند:

    1. **مدل/احراز هویت** - OAuth ارائه‌دهنده، کلیدهای API یا احراز هویت دستی (از جمله گزینه‌های محلی مانند LM Studio)؛ یک مدل پیش‌فرض انتخاب کنید.
    2. **فضای کاری** - مکان و فایل‌های راه‌اندازی اولیه.
    3. **Gateway** - درگاه، نشانی bind، حالت احراز هویت، دسترسی از طریق Tailscale.
    4. **کانال‌ها** - کانال‌های گفت‌وگوی داخلی و Pluginهای رسمی: iMessage، Discord، Feishu، Google Chat، Mattermost، Microsoft Teams، QQ Bot، Signal، Slack، Telegram، WhatsApp و موارد دیگر.
    5. **Daemon** - LaunchAgent ‏(macOS)، واحد کاربری systemd ‏(Linux/WSL2) یا Windows Scheduled Task بومی.
    6. **بررسی سلامت** - Gateway را راه‌اندازی می‌کند و فعال‌بودن آن را تأیید می‌کند.
    7. **Skills** - Skills پیشنهادی و وابستگی‌های اختیاری را نصب می‌کند.

    مدت‌زمان مورد انتظار را از ابتدا مشخص می‌کند و اگر مدل پیکربندی‌شده ناشناخته باشد
    یا احراز هویت نداشته باشد، هشدار می‌دهد. شرح کامل: [راه‌اندازی اولیه (CLI)](/fa/start/wizard).

  </Accordion>

  <Accordion title="آیا برای اجرای این به اشتراک Claude یا OpenAI نیاز دارم؟">
    خیر. OpenClaw را با **کلیدهای API** ‏(Anthropic/OpenAI/سایر ارائه‌دهندگان) یا **مدل‌های صرفاً محلی**
    اجرا کنید تا داده‌هایتان روی دستگاه خودتان باقی بماند. اشتراک‌ها (Claude Pro/Max، ChatGPT/Codex)
    روش‌های اختیاری برای احراز هویت نزد این ارائه‌دهندگان هستند.

    برای Anthropic: یک **کلید API** صورتحساب استاندارد به‌ازای مصرف ارائه می‌دهد؛ **Claude CLI**
    از ورود موجود Claude Code روی همان میزبان استفاده می‌کند. Anthropic در حال حاضر
    مسیر غیرتعاملی `claude -p` در Claude CLI را استفاده برنامه‌نویسی‌شده/Agent SDK در نظر می‌گیرد که
    همچنان از محدودیت‌های طرح اشتراک شما مصرف می‌کند؛ پیش از اتکا به رفتار اشتراک،
    مستندات فعلی صورتحساب Anthropic را بررسی کنید. برای میزبان‌های بلندمدت Gateway و خودکارسازی
    اشتراکی، کلید API ‏Anthropic انتخاب قابل‌پیش‌بینی‌تری است.

    OAuth مربوط به OpenAI Codex (اشتراک ChatGPT/Codex) برای مدل‌های عامل کاملاً پشتیبانی می‌شود.
    OpenClaw همچنین از گزینه‌های میزبانی‌شده مبتنی بر اشتراک، از جمله **Qwen Cloud
    Coding Plan**، **MiniMax Coding Plan** و **Z.AI / GLM Coding Plan** پشتیبانی می‌کند.

    مستندات: [Anthropic](/fa/providers/anthropic)، [OpenAI](/fa/providers/openai)،
    [Qwen Cloud](/fa/providers/qwen)، [MiniMax](/fa/providers/minimax)، [Z.AI (GLM)](/fa/providers/zai)،
    [مدل‌های محلی](/fa/gateway/local-models)، [مدل‌ها](/fa/concepts/models).

  </Accordion>

  <Accordion title="آیا می‌توانم بدون کلید API از اشتراک Claude Max استفاده کنم؟">
    بله. OpenClaw از استفاده مجدد Claude CLI برای طرح‌های Pro/Max/Team/Enterprise پشتیبانی می‌کند. Anthropic
    در حال حاضر مسیر `claude -p` مورد استفاده OpenClaw را مصرف از طرح اشتراک و مشمول
    محدودیت‌های طرح شما می‌داند، نه سهمیه رایگان جداگانه؛ برای جزئیات فعلی صورتحساب و پیوندهای
    مقاله‌های پشتیبانی خود Anthropic، به [Anthropic](/fa/providers/anthropic) مراجعه کنید.
    برای قابل‌پیش‌بینی‌ترین راه‌اندازی سمت سرور، به‌جای آن از یک کلید API ‏Anthropic استفاده کنید.
  </Accordion>

  <Accordion title="آیا از احراز هویت اشتراک Claude (Claude Pro یا Max) پشتیبانی می‌شود؟">
    بله، از طریق استفاده مجدد Claude CLI. نحوه محاسبه صورتحساب Anthropic برای استفاده از `claude -p`/Agent SDK
    در گذر زمان تغییر کرده است؛ پیش از اتکا به رفتار مشخصی در صورتحساب، برای وضعیت فعلی
    و پیوندهای تاریخ‌دار به مقاله‌های پشتیبانی Anthropic به [Anthropic](/fa/providers/anthropic)
    مراجعه کنید.

    احراز هویت با setup-token در Anthropic نیز همچنان یک مسیر توکن پشتیبانی‌شده است، اما OpenClaw در صورت امکان
    استفادهٔ مجدد از Claude CLI و `claude -p` را ترجیح می‌دهد. برای بارهای کاری تولیدی یا چندکاربره،
    کلید API ‏Anthropic همچنان انتخابی امن‌تر و قابل‌پیش‌بینی‌تر است. دیگر گزینه‌های میزبانی‌شدهٔ اشتراکی: [OpenAI](/fa/providers/openai)، [Qwen Cloud](/fa/providers/qwen)،
    [MiniMax](/fa/providers/minimax)، [Z.AI (GLM)](/fa/providers/zai).

  </Accordion>

</AccordionGroup>

<a id="why-am-i-seeing-http-429-ratelimiterror-from-anthropic"></a>

<AccordionGroup>
  <Accordion title="چرا خطای HTTP 429 rate_limit_error را از Anthropic می‌بینم؟">
    **سهمیه/محدودیت نرخ Anthropic** شما برای بازهٔ فعلی به پایان رسیده است. در **Claude
    CLI**، منتظر بازنشانی بازه بمانید یا طرح خود را ارتقا دهید. در صورت استفاده از **کلید API ‏Anthropic**،
    میزان استفاده/صورت‌حساب را در Anthropic Console بررسی کنید و در صورت نیاز محدودیت‌ها را افزایش دهید.

    اگر پیام مشخصاً `Extra usage is required for long context requests` است،
    درخواست در تلاش است از پنجرهٔ زمینهٔ 1M ‏Anthropic استفاده کند (یک مدل Claude 4.x با قابلیت عمومی 1M
    یا پیکربندی قدیمی `params.context1m: true`) و اعتبارنامهٔ فعلی شما
    واجد شرایط صورت‌حساب زمینهٔ طولانی نیست.

    یک **مدل جایگزین** تنظیم کنید تا هنگام محدودیت نرخ یک ارائه‌دهنده، OpenClaw همچنان پاسخ دهد.
    [مدل‌ها](/fa/cli/models)، [OAuth](/fa/concepts/oauth) و
    [نیاز Anthropic 429 به استفادهٔ اضافی برای زمینهٔ طولانی](/fa/gateway/troubleshooting#anthropic-429-extra-usage-required-for-long-context) را ببینید.

  </Accordion>

  <Accordion title="آیا AWS Bedrock پشتیبانی می‌شود؟">
    بله. OpenClaw یک ارائه‌دهندهٔ داخلی **Amazon Bedrock (Converse)** دارد. با وجود نشانگرهای محیطی AWS
    ‏(`AWS_ACCESS_KEY_ID`، `AWS_PROFILE`، `AWS_BEARER_TOKEN_BEDROCK`)،
    OpenClaw ارائه‌دهندهٔ ضمنی Bedrock را برای کشف مدل به‌طور خودکار فعال می‌کند؛ در غیر این صورت
    `plugins.entries.amazon-bedrock.config.discovery.enabled: true` را تنظیم کنید یا یک ورودی دستی
    برای ارائه‌دهنده اضافه کنید. [Amazon Bedrock](/fa/providers/bedrock) و [ارائه‌دهندگان مدل](/fa/providers/models) را ببینید.
    اگر جریان کلید مدیریت‌شده را ترجیح می‌دهید، پراکسی سازگار با OpenAI در جلوی Bedrock همچنان گزینه‌ای معتبر است.
  </Accordion>

  <Accordion title="احراز هویت Codex چگونه کار می‌کند؟">
    OpenClaw از **OpenAI Codex** از طریق OAuth (ورود به ChatGPT) پشتیبانی می‌کند. یک راه‌اندازی
    تازه بدون مدل اصلی، از دقیقاً `openai/gpt-5.6-sol` برای
    احراز هویت اشتراک ChatGPT/Codex به‌همراه اجرای بومی app-server ‏Codex استفاده می‌کند.
    احراز هویت مجدد، مدل صریح موجود را حفظ می‌کند، از جمله
    `openai/gpt-5.5`. اگر فضای کاری Codex مدل GPT-5.6 را ارائه نمی‌دهد،
    `openai/gpt-5.5` را صریحاً انتخاب کنید؛ OpenClaw بی‌سروصدا به نسخهٔ پایین‌تر تنزل نمی‌دهد. ارجاعات
    مدل با پیشوند قدیمی Codex، پیکربندی قدیمی‌ای هستند که توسط `openclaw doctor
    --fix` تعمیر می‌شوند. دسترسی مستقیم با کلید API ‏OpenAI برای سطوح غیرعاملی API ‏OpenAI
    همچنان در دسترس است و از طریق یک پروفایل مرتب‌شدهٔ کلید API ‏`openai`، برای مدل‌های
    عامل نیز قابل استفاده است. [ارائه‌دهندگان مدل](/fa/concepts/model-providers) و
    [راه‌اندازی اولیه (CLI)](/fa/start/wizard) را ببینید.
  </Accordion>

  <Accordion title="چرا OpenClaw هنوز پیشوند قدیمی OpenAI Codex را ذکر می‌کند؟">
    `openai` شناسهٔ فعلی ارائه‌دهنده و پروفایل احراز هویت برای کلیدهای API ‏OpenAI و
    OAuth ‏ChatGPT/Codex است؛ OpenAI Codex در آن ادغام شده است. ممکن است هنوز پیشوند قدیمی
    `openai-codex` را در پیکربندی‌های قدیمی‌تر و هشدارهای مهاجرت ببینید:

    - `openai/gpt-5.6-sol` = راه‌اندازی تازهٔ اشتراک ChatGPT/Codex با زمان‌اجرای بومی Codex برای نوبت‌های عامل.
    - `openai/gpt-5.5` = انتخاب صریح پشتیبانی‌شده برای پیکربندی موجود یا حساب‌های بدون دسترسی به GPT-5.6.
    - ارجاعات مدل قدیمی `openai-codex/*` = مسیر قدیمی که توسط `openclaw doctor --fix` تعمیر می‌شود.
    - `openai/gpt-5.5` به‌همراه یک پروفایل مرتب‌شدهٔ کلید API ‏`openai` = احراز هویت با کلید API برای یک مدل عامل OpenAI.
    - شناسه‌های قدیمی پروفایل احراز هویت `openai-codex` = شناسه‌های قدیمی که توسط `openclaw doctor --fix` مهاجرت می‌کنند.

    صورت‌حساب مستقیم OpenAI Platform را می‌خواهید؟ `OPENAI_API_KEY` را تنظیم کنید. احراز هویت اشتراک
    ChatGPT/Codex را می‌خواهید؟ `openclaw models auth login --provider openai` را اجرا کنید. ارجاعات
    مدل را زیر ارائه‌دهندهٔ رسمی `openai/*` نگه دارید. راه‌اندازی تازهٔ اشتراک
    از دقیقاً `openai/gpt-5.6-sol` استفاده می‌کند؛ doctor ارجاعات قدیمی با پیشوند Codex را
    بدون ارتقای انتخاب صریح `openai/gpt-5.5` تعمیر می‌کند.

  </Accordion>

  <Accordion title="چرا محدودیت‌های OAuth ‏Codex می‌تواند با وب ChatGPT متفاوت باشد؟">
    OAuth ‏Codex از بازه‌های سهمیهٔ مدیریت‌شده توسط OpenAI و وابسته به طرح استفاده می‌کند که ممکن است حتی در یک
    حساب یکسان، با تجربهٔ وب‌سایت/برنامهٔ ChatGPT متفاوت باشند.

    `openclaw models status` بازه‌های استفاده/سهمیهٔ ارائه‌دهنده را که در حال حاضر قابل مشاهده‌اند نشان می‌دهد، اما
    دسترسی‌های وب ChatGPT را ابداع نمی‌کند یا به دسترسی مستقیم API تبدیل نمی‌کند. برای
    مسیر مستقیم صورت‌حساب/محدودیت OpenAI Platform، از `openai/*` همراه یک کلید API استفاده کنید.

  </Accordion>

  <Accordion title="آیا از احراز هویت اشتراک OpenAI ‏(OAuth ‏Codex) پشتیبانی می‌کنید؟">
    بله، به‌طور کامل. OpenAI صریحاً استفاده از OAuth اشتراک را در ابزارها/گردش‌کارهای خارجی
    مانند OpenClaw مجاز می‌داند. راه‌اندازی اولیه می‌تواند جریان OAuth را برای شما اجرا کند.

    [OAuth](/fa/concepts/oauth)، [ارائه‌دهندگان مدل](/fa/concepts/model-providers) و [راه‌اندازی اولیه (CLI)](/fa/start/wizard) را ببینید.

  </Accordion>

  <Accordion title="چگونه OAuth ‏Gemini CLI را راه‌اندازی کنم؟">
    Gemini CLI از **جریان احراز هویت Plugin** استفاده می‌کند، نه شناسهٔ کلاینت یا رمز در `openclaw.json`.

    1. Gemini CLI را به‌صورت محلی نصب کنید تا `gemini` در `PATH` قرار گیرد:
       - Homebrew: `brew install gemini-cli`
       - npm: `npm install -g @google/gemini-cli`
    2. Plugin را فعال کنید: `openclaw plugins enable google`
    3. وارد شوید: `openclaw models auth login --provider google-gemini-cli --set-default`
    4. مدل پیش‌فرض پس از ورود: `google/gemini-3.1-pro-preview` (زمان‌اجرا `google-gemini-cli`)
    5. درخواست‌ها پس از ورود ناموفق‌اند؟ `GOOGLE_CLOUD_PROJECT` یا `GOOGLE_CLOUD_PROJECT_ID` را روی میزبان Gateway تنظیم و دوباره تلاش کنید.

    توکن‌های OAuth در پروفایل‌های احراز هویت روی میزبان Gateway ذخیره می‌شوند. جزئیات: [Google](/fa/providers/google)، [ارائه‌دهندگان مدل](/fa/concepts/model-providers).

  </Accordion>

  <Accordion title="آیا یک مدل محلی برای گفت‌وگوهای معمولی مناسب است؟">
    معمولاً خیر. OpenClaw به زمینهٔ بزرگ و ایمنی قوی نیاز دارد؛ کارت‌های کوچک زمینه را قطع می‌کنند
    و فیلترهای ایمنی سمت ارائه‌دهنده را نادیده می‌گیرند. اگر ناچارید، **بزرگ‌ترین** ساخت مدل ممکن را
    به‌صورت محلی اجرا کنید (LM Studio)؛ [مدل‌های محلی](/fa/gateway/local-models) را ببینید. مدل‌های کوچک‌تر/کوانتیزه‌شده
    خطر تزریق پرامپت را افزایش می‌دهند؛ [امنیت](/fa/gateway/security) را ببینید.
  </Accordion>

  <Accordion title="چگونه ترافیک مدل میزبانی‌شده را در یک منطقهٔ مشخص نگه دارم؟">
    نقاط پایانی مقید به منطقه را انتخاب کنید. OpenRouter گزینه‌های میزبانی‌شده در آمریکا را برای MiniMax، Kimi
    و GLM ارائه می‌دهد؛ برای نگه‌داشتن داده‌ها در همان منطقه، گونهٔ میزبانی‌شده در آمریکا را انتخاب کنید. همچنان می‌توانید
    Anthropic/OpenAI را با `models.mode: "merge"` در کنار این‌ها فهرست کنید تا گزینه‌های جایگزین
    ضمن رعایت ارائه‌دهندهٔ منطقه‌ای انتخاب‌شده، در دسترس بمانند.
  </Accordion>

  <Accordion title="آیا برای نصب این باید Mac Mini بخرم؟">
    خیر. OpenClaw روی macOS یا Linux اجرا می‌شود (Windows از طریق WSL2). Mac mini انتخاب محبوبی
    برای میزبان همیشه‌روشن است، اما یک VPS کوچک، سرور خانگی یا دستگاهی در ردهٔ Raspberry Pi نیز مناسب است.

    فقط **برای ابزارهای مختص macOS** به Mac نیاز دارید. برای iMessage، از [iMessage](/fa/channels/imessage)
    همراه `imsg` روی هر Mac واردشده به Messages استفاده کنید؛ اگر Gateway روی Linux یا جای دیگری اجرا می‌شود،
    `channels.imessage.cliPath` را روی یک پوشش SSH تنظیم کنید که `imsg` را روی آن Mac اجرا کند. برای دیگر
    ابزارهای مختص macOS، Gateway را روی یک Mac اجرا کنید یا یک Node ‏macOS را جفت کنید.

    مستندات: [iMessage](/fa/channels/imessage)، [Nodeها](/fa/nodes)، [حالت راه‌دور Mac](/fa/platforms/mac/remote).

  </Accordion>

  <Accordion title="آیا برای پشتیبانی از iMessage به Mac mini نیاز دارم؟">
    به **یک دستگاه macOS** واردشده به Messages نیاز دارید؛ نه لزوماً Mac mini، هر
    Mac مناسب است. از [iMessage](/fa/channels/imessage) همراه `imsg` استفاده کنید؛ Gateway می‌تواند روی همان
    Mac یا جای دیگری با پوشش SSH ‏`cliPath` اجرا شود.

    راه‌اندازی‌های رایج:

    - Gateway روی Linux/VPS و `channels.imessage.cliPath` تنظیم‌شده روی یک پوشش SSH که `imsg` را روی Mac واردشده به Messages اجرا می‌کند.
    - برای ساده‌ترین راه‌اندازی تک‌دستگاهی، همه‌چیز روی یک Mac.

    مستندات: [iMessage](/fa/channels/imessage)، [Nodeها](/fa/nodes)، [حالت راه‌دور Mac](/fa/platforms/mac/remote).

  </Accordion>

  <Accordion title="اگر برای اجرای OpenClaw یک Mac mini بخرم، می‌توانم آن را به MacBook Pro خود متصل کنم؟">
    بله. **Mac mini می‌تواند Gateway را اجرا کند** و MacBook Pro شما به‌عنوان یک **Node**
    (دستگاه همراه) متصل می‌شود. Nodeها Gateway را اجرا نمی‌کنند؛ آن‌ها قابلیت‌هایی مانند
    صفحه‌نمایش/دوربین/بوم و `system.run` را روی آن دستگاه اضافه می‌کنند.

    الگوی رایج: Gateway روی Mac mini همیشه‌روشن؛ MacBook Pro برنامهٔ macOS یا یک
    میزبان Node را اجرا می‌کند و با Gateway جفت می‌شود. با `openclaw nodes status` / `openclaw nodes list` بررسی کنید.

    مستندات: [Nodeها](/fa/nodes)، [CLI ‏Nodeها](/fa/cli/nodes).

  </Accordion>

  <Accordion title="آیا می‌توانم از Bun استفاده کنم؟">
    می‌توانید از Bun برای نصب وابستگی‌ها یا اجرای اسکریپت‌های بسته استفاده کنید. CLI و
    Gateway ‏OpenClaw به **Node** نیاز دارند، زیرا مخزن وضعیت رسمی از `node:sqlite` استفاده می‌کند؛ Bun
    آن API را ارائه نمی‌دهد.
  </Accordion>

  <Accordion title="Telegram: چه چیزی در allowFrom قرار می‌گیرد؟">
    `channels.telegram.allowFrom`، **شناسهٔ کاربری Telegram فرستندهٔ انسانی** (عددی) است،
    نه نام کاربری ربات. راه‌اندازی فقط شناسه‌های عددی کاربر را درخواست می‌کند؛ `openclaw doctor --fix`
    می‌تواند برای رفع ورودی‌های قدیمی `@username` تلاش کند.

    امن‌تر (بدون ربات شخص ثالث): به ربات خود پیام خصوصی دهید، `openclaw logs --follow` را اجرا کنید و `from.id` را بخوانید.

    Bot API رسمی: به ربات خود پیام خصوصی دهید، `https://api.telegram.org/bot<bot_token>/getUpdates` را فراخوانی کنید و `message.from.id` را بخوانید.

    شخص ثالث (با حریم خصوصی کمتر): به `@userinfobot` یا `@getidsbot` پیام خصوصی دهید.

    [کنترل دسترسی Telegram](/fa/channels/telegram#access-control-and-activation) را ببینید.

  </Accordion>

  <Accordion title="آیا چند نفر می‌توانند از یک شمارهٔ WhatsApp با نمونه‌های متفاوت OpenClaw استفاده کنند؟">
    بله، از طریق **مسیریابی چندعاملی**. پیام خصوصی WhatsApp هر فرستنده (`peer: { kind: "direct", id: "+15551234567" }`) را به یک `agentId` متفاوت متصل کنید تا هر شخص فضای کاری و مخزن نشست خود را داشته باشد. پاسخ‌ها همچنان از **همان حساب WhatsApp** ارسال می‌شوند؛ کنترل دسترسی پیام خصوصی (`channels.whatsapp.dmPolicy` / `channels.whatsapp.allowFrom`) برای هر حساب سراسری است. [مسیریابی چندعاملی](/fa/concepts/multi-agent) و [WhatsApp](/fa/channels/whatsapp) را ببینید.
  </Accordion>

  <Accordion title='آیا می‌توانم یک عامل «گفت‌وگوی سریع» و یک عامل «Opus برای کدنویسی» اجرا کنم؟'>
    بله. از مسیریابی چندعاملی استفاده کنید: به هر عامل مدل پیش‌فرض خودش را بدهید، سپس مسیرهای
    ورودی (حساب ارائه‌دهنده یا همتایان مشخص) را به هر عامل متصل کنید. نمونهٔ پیکربندی:
    [مسیریابی چندعاملی](/fa/concepts/multi-agent). همچنین [مدل‌ها](/fa/concepts/models) و
    [پیکربندی](/fa/gateway/configuration) را ببینید.
  </Accordion>

  <Accordion title="آیا Homebrew روی Linux کار می‌کند؟">
    بله، از طریق Linuxbrew:

    ```bash
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    echo 'eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"' >> ~/.profile
    eval "$(/home/linuxbrew/.linuxbrew/bin/brew shellenv)"
    brew install <formula>
    ```

    هنگام اجرای OpenClaw از طریق systemd، مطمئن شوید PATH سرویس شامل
    `/home/linuxbrew/.linuxbrew/bin` (یا پیشوند brew شما) است تا ابزارهای نصب‌شده با `brew`
    در پوسته‌های غیرورودی پیدا شوند. ساخت‌های اخیر همچنین دایرکتوری‌های رایج bin کاربر را در سرویس‌های
    systemd ‏Linux به ابتدا اضافه می‌کنند (برای نمونه `~/.local/bin`، `~/.npm-global/bin`،
    `~/.local/share/pnpm`، `~/.bun/bin`) و در صورت تنظیم، `PNPM_HOME`، `NPM_CONFIG_PREFIX`،
    `BUN_INSTALL`، `VOLTA_HOME`، `ASDF_DATA_DIR`، `NVM_DIR` و `FNM_DIR` را رعایت می‌کنند.

  </Accordion>

  <Accordion title="تفاوت نصب قابل‌تغییر git با نصب npm">
    - **نصب قابل‌تغییر (git):** دریافت کامل کد منبع، قابل‌ویرایش و مناسب‌تر برای مشارکت‌کنندگان. به‌صورت محلی می‌سازید و می‌توانید کد/مستندات را اصلاح کنید.
    - **نصب npm:** نصب سراسری CLI، بدون مخزن و مناسب برای «فقط اجرا کردن». به‌روزرسانی‌ها از dist-tagهای npm می‌آیند.

    مستندات: [شروع به کار](/fa/start/getting-started)، [به‌روزرسانی](/fa/install/updating).

  </Accordion>

  <Accordion title="آیا می‌توانم بعداً بین نصب‌های npm و git جابه‌جا شوم؟">
    بله، با `openclaw update --channel ...` روی یک نصب موجود. این کار **داده‌های شما را
    حذف نمی‌کند**؛ فقط نحوه نصب کد OpenClaw تغییر می‌کند. وضعیت (`~/.openclaw`) و
    فضای کاری (`~/.openclaw/workspace`) بدون تغییر باقی می‌مانند.

    از npm به git:

    ```bash
    openclaw update --channel dev
    ```

    از git به npm:

    ```bash
    openclaw update --channel stable
    ```

    برای پیش‌نمایش اولیه تغییر حالت برنامه‌ریزی‌شده، `--dry-run` را اضافه کنید. به‌روزرسان، اقدامات تکمیلی Doctor
    را اجرا می‌کند، منابع Plugin را برای کانال مقصد تازه‌سازی می‌کند و Gateway را مجدداً راه‌اندازی
    می‌کند، مگر اینکه `--no-restart` را ارسال کنید.

    نصب‌کننده نیز می‌تواند هرکدام از این حالت‌ها را اجباری کند:

    ```bash
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method git
    curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install.sh | bash -s -- --install-method npm
    ```

    نکات پشتیبان‌گیری: [محل ذخیره موارد روی دیسک](/fa/help/faq#where-things-live-on-disk).

  </Accordion>

  <Accordion title="آیا باید Gateway را روی لپ‌تاپم اجرا کنم یا روی یک VPS؟">
    قابلیت اطمینان 24/7 می‌خواهید؟ از یک **VPS** استفاده کنید. کمترین دردسر را می‌خواهید و با
    خواب/راه‌اندازی مجدد مشکلی ندارید؟ آن را به‌صورت محلی اجرا کنید.

    **لپ‌تاپ (Gateway محلی)**

    - **مزایا:** بدون هزینه سرور، دسترسی مستقیم به فایل‌های محلی، پنجره زنده مرورگر.
    - **معایب:** خواب دستگاه/قطعی شبکه اتصال آن را قطع می‌کند، به‌روزرسانی‌ها/راه‌اندازی‌های مجدد سیستم‌عامل در کار آن وقفه ایجاد می‌کنند، دستگاه باید بیدار بماند.

    **VPS / ابر**

    - **مزایا:** همیشه روشن، شبکه پایدار، بدون مشکلات خواب لپ‌تاپ، تداوم اجرای آسان‌تر.
    - **معایب:** اغلب بدون رابط گرافیکی (از اسکرین‌شات استفاده کنید)، فقط دسترسی راه‌دور به فایل‌ها، نیازمند SSH برای به‌روزرسانی‌ها.

    WhatsApp/Telegram/Slack/Mattermost/Discord همگی از طریق یک VPS به‌خوبی کار می‌کنند؛ موازنه اصلی
    میان مرورگر بدون رابط گرافیکی و پنجره قابل‌مشاهده است. [مرورگر](/fa/tools/browser) را ببینید.

    توصیه پیش‌فرض: اگر قبلاً با قطع اتصال Gateway مواجه شده‌اید، از VPS استفاده کنید؛ اجرای محلی زمانی عالی است
    که فعالانه از Mac استفاده می‌کنید و به دسترسی به فایل‌های محلی یا خودکارسازی رابط کاربری
    مرورگر قابل‌مشاهده نیاز دارید.

  </Accordion>

  <Accordion title="اجرای OpenClaw روی یک دستگاه اختصاصی چقدر اهمیت دارد؟">
    الزامی نیست، اما برای قابلیت اطمینان و جداسازی توصیه می‌شود.

    - **میزبان اختصاصی (VPS/Mac mini/Raspberry Pi):** همیشه روشن، وقفه‌های کمتر ناشی از خواب/راه‌اندازی مجدد، مجوزهای مرتب‌تر، تداوم اجرای آسان‌تر.
    - **لپ‌تاپ/رایانه رومیزی مشترک:** برای آزمایش و استفاده فعال مناسب است، اما هنگام خواب یا به‌روزرسانی دستگاه باید انتظار وقفه را داشته باشید.

    بهترین ترکیب هر دو حالت: Gateway را روی یک میزبان اختصاصی نگه دارید و لپ‌تاپ خود را به‌عنوان یک
    **Node** برای ابزارهای محلی صفحه‌نمایش/دوربین/اجرا جفت کنید. [Nodeها](/fa/nodes) و [امنیت](/fa/gateway/security) را ببینید.

  </Accordion>

  <Accordion title="حداقل نیازمندی‌های VPS و سیستم‌عامل توصیه‌شده چیست؟">
    - **حداقل مطلق:** 1 vCPU، ‏1 GB رم، حدود 500 MB فضای دیسک.
    - **توصیه‌شده:** ‏1-2 vCPU، ‏2 GB+ رم برای ظرفیت اضافه (گزارش‌ها، رسانه، چندین کانال). ابزارهای Node و خودکارسازی مرورگر ممکن است منابع زیادی مصرف کنند.

    سیستم‌عامل: **Ubuntu LTS** (یا هر Debian/Ubuntu مدرن)؛ مسیر نصب Linux که بیشترین آزمایش را پشت سر گذاشته است.

    مستندات: [Linux](/fa/platforms/linux)، [میزبانی VPS](/fa/vps).

  </Accordion>

  <Accordion title="آیا می‌توانم OpenClaw را در یک VM اجرا کنم و نیازمندی‌های آن چیست؟">
    بله. با یک VM مانند VPS رفتار کنید: باید همیشه روشن و قابل‌دسترسی باشد و رم کافی
    برای Gateway و هر کانالی که فعال می‌کنید داشته باشد.

    - **حداقل مطلق:** 1 vCPU، ‏1 GB رم.
    - **توصیه‌شده:** ‏2 GB+ رم برای چندین کانال، خودکارسازی مرورگر یا ابزارهای رسانه‌ای.
    - **سیستم‌عامل:** Ubuntu LTS یا یک Debian/Ubuntu مدرن دیگر.

    در Windows، برای راه‌اندازی دسکتاپ از **Windows Hub** یا برای یک VM ‏Gateway به سبک Linux
    با سازگاری گسترده ابزارها از WSL2 استفاده کنید. [Windows](/fa/platforms/windows)، [میزبانی VPS](/fa/vps) را ببینید.
    اجرای macOS در یک VM: [VM ‏macOS](/fa/install/macos-vm) را ببینید.

  </Accordion>
</AccordionGroup>

## مرتبط

- [پرسش‌های متداول](/fa/help/faq) - پرسش‌های متداول اصلی (مدل‌ها، نشست‌ها، Gateway، امنیت و موارد دیگر)
- [نمای کلی نصب](/fa/install)
- [شروع به کار](/fa/start/getting-started)
- [عیب‌یابی](/fa/help/troubleshooting)
