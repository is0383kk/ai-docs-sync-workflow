---
read_when:
    - در دسترس قرار دادن رابط کاربری کنترل Gateway خارج از localhost
    - خودکارسازی دسترسی به داشبورد از طریق tailnet یا اینترنت عمومی
summary: یکپارچه‌سازی Tailscale Serve/Funnel برای داشبورد Gateway
title: Tailscale
x-i18n:
    generated_at: "2026-07-27T16:37:38Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: e201a64ac427994401fae1b934d94e0c5afe976b4acd34d45b059978f5f1807e
    source_path: gateway/tailscale.md
    workflow: 16
---

OpenClaw می‌تواند Tailscale **Serve** (tailnet) یا **Funnel** (عمومی) را برای داشبورد Gateway و پورت WebSocket به‌طور خودکار پیکربندی کند. با این کار، gateway به loopback متصل باقی می‌ماند و Tailscale، HTTPS، مسیریابی و (برای Serve) سرآیندهای هویت را فراهم می‌کند.

## حالت‌ها

`gateway.tailscale.mode`:

| حالت            | رفتار                                                                    |
| --------------- | --------------------------------------------------------------------------- |
| `serve`         | Serve فقط در tailnet از طریق `tailscale serve`. gateway روی `127.0.0.1` باقی می‌ماند. |
| `funnel`        | HTTPS عمومی از طریق `tailscale funnel`. به یک گذرواژهٔ مشترک نیاز دارد.            |
| `off` (پیش‌فرض) | بدون خودکارسازی Tailscale.                                                    |

خروجی وضعیت و ممیزی برای این حالت Serve/Funnel در OpenClaw از **در معرض‌بودن Tailscale** استفاده می‌کند. `off` یعنی OpenClaw، Serve یا Funnel را مدیریت نمی‌کند؛ به این معنا نیست که دیمن محلی Tailscale متوقف شده یا از حساب خارج شده است.

## نمونه‌های پیکربندی

### فقط Tailnet (Serve)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve" },
  },
}
```

باز کنید: `https://<magicdns>/` (یا `gateway.controlUi.basePath` پیکربندی‌شدهٔ خود)

برای در معرض دسترس قرار دادن رابط کنترل از طریق یک سرویس نام‌گذاری‌شدهٔ Tailscale به‌جای نام میزبان دستگاه، `gateway.tailscale.serviceName` را روی نام سرویس تنظیم کنید:

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "serve", serviceName: "svc:openclaw" },
  },
}
```

سپس هنگام راه‌اندازی، نشانی سرویس به‌صورت `https://openclaw.<tailnet-name>.ts.net/` به‌جای نام میزبان دستگاه گزارش می‌شود. سرویس‌های Tailscale مستلزم آن‌اند که میزبان یک Node برچسب‌گذاری‌شده و تأییدشده در tailnet شما باشد — پیش از فعال‌سازی این قابلیت، برچسب را پیکربندی و سرویس را در Tailscale تأیید کنید؛ در غیر این صورت، `tailscale serve --service=...` هنگام راه‌اندازی gateway شکست می‌خورد.

### فقط Tailnet (اتصال به IP شبکهٔ Tailnet)

از این گزینه استفاده کنید تا gateway بدون Serve/Funnel مستقیماً روی IP شبکهٔ Tailnet شنود کند:

```json5
{
  gateway: {
    bind: "tailnet",
    auth: { mode: "token", token: "your-token" },
  },
}
```

از دستگاه دیگری در Tailnet متصل شوید:

- رابط کنترل: `http://<tailscale-ip>:18789/`
- WebSocket: `ws://<tailscale-ip>:18789`

<Note>
وقتی یک IPv4 قابل اتصال در Tailnet وجود داشته باشد، Gateway برای کلاینت‌های احرازهویت‌شده روی همان میزبان نیز به `http://127.0.0.1:18789` نیاز دارد. اگر هنگام راه‌اندازی هیچ نشانی Tailnet در دسترس نباشد، فقط به loopback برمی‌گردد؛ پس از در دسترس قرار گرفتن Tailscale، برای افزودن دسترسی مستقیم Tailnet آن را دوباره راه‌اندازی کنید. هیچ‌یک از این مسیرها دسترسی LAN یا عمومی اضافه نمی‌کنند.
</Note>

