---
read_when:
    - می‌خواهید OpenClaw کلیدهای API را از HashiCorp Vault بخواند
    - در حال راه‌اندازی SecretRefs روی یک دستگاه محلی یا سرور هستید
    - باید اعتبارنامه‌های ارائه‌دهندهٔ مدل با پشتیبانی Vault را پیکربندی کنید
summary: از Plugin همراه Vault برای برطرف‌کردن SecretRefها از HashiCorp Vault استفاده کنید
title: SecretRefهای خزانه
x-i18n:
    generated_at: "2026-07-27T15:46:07Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c1fa4895414e8cf44bb4ada191a7f7aa7b4eeda58f16be04d0c77080b7af96e3
    source_path: plugins/vault.md
    workflow: 16
---

# SecretRefهای Vault

Plugin همراه Vault به OpenClaw امکان می‌دهد SecretRefهای `exec` را هنگام راه‌اندازی Gateway و بارگذاری مجدد از
HashiCorp Vault دریافت کند. OpenClaw ارجاع‌های Vault را در پیکربندی ذخیره می‌کند، مقادیر دریافت‌شده را در اسنپ‌شات درون‌حافظه‌ای اسرار نگه می‌دارد
و کلیدهای API دریافت‌شده را دوباره در `openclaw.json` نمی‌نویسد.

اگر از قبل Vault را اجرا می‌کنید یا می‌خواهید کلیدهای ارائه‌دهندگان مدل خارج از
فایل‌های پیکربندی OpenClaw نگهداری شوند، از این قابلیت استفاده کنید. برای مدل زمان اجرای SecretRef، به
[مدیریت اسرار](/fa/gateway/secrets) مراجعه کنید.

## پیش از شروع

نیازمندی‌ها:

- OpenClaw با Plugin همراه `vault` در دسترس
- یک سرور Vault قابل دسترس
- احراز هویت Vault که بتواند یک توکن کلاینت با دسترسی خواندن مسیرهای اسراری ایجاد کند
  که OpenClaw باید دریافت کند
- محیطی که Gateway را راه‌اندازی می‌کند باید شامل `VAULT_ADDR` و یکی از این موارد باشد:
  `VAULT_TOKEN`، یا `OPENCLAW_VAULT_AUTH_METHOD=token_file` همراه با `VAULT_TOKEN_FILE`،
  یا ورود JWT/Kubernetes پیکربندی‌شده

برطرف‌کننده از طریق HTTP در Node با Vault ارتباط برقرار می‌کند. Gateway برای دریافت
SecretRefها به CLI مربوط به Vault نیازی ندارد.

پیش از اجرای فرمان‌های `openclaw vault`، Plugin همراه را فعال کنید:

```bash
openclaw plugins enable vault
```

## ذخیره کلید ارائه‌دهنده در Vault

OpenClaw به‌طور پیش‌فرض از KV v2 نصب‌شده در `secret` استفاده می‌کند که با نمونه‌های
سرور توسعه Vault مطابقت دارد. برای Vault در محیط عملیاتی، پیش از ایجاد شناسه‌های SecretRef،
`OPENCLAW_VAULT_KV_MOUNT` را روی مسیر واقعی نصب KV تنظیم کنید. با مقادیر پیش‌فرض OpenClaw، این
شناسه SecretRef:

```text
providers/openrouter/apiKey
```

این فیلد Vault را می‌خواند:

```text
secret/data/providers/openrouter -> apiKey
```

یکی از راه‌های ایجاد آن با CLI مربوط به Vault:

```bash
export OPENROUTER_API_KEY=<openrouter-api-key>
vault kv put secret/providers/openrouter apiKey="$OPENROUTER_API_KEY"
```

برای OpenClaw از توکن کلاینت با دامنه محدود استفاده کنید، نه توکن ریشه. برای چیدمان پیش‌فرض KV v2،
یک خط‌مشی حداقلی برای کلیدهای ارائه‌دهندگان مدل به این صورت است:

```hcl
path "secret/data/providers/*" {
  capabilities = ["read"]
}
```

## قابل مشاهده کردن Vault برای Gateway

