---
read_when:
    - چند عامل مجزا می‌خواهید (فضاهای کاری + مسیریابی + احراز هویت)
summary: مرجع CLI برای `openclaw agents` (فهرست‌کردن/افزودن/حذف‌کردن/اتصال‌ها/متصل‌کردن/جداکردن/تنظیم هویت)
title: عامل‌ها
x-i18n:
    generated_at: "2026-07-27T13:48:57Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 76a2e50462f6a52760dcb639405ed5f23857f2fa429469281e3acfa1eb61e974
    source_path: cli/agents.md
    workflow: 16
---

# `openclaw agents`

عامل‌های ایزوله (فضاهای کاری + احراز هویت + مسیریابی) را مدیریت کنید. اجرای `openclaw agents` بدون زیر‌دستور، معادل `openclaw agents list` است.

مرتبط:

- [مسیریابی چندعاملی](/fa/concepts/multi-agent)
- [فضای کاری عامل](/fa/concepts/agent-workspace)
- [پیکربندی Skills](/fa/tools/skills-config): پیکربندی نمایان‌بودن مهارت‌ها.

## مثال‌ها

```bash
openclaw agents list
openclaw agents list --bindings
openclaw agents add work --workspace ~/.openclaw/workspace-work
openclaw agents add work --workspace ~/.openclaw/workspace-work --bind telegram:*
openclaw agents add ops --workspace ~/.openclaw/workspace-ops --bind telegram:ops --non-interactive
openclaw agents bindings
openclaw agents bind --agent work --bind telegram:ops
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
openclaw agents set-identity --agent main --avatar avatars/openclaw.png
openclaw agents delete work
```

## سطح دستورات

### `agents list`

گزینه‌ها: `--json`، `--bindings` (قواعد کامل مسیریابی را شامل می‌شود، نه فقط تعداد/خلاصه‌های هر عامل).

### `agents add [name]`

گزینه‌ها: `--workspace <dir>`، `--model <id>`، `--agent-dir <dir>`، `--bind <channel[:accountId]>` (قابل تکرار)، `--non-interactive`، `--json`.

- ارائه هر پرچم صریح افزودن، دستور را به مسیر غیرتعاملی تغییر می‌دهد.
- حالت غیرتعاملی هم به نام عامل و هم به `--workspace` نیاز دارد.
- `main` رزروشده است و نمی‌توان از آن به‌عنوان شناسه عامل جدید استفاده کرد.
- حالت تعاملی، احراز هویت را با کپی‌کردن صرفاً اعتبارنامه‌های ایستای قابل‌انتقال (`api_key` و پروفایل‌های ایستای `token`) مقداردهی اولیه می‌کند، مگر اینکه اعتبارنامه‌ای با `copyToAgents: false` از این کار انصراف دهد؛ پروفایل‌های توکن نوسازی OAuth کپی نمی‌شوند، مگر اینکه ارائه‌دهنده‌ای با `copyToAgents: true` آن را بپذیرد. بدون کپی، OAuth فقط از طریق وراثت خواندنی از مخزن عامل واقعی `main` در دسترس می‌ماند. اگر عامل پیش‌فرض پیکربندی‌شده `main` نیست، برای پروفایل‌های OAuth در عامل جدید جداگانه وارد شوید.

### `agents bindings`

گزینه‌ها: `--agent <id>`، `--json`.

### `agents bind`

گزینه‌ها: `--agent <id>` (پیش‌فرض، عامل پیش‌فرض فعلی است)، `--bind <channel[:accountId]>` (قابل تکرار)، `--json`.

### `agents unbind`

گزینه‌ها: `--agent <id>` (پیش‌فرض، عامل پیش‌فرض فعلی است)، `--bind <channel[:accountId]>` (قابل تکرار)، `--all`، `--json`. یا `--all` یا یک یا چند مقدار `--bind` را می‌پذیرد، نه هر دو را.

### `agents set-identity`

