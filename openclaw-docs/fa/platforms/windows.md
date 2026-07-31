---
read_when:
    - نصب OpenClaw در Windows
    - انتخاب میان Windows Hub، Windows بومی و WSL2
    - راه‌اندازی برنامه همراه ویندوز یا حالت Node ویندوز
summary: 'پشتیبانی از Windows: Windows Hub، CLI و Gateway بومی، راه‌اندازی Gateway در WSL2، حالت Node و عیب‌یابی'
title: ویندوز
x-i18n:
    generated_at: "2026-07-27T15:34:10Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: c231b81971e1df9f3ee4de1b102c25328c242109331c6465dc802ec003af722b
    source_path: platforms/windows.md
    workflow: 16
---

OpenClaw همراه با یک برنامهٔ همراه بومی **Windows Hub** و پشتیبانی CLI ویندوز عرضه می‌شود.
از Windows Hub برای برنامهٔ دسکتاپی شامل راه‌اندازی، وضعیت سینی سیستم، گفت‌وگو، عیب‌یابی مرکز فرمان
و قابلیت‌های Node ویندوز استفاده کنید. برای استفادهٔ مستقیم از CLI/Gateway، از نصب‌کنندهٔ PowerShell
استفاده کنید. برای سازگارترین محیط اجرای Gateway با لینوکس، از WSL2 استفاده کنید.

## پیشنهادشده: Windows Hub

Windows Hub برنامهٔ همراه بومی WinUI برای Windows 10 20H2+ و
Windows 11 است. این برنامه بدون دسترسی مدیر نصب می‌شود و نصب‌کننده‌های امضاشدهٔ x64
و ARM64 را در صفحهٔ انتشار مستقل خود ارائه می‌کند.