برای Gateway محلی بدون کانتینر، تنظیمات Vault را در همان پوسته‌ای صادر کنید
که OpenClaw را راه‌اندازی می‌کند. روش احراز هویت پیش‌فرض، توکن کلاینت Vault را از
`VAULT_TOKEN` می‌خواند:

```bash
export VAULT_ADDR=https://vault.example.com
export VAULT_TOKEN=<vault-client-token>
```

اگر Vault Agent توکن را در فایل مقصد می‌نویسد، از احراز هویت مبتنی بر فایل توکن استفاده کنید:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=token_file
export VAULT_TOKEN_FILE=/vault/secrets/token
```

برای سرور Vault که با CA خصوصی امضا شده است، یا آن CA را در مخزن اعتماد میزبان نصب کنید
و اعتماد سیستمی Node را فعال کنید:

```bash
export NODE_USE_SYSTEM_CA=1
```

یا بسته PEM را مستقیماً ارائه کنید:

```bash
export NODE_EXTRA_CA_CERTS=/path/to/vault-ca.pem
```

این متغیرها باید هنگام راه‌اندازی OpenClaw موجود باشند. Plugin مربوط به Vault آن‌ها را به
فرایند برطرف‌کننده خود ارسال می‌کند.

برای احراز هویت غیرتعاملی JWT، از یک فایل JWT بار کاری و نقشی در Vault از نوع
`jwt` استفاده کنید:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=jwt
export OPENCLAW_VAULT_AUTH_MOUNT=jwt
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
export OPENCLAW_VAULT_JWT_FILE=/var/run/secrets/tokens/vault
```

فایل JWT باید توکن بار کاری فرافکنی‌شده باشد؛ مانند توکن حساب سرویس Kubernetes
با مخاطبی که نقش Vault آن را می‌پذیرد.
ورود تعاملی OIDC در مرورگر برای افراد مفید است، اما زمان اجرای Gateway به
ورود غیرتعاملی JWT یا فایل توکن نیاز دارد.

برای روش احراز هویت Kubernetes در Vault، از `kubernetes` استفاده کنید. این روش برای
Gatewayهایی در نظر گرفته شده است که به‌صورت Pod اجرا می‌شوند؛ نصب پیش‌فرض `kubernetes` است و فایل JWT پیش‌فرض
مسیر استاندارد توکن حساب سرویس است:

```bash
export VAULT_ADDR=https://vault.example.com
export OPENCLAW_VAULT_AUTH_METHOD=kubernetes
export OPENCLAW_VAULT_AUTH_ROLE=openclaw
```

فقط زمانی `OPENCLAW_VAULT_AUTH_MOUNT` را تنظیم کنید که احراز هویت Kubernetes در Vault در محلی
غیر از `auth/kubernetes` نصب شده باشد. فقط زمانی `OPENCLAW_VAULT_JWT_FILE` را تنظیم کنید که توکن
حساب سرویس در مسیری سفارشی فرافکنی شده باشد.

تنظیمات اختیاری:

```bash
export VAULT_NAMESPACE=<namespace-name>
export OPENCLAW_VAULT_KV_MOUNT=secret
export OPENCLAW_VAULT_KV_VERSION=2
```

بررسی کنید پوسته فعلی چه مواردی را می‌تواند ببیند:

```bash
openclaw vault status
```

وقتی بیش از یک ارائه‌دهنده اسرار مبتنی بر Vault پیکربندی شده است، یکی را با
نام مستعار انتخاب کنید:

```bash
openclaw vault status --provider-alias corp-vault
```

`openclaw vault status` هرگز `VAULT_TOKEN` را چاپ نمی‌کند؛ فقط گزارش می‌دهد که آیا
توکن، فایل توکن و فایل JWT تنظیم شده‌اند یا نه.

<Warning>
اگر Gateway به‌صورت سرویس، LaunchAgent، واحد systemd، وظیفه زمان‌بندی‌شده یا
کانتینر اجرا می‌شود، آن محیط زمان اجرا باید همان متغیرهای Vault را دریافت کند.
تنظیم متغیرها در یک پوسته تعاملی فقط وضعیت همان پوسته را اثبات می‌کند، نه Gateway
در حال اجرایی را که از قبل راه‌اندازی شده است.
</Warning>