### اینترنت عمومی (Funnel + گذرواژهٔ مشترک)

```json5
{
  gateway: {
    bind: "loopback",
    tailscale: { mode: "funnel" },
    auth: { mode: "password", password: "replace-me" },
  },
}
```

استفاده از `OPENCLAW_GATEWAY_PASSWORD` را به ثبت گذرواژه روی دیسک ترجیح دهید.

## نمونه‌های CLI

```bash
openclaw gateway --tailscale serve
openclaw gateway --tailscale funnel --auth password
```

## احراز هویت

`gateway.auth.mode` دست‌دهی را کنترل می‌کند:

| حالت                                                   | مورد استفاده                                                                            |
| ------------------------------------------------------ | ----------------------------------------------------------------------------------- |
| `none`                                                 | فقط ورودی خصوصی                                                                |
| `token` (پیش‌فرض وقتی `OPENCLAW_GATEWAY_TOKEN` تنظیم شده است) | توکن مشترک                                                                        |
| `password`                                             | راز مشترک از طریق `OPENCLAW_GATEWAY_PASSWORD` یا پیکربندی                             |
| `trusted-proxy`                                        | پروکسی معکوس آگاه از هویت؛ [احراز هویت پروکسی مورد اعتماد](/fa/gateway/trusted-proxy-auth) را ببینید |

### سرآیندهای هویت Tailscale (فقط Serve)

وقتی `tailscale.mode: "serve"` فعال و `gateway.auth.allowTailscale` برابر `true` باشد، احراز هویت رابط کنترل/WebSocket می‌تواند به‌جای توکن/گذرواژه از سرآیندهای هویت Tailscale (`tailscale-user-login`) استفاده کند. OpenClaw پیش از پذیرش، سرآیند را با تفکیک نشانی `x-forwarded-for` درخواست از طریق دیمن محلی Tailscale (`tailscale whois`) و تطبیق آن با نام ورود موجود در سرآیند تأیید می‌کند. درخواست فقط زمانی واجد شرایط این مسیر است که از loopback وارد شود و سرآیندهای `x-forwarded-for`، `x-forwarded-proto` و `x-forwarded-host` متعلق به Tailscale را حمل کند.

این جریان بدون توکن فرض می‌کند میزبان gateway مورد اعتماد است. اگر ممکن است کد محلی نامطمئن روی همان میزبان اجرا شود، `gateway.auth.allowTailscale: false` را تنظیم کنید و به‌جای آن احراز هویت با توکن/گذرواژه را الزامی کنید.

دامنهٔ این دور زدن:

- فقط برای سطح احراز هویت WebSocket رابط کنترل اعمال می‌شود. نقاط پایانی API مبتنی بر HTTP (`/v1/*`، `/tools/invoke`، `/api/channels/*` و غیره) هرگز از احراز هویت با سرآیند هویت Tailscale استفاده نمی‌کنند؛ آن‌ها همیشه از حالت عادی احراز هویت HTTP در gateway پیروی می‌کنند.
- برای نشست‌های اپراتور رابط کنترل که از قبل هویت دستگاه مرورگر را دارند، هویت تأییدشدهٔ Tailscale رفت‌وبرگشت جفت‌سازی با توکن راه‌اندازی/QR را رد می‌کند.
- این سازوکار خود هویت دستگاه را دور نمی‌زند: کلاینت‌های بدون دستگاه همچنان رد می‌شوند و اتصال‌های دارای نقش Node همچنان بررسی‌های عادی جفت‌سازی و احراز هویت را طی می‌کنند.

## نکات

