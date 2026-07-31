---
read_when:
    - اجرای راه‌اندازی‌های Gateway راه دور یا عیب‌یابی آن‌ها
summary: دسترسی از راه دور با استفاده از Gateway WS، تونل‌های SSH و tailnetها
title: دسترسی از راه دور
x-i18n:
    generated_at: "2026-07-27T14:08:37Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8f05e32fcfa16d5ddfcd684d0550c9af311914e2b4d91c95edad3490dc2e56d9
    source_path: gateway/remote.md
    workflow: 16
---

OpenClaw یک Gateway (گره اصلی) را روی یک میزبان اجرا می‌کند و هر کلاینت را به آن متصل می‌کند. Gateway مالک نشست‌ها، پروفایل‌های احراز هویت، کانال‌ها و وضعیت است؛ هر چیز دیگری یک کلاینت است.

- **اپراتورها** (شما یا برنامه macOS): وقتی Gateway در دسترس باشد، اتصال مستقیم WebSocket از طریق LAN/Tailnet ساده‌ترین روش است؛ تونل‌زنی SSH راهکار جایگزین همگانی است.
- **Nodeها** (iOS/Android و دستگاه‌های دیگر): به **WebSocket** Gateway متصل می‌شوند (LAN/tailnet یا تونل SSH).

## ایده اصلی

WebSocket مربوط به Gateway به‌طور پیش‌فرض روی **loopback** و در پورت `18789` (`gateway.port`) گوش می‌دهد. برای استفاده از راه دور، یا آن را از طریق Tailscale Serve / اتصال قابل‌اعتماد LAN-Tailnet در معرض دسترس قرار دهید، یا پورت loopback را از طریق SSH هدایت کنید.

## گزینه‌های توپولوژی

| راه‌اندازی                             | محل اجرای Gateway                                                                                    | مناسب برای                                                                                                                                          |
| --------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Gateway همیشه‌روشن در tailnet شما | میزبان دائمی (VPS یا سرور خانگی) که از طریق Tailscale یا SSH به آن دسترسی دارید                                        | لپ‌تاپ‌هایی که اغلب به خواب می‌روند اما نیاز دارند عامل همیشه روشن باشد. [exe.dev](/fa/install/exe-dev) (ماشین مجازی آسان) یا [Hetzner](/fa/install/hetzner) (VPS عملیاتی) را ببینید. |
| رایانه رومیزی خانگی                      | رایانه رومیزی؛ لپ‌تاپ از راه دور و از طریق حالت راه دور برنامه macOS متصل می‌شود (Settings → Connection → OpenClaw runs) | نگه‌داشتن عامل روی سخت‌افزاری که روشن می‌ماند. راهنمای عملیاتی: [دسترسی راه دور macOS](/fa/platforms/mac/remote).                                       |
| لپ‌تاپ                            | لپ‌تاپ که به‌طور امن از طریق تونل SSH یا Tailscale Serve در معرض دسترس قرار گرفته است (`gateway.bind: "loopback"` را حفظ کنید)                | راه‌اندازی‌های تک‌دستگاهی. [Tailscale](/fa/gateway/tailscale) و [وب](/fa/web) را ببینید.                                                                       |

برای راه‌اندازی‌های همیشه‌روشن و لپ‌تاپ، بهتر است `gateway.bind: "loopback"` را حفظ کرده و برای رابط کنترل از **Tailscale Serve** استفاده کنید، یا یک اتصال قابل‌اعتماد LAN/Tailnet با `gateway.remote.transport: "direct"` داشته باشید. تونل SSH راهکار جایگزینی است که از هر دستگاهی کار می‌کند.

## جریان فرمان (چه چیزی کجا اجرا می‌شود)

یک Gateway مالک وضعیت و کانال‌ها است؛ Nodeها تجهیزات جانبی هستند. نمونه (پیام Telegram که به ابزار یک Node هدایت می‌شود):

1. پیام Telegram به **Gateway** می‌رسد.
2. Gateway **عامل** را اجرا می‌کند و عامل تصمیم می‌گیرد که آیا ابزار Node فراخوانی شود یا نه.
3. Gateway از طریق WebSocket مربوط به Gateway، **Node** را فراخوانی می‌کند (`node.invoke` RPC).
4. Node نتیجه را برمی‌گرداند؛ Gateway به Telegram پاسخ می‌دهد.

