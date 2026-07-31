---
read_when:
    - می‌خواهید تنظیمات OpenClaw را با یک فایل policy.jsonc تألیف‌شده بررسی کنید
    - یافته‌های سیاستی را در lint فرمان doctor می‌خواهید
    - برای شواهد ممیزی به هش گواهی سیاست نیاز دارید
summary: مرجع CLI برای بررسی‌های انطباق `openclaw policy`
title: سیاست
x-i18n:
    generated_at: "2026-07-27T13:57:02Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 63e4faeab8dd6535e3d517439d3f58cdc167b6b7fade808a6482742ec9b5acf1
    source_path: cli/policy.md
    workflow: 16
---

# `openclaw policy`

`openclaw policy` توسط Plugin خط‌مشی همراه ارائه می‌شود. این افزونه یک لایه انطباق سازمانی بر روی تنظیمات موجود OpenClaw است، نه یک سیستم پیکربندی دوم. الزامات را در `policy.jsonc` تدوین می‌کنید؛ OpenClaw فضای کاری فعال را به‌عنوان شواهد مشاهده می‌کند؛ خط‌مشی انحراف را از طریق `doctor --lint` گزارش می‌دهد. خط‌مشی فراخوانی ابزارها را اعمال نمی‌کند یا رفتار زمان اجرا را هنگام درخواست بازنویسی نمی‌کند و مخازن اعتبارنامه مختص هر عامل، مانند `auth-profiles.json`، را گواهی نمی‌کند.

خط‌مشی کانال‌های پیکربندی‌شده، سرورهای MCP، ارائه‌دهندگان مدل، وضعیت SSRF شبکه، دسترسی ورودی/کانال، وضعیت در معرض‌بودن Gateway و فرمان‌های Node، کاوشگرهای تدوین‌شده مسیریابی پیام، دسترسی عامل به فضای کاری، وضعیت سندباکس، وضعیت مدیریت داده، وضعیت ارائه‌دهنده راز/نمایه احراز هویت و فراداده ابزارهای تحت حاکمیت (`TOOLS.md`) را بررسی می‌کند. هنگامی از آن استفاده کنید که یک فضای کاری به بیانی پایدار و قابل‌بررسی مانند «Telegram نباید فعال باشد» یا «ابزارهای تحت حاکمیت باید فراداده ریسک و مالک را اعلام کنند» نیاز دارد. اگر فقط به رفتار محلی بدون گواهی یا تشخیص انحراف نیاز دارید، پیکربندی ساده کافی است.

## شروع سریع

```bash
openclaw plugins enable policy
```

Plugin حتی در صورت نبود `policy.jsonc` فعال می‌ماند تا doctor بتواند به‌جای نادیده‌گرفتن بی‌سروصدای بررسی‌ها، نبود این مصنوع را گزارش کند.

`policy.jsonc` را دستی تدوین کنید؛ این مورد از تنظیمات فعلی تولید نمی‌شود. هر بخش سطح‌بالا یک فضای نام قانون است: یک بررسی فقط زمانی اجرا می‌شود که قانونی مشخص زیر آن وجود داشته باشد (بخش‌ها یا کلیدهای پشتیبانی‌نشده به‌جای نادیده‌گرفته‌شدن بی‌سروصدا به‌صورت `policy/policy-jsonc-invalid` شکست می‌خورند). نمونه‌ای حداقلی که همه بخش‌های پشتیبانی‌شده را پوشش می‌دهد:

```jsonc
{
  "channels": {
    "denyRules": [
      {
        "id": "no-telegram",
        "when": { "provider": "telegram" },
        "reason": "Telegram برای این فضای کاری تأیید نشده است.",
      },
    ],
  },
  "mcp": {
    "servers": {
      "allow": ["docs"],
      "deny": ["untrusted"],
    },
  },
  "models": {
    "providers": {
      "allow": ["openai", "anthropic"],
      "deny": ["openrouter"],
    },
  },
  "network": {
    "privateNetwork": {
      "allow": false,
    },
  },
  "routing": {
    "requireBindings": true,
    "requireConfiguredChannels": true,
    "probes": [
      {
        "id": "family-dm",
        "route": {
          "channel": "imessage",
          "peer": { "kind": "direct", "id": "+15555550123" },
        },
        "expect": {
          "agentId": "family",
          "matchedBy": ["binding.peer"],
        },
      },
    ],
  },
  "ingress": {
    "session": {
      "requireDmScope": "per-channel-peer",
    },
    "channels": {
      "allowDmPolicies": ["pairing", "allowlist", "disabled"],
      "denyOpenGroups": true,
      "requireMentionInGroups": true,
    },
  },
  "gateway": {
    "exposure": {
      "allowNonLoopbackBind": false,
      "allowTailscaleFunnel": false,
    },
    "auth": {
      "requireAuth": true,
      "requireExplicitRateLimit": true,
    },
    "controlUi": {
      "allowInsecure": false,
    },
    "remote": {
      "allow": false,
    },
    "http": {
      "denyEndpoints": ["chatCompletions", "responses"],
      "requireUrlAllowlists": true,
    },
    "nodes": {
      "denyCommands": ["system.run"],
    },
  },
  "agents": {
    "workspace": {
      "allowedAccess": ["none", "ro"],
      "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
    },
  },
  "dataHandling": {
    "sensitiveLogging": {
      "requireRedaction": true,
    },
    "telemetry": {
      "denyContentCapture": true,
    },
    "retention": {
      "requireSessionMaintenance": true,
    },
    "memory": {
      "denySessionTranscriptIndexing": true,
    },
  },
  "secrets": {
    "requireManagedProviders": true,
    "denySources": ["exec"],
    "allowInsecureProviders": false,
  },
  "auth": {
    "profiles": {
      "requireMetadata": ["provider", "mode"],
      "allowModes": ["api_key", "token"],
    },
  },
  "execApprovals": {
    "requireFile": true,
    "defaults": { "allowSecurity": ["deny"] },
    "agents": {
      "allowSecurity": ["deny", "allowlist"],
      "allowAutoAllowSkills": false,
      "allowlist": { "expected": ["deploy", "status"] },
    },
  },
  "tools": {
    "requireMetadata": ["risk", "sensitivity", "owner"],
    "profiles": {
      "allow": ["messaging", "minimal"],
    },
    "fs": {
      "requireWorkspaceOnly": true,
    },
    "exec": {
      "allowSecurity": ["deny", "allowlist"],
      "requireAsk": ["always"],
      "allowHosts": ["sandbox"],
    },
    "elevated": {
      "allow": false,
    },
    "denyTools": ["group:runtime", "group:fs"],
  },
}
```

نکات سراسری که از جدول‌های قوانین زیر آشکار نیستند:

- حذف `gateway.bind` درحالی‌که اتصال‌های غیر-loopback را رد می‌کنید، به این معناست که پیش‌فرض زمان اجرا را می‌پذیرید؛ برای انطباق سخت‌گیرانه، `gateway.bind: "loopback"` را تنظیم کنید.
- برای یک عامل فقط‌خواندنی، `mode` سندباکس را در پیش‌فرض‌ها/عامل مربوط به `all` یا `non-main` و `workspaceAccess` را به `none` یا `ro` تنظیم کنید. حالت سندباکس مفقود یا `off` یک خط‌مشی فقط‌خواندنی را برآورده نمی‌کند.
- `agents.workspace.denyTools` مقادیر `exec`، `process`، `write`، `edit` و `apply_patch` را می‌پذیرد. گروه‌های رد ابزار در پیکربندی، یعنی `group:fs` (تغییر فایل) و `group:runtime` (پوسته/فرایند)، وضعیت معادل را برآورده می‌کنند.
- بررسی‌های تأیید اجرای فرمان، مصنوع زنده `exec-approvals.json` را فقط هنگامی می‌خوانند که یک قانون `execApprovals` وجود داشته باشد؛ مصنوع مفقود یا نامعتبر، شواهد مشاهده‌ناپذیر است، نه قبولی ساختگی.
- شواهد راز و نمایه احراز هویت فقط وضعیت ارائه‌دهنده/منبع و فراداده SecretRef را ثبت می‌کند و هرگز مقادیر خام را ثبت نمی‌کند. خط‌مشی مخازن اعتبارنامه مختص هر عامل، مانند `auth-profiles.json`، را نمی‌خواند یا گواهی نمی‌کند.
- شواهد مدیریت داده فقط وضعیت سطح پیکربندی است (حالت پوشاندن، کلید تغییر ثبت محتوای تله‌متری، حالت نگهداری نشست و تنظیم نمایه‌سازی رونوشت). این شواهد گزارش‌ها، خروجی‌های تله‌متری، رونوشت‌ها یا فایل‌های حافظه را بررسی نمی‌کند و نتیجه پاک اثبات نمی‌کند که هیچ داده شخصی یا رازی در آن‌ها وجود ندارد.
- کاوشگرهای مسیریابی دوباره از تحلیل‌گر اتصال زمان اجرای OpenClaw استفاده می‌کنند. شواهد مسیریابی فقط شناسه کاوشگر، عامل حل‌شده، نوع تطبیق و فراداده پوشانده‌شده اتصال را ثبت می‌کند. این شواهد هرگز شناسه‌های همتا، حساب، انجمن، تیم یا نقش را ثبت نمی‌کند. افزودن بخش مسیریابی عمداً هش‌های خط‌مشی و گواهی را تغییر می‌دهد؛ خط‌مشی‌های بدون مسیریابی شکل فعلی شواهد خود را حفظ می‌کنند.