- Tailscale Serve/Funnel مستلزم نصب بودن CLI مربوط به `tailscale` و ورود به حساب است.
- `tailscale.mode: "funnel"` برای جلوگیری از دسترسی عمومی، تا زمانی که حالت احراز هویت `password` نباشد از شروع به کار خودداری می‌کند.
- `gateway.tailscale.serviceName` فقط برای حالت Serve اعمال می‌شود و به `tailscale serve --service=<name>` ارسال می‌شود. مقدار باید از قالب `svc:<dns-label>` مربوط به Tailscale استفاده کند؛ برای مثال، `svc:openclaw`. Tailscale مستلزم آن است که میزبان‌های سرویس، Nodeهای برچسب‌گذاری‌شده باشند و ممکن است سرویس پیش از آنکه Serve بتواند آن را منتشر کند، به تأیید در کنسول مدیریت نیاز داشته باشد.
- `gateway.tailscale.resetOnExit` هنگام خاموش‌شدن، پیکربندی `tailscale serve`/`tailscale funnel` را لغو می‌کند.
- `gateway.tailscale.preserveFunnel: true` مسیر `tailscale funnel` پیکربندی‌شده به‌صورت خارجی را در میان راه‌اندازی‌های مجدد gateway فعال نگه می‌دارد. با `mode: "serve"`، OpenClaw پیش از اعمال مجدد Serve، `tailscale funnel status` را بررسی می‌کند و اگر یک مسیر Funnel از قبل پورت gateway را پوشش دهد، از آن صرف‌نظر می‌کند. سیاست Funnel مدیریت‌شده توسط OpenClaw که فقط از گذرواژه استفاده می‌کند، بدون تغییر باقی می‌ماند.
- `gateway.bind: "tailnet"` وقتی IPv4 شبکهٔ Tailnet در دسترس باشد، از اتصال مستقیم Tailnet (بدون HTTPS و بدون Serve/Funnel) به‌همراه `127.0.0.1` محلی و الزامی استفاده می‌کند؛ در غیر این صورت، فقط به loopback برمی‌گردد.
- `gateway.bind: "auto"`، loopback را ترجیح می‌دهد؛ برای محدود کردن دسترسی شبکه به Tailnet و در عین حال حفظ دسترسی loopback روی همان میزبان، از `tailnet` استفاده کنید.
- Serve/Funnel فقط **رابط کنترل Gateway + WS** را در معرض دسترس قرار می‌دهند. Nodeها از طریق همان نقطهٔ پایانی WS در Gateway متصل می‌شوند، بنابراین Serve برای دسترسی Node نیز کار می‌کند.

### پیش‌نیازها و محدودیت‌های Tailscale

- Serve مستلزم فعال بودن HTTPS برای tailnet شما است؛ اگر فعال نباشد، CLI از شما درخواست می‌کند.
- Serve سرآیندهای هویت Tailscale را تزریق می‌کند؛ Funnel این کار را نمی‌کند.
- Funnel به Tailscale v1.38.3+، ‏MagicDNS، فعال بودن HTTPS و یک ویژگی Node مربوط به funnel نیاز دارد.
- Funnel فقط از پورت‌های `443`، `8443` و `10000` روی TLS پشتیبانی می‌کند.
- Funnel در macOS به نسخهٔ متن‌باز برنامهٔ Tailscale نیاز دارد.

## کنترل مرورگر (Gateway راه دور + مرورگر محلی)

برای اجرای Gateway روی یک دستگاه و کنترل مرورگر روی دستگاهی دیگر، یک **میزبان Node** را روی دستگاه مرورگر اجرا کنید و هر دو را در یک tailnet نگه دارید. Gateway اقدامات مرورگر را به Node پروکسی می‌کند؛ به سرور کنترل جداگانه یا نشانی Serve نیازی نیست.

برای کنترل مرورگر از Funnel اجتناب کنید؛ جفت‌سازی Node را مانند دسترسی اپراتور در نظر بگیرید.

## اطلاعات بیشتر

- نمای کلی Tailscale Serve: [https://tailscale.com/kb/1312/serve](https://tailscale.com/kb/1312/serve)
- فرمان `tailscale serve`: [https://tailscale.com/kb/1242/tailscale-serve](https://tailscale.com/kb/1242/tailscale-serve)
- نمای کلی Tailscale Funnel: [https://tailscale.com/kb/1223/tailscale-funnel](https://tailscale.com/kb/1223/tailscale-funnel)
- فرمان `tailscale funnel`: [https://tailscale.com/kb/1311/tailscale-funnel](https://tailscale.com/kb/1311/tailscale-funnel)

## مرتبط

- [دسترسی راه دور](/fa/gateway/remote)
- [کشف](/fa/gateway/discovery)
- [احراز هویت](/fa/gateway/authentication)
