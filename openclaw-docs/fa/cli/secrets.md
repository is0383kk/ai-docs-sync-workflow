---
read_when:
    - حل مجدد ارجاع‌های محرمانه در زمان اجرا
    - ممیزی بقایای متن ساده و ارجاع‌های حل‌نشده
    - پیکربندی SecretRefها و اعمال تغییرات پاک‌سازی یک‌طرفه
summary: مرجع CLI برای `openclaw secrets` (بارگذاری مجدد، ممیزی، پیکربندی، اعمال)
title: اسرار
x-i18n:
    generated_at: "2026-07-27T16:23:36Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 61f6f81e358ca2e6a97ac9498186b32f7a74d16052d226c398dad0030d47211e
    source_path: cli/secrets.md
    workflow: 16
---

# `openclaw secrets`

SecretRefها را مدیریت کنید و تصویر لحظه‌ای فعال زمان اجرا را سالم نگه دارید.

| فرمان     | نقش                                                                                                                                                                                         |
| ----------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `reload`    | RPC مربوط به Gateway (`secrets.reload`): ارجاع‌ها را دوباره تفکیک می‌کند و تصویر لحظه‌ای زمان اجرا با آگاهی از مالک را به‌صورت اتمی منتشر می‌کند (بدون نوشتن پیکربندی)؛ خرابی‌های واجد شرایط مالک ممکن است به‌شکل هشدار سرد یا کهنه منتشر شوند |
| `audit`     | پویش فقط‌خواندنی مخازن پیکربندی/احراز هویت/مدل تولیدشده و بقایای قدیمی برای متن ساده، ارجاع‌های تفکیک‌نشده و انحراف تقدم (ارجاع‌های اجرایی نادیده گرفته می‌شوند، مگر با `--allow-exec`)                      |
| `configure` | برنامه‌ریز تعاملی برای راه‌اندازی ارائه‌دهنده، نگاشت مقصد و بررسی پیش‌اجرایی (به TTY نیاز دارد)                                                                                                       |
| `apply`     | یک برنامه ذخیره‌شده را اجرا می‌کند (`--dry-run` فقط اعتبارسنجی می‌کند و به‌طور پیش‌فرض بررسی‌های اجرایی را نادیده می‌گیرد؛ حالت نوشتن، برنامه‌های شامل اجرا را بدون `--allow-exec` رد می‌کند)، سپس بقایای متن ساده هدف‌گذاری‌شده را پاک می‌کند |

