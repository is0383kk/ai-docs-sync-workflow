---
read_when:
    - تولید یا بازبینی طرح‌های `openclaw secrets apply`
    - اشکال‌زدایی خطاهای `Invalid plan target path`
    - درک رفتار اعتبارسنجی نوع و مسیر مقصد
summary: 'قرارداد برنامه‌های `secrets apply`: اعتبارسنجی هدف، تطبیق مسیر و دامنهٔ هدف `auth-profiles.json`'
title: قرارداد طرح اعمال اسرار
x-i18n:
    generated_at: "2026-07-27T15:33:24Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 71ee8afd958646930af4db3bbad08e033ff79da48890a989d72b361abcbda3bb
    source_path: gateway/secrets-plan-contract.md
    workflow: 16
---

این صفحه قرارداد سخت‌گیرانه‌ای را تعریف می‌کند که توسط `openclaw secrets apply` اعمال می‌شود. اگر مقصدی با این قواعد مطابقت نداشته باشد، عملیات اعمال پیش از تغییر هرگونه فایل شکست می‌خورد.

## الزامات فایل طرح

`openclaw secrets apply --from <plan.json>` فایل‌های معمولی تا سقف 16 MiB (16,777,216 bytes) را می‌پذیرد. این محدودیت بر کل فایل سریال‌سازی‌شده، از جمله فاصله‌های خالی، اعمال می‌شود. دایرکتوری‌ها، FIFOها، فایل‌های دستگاه و فایل‌های بزرگ‌تر از این حد، پیش از تجزیهٔ JSON یا اعتبارسنجی مقصد رد می‌شوند.

`openclaw secrets configure --plan-out <plan.json>` همین محدودیت را پیش از ایجاد فایل، بر خروجی سریال‌سازی‌شدهٔ UTF-8 اعمال می‌کند. طرح‌های دست‌نویس و مولدهای خارجی طرح نیز باید فایل سریال‌سازی‌شده را در این محدوده نگه دارند.

## ساختار فایل طرح

`openclaw secrets apply --from <plan.json>` انتظار یک آرایهٔ `targets` از مقصدهای طرح را دارد:

```json5
{
  version: 1,
  protocolVersion: 1,
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.openai.apiKey",
      pathSegments: ["models", "providers", "openai", "apiKey"],
      providerId: "openai",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
    {
      type: "auth-profiles.api_key.key",
      path: "profiles.openai:default.key",
      pathSegments: ["profiles", "openai:default", "key"],
      agentId: "main",
      ref: { source: "env", provider: "default", id: "OPENAI_API_KEY" },
    },
  ],
}
```

`openclaw secrets configure` طرح‌ها را با این ساختار تولید می‌کند. همچنین می‌توانید یکی را به‌صورت دستی بنویسید یا ویرایش کنید.

## درج یا به‌روزرسانی و حذف ارائه‌دهندگان

طرح‌ها می‌توانند دو فیلد اختیاری سطح‌بالا را نیز شامل شوند که نگاشت `secrets.providers` را همراه با نوشتن‌های هر مقصد تغییر می‌دهند:

- `providerUpserts` -- شیئی که کلیدهای آن نام‌های مستعار ارائه‌دهندگان هستند. هر مقدار یک تعریف ارائه‌دهنده است (همان ساختاری که در `secrets.providers.<alias>` در `openclaw.json` پذیرفته می‌شود، برای مثال یک ارائه‌دهندهٔ `exec` یا `file`).
- `providerDeletes` -- آرایه‌ای از نام‌های مستعار ارائه‌دهندگان برای حذف.

`providerUpserts` پیش از `targets` اجرا می‌شود؛ بنابراین یک `target.ref.provider` می‌تواند به نام مستعار ارائه‌دهنده‌ای ارجاع دهد که همان طرح آن را در `providerUpserts` معرفی می‌کند. بدون این ترتیب، طرح‌هایی که به نام مستعاری ارجاع می‌دهند که هنوز در `openclaw.json` پیکربندی نشده است، با `provider "<alias>" is not configured` شکست می‌خورند.

```json5
{
  version: 1,
  protocolVersion: 1,
  providerUpserts: {
    onepassword_anthropic: {
      source: "exec",
      command: "/usr/bin/op",
      args: ["read", "op://Vault/Anthropic/credential"],
    },
  },
  providerDeletes: ["legacy_unused_alias"],
  targets: [
    {
      type: "models.providers.apiKey",
      path: "models.providers.anthropic.apiKey",
      pathSegments: ["models", "providers", "anthropic", "apiKey"],
      providerId: "anthropic",
      ref: { source: "exec", provider: "onepassword_anthropic", id: "credential" },
    },
  ],
}
```