گزینه‌ها: `--agent <id>`، `--workspace <dir>`، `--identity-file <path>`، `--from-identity`، `--name <name>`، `--theme <theme>`، `--emoji <emoji>`، `--avatar <value>`، `--json`. بخش [تنظیم هویت](#set-identity) را در ادامه ببینید.

### `agents delete <id>`

گزینه‌ها: `--force`، `--json`.

- `main` را نمی‌توان حذف کرد.
- بدون `--force`، تأیید تعاملی الزامی است (در نشست غیر TTY ناموفق می‌شود؛ با `--force` دوباره اجرا کنید).
- دایرکتوری‌های فضای کاری، وضعیت عامل و رونوشت نشست به سطل زباله منتقل می‌شوند و به‌طور قطعی حذف نمی‌شوند. اگر سطل زباله در دسترس نباشد، حذف پیکربندی عامل همچنان موفق می‌شود و مسیرهایی را که به پاک‌سازی دستی نیاز دارند گزارش می‌کند.
- وقتی Gateway در دسترس باشد، حذف از طریق Gateway مسیریابی می‌شود تا پاک‌سازی پیکربندی و مخزن نشست از همان نویسنده‌ای استفاده کند که ترافیک زمان اجرا استفاده می‌کند. اگر Gateway در دسترس نباشد، CLI به مسیر محلی آفلاین بازمی‌گردد.
- اگر فضای کاری عامل دیگری همان مسیر باشد، داخل این فضای کاری قرار داشته باشد یا این فضای کاری را دربر گیرد، فضای کاری حفظ می‌شود و `--json` مقادیر `workspaceRetained`، `workspaceRetainedReason` و `workspaceSharedWith` را گزارش می‌کند.

## اتصال‌های مسیریابی

از اتصال‌های مسیریابی برای ثابت‌کردن ترافیک ورودی کانال به یک عامل مشخص استفاده کنید.

اگر همچنین می‌خواهید Skills قابل‌مشاهده برای هر عامل متفاوت باشند، `agents.defaults.skills` و `agents.entries.*.skills` را در `openclaw.json` پیکربندی کنید. [پیکربندی Skills](/fa/tools/skills-config) و [مرجع پیکربندی](/fa/gateway/config-agents#agentsdefaultsskills) را ببینید.

فهرست‌کردن اتصال‌ها:

```bash
openclaw agents bindings
openclaw agents bindings --agent work
openclaw agents bindings --json
```

افزودن اتصال‌ها:

```bash
openclaw agents bind --agent work --bind telegram:ops --bind discord:guild-a
```

همچنین می‌توانید هنگام ایجاد عامل، اتصال‌ها را اضافه کنید:

```bash
openclaw agents add work --workspace ~/.openclaw/workspace-work --bind telegram:* --bind discord:*
```

اگر `accountId` (`--bind <channel>`) را حذف کنید، OpenClaw آن را از هوک‌های راه‌اندازی Plugin، اتصال اجباری حساب یا تعداد حساب‌های پیکربندی‌شده کانال تعیین می‌کند.

اگر `--agent` را برای `bind` یا `unbind` حذف کنید، OpenClaw عامل پیش‌فرض فعلی را هدف قرار می‌دهد.

### قالب `--bind`

| قالب                       | معنا                                                                                            |
| ---------------------------- | -------------------------------------------------------------------------------------------------- |
| `--bind <channel>:*`         | با همه حساب‌های کانال مطابقت دارد.                                                                 |
| `--bind <channel>:<account>` | با یک حساب مطابقت دارد.                                                                                 |
| `--bind <channel>`           | فقط با حساب پیش‌فرض مطابقت دارد، مگر اینکه CLI بتواند با اطمینان دامنه حساب مختص Plugin را تعیین کند. |

### رفتار دامنه اتصال

- یک اتصال ذخیره‌شده بدون `accountId` فقط با حساب پیش‌فرض کانال مطابقت دارد.
- `accountId: "*"` حالت جایگزین کل کانال (همه حساب‌ها) است و نسبت به اتصال صریح حساب، اختصاصی‌بودن کمتری دارد.
- اگر همان عامل از قبل اتصال کانال منطبقی بدون `accountId` داشته باشد و بعداً با یک `accountId` صریح یا تعیین‌شده اتصال ایجاد کنید، OpenClaw به‌جای افزودن مورد تکراری، همان اتصال موجود را درجا ارتقا می‌دهد.

مثال‌ها:

```bash
# تطبیق با همه حساب‌های کانال
openclaw agents bind --agent work --bind telegram:*

# تطبیق با یک حساب مشخص
openclaw agents bind --agent work --bind telegram:ops

# اتصال اولیه فقط در سطح کانال
openclaw agents bind --agent work --bind telegram

# ارتقای بعدی به اتصال در سطح حساب
openclaw agents bind --agent work --bind telegram:alerts
```

پس از ارتقا، مسیریابی آن اتصال به `telegram:alerts` محدود می‌شود. اگر مسیریابی حساب پیش‌فرض را نیز می‌خواهید، آن را صریحاً اضافه کنید (برای مثال `--bind telegram:default`).

حذف اتصال‌ها:

```bash
openclaw agents unbind --agent work --bind telegram:ops
openclaw agents unbind --agent work --all
```

## فایل‌های هویت

هر فضای کاری عامل می‌تواند در ریشه فضای کاری شامل یک `IDENTITY.md` باشد:

- مسیر نمونه: `~/.openclaw/workspace/IDENTITY.md`
- `set-identity --from-identity` از ریشه فضای کاری (یا یک `--identity-file` صریح) می‌خواند.

مسیرهای آواتار نسبت به ریشه فضای کاری تعیین می‌شوند و حتی از طریق پیوند نمادین نیز نمی‌توانند از آن خارج شوند.

## تنظیم هویت

`set-identity` فیلدهای زیر را در `agents.entries.*.identity` می‌نویسد: `name`، `theme`، `emoji`، `avatar` (مسیر نسبی به فضای کاری، نشانی اینترنتی http(s) یا URI داده).

- `--agent` یا `--workspace` عامل هدف را انتخاب می‌کند. اگر `--workspace` با بیش از یک عامل مطابقت داشته باشد، دستور ناموفق می‌شود و از شما می‌خواهد `--agent` را ارائه کنید.
- فایل‌های تصویر آواتار محلی با مسیر نسبی به فضای کاری به 2 MB محدود هستند. نشانی‌های اینترنتی HTTP(S) و URIهای `data:` در برابر محدودیت اندازه فایل محلی بررسی نمی‌شوند.
- وقتی هیچ فیلد هویت صریحی ارائه نشود، دستور داده‌های هویت را از `IDENTITY.md` می‌خواند.

بارگذاری از `IDENTITY.md`:

```bash
openclaw agents set-identity --workspace ~/.openclaw/workspace --from-identity
```

بازنویسی صریح فیلدها:

```bash
openclaw agents set-identity --agent main --name "OpenClaw" --emoji "🦞" --avatar avatars/openclaw.png
```

نمونه پیکربندی:

```json5
{
  agents: {
    list: [
      {
        id: "main",
        identity: {
          name: "OpenClaw",
          theme: "space lobster",
          emoji: "🦞",
          avatar: "avatars/openclaw.png",
        },
      },
    ],
  },
}
```

## مرتبط

- [مرجع CLI](/fa/cli)
- [مسیریابی چندعاملی](/fa/concepts/multi-agent)
- [فضای کاری عامل](/fa/concepts/agent-workspace)
