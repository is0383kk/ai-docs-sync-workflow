---
read_when:
    - می‌خواهید یک عامل OpenClaw به تماس Google Meet بپیوندد
    - می‌خواهید یک عامل OpenClaw تماس جدیدی در Google Meet ایجاد کند
    - در حال پیکربندی Chrome، Node کروم یا Twilio به‌عنوان مسیر انتقال Google Meet هستید
summary: 'Plugin گوگل Meet: پیوستن به URLهای صریح Meet از طریق Chrome یا Twilio با پیش‌فرض‌های پاسخ‌گویی صوتی عامل'
title: Plugin گوگل Meet
x-i18n:
    generated_at: "2026-07-27T14:22:30Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 8a611e283fe900984a29b563969936a641c7af430b05933eb03b98dc93b5d0c8
    source_path: plugins/google-meet.md
    workflow: 16
---

افزونهٔ `google-meet` از طرف یک عامل OpenClaw به نشانی‌های اینترنتی صریح Meet می‌پیوندد. دامنهٔ آن عمداً محدود است:

- این افزونه فقط به نشانی‌های اینترنتی `https://meet.google.com/...` می‌پیوندد؛ هرگز با استفاده از شماره تلفنی که خودش پیدا کرده است وارد جلسه نمی‌شود.
- `googlemeet create` می‌تواند از طریق Google Meet API (یا راهکار جایگزین مرورگر) یک نشانی اینترنتی جدید Meet ایجاد کند و به‌طور پیش‌فرض به آن بپیوندد.
- مشارکت از طریق Chrome از یک نمایهٔ واردشدهٔ Chrome، در صورت تمایل روی یک Node جفت‌شده، استفاده می‌کند. مشارکت از طریق Twilio با استفاده از [افزونهٔ تماس صوتی](/fa/plugins/voice-call) یک شماره تلفن را به‌همراه PIN/DTMF شماره‌گیری می‌کند؛ این روش نمی‌تواند مستقیماً یک نشانی اینترنتی Meet را شماره‌گیری کند.
- `mode: "agent"` (پیش‌فرض) گفتار شرکت‌کنندگان را با یک ارائه‌دهندهٔ بلادرنگ رونویسی می‌کند، آن را به عامل پیکربندی‌شدهٔ OpenClaw می‌فرستد و پاسخ را با TTS معمول OpenClaw پخش می‌کند. `mode: "bidi"` به یک مدل صوتی بلادرنگ اجازه می‌دهد مستقیماً پاسخ دهد. `mode: "transcribe"` فقط برای مشاهده و بدون پاسخ صوتی به جلسه می‌پیوندد.
- هنگامی که افزونه به تماس می‌پیوندد، هیچ اعلام خودکار رضایتی انجام نمی‌شود.
- دستور CLI برابر `googlemeet` است؛ `meet` برای گردش‌کارهای گسترده‌تر کنفرانس تلفنی عامل رزرو شده است.

## شروع سریع

افزونه و وابستگی‌های صوتی محلی را نصب کنید، سپس کلید یک ارائه‌دهندهٔ بلادرنگ را تنظیم کنید. OpenAI ارائه‌دهندهٔ پیش‌فرض رونویسی برای حالت `agent` است؛ Google Gemini Live به‌عنوان ارائه‌دهندهٔ صوتی حالت `bidi` در دسترس است:

```bash
openclaw plugins install npm:@openclaw/google-meet
brew install blackhole-2ch sox
export OPENAI_API_KEY=sk-...
# فقط زمانی لازم است که realtime.voiceProvider برای حالت bidi برابر "google" باشد
export GEMINI_API_KEY=...
```

`blackhole-2ch` دستگاه صوتی مجازی `BlackHole 2ch` را نصب می‌کند که Chrome صدا را از مسیر آن هدایت می‌کند. نصب‌کنندهٔ Homebrew پیش از آن‌که macOS دستگاه را در دسترس قرار دهد، به راه‌اندازی مجدد نیاز دارد:

```bash
sudo reboot
```

پس از راه‌اندازی مجدد، هر دو مؤلفه را بررسی کنید:

```bash
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

افزونه پس از نصب به‌طور پیش‌فرض فعال است. فقط برای سفارشی‌سازی آن یک ورودی اضافه کنید:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        config: {},
      },
    },
  },
}
```

اگر نمی‌خواهید افزونه فعال باشد، `openclaw plugins disable google-meet` را اجرا کنید.

راه‌اندازی را بررسی کنید، سپس به جلسه بپیوندید:

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

خروجی `setup` برای عامل قابل‌خواندن و از حالت/انتقال آگاه است: نمایهٔ Chrome، سنجاق‌شدن Node و، برای پیوستن‌های بلادرنگ Chrome، پل صوتی BlackHole/SoX و بررسی معرفی با تأخیر را گزارش می‌کند. پیوستن‌های فقط‌مشاهده پیش‌نیازهای بلادرنگ را نادیده می‌گیرند:

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
```

وقتی واگذاری Twilio پیکربندی شده باشد، `setup` همچنین گزارش می‌کند که آیا `voice-call`، اعتبارنامه‌های Twilio و دسترسی عمومی Webhook آماده هستند یا نه. پیش از پیوستن عامل، هر بررسی `ok: false` را برای آن انتقال/حالت یک مانع در نظر بگیرید. برای خروجی قابل‌خواندن توسط ماشین از `--json` و برای پیش‌بررسی یک انتقال مشخص از `--transport chrome|chrome-node|twilio` استفاده کنید:

```bash
openclaw googlemeet setup --transport twilio
```

یا اجازه دهید عامل از طریق ابزار `google_meet` به جلسه بپیوندد:

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

در میزبان‌های Gateway غیر از macOS، `google_meet` برای کنش‌های مصنوعات، تقویم، راه‌اندازی، رونویسی، Twilio و `chrome-node` همچنان قابل‌مشاهده است، اما پاسخ صوتی محلی Chrome (`transport: "chrome"` با `mode: "agent"` یا `"bidi"`) پیش از رسیدن به پل صوتی مسدود می‌شود، زیرا این مسیر در حال حاضر به `BlackHole 2ch` در macOS وابسته است. در عوض از `mode: "transcribe"`، ورود تلفنی Twilio یا یک میزبان `chrome-node` مبتنی بر macOS استفاده کنید.

### ایجاد جلسه

```bash
openclaw googlemeet create --transport chrome-node --mode agent
openclaw googlemeet create --no-join
```

`create` دو مسیر دارد که در فیلد `source` نتیجه گزارش می‌شوند:

- **`api`**: زمانی استفاده می‌شود که اعتبارنامه‌های OAuth برای Google Meet پیکربندی شده باشند. قطعی است؛ به وضعیت رابط کاربری مرورگر وابسته نیست.
- **`browser`**: بدون اعتبارنامه‌های OAuth استفاده می‌شود. OpenClaw نشانی `https://meet.google.com/new` را روی Node سنجاق‌شدهٔ Chrome باز می‌کند و منتظر می‌ماند تا Google به یک نشانی اینترنتی واقعی دارای کد جلسه هدایت کند؛ نمایهٔ Chrome مربوط به OpenClaw در آن Node باید از قبل وارد Google شده باشد. پیوستن و ایجاد، هر دو پیش از بازکردن زبانه‌ای جدید، از یک زبانهٔ موجود Meet (یا زبانهٔ در حال پردازش `.../new` / درخواست حساب Google) دوباره استفاده می‌کنند؛ تطبیق زبانه، رشته‌های پرس‌وجوی بی‌ضرر مانند `authuser` را نادیده می‌گیرد.

`create` به‌طور پیش‌فرض می‌پیوندد و `joined: true` را به‌همراه نشست پیوستن برمی‌گرداند. برای اینکه فقط نشانی اینترنتی ایجاد شود، `--no-join` (CLI) یا `"join": false` (ابزار) را ارسال کنید.

برای اتاق‌هایی که از طریق API ایجاد شده‌اند، به‌جای به‌ارث‌بردن پیش‌فرض حساب Google، یک سیاست دسترسی صریح تنظیم کنید:

```bash
openclaw googlemeet create --access-type OPEN --transport chrome-node --mode agent
```

| `--access-type` | چه کسانی می‌توانند بدون درخواست ورود بپیوندند                          |
| --------------- | ------------------------------------------------------------------- |
| `OPEN`          | هر کسی که نشانی اینترنتی Meet را داشته باشد                         |
| `TRUSTED`       | کاربران مورداعتماد سازمان میزبان، کاربران خارجی دعوت‌شده و کاربران ورودی تلفنی |
| `RESTRICTED`    | فقط دعوت‌شدگان                                                     |

این مورد فقط برای اتاق‌های ایجادشده از طریق API اعمال می‌شود، بنابراین OAuth باید پیکربندی شده باشد. اگر پیش از وجود این گزینه احراز هویت کرده‌اید، پس از افزودن دامنهٔ `meetings.space.settings` به صفحهٔ رضایت OAuth خود، `openclaw googlemeet auth login --json` را دوباره اجرا کنید.

اگر راهکار جایگزین مرورگر با مانع ورود Google یا مجوز Meet روبه‌رو شود، ابزار `manualActionRequired: true` را همراه با `manualActionReason`، `manualActionMessage` و `browser.nodeId`/`browser.targetId`/`browserUrl` برمی‌گرداند. آن پیام را گزارش کنید و تا زمانی که اپراتور مرحلهٔ مرورگر را تمام نکرده است، از بازکردن زبانه‌های جدید Meet خودداری کنید.

### پیوستن فقط برای مشاهده

برای نادیده‌گرفتن پل بلادرنگ دوطرفه (بدون نیاز به BlackHole/SoX و بدون پاسخ صوتی)، `"mode": "transcribe"` را تنظیم کنید. پیوستن‌های Chrome در حالت رونویسی نیز اعطای مجوز میکروفن/دوربین OpenClaw و مسیر **Use microphone** در Meet را نادیده می‌گیرند؛ اگر Meet صفحهٔ میانی انتخاب صدا را نمایش دهد، خودکارسازی ابتدا **Continue without microphone** را امتحان می‌کند. انتقال‌های مدیریت‌شدهٔ Chrome در هر حالت یک ناظر زیرنویس Meet را به‌شکل بهترین‌تلاش نصب می‌کنند تا یادداشت‌های ماندگار بدون تغییر مسیر زندهٔ مشورت با عامل در دسترس باشند. `googlemeet status --json` و `googlemeet doctor` مقادیر `captioning`، `captionsEnabledAttempted`، `transcriptLines`، `lastCaptionAt`، `lastCaptionSpeaker`، `lastCaptionText` و یک دنبالهٔ `recentTranscript` را گزارش می‌کنند.

برای رونوشت محدود نشست، دقیقاً همان زبانهٔ Meet ردیابی‌شده را بخوانید:

```bash
openclaw googlemeet transcript <session-id>
openclaw googlemeet transcript <session-id> --since <next-index> --json
```