Nodeها سرویس Gateway را اجرا نمی‌کنند. در هر میزبان فقط یک Gateway باید اجرا شود، مگر اینکه عمداً پروفایل‌های ایزوله اجرا کنید ([چند Gateway](/fa/gateway/multiple-gateways) را ببینید). «حالت Node» برنامه macOS صرفاً یک کلاینت Node روی WebSocket مربوط به Gateway است.

## تونل SSH ‏(CLI و ابزارها)

```bash
ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
```

وقتی تونل برقرار است، `openclaw health` و `openclaw status --deep` از طریق `ws://127.0.0.1:18789` به Gateway راه دور دسترسی پیدا می‌کنند. `openclaw gateway status`، `openclaw gateway health`، `openclaw gateway probe` و `openclaw gateway call` نیز می‌توانند از طریق `--url` یک URL هدایت‌شده را هدف قرار دهند.

<Note>
`18789` را با `gateway.port` پیکربندی‌شده خود (یا `--port` / `OPENCLAW_GATEWAY_PORT`) جایگزین کنید.
</Note>

<Warning>
`--url` هرگز به اعتبارنامه‌های پیکربندی یا محیط بازنمی‌گردد. `--token` یا `--password` را صریحاً ارسال کنید؛ بدون آن‌ها کلاینت هیچ اعتبارنامه‌ای ارسال نمی‌کند و اگر Gateway مقصد به احراز هویت نیاز داشته باشد، اتصال ناموفق خواهد بود.
</Warning>

## پیش‌فرض‌های راه دور CLI

یک مقصد راه دور را ذخیره کنید تا فرمان‌های CLI به‌طور پیش‌فرض از آن استفاده کنند:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      url: "ws://127.0.0.1:18789",
      token: "your-token",
    },
  },
}
```

وقتی Gateway فقط روی loopback است، URL را روی `ws://127.0.0.1:18789` نگه دارید و ابتدا تونل SSH را باز کنید. در انتقال تونل SSH برنامه macOS، نام میزبان Gateway کشف‌شده در `gateway.remote.sshTarget` (`user@host` یا `user@host:port`) قرار می‌گیرد؛ `gateway.remote.url` همان URL تونل محلی باقی می‌ماند. اگر پورت راه دور با پورت محلی متفاوت است، `gateway.remote.remotePort` را تنظیم کنید.

تأیید کلید میزبان به‌طور پیش‌فرض سخت‌گیرانه است (`gateway.remote.sshHostKeyPolicy: "strict"`). برای واگذاری آن به پیکربندی مؤثر OpenSSH خود، مقدارش را روی `"openssh"` تنظیم کنید؛ پیش از فعال‌سازی، تنظیمات SSH کاربر و سیستم خود را بررسی کنید.

برای Gatewayی که از قبل روی LAN یا Tailnet قابل‌اعتماد در دسترس است، از حالت مستقیم استفاده کنید:

```json5
{
  gateway: {
    mode: "remote",
    remote: {
      transport: "direct",
      url: "ws://192.168.0.202:18789",
      token: "your-token",
    },
  },
}
```

## اولویت اعتبارنامه‌ها

تفکیک اعتبارنامه Gateway در مسیرهای فراخوانی/کاوش/وضعیت و پایش تأیید اجرای Discord از یک قرارداد مشترک پیروی می‌کند. میزبان Node نیز با یک استثنا در حالت محلی از همین قرارداد استفاده می‌کند (`gateway.remote.*` را نادیده می‌گیرد).

- اعتبارنامه‌های صریح (`--token`، `--password` یا `gatewayToken` یک ابزار) همیشه در مسیرهای فراخوانی که احراز هویت صریح را می‌پذیرند، اولویت دارند.
- ایمنی بازنویسی URL:
  - `--url` در CLI هرگز از اعتبارنامه‌های ضمنی پیکربندی/محیط دوباره استفاده نمی‌کند.
  - `OPENCLAW_GATEWAY_URL` محیط فقط می‌تواند از اعتبارنامه‌های محیط استفاده کند (`OPENCLAW_GATEWAY_TOKEN` / `OPENCLAW_GATEWAY_PASSWORD`).
