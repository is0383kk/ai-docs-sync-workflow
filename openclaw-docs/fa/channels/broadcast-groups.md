---
read_when:
    - پیکربندی گروه‌های پخش همگانی
    - اشکال‌زدایی پاسخ‌های چندعاملی در WhatsApp
sidebarTitle: Broadcast groups
status: experimental
summary: یک پیام WhatsApp را برای چند عامل پخش کنید
title: گروه‌های پخش همگانی
x-i18n:
    generated_at: "2026-07-27T13:42:23Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: a468e4c65d2cc89bda24e8e599f8a45015e3f77f1073612b105daed8877c0ff9
    source_path: channels/broadcast-groups.md
    workflow: 16
---

<Note>
**وضعیت:** آزمایشی. در 2026.1.9 اضافه شده است. فقط WhatsApp (کانال وب).
</Note>

## نمای کلی

گروه‌های پخش، **چندین عامل** را برای یک پیام ورودی یکسان اجرا می‌کنند. هر عامل پیام را در نشست ایزولهٔ خودش پردازش می‌کند و پاسخ خودش را می‌فرستد؛ بنابراین یک شمارهٔ WhatsApp می‌تواند در یک گفت‌وگوی گروهی یا پیام مستقیم، میزبان تیمی از عامل‌های تخصصی باشد.

گروه‌های پخش پس از فهرست‌های مجاز کانال و قواعد فعال‌سازی گروه ارزیابی می‌شوند. در گروه‌های WhatsApp، پخش زمانی انجام می‌شود که OpenClaw در حالت عادی پاسخ می‌داد (برای مثال: هنگام منشن‌شدن، بسته به تنظیمات گروه). این گروه‌ها فقط **عامل‌هایی را که اجرا می‌شوند** تغییر می‌دهند و هرگز تعیین نمی‌کنند که آیا یک پیام واجد شرایط پردازش است یا نه.

لاین زندهٔ QA برای WhatsApp شامل `whatsapp-broadcast-group-fanout` است که تأیید می‌کند یک پیام گروهی منشن‌دار می‌تواند پاسخ‌های قابل‌مشاهده و متمایزی از دو عامل پیکربندی‌شده ایجاد کند.

## پیکربندی

### راه‌اندازی پایه

یک بخش سطح‌بالای `broadcast` (در کنار `bindings`) اضافه کنید. کلیدها شناسه‌های همتای WhatsApp و مقادیر آرایه‌هایی از شناسه‌های عامل هستند:

- گفت‌وگوهای گروهی: JID گروه (برای مثال `120363403215116621@g.us`)
- پیام‌های مستقیم: شماره‌تلفن E.164 فرستنده (برای مثال `+15551234567`)

```json
{
  "broadcast": {
    "120363403215116621@g.us": ["alfred", "baerbel", "assistant3"]
  }
}
```

**نتیجه:** هر زمان OpenClaw در این گفت‌وگو پاسخ دهد، هر سه عامل را اجرا می‌کند.

هر شناسهٔ عامل فهرست‌شده باید در `agents.entries` وجود داشته باشد: اعتبارسنجی پیکربندی شناسه‌های ناشناخته را گزارش می‌کند و زمان اجرا با هشدار `Broadcast agent <id> not found in agents.entries; skipping` از آن‌ها صرف‌نظر می‌کند.

### راهبرد پردازش

`broadcast.strategy` نحوهٔ پردازش پیام توسط عامل‌ها را تعیین می‌کند:

| راهبرد             | رفتار                                                              |
| -------------------- | --------------------------------------------------------------------- |
| `parallel` (پیش‌فرض) | همهٔ عامل‌ها به‌طور هم‌زمان پردازش می‌کنند؛ پاسخ‌ها ممکن است با هر ترتیبی برسند.       |
| `sequential`         | عامل‌ها به‌ترتیب آرایه پردازش می‌کنند؛ هر عامل منتظر پایان عامل قبلی می‌ماند. |

```json
{
  "broadcast": {
    "strategy": "sequential",
    "120363403215116621@g.us": ["alfred", "baerbel"]
  }
}
```

### مثال کامل

```json
{
  "agents": {
    "list": [
      {
        "id": "code-reviewer",
        "name": "Code Reviewer",
        "workspace": "/path/to/code-reviewer",
        "sandbox": { "mode": "all" }
      },
      {
        "id": "security-auditor",
        "name": "Security Auditor",
        "workspace": "/path/to/security-auditor",
        "sandbox": { "mode": "all" }
      },
      {
        "id": "docs-generator",
        "name": "Documentation Generator",
        "workspace": "/path/to/docs-generator",
        "sandbox": { "mode": "all" }
      }
    ]
  },
  "broadcast": {
    "strategy": "parallel",
    "120363403215116621@g.us": ["code-reviewer", "security-auditor", "docs-generator"],
    "120363424282127706@g.us": ["support-en", "support-de"],
    "+15555550123": ["assistant", "logger"]
  }
}
```