ناظر حداکثر 2,000 خط تکمیل‌شدهٔ زیرنویس را در صفحهٔ Meet نگه می‌دارد. متن تدریجی قابل‌مشاهده تا زمان تکمیل ردیف زیرنویس در دنبالهٔ سلامت وضعیت باقی می‌ماند، بنابراین ذخیرهٔ `nextIndex` نمی‌تواند گسترش بعدی متن را از قلم بیندازد؛ خروج از جلسه، ردیف‌های قابل‌مشاهده را پیش از ثبت تصویر لحظه‌ای نهایی می‌کند. `droppedLines` تعداد خطوط ازدست‌رفته از ابتدای داده‌ها هنگام عبور از سقف را گزارش می‌کند. دنبالهٔ محدود `googlemeet transcript` همچنان فقط چهار نشست اخیراً پایان‌یافته را نگه می‌دارد و با Gateway بازنشانی می‌شود. به‌طور جداگانه، OpenClaw در طول جلسه ردیف‌های تکمیل‌شدهٔ زیرنویس را به پایگاه‌دادهٔ وضعیت مشترک می‌افزاید و هنگام خروج یک خلاصهٔ مشتق‌شده می‌نویسد. برای بررسی یا برون‌بری این یادداشت‌های ماندگار از [`openclaw transcripts`](/fa/cli/transcripts) استفاده کنید.

یادداشت‌برداری خودکار به‌طور پیش‌فرض فعال است. برای
غیرفعال‌کردن سراسری یادداشت‌های ماندگار، `transcripts.enabled: false` را تنظیم کنید؛ حالت صریح `transcribe` همچنان فقط
دنبالهٔ زندهٔ محدود خود را در دسترس قرار می‌دهد. پیوستن‌های Twilio جریان زیرنویس مرورگر را ندارند و
از طریق این مسیر ثبت نمی‌شوند.

برای یک کاوش شنیداری بله/خیر:

```bash
openclaw googlemeet test-listen <meet-url> --transport chrome-node
```

این دستور در حالت رونویسی به جلسه می‌پیوندد، منتظر حرکت تازهٔ زیرنویس/رونوشت می‌ماند و `listenVerified`، `listenTimedOut`، فیلدهای کنش دستی و سلامت فعلی زیرنویس را برمی‌گرداند.

### سلامت نشست بلادرنگ

در طول نشست‌های دارای پاسخ صوتی، وضعیت `google_meet` سلامت Chrome/پل صوتی را گزارش می‌کند: `inCall`، `manualActionRequired`، `providerConnected`، `realtimeReady`، `audioInputActive`، `audioOutputActive`، آخرین مُهرهای زمانی ورودی/خروجی، شمارنده‌های بایت و وضعیت بسته‌شدن پل. نشست‌های مدیریت‌شدهٔ Chrome فقط پس از آن عبارت معرفی/آزمایش را پخش می‌کنند که سلامت، `inCall: true` را گزارش کند؛ در غیر این صورت `speechReady: false` و تلاش برای گفتار مسدود می‌شود، نه اینکه بی‌سروصدا هیچ کاری انجام ندهد.

پیوستن‌های محلی Chrome از نمایهٔ واردشدهٔ مرورگر OpenClaw استفاده می‌کنند و برای مسیر میکروفن/بلندگو به `BlackHole 2ch` نیاز دارند. یک دستگاه BlackHole برای نخستین آزمایش دود کافی است، اما ممکن است پژواک ایجاد کند؛ برای صدای دوطرفهٔ پاک از دستگاه‌های مجازی جداگانه یا یک گراف به‌سبک Loopback استفاده کنید.

## Gateway محلی + Chrome در Parallels

برای اینکه صرفاً Chrome را در اختیار یک ماشین مجازی macOS قرار دهید، وجود Gateway کامل یا کلید API مدل درون ماشین مجازی ضروری نیست. Gateway و عامل را به‌صورت محلی اجرا کنید؛ یک میزبان Node را در ماشین مجازی اجرا کنید.

| محل اجرا             | موارد                                                                                           |
| -------------------- | ----------------------------------------------------------------------------------------------- |
| میزبان Gateway       | OpenClaw Gateway، فضای کاری عامل، کلیدهای مدل/API، ارائه‌دهندهٔ بلادرنگ، پیکربندی افزونهٔ Google Meet |
| ماشین مجازی macOS در Parallels | میزبان CLI/Node مربوط به OpenClaw، Chrome، SoX، BlackHole 2ch و یک نمایهٔ Chrome واردشده به Google |
| موارد غیرضروری در ماشین مجازی | سرویس Gateway، پیکربندی عامل، راه‌اندازی ارائه‌دهندهٔ مدل                                  |

وابستگی‌های ماشین مجازی را نصب، راه‌اندازی مجدد و بررسی کنید:

```bash
brew install blackhole-2ch sox
sudo reboot
system_profiler SPAudioDataType | grep -i BlackHole
command -v sox
```

افزونه را در ماشین مجازی، جایی که به‌طور پیش‌فرض فعال می‌شود، نصب کنید و میزبان Node را راه‌اندازی کنید:

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw node run --host <gateway-host> --port 18789 --display-name parallels-macos
```

اگر `<gateway-host>` یک نشانی IP شبکهٔ محلی بدون TLS است، استفاده از آن شبکهٔ خصوصی مورداعتماد را صریحاً مجاز کنید:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

هنگام نصب به‌عنوان LaunchAgent نیز از همان پرچم استفاده کنید (این یک متغیر محیطی فرایند است که اگر در دستور نصب وجود داشته باشد در محیط LaunchAgent ذخیره می‌شود، نه یک تنظیم `openclaw.json`):

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install --host <gateway-lan-ip> --port 18789 --display-name parallels-macos --force
openclaw node restart
```

Node را از میزبان Gateway تأیید کنید، سپس مطمئن شوید که هر دو قابلیت `googlemeet.chrome` و مرورگر/`browser.proxy` را اعلام می‌کند:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Meet را از طریق آن Node هدایت کنید:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["googlemeet.chrome", "browser.proxy"] },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          chrome: {
            guestName: "OpenClaw Agent",
            autoJoin: true,
            reuseExistingTab: true,
          },
          chromeNode: {
            node: "parallels-macos",
          },
        },
      },
    },
  },
}
```

اکنون به‌صورت معمول از میزبان Gateway به جلسه بپیوندید:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij
```

برای یک آزمایش دود تک‌دستوری که یک نشست ایجاد می‌کند یا از نشست موجود دوباره استفاده می‌کند، عبارتی شناخته‌شده را پخش می‌کند و سلامت نشست را چاپ می‌کند:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij
```

هنگام پیوستن بلادرنگ، خودکارسازی مرورگر نام مهمان را پر می‌کند، روی Join/Ask to join کلیک می‌کند و در صورت نمایش، اعلان نخستین اجرای Meet با عنوان "Use microphone" را می‌پذیرد (یا هنگام پیوستن صرفاً برای مشاهده و ایجاد جلسه فقط با مرورگر، "Continue without microphone" را انتخاب می‌کند). اگر نمایه از حساب خارج شده باشد، Meet منتظر پذیرش میزبان باشد، Chrome به مجوز میکروفون/دوربین نیاز داشته باشد، یا Meet روی یک اعلان حل‌نشده گیر کرده باشد، نتیجه `manualActionRequired: true` را همراه با `manualActionReason` و `manualActionMessage` گزارش می‌کند. تلاش مجدد را متوقف کنید، آن پیام را به‌همراه `browserUrl`/`browserTitle` گزارش دهید و فقط پس از تکمیل اقدام دستی دوباره تلاش کنید.

اگر `chromeNode.node` حذف شده باشد، OpenClaw تنها زمانی به‌طور خودکار انتخاب می‌کند که دقیقاً یک Node متصل، هم `googlemeet.chrome` و هم کنترل مرورگر را ارائه دهد؛ وقتی چند Node توانمند متصل‌اند، `chromeNode.node` (شناسه Node، نام نمایشی یا IP راه دور) را پین کنید.

### بررسی خطاهای رایج

| نشانه                                                  | راه‌حل                                                                                                                                                                                                                                                                                   |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `Configured Google Meet node ... is not usable: offline` | Node پین‌شده شناخته شده اما در دسترس نیست. مانع راه‌اندازی را گزارش دهید؛ مگر در صورت درخواست، بی‌سروصدا به انتقال دیگری بازنگردید.                                                                                                                                                      |
| `No connected Google Meet-capable node`                  | `npm:@openclaw/google-meet` را در ماشین مجازی نصب کنید، `openclaw plugins enable browser` را اجرا کنید، `openclaw node run` را راه‌اندازی کنید و جفت‌سازی را تأیید کنید. اگر Google Meet صراحتاً غیرفعال شده است، آن را نیز فعال کنید. تأیید کنید که `gateway.nodes.commands.allow` شامل `googlemeet.chrome` و `browser.proxy` است. |
| `BlackHole 2ch audio device not found`                   | `blackhole-2ch` را روی میزبان مورد بررسی نصب و آن را راه‌اندازی مجدد کنید.                                                                                                                                                                                                                         |
| `BlackHole 2ch audio device not found on the node`       | `blackhole-2ch` را در ماشین مجازی نصب و ماشین مجازی را راه‌اندازی مجدد کنید.                                                                                                                                                                                                                                  |
| Chrome باز می‌شود اما نمی‌تواند بپیوندد                             | در ماشین مجازی وارد نمایه مرورگر شوید، یا `chrome.guestName` را تنظیم‌شده نگه دارید. پیوستن خودکار مهمان از خودکارسازی مرورگر OpenClaw از طریق پراکسی مرورگر Node استفاده می‌کند؛ `browser.defaultProfile` مربوط به Node (یا یک نمایه نام‌گذاری‌شده نشست موجود) را به نمایه موردنظر هدایت کنید.                   |
| برگه‌های تکراری Meet                                      | `chrome.reuseExistingTab: true` را باقی بگذارید. OpenClaw برگه موجود برای همان URL را فعال می‌کند و پیش از باز کردن برگه‌ای دیگر، فرایند ایجاد از `.../new` در حال انجام یا برگه اعلان حساب Google دوباره استفاده می‌کند.                                                                                        |
| بدون صدا                                                 | صدای میکروفون/بلندگوی Meet را از مسیر صوتی مجازی مورد استفاده OpenClaw عبور دهید؛ برای صدای دوطرفه شفاف، از دستگاه‌های مجازی جداگانه یا مسیریابی به سبک Loopback استفاده کنید.                                                                                                                                |

## نکات نصب

حالت پیش‌فرض بازگشت صدای Chrome از دو ابزار خارجی استفاده می‌کند که OpenClaw آن‌ها را بسته‌بندی یا بازتوزیع نمی‌کند؛ آن‌ها را به‌عنوان وابستگی‌های میزبان از طریق Homebrew نصب کنید:

- `sox`: ابزار صوتی خط فرمان. Plugin برای پل صوتی پیش‌فرض PCM16 با فرکانس 24 kHz، فرمان‌های صریح دستگاه CoreAudio را صادر می‌کند.
- `blackhole-2ch`: درایور صوتی مجازی macOS که دستگاه `BlackHole 2ch` را برای مسیریابی Chrome/Meet فراهم می‌کند.

SoX تحت مجوز `LGPL-2.0-only AND GPL-2.0-only` است؛ BlackHole تحت GPL-3.0 است. اگر نصب‌کننده یا دستگاهی می‌سازید که BlackHole را همراه OpenClaw بسته‌بندی می‌کند، مجوز بالادستی BlackHole را بررسی کنید یا مجوز جداگانه‌ای از Existential Audio بگیرید.

## انتقال‌ها

| انتقال     | زمان استفاده                                                                                     |
| ------------- | -------------------------------------------------------------------------------------------- |
| `chrome`      | Chrome/صدا روی میزبان Gateway اجرا می‌شوند                                                        |
| `chrome-node` | Chrome/صدا روی یک Node جفت‌شده اجرا می‌شوند (برای مثال، یک ماشین مجازی macOS در Parallels)                        |
| `twilio`      | بازگشت به تماس تلفنی از طریق Plugin تماس صوتی، هنگامی که مشارکت با Chrome در دسترس نیست |

### Chrome

URL مربوط به Meet را از طریق کنترل مرورگر OpenClaw باز می‌کند و با نمایه مرورگر OpenClaw که وارد حساب شده است می‌پیوندد. در macOS، Plugin پیش از راه‌اندازی وجود `BlackHole 2ch` را بررسی می‌کند و در صورت پیکربندی، پیش از باز کردن Chrome فرمان سلامت/راه‌اندازی پل صوتی را اجرا می‌کند. برای Chrome محلی، نمایه را با `browser.defaultProfile` انتخاب کنید؛ در عوض، `chrome.browserProfile` به میزبان‌های `chrome-node` ارسال می‌شود.

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome
openclaw googlemeet join https://meet.google.com/abc-defg-hij --transport chrome-node
```