چرخه پیشنهادی برای اپراتور:

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets audit --check
openclaw secrets reload
```

اگر برنامه شما شامل SecretRefها/ارائه‌دهندگان `exec` است، در هر دو فرمان `apply` مربوط به اجرای آزمایشی و نوشتن، `--allow-exec` را ارسال کنید.

کدهای خروج برای CI/دروازه‌ها:

- `audit --check` در صورت وجود یافته، `1` را برمی‌گرداند.
- ارجاع‌های تفکیک‌نشده، `2` را برمی‌گردانند (صرف‌نظر از `--check`).

مرتبط: [مدیریت اسرار](/fa/gateway/secrets) · [سطح اعتبارنامه SecretRef](/fa/reference/secretref-credential-surface) · [امنیت](/fa/gateway/security)

## بارگذاری مجدد تصویر لحظه‌ای زمان اجرا

```bash
openclaw secrets reload
openclaw secrets reload --json
openclaw secrets reload --url ws://127.0.0.1:18789 --token <token>
```

از روش RPC مربوط به Gateway با نام `secrets.reload` استفاده می‌کند. مالکان سالم به‌طور مستقل تازه‌سازی می‌شوند. مالکان واجد شرایطی که با شکست مواجه شده‌اند، تنها زمانی که هویت ارجاع‌ها، تعریف ارائه‌دهندگان و قرارداد کامل و غیرمحرمانه مالک آن‌ها بدون تغییر باشد، کهنه می‌شوند؛ شکست‌های جدید یا تغییریافته سرد می‌شوند. این فعال‌سازی تنزل‌یافته موفق می‌شود و `warningCount` را گزارش می‌کند. شکست‌های سخت‌گیرانه یا نگاشت‌نشده خطا برمی‌گردانند و تصویر لحظه‌ای فعال قبلی را حفظ می‌کنند.

گزینه‌ها: `--url <url>`، `--token <token>`، `--timeout <ms>`، `--json`.

## ممیزی

وضعیت OpenClaw را برای موارد زیر پویش می‌کند:

- ذخیره‌سازی اسرار به‌صورت متن ساده
- ارجاع‌های تفکیک‌نشده
- انحراف تقدم (اعتبارنامه‌های `auth-profiles.json` که ارجاع‌های `openclaw.json` را تحت‌الشعاع قرار می‌دهند)
- بقایای `agents/*/agent/models.json` تولیدشده (مقادیر `apiKey` ارائه‌دهنده و سرآیندهای حساس ارائه‌دهنده)
- بقایای قدیمی (ورودی‌های قدیمی مخزن احراز هویت، یادآورهای OAuth)

پویش `.env` دایرکتوری مؤثر وضعیت و دایرکتوری حاوی پیکربندی فعال را پوشش می‌دهد. اگر هر دو مسیر به یک فایل اشاره کنند، آن فایل یک‌بار پویش می‌شود.

تشخیص سرآیند حساس ارائه‌دهنده بر اساس روش ابتکاری نام انجام می‌شود: سرآیندهایی را علامت‌گذاری می‌کند که نامشان با قطعه‌های رایج احراز هویت/اعتبارنامه مطابقت دارد (`authorization`، `x-api-key`، `token`، `secret`، `password`، `credential`).

```bash
openclaw secrets audit
openclaw secrets audit --check
openclaw secrets audit --json
openclaw secrets audit --allow-exec
```

ساختار گزارش:

- `status`: `clean | findings | unresolved`
- `resolution`: `refsChecked`، `skippedExecRefs`، `resolvabilityComplete`
- `summary`: `plaintextCount`، `unresolvedRefCount`، `shadowedRefCount`، `legacyResidueCount`
- کدهای یافته: `PLAINTEXT_FOUND`، `REF_UNRESOLVED`، `REF_SHADOWED`، `LEGACY_RESIDUE`

## پیکربندی (دستیار تعاملی)

تغییرات ارائه‌دهنده و SecretRef را به‌صورت تعاملی بسازید، بررسی پیش‌اجرایی را انجام دهید و در صورت تمایل اعمال کنید:

```bash
openclaw secrets configure
openclaw secrets configure --plan-out /tmp/openclaw-secrets-plan.json
openclaw secrets configure --apply --yes
openclaw secrets configure --providers-only
openclaw secrets configure --skip-provider-setup
openclaw secrets configure --agent ops
openclaw secrets configure --json
```

جریان: ابتدا راه‌اندازی ارائه‌دهنده (افزودن/ویرایش/حذف نام‌های مستعار `secrets.providers`)، سپس نگاشت اعتبارنامه (انتخاب فیلدها، تخصیص ارجاع‌های `{source, provider, id}`) و پس از آن بررسی پیش‌اجرایی و اعمال اختیاری.

پرچم‌ها:

- `--providers-only`: فقط `secrets.providers` را پیکربندی می‌کند و نگاشت اعتبارنامه را نادیده می‌گیرد
- `--skip-provider-setup`: راه‌اندازی ارائه‌دهنده را نادیده می‌گیرد و اعتبارنامه‌ها را به ارائه‌دهندگان موجود نگاشت می‌کند
- `--agent <id>`: کشف مقصد و نوشتن `auth-profiles.json` را به مخزن یک عامل محدود می‌کند
- `--allow-exec`: بررسی‌های اجرایی SecretRef را هنگام پیش‌اجرایی/اعمال مجاز می‌کند (ممکن است فرمان‌های ارائه‌دهنده را اجرا کند)

`--providers-only` و `--skip-provider-setup` را نمی‌توان با هم ترکیب کرد.

نکته‌ها:

- به یک TTY تعاملی نیاز دارد.
- فیلدهای حاوی اسرار در `openclaw.json` به‌علاوه `auth-profiles.json` را برای محدوده عامل انتخاب‌شده هدف می‌گیرد؛ سطح متعارف پشتیبانی‌شده: [سطح اعتبارنامه SecretRef](/fa/reference/secretref-credential-surface).
- از ایجاد نگاشت‌های جدید `auth-profiles.json` مستقیماً در جریان انتخاب‌گر پشتیبانی می‌کند.
- پیش از اعمال، تفکیک پیش‌اجرایی را انجام می‌دهد.
- در برنامه‌های تولیدشده، گزینه‌های پاک‌سازی به‌طور پیش‌فرض فعال‌اند (`scrubEnv`، `scrubAuthProfilesForProviderTargets`، `scrubLegacyAuthJson`). اعمال برای مقادیر متن ساده پاک‌شده یک‌طرفه است.
- `--plan-out` از ایجاد برنامه‌ای که فرم سریال‌شده UTF-8 آن از 16 MiB (16,777,216 bytes) بیشتر باشد خودداری می‌کند، مطابق با محدودیت ورودی `apply --from`.
- بدون `--apply`، CLI همچنان پس از بررسی پیش‌اجرایی، `Apply this plan now?` را درخواست می‌کند.
- با `--apply` (و بدون `--yes`)، CLI یک تأیید اضافی برای مهاجرت برگشت‌ناپذیر درخواست می‌کند.
- `--json` برنامه و گزارش پیش‌اجرایی را چاپ می‌کند، اما همچنان به یک TTY تعاملی نیاز دارد.

### ایمنی ارائه‌دهنده اجرایی

نصب‌های Homebrew اغلب فایل‌های اجرایی پیوند نمادی را در `/opt/homebrew/bin/*` ارائه می‌کنند. `allowSymlinkCommand: true` را فقط در صورت نیاز برای مسیرهای قابل‌اعتماد مدیر بسته تنظیم کنید و آن را با `trustedDirs` همراه سازید (برای مثال `["/opt/homebrew"]`). در Windows، اگر راستی‌آزمایی ACL برای مسیر ارائه‌دهنده در دسترس نباشد، OpenClaw با رویکرد بسته شکست می‌خورد؛ فقط برای مسیرهای قابل‌اعتماد، `allowInsecurePath: true` را روی آن ارائه‌دهنده تنظیم کنید تا بررسی امنیتی مسیر دور زده شود.

## اعمال یک برنامه ذخیره‌شده

```bash
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --json
```

`--dry-run` بررسی پیش‌اجرایی را بدون نوشتن فایل‌ها اعتبارسنجی می‌کند؛ بررسی‌های اجرایی SecretRef به‌طور پیش‌فرض در اجرای آزمایشی نادیده گرفته می‌شوند. حالت نوشتن، برنامه‌های حاوی SecretRefها/ارائه‌دهندگان اجرایی را بدون `--allow-exec` رد می‌کند. برای فعال‌سازی بررسی/اجرای ارائه‌دهنده اجرایی در هر یک از حالت‌ها، از `--allow-exec` استفاده کنید.

`--from` باید به یک فایل معمولی با اندازه حداکثر 16 MiB (16,777,216 bytes) اشاره کند. محدودیت بایت بر کل فایل سریال‌شده، از جمله فاصله‌های خالی، اعمال می‌شود.

مواردی که `apply` ممکن است به‌روزرسانی کند:

- `openclaw.json` (مقصدهای SecretRef + درج یا به‌روزرسانی/حذف ارائه‌دهنده)
- `auth-profiles.json` (پاک‌سازی مقصد ارائه‌دهنده)
- بقایای قدیمی `auth.json`
- فایل‌های `.env` در دایرکتوری‌های وضعیت مؤثر و پیکربندی فعال، برای کلیدهای شناخته‌شده اسرار که مقادیرشان مهاجرت داده شده‌اند

جزئیات قرارداد برنامه (مسیرهای مقصد مجاز، قواعد اعتبارسنجی، معناشناسی شکست): [قرارداد برنامه اعمال اسرار](/fa/gateway/secrets-plan-contract).

### چرا نسخه پشتیبان بازگردانی وجود ندارد

`secrets apply` عمداً نسخه‌های پشتیبان بازگردانی حاوی مقادیر قدیمی متن ساده نمی‌نویسد. ایمنی از بررسی پیش‌اجرایی سخت‌گیرانه به‌همراه اعمال تقریباً اتمی و بازیابی درون‌حافظه‌ای با حداکثر تلاش در صورت شکست حاصل می‌شود.

## مثال

```bash
openclaw secrets audit --check
openclaw secrets configure
openclaw secrets audit --check
```

اگر `audit --check` همچنان یافته‌های متن ساده را گزارش می‌کند، مسیرهای مقصد گزارش‌شده باقی‌مانده را به‌روزرسانی و ممیزی را دوباره اجرا کنید.

## مرتبط

- [مرجع CLI](/fa/cli)
- [مدیریت اسرار](/fa/gateway/secrets)
- [SecretRefهای Vault](/fa/plugins/vault)