## نحوهٔ کار

### جریان پیام

<Steps>
  <Step title="رسیدن پیام ورودی">
    یک پیام گروهی یا مستقیم WhatsApp می‌رسد.
  </Step>
  <Step title="مسیریابی و پذیرش">
    OpenClaw فهرست‌های مجاز کانال، قواعد فعال‌سازی گروه و مالکیت اتصال ACP پیکربندی‌شده را اعمال می‌کند.
  </Step>
  <Step title="بررسی پخش">
    اگر هیچ اتصال ACP پیکربندی‌شده‌ای مالک مسیر نباشد، OpenClaw بررسی می‌کند که آیا شناسهٔ همتا در `broadcast` وجود دارد یا نه.
  </Step>
  <Step title="اگر پخش اعمال شود">
    - همهٔ عامل‌های فهرست‌شده پیام را پردازش می‌کنند.
    - هر عامل کلید نشست و زمینهٔ ایزولهٔ خودش را دارد.
    - عامل‌ها به‌صورت موازی (پیش‌فرض) یا ترتیبی پردازش می‌کنند.
    - پیوست‌های صوتی پیش از توزیع، یک‌بار رونویسی می‌شوند؛ بنابراین عامل‌ها به‌جای برقراری فراخوانی‌های جداگانهٔ STT، یک رونویسی مشترک دارند.

  </Step>
  <Step title="اگر پخش اعمال نشود">
    OpenClaw مسیر عادی یا مسیر نشست ACP پیکربندی‌شده‌ای را که هنگام مسیریابی انتخاب شده است، اعزام می‌کند.
  </Step>
</Steps>

<Note>
گروه‌های پخش، فهرست‌های مجاز کانال یا قواعد فعال‌سازی گروه (منشن‌ها/فرمان‌ها/و غیره) را دور نمی‌زنند. آن‌ها فقط هنگامی که یک پیام واجد شرایط پردازش است، _عامل‌هایی را که اجرا می‌شوند_ تغییر می‌دهند.
</Note>

### ایزوله‌سازی نشست

هر عامل در یک گروه پخش، موارد زیر را کاملاً جداگانه نگه می‌دارد:

- **کلیدهای نشست** (`agent:alfred:whatsapp:group:120363...` در برابر `agent:baerbel:whatsapp:group:120363...`)
- **تاریخچهٔ مکالمه** (یک عامل پاسخ‌های عامل‌های دیگر را نمی‌بیند)
- **فضای کاری** (در صورت پیکربندی، سندباکس‌های جداگانه)
- **دسترسی به ابزارها** (فهرست‌های مجاز/غیرمجاز متفاوت)
- **حافظه/زمینه** (`IDENTITY.md`، `SOUL.md` و غیره، به‌صورت جداگانه)

یک استثنا عمداً مشترک است: **بافر زمینهٔ گروه** (پیام‌های اخیر گروه که برای زمینه استفاده می‌شوند) به‌ازای هر همتا مشترک است؛ بنابراین همهٔ عامل‌های پخش هنگام فعال‌شدن، زمینهٔ یکسانی را می‌بینند. این بافر پس از تکمیل توزیع، یک‌بار پاک می‌شود.

این قابلیت اجازه می‌دهد هر عامل شخصیت، مدل، Skills و دسترسی ابزار متفاوتی داشته باشد (برای مثال فقط‌خواندنی در برابر خواندن‌ونوشتن).

### مثال: نشست‌های ایزوله

در گروه `120363403215116621@g.us` با عامل‌های `["alfred", "baerbel"]`:

<Tabs>
  <Tab title="زمینهٔ Alfred">
    ```text
    نشست: agent:alfred:whatsapp:group:120363403215116621@g.us
    تاریخچه: [پیام کاربر، پاسخ‌های قبلی alfred]
    فضای کاری: ~/openclaw-alfred/
    ابزارها: خواندن، نوشتن، اجرا
    ```
  </Tab>
  <Tab title="زمینهٔ Baerbel">
    ```text
    نشست: agent:baerbel:whatsapp:group:120363403215116621@g.us
    تاریخچه: [پیام کاربر، پاسخ‌های قبلی baerbel]
    فضای کاری: ~/openclaw-baerbel/
    ابزارها: فقط خواندن
    ```
  </Tab>
</Tabs>

## موارد استفاده