صدای میکروفون/بلندگوی Chrome از پل صوتی محلی OpenClaw عبور می‌کند. اگر `BlackHole 2ch` نصب نباشد، پیوستن به‌جای ورود بدون مسیر صوتی، با خطای راه‌اندازی ناموفق می‌شود.

### Twilio

یک طرح شماره‌گیری سخت‌گیرانه که به [Plugin تماس صوتی](/fa/plugins/voice-call) واگذار می‌شود. این طرح صفحات Meet را برای یافتن شماره تلفن تجزیه نمی‌کند؛ Google Meet باید شماره تماس ورودی و PIN جلسه را ارائه دهد.

تماس صوتی را روی میزبان Gateway فعال کنید، نه روی Node مربوط به Chrome:

```json5
{
  plugins: {
    allow: ["google-meet", "voice-call", "google"],
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          defaultTransport: "chrome-node",
          // یا اگر Twilio باید پیش‌فرض باشد، "twilio" را تنظیم کنید
        },
      },
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          inboundPolicy: "allowlist",
          realtime: {
            enabled: true,
            provider: "google",
            instructions: "به‌عنوان عامل OpenClaw به این Google Meet بپیوندید. کوتاه صحبت کنید.",
            toolPolicy: "safe-read-only",
            providers: {
              google: {
                silenceDurationMs: 500,
                startSensitivity: "high",
              },
            },
          },
        },
      },
      google: {
        enabled: true,
      },
    },
  },
}
```

اعتبارنامه‌های Twilio را از طریق محیط ارائه دهید تا اسرار خارج از `openclaw.json` نگه داشته شوند:

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
export GEMINI_API_KEY=...
```

اگر OpenAI ارائه‌دهنده صدای بلادرنگ است، در عوض از `realtime.provider: "openai"` همراه با `OPENAI_API_KEY` استفاده کنید.

پس از فعال‌سازی `voice-call`، Gateway را راه‌اندازی مجدد یا بارگذاری مجدد کنید؛ تغییرات پیکربندی Plugin تا زمان بارگذاری مجدد اعمال نمی‌شوند. تأیید کنید:

```bash
openclaw config validate
openclaw plugins list | grep -E 'google-meet|voice-call'
openclaw googlemeet setup
```

وقتی واگذاری Twilio متصل باشد، `googlemeet setup` شامل بررسی‌های `twilio-voice-call-plugin`، `twilio-voice-call-credentials` و `twilio-voice-call-webhook` است.

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

برای یک توالی سفارشی از `--dtmf-sequence` استفاده کنید و برای مکث پیش از PIN، `w` یا ویرگول‌هایی را در ابتدای آن قرار دهید:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

## OAuth و بررسی پیش از اجرا

OAuth برای ایجاد پیوند Meet اختیاری است، زیرا `googlemeet create` می‌تواند به خودکارسازی مرورگر بازگردد. OAuth را برای ایجاد از طریق API رسمی، تفکیک فضا یا بررسی پیش از اجرای Meet Media API پیکربندی کنید. پیوستن‌های Chrome/Chrome-node هرگز به OAuth وابسته نیستند؛ در هر صورت از نمایه Chrome واردشده به حساب، BlackHole/SoX و (برای `chrome-node`) یک Node متصل استفاده می‌کنند.

### ایجاد اعتبارنامه‌های Google

در Google Cloud Console:

<Steps>
<Step title="یک پروژه ایجاد یا انتخاب کنید">
</Step>
<Step title="Google Meet REST API را فعال کنید">
</Step>
<Step title="صفحه رضایت OAuth را پیکربندی کنید">
Internal برای یک سازمان Google Workspace ساده‌ترین گزینه است. External برای راه‌اندازی‌های شخصی/آزمایشی کار می‌کند؛ تا زمانی که برنامه در حالت Testing است، هر حساب Google را که قرار است به آن مجوز دهد به‌عنوان کاربر آزمایشی اضافه کنید.
</Step>
<Step title="دامنه‌های درخواستی را اضافه کنید">
- `https://www.googleapis.com/auth/meetings.space.created`
- `https://www.googleapis.com/auth/meetings.space.readonly`
- `https://www.googleapis.com/auth/meetings.space.settings`
- `https://www.googleapis.com/auth/meetings.conference.media.readonly`
- `https://www.googleapis.com/auth/calendar.events.readonly` (جست‌وجوی تقویم)
- `https://www.googleapis.com/auth/drive.meet.readonly` (خروجی‌گرفتن از متن بدنه سند رونوشت/یادداشت هوشمند)

</Step>
<Step title="یک شناسه کلاینت OAuth ایجاد کنید">
نوع برنامه **Web application**. نشانی URI مجاز برای تغییر مسیر:

```text
http://localhost:8085/oauth2callback
```

</Step>
<Step title="شناسه کلاینت و راز کلاینت را کپی کنید">
</Step>
</Steps>

`meetings.space.created` برای `spaces.create` الزامی است. `meetings.space.readonly` نشانی‌های URL/کدهای Meet را به فضاها تفکیک می‌کند. `meetings.space.settings` به OpenClaw اجازه می‌دهد هنگام ایجاد اتاق از طریق API، تنظیمات `SpaceConfig` مانند `accessType` را ارسال کند. `meetings.conference.media.readonly` برای بررسی پیش از اجرای Meet Media API و کارهای رسانه‌ای است؛ Google ممکن است برای استفاده واقعی از Media API، ثبت‌نام در Developer Preview را الزامی کند. `calendar.events.readonly` فقط برای جست‌وجوی تقویم `--today`/`--event` لازم است. `drive.meet.readonly` فقط برای خروجی‌گرفتن `--include-doc-bodies` لازم است. اگر فقط به پیوستن مبتنی بر مرورگر Chrome نیاز دارید، OAuth را کاملاً نادیده بگیرید.

### ایجاد توکن تازه‌سازی

`oauth.clientId` و در صورت تمایل `oauth.clientSecret` را پیکربندی کنید (یا آن‌ها را به‌عنوان متغیرهای محیطی ارسال کنید)، سپس اجرا کنید:

```bash
openclaw googlemeet auth login --json
```

این فرمان یک جریان PKCE را با فراخوان بازگشتی localhost روی `http://localhost:8085/oauth2callback` اجرا می‌کند و یک بلوک پیکربندی `oauth` حاوی توکن تازه‌سازی چاپ می‌کند. هنگامی که مرورگر نمی‌تواند به فراخوان بازگشتی محلی دسترسی پیدا کند، برای جریان کپی/جای‌گذاری `--manual` را اضافه کنید:

```bash
OPENCLAW_GOOGLE_MEET_CLIENT_ID="your-client-id" \
OPENCLAW_GOOGLE_MEET_CLIENT_SECRET="your-client-secret" \
openclaw googlemeet auth login --json --manual
```

خروجی JSON:

```json
{
  "oauth": {
    "clientId": "your-client-id",
    "clientSecret": "your-client-secret",
    "refreshToken": "refresh-token",
    "accessToken": "access-token",
    "expiresAt": 1770000000000
  },
  "scope": "..."
}
```

شیء `oauth` را در پیکربندی Plugin ذخیره کنید:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {
          oauth: {
            clientId: "your-client-id",
            clientSecret: "your-client-secret",
            refreshToken: "refresh-token",
          },
        },
      },
    },
  },
}
```

اگر نمی‌خواهید توکن تازه‌سازی در پیکربندی باشد، متغیرهای محیطی را ترجیح دهید؛ ابتدا پیکربندی تفکیک می‌شود و سپس محیط به‌عنوان حالت بازگشتی استفاده می‌شود. اگر پیش از وجود پشتیبانی از ایجاد جلسه، جست‌وجوی تقویم یا خروجی‌گرفتن از بدنه سند احراز هویت کرده‌اید، `openclaw googlemeet auth login --json` را دوباره اجرا کنید تا توکن تازه‌سازی مجموعه دامنه‌های فعلی را پوشش دهد.

### تأیید OAuth با doctor

```bash
openclaw googlemeet doctor --oauth --json
```

این بررسی می‌کند که پیکربندی OAuth وجود داشته باشد و توکن نوسازی بتواند یک توکن دسترسی صادر کند، بدون بارگذاری زمان‌اجرای Chrome یا نیاز به Node متصل. گزارش فقط شامل فیلدهای وضعیت (`ok`، `configured`، `tokenSource`، `expiresAt`، پیام‌های بررسی) است و هرگز توکن دسترسی، توکن نوسازی یا راز کلاینت را چاپ نمی‌کند.

| بررسی                | معنا                                                                          |
| -------------------- | -------------------------------------------------------------------------------- |
| `oauth-config`       | `oauth.clientId` به‌همراه `oauth.refreshToken`، یا یک توکن دسترسی ذخیره‌شده در حافظهٔ نهان، موجود است |
| `oauth-token`        | توکن دسترسی ذخیره‌شده در حافظهٔ نهان همچنان معتبر است، یا توکن نوسازی توکن جدیدی صادر کرده است    |
| `meet-spaces-get`    | بررسی اختیاری `--meeting` یک فضای Meet موجود را پیدا کرد                       |
| `meet-spaces-create` | بررسی اختیاری `--create-space` یک فضای Meet جدید ایجاد کرد                         |

فعال‌بودن Meet API و دامنهٔ `spaces.create` را با بررسی ایجادِ دارای اثر جانبی اثبات کنید:

```bash
openclaw googlemeet doctor --oauth --create-space --json
```

دسترسی خواندن به یک فضای موجود را اثبات کنید:

```bash
openclaw googlemeet doctor --oauth --meeting https://meet.google.com/abc-defg-hij --json
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
```

یک `403` از این بررسی‌ها معمولاً یعنی Meet REST API غیرفعال است، توکن نوسازی دامنهٔ موردنیاز را ندارد، یا حساب Google نمی‌تواند به آن فضا دسترسی داشته باشد. خطای توکن نوسازی یعنی `openclaw googlemeet auth login --json` را دوباره اجرا کنید و بلوک جدید `oauth` را ذخیره کنید.

برای جایگزین مرورگر به OAuth نیازی نیست؛ احراز هویت Google در آنجا از نمایهٔ واردشدهٔ Chrome روی Node انتخاب‌شده می‌آید، نه از پیکربندی OpenClaw.

این متغیرهای محیطی به‌عنوان جایگزین پذیرفته می‌شوند:

- `OPENCLAW_GOOGLE_MEET_CLIENT_ID` یا `GOOGLE_MEET_CLIENT_ID`
- `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET` یا `GOOGLE_MEET_CLIENT_SECRET`
- `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` یا `GOOGLE_MEET_REFRESH_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN` یا `GOOGLE_MEET_ACCESS_TOKEN`
- `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` یا `GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT`
- `OPENCLAW_GOOGLE_MEET_DEFAULT_MEETING` یا `GOOGLE_MEET_DEFAULT_MEETING`
- `OPENCLAW_GOOGLE_MEET_PREVIEW_ACK` یا `GOOGLE_MEET_PREVIEW_ACK`

### یافتن، بررسی پیش‌نیازها و خواندن مصنوعات

```bash
openclaw googlemeet resolve-space --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet preflight --meeting https://meet.google.com/abc-defg-hij
```

پس از آنکه Meet رکوردهای کنفرانس را ایجاد کرد:

```bash
openclaw googlemeet artifacts --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet attendance --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet export --meeting https://meet.google.com/abc-defg-hij --output ./meet-export
```

با `--meeting`، `artifacts` و `attendance` به‌طور پیش‌فرض از جدیدترین رکورد کنفرانس استفاده می‌شود؛ برای هر رکورد نگه‌داری‌شده `--all-conference-records` را ارسال کنید.

جست‌وجوی تقویم پیش از خواندن مصنوعات، نشانی جلسه را از Google Calendar پیدا می‌کند (به توکن نوسازی شامل دامنهٔ فقط‌خواندنی رویدادهای Calendar نیاز دارد):

```bash
openclaw googlemeet latest --today
openclaw googlemeet calendar-events --today --json
openclaw googlemeet artifacts --event "Weekly sync"
openclaw googlemeet attendance --today --format csv --output attendance.csv
```

`--today` در تقویم `primary` امروز، رویدادی دارای پیوند Meet را جست‌وجو می‌کند؛ `--event <query>` متن رویداد منطبق را جست‌وجو می‌کند؛ `--calendar <id>` یک تقویم غیر اصلی را هدف می‌گیرد. `calendar-events` پیش‌نمایش رویدادهای منطبق را نمایش می‌دهد و مشخص می‌کند `latest`/`artifacts`/`attendance`/`export` کدام‌یک را انتخاب خواهد کرد.

اگر شناسهٔ رکورد کنفرانس را از قبل می‌دانید، مستقیماً به آن ارجاع دهید:

```bash
openclaw googlemeet latest --meeting https://meet.google.com/abc-defg-hij
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 --json
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 --json
```

اتاق یک فضای ایجادشده با API را ببندید:

```bash
openclaw googlemeet end-active-conference https://meet.google.com/abc-defg-hij
```

`spaces.endActiveConference` را فراخوانی می‌کند و برای فضایی که حساب مجاز می‌تواند مدیریت کند، به OAuth با دامنهٔ `meetings.space.created` نیاز دارد. نشانی Meet، کد جلسه یا `spaces/{id}` را می‌پذیرد و ابتدا آن را به منبع فضای API تبدیل می‌کند. این با `googlemeet leave` متفاوت است: `leave` مشارکت محلی/نشست OpenClaw را متوقف می‌کند؛ `end-active-conference` از Google Meet می‌خواهد کنفرانس فعال آن فضا را پایان دهد.

یک گزارش خوانا بنویسید:

```bash
openclaw googlemeet artifacts --conference-record conferenceRecords/abc123 \
  --format markdown --output meet-artifacts.md