Windows Hub مستقل از OpenClaw CLI و Gateway منتشر می‌شود. جدیدترین
نصب‌کنندهٔ پایدار Hub را از
[صفحهٔ انتشارهای Windows Hub](https://github.com/openclaw/openclaw-windows-node/releases/latest)
یا مستقیماً از طریق `releases/latest/download` دانلود کنید:

- [OpenClawCompanion-Setup-x64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-x64.exe)
- [OpenClawCompanion-Setup-arm64.exe](https://github.com/openclaw/openclaw-windows-node/releases/latest/download/OpenClawCompanion-Setup-arm64.exe)

اگر یکی از پیوندهای بالا خطای 404 می‌دهد، به [صفحهٔ انتشارهای Windows Hub](https://github.com/openclaw/openclaw-windows-node/releases)
بروید و جدیدترین انتشار پایدار Windows Hub را باز کنید. انتشارهای پایدار معمول OpenClaw
نیز یک ساخت سنجاق‌شده و اعتبارسنجی‌شده برای انتشار از Windows Hub را آینه می‌کنند؛ این آینه ممکن است از
یک انتشار مستقل جدیدتر Hub عقب‌تر باشد.

پس از نصب، **OpenClaw Companion** را از منوی Start یا سینی
سیستم اجرا کنید. نصب‌کننده همچنین میان‌برهایی برای راه‌اندازی Gateway، گفت‌وگو، تنظیمات،
بررسی به‌روزرسانی‌ها و حذف نصب اضافه می‌کند.

### Windows Hub شامل چه مواردی است

- وضعیت سینی سیستم و اجرا هنگام ورود.
- راه‌اندازی اولیه برای یک Gateway محلی WSL که در مالکیت برنامه است.
- تنظیمات اتصال برای Gatewayهای محلی، راه‌دور و تونل‌شده از طریق SSH.
- پنجرهٔ گفت‌وگوی بومی به‌همراه دسترسی به رابط کنترل مرورگر.
- عیب‌یابی مرکز فرمان برای نشست‌ها، میزان استفاده، کانال‌ها، Nodeها، جفت‌سازی
  و فرمان‌های تعمیر.
- حالت Node ویندوز برای بوم، صفحه‌نمایش، دوربین،
  اعلان‌ها، وضعیت دستگاه، گفتار و `system.run` کنترل‌شده توسط عامل.
- حالت سرور محلی MCP برای کارخواه‌های MCP مانند Claude Desktop، Claude Code
  و Cursor.

### نخستین اجرا

در نخستین اجرا، اگر Gateway ذخیره‌شدهٔ قابل‌استفاده‌ای وجود نداشته باشد،
Windows Hub راه‌اندازی را باز می‌کند. سریع‌ترین مسیر **راه‌اندازی محلی** است که یک
توزیع WSL `OpenClawGateway` در مالکیت برنامه فراهم می‌کند، Gateway را درون آن نصب می‌کند و
برنامه را جفت می‌کند. این کار توزیع Ubuntu موجود شما را صادر یا تغییر نمی‌دهد.

اگر از قبل Gateway دارید، **راه‌اندازی پیشرفته** را انتخاب کنید یا زبانهٔ اتصالات را باز کنید.
می‌توانید به موارد زیر متصل شوید:

- یک Gateway محلی روی این رایانه
- یک Gateway مبتنی بر WSL روی این رایانه
- یک Gateway راه‌دور با URL و توکن یا کد راه‌اندازی
- یک Gateway قابل‌دسترسی از طریق تونل SSH

پس از پایان راه‌اندازی، نماد سینی سبز می‌شود. **مرکز فرمان** را از
سینی باز کنید تا اتصال، جفت‌سازی، وضعیت Node و سلامت کانال را تأیید کنید.

## حالت Node ویندوز

Windows Hub می‌تواند به‌عنوان یک Node در OpenClaw ثبت شود تا عامل بتواند از طریق
Gateway از قابلیت‌های بومی اعلام‌شدهٔ ویندوز استفاده کند. پیش از اجرای فرمان‌های Node،
این فرمان‌ها باید توسط Node اعلام و در خط‌مشی Gateway مجاز شده باشند؛ برای مشاهدهٔ
مدل کامل اجازه/رد، به [Nodeها](/fa/nodes#command-policy) مراجعه کنید.

فرمان‌های رایج:

| خانواده | فرمان‌ها                                                                              |
| ------ | ------------------------------------------------------------------------------------ |
| بوم | `canvas.present`، `canvas.hide`، `canvas.navigate`، `canvas.eval`، `canvas.snapshot` |
| صفحه‌نمایش | `screen.snapshot`؛ `screen.record` به موافقت صریح نیاز دارد                          |
| دوربین | `camera.list`؛ `camera.snap` و `camera.clip` به موافقت صریح نیاز دارند                  |
| سیستم | `system.notify`، `system.run`، `system.run.prepare`، `system.which`                  |
| دستگاه | `location.get`، `device.info`، `device.status`                                       |
| گفتار   | `talk.ptt.start`، `talk.ptt.stop`، `talk.ptt.cancel`، `talk.ptt.once`، `talk.speak`  |

حالت Node به جفت‌سازی Gateway نیاز دارد. اگر برنامه درخواست جفت‌سازی نشان می‌دهد،
آن را از میزبان Gateway تأیید کنید:

```powershell
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Gateway فقط فرمان‌هایی را ارسال می‌کند که Node اعلام کرده و خط‌مشی سرور
اجازه داده باشد. فرمان‌های حساس به حریم خصوصی مانند `screen.record`، `camera.snap`
و `camera.clip` به موافقت صریح `gateway.nodes.commands.allow` نیاز دارند.

## حالت MCP محلی

Windows Hub می‌تواند همان رجیستری قابلیت‌های بومی ویندوز را به‌عنوان یک
سرور محلی MCP روی loopback در دسترس قرار دهد تا کارخواه‌های محلی MCP بتوانند بدون
Gateway در حال اجرای OpenClaw قابلیت‌های ویندوز را کنترل کنند.

آن را در تنظیمات Windows Hub و در بخش توسعه‌دهنده/پیشرفته فعال کنید. پس از فعال‌شدن
سرور، برنامه نقطهٔ پایانی loopback و توکن حامل را نمایش می‌دهد.

ماتریس حالت‌ها:

| حالت Node | سرور MCP | رفتار                           |
| --------- | ---------- | ---------------------------------- |
| خاموش       | خاموش        | برنامهٔ دسکتاپی فقط برای اپراتور          |
| روشن        | خاموش        | Node ویندوز متصل به Gateway     |
| خاموش       | روشن         | فقط سرور محلی MCP              |
| روشن        | روشن         | Node مبتنی بر Gateway به‌علاوهٔ سرور محلی MCP |

## CLI و Gateway بومی ویندوز

برای استفادهٔ مبتنی بر ترمینال، OpenClaw را از PowerShell نصب کنید:

```powershell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

بررسی کنید:

```powershell
openclaw --version
openclaw doctor
openclaw gateway status --json
```

راه‌اندازی مدیریت‌شده در صورت امکان از Windows Scheduled Tasks استفاده می‌کند. وظیفه،
اسکریپت خوانای `gateway.cmd` را در دایرکتوری وضعیت OpenClaw نگه می‌دارد، اما آن را
از طریق پوشش WScript تولیدشدهٔ `gateway.vbs` اجرا می‌کند تا Gateway پس‌زمینه
پنجرهٔ کنسول قابل‌مشاهده‌ای باز نکند. اگر ایجاد وظیفه رد شود، OpenClaw
به یک مورد ورود در پوشهٔ Startup مختص هر کاربر بازمی‌گردد.

سرویس Gateway را نصب کنید:

```powershell
openclaw gateway install
openclaw gateway status --json
```

برای استفادهٔ صرفاً CLI بدون سرویس مدیریت‌شدهٔ Gateway:

```powershell
openclaw onboard --non-interactive --skip-health
openclaw gateway run
```

## Gateway مبتنی بر WSL2

WSL2 همچنان سازگارترین محیط اجرای Gateway با لینوکس در ویندوز است. Windows
Hub می‌تواند یک Gateway مبتنی بر WSL و در مالکیت برنامه برای شما راه‌اندازی کند، یا می‌توانید آن را به‌صورت دستی
درون توزیع خود نصب کنید.

راه‌اندازی دستی:

```powershell
wsl --install
# یا یک توزیع را صریحاً انتخاب کنید:
wsl --list --online
wsl --install -d Ubuntu-24.04
```

systemd را درون WSL فعال کنید:

```bash
sudo tee /etc/wsl.conf >/dev/null <<'EOF'
[boot]
systemd=true
EOF
```

WSL را از PowerShell راه‌اندازی مجدد کنید:

```powershell
wsl --shutdown
```

سپس OpenClaw را با شروع سریع لینوکس درون WSL نصب کنید:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
openclaw gateway status
```

## شروع خودکار Gateway پیش از ورود به ویندوز

برای راه‌اندازی‌های WSL بدون رابط، مطمئن شوید زنجیرهٔ کامل راه‌اندازی حتی زمانی که هیچ‌کس
وارد ویندوز نمی‌شود اجرا می‌گردد.

درون WSL:

```bash
sudo apt-get install -y dbus-x11
sudo loginctl enable-linger "$(whoami)"
openclaw gateway install
```

در PowerShell با دسترسی Administrator:

```powershell
schtasks /create /tn "WSL Boot" /tr "wsl.exe -d Ubuntu --exec dbus-launch true" /sc onstart /ru "$env:USERNAME"
```

`Ubuntu` را با نام توزیع خود از خروجی زیر جایگزین کنید:

```powershell
wsl --list --verbose
```

<Note>
دو تغییر نسبت به دستورالعمل‌های قدیمی‌تر:

- **`dbus-launch true` به‌جای `/bin/true`**: در WSL >= 2.6.1.0 یک
  پس‌رفت ([microsoft/WSL #13416](https://github.com/microsoft/WSL/issues/13416))
  باعث می‌شود توزیع 15-20 ثانیه پس از خروج آخرین کارخواه به‌دلیل بیکاری خاتمه یابد، حتی
  اگر linger فعال باشد. `dbus-launch true` به‌عنوان راه‌حل موقت، یک فرایند فرزند init را
  زنده نگه می‌دارد (بحث جامعه، [microsoft/WSL #9245](https://github.com/microsoft/WSL/discussions/9245)).
- **`/ru "$env:USERNAME"` به‌جای `/ru SYSTEM`**: توزیع‌های WSL مختص هر کاربر (که
  راه‌اندازی پیش‌فرض هستند) برای حساب SYSTEM قابل‌مشاهده نیستند، بنابراین به‌نظر می‌رسد
  وظیفه اجرا می‌شود، اما توزیع هرگز شروع نمی‌شود. اجرا با حساب خودتان
  از این مشکل جلوگیری می‌کند؛ هنگام ایجاد وظیفه، ویندوز گذرواژهٔ شما را درخواست می‌کند.

</Note>

پس از راه‌اندازی مجدد، از درون WSL بررسی کنید:

```bash
systemctl --user is-enabled openclaw-gateway.service
systemctl --user status openclaw-gateway.service --no-pager
```

## ارائهٔ سرویس‌های WSL روی LAN

WSL شبکهٔ مجازی خودش را دارد. اگر دستگاه دیگری باید به سرویسی
درون WSL دسترسی پیدا کند، یک درگاه ویندوز را به IP فعلی WSL هدایت کنید. IP مربوط به WSL ممکن است
پس از راه‌اندازی‌های مجدد تغییر کند، بنابراین در صورت نیاز قانون هدایت را تازه‌سازی کنید.

نمونه در PowerShell با دسترسی Administrator:

```powershell
$Distro = "Ubuntu-24.04"
$ListenPort = 2222
$TargetPort = 22

$WslIp = (wsl -d $Distro -- hostname -I).Trim().Split(" ")[0]
if (-not $WslIp) { throw "WSL IP not found." }

netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=$ListenPort `
  connectaddress=$WslIp connectport=$TargetPort

New-NetFirewallRule -DisplayName "WSL SSH $ListenPort" -Direction Inbound `
  -Protocol TCP -LocalPort $ListenPort -Action Allow
```

نکته‌ها:

- SSH از دستگاهی دیگر، IP میزبان ویندوز را هدف می‌گیرد؛ برای نمونه `ssh user@windows-host -p 2222`.
- Nodeهای راه‌دور باید به یک URL قابل‌دسترسی Gateway اشاره کنند، نه `127.0.0.1`.
- برای دسترسی LAN از `listenaddress=0.0.0.0` و برای دسترسی صرفاً محلی از `127.0.0.1` استفاده کنید.

## عیب‌یابی

### نماد سینی نمایش داده نمی‌شود

Task Manager را برای `OpenClaw.Tray.WinUI.exe` بررسی کنید. اگر در حال اجرا است، بخش
نمادهای پنهان سینی را باز و آن را سنجاق کنید. در غیر این صورت، **OpenClaw Companion** را از
منوی Start اجرا کنید.

### راه‌اندازی محلی ناموفق است

گزارش راه‌اندازی را از Windows Hub باز کنید یا مورد زیر را بررسی کنید:

```powershell
notepad "$env:LOCALAPPDATA\OpenClawTray\Logs\Setup\easy-setup-latest.txt"
```

دلایل رایج: WSL غیرفعال، مجازی‌سازی مسدودشده، وضعیت قدیمی WSL
در مالکیت برنامه، یا خرابی شبکه هنگام نصب بستهٔ Gateway.

### برنامه می‌گوید جفت‌سازی لازم است

درخواست اپراتور یا Node را از Gateway تأیید کنید:

```powershell
openclaw devices list
openclaw devices approve <requestId>
```

اگر دستگاه از قبل توکن داشت، پس از تأیید از زبانهٔ اتصالات دوباره متصل شوید.

### گفت‌وگوی وب نمی‌تواند به Gateway راه‌دور دسترسی پیدا کند

گفت‌وگوی وب راه‌دور به HTTPS یا localhost نیاز دارد. برای گواهی‌های خودامضاشده،
به گواهی در ویندوز اعتماد کنید یا از یک تونل SSH به URL مبتنی بر localhost استفاده کنید.

### فرمان‌های `screen.snapshot`، دوربین یا صدا ناموفق هستند

مجوزهای ویندوز را برای دوربین، میکروفون، ضبط صفحه‌نمایش و
اعلان‌ها تأیید کنید. نصب‌های بسته‌بندی‌شده قابلیت‌های محافظت‌شده را اعلام می‌کنند، اما
ممکن است ویندوز در نخستین استفادهٔ یک فرمان از آن‌ها همچنان درخواست تأیید کند.

### اتصال Git یا GitHub ناموفق است

برخی شبکه‌ها HTTPS به GitHub را مسدود یا محدود می‌کنند. اگر `git clone` یا
`gh auth login` ناموفق است، شبکه‌ای دیگر، یک VPN یا پراکسی HTTP/HTTPS را امتحان کنید.

برای احراز هویت مبتنی بر توکن `gh` در نشست فعلی:

```powershell
$env:GH_TOKEN="<your-token>"
gh auth status
gh auth setup-git
```

هرگز توکن‌ها را commit نکنید یا آن‌ها را در issueها یا Pull requestها جای‌گذاری نکنید.

## مرتبط

- [نمای کلی نصب](/fa/install)
- [راه‌اندازی Node.js](/fa/install/node)
- [Nodeها](/fa/nodes)
- [رابط کنترل](/fa/web/control-ui)
- [پیکربندی Gateway](/fa/gateway/configuration)