- **تیم‌های عامل تخصصی**: یک گروه توسعه که در آن `code-reviewer`، `security-auditor`، `test-generator` و `docs-checker` هرکدام از دیدگاه خود به یک پیام یکسان پاسخ می‌دهند.
- **پشتیبانی چندزبانه**: یک گفت‌وگوی پشتیبانی با `support-en`، `support-de` و `support-es` که به زبان‌های خود پاسخ می‌دهند.
- **تضمین کیفیت**: `support-agent` پاسخ می‌دهد، درحالی‌که `qa-agent` بازبینی می‌کند و فقط زمانی پاسخ می‌دهد که مشکلی بیابد.
- **خودکارسازی وظایف**: `task-tracker`، `time-logger` و `report-generator` همگی یک به‌روزرسانی وضعیت یکسان را مصرف می‌کنند.

## بهترین شیوه‌ها

<AccordionGroup>
  <Accordion title="1. عامل‌ها را متمرکز نگه دارید">
    به‌جای یک عامل عمومی "dev-helper"، به هر عامل یک مسئولیت واحد و روشن (`formatter`، `linter`، `tester`) بدهید.
  </Accordion>
  <Accordion title="2. از شناسه‌ها و نام‌های توصیفی استفاده کنید">
    ```json
    {
      "agents": {
        "list": [
          { "id": "security-scanner", "name": "Security Scanner" },
          { "id": "code-formatter", "name": "Code Formatter" },
          { "id": "test-generator", "name": "Test Generator" }
        ]
      }
    }
    ```
  </Accordion>
  <Accordion title="3. دسترسی‌های ابزاری متفاوتی پیکربندی کنید">
    ```json
    {
      "agents": {
        "list": [
          { "id": "reviewer", "tools": { "allow": ["read", "exec"] } },
          { "id": "fixer", "tools": { "allow": ["read", "write", "edit", "exec"] } }
        ]
      }
    }
    ```

    `reviewer` فقط‌خواندنی است. `fixer` می‌تواند بخواند و بنویسد.

  </Accordion>
  <Accordion title="4. عملکرد را پایش کنید">
    با تعداد زیادی عامل، `"strategy": "parallel"` (پیش‌فرض) را ترجیح دهید، گروه‌های پخش را به تعداد انگشت‌شماری عامل محدود کنید و برای عامل‌های ساده‌تر از مدل‌های سریع‌تر استفاده کنید.
  </Accordion>
  <Accordion title="5. خرابی‌ها ایزوله می‌مانند">
    عامل‌ها مستقل از یکدیگر خراب می‌شوند. خطای یک عامل ثبت می‌شود (`Broadcast agent <id> failed: ...`) و عامل‌های دیگر را مسدود نمی‌کند.
  </Accordion>
</AccordionGroup>

## سازگاری

### ارائه‌دهندگان

گروه‌های پخش درحال‌حاضر فقط برای WhatsApp (کانال وب) پیاده‌سازی شده‌اند. کانال‌های دیگر پیکربندی `broadcast` را نادیده می‌گیرند.

### مسیریابی

گروه‌های پخش در کنار مسیریابی موجود کار می‌کنند:

```json
{
  "bindings": [
    {
      "match": { "channel": "whatsapp", "peer": { "kind": "group", "id": "GROUP_A" } },
      "agentId": "alfred"
    }
  ],
  "broadcast": {
    "GROUP_B": ["agent1", "agent2"]
  }
}
```

- `GROUP_A`: فقط alfred پاسخ می‌دهد (مسیریابی عادی).
- `GROUP_B`: agent1 و agent2 پاسخ می‌دهند (پخش).

<Note>
**اولویت:** `broadcast` بر اتصال‌های مسیر عادی اولویت دارد. اتصال‌های ACP پیکربندی‌شده (`bindings[].type="acp"`) انحصاری هستند: هنگامی که یکی از آن‌ها تطبیق پیدا کند، OpenClaw به‌جای پخش توزیعی، پیام را به نشست ACP پیکربندی‌شده اعزام می‌کند.
</Note>

## عیب‌یابی

