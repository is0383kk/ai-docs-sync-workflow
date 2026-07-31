---
read_when:
    - اجرای OpenClaw پشت یک پراکسی آگاه از هویت
    - راه‌اندازی Pomerium، Caddy یا nginx با OAuth در جلوی OpenClaw
    - رفع خطاهای عدم مجوز WebSocket 1008 در پیکربندی‌های پراکسی معکوس
    - تصمیم‌گیری درباره محل تنظیم HSTS و دیگر سرآیندهای مقاوم‌سازی HTTP
sidebarTitle: Trusted proxy auth
summary: احراز هویت Gateway را به یک پروکسی معکوس مورد اعتماد واگذار کنید (Pomerium، Caddy، nginx + OAuth)
title: احراز هویت پراکسی مورد اعتماد
x-i18n:
    generated_at: "2026-07-27T14:12:27Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 39bf8f12b3ae95f53b21bfed12deb1c8ed8f767711955bbee52c74538052a89f
    source_path: gateway/trusted-proxy-auth.md
    workflow: 16
---

<Warning>
**ویژگی حساس از نظر امنیتی.** این حالت، احراز هویت را به‌طور کامل به پراکسی معکوس شما واگذار می‌کند. پیکربندی نادرست می‌تواند Gateway شما را در معرض دسترسی غیرمجاز قرار دهد. پیش از فعال‌سازی، این صفحه را با دقت بخوانید.
</Warning>

## زمان استفاده

- OpenClaw را پشت یک **پراکسی آگاه از هویت** (Pomerium، Caddy + OAuth، nginx + oauth2-proxy، Traefik + forward auth) اجرا می‌کنید.
- پراکسی شما تمام مراحل احراز هویت را انجام می‌دهد و هویت کاربر را از طریق سرآیندها ارسال می‌کند.
- در محیط Kubernetes یا کانتینری هستید که پراکسی تنها مسیر دسترسی به Gateway است.
- با خطاهای WebSocket از نوع `1008 unauthorized` مواجه می‌شوید، زیرا مرورگرها نمی‌توانند توکن‌ها را در محموله‌های WS ارسال کنند.

## زمان‌هایی که نباید استفاده شود

- پراکسی شما کاربران را احراز هویت نمی‌کند و صرفاً پایان‌دهنده TLS یا متعادل‌کننده بار است.
- مسیری برای دسترسی به Gateway وجود دارد که پراکسی را دور می‌زند، مانند حفره‌های دیوار آتش یا دسترسی از شبکه داخلی.
- مطمئن نیستید که پراکسی شما سرآیندهای هدایت‌شده را به‌درستی حذف یا بازنویسی می‌کند.
- فقط به دسترسی شخصی تک‌کاربره نیاز دارید؛ در عوض Tailscale Serve + loopback را در نظر بگیرید.

## نحوه کار

<Steps>
  <Step title="پراکسی کاربر را احراز هویت می‌کند">
    پراکسی معکوس شما کاربران را احراز هویت می‌کند (OAuth، OIDC، SAML و غیره).
  </Step>
  <Step title="پراکسی یک سرآیند هویت اضافه می‌کند">
    پراکسی سرآیندی حاوی هویت کاربر احراز‌شده اضافه می‌کند (برای مثال، `x-forwarded-user: nick@example.com`).
  </Step>
  <Step title="Gateway منبع قابل‌اعتماد را تأیید می‌کند">
    OpenClaw بررسی می‌کند که درخواست از یک **IP پراکسی قابل‌اعتماد** (`gateway.trustedProxies`) آمده باشد و آدرس loopback یا رابط محلی خود Gateway نباشد.
  </Step>
  <Step title="Gateway هویت را استخراج می‌کند">
    OpenClaw ابتدا سرآیندهای الزامی و سپس هویت کاربر را از سرآیند پیکربندی‌شده می‌خواند.
  </Step>
  <Step title="صدور مجوز">
    اگر همه بررسی‌ها موفق باشند و کاربر، در صورت تنظیم بودن، از `allowUsers` عبور کند، درخواست مجاز شناخته می‌شود.
  </Step>
</Steps>

## پیکربندی

