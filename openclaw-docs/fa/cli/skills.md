---
read_when:
    - می‌خواهید ببینید کدام Skills در دسترس و آمادهٔ اجرا هستند
    - می‌خواهید در ClawHub جست‌وجو کنید یا Skills را از ClawHub، Git یا پوشه‌های محلی نصب کنید
    - می‌خواهید یک Skill در ClawHub را با ClawHub تأیید کنید
    - می‌خواهید نبود فایل‌های اجرایی/متغیرهای محیطی/پیکربندی را برای Skills اشکال‌زدایی کنید
summary: مرجع CLI برای `openclaw skills` (جست‌وجو/نصب/به‌روزرسانی/تأیید/فهرست/اطلاعات/بررسی/کارگاه)
title: Skills
x-i18n:
    generated_at: "2026-07-27T15:05:20Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 3eafd40704b666e6be185aa8148b60613c861a2899fb9b0cc3353212e8e4d678
    source_path: cli/skills.md
    workflow: 16
---

# `openclaw skills`

Skills محلی را بررسی کنید، در ClawHub جست‌وجو کنید، Skills را از ClawHub/Git/دایرکتوری‌های محلی نصب کنید، Skills موجود در ClawHub را اعتبارسنجی کنید و نصب‌های ردیابی‌شده توسط ClawHub را به‌روزرسانی کنید.

مرتبط:

- سامانه Skills: [Skills](/fa/tools/skills)
- کارگاه Skill: [کارگاه Skill](/fa/tools/skill-workshop)
- پیکربندی Skills: [پیکربندی Skills](/fa/tools/skills-config)
- نصب‌های ClawHub: [ClawHub](/fa/clawhub/cli)

## فرمان‌ها

```bash
openclaw skills search "calendar"
openclaw skills search --limit 20 --json
openclaw skills install @owner/<slug>
openclaw skills install @owner/<slug> --version <version>
openclaw skills install git:owner/repo
openclaw skills install git:owner/repo@main
openclaw skills install ./path/to/skill --as custom-name
openclaw skills install @owner/<slug> --force
openclaw skills install @owner/<slug> --force-install
openclaw skills install @owner/<slug> --acknowledge-clawhub-risk
openclaw skills install @owner/<slug> --agent <id>
openclaw skills install @owner/<slug> --global
openclaw skills update @owner/<slug>
openclaw skills update @owner/<slug> --force-install
openclaw skills update @owner/<slug> --acknowledge-clawhub-risk
openclaw skills update @owner/<slug> --global
openclaw skills update --all
openclaw skills update --all --agent <id>
openclaw skills update --all --global
openclaw skills verify @owner/<slug>
openclaw skills verify @owner/<slug> --version <version>
openclaw skills verify @owner/<slug> --tag <tag>
openclaw skills verify @owner/<slug> --card
openclaw skills verify @owner/<slug> --global
openclaw skills list
openclaw skills list --eligible
openclaw skills list --json
openclaw skills list --verbose
openclaw skills list --agent <id>
openclaw skills info <name>
openclaw skills info <name> --json
openclaw skills info <name> --agent <id>
openclaw skills check
openclaw skills check --agent <id>
openclaw skills check --json
openclaw skills workshop propose-create --name "qa-check" --description "QA checklist" --proposal ./PROPOSAL.md
openclaw skills workshop propose-update qa-check --proposal ./PROPOSAL.md
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "Not reusable"
openclaw skills workshop quarantine <proposal-id> --reason "Needs security review"
```

`search`، `update` و `verify` مستقیماً از ClawHub استفاده می‌کنند. `install @owner/<slug>` یک Skill از ClawHub نصب می‌کند، `install git:owner/repo[@ref]` یک Skill مبتنی بر Git را کلون می‌کند و `install ./path` یک دایرکتوری Skill محلی را کپی می‌کند. به‌طور پیش‌فرض، `install`، `update` و `verify` دایرکتوری `skills/` فضای کاری فعال را هدف می‌گیرند؛ با `--global`، دایرکتوری مشترک Skills مدیریت‌شده را هدف می‌گیرند. `list`/`info`/`check` همچنان Skills محلیِ قابل‌مشاهده برای فضای کاری و پیکربندی فعلی را بررسی می‌کنند.
فرمان‌های مبتنی بر فضای کاری، فضای کاری هدف را ابتدا از `--agent <id>`، سپس از دایرکتوری کاری فعلی در صورتی که داخل فضای کاری یک عامل پیکربندی‌شده باشد و پس از آن از عامل پیش‌فرض تعیین می‌کنند.

