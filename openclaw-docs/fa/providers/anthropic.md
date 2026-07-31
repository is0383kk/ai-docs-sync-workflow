---
read_when:
    - می‌خواهید از مدل‌های Anthropic در OpenClaw استفاده کنید
    - می‌خواهید نشست‌های Claude CLI یا Claude Desktop را در رایانه‌های جفت‌شده مرور کنید
summary: استفاده از Anthropic Claude از طریق کلیدهای API یا Claude CLI در OpenClaw
title: Anthropic
x-i18n:
    generated_at: "2026-07-27T15:35:40Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 08b34794352a559d549f7cf0cb88aca9cb537984049367f55be371bd8e0c10f0
    source_path: providers/anthropic.md
    workflow: 16
---

Anthropic خانواده مدل **Claude** را توسعه می‌دهد. OpenClaw از دو روش احراز هویت پشتیبانی می‌کند:

- **کلید API** - دسترسی مستقیم به API شرکت Anthropic با صورت‌حساب مبتنی بر میزان استفاده (مدل‌های `anthropic/*`)
- **Claude CLI** - استفاده مجدد از ورود موجود Claude Code روی همان میزبان

## پیگیری استفاده و هزینه

OpenClaw اعتبارنامه Anthropic موجود را تشخیص می‌دهد و بخش متناسب با استفاده را انتخاب می‌کند:

- اعتبارنامه‌های اشتراک/راه‌اندازی Claude بازه‌های سهمیه و بودجه اختیاری استفاده اضافی را نمایش می‌دهند.
- `ANTHROPIC_ADMIN_KEY` یا `ANTHROPIC_ADMIN_API_KEY` هزینه سازمانی گزارش‌شده توسط ارائه‌دهنده و استفاده از Messages API طی 30 روز را در بخش **Usage** رابط کاربری Control UI نمایش می‌دهد؛ شامل هزینه روزانه، مجموع توکن/کش، مدل‌های پراستفاده و دسته‌بندی هزینه‌ها.
- اعتبارنامه `sk-ant-admin...` ذخیره‌شده در پروفایل ارائه‌دهنده Anthropic به‌طور خودکار به‌عنوان کلید Admin API تشخیص داده می‌شود.