openclaw googlemeet attendance --conference-record conferenceRecords/abc123 \
  --format csv --output meet-attendance.csv
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --zip --output meet-export
openclaw googlemeet export --conference-record conferenceRecords/abc123 \
  --include-doc-bodies --dry-run
```

`artifacts` فرادادهٔ رکورد کنفرانس را به‌همراه فرادادهٔ منابع شرکت‌کننده، ضبط، رونوشت، ورودی ساخت‌یافتهٔ رونوشت و یادداشت هوشمند، در صورت ارائهٔ آن‌ها توسط Google، برمی‌گرداند. `--no-transcript-entries` جست‌وجوی ورودی‌ها را برای جلسه‌های بزرگ رد می‌کند. `attendance` شرکت‌کنندگان را به ردیف‌های نشست شرکت‌کننده گسترش می‌دهد که شامل زمان‌های نخستین/آخرین مشاهده، مدت کل نشست، پرچم‌های تأخیر/خروج زودهنگام و منابع تکراری شرکت‌کننده است که بر اساس کاربر واردشده یا نام نمایشی ادغام شده‌اند؛ `--no-merge-duplicates` منابع خام را جدا نگه می‌دارد و `--late-after-minutes`/`--early-before-minutes` آستانه‌ها را تنظیم می‌کنند.

`export` پوشه‌ای شامل `summary.md`، `attendance.csv`، `transcript.md`، `artifacts.json`، `attendance.json` و `manifest.json` می‌نویسد. `manifest.json` ورودی انتخاب‌شده، گزینه‌های برون‌بری، رکوردهای کنفرانس، فایل‌های خروجی، تعدادها، منبع توکن، هر رویداد Calendar استفاده‌شده و هشدارهای بازیابی ناقص را ثبت می‌کند. `--zip` همچنین بایگانی قابل‌انتقالی کنار پوشه می‌نویسد. `--include-doc-bodies` متن Google Docs مربوط به رونوشت/یادداشت هوشمند پیوندشده را از طریق Drive `files.export` برون‌بری می‌کند (به دامنهٔ فقط‌خواندنی Drive Meet نیاز دارد)؛ بدون آن، برون‌بری‌ها فقط شامل فرادادهٔ Meet و ورودی‌های ساخت‌یافتهٔ رونوشت هستند. شکست جزئی یک مصنوع (فهرست‌کردن یادداشت هوشمند، ورودی رونوشت یا خطای بدنهٔ سند) به‌جای شکست کل برون‌بری، هشدار را در خلاصه/مانیفست نگه می‌دارد. `--dry-run` همان داده‌ها را دریافت و JSON مانیفست را بدون ایجاد پوشه یا ZIP چاپ می‌کند.

عامل‌ها همین کنش‌ها را از طریق ابزار `google_meet` استفاده می‌کنند (`export`، `create` با `accessType`، `end_active_conference`، `test_listen`)؛ [ابزار](#tool) را ببینید.

### آزمون دود زنده

```bash
OPENCLAW_LIVE_TEST=1 \
OPENCLAW_GOOGLE_MEET_LIVE_MEETING=https://meet.google.com/abc-defg-hij \
pnpm test:live -- extensions/google-meet/google-meet.live.test.ts
```

```bash
openclaw googlemeet setup --transport chrome-node --mode transcribe
openclaw googlemeet test-listen https://meet.google.com/abc-defg-hij --transport chrome-node --timeout-ms 30000
```

| متغیر                                                                                                                  | هدف                                                                |
| ------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| `OPENCLAW_LIVE_TEST=1`                                                                                                    | آزمون‌های زندهٔ محافظت‌شده را فعال می‌کند                                             |
| `OPENCLAW_GOOGLE_MEET_LIVE_MEETING`                                                                                       | نشانی Meet نگه‌داری‌شده، کد یا `spaces/{id}`                              |
| `OPENCLAW_GOOGLE_MEET_CLIENT_ID` / `GOOGLE_MEET_CLIENT_ID`                                                                | شناسهٔ کلاینت OAuth                                                        |
| `OPENCLAW_GOOGLE_MEET_REFRESH_TOKEN` / `GOOGLE_MEET_REFRESH_TOKEN`                                                        | توکن نوسازی                                                          |
| `OPENCLAW_GOOGLE_MEET_CLIENT_SECRET`، `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN`، `OPENCLAW_GOOGLE_MEET_ACCESS_TOKEN_EXPIRES_AT` | اختیاری؛ همان نام‌های جایگزین بدون پیشوند `OPENCLAW_` نیز کار می‌کنند |

آزمون دود پایهٔ مصنوعات/حضور به `meetings.space.readonly` و `meetings.conference.media.readonly` نیاز دارد. جست‌وجوی Calendar به `calendar.events.readonly` نیاز دارد. برون‌بری بدنهٔ سند Drive به `drive.meet.readonly` نیاز دارد.

### نمونه‌های ایجاد

```bash
openclaw googlemeet create
```

نشانی جدید جلسه، منبع و نشست پیوستن را چاپ می‌کند. با OAuth از Meet API استفاده می‌کند؛ بدون آن، از نمایهٔ واردشدهٔ Node سنجاق‌شدهٔ Chrome استفاده می‌کند. JSON جایگزین مرورگر:

```json
{
  "source": "مرورگر",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

اگر جایگزین مرورگر ابتدا با ورود به Google یا مانع مجوز Meet روبه‌رو شود، `google_meet` به‌جای یک رشتهٔ ساده، جزئیات ساخت‌یافته برمی‌گرداند:

```json
{
  "source": "مرورگر",
  "error": "google-login-required: در نمایهٔ مرورگر OpenClaw وارد Google شوید، سپس ایجاد جلسه را دوباره امتحان کنید.",
  "manualActionRequired": true,
  "manualActionReason": "google-login-required",
  "manualActionMessage": "در نمایهٔ مرورگر OpenClaw وارد Google شوید، سپس ایجاد جلسه را دوباره امتحان کنید.",
  "browser": {
    "nodeId": "ba0f4e4bc...",
    "targetId": "tab-1",
    "browserUrl": "https://accounts.google.com/signin",
    "browserTitle": "ورود - حساب‌های Google"
  }
}
```

JSON ایجاد API:

```json
{
  "source": "api",
  "meetingUri": "https://meet.google.com/abc-defg-hij",
  "joined": true,
  "space": {
    "name": "spaces/abc-defg-hij",
    "meetingCode": "abc-defg-hij",
    "meetingUri": "https://meet.google.com/abc-defg-hij"
  },
  "join": {
    "session": {
      "id": "meet_...",
      "url": "https://meet.google.com/abc-defg-hij"
    }
  }
}
```

ایجاد به‌طور پیش‌فرض ملحق می‌شود، اما Chrome/Chrome-node همچنان برای پیوستن از طریق مرورگر به نمایهٔ Google واردشده نیاز دارد؛ اگر از حساب خارج شده باشد، OpenClaw خطای `manualActionRequired: true` یا خطای جایگزین مرورگر را گزارش می‌کند و از اپراتور می‌خواهد پیش از تلاش مجدد، ورود به Google را تکمیل کند.

`preview.enrollmentAcknowledged: true` را فقط پس از تأیید اینکه پروژهٔ Cloud، هویت OAuth و شرکت‌کنندگان جلسه در Google Workspace Developer Preview Program برای Meet media APIs ثبت‌نام شده‌اند تنظیم کنید.

## پیکربندی

مسیر رایج عامل Chrome فقط به Plugin فعال، BlackHole، SoX، کلید ارائه‌دهندهٔ بلادرنگ و ارائه‌دهندهٔ TTS پیکربندی‌شدهٔ OpenClaw نیاز دارد:

```json5
{
  plugins: {
    entries: {
      "google-meet": {
        enabled: true,
        config: {},
      },
    },
  },
}
```

### پیش‌فرض‌ها

| کلید                               | پیش‌فرض                                  | یادداشت‌ها                                                                                                                                                                                                             |
| --------------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `defaultTransport`                | `"chrome"`                               |                                                                                                                                                                                                                   |
| `defaultMode`                     | `"agent"`                                | `"realtime"` به‌عنوان نام مستعار قدیمی برای `"agent"` پذیرفته می‌شود؛ فراخواننده‌های جدید باید از `"agent"` استفاده کنند                                                                                                                        |
| `chromeNode.node`                 | تنظیم‌نشده                                    | شناسه/نام/IP ‏Node برای `chrome-node`؛ هنگامی‌که ممکن است بیش از یک Node دارای قابلیت متصل باشد، الزامی است                                                                                                                      |
| `chrome.launch`                   | `true`                                   | Chrome را برای پیوستن اجرا می‌کند؛ `false` را فقط هنگام استفادهٔ مجدد از یک نشست ازپیش‌باز تنظیم کنید                                                                                                                                 |
| `chrome.audioBackend`             | `"blackhole-2ch"`                        |                                                                                                                                                                                                                   |
| `chrome.guestName`                | `"OpenClaw Agent"`                       | در صفحهٔ مهمان Meet در حالت خارج‌شده از حساب نمایش داده می‌شود                                                                                                                                                                         |
| `chrome.autoJoin`                 | `true`                                   | تلاش برای پرکردن نام مهمان و کلیک روی Join Now در `chrome-node` بدون تضمین موفقیت                                                                                                                                                   |
| `chrome.reuseExistingTab`         | `true`                                   | به‌جای بازکردن زبانه‌های تکراری، یک زبانهٔ موجود Meet را فعال می‌کند                                                                                                                                                      |
| `chrome.waitForInCallMs`          | `20000`                                  | پیش از پخش مقدمهٔ پاسخ گفتاری، منتظر می‌ماند تا زبانهٔ Meet حضور در تماس را گزارش کند                                                                                                                                          |
| `chrome.audioFormat`              | `"pcm16-24khz"`                          | قالب صوتی جفت‌فرمان؛ `"g711-ulaw-8khz"` فقط برای جفت‌فرمان‌های قدیمی/سفارشی است که صوت تلفنی تولید می‌کنند                                                                                                   |
| `chrome.audioBufferBytes`         | `4096`                                   | بافر پردازش SoX برای فرمان‌های صوتی جفت‌فرمان تولیدشده (نصف بافر پیش‌فرض 8192 بایتی SoX، برای کاهش تأخیر لوله)؛ مقادیر به حداقل 17 بایت محدود می‌شوند                                         |
| `chrome.audioInputCommand`        | فرمان SoX تولیدشده                    | از CoreAudio ‏`BlackHole 2ch` می‌خواند و صوت را در `chrome.audioFormat` می‌نویسد                                                                                                                                        |
| `chrome.audioOutputCommand`       | فرمان SoX تولیدشده                    | صوت را در `chrome.audioFormat` می‌خواند و در CoreAudio ‏`BlackHole 2ch` می‌نویسد                                                                                                                                          |
| `chrome.bargeInInputCommand`      | تنظیم‌نشده                                    | فرمان اختیاری میکروفون محلی که برای تشخیص ورود هم‌زمان انسان به مکالمه هنگام پخش دستیار، PCM تک‌کانالهٔ 16 بیتی علامت‌دار با ترتیب بایت کم‌ارزش‌اول می‌نویسد؛ برای پل جفت‌فرمان میزبانی‌شده در Gateway اعمال می‌شود                          |
| `chrome.bargeInRmsThreshold`      | `650`                                    | سطح RMS که به‌عنوان وقفهٔ انسانی محسوب می‌شود                                                                                                                                                                           |
| `chrome.bargeInPeakThreshold`     | `2500`                                   | سطح اوج که به‌عنوان وقفهٔ انسانی محسوب می‌شود                                                                                                                                                                          |
| `chrome.bargeInCooldownMs`        | `900`                                    | حداقل تأخیر بین پاک‌سازی‌های مکرر وقفه                                                                                                                                                                |
| `mode` (برای هر درخواست)              | `"agent"`                                | حالت پاسخ گفتاری؛ جدول [حالت‌های عامل و دوسویه](#agent-and-bidi-modes) را ببینید                                                                                                                                       |
| `realtime.provider`               | `"openai"`                               | جایگزین سازگاری که هنگام تنظیم‌نبودن فیلدهای محدوده‌دار زیر استفاده می‌شود                                                                                                                                                |
| `realtime.transcriptionProvider`  | `"openai"`                               | شناسهٔ ارائه‌دهنده که حالت `agent` برای رونویسی بی‌درنگ استفاده می‌کند                                                                                                                                                       |
| `realtime.voiceProvider`          | تنظیم‌نشده                                    | شناسهٔ ارائه‌دهنده که حالت `bidi` برای صدای مستقیم بی‌درنگ استفاده می‌کند؛ برای Gemini Live روی `"google"` تنظیم کنید، درحالی‌که رونویسی حالت عامل روی OpenAI باقی می‌ماند. برای انتخاب مدل مشخص Gemini Live آن را با `realtime.model` همراه کنید. |
| `realtime.toolPolicy`             | `"safe-read-only"`                       | [حالت‌های عامل و دوسویه](#agent-and-bidi-modes) را ببینید                                                                                                                                                                 |
| `realtime.instructions`           | دستورالعمل‌های کوتاه برای پاسخ گفتاری          | به مدل می‌گوید کوتاه صحبت کند و برای پاسخ‌های عمیق‌تر از `openclaw_agent_consult` استفاده کند                                                                                                                              |
| `realtime.introMessage`           | `"Say exactly: I'm here and listening."` | هنگام اتصال پل بی‌درنگ یک‌بار گفته می‌شود؛ برای پیوستن بی‌صدا روی `""` تنظیم کنید                                                                                                                                       |
| `realtime.agentId`                | `"main"`                                 | شناسهٔ عامل OpenClaw که برای `openclaw_agent_consult` استفاده می‌شود                                                                                                                                                               |
| `voiceCall.enabled`               | `true`                                   | تماس PSTN ‏Twilio، ‏DTMF و خوشامدگویی آغازین را به Plugin تماس صوتی واگذار می‌کند                                                                                                                                 |
| `voiceCall.dtmfDelayMs`           | `12000`                                  | انتظار اولیه پیش از پخش دنبالهٔ DTMF برگرفته از PIN از طریق Twilio                                                                                                                                               |
| `voiceCall.postDtmfSpeechDelayMs` | `5000`                                   | تأخیر پیش از درخواست خوشامدگویی آغازین بی‌درنگ، پس از آنکه تماس صوتی بخش Twilio را آغاز می‌کند                                                                                                                        |

`chrome.audioBridgeCommand` و `chrome.audioBridgeHealthCommand` به یک پل خارجی اجازه می‌دهند به‌جای `chrome.audioInputCommand`/`chrome.audioOutputCommand` مالک کل مسیر صوتی محلی باشد؛ برای محدودیت حالت‌هایی که می‌توانند از آن‌ها استفاده کنند، [یادداشت‌ها](#notes) را ببینید.

یک مهاجرت `openclaw doctor --fix` برای ساختار قدیمی `realtime.provider: "google"` وجود دارد: اگر این فیلدها از قبل تنظیم نشده باشند، آن منظور را به `realtime.voiceProvider: "google"` به‌همراه `realtime.transcriptionProvider: "openai"` منتقل می‌کند.

### بازنویسی‌های اختیاری

```json5
{
  defaults: {
    meeting: "https://meet.google.com/abc-defg-hij",
  },
  browser: {
    defaultProfile: "openclaw",
  },
  chrome: {
    guestName: "OpenClaw Agent",
    waitForInCallMs: 30000,
    bargeInInputCommand: [
      "sox",
      "-q",
      "-t",
      "coreaudio",
      "External Microphone",
      "-r",
      "24000",
      "-c",
      "1",
      "-b",
      "16",
      "-e",
      "signed-integer",
      "-t",
      "raw",
      "-",
    ],
  },
  chromeNode: {
    node: "parallels-macos",
  },
  defaultMode: "agent",
  realtime: {
    provider: "openai",
    transcriptionProvider: "openai",
    voiceProvider: "google",
    model: "gemini-3.1-flash-live-preview",
    agentId: "jay",
    toolPolicy: "owner",
    introMessage: "Say exactly: I'm here.",
    providers: {
      google: {
        speakerVoice: "Kore",
      },
    },
  },
}
```

ElevenLabs برای شنیدن و صحبت‌کردن در حالت عامل:

```json5
{
  tts: {
    provider: "elevenlabs",
    providers: {
      elevenlabs: {
        modelId: "eleven_v3",
        speakerVoiceId: "pMsXgVXv3BLzUgSXRplE",
      },
    },
  },
  plugins: {
    entries: {
      "google-meet": {
        config: {
          realtime: {
            transcriptionProvider: "elevenlabs",
            providers: {
              elevenlabs: {
                modelId: "scribe_v2_realtime",
                audioFormat: "ulaw_8000",
                sampleRate: 8000,
                commitStrategy: "vad",
              },
            },
          },
        },
      },
    },
  },
}
```

صدای پایدار Meet از `tts.providers.elevenlabs.speakerVoiceId` می‌آید. پاسخ‌های عامل می‌توانند هنگام فعال‌بودن بازنویسی مدل TTS، از دستورالعمل‌های `[[tts:speakerVoiceId=... model=eleven_v3]]` برای هر پاسخ نیز استفاده کنند، اما پیکربندی، پیش‌فرض قطعی برای جلسه‌ها است. هنگام پیوستن، گزارش‌ها `transcriptionProvider=elevenlabs` را نشان می‌دهند و هر پاسخ گفتاری، `provider=elevenlabs model=eleven_v3 speakerVoiceId=<voiceId>` را ثبت می‌کند.

پیکربندی فقط برای Twilio:

```json5
{
  defaultTransport: "twilio",
  twilio: {
    defaultDialInNumber: "+15551234567",
    defaultPin: "123456",
  },
  voiceCall: {
    gatewayUrl: "ws://127.0.0.1:18789",
  },
}
```

با `voiceCall.enabled: true` (حالت پیش‌فرض) و انتقال Twilio، تماس صوتی پیش از بازکردن جریان رسانه‌ای بی‌درنگ، دنبالهٔ DTMF را برقرار می‌کند و سپس متن آغازین ذخیره‌شده را به‌عنوان خوشامدگویی اولیهٔ بی‌درنگ به‌کار می‌برد. اگر `voice-call` فعال نباشد، Google Meet همچنان می‌تواند طرح شماره‌گیری را اعتبارسنجی و ثبت کند، اما نمی‌تواند تماس Twilio را برقرار کند.

برای استفاده از زمان‌اجرای محلی و مورداعتماد Gateway، `voiceCall.gatewayUrl` را تنظیم‌نشده باقی بگذارید؛ این کار عامل فراخواننده را در تمام مدت فراخوانی حفظ می‌کند. نشانی URL پیکربندی‌شده برای Gateway همچنان یک مقصد صریح WebSocket است و نمی‌تواند منشأ Plugin را احراز هویت کند؛ پیوستن عامل‌های غیراپیش‌فرض به‌جای استفاده بی‌سروصدا از عاملی دیگر، به‌صورت بسته و ایمن ناموفق می‌شود. هنگامی که مسیریابی به‌ازای هر عامل لازم است، Google Meet و Voice Call را در همان فرایند Gateway اجرا کنید.

## ابزار

عامل‌ها از ابزار `google_meet` استفاده می‌کنند:

```json
{
  "action": "join",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "mode": "agent"
}
```

| `action`                | کاربرد                                                                                           |
| ----------------------- | ------------------------------------------------------------------------------------------------- |
| `join`                  | پیوستن به یک نشانی URL صریح Meet                                                                         |
| `create`                | ایجاد یک فضا (و پیوستن به‌صورت پیش‌فرض)؛ از `accessType`/`entryPointAccess` پشتیبانی می‌کند                    |
| `status`                | فهرست‌کردن نشست‌های فعال، یا بررسی یکی از آن‌ها با `sessionId`                                               |
| `setup_status`          | اجرای همان بررسی‌های `googlemeet setup`                                                         |
| `resolve_space`         | تفکیک یک URL/کد/`spaces/{id}` از طریق `spaces.get`                                                 |
| `preflight`             | اعتبارسنجی پیش‌نیازهای OAuth و تفکیک جلسه                                                 |
| `latest`                | یافتن جدیدترین رکورد کنفرانس برای یک جلسه                                                   |
| `calendar_events`       | پیش‌نمایش رویدادهای Calendar دارای پیوند Meet                                                           |
| `artifacts`             | فهرست‌کردن رکوردهای کنفرانس و فراداده شرکت‌کنندگان/ضبط‌ها/رونوشت‌ها/یادداشت‌های هوشمند                  |
| `attendance`            | فهرست‌کردن شرکت‌کنندگان و نشست‌های آن‌ها                                                        |
| `export`                | نوشتن بسته مصنوعات/حضور و غیاب/رونوشت/مانیفست؛ برای فقط مانیفست، `"dryRun": true` را تنظیم کنید |
| `recover_current_tab`   | متمرکزشدن روی یک زبانه موجود Meet یا بررسی آن، بدون بازکردن زبانه‌ای جدید                                      |
| `transcript`            | خواندن رونوشت محدود زیرنویس؛ `sinceIndex` از `nextIndex` قبلی ادامه می‌دهد           |
| `leave`                 | پایان‌دادن به یک نشست (Chrome روی Leave کلیک می‌کند؛ فقط زبانه‌هایی را می‌بندد که خودش باز کرده است؛ Twilio تماس را قطع می‌کند)                  |
| `end_active_conference` | پایان‌دادن به کنفرانس فعال Google Meet برای فضایی که با API مدیریت می‌شود                                    |
| `speak`                 | واداشتن عامل بلادرنگ به صحبت فوری، با دریافت `sessionId` و `message`                        |
| `test_speech`           | ایجاد/استفاده مجدد از یک نشست، فعال‌کردن عبارتی شناخته‌شده و بازگرداندن وضعیت سلامت Chrome                              |
| `test_listen`           | ایجاد/استفاده مجدد از یک نشست فقط‌نظارتی و انتظار برای حرکت زیرنویس/رونوشت                        |

`test_speech` همیشه `mode: "agent"` یا `"bidi"` را اجباری می‌کند و اگر از آن خواسته شود در `mode: "transcribe"` اجرا شود، ناموفق خواهد شد؛ زیرا نشست‌های فقط‌نظارتی نمی‌توانند گفتار تولید کنند. `speechOutputVerified` هم به بایت‌های تازه خروجی بلادرنگ و هم به صدای تازه و غیرساکتی نیاز دارد که هنگام همان خروجی از مسیر ضبط میکروفن پل بازگردد. خروجی قدیمی‌تر یا سیگنال بازگشتی یک نشست استفاده‌شده مجدد محسوب نمی‌شود و افزایش بایت‌های خروجی صوتی نیز دیگر به‌تنهایی گفتار تأییدشده را گزارش نمی‌کند.

برای انتقال‌های Chrome، `leave` پس از کلیک روی دکمه Leave call در Meet، زبانه متعلق به کاربر را که دوباره استفاده شده است باز نگه می‌دارد. زبانه‌هایی که OpenClaw باز کرده است پس از خروج بسته می‌شوند.

وقتی Chrome روی میزبان Gateway اجرا می‌شود از `transport: "chrome"` و وقتی روی یک Node جفت‌شده اجرا می‌شود از `transport: "chrome-node"` استفاده کنید. در هر دو حالت، ارائه‌دهندگان مدل و `openclaw_agent_consult` روی میزبان Gateway اجرا می‌شوند؛ بنابراین اعتبارنامه‌های مدل همان‌جا باقی می‌مانند. گزارش‌های حالت عامل هنگام راه‌اندازی پل، ارائه‌دهنده/مدل تفکیک‌شده رونویسی و پس از هر پاسخ ترکیب‌شده، ارائه‌دهنده/مدل/صدا/قالب خروجی/نرخ نمونه‌برداری TTS را دربر می‌گیرند. `mode: "realtime"` خام همچنان به‌عنوان نام مستعار سازگاری قدیمی برای `mode: "agent"` پذیرفته می‌شود، اما دیگر در enum مربوط به `mode` ابزار نمایش داده نمی‌شود.

`create` با یک اتاق مبتنی بر API و سیاست دسترسی صریح:

```json
{
  "action": "create",
  "transport": "chrome-node",
  "mode": "agent",
  "accessType": "OPEN"
}
```

پایان‌دادن به کنفرانس فعال یک اتاق شناخته‌شده:

```json
{
  "action": "end_active_conference",
  "meeting": "https://meet.google.com/abc-defg-hij"
}
```

اعتبارسنجی ابتدا-شنیدن پیش از ادعای مفیدبودن یک جلسه:

```json
{
  "action": "test_listen",
  "url": "https://meet.google.com/abc-defg-hij",
  "transport": "chrome-node",
  "timeoutMs": 30000
}
```

صحبت‌کردن در صورت درخواست:

```json
{
  "action": "speak",
  "sessionId": "meet_...",
  "message": "دقیقاً بگو: من اینجا هستم و گوش می‌دهم."
}
```

`status` در صورت دسترس‌بودن، وضعیت سلامت Chrome را دربر می‌گیرد:

| فیلد                                                                 | معنا                                                                                                                |
| --------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `inCall`                                                              | به نظر می‌رسد Chrome داخل تماس Meet است                                                                              |
| `micMuted`                                                            | وضعیت میکروفن Meet بر مبنای بهترین تلاش                                                                                      |
| `manualActionRequired` / `manualActionReason` / `manualActionMessage` | پیش از آنکه گفتار کار کند، نمایه مرورگر به ورود دستی، پذیرش توسط میزبان Meet، مجوزها یا تعمیر کنترل مرورگر نیاز دارد |
| `speechReady` / `speechBlockedReason` / `speechBlockedMessage`        | آیا گفتار Chrome مدیریت‌شده اکنون مجاز است؛ `speechReady: false` یعنی OpenClaw عبارت معرفی/آزمایش را ارسال نکرده است   |
| `providerConnected` / `realtimeReady`                                 | وضعیت پل صوتی بلادرنگ                                                                                            |
| `lastInputAt` / `lastOutputAt`                                        | آخرین صدای دریافت‌شده از/ارسال‌شده به پل                                                                                |
| `audioOutputRouted` / `audioOutputDeviceLabel`                        | آیا خروجی رسانه زبانه Meet به‌طور فعال به دستگاه BlackHole پل هدایت شده است                               |
| `lastOutputLoopbackAt` / `outputLoopbackSignalBytes`                  | خروجی تازه‌ای که اثرانگشت شکل موج آن در مسیر ضبط میکروفن BlackHole هم‌بسته شده است                        |
| `lastOutputLoopbackCorrelation`                                       | امتیاز هم‌بستگی که سیگنال ضبط‌شده را به تولید فعلی خروجی دستیار مرتبط می‌کند                                 |
| `outputGeneration` / `verifiedOutputGeneration`                       | شناسه‌های یکنواخت صعودی؛ برابری یعنی خروجی فعلی، نه گفتاری قدیمی‌تر، اثبات بازگشت صوتی را گذرانده است                |
| `lastOutputLoopbackRms` / `lastOutputLoopbackPeak`                    | اطلاعات تشخیصی انرژی صوتی برای جدیدترین قطعه ضبط بازگشتی تأییدشده                                                |
| `lastSuppressedInputAt` / `suppressedInputBytes`                      | ورودی بازگشتی که هنگام پخش صدای دستیار نادیده گرفته شده است                                                              |

## حالت‌های عامل و دوسویه

| حالت    | چه کسی پاسخ را تعیین می‌کند        | مسیر خروجی گفتار                     | زمان استفاده                                              |
| ------- | ----------------------------- | -------------------------------------- | ----------------------------------------------------- |
| `agent` | عامل پیکربندی‌شده OpenClaw | زمان‌اجرای عادی TTS در OpenClaw            | رفتار «عامل من در جلسه است» را می‌خواهید        |
| `bidi`  | مدل صوتی بلادرنگ      | پاسخ صوتی ارائه‌دهنده صدای بلادرنگ | حلقه مکالمه صوتی با کمترین تأخیر را می‌خواهید |

حالت `agent`: ارائه‌دهنده رونویسی بلادرنگ صدای جلسه را می‌شنود، رونوشت نهایی شرکت‌کنندگان از طریق عامل پیکربندی‌شده OpenClaw مسیریابی می‌شود و پاسخ از طریق TTS عادی OpenClaw به گفتار تبدیل می‌شود. قطعات نزدیک رونوشت نهایی پیش از مشورت با یکدیگر ادغام می‌شوند تا یک نوبت گفتار چندین پاسخ جزئی و منسوخ تولید نکند؛ ورودی بلادرنگ تا زمانی که صدای دستیار در صف همچنان در حال پخش است مهار می‌شود و بازتاب‌های اخیر رونوشت که شبیه گفتار دستیار هستند پیش از مشورت نادیده گرفته می‌شوند تا بازگشت صوتی BlackHole باعث نشود عامل به گفتار خودش پاسخ دهد.

حالت `bidi`: مدل صوتی بلادرنگ مستقیماً پاسخ می‌دهد و می‌تواند برای استدلال عمیق‌تر، اطلاعات جاری یا ابزارهای عادی OpenClaw، `openclaw_agent_consult` را فراخوانی کند. ابزار مشورت، عامل عادی OpenClaw را در پشت صحنه با زمینه رونوشت اخیر جلسه اجرا می‌کند و پاسخی کوتاه و مناسب گفتار بازمی‌گرداند؛ در حالت `agent`، OpenClaw آن پاسخ را مستقیماً به TTS می‌فرستد و در حالت `bidi`، مدل صوتی بلادرنگ می‌تواند آن را بیان کند. این حالت از همان سازوکار مشترک مشورت Voice Call استفاده می‌کند.

به‌طور پیش‌فرض، مشورت‌ها روی عامل `main` اجرا می‌شوند؛ `realtime.agentId` را تنظیم کنید تا یک مسیر Meet به فضای کاری اختصاصی عامل، پیش‌فرض‌های مدل، سیاست ابزار، حافظه و تاریخچه نشست متصل شود. مشورت‌های حالت عامل از یک کلید نشست `agent:<id>:subagent:google-meet:<session>` مختص هر جلسه استفاده می‌کنند تا پرسش‌های بعدی، ضمن به‌ارث‌بردن سیاست عادی عامل، زمینه جلسه را حفظ کنند. وقتی عاملی در حالت عامل `google_meet` را فراخوانی می‌کند، نشست مشاور پیش از پاسخ‌دادن به گفتار شرکت‌کننده از رونوشت فعلی فراخواننده منشعب می‌شود؛ نشست Meet جدا باقی می‌ماند تا پرسش‌های بعدی جلسه مستقیماً رونوشت فراخواننده را تغییر ندهند.

`realtime.toolPolicy` اجرای مشورت را کنترل می‌کند:

| سیاست           | رفتار                                                                                                                         |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `safe-read-only` | ابزار مشورت را در دسترس قرار می‌دهد؛ عامل عادی را به `read`، `web_search`، `web_fetch`، `x_search`، `memory_search`، `memory_get` محدود می‌کند |
| `owner`          | ابزار مشورت را در دسترس قرار می‌دهد؛ به عامل عادی اجازه می‌دهد از سیاست معمول ابزار خود استفاده کند                                                        |
| `none`           | ابزار مشورت را در اختیار مدل صوتی بلادرنگ قرار نمی‌دهد                                                                       |

کلید نشست مشورت به هر نشست Meet محدود است؛ بنابراین فراخوانی‌های بعدی مشورت در همان جلسه از زمینه مشورت قبلی دوباره استفاده می‌کنند.

پس از آنکه Chrome به‌طور کامل پیوست، بررسی آمادگی گفتاری را اجباری کنید:

```bash
openclaw googlemeet speak meet_... "دقیقاً بگو: من اینجا هستم و گوش می‌دهم."
```

آزمون دود کامل پیوستن و صحبت‌کردن:

```bash
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "دقیقاً بگو: من اینجا هستم و گوش می‌دهم."
```

## چک‌لیست آزمون زنده

پیش از واگذاری یک جلسه به عامل بدون نظارت:

```bash
openclaw googlemeet setup
openclaw nodes status
openclaw googlemeet test-speech https://meet.google.com/abc-defg-hij \
  --transport chrome-node \
  --message "دقیقاً بگو: آزمایش گفتار Google Meet کامل شد."
```

وضعیت مورد انتظار Chrome-node:

- `googlemeet setup` کاملاً سبز است و وقتی Chrome-node انتقال پیش‌فرض باشد یا یک Node سنجاق شده باشد، `chrome-node-connected` را نیز شامل می‌شود.
- `nodes status` اتصال Node انتخاب‌شده را نشان می‌دهد که هر دو `googlemeet.chrome` و `browser.proxy` را اعلام می‌کند.
- زبانه Meet متصل می‌شود و `test-speech` سلامت Chrome را با `inCall: true` برمی‌گرداند.

برای یک میزبان راه‌دور Chrome مانند ماشین مجازی macOS در Parallels، کوتاه‌ترین بررسی ایمن پس از به‌روزرسانی Gateway یا ماشین مجازی:

```bash
openclaw googlemeet setup
openclaw nodes status --connected
openclaw nodes invoke \
  --node parallels-macos \
  --command googlemeet.chrome \
  --params '{"action":"setup"}'
```

این کار ثابت می‌کند Plugin مربوط به Gateway بارگذاری شده، Node ماشین مجازی با توکن فعلی متصل است و پل صوتی Meet پیش از آنکه یک عامل زبانه واقعی جلسه را باز کند، در دسترس است.

برای یک آزمون دود Twilio، از جلسه‌ای استفاده کنید که جزئیات تماس تلفنی را ارائه می‌دهد:

```bash
openclaw googlemeet setup
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --pin 123456
```

وضعیت مورد انتظار Twilio:

- `googlemeet setup` شامل بررسی‌های سبز `twilio-voice-call-plugin`، `twilio-voice-call-credentials` و `twilio-voice-call-webhook` است.
- `voicecall` پس از بارگذاری مجدد Gateway در CLI در دسترس است.
- نشست برگردانده‌شده دارای `transport: "twilio"` و یک `twilio.voiceCallId` است.
- `openclaw logs --follow` نشان می‌دهد TwiML مربوط به DTMF پیش از TwiML بی‌درنگ ارائه شده و سپس یک پل بی‌درنگ با پیام خوشامدگویی اولیه در صف قرار گرفته است.
- `googlemeet leave <sessionId>` تماس صوتی واگذارشده را قطع می‌کند.

## عیب‌یابی

### عامل نمی‌تواند ابزار Google Meet را ببیند

فعال بودن Plugin را تأیید و Gateway را دوباره بارگذاری کنید؛ عامل در حال اجرا فقط ابزارهای Plugin ثبت‌شده توسط فرایند فعلی Gateway را می‌بیند:

```bash
openclaw plugins list | grep google-meet
openclaw googlemeet setup
```

در میزبان‌های Gateway غیر macOS، `google_meet` قابل مشاهده می‌ماند، اما کنش‌های پاسخ صوتی محلی Chrome پیش از رسیدن به پل صوتی مسدود می‌شوند. به‌جای مسیر پیش‌فرض عامل محلی Chrome، از `mode: "transcribe"`، تماس تلفنی Twilio یا یک میزبان macOS دارای `chrome-node` استفاده کنید.

### هیچ Node متصل و سازگار با Google Meet وجود ندارد

در میزبان Node:

```bash
openclaw plugins install npm:@openclaw/google-meet
openclaw plugins enable browser
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node run --host <gateway-lan-ip> --port 18789 --display-name parallels-macos
```

در میزبان Gateway:

```bash
openclaw devices list
openclaw devices approve <requestId>
openclaw nodes status
```

Node باید متصل باشد و `googlemeet.chrome` و `browser.proxy` را فهرست کند؛ پیکربندی Gateway نیز باید هر دو را مجاز بداند:

```json5
{
  gateway: {
    nodes: {
      commands: { allow: ["browser.proxy", "googlemeet.chrome"] },
    },
  },
}
```

اگر `googlemeet setup` در `chrome-node-connected` ناموفق بود یا گزارش Gateway حاوی `gateway token mismatch` بود، Node را با توکن فعلی Gateway دوباره نصب یا راه‌اندازی کنید:

```bash
OPENCLAW_ALLOW_INSECURE_PRIVATE_WS=1 \
  openclaw node install \
  --host <gateway-lan-ip> \
  --port 18789 \
  --display-name parallels-macos \
  --force
```

سپس سرویس Node را دوباره بارگذاری و این موارد را دوباره اجرا کنید:

```bash
openclaw googlemeet setup
openclaw nodes status --connected
```

### مرورگر باز می‌شود اما عامل نمی‌تواند به جلسه بپیوندد

برای پیوستن‌های فقط-مشاهده، `googlemeet test-listen` و برای پیوستن‌های بی‌درنگ، `googlemeet test-speech` را اجرا کنید؛ سپس سلامت Chrome برگردانده‌شده را بررسی کنید. اگر هریک `manualActionRequired: true` را گزارش کرد، `manualActionMessage` را به اپراتور نشان دهید و تا تکمیل کنش مرورگر، تلاش مجدد را متوقف کنید.

کنش‌های دستی رایج: وارد نمایه Chrome شوید؛ مهمان را از حساب میزبان Meet بپذیرید؛ هنگامی که اعلان بومی ظاهر می‌شود، مجوزهای میکروفون/دوربین Chrome را اعطا کنید؛ پنجره مجوز گیرکرده Meet را ببندید یا ترمیم کنید.

صرفاً به این دلیل که Meet می‌پرسد «Do you want people to hear you in the meeting?» وضعیت «وارد نشده» را گزارش نکنید؛ این صفحه میانی انتخاب صدا در Meet است. OpenClaw در صورت امکان از طریق خودکارسازی مرورگر روی **Use microphone** کلیک می‌کند و همچنان منتظر وضعیت واقعی جلسه می‌ماند؛ برای مسیر جایگزین مرورگرِ فقط-ایجاد، ممکن است به‌جای آن روی **Continue without microphone** کلیک کند، زیرا تولید URL به مسیر صوتی بی‌درنگ نیازی ندارد.

### ایجاد جلسه ناموفق است

`googlemeet create` در صورت پیکربندی OAuth از `spaces.create` در API مربوط به Meet استفاده می‌کند؛ در غیر این صورت از مرورگر Node سنجاق‌شده Chrome استفاده می‌کند. موارد زیر را تأیید کنید:

- **ایجاد از طریق API**: `oauth.clientId` و `oauth.refreshToken` (یا متغیرهای محیطی متناظر `OPENCLAW_GOOGLE_MEET_*`) موجود هستند و توکن نوسازی پس از افزوده‌شدن پشتیبانی ایجاد صادر شده است؛ ممکن است توکن‌های قدیمی فاقد `meetings.space.created` باشند، بنابراین `openclaw googlemeet auth login --json` را دوباره اجرا کنید.
- **مسیر جایگزین مرورگر**: `defaultTransport: "chrome-node"` و `chromeNode.node` به یک Node متصل دارای `browser.proxy` و `googlemeet.chrome` اشاره می‌کنند؛ نمایه OpenClaw در Chrome روی آن Node وارد شده و می‌تواند `https://meet.google.com/new` را باز کند.
- **تلاش‌های مجدد مسیر جایگزین مرورگر**: پیش از بازکردن زبانه جدید، از یک `.../new` موجود یا زبانه اعلان حساب Google دوباره استفاده کنید؛ به‌جای بازکردن دستی زبانه‌ای دیگر، فراخوانی ابزار را دوباره امتحان کنید.
- **کنش دستی**: اگر ابزار `manualActionRequired: true` را برگرداند، از `browser.nodeId`، `browser.targetId`، `browserUrl` و `manualActionMessage` برای راهنمایی اپراتور استفاده کنید؛ در یک حلقه تلاش مجدد نکنید.
- **صفحه میانی انتخاب صدا**: اگر Meet عبارت «Do you want people to hear you in the meeting?» را نشان داد، زبانه را باز نگه دارید. OpenClaw باید روی **Use microphone** یا (برای فقط-ایجاد) **Continue without microphone** کلیک کند و همچنان منتظر URL تولیدشده بماند؛ اگر نتواند، خطا باید به `meet-audio-choice-required` اشاره کند، نه `google-login-required`.

### عامل به جلسه می‌پیوندد اما صحبت نمی‌کند

```bash
openclaw googlemeet setup
openclaw googlemeet doctor
```

برای مسیر STT -> عامل OpenClaw -> TTS از `mode: "agent"` و برای مسیر جایگزین مستقیم صدای بی‌درنگ از `mode: "bidi"` استفاده کنید. `mode: "transcribe"` عمداً هیچ پل پاسخ صوتی را راه‌اندازی نمی‌کند. برای عیب‌یابی فقط-مشاهده، پس از صحبت شرکت‌کنندگان `openclaw googlemeet status --json <session-id>` را اجرا و `captioning`، `transcriptLines` و `lastCaptionText` را بررسی کنید. اگر `inCall` درست است اما `transcriptLines` همچنان `0` می‌ماند، ممکن است زیرنویس‌های Meet غیرفعال باشند، از زمان نصب مشاهده‌گر کسی صحبت نکرده باشد، رابط کاربری Meet تغییر کرده باشد یا زیرنویس زنده برای زبان/حساب جلسه در دسترس نباشد.

`googlemeet test-speech` همیشه مسیر بی‌درنگ را بررسی می‌کند و گزارش می‌دهد آیا برای آن فراخوانی، بایت‌های خروجی پل مشاهده شده‌اند یا نه. اگر `speechOutputVerified` نادرست و `speechOutputTimedOut` درست باشد، ممکن است ارائه‌دهنده بی‌درنگ گفتار را پذیرفته باشد اما OpenClaw رسیدن بایت‌های خروجی جدید به پل صوتی Chrome را مشاهده نکرده باشد.

همچنین بررسی کنید: کلید یک ارائه‌دهنده بی‌درنگ (`OPENAI_API_KEY` یا `GEMINI_API_KEY`) در میزبان Gateway در دسترس باشد؛ `BlackHole 2ch` در میزبان Chrome قابل مشاهده باشد؛ `sox` در آنجا وجود داشته باشد؛ میکروفون/بلندگوی Meet از مسیر صوتی مجازی عبور داده شوند (`doctor` باید برای پیوستن‌های بی‌درنگ محلی Chrome، `meet output routed: yes` را نشان دهد).

`googlemeet doctor [session-id]` نشست، Node، وضعیت حضور در تماس، دلیل کنش دستی، اتصال ارائه‌دهنده بی‌درنگ، `realtimeReady`، فعالیت ورودی/خروجی صدا، آخرین برچسب‌های زمانی صدا، شمارنده‌های بایت و URL مرورگر را چاپ می‌کند. برای JSON خام از `googlemeet status [session-id] --json` و برای تأیید نوسازی OAuth بدون افشای توکن‌ها از `googlemeet doctor --oauth` (با افزودن `--meeting` یا `--create-space`) استفاده کنید.

اگر زمان یک عامل به پایان رسیده و زبانه Meet از قبل باز است، بدون بازکردن زبانه دیگری آن را بررسی کنید:

```bash
openclaw googlemeet recover-tab
openclaw googlemeet recover-tab https://meet.google.com/abc-defg-hij
```

کنش ابزار معادل، `recover_current_tab` است: این کنش یک زبانه موجود Meet را برای انتقال انتخاب‌شده (کنترل مرورگر محلی برای `chrome` و Node پیکربندی‌شده برای `chrome-node`) بدون بازکردن زبانه یا نشست جدید، متمرکز و بررسی می‌کند و مانع فعلی (ورود، پذیرش، مجوزها یا وضعیت انتخاب صدا) را گزارش می‌دهد. فرمان CLI با Gateway پیکربندی‌شده ارتباط برقرار می‌کند که باید در حال اجرا باشد؛ `chrome-node` همچنین مستلزم اتصال Node است.

### بررسی‌های راه‌اندازی Twilio ناموفق هستند

`twilio-voice-call-plugin` زمانی ناموفق است که `voice-call` مجاز یا فعال نباشد: آن را به `plugins.allow` اضافه کنید، `plugins.entries.voice-call` را فعال و Gateway را دوباره بارگذاری کنید.

`twilio-voice-call-credentials` زمانی ناموفق است که پشتیبان Twilio فاقد SID حساب، توکن احراز هویت یا شماره تماس‌گیرنده باشد:

```bash
export TWILIO_ACCOUNT_SID=AC...
export TWILIO_AUTH_TOKEN=...
export TWILIO_FROM_NUMBER=+15550001234
```

`twilio-voice-call-webhook` زمانی ناموفق است که `voice-call` هیچ دسترسی عمومی Webhook نداشته باشد یا `publicUrl` به فضای شبکه حلقه‌بازگشت/خصوصی اشاره کند. از `localhost`، `127.0.0.1`، `0.0.0.0`، `10.x`، `172.16.x`-`172.31.x`، `192.168.x`، `169.254.x`، `fc00::/7` یا `fd00::/8` به‌عنوان `publicUrl` استفاده نکنید؛ فراخوان‌های شرکت مخابراتی نمی‌توانند به آن‌ها دسترسی پیدا کنند. `plugins.entries.voice-call.config.publicUrl` را روی یک URL عمومی تنظیم کنید یا دسترسی از طریق تونل/Tailscale را پیکربندی کنید:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        enabled: true,
        config: {
          provider: "twilio",
          fromNumber: "+15550001234",
          publicUrl: "https://voice.example.com/voice/webhook",
        },
      },
    },
  },
}
```

برای توسعه محلی، به‌جای URL میزبان خصوصی از تونل یا دسترسی Tailscale استفاده کنید:

```json5
{
  plugins: {
    entries: {
      "voice-call": {
        config: {
          tunnel: { provider: "ngrok" },
          // یا
          tailscale: { mode: "funnel", path: "/voice/webhook" },
        },
      },
    },
  },
}
```

Gateway را دوباره راه‌اندازی یا بارگذاری کنید، سپس:

```bash
openclaw googlemeet setup --transport twilio
openclaw voicecall setup
openclaw voicecall smoke
```

`voicecall smoke` به‌طور پیش‌فرض فقط آمادگی را بررسی می‌کند. یک شماره مشخص را به‌صورت اجرای آزمایشی بررسی کنید:

```bash
openclaw voicecall smoke --to "+15555550123"
```

فقط برای برقراری عمدی یک تماس خروجی زنده، `--yes` را اضافه کنید:

```bash
openclaw voicecall smoke --to "+15555550123" --yes
```

### تماس Twilio آغاز می‌شود اما هرگز وارد جلسه نمی‌شود

تأیید کنید رویداد Meet جزئیات تماس تلفنی را ارائه می‌دهد و شماره تماس دقیق را همراه با PIN یا یک توالی سفارشی DTMF وارد کنید:

```bash
openclaw googlemeet join https://meet.google.com/abc-defg-hij \
  --transport twilio \
  --dial-in-number +15551234567 \
  --dtmf-sequence ww123456#