نصب‌ها از Git و دایرکتوری محلی انتظار دارند `SKILL.md` در ریشه منبع وجود داشته باشد. نامک نصب، در صورت معتبر بودن، ابتدا از `name` در frontmatter فایل `SKILL.md` و سپس از نام دایرکتوری منبع یا مخزن گرفته می‌شود؛ برای بازنویسی آن از `--as <slug>` استفاده کنید.
`--version` فقط مختص ClawHub است. نصب Skill از مشخصات بسته npm یا مسیرهای zip/آرشیو پشتیبانی نمی‌کند و `openclaw skills update` فقط نصب‌های ردیابی‌شده توسط ClawHub را به‌روزرسانی می‌کند.

نصب وابستگی‌های Skill مبتنی بر Gateway که از فرایند راه‌اندازی اولیه یا تنظیمات Skills آغاز می‌شوند، در عوض از مسیر درخواست جداگانه `skills.install` استفاده می‌کنند.

نکات:

| پرچم/رفتار                    | توضیحات                                                                                                                                                                                                                                                                       |
| -------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `search [query...]`              | پرس‌وجوی اختیاری؛ برای مرور خوراک جست‌وجوی پیش‌فرض ClawHub آن را حذف کنید.                                                                                                                                                                                                                |
| `search --limit <n>`             | تعداد نتایج بازگردانده‌شده را محدود می‌کند.                                                                                                                                                                                                                                                            |
| `install git:owner/repo[@ref]`   | یک Skill مبتنی بر Git نصب می‌کند. ارجاع‌های شاخه می‌توانند شامل اسلش باشند، مانند `git:owner/repo@feature/foo`.                                                                                                                                                                                      |
| `install ./path/to/skill`        | یک دایرکتوری محلی را نصب می‌کند که ریشه آن شامل `SKILL.md` است.                                                                                                                                                                                                                        |
| `install --as <slug>`            | نامک استنباط‌شده را برای نصب از Git و دایرکتوری محلی بازنویسی می‌کند.                                                                                                                                                                                                                 |
| `install --version <version>`    | فقط برای ارجاع‌های Skill در ClawHub اعمال می‌شود.                                                                                                                                                                                                                                               |
| `install --force`                | پوشه Skill موجود در فضای کاری را برای همان نامک بازنویسی می‌کند.                                                                                                                                                                                                                  |
| `install/update --force-install` | پیش از تکمیل اسکن ClawHub، یک Skill در انتظار و مبتنی بر GitHub را از ClawHub نصب می‌کند.                                                                                                                                                                                                   |
| `--global`                       | دایرکتوری مشترک Skills مدیریت‌شده را هدف می‌گیرد؛ نمی‌توان آن را با `--agent <id>` ترکیب کرد.                                                                                                                                                                                                  |
| `--agent <id>`                   | فضای کاری یک عامل پیکربندی‌شده را هدف می‌گیرد؛ استنباط از دایرکتوری کاری فعلی را بازنویسی می‌کند.                                                                                                                                                                                            |
| `update @owner/<slug>`           | یک Skill ردیابی‌شده را به‌روزرسانی می‌کند. برای هدف‌گیری دایرکتوری مشترک Skills مدیریت‌شده به‌جای فضای کاری، `--global` را اضافه کنید.                                                                                                                                                            |
| `update --all`                   | نصب‌های ردیابی‌شده ClawHub را در فضای کاری انتخاب‌شده یا، با `--global`، در دایرکتوری مشترک Skills مدیریت‌شده به‌روزرسانی می‌کند.                                                                                                                                                               |
| `verify @owner/<slug>`           | به‌طور پیش‌فرض پوشش JSON مربوط به `clawhub.skill.verify.v1` در ClawHub را چاپ می‌کند. پرچم `--json` وجود ندارد، زیرا JSON از قبل پیش‌فرض است. برای سازگاری، نامک‌های بدون مالک در صورتی پذیرفته می‌شوند که Skill از قبل نصب شده یا بدون ابهام باشد؛ ارجاع‌های دارای مالک از ابهام درباره ناشر جلوگیری می‌کنند. |
| منشأ `verify`              | هنگامی که ClawHub منشأ منبعِ تعیین‌شده توسط سرور را بازمی‌گرداند، JSON اعتبارسنجی نیز شامل یک `openclaw.verifiedSourceUrl` تثبیت‌شده به یک کامیت است. نشانی‌های منبع ناموجود یا خوداظهاری‌شده فقط در پوشش خام منشأ باقی می‌مانند و ارتقا داده نمی‌شوند.                                           |
| انتخابگر نسخه `verify`        | `verify` برای Skills نصب‌شده از ClawHub از `.clawhub/origin.json` استفاده می‌کند، بنابراین نسخه نصب‌شده را در برابر همان رجیستری‌ای که از آن آمده است اعتبارسنجی می‌کند. `--version` و `--tag` انتخابگر نسخه را بازنویسی می‌کنند، اما در صورت وجود فراداده مبدأ، همان رجیستری نصب‌شده را حفظ می‌کنند.                    |
| `verify --card`                  | به‌جای JSON، Markdown تولیدشده برای کارت Skill را چاپ می‌کند. وقتی ClawHub مقدار `ok: false` یا `decision: "fail"` را بازگرداند، با کد خروجی غیرصفر پایان می‌یابد؛ امضاهای بدون امضا صرفاً اطلاع‌رسان هستند، مگر اینکه سیاست ClawHub تغییر کند.                                                                             |
| اثر انگشت کارت Skill           | بسته‌های نصب‌شده ClawHub می‌توانند شامل یک `skill-card.md` تولیدشده باشند. OpenClaw اعتبارسنجی را تصمیم سرور ClawHub تلقی می‌کند و صرفاً به‌دلیل تغییر اثر انگشت بسته توسط آن کارت تولیدشده، یک Skill نصب‌شده را رد نمی‌کند.                                              |
| `check --agent <id>`             | فضای کاری عامل انتخاب‌شده را بررسی می‌کند و گزارش می‌دهد کدام Skills آماده واقعاً در پرامپت یا سطح فرمان آن عامل قابل‌مشاهده هستند.                                                                                                                                              |
| `list`                           | کنش پیش‌فرض هنگامی که هیچ زیرفرمانی ارائه نشده است.                                                                                                                                                                                                                                    |
| خروجی `list`/`info`/`check`     | خروجی رندرشده به stdout می‌رود. با `--json`، محتوای قابل‌خواندن برای ماشین برای پایپ‌ها و اسکریپت‌ها در stdout باقی می‌ماند.                                                                                                                                                                |

