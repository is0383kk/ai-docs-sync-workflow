---
read_when: You want per-agent sandboxing or per-agent tool allow/deny policies in a multi-agent gateway.
sidebarTitle: Multi-agent sandbox and tools
status: active
summary: سندباکس و محدودیت‌های ابزار برای هر عامل، تقدم و مثال‌ها
title: سندباکس و ابزارهای چندعاملی
x-i18n:
    generated_at: "2026-07-27T14:50:58Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 0e07d07c30b844be1e1d93db62fcdaab72c47a5248367559642a959bf09ad193
    source_path: tools/multi-agent-sandbox-tools.md
    workflow: 16
---

هر عامل در یک راه‌اندازی چندعاملی می‌تواند sandbox و سیاست ابزار سراسری را بازنویسی کند. این صفحه پیکربندی هر عامل، قواعد تقدم و نمونه‌ها را پوشش می‌دهد.

<CardGroup cols={3}>
  <Card title="sandbox‌سازی" href="/fa/gateway/sandboxing">
    بک‌اندها و حالت‌ها — مرجع کامل sandbox.
  </Card>
  <Card title="sandbox در برابر سیاست ابزار در برابر حالت ارتقایافته" href="/fa/gateway/sandbox-vs-tool-policy-vs-elevated">
    اشکال‌زدایی «چرا این مسدود شده است؟»
  </Card>
  <Card title="حالت ارتقایافته" href="/fa/tools/elevated">
    اجرای ارتقایافته برای فرستندگان مورداعتماد.
  </Card>
</CardGroup>

<Warning>
احراز هویت در محدوده عامل است: هر عامل مخزن احراز هویت `agentDir` مخصوص خود را در `~/.openclaw/agents/<agentId>/agent/openclaw-agent.sqlite` دارد. هرگز `agentDir` را میان عامل‌ها دوباره استفاده نکنید. عامل‌هایی که نمایه محلی ندارند می‌توانند نمایه‌های احراز هویت عامل پیش‌فرض/اصلی را بخوانند، اما توکن‌های نوسازی OAuth در مخازن عامل‌های ثانویه تکثیر نمی‌شوند. اگر اعتبارنامه‌ها را به‌صورت دستی کپی می‌کنید، فقط نمایه‌های ایستای قابل‌حمل `api_key` یا `token` را کپی کنید.
</Warning>

---

## نمونه‌های پیکربندی

<AccordionGroup>
  <Accordion title="نمونه ۱: عامل شخصی + عامل خانوادگی محدود">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "name": "Personal Assistant",
            "workspace": "~/.openclaw/workspace",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "family",
            "name": "Family Bot",
            "workspace": "~/.openclaw/workspace-family",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read", "message"],
              "deny": ["exec", "write", "edit", "apply_patch", "process", "browser"],
              "message": {
                "crossContext": {
                  "allowWithinProvider": false,
                  "allowAcrossProviders": false
                }
              }
            }
          }
        ]
      },
      "bindings": [
        {
          "agentId": "family",
          "match": {
            "provider": "whatsapp",
            "accountId": "*",
            "peer": {
              "kind": "group",
              "id": "120363424282127706@g.us"
            }
          }
        }
      ]
    }
    ```

    **نتیجه:**

    - عامل `main`: روی میزبان اجرا می‌شود و به همه ابزارها دسترسی دارد.
    - عامل `family`: در Docker اجرا می‌شود (برای هر عامل یک کانتینر) و فقط امکان ارسال پیام با `read` و در گفت‌وگوی جاری را دارد.

  </Accordion>
  <Accordion title="نمونه ۲: عامل کاری با sandbox مشترک">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "personal",
            "workspace": "~/.openclaw/workspace-personal",
            "sandbox": { "mode": "off" }
          },
          {
            "id": "work",
            "workspace": "~/.openclaw/workspace-work",
            "sandbox": {
              "mode": "all",
              "scope": "shared",
              "workspaceRoot": "/tmp/work-sandboxes"
            },
            "tools": {
              "allow": ["read", "write", "apply_patch", "exec"],
              "deny": ["browser", "gateway", "discord"]
            }
          }
        ]
      }
    }
    ```
  </Accordion>
  <Accordion title="نمونه ۲ب: نمایه سراسری کدنویسی + عامل فقط پیام‌رسانی">
    ```json
    {
      "tools": { "profile": "coding" },
      "agents": {
        "list": [
          {
            "id": "support",
            "tools": { "profile": "messaging", "allow": ["slack"] }
          }
        ]
      }
    }
    ```

    **نتیجه:**

    - عامل‌های پیش‌فرض ابزارهای کدنویسی را دریافت می‌کنند.
    - عامل `support` فقط برای پیام‌رسانی است (+ ابزار Slack).

  </Accordion>
  <Accordion title="نمونه ۳: حالت‌های متفاوت sandbox برای هر عامل">
    ```json
    {
      "agents": {
        "defaults": {
          "sandbox": {
            "mode": "non-main",
            "scope": "session"
          }
        },
        "list": [
          {
            "id": "main",
            "workspace": "~/.openclaw/workspace",
            "sandbox": {
              "mode": "off"
            }
          },
          {
            "id": "public",
            "workspace": "~/.openclaw/workspace-public",
            "sandbox": {
              "mode": "all",
              "scope": "agent"
            },
            "tools": {
              "allow": ["read"],
              "deny": ["exec", "write", "edit", "apply_patch"]
            }
          }
        ]
      }
    }
    ```
  </Accordion>
