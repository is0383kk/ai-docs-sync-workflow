---
read_when:
    - می‌خواهید فایل‌های گردش‌کار `.prose` را اجرا کنید یا بنویسید
    - می‌خواهید Plugin ‏OpenProse را فعال کنید
    - باید درک کنید که OpenProse چگونه به اجزای بنیادی OpenClaw نگاشت می‌شود
sidebarTitle: OpenProse
summary: OpenProse یک قالب گردش‌کار مبتنی بر Markdown برای نشست‌های هوش مصنوعی چندعاملی است. در OpenClaw، به‌صورت یک Plugin همراه با فرمان اسلش `/prose` و یک بسته Skills ارائه می‌شود.
title: OpenProse
x-i18n:
    generated_at: "2026-07-27T14:31:51Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8b04eb23bf827fbec6db11c1e95993e7f6c617451c5f4fda771ad078674c12bc
    source_path: prose.md
    workflow: 16
---

OpenProse قالبی قابل‌حمل و مبتنی بر Markdown برای هماهنگ‌سازی نشست‌های هوش مصنوعی
است. در OpenClaw، به‌صورت Plugin عرضه می‌شود که یک بسته مهارت OpenProse
و فرمان اسلش `/prose` را نصب می‌کند. برنامه‌ها در فایل‌های `.prose` قرار می‌گیرند و می‌توانند
چندین زیرعامل را با جریان کنترل صریح ایجاد کنند.

<CardGroup cols={3}>
  <Card title="نصب" icon="download" href="#install">
    Plugin ‏OpenProse را فعال و Gateway را راه‌اندازی مجدد کنید.
  </Card>
  <Card title="اجرای برنامه" icon="play" href="#slash-command">
    برای اجرای یک فایل `.prose` یا برنامه راه‌دور از `/prose run` استفاده کنید.
  </Card>
  <Card title="نوشتن برنامه‌ها" icon="pencil" href="#example-parallel-research-and-synthesis">
    گردش‌کارهای چندعاملی را با گام‌های موازی و ترتیبی بنویسید.
  </Card>
</CardGroup>

## نصب

<Steps>
  <Step title="فعال‌کردن Plugin">
    ‏OpenProse همراه محصول ارائه می‌شود، اما به‌طور پیش‌فرض غیرفعال است. آن را فعال کنید:

    ```bash
    openclaw plugins enable open-prose
    ```

  </Step>
  <Step title="راه‌اندازی مجدد Gateway">
    ```bash
    openclaw gateway restart
    ```
  </Step>
  <Step title="تأیید">
    ```bash
    openclaw plugins list | grep prose
    ```

    باید `open-prose` را به‌صورت فعال ببینید. اکنون فرمان مهارت `/prose`
    در گفت‌وگو در دسترس است.

  </Step>
</Steps>

از یک نسخه دریافت‌شده مخزن می‌توانید Plugin را مستقیماً نصب کنید:
`openclaw plugins install ./extensions/open-prose`

## فرمان اسلش

‏OpenProse، فرمان `/prose` را به‌عنوان فرمان مهارتی قابل‌فراخوانی توسط کاربر ثبت می‌کند:

```text
/prose help
/prose run <file.prose>
/prose run <handle/slug>
/prose run <https://example.com/file.prose>
/prose compile <file.prose>
/prose examples
/prose update
```

‏`/prose run <handle/slug>` به `https://p.prose.md/<handle>/<slug>` تفکیک می‌شود.
نشانی‌های URL مستقیم با استفاده از ابزار `web_fetch` بدون تغییر دریافت می‌شوند.

اجراهای راه‌دور سطح‌بالا صریح هستند. واردسازی‌های راه‌دور درون یک برنامه `.prose`
وابستگی‌های کد تعدی‌شونده‌اند: پیش از آنکه OpenProse هر مقصد راه‌دور `use` را دریافت کند،
فهرست واردسازی تفکیک‌شده را نمایش می‌دهد و از اپراتور می‌خواهد برای آن اجرا دقیقاً
`approve remote prose imports` را پاسخ دهد.

## قابلیت‌ها

- پژوهش و ترکیب چندعاملی با موازی‌سازی صریح.
- گردش‌کارهای تکرارپذیر و ایمن از نظر تأیید (بازبینی کد، تریاژ رخداد، پایپ‌لاین‌های محتوا).
- برنامه‌های قابل‌استفاده مجدد `.prose` که می‌توانید در زمان‌های اجرای عامل پشتیبانی‌شده اجرا کنید.

## مثال: پژوهش و ترکیب موازی

```prose
# پژوهش و ترکیب با دو عامل که به‌صورت موازی اجرا می‌شوند.

input topic: "چه چیزی را باید پژوهش کنیم؟"

agent researcher:
  model: sonnet
  prompt: "به‌طور کامل پژوهش می‌کنید و منابع را ذکر می‌کنید."

agent writer:
  model: opus
  prompt: "خلاصه‌ای مختصر می‌نویسید."

parallel:
  findings = session: researcher
    prompt: "درباره {topic} پژوهش کنید."
  draft = session: writer
    prompt: "{topic} را خلاصه کنید."

session "یافته‌ها و پیش‌نویس را در یک پاسخ نهایی ادغام کنید."
  context: { findings, draft }
```