ارائه‌دهندگان اجرایی معرفی‌شده از طریق `providerUpserts` همچنان مشمول قواعد رضایت اجرای دستور در [رفتار رضایت ارائه‌دهندهٔ اجرایی](#exec-provider-consent-behavior) هستند: طرح‌های حاوی ارائه‌دهندگان اجرایی در حالت نوشتن به `--allow-exec` نیاز دارند.

## دامنهٔ مقصدهای پشتیبانی‌شده

مقصدهای طرح برای مسیرهای اعتبارنامهٔ پشتیبانی‌شده در [سطح اعتبارنامهٔ SecretRef](/fa/reference/secretref-credential-surface) پذیرفته می‌شوند.

## رفتار نوع مقصد

`target.type` باید یک نوع مقصد شناخته‌شده باشد و `target.path` نرمال‌شده باید با الگوی مسیر ثبت‌شدهٔ آن نوع مطابقت داشته باشد.

برخی انواع مقصد، علاوه بر نام نوع معیار خود، برای طرح‌های موجود یک نام مستعار سازگاری را به‌عنوان `target.type` می‌پذیرند:

| نوع معیار                             | نام مستعار پذیرفته‌شده                         |
| ------------------------------------ | ----------------------------------------------- |
| `models.providers.apiKey`            | `models.providers.*.apiKey`                     |
| `skills.entries.apiKey`              | `skills.entries.*.apiKey`                       |
| `channels.googlechat.serviceAccount` | `channels.googlechat.accounts.*.serviceAccount` |

## قواعد اعتبارسنجی مسیر

هر مقصد با تمام موارد زیر اعتبارسنجی می‌شود:

- `type` باید یک نوع مقصد شناخته‌شده باشد.
- `path` باید یک مسیر نقطه‌ای غیرخالی باشد.
- `pathSegments` را می‌توان حذف کرد. اگر ارائه شود، باید دقیقاً به همان مسیر `path` نرمال‌سازی شود.
- بخش‌های ممنوع رد می‌شوند: `__proto__`، `prototype`، `constructor`.
- مسیر نرمال‌شده باید با الگوی مسیر ثبت‌شده برای نوع مقصد مطابقت داشته باشد.
- اگر `providerId` یا `accountId` تنظیم شده باشد، باید با شناسهٔ کدگذاری‌شده در مسیر مطابقت داشته باشد.
- مقصدهای `auth-profiles.json` به `agentId` نیاز دارند.
- هنگام ایجاد یک نگاشت `auth-profiles.json` جدید، `authProfileProvider` را درج کنید.

## رفتار در صورت شکست

اگر اعتبارسنجی مقصدی شکست بخورد، عملیات اعمال با خطایی مانند زیر پایان می‌یابد:

```text
مسیر مقصد طرح برای models.providers.apiKey نامعتبر است: models.providers.openai.baseUrl
```

برای یک طرح نامعتبر هیچ نوشتنی ثبت نمی‌شود: تفکیک مقصد و اعتبارسنجی مسیر پیش از دست‌کاری هرگونه فایل اجرا می‌شوند. به‌طور جداگانه، هنگامی که نوشتن یک طرح معتبر آغاز می‌شود، عملیات اعمال ابتدا از هر فایل دست‌کاری‌شده یک اسنپ‌شات می‌گیرد و اگر نوشتنی در ادامهٔ همان اجرا شکست بخورد، آن اسنپ‌شات‌ها را بازیابی می‌کند؛ بنابراین نوشتن جزئی هرگز وضعیت پیکربندی، نمایهٔ احراز هویت یا محیط را ناهماهنگ باقی نمی‌گذارد.

## رفتار رضایت ارائه‌دهندهٔ اجرایی

- `--dry-run` به‌طور پیش‌فرض بررسی‌های SecretRef اجرایی را نادیده می‌گیرد.
- طرح‌های حاوی SecretRefها/ارائه‌دهندگان اجرایی در حالت نوشتن رد می‌شوند، مگر اینکه `--allow-exec` تنظیم شده باشد.
- هنگام اعتبارسنجی/اعمال طرح‌های حاوی اجرای دستور، `--allow-exec` را هم در فرمان اجرای آزمایشی و هم در فرمان نوشتن وارد کنید.

## نکات دامنهٔ زمان اجرا و ممیزی

- ورودی‌های فقط‌ارجاعی `auth-profiles.json` ‏(`keyRef`/`tokenRef`) در تفکیک اعتبارنامهٔ زمان اجرا و پوشش ممیزی گنجانده می‌شوند.
- `secrets apply` مقصدهای پشتیبانی‌شدهٔ `openclaw.json`، مقصدهای پشتیبانی‌شدهٔ `auth-profiles.json` و سه مرحلهٔ پاک‌سازی اختیاری را می‌نویسد که هرکدام به‌طور پیش‌فرض فعال‌اند: `scrubEnv` (مقادیر متن سادهٔ مهاجرت‌یافته را از فایل‌های `.env` در دایرکتوری‌های وضعیت مؤثر و پیکربندی فعال حذف می‌کند)، `scrubAuthProfilesForProviderTargets` (باقی‌مانده‌های متن ساده/ارجاع استفاده‌نشده را در `auth-profiles.json` برای ارائه‌دهندگانی که طرح به‌تازگی مهاجرت داده است پاک می‌کند) و `scrubLegacyAuthJson` (ورودی‌های مهاجرت‌یافتهٔ `api_key` را از مخزن‌های قدیمی `auth.json` حذف می‌کند). برای رد کردن هر مرحله، هرکدام از `options.scrubEnv`، `options.scrubAuthProfilesForProviderTargets` یا `options.scrubLegacyAuthJson` را در طرح روی `false` تنظیم کنید.

## بررسی‌های اپراتور

```bash
# اعتبارسنجی طرح بدون نوشتن
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run

# سپس اعمال واقعی
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json

# برای طرح‌های حاوی اجرای دستور، در هر دو حالت صریحاً رضایت دهید
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from /tmp/openclaw-secrets-plan.json --allow-exec
```

اگر عملیات اعمال با پیام مسیر مقصد نامعتبر شکست خورد، طرح را با `openclaw secrets configure` دوباره تولید کنید یا مسیر مقصد را به یکی از ساختارهای پشتیبانی‌شدهٔ بالا اصلاح کنید.

## مستندات مرتبط

- [مدیریت اسرار](/fa/gateway/secrets)
- [CLI ‏`secrets`](/fa/cli/secrets)
- [سطح اعتبارنامهٔ SecretRef](/fa/reference/secretref-credential-surface)
- [مرجع پیکربندی](/fa/gateway/configuration-reference)