- پیش‌فرض‌های حالت محلی:
  - توکن: `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token` -> `gateway.remote.token` (بازگشت به مقدار راه دور فقط وقتی توکن محلی تنظیم نشده باشد)
  - رمز عبور: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.auth.password` -> `gateway.remote.password` (بازگشت به مقدار راه دور فقط وقتی رمز عبور محلی تنظیم نشده باشد)
- پیش‌فرض‌های حالت راه دور:
  - توکن: `gateway.remote.token` -> `OPENCLAW_GATEWAY_TOKEN` -> `gateway.auth.token`
  - رمز عبور: `OPENCLAW_GATEWAY_PASSWORD` -> `gateway.remote.password` -> `gateway.auth.password`
- استثنای حالت محلی میزبان Node: `gateway.remote.token` / `gateway.remote.password` نادیده گرفته می‌شوند.
- بررسی‌های توکن کاوش/وضعیت راه دور به‌طور پیش‌فرض سخت‌گیرانه‌اند: هنگام هدف‌گیری حالت راه دور فقط از `gateway.remote.token` استفاده می‌کنند (بدون بازگشت به توکن محلی).
- بازنویسی‌های محیط Gateway فقط از `OPENCLAW_GATEWAY_*` استفاده می‌کنند.

## دسترسی راه دور رابط گفت‌وگو

WebChat پورت HTTP جداگانه‌ای ندارد؛ رابط گفت‌وگوی SwiftUI مستقیماً به WebSocket مربوط به Gateway متصل می‌شود.

- `18789` را از طریق SSH هدایت کنید (بخش بالا را ببینید)، سپس کلاینت‌ها را به `ws://127.0.0.1:18789` متصل کنید.
- برای حالت مستقیم LAN/Tailnet، کلاینت‌ها را به URL خصوصی پیکربندی‌شده `ws://` یا URL امن `wss://` متصل کنید.
- در macOS، حالت راه دور برنامه انتقال انتخاب‌شده را به‌طور خودکار مدیریت می‌کند.

## حالت راه دور برنامه macOS

برنامه نوار منوی macOS همین راه‌اندازی را از ابتدا تا انتها انجام می‌دهد: بررسی‌های وضعیت راه دور، WebChat و هدایت Voice Wake. راهنمای عملیاتی: [دسترسی راه دور macOS](/fa/platforms/mac/remote).

## قواعد امنیتی (راه دور/VPN)

Gateway را فقط روی **loopback** نگه دارید، مگر اینکه مطمئن باشید به bind نیاز دارید.

- **Loopback + SSH/Tailscale Serve** امن‌ترین پیش‌فرض است (بدون قرارگیری در معرض دسترسی عمومی).
- `ws://` متن ساده برای میزبان‌های loopback، خصوصی/LAN ‏(RFC 1918)، link-local، ‏CGNAT، ‏`.local` و `.ts.net` پذیرفته می‌شود. میزبان‌های عمومی راه دور باید از `wss://` استفاده کنند.
- **Bindهای غیر-loopback** ‏(`lan`/`tailnet`/`custom`، یا `auto` وقتی loopback در دسترس نیست) باید از احراز هویت Gateway استفاده کنند: توکن، رمز عبور یا پراکسی معکوس آگاه از هویت با `gateway.auth.mode: "trusted-proxy"`.
- `gateway.remote.token` / `.password` منابع اعتبارنامه کلاینت هستند؛ به‌تنهایی احراز هویت سرور را پیکربندی نمی‌کنند.
- مسیرهای فراخوانی محلی فقط وقتی `gateway.auth.*` تنظیم نشده باشد، می‌توانند از `gateway.remote.*` به‌عنوان مقدار جایگزین استفاده کنند.
- اگر `gateway.auth.token` / `gateway.auth.password` صریحاً از طریق SecretRef پیکربندی شده اما تفکیک‌نشده باشد، تفکیک به‌صورت بسته و ناموفق انجام می‌شود (بدون اینکه بازگشت به مقدار راه دور آن را پنهان کند).
- `gateway.remote.tlsFingerprint` گواهی TLS راه دور را برای `wss://` سنجاق می‌کند، از جمله ترافیک اپراتور/کنترل و Node همراه در حالت مستقیم macOS. بدون سنجاق ذخیره‌شده، macOS فقط پس از موفقیت اعتماد عادی سیستم، در نخستین استفاده سنجاق می‌کند؛ Gatewayهای خودامضاشده یا دارای CA خصوصی به اثر انگشت صریح یا اتصال راه دور از طریق SSH نیاز دارند.
- **Tailscale Serve** می‌تواند وقتی `gateway.auth.allowTailscale: true` است، ترافیک رابط کنترل/WebSocket را از طریق سرآیندهای هویت احراز هویت کند. نقاط پایانی HTTP API از این احراز هویت سرآیندی استفاده نمی‌کنند و در عوض از حالت عادی احراز هویت HTTP مربوط به Gateway پیروی می‌کنند. این جریان بدون توکن فرض می‌کند میزبان Gateway قابل‌اعتماد است؛ برای استفاده از احراز هویت با راز مشترک در همه‌جا، آن را روی `false` تنظیم کنید.
- احراز هویت **پراکسی قابل‌اعتماد** به‌طور پیش‌فرض انتظار یک پراکسی غیر-loopback آگاه از هویت را دارد. پراکسی‌های معکوس loopback روی همان میزبان به `gateway.auth.trustedProxy.allowLoopback = true` صریح نیاز دارند.
- کنترل مرورگر را مانند دسترسی اپراتور در نظر بگیرید: فقط tailnet، همراه با جفت‌سازی آگاهانه Node.

