---
read_when: You want multiple agents with separate workspaces, auth, and sessions in one Gateway process.
sidebarTitle: Multi-agent routing
status: active
summary: 'مسیریابی چندعاملی: مرزهای عامل‌ها، حساب‌های کانال و اتصال‌ها'
title: مسیریابی چندعاملی
x-i18n:
    generated_at: "2026-07-27T15:23:35Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 46df162388205e46d5a4ea3567c8c8f7016117d2ecafe1184a35b4c95798fd80
    source_path: concepts/multi-agent.md
    workflow: 16
---

چندین عامل _ایزوله_ را در یک فرایند Gateway اجرا کنید؛ هرکدام با فضای کاری، دایرکتوری وضعیت (`agentDir`) و تاریخچه نشست مبتنی بر SQLite مختص خود، به‌همراه چندین حساب کانال (برای مثال، دو شماره WhatsApp). پیام‌های ورودی از طریق **اتصال‌ها** به عامل درست هدایت می‌شوند.

یک **عامل** محدوده کامل هر شخصیت است: فایل‌های فضای کاری، نمایه‌های احراز هویت، رجیستری مدل و مخزن نشست. یک **اتصال**، حساب کانال (یک فضای کاری Slack، یک شماره WhatsApp و غیره) را به یکی از این عامل‌ها نگاشت می‌کند.

## عامل چیست

هر عامل موارد مختص خود را دارد:

- **فضای کاری**: فایل‌ها، `AGENTS.md`/`SOUL.md`/`USER.md`، یادداشت‌های محلی، قواعد شخصیت.
- **دایرکتوری وضعیت** (`agentDir`): نمایه‌های احراز هویت، رجیستری مدل، پیکربندی هر عامل.
- **مخزن نشست**: تاریخچه گفت‌وگو و وضعیت مسیریابی در `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`.

نمایه‌های احراز هویت مختص هر عامل‌اند و از مسیر زیر خوانده می‌شوند:

```text
~/.openclaw/agents/<agentId>/agent/auth-profiles.json
```

<Note>
`sessions_history` مسیر امن‌تری برای یادآوری میان‌نشستی است: این مسیر یک نمای محدود و ویرایش‌شده برمی‌گرداند، نه تخلیه خام رونوشت. امضاهای بلوک‌های تفکر، جزئیات محتوای نتیجه ابزار، داربست `<relevant-memories>`، تگ‌های XML فراخوانی ابزار (`<tool_call>`، `<function_call>` و صورت‌های جمع/تنزل‌یافته آن‌ها) و XML فراخوانی ابزار MiniMax را حذف می‌کند، سپس خروجی را کوتاه کرده و بر اساس اندازه بایتی محدود می‌کند.
</Note>

<Warning>
هرگز `agentDir` را بین عامل‌ها دوباره استفاده نکنید؛ این کار باعث تداخل وضعیت احراز هویت/نشست می‌شود. هنگامی‌که اعتبارنامه OAuth محلی یک عامل ثانویه منقضی شده باشد یا تازه‌سازی آن شکست بخورد، OpenClaw اعتبارنامه عامل پیش‌فرض/اصلی با همان شناسه نمایه را می‌خواند و هر توکنی را که تازه‌تر باشد به کار می‌گیرد، بدون آنکه توکن تازه‌سازی را در مخزن عامل ثانویه کپی کند. اگر حساب OAuth کاملاً مستقلی می‌خواهید، از همان عامل وارد شوید. اگر اعتبارنامه‌ها را دستی کپی می‌کنید، فقط نمایه‌های ایستای قابل‌انتقال `api_key` یا `token` را کپی کنید؛ داده‌های تازه‌سازی OAuth به‌طور پیش‌فرض قابل‌انتقال نیستند (`copyToAgents` می‌تواند یک نمایه را صریحاً مشمول این قابلیت کند).
</Warning>