</AccordionGroup>

---

## تقدم پیکربندی

وقتی پیکربندی سراسری (`agents.defaults.*`) و پیکربندی مختص عامل (`agents.entries.*.*`) هر دو وجود داشته باشند:

### پیکربندی sandbox

تنظیمات مختص عامل، تنظیمات سراسری را بازنویسی می‌کنند:

```text
agents.entries.*.sandbox.mode > agents.defaults.sandbox.mode
agents.entries.*.sandbox.scope > agents.defaults.sandbox.scope
agents.entries.*.sandbox.workspaceRoot > agents.defaults.sandbox.workspaceRoot
agents.entries.*.sandbox.workspaceAccess > agents.defaults.sandbox.workspaceAccess
agents.entries.*.sandbox.docker.* > agents.defaults.sandbox.docker.*
agents.entries.*.sandbox.browser.* > agents.defaults.sandbox.browser.*
agents.entries.*.sandbox.prune.* > agents.defaults.sandbox.prune.*
```

<Note>
`agents.entries.*.sandbox.{docker,browser,prune}.*` برای آن عامل، `agents.defaults.sandbox.{docker,browser,prune}.*` را بازنویسی می‌کند (وقتی محدوده sandbox به `"shared"` منتهی شود، نادیده گرفته می‌شود).
</Note>

### محدودیت‌های ابزار

ترتیب پالایش به این صورت است:

<Steps>
  <Step title="نمایه ابزار">
    `tools.profile` یا `agents.entries.*.tools.profile`.
  </Step>
  <Step title="نمایه ابزار ارائه‌دهنده">
    `tools.byProvider[provider].profile` یا `agents.entries.*.tools.byProvider[provider].profile`.
  </Step>
  <Step title="سیاست ابزار سراسری">
    `tools.allow` / `tools.deny`.
  </Step>
  <Step title="سیاست ابزار ارائه‌دهنده">
    `tools.byProvider[provider].allow/deny`.
  </Step>
  <Step title="سیاست ابزار مختص عامل">
    `agents.entries.*.tools.allow/deny`.
  </Step>
  <Step title="سیاست ارائه‌دهنده عامل">
    `agents.entries.*.tools.byProvider[provider].allow/deny`.
  </Step>
  <Step title="سیاست ابزار sandbox">
    `tools.sandbox.tools` یا `agents.entries.*.tools.sandbox.tools`.
  </Step>
  <Step title="سیاست ابزار زیرعامل">
    `tools.subagents.tools`، در صورت کاربرد.
  </Step>
</Steps>

<AccordionGroup>
  <Accordion title="قواعد تقدم">
    - هر سطح می‌تواند ابزارها را بیشتر محدود کند، اما نمی‌تواند ابزارهای ردشده در سطوح قبلی را دوباره مجاز کند.
    - اگر `agents.entries.*.tools.sandbox.tools` تنظیم شده باشد، برای آن عامل جایگزین `tools.sandbox.tools` می‌شود.
    - اگر `agents.entries.*.tools.profile` تنظیم شده باشد، برای آن عامل `tools.profile` را بازنویسی می‌کند.
    - کلیدهای ابزار ارائه‌دهنده، `provider` (برای مثال `google-antigravity`) یا `provider/model` (برای مثال `openai/gpt-5.4`) را می‌پذیرند.

  </Accordion>
  <Accordion title="رفتار فهرست مجاز خالی">
    اگر هر فهرست مجاز صریح در این زنجیره باعث شود هیچ ابزار قابل‌فراخوانی برای اجرا باقی نماند، OpenClaw پیش از ارسال پرامپت به مدل متوقف می‌شود. این رفتار عمدی است: عاملی که با ابزار مفقودی مانند `agents.entries.*.tools.allow: ["query_db"]` پیکربندی شده است، باید تا زمان فعال‌شدن Plugin ثبت‌کننده `query_db` با خطایی آشکار متوقف شود، نه اینکه به‌عنوان عاملی فقط متنی ادامه دهد.
  </Accordion>