سابقه هزینه Admin API از [API استفاده و هزینه](https://platform.claude.com/docs/en/manage-claude/usage-cost-api) شرکت Anthropic دریافت می‌شود. این مبلغ واقعی صورت‌حساب ارائه‌دهنده است و با هزینه تخمینی OpenClaw که از نشست‌ها محاسبه می‌شود تفاوت دارد.

<Warning>
بک‌اند Claude CLI در OpenClaw، رابط خط فرمان نصب‌شده Claude Code را در
حالت چاپ غیرتعاملی (`claude -p`) اجرا می‌کند. مستندات فعلی Claude Code شرکت Anthropic
این حالت را استفاده برنامه‌نویسی/Agent SDK توصیف می‌کند. به‌روزرسانی پشتیبانی Anthropic در 15 ژوئن 2026
تغییر اعلام‌شده برای صورت‌حساب جداگانه Agent SDK را متوقف کرد: استفاده از Claude
Agent SDK، `claude -p` و برنامه‌های شخص ثالث همچنان از محدودیت‌های استفاده
اشتراک واردشده کسر می‌شود و اعتبار ماهانه Agent SDK که پیش‌تر اعلام شده بود،
تا زمانی که Anthropic این طرح را بازنگری کند در دسترس نیست.

Claude Code تعاملی نیز همچنان از محدودیت‌های طرح Claude واردشده استفاده می‌کند.
احراز هویت با کلید API مستقیماً بر اساس میزان مصرف صورت‌حساب می‌شود و به آن طرح وابسته نیست.
برای میزبان‌های بلندمدت Gateway، خودکارسازی اشتراکی و هزینه قابل‌پیش‌بینی
محیط عملیاتی، از یک کلید API شرکت Anthropic استفاده کنید.

مقالات پشتیبانی فعلی Anthropic ممکن است این رفتار را بدون انتشار نسخه‌ای از
OpenClaw تغییر دهند:

- [مرجع Claude Code CLI](https://code.claude.com/docs/en/cli-usage)
- [استفاده از Claude Agent SDK با طرح Claude](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
- [استفاده از Claude Code با طرح Pro یا Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
- [استفاده از Claude Code با طرح Team یا Enterprise](https://support.claude.com/en/articles/11845131-using-claude-code-with-your-team-or-enterprise-plan)
- [مدیریت هزینه‌های Claude Code](https://code.claude.com/docs/en/costs)

</Warning>

## شروع کار

<Tabs>
  <Tab title="کلید API">
    **بهترین گزینه برای:** دسترسی استاندارد به API و صورت‌حساب مبتنی بر میزان استفاده.

    <Steps>
      <Step title="دریافت کلید API">
        یک کلید API در [کنسول Anthropic](https://console.anthropic.com/) ایجاد کنید.
      </Step>
      <Step title="اجرای راه‌اندازی اولیه">
        ```bash
        openclaw onboard
        # انتخاب کنید: کلید API شرکت Anthropic
        ```

        یا کلید را مستقیماً ارسال کنید:

        ```bash
        openclaw onboard --anthropic-api-key "$ANTHROPIC_API_KEY"
        ```
      </Step>
      <Step title="اطمینان از دردسترس‌بودن مدل">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    ### نمونه پیکربندی

    ```json5
    {
      env: { ANTHROPIC_API_KEY: "example-anthropic-key-not-real" },
      agents: { defaults: { model: { primary: "anthropic/claude-opus-5" } } },
    }
    ```

  </Tab>

  <Tab title="Claude CLI">
    **بهترین گزینه برای:** استفاده مجدد از ورود موجود Claude CLI بدون کلید API جداگانه.

    <Steps>
      <Step title="اطمینان از نصب و ورود Claude CLI">
        با دستور زیر بررسی کنید:

        ```bash
        claude --version
        ```
      </Step>
      <Step title="اجرای راه‌اندازی اولیه">
        ```bash
        openclaw onboard
        # انتخاب کنید: Claude CLI
        ```

        OpenClaw اعتبارنامه‌های موجود Claude CLI را تشخیص می‌دهد و دوباره استفاده می‌کند.
      </Step>
      <Step title="اطمینان از دردسترس‌بودن مدل">
        ```bash
        openclaw models list --provider anthropic
        ```
      </Step>
    </Steps>

    <Note>
    جزئیات راه‌اندازی و زمان اجرا برای بک‌اند Claude CLI در [بک‌اندهای CLI](/fa/gateway/cli-backends) آمده است.
    </Note>

    <Warning>
    استفاده مجدد از Claude CLI مستلزم آن است که فرایند OpenClaw روی همان میزبانی اجرا شود که
    ورود Claude CLI در آن انجام شده است. نصب‌های Docker می‌توانند پوشه خانگی کانتینر را پایدار نگه دارند و
    در همان‌جا وارد Claude Code شوند؛ به
    [بک‌اند Claude CLI در Docker](/fa/install/docker#claude-cli-backend-in-docker) مراجعه کنید.
    سایر نصب‌های کانتینری مانند [Podman](/fa/install/podman)،
    `~/.claude` میزبان را در راه‌اندازی یا زمان اجرا سوار نمی‌کنند؛ در آن‌ها از کلید API شرکت Anthropic استفاده کنید یا
    ارائه‌دهنده‌ای با OAuth مدیریت‌شده توسط OpenClaw، مانند
    [OpenAI Codex](/fa/providers/openai)، انتخاب کنید.
    </Warning>

    ### دریافت توکن راه‌اندازی

    `claude setup-token` را روی هر دستگاهی که Claude Code در آن نصب است اجرا کنید. این دستور
    توکنی بلندمدت با پیشوند `sk-ant-oat01-` چاپ می‌کند.

    هنگام راه‌اندازی اولیه، در برنامه macOS با انتخاب
    **Anthropic setup-token** در بخش **Connect with an API key or token** توکن را جای‌گذاری کنید، یا از دستور زیر استفاده کنید:

    ```bash
    openclaw models auth login --provider anthropic --method setup-token
    ```

    ### نمونه پیکربندی

    ارجاع متعارف مدل Anthropic را همراه با بازنویسی زمان اجرای CLI ترجیح دهید:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-5" },
          models: {
            "anthropic/claude-opus-5": {
              agentRuntime: { id: "claude-cli" },
            },
          },
        },
      },
    }
    ```

    ارجاع‌های قدیمی مدل `claude-cli/claude-opus-4-7` همچنان برای
    سازگاری کار می‌کنند، اما پیکربندی جدید باید انتخاب ارائه‌دهنده/مدل را به‌شکل
    `anthropic/*` نگه دارد و بک‌اند اجرا را در سیاست زمان اجرای ارائه‌دهنده/مدل قرار دهد.

    ### صورت‌حساب و `claude -p`

    OpenClaw برای اجراهای Claude CLI از مسیر غیرتعاملی `claude -p` در Claude Code استفاده می‌کند.
    Anthropic در حال حاضر این مسیر را استفاده برنامه‌نویسی/Agent SDK تلقی می‌کند:

    - به‌روزرسانی پشتیبانی Anthropic در 15 ژوئن 2026، طرح اعتبار جداگانه Agent SDK را که پیش‌تر اعلام شده بود
      متوقف کرد.
    - استفاده Claude Agent SDK در طرح اشتراکی، `claude -p` و برنامه‌های شخص ثالث
      همچنان از محدودیت‌های استفاده اشتراک واردشده کسر می‌شود.
    - اعتبار ماهانه Agent SDK که پیش‌تر اعلام شده بود، تا زمانی که
      Anthropic این طرح را بازنگری کند در دسترس نیست.
    - ورودهای Console/کلید API از صورت‌حساب پرداخت بر اساس مصرف API استفاده می‌کنند و
      اعتبار Agent SDK اشتراک را دریافت نمی‌کنند.

برای اطلاعیه توقف، به [مقاله طرح Agent SDK
شرکت Anthropic](https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan)
و برای رفتار اشتراک Claude Code به مقالات طرح
[Pro/Max](https://support.claude.com/en/articles/11145838-use-claude-code-with-your-pro-or-max-plan)
و
[Team/Enterprise](https://support.claude.com/en/articles/11845131-use-claude-code-with-your-team-or-enterprise-plan)
مراجعه کنید.

    Anthropic می‌تواند رفتار صورت‌حساب و محدودیت نرخ Claude Code را بدون انتشار نسخه‌ای از
    OpenClaw تغییر دهد. هر زمان پیش‌بینی‌پذیری صورت‌حساب اهمیت دارد، `claude auth status`، `/status` و
    مستندات پیوندشده Anthropic را بررسی کنید.

    <Tip>
    برای خودکارسازی اشتراکی محیط عملیاتی، به‌جای
    Claude CLI از کلید API شرکت Anthropic استفاده کنید. OpenClaw همچنین از گزینه‌های اشتراکی
    [OpenAI Codex](/fa/providers/openai)، [Qwen Cloud](/fa/providers/qwen)،
    [MiniMax](/fa/providers/minimax) و [Z.AI / GLM](/fa/providers/zai) پشتیبانی می‌کند.
    </Tip>

  </Tab>
</Tabs>

## نشست‌های Claude در رایانه‌های مختلف

Plugin همراه Anthropic یک گروه **Claude Code** به نوار کناری معمول نشست‌ها
اضافه می‌کند. ردیف‌ها در پنل معمول Chat باز می‌شوند. این Plugin نشست‌های بایگانی‌نشده Claude
Code را در Gateway و میزبان‌های Node متصل شناسایی می‌کند:

- نشست‌های Claude CLI از رکوردهای معتبر فهرست پروژه می‌آیند. برای رونوشت‌های
  فهرست‌نشده، یک سازوکار بازگشت محدود فراداده، نشست‌های هم‌زمان تعاملیِ غیرزنجیره‌جانبی
  (`cli`) و نشست‌های CLI بدون رابط Agent SDK (`sdk-cli`) را در مسیر
  `~/.claude/projects/` تشخیص می‌دهد.
- نشست‌های Claude Desktop زمانی از عنوان Desktop، زمان فعالیت و
  وضعیت بایگانی استفاده می‌کنند که فراداده آن به همان شناسه نشست Claude Code اشاره کند.
- نشستی که فقط متعلق به CLI است پرچم بایگانی ندارد، بنابراین تا زمانی که
  رونوشت آن موجود باشد قابل‌مشاهده می‌ماند.

برای شناسایی، هیچ پیکربندی اضافی OpenClaw لازم نیست. Plugin شرکت Anthropic
به‌صورت همراه ارائه شده و به‌طور پیش‌فرض فعال است؛ یک Node بومی macOS هنگامی که پوشه محلی
`~/.claude/projects/` وجود داشته باشد، فرمان‌های فقط‌خواندنی نشست Claude را اعلام می‌کند.
هنگامی که این فرمان‌ها برای نخستین بار ظاهر می‌شوند، ارتقای جفت‌سازی Node را تأیید کنید.

نوار کناری ردیف‌ها را بر اساس میزبان Gateway یا Node جفت‌شده گروه‌بندی می‌کند و به‌محض پاسخ هر رایانه،
جدیدترین صفحه محدود آن میزبان را نمایش می‌دهد. پس از تغییر اتصال میزبان،
هنگامی که صفحه دوباره در کانون قرار می‌گیرد و هنگام قابل‌مشاهده‌بودن حداکثر هر
30 ثانیه دوباره همگام‌سازی می‌کند؛ بنابراین نشست‌های Claude که خارج از OpenClaw ایجاد شده‌اند
بدون بارگذاری مجدد ظاهر می‌شوند. کاتالوگ تغییریافته، بررسی تکمیلی سریع‌تری دریافت می‌کند. از **Load more
sessions** زیر یک گروه کاتالوگ استفاده کنید تا صفحه بعدی برای هر میزبانی که
سابقه بیشتری دارد افزوده شود؛ ردیف‌های افزوده‌شده قابل‌مشاهده می‌مانند و هنگام
تازه‌سازی‌ها تا همان عمق دوباره دریافت می‌شوند. کلاینت‌های کاتالوگ از `sessions.catalog.list` استفاده می‌کنند؛ بازکردن یک ردیف از
`sessions.catalog.read` استفاده می‌کند.

در دست گرفتن ترمینال، `claude` را پیش از PATH سرویس/دیمون، از PATH پوسته ورود
کاربر مالک میزبان رفع می‌کند. این کار نشست‌های اجراشده از برنامه را با
Claude CLI که اپراتور در یک ترمینال عادی دریافت می‌کند هم‌راستا نگه می‌دارد.

با انتخاب یک ردیف، ابتدا جدیدترین صفحه رونوشت خوانده می‌شود. **Load older transcript
items** یک نشانگر بایت مات را دنبال می‌کند و به‌جای بارگذاری کل سابقه، بخش محدود دیگری از
فایل JSONL را می‌خواند. محتوای عادی کاربر، دستیار،
استدلال، فراخوانی ابزار و نتیجه ابزار حفظ می‌شود. هر موردی که
از سقف ایمنی Node/Gateway بزرگ‌تر باشد، به‌وضوح به‌عنوان ناقص علامت‌گذاری می‌شود.

برای یک ردیف محلی Gateway با `claude-cli`، تایپ‌کردن در کادر نوشتن معمول، تابع
`sessions.catalog.continue` را فراخوانی می‌کند. OpenClaw رکورد کاتالوگ محلی را دوباره رفع می‌کند،
یک نشست بومی قفل‌شده به مدل ایجاد می‌کند یا دوباره به‌کار می‌گیرد، حداکثر 200 مورد قابل‌مشاهده
یا 512 KiB را وارد می‌کند و اتصال Claude CLI را مقداردهی اولیه می‌کند. نوبت نخست با
`--fork-session` ادامه می‌یابد؛ Claude یک شناسه نشست جدید به انشعاب اختصاص می‌دهد، بنابراین نوبت‌های بعدی از
انشعاب استفاده می‌کنند و نشست مبدأ دست‌نخورده باقی می‌ماند.

یک میزبان Node بدون رابط نیز می‌تواند با فعال‌کردن
تنظیم محلی Node در زیر و راه‌اندازی مجدد میزبان Node، امکان ادامه نشست‌های Claude CLI خود را فراهم کند:

```json5
{
  nodeHost: {
    agentRuns: {
      claude: { enabled: true },
    },
  },
}
```

Node تنها زمانی `agent.cli.claude.run.v1` را اعلام می‌کند که این تنظیم فعال باشد
و فایل اجرایی محلی `claude` آن رفع شود. OpenClaw رکورد کاتالوگ را
روی آن Node دوباره رفع می‌کند، همان سابقه محدود را وارد می‌کند و نشست پذیرفته‌شده را
به Node و پوشه کاری گزارش‌شده توسط کاتالوگ متصل می‌کند. هر نوبت، فرایند واقعی
`claude -p` آن Node را با استفاده از فایل‌ها و ورود Claude همان Node اجرا می‌کند.
سیاست تأیید اجرای Node همچنان اعمال می‌شود؛ Gateway نمی‌تواند این انتخاب را تحمیل کند.

ادامه از Node در نسخه 1 فقط یک‌بار اجرا می‌شود. پیکربندی MCP بازگشتی Gateway و
آرگومان‌های Plugin مهارت‌های Gateway را حذف می‌کند، از رونوشت Gateway دوباره مقداردهی اولیه نمی‌شود و
پیوست‌ها و تصاویر را رد می‌کند. ردیف‌های Claude Desktop فقط قابل‌مشاهده باقی می‌مانند. Nodeهای
برنامه بومی macOS نیز تا زمانی که برنامه فرمان اجرا را اعلام کند، فقط قابل‌مشاهده باقی می‌مانند.

<Note>
نشست‌های Claude در Node جفت‌شده فقط‌خواندنی باقی می‌مانند، مگر اینکه Node بدون رابط صریحاً
`agent.cli.claude.run.v1` را اعلام کند. OpenClaw هرگز فراداده Claude Desktop را
تغییر نمی‌دهد و نشست‌های Claude را بایگانی نمی‌کند. این صفحه به اتصال اپراتوری
با دامنه نوشتن نیاز دارد، زیرا از `node.invoke` احرازشده استفاده می‌کند؛ فهرست‌کردن و خواندن
حتی در Node دارای قابلیت ادامه نیز فقط‌خواندنی باقی می‌مانند.
</Note>

برای دستور Node و مرز امنیتی، به [Nodeها: نشست‌ها و رونوشت‌های Claude](/fa/nodes#claude-sessions-and-transcripts)
مراجعه کنید.

## پیش‌فرض‌های تفکر (Claude Opus 5، Sonnet 5، Mythos 5، Fable 5، 4.8 و 4.6)

`anthropic/claude-opus-5` به‌طور پیش‌فرض از تفکر تطبیقی با میزان تلاش `high` استفاده می‌کند.
برای غیرفعال‌کردن تفکر از `/think off` و برای استفاده از سطوح بالاتر تلاش بومی مدل
از `/think xhigh|max` استفاده کنید. OpenClaw بودجه‌های دستی تفکر، پارامترهای سفارشی
نمونه‌برداری، پیش‌نویس‌های دستیار و Priority Tier را برای Opus 5 حذف می‌کند، زیرا
Anthropic از این قابلیت‌های درخواست در این مدل پشتیبانی نمی‌کند. کاتالوگ،
پنجره زمینه 1,000,000 توکنی، محدودیت خروجی 128,000 توکنی، ورودی تصویر
و قیمت‌گذاری ورودی/خروجی `$5/$25` آن را منتشر می‌کند.

`anthropic/claude-sonnet-5` از همان پیش‌فرض‌های تفکر تطبیقی و محدودیت‌های
درخواست استفاده می‌کند. کاتالوگ تا 31 اوت 2026 از قیمت‌گذاری مقدماتی ورودی/خروجی
`$2/$10` متعلق به Anthropic استفاده می‌کند؛ قیمت‌گذاری استاندارد `$3/$15`
از 1 سپتامبر 2026 آغاز می‌شود.

`anthropic/claude-fable-5` همیشه از تفکر تطبیقی استفاده می‌کند و میزان تلاش پیش‌فرض آن
`high` است. Anthropic اجازه غیرفعال‌کردن تفکر را برای این مدل نمی‌دهد؛ بنابراین
`/think off` و `/think minimal` در عوض به میزان تلاش `low` نگاشت می‌شوند. OpenClaw همچنین
مقادیر سفارشی دما را از درخواست‌های Fable 5 حذف می‌کند، زیرا Anthropic
هرگونه بازنویسی دما را در درخواست‌های دارای تفکر رد می‌کند.

`anthropic/claude-mythos-5` مدلی با دسترسی محدود و همان قرارداد
تفکر تطبیقیِ همیشه‌فعال است. مقدار پیش‌فرض OpenClaw برابر `high` است، `/think off` و
`/think minimal` را به `low` نگاشت می‌کند و پارامترهای نمونه‌برداری انتخاب‌شده توسط فراخواننده را حذف می‌کند.
کاتالوگ، پنجره زمینه 1,000,000 توکنی، محدودیت خروجی 128,000 توکنی،
ورودی تصویر و قیمت‌گذاری ورودی/خروجی `$10/$50` آن را منتشر می‌کند.

در OpenClaw، تفکر Claude Opus 4.8 به‌طور پیش‌فرض غیرفعال می‌ماند. هنگامی که
تفکر تطبیقی را صراحتاً با `/think high|xhigh|max` فعال کنید، OpenClaw
مقادیر تلاش Opus 4.8 متعلق به Anthropic را ارسال می‌کند؛ مقدار پیش‌فرض مدل‌های Claude 4.6
(Opus 4.6 و Sonnet 4.6) برابر `adaptive` است.

برای هر پیام با `/think:<level>` یا در پارامترهای مدل بازنویسی کنید:

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-5": {
          params: { thinking: "high" },
        },
      },
    },
  },
}
```

<Note>
مستندات مرتبط Anthropic:
- [تفکر تطبیقی](https://platform.claude.com/docs/en/build-with-claude/adaptive-thinking)
- [تفکر توسعه‌یافته](https://platform.claude.com/docs/en/build-with-claude/extended-thinking)

</Note>

## سازوکار جایگزین هنگام امتناع ایمنی (Claude Fable 5)

<Warning>
استفاده از Claude Fable 5 به‌معنای استفاده هم‌زمان از Claude Opus 4.8 نیز هست. Fable 5 با
طبقه‌بندهای ایمنی عرضه می‌شود که ممکن است درخواستی را نپذیرند و راهکار بازیابی تأییدشده
Anthropic این است که `claude-opus-4-8` آن نوبت را پاسخ دهد. OpenClaw برای درخواست‌های مستقیم
با کلید API، به‌طور خودکار این قابلیت را فعال می‌کند؛ بنابراین برخی نوبت‌های Fable
توسط Claude Opus 4.8 پاسخ داده و صورتحساب می‌شوند. اگر خط‌مشی یا بودجه شما
نوبت‌های پاسخ‌داده‌شده با Opus را نمی‌پذیرد، `anthropic/claude-fable-5` را انتخاب نکنید.
</Warning>

### دلیل وجود این قابلیت

طبقه‌بندهای Fable 5 در درخواست‌های مربوط به حوزه‌های محدودشده، `stop_reason: "refusal"`
برمی‌گردانند و در کارهای بی‌ضرر اما نزدیک به این حوزه‌ها نیز مثبت کاذب دارند
(ابزارهای امنیتی، علوم زیستی یا حتی درخواست از مدل برای بازتولید استدلال خام خود).
بدون سازوکار جایگزین، نوبت با خطا پایان می‌یابد، هرچند مدل دیگری از Claude
به‌راحتی می‌تواند به آن پاسخ دهد؛ پیام امتناع خود Anthropic به یکپارچه‌سازهای API
اعلام می‌کند که یک مدل جایگزین پیکربندی کنند.

### نحوه عملکرد

1. OpenClaw برای هر درخواست مستقیم با کلید API به `anthropic/claude-fable-5`،
   اعلام پذیرش سازوکار جایگزین سمت سرور Anthropic را ارسال می‌کند:
   سرآیند بتای `server-side-fallback-2026-06-01` به‌همراه
   `fallbacks: [{"model": "claude-opus-4-8"}]`. Claude Opus 4.8 تنها
   مقصد جایگزینی است که Anthropic برای Fable 5 مجاز می‌داند.
2. فقط امتناع طبقه‌بند ایمنی سازوکار جایگزین را فعال می‌کند. محدودیت نرخ،
   اضافه‌بار و خطاهای سرور دقیقاً مانند قبل رفتار می‌کنند و از
   [جابجایی خودکار مدل](/fa/concepts/model-failover) معمول OpenClaw عبور می‌کنند.
3. بازیابی درون همان فراخوانی انجام می‌شود. امتناع پیش از هر خروجی،
   جز از نظر تأخیر نامرئی است؛ کل پاسخ از Opus 4.8 می‌آید. هنگام امتناع
   در میانه جریان، متن جزئی به‌عنوان پیشوندی که مدل جایگزین از آن ادامه می‌دهد
   حفظ می‌شود، اما استدلال و فراخوانی‌های ابزار مدل امتناع‌کننده طبق قواعد بازپخش Anthropic
   کنار گذاشته می‌شوند (نباید دوباره بازگردانده یا اجرا شوند).
4. اگر Claude Opus 4.8 نیز امتناع کند، نوبت دقیقاً مانند پیش از این قابلیت،
   امتناع را به‌شکل خطا نمایش می‌دهد.

سازوکار جایگزین در سطح API متعلق به Anthropic رخ می‌دهد؛ بنابراین لازم نیست
`claude-opus-4-8` در فهرست مدل‌های پیکربندی‌شده یا زنجیره جایگزینی شما باشد؛
کلید API دارای قابلیت Fable همیشه می‌تواند Opus را ارائه کند.

### مشاهده‌پذیری و صورتحساب

- نوبتی که توسط مدل جایگزین پاسخ داده می‌شود، یک عیب‌یابی `provider_fallback` را
  روی پیام دستیار ثبت می‌کند که `fromModel` و `toModel` را نام می‌برد و
  `responseModel` پیام، `claude-opus-4-8` را گزارش می‌کند.
- Anthropic برای هر تلاش صورتحساب صادر می‌کند: امتناع پیش از خروجی رایگان است و بازیابی
  با نرخ‌های Claude Opus 4.8 (در حال حاضر نصف نرخ‌های Fable 5) محاسبه می‌شود. برآورد
  هزینه هر نوبت در OpenClaw، نوبت‌های پاسخ‌داده‌شده توسط مدل جایگزین را برای تطابق با نرخ Opus قیمت‌گذاری می‌کند.
- امتناع در میانه جریان علاوه‌براین، بخش جزئی Fable را که پیش‌تر پخش شده است
  در سمت Anthropic مشمول هزینه می‌کند؛ این بخش در میزان مصرف هر تلاش API گزارش می‌شود،
  اما در برآورد هزینه هر نوبت OpenClaw ادغام نمی‌شود.

### دامنه

برای `anthropic/claude-fable-5` با احراز هویت کلید API در برابر
`api.anthropic.com` اعمال می‌شود. درخواست‌های OAuth (استفاده مجدد از اشتراک Claude CLI)،
نشانی‌های پایه پروکسی، Bedrock، Vertex و Foundry بدون تغییر هستند و همچنان
امتناع‌ها را در آن محیط‌ها به‌شکل خطا نمایش می‌دهند.

تأیید زنده: درخواست بی‌ضرری که از Fable 5 می‌خواهد زنجیره تفکر خام خود را
بازتولید کند، هنگام ارسال بدون مدل جایگزین با `category: "reasoning_extraction"`
رد می‌شود؛ همان درخواست از طریق OpenClaw یک پاسخ عادی ارائه‌شده توسط Opus را
با عیب‌یابی پیوست‌شده `provider_fallback` برمی‌گرداند.

برای رفتار زیربنایی، به [راهنمای امتناع‌ها و سازوکار جایگزین
Anthropic](https://platform.claude.com/docs/en/build-with-claude/refusals-and-fallback)
مراجعه کنید.

## ذخیره‌سازی موقت پرامپت

OpenClaw از قابلیت ذخیره‌سازی موقت پرامپت Anthropic برای احراز هویت کلید API پشتیبانی می‌کند.

| مقدار               | مدت نگهداری حافظه نهان | توضیحات                            |
| ------------------- | -------------- | -------------------------------------- |
| `"short"` (پیش‌فرض) | 5 دقیقه      | برای احراز هویت کلید API به‌طور خودکار اعمال می‌شود |
| `"long"`            | 1 ساعت         | حافظه نهان توسعه‌یافته                         |
| `"none"`            | بدون ذخیره‌سازی موقت     | ذخیره‌سازی موقت پرامپت را غیرفعال می‌کند                 |

```json5
{
  agents: {
    defaults: {
      models: {
        "anthropic/claude-opus-4-6": {
          params: { cacheRetention: "long" },
        },
      },
    },
  },
}
```

<AccordionGroup>
  <Accordion title="بازنویسی حافظه نهان برای هر عامل">
    از پارامترهای سطح مدل به‌عنوان مبنا استفاده کنید، سپس عامل‌های مشخص را از طریق `agents.entries.*.params` بازنویسی کنید:

    ```json5
    {
      agents: {
        defaults: {
          model: { primary: "anthropic/claude-opus-4-6" },
          models: {
            "anthropic/claude-opus-4-6": {
              params: { cacheRetention: "long" },
            },
          },
        },
        list: [
          { id: "research", default: true },
          { id: "alerts", params: { cacheRetention: "none" } },
        ],
      },
    }
    ```

    ترتیب ادغام پیکربندی:

    1. `agents.defaults.models["provider/model"].params`
    2. `agents.entries.*.params` (`id` منطبق، بازنویسی بر اساس کلید)

    این امکان به یک عامل اجازه می‌دهد حافظه نهان بلندمدت را حفظ کند، درحالی‌که عامل دیگری روی همان مدل ذخیره‌سازی موقت را برای ترافیک انفجاری یا کم‌استفاده غیرفعال می‌کند.

  </Accordion>

  <Accordion title="نکات Claude روی Bedrock">
    - مدل‌های Anthropic Claude روی Bedrock ‏(`amazon-bedrock/*anthropic.claude*`) در صورت پیکربندی، عبور مستقیم `cacheRetention` را می‌پذیرند.
    - مدل‌های غیر Anthropic در Bedrock هنگام اجرا به `cacheRetention: "none"` اجبار می‌شوند.
    - پیش‌فرض‌های هوشمند کلید API نیز هنگامی که مقدار صریحی تنظیم نشده باشد، `cacheRetention: "short"` را برای ارجاع‌های Claude روی Bedrock مقداردهی اولیه می‌کنند.

  </Accordion>
</AccordionGroup>

## پیکربندی پیشرفته

<AccordionGroup>
  <Accordion title="حالت سریع">
    کلید مشترک `/fast` در OpenClaw، فیلد `service_tier` متعلق به Anthropic را برای ترافیک مستقیم کلید API به `api.anthropic.com` تنظیم می‌کند.

    | فرمان | نگاشت به |
    |---------|---------|
    | `/fast on` | `service_tier: "auto"` |
    | `/fast off` | `service_tier: "standard_only"` |

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-sonnet-4-6": {
              params: { fastMode: true },
            },
          },
        },
      },
    }
    ```

    <Note>
    - فقط برای درخواست‌های مستقیم `api.anthropic.com` که با کلید API ایجاد شده‌اند اعمال می‌شود. درخواست‌های OAuth/توکن اشتراک و مسیرهای پروکسی هرگز فیلد `service_tier` دریافت نمی‌کنند.
    - هنگامی که هر دو تنظیم شده باشند، پارامتر صریح `serviceTier` یا `service_tier`، مقدار `/fast` را بازنویسی می‌کند.
    - Claude Opus 5 و Sonnet 5 از Priority Tier پشتیبانی نمی‌کنند؛ بنابراین OpenClaw برای این مدل‌ها `service_tier` را حذف می‌کند.
    - در حساب‌های فاقد ظرفیت Priority Tier، ‏`service_tier: "auto"` ممکن است به `standard` تبدیل شود.

    </Note>

  </Accordion>

  <Accordion title="درک رسانه (تصویر و PDF)">
    Plugin همراه Anthropic، درک تصویر و PDF را ثبت می‌کند. OpenClaw
    قابلیت‌های رسانه را به‌طور خودکار از احراز هویت پیکربندی‌شده Anthropic
    تعیین می‌کند؛ به پیکربندی اضافی نیازی نیست.

    | ویژگی        | مقدار                 |
    | --------------- | --------------------- |
    | مدل پیش‌فرض   | `claude-opus-5`       |
    | ورودی پشتیبانی‌شده | تصاویر، اسناد PDF |

    هنگامی که تصویر یا PDF به مکالمه‌ای پیوست شود، OpenClaw آن را
    به‌طور خودکار از طریق ارائه‌دهنده درک رسانه Anthropic مسیریابی می‌کند.

  </Accordion>

  <Accordion title="پنجره زمینه 1M">
    Claude Opus 5، Sonnet 5، Mythos 5 و Fable 5 دارای پنجره ورودی دقیق
    1,000,000 توکنی هستند و از حداکثر 128,000 توکن خروجی پشتیبانی می‌کنند.
    پنجره زمینه 1M متعلق به Anthropic در مدل‌های Claude 4.x با تفکر تطبیقی نیز
    به‌صورت عمومی در دسترس است: Opus 4.8،
    Opus 4.7، Opus 4.6 و Sonnet 4.6. OpenClaw اندازه این مدل‌ها را
    به‌طور خودکار تعیین می‌کند و به `params.context1m` نیازی نیست:

    ```json5
    {
      agents: {
        defaults: {
          models: {
            "anthropic/claude-opus-5": {},
            "anthropic/claude-sonnet-5": {},
            "anthropic/claude-mythos-5": {},
            "anthropic/claude-opus-4-6": {},
          },
        },
      },
    }
    ```

    پیکربندی‌های قدیمی‌تر می‌توانند `params.context1m: true` را نگه دارند؛ این گزینه برای
    این مدل‌ها بی‌اثر و بی‌ضرر است و OpenClaw دیگر در هیچ شرایطی
    سرآیند بتای منسوخ‌شده `context-1m-2025-08-07` را ارسال نمی‌کند. ورودی‌های قدیمی‌تر
    پیکربندی `anthropicBeta` با آن مقدار هنگام تعیین سرآیندهای درخواست حذف می‌شوند
    و مدل‌های قدیمی‌تر و پشتیبانی‌نشده Claude در پنجره زمینه عادی خود باقی می‌مانند.

    `params.context1m: true` برای پشتیبان Claude CLI
    ‏(`claude-cli/*`) نیز به همین شکل عمل می‌کند: مدل‌های واجد شرایط Opus و Sonnet
    که قابلیت دسترسی عمومی را دارند، از قبل پنجره 1M را به‌طور خودکار دریافت می‌کنند؛
    بنابراین این پارامتر در آنجا نیز اختیاری است.

    <Warning>
    به دسترسی زمینه طولانی در اعتبارنامه Anthropic شما نیاز دارد. احراز هویت با OAuth/توکن اشتراک، سرآیندهای بتای الزامی Anthropic را حفظ می‌کند، اما اگر سرآیند بتای منسوخ‌شده 1M در پیکربندی قدیمی‌تر باقی مانده باشد، OpenClaw آن را حذف می‌کند.
    </Warning>

  </Accordion>

  <Accordion title="زمینه 1M در Claude Opus 5">
    `anthropic/claude-opus-5` و گونه `claude-cli` آن به‌طور پیش‌فرض پنجره زمینه 1M
    دارند و به `params.context1m: true` نیازی نیست.
  </Accordion>
</AccordionGroup>

## عیب‌یابی

<AccordionGroup>
  <Accordion title="خطاهای 401 / نامعتبرشدن ناگهانی توکن">
    احراز هویت توکنی Anthropic منقضی می‌شود و ممکن است لغو شود. برای راه‌اندازی‌های جدید، به‌جای آن از کلید API متعلق به Anthropic استفاده کنید.
  </Accordion>

  <Accordion title='هیچ کلید API برای ارائه‌دهندهٔ "anthropic" یافت نشد'>
    احراز هویت Anthropic **برای هر عامل جداگانه** است؛ عامل‌های جدید کلیدهای عامل اصلی را به ارث نمی‌برند. فرایند راه‌اندازی اولیه را برای آن عامل دوباره اجرا کنید (یا یک کلید API را روی میزبان Gateway پیکربندی کنید)، سپس با `openclaw models status` بررسی کنید.
  </Accordion>

  <Accordion title='هیچ اعتبارنامه‌ای برای نمایهٔ "anthropic:default" یافت نشد'>
    برای مشاهدهٔ نمایهٔ احراز هویت فعال، `openclaw models status` را اجرا کنید. فرایند راه‌اندازی اولیه را دوباره اجرا کنید یا یک کلید API برای مسیر آن نمایه پیکربندی کنید.
  </Accordion>

  <Accordion title="هیچ نمایهٔ احراز هویتی در دسترس نیست (همه در دورهٔ انتظار هستند)">
    در `openclaw models status --json`، `auth.unusableProfiles` را بررسی کنید. دوره‌های انتظار ناشی از محدودیت نرخ Anthropic ممکن است مختص مدل باشند، بنابراین ممکن است همچنان بتوان از یک مدل هم‌خانوادهٔ Anthropic استفاده کرد. نمایهٔ Anthropic دیگری اضافه کنید یا تا پایان دورهٔ انتظار صبر کنید.
  </Accordion>
</AccordionGroup>

<Note>
راهنمای بیشتر: [عیب‌یابی](/fa/help/troubleshooting) و [پرسش‌های متداول](/fa/help/faq).
</Note>

## مرتبط

<CardGroup cols={2}>
  <Card title="انتخاب مدل" href="/fa/concepts/model-providers" icon="layers">
    انتخاب ارائه‌دهندگان، ارجاع‌های مدل و رفتار جایگزینی هنگام خرابی.
  </Card>
  <Card title="بک‌اندهای CLI" href="/fa/gateway/cli-backends" icon="terminal">
    جزئیات راه‌اندازی و زمان اجرای بک‌اند Claude CLI.
  </Card>
  <Card title="کش‌کردن پرامپت" href="/fa/reference/prompt-caching" icon="database">
    نحوهٔ عملکرد کش‌کردن پرامپت در میان ارائه‌دهندگان.
  </Card>
  <Card title="OAuth و احراز هویت" href="/fa/gateway/authentication" icon="key">
    جزئیات احراز هویت و قواعد استفادهٔ مجدد از اعتبارنامه‌ها.
  </Card>
</CardGroup>