Skills از فضای کاری هر عامل و ریشه‌های مشترکی مانند `~/.openclaw/skills` بارگیری می‌شوند، سپس بر اساس فهرست مجاز مؤثر Skills عامل فیلتر می‌شوند. برای خط‌پایه مشترک از `agents.defaults.skills` و برای جایگزینی مختص هر عامل از `agents.entries.*.skills` استفاده کنید (ورودی‌های صریح، مقدار پیش‌فرض را جایگزین می‌کنند و با آن ادغام نمی‌شوند). به [Skills: مختص هر عامل در برابر مشترک](/fa/tools/skills#per-agent-vs-shared-skills) و [Skills: فهرست‌های مجاز عامل](/fa/tools/skills#agent-allowlists) مراجعه کنید.

ذخیره‌سازی متعلق به Plugin از پیکربندی همان Plugin پیروی می‌کند؛ افزودن عامل دوم
به‌طور خودکار همه مخازن سراسری Plugin را تفکیک نمی‌کند. برای مثال، زمانی‌که
شخصیت‌ها نباید دانش ویکی کامپایل‌شده را به‌اشتراک بگذارند،
[خزانه‌های مختص هر عامل Memory Wiki](/fa/concepts/multi-agent#per-agent-memory-wiki-vaults)
را پیکربندی کنید.

<Note>
**نکته فضای کاری:** فضای کاری هر عامل، **cwd پیش‌فرض** است، نه یک سندباکس سخت‌گیرانه. مسیرهای نسبی درون فضای کاری تفکیک می‌شوند، اما مسیرهای مطلق می‌توانند به مکان‌های دیگر میزبان دسترسی پیدا کنند، مگر آنکه سندباکس فعال باشد. به [سندباکس‌سازی](/fa/gateway/sandboxing) مراجعه کنید.
</Note>

## مسیرها

| مورد                             | پیش‌فرض                                                                                | بازنویسی                                                                                    |
| -------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| پیکربندی                           | `~/.openclaw/openclaw.json`                                                            | `OPENCLAW_CONFIG_PATH`                                                                      |
| دایرکتوری وضعیت                        | `~/.openclaw`                                                                          | `OPENCLAW_STATE_DIR`                                                                        |
| فضای کاری عامل پیش‌فرض        | `~/.openclaw/workspace` (یا `workspace-<profile>` هنگامی‌که `OPENCLAW_PROFILE` تنظیم شده باشد)      | `agents.entries.*.workspace`، سپس `agents.defaults.workspace`، یا `OPENCLAW_WORKSPACE_DIR` |
| فضای کاری سایر عامل‌ها          | `<stateDir>/workspace-<agentId>` (یا `<agents.defaults.workspace>/<agentId>` هنگامی‌که تنظیم شده باشد) | `agents.entries.*.workspace`                                                                |
| دایرکتوری عامل                        | `~/.openclaw/agents/<agentId>/agent`                                                   | `agents.entries.*.agentDir`                                                                 |
| نشست‌ها و رونوشت‌ها         | `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite`                             | —                                                                                           |
| مصنوعات نشست قدیمی/بایگانی‌شده | `~/.openclaw/agents/<agentId>/sessions`                                                | —                                                                                           |

### حالت تک‌عاملی (پیش‌فرض)

اگر چیزی پیکربندی نکنید، OpenClaw یک عامل را اجرا می‌کند:

- `agentId` به‌طور پیش‌فرض `main` است.
- کلید نشست‌ها به‌شکل `agent:main:<mainKey>` است (`mainKey` پیش‌فرض، `main` است).
- فضای کاری به‌طور پیش‌فرض `~/.openclaw/workspace` است (یا `workspace-<profile>`، هنگامی‌که `OPENCLAW_PROFILE` روی مقداری غیر از `default` تنظیم شده باشد).
- وضعیت به‌طور پیش‌فرض `~/.openclaw/agents/main/agent` است.

## ابزار کمکی عامل

یک عامل ایزوله جدید اضافه کنید:

```bash
openclaw agents add work
```

پرچم‌ها: `--workspace <dir>`، `--model <id>`، `--agent-dir <dir>`، `--bind <channel[:accountId]>` (قابل‌تکرار)، `--non-interactive` (نیازمند `--workspace`).

برای مسیریابی پیام‌های ورودی، `bindings` را اضافه کنید (ویزارد انجام این کار را پیشنهاد می‌دهد)، سپس تأیید کنید:

```bash
openclaw agents list --bindings
```

## شروع سریع

<Steps>
  <Step title="ایجاد فضای کاری هر عامل">
    ```bash
    openclaw agents add coding
    openclaw agents add social
    ```

    هر عامل فضای کاری مختص خود را با `SOUL.md`، `AGENTS.md` و `USER.md` اختیاری، به‌همراه یک `agentDir` اختصاصی و مخزن نشست زیر `~/.openclaw/agents/<agentId>` دریافت می‌کند.

  </Step>
  <Step title="ایجاد حساب‌های کانال">
    در کانال‌های دلخواه خود، برای هر عامل یک حساب ایجاد کنید:

    - Discord: برای هر عامل یک بات، Message Content Intent را فعال کنید و هر توکن را کپی کنید.
    - Telegram: برای هر عامل یک بات از طریق BotFather، سپس هر توکن را کپی کنید.
    - WhatsApp: هر شماره تلفن را به حساب مربوطه پیوند دهید.

    ```bash
    openclaw channels login --channel whatsapp --account work
    ```

    راهنماهای کانال را ببینید: [Discord](/fa/channels/discord)، [Telegram](/fa/channels/telegram)، [WhatsApp](/fa/channels/whatsapp).

  </Step>
  <Step title="افزودن عامل‌ها، حساب‌ها و اتصال‌ها">
    عامل‌ها را زیر `agents.entries`، حساب‌های کانال را زیر `channels.<channel>.accounts` اضافه کنید و آن‌ها را با `bindings` به یکدیگر متصل کنید (مثال‌ها در ادامه آمده‌اند).
  </Step>
  <Step title="راه‌اندازی مجدد و تأیید">
    ```bash
    openclaw gateway restart
    openclaw agents list --bindings
    openclaw channels status --probe
    ```
  </Step>
</Steps>

## چندین عامل، چندین شخصیت

هر `agentId` پیکربندی‌شده، مرز شخصیتی متمایزی برای وضعیت اصلی عامل است:

- حساب‌های متفاوت برای هر کانال (به‌ازای هر `accountId`).
- شخصیت‌های متفاوت (`AGENTS.md`/`SOUL.md` مختص هر عامل).
- احراز هویت و نشست‌های جداگانه، که دسترسی میان‌عاملی به آن‌ها فقط از طریق قابلیت‌های صریح یا پیکربندی Plugin فعال می‌شود.

این امکان می‌دهد چندین نفر یک Gateway را به‌اشتراک بگذارند، درحالی‌که وضعیت اصلی عامل‌ها جدا باقی می‌ماند.

## خزانه‌های مختص هر عامل Memory Wiki

Memory Wiki به‌طور پیش‌فرض از یک خزانه سراسری استفاده می‌کند. برای جدا نگه‌داشتن
دانش کامپایل‌شده عامل پشتیبانی از دانش عامل بازاریابی،
`plugins.entries.memory-wiki.config.vault.scope` را روی `agent` تنظیم کنید:

```json5
{
  plugins: {
    entries: {
      "memory-wiki": {
        enabled: true,
        config: {
          vault: {
            scope: "agent",
            path: "~/.openclaw/wiki",
          },
        },
      },
    },
  },
}
```

مسیر پیکربندی‌شده، دایرکتوری والد است. OpenClaw شناسه نرمال‌شده
عامل را به آن می‌افزاید و مسیرهایی مانند `~/.openclaw/wiki/support` و
`~/.openclaw/wiki/marketing` تولید می‌کند. هنگامی‌که چند عامل پیکربندی شده باشند، عملیات
CLI و Gateway در محدوده عامل به تعیین صریح عامل نیاز دارند. برای جزئیات
فیلترکردن پل، مهاجرت و مرز اعتماد، به
[خزانه‌های مختص هر عامل Memory Wiki](/fa/plugins/memory-wiki#per-agent-vaults) مراجعه کنید.

## جست‌وجوی حافظه QMD میان‌عاملی

برای اینکه یک عامل بتواند رونوشت نشست‌های QMD عامل دیگری را جست‌وجو کند، مجموعه‌های اضافی را زیر `agents.entries.*.memory.search.qmd.extraCollections` اضافه کنید. هنگامی‌که همه عامل‌ها باید مجموعه‌های یکسانی را به‌اشتراک بگذارند، از `memory.search.qmd.extraCollections` استفاده کنید.

```json5
{
  agents: {
    defaults: {
      workspace: "~/workspaces/main",
    },
    entries: {
      main: {
        workspace: "~/workspaces/main",
        memory: {
          search: {
            qmd: {
              extraCollections: [{ path: "notes" }], // درون فضای کاری تفکیک می‌شود -> مجموعه‌ای با نام "notes-main"
            },
          },
        },
      },
      family: { workspace: "~/workspaces/family" },
    },
  },
  memory: {
    backend: "qmd",
    search: {
      qmd: {
        extraCollections: [{ path: "~/agents/family/sessions", name: "family-sessions" }],
      },
    },
    qmd: { includeDefaultMemory: false },
  },
}
```

مسیر یک مجموعه اضافی می‌تواند بین عامل‌ها مشترک باشد، اما هنگامی‌که مسیر خارج از فضای کاری عامل است، `name` آن همچنان صریح باقی می‌ماند. مسیرهای درون فضای کاری در محدوده عامل باقی می‌مانند تا هر عامل مجموعه جست‌وجوی رونوشت مختص خود را حفظ کند.

## یک شماره WhatsApp، چندین نفر (تفکیک پیام مستقیم)

با تطبیق فرستنده E.164 (`+15551234567`) با `peer.kind: "direct"`، پیام‌های مستقیم متفاوت WhatsApp را در **یک** حساب WhatsApp به عامل‌های متفاوت هدایت کنید. پاسخ‌ها همچنان از همان شماره WhatsApp ارسال می‌شوند؛ هویت فرستنده مختص هر عامل وجود ندارد.

<Note>
گفت‌وگوهای مستقیم به‌طور پیش‌فرض در کلید نشست اصلی عامل ادغام می‌شوند، بنابراین ایزوله‌سازی واقعی مستلزم یک عامل برای هر شخص است.
</Note>

```json5
{
  agents: {
    list: [
      { id: "alex", workspace: "~/.openclaw/workspace-alex" },
      { id: "mia", workspace: "~/.openclaw/workspace-mia" },
    ],
  },
  bindings: [
    {
      agentId: "alex",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230001" } },
    },
    {
      agentId: "mia",
      match: { channel: "whatsapp", peer: { kind: "direct", id: "+15551230002" } },
    },
  ],
  channels: {
    whatsapp: {
      dmPolicy: "allowlist",
      allowFrom: ["+15551230001", "+15551230002"],
    },
  },
}
```

کنترل دسترسی پیام مستقیم (جفت‌سازی/فهرست مجاز) برای هر حساب WhatsApp سراسری است، نه مختص هر عامل. برای گروه‌های مشترک، گروه را به یک عامل متصل کنید یا از [گروه‌های پخش](/fa/channels/broadcast-groups) استفاده کنید.

## قواعد مسیریابی

اتصال‌ها قطعی‌اند و خاص‌ترین مورد برنده می‌شود. برای ترتیب کامل سطوح (همتای دقیق، همتای والد، نویسه عام همتا، انجمن+نقش‌ها، انجمن، تیم، حساب، کانال، عامل پیش‌فرض)، به [مسیریابی کانال](/fa/channels/channel-routing#routing-rules-how-an-agent-is-chosen) مراجعه کنید. چند قاعده مهم که باید در اینجا برجسته شوند:

- اگر چند اتصال در یک سطح یکسان تطبیق داشته باشند، نخستین مورد بر اساس ترتیب پیکربندی برنده می‌شود.
- اگر یک اتصال چندین فیلد تطبیق تنظیم کند (برای مثال `peer` + `guildId`) همه فیلدهای مشخص‌شده باید تطبیق داشته باشند (معنای `AND`).
- اتصالی که `accountId` را حذف کند، فقط با حساب پیش‌فرض تطبیق دارد، نه با همه حساب‌ها. برای حالت جایگزین سراسری کانال از `accountId: "*"` یا برای یک حساب از `accountId: "<name>"` استفاده کنید. افزودن دوباره همان اتصال با یک شناسه حساب صریح، به‌جای تکرار آن، اتصال موجودِ صرفاً کانالی را ارتقا می‌دهد.

## چندین حساب / شماره تلفن

کانال‌هایی که از چندین حساب پشتیبانی می‌کنند (برای مثال WhatsApp)، از `accountId` برای شناسایی هر ورود استفاده می‌کنند. هر `accountId` به عامل مختص خود هدایت می‌شود، بنابراین یک سرور می‌تواند بدون ترکیب نشست‌ها میزبان چندین شماره تلفن باشد.

برای انتخاب حسابی که هنگام حذف `accountId` استفاده می‌شود، `channels.<channel>.defaultAccount` را تنظیم کنید. اگر تنظیم نشده باشد، OpenClaw در صورت وجود از `default` استفاده می‌کند؛ در غیر این صورت، نخستین شناسه حساب پیکربندی‌شده (پس از مرتب‌سازی) را به‌کار می‌گیرد.

کانال‌هایی که از چند حساب پشتیبانی می‌کنند: `discord`، `feishu`، `googlechat`، `imessage`، `irc`، `line`، `mattermost`، `matrix`، `nextcloud-talk`، `nostr`، `signal`، `slack`، `telegram`، `whatsapp`، `zalo`، `zalouser`.

## مفاهیم

- `agentId`: یک «مغز» (فضای کاری، احراز هویت مختص هر عامل و مخزن نشست مختص هر عامل).
- `accountId`: یک نمونه حساب کانال (برای مثال حساب WhatsApp با `personal` در برابر `biz`).
- `binding`: پیام‌های ورودی را بر اساس `(channel, accountId, peer)` و در صورت نیاز، شناسه‌های انجمن/تیم، به یک `agentId` هدایت می‌کند.
- گفت‌وگوهای مستقیم به `agent:<agentId>:<mainKey>` فروکاسته می‌شوند («اصلی» مختص هر عامل؛ `session.mainKey` را ببینید).

## نمونه‌های پلتفرم

<AccordionGroup>
  <Accordion title="بات‌های Discord برای هر عامل">
    هر حساب بات Discord به یک `accountId` یکتا نگاشت می‌شود. هر حساب را به یک عامل متصل کنید و فهرست‌های مجاز را برای هر بات جداگانه نگه دارید.

    ```json5
    {
      agents: {
        list: [
          { id: "main", workspace: "~/.openclaw/workspace-main" },
          { id: "coding", workspace: "~/.openclaw/workspace-coding" },
        ],
      },
      bindings: [
        { agentId: "main", match: { channel: "discord", accountId: "default" } },
        { agentId: "coding", match: { channel: "discord", accountId: "coding" } },
      ],
      channels: {
        discord: {
          groupPolicy: "allowlist",
          accounts: {
            default: {
              token: "DISCORD_BOT_TOKEN_MAIN",
              guilds: {
                "123456789012345678": {
                  channels: {
                    "222222222222222222": { allow: true, requireMention: false },
                  },
                },
              },
            },
            coding: {
              token: "DISCORD_BOT_TOKEN_CODING",
              guilds: {
                "123456789012345678": {
                  channels: {
                    "333333333333333333": { allow: true, requireMention: false },
                  },
                },
              },
            },
          },
        },
      },
    }
    ```

    - هر بات را به انجمن دعوت و Message Content Intent را فعال کنید.
    - توکن‌ها در `channels.discord.accounts.<id>.token` قرار می‌گیرند (حساب پیش‌فرض می‌تواند از `DISCORD_BOT_TOKEN` استفاده کند).

  </Accordion>
  <Accordion title="بات‌های Telegram برای هر عامل">
    ```json5
    {
      agents: {
        list: [
          { id: "main", workspace: "~/.openclaw/workspace-main" },
          { id: "alerts", workspace: "~/.openclaw/workspace-alerts" },
        ],
      },
      bindings: [
        { agentId: "main", match: { channel: "telegram", accountId: "default" } },
        { agentId: "alerts", match: { channel: "telegram", accountId: "alerts" } },
      ],
      channels: {
        telegram: {
          accounts: {
            default: {
              botToken: "123456:ABC...",
              dmPolicy: "pairing",
            },
            alerts: {
              botToken: "987654:XYZ...",
              dmPolicy: "allowlist",
              allowFrom: ["tg:123456789"],
            },
          },
        },
      },
    }
    ```

    - با BotFather برای هر عامل یک بات ایجاد و توکن هرکدام را کپی کنید.
    - توکن‌ها در `channels.telegram.accounts.<id>.botToken` قرار می‌گیرند (حساب پیش‌فرض می‌تواند از `TELEGRAM_BOT_TOKEN` استفاده کند).
    - برای استفاده از چند بات در یک گروه Telegram، هر بات را دعوت کنید و باتی را که باید پاسخ دهد منشن کنید.
    - Privacy Mode در BotFather را برای هر بات گروه غیرفعال کنید (`/setprivacy` -> Disable)، سپس بات را حذف و دوباره اضافه کنید تا Telegram تنظیم را اعمال کند.
    - گروه‌ها را با `channels.telegram.groups` مجاز کنید، یا فقط برای استقرارهای گروهی مورد اعتماد از `groupPolicy: "open"` استفاده کنید.
    - شناسه‌های کاربران فرستنده را در `groupAllowFrom` قرار دهید. شناسه‌های گروه و ابرگروه باید در `channels.telegram.groups` قرار گیرند، نه `groupAllowFrom`.
    - بر اساس `accountId` اتصال ایجاد کنید تا هر بات به عامل خودش هدایت شود.

  </Accordion>
  <Accordion title="شماره‌های WhatsApp برای هر عامل">
    پیش از راه‌اندازی Gateway، هر حساب را پیوند دهید:

    ```bash
    openclaw channels login --channel whatsapp --account personal
    openclaw channels login --channel whatsapp --account biz
    ```

    `~/.openclaw/openclaw.json` (JSON5):

    ```js
    {
      agents: {
        list: [
          {
            id: "home",
            default: true,
            name: "Home",
            workspace: "~/.openclaw/workspace-home",
            agentDir: "~/.openclaw/agents/home/agent",
          },
          {
            id: "work",
            name: "Work",
            workspace: "~/.openclaw/workspace-work",
            agentDir: "~/.openclaw/agents/work/agent",
          },
        ],
      },

      // مسیریابی قطعی: نخستین تطبیق برنده است (ابتدا مشخص‌ترین مورد).
      bindings: [
        { agentId: "home", match: { channel: "whatsapp", accountId: "personal" } },
        { agentId: "work", match: { channel: "whatsapp", accountId: "biz" } },

        // بازنویسی اختیاری برای هر همتا (مثال: ارسال یک گروه مشخص به عامل کاری).
        {
          agentId: "work",
          match: {
            channel: "whatsapp",
            accountId: "personal",
            peer: { kind: "group", id: "1203630...@g.us" },
          },
        },
      ],

      // به‌طور پیش‌فرض غیرفعال است: پیام‌رسانی عامل‌به‌عامل باید صراحتاً فعال و در فهرست مجاز قرار داده شود.
      tools: {
        agentToAgent: {
          enabled: false,
          allow: ["home", "work"],
        },
      },

      channels: {
        whatsapp: {
          accounts: {
            personal: {
              // بازنویسی اختیاری. پیش‌فرض: ~/.openclaw/credentials/whatsapp/personal
              // authDir: "~/.openclaw/credentials/whatsapp/personal",
            },
            biz: {
              // بازنویسی اختیاری. پیش‌فرض: ~/.openclaw/credentials/whatsapp/biz
              // authDir: "~/.openclaw/credentials/whatsapp/biz",
            },
          },
        },
      },
    }
    ```

  </Accordion>
</AccordionGroup>

## الگوهای رایج

<Tabs>
  <Tab title="کارهای روزمره با WhatsApp و کار عمیق با Telegram">
    بر اساس کانال تفکیک کنید: WhatsApp را به یک عامل سریع برای کارهای روزمره و Telegram را به یک عامل Opus هدایت کنید.

    ```json5
    {
      agents: {
        list: [
          {
            id: "chat",
            name: "Everyday",
            workspace: "~/.openclaw/workspace-chat",
            model: "anthropic/claude-sonnet-4-6",
          },
          {
            id: "opus",
            name: "Deep Work",
            workspace: "~/.openclaw/workspace-opus",
            model: "anthropic/claude-opus-4-6",
          },
        ],
      },
      bindings: [
        { agentId: "chat", match: { channel: "whatsapp", accountId: "*" } },
        { agentId: "opus", match: { channel: "telegram", accountId: "*" } },
      ],
    }
    ```

    این نمونه‌ها از `accountId: "*"` استفاده می‌کنند تا اگر بعداً حساب‌هایی اضافه کردید، اتصال‌ها همچنان کار کنند. برای هدایت یک پیام مستقیم/گروه مشخص به Opus و نگه‌داشتن بقیه روی عامل گفت‌وگو، یک اتصال `match.peer` برای آن همتا اضافه کنید — تطبیق‌های همتا همیشه بر قواعد سراسری کانال اولویت دارند.

  </Tab>
  <Tab title="یک کانال، هدایت یک همتا به Opus">
    WhatsApp را روی عامل سریع نگه دارید، اما یک پیام مستقیم را به Opus هدایت کنید:

    ```json5
    {
      agents: {
        list: [
          {
            id: "chat",
            name: "Everyday",
            workspace: "~/.openclaw/workspace-chat",
            model: "anthropic/claude-sonnet-4-6",
          },
          {
            id: "opus",
            name: "Deep Work",
            workspace: "~/.openclaw/workspace-opus",
            model: "anthropic/claude-opus-4-6",
          },
        ],
      },
      bindings: [
        {
          agentId: "opus",
          match: { channel: "whatsapp", accountId: "*", peer: { kind: "direct", id: "+15551234567" } },
        },
        { agentId: "chat", match: { channel: "whatsapp", accountId: "*" } },
      ],
    }
    ```

    اتصال‌های همتا همیشه اولویت دارند، بنابراین آن‌ها را بالاتر از قاعده سراسری کانال نگه دارید.

  </Tab>
  <Tab title="عامل خانواده متصل به یک گروه WhatsApp">
    یک عامل اختصاصی خانواده را با الزام منشن و سیاست ابزار محدودتر به یک گروه WhatsApp متصل کنید:

    ```json5
    {
      agents: {
        list: [
          {
            id: "family",
            name: "Family",
            workspace: "~/.openclaw/workspace-family",
            identity: { name: "Family Bot" },
            groupChat: {
              mentionPatterns: ["@family", "@familybot", "@Family Bot"],
            },
            sandbox: {
              mode: "all",
              scope: "agent",
            },
            tools: {
              allow: [
                "exec",
                "read",
                "sessions_list",
                "sessions_history",
                "sessions_send",
                "sessions_spawn",
                "session_status",
              ],
              deny: ["write", "edit", "apply_patch", "browser", "canvas", "nodes", "cron"],
            },
          },
        ],
      },
      bindings: [
        {
          agentId: "family",
          match: {
            channel: "whatsapp",
            peer: { kind: "group", id: "120363999999999999@g.us" },
          },
        },
      ],
    }
    ```

    فهرست‌های مجاز/ممنوع ابزار، **ابزار** هستند نه مهارت. اگر مهارتی نیاز به اجرای یک فایل دودویی دارد، مطمئن شوید `exec` مجاز است و فایل دودویی در سندباکس وجود دارد. برای کنترل سخت‌گیرانه‌تر، `agents.entries.*.groupChat.mentionPatterns` را تنظیم کنید و فهرست‌های مجاز گروه را برای کانال فعال نگه دارید.

  </Tab>
</Tabs>

## پیکربندی سندباکس و ابزار برای هر عامل

هر عامل می‌تواند محدودیت‌های سندباکس و ابزار مختص خود را داشته باشد:

```js
{
  agents: {
    list: [
      {
        id: "personal",
        workspace: "~/.openclaw/workspace-personal",
        sandbox: {
          mode: "off",  // بدون سندباکس برای عامل شخصی
        },
        // بدون محدودیت ابزار - همه ابزارها در دسترس‌اند
      },
      {
        id: "family",
        workspace: "~/.openclaw/workspace-family",
        sandbox: {
          mode: "all",     // همیشه در سندباکس
          scope: "agent",  // یک کانتینر برای هر عامل
          docker: {
            // راه‌اندازی اولیه اختیاری و یک‌باره پس از ایجاد کانتینر
            setupCommand: "apt-get update && apt-get install -y git curl",
          },
        },
        tools: {
          allow: ["read"],                    // فقط ابزار خواندن
          deny: ["exec", "write", "edit", "apply_patch"],    // منع سایر ابزارها
        },
      },
    ],
  },
}
```

<Note>
`setupCommand` زیر `sandbox.docker` قرار دارد و هنگام ایجاد کانتینر یک‌بار اجرا می‌شود. وقتی دامنه نهایی `"shared"` باشد، بازنویسی‌های مختص هر عامل در `sandbox.docker.*` نادیده گرفته می‌شوند.
</Note>

این امکانات را در اختیار شما قرار می‌دهد:

- **جداسازی امنیتی**: ابزارهای عامل‌های غیرقابل اعتماد را محدود کنید.
- **کنترل منابع**: عامل‌های مشخص را در سندباکس اجرا کنید و سایر عامل‌ها را روی میزبان نگه دارید.
- **سیاست‌های انعطاف‌پذیر**: مجوزهای متفاوت برای هر عامل.

<Note>
`tools.elevated` هم یک دروازه سراسری (`tools.elevated.enabled`/`allowFrom`) و هم یک دروازه مختص هر عامل (`agents.entries.*.tools.elevated.enabled`/`allowFrom`) دارد. دروازه مختص هر عامل فقط می‌تواند محدودیت دروازه سراسری را بیشتر کند — برای اجرای فرمان‌های دارای سطح دسترسی بالاتر، هر دو باید فرستنده را مجاز بدانند. برای هدف‌گیری گروهی، از `agents.entries.*.groupChat.mentionPatterns` استفاده کنید تا @منشن‌ها به‌درستی به عامل موردنظر نگاشت شوند.
</Note>

برای نمونه‌های تفصیلی، [سندباکس و ابزارهای چندعاملی](/fa/tools/multi-agent-sandbox-tools) را ببینید.

## مرتبط

- [عامل‌های ACP](/fa/tools/acp-agents) — اجرای مهارهای کدنویسی خارجی
- [مسیریابی کانال](/fa/channels/channel-routing) — نحوهٔ مسیریابی پیام‌ها به عامل‌ها
- [حضور](/fa/concepts/presence) — حضور و دسترس‌پذیری عامل
- [نشست](/fa/concepts/session) — جداسازی و مسیریابی نشست
- [زیرعامل‌ها](/fa/tools/subagents) — راه‌اندازی اجراهای پس‌زمینهٔ عامل