</AccordionGroup>

سیاست‌های ابزار از صورت‌های کوتاه `group:*` پشتیبانی می‌کنند که به چندین ابزار گسترش می‌یابند. برای فهرست کامل، به [گروه‌های ابزار](/fa/gateway/sandbox-vs-tool-policy-vs-elevated#tool-groups-shorthands) مراجعه کنید.

بازنویسی‌های ارتقایافته هر عامل (`agents.entries.*.tools.elevated`) می‌توانند اجرای ارتقایافته را برای عامل‌های خاص بیشتر محدود کنند. برای جزئیات به [حالت ارتقایافته](/fa/tools/elevated) مراجعه کنید.

---

## مهاجرت از تک‌عاملی

<Tabs>
  <Tab title="پیش از مهاجرت (تک‌عاملی)">
    ```json
    {
      "agents": {
        "defaults": {
          "workspace": "~/.openclaw/workspace",
          "sandbox": {
            "mode": "non-main"
          }
        }
      },
      "tools": {
        "sandbox": {
          "tools": {
            "allow": ["read", "write", "apply_patch", "exec"],
            "deny": []
          }
        }
      }
    }
    ```
  </Tab>
  <Tab title="پس از مهاجرت (چندعاملی)">
    ```json
    {
      "agents": {
        "list": [
          {
            "id": "main",
            "default": true,
            "workspace": "~/.openclaw/workspace",
            "sandbox": { "mode": "off" }
          }
        ]
      }
    }
    ```
  </Tab>
</Tabs>

<Note>
کلیدهای پیکربندی قدیمی `agents.defaults.*`/`agents.entries.*.*` (مانند `sandbox.perSession`، `agentRuntime`، `embeddedPi`) به‌وسیله `openclaw doctor` مهاجرت داده می‌شوند؛ از این پس `agents.defaults` + `agents.entries` را ترجیح دهید.
</Note>

---

## نمونه‌های محدودیت ابزار

<Tabs>
  <Tab title="عامل فقط‌خواندنی">
    ```json
    {
      "tools": {
        "allow": ["read"],
        "deny": ["exec", "write", "edit", "apply_patch", "process"]
      }
    }
    ```
  </Tab>
  <Tab title="اجرای پوسته با ابزارهای سامانه فایل غیرفعال">
    ```json
    {
      "tools": {
        "allow": ["read", "exec", "process"],
        "deny": ["write", "edit", "apply_patch", "browser", "gateway"]
      }
    }
    ```

    <Warning>
    این سیاست ابزارهای سامانه فایل OpenClaw را غیرفعال می‌کند، اما `exec` همچنان یک پوسته است و می‌تواند هرجا که سامانه فایل میزبان یا sandbox انتخاب‌شده اجازه دهد، فایل بنویسد. برای عامل فقط‌خواندنی، `exec` و `process` را رد کنید، یا دسترسی پوسته را با کنترل‌های سامانه فایل sandbox مانند `agents.defaults.sandbox.workspaceAccess: "ro"` یا `"none"` ترکیب کنید.
    </Warning>

  </Tab>
  <Tab title="فقط ارتباطی">
    ```json
    {
      "tools": {
        "sessions": { "visibility": "tree" },
        "allow": ["sessions_list", "sessions_send", "sessions_history", "session_status"],
        "deny": ["exec", "write", "edit", "apply_patch", "read", "browser"]
      }
    }
    ```

    `sessions_history` در این نمایه همچنان به‌جای تخلیه خام رونوشت، نمایی محدود و پاک‌سازی‌شده از یادآوری را برمی‌گرداند. یادآوری دستیار، تگ‌های تفکر، ساختار `<relevant-memories>`، محموله‌های XML فراخوانی ابزار در متن ساده (از جمله `<tool_call>...</tool_call>`، `<function_call>...</function_call>`، `<tool_calls>...</tool_calls>`، `<function_calls>...</function_calls>` و بلوک‌های ناقص فراخوانی ابزار)، ساختار تنزل‌یافته فراخوانی ابزار، توکن‌های کنترلی لو‌رفته مدل با نویسه‌های ASCII/تمام‌عرض و XML ناقص فراخوانی ابزار MiniMax را پیش از حذف اطلاعات حساس/کوتاه‌سازی حذف می‌کند.

  </Tab>
</Tabs>

---

## دام رایج: "non-main"

<Warning>
`agents.defaults.sandbox.mode: "non-main"` کلید نشست را با کلید نشست اصلی مقایسه می‌کند (همیشه `"main"`؛ `session.mainKey` توسط کاربر قابل‌پیکربندی نیست و OpenClaw درباره هر مقدار دیگری هشدار می‌دهد و آن را نادیده می‌گیرد)، نه با شناسه عامل. نشست‌های گروه/کانال همیشه کلیدهای مختص خود را دریافت می‌کنند، بنابراین غیر اصلی در نظر گرفته شده و sandbox‌سازی می‌شوند. اگر می‌خواهید عاملی هرگز sandbox‌سازی نشود، `agents.entries.*.sandbox.mode: "off"` را تنظیم کنید.
</Warning>

---

## آزمایش

پس از پیکربندی sandbox و ابزارهای چندعاملی:

<Steps>
  <Step title="بررسی تفکیک عامل">
    ```bash
    openclaw agents list --bindings
    ```
  </Step>
  <Step title="تأیید کانتینرهای sandbox">
    ```bash
    docker ps --filter "name=openclaw-sbx-"
    ```
  </Step>
  <Step title="آزمایش محدودیت‌های ابزار">
    - پیامی بفرستید که به ابزارهای محدودشده نیاز داشته باشد.
    - تأیید کنید که عامل نمی‌تواند از ابزارهای ردشده استفاده کند.

  </Step>
  <Step title="پایش گزارش‌ها">
    ```bash
    openclaw logs --follow | grep -E "routing|sandbox|tools"
    ```
  </Step>
</Steps>

---

## عیب‌یابی

<AccordionGroup>
  <Accordion title="عامل با وجود `mode: 'all'` در sandbox قرار نگرفته است">
    - بررسی کنید آیا `agents.defaults.sandbox.mode` سراسری وجود دارد که آن را بازنویسی کند.
    - پیکربندی مختص عامل تقدم دارد، بنابراین `agents.entries.*.sandbox.mode: "all"` را تنظیم کنید.

  </Accordion>
  <Accordion title="ابزارهایی که با وجود فهرست منع همچنان در دسترس‌اند">
    - [ترتیب کامل فیلترکردن](#tool-restrictions) را بررسی کنید: پروفایل ← پروفایل ارائه‌دهنده ← سیاست سراسری ← سیاست ارائه‌دهنده ← سیاست عامل ← سیاست ارائه‌دهندهٔ عامل ← محیط ایزوله ← زیرعامل.
    - هر سطح فقط می‌تواند محدودیت بیشتری اعمال کند و نمی‌تواند مجوزی را بازگرداند.
    - برای اشکال‌زدایی گام‌به‌گام، به [محیط ایزوله در برابر سیاست ابزار در برابر حالت ارتقایافته](/fa/gateway/sandbox-vs-tool-policy-vs-elevated) مراجعه کنید.

  </Accordion>
  <Accordion title="کانتینر برای هر عامل به‌صورت جداگانه ایزوله نشده است">
    - مقدار پیش‌فرض `scope` برابر با `"agent"` است (یک کانتینر برای هر شناسهٔ عامل).
    - برای اختصاص یک کانتینر به هر نشست، `scope: "session"` را تنظیم کنید؛ یا برای استفادهٔ مجدد از یک کانتینر میان عامل‌ها، `scope: "shared"` را تنظیم کنید.

  </Accordion>
</AccordionGroup>

---

## مرتبط

- [حالت ارتقایافته](/fa/tools/elevated)
- [مسیریابی چندعاملی](/fa/concepts/multi-agent)
- [پیکربندی محیط ایزوله](/fa/gateway/config-agents#agentsdefaultssandbox)
- [محیط ایزوله در برابر سیاست ابزار در برابر حالت ارتقایافته](/fa/gateway/sandbox-vs-tool-policy-vs-elevated) — اشکال‌زدایی «چرا این مسدود شده است؟»
- [ایزوله‌سازی](/fa/gateway/sandboxing) — مرجع کامل محیط ایزوله (حالت‌ها، دامنه‌ها، بک‌اندها، ایمیج‌ها)
- [مدیریت نشست](/fa/concepts/session)