### مرجع قوانین خط‌مشی

هر قانون زیر اختیاری است؛ یک بررسی فقط زمانی اجرا می‌شود که قانون وجود داشته باشد. وضعیت مشاهده‌شده، پیکربندی موجود OpenClaw یا فراداده فضای کاری است.

#### هم‌پوشان‌های محدوده‌دار

هنگامی از `scopes.<scopeName>` استفاده کنید که عامل‌ها یا کانال‌های مشخص به خط‌مشی سخت‌گیرانه‌تری نسبت به خط مبنای سطح‌بالا نیاز دارند. نام محدوده فقط یک برچسب است؛ تطبیق از انتخاب‌گر داخل محدوده استفاده می‌کند. هم‌پوشان‌ها افزایشی هستند: قانون سراسری همچنان اجرا می‌شود و قانون محدوده‌دار می‌تواند یافته خود را در برابر همان شواهد اضافه کند.

| انتخاب‌گر     | بخش‌های پشتیبانی‌شده                                                             | زمان استفاده                                          |
| ------------ | ------------------------------------------------------------------------------ | ------------------------------------------------- |
| `agentIds`   | `tools`، `agents.workspace`، `sandbox`، `dataHandling.memory`، `execApprovals` | یک یا چند عامل زمان اجرا به قوانین سخت‌گیرانه‌تری نیاز دارند.   |
| `channelIds` | `ingress.channels`                                                             | یک یا چند کانال به قوانین ورودی سخت‌گیرانه‌تری نیاز دارند. |

اگر ورودی `agentIds` در `agents.entries.*` وجود نداشته باشد، OpenClaw به‌جای نادیده‌گرفتن قانون محدوده‌دار، آن را در برابر وضعیت سراسری/پیش‌فرض به‌ارث‌رسیده برای شناسه عامل زمان اجرای مربوط ارزیابی می‌کند.

```jsonc
{
  "tools": {
    "exec": {
      "allowHosts": ["sandbox", "node"],
    },
  },
  "sandbox": {
    "requireMode": ["all", "non-main"],
  },
  "scopes": {
    "release-workspace": {
      "agentIds": ["release-agent", "review-agent"],
      "agents": {
        "workspace": {
          "allowedAccess": ["none", "ro"],
        },
      },
    },
    "release-lockdown": {
      "agentIds": ["release-agent"],
      "tools": {
        "exec": {
          "allowHosts": ["sandbox"],
          "allowSecurity": ["deny", "allowlist"],
          "requireAsk": ["always"],
        },
        "denyTools": ["exec", "process", "write", "edit", "apply_patch"],
      },
      "sandbox": {
        "requireMode": ["all"],
        "allowBackends": ["docker"],
      },
      "dataHandling": {
        "memory": {
          "denySessionTranscriptIndexing": true,
        },
      },
    },
    "shell-sandbox": {
      "agentIds": ["shell-agent"],
      "sandbox": {
        "allowBackends": ["openshell"],
        "containers": {
          "requireReadOnlyMounts": false,
        },
      },
    },
    "telegram-ingress": {
      "channelIds": ["telegram"],
      "ingress": {
        "channels": {
          "allowDmPolicies": ["pairing"],
          "denyOpenGroups": true,
          "requireMentionInGroups": true,
        },
      },
    },
  },
}
```

همان عامل می‌تواند، مانند نمونه بالا، در چند محدوده ظاهر شود؛ به‌شرط آنکه هر محدوده بر فیلدی متفاوت حاکم باشد. فیلد محدوده‌دار تکراری برای همان عامل باید به همان اندازه یا محدودکننده‌تر باشد؛ ادعای تکراری ضعیف‌تر رد می‌شود (فهرست‌های مجاز زیرمجموعه، فهرست‌های ردشده ابرمجموعه و مقادیر بولی الزامی ثابت هستند).

قوانین وضعیت کانتینر (`sandbox.containers.*`) فقط در برابر شواهدی بررسی می‌شوند که بک‌اند سندباکس عامل تطبیق‌یافته بتواند ارائه کند. اگر بک‌اند نتواند قانونی را که برای آن فعال کرده‌اید مشاهده کند، خط‌مشی به‌جای قبولی، `policy/sandbox-container-posture-unobservable` را گزارش می‌کند؛ قوانین کانتینر را به گروه‌های عاملی محدود کنید که از بک‌اندی استفاده می‌کنند که قادر به ارائه آن‌هاست.

`ingress.session.requireDmScope` سطح‌بالا سراسری باقی می‌ماند؛ `session.dmScope` شواهد قابل‌انتساب به کانال نیست، بنابراین نمی‌توان آن را با `channelIds` محدود کرد.

هر محدوده موجود در `policy.jsonc` باید معتبر و قابل‌اعمال باشد.

#### کانال‌ها

| فیلد خط‌مشی                         | وضعیت مشاهده‌شده                          | زمان استفاده                                                     |
| ------------------------------------ | --------------------------------------- | ------------------------------------------------------------ |
| `channels.denyRules[].when.provider` | ارائه‌دهنده و وضعیت فعال‌بودن `channels.*` | کانال‌های پیکربندی‌شده از ارائه‌دهنده‌ای مانند `telegram` را رد کنید. |
| `channels.denyRules[].reason`        | پیام یافته و زمینه راهنمای اصلاح | توضیح دهید چرا ارائه‌دهنده رد شده است.                          |

#### سرورهای MCP

| فیلد خط‌مشی        | وضعیت مشاهده‌شده      | زمان استفاده                                                   |
| ------------------- | ------------------- | ---------------------------------------------------------- |
| `mcp.servers.allow` | شناسه‌های `mcp.servers.*` | الزام کنید که هر سرور MCP پیکربندی‌شده در یک فهرست مجاز باشد. |
| `mcp.servers.deny`  | شناسه‌های `mcp.servers.*` | شناسه‌های مشخص سرور MCP پیکربندی‌شده را رد کنید.                   |

#### ارائه‌دهندگان مدل

| فیلد خط‌مشی             | وضعیت مشاهده‌شده                                   | زمان استفاده                                                                        |
| ------------------------ | ------------------------------------------------ | ------------------------------------------------------------------------------- |
| `models.providers.allow` | شناسه‌های `models.providers.*` و ارجاع‌های مدل انتخاب‌شده | الزام کنید که ارائه‌دهندگان پیکربندی‌شده و ارجاع‌های مدل انتخاب‌شده از ارائه‌دهندگان تأییدشده استفاده کنند. |
| `models.providers.deny`  | شناسه‌های `models.providers.*` و ارجاع‌های مدل انتخاب‌شده | ارائه‌دهندگان پیکربندی‌شده و ارجاع‌های مدل انتخاب‌شده را بر اساس شناسه ارائه‌دهنده رد کنید.               |

#### شبکه

| فیلد سیاست                   | وضعیت مشاهده‌شده                      | زمان استفاده                                                           |
| ------------------------------ | ----------------------------------- | ------------------------------------------------------------------ |
| `network.privateNetwork.allow` | راه‌های گریز SSRF در شبکه خصوصی | روی `false` تنظیم کنید تا دسترسی به شبکه خصوصی الزاماً غیرفعال بماند. |

#### مسیریابی پیام

| فیلد سیاست                        | وضعیت مشاهده‌شده                                      | زمان استفاده                                                               |
| ----------------------------------- | --------------------------------------------------- | ---------------------------------------------------------------------- |
| `routing.requireBindings`           | اتصال‌های مسیر کانال، به‌استثنای اتصال‌های ACP      | وجود حداقل یک اتصال مسیریابی پیام را الزامی کنید.                          |
| `routing.requireConfiguredChannels` | شناسه‌های کانال اتصال و شناسه‌های پیکربندی‌شده `channels.*` | شناسه‌های کانال اتصال منسوخ یا دارای غلط املایی را شناسایی کنید.                        |
| `routing.probes[].route`            | حل‌کننده عمومی مسیر OpenClaw                  | یک مسیر ورودی نمونه را بدون ارسال پیام توصیف کنید.     |
| `routing.probes[].expect.agentId`   | شناسه عامل حل‌شده                                   | رسیدن مسیر به عامل بازبینی‌شده را الزامی کنید.                         |
| `routing.probes[].expect.matchedBy` | نوع تطبیق حل‌کننده                                 | اختصاصی‌بودن اتصال بازبینی‌شده برای همتا، حساب، کانال یا موارد دیگر را الزامی کنید. |

