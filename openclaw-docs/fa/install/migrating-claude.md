---
read_when:
    - از Claude Code یا Claude Desktop می‌آیید و می‌خواهید دستورالعمل‌ها، سرورهای MCP و مهارت‌ها را حفظ کنید
    - باید بدانید OpenClaw چه چیزهایی را به‌طور خودکار وارد می‌کند و چه چیزهایی فقط در بایگانی باقی می‌مانند
summary: انتقال وضعیت محلی Claude Code و Claude Desktop به OpenClaw با واردسازی دارای پیش‌نمایش
title: مهاجرت از Claude
x-i18n:
    generated_at: "2026-07-27T15:21:31Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0d5a5e63727e1583fc3fa27ac45215c72df9074b21d7c5f6b33800bec916769b
    source_path: install/migrating-claude.md
    workflow: 16
---

OpenClaw وضعیت محلی Claude را از طریق ارائه‌دهندهٔ مهاجرت Claude که همراه برنامه ارائه می‌شود، وارد می‌کند. این ارائه‌دهنده پیش از تغییر وضعیت، پیش‌نمایش همهٔ موارد را نمایش می‌دهد و اسرار را در طرح‌ها و گزارش‌ها پنهان می‌کند. اجرای مستقل `openclaw migrate` یک نسخهٔ پشتیبان تأییدشده ایجاد می‌کند؛ مسیر راه‌اندازی اولیهٔ تازه، واردسازی را مرحله‌بندی می‌کند و فقط پس از موفقیت تأیید آن را منتشر می‌کند.

<Note>
واردسازی هنگام راه‌اندازی اولیه به یک راه‌اندازی تازهٔ OpenClaw نیاز دارد. اگر از قبل وضعیت محلی OpenClaw دارید، ابتدا پیکربندی، اطلاعات احراز هویت، نشست‌ها و فضای کاری را بازنشانی کنید، یا پس از بازبینی طرح، مستقیماً از `openclaw migrate` همراه با `--overwrite` استفاده کنید.
</Note>

## دو روش واردسازی

<Tabs>
  <Tab title="جادوگر راه‌اندازی اولیه">
    جادوگر وقتی وضعیت محلی Claude را شناسایی کند، گزینهٔ Claude را ارائه می‌دهد.

    ```bash
    openclaw onboard --flow import
    ```

    یا یک منبع مشخص را تعیین کنید:

    ```bash
    openclaw onboard --import-from claude --import-source ~/.claude
    ```

  </Tab>
  <Tab title="CLI">
    برای اجراهای اسکریپتی یا تکرارپذیر از `openclaw migrate` استفاده کنید. برای مرجع کامل، [`openclaw migrate`](/fa/cli/migrate) را ببینید.

    ```bash
    openclaw migrate claude --dry-run
    openclaw migrate apply claude --yes
    ```

    برای واردسازی پوشهٔ خانگی یا ریشهٔ پروژهٔ مشخصی از Claude Code، `--from <path>` را اضافه کنید.

  </Tab>
</Tabs>

## مواردی که وارد می‌شوند

<AccordionGroup>
  <Accordion title="دستورالعمل‌ها و حافظه">
    - محتوای `CLAUDE.md` و `.claude/CLAUDE.md` پروژه در `AGENTS.md` فضای کاری عامل OpenClaw کپی یا به آن افزوده می‌شود.
    - محتوای `~/.claude/CLAUDE.md` کاربر به `USER.md` فضای کاری افزوده می‌شود.

  </Accordion>
  <Accordion title="سرورهای MCP">
    تعریف‌های سرور MCP، در صورت وجود، از `.mcp.json` پروژه، `~/.claude.json` مربوط به Claude Code و `claude_desktop_config.json` مربوط به Claude Desktop وارد می‌شوند.
  </Accordion>
  <Accordion title="Skills و فرمان‌ها">
    - Skills متعلق به Claude که فایل `SKILL.md` دارند، در پوشهٔ Skills فضای کاری OpenClaw کپی می‌شوند.
    - فایل‌های Markdown فرمان Claude در `.claude/commands/` یا `~/.claude/commands/` با `disable-model-invocation: true` به Skills متعلق به OpenClaw تبدیل می‌شوند.

  </Accordion>
