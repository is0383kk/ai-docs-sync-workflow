---
read_when:
    - کار روی قابلیت‌های کانال Google Chat
summary: وضعیت پشتیبانی، قابلیت‌ها و پیکربندی برنامه Google Chat
title: Google Chat
x-i18n:
    generated_at: "2026-07-27T16:13:56Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9d3fb96564294b57040327bb21ab7331bf8412eb04f879a9c7ea1018ba2bddab
    source_path: channels/googlechat.md
    workflow: 16
---

Google Chat به‌عنوان Plugin رسمی `@openclaw/googlechat` اجرا می‌شود: پیام‌های مستقیم و فضاها از طریق Webhookهای Google Chat API (فقط نقطه پایانی HTTP، بدون Pub/Sub).

## نصب

```bash
openclaw plugins install @openclaw/googlechat
```

نسخه محلی مخزن (هنگام اجرا از یک مخزن git):

```bash
openclaw plugins install ./path/to/local/googlechat-plugin
```

## راه‌اندازی سریع (مبتدی)

1. یک پروژه Google Cloud ایجاد و **Google Chat API** را فعال کنید.
   - به این نشانی بروید: [اعتبارنامه‌های Google Chat API](https://console.cloud.google.com/apis/api/chat.googleapis.com/credentials)
   - اگر API از قبل فعال نیست، آن را فعال کنید.
2. یک **Service Account** ایجاد کنید:
   - روی **Create Credentials** > **Service Account** کلیک کنید.
   - هر نامی که می‌خواهید برای آن انتخاب کنید (برای مثال، `openclaw-chat`).
   - مجوزها و حساب‌های اصلی را خالی بگذارید (**Continue** و سپس **Done**).
3. **کلید JSON** را ایجاد و دانلود کنید:
   - روی حساب سرویس جدید کلیک کنید > زبانه **Keys** > **Add Key** > **Create new key** > **JSON** > **Create**.
4. فایل JSON دانلودشده را روی میزبان Gateway خود ذخیره کنید (برای مثال، `~/.openclaw/googlechat-service-account.json`).
5. یک برنامه Google Chat در [پیکربندی Chat در Google Cloud Console](https://console.cloud.google.com/apis/api/chat.googleapis.com/hangouts-chat) ایجاد کنید:
   - بخش **Application info** (نام برنامه، URL آواتار، توضیحات) را تکمیل کنید.
   - گزینه **Interactive features** را فعال کنید.
   - در بخش **Functionality**، گزینه **Join spaces and group conversations** را علامت بزنید.
   - در بخش **Connection settings**، گزینه **HTTP endpoint URL** را انتخاب کنید.
   - در بخش **Triggers**، گزینه **Use a common HTTP endpoint URL for all triggers** را انتخاب کنید و آن را روی URL عمومی Gateway خود به‌همراه `/googlechat` تنظیم کنید (به [URL عمومی](#public-url-webhook-only) مراجعه کنید).
   - در بخش **Visibility**، گزینه **Make this Chat app available to specific people and groups in `<Your Domain>`** را علامت بزنید و نشانی ایمیل خود را وارد کنید.
   - روی **Save** کلیک کنید.
6. وضعیت برنامه را فعال کنید: صفحه را تازه‌سازی کنید، **App status** را پیدا کنید، آن را روی **Live - available to users** تنظیم کنید و دوباره **Save** را بزنید.
7. OpenClaw را با حساب سرویس و مخاطب Webhook پیکربندی کنید (باید با پیکربندی برنامه Chat مطابقت داشته باشد):
   - متغیر محیطی: `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE=/path/to/service-account.json` (فقط حساب پیش‌فرض)، یا
   - پیکربندی: به [نکات برجسته پیکربندی](#config-highlights) مراجعه کنید. `openclaw channels add --channel googlechat` همچنین `--audience-type`، `--audience`، `--webhook-path` و `--webhook-url` را می‌پذیرد.
8. Gateway را راه‌اندازی کنید. Google Chat درخواست‌های POST را به مسیر Webhook شما ارسال می‌کند (پیش‌فرض `/googlechat`).

## افزودن به Google Chat

پس از اجرای Gateway و قرار گرفتن ایمیل شما در فهرست دسترسی:

1. به [Google Chat](https://chat.google.com/) بروید.
2. روی نماد **+** (به‌علاوه) کنار **Direct Messages** کلیک کنید.
3. **App name** پیکربندی‌شده در Google Cloud Console را جست‌وجو کنید.
   - ربات در فهرست مرور Marketplace ظاهر _نمی‌شود_، زیرا یک برنامه خصوصی است؛ آن را با نام جست‌وجو کنید.
4. ربات را انتخاب کنید، روی **Add** یا **Chat** کلیک کنید و پیامی بفرستید.

## URL عمومی (فقط Webhook)

Webhookهای Google Chat به یک نقطه پایانی عمومی HTTPS نیاز دارند. برای امنیت، **فقط مسیر `/googlechat`** را در اینترنت در دسترس قرار دهید و داشبورد OpenClaw و سایر نقاط پایانی را خصوصی نگه دارید.

### گزینه A: Tailscale Funnel (پیشنهادی)

از Tailscale Serve برای داشبورد خصوصی و از Funnel برای مسیر عمومی Webhook استفاده کنید.

1. بررسی کنید Gateway به چه نشانی متصل است:

   ```bash
   ss -tlnp | grep 18789
   ```

   IP را یادداشت کنید (برای مثال، `127.0.0.1`، `0.0.0.0` یا یک نشانی Tailscale `100.x.x.x`).

2. داشبورد را فقط برای tailnet در دسترس قرار دهید (درگاه 8443):

   ```bash
   # اگر به localhost متصل است (127.0.0.1 یا 0.0.0.0):
   tailscale serve --bg --https 8443 http://127.0.0.1:18789

   # اگر فقط به یک IP مربوط به Tailscale متصل است:
   tailscale serve --bg --https 8443 http://100.x.x.x:18789
   ```

3. فقط مسیر Webhook را به‌صورت عمومی در دسترس قرار دهید:

   ```bash
   # اگر به localhost متصل است (127.0.0.1 یا 0.0.0.0):
   tailscale funnel --bg --set-path /googlechat http://127.0.0.1:18789/googlechat

   # اگر فقط به یک IP مربوط به Tailscale متصل است:
   tailscale funnel --bg --set-path /googlechat http://100.x.x.x:18789/googlechat
   ```

4. اگر از شما خواسته شد، برای فعال‌کردن Funnel برای این Node، به URL مجوزدهی نمایش‌داده‌شده در خروجی بروید.

5. بررسی کنید:

   ```bash
   tailscale serve status
   tailscale funnel status
   ```

URL عمومی Webhook شما `https://<node-name>.<tailnet>.ts.net/googlechat` است؛ داشبورد در `https://<node-name>.<tailnet>.ts.net:8443/` فقط در tailnet باقی می‌ماند. از URL عمومی (بدون `:8443`) در پیکربندی برنامه Google Chat استفاده کنید.

> توجه: این پیکربندی پس از راه‌اندازی مجدد نیز حفظ می‌شود. بعداً آن را با `tailscale funnel reset` و `tailscale serve reset` حذف کنید.

### گزینه B: پراکسی معکوس (Caddy)

فقط مسیر Webhook را پراکسی کنید:

```caddy
your-domain.com {
    reverse_proxy /googlechat* localhost:18789
}
```

درخواست‌های ارسالی به `your-domain.com/` نادیده گرفته می‌شوند یا پاسخ 404 دریافت می‌کنند، درحالی‌که `your-domain.com/googlechat` به OpenClaw هدایت می‌شود.

### گزینه C: Cloudflare Tunnel

قواعد ورودی تونل را طوری پیکربندی کنید که فقط مسیر Webhook را هدایت کنند:

- **Path**: `/googlechat` -> `http://localhost:18789/googlechat`
- **Default rule**: HTTP 404 (Not Found)

## نحوه کار

1. Google Chat داده‌های JSON را با درخواست POST به مسیر Webhook در Gateway می‌فرستد (فقط POST، نوع محتوای JSON الزامی و نرخ درخواست برای هر IP محدود است).
2. OpenClaw هر درخواست را پیش از ارسال احراز هویت می‌کند:
   - رویدادهای برنامه Chat شامل `Authorization: Bearer <token>` هستند؛ توکن پیش از تجزیه کامل بدنه تأیید می‌شود.
   - رویدادهای افزونه Google Workspace توکن را در بدنه حمل می‌کنند (`authorizationEventObject.systemIdToken`) و پیش از تأیید، تحت محدودیت سخت‌گیرانه‌تر پیش از احراز هویت (16 KB، 3 s) خوانده می‌شوند.
3. توکن در برابر `audienceType` + `audience` بررسی می‌شود:
   - `audienceType: "app-url"` ← مخاطب، URL مربوط به Webhook HTTPS شما است.
   - `audienceType: "project-number"` ← مخاطب، شماره پروژه Cloud است.
   - توکن‌های افزونه تحت `app-url` علاوه بر این، نیاز دارند `appPrincipal` روی شناسه عددی سرویس‌گیرنده OAuth 2.0 برنامه تنظیم شده باشد (21 رقم، نه ایمیل)؛ در غیر این صورت، تأیید با ثبت هشدار ناموفق می‌شود.
4. پیام‌ها بر اساس فضا هدایت می‌شوند:
   - فضاها نشست‌های مجزای مربوط به هر فضا با شناسه `agent:<agentId>:googlechat:group:<spaceId>` دریافت می‌کنند؛ پاسخ‌ها به رشته پیام ارسال می‌شوند.
   - پیام‌های مستقیم به‌طور پیش‌فرض در نشست اصلی عامل ادغام می‌شوند؛ برای نشست‌های پیام مستقیم مجزا به‌ازای هر همتا، `session.dmScope` را تنظیم کنید (به [نشست](/fa/concepts/session) مراجعه کنید).
5. دسترسی به پیام‌های مستقیم به‌طور پیش‌فرض مبتنی بر جفت‌سازی است. فرستندگان ناشناس یک کد جفت‌سازی دریافت می‌کنند؛ با دستور زیر تأیید کنید:
   - `openclaw pairing approve googlechat <code>`
6. فضاهای گروهی به‌طور پیش‌فرض به @اشاره نیاز دارند. اشاره‌ها از حاشیه‌نویسی‌های `USER_MENTION` در Chat که برنامه را هدف می‌گیرند تشخیص داده می‌شوند؛ اگر تشخیص به نام منبع کاربر برنامه نیاز دارد، `botUser` را تنظیم کنید (برای مثال، `users/1234567890`).
7. هنگامی که تأیید اجرای دستور یا Plugin از Google Chat آغاز شود و تأییدکننده پایداری در `users/<id>` پیکربندی شده باشد، OpenClaw یک کارت تأیید بومی (`cardsV2`) در فضای یا رشته مبدأ ارسال می‌کند. دکمه‌های کارت حاوی توکن‌های بازخوانی مبهم هستند؛ اعلان دستی `/approve <id> <decision>` فقط زمانی ظاهر می‌شود که تحویل بومی در دسترس نباشد.

### دوام ورودی

پس از احراز هویت درخواست، OpenClaw شیء مجوزدهی افزونه را از فضای ذخیره‌سازی حذف می‌کند و پیش از بازگرداندن `200`، رویدادهای `MESSAGE` مربوط به Google Chat را به‌طور پایدار در صف قرار می‌دهد. خرابی ماندگاری، `503` را بازمی‌گرداند و به Google Chat اجازه می‌دهد به‌جای تأیید رویدادی که ممکن است از دست برود، دوباره تلاش کند.

پیام‌های در انتظار یا قابل‌تلاش مجدد پس از راه‌اندازی مجدد Gateway باقی می‌مانند، به‌ازای هر فضا به‌صورت سریالی پردازش می‌شوند و تا زمانی که رکورد تکمیل فعال یا نگه‌داری‌شده وجود دارد، از نام منبع پیام Google Chat برای جلوگیری از ورودی‌های تکراری صف استفاده می‌کنند. کنش‌های غیرپیامی مسیر Webhook جداشده موجود خود را حفظ می‌کنند و این تضمین صف پایدار را دریافت نمی‌کنند. تحویل در مرز صف تا عامل همچنان حداقل یک‌بار انجام می‌شود، بنابراین خرابی هنگام تحویل می‌تواند یک نوبت را دوباره اجرا کند.

## مقصدها

برای تحویل و فهرست‌های مجاز از این شناسه‌ها استفاده کنید:

- پیام‌های مستقیم: `users/<userId>` (پیشنهادی).
- فضاها: `spaces/<spaceId>`.
- ایمیل خام `name@example.com` تغییرپذیر است و فقط وقتی `channels.googlechat.dangerouslyAllowNameMatching: true` باشد برای تطبیق فهرست مجاز استفاده می‌شود.
- منسوخ‌شده: `users/<email>` به‌عنوان شناسه کاربر در نظر گرفته می‌شود، نه ورودی ایمیل در فهرست مجاز.
- پیشوندهای `googlechat:`، `google-chat:` و `gchat:` پذیرفته و حذف می‌شوند.

## نکات برجسته پیکربندی

```json5
{
  channels: {
    googlechat: {
      enabled: true,
      serviceAccountFile: "/path/to/service-account.json",
      // یا serviceAccountRef: { source: "file", provider: "filemain", id: "/channels/googlechat/serviceAccount" }
      audienceType: "app-url",
      audience: "https://gateway.example.com/googlechat",
      appPrincipal: "123456789012345678901", // فقط برای تأیید افزونه؛ شناسه عددی سرویس‌گیرنده OAuth
      webhookPath: "/googlechat",
      botUser: "users/1234567890", // اختیاری؛ به تشخیص اشاره کمک می‌کند
      allowBots: false,
      dmPolicy: "pairing",
      allowFrom: ["users/1234567890"],
      groupPolicy: "allowlist",
      groups: {
        "spaces/AAAA": {
          enabled: true,
          requireMention: true,
          users: ["users/1234567890"],
          systemPrompt: "فقط پاسخ‌های کوتاه.",
        },
      },
      typingIndicator: "message",
      mediaMaxMb: 20,
    },
  },
}
```

نکات:

- اعتبارنامه‌های حساب سرویس: `serviceAccountFile` (مسیر)، `serviceAccount` (رشته یا شیء JSON درون‌خطی)، یا `serviceAccountRef` (SecretRef مربوط به متغیر محیطی/فایل). متغیرهای محیطی `GOOGLE_CHAT_SERVICE_ACCOUNT` (JSON درون‌خطی) و `GOOGLE_CHAT_SERVICE_ACCOUNT_FILE` (مسیر) فقط برای حساب پیش‌فرض اعمال می‌شوند. راه‌اندازی‌های چندحسابی از `channels.googlechat.accounts.<id>` با همان کلیدها، از جمله `serviceAccountRef` به‌ازای هر حساب، استفاده می‌کنند.
- وقتی `webhookPath` تنظیم نشده باشد، مسیر پیش‌فرض Webhook برابر `/googlechat` است؛ `webhookUrl` نیز می‌تواند مسیر را ارائه کند.
- کلیدهای گروه باید شناسه‌های پایدار فضا باشند (`spaces/<spaceId>`). کلیدهای نام نمایشی منسوخ شده‌اند و با همین عنوان ثبت می‌شوند.
- `dangerouslyAllowNameMatching` تطبیق حساب اصلی ایمیل تغییرپذیر را برای فهرست‌های مجاز دوباره فعال می‌کند (حالت سازگاری اضطراری)؛ doctor درباره ورودی‌های ایمیل هشدار می‌دهد.
- کنش‌های واکنش Google Chat ارائه نمی‌شوند. Plugin از احراز هویت حساب سرویس استفاده می‌کند، درحالی‌که نقاط پایانی واکنش Google Chat به احراز هویت کاربر نیاز دارند. پیکربندی موجود `actions.reactions` برای سازگاری پذیرفته می‌شود، اما اثری ندارد.
- کارت‌های تأیید بومی از کلیک دکمه `cardsV2` در Google Chat استفاده می‌کنند، نه رویدادهای واکنش. تأییدکنندگان از `allowFrom` یا `defaultTo` می‌آیند و باید مقادیر عددی پایدار `users/<id>` باشند.
- کنش‌های پیام فقط متن `send` را ارائه می‌کنند. بارگذاری پیوست Google Chat به احراز هویت کاربر نیاز دارد، درحالی‌که این Plugin از احراز هویت حساب سرویس استفاده می‌کند؛ بنابراین بارگذاری فایل خروجی ارائه نمی‌شود.
- `typingIndicator`: `message` (پیش‌فرض) یک جای‌نگهدار `_<Bot> is typing..._` ارسال می‌کند و آن را به نخستین پاسخ تبدیل می‌کند؛ `none` آن را غیرفعال می‌کند؛ `reaction` به OAuth کاربر نیاز دارد و درحال‌حاضر تحت احراز هویت حساب سرویس با ثبت خطا به `message` بازمی‌گردد.
- پیوست‌های ورودی (نخستین پیوست هر پیام) از طریق Chat API در پایپ‌لاین رسانه دانلود می‌شوند و سقف آن‌ها با `mediaMaxMb` تعیین می‌شود (پیش‌فرض 20).
- پیام‌های نوشته‌شده توسط ربات به‌طور پیش‌فرض نادیده گرفته می‌شوند. با `allowBots: true`، پیام‌های پذیرفته‌شده ربات از [محافظت مشترک در برابر حلقه ربات](/fa/channels/bot-loop-protection) استفاده می‌کنند: `channels.defaults.botLoopProtection` را پیکربندی کنید و سپس با `channels.googlechat.botLoopProtection` یا `channels.googlechat.groups.<space>.botLoopProtection` بازنویسی کنید.

جزئیات مرجع اسرار: [مدیریت اسرار](/fa/gateway/secrets).

## عیب‌یابی

### 405 Method Not Allowed

اگر Google Cloud Logs Explorer خطاهایی مانند زیر نشان می‌دهد:

```text
status code: 405, reason phrase: پاسخ خطای HTTP: HTTP/1.1 405 Method Not Allowed
```

کنترل‌کننده Webhook ثبت نشده است. دلایل رایج:

1. **کانال پیکربندی نشده است**: بخش `channels.googlechat` وجود ندارد. با دستور زیر بررسی کنید:

   ```bash
   openclaw config get channels.googlechat
   ```

   اگر "Config path not found" را برمی‌گرداند، پیکربندی را اضافه کنید (به [نکات برجستهٔ پیکربندی](#config-highlights) مراجعه کنید).

2. **Plugin فعال نیست**: وضعیت Plugin را بررسی کنید:

   ```bash
   openclaw plugins list | grep googlechat
   ```

   اگر "disabled" را نشان می‌دهد، `plugins.entries.googlechat.enabled: true` را به پیکربندی خود اضافه کنید.

3. **Gateway پس از تغییرات پیکربندی راه‌اندازی مجدد نشده است**:

   ```bash
   openclaw gateway restart
   ```

بررسی کنید که کانال در حال اجرا است:

```bash
openclaw channels status
# باید نشان دهد: Google Chat default: enabled, configured, ...
```

### مشکلات دیگر

- `openclaw channels status --probe` خطاهای احراز هویت و پیکربندی audience مفقود را نمایش می‌دهد (هر دو `audience` و `audienceType` الزامی هستند).
- اگر هیچ پیامی دریافت نمی‌شود، نشانی Webhook برنامهٔ Chat و پیکربندی محرک را تأیید کنید.
- اگر محدودسازی بر اساس اشاره پاسخ‌ها را مسدود می‌کند، `botUser` را روی نام منبع کاربر برنامه تنظیم و `requireMention` را بررسی کنید.
- `openclaw logs --follow` هنگام ارسال پیام آزمایشی نشان می‌دهد که آیا درخواست‌ها به Gateway می‌رسند یا خیر.

## مرتبط

- [نمای کلی کانال‌ها](/fa/channels) — همهٔ کانال‌های پشتیبانی‌شده
- [مسیریابی کانال](/fa/channels/channel-routing) — مسیریابی نشست برای پیام‌ها
- [پیکربندی Gateway](/fa/gateway/configuration)
- [گروه‌ها](/fa/channels/groups) — رفتار گفت‌وگوی گروهی و محدودسازی بر اساس اشاره
- [جفت‌سازی](/fa/channels/pairing) — احراز هویت پیام مستقیم و جریان جفت‌سازی
- [امنیت](/fa/gateway/security) — مدل دسترسی و مقاوم‌سازی