شناسه‌های پروب باید یکتا باشند. یک مسیر از `channel`، `accountId` اختیاری،
`peer`، `parentPeer`، `guildId`، `teamId` و `memberRoleIds` پشتیبانی می‌کند. انواع همتا
عبارت‌اند از `direct`، `group` و `channel`. `matchedBy` ممکن است شامل یک یا چند نوع
تطبیق زمان اجرا باشد، از جمله `binding.peer`، `binding.account`، `binding.channel`
یا `default`.

بررسی‌های مسیریابی فقط بررسی‌های انطباق هستند. آن‌ها راه‌اندازی،
تحویل پیام، تقدم اتصال‌ها یا رفتار بازگشتی را تغییر نمی‌دهند. یافته‌ها به
بازبینی اپراتور نیاز دارند، زیرا تغییر خودکار یک اتصال ممکن است پیام‌های
خصوصی را به مسیر دیگری هدایت کند.

#### ورودی و دسترسی کانال

| فیلد سیاست                              | وضعیت مشاهده‌شده                                                 | زمان استفاده                                                           |
| ----------------------------------------- | -------------------------------------------------------------- | ------------------------------------------------------------------ |
| `ingress.session.requireDmScope`          | `session.dmScope`                                              | یک دامنه جداسازی بازبینی‌شده برای پیام مستقیم را الزامی کنید.                 |
| `ingress.channels.allowDmPolicies`        | `channels.*.dmPolicy` و فیلدهای قدیمی سیاست پیام مستقیم کانال      | فقط سیاست‌های بازبینی‌شده کانال پیام مستقیم را مجاز کنید.               |
| `ingress.channels.denyOpenGroups`         | سیاست ورودی کانال، حساب و گروه                     | ورودی باز گروه را برای کانال‌ها و حساب‌های پیکربندی‌شده رد کنید.      |
| `ingress.channels.requireMentionInGroups` | پیکربندی دروازه اشاره برای کانال، حساب، گروه، سرور و موارد تودرتو | در صورت بازبودن یا مشروط‌بودن ورودی گروه به اشاره، دروازه‌های اشاره را الزامی کنید. |

#### Gateway

| فیلد سیاست                            | وضعیت مشاهده‌شده                                 | زمان استفاده                                                                             |
| --------------------------------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------ |
| `gateway.exposure.allowNonLoopbackBind` | `gateway.bind`                                 | روی `false` تنظیم کنید تا اتصال Gateway به loopback الزامی شود.                                  |
| `gateway.exposure.allowTailscaleFunnel` | وضعیت سرویس‌دهی/تونل Gateway در Tailscale         | روی `false` تنظیم کنید تا در معرض قرارگیری از طریق Tailscale Funnel رد شود.                                    |
| `gateway.auth.requireAuth`              | `gateway.auth.mode`                            | روی `true` تنظیم کنید تا احراز هویت غیرفعال Gateway رد شود.                                       |
| `gateway.auth.requireExplicitRateLimit` | `gateway.auth.rateLimit`                       | روی `true` تنظیم کنید تا پیکربندی صریح محدودیت نرخ احراز هویت الزامی شود.                            |
| `gateway.controlUi.allowInsecure`       | کلیدهای احراز هویت/دستگاه/مبدأ ناامن در رابط کنترل | روی `false` تنظیم کنید تا کلیدهای در معرض قرارگیری ناامن رابط کنترل رد شوند.                         |
| `gateway.remote.allow`                  | حالت/پیکربندی Gateway راه دور                     | روی `false` تنظیم کنید تا حالت Gateway راه دور رد شود.                                          |
| `gateway.http.denyEndpoints`            | نقاط پایانی API مبتنی بر HTTP در Gateway                     | شناسه‌های نقطه پایانی مانند `chatCompletions` یا `responses` را رد کنید.                          |
| `gateway.http.requireUrlAllowlists`     | ورودی‌های دریافت URL مبتنی بر HTTP در Gateway                  | روی `true` تنظیم کنید تا فهرست‌های مجاز URL برای ورودی‌های دریافت URL الزامی شوند.                         |
| `gateway.nodes.denyCommands`            | `gateway.nodes.commands.deny`                  | ردشدن شناسه‌های دقیق فرمان Node، مانند `system.run`، در پیکربندی OpenClaw را الزامی کنید. |

`gateway.nodes.denyCommands` یک قاعده دقیق و حساس به بزرگی و کوچکی حروف برای اَبَرمجموعه رد سیاست است.
زمانی از آن استفاده کنید که سیاست باید ثابت کند فرمان‌های دارای امتیاز Node به‌صراحت
در پیکربندی OpenClaw رد شده‌اند. استقراری که عمداً یک فرمان دارای امتیاز
Node را مجاز می‌کند، باید پس از بازبینی `policy.jsonc` را به‌روزرسانی کند، نه اینکه فقط به
`gateway.nodes.commands.allow` متکی باشد.

#### فضای کاری عامل

| فیلد سیاست                     | وضعیت مشاهده‌شده                                                                           | زمان استفاده                                                                                 |
| -------------------------------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| `agents.workspace.allowedAccess` | `agents.defaults.sandbox.workspaceAccess` و `agents.entries.*.sandbox.workspaceAccess` | فقط مقادیر دسترسی بازبینی‌شده فضای کاری سندباکس، مانند `none` یا `ro`، را مجاز کنید.                       |
| `agents.workspace.denyTools`     | پیکربندی سراسری و مختص هر عامل برای رد ابزارها                                                    | ردشدن ابزارهای تغییر (`exec`، `process`، `write`، `edit`، `apply_patch`) را الزامی کنید. |

#### وضعیت سندباکس

| فیلد سیاست                                          | وضعیت مشاهده‌شده                                          | زمان استفاده                                                       |
| ----------------------------------------------------- | ------------------------------------------------------- | -------------------------------------------------------------- |
| `sandbox.requireMode`                                 | `agents.defaults.sandbox.mode` و حالت مختص هر عامل       | فقط حالت‌های بازبینی‌شده سندباکس، مانند `all` یا `non-main`، را مجاز کنید. |
| `sandbox.allowBackends`                               | `agents.defaults.sandbox.backend` و بک‌اند مختص هر عامل | فقط بک‌اندهای بازبینی‌شده سندباکس، مانند `docker`، را مجاز کنید.         |
| `sandbox.containers.denyHostNetwork`                  | حالت شبکه سندباکس/مرورگر مبتنی بر کانتینر           | حالت شبکه میزبان را رد کنید.                                        |
| `sandbox.containers.denyContainerNamespaceJoin`       | حالت شبکه سندباکس/مرورگر مبتنی بر کانتینر           | پیوستن به فضای نام شبکه کانتینری دیگر را رد کنید.              |
| `sandbox.containers.requireReadOnlyMounts`            | حالت اتصال سندباکس/مرورگر مبتنی بر کانتینر             | فقط‌خواندنی‌بودن اتصال‌ها را الزامی کنید.                                |
| `sandbox.containers.denyContainerRuntimeSocketMounts` | مقصدهای اتصال سندباکس/مرورگر مبتنی بر کانتینر          | اتصال سوکت‌های زمان اجرای کانتینر را رد کنید.                          |
| `sandbox.containers.denyUnconfinedProfiles`           | وضعیت نمایه امنیتی کانتینر                      | نمایه‌های امنیتی بدون محدودیت کانتینر را رد کنید.                   |
| `sandbox.browser.requireCdpSourceRange`               | محدوده مبدأ CDP مرورگر سندباکس                        | اعلام یک محدوده مبدأ برای در معرض قرارگیری CDP مرورگر را الزامی کنید.        |

سیاست، نبود `sandbox.mode` را به‌عنوان مقدار پیش‌فرض ضمنی آن، یعنی `off`، در نظر می‌گیرد؛ بنابراین
`sandbox.requireMode` یک سندباکس تازه یا پیکربندی‌نشده را خارج از
فهرست مجازی مانند `["all"]` گزارش می‌کند.

#### مدیریت داده‌ها

