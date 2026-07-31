---
read_when:
    - کار روی قابلیت‌های کانال Microsoft Teams
summary: وضعیت پشتیبانی، قابلیت‌ها و پیکربندی ربات Microsoft Teams
title: Microsoft Teams
x-i18n:
    generated_at: "2026-07-27T16:15:49Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 5a4cf686da27e28b58f7afaad8cc837dbddb93219cde0c37285f9f6895f6fb8c
    source_path: channels/msteams.md
    workflow: 16
---

وضعیت: متن + پیوست‌های پیام خصوصی پشتیبانی می‌شوند؛ ارسال فایل در کانال/گروه به `sharePointSiteId` + مجوزهای Graph نیاز دارد (به [ارسال فایل در گفت‌وگوهای گروهی](#sending-files-in-group-chats) مراجعه کنید). نظرسنجی‌ها از طریق Adaptive Cards ارسال می‌شوند. کنش‌های پیام، `upload-file` صریح را برای ارسال‌هایی که ابتدا فایل را می‌فرستند، ارائه می‌کنند.

## Plugin همراه

Microsoft Teams در نسخه‌های فعلی OpenClaw به‌صورت Plugin همراه عرضه می‌شود؛ در بیلد بسته‌بندی‌شده معمول، نصب جداگانه‌ای لازم نیست.

در بیلد قدیمی‌تر یا نصب سفارشی‌ای که Teams همراه را مستثنا می‌کند، بسته npm را مستقیماً نصب کنید:

```bash
openclaw plugins install @openclaw/msteams
```

برای دنبال‌کردن برچسب رسمی نسخه فعلی، از بسته بدون نسخه استفاده کنید. تنها زمانی یک نسخه دقیق را ثابت کنید که به نصب تکرارپذیر نیاز دارید.

پرداخت محلی (اجرا از یک مخزن git):

```bash
openclaw plugins install ./path/to/local/msteams-plugin
```

جزئیات: [Pluginها](/fa/tools/plugin)

## راه‌اندازی سریع

[`@microsoft/teams.cli`](https://www.npmjs.com/package/@microsoft/teams.cli) ثبت ربات، ایجاد مانیفست و تولید اطلاعات اعتبارسنجی را در یک فرمان انجام می‌دهد.

**1. نصب و ورود**

```bash
npm install -g @microsoft/teams.cli@preview
teams login
teams status   # بررسی کنید وارد شده‌اید و اطلاعات مستأجر خود را می‌بینید
```

<Note>
Teams CLI در حال حاضر در مرحله پیش‌نمایش است. فرمان‌ها و پرچم‌ها ممکن است بین نسخه‌ها تغییر کنند.
</Note>

**2. راه‌اندازی تونل** (Teams نمی‌تواند به localhost دسترسی پیدا کند)

در صورت نیاز، devtunnel CLI را نصب و احراز هویت کنید ([راهنمای شروع](https://learn.microsoft.com/en-us/azure/developer/dev-tunnels/get-started)).

```bash
# راه‌اندازی یک‌باره (نشانی پایدار در نشست‌های مختلف):
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# در هر نشست توسعه:
devtunnel host my-openclaw-bot
# نقطه پایانی شما: https://<tunnel-id>.devtunnels.ms/api/messages
```

<Note>
`--allow-anonymous` الزامی است، زیرا Teams نمی‌تواند با devtunnels احراز هویت کند. هر درخواست ورودی ربات همچنان توسط Teams SDK اعتبارسنجی می‌شود.
</Note>

گزینه‌های جایگزین: `ngrok http 3978` یا `tailscale funnel 3978` (نشانی‌ها ممکن است در هر نشست تغییر کنند).

**3. ایجاد برنامه**

```bash
teams app create \
  --name "OpenClaw" \
  --endpoint "https://<your-tunnel-url>/api/messages"
```

این فرمان یک برنامه Entra ID ‏(Azure AD) ایجاد می‌کند، یک رمز محرمانه کلاینت می‌سازد، مانیفست برنامه Teams را (همراه با آیکون‌ها) ساخته و بارگذاری می‌کند و یک ربات مدیریت‌شده توسط Teams ثبت می‌کند (بدون نیاز به اشتراک Azure). خروجی شامل `CLIENT_ID`، `CLIENT_SECRET`، `TENANT_ID` و یک **Teams App ID** است؛ همچنین امکان نصب مستقیم برنامه در Teams را ارائه می‌دهد.

**4. پیکربندی OpenClaw** با استفاده از اطلاعات اعتبارسنجی موجود در خروجی:

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<CLIENT_ID>",
      appPassword: "<CLIENT_SECRET>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

یا مستقیماً از متغیرهای محیطی استفاده کنید: `MSTEAMS_APP_ID`، `MSTEAMS_APP_PASSWORD`، `MSTEAMS_TENANT_ID`.

**5. نصب برنامه در Teams**

`teams app create` از شما می‌خواهد برنامه را نصب کنید؛ "Install in Teams" را انتخاب کنید. برای دریافت پیوند نصب در زمانی دیگر:

```bash
teams app get <teamsAppId> --install-link
```

**6. بررسی عملکرد همه‌چیز**

```bash
teams app doctor <teamsAppId>
```

این فرمان ثبت ربات، پیکربندی برنامه AAD، اعتبار مانیفست و راه‌اندازی SSO را عیب‌یابی می‌کند.

برای محیط عملیاتی، به‌جای رمزهای محرمانه کلاینت، [احراز هویت فدرال](#federated-authentication-certificate-plus-managed-identity) (گواهی یا هویت مدیریت‌شده) را در نظر بگیرید.

<Note>
گفت‌وگوهای گروهی به‌طور پیش‌فرض مسدود هستند (`channels.msteams.groupPolicy: "allowlist"`). برای مجازکردن پاسخ‌های گروهی، `channels.msteams.groupAllowFrom` را تنظیم کنید، یا برای مجازکردن هر عضو (با الزام منشن) از `groupPolicy: "open"` استفاده کنید.
</Note>

## اهداف

- از طریق پیام‌های خصوصی، گفت‌وگوهای گروهی یا کانال‌های Teams با OpenClaw صحبت کنید.
- مسیریابی را قطعی نگه دارید: پاسخ‌ها همیشه به همان کانالی بازمی‌گردند که از آن دریافت شده‌اند.
- رفتار امن کانال را پیش‌فرض قرار دهید (مگر آنکه خلاف آن پیکربندی شده باشد، منشن الزامی است).

## نوشتن پیکربندی

به‌طور پیش‌فرض، Microsoft Teams می‌تواند به‌روزرسانی‌های پیکربندی فعال‌شده توسط `/config set|unset` را بنویسد (نیازمند `commands.config: true`).

برای غیرفعال‌سازی:

```json5
{
  channels: { msteams: { configWrites: false } },
}
```

## کنترل دسترسی (پیام‌های خصوصی + گروه‌ها)

**دسترسی پیام خصوصی**

- پیش‌فرض: `channels.msteams.dmPolicy = "pairing"`. فرستندگان ناشناس تا زمان تأیید نادیده گرفته می‌شوند.
- `channels.msteams.allowFrom` باید از شناسه‌های پایدار شیء AAD یا گروه‌های دسترسی ایستای فرستنده مانند `accessGroup:core-team` استفاده کند.
- برای فهرست‌های مجاز به تطبیق UPN/نام نمایشی متکی نباشید؛ این مقادیر ممکن است تغییر کنند. OpenClaw تطبیق مستقیم نام را به‌طور پیش‌فرض غیرفعال می‌کند؛ برای فعال‌سازی آن، `channels.msteams.dangerouslyAllowNameMatching: true` را تنظیم کنید.
- هنگامی که اطلاعات اعتبارسنجی اجازه دهد، جادوگر می‌تواند نام‌ها را از طریق Microsoft Graph به شناسه تبدیل کند.

**دسترسی گروه**

- پیش‌فرض: `channels.msteams.groupPolicy = "allowlist"` (مسدود است، مگر اینکه `groupAllowFrom` را اضافه کنید). هنگامی که `channels.msteams.groupPolicy` تنظیم نشده باشد، `channels.defaults.groupPolicy` می‌تواند پیش‌فرض مشترک را بازنویسی کند.
- `channels.msteams.groupAllowFrom` تعیین می‌کند کدام فرستندگان یا گروه‌های دسترسی ایستای فرستنده می‌توانند در گفت‌وگوهای گروهی/کانال‌ها فعال‌سازی کنند (در صورت نبود، از `channels.msteams.allowFrom` استفاده می‌شود).
- برای مجازکردن هر عضو، `groupPolicy: "open"` را تنظیم کنید (همچنان به‌طور پیش‌فرض منشن الزامی است).
- برای مسدودکردن **همه** کانال‌ها، `channels.msteams.groupPolicy: "disabled"` را تنظیم کنید.

مثال:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      groupAllowFrom: ["00000000-0000-0000-0000-000000000000", "accessGroup:core-team"],
    },
  },
}
```

**فهرست مجاز تیم + کانال**

- با فهرست‌کردن تیم‌ها و کانال‌ها در `channels.msteams.teams`، دامنه پاسخ‌های گروه/کانال را محدود کنید.
- به‌جای نام‌های نمایشی تغییرپذیر، از شناسه‌های پایدار مکالمه Teams در پیوندهای Teams به‌عنوان کلید استفاده کنید (به [شناسه‌های تیم و کانال](#team-and-channel-ids-common-gotcha) مراجعه کنید).
- هنگامی که `groupPolicy="allowlist"` و یک فهرست مجاز تیم‌ها وجود داشته باشد، فقط تیم‌ها/کانال‌های فهرست‌شده پذیرفته می‌شوند (با الزام منشن).
- جادوگر پیکربندی ورودی‌های `Team/Channel` را می‌پذیرد و آن‌ها را برای شما ذخیره می‌کند.
- هنگام راه‌اندازی، OpenClaw نام‌های فهرست مجاز تیم/کانال و کاربر را به شناسه تبدیل می‌کند (اگر مجوزهای Graph اجازه دهند) و نگاشت را در گزارش ثبت می‌کند. نام‌های حل‌نشده همان‌گونه که وارد شده‌اند نگه داشته می‌شوند، اما مگر آنکه `channels.msteams.dangerouslyAllowNameMatching: true` تنظیم شده باشد، برای مسیریابی نادیده گرفته می‌شوند.

مثال:

```json5
{
  channels: {
    msteams: {
      groupPolicy: "allowlist",
      teams: {
        "My Team": {
          channels: {
            General: { requireMention: true },
          },
        },
      },
    },
  },
}
```

<details>
<summary><strong>راه‌اندازی دستی (بدون Teams CLI)</strong></summary>

### نحوه کار

1. اطمینان حاصل کنید که Plugin مربوط به Microsoft Teams در دسترس است (در نسخه‌های فعلی همراه است).
2. یک **Azure Bot** ایجاد کنید (App ID + رمز محرمانه + شناسه مستأجر).
3. یک **بسته برنامه Teams** بسازید که به ربات ارجاع دهد و شامل مجوزهای RSC زیر باشد.
4. برنامه Teams را در یک تیم (یا برای پیام‌های خصوصی، در دامنه شخصی) بارگذاری/نصب کنید.
5. `msteams` را در `~/.openclaw/openclaw.json` (یا متغیرهای محیطی) پیکربندی و Gateway را راه‌اندازی کنید.
6. Gateway به‌طور پیش‌فرض در `/api/messages` به ترافیک Webhook مربوط به Bot Framework گوش می‌دهد.

### مرحله 1: ایجاد Azure Bot

1. به [Create Azure Bot](https://portal.azure.com/#create/Microsoft.AzureBot) بروید.
2. زبانه **Basics** را تکمیل کنید:

   | فیلد               | مقدار                                                    |
   | ------------------ | -------------------------------------------------------- |
   | **Bot handle**     | نام ربات شما، برای مثال `openclaw-msteams` (باید یکتا باشد) |
   | **Subscription**   | اشتراک Azure خود را انتخاب کنید                           |
   | **Resource group** | یک مورد جدید ایجاد یا از مورد موجود استفاده کنید           |
   | **Pricing tier**   | **Free** برای توسعه/آزمایش                               |
   | **Type of App**    | **Single Tenant** (توصیه‌شده؛ یادداشت زیر را ببینید)       |
   | **Creation type**  | **Create new Microsoft App ID**                          |

<Warning>
ایجاد ربات‌های چندمستأجری جدید پس از 2025-07-31 منسوخ شد. برای ربات‌های جدید از **Single Tenant** استفاده کنید.
</Warning>

3. روی **Review + create** و سپس **Create** کلیک کنید (~1-2 دقیقه).

### مرحله 2: دریافت اطلاعات اعتبارسنجی

1. منبع Azure Bot → **Configuration** → **Microsoft App ID** را کپی کنید (`appId` شما).
2. **Manage Password** → App Registration → **Certificates & secrets** → **New client secret** → **Value** را کپی کنید (`appPassword` شما).
3. **Overview** → **Directory (tenant) ID** را کپی کنید (`tenantId` شما).

### مرحله 3: پیکربندی نقطه پایانی پیام‌رسانی

1. Azure Bot → **Configuration**.
2. **Messaging endpoint** را تنظیم کنید:
   - محیط عملیاتی: `https://your-domain.com/api/messages`
   - توسعه محلی: از تونل استفاده کنید (به [توسعه محلی](#local-development-tunneling) مراجعه کنید)

### مرحله 4: فعال‌سازی کانال Teams

1. Azure Bot → **Channels**.
2. روی **Microsoft Teams** → Configure → Save کلیک کنید.
3. شرایط خدمات را بپذیرید.

### مرحله 5: ساخت مانیفست برنامه Teams

- یک ورودی `bot` با `botId = <App ID>` اضافه کنید.
- دامنه‌ها: `personal`، `team`، `groupChat`.
- `supportsFiles: true` (برای مدیریت فایل در دامنه شخصی الزامی است).
- مجوزهای RSC را اضافه کنید (به [مجوزهای RSC](#current-teams-rsc-permissions-manifest) مراجعه کنید).
- آیکون‌ها را ایجاد کنید: `outline.png` (32x32) و `color.png` (192x192).
- `manifest.json`، `outline.png` و `color.png` را با هم فشرده کنید.

### مرحله 6: پیکربندی OpenClaw

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      appPassword: "<APP_PASSWORD>",
      tenantId: "<TENANT_ID>",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

متغیرهای محیطی: `MSTEAMS_APP_ID`، `MSTEAMS_APP_PASSWORD`، `MSTEAMS_TENANT_ID`.

### مرحله 7: اجرای Gateway

هنگامی که Plugin در دسترس باشد و پیکربندی `msteams` دارای اطلاعات اعتبارسنجی باشد، کانال Teams به‌طور خودکار راه‌اندازی می‌شود.

</details>

## احراز هویت فدرال (گواهی به‌همراه هویت مدیریت‌شده)

برای محیط عملیاتی، OpenClaw از طریق `channels.msteams.authType: "federated"` از **احراز هویت فدرال** به‌عنوان جایگزینی برای رمزهای محرمانه کلاینت پشتیبانی می‌کند. دو روش وجود دارد:

### گزینه A: احراز هویت مبتنی بر گواهی

از یک گواهی PEM ثبت‌شده در برنامه Entra ID خود استفاده کنید.

**راه‌اندازی:**

1. یک گواهی تولید یا دریافت کنید (قالب PEM همراه با کلید خصوصی).
2. Entra ID → App Registration → **Certificates & secrets** → **Certificates** → گواهی عمومی را بارگذاری کنید.

**پیکربندی:**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      tenantId: "<TENANT_ID>",
      authType: "federated",
      certificatePath: "/path/to/cert.pem",
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**متغیرهای محیطی:**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_CERTIFICATE_PATH=/path/to/cert.pem`

### گزینه B: هویت مدیریت‌شده Azure

برای احراز هویت بدون گذرواژه در زیرساخت Azure ‏(AKS، App Service، ماشین‌های مجازی Azure) از هویت مدیریت‌شده Azure استفاده کنید.

**نحوه کار:**

1. پاد/ماشین مجازی ربات دارای یک هویت مدیریت‌شده است (اختصاص‌یافته توسط سیستم یا کاربر).
2. یک اطلاعات اعتبارسنجی هویت فدرال، هویت مدیریت‌شده را به ثبت برنامه Entra ID پیوند می‌دهد.
3. در زمان اجرا، OpenClaw برای دریافت توکن‌ها از نقطه پایانی Azure IMDS از `@azure/identity` استفاده می‌کند.
4. توکن برای احراز هویت ربات به Teams SDK ارسال می‌شود.

**پیش‌نیازها:**

- زیرساخت Azure با هویت مدیریت‌شده فعال (هویت بار کاری AKS، App Service، VM).
- اعتبارنامه هویت فدرال‌شده در ثبت برنامه Entra ID ایجاد شده است.
- دسترسی شبکه به IMDS (`169.254.169.254:80`) از پاد/VM.

**پیکربندی (هویت مدیریت‌شده اختصاص‌یافته توسط سیستم):**

```json5
{
  channels: {
    msteams: {
      enabled: true,
      appId: "<APP_ID>",
      tenantId: "<TENANT_ID>",
      authType: "federated",
      useManagedIdentity: true,
      webhook: { port: 3978, path: "/api/messages" },
    },
  },
}
```

**پیکربندی (هویت مدیریت‌شده اختصاص‌یافته توسط کاربر):** `managedIdentityClientId: "<MI_CLIENT_ID>"` را به بلوک بالا اضافه کنید.

**متغیرهای محیطی:**

- `MSTEAMS_AUTH_TYPE=federated`
- `MSTEAMS_USE_MANAGED_IDENTITY=true`
- `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID=<client-id>` (فقط اختصاص‌یافته توسط کاربر)

### راه‌اندازی هویت بار کاری AKS

برای استقرارهای AKS که از هویت بار کاری استفاده می‌کنند:

1. **هویت بار کاری را** در کلاستر AKS خود **فعال کنید**.
2. در ثبت برنامه Entra ID، **یک اعتبارنامه هویت فدرال‌شده ایجاد کنید**:

   ```bash
   az ad app federated-credential create --id <APP_OBJECT_ID> --parameters '{
     "name": "my-bot-workload-identity",
     "issuer": "<AKS_OIDC_ISSUER_URL>",
     "subject": "system:serviceaccount:<NAMESPACE>:<SERVICE_ACCOUNT>",
     "audiences": ["api://AzureADTokenExchange"]
   }'
   ```

3. حساب سرویس Kubernetes را با شناسه کلاینت برنامه **حاشیه‌نویسی کنید**:

   ```yaml
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: my-bot-sa
     annotations:
       azure.workload.identity/client-id: "<APP_CLIENT_ID>"
   ```

4. برای تزریق هویت بار کاری، **به پاد برچسب بزنید**:

   ```yaml
   metadata:
     labels:
       azure.workload.identity/use: "true"
   ```

5. **دسترسی شبکه را** به IMDS (`169.254.169.254`) **مجاز کنید**: اگر از NetworkPolicy استفاده می‌کنید، یک قاعده خروجی برای `169.254.169.254/32` روی درگاه 80 اضافه کنید.

### مقایسه انواع احراز هویت

| روش                       | پیکربندی                                      | مزایا                                    | معایب                                             |
| ------------------------- | --------------------------------------------- | ---------------------------------------- | ------------------------------------------------- |
| **رمز کلاینت**            | `appPassword`                            | راه‌اندازی ساده                          | نیازمند چرخش راز، امنیت کمتر                      |
| **گواهی**                 | `authType: "federated"` + `certificatePath`       | بدون راز مشترک روی شبکه                  | سربار مدیریت گواهی                                |
| **هویت مدیریت‌شده**       | `authType: "federated"` + `useManagedIdentity`       | بدون گذرواژه، بدون نیاز به مدیریت رازها | نیازمند زیرساخت Azure                              |

`certificateThumbprint` را می‌توان همراه با `certificatePath` تنظیم کرد، اما در حال حاضر مسیر احراز هویت آن را نمی‌خواند؛ این مقدار فقط برای سازگاری آینده پذیرفته می‌شود.

**پیش‌فرض:** وقتی `authType` تنظیم نشده باشد، OpenClaw از احراز هویت با رمز کلاینت (`appPassword`) استفاده می‌کند. پیکربندی‌های موجود بدون تغییر به کار خود ادامه می‌دهند.

## توسعه محلی (تونل‌سازی)

Teams نمی‌تواند به `localhost` دسترسی پیدا کند. از یک تونل توسعه پایدار استفاده کنید تا URL در نشست‌های مختلف ثابت بماند:

```bash
# راه‌اندازی یک‌باره:
devtunnel create my-openclaw-bot --allow-anonymous
devtunnel port create my-openclaw-bot -p 3978 --protocol auto

# هر نشست توسعه:
devtunnel host my-openclaw-bot
```

گزینه‌های جایگزین: `ngrok http 3978` یا `tailscale funnel 3978` (ممکن است URLها در هر نشست تغییر کنند).

اگر URL تونل تغییر کرد، نقطه پایانی را به‌روزرسانی کنید:

```bash
teams app update <teamsAppId> --endpoint "https://<new-url>/api/messages"
```

## آزمایش ربات

**اجرای عیب‌یابی:**

```bash
teams app doctor <teamsAppId>
```

ثبت ربات، برنامه AAD، مانیفست و پیکربندی SSO را در یک مرحله بررسی می‌کند.

**ارسال پیام آزمایشی:**

1. برنامه Teams را نصب کنید (پیوند نصب از `teams app get <id> --install-link`).
2. ربات را در Teams پیدا کنید و یک پیام مستقیم ارسال کنید.
3. گزارش‌های Gateway را برای فعالیت ورودی بررسی کنید.

## متغیرهای محیطی

این کلیدهای پیکربندی مرتبط با احراز هویت را می‌توان به‌جای `openclaw.json` از طریق متغیرهای محیطی تنظیم کرد (سایر کلیدهای پیکربندی، مانند `groupPolicy` یا `historyLimit`، فقط از طریق پیکربندی قابل تنظیم‌اند):

| متغیر محیطی                           | کلید پیکربندی             | یادداشت                                  |
| ------------------------------------- | ------------------------- | ---------------------------------------- |
| `MSTEAMS_APP_ID`                    | `appId`        |                                          |
| `MSTEAMS_APP_PASSWORD`                    | `appPassword`        |                                          |
| `MSTEAMS_TENANT_ID`                    | `tenantId`        |                                          |
| `MSTEAMS_AUTH_TYPE`                    | `authType`        | `"secret"` یا `"federated"` |
| `MSTEAMS_CERTIFICATE_PATH`                    | `certificatePath`        | فدرال‌شده + گواهی                        |
| `MSTEAMS_CERTIFICATE_THUMBPRINT`                    | `certificateThumbprint`        | پذیرفته می‌شود، برای احراز هویت الزامی نیست |
| `MSTEAMS_USE_MANAGED_IDENTITY`                    | `useManagedIdentity`        | فدرال‌شده + هویت مدیریت‌شده              |
| `MSTEAMS_MANAGED_IDENTITY_CLIENT_ID`                    | `managedIdentityClientId`        | فقط هویت مدیریت‌شده اختصاص‌یافته توسط کاربر |

## کنش اطلاعات عضو

OpenClaw برای Microsoft Teams یک کنش `member-info` با پشتوانه Graph ارائه می‌کند تا عامل‌ها و خودکارسازی‌ها بتوانند جزئیات تأییدشده فهرست اعضا را برای یک مکالمه پیکربندی‌شده بازیابی کنند.

الزامات:

- مجوزهای RSC ‏`ChannelSettings.Read.Group` و `TeamMember.Read.Group` (از قبل در مانیفست پیشنهادی موجود هستند).

هرگاه اعتبارنامه‌های Graph پیکربندی شده باشند، این کنش در دسترس است؛ کلید جداگانه‌ای به نام `channels.msteams.actions.memberInfo` وجود ندارد.
جست‌وجوهای کانال استاندارد، هویت منطبق در فهرست اعضای تیم، نام نمایشی، ایمیل و نقش‌ها را برمی‌گردانند.
در پیام مستقیم یا گفت‌وگوی گروهی فعلی، این کنش می‌تواند شناسه کاربری پایدار فرستنده مورد اعتماد را برگرداند.
جست‌وجوی اعضای کانال خصوصی/اشتراکی و گفت‌وگوهای غیرجاری به مجوزهای اضافی فهرست اعضا نیاز دارد
و خط‌مشی پایه مجوز پیش‌فرض آن‌ها را رد می‌کند.

## زمینه تاریخچه

- `channels.msteams.historyLimit` تعداد پیام‌های اخیر کانال/گروه را که در پرامپت گنجانده می‌شوند کنترل می‌کند. در صورت نبود آن، از `messages.groupChat.historyLimit` استفاده می‌شود و سپس مقدار پیش‌فرض 50 است. برای غیرفعال‌سازی، `0` را تنظیم کنید.
- تاریخچه رشته دریافت‌شده بر اساس فهرست‌های مجاز فرستندگان (`allowFrom` / `groupAllowFrom`) فیلتر می‌شود؛ بنابراین مقداردهی اولیه زمینه رشته فقط پیام‌های فرستندگان مجاز را شامل می‌شود.
- زمینه پیوست نقل‌قول‌شده (تجزیه‌شده از HTML طرح‌واره Skype Reply در پیوست‌های خود پاسخ) بدون فیلتر منتقل می‌شود؛ در حال حاضر فقط مقداردهی اولیه تاریخچه رشته، فیلتر فهرست مجاز فرستندگان را اعمال می‌کند.
- تاریخچه پیام مستقیم را می‌توان با `channels.msteams.dmHistoryLimit` (نوبت‌های کاربر) محدود کرد. بازنویسی‌های مختص هر کاربر: `channels.msteams.dms["<user_id>"].historyLimit`.

## مجوزهای RSC فعلی Teams (مانیفست)

این‌ها **مجوزهای resourceSpecific موجود** در مانیفست برنامه Teams ما هستند. این مجوزها فقط در تیم/گفت‌وگویی اعمال می‌شوند که برنامه در آن نصب شده است.

**برای کانال‌ها (محدوده تیم):**

- `ChannelMessage.Read.Group` (Application) - دریافت همه پیام‌های کانال بدون @mention
- `ChannelMessage.Send.Group` (Application)
- `Member.Read.Group` (Application)
- `Owner.Read.Group` (Application)
- `ChannelSettings.Read.Group` (Application)
- `TeamMember.Read.Group` (Application)
- `TeamSettings.Read.Group` (Application)

**برای گفت‌وگوهای گروهی:**

- `ChatMessage.Read.Chat` (Application) - دریافت همه پیام‌های گفت‌وگوی گروهی بدون @mention

مجوزهای RSC را از طریق CLI ‏Teams اضافه کنید:

```bash
teams app rsc add <teamsAppId> ChannelMessage.Read.Group --type Application
```

## نمونه مانیفست Teams (با اطلاعات حذف‌شده)

نمونه‌ای حداقلی و معتبر با فیلدهای الزامی. شناسه‌ها و URLها را جایگزین کنید.

```json5
{
  $schema: "https://developer.microsoft.com/en-us/json-schemas/teams/v1.23/MicrosoftTeams.schema.json",
  manifestVersion: "1.23",
  version: "1.0.0",
  id: "00000000-0000-0000-0000-000000000000",
  name: { short: "OpenClaw" },
  developer: {
    name: "سازمان شما",
    websiteUrl: "https://example.com",
    privacyUrl: "https://example.com/privacy",
    termsOfUseUrl: "https://example.com/terms",
  },
  description: { short: "OpenClaw در Teams", full: "OpenClaw در Teams" },
  icons: { outline: "outline.png", color: "color.png" },
  accentColor: "#5B6DEF",
  bots: [
    {
      botId: "11111111-1111-1111-1111-111111111111",
      scopes: ["personal", "team", "groupChat"],
      isNotificationOnly: false,
      supportsCalling: false,
      supportsVideo: false,
      supportsFiles: true,
    },
  ],
  webApplicationInfo: {
    id: "11111111-1111-1111-1111-111111111111",
  },
  authorization: {
    permissions: {
      resourceSpecific: [
        { name: "ChannelMessage.Read.Group", type: "Application" },
        { name: "ChannelMessage.Send.Group", type: "Application" },
        { name: "Member.Read.Group", type: "Application" },
        { name: "Owner.Read.Group", type: "Application" },
        { name: "ChannelSettings.Read.Group", type: "Application" },
        { name: "TeamMember.Read.Group", type: "Application" },
        { name: "TeamSettings.Read.Group", type: "Application" },
        { name: "ChatMessage.Read.Chat", type: "Application" },
      ],
    },
  },
}
```

### ملاحظات مانیفست (فیلدهای الزامی)

- `bots[].botId` **باید** با شناسه برنامه Azure Bot مطابقت داشته باشد.
- `webApplicationInfo.id` **باید** با شناسه برنامه Azure Bot مطابقت داشته باشد.
- `bots[].scopes` باید سطوحی را که قصد استفاده از آن‌ها را دارید شامل شود (`personal`، `team`، `groupChat`).
- `bots[].supportsFiles: true` برای مدیریت فایل در محدوده شخصی الزامی است.
- `authorization.permissions.resourceSpecific` باید خواندن/ارسال کانال را برای ترافیک کانال شامل شود.

### به‌روزرسانی یک برنامه موجود

```bash
# مانیفست را دانلود، ویرایش و دوباره بارگذاری کنید
teams app manifest download <teamsAppId> manifest.json
# manifest.json را به‌صورت محلی ویرایش کنید...
teams app manifest upload manifest.json <teamsAppId>
# اگر محتوا تغییر کرده باشد، نسخه به‌طور خودکار افزایش می‌یابد
```

پس از به‌روزرسانی، برنامه را در هر تیم دوباره نصب کنید و برای پاک‌شدن فراداده ذخیره‌شده برنامه، **Teams را کاملاً ببندید و دوباره اجرا کنید** (صرفاً پنجره را نبندید).

<details>
<summary>به‌روزرسانی دستی مانیفست (بدون CLI)</summary>

1. `manifest.json` را با تنظیمات جدید به‌روزرسانی کنید.
2. **فیلد `version` را افزایش دهید** (برای مثال، `1.0.0` → `1.1.0`).
3. مانیفست را همراه با آیکن‌ها (`manifest.json`، `outline.png`، `color.png`) **دوباره فشرده کنید**.
4. فایل zip جدید را بارگذاری کنید:
   - **Teams Admin Center:** Teams apps → Manage apps → برنامه خود را پیدا کنید → Upload new version.
   - **Sideload:** Teams → Apps → Manage your apps → Upload a custom app.

</details>

## قابلیت‌ها: فقط RSC در برابر Graph

### با **فقط RSC ‏Teams** (برنامه نصب شده، بدون مجوزهای Graph API)

کار می‌کند:

- خواندن محتوای **متنی** پیام کانال.
- ارسال محتوای **متنی** پیام کانال.
- دریافت پیوست‌های فایل **شخصی (پیام مستقیم)**.

کار نمی‌کند:

- **محتوای تصویر یا فایل** کانال/گروه (بار داده فقط شامل یک جای‌نگهدار HTML است).
- بارگیری پیوست‌های ذخیره‌شده در SharePoint/OneDrive.
- خواندن تاریخچه پیام‌ها فراتر از رویداد زنده Webhook.

### با **RSC ‏Teams + مجوزهای Application ‏Microsoft Graph**

موارد زیر را اضافه می‌کند:

- بارگیری محتوای میزبانی‌شده (تصاویر چسبانده‌شده در پیام‌ها).
- بارگیری پیوست‌های فایل ذخیره‌شده در SharePoint/OneDrive.
- خواندن تاریخچه پیام‌های کانال/گفت‌وگو از طریق Graph.

### RSC در برابر Graph API

| قابلیت                       | مجوزهای RSC              | Graph API                                  |
| ---------------------------- | ------------------------ | ------------------------------------------ |
| **پیام‌های بلادرنگ**         | بله (از طریق Webhook)    | خیر (فقط نظرسنجی دوره‌ای)                  |
| **پیام‌های تاریخی**          | خیر                      | بله (امکان پرس‌وجوی تاریخچه)               |
| **پیچیدگی راه‌اندازی**       | فقط مانیفست برنامه       | نیازمند رضایت مدیر + جریان توکن            |
| **کارکرد در حالت آفلاین**    | خیر (باید در حال اجرا باشد) | بله (پرس‌وجو در هر زمان)                |

**جمع‌بندی:** RSC برای شنود بلادرنگ است؛ Graph API برای دسترسی تاریخی است. برای دریافت پیام‌های ازدست‌رفته در زمان آفلاین بودن، به Graph API همراه با `ChannelMessage.Read.All` نیاز دارید (نیازمند رضایت مدیر).

## رسانه + تاریخچه با قابلیت Graph

فقط مجوزهای برنامه Microsoft Graph موردنیاز برای دامنه‌ها و داده‌های Teams که استفاده می‌کنید فعال کنید:

1. Entra ID (Azure AD) **App Registration** ← مجوزهای **Application permissions** مربوط به Graph را اضافه کنید:
   - `ChannelMessage.Read.All` برای پیوست‌های کانال و تاریخچه کانال.
   - `Chat.Read.All` برای پیوست‌های گفت‌وگوی گروهی و تاریخچه گفت‌وگوی گروهی.
   - `Files.Read.All` هنگامی که بایت‌های پیوست باید از فضای ذخیره‌سازی SharePoint/OneDrive دانلود شوند؛ راه‌اندازی‌هایی که فقط از تاریخچه استفاده می‌کنند به آن نیاز ندارند.
2. برای مستأجر، **Grant admin consent** را انجام دهید.
3. **نسخه مانیفست** برنامه Teams را افزایش دهید، دوباره بارگذاری کنید و **برنامه را در Teams مجدداً نصب کنید**.
4. برای پاک‌کردن فراداده ذخیره‌شده برنامه، **Teams را کاملاً ببندید و دوباره اجرا کنید**.

### بازیابی فایل کانال/گروه (`graphMediaFallback`)

Teams می‌تواند نشانگرهای فایل را از فعالیت HTML ارسال‌شده به ربات حذف کند. در این حالت، فعالیت Bot Framework از یک پیام HTML معمولی قابل‌تشخیص نیست؛ ارجاع کامل پیوست فقط در نسخه Graph پیام وجود دارد.

پس از اعطای مجوزهای بالا، سازوکار جایگزین را فعال کنید:

```json5
{
  channels: {
    msteams: {
      graphMediaFallback: true,
    },
  },
}
```

این فقط برای کانال‌ها و گفت‌وگوهای گروهی اعمال می‌شود. هرگاه یک فعالیت HTML هیچ رسانه قابل‌دانلود مستقیمی تولید نکند، از جمله پیام‌های معمولی یا پیام‌هایی که فقط شامل اشاره هستند، یک جست‌وجوی پیام Graph اضافه می‌کند. مقدار پیش‌فرض `false` است تا نصب‌های موجود به‌طور خودکار با ترافیک اضافی Graph یا خطاهای مجوز مواجه نشوند.

**اشاره به کاربران:** ‎@mentionها برای کاربرانی که از قبل در گفت‌وگو حضور دارند، بدون تنظیم اضافی کار می‌کنند. برای جست‌وجو و اشاره پویا به کاربرانی که **در گفت‌وگوی فعلی نیستند**، مجوز `User.Read.All` از نوع Application را اضافه کنید و رضایت مدیر را اعطا کنید.

## محدودیت‌های شناخته‌شده

### مهلت‌های زمانی Webhook

Teams پیام‌ها را از طریق Webhook ‏HTTP تحویل می‌دهد. OpenClaw مهلت‌های زمانی ثابت سرور HTTP را روی شنونده آن Webhook اعمال می‌کند: 30s برای بی‌فعالیتی، 30s برای کل درخواست و 15s برای دریافت سرآیندها. رسانه ورودی اختیاری و غنی‌سازی زمینه، بودجه مشترک 10‌ثانیه‌ای دارند. SDK پس از آنکه فعالیت خام به‌صورت پایدار افزوده شد بازمی‌گردد؛ نوبت عامل به‌طور مستقل تخلیه می‌شود و پاسخ‌ها را به‌صورت پیش‌دستانه ارسال می‌کند. اگر رسیدگی به درخواست یا پذیرش پایدار از پنجره زمانی انتقال عبور کند، Teams ممکن است فعالیت را دوباره امتحان کند و سنگ‌قبر ورودی، شناسه رویداد تکراری را رد می‌کند.

### پشتیبانی از ابر Teams و URL سرویس

این مسیر Teams مبتنی بر SDK برای ابر عمومی Microsoft Teams به‌صورت زنده اعتبارسنجی شده است.

پاسخ‌های ورودی از زمینه نوبت ورودی SDK ‏Teams استفاده می‌کنند. عملیات پیش‌دستانه خارج از زمینه ــ ارسال، ویرایش، حذف، کارت‌ها، نظرسنجی‌ها، پیام‌های رضایت فایل و پاسخ‌های طولانی‌مدت صف‌شده ــ از ارجاع ذخیره‌شده گفت‌وگوی `serviceUrl` استفاده می‌کنند. ابر عمومی به‌طور پیش‌فرض از محیط ابر عمومی SDK ‏Teams استفاده می‌کند و ارجاع‌های ذخیره‌شده روی میزبان عمومی Teams Connector را مجاز می‌داند: `https://smba.trafficmanager.net/`.

ابر عمومی حالت پیش‌فرض است. برای ربات‌های معمول ابر عمومی نیازی به تنظیم `channels.msteams.cloud` یا `channels.msteams.serviceUrl` ندارید.

برای ابرهای غیرعمومی Teams، ‏`cloud` و مرز پیش‌دستانه متناظر را، هنگامی که Microsoft آن را منتشر می‌کند، تنظیم کنید:

- `channels.msteams.cloud` پیش‌تنظیم ابری SDK ‏Teams را برای احراز هویت، اعتبارسنجی JWT، سرویس‌های توکن و دامنه Graph انتخاب می‌کند.
- `channels.msteams.serviceUrl` مرز نقطه پایانی Bot Connector را انتخاب می‌کند که برای اعتبارسنجی ارجاع‌های ذخیره‌شده گفت‌وگو پیش از اجرای ارسال‌ها، ویرایش‌ها، حذف‌ها، کارت‌ها، نظرسنجی‌ها، پیام‌های رضایت فایل و پاسخ‌های طولانی‌مدت صف‌شده استفاده می‌شود. این مورد برای ابرهای SDK ‏USGov و DoD الزامی است. برای China/21Vianet، ‏OpenClaw از پیش‌تنظیم `China` متعلق به SDK استفاده می‌کند و URLهای سرویس ذخیره‌شده/پیکربندی‌شده را فقط روی میزبان‌های کانال Azure China Bot Framework می‌پذیرد.

Microsoft نقاط پایانی عمومی و پیش‌دستانه Bot Connector را در بخش [ایجاد گفت‌وگو](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages?tabs=dotnet#create-the-conversation) از مستندات پیام‌رسانی پیش‌دستانه Teams منتشر می‌کند. در صورت وجود، از `serviceUrl` فعالیت ورودی استفاده کنید؛ در غیر این صورت، از جدول Microsoft در زیر استفاده کنید.

| محیط Teams       | پیکربندی OpenClaw                                             | `serviceUrl` پیش‌دستانه                         |
| ---------------- | ------------------------------------------------------------- | ----------------------------------------------------- |
| عمومی            | نیازی به پیکربندی cloud/serviceUrl نیست                       | `https://smba.trafficmanager.net/teams`                                    |
| GCC              | ‏`serviceUrl` را تنظیم کنید؛ پیش‌تنظیم ابری جداگانه‌ای در SDK ‏Teams وجود ندارد | `https://smba.infra.gcc.teams.microsoft.com/teams` |
| GCC High         | `cloud: "USGov"` + `serviceUrl`                       | `https://smba.infra.gov.teams.microsoft.us/teams`                                    |
| DoD              | `cloud: "USGovDoD"` + `serviceUrl`                       | `https://smba.infra.dod.teams.microsoft.us/teams`                                    |
| China/21Vianet   | `cloud: "China"`                                            | از `serviceUrl` فعالیت ورودی استفاده کنید      |

مثال برای GCC، که Microsoft برای آن یک URL سرویس پیش‌دستانه جداگانه مستند کرده است، اما SDK ‏Teams هیچ پیش‌تنظیم ابری جداگانه‌ای برای GCC ارائه نمی‌کند:

```json
{
  "channels": {
    "msteams": {
      "serviceUrl": "https://smba.infra.gcc.teams.microsoft.com/teams"
    }
  }
}
```

مثال برای GCC High:

```json
{
  "channels": {
    "msteams": {
      "cloud": "USGov",
      "serviceUrl": "https://smba.infra.gov.teams.microsoft.us/teams"
    }
  }
}
```

`channels.msteams.serviceUrl` به میزبان‌های پشتیبانی‌شده Microsoft Teams Bot Connector محدود است. هنگامی که یک URL سرویس پیکربندی شده باشد، OpenClaw پیش از اجرای ارسال‌ها، ویرایش‌ها، حذف‌ها، کارت‌ها، نظرسنجی‌ها یا پاسخ‌های طولانی‌مدت صف‌شده بررسی می‌کند که `serviceUrl` گفت‌وگوی ذخیره‌شده از همان میزبان استفاده کند. با پیکربندی پیش‌فرض ابر عمومی، اگر یک گفت‌وگوی ذخیره‌شده به خارج از میزبان عمومی Teams Connector اشاره کند، OpenClaw به‌صورت بسته و امن از ادامه کار خودداری می‌کند. پس از تغییر تنظیمات ابر/URL سرویس، یک پیام تازه از گفت‌وگو دریافت کنید تا ارجاع ذخیره‌شده گفت‌وگو به‌روز باشد.

China/21Vianet در جدول نقاط پایانی پیش‌دستانه Teams متعلق به Microsoft هیچ URL عمومی و پیش‌دستانه جداگانه‌ای برای `smba` ندارد. ‏`cloud: "China"` را پیکربندی کنید تا SDK ‏Teams از نقاط پایانی احراز هویت، توکن و JWT متعلق به Azure China استفاده کند. سپس ارسال‌های پیش‌دستانه به یک ارجاع ذخیره‌شده گفت‌وگو از فعالیت ورودی China Teams، یا یک URL سرویس صریحاً پیکربندی‌شده، روی مرز کانال Azure China Bot Framework ‏(`*.botframework.azure.cn`) نیاز دارند. کمک‌کننده‌های Teams مبتنی بر Graph برای `cloud: "China"` غیرفعال هستند تا زمانی که OpenClaw درخواست‌های Graph را از طریق نقطه پایانی Azure China Graph مسیریابی کند.

### قالب‌بندی

Markdown در Teams نسبت به Slack یا Discord محدودتر است:

- قالب‌بندی پایه کار می‌کند: **پررنگ**، _مورب_، `code`، پیوندها.
- Markdown پیچیده (جدول‌ها، فهرست‌های تودرتو) ممکن است به‌درستی نمایش داده نشود.
- Adaptive Cards برای نظرسنجی‌ها و ارسال‌های ارائه معنایی پشتیبانی می‌شوند (پایین را ببینید).

## پیکربندی

تنظیمات کلیدی (برای الگوهای مشترک کانال، به [/gateway/configuration](/fa/gateway/configuration) مراجعه کنید):

- `channels.msteams.enabled`: کانال را فعال/غیرفعال می‌کند.
- `channels.msteams.appId`، `channels.msteams.appPassword`، `channels.msteams.tenantId`: اطلاعات اعتبارسنجی ربات.
- `channels.msteams.cloud`: محیط ابری SDK مربوط به Teams ‏(`Public`، `USGov`، `USGovDoD` یا `China`؛ پیش‌فرض `Public`). برای ابرهای SDK مربوط به USGov/DoD آن را با `serviceUrl` تنظیم کنید؛ چین از پیش‌تنظیم SDK و ارجاعات مکالمهٔ ذخیره‌شدهٔ Azure China Bot Framework استفاده می‌کند و تا زمان عرضهٔ مسیریابی Azure China Graph، کمک‌کننده‌های مبتنی بر Graph غیرفعال هستند.
- `channels.msteams.serviceUrl`: مرز URL سرویس Bot Connector برای عملیات پیش‌دستانهٔ SDK. ابر عمومی از پیش‌فرض SDK استفاده می‌کند؛ برای GCC ‏(`https://smba.infra.gcc.teams.microsoft.com/teams`)، GCC High یا DoD تنظیم کنید. هنگامی که ارجاع مکالمهٔ ذخیره‌شده از Teams تحت مدیریت 21Vianet آمده باشد، چین میزبان‌های کانال Azure China Bot Framework را می‌پذیرد.
- `channels.msteams.webhook.port` (پیش‌فرض `3978`).
- `channels.msteams.webhook.path` (پیش‌فرض `/api/messages`).
- `channels.msteams.dmPolicy`: `pairing | allowlist | open | disabled` (پیش‌فرض `pairing`).
- `channels.msteams.allowFrom`: فهرست مجاز پیام مستقیم (شناسه‌های شیء AAD توصیه می‌شوند). هنگامی که دسترسی Graph موجود باشد، راه‌انداز در حین پیکربندی نام‌ها را به شناسه‌ها تبدیل می‌کند.
- `channels.msteams.dangerouslyAllowNameMatching`: کلید اضطراری برای فعال‌سازی دوبارهٔ تطبیق تغییرپذیر UPN/نام نمایشی و مسیریابی مستقیم نام تیم/کانال.
- `channels.msteams.textChunkLimit`: اندازهٔ قطعه‌های متن خروجی برحسب نویسه (پیش‌فرض `4000` و صرف‌نظر از مقدار پیکربندی‌شدهٔ بالاتر، دارای سقف قطعی `4000`).
- `channels.msteams.streaming.chunkMode`: ‏`length` (پیش‌فرض) یا `newline` برای تقسیم در خطوط خالی (مرز پاراگراف‌ها) پیش از قطعه‌بندی براساس طول.
- `channels.msteams.mediaAllowHosts`: فهرست مجاز میزبان‌ها برای پیوست‌های ورودی (به‌طور پیش‌فرض دامنه‌های Microsoft/Teams: ‏Graph، SharePoint/OneDrive، ‏Teams CDN، ‏Bot Framework و Azure Media Services).
- `channels.msteams.mediaAuthAllowHosts`: فهرست مجاز برای افزودن سرآیندهای Authorization هنگام تلاش مجدد برای دریافت رسانه (به‌طور پیش‌فرض میزبان‌های Graph و Bot Framework).
- `channels.msteams.graphMediaFallback`: فعال‌سازی جست‌وجوی پیام از طریق Graph هنگامی که HTML کانال/گروه نشانگرهای فایل را ندارد (پیش‌فرض `false`؛ به [بازیابی فایل کانال/گروه](#channelgroup-file-recovery-graphmediafallback) مراجعه کنید).
- `channels.msteams.mediaMaxMb`: بازنویسی محدودیت اندازهٔ رسانه برای هر کانال برحسب MB. اگر تنظیم نشده باشد، از `agents.defaults.mediaMaxMb` استفاده می‌شود.
- `channels.msteams.requireMention`: الزام @mention در کانال‌ها/گروه‌ها (پیش‌فرض `true`).
- `channels.msteams.replyStyle`: ‏`thread | top-level` (به [سبک پاسخ](#reply-style-threads-vs-posts) مراجعه کنید).
- `channels.msteams.teams.<teamId>.replyStyle`: بازنویسی برای هر تیم.
- `channels.msteams.teams.<teamId>.requireMention`: بازنویسی برای هر تیم.
- `channels.msteams.teams.<teamId>.tools`: بازنویسی‌های پیش‌فرض خط‌مشی ابزار برای هر تیم (`allow`/`deny`/`alsoAllow`) که در نبود بازنویسی کانال استفاده می‌شوند.
- `channels.msteams.teams.<teamId>.toolsBySender`: بازنویسی‌های پیش‌فرض خط‌مشی ابزار برای هر فرستنده در هر تیم (نویسهٔ عام `"*"` پشتیبانی می‌شود).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`: بازنویسی برای هر کانال.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.requireMention`: بازنویسی برای هر کانال.
- `channels.msteams.teams.<teamId>.channels.<conversationId>.tools`: بازنویسی‌های خط‌مشی ابزار برای هر کانال (`allow`/`deny`/`alsoAllow`).
- `channels.msteams.teams.<teamId>.channels.<conversationId>.toolsBySender`: بازنویسی‌های خط‌مشی ابزار برای هر فرستنده در هر کانال (نویسهٔ عام `"*"` پشتیبانی می‌شود).
- کلیدهای `toolsBySender` باید از پیشوندهای صریح استفاده کنند: `channel:`، `id:`، `e164:`، `username:`، `name:` (کلیدهای قدیمی بدون پیشوند همچنان فقط به `id:` نگاشت می‌شوند).
- `channels.msteams.authType`: نوع احراز هویت — `"secret"` (پیش‌فرض) یا `"federated"`.
- `channels.msteams.certificatePath`: مسیر فایل گواهی PEM (احراز هویت فدرال + گواهی).
- `channels.msteams.certificateThumbprint`: اثر انگشت گواهی؛ پذیرفته می‌شود، اما برای احراز هویت الزامی نیست.
- `channels.msteams.useManagedIdentity`: فعال‌سازی احراز هویت با هویت مدیریت‌شده (حالت فدرال).
- `channels.msteams.managedIdentityClientId`: شناسهٔ کلاینت برای هویت مدیریت‌شدهٔ تخصیص‌یافته به کاربر.
- `channels.msteams.sharePointSiteId`: شناسهٔ سایت SharePoint برای بارگذاری فایل در گفت‌وگوهای گروهی/کانال‌ها (به [ارسال فایل در گفت‌وگوهای گروهی](#sending-files-in-group-chats) مراجعه کنید).
- `channels.msteams.welcomeCard`، `channels.msteams.groupWelcomeCard`، `channels.msteams.promptStarters`: کارت تطبیقی خوشامدگویی که در نخستین تماس پیام مستقیم/گروهی نمایش داده می‌شود، به‌همراه دکمه‌های درخواست پیشنهادی آن.
- `channels.msteams.responsePrefix`: متنی که به ابتدای پاسخ‌های خروجی افزوده می‌شود.
- `channels.msteams.feedbackEnabled` (پیش‌فرض `true`)، `channels.msteams.feedbackReflection` (پیش‌فرض `true`)، `channels.msteams.feedbackReflectionCooldownMs`: بازخورد پسندیدن/نپسندیدن برای پاسخ‌ها و پیگیری تأملی پس از بازخورد منفی.
- `channels.msteams.sso`، `channels.msteams.delegatedAuth`: اتصال OAuth در Bot Framework و دامنه‌های واگذارشدهٔ Graph برای جریان‌های مبتنی بر SSO؛ ‏`sso.enabled: true` به `sso.connectionName` نیاز دارد.

## مسیریابی و نشست‌ها

- کلیدهای نشست از قالب استاندارد عامل پیروی می‌کنند (به [/concepts/session](/fa/concepts/session) مراجعه کنید):
  - پیام‌های مستقیم نشست اصلی را به‌اشتراک می‌گذارند (`agent:<agentId>:<mainKey>`).
  - پیام‌های کانال/گروه از شناسهٔ مکالمه استفاده می‌کنند:
    - `agent:<agentId>:msteams:channel:<conversationId>`
    - `agent:<agentId>:msteams:group:<conversationId>`

## سبک پاسخ: رشته‌ها در برابر پست‌ها

Teams برای یک مدل دادهٔ زیربنایی یکسان، دو سبک رابط کاربری کانال دارد:

| سبک                     | توضیحات                                                     | `replyStyle` توصیه‌شده |
| ------------------------ | ----------------------------------------------------------- | ------------------------ |
| **Posts** (کلاسیک)      | پیام‌ها به‌شکل کارت‌هایی با پاسخ‌های رشته‌ای در زیر ظاهر می‌شوند | `thread` (پیش‌فرض)       |
| **Threads** (مشابه Slack) | پیام‌ها به‌صورت خطی و بیشتر شبیه Slack جریان می‌یابند       | `top-level`              |

**مشکل:** ‏API مربوط به Teams مشخص نمی‌کند که کانال از کدام سبک رابط کاربری استفاده می‌کند. اگر از `replyStyle` نادرست استفاده کنید:

- `thread` در کانالی با سبک Threads ← پاسخ‌ها به‌شکل تودرتوی نامناسب ظاهر می‌شوند.
- `top-level` در کانالی با سبک Posts ← پاسخ‌ها به‌جای قرارگرفتن در رشته، به‌صورت پست‌های سطح‌بالای جداگانه ظاهر می‌شوند.

**راه‌حل:** ‏`replyStyle` را براساس نحوهٔ پیکربندی کانال، برای هر کانال تنظیم کنید:

```json5
{
  channels: {
    msteams: {
      replyStyle: "thread",
      teams: {
        "19:abc...@thread.tacv2": {
          channels: {
            "19:xyz...@thread.tacv2": {
              replyStyle: "top-level",
            },
          },
        },
      },
    },
  },
}
```

### تقدم تفکیک مقدار

هنگامی که ربات پاسخی را به یک کانال می‌فرستد، `replyStyle` از اختصاصی‌ترین بازنویسی تا مقدار پیش‌فرض تفکیک می‌شود. نخستین مقدار غیر از `undefined` برنده است:

1. **برای هر کانال** — `channels.msteams.teams.<teamId>.channels.<conversationId>.replyStyle`
2. **برای هر تیم** — `channels.msteams.teams.<teamId>.replyStyle`
3. **سراسری** — `channels.msteams.replyStyle`
4. **پیش‌فرض ضمنی** — برگرفته از `requireMention`:
   - `requireMention: true` ← `thread`
   - `requireMention: false` ← `top-level`

اگر `requireMention: false` را بدون `replyStyle` صریح به‌صورت سراسری تنظیم کنید، اشاره‌ها در کانال‌های با سبک Posts حتی وقتی پیام ورودی پاسخی در رشته بوده است، به‌شکل پست‌های سطح‌بالا ظاهر می‌شوند. برای جلوگیری از رفتارهای غیرمنتظره، `replyStyle: "thread"` را در سطح سراسری، تیم یا کانال تثبیت کنید.

برای ارسال‌های پیش‌دستانه به یک مکالمهٔ کانال ذخیره‌شده (پاسخ‌های صف‌شدهٔ فراخوانی ابزار، عامل‌های طولانی‌مدت)، همان تفکیک تیم/کانال اعمال می‌شود؛ گفت‌وگوهای گروهی و مکالمات شخصی (پیام مستقیم)، صرف‌نظر از `replyStyle`، همیشه برای ارسال‌های پیش‌دستانه به `top-level` تفکیک می‌شوند.

### حفظ زمینهٔ رشته

هنگامی که `replyStyle: "thread"` برقرار است و ربات از داخل یک رشتهٔ کانال @mention شده باشد، OpenClaw ریشهٔ اصلی رشته را دوباره به ارجاع مکالمهٔ خروجی (`19:...@thread.tacv2;messageid=<root>`) متصل می‌کند تا پاسخ در همان رشته قرار گیرد. این رفتار هم برای ارسال‌های زنده (در همان نوبت) و هم ارسال‌های پیش‌دستانه‌ای برقرار است که پس از انقضای زمینهٔ نوبت Bot Framework انجام می‌شوند (برای مثال، عامل‌های طولانی‌مدت و پاسخ‌های صف‌شدهٔ فراخوانی ابزار از طریق `mcp__openclaw__message`).

ریشهٔ رشته از `threadId` ذخیره‌شده در ارجاع مکالمه گرفته می‌شود. ارجاعات ذخیره‌شدهٔ قدیمی‌تر که مربوط به پیش از `threadId` هستند، به `activityId` بازمی‌گردند (هر فعالیت ورودی که آخرین بار مکالمه را مقداردهی اولیه کرده است)؛ بنابراین استقرارهای موجود بدون مقداردهی مجدد همچنان کار می‌کنند.

هنگامی که `replyStyle: "top-level"` برقرار است، به ورودی‌های رشتهٔ کانال عمداً به‌شکل پست‌های سطح‌بالای جدید پاسخ داده می‌شود و هیچ پسوند رشته‌ای متصل نمی‌شود. این رفتار برای کانال‌های با سبک Threads صحیح است؛ مشاهدهٔ پست‌های سطح‌بالا در جایی که انتظار پاسخ‌های رشته‌ای داشتید، به این معناست که `replyStyle` برای آن کانال نادرست تنظیم شده است.

## پیوست‌ها و تصاویر

**محدودیت‌های فعلی:**

- **پیام‌های مستقیم:** تصاویر و پیوست‌های فایل از طریق APIهای فایل ربات Teams کار می‌کنند.
- **کانال‌ها/گروه‌ها:** پیوست‌ها در فضای ذخیره‌سازی M365 ‏(SharePoint/OneDrive) قرار دارند. محتوای Webhook فقط شامل یک قطعهٔ HTML است، نه بایت‌های واقعی فایل. **برای دانلود پیوست‌های کانال، مجوزهای Graph API الزامی هستند**.
- برای ارسال‌های صریحی که فایل در آن‌ها اولویت دارد، از `action=upload-file` همراه با `media` / `filePath` / `path` استفاده کنید؛ `message` اختیاری به متن/نظر همراه تبدیل می‌شود و `filename` (یا `title`) نام فایل بارگذاری‌شده را بازنویسی می‌کند.

بدون مجوزهای Graph، پیام‌های کانال دارای تصویر فقط به‌شکل متن دریافت می‌شوند (محتوای تصویر برای ربات قابل‌دسترسی نیست).
OpenClaw به‌طور پیش‌فرض رسانه را فقط از نام‌های میزبان Microsoft/Teams دانلود می‌کند. با `channels.msteams.mediaAllowHosts` بازنویسی کنید (برای مجازکردن هر میزبانی از `["*"]` استفاده کنید).
سرآیندهای Authorization فقط برای میزبان‌های موجود در `channels.msteams.mediaAuthAllowHosts` افزوده می‌شوند (به‌طور پیش‌فرض میزبان‌های Graph و Bot Framework). این فهرست را محدود نگه دارید (از پسوندهای چندمستأجری اجتناب کنید).

## ارسال فایل در گفت‌وگوهای گروهی

ربات‌ها می‌توانند با استفاده از جریان داخلی FileConsentCard فایل‌ها را در پیام‌های مستقیم ارسال کنند. **ارسال فایل در گفت‌وگوهای گروهی/کانال‌ها** به پیکربندی بیشتری نیاز دارد:

| زمینه                    | نحوهٔ ارسال فایل‌ها                              | پیکربندی لازم                                      |
| ------------------------ | ------------------------------------------------ | ------------------------------------------------- |
| **پیام‌های مستقیم**      | FileConsentCard ← کاربر می‌پذیرد ← ربات بارگذاری می‌کند | بدون پیکربندی اضافی کار می‌کند                    |
| **گفت‌وگوهای گروهی/کانال‌ها** | بارگذاری در SharePoint ← کارت بومی فایل          | به `sharePointSiteId` + مجوزهای Graph نیاز دارد |
| **تصاویر (هر زمینه‌ای)** | درون‌خطی با کدگذاری Base64                       | بدون پیکربندی اضافی کار می‌کند                    |

### چرا گفت‌وگوهای گروهی به SharePoint نیاز دارند

ربات‌ها از هویت برنامه استفاده می‌کنند، درحالی‌که منبع `/me` در Microsoft Graph ‏[به کاربر واردشده نیاز دارد](https://learn.microsoft.com/en-us/graph/api/user-get?view=graph-rest-1.0). برای ارسال فایل در گفت‌وگوهای گروهی/کانال‌ها، ربات فایل را در یک **سایت SharePoint** بارگذاری می‌کند و پیوند اشتراک‌گذاری می‌سازد.

### پیکربندی

1. **مجوزهای Graph API را اضافه کنید** در Entra ID (Azure AD) ← App Registration:
   - `Sites.ReadWrite.All` (Application) — بارگذاری فایل‌ها در SharePoint.
   - `ChatMember.Read.All` (Application) — مجوز حداقلی در سطح مستأجر برای ارسال فایل در گفت‌وگوی گروهی. `Chat.Read.All` نیز کار می‌کند و هنگامی که تاریخچهٔ گفت‌وگوی گروهی فعال باشد، از قبل این مورد را پوشش می‌دهد. به‌عنوان جایگزین برای هر گفت‌وگو، از [مجوز رضایت مختص منبع](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent) ‏`ChatMember.Read.Chat` استفاده کنید.
2. **رضایت مدیر را** برای مستأجر اعطا کنید.
3. **شناسهٔ سایت SharePoint خود را دریافت کنید:**

   ```bash
   # از طریق Graph Explorer یا curl با یک توکن معتبر:
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/{hostname}:/{site-path}"

   # مثال: برای سایتی در "contoso.sharepoint.com/sites/BotFiles"
   curl -H "Authorization: Bearer $TOKEN" \
     "https://graph.microsoft.com/v1.0/sites/contoso.sharepoint.com:/sites/BotFiles"

   # پاسخ شامل این مورد است: "id": "contoso.sharepoint.com,guid1,guid2"
   ```

4. **پیکربندی OpenClaw:**

   ```json5
   {
     channels: {
       msteams: {
         // ... سایر تنظیمات ...
         sharePointSiteId: "contoso.sharepoint.com,guid1,guid2",
       },
     },
   }
   ```

### رفتار اشتراک‌گذاری

| زمینه و مجوز                                                  | رفتار اشتراک‌گذاری                                          |
| ----------------------------------------------------------------------- | --------------------------------------------------------- |
| کانال + `Sites.ReadWrite.All`                                         | پیوند اشتراک‌گذاری در سطح سازمان (همه افراد سازمان می‌توانند دسترسی داشته باشند) |
| گفت‌وگوی گروهی + `Sites.ReadWrite.All` + مجوز خواندن پشتیبانی‌شده برای اعضای گفت‌وگو | پیوند اشتراک‌گذاری به‌ازای هر کاربر (فقط اعضای گفت‌وگو می‌توانند دسترسی داشته باشند)      |
| گفت‌وگوی گروهی بدون مجوز خواندن پشتیبانی‌شده برای اعضای گفت‌وگو                   | ارسال به‌صورت بسته ناموفق می‌شود                                         |

اشتراک‌گذاری به‌ازای هر کاربر امن‌تر است، زیرا فقط شرکت‌کنندگان گفت‌وگو می‌توانند به فایل دسترسی داشته باشند. OpenClaw برای گفت‌وگوهای گروهی به جست‌وجوی موفق اعضا نیاز دارد؛ پایان مهلت، خرابی‌های انتقال، نتایج خالی و رد درخواست توسط Graph API به‌جای گسترش دسترسی به کل سازمان، باعث ناموفق‌شدن ارسال می‌شوند.

### رفتار جایگزین

| سناریو                                                         | نتیجه                                           |
| ---------------------------------------------------------------- | ------------------------------------------------ |
| گفت‌وگوی گروهی + فایل + مجوزهای SharePoint و اعضا پیکربندی شده‌اند | بارگذاری در SharePoint و ارسال کارت بومی فایل    |
| گفت‌وگوی گروهی + فایل + مجوزهای SharePoint یا اعضا موجود نیستند     | شکست با خطای پیکربندی قابل‌اقدام      |
| کانال + فایل + `sharePointSiteId` پیکربندی شده است                   | بارگذاری در SharePoint و ارسال کارت بومی فایل    |
| گفت‌وگوی شخصی + فایل                                             | جریان FileConsentCard (بدون SharePoint کار می‌کند)  |
| هر زمینه‌ای + تصویر                                              | درون‌خطی با کدگذاری Base64 (بدون SharePoint کار می‌کند) |

### محل ذخیره فایل‌ها

فایل‌های بارگذاری‌شده در پوشه‌ای به نام `/OpenClawShared/` در کتابخانه پیش‌فرض اسناد سایت SharePoint پیکربندی‌شده ذخیره می‌شوند.

## نظرسنجی‌ها (Adaptive Cards)

OpenClaw نظرسنجی‌های Teams را به‌شکل Adaptive Cards ارسال می‌کند (API بومی برای نظرسنجی Teams وجود ندارد).

- CLI: `openclaw message poll --channel msteams --target conversation:<id> --poll-question "..." --poll-option "..." --poll-option "..."`.
- رأی‌ها توسط Gateway در SQLite وضعیت Plugin مربوط به OpenClaw در مسیر `state/openclaw.sqlite` ثبت می‌شوند.
- فایل‌های موجود `msteams-polls.json` توسط `openclaw doctor --fix` وارد می‌شوند، نه توسط Plugin در حال اجرا.
- برای ثبت رأی‌ها، Gateway باید آنلاین بماند.
- نظرسنجی‌ها خلاصه نتایج را به‌طور خودکار ارسال نمی‌کنند و هنوز CLI برای نتایج نظرسنجی وجود ندارد.

## کارت‌های ارائه

با استفاده از ابزار `message`، CLI یا تحویل عادی پاسخ، بارهای معنایی ارائه را برای کاربران یا مکالمات Teams ارسال کنید. OpenClaw آن‌ها را بر اساس قرارداد عمومی ارائه، به‌شکل Adaptive Cards در Teams رندر می‌کند.

پارامتر `presentation` بلوک‌های معنایی را می‌پذیرد. وقتی `presentation` ارائه شده باشد، متن پیام اختیاری است. دکمه‌ها به‌شکل اقدامات ارسال Adaptive Card یا URL رندر می‌شوند. منوهای انتخاب در رندرکننده Teams بومی نیستند؛ بنابراین OpenClaw پیش از تحویل، آن‌ها را به متن خوانا تبدیل می‌کند.

**ابزار عامل:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:<id>",
  presentation: {
    title: "سلام",
    blocks: [{ type: "text", text: "سلام!" }],
  },
}
```

**CLI:**

```bash
openclaw message send --channel msteams \
  --target "conversation:19:abc...@thread.tacv2" \
  --presentation '{"title":"سلام","blocks":[{"type":"text","text":"سلام!"}]}'
```

برای جزئیات قالب مقصد، بخش [قالب‌های مقصد](#target-formats) را در ادامه ببینید.

## قالب‌های مقصد

مقصدهای MSTeams برای تمایز میان کاربران و مکالمات از پیشوندها استفاده می‌کنند:

| نوع مقصد         | قالب                           | مثال                                                                                                |
| ------------------- | -------------------------------- | ------------------------------------------------------------------------------------------------------ |
| کاربر (با شناسه)        | `user:<aad-object-id>`           | `user:40a1a0ed-4ff2-4164-a219-55518990c197`                                                            |
| کاربر (با نام)      | `user:<display-name>`            | `user:John Smith` (به Graph API نیاز دارد)                                                                 |
| گروه/کانال       | `conversation:<conversation-id>` | `conversation:19:abc123...@thread.tacv2`                                                               |
| گروه/کانال (خام) | `<conversation-id>`              | `19:abc123...@thread.tacv2`، `19:...@unq.gbl.spaces`، یا یک شناسه خام `a:`/`8:orgid:`/`29:` مربوط به Bot Framework |

**نمونه‌های CLI:**

```bash
# ارسال به کاربر با شناسه
openclaw message send --channel msteams --target "user:40a1a0ed-..." --message "سلام"

# ارسال به کاربر با نام نمایشی (جست‌وجوی Graph API را فعال می‌کند)
openclaw message send --channel msteams --target "user:John Smith" --message "سلام"

# ارسال به گفت‌وگوی گروهی یا کانال
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" --message "سلام"

# ارسال کارت ارائه به یک مکالمه
openclaw message send --channel msteams --target "conversation:19:abc...@thread.tacv2" \
  --presentation '{"title":"سلام","blocks":[{"type":"text","text":"سلام"}]}'
```

**نمونه‌های ابزار عامل:**

```json5
{
  action: "send",
  channel: "msteams",
  target: "user:John Smith",
  message: "سلام!",
}
```

```json5
{
  action: "send",
  channel: "msteams",
  target: "conversation:19:abc...@thread.tacv2",
  presentation: {
    title: "سلام",
    blocks: [{ type: "text", text: "سلام" }],
  },
}
```

<Note>
بدون پیشوند `user:`، نام‌ها به‌طور پیش‌فرض به‌عنوان گروه یا تیم تفکیک می‌شوند. هنگام هدف‌گیری افراد با نام نمایشی، همیشه از `user:` استفاده کنید.
</Note>

## پیام‌رسانی پیش‌دستانه

- پیام‌های پیش‌دستانه فقط **پس از** تعامل کاربر امکان‌پذیرند، زیرا OpenClaw در آن زمان ارجاعات مکالمه را ذخیره می‌کند.
- برای `dmPolicy` و محدودسازی با فهرست مجاز، [/gateway/configuration](/fa/gateway/configuration) را ببینید.

## شناسه‌های تیم و کانال (اشتباه رایج)

پارامتر پرس‌وجوی `groupId` در URLهای Teams، شناسه تیم مورد استفاده برای پیکربندی **نیست**. در عوض، شناسه‌ها را از مسیر URL استخراج کنید:

**URL تیم:**

```text
https://teams.microsoft.com/l/team/19%3ABk4j...%40thread.tacv2/conversations?groupId=...
                                    └────────────────────────────┘
                                    شناسه مکالمه تیم (این مورد را URL-decode کنید)
```

**URL کانال:**

```text
https://teams.microsoft.com/l/channel/19%3A15bc...%40thread.tacv2/ChannelName?groupId=...
                                      └─────────────────────────┘
                                      شناسه کانال (این مورد را URL-decode کنید)
```

**برای پیکربندی:**

- کلید تیم = بخش مسیر پس از `/team/` (URL-decoded، برای مثال `19:Bk4j...@thread.tacv2`؛ مستأجرهای قدیمی‌تر ممکن است `@thread.skype` را نمایش دهند که آن نیز معتبر است).
- کلید کانال = بخش مسیر پس از `/channel/` (URL-decoded).
- برای مسیریابی OpenClaw، پارامتر پرس‌وجوی `groupId` را **نادیده بگیرید**. این شناسه گروه Microsoft Entra است، نه شناسه مکالمه Bot Framework که در فعالیت‌های ورودی Teams استفاده می‌شود.

## کانال‌های خصوصی

ربات‌ها در کانال‌های خصوصی پشتیبانی محدودی دارند:

| قابلیت                      | کانال‌های استاندارد | کانال‌های خصوصی       |
| ---------------------------- | ----------------- | ---------------------- |
| نصب ربات             | بله               | محدود                |
| پیام‌های بی‌درنگ (Webhook) | بله               | ممکن است کار نکند           |
| مجوزهای RSC              | بله               | ممکن است رفتار متفاوتی داشته باشد |
| @mentions                    | بله               | اگر ربات در دسترس باشد   |
| تاریخچه Graph API            | بله               | بله (با مجوزها) |

**راه‌حل‌های جایگزین در صورت کارنکردن کانال‌های خصوصی:**

1. برای تعامل با ربات از کانال‌های استاندارد استفاده کنید.
2. از پیام‌های مستقیم استفاده کنید؛ کاربران همیشه می‌توانند مستقیماً به ربات پیام دهند.
3. برای دسترسی به تاریخچه از Graph API استفاده کنید (به `ChannelMessage.Read.All` نیاز دارد).

## عیب‌یابی

### مشکلات رایج

- **تصاویر در کانال‌ها نمایش داده نمی‌شوند:** مجوزهای Graph یا رضایت مدیر موجود نیست. برنامه Teams را دوباره نصب کنید و Teams را کاملاً ببندید و دوباره باز کنید.
- **در کانال پاسخی دریافت نمی‌شود:** اشاره‌ها به‌طور پیش‌فرض الزامی‌اند؛ `channels.msteams.requireMention=false` را تنظیم کنید یا برای هر تیم/کانال پیکربندی جداگانه انجام دهید.
- **عدم تطابق نسخه (Teams همچنان مانیفست قدیمی را نمایش می‌دهد):** برنامه را حذف و دوباره اضافه کنید و برای تازه‌سازی، Teams را کاملاً ببندید.
- **خطای 401 Unauthorized از Webhook:** هنگام آزمایش دستی بدون Azure JWT قابل‌انتظار است؛ یعنی نقطه پایانی در دسترس است، اما احراز هویت ناموفق بوده است. برای آزمایش صحیح از Azure Web Chat استفاده کنید.

### خطاهای بارگذاری مانیفست

- **"Icon file cannot be empty":** مانیفست به فایل‌های آیکونی ارجاع می‌دهد که 0 بایت هستند. آیکون‌های PNG معتبر ایجاد کنید (32x32 برای `outline.png` و 192x192 برای `color.png`).
- **"webApplicationInfo.Id already in use":** برنامه همچنان در تیم/گفت‌وگوی دیگری نصب است. ابتدا آن را پیدا و حذف نصب کنید یا 5-10 دقیقه برای انتشار تغییرات منتظر بمانید.
- **نمایش "Something went wrong" هنگام بارگذاری:** در عوض از طریق [https://admin.teams.microsoft.com](https://admin.teams.microsoft.com) بارگذاری کنید، DevTools مرورگر را باز کنید (F12) → زبانه Network، و بدنه پاسخ را برای مشاهده خطای واقعی بررسی کنید.
- **Sideload ناموفق است:** به‌جای "Upload a custom app"، گزینه "Upload an app to your org's app catalog" را امتحان کنید؛ این کار اغلب محدودیت‌های sideload را دور می‌زند.

### مجوزهای RSC کار نمی‌کنند

1. بررسی کنید که `webApplicationInfo.id` دقیقاً با App ID ربات مطابقت داشته باشد.
2. برنامه را دوباره بارگذاری و در تیم/گفت‌وگو مجدداً نصب کنید.
3. بررسی کنید آیا مدیر سازمان مجوزهای RSC را مسدود کرده است.
4. تأیید کنید که از دامنه درست استفاده می‌کنید: `ChannelMessage.Read.Group` برای تیم‌ها و `ChatMessage.Read.Chat` برای گفت‌وگوهای گروهی.

## منابع

- [ایجاد Azure Bot](https://learn.microsoft.com/en-us/azure/bot-service/bot-service-quickstart-registration) - راهنمای راه‌اندازی Azure Bot
- [Teams Developer Portal](https://dev.teams.microsoft.com/apps) - ایجاد/مدیریت برنامه‌های Teams
- [طرح‌واره مانیفست برنامه Teams](https://learn.microsoft.com/en-us/microsoftteams/platform/resources/schema/manifest-schema)
- [دریافت پیام‌های کانال با RSC](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/channel-messages-with-rsc)
- [مرجع مجوزهای RSC](https://learn.microsoft.com/en-us/microsoftteams/platform/graph-api/rsc/resource-specific-consent)
- [مدیریت فایل ربات Teams](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/bots-filesv4) (کانال/گروه به Graph نیاز دارد)
- [پیام‌رسانی پیش‌دستانه](https://learn.microsoft.com/en-us/microsoftteams/platform/bots/how-to/conversations/send-proactive-messages)
- [@microsoft/teams.cli](https://www.npmjs.com/package/@microsoft/teams.cli) - CLI تیمز برای مدیریت ربات

## مرتبط

- [نمای کلی کانال‌ها](/fa/channels) - همه کانال‌های پشتیبانی‌شده
- [جفت‌سازی](/fa/channels/pairing) - احراز هویت پیام مستقیم و جریان جفت‌سازی
- [گروه‌ها](/fa/channels/groups) - رفتار گفت‌وگوی گروهی و کنترل دسترسی بر اساس اشاره
- [مسیریابی کانال](/fa/channels/channel-routing) - مسیریابی نشست برای پیام‌ها
- [امنیت](/fa/gateway/security) - مدل دسترسی و مقاوم‌سازی