```

برای ایجاد مکث پیش از PIN، در `--dtmf-sequence` از `w` در ابتدا یا از ویرگول استفاده کنید.

اگر تماس ایجاد شده اما فهرست شرکت‌کنندگان Meet هرگز شرکت‌کننده تلفنی را نشان نمی‌دهد:

- `openclaw googlemeet doctor <session-id>`: شناسه تماس واگذارشده Twilio، اینکه آیا DTMF در صف قرار گرفته و اینکه آیا پیام خوشامدگویی آغازین درخواست شده است را تأیید کنید.
- `openclaw voicecall status --call-id <id>`: تأیید کنید تماس همچنان فعال است.
- `openclaw voicecall tail`: تأیید کنید Webhookهای Twilio به Gateway می‌رسند.
- `openclaw logs --follow`: به‌دنبال توالی Twilio در Meet بگردید: Google Meet پیوستن را واگذار می‌کند، Voice Call، TwiML مربوط به DTMF پیش از اتصال را ذخیره و ارائه می‌کند، Voice Call، TwiML بی‌درنگ را برای تماس Twilio ارائه می‌کند و سپس Google Meet گفتار آغازین را با `voicecall.speak` درخواست می‌کند.
- `openclaw googlemeet setup --transport twilio` را دوباره اجرا کنید؛ بررسی سبز راه‌اندازی الزامی است، اما درستی توالی PIN جلسه را ثابت نمی‌کند.
- تأیید کنید شماره تماس به همان دعوت‌نامه و منطقه Meet مربوط به PIN تعلق دارد.
- اگر Meet دیر پاسخ می‌دهد یا رونوشت تماس پس از ارسال DTMF پیش از اتصال همچنان اعلان PIN را نشان می‌دهد، `voiceCall.dtmfDelayMs` را از مقدار پیش‌فرض 12 ثانیه افزایش دهید.
- اگر شرکت‌کننده متصل می‌شود اما پیام خوشامدگویی را نمی‌شنوید، `openclaw logs --follow` را برای درخواست `voicecall.speak` پس از DTMF و پخش TTS جریان رسانه یا مسیر جایگزین `<Say>` در Twilio بررسی کنید. اگر رونوشت همچنان «enter the meeting PIN» را نشان می‌دهد، بخش تلفنی هنوز به اتاق Meet نپیوسته است؛ بنابراین شرکت‌کنندگان گفتار را نخواهند شنید.

اگر Webhookها نمی‌رسند، ابتدا Plugin تماس صوتی را اشکال‌زدایی کنید: ارائه‌دهنده باید بتواند به `plugins.entries.voice-call.config.publicUrl` یا تونل پیکربندی‌شده دسترسی پیدا کند. به [عیب‌یابی تماس صوتی](/fa/plugins/voice-call#troubleshooting) مراجعه کنید.

## نکات

API رسمی رسانه در Google Meet دریافت‌محور است، بنابراین صحبت‌کردن در تماس همچنان به یک مسیر شرکت‌کننده نیاز دارد. این Plugin این مرز را آشکار نگه می‌دارد: Chrome مشارکت در مرورگر و مسیریابی صدای محلی را مدیریت می‌کند؛ Twilio مشارکت از طریق شماره‌گیری تلفنی را مدیریت می‌کند.

حالت‌های پاسخ صوتی Chrome به `BlackHole 2ch` به‌همراه یکی از موارد زیر نیاز دارند:

- `chrome.audioInputCommand` به‌همراه `chrome.audioOutputCommand`: OpenClaw مالک پل است و صدا را در `chrome.audioFormat` میان این فرمان‌ها و ارائه‌دهنده انتخاب‌شده لوله‌کشی می‌کند. حالت `agent` از رونویسی بلادرنگ به‌همراه TTS معمولی استفاده می‌کند؛ حالت `bidi` از ارائه‌دهنده صوتی بلادرنگ استفاده می‌کند. مسیر پیش‌فرض PCM16 با نرخ 24 kHz و `chrome.audioBufferBytes: 4096` است؛ G.711 mu-law با نرخ 8 kHz برای جفت‌فرمان‌های قدیمی همچنان در دسترس است.
- `chrome.audioBridgeCommand`: یک فرمان پل خارجی مالک کل مسیر صدای محلی است و باید پس از راه‌اندازی یا اعتبارسنجی سرویس پس‌زمینه خود خارج شود. فقط برای `bidi` معتبر است، زیرا حالت `agent` برای TTS به دسترسی مستقیم به جفت‌فرمان نیاز دارد.

با پل جفت‌فرمان Chrome، `chrome.bargeInInputCommand` می‌تواند به یک میکروفون محلی جداگانه گوش دهد و هنگامی که انسانی شروع به صحبت می‌کند، پخش دستیار را پاک کند؛ در نتیجه حتی زمانی که ورودی لوپ‌بک مشترک BlackHole هنگام پخش دستیار موقتاً سرکوب شده است، گفتار انسان بر خروجی دستیار مقدم می‌ماند. مانند `chrome.audioInputCommand`/`chrome.audioOutputCommand`، این یک فرمان محلی پیکربندی‌شده توسط اپراتور است: از یک مسیر فرمان صریح و مورد اعتماد یا فهرست آرگومان‌ها استفاده کنید و هرگز از اسکریپتی در مکانی نامطمئن استفاده نکنید.

برای صدای دوطرفه شفاف، خروجی Meet و میکروفون Meet را از طریق دستگاه‌های مجازی جداگانه یا یک گراف دستگاه مجازی به‌سبک Loopback مسیریابی کنید؛ یک دستگاه مشترک BlackHole می‌تواند صدای دیگر شرکت‌کنندگان را دوباره به تماس بازگرداند و پژواک ایجاد کند.

`googlemeet speak` پل صوتی فعال پاسخ صوتی را برای یک نشست Chrome راه‌اندازی می‌کند؛ `googlemeet leave` آن را متوقف می‌کند (و برای نشست‌های Twilio که از طریق تماس صوتی واگذار شده‌اند، تماس زیربنایی را قطع می‌کند). برای بستن کنفرانس فعال Google Meet در یک فضای مدیریت‌شده با API نیز از `googlemeet end-active-conference` استفاده کنید.

## مرتبط

- [نمای کلی Pluginهای جلسه](/fa/plugins/meeting-plugins)
- [Plugin تماس صوتی](/fa/plugins/voice-call)
- [حالت گفت‌وگو](/fa/nodes/talk)
- [ساخت Pluginها](/fa/plugins/building-plugins)