## نگاشت زمان اجرای OpenClaw

برنامه‌های OpenProse به اجزای بنیادین OpenClaw نگاشت می‌شوند:

| مفهوم OpenProse           | ابزار OpenClaw                                    |
| ------------------------- | ----------------------------------------------- |
| ایجاد نشست / ابزار Task   | `sessions_spawn`                                |
| خواندن / نوشتن فایل       | `read` / `write`                                |
| دریافت از وب              | `web_fetch` (`exec` + curl هنگامی که POST لازم است) |

<Warning>
  اگر فهرست مجاز ابزارهای شما `sessions_spawn`، `read`، `write` یا
  `web_fetch` را مسدود کند، برنامه‌های OpenProse ناموفق خواهند بود. [پیکربندی فهرست مجاز
  ابزارها](/fa/gateway/config-tools) را بررسی کنید.
</Warning>

## مکان فایل‌ها

‏OpenProse وضعیت را در فضای کاری شما، زیر `.prose/` نگه می‌دارد:

```text
.prose/
├── .env                      # پیکربندی (key=value)، برای نمونه OPENPROSE_POSTGRES_URL
├── runs/
│   └── {YYYYMMDD}-{HHMMSS}-{random}/
│       ├── program.prose     # کپی برنامه در حال اجرا
│       ├── state.md          # وضعیت اجرا
│       ├── bindings/
│       ├── imports/          # اجراهای تودرتوی برنامه راه‌دور
│       └── agents/
└── agents/                   # عامل‌های ماندگار محدود به پروژه
```

عامل‌های ماندگار سطح کاربر (مشترک میان پروژه‌ها) در این مکان قرار دارند:

```text
~/.prose/agents/
```

## بک‌اندهای وضعیت

<AccordionGroup>
  <Accordion title="سامانه فایل (پیش‌فرض)">
    وضعیت در `.prose/runs/...` در فضای کاری نوشته می‌شود. هیچ وابستگی
    اضافی لازم نیست.
  </Accordion>
  <Accordion title="درون‌بافت">
    وضعیت گذرا در پنجره بافت نگه داشته می‌شود؛ آن را با `--in-context` انتخاب کنید.
    برای برنامه‌های کوچک و کوتاه‌مدت مناسب است.
  </Accordion>
  <Accordion title="sqlite (آزمایشی)">
    آن را با `--state=sqlite` انتخاب کنید. به فایل اجرایی `sqlite3` در `PATH`
    نیاز دارد (در صورت نبود آن به سامانه فایل بازمی‌گردد)؛ وضعیت در
    `.prose/runs/{id}/state.db` قرار می‌گیرد.
  </Accordion>
  <Accordion title="postgres (آزمایشی)">
    آن را با `--state=postgres` انتخاب کنید. به `psql` و یک رشته اتصال در
    `OPENPROSE_POSTGRES_URL` نیاز دارد (آن را در `.prose/.env` تنظیم کنید).

    <Warning>
      اطلاعات اعتبارسنجی Postgres وارد گزارش‌های زیرعامل می‌شود. از پایگاه داده‌ای اختصاصی
      با کمترین سطح دسترسی استفاده کنید.
    </Warning>

  </Accordion>
</AccordionGroup>

## امنیت

با فایل‌های `.prose` مانند کد رفتار کنید. پیش از اجرا، آن‌ها را همراه با واردسازی‌های راه‌دور
`use` بازبینی کنید. درخواست‌های سطح‌بالای `/prose run https://...` صریح هستند، اما
واردسازی‌های راه‌دور تعدی‌شونده پیش از دریافت یا اجرا، برای هر اجرا به تأیید
نیاز دارند. برای کنترل اثرات جانبی از فهرست‌های مجاز ابزار و دروازه‌های
تأیید OpenClaw استفاده کنید. برای گردش‌کارهای قطعی و مشروط به تأیید، با
[Lobster](/fa/tools/lobster) مقایسه کنید.

## مرتبط

<CardGroup cols={2}>
  <Card title="مرجع Skills" href="/fa/tools/skills" icon="puzzle-piece">
    نحوه بارگذاری بسته مهارت OpenProse و دروازه‌هایی که اعمال می‌شوند.
  </Card>
  <Card title="زیرعامل‌ها" href="/fa/tools/subagents" icon="users">
    لایه بومی هماهنگ‌سازی چندعاملی OpenClaw.
  </Card>
  <Card title="تبدیل متن به گفتار" href="/fa/tools/tts" icon="volume-high">
    خروجی صوتی را به گردش‌کارهای خود اضافه کنید.
  </Card>
  <Card title="فرمان‌های اسلش" href="/fa/tools/slash-commands" icon="terminal">
    همه فرمان‌های گفت‌وگوی موجود، از جمله /prose.
  </Card>
</CardGroup>

وب‌سایت رسمی: [https://www.prose.md](https://www.prose.md)