## ایجاد و اعمال برنامه SecretRef

برنامه‌ای ایجاد کنید که کلید API ارائه‌دهنده مدل OpenRouter را به Vault نگاشت کند:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openrouter-id providers/openrouter/apiKey
```

برنامه را اعمال و تأیید کنید:

```bash
openclaw secrets apply --from ./vault-secrets-plan.json --dry-run --allow-exec
openclaw secrets apply --from ./vault-secrets-plan.json --allow-exec
openclaw secrets audit --check --allow-exec
openclaw secrets reload
```

از `--allow-exec` استفاده کنید، زیرا Plugin مربوط به Vault از طریق ارائه‌دهنده exec SecretRef
مدیریت‌شده توسط OpenClaw عملیات دریافت را انجام می‌دهد.

اگر Gateway هنوز در حال اجرا نیست، پس از اعمال برنامه آن را به‌طور معمول راه‌اندازی کنید
و `openclaw secrets reload` را اجرا نکنید.

## پیکربندی کلیدهای بیشتر ارائه‌دهندگان

میان‌برهای داخلی:

```bash
openclaw vault setup --openai-id providers/openai/apiKey
openclaw vault setup --anthropic-id providers/anthropic/apiKey
openclaw vault setup --openrouter-id providers/openrouter/apiKey
```

چند کلید ارائه‌دهنده در یک برنامه:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --openai-id providers/openai/apiKey \
  --anthropic-id providers/anthropic/apiKey \
  --openrouter-id providers/openrouter/apiKey
```

برای ارائه‌دهندگان همراهی که میان‌بر ندارند، یا ارائه‌دهندگان مدل سازگار با OpenAI و
سفارشی که از قبل پیکربندی شده‌اند، از `--provider-key` استفاده کنید:

```bash
openclaw vault setup \
  --plan-out ./vault-secrets-plan.json \
  --provider-key local-openai=providers/local-openai/apiKey \
  --provider-key groq=providers/groq/apiKey
```

هر `--provider-key <provider=id>` یک SecretRef را در
`models.providers.<provider>.apiKey` می‌نویسد. برای ارائه‌دهندگان سفارشی، این فرمان تنظیمات
`baseUrl`، `api` یا `models` ارائه‌دهنده را ایجاد نمی‌کند؛ ابتدا آن‌ها را پیکربندی کنید.

برای هر مسیر هدف شناخته‌شده SecretRef از `--target <path=id>` استفاده کنید:

```bash
openclaw vault setup \
  --target channels.telegram.botToken=channels/telegram/botToken \
  --target models.providers.openai.headers.x-api-key=providers/openai/proxyKey \
  --target auth-profiles:main:profiles.openai.key=providers/openai/apiKey
```

مسیرهای هدف بدون پیشوند روی `openclaw.json` اعمال می‌شوند. برای
هدف‌های موجود `auth-profiles.json` از `auth-profiles:<agentId>:<path>` استفاده کنید.
مسیر هدف باید یک هدف SecretRef ثبت‌شده OpenClaw باشد. فرمان راه‌اندازی،
اسرار نام‌گذاری‌شده دلخواهی در OpenClaw ایجاد نمی‌کند؛ Vault همچنان
مخزن اسرار باقی می‌ماند و OpenClaw فقط SecretRefها را در فیلدهای پیکربندی پشتیبانی‌شده ذخیره می‌کند.

## قالب شناسه SecretRef

شناسه‌های SecretRef مربوط به Vault از این قرارداد استفاده می‌کنند:

```text
<vault-secret-path>/<field>
```

نمونه‌ها:

| شناسه SecretRef                  | خواندن پیش‌فرض KV v2 از Vault           | فیلد بازگردانده‌شده |
| ----------------------------- | ---------------------------------- | -------------- |
| `providers/openrouter/apiKey` | `secret/data/providers/openrouter` | `apiKey`       |
| `providers/openai/apiKey`     | `secret/data/providers/openai`     | `apiKey`       |
| `teams/agent-prod/openrouter` | `secret/data/teams/agent-prod`     | `openrouter`   |

