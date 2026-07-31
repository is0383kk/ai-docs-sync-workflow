---
read_when:
    - نصب یا پیکربندی چارچوب acpx برای Claude Code / Codex / Gemini CLI
    - فعال‌سازی پل MCP ابزارهای Plugin یا ابزارهای OpenClaw
    - پیکربندی حالت‌های مجوز ACP
summary: 'راه‌اندازی عامل‌های ACP: پیکربندی هارنس acpx، راه‌اندازی Plugin و مجوزها'
title: عامل‌های ACP — راه‌اندازی
x-i18n:
    generated_at: "2026-07-27T17:10:48Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: ae3750092175b44252dd080717a1af176995df43c653f245f82d7e556cfd25eb
    source_path: tools/acp-agents-setup.md
    workflow: 16
---

برای نمای کلی، راهنمای عملیاتی اپراتور و مفاهیم، به [عامل‌های ACP](/fa/tools/acp-agents) مراجعه کنید.

این صفحه پیکربندی مهار acpx، راه‌اندازی Plugin برای پل‌های MCP و پیکربندی مجوزها را پوشش می‌دهد.

تنها زمانی از این صفحه استفاده کنید که مسیر ACP/acpx را راه‌اندازی می‌کنید. برای پیکربندی زمان اجرای بومی app-server در Codex، از [مهار Codex](/fa/plugins/codex-harness) استفاده کنید. برای کلیدهای API در OpenAI یا پیکربندی ارائه‌دهنده مدل OAuth در Codex، از [OpenAI](/fa/providers/openai) استفاده کنید.

Codex دو مسیر OpenClaw دارد:

| مسیر                       | پیکربندی/دستور                                         | صفحه راه‌اندازی                         |
| -------------------------- | ------------------------------------------------------ | --------------------------------------- |
| app-server بومی Codex      | ارجاع‌های عامل `/codex ...`، `openai/gpt-*`                | [مهار Codex](/fa/plugins/codex-harness) |
| سازگارکننده صریح Codex ACP | `/acp spawn codex`، `runtime: "acp", agentId: "codex"` | این صفحه                                |

مگر اینکه صراحتاً به رفتار ACP/acpx نیاز داشته باشید، مسیر بومی را ترجیح دهید.

## پشتیبانی مهار acpx (فعلی)

نام‌های مستعار داخلی مهار acpx (از وابستگی پین‌شده `acpx`):