<AccordionGroup>
  <Accordion title="عامل‌ها پاسخ نمی‌دهند">
    **بررسی کنید:**

    1. شناسه‌های عامل در `agents.entries` وجود دارند (اعتبارسنجی پیکربندی شناسه‌های ناشناخته را رد می‌کند).
    2. قالب شناسهٔ همتا صحیح است (JID گروه مانند `120363403215116621@g.us`، یا E.164 مانند `+15551234567` برای پیام‌های مستقیم).
    3. پیام از کنترل‌های عادی عبور کرده است (قواعد منشن/فعال‌سازی همچنان اعمال می‌شوند).

    **اشکال‌زدایی:**

    ```bash
    openclaw logs --follow | grep -i broadcast
    ```

    یک توزیع موفق، `Broadcasting message to <n> agents (<strategy>)` را ثبت می‌کند.

  </Accordion>
  <Accordion title="فقط یک عامل پاسخ می‌دهد">
    **علت:** ممکن است شناسهٔ همتا در اتصال‌های مسیر عادی باشد، اما در `broadcast` نباشد، یا ممکن است با یک اتصال ACP انحصاری پیکربندی‌شده تطبیق پیدا کند.

    **راه‌حل:** همتاهای متصل به مسیر عادی را به پیکربندی پخش اضافه کنید؛ یا اگر پخش توزیعی موردنظر است، اتصال ACP پیکربندی‌شده را حذف یا تغییر دهید.

  </Accordion>
  <Accordion title="مشکلات عملکرد">
    اگر با تعداد زیادی عامل عملکرد کند است: تعداد عامل‌ها در هر گروه را کاهش دهید، از مدل‌های سبک‌تر استفاده کنید و زمان راه‌اندازی سندباکس را بررسی کنید.
  </Accordion>
</AccordionGroup>

## مثال‌ها

<AccordionGroup>
  <Accordion title="مثال 1: تیم بازبینی کد">
    ```json
    {
      "broadcast": {
        "strategy": "parallel",
        "120363403215116621@g.us": [
          "code-formatter",
          "security-scanner",
          "test-coverage",
          "docs-checker"
        ]
      },
      "agents": {
        "list": [
          {
            "id": "code-formatter",
            "workspace": "~/agents/formatter",
            "tools": { "allow": ["read", "write"] }
          },
          {
            "id": "security-scanner",
            "workspace": "~/agents/security",
            "tools": { "allow": ["read", "exec"] }
          },
          {
            "id": "test-coverage",
            "workspace": "~/agents/testing",
            "tools": { "allow": ["read", "exec"] }
          },
          { "id": "docs-checker", "workspace": "~/agents/docs", "tools": { "allow": ["read"] } }
        ]
      }
    }
    ```

    یک قطعه‌کد در گروه چهار پاسخ ایجاد می‌کند: اصلاحات قالب‌بندی، یک یافتهٔ امنیتی، یک شکاف پوشش و یک ایراد جزئی مستندات.

  </Accordion>
  <Accordion title="مثال 2: پایپ‌لاین چندزبانه">
    ```json
    {
      "broadcast": {
        "strategy": "sequential",
        "+15555550123": ["detect-language", "translator-en", "translator-de"]
      },
      "agents": {
        "list": [
          { "id": "detect-language", "workspace": "~/agents/lang-detect" },
          { "id": "translator-en", "workspace": "~/agents/translate-en" },
          { "id": "translator-de", "workspace": "~/agents/translate-de" }
        ]
      }
    }
    ```
  </Accordion>
</AccordionGroup>

## مرجع API

### طرح‌وارهٔ پیکربندی

```typescript
interface OpenClawConfig {
  broadcast?: {
    strategy?: "parallel" | "sequential";
    [peerId: string]: string[];
  };
}
```

### فیلدها

<ParamField path="strategy" type='"parallel" | "sequential"' default='"parallel"'>
  نحوهٔ پردازش عامل‌ها. `parallel` همهٔ عامل‌ها را به‌طور هم‌زمان اجرا می‌کند؛ `sequential` آن‌ها را به‌ترتیب آرایه اجرا می‌کند.
</ParamField>
<ParamField path="[peerId]" type="string[]">
  JID گروه WhatsApp یا شماره‌تلفن E.164. مقدار، آرایه‌ای از شناسه‌های عامل است که همگی باید پیام‌های آن همتا را پردازش کنند.
</ParamField>

## محدودیت‌ها

1. **حداکثر عامل‌ها:** محدودیت سختی وجود ندارد، اما تعداد زیاد عامل‌ها (10+) می‌تواند باعث کندی شود.
2. **زمینهٔ مشترک:** عامل‌ها پاسخ‌های یکدیگر را نمی‌بینند (طبق طراحی).
3. **ترتیب پیام‌ها:** پاسخ‌های موازی ممکن است با هر ترتیبی برسند.
4. **محدودیت‌های نرخ:** همهٔ پاسخ‌ها از یک حساب WhatsApp ارسال می‌شوند، بنابراین پاسخ هر عامل در همان محدودیت‌های نرخ WhatsApp محاسبه می‌شود.

## مرتبط

- [مسیریابی کانال](/fa/channels/channel-routing)
- [گروه‌ها](/fa/channels/groups)
- [ابزارهای سندباکس چندعاملی](/fa/tools/multi-agent-sandbox-tools)
- [جفت‌سازی](/fa/channels/pairing)
- [مدیریت نشست](/fa/concepts/session)
