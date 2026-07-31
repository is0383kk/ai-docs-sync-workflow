---
read_when:
    - کار روی پروتکل Gateway، کلاینت‌ها یا انتقال‌ها
summary: معماری، مؤلفه‌ها و جریان‌های کلاینت Gateway مبتنی بر WebSocket
title: معماری Gateway
x-i18n:
    generated_at: "2026-07-27T13:59:59Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: f8054bd87f738b957c24f8d6965d55365de2293d44902530a9ba778afa597cc7
    source_path: concepts/architecture.md
    workflow: 16
---

## نمای کلی

- یک **Gateway** واحد و با عمر طولانی، مالک همهٔ سطوح پیام‌رسانی است (WhatsApp از طریق
  Baileys، Telegram از طریق grammY، Slack، Discord، Signal، iMessage و WebChat).
- کلاینت‌های صفحهٔ کنترل (برنامهٔ macOS، CLI، رابط وب و خودکارسازی‌ها) از طریق
  **WebSocket** روی میزبان bind پیکربندی‌شده (پیش‌فرض
  `127.0.0.1:18789`) به Gateway متصل می‌شوند.
- **Nodeها** (macOS/iOS/Android/بدون رابط) نیز از طریق **WebSocket** متصل می‌شوند، اما
  `role: node` را با قابلیت‌ها/دستورهای صریح اعلام می‌کنند.
- برای هر میزبان یک Gateway وجود دارد؛ این تنها جایی است که نشست WhatsApp را باز می‌کند.
- **میزبان canvas** توسط سرور HTTP Gateway در مسیر زیر ارائه می‌شود:
  - `/__openclaw__/canvas/` (HTML/CSS/JS قابل‌ویرایش توسط عامل)
  - `/__openclaw__/a2ui/` (میزبان A2UI)

  این میزبان از همان پورت Gateway استفاده می‌کند (پیش‌فرض `18789`).

## مؤلفه‌ها و جریان‌ها

### Gateway (سرویس پس‌زمینه)

- اتصال‌های ارائه‌دهندگان را نگه می‌دارد.
- یک API نوع‌دار WS ارائه می‌کند (درخواست‌ها، پاسخ‌ها و رویدادهای ارسالی از سرور).
- فریم‌های ورودی را بر اساس JSON Schema اعتبارسنجی می‌کند.
- رویدادهایی مانند `agent`، `chat`، `presence`، `health`، `heartbeat` و `cron` منتشر می‌کند.

### کلاینت‌ها (برنامهٔ Mac / CLI / مدیریت وب)

- برای هر کلاینت یک اتصال WS وجود دارد.
- درخواست‌ها را ارسال می‌کنند (`health`، `status`، `send`، `agent`، `system-presence`).
- در رویدادها مشترک می‌شوند (`tick`، `agent`، `presence`، `shutdown`).

### Nodeها (macOS / iOS / Android / بدون رابط)

- با `role: node` به **همان سرور WS** متصل می‌شوند.
- در `connect` یک هویت دستگاه ارائه می‌کنند؛ جفت‌سازی **مبتنی بر دستگاه** است (نقش `node`) و
  تأیید در مخزن جفت‌سازی دستگاه نگه‌داری می‌شود.
- دستورهایی مانند `canvas.*`، `camera.*`، `screen.record` و `location.get` را ارائه می‌کنند.

جزئیات پروتکل: [پروتکل Gateway](/fa/gateway/protocol)

### WebChat

- رابط کاربری ایستایی که برای تاریخچهٔ گفت‌وگو و ارسال پیام از API مبتنی بر WS در Gateway استفاده می‌کند.
- در راه‌اندازی‌های راه دور، از طریق همان تونل SSH/Tailscale سایر
  کلاینت‌ها متصل می‌شود.

## چرخهٔ عمر اتصال (یک کلاینت)

```mermaid
sequenceDiagram
    participant Client
    participant Gateway

    Client->>Gateway: req:connect
    Gateway-->>Client: res (موفق)
    Note right of Gateway: یا خطای پاسخ + بستن
    Note left of Client: payload=hello-ok<br>نمای فوری: حضور + سلامت

    Gateway-->>Client: event:presence
    Gateway-->>Client: event:tick

    Client->>Gateway: req:agent
    Gateway-->>Client: res:agent<br>تأیید {runId, status:"accepted"}
    Gateway-->>Client: event:agent<br>(جریانی)
    Gateway-->>Client: res:agent<br>نهایی {runId, status, summary}
```

## پروتکل روی سیم (خلاصه)