| نام مستعار        | پوشش‌دهنده                                                                                                           |
| ------------ | --------------------------------------------------------------------------------------------------------------- |
| `claude`     | [Claude Code](https://claude.ai/code)                                                                           |
| `codex`      | [Codex CLI](https://codex.openai.com)                                                                           |
| `copilot`    | [GitHub Copilot CLI](https://docs.github.com/copilot/how-tos/copilot-chat/use-copilot-chat-in-the-command-line) |
| `cursor`     | [Cursor CLI](https://cursor.com/docs/cli/acp) (`cursor-agent acp`)                                              |
| `droid`      | [Factory Droid](https://www.factory.ai)                                                                         |
| `fast-agent` | [fast-agent](https://fast-agent.ai)                                                                             |
| `gemini`     | [Gemini CLI](https://github.com/google/gemini-cli)                                                              |
| `iflow`      | [iFlow CLI](https://github.com/iflow-ai/iflow-cli)                                                              |
| `kilocode`   | [Kilocode](https://kilocode.ai)                                                                                 |
| `kimi`       | [Kimi CLI](https://github.com/MoonshotAI/kimi-cli)                                                              |
| `kiro`       | [Kiro CLI](https://kiro.dev)                                                                                    |
| `mux`        | [Mux](https://mux.coder.com)                                                                                    |
| `opencode`   | [OpenCode](https://opencode.ai)                                                                                 |
| `openclaw`   | پل ACP در OpenClaw (`openclaw acp` بومی)                                                                     |
| `pi`         | [عامل کدنویسی Pi](https://github.com/mariozechner/pi)                                                           |
| `qoder`      | [Qoder CLI](https://docs.qoder.com/cli/acp)                                                                     |
| `qwen`       | [Qwen Code](https://github.com/QwenLM/qwen-code)                                                                |
| `trae`       | [Trae CLI](https://docs.trae.cn/cli)                                                                            |

`factory-droid` و `factorydroid` نیز به سازگارکننده داخلی `droid` تفکیک می‌شوند.

وقتی OpenClaw از بک‌اند acpx استفاده می‌کند، برای `agentId` این مقادیر را ترجیح دهید، مگر اینکه پیکربندی acpx شما نام‌های مستعار سفارشی برای عامل‌ها تعریف کرده باشد.
اگر نصب محلی Cursor شما همچنان ACP را به‌صورت `agent acp` ارائه می‌کند، به‌جای تغییر مقدار پیش‌فرض داخلی، دستور عامل `cursor` را در پیکربندی acpx خود بازنویسی کنید.

استفاده مستقیم از CLI در acpx می‌تواند سازگارکننده‌های دلخواه را نیز از طریق `--agent <command>` هدف بگیرد، اما این مسیر گریز خام یک قابلیت CLI در acpx است (نه مسیر معمول `agentId` در OpenClaw).

کنترل مدل به قابلیت‌های سازگارکننده وابسته است. ارجاع‌های مدل Codex ACP پیش از راه‌اندازی توسط OpenClaw عادی‌سازی می‌شوند. مهارهای دیگر به `models` در ACP به‌همراه پشتیبانی از `session/set_model` نیاز دارند؛ اگر مهاری نه آن قابلیت ACP و نه پرچم راه‌اندازی مدل خودش را ارائه کند، OpenClaw/acpx نمی‌تواند انتخاب مدل را تحمیل کند.

## پیکربندی الزامی

خط پایه اصلی ACP:

```json5
{
  acp: {
    enabled: true,
    // اختیاری. مقدار پیش‌فرض true است؛ برای توقف موقت توزیع ACP درحالی‌که کنترل‌های /acp حفظ می‌شوند، آن را روی false تنظیم کنید.
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "codex",
    allowedAgents: [
      "claude",
      "codex",
      "copilot",
      "cursor",
      "droid",
      "gemini",
      "iflow",
      "kilocode",
      "kimi",
      "kiro",
      "openclaw",
      "opencode",
      "qwen",
    ],
    stream: {
      deliveryMode: "live",
    },
  },
}
```

پیکربندی اتصال رشته گفتگو میان سازگارکننده‌های کانال پشتیبانی‌شده مشترک است:

```json5
{
  session: {
    threadBindings: {
      enabled: true,
      idleHours: 24,
      maxAgeHours: 0,
      spawnSessions: true,
    },
  },
}
```

اگر ایجاد ACP متصل به رشته گفتگو کار نمی‌کند، ابتدا پرچم قابلیت سازگارکننده را بررسی کنید:

- Discord: `session.threadBindings.spawnSessions=true`

اتصال‌های مکالمه فعلی به ایجاد رشته فرزند نیاز ندارند. آن‌ها به یک زمینه مکالمه فعال و سازگارکننده کانالی نیاز دارند که اتصال‌های مکالمه ACP را ارائه کند.

به [مرجع پیکربندی](/fa/gateway/configuration-reference) مراجعه کنید.

## راه‌اندازی Plugin برای بک‌اند acpx

نصب‌های بسته‌بندی‌شده از Plugin رسمی زمان اجرای `@openclaw/acpx` برای ACP استفاده می‌کنند.
پیش از استفاده از نشست‌های مهار ACP، آن را نصب و فعال کنید:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

پس از `pnpm install`، پرداخت‌های منبع نیز می‌توانند از Plugin فضای کاری محلی استفاده کنند.

با این مورد شروع کنید:

```text
/acp doctor
```

اگر `acpx` را غیرفعال کرده‌اید، آن را از طریق `plugins.allow` / `plugins.deny` رد کرده‌اید، یا می‌خواهید به Plugin بسته‌بندی‌شده بازگردید، از مسیر صریح بسته استفاده کنید:

```bash
openclaw plugins install @openclaw/acpx
openclaw config set plugins.entries.acpx.enabled true
```

نصب فضای کاری محلی هنگام توسعه:

```bash
openclaw plugins install ./path/to/local/acpx-plugin
```

سپس سلامت بک‌اند را بررسی کنید:

```text
/acp doctor
```

### کاوش راه‌اندازی زمان اجرای acpx

Plugin ‏`acpx` زمان اجرای ACP را مستقیماً تعبیه می‌کند (هیچ فایل اجرایی یا نسخه جداگانه‌ای از `acpx` برای پیکربندی وجود ندارد). به‌طور پیش‌فرض، بک‌اند تعبیه‌شده را هنگام راه‌اندازی Gateway ثبت می‌کند و پیش از سیگنال `ready` در Gateway منتظر کاوش راه‌اندازی می‌ماند. فقط برای اسکریپت‌ها یا محیط‌هایی که عمداً کاوش راه‌اندازی را غیرفعال نگه می‌دارند، `OPENCLAW_ACPX_RUNTIME_STARTUP_PROBE=0` یا `OPENCLAW_SKIP_ACPX_RUNTIME_PROBE=1` را تنظیم کنید. برای یک کاوش صریح برحسب تقاضا، `/acp doctor` را اجرا کنید.

هنگامی که یک مسیر یا مقدار پرچم باید به‌صورت یک توکن argv باقی بماند، دستور یک عامل ACP را با آرگومان‌های ساخت‌یافته بازنویسی کنید:

```json
{
  "plugins": {
    "entries": {
      "acpx": {
        "enabled": true,
        "config": {
          "agents": {
            "claude": {
              "command": "node",
              "args": ["/path/to/custom adapter.mjs", "--verbose"]
            }
          }
        }
      }
    }
  }
}
```

- `agents.<id>.command` فایل اجرایی یا رشته دستور موجود برای آن عامل ACP است.
- `agents.<id>.args` اختیاری است. پیش از اینکه OpenClaw هر مورد آرایه را از طریق رجیستری فعلی رشته دستور acpx عبور دهد، آن مورد برای پوسته نقل‌قول‌گذاری می‌شود.

به [Pluginها](/fa/tools/plugin) مراجعه کنید.

### بارگیری خودکار سازگارکننده

`acpx` سازگارکننده‌های ACP (برای مثال پل‌های ACP در Claude و Codex) را هنگام نخستین استفاده از طریق `npx` به‌طور خودکار بارگیری می‌کند. لازم نیست بسته‌های سازگارکننده را دستی نصب کنید و برای خود OpenClaw نیز مرحله postinstall جداگانه‌ای وجود ندارد. اگر بارگیری یا ایجاد سازگارکننده ناموفق باشد، `/acp doctor` خطا را گزارش می‌کند.

### پل MCP برای ابزارهای Plugin

نشست‌های ACPX به‌طور پیش‌فرض ابزارهای ثبت‌شده توسط Pluginهای OpenClaw را در اختیار مهار ACP قرار **نمی‌دهند**.

اگر می‌خواهید عامل‌های ACP مانند Codex یا Claude Code بتوانند ابزارهای Plugin نصب‌شده در OpenClaw مانند بازیابی/ذخیره حافظه را فراخوانی کنند، پل اختصاصی را فعال کنید:

```bash
openclaw config set plugins.entries.acpx.config.pluginToolsMcpBridge true
```

کارکرد آن:

- یک سرور داخلی MCP با نام `openclaw-plugin-tools` را به راه‌اندازی اولیه نشست ACPX تزریق می‌کند.
- ابزارهای Plugin را که از قبل توسط Pluginهای نصب‌شده و فعال OpenClaw ثبت شده‌اند، ارائه می‌کند.
- هویت نشست فعال ACP را به کارخانه‌های ابزار Plugin منتقل می‌کند تا ابزارهای مختص عامل در فضای نام همان عامل باقی بمانند.
- این قابلیت را صریح و به‌طور پیش‌فرض غیرفعال نگه می‌دارد.

نکات امنیتی و اعتماد:

- این کار سطح ابزار مهار ACP را گسترش می‌دهد.
- عامل‌های ACP فقط به ابزارهای Plugin که از قبل در Gateway فعال هستند دسترسی پیدا می‌کنند.
- این قابلیت را هم‌مرز با اعتمادی در نظر بگیرید که برای اجرای آن Pluginها در خود OpenClaw لازم است.
- پیش از فعال‌کردن آن، Pluginهای نصب‌شده را بازبینی کنید.

`mcpServers` سفارشی همچنان مانند گذشته کار می‌کنند. پل داخلی ابزارهای Plugin یک امکان اختیاری اضافی است، نه جایگزینی برای پیکربندی عمومی سرور MCP.

### پل MCP برای ابزارهای OpenClaw

نشست‌های ACPX به‌طور پیش‌فرض ابزارهای داخلی OpenClaw را نیز از طریق MCP ارائه **نمی‌کنند**. وقتی یک عامل ACP به ابزارهای داخلی منتخب مانند `cron` نیاز دارد، پل جداگانه ابزارهای اصلی را فعال کنید:

```bash
openclaw config set plugins.entries.acpx.config.openClawToolsMcpBridge true
```

کارکرد آن:

- یک سرور داخلی MCP با نام `openclaw-tools` را به راه‌اندازی اولیه نشست ACPX تزریق می‌کند.
- ابزارهای داخلی منتخب OpenClaw را ارائه می‌کند. سرور اولیه `cron` را ارائه می‌کند.
- ارائه ابزارهای اصلی را صریح و به‌طور پیش‌فرض غیرفعال نگه می‌دارد.

### پیکربندی مهلت عملیات زمان اجرا

Plugin ‏`acpx` به‌طور پیش‌فرض برای عملیات راه‌اندازی و کنترل زمان اجرای تعبیه‌شده 120 ثانیه مهلت در نظر می‌گیرد. این مهلت به مهارهای کندتر مانند Gemini CLI زمان کافی می‌دهد تا راه‌اندازی و مقداردهی اولیه ACP را کامل کنند. اگر میزبان شما به محدودیت عملیاتی متفاوتی نیاز دارد، آن را بازنویسی کنید:

```bash
openclaw config set plugins.entries.acpx.config.timeoutSeconds 180
```

نوبت‌های زمان اجرا از مهلت‌های عامل/اجرای OpenClaw، از جمله `/acp timeout`، استفاده می‌کنند.
`sessions_spawn` بازنویسی مهلت برای هر فراخوانی را نمی‌پذیرد؛ مسیر اپراتور `agents.defaults.subagents.runTimeoutSeconds` است. پس از تغییر `timeoutSeconds`، Gateway را مجدداً راه‌اندازی کنید.

### پیکربندی عامل کاوش سلامت

وقتی `/acp doctor` یا کاوش راه‌اندازی، بک‌اند را بررسی می‌کند، Plugin همراه `acpx` یک عامل مهار را می‌آزماید. اگر `acp.allowedAgents` تنظیم شده باشد، مقدار پیش‌فرض آن نخستین عامل مجاز است؛ در غیر این صورت مقدار پیش‌فرض آن `codex` است. اگر استقرار شما برای بررسی‌های سلامت به عامل ACP دیگری نیاز دارد، عامل کاوش را صریحاً تنظیم کنید:

```bash
openclaw config set plugins.entries.acpx.config.probeAgent claude
```

پس از تغییر این مقدار، Gateway را مجدداً راه‌اندازی کنید.

## پیکربندی مجوزها

نشست‌های ACP به‌صورت غیرتعاملی اجرا می‌شوند — هیچ TTY برای تأیید یا رد درخواست‌های مجوز نوشتن فایل و اجرای پوسته وجود ندارد. Plugin ‏acpx دو کلید پیکربندی ارائه می‌کند که نحوه مدیریت مجوزها را کنترل می‌کنند:

این مجوزهای هارنس ACPX از تأییدهای اجرای OpenClaw و پرچم‌های دور زدن ارائه‌دهنده در بک‌اند CLI، مانند Claude CLI `--permission-mode bypassPermissions`، جدا هستند. `approve-all` در ACPX کلید اضطراری سطح هارنس برای نشست‌های ACP است.

برای مقایسه گسترده‌تر میان `tools.exec.mode` در OpenClaw، تأییدهای Codex Guardian
و مجوزهای هارنس ACPX، به
[حالت‌های مجوز](/fa/tools/permission-modes) مراجعه کنید.

### `permissionMode`

کنترل می‌کند عامل هارنس کدام عملیات را بدون درخواست تأیید انجام دهد.

| مقدار           | رفتار                                                  |
| --------------- | --------------------------------------------------------- |
| `approve-all`   | همه نوشتن‌های فایل و فرمان‌های پوسته را به‌طور خودکار تأیید می‌کند.          |
| `approve-reads` | فقط خواندن‌ها را به‌طور خودکار تأیید می‌کند؛ نوشتن و اجرا نیازمند درخواست تأیید هستند. |
| `deny-all`      | همه درخواست‌های مجوز را رد می‌کند.                              |

### `nonInteractivePermissions`

کنترل می‌کند وقتی باید درخواست مجوز نمایش داده شود اما TTY تعاملی در دسترس نیست (که در نشست‌های ACP همیشه چنین است)، چه اتفاقی بیفتد.

| مقدار  | رفتار                                                                 |
| ------ | ------------------------------------------------------------------------ |
| `fail` | نشست را با `PermissionPromptUnavailableError` متوقف می‌کند. **(پیش‌فرض)** |
| `deny` | مجوز را بی‌سروصدا رد می‌کند و ادامه می‌دهد (تنزل تدریجی).        |

### پیکربندی

از طریق پیکربندی Plugin تنظیم کنید:

```bash
openclaw config set plugins.entries.acpx.config.permissionMode approve-all
openclaw config set plugins.entries.acpx.config.nonInteractivePermissions fail
```

پس از تغییر این مقادیر، Gateway را راه‌اندازی مجدد کنید.

<Warning>
پیش‌فرض‌های OpenClaw عبارت‌اند از `permissionMode=approve-reads` و `nonInteractivePermissions=fail`. در نشست‌های غیرتعاملی ACP، هر نوشتن یا اجرایی که درخواست مجوز را فعال کند ممکن است با `PermissionPromptUnavailableError: Permission prompt unavailable in non-interactive mode` ناموفق شود.

اگر لازم است مجوزها را محدود کنید، `nonInteractivePermissions` را روی `deny` تنظیم کنید تا نشست‌ها به‌جای از کار افتادن، به‌تدریج تنزل یابند.
</Warning>

## مرتبط

- [عامل‌های ACP](/fa/tools/acp-agents) — نمای کلی، راهنمای عملیاتی اپراتور، مفاهیم
- [زیرعامل‌ها](/fa/tools/subagents)
- [مسیریابی چندعاملی](/fa/concepts/multi-agent)