| فیلد سیاست                                        | وضعیت مشاهده‌شده                                                                                     | زمان استفاده                                                               |
| --------------------------------------------------- | -------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `dataHandling.sensitiveLogging.requireRedaction`    | `logging.redactSensitive`                                                                          | روی `true` تنظیم کنید تا `logging.redactSensitive: "off"` رد شود.              |
| `dataHandling.telemetry.denyContentCapture`         | `diagnostics.otel.captureContent`                                                                  | روی `true` تنظیم کنید تا ثبت محتوای تله‌متری رد شود.                     |
| `dataHandling.retention.requireSessionMaintenance`  | `session.maintenance.mode`                                                                         | روی `true` تنظیم کنید تا حالت مؤثر نگهداشت نشست `enforce` الزامی شود. |
| `dataHandling.memory.denySessionTranscriptIndexing` | `memory.qmd.sessions.enabled`، `memory.search.experimental.sessionMemory` و بازنویسی‌های مختص هر عامل | روی `true` تنظیم کنید تا نمایه‌سازی رونوشت نشست در حافظه رد شود.       |

#### اسرار

| فیلد سیاست                      | وضعیت مشاهده‌شده                                           | زمان استفاده                                                                |
| --------------------------------- | -------------------------------------------------------- | ----------------------------------------------------------------------- |
| `secrets.requireManagedProviders` | SecretRefهای پیکربندی و اعلان‌های `secrets.providers.*` | روی `true` تنظیم کنید تا اشاره SecretRefها به ارائه‌دهندگان اعلام‌شده الزامی شود.     |
| `secrets.denySources`             | منابع ارائه‌دهنده اسرار و منابع SecretRef            | منابعی مانند `exec`، `file` یا نام منبع پیکربندی‌شده دیگری را رد کنید. |
| `secrets.allowInsecureProviders`  | پرچم‌های وضعیت ناامن ارائه‌دهنده اسرار                   | روی `false` تنظیم کنید تا ارائه‌دهندگانی که وضعیت ناامن را انتخاب می‌کنند رد شوند.      |

#### تأییدهای اجرا

بررسی‌های تأیید اجرا، مصنوع زمان اجرای `exec-approvals.json` را می‌خوانند:
به‌طور پیش‌فرض `~/.openclaw/exec-approvals.json`، یا
هنگامی که `OPENCLAW_STATE_DIR` تنظیم شده باشد، `$OPENCLAW_STATE_DIR/exec-approvals.json`.
قواعد وضعیت در `execApprovals.defaults.*` یا `execApprovals.agents.*`
به شواهد مصنوع خواندنی نیاز دارند؛ یک مصنوع مفقود یا نامعتبر به‌جای قبولی
بر پایه بهترین تلاش، به‌عنوان شواهد مشاهده‌ناپذیر گزارش می‌شود. پس از خواندنی‌شدن،
فیلدهای حذف‌شده مقادیر پیش‌فرض زمان اجرا را به ارث می‌برند: `defaults.security` مفقود برابر با `full` است و
امنیت مفقود عامل نیز آن مقدار پیش‌فرض را به ارث می‌برد. شواهد شامل `defaults`،
`agents.*`، `agents.*.allowlist[].pattern`، `argPattern` اختیاری، وضعیت مؤثر
`autoAllowSkills` و منبع ورودی هستند — و هرگز شامل مسیر/توکن سوکت،
`commandText`، `lastUsedCommand`، مسیرهای حل‌شده یا مُهرهای زمانی نیستند.

| فیلد سیاست                                | وضعیت مشاهده‌شده                                                                         | زمان استفاده                                                                                |
| ------------------------------------------- | -------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| `execApprovals.requireFile`                 | مسیر `exec-approvals.json` زمان اجرای فعال                                              | برای الزام به وجود داشتن و تجزیه‌پذیر بودن مصنوع تأییدها، روی `true` تنظیم کنید.                     |
| `execApprovals.defaults.allowSecurity`      | `defaults.security`، با مقدار پیش‌فرض `full`                                              | فقط حالت‌های امنیتی تأییدِ پیش‌فرضِ تأییدشده را مجاز کنید.                                    |
| `execApprovals.agents.allowSecurity`        | `agents.*.security`، با ارث‌بری از مقادیر پیش‌فرض                                               | فقط حالت‌های امنیتی تأییدِ مؤثرِ تأییدشده برای هر عامل را مجاز کنید.                        |
| `execApprovals.agents.allowAutoAllowSkills` | `defaults.autoAllowSkills` و `agents.*.autoAllowSkills`، با ارث‌بری از مقادیر پیش‌فرض زمان اجرا | برای الزام به فهرست‌های مجاز دستیِ سخت‌گیرانه بدون تأیید ضمنی CLI مربوط به مهارت، روی `false` تنظیم کنید. |
| `execApprovals.agents.allowlist.expected`   | الگوی تجمیعی `agents.*.allowlist[]` و ورودی‌های اختیاری argPattern               | الزام کنید که فهرست مجاز تأییدها با مجموعه الگوهای بازبینی‌شده مطابقت داشته باشد.                      |

مثال: وجود مصنوع تأییدها را الزامی کنید، مقادیر پیش‌فرض سهل‌گیرانه را رد کنید و
فقط وضعیت تأیید اجرای بازبینی‌شده را برای عامل‌های منتخب مجاز کنید.

```jsonc
{
  "execApprovals": {
    "requireFile": true,
    "defaults": {
      // حالت‌های امنیتی: "deny"، "allowlist" یا "full".
      // این مقدار پیش‌فرض فقط وضعیت ردِ کاملاً محدودشده را مجاز می‌کند.
      "allowSecurity": ["deny"],
    },
  },
  "scopes": {
    "restricted-shell": {
      "agentIds": ["family-agent", "groups-agent"],
      "execApprovals": {
        "agents": {
          // عامل‌های منتخب می‌توانند از وضعیت فهرست مجاز بازبینی‌شده استفاده کنند، اما نه "full".
          "allowSecurity": ["allowlist"],
          // false یعنی CLIهای مهارت باید به‌جای آن‌که به‌طور ضمنی توسط autoAllowSkills تأیید شوند،
          // در فهرست مجاز بازبینی‌شده ظاهر شوند.
          "allowAutoAllowSkills": false,
          "allowlist": {
            "expected": [
              // ورودی ساده: الگوی دقیق اجرایی بازبینی‌شده بدون argPattern.
              "travel-hub",
              // ورودی مقید: الگو به‌همراه عبارت منظم آرگومان بازبینی‌شده.
              { "pattern": "calendar-cli", "argPattern": "^sync\\b" },
              "/bin/date",
            ],
          },
        },
      },
    },
  },
}
```

#### پروفایل‌های احراز هویت

| فیلد سیاست                    | وضعیت مشاهده‌شده                               | زمان استفاده                                                                                   |
| ------------------------------- | -------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `auth.profiles.requireMetadata` | فراداده ارائه‌دهنده و حالتِ `auth.profiles.*` | کلیدهای فراداده‌ای مانند `provider` و `mode` را در پروفایل‌های احراز هویت پیکربندی الزامی کنید.               |
| `auth.profiles.allowModes`      | `auth.profiles.*.mode`                       | فقط حالت‌های پشتیبانی‌شده پروفایل احراز هویت مانند `api_key`، `aws-sdk`، `oauth` یا `token` را مجاز کنید. |

#### فراداده ابزار

| فیلد سیاست            | وضعیت مشاهده‌شده                   | زمان استفاده                                                                                   |
| ----------------------- | -------------------------------- | ------------------------------------------------------------------------------------------ |
| `tools.requireMetadata` | اعلان‌های کنترل‌شده `TOOLS.md` | ابزارهای کنترل‌شده را ملزم کنید کلیدهای فراداده‌ای مانند `risk`، `sensitivity` یا `owner` را اعلام کنند. |

#### وضعیت ابزار

| فیلد سیاست                    | وضعیت مشاهده‌شده                                              | زمان استفاده                                                                                                 |
| ------------------------------- | ----------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| `tools.profiles.allow`          | `tools.profile` و `agents.entries.*.tools.profile`        | فقط شناسه‌های پروفایل ابزار مانند `minimal`، `messaging` یا `coding` را مجاز کنید.                                 |
| `tools.fs.requireWorkspaceOnly` | `tools.fs.workspaceOnly` و بازنویسی‌های `tools.fs` برای هر عامل | برای الزام به وضعیت ابزار فایل‌سیستمِ محدود به فضای کاری، روی `true` تنظیم کنید.                                         |
| `tools.exec.allowSecurity`      | `tools.exec.security` و امنیت اجرای هر عامل           | فقط حالت‌های امنیت اجرای مانند `deny` یا `allowlist` را مجاز کنید.                                            |
| `tools.exec.requireAsk`         | `tools.exec.ask` و حالت پرسش اجرای هر عامل                | وضعیت تأییدی مانند `always` را الزامی کنید.                                                               |
| `tools.exec.allowHosts`         | `tools.exec.host` و مسیریابی میزبان اجرای هر عامل           | فقط حالت‌های مسیریابی میزبان اجرا مانند `sandbox` را مجاز کنید.                                                    |
| `tools.elevated.allow`          | `tools.elevated.enabled` و وضعیت ارتقایافته هر عامل     | برای الزام به غیرفعال ماندن حالت ابزار ارتقایافته، روی `false` تنظیم کنید.                                           |
| `tools.alsoAllow.expected`      | `tools.alsoAllow` و `tools.alsoAllow` هر عامل           | ورودی‌های دقیق `alsoAllow` را الزامی کنید و اعطای ابزار افزایشیِ مفقود یا غیرمنتظره را گزارش دهید.                 |
| `tools.denyTools`               | `tools.deny` و `agents.entries.*.tools.deny`              | الزام کنید فهرست‌های رد ابزارِ پیکربندی‌شده شامل شناسه‌ها یا گروه‌های ابزاری مانند `group:runtime` و `group:fs` باشند. |