فیلد بازگردانده‌شده Vault باید رشته باشد.

برای KV v1، تنظیم کنید:

```bash
export OPENCLAW_VAULT_KV_VERSION=1
```

سپس `providers/openrouter/apiKey` این مورد را می‌خواند:

```text
secret/providers/openrouter -> apiKey
```

## مواردی که OpenClaw ذخیره می‌کند

اعمال برنامه راه‌اندازی Vault، یک ارائه‌دهنده مدیریت‌شده توسط Plugin را ذخیره می‌کند:

```json
{
  "source": "exec",
  "pluginIntegration": {
    "pluginId": "vault",
    "integrationId": "vault"
  }
}
```

فیلدهای اعتبارنامه به آن ارائه‌دهنده اشاره می‌کنند:

```json
{ "source": "exec", "provider": "vault", "id": "providers/openrouter/apiKey" }
```

مقدار دریافت‌شده فقط در اسنپ‌شات فعال اسرار زمان اجرا نگهداری می‌شود.

## کانتینرها و استقرارهای مدیریت‌شده

Gatewayهای کانتینری همچنان از همان Plugin و پیکربندی SecretRef استفاده می‌کنند.
کانتینر باید این موارد را دریافت کند:

- `VAULT_ADDR`
- یک منبع احراز هویت:
  - `VAULT_TOKEN`
  - `OPENCLAW_VAULT_AUTH_METHOD=token_file` همراه با `VAULT_TOKEN_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=jwt` همراه با `OPENCLAW_VAULT_AUTH_MOUNT`،
    `OPENCLAW_VAULT_AUTH_ROLE` و `OPENCLAW_VAULT_JWT_FILE`
  - `OPENCLAW_VAULT_AUTH_METHOD=kubernetes` همراه با `OPENCLAW_VAULT_AUTH_ROLE`؛ در صورت نیاز
    `OPENCLAW_VAULT_AUTH_MOUNT` یا `OPENCLAW_VAULT_JWT_FILE` را بازنویسی کنید
- موارد اختیاری `VAULT_NAMESPACE`، `OPENCLAW_VAULT_KV_MOUNT` و
  `OPENCLAW_VAULT_KV_VERSION`

هنگام استفاده از Kubernetes، اگر Vault برای خوشه احراز هویت Kubernetes را پیکربندی کرده است،
`OPENCLAW_VAULT_AUTH_METHOD=kubernetes` را ترجیح دهید. فقط زمانی از
`OPENCLAW_VAULT_AUTH_METHOD=jwt` استفاده کنید که Vault طوری پیکربندی شده باشد که خوشه را
یک صادرکننده عمومی JWT/OIDC در نظر بگیرد. هر دو گزینه از توکن بلندمدت Vault
در یک Secret مربوط به Kubernetes بهتر هستند. استقرارهای سایدکار یا تزریق‌کننده Vault Agent می‌توانند
در عوض از `token_file` استفاده کنند.

برای راه‌اندازی‌های چندمستاجری Vault، مسیریابی مستاجر را در خط‌مشی Vault و
پیکربندی استقرار نگه دارید. OpenClaw به نصب، نقش یا مسیر ثابتی نیاز ندارد: هر
محیط Gateway می‌تواند `OPENCLAW_VAULT_KV_MOUNT`،
`OPENCLAW_VAULT_AUTH_ROLE` و شناسه‌های SecretRef خود را تنظیم کند. اگر یک Gateway مشترک باید به‌طور هم‌زمان
اسرار کاربران مختلف Vault را دریافت کند، از ارائه‌دهندگان exec با پیکربندی دستی
استفاده کنید که محیط‌های احراز هویت مجزا را دربر می‌گیرند، یا مستاجران را میان محیط‌های Gateway
با محیط‌های Vault جداگانه تقسیم کنید.

## مرتبط

- [مدیریت اسرار](/fa/gateway/secrets)
- [`openclaw secrets`](/fa/cli/secrets)
- [فهرست Pluginها](/fa/plugins/plugin-inventory)