```json5
{
  gateway: {
    // احراز هویت پراکسی قابل‌اعتماد به‌طور پیش‌فرض انتظار دارد IP مبدأ پراکسی loopback نباشد
    bind: "lan",

    // حیاتی: فقط IPهای پراکسی خود را در اینجا اضافه کنید
    trustedProxies: ["10.0.0.1", "172.17.0.1"],

    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        // سرآیند حاوی هویت کاربر احراز‌شده (الزامی)
        userHeader: "x-forwarded-user",

        // اختیاری: سرآیندهایی که وجودشان الزامی است (تأیید پراکسی)
        requiredHeaders: ["x-forwarded-proto", "x-forwarded-host"],

        // اختیاری: محدودسازی به کاربران مشخص (خالی = اجازه به همه)
        allowUsers: ["nick@example.com", "admin@company.org"],

        // اختیاری: اجازه به پراکسی loopback روی همان میزبان پس از پذیرش صریح
        allowLoopback: false,

        // اختیاری: اجازه به کاربران احراز‌شده پراکسی برای ثبت دستگاه‌های مرورگر جدید
        deviceAutoApprove: {
          enabled: false,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

<Warning>
**قواعد زمان اجرا، به‌ترتیب ارزیابی**

1. IP مبدأ درخواست باید با `gateway.trustedProxies` مطابقت داشته باشد (با پشتیبانی از CIDR)، در غیر این صورت رد می‌شود (`trusted_proxy_untrusted_source`).
2. درخواست‌های با مبدأ loopback (`127.0.0.1`، `::1`) رد می‌شوند، مگر اینکه `gateway.auth.trustedProxy.allowLoopback = true` فعال باشد و آدرس loopback نیز در `trustedProxies` قرار داشته باشد (`trusted_proxy_loopback_source`). این بررسی پیش از بررسی سرآیندها انجام می‌شود؛ بنابراین مبدأ loopback حتی در صورت نبود سرآیندهای الزامی نیز به همین شیوه رد می‌شود.
3. مبدأهای غیر-loopback که با یکی از آدرس‌های رابط شبکه محلی خود میزبان Gateway مطابقت دارند، برای جلوگیری از جعل رد می‌شوند (`trusted_proxy_local_interface_source`). اگر خودِ شناسایی رابط‌ها نیز ناموفق باشد، درخواست رد می‌شود (`trusted_proxy_local_interface_check_failed`).
4. `requiredHeaders` و `userHeader` باید موجود و غیرخالی باشند.
5. `allowUsers`، اگر خالی نباشد، باید کاربر استخراج‌شده را شامل شود.

**شواهد سرآیند هدایت‌شده، محلی‌بودن loopback را برای بازگشت مستقیم محلی نادیده می‌گیرند.** اگر درخواستی از loopback برسد اما دارای یک `Forwarded`، هرگونه سرآیند `X-Forwarded-*` یا سرآیند `X-Real-IP` باشد، این شواهد آن را از بازگشت رمز عبور مستقیم محلی و دروازه‌بانی هویت دستگاه محروم می‌کنند، هرچند همچنان به‌دلیل loopback بودن در احراز هویت پراکسی قابل‌اعتماد ناموفق است.

`allowLoopback` به فرایندهای محلی روی میزبان Gateway به همان اندازه پراکسی معکوس اعتماد می‌کند. آن را فقط زمانی فعال کنید که Gateway همچنان با دیوار آتش از دسترسی مستقیم راه دور محافظت می‌شود و پراکسی محلی، سرآیندهای هویت ارائه‌شده توسط کلاینت را حذف یا بازنویسی می‌کند.

کلاینت‌های داخلی Gateway که از پراکسی معکوس عبور نمی‌کنند باید از `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` استفاده کنند، نه از سرآیندهای هویت پراکسی قابل‌اعتماد. استقرارهای غیر-loopback رابط کاربری کنترل همچنان به `gateway.controlUi.allowedOrigins` صریح نیاز دارند.
</Warning>

### مرجع پیکربندی

<ParamField path="gateway.trustedProxies" type="string[]" required>
  آرایه‌ای از آدرس‌های IP پراکسی یا CIDRهایی که باید قابل‌اعتماد شناخته شوند. درخواست‌های سایر IPها رد می‌شوند.
</ParamField>
<ParamField path="gateway.auth.mode" type="string" required>
  باید `"trusted-proxy"` باشد.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.userHeader" type="string" required>
  نام سرآیندی که هویت کاربر احراز‌شده را در بر دارد.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.requiredHeaders" type="string[]">
  سرآیندهای دیگری که برای قابل‌اعتماد شناخته‌شدن درخواست باید موجود باشند.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowUsers" type="string[]">
  فهرست مجاز هویت‌های کاربری. خالی‌بودن به‌معنای اجازه به همه کاربران احراز‌شده است.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.allowLoopback" type="boolean" default="false">
  پشتیبانی اختیاری از پراکسی‌های معکوس loopback روی همان میزبان.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.enabled" type="boolean" default="false">
  هویت دستگاه‌های جدید رابط کاربری کنترل و WebChat را پس از احراز هویت پراکسی قابل‌اعتماد، به‌طور خودکار تأیید می‌کند.
</ParamField>
<ParamField path="gateway.auth.trustedProxy.deviceAutoApprove.scopes" type="string[]" default='["operator.read", "operator.write", "operator.approvals"]'>
  حداکثر دامنه‌های اعطاشده به دستگاه مرورگری که به‌طور خودکار تأیید شده است. فهرست‌کردن صریح `operator.admin` به هر کاربر احراز‌شده توسط پراکسی اجازه می‌دهد اعطای خودکار دستگاه با دسترسی کامل مدیر را درخواست کند، باعث می‌شود درخواست‌های بدون دامنه به‌طور خودکار دسترسی کامل مدیر دریافت کنند و یافته ممیزی امنیتی CRITICAL با شناسه `gateway.trusted_proxy_device_auto_approve_admin` به‌همراه هشدار راه‌اندازی Gateway را فعال می‌کند.
</ParamField>

<Warning>
`allowLoopback` را فقط زمانی فعال کنید که پراکسی معکوس محلی، مرز اعتماد موردنظر باشد. هر فرایند محلی که بتواند به Gateway متصل شود می‌تواند برای ارسال سرآیندهای هویت پراکسی تلاش کند؛ بنابراین دسترسی مستقیم به Gateway را به میزبان محدود نگه دارید و سرآیندهای تحت مالکیت پراکسی مانند `x-forwarded-proto` یا، در صورت پشتیبانی پراکسی، یک سرآیند ادعای امضاشده را الزامی کنید.
</Warning>

## تأیید خودکار دستگاه

احراز هویت پراکسی قابل‌اعتماد می‌تواند به‌صورت اختیاری از هویت پراکسی به‌عنوان مرز تأیید برای دستگاه‌های مرورگر جدید استفاده کند:

```json5
{
  gateway: {
    auth: {
      mode: "trusted-proxy",
      trustedProxy: {
        userHeader: "x-forwarded-user",
        allowUsers: ["operator@example.com"],
        deviceAutoApprove: {
          enabled: true,
          scopes: ["operator.read", "operator.write", "operator.approvals"],
        },
      },
    },
  },
}
```

مقدار پیش‌فرض `enabled: false` است. هنگام فعال‌بودن، همه قواعد زیر اعمال می‌شوند:

1. WebSocket باید از طریق روش `trusted-proxy`، با هویت کاربری غیرخالی که در صورت پیکربندی فهرست مجاز از `allowUsers` عبور کرده است، احراز هویت شده باشد. اتصال‌های مبتنی بر توکن، رمز عبور، Tailscale و اتصال‌های احراز هویت‌نشده هرگز از این سیاست استفاده نمی‌کنند.
2. فقط دستگاه مرورگر جدید رابط کاربری کنترل یا WebChat می‌تواند به‌طور خودکار تأیید شود. هر درخواست برای دستگاه موجود، از جمله ارتقای دامنه، با `openclaw devices approve <requestId>` در انتظار تأیید دستی باقی می‌ماند.
3. دستگاه با نقش `operator` تأیید می‌شود. اگر درخواست اتصال شامل دامنه‌ها باشد، مجوز اعطاشده دقیقاً اشتراک دامنه‌های درخواستی و `deviceAutoApprove.scopes` است. اگر درخواست دامنه‌ها را حذف کند، فهرست پیکربندی‌شده اعطا می‌شود؛ اگر آن فهرست حذف شده باشد، مقدار پیش‌فرض آن `operator.read`، `operator.write` و `operator.approvals` است. سپس مجوز حاصل، در صورت وجود، علاوه‌براین با سرآیند پراکسی [`x-openclaw-scopes`](#control-ui-pairing-behavior) اتصال محدود می‌شود؛ بنابراین پراکسی‌ای که دامنه‌های کاربر را محدود می‌کند، مجوز **ماندگار** دستگاه را نیز، نه فقط نشست را، محدود می‌کند؛ سرآیند موجود اما خالی هیچ دامنه‌ای اعطا نمی‌کند. این محدودیت حتی زمانی اعمال می‌شود که کلاینت فهرست دامنه‌های خود را حذف کند.
4. `operator.admin` فقط با فهرست‌شدن صریح در `deviceAutoApprove.scopes` مجاز است. در صورت فهرست‌شدن، هر کاربر احراز‌شده توسط پراکسی می‌تواند دسترسی کامل مدیر را روی دستگاه مرورگر جدید درخواست کند و به‌طور خودکار دریافت کند؛ درخواست‌های بدون دامنه نیز به‌طور خودکار دسترسی کامل مدیر دریافت می‌کنند. `openclaw security audit` یافته CRITICAL با شناسه `gateway.trusted_proxy_device_auto_approve_admin` را گزارش می‌کند و Gateway هنگام راه‌اندازی یک‌بار هشدار ثبت می‌کند. تا زمانی که نقش‌های مختص هر هویت در دسترس قرار گیرند، تأیید دستی مدیر با `openclaw devices approve` یا `openclaw devices rotate` را ترجیح دهید.

<Warning>
فعال‌کردن این گزینه، ثبت دستگاه‌های مرورگر جدید را به‌طور کامل به هویت پراکسی معکوس واگذار می‌کند. یک حساب پراکسی به‌خطر‌افتاده می‌تواند دستگاهی ماندگار با همه دامنه‌های پیکربندی‌شده ثبت کند. فهرست‌کردن `operator.admin` آن دستگاه را بدون تأیید دستی به مدیر کامل تبدیل می‌کند. دسترسی به Gateway را فقط از طریق پراکسی ممکن نگه دارید، احراز هویت قوی پراکسی را الزامی کنید، سرآیندهای هویت را بازنویسی کنید و از فهرست محدود `allowUsers` استفاده کنید.
</Warning>

## رفتار جفت‌سازی رابط کاربری کنترل

وقتی `gateway.auth.mode = "trusted-proxy"` فعال باشد و درخواست بررسی‌های پراکسی قابل‌اعتماد را با موفقیت پشت سر بگذارد، نشست‌های WebSocket رابط کاربری کنترل می‌توانند بدون هویت جفت‌سازی دستگاه متصل شوند.

پیامدهای دامنه:

- نشست‌های WebSocket رابط کاربری کنترل بدون دستگاه متصل می‌شوند، اما به‌طور پیش‌فرض هیچ دامنه اپراتوری دریافت نمی‌کنند. OpenClaw فهرست دامنه‌های درخواستی را به `[]` پاک می‌کند تا نشستی که به دستگاه یا توکن جفت‌شده و تأییدشده متصل نیست، نتواند مجوزها را برای خود اعلام کند.
- اگر پس از اتصال موفق WebSocket، متدها با `missing scope` ناموفق شدند، از HTTPS استفاده کنید تا مرورگر بتواند هویت دستگاه را ایجاد و جفت‌سازی را تکمیل کند. به [HTTP ناامن رابط کاربری کنترل](/fa/web/control-ui#insecure-http) مراجعه کنید.
- پیکربندی‌های قدیمی‌تر که همچنان کلید بازنشسته
  `gateway.controlUi.dangerouslyDisableDeviceAuth=true` را در بر دارند، از
  [مهاجرت ارتقای رابط کاربری کنترل](/fa/web/control-ui#device-pairing-first-connection) محدودشده استفاده می‌کنند.

محدودسازی دامنه توسط پراکسی معکوس: اگر پراکسی شما هنگام درخواست ارتقای WebSocket رابط کاربری کنترل، `x-openclaw-scopes` را ارسال کند، OpenClaw دامنه‌های نشست را به اشتراک دامنه‌های درخواستی و دامنه‌های اعلام‌شده محدود می‌کند. این سرآیند دامنه‌ای اعطا نمی‌کند؛ فقط دامنه‌هایی را که نشست می‌تواند داشته باشد محدود می‌سازد. وقتی `deviceAutoApprove.enabled` برابر true باشد، همین محدودیت برای مجوز ماندگار دستگاه که توسط [تأیید خودکار دستگاه](#automatic-device-approval) نوشته می‌شود نیز اعمال می‌شود؛ بنابراین دستگاه تأییدشده خودکار هرگز دامنه‌هایی بیش از موارد اعلام‌شده توسط پراکسی نخواهد داشت.

پیامدها:

- جفت‌سازی دیگر دروازه اصلی دسترسی رابط کاربری کنترل بدون دستگاه نیست. وقتی `deviceAutoApprove.enabled` برابر true باشد، هویت پراکسی همچنین به دروازه تأیید ثبت دستگاه‌های مرورگر جدید تبدیل می‌شود.
- سیاست احراز هویت پراکسی معکوس و `allowUsers` شما به کنترل دسترسی مؤثر تبدیل می‌شوند.
- ورودی Gateway را فقط به IPهای پراکسی قابل‌اعتماد محدود نگه دارید (`gateway.trustedProxies` + دیوار آتش).

کلاینت‌های سفارشی WebSocket نشست رابط کاربری کنترل نیستند. ورودی بازنشسته ارتقای رابط کاربری کنترل، دسترسی موقت به کلاینت‌های دلخواه
`client.mode: "backend"` یا کلاینت‌های دارای ساختار CLI اعطا نمی‌کند. خودکارسازی سفارشی باید از
هویت/جفت‌سازی دستگاه، مسیر کمکی backend رزروشده مستقیم محلی `client.id: "gateway-client"`
یا [Plugin مربوط به RPC مدیریتی HTTP](/fa/plugins/admin-http-rpc)
استفاده کند، هرگاه سطح درخواست/پاسخ HTTP گزینه مناسب‌تری باشد.

## سرآیند دامنه‌های اپراتور

احراز هویت پروکسی مورداعتماد یک حالت HTTP **حامل هویت** است، بنابراین فراخواننده‌ها می‌توانند به‌صورت اختیاری دامنه‌های دسترسی اپراتور را با `x-openclaw-scopes` در درخواست‌های API مبتنی بر HTTP اعلام کنند.

نکته: دامنه‌های دسترسی WebSocket توسط دست‌دهی پروتکل Gateway و پیوند هویت دستگاه تعیین می‌شوند. در درخواست‌های ارتقای WebSocket رابط کاربری کنترل، `x-openclaw-scopes` فقط سقفی برای دامنه‌های دسترسی نشستِ مورد مذاکره است، نه اعطای دسترسی. به [رفتار جفت‌سازی رابط کاربری کنترل](#control-ui-pairing-behavior) مراجعه کنید.

نمونه‌ها:

- `x-openclaw-scopes: operator.read`
- `x-openclaw-scopes: operator.read,operator.write`
- `x-openclaw-scopes: operator.admin,operator.write`

رفتار:

- وقتی سرآیند وجود داشته باشد، OpenClaw مجموعه دامنه‌های دسترسی اعلام‌شده را رعایت می‌کند.
- وقتی سرآیند وجود داشته اما خالی باشد، درخواست **هیچ** دامنه دسترسی اپراتوری را اعلام نمی‌کند.
- وقتی سرآیند وجود نداشته باشد، APIهای معمول HTTP حامل هویت به مجموعه استاندارد و پیش‌فرض دامنه‌های دسترسی اپراتور بازمی‌گردند (`operator.admin`، `operator.read`، `operator.write`، `operator.approvals`، `operator.pairing`، `operator.talk.secrets`).
- **مسیرهای HTTP افزونه** با احراز هویت Gateway به‌طور پیش‌فرض محدودترند: وقتی `x-openclaw-scopes` وجود نداشته باشد، دامنه دسترسی زمان اجرای آن‌ها فقط به `operator.write` بازمی‌گردد.
- درخواست‌های HTTP با مبدأ مرورگر، حتی پس از موفقیت احراز هویت پروکسی مورداعتماد، همچنان باید `gateway.controlUi.allowedOrigins` (یا حالت بازگشت عامدانه مبتنی بر سرآیند Host) را با موفقیت بگذرانند.

قاعده عملی: هرگاه می‌خواهید درخواست پروکسی مورداعتماد محدودتر از پیش‌فرض‌ها باشد، یا زمانی که یک مسیر افزونه با احراز هویت Gateway به چیزی قوی‌تر از دامنه دسترسی نوشتن نیاز دارد، `x-openclaw-scopes` را صریحاً ارسال کنید.

## خاتمه TLS و HSTS

از یک نقطه خاتمه TLS استفاده کنید و HSTS را در همان‌جا اعمال کنید.

<Tabs>
  <Tab title="خاتمه TLS در پروکسی (توصیه‌شده)">
    وقتی پروکسی معکوس شما HTTPS را برای `https://control.example.com` مدیریت می‌کند، `Strict-Transport-Security` را در پروکسی برای آن دامنه تنظیم کنید.

    - برای استقرارهای در معرض اینترنت مناسب است.
    - گواهی و سیاست سخت‌سازی HTTP را در یک محل نگه می‌دارد.
    - OpenClaw می‌تواند پشت پروکسی روی HTTP حلقه‌بازگشت باقی بماند.

    نمونه مقدار سرآیند:

    ```text
    Strict-Transport-Security: max-age=31536000; includeSubDomains
    ```

  </Tab>
  <Tab title="خاتمه TLS در Gateway">
    اگر خود OpenClaw مستقیماً HTTPS را ارائه می‌کند (بدون پروکسی خاتمه‌دهنده TLS)، موارد زیر را تنظیم کنید:

    ```json5
    {
      gateway: {
        tls: { enabled: true },
        http: {
          securityHeaders: {
            strictTransportSecurity: "max-age=31536000; includeSubDomains",
          },
        },
      },
    }
    ```

    `strictTransportSecurity` یک مقدار رشته‌ای سرآیند یا `false` را برای غیرفعال‌سازی صریح می‌پذیرد.

  </Tab>