## اجرای بررسی‌ها

هنگام نگارش، بررسی‌های صرفاً سیاستی را اجرا کنید:

```bash
openclaw policy check
openclaw policy check --json
openclaw policy check --severity-min error
```

`policy check` فقط مجموعه بررسی‌های سیاست را اجرا می‌کند و شواهد، یافته‌ها
و هش‌های گواهی را منتشر می‌کند. هنگامی که Plugin سیاست فعال باشد، همین یافته‌ها در
`openclaw doctor --lint` نیز ظاهر می‌شوند.

یک فایل سیاست اپراتور را با خط مبنای نگارش‌شده مقایسه کنید:

```bash
openclaw policy compare --baseline official.policy.jsonc
openclaw policy compare --baseline official.policy.jsonc --policy policy.jsonc --json
```

`policy compare` نحو فایل سیاست را با نحو فایل سیاست مقایسه می‌کند؛ این دستور
وضعیت زمان اجرا، شواهد، اطلاعات اعتبارسنجی یا اسرار را بررسی نمی‌کند. از همان
فراداده قواعدی استفاده می‌کند که هم‌پوشانی‌های دامنه‌دار را کنترل می‌کند: فهرست‌های مجاز باید برابر یا
محدودتر بمانند، فهرست‌های رد باید برابر یا گسترده‌تر بمانند، مقادیر بولی الزامی باید
مقدار خود را حفظ کنند، رشته‌های مرتب‌شده فقط می‌توانند به‌سمت انتهای سخت‌گیرانه‌ترِ
ترتیب پیکربندی‌شده حرکت کنند و فهرست‌های دقیق باید مطابقت داشته باشند. خط مبنا می‌تواند یک
سیاست نگارش‌شده توسط سازمان باشد؛ سیاست بررسی‌شده می‌تواند مقادیر سخت‌گیرانه‌تر یا
قواعد اضافی بیفزاید. یک قاعده سطح‌بالای بررسی‌شده می‌تواند قاعده دامنه‌دار خط مبنا را هنگامی
برآورده کند که به همان اندازه یا بیشتر محدودکننده باشد. لازم نیست نام دامنه‌ها میان
فایل‌ها مطابقت داشته باشد؛ مقایسه بر اساس انتخاب‌گر (`agentIds`/`channelIds`) و فیلد کلیدگذاری می‌شود.
برای کاوشگرهای مسیریابی، هر شناسه کاوشگر خط مبنا باید با همان مسیر
و عامل مورد انتظار باقی بماند. یک سیاست بررسی‌شده می‌تواند کاوشگر اضافه کند یا `matchedBy` را محدودتر کند، اما
حذف یک کاوشگر، تغییر مسیر یا عامل آن، یا گسترش انواع تطبیق پذیرفته‌شده آن
ضعیف‌تر است.

مقایسه پاک (`--json`):

```json
{
  "ok": true,
  "baselinePath": "official.policy.jsonc",
  "policyPath": "policy.jsonc",
  "rulesChecked": 3,
  "findings": []
}
```

خروجی پاک `policy check --json` شامل هش‌های پایداری است که اپراتور یا
ناظر می‌تواند ثبت کند:

```json
{
  "ok": true,
  "attestation": {
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "checksRun": 5,
  "checksSkipped": 0,
  "findings": []
}
```

## پیکربندی سیاست