نصب و به‌روزرسانی Skills جامعه ClawHub، پیش از دانلود اعتماد را بررسی می‌کنند.
انتشارهای آرشیوی نسخه‌دار جامعه از فراداده اعتمادِ همان انتشار دقیق استفاده می‌کنند.
Skills مبتنی بر GitHub که از resolver استفاده می‌کنند، به resolver نصب ClawHub متکی‌اند تا پیش از بازگرداندن یک کامیت تثبیت‌شده، سیاست اسکن و نصب اجباری را اعمال کند؛ برای نصب یک Skill در انتظار و مبتنی بر GitHub پیش از تکمیل آن اسکن، از `--force-install` استفاده کنید. انتشارهای مخرب یا مسدودشده جامعه رد می‌شوند. انتشارهای پرخطر جامعه نیازمند بازبینی و `--acknowledge-clawhub-risk` هستند، هنگامی که یک فرمان غیرتعاملی باید پس از آن بازبینی ادامه یابد. ناشران رسمی Skill در ClawHub و منابع Skill همراه OpenClaw از این اعلان اعتماد به انتشار عبور می‌کنند.

## کارگاه Skill

`openclaw skills workshop` پیشنهادهای در انتظار Skills را در فضای کاری انتخاب‌شده مدیریت می‌کند.
پیشنهادها تا زمانی که اعمال نشوند، Skills فعال نیستند. برای ذخیره‌سازی پیشنهاد،
محافظت‌های فایل‌های پشتیبان، متدهای Gateway و خط‌مشی تأیید، به
[کارگاه Skills](/fa/tools/skill-workshop) مراجعه کنید.

```bash
openclaw skills workshop propose-create \
  --name "qa-check" \
  --description "چک‌لیست تکرارپذیر QA" \
  --proposal ./PROPOSAL.md
openclaw skills workshop propose-create \
  --name "qa-check" \
  --description "چک‌لیست تکرارپذیر QA" \
  --proposal-dir ./qa-check-proposal
openclaw skills workshop propose-update qa-check --proposal ./PROPOSAL.md
openclaw skills workshop list
openclaw skills workshop inspect <proposal-id>
openclaw skills workshop revise <proposal-id> --proposal ./PROPOSAL.md
openclaw skills workshop apply <proposal-id>
openclaw skills workshop reject <proposal-id> --reason "تکراری"
openclaw skills workshop quarantine <proposal-id> --reason "نیازمند بازبینی امنیتی"
```

`propose-create`، `propose-update` و `revise` همچنین `--goal <text>`
و `--evidence <text>` را می‌پذیرند تا انگیزه پیشنهاد و یادداشت‌های پشتیبان
در کنار محتوای `--proposal`/`--proposal-dir` ثبت شوند.

## مرتبط

- [مرجع CLI](/fa/cli)
- [Skills](/fa/tools/skills)