</AccordionGroup>

## مواردی که فقط در بایگانی باقی می‌مانند

ارائه‌دهنده این موارد را برای بازبینی دستی در گزارش مهاجرت کپی می‌کند، اما آن‌ها را در پیکربندی فعال OpenClaw بارگذاری **نمی‌کند**:

- قلاب‌های Claude
- مجوزهای Claude و فهرست‌های مجاز گستردهٔ ابزارها
- پیش‌فرض‌های محیطی Claude
- `CLAUDE.local.md`
- `.claude/rules/`
- زیرعامل‌های Claude در `.claude/agents/` یا `~/.claude/agents/`
- پوشه‌های حافظهٔ نهان، طرح‌ها و تاریخچهٔ پروژهٔ Claude Code
- افزونه‌های Claude Desktop و اطلاعات احراز هویت ذخیره‌شده در سیستم‌عامل

OpenClaw از اجرای خودکار قلاب‌ها، اعتماد به فهرست‌های مجاز مجوزها یا رمزگشایی وضعیت مبهم اطلاعات احراز هویت OAuth و Desktop خودداری می‌کند. پس از بازبینی بایگانی، موارد موردنیاز را به‌صورت دستی منتقل کنید.

## انتخاب منبع

بدون `--from`، OpenClaw پوشهٔ خانگی پیش‌فرض Claude Code در `~/.claude`، فایل وضعیت نمونه‌برداری‌شدهٔ `~/.claude.json` مربوط به Claude Code و پیکربندی MCP مربوط به Claude Desktop در macOS را بررسی می‌کند.

وقتی `--from` به ریشهٔ یک پروژه اشاره کند، OpenClaw فقط فایل‌های Claude همان پروژه، مانند `CLAUDE.md`، `.claude/settings.json`، `.claude/commands/`، `.claude/skills/` و `.mcp.json` را وارد می‌کند. هنگام واردسازی از ریشهٔ پروژه، پوشهٔ خانگی سراسری Claude خوانده نمی‌شود.

## روند پیشنهادی

<Steps>
  <Step title="پیش‌نمایش طرح">
    ```bash
    openclaw migrate claude --dry-run
    ```

    طرح همهٔ مواردی را که تغییر خواهند کرد فهرست می‌کند؛ از جمله تداخل‌ها، موارد ردشده و مقادیر حساسی که از فیلدهای تودرتوی `env` یا `headers` در MCP پنهان شده‌اند.

  </Step>
  <Step title="اعمال همراه با نسخهٔ پشتیبان">
    ```bash
    openclaw migrate apply claude --yes
    ```

    OpenClaw پیش از اعمال تغییرات، یک نسخهٔ پشتیبان ایجاد و تأیید می‌کند.

  </Step>
  <Step title="اجرای Doctor">
    ```bash
    openclaw doctor
    ```

    [Doctor](/fa/gateway/doctor) پس از واردسازی، مشکلات پیکربندی یا وضعیت را بررسی می‌کند.

  </Step>
  <Step title="راه‌اندازی مجدد و تأیید">
    ```bash
    openclaw gateway restart
    openclaw status
    ```

    تأیید کنید که Gateway سالم است و دستورالعمل‌ها، سرورهای MCP و Skills واردشده بارگذاری شده‌اند.

  </Step>
</Steps>

## مدیریت تداخل‌ها

وقتی طرح تداخل‌هایی را گزارش دهد (فایل یا مقدار پیکربندی از قبل در مقصد وجود داشته باشد)، عملیات اعمال از ادامه خودداری می‌کند.

