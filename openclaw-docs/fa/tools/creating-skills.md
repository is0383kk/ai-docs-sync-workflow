---
read_when:
    - شما در حال ایجاد یک مهارت سفارشی جدید هستید
    - به یک گردش‌کار شروع سریع برای Skills مبتنی بر SKILL.md نیاز دارید
    - می‌خواهید از کارگاه Skills برای پیشنهاد یک مهارت جهت بازبینی عامل استفاده کنید
sidebarTitle: Creating skills
summary: Skills سفارشی فضای کاری در قالب SKILL.md را برای عامل‌های OpenClaw خود بسازید، آزمایش کنید و منتشر کنید.
title: ایجاد Skills
x-i18n:
    generated_at: "2026-07-27T16:01:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: cba2aa863ebd083d4592e8a764dbdc2c30a0dd8aff49d273927e82df0069bc81
    source_path: tools/creating-skills.md
    workflow: 16
---

Skills به عامل می‌آموزند چگونه و چه زمانی از ابزارها استفاده کند. هر Skill یک دایرکتوری است
که شامل یک فایل `SKILL.md` با فرانت‌متر YAML و دستورالعمل‌های Markdown است.
OpenClaw مهارت‌ها را با [ترتیب تقدم](/fa/tools/skills#loading-order) مشخصی از چندین ریشه بارگذاری می‌کند.

## نخستین Skill خود را بسازید

<Steps>
  <Step title="دایرکتوری Skill را ایجاد کنید">
    Skills در پوشه `skills/` فضای کاری شما قرار می‌گیرند:

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/hello-world
    ```

    برای سازمان‌دهی می‌توانید Skills را در زیرپوشه‌ها گروه‌بندی کنید — نام Skill همچنان
    با فرانت‌متر `SKILL.md` تعیین می‌شود، نه مسیر پوشه:

    ```bash
    mkdir -p ~/.openclaw/workspace/skills/personal/hello-world
    # نام Skill همچنان "hello-world" است و با /hello-world فراخوانی می‌شود
    ```

  </Step>

  <Step title="SKILL.md را بنویسید">
    فرانت‌متر فراداده را تعریف می‌کند؛ بدنه دستورالعمل‌ها را به عامل ارائه می‌دهد.

    ```markdown
    ---
    name: hello-world
    description: یک Skill ساده که پیام خوشامدگویی چاپ می‌کند.
    ---

    # سلام دنیا

    وقتی کاربر پیام خوشامدگویی می‌خواهد، از ابزار `exec` برای اجرای دستور زیر استفاده کنید:

    ```bash
    echo "سلام از Skill سفارشی شما!"
    ```
    ```

    قواعد نام‌گذاری:
    - برای `name` از حروف کوچک، ارقام و خط تیره استفاده کنید.
    - نام دایرکتوری و `name` فرانت‌متر را هم‌خوان نگه دارید.
    - `description` به عامل و در بخش کشف فرمان‌های اسلش نمایش داده می‌شود —
      آن را تک‌خطی و کوتاه‌تر از 160 نویسه نگه دارید.

  </Step>

  <Step title="بارگذاری Skill را تأیید کنید">
    ```bash
    openclaw skills list
    ```

    OpenClaw به‌طور پیش‌فرض فایل‌های `SKILL.md` زیر ریشه‌های Skills را پایش می‌کند. اگر
    پایشگر غیرفعال است یا در حال ادامه‌دادن یک نشست موجود هستید، نشست جدیدی
    آغاز کنید تا عامل فهرست به‌روزشده را دریافت کند:

    ```bash
    # از چت — نشست فعلی را بایگانی کنید و نشستی تازه آغاز کنید
    /new

    # یا Gateway را دوباره راه‌اندازی کنید
    openclaw gateway restart
    ```

  </Step>

  <Step title="آن را آزمایش کنید">
    ```bash
    openclaw agent --message "یک پیام خوشامدگویی به من بده"
    ```

    یا یک چت باز کنید و مستقیماً از عامل بخواهید. برای
    فراخوانی صریح آن با نام، از `/skill hello-world` استفاده کنید.

  </Step>
</Steps>

## مرجع SKILL.md

### فیلدهای الزامی

| فیلد         | توضیحات                                                     |
| ------------- | --------------------------------------------------------------- |
| `name`        | نامک یکتا با حروف کوچک، ارقام و خط تیره        |
| `description` | توضیحی تک‌خطی که به عامل و در خروجی کشف نمایش داده می‌شود |

### کلیدهای اختیاری فرانت‌متر

| فیلد                      | پیش‌فرض | توضیحات                                                                      |
| -------------------------- | ------- | -------------------------------------------------------------------------------- |
| `user-invocable`           | `true`  | Skill را به‌عنوان فرمان اسلش کاربر در دسترس قرار می‌دهد                                         |
| `disable-model-invocation` | `false` | Skill را از پرامپت سیستمی عامل خارج نگه می‌دارد (همچنان از طریق `/skill` اجرا می‌شود)        |
| `command-dispatch`         | —       | برای هدایت مستقیم فرمان اسلش به ابزار و دورزدن مدل، روی `tool` تنظیم کنید |
| `command-tool`             | —       | نام ابزاری که هنگام تنظیم‌بودن `command-dispatch: tool` فراخوانی می‌شود                         |
| `command-arg-mode`         | `raw`   | برای ارسال به ابزار، رشته آرگومان‌های خام را به ابزار می‌فرستد                      |
| `homepage`                 | —       | نشانی اینترنتی که با عنوان «وب‌سایت» در رابط کاربری Skills در macOS نمایش داده می‌شود                                    |

برای فیلدهای دروازه‌بندی (`requires.bins`، `requires.env` و غیره)،
[Skills — دروازه‌بندی](/fa/tools/skills#gating) را ببینید.

### استفاده از `{baseDir}`

بدون کدنویسی ثابت مسیرها، به فایل‌های داخل دایرکتوری Skill ارجاع دهید —
عامل `{baseDir}` را نسبت به دایرکتوری خود Skill تفکیک می‌کند:

```markdown
اسکریپت کمکی را در `{baseDir}/scripts/run.sh` اجرا کنید.
```

## افزودن فعال‌سازی شرطی

Skill خود را دروازه‌بندی کنید تا فقط زمانی بارگذاری شود که وابستگی‌هایش در دسترس باشند:

```markdown
---
name: gemini-search
description: جست‌وجو با استفاده از Gemini CLI.
metadata: { "openclaw": { "requires": { "bins": ["gemini"] }, "primaryEnv": "GEMINI_API_KEY" } }
---
```

<AccordionGroup>
  <Accordion title="گزینه‌های دروازه‌بندی">
    | کلید | توضیحات |
    | --- | --- |
    | `requires.bins` | همه فایل‌های اجرایی باید در `PATH` موجود باشند |
    | `requires.anyBins` | دست‌کم یک فایل اجرایی باید در `PATH` موجود باشد |
    | `requires.env` | هر متغیر محیطی باید در فرایند یا پیکربندی موجود باشد |
    | `requires.config` | مقدار هر مسیر `openclaw.json` باید درست‌نما باشد |
    | `os` | فیلتر پلتفرم: `["darwin"]`، `["linux"]`، `["win32"]` |
    | `always` | برای نادیده‌گرفتن همه دروازه‌ها و گنجاندن همیشگی Skill، `true` را تنظیم کنید |

    مرجع کامل: [Skills — دروازه‌بندی](/fa/tools/skills#gating).

  </Accordion>
  <Accordion title="محیط و کلیدهای API">
    یک کلید API را به ورودی Skill در `openclaw.json` متصل کنید:

    ```json5
    {
      skills: {
        entries: {
          "gemini-search": {
            enabled: true,
            apiKey: { source: "env", provider: "default", id: "GEMINI_API_KEY" },
          },
        },
      },
    }
    ```

    کلید فقط برای همان نوبت عامل به فرایند میزبان تزریق می‌شود.
    این کلید به سندباکس نمی‌رسد — به
    [متغیرهای محیطی سندباکس‌شده](/fa/tools/skills-config#sandboxed-skills-and-env-vars) مراجعه کنید.

  </Accordion>
</AccordionGroup>

## پیشنهاد از طریق کارگاه Skill

برای Skills پیش‌نویس‌شده توسط عامل، یا زمانی که می‌خواهید پیش از عملیاتی‌شدن یک Skill،
اپراتور آن را بازبینی کند، به‌جای نوشتن مستقیم `SKILL.md` از پیشنهادهای
[کارگاه Skill](/fa/tools/skill-workshop) استفاده کنید.

```bash
# یک Skill کاملاً جدید پیشنهاد دهید
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "یک Skill ساده که پیام خوشامدگویی چاپ می‌کند." \
  --proposal ./PROPOSAL.md

# به‌روزرسانی یک Skill موجود را پیشنهاد دهید
openclaw skills workshop propose-update hello-world \
  --proposal ./PROPOSAL.md \
  --description "Skill خوشامدگویی به‌روزشده"
```

وقتی پیشنهاد شامل فایل‌های پشتیبانی است، از `--proposal-dir` استفاده کنید:

```bash
openclaw skills workshop propose-create \
  --name "hello-world" \
  --description "یک Skill ساده که پیام خوشامدگویی چاپ می‌کند." \
  --proposal-dir ./hello-world-proposal/
```

دایرکتوری باید در ریشه خود شامل `PROPOSAL.md` باشد. فایل‌های پشتیبانی در
`assets/`، `examples/`، `references/`، `scripts/` یا `templates/` قرار می‌گیرند.

پس از بازبینی:

```bash
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop apply <proposal-id>
```

برای چرخه کامل پیشنهاد، [کارگاه Skill](/fa/tools/skill-workshop) را ببینید.

## انتشار در ClawHub

<Steps>
  <Step title="از کامل‌بودن SKILL.md خود مطمئن شوید">
    مطمئن شوید `name`، `description` و همه فیلدهای دروازه‌بندی `metadata.openclaw`
    تنظیم شده‌اند. اگر صفحه پروژه دارید، یک نشانی `homepage` اضافه کنید.
  </Step>
  <Step title="CLI مستقل ClawHub را نصب و وارد حساب شوید">
    ```bash
    npm i -g clawhub
    clawhub login
    ```
  </Step>
  <Step title="منتشر کنید">
    ```bash
    clawhub skill publish ./path/to/hello-world
    ```

    برای بازنویسی نسخه استنباط‌شده یا انتشار تحت مالک مشخص،
    `--version <version>` یا `--owner <owner>` را اضافه کنید. برای روند کامل، محدوده‌بندی مالک
    و دیگر فرمان‌های نگه‌داری (`clawhub sync`، `clawhub skill rename` و غیره)،
    [ClawHub — انتشار](/fa/clawhub/publishing) و
    [CLI ‏ClawHub](/fa/clawhub/cli) را ببینید.

  </Step>
</Steps>

## روش‌های برتر

<Tip>
  - **مختصر باشید** — مدل را درباره *کاری* که باید انجام دهد راهنمایی کنید، نه درباره اینکه چگونه یک هوش مصنوعی باشد.
  - **اول ایمنی** — اگر Skill شما از `exec` استفاده می‌کند، مطمئن شوید پرامپت‌ها
    امکان تزریق فرمان دلخواه از ورودی نامطمئن را نمی‌دهند.
  - **به‌صورت محلی آزمایش کنید** — پیش از اشتراک‌گذاری از `openclaw agent --message "..."` استفاده کنید.
  - **از ClawHub استفاده کنید** — پیش از ساختن از ابتدا، Skills جامعه را در [clawhub.ai](https://clawhub.ai)
    مرور کنید.
</Tip>

## مرتبط

<CardGroup cols={2}>
  <Card title="مرجع Skills" href="/fa/tools/skills" icon="puzzle-piece">
    ترتیب بارگذاری، دروازه‌بندی، فهرست‌های مجاز و قالب SKILL.md.
  </Card>
  <Card title="کارگاه Skill" href="/fa/tools/skill-workshop" icon="flask">
    صف پیشنهاد برای Skills پیش‌نویس‌شده توسط عامل.
  </Card>
  <Card title="پیکربندی Skills" href="/fa/tools/skills-config" icon="gear">
    شِمای کامل پیکربندی `skills.*`.
  </Card>
  <Card title="ClawHub" href="/clawhub" icon="cloud">
    Skills را در رجیستری عمومی مرور و منتشر کنید.
  </Card>
  <Card title="ساخت Pluginها" href="/fa/plugins/building-plugins" icon="plug">
    Pluginها می‌توانند Skills را همراه ابزارهایی که مستند می‌کنند ارائه دهند.
  </Card>
</CardGroup>