- انتقال: WebSocket، با فریم‌های متنی دارای محتوای JSON.
- فریم نخست **باید** `connect` باشد.
- پس از دست‌دهی:
  - درخواست‌ها: `{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
  - رویدادها: `{type:"event", event, payload, seq?, stateVersion?}`
- `hello-ok.features.methods` / `events` فرادادهٔ کشف هستند، نه یک
  خروجی تولیدشده از همهٔ مسیرهای کمکی قابل‌فراخوانی.
- احراز هویت با راز مشترک، بسته به حالت احراز هویت پیکربندی‌شدهٔ Gateway، از `connect.params.auth.token` یا
  `connect.params.auth.password` استفاده می‌کند.
- حالت‌های حامل هویت، مانند Tailscale Serve
  (`gateway.auth.allowTailscale: true`) یا
  `gateway.auth.mode: "trusted-proxy"` غیر-loopback، احراز هویت را از سرآیندهای درخواست
  به‌جای `connect.params.auth.*` انجام می‌دهند.
- `gateway.auth.mode: "none"` در ورودی خصوصی، احراز هویت با راز مشترک را
  کاملاً غیرفعال می‌کند؛ این حالت را در ورودی عمومی/غیرقابل‌اعتماد خاموش نگه دارید.
- کلیدهای تکرارناپذیری برای روش‌های دارای اثر جانبی (`send`، `agent`) الزامی هستند تا
  تلاش مجدد با ایمنی انجام شود؛ سرور یک حافظهٔ نهان کوتاه‌عمر برای حذف موارد تکراری نگه می‌دارد.
- Nodeها باید `role: "node"` را به‌همراه قابلیت‌ها/دستورها/مجوزها در `connect` درج کنند.

## جفت‌سازی و اعتماد محلی

- همهٔ کلاینت‌های WS (اپراتورها + Nodeها) در `connect` یک **هویت دستگاه** درج می‌کنند.
- شناسه‌های دستگاه جدید به تأیید جفت‌سازی نیاز دارند؛ Gateway برای اتصال‌های بعدی یک **توکن دستگاه**
  صادر می‌کند.
- اتصال‌های مستقیم loopback محلی می‌توانند به‌طور خودکار تأیید شوند تا تجربهٔ کاربری روی همان میزبان
  روان بماند.
- OpenClaw همچنین برای جریان‌های کمکی قابل‌اعتماد مبتنی بر راز مشترک، یک مسیر محدود اتصال به خود در
  بک‌اند/محفظهٔ محلی دارد.
- اتصال‌های tailnet و LAN، از جمله bindهای tailnet روی همان میزبان، همچنان به
  تأیید صریح جفت‌سازی نیاز دارند.
- همهٔ اتصال‌ها باید nonce مربوط به `connect.challenge` را امضا کنند. محتوای امضای `v3`
  همچنین `platform` و `deviceFamily` را مقید می‌کند؛ Gateway هنگام اتصال مجدد، فرادادهٔ جفت‌شده را تثبیت می‌کند و
  برای تغییرات فراداده به جفت‌سازی ترمیمی نیاز دارد.
- اتصال‌های **غیرمحلی** همچنان به تأیید صریح نیاز دارند.
- احراز هویت Gateway (`gateway.auth.*`) همچنان برای **همهٔ** اتصال‌ها، محلی یا
  راه دور، اعمال می‌شود.

جزئیات: [پروتکل Gateway](/fa/gateway/protocol)، [جفت‌سازی](/fa/channels/pairing)،
[امنیت](/fa/gateway/security).

## نوع‌دهی پروتکل و تولید کد

- طرحواره‌های TypeBox پروتکل را تعریف می‌کنند.
- JSON Schema از این طرحواره‌ها تولید می‌شود.
- مدل‌های Swift از JSON Schema تولید می‌شوند.

## دسترسی راه دور

- روش ترجیحی: Tailscale یا VPN.
- روش جایگزین: تونل SSH

  ```bash
  ssh -N -L 18789:127.0.0.1:18789 user@gateway-host
  ```

- همان دست‌دهی و توکن احراز هویت در تونل نیز اعمال می‌شوند.
- TLS و سنجاق‌کردن اختیاری را می‌توان برای WS در راه‌اندازی‌های راه دور فعال کرد.

## نمای فوری عملیات

- راه‌اندازی: `openclaw gateway` (در پیش‌زمینه، ثبت گزارش در stdout).
- سلامت: `health` از طریق WS (همچنین در `hello-ok` گنجانده شده است).
- نظارت: launchd/systemd برای راه‌اندازی مجدد خودکار.

## ناورداها

- دقیقاً یک Gateway در هر میزبان، یک نشست Baileys را کنترل می‌کند.
- دست‌دهی الزامی است؛ هر فریم نخست غیر-JSON یا غیر-connect باعث بسته‌شدن قطعی می‌شود.
- رویدادها بازپخش نمی‌شوند؛ کلاینت‌ها باید در صورت وجود شکاف، داده‌ها را تازه‌سازی کنند.

## مرتبط

- [حلقهٔ عامل](/fa/concepts/agent-loop) — چرخهٔ اجرای دقیق عامل
- [پروتکل Gateway](/fa/gateway/protocol) — قرارداد پروتکل WebSocket
- [صف](/fa/concepts/queue) — صف دستورها و هم‌زمانی
- [امنیت](/fa/gateway/security) — مدل اعتماد و مقاوم‌سازی