بررسی عمیق: [امنیت](/fa/gateway/security).

### macOS: تونل SSH پایدار از طریق LaunchAgent

برای کلاینت‌های macOS، ساده‌ترین راه‌اندازی پایدار از یک ورودی پیکربندی SSH به نام `LocalForward` به‌همراه یک LaunchAgent استفاده می‌کند که تونل را در برابر راه‌اندازی مجدد و خرابی فعال نگه می‌دارد.

#### گام 1: افزودن پیکربندی SSH

`~/.ssh/config` را ویرایش کنید:

```ssh
Host remote-gateway
    HostName <REMOTE_IP>
    User <REMOTE_USER>
    LocalForward 18789 127.0.0.1:18789
    IdentityFile ~/.ssh/id_rsa
```

`<REMOTE_IP>` و `<REMOTE_USER>` را با مقادیر خود جایگزین کنید.

#### گام 2: کپی‌کردن کلید SSH (یک‌بار)

```bash
ssh-copy-id -i ~/.ssh/id_rsa <REMOTE_USER>@<REMOTE_IP>
```

#### گام 3: پیکربندی توکن Gateway

```bash
openclaw config set gateway.remote.token "<your-token>"
```

اگر Gateway راه دور از احراز هویت با رمز عبور استفاده می‌کند، به‌جای آن از `gateway.remote.password` استفاده کنید. `OPENCLAW_GATEWAY_TOKEN` همچنان به‌عنوان بازنویسی در سطح پوسته معتبر است، اما راه‌اندازی پایدار کلاینت راه دور، `gateway.remote.token` / `gateway.remote.password` است.

#### گام 4: ایجاد LaunchAgent

آن را با نام `~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist` ذخیره کنید:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.ssh-tunnel</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/bin/ssh</string>
        <string>-N</string>
        <string>remote-gateway</string>
    </array>
    <key>KeepAlive</key>
    <true/>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

#### گام 5: بارگذاری LaunchAgent

```bash
launchctl bootstrap gui/$UID ~/Library/LaunchAgents/ai.openclaw.ssh-tunnel.plist
```

تونل هنگام ورود به سیستم به‌طور خودکار آغاز می‌شود، پس از خرابی دوباره راه‌اندازی می‌شود و پورت هدایت‌شده را فعال نگه می‌دارد.

<Note>
اگر یک LaunchAgent باقی‌مانده با نام `com.openclaw.ssh-tunnel` از راه‌اندازی قدیمی‌تر دارید، آن را از حالت بارگذاری خارج کرده و حذف کنید.
</Note>

#### عیب‌یابی

```bash
# بررسی کنید آیا تونل در حال اجرا است
ps aux | grep "ssh -N remote-gateway" | grep -v grep
lsof -i :18789

# تونل را دوباره راه‌اندازی کنید
launchctl kickstart -k gui/$UID/ai.openclaw.ssh-tunnel

# تونل را متوقف کنید
launchctl bootout gui/$UID/ai.openclaw.ssh-tunnel
```

| ورودی پیکربندی                         | کارکرد                                                 |
| ------------------------------------ | ------------------------------------------------------------ |
| `LocalForward 18789 127.0.0.1:18789` | پورت محلی 18789 را به پورت راه‌دور 18789 هدایت می‌کند               |
| `ssh -N`                             | SSH بدون اجرای فرمان‌های راه‌دور (فقط هدایت پورت) |
| `KeepAlive`                          | اگر تونل از کار بیفتد، آن را به‌طور خودکار راه‌اندازی مجدد می‌کند              |
| `RunAtLoad`                          | هنگام ورود، با بارگذاری LaunchAgent تونل را راه‌اندازی می‌کند        |

## مرتبط

- [Tailscale](/fa/gateway/tailscale)
- [احراز هویت](/fa/gateway/authentication)
- [راه‌اندازی Gateway راه‌دور](/fa/gateway/remote-gateway-readme)