پیکربندی سیاست در `plugins.entries.policy.config` قرار دارد.

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "enabled": true,
        "config": {
          "enabled": true,
          "path": "policy.jsonc",
          "workspaceRepairs": false,
          "expectedHash": "sha256:...",
          "expectedAttestationHash": "sha256:...",
        },
      },
    },
  },
}
```

| تنظیم                   | هدف                                                         |
| ------------------------- | --------------------------------------------------------------- |
| `enabled`                 | بررسی‌های سیاست را حتی پیش از وجود `policy.jsonc` فعال کنید.         |
| `workspaceRepairs`        | به `doctor --fix` اجازه دهید تنظیمات فضای کاری مدیریت‌شده توسط سیاست را ویرایش کند. |
| `expectedHash`            | قفل هش اختیاری برای مصنوع سیاست تأییدشده.            |
| `expectedAttestationHash` | قفل هش اختیاری برای آخرین بررسی پاکِ پذیرفته‌شده سیاست.    |
| `path`                    | مکان مصنوع سیاست نسبت به فضای کاری.             |

برای غیرفعال کردن بررسی‌های سیاست در یک فضای کاری درحالی‌که Plugin نصب باقی می‌ماند،
`plugins.entries.policy.config.enabled` را روی `false` تنظیم کنید.

## پذیرش وضعیت سیاست

نمونه خروجی JSON:

```json
{
  "ok": true,
  "attestation": {
    "checkedAt": "2026-05-10T20:00:00.000Z",
    "policy": {
      "path": "policy.jsonc",
      "hash": "sha256:..."
    },
    "workspace": {
      "scope": "policy",
      "hash": "sha256:..."
    },
    "findingsHash": "sha256:...",
    "attestationHash": "sha256:..."
  },
  "evidence": {
    "channels": [
      {
        "id": "telegram",
        "provider": "telegram",
        "source": "oc://openclaw.config/channels/telegram",
        "enabled": false
      }
    ],
    "mcpServers": [
      {
        "id": "docs",
        "transport": "stdio",
        "source": "oc://openclaw.config/mcp/servers/docs",
        "command": "npx"
      }
    ],
    "modelProviders": [
      {
        "id": "openai",
        "source": "oc://openclaw.config/models/providers/openai"
      }
    ],
    "modelRefs": [
      {
        "ref": "openai/gpt-5.6-sol",
        "provider": "openai",
        "model": "gpt-5.6-sol",
        "source": "oc://openclaw.config/agents/defaults/model"
      }
    ],
    "network": [
      {
        "id": "browser-private-network",
        "source": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
        "value": false
      }
    ],
    "gatewayExposure": [
      {
        "id": "gateway-bind",
        "kind": "bind",
        "source": "oc://openclaw.config/gateway/bind",
        "value": "loopback",
        "nonLoopback": false,
        "explicit": true
      }
    ],
    "agentWorkspace": [
      {
        "id": "agents-defaults-workspace-access",
        "kind": "workspaceAccess",
        "source": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
        "scope": "defaults",
        "value": "ro",
        "sandboxMode": "all",
        "sandboxModeSource": "oc://openclaw.config/agents/defaults/sandbox/mode",
        "sandboxEnabled": true,
        "explicit": true
      },
      {
        "id": "agents-defaults-tool-exec",
        "kind": "toolDeny",
        "source": "oc://openclaw.config/tools/deny",
        "scope": "defaults",
        "tool": "exec",
        "denied": true,
        "explicit": true
      }
    ],
    "secrets": [
      {
        "id": "vault",
        "kind": "provider",
        "source": "oc://openclaw.config/secrets/providers/vault",
        "providerSource": "env"
      },
      {
        "id": "oc://openclaw.config/models/providers/openai/apiKey",
        "kind": "input",
        "source": "oc://openclaw.config/models/providers/openai/apiKey",
        "provenance": "secretRef",
        "refSource": "env",
        "refProvider": "vault"
      }
    ],
    "authProfiles": [
      {
        "id": "github",
        "source": "oc://openclaw.config/auth/profiles/github",
        "validMetadata": true,
        "provider": "github",
        "mode": "token"
      }
    ],
    "tools": [
      {
        "id": "deploy",
        "source": "oc://TOOLS.md/tools/deploy",
        "line": 12,
        "risk": "critical",
        "sensitivity": "restricted",
        "capabilities": ["IRREVERSIBLE_EXTERNAL"]
      }
    ]
  },
  "checksRun": 30,
  "checksSkipped": 0,
  "findings": []
}
```

`attestation.policy.hash` مصنوعِ قانونِ تدوین‌شده را شناسایی می‌کند. `evidence`
وضعیت مشاهده‌شدهٔ OpenClaw را که بررسی‌ها از آن استفاده کرده‌اند ثبت می‌کند و
`workspace.hash` آن بارِ شواهد را شناسایی می‌کند. `findingsHash` مجموعهٔ دقیق
یافته‌ها را شناسایی می‌کند. `checkedAt` زمان اجرای بررسی را ثبت می‌کند.
`attestationHash` ادعای پایدار (هش خط‌مشی، هش شواهد،
هش یافته‌ها و وضعیت پاک/دارای تغییر) را شناسایی می‌کند و عمداً `checkedAt` را کنار می‌گذارد،
تا یک وضعیت خط‌مشی یکسان همیشه همان هش گواهی را تولید کند. این
چهار مقدار در کنار هم چهارتایی ممیزی یک بررسی خط‌مشی را تشکیل می‌دهند.

اگر یک Gateway یا سرپرست از خط‌مشی برای مسدودکردن، تأییدکردن یا حاشیه‌نویسی
یک اقدام زمان اجرا استفاده می‌کند، باید هش گواهی آخرین بررسی
پاک را ثبت کند. `checkedAt` برای گزارش‌های ممیزی در خروجی JSON باقی می‌ماند، اما بخشی از
هش پایدار نیست.

چرخهٔ حیات پذیرش وضعیت خط‌مشی:

1. تدوین یا بازبینی `policy.jsonc`.
2. اجرای `openclaw policy check --json`.
3. اگر پاک بود، `attestation.policy.hash` را به‌عنوان `expectedHash` ثبت کنید.
4. `attestation.attestationHash` را به‌عنوان `expectedAttestationHash` ثبت کنید.
5. `openclaw doctor --lint` را در CI یا دروازه‌های انتشار دوباره اجرا کنید.

اگر قوانین خط‌مشی عمداً تغییر می‌کنند، هر دو هش پذیرفته‌شده را با استفاده از یک
بررسی پاک به‌روزرسانی کنید. اگر فقط تنظیمات فضای کاری تغییر کنند (خط‌مشی یکسان بماند)،
معمولاً فقط `expectedAttestationHash` تغییر می‌کند.

فعال‌سازی یا ارتقای قوانین `agents.workspace`، شواهد `agentWorkspace` را
به هش فضای کاری و هش گواهی اضافه می‌کند؛ پس از فعال‌سازی، شواهد جدید را
بازبینی و هش‌های گواهی پذیرفته‌شده را تازه‌سازی کنید. فعال‌سازی یا ارتقای
قوانین وضعیت ابزار نیز به همین شیوه شواهد `toolPosture` را اضافه می‌کند.

`openclaw policy watch` بررسی را دوباره اجرا می‌کند و زمانی گزارش می‌دهد که شواهد فعلی دیگر
با `expectedAttestationHash` مطابقت ندارند:

```bash
openclaw policy watch --json
```

در CI یا اسکریپت‌هایی که به یک ارزیابی واحدِ انحراف نیاز دارند، از `--once` استفاده کنید. بدون
`--once`، به‌طور پیش‌فرض هر دو ثانیه یک‌بار نظرسنجی می‌کند؛ برای تغییر
بازه از `--interval-ms` استفاده کنید.

## یافته‌ها

| شناسه بررسی                                                 | یافته                                                                           |
| -------------------------------------------------------- | --------------------------------------------------------------------------------- |
| `policy/policy-jsonc-missing`                            | خط‌مشی فعال است، اما `policy.jsonc` وجود ندارد.                                  |
| `policy/policy-jsonc-invalid`                            | خط‌مشی قابل تجزیه نیست یا شامل مدخل‌های قاعده‌ای ناقص است.                       |
| `policy/policy-hash-mismatch`                            | خط‌مشی با `expectedHash` پیکربندی‌شده مطابقت ندارد.                                  |
| `policy/attestation-hash-mismatch`                       | شواهد فعلی خط‌مشی دیگر با گواهی پذیرفته‌شده مطابقت ندارد.               |
| `policy/policy-conformance-invalid`                      | فایل خط‌مشی مبنا یا بررسی‌شده دارای نحو مقایسه نامعتبر است.                  |
| `policy/policy-conformance-missing`                      | در فایل خط‌مشی بررسی‌شده، قاعده‌ای که فایل خط‌مشی مبنا الزام می‌کند وجود ندارد.     |
| `policy/policy-conformance-weaker`                       | فایل خط‌مشی بررسی‌شده مقداری ضعیف‌تر از فایل خط‌مشی مبنا دارد.           |
| `policy/channels-denied-provider`                        | یک کانال فعال با قاعده منع کانال مطابقت دارد.                                   |
| `policy/mcp-denied-server`                               | یک سرور MCP پیکربندی‌شده به‌موجب خط‌مشی ممنوع است.                                      |
| `policy/mcp-unapproved-server`                           | یک سرور MCP پیکربندی‌شده خارج از فهرست مجاز است.                                 |
| `policy/models-denied-provider`                          | یک ارائه‌دهنده مدل یا ارجاع مدل پیکربندی‌شده از ارائه‌دهنده‌ای ممنوع استفاده می‌کند.                  |
| `policy/models-unapproved-provider`                      | یک ارائه‌دهنده مدل یا ارجاع مدل پیکربندی‌شده خارج از فهرست مجاز است.                |
| `policy/network-private-access-enabled`                  | درحالی‌که خط‌مشی آن را منع می‌کند، یک راه فرار SSRF برای شبکه خصوصی فعال است.             |
| `policy/routing-bindings-required`                       | خط‌مشی اتصال مسیر کانال را الزامی می‌کند، اما هیچ‌یک پیکربندی نشده است.                  |
| `policy/routing-binding-channel-unconfigured`            | یک اتصال مسیر، کانالی را نام می‌برد که در `channels.*` وجود ندارد.                         |
| `policy/routing-agent-mismatch`                          | یک مسیر تعریف‌شده به عامل دیگری منتهی می‌شود.                                  |
| `policy/routing-match-kind-mismatch`                     | یک مسیر تعریف‌شده با ویژگی اتصال غیرمنتظره‌ای مطابقت دارد.                   |
| `policy/ingress-dm-policy-unapproved`                    | خط‌مشی پیام مستقیم یک کانال خارج از فهرست مجاز خط‌مشی است.                              |
| `policy/ingress-dm-scope-unapproved`                     | `session.dmScope` با دامنه جداسازی پیام مستقیم الزامی خط‌مشی مطابقت ندارد.          |
| `policy/ingress-open-groups-denied`                      | خط‌مشی گروه یک کانال `open` است، درحالی‌که خط‌مشی ورودی باز گروه را منع می‌کند.          |
| `policy/ingress-group-mention-required`                  | یک مدخل کانال یا گروه درحالی‌که خط‌مشی دروازه‌های اشاره را الزامی می‌کند، آن‌ها را غیرفعال کرده است.       |
| `policy/gateway-non-loopback-bind`                       | وضعیت اتصال Gateway درحالی‌که خط‌مشی آن را منع می‌کند، امکان دسترسی غیرـloopback را می‌دهد.         |
| `policy/gateway-auth-disabled`                           | درحالی‌که خط‌مشی احراز هویت را الزامی می‌کند، احراز هویت Gateway غیرفعال است.                     |
| `policy/gateway-rate-limit-missing`                      | درحالی‌که خط‌مشی آن را الزامی می‌کند، وضعیت محدودیت نرخ احراز هویت Gateway صریح نیست.          |
| `policy/gateway-control-ui-insecure`                     | کلیدهای دسترسی ناامن رابط کنترل Gateway فعال هستند.                         |
| `policy/gateway-tailscale-funnel`                        | درحالی‌که خط‌مشی آن را منع می‌کند، دسترسی Tailscale Funnel در Gateway فعال است.               |
| `policy/gateway-remote-enabled`                          | درحالی‌که خط‌مشی آن را منع می‌کند، حالت راه‌دور Gateway فعال است.                              |
| `policy/gateway-http-endpoint-enabled`                   | یک نقطه پایانی API مبتنی بر HTTP در Gateway، با وجود منع خط‌مشی فعال است.                    |
| `policy/gateway-http-url-fetch-unrestricted`             | ورودی واکشی URL مبتنی بر HTTP در Gateway، فاقد فهرست مجاز URL الزامی است.                      |
| `policy/gateway-node-command-denied`                     | یک فرمان Node که خط‌مشی آن را منع می‌کند، در پیکربندی OpenClaw منع نشده است.                 |
| `policy/agents-workspace-access-denied`                  | حالت سندباکس عامل یا دسترسی فضای کاری خارج از فهرست مجاز خط‌مشی است.           |
| `policy/agents-tool-not-denied`                          | پیکربندی یک عامل یا پیکربندی پیش‌فرض، ابزاری را که خط‌مشی الزام می‌کند منع نکرده است.               |
| `policy/tools-profile-unapproved`                        | پروفایل ابزار سراسری یا مختص عامل پیکربندی‌شده خارج از فهرست مجاز است.           |
| `policy/tools-fs-workspace-only-required`                | ابزارهای فایل‌سیستم با وضعیت مسیر محدود به فضای کاری پیکربندی نشده‌اند.             |
| `policy/tools-exec-security-unapproved`                  | حالت امنیتی اجرا خارج از فهرست مجاز خط‌مشی است.                               |
| `policy/tools-exec-ask-unapproved`                       | حالت پرسش اجرا خارج از فهرست مجاز خط‌مشی است.                                    |
| `policy/tools-exec-host-unapproved`                      | مسیریابی میزبان اجرا خارج از فهرست مجاز خط‌مشی است.                                |
| `policy/tools-elevated-enabled`                          | درحالی‌که خط‌مشی آن را منع می‌کند، حالت ابزار ارتقایافته فعال است.                              |
| `policy/tools-also-allow-missing`                        | در فهرست `alsoAllow` پیکربندی‌شده، مدخلی که خط‌مشی الزام می‌کند وجود ندارد.             |
| `policy/tools-also-allow-unexpected`                     | فهرست `alsoAllow` پیکربندی‌شده شامل مدخلی است که خط‌مشی انتظار ندارد.           |
| `policy/tools-required-deny-missing`                     | فهرست منع ابزار سراسری یا مختص عامل شامل ابزار ممنوع الزامی نیست.     |
| `policy/sandbox-mode-unapproved`                         | حالت سندباکس خارج از فهرست مجاز خط‌مشی است.                                     |
| `policy/sandbox-backend-unapproved`                      | بک‌اند سندباکس خارج از فهرست مجاز خط‌مشی است.                                  |
| `policy/sandbox-container-posture-unobservable`          | یک قاعده وضعیت کانتینر برای بک‌اندی فعال است که نمی‌تواند آن را مشاهده کند.         |
| `policy/sandbox-container-host-network-denied`           | یک سندباکس یا مرورگر مبتنی بر کانتینر از حالت شبکه میزبان استفاده می‌کند.                     |
| `policy/sandbox-container-namespace-join-denied`         | یک سندباکس یا مرورگر مبتنی بر کانتینر به فضای نام کانتینر دیگری می‌پیوندد.          |
| `policy/sandbox-container-mount-mode-required`           | یک اتصال سندباکس یا مرورگر مبتنی بر کانتینر فقط‌خواندنی نیست.                     |
| `policy/sandbox-container-runtime-socket-mount`          | یک اتصال سندباکس یا مرورگر مبتنی بر کانتینر، سوکت زمان‌اجرای کانتینر را در معرض دسترسی قرار می‌دهد. |
| `policy/sandbox-container-unconfined-profile`            | درحالی‌که خط‌مشی آن را منع می‌کند، پروفایل سندباکس کانتینر بدون محدودیت است.                    |
| `policy/sandbox-browser-cdp-source-range-missing`        | درحالی‌که خط‌مشی محدوده مبدأ CDP مرورگر سندباکس را الزامی می‌کند، این محدوده وجود ندارد.             |
| `policy/data-handling-redaction-disabled`                | درحالی‌که خط‌مشی آن را الزامی می‌کند، حذف اطلاعات حساس از گزارش‌ها غیرفعال است.                  |
| `policy/data-handling-telemetry-content-capture`         | درحالی‌که خط‌مشی آن را منع می‌کند، ضبط محتوای تله‌متری فعال است.                       |
| `policy/data-handling-session-retention-not-enforced`    | درحالی‌که خط‌مشی آن را الزامی می‌کند، نگه‌داری حفظ نشست اعمال نمی‌شود.            |
| `policy/data-handling-session-transcript-memory-enabled` | درحالی‌که خط‌مشی آن را منع می‌کند، نمایه‌سازی حافظه رونوشت نشست فعال است.              |
| `policy/secrets-unmanaged-provider`                      | یک SecretRef پیکربندی به ارائه‌دهنده‌ای ارجاع می‌دهد که زیر `secrets.providers` اعلام نشده است.  |
| `policy/secrets-denied-provider-source`                  | یک ارائه‌دهنده راز پیکربندی یا SecretRef از منبعی استفاده می‌کند که خط‌مشی آن را منع کرده است.             |
| `policy/secrets-insecure-provider`                       | درحالی‌که خط‌مشی آن را منع می‌کند، یک ارائه‌دهنده راز وضعیت ناامن را می‌پذیرد.               |
| `policy/auth-profile-invalid-metadata`                   | پروفایل احراز هویت پیکربندی فاقد فراداده معتبر ارائه‌دهنده یا حالت است.                 |
| `policy/auth-profile-unapproved-mode`                    | حالت پروفایل احراز هویت پیکربندی خارج از فهرست مجاز خط‌مشی است.                       |
| `policy/exec-approvals-missing`                          | خط‌مشی `exec-approvals.json` را الزامی می‌کند، اما این مصنوع وجود ندارد.               |
| `policy/exec-approvals-invalid`                          | مصنوع تأییدهای اجرای پیکربندی‌شده قابل تجزیه نیست.                          |
| `policy/exec-approvals-default-security-unapproved`      | پیش‌فرض‌های تأیید اجرا از حالت امنیتی خارج از فهرست مجاز خط‌مشی استفاده می‌کنند.          |
| `policy/exec-approvals-agent-security-unapproved`        | حالت امنیتی مؤثر تأیید اجرای مختص عامل خارج از فهرست مجاز است.       |
| `policy/exec-approvals-auto-allow-skills-enabled`        | یک عامل تأیید اجرا درحالی‌که خط‌مشی آن را منع می‌کند، به‌طور ضمنی CLIهای مهارت را خودکار مجاز می‌کند.   |
| `policy/exec-approvals-allowlist-missing`                | فهرست مجاز تأییدها فاقد الگویی است که خط‌مشی الزام می‌کند.                  |
| `policy/exec-approvals-allowlist-unexpected`             | فهرست مجاز تأییدها شامل الگویی است که خط‌مشی انتظار ندارد.                |
| `policy/tools-missing-risk-level`                        | اعلان یک ابزار تحت حاکمیت فاقد فراداده ریسک است.                             |
| `policy/tools-unknown-risk-level`                        | اعلان یک ابزار تحت حاکمیت از مقدار ریسک ناشناخته‌ای استفاده می‌کند.                           |
| `policy/tools-missing-sensitivity-token`                 | اعلان یک ابزار تحت حاکمیت فاقد فراداده حساسیت است.                      |
| `policy/tools-missing-owner`                             | اعلان یک ابزار تحت حاکمیت فاقد فراداده مالک است.                            |
| `policy/tools-unknown-sensitivity-token`                 | اعلان یک ابزار تحت حاکمیت از مقدار حساسیت ناشناخته‌ای استفاده می‌کند.                    |

یک یافته می‌تواند هم شامل `target` (چیز مشاهده‌شده در فضای کاری که
مطابقت ندارد) و هم شامل `requirement` (قاعده تعریف‌شده‌ای که باعث ایجاد یافته شده است) باشد.
امروزه هر دو رشته‌های نشانی `oc://` هستند، اما نام فیلدها نقش خط‌مشی را
توصیف می‌کنند، نه قالب نشانی را.