</Tabs>

### راهنمای عرضه

- هنگام اعتبارسنجی ترافیک، ابتدا با حداکثر سن کوتاهی شروع کنید (برای مثال `max-age=300`).
- فقط پس از اطمینان بالا، آن را به مقادیر بلندمدت افزایش دهید (برای مثال `max-age=31536000`).
- فقط در صورتی `includeSubDomains` را اضافه کنید که هر زیردامنه برای HTTPS آماده باشد.
- فقط در صورتی از پیش‌بارگذاری استفاده کنید که عمداً الزامات پیش‌بارگذاری را برای کل مجموعه دامنه‌های خود برآورده می‌کنید.
- توسعه محلیِ محدود به حلقه‌بازگشت از HSTS سودی نمی‌برد.

## نمونه‌های راه‌اندازی پروکسی

<AccordionGroup>
  <Accordion title="Pomerium">
    Pomerium هویت را در `x-pomerium-claim-email` (یا سرآیندهای ادعای دیگر) و یک JWT را در `x-pomerium-jwt-assertion` ارسال می‌کند.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // نشانی IP مربوط به Pomerium
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-pomerium-claim-email",
            requiredHeaders: ["x-pomerium-jwt-assertion"],
          },
        },
      },
    }
    ```

    قطعه پیکربندی Pomerium:

    ```yaml
    routes:
      - from: https://openclaw.example.com
        to: http://openclaw-gateway:18789
        policy:
          - allow:
              or:
                - email:
                    is: nick@example.com
        pass_identity_headers: true
    ```

  </Accordion>
  <Accordion title="Caddy با OAuth">
    Caddy با افزونه `caddy-security` می‌تواند کاربران را احراز هویت کرده و سرآیندهای هویت را ارسال کند.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // نشانی IP پروکسی Caddy/sidecar
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```

    قطعه Caddyfile:

    ```caddy
    openclaw.example.com {
        authenticate with oauth2_provider
        authorize with policy1

        reverse_proxy openclaw:18789 {
            header_up X-Forwarded-User {http.auth.user.email}
        }
    }
    ```

  </Accordion>
  <Accordion title="nginx + oauth2-proxy">
    oauth2-proxy کاربران را احراز هویت می‌کند و هویت را در `x-auth-request-email` ارسال می‌کند.

    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["10.0.0.1"], // نشانی IP مربوط به nginx/oauth2-proxy
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-auth-request-email",
          },
        },
      },
    }
    ```

    قطعه پیکربندی nginx:

    ```nginx
    location / {
        auth_request /oauth2/auth;
        auth_request_set $user $upstream_http_x_auth_request_email;

        proxy_pass http://openclaw:18789;
        proxy_set_header X-Auth-Request-Email $user;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
    ```

  </Accordion>
  <Accordion title="Traefik با احراز هویت ارجاعی">
    ```json5
    {
      gateway: {
        bind: "lan",
        trustedProxies: ["172.17.0.1"], // نشانی IP کانتینر Traefik
        auth: {
          mode: "trusted-proxy",
          trustedProxy: {
            userHeader: "x-forwarded-user",
          },
        },
      },
    }
    ```
  </Accordion>
</AccordionGroup>

## پیکربندی ترکیبی توکن

اگر یک توکن اشتراکی نیز پیکربندی شده باشد (`gateway.auth.token` یا `OPENCLAW_GATEWAY_TOKEN`) راه‌اندازی Gateway احراز هویت پروکسی مورداعتماد را رد می‌کند. این دو مانعةالجمع‌اند، زیرا یک توکن اشتراکی به فراخواننده‌های همان میزبان اجازه می‌دهد از مسیری کاملاً متفاوت با هویت تأییدشده توسط پروکسی که این حالت برای اجرای آن در نظر گرفته شده است، احراز هویت کنند.

اگر راه‌اندازی با خطایی مانند `gateway auth mode is trusted-proxy, but a shared token is also configured` ناموفق شد:

- هنگام استفاده از حالت پروکسی مورداعتماد، توکن اشتراکی را حذف کنید، یا
- اگر قصد دارید از احراز هویت مبتنی بر توکن استفاده کنید، `gateway.auth.mode` را به `"token"` تغییر دهید.

سرآیندهای هویت پروکسی مورداعتماد روی حلقه‌بازگشت همچنان به‌شکل بسته و امن شکست می‌خورند: فراخواننده‌های همان میزبان بی‌سروصدا به‌عنوان کاربران پروکسی احراز هویت نمی‌شوند. فراخواننده‌های داخلی OpenClaw که پروکسی را دور می‌زنند می‌توانند در عوض با `gateway.auth.password` / `OPENCLAW_GATEWAY_PASSWORD` احراز هویت کنند. بازگشت به توکن در حالت پروکسی مورداعتماد عمداً پشتیبانی نمی‌شود.

## چک‌لیست امنیتی

پیش از فعال‌سازی احراز هویت پروکسی مورداعتماد، موارد زیر را تأیید کنید:

- [ ] **پروکسی تنها مسیر است**: درگاه Gateway از همه‌چیز به‌جز پروکسی شما با فایروال مسدود شده است.
- [ ] **trustedProxies حداقلی است**: فقط نشانی‌های IP واقعی پروکسی شما، نه کل زیرشبکه‌ها.
- [ ] **منبع پروکسی حلقه‌بازگشت عامدانه است**: احراز هویت پروکسی مورداعتماد برای درخواست‌های دارای منبع حلقه‌بازگشت به‌شکل بسته و امن شکست می‌خورد، مگر اینکه `gateway.auth.trustedProxy.allowLoopback` صریحاً برای یک پروکسی روی همان میزبان فعال شده باشد.
- [ ] **پروکسی سرآیندها را حذف می‌کند**: پروکسی شما سرآیندهای `x-forwarded-*` دریافتی از کلاینت‌ها را بازنویسی می‌کند (نه اینکه به آن‌ها بیفزاید).
- [ ] **خاتمه TLS**: پروکسی شما TLS را مدیریت می‌کند؛ کاربران از طریق HTTPS متصل می‌شوند.
- [ ] **allowedOrigins صریح است**: رابط کاربری کنترل خارج از حلقه‌بازگشت از `gateway.controlUi.allowedOrigins` صریح استفاده می‌کند.
- [ ] **allowUsers تنظیم شده است** (توصیه‌شده): به‌جای اجازه‌دادن به هر فرد احرازشده، دسترسی را به کاربران شناخته‌شده محدود کنید.
- [ ] **پیکربندی ترکیبی توکن وجود ندارد**: `gateway.auth.token` و `gateway.auth.mode: "trusted-proxy"` را هم‌زمان تنظیم نکنید.
- [ ] **بازگشت محلی به گذرواژه خصوصی است**: اگر `gateway.auth.password` را برای فراخواننده‌های مستقیم داخلی پیکربندی می‌کنید، درگاه Gateway را با فایروال مسدود نگه دارید تا کلاینت‌های راه‌دورِ خارج از پروکسی نتوانند مستقیماً به آن دسترسی پیدا کنند.
- [ ] **تأیید خودکار دستگاه عامدانه است**: اگر `deviceAutoApprove.enabled` برابر با true است، امنیت حساب پروکسی معکوس را مرز ثبت دستگاه در نظر بگیرید و فهرست دامنه‌های دسترسی اعطاشده را غیرمدیریتی و حداقلی نگه دارید.

## ممیزی امنیتی

`openclaw security audit` احراز هویت پروکسی مورداعتماد را با یافته‌ای دارای شدت **بحرانی** علامت‌گذاری می‌کند. این رفتار عمدی است؛ یادآوری می‌کند که امنیت را به راه‌اندازی پروکسی خود واگذار کرده‌اید.

ممیزی موارد زیر را بررسی می‌کند:

- هشدار/یادآوری بحرانی پایه `gateway.trusted_proxy_auth`.
- نبود پیکربندی `trustedProxies`.
- نبود پیکربندی `userHeader`.
- `allowUsers` خالی (به هر کاربر احرازشده اجازه می‌دهد).
- فعال‌بودن `allowLoopback` برای منابع پروکسی روی همان میزبان.
- فعال‌بودن تأیید خودکار دستگاه مرورگر (جفت‌سازی دستگاه جدید را به هویت پروکسی واگذار می‌کند).

هرگاه رابط کاربری کنترل در دسترس قرار گیرد، یافته‌های جداگانه و غیرمختص به پروکسی مورداعتماد نیز اعمال می‌شوند: `gateway.controlUi.allowedOrigins` عام یا مفقود، و بازگشت مبدأ مبتنی بر سرآیند Host.

## عیب‌یابی

<AccordionGroup>
  <Accordion title="trusted_proxy_untrusted_source">
    درخواست از نشانی IP موجود در `gateway.trustedProxies` نیامده است. بررسی کنید:

    - آیا نشانی IP پروکسی درست است؟ (نشانی‌های IP کانتینر Docker ممکن است تغییر کنند.)
    - آیا یک متعادل‌کننده بار جلوی پروکسی شما قرار دارد؟
    - برای یافتن نشانی‌های IP واقعی از `docker inspect` یا `kubectl get pods -o wide` استفاده کنید.

  </Accordion>
  <Accordion title="trusted_proxy_loopback_source">
    OpenClaw یک درخواست پروکسی مورداعتماد با منبع حلقه‌بازگشت را رد کرد.

    بررسی کنید:

    - آیا پروکسی از `127.0.0.1` / `::1` متصل می‌شود؟
    - آیا می‌خواهید احراز هویت پروکسی مورداعتماد را با یک پروکسی معکوس حلقه‌بازگشت روی همان میزبان استفاده کنید؟

    راه‌حل:

    - برای کلاینت‌های داخلی روی همان میزبان که از پروکسی عبور نمی‌کنند، احراز هویت با توکن/گذرواژه را ترجیح دهید، یا
    - ترافیک را از طریق یک نشانی پروکسی مورداعتمادِ غیرحلقه‌بازگشت هدایت کنید و آن نشانی IP را در `gateway.trustedProxies` نگه دارید، یا
    - برای یک پروکسی معکوس عامدانه روی همان میزبان، `gateway.auth.trustedProxy.allowLoopback = true` را تنظیم کنید، نشانی حلقه‌بازگشت را در `gateway.trustedProxies` نگه دارید و مطمئن شوید پروکسی سرآیندهای هویت را حذف یا بازنویسی می‌کند.

  </Accordion>
  <Accordion title="trusted_proxy_local_interface_source / trusted_proxy_local_interface_check_failed">
    نشانی IP منبع درخواست با یکی از نشانی‌های رابط شبکه غیرحلقه‌بازگشت خود میزبان Gateway (نه پروکسی) مطابقت داشت؛ این حفاظی در برابر ترافیک جعل‌شده همان میزبان در tailnetها یا شبکه‌های پل Docker است. `..._check_failed` یعنی خود کشف رابط با خطا مواجه شده است، بنابراین OpenClaw به‌شکل بسته و امن شکست می‌خورد.

    بررسی کنید:

    - آیا فرایندی روی خود میزبان Gateway مستقیماً سرآیندهای هویت را ارسال می‌کند و پروکسی را دور می‌زند؟
    - آیا پروکسی در همان فضای نام شبکه Gateway اجرا می‌شود و نشانی IP آن نیز به‌عنوان یک رابط محلی نمایش داده می‌شود؟

    راه‌حل: ترافیک پروکسی را از طریق نشانی‌ای هدایت کنید که روی میزبان Gateway نیز به‌صورت محلی متصل نشده باشد، یا فقط برای یک راه‌اندازی واقعی پروکسی روی همان میزبان از `allowLoopback` استفاده کنید.

  </Accordion>
  <Accordion title="trusted_proxy_user_missing">
    سرآیند کاربر خالی یا مفقود بود. بررسی کنید:

    - آیا پروکسی شما برای ارسال سرآیندهای هویت پیکربندی شده است؟
    - آیا نام سرآیند درست است؟ (به بزرگی و کوچکی حروف حساس نیست، اما املای آن مهم است)
    - آیا کاربر واقعاً در پروکسی احراز هویت شده است؟

  </Accordion>
  <Accordion title="trusted_proxy_missing_header_*">
    یک سرآیند الزامی وجود نداشت. بررسی کنید:

    - پیکربندی پروکسی خود را برای آن سرآیندهای مشخص.
    - اینکه آیا سرآیندها در جایی از زنجیره حذف می‌شوند.

  </Accordion>
  <Accordion title="trusted_proxy_user_not_allowed">
    کاربر احراز هویت شده است، اما در `allowUsers` نیست. او را اضافه کنید یا فهرست مجاز را حذف کنید.
  </Accordion>
  <Accordion title="trusted_proxy_no_proxies_configured / trusted_proxy_config_missing">
    `gateway.auth.mode` برابر با `"trusted-proxy"` است، اما `gateway.trustedProxies` خالی است، یا خود `gateway.auth.trustedProxy` وجود ندارد. تا زمانی که هر دو تنظیم نشوند، همهٔ درخواست‌ها رد می‌شوند.
  </Accordion>
  <Accordion title="trusted_proxy_origin_not_allowed">
    احراز هویت پروکسی مورد اعتماد موفق بود، اما هدر `Origin` مرورگر بررسی‌های مبدأ Control UI را پشت سر نگذاشت.

    بررسی کنید:

    - `gateway.controlUi.allowedOrigins` مبدأ دقیق مرورگر را شامل می‌شود.
    - به مبدأهای دارای نویسهٔ عام متکی نیستید، مگر اینکه عمداً رفتار «اجازه به همه» را بخواهید.
    - اگر عمداً از حالت بازگشت به هدر Host استفاده می‌کنید، `gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback=true` آگاهانه تنظیم شده است.

  </Accordion>
  <Accordion title="اتصال برقرار می‌شود، اما متدها نبود محدوده را گزارش می‌کنند">
    WebSocket متصل می‌شود، اما `chat.history`، `sessions.list` یا
    `models.list` با `missing scope: operator.read` ناموفق می‌شود.

    علت‌های رایج:

    - نشست Control UI بدون دستگاه: احراز هویت پروکسی مورد اعتماد می‌تواند اتصال WebSocket را بدون هویت دستگاه بپذیرد، اما OpenClaw به‌صورت طراحی‌شده محدوده‌های نشست‌های بدون دستگاه را پاک می‌کند.
    - کلاینت سفارشی بک‌اند: ورودی ارتقای منسوخ‌شدهٔ Control UI هرگز به کلاینت‌های WebSocket دلخواهِ بک‌اند یا دارای ساختار CLI دسترسی نمی‌دهد.
    - `x-openclaw-scopes` بیش‌ازحد محدود: اگر پروکسی این هدر را به درخواست ارتقای WebSocket در Control UI تزریق کند، محدوده‌های نشست به همان مجموعه محدود می‌شوند. مقدار خالی هدر باعث می‌شود هیچ محدوده‌ای وجود نداشته باشد.

    راه‌حل:

    - برای Control UI از HTTPS استفاده کنید تا مرورگر بتواند هویت دستگاه را ایجاد و جفت‌سازی را تکمیل کند.
    - برای خودکارسازی سفارشی، از هویت دستگاه/جفت‌سازی، مسیر کمکی رزروشدهٔ بک‌اند محلی مستقیم `gateway-client` یا [RPC مدیریتی HTTP](/fa/plugins/admin-http-rpc) استفاده کنید.
    - کلید منسوخ‌شدهٔ `gateway.controlUi.dangerouslyDisableDeviceAuth` را به پیکربندی فعلی اضافه نکنید. نصب‌های قدیمی به‌طور خودکار از مهاجرت یک‌بارهٔ خودجفت‌سازی استفاده می‌کنند.

  </Accordion>
  <Accordion title="WebSocket همچنان ناموفق است">
    مطمئن شوید پروکسی شما:

    - از ارتقاهای WebSocket پشتیبانی می‌کند (`Upgrade: websocket`، `Connection: upgrade`).
    - هدرهای هویت را در درخواست‌های ارتقای WebSocket ارسال می‌کند (نه فقط HTTP).
    - مسیر احراز هویت جداگانه‌ای برای اتصال‌های WebSocket ندارد.

  </Accordion>
</AccordionGroup>

## مهاجرت از احراز هویت توکنی

<Steps>
  <Step title="پیکربندی پروکسی">
    پروکسی خود را برای احراز هویت کاربران و ارسال هدرها پیکربندی کنید.
  </Step>
  <Step title="آزمایش مستقل پروکسی">
    تنظیمات پروکسی را به‌طور مستقل آزمایش کنید (curl با هدرها).
  </Step>
  <Step title="به‌روزرسانی پیکربندی OpenClaw">
    پیکربندی OpenClaw را با احراز هویت پروکسی مورد اعتماد به‌روزرسانی کنید.
  </Step>
  <Step title="راه‌اندازی مجدد Gateway">
    Gateway را مجدداً راه‌اندازی کنید.
  </Step>
  <Step title="آزمایش WebSocket">
    اتصال‌های WebSocket را از Control UI آزمایش کنید.
  </Step>
  <Step title="ممیزی">
    `openclaw security audit` را اجرا و یافته‌ها را بررسی کنید.
  </Step>
</Steps>

## مرتبط

- [پیکربندی](/fa/gateway/configuration) — مرجع پیکربندی
- [محدوده‌های اپراتور](/fa/gateway/operator-scopes) — نقش‌ها، محدوده‌ها و بررسی‌های تأیید
- [دسترسی از راه دور](/fa/gateway/remote) — الگوهای دیگر دسترسی از راه دور
- [امنیت](/fa/gateway/security) — راهنمای کامل امنیت
- [Tailscale](/fa/gateway/tailscale) — جایگزینی ساده‌تر برای دسترسی محدود به tailnet
