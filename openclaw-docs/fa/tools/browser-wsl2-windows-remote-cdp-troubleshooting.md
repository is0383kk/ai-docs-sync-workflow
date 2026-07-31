---
read_when:
    - اجرای Gateway ‏OpenClaw در WSL2 درحالی‌که Chrome روی Windows قرار دارد
    - مشاهده خطاهای هم‌پوشان مرورگر/control-ui در WSL2 و Windows
    - تصمیم‌گیری میان Chrome MCP محلیِ میزبان و CDP خامِ راه دور در پیکربندی‌های چندمیزبانه
summary: عیب‌یابی لایه‌به‌لایه Gateway در WSL2 و CDP راه‌دور Chrome در Windows
title: عیب‌یابی WSL2 + Windows + پروتکل CDP از راه دور Chrome
x-i18n:
    generated_at: "2026-07-27T14:44:01Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 66ec4ed5bfccc66b594a43d56296c69242e8b9cf50b36c6cb3990b1d6ea58faa
    source_path: tools/browser-wsl2-windows-remote-cdp-troubleshooting.md
    workflow: 16
---

در پیکربندی رایجِ میزبانِ تفکیک‌شده، OpenClaw Gateway درون WSL2 اجرا می‌شود، Chrome
در Windows اجرا می‌شود و کنترل مرورگر باید از مرز WSL2/Windows عبور کند. چندین
مشکل مستقل ممکن است هم‌زمان ظاهر شوند (نگاه کنید به
[شمارهٔ 39369](https://github.com/openclaw/openclaw/issues/39369)): انتقال CDP،
امنیت مبدأ Control UI و توکن/جفت‌سازی ممکن است هرکدام به‌طور مستقل
با خطا مواجه شوند، درحالی‌که خطاهایی با ظاهر مشابه ایجاد می‌کنند. به‌جای حدس‌زدن
اینکه کدام‌یک خراب است، لایه‌های زیر را به‌ترتیب بررسی کنید.

## ابتدا حالت مرورگر مناسب را انتخاب کنید

### گزینهٔ 1: CDP راه‌دور خام از WSL2 به Windows

از یک پروفایل مرورگر راه‌دور استفاده کنید که از WSL2 به نقطهٔ پایانی CDP مربوط به
Chrome در Windows اشاره می‌کند. این گزینه را زمانی انتخاب کنید که Gateway درون WSL2 باقی می‌ماند،
Chrome در Windows اجرا می‌شود و کنترل مرورگر باید از مرز WSL2/Windows عبور کند.

### گزینهٔ 2: Chrome MCP محلیِ میزبان

از درایور `existing-session` (پروفایل `user`) فقط زمانی استفاده کنید که Gateway
روی همان میزبان Chrome اجرا می‌شود، به وضعیت محلی مرورگرِ واردشده به حساب نیاز دارید،
به انتقال مرورگر بین میزبان‌ها نیاز ندارید و به `responsebody`،
خروجی PDF، رهگیری دانلود یا عملیات دسته‌ای نیاز ندارید (پروفایل‌های Chrome MCP
از این موارد پشتیبانی نمی‌کنند).

برای WSL2 Gateway همراه با Chrome در Windows، از CDP راه‌دور خام استفاده کنید. Chrome MCP
محلیِ میزبان است، نه پلی از WSL2 به Windows.

## معماری عملیاتی

- WSL2، Gateway را روی `127.0.0.1:18789` اجرا می‌کند
- Windows، Control UI را در یک مرورگر عادی در `http://127.0.0.1:18789/` باز می‌کند
- Chrome در Windows یک نقطهٔ پایانی CDP را روی پورت `9222` ارائه می‌کند
- WSL2 می‌تواند به آن نقطهٔ پایانی CDP در Windows دسترسی پیدا کند
- OpenClaw یک پروفایل مرورگر را به نشانی قابل‌دسترسی از WSL2 هدایت می‌کند

## قانون حیاتی برای Control UI

هنگامی که رابط کاربری از Windows باز می‌شود، مگر اینکه عمداً HTTPS را راه‌اندازی
کرده باشید، از localhost ویندوز استفاده کنید:

```text
http://127.0.0.1:18789/
```

به‌طور پیش‌فرض از IP شبکهٔ محلی استفاده نکنید. HTTP ساده روی نشانی شبکهٔ محلی یا tailnet
می‌تواند رفتار مربوط به مبدأ ناامن/احراز هویت دستگاه را فعال کند که ارتباطی با خود CDP ندارد. نگاه کنید به
[Control UI](/fa/web/control-ui).

## اعتبارسنجی لایه‌به‌لایه

از بالا به پایین پیش بروید؛ از هیچ مرحله‌ای عبور نکنید. رفع یک لایه ممکن است همچنان
خطای متفاوتی را از لایه‌ای پایین‌تر نمایان نگه دارد.

### لایهٔ 1: بررسی کنید Chrome در Windows، CDP را ارائه می‌کند

```powershell
chrome.exe --remote-debugging-port=9222 --user-data-dir="$env:LOCALAPPDATA\OpenClaw\ChromeCDP"
```

Chrome 136 و نسخه‌های بعدی، سوئیچ‌های خط فرمان اشکال‌زدایی راه‌دور را برای
پوشهٔ پیش‌فرض داده‌های Chrome نادیده می‌گیرند. مطابق نمونهٔ بالا از یک پوشهٔ دادهٔ جداگانه و
غیرپیش‌فرض استفاده کنید. نگاه کنید به
[تغییر امنیتی اشکال‌زدایی راه‌دور](https://developer.chrome.com/blog/remote-debugging-port)
در Chrome. این کار پروفایل عادی Chrome را که به حساب وارد شده است، از راه دور قابل‌کنترل نمی‌کند.

ابتدا از Windows خود Chrome را بررسی کنید:

```powershell
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://127.0.0.1:9222/json/list
```

اگر این کار ناموفق بود، شنونده‌های Windows را در ادامه عیب‌یابی کنید. هنوز مشکل از
OpenClaw نیست.

#### پیش از تغییر portproxy، IPv4 و IPv6 را عیب‌یابی کنید

Chromium ابتدا تلاش می‌کند اشکال‌زدایی راه‌دور را به `127.0.0.1` متصل کند و تنها در صورتی که
اتصال IPv4 ناموفق باشد، به `[::1]` برمی‌گردد. یک قانون پایدار `v4tov4` که روی
`127.0.0.1:9222` گوش می‌دهد، ممکن است پیش از شروع Chrome آن نقطهٔ پایانی را اشغال کند. سپس Chrome
به `[::1]:9222` برمی‌گردد، درحالی‌که قانون قدیمی ترافیک IPv4 را به شنوندهٔ
خودش بازمی‌گرداند و پاسخی خالی ارائه می‌کند.

به‌جای استنباط شنونده‌ها و قوانین پراکسی از نسخهٔ Chrome، آن‌ها را در Windows
بررسی کنید:

```powershell
netstat -ano | findstr :9222
netsh interface portproxy show all
curl.exe http://127.0.0.1:9222/json/version
curl.exe http://[::1]:9222/json/version
```

برای هر PID از `netstat`، از `tasklist /fi "PID eq <PID>"` استفاده کنید.

- اگر `chrome.exe` روی `127.0.0.1` پاسخ می‌دهد، هر قانون portproxy را که هم‌زمان
  روی `127.0.0.1:9222` گوش می‌دهد حذف کنید. فقط نشانی آداپتور Windows را که از WSL2 قابل‌دسترسی است
  به `127.0.0.1` هدایت کنید.
- اگر `chrome.exe` فقط روی `[::1]` پاسخ می‌دهد، شنوندهٔ قابل‌دسترسی از WSL2 را
  با `v4tov6` به `::1` هدایت کنید، نه به یک نشانی IPv4 استفاده‌نشده:

  ```powershell
  netsh interface portproxy add v4tov6 listenaddress=WINDOWS_HOST_OR_IP listenport=9222 connectaddress=::1 connectport=9222
  ```

شنونده را به نشانی آداپتوری متصل کنید که WSL2 به آن نیاز دارد. پورت CDP را
روی `0.0.0.0`، نشانی شبکهٔ محلی یا نشانی tailnet در معرض دسترسی قرار ندهید: CDP امکان کنترل
نشست مرورگر را می‌دهد.

### لایهٔ 2: بررسی کنید WSL2 می‌تواند به نقطهٔ پایانی Windows دسترسی پیدا کند

از WSL2، دقیقاً همان نشانی را که می‌خواهید در `cdpUrl` استفاده کنید آزمایش کنید:

```bash
curl http://WINDOWS_HOST_OR_IP:9222/json/version
curl http://WINDOWS_HOST_OR_IP:9222/json/list
```

نتیجهٔ مطلوب:

- `/json/version`، JSON حاوی فرادادهٔ Browser / Protocol-Version را برمی‌گرداند
- `/json/list`، JSON را برمی‌گرداند (اگر هیچ صفحه‌ای باز نیست، آرایهٔ خالی قابل‌قبول است)

اگر این کار ناموفق بود، Windows هنوز پورت را در دسترس WSL2 قرار نداده است، نشانی
برای سمت WSL2 نادرست است یا دیوار آتش/هدایت پورت/پراکسی وجود ندارد. پیش از دست‌زدن
به پیکربندی OpenClaw، این مورد را برطرف کنید.

### لایهٔ 3: پروفایل مرورگر صحیح را پیکربندی کنید

OpenClaw را به نشانی قابل‌دسترسی از WSL2 هدایت کنید:

```json5
{
  browser: {
    enabled: true,
    defaultProfile: "remote",
    profiles: {
      remote: {
        cdpUrl: "http://WINDOWS_HOST_OR_IP:9222",
        attachOnly: true,
        color: "#00AA00",
      },
    },
  },
}
```

نکات:

- از نشانی قابل‌دسترسی از WSL2 استفاده کنید، نه نشانی‌ای که فقط در Windows کار می‌کند
- برای مرورگرهایی که به‌صورت خارجی مدیریت می‌شوند، `attachOnly: true` را حفظ کنید
- `cdpUrl` می‌تواند `http://`، `https://`، `ws://` یا `wss://` باشد
- هنگامی که می‌خواهید OpenClaw، `/json/version` را کشف کند، از HTTP(S) استفاده کنید
- فقط زمانی از WS(S) استفاده کنید که ارائه‌دهندهٔ مرورگر یک URL مستقیم سوکت DevTools
  در اختیارتان قرار می‌دهد
- پیش از انتظار موفقیت OpenClaw، همان URL را با `curl` آزمایش کنید

### لایهٔ 4: لایهٔ Control UI را جداگانه بررسی کنید

`http://127.0.0.1:18789/` را از Windows باز کنید، سپس بررسی کنید:

- مبدأ صفحه با چیزی که `gateway.controlUi.allowedOrigins` انتظار دارد مطابقت دارد
- احراز هویت با توکن یا جفت‌سازی به‌درستی پیکربندی شده است
- یک مشکل احراز هویت Control UI را به‌اشتباه به‌عنوان مشکل مرورگر
  عیب‌یابی نمی‌کنید

صفحهٔ مفید: [Control UI](/fa/web/control-ui).

### لایهٔ 5: کنترل سرتاسری مرورگر را بررسی کنید

از WSL2:

```bash
openclaw browser --browser-profile remote open https://example.com
openclaw browser --browser-profile remote tabs
```

نتیجهٔ مطلوب:

- زبانه در Chrome ویندوز باز می‌شود
- `browser tabs`، هدف را برمی‌گرداند
- عملیات بعدی (`snapshot`، `screenshot`، `navigate`) از همان
  پروفایل کار می‌کنند

## خطاهای گمراه‌کنندهٔ رایج

| پیام                                                                                 | معنا                                                                                                                                                                           |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `control-ui-insecure-auth`                                                              | مشکل مبدأ رابط کاربری/بافت امن، نه مشکل انتقال CDP                                                                                                                     |
| `token_missing`                                                                         | مشکل پیکربندی احراز هویت                                                                                                                                                        |
| `pairing required`                                                                      | مشکل تأیید دستگاه                                                                                                                                                           |
| `Remote CDP for profile "remote" is not reachable`                                      | WSL2 نمی‌تواند به `cdpUrl` پیکربندی‌شده دسترسی پیدا کند                                                                                                                                         |
| پاسخ خالی CDP / `other side closed` از طریق portproxy                               | ناهماهنگی شنوندهٔ Windows یا حلقهٔ بازگشتی؛ هر دو خانوادهٔ loopback و `netsh interface portproxy show all` را بررسی کنید                                                                 |
| `Browser attachOnly is enabled and CDP websocket for profile "remote" is not reachable` | نقطهٔ پایانی HTTP پاسخ داد، اما WebSocket مربوط به DevTools باز نشد                                                                                                        |
| viewport قدیمی / بازنویسی‌های حالت تاریک / زبان / آفلاین پس از یک نشست راه‌دور          | برای بستن نشست و آزادکردن اتصال ذخیره‌شدهٔ Playwright/CDP بدون راه‌اندازی مجدد Gateway یا مرورگر خارجی، `openclaw browser --browser-profile remote stop` را اجرا کنید |
| پایان مهلت هنگام بررسی دسترسی‌پذیری CDP                                                         | معمولاً همچنان مشکل دسترسی‌پذیری CDP یا نقطهٔ پایانی راه‌دور کند/غیرقابل‌دسترسی است                                                                                                             |
| `Playwright page enumeration timed out after 3000ms`                                    | CDP راه‌دور متصل شد، اما خواندن پایدار زبانه متوقف ماند                                                                                                                     |
| `No Chrome tabs found for profile="user"`                                               | پروفایل محلی Chrome MCP در جایی انتخاب شده است که هیچ زبانهٔ محلیِ میزبان در دسترس نیست                                                                                                          |

## فهرست بررسی عیب‌یابی سریع

1. Windows: کدام‌یک از `127.0.0.1` یا `[::1]` روی `/json/version` پاسخ می‌دهد و
   آیا آن شنونده متعلق به `chrome.exe` است؟
2. WSL2: آیا `curl http://WINDOWS_HOST_OR_IP:9222/json/version` کار می‌کند؟
3. پیکربندی OpenClaw: آیا `browser.profiles.<name>.cdpUrl` دقیقاً از همان
   نشانی قابل‌دسترسی از WSL2 استفاده می‌کند؟
4. Control UI: آیا به‌جای IP شبکهٔ محلی، `http://127.0.0.1:18789/` را باز می‌کنید؟
5. آیا به‌جای CDP راه‌دور خام، تلاش می‌کنید از `existing-session` میان WSL2 و Windows
   استفاده کنید؟

ابتدا نقطهٔ پایانی Chrome در Windows را به‌صورت محلی بررسی کنید، سپس همان نقطهٔ پایانی را
از WSL2 بررسی کنید و تنها پس از آن پیکربندی OpenClaw یا احراز هویت Control UI را عیب‌یابی کنید.

## مرتبط

- [مرورگر](/fa/tools/browser)
- [ورود به مرورگر](/fa/tools/browser-login)
- [عیب‌یابی مرورگر در Linux](/fa/tools/browser-linux-troubleshooting)