نمونه یافته‌ها:

```json
{
  "checkId": "policy/channels-denied-provider",
  "severity": "error",
  "message": "کانال 'telegram' از ارائه‌دهنده ممنوع 'telegram' استفاده می‌کند.",
  "source": "policy",
  "path": "پیکربندی openclaw",
  "ocPath": "oc://openclaw.config/channels/telegram",
  "target": "oc://openclaw.config/channels/telegram",
  "requirement": "oc://policy.jsonc/channels/denyRules/#0",
  "fixHint": "Telegram برای این فضای کاری تأیید نشده است."
}
```

```json
{
  "checkId": "policy/tools-missing-risk-level",
  "severity": "error",
  "message": "ابزار 'deploy' در TOOLS.md طبقه‌بندی صریح ریسک ندارد.",
  "source": "policy",
  "path": "TOOLS.md",
  "line": 12,
  "ocPath": "oc://TOOLS.md/tools/deploy",
  "target": "oc://TOOLS.md/tools/deploy",
  "requirement": "oc://policy.jsonc/tools/requireMetadata"
}
```

```json
{
  "checkId": "policy/mcp-unapproved-server",
  "severity": "error",
  "message": "سرور MCP با نام 'remote' در فهرست مجاز خط‌مشی نیست.",
  "source": "policy",
  "path": "پیکربندی openclaw",
  "ocPath": "oc://openclaw.config/mcp/servers/remote",
  "target": "oc://openclaw.config/mcp/servers/remote",
  "requirement": "oc://policy.jsonc/mcp/servers/allow"
}
```