<Warning>
فقط زمانی دوباره با `--overwrite` اجرا کنید که جایگزینی مقصد موجود عمدی باشد. ارائه‌دهندگان ممکن است همچنان برای فایل‌های بازنویسی‌شده، نسخه‌های پشتیبان جداگانهٔ هر مورد را در پوشهٔ گزارش مهاجرت بنویسند.
</Warning>

در نصب تازهٔ OpenClaw، تداخل‌ها غیرمعمول‌اند. این تداخل‌ها معمولاً زمانی ظاهر می‌شوند که واردسازی را در راه‌اندازی‌ای که از قبل ویرایش‌های کاربر را دارد، دوباره اجرا کنید.

## خروجی JSON برای خودکارسازی

```bash
openclaw migrate claude --dry-run --json
openclaw migrate apply claude --json --yes
```

برای `migrate apply` خارج از پایانهٔ تعاملی، `--yes` الزامی است؛ بدون آن، OpenClaw به‌جای اعمال تغییرات خطا می‌دهد، بنابراین اسکریپت‌ها و پایپ‌لاین CI باید `--yes` را صریحاً ارسال کنند. ابتدا با `--dry-run --json` پیش‌نمایش بگیرید، سپس وقتی طرح درست به نظر رسید، با `--json --yes` آن را اعمال کنید.

## عیب‌یابی

<AccordionGroup>
  <Accordion title="وضعیت Claude خارج از ~/.claude قرار دارد">
    `--from /actual/path` ‏(CLI) یا `--import-source /actual/path` ‏(راه‌اندازی اولیه) را ارسال کنید.
  </Accordion>
  <Accordion title="راه‌اندازی اولیه از واردسازی در یک راه‌اندازی موجود خودداری می‌کند">
    واردسازی هنگام راه‌اندازی اولیه به یک راه‌اندازی تازه نیاز دارد. وضعیت را بازنشانی و راه‌اندازی اولیه را دوباره انجام دهید، یا مستقیماً از `openclaw migrate apply claude` استفاده کنید که از `--overwrite` و کنترل صریح نسخهٔ پشتیبان پشتیبانی می‌کند.
  </Accordion>
  <Accordion title="سرورهای MCP از Claude Desktop وارد نشدند">
    Claude Desktop فایل `claude_desktop_config.json` را از مسیری مختص پلتفرم می‌خواند. اگر OpenClaw آن را به‌طور خودکار شناسایی نکرد، `--from` را به پوشهٔ آن فایل هدایت کنید.
  </Accordion>
  <Accordion title="فرمان‌های Claude با فراخوانی غیرفعال مدل به Skills تبدیل شدند">
    این رفتار عمدی است. فرمان‌های Claude به‌دست کاربر فعال می‌شوند، بنابراین OpenClaw آن‌ها را به‌صورت Skills با `disable-model-invocation: true` وارد می‌کند. اگر می‌خواهید عامل آن‌ها را به‌طور خودکار فراخوانی کند، فرانت‌متر هر Skill را ویرایش کنید.
  </Accordion>
</AccordionGroup>

## مرتبط

- [`openclaw migrate`](/fa/cli/migrate): مرجع کامل CLI، قرارداد Plugin و ساختارهای JSON.
- [راهنمای مهاجرت](/fa/install/migrating): همهٔ مسیرهای مهاجرت.
- [مهاجرت از Hermes](/fa/install/migrating-hermes): مسیر دیگر واردسازی بین‌سیستمی.
- [راه‌اندازی اولیه](/fa/cli/onboard): روند جادوگر و پرچم‌های غیرتعاملی.
- [Doctor](/fa/gateway/doctor): بررسی سلامت پس از مهاجرت.
- [فضای کاری عامل](/fa/concepts/agent-workspace): محل قرارگیری `AGENTS.md`، `USER.md` و Skills.