```json
{
  "checkId": "policy/models-unapproved-provider",
  "severity": "error",
  "message": "ارجاع مدل 'anthropic/claude-sonnet-4.7' از ارائه‌دهنده تأییدنشده 'anthropic' استفاده می‌کند.",
  "source": "policy",
  "path": "پیکربندی openclaw",
  "ocPath": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "target": "oc://openclaw.config/agents/defaults/model/fallbacks/#0",
  "requirement": "oc://policy.jsonc/models/providers/allow"
}
```

```json
{
  "checkId": "policy/network-private-access-enabled",
  "severity": "error",
  "message": "تنظیم شبکه 'browser-private-network' دسترسی به شبکه خصوصی را مجاز می‌کند.",
  "source": "policy",
  "path": "پیکربندی openclaw",
  "ocPath": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "target": "oc://openclaw.config/browser/ssrfPolicy/dangerouslyAllowPrivateNetwork",
  "requirement": "oc://policy.jsonc/network/privateNetwork/allow"
}
```

```json
{
  "checkId": "policy/gateway-non-loopback-bind",
  "severity": "error",
  "message": "تنظیم اتصال Gateway با نام 'gateway-bind' امکان دسترسی از نشانی‌های غیر loopback را فراهم می‌کند.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/bind",
  "target": "oc://openclaw.config/gateway/bind",
  "requirement": "oc://policy.jsonc/gateway/exposure/allowNonLoopbackBind"
}
```

```json
{
  "checkId": "policy/gateway-node-command-denied",
  "severity": "error",
  "message": "فرمان Node در Gateway با نام 'system.run' توسط خط‌مشی منع شده، اما در پیکربندی OpenClaw منع نشده است.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/gateway/nodes/commands/deny",
  "target": "oc://openclaw.config/gateway/nodes/commands/deny",
  "requirement": "oc://policy.jsonc/gateway/nodes/denyCommands",
  "fixHint": "'system.run' را به gateway.nodes.commands.deny اضافه کنید یا پس از بازبینی، خط‌مشی را به‌روزرسانی کنید."
}
```

```json
{
  "checkId": "policy/agents-workspace-access-denied",
  "severity": "error",
  "message": "مقدار 'rw' برای workspaceAccess در محیط ایزوله agents.defaults طبق خط‌مشی مجاز نیست.",
  "source": "policy",
  "path": "openclaw config",
  "ocPath": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "target": "oc://openclaw.config/agents/defaults/sandbox/workspaceAccess",
  "requirement": "oc://policy.jsonc/agents/workspace/allowedAccess"
}
```

## ترمیم

`doctor --lint` و `policy check` فقط‌خواندنی هستند.

`doctor --fix` تنها زمانی تنظیمات فضای کاری مدیریت‌شده توسط خط‌مشی را ویرایش می‌کند که
`workspaceRepairs` به‌صراحت فعال شده باشد؛ در غیر این صورت، بررسی‌ها مواردی را که
ترمیم می‌کردند گزارش می‌کنند و تنظیمات را بدون تغییر باقی می‌گذارند.

در این نسخه، ترمیم می‌تواند کانال‌های منع‌شده توسط `channels.denyRules` را غیرفعال کند و
ترمیم‌های محدودسازی خودکار فهرست‌شده در ادامه را اعمال کند. `workspaceRepairs` را
فقط پس از بازبینی فایل خط‌مشی فعال کنید، زیرا یک قاعده معتبر می‌تواند
پیکربندی فضای کاری را تغییر دهد:

- تنظیم `tools.elevated.enabled=false` هنگامی که یک خط‌مشی سراسری ابزارهای دارای دسترسی ارتقایافته را منع می‌کند
- افزودن شناسه‌های ابزار الزامیِ منع‌شده که وجود ندارند به `tools.deny` یا
  `agents.entries.*.tools.deny` هنگامی که خط‌مشی منع آن ابزارها را الزامی می‌کند
- تنظیم گزینه‌های ناامن `gateway.controlUi.*` روی `false`
- تنظیم `gateway.mode=local` هنگامی که خط‌مشی حالت Gateway راه‌دور را منع می‌کند
- تنظیم مسیرهای گزارش‌شده `gateway.http.endpoints.*.enabled` روی `false` هنگامی که خط‌مشی
  نقاط پایانی API‏ HTTP در Gateway را منع می‌کند
- تنظیم مسیرهای گزارش‌شده `groupPolicy` برای ورودی کانال روی `allowlist` هنگامی که خط‌مشی
  ورودی گروه باز را منع می‌کند
- تنظیم مسیرهای گزارش‌شده `requireMention` برای ورودی کانال روی `true` هنگامی که خط‌مشی
  اشاره‌کردن در گروه را الزامی می‌کند
- تنظیم `logging.redactSensitive=tools` هنگامی که خط‌مشی پوشاندن اطلاعات حساس
  در گزارش‌ها را الزامی می‌کند
- تنظیم `diagnostics.otel.captureContent=false`، یا
  `diagnostics.otel.captureContent.enabled=false` برای تنظیمات ثبت تله‌متری
  به‌شکل شیء، هنگامی که خط‌مشی ثبت محتوای تله‌متری را منع می‌کند

ترمیم ابزارهای دارای دسترسی ارتقایافته در محدوده فقط شناسایی می‌شود. ترمیم‌های مدیریت داده در محدوده نیز
هنگامی که یافته، پیکربندی مشترک گزارش‌گیری یا تله‌متری را گزارش می‌کند نادیده گرفته می‌شوند،
زیرا تغییر تنظیم مشترک، مواردی فراتر از هدف خط‌مشی در محدوده را تحت‌تأثیر قرار می‌دهد.

ترمیم‌های الزامیِ منع در محدوده، هنگامی که یافته `tools.deny` ریشه‌ای به‌ارث‌رسیده را
گزارش می‌کند، نادیده گرفته می‌شوند؛ زیرا افزودن ابزار الزامی به پیکربندی ریشه، مواردی
فراتر از هدف خط‌مشی در محدوده را تحت‌تأثیر قرار می‌دهد. ترمیم‌های الزامیِ منع محلی عامل می‌توانند
مسیر گزارش‌شده `agents.entries.*.tools.deny` را به‌روزرسانی کنند.

ترمیم‌های ورودی کانال در محدوده، هنگامی که یافته `channels.defaults.*` به‌ارث‌رسیده را
گزارش می‌کند، نادیده گرفته می‌شوند؛ زیرا تغییر پیش‌فرض مشترک کانال، مواردی
فراتر از هدف خط‌مشی در محدوده را تحت‌تأثیر قرار می‌دهد. یافته‌های فهرست مجاز واکشی URL از طریق HTTP در Gateway
همچنان دستی باقی می‌مانند، زیرا ترمیم خودکار نمی‌تواند مقادیر صحیح فهرست مجاز URL نقطه پایانی را
انتخاب کند.

یافته‌های اتصال Gateway و فرمان Node همچنان نیازمند بازبینی هستند. هنگامی که
`policy/gateway-non-loopback-bind` یا `policy/gateway-node-command-denied`
قابل نگاشت به یک مسیر پیکربندی باشد، `doctor --fix` تغییر پیشنهادی
`gateway.bind` یا `gateway.nodes.commands.deny` را به‌عنوان راهنمای پیش‌نمایش
نادیده‌گرفته‌شده گزارش می‌کند. این تغییر را اعمال نمی‌کند و تا زمانی که یک اپراتور
پیکربندی یا خط‌مشی را بازبینی و به‌روزرسانی نکند، یافته ترمیم‌شده محسوب نمی‌شود.

```jsonc
{
  "plugins": {
    "entries": {
      "policy": {
        "config": {
          "workspaceRepairs": true,
        },
      },
    },
  },
}
```

## کدهای خروج

| فرمان          | `0`                                                    | `1`                                                                 | `2`                          |
| ---------------- | ------------------------------------------------------ | ------------------------------------------------------------------- | ---------------------------- |
| `policy check`   | هیچ یافته‌ای در آستانه وجود ندارد.                          | یک یا چند یافته به آستانه رسیده‌اند.                             | خطای آرگومان یا زمان اجرا. |
| `policy compare` | فایل خط‌مشی دست‌کم به‌اندازه خط مبنا سخت‌گیرانه است. | فایل خط‌مشی نامعتبر، مفقود یا ضعیف‌تر از قواعد خط مبنا است. | خطای آرگومان یا زمان اجرا. |
| `policy watch`   | هیچ یافته‌ای وجود ندارد و هش پذیرفته‌شده به‌روز است.              | یافته‌هایی وجود دارند یا گواهی پذیرفته‌شده منقضی است.                    | خطای آرگومان یا زمان اجرا. |

## مرتبط

- [حالت lint در Doctor](/fa/cli/doctor#lint-mode)
- [CLI مسیر](/fa/cli/path)
